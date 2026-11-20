# 专题 35：会话、目标屏、清理、唤醒、启动与 worker 生命周期（`4235af9`）

评审对象是产品仓库 `4235af9` 中这一块的最终形态，对照旧候选 `3f45af6`（实施方上一批 40 个提交的终点）和主分支基线 `7d5be70`。没有设备；设备结论全部来自实施方交回的原件，镜像为 Android Studio 的 API 34 模拟器（下称 API34）和 MuMu（API 35，不能建后台屏）。

文中"后台屏"指 `session open --display virtual` 创建的、由 worker 拥有的虚拟显示屏；"lease"指 worker 内存里的当前会话对象；"清理未确认"指 UiAutomation 注销、回调线程停止或后台屏释放中有一步没有被确认完成。

---

## 1. 结论摘要

**整体状态：** 会话状态机本身是正确的、锁的使用一致，`Sessions` 的 JVM 测试真实地验证了它的行为；但这一块的复杂度主要来自后台屏和"清理逐阶段确认"两套机制，而它们在真实任务证据里几乎没有被用到。另外，启动确认的规则会把冷启动系统性地报成 UNKNOWN。

最重要的发现：

1. **`open` 的落点确认只取一次样本，冷启动会系统性地得到 UNKNOWN/退出码 3。** 取样时刻是动作后观察结束的时候（默认 1.5 秒，而且一看到变化就提前结束）。在我能扫描到的约 10 次 App 启动里有 2 次 UNKNOWN：VLC 首次冷启动（API34 后台屏）和 WPS 打开 PDF（MuMu 主屏）。两次都是带入口页的 App，事后都证实已经启动。基线和旧候选在主屏上会报 done；现在主屏也改成要求"目标包可见"，这一点是评审方前两轮推动的，**现在看是评审方的错误建议**。最小修法：在剩余预算内轮询任务列表；或者主屏按平台返回码报 done、落点只作事实附带。【源码＋原件 2 例】
2. **后台屏是这一块复杂度的主要来源，真实任务证据里没有一次因为它受益。** 它带来了 `--display`、`--replace`、种类不符的拒绝、建屏前的静态检查（包括 6 个隐藏接口的 Settings 读取助手）、启动前跨屏任务检查和 `--multiple-task`、跨屏落点、后台屏截屏（6 个隐藏接口）、通知 Activity 的跨屏门（也就是待决题 1），触及约 30 个 Kotlin 文件。四臂效率任务全部在主屏完成。后台屏的实测反而记录了害处：主屏上的 Chrome 任务被迁入后台屏，关屏时随之被销毁；主屏已有任务的 App 默认被拒；App 状态在两块屏之间共享。【源码＋原件】
3. **"清理逐阶段确认"机制做工精细，但防的是一个现在几乎不会自然发生的故障。** 它的设计动机是系统进程重启后注销失败（上一轮评审的 N1/N2）。worker 现在会在系统服务死亡时直接退出（`e6bab80`），这条主要的自然触发路径已经不存在。所有非注入的运行里没有出现过一次 `cleanupError`。最复杂的编排（`UiBackend` 里的 pending 清单、"用一次新注册证明旧注册已释放"、close 时重试）没有任何 JVM 测试，只靠 4 个故障注入案例各跑一次。【源码＋原件】
4. **显式 `session open` 模型本身是可以接受的。** 它每个任务多 1 次调用，idle 到期后多 1–2 次调用（重开会话，再重新 snapshot 拿 REF）。它的安全收益几乎全部来自后台屏："后台屏建不成后命令悄悄落到主屏"。如果去掉后台屏，应回到基线做法：第一条命令隐式建立主屏 lease，设备端释放后仍要求显式重开。【源码＋推断】
5. 小缺陷：idle 到期与 `open --display virtual`/`wake` 恰好撞在同一秒内时，会报一次虚假的 `uiCleanup`；`wake` 在 `interactive=true` 时就返回，主屏可能还是 OFF（原件 078），紧接着的后台屏输入会被拒；文档对清理失败的出路写的是较重的 "Force executor exit"，而普通的 "Stop idle executor" 已经够用（U− 原件）；系统进程死亡后的恢复实际需要重启 Shizuku，公开指引没有写。

**对待决题 1（后台屏通知打开）的看法：** 四个选项都假定"后台屏上照常派发 Activity 类通知"，漏掉了最简单的一项 E："后台屏 lease 上不支持 Activity 类通知，零派发拒绝"。如果用户决定冻结或去掉后台屏，这道题整体消失。若必须在 A–D 中选，D 的原则（派发和观察分开报）是对的，而且应该同样用在 `open` 上（见第 1 条）。

---

## 2. 确定缺陷

### 2.1 【中高，可用性】启动落点只取一次样本，冷启动系统性 UNKNOWN/3

- **触发条件：** `open`（包名、URL 或 typed intent）启动的 App 在动作后观察结束时还没有可见任务。这包括冷启动、带入口页（trampoline）的 App、模拟器上的慢启动。主屏和后台屏都会触发。
- **源码：**
  - `ActivityLaunch.kt:199-201`：先执行 `observe(result)`，然后只调用一次 `confirmLanding`。
  - `ActivityLaunch.kt:298-308`：读一次 `liveTasks()`，没有命中就是 UNKNOWN。
  - `UiPostAction.kt:39-40,63-67,154`：观察窗默认 1500 毫秒，`stableWithoutBaseline()` 或 `CHANGED` 时提前结束。
  - `ActivityTasks.kt:101-117`：只认 `isVisible` 而且顶层是目标包的任务。
  - 按 AOSP 14 的 `ActivityRecord.setVisibility`（shell transition 下"Defer committing visibility until transition starts"）和 `Task.fillTaskInfo`（`isVisible = hasVisibleChildren()`），任务要等转场开始后才算可见。我下载的副本在 `agents7/35/aosp/a14-ActivityRecord.java:5423-5444`、`a14-Task.java:3445`。
- **原件：**
  - `ST/12-sessions/03-authority/preflight-fix/runtime/api34/commands/093-vlc-virtual-real-view-once/stdout.bin`：START_SUCCESS，settleMs=1502，samples=6，`landing.state=unknown`，matchingTaskCount=0，结果 UNKNOWN。094–096 随后证实 VideoPlayerActivity 已经在该后台屏上。
  - `ST/16-delivery/runtime/mumu-view/commands/reading/004-open-new-own-PDF-current-WPS-component/stdout.bin`：MuMu 主屏，handler 是 `PreStartActivity2`，START_SUCCESS，settleMs=2185，samples=1，可见任务仍是 Chrome，结果 UNKNOWN。005 的迟后截图里已经是 WPS PDF。
  - 【重算】脚本 `agents7/35/scripts/scan_landing.py` 扫描 `ST` 下含 `"command":"open"` 的原件：landing 为 observed 的约 13 个文件、unknown 的 4 个文件（对应 2 次命令）。
  - 交付稿还记录了 Feeder 通知打开的一次 UNKNOWN/3，那是通知路径，规则同源。
- **实际影响：** 指引要求 UNKNOWN 之后"先确认未发生且操作已结束，再动作；再遇 UNKNOWN 就停"（`SKILL.md:217-223`）；batch 遇到 UNKNOWN 会停（`displays.md:87`）。冷启动本来是最常见的启动形态，现在每次都要多一次核对读取，batch 也会被打断。这样还削弱了 UNKNOWN 作为"真的不确定"的信号意义。基线（`7d5be70` 的 `SystemActions.kt:190-195`）和旧候选（`3f45af6` 的 `ActivityLaunch.kt:247-253`）在主屏上都按平台返回码报 done。
- **最小修正方向（二选一）：**
  1. 观察结束后，在剩余请求预算内每 100–200 毫秒轮询一次 `getTasks`，直到 observed 或预算耗尽。open 默认预算 5 秒，观察窗 1.5 秒，余量足够；`getTasks` 是一次廉价的 Binder 调用。
  2. 更简单：平台返回成功码就报 done，`landing` 只作附带事实；只有 `elsewhere`（目标包出现在别的屏上）才报 UNKNOWN。
  
  我更倾向第 2 种：调用者下一步本来就要 snapshot。
- **确定程度：** 高。有源码链和 2 份原件；样本是 1 个 API34 镜像加 1 个 MuMu，两个不同的 App。

### 2.2 【低】idle 到期在同一请求内被检出时，虚假报告 `uiCleanup`

- **触发条件：** 会话恰好在两次定时检查（每 1 秒一次，`WorkerService.kt:40,49-56`）之间到期，而这时到达的请求是 `session open --display virtual` 或 `wake`。
- **源码：**
  - `Sessions.kt:138`：`checkChange` 先调用 `expire()`。
  - `Sessions.kt:329-332`：`invalidateLease` 调用 `clearLeaseUi`。
  - `UiBackend.kt:643-654`：`clear` 立即把一个 `detachActive=true` 的待清理项放进清单，并把真正的清理排到 UI 执行线程上。
  - 同一调用里，`Sessions.kt:155`（以及 wake 用的 `Sessions.kt:214`）随即调用 `requireCleanUi`；这时 `cleanupFailure` 返回 "Target UI resources await cleanup"（`UiBackend.kt:733`），请求被拒为 `uiCleanup`。
- **影响：** 窗口在 1 秒左右，概率低。但报错文案是"清理未确认，重试 close 或到设备上强制结束"，会把调用者引到不必要的恢复流程上。
- **修正：** `requireCleanUi` 忽略仍处于 `detachActive` 或 `localPending` 状态、还没执行过的项；或者在本次 expire 刚排入清理时，先等 UI 执行线程跑完这一项再判断。
- **确定程度：** 中。只是源码推导，没有原件；JVM 测试 `SessionsTest.kt:147-158` 用一个永远 pending 的假清理复现了同一判定。

### 2.3 【低】`wake` 在主屏仍为 OFF 时就报 done

- **源码：** `DeviceWake.kt:133` 只要 `interactive` 为真就返回。而后台屏的输入门要求主屏 `displayState == ON`（`DeviceFacts.kt:76-81`，`UiBackend.kt:83-86`）。
- **原件：** `formal-result.md` 记录 078 的即时结果是 `interactive=true`、`displayState=1`（OFF），080 稍后才是 ON。
- **影响：** `wake` 之后紧接着在后台屏上输入，可能得到 `inputUnavailable`。主屏输入只检查 `interactive`，不受影响。
- **修正：** 等待条件加上"主屏 `displayState` 为 ON"，仍在原预算内完成。
- **确定程度：** 中高（源码加 1 份原件）。

### 2.4 【低，文档】清理失败与系统进程死亡后的出路写得不准

- 指引和报错都把清理失败的出路写成 "Force executor exit"（`Sessions.kt:355-356`，`displays.md:51`，`connection.md:150-151`）。但待清理项不算资源，不会阻止正常停止（`Sessions.kt:280-283` 只检查资源），所以设备上的普通 "Stop idle executor" 就够用。U− 原件就是用正常 Stop 释放了 worker 和注册位（`cleanup-faults-f7/runtime/result.md` 表格的 U− 一行）。指引应先写较轻的出路，并明确这是要请用户在设备上做的事。
- 系统进程死亡后，实测恢复要先通过原生 starter 重启 Shizuku，再开 Gateway，再 Refresh；第一次 Refresh 还因 Shizuku 未就绪而失败（`ST/runtime/api34/result.md:61`，命令 111–121）。公开指引（`connection.md`）完全没有提到这种情形，只写了"worker 不可用时用 Refresh"。
- **确定程度：** 高（源码加原件）。

### 2.5 【低】`session close` 预算耗尽时，把超时说成清理未确认

- **源码：** `TargetManagement.kt:138,142-143`。如果 `budget.check()` 抛出超时，回执用 `ui.cleanupFailure ?: message(error)` 作为清理详情，于是得到退出码 3 和"UI cleanup unconfirmed…: Request budget exhausted"，而清理本身可能已经完成（lease 早在 `prepareClose` 中失效，`clear` 也已先行执行）。
- **影响：** 只是误导，再执行一次 close 就会收敛。
- **确定程度：** 中（源码推导）。

### 2.6 【外观】`session open` 缺少 `--display` 时，报错里出现字面 `\n\t`

- 【实跑】`venv4/bin/androidctl session open` 输出 `Missing option '--display'. Choose from:\n\tmain,\n\tvirtual; code=arguments`，退出码 2。原因是控制字符被转义显示。

### 2.7 测试缺口（不是功能缺陷）

`UiBackend` 的整套清理编排没有任何 JVM 测试，包括 `clear`、`clean`、`cleanupRetry`、`discardConnection`、`discardTarget`、`registrationConfirmationReady`、`connect(confirmingRelease)`，以及 `TargetManagement.close`。依据是在测试目录里 grep `cleanupRetry|discardTarget|registrationConfirmation` 只命中 Sessions 层的 lambda。举例来说，删掉 `UiBackend.kt:716` 的 `removeAll { it.complete }`，清理就会永远 pending，但没有任何 JVM 测试会变红。这部分只由 4 个故障注入案例各跑一次覆盖。【源码】

---

## 3. 设计与简化

### 3.1 会话状态机（现状）

**状态对象（全部在 worker 内存，worker 死亡即丢失）：**

| 对象 | 位置 | 写入方 | 锁 |
| --- | --- | --- | --- |
| `current: Lease?`，记录 id、owner（pairing＋name）、touched、idleMs、target(kind, displayId, stream) | `Sessions.kt:66` | `publish`、`invalidateLease` | `Sessions` 对象锁 |
| `released`：被设备端"Release current session"释放过的 (pairing, name) | `:67` | `releaseLocal` 加入；`publish` 时移除 | 同上 |
| `revoked`：已撤销的 pairing | `:68` | `revoke`、`reconcile` | 同上 |
| `resources`：进行中的 UI、Shell、File 资源，以及会话变更、关闭、唤醒的占位 | `:69` | `acquireUi`、`reserve`、`prepareChange`、`prepareClose`、`acquireDeviceUi` 加入；`finished` 移除 | 同上 |
| `stopped` | `:71` | `stop` | 同上 |
| `pendingCleanup` 清单，每项是 `PendingUiCleanup{detachActive, stream, connection, localPending, error}` | `UiBackend.kt:47,719-738` | `clear`、`discardConnection`、`discardTarget` 加入；`clean`、`registerAutomation` 移除已完成项 | 清单自身的锁；各字段只在 UI 单线程执行器上改 |
| `AutomationCleanup{disconnectAttempted, registrationPending, callbacksPending, disconnectError, callbackError}` | `AutomationCleanup.kt:9-21` | UI 执行器 | volatile |
| `automation`、`platform`、`connectionCleanup`、`generation` | `UiBackend.kt:25-28` | UI 执行器 | 只在单线程上改 |

锁顺序一致：总是先取 `Sessions` 锁，再取 `pendingCleanup` 或 `UiDelivery` 的锁。我没有找到反向的取锁路径。【源码】

**各转换路径：**

- **open main 或 virtual：**
  - `prepareChange` 分两段（`Sessions.kt:98-129`）：第一段在锁内检查 owner、资源、种类，同种就直接复用；然后在锁外跑静态检查；第二段重新检查，要求看到的还是同一个 `held`，否则报 `sessionChanged`。之后建立新的 Lease 和占位资源，并让旧 lease 失效。
  - 执行器上跑 `changeTarget`（`TargetManagement.kt:54-83`）：只要有旧 lease 或目标是 virtual，就先检查清理；然后建屏，在 `synchronized(run)` 里 `publish`；失败就丢弃新屏。
- **同种复用：** 返回原 lease，只更新 touched，**不改 idle**（`Sessions.kt:108-113`，`TargetManagement.kt:123`）。
- **异种 `--replace`：** 旧 lease 在执行器建屏之前就已失效（`Sessions.kt:126`），所以**不是原子的**：新屏建不成时，没有任何会话，旧后台屏的内容已经销毁（`DESTROY_CONTENT_ON_REMOVAL`，`WorkDisplay.kt:72`）。`displays.md:35-37` 如实写了这一点。主屏换成后台屏失败时，只损失主屏的 REF，代价很小。
- **takeover：** 同一 pairing 内即可接管，永不复用；`released` 状态的 owner 被拒（`Sessions.kt:139-145`）。
- **idle 到期：** 定时器每秒一次，加上每次请求入口都会检查；条件是资源为空且超时（`Sessions.kt:289-292`）。
- **显式 close：** `prepareClose` 立即让 lease 失效，再在执行器上重试旧的待清理项。没有 lease 时也可以执行（不检查 owner），所以可以反复重试；没有待清理项时报 "No current session; UI cleanup confirmed"，因此**会收敛**。
- **设备端释放：** 把 owner 加入 `released` 并让 lease 失效；之后普通命令报 `sessionReleased`，显式 open 可以恢复，takeover 被拒，wake 照常可用。与 `connection.md:152-155` 一致。
- **清理未确认后：**
  - 被挡：UI 命令（只有"注册待确认"这一种情况例外，可以经正常注册确认后放行）、建后台屏、异种替换、takeover、wake。
  - 放行：没有 lease 时显式开 main、在已有 lease 下跑 shell 和文件、idle 到期、正常 Stop。
  - 以上与 `connection.md:146-151`、`displays.md:43-56` 一致。【源码】
- **worker 死亡：** 内存状态全部丢失，新 worker 没有 lease，`callerState=none`；后台屏随进程释放（DisplayManager 跟随 Binder 死亡）。
- **系统服务死亡：** `WorkerService.kt:20-26` 用 `linkToDeath` 收到通知后执行 `exitProcess(0)`。主机看到的是在途请求 UNKNOWN，随后连接不可用；恢复要人在设备上操作（见 2.4）。
- **主机 daemon 重启：** lease 在设备上，用 pairing 加会话名作键，不受影响；这是设备端 lease 的一个真实优点。

**没有找到的问题：** 遗漏释放、错误放行、跨会话越权。wake 的准入（`Sessions.kt:207-216`）要求没有别的 owner、没有任何活动资源；它注入唤醒键走的是 InputManager（`AppInput.kt:75-81`），不连接 UiAutomation，与他人会话不冲突，也没有越权：配对本身就是授权边界。【源码】

### 3.2 后台屏整套能力：值得吗

**它在证据里帮到了什么：** 我没有找到一例"因为有后台屏才完成"的真实任务。

- 最终四臂效率任务全部在主屏，提示里写明"会话已打开为 main"（`ST/16-delivery/efficiency/github-new/prompt.md`）。
- 后台屏上的运行都是实施方专门设计的能力验证：Contacts 主屏一份草稿、后台屏一份草稿；VLC；regex101；通知打开。

**它带来的真实问题（原件）：**

1. 主屏已有任务的 App 默认被拒（064：Contacts 的 `taskOnOtherDisplay`）。要绕过只能用 `--multiple-task`，而且只对部分入口有效（`ActivityLaunch.kt:316-334`）。用户常用的 App 恰恰最可能已经在主屏上有任务，**包括不可见的后台任务**（`ActivityTasks.kt:84`）。
2. 跨 App 通知会把主屏上已有的任务迁入后台屏，关屏时随之销毁（`ST/15-notifications/runtime/api34-notifications/review.md:45,49`，Chrome 的 task 266，1 例）。这直接违背了"不打扰用户主屏"的初衷。
3. App 的数据、进程和播放服务在两块屏之间共享（`displays.md:105-107`）。
4. MuMu 不能建后台屏。

**它的代价（源码）：**

- 专属代码约 450–550 行：`WorkDisplay.kt` 212 行、`WorkDisplaySettings.kt` 59 行、`DisplayTarget.kt` 31 行，再加上 `ActivityLaunch` 和 `ActivityTasks` 的跨屏检查、`TargetManagement` 的建屏和丢弃、`Sessions` 的 REPLACE 和 displayMismatch、`UiBackend` 的目标就绪判断、通知 Activity 的跨屏确认。
- 13 个专属隐藏接口：截屏 6 个，多窗口和 Settings 读取 6 个，`supportsMultiWindow` 1 个。
- 显示编号贯穿约 30 个 Kotlin 文件；`displays.md` 107 行，其中大半在讲后台屏。
- 待决题 1 整题只因为后台屏而存在。
- `--display`、`--replace`、`takeover --display`、`displayMismatch`、`taskOnOtherDisplay`、`multipleTaskUnsupported`、`displayUnsupported`，都是调用者必须学的概念。

**判断：** 它是这一块乃至整个产品复杂度的主要来源之一，而在当前证据里收益为零，还有已经实测到的副作用。对于一个人维护、面向 LLM 的 alpha 产品，这笔账不划算。是否删除属于用户已确认过的"能力全部保留"范围，所以放到第 6 节由用户裁决。

### 3.3 显式 open 模型：变简单了还是变麻烦了

**对 LLM 调用者的影响：**

- 规则是一条："任务开始先 `session open --display KIND`"，好学。
- 成本：每个任务多 1 次 open、1 次 close。idle 默认 10 分钟（`Sessions.kt:349`），到期后命令报 `sessionRequired`、退出码 2；要重开，旧 REF 全部失效，还得重新 snapshot。实施方真实任务里撞到过一次：`ST/16-delivery/runtime/mumu-view/view-result.md:18` 记录操作者离线整理时 10 分钟到期。
- `sessionRequired` 的报错不区分"从未开过"和"已到期"（`callerState` 都是 none，`Sessions.kt:322-327`），调用者得自己推断。

**安全收益：** 早先定稿 §2.1 列出的理由，核心是"后台屏建不成后，后续命令悄悄落到真人主屏"（F17 指上一轮评审编号的缺陷"无会话时隐式取得主屏会话"）。这条理由**只在有后台屏时成立**。另一条"同名调用者为跑 shell 开主屏，把原调用者的后台屏销毁"，同样是后台屏特有的。

**更简单而同样安全的做法：**

- 如果去掉后台屏：回到基线 `7d5be70` 的 `Sessions.acquire`（`Sessions.kt:48-59`），首条命令隐式建立主屏 lease，保留"设备端释放后必须显式重开"。`--display`、`--replace`、`takeover --display` 都可以删除。
- 如果保留后台屏：显式 open 可以保留，代价不大。建议两点：一是 `sessionRequired` 在 owner 刚到期时写明 "expired"，只需在到期时记下 (pairing, name)，与 `released` 同构；二是把默认 idle 调到 30 分钟。另一条可行的折中是"只有后台屏需要显式 open，主屏隐式"，但需要记住"上次是 virtual"，第三轮评审 §7 已经删掉过这类记录，反复折腾不值得。

**结论：** 显式 open 本身不是问题，问题在于它为后台屏而存在。

### 3.4 静态建屏检查中的 Settings 助手：应删

- 用途：`WorkDisplaySettings.kt`（59 行，6 个隐藏接口，经 external provider 读 `force_resizable_activities`）只在 `supportsMultiWindow` 为 false 时执行（`WorkDisplay.kt:50`）。
- 实测：API34 模拟器上 `supportsMultiWindow` 实际为 true（`preflight-fix/runtime/api34/diagnostic-result.md`："平台 multi-window 实际 true，仅诊断操作数 base=false"）。所以生产路径上这段代码从没被执行；唯一一次执行，是用专门改过的诊断 APK 强制走进去的。
- 由来：它源自本轮自己引入的一个缺陷，即用错误的 context 和错误的键名读 Settings（`virtual-preflight-investigation.md`）。修复之后又加了一套机制。
- 建议：删掉这段读取，`supportsMultiWindow=false` 直接报 `displayUnsupported`。代价是少数低内存设备、手表、电视上开了开发者强制选项的情形被误拒。【源码＋原件】

### 3.5 清理逐阶段确认：保留拆分，砍掉"用新注册证明旧注册已释放"

- **该保留的：** 第三轮评审 N2 的核心修正是对的，即"待清理不算资源，不挡 shell、文件、到期和正常 Stop"。对应 `Sessions.kt` 里的 `requireCleanUi` 和 `requireIdle` 分开。
- **可删的：**
  - 用一次新的 UiAutomation 注册来证明旧注册已释放，包括 close 时的重注册和 UI 准入时的确认：`UiBackend.kt:53-57,120-123,548-550,607-610,667-679`，`AutomationCleanup.registrationOnly/confirmReleased`，`Sessions.acquireUi` 的 `allowRegistrationConfirmation` 参数。
  - 理由：自然触发源（系统进程重启）已由 `e6bab80` 消除；所有非注入运行里没有出现过 `cleanupError`（在 `ST` 中 grep，排除 `cleanup-faults` 后为 0）；这段编排没有 JVM 测试；出问题时设备上的普通 Stop 已经可以收敛（U− 原件）。
  - 删后：注销没有确认时保持封锁，出路写"请用户在设备上 Stop idle executor"。后台屏释放、回调线程停止这类可以幂等重试的步骤，仍可以用 close 重试。
- **代价：** 失去对一种未观察到的故障的远程恢复能力。【源码＋原件】

### 3.6 其它小项

- `wake` 被待清理挡住（`Sessions.kt:214`）没有机制上的必要，因为它不碰 UiAutomation，可以去掉这一检查。
- 同种重开不更新 idle（`TargetManagement.kt:123`），导致想延长 idle 只能先关再开，旧 REF 随之失效。可以改成"同种重开时采用新的 idle"。

### 3.7 评审方前几轮现在看错了或代价过高的建议

1. **早先定稿 §2.2 和第三轮 §2.2 第 9 条：** "等动作后观察结束取一次样，主屏和后台屏同一规则，未确认就 UNKNOWN/3"。它假定观察窗足够长，实际上观察会提前结束，冷启动也超过 1.5 秒。它把主屏启动从 done 改成了 UNKNOWN。早先定稿的验收标准"VLC 在后台屏启动，落点确认，退出码 0"只在第二次热启动时达成，首次冷启动没有达成（原件 093 和 106）。**这条建议应撤回**，改成 2.1 的做法。
2. **第三轮 §3.2 第 11 题：** "重试时按可观察事实逐阶段确认（UiAutomation 用下一次注册成功作证）"。诊断是对的，方案过度：它是为 N1 那种故障设计的，而 N1 本身已经修掉。
3. **早先定稿 §2.1 第 2 条和第 3 条：** "所有命令都需要显式会话，主屏也一样"。它的安全论据只对后台屏成立。保留后台屏时可以接受；如果去掉后台屏，应撤回。
4. **早先定稿 §2.1 第 9 条：** "建屏前提先检查，通过后再让旧会话失效"。这一条有一定价值，能保住主屏的 REF，但它直接引出了多窗口和 Settings 这一层检查，以及本轮那个 Settings 读取缺陷。前提检查只保留便宜的权限和特性检查即可。

---

## 4. 证据审计

| 实施方声称 | 核对 | 结论 |
| --- | --- | --- |
| 清理故障注入 D、C、U+、U− 证明了边界。这四个名字分别指：后台屏首次释放前失败、后台屏图像回调线程不停、UiAutomation 断开后仍报注册未释放、系统注册位被占 | `cleanup-faults-f7/runtime/result.md`、`source-binding.json` 表明故障 APK 由 `f7a214d` 加注入补丁构建。我核对过，`f7a214d..HEAD` 期间 `Sessions`、`TargetManagement`、`AutomationCleanup`、`WorkDisplay` 没有改动；`UiBackend` 的清理部分只在 `clear` 的 finally 里多了一行 `notifications.endLease()`。版本基本匹配。每案只跑 1 次。U− 里"旧 owner 就是原注册"是推断，原件自己也这么写 | **部分成立**：证明的是人为注入下 4 种路径按设计工作，且可收敛（U− 靠正常 Stop）。它不能说明自然故障的发生率；这段编排也没有 JVM 回归保护 |
| API34 上正式的 lease、launch、wake 组合成立 | `formal-result.md` 用的 APK 是 `1ef02e62…`，对应 `f7a214d`；之后 `ActivityTasks` 只是重构（`activityLanding` 参数拆开，语义相同，我看过 diff）。覆盖了 Contacts 两次启动 observed、默认冲突拒绝、`--multiple-task`、同种复用（REF 和图仍可用）、后台屏内 wake、VLC 主屏和后台屏各一次 observed，以及 VLC 首次 UNKNOWN（被保留，没有被后继成功覆盖）。无 lease 的主屏 wake 沿用的是更早版本的证据，原件自己说明了 | **部分成立**：组合可以跑通，样本是 1 个镜像、约 5 次启动。但"启动确认"这一项，首次冷启动就没过，属于可用性失败，交付稿只当作"如实保留的事实"，没有当作设计问题 |
| trueLocked 分支未实跑 | `formal-result.md`："没有真正 locked/PIN 前提，trueLocked 分支未验" | **成立**（如实） |
| preflight 修正属于缺陷修复 | 041 原件：`Given calling package com.rainng.androidctl does not match caller's uid 2000`。修正前的提交 `2a78f7e` 是本轮自己的中间提交，最终历史里修正已经并入 `83c3a6a` | **成立，但需要说清：** 修的是本轮自己引入的缺陷；修复引入了 6 个隐藏接口，生产路径在测过的设备上从不执行（3.4） |
| 系统服务死亡链（N1，即"系统进程重启后 worker 握着失效连接"的缺陷）已在 API34 证明 | `ST/runtime/api34/result.md:53-61`：`kill -9` system_server 一次，约 14 秒后旧 worker 的 /proc 不存在，Shizuku 仍活；恢复需要重启 Shizuku、开 Gateway、Refresh 两次。第一次在前提上就失败（adbd 以 root 重启，Shizuku 和 worker 同时消失），原件保留了 | **部分成立：** 旧 worker 消失与 `linkToDeath` 退出相符，但原件也承认没有取得进程退出码，不能直接证明是这个回调导致的。恢复路径比公开文档写的重（2.4）。样本 1 次 |
| 显式 lease 后，六类命令无会话时一律 `sessionRequired` | `12-sessions/runtime/mumu/result.md`：025–030 共 6 条命令均为 not sent、退出码 2（MuMu，版本 `2a78f7e`）。源码 `Sessions.kt:74-79,195-200` 一致 | **成立** |
| MuMu 静态拒绝保住原主屏会话 | 同一文件中，039 报 `displayUnsupported`（ADD_TRUSTED_DISPLAY），040 显示 lease 仍在，041/042 原 REF 和原图仍可用 | **成立**（MuMu，1 次） |
| 效率四臂反映了新模型的调用成本 | 四臂提示写明"会话已打开为 main，无需重新 open；收尾由 root 关闭" | **不能用来评价会话模型**：open、close、到期的成本都在实验之外 |

失败和 UNKNOWN 的留痕情况：VLC 093、WPS 004、Feeder 074、N1 第一次前提失败、MuMu 管理 dump 的退出码 139，都在原件和交付稿里如实保留，没有被后继成功覆盖。这一点值得肯定。

**测试抽查（是在验证行为，还是在复述实现）：**

1. `SessionsTest`（`SessionsTest.kt:32-183`）用真实的 `Sessions`、假时钟和假清理回调。删掉同种复用、`displayMismatch`、第二段的 `checked !== held`、`requireCleanUi`、`released` 中的任何一个判断，都会有测试变红。**这是行为测试。**
2. `DisplayTargetTest:78`：真实的 `WorkDisplay.preflight` 加一个拒绝权限的 ContextWrapper，验证旧 lease 保留、执行器没有排任务。**这是行为测试**，但只覆盖主屏和权限拒绝这一支。
3. `UiCleanupTest`：验证 `AutomationCleanup` 的标志变化和 `confirmWorkDisplayRemoval` 纯函数。粒度正确，但**不覆盖** `UiBackend` 的编排（见 2.7）。
4. `ActivityLaunchTest:155-245`：假的平台对象喂任务列表，验证分类规则。分类是对的，但测试把"观察后只读一次"当作规格固定下来了（`liveReads` 断言），所以发现不了 2.1。
5. `DeviceWakeTest`：假的设备事实和输入，验证 down/up、轮询和不碰 lease。是行为测试，但没有覆盖 2.3（只看 interactive）。
6. 主机端 `test_display.py`：路由测试，验证 CLI 参数被正确转成 RPC，不读旧的 display.json。主机端很薄，这样的覆盖相称。【实跑】`pytest tests/test_display.py`：21 passed。
7. `WorkerService` 的系统服务死亡回调：没有测试（实施方明说了），只有 1 次设备运行。

---

## 5. 对相关待决题的评价

**待决题 1（后台屏 Activity 通知）：**

- 四个选项是真实的候选，但都在"后台屏上派发 Activity 通知"这个框架内取舍。
- **漏掉的选项 E：** 在后台屏 lease 上，Activity 类通知一律零派发拒绝，提示改用主屏会话。这不需要新机制，不会迁移任务，也不会误拒；代价是后台屏会话里不能点开通知。考虑到 Chrome 迁移后被销毁这一原件，E 比 C 安全，比 A 和 D 简单。
- 推荐 D 的依据（派发与观察分开）本身成立。同一原则应该一并用在 `open` 上（2.1）。如果用户决定冻结或删除后台屏，这道题消失。

待决题 2–4 不属于本专题。

---

## 6. 需要用户裁决的新取舍

### 6.1 后台屏的去留（推荐 B，其次 C）

| 候选 | 得 | 失 |
| --- | --- | --- |
| A 维持现状 | 能力完整 | 维护本节列出的全部面，以及待决题 1；文档负担最重 |
| B 冻结为实验能力：公开指引不再介绍，代码保留但不再扩展；同时按 3.4 删掉 Settings 助手，按选项 E 拒绝通知 Activity | 立刻减轻调用者的学习面和维护面；保留以后的可能 | 代码面还在；需要处理"实验能力"的标注 |
| C 删除：去掉后台屏、`--display`、`--replace`，回到隐式主屏 lease | 删掉约 500 行和 13 个隐藏接口，待决题 1 消失，会话模型回到基线那样简单 | 失去"在不打扰主屏的屏上工作"的可能性。现有证据里它没兑现过，而且有迁移和销毁用户任务的副作用 |

剩余未知：是否有真实用户或场景（例如批量后台抓取）确实依赖后台屏。证据里没有。

### 6.2 启动确认的语义（推荐 B）

| 候选 | 说明 |
| --- | --- |
| A 维持现状 | 冷启动大约 1/5 是 UNKNOWN/3（2 个镜像、约 10 次） |
| B 主屏按平台返回码报 done，落点作事实附带；后台屏只在 `elsewhere` 时报 UNKNOWN | 最简单；调用者下一步本来就要读屏 |
| C 保留"可见才算完成"，改为预算内轮询 | 语义最严；多一个轮询循环 |

B 和 C 都优于 A。

### 6.3 会话模型（取决于 6.1）

- 若 6.1 选 C：恢复隐式主屏 lease。
- 若选 A 或 B：保留显式 open，默认 idle 改为 30 分钟，到期给出明确提示（3.3）。

实施细节，实施者可以自行决定：2.2–2.6 的修正、3.5 的删减范围、3.6。

---

## 7. 没能核实的内容

- 我没有设备，所有判断都来自原件；没有在真机或厂商 ROM 上验证任何结论。
- 冷启动 UNKNOWN 的比例只来自 2 个镜像、约 10 次启动，而且是脚本按文件扫的（`scan_landing.py`）。batch 内或其它格式的 open 可能有遗漏。
- 我按 AOSP 14 源码推断"任务可见要等转场开始"，没有取得设备上的时序日志。
- 系统服务死亡后旧 worker 消失，是否确实由 `linkToDeath` 回调导致（而不是其它原因），原件不能区分。
- `PKG/source/androidctl/docs/` 中架构文档关于会话部分的逐句一致性，我只抽查了 `connection.md` 的对应段落（在 06-worker-lifetime 的 docs 差量中），没有逐句核对。
- 2.2 和 2.5 只是源码推导，没有原件。
- 隐藏接口计数（基线约 20 个、旧候选约 28 个、当前约 32 个，只计本专题相关的）是按 `HiddenApi.kt` 成员名人工归类得到的。行数统计（14 个相关文件：基线 1146、旧候选 2423、当前 2910）是 `git show | wc -l` 的结果。两者都只是量级参考。

**附：状态和机制量级对比**

| 类别 | `7d5be70` | `3f45af6` | `4235af9` |
| --- | --- | --- | --- |
| 14 个相关文件的行数 | 1146 | 2423 | 2910 |
| 本块隐藏接口（约） | 20 | 28 | 32 |
| 会话状态 | current、released、revoked、resources、stopped | 同上，Lease 加 target；多出 closing 占位 | 同上，加 SessionChange{resource, previous, reused} 和 OPEN/REPLACE/TAKEOVER 三种模式，preflight 分两段 |
| 清理状态 | 1 个字符串 `closeError` | `closeError` 加一个永久的 closing 资源 | pending 清单（每项 5 个字段）、AutomationCleanup（5 个字段）、`connectionCleanup`、注册确认路径 |
| 超时和常量 | idle 10 分钟、到期定时 1 秒 | 同左 | idle 10 分钟、上限 24 小时、清理 2 秒、到期定时 1 秒、落点依赖的观察窗 1.5 秒、任务事实 8 条或 8 KiB、后台屏图像 3 张 |

可删除项汇总：Settings 助手（3.4）；注册确认路径（3.5）；wake 的清理门（3.6）。如果用户选择删除后台屏，还可以删掉第 6.1 节 C 所列的全部内容。
