# 第三轮草案 F1–F30 独立事实复核

复核人：独立只读复核者。范围：`DRAFT.md`"二、准备引用的事实"F1–F30，外加"一、立场"里与事实直接冲突的地方，以及评审方上一版定稿的三条说法。

路径缩写：`SRC`=候选只读克隆（HEAD=3f45af6）；`P`=`ROOT/ultragoal/2026-11-11-next-improve/prototypes/route-review`；`E`=`P/exploration-20261119`。我自己写的脚本和下载的上游文件都在 `scratchpad/agents6/34/`（下称 `W`），上游文件在 `W/dl/`。编号 26–33 的报告正文我没有读。agents6/26/28/29 与 agents5 目录里已下载的 AOSP 文件，我抽了一份重新下载核对哈希（A15 `InputDispatcher.cpp` 逐字节相同），其余直接使用。

证据标签：【原件】指我直接读过的回执、日志或树；【源码】给出文件、行号和版本；【重算】给出脚本位置；【推断】。所有运行结论只来自模拟器：MuMu（Android 15/API35）或 Android Studio API34 镜像（`sdk_phone64_x86_64`，emulator-5584）。

---

## 一、逐条核对

### F1 交付失败只改写四个命令；batch 另行停批
**判定：属实，行号要微调。**
- 【源码 3f45af6】`ui_client.py:1387-1404` `finish_primary_result`：只有 snapshot/find（树没有交付）和 screenshot/zoom（图没有交付）会被改成 failed。其它命令即使带 `--out`、`--image-out` 且保存失败，`result` 仍是 done，`exit_code()` 仍为 0（`protocol.py:53-62`）。单条命令的 stdout 写失败也只记 `deliveryError`，不改退出码（`output.py:107-121`）。
- batch 的停批逻辑实际在 `ui_batch.py:360-376`（`delivery_error` 定义在 `:198-207`），不在 356-372。`snapshotError`、`windowsError`、`imageError` 任一存在，或 stdout 写失败，batch 都会停下并退出 1。
- 补充：batch 也会因 `windowsError` 停批，单条命令不会。草案只写了"附件"，这一项可以并入。

### F2 `finally` 里的 close 异常会吞掉结果
**判定：属实，但"报成未发出"有条件。**
- 【源码】`ui_client.py:1001-1007`：`run_request` 在 `finally` 里调用 `await client.close()`。`close` 经 `local_call.py:301-303` 进入 `transport.py:514-526` 和 `:54-71`，后者对非超时异常 abort 之后会 `raise`。所以 close 抛错时，已经拿到的 `(result, observation, image)` 会被异常顶替。
- 草案立场 P16(b) 说这种情况会"报成未发出"。只有异常是 `Fault` 时才成立：`cli_options.py:137-142` 把 `Fault` 映射为 `rejected`，即 not sent，退出码 2。其它异常（例如 `OSError`）不会被这一层捕获，会以未处理异常退出。哪些 close 路径会抛 `Fault`，我没有逐一追查。

### F3 `windowAcceptsTouchAt` 三版相同；`Transform::transform(Region)` 的语义
**判定：属实。**
- 【源码】`InputDispatcher.cpp` 中该函数在 android-14.0.0_r1 位于 `:516`，15.0.0_r1 位于 `:576`，16.0.0_r1 位于 `:582`。三版函数体逐行相同：先把 `touchableRegion` 和点都经 `displayTransform` 变换，再对点坐标取 `std::floor` 后做 `contains`。唯一差别是参数类型：14 是 `int32_t displayId`，15/16 是 `ui::LogicalDisplayId`。调用方 14/15 传 `getTransformLocked(displayId)`，16 有一处改为局部变量 `displayTransform`（`:1507`）。
- 【源码】`Transform.cpp` 中 `Transform::transform(const Region&)`：14 与 15 位于 `:301-319`，16 位于 `:307-325`，实现相同。类型高于 TRANSLATE 时，`preserveRects()` 为真就逐矩形变换，否则只变换外接矩形 `reg.bounds()`。草案没写的一个细节：纯平移时会把平移量四舍五入成整数（`floorf(tx()+0.5f)`）。

### F4 交给输入系统的显示变换恒保矩形
**判定：属实，14、15、16 三个标签各自确认过。**
- 【源码，自行下载】`ProjectionSpace.h` 在 14/15/16 三版的 md5 都是 `7e26bcbb…`，`getTransform()` 的组合是"旋转（`toRotationFlags`，只取 90° 的倍数）× 目标平移 × 轴向缩放 `scale.set(sx,0,0,sy)` × 源平移"。`Output.cpp` 中 `outputState.transform = layerStackSpace.getTransform(displaySpace)`，三版分别位于 14 `:205`、15 `:209`、16 `:215`。`DisplayDevice.cpp` 的 `getInputInfo` 计算 `displayTransform = undoPhysicalOrientation * getTransform()`，三版分别位于 14 `:155`、15 `:152`、16 `:163`；`undoPhysicalOrientation` 同样是 90° 倍数的旋转。交给输入系统的值是 `info.transform = displayTransform.inverse()`，保矩形变换的逆仍然保矩形。
- 限定：这只覆盖 SurfaceFlinger 的 DisplayDevice 路径。厂商改写的 SurfaceFlinger 不在核对范围内【推断】。

### F5 `getSurroundingText` 默认 offset=-1；Compose 没有覆写；候选要求 offset≥0
**判定：基本属实，"IC commit 的绝对锚点门"需要改措辞。**
- 【源码】AOSP 14 `InputConnection.java:379-400` 的默认实现返回 `new SurroundingText(..., -1)`。`BaseInputConnection` 有覆写（`:638-648`），但 Compose 的连接不继承它。
- 【源码，自行下载 androidx】在 `androidx-main` 提交 `2cda5a0d3a6a…`（2026-11-18）和分支 `androidx-compose-release` 提交 `ac6d2ce3d991…` 上分别核对：
  - 旧版 `compose/ui/.../text/input/RecordingInputConnection.android.kt`（`: InputConnection`）、foundation 下同名的 `RecordingInputConnection.android.kt`，以及新版 `StatelessInputConnection.android.kt`（`: InputConnection`），全文都没有 `getSurroundingText`。
  - 外层包装 `NullableInputConnectionWrapperApi21/24/25/34` 也没有覆写，因此都会落到接口默认实现，返回 -1。
  - agents6/28 已下载的两份 main 文件与我下载的逐字节相同。
  - 更早的 Compose 发布版本没有逐一核对。
- 【源码 3f45af6】offset≥0 的要求写在 `EditorSurrounding.kt:11-12`（`interpretable`）和 `:18`（`contains`）。两个命令在不同位置用到它，性质不同：
  - **select** 是发送前的门：`CurrentEditorInput.kt:99-100` 在不可解释时直接返回 NOT_SENT。
  - **commit** 不是门：命令照常发出，只在回读核验时，`EditorSurrounding.kt:73-74` 返回 UNKNOWN（"Surrounding text has no interpretable absolute selection"）。
  - 建议改写为："IC select 在 offset<0 时不发出；IC commit 照发，但核验只能得到 UNKNOWN。"
- 【原件】`P/r2-text/runtime-api34/commands/bitwarden-range-batch` 与 `bitwarden-caret-batch`，共 2 个批次。AX 选区都是 DONE。IC commit 前后 `offset=-1`，写入后文本正确（range：`R2T-ABQ🦊CD-Z`；caret：`R2T-AB😀Q🦊中CD-Z`），结果为 UNKNOWN，退出码 3。
- 样本与版本：API34 模拟器，1 个 App，2 个批次。APK 是 "e7d private-select 原型"，基于 3f45af6 打补丁，补丁不涉及 editor 代码（`P/r2-text/source-binding.json`）。

### F6 `editor select` 来自 d5ae012，不在发布标签里
**判定：属实。**
- 【源码】`d5ae012 feat(input): edit and select through the current editor` 新增 `ui_commands.py` 中的 `editor.command("select")`。
- 本地标签只有 v0.1.0、v0.1.1、v0.3.0，都不包含 d5ae012。基线 7d5be70 写的是 "prepare version 0.4.0"，但没有打 0.4.0 标签。PyPI 上的发布历史我没有核对。

### F7 regex101 的 AX→IC 位置映射
**判定：属实，需注明构建版本。**
- 【原件】`E/api34-ax-ic/commands`：
  - `031`：节点 `e878585760` 的 AX 文本是 `GET /health 200↵\nGET /missing 404↵\nPOST /upload 503`。我重算长度为 51 个 UTF-16 单位，原生文本为 49。`select … 48 51` 的结果是 DONE matches。
  - `032`：`editor commit 502` 的 before 是 `offset 0, length 49, selection 46..49`；after 文本末尾变成 `502`，caret 在 49，DONE。
- 样本与版本：API34 上的 Chrome Beta，1 次。APK 是研究 Agent "after-observe"（README 写明不是冻结产品 c669 的行为结论）。

### F8 网页上 AX 选区"立即读回"与"挪后读回"
**判定：属实，补充版本和站点构成。**
- 【原件】立即读回 3 次全部是 differs，都在 Wikipedia（Chrome，API34），构建为 e7d 原型：`P/r2-text/runtime-api34/commands/wikipedia-range-batch`（6..9）、`wikipedia-caret-batch`（8..8）、`wikipedia-city-repair-batch`（8..9）。我在整个 `ROOT` 里搜 "AX selection differs"，只有这 3 处。
- 挪到动作后观察之后读回，3 次全部 DONE：`P/r2-text/after-observe/runtime-api34/commands/wikipedia-range-after-observe-once`、`city-ax-commit-batch` 第 1 步（after-observe 构建），以及 `E/api34-ax-ic/031`（regex101，研究 Agent 1fced）。
- "前后对照"不在同一个 APK 上。失败的 3 次里没有 regex101。

### F9 E9 与公开缓存对比（MuMu API35，Wikipedia 4226 节点页，各 1 次）
**判定：所有数字属实。**
- 【重算 `W/e9.py`，读取 `P/r2-nodes/runtime-mumu/results.json` 和 `commands/*-log/stdout.bin` 中的 AndroidCtlCollector 行】

| 场景 | collect (ms) | 事务 started=completed | CLI (s) | 采集窗口 (ms) | 结果 |
| --- | --- | --- | --- | --- | --- |
| 滚动后 E9 | 705 | 361 | 2.938 | 1390 | 4226 节点，partial=false，missing=[]，unsettled |
| 滚动后公开缓存 | 990 | 1437 | 3.109 | 1406 | 同上 |
| 展开后 E9 | 903 | 883 | 2.860 | 1386 | 同上 |
| 展开后公开缓存 | 884 | 980 | 2.891 | 1390 | 同上 |

- 1437/361 = 3.98 倍。"约 1.39 s"成立，四次的范围是 1.386–1.406 s。
- 草案没写的事实：
  - 四次采样都**超出了采集截止**（`finishedAfterCollectionDeadlineMs` 分别为 E9 75/184，缓存 323/191 ms）。
  - 四次在同一个 worker（`97172424…`）上依次运行。
  - APK 是研究构建（`1881a0f7…`，候选加研究补丁），不是候选原样。

### F10 E9 的出处与手写部分
**判定：基本属实，"两个 AIDL 在 14–17 未变"要改措辞。**
- 【源码】`ab7ff63 feat(observation): collect larger trees with scoped prefetch` 新增了 `CollectionNodes.kt`。
  - 解析用 `AccessibilityNodeInfo.CREATOR`（`CollectionNodes.kt:128-129`）。
  - 手写部分：`onTransact` 外层布局（`:115-130`，包括 `enforceInterface`、尾部 `readInt` 和 `dataAvail()==0`）；7 参数的 `findAccessibilityNodeInfoByAccessibilityId`（`HiddenApi.kt:117-128`）；两个事务常量名 `TRANSACTION_setFind…`、`TRANSACTION_setPrefetch…`（`:111-116`），它们是反射读出的值，不是硬编码。
  - 基线 `AccessibilityConnection.kt:131` 已有 `ActionCallback`。
- 基线的读取方式：API 33 及以上用公开 `getChild`；33 以下走 `AccessibilityInteractionClient` 的隐藏方法（基线 `NodeAccess.kt:12`、`HiddenApi.kt:162-171`）。基线关闭缓存同样用隐藏接口（基线 `AccessibilityConnection.kt:33-37`）。候选对应的行号 `AccessibilityConnection.kt:41`、`HiddenApi.kt:83,90` 无误。
- 【源码，自行下载 14/15/16/17 r1】两个 AIDL **文件本身有变化**：
  - 15 起加了 `@RequiresNoPermission` 注解。
  - 回调接口在末尾追加了 `sendAttachOverlayResult`。
  - 服务接口在 15 有改动，17 把 `ScreenCapture` 改成了 `ScreenCaptureInternal`。
  - E9 用到的三个方法签名和它们在接口中的位置（也就是事务号）四版一致：服务接口第 2 个方法；回调接口第 0 和第 2 个方法（`W/aidl.py`）。
  - 准确说法："E9 依赖的方法签名与顺序在 14–17 未变。"

### F11 公开查询的等待时长与滚动后不预取
**判定：基本属实，"固定等待"要改。**
- 【源码，自行下载】`AccessibilityInteractionClient.java`：`TIMEOUT_INTERACTION_MILLIS = 5000` 在 A15 `:114`（A14 `:112`，A16 `:114`）。`waitForResultTimedLocked` 是"**最多**等 5 秒"，结果到达就返回，调用方不能缩短这个上限。所以应写"单次公开查询的阻塞上限固定为 5 秒，不受本地预算约束"，不能写"固定等 5 秒"。
- 滚动后不预取：A15 `:116-117` 中 `DISABLE_PREFETCHING_FOR_SCROLLING_MILLIS = getSendRecurringAccessibilityEventsInterval()*1.5`，默认值为 100×1.5 = 150 ms（`ViewConfiguration.java:262`）。条件是该客户端在 `onAccessibilityEvent` 中收到了该窗口的 `TYPE_VIEW_SCROLLED`（`:1155-1156`），在 `:648-651` 清掉预取标志。

### F12 本地截止被计成"节点不可用"
**判定：属实。**
- 【源码】`CollectionNodes.kt:38`（截止后直接返回 null）、`:64-67`（`:66` 无回调返回 null，`:67` 截止后返回 null）、`TreeCollector.kt:318-321`（null 计入 `unavailable++` 和 `markMissing`），随后写入 `:334` `unavailableChildren`。

### F13 稀疏表示实测（MuMu，California 页）
**判定：数字全部属实；草案漏了一个混杂因素。**
- 【原件，`P/r2-sparse/runtime-mumu`】基线 `raw/baseline-california-expanded.tree.json` 为 8,382,705 B、5485 节点、partial；稀疏 `raw/sparse-california-expanded.tree.json` 为 6,185,243 B、8191 节点、partial。
- CLI `elapsedSeconds` 为 5.234 和 6.579。稀疏臂 `trace.json:269` 记录 `expandedBytes 12478962`、`expansionSeconds 0.891`。
- 序号：基线臂 sequence=11；稀疏臂 sequence=1，并有 notice "New worker"。
- missing 的原因：基线臂有 `treeByteLimit` 和 `collectionLimit` 各 1 条；稀疏臂 2003 条中没有任何顶层原因码。
- 【源码】节点上限截停发生在 `TreeCollector.kt:315` 的循环条件，不写原因码。`:138-139` 的 `collectionLimit` 只在队列非空时写出，与时间截止共用同一个码。
- 漏掉的混杂因素：稀疏臂多采了 2706 个节点（+49%），设备侧 snapshot RPC 3.984 s，基线臂 3.406 s。另外稀疏臂回执有 `windowsError: Request budget exhausted before artifact delivery`。立场 P14 把变慢只归因于"展开加冷 worker"，漏了节点数这一项。

### F14 身份派生中可见性进入三处
**判定：属实，有一处补充。**
- 【源码】`UiSemantics.kt:159-160`、`:238-243`、`:287-291` 三处在 3f45af6 上行号无误。基线 7d5be70 对应 `:154`、`:239-243`、`:287-291`，逻辑相同。
- 另有 `:132` 一处：`root.optBoolean("visible")` 决定是否把后代记为已吸收（consumed）。它影响吸收，不直接影响身份四元组。身份原型保留了这一处。

### F15 MDN reveal 原件与离线扫描
**判定：MDN 两对属实；离线扫描数字取决于口径，需要改写。**
- 【重算 `W/mdn.py`】`E/mumu/raw` 中 079→081 和 085→087 两对，按 `(windowId, sourceId)` 匹配目标节点。目标自身只有 `visible`（false→true）、`bounds` 和派生的 `row.label`（null→"Examples"）变化，派生的 `ref`、`parent` 随之改变。回执为 reveal `unobservable`，错误码 `selectionChanged`。
- 【重算】`E/mumu/raw` 共 115 棵树，114 个相邻对，可以确认。我用两种口径重数：
  - 宽口径（`W/visid.py`：节点自身原始字段除 visible/bounds 外相同，而身份四元组变了）：**13 对、137 个节点、44 个可点**。
  - 加上"后代内容和祖先原始字段（不含 actions）也不变"（`W/visid3.py`）：**11 对、129 个节点、43 个可点**。
  - 草案的"12 对 / 130 / 44"落在这两种口径之间。其中 44 是宽口径 137 个节点里的可点数，不是 130 个里的。建议写成："约 130 个（按口径 129–137），11–13 对，其中约 43–44 个可点。"
- 【重算 `W/allnodes.py`、`W/unav.py`】163,366 是 `P` 下全部 271 份 `*.tree.json` 的节点总数，不是"全部保存节点"（把附件解码件也算上是 247,484）。这 163,366 个节点里，**`unavailable` 列表没有一个非空**；另有 42,056 个节点（8 份稀疏编码文件）省略了这个键。所以"收紧出现 0 次"只是因为语料里根本没有字段不可用的节点，不能说明这种风险不存在。

### F16 身份窄原型与实测
**判定：属实。**
- 【原件】`E/identity-prototype/source.patch` 只改了 `UiSemantics.kt` 的五个条件，另外改了测试。
- `E/identity-runtime/mumu/commands`：
  - `035-prototype-reveal-once`（Maps）：reveal observed，effect changed。
  - `050-mdn-reveal-examples-once`：observed。
  - `052-mdn-consume-retained-original-ref`：用原 REF 点击，DONE。
  - 对照组 `014-baseline-reveal-once`（c669）：unobservable。
- 都在 MuMu 上，各 1 次。

### F17 通知监听器的全局引用累积
**判定：属实，需注明证据形态。**
- 【重算，自写 HPROF 解析器 `W/hp.py`，读 `ROOT/package-extras/notification-heap-excerpts/{reg2,reg6,exited}/sparse-excerpt.hprof`】
  - 探针组件 `com.android.shell/…ListenerLifetimeProbe` 的 `ManagedServiceInfo` 实例：reg2 有 2 个，各有 1 个 JNI 全局根；reg6 有 6 个，各 1 个；exited 有 7 个，全局根都是 0。
  - 所有实例的 `service` 都是不同的 `Stub$Proxy`，但 `mRemote` 指向同一个 `BinderProxy 0x140e5bc0`。
  - 注意：读到的是实施方导出的**摘录**，原始 hprof 不在包内。实施方的 `inspect_hprof.py` 输出（`P/r2-notifications/runtime-api34-recovered/commands/*-analysis`）与我的解析一致。exited 有 7 个而不是 6 个，说明退出前又注册过一次。
- 【源码，自行下载 art】`java_vm_ext.cc` 的 `kGlobalsMax = 51200`：14 位于 `:63`，15/16 位于 `:62`。`AddGlobalRef` 失败后 `LOG(FATAL)`：14 `:702`，15 `:709`，16 `:727`。ART 是 mainline 模块，设备上实际运行的版本可能与标签不同【推断】。
- 【源码】`ManagedServices.java` 在 14/15/16 中都没有 `unlinkToDeath`。14 在 `:1574`（bind）和 `:1709`（`registerServiceImpl`）调用 `linkToDeath`；agents5 那份 A14 与我重新下载的逐字节相同。摘录中 system_server 已有约 1754 个全局根，所以离 51,200 的余量约 4.9 万。
- 【原件】`P/r2-notifications/runtime-mumu-lifetime/native/heap-a-dump-once`：MuMu（127.0.0.1:16416）上 `am dumpheap` 返回 `SecurityException: Process not debuggable: android`，退出码 255。

### F18 所有租约结束路径都经过 `invalidateLease`
**判定：基本属实，有一个例外。**
- 【源码】`Sessions.kt` 里 `current` 只有三处赋值：`:94`、`:131`、`:282`。进程内的结束路径（接管/替换 `:120`、`releaseLocal :176`、`revoke :185`、`expire :243`）都经过 `invalidateLease :281-284` 再到 `clearLeaseUi`。
- 例外：**worker 正常 Stop** 走 `WorkerService.prepareStop :115-123` → `sessions.stop() :232-235`（只做 `requireIdle` 并设置 `stopped`），再由 `destroy()` 调 `exitProcess`。这条路径不调用 `invalidateLease`，租约随进程一起消失。

### F19 通知 PendingIntent 的隐藏接口与 Termux 样本
**判定：属实，时间点措辞要准确。**
- 【源码，14/15/16 自行下载】`PendingIntent.getIntent()` 标了 `@hide @UnsupportedAppUsage`，权限由服务端检查（A15 `ActivityManagerService.java:5790-5792`）。`queryIntentComponents(int)` 标了 `@hide @SystemApi(MODULE_LIBRARIES) @RequiresPermission(GET_INTENT_SENDER_INTENT)`。Shell 的 `AndroidManifest.xml` 在 14/15/16 都声明了这个权限（A14 `:359`）。
- 【原件】`P/r2-notifications/runtime-vd-api34/native/c-positive-send-once`，API34 模拟器：
  - 通知身份：creatorPackage/Uid 为 `com.termux/10145`，目标 display 3。
  - `dispatchStartedMs→callReturnedMs` 用了 15 ms。callReturned 之后 16 ms 内读到的任务 201 **已在 display 3 上存在，但 visible=false**，所以措辞应是"任务已存在、不可见"，不是"任务不可见"。
  - `c-after-send-activities` 在探针进程启动后约 0.6 s（发送后约 0.3 s）开始 dumpsys，此时 Task #201 visible=true，TermuxActivity 为 topResumed。
  - `commands/c-positive-vd-snapshot` 在发送约 51 s 后才执行。
- 【原件】`native/e-conflict-zero-send`：preflight 判定 conflict（display 0 上已有 TermuxActivity 任务 202），`sendCallCount=0`。
- 【源码】候选的通知写操作只在发送前后各读一次任务事实（`NotificationActions.kt:335-360`），没有调用 `postAction.settle`。

### F20 VLC 样本
**判定：属实。**
- 【原件】`P/r2-sessions/runtime-attempt-2/commands`：
  - `main-a-paused-media`：VLC 会话 PAUSED(2)。
  - `library-media-tap-once`：在 display 2 上 tap，`--expect org.videolan.vlc`，DONE。
  - `after-click-media` 与 `after-late-media`：PLAYING(3)。
  - `before-click-activities` 与 `after-click-activities`：Task #198 在 display 0，#199 在 display 2，前后都没有变化。
- 两个屏上是同一个包，媒体会话按 App 共享。API34，1 次。

### F21 UiAutomation 断开与单一注册位
**判定：属实，有一处补充。**
- 【源码】`UiAutomation.disconnect()` 在 14/15/16 中相同：状态已是 DISCONNECTED 且 `useAccessibility()` 时直接返回。另外，**第一次调用在发出远程调用之前就已经把状态置为 DISCONNECTED**，所以即使第一次的远程调用失败，同一个对象之后再调也只会在本地返回。
- 【源码】`UiAutomationManager`：14 `:102-105`，15 `:108-111`，16 `:108-110`，已有注册时抛出 "already registered!"。
- 【源码】候选 `UiBackend.kt:503-521`：`unregister(ui)` 抛错时会跳过 `:518` 的 `automation = null`，`automation` 不被清空。

### F22 清理失败后的连带阻塞
**判定：属实。**
- 【源码】`Sessions.kt:188-202`（`reserve` → `requireResources`，任何 UI 资源都会拒绝 shell/file，调用方见 `ShellJobs.kt:34`、`FileJobs.kt:25`）；`:241-244`（有资源时 `expire` 不执行）；`:232-235`（`stop` 调 `requireIdle`）；`WorkerService.kt:115-123`。
- 补充：`prepareChange :115` 的 `requireIdle` 也会挡住新的 session open。

### F23 wake 的注入通道
**判定：属实，有补充。**
- 【源码】wake 走 `SystemActions.kt:101-117` 和 `policyKey :163-199`，使用 `UiAutomation.injectInputEvent`。`AppInput.kt:64-71` 的 `sender` 与 `:89-117` 的 `InputSender` 用的是 `IInputManager.injectInputEventToTarget`（`HiddenApi.kt:149`），`key`、`type`、`back` 都走这条（`SystemActions.kt:49-51,130`）。
- 补充：**home 和 recents 也走 `policyKey`，即 UiAutomation 通道**（`:52-53`）。wake 的轮询还用到 `ui.platform(run).signals`（`:108`）。
- 【源码，A14 自行下载】`adb shell input keyevent` 的路径是 `InputShellCommand` → `InputManagerGlobal.injectInputEvent` → `injectInputEventToTarget(event, mode, INVALID_UID)`。

### F24 system_server 重启后 worker 仍存活
**判定：属实。**
- 【原件】`E/api34/commands/035-final-native-identity`（emulator-5584）：
  - system_server 从 3668 变为 4058，zygote64 从 3608 变为 4000；stderr 显示 `/proc/3668`、`/proc/3608` 已不存在。
  - `shizuku_server 2148` 和 `com.rainng.androidctl:executor 2268` 仍在，父进程都是 PID 1。
- 【源码】`UiBackend.kt:440-444`：在 `:442` 判断 `platform != null` 就直接复用。`WorkerService.kt:149`：未停止就报 ready。worker 实际报出 ready 这一点没有原件，是源码推断。

### F25 文件发布的 ETXTBSY
**判定：属实，vboxsf 的说法需改为"statfs 魔数"。**
- 【原件】`E/mumu/probe-output.txt`：writer 未关时 `renameat2(RENAME_NOREPLACE) rc=-1 errno=26 Text file busy`；关闭后 rc=0。
- `commands/040-filesystem-readonly`：`/sdcard/Download` 和 `/storage/emulated/0/Download` 的 statfs 类型是 `786f4256`（vboxsf 魔数 "VBox"），`/sdcard` 是 FUSE（`65735546`）。但 `/proc/self/mountinfo` 中**没有** vboxsf 条目，只有 4 条 fuse。MuMu，探针跑了 1 次。

### F26 真实流程统计（U28）
**判定：部分属实，几处口径需要说明。**
- 【重算 `W/cli_count.py`、`W/flowrange.py`】`E` 下 5 个目录共有 327 次 CLI 调用。按流程编号区间数：
  - MuMu 9 条流程：WPS 30、同程 5、Maps 15、MDN 19、GitHub 10、GOV.UK 14、OSM 9、JD 7、Bili 22。
  - API34 3 条：Firefox 17、NIST 17、regex101 36。
  - 合计 201 次。去掉 regex101 区间内的 session close/open 2 次，正好 199。"12 条、199 次"可以复现，但依赖这种区间划分。"9 条完成"沿用实施方的判定（JD、同程、Firefox 不算完成），我没有逐条重新判定。
- 【原件】GitHub `E/mumu/commands/101-github-open-issue`：`click: done; changed`，没有 unsettled，title 仍是 "Issues · python/cpython"，effect unchanged；`103` 才进入 issue 页。属实。
- 【原件】NIST `E/api34-recovery/commands/017`：`wait --contains Ethanol` 命中 `matchedText` 为 URL `https://webbook.nist.gov/cgi/cbook.cgi?Name=ethanol…`。宿主是 WebView Browser Tester。属实。
- 【重算 `W/actshare.py`】非 `--json` CLI 的 stdout 中，`"actions":[…]` 所占字节：
  - MuMu 50.7%（149 次调用，2.87 MB）。
  - API34 分目录：api34 42.6%、api34-recovery 60.8%、editor-pair 57.2%、ax-ic 61.2%，合计 58.8%。
  - "60%（API34）"只对应 api34-recovery 一个目录。建议写成"MuMu 约 51%，API34 各目录 43–61%，合计约 59%"。
- 【原件 `W/misuse.py`】CLI 误用：明确属于用法错误的有 3 次，即 `keys BACK`（没有该命令）、`key BACK`（参数不受支持）、recovery 构建中的 `select --help`（没有该命令）。如果把 `018 open` 缺少 handler，或 `206` 选择器不匹配也算进去，可以到 4–5 次。"4 次"的口径没有写明。
- 查帮助 10 次：5 个目录合计正好 10 次（mumu 4、api34 1、recovery 3、editor-pair 1、ax-ic 1）。但这 10 次是 327 次调用里的数；在 199 次的口径内只有 5 次。草案把两个口径混在一起。
- MDN：081 和 087 的输出里各有新 REF（`e748160918`、`e748161509`），082 和 088 直接点击新 REF，额外 CLI 调用确实是 0 次。不过这两份输出各约 34.7 KB，超过 `ops.py` 的 7000 B 直接打印阈值，操作者必须另外读文件。
- `ops.py` 的阈值：mumu、api34、recovery、editor-pair 为 7000 B（`mumu/ops.py:37`），**ax-ic 为 3000 B**。
- "约 11 次即时观察仍是旧页"**未能核实**：需要逐条人工判读，我只确认了 GitHub 101 这一例。

### F27 规模重算
**判定：叶命令属实；长选项、隐藏成员、规范句未能按原口径复现。**
- 【重算，用候选源码在 venv 里构建 Typer 命令树，`W/leaves.py`】
  - 叶命令：基线 47，候选 70。再加上可直接执行的 `notifications` 组，候选为 71，"47→71"成立。
  - 长选项：基线 38（去掉补全选项），候选 70（不计次级开关 `--no-print` 则为 69）。草案的 68 没有复现。
- 隐藏成员（`W/refl.py`）：反射调用里不同的成员名，基线 40、候选 67；`HiddenApi.kt` 中 lazy 成员，基线 44、候选 55。"34→57"的口径我复现不了，各种口径下增幅约 25%–68%。
- 规范句（`W/sent.py`、`W/norm.py`）：指引从 2 个文件 81,508 B 增加到 12 个文件 147,654 B（1.81 倍）；按关键词算的规范句从 149 到 277（1.86 倍）。"331→594"（1.79 倍）的倍数相近，绝对值的口径我复现不了。
- 编译期桩：`P/r2-coordinate/workspace/android/research/window-probe-stubs` 中，WindowInfosListener 有构造器、onWindowInfosChanged、register、unregister，加上 DisplayInfo 的 3 个字段；InputWindowHandle 有 7 个字段。合计 14 个，不计构造器为 13。这些桩在原型里，不在候选源码中。

### F28 r3 提交表没有归属的事项
**判定：属实，需要两点限定。**
- 【源码】上一版 F15 的位置应是 `DeviceFacts.kt:39`（锁态调用块是 `:37-40`），不是 36-38。
- 【原件】r3 主稿 `:278-295` 的 16 个提交里，没有任何一项涉及锁态参数。但完整处置底表 `integration-draft.md:144` 已经承认它是缺陷（"固定正确 device0"）。所以准确说法是：底表接受了，提交表没有承接。
- 效率对照：`integration-draft.md:305` 把 A1 推迟到"冻结单 View 之后"。第 16 项只写了"记录误操作、误拒、交付及实际成本"，没有新旧呈现的同后端对照。

### F29 结构记录封套的往返验证
**判定：属实，有一个限定。**
- 【原件】`P/r2-delivery/probe.py:10,164` 用的是 `difflib.unified_diff`，即按行的文本 diff。封套方案的验证只有 `R/r2-delivery.md:77` 的文字描述，包内没有脚本或输出。
- 【重算 `W/rt.py`，候选 3f45af6 的 `ui_view`/`ui_diff`】往返后 `render` 逐字相等，`changes` 为空：
  - Contacts seed（`stages/28-current-editor/runtime-api34/oracles/contacts-seed.raw.json`，其哈希与 `r2-delivery/results.json` 一致）：35 条记录。
  - Chrome 4226 页：`a-e9-first` 和 `a-single-warm` 为 98 条，`preparation-page` 和 `b-e9-warm` 为 99 条。实施方写 98，草案写 99，差别来自用的是哪一次采样。
- 限定：Contacts 的 35 条里有 21 条、Chrome 的记录里有 6–7 条 `key=None`，而 `ui_diff.changes` 完全忽略没有 key 的记录。

### F30 窗口缺席
**判定：属实。**
- 【原件】`P/r2-window-absence/runtime-api34/commands/contacts-back-wait-old-ref`，同一个 batch：
  - `back --settle 0`：done，samples 0。
  - wait `--gone --window 225`：done，`basis windowAbsent`。seq 7 的窗口列表是 [221,216]，seq 8 是 [221,216,226]，两次都没有 225，节点查询没有发起。
  - `focus e592373416`：not sent，原因 "Ref is absent from the current observation"，`dispatchStarted=false`。
- 研究 APK，API34，1 次。

---

## 二、草案"立场"中与事实直接矛盾的地方

1. **P5**："没有任何'无障碍选区失败而输入法选区成功'的样本"。**与原件矛盾。** 同一 Wikipedia 字段上，AX select 立即读回 3 次都是 differs（`wikipedia-caret-batch` 8..8、`wikipedia-range-batch` 6..9、`wikipedia-city-repair-batch` 8..9），而 IC `editor.select` 对同样的区间 2 次 DONE matches（`wikipedia-caret-ic-control` 8..8、`wikipedia-city-ic-control` 8..9，均在 `P/r2-text/runtime-api34`）。这里 AX 的"失败"来自读回时机，挪后读回的构建已经修好（F8），但样本确实存在。
2. **P7**："所有结束路径汇于 `invalidateLease` 一个接点"。worker 正常 Stop 不经过它（F18）。
3. **P14**：把稀疏臂变慢归因于"重新展开 0.891 s 加冷 worker"，漏了稀疏臂多采 2706 个节点（5485 对 8191）这个混杂因素，设备侧 RPC 多了 0.58 s（F13）。
4. **P3**："公开查询固定等 5 秒"。源码是最长 5 秒的阻塞上限（F11）。
5. **P16(b)**："报成'未发出'"。只有 close 抛 `Fault` 时才是这样，其它异常会以未处理异常退出（F2）。
6. **P15**：说 wake 改走 `injectInputEventToTarget` 就"不连 UiAutomation"。这在 wake 内部可行，但 home 和 recents 现在同样走 UiAutomation 的 `policyKey`（F23），草案没提。这一条只是遗漏，不算矛盾。
7. **P13**："额外调用 0 次"按 CLI 调用数成立，但两份 reveal 输出各约 34.7 KB，超过操作者直接打印的阈值（F26）。这一条是限定，不算矛盾。
8. **P18**："隐藏接口成员（候选 57）"我复现不了（F27），引用时建议标为实施方口径。

---

## 三、评审方上一版定稿三条说法的抽查

1. **§2.5"每次注册拿到的初始数组是空的"：没有被原件否定，但需要加条件。**
   - 【原件】`P/r2-coordinate` 与 `P/coordinate-window` 中 13 次记录（API34、MuMu API35、P3、P5）的 `initialWindowCount` 全部为 0。
   - 【源码】`WindowInfosListenerReporter.cpp` 中 `addWindowInfosListener` 返回的是**本进程缓存的最近一次值**（A14 `:55-57`）；最后一个监听者注销时才清空缓存（`:73-77`；15 `:79`，16 `:80` 同理）。A16 `WindowInfosListener.java:55-58` 也写明 "If there was nothing registered previously, then the data can be empty"。
   - 所以准确说法是："本进程没有其它活着的监听者时，初始数组为空。"按"每条命令注册、用完注销"的用法，这一点成立；如果同进程里有常驻监听者就不成立。
2. **§2.6.3"偏移以编辑器的真实文本为准"：被原件否定。**
   - 【原件】`E/api34-ax-ic/031-032`：AX select 请求 48..51 选中的是 AX 文本（51 个单位，Chrome 加了 `↵` 标记）中的 "503"，而 IC 读回的原生选区是 46..49（原生文本 49 个单位）。
   - 这说明 AX 选区偏移按无障碍节点文本解释，不按编辑器的真实文本。候选回执本身也标注了 `textSource: "accessibility node text; may include labels"`（见 F5 里 Bitwarden 和 Contacts 的回执）。
3. **§2.5 规则 7"首版只支持恒等的显示变换"：被新证据否定。**
   - 【原件】`P/r2-coordinate/runtime-p5/facts.json`：API34 上开启真实旋转后，显示矩阵为 `[0,1,0,-1,0,720,0,0,1]`。按恒等门的原型直接拒绝（"This probe supports only identity display transform"），而显式 `--expect` 的滑动正常完成。也就是说，任何横屏都会让默认接收者不可用。
   - 【源码】F4 表明交给输入系统的显示变换在 14/15/16 都只由平移、轴向缩放和 90° 旋转组成，必然保矩形，按平台的逐矩形语义支持并不难。
   - 样本：1 次旋转，API34。

---

## 四、未能核实的内容
- F26 中"约 11 次即时观察仍是旧页"：需要逐条人工判读，没有做。
- F26 中"9 条完成"：沿用实施方的分类，没有重新判定。
- F27 中长选项 68、隐藏成员 34→57、规范句 331→594 的具体口径。
- F17 的原始 hprof：只拿到摘录，原件不在包内。
- F5 中 2024 年以前的 Compose 发布版本没有逐一核对；PyPI 上的发布历史没有核对（F6）。
- F2 里哪些 close 路径会抛 `Fault`，没有逐一追查。

---

## 五、F1–F30 判定总表

| 编号 | 判定 | 要点 |
| --- | --- | --- |
| F1 | 属实 | batch 停批行号应为 `ui_batch.py:360-376`；windowsError 也会停批 |
| F2 | 属实 | "未发出"仅在异常为 `Fault` 时成立 |
| F3 | 属实 | 14/15/16 函数体相同，只有参数类型变化 |
| F4 | 属实 | 14/15/16 分别核对；ProjectionSpace.h 三版逐字相同 |
| F5 | 基本属实，需改措辞 | select 是发送前门；commit 照发，核验得 UNKNOWN；androidx main 2cda5a0d 与 compose-release ac6d2ce3 都没有覆写 |
| F6 | 属实 | 标签只有 v0.1.0/v0.1.1/v0.3.0 |
| F7 | 属实 | 需注明研究 Agent 构建，1 次 |
| F8 | 属实 | 失败 3 次全在 Wikipedia；前后不是同一个 APK |
| F9 | 属实 | 所有数字重算一致；四次都超出采集截止 75–323 ms |
| F10 | 基本属实，需改措辞 | AIDL 文件有变化，E9 所用签名与顺序未变 |
| F11 | 基本属实，需改措辞 | 5 秒是上限，不是固定等待；A14 在 :112 |
| F12 | 属实 | — |
| F13 | 属实 | 漏了节点数 5485 对 8191 的混杂，以及 windowsError |
| F14 | 属实 | 另有 :132 的吸收门 |
| F15 | 部分属实，需改措辞 | 扫描结果按口径为 11–13 对、129–137 个、43–44 个可点；"0 次"是因为语料里没有不可用字段 |
| F16 | 属实 | — |
| F17 | 属实 | 证据是摘录；退出后 7 个实例；ART 是 mainline 模块 |
| F18 | 基本属实，需改措辞 | worker Stop 不经过 invalidateLease |
| F19 | 属实 | +16 ms 时任务已存在但 visible=false |
| F20 | 属实 | 同包，媒体会话按 App 共享 |
| F21 | 属实 | 首次断开在远程调用前就置为 DISCONNECTED |
| F22 | 属实 | 新会话 open 也会被挡 |
| F23 | 属实 | home/recents 也走 UiAutomation 通道 |
| F24 | 属实 | — |
| F25 | 属实，需改措辞 | statfs 魔数是 vboxsf，mountinfo 里没有 vboxsf |
| F26 | 部分属实 | 199 可按区间复现；60% 只对应一个目录；帮助 10 次跨口径；误用口径不明；"约 11 次"未能核实 |
| F27 | 部分属实 | 叶命令 47→71 成立；长选项 38→70（不是 68）；隐藏成员与规范句复现不了 |
| F28 | 属实 | 行号应为 :39；底表承认缺陷但提交表没有承接 |
| F29 | 属实 | Chrome 记录数 98/99 取决于采样；无 key 的记录不进入 changes |
| F30 | 属实 | 研究 APK，API34，1 次 |
