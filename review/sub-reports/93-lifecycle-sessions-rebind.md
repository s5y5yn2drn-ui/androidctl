# 专题核查：会话与显示选择、自动化登记与清理、常驻进程死亡与重绑、会话到期

核查对象：产品 `f401deb`（本专题的改动集中在 `1973106`），对照第六轮候选 `fb75afc`。只读；没有设备、没有构建，JVM 测试没有运行。主机端在 `S/r7/run` 上用 `S/r7/venv7` 做了本地复现。下文"常驻进程"指经 Shizuku 以 shell 身份运行、用 UiAutomation 读写界面的那个执行进程；"租约"指会话对显示目标的占用；"后台屏"指 AndroidCtl 自建的虚拟显示；"引用"指打印给调用者的节点编号（如 `e350999601`）。行号默认是 `f401deb` 的。

## 摘要

1. 这条链做对了，我没有找到需要挡合入的问题。重绑的"预期退出"标记按进程记（挂在每次附着的对象上），在发出销毁之前置位；最短存活 5 秒从附着成功那一刻算，用单调时钟。所有回调和绑定都排在主线程上；Shizuku 自身重启时与已有的自动绑定不会重复触发。旧引用不可能点到新页面：引用编号每个进程随机起步，是 9 位数（`UiRefs.kt:8`）。设备实测和源码对得上。
2. A2（别的自动化工具占着登记）修得正确。只拆反射那一层包装，超时与同步抛错就分清了；同步抛错不再留下"需要注销"的状态；"已登记"的识别与平台异常一致。
3. 会话到期和第 25 条的六条细则、第三次答复两条都已落实，也有设备实测（`016`、`024`、`027`、`030`、`032`、`033`）。到期由常驻进程里每秒一次的定时检查触发，任务表在销毁后台屏之前、在锁外查询，所以第一条报错通常就能带上完整的任务记录。`status` 确实不续期。
4. 默认主屏没有带来新的"悄悄操作用户手机"的风险。有到期记录的路径上，提示写在成功回执的首行，报错里也照旧要求显式写显示。记录丢失的几条路径，裁决里已经列为已知缺口。
5. 有一个首次使用缺陷，建议进下一轮：常驻进程附着的那一两秒里（首次启动，以及意外退出后自动重绑期间），命令拿到的是 `workerUnavailable` "Worker is unavailable; check the device App"，码表让调用者"请用户点刷新"，而实际上只要几秒后重试即可。真实准备阶段碰到过一次（`r6-v1-readiness-diagnosis.md`）。改起来约 5 行。
6. 其它都是小项（进下一轮或可选）。重绑提示可能挂到很久之后一条无关的"需要会话"报错上。A2 回执只说"等对方退出"，对常驻型的自动化服务不够。到期任务记录在默认文本里是转义过的原始 JSON，每个任务约 800 字节。"New worker: previous refs are invalid"这条旧提示在 5 次后台屏真实运行里次次出现，现在可以删掉。
7. 新增的状态与收益大体相称；`WorkerConnection` 删掉了自有的死亡监听，实际上比上一版简单。可以合并或删掉的"上一次发生了什么"的记录有两三处，见第 7 节。

---

## 1. 重绑（第 19 条）

### 1.1 状态机（源码推断）

`WorkerConnection.kt` 全文已读。状态有：当前附着 `connection: Attached?`、当前尝试 `attempt: Attempt?`（`attaching` 标志）、`lastFailure`、`detached`。每个 `Attached` 带 `connectedAt`、`expectedStop`、`recoveryPending`。

- **连接**：`onServiceConnected`（`:56-91`）读状态、核版本和 UID、`reconcile` 之后，在锁内发布 `Attached(candidate, reconciled, now(), recovering)`（`:75`）。`connectedAt` 取的是附着被接受的时刻，不是进程实际出生的时刻，所以进程年龄只会被低估，偏向不重绑。这个方向是安全的。
- **断开**：`onServiceDisconnected`（`:95-109`）在锁内清空连接，然后判断：`previous != null && !previous.expectedStop && now() - previous.connectedAt > MIN_CONNECTED_MS`，其中 `MIN_CONNECTED_MS = 5000`（`:293`），用严格大于。满足时调用 `bind(recovering = true)`。
- **预期退出标记**：`destroyAndConfirm` 一进入就在锁内置位，而且只置给目标相同的那个附着（`:226`）。普通停止要先通过 `prepareStop`（`:203`），失败时不会走到这里，所以不会留下错误的标记。标记挂在附着对象上，不是全局布尔值，第四节第 4 处的提示已落实。
- **线程**：Shizuku 客户端的 `connected()`、`died()` 都 post 到主线程（`S/agents10/83/shizuku/ShizukuServiceConnection.java`，Shizuku-API 主干；实施方另读了 13.1.5 的 sources jar，结论相同）。`bind()` 也经 `mainExecutor` 排队（`:116-147`），三者串行。`expectedStop` 和 `recoveryPending` 不是 volatile，但读写都在 `synchronized(this@WorkerConnection)` 里（`:97-105`、`:177-181`、`:226`），`attaching` 是 volatile。我没有发现可见性问题。
- **两路回调的先后**：本轮删掉了 App 自己挂的死亡通知，只留 Shizuku 的 `onServiceDisconnected`。Shizuku 客户端自己会对服务 binder 调用 `linkToDeath`（上游文件 `connected()` 中），所以不会漏报。`died()` 带去重标志 `dead`。我认为删掉是对的：上一版两路回调的归属和先后正是复杂度的来源。重绑必须排在 Shizuku 清缓存之后（`:107` 的注释），用 `mainExecutor.execute` 排队就保证了这一点。
- **连接超时**：`BIND_TIMEOUT_MS = 5000`（`:326`）只决定"5 秒内已有尝试就不重复"和 `readFailure` 的文字，不会主动取消尝试。

### 1.2 逐项结论

| 问题 | 结论 | 依据 |
| --- | --- | --- |
| 预期退出是否按进程记 | 是，挂在 `Attached` 上，只对目标相同的那个附着置位 | 源码 `:226`、`:279-289` |
| 最短存活取多少、怎么计时 | 5 秒，严格大于；起点是附着被接受的时刻；`SystemClock.elapsedRealtime`（含休眠，单调） | 源码 `:20`、`:75`、`:104` |
| 成熟进程的普通停止 | 不重绑。`expectedStop` 与存活时长无关，在 `destroy()` 之前置位；断开回调无论早于还是晚于 `stopExecutor` 末尾清空连接，都不会重绑（晚于时 `:99` 直接返回）。设备上 4.38 秒那次区分不出两种原因，但源码可以判断，JVM 测试 `explicitStopAndForceDoNotRebind...` 用的是 `now = 10000`，正是成熟进程（该测试我没有运行） | 源码推断；测试 `WorkerConnectionTest.kt:187-209` |
| 强制退出 | 同一入口 `destroyAndConfirm`，不重绑 | 实测 `r6-lifecycle-045…052`；源码 |
| 绑定—崩溃—绑定循环 | "连上后 5 秒内死"的不循环，构造期间就崩的只会表现为绑定超时。存活超过 5 秒、又会自己再崩的进程会无限循环（每圈至少约 6.5 秒）。这是裁决"每次意外退出都重绑一次"的直接结果，测试 `everyMatureUnexpectedDeath…` 连续断言了 3 次。见发现 F4 | 源码；测试 `:138-160` |
| 重绑进行中到来的请求 | `workerUnavailable`，文字是 "Worker is unavailable; check the device App"：断开时写的 `lastFailure` 会被排队的 `bind()` 置空（`:137`），5 秒内 `readFailure()` 返回空。见发现 F1 | 源码；实测 `040`（kill 后新进程尚未出现） |
| 在途请求 | `target.call` 抛 `DeadObjectException`，网关按"不确定"回报（`NetworkGateway.kt:268`、`Wire.kt:86`），主机显示 UNKNOWN；不重放 | 源码 |
| 旧会话、后台屏 | 都在旧进程内存里，随进程消失；实测 `043` 只剩 display0，旧任务不在活动表里 | 实测 |
| 旧引用 | 新进程的引用从随机的 9 位数起步（`UiRefs.kt:8`），解析只查当前样本的 `targets`（`UiSnapshot.kt:76-83`），旧引用一律报 `stale`，不会落到新页面上 | 源码；实测 `042` 用旧引用 click 得到 not sent |
| "界面清理未确认"的封锁 | `pendingCleanup` 是旧进程的内存，随之作废，新进程从空开始。旧进程持有的 UiAutomation 登记会在其死亡时由系统收回（推断：平台按登记者 binder 的死亡注销；本轮没有对着 AOSP 复核） | 源码 + 推断 |
| 调用者第一条命令 | 需要会话的命令：`sessionRequired`，后面附重绑提示，并照旧写明 `session open --display main|virtual`。调用者据此重开会话、重新采样即可，不会误用旧引用 | 实测 `042`：`click: not sent; No current session; run session open --display main|virtual explicitly; Worker exited unexpectedly and was automatically reconnected; old sessions, work displays and their tasks are invalidated; code=sessionRequired`（我用主机渲染函数复现了默认文本） |
| 附一次后清掉 | 是，`recoveryNotice` 在锁内读后清零（`:177-181`）；实测 `044` 第二次已经没有提示。另见发现 F2（可能挂到很久以后） | 实测 + 源码 |
| 与 Shizuku 重启的自动绑定是否重复 | 不重复。顺序是"先断开，再收到 binder"时，第二次 `bind()` 被"5 秒内已有尝试"挡住（`:122`）；反过来时，旧尝试的断开回调因 `attempt !== this` 被忽略（`:93`、`:98`） | 源码 |
| Shizuku 不可用时 | 重绑在 `requireAvailable` 失败，写入 `lastFailure`，不自重试（测试 `:174-184`）。Shizuku 回来后由粘性监听再绑定，但这次 `recovering = false`，提示丢失，见 F3 | 源码 |
| App 进程被杀 | 常驻进程是 daemon，以 UID 2000 运行，App 被杀后仍然存活；`WorkerConnection` 的内存状态（标记、提示）随 App 进程丢失。网关是 `START_NOT_STICKY`，要用户重开 App；重开后附着到同一个常驻进程，会话和后台屏照旧，没有提示，这是对的。若常驻进程也在这期间死了，调用者只看到 "No current session"，没有提示 | 源码推断，无设备证据 |

### 1.3 设备证据怎么读

- `r6-lifecycle-039`：附着约 3 分钟后 `kill -9` 一次；`041` 没有点刷新，新进程已经 ready；`042` 旧引用请求零派发，附了提示；`044` 不再重复。这条链完整，支持实施方的结论。
- `048`、`055`：强制退出、普通停止之后的 `status` 都是同一句 "Worker is unavailable; check the device App"。它和"正在附着"、"首次启动中"三种状态共用一句话，见 F1。

---

## 2. 自动化登记（A2）

### 2.1 源码与平台

- `AutomationCleanup.register`（`AutomationCleanup.kt:27-45`）：先置 `registrationPending = true`，再调用连接。抛错时用 `rootCause` 拆一层 `InvocationTargetException`（`Wire.kt:73`）。原异常是 `TimeoutException` 时保留待注销，其余一律清掉。所以超时之后会做一次 `disconnect()`：这时 UiAutomation 的状态是 FAILED，平台允许 disconnect（AOSP 16 `UiAutomation.java:399-411` 置 FAILED）。同步抛错时状态停在 CONNECTING，此时调用 disconnect 必然抛错，上一轮正是在这里把自己锁死的。本轮不再对这种情况调用 disconnect，回调线程照常停止。我核实这个修法正确。
- "已登记"的识别：`cause is IllegalStateException && message.contains("already registered")`。平台在 system_server 里抛 `IllegalStateException("UiAutomationService " + … + "already registered!")`，跨 binder 后异常类型和消息保留。Android 11 起 `Parcel.readException` 会把远端栈作为 cause 追加上去，但这里只看顶层异常，不受影响。测试 `UiCleanupTest.kt:36` 用的就是没有空格的真实写法。厂商 ROM 的文字没有核过；对不上时按"其它同步抛错"处理，不会再锁死。
- 回执：`busy`，`subject=automation`，文字是 "Another automation tool holds the UiAutomation registration; takeover has no effect; wait for it to exit"。码表 `busy` 一行按主体分两句写（`contract/reason-codes.json:15`），首页（`SKILL.md:24-25`）和连接专题（`connection.md:96`）一致。主机渲染为 `…: not sent; …; code=busy; subject=automation`。
- 调用者重试是安全的：每次重试只是一次快速失败的同步调用，不留封锁。

### 2.2 回执是否说清了"该怎么办"

- 占着登记的若是 `uiautomator dump` 这类短命进程，等一两秒重试即可，文字够用。
- 若是常驻型的自动化服务（UIAutomator2 或 Appium 的服务端、Maestro、正在跑的 instrumentation 测试，或者用户另开的同类工具），"等它退出"可能永远等不到。调用者能做的只有告诉用户。AndroidCtl 认不出是哪个工具，平台的异常里没有对方的身份。见发现 F5：建议回执加半句"若不退出，请用户停止另一个自动化工具"。这与裁决原文"等对方退出"不冲突，只是补上第二条出路。
- 除了等待，没有更好的办法：平台全局只允许一个 UiAutomation 登记，没有共享或抢占的接口；而接管只改租约，不改登记，裁决说"接管无效"是对的。

### 2.3 "界面清理未确认"的封锁还会在哪些情形出现

由 `UiBackend.cleanupStatus()` 汇总（`UiBackend.kt:50-57`、`:640-656`）。仍会出现的情形：

1. 登记超时之后，那一次 `disconnect()` 抛错。按设计只尝试一次（`disconnectAttempted`，`AutomationCleanup.kt:54-63`）。
2. 回调线程在期限内没有停下（`UiBackend.kt:545-558`）。
3. 关闭后台屏失败（`pending.stream?.close`）。
4. 本地样本或输入状态清理抛错。

解除办法：`session close` 会对现存的每一项再试一次（`cleanupRetry`，`UiBackend.kt:595-600`），但不重试 disconnect。用户点"Stop idle executor"（应用主动停止，不重绑）再点"刷新状态"，就能得到一个干净的新进程。指引里已经写了"之后还要点刷新"（`Sessions.kt:422-423`），第四节第 4 处的最后一条已落实。常驻进程意外退出后重绑，封锁也随之消失。A2 那条路径不再产生封锁。

---

## 3. 会话到期与后台屏（第 12 条）

- **触发**：常驻进程每秒执行一次 `sessions.expire()`（`WorkerService.kt:39-56`），各会话命令和 `status` 也会顺带调用。到期在会话锁内作废租约，并通过 `ui.clear(lease)` 把清理排进界面单线程执行器（`UiBackend.kt:570-582`）；锁内没有阻塞调用。
- **任务表在哪查**：在执行器里先 `recordTasks`、后 `clean`（销毁后台屏），不在会话锁内，与第四节第 9 处的要求一致。查询本身没有超时；它若卡住，会拖住后续界面命令（同一个执行器），但不会拖住会话管理和 `status`。裁决没有要求加超时，可以接受。
- **失败和"尚未完成"**：`SessionExpiry.taskRecord` 初值是 "Task record pending"，查询失败时写 "Task record unavailable: …"（`SessionExpiry.kt:7`、`:34-35`）。它是 volatile，字符串整体替换，已经交出去的消息不会被改写。由于定时检查每秒一次，等到调用者的下一条命令时，记录通常早已完成；"pending"只在极短的窗口里出现。
- **剩余秒数从哪来**：`Sessions.idleRemainingSeconds`（`Sessions.kt:374-378`），向上取整；有资源占用时按"空闲不流逝"算。`status` 走 `localStatus`/`callerStatus`，只调用 `expire()`，不改 `touched`，所以确实不续期（`:350-358`）。实测 `028`：`idleRemainingSeconds=10`；之后到期，`029` 已经没有租约。
- **关闭被拦下后的交互**：实测 `027`：`session close` 回 not sent，`code=externalTasks`，reason 是 "External tasks remain on the work display; ask the user, then choose --return-tasks or --discard-tasks"，并附 `taskDisposition`、`target`、`idleRemainingSeconds`。主机默认文本另加一行提示两种选项。三条出路都可以照做：`--return-tasks`（要征得用户同意）、`--discard-tasks`，或者继续工作或用 `snapshot` 保持（指引 `displays.md` 写了"snapshot renews; status does not"）。注意：被拦下这一刻本身会重置一次空闲计时（`finished(resource)` 写 `touched`），所以回执里的剩余秒数约等于整个空闲时长。实测 10 秒的租约回的就是 10。这与风险提示一致，是如实报告，不是缺陷。
- **到期记录存什么、何时清**：按会话身份（配对加会话名）存显示种类、displayId 和任务记录（`Sessions.kt:335-343`）。同一身份成功开会话时（`publish`，`:233`）或 `session close` 时（`:194`）清掉；常驻进程退出时丢失。没有其它清理路径，记录数受会话名的种类限制，内存可忽略。
- **到期后第一条命令的回执**：
  - 界面类、shell 和文件的启动：`sessionRequired`，带到期描述、后台屏编号、任务记录、"To continue on the work display, use --display virtual"，以及完整的 `session open --display main|virtual` 写法（实测 `030`）。
  - 不写 `--display` 的 `session open`：在主屏新开，首行附同一句提示，然后清掉记录（实测 `032`：`session open: done; target=main/display0; Target established; To continue on the work display, use --display virtual`）。
  - 写了显示的 `session open`、`session takeover`：照常新开，不附提示，记录清掉（源码 `Sessions.kt:128`）。
  - `session close`：回 "No current session"，记录清掉，没有到期信息。这是裁决列出的已知缺口。
  - `status`：`callerState=none`，不区分"到期"和"从未打开"（`Sessions.kt:380-385`），见 F11。
  - `wake`：不需要租约，照常执行。
- **默认文本的可读性**：任务记录是 `JSONObject.toString()` 嵌在消息字符串里的。主机默认文本再加一层转义后，每个任务约 800 字节，满是 `\"` 和 `\\/`。我用 `r6-lifecycle-030` 的原件、调用主机 `ui_output.result_line` 复现过（脚本 `S/agents11/93/render2.py`）。上限是每类 8 个任务、8 KB，最多 4 类。见 F6。

---

## 4. `session open` 的显示选择（第 25 条）

### 4.1 是否在同一把锁里决定

是。主机不写 `--display` 时请求里不带 `displayKind`（`app.py:845-852`）；设备端 `DisplayKind.requested(args, allowOmitted = mode == OPEN)`（`TargetManagement.kt:80`）。`Sessions.prepareChange` 在一个 `synchronized` 块里依次做准入、到期检查、归属检查，再算 `selected = target ?: checked?.target ?: MAIN`，复用或新建也在同一个块里完成（`Sessions.kt:110-130`）。主机没有先查后发，第四节第 10 处的建议已落实。

### 4.2 细则逐条

| 细则 | 结论 | 依据 |
| --- | --- | --- |
| 默认主屏只对新开生效；已有会话跟随原显示，不触发替换 | 对 | 实测 `016`（新开：main/display0）、`024`（复用 virtual/display2，idle 不变） |
| `session takeover` 必须写 `--display` | 对；主机在参数阶段拒绝，设备端也会拒绝 | 本地复现：`session takeover: not sent; session takeover requires --display. …; code=arguments; argument=display`，退出码 2 |
| `--replace` 必须写 `--display` | 对 | 本地复现：`session open: not sent; --replace requires --display. Use … --replace.; code=arguments; argument=display`（文字里有个 ".;" 的标点小瑕疵） |
| 首行总写显示 | 成功时对，新开、复用、替换、接管四种情形都写 `target=KIND/displayN`。失败或 UNKNOWN 时目标为空，不写（此时本来就没有确定的显示，合理） | 源码 `app.py:203-206`；本地复现见下 |
| 后台屏上到期的报错里加提示 | 对 | 实测 `030` |
| 首页示例仍写 `--display main` | 对 | `SKILL.md:18` |
| 缺参数时把 `\n\t` 原样打出来的报错 | 已修（`app.py:1022` 合并空白） | 本地复现 `--display foo`、`--display` 缺值都是单行 |
| 第三次答复：不写显示的 open 碰到后台屏到期记录，在主屏新开、首行附提示、再清记录 | 对 | 实测 `032`、`033`；源码 `Sessions.kt:128`、`:233` |
| 第三次答复："需要会话"的报错照旧写 `--display main|virtual` | 对 | 实测 `042`、`044` |

四种情形首行的真实文字（主机渲染函数本地复现，与设备原件 `016`、`024`、`032` 一致）：

```
session open: done; target=main/display0; Target established                       （新开）
session open: done; target=virtual/display7; Target reused; idle configuration unchanged （复用）
session open: done; target=main/display0; Target established                       （--replace 替换）
session takeover: done; target=virtual/display9; Target established                （接管）
```

替换、接管和新开在首行上无法区分；`previousSession` 只在 `--json` 里，见 F12。

### 4.3 组合情形（源码推断）

- 另一个配对，或同一配对的另一个会话名占着租约：无论写不写显示，一律报 `busy`（`Sessions.kt:159-163`）；接管只在同一配对内有效（`:161`）。没有因为省略显示而越权。
- 同名会话在另一种显示上：不写显示就复用；写了另一种显示但没带 `--replace`，报 `displayMismatch` 并给出可照抄的写法（`:173-178`）。
- `--replace` 加同种显示：复用，不重建后台屏（`:114`）。这是上一轮就有的行为，`--help` 和 `displays.md` 都写了。
- 用户在设备上释放了会话（`sessionReleased`）之后，不写显示的 `session open` 会直接在主屏重开（`checkChange` 只在接管时检查 released，`:156`）。这在上一轮写 `--display main` 时同样可行，默认值只是少打几个字。报错和码表都要求先问用户，不必为此改代码。

### 4.4 `--help`（本地运行）

```
Open a session: follow its existing display, or use main for a new session.
Use --display to choose main or virtual. Reusing the same kind preserves its
target, refs, images and idle configuration.
```

够用。它没有说"主屏就是用户正在用的那块屏"，但 `--display` 的说明写的是 "Use the main screen or an owned virtual work display"，可以接受。

### 4.5 默认主屏的误操作风险（用户不顾评审方推荐的选择）

我的判断是风险低，现有提示足够，不建议推翻这条裁决：

- 调用者想用后台屏，必须写过一次 `--display virtual`，它的上下文里有这条命令；大模型重开时通常照抄。真实运行里 5 次后台屏题都显式写了 `--display virtual`，20 次运行没有一次省略显示（`efficiency/r6-*-candidate-*/cli-index.json`）。
- 到期之后，调用者先看到的通常是 `sessionRequired`，里面同时写着"要继续用后台屏请写 `--display virtual`"和"显式打开"。即使它照样省略显示，开会话这一步本身不触碰界面，成功回执的首行已经写明 `target=main/display0` 并附提示；要改回后台屏，它下一条 `session open --display virtual` 会拿到 `displayMismatch`，被引导去写 `--replace`，多一个回合就能纠正。
- 真正"无提示落到主屏"的只有记录丢失的几条路径：先发了 `session close`、用户在设备上释放、常驻进程重启（重绑提示只说后台屏已作废，不说原会话在哪块屏；这是第三次答复的明确选择）、配对被撤销。这几条都已在裁决里列为已知缺口。
- 可以零成本再降一点风险：省略显示而新开主屏时，首行写成 `target=main/display0 (default)`，让调用者看出这是默认选的而不是它要求的。列为可选（F7）。

---

## 5. 管理页 `singleTask`（第 21 条）

- 从桌面重入会清掉叠在上面的设置页：实测 `r6-lifecycle-007…011`（电池优化设置页叠在管理页的任务上，任务里有 2 个页面；从桌面重入后只剩管理页，同一任务编号）。这是裁决写明的代价。用户在设置页里中途切走、再从桌面回来，要重新进入设置页。影响小。
- 通知入口：`PendingIntent.getActivity(this, 0, Intent(this, MainActivity::class.java), …)`（`GatewayService.kt:44-46`），没有 flags。对 `singleTask` 而言效果与桌面重入相同，也会清掉上面的页面（推断，没有设备证据）。
- `MainActivity` 不重写 `onNewIntent`，也不读启动参数，没有逻辑依赖新 Intent（第六轮核过，本轮没有变化）。
- 深链：清单里只有 MAIN/LAUNCHER（`AndroidManifest.xml:16-21`），没有其它入口，不受影响。
- 多窗口：`singleTask` 只允许一个实例，不能分屏开两个管理页。这不是需求，可以接受。
- 后台屏：调用者打开 AndroidCtl 自身会被拒绝（`ActivityLaunch.kt:197`），所以不会把管理页的任务搬到后台屏去。
- Shizuku 授权界面在 Shizuku 自己的任务里，不受影响（推断）。

结论：副作用只有已知的那一条，不必处理。

---

## 6. 首次 `status` 报常驻进程未就绪（后台屏题第 4 次的准备阶段）

**事实**（`ST/r6-v1-readiness-diagnosis.md` 与原件）：网关前台服务启动后约 40 毫秒就开始监听；首次 `status` 在这之后约 0.4 秒发出，再过约 0.25 秒完成设备认证；常驻进程的 binder 在约 1.8 秒后才返回，之后 App 还要做 `status` 和 `reconcile`。回执是 `workerUnavailable`，文字为 "Worker is unavailable; check the device App"。三分钟后的 `status` 已就绪。实施方"首次异步附着的窗口期"这个解释有时间线和源码支持，我同意：`GatewayService.kt:135-158` 先 `newWorker.start()`、再启动网络，网络不等附着完成；`WorkerConnection.call` 立即 `requireTarget`，附着前就报 `workerUnavailable`。

**真实使用中会不会碰到**：

- 用户手动点"启动网关"，再去告诉调用者开始，中间的人手延迟一般超过 2 秒，很少碰到。
- 下面几种情形会碰到：脚本化或自动化的启动（正是这次的情形）；网关刚被用户重新打开；App 被系统或更新杀掉后重开；**意外退出后自动重绑的那一两秒**（第 19 条新增的窗口，实测 `040` 显示 kill 后新进程尚未出现）；附着慢的设备（低配模拟器、冷启动，Shizuku 拉起进程本身约 1.6 秒）。

**回执有没有说"稍后重试"**：没有。文字说 "check the device App"，码表 `workerUnavailable` 一行（`reason-codes.json:150`）说 "Ask the user to use Refresh status in AndroidCtl to attach a worker"，属于"用户"类。调用者会停下来去问用户，用户点刷新时 `bind()` 发现已有尝试，什么也不做（`WorkerConnection.kt:122`），实际上问了也白问。附着超过 5 秒时，文字变成 "Worker binding timed out; use Refresh status"，而附着可能仍在进行。

**算不算该修的首次使用缺陷**：算，但是小缺陷（F1）。它不会造成误操作，只是多一次人工往返，而且恰好发生在第一印象最重要的首次连接上，以及自动重绑这个本来"免人工"的路径上，让第 19 条的收益打了折扣。

---

## 7. 新增状态与收益是否相称

本轮这条链新增或改动的状态：

| 位置 | 新增 | 评价 |
| --- | --- | --- |
| `WorkerConnection` | `connectedAt`、`expectedStop`、`recoveryPending`、`Attempt.recovering`；删掉了自有的死亡监听、`Attached.close/linked` | 净简化。`WorkerBinding.kt`（69 行）只是把 Shizuku 的调用抽成接口，便于用真实的 `WorkerConnection` 做测试，没有复制状态机，可以接受 |
| `Sessions` | `expired` 由集合改为映射，值为 `SessionExpiry`；`Lease.expiry`；`SessionChange.expiryGuidance`；`idleRemainingSeconds` | 必要。`Lease.expiry` 只是为了把记录交给 `ui.clear`，也可以直接作为 `clearLeaseUi` 的参数传过去，省一个字段（可选） |
| `SessionExpiry.kt` | 38 行 | 相称 |
| `AutomationCleanup.register` | 约 15 行 | 相称，修的是真实的锁死 |

"上一次发生了什么"的记录现在至少有这些：`Sessions.released`（集合）、`Sessions.expired`（映射）、`Attached.recoveryPending`、`WorkerConnection.lastFailure`、`UiSnapshot.first`（"New worker: previous refs are invalid"）、`GatewayService.error/workerObservationError/stopFailure`。可以删或合并的：

1. **删掉 `UiSnapshot.first` 及其提示**（F8）。它从基线起就有（`f2f55bb`），作用是告诉调用者旧引用失效。现在引用每个进程随机起步，旧引用必然被拒为 `stale`，重绑另有提示。它在真实运行里是纯噪声：5 次后台屏题的第一条 `open` 首行都带着 "New worker: previous refs are invalid"（`efficiency/r6-v1-candidate-0{1..5}/evidence/cli/*/child.stdout`），而那时调用者根本还没有引用。删掉约 8 行。
2. **`released` 和 `expired` 可以合并**成一张"会话身份到结束原因"的表，`callerState` 就能顺带报 `expired`（F11）。收益小，可选。
3. `lastFailure` 与 `readFailure()` 现算的超时文字，可以一并改成按状态给出文字（附着中、用户停止、断开、被拒），与 F1 一起做。

总体判断：这条链的新增代码与收益相称，没有发现为了边角情形而堆出来的机制。

---

## 发现清单

### F1 附着窗口里的 `workerUnavailable` 让调用者去找用户，而不是稍后重试

- 现象：首次启动和意外退出后自动重绑的那一两秒里，命令得到 "Worker is unavailable; check the device App"，码表让调用者请用户点刷新；附着超过 5 秒时又说 "binding timed out"。用户停止之后的状态也是同一句话（实测 `048`、`055`）。
- 触发：附着尚未完成时到来的任何命令。
- 是否已在真实任务里发生：在真实准备阶段发生过一次（`runtime/api34/commands/3896-…/stdout.bin`），没有发生在模型运行中。
- 影响：多一次人工往返；削弱了自动重绑"免人工"的效果。
- 最小修正：`readFailure()` 在 `attempt.attaching` 且未超时时，返回 "Worker is attaching; retry in a few seconds"；重绑尝试可以说 "reconnecting after an unexpected exit"。`stopExecutor` 之后写 "Worker stopped on the device; use Refresh status"。码表 `workerUnavailable` 一行补一句"消息说正在附着时，等几秒重试"。不新增原因码，符合第 24 条。
- 维护代价：约 5–10 行代码、1 条 JVM 测试、码表一句。
- 分类：确定缺陷（小）／**进下一轮**（成本很低，若合入前顺手改也不冒风险）。

### F2 重绑提示可能挂到很久以后一条无关的报错上，也可能被别的调用者拿走

- 现象：提示只在新进程回 `sessionRequired` 时消费（`WorkerConnection.kt:149-181`）。如果重绑后调用者的第一条命令就是 `session open` 并且成功（例如它从 UNKNOWN 的回执里自己判断出要重开），提示会一直留着，直到很久以后比如一次空闲到期，才附在那条到期报错后面，让人以为常驻进程又崩了一次。多个配对共用设备时，谁先碰到 `sessionRequired` 谁拿走提示。
- 是否已发生：没有，源码推断。
- 影响：误导排查，不会导致误操作。
- 最小修正：新进程上第一次成功的会话命令（或任意一次成功调用）就清掉 `recoveryPending`。约 2 行。
- 分类：确定缺陷（很小）／进下一轮。

### F3 重绑本身失败、之后由别的途径恢复时，提示丢失

- 现象：比如 Shizuku 与常驻进程一起退出，重绑因 Shizuku 不可用而失败；Shizuku 回来后粘性监听用 `recovering = false` 再绑定。调用者只看到 "No current session"。
- 影响：可以忽略：报错本身正确，旧引用照样被拒。
- 分类：设计取舍／不必处理。

### F4 存活超过 5 秒、会自己再崩的进程会无限重绑

- 现象：每圈至少约 6.5 秒（5 秒加上 Shizuku 拉起进程），没有次数上限。
- 是否已发生：没有。这需要一种不靠外部刺激、启动 5 秒后自己崩的缺陷；由命令触发的崩溃受调用者的节奏限制。
- 依据：用户裁决明确是"每次"；测试断言的就是连续重绑。
- 可选修正：自动恢复来的进程若在 60 秒内又死，就不再自动重绑。约 3 行。
- 分类：设计取舍／不必处理（可选）。

### F5 A2 回执对常驻型自动化服务只说"等它退出"

- 现象与影响：见 2.2。调用者可能空等或反复重试。
- 最小修正：在原文后加 "if it does not exit, ask the user to stop the other automation tool"，码表同步一句。不改原因码。
- 分类：可选建议／进下一轮。

### F6 到期任务记录在默认文本里是双重转义的原始 JSON

- 现象：每个任务约 800 字节，`baseActivity`、`topActivity`、`realActivity` 等字段重复相同的组件名，再加 `\"`、`\\/` 转义；本地复现见第 3 节。上限是每类 8 个、8 KB，最多 4 类。
- 是否已发生：设备检查 `030` 中出现（1 个任务，817 字节）；不在模型运行中。
- 影响：报错可读性差；调用者要转告用户"哪些 App 随后台屏消失了"，真正需要的只有包名和任务编号。
- 最小修正：消息里只写 `movedTasks: com.android.contacts(115)` 这样的摘要，完整事实放进结构化字段（`--json` 可见）。
- 分类：可选建议／进下一轮（属于默认文本瘦身的范畴）。

### F7 省略显示而新开主屏时，首行看不出这是默认选的

- 分类：可选建议／不必处理。改法：首行写成 `target=main/display0 (default)`，约 2 行。

### F8 "New worker: previous refs are invalid"是纯噪声，可以删掉

- 依据：真实运行 5/5 出现；理由见第 7 节。
- 分类：可选建议／进下一轮（基线遗留，不是本轮引入）。

### F9 （顺带，范围外）`session close` 成功时默认文本打印完整的任务处置 JSON

- 真实运行里，每次后台屏题的收尾都打印一大段 `taskDisposition`（`efficiency/r6-v1-candidate-01/evidence/cli/1795624100567455700-…/child.stdout`），而状态只是 `clear`。转给输出专题。
- 分类：可选建议。

### F10 （顺带）`sessionRequired` 的 not sent 回执后面跟着 `dispatchStarted=unknown`

- 本地复现：`click: not sent; …; code=sessionRequired; dispatchStarted=unknown`。会话准入阶段被拒，必然没有派发，和 not sent 自相矛盾。来源是 `ui_output.py:95-101`：数据里没有这个字段时一律写 unknown。我没有核实是不是本轮引入的。
- 分类：可选建议，转给输出专题。

### F11 到期后 `status` 只报 `callerState=none`

- 分类：可选建议。可以与第 7 节第 2 条一起做。

### F12 替换、接管的首行与新开相同，不显示被替换的会话名

- 分类：可选建议／不必处理。`--replace` 是调用者自己要求的，`--json` 里有 `previousSession`。

---

## 已核实无问题

1. 预期退出标记按附着对象记录，在发出销毁之前置位；`prepareStop` 失败时不置位；旧进程的停止不会抑制新进程的恢复（源码 `:201-226`；测试 `:187-225`）。
2. 成熟进程的普通停止不重绑（源码判断；设备上 4.38 秒那次虽然区分不出原因，但不影响结论）。
3. 最短存活时长：5 秒，严格大于，从附着被接受时起算，单调时钟。
4. 所有绑定和断开回调都在主线程串行；共享字段都受锁或 volatile 保护；删掉自有死亡监听不会漏报。
5. 与 Shizuku 重启时的自动绑定不会重复。
6. 在途请求遇到进程死亡报不确定，不重放；实测 `042` 零派发。
7. 旧引用无法命中新进程的页面（随机起步）。
8. 重绑提示只附一次，并照旧写明 `session open --display main|virtual`（实测 `042`、`044`）。
9. A2：超时与同步抛错分得清；同步抛错不再锁死；"已登记"识别正确，Android 11 起追加的远端栈不影响判断；回执复用 `busy` 加主体，按裁决分两句写。
10. 到期在锁内，任务查询在锁外、在销毁之前；pending 和 unavailable 都如实写；已经交出的消息不会被改写；定时检查让第一条报错通常就带着完整记录。
11. `status` 不续期；剩余秒数向上取整；被拦下的关闭回执带剩余秒数，三条出路都可以照做。
12. 第 25 条六条细则与第三次答复两条全部落实；主机没有"先查后发"；省略显示不越权；缺参数报错已经是单行。
13. `singleTask` 只有已知的那一个副作用。
14. "界面清理未确认"在重绑后随旧进程作废，没有被带过来；提示写了"停止之后还要点刷新"。

## 未能核实的事项

- 我没有运行 JVM 测试（`WorkerConnectionTest`、`SessionExpiryTest`、`UiCleanupTest`、`SessionsTest`）；上文引用的测试只读过源码。
- Shizuku 停止或撤销授权时常驻进程是否退出；Shizuku 13.1.5 服务端的行为（只核了客户端，而且是主干版本）。
- 常驻进程死亡时，系统收回其 UiAutomation 登记的具体机制（推断，本轮没有对 AOSP 复核）。
- 厂商 ROM 上"已登记"异常的文字；Android 14、15 的确切文字（沿用第六轮的核查结果）。
- 通知入口下 `singleTask` 的实际表现；多窗口下的表现。
- 真机上的附着耗时，以及首次连接窗口在真实用户流程中出现的频率（只有模拟器上的一次）。
- 附着超过 5 秒、用户又点刷新时，取消旧尝试再重新绑定在 Shizuku 一侧会怎样（源码推断不会循环，没有证据）。
- F10 是否本轮引入。
