# 任务 122：回执 → 主机端交付 → 首行／正文／文件／历史 → 退出码（候选 `409e853`）

`S` = `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad`。工作目录 `S/agents14/122/`。

## 结论（先看这里）

1. **没有发现把"已执行"改写成"未执行"或把失败写成成功的路径。** 我用替身设备驱动真实 CLI 入口（`app.main`），跑了 58 个组合，每个组合分别在 `f401deb` 和 `409e853` 两版上跑。动作已发出之后，无论界面树取回失败、附件环节抛异常或被中断、渲染抛异常、历史文件写失败、stdout 断开、连接关闭失败，`done`/`failed`/`UNKNOWN` 都原样保留。退出码按 0/1/2/3 的规则只往上调。只有 `snapshot`/`find`/`screenshot` 这三个只读命令会被交付环节改成 `failed`。真实记录（82f3a82 正式尝试 826 条命令）里首行状态和退出码全部一一对应，没有出现 `done` 加非零退出码。
2. **确定缺陷（新引入，轻）：`fill` 完成但退出码非零时，首行把步骤字段重复打两遍。** 例：`fill: done; focus=done; fill=done; submit=done; fill=done, verification="matches"; submit=done; …`。原因是 `fa41bce` 为了让清理指引显示出来，对 `fill` 的 `done` 一律放出顶层 reason，而设备端的 reason 本身就是 `focus=…; fill=…; submit=…` 这样的步骤摘要。测试用的样本 reason 不带这段摘要，测试解析器遇到重名字段时后者直接覆盖前者，所以测试抓不到这个问题。
3. **首行"按字段表声明"的重构只是部分达到目的。** 表只覆盖 6 类命令（剪贴板、通知、select、open、scroll-until、wait）。`result_line` 现在 113 行，实施方在 G0 时说的是 88 行，后续提交又长回去了；按命令判断的分支从 18 处增加到 25 处。表在函数里 5 个不同位置各调用一次，靠 `if command == …` 来保持顺序；表函数内部还专门为 `stopReason` 写了特判。`SUCCESS_FIELDS["scroll-until"]` 实际是死代码。退出码"集中"其实只是加了一个一行的 setter，设备回执里自带的 `exit`、构造函数传 `exit=`、`exit_code()` 里的 deliveryError 覆盖，这些旁路都还在。
4. **"442 份回执逐字一致"只覆盖 G0 那一个提交，而且只比较首行；原始样本和脚本都没放进材料包，没法复现。** 我自己做了独立复核：分别用 f401deb 和 409e853 两份契约（107 条和 113 条回执），比较 f401deb 渲染器和 G0（`5f76af5`）渲染器的首行输出，结果 0 差异，说明 G0 这个提交本身确实没改变首行行为。但它之后 `b263466`、`fb35dbf`、`fa41bce` 三个提交都有意改了输出，这部分不在对照范围内。
5. **两版的输出差异大多是改善**（首行带上正文字节数和行数，重复的错误原文去掉了，超限摘要按命令给出下一步，wait 失败时显示真实原因，root 不可用这一事实现在会显示，`not sent` 不再附带 `dispatchStarted=unknown`）。退步只有第 2 条。另有两处弱化：`scroll-until` 找到目标时首行不再给 `stop=found`，也不再给 `complete:false`；摘要的文字在文件没能保存时仍写"Search the saved file"。新的超限提示在 82f3a82 的正式尝试里一次都没触发过，没有真实使用的证据。

---

## 1. 方法

- 替身框架：`S/agents14/122/harness.py`。它替换 `UiClient.call` 和 `close`，按给定场景返回设备回执，并按 64 KiB 分片返回 `ui.read` 的附件（windows/tree/image），也可以按场景注入失败、抛 RuntimeError 或 KeyboardInterrupt。可选：让 `history.save` 抛 OSError、让 `ui_view.render` 抛异常、把 stdout 换成一写就抛 OSError 的流、让连接关闭失败。之后用真实的 `app.main()` 和 argv 执行，记录 stdout、stderr、进程退出码，以及工作区里生成的文件。
- 场景定义在 `mkscen.py` 生成的 `scen/*.json` 中，一共 58 个。回执取自 `contract/wire-contract.json`（如 plain-click、ime-done、enter-first-mismatch、enter-up-failure、selector-static），也有我手工构造的 failed、UNKNOWN、done+exit3。观察数据有三种来源：测试替身里的小树；实测原始树 `EV/runtime/api34/verification/r7-final-directed/evidence/000033-…json`（4,976 个节点）；合成的 1,500 节点树，渲染后正文 140 KB，用来触发 64 KiB 上限。
- 运行：`S/agents14/122/runall.sh`（用 `S/r7/venv7` 和 `S/r8/venv8` 分别跑全部场景），汇总在 `S/agents14/122/show.txt`，原始输出在 `runs/v7|v8/<场景>/out.json`。
- 契约首行对照：`rendercontract.py`，结果在 `contract-v7.json`/`contract-v8.json`、`r-contract-f401-{v7,g0}.json`、`r-wire-contract-{v7,g0}.json`。

## 2. 确定缺陷

### 2.1 `fill` 完成且退出码非零时，首行重复步骤字段（主机端实际运行）

- 位置：`androidctl/src/androidctl/ui_output.py:220-238` `reason_facts`。`fa41bce` 新增了 `or result.result == "done"`。
- 触发条件：`fill` 的结果为 `done`，并且 `exit_code() != 0`。具体包括：设备端清理不确定导致 exit 3（`AutomationCleanup.kt:74-82 withUiCleanup` 把 reason 改写成"指引; focus=done; fill=done; submit=done"）；本机连接关闭失败；`--screenshot` 图片未交付；超限正文文件未保存（exit 1）；渲染或附件环节出现异常。
- 实际输出：
  - 场景 H01（连接关闭失败，exit 3）：`fill: done; focus=done; fill=done; submit=done; fill=done, verification="matches"; submit=done; snapshotPath=…; local connection cleanup unconfirmed; …`
  - H03（超限正文，历史文件写失败，exit 1）和 C05（渲染异常，exit 1）也一样。
  - H02（设备端 exit 3）：`fill: done; UI cleanup unconfirmed; inspect status; focus=done; fill=done; submit=done; fill=done, verification="matches"; submit=done; …`。这里清理指引能显示出来，比 f401deb 好（f401deb 会把它吞掉，这是上一轮报告 91 第 4 条指出的问题），但步骤字段重复了。
- 影响：同一行里 `fill=` 出现两次，值不一样。调用者按字段读取时可能读到简略的那一次，丢掉 verification 和 actualValue。结论方向不会反转，但违背了"首行字段唯一"的约定（`test_ui_receipts.py:107` 对其他字段有 `count(...)==1` 的断言）。
- 为什么测试没抓到：`test_result_output.py` 里的 `test_done_fill_with_nonzero_exit_keeps_its_top_level_cleanup_guidance` 用的 reason 只有指引、没有步骤摘要；`parse_result_line` 遇到重名字段时后者直接覆盖前者，不报错。我用这个解析器解析 H01 的输出，得到的 `fill` 只剩一个值。
- 建议：`fill` 为 `done` 时，只输出 reason 中不属于步骤摘要的前缀；或者让 `withUiCleanup` 把指引另外放进一个字段。`parse_result_line` 遇到重名字段时应该直接断言失败，这样所有使用它的测试都会自动检查字段唯一。代价：改几行代码，加一条使用真实 reason 的测试。验证：重跑 H01–H03。

## 3. 核对过、确认没有问题的组合（主机端实际运行，两版结果相同，除非另注）

| 场景 | 首行状态 | 退出码 | 说明 |
| --- | --- | --- | --- |
| A02 click done，tree/windows 取回失败 | done，带 snapshotError/windowsError | 0 | 符合"动作观察缺失不改变主结果" |
| A03/A04 click done，附件环节抛 RuntimeError 或 KeyboardInterrupt | done | 1 | stderr 输出 `delivery: done; …`；动作没有被改写 |
| A05 click done，历史文件写失败，正文小 | done，提示"full observation follows in stdout" | 0 | 完整打印的正文可以替代自动保存的文件 |
| G02 click done，正文超限，历史文件写失败 | done，snapshotError | 1 | 符合 images.md："known action … remain with exit 1" |
| A10 click --screenshot，图片取回失败 | done，imageError | 1 | |
| A11 连接关闭失败 | done | 3 | |
| A13 设备端 done+exit3，tree 取回失败 | done，带指引 | 3 | 退出码 3 没有被交付错误降级 |
| B01 failed 且 dispatchStarted=true，tree 取回失败 | failed，带 dispatchStarted=true | 1 | |
| B02/B03 UNKNOWN（设备端或传输层） | UNKNOWN，带"Do not repeat automatically…" | 3 | |
| B04 Fault | not sent | 2 | 409e853 去掉了 `dispatchStarted=unknown` |
| B05 发请求时被中断 | `androidctl: UNKNOWN; Command interrupted…`，另一行"Do not repeat this action automatically." | 3 | 首行丢了命令名（两版都这样，见 6.3） |
| B06/C06/A09 stdout 断开 | 无首行；stderr 输出 `delivery: <原结果>` | 3／1／1 | 原状态保留在 stderr |
| C01/C02 fill not sent，但 focus 和 fill 两步已完成 | not sent，各步骤结果都在 | 2 | 前缀事实保留；exit 2 的含义写在 reference.md 中（"composite earlier effects may remain"） |
| C03 fill UNKNOWN | UNKNOWN | 3 | |
| D02 snapshot，tree 取回失败 | failed | 1 | |
| D03/D08/G07 snapshot 默认或 --print，历史文件写失败 | done，带"follows in stdout" | 0 | |
| D07/G08/G10 --no-print、--raw、--json，历史文件写失败 | failed | 1 | 与 images.md 表格一致 |
| D09 历史文件写失败，且 stdout 断开 | stderr 输出 `delivery: failed` | 1 | `print_delivery` 的 finally 把结果改为失败 |
| D12 --out 指向已存在的文件 | not sent，code=fileExists | 2 | 发请求前就拒绝 |
| E01 screenshot，图片失败 | failed | 1 | |
| F01 wait 节点匹配成功，但树取回失败 | done，snapshotError | 1 | 409e853 显示真实原因"capture expired"，f401deb 只有笼统文字 |

源码核对（源码推断，`409e853`）：会改写主结果的只有 `ui_client.py:1076-1091`（snapshot/find/screenshot）、`:640,652`（find 的缓存刷新）、`:1149-1156`（print 的 finally，同样只限 snapshot/find）。`deliver()` 用 try 包住渲染和保存（`:725-772`），`finish_delivery_result` 只会把 0 改成 1（`:1114-1122`）。`Result.exit_code()`（`protocol.py:69-81`）里，deliveryError 只会把 0 改成 1。

真实记录（实测，82f3a82，非 baseline 的 38 次尝试共 826 条命令，`cli-index`/`manifest.json`）：done/0 有 798 条，UNKNOWN/3 有 13 条，not sent/2 有 8 条，failed/1 有 6 条，没有出现状态与退出码不一致的情况。

## 4. 本轮重构是否达到目的

### 4.1 首行"按字段表声明"（源码推断，加主机端实际运行）

- `COMMAND_FIELDS`/`SUCCESS_FIELDS`（`ui_output.py:38-80`）只覆盖剪贴板、通知、select、open、scroll-until、wait。点击、输入、滚动、按键、fill 这类最常用的动作，字段仍然散在 `observation_facts`、`value_facts`、`input_facts`、`dispatch_facts` 以及 `result_line` 内部的二十多个 if 里。例如本轮新增的 `rootUnavailableObserved` 就没有进表，而是另写了一个分支（`:298-301`）。
- 加或改一个首行字段需要动几处：如果字段属于表内命令，只改表 1 处，再加测试和契约样本；如果属于其他命令，要在某个语义函数里加一个分支，并且得清楚它在 `result_line` 中的相对位置。字段顺序靠 `result_line` 里 5 处分散的 `declared_field_facts` 调用来维持（`:155,159`，`launch_facts` `:322`，`:186`，`:209`，外加 `scroll_search_facts` `:335`）。所以分支并没有消失，而是从字段循环变成了"调用点上的命令判断"。
- 规模：`result_line` 现在 113 行（`:105-217`）。实施方在 G0 时记录的是 88 行，之后 `b263466`、`fb35dbf`、`fa41bce` 又加回来了。整个文件里 `command ==/in/not in` 判断从 f401deb 的 18 处增加到 25 处。
- 表函数并不纯：`declared_field_facts` 为 `stopReason` 写了特判，改字段名为 `stop`、改用 str 渲染（`:92-99`），这是实施方在 G0 中途发现加了引号之后补的；渲染方式也按命令分为 `structured_json` 和 `compact` 两种（`:87-91`）。
- 死代码：Android 端的 scroll-until 只有在 `stopReason == "found"` 时才返回 DONE（`ScrollProgress.kt:56`，其余 `stop()` 默认都是 FAILED），而 `scroll_search_facts` 在 done+found 时直接返回（`:333-334`），所以 `SUCCESS_FIELDS["scroll-until"]` 永远不会被用到。
- 一处隐蔽的行为：`launch_facts` 对所有命令都会调用 `declared_field_facts(result, "open")`（`:322`），没有先判断 command 是不是 open。目前只有 `ActivityLaunch.kt` 产出这些字段，所以还没有出错，但读代码的人会以为这是 open 专用的。
- 结论（设计意见）：这次重构把原来 280 行的单个函数拆成了若干语义函数，可读性确实提高了，这是真实收益。但"按表声明"这个说法言过其实。我建议不要继续扩表：要么诚实地把它当成"6 类命令的字段清单"，要么把 stopReason 的特判和 open 的无条件调用清理掉，让表真正只描述字段。

### 4.2 测试解析集中（源码推断）

- `ui_test_support.py:12-55` 中的 `parse_result_line` 被 25 个测试文件引用。剩下直接用原始文本做断言的地方：首行字面量 golden 16 处（`test_wire_contract.py` 9 处，`test_result_output.py` 5 处，另有 2 处）；首行或 `splitlines()[0]` 定位 21 处；带 `=` 的子串断言 33 处（多数是 status/job 这类非 UI 渲染，或者否定性检查）。这些和实施方 `r7-g0-review-fixes.md` 列出的"保留理由"基本对得上，没有发现新的分散解析器。
- 解析器本身的弱点：遇到重名字段时后者覆盖前者，不报错，这正是 2.1 那个缺陷在测试里漏掉的原因。另外，对自由文字的引号配对和空格加 `device=` 的边界，它是按现有输出格式写死识别的。
- 改一次输出要连带改多少测试：如果只是改字段值或加字段，通常只影响断言了该字段的测试，加上契约 golden 那 16 处；如果改的是分隔方式或顺序，还会影响那些 golden 和 33 处子串断言。比第七轮报告里说的"约 65% 的测试断言渲染文字"好了很多，但 `test_wire_contract.py` 仍然是改输出时的主要牵连点。

### 4.3 退出码"集中赋值"（源码推断）

- 主机端的 `Result.set_exit` 只有一行 `self.exit = code`（`protocol.py:66-67`），Android 端的 `setExit` 也只有一行 `put("exit", code)`（`Wire.kt:56`）。这次提交真正的收益，是删掉了 5 处各自定义的常量 3，并统一用 `UNCERTAIN_EXIT`。
- 绕过 setter 的写法仍然存在：
  - 设备回执自带的 `exit`，主机端经 `Result.model_validate` 直接接收；
  - `local_call.py:244,253,263,271` 用 `Result(..., exit=result.exit)` 构造新结果；
  - `exit_code()` 在计算时用 deliveryError 把 0 覆盖成 1（`protocol.py:81`）；
  - `BorrowedPaste.kt:345` 先 `remove("exit")`，再按 restore 状态重新设置，其中 1 是硬编码；
  - 进程出口有 `ui_commands.py:69`、`app.py:422,673,1140,1156`、`job_commands.py:328`；`job_commands.py:314` 直接读 `result.exit`。
- 结论：赋值的写法统一了，但"退出码怎么决定"并没有收到一个函数里，这个说法应该收窄。这些旁路目前都没有导致错误的退出码（第 3 节的组合已经覆盖到）。

### 4.4 "442 份回执和 3 份 raw 逐字一致"

- 对照怎么做的：按 `r7-output-baseline/README.md` 的说法，在 G0（`5f76af5`）之前把 442 个渲染输入和输出（326 条测试输出加 116 条契约回执）连同 3 份 raw 树冻结下来，G0 之后重放比较首行、选择明细和通知行，raw 树另外比较正文。
- 覆盖不到的部分：G0 之后所有有意改动输出的提交（3 个）；保存、历史、退出码、JSON 输出；以及 `result_line` 之外的 `render_fields`/`heading` 路径。
- 可复现性：材料包里只有一行日志（`EV/r7-output-baseline/g0-replay.log`："442 receipts + 3 raw trees compared; 0 differences"），`capture.py`、`receipts-before.json`、`trees-before.json` 都没有附上，所以属于证据不足。
- 我的独立复核（主机端实际运行）：用 f401deb 渲染器和"f401deb 加 5f76af5 的 ui_output.py"渲染器，分别渲染 f401deb 契约（107 条）和 409e853 契约（113 条）的回执，首行 0 差异（`r-contract-f401-{v7,g0}.json`，`r-wire-contract-{v7,g0}.json`）。G0 本身不改变首行行为，这一点我认为可信。

## 5. 与 `f401deb` 的输出差异（主机端实际运行）

契约回执（113 条中有 12 条首行不同，`contract-v7.json` 对 `contract-v8.json`）和 58 个场景的逐类判断：

| 变化 | 例子 | 判断 |
| --- | --- | --- |
| 首行增加 `snapshotBytes=…; snapshotLines=…` | 所有保存了正文的命令 | 改善：调用者能决定要不要读文件；每行多约 40 字节 |
| 历史文件名去掉窗口标题，只保留 序号-时间戳-应用 | `000001-…-example.shop.txt` | 无差别或略好（属计划内小项 16） |
| 和 reason 相同的 snapshotError/imageError 不再重复显示 | D02、D07、G04、E01 | 改善（上一轮报告 91 第 5 条） |
| 超限摘要按 find、REF、整页分别给下一步，并给出实际字节数和行数 | G01、G03、G11、G12 | 改善；建议的 `find Q --context 0 --max-chars 16384` 实测 2–6 KB，可用。但正式尝试中一次也没触发过（实测：`grep "exceeds 65536"` 在 38 次候选尝试中为 0 条），没有真实使用证据 |
| 文件没能保存时，摘要仍写"Search the saved file" | G02、G04、G13 | 轻微退步或不准确；首行已有 snapshotError，影响小 |
| `not sent` 且没有派发信息时，不再输出 `dispatchStarted=unknown` | B04、session-close-*、repair-ambiguous-fill | 改善：去掉了多余信息 |
| 新增 `timing=ref-repair; Rejected REF was not verified…`、`rootUnavailableObserved`、`cutoffStage`、reveal 时的 `target still offscreen` | 契约中的对应用例 | 改善：都是新的真实事实 |
| scroll-until 找到目标时，match 只保留 matches/refs，去掉了 `stop=found` 和 `complete:false` | scroll-until-*-found | 中性偏弱："在不完整采样上找到"这一信息只在 JSON 里还有。正面事实不受影响，可以接受 |
| fill done 且非零退出时，显示清理指引，但步骤字段重复 | H01–H03、C05 | 指引显示出来是改善，重复是退步（见 2.1） |
| JSON 中 `snapshotCompleteness.missingCount` 改名为 `missingRecords`，并新增 incompleteParents | G10 | JSON 结构的破坏性改名，属计划内的下-13（第七轮评审清单中"进下一轮"的第 13 条），指引已同步；实施方记录中没有写明"JSON 键改名" |

实施方记录中没有提到的行为变化：一是 2.1 的字段重复；二是 JSON 键改名，设计里写了，但没有当作兼容性变化列出来。

## 6. 一致性、默认上限和其他

### 6.1 保存文件、`--raw`、`--print`、`--cached` 与历史记录

这几种情形的实测结果都和 `images.md` 的交付表一致：

- `--raw` 只保存 JSON、不打印；保存失败时结果为 failed、exit 1（G08）。
- `--print` 时，完整打印可以替代自动保存的文件（D08、G07）。
- `--no-print` 和 `--json` 都要求文件必须保存成功（D07、G10）。
- `--cached` 和 REF 读取正常（G06、G12）。
- 显式 `--out` 指向已存在的文件时，在发请求前就以 not sent 拒绝（D12）。

### 6.2 默认上限

上限按 UTF-8 字节严格大于 65,536 时触发（`ui_client.py:751-758`）。摘要文字改为引用常量，上一轮的小瑕疵已修好（`:915`）。三类提示中：

- find 的提示可以照着执行；
- REF 的提示（`snapshot REF --depth 1` 后再读子 REF）可以照着执行；
- 整页的提示是先 find 关键词、再 snapshot REF，可以执行；但文件没保存时，提示里的"Search the saved file"不成立（见第 5 节）。

### 6.3 可删可并的地方

- `reason_facts` 的 `:239-244` 与 `Output.error_fields`（`output.py:71-79`）完全重复（都输出 code/argument/subject/stage），可以直接复用，不损失任何能力。
- `SUCCESS_FIELDS["scroll-until"]` 是死代码，可以删（见 4.1）。
- `ui_client.py:646` 的 `findRefreshed` 只在 JSON 里还有用，文本已经改用 `fresh`，保留即可。
- 发请求时被中断的情形（B05），首行命令名是 `androidctl` 而不是 `click`，已收到的事实也会丢失。原因是 `UiClient.request` 只捕获 Fault，KeyboardInterrupt 会一直冒到 `app.main`（`app.py:1132-1146`）。指引里有"Do not repeat"，所以影响小，属于可选建议。
- `contract/wire-contract.json` 本轮多了 307 行：11 条新回执用例（wait×2、点击 ref-repair、scroll-until 路由不可用、root 不可用或已恢复、session close 等），以及 `textFolding` 4 条。每条都对应一个新的跨语言事实，并且被 `test_wire_contract.py` 用到。行数多主要是 JSON 缩进展开造成的，不算冗余，我认为有必要保留。

## 7. 需要用户决定的问题

**问 1：`fill` 完成且退出码非零时，首行怎么显示 reason。**
- A：只显示 reason 里的指引前缀，去掉步骤摘要。改几行代码，首行不再重复。
- B：Android 端把清理指引放进单独的字段，比如 `cleanupGuidance`，reason 保持原样。语义更干净，但要改契约和两端代码。
- C：维持现状。指引能看到，但字段重复。
- 推荐 A，同时让 `parse_result_line` 遇到重名字段时直接报错。

**问 2：要不要继续把首行字段迁进字段表。**
- A：到此为止，清理掉 stopReason 特判和死代码，并把这件事表述为"6 类命令的字段清单"。
- B：把所有动作的字段都迁进去。这需要能声明条件和顺序的表，本质上就是规则引擎，代价大、收益不明。
- 推荐 A。
