# 专题 103：本轮主机（Python）端差量的逐行评审与动手验证

评审对象：`f401deb` 相对 `fb75afc` 的 `androidctl/src` 差量（10 个文件），以及对应的主机测试差量。
工作目录与脚本：`S/agents11/103/`（`exp1.py`–`exp10.py`，输出 `exp1.out`–`exp4.out`），`S` 指 COMMON.md 里的 scratchpad 根目录。

## 摘要

- 本轮主机差量的主干是对的。我逐行读了 `ui_client.py`、`ui_output.py`、`ui_view.py` 全文和其余 7 个文件的差量，并在本机复现了这几件事，都与第 14、11、18、23、25 条裁决一致：输出上限按 UTF-8 字节判定（65,536 字节照打全文，超过就打摘要，3 字节和 4 字节字符都试过）；文件内容与全文逐字节相同；保存失败时 `snapshot`、`find` 报失败，动作命令和 `wait` 保持完成、退出码为 1；`find` 缓存 0 条时新采一次，新采的各种结局都核过；`--raw` 的拒绝组合有效；`session open` 和 `session takeover` 的新默认行为正确。
- 没有发现必须挡合入的缺陷。确定的缺陷有 4 条，都进下一轮：
  1. `reveal` 已完成但目标仍在屏外时，默认文本不再给出任何直接信号（`f401deb` 有意把测试反过来了）。
  2. `find` 判断"0 条要不要新采"的那段代码没有异常保护。树的形状异常时，结果从"失败、退出码 1"变成整条命令"结果未知、退出码 3"。
  3. 配额清理失败时，同一组文件路径同时写进 `snapshotError` 和 `historyCleanupError`，首行可达约 31 KB。
  4. 带标签的平台区段动作在默认文本里被丢掉，指引写的只是"省略无名的"。
- 另有 2 条疑点要用户知道：默认头部不再显示锁屏和亮屏状态，这超出了第四节第 9 处列出的可删范围；保存要在请求时限内完成，超限的观察碰上时限就只剩摘要。
- 静态检查：`mypy`（含 linux、darwin 两个平台分支）、`ruff check`、`ruff format --check` 全部通过。我跑了约 60 条参数错误，没有一条把回溯（traceback）漏给调用者。把以往各包里的 121 份原始观察喂给新渲染器，在 9 种视图下都没有崩溃；对现行格式的观察，摘要最大只有 455 字节。
- 测试方面：新逻辑的关键分支大多有测试。有一处断言对失败场景是自动成立的（`"fresh" in stdout`），另外新采在"结果未知"和"树交付失败"两种结局下没有测试。

## 一、发现清单（按严重程度）

### F1 `reveal` 已完成但目标仍在屏外时，默认文本不给信号

- **性质**：确定（主机端本地复现）；属于本轮有意的改动带来的副作用。
- **位置**：`ui_output.py:45`（`concise_success`），以及 `ui_output.py:532-544`（`input_facts` 在成功时整组省略，其中包括 `reveal`）。测试 `test_node_actions.py:158-162` 在 `f401deb` 里从 `"reveal=" in lines` 改成了 `not in lines`。
- **复现**：`exp5.py` 用契约样本 `reveal-offscreen`（`reveal.visible=false`）跑了两个候选：
  - `fb75afc` 打印：`reveal: done; reveal={"visible":false,"bounds":[0,100,20,100],...}`
  - `f401deb` 打印：`reveal: done`
- **后果**：第 13 条之后，默认视图会隐藏屏外节点（`ui_view.py:372-374`），所以动作后的正文里也找不到这个目标。调用者只能从"正文里没有这个目标"或 `change=unchanged` 去推断。
- **真实任务里**：网页题第 1 次运行出现过一次 `reveal`，那次成功了。首行是 `change=changed at content,e950416952:visible,bounds`，目标也出现在正文里，所以没有受影响。失败的情形在 20 次运行里没有出现。
- **最坏结果**：下一步 `click` 被拒为 `notReady: target is not visible`（未发出，无害），多花一个回合。
- **修正方向**：成功时仍省略 `reveal` 的细节，但在 `visible` 不为 true 或 `state` 不是 `observed` 时，首行补一个短标记（例如 `reveal=still offscreen`）。约 3 行代码加 1 条测试。
- **去向**：进下一轮。

### F2 `find` 的"0 条就新采"判断没有异常保护

- **性质**：确定（主机端本地复现）。
- **位置**：`ui_client.py:618-634`。`ui_view.project(...)`、`ui_view.find(...)` 和 `tree["selectionScope"]` 都在 `run_request` 里直接执行，外面只有 `try/finally`。正常的渲染路径则会把 `Fault`、`KeyError`、`TypeError` 收成 `snapshotError`（`ui_client.py:818-819, 890-891`）。
- **复现**：`exp8.py` 构造了一个动作字段形状异常（`actions=[{"id":16}]`）的树：
  - 首次采样是新采时：`find: failed; unsupported format string ...; snapshotError=...`，退出码 1，符合预期。
  - 首次采样是缓存时：异常从 `execute()` 逃出，被 `app.py:1035-1040` 的通用兜底收成 `androidctl: UNKNOWN; TypeError ...`，退出码 3，也不保存文件。
- **触发条件**：设备送来主机投影处理不了的树，或者树里缺 `selectionScope`。两种都只会在版本错配或设备端缺陷时出现。121 份真实观察和 20 次运行里都没有发生（`exp4.out`）。
- **后果**：一条只读命令报成"结果未知"。按指引，调用者会改用 `snapshot` 恢复，不会造成重复动作。
- **修正方向**：用同样的 `except (Fault, KeyError, TypeError)` 包住判断；出异常时不新采，交给后面的渲染去报 `snapshotError`。另加 1 条测试。
- **去向**：进下一轮。

### F3 配额清理失败时，同一组路径写进两个诊断字段，首行可达约 31 KB

- **性质**：确定（主机端本地复现）；差量放大了已有的行为。
- **位置**：`history.py:135-148`（`quota_error()` 和 `warning()` 都把全部失败路径拼进去），以及 `ui_client.py:954-959`（把两者分别写成 `snapshotError` 和 `historyCleanupError`）。
- **复现**：`exp10.py` 放了 150 个旧文件并让删除全部失败，得到 `snapshot_error` 15,813 字节、`warning` 15,786 字节。`exp1.py` 的 `unlink` 情形确认两个字段同时出现在首行：`snapshotError="History quota remains occupied. Close handles to retained files: …(150 条)"`，后面还有 `historyCleanupError="History cleanup incomplete; retained: …(同样 150 条)"`。
- **触发条件**：自动历史已满，而且旧文件删不掉。在 Windows 上，旧文件被别的进程占着就会这样。
- **后果**：首行膨胀；同一件事报了两遍，与第 18 条"一件交付物一个诊断字段"的精神不符。退出码和主结果是对的。
- **修正方向**：因配额失败时，只保留 `snapshotError`，或者把 `historyCleanupError` 去重；路径列表只列前几条，再给一个总数。几行代码。
- **去向**：进下一轮。

### F4 带标签的平台区段动作在默认文本里被丢掉

- **性质**：确定（主机端本地复现）；同时与指引文字不一致。
- **位置**：`ui_view.py:117-126`。`platform_action and identifier not in ACTION_NAMES` 时一律跳过，不看有没有标签。`is_target()`（`ui_view.py:174`）仍把只带这类动作的节点当成目标、打印引用。
- **复现**（`exp6.py`）：

  | 节点上的动作 | `fb75afc` | `f401deb` |
  |---|---|---|
  | 16908375（DRAG_START），标签 "Drag item" | `action-label="Drag item"` | `text e1 "Body"`，没有任何动作 |
  | 16908356（SHOW_TOOLTIP），无标签 | `action-name="action-1020044"` | 只剩 `text e1 "Body"` |

  加 `--all` 后两者都能看到。
- **指引**：`skills/androidctl/observations.md` 写的是 "Unnamed platform-standard actions are omitted"，只说省略无名的，没说带标签的也省。
- **后果**：少见。节点带着引用却看不出能做什么，调用者要加 `--all` 才看得到。
- **修正方向**：二选一：带标签时保留；或者把指引改成"平台区段动作在默认文本里一律省略"。
- **去向**：进下一轮（可读性）。

### F5 渲染把具体的交付错误覆盖成泛泛的一句

- **性质**：确定（源码推断，`exp3.py` 的 `tree-delivery-failed` 情形部分复现）；`fb75afc` 起就有，不是本轮引入的。
- **位置**：`ui_client.py:799-800` 写 `facts["snapshotError"] = "Node observation is unavailable"`，`ui_client.py:731` 再用 `result.data.update(...)` 覆盖掉 `attachments()` 早先写入的具体错误（例如 `Attachment hash mismatch`、`Request budget exhausted before artifact delivery`，见 `ui_client.py:259-261`）。
- **后果**：第 18 条之后每件交付物只剩一个诊断字段，带 `find` 或 REF 时，首行看不到具体原因。具体原因还在正文的 `artifact` 记录和 `--json` 的 `artifacts.tree.deliveryError` 里，所以损失有限。
- **修正方向**：改用 `setdefault`。1 行。
- **去向**：进下一轮。

### D1（疑点，设计取舍）默认头部不再显示锁屏和亮屏状态

- **位置**：`ui_view.py:620-634`（`metadata()` 不再整理 `display`）和 `ui_view.py:903-935`（`header()` 只输出 `partial`、`missing`、`unavailable`、`limits`、`treeProblem`）。
- **复现**（`exp9.py`）：树的 `display` 带 `keyguardShowing=true`、`deviceLocked=true`、`interactive=false`、`displayState=1` 时：
  - `fb75afc` 头部有一行 `observation {"display":{"displayState":1,"interactive":false,"keyguardShowing":true,"deviceLocked":true,...}}`。
  - `f401deb` 头部只剩 `androidctl capture="w:7"`。
- **与裁决的关系**：第四节第 9 处列出的可删项是"连接号、起止时间戳、样本年龄和非 `--boxes` 时的窗口几何"，锁屏和亮屏状态不在其中。实施方在内部 docs 的 `reference/ui.md:96` 写了"display：默认头部不展开；raw 保完整事实"，所以这是有意为之，但没有回到用户那里确认。
- **后果**：锁屏时，调用者只能从窗口包名（系统界面）推断；输入命令会以具体原因拒绝，不会误操作。
- **建议**：请用户确认。如果要恢复，只在值不是默认值时打印 `locked` 或 `screenOff` 这类短标记即可。
- **去向**：进下一轮，或不必处理。

### D2（疑点，已有行为）超限观察碰上请求时限时只剩摘要

- **位置**：`ui_client.py:915-918` 和 `history.py:181-183`：自动保存在请求时限用完后直接放弃。
- **复现**（`exp7.py`）：时限已过、正文超限的 `click` 打印 `click: done; snapshotError="Request budget exhausted before saving snapshot file"`，退出码 1。没超限时照旧打全文、退出码 0。
- **背景**：这是第 14 条答复和第四节第 1 处列出的"四种存不成"之一，符合裁决。真实运行里设备通常在时限前返回，20 次运行没有出现。
- **去向**：不必处理。若要改，可以给本地保存单留一小段不受设备时限约束的余量。

### R（可读性，不影响结果）

1. 新采失败时首行仍带 `fresh` 标记（`ui_output.py:91-94`）。例：`find: failed; Cached sample had zero matches; fresh collection failed: …; code=timeout; fresh; Observation: unavailable`（`exp3.out`）。改成只在拿到新样本时打 `fresh` 更准确。
2. `snapshot` 保存失败时，首行的 reason 和 `snapshotError` 内容相同，整句重复一次，例如 `snapshot: failed; Cache path is not a directory…; snapshotError="Cache path is not a directory…"`（`exp1.out`）。
3. 名字变化被拒时，"not sent"出现三次：主结果、reason 里的 "action was not sent"、`nameChange` 那一行结尾的 "; not sent"（`ui_output.py:76-85`）。
4. 死代码：`ui_view.py:916` 的 `missingSummary`，`metadata()` 已不再生成；`ui_view.py:923` 的 `limits`，树里没有这个键（设备端只在等待谓词里写 `limits`）。测试辅助 `ui_test_support.py` 里也还在解析 `windows-source`、`events` 和 `event`。
5. 指引 `observations.md` 写 "Cached reads retain sample identity and age"，但默认文本已不显示样本年龄（`age=` 和 `snapshotAgeMs` 都删了），年龄只在 `--json` 里。建议改指引的措辞。
6. 摘要里那句缩小范围的写法固定是 `androidctl snapshot --cached --depth 2`。对 `snapshot REF --all` 这样的调用，它丢掉了 REF。能用，但不是最贴切。

## 二、逐文件核对结果

### `ui_client.py`（通读全文）

- **输出上限**（`:45, :733-740, :763-765, :865-889, :1086-1090`）：
  - 判定用 `len(body.encode("utf-8")) > 65_536`，所以恰好 65,536 字节照打全文。
  - 三种情况下不打摘要：`--json`、显式 `--print`/`--no-print`、`--raw`。
  - 用摘要时传给 `finish_delivery_result` 的 `print_body` 为假，于是不会走"打印即补存"。`print_delivery` 里"正文已打印"的判断也排除了摘要。第四节第 1 处的两处风险都落实了。
  - 实测（`exp2.out`）：65,536 字节照打全文；65,538 和 65,539 字节打摘要；保存的文件与全文逐字节相同。
  - 摘要包含第 14 条要求的各项：首行、带 `capture` 的溯源行、缺口行、`find` 命中摘要行、文件路径（在首行的 `snapshotPath` 里）、缩小范围的写法。
  - 显式 `--out` 时不另存历史副本，路径指向 `--out` 给的文件（`exp7.py`）。
- **保存失败的落点**（`exp1.out`）：
  - 工作区不可用或配额删不掉时，`snapshot`、`find` 报 `failed`，退出码 1。
  - `click`、`wait`、`locate`、`scroll-until` 保持 `done`，退出码 1，首行带 `snapshotError`。
  - 正文没超限时，照打全文、退出码 0，`snapshotError` 末尾写 "full observation follows in stdout"。
  - 显式 `--out` 写失败时报 `failed`，退出码 1。
  - `historyError` 和 `snapshotWarning` 已从源码和测试里清掉。
- **`find` 新采**（`exp3.out`）：
  - 第二次请求带 `cached=False`，范围沿用缓存样本的 `selectionScope`，在保存之前发出，与同一条命令共用时限。
  - 新采后命中：`done`、退出码 0、`fresh`，只保存一份文件。
  - 新采后仍 0 条：`done`、退出码 0。
  - 新采后 0 条且采样不完整：`done`、退出码 0，首行带 `observationWarning="partial"`。
  - 新采的树交付失败、结果未知、未发出：都报 `failed`、退出码 1，原因以 "Cached sample had zero matches; fresh collection failed:" 开头。
  - 以上与第 11 条答复一致。唯一的问题见 F2。
- **`close_preserving_result` 重构**（`:653-656`）：与旧的内联写法等价。新采之后，关连接失败会记到新采的结果上，这是对的。
- **`prepare_ui` 删掉的校验**：
  - `select`：REF 和起止位置已由 typer 参数（`min`/`max`、必填 REF）把关，空 REF 由 `normalized_ref` 拒绝。
  - `range`：REF 必填。
  - `reveal`：命令行上已不存在动作选择的选项。
  - 截图区域：截图命令没有那些选项。
  - 实测都在连接前拒绝（见第三节）。`select e1 5 2`（起点大于终点）照旧发给设备；这是有意保留的"倒序请求"测试，不是遗漏。

### `ui_output.py`（通读全文）

- `f401deb` 的修正（`:56-58`）：主结果是 `done` 但退出码不为 0 时，首行也打印原因。我检查了其它命令的渲染：`Output.heading`、`app.render_summary`、`file_jobs` 都是无条件打印 reason，所以这条回归只存在于界面命令的首行，已经修全。
- `nameChange`、`predicate_summary`（`refs` 只列前 20 个、`matches` 给总数）、`open` 成功时的 `display=`：字段形状都与设备端一致：
  - `UiRefRepair.kt:49-71` 总会给出 `oldName`、`ref`、`newName`；
  - `visibleTaskFact` 总带 `displayId`；
  - `landing` 总是对象。
- 滚动查找成功时去掉"页面可能还在变化"（`:335-341`）：只在命中数大于 0、未检测到加载时去掉，符合第三节乙"必须改进"的答复。
- 成功时的精简（不打印 `samplingStopped`、`inputTarget`/`point`/`gesture`/`keys`、`textLength`、`open` 的启动细节）符合第 15 条。唯一越线的是 F1。

### `ui_view.py`（通读全文）

- `project()`：显式 REF 时屏外子树也显示（`:345-356, :372`），与新指引一致。
- `folded()`：所有连接号类字符（Unicode `Pd` 类）都当作空格，在 NFKC 之后做。没有问题。
- `screen()`：事件只留提示条（toast）文字；窗口边界只在 `--boxes` 时打印；边缘窗口不再带边界。符合第 15 条。
- `header()`：见 D1 和 R4。
- `json_value()` 和 `node_line()` 不变。多字节文字与上限判定之间的配合已实测。

### `app.py`

- `session open`：不写 `--display` 时请求里不带 `displayKind`，设备端 `DisplayTarget.kt:17` 支持这种写法；`--replace` 不带 `--display` 时在连接前拒绝。
- `session takeover` 仍要求 `--display`，报错信息友好。
- `idleRemaining` 的输出：如果设备端是旧版、不带这个字段，`emit_status` 会因 `KeyError` 被通用兜底收成"结果未知"。主机和 Android 端版本是绑在一起发的，我只记作版本错配下的已知风险，不列为发现。

### 其余文件

- `job_commands.py`、`local_call.py`：只是关闭连接的样板改成共用函数，与旧写法等价。
- `local_server.py`、`state.py`、`transport.py`：原因码改名，新旧对照：
  - `internalError` 改为 `internal`
  - `identityChanged` 改为 `tlsIdentity`，加 `tls` 阶段
  - `messageLimit` 改为 `jsonLimit`
  - 观察文件和图片文件的父目录错误改为 `fileParent`

  `grep` 确认旧码在源码和指引里都已无残留，新码都在 `contract/reason-codes.json` 里；`test_reason_codes.py` 会枚举源码里的码字面量并与码表比对（本机跑过，通过）。
- `ui_commands.py`：
  - `--raw` 的拒绝只认命令行上显式给出的选项（`get_parameter_source` 为 `COMMANDLINE`），连显式写的 `--context 0` 也拒绝，这是有意的。
  - `--raw` 默认不打印；`--raw --print` 打印整份 JSON。
  - 各条报错里的示例改成调用者自己写的 REF。

## 三、动手验证记录

所有命令都在 `S/agents11/103/ws` 下执行，`HOME` 指到工作目录，用 `S/r7/venv7`。

**参数错误（约 60 条，节选）**：全部是单行的 `…: not sent; …; code=arguments`（或 `fileParent`、`fileExists`），退出码 2，没有回溯。

```
$ androidctl snapshot --raw --all
snapshot: not sent; --raw reads the whole observation and cannot combine with --all; use androidctl snapshot --cached --raw, or omit --raw for a projected view; code=arguments   [exit 2]
$ androidctl snapshot e15 --raw            -> cannot combine with REF          [exit 2]
$ androidctl find x --raw --context 2      -> cannot combine with --context    [exit 2]
$ androidctl --json snapshot --raw --boxes -> 单个 JSON 对象，code=arguments   [exit 2]
$ androidctl screenshot --marks            -> Standalone screenshot has no node tree to label   [exit 2]
$ androidctl session open --replace        -> --replace requires --display …; argument=display  [exit 2]
$ androidctl session takeover              -> session takeover requires --display …             [exit 2]
$ androidctl snapshot --out /nonexistent/dir/x.txt -> code=fileParent  [exit 2]
$ androidctl snapshot --out exists.txt     -> code=fileExists           [exit 2]
$ androidctl reveal                        -> reveal requires an explicit REF …   [exit 2]
$ androidctl range e1 1e40                 -> must convert to a finite Android Float   [exit 2]
$ androidctl wait --any-of '[{"text":"a","bogus":1}]' -> contains an unknown condition field   [exit 2]
```

**输出上限、多字节字符和保存失败**：`exp1.py`、`exp2.py`、`exp7.py`，关键输出已在第二节列出。

**`find` 新采的六种结局**：`exp3.py`，见第二节。

**异常树**：`exp8.py`，见 F2。

**真实观察回放**（`exp4.py` 和 `exp4b.py`）：

- 取材：以往各包里 121 份 `.androidctl/snapshots/*.json`。其中 51 份是旧的逐行 JSON 格式，17 份是更早的结构，3 份是原型阶段的动作形状（列表元素是对象）；这三类不适用，剩下的是现行格式。
- 渲染时都使用新渲染器，视图有 9 种：默认、`--all`、`--boxes`、`--depth 2`、`--interactive`、`--max-chars 2000`、REF、命中查找、0 条查找，外加 `--raw`。
- 结果：现行格式的观察没有一次崩溃，`snapshotError` 只出现在树缺失的那一份上。
- 超过 64 KiB 的正文：默认视图 10 次、`--all` 13 次，最大的 `--all` 正文 1.37 MB；摘要最大 455 字节；`--depth 2` 的正文最大 3.9 KB，说明摘要给的缩小写法有效。
- 20 次真实运行的原件里，摘要出现 2 次，都是网页题上的 `snapshot --all` 和 `snapshot --cached --all`，首行和摘要的形状正确。

**静态检查**（在 `S/r7/run` 下，缓存指到工作目录，仓库未产生改动）：

```
python -m mypy --cache-dir <W>/mypycache                       -> Success: no issues found in 33 source files
python -m mypy … --platform linux  androidctl/src               -> Success (29 files)
python -m mypy … --platform darwin androidctl/src               -> Success (29 files)
python -m ruff check --no-cache .                               -> All checks passed!
python -m ruff format --check --no-cache .                      -> 94 files already formatted
pytest -p no:cacheprovider -q test_ui_reading_delivery.py test_reason_codes.py -> 43 passed
```

## 四、测试评估

- **覆盖到的**：
  - 输出上限：字节边界的 65,536 和 65,537，用 3 字节字符构造；7 个命令各自在保存成功和失败两种情况下的主结果与退出码；显式 `--print` 越过上限；摘要保留提示条文字。
  - `find` 新采：命中、不完整 0 条、失败、本来就是新采四种情形。
  - `--raw` 的 8 种拒绝组合；`--raw` 默认不打印。
  - 原因码表与源码字面量的双向比对。
  - 主结果是完成、退出码非 0 时显示原因（`test_result_output.py`）。
- **断言过松**：`test_ui_reading_delivery.py:167` 的 `assert "fresh" in response.stdout` 在失败情形里必然成立，因为 stdout 本来就含 "fresh collection failed"，所以它没有真正检验 `fresh` 标记。建议改成匹配 `; fresh;`。
- **缺的分支**：
  - 新采的结果是未知或未发出时，以及新采成功但树交付失败时（`exp3` 已手动验证行为正确，但没有回归测试）；
  - 判断新采前的异常树（F2）；
  - 显式 `--out` 配合超限正文；
  - `reveal` 已完成但目标仍不可见时的首行（现有测试断言的正是 F1 的行为）；
  - 配额失败时诊断字段是否重复（F3）。
- **在测模拟对象而非行为的**：没有发现明显的例子。新采的测试走的是真实的本机服务加模拟网关的整条链路，这一点值得肯定。

## 五、我读过并认为没有问题的文件和函数

- `ui_client.py`：
  - `requested_action`、`UiClient.request`、`execute`（`UiClient` 的方法）、`attachments`、`blob`（偏移、哈希、压缩、上界检查）；
  - `delivery_paths`、`normalized_ref`；
  - `prepare_ui`（除已说明的删除项外）；
  - `execute`（模块级函数，F2 那一段除外）、`deliver`、`render_observation`（F5 那一行除外）；
  - `save_delivery`（F3 是 `history` 一侧的问题）、`finish_delivery_result`、`print_delivery`、`result_wire`。
- `ui_output.py`：
  - `result_line`（F1 和 R1、R3 除外）、`change_summary`、`predicate_summary`、`step_facts`、`value_facts`；
  - `clipboard_get_facts`、`borrowed_paste_facts`、`selection_lines`、`structured_json`、`display_path`。
- `ui_view.py`：
  - `project`、`folded`、`find`、`matched_nodes`、`screen`、`narrow_edge_window`、`content_sections`、`metadata`；
  - `json_value`、`node_line`、`reading_view`（含 `--max-chars` 截断和省略合并）、`observation_nodes`、`attributes`；
  - `compact_actions`（F4 除外）。
- `app.py`：`session_open`、`session_takeover`、`render_summary`、`emit_status`（版本错配风险见第二节）、`main` 的消息归一化。
- `local_call.close_preserving_result`；`job_commands` 的两处调用；`local_server`、`state`、`transport` 的原因码改名；`ui_commands.UiCommand.invoke_command` 的 `--raw` 拒绝；`snapshot`、`find` 的 `print_body` 默认值；`scroll`、`scroll-until`、`target` 的报错示例。
- `history.py`：我只读未改。配额与清理逻辑本身正确，只有错误文本的体量问题（F3）。

## 六、去向汇总

| 编号 | 性质 | 去向 | 改动代价 |
|---|---|---|---|
| F1 `reveal` 仍不可见时没有信号 | 确定（本地复现） | 进下一轮 | 约 3 行加 1 条测试 |
| F2 新采判断没有异常保护 | 确定（本地复现） | 进下一轮 | 约 5 行加 1 条测试 |
| F3 配额失败的诊断字段重复且体量大 | 确定（本地复现） | 进下一轮 | 几行 |
| F4 带标签的平台区段动作被省略 | 确定（本地复现），与指引不一致 | 进下一轮 | 1 行或改指引 |
| F5 具体交付错误被泛泛的一句覆盖 | 确定（源码推断），已有行为 | 进下一轮 | 1 行 |
| D1 头部不显示锁屏和亮屏状态 | 设计取舍，未回到用户确认 | 请用户确认；进下一轮或不必处理 | 几行 |
| D2 时限用完时只剩摘要 | 符合裁决 | 不必处理 | — |
| R1–R6 | 可读性 | 进下一轮，可与整理提交一起做 | 很小 |

没有一条达到"挡合入"的门槛：它们都不会导致误操作、重复执行或数据丢失，主路径也没有变差。

## 七、未能核实的事项

- 设备端是否会在任何正常版本组合下送来缺 `selectionScope` 或动作形状异常的树（决定 F2 的实际概率）。没有设备，无法验证。
- Windows 上的路径行为：保留名、反斜杠、跨盘符时 `display_path` 退回绝对路径，以及配额文件在 Windows 上被占用的真实频率。本机是 Linux 且以 root 运行，只读目录这类权限失败无法构造，所以我用"缓存路径是文件"和"删除抛错"来模拟保存失败。
- 新采在大页面上实际用时（1.5 到 3 秒）与 5 秒默认时限的余量。只能从材料推断，没有实测。
- 主机全量测试由主评审在跑，我只跑了两个新测试文件。
