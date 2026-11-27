# 专题 130：规模、职责划分和长期维护成本（第八轮，候选 `409e853`）

对象：产品 `409e853`（`S/r8/prod`），对照 `f401deb`（第七轮候选）、`fb75afc`（第六轮候选）、`7d5be70`（基线）；docs `090b1a2`，对照 `dfe5e45`。全程只读，没有设备。`S` 指评审暂存目录。我的脚本和中间结果在 `S/agents14/130/`：`sizes.sh`→`sizes.tsv`（四版本逐文件行数和字节数），`agg.py`、`groups.py`（按文件和模块汇总），`funclen.py`、`ktfun.py`（函数长度），`clitree.py`→`clitree.json`（从 `409e853` 的 Typer 注册里导出的 61 个叶命令及其参数），`ledger.py`→`ledger.json`、`ledger-opts.txt`（48 次尝试的命令和参数台账），`asserts.py`（主机测试断言粗分类），`fields.py`、`dead.py`（无消费者的粗筛）。

我原本想在改过的副本上跑主机测试，做"改一处输出会让多少测试失败"的实验。这一步被权限系统拦下了（共用说明也要求不要重复跑全量），所以第 4、5 节"要改几处"的数字全部来自源码检索和提交差量，属于源码推断，不是实跑结果。

## 结论（先读这里）

1. **本轮名义上是"打磨和维护性删减"，但除了独立 docs 仓库，每一项规模都在涨。** 主代码 +898 行（Kotlin +487，Python +411），测试 +2,507 行（Kotlin +931，Python +1,576），公开指引 +6.0 KB（+8.4%）。测试增量与主代码增量之比约 2.8 比 1。唯一真正的删减在 docs 仓库：301,632 字节降到 149,809 字节，刚好低于 150 KB 的目标（只差 191 字节）。【我量的】
2. **第五节丁的五项维护性删减（第七轮决定文件里列的五项必做删减），只有 docs 那一项净删了东西。** 首行函数 `result_line` 从 280 行降到 113 行，达到了不超过 120 行的目标，但同一个文件 `ui_output.py` 从 598 行涨到 734 行（+136）。"测试解析集中"那次提交让测试净增 210 行。退出码集中是把 19 处 `put("exit", N)` 换成一行包装函数（`setExit`／`set_exit`），没有加任何约束。通用 loading 判定确实删掉了三个只对 Maps 有效的字符串，代码行数大约持平。【源码、提交差量】
3. **字段表没有接住之后的输出改动。** 重构提交 `5f76af5` 把 `result_line` 降到 88 行。随后 5 个提交又改了大约 10 处首行输出，其中只有 1 处（`wait` 的 `cutoffStage`）用到了字段表，其余都直接写成条件代码；`result_line` 本身在同一轮里又长回 25 行。`scroll-until` 找到目标时的特例现在分散在两个函数里。所以按"加一个首行字段要改几处"来量，`f401deb` 与 `409e853` 基本一样：大约 7 个文件（设备端生产代码、Kotlin 测试、契约样本、主机端呈现代码、一到两个主机测试、指引）。字段表只把其中"主机端呈现代码"一项从一处条件代码变成一行表项，而且只适用于 6 类命令。【源码】
4. **本轮新造了一处"两端各算一遍"。** 主机端 `ui_output.py:120-126` 的 `show_baseline_age` 条件，与设备端 `UiPostAction.kt:205-210` 的 `usesCurrentBaseline` 逐项相同（back/home/recents/notifications/open，不带引用的 key，`type --via keys`）。它是 `b263466` 加的。以后改其中一端，另一端不会报错。【源码】
5. **48 次尝试的命令台账。** 候选和新题 38 次尝试共 852 条命令，只用到 61 个叶命令里的 25 个；66 个参数名里有 46 个一次都没用过。公开指引里本轮涨得最多的专题 `observations.md`（13.1 KB，+2.2 KB），38 次候选尝试里**一次都没被读过**；`connection.md` 也是 0 次。调用者读的几乎只有首页 `SKILL.md`（38/38）。所以没被用到的能力，大多属于"指引首页没教、调用者不知道"，而不是"确实没有场景"；新加的专题文字没有到达调用者。【实测，原始会话记录的工具调用】
6. **评审方自己要的统计字段漏进了调用者的首行。** `rootUnavailableObserved`／`rootRecoveredWithinRequest` 是为新-5（评审方上一轮新提、留给本轮用户再定的问题：没有可用界面树时，读取失败后要不要一直重试到命令时限）的统计加的。实测里它出现在 23 条命令的首行，其中 20 条写的是 `rootRecoveredWithinRequest=true`，16 条是 `click: done`；调用者对这一事实无事可做。这是为极少见情形付出常驻输出、状态和测试的例子。【实测，`r7-new-01/evidence/cli/1795733576791130700-…/child.stdout:1`】
7. **测试在增长，但大多是新增，而且有一部分在钉死实现。** 本轮输出类提交的测试改动以新增为主（例如 `fb35dbf` 测试 +539/−109），说明"改输出就碎一片"有所缓解。但解析函数迁移后，有 6 个测试同时保留了整行逐字比较和逐字段断言，两种断言重复。新增的 `PostActionFactsTest.kt`（331 行）直接断言 `PostActionWindow.collectionUntil`、`until`、`canStart(1600)` 这类内部时间状态。【源码】
8. **再来一轮同样形式的打磨，代码库会更难维护。** 依据：连续三轮每条"下-N"都落成"新状态加新字段加新测试加新指引段落"；连标着删减的两项重构，在同一轮里都被后续条目部分抵消；新文字没有被调用者读到。详见第 7 节。

---

## 1. 规模表

### 1.1 四个版本的总量（`wc -l`；指引为字节）

| 类别 | 基线 `7d5be70` | 第六轮 `fb75afc` | 第七轮 `f401deb` | 本轮 `409e853` | 本轮增量 |
| --- | --- | --- | --- | --- | --- |
| Kotlin 主代码 | 9,360 | 19,249 | 19,607 | 20,094 | +487 |
| Kotlin 测试 | 4,612 | 16,728 | 17,639 | 18,570 | +931 |
| Python 主代码 | 8,008 | 11,495 | 11,569 | 11,980 | +411 |
| Python 测试 | 16,775 | 25,507 | 26,447 | 28,023 | +1,576 |
| 公开指引 `skills/` | 81,558 B | 68,991 B | 72,311 B | 78,355 B | +6,044 B |
| 发布工具 `tools/` | 1,431 | 1,431 | 1,431 | 1,431 | 0 |
| 契约 `wire-contract.json` | — | 363 KB | 364 KB | 371 KB | +7 KB |
| 独立 docs 仓库 | 196,438 B | 290,239 B | 301,632 B | 149,809 B | −151,823 B |

主评审给的粗数字我全部核对过，一致。补充一点：Kotlin 测试里有一个 12 行的 `android/icu/lang` 桩文件，不影响结论。

指引按文件（字节，`f401deb`→`409e853`）：`observations.md` 10,878→13,106（+2,228），`actions.md` 11,136→13,068（+1,932），`connection.md` 7,365→8,192（+827），`displays.md` 6,217→7,004（+787），`images.md` +327，`input.md` +199，`SKILL.md` −4，`reference.md` −253。

### 1.2 按模块（两端都是平铺目录，我按文件前缀和职责归类，见 `groups.py`）

Kotlin 主代码：

| 模块 | 基线 | 第六轮 | 第七轮 | 本轮 | 本轮增量 |
| --- | --- | --- | --- | --- | --- |
| UI 动作：动作、fill、滚动、等待、建议、借用粘贴 | 1,969 | 4,704 | 4,828 | 5,175 | +347 |
| UI 读取：采集、快照、投影、语义、交付、引用 | 2,389 | 5,181 | 5,237 | 5,291 | +54 |
| 连接、网关、worker、协议 | 2,674 | 2,929 | 2,969 | 3,014 | +45 |
| 后台屏、activity、会话 | 231 | 2,064 | 2,202 | 2,226 | +24 |
| 图像、手势 | 359 | 1,449 | 1,449 | 1,467 | +18 |
| 剪贴板、通知 | 0 | 1,097 | 1,097 | 1,096 | −1 |
| 作业、文件 | 994 | 1,007 | 1,007 | 1,007 | 0 |
| App 管理界面 | 744 | 818 | 818 | 818 | 0 |

Python 主代码：

| 模块 | 基线 | 第六轮 | 第七轮 | 本轮 | 本轮增量 |
| --- | --- | --- | --- | --- | --- |
| UI 命令、客户端、视图、输出 | 1,909 | 4,733 | 4,782 | 5,072 | +290 |
| CLI 入口、参数、通用输出 | 1,054 | 1,378 | 1,398 | 1,497 | +99 |
| 守护进程、协调、本地传输、状态 | 2,677 | 2,884 | 2,899 | 2,922 | +23 |
| 配对、身份 | 538 | 589 | 589 | 589 | 0 |
| 作业、文件 | 1,830 | 1,911 | 1,901 | 1,900 | −1 |

本轮增长几乎全落在两端的 UI 层（Kotlin UI 动作 +347、Python UI +290），其次是 `app.py` 的生命周期回执（+97）。作业、文件、配对、通知、剪贴板没动。逐文件完整表在 `sizes.tsv`。

### 1.3 本轮净增最多的 15 个文件，各自涨在哪

| 文件 | 本轮 | 四版本行数 | 涨在哪（提交：增/删） |
| --- | --- | --- | --- |
| `PostActionFactsTest.kt` | +331 | 0/0/0/331 | 全新（`b263466`）：观察窗口、首读放宽、焦点丢失、根读失败和恢复、wait 截断的单元测试 |
| `test_result_output.py` | +328 | 821/1220/1290/1618 | `5f76af5` +114/−61（改用解析函数）、`b263466` +98、`29547c6` +103（生命周期回执）、`fb35dbf` +83 |
| `WorkerConnectionTest.kt` | +293 | 0/0/279/572 | `8c4a28d` +331/−38：worker 重连归属、恢复提示 |
| `test_observation_files.py` | +289 | 1250/1931/1951/2240 | 5 个提交逐次追加，全是新增，几乎没有删 |
| `test_ui_reading_delivery.py` | +255 | 0/0/301/556 | `fb35dbf` +120、`fa41bce` +69、`82f3a82` +93/−21：交付失败链 |
| `UiPostAction.kt` | +177 | 195/237/286/463 | `b263466` +225/−48：`PostActionWindow`、`onlyNavigationChanges`、`lostFocus`、根读 sticky 事实 |
| `test_ui_view.py` | +174 | 395/1023/1062/1236 | `fb35dbf` +195/−21：读取视图的省略和折叠 |
| `ui_output.py` | +136 | 0/530/598/734 | `5f76af5` +185/−119（字段表）、`b263466` +33、`fb35dbf` +66 |
| `test_cli_arguments.py` | +125 | 606/767/801/926 | `29547c6` +121：命令行错误提示 |
| `test_wire_contract.py` | +105 | 460/1419/1451/1556 | 5 个提交分摊 |
| `app.py` | +97 | 794/1039/1059/1156 | `29547c6` +112/−15：`task_disposition_summary`（38 行）、用法错误提示、状态字段过滤 |
| `WireContractTest.kt` | +91 | 714/1929/1964/2055 | `b263466`、`fb35dbf` |
| `test_ui_delivery.py` | +85 | 482/866/863/948 | `b263466` +60 |
| `test_display.py` | +84 | 0/306/360/444 | `29547c6` +95 |
| `ActivityLaunchTest.kt` | +63 | 0/927/1014/1077 | `b263466` +78/−15 |

15 个里有 11 个是测试文件。主代码里涨得最多的是 `UiPostAction.kt`、`ui_output.py`、`app.py`。

### 1.4 第五节丁五项维护性删减的实际账

| 项 | 目标 | 实际 | 删了 | 加回 | 判断 |
| --- | --- | --- | --- | --- | --- |
| 1. 测试里的输出解析集中到一个函数 | 集中到一个函数 | `ui_test_support.py:12` 的 `parse_result_line`，共 47 行，21 个测试文件调用 124 次 | `5f76af5` 测试 −296 | +506（净 +210）；之后 5 个提交测试又 +1,375/−329 | 做到了"集中"，但没有减少行数。部分迁移是在原有整行断言之外再加逐字段断言（见第 4 节） |
| 2. 首行改成按命令声明字段的表 | `result_line` ≤120 行 | 113 行（`5f76af5` 时 88 行，之后长回 25 行） | 函数内约 167 行 | `ui_output.py` 整体 +136；表 43 行；新增 11 个语义小函数 | 函数行数达标，文件变大。表只管 6 类命令（剪贴板、通知读取、select、open、scroll-until、wait），其余仍是代码 |
| 3. 内部 docs 删减 | ≤150 KB | 149,809 B | −859 行、约 152 KB | +392 行 | 达标，是五项里唯一真正的净删。`testing/cases.md` 57.7 KB，占剩余 docs 的 38% |
| 4. 合并两套 loading 判定，删 Maps 专用文字 | 产品里不留只对某道题有效的文字 | `UiSemantics.loading(nodes, includeText)` 一个定义（`UiSemantics.kt:398-420`），文字集只剩 `"loading"`、`"正在加载"` | 3 个 Maps 字符串，`navigationLoading` 13 行 | `loading()` 加 10 行 | 达标。语义上有一点变化：导航判定现在也认进度条；`SettledSamples.accept` 在没有基线时把 loading 树当作不完整（`UiPostAction.kt:269`） |
| 5. 设置退出码的 19 处改成调用同一个函数 | 两种语言各一个函数，不改行为 | Kotlin `Wire.kt:56` `setExit(code) = put("exit", code)`；Python `protocol.py:66` `set_exit`。19 处全部替换（`f03cc76`，+25/−26） | 0 | +7 | 形式上做到了，维护上没有收益：函数里没有约束，也不要求同时给出原因。`exit_code()` 里"有交付错误时 0 改 1"的规则仍在另一处（`protocol.py:69-80`）。`local_call.py:244-271` 仍直接构造 `exit=` |

小结：五项里，第 3 项在独立 docs 仓库里，第 4 项删的是字符串。产品仓库里没有任何一项让总行数下降。

---

## 2. 职责划分

### 2.1 各模块一句话职责

Kotlin（`android/app/src/main/java/com/rainng/androidctl/`）：
- UI 读取：通过隐藏接口和无障碍服务采集窗口树（`TreeCollector`、`Collection*`），投影成稀疏节点（`UiSemantics`、`NodeFacts`、`NodeEncoding`），发布快照并按引用和预算交付（`UiSnapshot`、`UiPublication`、`UiDelivery`、`UiBudget`）。
- UI 动作：解析目标，派发动作、填写、滚动、按键，做动作后的观察和确认（`UiActions`、`UiPostAction`、`UiCondition`、`UiScroll*`、`AppInput`、`BorrowedPaste`），以及拒绝后的建议（`UiAdvice`）。
- 后台屏、activity：创建虚拟屏，启动和搬动任务，会话结束时清理任务（`ActivityLaunch`、`DisplayTasks`、`TargetManagement`、`WorkDisplay`）。
- 连接：TLS 网关、worker 进程绑定和重连、RPC 分派（`GatewayService`、`NetworkGateway`、`WorkerConnection`、`ShellProtocol`）。
- 作业、文件：shell 作业、输出存储、文件传输（`ShellJob*`、`FileJob*`、`OutputStore`、`TransferHandle`）。
- 剪贴板、通知、图像、手势：各自独立的小块。

Python（`androidctl/src/androidctl/`）：
- `app.py`、`cli_options.py`：Typer 命令注册和非 UI 命令的呈现。
- `ui_commands.py`：UI 命令注册；`ui_client.py`：组请求、调用、保存和交付正文；`ui_view.py`：把稀疏树渲染成文本；`ui_output.py`：首行。
- `coordinator.py`、`transport.py`、`local_call.py`、`local_server.py`、`daemon.py`：本地守护进程、设备连接、锁和截止。
- `identity.py`、`private_files.py`：配对和凭据；`job_commands.py`、`file_jobs.py`、`user_files.py`：作业和文件。

### 2.2 超过 800 行的文件

| 文件 | 行数 | 判断 |
| --- | --- | --- |
| `ui_commands.py` | 1,756 | 其中 `register()` 一个函数就有 1,326 行（基线 553，`f401deb` 1,273）。它把全部 UI 命令写成闭包注册在一个函数里。每个闭包本身不大，但整个文件没法按命令拆开读，也没法单独导入。设计取舍，见第 6 节第 8 项。 |
| `ui_client.py` | 1,170 | 一个文件管四个阶段：请求准备（`prepare_ui` 197 行）、执行、渲染正文（`render_observation` 153 行）、保存和交付（`save_delivery`、`finish_delivery_result`、`print_delivery`）。阶段划分是清楚的，只是都放在一个文件里。可以按"请求"和"交付"拆成两个文件，但收益不大。 |
| `app.py` | 1,156 | 一个文件管 40 个顶层函数：全局选项、状态、设备、会话、技能导出、帮助、错误 JSON、用法错误提示。职责最杂。本轮加的用法错误提示依赖私有模块 `typer._click.exceptions`（`app.py:19`）。typer 已经精确锁版本，所以不是缺陷，但每次升级 typer 都要人工核一次。 |
| `UiActions.kt` | 1,145 | 同一个类里管 locate、act、select、touch、key、借用粘贴（内部类）、range、行勾选、fill（内部类，focus/fill/submit 三步）、值校验和焦点等待。是两端最典型的"一个文件承担多件事"。 |
| `ui_view.py` | 977 | 属性、投影、阅读裁剪三层，职责清楚。`reading_view` 164 行，偏长。 |

两端都没有特别长的单个函数。Kotlin 最长的是 `NodeFacts.read`（114 行），`UiPostAction.settle` 76 行。复杂度不在函数长度上，而在每次请求要带的状态和事实数量上：`UiPostAction` 一个文件里就有 `PostActionWindow` 的 4 个可变状态，`SettledSamples` 的 8 个可变状态，`report()` 的 7 个参数。

### 2.3 同一个事实两处各算一遍

| 事实 | 第一处 | 第二处 | 现状 |
| --- | --- | --- | --- |
| 哪些命令以"当前样本"为基线，需要显示基线年龄 | `UiPostAction.kt:205-210` `usesCurrentBaseline` | `ui_output.py:120-126` `show_baseline_age` | **本轮新造**（`b263466`），没有共享样本，也没有测试对比两端。建议设备端直接发一个字段，主机只读不算。代价：改约 10 行；需要更新一处契约样本。 |
| 文字归一化（NFKC、大小写折叠、连字符变空格） | `UiCondition.kt:265` `folded` | `ui_view.py:401` `folded` | 上一轮的建议已落实：契约里新增了 `textFolding` 共享样本（`wire-contract.json:491`）。核过，没有问题。 |
| 只读命令集合 | `RpcMethods.kt:21` `UI_READ_ONLY`（含 `apps`） | `protocol.py:30` `READ_ONLY_COMMANDS`（含 `find`） | 两个集合用途不同（调度和呈现），差异合理。记为"核过"。 |
| 首行问题字段（code/argument/subject/stage） | `output.py` `Output.error_fields` | `ui_output.py:236-241` `reason_facts` 里的循环 | 实施方自己的复查 O2 提出过。`app.py` 已经改为调用 `error_fields`，UI 这边仍是复制的一份。改一处调用就能消掉。 |
| freshRepair（旧引用被拒、已附新树）时机判断 | `UiRun.kt:36-44` | 由 `UiPublication` 和 `withActionFacts` 两个出口消费 | 判断本身只有一份，两处是消费。核过，没有重复。 |

### 2.4 状态分散、需要同步的地方

- **一个首行字段散在约 7 个文件里。** 本轮新加的字段实际分布（`git grep -c`）：`cutoffStage` 7 个文件，`rootUnavailableObserved` 7 个，`freshRepair` 14 个，`collectionCutoff` 8 个，`noChangeObserved` 4 个。这些文件之间没有任何机制保证一致，只靠评审。
- **原因码表 `contract/reason-codes.json`**（150 行，24.6 KB）：登记测试会双向核对"源码里有这个码"和"表里有这一行"，但表里每条的"下一步"文字没有任何运行时消费者，CLI 不输出它，指引也只提到 39 个码。它是第三份需要和源码、指引同步的文字。
- **观察相关的 sticky 状态**：`SettledSamples` 的 `rootUnavailableObserved`、`rootRecoveredWithinRequest`，`UiWait` 的 `sampleLimitObserved`、`previousLimits`、`cutoffStage`，`PostActionWindow` 的 `firstAttempt`、`waitingForApp`、`usable`。实施方复查（`r7-cumulative-maintenance.md` 第 49-50 行）逐个说明了它们各有消费者。我抽查了源码，这个说法成立；但它也说明，一个"动作后再看一眼"的功能，现在要维护十几个彼此相关的标志位。

---

## 3. 无消费者路径

### 3.1 没有调用方的函数

用名称引用计数粗筛（`dead.py`）：Python 只找到 `local_server.py:33` 的 `is_serving`，它是覆盖 asyncio 的方法，不算死代码。Kotlin 没有找到只被定义、从未引用的 `fun`。**核过，基本干净。** 这说明实施方的问题不在于留下死代码，而在于给活代码加分支。

只为测试而存在的生产接口，我抽到一例：`SettledSamples.observe` 的 `budget: UiBudget? = null` 和可空的 `inspect`。生产代码的两处调用都会传 budget（`UiPostAction.kt:169,182`），只有 `CollectionPublicationTest.kt:109,119` 走 null 这条路。影响很小，记为可选。

### 3.2 没有发出方的原因码

登记测试双向核对，所以不存在"表里有、源码里没有"的码。真正的问题在使用面：150 个码里，38 次候选尝试的 852 条命令输出中只出现过 5 个：`stale` 5 次、`scopeUnknown` 5 次、`notReady` 5 次、`rootUnavailable` 2 次、`routeUnavailable` 1 次。这不是删码的理由，因为罕见失败本来就罕见。但它说明那 24 KB 的"下一步"文字是按 150 种情形写、却从不展示给调用者的维护负担。

### 3.3 设备端新字段的使用情况

本轮新加的 6 个设备端字段在 852 条候选和新题输出里的出现次数：

| 字段 | 出现次数 |
| --- | --- |
| `collectionCutoff` | 25 |
| `rootUnavailableObserved`／`rootRecoveredWithinRequest` | 23 |
| `freshRepair` | 4 |
| `cutoffStage` | 0 |
| `noChangeObserved` | 0 |
| `samplingStopped` | 0 |

后两项和 `samplingStopped` 的触发条件本来就窄，0 次不说明它们错了，但说明它们的价值没有实测证据。

### 3.4 命令使用台账（48 次尝试，`ledger.json`）

口径：候选 20 次（`82f3a82`）、新十题 18 次（`82f3a82`）、基线 10 次（`7d5be70`，命令集不同，只作参照）。叶命令和参数取自 `409e853` 的实际注册。"指引"一栏是该命令或参数在 `skills/androidctl/*.md` 里出现的次数，括号里是首页 `SKILL.md` 的次数。

用过的命令（候选/基线/新题）：

| 命令 | 候选 | 基线 | 新题 |
| --- | --- | --- | --- |
| `click` | 101 | 59 | 226 |
| `snapshot` | 63 | 43 | 129 |
| `fill` | 27 | 29 | 65 |
| `scroll` | 37 | 38 | 21 |
| `back` | 31 | 2 | 30 |
| `scroll-until` | 18 | 0 | 4 |
| `open` | 5 | 0 | 12 |
| `recents` | 0 | 0 | 15 |
| `status` | 9 | 0 | 0 |
| `apps` | 5 | 0 | 3 |
| `home` | 0 | 0 | 8 |
| `ime` | 0 | 0 | 7 |
| `notifications`（不带子命令） | 0 | 0 | 5 |
| `session open`／`session close` | 各 5 | 0 | 0 |

以下命令各有 1 到 4 次：`key`、`help`、`wait`、`find`、`check`、`long-click`、`collapse`、`type`、`pinch`、`uncheck`、`notifications list`。基线大量使用 `screenshot`（30）、`tap`（20）、`swipe`（19），候选和新题几乎不用（`swipe` 1 次）。

候选和新题里从未用过的 36 个叶命令：

| 类别 | 命令 | 指引覆盖 |
| --- | --- | --- |
| UI 精细动作 | `locate`、`action`、`range`、`select`、`reveal`、`focus`、`expand`、`dismiss` | 专题里各提到 1 到 4 次，首页只有 `reveal` |
| 手势 | `timeline`、`drag`、`tap` | 只在专题里 |
| 作业、文件 | `shell`、`push`、`pull`、`job *` | 只在专题里 |
| 剪贴板 | `clipboard *` | 只在专题里 |
| 通知 | `notifications get` | 只在专题里 |
| 管理类 | `pair`、`devices`、`use`、`apk`、`skill`、`session takeover`、`device *`、`daemon *`、`wake` | — |

参数：66 个参数名里有 46 个在候选和新题中从未出现，例如：

- 读取类：`snapshot` 的 `--cached`、`--raw`、`--all`、`--in`、`--window`、`--depth`、`--interactive`、`--max-chars`、`--marks`、`--out`，以及 `find --context`；
- 等待类：`wait` 的 `--gone`、`--app`、`--idle`、`--any-of`、`--screen-change`；
- 启动类：`open` 的 `--url`、`--package`、`--component`、`--extra`、`--move-existing` 等；
- 选择器：所有动作命令共有的 `--text`、`--id`、`--in`、`--window`、`--windows`、`--all-windows`、`--timeout`、`--settle`。

实际用到的参数集中在：`fill --submit`（55）、`snapshot --screenshot`（68）、`scroll-until --contains/--max-steps/--timeout`（约 20）、`click --name/--role`（53）。

调用者读了哪些指引（从原始会话记录的工具调用里统计）：

| 文件 | 候选和新题 38 次里读过的次数 |
| --- | --- |
| `SKILL.md` | 38 |
| `images.md` | 7 |
| `actions.md` | 6 |
| `input.md` | 6 |
| `displays.md` | 5 |
| `reference.md` | 4 |
| `notifications.md` | 1 |
| `observations.md`、`connection.md`、`jobs-files.md`、`clipboard.md` | 0 |

另有 `androidctl help actions` 1 次。

**怎么读这份台账。** 按用户的原则，不以少量任务中的使用次数决定能力去留，所以下面把"没人用"分成三类：

- **调用者不知道（指引首页没教）**：`snapshot --cached`／`REF`、`--raw`、`find --context`、`wait --gone`、`locate`、`action`。这些是 Playwright 一类工具的核心读法，首页只提到 `--cached` 和 `--raw` 各 1 次，实测 0 次。它们出现在 0 次被读的 `observations.md` 里。**删它们的能力代价大**（调用者会被迫用新采加截图来替代），而它们本身又不大。应当保留，问题在指引。
- **确实没有场景（就这批题而言）**：作业、文件、剪贴板、`timeline`、`drag`、`range`、`select`。它们和题目形态无关，指引也写了，只是题目用不到。删掉的代价是失去能力，收益只是测试和文档行数。建议冻结，不删。
- **少用而且有更简单的替代**：`open` 的意图参数族（`--extra`、`--component`、`--category`、`--mime-type`、`--grant-read-uri-permission`、`--move-existing`），可以由 `shell am start` 覆盖；`session takeover`、`device address`／`device trust` 是管理类命令。删 `open` 意图参数族约能减 `activity_intents.py` 的 223 行和 `ActivityIntent.kt` 的 210 行，以及相应测试，代价是调用者要会写 `am start`。这需要用户取舍，见第 6 节问题 Q1。

---

## 4. 测试质量

### 4.1 数量和比例

本轮主代码 +898 行，测试 +2,507 行，比例约 2.8 比 1（Python 3.8 比 1，Kotlin 1.9 比 1）。全量上，测试是主代码的 1.45 倍（46,593 比 32,074）。

### 4.2 保护行为还是钉死实现（抽样）

- **好的一面。** 本轮输出类提交的测试改动以新增为主，删除很少：`b263466` −56、`29547c6` −27、`fb35dbf` −109、`fa41bce` −41、`82f3a82` −21。对比上一轮，一行修复要改 409 行测试。逐字比较的字面量也在下降：首行形如 `"cmd: done; …"` 的测试字面量从 30 处降到 22 处，`dispatchStarted=` 字面量从 15 处降到 6 处。可以判断解析函数确实降低了改输出时的连带改动。【源码检索】
- **重复断言。** 有 6 个测试同时用 `parse_result_line` 和整行 `==` 比较同一份输出。例如 `test_result_output.py` 的 `test_ui_text_shows_only_implicit_context`：保留了 `assert text == f"click: done{suffix}\n"`，又追加了 12 行逐字段断言。整行比较已经覆盖了这些字段，逐字段断言没有增加任何保护，只增加了维护量。
- **用启发式分类粗看主机测试断言**（`asserts.py`，只供参考）：对输出文字的整行等值约 16 处，对输出文字的子串断言约 94 处，`f401deb` 时分别是 18 处和 100 处，变化不大。所以"改一句回执文字"仍然要改 1 处源码加 1 到 2 处测试；这在两个版本里差不多，本来就不是瓶颈。
- **钉死实现。** 新的 `PostActionFactsTest.kt` 直接断言内部类的时间状态，例如 `PostActionWindow(0,1500,4250,true).collectionUntil == 4250`、`canStart(1600)`、`until == 3500`（第 17-35 行）。只要调整观察窗口的实现方式（例如把 `PostActionWindow` 合进 `UiBudget`），即使外部行为不变，这些测试也要重写。测试名也很长，例如 `theFirstTreeCanFinishPastSettleWhileFailuresDoNotExtendRetryAndNoAppGetsANewWindow`、`repairProvenanceSurvivesAChangedRefusalReasonButDoesNotDescribeOtherSamplesOrDispatches`，一个测试名就编码了一整条裁决，说明测试和规则逐条对应。规则一旦被推翻，就只能删测试，没法复用。
- **三层断言同一事实。** 新字段普遍在 Kotlin 单测、契约样本、Python 呈现测试三层各断言一次（见 2.4 的文件分布）。契约样本本来就是两端对拍的，Kotlin 单测和 Python 测试里重复断言字段存在与否的部分可以删。

### 4.3 删掉哪类测试不会降低保护

| 类别 | 估计行数 | 风险 |
| --- | --- | --- |
| 同时有整行比较和逐字段断言的测试里的逐字段部分 | 约 60 到 100 | 零 |
| 墓碑测试（"已删的选项必须被拒绝"）；上一轮列了约 10 处，小-56 允许不做，现状未变 | 约 100 | 零 |
| 契约样本已经覆盖的、Kotlin 单测里对 JSON 字段存在与否的重复断言 | 300 到 800，需逐个判断 | 低到中 |
| 断言内部时间状态的单元测试，改写为通过 `settle()` 的外部结果断言 | 不减行数 | 降低换实现时的成本 |

合计能删的约占测试的 2% 到 3%，与上一轮的判断一致。测试的问题不是太多，而是每加一条规则就固定加 3 到 7 个文件的测试。

---

## 5. 本轮两项重构和通用 loading 判定，是否降低了维护责任

衡量口径：`f401deb` 与 `409e853` 对比，全部是源码推断。

| 操作 | `f401deb` | `409e853` | 差别 |
| --- | --- | --- | --- |
| 给 `open`、`select`、`scroll-until`、剪贴板、通知读取、`wait` 加一个首行字段 | 在 `result_line` 对应分支加 1 到 3 行条件代码，另加设备端、契约、测试、指引共约 6 个文件 | 在 `COMMAND_FIELDS` 加 1 项；成功也要显示时，再在 `SUCCESS_FIELDS` 加 1 项。其余 6 个文件不变 | 少了几行，文件数相同（约 7 个） |
| 给其他命令（click、fill、scroll、snapshot、find……）加一个首行字段 | 在 `result_line` 里加条件 | 在 11 个语义小函数里选一个加条件，或者直接写进 `result_line` | 没有改善；本轮后续 10 处输出改动里有 9 处走了这条路 |
| 改一句回执文字 | 1 处源码，加 1 到 2 处测试 | 相同 | 无差别 |
| 加一种拒绝原因（新原因码） | 生产端 1 处，`reason-codes.json` 1 行（登记测试会逼你加），指引可选，测试 1 到 2 处；共 3 到 5 个文件 | 相同 | 退出码集中对此没有影响 |
| 加一种"完成但退出码非零"的情形 | 写 `put("exit", N)`，`f401deb` 的规则保证首行会打原因 | 写 `setExit(N)` | 只是换了写法；仍然没有强制同时写原因 |
| 改 loading 判定 | 两处：`UiSemantics.loading` 和 `UiPostAction.navigationLoading` | 一处，带 `includeText` 开关 | 改善，责任变成一处 |

结论：通用 loading 判定确实减少了一处责任。字段表在函数行数上达标，但只对 6 类命令有用，而且本轮后续改动大多没走它；它降低的是"看懂 `result_line`"的成本，没有降低"加一条输出规则"的成本。退出码集中没有降低任何责任。

---

## 6. 如果只有一个人维护：删、合并、重写的前十项

| 序 | 现状 | 动作 | 估计减少行数 | 能力代价 | 风险 |
| --- | --- | --- | --- | --- | --- |
| 1 | 指引首页之外的 9 个专题共约 71 KB，实测几乎不被读，`observations.md` 读了 0 次，却是本轮涨得最多的 | 把调用者真正需要的读法（`--cached`、`REF`、`find --context`、`wait --gone`、`--raw`）各用一两句写进首页；专题只保留参数细节；不再为每条裁决加专题段落 | 指引 −20 到 −30 KB | 无 | 低；需要重新实测（"未读"的成本已经在付） |
| 2 | `rootUnavailableObserved`／`rootRecoveredWithinRequest` 在首行出现，23 次里 20 次是"已恢复" | 首行只在结果不是 done、并且没有恢复时显示；sticky 统计只保留在 `--json` 里，或者整个删掉（它是为评审统计加的） | 约 −30 到 −60（含测试） | 无 | 低 |
| 3 | `show_baseline_age` 与 `usesCurrentBaseline` 两端重复 | 设备端发一个字段，主机端只读 | 约 −10 | 无 | 低 |
| 4 | `UiPostAction` 的 `onlyNavigationChanges`、`removeNavigationChanges`、`lostFocus`（约 90 行），用来在"只有焦点或动作变化"时把 changed 改报为 unsettled | 退回到只用几何变化的规则，或者合成一个"忽略字段集"的比较 | −60 到 −90，测试约 −100 | `change=` 在焦点转移时会多报 changed | 中；需要设备核对网页题 |
| 5 | `reason-codes.json` 有 150 条"下一步"文字，没有运行时消费者 | 二选一：A. 把文字接进首行的拒绝输出，让它对调用者有用；B. 表只留码和类别，删掉文字 | B 约 −15 KB | B 无 | 低 |
| 6 | `setExit`／`set_exit` 是空包装 | 要么删掉回到直接赋值（诚实），要么改成 `setExit(code, why)` 强制写原因（第七轮第 12 问用户选了 A，即不做后者） | ±10 | 无 | 零 |
| 7 | 首行字段表只覆盖 6 类命令，后续改动绕过它 | 不再扩展；接受"语义小函数加表"的现状，停止为它投入 | 0 | 无 | 零 |
| 8 | `ui_commands.register()` 一个函数 1,326 行 | 按命令族拆成 4 到 5 个模块级注册函数（读取、动作、手势、系统） | 行数不变 | 无 | 低；只是移动代码，`--help` 输出要做逐字节对照 |
| 9 | `open` 的意图参数族（`activity_intents.py` 223 行加 `ActivityIntent.kt` 210 行，另有测试） | 需用户定，见 Q1 | −400 到 −900（含测试） | 失去带类型的意图启动，要改用 `shell am start` | 中 |
| 10 | 重复断言（整行比较加逐字段；三层断言同一事实） | 去掉多余的一层 | −400 到 −900 | 无 | 低到中 |

**不该动的部分。** 采集器和隐藏接口；后台屏和任务清理；借用粘贴；作业和文件；配对和 TLS；契约对拍机制本身；`parse_result_line`；docs 现在的规模。理由：要么是核心能力，要么已经稳定、不再增长（作业、文件、配对三轮几乎零改动），要么刚刚做完删减。

**实施方在细枝末节上过度投入的具体证据：**

- `PostActionWindow` 为"第一份树还没拿到"的情形维护 `firstAttempt`、`waitingForApp`、`usable` 三个状态和四个派生属性（`UiPostAction.kt:27-54`），只为让一种情形下的首读多等一会儿。
- `lostFocus` 按"两对固定的焦点动作 ID 互换"来判断"只是焦点丢了"（`UiPostAction.kt:414-447`，函数上带着 `@Suppress("CyclomaticComplexMethod")`）。
- `task_disposition_summary`（`app.py`，38 行）为 `session close` 的回执逐个类别列出最多 5 个包名和省略数，并从 6 个候选字段里提取组件名。实测 `session close` 只用过 5 次。
- 首行出现 `rootRecoveredWithinRequest=true`、`timing=ref-repair`、`Rejected REF was not verified for this sample` 这类给评审看、而不是给调用者用的事实。
- 实施方的工作记录本身：一个 74 小项的处置表加十几份分项复查，对应的产品改动是 +898 行。

---

## 7. 再来一轮同样形式的打磨，会更好维护还是更难维护

会更难维护。三轮的形态是一样的：评审列出几十条"下-N"和小项，实施方逐条落实，每一条都落成"一个新状态或新字段，加 3 到 7 个文件的测试，加一段指引"。连标着删减的两项也是这样：字段表在同一轮里被后续条目绕开，`result_line` 长回 25 行；退出码集中变成空包装。规模数据（本轮主代码 +898、测试 +2,507、指引 +6 KB）和实测数据（新加的指引专题 0 次被读、新的统计字段大多以"已恢复"出现在成功回执上）都说明，新增的维护负担没有换来调用者可观察的收益。代码库里也几乎没有死代码可删了，所以再打磨一轮，剩下的只能是继续加分支。

更好的做法是：停止逐条规则式的评审。改为由真实任务失败驱动，一个问题一个提交。删减按带数字的目标、限时做，并且计入总行数，而不只计某个函数的行数。

---

## 需要用户定的问题

**Q1：`open` 的意图参数族要不要保留？**

- 背景：约 430 行主代码加对应测试；48 次尝试里只用过 `open --action`（4 次），其余意图参数从未用过；`shell am start` 能覆盖。
- A：保留现状，冻结，不再加规则。好处是不损失能力；代价是持续的测试维护。
- B：只保留 `--url` 和 `--action`，其余参数删掉，指引改为教 `shell am start`。好处是约 −400 到 −900 行；代价是调用者要自己写 am 命令，回执里也没有启动确认。
- C：全部删掉，只留包名启动和 `--url`。收益最大，失去的能力也最多。
- 推荐：A，并且只在下一次改到它时考虑 B。

**Q2：首行里给评审统计用的字段怎么处理？**（`rootUnavailableObserved`／`rootRecoveredWithinRequest`；新-5 本轮要由用户再定）

- A：保留现状。
- B：成功并且已恢复时不在首行显示。
- C：整个删掉，包括 sticky 状态和测试。
- 推荐：B；如果新-5 选了 A（不放宽重试），就选 C。

**Q3：原因码表的"下一步"文字怎么办？**

- A：接进拒绝时的首行输出。好处是调用者能直接看到下一步；代价是首行变长，每个码的文字都要按调用者能读懂的标准审一遍。
- B：删掉文字，表只留码和类别。好处是少维护一份 24 KB 的文字；代价是维护者失去一份"每个码该怎么处理"的说明。
- 推荐：B。

## 核对过、没有问题的地方

- 主评审给的四版本粗数字：全部一致。
- 死函数：没有发现（见 3.1）。
- 文字归一化两端漂移：已经有共享样本。
- 删掉 Maps 专用字符串：确认已删（`git grep "正在查找\|best route"` 无结果）。
- docs 仓库 ≤150 KB：确认 149,809 字节。
- 实施方复查里说"sticky 状态各有消费者"：抽查源码，说法成立。
- 主评审给的四版本粗数字之外，`result_line` ≤120 行的目标：确认 113 行（但见第 1、5 节对它实际效果的保留）。
