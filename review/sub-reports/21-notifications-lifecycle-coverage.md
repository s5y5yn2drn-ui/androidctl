# 21 通知读写、listener 持有期、自身保护、控制者通知与生命周期、A/B 覆盖、D1/D5/D7

评审对象：候选 `3f45af6`（`NEW/source/androidctl`），实施方 U26 材料（`T/...`）。行号均为 3f45af6。
`C/` = `NEW/source/androidctl/android/app/src/main/java/com/rainng/androidctl/`；`U/` = `T/research/notifications/upstream/`。
本人下载的上游：`/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents4/21/up/`（Shell `AndroidManifest.xml` android-14/15/16.0.0_r1；`ManagedServices.java` android-14/15.0.0_r1）。
没有设备，没有构建，没有重跑 pytest；host 失败数用的是上轮我方 `hostcheck/pytest-full.log` 原件。

## 1. 结论摘要

1. **通知写应保留**（修正我方上轮"首次只留只读"）。决定性的源码事实是：VD 会话里裸 `notifications`（下拉通知栏）被直接拒绝（`C/SystemActions.kt:41`）。所以在后台屏模式下，结构化写是不切主屏就能处理通知的唯一路径。action、dismiss 无条件保留。open 要补"在 virtual 会话里落到哪块屏"的处理。reply 保留，发布前在模拟器 Messages 上做一次安全的真实投递验证即可，不必砍。
2. **shell 不注册 listener 就直接读通知，这条路不存在**（源码核对＋实施方有设备原件）。AOSP 14/15/16 的 Shell manifest 都没有声明 `ACCESS_NOTIFICATIONS`，`getActiveNotifications*` 和 `getHistoricalNotifications*` 都需要这个权限。`cancelNotificationWithTag` 对他包会抛 SecurityException。`dumpsys`/`cmd notification` 只有文本，拿不到 PendingIntent。实施方在 `T/research/notifications/report.md:17-21` 已做过这项排除，结论成立。
3. **题 4 的 A/B/C/D 都是在局部打补丁。可以整体消除的方案 E**：每个 worker 只用一个 listener Binder 对象，仍然每条命令注册、注销。注销失败只记诊断，不挡任何能力，下一次使用时顺带补注销。依据有两条：A14/15/16 的 `ManagedServices.removeServiceImpl` 按 Binder 身份一次删掉该 Binder 的全部登记；注册/注销实测只需 0–3 ms。D5、"多旧 token 累积"、lease 耦合、清理三态、exit 覆写会一起消失。候选的 D5 根因是 `NotificationAccess.prepare` 每次 `new NotificationListener()`（`C/NotificationAccess.kt:45`），再加上一道全局门。
4. 实施方推荐的 C（每 lease）不同意。它把设备级的平台登记绑到 lease 上，保留了 close 时的全局失败门，性能上也没有可测的收益。
5. postTime 严格匹配作默认不合适。更好的第三种做法：action/open/dismiss 默认比对 key＋index＋title＋creator；reply 默认比对"内容指纹"（title/text 的哈希），不比对 postTime；把这些值合成一个调用者只需复制一次的选择串。
6. D5、D7、self-open、ongoing 预拒的源码定位，实施方都描述得对。self 护栏建议按 `sbn.packageName == own` 覆盖全部写命令，不只用 open 的 creator 判断。dismiss 能藏掉控制者通知。
7. D1 修法描述基本正确，有一处小误：22 项不全在 `test_display.py` 的五处，另有 2 项来自 `test_ui_batch.py:338`。候选自带 CI 在 ubuntu/macOS 上跑 pytest，按原样会是红的，所以"已审 1884 passed"只是 Windows 结果。
8. A/B 覆盖：b-disposition 标"采用/调整后采用"的 5 项在源码里没有（应用笔记、数据非指令、WebView 专项折叠、stale 候选、shell 配方）。实施方本轮已如实改称"未落地旧建议"，审计如实。值得提前的未纳入项是"数据非指令"一句话；ADB bootstrap 有门槛收益，但不该进首版。

## 2. 独立判断

### 2.1 通知写（action/open/dismiss/reply）去留

**调用者用它做成了什么**（源码核对）

- **后台屏模式下处理通知**：`SystemActions.kt:41` 在 `target.kind == VIRTUAL` 时拒绝 `home`/`recents`/`notifications`。我方上轮让调用者用"shade＋节点"替代通知写，这在 VD 会话里根本走不通。调用者只能先 close 或切到 main 会话，打扰真人主屏，再去下拉通知栏。结构化写是唯一不切主屏的路径。
- **不经 UI 执行 broadcast/service 型 PendingIntent**，比如"标为已读/暂停/归档"。这类 PI 没有界面，直接 `sendAndReturnResult`（`NotificationActions.kt:206`）即可。走 shade 要展开、找节点、可能还要展开 group，每步都要观察，大约 3–5 次调用加 1–2 次树读取。结构化写是 get＋action 两次调用。属于推断，没有成对实测。
- **RemoteInput 回复**：SystemUI 内联回复走 UI 要 展开→Reply→输入（IME 或 IC）→发送，至少 4 步，还受 IME/焦点影响。结构化回复是 get＋reply 两步，不碰 IME（`NotificationReplyInput.kt`、`NotificationActions.kt:292-313`）。
- dismiss：替代方案是滑动手势，结构化方式更简单。

**风险与失败模式**（源码核对＋上游）

- **BAL**：Activity 型 PI 由 shell 带 `MODE_BACKGROUND_ACTIVITY_START_ALLOW_ALWAYS`（API34 上是 `ALLOWED`＋ByPermission）发出（`NotificationActions.kt:211-231`）。shell 持有 `START_ACTIVITIES_FROM_BACKGROUND`（Shell manifest 14/15/16 都有，本人核对）。实施方 API36/MuMu35 有正例。厂商 ROM 未知，需设备。
- **open 的落屏**：`configureNotificationActivityOptions` 只设 BAL 模式，没有 `setLaunchDisplayId`（`:199-206,211-231`）。virtual 会话里调 open，Activity 会出现在主屏（或该 task 已在的屏）。`RpcMethods.requiresDisplay` 把通知组排除，没有任何拒绝或提示。`skills/androidctl/notifications.md` 也没写"open 作用在主屏"。这和后台屏"不打扰真人"的承诺直接冲突，我认为是现实现最大的可用性缺口，严重度高于 postTime。
- **PI 目的地未知**：`landing.state` 恒为 unknown（`:433-467`）。前后两次 root task 表各最多 8 项，多数调用里没有消费者，建议裁掉，改成"调用后自行 snapshot"。实施方也认为可以裁。
- **reply：本地接受 ≠ 送达**。platform code 只说明 PI 已投递。共享 PI 的 a→b 错绑是 producer 的问题，SystemUI 内联回复同样会中，不是本工具独有的风险（推断，与实施方一致）。
- **dismiss 与 ongoing**：A14 `U/a14-NotificationManagerService.java:4697-4705`、A16 `:5833-5840` 用 listener 取消时排除 `FLAG_ONGOING_EVENT`（A16 在 refactor 开关下还排除 `FLAG_LIFETIME_EXTENDED_BY_DIRECT_REPLY`）。候选 `NotificationActions.kt:406-419` 是 `while(true)` 轮询直到截止，结果是 UNKNOWN，并且停批。
- **主屏就绪门**：`begin()`→`inputReady(ui.deviceFacts())`（`NotificationActions.kt:124-128`，`DeviceFacts.kt:69-74`）对所有写生效，包括不碰 UI 的 broadcast/service 型 PI 和 dismiss。这让锁屏或熄屏时的"标为已读/暂停"做不了。门的安全理由看不出来：shell 本来就能 `am broadcast`，"full 不是沙箱"。属于设计建议，见 §6。

**实现与维护**（源码核对）

通知相关 Kotlin 约 1190 行（Actions 478、Projection 351、Operations 226、Access 83、ReplyInput 52），其中写相关约 530 行。长期要维护的平台接点有：隐藏 API 反射（`HiddenApi.kt:24-65`）、BAL 选项版本分支、RemoteInput 选择、UNKNOWN 纪律、清理三态。清理三态这一块在方案 E 下整块删除，见 §2.2。

**分项推荐**（设计建议）

| 命令 | 推荐 | 条件 |
|---|---|---|
| action | 保留 | 非 Activity 型 PI 考虑免主屏就绪门 |
| dismiss | 保留 | ongoing（及已置的 lifetime-extended 位）派发前拒绝；自身包拒绝 |
| open | 保留 | virtual 会话要么带 `launchDisplayId=目标VD`（需设备验证 PI sender options 是否生效），要么明确拒绝或提示"作用主屏"；裁掉 landing 表；自身包拒绝 |
| reply | 保留 | 发布前做一次模拟器 Google Messages 验证（`adb emu sms send` 造来信，再 reply）。模拟器 modem 是仿真的，不经运营商（推断，需确认镜像）。公开文档写清"真实外发、平台 code 不证明送达" |

"真实聊天投递没验证"不足以把 reply 排除在首版之外。理由有三：RemoteInput 契约和 SystemUI、Wear、Auto 用的是同一份；增量代码小；存在安全的验证手段。不足之处可以靠一次安全验证补齐，用不着删能力。

### 2.2 listener 持有期（题 4）：能否整体消除

**A. 不注册 listener 直接读，不可行**（源码核对＋原件核对）

| 路径 | 结论 | 证据 |
|---|---|---|
| `getActiveNotifications(WithAttribution)` | 需要 `ACCESS_NOTIFICATIONS`（`@EnforcePermission`），Shell 未声明 | `U/a16-NotificationManagerService.java:5359-5363`；本人取的 Shell manifest 14/15/16 都没有该权限，只有 `STATUS_BAR_SERVICE`、`STATUS_BAR`、`MANAGE_NOTIFICATIONS`、`MANAGE_NOTIFICATION_LISTENERS`、`INTERACT_ACROSS_USERS(_FULL)`、`START_ACTIVITIES_FROM_BACKGROUND`、`RECEIVE_SENSITIVE_NOTIFICATIONS`（15/16） |
| `getHistoricalNotifications*`、`getNotificationHistory` | 同样需要 `ACCESS_NOTIFICATIONS` | A16 `:5473-5495,5511-5520` |
| `getAppActiveNotifications(pkg)` | `checkCallerIsSystemOrSameApp`，shell 不满足 | A16 `:5407-5410` |
| `cancelNotificationWithTag(他包)` | `resolveNotificationUid` 对非同包、非 android 的调用者抛 SecurityException | A16 `:4219-4232,8994-9016` |
| `dumpsys notification --noredact`、`cmd notification list/get` | 只有文本，PI 是字符串，不是 Binder token，不能 send，RemoteInput 不可用 | 实施方 `T/research/notifications/report.md:17` |
| 设备 | API34/API36 AVD 上 shell 的 `ACCESS_NOTIFICATIONS` 为 denied，直接读抛 SecurityException；临时 register/get/unregister 正常 | `T/research/environment/followup.md:98`；`T/prototypes/device/selected-validation/platform-avds/api34/notifications-result.md:13`（原件核对，仅读结果页） |
| App 自有 NLS（shell 用 `MANAGE_NOTIFICATION_LISTENERS` 开授权） | 能读，但会持久改用户设置；内容进 App UID，15+ 敏感内容会被脱敏；PI 从 App UID 发出，失去 shell 的 BAL 特权 | 推断；比现方案差 |

所以"每命令注册/注销""跨命令回调寿命""URI grant 归属 shell"这几件事都不会因为换读法而消失。实施方考虑过这条路，否定理由成立。

**B. 局部补丁和可整体消除的部分**（源码核对＋上游核对）

- 候选每条命令都 `new NotificationListener()`（`C/NotificationAccess.kt:45`），注销失败就把该 Binder 的闭包挂在 `NotificationOperations.access` 上，设 `cleanupFailure`（`NotificationOperations.kt:72`）。这个值永不复位，也没有重试入口。它经 `UiBackend.kt:46` 进入 `Sessions.acquireUi:144`，拒绝所有新的 UI 执行命令；`UiBackend.kt:139` 不释放 Resource，于是 shell/file、idle stop、expiry 一并被挡。恢复只能靠真人在设备上点 "Force executor exit"，因为 `skills/androidctl/connection.md:170` 写明 "There is no remote force command"。对远程调用者来说这是死路。
- 上游：`ManagedServices.removeServiceImpl` 在 A14（本人取 `ms-android-14.0.0_r1.java:1670-1684`）、A15（`:1750-1764`）、A16（`U/a16-ManagedServices.java:1922-1936`）里都是**倒序遍历、删掉所有 `asBinder()==` 同一 Binder 且同 user 的登记**。`unregisterService` 不查权限，找不到也不报错（A14 `:1084-1089`，A16 `:1236-1240`）。`registerServiceImpl` 每次 `linkToDeath`＋`mServices.add`（A16 `:1958-1967`），Binder 死亡时逐项移除。
- 实施方反对"只记诊断"的第二条理由是：同 component 再注册不会替换旧 token，NMS 会同时持有多个旧 token（`notifications-coverage.md` §3.2 第 2 点）。这只在"每次新建 Binder"的前提下成立。**改用 worker 级单例 Binder，重复注册只是同一 Binder 的多条登记，一次注销就全部删掉**，累积问题从结构上消失。
- 成本：receipt 里 `notificationRegisterMs`/`notificationCleanupMs` 全部在 0–3 ms（离线重算：对 `T/stages`、`T/prototypes` 下的 JSON 用 grep 统计，register 取值只有 0/1，cleanup 在 0–3）。"少反复注册注销"在性能上没有可测收益。

**C. 方案 E：单例 Binder，每命令注册/注销，注销失败不设门**（设计建议）

- UiBackend（或 worker）持有一个 `NotificationListener` Binder 和一个 `NotificationAccess`（manager、listener、方法），懒创建。
- 每条命令：`register(listener)` → 读或派发 → finally `unregister(listener)`。
- 注销失败：主回执不变，**不改 exit**，附一条 `warnings`。置 `unregisterPending=true`，status 诊断可见。不保留 Resource，不设 `cleanupFailure` 门。
- 下次通知命令开头、以及 `prepareStop` 中，各尽力补一次 `unregister(listener)`。补注销失败也不阻止 stop。worker 死亡时 `binderDied` 兜底。
- 注册抛错（可能已登记）也由 finally 的 `unregister(listener)` 覆盖，不需要"未知注册"状态。
- 删除的东西：`cleanupPending`、三态 `data.cleanup`、`notificationCleanup` 的 exit 覆写（`NotificationOperations.kt:102-112`）、`UiBackend.kt:139` 的例外、`skills/.../notifications.md:30-34` 一段契约、对应契约样本与测试。
- 保留的东西：观察窗口和现状一样短（每命令）；不和 lease 耦合；通知仍是设备级能力。
- 剩余风险：如果注销持续失败（system_server Binder 异常），listener 会留到 worker 死亡。这正是 D 案的常态，后果有界，status 可见。

**D. 对四案的评判**

| 案 | 评价 |
|---|---|
| A 现状 | D5 一次失败就让远程调用者陷入死局，不可取 |
| B 局部隔离 | 方向对，但实施方按"每命令新 Binder"设想它，所以多出 pending token 管理。换成单例 Binder，B 就退化成 E，几乎没有额外状态 |
| C 每 lease（实施方推荐） | 不同意。性能收益为零（0–3 ms）。把设备级登记绑到 lease，close/expiry/revoke 时的注销失败仍然走全局 closing 门，D5 只是换了触发时机。观察窗口延长到 lease 寿命（最长 24h idle）。多一份"lease 持有的 listener"状态要随 `ui.clear(lease)` 维护 |
| D 每 worker | 代码最少，但观察窗口跨 Gateway OFF、close、撤销配对，并且现在每 2 s 还会收到一次 Gateway 自身通知的回调（D7 未修前，`GatewayService.kt:104-115`）。收益只有每命令省 1–3 ms |
| **E 单例＋每命令＋失败不设门** | 推荐。观察窗口和 A 一样，代码比 A、C 都少，D5 结构性消失 |

把握：上游行为在 14/15/16 三版源码一致，把握高。下面几点需设备确认：同一 Binder 重复 register 后一次 unregister 确实全部移除（`dumpsys notification` 的 live listener 计数）；单例 Binder 跨命令复用时 `getActiveNotificationsFromListener` 的 token 校验正常。

**题 4 是不是"在可整体消除的问题上做局部优化"**：是。D5 的根因是"每命令新 Binder＋全局门"，与持有期无关。实施方把它当成持有期问题，在 A/B/C/D 里挑选，C 甚至把失败门保留到 close。

### 2.3 D5、self-open、ongoing 预拒：处置核对

- **D5**：实施方 §3.1 的描述准确（源码核对）：挡的是新 UI 执行命令，`ui.read` 走 `acquireExistingUi`，不受影响。我方上轮写"所有 UI 请求"过宽。实施方不采用"只记诊断丢 token"的理由，在新 Binder 前提下成立，在单例 Binder 下不成立，见 §2.2。
- **self-open**：`GatewayService.kt:38-40` 的 contentIntent 指向 MainActivity，`NotificationActions.kt:238-269` 没有自身拒绝，`ActivityLaunch.kt:187-189` 却拒绝自身 handler。两条通道不一致，属确定缺陷。实际泄露有限：像素有 FLAG_SECURE（`MainActivity.kt:95`，配对码对话框 `ManagementScreen.kt:323-325` 显式 `SecureOn`，其余 AlertDialog 继承父窗口）；节点是 `protected`（`TreeCollector.kt:240`）；事件有过滤（`UiSignals.kt:165-167`）；坐标和 UID 输入用 `requireExternalRoot`/`inputUid` 拒绝自身（`AppInput.kt:77-87`）。所以后果是"能把管理页拉到前台遮住主屏"，读不到内容，也点不了。
- **自身保护建议扩成按 `sbn.packageName == ownPackage` 拒绝全部写**，不只 open 的 creator。原因：listener 取消只排除 ONGOING（A14 `:4700`），而 Gateway 的 FGS 通知只被 NMS 加 `FLAG_NO_CLEAR`（A14 `:7927-7930`）。是否带 ONGOING 取决于 AMS，未核，需设备。如果能被 dismiss，控制者通知这个唯一的"有人在远程控制"信号就会被调用者藏掉。现在每 2 s 的无条件 notify 会把它补回来；D7 去重修好以后反而不再补。这一点只有推断，需设备。
- **ongoing 预拒**：同意。补充建议：同时检查 `FLAG_LIFETIME_EXTENDED_BY_DIRECT_REPLY` 位（API35+ 才存在；位已置时预拒，A14 不会出现该位）。预拒不是穷尽分类，仍保留取消后的后验。

### 2.4 版本地址：key＋postTime 严格匹配

- 事实（源码核对）：每次 enqueue 都新建 SBN，取 `System.currentTimeMillis()`（A16 `:8454-8456`）；候选严格比较（`NotificationActions.kt:245`）。失败时 `rememberSelection` 已把当前 identity（含新 postTime）放进 NOT_SENT 回执，所以对**低频更新**的通知，恢复成本是 1 次重试。
- 对 LLM 调用者：list→act 之间通常隔 5–30 s。进度、下载、媒体、导航这类通知 1 s 级更新，严格匹配下几乎永远不会收敛，重试也跟不上。聊天通知在新消息到达时更新，此时拒绝反而有价值，因为回复的语境变了。有些 App 不改内容也会重发（同步已读、静默刷新），严格匹配会无谓拒绝。频率需设备统计。
- **第三种做法**（设计建议）：
  1. action/open/dismiss 默认比对 key、index、title（已有）、creator/creatorUid（已有），**不比对 postTime**。title 和 creator 才是"动作含义没变"的检查，postTime 什么也没说明。
  2. reply 默认比对**内容指纹**：get 时由 worker 计算 `hash(title, text, 最新 remoteInputHistory)`，reply 时重算比较。新消息到达就拒绝，无内容变化的重发就放行。这比 postTime 准确。
  3. 把 key/index/指纹/creator 编码成一个由 get 输出的**选择串**（无状态，worker 重算校验），调用者只复制一个值。现状要复制 6 个参数（`notifications.md:51-55`），出错面大，token 也多。
  4. 保留 `--exact`（含 postTime）给需要严格版本的场景。
- 实施方保严格默认的理由是"可能执行未观察的版本"。对 dismiss 和大多数 action 来说，title＋creator 相同就是同一含义的动作；对 reply，内容指纹比 postTime 更直接。实施方 §2 第 24 行说"同 index 的动作可能已代表不同意图"，这种情况 title 检查已经拦住；title 和 creator 都不变而含义变了，是 producer 的问题（和共享 PI 错绑同类），postTime 也拦不住。

### 2.5 权限与自身保护（#1、#2）

- Android 14 下限：本专题无反对意见。Shell 权限集合在 14/15/16 之间稳定（本人核对 manifest）；`RECEIVE_SENSITIVE_NOTIFICATIONS` 只有 15/16 有，14 上可能拿到脱敏正文（推断，与实施方 report.md:22 一致）。
- 四处保护的一致性：像素（FLAG_SECURE）、节点（protected）、事件正文（UiSignals 过滤）、输入（requireExternalRoot/UID）四处一致。不一致的只有**通知写**（open 用 creator 判断缺失，dismiss/action 不看自身包），以及 typed open（已拒）。按 §2.3 统一成一条 `sbn.packageName/creator == own` 即可，不需要更多机制。
- "full 不是沙箱"的表述：`skills/androidctl/observations.md:173-177` 已有。shell `am start` 能绕过一切自身 UI 护栏，护栏防的是误操作，不是恶意调用者，文档应该这样说。实施方表述一致。

### 2.6 控制者通知与生命周期（#33、D7）

- D7（源码核对）：`observeWorker` 每 2 s（`GatewayService.kt:31-37,436`）→`enqueueManagement`→`publishStatus`→`publishNotification` 无条件 `notify`（`:104-115,310-313`）。错误只在 `enqueueManagementCommand` 成功时清除（`:280-284`），poll 失败写入的错误（`:299-302`）之后不会被成功的 poll 清掉。实施方"按可见投影去重、只清 poll 自身错误"的修法正确，而且是最小修法。附带效果：每 2 s 一次 notify 会让 Gateway 通知的 postTime 不断变化，并给所有 listener（包括 D 案常驻 listener）发回调。
- 四者关系（源码核对＋文档）：
  - daemon（host）：只持连接和本地会话缓存，退出不结束 lease/worker。
  - Gateway：App 前台服务，网络入口和配对。OFF 时只 detach（`detachWorker:400-419`），worker、lease、job 照常运行，控制者通知随 Gateway 消失。
  - worker：Shizuku UserService，持有 Sessions/lease/VD/job。没有"lease 全部到期就退出"的机制。
  - lease：在 worker 内，idle 最长 24h。
- 状态没有重复权威：通知和管理页读同一份 `WorkerConnection.Attached.status`。缺口在**可见性**：Gateway OFF 以后 worker 和 lease 可以无通知地存活到 expiry，甚至更久。这是 U9 的取舍，不是本轮的缺陷。但如果选 D 案（listener 每 worker），这段不可见期还会多出持续的通知监听，这是我不推荐 D 的另一理由。

### 2.7 D1

- 原件核对（`hostcheck/pytest-full.log`，Linux/Py3.12）：23 失败 = `test_display.py` 20 项 + `test_ui_batch.py` 3 项。
- 源码核对：`test_display.py:78,166,388,436,523` 和 **`test_ui_batch.py:338`** 都是 `path.parent.mkdir()`，没有 `parents=True`。POSIX 上 `state_directory()` = `~/.local/state/androidctl`（`private_files.py:21-24`），HOME 被隔离到 tmp（`conftest.py:24`），中间两级不存在，所以报 ENOENT。Windows 上父目录就是 tmp，因此能通过。实施方写"五处×参数化足以解释 22 项"，漏了 `test_ui_batch.py:338`（2 项：`test_bad_preference_blocks_only_batches_with_target_steps[False/True]`）。修法相同，不影响结论。
- 第 23 项：`workspace.read`（`workspace.py:123-130`）只接 `FileNotFoundError`。`.androidctl` 是普通文件时，POSIX 抛 `NotADirectoryError`，变成 Fault；Windows 抛 FileNotFoundError。按缺失配置处理 ENOTDIR 是正确修法（实施方描述正确）。
- **对"候选已验收"的影响**：候选自带的 `.github/workflows/quality.yml:17-33` 在 ubuntu-24.04（py3.12、3.14）和 macos-15-intel 上跑 `python -m pytest`，按原样这三个 job 会失败。`T/deliverables/candidate-final.md:40` 写的"完整 host/release 结果 1884 passed"没有注明平台，实际是 Windows 结果。缺陷本身很小，但"候选质量已关闭"这一说法对候选宣称支持的平台不成立，应改为"Windows host 通过，POSIX host 23 项失败未修"。
- 实施方处置写"23 失败是外评 Linux 实测，本轮未复跑"，如实，但没有承认这等于候选自身 CI 红。

### 2.8 原报告 A/B 覆盖抽查

抽查 10 项（源码核对：在候选 `skills/`、`androidctl/src`、`android/app/src/main` 中 grep）：

| 原 B 建议 | b-disposition | 源码实际 | 实施方本轮表述 | 判定 |
|---|---|---|---|---|
| 应用笔记（:186） | 调整后采用 | 无（skills/README 无 notes 约定） | "未落地旧建议" | 如实 |
| 屏幕/通知/shell 是不可信数据（:223） | 建议采用 | 无（无 untrusted/not instructions 语句） | 未落地，可补一处 | 如实；**应提前**，一句话、零维护 |
| WebView 文本块折叠（:102） | 调整后采用 | 无专项代码（host 无 webview） | 未采用专项，通用投影部分等效 | 如实 |
| 每个非 done 给可运行 next（:124） | 调整后采用 | 约 30 处内联 `; run snapshot` 之类尾句（离线 grep，`[;.] (run\|use\|retry\|restart...)` 共 31 处） | "约 20 处" | 基本如实，数略低 |
| staleRef 候选（:126） | 调整后采用 | 无，只有 `ui_view.py:221` "run snapshot" | 未做 | 如实 |
| --restart/shell 配方（:155） | 调整后采用边界 | skills 无 force-stop/am 配方 | 未提 | **遗漏**，实施方余项表没列 |
| 不强制 session open/alias（:179） | 建议采用 | 隐式 acquire 有，alias 无 | 交会话线 | 如实 |
| toast/通知事件元数据（:80） | 建议采用 | 有（`UiRecentEvents.kt:30,78,92`） | 已覆盖 | 如实 |
| 控制者通知（:211） | 调整后采用 | 有（GatewayService） | 已覆盖 | 如实 |
| ADB bootstrap（:201） | 待验证，独立候选 | 无 | 本轮未纳入 | 如实 |

未纳入项里对"能力强、使用高效"影响大的判断：

- **数据非指令提示**：应进首版，成本一行。
- **ADB bootstrap**：它对已有 adb 的开发者省掉"装 Shizuku、每次开机用无线调试或 adb 启动 Shizuku、授权"三步。但非 root 设备上 Shizuku 本身也要靠 adb 启动，两者门槛差主要是"多装一个 App 外加一次授权"。它还引入第二个 provider：worker 启动、Binder 交接、断连后存活、重绑，维护责任不小，主机还必须有 adb 直连（远程 Wi‑Fi 配对场景用不上）。结论：降门槛有真实收益，但不是首版阻塞项，适合首版之后单独立项。属于推断，没有用户画像数据。
- OCR、profile、录制回放、真人让路、协议解耦：多模态模型直接看截图已经覆盖 OCR 的主要场景；其余对首版效率影响小。同意不提前。

### 2.9 外评验证清单处置

- V21/V22（删 IC、zoom 后的回归）：保留 IC/zoom 后确实没有依赖，"无需补证"成立。
- V19/V20（事件节拍、toast 捕获）：不属本专题，不评。
- B1–B3（构建检查）："现不为研究文档重跑全 CI"本身成立，但 D1 表明候选在自带 CI 平台上已经不通过。获批实施时 POSIX pytest 必须列为门，不能沿用"已审 1884 passed"。
- 通知相关：外评没有专门的 V 编号。实施方 §5 的补证表（postTime 节奏、MessagingStyle、ongoing、D5、真实聊天、D7）合理。其中"真实聊天"一项我认为有安全手段（模拟器 Messages），实施方认为"不能自动视为安全"，需要确认模拟器镜像的 SMS 是否只在仿真 modem 内回环，见 §7。

## 3. 对照实施方

| 实施方结论 | 判定 | 依据 |
|---|---|---|
| 保留通知 D，"shade 替代会占主屏并丢直接 PI 地址" | **成立**（理由比实施方写的更强） | `SystemActions.kt:41` 在 VD 会话里拒绝 shade，替代路径不存在 |
| 结构化 open "不随 VD 迁移" | 成立，但处置不足 | 只写了限制，没提出 launchDisplayId 或拒绝方案，skill 也没写，调用者会在后台屏会话里打扰主屏 |
| "现在读不到聊天内容"被一手源反驳 | **成立**，更正我方 | A16 `Notification.java:9388-9414` 用最新消息填 EXTRA_TITLE/TEXT |
| shell 无 ACCESS_NOTIFICATIONS，direct read 不可行 | **成立** | 本人核 Shell manifest 14/15/16＋实施方设备原件 |
| D5 描述（挡新 UI 执行命令，不含 ui.read） | **成立**，更正我方"所有 UI 请求" | `UiDelivery.read`→`acquireExistingUi` |
| "只记诊断"不安全：多 token 累积、Binder 异常≠死亡 | **部分成立** | 前提是每命令新 Binder；单例 Binder 下累积不存在（removeServiceImpl 按 Binder 删全部）。"异常≠死亡"成立，所以 E 方案仍做补注销，不依赖死亡 |
| 题 4 推荐 C（每 lease） | **不成立** | 零性能收益，保留 close 时全局门，绑错所有权；E 严格更优 |
| postTime 严格默认 | **不成立**（作为默认） | 高频更新通知不收敛；title/creator/内容指纹才表达"含义没变" |
| self-open 修 creator 拒绝 | 部分成立 | 应扩到所有写命令按自身包判断（dismiss 可能藏控制者通知，需设备） |
| ongoing 预拒 | 成立 | 补 lifetime-extended 位 |
| D7 修法 | 成立 | 源码一致 |
| D1 "五处×参数化解释 22 项" | 部分成立 | 漏 `test_ui_batch.py:338`；未承认候选自带 CI 会红 |
| ENOTDIR 修法 | 成立 | |
| A/B 覆盖"如实" | 基本成立 | 漏 shell 配方（b-disposition:155）；next 计数偏低；其余如实 |
| landing 表可裁 | 成立 | |
| MessagingStyle 不默认新增 | 成立 | latest 已在 text；多条历史按需再议 |

## 4. 对上轮外评（我方）的更正

1. §2.4 "通知首次只留只读，写操作用打开通知栏＋节点"：**错**。VD 会话里 shade 被拒，替代路径不存在。应保留写。
2. "现在读不到聊天内容"：**过强**。MessagingStyle 的最新一条已经在 title/text 里；缺的只是多条历史和发送者结构。
3. D5 "挡住所有 UI 请求"：**过宽**，`ui.read` 不受影响。"worker 死亡后自动移除，所以只记诊断即可"：**论证不完整**。在每命令新 Binder 的现实现下会累积多个 token；正确的修法要配合单例 Binder 和补注销。
4. "地址应缩为 key 加动作索引"：方向对，但去掉 title、creator 过头了。title 和 creator 是有效的含义检查，应保留；postTime 换成可选，reply 用内容指纹。
5. "reply 需要在真实聊天 App 上验证后再发布"：要求本身合理，但我方当时把它当作"不进首版"的理由，过重。有安全的验证途径，应该补证而不是删能力。

## 5. 确定缺陷（本人核过）

| # | 位置 | 可观察后果 |
|---|---|---|
| 1 | `NotificationOperations.kt:72`（cleanupFailure 永不复位）＋`UiBackend.kt:46,139`＋`Sessions.kt:144` | 一次注销异常后，所有新 UI 执行命令、新 shell/file、idle stop、expiry 都被拒，远程无法恢复（`connection.md:170` "no remote force command"） |
| 2 | `NotificationAccess.kt:45` 每次新建 Binder | 失败后若有任何路径再注册，NMS 会累积多个独立 token（结构原因，上游 removeServiceImpl 按 Binder 删） |
| 3 | `NotificationActions.kt:238-269` 无自身检查；`GatewayService.kt:38-40` | open 能把 AndroidCtl 管理页拉到前台，与 `ActivityLaunch.kt:187-189` 不一致 |
| 4 | `NotificationActions.kt:406-419`，无 flags 预检 | ongoing 通知 dismiss 等到截止才报 UNKNOWN，并停批 |
| 5 | `NotificationActions.kt:199-206,211-231` 无 launchDisplay；`RpcMethods.requiresDisplay` 排除通知 | virtual 会话里 notifications.open 在主屏启动 Activity，没有拒绝也没有文档提示 |
| 6 | `GatewayService.kt:104-115,280-302` | 每 2 s 无条件 notify；poll 错误粘住不清（D7） |
| 7 | `androidctl/tests/test_display.py:78,166,388,436,523`、`test_ui_batch.py:338`、`workspace.py:127` | POSIX 上 23 项失败；候选 CI 的 ubuntu/macOS job 会失败 |

## 6. 需要用户裁决的事项

**6.1 listener 持有期**（替代实施方题 4 的 A/B/C/D）

| 案 | 能力 | 调用者负担 | 维护 | 观察/授权窗口 |
|---|---|---|---|---|
| A 现状 | 全 | 失败后死局 | 三态＋门＋契约 | 每命令 |
| C 每 lease | 全 | close 失败仍死局 | lease 持有状态＋closing 门 | 到 lease 结束（≤24h idle） |
| D 每 worker | 全 | 无 | 最少 | 跨 Gateway OFF/close，直到 worker 退出 |
| **E 单例 Binder＋每命令＋失败不设门（推荐）** | 全 | 无 | 少于 A/C；删三态、门、exit 覆写 | 每命令（失败时到补注销或 worker 死） |

推荐 E。未知：同一 Binder 多次登记后一次注销是否全部移除，需设备确认 `dumpsys notification` 里 listener 计数；源码 14/15/16 都支持这一点。

**6.2 通知写范围**：推荐保留 action/open/dismiss/reply，附带 §2.1 的条件。reply 发布前做一次模拟器 Messages 验证。用户如果不接受任何真实外发路径的验证，可以先发布 reply 并在文档注明"仅本地接受已验"，或者暂缓 reply；action/open/dismiss 不受影响。

**6.3 版本地址默认**：推荐"含义检查默认、postTime 可选、reply 用内容指纹、单串选择"。备选是现严格默认，代价是进度/媒体类通知几乎无法操作。

**6.4 非 Activity 型 PI 与 dismiss 是否免主屏就绪门**：免掉以后，锁屏或熄屏时也能"标为已读/暂停/清除"；代价是真人不在场时也会发生后台副作用。这本来就在 full shell 能力之内（`am broadcast`）。我倾向免门，Activity 型继续保留门。这是新的用户取舍。

**6.5 virtual 会话里的 open**：可以 (a) 落到目标 VD（需设备证实 sender options 生效），(b) 拒绝并提示切 main，(c) 保持现状但写进文档。推荐 a，a 不可行就 b。

## 7. 会改变取舍的验证问题

| 验证 | 依赖它的判断 | 结果 → 选择 |
|---|---|---|
| 单例 Binder 注册 2 次、注销 1 次后，`dumpsys notification` 的 live listener 是否为 0 | E 案可行性 | 为 0 → 选 E；不为 0（某 ROM 改了 ManagedServices）→ E 加"注销到成功前不再注册"的一行门，仍优于 C |
| PI `sendAndReturnResult` 带 `ActivityOptions.setLaunchDisplayId(VD)` 时，Activity 是否落在 shell 拥有的 VD 上 | §6.5 | 落在 VD → 选 a；被忽略或抛错 → 选 b |
| Gateway FGS 通知能否被 `cancelNotificationsFromListener` 取消 | 自身保护范围 | 能 → 写命令按自身包拒绝必须做，否则 D7 修好后控制者通知会被藏掉；不能 → 只需 open 护栏 |
| 常见 App 在 30 s 内同 key 重发且内容不变的频率（进度、媒体、聊天各取一个） | postTime 默认 | 常见 → 内容指纹/含义检查默认；罕见 → 严格默认代价小，也可以保留 |
| 模拟器镜像里 Messages 外发 SMS 是否只在仿真 modem 内回环（可用 `adb emu` 控制台和第二个模拟器端口观察） | reply 能否安全补证 | 只在仿真 modem 内 → 补一次真实 App reply 验证后保留；不是 → 维持"仅本地接受已验"，或由用户决定暂缓 |
| POSIX 上修好 6 处 mkdir 和 ENOTDIR 后全量 pytest | "候选已验收"表述 | 全过 → 可称 host 多平台通过；仍有失败 → 继续定位 |

## 8. 未核实与局限

- 没有设备，§7 所有项都没跑。实施方设备原件只读了结果页，没有逐字节核对 stdout/trace。
- Gateway FGS 通知在 A14+ 是否带 `FLAG_ONGOING_EVENT`（AMS 侧）没有核对。
- `PendingIntent` sender 的 `launchDisplayId` 在 A14–16 `PendingIntentRecord`/`SafeActivityOptions` 中的合并规则没有读上游，§6.5(a) 只是推断。
- 没读原报告 A/B 全文，只读了 b-disposition 和实施方 §6。抽查 10 项，不是全量。
- 时延统计（0–3 ms）来自实施方各阶段 receipt 的 grep 汇总，没有区分 API 版本或设备。
- 主屏就绪门的原始设计理由没有在 `T/decisions.md` 里全文检索确认。
