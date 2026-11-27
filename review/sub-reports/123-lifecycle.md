# 任务 123：会话、连接、重连、清理、资源归属（行为链深审，`409e853`）

## 结论（先读这里）

1. **没有需要挡合入的缺陷。** 设备端常驻进程（下称 worker，经 Shizuku 以 shell 身份运行的执行进程）的连接状态机里没有"进得去出不来"的状态；任何卡住的情形都有一条手动出路（设备上的 Refresh status）。重连期间到达的命令一律 `not sent`，正在执行时进程死掉的命令一律 UNKNOWN。主机和设备两端都没有自动重发，所以不存在"动作被悄悄重做一遍"的路径（源码推断，见 §1.3）。
2. **自动重连这套规则是一层很薄的保险，规则里的例外基本没有实际用处。** 在 48 次正式尝试里，worker 意外退出 0 次，自动重连 0 次，60 秒例外 0 次。自动重连只在定向验证里由操作者故意杀进程时触发过：2 次自动重连，1 次被 60 秒例外挡下。它救回的只是"不用请用户点一次 Refresh"，旧会话、后台屏和任务照样全部作废。代码已经写好，有 16 个 JVM 测试，留着成本很低，我不建议删。但 60 秒例外与"自动连上的进程不再自动重连"这种更简单的规则，在现有证据下效果分不出来。这是设计意见，见 §4。
3. **确定缺陷都是小项：**（a）几种"请点 Refresh"的文字有的写 "ask the user"，有的不写，调用者会以为可以自己去点；（b）"旧会话已作废"的提示是全设备只有一份，可能被别的配对或会话名消费掉或清掉（上一轮 F2 的后一半没有修）；（c）`app.py` 新加的用法提示接在 click 原文后面，又产生了 `.;` 标点；（d）指引里同一个字段写了两个名字，`callerSession` 和 `callerState`。
4. **证据不足：** 定向验证 041–060 号命令的原始输出不在证据包里，只有叙述。我用同一窗口的 logcat 核对了四次绑定和两次死亡的时间，结论与叙述一致，但三条回执的原文（"connecting"、"60 秒内不再自动重连"、作废提示）没有原件可核。
5. **本轮 `app.py` +127 行、`WorkerConnection.kt` +94 行，逐块看都在落实上一轮的裁决，没有发现无人使用的路径。** 但 `WorkerConnectionTest.kt` 为这 94 行加了 331 行测试，测试与产品代码之比超过 3:1，围着一个实测零触发的机制。维护者应当知道这个比例（§5）。

---

## 1. 链条逐段核对

### 1.1 调用者开会话 → 主机守护进程 → 设备

- 主机按"查询类"和"动作类"各维持一条到设备的 WebSocket（`coordinator.py:91-134`，`Device.exchange`）。连不上或认证失败一律算 `not sent`（`coordinator.py:197-205`）。发出之后收不到回复，算 `TransportFailure` → UNKNOWN（`transport.py:452-466`，`coordinator.py:195-196`）。**主机没有任何自动重发**：我看了 `coordinator.py`、`transport.py`、`local_call.py`、`ui_client.py` 里所有的循环。`local_call.py:136` 只重试连接本地守护进程，`ui_client.py:276` 只按偏移续读已经发布的数据块，都不重发设备命令（源码推断）。
- 同一会话的请求在同一条连接上排队；别的会话占着连接时，会立刻拿到 `connectionBusy`（`transport.py:398-411`）。本轮新加的 `busy_message`（`transport.py:146-164`）会写出占用者，以及占用者剩余的请求预算。我核对过：它读的是 `requests` 字典里的第一项。等待者超时时，自己的票据已经在 `acquire` 的异常分支里删掉了（`:402-406`），所以报出的确实是占用者。第 424 行用新的期限覆盖同一个键，字典顺序不变。文字写的是 "retry this unsent command"，与"未发出"一致。**核对无问题。**
- 设备网关：RPC 进 `NetworkGateway.handle`，异常时按 `uncertain(error)` 标记（`NetworkGateway.kt:262-268`，`Wire.kt:90`）。`Rejected` 用它自带的标志（`workerUnavailable` 为 false），其它异常一律算不确定，例如 worker 死亡时的 `DeadObjectException`。**核对无问题。**

### 1.2 worker 死亡 → 自动重连 → 恢复资格（`WorkerConnection.kt`，409e853）

状态：`connection`（当前附着）、`attempt`（当前绑定尝试，带 `attaching`、`automatic`）、`lastFailure`、`detached`、`recoveryNoticePending`（作废提示是否待送出，全对象一份）、`Attached.automatic`、`Attached.connectedAt`、`Attached.expectedStop`。所有回调和 `bind()` 都在主线程串行执行（`bind` 经 `mainExecutor.execute` 排队，`:137-138`），共享字段的读写都在 `synchronized(this)` 里。

| 情形 | 行为 | 调用者看到的文字 | 依据 |
| --- | --- | --- | --- |
| 绑定中，未满 5 秒 | 新命令被 `requireTarget` 拒绝 | `Worker is connecting; retry in a few seconds`，`not sent` | `:39-50`，`:308-315` |
| 绑定中，超过 5 秒 | 不主动取消；Refresh 会重新绑定 | `Worker binding timed out; use Refresh status on the device` | `:42-43`，`:143` |
| 手动连上的，存活不到 5 秒就死 | 不自动重连 | `Worker disconnected; use Refresh status on the device` | `:115-123` |
| 手动连上的，存活超过 5 秒后死 | 自动重连一次 | 先是 connecting，然后新进程回 `sessionRequired` 并附作废提示 | `:114-129` |
| 自动连上的，存活不到 60 秒就死 | 不自动重连 | `Worker exited again within 60 seconds of an automatic reconnection; it will not reconnect automatically; ask the user to use Refresh status on the device` | `:120-122` |
| 自动连上的，存活超过 60 秒后死 | 再自动重连一次 | 同"手动连上、存活超过 5 秒"那一行 | `:115-116` |
| 停止或强制退出（`expectedStop`） | 不重连 | `Worker stopped on the device; use Refresh status` | `:110-113`，`:284-289` |
| 附着时被拒（版本或 UID 不对、附着中途死亡） | 不重连，之后的断开回调被忽略（`:102`） | `Worker attachment rejected: …` | `:83-90` |
| 绑定时 Shizuku 不可用 | 不重连 | Shizuku 的原因，例如 "Start Shizuku using ADB…" | `:163-167` |

**有没有"进得去出不来"的状态**：我逐一构造了上表每一行之后的出路，都能经 `bind()`（Refresh 按钮，或 Shizuku 重新就绪触发的 `ready()`）回到"绑定中"。`bind()` 只在两种情况下直接返回：worker 仍然活着；或者已有一次不到 5 秒的尝试正在进行（`:141-147`）。后一种在 5 秒后自然失效。附着被拒之后 `attaching=false`，下一次 `bind()` 会解绑旧尝试、新建尝试。**结论：没有死状态**（源码推断）。

**60 秒窗口的起点和归属**：起点是 `Attached.connectedAt`，即身份、版本和 reconcile 全部通过、附着被接受的那一刻（`:78`），用的是单调时钟。窗口挂在每一次附着对象上，不是全局计时器。一部设备只有一个 `WorkerConnection`，也只有一个当前租约（`Sessions.current`），所以并发和多会话不会把别的会话的计时算进来。手动绑定撞上进行中的自动尝试时，把来源降为手动（`:144`）；反过来，自动绑定不会把手动来源改回自动。如果自动尝试已经附着成功，之后再点 Refresh 是空操作（`:141`），这次连接仍然算自动的。这与 `connection.md:115-116` 的写法一致。**核对无问题。**

**设备侧佐证（实测，logcat）**：`EV/runtime/api34/startup-r7-g1a-check-01/logcat-all.log`：

- 17209/17211 行：首次手动绑定（17:07:05.008）。
- 第一次故意杀进程：`Binder … is dead` 在 17:22:17.961；17:22:18.030 `UiAutomation service owner died`（22177 行）；17:22:18.035 新的服务记录，即自动重连（22181 行）；17:22:19.089 拿到 binder。从死亡到新进程可用约 1.1 秒。
- 第二次故意杀进程：17:22:59.840 新进程死亡，存活约 40.8 秒，不到 60 秒。到 17:23:59.090 用户点 Refresh 之前**没有新的服务记录**，说明 60 秒例外确实挡住了。
- 17:23:59.090 手动绑定（22486 行）；17:24:42.555 又一次自动重连（22738 行）。前一个手动进程存活约 42.7 秒，超过 5 秒，所以恢复了自动重连资格。
- 这同时解决了上一轮的一个未核事项：worker 死亡时，系统会收回它的 UiAutomation 登记（22177 行）。

### 1.3 重连期间与进程死亡时的命令回执

- **死亡回调到达之前的毫秒级窗口**：`connection` 还没有清空，但 `pingBinder` 已经失败，而 `lastFailure` 是 null。这时命令拿到兜底文字 `Worker is unavailable; check the device App`（`:312`）。按 logcat，这个窗口只有几十毫秒，可以忽略。
- **重连进行中**：`not sent`，文字是 connecting。不会被误报为已执行。
- **正在执行时进程死掉**：`target.call` 抛 `DeadObjectException`，不是 `Rejected`，所以网关标为不确定，主机显示 UNKNOWN，不重发。
- **新进程上的第一条需要会话的命令**：`sessionRequired`，在 `error.message` 和 `reason` 后面附上作废提示（`:179-185`、`:192-201`），并照旧写明 `session open --display main|virtual`。旧引用不会落到新页面上（上一轮已核：引用编号每个进程随机起步）。
- **结论：没有误报"已执行"，也没有静默重发的路径**（源码推断，加上述 logcat）。

### 1.4 会话到期、显示偏好与后台屏任务保护

- 到期由 worker 每秒一次的定时任务触发（`WorkerService.kt:39-57`）；`Sessions.expire()` 只在 `resources.isEmpty()` 时执行（`Sessions.kt:335-343`）。会话里有界面命令、前台 shell 任务或文件传输在进行时不会到期。后台 shell 任务会提前释放占用（`ShellJob.kt:116-120`），不会让租约永不到期。
- 到期直接作废租约，并通过 `ui.clear` 销毁后台屏（下称 VD，AndroidCtl 自建的虚拟显示）。VD 带 `DESTROY_CONTENT_ON_REMOVAL`（`WorkDisplay.kt:72`），所以上面的所有任务都会被销毁，包括"外来任务"（不是本会话打开、却出现在 VD 上的任务，例如分享时跳出的另一个 App）。这是第六轮第 12 条明确的取舍，`displays.md:93-99` 写了，到期报错里也带任务记录。**不会清掉主屏上别人的资源，也不会漏掉自己的 VD。** 唯一的"清掉别人的东西"是 VD 上的外来任务，属于已裁决的取舍，我没有新证据要求推翻。
- 接管只能在同一配对内进行；别的配对或别的会话名一律 `busy`（`Sessions.kt:159-163`）。省略 `--display` 只影响新开会话选哪块屏，不越权。上一轮已核，本轮这部分没有改动。
- 本轮新增 `callerState=expired`（`Sessions.kt:384`）：优先级依次是 released、active、expired、none。我核对过，不会有一个键同时落在两种状态里却报错的情形。**核对无问题。**

### 1.5 关闭与清理

- **关闭与进行中的命令**：同一主机、同一会话时，两条请求在主机连接上串行；关闭会等前一条做完，预算不够就报 `connectionBusy`，并说明"未发出"。不同主机用同一个会话身份时，设备端 `prepareClose` 里的 `requireIdle()` 返回 `busy "Active ui must finish"`（`Sessions.kt:193`）。**不会在命令执行中途拆掉显示。**
- **关闭途中 worker 死亡**：关闭拿到 UNKNOWN；VD 随进程销毁，任务按 `DESTROY_CONTENT` 处理，即使调用者写了 `--return-tasks` 也一样。之后重发 `session close`，得到 `No current session` 加作废提示。行为如实，指引里写了 "worker exit/restart … destroy virtual directly"。
- **剪贴板借用**：粘贴进行中 worker 死亡时，原剪贴板不会恢复，回执是 UNKNOWN。指引 `input.md:62` 和 `reference.md:61-62` 要求读 `restore`、不要重复粘贴，但没有提"进程死亡时恢复不会发生"。这是残留风险，极少发生，列为可选建议（P4）。
- **临时文件和 helper**：worker 的锁文件用 `O_CLOEXEC` 打开（`c/files.c:181`），helper 继承不到，所以不会挡住新进程启动。helper 后代和临时文件在 worker 死亡后是否残留，指引写了"不证明已消失"（`connection.md:123-124`）。我没有深挖 `execute.c` 的子进程回收逻辑，那属于 shell 专题。
- **端口转发**：主机从不创建 adb forward；它由调用者自己建、自己删（`connection.md:80-86`）。主机守护进程退出与端口转发无关。

### 1.6 主机守护进程异常退出后，设备端残留什么

- 网关侧的 WebSocket 断开，由 `NetworkGateway.expire` 回收。**设备租约不随连接结束**（`displays.md:94`："Host daemon stop/device forget do not end its lease"）。所以会残留：租约（直到空闲到期，默认 30 分钟）、VD 及其上的任务、正在跑的 shell 任务、设备端文件传输记录（由 `files::expire` 回收）。在这段时间里，别的配对会拿到 `busy`。
- 守护进程重启后，同一配对加同一会话名会接回原租约，这是设计目标。主机本地的传输记录会丢失（`connection.md:105`）。
- 剪贴板借用和界面动作都在设备上完成，不依赖主机在线。
- **结论：残留都是有意设计并已写进指引的，没有发现漏写的残留**（源码推断）。

---

## 2. 竞态（源码推断，给出交错顺序）

| 编号 | 交错 | 结果 | 判断 |
| --- | --- | --- | --- |
| R1 | 断开回调置 `CONNECTING` 并排队 `bind(auto)` → Shizuku `ready()` 排队 `bind(manual)` → 主线程先执行 auto，新建尝试 → 再执行 manual，看到 5 秒内的尝试，把来源降为手动后返回 | 只绑定一次，来源是手动 | 正确（测试 `manualOrReadyEventsShareAQueuedAutomaticBind…`） |
| R2 | 停止：`destroyAndConfirm` 置 `expectedStop` → `destroy()` → 主线程上的 `onServiceDisconnected` 写 `STOPPED` → 后台线程确认死亡，再写一次 `STOPPED`，清提示 | 不重连，文字一致 | 正确 |
| R3 | 旧 worker A 的在途请求 → A 死亡 → B 自动附着 → A 的回复迟到（只可能是异常） | `connection === observed` 的判断挡住它，不会清掉或消费 B 的提示 | 正确（`:189`、`:206`） |
| R4 | 重连后，配对 P2（之前没有会话）的命令先到 → P2 得到 `sessionRequired` 并**消费**提示 → 原持有者 P1 的命令只得到 "No current session" | P1 不知道 worker 重启过 | **小缺陷 D2** |
| R5 | 重连后，P2 的 `session open` 先成功 → `recoveryNoticePending=false` → P1 的命令拿到 `busy`（P2 占着），从此看不到作废提示 | 同上 | **小缺陷 D2** |
| R6 | 两条命令同时"触发重连" | 命令不会触发重连，只有死亡回调和 Refresh/ready 会；死亡回调对同一次尝试只生效一次（`:101-102`） | 不成立 |
| R7 | 重连与清理并发 | 清理在旧进程的执行器里排队，随旧进程消失；新进程从空状态开始（`UiBackend.kt:570-582`） | 正确 |
| R8 | 到期与正在提交的 `session open` 重叠 | `prepareChange` 先在锁内调用 `expire()`，再判断是否复用（`Sessions.kt:110-130`） | 正确 |

---

## 3. 调用者看得到的提示

| 文字 | 准确吗 | 下一步唯一吗 | 问题 |
| --- | --- | --- | --- |
| `Worker is connecting; retry in a few seconds` | 是 | 是 | 无；上一轮 F1 已修 |
| `Worker exited again within 60 seconds … ask the user to use Refresh status on the device` | 是 | 是 | 偏长，但没问题 |
| `Worker disconnected; use Refresh status on the device` | 是 | **主语不清** | 调用者自己点不到；与上一行不一致（D1） |
| `Worker stopped on the device; use Refresh status` | 是 | 同上 | D1 |
| `Worker binding timed out; use Refresh status on the device` | 基本准确（附着可能仍在进行；本机实测附着只要约 0.7–1.1 秒，所以 5 秒阈值够用） | 同上 | D1 |
| `Worker attachment rejected: <异常>` | 是 | **没有下一步** | D1 |
| `Worker is unavailable; check the device App`（兜底） | 是 | 含糊 | 只在几十毫秒的窗口里出现，可以接受 |
| 作废提示 `Worker exited unexpectedly; old sessions, work displays and their tasks are invalidated`，附在 `sessionRequired` 后面 | 是 | 是（同一条里写着 `session open --display main\|virtual`） | 只送一份，见 D2 |
| `sessionRequired` 加到期描述 | 是 | 是 | 上一轮已核 |
| `connectionBusy` 加占用者和剩余预算 | 是，指引写明它只是估计 | 是 | 无 |
| 锁屏和熄屏短标记：`mainLocked=true`、`mainInteractive=false`、`displayOff="displayN"`（`ui_view.py:505-510`） | 是。`keyguardShowing` 和 `deviceLocked` 都是整机事实，所以加了 `main` 前缀；`displayState` 取的是目标显示（`UiBackend.kt:77`），所以 `displayOff` 带显示编号。前台有锁屏上可显示的界面（如闹钟）时，`isKeyguardLocked` 仍为 true，而输入本来也会被 `inputReady` 拒绝，所以这个标记不会误导。只有安全锁屏而未锁定时不加标记，与 `observations.md:94-97` 一致 | 标记不给下一步；下一步在动作的拒绝文字里 | 48 次正式尝试里**一次都没有出现**（实测：在 `efficiency/**/child.stdout` 里搜这三个词为零），只有主机单测 `test_observation_files.py:696-751`。效果未经实测，需设备 |

不会诱导无效补查：connecting 那条明确说"几秒后重试"，没有引导调用者去 status 或 snapshot。`requireCurrent` 的文字从 "take a new snapshot" 改成了 "inspect status first"（`Sessions.kt:307`），更准确。

---

## 4. 是否过度设计：自动重连、60 秒例外、手动恢复资格

**事实：**

- 48 次正式尝试：worker 意外退出 0 次，作废提示 0 次（`invalidated` 在 `efficiency/` 下零命中），自动重连 0 次。`sessionRequired` 只出现在模型读指引的文本里，没有出现在命令输出里（实测，grep `EV/efficiency`）。
- 定向验证（`r7-runtime-preparation.md:51-57`）：三次故意杀进程组成一条资格链，触发 2 次自动重连、1 次 60 秒例外、1 次 Refresh 恢复资格。logcat 时间线与叙述一致（§1.2）。
- 前几轮材料里，worker 意外死亡也只出现在故意做的生命周期测试中（`r6-lifecycle-039…044`）。

**它救回了什么**：只救回"不用请用户在手机上点一次 Refresh"。会话、VD、任务、引用、设备端任务全部作废，调用者必须重开会话、重新采样。

**如果只保留"死了就报错、提示请用户 Refresh"，调用者会损失什么**：一次人工往返。用户不在手机旁时，任务会停住，直到用户回来。这对"无人值守的 agent"是真实的损失，但发生频率在现有证据里是零。可能的来源包括低内存设备上 LMK（低内存回收）杀 shell 进程、厂商 ROM 的后台限制，这些在模拟器上测不到（需设备）。

**60 秒例外值不值**：它防的是"每次活过 5 秒、然后自己崩溃"的无限重连。这种情形也是零实测。更简单的替代规则是"自动连上的进程死了就不再自动重连，必须 Refresh"，只需改一个条件：`recover = !previous.automatic && age > 5s`，省掉一个常量和一条文字。代价是：自动连上、又健康运行了几小时的进程第二次死掉时需要人工介入。在零实测的情况下，两者分不出优劣。

**我的判断（设计意见）**：

- 保留自动重连。它已经做完，有测试，有设备实测；状态比上一版还少了一套死亡监听；而且主要的维护成本已经付过了。
- **不建议再为它加任何东西**，比如次数上限、退避、按原因区分。
- 60 秒例外可以保留，也可以换成上面那条更简单的规则；用户若想减少需要理解的规则，可以选后者。这不值得为它单开一轮。

---

## 5. 本轮增量逐块评价

### `WorkerConnection.kt`（+94 行）

| 块 | 解决什么 | 值不值 |
| --- | --- | --- |
| `readFailure` 在附着中返回 connecting（`:39-50`） | 上一轮 F1：附着窗口里让调用者去找用户 | 值，约 5 行 |
| `Attempt.automatic` 改成可写，手动绑定把它降为手动（`:54`、`:144`） | 读-32：Refresh 或 Shizuku 就绪要恢复资格 | 值，2 行 |
| 断开回调分出四种原因和文字，加 60 秒判断（`:106-125`） | 下-15 和第 17 项 | 必要；60 秒那一支的价值见 §4 |
| `recoveryNoticePending` 从附着对象移到连接对象（`:30`、`:148`、`:205-209`） | 小-29：自动重连失败、之后手动接上时提示丢失 | 修的是零实测的边角情形；一个布尔值，代价小。但它变成全设备只有一份，带来 D2 |
| `session.open`/`takeover` 成功时清提示（`:186-190`） | 上一轮 F2 前一半 | 值 |
| 空 reason 时不加 `"; "`（`:183-184`） | 小-28 | 值，1 行 |
| `destroyAndConfirm` 带上停止时的尝试，写 `STOPPED` 并清提示（`:253-293`） | 停止后的状态文字；旧的停止不覆盖新的尝试 | 值；文字是 F1 的一部分 |
| `detach` 清提示（`:300`） | 一致性 | 1 行，可以接受 |
| `requireTarget` 参数改名 | 消除与成员 `binding` 同名的遮蔽 | 中性 |

测试：`WorkerConnectionTest.kt` 增加 331 行、删除 38 行（`git diff f401deb HEAD --stat`），共 16 个测试。测试覆盖了 §2 的 R1–R3 和 §1.2 表中各行，但没有覆盖 R4 和 R5。测试与产品代码之比约 3.5:1，围着一个实测零触发的机制，这是维护负担，不是缺陷。

### `app.py`（+127 行）

| 块 | 解决什么 | 值不值 |
| --- | --- | --- |
| `session`、`daemon`、`device` 不带子命令时报 `Missing command; choose …`（`app.py:1083-1090`，并去掉 `no_args_is_help`） | 让裸命令得到一行可执行的错误，而不是整页帮助 | 值，约 8 行。本地运行：`session: not sent; Missing command; choose open, close, takeover; …`，退出码 2 |
| `notifications get -KEY` 提示加 `--`（`:1091-1100`） | 通知键可能以 `-1\|` 开头，click 会把它当成选项 | 值，针对真实格式。本地运行确认：`-1\|android\|…` 能触发提示，`-abc` 不会 |
| `fill`、`scroll-until` 缺参数时给示例（`:1101-1121`） | 第一次使用时的语法 | 可选。正式运行里没有出现过这两种缺参数（实测：`Missing option` 和 `Missing argument` 零命中），属于预防性代码，约 20 行 |
| `emit_status` 隐藏 `coordinates` 和空的 `maintenance`，去掉 `workerId`/`gateway` | 默认文本瘦身 | 值。注意：去掉 `workerId` 之后，调用者从默认文本看不出 worker 换过了，只能靠 D2 那条一次性的提示 |
| `emit_diagnostics` 和 `job_commands.show` 去掉重复的 `output.error` | 重复的错误行 | 值 |
| `render_summary` 加 `previousSession`；错误字段统一走 `output.error_fields` | 上一轮 F12；去掉重复代码 | 值 |
| `task_disposition_summary`（约 40 行） | 上一轮 F9：关闭成功时打印整段 JSON | 值。正式运行 `r7-v1-candidate-01` 的关闭输出已经变成一行摘要（`…/1795726808321545800-…/child.stdout`）。阻断、部分完成等状态照旧打印完整详情，符合指引 |
| 几处消息末尾去掉句号 | 避免 `.;` | 值，但只改了自写的消息，click 原文接上新提示后又出现了 `.;`（D3） |
| `from typer._click.exceptions import …` | 用来识别错误类型 | 依赖私有模块。`cli_options.py` 本来就在用 `typer._click.core`，不是本轮新开的口子，但升级 typer 时会一起坏 |

`transport.py`、`coordinator.py`、`file_jobs.py`、`local_call.py`、`workspace.py`：`exit` 赋值统一改用 `set_exit`，`connectionBusy` 的文字加了占用者，证书失败分成两个原因码（与 Kotlin 一致）。都是机械改动，**核对无问题。**

---

## 6. 发现清单

### D1 "请点 Refresh"类文字主语不一致，"附着被拒"没有下一步（确定缺陷，小）

- **位置**：`WorkerConnection.kt:43`、`:88`、`:334`、`:335`，对照 `:121-122`。
- **触发**：手动连上的进程存活不到 5 秒就死、停止之后、绑定超时、附着被拒。
- **影响**：调用者会把 "use Refresh status on the device" 理解成自己能做的事，而只有用户能在手机上点。"Worker attachment rejected: DeadObjectException" 不告诉调用者该做什么。码表 `reason-codes.json:150` 写了"请用户点刷新"，但码表不在调用者面前。
- **建议**：统一写成 "ask the user to use Refresh status on the device"；附着被拒的文字后面也加这一句。
- **代价**：4 处字符串，加 JVM 测试里对应的断言。
- **验证**：JVM 测试。

### D2 作废提示全设备一份，可能被别的配对或会话名拿走或清掉（确定缺陷，小；上一轮 F2 的后一半）

- **位置**：`WorkerConnection.kt:186-190`、`:205-209`。
- **触发**：见 §2 的 R4 和 R5。必须有多个配对，或同一配对用了多个会话名，并发使用同一部设备。
- **影响**：原来持有会话的调用者得不到"worker 重启过、后台屏和任务已作废"的说明。这不会造成误操作：它总会拿到 `sessionRequired` 或 `busy`。
- **可选处理**：
  - A：维持现状，在 `connection.md:117-120` 写明"提示只给第一个碰到的调用者"。几乎零代价。
  - B：新进程把作废事实交给 `Sessions`，当作"所有身份共同的到期记录"，对每个 (配对, 会话名) 各报一次。需要 App 进程把事实传给 worker，约 20 行，加一条跨进程测试。
  - C：每次 `sessionRequired` 都附提示，直到第一次成功开会话为止。约 3 行，代价是可能重复提示。
- **推荐**：A。多配对共用一部设备不是常见场景，不值得为此再加状态。

### D3 用法提示接在 click 原文后面，产生 `.;`（确定缺陷，外观）

- **位置**：`app.py:1101-1121`。
- **复现（主机端实际运行）**：`androidctl scroll-until e1 down` → `Missing option '--max-steps'.; for example, …`；`androidctl fill` → `Missing argument '[REF] TEXT'.; for example, …`。
- **建议**：拼接前去掉 `message` 末尾的句点。1 行代码，加一条主机测试。

### D4 指引里同一个字段两个名字（确定缺陷，文档）

- **位置**：`displays.md:22` 写 `callerSession=expired`，与默认文本 `app.py:181` 一致；`connection.md:101` 写的是 JSON 字段名 `callerState`。
- **影响**：调用者在 `--json` 输出里找 `callerSession` 会找不到。
- **建议**：在 `connection.md` 里两个名字都写出来，或者在 `displays.md` 里注明 JSON 字段名。

### E1 定向验证 041–060 号命令原件缺失（证据不足）

- **事实**：`r7-runtime-preparation.md:51-57` 的资格链只有叙述。`EV/runtime/api34/commands/` 里只有 `r7-g1a-012…037`、`065`、`069`、`074`、`081`；在 `S/r8` 下全盘搜索 `g1a-04*`、`g1a-05*`，零结果。
- **已用 logcat 补核**：绑定和死亡的时间、60 秒例外确实挡住了重连（§1.2）。
- **无法补核**：connecting、60 秒例外、作废提示三条回执在设备上的原文。源码与 JVM 测试支持这几段文字，但"设备上实际输出了这句话"没有原件。
- **建议**：下一轮要么补上这几份 stdout，要么在交付结论里注明只有 logcat 佐证。
- **补充**：`WorkerConnection.kt` 自 `8c4a28d` 以来没有改动（`git diff 8c4a28d HEAD`），所以这条链对 `409e853` 同样适用。

### P1 60 秒例外可以换成更简单的规则（设计取舍，交用户）

- **背景**：见 §4。
- **选项**：
  - A：维持现状。
  - B：改成"自动连上的进程死了不再自动重连"。少一个常量、一条文字、两个测试；代价是健康运行了很久的自动进程第二次死掉时要人工介入。
  - C：完全去掉自动重连，只提示请用户 Refresh。要删大约 60 行产品代码和约 10 个测试；代价是每次意外退出都要人工介入。
- **推荐**：A，零成本。B 可以作为以后精简时的候选。不推荐 C：已经付过的成本拿不回来，零实测也不等于零发生，至少低内存真机上的 LMK 还没有被测过。

### P2 停止之后的旧会话没有说明（可选）

- **位置**：`WorkerConnection.kt:284-288`。
- **现象**：用户点 Stop 时，作废提示被清掉。用户再 Refresh 之后，调用者的旧会话只得到 "No current session"。
- **判断**：停止是用户主动做的，用户知道发生了什么；调用者照提示重开会话即可。不必处理。

### P3 `fill`/`scroll-until` 缺参数示例是预防性代码（可选）

- 正式运行中零触发。留着无害，约 20 行。以后精简时可以优先考虑删掉。

### P4 粘贴途中 worker 死亡时原剪贴板不会恢复，指引没有写（可选）

- **位置**：`BorrowedPaste.kt`，`input.md:62`。
- **建议**：在 `input.md` 里加半句"worker 退出时恢复不会发生"。需设备确认。

## 7. 核对过、确实没问题的

1. 主机和设备两端都没有自动重发；重连期间的命令都是 `not sent`，进程死亡时在途的命令都是 UNKNOWN。
2. 状态机没有死状态；所有回调串行执行；旧尝试和旧 worker 迟到的回调被 `attempt === this` 和 `connection === observed` 挡住。
3. 60 秒窗口按每次附着各自计算，从附着被接受时起算，用单调时钟；手动来源不会被自动来源覆盖；被动轮询（`MainActivity.pollStatus`、`observeWorker`）不会调用 `bind`（上一轮记录和本轮设计文档都这么说，我核对了 `GatewayService.kt:198-208`）。
4. 停止或强制退出不会触发重连；停止的说明不会被后续代码清空。
5. 到期只在没有占用时发生；到期和关闭都不会清掉主屏上别人的资源；关闭会等进行中的命令结束。
6. worker 死亡时，系统收回它的 UiAutomation 登记（logcat 22177 行），锁文件不会被 helper 继承。
7. `connectionBusy` 的新文字报出的是真正的占用者，并说明命令未发出。
8. 锁屏和熄屏三个短标记的取值与命名正确（源码推断；没有实测）。
9. `app.py` 的关闭摘要在正式运行里只占一行，阻断和部分完成时照旧给出完整详情。

## 8. 没有做的事

- 没有运行 JVM 测试（这里没有 Android SDK），也没有运行主机全量测试（主评审在跑）。主机端只用 `S/r8/venv8/bin/androidctl` 实际运行了 9 条用法错误命令（§5、D3）。
- 没有深挖 shell helper 在 worker 死亡后的回收（属于 shell 专题），也没有核对 Shizuku 服务端重启时 daemon 型 worker 是否存活（需设备）。
