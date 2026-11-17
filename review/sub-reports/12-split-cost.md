# 拆分成本评估：R0-C"六块拆出首次合入"是否现实（基于实际 git 操作）

评审对象：候选 `3f45af6`（40 个线性提交，基线 `7d5be70`）。所有实验都在独立克隆 `scratchpad/agents2/work12/repo` 及其 worktree 中完成，没有改动 REPO、PKG 或 `/home/user/androidctl`，没有 push。
证据标注：【实测】指在克隆里实际执行得到的数字；【源码推断】指读代码得出的判断；【设计建议】。Android 侧在本机没有 SDK，**Android 编译与 JVM 测试：未验证，需构建环境**。

---

## 0. 结论摘要

1. **六块全部拆出在技术上可行，不是"交织过深拆不动"。**【实测】六块都已在临时分支上从 HEAD 做 revert 并解决冲突，host 侧 `ruff check`、`ruff format --check`、`mypy` 全部通过，`pytest` 的失败集合没有新增（只减少）。六块合并在一条分支上（`split/all-six`）也做到了同样结果：净删 13,194 行，其中产品代码 Kotlin −2,899/+262、Python −1,359/+89。
2. **按难度分三档。** 干净可拆：zoom(37)、借贴(36)、后台屏整项(38)。可拆但需要按语义动手：通知写(31+32)、diff(8)。需要语义手术、且风险集中在 Android 侧无法验证的部分：IC(29)。没有一块属于"交织过深、不建议拆"。
3. **拆分的真实成本不在冲突数量，而在"静默合并"。**【实测】有 5 处 git 自动合并没有产生冲突标记，结果却是错的：残留的旧 `execute` 函数体、丢失的 import、`displayShape()` 退回只读主屏、`missingFactReason` 被旧的内联分支替换、遗留的 `"zoom"` 判断。Python 侧靠 pytest 和 ruff 全部抓到了；Kotlin 侧只能靠人工比对和 grep。**因此 Android 构建和 JVM 测试是每一块拆分的硬门槛**，本报告的 Kotlin 部分只做到了"静态无残留引用"这一步。
4. **第 38 项"只保留内部贯穿"（R2-C）可以干净地切出来，但切出来的正是实施方担心的半截接口。**【源码推断】main Kotlin 中约 75 行新增、36 行删除属于贯穿（13 个文件，约 30 个 hunk），只有 main 时这些 `displayId` 全部恒为 0。它不进协议，代价不大，但也没有当下收益。没有任何 git 操作能直接产出 R2-C，只能手工制作。相比之下，**整项拆出 38 的实测成本最低**：只有 2 个后续提交受波及，revert 冲突 7 个文件、7 个 hunk，另有 2 处测试需要跟着改。它还顺带消除了本机 23 个已知 Linux 失败中的 22 个（全部是第 38 项的主机偏好测试）。
5. **六块之间基本独立。**【实测+源码】唯一需要注意顺序的地方是 37→38 重叠（`VisualInput`、`UiRun`、`ImageRegion`），以及 29、31 先后重构了 `UiBackend`、`UiRun`、`RpcMethods` 的同一组分派代码。建议的拆出顺序是从新到旧：38 → 37 → 36 → 32 → 31 → 29 → 8。diff(8) 只在 host 侧，与其它块没有交叉。
6. **"在顶端追加 revert"与"重写 40 项序列"相比。**【实测】一次性 rebase 丢掉 7 个提交，会让 11 个后续提交冲突，至少 36 个文件、62 个 hunk（这是下界）。从 HEAD 追加 revert 时，逐块冲突合计约 49 个文件次。按 blame 统计，有 14 个保留下来的后续提交改过这六块引入的行。追加 revert 不符合"修正归回所属提交"的要求；重写符合，但每个被改的中间提交都要重新构建验证。**折中做法**：尾部五项（31、32、36、37、38）只影响第 33–40 项，用 rebase 丢弃，代价接近 revert。早期两项（8、29）用 rebase 会波及 13 个和 9 个后续提交，如果决定拆，应在顶端追加 revert，或者作为整序列重写的一部分接受这笔代价。
7. **推荐切法（成本最低、收益明确）**：38、37、36、31+32 整项拆出（rebase 丢弃）；8 diff 拆出（host-only，可在本机完整验证）；29 IC 按 revert 拆出，但要以 Android 构建、JVM 测试和一次真机 attach/detach 冒烟为合入门槛。"只摘公开面"在这几块上都不划算，理由见 §5。相对 R0-B，多出的工作量估计为 1.5–3 个工程日（不含设备补证），换来主线少约 4.2k 行产品代码、少约 2.6k 行契约样本，Linux host 测试也几乎全绿。

---

## 1. 总表

冲突计数：从 HEAD 单独 revert 该块，统计含冲突标记的文件数和 hunk 数。rebase 一列用"冲突方取被重放提交的版本"继续推进，因此是**下界**。blame 一列统计后续提交的改动 hunk（±1 行上下文）落在该块引入行上的次数，这个口径不受冲突解决方式影响。

| 块 | 提交 | 从 HEAD revert：冲突文件/hunk【实测】 | rebase 丢弃：冲突的后续提交/文件/hunk（下界）【实测】 | blame：改过其行的后续提交数【实测】 | 拆后 host 检查【实测】 | 拆后净删（产品 Kotlin / Python / 契约）【实测】 | 分级 |
|---|---|---|---|---|---|---|---|
| 38 后台屏（整项） | 9321313 | 7 / 7（+2 处后续测试跟改） | 2 / 7 / 7 | 2（39、40） | ruff✓ format✓ mypy✓；pytest 1 failed / 1758 passed（失败集合 ⊂ 基线，消除 22 个） | −877+168 / −264+13 / −342+174 | 干净可拆 |
| 37 zoom | cace585 | 7 / 9（含 1 个 UD） | 2 / 7 / 9 | 3（38、39、40） | 全✓；23 failed（= 基线集合） | −336+33 / −100+11 / −57+1 | 干净可拆（有 2 处静默合并需改） |
| 36 借贴 | 8d1b45b | 4 / 4 | 2 / 4 / 4 | 2（37、38） | 全✓；23 failed（= 基线） | −521+27 / −35+7 / −569 | 干净可拆 |
| 31+32 通知写 | 9b67f68、483d755 | 先 32：3/3（全是文档）；再 31：9/≈22 | 2 / 9 / 20 | 31→5、32→1 | 全✓；22 failed（= 基线去掉一个被删参数） | −663+32 / −186+2 / −1264 | 可拆，需语义手术（分派重构与剪贴板共用） |
| 29 IC | d5ae012 | 15 / 22（含 2 个 UD） | 5 / 15 / 15 | 9 | 全✓；23 failed（= 基线） | −547+37 / −91+1 / −438 | 可拆，需语义手术；Android 侧风险最高 |
| 8 diff | b667727 | 8 / 42（ui_commands 28 个 hunk） | 5 / 8 / 21（严重低估） | 13 | 全✓；23 failed（= 基线） | 0 / −688+56 / 0（另删测试 1,807 行） | 可拆，需语义手术，但只在 host 侧，可完整验证 |
| **六块合并** | 上述 7 个 | 增量冲突（在已拆 38 的基础上依次拆）：37 9/7，36 4/4，32 1/1，31 6/5，29 12/16，8 8/≈9 | 11 / 36 / 62 | 保留下来的后续提交中 14 个受波及 | 全✓；**1 failed / 1550 passed**（失败集合 ⊂ 基线） | Kotlin −2899+262 / Py −1359+89 / 契约 −2639+171 | 可行 |

基线事实【实测】：本克隆的 venv（Python 3.12、`-c constraints-ci.txt -e .[dev]`）在 `androidctl/tests` 上的结果是 23 failed / 1799 passed / 17 skipped。简报写的 1850 passed 可能把 `tools/release/tests` 也算了进去，判断以失败集合为准。23 个失败中有 22 个是 `test_display.py` 或 `test_bad_preference_*`，都属于第 38 项的主机偏好；剩下 1 个是 `test_ui_batch.py::test_explicit_only_without_diff_does_not_prepare_history`。

临时分支（都在 `scratchpad/agents2/work12/repo`，各有 worktree）：
`split/rv38-display`（wt38）、`split/rv37-zoom`（wt37）、`split/rv36-borrowed-paste`（wt36）、`split/rv3132-notif-write`（wt31）、`split/rv29-ic`（wt29）、`split/rv08-diff`（wt08）、`split/all-six`（wtall）。rebase 测量分支为 `rb38 rb37 rb36 rb3132 rb29 rb08 rb-all6`，只用于计数，内容不可用。脚本放在 `work12/`（`measure_revert.sh`、`measure_rebase.sh`、`blame_overlap.py`、`hunks38.py`、`runwt.sh`、`residue.sh`、`orphans.sh`），结果在 `work12/results/`。

---

## 2. 逐块详述

### 2.1 第 37 项 zoom（cace585）

- **依赖**【源码+实测】：整文件属于本块的有 `ImageRegion.kt`(211)、`ImageRegionTest.kt`。需要手术的共用文件有 `VisualInput.kt`（保留原生 Bitmap 的 `ImageState`、`captureGeometry`、`softwareSource`）、`UiBackend.kt`、`UiRun.kt`、`UiDelivery.kt`（`imageCapture`）、`UiPublication.kt`、`RpcMethods.kt`（UI_READ_ONLY 里的 `"zoom"`），以及 host 的 `ui_commands.py`（zoom 命令约 45 行）、`ui_client.py`（prepare_ui 的 zoom 校验约 30 行、result_line）。前置依赖：21/22 项的图像帧（不拆）。后续引用：38 在 `ImageRegion` 加了 `expectedDisplayKind` 白名单，在 `VisualInput` 加了 stream 分支；38 往契约 `displayRouting.targetMethods` 加了 `"zoom"`；`displays.md` 写着"work display 保留一份软件源供 zoom"；第 40 项新增的测试 `anImageWithoutAPreviouslyPublishedTreeCannotSupplyABaseline` 直接调用 zoom 的 `UiDelivery.imageCapture`。
- **revert**【实测】：7 个文件、9 个 hunk（ImageRegion 是 UD）。除冲突标记外还有两处**静默合并**：一是 `VisualInput.geometry()` 被合成为引用未定义的 `facts`，`displayShape()` 回退为只读 `deviceFacts()`（主屏）；二是 `UiRun.response()` 的 `missingFactReason()` 被换回旧的内联 if 链，丢掉了第 38 项的 `"wake","apps"` 文案。两处都已手工改正（提交 72fe5d6）。另外需要跟改第 40 项的测试、`test_display.py`、`DisplayTargetTest`、契约方法表和两处文档句子。
- **rebase**【实测】：受影响的是第 38 项（5 个文件、7 个 hunk）和第 39 项（2 个文件、2 个 hunk）。
- **拆后**：host 全绿，失败集合等于基线。Kotlin 被删符号 3 个，残留引用 0。
- **代价外的注意**【源码推断】：`softwareSource()`（HARDWARE Bitmap 转软件）是 zoom 引入的，38 的主屏截图路径也在用。拆掉 zoom 后主屏回到直接缩放 HARDWARE Bitmap 的旧路径，需要在真机上确认 `scale`、`marks` 写入没有回归（需设备）。
- **分级：干净可拆。**

### 2.2 第 36 项借贴（8d1b45b）

- **依赖**：整文件属于本块的有 `BorrowedPaste.kt`(354)、`BorrowedPasteTest.kt`(351)。共用文件：`ClipboardAccess.kt`（63 行改动）、`ClipboardOperations.kt`、`UiActions.kt`（`paste()` 与 `NodePasteTarget` 约 106 行）、`UiBackend.key()`、`UiRun`、`KeyPlan`、`input_keys.py`（`--via paste`）、`ui_client.py`（borrow/paste/restore 的 result_line）。前置依赖是第 35 项结构剪贴板，本块只依赖它，不改变它。后续：38 在 paste 代码里把 `dispatch` 改成 `dispatchInput`，并在契约 `pasteResults` 中加了 `expectedDisplayKind`。
- **revert**【实测】：4 个文件、4 个 hunk，都是"38 改过的行处于被删块内"，机械处理即可。`reference.md` 的同一行表格要同时保留 zoom、删掉 paste，需要理解但很小。
- **残留**：`reference.md` 中还有一行 `data.borrow…` 的字段索引没被 revert 删掉，合并分支已清理。35 项里 null 的"当次准入"采样本次没动；R3-B 建议的简化是独立工作，估计 20–40 行。
- **拆后**：host 全绿，失败集合等于基线。Kotlin 被删符号残留 0。
- **分级：干净可拆。**

### 2.3 第 31、32 项通知写（9b67f68 action/open/dismiss，483d755 reply）

- **依赖**：整文件属于本块的有 `NotificationActions.kt`(≈478)、`NotificationReplyInput.kt`(52)、`NotificationActionsTest.kt`。共用文件：31 把 `NotificationRead` 重命名为 `NotificationOperations` 并改成可注入 `operation` 的通用执行器，同时引入 `NotificationMethods{reads,writes}`、`NotificationRequest.validate`、host 的 `structured_json`、`NOTIFICATION_COMMANDS`。**第 35 项剪贴板直接沿用了这套结构**（`CLIPBOARD_COMMANDS` 与 `structured_json` 并列，`RpcMethods.publishesObservation` 的排除表）。还有 `HiddenApi`（cancel、sendAndReturnResult）、`NotificationAccess`（prepareCancel）、`ActivityTasks`（`ACTIVITY_FACT_COUNT` 改为 internal）。`notifications.open` 的 landing 复用的是**第 28 项**的 `activityTasks()`/`boundedActivityFacts`，不是第 38 项。38 把 `notifications.open` 列为不需要 display 的设备能力（`test_display.py`、`DisplayTargetTest`、契约 `deviceUiMethods`）。
- **revert**【实测】：先 revert 32，只有 3 个文档冲突。再 revert 31，9 个文件约 22 个 hunk，需要的决定是"保留 31 的通用化骨架还是退回 `NotificationRead`"。我退回了 `NotificationRead`，并把 `NotificationMethods.*` 替换为 `NotificationProjection.commands`，`structured_json` 和 `CLIPBOARD_COMMANDS` 因剪贴板仍在使用而保留。还有一处**静默合并**：`test_notifications.py` 丢了第 31 项加的 `local_call` import，而后续测试在用，pytest 报了 16 个 setup error 后补回。文档里 `notifications.md` 是第 34 项新建的文件，revert 碰不到它的"Notification actions / text replies"两节，需要手工删除。
- **rebase**【实测】：受影响的是第 34 项（3 个文件、3 个 hunk，都是文档）和第 35 项（6 个文件、17 个 hunk，`RpcMethods`、`UiBackend`、`UiRun`、`WireContractTest`、`ui_client`、契约）。
- **拆后**：host 全绿，失败集合等于基线（少了一个 `notifications.open` 参数化用例）。契约少 1,264 行样本。
- **分级：可拆，需语义手术。** 手术集中在"通知/剪贴板共用的分派骨架保留到什么程度"。

### 2.4 第 29 项 IC（d5ae012）

- **依赖**：整文件属于本块的有 `IEditorBridge.aidl`、`EditorBridge.kt`(96)、`CurrentEditorInput.kt`(164)、`EditorSurrounding.kt`(129)、`editor_input.py`，以及各自的测试。共用且要手术的文件：
  - `WorkerConnection.kt`（142 行改动）。IC 引入了 `Attached(target, status, editor, death)` 连接记录和死亡回调，第 33 项（gateway 控制器状态）建立在这套结构之上。拆 IC 时必须**保留 Attached 结构而去掉 editor**，涉及 death recipient 的身份判定方式（我改为按 `Attached` 实例判断，用了 `lateinit` 自引用）。这是六块中**唯一改动连接生命周期的地方，只能靠 Android 构建和真机 attach/detach 验证**。
  - `ui_batch.py`：IC 为 `editor commit/select` 引入了"嵌套命令组"的 batch 解析，后来通知组（`NotificationsGroup`）和剪贴板组都依赖它，**必须保留**。
  - `UiBackend`（`editor()`、`observedOperation` 拆分）、`UiRun`、`RpcMethods`、`IExecutor.aidl`、`WorkerService.setEditorBridge`、`BrowserAccessibilityService.kt`（65 行）、`res/xml/browser_accessibility_service.xml`、`strings.xml`。
  - 38 在 `editor()` 里加了 main-only 检查，并改了 `CurrentEditorInput`（DisplayKind）。IC 本身**不依赖** 38，是 38 依赖 IC 的存在。
- **revert**【实测】：15 个文件、22 个 hunk（2 个 UD）。有两处**静默合并**：一是 `UiBackend` 中旧版 `execute` 的整段函数体被拼接进了 `validateRequest()` 末尾（Kotlin 编译会报错）；二是 `WorkerConnection` 的 `manage()` 和 `detach()` 被悄悄退回到第 33 项之前的写法。我的解决办法是：Python 的 `ui_batch`、`ui_client`、`ui_commands` 和 `WorkerConnection` 先恢复为 HEAD 版本，再手工删除 editor 部分。
- **rebase**【实测】：5 个后续提交冲突（30、31、33、34、38），共 15 个文件、15 个 hunk。
- **拆后**：host 全绿，失败集合等于基线。Kotlin 被删符号残留 0（只剩 `DisplayTargetTest` 中 `"editor.commit"` 一行，已删）。
- **分级：可拆，需语义手术。** Android 侧风险最高（连接生命周期），需构建环境+需设备。

### 2.5 第 8 项 diff（b667727）

- **依赖**：整文件属于本块的有 `ui_diff.py`(357)、`test_ui_diff.py`(1,033)。共用文件：
  - `history.py`：`History` 类、`BASELINE_NAME`、配额里的 baseline 组。第 9 项 batch 的"保护本次交付文件"也用了 `History.saved/make_room`，**History 类要保留，只去掉 baseline**。
  - `ui_client.py`：`View.diff`、`delivery_scope` 的 comparison/candidate、`deliver`/`render_observation` 的 comparison 分支、`commit_baseline`/`baseline_failure`/`require_diff_destinations`。`ui_diff.source_facts` 不属于 diff 语义，要挪回 `ui_client`。
  - `ui_commands.py`：之后**每加一条 UI 命令都会带一个 `diff: Diff = False` 参数和一个 `diff` 实参**，HEAD 中共 48 行，分布在约 25 条命令上，这就是 revert 时 28 个 hunk 的来源。
  - `ui_batch.py`：第 9 项 batch 的终结阶段（`finish(candidate, protected, …)`，约 80 行）以及约 20 个 batch 测试**全是为 diff 基线服务的**。
  - 第 39 项阅读选项在 `ui_diff` 里加了 11 行（投影比较），`notifications` 裸命令的 `--diff` 也在其中。
  - 只涉及 host，与 Android、契约无关。
- **revert**【实测】：8 个文件、42 个 hunk。我没有逐个解决冲突，而是"恢复 HEAD 再做删除手术"：改动 5 个源文件（history −51、ui_batch −101、ui_client −177、ui_commands −58），删除 22 个仅测 diff 的 batch 测试、2 个通知测试、node_actions/range 各 1 个测试，以及 1 个参数行；文档 7 个文件，其中 `observations.md` 的"Explicit Diffs"一节约 55 行。
- **rebase**【实测】：5 个后续提交冲突，共 8 个文件、21 个 hunk。这是严重低估：取"被重放提交版本"会把 `ui_diff.py` 带回来。blame 显示有 **13** 个后续提交改过 diff 引入的行（9、11、15、17、19、21、22、26、30、34、37、38、39），所以**重写历史时 diff 是最贵的一块**。
- **拆后**：host 全绿，失败集合等于基线。这是唯一能在本机**完整**验证的一块。顺带副作用：`delivery_scope` 的 `track_history` 参数变成没有调用方传 True，`History.groups()` 退化成 `scan()`，可以再顺手删掉约 10 行。
- **分级：可拆，需语义手术，但验证完整、风险低。**

### 2.6 第 38 项后台屏整项（9321313）

- **依赖**：整文件属于本块的有 `WorkDisplay.kt`(150)、`TargetManagement.kt`(119)、`DisplayTarget.kt`(31)、`DisplayTargetTest.kt`(192)、`test_display.py`(567)、`skills/androidctl/displays.md`(90)。其余 66 个文件都要改，契约有 66 个 hunk（每条 UI 请求样本都加了 `expectedDisplayKind`）外加 56 行 `displayRouting` 节。后续：第 39 项的 `test_ui_arguments` 中 wake 只接受 `--timeout` 的两个测试依赖 38 对 wake 的改动；第 40 项给 `Sessions` 加了 `acquireExistingUi(owner, expected)`，`UiDelivery.read` 用它，`SessionsTest`、`UiDeliveryTest`、`UiTreeBudgetTest` 和 1 个契约样本也用到了 kind。
- **revert**【实测】：7 个文件、7 个 hunk。需要的决定是把第 40 项的修复（`ui.read` 不创建新租约）改写成不带 kind 的 `acquireExistingUi(owner)`：13 行，保留了"清理中仍 busy"的语义。另外改写第 40 项的 4 处测试，删掉 2 个依赖 wake 新行为的第 39 项测试。
- **拆后**：host 全绿，失败从 23 降到 1。被删的主类符号在 Kotlin 中残留引用为 0。
- **整项拆出顺带丢掉的非后台屏改动**【源码推断】：38 里还捆了几处与 virtual 无关的改动，约 60 行：
  - `dispatchInput()`：在每次派发前重查输入就绪；
  - `UiBackend.clear()`：清理失败聚合，新增 `recordCleanupFailure`；
  - `RpcMethods.publishesObservation()`：把四处重复的"是否发布观察"判断合并成一个函数；
  - `wake`：不再做事后 settle，CLI 只接受 `--timeout`；
  - `requireResources` 小重构，以及 `requireOperatingWindow` 改名和提示文案。
  如果需要这些改动，应作为一个独立的"main-only 改进"小提交补回，估计 60–80 行加对应测试。
- **分级：干净可拆。**

---

## 3. 第 38 项 hunk 归类（R2-C"保留内部贯穿、拆出 virtual"是否可行）

方法：先用 `hunks38.py` 按关键词把每个 hunk 粗分到 V（virtual/建屏/偏好）、K（种类协议）、T（target/displayId）、O（其他），再逐个阅读 main Kotlin 的 T 类和 O 类 hunk 人工改判【源码推断】。

第 38 项总量【实测】：main Kotlin 33 个文件 +866/−154；Kotlin 测试 +552/−21；Python 源码 +264/−13；Python 测试 +802/−6；契约 +341/−174；文档 +125/−15。

| 归类 | 文件（main Kotlin，约数） | 行数（+/−） | 说明 |
|---|---|---|---|
| **A 内部贯穿**（main-only 也会"用到"，但值恒为 0） | AccessibilityConnection(+15/−4，`windows()` 按 displayId 选、`beginCheck`/放大镜带 displayId)、TreeCollector(+4/−4)、UiSignals(+8/−2，事件按 displayId 过滤)、UiObservation(+6/−3)、DeviceFacts(`deviceFacts(displayId)` +7/−4)、NodeFacts(−2，删除 `defaultWindows`)、AppInput(+1/−1)、VisualInput(target displayId、targetFacts，+10/−8)、UiBackend(targetFacts，Collector/Connection 传参，+12/−5)、DisplayTarget(只留 MAIN 约 8)、Sessions(`Lease.target`、status 中的 target，+2)、UiPublication/UiPostAction(+3/−3) | **约 +75 / −36，13 个文件，约 30 个 hunk** | 行为不变；`NodeFacts.defaultWindows` 这类散落的 `== 0` 过滤收拢到一处，这是唯一的"当下收益" |
| **B virtual 专属** | WorkDisplay(150)、TargetManagement(119，含把 session.open/close/takeover 从 WorkerService 搬过来，WorkerService −32)、ActivityTasks(+139，virtual 任务预检与落点)、ActivityLaunch(+105)、HiddenApi(+19，liveTasks、captureDisplay)、DisplayTarget(kind/requested/stream 约 23)、Sessions(prepareChange/publish/acquireUi/requireTarget 约 +85/−25)、RpcMethods(requiresDisplay、SESSION_COMMANDS 约 12)、UiBackend(targetInputReady 的 VIRTUAL 分支、targets、stream close 约 30)、DeviceFacts(mainReadyForWorkDisplay 7)、SystemActions(home/recents/notifications 拒绝 5)、CurrentEditorInput(8)、ImageRegion/UiDelivery(2)、VisualInput(stream 分支 8)、UiScope.operatingWindow 与 UiCondition/UiActions 使用处(约 15)、Management*/Gateway/strings(11) | **约 +735 / −90** | 加上 host 的 `app.py`(display 命令 95 行)、`workspace.py`(偏好 93 行)、`ui_batch` 冻结偏好(21 行)、`ui_client` kind(11 行)，以及契约 66 个 hunk 和 `displays.md` |
| **C 与显示无关的捆绑改动** | dispatchInput 重查就绪、清理聚合、publishesObservation 合并、wake 不 settle、requireResources | **约 +60 / −30** | 应独立成提交，与本项去留无关 |

判断：
- **R2-C 可以干净切出**，A 类与 B 类在 hunk 粒度上基本可分。唯一缠在一起的是 `UiScope.operatingWindow`：它在 displayId≠0 时改用 `focused`，属于 B；但 `UiActions`、`UiCondition`、`VisualInput` 对它的调用替换了原来的"active 且非 IME"判断，只留 main 时应退回旧判断。
- **但切出来的 A 正是"临时 main-only 接口"**：13 个文件中的 `displayId` 参数和 `Lease.target` 在主线上恒为 0 或 MAIN，不经过协议，也没有测试能区分它们与常量的差别。实施方的担心**属实**，不过规模只有约 110 行，而且不对外。
- **git 产不出 R2-C**：必须整项 revert 后按 A 手工补回约 110 行，或者手工把 38 削成 A。两种都要 Android 构建，并重新跑 `UiSignalsTest`、`UiScopeTest`、`DisplayTargetTest` 中与 main 相关的部分。估计 0.5–1 个工程日，外加构建验证。
- **整项拆出与整项保留的比较**：整项拆出的实测冲突为 7+7，host 测试全绿，并消除 22 个 Linux 失败。代价是以后重新引入 virtual 时要重放约 110 行贯穿，加上与那时代码的冲突；另外 C 类约 60 行如需要要单独补回。整项保留（R2-A）则要修 D3（`app.py:839` 发 RPC 前写偏好）、D4（`ActivityLaunch` 落点判定）、T7（文案和 status 显示），约 30–60 行；但协议必填 kind、主机偏好文件和契约 66 个 hunk 会全部进入主线，22 个 Linux 偏好测试的可移植性问题也还要另外修。
- **【设计建议】**：主线选"整项拆出 + C 类单独补回"，第 38 项原样保留在分支上，等 P3 真机探针结论出来再决定是否回来。不建议做 R2-C 的手工切分：它的收益只在 virtual 回来时才兑现，而回来时重放 A 类的成本也就是现在切分的成本。

---

## 4. 六块之间的依赖与拆出顺序

| 关系 | 证据 | 影响 |
|---|---|---|
| 38 → 37 | 38 改了 `ImageRegion` 的参数白名单、`VisualInput` 的 stream 分支、契约 `displayRouting` 中的 zoom、`displays.md` 中的 zoom 句子 | 先拆 38 再拆 37 时，合并分支上 `VisualInput` 可以直接取 `8d1b45b` 版本（37、38 之后无人再改），比单独拆更省 |
| 38 → 36 | 38 改了 paste 代码里的 `dispatchInput`，以及契约 `pasteResults` 的 kind | 机械冲突 |
| 38 → 31 | `notifications.open` 被列为设备能力（测试与契约） | 机械删除 |
| 38 → 29 | 38 在 `editor()` 中加 main-only 检查，并改了 `CurrentEditorInput` | 机械冲突 |
| 31 → 35（保留项） | 剪贴板沿用 31 的 `structured_json`、`*_COMMANDS`、分派骨架 | 拆 31 时这些要保留，属于语义手术 |
| 29 → 30–35（保留项） | batch 嵌套命令组、`WorkerConnection.Attached`、`observedOperation` 拆分 | 拆 29 时这些要保留 |
| 8 → 9、39（保留项） | batch 终结流程、history 配额、`ui_diff` 投影 | 拆 8 时会顺带简化 9 |
| 36 → 35 | 借贴依赖 35，不反向依赖 | 无 |
| 31 open → 28 | landing 复用第 28 项的任务读取，不是 38 | 无 |
| 37 截图路径 → 38？ | 不依赖；是 38 在 zoom 之后修改了它 | 无 |

**建议顺序**：38 → 37 → 36 → 32 → 31 → 29 → 8，从新到旧，每步都基于上一步已解决的状态。合并分支 `split/all-six` 就是按这个顺序做的。各步增量冲突【实测】：38 为 7/7；37 为 9/7（含 4 个 DU/UD，都是"38 已删的文件"）；36 为 4/4；32 为 1/1；31 为 6/5；29 为 12/16；8 为 8/≈9（以 cherry-pick 单块解法的方式测得）。合计约 47 个文件次、约 49 个 hunk，与六块单独拆的总和相当，**一起拆不会额外放大成本**。

我解决冲突的步骤数（作为难度的粗略代理，每步是一次读写或命令往返）：36 约 4 步；37 约 8 步（另加 1 步修静默合并）；32+31 约 12 步（另加 1 步补 import）；38 约 10 步；29 约 13 步；8 约 16 步；合并分支额外约 20 步。

---

## 5. 替代切法比较

| 切法 | 适用块 | 要动多少【源码推断，按 HEAD 计】 | 留下的负担 |
|---|---|---|---|
| **整项 revert 或 rebase 丢弃**（本报告实测） | 全部 | 见总表 | 无死代码；需要重新导出契约，并做 Android 构建与 JVM 测试 |
| **只摘公开面**（CLI 注册、RpcMethods 方法表、skill 文档） | 38 | 删 `app.py` 的 display 组约 95 行和 `workspace.py` 偏好约 93 行，`ui_batch` 冻结逻辑约 21 行，host 固定发送 main；删 `displays.md` | Android 侧约 600 行 virtual 代码仍可经 RPC 到达（`display.create` 仍在 `SESSION_COMMANDS`）；协议必填 kind 与契约 66 个 hunk 仍在；22 个 Linux 偏好测试会随 host 偏好一起消失 |
|  | 37 zoom | 删 zoom 命令约 45 行、`prepare_ui` 校验约 30 行、方法表 1 处、文档约 50 行 | **有运行时成本的死代码**：每次截图后每个租约仍保留一份完整原生 Bitmap（`images.md` 自己写了"即使不用 zoom 也占内存"），除非同时改 `VisualInput`，而那就等于 revert |
|  | 36 借贴 | 在 `input_keys.validate` 中拒绝 `--via paste`（约 5 行），文档约 40 行 | `BorrowedPaste.kt` 354 行和契约样本 569 行变成死代码；Android 的 `key()` 分支仍可经 RPC 触发 |
|  | 31+32 | 删 4 个子命令约 230 行、方法表的 `writes`、文档约 75 行 | Android 写路径约 660 行仍可经 RPC 到达；契约 1,264 行仍在，并且仍会参与 `WireContractTest` |
|  | 29 IC | 删 `editor` 组约 35 行、方法表、文档约 50 行 | **运行时仍然活着**：每次 worker attach 都会创建 `EditorBridge` 并调用 `setEditorBridge`，无障碍服务仍承载 bridge。死代码不是"不可达"，而是"不可调用但持续运行" |
|  | 8 diff | 删 `Diff` 选项 48 行、文档约 60 行 | `ui_diff.py` 357 行、history 基线、batch 终结约 80 行全部变成死代码。diff 只在 host 侧，revert 已经完整验证，**摘公开面没有任何优势** |
| **编译期或构建期开关**（BuildConfig/flag 控制方法表） | 任意 | 每块约 10–20 行开关，加 CI 双配置 | 增加两套构建配置和测试矩阵，正是用户要避免的过度工程。**不推荐** |
| **只摘公开面，协议字段保留并标注"不承诺兼容"** | 38、31+32 | 同"只摘公开面"，另加契约注释 | 公开文档干净；内部 docs 仓与契约仍要维护；字段一旦被外部脚本用上就难收回。只适合"近期大概率会回来"的块 |
| **整项保留，只修 D3、D4、T7**（R2-A） | 38 | D3 约 15 行（`app.py:839`，改为先写偏好再发 RPC）、D4 约 20 行（`ActivityLaunch`，只有正面观察到落在别处时才非零退出）、T7 约 10–20 行（文案加 status 显示偏好）；另外 22 个 Linux 测试的可移植性要修（量未评估） | 协议必填 kind、主机偏好与跨进程锁、契约 66 个 hunk 成为长期契约 |

结论：在实测成本这么低的前提下，"摘公开面"只对 38 有一点吸引力（host 侧最省事），但它留下了一条可经 RPC 到达的 Android 功能和必填协议字段，后续维护成本比整项拆出更高。其余五块的 revert 都已经实测可行，摘公开面在它们上面没有优势。

---

## 6. 结论与推荐切法

**R0-C 现实吗？现实。** 六块全部拆出的 host 侧工作已经在 `split/all-six` 上完整做完并验证，Kotlin 侧也静态检查到被删符号零残留。剩下的不确定性全部在"Android 能否编译、JVM 测试与契约能否重新导出通过、IC 拆除后连接生命周期是否正常"上，**需构建环境/需设备**。

**推荐的具体切法**【设计建议】：

| 块 | 处置 | 方式 | 合入门槛 |
|---|---|---|---|
| 38 后台屏 | 整项拆出；把 C 类约 60 行（dispatchInput、清理聚合、publishesObservation、wake）作为独立 main-only 提交补回 | rebase 丢弃（只波及 39、40） | Android 构建；`SessionsTest`、`UiDeliveryTest`；重新导出契约 |
| 37 zoom | 拆出 | rebase 丢弃（波及 38、39、40；38 被丢后实际只剩 39、40） | 构建；确认去掉 `softwareSource` 后主屏截图和 marks 在真机上不回归 |
| 36 借贴 | 拆出；35 的 null"当次准入"按 R3-B 简化 | rebase 丢弃 | 构建；`ClipboardTest` |
| 31+32 通知写 | 拆出，保留 31 引入、现被剪贴板共用的 `structured_json` 与 `*_COMMANDS` | rebase 丢弃（波及 34、35） | 构建；`NotificationReadTest`、`WireContractTest`；契约导出 |
| 29 IC | 拆出 | 追加 revert，或作为重写的一部分（波及 30、31、33、34、38） | **构建 + JVM 测试 + 一次真机 worker attach、detach、死亡重连冒烟**（WorkerConnection 改动） |
| 8 diff | 拆出 | 追加 revert（在 host 完整验证；rebase 会波及 13 个提交） | host 测试（已过）；docs 仓同步 |

**相对 R0-B 的增量**【设计建议，基于实测冲突规模的估计】：
- 工作量：以"顶端追加 revert"计，六块冲突解决与 host 验证约 1 个工程日（我在这里已经完成一遍，可以直接参照分支）。Android 构建修错、JVM 测试、契约重导出约 0.5–1 天。docs 仓同步约 0.5 天。合计 **1.5–3 个工程日**，不含设备补证。如果改为重写 40 项序列，需要另加约 1–2 天：要解决 11 个后续提交，每个都要可构建。
- 收益：主线少约 2.6k 行 Kotlin 产品代码、1.3k 行 Python 产品代码、2.5k 行契约样本、约 5.8k 行测试、约 430 行公开指引。没有了必填 kind 协议字段和主机偏好文件。本机 Linux host 测试从 23 个失败降到 1 个。R0-B 里本来要修的 D3、D4、T7 随 38 一起移出主线，不再需要修。
- 损失：六块能力（各自的补证计划见主报告 §6），以及 38 中 C 类的小改进（建议补回）。

**线性历史的要求**：用户要求"修正要归回所属提交"，所以**重写 40 项序列更符合要求**。追加 revert 会在历史里留下"加 13k 行再删 13k 行"的噪声，也违背了序列作为评审叙事的用途。实测代价对比：
- 尾部五项（31、32、36、37、38）用 rebase 丢弃，只波及第 33–40 项。逐块冲突是 2 个提交 × 4–9 个文件，代价与 revert 几乎相同。**这五项应当直接从序列中删掉。**
- 早期两项（8、29）用 rebase 丢弃会分别波及 13 个和 9 个后续提交（按 blame）；冲突解决的实测下界是 5 个提交、8 个文件、21 个 hunk，以及 5 个提交、15 个文件、15 个 hunk。真实成本更高，因为每个中间提交都要能单独构建。如果用户坚持线性叙事，就接受这笔约 1–2 天的额外成本。否则，对这两项在序列末尾追加两个"移除 X"的 revert 提交，是成本最低的妥协。

---

## 7. 公开指引与 docs 仓受影响的范围

公开指引（本仓 `skills/androidctl/*`、`README.md`）：已在各分支中实际修改，合并分支共改 11 个文件，+41/−432 行。需要特别注意的是第 34 项后来新建的 `notifications.md`、`input.md` 和 `reference.md` 主题表：**revert 碰不到这些后建文件里的段落，只能手工删**，`reference.md` 的同一表格行被五块轮流修改，是文档冲突的热点。

独立 docs 仓（`PKG/source/androidctl/docs/`，只定位，不实操）：
- zoom：`architecture.md`《观察、交付与调度》；`reference/ui.md`《图像与坐标》；`testing/cases.md`《开始前准备》《操作与证据》。
- 借贴：`architecture.md`《观察、交付与调度》；`reference/ui.md`《显式 IME、文本 keys 与编辑组合键》；`testing/cases.md`《同次借用AX粘贴验收》。
- IC：`architecture.md`《组件与边界》《观察、交付与调度》；`reference/ui.md`《当前编辑器输入》；`reference/connection.md`《管理状态与边界》。
- 通知写：`architecture.md`《观察、交付与调度》；`reference/ui.md`《明确文本回复》及其前面的通知动作段。
- diff：`architecture.md`《状态寿命和资源所有权》；`reference/README.md`《观察正文与文件》《静态UI批次交付》；`reference/ui.md` 中多处（fill、当前编辑器、IME、只读等待、近期事件、静态 UI 批次）。
- 后台屏：`architecture.md`《状态寿命和资源所有权》《观察、交付与调度》；`reference/connection.md`《目标显示与工作区选择》；`reference/ui.md` 开头、《图像与坐标》《App 与系统操作》；`testing/cases.md`《开始前准备》。

---

## 8. 未验证范围

- **Android 编译与 JVM 测试：未验证，需构建环境。** 所有 Kotlin 改动只做了静态检查：被删文件中的类、对象、函数在 `src/main` 和 `src/test` 中残留引用为 0（`residue.sh`）；新引入但已无人引用的符号只发现 1 个（已修）；冲突处和静默合并处做了人工比对。不能排除还有未发现的静默合并，例如可见性、默认参数或 lateinit 自引用的写法问题。
- **契约 JSON**：各分支中是手工删样本和方法表条目，没有用 Gradle 导出重新生成。Python 侧 `test_wire_contract.py` 已通过，Kotlin 侧 `WireContractTest` 未运行。合入前必须按 AGENTS.md 的流程重新导出并双向验证。
- **设备行为**：IC 拆除后的 worker attach、detach、死亡重连；zoom 拆除后主屏 HARDWARE Bitmap 的截图与 marks；38 拆除后的 `ui.read` 无租约语义。以上都**需设备**。
- **R2-C 手工切分**没有实做，只做了 hunk 归类和行数估计。
- **rebase 冲突数**是下界（冲突方取被重放提交的版本，会把被删内容带回）。blame 口径给出的是受波及提交数，但不等于冲突数。
- 第 38 项 C 类改动（dispatchInput 等）是否值得补回，本报告没有评估它们的必要性。
