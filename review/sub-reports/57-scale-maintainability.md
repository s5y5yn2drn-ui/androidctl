# 专题 57：规模、可读性、职责划分、维护成本、测试质量、提交历史与工作方式

评审对象：产品仓库 `cab84b8`（本轮在第四轮候选 `4235af9` 之后加了 34 个提交），对照主分支基线 `7d5be70`；独立 docs 仓库 `31bae00`（本轮加 22 个提交）；实施方材料包（下文记作 `PKG`，阶段目录记作 `ST` = `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`）。

全程只读。我写的脚本都在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents8/57/` 下，用 `python3 -I` 运行，通过 `git show 版本:路径` 读源码，不碰仓库工作树：

- `subsys.py`：按子系统统计行数和拒绝码；
- `codes.py`：统计失败码；
- `contract.py`：统计契约样本；
- `ktfun.py`、`pyfun.py`：统计函数长度；
- `cli.py`、`cli2.py`：统计命令和选项；
- `runs.py`、`runs2.py`：统计 70 次运行里调用了哪些命令。

说法约定：
- **调用者**：按公开指引使用这个 CLI 的 LLM agent。
- **手写采集器**：worker 自己实现无障碍节点查询的 Binder 回调，用来代替公开的 `AccessibilityNodeInfo` 查询。
- **后台屏**：worker 创建的虚拟显示屏。
- **结果未知**：回执里的 `UNKNOWN`，退出码 3。
- **事实字段**：回执里记录"尝试了几次、返回了几次、排队了几个"这类过程计数的字段。

---

## 1. 结论摘要

**一句话：** 这一轮是项目开始以来第一次真正"做减法"。产品净减约 4,300 行，三个删除提交都干净。但它没有改变系统的形态。剩下的近 1.9 万行 Kotlin 和 1.1 万行主机代码仍然由三样东西主导：
- 无类型的 JSON 对象和按字符串分派的命令；
- 185 个失败原因码；
- 每个动作都要记账的过程事实字段。

过程上，产品改动和过程产物的比例严重失衡：34 个提交，对应 208 份过程文档（2.48 MB 文字）和 46 份内部 review。

最重要的发现：

1. **规模（重算）。** 三个版本依次为 `7d5be70`、`4235af9`、`cab84b8`：
   - Kotlin 主源码：9,360、20,278、18,881；
   - 主机源码：8,008、12,067、11,220；
   - 命令：47、66、62（叶子命令）；
   - 选项位：233、877、601；
   - 隐藏接口惰性成员：51、70、62；
   - 契约样本：175 KB、496 KB、438 KB。

   本轮整块删掉 8 个 Kotlin 文件（1,239 行）和 2 个主机文件（527 行）。相对基线，Kotlin 仍是 2.0 倍，主机是 1.4 倍。
2. **第四轮定的删除项基本删干净了，有一处没做，也有一些残留。**
   - 通知写入、编辑器输入通道、`--diff-from` 与专用保存格式、6 个 Settings 隐藏成员、全局字体表改写，这五项在产品代码里都没有残留【源码】。
   - **`--multiple-task` 没有删**（第 2 项明确要求删）。实施方把它和"搬屏归因 A/B"捆在一起等待用户选择，但删除它与归因无关（§2 D1）。
   - 残留有三类：约 10 个"已删选项必须被拒绝"的墓碑测试；通知读取结果里仍导出只为已删写操作服务的 PendingIntent 和 RemoteInput 描述；历史配额仍兼容旧 `.xml/.source` 文件。
3. **可读性：局部好，整体重。**
   - 函数短：Kotlin 1,020 个函数里只有 3 个超过 60 行。命名准确，少量注释基本在解释"为什么"。
   - 难点在全局：Kotlin 端没有类型化的结果模型，主源码里有 1,925 次 JSON 读写调用、75 处 `command == "..."` 分派。
   - 失败码：设备端 123 个不同的拒绝码，加上主机端，共 185 个。其中约 30 个只是"某个参数不合法"。
   - 每个动作链（指针、粘贴、剪贴板、清理）都带一套"尝试/返回/排队/跳过"的计数，用来支撑结果未知的语义。
   - 这套语义本身有价值，但它被摊在了每个子系统里，没有收拢成一处。
4. **测试多数在验证可观察行为，但比例偏向极端时序。**
   - 主机测试与主机源码之比是 2.25 倍，Kotlin 是 0.82 倍。抽查的 34 个测试里，约 26 个断言的是回执或输出；约 8 个锁内部计数或文案。
   - 多数测试名描述的是"响应提前到达""第二次注销失败"之类的边角时序。
   - 本轮修复提交里，测试增量常是源码增量的 4 到 9 倍。例如 `cfc16b0`：源码加 27 行，测试加 237 行。
   - 契约样本 438 KB 确实是双向对拍，Python 一侧我实跑通过。但 44% 的体量是 12 个完整采样（192 KB），重生成流程只有 Windows PowerShell 版。
5. **当前头 `cab84b8` 没有一次完整的质量检查。**
   - 最后一次完整 `task quality` 跑在 `0832f88`：主机 1,878 项通过；Android 因一条 lint 错误让整体退出码为 201【原件 `ST/quality-integrated.log`】。
   - 之后的 `93fd40d` 只跑了受影响的 lint 和发行单元测试；再往后 4 个文档提交（其中一个改了 `ui_commands.py` 的帮助文字）没有跑过完整检查。
   - 实施方的说法如实，没有夸大。
6. **"钻牛角尖"的主要去向**，是第 6 项的能力验收和成本门的取证，而不是代码。
   - 70 次模型运行里，正常任务从不使用 batch、多指、拖拽、剪贴板、通知；这些能力只出现在专为验收设计的任务里【重算 `runs2.py`】。
   - 为找"非它不可"的题目和十题图像集，实施方做了大量准备、方法 review 和环境恢复。
   - 第 22 项要求去掉"对取证文档的反复复核"，实施方只做到一部分：本轮仍有约 13 份 review 审的是方法稿、运行记录和题面，不是源码。
   - 源头一半在第四轮评审方：第四轮评审 §7 提出的固定任务、5 次以上、同快照交替，以及第 6、7 项的门槛表。

**对两道待决题（从维护角度）：**
- 坐标题（候选按截图坐标点击时，要求接收触摸的窗口全属于同一 UID，桌面因壁纸窗口被拒）：A 到 D 四个选项都保留 WindowInfos 采样、编译期桩和清理责任，维护成本差别很小。按维护成本选不出谁更好，应按能力取舍来选。
- 归因题本身代码量小。不应让它阻塞 `--multiple-task` 的删除。

---

## 2. 确定缺陷

本专题偏结构，没有新的产品功能缺陷。下列是维护层面的确定问题。

| # | 级别 | 问题 | 位置与证据 | 影响 | 最小修正 | 确定度 |
| --- | --- | --- | --- | --- | --- | --- |
| D1 | 必须修（范围） | 第 2 项要求删除的 `--multiple-task` 仍在 | `ActivityLaunch.kt:185-186, 270-277, 318-334, 399-407`；`ActivityIntent.kt:57`；`open --help` 实跑仍列出该选项；`skills/androidctl/actions.md:379-380`、`displays.md:83`；`ActivityLaunchTest.kt:346-382`；`WireContractTest.kt:1115-1188`。`ST/plan.md`"未决"段把"显式搬来参数、删除 multiple-task、整表推进"捆成一个"完整行为"等待 A/B | 已决定删除的能力继续占用代码、测试、契约和指引；拒绝文案里还在引导调用者"use --multiple-task" | 立即单独删除（估算源码约 50 行、测试约 60 行、指引 6 行、契约 1 组样本）。删除它不依赖归因 A/B：归因只决定"意外搬动由哪条命令报告"，与"另开任务"这个旗标无关 | 源码与实跑，确定 |
| D2 | 必须修（流程） | 当前头没有一次干净的完整质量检查 | `ST/quality-integrated.log`：头 `0832f88`，`QUALITY_EXIT_CODE=201`，原因是 `ImageMarks.kt:72` 的 `UseKtx` lint。`ST/quality-marks-ktx.log`：同一个头加未提交的修正，只跑了 `ImageMarksTest`、`lintCheck`、`testReleaseUnitTest`。`93fd40d..cab84b8` 又改了 `ui_commands.py` 和两份指引，只跑过 ruff 和帮助输出 | "当前树全部通过"的说法目前不成立（实施方也没这样说） | 在 `cab84b8` 或整理后的头上跑一次完整 `task quality` 和 `task ci` | 原件，确定 |
| D3 | 可选 | 有提交弄坏了测试，靠后续提交补上 | `f8f69a1`、`9985eca`、`b782587`、`f38686f`、`0832f88` 五个测试专用提交，修的是 `31a5786`、`566a950`、`ab13a08` 等提交引入的测试失败。`ST/quality-current.log` 在 `c804a59` 上记录主机 5 项失败、Android 2 项失败 | 每项改动都做了独立源码 review，但没有随改随跑全部测试，失败到综合检查才暴露。最终会整理历史，所以不影响成品，但说明"每项 review"并不能替代"每项跑测试" | 每个行为提交之后跑一次该语言的全部单元测试（主机约 2.5 分钟） | 原件，确定 |
| D4 | 可选 | 契约重生成流程只有 Windows PowerShell 版本（第四轮 M3，未修） | `AGENTS.md`"Testing Guidelines"一节 | 本轮契约文件改了 +494 / −1,803 行，非 Windows 维护者需要自己翻译命令 | 补 POSIX 写法，或做成一个 Gradle 任务 | 源码，确定 |
| D5 | 可选 | 通知读取仍导出只为已删写操作服务的字段 | `NotificationOperations.kt:177-236`：action 的 `pendingIntent`（creatorUid、type、immutable）、`authenticationRequired`、`allowGeneratedReplies`；RemoteInput 的 `resultKey`、`freeForm`、`editChoicesBeforeSending`、`allowedDataTypes`。`skills/androidctl/notifications.md:11-15, 29-30` 仍描述它们 | 调用者看到按钮和回复框的结构，却没有任何命令能用；每次读取的 192 KiB 预算还要为这些字段做省略记账 | 只保留 action 的 index 和 title。估算源码约 60 行，契约和指引相应缩小 | 源码，确定 |

---

## 3. 设计与简化

### 3.1 三个版本的子系统分布（重算 `subsys.py`）

Kotlin 主源码含 C 和 AIDL，按文件归类。基线的按键和文字输入写在 `UiActions.kt` 里，所以基线"文字输入"一栏是 0，"动作"一栏偏大。

| 子系统（代表文件） | 7d5be70 | 4235af9 | cab84b8 | 本轮 | 累计 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 会话、目标与显示（Sessions、TargetManagement、WorkDisplay、UiBackend、AutomationCleanup、DeviceWake） | 703 | 1,760 | 1,675 | −85 | +972 |
| 启动与任务（ActivityLaunch、ActivityIntent、ActivityTasks） | 0 | 762 | 764 | +2 | +764 |
| 采集与节点（TreeCollector、Collection*、NodeFacts、NodeEncoding、UiSemantics、UiSnapshot、UiTreeBudget） | 1,451 | 2,325 | 2,311 | −14 | +860 |
| 引用、选择与等待（UiRefs、UiRefRepair、UiCondition、UiSelector、UiSelection、UiRecentEvents、UiSignals、UiScrollUntil） | 573 | 1,626 | 1,667 | +41 | +1,094 |
| 动作与观察发布（UiActions、NodeActions、FillProgress、Scroll*、UiPostAction、UiPublication、UiDelivery、ObservedChange、UiRun、ActionReply） | 2,084 | 4,018 | 4,085 | +67 | +2,001 |
| 文字输入、按键与剪贴板（BorrowedPaste、Clipboard*、Key*、AppInput、原 Editor*） | 0 | 1,878 | 1,476 | **−402** | +1,476 |
| 指针、多指与坐标派发（PointerGesture、PointerPlan、VisualInput、WindowInput*） | 359 | 1,285 | 1,285 | 0 | +926 |
| 图像（ImageMarks、ImageRegion、ImageCoordinates） | 0 | 438 | 462 | +24 | +462 |
| 通知 | 0 | 1,586 | 627 | **−959** | +627 |
| 连接、配对与 worker（Gateway*、NetworkGateway、Pairings、WorkerConnection、WorkerService、Management*、Wire） | 2,802 | 3,096 | 3,070 | −26 | +268 |
| 隐藏接口表（HiddenApi） | 172 | 288 | 238 | −50 | +66 |
| 文件与 shell 作业（含 C） | 2,266 | 2,274 | 2,271 | −3 | +5 |
| **合计** | 10,410 | 21,336 | 19,931 | −1,405 | +9,521 |

| 主机子系统（文件） | 7d5be70 | 4235af9 | cab84b8 | 本轮 | 累计 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 命令声明（`ui_commands`、`app`、`cli_options`） | 1,656 | 3,168 | 2,696 | −472 | +1,040 |
| 呈现与交付（`ui_view`、`ui_client`、`output`、`node_facts`） | 1,301 | 2,572 | 2,682 | +110 | +1,381 |
| batch（`ui_batch`） | 0 | 317 | 334 | +17 | +334 |
| 保存、比较与历史（`ui_diff`、`view_file`、`history`、`workspace`） | 204 | 841 | 314 | **−527** | +110 |
| 参数解析（`activity_intents`、`input_keys`、`pointer_plan`） | 0 | 287 | 287 | 0 | +287 |
| 连接与守护进程（`coordinator`、`transport`、`local_*`、`state`、`identity`、`private_files` 等） | 3,017 | 3,033 | 3,058 | +25 | +41 |
| 文件与 shell 作业（`file_jobs`、`job_commands`、`user_files`） | 1,830 | 1,849 | 1,849 | 0 | +19 |
| **合计** | 8,008 | 12,067 | 11,220 | −847 | +3,212 |

**本轮删了什么（实际行数）【源码 `git show --numstat`】：**
- 通知写入（`5b9876f`）：Kotlin −1,034，Kotlin 测试 −1,265，主机 −115，主机测试 −467，契约 −1,452，指引 −141。整块删掉 `NotificationActions.kt`（589 行）、`NotificationSelection.kt`（193）、`NotificationReplyInput.kt`（52）。
- 编辑器通道（`21e7643`）：Kotlin −570 / +63。删掉 `EditorBridge`、`EditorSurrounding`、`CurrentEditorInput`、`IEditorBridge.aidl`，共 346 行。
- 阅读与保存（`5a817f1`）：主机 −690 / +139，主机测试 −734 / +196。删掉 `ui_diff.py`（266）和 `view_file.py`（261）。
- 动作输出选项（`0444e34`）：主机 −385 / +56。
- Settings 绕路（`b7c9a1c`）：删掉 `WorkDisplaySettings.kt`（59 行），合计 −93。
- 字体（`82318e9`）：Kotlin +61 / −43，改成本地点阵字形。

**本轮加了什么：**
- 新文件只有 `ActionReply.kt`（64 行，回调回包畸形时快速报错）。
- 增长主要在修复提交，而且以测试为主：`56dbd7b` 等待多匹配（源码 +136、测试 +433）、`897a300` 清理状态（+118 / +250）、`566a950` 回调诊断（+97 / +294）、`8731032` 本地失败后保留执行事实（+78 / +216）、`3b7fb0e` 工作区权限（+57 / +203）、`cfc16b0` 引用复用（+27 / +237）。

**隐藏接口（源码）：**
- `HiddenApi.kt` 惰性成员 51、70、62。本轮删了 8 个：Settings 链 6 个（`activityManager`、`activityManagerInterface`、`getContentProviderExternal`、`removeContentProviderExternalAsUser`、`contentProvider`、`callContentProvider`），加通知写入 2 个（`cancelNotifications`、`sendNotificationIntent`）。没有新增。
- `HiddenApi` 之外仍有反射或手写 Binder 的文件，从 10 个降到 8 个：
  - `ImageMarks` 的 4 处字体反射已删；
  - `ClipboardAccess.kt:58-206` 自带 9 处反射（IClipboard、IAppOpsService、`OP_READ_CLIPBOARD`），没有登记到 `HiddenApi`；
  - 3 个手写 `onTransact`：`AccessibilityConnection.kt:153`、`CollectionNodes.kt:111`、`NotificationAccess.kt:63`；
  - 2 个编译期平台桩加 1 条 R8 规则（`android/platform-stubs/`、`android/app/window-input.pro`）不变。
- **建议**：把 `ClipboardAccess` 的反射移进 `HiddenApi`，让它成为唯一的隐藏接口清单。机械工作，约 30 行。

### 3.2 第四轮删除项的残留核查（`git grep cab84b8`）

| 删除项 | 产品代码 | wire / 契约 | 测试 | 指引 / docs | 结论 |
| --- | --- | --- | --- | --- | --- |
| 通知写入 | 无写入路由；`NotificationMethods.commands` 只剩 list/get（`NotificationOperations.kt:9-10`） | 无 `notificationActionResults` 段 | `test_notifications.py:235` 墓碑测试（open/action/reply/dismiss 必须被拒） | 仍描述 PendingIntent 和 RemoteInput（D5） | 删干净；有字段残留 |
| 编辑器通道 | 无；`BrowserAccessibilityService.kt` 退回 10 行空服务，XML 无输入法声明 | 无 | `test_text_input.py:75-95,171`、`BorrowedPasteTest.kt:269-272` 墓碑测试 | 无 | 干净 |
| `--diff-from` 与专用格式 | 无 | 无 | `test_ui_arguments.py:39`、`test_image_delivery.py:124`、`test_ui_batch.py:475,1179,1268-1283,1459`（检查旧固定基线文件名不被读写） | docs 和指引无 | 干净；墓碑测试 6 处 |
| 6 个 Settings 隐藏成员 | 无（`force_resizable` 0 处） | 无 | 无 | `architecture.md:89` 用一句"不再取得 Settings provider"说明 | 干净 |
| 全局字体表改写 | 无（`Typeface`、`SystemFonts`、`FontManager` 0 处） | 无 | 无 | 无 | 干净 |
| `--multiple-task`（第 2 项） | **仍在** | 仍在 | 仍在 | 仍在 | **未删**（D1） |

**墓碑测试**（断言"已删选项被拒绝"或"旧文件不被读写"）约 10 个，引用约 31 处。对 alpha 产品，"已删选项被拒绝"由 Typer 的未知选项机制保证，无需逐个锁定。建议删除，约 150 行测试。
- 历史配额仍兼容旧 `.xml/.source` 文件（`history.py:32`；`test_history.py:736` "legacy png"）。0.4.0 alpha 没有向后兼容义务，可以删掉。

### 3.3 可读性与职责：逐文件评价

函数统计用 `ktfun.py`、`pyfun.py`。Kotlin 主源码 1,020 个函数：≥40 行的 46 个，≥60 行的 3 个，≥100 行的 1 个（`NodeFacts.read`，112 行，一个大的字段映射）。行宽上限 140，平均 41 字符。注释 128 行，主机 102 行，大多数在解释原因，例如：
- `UiRun.kt:113`："Keep the execution fact before fallible observation"；
- `HiddenApi.kt:194`："CaptureArgs remains nested … across Android 14–17"。

| 文件 | 行 | 函数 | 类级 var | 拒绝码 | 评价 |
| --- | ---: | ---: | ---: | ---: | --- |
| `UiActions.kt` | 1,071 | 51 | 16 | 11 | 所有节点动作（fill、check、range、row 选择、paste、key 前置）放在一个类里，并加了 `@Suppress("LargeClass")`。函数不长，但 JSON 读写 156 次，大量 `optString`/`put` 字符串键。**应拆**：fill 与核值（约 300 行）、check/row 选择（约 200 行）、通用准入（约 200 行）。纯机械拆分，能力不变 |
| `UiBackend.kt` | 665 | 39 | 17 | 8 | 连接、请求路由、采集循环、wake、阶段化清理都在这里（`:45-55`、`:567-660`）。`execute` 的 catch 分支按命令种类分三种失败形状（`:126-140`），说明结果模型没有统一。`perform` 和 `routeObservedOperation` 是字符串 `when` 分派（`:219-290`）。**应拆**：清理（约 120 行）移到 `AutomationCleanup`，路由改为命令表 |
| `UiCondition.kt` | 479 | 23 | 7 | 4 | wait 条件求值。`evaluate` 57 行、6 层缩进。复杂度来自 contains/selector/gone/idle/screen-change/any-of 六类条件加 partial 规则，是**必要复杂度** |
| `UiDelivery.kt` | 466 | 17 | 12 | 5 | `publish` 80 行（`:195-275`），在 192 KiB 回包预算里逐个"接纳"树、窗口目录、图片，`synchronized` 11 处。复杂度的根源是"一次回复装下全部附件"的预算设计（见 §3.5 差距 3） |
| `VisualInput.kt` | 458 | 21 | 6 | 8 | 坐标输入、图像帧有效性、区域截图入口。与 `WindowInputListener` 一起承担坐标派发的 UID 守卫。这部分复杂度是坐标落点保护的必要代价；坐标题的四个选项都不会明显减少它 |
| `ActivityLaunch.kt` | 407 | 22 | 0 | 8 | 无可变状态，函数纯净，是全仓最好读的大文件之一。`--multiple-task`（D1）和启动前跨屏冲突检查占约 90 行 |
| `TreeCollector.kt` 加 `Collection*` | 373 + 365 | 16 + | 8 | 2 | 手写采集器。`addNode`、`addChildren` 各 45 行，回调回包解析（`CollectionReplies.kt`）处理尾字节和未知事务。维护负担是手写 Parcel 布局，第四轮已论证。去留取决于第 7 项实验，本专题不重复 |
| `BorrowedPaste.kt` | 373 | 22 | 2 | 3 | 借用粘贴的完整状态机：读原剪贴板、空值准入、借写、广告核验、AX PASTE、观察、比对后恢复。JSON 写入 45 处，事实字段很多（`paste.state`、`restore.attempted` 等）。保护用户剪贴板是真实需要；但每个阶段都输出给调用者的事实字段，调用者几乎用不上。**可选**：对外只留 `paste`、`verification`、`restore` 三个状态，内部计数只进诊断 |
| `Sessions.kt` | 365 | 13 | 3 | 9 | 锁内状态清楚：`current`、`released`、`expired`、`revoked`、`resources`（`:65-72`）。到期诊断（`expired` 集合）是第 12 项要求的，必要 |
| `PointerGesture.kt` | 352 | 16 | 16 | 1 | 6 个计数器（`attempted`、`returned`、`successful`、`synchronous`、`queued`、`skipped`，`:104-110`）加 `possible` 指针表，每帧输出 `PointerFrame.json()`。计数是为了在部分发送后发对 CANCEL，有必要；但对外输出每帧事实没有必要。只有多指时间线和捏合用它 |
| `UiRecentEvents.kt` 加 `UiSignals.kt` | 320 + 169 | 12 + | 9 | 2 | 事件唤醒 wait 是性能上必要的。"近期事件"作为对外字段输出（`UiBackend.kt:300-312`），我在 70 次运行的模型轨迹里没有看到调用者据此决策（未逐条核）。**可选删**：只保留唤醒，约 −200 行 |
| `NotificationProjection.kt` | 313 | 21 | 1 | 0 | 只读通知的投影。大半是按 UTF-16 长度和 from/count 记录省略的预算逻辑（`:250-260`），对一个 0 次使用的只读能力偏重 |

| 主机文件 | 行 | 评价 |
| --- | ---: | --- |
| `ui_commands.py` | 1,583 | 一个 1,213 行的 `register()`（`:371`）里嵌套约 45 个命令定义。逻辑不复杂，是声明式的长。本轮选项位从 877 降到 601（第 9 项落实：`click` 只剩选择器、`--touch`、`--timeout`、`--settle`、`--screenshot`、`--no-print` 和 4 个全局选项）。**应拆**：按命令组拆成 4–5 个模块，机械工作 |
| `ui_client.py` | 1,497 | RPC 客户端、`result_line`（232 行，`:165`，逐命令拼回执首行）、`prepare_ui`（188 行）、交付与保存混在一起。第四轮已建议拆成 client、receipt、delivery 三个模块，本轮没动 |
| `ui_view.py` | 975 | 紧凑阅读呈现，本轮 +130。`reading_view` 138 行、`attributes` 123 行、`screen` 111 行。第 8 项的"一行一节点"是本轮最有价值的变更之一，这里的复杂度大体必要 |
| `app.py` | 922 | 结果输出、退出码、状态渲染。`emit` 路径清楚 |
| `ui_batch.py` | 334 | `compile_plan` 107 行、6 层嵌套（`:65`），要在不执行回调的前提下解析 Typer 语法。测试 1,658 行，是源码的 5 倍 |
| `coordinator.py` / `transport.py` | 797 / 541 | 基线就有，本轮未动；`_open` 137 行、`request` 105 行 |

**失败原因码（重算 `codes.py`）。**

| | 7d5be70 | 4235af9 | cab84b8 |
| --- | ---: | ---: | ---: |
| 设备端 `Rejected("x")` 不同值 | 95 | 129 | 123 |
| 设备端其它 reason/code 字面量（去重后并集） | 99 | 142 | 136 |
| 主机端 `Fault("x")` | 60 | 61 | 62 |
| 两端去重并集 | 144 | 192 | **185** |

设备端按子系统分布（cab84b8；括号内是只在该子系统出现的码）：
- 动作与观察发布 35（18）
- 连接与 worker 27（20）
- 会话与显示 25（13）
- 文件与 shell 19（17）
- 采集 14（3）
- 引用与等待 13（2）
- 指针 12（5）
- 文字输入与剪贴板 11（4）
- 启动 9（7）
- 图像 4（1）
- 通知 3（1）

另有 32 个码跨子系统共用。完整清单在 `57/codes-cab.txt`。

可以合并的：
- 约 30 个码是"某个参数不合法"：`action`、`actionLabel`、`actionName`、`depth`、`direction`、`duration`、`imageFormat`、`imagePath`、`key`、`message`、`offset`、`path`、`point`、`query`、`range`、`region`、`scope`、`script`、`selector`、`sessionName`、`size`、`url`、`deviceName`、`pairingName` 等。可以统一成 `arguments` 加一个 `field` 字段。
- 约 10 个是"状态已变化"类：`staleRef`、`staleImage`、`staleWorker`、`sourceChanged`、`containerChanged`、`selectionChanged`、`identityChanged`、`sessionChanged`、`connectionChanged`、`temporaryChanged`，可以合成 `stale`、`changed` 两类。

合并后约 120 个，对调用者有决策意义的不超过 40 个。

**哪些复杂度是必要的（不建议删）：**
- 会话 lease 与资源占用；
- 引用身份核对和"至多一次同请求修复"（`UiRefRepair.kt:13`）；
- wait 条件与 partial 规则；
- 坐标派发的 UID 守卫（防止误点别的 App）；
- 借用粘贴恢复前比对（防止覆盖用户新复制的内容）；
- 文件传输"未发送"的四条证明；
- 紧凑阅读呈现。

这些是"不误操作"和"行为可预期"的核心。

**只为极端情形存在的分支（可议）：**
- 通知监听器"每命令一次注册修复"（`NotificationOperations.kt:47-57`）；
- 回调尾字节容忍（`56a43d7`）；
- 清理"排队与失败区分"（`897a300`，+467 行）；
- 本地失败后保留执行事实（`8731032`）；
- 提前到达的响应仍保留清理（`BorrowedPasteTest` 中多例）；
- 字节预算内的逐字段省略记账（通知、剪贴板、回执）。

单看每一条都有理由。合起来是一个模式：**每条路径都要回答"这次失败时到底发生到了哪一步"**。这是结果未知契约的直接代价（见 §3.5 差距 1）。

### 3.4 "应删除、合并、重构"清单

| # | 项 | 估算 | 能力得失 |
| --- | --- | --- | --- |
| 1 | 删 `--multiple-task`（D1） | 源码 −50，测试 −60，指引 −6 | 无损失（已决定删除） |
| 2 | 通知只读字段收缩（D5） | 源码 −60，契约约 −2 KB，指引 −5 | 失去对按钮和回复框结构的描述，反正已无命令可用 |
| 3 | 删墓碑测试和旧文件兼容 | 测试 −150，`history.py` 约 −15 | 无 |
| 4 | 参数类失败码统一为 `arguments` 加 `field` | 设备约 30 处、主机约 15 处改写，指引 reference 表缩短 | 调用者按 `field` 分辨，信息不丢 |
| 5 | `ClipboardAccess` 反射移入 `HiddenApi` | 约 ±30 | 无；隐藏接口清单变成唯一 |
| 6 | 拆 `UiActions`、`UiBackend`、`ui_commands.register`、`ui_client` | 净 0，文件变多 | 无；单文件降到 400 行以内 |
| 7 | "近期事件"不作对外字段，只保留唤醒 | 源码约 −200，测试约 −200，契约 −数 KB | 失去回执里的事件摘要；证据弱，可选 |
| 8 | 对外事实字段收拢：粘贴、指针、剪贴板只报"状态加原因"，过程计数进 `--raw` 的诊断子对象 | 源码约 −150，测试改写约 −600 | 调用者看到的回执变短；结果未知语义不变 |
| 9 | 契约样本瘦身：12 个完整采样换成 3–4 个代表样本，`output` 段（45 KB，基线至今未变）换成生成式 | 438 KB → 约 200 KB | 仍是双向对拍，覆盖略减 |
| 10 | （不建议）去掉 WindowInfos 前检、回到基线普通注入（见 §5） | 约 −300 源码、−180 测试、2 个桩加 1 条 R8 规则 | 失去坐标落点保护，与已定事项冲突 |
| 11 | 若第 6 项降级为实验：timeline/pinch/drag、结构化剪贴板、batch 冻结，不再写验收 | 代码不删，验收投入归零 | 见 §6 A1 |

1–6 是实施细节，实施方可以直接做；7–11 涉及对外行为，交给用户。

### 3.5 如果重新设计

目标：给 LLM agent 用的 Android 自动化 CLI，单人维护。

**干净的整体结构（建议）：**

```
CLI（命令声明、呈现）──本地守护进程（连接复用、工作区文件）──TLS──设备 worker
设备 worker：
  会话（lease、资源、到期）
  采集（一条路线）→ 节点模型（类型化）→ 引用表
  动作执行器：命令表 {名字 → (校验, 准入, 执行, 结果)}
  结果：Outcome{done|failed|not_sent|unknown} + code（约 40 个）+ 少量领域数据
  附件：观察正文和图片写进 worker 的会话目录，主机按 id 拉取
```

- **结果语义**：四种结果加一个原因码，再加"已执行前缀"一个字段。不对外暴露过程计数。
- **类型**：Kotlin 端有类型化的命令参数和结果（数据类加序列化），由一份 schema 同时生成 Python 模型和契约样本。契约样本只用来防回归，不承担"规范"职责。
- **能力**：snapshot、find、click/fill/scroll、wait、back/home、open、screenshot（含 region）、tap/swipe、shell、文件，约 30 个命令。高级能力（batch、多指、剪贴板对象、通知、后台屏）单独挂在 `experimental` 下，有需要时再加验收。
- **文档**：代码注释写为什么，一页架构说明写组件和状态寿命，公开指引写怎么用。不另设 28 万字节的私有参考。

**与现状差距最大的三处：**

1. **结果语义被摊到每条路径。**
   - 现状：185 个原因码；每个子系统都有自己的过程计数和事实字段；`UiBackend.execute` 的 catch 分支按命令种类产生三种失败形状（`UiBackend.kt:126-140`）。
   - 本轮新增的修复提交大多就是在补这些计数的边角（`566a950`、`897a300`、`8731032`）。
   - 这是维护负担的第一来源。要收拢，需要一个统一的 `Outcome` 构造器和"前缀"概念，所有路径只报这两样。
2. **无类型的 JSON 与字符串分派。**
   - Kotlin 主源码 1,925 次 JSON 读写调用（基线 868），`run.command ==` 等分派 75 处，数据类只有 68 个。
   - 两端靠 438 KB 契约样本加 3,169 行契约测试（`WireContractTest.kt` 1,870 行、`test_wire_contract.py` 1,299 行）对齐。
   - 每改一个字段，要动 Kotlin、Python、契约三处，再在 Windows 上重生成。
3. **"一次回复装下全部附件"的预算设计。**
   - 192 KiB 回包预算，逐个接纳树、目录、图片（`UiDelivery.publish`），主机侧还要预留（`UiBudget`、`ui_client.attachments`、`finish_delivery_result`）。
   - 字节预算的省略记账因此蔓延到通知、剪贴板、回执、阅读呈现。
   - 如果附件改为 worker 落盘、主机按 id 拉取，预算只管正文，这一层大部分可以消失。

**渐进还是重写：** 渐进可达，不需要重写。
- 第 3 处可以先在附件路径上做（区域截图已经是"新采、单独交付"的形态）；
- 第 1 处可以按子系统逐个收拢，先从对外字段做起，内部计数保留；
- 第 2 处工作量最大，可以从"新命令用类型化结果"开始，老命令按需迁移。

三项加起来是数周的专注工作，但每一步都可以独立合入，回归保护由现有的行为测试承担。真正需要重写的只有手写采集器（取决于第 7 项）。

---

## 4. 证据审计

### 4.1 规模数字

主评审给的背景数字我全部复核过：
- Kotlin 9,360 / 20,278 / 18,881（只算 .kt）；
- 主机 8,008 / 12,067 / 11,220；
- Kotlin 测试 4,622 / 15,683 / 15,495（含测试目录里的 Java 桩，比主评审多约 10 行）；
- 主机测试 16,775 / 25,573 / 25,278；
- 指引 1,253 / 2,662 / 2,419 行；
- 契约 175,142 / 496,272 / 438,141 字节；
- 选项位 233 / 877 / 601。

一个差异：叶子命令我数到 47 / 66 / 62，主评审是 63。差一个，可能是计没计 `help` 一类的组回调。【重算 `cli.py`，三个 venv 实跑】

### 4.2 质量检查记录（逐份读原件）

| 原件 | 头 | 实际内容 | 实施方说法 | 判定 |
| --- | --- | --- | --- | --- |
| `ST/quality-current.log` | `c804a59` | 主机 5 失败、1,873 通过、5 跳过；Android "526 tests completed, 2 failed"（第 199 行）；`QUALITY_EXIT_CODE=201` | `ST/README.md:45` 写的是主机 1873/5/5、Android 524 通过 2 失败 | 成立 |
| `ST/quality-publication-fix.log` | `ffc77c3` 加修改 | 两个测试类 20 项通过；首次日志管道写错，第一次 release 增量编译失败，重跑通过 | 如实记录了命令传参错误 | 成立 |
| `ST/quality-host-fix.log` | `f38686f` | 5 个原失败测试通过；ruff、mypy、diff 检查通过 | 成立 | 成立 |
| `ST/quality-integrated.log` | `0832f88` | 主机"1878 passed, 5 skipped"；Android `testDebugUnitTest` 任务执行，无失败输出；`lintDebug FAILED`（`ImageMarks.kt:72 UseKtx`）；`QUALITY_EXIT_CODE=201` | "host 1878 passed、debug 526 通过，退出 201 仅剩 UseKtx lint" | 主机数和退出原因成立。"debug 526"在日志里看不到计数（Gradle 不打印通过数）；526 与我数的 `@Test` 个数一致，可能来自测试报告 XML，**部分核实** |
| `ST/quality-marks-ktx.log` | `0832f88` 加未提交修正 | 第一次调用失败，修正后 `ImageMarksTest` 加 `lintCheck` 通过；`testReleaseUnitTest` 通过 | "字体 5、release 完整 526 及 Android lint 通过" | 成立；这不是完整 `task quality` |
| 当前头 `cab84b8` | — | `93fd40d..cab84b8` 改了 `ui_commands.py` 帮助文字、`SKILL.md`、`actions.md`；只有 `key-help-check.log`（帮助输出加 ruff） | 实施方没有声称完整检查通过 | **当前头没有干净的完整检查**（D2） |

我在 `cab84b8` 上实跑了 `test_wire_contract.py` 和 `test_ui_arguments.py`：395 通过、3 跳过【实跑，`S/r5/venv5`】。

### 4.3 测试质量抽查

**Kotlin，19 个**：
- `SessionsTest` 3 个；
- `UiCleanupTest` 2 个；
- `BorrowedPasteTest` 4 个（含 `unchangedTextCannotProvePaste…`、`explicitRestoreFailure…`、`responseDuringOriginalRead…`、`earlyResponseOrLeaseRevocation…`）；
- `PointerGestureTest`、`UiDeliveryTest`、`UiBudgetTest`、`ClipboardTest` 只看测试名，加读 2 个正文。

**主机，17 个**：
- `test_observation_files` 2 个；
- `test_result_output` 2 个；
- `test_ui_arguments` 2 个；
- `test_ui_batch` 2 个；
- `test_image_delivery` 1 个；
- `test_text_input` 1 个；
- `test_history`、`test_workspace`、`test_ui_view` 只看测试名。

**结论：**
- **多数在验证可观察行为。** Kotlin 测试注入时钟和假平台，断言的是回执的 `result`、`data.*.state`、发送次数。例如 `BorrowedPasteTest.kt:170-202` 断言"字段不变时报结果未知""恢复失败时整体 failed 但 paste 为 done"，这正是第 5 项要求的行为。主机测试断言的是打印文本和 JSON，对一个给 LLM 用的 CLI，输出文本就是接口，锁文本是合理的（例如 `test_result_output.py:343-360` 的 bidi 控制字符转义）。
- **耦合内部状态或文案的约占五分之一**：
  - `UiCleanupTest.kt:15-35` 数 `disconnects`、`callbacks` 的调用次数；
  - `SessionsTest.kt:50` 断言消息包含 "No current session"；
  - `PointerGestureTest` 断言每帧计数；
  - 主机的"old fixed baseline name"类墓碑测试。
- **分布偏向极端时序。** 抽查文件的测试名里，约三分之二描述的是"响应提前""第二次失败""截止恰在某一步之后"这类边角。主路径（例如"粘贴成功并恢复"）通常各只有 1 个测试。这和 §3.3 的"每条路径都要回答发生到了哪一步"是同一个来源。
- **比例：** 主机测试是源码的 2.25 倍，Kotlin 是 0.82 倍。
  - 主机测试是资产。它覆盖的是调用者真实看到的输出，删能力时也删得干净（本轮 `5a817f1` 删了 734 行测试）。
  - 负担集中在两处：`test_ui_batch.py`（1,658 行，对 334 行源码）和 `test_observation_files.py`（1,830 行）。修复提交里测试增量是源码的 4–9 倍，每个边角修复都配一整套时序测试。
  - 建议：修边角缺陷时只加一个回归测试，不为同一问题枚举所有时序组合。
- **平台集成层仍然没有 JVM 测试**：`UiBackend`、`TreeCollector`、`WorkDisplay`、`NotificationAccess`、`WindowInputListener` 这些最脆弱的部分只靠模拟器（第四轮已指出，未变）。

**契约样本 438 KB 值不值：**
- 值，但体量可以减半。它是 Kotlin 生产代码生成、Python 生产代码消费的真对拍，曾经抓到过两端漂移（第四轮记录）。
- 体量分布【重算 `contract.py`】：`captures` 192 KB（12 个完整采样，44%）、`actionResults` 52 KB、`output` 45 KB（基线至今完全未变）、`pasteResults` 12 KB。
- 12 个完整采样大部分是同一棵树的变体，3–4 个就够覆盖编码分支。
- 本轮契约文件改了 +494 / −1,803 行，5 个提交触及它。每次都要在 Windows 上重生成、人工审 diff（D4）。

### 4.4 提交与 docs

- **产品 34 个提交**：
  - 都是 Conventional Commit，有 scope，题目准确。粒度合理：三个大删除各一个提交，修复各一个提交。
  - 问题有两个：5 个"测试修正"提交说明行为提交没有随手跑测试（D3）；4 个只改一两行指引的提交（`a6859c1`、`a873616`、`7693794`、`cab84b8`）可以合并。
  - 最终会整理，所以这些不影响成品。
- **九组划分**（`ST/plan.md`"最终审阅序列的准备边界"）：
  - 按责任划分合理，与 §3.1 的子系统大体一一对应。
  - 第 9 组"共享契约 fixture、全局验证、打包及指引索引"有变成兜底大提交的风险。实施方自己写了"不能把所有业务 wire 留给一个巨大的最后提交"，但 438 KB 的单个 JSON 文件要按组拆进 8 个提交，操作上很繁琐。
  - 从基线累计看是 +45,253 / −5,465 行，平均每组约 5,000 行。这样的提交不是为了逐个审阅，而是为了让历史可读。
  - **"最后再整理"的安排是对的**：现在整理，待决题一定就要重做。但第 21 项写的"之后又打磨多轮，合入前再整理一次"应改成"只在合入前整理一次"，中间轮次不整理。
- **docs 仓库（`31bae00`）**：
  - 279,557 字节，9 个文件：`reference/ui.md` 90 KB、`testing/cases.md` 63 KB、`reference/connection.md` 36 KB、`architecture.md` 31 KB。比第四轮的 292 KB 小 4%。
  - 本轮 22 个提交几乎一一对应产品提交。第 23 项写的是"按行为改动的批次更新，不必逐个提交配对"，这是允许而非禁止，但说明实施方仍在逐个配对。
  - **是否只写当前设计**：大体是，但"旧"字仍出现 127 次、"不再"21 次，例如 `reference/connection.md:165`"主机不再读写 display.json，已有旧文件保…"、`architecture.md:89`"不再取得 Settings provider"。这些是变更说明，第 23 项要求放进提交信息。
  - **抽查与产品一致**：`--multiple-task`（`reference/ui.md:280` 与源码一致，都还在）、30 分钟空闲（`Sessions.kt:358 DEFAULT_IDLE_MS = 1800000L` 与 `reference/connection.md:149-151` 一致）、Settings 删除（`architecture.md:89` 与 `b7c9a1c` 一致）、通知只读（`reference/ui.md:294` 与 `NotificationOperations.kt` 一致）。四处全部一致。
  - **可读性差**：段落是不断句的长串术语，例如 `reference/ui.md:161`"确知整命令零派发时跳过动作后观察，保留已有缓存及动作前采样事实，不伪报已观察无变化。输入目标/自身准入拒绝、首次明确 UID mismatch 及首次本地未调用不消耗当前图……"。没读过源码的人读不懂，读过源码的人又不需要它。代码注释只有 230 行，"为什么"几乎都在这里，而这里又难读。

### 4.5 工作方式与证据成本（重算）

| 量 | 数 |
| --- | ---: |
| `ST/` 下 Markdown 文档 | 208 份，2,481,820 字节，约 26,200 行 |
| 其中顶层文档 | 59 份，739,682 字节；最大的是 `real-tasks.md` 197,893 字节（53 个小节，按时间流水记录） |
| `ST/` 全部文件 | 20,777 个，318 MB；`runtime/` 250 MB，3,729 个设备命令原件目录 |
| 内部 review | 46 份 |
| 模型运行 | 70 次，950 次 CLI 调用，输入 4,844 万 token，墙钟 4.2 小时 |
| 包内原件索引 | `PKG/EVIDENCE_INDEX.tsv` 21,216 行 |
| 产品变化 | 34 个提交，净 −4,295 行（+4,595 / −8,890），其中 Kotlin 主源码 −1,405、主机 −847 |

**46 份内部 review 找到过真问题吗？找到过，但不多。**
- **产品源码层面**：
  - `input-review.md` M1：不可核验的原选区被新加的 unchanged 条件困在局部循环，挤掉动作后观察（`UiActions.kt:305-311`）。真实行为缺陷，已修。
  - `host-m8-review.md` M1：batch 在 text 模式下，某步已返回后的本地异常不打印原因。真实缺陷，已修。
- **文档漂移**：`cumulative-maintenance-review.md` M1、M2（公开连接说明仍要求读已删的服务连接状态）；`input-review.md` M2。
- **实验代码**：`collection/probes/source-review.md` M1（探针用例在没有真正等待截止时也能通过）。
- **计划和题面措辞**：`capability-gate-review.md` MF1、MF2（拖拽归类与图像题前提的计划解释偏严），`input-target-question-review.md` M1、M2。
- 其余约 34 份都是"必须修 0、可选 0"或只有可选。

所以，按源码 review 计约 28 份，找到 2 个产品缺陷和 3 处文档漂移，命中率不低，值得保留。另有约 13 份是方法、运行、题面 review：`b2-method`、`m1p1-method`、`m1p1-runtime`、`region-method`、`marks-live-method`、`migration-question`、`input-target-question`、`capability-gate`、`plan-review`、`remaining-runtime`、`runtime/api34/cold-checkpoint`、`validation-integration`、`docs-consolidation`。它们除了修计划措辞之外没有改变任何产品结论。

**第 22 项照做了没有：部分照做。**
- "每项改动独立源码 review"做了：README 逐行对应提交和 review。
- "去掉对取证文档的反复复核"没有完全做到。上面 13 份里，`m1p1-runtime-review`（"独立离线复核 M1/P1 末节及其十份 metrics、首次 final、CLI 索引"）和 `remaining-runtime-review` 就是对运行证据的复核。
- 交付包又附了 `verify_package.py`，逐条比对 8,451 条轨迹记录，这是"对证据的证据"。

**文风。** 实施方文档里的防御性否定极多：
- `ST/*.md` 顶层 59 份里，"不能"409 次、"不是"120 次、"不把"80 次、"不据此"12 次、"不代表"14 次、"不声称"9 次；
- 工作稿首页一段就有"不能据此宣称全量通过""不冒称实际模型收益""不是证明消失窗口内必无元素""不据普通替代删除已选能力"（`T/deliverables/r4-candidate.md`）；
- 公开指引的否定密度与基线持平（每百词约 1.6 次 "not"），所以这个问题主要在过程文档，不在产品。

对读者的负担是实在的：每句话先排除三种误读，再说一件事；读者要自己从否定里拼出正面结论。这种文风是被前几轮"证据要如实、不得夸大"的要求逼出来的。

**我对"钻牛角尖"的具体判断：**

实施方这一轮把精力花在三处。

1. **产品修正（值得）。** 三大删除、紧凑阅读（重放字节降到旧版的 29%–49%）、动作选项收缩、会话到期、启动落点、wait 多匹配、工作区权限。这部分高效：两天内完成 34 个提交，删除都很干净。
2. **第 6 项能力验收与成本门的取证（大部分不值得）。** `real-tasks.md` 的 53 个小节里，大部分是在为验收找题目和恢复环境：
   - Maps 网络错误（C1 冷组、Play 组）；
   - 找"非它不可"的拖拽题；
   - NPS 自然相册凑十题图像集，结果"未取得合格十题"；
   - 区域截图十题，结果"正确率门未过"；
   - M1/P1 多指的 MuMu 恢复。

   结果是：batch 和 scroll-until 的首轮效率门没过，区域截图门没过，标注没有成绩，拖拽被坐标题卡住。验收本身没有给出"保留还是降级"的结论。按第 6 项的规则，要连续两次全量不过才降级，所以下一轮还得再做一遍。
3. **结果语义的边角修复（一半值得）。** 回调畸形快速报错、清理排队与失败区分、本地失败后保留执行事实，单看都对，但每个都配了 200–300 行时序测试。这是第四轮"失败、结果未知、部分结果都要如实"要求的延伸。

**评审方第四轮的源头：**
- **第四轮评审 §7 和 §9**（`androidctl-r4-candidate-review-4235af9.md:353`）提出"固定任务集，基线和候选各 5 次以上，同一快照起跑"。对于看"两倍以内的差别"这是对的，但被套用到了第 6 项每个少用能力的验收上。
- **第 6 项的验收细则**：5 次里成功 3 次；中位轮数少三成；十来个小题。这些是讨论中按"能力全保留"的前提补出来的门槛。第四轮评审方的子报告 43 原本建议删掉这些能力（`sub-reports/43-scale-maintainability.md` §3.3），最后的折中却是"保留，加验收"，代价比删和保都高。
- **第 7 项的 1.15 倍阈值**：门槛量的是"同内容采样的耗时比"，结果 1.1994 没过就保留了手写采集器。在一个 8,192 节点样本上差 4 个百分点，决定的是要不要继续维护手写 Binder 回调。这个门槛量的东西对，但阈值定得太死（另有专题评，这里只记它是验收投入的来源之一）。
- **"取证如实"的反复强调**：第三、四轮评审都要求"失败、UNKNOWN、partial 如实保留，不把单次成功说成普遍"。这本身没错，但实施方把它内化成了每句话都要附免责，以及 46 份 review。

---

## 5. 对两道待决题的看法（只从维护成本角度）

- **坐标题（M6）**：第四轮记的缺陷——候选按截图坐标点击时，要求接收触摸的窗口全属于同一 UID，桌面因壁纸窗口也接收触摸而被拒。
  - 按题面（`ST/input-target-decision.md:38-41`），A 到 D 的差别都在"派发时传不传目标 UID"。四个选项都保留首个 DOWN 点"不得命中 worker 自身界面"的前检，因而都保留 WindowInfos 采样、两个编译期桩、R8 规则、触区和变换计算、注销清理。
  - C 还要多一个公开参数和两套准入语义，维护成本略高；A、B、D 在代码量上相近【源码与题面，推断差别在几十行以内】。
  - 所以从维护角度看，这道题选哪项都不会明显减负，应按能力取舍来定。实施方的推荐 B 不增加参数，维护上没有劣势。
  - 题面漏掉了一个维护上真正省事的选项：去掉 WindowInfos 前检，回到基线的普通注入。这能删掉约 300 行源码、约 180 行测试、2 个桩和 1 条 R8 规则。代价是失去"不点到自己界面、不点到别的 App"的保护。这条保护属于第四轮"维持不变"的已定事项（"落点不属于目标 App 就不发"）。我不推荐推翻，只是指出：这道题的四个选项都不是减负选项。
- **归因题（A/B）**：两个选项的代码差别很小，都是在已有任务表比对上多一处调用。它的真实代价是被当成"完整行为"的锚点，挡住了 `--multiple-task` 的删除和 close 时的比对（D1）。建议先把与归因无关的部分做完。

---

## 6. 需要用户裁决的新取舍

### A1. 第 6 项能力验收要不要继续做

背景：70 次运行里，正常任务从不使用 batch、多指、拖拽、剪贴板、通知（§4.5）。验收首轮没有一项给出明确结论，按规则还要再跑一轮全量。

- **A**：维持第 6 项原样，下一轮继续做全量验收。得：规则一致。失：再耗一轮准备和环境恢复，结论仍可能不明。
- **B**：保留代码和指引，**取消逐项验收门**；这些能力在首页标"高级，按需使用"，只在真实任务暴露问题时再投入。得：立即省掉最大的一块投入。失：没有"它确实有用"的证据。
- **C**：降为实验（不进首页、不再投入），代码保留。得：维护面明确缩小。失：违背第 6 项的"正式能力"定位。
- **D**：删除其中零使用且隐藏接口最重的两项（结构化剪贴板的空值证明、通知监听器），其余按 B 处理。得：去掉约 800 行和约 10 个隐藏成员。失：失去对应能力。

**推荐 B**：成本最低，可逆。剩余未知：用户是否有本项目之外的真实需求依赖这些能力。

### A2. 结果语义收拢的程度

- **A**：维持现状（185 个码，过程计数进入回执）。
- **B**：参数类码合并成 `arguments` 加 `field`，"已变化"类合成两类，约减到 120 个；过程计数移进 `--raw` 的诊断。得：指引 reference 表变短，调用者读回执更快。失：要改契约和指引一次。
- **C**：按 §3.5 的设计统一成 `Outcome` 加约 40 个码加"前缀"，分子系统逐步迁移。得：维护负担的第一来源被根除。失：数周工作，期间两种形态并存。

**推荐 B 现在做、C 作为合入后的方向**。剩余未知：调用者是否依赖某些细分码做分支（70 次轨迹里未见，但未逐条核）。

### A3. 过程与证据的规格

- **A**：维持现状。
- **B**：只保留"每个产品提交一份源码 review"和"每次运行记录版本"。不再写方法、运行、题面 review；`real-tasks.md` 改成一张结果表；文档只写正面结论，限制在 1 页工作稿加 1 张结果表。
- **C**：在 B 的基础上取消外发包的原件索引和逐条校验，评审方按需索取原件。

**推荐 B**；C 取决于评审方是否愿意按需索取。剩余未知：评审方下一轮能否接受更少的原件。

### A4. 契约样本与重生成流程

- **A**：维持 438 KB，只补 POSIX 重生成命令。
- **B**：瘦身到约 200 KB（采样减到 3–4 个，`output` 段改为生成式），加 POSIX 流程。
- **C**：改为 schema 驱动，两端模型和样本都从 schema 生成（属于 A2-C 的一部分）。

**推荐 B**。

以下归为实施细节，不需要用户定：删 `--multiple-task`、删墓碑测试和旧文件兼容、`ClipboardAccess` 反射移入 `HiddenApi`、拆大文件、修测试提交合并、在当前头跑完整检查、历史整理只在合入前做一次。

---

## 7. 没能核实的内容

- Android 测试在 `0832f88` 上是否正好 526 项通过：日志没有计数，只能与 `@Test` 个数对照。
- 当前头 `cab84b8` 的完整 `task quality` 和 `task ci`：我不能跑 Gradle，也没有重跑主机全量（主评审在跑）。我只跑了 2 个主机测试文件（395 项通过）。
- 70 次运行里，调用者是否在决策中用过"近期事件"、细分原因码或通知的 PendingIntent 字段：我只统计了命令名，没有逐条读模型轨迹。
- 坐标题各选项的精确代码差：只读了题面和 review，各选项都没有实现，无从逐行对照。
- docs 仓库只抽查了 4 处一致性，没有通读 28 万字节。
- 重新设计中"附件落盘、按 id 拉取"的性能和可靠性：设计建议，未验证。
