# 第六轮专题 70：主机端的命令入口、结果、超时、附件交付、保存与打印、退出码，以及 shell／文件／daemon 的回归

评审对象：产品 `fb75afc`（主机端 `androidctl/src/androidctl/`、公开指引 `skills/androidctl/`），对照基线 `7d5be70` 和第五轮候选 `cab84b8`；独立 docs `66bb380a`。
工作目录：`S/agents9/70/`（`S` 为评审 scratchpad 根目录）。我写的脚本和原始输出都在这里：`codes.py`／`codes.json`（原因码计数）、`cli_stats.py`／`rows.json`／`firstline.py`／`reads.py`（真实任务原件统计）、`probe/probe.py`（替换设备调用、在真实 `app.main()` 里注入中断和保存失败）、`local-errors.txt`／`local-errors2.txt`／`probe-results.txt`（实跑输出原文）。
主机实跑用 `S/r6/venv6`（`fb75afc`），对照 `S/r5/venv5`（`cab84b8`）和 `S/hostcheck/venv-base`（`7d5be70`）；没有设备。

## 1. 结论摘要

整体状态：主机端这一块**可以随候选合入**。结果模型的主干（四个主结果词 done／failed／not sent／UNKNOWN，加 shell／传输专用的 still running）稳定、可预期；第五轮裁决里与这一块有关的三件事（删除 batch、保存规则只看交付方式、原因码合并）都已落实，我用脚本和实跑逐项核过。剩下的问题都是小修或简化，不构成合入障碍。

最重要的发现：

1. **原因码合并按第 15 题的清单逐个落实了**【重算】。两端去重后的码从 165 个减到 144 个（设备端 `Rejected` 123→106，主机端 `Fault` 62→56），连同在别处构造的码从 174 个减到 153 个，与决定文件估算的"约 153"一致。20 个纯参数码全部消失；4 个"部分符合"的码只保留依赖状态的用法；`staleRef`／`staleImage` 合成 `stale`，`containerChanged`／`selectionChanged` 合成 `changed`，21 处构造全部带了 `subject`；决定文件点名的三处消费者（`UiRun.kt:129`、`UiRefRepair.kt:50-52`、`images.md`）都已改成读新字段。
2. **第 15 题后半句"过程计数从默认文本去掉"只做了一半**（必须修，很小）【原件】【源码】。成功的手势已经变短，但成功的 `back`／`home`／`recents`／`key` 仍在默认文本里打印完整的逐事件进度 `keys={"plannedEvents":2,"attempted":2,…}`（每次约 414 字节；20 次计分的候选运行里有 17 次），成功的 `scroll-until` 仍打印 `completedSteps=N` 和整份 `step={…}` 回执（12 次）。公开指引 `reference.md:81-82` 却写着"普通文本会简写已知成功的派发和进度"。
3. **"已知结果不被本地失败吞掉"在 UI 和非 UI 路径上基本成立**【实跑】。我在真实 `app.main()` 里注入了真实 SIGINT 和保存失败：附件读取中、关闭连接中、保存中被中断时，`click: done` 和已存路径都保留下来。仍然会丢的只有"设备回执还没到就被中断"这一种（按设计报 UNKNOWN），但命令名会变成 `androidctl`，与指引"顶层 command 是固定命令路径"不符；打印中途被中断时，结果会被完整地再打印一遍，与独立 docs 的"已打印结果不重印"不符。两处都是低严重度。
4. **结果语义比实际使用需要的精细，但真正有保护价值的部分值得保留**【原件】【推断】。20 次计分的候选运行共发出 297 次 CLI 调用，非零退出只有 10 次，全是设备端如实的 not sent（8 次）或 failed（2 次）；没有一次 UNKNOWN、交付失败、保存失败或连接清理失败。也就是说，"主回执与退出码分责"那一整套规则在普通任务里从未被触发。其中"not sent（可以安全重试）"和"UNKNOWN（不要重做）"的区分是手机上防止重复下单、重复发送的真实保护，应当保留；"本地连接关闭未确认就把任何结果的退出码改成 3"这类边角规则可以收掉。
5. **保存与打印对调用者几乎没有负担**【原件】。候选运行里模型从没用过 `--raw`、`--no-print`、`--print`、`--out`、`--json`；默认文本打印并保存同一份正文。模型读保存文件只有 1 次，原因是主机工具把一次 `snapshot --all`（约 4.5 万 token）截断了，模型随后用 `Select-String` 在保存的文件里搜索。这正是自动保存存在的理由。
6. **shell／push／pull／job／apk／apps／pair／device／daemon 没有发现误伤**【源码】【实跑】。与基线相比的行为变化都是有意的：参数错误改成 `arguments` 加字段并给出可照抄的写法；中断和打印失败时保留已知结果；`daemon stop` 确认阶段保留原数据。相关的 5 个测试文件我在 Linux 上跑过：425 通过、1 跳过。

这一块是否构成合入障碍：**不构成。** 第 2 条应在合入前顺手修；其余都可以合入后再做。

## 2. 确定缺陷

### 2.1 第 15 题"成功的过程计数不进默认文本"只落实到手势（必须修，按裁决；改动约 5 行；有原件，确定）

- 触发条件：任何按键类动作（`back`、`home`、`recents`、`key`）成功；`scroll-until` 成功（`stop=found`）。
- 源码：`ui_output.py:24-27` 只把 `tap/swipe/drag/pinch/timeline` 认作"简写的手势"；`ui_output.py:463-476` 的 `input_facts` 在其它命令上照样输出 `keys`；`ui_output.py:155-168` 对 `scroll-until` 不论成败都打印 `completedSteps`、`step`。
- 原件：`ST/efficiency/r5-v1-candidate-02/evidence/cli/` 下 `back` 的 stdout 首行是
  `back: done; Platform key dispatch returned; …; keys={"plannedEvents":2,"attempted":2,"returned":2,"successfulPrefix":[0,1],"syncReturned":2,"lastReturned":{…13 个字段…},"inFlight":null,"failure":null,"stop":"completed","possibleDown":[],"cleanup":[],"omitted":{…}}; snapshotPath=…`。
  我统计了 20 次计分候选运行（GitHub、Feeder、后台屏 Contacts、Maps 各 5 次）：17 次成功的 `back` 都带 `keys={…}`，合计 7,038 字节；12 次成功的 `scroll-until` 都带 `completedSteps` 和 `step={…}`，合计约 2,400 字节。
- 影响：成本不大（每次几百字节），但它直接违背第 15 题和第五轮发现 B12 的修法（"进度只在 `--json` 或未完成时打印"），也和 `reference.md:81-82` 的说法矛盾。实施方交付说明 `T/deliverables/r5-candidate.md:54` 写"成功过程计数默认省略"，只对手势成立。
- 最小修正：命令结果为 done 时，`input_facts` 不输出 `keys`、`inputTarget`、`point`、`gesture`（不再限定手势命令）；`scroll-until` 在 `stop=found` 时不输出 `step`（`completedSteps` 可留可去，按裁决应去掉）。

### 2.2 回执到达之前被中断：命令名丢失，"什么都没发出"也报 UNKNOWN（可选；实跑，确定）

- 触发条件：命令在拿到设备回执之前收到 Ctrl‑C，包括还在连接本地 daemon、还没选到设备的时候。
- 源码：`app.py:1014-1029` 把这类情况统一构造成 `Result(command="androidctl", result="UNKNOWN")`，`data` 为空。
- 实跑（`probe-results.txt`）：`click e15` 在请求阶段收到真实 SIGINT，文本输出 `androidctl: UNKNOWN; Command interrupted; execution outcome is unknown`，JSON 是 `{"command":"androidctl","result":"UNKNOWN",…,"data":{}}`，退出 3。
- 影响：处理方向保守，不会诱发重复操作；但 `reference.md:89` 写着顶层 `command` 是"固定的命令路径"，这里不符；在连接之前就按下 Ctrl‑C 时实际什么都没发出，也被报成"结果未知"。这是第五轮发现 B10 的后半句（"还没返回才报 UNKNOWN，并保留命令名"），这轮只做了前半句。
- 最小修正：`CliCommand.invoke` 捕获 `KeyboardInterrupt` 时用 `command_name(ctx)`；传输层已有 `before_send` 钩子（`transport.py:400-401`），可以用它记一个"已经发出"的标志，没发出时报 not sent。约 15 行。

### 2.3 打印中途被中断时，结果会被再打印一遍（低；实跑，确定）

- 触发条件：结果行已经写到 stdout，打印正文时收到 Ctrl‑C 或发生其它本地异常。
- 源码：`ui_client.py:1066-1069` 把异常交给 `CommandFailure`；`app.py:1030-1037` 再用 `render_fields` 把整个 Result 按字段打印一遍。
- 实跑：在第二次输出时注入 `KeyboardInterrupt`，stdout 先是 `click: done; snapshotPath=…`，接着又是 `click: done; …` 和十几行 `key=value`，最后是 `delivery: done; KeyboardInterrupt`，退出 1。
- 影响：信息是对的，只是重复；但独立 docs `reference/README.md:63` 写着"已打印结果不重印"。
- 最小修正：二选一——改代码（`print_delivery` 记下结果行是否已经打出，已打出就只补一行交付错误），或者改 docs 那一句。

### 2.4 主机端自己生成的 not sent 后面跟着 `dispatchStarted=unknown`（低；实跑，确定）

- 触发条件：在发出请求之前由主机拒绝，例如没有配对设备（`deviceRequired`）、连接失败。
- 源码：`ui_output.py:69-75`：只要结果不是 done、回执里又没有 `dispatchStarted`，就打印 `dispatchStarted=unknown`。
- 实跑：`androidctl click @e15`（无设备）输出 `click: not sent; No paired devices; …; code=deviceRequired; dispatchStarted=unknown; …`。
- 影响：not sent 的定义就是"没有派发"，后面却写"派发与否未知"，自相矛盾，会让模型多想一步。结果为 not sent 时不打印这个字段即可。

## 3. 设计与简化

### 3.1 结果模型讲清楚：从设备回执到退出码

一条 UI 命令的过程（`ui_commands.py:27-42`、`ui_client.py:157-264, 643-1069`）：

1. **构造与本地校验**：回调按 Typer 参数构造 `UiInvocation`；`prepare_ui` 检查参数组合和本地目标路径。这里抛出的错误都是 `not sent`，退出 2，此时没有连接。
2. **请求**：`UiClient.request` 只发一次请求。传输层没收到回执时报 `UNKNOWN`（`TransportFailure`）；被拒绝时报 `not sent`；其它情况用设备回执。从这一步开始，这个 `Result` 对象就是唯一的主结果。
3. **附件**：按回执里的 `capture` 分块读取树、窗口、图片（`ui.read`）。每个附件单独记录错误（`snapshotError`／`windowsError`／`imageError`），中断只追加 `deliveryError`，主结果不变。
4. **渲染与保存**（`deliver`）：渲染正文，保存显式文件和自动文件（`.androidctl/snapshots`、`.androidctl/shots`）。异常只追加 `deliveryError`。
5. **交付判定**（`finish_delivery_result`）：按"每件交付物以什么方式交付"判断。done 的 `snapshot`／`find` 缺树或缺正文时改成 failed；`screenshot` 缺图时改成 failed；其它命令保留主结果，只把退出码 0 改成 1。
6. **打印**（`print_delivery`）：text 先打一行结果，再打正文；JSON 只打 Result。stdout 写失败时记 `deliveryError`；`snapshot`／`find` 既没存成文件也没完整打印时改成 failed。
7. **关闭连接**：本地连接关闭失败或被中断时记 `localCloseError`，并把退出码设成 3。

**主结果词与退出码**（`protocol.py:66-88`、`reference.md:44-64`）：

| 主结果 | 默认退出码 | 会出现的其它退出码及原因 |
| --- | ---: | --- |
| done | 0 | 1：必需的交付物缺失或本地交付出错（结果词仍是 done）。3：设备端清理未确认（会话关闭、自动化清理、shell 清理）、本地连接关闭未确认、工作区配置写入不确定。 |
| failed | 1 | 3：同上的清理未确认。复合动作（`fill`、`scroll-until`）只要有一步已派发，整体就是 failed，已执行的前缀保留在步骤字段里。 |
| not sent | 2 | 3：本地连接关闭未确认（见 3.3）。 |
| UNKNOWN | 3 | — |
| still running（只用于 shell／传输／daemon stop） | 4 | 3：本地连接关闭未确认。 |

"主回执与整命令退出码分责"的具体规则是：**结果词只描述设备上的主操作；退出码取主操作、每件必需交付物、本地交付、清理这几方中最坏的一个**。实现上分三层：回执里显式给出的 `exit` 优先；否则按结果词映射；最后，如果算出来是 0 但存在 `deliveryError`，就改成 1（`protocol.py:78`），必需交付物缺失也改成 1（`ui_client.py:1024-1032`），本地关闭失败则无条件设成 3（`protocol.py:80-82`）。

partial（采样不完整，`observationWarning="partial; missing=N"`）和动作后的观察缺口一律不影响结果词和退出码（`reference.md:57`）。

**一个模型要读懂一次失败需要的概念**：结果词（5 个）；退出码，以及它可以和结果词不一致；`reason`；`code` 加上 `argument`／`subject`／`stage`；`dispatchStarted`；复合动作的步骤字段（`focus/fill/submit`、`completedSteps/step/stop`）；`change=` 的 5 种状态；`observationWarning`；`snapshotPath`／`imagePath`；7 个交付诊断字段（`snapshotError`、`snapshotWarning`、`snapshotProblem`、`windowsError`、`imageError`、`historyError`、`deliveryError`）；`cleanupError`／`localCloseError`；UNKNOWN 时"不要重做"。大约 12 组概念、25 个字段名。

对照真实使用：20 次计分候选运行（GitHub、Feeder、后台屏 Contacts、Maps 各 5 次；候选版本 `a27fe5aa`，树与 `fb75afc` 相同）里共 297 次 CLI 调用，退出码分布是：0 共 287 次，1 共 2 次，2 共 8 次；没有 3 和 4（`cli_stats.py` 的输出）。8 次退出 2 都来自设备端（5 次目标不能朝该方向滚动、3 次引用已过期），2 次退出 1 都是如实的业务失败（一次 `wait` 采样超过截止、一次 `scroll-until` 滚到尽头）。没有一次 UNKNOWN，没有一次结果级的 `deliveryError`、`historyError`、`snapshotWarning`、`localCloseError`（逐文件 grep）。

### 3.2 与 Playwright CLI 对比：哪些精细是必要的

Playwright 的 CLI 成功退 0、失败退 1。AndroidCtl 多出来的部分，我分成三档：

- **应当保留**：not sent（退出 2，可以安全重试）与 UNKNOWN（退出 3，不要重做）的区分。手机上的动作不幂等（发消息、下单），回执又要经过 Wi‑Fi 和 Binder，"发没发出去"是调用者真正需要知道的；真实运行里 8 次 not sent 中有 5 次附带了可以照抄的修正命令（`use androidctl scroll e… forward`），另 3 次是引用过期，提示重新观察。复合动作的"已执行前缀"同理。
- **可以保留但不要再扩**：done 加退出 1（动作已发生、必需交付物缺失）。按第 6 题第三处的裁决，这条规则已经收成"只看交付方式"，实现也干净（第五轮按失败原因分叉的权限特例已删除，`ui_client.py:952-1032` 里不再有 `permissions_failed`）。
- **应当收掉**：(a) 本地连接关闭未确认时，无条件把退出码设成 3（3.3）；(b) 一次保存失败同时写三个字段：我实跑 `.androidctl/snapshots` 是普通文件的情形，text 结果行同时出现 `snapshotError="Cache path is not a directory…"`、`snapshotWarning="Warning: automatic observation file was not saved…"`、`historyError="Cache path is not a directory…"`，同一件事说了三遍；(c) 正文里的附件状态行用的也是 `deliveryError` 这个名字（例：`artifact kind="tree" state="unavailable" code="scopeUnknown" deliveryError="One complete task window is required…"`，原件 `ST/efficiency/r5-s1-core-candidate-01/evidence/cli/1795495835964878200-…/child.stdout`），可它表示的是设备端没给出树，和 Result 级"本地交付失败、退出码升为 1"的 `deliveryError` 不是一回事，而且这次退出码是 0。同名不同义，容易误读。

能力得失：收掉这三处不损失任何保护，交付诊断只剩"哪件交付物缺了、为什么"一种写法；维护上可以删掉 `finish_delivery_result` 里一半的布尔量（现在有 12 个）。

### 3.3 本地连接关闭未确认就把退出码设成 3：保护不了什么（建议改成警告；推断，中等把握）

- 源码：`protocol.py:80-82` 的 `local_close_failed` 无条件设 `exit = 3`；四处调用点（`app.py:383`、`job_commands.py:418, 499`、`ui_client.py:661`）。
- 实跑：`click e15` 在关闭阶段收到 SIGINT，输出 `click: done; …; local connection cleanup unconfirmed; inspect with androidctl daemon status; localCloseError="CancelledError"`，退出 3；被设备拒绝的 not sent 再遇到关闭失败，也退出 3（`probe-results.txt` 的 `rejected` 场景）。
- 分析：这里关闭的是 CLI 到本机 daemon 的 WebSocket（`transport.py:54-71`，超时或异常时直接 `abort()`）。CLI 进程随后就退出，操作系统会关掉这个套接字，daemon 能看到断开。我在主机代码里没找到一种"本地关闭没确认，所以设备或 daemon 状态变得不确定"的路径【推断】。把 not sent 变成 3、把 still running（意思是"去查原任务 ID"）变成 3，反而丢了更有用的信号。
- 建议：保留 `localCloseError` 作为一行提示，不改退出码。如果实施方能举出关闭不完整会让 daemon 一直占着会话的具体路径，就保留现状，但至少不要覆盖 2 和 4。

### 3.4 实施方第五轮后的"已知结果保留"工作：前一半值得，后一半有钻牛角尖的迹象

- 第五轮评审的发现 B10（`sub-reports/52` §2.3）是"单条 UI 命令拿到结果后被 Ctrl‑C，整条变成 UNKNOWN"，我们当时定为低严重度，估计修法约 20 行。
- 这轮实施方先在 UI 路径修了（`ST/r5-delivery.md`），又把同一规则推广到全部非 UI 出口（`ST/u33-known-results.md`）：`job get --wait` 等待中被中断、`job output` 导出后关闭被中断、`daemon stop` 确认阶段被中断，乃至确认用的文件锁关闭失败也单独记 `cleanupError` 并设退出 3（`local_call.py:202-268`）。对应的新测试有 `test_non_ui_delivery.py` 452 行，`test_ui_execution.py` 增加 445 行，另有多轮独立 review、修前和修后日志、Linux 复跑。
- 判断：UI 那一半是真问题，修得对。非 UI 那一半里，`job get --wait` 中断后保留最新回执有实际价值；"daemon stop 确认锁关闭失败"这类情形在真实使用里几乎不会出现，却引来了专门的结果分支、退出码规则和测试。这正是"字面贯彻'已知结果只有一个拥有者'"导致在低价值处大量投入的例子；源头一半在第五轮评审把 B10 写成一条普遍规则，一半在 `reference.md:45` 那句"已知事实在交付、中断或清理失败后都保留"被当成必须处处证明的契约。
- 建议：**不回退**（代码已经写完，行为保守、无害），但冻结这条规则，不再为新的边角情形补分支；以后指引里写成"尽量保留"即可，不必当成需要逐出口证明的硬契约。

### 3.5 实施方说"相似代码保护的是不同结果拥有者"——部分属实

- 关闭连接的 finally 块（`failure = sys.exception()` → 尝试关闭 → 有原异常就 `add_note`，否则 `local_close_failed`）逐字重复了 4 次：`app.py:375-383`、`job_commands.py:410-418, 491-499`、`ui_client.py:653-661`。打印出口的 try／except → `delivery_failed` → `attach` → `CommandFailure` 重复了 3 次：`app.py:344-353`、`job_commands.py:216-328`、`ui_client.py:1045-1069`。错误码加 `argument/subject/stage` 的格式化也写了 3 份：`output.py:234-241`、`app.py:204-209`、`ui_output.py:51-56`。
- "拥有者不同"是事实：每处持有的 Result 来源确实不同。但样板代码完全相同，抽一个以"取当前结果"回调为参数的小函数，并不会把不同的拥有者合并起来。所以这条理由只能说明为什么不能共用一个全局"最后结果"，说明不了为什么要复制 4 遍。合并后大约少 40 行，属于可选整理。

### 3.6 `prepare_ui` 里 batch 删除后已经走不到的校验

- `prepare_ui`（`ui_client.py:405-640`，236 行）里有一批检查是为 batch 的 JSON 计划写的：`select`／`range`／`action`／`reveal` 要求"显式 REF、不带选择器"，`select` 的端点是 32 位整数，截图区域的参数键集合等。现在 CLI 的 Typer 签名已经保证了这些（例：`ui_commands.py:900-903` 的 `select_text` 只接受 REF 和两个非负整数，根本没有选择器参数）。
- 实施方在 `ST/u33-host-chain.md` 里承认这些分支"只能由手构 UiInvocation 触发"，但以"有的还混着真实规则"为由不删。我同意不能整块删（`range` 的 Float 可表示性、`action` 选项唯一性、区域与 REF 互斥都是 CLI 签名表达不了的真实规则），但能删的部分大约 60–80 行。可选。

### 3.7 `snapshot --raw` 在默认 text 模式下把整份原始 JSON 打到 stdout（可选）

- 实跑：`snapshot --raw` 打印结果行后，紧跟着打印整个 `{"windows":…,"tree":…}` JSON，同时保存为 `.json` 文件。源码：`print_delivery` 只看 `print_body`，不区分 raw（`ui_client.py:1053-1055`）。
- 原始树的上限是 8 MiB（`protocol.py:19`）。`--raw` 是排查用的选项，一不小心就会占满模型的上下文。建议 `--raw` 默认只保存不打印（`--print` 可以显式打开），并同步 `observations.md:34-37`。候选运行里没人用过 `--raw`，所以这条目前没有实际代价。

### 3.8 超时与截止

- 时长语法（`cli_options.py:148-160`）：整数加 `ms/s/m/h`，只有 `0` 可以不带单位；小数、负数、不带单位都拒绝。各参数的范围：`--timeout` 1ms–60s，`--settle` 和 `job get --wait` 0–60s，`--run` 1ms–10m，`--idle` 是 1s–24h 的整秒，手势时长 50ms–5s。
- 截止只有一个：主机截止时间等于 `--timeout`（`ui_client.py:644`），本机 daemon 和设备拿到的都是剩余时间（`transport.py:464-476`）。设备端在这个时间里给附件读取预留了时间（`UiBudget.kt:19-39`：主机留 200ms，每个 64 KiB 块留 50ms，收尾留 100ms）。这一轮仍用手写的采集路线，所以第四轮"换成公开路线后，主机多等设备最多约 5 秒"的宽限没有实现，也不需要。
- "设备超时但已经有部分结果"的完整路径：设备返回 failed，附带 `completedSteps`／`step`／`predicate`；主机读附件时如果预算用完，只记 `snapshotError`（`ui_client.py:236-257`），主结果和步骤事实保留；保存前预算用完，只记对应文件的错误（`ui_client.py:888-891`）；打印照常。**只要回执到了主机，部分结果不会在后续任何一步丢失**【源码】【实跑】。
- 仍会丢的路径：(1) 回执没在截止前到达主机（App 卡住、同步 Binder 调用超时、网络断开），这时主机报 UNKNOWN，设备那边的部分结果收不到——这是协议本身的限制，不是缺陷；(2) 回执到达前被 Ctrl‑C（2.2）；(3) 主机进程被 SIGKILL 或 SIGTERM 直接杀掉，不会有任何输出，已经写了一半的文件由 `user_files.create`（`user_files.py:32-56`）在 Python 能处理异常时删除，SIGKILL 时会留下半个文件——不可避免，也不值得处理；(4) 附件读到一半被中断：已经读到的树不会保存（`ui_client.py:187-192` 直接返回空观察），这是实施方明确的取舍（`ST/u33-host-chain.md`"不承诺继续保存已读而未落盘的其它附件"），我同意；(5) 保存目录不可写：text 模式下只是警告，`--json`／`--no-print` 下退出 1，`snapshot` 报 failed（3.9 的实跑表）；(6) stdout 管道关闭：`Output.echo` 吞掉 `OSError` 记进 `deliveryError`，退出 1；`snapshot`／`find` 如果正文没存成文件又没打印完整，改报 failed（`ui_client.py:1056-1064`）。

### 3.9 保存与打印：调用者需要知道的规则

实跑（`probe-results.txt`，自动保存目录 `.androidctl/snapshots` 被换成一个普通文件）：

| 调用 | 结果词／退出码 | 说明 |
| --- | --- | --- |
| `click e15`（text） | done／0 | 打印了正文，保存失败只是警告 |
| `--json click e15` | done／1 | 正文只能通过文件交付，文件缺失时退出 1 |
| `snapshot`（text） | done／0 | 打印的正文代替了文件 |
| `--json snapshot` | failed／1 | 主交付物缺失 |
| `snapshot --no-print` | failed／1 | 同上 |
| `click e15`，保存阶段收到 Ctrl‑C | done／1 | `snapshotWarning` 加 `deliveryError=KeyboardInterrupt` |

与第 6 题第三处的裁决逐条一致。

调用者真正需要知道的规则只有三条：默认 text 打印并保存同一份正文，路径在 `snapshotPath`；主机工具截断了输出时，去读这个文件；文件是缓存，最多 150 个文件或 20 MiB，超出后从最旧的开始回收（`history.py:17-18`），长期要用的要复制出去。其余规则（`--json`／`--no-print`／`--out`／`--raw` 的交付要求）只在调用者主动用这些选项时才有关。全部 22 次候选运行（含两次未计分的历史运行）里这些选项的使用次数都是 0；用到的只有 `--all` 14 次、`--cached` 13 次、`--timeout` 13 次、`--screenshot` 12 次。

模型读保存文件：在 20 次计分候选运行的 317 次模型工具调用里只有 1 次（`ST/efficiency/r5-c2-final-candidate-01/native.rollout.jsonl`，第 37–42 条），起因是 `snapshot --cached --all` 输出了约 4.5 万 token，被主机工具截断（工具返回里写着 "Warning: truncated output (original token count: 45044)"），模型随后用 `Select-String` 在 `.androidctl/snapshots/000004*` 里搜索。

### 3.10 主机端现在是否"薄"

不算薄：主机源码从基线的 8,008 行增长到 11,495 行（第五轮候选是 11,220 行，删除 batch 的 334 行被交付修正和非 UI 已知结果抵消了），测试从 16,775 行增长到 25,507 行。增长主要来自观察正文的投影和渲染（`ui_view.py` 1,020 行、`ui_output.py` 530 行）、参数准备（`ui_commands.py` 1,670 行）和交付判定，这些放在主机端是合理的。这一轮 `ui_client.py` 所谓的"大改"，主要是把结果行渲染原样搬进新文件 `ui_output.py`（第五轮的 `ui_client.py` 有 1,497 行，现在 1,078 加 530 行），职责分得更清楚了，不算重写。

可以直接删或合并的层：3.5 的样板（约 40 行）、3.6 的死分支（约 60–80 行）、3.2(b)(c) 的重复诊断字段。没有哪一整层应当重做。

### 3.11 shell／push／pull／job／apk／apps／pair／device／daemon 的回归

我对比了 `7d5be70` 与 `fb75afc` 的 `job_commands.py`、`file_jobs.py`、`user_files.py`、`identity.py`、`local_call.py`、`output.py`、`private_files.py`、`workspace.py`、`transport.py`，以及 `app.py` 中非 UI 的部分：

| 变化 | 判断 |
| --- | --- |
| 参数类错误码改成 `arguments` 加 `argument`（`endpoint`、`stream`、`script`、`filePath`、`scriptFile` 中与内容有关的用法、daemon 内部的 `direction`／`runMs`），报错补了可照抄的写法 | 有意，按第 15 题；读脚本文件失败时仍是 `scriptFile`，符合"依赖状态的用法保留原码" |
| `job get --wait` 中断时保留最新回执；`shell` 等待中断后不再读输出；导出成功后关闭失败保留导出结果 | 有意，见 3.4 |
| `emit`／`show` 打印失败时通过 `CommandFailure` 交付原结果 | 有意；副作用是可能重印（2.3） |
| `daemon stop` 确认阶段保留原数据和显式退出码，中断时报 UNKNOWN 而不是 `androidctl: UNKNOWN` | 有意 |
| `Output.attach` 合并已有的 `deliveryError`，不再覆盖 | 有意，修正了早先后来的管道错误覆盖原错误的问题 |
| `private_directory(repair_existing=True)`：把权限过宽的已有缓存目录收紧一次 | 有意，第四轮第 14 项 |
| `workspace.read` 把 `NotADirectoryError` 当作"没有配置" | 有意，小修 |
| `session open` 新增 `--display`（必填）、`--replace`、`--return-tasks`／`--discard-tasks`，`--idle` 默认值从 10m 改成 30m | 有意，属于会话专题 |
| 新增 `help` 命令（离线读指引） | 有意 |

没有发现误伤：输出格式（`show` 函数体只是缩进了一层）、超时、原子发布、清理和后台任务的逻辑都没变。测试方面，`test_non_ui_delivery.py`（新增）、`test_jobs.py`、`test_result_output.py`、`test_observation_files.py`、`test_ui_execution.py` 我在 Linux 上用 `S/r6/venv6` 跑过：**425 passed、1 skipped，16 秒**【实跑】。这些测试用真实的 CLI 子进程和真实的 SIGINT 走完 Python 的取消流程，验证的是外部可见的行为，不是在复述实现。

## 4. 证据审计

| 实施方的声称 | 核对结果 |
| --- | --- |
| 第 15 题原因码按范围合并（`ST/r5-reasons.md`） | **成立**【重算】。`codes.py` 显示：20 个纯参数码在两端全部消失；`deviceName` 只剩 `state.py` 的重名用法，`point` 只剩越界用法（`PointerPlan.kt:52`、`AppInput.kt:155`），`region` 只剩越界用法（`ImageRegion.kt:189`），`scriptFile` 只剩读取失败的用法（`job_commands.py:79`）；`stale`／`changed` 的 21 处构造全部带 `subject`（ref 5、image 8、container 4、selection 4）。新增的 6 个码（`externalTasks`、`taskReturn`、`unexpectedTaskMove`、`collectionLimit`，以及合并出来的 `stale`、`changed`）都来自其它题。另外：`arguments` 共有 216 处，其中 72 处带 `argument` 字段、144 处不带——按决定文件"现有其它 arguments 不作全局补字段"，这符合裁决，但调用者不能指望 `argument` 字段总是存在。 |
| 成功过程计数默认省略（`T/deliverables/r5-candidate.md:54`） | **部分成立**：只对 5 种手势成立；按键类和 `scroll-until` 没做（2.1）。 |
| 保存规则只看交付方式（`ST/r5-delivery.md` 第一行） | **成立**【实跑】（3.9 的表）；权限特例已删除。 |
| UI 和非 UI 都保留已知结果（`ST/r5-delivery.md`、`ST/u33-known-results.md`） | **基本成立**【实跑】：附件读取、关闭连接、保存、打印各个阶段被中断时都保留了 done；例外见 2.2、2.3。另外，实施方所说的"实际 SIGINT"测试我在 Linux 上复跑通过。 |
| "相似代码保护不同结果拥有者"（`ST/u33-host-chain.md`） | **部分成立**（3.5）。 |
| 时长不带单位仍然拒绝，报错给出可照抄的写法（第 4 题） | **成立**【实跑】：`wait --timeout 30` 报 `Use an integer with ms, s, m or h; add a unit, for example 1s. Only 0 may omit the unit; inspect accepted syntax with: androidctl wait --help`，退出 2（第五轮候选的报错没有示例，也没有指向 `--help`）。第 4 题要求报告这类被拒的次数：20 次计分候选运行里是 **0 次**（第五轮评审统计的是 10 次）。`--timeout 1.5s` 的报错同样写着"add a unit"，对小数输入来说有点误导，影响很小。 |
| 接受大小写不同的按键名和带 `@` 的引用，`page-*` 是正式方向（第 4 题） | **成立**【实跑】：`key ENTER`、`key Ctrl+a`、`click @e15` 都通过了本地校验，走到"没有配对设备"那一步；`key A` 被拒，并给出 `androidctl key a or androidctl key shift+a`；`scroll e1 sideways` 的报错列出全部 10 个方向；`click @` 报 `Supply a current REF, for example: androidctl click e15`；`--return-tasks --discard-tasks` 一起用时报互斥。第五轮候选在这几处都拒绝，或者报错里没有示例。 |
| `--json` 下错误的形状 | **成立**【实跑】：参数错误同样输出单行 JSON `{"command","result":"not sent","reason","data":{},"error":{"code":"arguments","message"}}`，退出 2。|
| 交付与已知结果相关的检查都在 Windows 上跑过，另有 Linux 复跑 | Windows 日志未能核实（没有执行包内任何脚本）；Linux 侧我自己复跑了 5 个测试文件，见 3.11。 |
| 帮助与指引一致 | **基本成立**【实跑】：`--help` 列出的命令、`scroll` 的 10 个方向、`--settle`、`--timeout` 的范围都与 `reference.md` 一致。小出入：`snapshot --raw` 的帮助说"Save separate artifacts and the whole collected tree as JSON"，实际保存的是单个 JSON 对象，而且会打印（3.7）；缺少必填参数时 `session open` 的报错里出现了字面的 `\n\t`（`Choose from:\n\tmain,\n\tvirtual`）。 |

关于如实保留失败：计分候选运行里的 2 次 failed 和 8 次 not sent，stdout 原文与 `cli-index.json` 记录的 `childExit` 全部一致（`cli_stats.py` 交叉核对过）。

## 5. 需要用户裁决的新取舍

**第 1 题：边角情形的退出码要不要收紧**（涉及公开契约，所以交给用户定）

背景：现在一个结果可以是"done 但退出 3"或"not sent 但退出 3"，原因只是 CLI 到本机 daemon 的连接没确认关好；保存失败会同时写三个诊断字段。真实运行里这些都没出现过，但它们在指引里各占规则（`reference.md:52-64`、`images.md:78-100`）。

- A：维持现状。
- B（推荐）：本地连接关闭未确认只给一行提示，不改退出码；一件交付物失败只写一个诊断字段（例如 `snapshotError`），删掉 `historyError`／`snapshotWarning` 的重复；附件状态行里的 `deliveryError` 改个名字，和 Result 级的那个区分开。主结果词和 0／1／2／3／4 的含义都不变。得：指引少几条规则，`finish_delivery_result` 能简化一半；失：没有我能找到的真实保护。剩余未知：daemon 在连接未正常关闭时是否会占着会话（3.3，我从代码推断不会）。
- C：进一步向 Playwright 靠拢，只留 0／1，结果词放在文本里。失去"not sent 可以安全重试、UNKNOWN 不要重做"的区分；不推荐。

其余各条（2.1–2.4、3.5–3.7）都属于实施细节，实施方自己可以定。

## 6. 有判别价值的补验

1. **本地连接没关好时，daemon 是否会持续占着会话**：在 `close()` 前用 SIGKILL 杀掉 CLI 进程，随后立刻用同一会话发 `status` 和一条动作。如果 daemon 报会话忙、或者要等很久，3.3 的建议就不成立，应保留退出 3；如果立即可用，就按第 5 节 B 收掉。只需要本机 daemon 加一台模拟器。
2. **按键类回执简写之后的效果**：不需要单独补验；2.1 修完以后，在下一次固定任务里顺带看 `back` 的结果行长度即可。

## 7. 没能核实的内容

- 设备端的超时边界（`UiBudget` 预留时间在慢设备、慢网络下够不够，回执晚于主机截止的实际频率）：需要设备，未验证。
- 实施方在 Windows 上的测试日志、Gradle 结果、JVM 测试：只读了说明，没有执行包内任何脚本。
- Windows 上的路径显示（`display_path` 跨盘时显示绝对路径）和私有目录 ACL 收紧：只能读源码，没有在 Windows 上运行。
- 独立 docs 的其它章节和 `jobs-files` 公开指引里关于发布边界的长段落，我只核了与结果、退出码有关的句子；它们的篇幅（例如 `jobs-files.md:74-96`）对 LLM 调用者是否过重，留给指引专题判断。
