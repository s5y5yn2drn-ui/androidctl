# 红队评审：对评审方替代方案与用户决定的攻击（候选 `3f45af6`）

写作范围：只攻击替代方案，不平衡、不替其找优点。路径约定：
- `R` = 调研候选只读克隆 `scratchpad/review-copy`（Kotlin 在 `android/app/src/main/java/com/rainng/androidctl/`，下文 Kotlin 文件名均指此目录）。
- `T` = 实施方包 `NEW/ultragoal/2026-11-11-next-improve`。
- `U` = 我从 `android.googlesource.com` 下载的上游固定副本，目录 `scratchpad/agents5/23/up/`，文件名带版本标签（`android-14.0.0_r1`、`android-15.0.0_r1`、`android-16.0.0_r1`；`*.a14`/`*.a16` 同义）。包内已有的 A14/A15/A17 窗口源码引用为 `T/research/route-review-3f45af6/sources/coordinate-window-entry/<tag>/`（下称 `CWE`）。
- 证据类别：【原件】【源码】【重算】【推断】【需设备】。本评审没有设备、没有 Android 构建、没有运行包内脚本。

---

## 1. 摘要（按严重程度）

1. **坐标接收者 C 接入常驻 worker：SF 的 ack 背压会把 worker 的慢回调变成全系统输入窗口信息延迟。** SurfaceFlinger 在任一 listener 未 ack 时把非强制更新压住不发，InputDispatcher 也是同一组 listener；本地 ack 要等 Java 回调返回。A14/A15/A16 一致。实施方两份材料都没有提到。（硬性风险，需条件：回调无锁、只在单条命令内短期注册）【源码】
2. **退路链最后一级"不限制"把安全方向倒过来：** 越不知道窗口归属越放开，同时拆掉了"坐标输入不能打到 AndroidCtl 自身管理界面"这条 UID 护栏。（硬反例）【源码】
3. **后台屏会话的 `notifications open` 带上 launchDisplayId 后会被平台采纳。但如果目标 App 在主屏已有任务，平台会把真人的任务整个搬到后台屏。** broadcast 或 service 类 PendingIntent 发出前根本无法判断会落在哪块屏。所以"做不到就拒绝"只能在发送前按类型拒绝，不能靠发送后检查。（硬反例，需加前置规则）【源码】
4. **"还是那条通知"不比对发布时间、也不比对内容时：** 聊天通知的标题、来源、按钮序号都不变，新消息到达后，"标为已读"或 dismiss 会作用于调用者没看过的消息。（硬反例）【源码＋推断】
5. **后台屏默认拒绝的冲突检查把主屏后台任务（recents）也算在内，常用 App 几乎都会被拒。** 显式另开对 singleTask/singleInstance 无效。App 内部之后再跳转，仍会把真人的主屏任务搬走。（需条件）【源码】
6. **每命令注册通知 listener：** ManagedServices 从不 `unlinkToDeath`，每次 register 都在 system_server 留一个强全局引用，直到 worker 死亡。单对象方案只减少代理数，不减少死亡回调。注销失败期间，listener 还会持续拿到 URI 授权和完整通知包。（需条件）【源码】
7. **对任意输入框按位置设置选区：** TextView 对已可编辑但未聚焦的框不会请求焦点；会弹出浮动选择工具条；选区与当前相同时返回 false。偏移基准是 App 的真实文本，Contacts 原件的无障碍文本带 `, 姓名` 后缀，长度 32 而真实文本 28。（需条件）【源码＋原件】
8. **借贴关浮层：** 恢复原剪贴板时如果不给原内容加标记，浮层会把真人原来的剪贴内容弹出来；如果加了标记，原剪贴的 extras 就被改了。目标 App 粘贴时触发的"已从剪贴板粘贴"toast 关不掉。模拟器上不管来源都会抑制，"仅模拟器验证"测不出真机上的来源条件。（需条件）【源码】
9. **窗口缺席时"其它条件立刻报错"：** 平台会把被完全遮挡、未聚焦的窗口从无障碍窗口列表里拿掉，窗口 ID 不变、之后还会回来，这样会误报失败。（需条件）【源码】
10. **`session open --display` 必填，且不再隐式取得会话：** 重开会话会销毁后台屏和其中的任务，而新设计让"去开会话"的提示大幅增多。worker 重建后"上次目标"丢失，调用者容易照示例开到主屏，侵入真人的屏幕。（需条件）【源码＋推断】
11. **`--diff-from PATH`：** 历史配额是 150 个文件、20 MiB，现基线受保护，PATH 不受保护。保存的 `.txt` 是按预算截断的阅读视图，不是完整规范记录。batch 中间步骤没有可引用的路径。（需条件）【源码】
12. **启动落点"未确认就停"扩到主屏：** WebAPK、TWA 这类跳板到 Chrome 的 App，以及首启就弹权限框的 App，在现实现的匹配规则下会被判未确认。（需条件）【源码＋推断】
13. **稀疏树、目标规则、`type --via`、结果与退出码统一等小项各有一处要补的条件，** 见逐项。其中"落点未确认改成 failed"会诱发重复发送 SEND 类意图。（需条件）【推断】

---

## 2. 逐项

### 2.1 `session open --display main|virtual` 必填；主机不存显示种类

**现状核对。** `Sessions.acquire()` 和 `acquireUi()` 在没有租约时用 `keepLease()` 隐式建立一个 MAIN 租约（`Sessions.kt:57-64,93-98,140-152`）。shell 和文件命令走 `sessions.acquire(owner)`（`ShellJobs.kt:33`、`FileJobs.kt:24`），通知、剪贴板、apps、wake 走 `UiBackend.execute → acquireUi`（`UiBackend.kt:108`）。空闲 10 分钟后 `expire()` 让租约失效（`Sessions.kt:241-244,301`）。`prepareChange()` 在同一 owner 已持有租约时会新建租约并让旧租约失效，后者触发 `clearLeaseUi`（`Sessions.kt:100-123,279-282`），而 `UiBackend.clear()` 会关闭后台屏（`UiBackend.kt:478-497`，`lease.target.stream?.close()`）。【源码】

**A1 重开即销毁：新设计会诱发更多次重开。**
- 场景：CLI 超时或 UNKNOWN 后，调用者不确定会话是否还在；或者收到任何一条"没有会话"的报错。按新规则，所有命令族都要先 `session open`。LLM 的典型补救是再执行一次 `session open --display virtual`。如果原租约其实还在，这一下会销毁后台屏，屏上 App 的未保存内容和任务一并丢失（`displays.md` 和 `session close` 的帮助文本都写明 "destroys its tasks and unsaved UI"，`app.py:757-760`）。
- 证据：【源码】`Sessions.kt:110-122`；【推断】调用者的行为模式。
- 后果：不可逆地丢掉后台屏上的工作。
- 严重程度：**需条件**。条件是：同一 owner 已持有同种类租约时，`session open --display X` 原样返回当前租约，不清空任何东西；确实要重置时必须显式写 `--reset` 或用 `session close` 加 open。

**A2 "报错带上次目标"只存在 worker 内存里。**
- 场景：worker 被 **Refresh status**、**Force executor exit**、Shizuku 重启、设备重启或 App 升级重建（`connection.md:155-175`）。新 worker 的 `Sessions` 里没有旧 owner 的任何记录。现结构里，租约失效后连 `current` 都会被清空（`Sessions.kt:279-282`），要带上次目标得另加一张表。
- 后果：worker 重建后报错里没有上次目标。调用者只能依赖自己的上下文；上下文被压缩，或者换了一个续跑的 agent 时，最可能照 SKILL 首页示例执行 `--display main`，于是从后台屏转到真人正在用的主屏上操作。这正是被删除的主机偏好原来要拦的序列。
- 证据：【源码】`Sessions.kt` 全文无跨租约的历史；【推断】调用者行为。
- 严重程度：**需条件**。条件有三条：(a) 报错必须区分"本 worker 有记录：上次是 X"和"worker 已重建，上次目标未知"，不要在第二种情况下给缺省值；(b) SKILL 示例不要写字面量 `main`，改写 `--display KIND` 并解释如何选；(c) 一次性写入的这份记录也要随 `revoke` 清掉。

**A3 后台屏建屏前置条件与 wake 必须先有会话，形成三步绕路。**
- 场景：熄屏或锁屏时要开后台屏。`mainReadyForWorkDisplay` 要求主屏亮且已解锁（`TargetManagement.kt:103-107`）。新规则下 `wake` 也要先有会话，于是要走 `session open --display main`、`wake`、`session open --display virtual` 三步。现状下 `wake` 是设备级命令，可以直接执行（`displays.md:62-65`）。
- 严重程度：**轻微**。最小修补：`wake` 免会话，或者建后台屏失败时在报错里直接给出这三步。

**A4 共享的 `default` 会话名。** 两个调用者用同一个名字时，B 执行 `session open --display main` 就会销毁 A 的后台屏。现状和 D 方案下同样存在，不是新问题。**轻微**【源码】`SessionOwner.sameSession` 只比较 pairing 和 name。

**A5 依赖面。** 契约样本里有 `expectedDisplayKind` 字段（`contract/wire-contract.json` 第 6、28、50 行），`test_display.py`、`test_ui_batch.py`、`test_wire_contract.py`、`test_cli_arguments.py` 都依赖偏好文件或种类字段，`SessionsTest.kt` 有 25 处 `acquire(`，SKILL 首页第 22–30 行与 `displays.md` 全文描述的也是缺省 main 加偏好。这些都要一起改，而且契约要按 AGENTS.md 的流程重新生成。**轻微**（是工作量，不是反例）【源码】

**A6 batch。** batch 不接受 `session open`（只接受 UI 命令，`batch.md:14-17`）。删掉种类冻结后，batch 的整个执行期间都由设备租约决定目标；只要执行中有资源占用，租约就不会到期（`Sessions.kt:241-244`）。**查过没问题。**

### 2.2 不碰屏幕的命令也必须先开会话

**现状核对**见 2.1。后台 shell 任务在 `GO` 后调用 `releaseForeground()` 释放前台资源（`ShellJob.kt:105-118`）；任务的后续操作 get、output、cancel 只校验 owner，不要求当前租约（`ShellJobs.kt:56-80`、`JobTable.kt:26-33`）。文件传输在进行中依赖当前租约（`FileJob.kt:37`），而租约有资源时不会到期。【源码】

- **B1 一条命令变两条。** 例如 `shell 'pm list packages'`、`files pull`、`clipboard get`、`apps`、`notifications list`、`wake`。代价是多一次往返和一次概念选择。纯 shell 任务被迫选一种显示：选 virtual 会建出无用的屏，还要求主屏亮且解锁；选 main 没有副作用。**轻微**。最小修补：报错文案直接给出 `session open --display main`，并说明 main 对不碰屏幕的命令没有副作用。
- **B2 后台任务的后续操作。** 租约过期后，get、output、cancel 仍然可用，查过没问题。但长时间运行的后台 shell 任务与"新命令必须有会话"同时存在时，调用者想再起一个诊断 shell 就得重新 open。如果对方是 virtual，重开会销毁后台屏（见 A1）。**需条件**，修补同 A1。
- **B3 鸡生蛋。** `session open --display virtual` 失败后想用 shell 诊断（例如 `dumpsys display`），可以先 `session open --display main` 再执行 shell。不存在死锁。如果别人持有会话（busy），现状下 shell 同样不可用。**查过没问题。**
- **B4 `wake` 在没有会话时的用途**见 A3。**轻微。**

### 2.3 启动落点：界面稳定后目标屏上出现目标 App 才算确认，未确认就停，主屏与后台屏同规则

**现状核对。** 后台屏的确认要求：唯一一个可见叶任务，其 `topActivityInfo.uid == handler.uid`，并且 `leaf.topActivity ∈ {handler.component, actualComponent}`（`ActivityTasks.kt:117-150`）。主屏用"匹配根任务自准备以来发生过转换"（`ActivityTasks.kt:219-259`），未确认时不改退出码（`ActivityLaunch.kt:253-266`）。歧义 handler 在发送前就以 NOT_SENT 拒绝（`ActivityLaunch.kt:200-217`），所以不会出现系统的选择器界面。【源码】

- **C1 跳板到另一个包。** WebAPK 和 TWA（Trusted Web Activity）的入口 Activity 在 App 自己的包里，随后启动 Chrome 的 Activity，窗口和顶 Activity 的包变成 `com.android.chrome`。按包和 UID 确认就会判未确认，按"未确认就停"会一直停。Google Lens 之类的桩 App 也是这种结构。**需条件**【推断：上游 TWA 实现不在固定材料里；证实办法：在模拟器上装一个 TWA 或 WebAPK，执行 `open --package`，看任务的 base 和 top】。修补：允许"目标屏上新出现或刚转到前台的任务，其 baseActivity 或 realActivity 属于目标包"。
- **C2 首启弹权限框。** targetSdk≥33 的 App 首启时常弹通知权限框。GrantPermissionsActivity 属于 permissioncontroller 包，被放在 App 自己的任务里，所以顶 Activity 的包不是目标包。按现规则 `leaf.topActivity` 不匹配，会判未确认。如果改按窗口判断：带模态触摸区的对话框窗口会把下层 App 窗口完全盖住，无障碍窗口列表会丢掉下层窗口（见 2.16 引用的 `AccessibilityController.java:1680-1700`），也判不到。**需条件**【源码＋推断】。修补同 C1，按任务的 base 判断。
- **C3 启动后立刻跳登录页。** 同包内跳转。如果新规则按包和 UID 判断不受影响，但现实现要求顶 Activity 等于 handler，所以会误判。用户的决定写的是"目标 App"，实现时必须放宽到包级别，否则主屏也会大量停下。**需条件**【源码】`ActivityTasks.kt:129-130`。
- **C4 "界面稳定"。** 游戏、视频启动页、Lottie 加载动画一直在变，界面永远稳定不了。如果确认以稳定为前置条件，就会超时判未确认。**需条件**：确认取观察窗口截止时最后一次采样，不要求稳定。【推断】
- **C5 目标 App 已在前台。** 按存在性判断时，启动其实什么都没改也会被判为确认，即误确认。主屏现规则要求根任务发生转换，比新规则严。**轻微。**
- **C6 主屏新增"停"的影响。** 现在主屏未确认仍是 exit 0，batch 不停（`ActivityLaunch.kt:253-256` 只对 virtual 加 exit 1）。改了以后，C1–C4 的情形会让主屏 batch 中断。**需条件**，修补同上。

### 2.4 后台屏要开的 App 在主屏已有任务时默认拒绝，显式选项才另开

- **D1 拒绝面远大于"主屏开着"。** 冲突检查扫描 `getAllRootTaskInfos` 的全部根任务，以及 `getTasks(Int.MAX_VALUE, filterOnlyVisibleRecents=false, …, INVALID_DISPLAY)`（`ActivityTasks.kt:50,85-107`），所以真人最近用过、留在最近任务里的不可见任务也算冲突。微信、Chrome、设置这类常用 App 几乎都会被拒。【源码】这个拒绝从平台行为看是对的：`findTask` 会跨屏复用任务（`U/ActivityStarter.android-14.0.0_r1.java:2846-2885`），然后 `reparent` 到目标屏（同文件 `2944-2999`，`differentTopTask = true` 后执行 `reparentToTargetRootTask`）。但代价是默认路径在常用 App 上基本走不通。**需条件**：报错要说明"是可见任务还是最近任务里的后台任务"，并直接给出显式选项。
- **D2 显式另开对 singleTask/singleInstance 无效。** `putIntoExistingTask = (NEW_TASK && !MULTIPLE_TASK) || isLaunchModeOneOf(SINGLE_INSTANCE, SINGLE_TASK)`（`U/ActivityStarter.android-14.0.0_r1.java:2860-2862`）。评审方建议在解析阶段拒绝，但只能拒入口 Activity。**硬反例的部分**：入口是 standard 的 App 用 `--multiple-task` 开在后台屏后，调用者在 App 里一点，App 内部启动它的 singleTask 页面（VLC 正是这种结构），平台就把真人主屏上的同包任务 reparent 到后台屏。显式选项拦不住，而且这一步发生在后续的 click 里，不在 open 里。【源码】最小修补：后台屏会话里每次动作完成后，对比主屏任务集合，发现同包任务被移入后台屏时，在回执里把它作为显著事件报告；文档也写明显式另开只保护入口。
- **D3 亲和性。** 新任务的 affinity 默认等于包名。App 内部带 NEW_TASK 的跳转会按 affinity `findTask`，同样可能命中主屏任务。**需条件**，修补同 D2。【源码】同上 2846-2885。

### 2.5 通知 listener：整个 worker 只用一个对象，每命令注册、用完注销，注销失败只记一笔下次补

**上游核对**（`U/ManagedServices.*`、`U/NotificationManagerService.*`）：
- 重复注册不抛异常，会重复登记。`registerServiceImpl(info)` 直接执行 `linkToDeath` 再 `mServices.add(info)`，没有去重（A14 `ManagedServices:1706-1716`；A15 `1786-1796`；A16 `1958-1968`）。`NMS.registerListener` 只检查 `enforceSystemOrSystemUI` 后就调 `registerSystemService`（A14 `NMS:4534-4538`）。【源码】
- 注销按对象身份删除全部登记。`removeServiceImpl` 倒序遍历，删掉所有 `asBinder()==` 且 `userid==` 的项，不提前退出（A14 `1670-1683`；A16 `1922-1935`）。评审方这一点成立。【源码】
- 注册或注销失败后对象可以复用。Binder 对象本身没有状态；只有已死的 binder 才会被 `registerServiceImpl` 拒绝（`catch RemoteException`）。【源码】
- A14、A15、A16 的上述逻辑一致。

攻击：
- **E1 死亡回调累积（单对象方案并未消除）。** 三个版本都**没有** `unlinkToDeath`：注销只做 `mServices.remove` 加 `onServiceRemovedLocked`（A14 `1685-1690`；全文只有 linkToDeath 出现在 1574、1709，A15、A16 相同）。`JavaDeathRecipient` 构造时创建强全局引用 `NewGlobalRef`，只有 binderDied 或 unlink 才降级（`U/android_util_Binder.android-14.0.0_r1.cpp:565-602`）。所以每注册一次，system_server 里就多一个持有 `ManagedServiceInfo` 的强全局引用，直到 worker 进程死亡。现状下每命令新建对象，还会额外多出 BinderProxy；单对象只省掉代理，死亡回调数照样等于"本 worker 生命周期内的通知命令数"。【源码】后果：长期轮询通知的 agent 会让 system_server 的 JNI 全局引用持续增长（ART 全局引用表有上限，超限会中止进程【推断】）。**需条件**：每 worker 注册一次、保持到 worker 停止或会话结束（即实施方题 4 的持有期思路），或者给每 worker 的通知命令数设上限。评审方"延长持有期没有收益"的论据只看了注册耗时，漏了这项。
- **E2 注销失败到补注销之间的副作用。** 登记还在时，`onServiceAdded` 已经执行 `updateUriPermissionsForActiveNotificationsLocked(info, true)`，把活动通知的 URI 读权授给了 shell（A14 `NMS:11046-11058`），这些授权直到移除才收回。每条新通知都会以完整 SBN 包裹发到 worker：listener 虽然什么都不读，跨进程的数据流仍在。A16 还有一项：`onServiceAdded` 会对持有 STATUS_BAR_SERVICE 的调用者把 `isSystemUi` 置真，注释写明 shell 也算在内（A16 `NMS:13119-13133`）；direct reply 的生命期延长更新只发给遍历到的第一个 `isSystemUi()` listener，然后 `break`（A16 `NMS:13523-13546`）。如果 SystemUI 重启后才重新登记，排在我们这个残留 listener 之后，就收不到这类更新。【源码＋推断】**需条件**：补注销不能只等"下次使用"，要在会话结束或 worker 空闲时主动补；报告里要写清残留期间的这些影响。
- **E3 并发串扰。** 通知命令都在单线程 executor 上运行（`UiBackend.kt:21`），而且 UI 资源互斥（`Sessions.requireResources`，`Sessions.kt:198-202`），不会有两条通知命令同时使用同一个对象。listener 回调什么也不做（`NotificationAccess.kt:66-82`），读取用的是 `getActiveNotificationsFromListener`，不经过回调。**查过没问题。**
- **E4 "不封锁"本身。** 注销实际会失败的原因几乎只有 system_server 侧 Binder 失败，这时整个系统也不正常。去掉封锁门是合理的。**查过没问题**（前提是做了 E1、E2 的修补）。

### 2.6 确认"还是那条通知"：比对标题、来源、按钮序号；回复再比对内容；调用者只复制一个串

**现状核对**：比对 key、postTime、动作序号、**动作标题**、creatorPackage/creatorUid（`NotificationActions.kt:238-293`）。【源码】

- **F1 聊天通知被替换（硬反例）。** MessagingStyle 通知的标题是会话名，新消息到达后标题不变，来源不变，动作布局（回复 / 标为已读）也不变。按新规则，`action` 执行"标为已读"或 `dismiss` 时都不比对内容，会把调用者没读过的新消息标成已读或清掉。现状下 postTime 每次更新都会变，正好拦住这种情况。【源码：现状比对点；推断：聊天 App 的标题行为】**硬反例**。最小修补：所有写操作默认都比对内容指纹（标题以外的文本、消息条数、`EXTRA_MESSAGES` 最后一条）；只有带进度或标记为 onlyAlertOnce 的进度类通知才降级为不比内容。也可以改比 `Notification.when`：聊天更新通常会改它，进度更新多半复用 builder 不改它（【推断】，需设备核）。
- **F2 必须保留"动作标题"。** 用户原话是"标题、来源、按钮序号"，没有提动作标题。媒体通知同一序号会在播放和暂停之间切换，只比序号就会按反。现状比对动作标题（`NotificationActions.kt:276-278`），新方案必须明确保留。**需条件。**
- **F3 无标题通知。** 自定义 RemoteViews 的通知、部分前台服务通知没有 `EXTRA_TITLE`，比对就退化成 key 加来源加序号，比现状的 postTime 弱得多。**轻微**，修补同 F1。
- **F4 选择串与过期。** 串里至少要有 key、creatorUid、动作序号、动作标题、内容指纹。没有时间成分就永不过期：每天重发、key 和标题都不变的提醒，会被几天前的串命中新的一条。**轻微**。可以在串里加 `when`，或者加一个签发时间并设一个宽松上限（例如 24 小时）。
- **F5 按钮序号漂移。** 系统生成的智能动作不在 `notification.actions` 里（读的是 `notification.actions`，`NotificationActions.kt:171-184`），所以不会让序号漂移。App 自己重排动作时，有动作标题比对就能拦住。**查过没问题**（前提是 F2）。

### 2.7 后台屏会话里打开通知：落到后台屏，做不到就拒绝

**上游核对**：
- 调用方选项会被采纳。`PendingIntentRecord.sendInner` 把 shell 传入的 options 作为 caller options（A14 `PendingIntentRecord:456-469`；A15 `529-537`；A16 `543-551`）。`SafeActivityOptions.getOptions` 分别检查原选项和调用方选项后合并，合并时用 `b1.putAll(b2)`，调用方的值覆盖创建者的值（A14 `SafeActivityOptions:189-204,256-266`；A15 `205,266`；A16 `183,244`），所以 `launchDisplayId` 以 shell 给的为准。调用方选项的权限检查用的是真实调用者（shell）的 UID，看 `isCallerAllowedToLaunchOnDisplay`（A14 `326-335`）。后台屏由 worker 以 shell 身份创建；typed open 已经能以同样身份落到这块屏，因此我推断权限上会通过（【推断】，`ActivityTaskSupervisor` 没有下载）。
- 后台启动限制。A14–A16 的 `BackgroundActivityStartController` 没有任何按显示区分的逻辑（三版 grep `displayId` 都没有命中）。候选用 `MODE_BACKGROUND_ACTIVITY_START_ALLOW_ALWAYS` 传递 shell 的权限（`NotificationActions.kt:209-229`）。【源码】

攻击：
- **G1 主屏已有任务会被搬走（硬反例）。** 典型场景就是真人正在用的聊天 App 来了通知：内容意图解析到该 App 已有的任务，平台按 `setTargetRootTaskIfNeeded` 把它 reparent 到后台屏（`U/ActivityStarter.android-14.0.0_r1.java:2944-2999`）。"落到后台屏"在这里意味着把真人的 App 从主屏拿走。PendingIntent 的目标组件拿不到：候选只读了创建者和类型（`NotificationOperations.kt:206-226`），而且创建者不一定等于目标（例如新闻 App 的通知打开 Chrome）。所以按 2.4 的规则在发送前查冲突，只能查创建者的包，查不全。**硬反例**。最小修补：后台屏会话里，如果创建者的包在其它屏有任务，就在发送前拒绝；发送后再核对主屏任务集合有没有变化，一旦有任务被移入后台屏，就作为显著失败报告（这时副作用已经发生，回执里要写明）。
- **G2 broadcast 或 service 类内容意图无法控制落屏。** 候选对非 activity 类型不带 options（`NotificationActions.kt:197-205`）。接收方 App 收到广播后自己 startActivity：如果它在主屏有可见窗口，后台启动限制会放行，界面落到主屏；如果没有，启动会被拦下，什么都没发生。两种结果都无法在发送前预测。"做不到就拒绝"在这里只能做成"后台屏会话里，非 activity 类型的 open 在发送前一律拒绝"。代价是这些通知在后台屏模式下打不开。**需条件**【源码＋推断】。
- **G3 范围漏项。** 用户只决定了 open。`notifications action` 对应的 PendingIntent 如果是 activity 类型（例如"查看""打开"按钮），同样会落到主屏或搬走任务。规则要覆盖所有 activity 类型的通知写操作。**需条件**【源码】`NotificationActions.kt:187-207` 对 action 和 open 走同一个 `pending()`。
- **G4 A14/15/16 一致性。** merge 逻辑和 BAL 无显示分支在三版里一致。A16 的 `setCallerOptions(opts, callingPid, callingUid)` 签名变了，但语义相同。**查过没问题。**

### 2.8 默认正文大幅精简

- **H1 隐去"默认动作"后调用者失去可发现性。** 需要定义什么叫"隐含"。如果把非 `clickable/scrollable` 状态所蕴含的标准动作也隐去，例如 `dismiss`(1048576)、`expand`/`collapse`(262144/524288)、`long-click`、`set-progress`、各方向滚动，调用者就不知道某个卡片可以用无障碍方式滑删，只好改用坐标滑动，而坐标路径正是规则最多的那条。【推断】**需条件**：只隐去由已打印状态直接蕴含的动作（clickable 对应 click、focus 类、show-on-screen），其余标准动作保留短名。
- **H2 收起"窄条窗口"。** 按几何规则会被一起藏掉的真实窗口有：heads-up 通知（SystemUI 顶部条，遮住 App 顶栏，坐标点击会落在它上面）、分屏分隔条（`TYPE_DOCK_DIVIDER` 被特别保留在无障碍列表中，`U/AccessibilityController.android-14.0.0_r1.java:1689-1694`，它是识别"当前处于分屏"的唯一线索）、聊天气泡、悬浮迷你窗。输入法候选条在 IME 窗口内部，不受影响；App 内的底部播放条是 App 窗口的一部分，也不受影响。**需条件**：按窗口类型白名单收起（状态栏、导航栏），不按几何规则；坐标路径强制全显（评审方已写）。
- **H3 无内容事件不打印。** 没有文字的 `WINDOW_STATE_CHANGED` 可能说明出现了新对话框，`VIEW_SCROLLED` 说明滚动确实发生过。不打印的话，只能靠"变化结论"兜底。**轻微。**
- **H4 按名字调用。** 标准动作名来自固定表（`UiSemantics.kt:35-48`），与系统语言无关。自定义动作的名字就是 App 的本地化标签，系统语言一变就变；两个自定义动作同名时必须拒绝。标准名与自定义标签相同时（例如自定义动作就叫 "dismiss"）需要命名空间区分。**轻微。**

### 2.9 `find` 默认搜全部已采节点

- **I1 输出膨胀。** Chrome 页有 4,226 个节点（评审方原件数）。查一个短词（例如"1""Add"）可能命中上百个，而 `find` 除了 `max_chars` 之外没有命中数上限（`ui_view.py:268-290,701-715`）。**需条件**：默认最多列 N 条，附总数和离屏条数。
- **I2 懒加载列表。** 原生 RecyclerView 不暴露未加载的项。"已采集全部节点里没有"会让调用者比现在更相信"不存在"。**需条件**：零命中且存在可滚动容器时，提示用 `scroll-until`。
- **I3 与 `--all` 冲突。** 现在 `--all` 同时控制"包含隐藏节点"和"附加字段"（`ui_commands.py:215`、`ui_view.py:258`）。默认改为包含隐藏节点以后，`--all` 对 find 只剩附加字段的含义，语义就分裂了。**轻微**，需要拆成两个开关或重新写文档。
- **I4 与 `reveal`、`scroll-until`。** 离屏命中带引用，可以直接 `reveal`，不冲突。**查过没问题。**

### 2.10 "可点目标"规则设备和主机一起改（16908348 归普通动作）

**现状核对**：设备端 `publicTarget`（`UiSemantics.kt:52-56`）同时决定截图标注（`ImageMarks.kt:151`）、命名时子节点的边界（`UiSemantics.kt:160`）和归并（`UiSemantics.kt:364`）；主机端 `is_target`（`ui_view.py:53,72-86`）只影响阅读。引用对全部节点都存在，与是否为目标无关（`UiRefs.kt:39-41`）。【源码】

- **J1 只带上下文点击的控件。** 原生 `setContextClickable(true)` 的 View 通常也可以点击，所以只靠上下文点击可用的控件很少见。它仍然有引用，可以用 `action REF --action context-click` 执行。损失只是在 `--interactive` 视图和截图标注里看不到它。**轻微。**
- **J2 命名归并的连带变化。** 去掉边界后，文本节点会被归并进最近的目标祖先的名字。如果网页上有一个包住整篇文章、带 onclick 的大容器，它的名字会吸收很多段落（上限 `MAX_CONTEXT_NODES`），名字变长，反而增加输出；子文本节点被标为已消费后，主机端还要靠 `local_names` 再展开。评审方说"经重算向好"，依据是 sub-report 19，按规则我不读。**需条件**【推断】。证实办法：用三份 Chrome 原件分别按新旧规则重算节点名长度的分布和 `nameConsumed` 节点数。
- **J3 契约与测试。** 契约样本里没有 16908348（`grep -c` 为 0），截图标注和命名的单元测试需要更新，但没有跨语言的硬冲突。**轻微。**

### 2.11 diff 改成 `--diff-from PATH`，删持久基线

**现状核对**：基线是工作区里的一份规范树记录，在配额回收中受保护（`history.py:17-18,49-54,126-143`）；历史配额是 150 个文件、20 MiB（`history.py:20-21`）。batch 里不做引用替换，中间步骤不下载没有请求的附件（`batch.md:27-41`）。【源码】

- **K1 路径被配额回收。** 网页快照加上截图，几步之内就可能挤出 20 MiB。调用者指定的旧文件可能已经被回收，而现在的基线永远不会被回收。**需条件**：被 `--diff-from` 读过的文件要加保护，或者回收时保留每个会话最近一份完整的规范记录。
- **K2 保存格式不够用。** `.txt` 正文是按 `--max-chars` 截断、按视图过滤后的投影（`observations.md:29,60-89`）。现在的 diff 比的是未截断的规范记录，并且有"预算扩大则输出全文"的判断。只拿 `.txt` 做基线会把被截断的节点误报成"新增"。所以要么每次快照多存一份规范记录（文件数翻倍，配额实际减半），要么接受精度下降。**需条件**，实现之前必须先定下 PATH 指向的格式。
- **K3 batch 中间步骤。** 计划在执行前就固定了，无法引用前一步在执行时才生成的路径（`batch.md:27-29`）。现在的 batch diff 也不做相邻步骤比较（`batch.md:62-67`），所以没有失去什么。**查过没问题。**
- **K4 拿错文件。** 不同设备、会话或连接的文件会被比较键拦下，输出全文（现 `ui_diff.py:27-56`，前提是保留这段逻辑）。同一会话但更早的文件不会被拦，变化会被归到最近一次动作上，造成误归因。**轻微**。回执里写出 base 的采样时间和序号即可。
- **K5 保存失败或 `--no-print`。** `--no-print` 照样保存，查过没问题。保存失败时没有路径可引用，现状下保存失败也不会推进基线，两者一样。**查过没问题。**

### 2.12 看小字改为"重新截一张，只要指定区域的原始分辨率"

**现状核对**：zoom 只能从本租约驻留的同一张图上裁（`VisualInput.kt:132-148`），`zoom REF` 用该 REF 缓存的边界（`images.md:67`）。后台屏的截图每次都通过 `captureDisplay` 新取（`WorkDisplay.kt:89-110`），所以重截在技术上可行。【源码】

- **L1 瞬时内容。** Toast、Snackbar、验证码闪现、动画中途的文字、视频画面：预览图里看到了但看不清，再截一张时已经没了。**需条件**（V8 的判别条件）。
- **L2 `zoom REF` 时点错位。** REF 的边界来自旧树，裁的却是新帧。如果界面滚动过或布局变过，裁出来的是别的内容，而且没有任何报错。**需条件**：重截时同时取一次目录或树的签名，与引用来源的签名不一致时，拒绝 `zoom REF`，只允许 `--region`。
- **L3 受保护窗口。** FLAG_SECURE 的内容在新旧两种方式下都是黑的。**查过没问题。**
- **L4 成本。** 每次 zoom 都要整帧截图再裁剪，多一次截图的耗时，量级是几十到一两百毫秒【推断】。**轻微。**

### 2.13 界面树只存非默认值

- **M1 两端默认值表漂移。** Kotlin 端签名用 `node.opt(field)`（`UiSnapshot.kt:103-142`），缺键时得到 null；主机端有的地方写 `node.get("visible") is not False`，有的地方直接下标 `node["view"]["name"]`（`ui_view.py` 中有 22 处）。设备端 `UiSemantics.kt` 有 25 处严格的 `getBoolean/getString/getInt`，例如 `child.getBoolean("visible")`（`UiSemantics.kt:159`），缺键就抛 JSONException，整个观察失败。这种失败是显性的，不是静默的。**需条件**：默认值表从共享契约生成，契约测试覆盖"某字段等于默认值"的样本。
- **M2 稠密节点和稀疏节点混在一起。** 如果有一条路径（例如 `FreshNodes`、`UiRefRepair` 新取的节点）产出的是稠密节点，同样的界面会得到不同的签名，造成假的"有变化"，和 N1 动作次序缺陷是同一类问题。**需条件**：只在定稿时统一稀疏化一次，签名也只在那之后计算。
- **M3 "未知"和"默认"。** 现在有 `unavailable` 数组来区分读不到的字段（`UiSnapshot.kt:103` 把它纳入签名）。稀疏化之后必须保留这个数组，否则读不到的布尔字段会被当成默认值 false。**需条件。**
- **M4 旧保存文件。** diff 基线有版本检查，不兼容时输出全文（`observations.md:79-81`）。**查过没问题。**

### 2.14 独立选区改为对任意输入框按位置设置（无障碍 SET_SELECTION）

**上游**：`TextView.performAccessibilityActionInternal` 中 `ACTION_SET_SELECTION`（`U/TextView.android-14.0.0_r1.java:14276-14302`）。

- **N1 对可编辑框不请求焦点。** 只调用 `requestFocusOnNonEditableSelectableText()`，这个方法只对不可编辑的文本请求焦点。对一个未聚焦的 EditText，选区 span 设上了，焦点却还在别的框上；随后的 `type --via editor` 或按键会写进当前聚焦的那个框。**需条件**：先确认目标是当前聚焦的编辑器，不是就先聚焦再选，并在回执里写明焦点所在的节点。【源码】
- **N2 浮动选择工具条。** 选区非空时会调用 `mEditor.startSelectionActionModeAsync(false)`，屏幕上弹出剪切、复制、粘贴工具条。在主屏上真人看得见；这个工具条也会进入树和截图，改变下一次观察。**轻微到需条件**，回执里应说明。【源码】
- **N3 选区与当前相同时返回 false。** `if (getSelectionStart() != start || getSelectionEnd() != end)` 不成立时直接 `return false`，会被误报为"动作失败"。**需条件**：执行前先读当前选区。【源码】
- **N4 偏移基准是真实文本，无障碍文本带标签。** E13 Contacts 原件里，焦点节点的 AX 文本是 `ICF11-CT-ICF11-CT-I-中文abcdef, 姓名`（textLength 32），而 IC 的 surrounding text 是 `…中文abcdef`（28 个字符）（`T/../../prototypes/device/followup/commands/e13-contacts-after-insert/stdout.bin`，即 `NEW/ultragoal/2026-11-11-next-improve/prototypes/device/followup/commands/...`）【原件】。TextView 的选区按 `mText`（真实文本）计算（`TextView:15387-15389`）。后缀标签只让末端越界，越界时返回 false；如果某个 App 用的是**前缀**标签，所有偏移都会静默错位，而纯无障碍路径没有手段核对。IC 路径能拿到 surrounding text 作为真实文本，这是它能核对、无障碍路径不能核对的地方。**需条件到硬反例**：前缀标签的实例我手上没有原件（【需设备】，证实办法：找一个用 `setText` 拼 contentDescription 或自定义 AccessibilityDelegate 加前缀的 App）。
- **N5 UTF-16。** TextView 两端都以 Java 的 char（UTF-16）为单位，一致。Compose 的 TextField 有 VisualTransformation 时，语义文本和偏移映射可能不一致（【推断】）。**轻微。**
- **N6 WPS。** E13 中 WPS 正文的 AX 属性是 `editable=false`、`text=null`（`inputs.md:33`）【原件经实施方转述】。无障碍选区在这里本来就不可用；评审方说"IC 选区在 WPS 上被忽略"，那么两条路在 WPS 上都不行，不是新的损失。**查过没问题。**

### 2.15 借贴保留并"关掉系统'已复制'浮层"

**上游**：SystemUI 的 `ClipboardListener.shouldSuppressOverlay` 只在 **(模拟器 或 来源 == `com.android.shell`)** 且 description extras 里 `com.android.systemui.SUPPRESS_CLIPBOARD_OVERLAY` 为真时抑制（A14 `U/ClipboardListener.android-14.0.0_r1.java:118-131`，A15 `125-137`，A16 `175-187`；A16 另外多了一个 `ClipboardOverlaySuppressionController` 分支，`138-147`）。候选写剪贴板时来源是 `com.android.shell`（`ClipboardAccess.kt:109-122,206`）。【源码】

- **O1 抑制只对来源 shell 生效，模拟器上无条件生效。** A14 用 `ro.boot.qemu` 判断，A15/16 用 `Build.IS_EMULATOR`。用户限定只用 MuMu 和 AS 模拟器验证，而在模拟器上，即便来源写错也会被抑制，所以真机上的来源条件在验证里测不出来。**需条件**：增加一条单元测试，断言 setPrimaryClip 的包名参数就是 `com.android.shell`；报告里写明"模拟器上抑制无条件生效"。
- **O2 恢复原剪贴板。** 恢复时调用的是同一个 `setPrimaryClip`（`ClipboardAccess.kt:107`），所以会再触发一次 SystemUI 回调。如果不给原内容加标记，浮层会把真人的原剪贴内容（可能是密码）弹到主屏上；如果加了标记，`ClipDescription.getExtras()` 里会多出一个 SystemUI 键，读剪贴板描述的 App 能看到。实施方明确不修改原内容的 description（`inputs.md:89-93`），用户决定写的是"关掉浮层"，两者冲突，需要用户二选一。另外，恢复本身已经把剪贴板的来源从原 App 改成了 shell，这是现状就有的可观察变化。**需条件。**
- **O3 关不掉的"已从剪贴板粘贴"toast。** 目标 App 粘贴时会读剪贴板，读者 UID 不等于剪贴板主人（shell），于是 ClipboardService 给真人弹出 "<App> pasted from your clipboard"（A14 `U/ClipboardService.android-14.0.0_r1.java:1362-1420`；A15 `1439`；A16 `1436`）。浮层标记控制不了它。要关只能全局改 `clipboard_show_access_notifications` 这项隐私设置，不应该这样做。**需条件**：文档里不能承诺"借贴不打扰真人"。
- **O4 OEM。** 三星、小米等的剪贴板 UI 和输入法的剪贴板建议条不读这个 extra。**轻微**【推断】。

### 2.16 等窗口消失而它已不在：等"消失"算成功，等其它条件立刻报错

**上游**：无障碍窗口列表会丢掉**完全被覆盖且未聚焦**的窗口（`windowMattersToAccessibility`：`unaccountedSpace.quickReject(regionInScreen)` 时返回 false，`U/AccessibilityController.android-14.0.0_r1.java:1680-1700`）；最近任务动画期间被忽略的窗口也会被丢掉（同处 1682）。窗口 ID 不变，遮挡消失后会回来。【源码】

- **P1 "其它条件立刻报错"会误报。** 例如等窗口 W 里出现某段文字时，App 弹出一个全屏插页广告、全屏的"给我们评分"页、来电全屏界面，或者执行最近任务手势，W 会暂时从列表中消失。新语义下会立刻判失败，而几秒后 W 其实会回来。**需条件**：非 gone 条件遇到缺席时继续等到截止，回执标注"窗口缺席，可能被遮挡"；或者只有当窗口所属的任务也不在了才立刻失败。
- **P2 "消失"误判成功。** 同样的遮挡会让 `wait --gone --window W` 提前成功，调用者以为对话框关了，实际只是被盖住。**需条件**：gone 的成功要求连续两次采样都缺席，并附说明。
- **P3 与动作后检查统一。** 动作后检查把缺席判为"有变化"（`UiPostAction.kt:43-44,97-110`），被盖住确实也算变化，这一侧没问题。统一成一张解释表时，不要把 P1 的"立刻失败"反向套到动作后检查上。**轻微。**

### 2.17 提交历史：先做到最终形态再切片

- **Q1 中间提交没有设备验收。** 模拟器的验收证据只对应最终树。从最终树往回切出来的中间提交从来没有以独立形态在设备上跑过，"每个提交可构建、可测试"只能由主机 pytest、Kotlin 单元测试和 Gradle 构建来证明。每个提交都跑一遍 Android 构建（含 NDK）成本很高。**需条件**：中间提交只声明"构建和单元测试通过"，设备验收只挂在最终提交上，并写明最终提交与验收用的树逐字节相同。
- **Q2 跨语言契约与帮助导出的一致性。** 每个切片都要同时带上 Kotlin、Python 和 `contract/wire-contract.json`、skills 文档的对应改动，否则契约测试会在中间提交失败。评审方自己报告过"六块拆出后有五处静默错误"（主报告 §8.9）。**需条件**：用脚本逐个提交跑契约测试和帮助导出测试。【推断】

### 2.18 其它一并同意的小项

| 小项 | 攻击 | 程度 |
| --- | --- | --- |
| `type --via keys\|paste\|editor` | `editor` 是 IC 通道，只能用于主屏（`displays.md:68-70`），只写入当前聚焦的编辑器。`type REF --via editor` 在 REF 不是当前焦点时，会写到别的框里。`paste` 要求节点暴露 ACTION_PASTE（`UiActions.kt:262`）。三种方式的前提条件和回执都不同，合到一个动词下，非法组合的矩阵变大 | **需条件**：`--via editor` 带 REF 时，REF 必须是焦点，否则拒绝 |
| 通知操作拒绝作用于自身通知 | 自身包应当是 App 包 `com.rainng.androidctl`，不能写成 worker 的 `com.android.shell`；否则会误拒 shell 发出的 bugreport 等通知 | 查过没问题（实现时注意） |
| 点击未发出时跳过事后检查 | 只有第一次调用明确 mismatch，或确知一次都没尝试，才是 NOT_SENT（`PointerGesture`，见 `inputs.md:256-262` 转述；`AppInput.kt:96-110`）。超时或异常仍是 UNKNOWN，应照常采样。只要按 NOT_SENT 判断就没问题 | 查过没问题 |
| 变化只给一个结论 | fill 的局部核值和树的签名矛盾时，需要有优先级规则；JSON 细节保留，所以不丢信息 | 轻微 |
| 采样最小间隔 100 ms | 只会让检测延迟最多 100 ms | 查过没问题 |
| 建屏前先检查再结束旧会话 | 只能减少、不能保证：静态检查通过后，`createVirtualDisplay` 仍可能失败，显示状态也可能不是 ON（`TargetManagement.kt:103-107`、`WorkDisplay.kt:70-77`）；检查到创建之间屏幕也可能被锁。文档只能写"减少" | 轻微 |
| 结果与退出码统一 | 落点未确认时如果把结果改成 failed 或 not sent，调用者会重试 open。对结构化的 ACTION_SEND 来说，这会造成重复撰写甚至重复发送。应该统一为 UNKNOWN 加非零退出码，不能是 failed | **需条件** |
| 动作可按名字调用 | 见 H4 | 轻微 |

---

### 2.19 坐标点击默认接收者（C）的接入风险、退路链与拒绝规则

方向已经定了，这里不讨论方向，只列风险。

**(a) 把 WindowInfosListener 放进常驻 worker 的具体风险**

1. **SF 的 ack 背压（新发现，高风险）。** `WindowInfosListenerInvoker::windowInfosChanged` 在 `mUnackedState` 非空、且不是强制调用时，只把更新存进 `mDelayedUpdate`，不发给任何 listener（A14 `U/WindowInfosListenerInvoker.cpp.a14:100`；A15 `CWE/android-15.0.0_r1/WindowInfosListenerInvoker.cpp:98-108`；A16 `U/...cpp.a16:104`）。InputDispatcher 本身就是这组 listener 之一（`T/research/input/upstream/native14.txt:740-742`；A17 `a17-InputDispatcher.cpp:935-938`）。本进程的 reporter 要先同步调用本地 listener，再调用 `ackWindowInfosReceived`（`CWE/android-15.0.0_r1/WindowInfosListenerReporter.cpp:91-110`），而 JNI 回调又同步调用 Java 的 `onWindowInfosChanged`（`android_window_WindowInfosListener.cpp:95-120`）。所以 worker 的回调只要阻塞（锁竞争、GC、等 UI 执行器、binder 线程被占满），就会延迟全系统的非强制窗口几何更新，例如动画中触摸区域的变化。真人在主屏上的触摸可能按旧几何被分发。这个窗口一直持续到我们 ack、出现强制更新，或者 worker 死亡（死亡时才会自动 ack，`WindowInfosListenerInvoker.cpp:64-85`）。实施方的 inputs.md 和 review.md 都没有讨论这一点（grep `ack`、`unacked`、`mDelayedUpdate` 均无命中）。【源码】**最小条件**：Java 回调只做一次 `AtomicReference.set`，不加锁、不分配大对象；只在单条坐标命令执行期间注册（实施方的"有限 C"本来就是这个方向），不做常驻监听。
2. **回调线程与 binder 线程池。** 回调跑在 worker 的 binder 线程上。worker 的 RPC 是同步阻塞的，`execute` 一直等到 `run.finished`（`UiBackend.kt:144-145`、`TargetManagement.kt:33-35`）。并发请求多时，SF 的 oneway 回调会排队，ack 也跟着延迟（见上条）。**需条件**【推断】。
3. **R8 与子类化。** release 构建开了 `isMinifyEnabled = true`（`android/app/build.gradle.kts:166-171`）。父类 `android.window.WindowInfosListener` 不在 SDK 的 android.jar 里，R8 不知道 `onWindowInfosChanged` 是对库方法的覆盖，可能把它当成无用方法删掉或改名。之后 JNI 按父类的 methodID 虚调用会抛 `AbstractMethodError`，但 JNI 层只记日志并清除异常（`android_window_WindowInfosListener.cpp:115-119`；A16 同处 116），结果是**静默地**再也收不到数据。debug 构建不做混淆，测不出这个问题；实施方的探针是独立的 d8 jar，没有经过 R8（`coordinate-window/build/`）。**需条件**：加精确的 keep 规则，并在 release 构建上验收（实施方也已指出需要验证）。
4. **弱引用。** native 层只持有 Java listener 的弱全局引用（`NewWeakGlobalRef`，同文件 93；A16 94）；Java 对象一旦被回收，回调就静默停止。worker 必须一直强引用这个对象，直到 unregister 返回。**需条件**【源码】。
5. **注册返回的缓存为空。** `register()` 返回的是本进程最后一次收到的缓存，首次注册时为空（`CWE/android-14.0.0_r1/WindowInfosListener.java:50-62`）；原件中 initial 是 0 个窗口、0 个 display（`coordinate-window/runtime-api34/commands/probe-once/stdout.bin`）【原件】。SF 在注册时会设置 `eInputInfoUpdateNeeded`（`CWE/android-15.0.0_r1/SurfaceFlinger.cpp:9159-9164`），所以通常很快会有第一次回调，原件中约 10 ms（`callbackDeadline − 3000` 到回调时间点）【原件，重算自 stdout 字段】。按单条命令注册就必须等这第一次回调。
6. **注销与注册都无法确认成功。** JNI 的 `nativeRegister` 和 `nativeUnregister` 都忽略状态码（`android_window_WindowInfosListener.cpp:137-157`）；reporter 中远端移除失败时不会把 listener 从集合中删掉（`WindowInfosListenerReporter.cpp:64-87`）。权限被拒和"已注册但还没有数据"从 Java 侧看起来一样。实施方已指出。【源码】
7. **权限。** 需要 `ACCESS_SURFACE_FLINGER`，或调用者是 SYSTEM/GRAPHICS（`SurfaceFlinger.cpp` A14 9077-9104、A15 10432-10460、A17 10717-10744）。代码里有 TODO(b/270566761) 计划收紧到 "system_server and shell"。shell 目前有这个权限，今后也在保留之列。**轻微。**
8. **字段差异与坐标变换。** `InputWindowHandle.frame` 在 A14 是四个 int，A15/A17 是 Rect（实施方已核）；`DisplayInfo` 在 A17 增加了公开构造器。A14、A15、A16、A17 中 `WindowInfosListener` 的抽象方法和 register/unregister 签名一致（`U/WindowInfosListener.java.a16:32,48,61,68`）。触摸区域要经过 `DisplayInfo.mTransform` 换算，不能直接用窗口自己的 transform（原件中 IME 窗口的 Y 平移是 −48，但它的区域本来就是屏幕坐标）。旋转、`wm size` 和厂商的分辨率切换都会让 display transform 不再是单位矩阵。**需条件**：先只支持单位矩阵，其它情况在发送前拒绝，并写进文档。
9. **inputConfig 的解释。** 原件里有 `ActivityRecordInputSink`，ownerUid 1000，区域是 `[-7199,-12799,7200,12800]`，inputConfig=5【原件】。如果没有正确跳过 NOT_VISIBLE、NOT_TOUCHABLE、SPY 这些位，预测就会落到 system UID，变成误拒。**需条件**（实施方的有限 C 规则已经列出）。
10. **持续分配。** 只要注册着，SF 每次输入信息变化（动画期间可能每帧都变）都会在 worker 里分配整组 `InputWindowHandle[]`（`fromWindowInfos`），增加 GC 压力，而 GC 停顿又会反过来拖慢第 1 条里的 ack。这是又一个不做常驻监听的理由。【推断】

**(b) 退路链"内部接口 → 已采集窗口列表 → 不限制"**

- **第 1 级失败能否判定。** 类不存在、链接失败会抛异常，可以判定。权限被拒或回调被 R8 删掉时 Java 侧不报错，只能靠"注册后一定时间内没收到回调"来判定，按单条命令注册时可以做到（5 号所述的首次回调）。常驻注册时，数据陈旧和"界面没变"无法区分。**需条件**：每条命令都注册，并设首次回调期限，超时才降到下一级。
- **第 2 级（无障碍窗口列表）可行，但有缺口。** 完全被覆盖的窗口、不可触摸的窗口不在列表里（`AccessibilityController:1680-1700`），正好与命中判断的需要一致。缺口是：列表里没有 UID，要读根节点包名再换算；评审方指出 IME 窗口目前是 `content: notRead`；嵌入式跨 UID 的内容不是独立的无障碍窗口，会按宿主 UID 预测，然后被平台拒绝。这样是安全的，只是能力回到现状 B。**查过没问题**（安全方面）。
- **第 3 级"不限制"（硬反例）。** 不限制就是不定向注入，平台不再校验 UID。现在"坐标输入不能落到 AndroidCtl 自己的界面"这条护栏，靠的正是定向 UID 加 `inputUid` 拒绝自身 UID（`AppInput.kt:78-88`），以及活动根节点包名不是自身包（`UiObservation.kt:81-87`）。在分屏、自由窗口或部分覆盖时，自身窗口可能不是活动根窗口，却出现在点击位置上；这时不定向点击会落到管理界面上，例如强制退出执行器、续签证书、撤销配对。被提示注入的 agent 可以借此自我授权。而且"不限制"恰好在"连窗口列表都读不到"、对现场知道得最少的时候启用，保护方向是反的。**硬反例**。最小修补：第 3 级改为"拒绝，并提示用 `--expect`"。如果一定要保留不限制，至少在目标屏上出现任何自身窗口时拒绝，并且只用于调用者显式要求的那一次。
- **切换条件要按命令判定、写进回执。** 每次回执写明用了哪一级（`receiverSource: windowInfos|axWindows|none`），否则调用者无法解释误拒。**需条件。**

**(c) "预测不出时拒绝并让调用者手动指定，不自动重发"**

- **漏洞 1：手动指定需要知道包名。** 调用者要用 `--expect PACKAGE` 点软键盘时，必须知道 IME 的包名，而当前观察里 IME 窗口的内容是 notRead，没有包名（评审方转述）。结果是"拒绝后让调用者指定"走进死路。**需条件**：拒绝的回执里给出点击位置所在窗口的包名（来自窗口列表或 `Settings.Secure.default_input_method`）。
- **漏洞 2：部分发出。** 多指手势中，第二个指头按下时 split 到不同 UID，或者 slippery 窗口在移动中换了窗口，都会在中途被拒；这时首个按下已经发出，不属于"预测不出"，规则没有覆盖。现执行器会给 UNKNOWN、记录已发出的前缀并发 CANCEL（实施方 `inputs.md:256-262`），只要不改成 NOT_SENT，就不会被误当作可以重发。**需条件**：规则文本限定在"首个按下之前"。
- **漏洞 3：预测成功但窗口变了，同 UID。** 平台只核 UID，不核具体窗口，点到同一 App 的另一个弹窗不会被拒。这是坐标点击固有的问题，不是新漏洞。**轻微。**
- 自动重发：现执行器没有重发路径。**查过没问题。**

---

## 3. 需要设备才能确认的攻击

| # | 攻击 | 判别方法 |
| --- | --- | --- |
| S1 | 2.19(a)1 ack 背压 | 在 worker 回调里人为 sleep 500 ms，同时让主屏执行窗口动画，用 `dumpsys SurfaceFlinger` 看 WindowInfosListenerInvoker 的 debug 信息（`getDebugInfo` 里的 maxSendDelay 和 pendingMessageCount）是否升高 |
| S2 | 2.19(a)3 R8 删除回调覆盖 | 用 release 构建接线后，看是否持续收到回调；或者用 dexdump 检查子类里是否还有 `onWindowInfosChanged` |
| S3 | 2.7 G1 通知 open 搬走主屏任务 | 主屏打开某聊天 App，发一条通知，在后台屏会话里带 launchDisplayId 执行 open，前后各读一次 `am stack list` |
| S4 | 2.4 D2 `--multiple-task` 之后 App 内跳转搬走任务 | 用 VLC 或其它带 singleTask 内部页的 App 复现 |
| S5 | 2.3 C1/C2 TWA、WebAPK 和首启权限框判为未确认 | 各启动一次，看任务的 base 和 top，以及窗口列表 |
| S6 | 2.15 O2/O3 恢复时的浮层和粘贴 toast | 只能在真机上判别来源条件；模拟器上无条件抑制，结果没有判别力 |
| S7 | 2.16 P1 遮挡导致窗口缺席 | 弹一个全屏 Activity 盖住对话框窗口，看无障碍窗口列表里 ID 是否消失后又回来 |
| S8 | 2.14 N1–N4 无障碍选区 | Contacts 上对未聚焦的框执行 SET_SELECTION 后 type，看写到哪个框；看是否出现工具条；看重复设置同一选区的返回值 |
| S9 | 2.5 E1 死亡回调累积 | 连续执行 N 条通知命令，比较 `dumpsys meminfo system_server` 的 JNI 全局引用数（或 `dumpsys activity` 里的 binder proxy 计数）的增长 |
| S10 | 2.6 F1 聊天通知 | 在模拟器短信 App 上连发两条消息，确认通知的 key 和标题不变、`when` 和 postTime 变化 |

## 4. 未核实与局限

- 没有设备、没有构建、没有运行任何包内脚本。上游源码是 `android.googlesource.com` 的 r1 标签，不是运行镜像逐字的版本；厂商 ROM 没有覆盖。
- 本轮新下载并放在 `scratchpad/agents5/23/up/` 的文件：ManagedServices、PendingIntentRecord、SafeActivityOptions、ClipboardService、ClipDescription、BackgroundActivityStartController、SystemUI ClipboardListener（以上均为 A14/15/16）；NotificationManagerService（A14/16）；ActivityStarter、TextView、android_util_Binder.cpp、AccessibilityController、AccessibilityWindowManager、AccessibilityWindowsPopulator（以上均为 A14）；WindowInfosListener.java、android_window_WindowInfosListener.cpp（A16）；WindowInfosListenerInvoker.cpp（A14、A16）。没有下载 `ActivityTaskSupervisor.isCallerAllowedToLaunchOnDisplay`，2.7 中"权限会通过"是凭 typed open 的先例推断的；A15 的 ActivityStarter 没有下载，2.4 和 2.7 中的 reparent 逻辑只核了 A14。
- 2.5 E1 中"全局引用超限会中止进程"是推断，我没有核 ART 全局引用表的上限常数。
- 2.10 J2 的"名字变长"需要用评审方的 Chrome 原件重算。按规则我没有读 sub-report 19，不知道它的重算口径。
- 2.3 C1（TWA、WebAPK 的任务结构）和 2.6 F1（聊天 App 的标题与 `when` 行为）是对第三方 App 的推断。
- 我没有逐字核对评审方所说"IME 窗口 content notRead"的原件，2.19(c) 漏洞 1 依赖这个转述。
