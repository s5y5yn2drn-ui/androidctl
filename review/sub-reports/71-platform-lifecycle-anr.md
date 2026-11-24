# 专题核查：平台与权限、Gateway／Shizuku／最小无障碍服务、worker 与资源生命周期、通知只读、System UI 无响应事件

核查对象：产品 `fb75afc`（与被实测的 `a27fe5aa` 树相同），对照主分支基线 `7d5be70` 和第五轮候选 `cab84b8`。只读，没有设备、没有构建。

路径约定：`SRC` 为产品只读克隆；`PKG` 为实施方本轮材料包；`ST` 指 `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`；`U35` 目录指 `ST/runtime/api361-u35`（实施方在 Android Studio 自带模拟器、Android 16／API 36.1 带 Google Play 镜像上做的诊断与 Maps 对照）；`api361-play` 目录指 `ST/runtime/api361-play`（同一种镜像的旧实例，上一阶段使用）。我写的统计脚本与输出在 `S/agents9/71/sysui_stats.py`、`S/agents9/71/sysui_stats.out`（`S` 为评审 scratchpad），下载的 AOSP 文件在 `S/agents9/71/aosp/`。

---

## 1. 结论摘要

**整体状态**：平台层的形态是对的，而且比基线干净。Android 14 下限确实带来了简化：版本分支从约 14 处降到 3 处，节点地址相关的隐藏常量删了 6 个。资源归属清楚，异常路径基本都能释放。System UI 无响应（ANR）这件事，证据足以支持一个比实施方更明确的判断：**这是低配模拟器冷启动后约一分钟内 CPU 饥饿造成的，AndroidCtl 运行时不是必要条件**，产品不需要为它改设计。本专题我只找到一处应在合入前修掉的回归，改动很小。

最重要的发现：

1. **【必须修，小】UiAutomation 登记冲突会让 worker 的 UI 能力卡死，直到有人在设备上手动停止 worker。** 如果在 AndroidCtl 连接 UiAutomation 的那一刻，另一方（比如 `uiautomator dump`、Appium 或别的自动化工具）正持有登记，平台会抛出 "already registered"。这时 `UiAutomation` 的内部状态停在"连接中"，随后 `disconnect()` 必然抛错。候选把这种情况记成"注销未确认"，而且设计上不重试（`AutomationCleanup.kt:32-50`，测试 `UiCleanupTest.kt:26-34` 把"不重试"写死了）。结果是之后每条 UI 命令都会被 `requireCleanUi` 拒绝（`Sessions.kt:250-252`、`UiBackend.kt:119`），提示要求用户去设备上点"Stop idle executor"。基线专门处理过这个情况（`7d5be70:UiBackend.kt:419-434` 会退而调用 `UiAutomationConnection.disconnect()`，并把 "Already disconnected." 视为什么都没登记），候选删掉了对应的隐藏成员 `disconnectConnection`，也就丢了这层处理。依据是源码和 AOSP 16，没有上设备验证。
2. **System UI ANR 的归因（重点）**：一共 4 次 System UI ANR，全部发生在 2 核配置，而且都在冷启动后约 50 秒内。基线和候选都出现过（旧实例 2 GiB／2 核时，候选首次准备 1 次、基线第 1 次 1 次；新实例上候选 2 次）。4 核配置下两边都没有出现：诊断 1 次加正式 10 次，候选 6 次、基线 5 次全部为 0。两份 ANR 栈里，System UI 主线程累计只执行了约 1.1–1.4 秒，却在 CPU 队列里等了 18–23 秒；System UI 的 70 个线程中没有一个在处理无障碍查询。4 GiB／2 核那次 ANR 发生在任何 UiAutomation 登记之前。另外，10 次 4 核正式运行里每次都有约 10 分钟的密集操作，System UI 没有记下任何一次 30 帧以上的掉帧，也没有超过 700 ms 的长帧。
3. **候选默认不采集 System UI 的窗口，基线也一样。** 默认范围只读当前活动的非输入法窗口（`UiScope.kt:64-67, 79-100`）。窗口列表由 system_server 提供，不占用 System UI 主线程。只有通知栏或 System UI 对话框成为活动窗口时，才会去读 System UI。
4. **候选确实给 System UI 增加了一项基线没有的负载：前台通知会动态更新。** 通知正文包含每条命令都会变化的资源计数，状态每 2 秒轮询一次（`GatewayService.kt:82-102, 115-129, 451`）。正式运行中，每次运行 AndroidCtl 发出 8–29 次通知更新，基线每次只有 1 次（我离线重算的结果）。在 2 GiB 那次 ANR 的 5 秒窗口里，这类更新有 3 次。量不大，不是根因，但它是一项可以去掉的系统负载。
5. **通知只读**：范围小，读取正确，只在某个租约第一次调用通知命令时登记一个系统级监听，租约结束时注销，同一个 Binder 对象跨租约复用。实施方承认的"Android 15 以上死亡回调与 native 释放未知"，风险可以忽略：AOSP 14 和 16 的 `ManagedServices` 在注销时都不会 `unlinkToDeath`，所以每个用过通知命令的租约会在 system_server 里留下一个很小的死亡回调对象，worker 退出时一并清掉。公开指引里有一处"full mode"，这个说法已经没有对应的定义。
6. **隐藏接口**：`HiddenApi` 里的惰性成员数，基线 51、上一轮 62、本轮 62，本轮没有变化。另外在 `HiddenApi` 之外，剪贴板约有 9 处反射、显示与任务约有 20 处反射读取常量和字段。最脆弱的是私有字段 `UiAutomation.mRemoteCallbackThread` 和 12 个参数的 `startActivityAsUser`。

**是否构成合入障碍**：第 1 条应在合入前修掉，改动约 10–20 行，风险低。其余都不构成障碍。ANR 这件事不需要改产品；我建议顺手把前台通知里的资源计数去掉（可选）。

---

## 2. 确定缺陷

### D1【必须修，小】UiAutomation 登记被占用时，worker 的 UI 能力永久卡死

- **触发条件**：worker 第一次连接 UiAutomation 时，系统里已经有另一个 UiAutomation 登记。可能来源有：`adb shell uiautomator dump`、Appium 或 UIAutomator2 服务、其他测试工具，或者调用者自己通过 `androidctl shell` 运行 `uiautomator dump`，并且恰好与某个租约的第一条 UI 命令重叠。
- **平台行为**【源码，AOSP android-16.0.0_r1】：
  - `UiAutomationManager.registerUiTestAutomationServiceLocked` 发现已有登记时，会抛出 `IllegalStateException("...already registered!")`（我核对的下载文件第 108-111 行）。
  - `UiAutomation.connectWithTimeout` 只捕获 `RemoteException`（UiAutomation.java 第 384-397 行），所以这个异常会直接抛出，`mConnectionState` 停在 `CONNECTING`。
  - 之后调用 `disconnect()`，第一步就会抛出 `IllegalStateException("Cannot call disconnect() while connecting")`（第 438-445 行）。这时系统里其实没有任何属于 AndroidCtl 的登记。
- **候选行为**【源码】：
  - `UiBackend.registerAutomation` 先调用 `cleanup.registrationStarted()`，再连接（`UiBackend.kt:533-534`）。连接失败后进入 `discardConnection`，再到 `AutomationCleanup.close`。
  - `close` 中调用 `disconnect` 时抛错，于是记下 "UiAutomation registration release is unconfirmed"，并把 `disconnectAttempted` 置为 true，以后不再尝试（`AutomationCleanup.kt:32-41`）。
  - 结果 `registrationPending` 永远为 true，`PendingUiCleanup` 永远不完成，`ui.cleanupFailure` 一直不为空。之后所有 UI 命令（包括通知、剪贴板、截图）都在 `UiBackend.kt:119` 被 `requireCleanUi` 拒绝，返回 `uiCleanup`，提示为 "ask the user to use Stop idle executor on the device"（`Sessions.kt:399-400`）。
  - 单元测试 `UiCleanupTest.kt:26-34` 断言的正是"失败后第二次 close 不再调用 disconnect"。这个测试只是在复述实现，并没有覆盖平台的这条真实路径。
- **与基线比较**：基线 `7d5be70:UiBackend.kt:419-434` 在 `disconnect()` 抛出 `IllegalStateException` 且尚未取得连接时，改为调用 `UiAutomationConnection.disconnect()`。如果返回 "Already disconnected."，就说明本地连接对象从来没有登记成功，按已清理处理。这个判断依据的是本地状态（`mClient` 只在登记返回后才赋值），是确定的。候选在缩减隐藏成员时删掉了 `disconnectConnection`（在 `8d0416c` 整理提交里），这个分支也随之消失。
- **实际影响**：对远程 LLM 调用者来说，这是一次硬停：所有 UI 命令都失败，只能请人去设备上操作。实测材料里没有出现过这种冲突，准备脚本里的原生 dump 和产品登记都是先后执行的。所以这是源码推断，没有设备复现。
- **最小修正**：恢复基线的退路，再补一个只读判定。连接抛出非超时异常时，如果 `UiAutomation` 还在"连接中"，直接调用 `UiAutomationConnection.disconnect()`；把 "Already disconnected." 视为没有登记，标记 `registrationPending=false`，并结束回调线程。另一种做法是：连接抛出 `IllegalStateException` 时根本不进入"需要注销"的状态。两种都约 10–20 行。错误码应如实写成"另一个自动化工具占用了 UiAutomation"，不要写成需要人工停止 worker。
- **确定程度**：平台行为和候选控制流都来自源码，确定程度高；设备上的表现【需设备】。

### D2【可选】前台通知随每条命令变化，给 System UI 增加不必要的负载

- 通知正文包含当前租约、目标屏以及 `UI: n / Shell: n / File: n` 资源计数（`GatewayService.kt:82-102`，字符串 `session_resources`）。状态每 2 秒轮询一次，内容变化就调用 `notify`（`GatewayService.kt:33-43, 115-129, 451`）。基线的前台通知是静态的。
- 【重算】我统计了 U35 新实例连续日志中 `notification_enqueue ... com.rainng.androidctl` 的次数（脚本 `S/agents9/71/sysui_stats.py`）：候选正式运行每次 8–29 次（中位数约 19），基线每次 1 次。2 GiB 那次 ANR 窗口内，候选在 43.958、46.976、48.375 和 52.111 秒各有一次（`U35/logs/u35-c1-candidate-01-retry1/logcat-all.log` 第 16654、17439、17980、19058 行）。
- 影响：每次更新，System UI 的通知管线都要在主线程上做一部分处理，同时系统还要把它分发给所有通知监听者。正式运行中没有观察到它造成掉帧（见 §4），所以只是可去除的负载，不是缺陷根因。
- 最小修正：去掉资源计数，只在租约持有者或目标屏变化时更新。README 第 86 行公开描述了通知会显示租约，这一部分保留。

### D3【可选】指引里有"full mode"这个未定义的说法

`skills/androidctl/notifications.md:17` 写着 "Both need full mode"。产品代码和其他指引里都没有权限模式（`git grep` 只在这一处出现，`cab84b8` 就有）。调用者可能据此去找一个并不存在的开关。改成"需要当前租约和 user0"即可。

### D4【可选】两处上一轮已经指出、本轮没处理的小反射

- `NotificationOperations.kt:135` 通过反射调用 `UserHandle.getIdentifier`。这是公开 API，可以直接调用。上一轮记录 `51-notifications-images.md` 的 D4 已经指出。
- `ClipboardAccess.kt:58-206` 有约 9 处反射，没有收进 `HiddenApi`，所以隐藏接口清单仍不是唯一的。上一轮记录 `57-scale-maintainability.md` 第 140 行已经建议移进去。

---

## 3. 运行结构、资源所有权与设计判断

### 3.1 进程与组件（谁启动谁）【源码】

| 组件 | 进程／身份 | 由谁启动 | 职责 |
| --- | --- | --- | --- |
| 管理界面 `MainActivity` | App 进程，普通应用 UID | 用户 | 授权 Shizuku、开关 Gateway、显示和关闭配对码、撤销配对、释放会话、停止或强制结束 worker、电池与浏览器支持入口。窗口设置了 `FLAG_SECURE`（`MainActivity.kt:93`，本轮新增），每 2 秒刷新本地状态。 |
| `GatewayService` | App 进程，前台服务（connectedDevice 类型） | 用户在界面上打开 Gateway（`startForegroundService`） | 在 `0.0.0.0:18843` 上监听 TLS WebSocket（`NetworkGateway.kt:52`，与基线相同）；按 Bearer 凭据认证、处理配对；通过 `WorkerConnection` 绑定 worker；每 2 秒向 worker 查询状态；维护前台通知。 |
| worker `WorkerService` | Shizuku 用户服务，以 shell UID 2000 运行，`daemon(true)` | Gateway 通过 `Shizuku.bindUserService` 启动（`WorkerConnection.kt:50-57, 179`） | 持有会话和租约、UI 后端（UiAutomation）、shell 作业、文件作业。只接受本 App UID 的调用（`WorkerService.kt:178`）。 |
| 最小无障碍服务 `BrowserAccessibilityService` | App 进程 | 用户在系统无障碍设置里开启，Gateway 不控制 | 两个空回调；`canRetrieveWindowContent=false`；作用只是让 Chromium 类浏览器认为无障碍已开启，从而生成网页内容树。worker 用 `FLAG_DONT_SUPPRESS_ACCESSIBILITY_SERVICES` 连接，避免把它压制掉（`UiBackend.kt:534`）。 |
| 主机 daemon | 主机上的用户进程 | CLI 按需启动 | 每台设备一条修改通道和一条查询通道；连续约 5 分钟空闲后退出。 |
| CLI | 主机 | 调用者 | 解析参数、选择设备和会话、输出结果。 |

**谁持有 UiAutomation**：worker 的 `UiBackend`。它在某个租约第一条真正的 UI 命令时才连接（`UiBackend.kt:475-500`），租约结束时由 `clear(lease)` 断开（`UiBackend.kt:567-578`）。所以每个用到 UI 的租约只有一次 flags=0x1 的登记。U35 每次正式运行的日志里恰好只有 1 次 flags=0x1，其余几次 flags=0x0 来自准备脚本的原生 `uiautomator dump`【重算】。

**通知监听**：`NotificationOperations` 在 worker 生命周期内只创建一个 `NotificationAccess`（同一个 Binder 对象），在某个租约第一次调用 `notifications list/get` 时登记（`NotificationOperations.kt:28-71`），租约结束时在 UI 线程的 `clear` 的 finally 中注销（`UiBackend.kt:571-577`，`NotificationOperations.kt:73-84`）。登记走的是 `INotificationManager.registerListener`：shell 拥有 `STATUS_BAR_SERVICE` 权限，可以通过 `enforceSystemOrSystemUI` 检查（AOSP 16 `NotificationManagerService` 第 5629-5633、6443-6447 行）。监听以"系统服务"身份登记，组件名伪装成 `com.android.shell/...NotificationListener`。登记期间，系统会把所有通知的发布和移除回调单向发给 worker，worker 直接丢弃。docs `architecture.md:98` 已披露：登记期间会收到所有用户的通知，并让 shell 获得通知附件 URI 的读取授权。

**worker 死亡**：App 一侧的 `DeathRecipient` 进入 `lost()`，清空连接并记下 "Worker died"（`WorkerConnection.kt:82-88, 119-136`）。之后主机命令返回 `workerUnavailable`。**不会自动重新绑定**，要等用户在设备上点"Refresh status"（`GatewayService.kt:199-209`），或者 Shizuku 重启时触发重新绑定。这一点与基线相同。worker 死后，system_server 通过 Binder 死亡通知自动清掉它的 UiAutomation 登记和通知监听；后台虚拟屏也会被释放，因为建屏时带了 `DESTROY_CONTENT_ON_REMOVAL`，屏上的任务会被销毁（指引里已披露"退出直接销屏"）。

**system_server 重启**：worker 在构造时就对 `activity` 服务做了 `linkToDeath`，回调里直接 `exitProcess(0)`（`WorkerService.kt:20-26`）。这个设计干净。

### 3.2 隐藏接口清点（`HiddenApi.kt` 中的惰性成员；不含 10 个只用于 `Class.forName` 的类型句柄）

| 用途 | 基线 `7d5be70` | `cab84b8` | `fb75afc` | 说明 |
| --- | --- | --- | --- | --- |
| 采集（节点、窗口、映射） | 21 | 15 | 15 | 少的 6 个是节点地址和 leash 常量，以及旧版本的 `NodeAccess` 路径。手写的节点查询回调（`findNode` 和两个回调事务号）仍在，这一点不属于本专题。 |
| 节点动作 | 2 | 2 | 2 | `performAccessibilityAction` 和回调事务号。 |
| UiAutomation 生命周期 | 9 | 6 | 6 | 删了旧版 `connect`、`disconnectConnection`（删后者导致 D1）。 |
| 输入注入 | 2 | 3 | 3 | 改为 `injectInputEventToTarget` 加 `setDisplayId`。 |
| 显示、任务、启动、截图 | 3 | 13 | 13 | 后台屏、任务事实、意图解析、`captureDisplay` 链。 |
| 通知 | 0 | 7 | 7 | 只读。 |
| 锁屏、用户、服务 | 7 | 6 | 6 | |
| **合计** | **44**（含类型句柄共 51） | **52**（62） | **52**（62） | 本轮无增减。 |
| `HiddenApi` 之外 | 约 6 处动态反射 | 剪贴板约 9 处，VD／Display 标志常量 10 个，TaskInfo 字段约 10 个 | 同左 | 剪贴板反射没有收进清单（D4）。 |

【源码】按 `git show 版本:HiddenApi.kt` 和 `git grep` 统计；与上一轮记录 57 的 51／62 一致。

**Android 14／15／16 稳定性**（AOSP 已下载文件核对的部分标【源码】，其余为【推断】）：

- 采集与动作：`IAccessibilityServiceConnection` 里的 `findAccessibilityNodeInfoByAccessibilityId`、`performAccessibilityAction`、`getWindows`、`getMagnificationConfig`，在 14、15、16 的 AIDL 中签名完全相同【源码，`S/agents6/26/aosp/IASC-*.aidl`】。风险中等：这部分接口在活跃演进，但到 16 为止都稳定。
- UiAutomation：`UiAutomation(Looper, IUiAutomationConnection)` 在 14、15、16 都在，16 上标了 `@Deprecated`；`connectWithTimeout`、`disconnect`、`getCache`（15 起）和私有字段 `mRemoteCallbackThread` 在 14、15、16 都在【源码】。**私有字段反射是整张表里最脆弱的一项**；而它的用途只是在注销后 join 平台内部线程以"证明回调已停止"，其实 `disconnect()` 自己就会 quit 这个线程（AOSP 16 第 458-461 行）。见 §3.4。
- 显示与任务：`startActivityAsUser` 有 12 个参数，`getTasks` 是 14 引入的 4 参数形式。这些在 14–16 上实际运行过【原件，三种模拟器】；参数个数是最容易在 17 上变化的地方【推断】。
- 通知：`registerListener`、`getActiveNotificationsFromListener` 在 14–16 稳定【源码，NMS 16】。
- 剪贴板：`IClipboard` 方法在 13、14 都改过签名（加了 attributionTag 和 deviceId），是历史上漂移最多的一类【推断】。

**Android 14 下限带来的简化是否兑现**：兑现了。代码里的 `SDK_INT` 分支从基线约 14 处降到 3 处（一处按 15 区分缓存入口，两处按 16 读三态勾选框）【源码】；节点地址常量、`NodeAccess`、旧版 `connect` 都删掉了。但有一点不能简化：**`AccessibilityWindowInfo` 在 14 和 16 上仍然走对象池**，`recycle()` 会先 `clear()` 再放回池（我下载的 AOSP 14 第 639-645 行、16 第 649-655 行；`AccessibilityNodeInfo.recycle()` 从 13 起已是空方法）。所以窗口对象仍然必须遵守"谁取得谁回收、回收后不再使用"。候选做到了，见下节。

### 3.3 资源所有权【源码】

| 资源 | 创建者 | 释放者与时机 | 异常路径 | 判断 |
| --- | --- | --- | --- | --- |
| UiAutomation 登记和两个回调线程 | `UiBackend.registerAutomation` | 租约结束 `clear`，或连接失败 `discardConnection`，统一经 `AutomationCleanup.close` | 连接抛错时见 D1；注销失败后不重试，阻塞所有 UI 命令 | 正常路径干净；失败策略过严，见 §3.4 |
| 窗口对象 `AccessibilityWindowInfo` | `AccessibilityConnection.windows()`；其他显示器的窗口当场回收 | 采集：`TreeCollector.collect()` 的 finally（`TreeCollector.kt:68-74`）；准备阶段失败：`UiBackend.kt:444-446`；其他调用点（`SystemActions:174-183`、`UiActions:971-983`、`UiPublication:72-77`、`VisualInput:213-222, 399-421`）各自 try/finally | 从 `prepareCollection` 返回到 `collect()` 之间没有会抛异常的代码；`assemble` 不再访问窗口对象 | 把窗口回收收归采集作用域的那次重构（`da19691`）是干净的：删掉了 `Closeable`、`collector` 字段和三处转交，没有发现重复回收或回收后使用 |
| 节点对象 | 采集回调 | 不需要回收（14 起是空操作），由 `UiSnapshot` 持有到下一次采样或租约结束 | 无 | 正确 |
| 节点查询与动作的 Binder 回调 | 每次调用新建 | `CollectionNodes.close()` 和 `ActionCallback.finish()` 都在 finally 中 | 迟到的回调被丢弃 | 正确 |
| 位图 | `VisualInput`、`WorkDisplay.screenshot` | finally 中 `recycle` | 有 | 正确 |
| 后台虚拟屏、ImageReader、消费线程 | `TargetManagement.createTarget` 调用 `WorkDisplay.create` | 租约结束经 `UiBackend.clean` 调用 `WorkDisplay.close`；没有发布的创建经 `discardTarget`（`TargetManagement.kt:145-151`） | `close` 会先确认屏幕已消失、线程已停止，再关闭 reader；线程没停时不关 reader | 正确，略重（见下）；worker 死亡时屏上任务被销毁，已披露 |
| 通知监听 | `NotificationOperations` | 租约结束注销；同一 Binder 跨租约复用 | 注销失败只写诊断，不阻塞 | 正确；system_server 侧的残留见下 |
| 文件描述符、子进程 | `execute.c`、`files.c`、`ShellJobs` | 本轮 C 代码与基线完全相同（`git diff --stat` 为空）；全部 `O_CLOEXEC`，使用 `close_range` 和 subreaper | — | 没有变化，不展开 |
| 后台任务 | worker 的单线程 UI executor、每秒维护定时器；Gateway 的管理线程池和 2 秒状态轮询 | 随进程或服务生命周期 | 响应超时后，UI executor 上的工作会继续跑完再释放资源，这是有意的 | 合理 |

**"Android 15 以上通知监听的 JNI／Java 死亡回调与 native 释放未知"是什么风险**【源码，AOSP 14、16 `ManagedServices`】：

- `registerServiceImpl` 会对监听 Binder 调用 `linkToDeath(info)`（16 第 1958-1963 行）。
- `removeServiceImpl` 和 `removeServiceLocked` 只把条目移出列表，整个文件里没有 `unlinkToDeath`（14 第 1670-1691 行、16 第 1922-1943 行，`grep` 结果为 0）。
- 所以每个调用过通知命令的租约，都会在 system_server 里留下一个 `ManagedServiceInfo` 和一个死亡回调，直到 worker 退出。worker 一侧因为 Binder 对象复用，不会增长。
- 量级是"用过通知的租约数"乘以几百字节，worker 重启就清零。风险可以忽略，不值得再测。我同意上一轮记录 51 的判断。
- 如果想彻底消除，可以在 worker 生命周期内只登记一次，但那样在租约之外也会一直收到通知回调，与已定的"每租约登记、结束注销"冲突，不建议改。

### 3.4 设计与简化建议

1. **UI 清理失败后的"全局阻塞，等人工处理"策略过严**（与 D1 相关）。
   - UiAutomation 登记本身就有平台级互斥：如果旧登记真的残留，下一次连接会直接失败并给出原因。所以对"注销未确认"预先封住所有 UI 命令，并要求有人到设备上操作，对远程 LLM 调用者代价过高。
   - 建议：UiAutomation 这一项改成"下一次连接前再试一次注销，失败就让连接自然报错"；后台虚拟屏没有确认移除时仍然保留阻塞，因为残留的屏真的会把任务藏起来。
   - 同时可以删掉对 `mRemoteCallbackThread` 的私有字段反射：`disconnect()` 已经会 quit 这个线程，自己的回调线程照常 join 即可。这样少一个最脆弱的隐藏成员，代价只是不再"证明"平台内部线程已经退出。
   - 能力不受影响，代码会更少。
2. **worker 死亡后不自动重新绑定**（与基线相同）。对远程调用者来说，worker 一旦崩溃就必须有人到场；而 worker 重启后旧状态反正全部丢失，自动重新绑定一次不会有误操作风险。这是行为策略，列入 §5 由用户决定。
3. **前台通知**：见 D2。
4. **`HiddenApi` 作为唯一清单**：把剪贴板的反射收进来（D4），机械工作。
5. **不需要做的**：不需要"默认不采集 System UI 窗口"，现在本来就不采集；不需要给查询加节流，没有证据表明稳态查询对 System UI 有可测影响（见 §4）；也不需要再研究通知监听的 Android 15 回收问题。

### 3.5 前几轮决定与建议的再审视

- 上一轮把"收缩隐藏成员"当作维护目标，这个方向对。但 `disconnectConnection` 是作为"没用的成员"被删掉的，它其实承担着一条平台异常路径（D1）。以后删隐藏成员时，应同时核对它在失败路径上的用途。
- "清理未确认就阻塞、等人工"的保守策略来自早几轮对生命周期正确性的要求（实施方 25h／M8 生命周期评审）。对后台虚拟屏它是合理的；对 UiAutomation 登记来说，是在为"证明"而增加维护和可用性成本，属于钻牛角尖的一种表现，建议按上面第 1 条收窄。
- 关于 System UI ANR 的诊断：用户在第五轮之后的消息里（实施方记作 U35）专门要求诊断，所以投入有正当来源。实施方的数据采集是到位的：及时保存了 ANR 栈、PSI、连续日志，还配了时钟对齐。但分析文档有约 42 KB，大量篇幅花在"不能据此下结论"的限定语上，却没有用正式运行里"零掉帧"这条最有区分力的负面证据（见 §4），所以最后停在"根因未知"。投入和结论不成比例，有钻牛角尖的迹象。不过它没有据此去改产品，这一点是对的。

---

## 4. System UI 无响应事件（重点）

### 4.1 发生了什么、几次、在什么配置下

| 实例与配置 | 版本 | 事件 | 时间点 | 证据 |
| --- | --- | --- | --- | --- |
| 旧实例，2 GiB／2 核，软件渲染 | 候选首次准备 | "System UI isn't responding"对话框，原因没有保存 | 准备阶段打开 Maps 前后 | 【原件】我直接看了 `api361-play/r5-c1-play-candidate-01-first-systemui-anr.jpg` |
| 同上 | 候选唯一一次冷恢复 | 无 ANR | — | 实施方引用的 3593 和 3596（未逐字核对） |
| 同上 | 基线第 1 次 | `edge-swipe` 输入超时 5003 ms | 冷启后，激活附近 | 实施方引用的 3608 lastanr（未逐字核对） |
| 新实例，2 GiB／2 核 | 候选第一轮 | 无 ANR，但 System UI 有 781 帧掉帧和约 20 秒的长帧；准备阶段因 Maps 离线失败 | 冷启后约 30 秒 | 【重算】`sysui_stats.out` |
| 新实例，2 GiB／2 核 | 候选环境重试 | `edge-swipe` 输入超时 5001 ms | guest 10:26:48.9，冷启后约 46 秒 | 【原件】`U35/logs/u35-c1-candidate-01-retry1/logcat-all.log` 第 18061、18119 行；`U35/commands/121-.../stdout.bin` |
| 新实例，4 GiB／2 核 | 候选诊断 | `KeyguardService` 服务启动超时 20062 ms | guest 10:41:15.6 | 【原件】`U35/logs/u35-diag-candidate-4g-01/logcat-all.log` 第 15832 行；`U35/commands/145-.../stdout.bin` |
| 新实例，4 GiB／4 核 | 候选诊断 | 无 ANR；冷启时 System UI 有 428 帧掉帧和约 10 秒的长帧 | — | 【重算】 |
| 4 GiB／4 核（第 1 对）、6 GiB／4 核（后 4 对和候选的恢复重跑） | 候选 5 次加 1 次恢复，基线 5 次正式运行 | 全部无 ANR | — | 【重算】11 份连续日志中 `am_anr` 都为 0 |

**结论**：System UI ANR 共 4 次，全部在 2 核配置、冷启动后一分钟内。旧实例上候选和基线各有 1 次。新实例上的 2 次都是候选，因为新实例上基线从来没有在 2 核配置下运行过。4 核配置下两边都是 0。"候选和基线都出现"这句话，只在旧实例 2 GiB／2 核的配置下成立。

### 4.2 当时 System UI 主线程在做什么【原件】

- **2 GiB edge-swipe**（121 原件第 79-91 行）：主线程状态为 Runnable／R，`schedstat=(1356092113 22806786416 3046)`，即累计执行 1.36 秒、累计等待 CPU 22.8 秒。栈顶是 Keyguard 相关的协程，没有在等 Binder 或锁。
  - 日志顺序：41.923 秒，System UI 主线程记下"Skipped 744 frames"（约 12 秒没有出帧），这早于 Gateway 被点开。43.761 秒，点击投递给 AndroidCtl 管理页和 `edge-swipe` 等手势监视器（第 16610 行）。47.906 秒，产品第一次以 flags=0x1 登记 UiAutomation（第 17857 行），这时事件已经等了约 4.1 秒。48.902 秒判定 ANR。
  - 实施方说"产品 flags1 登记晚于事件开始等待约 4.14 秒，不能解释最初的积压"，这一点成立。
- **4 GiB 服务超时**（145 原件第 992-1006 行）：主线程状态为 Runnable／R，`schedstat=(1107569394 18260346108 ...)`，栈在 `SystemUIService.onCreate`、`startServicesIfNeeded`、`CentralSurfacesImpl.start`，也就是 System UI 还没完成启动。memory PSI 为 0，cpu PSI 的 avg10 为 89.92。
  - `am_anr` 在 15.573 秒；第一次 UiAutomation 登记（原生 dump，flags=0x0）在 17.087 秒（第 16217 行）；产品 UiAutomation、Gateway、worker 都更晚。AndroidCtl 管理页进程在 13.292 秒启动（第 15456 行），APK 安装在 10.577 秒完成（第 14914 行），后者属于准备脚本。
  - 结论：AndroidCtl 运行时不是这次 ANR 的必要条件。
- **两份 ANR 附件中 System UI 进程的全部线程栈里，`accessibility` 字样出现 0 次**【重算，grep】。也就是说，取栈时 System UI 没有任何线程在处理无障碍节点查询。

### 4.3 AndroidCtl 的各项行为会不会在 System UI 主线程上产生工作

| 行为 | 是否在 System UI 主线程上产生工作 | 依据 |
| --- | --- | --- |
| 默认 snapshot、等待、动作后再采样 | 否。只读当前活动的非输入法窗口；在普通 App 前台时，查询落在该 App 的 UI 线程上（这次是 Maps） | `UiScope.kt:64-67, 79-100`，`TreeCollector.kt:216-226`；基线 `7d5be70:TreeCollector.kt:132` 相同 |
| 窗口列表（`getWindows`） | 否，由 system_server 的无障碍窗口管理提供 | AOSP 结构【推断】 |
| `--all-windows`、按包名范围、通知栏展开后的 snapshot、System UI 对话框在前台时 | 是，会读 System UI 的根节点或整棵树 | `TreeCollector.kt:216-226` |
| 输入注入（点按、滑动） | 与人手触摸相同：手势监视器（`edge-swipe`、`swipe-up`）会收到所有触摸 | 日志第 16610 行 |
| UiAutomation 登记和注销 | 每次都会让所有 App（包括 System UI）收到一次无障碍状态变化；之后各 App 会生成无障碍事件 | 每个租约 1 次；准备脚本的原生 dump 每次也会触发 |
| 最小无障碍服务 | 开启后效果同上（各 App 生成事件）；服务的回调本身在 AndroidCtl 的 App 进程 | manifest、xml |
| 通知监听 | 否，回调在 worker | — |
| 前台通知更新 | 是，System UI 的通知管线要处理 | D2 |
| 展开通知栏（`GLOBAL_ACTION_NOTIFICATIONS`） | 是，只在调用者显式运行 `notifications` 时 | `SystemActions.kt:67-76` |

**稳态负载的直接证据【重算】**：U35 的 10 次正式 Maps 运行（候选 5 次、基线 5 次，4 或 6 GiB，4 核），以及候选第 4 次的恢复重跑，连续日志都覆盖了从冷启到收尾的全程（每次约 7–17 分钟，包括约 10 分钟的模型操作）。System UI 进程在这期间：
- 没有任何一次主线程 Choreographer 记下 30 帧以上的掉帧（这是平台记日志的门槛）；
- 没有任何一次超过 700 ms 的长帧（HWUI 记日志的门槛）；
- 唯一例外是候选第 4 次恢复重跑那次冷启：13:44:15–29 有 377 帧掉帧和 8.6 秒长帧，但它发生在第一次 UiAutomation 登记（13:44:34）之前。

可见，在 4 核环境里，AndroidCtl 稳态使用对 System UI 的影响低到测不出来。2 核配置下的稳态负载没有测过，因为所有 2 核尝试都在准备阶段就终止了。

### 4.4 对实施方结论的评价

- "输入超时伴资源竞争"：**成立**。依据是 schedstat、PSI（cpu 93.70、memory 16.44／6.45）、kswapd 和 major fault。
- "服务启动超时"：**成立**，而且应该说得更明确：它发生在任何 AndroidCtl UiAutomation 登记之前，System UI 当时还在启动服务。
- "准备脚本固定点击没有核对焦点"：**成立**。4 GiB 那次，点击实际投给了 "Application Not Responding: com.android.systemui" 窗口（日志第 37622 行）。脚本已改为点击前核对焦点。
- "根因未知，不宣称修复"：**过度保守，但没有过度免责**。实施方没有把"两版都出现"直接当成环境问题，也没有借此改产品，这是对的。但现有证据已经足以支持一个可以拿来做决策的判断：在 2 核、软件渲染的模拟器上，冷启动后约一分钟内 System UI 启动遭遇 CPU 饥饿，准备脚本又在这段时间里安装 APK（含 dex2oat）、启动 Shizuku、做原生 dump、点击；AndroidCtl 运行时不是必要条件，4 核下稳态负载测不出影响。实施方没有使用"4 核正式运行零掉帧"这条负面证据，也没有回答默认是否采集 System UI 窗口，所以停在了"不能排除共同路径"。
- 实施方提到"两版的通知负载不同"，这一点正确，但它没有量化。本报告 D2 补上了数据（8–29 次对 1 次）。

### 4.5 对产品的含义

- **不需要**为这件事改采集范围、加查询节流，或在公开指引里写 ANR 处理方法。调用者如果在 snapshot 里看到 "System UI isn't responding" 对话框，可以像普通对话框一样处理。
- **建议**（可选）：去掉前台通知的资源计数（D2），这是候选唯一新增的、可以消除的 System UI 负载。
- **建议**：在内部测试文档里写明模拟器的最低配置（至少 4 核），以及准备流程要等系统稳定后再开始（例如冷启后等 System UI 有 30 秒以上不再记录掉帧，或者等待固定时间后再核对输入焦点）。公开指引不需要写。

### 4.6 能区分"环境问题"和"产品贡献"的最小补验

不需要模型，也不计正式次数：

1. **时机对照**：在 2 GiB／2 核（或 4 GiB／2 核）配置下，冷启动后交替做 3 对：(A) 现行准备流程，boot_completed 后立即开始；(B) 完全相同的流程，但先等 System UI 稳定。
   - 如果 A 仍有 ANR、B 没有，就是启动时机和环境问题，可以就此结案。
   - 如果 B 也出现，再看 ANR 前 10 秒内 AndroidCtl 有没有动作。
2. **2 核稳态负载**：在已经稳定的 2 核设备上，先空闲 5 分钟，再跑 10 分钟 AndroidCtl 循环（snapshot、点按、等待），候选和基线各跑一次，比较 System UI 的 Choreographer 掉帧和 Davey 长帧次数。
   - 三组如果没有差别，就可以确认产品稳态影响可以忽略；
   - 如果候选明显高于基线，就去查通知更新（D2）和登记频率。

---

## 5. 通知只读

- **范围**【源码】：
  - 只读，三个命令：`notifications list`、`notifications get KEY`，以及不带子命令的 `notifications`（展开通知栏）。
  - `list` 返回身份信息加 title 和 text 摘要。`get` 另外返回 bigText、subText、textLines、已发送的回复历史和动作标题（`NotificationProjection.kt`）。
  - 写操作（打开通知、执行动作、回复、清除）以及 PendingIntent 描述都已删除。
- **在 shell 身份下如何实现**：见 §3.1。shell 没有 `ACCESS_NOTIFICATIONS` 权限，所以不能用公开的 `getActiveNotifications`；要拿到结构化的通知对象，唯一办法就是以系统身份登记一个监听（上一轮记录 51 核对过 AOSP 14 的 shell 权限，我在 AOSP 16 上复核了 `registerListener` 的权限检查）。
- **展开通知栏**：通过 UiAutomation 调用 `performGlobalAction(GLOBAL_ACTION_NOTIFICATIONS)`，派发成功就返回 done，不确认通知栏是否真的展开（`SystemActions.kt:67-76`）。后台虚拟屏会话里拒绝这个命令（`SystemActions.kt:40`）。
- **真实使用情况**：35 次计分运行和 2 次历史运行的 CLI 索引里，没有任何一次调用 `notifications`【重算，统计各运行的 `cli-index.json`】；固定任务里本来也没有需要读通知的题。实际 App 上的冒烟有两次：用 Feeder 阅读器的真实通知做了 list 和 get，主屏和后台屏各一次；还有一次是 Feeder 通知的 key 和标题核对。这些只核对了 `ST/real-tasks.md` 第 834、910-930 行的流水记录，没有逐一打开原件。
- **值不值得保留为独立命令组**：值得，现状就合适。
  - 它能做到 snapshot 做不到的事：不展开通知栏、不改变前台，就能拿到完整文本（例如验证码或消息正文），在后台屏会话里也能用。
  - 生产代码约 530 行（Access 77、Operations 175、Projection 279），隐藏成员 7 个，只在用到时才登记。
  - 维护风险主要在 `INotificationManager` 的系统接口上，在 14–16 稳定。
  - 删掉它省下的维护量有限，却会让调用者在读通知时不得不展开通知栏、读 System UI 的树，后者反而是对 System UI 主线程最重的一种查询。
  - 需要的修正只有 D3（"full mode"）和 D4（无用的反射）。

---

## 6. 管理界面、权限、Gateway、首次配对

- **从人的角度看首次使用路径**（README 第 40-97 行、`connection.md`、字符串资源）：
  1. `pip install`
  2. `androidctl apk --out`
  3. 安装 APK
  4. 安装 Shizuku
  5. 打开开发者选项和无线调试（或 USB 调试）
  6. 启动 Shizuku
  7. 打开 AndroidCtl，点"Authorize Shizuku"并在弹窗里允许
  8. 打开 Gateway，并允许通知权限
  9. 等到 Listening 和 Executor attached
  10. 点"Show new pairing code"
  11. 在主机上执行 `androidctl pair 地址 配对码`（同一局域网，或者自己做 `adb forward`）

  共 11 步必做，2 步可选：电池优化豁免；浏览器支持，需要去系统无障碍设置开启，侧载安装的 App 还要先允许受限设置。每次重启后要重新启动 Shizuku、重新打开 Gateway。这与基线一样，本轮变化只是新增了浏览器支持状态显示和 `FLAG_SECURE`。

- **`connection.md` 是否准确**：与代码一致。
  - 6 位配对码，600 秒、5 次错误或一次注册后失效（`Pairings.kt:139-144, 52, 82`）。
  - TOFU（首次连接信任设备出示的证书）及之后的证书固定。
  - 显示的地址只是候选，不代表一定能连通。
  - 需要手动"Refresh status"。
  - 不足之处：没有提到界面里的电池优化入口（可选补充）。`notifications.md` 的"full mode"见 D3。
- **明显的安全问题**：没有发现新的问题。
  - Gateway 监听所有 IPv4 接口（`0.0.0.0`，与基线相同），所以在移动网络或公共 Wi-Fi 上也会暴露一个 TLS 握手端点。不过没有配对窗口时，只有持有 Bearer 凭据的主机能通过认证。
  - 配对窗口只能由人在界面上打开；每个窗口最多试 5 次，猜中概率约 5×10⁻⁶。
  - 主机凭据是主机生成的随机串（40–128 字符），设备只保存它的 SHA-256（`Pairings.kt:65, 77`），比较用常量时间。主机侧凭据保存在用户私有目录。
  - 已知的设计边界：首次配对的 TLS 不绑定配对码（不是 PAKE），主动中间人可以在首次配对时介入。这一点已在指引中披露，与基线相同。
- **`FLAG_SECURE`（本轮新增）**：防止截屏或录屏泄露配对码，代价是原生截图里管理页是黑的。实施方已正确识别：这不是故障（`ST/runtime/api361-play/u34-anr-analysis.md` 第 91 行）。管理页本来就在采集时被标为"受保护"，两者一致。
- **权限模型**：产品里没有权限模式，所有已配对主机都拥有完整控制权。对单用户的 alpha 产品，这个选择合适；唯一的残留是 D3 那句文案。

---

## 7. 证据审计（实施方关键声称）

| 声称 | 判定 | 依据 |
| --- | --- | --- |
| 2 GiB 那次的 System UI 迟滞早于 Gateway 和产品 UiAutomation；产品 flags=0x1 登记晚于事件开始等待约 4.1 秒 | 成立 | 【原件】日志第 16167、16610、17587、17857、18061 行 |
| 主线程采样时为 Runnable，执行 1.356 秒、等待 22.807 秒，没有 Binder 或锁等待 | 成立 | 【原件】121 第 79-91 行 |
| 4 GiB ANR 早于原生 dump 和所有产品动作 | 成立 | 【原件】第 15832 行早于第 16217 行 |
| 4 GiB 点击误投到 ANR 对话框 | 成立 | 【原件】第 37622 行 |
| 6 GiB 运行无新 ANR | 成立，而且范围更大：4 核的 11 份日志全部为 0，System UI 也没有掉帧 | 【重算】 |
| "两版共同路径中通知负载不同" | 部分成立：实施方指出了，但没有量化 | 【重算】候选 8–29 次对基线 1 次 |
| "Android 15 以上通知监听的死亡回调与 native 释放仍未知" | 部分成立：问题本身是真的，但风险可以由 AOSP 源码直接界定为可忽略，不必继续保留为"未知" | 【源码】ManagedServices 14、16 |
| "窗口回收归采集作用域"的重构干净 | 成立 | 【源码】`da19691`；AOSP 确认窗口对象仍走对象池，所以这层纪律有必要 |
| 生命周期评审"Must-fix 0" | 不成立（在 D1 这一点上） | 【源码】评审没有覆盖"连接抛错时平台停在连接中"这条路径；`UiCleanupTest` 只是在复述"不重试" |
| 旧实例上候选首次准备的 ANR 原因 | 核不到 | 原件里没有保存原因，实施方也如实写了"没有可归属的原因" |
| 通知只读在真实 App 上走通 | 只核到流水记录，原件未核 | `ST/real-tasks.md` 第 834、910-930 行 |

关于失败、UNKNOWN 等是否如实保留：ANR 相关的准备失败、旧实例上那次 UNKNOWN、4 GiB 点击误投都单列保留了，没有计为模型失败，也没有补写成成功。实施方在"不把单次成功说成普遍结论"上做得很充分，甚至过度。

---

## 8. 需要用户裁决的新取舍

**唯一一项：worker 意外退出后，是否允许自动或远程恢复。**

- **A（现状）**：worker 死亡或清理未确认时，必须有人在设备上点"Refresh status"或"Stop idle executor"。
  - 得：每次恢复都由人明确决定。
  - 失：远程 LLM 调用者在这种情况下只能停下，等人到场。
- **B**：worker 死亡时，Gateway 自动重新绑定一次；旧状态照常全部作废，调用者会看到新的 worker 标识。
  - 得：崩溃后不需要人到场。
  - 失：多约 10 行代码。
  - 没有误操作风险：新 worker 没有旧的引用和作业，不会重放任何动作。
- **C**：B 的基础上，再加一个远程"重启 worker"命令，只在没有活动租约和作业时允许执行。
  - 得：还能覆盖"清理未确认"这类情况。
  - 失：多一条需要权限判断的远程路径。
- **D**：只修 D1，把 UiAutomation 的清理失败改成可以自动重试，其余保持 A。

**我的推荐**：B 加 D。D 的修正属于实施细节，可以直接做；B 改变了"恢复必须由人发起"这条既有原则，所以请用户确认。剩余未知是：真实设备上 worker 意外退出的频率，目前没有任何数据。

以下属于**实施细节**，不需要用户定：D1 的具体修法；去掉 `mRemoteCallbackThread` 反射；D2 的通知简化；D3、D4；在内部测试文档里写明模拟器的最低配置。

---

## 9. 有判别价值的补验

1. **UiAutomation 冲突复现**：在模拟器上让 `uiautomator dump` 循环运行，同时打开一个新租约并执行第一条 UI 命令。
   - 如果结果是后续所有 UI 命令都返回 `uiCleanup`，D1 就得到实测确认，必须修。
   - 如果平台在这种情况下的行为与源码推断不同（例如不会停在"连接中"），D1 降为可选。
2. **System UI ANR 的时机对照和 2 核稳态负载**：做法见 §4.6。只有在用户还关心"是否需要写入最低配置"时才值得做。如果结论是"仅与启动时机有关"，就可以正式结案，不再保留"根因未知"。
3. 这一块**不需要**再做 Android 15 通知监听的内存测量，也不需要更多 Maps 重跑。

---

## 10. 没能核实的内容

- 所有设备行为都只有源码推断，没有设备；尤其是 D1 在真实设备上的表现，以及 Android 17 上各隐藏成员是否还能用。
- 旧实例上基线 ANR 的 lastanr 原件（3608）和候选首次准备的命令原件，我只看了图，引用的是实施方的概述。
- Gateway 绑定 `0.0.0.0` 时，Android 的 NIO 是否会映射成双栈、接受 IPv6 连接（界面上会显示 IPv6 候选地址）：按 OpenJDK 的行为推断是双栈，没有核对 Android libcore 的源码。
- 最小无障碍服务订阅的那组事件类型是不是 Chromium 生成网页内容树所必需的：不在本专题范围，没有核对。
- 通知只读在真实 App 上的原始回执：只核对了流水记录。
- 所有结论都来自模拟器（API 34、35、36 镜像）。真机和厂商系统（例如厂商对 System UI 或通知管理的改动）上的表现，完全没有证据。
