# 显式会话模型：可实施设计（候选 3f45af6 → 重做第 38 项）

对象：候选 `3f45af6`（行号均取自此版本），基线 `7d5be70`。没有设备，没有 Android 构建环境，未运行任何包内脚本。
标注：【源码】候选源码推断；【基线】`git show 7d5be70:` 核对；【实测】包内原件；【建议】设计。本文不引用上游行号（本设计不依赖平台实现细节，平台相关结论全部归入“需设备”）。

---

## 0. 结论（先读这一节）

1. **改动核心只有一处：`Sessions` 不再隐式新建租约。**【源码】今天所有“静默落主屏”都来自同一行：`Sessions.keepLease` 在没有租约时新建 `DisplayTarget.MAIN` 租约（`android/app/src/main/java/com/rainng/androidctl/Sessions.kt:94`），`acquire`（`:56-64`）与 `acquireUi`（`:136-150`）都走它。把这一分支改成抛 `noSession`，主机偏好文件、`expectedDisplayKind`、`requireTarget`、`display select`、batch 冻结就全部失去存在理由，可以整体删除。
2. **`session open --display main|virtual`（缺省 `main`）是唯一的租约入口**；`session takeover` 接受同样的两个选项。`display` 命令组整个删除，不留别名。从 virtual 回 main 用 `session open --display main`，一条命令原子替换，不需要“关屏但保留会话”的中间态。后台屏只随租约生灭。
3. **virtual 打开的原子性已经基本具备，只需补一处。**【源码】`prepareChange` 先使旧租约失效（`Sessions.kt:120`），`publish` 只在建屏成功后执行（`TargetManagement.kt:50-54`），所以失败或超时后设备上一定**没有租约**，不会留下 main 租约。候选的漏洞在主机：失败后下一条命令会被隐式授予 main。第 1 条修好之后，这个漏洞就没了。另外建议把 `publish` 与 `done` 事实放进同一个 `synchronized(run)`，使设备对 open 的回答只有 done 或 failed 两种（见 §2.3）。
4. **不依赖显示的命令推荐方案 (a)：一律要求已有租约。**shell、push/pull、notifications、clipboard、apps、wake 都在内。今天这些命令本来就隐式取得**同一个**租约，并靠它做“一台设备一个主人”的互斥（基线与候选相同）。(a) 只是去掉“隐式”二字，代码最少、规则只有一句。已开始任务的后续操作不要求租约，今天就是如此，保持不变：`job get/cancel/output`、`file.get/cancel/chunk/finish`。`status`、`devices`、`pair` 同样不要求。
   代价是纯 shell 或纯文件任务也必须先 `session open`。这改变了基线行为，**列为需要用户确认的取舍**（§9 Q1）。
5. **保留 `sessionReleased`，不要并入 `noSession`。**设备侧“释放当前会话”表示真人要求 agent 停下。如果统一成“没有会话，请重开”，LLM 会照做重开，把真人的释放覆盖掉。这个标记和错误码在基线就已存在（基线 `Sessions.kt:52-54`），不算新增状态。
6. **统一错误**：`code=noSession`，`result=not sent`，exit 2，零副作用。文案：`No open session; run session open (add --display virtual for a work display). Previous refs and images are invalid.` batch 不需要预检：首步就会零副作用地被拒；中途丢失租约时，该步报 `noSession`、`sessionChanged`、`sessionReleased` 或 `busy`，batch 照常停下。
7. **规模估算**【建议】：host 源码约 −230/+40，host 测试约 −560/+150，Kotlin 源码约 −90/+35，Kotlin 测试约 −120/+160，契约约 −130/+40，文档约 −150/+60。净减约 900 行，同时消除 22 个 Linux 下失败的偏好测试（依据底稿 12 的实测计数）。
8. **同名会话无法保护**：两个 agent 共用同一个 `-s` 名字（含 `default`）时，设备把它们视为同一主体，B 的 `session open` 会替换 A 的后台屏。这是会话名语义本身的问题，基线也一样。处理办法是写进指引：“每个任务用自己的名字”。是否要加一道硬限制列为 §9 Q3。

---

## 1. 命令清单与现状（候选 3f45af6）

列说明：“租约”指对 `Sessions.current` 的处理方式；“显示”指 `RpcMethods.requiresDisplay` 是否为真（为真时主机必须带上 `expectedDisplayKind`）；“互斥”指登记哪类 `Resource`，以及 `changing` 是否为真（为真时，任何其它在途资源都会导致 busy）。

| 命令（CLI → RPC） | 今天对租约的行为 | 显示 | 资源互斥 | 位置 |
|---|---|---|---|---|
| UI 读：snapshot/locate/wait/screenshot/zoom | 无租约则**隐式新建 MAIN**；别的主体持有则 busy；偏好为 virtual 时先报 `displayRequired` | 是（必填 kind） | UI，changing=false（与 shell/file 共存，与其它 UI 互斥） | `UiBackend.kt:99-145`（`:107-108`）；`Sessions.kt:136-150`，`keepLease` 在 `:90-97`；`RpcMethods.kt:19,48-56` |
| UI 写：click/fill/tap/key/type/open/back/home/recents/bare notifications/… | 同上 | 是 | UI，changing=true（任何在途资源都 busy） | 同上 |
| IC：editor.commit/select | 同上；virtual 时报 `displayUnsupported` | 是 | UI，changing=true | `UiBackend.kt:239-243` |
| `ui.read`（动作早返回后的附件分块读取） | **只用已有租约**（`acquireExistingUi`）；没有租约时报 `captureUnavailable`；kind 不符报 `displayMismatch`；不登记资源，但会续 idle | 是 | 无 | `UiDelivery.kt:366-371`；`Sessions.kt:78-88` |
| batch | 主机在首步前读一次偏好并冻结；每一步按上面各行处理 | 含目标步时是 | 逐步 | `ui_batch.py:307-319`；`ui_client.py:594-607` |
| shell（前台、`--background`）→ `shell.start` | **隐式新建 MAIN**（`acquire(owner)`），别的主体持有则 busy | 否 | SHELL，changing=true。前台占用到首次执行结果；后台在 GO 之后释放（`ShellJob.kt:120`）；GO 前要核 `isCurrent(lease)`（`ShellJob.kt:106-118`） | `ShellJobs.kt:33-34`；`job_commands.py:546` |
| `job get` / `job cancel` / `job output` | **不要求租约**，按 job 的 owner `(pairing,name)` 核对 | 否 | 无 | `ShellJobs.kt:55-80`；`JobTable.kt:26-33` |
| push/pull（host 任务 `t` ID → 设备 `file.begin`） | `file.begin` **隐式新建 MAIN**；host 任务归属 `(device,pairing,session)` | 否 | FILE，changing=true，一直占到传输结束；占用期间租约不会到期（`Sessions.kt:243`） | `FileJobs.kt:24-25`；`FileJob.kt:37`；`file_jobs.py:214`；`coordinator.py:499-512` |
| `file.get/cancel/chunk/finish` | 不要求租约，按 owner 核对，只检查撤销 | 否 | 沿用原 FILE 资源 | `FileJobs.kt:31-46`；`FileJob.kt:184-187` |
| notifications list/get | **隐式新建 MAIN** | 否 | UI，changing=false | `RpcMethods.kt:19`；`UiBackend.kt:185-193` |
| notifications action/open/dismiss/reply | 同上（用户已决定首次合入删除这一组） | 否 | UI，changing=true | `RpcMethods.kt:23` |
| clipboard get / set / clear | **隐式新建 MAIN** | 否 | UI；get changing=false，set/clear changing=true | `UiBackend.kt:176-184` |
| apps | **隐式新建 MAIN** | 否 | UI，changing=false | `UiBackend.kt:195-198` |
| wake | **隐式新建 MAIN**；在 virtual 租约下也可用 | 否 | UI，changing=true | `SystemActions.kt:40-47,82-84` |
| status | 不取租约；返回 `callerState`（active/released/none）和 `target` | 否 | 无（读状态时会顺带触发 `expire()`） | `WorkerService.kt:66-67`；`Sessions.kt:256-279` |
| devices / pair（`pair.register`） | 只在主机或 Gateway 层处理，不进 worker 租约 | 否 | 无 | `app.py:360,383`；`NetworkGateway.kt:313` |
| session open | `prepareChange`：同主体可替换，别的主体 busy，要求无在途资源；旧租约**立即**失效；MAIN 直接 publish | — | 管理用的 UI 资源 | `app.py:737-756`；`TargetManagement.kt:12-36,38-98`；`Sessions.kt:99-134` |
| display create | 同上，target=VIRTUAL；建屏成功才 publish；主机只在 `done` 时写 `virtual` 偏好 | — | 同上 | `app.py:815-866`；`TargetManagement.kt:28-29,100-105` |
| display select | 只写主机 `display.json`，不联系设备租约 | — | 主机文件锁 | `app.py:780-805`；`workspace.py:23-111` |
| session close | `prepareChange` 使旧租约失效，不 publish；没有租约时仍返回 done | — | 管理用的 UI 资源 | `app.py:760`；`TargetManagement.kt:48,58` |
| session takeover | 允许同 pairing 的其它会话名；固定建 MAIN；idle 固定为默认值 | — | 同上 | `app.py:770`；`TargetManagement.kt:18-25,30` |
| 设备侧“释放当前会话” | 核对 leaseId，标记 `released`，使租约失效；在途资源继续 | — | 不检查资源 | `GatewayService.kt:267-272`；`WorkerService.kt:85-94`；`Sessions.kt:172-177` |
| 设备侧 Stop / Force exit | Stop 要求没有未结束的 job，然后销毁 worker；Force 直接销毁。新 worker 里没有租约，也没有 `released` 标记 | — | — | `WorkerConnection.kt:217-247`；`WorkerService.kt:115-134` |
| 撤销配对 | 使该 pairing 的租约失效，取消其 job | — | — | `WorkerService.kt:96-113` |

基线对照【基线】：基线没有 kind，也没有 TargetManagement。UI、shell、file、wake、apps 全部经 `acquire` 隐式取得 MAIN（基线 `Sessions.kt:48-60`，`UiBackend.kt:66`，`ShellJobs.kt:33`，`FileJobs.kt:24`）。基线的 `ui.read` 也会隐式取得租约（基线 `UiDelivery.kt:142`）。`released` 拒绝隐式重取的规则基线就有。**“隐式租约”是基线的产品行为，不是候选引入的。**

---

## 2. 模型定义

### 2.1 状态机【建议】

设备上只有一个 `current: Lease?`，取值是三态：`none`、`main`、`virtual(displayId)`。另外有两个按主体保存的内存标记，都已存在，维持不变：`released`（主体被设备侧释放）和 `revoked`（pairing 被撤销）。

| 事件 | 前置条件 | 结果 |
|---|---|---|
| `session open --display K [--idle D]` | 未停止；pairing 未撤销；current 为空或属于同一主体；没有任何在途资源 | 旧租约（如有）立即失效并开始清理（后台屏随之销毁）。K=main：立即发布 main。K=virtual：建屏成功才发布，否则保持 none。同时清除本主体的 `released` 标记 |
| `session takeover --display K [--idle D]` | 同上，但允许 current 属于**同 pairing 的其它会话名**；本主体被 released 时拒绝（保留现规则） | 同 open |
| `session close` | current 为空或属于本主体；没有在途资源 | 有租约则失效并清理，没有则 done（幂等）。返回 done |
| idle 到期 | 没有在途资源，且 `now − touched ≥ idle` | → none，后台屏销毁 |
| 设备侧释放 | 管理页确认的 leaseId 仍是当前租约 | → none，并给该主体加 `released` 标记；在途 shell/transfer 继续 |
| 撤销配对 | — | → none（若属于该 pairing），并取消该 pairing 的 job |
| worker 重建（Stop/Force/崩溃 → Refresh） | — | 新 worker 从 none 开始，`released` 标记丢失，后台屏随旧进程消失 |
| Gateway 关闭再打开 | — | 租约不受影响，idle 照常计时；重开后如已到期则为 none |
| daemon 重启 | — | 设备不受影响。主机不保存任何会话状态，后续命令直接继续使用设备上的租约 |

**谁能创建**：只有 `session.open` 和 `session.takeover`（同一段代码）。删除 `keepLease` 的新建分支后，`Sessions` 里不再有其它路径会把 `current` 从 null 改成非 null。`publish` 是唯一的赋值点。

**`--display` 缺省值：建议缺省为 `main`，不设为必填。**理由如下：
- 产品默认面就是主屏。后台屏是可选能力，而且在拿到真机证据前还是实验能力。
- 纯 shell 或文件任务（方案 a 下也要 open）若被迫选一个毫无意义的显示种类，只会增加噪声。
- 候选的事故不是出在“开会话时选错种类”，而是出在“租约丢了之后被静默改成 main”，后者已被 §0-1 消除。
- 漏写 `--display virtual` 时，open 的回执和随后每一条 UI 结果都**不会**出现 `display=virtual#N`，这个错误能当场看出来。
- 基线及所有现存脚本里的 `session open` 写法保持有效。

这是一个小取舍：必填能多防一次“忘写参数”，但代价落在每一次 open 上。我把它列为 §9 Q2，推荐缺省 main。

**`--idle`**：沿用现规则，1s–24h，缺省 10m，两种显示同一套规则。takeover 也开放这个选项，候选里 takeover 的 idle 被写死，见 `TargetManagement.kt:18-25`。

**已有租约时再执行 `session open`**（沿用基线“open 总是新建租约”的语义）：

| 情况 | 行为 | 返回 |
|---|---|---|
| 同主体、同种类、无在途资源 | 替换：旧租约失效，引用和图像清空；若为 virtual，**旧后台屏被销毁**并新建一块 | done；`data.previousSession=<name>`，`data.target={kind,displayId}`；文本 `refs and images cleared` |
| 同主体、不同种类 | 替换。virtual→main 时销毁后台屏；main→virtual 时若建屏失败，结果为 **none** | 同上，失败见 §2.3 |
| 同主体、有在途资源（前台 shell、传输、UI、未完成的清理） | 拒绝，现状 `Sessions.kt:116` | not sent，`busy: Active shell must finish` |
| current 属于别的会话名 | 拒绝 | not sent，`busy: <pairing>/<name> idle=Ns` |
| 主体被设备侧释放过 | open 允许，并清除标记（显式覆盖）；takeover 拒绝 | — |

### 2.2 租约与资源的关系（不变）

`Resource(lease, kind)` 及其互斥规则保持不变（`Sessions.kt:198-202`）。资源不为空时租约不会到期（`:243`），也不能被 open、close、takeover 替换（`:116`）。设备侧释放和撤销可以在资源在途时使租约失效，资源由原执行者自行收尾。

### 2.3 `session open --display virtual` 的原子性

现状【源码】：
1. `TargetManagement.execute` 调 `prepareChange`。这一步登记管理资源、**同步**使旧租约失效（`Sessions.kt:117-121`），然后进入 executor。旧租约的 `ui.clear` 先于本次变更入队，用的是同一个 executor（`UiBackend.kt:43,477-500`），因此一定先执行。
2. `changeTarget` → `createTarget` → `WorkDisplay.create`。成功后在 `synchronized(run){ checkActive(); publish }` 里发布（`TargetManagement.kt:49-54`）。`checkActive` 在请求已经回复之后会抛 `requestEnded`（`UiRun.kt:92-95,126-127`），所以**回复之后不可能再发布**。
3. 没有发布时，`finally` 关闭 stream（`TargetManagement.kt:83-94`）。

由此得到不变量（现状已成立）：**open virtual 结束后，设备上要么是该主体的 virtual 租约，要么什么都没有，绝不会是 main。**

需要补的一处【建议】：现在 `run.remember(done)` 位于 publish 的同步块**之外**（`TargetManagement.kt:56-60`）。如果回复恰好落在“已发布”和“已记 done”之间，回执仍是 `UNKNOWN: Work display creation pending`。修法有两步：
- 把 `remember(done)` 挪进 `synchronized(run)`，紧跟 `publish` 之后；
- 把初始事实和超时 catch 从 `UNKNOWN` 改为 `FAILED`，文案为 "Session was not opened: request ended before the target was ready"。理由是回复之后不可能再发布。

这样设备对 open 只会答 **done** 或 **failed**。只有传输丢包时主机才会给出 UNKNOWN（exit 3）。

**结果未知时客户端怎么确认**：执行 `status`，按下表判断。
- `callerSession=active` 且 `display=virtual#N`：成功，继续。
- `callerSession=none`：失败。若同时 `ui>0`，表示建屏后的清理还在进行，等它归零后重新 `session open --display virtual`；直接重开会得到 `busy`，而不是错误地落到别处。

不建议用“再 open 一次”来确认：如果上一次其实成功了，再 open 会销毁刚建好的屏。

**建屏失败如何说明缺什么**：错误码沿用现有的
- `displayUnavailable`：主屏未亮、未解锁，或新屏状态不是 ON，见 `DeviceFacts.mainReadyForWorkDisplay` 与 `WorkDisplay.kt:47,77`；
- `displayUnsupported`：平台未建屏，或未保留 TRUSTED、OWN_FOCUS、STEAL_TOP_FOCUS_DISABLED，见 `WorkDisplay.kt:61,69`；
- `uiCleanup`、`busy`。

**给能力探针留的接口**：`WorkDisplay.create` 在失败时抛出带结构化数据的 `Rejected`，数据挂在 `data.workDisplay`：

```
{stage: "mainReady|create|flags|state", requestedFlags, actualFlags, missingFlags: ["OWN_FOCUS", ...], displayState}
```

现有 `data.createdDisplay`（`TargetManagement.kt:64-73`）并入这个对象。探针命令（由另一位评审员设计）调用同一个 `WorkDisplay.create` 加 `close`、但不 publish，输出同一个对象。open 失败的文本行打印 `missing=OWN_FOCUS,...`，并补一句 `no session is open`。

### 2.4 统一的“没有会话”错误

| 码 | 何时 | result / exit | 文案（设备 reason；主机原样打印） | agent 应做 |
|---|---|---|---|---|
| `noSession` | 请求进入准入时，本主体没有当前租约，也没有在途资源 | not sent / 2 | `No open session; run session open (add --display virtual for a work display). Previous refs and images are invalid.` | 按原意图重开，然后重新观察 |
| `sessionReleased` | 主体被设备侧释放（已有） | not sent / 2 | `The device user released this session; stop unless the user asks you to continue (then run session open).` | 停下，向用户确认 |
| `busy` | 别的主体持有租约；或本主体没有租约但还有在途、清理中的资源（open 进行中、清理未完成） | not sent / 2 | 保持现文案 | 等待，或改用自己的会话名 |
| `sessionChanged` | **请求执行中**租约被替换或失效（已有，`Sessions.kt:213`） | 视派发情况：not sent、failed 或 UNKNOWN | `The session ended or changed during this request; run status, then session open if needed` | 按回执的派发事实处理，不要直接重发 |

准入顺序（`Sessions.acquire*`）：
1. stopping；
2. 名字或撤销校验；
3. `expire()`；
4. `released` → `sessionReleased`；
5. 别的主体持有 → `busy`；
6. （UI）`uiCleanup`；
7. （UI）`requireResources(changing)` → `busy`；
8. 没有租约：有资源 → `busy`，否则 → `noSession`。

**batch**：删除主机预检（`ui_batch.py:307-319`）。预检必然有竞态，而首步的 `noSession` 本来就是零副作用的预检。中途丢失租约时，该步返回上表某个码，`stopping` 为真，batch 停下，并保留已执行的前缀和该步已有的附件。这一机制现在就存在，不需要新代码。

### 2.5 `ui.read` 在租约刚结束时

【建议】`UiDelivery.read` 改为调用 `sessions.acquire(owner)`：不新建租约，不登记资源，但续 idle。这等同于删掉 kind 参数后的 `acquireExistingUi`，两个函数合并为一个。
- 租约已结束：`noSession`。
- 本主体已重开：`captureUnavailable`（capture 属于旧租约，见 `UiDelivery.kt:407`）。

主机端 `blob()` 会把非 done 转成 `Fault("captureUnavailable", reason)`（`ui_client.py:704-705`），`attachments()` 写入 `snapshotError`、`windowsError`、`imageError`，**动作结果和 exit 都不变**（`ui_client.py:683-685`）。文本因此会显示动作 done，同时附 `snapshotError=No open session; ...`。这已经足够，不需要新字段。

### 2.6 REF、图像帧、坐标资格与租约

【源码】租约一失效，`clearLeaseUi → UiBackend.clear` 就会清空 `delivery`、`snapshots`、`visuals`（`UiBackend.kt:477-500`），REF、当前图和坐标资格随之全部失效。ref 计数在 close、idle 时**不**重置，worker 重启才重置（连接文档 §会话与释放），所以同一个 worker 内，旧 REF 不会和新租约的 REF 撞号。旧 REF 会得到 `staleRef: Ref is absent from the current observation ... run snapshot`（`UiSnapshot.kt:76-83`）。worker 重启后的首次观察会附上 `New worker: previous refs are invalid`（`UiSnapshot.kt:68-72`）。

表达方式【建议】：不新增机制。open 的回执文本写 `refs and images cleared`（基线就是这句，候选改成了 "Previous target resources cleared"，应改回）。`noSession` 文案里已说明旧 REF 失效。

---

## 3. 不依赖显示的命令怎么处理会话（重点）

### 3.1 今天这些命令靠租约做了什么【源码 + 基线】

1. **一台设备一个主人**：别的主体持有租约时，shell、file、notifications、clipboard、apps、wake 都会得到 `busy`（`Sessions.kt:74`）。这是基线的产品规则：“其它 owner 不能读完整屏幕或变更，仅可查最小状态”（`docs/reference/connection.md` §会话与释放）。shell 能做任何事（`input tap`、`am start`、`cmd clipboard`），通知和剪贴板里是隐私内容，所以这层互斥是实质性的。
2. **资源账本挂在租约上**：`Resource(lease, kind)`。`FileJob.start` 在租约内取 begin 许可（`FileJob.kt:37`）；`ShellJob.ready` 在 GO 前核 `isCurrent(lease)`（`ShellJob.kt:106-112`）；释放或撤销后，排队中的工作不会开始。
3. **归属不靠租约**：后台 shell job 和文件任务的 owner 是 `(pairing, name)`（`JobTable.kt:31`；host 端 `file_jobs.py:634-639` 是 `(device,pairing,session)`）。后续查询、取消、chunk、finish 都不要求租约。

### 3.2 方案推演

**(a) 一律要求已有租约**（`job/file` 后续操作、`status`、`devices`、`pair` 除外）

- 静默落主屏：不会。
- 多 agent：与今天完全相同（busy）。
- 长 shell job 跨租约：后台 job 继续运行，原主体不需要租约也能查询或取消；新开 shell 需要先 open。前台 shell 运行期间占着资源，租约不会到期，与今天相同。
- 心智负担：一条规则，“先 open，再做任何设备操作”。
- 代码：最少。把 `keepLease` 的新建分支换成 `noSession`；ShellJobs、FileJobs、UiBackend、UiDelivery 的调用点**一行不改**，因为它们都已经调用 `acquire` 或 `acquireUi`。
- 副作用：纯 shell 或纯 push/pull 任务也要先 `session open`。这一步在设备上没有可见效果：main 的 open 不连接 UiAutomation、不碰屏幕，`createTarget` 对 MAIN 直接返回（`TargetManagement.kt:101`）。
- 一个容易忽略的边角：`session open --display virtual` 要求主屏亮着且已解锁。主屏熄灭时无法先 `wake`，因为 wake 本身也要租约。这不构成问题：后台屏的使用前提就是真人正在用手机，屏幕是亮的。open 失败的文案应写“main display is off or locked; work displays need the phone in active use — use --display main, or ask the user to unlock”。

**(b) 这些命令都不需要租约，按设备级请求执行**

- 如果连“别的主体持有就 busy”也不检查，就丢掉了 3.1-1 的互斥：A 在后台屏工作时，B 可以用 shell 往主屏注入 `input tap`，也可以读 A 任务中途收到的通知验证码。这是能力退化。
- 如果保留这项检查，它就退化成“设备空闲时允许免租约执行”。结果是资源账本要支持“无租约资源”：`Resource.lease` 改成 owner，`finished()` 的 touch、`FileJob.start` 和 `ShellJob.ready` 的租约核对都要改写，代码比今天还多。
- 静默落主屏：UI 路由不受影响。但 `wake` 会在没有任何会话时点亮用户主屏，`clipboard set` 会改写用户剪贴板，而且都不经过显式会话。
- 长 job 的归属与 (a) 相同。
- 心智负担：“哪些要 open、哪些不要”需要一张表。
- 不推荐。

**(c) 分两类：改变设备状态或占用执行资源的命令要求租约，只读设备查询不要求**

- 分类：要求租约的是 shell、file.begin、clipboard set/clear、wake；不要求的是 apps、notifications list/get、clipboard get。
- 只读查询照样要读隐私（通知、剪贴板），所以“别的主体持有则 busy”必须保留。于是 (c) 与 (a) 的唯一差别是：**设备空闲时**可以不开会话直接读 apps、通知和剪贴板。
- 代码：这几条读命令现在走 UI executor 并登记 UI 资源（`UiBackend.kt:108`），无租约时要另做一条“无租约只读”路径，大约 +30 行，外加测试。
- 收益很小：读通知或剪贴板几乎总发生在一个 UI 任务中途，那时会话本来就开着；apps 列表同理。
- 心智负担：需要记住分类，而且 clipboard get 与 set 规则不同，容易混淆。
- 不推荐。

**比较**

| | 静默落主屏 | 多 agent 互斥 | 后台 job 归属 | LLM 负担 | 代码 |
|---|---|---|---|---|---|
| (a) | 无 | 保持 | `(pairing,name)`，不变 | 1 条规则 | 最少（删分支） |
| (b) | UI 无；wake、clipboard 可在无会话时动设备 | 弱化，或付出代码代价保留 | 不变 | 需要分类表 | 最多 |
| (c) | 无 | 保持 | 不变 | 需要记分类 | 中等 |

**推荐 (a)。**它就是用户“任何命令”的字面意思，代码最少，并且原样保留基线“一台设备一个主人”的语义。唯一真实的代价是纯 shell 或纯文件任务多一条 `session open`，而它在设备上没有可见效果。这改变了基线用法，需要用户确认，见 §9 Q1。我的推荐不变。

---

## 4. 情形推演表

“主屏动作”指：没有收到拒绝的情况下，在用户主屏上产生输入、启动、亮屏或剪贴板改动。

| # | 情形 | 新模型：发生什么 / 返回 | 新模型主屏动作 | 候选现状 |
|---|---|---|---|---|
| 1 | 正常主屏 | `session open` → done `display=main`；之后的 UI 落在主屏 | 是（本意） | 同；但不 open 也能用（隐式） |
| 2 | 正常后台屏 | `session open --display virtual` → done `display=virtual#N`；UI 落在 N；每条 UI 结果的屏幕行带 `display=virtual#N` | 否 | `display create` 写主机偏好；每个请求带 kind |
| 3 | 后台屏 idle 到期 | 租约 → none，后台屏销毁；**任何**命令都得到 `noSession` | 否 | UI 报 `displayRequired`；但 shell、apps、wake、clipboard、notifications 会**隐式取得 MAIN**，之后 UI 报 `displayMismatch`；wake 会点亮主屏 |
| 4 | 主屏 idle 到期 | `noSession`；agent 重开后重新观察 | 否 | 静默新建 main，旧 REF 失效，selector 动作直接执行 |
| 5 | worker 重建 | `noSession`；首次观察附 `New worker` | 否 | 偏好 virtual → `displayRequired`；偏好 main → 隐式 main |
| 6 | open virtual 失败 | failed，带 `code=displayUnsupported/Unavailable`、`data.workDisplay`、`no session is open`；之后的命令都得到 `noSession` | 否 | 偏好保持 main 时，下一条 snapshot **隐式取得 main**（API37 上真实发生过 VIEW 落在 display0，见底稿 02 F1） |
| 7 | open virtual 结果未知 | 设备侧只会答 done 或 failed（§2.3 修正后）；主机丢包 → UNKNOWN exit 3 → `status`：active+virtual 则继续，none 则重开 | 否 | 设备可能答 UNKNOWN；主机不写偏好 → 同第 6 行的漏洞 |
| 8 | close 后再发命令 | `noSession` | 否 | 偏好 main → 隐式 main；偏好 virtual → `displayRequired` |
| 9 | 已有 virtual 时 open main | 销毁后台屏（迁入的 task 可能丢失），发布 main；回执带 `previousSession` | 之后的 UI 落在主屏（本意） | `session open` 建 main，偏好仍为 virtual → 下一条 UI 报 `displayMismatch`，提示又让再 open，形成循环（底稿 02 F3） |
| 10 | 已有 main 时 open virtual | 成功 → virtual；失败 → none | 否 | 同（create 路径） |
| 11 | 两个 agent 用同一会话名 | 设备视为同一主体。B 的 open 替换 A 的租约（后台屏被销毁）；A 之后的新 snapshot 落到 B 的租约上，A 的旧 REF 得到 `staleRef` | 可能（B 开 main 后 A 继续发 selector 动作），这是会话名冲突 | 同，另外还共享主机偏好 |
| 12 | 不同会话名 | 后来者得到 `busy: pairing/name idle=Ns` | 否 | 同 |
| 13 | 默认会话名 `default` | 与普通名字相同；任务之间没有遗留状态 | 否 | 上一个任务遗留的 virtual 偏好会让后续任务被 `displayRequired` 卡住，提示还引导去建屏（底稿 02 F2） |
| 14 | 跨工作目录 | 主机没有会话状态，行为完全一致 | 否 | 偏好跨 workspace 共享，一处修改影响其它 workspace |
| 15 | batch 中途丢失租约 | 该步得到 `noSession`、`sessionChanged`、`sessionReleased` 或 `busy`，batch 停下，保留已执行前缀 | 否 | 偏好 main 时，后续步骤会隐式取得新 main 租约，**selector 步骤照常执行** |
| 16 | `ui.read` 晚到 | `noSession` 或 `captureUnavailable` → 写入 `snapshotError`、`imageError`；动作结果不变 | 否 | `captureUnavailable` / `displayMismatch`，效果相同 |
| 17 | 后台 shell job 跨会话 | close、到期、释放都不影响 job；原主体不需要租约即可 `job get/cancel/output`；新开 shell 需要先 open | 否（job 自己的副作用除外） | job 行为相同；新开 shell 会隐式取得 main |
| 18 | 文件传输跨会话 | 传输期间 FILE 资源阻止到期、close、open（busy）；设备侧释放后传输继续，chunk 和 finish 照常；之后新的 push 需要先 open | 否 | 相同；新的 push 会隐式取得 main |
| 19 | daemon 重启 | 设备上的租约保留，命令直接继续；host 侧传输事实丢失（现状如此） | 否 | 同，另加偏好文件仍然生效 |
| 20 | Gateway 关闭再打开 | 租约保留，idle 照常计时；重开后已到期则 `noSession` | 否 | 到期后隐式重取（种类取决于偏好） |
| 21 | 设备侧释放 | 后台屏销毁；任何命令得到 `sessionReleased`；`status` 显示 `callerSession=released`；agent 只有在用户要求时才 `session open` | 否 | 相同（`acquire` 本来就检查 released） |
| 22 | takeover | `session takeover [--display K]`：同 pairing、无在途资源时接管，旧租约（含后台屏）结束；原持有者之后得到 `busy` | 视 K 而定（本意） | 固定建 main；偏好为 virtual 时，接管后下一条 UI 报 `displayMismatch` |

---

## 5. 要删什么、要加什么（按文件）

### 5.1 Android（`android/app/src/main/java/com/rainng/androidctl/`）

| 文件 | 删 | 改 / 加 | 行数估计 |
|---|---|---|---|
| `Sessions.kt` | `keepLease`（`:90-97`）的新建分支；`acquire` 的 `expected` 参数（`:56-64`）；`acquireExistingUi`（`:78-88`）；`requireTarget`（`:152-170`）；`acquireUi` 的 `expected` 参数 | `acquire`、`acquireUi` 在 `held==null` 时调用 `noLease()`：先 `requireIdle()`（busy），再 `throw Rejected("noSession", …)`；`sessionReleased` 文案改为“用户释放，停下” | −45 / +10 |
| `DisplayTarget.kt` | `DisplayKind.requested(args)`（`:13-16`） | `DisplayKind.of(wire)`，用于解析 `session.open` 的 `display` | −4 / +3 |
| `TargetManagement.kt` | `display.create` 分支（`:20,28`，`managementName` 的 else 分支） | 目标种类改由 `args.getString("display")` 决定（open 和 takeover）；takeover 也读 `idleSeconds`；`remember(done)` 移进 `synchronized(run)`；未发布时一律报 FAILED（§2.3）；回执文案改为 `refs and images cleared`；建屏失败时附 `data.workDisplay` | −10 / +15 |
| `RpcMethods.kt` | `requiresDisplay`（`:48-56`）；`SESSION_COMMANDS` 中的 `display.create` | — | −10 |
| `UiBackend.kt` | `:107` 的 kind 解析；`acquireUi` 调用中的 `expected` 实参 | — | −2 |
| `UiDelivery.kt` | `:370` 的 `acquireExistingUi(owner, DisplayKind.requested(...))` | 改为 `sessions.acquire(owner)` | ±1 |
| `CurrentEditorInput.kt:18,20`、`ImageRegion.kt:67` | 参数白名单中的 `"expectedDisplayKind"` | — | −3 |
| `WorkDisplay.kt` | — | 失败时抛出带 `stage/missingFlags` 的结构化 `Rejected`（供探针复用） | +10 |
| 不动 | `ShellJobs.kt`、`FileJobs.kt`、`ShellJob.kt`、`FileJob.kt`、`WorkerService.kt`、`GatewayService.kt`、`WorkerConnection.kt`、`SystemActions.kt`（virtual 下拒 home/recents/notifications 的逻辑保留）、`ActivityLaunch.kt`（virtual 落点判定另议，见底稿 02 F4）、管理页、FGS 通知 | | 0 |

`prepareChange`、`publish`：**保留**，它们就是原子 open 的实现。`Lease.target`：保留。`released`：保留。

### 5.2 Host（`androidctl/src/androidctl/`）

| 文件 | 删 | 改 / 加 | 行数估计 |
|---|---|---|---|
| `workspace.py` | `DisplayKind`、`display_path`、`read_displays`、`read_display`、`save_display`（`:16-111`）及多余的 import | — | −100 |
| `app.py` | `display_app` 注册（`:65-71`）；`display_select`（`:774-805`）；`display_create`（`:808-866`） | `session_open` 加 `--display [main\|virtual]`（缺省 main），发送 `{"idleSeconds", "display"}`；`session_takeover` 加 `--display` 和 `--idle`；`emit_status` 把 target 合进 session 行：`session="task" pairing="host" display=virtual#12 idle=3s`；session=none 时追加 `run session open` | −100 / +25 |
| `ui_client.py` | `uses_display`（`:70-71`）；`expected_display_kind`、`freeze_display` 以及 `call` 中的注入（`:594-607`） | — | −18 |
| `ui_batch.py` | 预检冻结（`:307-319`） | — | −14 |
| 结果渲染（`ui_view.py` 的 screen 行或 `ui_client` 的 result line） | — | 观察到的 `target.displayId != 0` 时，在屏幕行加 `display=virtual#<id>`；主屏不加 | +6 |

### 5.3 测试

- 删除：`androidctl/tests/test_display.py`（567 行，其中大部分是偏好文件测试）；`test_wire_contract.py::test_display_routing_and_public_request_samples_match_the_shared_contract`；`test_ui_batch.py` 中的冻结相关部分（8 处）；`test_cli_arguments.py` 中 `display` 命令的 2 处。
- 新增：host 侧约 120–150 行，见 §8。
- `DisplayTargetTest.kt`（192 行）重写为 open、close、noSession 的状态机测试，约 150 行。
- `SessionsTest.kt` 中 35 处 `acquire*` 调用：先 open 再取；无 open 的用例改为断言 `noSession`。
- 其它 12 个测试类通过共享 helper `opened(owner)` 取得租约，涉及 `BorrowedPasteTest`、`CurrentEditorInputTest`、`JobTableTest`、`KeySequenceTest`、`NotificationReadTest`、`PointerGestureTest`、`UiConditionTest`、`UiDeliveryTest`、`UiOutcomeTest`、`UiScrollUntilTest`、`UiTreeBudgetTest`、`UiWindowObservationTest`。

### 5.4 `display close` 与 `session close`

候选里没有 `display close`，只有 `display select` 和 `display create`，两者都删除。后台屏只随租约生灭：
- 关闭后台屏但保留会话：`session open --display main`，一条命令原子替换；
- 结束一切：`session close`。

不提供“后台屏关闭、租约保留”的中间态，因为它的语义就等于 main 租约。

### 5.5 文档

| 文件 | 改动 |
|---|---|
| `skills/androidctl/SKILL.md` §Start A Task | 换成 §7.2 的文字 |
| `skills/androidctl/displays.md` | 删掉偏好、select、create、冻结相关段落；从 90 行压到约 35 行 |
| `skills/androidctl/connection.md` 会话段 | 补一句“任何设备命令都需要已开会话” |
| `docs/reference/connection.md` | “接入到首次业务”第 3 步；“会话与释放”首段；“目标显示与工作区选择”整节重写 |
| `docs/architecture.md:58,84,86` | 相应改写 |
| `docs/reference/ui.md:5` | 相应改写 |
| `docs/reference/jobs-files.md:11` | “取得租约”改为“要求已开租约” |

---

## 6. 协议与契约变化

**Wire：**
- `session.open`、`session.takeover` 的 args：`{idleSeconds: int, display: "main"|"virtual"}`，两项都**必填**。主机总会发送；设备不在线上做缺省。`session.close` 的 args 为空。
- 删除方法 `display.create`。
- 所有 UI 请求和 `ui.read` 的 args 删除 `expectedDisplayKind`。
- 错误码新增 `noSession`；删除 `displayRequired`、`displayMismatch`。保留 `sessionReleased`、`sessionChanged`、`busy`、`displayUnsupported`、`displayUnavailable`、`uiCleanup`、`captureUnavailable`（仅用于 capture 不属于当前租约的情况）。
- open 回执的 `data`：`target{kind,displayId}`、`previousSession`；virtual 时附 `requestedFlags/actualFlags`；失败时附 `workDisplay{stage,requestedFlags,actualFlags,missingFlags,displayState}`，取代 `createdDisplay`。
- 设备对 open 和 takeover 不再回答 UNKNOWN。
- status 的 `session.target`、`callerState` 保持不变。观察的 `target.displayId` 保持不变；主机靠 `displayId != 0` 判定 virtual，不新增 wire 字段。

**`contract/wire-contract.json` 受影响的类别：**
- 删除 `displayRouting` 整节（约 56 行）；
- 从下列类别的请求样本中删掉 `expectedDisplayKind`：`actionResults` 37 处、`waitResults` 10 处、`editorResults` 7 处、`compoundResults` 6 处、`pasteResults` 7 处（借用粘贴整节将按用户决定删除）；
- 新增 `session.open` 的 main、virtual 成功样本和 virtual 失败样本，以及 `noSession` 拒绝样本；
- `queryMethods` 不变。

重新生成流程按 AGENTS.md：Kotlin 导出候选 → 人工审 → 替换 → 两端校验。

---

## 7. 对基线行为的破坏性变化

### 7.1 今天能跑、改后会得到 `noSession` 的调用

1. 不先 `session open` 的任何 UI 命令，包括 `snapshot`、`open`、`tap`、`wait`、`screenshot`。
2. 不先 open 的 `shell`、`shell --background`、`push`、`pull`。
3. 不先 open 的 `apps`、`wake`、`clipboard *`、`notifications list/get`。
4. 租约 idle 到期（默认 10 分钟）、`session close`、worker 重建之后的**任何**设备命令。基线下这些都会静默续上。
5. 仍然免租约的：`status`、`devices`、`pair`、`device *`、`daemon *`、`use`、`job get/cancel/output`、push/pull 的进度查询与取消（`job get/cancel` 对 `t` ID）。

候选新增而将被删除的：`display select`、`display create`、`display.json`。这些不属于基线，不构成对外破坏。

### 7.2 SKILL.md “Start A Task” 替换文字（≤10 行）

```
Use `devices` for an existing pairing, or pair with the supplied address/code. Choose one
session name per task (not `default` when others may share the device) and open it before
any device command, including shell, files, apps, clipboard and wake:

androidctl --device DEVICE -s task session open                    # main screen
androidctl --device DEVICE -s task session open --display virtual  # background work display
androidctl --device DEVICE -s task snapshot

`noSession`: the session ended (idle, close, worker restart). Reopen with the same --display,
then observe again; old refs are invalid. `sessionReleased`: the phone's user released it; stop
unless told to continue. Respect `busy`; takeover needs explicit user instruction. End with `session close`.
```

### 7.3 `status` 应显示什么

```
device="lab-phone" connected worker=ready
session="task" pairing="host" display=virtual#12 idle=3s
callerSession=active
ui=0 shell=0 file=0
```

- 没有租约时显示 `session=none (run session open)`，并显示 `callerSession=none|released`。
- 删除候选里单独输出的 `target={...}` JSON 行，把它并进 session 行。
- 不显示任何主机偏好，因为已经没有偏好。
- JSON 中 `data.session.target` 保持不变。

---

## 8. 验收标准

### 8.1 JVM（`android/app/src/test`），每条都应能直接断言

1. 没有租约时，`acquire(owner)` 和 `acquireUi(owner, changing=*)` 抛 `noSession`，并且之后 `current` 仍为 null（`localStatus().leaseId == null`）。
2. 没有租约但存在在途资源（模拟 open 进行中，或 `closing` 未完成）时，抛 `busy`，不是 `noSession`。
3. `released` 主体的任何 acquire 抛 `sessionReleased`；`session.open` 之后可用；`session.takeover` 仍被拒绝。
4. 别的主体持有租约时，acquire 抛 `busy`（文案含持有者）；open 也抛 `busy`。
5. `session.open{display:"main"}` 之后 `current.target.kind == MAIN`；`{display:"virtual"}` 在 WorkDisplay.create 抛错（用假 WorkDisplay）时：结果为 FAILED，`current == null`，stream 已 close，管理资源已释放。
6. 已有 main 租约时执行 open virtual 且建屏失败：旧 main 租约已失效（`isCurrent(old) == false`），`current == null`。
7. 回复先于发布（让 budget 在 create 中途耗尽）：结果为 FAILED，不是 UNKNOWN；随后 `publish` 不会执行；`current == null`。
8. 发布先于回复：结果为 done，并带 `target.kind == virtual`（验证 `remember` 已在同步块内）。
9. `args` 缺少 `display` 或值非法：返回 `arguments`，不改动租约。
10. idle 到期后 `current == null`，并调用了 `clearLeaseUi`（stream.close 被调用）。
11. `ui.read`：租约结束后得到 `noSession`；同主体重开后读旧 capture 得到 `captureUnavailable`。
12. `shell.start`、`file.begin` 在没有租约时得到 `noSession`；`job.get/cancel`、`file.get/cancel/chunk/finish` 在没有租约时仍可由原 owner 完成。
13. 后台 shell 在 GO 之后执行 `session.close`：close 成功，job 仍在运行，可以查询。
14. 任何请求 args 中出现 `expectedDisplayKind` 时，按现有未知参数规则拒绝（实施时确认现规则是“拒绝”还是“忽略”，两者择一并写测试）。
15. `WireContractTest`：`displayRouting` 已删除；`session.open` 样本两端一致。

### 8.2 Host（`androidctl/tests`）

1. `session open` 缺省发送 `{"idleSeconds":600,"display":"main"}`；`--display virtual` 发送 virtual；非法值在 CLI 层得到 exit 2，不联系 daemon。
2. `session takeover --display virtual --idle 5m` 发送对应 args。
3. 任何 UI、batch 请求的 args 中都不含 `expectedDisplayKind`；不读、不建 `display.json`（用不存在的 state 目录断言没有文件被创建）。
4. `display` 子命令不存在（`androidctl display` 退出码 2，并给出用法错误）。
5. 设备返回 `noSession` 时，文本形如 `snapshot: not sent; No open session; ...; code=noSession`，exit 2。
6. batch 首步 `noSession`：batch 为 not sent，`steps` 只含该步，不做额外 RPC（用 fake LocalCall 计数）；第 2 步 `noSession`：保留第 1 步结果，exit 非零。
7. 动作 done 后 `ui.read` 返回 `noSession`：结果仍为 done，exit 0（若有现行规则则以其为准），并带 `snapshotError`。
8. 观察的 `target.displayId == 12` 时屏幕行含 `display=virtual#12`；为 0 时不含。
9. `status`：有租约时 session 行含 `display=main` 或 `display=virtual#N`；无租约时含 `session=none`，`callerSession` 原样输出。
10. open 结果为主机 UNKNOWN（传输丢失）时 exit 3，文本提示 `run status`。

### 8.3 必须上设备的项

见文末“必须上设备验证的清单”。

---

## 9. 需要用户再定的问题（只列真实取舍）

**Q1. 纯 shell 和文件任务是否也必须先 `session open`？**
- (a) 是。字面执行“任何命令”，代码最少，保留单主人互斥。**推荐。**
- (c) 不是。设备空闲时允许免开会话执行只读查询（apps、通知、剪贴板读取），别人持有时仍 busy。约 +30 行，外加一张分类表。
- (b) 完全设备级：会削弱多 agent 互斥，不推荐。

影响：今天 `androidctl shell 'ls'` 一条命令就能跑；选 (a) 之后需要两条。

**Q2. `--display` 是否必填？**
- 缺省 main（推荐）：兼容现有写法；漏写 virtual 时，回执和屏幕行上能当场看出来。
- 必填：每次 open 都要有意识地选择，能多防一次漏写；代价是所有 open（包括纯 shell 任务）都多一个参数，而且基线写法全部失效。

**Q3. 是否禁止用 `default` 会话名开后台屏？**（`session open --display virtual -s default` 时，主机直接拒绝并提示 `-s NAME`）
- 这能挡住“多个未命名 agent 共享 default，一个 open 销毁另一个的后台屏”。主机侧约 5 行，没有设备状态。
- 不加则完全依赖指引里的命名约定。
- 我倾向**不加**，靠 §7.2 的指引。这是窄守卫，而同名冲突对 main 同样存在。如果用户更在意后台屏误伤真人主屏，可以加上。

（`session open` 在已有同主体租约时是替换还是拒绝：我维持基线的“替换”语义，并在回执中给出 `previousSession`。不单列为问题。）

---

## 10. 必须上设备验证的清单

前提：每条都在至少一台 Android 14+ 真机（Pixel 类）和一台国产 ROM 上跑。下文 `A=androidctl --device D -s t`。观察主屏副作用用 `adb shell dumpsys activity activities | grep -E "displayId|topResumed"` 和 `adb shell dumpsys display | grep -A2 "AndroidCtl work display"`，分别在操作前后各取一次。

1. **open virtual 成功路径**
   - 操作：亮屏解锁，执行 `A session open --display virtual`，然后 `A status`。
   - 看什么：回执 done 且 `display=virtual#N`；`dumpsys display` 中有该屏，flags 含 TRUSTED、OWN_FOCUS、STEAL_TOP_FOCUS_DISABLED。
   - 结果含义：done，后台屏可用；`displayUnsupported` 且 `missingFlags` 非空，该 ROM 不支持，产品文案要据此写明适用面。
2. **建屏失败不留租约**
   - 操作：熄屏，或锁屏后执行 `A session open --display virtual`，再执行 `A snapshot`、`A shell 'echo x'`、`A wake`、`A clipboard get`。
   - 看什么：open 返回 `displayUnavailable`；后续四条都是 `noSession`；主屏仍然熄灭或锁定，前台 activity 不变。
   - 结果含义：任何一条进入执行（尤其 wake 点亮了屏幕），说明 `keepLease` 仍有残余路径，属于实现缺陷。
3. **open 中途失去回复**
   - 操作：open virtual 刚发出时移除 adb forward 或断网，恢复后执行 `A status`。
   - 看什么：`callerSession` 为 active+virtual 或 none 二者之一；不会出现 main。
   - 结果含义：出现 main 说明原子性被破坏；为 none 且 `ui>0` 时，等待归零后重开应成功。
4. **virtual idle 到期**
   - 操作：`A session open --display virtual --idle 5s`，`A open --component <pkg/act>`，等 10s，然后 `A snapshot`。
   - 看什么：`dumpsys display` 中后台屏已消失；snapshot 返回 `noSession`；主屏前台 activity 不变；记录迁入后台屏的 task 去向（是否被销毁）。
   - 结果含义：task 在主屏出现，说明销屏迁移策略与 `DESTROY_CONTENT_ON_REMOVAL` 的预期不符，文档和 close 文案要改。
5. **virtual → main 替换**
   - 操作：在第 4 条的前半段之后执行 `A session open`。
   - 看什么：后台屏消失；回执带 `previousSession`；`A snapshot` 落在主屏；之前的后台 task 去向同第 4 条。
6. **worker 重建**
   - 操作：virtual 会话中，在 App 管理页 Stop（或 Force）再 Refresh，然后 `A snapshot`。
   - 看什么：返回 `noSession`；后台屏随进程消失；主屏不变。
7. **设备侧释放**
   - 操作：virtual 会话中，在管理页点“释放当前会话”，然后 `A snapshot`，再 `A status`。
   - 看什么：返回 `sessionReleased`；`callerSession=released`；后台屏已销毁。
8. **后台 job 跨会话**
   - 操作：`A shell --background 'sleep 30; echo ok'`，`A session close`，`A job get <id> --wait 40s`。
   - 看什么：close 返回 done；job 最终 done，输出 `ok`；close 之后 `A shell 'echo y'` 返回 `noSession`。
9. **传输跨释放**
   - 操作：`A push <100MB> /sdcard/x`，传输期间在管理页释放会话。
   - 看什么：push 最终 done 或按原规则结束；`A session open` 在传输结束前返回 busy，结束后成功。
10. **真人同时在用主屏**
    - 操作：virtual 会话中，真人在主屏连续打字；agent 在后台屏执行 `fill` 和 `tap`。
    - 看什么：主屏输入法和焦点不被抢走；后台屏动作 done。
    - 结果含义：一旦串扰，后台屏不能转为正式能力。这一条与探针评审员的清单合并执行。
11. **wake 在 virtual 下**
    - 操作：virtual 会话中让主屏自动熄灭，然后 `A tap ...`，再 `A wake`、`A tap ...`。
    - 看什么：第一次 tap 返回 `inputUnavailable`；wake 点亮主屏（不解锁）。安全锁下第二次 tap 仍被拒，这符合预期。
12. **同名冲突演示**（用于确认 §9 Q3）
    - 操作：终端 1 执行 `-s default session open --display virtual` 加 `open`；终端 2 执行 `-s default session open`；终端 1 再执行 `snapshot`。
    - 看什么：终端 1 的 snapshot 落在主屏，屏幕行没有 `display=virtual`。用这个结果决定 Q3 是否要加主机守卫。
