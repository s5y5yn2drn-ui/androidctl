# 第五轮子评审 51：只读通知、图像（截图、区域截图、标注、坐标资格）与 worker 资源生命周期

评审对象：产品 `cab84b8`（只读克隆 `S/r5/prod`），实施方材料包 `PKG`（下文 `ST` = `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`）。AOSP 对照文件下载到 `S/agents8/51/aosp/`（标签 `android-14.0.0_r1`、`android-15.0.0_r1`）。所有设备事实来自实施方在 Android Studio API34 模拟器和 MuMu 上的原件，没有真机。

---

## 1. 结论摘要

这一块整体是干净的：通知写路径删得彻底，图像标注改成局部点阵后实现正确而且短，区域截图不再留存原生像素，worker 侧资源的持有和释放链路清楚。问题主要出在**验收方法**，不在代码：第四轮第 6 项给"区域截图"和"图像标注"定的正确率门量错了东西。实施方在这里投入了大量准备工作，产出的结论按设计本来就不可能通过。

最重要的发现：

1. **通知删除完整。** 产品仓、docs 仓、wire 契约、Kotlin、主机端、测试和指引里都找不到写路径残留，旧子命令在发出 RPC 之前就被拒绝。【源码】【实跑】
2. **"Android 15 以上死亡回调累计"这个说法定位不准。** 按 AOSP 源码，Android 14 和 15 上 `unregisterListener` 都只把条目从列表里移除，从不调用 `unlinkToDeath`，所以每登记一次，system_server 里就留下一个死亡接收者，直到 worker 进程退出。Android 14 上实施方实测过这种增长（每个周期约 2 个强引用根，worker 退出后清零）。Android 15 改成 JNI 只持弱引用、由 Java 侧列表持有，**多半反而能被 GC 回收**（推断）。累计量按"用过通知读取的租约数"计，不按读取次数计，正常使用下可以忽略。不值得再投入测量。【源码】【推断】
3. **区域截图十道题两臂都是 10/10，这个结果在设计时就已注定。** 两臂都拿到了完整原生 PNG，也都能在本地裁图；产品的区域截图只是同一分辨率原生像素的一块，不会多出任何信息。不用区域截图的一臂确实是用 PIL 或 System.Drawing 在本地裁图、最近邻放大后读出来的。区域截图的独立价值只在两种条件下存在：调用者没有本地裁图手段，或者原生整图超过模型的图像输入上限、会被缩小。这项能力应当保留为便利能力，但撤掉正确率门。【原件】
4. **标注的正确率门量错了东西。** 标注不会让图更清楚，它的作用是把"看到的东西"对应到可操作的 REF。在野牛相册那组题里，不用标注的一臂直接给坐标就全对了；湿地相册找不到"看不清"的题，正说明这个出题前提本身就和标注无关。标注真正的价值和第 6 道待决题（按截图坐标点击这条路保留多少）直接挂钩。【原件】
5. **标注有一个能误导人的实际缺陷：容器的标签占住了第一格子项的位置。** 湿地网格的实拍图里，左上第一张照片的左上角写着整个网格容器的 REF（`e797305192`），照片自己的 REF 被挤到这一格底部。调用者照着最近的标签去点，会点到容器上。只要按面积从小到大排序再放标签就能修好，大约 1 行。【原件】
6. **U31 资源清理记录如实。** 两次删除主机磁盘都在创建进程前被自动审批拒绝，原文、命令和"未执行"的状态都保留了下来。MuMu 第 2 个实例按精确身份删除，实际释放了约 26.3 GB。但实施方又**保留了一个 26.3 GB 的归档**，旧 API34 快照和 Play AVD 的数据盘也还在，加起来到底占多少空间没有汇总到一处，需要用户本人处理。【原件】

对待决题的看法：本专题和"坐标点击 A/B/C/D"有关。标注是"按图找到目标，再用 REF 操作"的桥梁。坐标点击能力越弱，标注就越该做成主路径，并按调用成本来评价，不该按正确率评价。

---

## 2. 确定缺陷

### 必须修（中）

**D1 标注的标签放置让容器抢占了子项最自然的标签位置。**
- 触发条件：`snapshot --screenshot --marks`，而且一个可公开的容器（网格、列表）和它的子项都在 40 个标注名额里。
- 位置：`ImageMarks.kt:51-78` 按源节点顺序依次放标签，候选位置是左上、右上、左下、右下四个角，取第一个不和已有标签相交的。父节点在先序遍历里排在子项前面，所以容器先占走左上角，而网格容器的左上角恰好就是第一个子项的左上角。
- 原件：`ST/runtime/api34/images/wetland-natural-default-marks.jpg`（我直接看过）：第一张照片左上角是 `e797305192`，底部是 `e797305193`；2158 号命令点开第一张照片实际用的是 `e797305193`。实施方在 `ST/marks-preparation.md` 里也写了"第一格实际照片标签仍在该格底部，上方另有整网格容器标签"。容器的身份我是根据实施方的这句说明认定的，完整树不在包里，没有逐节点核对。
- 影响：调用者照着"离照片最近的标签"去操作，会把容器当成照片。这正是标注最该防止的误操作。
- 最小修正：放标签前按框面积从小到大排序（子项先放），或者框里还包含其它已标目标的容器就不标。改动 1 到 5 行，再加一个 JVM 用例。

### 可选

**D2 单独执行 `screenshot --marks` 时这个选项接受了却什么也不做。** `screenshot` 不采节点，`UiPublication.kt:63` 传入的观察是 `null`，于是标注数为 0，只给出原因 "No associated node observation"。`--help` 却写着 "Label public REF targets on the requested image"。【源码】【实跑：`androidctl screenshot --help`】指引 `images.md:56` 写了这一点，但调用者很容易踩中。修正：在 `screenshot` 上不带 REF 时直接拒绝 `--marks`，或者从这个命令上删掉这个选项（约 5 行）。

**D3 USER_ALL 通知的 key 以 `-1|` 开头，不加 `--` 会被当成选项解析。** 【实跑】`androidctl notifications get '-1|android|55|null|1000'` 报 `No such option: -1`。实施方自己在 230 号命令遇到过，`notifications.md` 却没有写。修正：指引里加一句 `notifications get -- KEY`，或者让 Typer 接受以 `-` 开头的参数。

**D4 通知投影里有不需要的反射。** `NotificationOperations.kt:138`（文件内行号 215 附近）用 `HiddenApi.method(UserHandle, "getIdentifier")` 反射取用户号，但 `UserHandle.getIdentifier()` 从 API 24 起就是公开接口，而最低版本是 34。可以直接调用。【源码】

**D5 两个标签互相重叠时的兜底是"直接放左上角"**（`ImageMarks.kt:71` 的 `?: corners.first()`）。实施方承认右侧标签会互相重叠；WPS 那张图不在包里，无法核实。D1 修好后影响会减小。REF 太长（`e` 加 9 到 10 位随机起点数字，`UiRefs.kt:8`），每个标签在 2 倍像素下宽约 120 px，这是重叠的主要原因，见 §3.2。

没有发现资源泄漏类的确定缺陷，见 §3.4。

---

## 3. 设计与简化

### 3.1 只读通知

**形态。** 每个租约在第一次 `list/get` 时用 `INotificationManager.registerListener` 登记一个 shell 组件名下的系统监听器（`NotificationAccess.kt:36-58`），通过 `getActiveNotificationsFromListener` 读取，租约结束时在 UI executor 上注销（`UiBackend.kt:575-585`）。同一个 Binder 对象跨租约复用。

**为什么一定要监听器。** 公开的 `getActiveNotifications` 在 NMS 里要求 `ACCESS_NOTIFICATIONS`（AOSP 14 `NotificationManagerService.java:4350-4355`），shell 的 Manifest 里没有这个权限，只有 `STATUS_BAR_SERVICE`、`MANAGE_NOTIFICATIONS` 和 `MANAGE_NOTIFICATION_LISTENERS`（AOSP 14 `packages/Shell/AndroidManifest.xml:285,436,554`）。`registerListener` 只检查 `enforceSystemOrSystemUI`，而 `STATUS_BAR_SERVICE` 正好满足这个检查（`NMS:4534-4537`）。所以以 shell 身份拿到结构化通知对象，唯一的办法就是登记监听器。【源码】实施方早期也做过这组权限对照，但材料只在 `stages/notifications-plan/plan.md:135` 留了一句，原件我没有核到。

**规模。** 生产代码约 627 行（Access 77、Operations 237、Projection 313），另有 `HiddenApi.kt` 约 25 行、`UiBackend`/`WorkerService`/`RpcMethods` 的接线约 20 行，主机端约 60 行；测试 764 行（Kotlin 518、Python 246）；公开指引 56 行；wire 契约 `notificationResults` 一节约 5.8 KB。用到的隐藏成员大约 9 个：两个 AIDL 类及各自的 `Stub.asInterface`、`registerListener`、`unregisterListener`、`getActiveNotificationsFromListener`、`ParceledListSlice.getList`、`NotificationListenerService.TRIM_FULL`，还有一个手写的 `INotificationListener` Binder。

**死亡回调累计（实施方所说的 Android 15 以上缺口）。** 推理过程如下，Android 14 部分有实测支持：
- `ManagedServices.registerServiceImpl` 会调用 `linkToDeath(info)`（A14 `:1709`，A15 `:1789`）；`removeServiceImpl → removeServiceLocked` 只做 `mServices.remove` 和 `onServiceRemovedLocked`，**两个版本都不调用 `unlinkToDeath`**（A14 `:1670-1690`，A15 `:1750-1770`）。【源码】
- Android 14 的 JNI `JavaDeathRecipient` 持有全局强引用（`android_util_Binder.cpp:565-576`），所以每登记一次，就有一个 `ManagedServiceInfo` 加一个 JNI 全局引用留在 system_server 里，直到 worker 的 Binder 死掉。实施方以前在 API34 实测过"2 个周期，强引用根从 2 增到 6，worker 退出后归 0"（`S/pkg6/root/.../reviews/route-r2-notifications-mumu-runtime.md` 里转述的 API34 结论）。我核到的是这句转述，没有核到它的原始堆转储。
- Android 15 在编译开关 `BINDER_DEATH_RECIPIENT_WEAK_FROM_JNI` 打开、并且接收方目标版本至少为 V 时，JNI 只持弱引用，改由 `BinderProxy.mDeathRecipients` 这个 Java 列表持有强引用（A15 `android_util_Binder.cpp:575-597`，`BinderProxy.java:620-641`）。注销之后，`ManagedServiceInfo` 和 `BinderProxy` 之间只剩一个纯 Java 环，能被 GC 回收。【推断】也就是说，Android 15 很可能比 14 **更好**，不是更差。实施方在 MuMu 的 Android 15 上取堆被拒（`Process not debuggable`），这个缺口确实测不到，但它的方向标反了。
- 影响有多大：每个"读过通知的租约"留下约 1 个接收者（A14 上约 2 个强引用根），通知不能自动修复的那种"未知监听器"再加 1 个。system_server 的 JNI 全局引用上限是几万量级，要几万个这样的租约都没有重启 worker 才会接近上限。对长期运行的 worker 来说，正常使用下影响可以忽略。【推断】

**有没有更简单的只读实现。**

| 做法 | 代价 | 得失 |
| --- | --- | --- |
| 现在：每个租约登记一次，结束时注销 | 每个用过通知的租约累计 1 个接收者；还有注销失败诊断（`notificationCleanup` 字段、指引两段、测试） | 结构化对象、空闲时不登记 |
| 每个 worker 只登记一次，永不注销 | 删掉 `endLease`、`cleanupFailure`、状态字段及相应文档测试，约 40 行加文档 | 累计恒为 1；worker 存活期间一直挂着一个系统监听器，NMS 对每条新通知发一次 oneway 回调。SBN 用 holder 传递，监听器不取就不会搬运正文。代价很小 |
| 每条命令登记、读取、注销各一次 | 累计量随读取次数增长 | 比现在更差 |
| `cmd notification list/get KEY` 或 `dumpsys notification --noredact` | 不需要监听器和隐藏接口；但输出是 `NotificationRecord.dump` 的文本，没有稳定格式（AOSP 14 `NotificationShellCmd.java:323-338`） | 换来解析脆弱、跨版本漂移、拿不到动作和意图的结构；不推荐 |

建议：保留监听器。**把"Android 15 以上累计"从未关闭缺口里移出**，改写成一句已知性质：每用过一次通知的租约在系统进程里留一个死亡接收者，worker 退出时释放。注销失败诊断可以考虑简化成一条日志（失败的后果只是多留一个监听器直到 worker 退出），这属于可选项。

**投影过度。** `NotificationProjection` 加上字节预算机制约 300 行，为了 192 KiB 的回复上限逐字段记录省略了什么（UTF-16 长度、数组起点和数量）。实测 23 条通知的 `list` 只有 19 KB（1434 号命令）。另外，动作的 `PendingIntent` 创建者、类型、是否不可变，以及 RemoteInput 的 `resultKey/allowedDataTypes/choices`、`semanticAction`、`allowGeneratedReplies`、`contextual`，都是为已删除的写路径服务的事实，只读调用者用不上：调用者看到动作标题，最多也只能去通知栏里点。实施方内部 review 说"不能以写入口消失为由继续删除"，我不同意。建议只留 key、包名、postTime、标题、正文类字段、分类、渠道、分组和动作标题；预算改成简单的"每字段截断加条数上限"。可以减掉约 150 到 200 行生产代码、相应测试和 wire 样本，能力几乎无损。这条属于可选。

**真实任务里用到了吗？** 70 次真实任务的 `cli-index.json` 里**一次也没有**调用 `notifications`【重算：`S/agents8/51` 中用一次性脚本遍历 `ST/efficiency/*/cli-index.json`，没有任何 argv 含 `"notifications"`】。只有 Feeder 那组专门验证（1432–1450）。那组的通知内容是实施方新建的测试订阅 "R4 notification test 20261122" 经正常同步产生的，不是用户的自然业务。读取能力本身已经证明可用：主屏和后台屏两个租约里，同一个 key 和 postTime、两个动作都完整返回，没有搬动 Feeder 的任务。【原件】

### 3.2 图像标注

- 实现：`82318e9` 删掉了改写进程全局字体表的 `ensureSystemFonts()`，换成只含 `e` 和 0–9 的 5×7 点阵；`93fd40d` 只是把 `save/translate/restore` 换成 KTX 的 `withTranslation`。点阵字形逐个核过，没有错字；不开抗锯齿、按 2 倍整数缩放，所以字形清晰；`Canvas.setBitmap(null)` 放在 finally 里。整个 `ImageMarks.kt` 196 行，正确、简洁。第 6 项要求"先试公开字体接口"，实施方做了：API34 上两条公开构造路线都在 `CustomFallbackBuilder.build` 处抛 NPE（`ST/font/README.md`，140 号命令）。实施方遵守了决定，没有多余投入。【源码】【原件（转述）】
- 可读性：两张包内实图（Contacts 空页、湿地网格）的编号都清楚可辨。主要问题有两个：一是上面 D1 的容器抢位；二是**标签太长**。REF 是 `e` 加 9 到 10 位数字（随机起点 `UiRefs.kt:8`），每个标签约 120×20 px，在 720 px 宽的图上会盖住照片的一大块，右侧也会互相挤。
- 更简单、更好读的做法（设计建议）：图上画**每张图内的短序号**（1–40），把序号和 REF 的对应放进 `data.image.marks.labels`，指引里写"按序号查 REF"。标签宽度能降到原来的约 1/4，重叠和遮挡大幅减少，点阵只需要数字。代价是调用者要多查一步（对应表在同一个回执里，几乎不增加调用成本）。这需要改 wire 和指引，大约 40 行。是否值得做，取决于 §5 的坐标决策。

### 3.3 区域截图与图像句柄

**图像句柄的生命周期（源码）。** worker 只保留**一个** `ImageState`（`VisualInput.kt:21-49`），内容是最近一次完整截图的元数据和布局 JSON，大小是 KB 级，**不保留像素**：源 Bitmap 和缩放、标注用的临时图都在编码后的 finally 里回收（`:113-116`、`:175-178`、`:276`）。ID 是 `i` 加计数器，区域截图是 `r` 加计数器，区域截图的 ID 不能再被引用。

失效条件：
1. 再取任何一张完整截图，包括动作附带的 `--screenshot`，旧的就被替换。
2. 租约结束时 `visuals.clear()`。
3. 连接代次改变。
4. 对坐标输入而言，还有几种失效：任何可能已经派发的动作（`UiBackend.kt:169,173,257,261`）、观察签名变化（`:413`）、120 秒期限（`VisualInput.kt:454`）、显示形状、窗口、锁屏或放大状态改变。

区域截图只检查租约、连接、显示号、尺寸和旋转，没有时间期限。数量上限恒为 1，内存只有元数据。这比上一代"留存原生源像素"的契约简单很多，是一项正确的简化。【源码】实施方测得的 RSS 和高水位数字（`ST/region-result.md` 末节）如实标注了"不能推出释放量"。

**"坐标输入必须引用图像句柄"不完全准确。** `--image` 在 CLI 和 worker 上都是**可选**的（`ui_commands.py:1312-1348,1374-1390`，`VisualInput.kt:290`）。真正的前提是：存在一张当前有效、尚未失效的完整截图。调用成本是第一次坐标操作前需要多一次截图；之后可以用动作自带的 `--screenshot` 把新图接上（`UiPublication.kt:182-186` 会发布并重新认定这张图）。这个成本可以接受。

**区域截图十道题。**
- 题目是 MuMu 上 WPS 打开的 Arduino UNO 原理图第 1 页，问的都是小字：F1、U1、U2、T1、串阻、Z1/Z2、C5、谐振器、L2、I²C 脚号（`ST/region-preparation.md`）。原生分辨率 1080×1920，默认预览是 720×1280 的 JPEG，质量 80（`ImageCoordinates.kt:18-25`，`VisualInput.kt:456`）。我看过默认图和 MCU 区域图：默认图上这些小字确实读不出，原生区域图勉强能读。
- **两臂都拿到了完整原生 PNG。** 不用区域截图的一臂实际调用了 `exec_command`：用 PIL 和 System.Drawing 从完整 PNG 里裁出 usb/power/mcu 三块，还用最近邻插值放大到 640×640 再看（`ST/efficiency/region-without-01/native.rollout.jsonl.review.jsonl`，序号 19、26、40、58）。产品的区域截图等于"同一分辨率下的原生裁片"，信息量是完整 PNG 的子集。所以"至少一题只有用了才对"**按构造就不可能成立**。【原件】
- 成本方面，用区域截图的一臂总输入少约 15%，墙钟少约 26%，但未缓存的输入加输出反而多；各只跑了一次，不能下结论。
- **独立价值成立的条件**（设计判断）：(a) 调用者没有本地图像库，或者不能执行代码；(b) 原生整图超过模型的图像输入上限会被下采样，比如 1440×3200 的真机，或平板大屏，这时完整 PNG 送进模型并不比预览清楚多少，本地裁图或区域截图才能拿到原生细节；(c) 想用很少的图像 token 只看一小块。这三条都和"默认预览被压缩"有关，但关键在调用者有没有裁图手段，而不在默认预览本身。
- **处置建议**：保留区域截图，作为 `screenshot` 上的便利参数。实现已经很薄：`ImageRegion.kt` 208 行，加上 `VisualInput` 里约 95 行。**撤掉第 6 项给它定的正确率门**，不再为它设计实验。指引里写清楚它等于"PNG 原图的一块"，并说明什么时候用它，什么时候直接取 PNG 在本地裁。不建议删除：`screenshot REF` 一次调用就能拿到某个元素的原生像素（图标、验证码、小字），这对没有本地工具的调用者有实际价值。
- 一个语义上需要注意的点：区域截图采的是**新像素**，坐标却取自旧图。中间内容滚动或动画变化后，裁出来的可能不是调用者当初看到的东西，回执只标了 `mappingKnown=false`，不告诉调用者内容是否变了。指引里写了（`images.md:75-76`），可以接受。

**标注实验。**
- 野牛相册 10 题两臂都全对。不用标注的一臂直接给出 240 px 网格的格心坐标，例如 `(360,536)`，这个答法被计为正确（`ST/efficiency/marks-without-01/first-final.json`）。用标注的一臂给的是 REF。两臂的调用次数相同（3 轮、2 次工具调用），token 用量也接近。【原件】
- 湿地相册 19 张照片，实施方说默认网格上就能分辨鱼、喙色和姿态，凑不出约十道"默认看不清"的题，所以没有开正式测（`ST/marks-result.md` 末节）。我看过 2157 号命令的标注图，同意"看得清"。
- 这说明什么：**"默认预览看不清"这个出题前提对标注完全没有意义。** 标注不提升清晰度，它解决的是"从像素对应到 REF"。实施方按字面执行第 6 项，花了两轮准备（野牛 10 题冻结和判分、湿地 19 张原件导入和 10 道候选题私有判分，还有方法 review），去找一种按定义就不该存在的题。这是典型的钻牛角尖，源头在第四轮第 6 项把区域截图和标注合并成同一条"正确率加读不清"的门。
- 正确的量法（设计建议）：在**目标无名、靠图辨认**的真实页面上（照片网格、地图标注、无文本图标）比较两臂完成同一操作任务的调用次数、token 和误操作数。"不用标注"一臂允许坐标点击，也允许读 bounds 自己对应。如果第 6 道待决题把坐标点击收窄（比如桌面上被拒），标注就是图像到操作的主桥梁，价值会直接体现在成功率上。

### 3.4 worker 侧资源生命周期

| 资源 | 持有者 | 粒度 | 释放点 | 异常路径 |
| --- | --- | --- | --- | --- |
| UiAutomation 连接和回调线程 | `UiBackend.connectionCleanup` | 租约 | `clear(lease)` → 在 executor 上 `clean()`（`UiBackend.kt:575-625`） | 失败进入 `pendingCleanup` 和 `uiCleanup` 诊断，新的 UI 命令被 `requireCleanUi` 拒绝；由 `forceExecutor` 兜底 |
| 后台屏（VirtualDisplay 和取像流） | `Lease.target.stream` | 租约 | 同上，`pending.stream.close` | 同上 |
| 节点缓存、REF 表 | `snapshots` | 租约 | `snapshots.close(null)` | 在 `attempt` 内执行，记入错误 |
| 图像元数据（不含像素） | `VisualInput.image` | 租约，最多 1 个 | `visuals.clear()` | 同上 |
| 通知监听器 | `NotificationOperations.access` | 跨租约复用，每个租约登记一次 | `clear` 的 finally 里调 `endLease` | 失败写入 `notificationCleanup`，不阻塞后续命令 |
| WindowInfos 监听器（坐标点击预测窗口归属） | 本条命令 | 命令 | `WindowInputListener` 的 finally | 失败转成 `exit=CLEANUP_UNCONFIRMED` |
| Bitmap | 本条命令 | 命令 | 编码后在 finally 里 `recycle` | 抛异常时同样回收 |

到期由周期任务 `sessions::expire` 驱动（`WorkerService.kt:40`），默认 30 分钟，并且只在没有在途资源时触发（`Sessions.kt:290-296`）。releaseLocal、revoke、替换、到期都走 `invalidateLease → clearLeaseUi → ui.clear`。我没有发现会泄漏的路径。【源码】

**关于"资源计数归零"这条证据要说清楚。** `status` 里的 `ui/shell/file` 计的是 `Sessions.resources`，也就是**在途命令**的预留，不是租约持有的屏幕、连接和监听器。在两条命令之间它本来就是 0：1434 号命令运行时，控制者通知里就显示 `UI: 0 / Shell: 0 / File: 0`，那时租约还开着。所以"关闭后计数归零"（1449、2172）只能证明没有在途命令；它本身不证明后台屏和监听器已经释放。真正的释放证据是另外几样：`session close` 回执里的 "UI cleanup confirmed"、`diagnostics` 为空、1450 号命令 `dumpsys display` 只剩 display 0，以及更早 API34 上 227/229/232 号命令 `dumpsys notification` 里监听器出现又消失（`com.android.shell/com.rainng.androidctl.g4`，5 个变 6 个再变回 5 个）。这些原件我都看过，都成立。实施方的文字引用资源计数时没有越界，只是读者容易误会。【原件】

---

## 4. 证据审计

| 实施方声称 | 判定 | 依据 |
| --- | --- | --- |
| 通知写入口、选择串、守卫、落点、隐藏发送和取消成员已完整删除 | 成立 | 两个仓库全文检索没有残留；`notifications open` 本地实跑报 `No such command`；wire 契约里只剩 `notificationResults`【源码】【实跑】 |
| 每租约一个监听器，正常 close 后注销 | 成立（API34） | 227/229/232 号 `dumpsys notification`【原件】 |
| Feeder 真实通知在主屏和后台屏同一个 key、postTime、正文，没有搬动任务 | 成立；但通知来自实施方新建的测试订阅 | 1434–1443 号命令【原件】 |
| "Android 15 以上通知死亡回调累计释放未知" | 部分成立：确实没测到；但"只有 15 以上才是缺口"的提法不对，Android 14 的累计已经实测，Android 15 的机制多半更好 | §3.1【源码】【推断】 |
| 区域截图两臂 10/10，门没过，不自动降级 | 成立；但这个门按设计不可能通过 | 两臂的 `first-final.json`；不用一臂的轨迹显示本地裁图【原件】 |
| 区域截图不保留原生像素，到期或关闭后旧图引用失效 | 成立 | `VisualInput.kt`；2174 号命令 `staleImage`【源码】【原件（转述于 review，并核对了源码）】 |
| 标注：湿地找不到合格题，不启动正式测 | 事实成立；结论的含义被实施方当成"验收缺口"，我认为应理解为"这道门本身不适用" | 2157 号命令的图【原件】 |
| 点阵标签"多处完整编号可读，右侧互叠" | Contacts 和湿地两张图里可读成立；WPS 那张 `199-pure823-wps-home-marks.jpg` 和野牛对照图 `fossify-pair-with.jpg` 都不在包里，核不到 | — |
| 资源计数为 0 且诊断为空 | 原件成立；含义见 §3.4 | 1449、2172 号命令【原件】 |
| U31：主机磁盘清理两次被拒，未执行，未换工具绕过；source2 已删，释放 26.3 GB | 成立 | 两份拒绝记录、jsonl 原文、`source2-retirement.json`【原件】 |
| 保留 26.3 GB 归档"供后续复验" | 如实记录，但和 U31 的精神有冲突，需要用户确认 | `ST/runtime/mumu/source2-retirement.json` 的 `archiveRetained` |
| 内存：高水位 138968 到 178376 kB，区域截图和 close 后不再增加 | 原件数字与表一致；解释留有余地，没有冒称释放 | `ST/region-result.md`；2148/2166/2169/2171 号命令的转述（review 核过，我没有逐个打开） |

关于测试：`NotificationReadTest` 用替身直接测 `NotificationOperations` 的复用、修复和注销失败，这是在验证行为。`ImageMarksTest` 只有 5 个用例，覆盖规则和元数据，**没有断言任何像素和标签布局**，所以 D1 这类问题测试抓不到（`marks-canvas-review.md` 自己也承认）。

---

## 5. 对相关待决题的评价

"坐标点击 A/B/C/D"（第四轮缺陷 M6：桌面上壁纸窗口也接收触摸，导致按坐标点击被拒）不归我审，但它决定了标注的定位：
- 如果坐标点击保持通用可用，标注就是锦上添花，可以保持现在的规模，修好 D1 即可。
- 如果坐标点击收窄（只允许单一 UID 的应用窗口，或者桌面上不可用），"看图选目标"就只能走标注转 REF 这条路。这时值得把标注改成短序号（§3.2），并按调用成本做一组对照。
- 实施方给出的选项里没有把标注列为替代路径。我建议在裁决坐标题时，明确把"标注加 REF"写成被拒场景下的推荐路线，写进指引。

---

## 6. 需要用户裁决的新取舍

**取舍 1：区域截图和图像标注的验收标准（推翻第四轮第 6 项里这两项的正确率门）。**
- A：维持现在的门（正确率，且至少一题只有用了才对）。代价：区域截图在"调用者能本地裁图"的前提下按构造不可能通过；标注的出题前提不成立。实施方会继续空转。
- B：撤销这两项的正确率门，改成"能力冒烟（一次真实成功）加实现保持精简"，不再设计专门实验。推荐。
- C：改用调用成本门：在无名目标、靠图辨认的操作任务上比较调用次数和误操作次数（标注）；在"调用者不能执行本地代码"的条件下比较（区域截图）。代价是又要设计实验，只有在坐标点击被收窄时才值得。
- D：降为实验或删除。代价：区域截图约 300 行、标注约 200 行可以删掉；失去"没有本地工具时看原生细节"和"按图转 REF"两种能力。
- 推荐 **B**；坐标题如果裁成收窄，标注再按 C 补一组。剩余未知：大屏真机上，模型图像输入上限对完整 PNG 的实际影响。

**取舍 2：通知投影是否收缩到只读需要的字段。**
- A：维持现状（含意图、RemoteInput 描述、逐字段预算省略）。
- B：只留身份、文本、分组和动作标题，改成简单截断。减少约 150 到 200 行，删掉部分 wire 样本。推荐。
- 剩余未知：有没有调用者依赖 `PendingIntent` 的类型。真实任务里没有用到。

**取舍 3：通知监听器的登记粒度。**
- A：维持每个租约登记一次（第 3 项现状）。把"Android 15 以上累计"从缺口里移出，改写成已知性质。推荐，零代码改动。
- B：改成每个 worker 登记一次、永不注销。累计恒为 1，并删掉注销失败诊断（约 40 行加文档和测试）；worker 存活期间一直挂着一个系统监听器。
- 两者差别很小，A 不需要动代码。

**取舍 4（U31）：保留的 26.3 GB MuMu 归档和被拒删除的旧快照、Play 数据盘。**
- 需要用户本人决定是否删除，并亲手执行，因为自动审批拒绝了实施方的删除。建议实施方先把剩余待删对象和总字节数汇总成一张表，再交给用户。

实施细节（不需要用户裁决）：D1 标签排序；D2 `screenshot --marks` 的拒绝；D3 指引里补 `--`；D4 去掉反射。

---

## 7. 没能核实的内容

- Android 15 上死亡接收者到底是弱引用还是强引用，取决于编译开关和接收方目标版本，需要设备才能确认。API34 上"2 增到 6"的原始堆转储我只看到转述。
- `fossify-pair-with.jpg`、`199-pure823-wps-home-marks.jpg` 不在包里，"右侧标签互叠"的程度和野牛那组标注图的版面无法直接核对。
- 湿地标注图里 `e797305192` 是容器这一点，我依据的是实施方说明和 2158 号命令的点击对象；包里没有 2157 号命令的完整树。
- 实施方转述的 2148/2166/2169/2171/2174 号命令内存和失效原件，我没有逐个打开。
- 主机侧剩余待删对象的总大小（旧快照 ram.bin、Play 数据盘等）没有汇总，我无法计算。
- 没有设备，以上所有设备行为都来自实施方原件。
