# 专题 05：主机 CLI、阅读视图、diff、batch、公开指引与 LLM 使用面

评审范围：CURRENT_SCOPE 方向 6、7、8、9、10、34、39，以及 40 的 host 侧。候选 `3f45af6`，基线 `7d5be70`。
源码行号以 REPO（`review-copy`）相对路径给出。证据类别标注为【实测】【源码推断】【设计建议】，发现类型标注为【确定缺陷】【设计取舍】【证据缺口】。

自做的工作：从 `git archive` 副本做了非 editable 安装（候选 `agents/venv05`，基线 `agents/venv05b`），用 typer 命令树统计命令和选项，离线跑了参数预检和 batch 预检的拒绝路径（没有连设备）。对四次成对真实任务（`stages/final-verification/paired-runtime/{contacts-1,contacts-2,chrome-1c,chrome-2}`）的 CLI 原始 stdout 和 native rollout 做了逐行分类，并用 tiktoken `o200k_base` 计数。脚本在 `agents/w05/scripts/`，都用 `python -I` 运行，没有执行包内任何脚本。

---

## 1. 结论摘要

1. **【实测，最重要】候选的默认阅读输出在真实任务里比基线更贵，不是更省。** Contacts 成对任务（同一 App、同一流程、各 16 条 CLI）：候选 CLI stdout 为 **44,077 o200k tokens**，基线为 **9,775**，约 **4.5 倍**。原生 input 从 812,987 增到 1,461,258（+80%），wall 从 171s 增到 207s。拆分来源：
   - 每个可交互节点默认打印完整 `actions` 对象数组（`{"id","name","label":null}` 逐项），占候选 stdout 字节的 **56%**（83.8KB/149.3KB）。
   - 默认逐条打印近期 `event`，占 9%。
   - 结果首行展开 `effect`、`windows/selection/content`、四个 `baseline*` 字段，从 1.7k tokens 涨到 4.4k。
   - 每次还打印非选中的系统窗口、重复的 `windowsSource`、`artifact {"state":"not requested"}`。

   交付方宣称的“行格式约省 11.5%”只描述**同一字段集**下的语法差异。我按同口径复算，行语法本身确实省约 15–25%。但后来别的方向（`af36a32` 动作、`0e583db` 近期事件、16–19 结果维度）往默认正文里加的字段远大于这点收益。
2. **【实测】默认打印没有字符上限，被宿主截断后 agent 还会重读。** Chrome 候选首个 `snapshot` 有 11,694 tokens，被 harness 截断（`max_output_tokens:10000`）。agent 随即重读保存文件，又被截断（11,649 tokens），之后才改用 `--depth 1`。浪费了约 23k tokens 和两轮往返。同一页 70% 的字节是 actions 数组。
3. **方向 7（行格式、删 XML 出口）：保留，但默认投影必须重新裁决，而且应比交付方的“actions-A / events-A”更激进。** 按我实测的投影变体：
   - 只做交付方推荐的 A（省 7 个普通动作）：44.1k→31.7k tokens。
   - 再去掉事件明细：29.4k。
   - actions 只保留非隐含能力的短名（custom、带 label、expand/dismiss 等），并去掉事件明细：**16.0k**。
   - 再折叠系统窗口、去掉重复 `windowsSource` 和 `not requested`、压缩结果首行，可接近或低于基线。

   这三个“未批准选项”不是可选优化，而是修复回退。
4. **方向 8（显式 diff 和唯一阅读基线）：建议删除，或替换为无状态的“动作后未变化则不打印正文”。** 依据如下：
   - 四次真实业务里 agent 一次都没用 `--diff`。
   - 研究自己的离线语料估计，保守 diff 只比紧凑行全量低 **11.28%**（`research/cli/report.md:51`），还比不上默认 actions 精简一项。
   - 页面导航会退化为 full（111 条变化 6,064 tokens，全量只有 912 tokens）。
   - 为此付出的代价：每次 `--diff` 持久化一份完整 canonical 树（真实 4.9MB，8192 节点可到 8MiB），外加 workspace 锁、配额、stdout 后提交、maxChars 上下文，以及与 batch 的交叉规则。合计约 550 行产品码、约 900 行测试、约 1,000 词公开文稿。
5. **方向 9–10（静态 batch、全局选项按命令边界）：两项都保留。batch 去掉与 diff 的耦合后保持现状。** batch 复用真实 parser 的设计是对的，代码量适中（417 行）。它对 LLM 的真实价值在于减少**模型回合**：候选每回合约 6 万 tokens 的累计上下文，远大于附件字节；但这一点没有测过，四次真实任务也没有自然使用。全局选项后置是对 LLM 常见误用的低成本兼容，保留。
6. **方向 6（UI 执行返回 Result）：保留。** 深层 `typer.Exit` 已从 ui_client 消除，只在命令边界退出（`ui_commands.py:42`、`ui_batch.py:417`）。
7. **方向 34（包内 Markdown 供 help/export）：机制保留**，单一来源，漂移风险低。但内容需要大幅瘦身。公开指引从 81.5KB/11.5k 词增到 147.6KB/21.0k 词（+82%）。真实任务里候选 agent 在首个动作前读了 117–127KB 文档，基线约 85KB。公开 topic 里写进了锁、配额、ACK、发布提交点等内部机制（例如 `observations.md:62-117`、`batch.md`），这些不改变 LLM 的决策。
8. **方向 39（动作共用阅读选项、文本默认打印）：选项本身有实际使用证据，保留。** Chrome 候选自然用了 `--depth 1`、`--interactive`、`find --context`。缺的是一个默认上限或更小的默认投影。“基线序号/年龄”四字段和“maxChars 扩大回 full”只服务 diff，随 diff 一起删除或合并。
9. **LLM 使用面明显膨胀**：叶命令 47→70，命令局部选项实例 233→611，不同选项名 33→64，`click --help` 131→319 词，文本结果首行可能出现的字段 35→111。与 Playwright MCP、mobile-mcp、droidrun 等同类工具相比，命令面和单次结果都大 3–5 倍以上（同类工具规模凭记忆，见 §3.1，未在线核实）。
10. **host 代码**：
    - `ui_commands.py` 的 2,106 行主要是同一组阅读/图像选项手工重复 20–24 次。
    - `ui_client.py` 的 `result_line` 有 269 行逐字段特判（基线 107 行）。
    - diff 与 history、batch 交织，使交付路径多了 `store/track_history/retained/candidate` 四条状态线。

    删除 diff 并把选项声明表驱动化，可以直接减掉约 1,200–1,500 行产品码。

---

## 2. 负责方向处置表

| 方向 | 处置 | 依据 | 依赖与真实损失 | 维护代价变化 |
| --- | --- | --- | --- | --- |
| 6 UI 执行返回 Result | **保留** | ui_client 已无深层退出；batch 和单命令共用执行与交付 | 无 | 中性偏好 |
| 7 默认行、结构 JSON、删 XML 出口 | **保留格式；调整默认投影（必须）** | 行语法本身略省；默认字段膨胀导致 4.5× 回退 | 要同时动 actions 投影（23–27）、事件投影（14–15）、结果首行（16–19） | 投影规则变少（去掉逐事件、逐动作对象） |
| 8 显式 diff、唯一基线、真实交付提交点 | **删除**（次选：替换为无状态“unchanged 不打印正文”） | 零自然使用；离线收益约 11%；导航退化为 full；MB 级持久化 | 损失：大页、小改动时少打印一些 token；`--json` 下的结构化 changes | 删约 550 行产品码、约 900 行测试、约 1k 词文稿；history 只剩自动产物 |
| 9 静态 batch（真实 parser 预检、前缀停止） | **保留**；去掉 diff 交叉 | 设计正确；附件约少 72%，但未测出提速；可减少模型回合（未测） | 无 | `finish()` 中的 baseline 提交约 60 行可删 |
| 10 全局选项按命令边界消费 | **保留** | 约 40 行；减少 LLM 选项位置错误 | 依赖 `typer._click` 私有模块（`cli_options.py:10`），typer 已锁 `==0.27.2` | 低 |
| 34 包内 Markdown 供 help/export | **保留机制；调整内容** | 单一来源（`app.py:552-560`，pyproject 映射 skills 目录） | 无 | 需要把 12 个 topic 压到约 5 个，删掉内部机制叙述 |
| 39 动作共用阅读选项、默认打印、--no-print、基线序号/年龄、maxChars 扩大回 full | **选项保留**；**加默认上限或更小默认投影**；**基线四字段和“扩大回 full”随 diff 删除** | `--depth`/`--interactive` 有自然使用；Chrome 截断后重读 | 无 | 删字段 |
| 40 host 侧（真实 UTF-8 树定稿后安装） | 只核 host 消费：`ui_client.py` 新增 8 行，无独立意见 | — | — | — |

---

## 3. 发现（按重要性）

### F1【设计取舍→应视为回退】默认阅读正文膨胀 4.5 倍，主要来自 actions 对象数组和逐条事件

- **证据【实测，当前候选=99e31a 匹配物料】**：`stages/final-verification/paired-runtime/contacts-{1,2}/evidence/cli/*/child.stdout`。

| 臂 | CLI stdout bytes | 其中 actions 数组 | 其中 event/events 行 | 结果首行 | CLI stdout o200k tokens | 原生 input tokens | wall |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Contacts 基线 | 34,546 | 0 | 0 | 6,197 B / 1.7k tok | 9,775 | 812,987 | 171.4s |
| Contacts 候选 | 149,302 | **83,814** | 12,818 | 16,515 B / 4.4k tok | **44,077** | 1,461,258 | 207.0s |
| Chrome 候选(1c) | 183,595 | 93,580 | 23,685 | 15,618 B | 54,887 | 2,447,717 | 300.1s |
| Chrome 基线 | 35,845 | 0 | 0 | 10,046 B | 10,610 | 1,041,124 | 255.6s |

- **同一页面的两种输出对比**：Contacts “Create new contact”页，基线 click 输出 2,314B，候选 11,423B。
  - 基线 XML 只对 editable 打印 `actions="clear-focus select … fill ime"` 短名，其他目标只打印 expand/collapse/dismiss/long-click 这类非隐含动作（基线 `ui_view.py:52-77`）。
  - 候选 `ui_view.py:120-121` 对每个 target 打印完整原数组。例如 Cancel 按钮有 10 个对象，包含 `action-40`、`action-100`、`set-selection` 等对 LLM 无用的项。
- **Chrome 对比的局限**：Chrome 的成对比较不是等内容比较，基线没有读到网页正文。但同一份候选输出里，actions 已占 70%。
- **变体测算【实测原件，离线重投影，脚本 `agents/w05/scripts/measure_out.py`】**：

| Contacts 候选 16 条 CLI 合计 | o200k tokens |
| --- | ---: |
| 原样 | 44,077 |
| 交付方 actions-A（省 7 个普通无 label 动作） | 31,670 |
| actions-A + 默认不打印逐条 event | 29,402 |
| actions 仅保留短名且去掉隐含项（click/fill/long-click/scroll-*/set-selection/action-NN），并去掉逐条 event | **15,952** |
| 基线 | 9,775 |

  最后一个变体剩余部分的分布：节点 5.97k、结果首行 4.45k、header 行 2.1k、window 行 1.83k、events 摘要 1.2k、artifact 0.15k。
- **其余可删的噪声**：
  - 非选中系统窗口逐行打印（`ui_view.py:333-354`）。导航栏、状态栏每次 2–3 行，且 `content:"notRead"`。
  - header 中 `windowsSource` 与 `snapshotFacts` 的同源字段重复（`ui_view.py:505-506`）。
  - `artifact {"kind":"image","state":"not requested"}` 在每次未请求图像时都打印（`ui_view.py:376-378`，条件 `state != "available"` 把“not requested”也算进去）。公开文档没有说明这一行。
  - 结果首行的 `effect` 把 actions 数组的 before/after `jsonPreview` 也塞了进来（chrome-1c fill 首行 1,929B，关键事实只有 `fill=done, verification=matches, actualValue=…`）。
- **影响**：每轮累计上下文变大，回合成本和 wall 一起增加。超过宿主输出上限时触发截断和重读（见 F2）。
- **建议【设计建议】**：
  - 默认 actions 改为短名字符串，只列“非隐含”能力：custom 或带 label 的用 `name:label`；expand/collapse/dismiss/ime 等保留；click/long-click/fill/scroll 由 role/scroll 属性表达；focus/select/set-selection/action-NN 无 label 的省略。`--all`/`--raw` 保留全表。`action --action-id` 的需求由 `--all` 满足。
  - 事件默认只打印一行摘要，带文字的事件保留文字提示。
  - 系统窗口只在 IME 或对话框可见、或 `--all-windows` 时打印。
  - 去掉 `not requested` 和重复的 source 块。
  - 结果首行收敛为 `cmd: result; reason; 关键核值; change=<state>; path=...`，维度细节进 JSON。

### F2【证据缺口→已有反例】默认打印无上限，被宿主截断后发生重读

- **【实测】** `chrome-1c/native.rollout.jsonl`：`snapshot` 输出 “Warning: truncated output (original token count: 11694)”；随后 `Get-Content` 读保存文件，再次截断（11,649）。
- 候选把文本默认改为打印（`ui_commands.py:189-196`，`ui_client.py:1419-1423`），却没有默认 `--max-chars`。基线默认不打印，需要显式 `--print`，因此没有这个问题。
- **建议**：做完 F1 的投影瘦身后，再给文本默认打印设一个保守上限（例如 8–12k 字符）。超出时给 omission，并提示用 `snapshot --cached --depth/--interactive/REF` 读剩余部分。现有 omission 机制已经支持，不需要新机制。

### F3【设计取舍】diff（方向 8）收益小、使用为零、持久责任重

- **防的是什么**：大页面上小变化时重复打印全量。
- **真实发生过吗**：
  - 四次成对业务的 argv 中没有一次 `--diff`（`*/cli-index.json`）。
  - 研究离线语料（1,166 样本）中，保守 diff 只比紧凑行低 11.28%；`report.md:168` 自估总 token 削减 4.45%。
  - 真实导航 111/115 条变化（6,064/6,238 tokens），反超 full（912/1,095），只能退回 full（`research/cli/followup/large-page/report.md`）。
  - 稳定页 0 变化的收益约 900 tokens/次。按 F1 瘦身后，全量本身只有约 1k tokens，diff 的绝对收益进一步变小。
- **成本【源码推断】**：
  - `ui_diff.py` 357 行。
  - `history.py` 中 baseline 预留、配额、提交逻辑约 60 行。
  - `ui_client.py` 的 `require_diff_destinations`、`delivery_scope` 的 store/track_history/retained、`commit_baseline`、`baseline_failure` 约 100 行。
  - `ui_batch.finish` 约 60 行。
  - `test_ui_diff.py` 约 900 行。
  - `observations.md:62-117` 和 `batch.md` 后半约 1,000 词。
  - 每次 `--diff` 都在 workspace 锁内读入并写出整棵 canonical 树（实测 4.9MB；8192 节点上限对应 8MiB），占 20MiB 历史配额的 23.5%。
- **还有一处结构性局限【源码推断】**：基线按 workspace 只有一份（`history.py:17` `BASELINE_NAME`），比较键却包含 device/session（`ui_diff.py:40-55`）。同一 workspace 中交替使用两个 session 时，双方会互相把基线重置为 `contextChanged`→full。SKILL 允许共享 workspace，只要求显式指定 device/session。
- **删掉后的损失**：
  - 大页小改动时多打印若干百到一千 tokens。
  - `--json` 下的结构化 changes 消失。
  - 用户已批准“实现差分但显式开启”（L2142），删除需要重新裁决，见 §4-B。
- **更简单的替代【设计建议】**：设备侧已经为每个动作计算 `observedChange`（unchanged/changed）。文本模式下，动作结果为 `observation unchanged` 时只打印一行“正文与动作前样本相同（seq N）”，`--print` 强制打印。这样不需要 host 持久状态，就覆盖了 diff 最常见的收益场景（无变化）。

### F4【设计取舍】batch：设计合理，价值未测，与 diff 的耦合应删除

- **实现**：
  - `ui_batch.compile_plan`（`ui_batch.py:61-178`）用真实 Typer 根命令逐步 `make_context`，拒绝全局选项、非 UI 命令和 help/version，预检通过前不连接设备。
  - 执行时共享一个 `UiClient`，按步计时。只在显式请求、最后一步或停止步拉附件（`explicit_delivery:181-190`）。
  - 离线试了四种拒绝，都在零执行时以 exit 2 返回：步骤内 `--device`、步骤内 `--json`、`shell` 步骤、`--print` 与 `--json` 冲突。
- **证据【实测，研究原型 E5】**：附件读取 4→1，压缩字节约 28KB→7.8KB（-72%），但完整进程未见端到端提速（`research/cli/report.md:133-137`）。最终真实任务中没有自然使用。
- **真实价值所在【设计推断】**：对 LLM 来说，关键收益是**模型回合数**。候选 Contacts 每回合约 61k input tokens（1,461,258 / 24 responses）。`reveal X; fill X` 这类同页已知 REF 的序列（contacts-2 就有两对）合并后可以省 2 个回合，约 12 万 tokens，远大于附件字节。这一点没有测过。
- **建议**：
  - 保留 batch。
  - 删除 diff 后，`finish()` 的 baseline 候选、受保护保存路径、最终锁内提交都可以去掉（`ui_batch.py:212-271`）。
  - 文稿里“默认打印不触发中间附件、显式 `--print` 触发”的三态语义（`batch.md`）不必改动，但可以简化为“中间步只给结果行；要正文请加 `--print`”。
  - 在 SKILL 中给一个 `reveal`+`fill` 的具体 batch 范例，比现在的“只在完全已知的静态序列使用”更能引导实际使用。

### F5【设计取舍】LLM 使用面规模与文稿语言

- **【实测，本地安装统计】**（脚本 `agents/w05/scripts/count.py`，原始输出在 `agents/w05/{cand,base}_cmds.txt`）：

| 指标 | 基线 7d5be70 | 候选 3f45af6 |
| --- | ---: | ---: |
| 叶命令 / 命令组 | 47 / 5 | 70 / 9 |
| 命令局部选项实例（不含 4 个全局） | 233 | 611 |
| 不同局部选项名 | 33 | 64 |
| `click` 局部选项 / `--help` 词数 | 11 / 131 | 22 / 319 |
| 文本结果首行可出现的字段键（`result_line`） | 35（107 行） | 111（269 行） |
| 公开指引 | 2 文件 / 81.5KB / 11,534 词 | 13 文件 / 147.6KB / 20,993 词 |
| `contract/wire-contract.json` | 175KB | 441KB |
| host 产品 Python | 约 7.98k 行 | 11.96k 行（+4,572/-621）；测试 +9,458 |

- **“打开 App→找字段→输入→核验→保存”最少要读多少**：
  - SKILL.md（1,128 词）给了 `click`/`fill`/`snapshot --cached` 范例，但没有给 `open`/`apps`，也没有讲保存或确认。实际要再读 actions.md（3,645 词）和 observations.md（4,131 词）才能理解行格式和 fill 结果。
  - 真实任务中，Contacts 候选首个动作前读了 116.8KB 文档（基线 85.1KB）；Chrome 候选读了 127KB，还额外读了 images.md。
- **语言密度**：否定和限定词密度与基线相近（`not/no/never/cannot/without` 每千词 28.2 vs 27.3），也就是继承了旧风格；但总量翻倍。最容易让 LLM 误用或多一轮往返的地方：
  1. “baseline”一词有两种含义：结果首行的 `baselineSequence/AgeMs/...` 指动作前采样，`--diff` 的 baseline 指 workspace 阅读基线。
  2. `--print` 与 `--json` 一起用直接报错并退出 2。基线文档长期教 agent 用 `--print`；带着旧习惯迁移的 agent 会多一轮。
  3. 正文里每个 target 都有 actions 对象数组，诱导 agent 去用 `action --action-id 64` 这类无语义动作。
  4. `--help` 中 `--diff` 的说明写 “Incompatible with --raw”，但大多数命令根本没有 `--raw`（`ui_commands.py:170-177`）。
  5. observations.md 第 1 段就要读者区分 “published / delivered / downloaded / completeness / collection policy” 五个概念，对 LLM 的下一步决策没有作用。
  6. 结果状态 `failed` 可以伴随 done 的步骤，`done` 又必须另外核业务；SKILL 的表格说清楚了，但到处重复的 “X is not Y” 提醒稀释了这个重点。
- **与同类工具对比【设计参照，凭记忆，未在线核实版本】**：
  - Playwright MCP 约 20–25 个工具，交互工具一般只需 `element` 描述加 `ref`，快照是 ARIA YAML 带 ref，不逐节点列动作。
  - mobile-mcp 约 15–20 个工具，元素列表只有 type/text/label/坐标。
  - droidrun 约 10 个工具（按索引点击、输入、滑动、返回、启动 App、取状态）。
  - uiautomator2 是 Python API，不是面向 LLM 的 CLI。

  候选在命令数和单次输出体积上都明显偏大。“细分”主要出现在：
  - 8 个单动作命令：check/uncheck/focus/long-click/expand/collapse/dismiss/ime。
  - 5 个指针命令：tap/swipe/drag/pinch/timeline。
  - action/reveal/range 三个显式动作命令。
  - 通知写操作要求同时抄写 `--post-time --creator --creator-uid --title|--title-null` 守卫（方向 30–32，不在本专题处置）。

  其中前两类在基线就已存在（基线 check…ime 也是 8 个），不全是本轮新增。本轮新增的主要负担是**每条命令附加的约 11 个阅读和图像选项**，以及**结果字段**。

### F6【设计取舍】默认打印、--no-print、--json、--print 规则

现有规则（`ui_client.py:937-938`，`print_delivery:1407-1427`）：
- 文本模式默认打印正文；`--no-print` 抑制。
- `--json` 从不在 stdout 放正文。
- 显式 `--print` 加 `--json` 时拒绝，`--no-print` 加 `--json` 时接受。
- batch 中间步“默认”不打印，显式 `--print` 才拉附件，因此是三态。

单命令规则一致，但有两处可以简化：
1. 不必拒绝 `--print --json`，可以让它成为 no-op，或者把正文放进 `data.body`。这样能消除一个只为“语义纯洁”存在的失败路径。
2. 只有保存成功才打印（`"snapshotPath" in result.data`），因此配额或磁盘失败时，已渲染好的正文也不给出，agent 只能重新采样。正文打印不必依赖保存成功。

### F7【设计取舍】host 结构

- **`ui_commands.py`（2,106 行）**：
  - `max_chars: MaxChars` 出现 22 次，`print_body: Print` 24 次，`diff: Diff` 23 次，`image_format` 20 次。每个命令函数手工声明并逐个转发阅读选项，`run_action` 也是逐参数转发（`ui_commands.py:355-389`）。
  - 方向 10 已有“把一组参数程序化附加到所有子命令”的先例（`cli_options.py:93-118` `GlobalOptionsGroup.share`）。阅读选项和图像选项同样可以用一个共享参数组附加到 `UiCommand`，回调里组装 `View`，预计能减 600–900 行。
- **`ui_client.py`（1,436 行）**：
  - `result_line`（194–462）为通知、剪贴板、editor、open、scroll-until、image 各写了一套字段白名单，与 JSON 输出形成两套视图。可以改成“首段固定摘要，后接一组按命令声明的字段”，或直接 `facts=<compact JSON 的选择子集>`。
  - `prepare_ui`（836–990）集中写了 zoom/range/action/touch/expect 的逐命令参数校验，这是为了让 batch 预检复用。它可以留在共享层，但宜放回各命令的声明处。
  - diff 删除后，`delivery_scope` 的 `store/track_history/retained/candidate` 四条线可以收成一条“保存自动产物”。
- **host 持久状态，共 4 类**：
  - 用户级：pairing 记录与 `device.json` 选择（基线已有）；`display.json` 与 `display.lock`（方向 38 新增，主机级 device+session 显示偏好）。
  - workspace 级：`config.json`（use 默认）；`snapshots/`、`shots/` 历史（150 文件/20MiB）；`history.lock`；`.gitignore`；`diff-baseline.json`（新增，MB 级）。
  - 结论：只有 `diff-baseline.json` 是本专题新增的重状态，建议删。其余必要。

### F8 确定缺陷与小问题

| # | 类型 | 位置 | 触发路径与影响 | 建议 |
| --- | --- | --- | --- | --- |
| D1 | 【确定缺陷，低】 | `androidctl/src/androidctl/ui_commands.py:170-177` | 所有动作命令（如 `click --help`）的 `--diff` 帮助写 “Incompatible with --raw”，但这些命令没有 `--raw`。LLM 可能尝试 `--raw` 并收到 “No such option” | 只在 snapshot/find 显示，或删除该句 |
| D2 | 【确定缺陷，低→噪声】 | `androidctl/src/androidctl/ui_view.py:376-378` | 未请求图像的每次观察都打印 `artifact {"kind":"image","state":"not requested"}`（真实 stdout 每条都有）；公开文档没有说明 | 条件中排除 `not requested` |
| D3 | 【易误用】 | `androidctl/src/androidctl/ui_diff.py:291-299` | diff changes 超出 `--max-chars` 时，omission 提示 `read: "snapshot --cached --diff"`。但本次已提交为新基线，若照旧带同一 `--max-chars` 重跑会得到 0 变化，被省略的变化无法重读（`observations.md:104-106` 已承认）。只有去掉或扩大预算才会触发 `textBudgetExpanded`→full | 随 diff 删除；若保留，提示应写明“去掉 --max-chars” |
| D4 | 【设计缺口】 | `androidctl/src/androidctl/history.py:17`、`androidctl/src/androidctl/ui_diff.py:40-55` | 每个 workspace 一份基线，同一 workspace 的多 session/device 交替 diff 时互相重置 | 随 diff 删除 |
| D5 | 【设计缺陷，低】 | `androidctl/src/androidctl/ui_client.py:1419-1423` | 只有保存成功才打印正文；配额或保存失败时正文在内存中却不打印 | 打印不依赖保存 |
| D6 | 【死代码】 | `androidctl/src/androidctl/history.py:117` | `MAX_FILES < 1` 恒为假 | 删除 |
| D7 | 【维护风险】 | `androidctl/src/androidctl/cli_options.py:10` | 导入 `typer._click.core` 私有模块；依赖 `typer==0.27.2` 锁定 | 可接受，但升级 typer 前需回归 |

没有发现会导致错误派发或数据损坏的 host 缺陷。批量预检、全局选项拒绝和 `--print/--json` 冲突的离线行为与文档一致。

---

## 4. 建议重新裁决的事项（A/B/C/D）

### 4-A 默认阅读正文内容（覆盖交付方的“actions 精简”和“事件精简”两项未批准选项）

| 选项 | 内容 | 优点 | 代价 |
| --- | --- | --- | --- |
| A | 保持现状 | 无改动 | 已实测 4.5× token 回退，并引发截断重读 |
| B | 交付方推荐：actions-A（只省 7 个普通无 label 动作）+ 事件-A（摘要 + 文字提示） | 改动小；数组形状不变 | Contacts 实测只降到约 29.4k，仍为基线的 3 倍 |
| C（**推荐**） | actions 改为“非隐含能力短名”（custom/label 用 `name:label`；省略 click/long-click/fill/scroll/focus/select/set-selection/无 label 的 action-NN）；事件只给一行摘要；系统窗口只在相关时打印；去掉 `not requested` 和重复 source；结果首行收敛；加默认文本上限 | 实测降到约 16k，再加上首行和窗口收敛可接近基线；LLM 更少被引向数字 action-id | `--action-id` 需要 `--all` 才能发现；混合形状需在文档用一句话说明；需同步 diff/batch 的视图（若 diff 保留） |
| D | 默认只打印可交互节点和带文本节点，折叠纯布局容器，相当于把 `--interactive` 加文本节点设为默认 | 最省 | 现有投影已经折叠空布局（`ui_view.py:260`），额外收益有限；丢失结构上下文，容易误读同名项 |

剩余未知：C 的短名会不会让 agent 漏用某个 custom action。补证方式：用一对新的 Contacts 或 Chrome 成对任务跑 C。如果 input 降到 ≤1.2× 基线且业务完成，就定稿；如果出现因缺 action 而多一轮 `--all`，再考虑保留 label 动作的对象形式。

### 4-B diff（用户 L2142 已选“实现差分但显式开启”，建议重议）

| 选项 | 内容 | 优点 | 代价 |
| --- | --- | --- | --- |
| A | 保持现状 | 能力完整 | 约 550 行产品码、约 900 行测试、约 1k 词文档；MB 级基线；零自然使用 |
| B（**推荐**） | 删除 `--diff` 和 workspace 基线；动作后若设备报告 `observation unchanged`，文本只打印一行引用，`--print` 强制全文 | 无 host 持久状态；覆盖最常见的“无变化”收益；batch、交付路径和文档同步简化 | 失去大页小改动的逐节点 delta 和 JSON 结构化 changes |
| C | 保留 diff，但基线改存“已渲染投影 records”（KB 级）而非 canonical 树，view 或预算变化一律 full，并按 (device, session) 分文件 | 去掉 MB 级 IO、maxChars 上下文、`textBudgetExpanded` | 仍有持久状态、锁和配额，零使用问题仍在 |
| D | 只保留 `--json` 下的 diff，文本不提供 | 脚本消费者可用 | 主要用户（LLM 文本）得不到收益，复杂度几乎不降 |

剩余未知：是否有用户在长列表逐项编辑类任务中依赖 diff。如果有真实证据（例如 WPS 逐格编辑的成对对照中 diff 每回合省 >2k tokens 且被自然采用），改选 C。

### 4-C 公开指引规模

| 选项 | 内容 |
| --- | --- |
| A | 保持 13 个文件、21k 词 |
| B（**推荐**） | SKILL.md 自足覆盖“配对→open→snapshot→click/fill→核验→close”的完整范例，包括 `open`/`apps`。topic 合并为约 5 个：observations+actions、input+clipboard、images、notifications、connection+jobs-files+displays。删除锁、配额、ACK、发布提交点、`published≠delivered` 等内部叙述（内部 `docs/reference/ui.md` 已有）。目标是总量 ≤10k 词 |
| C | 保持文件拆分，只删内部叙述 |
| D | 只保留 SKILL 加 `--help`，删除 topic |

推荐 B。依据是真实任务中 agent 首个动作前读了约 120KB 文档，而且 SKILL 本身不足以完成一个最小业务。

### 4-D `--print` 与 `--json`

- A：保持拒绝。
- B（推荐）：`--print --json` 合法，把正文放进 `data.body`；`--json` 默认不放。这样一次调用就能拿到结构化结果加正文。
- C：`--print` 在 JSON 下视为 no-op。

推荐 B：它去掉一个失败路径，同时少一次读文件调用。

---

## 5. 对既有内部 review 与交付方声明的对照

1. **`deliverables/candidate-final.md:57、76-115` 与 `reviews/u25-pending-questions-review.md`** 把默认 events/actions 精简定为“未批准的可选改进，保持现合同不阻塞”，并明确“本文不推导 token 节省或 agent 净收益”。

   我不同意。同一份交付稿第 63–72 行已列出两对候选的 input 都明显更高（Contacts +80%，Chrome +135%），却没有把它与默认正文字段关联起来。原件可以直接分解出：Contacts 候选 stdout 的 56% 是 actions 数组。“不改变当前可见内容需要授权”这个理由也站不住：用户在 L2142 批准的是“默认行格式”以提升效率，并没有批准“每目标默认印全 actions 对象”。这两项字段默认是在阶段 13、22 计划中由实施方自行决定的（`external-host-audit.md:71` 自己也承认没有用户明示）。
2. **`research/cli/followup/large-page/report.md` 的“行格式约省 15%”** 在其口径内成立（同字段、同 metadata）。但它测量的投影早于 actions 对象数组和近期事件的加入，不能用来代表最终默认正文的成本。交付方继续引用“约省 11.5%”作为效率依据，属于口径错配。
3. **`reviews/final-maintainability.md:33`** 称 “ui_diff/history 保一份 canonical base … 这些保留责任实际用于失败交付及 batch 前缀，不能只因代码较长删除”。我不同意它的推理：删除理由不是“代码长”，而是零自然使用、离线收益约 11%、MB 级持久状态和跨 session 失效。batch 前缀不依赖 diff。
4. **同一 review 第 46 行** 称“不建议把内部机制搬入 SKILL”。但现公开 topic 已经包含大量内部机制（锁、配额、提交点、ACK），方向与该 review 的主张相反，它没有指出这一点。
5. **`reviews/final-paired-runtime.md`** 正确指出成本差“不能归因单一机制”。但 Contacts 一对路径高度可比（同 16 条 CLI、同字段），逐行原件足以把大部分差额归因到默认正文字段。“不归因”的保守表述在这里掩盖了可以操作的结论。
6. **`reviews/external-host-audit.md:3、18-19`** 已确认“效率-1/效率-2 成立”，结论却是交用户选择。我同意要交用户选择，但它没有给出成对任务量级，使问题看起来像微调，实际是本轮最大的使用效率回退。
7. **交付方“batch 附件少 72% 但无端到端提速”的陈述准确。** 我补充的是：对 LLM 而言，batch 的潜在主要收益在于模型回合，而这一点从未被测。

---

## 6. 未覆盖范围与必要补证

**未覆盖**：
- 未逐行审 `ui_commands.py` 每个命令的参数语义，也未审 `job_commands.py`、`file_jobs.py`、`app.py` 的配对和 daemon 部分。
- 未运行 host 测试套件。
- 没有设备，所有运行结论都来自包内原件。
- 同类工具规模凭记忆，未在线核实版本。
- 内部 `docs/reference/ui.md` 只看了目录结构，未逐段比对与公开 topic 的一致性。
- tokenizer 计数是 o200k_base 对 CLI stdout 的离线计数，不等于模型实际计费，也不包括 harness 的 JSON 转义。

**会改变判断的补证**（各一对即可，不需要矩阵）：
1. **4-A 的 C 方案成对对照**（Contacts 同任务）：若候选 input 降到 ≤1.2× 基线且业务完成，即可定稿默认投影；若因缺 action 而多出 `--all` 回合，改为保留带 label 动作的对象形式。
2. **batch 回合收益**：在同一 Contacts 任务的 SKILL 中加一个 `reveal+fill` batch 范例，观察 agent 是否自然采用、回合是否减少。若无人采用，可把 batch 降级为不在 SKILL 推荐的进阶工具；若采用且回合 -20% 以上，batch 价值成立。
3. **diff 的真实需求**：仅当有长列表或表格逐项编辑任务（如 WPS）成对显示 `--diff` 被自然使用、且每回合省 >2k tokens 时，才改选 4-B 的 C。
