# 第四轮专题评审：通知的读取、写操作、监听器生命周期与第一道待决题（`4235af9`）

评审范围：设备端 `NotificationAccess.kt`、`NotificationOperations.kt`、`NotificationActions.kt`、`NotificationSelection.kt`、`NotificationReplyInput.kt`、`NotificationProjection.kt`、`ActivityTasks.kt`、`UiBackend.kt` 中的通知接线、`Sessions.kt` 的会话结束接点、`HiddenApi.kt` 的通知部分及对应 JVM 测试；主机端 `ui_commands.py`、`ui_client.py` 的通知部分与 `test_notifications.py`；指引 `skills/androidctl/notifications.md` 与 `SKILL.md` 相关段落；实施方运行原件（`ST/15-notifications/`）。

文中"实施方"指在隔离分支上写代码的另一个 agent；"后台屏"指会话目标为虚拟显示屏（`session open --display virtual`）；"主屏"指默认显示屏；"PI"指 PendingIntent；"creator"指 PI 的创建者包和 UID；"选择串"（selection）是 `notifications get` 返回的一段 base64 文本，写操作必须原样传回；"lease"是设备端一次会话的租约。下文 `PKG`、`ST`、`R` 的含义：`PKG` 为实施方材料包根目录，`ST = PKG/ultragoal/2026-11-11-next-improve/stages/41-route-rebuild`，`R = ST/15-notifications/runtime/api34-notifications`。所有设备事实只来自一台 Android Studio API 34 模拟器（实施方运行，我没有设备）。

---

## 1. 结论摘要

**整体状态**：通知这一块的代码现在是干净、可读、行为测试覆盖较好的形态；读、回复、按钮、清除四类操作在主屏会话上的设计基本正确，监听器生命周期按第三轮的决定实现且与源码一致。真正的问题集中在一处：**后台屏会话里打开会启动界面的通知**。这一处的暂行规则（按 creator 做事前拒绝、按 creator 可见与否决定完成）既挡不住实测中最严重的伤害，又制造假的 UNKNOWN。

最重要的发现：

1. **实测伤害比待决题稿写的更重**【原件】：Feeder 那次运行前，Chrome 任务 #266 是**主屏上正在显示的前台任务**（`R/commands/formal/073-…/stdout.bin` 第 3 行 `visible=true`，位于 Display #0 顶部）；发送通知后它被整体搬到后台屏 #4（075），会话关闭后它的 Activity 进入销毁（078 `t266 f}`），最后只剩最近任务列表里的记录（089）。也就是说，后台屏会话把用户眼前的 App 拿走了，关会话时又把它的界面销毁了。四个选项（A/B/C/D）**没有一个能阻止这件事**，实施方稿件写明了这一点，但把它放在"边界"里，而不是作为选项比较的核心。
2. **漏掉了最简单的候选**：后台屏会话里不支持会启动界面的通知写操作（open、Activity 类型的按钮和回复），主屏不变。它是唯一能真正排除这类搬屏的做法，还能删掉通知专用的事前任务检查、落点确认、目标屏选项这一整段。我推荐这个做法（下文称 E）；如果用户确实要在后台屏打开通知，次选是"D 加上搬屏检测"（下文称 F），而不是单纯的 D。
3. **第三轮评审"按 creator 可见与否决定完成"这条建议现在看是错的**：它在代理跳转场景（Feeder 先开自己的中转页，再开 Chrome）里必然报 UNKNOWN 并停掉 batch，而它比平台返回码多提供的信息很少。第三轮自己预言过这个缺口，但仍把它当成完成条件，这一条应撤回。
4. **写操作被拒时返回的"新选择串"可能让调用者跳过阅读**【源码】：因正文变化而拒绝回复时，回执里带一个立刻可用的新选择串，却不带变化后的正文（`NotificationActions.kt:141-153` 的事实不含 text/bigText）。LLM 调用者很可能拿新串直接重发，护栏的意义（"先读到新消息再回复"）就落空了。
5. **监听器生命周期**：实现与第三轮 §2.4 一致，结束接点、故障分支、"未知监听器"修复都有源码和原件支撑。但"每个用过通知的 lease 登记一次"仍会在系统进程里每次留下一个死亡通知引用，直到 worker 退出（AOSP 16 `ManagedServices` 注销时不解除死亡通知）；文档没写。普通用量下无害。
6. **价值证据薄**：八次真实任务效率对照（GitHub、Maps 两类）里没有一次用到通知命令【原件，rollout 检索】；全部通知证据来自专门构造的 Termux、短信、Feeder 场景。通知整块新增约 1,700 行 Kotlin 主代码、1,470 行 Kotlin 测试、1,400 词指引；基线 `7d5be70` 只有"展开通知栏"这一个全局动作。

**对第一道待决题的看法**：A/B/C/D 都是真实候选，事实核对基本成立；但推荐 D 的依据只看了"批处理被错误停下"，没把"用户前台任务被搬走并在会话结束时销毁"当作主要代价。推荐 E；若用户要保留后台屏打开通知，选 F。

---

## 2. 确定缺陷

### 缺陷 1（中高）：后台屏 Activity 通知会搬走并最终销毁另一个 App 的主屏任务，结果只报"未确认落点"

- **触发条件**：后台屏会话里 `notifications open`（或 Activity 类型的 `action`），通知 PI 的创建者没有别屏任务，但它实际打开的是另一个 App（代理中转后打开浏览器、系统通知打开设置等），且那个 App 在主屏有任务。
- **位置**：事前检查只看 creator 包（`NotificationActions.kt:520-542`，调用 `ActivityTasks.kt:79-93` 的 `virtualActivityPreflight`）；发送时带目标屏编号（`NotificationActions.kt:206-235`）；后台屏移除时销毁内容（`WorkDisplay.kt:72` 的 `DESTROY_CONTENT_ON_REMOVAL`）。
- **实际影响**【原件】：074 事前检查 `preflight.state=clear, liveCount=4`，此时 Chrome #266 就在这 4 个任务里、位于主屏且可见；发送返回 `code=0`；观察结束时后台屏可见的是 Chrome #266，于是 `landing.state=unknown`，UNKNOWN、退出码 3、batch 第二步未执行。075 确认 #266 已在 Display #4；078 中它的 Activity 处于 finishing；089 中它只剩最近任务记录（`Recent #3`）。数据侧：Feeder 数据库第 31 行 `unread=1, notified=1, read_time=1795155620653`，其余 14 行 `notified=0`（075 原件），与实施方描述一致。
- **为什么算缺陷而不只是"已知边界"**：后台屏会话存在的意义就是不打扰用户；这里用户眼前的 App 被拿走，会话关闭后它的界面被销毁。现在的结果把它报成"落点未确认"，回执里虽然列出了后台屏上可见的 Chrome，但没有说"这个任务原来在主屏"。如果改成 D，这个回执会变成 done，伤害更隐蔽。
- **推断的普遍性**【推断，未实测】：系统发的通知（创建者是 `android` 或 SystemUI）打开设置等界面时，事前检查查的是 `android` 包，必然通过，接收方的主屏任务同样会被搬。所以"创建者不等于接收方"不是少见情形。
- **最小修正方向**：见 §5，E（后台屏拒绝）或 F（保留并检测搬屏）。F 的实现只需在事前检查时记下所有任务的"任务号到显示屏"对应，观察后再读一次，任何原来在别的屏、现在在目标屏的任务都列为 `movedTasks`。074 的数据里这两次读取已经都有了。
- **确定程度**：现象有原件（1 次运行，1 台 API 34 模拟器，1 个代理 App）；成因有源码；系统通知场景是推断。

### 缺陷 2（中）：拒绝写操作时返回可直接使用的新选择串，却不返回变化的内容

- **触发条件**：调用者 `get` 之后、`reply`（或消息类通知的其它写操作）之前，正文或标题变了（聊天 App 来了新消息）。
- **位置**：拒绝前把"当前选择串"写进回执（`NotificationActions.kt:255-258` 先 `rememberChoice(notification, fresh)`，再在 `:298-316` 比对失败抛出）；回执的通知事实只有 key、postTime、包、UID、标题、类别、分组（`NotificationActions.kt:141-153`），没有 text/bigText。指引写的是"拒绝时返回新选择串和当前事实，先审阅再决定"（`notifications.md` "When complete current facts are available…"一段）。
- **实际影响**：对正文变化而言，回执里没有能审阅的东西；LLM 最自然的反应是拿新串立刻重发。护栏要防的正是"没看到新消息就回复"，这样就失效了。测试 `NotificationActionsTest.kt:193-202`、`:220-246` 断言了"拒绝时返回非空的新选择串"，即把这个行为固定了下来。
- **最小修正**：二选一。因内容变化（标题、正文、消息分类）被拒时不返回选择串，要求重新 `get`；或者在回执里同时给出新的 title/text/bigText。前者更省事，也更符合"先读再写"。
- **确定程度**：源码确定；LLM 是否会盲目重发是推断。

### 缺陷 3（低）：后台屏会话里，Activity 类型的回复会在主屏启动界面

- **位置**：目标屏选择只对 open/action 生效（`NotificationOperations.kt:14-21`）；回复的发送选项因此不带目标屏（`NotificationActions.kt:192-197`），事前也不检查任务。指引承认"reply… keep their existing admission… without virtual display options"。
- **影响**：多数回复 PI 是广播或服务类型，不受影响；少数 App 用 Activity 接收回复时，后台屏会话会在用户的主屏上弹出界面。
- **修正**：E 方案下顺带解决（后台屏拒绝所有 Activity 类型的通知发送）；否则至少在后台屏拒绝 Activity 类型的回复。
- **确定程度**：源码确定；未实测（没有这种样本）。

### 缺陷 4（低）：`SKILL.md` 仍要求调用者保存 key/postTime

- **位置**：`skills/androidctl/SKILL.md:192-195`："preserve the complete key/postTime and required action/creator selection"。
- **影响**：与 `notifications.md`（postTime 不是写条件，只传一个选择串）和 `9f80d5f` 的设计矛盾，LLM 读入口文档时会被误导去保存多余字段。
- **修正**：改成"写操作前重新 `get`，原样传回 `selection`"。

### 缺陷 5（低，平台差异）：Android 16 上"提升的常驻通知"清除会等满超时后报 UNKNOWN

- **位置**：`NotificationActions.kt:318-325` 只在"常驻且未提升"时事前拒绝，注释称"在 AOSP 14–17 上充分，17 允许清除提升的常驻通知"。AOSP 16 的 `cancelNotificationFromListenerLocked` 只排除 `FLAG_ONGOING_EVENT`（`agents5/23/up/NotificationManagerService.android-16.0.0_r1.java:5833-5839`），而 16 已有 `FLAG_PROMOTED_ONGOING`（同文件 :4618 等）。
- **影响**：16 上对提升的常驻通知调用清除，平台静默不执行，产品循环复读到截止时间（`:496-511`，由预算检查终止），报 UNKNOWN、退出 3。结果是诚实的，只是浪费整个超时。
- **修正**：可选；在指引里加一句"Android 16 上提升的常驻通知通常不能清除"即可，不必加版本表。
- **确定程度**：源码（AOSP 16 r1）推断，未实测。

### 未发现缺陷的检查项（证据支持实施方）

- **对错通知执行写操作**：每次写都按 key 重新读当前通知，比对来源包/UID、标题、消息分类；消息类和所有回复再比对 text+bigText；按钮比对原序号、标题、PI creator（`NotificationSelection.kt:47-57`，`NotificationActions.kt:298-316`）。两个不同的选择串要撞成同一串，必须 key 相同（key 内含用户、包、id、tag、UID），且标题指纹相同。同一 key 下 App 重用通知、标题不变、非消息类、正文不同时会被视为同一对象——指引如实写了这一点。我认为这对非消息类（进度、下载、前台服务）是合理的。
- **回复发给错的会话**：会话由 key 和标题（MessagingStyle 会把会话名或发送者写进标题）锁定，回复必比对正文；只填所选 RemoteInput 的一个结果键（`NotificationReplyInput.kt:41-52`）；不可变 PI 事前拒绝（`NotificationActions.kt:269-271`）。没有找到发错会话的路径。
- **派发与失败语义**：远程异常、带原因的取消异常按 UNKNOWN，平台明确拒绝和安全异常按 FAILED（`NotificationActions.kt:582-587`）；返回码 1（意图退回调用者）不当作已启动（`:440-441`）；每个写操作只发一次，修复只重复读取不重复发送（`NotificationOperations.kt:57-66`）。成立。
- **自身通知护栏、常驻通知护栏**：只拒绝自身包的通知和"常驻且未提升"的通知，不会误拒普通场景（037、038 两条拒绝在实施方派生汇总中可见，原件未打包，见 §4）。
- **清除循环不会无限等待**：`awaitNextCheck` 本身不抛超时（`UiBudget.kt:84-87`），但每轮读取前的 `current()` 会做预算检查（`UiBackend.kt:383-386`），所以超时后落到 UNKNOWN。成立。

---

## 3. 设计与简化

### 3.1 通知整块的价值与代价

- **基线**：`7d5be70` 只有 `notifications` 展开通知栏（`SystemActions.kt`、`RpcMethods.kt:20`）；读、写、监听器全部是旧候选 `3f45af6` 加的，本轮把它们重写成现在的形态（本轮对通知相关 10 个文件 +1,719/−559）。
- **代码量**【重算，`git diff --stat 7d5be70 HEAD`】：通知相关主代码与 `ActivityTasks.kt` 共 +1,733 行；Kotlin 测试 1,468 行；主机测试 597 行；`notifications.md` 1,413 词（旧候选 923 词）。
- **真实任务里帮到了什么**：两阶段八次效率对照（GitHub、Maps 两类任务，`ST/efficiency/*`、`ST/16-delivery/efficiency/*`）的 rollout 中，通知命令只出现在加载的指引文本里，没有一次被调用【原件，文本检索】。现有价值证据全是为通知专门搭的场景。
- **依赖的隐藏接口与 shell 特权**：
  - `ServiceManager.getService("notification")` 与 `INotificationManager.Stub.asInterface`；手写 `INotificationListener` 的 Binder（`NotificationAccess.kt:72-87`）。
  - `registerListener`：平台要求系统或 `STATUS_BAR_SERVICE` 权限（AOSP 16 NMS :5629-5633、:6443-6447），即依赖 shell UID 的特权；组件名伪装成 `com.android.shell/…`。
  - `unregisterListener`、`getActiveNotificationsFromListener(TRIM_FULL)`、`cancelNotificationsFromListener`、`ParceledListSlice.getList`、`UserHandle.getIdentifier`（反射）。
  - `PendingIntent.sendAndReturnResult`（隐藏方法）；`ActivityOptions.setPendingIntentBackgroundActivityLaunchAllowedByPermission`（34 上隐藏）；`setLaunchDisplayId`。
  - 落点相关：`ActivityTaskManager.getService().getTasks(…)` 与任务对象字段反射，要求 `REAL_GET_TASKS`；`ActivityManager.getCurrentUser`。
- **状态标志清单**：`NotificationOperations.access`（每个 worker 一个，`NotificationOperations.kt:27`）、`NotificationAccess.possibleRegistered`（`NotificationAccess.kt:16`）、`NotificationOperations.cleanupFailure`（`:29-30`，显示在 `status` 与管理页）。与第三轮"一个对象、一个标志"的要求一致；旧候选里的"cleanupPending"语义已改名并分离出诊断。
- **最小形态及其损失**：
  - 只读（list/get）：保留"看见通知"这一最常用能力，损失回复和按钮。回复是对 agent 最有价值的写操作（不进 App 就能回消息），不建议砍。
  - 我建议的最小形态是：读、回复、非 Activity 按钮、清除在所有会话保留；打开通知和 Activity 类型发送只在主屏会话支持（即 E）。损失的只有"在后台屏里直接跳到通知指向的那一页"；替代路径是 `get` 读内容、后台屏里 `open` App 自己导航，或切到主屏会话。

### 3.2 后台屏 Activity 通知：建议删除通知专用的事前检查与落点确认

如果用户选 E，可删：`NotificationActions.kt` 的 `taskPreflight`、`confirmLanding`、`unconfirmedLanding` 中的落点部分、`send` 中的目标屏分支（约 100 行）；`NotificationMethods.activityTarget` 及它在 `UiRun.kt:109-115`、`RpcMethods.kt:56-59`、`UiPublication.kt:47`、`UiPostAction.kt:13` 的接线；对应的 8 个左右 JVM 测试；指引里两段（约 250 词）。`ActivityTasks.kt` 仍由 `open` App 使用，保留。

如果用户选 F：保留事前检查，删掉"creator 可见才算完成"，改成派发结果为主，另加搬屏检测。代码量与现在相当。

### 3.3 选择串：保留，但缩短并修正拒绝回执

- 对 LLM 好用的地方：只有一个参数，原样复制；坏串在请求边界就被拒（`NotificationSelection.kt:83-111`），安全失败。指引要求"从一次新的 get 复制完整选择串"，表述清楚。
- 代价：一个两按钮通知的选择串长 534 个字符（`R/commands/formal/057-…/command.json`）【原件】，每次写操作都要搬运一遍。指纹用完整 SHA-256（64 位十六进制）是过度的：这里要防的是 App 更新，不是攻击者。截到 16 位十六进制后，长度约减半。这是实施细节，不需要用户裁决。
- 缺陷 2 的修正属于这一块。

### 3.4 监听器生命周期：现状可接受；另有一个更简单的整体做法供参考

- 现状【源码】：一个 worker 一个监听对象；lease 内第一条通知命令登记；会话通过 `Sessions.invalidateLease`（`Sessions.kt:329-332`）调用 `UiBackend.clear`，在 UI 单线程执行器上清理后调 `endLease`（`UiBackend.kt:643-653`）；注销失败只记诊断、不阻塞（`NotificationOperations.kt:88-99`；会话准入只看 `ui.cleanupFailure`，不含通知诊断）；被平台以"未知监听器"拒绝时每条命令修复一次（`:50-66`、`:101-121`）。worker 死亡后平台通过死亡通知清掉登记（AOSP 16 `ManagedServices.java:1958-1961` 登记时挂死亡通知）。
- 没有写进文档的残留【源码推断】：注销路径 `removeServiceImpl` 不解除死亡通知（AOSP 16 `ManagedServices.java:1922-1943`，全文无 `unlinkToDeath`）。所以"每用过通知的 lease 留一个"会累积到 worker 退出。第三轮在 API 34 上测得重复登记会让系统进程的强引用从 2 增到 6。每 lease 一次时，要到上限需要上万个用过通知的会话，普通使用无害；但"不会累积"这句话不成立，文档应写成"随用过通知的会话数增长，worker 退出后归零"。
- 更简单的做法（供用户参考，不是缺陷）：worker 首次用到通知时登记一次，之后不注销，进程退出由平台清理。好处：累积恒为 1；可删除 `endLease`、`cleanupFailure`、`status` 诊断字段、管理页文案及 4 个左右测试和一段指引。代价：worker 空闲时系统里一直挂着一个 shell 身份的监听器，接收全部通知的回调（我们的 Binder 直接丢弃）。第三轮因为这个代价选了"每 lease 一次"，且已由用户确认；现实现干净、有测试，我不建议再改，除非用户更在意代码量。

### 3.5 指引太长

`notifications.md` 1,413 词，包含大量调用者不需要的内部细节（UTF-16 单位、192 KiB／96 KiB 预算、null 与空串区分、修复只重复哪次读取、注销失败的诊断字段清除条件等）。对 LLM 调用者需要的只有：怎么读、怎么拿选择串、四种写的含义、哪些情况会被拒、UNKNOWN 时不要重发、平台接受不等于 App 已处理、后台屏的限制。估计可以压到一半以内。

### 3.6 评审方前几轮建议的回看

- **第三轮 §3.1 第 1 项"发送后以 creator 包和 UID 可见作为完成条件"——撤回**。它在代理场景必然产生假 UNKNOWN 并停 batch；在同 App 场景，平台返回码 0/2/3 已经说明启动发生了，可见性检查只证明"现在可见"，不证明因果（同一 App 之前就在后台屏上时也会"确认"）。它的维护成本（观察后再读任务、多一个 pending 状态、相关测试）大于收益。
- **第三轮把用户的选择框成"维持现状"与"加 `queryIntentComponents`"两项——框窄了**。漏了"后台屏不支持"和"检测搬屏"两条更便宜的路。第三轮自己承认事前检查拦不住接收方任务被搬，却没有把"根本不在后台屏做这件事"列为候选。
- **第三轮"暂按 creator 做事前拒绝"本身不算错**：它是零接口代价的有限保护，066 证明它确实在同 App 场景挡住了一次发送；但它的覆盖面比第三轮写的还窄，见 §5。
- **第三轮 §2.4 每会话登记一次**：实现与预期一致；只是"累积等于用过通知的会话数"这条，第三轮写在依据里了，实施方文档没写，补上即可。

---

## 4. 证据审计

| 实施方声称 | 核对结果 | 依据 |
| --- | --- | --- |
| Termux 本地按钮动作两次都 done，旧串对非消息类按钮仍有效（005、019） | **部分成立（只能核派生汇总）**。`R/main-vd-progress.json` 与 `R/costs.json` 记录 005/019 `done`、`dispatchStarted=true`；但 001–053 号命令的原始 stdout 和 trace 没有打进材料包（`R/evidence-index.json` 列出了它们，目录里只有 054 起的命令）。"action.txt 从 1 行变 2 行"无法核实。 | 【原件：派生汇总】 |
| 正文变化后旧串回复被拒、无派发（017）；新串回复 done（021），只写入本地文件 | **部分成立**，同上只有派生汇总：017 `not sent`、退出 2、`dispatchStarted=false`、原因"text or bigText changed"；021 `done`、原因"Platform dispatch returned; App outcome is not verified"。"平台接受不等于聊天送达"这条边界写得如实：样本是 Termux:API 的本地广播回复，没有任何真实聊天 App 或短信回复。 | 【原件：派生汇总】 |
| 按钮标题变化后旧串被拒（030）；短信正文变化后旧串清除被拒（045） | **部分成立**，派生汇总一致；原件未打包。 | 【原件：派生汇总】 |
| 同 App 后台屏打开成功并确认落点（057） | **成立**。stdout 显示 `preflight.state=clear`、`code=0 START_SUCCESS`、`landing.state=observed`、匹配任务 #263 在 Display 2，`done`。样本 1 次。 | `R/commands/formal/057-…/stdout.bin` 【原件】 |
| 同 App 隐藏任务零派发保护（066） | **成立（作为拒绝）**：`not sent`、退出 2、`dispatchStarted=false`，冲突任务 #264 在 Display 0、`visible=false`；067 确认 #264 仍在主屏。**但"若发送就会被搬"是推断**，没有对照发送；由 075 的跨 App 搬迁和 AOSP `ActivityStarter` 推断是合理的。 | `R/commands/formal/066-…`、`067-…` 【原件】 |
| Feeder 打开网址：已派发、UNKNOWN/3、batch 第二步未执行 | **成立**。074 batch 输出 `planned=2`，只有 step 1，`exit=3`；`code=0`、`displayId=4`、`landing.matchingTaskCount=0`、可见 `com.chrome.beta/10148` 任务 #266；观察时树里只有地址栏 `rfc-editor.org/info/rfc10021/`（`saved-bodies/formal/074-…tree.json`），076 稍后才出现正文，且树里有 2 处 `collectionLimit`（与实施方所写一致）。 | 【原件】 |
| Chrome 任务从主屏被搬到后台屏，会话结束后不回主屏 | **成立，且比稿件写得更重**：073 中 #266 是主屏**可见的顶部任务**；075 在 Display #4；078 Activity 处于 finishing；089 不在任何显示屏、只在最近任务。 | 073/075/078/089 【原件】 |
| Feeder 本地副作用：通知已消失，数据库行 31 `notified=1`、`unread=1` | **成立**。 | 075 sqlite 输出 【原件】 |
| 监听器 live 1 到 0 | **后台屏链成立**：056 本组件在 Live 列表中（1 条），060、070 不在（0 条）。主屏链（004–053）只有派生汇总。只数平台登记表，不涉及 JNI 引用——实施方自己也这样限定，如实。 | `R/commands/formal/056/060/070` 【原件】 |
| 故障窄验：A 情形注销前故障保留诊断；B 情形准备失败、未完成；C 情形"未知监听器"真实修复后清除成功 | **基本成立，B 的失败如实保留**。实施方结果页明确写"B 有真实准备失败，不能写成完整 A/B/C 全通过"，并分列 C 的非通知可用性与真实修复。我只读了结果页与故障补丁（`ST/15-notifications/runtime/listener-fault/listener-fault.patch`，确认故障注入只在注销前后抛错），没有逐条核日志。注意：故障是在单独构建的带注入 APK 上跑的，不是正式 APK。 | `…/listener-fault/runtime/api34/result.md` 【原件，部分】 |
| 内部 review "0/0 关闭" | 不作为证据。§2 的缺陷 2、3、4 内部 review 均未提出。 | — |

关于"单次成功被说成普遍结论"：实施方在通知部分措辞克制，多处写明"不能推广"。唯一偏乐观之处是把 066 称为"已实证的保护"——实证的是"拒绝发生了"，不是"若发送会造成搬迁"。

### 测试抽查

- JVM 测试通过假平台对象驱动真实的 `NotificationActions`／`NotificationOperations` 逻辑，断言的是外部可见行为：派发次数、结果、错误码、调用顺序（例如 `NotificationActionsTest.kt:314-337` 断言"读任务、发送、观察、再读任务"的顺序和最终 done；`:204-218` 断言非消息类进度变化不使选择失效；`NotificationReadTest.kt:12-49` 断言一个对象只登记一次、结束时注销一次）。这是行为测试，不是复述实现。
- 缺口：
  - 没有测试覆盖"Sessions 结束 lease 后，UiBackend 确实调用了通知注销"这条接线；`SessionsTest` 只测 `clearLeaseUi` 被调用，`NotificationReadTest` 直接调用 `endLease()`。真实接线由运行原件（053、060、070 的 live 0）覆盖。
  - 缺陷 2 的行为被测试固定成了预期（`:193-202`、`:220-246`）。
- 主机测试【实跑】：`venv4/bin/python -m pytest -p no:cacheprovider -q androidctl/tests/test_notifications.py`，52 项通过。主机端只做参数校验和输出格式，逻辑很薄。

---

## 5. 对第一道待决题的评价

### 5.1 选项事实核对

- A（维持现状）、B（用 `queryIntentComponents` 查初始接收方）、C（取消事前检查，派发与观察分开报）、D（保留 creator 检查，派发与观察分开报）四项都是真实可实现的候选；稿件对 B 的边界判断正确：查询只能看到 Feeder 自己的中转 Activity，看不到之后打开的 Chrome（Feeder 中转页另发 ACTION_VIEW 再 `finish()`，见 `runtime-materials/feeder-2.25.0/source/` 三个文件，我只读了稿件的引用，未逐行复核这些源码）。
- 稿件对 D 的收益判断（"不再因身份不等而停 batch"）成立；代价写了"仍拦不住代理搬 Chrome"，但没有写这次被搬的是**用户正在看的前台任务**，以及会话结束时它的界面被销毁。

### 5.2 漏掉的候选

- **E：后台屏会话不支持会启动界面的通知发送**。open、Activity 类型 action、Activity 类型 reply 在后台屏一律事前拒绝（not sent，退出 2），错误信息指向替代路径（`get` 读内容；后台屏里 `open` App；或主屏会话）。广播／服务类型的 action、reply 和 dismiss 照常。
  - 收益：唯一能排除"通知把别的 App 的主屏任务搬到后台屏"的做法（不论创建者、代理、系统通知）；删除通知专用的事前任务检查、落点确认、目标屏选项和相关测试、指引；缺陷 3 自动消失；不需要任何新接口；不再有这道待决题。
  - 损失：后台屏里无法直接跳到通知指向的那一页。现有证据中这个能力只成功过一次（057，Termux 打开自己）。另外，现在的事前检查会拒绝"用户最近用过、在最近任务里仍有活动记录的 App"，而用户经常收到通知的 App（聊天、邮件）多半正属于这一类，所以即使保留，后台屏打开通知能成功的场景本来就窄【推断，基于 `getTasks` 返回含隐藏任务，066 已体现】。
- **F：D 加搬屏检测**。保留 creator 事前检查；平台接受即为派发结果；观察后对比发送前后的任务表，任何"原来在别的屏、现在在目标屏"的任务列为 `movedTasks`，此时结果报 UNKNOWN（或至少在默认文本里醒目提示"某 App 的任务已从主屏移到后台屏，会话结束时它的界面会被销毁"）。
  - 收益：保留后台屏打开通知；伤害发生时一定被报告，不会像 D 那样被 done 掩盖；不需要新接口，数据已有（074 的事前检查已经读到了 #266 在 Display 0）。
  - 损失：仍然不能阻止伤害，只能事后告知；维护量与现在相当。
- 第三种可能（不推荐在这一题内做）：后台屏移除时不销毁内容，让任务回到主屏。这会让 agent 在后台屏开过的所有 App 在会话结束时出现在用户主屏上，影响整个后台屏设计，需另题评估。

### 5.3 我的推荐

**E**。依据：（1）唯一实测的跨 App 情形造成的是不可逆的用户可见伤害，A/B/C/D 都挡不住；（2）真实任务证据中没有一次需要在后台屏打开通知；（3）事前检查本身就让后台屏打开通知只对"用户最近没用过的 App"可用；（4）这是 alpha 产品、单人维护，删掉一整段特例比维护一个覆盖不全的护栏更值。如果用户明确需要后台屏打开通知，选 **F**，不选 D。

**对实施方推荐 D 的评价**：依据中"把派发接受与观察分开报"是对的（应当采纳，E 和 F 在主屏和其它路径上也用这个原则）；但"保留 creator 检查即可"低估了跨 App 的代价，且 D 会把 074 这类伤害报成 done。

**对评审方第三轮建议的评价**：如 §3.6，"creator 可见才算完成"应撤回；"先按 creator 做事前拒绝"作为暂行做法不算错，但第三轮给用户的二选一框窄了。

剩余未知：
- 真机和厂商 ROM 上 `setLaunchDisplayId` 与已有任务的搬迁行为是否相同【需设备】。
- 系统通知（创建者为 `android`／SystemUI）打开设置时是否同样搬迁主屏的设置任务【推断，需设备】。
- Chrome 被销毁界面后，标签页和数据是否完整恢复（实施方明确未验）。
- E 方案下 agent 的替代路径（后台屏 `open` App 再导航）在真实任务中的成本，未测。

---

## 6. 需要用户裁决的新取舍

1. **后台屏 Activity 通知（第一道待决题的替代表述）**
   - 候选：E（后台屏不支持，推荐）；F（保留并检测搬屏）；D（实施方推荐）；B（加查询接口）。
   - E 得：彻底排除这类搬屏，删一段特例；失：后台屏无法直接跳到通知目标页。
   - F 得：保留能力，伤害一定被报告；失：伤害不能阻止，维护量不变。
   - D 得：batch 不再被假 UNKNOWN 停下；失：伤害被 done 掩盖。
   - B 得：直接跨包启动可事前拦截；失：多一个隐藏系统接口，对代理和系统通知的后继跳转无效。
   - 剩余未知见 §5.3。
2. **（可选）监听器改为每个 worker 登记一次、不注销**：只在用户想进一步减代码时考虑；得失见 §3.4。我倾向维持现状。

以下归实施细节，不需要用户定：缺陷 2 的修正（内容变化被拒时不返回选择串）、缺陷 4 的 `SKILL.md` 修文、选择串指纹缩短、指引压缩、文档补写"死亡通知随用过通知的会话数累积"、Android 16 提升常驻通知的说明。

---

## 7. 没能核实的内容

- 主屏链 001–053 号命令（Termux 本地按钮与回复、旧串拒绝、短信正文变化、自身与常驻护栏、深层坏串）的原始输出：材料包未收录，只能核实施方的派生汇总。
- 故障窄验（A/B/C）的逐条日志：只读了结果页和注入补丁。
- Feeder 2.25.0 源码中中转 Activity 的行为：只读了稿件的引用，没有逐行复核。
- `STATUS_BAR_SERVICE` 是否在 shell 的权限清单里：根据 NMS 的检查推断 shell 具备，未下载 Shell 清单核对。
- 真实 Activity 类型按钮、真实聊天 App 回复、媒体按钮：实施方未测，我也无设备。
- Android 15 及以上监听器引用是否同样累积：无原件（第三轮已记录 API 35 上堆导出被拒）。
- 所有关于后台屏和任务搬迁的结论只来自 1 台 API 34 模拟器、每个场景 1 次运行；MuMu 不能建后台屏，真机与厂商 ROM 未覆盖。
