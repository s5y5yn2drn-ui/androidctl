# 专题 43：整轮候选的规模、架构、维护责任、测试质量与提交历史

评审对象：产品仓库从主分支基线 `7d5be70` 起的 68 个提交，到最终候选 `4235af9`；中间点是旧候选 `3f45af6`（前 40 个提交）。另有独立 docs 仓库（`9b8c5bd`，65 个提交）。
本专题全程只读。所有脚本都在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents7/43/` 下，用 `python3 -I` 运行，读取 `git show 版本:路径`，不碰工作树。

术语：
- **E9**：候选里手写的无障碍回调采集器（`CollectionNodes.kt`、`CollectionReplies.kt` 等）。它不经公开 `getChild`，而是反射调用隐藏的 `IAccessibilityServiceConnection.findAccessibilityNodeInfoByAccessibilityId`，自己实现回调 Binder 并解析回包。
- **后台屏 / VD**：由 worker 创建的 virtual display，用作"工作屏"。
- **四臂**：实施方本轮最后做的 4 次真实任务对照，Maps 和 GitHub 各一对，每对是"新呈现"与"旧呈现"。
- **wire 契约**：`contract/wire-contract.json`，Kotlin 端生成、Python 端消费的共享样本文件。

---

## 1. 结论摘要

**整体状态。** 这一轮名义上是"删除、合并、简化"，实际是一次重构。产品主源码又净增约 2,150 行，同时进程内状态、隐藏接口、wire 契约、指引都在增长。拿一个人维护的 alpha 产品来衡量，最终候选的规模和机制**已经明显超出需要**。根因不在实施质量，而在前提：用户此前确认的"能力全部保留"。在这个前提下，实施方和评审方都只能在每项能力内部做局部收缩，没有人能删掉一整项能力。

最重要的发现：

1. **规模总账（重算）。** Kotlin 主源码 9,360 → 18,226 → 20,278 行，主机源码 8,008 → 11,959 → 12,067 行，指引 1,254 → 2,258 → 2,663 行。
   - 本轮新增的部分：`@Volatile` 字段 36 → 46，`synchronized` 150 → 165，类级 `var` 字段 182 → 203。
   - HiddenApi 成员 65 → 70，另加两个编译期平台桩和一条 R8 保留规则。
   - wire 契约 441 KB → 496 KB，不同键 431 → 499。
   - 每个命令的选项总数 233（基线）→ 891 → 877。`wait` 一个命令有 26 个选项。
   - 本轮 28 个提交中，题目写着 unify/one/share 的那些大多是加得比删得多，例如 `826c180` 加 500 行、删 236 行，`9f80d5f` 加 388 行、删 200 行。
2. **真实使用证据集中在小核心。** 本轮 8 次真实任务共 84 次 CLI 调用【重算 `usage.py`】，只用到 snapshot、click、scroll、wait、fill、find、back、reveal、expand、screenshot。
   - 后台屏、通知对象命令、剪贴板、type/select、batch、`--diff-from`、timeline/pinch/drag 都是 0 次。
   - 上一轮 12 条流程 199 次命令里，这些能力也几乎是 0 次（batch 3 次，editor 各 4 次）。
   - 这些零使用能力加起来约占新增代码的一半（见 §3.3 表）。
3. **没有任何一次实测把最终候选和基线比过。** 四臂里的"old"是最终产品打了旧阅读呈现补丁的版本【原件 `ST/16-delivery/efficiency/final-old-source-binding.json`】，不是 `7d5be70`。
   - 两轮四臂的方向互相矛盾：第 0 阶段 Maps 新版更省、GitHub 新版贵 2.4 倍；最终阶段 Maps 新版贵 2 倍、GitHub 新版省 2.7 倍【原件 `T/deliverables/route-rebuild-candidate.md:168-184`】。
   - 整个项目的起点问题是"候选的输入是基线的 1.7–2.2 倍"。这个问题至今没有回答。
4. **隐藏接口的风险比数量看上去低。** 我从 AOSP 下载 14、15、16 和 main 四个版本，逐条比对了 21 个关键签名，全部未变。唯一例外是 `ITrustManager.isDeviceLocked` 的第二个参数，在 Android 15 从 displayId 改成了 deviceId，代码已经处理。
   - 真正的维护负担不在签名漂移，而在三个地方：手写 Binder/Parcel 回调、为字体和设置项写的绕路代码、只能在模拟器上验证的平台行为。
5. **测试本身质量尚可，但保护面错位。** 主机测试多数验证可观察行为。Android JVM 测试多是"JSON 进、JSON 出"的纯逻辑测试，大约三分之一耦合内部状态。
   - 真正脆弱的平台集成层没有 JVM 测试，包括 `UiBackend`、`TreeCollector`、`WorkDisplay`、`NotificationAccess`、`EditorBridge`、`TargetManagement`。
   - wire 契约是真的两端对拍：Kotlin 生产代码生成，Python 生产代码消费。我实跑 Python 一侧 194 项全部通过。
6. **质量门的分量不大。** 首轮退出码 201 的原因是环境：PATH 里有一个游离引号，导致 Android 测试执行器启动不了。各组成部分分别都过了。
   - 我在 Linux 上补跑了 ruff、format，以及 mypy 的 native、linux、darwin、win32 四个平台分支，全部通过（也就是 `task ci` 缺的那部分 Python 检查）。
   - 剩下的真实空缺：最终树没有重新构建 APK/R8；macOS 没有原生运行。

**对第四道待决题（提交历史形态）的看法。** A、B、C、D 都默认"整体合入现有范围"，这个前提本身应先推翻。建议先定合入范围（见 §6 第 1 项），再从最终树按子系统生成 6–10 个提交（选项 E）。实施方推荐的 C（从基线按最终行为全面重切）在范围未定时做，是白做。

**最该动的三处：**

1. 删掉后台屏会话和通知对象写操作（连同系统级通知监听器）。第一道待决题随之消失。
2. 文本输入、剪贴板、阅读交付只保留一条主路：删 `--diff-from` 与 View 文件解码、batch、结构剪贴板的"空值证明"、timeline/pinch，输入法通道与借用粘贴二选一。
3. 用公开的预取加缓存窗口替换 E9 手写回调采集；docs 仓库收缩成一页架构说明，把"为什么"写回代码注释。

---

## 2. 确定缺陷

本专题偏规模与结构，没有发现新的产品功能缺陷。下列几条是维护层面的确定问题：

| # | 问题 | 位置 | 影响 | 最小修正 | 确定度 |
| --- | --- | --- | --- | --- | --- |
| M1 | 三份描述（公开指引、docs 仓库、代码）已经互相漂移 | 实施方本轮的累计可维护性 review 自己发现了两处必须修的漂移：一处是 View 文件已删除固定基线，docs 和 `batch.md` 仍要求预留它；另一处是会话清理的旧规则和主机显示偏好【原件 `ST/integration-maintenance-review.md` MF1/MF2】 | 调用者会为不存在的文件预留路径；维护者可能重新引入已删除的机制 | 见 §3.5：收缩 docs 仓库 | 原件，确定 |
| M2 | 实施方的可维护性 review 在结构上没法建议删能力 | 同上文件："必要复杂度仍来自已选行为……Optional 为空" | 所有"0/0 关闭"都只在"能力全保留"的范围内成立，不能当作"规模合理"的证据 | 让用户重定范围（§6 第 1 项） | 原件，确定 |
| M3 | 契约再生成流程只写了 Windows PowerShell 版本 | `AGENTS.md` 的 Testing Guidelines 一节 | 项目声称 Linux/macOS 都能开发；496 KB 的契约一轮改了 +3,352/−1,963 行，频繁再生成，非 Windows 维护者要自己翻译命令 | 补一段 POSIX 写法，或者改成 Gradle 任务 | 源码，确定，轻微 |
| M4 | 最终树没有经过 APK/R8 构建 | 质量门只跑了单元测试和 lint【原件 `ST/16-delivery/runtime/final-quality/result.md`】；APK 沿用较早冻结点的产物 | 本轮新增的 `window-input.pro` 保留规则和编译期平台桩，只有发行构建会检验"桩没有进 DEX、回调方法没有被裁掉" | 合入前在最终树上跑一次 `release:android` 并做一次设备冒烟 | 原件，确定；是否真有问题【需设备】 |

---

## 3. 设计与简化

### 3.1 规模总账（重算）

脚本：`43/lines.py`、`43/subsys.py`、`43/cli.py`、`43/cli2.py`、`43/state.py`、`43/codes.py`。我的数和主评审的粗数在 0.2% 以内一致；Android 测试差约 10 行，来自测试目录里的 Java 桩。

| 指标 | 基线 7d5be70 | 旧候选 3f45af6 | 最终 4235af9 | 本轮变化 |
| --- | ---: | ---: | ---: | ---: |
| Kotlin 主源码行 | 9,360 | 18,226 | 20,278 | +2,052 |
| 另有 C、AIDL、平台桩 | 1,050 | 1,058 | 1,099 | +41 |
| 主机源码行 | 8,008 | 11,959 | 12,067 | +108 |
| 主机测试行（`def test_` 个数） | 16,775 | 25,751 | 25,573（638 个，参数化后 1,936 项含 release 工具） | −178 |
| Android 测试行（@Test 个数） | 4,612 | 13,076 | 15,669（518 个） | +2,593 |
| 指引行 / 字节 | 1,254 / 81.6 KB | 2,258 / 147.7 KB | 2,663 / 172.9 KB | +405 / +25 KB |
| CLI 叶子命令 | 47 | 70 | 66 | −4 |
| 不同长选项（含 completion、version、json） | 40 | 72 | 70 | −2 |
| 各命令选项总数 | 233 | 891 | 877 | −14 |
| `wait --help` 字节 | 1,008 | 2,648 | 2,835 | +187 |
| 设备端 RPC 方法与 UI 命令名（`RpcMethods`、通知和剪贴板方法表、`WorkerService` 分派，去重） | 39 | 61 | 62 | +1 |
| wire 契约字节 / 不同键 | 175 KB / 169 | 441 KB / 431 | 496 KB / 499 | +55 KB / +68 |
| Android 拒绝码（`Rejected("x")` 不同值） | 95 | 123 | 129 | +6 |
| 主机错误码（`Fault("x")` 不同值） | 60 | 62 | 61 | −1 |
| HiddenApi 惰性成员 | 51 | 65 | 70 | +5 |
| HiddenApi 之外的反射或手写 Binder 点（文件数） | 1 | 8 | 9 | +1 |
| 编译期平台桩 + R8 规则 | 0 | 0 | 2 + 1 | 新增 |
| `@Volatile` / `synchronized` / 类级 var 字段 | 29 / 118 / 118 | 36 / 150 / 182 | 46 / 165 / 203 | +10 / +15 / +21 |
| Kotlin `const val`（其中调参常量） | 96（57） | 141（64） | 150（66） | +9（+2） |
| 主机大写常量（其中调参常量） | 44（23） | 64（25） | 66（25） | +2 |

**Android 主源码按子系统拆分**（.kt 加 AIDL 和 C；`UiBackend` 算在"会话与目标"）：

| 子系统 | 基线 | 旧候选 | 最终 | 本轮 | 整轮 | 本轮为什么变 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| 会话与目标（Sessions、TargetManagement、WorkDisplay*、UiBackend、Activity*、DeviceWake 等） | 1,162 | 2,625 | 3,091 | +466 | +1,929 | "显式 lease 唯一目标"删了主机偏好，但设备端新增阶段化清理（`AutomationCleanup`、`PendingUiCleanup`）、wake 自有路径、`WorkDisplaySettings` 读设置项，`UiBackend` 从 540 涨到 743 |
| 采集与节点（TreeCollector、Collection*、NodeFacts/Encoding、UiCondition 等） | 2,089 | 3,618 | 3,989 | +371 | +1,900 | E9 快速失败诊断拆出 `CollectionReplies`（173 行）和 `CollectedChildren`（56 行）；稀疏编码收到 `NodeEncoding` |
| 输入（UiActions、Pointer*、Key*、Editor*、Clipboard*、WindowInput* 等） | 1,141 | 4,429 | 4,982 | +553 | +3,841 | 删了 editor 命令组，但新增 WindowInfos 坐标接收者（296 行加桩）、`UiSelection`/`NodeSelection`（214 行） |
| 通知 | 0 | 1,190 | 1,606 | +416 | +1,606 | 原意是"监听器换成一个标志、身份参数换成一个选择串"，实际新增 `NotificationSelection`（193 行），`NotificationActions` 从 478 涨到 589 |
| 图像 | 359 | 807 | 896 | +89 | +537 | 删了 zoom 和跨请求的原生帧驻留，改成"新采只读区域" |
| 结果与交付 | 517 | 1,227 | 1,299 | +72 | +782 | 动作事实与交付失败分开 |
| 连接与配对 | 2,893 | 3,191 | 3,271 | +80 | +378 | 系统服务死亡时退出、控制者通知 |
| 文件与作业 | 2,197 | 2,197 | 2,202 | +5 | +5 | 只修了 push writer 的关闭顺序 |

主机端本轮几乎持平（+108）：`ui_batch.py` 417→317、`workspace.py` 162→66、`history.py` 306→248、`ui_diff.py` 357→266，被新的 `view_file.py`（261）、`node_facts.py`（80）、`ui_view.py`（+126）、`ui_client.py`（+84）抵消。**本轮唯一真正"净减"的是主机测试（−178）。**

**先加后删的量。** 68 个提交逐个累加，源码共加 21,297 行、删 6,054 行，而 `7d5be70..HEAD` 的净差量是加 17,341、删 2,264。所以历史里约有 3,950 行源码、约 4,700 行测试是"加了又删"。用 `git blame` 看【重算 `blame.py`】，旧 40 个提交新增的源码在最终树里仍保留约 77%（12,481 行），本轮 28 个提交留下 4,963 行。

### 3.2 隐藏（非公开 SDK）接口清单（最终版）

三个版本的数量：HiddenApi 惰性成员 51 / 65 / 70；HiddenApi 之外直接 `Class.forName`、`HiddenApi.method/field` 或手写 `onTransact` 的文件 1 / 8 / 9；编译期平台桩 0 / 0 / 2（`android/platform-stubs/` 下两个 Java 文件，供 `WindowInputListener` 继承隐藏的 `WindowInfosListener`）。成员清单见 `43/hidden-*.txt`。

签名稳定性：我下载了 AOSP `android-14.0.0_r1`、`android-15.0.0_r1`、`android-16.0.0_r1` 和 `main` 四个标签下的对应文件（`43/aosp/`），逐条比对。另外复用了已下载的 IASC/IAICC 14–16 和 UiAutomation 14–16。

| 组 | 成员（最终版） | 用途与消费者 | 14 → main 签名 | 断裂风险评估 |
| --- | --- | --- | --- | --- |
| UiAutomation 自建连接 | `UiAutomationConnection` 构造器、`UiAutomation(Looper, IUiAutomationConnection)`、`connectWithTimeout`、`disconnect`、`getConnectionId`、`getCache`、`mRemoteCallbackThread` 字段 | `UiBackend` 连接与清理 | 构造器和 connect 未变；私有字段 `mRemoteCallbackThread` 是实现细节 | 低到中：私有字段改名会让清理诊断退化；基线就有同类依赖 |
| 节点与缓存 | `getSourceNodeId`、`getChildId`、`LEASHED_NODE_ID`（私有字段）、`setConnectionId`、`setSealed`、`AccessibilityInteractionClient.getConnection/getCache`、cache `setEnabled` | E9 定稿、关闭缓存 | 未变 | 低：多年 @hide 稳定 |
| **E9 手写回调** | IASC `findAccessibilityNodeInfoByAccessibilityId`（7 个参数）、`performAccessibilityAction`、`getWindows`、`getMagnificationConfig`；IAICC Stub 的 `TRANSACTION_setFind…`、`setPrefetch…`、`setPerform…` 常量；自写 `onTransact` 按布局读 Parcel | `CollectionNodes.kt:110-140`、`AccessibilityConnection.kt:154-171` | 14/15/16 方法与参数未变；IAICC 16 在末尾加了 `sendAttachOverlayResult`，不影响已用的事务 | 中：事务码靠反射读，能抗重排；外层 Parcel 布局是手写的，平台改 oneway 参数就会断。本轮已改成快速失败 |
| 坐标接收者 | `WindowInfosListener`（继承隐藏抽象类，编译期桩）、`DisplayInfo.mDisplayId/mLogicalSize/mTransform`、`InputWindowHandle.ownerUid/displayId/inputConfig/touchableRegion` | `WindowInputListener.kt`，只给默认图像手势用 | 14/15/main 的 `onWindowInfosChanged(InputWindowHandle[], DisplayInfo[])` 和 `register()` 未变 | 中：编译期继承隐藏类，依赖 R8 规则和隐藏 API 运行期放行；只在模拟器发行构建里验证过 |
| 输入注入 | `IInputManager.injectInputEventToTarget(InputEvent,int,int)`、`InputEvent.setDisplayId` | 触摸、按键、wake | 未变 | 低 |
| 启动与任务 | `IActivityTaskManager.startActivityAsUser`（12 个参数）、`getTasks(int,boolean,boolean,int)`、`IPackageManager.resolveIntent/queryIntentActivities(Intent,String,long,int)`、`AppGlobals.getPackageManager`、`ParceledListSlice.getList`、`ActivityManager.getCurrentUser`、`ActivityTaskManager.supportsMultiWindow` | `ActivityLaunch`、`ActivityTasks`、`WorkDisplay` 预检 | 未变 | 低到中：startActivityAsUser 参数多，历史上改过 |
| **设置项绕路** | `IActivityManager.getContentProviderExternal`、`removeContentProviderExternalAsUser`、`ContentProviderHolder.provider`、`IContentProvider.call(AttributionSource,…)` | **只给** `WorkDisplaySettings.kt:8-59` 读 `force_resizable_activities` 一个设置项，用来在后台屏预检里多给一种"支持"判断 | 未变 | 低风险，但 59 行代码只换来一个边角判断（§3.3） |
| 截屏 | `IWindowManager.captureDisplay`、`createSyncCaptureListener`、`getBuffer`、`getHardwareBuffer`、`asBitmap` | **只给后台屏截图** `WorkDisplay.kt:127-150` | 14–main 均为 `captureDisplay(int, ScreenCapture.CaptureArgs, ScreenCapture.ScreenCaptureListener)`；代码沿参数类型反射，不写死类名 | 中：ScreenCapture 类所在包历史上变过 |
| 锁屏 | `IWindowManager.isKeyguardLocked/isKeyguardSecure(int)`、`ITrustManager.isDeviceLocked(int,int)` | 设备事实 | **isDeviceLocked 的第二参数在 15 从 displayId 改为 deviceId**（同元数，语义变了），代码传 0 两种语义都成立 | 中：近两个版本签名语义变过 |
| 通知监听器 | `INotificationManager.registerListener/unregisterListener/getActiveNotificationsFromListener/cancelNotificationsFromListener`、`INotificationListener` 自写 `onTransact`、`NotificationListenerService.TRIM_FULL`、`UserHandle.getIdentifier`、`PendingIntent.sendAndReturnResult`（7 个参数） | 通知对象命令 | 未变 | 低到中：签名稳，但行为风险高——以 shell 身份登记**系统级**监听器，登记期间能收到所有用户的通知，并获得通知附件 URI 读授权【源码，docs `architecture.md:94` 自述】 |
| 剪贴板与 AppOps | `IClipboard.get/set/clearPrimaryClip`（5 个参数）、`IAppOpsService.noteOperation`（7 个参数）、`AppOpsManager.OP_READ_CLIPBOARD`、SystemUI 的 `SUPPRESS_CLIPBOARD_OVERLAY` extra | `ClipboardAccess.kt`；`noteOperation` 只用来证明"读到 null 等于剪贴板为空"（`ClipboardAccess.kt:120-169`） | 未变 | 低到中：SystemUI extra 是约定，不是接口 |
| **字体绕路** | `Typeface.getSystemFontMap/setSystemFontMap/DEFAULT_FAMILY`、`IFontManager.getFontConfig`、`SystemFonts.buildSystemFallback/buildSystemTypefaces` | **只给**图像标注画编号文字（`ImageMarks.kt:87-115`）；会**改写 worker 进程全局的字体表** | 未变 | 低风险，但为画几个编号改进程全局状态，性价比低 |
| 其它 | `ServiceManager.getService`、各 `$Stub.asInterface`、MagnificationConfig/TransformationSpec 字段 | 通用 | 未变 | 低 |

结论【上游，已核 4 个版本】：签名层面 14 到 main 基本稳定，"近两个版本才有或改过"的只有 `isDeviceLocked` 的语义。断裂风险主要在三处：

- 手写 Parcel 布局（E9 和通知回调）；
- 继承隐藏类加编译期桩（WindowInfos）；
- 厂商 ROM 或隐藏 API 策略【需设备，未核】。

本轮新增的 5 个 HiddenApi 成员里，4 个只服务 `WorkDisplaySettings` 这一个边角判断。

### 3.3 重资产逐项评估与去留

"源/测/指引"只算专属文件；分散在 `UiActions`、`UiBackend`、`ui_commands.py`、契约里的接线代码没有计入，所以是下限。使用证据来自：本轮 8 次真实任务 84 次调用【重算 `43/usage.txt`】；上一轮 12 条流程 199 次命令【评审方旧报告 `sub-reports/31-real-flows.md:209-211`，我核对了引用】。

| 资产 | 最终行数（源/测/指引） | 隐藏接口 | 真实任务使用 | 去留建议 |
| --- | --- | --- | --- | --- |
| E9 手写回调采集 | 835 / 766 / 0（基线 TreeCollector 261） | IASC、IAICC 事务码、手写 Parcel、setSealed 等约 10 个 | 每次采集都走它 | **重做为公开缓存窗口**：采集期间开平台缓存，用公开 `getChild(i, FLAG_PREFETCH_DESCENDANTS_DEPTH_FIRST)`，结束时关缓存。上一轮唯一的对照实验（单样本）里，展开后两条路线耗时持平，滚动后 E9 少用约 30% 时间；第三轮报告自己写明"按事先判据，结果落在'换'的一侧"，却仍保留了 E9【旧报告 R3 §2.3】。本轮 E9 又长了约 370 行。换掉之后可删手写 Binder、Parcel 解析和仿平台定稿，约减 500 行源码和对应测试。风险：公开路径单次查询阻塞上限固定 5 秒，滚动后约 150 毫秒内不预取。**需要一次实测**：同一长网页滚动后两路线各 5 次 |
| 后台屏会话 | 489 / 432 / 107，另有大量接线：通知 virtual Activity、启动冲突预检、`displayKind` 参数、`TargetManagement` 换种类 | `captureDisplay` 链 5 个、设置项绕路 4 个、`supportsMultiWindow`；需要 ADD_TRUSTED_DISPLAY 等 5 项权限 | 0 次（两批都是 0）；MuMu 根本建不了后台屏 | **删除**。已观察到副作用：同一个 Chrome task 从主屏被搬到后台屏，结束后没有回主屏【原件，交付稿 `route-rebuild-candidate.md:142`】。删掉之后第一道待决题整题消失，`session open --display` 变成无参数。损失：不打扰用户主屏的后台操作。这对"给 LLM 用的自动化 CLI"不是核心能力，用户真需要时再单独立项 |
| 通知对象读写与监听器 | 1,586 / 2,351 / 173 | 约 12 个，含系统级监听器 | 0 次 | **删除对象命令**（list、get、action、reply、dismiss、open），保留现有 `notifications`（展开通知栏）加普通 UI 操作。损失：不展开通知栏就能回复或点按钮。收益：去掉"shell 身份系统级监听器收到所有用户通知"这一隐私面，去掉选择串指纹格式（SHA-256 + UTF-16 长度前缀 JSON 数组），去掉第一道待决题的一半。这是本轮增长最多的子系统（+416 行源码、+711 行测试） |
| 输入法通道与 Chrome 桥 | 431 / 191 / 0（`EditorBridge`、`EditorSurrounding`、`CurrentEditorInput`、`BrowserAccessibilityService`、AIDL） | 无隐藏接口，但要求用户在系统设置里启用无障碍服务 | 本轮 0 次；上一轮 `editor select/commit` 各 4 次，都是实施方自己操作 | **`BrowserAccessibilityService` 保留**：没有启用的无障碍服务，Chrome 不提供网页无障碍树，读网页是核心能力。**`type --via editor` 与 `--via paste` 二选一**（见下一行） |
| 剪贴板借用粘贴 + 结构剪贴板 | 354 / 351 / 0；另有结构剪贴板 546 / 538 / 47 | IClipboard、AppOps、SystemUI extra | 0 次 | **粘贴保留一条**（处理 `fill` 写不进的中日韩文本和 Compose/网页输入框）。**结构剪贴板删去"空值证明"**（AppOps noteOperation 与锁屏事实），get/set 只保留朴素结果。三条文本通道（keys、paste、editor）加 AX `fill` 太多：建议保留 fill + keys + 其中一条非 ASCII 通道。证据不足以判定 editor 和 paste 哪个更好，交用户定 |
| 多指时间线 / pinch / drag | 579 / 469 / 0 | 无（共享注入通道） | 两批都是 0 | **删 timeline、pinch**，保留 tap、swipe、长按，drag 可以作为带停顿的 swipe。约减 400 行源码。损失：地图双指缩放（可以用 App 的缩放按钮或双击代替） |
| 保存 / 比较 / 历史配额 | 775 / 1,274 / 0（`ui_diff`、`view_file`、`history`） | 无 | `--diff-from` 两批都是 0；`--out` 偶有使用 | **删 `--diff-from` 与 View 文件解码器**（`view_file.py` 261 行、`ui_diff.py` 266 行、约 900 行测试）；保留 `--out` 和有界历史。LLM 自己比较两次输出就够了；比较功能还带来"格式、域、尾标、部分采集退全文"一整套规则 |
| batch | 317 / 1,599 / 86 | 无 | 本轮 0 次；上一轮 3 次（实施方操作），上一轮报告记"无稳定端到端加速" | **删除或冻结**。测试和源码之比 5:1，还带来"已执行前缀、每步非零即停"的语义。若用户坚持"减少往返"，更便宜的做法是动作命令接受一个 `--then-wait 条件`（第二道待决题的 B 选项），而不是一个静态脚本解释器 |
| 事件原件（UiRecentEvents、UiSignals） | 489 / 418 / 0 | 无 | 被动出现在输出里；wait 用事件唤醒 | 保留"事件唤醒 wait"（性能），**删"近期事件事实"对外字段**。证据弱，可选 |
| WindowInfos 坐标接收者 | 296 / 178 / 0 + 2 个桩 + R8 规则 | 继承隐藏类 | tap/swipe 本轮 0 次，上一轮少量 | 保护作用真实：防止坐标手势落进别的 App 的窗口。但这是隐藏程度最深的接线。**保留**，前提是删掉其它坐标高级功能后它的消费者只剩 tap/swipe |
| typed Intent 启动与落点确认 | 938 / 1,186 / 0 | IATM、IPM | `open` 本轮 0 次（由测试夹具开 App） | **保留"启动并确认落点"**；component、extra、category、mime、grant 这些 typed 字段砍到 `--url`/`--package`/`--component` 三种 |
| 图像标注与区域 | 383 / 197 / 0 | 字体绕路 | screenshot 1 次 | 区域保留；**标注改为不画文字或换一种不依赖系统字体的画法**，删掉字体绕路 |

**删除上述建议项的粗估**（下限，不含接线和契约）：源码约 4,000–4,500 行（约占最终主源码的 13%），测试约 5,500 行，指引约 450 行；隐藏接口约 25 个成员；`@Volatile` 和锁约减三分之一，因为后台屏、通知 Access、EditorBridge、WorkDisplay 的字段都是长寿命状态。第一、第二道待决题会因此消失或简化。【推断，按文件计数】

### 3.4 模块职责与上帝文件

- **`ui_commands.py`（2,070 行）**：基本上是一个 1,650 行的 `register()`（从第 415 行起），逐个命令声明 Typer 参数。每个动作命令都重复约 12 个观察和交付选项（screenshot、image-format、marks、image-out、all、boxes、depth、interactive、max-chars、print、diff-from、settle）。这不是逻辑复杂，而是**选项乘法**：877 个选项位置的根源。建议把观察选项收成一个 `--view` 预设，或只放在 snapshot/wait 上，动作命令只接受 `--screenshot`。
- **`ui_client.py`（1,520 行）**：混合了三件事：RPC 客户端（`UiClient`）、逐命令的回执一行文本（`result_line` 260 行，第 186–446 行）、交付与保存（`deliver`、`render_observation`、`save_delivery`）。拆成 client、receipt、delivery 三个模块是机械工作，收益中等。
- **`UiBackend.kt`（743 行）**：连接 UiAutomation、执行路由、采集、wake、阶段化清理都在一个类里，27 个成员。`UiActions.kt`（1,080 行、54 个函数）承担 fill、check、paste、range、touch 等所有节点动作，却只有 `UiActionsTest` 2 个测试；它们的行为主要靠其它单元测试和模拟器覆盖。
- **重复机制（设计上有意，但应知道代价）**：
  - 设备端选择器（`UiSelector`）和主机端 `find` 是两套文本匹配器，靠契约里的 `entityForms` 同步 XML 实体解码候选；
  - 节点默认值表在 Kotlin（`NodeEncoding`）和 Python（`node_facts.py`）各一份，靠契约的 `nodeEncoding` 对齐；
  - 动作名表两端各一份。
  每改一个字段，要动两种语言加 496 KB 契约。
- **只有一个消费者的机制**：`WorkDisplaySettings`（只服务一个预检判断）、字体绕路（只服务标注文字）、`ClipboardAccess.emptyBasis` 的 AppOps 证明（只服务"null 按空处理"）、`NotificationAccess.possibleRegistered` 加一次修复额度（只服务监听器）。
- **主机与设备的职责划分大体合理**：语义投影、身份、条件、核值放在设备端是对的，避免往返和陈旧。可议的有两处：
  - `find` 放在主机、选择器放在设备，造成双实现。建议 `find` 也走设备，复用 `UiSelector`，主机只渲染。或者反过来让 `locate` 和 `wait` 的匹配只认设备端，删除主机端的实体候选表。
  - 回执一行文本（`result_line`）在主机按命令逐个拼，设备端的 reason 又是另一套英文句子，两端都有"人话"。

### 3.5 docs 仓库与三份描述

- 体量：docs 仓库 292 KB，全中文；其中 `reference/ui.md` 90 KB、`testing/cases.md` 75 KB、`architecture.md` 34 KB。公开指引 173 KB，英文。代码注释极少：Kotlin 约 143 行、Python 约 45 行注释，对应 3.2 万行源码。
- 性质：docs 的段落大量是变更说明和否定式边界。`reference/*.md` 里"旧"出现 96 次、"不再"18 次，"不是／不等于／不能／不证明"176 次。例如 `reference/ui.md` 一段写"host 固定 diff-baseline……已删除；旧固定文件不读、不迁移、不续写"。这类内容属于提交说明，不该留在设计文档里。`architecture.md` 第 88–108、161–169 行是高密度实现细节（类名、线程、调用顺序），没读过源码的人几乎读不懂，读过源码的人又不需要它。
- 代价：已经产生了实际漂移（§2 M1）。docs 是独立私有仓库，65 个提交随产品提交一一配对，几乎每个产品提交都要配一个 docs 提交。
- 建议：
  - docs 收缩成一页架构说明（组件、状态寿命表、并发归属），约 5–8 KB；
  - 机制的"为什么"写进对应代码的 KDoc/docstring；
  - 变更史留在提交信息里；
  - `testing/cases.md` 如果是真机回归清单可以保留，但要改成清单格式；
  - 公开指引是唯一面向调用者的合同，docs 不再复述它。

### 3.6 评审方前几轮建议，现在看哪些错了或代价过高

- **"方向对、能力全部保留"**【R3 §1.1】：这是规模失控的直接前提。R3 自己预测落地后是约 67 个命令、约 70 个隐藏成员、状态"最多持平"，最终结果几乎一模一样（66 个命令、70 个成员，状态还增加了）。这说明 R3 已经看到了规模，却没有建议删能力。现在看，**应当推翻"全部保留"**。
- **"保留 E9"**【R3 §2.3】：违背了 R3 自己事先定的判据，用的是单样本的理由。代价是本轮 E9 又长了约 370 行。建议推翻。
- **通知监听器"每个会话登记一次"、选择串**【R3 §2.4 及 §7】：R3 把它当简化提出，落地变成 +416 行源码、+711 行测试和一个 SHA-256 指纹格式。方向本身不错，但对一个零使用的能力，在里面做精修本身就是浪费。
- **WindowInfos 接收者"唯一例外"的批准**：保护作用成立，但它是本轮唯一新增的"继承隐藏类"接线。在不删其它坐标高级功能时，它的性价比偏低。
- **`--diff-from PATH` 取代固定基线**：方向对（净删了预留和提交逻辑），但更好的做法是整个比较功能都不要。
- 上一轮把"动作后仍见旧页"列为最高频摩擦，这一点仍成立；本轮只改了指引。

---

## 4. 证据审计

| 实施方声称 | 核对 | 结论 |
| --- | --- | --- |
| 本轮是"删除、合并、简化" | §3.1 重算：Kotlin +2,052 行，状态和锁增加，HiddenApi +5，契约 +55 KB，指引 +405 行；只有命令少 4 个、主机测试少 178 行 | **不成立**，至少在规模上不成立。删了主机偏好、editor 组、zoom、固定基线、驻留帧，但设备端补回的更多 |
| 累计可维护性 review "Optional 0" | 原件 `ST/integration-maintenance-review.md` 明写 review 范围是"已选行为"的必要复杂度 | 成立但边界很窄：它不评估能力去留，不能当作规模合理的证据 |
| `task quality` 不能写成一次全入口退出 0 | 原件 `final-quality/task-quality.result.json` 退出码 201；pytest 1,936 passed、2 skipped；`recovery-android-quality.result.json` 退出码 0，518 tests | 成立，如实。原因是环境 PATH，不是产品 |
| 质量门运行的代码等于最终树 | 质量门在 `041eaa7` 加 3 个文稿 WIP 上跑；`git diff 041eaa7 4235af9` 只改 `README.md` 和 `SKILL.md` | 成立：代码与最终树相同 |
| Linux 27 个模块 1,428 passed | 原件 `linux-final/result.md`；源是 `f146f32`，与 `4235af9` 只差 README、SKILL、batch.md 三份文稿 | 成立；只是部分模块 |
| `task ci` 类型分支未跑 | 我补跑：`mypy --platform linux/darwin/win32` 和 native 都是"no issues found in 35 source files"；`ruff check` 通过；`ruff format --check` 94 个文件通过（版本与 `constraints-ci.txt` 一致：mypy 1.10.1、ruff 0.15.14）【实跑，`--cache-dir` 和 `--no-cache` 指向工作目录，SRC 状态干净】 | 现在**补齐了 Python 静态检查**；macOS 原生运行仍缺 |
| wire 契约两端对拍 | 源码：`WireContractTest.kt:22-27` 用生产投影器和生产者生成样本并与 fixture 比；Python `test_wire_contract.py` 用生产 `Result`、`UiClient.blob` 和渲染器消费同一 fixture。我实跑 Python 一侧：194 passed | 成立，是"生产者 → 定稿文件 ← 消费者"的真对拍；不覆盖平台派发与采样（契约 README 自述） |
| 效率：最终四臂 | 原件交付稿第 179–184 行：Maps 新版 773k 对旧版 382k 输入，GitHub 新版 875k 对旧版 2,341k；第 0 阶段方向相反（第 168–173 行）。`final-old-source-binding.json` 显示"old"是最终产品加旧阅读补丁 | 实施方"净因果未知"的表述如实。但**整轮没有对基线的对照**，这一点交付稿没有突出 |
| "未授权删除的能力继续保留" | 与 R3 一致，确实都保留了 | 成立；这正是规模的来源 |

**单次成功被说成普遍结论**：本专题范围内没有发现。交付稿对 UNKNOWN、退出码 139、exit2 等失败保留得相当完整。但证据体系本身的体量（每个提交配 review、配运行原件、配 docs 提交）与产品规模一样，超出了一个人维护的需要。

---

## 5. 测试质量

**抽样**：主机 19 个、Android 30 个，覆盖会话、采集、输入、通知、图像、结果、连接、文件、契约【`43/sample-py.txt`、`43/sample-kt.txt`，加 6 个文件的定向抽样】。

| 类别 | 主机（19） | Android（30） | 例子 |
| --- | ---: | ---: | --- |
| 验证可观察行为（真实 CLI、协调器、文件、编解码；或纯函数的输入与输出） | 13 | 16 | `test_ui_delivery.py:776`（预检拒绝时不连 RPC，文件不变）、`test_daemon_lifecycle.py:68`、`UiRefRepairTest`（录制的候选池）、`ShellProtocolTest`、`PairingsTest`、`UiSemanticsTest:276` |
| 复述实现（断言内部调用序列、私有状态或内部 API） | 3 | 11 | `BorrowedPasteTest`（`calls.contains("borrow")`）、`UiCleanupTest`（`cleanup.complete`）、`UiScrollUntilTest:304`（`needsScroll`、`inspect`）、`SessionsTest`（`prepareChange`、`localStatus`）、`KeySequenceTest`（内部 attempted 计数）、`test_activity_intents.py:187`（`prepared.invocation` 内部形状） |
| 纯常量、文案、表断言 | 3 | 3 | `test_wire_contract.py:528`、`test_result_output.py:1030`（退出码表）、`NodeActionsTest`（断言 `16908348`）、`DeviceWakeTest`（整句 reason 文本） |

估计比例：主机约 70% 行为、15% 复述、15% 常量；Android 约 55% 行为、35% 复述、10% 常量。文案耦合不重：Kotlin 测试里完整句子的 `assertEquals` 只有 44 处（共 3,205 个断言）；Python 里 `assert` 句子或 "in" 断言约 100 处（共 3,928 个）。

**保护还是阻力**：

- 主机测试多数从 CLI 或协调器入口驱动，对重构是保护。
- Android 测试多是"夹具加 JSON"，夹具直接构造内部类（`Fixture`、`PasteFixture`、`Sessions.openForTest`）。一旦重构内部切分（比如删掉后台屏或合并文本通道），要整片重写；本轮 `83c3a6a` 删了 1,131 行测试，`041eaa7` 删了 1,567 行，`826c180` 删了 588 行。这是阻力，但属于随代码删除一起删的那种，不是"改一行文案改十个测试"。
- 本轮测试差量与源码差量之比是 10,931 : 7,961 ≈ 1.37；旧 40 个提交是 20,530 : 19,390 ≈ 1.06。没有发现文案级连锁修改的迹象。
- **最大的问题是保护面错位**：最脆弱的平台层没有 JVM 测试，包括 `UiBackend`、`TreeCollector`、`CollectionNodes` 的 Binder 路径、`WorkDisplay`、`NotificationAccess`、`EditorBridge`、`WindowInputListener` 的真实回调、`TargetManagement`。这些只靠模拟器单样本运行。2.5 万行主机测试和 1.57 万行 Android 测试，多数在保护本来就不太会坏的纯逻辑。
- 主机测试有 368 个 `parametrize` 和 383 个 `monkeypatch.setattr`。`test_ui_batch.py` 1,599 行，是 `ui_batch.py` 317 行的 5 倍。

---

## 6. 对相关待决题的评价与需要用户裁决的新取舍

### 6.1 第四道待决题：最终提交历史形态

事实【重算 `43/churn.txt`、`43/blame.py`】：

- 68 个提交里，源码"加了又删"约 3,950 行，测试约 4,700 行。
- 旧 40 个提交的源码在最终树里保留约 77%。
- 15 个提交单个新增源码超过 500 行，每个再配相近量的测试和 docs，单提交审阅单位已经很大。
- 每个中间提交能否独立构建、独立测试通过，实施方的材料没有逐提交证据，测试都绑在冻结点上【未能核实】。

| 选项 | 评价 |
| --- | --- |
| A 保留 40+28 | 历史里有约 4 千行源码先加后删，还有被推翻的机制（显示偏好、editor 组、固定基线、zoom）。供以后考古有用，合入主分支不合适 |
| B 只重切本轮 | 改善有限，旧 40 个提交里被替换的机制仍在 |
| C 从基线按最终行为全面重切（实施方推荐） | 范围没定之前做是浪费；如果按 §3.3 删能力，重切出来的提交大半要再改。中间提交没有测试结果，宣称的"可按行为 bisect"不成立 |
| D 各 squash 成一个 | 末树容易核对，但 4.9 万行的单个提交没法审、没法回退 |
| **E（建议，原方案漏掉）** | **先由用户定合入范围**（§6.2 第 1 项）；然后从最终树按子系统生成 6–10 个提交：①平台底座（Android 14+、管理窗安全、worker 生命周期、传输分块）；②结果与退出码；③会话与清理；④采集与节点、紧凑阅读；⑤输入；⑥图像；⑦指引与帮助；⑧契约。**每个提交在合入前跑一次全量质量门**，因为子系统少，这个成本可以承受。旧的 68 个提交留在一个归档分支上，供追溯 review 记录 |

### 6.2 需要用户裁决的新取舍

1. **是否推翻"能力全部保留"，按 §3.3 删除零使用的重资产。**
   - 候选 A：维持现状，整体合入。代价：接下来维护约 3.2 万行源码、70 个以上隐藏成员、496 KB 契约、173 KB 指引；第一、二道待决题还要继续投入。
   - 候选 B（推荐）：删后台屏、通知对象命令、`--diff-from`、batch、timeline/pinch、结构剪贴板空值证明、字体绕路、设置项绕路；文本通道减为三条。
   - 候选 C：只删后台屏和通知（风险最高、使用最少的两块），其余保留。
   - 我推荐 B，退而求其次是 C。剩余未知：用户本人是否有后台屏或通知的真实需求（这只有用户知道）。
2. **E9 是否换成公开缓存窗口。** 候选：保留 E9；换公开缓存窗口（推荐，先做 5 对 5 的同页滚动和展开实测，耗时差在 20% 以内就换）。剩余未知：大页面和真机上的分布。
3. **合入策略。**
   - 整体合入：风险最高，把所有零使用机制一起带进主分支。
   - 分批合入：先合平台底座和结果语义，再合阅读和输入；风险中等，但每批都要重新审接缝。
   - 挑选合入（推荐）：从最终树挑 §6.3 的"必须拿"和"值得拿"，按子系统提交。

### 6.3 如果从基线重新出发：68 个提交里拿什么

目标是"给 LLM 用、能力够用、调用便宜、可维护"。按提交主题分层：

**必须拿**（修掉基线真实缺陷，或带来核心能力）：

- `04b87b4`：要求 Android 14、删旧平台路径。这是简化。
- `d71a282`：管理窗安全。
- `6d18ebc`：push writer 先关闭再发布，真实缺陷。
- `ccfdaed`：配置父路径不是目录时的处理。
- `e6bab80`：系统服务死亡时 worker 退出，避免"活着但不可用"的死路。
- `865d816`：UI 分块与回复上限。
- `5b78002`：全局选项可以放在命令边界，减少 LLM 用法错误。
- `5d86ef1` 加 `d0c52c3`：行和 JSON 观察、紧凑正文、`find` 搜全部已采节点。这是调用成本的核心，但对基线的收益未实测。
- `600e29e`：动作事实与交付失败分开、退出码。
- `0027b52`：动作排序后再签名。
- `2cf6a46`：共享主题帮助。
- `9fcfb04`：结果里报告窗口切换。
- `13aeb4b` 的选择器部分：wait 按选择器匹配。
- `af36a32` 的 reveal 部分：reveal 本轮用了 4 次。
- `674b4e9`：坐标绑定到实际交付的预览图。修的是基线截图缩放与坐标不一致。
- `01077a8`：零派发时保留图像资格。
- `08cac85`：最小采样间隔。
- `eb78116`：用事件唤醒 wait。
- `11ca997` 的"启动并确认落点"部分。

**值得拿**：

- `0e583db` 的事件唤醒部分；
- `702da4a`：range；
- `9ad9fdb`：type --via keys 和按键组合；
- `8d1b45b` 或 `d5ae012` 二选一，做非 ASCII 输入；
- `12f6078`：浏览器无障碍服务，网页阅读依赖它；
- `cace585`/`3ef0ec6`：新采只读区域；
- `5b3adb7` 加 `2beaadd`：受约束的定向注入和坐标接收者；
- `6fc57c3`：清理分阶段、缩窄 pending 门；
- `d9ad562`：wake 走输入注入；
- `83c3a6a` 的"显式 lease"部分（不含 virtual）；
- `b2b1d1d`：截止归因（若保留 E9）。

**可以不拿**：

- `ec7eb6d`：batch；
- `b667727` 加 `041eaa7` 的比较部分：diff；
- `24399ff`：timeline、drag、pinch；
- `80f1308`：图像标注；
- `f397bb1`：结构剪贴板；
- `ee841ea`：可选节点元数据；
- `06090cf`：scroll-until，可以由 scroll 加 find 组合；
- `e95c07e` 的多窗口选项（保留默认窗口加 `--all-windows` 即可）；
- `bba1416`：控制者通知；
- `ab7ff63`：E9，若换成公开缓存。

**不应拿**：

- `9321313` 加 `83c3a6a` 的 virtual 部分：后台屏；
- `845f09a`、`9b67f68`、`483d755`、`857c26e`、`f68c477`、`9f80d5f`、`c03b08f`：通知对象读写与监听器。

---

## 7. 没能核实的内容

- 每个中间提交是否能独立构建、独立通过测试（没有逐提交证据；我也不能跑 Gradle）。
- 最终树的 APK/R8 产物是否正确：WindowInfos 桩是否被排除在 DEX 之外，`onWindowInfosChanged` 是否被保留【需设备、需构建】。
- 隐藏 API 在真机和厂商 ROM 上的策略与实现差异：我只核了 AOSP 源码签名，没有核运行期放行策略。
- §3.3 中"E9 换公开缓存"的性能结论：只有上一轮单样本数据，需要实测。
- 删除建议的行数是按专属文件计的下限；接线代码与契约的减少量没有逐行计算。
- 全量主机测试在 Linux 上的结果由主评审汇总，我只跑了 `test_wire_contract.py`（194 passed）和静态检查。
- docs 仓库的 bundle 缺前置提交，无法在本地克隆，只能读包内展开的最终内容和差量统计表；docs 在基线时的字节数没有核到。
