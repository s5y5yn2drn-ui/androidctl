# 第七轮专题 102：Android 端本轮差量逐行评审

评审对象：产品仓库 `fb75afc..f401deb` 中 `android/app/src/main` 的全部差量（31 个文件，+597/−239，含 `AndroidManifest.xml`），以及对应的 21 个 JVM 测试文件差量。行号默认指 `f401deb`。全程只读；没有设备，没有跑 Gradle。

## 摘要

1. **没有找到必须挡合入的错误。** 本轮 Kotlin 差量整体质量不错：可见性规则的所有读点都改到了，新错误码全部登记在册，跨语言契约样本都是真实代码产出的，重绑逻辑与 Shizuku 的回调语义对得上。
2. 最值得在下一轮处理的是**方向词映射的形状判断**（`UiScroll.kt:55,66-71`）。一个比宽更高、只公布"前进／后退"的横向列表（例如竖屏下整屏的横向翻页列表），`scroll down` 会被映射成"前进"，结果是横向翻页，回执却报完成。性质：很可能的错误；需要真机样本确认实际触发频率。
3. 有三处确定但影响小的逻辑缺口：
   - 自动重连的提示可能附到无关的报错上（`WorkerConnection.kt:75,177-181`）。
   - `open` 改成先确认落地、再观察以后，落地一直确认不了时，观察拿不到预算（`ActivityLaunch.kt:211-218,328-357`）。
   - `wait --text` 未命中时回报的 `complete` 没有扣除读不到的文字字段，和 `scroll-until --contains` 的口径不一致（`UiCondition.kt:97-105` 对 `ScrollContainer.kt:39`）。
4. 方向上的疑点：导航后的"加载中"判定硬编码了 Google 地图找路线时的文案（"正在查找最佳路线"／"finding the best route"，`UiPostAction.kt:284`），测试也直接断言这条文案。这属于对评测题的定向适配。另外，"加载中"在产品里现在有两套定义。
5. `WorkerConnectionTest.kt` 新增 279 行，测的是**事件顺序状态机**：重复的断开回调、过期的连接尝试、停止过程中断开、`detach` 抢在排队的重绑之前、短命进程不重绑、重连提示只出现一次。它没有测真实多线程竞态，也没有测真实的 Shizuku。但实际代码里所有连接回调和 `bind` 都串行在主线程上，跨线程的只有 `call`／停止／强制退出三条路径，而它们都在锁或 `@Volatile` 保护之下。所以这个覆盖面对当前设计基本够用。
6. 三个改动最集中的文件（`WorkerConnection.kt`、`UiCondition.kt`、`UiPostAction.kt`）通读后结构仍然清楚：没有死代码，没有互相矛盾的注释。只有个别可读性小问题（见第 13 条）。

---

## 一、发现清单（按严重程度排序）

### 1. 方向词映射：用"高大于宽"判定竖向容器，会把横向列表当成竖向

- **位置**：`UiScroll.kt:52-60`（`genericAction` 新增的 `|| verticalBounds(node)` 分支）、`UiScroll.kt:65-71`（`verticalBounds`）。
- **触发条件**，三条同时满足：
  1. 容器只公布"前进／后退"滚动动作，没有任何方向性动作（上、下、左、右、翻页）；
  2. 类名不是 `ViewPager`；
  3. `collectionInfo.columns` 不等于 1，但边界高大于宽。
- **典型例子**：竖屏下整屏展示的横向 `RecyclerView`（图片画廊、短视频或故事流常用"横向线性布局 + 吸附"实现）。按我对 androidx RecyclerView 的了解，它的无障碍节点只公布前进／后退，`collectionInfo` 为 1 行 N 列。这一点我没在本机核实（见"未能核实的事项"）。
- **后果**：调用者发 `scroll down`，实际横向翻到下一页，回执报"已完成"，调用者以为页面往下滚了。在 `scroll-until` 里则表现为沿错误方向推进，直到找到或步数耗尽。
- **是否已在真实任务里发生**：未知，本轮的运行记录我没有逐条核对这一点。
- **依据**：源码推断。新测试 `UiScrollTest.kt:149-158` 只覆盖"高的自定义类、无 collectionInfo"和"宽的"两种情形，没有覆盖"高但 collectionInfo 表明横向"的情形。
- **背景**：这条来自第六轮附录一"只公布前后方向的竖向容器接受 down、up"的逐条处理。用户授权的是"竖向容器"，形状只是实施方选的近似判据。
- **最小修正**：`collectionInfo` 明确是 1 行多列时不走形状判据；也就是只在 `collectionInfo` 缺失或行数大于 1 时才用 `verticalBounds`。改一行，加一条测试。
- **分类**：很可能的错误（平台假设）。去向：**进下一轮**。如果主评审在运行记录里找到它实际误滚的样本，就升为挡合入。

### 2. 自动重连提示可能附到之后无关的"需要会话"报错上

- **位置**：`WorkerConnection.kt:75`（`Attached(..., recovering)` 把 `recoveryPending` 设为真）、`:149-175`（`call` 只在遇到 `sessionRequired` 时附加并消费提示）、`:177-181`。
- **触发路径**：常驻进程意外退出，并且已连接超过 5 秒（`MIN_CONNECTED_MS`），于是自动重绑成功。调用者接下来先跑 `session open` 或 `status`，这两条不会返回 `sessionRequired`，提示没有被消费。之后过了很久，会话因空闲到期，下一条命令得到的报错是："Session expired after its idle timeout…; Worker exited unexpectedly and was automatically reconnected; old sessions, work displays and their tasks are invalidated"。后半句是早已过时的旧事件，会把到期误导成常驻进程又崩了一次。
- **另外两处小缺口**：
  - 自动重绑失败以后，用户在设备上点"刷新状态"，走的是 `bind(recovering = false)`（`:116,133`），提示就丢了。
  - 主机端有两个配对各自持有会话时，只有先撞上的那一个能看到提示。
- **后果**：只是文字误导，不会造成误操作。
- **最小修正**：在 `Sessions.publish` 成功（建立新会话）时清掉 `recoveryPending`；或者把提示绑定到重连前已存在的会话键上。改几行。
- **分类**：确定的错误，影响低。去向：**进下一轮**。

### 3. `open`：落地一直确认不了时，交付用的观察被饿死

- **位置**：`ActivityLaunch.kt:211-218`（新顺序：先 `confirmLanding`，再 `observeLanding`）、`:328-357`（`confirmLanding` 在 `do … while (run.budget.remaining() > 0)` 里每轮 `pause()`）。
- **触发条件**：同时满足以下两条。
  1. 落地状态为 `pending`（正常启动成功）或目标在后台虚拟屏上。
  2. 目标包和 UID 始终不出现在目标屏上，而且也不在别的屏上（landing 状态一直是 `unknown`）。典型情形：中转 Activity 立刻转到另一个包，例如深链交给浏览器；或者冷启动超过整个请求预算。
- **后果**：第一次 `confirmLanding` 把整个请求预算轮询完，随后 `observe(landed)` 在零预算下运行。`UiPostAction.settle` 的循环一次也不进，最后只给出 `snapshotError`。旧顺序是先观察（约 1.5 秒的稳定窗口），再轮询确认。所以在这种失败情形下，旧版至少带一份观察，新版没有，调用者要多跑一次 `snapshot`。确认成功的正常路径不受影响，而且反而能拿到落地后的界面树，这正是本轮改动的目的。
- **依据**：源码推断。实施方的组内评审（`ST/r6-launch-paste-review.md`）讨论过"观察耗尽预算会撤销已确认的完成"，并已修好；但没有讨论"落地轮询耗尽预算导致没有观察"这一反方向。
- **最小修正**：第一次 `confirmLanding` 的轮询截止设为 `minOf(deadline, 启动时刻 + 稳定窗口)`，超时就进入观察，观察后的第二次确认再用剩余预算。维护代价小。
- **分类**：很可能的回归（只在失败分支）。去向：**进下一轮**。

### 4. "加载中"的导航判定硬编码了评测题文案，并与已有定义分叉

- **位置**：`UiPostAction.kt:255-267`（`navigationLoading`）、`:284`（`loadingText`）。测试在 `UiOutcomeTest.kt` 新增的 `navigationReportsLoading…`，循环里直接断言 `"正在查找最佳路线"`。
- **现象**：
  - 第六轮评审的 B2 要求"界面树里有明显加载文字时标成可能还在变化"。实施方的做法是精确匹配五个字符串，其中三个是 Google 地图找路线时的文案（中英文），而路线题正是固定评测任务之一。这属于对评测题的定向适配，换到别的 App 上不会泛化，也会让路线题的成本对比带上偏差。
  - "加载中"在产品里现在有两套定义：
    - `UiSemantics.loading`（`UiSemantics.kt:397-402`，只认不确定进度的 `ProgressBar`）用于 `wait --idle`、`SettledSamples.accept` 的"完整"判定，以及 `classification`。
    - 新的文字判定只用于导航类命令的回报。
    - 后果是：同一屏显示"Loading…"文字时，`click` 回报 `settle=loading`，而 `wait --idle` 可能已经判为稳定。
  - 次要的一点：精确匹配"loading"时，列表底部常驻的"Loading…"页脚也会让每次点击都报 loading。
- **是否已在真实任务里发生**：没有核对路线题记录里这条判定的命中情况，请主评审结合成本页确认。
- **最小修正**：二选一。
  - 去掉和具体 App 绑定的文案，只保留通用词（loading、加载中、正在加载）。
  - 或者把文字判定并入 `UiSemantics.loading`，让 `idle` 和动作回报用同一规则。
- **分类**：设计取舍兼方向疑点。去向：**进下一轮**。是否把"去掉地图专用文案"提前到合入前，由用户决定：改动只有一行，但会影响本轮路线题成本数字的可比性。

### 5. `wait --text` 未命中时的 `complete` 没有扣除读不到的字段

- **位置**：`UiCondition.kt:97-105`。未命中时的兜底谓词写 `"complete" to completeTree(tree)`。但 `text()`（`:208-218`）丢掉了 `matchText` 返回的 `unknown` 信息。
- **对比**：`ScrollContainer.inspectText`（`ScrollContainer.kt:39`）写的是 `complete && unknown == 0`。
- **触发条件**：某个可见节点的 text、description 或 stateDescription 读取失败（出现在 `unavailable` 里），而树本身完整。
- **后果**：失败回执写着 `complete=true`，调用者会把它理解成"确定不存在"，其实有字段没读到。
- **最小修正**：`text()` 统计 unknown，兜底谓词带上 `unknown`，并让 `complete` 扣掉它。
- **分类**：确定的不一致（新代码与旧规则不一致），影响低。去向：**进下一轮**。

### 6. 坐标触摸的"被遮挡"判定可能误拒

- **位置**：`AppInput.kt:157-175`。用无障碍窗口列表中层级更高的窗口边界判断目标中心点是否被盖住。
- **疑点**：无障碍窗口列表里是否会出现不可触摸、却覆盖较大区域的窗口（例如某些无障碍服务的覆盖层，或系统装饰类窗口）。如果有，触摸会被误拒为 `covered`。反过来，它确实能拦住输入法、悬浮窗和通知横幅，与第六轮 B6 的要求一致。
- **需要的证据**：在开着常见无障碍服务（例如无障碍菜单的悬浮按钮）的设备上，看 `--boxes` 输出的窗口列表。
- **分类**：值得问实施方的疑点。去向：进下一轮观察，不必现在改。

### 7. 任意一个条件（any-of）未命中时，`matches` 和 `refs` 混入了"等待消失"条件里仍存在的节点

- **位置**：`UiCondition.kt:457-472`。测试 `UiConditionTest.kt` 新增的 `unmatchedAlternatives…` 明确断言了这一语义：两个"等待消失"条件仍在的节点被汇总成 `matches=2`。
- **疑点**：主机端把谓词渲染在 `match=` 标签下（`ui_output.py:196-203`）。调用者读到 `match={matches:2, refs:[…]}` 时，未必明白这是"阻止消失条件成立的节点"，而不是"命中"。单个"等待消失"条件本来就这样计数，这一轮只是把它带进了 any-of。
- **分类**：疑点或可读性问题。去向：进下一轮，或者不必处理，由用户决定。

### 8. 到期时记下的任务以 JSON 原文嵌在报错消息里

- **位置**：`SessionExpiry.kt:22-36`。
- **现象**：
  - 任务记录按类别分组，每类最多 8 条、8 KB（`ActivityTasks.kt:145-146`），然后整体 `toString()` 拼进 `sessionRequired` 的消息里，最坏情况约 32 KB。
  - 没有任务时写成 `Task record: {}`，调用者不容易读懂。
  - 另外，同一条消息里已经有"run session open --display main|virtual explicitly"，又追加"To continue on the work display, use --display virtual"，有些重复。不过这是第 25 条裁决要求的写法。
- **分类**：可读性或设计取舍。去向：进下一轮。可以考虑把记录放进结构化字段，在消息里只写摘要。

### 9. 导航命令下"只有几何变化"会把已稳定的结果降为"可能还在变化"

- **位置**：`UiPostAction.kt:251,269-281`。
- **疑点**：`key`、`ime` 等也在导航命令集合里（`:158`）。比如一次按键只让页面平移了几个像素，可见性标记没有翻转（可见性现在按窗口裁剪，大幅滚动通常会翻转，所以只有小幅平移会走到这里）。这时即使连续两次采样已经稳定，也会报 `unsettled`。这符合 B2 的字面要求，但对按键滚动类操作会多给一句"页面可能还在变化"。
- **分类**：疑点，影响很小。去向：不必处理，或者进下一轮观察。

### 10. 自动恢复期间 `workerUnavailable` 的措辞

- **位置**：`WorkerConnection.kt:103`（先写入"Worker disconnected; use Refresh status on the device"），以及随后排队的 `bind` 在 `:137` 把它清空。
- **现象**：在自动重绑排队或进行中的这段时间，调用者先拿到"请在设备上点刷新状态"，接着又拿到"Worker is unavailable; check the device App"。两句都没说"正在自动重连，请稍后重试"，调用者可能去打扰用户。
- **分类**：风格或可读性。去向：进下一轮。

### 11. 未确认的停止会留下 `expectedStop = true`

- **位置**：`WorkerConnection.kt:226`。
- **现象**：`destroyAndConfirm` 先把 `expectedStop` 置真，再去发 `destroy`。如果确认失败（抛出 `workerExitUnconfirmed`），这个标记保留。之后这个进程如果真的意外退出，既不会自动重绑，也不会写 `lastFailure`。这时调用者只能看到兜底文案。
- **分类**：疑点，影响很小。这种情况本来就要求用户先查看状态。去向：不必处理。

### 12. `BorrowedPaste.save()` 在执行中途也会重写 reason

- **位置**：`BorrowedPaste.kt:334-359`。
- **现象**：状态计算从 `execute` 末尾挪进了 `save()`，而 `save()` 在整个过程中被调用很多次。结果是，中途记住的回执原因会被改写成"Borrowed paste did not complete: effect=unobservable, cleanup=…"，覆盖原来的"Borrowed paste has not dispatched"或"Borrowed paste pending"。
- **安全性核实**：我逐个核对了中途调用 `save()` 的时点。`restore.state` 在写入借用剪贴板之前（`:150`）已经设为 `pending`，所以从写剪贴板到恢复结束，中途结果都是 `UNKNOWN`，不会在已派发粘贴之后误报"未发送"。
- **分类**：风格（中途措辞不够准确），不是错误。去向：不必处理。

### 13. 可读性小问题（不凑数，只列确实影响阅读的）

- `WorkerConnection.kt:270-271`：参数名 `binding: Attached?` 遮蔽了同名字段 `binding: WorkerBinding`，函数体里只好写 `this.binding.requireAvailable()`。改个参数名即可。
- `WorkerConnection.kt:160`：结果形态的报错原本没有 `reason` 时，会拼出以"; "开头的原因。
- 退出码 3 在三处各自定义：`BorrowedPaste.kt:402` 的 `RESTORE_UNKNOWN_EXIT`、`AutomationCleanup.kt:90` 的 `CLEANUP_UNCONFIRMED_EXIT`，以及主机端的 `UNCERTAIN_EXIT`。

---

## 二、对 JVM 测试的评价

- `WorkerConnectionTest.kt`（+279 行）
  - **做法**：用假的 `WorkerBinding`（手动排空的执行器队列）、代理的 `IBinder`／`IExecutor`、可注入的时钟，在单线程上按顺序驱动回调。
  - **测到的**：
    - 同一次断开回调重复到达只重绑一次，重连后不重放请求；
    - 存活不到 5 秒的进程，或者绑定前检查失败时，不会循环重绑；
    - 显式停止或强制退出不重绑，但新进程的意外退出仍会重绑；
    - 停止被拒绝时不打"预期退出"标记；
    - `detach` 能阻止已经排队的重绑；
    - 重连提示在两种报错形态下各出现一次。
  - **没测到的**：
    - 真实的多线程交错，例如 `call` 与断开回调并发、新连接尝试的 `onServiceConnected` 进行到一半时旧连接断开；
    - Shizuku 自身的行为（假绑定从不重发回调）；
    - `unbind(previous)` 有没有被调用（假实现是空操作，没有断言）。
  - **判断**：Shizuku 的 `ShizukuServiceConnection.connected`／`died` 都是 post 到主线程 Handler 的（见下文"已核实无问题"），`bind` 也在主线程执行器上，所以真正的竞态面很小，这组测试测的就是实际存在的那部分风险。没有发现用模拟对象把被测逻辑本身模拟掉的情形。
- `UiCleanupTest.kt`：覆盖了"已被其它工具占用"转成 `busy`，以及同步失败时只清理回调、不做注销。还覆盖了一个负例：`SecurityException` 的消息里即使含有 "already registered"，也不会被误判。超时仍保留"待注销"。断言精确。
- `SessionExpiryTest.kt`：用真实单线程执行器验证任务记录在会话锁外执行、记录完成前清理不算完成、记录前后报错文案不同。是行为测试，不是测实现细节。
- `UiOutcomeTest.kt` 的导航判定测试：验证了"只有边界变化时降级"和"标题变了恢复为已变化"。但加载文字的断言把地图专用文案固化成了测试（见第 4 条）。
- `UiScrollTest.kt`：缺"高但 collectionInfo 表明横向"的反例（见第 1 条）。
- `UiScrollUntilTest.kt`：旧的"歧义即失败"用例被改写成"多个可见命中即成功"，断言 `refs` 长度为 2，与新语义一致。其中 `absorbed` 用例的期望由 0 改为 1，即被并入父节点的子节点现在也计为命中。这与 `wait` 的选择器条件（`UiCondition.selected` 用 `matches.matches`）一致，但打印出的引用可能是不单独显示的子节点。建议主评审在主机输出上确认这些引用是否都能直接操作。
- `WireContractTest.kt`：新增的"名字变化"样本通过真实的 `UiRefRepair.perform` 产出，只把计时字段归零。`open-pending` 样本改为在落地读取时取早期回执，没有为了迎合新顺序手改期望。

---

## 三、已核实无问题的文件和函数（覆盖面）

**连接与重绑**

- `WorkerBinding.kt` 全文：只是把原来的 Shizuku 调用原样搬进接口。监听器的登记（`start`）和注销（`stop`）成对，`requireAvailable` 的两种模式与旧代码一致。
- `WorkerConnection.kt`：
  - `onServiceDisconnected` 的去重：`!current()` 或"不在连接中且没有连接"时直接返回。
  - 恢复门槛：`expectedStop` 和连接时长 `MIN_CONNECTED_MS`。
  - `bind` 在主线程执行器上排队。我从 Shizuku-API 上游下载了 `ShizukuServiceConnection.java`、`Shizuku.java` 和 `ShizukuServiceConnections.java`（放在 `S/agents11/102/shizuku/`；注意取的是上游最新版，项目锁的是 13.1.5）。`died()` 把回调 post 到主线程，回调执行完后 `connections.clear()` 并从缓存里移除。所以排在其后的 `bind` 拿到的是一个新的连接对象，与 `:107` 的注释吻合。
  - 删掉自己的 `DeathRecipient` 以后，死亡检测完全依赖 Shizuku 在 binder 死亡时回调 `onServiceDisconnected`。上游代码确实如此。
  - `stopExecutor` 在断开回调先到或后到两种顺序下都不会重绑。
  - `forceExecutor` 不清连接，由随后的断开回调按 `expectedStop` 静默处理。
  - `expectedStop` 和 `recoveryPending` 的读写都在同一把锁内。
- `GatewayService.kt`：通知正文去掉资源计数；资源字符串仍被 `ManagementScreen.kt:254` 使用，不是无用资源。`GatewayClient.kt`：`messageLimit` 改为 `jsonLimit`，并且已登记。
- `AndroidManifest.xml` 的 `singleTask`：`MainActivity` 不依赖 `onNewIntent`，通知的 PendingIntent 也不带数据，改动安全。

**会话与后台虚拟屏**

- `Sessions.kt`：
  - `prepareChange` 拆出 `checkTargetChange` 后，`requireCleanUi` 的条件与旧版等价。
  - 不写显示时的语义逐条符合第 25 条：新开默认主屏；已有会话跟随原显示，且不跑预检（`SessionsTest` 有断言）；`--replace` 和 `takeover` 必须写显示（`TargetManagement.kt:73-80` 加主机端 `app.py:838-844,887-893`）；后台虚拟屏上到期的记录只在新开时附提示，`publish` 成功后清掉。
  - `idleRemainingSeconds` 向上取整，有资源占用时返回完整空闲时长，与 `status` 的展示一致。
- `SessionExpiry.kt`、`UiBackend.clear`：任务记录在 UI 执行器上、清理之前、会话锁之外读取；字段 `@Volatile`；`held.expiry` 在入队前写入，有 happens-before 保证。
- `TargetManagement.kt`、`DisplayTarget.kt`、`DisplayTasks.kt`（只改了可见性）、`WorkDisplay.kt`（只改了错误码和措辞）。

**可见性和观察**

- 可见性规则的所有读点：`NodeFacts.read` 的全部调用者（`TreeCollector.kt:191,280`、`FreshNodes.kt:26,42,75`）都传了所属窗口的边界，代码里已经不再有直接读取 `isVisibleToUser` 的地方。`UiActions` 读取输入焦点时改为只读地址，`sourceId` 的编码（`HiddenApi.sourceNodeId…toString()`）与 `NodeFacts.kt:176-180` 完全一致。
- `TreeCollector`：`Root.bounds` 为空的唯一情形是窗口不在当前目录里，而这种情况 `readSelectedRoot` 在 `enqueue` 之前就已经返回，`requireNotNull` 不会抛出。
- `FreshNodes`：`readTarget` 是私有函数，只经 `read` 调用，窗口边界总是先赋值后使用。
- `ObservedChange.directory` 每次都构造副本，`onlyGeometryChanged` 删除 `bounds` 不会改到缓存的观察。
- `UiSnapshot.contentSignature(includeBounds)`、`ScrollContainer.predicate` 和 `ScrollProgress.checked`：`matches` 已经只计可见节点（`UiSelector.matches` 首行就排除不可见节点），`rootJson` 仍有调用者。初始采样不完整时继续观察，到截止时间报 `observationIncomplete` 和"未发送"，与附录一的处理一致。
- `UiCondition`：
  - 改名后的 `noSnapshot` 在抛出处（`UiDelivery.kt:98,100`）和捕获处（`UiCondition.kt:356`）一致。
  - `stopsOnRepeatedLimits` 只对"等待消失"和 `idle` 生效，与第 17 条一致。
  - 连字符规范化按 Unicode 的 Pd 类处理，测试覆盖了补充平面字符。

**动作与回执**

- `UiRefRepair.nameChange`：身份包含 `sourceId`、窗口和行位置，只有"同一个平台节点、同一位置、名字不同"才会命中，属于偏保守的拒绝。`run.measure` 是累加计时，两次 `preWaitMs` 不会互相覆盖。
- `AutomationCleanup.register`：我下载了 AOSP 14、15、16 的 `UiAutomationManager.java` 和 `UiAutomationConnection.java`（放在 `S/agents11/102/aosp/`，标签分别是 `android-14.0.0_r1`、`android-15.0.0_r1`、`android-16.0.0_r1`）。
  - 三个版本的占用报错都是 `IllegalStateException("UiAutomationService …already registered!")`，按消息里包含 "already registered" 识别是成立的。
  - 本地的 "Already connected." 不会被误匹配。
  - 本仓库 `S/agents9/66/aosp16/UiAutomation.java` 的 `connectWithTimeout:363-419` 表明，注册成功之后唯一的异常出口是 `TimeoutException`。所以"非超时就不需要注销"的判断成立。
- `BorrowedPaste`：第 22 条的"效果一致就主结果报完成，恢复失败或未知用退出码 1、3 体现"已经落实。中途回执不会在派发之后误报"未发送"（见第 12 条）。
- `AppInput` 的可见性提示、`UiAdvice.visibilityAdvice`、`VisualInput`（`userScope`）、`UiActions` 把 `routeUnavailable` 统一用于 `locate`、`UiDelivery`、`UiReceipt`（新增的待截断文本只在超出预算时才处理）。
- `ActivityLaunch`：
  - `elsewhere` 改报 UNKNOWN；
  - 读取失败时把已经得到的 DONE 降为 UNKNOWN；
  - 检测到外来任务迁入时以 `unexpectedTaskMove` 优先；
  - `inspectLandingTasks` 只在请求已结束且结果已是 DONE 时跳过读取。
  这些分支都与组内评审修正后的描述一致（预算方向的问题见第 3 条）。

**登记与契约**

- 错误码登记：我用脚本提取了主源码里全部字面量形式的 `Rejected("…")` 错误码，与 `contract/reason-codes.json` 比对，没有遗漏（提取结果在 `S/agents11/102/kotlin-codes.txt`）。以变量传入的错误码（例如 `limits.first()`）不在这次比对范围内。
- 契约字段：`idleRemainingSeconds`（主机 `app.py:174,225`）、`nameChange`、扫描查找谓词里的 `refs`、any-of 里的 `refs` 和 `matches`、`exit` 字段，主机端都认识。扫描查找谓词去掉的 `ref`、`root`、`windowId` 和 `candidates`，主机端没有读取。

---

## 四、去向汇总

| 条目 | 性质 | 去向 |
| --- | --- | --- |
| 1 方向词形状判据误把横向列表当竖向 | 很可能的错误 | 进下一轮（有实际误滚样本则挡合入） |
| 2 重连提示附到无关报错 | 确定的错误，影响低 | 进下一轮 |
| 3 `open` 落地轮询饿死观察 | 很可能的回归，只在失败分支 | 进下一轮 |
| 4 加载文案硬编码评测题文案，"加载中"两套定义 | 设计取舍兼方向疑点 | 进下一轮（是否提前删去地图文案由用户定） |
| 5 `wait --text` 的 `complete` 口径 | 确定的不一致，影响低 | 进下一轮 |
| 6 遮挡判定可能误拒 | 疑点 | 进下一轮观察 |
| 7 any-of 的 `matches` 语义 | 疑点 | 进下一轮或不必处理 |
| 8 任务记录以 JSON 嵌在消息里 | 可读性 | 进下一轮 |
| 9 几何变化降级 | 疑点 | 不必处理 |
| 10 恢复期间的报错措辞 | 风格 | 进下一轮 |
| 11 未确认停止残留标记 | 疑点 | 不必处理 |
| 12 借用粘贴中途改写原因 | 风格 | 不必处理 |
| 13 参数遮蔽等 | 风格 | 顺手改 |

没有条目达到"挡合入"的门槛。

## 五、未能核实的事项

- 第 1 条的实际触发面：目标 App 里横向列表是否只公布前进／后退。这取决于各 App 使用的 RecyclerView 或 Compose 版本，需要真机上的 `--boxes`、动作列表样本。
- 第 6 条：无障碍窗口列表是否会出现不可触摸的覆盖层。需要真机样本。
- Shizuku：我读的是上游最新源码，项目锁定的 13.1.5 版本的 `ShizukuServiceConnection` 是否完全一致没有核对。
- 可见性按窗口裁剪在 Chrome 长网页、后台虚拟屏上的实际效果：属于运行证据，归其他专题。本专题只核了源码层面的一致性。
- 第 4 条的加载文字判定在路线题记录里的命中次数，以及它对成本数字的影响：没有统计。
- 主机端如何渲染扫描查找的多个 `refs`，这些引用能否直接操作：属于主机专题，本专题没有逐行看。
