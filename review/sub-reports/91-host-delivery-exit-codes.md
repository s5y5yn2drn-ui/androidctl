# 专题核查：主机端"观察 → 正文／摘要 → 保存 → 首行与退出码"行为链（候选 `f401deb`）

核查员工作目录：`S/agents11/91`（`S` = 本轮评审的 scratchpad 根）。本报告中"本机复现"的脚本都在这里：`cap.py`（直接调用 `ui_client.deliver` 和 `print_delivery` 的小框架）、`exp1.py`–`exp4.py`（各组实验）、`calls.py`（从模型原生记录里按顺序列出工具调用与返回字节）、`bytes.py`（按命令类别对比上一轮与本轮的标准输出字节）。实验输入是实施方后台屏长网页检查保存的真实原始观察 `ultragoal/…/42-r4-decisions/runtime/api34/verification/r6-delivery-check/.androidctl/snapshots/000012-…-List_of_national_capitals_-_Wikipedia.json`（2,931 个节点，采样不完整）。

## 摘要

1. **上限本身做对了，但两次真实触发都没有省下成本。** 判断按实际 UTF-8 字节，严格大于 65,536 才触发，同时进入了"打印即补存"、主结果和退出码三处逻辑；`--json`、`--out`、`--print`、`--no-print`、`--raw` 的表现都符合第 14、23 条（本机复现）。20 次真实运行里上限只触发 2 次，两次都是网页题里的整页 `snapshot --all`。两次模型拿到摘要后都**立刻用 Get-Content 读了整个文件**，读回的内容被执行框架截在约 89 KB。上一轮同样的整页 `--all` 直接打印时，模型收到的也是 89 KB。也就是说，读到的内容一样多，还多了一回合。这两次运行正好是网页题五次里最贵的两次（677,133 和 519,095 总输入，中位数 387,934）。
2. **成本真正来自 `--all` 本身，而且有两处是工具自己造成的。** 第一，`--all` 下 A1 的省略整个不生效：已保存的两个整页 `--all` 文件里，`action-40`、`action-400` 这类无名标准动作占 32%–39% 的字节，所有 `action-name=` 合计占 41%–50%。在维基页上只要把默认的动作省略也用到 `--all`，正文就从 843 KB 降到 406 KB。有一次模型自己写了正则去删 `action-name`。第二，`find` 输出里的每条省略记录都被改写成 `read="snapshot --cached --all"`，连带引用的定向读法也被覆盖了。命中 0 条时这条提示没有意义，因为 `find` 本来就会搜屏外文字。触发上限的两次里，有一次正是模型照抄了这条提示。
3. 摘要里"怎样缩小范围"那一行固定写成 `androidctl snapshot --cached --depth 2`，跟用户实际敲的是什么命令无关。这条写法照抄能跑，但很少有用：维基页上只剩 1.1 KB，正文区 384 个节点全被省略。真实运行里两次都没人用它。评审方第六轮的风险提示建议按命令给不同写法，实施方没有采纳。
4. **首行和退出码：没发现把做过的动作报成失败或未发出的路径。** 只有 `snapshot`、`find`、`screenshot` 的主结果会被交付环节改写。`f401deb` 修的是 `545d095` 引入的一个回归：完成、退出码非零时首行丢了原因。修法本身对，但不完整：`fill` 完成、退出码 3 时，"请用户在设备上停止空闲执行器"那句指引仍被吞掉。这个问题旧已有之，不是本轮新引入的。
5. 同一件交付物现在只有一个诊断字段（`snapshotError`/`imageError`），`historyError`、`snapshotWarning` 在主机源码、指引和契约里都已清零。缺点是 `snapshot`/`find` 失败时，首行把同一段错误原文打了两遍。
6. 默认文本瘦身实际有效：同类命令的输出比上一轮少 25%–60%。采集编号、缺口标记、提示条文字都还在。322 次真实调用的默认输出里 `action-NNN` 清零，只剩 2 次 `find --all`。剩下能删的不多（约 3%–6%），见第 2 节。
7. 首轮质量检查的 60 个失败里，只有 1 个对应真实回归，另外 59 个是测试没跟上 `545d095` 的有意改动。这说明 `545d095`、`642799b` 提交时没有跑过主机全量测试。主机测试约 65% 在断言渲染文字，耦合确实很重。
8. 第 18 条的实验测的是"强杀命令行进程"，不是"本机连接没关好"，回答的是相邻的问题。它证明孤儿请求会占着会话一直到原截止时间，`wait` 最长 60 秒；这期间下一条命令报 `connectionBusy`，原因文字里既没有占用者，也没有释放时间，码表还把它写成"另一个会话"。这值得进下一轮处理，不挡合入。

**总体判断**：这条链上没有挡合入的缺陷。最值得进下一轮的是第 2 条里的两处（`--all` 下也省略无名标准动作，`find` 省略记录给定向的读法），以及第 3 条的摘要缩小提示。这三处每处都只改几行，但它们直接决定上限是"真省成本"还是"把成本挪去读文件"。

---

## 1. 输出上限（第 14 条，第四节第 1 处）

### 1.1 阈值与依据——已核实无问题

- 常量是 `DEFAULT_BODY_BYTES = 65_536`（`androidctl/src/androidctl/ui_client.py:45`）。判断是 `len(body.encode("utf-8")) > DEFAULT_BODY_BYTES`（`ui_client.py:735-741`），按实际 UTF-8 字节、严格大于。依据：源码推断，另有测试 `test_default_limit_counts_utf8_bytes_and_explicit_print_overrides_it`（我跑过，通过）。
- 实施方的依据（`ultragoal/…/42-r4-decisions/r6-delivery-implementation.md` 第 56–58 行）是"取区间低端；后台屏长网页修好后约 49 KB 仍可全文"。我补一条更硬的依据：这个执行框架每次工具输出的上限是 `max_output_tokens: 20000`，真实截断点在约 89 KB（实测，两次读文件返回 89,082 和 88,646 字节，都带 "truncated output"）。64 KiB 低于这个截断点，所以低于上限的正文在这个框架里都能完整送到。取低端是对的。
- 小瑕疵：摘要里那句 `Observation exceeds 65536 UTF-8 bytes` 是写死的字符串（`ui_client.py:878-881`），没有引用常量。我把常量改成 4000 跑实验，文字仍写 65536（本机复现，`exp3.py`）。性质：可选建议；去向：进下一轮，改一行。

### 1.2 判断落在哪一步——已核实无问题

上限在 `deliver` 里求出 `summary`（`ui_client.py:735-742`），然后：

- 传给 `finish_delivery_result` 的 `print_body` 在有摘要时为假（`ui_client.py:766-768`），于是 `body_file_required` 为真，"正文已打印、文件丢了也算交付"的补救分支被关掉（`ui_client.py:1008-1016`）；
- `print_delivery` 里 `body_printed = execution.summary is None and …`（`ui_client.py:1086-1088`），摘要不算正文已打印；
- `--json`（`context.as_json`）、显式 `--print`（`view.print_body is not None`）、`--raw` 都绕开摘要。

第四节第 1 处提醒的三个位置都改到了。

### 1.3 真实摘要（本机复现，`exp1.py`，输入为维基页原始观察）

`snapshot --cached --all`：正文 843,452 字节，标准输出 523 字节：

```
snapshot: done; observationWarning="partial; missing=2"; snapshotPath="ws/.androidctl/snapshots/000001-…-Chrome_Beta__List_of_national_capitals_-_Wikipedia.txt"
androidctl capture="3f02:32"
observation {"partial":true,"missing":2}
screen app="com.chrome.beta" title="Chrome Beta: List of national capitals - Wikipedia" keyboard="hidden" unreadable
Observation exceeds 65536 UTF-8 bytes; full text requires a saved file.
Narrow the current sample: androidctl snapshot --cached --depth 2
```

`find a --all`（1,205 个命中）：正文 681,684 字节，摘要比上面多一行 `find matches=1205 offscreen=1091 shown=1205`。

真实运行里的两份摘要（实测，`efficiency/r6-c2-candidate-01/evidence/cli/1795616656204020000-…/child.stdout` 和 `r6-c2-candidate-05/evidence/cli/1795620225342291500-…/child.stdout`）形状相同，各约 600 字节。

对照裁决的五项要求（首行、采集编号、缺口行、`find` 命中行、文件路径、缩小写法）：全部都在。路径在首行里（`snapshotPath=`），不在摘要正文里。提示条在摘要里也保留（`ui_client.py:861-863`，另有测试 `test_oversized_observation_summary_keeps_this_samples_toast_text`）。

### 1.4 调用者拿到摘要后能否直接继续——不能；缩小提示照抄有效但很少有用

- **对动作命令**：摘要去掉了所有节点，也就去掉了所有引用。点击之后想做下一步，必须读文件，或者再取一次观察（本机复现：`click` 加小阈值，见 `exp3.py`）。这是裁决"超限只能读文件"的直接后果，属于设计取舍。
- **缩小提示**：`snapshot --cached --depth 2` 是合法命令，读的是同一份缓存样本，照抄能跑。但在维基页上它只给 1.1 KB，`main` 区 384 个节点全被省成一条 `omission reason="depth"`（本机复现）。模型用 `--all` 通常是为了看屏外内容，这条提示对这种需求没有帮助。对 `find … --all` 给的也是同一句 `snapshot`，没有提示"去掉 `--all`，默认最多显示 20 个命中"。评审方的风险提示（`review/sub-reports/83-ruling-risks.md:110`）原本建议按命令给 `snapshot --cached REF --all`、`find TEXT`、`snapshot --cached --interactive`、`--max-chars`。实施方选了一句通用写法，裁决本身没要求按命令区分，所以这不算违反裁决。
- 摘要里没有文件大小和行数。模型无从判断"整读会被截断，应该先搜"。指引 `skills/androidctl/SKILL.md:85-87` 写的是 "Read/search that file after host-tool truncation"，等于默许整读。
- **建议**（可选建议，进下一轮）：缩小提示按命令和选项给。用了 `--all` 时优先给"去掉 `--all`"或"`snapshot --cached REF --all`"；`find` 给"去掉 `--all`"。摘要补一行文件字节数和行数，加一句"先搜索文件再读"。维护代价约 20 行，外加 2–3 个测试。

### 1.5 各命令超限且存不成文件——符合裁决（本机复现，`exp1.py`、`exp3.py`，工作区 `.androidctl` 被一个同名文件挡住）

| 命令 | 主结果 | 退出码 | 首行诊断 |
| --- | --- | --- | --- |
| `snapshot --all` | failed | 1 | 原因和 `snapshotError` |
| `find a --all` | failed | 1 | 同上 |
| `click`（摘要） | done | 1 | `snapshotError=` |
| 不带子命令的 `notifications` | done | 1 | `snapshotError=` |
| `wait --text`（摘要） | done | 1 | `snapshotError=` |
| `locate` | done | 1 | `snapshotError=` |
| `scroll-until` | done | 1 | `snapshotError=` |
| `click` 结果未知（UNKNOWN） | UNKNOWN | 3 | 加"不要自动重做" |
| `snapshot --all --print` | done | 0 | 警告，正文全文已打印 |
| `snapshot`（未超限） | done | 0 | 警告，正文全文已打印 |
| `click --no-print` | done | 1 | `snapshotError=` |

`locate` 和 `wait` 同样处理（完成，退出码 1）。裁决写的是"`snapshot`、`find` 这类以观察为主的命令"。`locate` 的主结果是一个可操作的引用，按动作类处理说得通，`skills/androidctl/images.md:85` 也照此写明。判断：已核实无问题。

### 1.6 `--out`、`--json`、`--print`——已核实无问题

- `--out mine.txt` 超限：摘要照打，首行 `snapshotPath="mine.txt"`，指向 `--out` 给的文件，没有另存历史副本（本机复现）。
- `--json` 超限：只打印约 490 字节的结果对象，不受上限影响（本机复现）。
- 显式 `--print`：843 KB 全文照打；文件存不成时仍报完成、退出码 0，只附一条警告（本机复现）。这与第四节第 1 处的读法一致。

### 1.7 20 次真实运行里的触发情况与读文件成本（实测）

| 运行 | 命令 | 正文文件 | 之后模型做了什么 | 模型收到的字节 |
| --- | --- | --- | --- | --- |
| 网页题第 1 次 | `snapshot --cached --all` | 168,039 | 立即 `Get-Content -Raw` 整读 | 摘要 934，读文件 89,082（被截断） |
| 网页题第 5 次 | `snapshot --all` | 259,096 | 整读，然后 `Select-String`，再按行号切片并用正则删掉 `action-name` | 摘要 986，读文件 88,646 + 87,508 + 23,093 |

其余 18 次没有触发。默认文本的最大单次输出是 10,940 字节（网页题第 5 次的 `back`）。

**上一轮对照**：上一轮网页题第 1 次在同一位置执行 `snapshot --cached --all`，直接打印 180,175 字节，模型收到的是截断后的 89,328 字节（`S/pkg9/…/efficiency/r5-c2-final-candidate-01`，实测）。本轮两次加起来，模型收到的内容和上一轮几乎一样多，还多了一次往返。按原生记录逐次的输入令牌估算：网页题第 1 次那次多出的回合输入约 24,374 令牌，占这次运行 677,133 的 3.6%；网页题第 5 次三次读文件后，上下文从约 30K 涨到约 56K 令牌，之后每回合都要重送。触发的两次正好是网页题五次里最贵的两次（677,133、519,095；另外三次 314,586–387,934）。这个贵不全是上限造成的，但上限没有起到省成本的作用。

**性质**：设计取舍（上限作为防线是对的）+ 证据显示没有收益。**去向**：进下一轮，改法见第 2.4 节和 1.4 节，不必撤掉上限。

---

## 2. 默认文本瘦身与 A1、A4（第 15 条）

### 2.1 字节变化（实测，`bytes.py`，同一任务、同一命令类别的标准输出中位数）

| 任务 | 命令 | 上一轮 | 本轮 | 变化 | 首行（上一轮→本轮） |
| --- | --- | --- | --- | --- | --- |
| 网页题 | snapshot | 11,186 | 6,167 | −45% | 261→255 |
| 网页题 | click | 11,517 | 6,890 | −40% | 323→306 |
| 网页题 | fill | 11,683 | 6,230 | −47% | 574→471 |
| 网页题 | back | 10,837 | 7,600 | −30% | 929→306 |
| 网页题 | find | 5,273 | 3,846 | −27% | 288→289 |
| 网页题 | find --all | 8,894 | 8,601 | −3% | — |
| 长列表题 | click | 1,682 | 866 | −49% | 426→272 |
| 长列表题 | snapshot | 5,674 | 4,729 | −17% | — |
| 长列表题 | scroll-until | 6,238 | 5,460 | −12% | 1,022→680 |
| 后台屏题 | click | 1,899 | 753 | −60% | 418→257 |
| 后台屏题 | open | 2,020 | 745 | −63% | 1,630→465 |
| 后台屏题 | fill | 2,439 | 1,501 | −38% | — |
| 路线题 | snapshot | 6,832 | 3,339 | −51% | — |
| 路线题 | scroll | 5,121 | 3,868 | −24% | — |

除 `find --all` 外都明显下降。

### 2.2 该留的都还在——已核实无问题

- 采集编号：每份正文都有 `androidctl capture="…:N"` 一行（实测，全部 20 次运行）。
- 缺口标记：首行 `observationWarning="partial; missing=N"`，正文头 `observation {"partial":true,"missing":N}`，19 份输出带这个标记。
- 提示条：`toast text=["R4 Local Contact saved","Contacts"]` 出现 13 次。
- `open` 成功只留 `display=N` 和界面树，启动细节进 `--json`（实测：后台屏题 `open` 的首行只剩 465 字节）。
- A4：成功的 `scroll-until` 不再打印 `completedSteps`（`ui_output.py:217-219`，实测）；动作成功时不打印 `dispatchStarted`（`ui_output.py:96-102`）。

### 2.3 `action-NNN` 是否清零

在 20 次运行全部 322 次调用的 `child.stdout` 里搜 `action-name="action-<十六进制>"`：默认输出 0 次。命中的只有 2 次 `find May --all`（网页题第 1、3 次，各 28 处）。保存的文件里有 4 份含这类记号，全部来自 `--all`（实测）。

### 2.4 发现：`--all` 下 A1 整个不生效，这是触发上限的主因

- 源码：`compact_actions` 的两处省略都以 `not all_nodes` 为前提（`ui_view.py:117-122`、`ui_view.py:139-143`）。`--all` 会打印每个节点的全部动作，包括 `action-40`（编号 64，无障碍焦点）、`action-400`、`action-800`、`action-100/200`、`action-1020044`。
- 实测：网页题第 5 次保存的 `--all` 文件 259,096 字节，其中 `action-name=` 合计 106,627 字节（41%），无名标准动作 82,498 字节（32%）；网页题第 1 次的文件 168,039 字节，对应比例是 50% 和 39%。网页题第 5 次里，模型读文件时自己加了 `-replace ' action-name="[^"]*"',''`。
- 本机复现（`exp2.py`）：维基页 `snapshot --all` 是 843,452 字节；把 `--all` 的动作省略改成和默认一样后是 405,678 字节，减了一半以上。
- 裁决 A1 的字面只管"默认输出"，所以这不违反裁决。但 `--all` 的帮助写的是"Include hidden nodes and extra fields"，模型用它几乎都是为了屏外节点，不是为了动作清单；完整动作清单本来就在 `--raw` 里。
- **性质**：设计取舍，成本证据明确。**去向**：进下一轮（不挡合入：只多花钱，不会误操作）。最小改法：`--all` 只加屏外节点，动作省略照用默认规则，把"全部动作"留给 `--raw`。维护代价：去掉两处 `not all_nodes and`，改 3–5 个断言 `--all` 动作的测试，改指引一句。

### 2.5 发现：`find` 的省略记录一律指向整页 `--all`

- 源码：`observation_nodes` 在 `find` 时把每条省略记录的 `read` 都覆盖成 `"snapshot --cached --all"`（`ui_view.py:896-898`）。`omission()` 本来会为有引用的子树生成定向的 `snapshot REF`（`ui_view.py:690-693`），也被覆盖了。这处从 `fb75afc` 起就有，不是本轮引入的。
- 实测：20 次运行里这条提示出现了 117 次。网页题第 1 次：`find "hide cancellation"` 新采后仍是 0 条，输出是 `omission reason="find" nodes=654 read="snapshot --cached --all" ref="e950415517"`。模型下一条就照抄执行 `snapshot --cached --all`，触发上限，然后读了 89 KB 文件（`calls.py` 输出）。
- 命中 0 条时这条提示毫无意义：`find` 本来就会搜已采到的屏外文字（`ui_commands.py:711-716` 的帮助），整页 `--all` 不会多出这段文字。同一份输出的 `find` 行还写着"用 scroll-until 去取未采集的内容"，两条提示互相矛盾。
- **性质**：确定缺陷（指引误导，抬高成本）。**去向**：进下一轮。最小改法：有引用时保留 `snapshot REF`，需要屏外时写 `snapshot --cached REF --all`；命中 0 条时不给省略记录的读法。维护代价约 5 行，外加 2 个测试。

### 2.6 成功时还可删的噪声（实测，按真实输出逐字段估算）

| 部分 | 占全部标准输出字节 | 判断 |
| --- | --- | --- |
| `snapshotPath="…"` | 网页题 4.2%，长列表题 6.9%，后台屏题 19.1%，路线题 6.2% | 被这个执行框架放大了：当前目录是仓库根，路径里有 `ultragoal\…\workspace\`。正常使用约 100 字节。文件名里的时间戳和标题可以再缩短，可选。 |
| `window id=… active focused content="full" selected` 和 `edge-windows count=2 content="notRead" …` | 合计 3.1% | 只有一个应用窗口而且已读全时，调用者用不到。可选：单窗口时省略。 |
| `scroll-until` 成功时 `match=` 里的 `matchedText`（与正文节点行重复，约 280 字符）、`"complete":true`、`"unknown":0`、`windowId`、`field`，以及 `stop=found` | 每次约 350 字节，约占这条输出的 6% | 可选：成功时只留 `refs` 和 `matches`。 |
| `change=changed at content,eNNN:actions` 里的 `:actions` 等字段名 | 很小 | 不必处理。 |
| 正文首行和正文头重复的缺口信息 | 很小 | 文件也需要，保留。 |

节点行本身占 57%–85%。头部和元数据已经接近下限，再往下省要动投影规则（比如祖先和子节点同名重复显示），这不在第 15 条范围内。**去向**：可选建议，进下一轮。

---

## 3. 首行格式与退出码

### 3.1 现在的首行形态（源码推断，`ui_output.py:37-316`；实测出现的形态见下）

`<命令>: <主结果>[; 原因][; code=…][; 名字变化行][; change=…][; cached|fresh][; dispatchStarted=…][; 各命令事实][; match=…][; observationWarning=…][; Observation: unavailable…][; snapshotPath=…/imagePath=…][; 各类 Error 字段][; 结果未知的固定指引][; 本机连接未关好提示][; 设备和会话来源]`

20 次运行里出现过的主结果：`done`（共 288 次）、`not sent`（6 次）、`failed`（2 次）。**没有一次是"完成但退出码非零"**（实测，`cli-index` 退出码对首行），所以这类路径全部只能靠源码推断和本机复现。

### 3.2 `f401deb` 修的回归

- 根因：`545d095` 为落实第 15 条"成功时不打印诊断说明短语"，把原因的打印条件加成 `result.result != "done"`（`git log -S` 定位）。设备端在 UI 清理没有确认时，给完成的结果写入原因"UI cleanup unconfirmed; ask the user to use Stop idle executor, then Refresh status on the device: …"，并把退出码设为 3（`android/…/AutomationCleanup.kt:73-81`、`UiRun.kt:143-148`）。这条原因从此不再打印。`notifications list` 完成加退出码 1 时的原因也一样丢了。测试 `test_nonzero_notification_result_keeps_read_facts[done-1]` 抓到了这个问题。
- 修法：条件改成 `result.result != "done" or result.exit_code() != 0`（`ui_output.py:58`）。本机复现（`exp4.py`）：`click` 完成加退出码 3 时，首行恢复了这条指引。
- **修得不完整**：`fill` 有单独的分支，`data.fill` 是字典且没有 `error` 时一律不打印原因（`ui_output.py:60-67`）。本机复现：`fill: done; fill=done, verification="matches"; cleanupError={"code":"uiCleanup","message":"worker gone"}`，"请用户在设备上停止空闲执行器"那句指引丢了。`fb75afc` 里就是这样，所以是旧已有之。影响：调用者仍能看到 `cleanupError`，也知道动作已经完成，不会重做；只是缺了下一步的操作指引。**性质**：确定缺陷（小）。**去向**：进下一轮。改法是在 `fill` 分支里也放行"退出码非零"，一行代码加一个测试。

### 3.3 "完成加退出码 1／3"的来源，调用者能否看出"已做过，不要重做"（本机复现，`exp3.py`、`exp4.py`）

| 来源 | 退出码 | 首行 | 能否分清 |
| --- | --- | --- | --- |
| 必需的正文文件存不成（摘要、`--no-print`、`--json`、`--out`、`--raw`） | 1 | `click: done; …; snapshotError="…"` | 能 |
| 设备端 UI 清理未确认 | 3 | `click: done; UI cleanup unconfirmed; ask the user …; cleanupError={…}` | 能（`fill` 例外，见 3.2） |
| 借用粘贴恢复失败 | 1 | `type: done; effect=matches; paste=attempted; restore=failed; restoreReason="…"` | 能 |
| 借用粘贴恢复未知 | 3 | `type: done; effect=matches; paste=attempted; restore=unknown` | 能（没有解释文字，但事实齐全） |
| 本机连接没关好 | 3 | `click: done; local connection cleanup unconfirmed; inspect with androidctl daemon status; localCloseError="…"` | 能 |
| 标准输出写失败 | 1 | 标准错误输出 `delivery: done; …` | 能 |

指引 `skills/androidctl/SKILL.md:106` 写着 "Do not repeat a done action for an unchanged/partial observation or nonzero exit"，与上表一致。退出码 3 与"结果未知"共用一个码，只看退出码的调用者会当作"未知，不要重做"处理，方向是安全的。

### 3.4 有没有路径把做过的动作报成失败或未发出——没有找到（源码推断）

- 交付环节只改写 `snapshot`、`find`（`ui_client.py:1017-1028`、`1092-1099`）和 `screenshot`（`ui_client.py:1029-1031`）的主结果，三者都是只读命令。
- `deliver` 的异常只写 `deliveryError`（`ui_client.py:757-758`），`print_delivery` 的异常带着原结果抛出 `CommandFailure`（`ui_client.py:1101-1104`），都不会改主结果。
- 参数阶段的拒绝（`CliCommand.invoke` 把 `Fault` 转成"未发出"，`cli_options.py:137-142`）发生在连接之前。设备回复后才出现的 `Fault` 在 `UiClient.request` 里就转成了"结果未知"或"拒绝"（`ui_client.py:169-176`）。
- 反方向（把没执行的报成完成）：`find` 新采失败会改成 `failed`（`ui_client.py:636-649`）；新采没有命中但样本不完整时报完成，并带 `fresh` 和缺口标记，符合第 11 条。

### 3.5 一件交付物一个诊断字段（第 18 条后一半）——已核实，有一个小瑕疵

- `historyError`、`snapshotWarning` 在主机源码、指引、契约里清零（`grep`，只剩 docs 仓库 `reference/README.md:73` 里的说明文字）。同一次保存失败只写 `snapshotError`；正文已打印时在同一字段里追加 "; automatic file unavailable; full observation follows in stdout"；独立的历史清理失败用 `historyCleanupError`（本机复现）。
- 瑕疵 1：`snapshot`/`find` 因保存失败报失败时，`result.reason` 就是 `snapshotError` 原文，首行先打原因、再打 `snapshotError=`，同一段（含绝对路径，约 200 字节）出现两遍（本机复现，`exp1.py`）。`ui_output.py:287-293` 已经对 `snapshotProblem` 做了去重，对原因没做。**去向**：不必处理，或进下一轮顺手改，三行代码。
- 瑕疵 2：同一个 `snapshotError` 既用于投影错误（`ui_client.py:808`、`823`）也用于保存错误（`ui_client.py:916`、`945`），后写的会覆盖先写的。目前没有两者同时出现的真实样本。**性质**：证据不足，不必处理。

---

## 4. 选项组合与参数阶段拒绝（本机复现，`S/r7/venv7/bin/androidctl`，没有设备）

- `--raw` 与 `--all`、`--depth`（含 `--depth 0`）、`--max-chars`、`--interactive`、`--boxes`、`find` 的 `--context` 同写，以及 `snapshot REF --raw`：都在参数阶段拒绝，退出码 2，多个冲突一次列全，例如 `cannot combine with --all, --depth; use androidctl snapshot --cached --raw, or omit --raw for a projected view`。这与第 23 条及其第三次答复一致。只判"命令行上显式写了"（`ui_commands.py:27-51`），默认值不会误伤。已核实无问题。
- `--raw --no-print`、`find QUERY --raw --out a.txt` 照常接受，到需要设备时才失败。`--json` 与显式 `--print` 同写被拒，并给出两种可照抄的写法。已核实无问题。
- 动作命令、`wait`、`scroll-until` 上写 `--print` 或 `--all` 会被拒绝，"No such option" 后面附 `--help` 的写法。`notifications --no-print list` 被拒，并给出正确位置的写法。已核实无问题。
- 小问题：`--raw` 冲突时给的写法一律是 `snapshot --cached --raw`。用户原本要的是新采（例如 `snapshot --window 1 --raw --all`）时，这条写法悄悄换成了读缓存，读到的可能是旧样本，甚至没有样本。可选建议，按是否 `--cached` 给 `snapshot --raw` 或 `snapshot --cached --raw`。去向：进下一轮，约 3 行。
- 小问题：`snapshot --print --no-print` 不报错，后写的生效。不必处理。
- `snapshot --help` 里 `--print/--no-print` 的说明没提"默认超过 64 KiB 只打摘要"，只在指引里有。可选，在帮助里补一句。

没有发现自相矛盾的组合。

---

## 5. 首轮 60 个测试失败说明了什么

- 首轮质量日志 `ultragoal/…/r6-final-quality/quality.log` 开始于 13:23 UTC，`f401deb` 提交于 14:06 UTC，所以首轮测的是 `642799b` 那一版。结果是 60 个失败、1,964 个通过；第二轮 `quality-02.log` 是 2,028 个全过。
- 分类（逐条读断言）：
  - **真实回归 1 个**：`test_notifications.py::test_nonzero_notification_result_keeps_read_facts[done-1]`。它在 `f401deb` 里**没有被改动**，靠源码修复转绿。同一根因也影响所有"完成加原因加退出码非零"的界面结果（见 3.2）。
  - **测试没跟上有意改动 59 个**：
    - `test_observation_files.py` 约 51 个，断言旧的头部元数据（`sequence`、`startedMs`、`display`、`missingSummary`、`windowsSource`）、旧字段 `historyError`、旧警告行 "Warning: automatic observation file was not saved"、旧首行 `cached; sequence=7; sample=…`；
    - `test_node_actions.py` 4 个和 `test_text_input.py` 1 个，断言 `prepare_ui` 内部的冗余校验，那些校验在 `545d095` 已删除，现在由命令行参数解析拦住；
    - `test_activity_intents.py` 1 个，断言 `open` 成功时打印启动细节，与第 15 条相反。
- **改测试有没有盖掉真问题**：我逐处看了 `f401deb` 的测试差量。新断言大多在旧断言的位置改成"文本里没有、`--json` 或 `--raw` 里有"，事实层面没有放松。`test_delivery_timing…` 从定值 155 改成"30 + 实际耗时"，合理。`prepare_ui` 的校验测试改到命令行层并确认没有发出请求，覆盖面没有缩小。**没有发现被盖掉的真问题。**
- **耦合程度**（估算，用正则粗分主机测试函数）：499 个测试函数里约 326 个（65%）直接断言渲染出的文字（`stdout`、`readouterr`、`result_line`、`render_observation`、`.body`）；`test_observation_files.py` 是 42/43，`test_cli_arguments.py` 35/36，`test_result_output.py` 34/38。这次一个"头部合成一行"的改动就让约 50 个测试同时失败。
- **含义**：
  1. 每次调输出格式，都要成批机械改测试。改的人容易一并"改到通过"，真回归就会混在里面被放过。这次真回归是靠一个恰好没被改的测试抓到的。
  2. `545d095`、`642799b` 提交时显然没跑主机全量测试。合入流程要整理成 6 到 10 个提交，**每个提交都应能单独通过全量测试**，否则对分查找会失效。这是给合入流程的提醒，不挡合入。
  3. 建议（进下一轮）：事实断言改为断言 `Result.data` 或 `--json`；渲染文字只留少量按命令的"金样"测试，每类命令一份，改格式时整份更新、人工看差量。代价是一次性改约 300 个断言，可以分批做。

---

## 6. 重复维护的格式

同一事实现在由几处各自生成（源码推断）：

| 事实 | 首行（`ui_output.result_line`） | 正文头（`ui_view.metadata/header`） | 摘要 | `--json`（`Result.data`） | 保存文件 | `--raw` 和契约样本 |
| --- | --- | --- | --- | --- | --- | --- |
| 采样缺口 | `observationWarning=`（由 `snapshotCompleteness` 拼） | `observation {…}`（由树另拼） | 复用正文头 | `snapshotCompleteness` | 同正文 | `tree.partial/missing` |
| 采集编号 | 无 | `capture=` | 复用 | `capture` | 同正文 | — |
| 缓存或新采 | `cached`/`fresh` | `snapshotCached=true`（仅 `--all`） | 复用 | `snapshotCached`、`findRefreshed` | — | — |
| 保存路径 | `snapshotPath=`（相对路径） | — | 首行 | 绝对路径 | — | — |
| `find` 命中数 | `matches=`、`offscreenMatches=` | `find matches=… offscreen=…` | 复用 | `matches`、`offscreenMatches` | 同正文 | — |

- 摘要复用了正文头和渲染函数，这一处没有重复，做得对。真正重复的是"首行"和"正文头"两套格式化，外加 `--json` 字段，每个事实写两三遍。
- **首行呈现**从第六轮评审时的 245 行涨到现在约 280 行（`ui_output.py:37-316`），按命令和主结果写了几十个特判。`f401deb` 修的回归（全局"成功不打原因"的条件和 `fill`、`clipboard.get` 的特判互相作用）和 3.2 发现的 `fill` 遗漏，都是这种写法的直接症状。所以第六轮"合入后把首行呈现改成按命令的字段表"的建议**现在更有必要**。
- **代价估算**：把首行改成"字段名 → 格式、何时打印（成功、失败、退出码非零）"的表，约 1–2 天；会改动 `test_result_output.py` 的大部分断言和别处约 30 个首行字面断言；字段顺序可能小变，调用者不依赖顺序，风险低。"首行和正文头合并为一处生成"收益较小，不建议。**去向**：进下一轮，排在第 5 节的测试改造之后或同时做。

---

## 7. 第 18 条：强杀命令行进程的实验

材料：`ultragoal/…/42-r4-decisions/r6-runtime-preparation.md` 第 60–77 行；脚本 `runtime/api34/r6-cli-disconnect.py`（只读，没有执行）；原件 `runtime/api34/commands/r6-18-015-*`、`r6-18-016-*`。

- **实验做了什么**：用 `--json` 启动一条 30 秒的 `wait`（等一段不存在的文字），用 `status` 确认会话的 `ui` 资源已被占用后，按进程号强杀这条命令行进程；随后依次发 `status`、5 秒上限的 `snapshot`、35 秒上限的 `snapshot`、`status`。再用 25 秒的 `shell` 做同样的实验。用的是改动前的 `a27` 物料。
- **结果**（实测）：强杀后，本机服务和设备端的 `wait` 继续跑到原截止时间。5 秒的 `snapshot` 等了 5.188 秒后报 `not sent`，`code=connectionBusy`，原因 "Connection is occupied within this RPC budget"；35 秒的 `snapshot` 等了 23.797 秒后成功。`shell` 那条不占界面通道，界面命令随时可用，原任务正常结束。
- **能否回答第 18 条的问题**：只能回答一半。第 18 条的前一半问的是"本机连接没关好"：命令结果已经拿到，只是本机连接的关闭报错（`close_preserving_result`，`local_call.py:36-47`）。这种情况下退出码设为 3（`protocol.py:80-82`），问题是"改成 0 会不会让本机服务占着会话"。强杀进程测的是"调用者突然消失"，是相邻但不同的情形。实验不能证明自然发生的关闭报错之后会话一定空闲，所以实施方"保持原退出码"是按"确认不了就保持现状"执行的，结论处理正确（`r6-runtime-preparation.md:75` 也这样写了）。
- **"命令行退出后仍占到原截止时间"值不值得处理**：值得，但不挡合入。
  - 触发条件：调用者的执行框架有自己的工具超时，或者用户中断，杀掉了一条长 `wait`、`scroll-until`、动作命令。指引允许 `wait` 最长 60 秒。
  - 影响：之后同一会话的每条界面命令都会在自己的时限内排队，等不到就报 `connectionBusy`，最长约 60 秒。原因文字里没有占用者，也没有预计释放时间。码表 `contract/reason-codes.json:30` 写的是 "Wait for the other session's request to release…"，而这里占用的是**同一个会话**的孤儿请求，指引不符。调用者可能误以为有别的工具在抢设备。
  - 20 次真实运行里没有出现（实测）。
  - 最小修正方向：
    1. 本机服务发现客户端断开时，主动取消**只读**的 `wait`、`snapshot`、`find`。动作类请求不取消，免得半途中断。
    2. 退一步，`connectionBusy` 的原因里写明占用者（本会话或其他会话）和预计释放时间，码表按主体分两句。
  - 维护代价：第 2 条约 20 行加码表一句；第 1 条要动本机服务的请求生命周期，约 50–80 行加测试，并需要设备验证。**性质**：设计取舍。**去向**：进下一轮。

---

## 8. 第 11 条在主机端的部分（`find` 命中 0 条时自动新采）——已核实无问题

- 新采在保存之前（`ui_client.py:614-656` 在 `run_request` 内完成，之后才 `deliver`），第一次 0 条的正文不会写进历史。
- 新采沿用缓存样本的范围（`"scope": tree["selectionScope"]`），不需要新参数。
- 共用一个时限：第二次请求里的 `timeoutMs` 虽然沿用原值，但传输层会按剩余时间截短（`transport.py:473-474`）。
- 新采失败时原因写作 "Cached sample had zero matches; fresh collection failed: …"，主结果为 `failed`。首行有 `fresh` 标记（`ui_output.py:90-94`）；实测 7 次 `find: done; fresh; …`。
- 副作用：首行的 `fresh` 也会在"上一条动作没有稳定导致重采"时出现，两种重采不分，对调用者无害。

---

## 9. 发现汇总与分类建议

| 编号 | 发现 | 依据 | 性质 | 去向 |
| --- | --- | --- | --- | --- |
| 1 | 两次真实触发里，摘要把成本挪去读文件：模型收到的字节和上一轮一样多，多一回合 | 实测 | 设计取舍，收益为零 | 进下一轮（靠 2、3、4 改善，不撤上限） |
| 2 | `--all` 下无名标准动作照打，占 32%–39% 字节；去掉后维基页正文 843→406 KB | 实测、本机复现 | 设计取舍 | 进下一轮 |
| 3 | `find` 省略记录一律写 `snapshot --cached --all`，覆盖定向读法；0 条时没有意义；直接引发 1 次触发 | 实测、源码 `ui_view.py:896-898` | 确定缺陷（指引误导） | 进下一轮 |
| 4 | 摘要缩小提示固定为 `snapshot --cached --depth 2`，与命令无关；没有文件大小；"65536"写死 | 本机复现 | 可选建议 | 进下一轮 |
| 5 | `fill` 完成加退出码非零时仍吞掉原因（UI 清理指引） | 本机复现、源码 `ui_output.py:60-67` | 确定缺陷（小，旧已有之） | 进下一轮 |
| 6 | `snapshot`/`find` 保存失败时首行原文重复两遍 | 本机复现 | 可选 | 不必处理或顺手改 |
| 7 | 孤儿请求占着会话直到原截止；`connectionBusy` 原因和码表把同会话说成"另一个会话" | 实测（实施方原件）、源码 | 设计取舍 | 进下一轮 |
| 8 | 主机测试约 65% 断言渲染文字；两个提交在测试失败状态下提交 | 估算、实测日志 | 维护问题、过程问题 | 进下一轮；合入整理时每个提交跑全量 |
| 9 | 首行呈现约 280 行，特判互相作用已产生 1 个回归和 1 个遗漏；"字段表"建议更有必要 | 源码 | 可选建议 | 进下一轮 |
| 10 | 成功时剩余噪声：单窗口时的窗口行和边缘窗口行约 3%；`scroll-until` 成功的 `match=` 细节约 6% | 实测 | 可选建议 | 进下一轮 |
| 11 | `--raw` 冲突时给的写法一律是读缓存，新采意图被改写 | 本机复现 | 可选建议 | 进下一轮 |

**挡合入：无。** 这条链上的每个发现都不会造成误操作、重复执行或资源损失，也不与裁决相反。

## 10. 已核实无问题

- 阈值 65,536 按 UTF-8 字节、严格大于；取低端有依据：低于执行框架约 89 KB 的截断点。
- 上限判断同时进入了"打印即补存"、`snapshot`/`find` 的主结果、退出码三处；摘要不算"正文已打印"。
- 摘要包含首行、采集编号、缺口行、`find` 命中行、提示条、路径（在首行）、一行缩小写法。
- `--json` 不受上限影响；`--out` 时路径指向 `--out` 的文件；显式 `--print` 越过上限，文件存不成时仍只是警告；`--no-print` 只保存。
- 超限而存不成文件时：`snapshot`、`find` 报失败；动作、`wait`、`locate`、滚动查找、不带子命令的 `notifications` 报完成、退出码 1；结果未知仍为 3。
- 没有路径把已执行的动作报成失败或未发出，也没有路径把未执行的报成完成。
- 一件交付物一个诊断字段；`historyError`、`snapshotWarning` 已清零。
- `--raw` 默认只保存；`--raw --print` 打印整份 JSON；`--raw` 与六种形状选项及 REF 在参数阶段拒绝并给可照抄的写法。
- 默认文本里 `action-NNN` 清零（322 次调用）；采集编号、缺口标记、提示条都在；同类命令字节下降 25%–60%；A4 的成功过程计数已省略；`open` 成功只留显示编号和界面树。
- `f401deb` 的修法在通用路径上正确；对应的测试我跑过：`test_result_output.py`、`test_ui_reading_delivery.py`、`test_notifications.py` 共 169 个通过、1 个跳过。
- `f401deb` 改动的测试没有盖掉真问题。
- 第 11 条的主机部分：新采在保存前、同一时限、同一范围、有 `fresh` 标记、失败原因写清。
- 第 18 条前一半保持原退出码的处理，与"确认不了就保持现状"一致。

## 11. 未能核实的事项

- 所有"完成但退出码非零"的路径，在 20 次真实运行里一次都没发生；本节结论来自源码和本机复现，没有真实样本。
- 动作命令、`wait`、`locate`、滚动查找、不带子命令的 `notifications` 真实超限的情形没有发生；我用降低阈值的方式复现了逻辑，但没有真实页面上的样本。
- 在 `wait` 或动作命令产生摘要之后，摘要提示的 `snapshot --cached` 读到的是否正是同一份样本：取决于设备端的缓存规则，没有核实（Kotlin 侧未读，没有设备）。
- `fill` 完成加 UI 清理未确认时，设备端是否真的把指引放在顶层 `reason` 里：我是按 `withUiCleanup`（`AutomationCleanup.kt:73-81`）对所有结果的写法推断的，用的是合成输入。
- 自然发生的"本机连接关闭报错"之后会话是否空闲：实施方的实验没有覆盖，我也无法复现。
- 主机测试"约 65% 断言文字"是正则粗分的估算，不是逐个阅读的结果。
