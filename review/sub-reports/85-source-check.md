# 第六轮决定文件：与 `fb75afc` 源码的核对

核对对象：`/home/user/androidctl/review/androidctl-r6-decisions.md`（下称"决定文件"），以及它第四、五节依据的两份工作记录 83 号（裁决落地风险）和 82 号（原因码清单）。源码一律指 `fb75afc`（`S/r6/prod`，工作树干净）。Kotlin 文件省略前缀 `android/app/src/main/java/com/rainng/androidctl/`，主机文件省略前缀 `androidctl/src/androidctl/`，指引文件省略前缀 `skills/androidctl/`。

证据类别：【源码】读 `fb75afc` 得出；【实跑】在 `S/r6/venv6`（可编辑安装的 `fb75afc` 主机端）里调用产品自己的函数；【重算】我自己写的脚本，放在 `S/agents10/85/`（`codes.py`、`vis2.py`、`inline2.py`、`fix13.py`、`deliv.py`、`acts.py`、`acts2.py`），都用 `-I` 运行，只读材料包；【原件】读材料包里的原始输出；【推断】；【需设备】。没有设备，也没有跑 Gradle，Android 一侧的结论全部来自读源码。

---

## 1. 总评

- 决定文件里的源码陈述总体可靠。我逐条核了约 90 处，与源码明确不符或会让实施方做错的有 6 处，需要收紧的有 12 处，没有发现结论与源码相反的风险提示。
- 最严重的两处：
  - 第 3 条 A1 的修法和它自己写的规则不一致。照修法做完，默认输出里仍会打印平台的"显示提示"动作（`action-1020044`）、`copy`/`paste`/`cut`，以及标签读不到时的 `select`/`clear-selection`。
  - 第 14 条要求的摘要内容里没有短溯源行和缺口标记，而第 15 条恰恰要求这两样必须留在默认文本里。
- 其余 4 处：第 23 条漏了 `find --raw`；第五节甲组第 7 组没写清 `point`、`region`、`range` 的另一半用法要归到哪里；乙组第 1 问把 `--print` 的现状写成"与默认相同、普遍存在"，实际上它只在 4 个命令上有；乙组第 6 问的"20 次运行没人出错"其实只有 5 次运行由模型自己开过会话。
- 第四节（来自 83 号记录）可靠。我独立重推了九处里的全部主要结论，文件和行号基本准确。只有一处数字不对：`--json` 的上界不是一律 192 KiB，`shell --json` 最多约 350 KB。83 号记录里另有约 10 条对实施方有用的提示没有收进决定文件，列在第 5 节。
- 第五节甲组（来自 82 号记录）可靠。我用独立脚本复现了 156 个码、设备端 114 个、主机端 62 个、两端共用 20 个，也复现了真实运行里 15 次 `code=`。第 1 到第 6 组逐码核过，"三处走同一个函数""文字几乎一一对应"等说法都属实；合并不会改变任何退出码，也没有消费者按旧码分支。

---

## 2. 与源码不符，必须改

### 2.1 第 3 条 A1：修法与规则不一致，照修法做完 A1 只修了一部分

- **决定文件原文**："把'默认省略'的判断放到'自定义动作'之前；需要参数的标准动作默认不打印；补一条包含这几个编号（64、128、256、512、1024、2048）的渲染测试。规则说清楚：……平台的标准动作里没有对应命令的那些，默认省略。"
- **源码事实**【源码】`ui_view.py:40-53, 55-69, 106-144`；`node_facts.py:42-72`：
  - `compact_actions` 的分支顺序是：不在命名表 `ACTION_NAMES` 里，就打印 `{"name": "action-XXXX"}` 或标签；在命名表里、有标签，就打印名字加标签；**标签读不到时只打印名字**；然后才轮到"默认省略"的判断；最后打印有专门命令的动作。
  - 那六个编号正好是"默认省略集合减去命名表"，这一点决定文件写对了。
  - 但不在命名表、也不在"默认省略"或"需要参数"集合里的平台标准动作还有：16908356 和 16908357（显示、隐藏提示），16908373 到 16908376（拖放、显示文字建议）。在命名表里、没有对应命令、也不在"默认省略"集合里的有 16384、32768、65536（`copy`、`paste`、`cut`）。按修法的字面去做，这些动作仍会打印。
  - "标签读不到时只打印名字"这一支排在默认省略之前，所以 `select`、`clear-selection` 这类本该省略的动作也会打印。
- **影响有多大**【原件】【重算】：
  - 在候选保存的观察文件里，普通 Chrome 按钮的默认行就带着 `action-name="action-1020044"`，也就是显示提示。例如 `r5-c2-final-candidate-0*/workspace/.androidctl/snapshots/*.txt` 里的 `button e222291924 "New tab" clickable +long-click action-name="action-40" action-name="action-100" action-name="action-200" action-name="action-1020044"`。
  - 全部候选观察文件里，`action-1020044` 共出现 254 次；`paste`、`copy`、`cut` 各 30 次；`select`、`clear-selection` 各 56 次。作为对照，`action-40` 是 6,696 次。
  - 在 601 份历史树上用 `fb75afc` 自己的 `compact_actions` 重算（`acts.py`、`acts2.py`），修法覆盖不到的项里，显示提示一项就有约 4.8 万次，几乎全在 Chrome 上。
- **另一处字面歧义**：`PARAMETERIZED_ACTIONS` 里包括 2097152（`fill`）和 16908349（`set-progress`），它们需要参数、但有专门命令，现在默认会打印。"需要参数的标准动作默认不打印"按字面会把它们也藏掉。
- **建议的改后文字**："默认文本只打印两类动作：有专门命令的（`NAMED_COMMAND_ACTIONS`，滚动方向另在 `scroll` 字段里打印），以及平台标准动作表之外的自定义动作。平台标准动作没有对应命令的，不论是否需要参数，一律默认省略。这一判断放在'自定义动作'和'标签读不到'两个分支之前。渲染测试至少覆盖 64、128、256、512、1024、2048、16908343（滚动到位置）、16908356（显示提示）、16384（复制），以及一个标签读不到的 `select`。"

### 2.2 第 14 条：摘要的最低内容漏了第 15 条要求保留的两样

- **决定文件原文**（第 14 条落实要点）："摘要里至少要有结果首行、文件路径和怎样缩小范围的提示，其余由实施方定。"
- **源码事实**：
  - 文本模式下，`capture` 只出现在正文的头一行 `androidctl capture=... sequence=...` 里（【源码】`ui_view.py:1001-1020`），首行里没有。缺口信息（`partial`、`missing`、`missingSummary`）也只在正文的 `observation {...}` 行里（`ui_view.py:650-732`）。首行只有缺口计数 `observationWarning=`（`ui_output.py:169-178`）。
  - 结果未知之后的恢复规则要求拿原来的 capture 与新快照的 capture 对比（`reference.md:66-76`）。第 15 条明说采集编号和缺口标记"必须留在默认文本里"。
  - 83 号记录的 2.1 节把"正文的头部行"列为摘要必须有的第 2 项，决定文件转述时漏了。
- **影响**：一条结果未知的动作，如果附带的观察超过上限，按决定文件的最低要求只打摘要，调用者就在文本里拿不到 capture，恢复规则失去依据。
- **建议的改后文字**："摘要至少要有：结果首行；第 15 条的短溯源行（含采集编号）和采样缺口标记；正文的字节数和文件路径；一行可照抄的缩小范围写法。`find` 超限时另外保留命中摘要行（命中数、屏外命中数）。"

### 2.3 第 23 条：`find --raw` 也是"整份原始 JSON 打到标准输出"，决定文件只写了 `snapshot`

- **源码事实**【源码】：`find` 有 `--raw` 选项，帮助文字是 "Save separate artifacts with the whole collected tree as JSON."（`ui_commands.py:698-705`）。`render_observation` 只要 `view.raw` 为真，正文就是整份原始 JSON（`ui_client.py:782`），后面的投影只在 `not view.raw` 时才替换正文（`ui_client.py:839`）。所以 `find --raw` 和 `snapshot --raw` 一样，会把几 MB 的 JSON 打到标准输出，并存成 `.json`。【实跑】在后台屏长页上，`--raw` 的正文是 3,728,389 字节（`deliv.py`）。
- **建议**：第 23 条、第二节第四轮第 4 项那一行、附录一清单里的"`snapshot --raw` 默认只保存"，都改成"`snapshot --raw` 和 `find --raw` 默认只保存、不打印"。如果有意只改 `snapshot`，要写明 `find --raw` 是否受第 14 条的上限约束。

### 2.4 第五节甲组第 7 组：没写 `point`、`region`、`range` 的另一半用法要归到哪里

- **决定文件原文**："`actionLabel`、`actionName`、`duration`，以及 `point`、`region`、`range` 里取值越界的用法……减少 6。"
- **源码事实**【源码】：这三个码各自还有不属于"取值越界"的用法。要让它们完全消失、减少 6 个，这些用法必须移到别的码下。82 号记录 §三乙写了，决定文件没有写：
  - `point`：`AppInput.kt:155` 的 "Fresh target center is outside the display"，归 `notReady`。同一函数里"目标不可见""宽高为零"已经报 `notReady`（`AppInput.kt:139-150`）。
  - `region`：`ImageRegion.fromRef` 调用 `within`（`ImageRegion.kt:160, 189`），目标的当前边界超出屏幕，归 `notReady`。`within` 要区分调用方。
  - `range`：`NodeRange.kt:45, 48` 的"最小值、最大值读不到"，归 `rangeUnavailable`（`NodeRange.kt:42`）。
- **影响**：用户批准时看不到 `notReady` 会多出两种用法；实施方照表去做，只并越界用法，个数只会减 3。
- **建议**：在第 7 组那一行补一句："另一半用法归位：`point` 和 `region` 的'目标在屏外'归 `notReady`，`range` 的'范围读不到'归 `rangeUnavailable`。"
- 附带一个相邻的事实，供实施时一并看【源码】：`ImageRegion.fromRef` 先用 `rectangle()` 校验目标的当前边界，边界为零面积时报的是 `arguments, argument=region`（`ImageRegion.kt:170-181`）。可是调用者的参数并没有错。第 13 条之后零面积节点会更常见，这一处也应归 `notReady`。它不在第 24 条现有的分组里，由用户决定是否顺手处理。

### 2.5 第五节乙组第 1 问：`--print` 的现状写得不准，影响选项的得失

- **决定文件原文**："`--print` 是现有选项，现在与默认行为相同。"
- **源码事实**【源码】：
  - 只有 4 个命令有 `--print/--no-print`：`snapshot`、`find`、`locate`、`screenshot`（`ui_commands.py:536, 708, 753, 1438`）。`wait`、所有动作命令和滚动查找只有 `--no-print`（`ui_commands.py:453` 等约 20 处）。
  - 显式 `--print` 和默认并不完全相同：它会把正文标成"请求了的交付物"，正文没交付时写 `snapshotError` 并把退出码改成 1（`ui_client.py:1007-1032` 的 `observation_requested`）。另外 `--print --json` 在参数阶段就被拒绝（`ui_client.py:576-581`）。
- **影响**：推荐项 A 说"显式 `--print` 越过上限，就有一个'我确实要全文'的出口"。但这个出口只存在于上面 4 个命令。`wait` 成功时附带的命中正文、动作附带的观察、滚动查找的最终观察一旦超限，按 A 也只能去读文件。用户应当知道这一点。
- **建议的改后文字**（背景）："`--print` 只在 `snapshot`、`find`、`locate`、`screenshot` 上有；显式写时，正文没有交付会使退出码变成 1，其余与默认相同。`wait`、动作命令和滚动查找只有 `--no-print`。"A 项的"失"里补一句："`wait`、动作命令和滚动查找没有 `--print`，超限时只能读文件；要给它们加 `--print` 就是新增选项。"

### 2.6 第五节乙组第 6 问："20 次运行里没有人在这里出错"夸大了证据

- **原件事实**【原件】：在 20 次候选计分运行的 `evidence/cli/*/intent.json` 里，只有后台屏题的 5 次（`r5-v1-candidate-01` 到 `05`）由模型自己发过 `session open`，写的都是 `--display virtual`。其余 15 次的第一条命令直接就是 `snapshot` 等，会话多半由执行框架事先开好【推断】。全部 37 份运行里，没有一次出现 "Missing option '--display'"。
- **建议的改后文字**："模型自己开会话的 5 次运行（都在后台屏题）都写对了 `--display`；另外 15 次的会话由执行框架事先开好。"
- 附带说明缺 `--display` 时现在报什么【实跑】：`session open: not sent; Missing option '--display'. Choose from:\n\tmain,\n\tvirtual; inspect accepted syntax with: androidctl session open --help; code=arguments`。这里的 `\n\t` 是原样打出的转义字符，正是附录一里"命令行框架的缺参数报错"那一条，不论第 6 问怎么选都可以顺手修。

---

## 3. 不准确或需收紧，建议改

1. **第四节第 1 处，"`--json` 的上界是设备回复的 192 KiB"。** 这对 `shell --json` 不成立。`shell` 的两个输出流是另外分块取回的，每个流最多 128 KiB（`protocol.py:21`、`job_commands.py:156`），再各自转成 base64 放进 `data.outputBytes`（`job_commands.py:216-222`），最坏约 350 KB。83 号记录 2.1 节自己也写了约 350 KB。建议改成："界面类回执的上界是设备回复的 192 KiB；`shell --json` 最多约 350 KB。"

2. **第四节第 1 处和乙组第 3 问，"现有代码在这四种情形下把正文全打出来作补救"。** 正文确实都打出来了，但只有"文本模式、自动保存失败"算补救：结果仍是 done、退出码 0，并附 `snapshotWarning`。显式 `--out` 写失败、`--raw`、`--no-print`、`--json` 都不算补救（`ui_client.py:975-984` 的 `body_file_required`）：`snapshot` 和 `find` 报失败、退出码 1，其中 `--out` 和 `--raw` 照样把整份正文打出来。【实跑】`deliv.py` 让工作区不可写：默认模式是 `done`、0、三个诊断字段；`--no-print`、`--json`、`--raw` 都是 `failed`、1。建议改成："自动保存失败时，文本模式现在打全文并按交付成功算；`--out`、`--raw` 失败时打全文但按失败算。"

3. **第四节第 1 处只点了 `finish_delivery_result`。** `print_delivery` 的 `finally` 也要知道"打的是摘要、不是正文"（`ui_client.py:1056-1064`）。如果实施方把打了摘要也记成 `body_printed = True`，那么保存失败时 `snapshot` 会被误报成 done。建议补半句："`print_delivery` 里'正文已打印'的判断也要把摘要排除在外。"

4. **第 18 条，"一件交付物只写一个诊断字段"没有说现在是哪三个。** 【实跑】工作区不可写时，默认文本模式同时写 `historyError`、`snapshotError`（两者内容相同）和 `snapshotWarning`；`--no-print` 或 `--json` 时是 `historyError`、`snapshotError`，再加上同样内容的 `reason`。来源在 `ui_client.py:932-938`（`historyError`）、`ui_client.py:939-943`（`snapshotError` 由 `setdefault` 抄一份）、`ui_client.py:980-984`（`snapshotWarning`）。建议写明这三个字段，并说明保留哪一个。

5. **第 3 条 A2 的修法和措辞。**
   - 能做通【源码】：`registrationStarted()` 在 `connectWithTimeout` 之前调用（`UiBackend.kt:533-534`）。断开只试一次（`AutomationCleanup.kt:32-41` 的 `disconnectAttempted`）。连接同步抛错时把 `registrationPending` 清掉即可，回调线程照常停止。
   - 实施方必须知道两点：
     - 这是反射调用，超时也会被包成 `InvocationTargetException`。必须先用 `rootCause`（`Wire.kt:73`）取出原异常，才能分清是超时还是同步抛错。
     - 同步抛错不一定是"另一个工具占着"，系统进程断开、反射失败也会同步抛错。所以回执写"多半是另一个自动化工具占用了登记"更准确。
   - 这条回执现在的原因码是 `workerFailure`，也就是所有非 `Rejected` 异常的默认码（`Wire.kt:80`）。第 24 条又规定"批准之前实施方不动原因码"，两者有张力，见第 4 节 C3。
   - 没有别的路径会把"连接同步抛错"留成待注销。其他会触发封锁的路径都是真的清理失败：回调线程停不下来（`UiBackend.kt:553`）、后台屏的图像读取端或显示没有销毁（`WorkDisplay.kt:181, 193, 210`）。按裁决，这些继续封锁。

6. **第 3 条 A4 的范围列举。** 成功时还会打印过程信息的另有两处，决定文件没有列：
   - `reveal`：打印 `reveal={...,"reads":N,"cutoffMs":...}`（`UiActions.kt:131`，经 `ui_output.py:463-477` 的 `input_facts`）。
   - `fill` 的各步：`step_facts` 对每一步都调用 `input_facts`，不论成败（`ui_output.py:440-460`）。提交改走按键时，会打出按键进度。
   - 建议把"按键类、`type`、`click --touch`、滚动查找"改成"所有成功的动作命令（例如按键类、`type`、`click --touch`、`reveal`、`fill` 的各步、滚动查找）"。

7. **第四节第 2 处，"观察、采集顺序、动作前读取三处共用"。** 可见位的来源实际有 6 个调用点：`NodeFacts.read` 被 `TreeCollector.kt:274`、`FreshNodes.kt:23, 39, 72`、`UiActions.kt:65`（读当前输入焦点）调用，另有 `TreeCollector.kt:185` 直接读平台值。如果改 `NodeFacts.read` 的签名，这些调用点都要传窗口边界。建议列出这些位置。

8. **第五节乙组第 5 问。**
   - A 项写的是"与所在窗口的交集"，又说"符合裁决字面"。裁决字面"屏内"指显示区域；按窗口算是 83 号记录推荐的细化，两者在全屏时相同。
   - 数字要按读法分开写【重算】（`vis2.py`，与 83 号记录一致）：按显示区域算，主屏有 107 个"自身面积为正、整个在屏外、平台报可见"的节点（京东 58、设置 39、Chrome 10）；按窗口算约 99 个，因为 Chrome 的 10 个不算、通讯录多 2 个。B 项"只影响 2 个"属实。
   - 建议把 A 改成"与所在窗口的交集为零（全屏时等同字面的'屏内'）；主屏约 99 个"。

9. **第五节乙组第 4 问 A 项，"与 `reference.md` 的退出码表一致"。** 原文核对【源码】：
   - `reference.md:52`：退出码 3 那一行写 "Dispatch, required receipt/verification or cleanup/persistence is uncertain. A known main result can remain done."
   - `input.md:61`："Done needs confirmed PASTE, matching local effect and completed conditional cleanup"。
   - 两处确实相反。
   - 表里明确允许的只有"done 加退出码 3"。"done 加退出码 1"表里没有直接写，只能从 `reference.md:56-57`（"Delivery errors change a computed zero to nonzero while preserving an existing nonzero/cleanup exit"）间接推出。
   - 建议改成"与 `reference.md:52` 一致，退出码 1 的用法需要在表里补一句"。

10. **第二节第四轮第 16 项那一行写"取消"。** 第六节第 6 处让 `gone`、`idle` 保留提前失败。建议改成"对正向等待条件取消；`gone`、`idle` 保留"。

11. **第 15 条的"诊断说明短语"指的是首行里的东西，不在正文里。** 例如 `(original target unknown: Original target is not identifiable in the final observation)`，来自 `UiEffect.kt:49`，经首行的 `change_summary` 打印。候选运行里出现 55 次以上【原件】。建议注明位置，免得实施方只去改正文。

12. **第 21 条的落点。**
    - 清单里只有一个 Activity：`MainActivity`，`exported="true"`，没有别名，没有 `launchMode` 和 `taskAffinity`（`AndroidManifest.xml:16-21`）。管理页从自己的任务里打开两个设置页（`MainActivity.kt:178, 186`，没有新任务标志）；通知点开管理页用 `GatewayService.kt:45`。
    - 要删的例外说明在 `images.md:140-141`（"Metadata cannot identify an arbitrary middle Activity in another App's task"）和内部文档 `docs/reference/ui.md:290` 末句。`displays.md:47` 的 "middle Activity" 说的是任务列表的覆盖面，不要一起删。
    - "AndroidCtl 永远是自己任务的根"：按平台对 `singleTask` 的处理，基本成立【推断】，依据是 AOSP `ActivityStarter` 的规则。外来调用会被强制放进新任务；把它放进指定任务、而那个任务的根不是它时，平台会抛异常。例外是别的 App 声明了相同的任务亲和性这类刻意构造的情形。没有设备验证。建议保留六字段的自身检查，不为删几行去改它（83 号记录也这样建议）。

13. **小处。**
    - 82 号记录和决定文件说"漏数了 4 个动态构造的码"。其中 `replyUnavailable` 是经子类 `TransportFailure("replyUnavailable", …)` 用字面量构造的（`transport.py:438`），不算动态；它是被旧脚本的正则漏掉的。不影响计数。
    - 82 号记录估第 3 组"测试约 2 处断言"，实际 `UiDeliveryTest.kt` 里有 10 处左右断言 `noPublishedSnapshot`。决定文件没有写处数，不用改。

---

## 4. 做不通或互相打架的地方

**C1. 第 14 条（摘要）与第 15 条（保留采集编号和缺口标记）。** 见 2.2。按决定文件现在的文字，两条可以同时满足，但实施方很可能做出"超限时连短溯源行一起省掉"的摘要。这是实施细节，不需要用户裁决，在第 14 条的落实要点里补上即可。

**C2. 第 14 条（上限）与乙组第 1 问 A（`--print` 越过上限）。** 见 2.5。不冲突，但 A 的"出口"只覆盖 4 个命令。这仍属于第 1 问本身，用户裁决时要看到修正后的背景。不另开新题。

**C3. A2 的新回执与第 24 条"批准之前不动原因码"。** A2 要求回执说明"另一个自动化工具占用了登记"。这条回执现在落在默认码 `workerFailure` 上，按 82 号记录的分类属于"只能报告"。它的下一步其实是"告诉用户、等对方退出后原样重试"。A2 在这一批先做，第 24 条的清单还没批，实施方会卡在用哪个码上。需要用户点头，因为它涉及原因码：
- **A（推荐）**：复用已有的 `busy`，加 `subject=uiAutomation`。理由是下一步与 `busy` 的"尊重当前占用者、稍后重试"相同，符合第 24 条的新规矩。
- B：新增一个码，例如 `automationBusy`，补进第 24 条的清单一起批。
- C：码不变，仍是 `workerFailure`，只改消息文字。得：完全不动原因码。失：调用者按码看到的是"只能报告"一类，与实际下一步不符。

**C4. 第 19 条（自动重绑）与"界面清理未确认就封锁"的门、A2。** 代码上不打架：
- A2 之后，"别的工具占着"不再进封锁。
- 常驻进程意外死亡后，新进程内存里没有清理记录，自动重绑会顺带解开封锁。所以"门保留"实际只在一个常驻进程的生命期内成立。
- 用户按封锁提示去点"停止空闲执行器"，属于应用发起的停止，不重绑。用户还要再点一次刷新（`connection.md:105`），提示文字应写明这一步（决定文件第四节第 4 处已写）。
- 这是实施细节。

**C5. 第 13 条与第 11、16、17 条、B3。**
- 第 11 条：`find` 用搜索投影，包含不可见节点并标出屏外（`ui_view.py:939-998`），可见位改了也照常找得到，只是屏外命中数在后台屏上变得有意义。
- 第 16 条：现在滚动查找的选择器模式按解析后的目标过滤可见（`ScrollContainer.kt:91`）。可见子节点的选择目标如果是零面积、被改标的父节点，现在就永远不算命中；第 16 条对齐 `wait`、改按命中节点之后，这个问题自然消失。两条都做，顺序无所谓。
- 第 17 条依赖第 13 条，决定文件已经排了先后。
- B3 已经要求与第 13 条一起做。
- 没有发现按字面做不通的地方。

**C6. 第 10 条（附一行，原因码保持"已过期"）与第 24 条。** 第 24 条的候选清单不碰 `stale`、`changed`，82 号记录 §四第 1 条明确不合并，并引第 10 条作理由。两条相容。

**C7. 第 22 条的实施边界（实施细节）。** `BorrowedPaste.kt:119-125` 现在的覆盖规则不只作用于"粘贴已成功"：恢复失败时，主结果哪怕本来是 NOT_SENT 也会被改成 FAILED；恢复状态未知时，任何主结果都会被改成 UNKNOWN。第 22 条只应改"效果已核对一致"这一种，其余保持现状。决定文件没写这层边界，建议在落实要点里补一句。

**C8. 第 11 条的时限（实施细节）。** 主机端重采是第二次请求，与第一次共用命令的截止时间（`ui_client.py:644`，`find` 默认 5 秒）。大页面上新采要 1.5 到 3 秒，可能以时限失败告终。决定文件第四节第 6 处写了失败怎么报，没写这个原因。

没有发现需要用户另行裁决的冲突，C3 除外。

---

## 5. 83 号记录里应当补进第四节的风险

通读 83 号记录的 2.1 到 2.9，以下各条对实施方有用，决定文件没有收。每条我都回到源码核过。

1. **契约样本可能要重新导出（第 13 条）。** 【重算】`contract/wire-contract.json` 里有 2 个"零面积、可见位为默认值真"的节点（`actionResults[29]` 里 `selected.target` 和 `resolved.target` 的边界 `[0,100,20,100]`）。如果生产端测试套用了新规则，样本会变，要按 `AGENTS.md` 的流程导出、核对、替换，再跑两端的契约测试。
2. **第 13 条之后，调用者会看到的行为变化，指引要写。** 都来自 83 号记录 2.2 节的表，源码核过：
   - 后台屏上，输入框被滚出屏外时，`fill` 的读回改报"无法观察"（`UiActions.kt:830-831`）。
   - 原先被屏外同名文字"提前满足"的正向等待，改后会去等，第 17 条之后会等到超时。
   - 点击屏外的引用会被拒为"目标不可见"（`NodeActions.kt:110`），要先 `reveal` 或滚动。
   - 83 号记录建议在指引里加一句"点击被拒为不可见时，试试点它可见的子节点或用 `click --touch`"。
3. **按名字选择会解析到不可见的父节点（第四节第 3 处的展开）。** 读 `UiSelector.kt` 的实际行为：
   - 选择器只匹配可见节点（`UiSelector.kt:54`）。
   - 命中后取该节点的 `selectionTarget` 作为目标（`UiSelector.kt:47`）。`selectionTarget` 是吞掉它名字的可点击父节点，由 `UiSemantics.kt:355-362` 设置，不看父节点是否可见。
   - 父节点的可见性在选择阶段不检查。普通动作要等到动作前就绪检查时，才被拒为"目标不可见"（`NodeActions.kt:110`，原因码 `notReady`）。`locate` 会在等待循环里一直报"未就绪"，直到超时（`UiActions.kt:942-951`）。滚动查找的选择器模式则把这个目标直接过滤掉（`ScrollContainer.kt:91`）。
   - 所以"选择器……并拒绝"的说法在效果上对，但拒绝不发生在 `UiSelector` 里。三种命令各自的表现不同，测试要分开写。
4. **第 12 条的锁。** 到期在 `Sessions` 的锁内调用 `invalidateLease`，`ui.clear` 再把清理丢到另一个线程异步做（`Sessions.kt:321-327`、`UiBackend.kt:567-578`）。查询任务表要放在那个异步清理里、销毁后台屏之前，不能放进 `Sessions` 的锁里，否则一次卡住的系统调用会拖住所有会话命令。异步清理可能晚于下一条命令完成，要允许报"任务记录尚未完成"。另外，被拦下关闭时的空闲计时会被重置一次，那一刻的剩余秒数约等于整个空闲时长。
5. **第 18 条要同步收窄的指引。** `reference.md:52` 退出码 3 那一行里的 "cleanup/persistence" 包括本机连接没关好，改了退出码之后要相应改写。
6. **`--raw` 加投影选项会被静默忽略。** `--raw` 时正文就是整份原始 JSON，`--all`、`--depth`、`--max-chars`、`--interactive` 不起作用，也不报错（`ui_client.py:782, 803`）。83 号记录建议在参数阶段拒绝，或者明说被忽略；`--raw --no-print` 照常接受。第 23 条改默认之后，这些组合更需要写清。
7. **第 19 条的标记要按目标进程记，不能用全局布尔值。** 否则"停掉旧进程、又起了新进程、新进程崩溃"会被误认为主动停止。`onServiceDisconnected` 那一路不带目标（`lost(null, …)`，`WorkerConnection.kt:115-117`），要和当前连接的目标比较，或者只让死亡通知那一路负责重绑。两路都会进 `lost()`，第二次因为连接已空而返回（`WorkerConnection.kt:125-128`）。
8. **第 17 条的超时回执要补截断信息。** 文字条件在 0 命中时谓词是 null（`UiCondition.kt:196-206`）。截断信息 `limits` 现在只在提前失败时写（`UiCondition.kt:428`）。去掉正向条件的提前失败之后，超时回执里就没有任何截断信息，要主动补上 `complete=false` 和 `limits`。
9. **第 11 条要改的指引和帮助。** `SKILL.md:73-74` 和 `find --help`（`ui_commands.py:686-690`）都要补上"命中 0 条时新采一次"。
10. **第 10 条的两个边角。** 名字由空变成非空时，`UiRefRepair.kt:98` 不排除，引用修复可能成功，这条现有行为要保持并有测试。新字段不要放进 `resolved`，因为 `resolved` 在首行里表示"实际执行用的目标"（`ui_output.py:82-86`）。

---

## 6. 已核对无误的重要陈述（背书）

1. 第 1 条：9 个提交里只有第 1 个（`9d9d3b8`）和最后一个（`fb75afc`）的主机侧能导入。【实跑】用 `git archive` 导出每个提交的主机源码，逐个 `import androidctl.app`；中间 7 个分别报 `NameError`（`UiCommand`、`SCROLL_DIRECTIONS`、`clipboard`、`TyperGroup`）。
2. 第 2 条：版本完全相等的握手有三处。主机对设备：`transport.py:274`。设备网关对主机请求头：`NetworkGateway.kt:66`。应用对常驻进程：`WorkerConnection.kt:75`。主机命令行对本机后台服务另有：`local_server.py:64`、`local_call.py:129`。
3. 第四节第 5 处，"只改 `VERSION` 一行就够"：`git grep '0\.4\.0' fb75afc` 只命中 `VERSION:1`；`pyproject.toml:7, 60-61` 从 `VERSION` 动态读版本；Android 端在 `build.gradle.kts:22-41` 读 `../VERSION`；契约样本里没有版本串。
4. 第四节第 5 处，"可编辑安装的元数据不会自动更新"：`protocol.py:4, 11` 用 `importlib.metadata.version`；`venv6` 的 `androidctl-0.4.0.dist-info/METADATA` 写死 `Version: 0.4.0`，是安装时生成的。主机测试用的也是元数据里的版本（`test_global_options.py:13, 358`），所以版本不一致时测试照样通过。CI 每次全新安装（`.github/workflows/quality.yml:32, 46, 73`）。我是从机制判断的，没有另建虚拟环境实测。
5. A1 的六个编号正是"默认省略集合减去命名表"（`ui_view.py:55-69` 对照 `node_facts.py:42-72`）。
6. A2 的成因链：`UiBackend.kt:533` 在连接前标记"需要注销"；`AutomationCleanup.kt:32-41` 只试一次；`Sessions.kt:251` 的门。
7. 第 10 条：引用修复时新采的树经 `sample` 装成当前样本（`UiBackend.kt:401-413`）；新树不完整时报 `identityUnknown`（`UiRefRepair.kt:62`）；名字不同的候选直接排除，于是报 `stale, subject=ref`（`UiRefRepair.kt:72, 98`）；新采失败时只报拒绝。
8. 第 11 条：`find` 在主机端改写成带 `cached: true, find: true` 的 `snapshot`（`ui_commands.py:711-713`、`ui_client.py:167-169`），匹配在主机端做（`ui_view.py:939-998`）；"上一条动作未稳定就重采"在设备端（`UiSnapshot.kt:91-113`）；`snapshot` 本身接受 `scope` 参数（`ui_commands.py:558`），所以只改主机端就能实现。
9. 第 12 条：`status` 走 `callerStatus`，不续期（`WorkerService.kt:71-72`、`Sessions.kt:339-342`）；`snapshot` 和 `snapshot --cached` 都经 `acquireUi` 续期（`UiBackend.kt:114`、`Sessions.kt:223-235`）；到期时直接作废租约并清理（`Sessions.kt:321-327`）；维护线程每秒检查一次（`WorkerService.kt:40, 49-56`）。
10. 第 14 条的背景："主机现在没有默认的字符或深度上限"。`depth`、`maxChars` 默认是 None（`ui_client.py:637`）；`find` 只有默认 20 条命中的条数上限（`ui_view.py:103`）。
11. 第 13 条的背景数字：那份后台屏网页树有 4,976 个节点，4,972 个可见，4,426 个贴在屏幕下沿、高度为零，531 个真正在屏内。【重算】
12. 第四节第 1 处的离线数字：默认正文 435,415 字节，按第 13 条改标 4,441 个节点后降到 49,298 字节；`--all` 仍约 1.4 MB。【实跑】`fix13.py`
13. 第四节第 1 处："`--json` 只打印结果对象"（`ui_client.py:1047-1049`）；显式 `--out` 时不另存历史副本（`ui_client.py:896`）；四种存不成文件的情形都有对应代码：历史上限 150 个文件、20 MiB（`history.py:17-18`），保存前时限用完（`ui_client.py:888-891`）等。
14. 第 23 条：`--raw` 保存失败按交付失败算（`ui_client.py:975`）【实跑】；原始树附件的上限是 8 MiB（`protocol.py:19`、`Wire.kt:20`）。
15. 第四节第 2 处：引用身份是六个字段——来源节点编号、窗口、角色、名字、行标签、行列位置（`SelectionIdentity.kt:8-15`、`UiSemantics.kt:320-334`），不含可见位和边界；名字和角色的计算不读可见位（`UiSemantics.kt` 里只有第 138 行、第 399 行读它，分别管"名字是否被父节点吞掉"的显示和加载判断）。
16. 可见位进入整页签名（`UiSnapshot.kt:136`，经 `NodeFacts.observationFields` 含 `visible`、`bounds`）。
17. 可见位直接取平台值（`NodeFacts.kt:111`），采集顺序另读一次（`TreeCollector.kt:185`）。
18. 主机渲染遇到不可见节点时把子节点接到它的位置（`ui_view.py:363-365`）；稀疏编码里可见位默认为真，不继承父节点（`NodeEncoding.kt:39`、`node_facts.py:36`）；采集队列先按"是否不可见"排序（`TreeCollector.kt:61`）。
19. "主屏上 Chrome 已经把零面积、带正面积后代的节点标为不可见"：【重算】`inline2.py` 得到主屏 185 次全部不可见、后台屏 22 次全部可见，与 83 号记录的 186、22 基本一致。
20. 第四节第 4 处：两类退出都进 `lost()`（`WorkerConnection.kt:82-89, 115-136`），没有"主动停止"标记；`forceExecutor` 只销毁、不清连接（`WorkerConnection.kt:232-241`）；常驻进程只有两处 `exitProcess`（`WorkerService.kt:25, 147`），没有空闲自动退出；`bind()` 在 5 秒内不重复（`WorkerConnection.kt:155-160, 355`）。
21. 第 17 条和第四节第 8 处：连续两次截断的提前失败对所有等待条件都生效（`UiCondition.kt:424-431`）；等待消失要求完整的树、零匹配、零未知（`UiCondition.kt:175`）；`idle` 的稳定判断只接受完整的树（`UiPostAction.kt:183-197, 218`）。
22. 第 16 条：滚动查找选择器模式现在要求"恰好一个可见目标、树完整、引用可用、身份完整"（`ScrollContainer.kt:87-113`），多个命中报 `ambiguous`（`ScrollProgress.kt:64-68`）；`wait` 的选择器模式只要有可见命中即成功（`UiCondition.kt:143-159`）。
23. 第 17 条："按名字选择时有可见节点缺子节点才拒绝"（`UiSelector.kt:95-101`，经 `UiActions.kt:910-917` 以上限原因码拒绝）。
24. 第 18 条：退出码在 `protocol.py:80-82` 改成 3；调用点 `ui_client.py:661`、`app.py:383`、`job_commands.py:418, 499`；提示文字在 `output.py:49-55`。
25. 第 22 条：`BorrowedPaste.kt:119-125`——恢复失败时主结果改为失败，恢复状态未知时改为结果未知。
26. 第五节甲组的计数：【重算】`codes.py` 用字面量正则得到 153 个；补上三个由条件或默认值构造的码（`connectionFailed`、`publishFailed`、`workerFailure`），再把两个经条件构造、也用于设备端的码算回设备端（`publishUnsupported`，`TransferHandle.kt:158-159`；`unauthorized`，`NetworkGateway.kt:72`），得到 156 个，设备端 114、主机端 62、两端共用 20，与 82 号记录完全一致。真实运行里文本 `code=` 共 15 次（`notReady` 7、`staleRef` 4、`stale` 3、`arguments` 1）。【原件】
27. 第 1 组："三处走同一个检查函数"属实（`user_files.py:68-72`，调用在 `ui_client.py:373, 382`、`job_commands.py:438`）；唯一的测试断言在 `test_ui_arguments.py:564`；没有消费者按这两个码分支。
28. 第 2 组：`identityChanged`（`state.py:288-290`）和 `tlsIdentity`（`coordinator.py:302`、`transport.py:217, 284`、`transport.py:81` 的证书校验）判的是同一件事，提示都是 `device trust`；`transport.py:94-100` 里确实混着时钟和过期两种情形；没有分支、指引或契约用到 `identityChanged`。
29. 第 3、4、5 组的构造位置都属实：`UiDelivery.kt:98, 100`，`UiCondition.kt:335` 的分支集合；`local_server.py:171`、`file_jobs.py:554`；`GatewayClient.kt:34`、`transport.py:41` 对 `UiReceipt.kt:11`、`WorkerConnection.kt:191`、`transport.py:490`。
30. 第 6 组：6a 同一检查按命令分别报 `routeUnavailable` 或 `notReady`（`UiActions.kt:1007-1010`），文字相同；6b `VisualInput.kt:389` 对照 `userScope` 的 5 处；6c 主屏截图失败报 `screenshotUnavailable`（`VisualInput.kt:181-186`），后台屏报 `imageUnavailable`（`WorkDisplay.kt:136-146`），而读取端失败一旦发生就不清（`WorkDisplay.kt:121-130`）。三处都在派发之前，主结果都是"未发出"；主机、测试、指引、契约都没有按这些码分支，所以不改变任何退出码或主结果（主结果由 `UiRun.kt:124-130` 决定，只对 `wait` 的 `arguments`/`condition` 和 `snapshot` 的 `collectionProtocol` 看码）。
31. 第 7 组各码的构造位置都属实（`NodeActions.kt:83, 89, 91`；`PointerGesture.kt:179`；`PointerPlan.kt:52`；`ImageRegion.kt:189`；`NodeRange.kt:42-50`），没有消费者按它们分支。
32. 上一轮定为不合并的 6 个码是 `identityChanged`、`sessionChanged`、`workerLockChanged`、`staleWorker`、`sourceChanged`、`temporaryChanged`（`androidctl-r5-decisions.md` 第 15 题），决定文件的重审结论与 82 号记录一致。
33. 第五节乙组第 4 问引用的两处原句和行号属实（`input.md:61`、`reference.md:52`）；第 6 问"首页示例写了 `--display main`"属实（`SKILL.md:18`、`README.md:145`）。
34. 第六节：`shell` 文本模式每个流最多显示 4,096 字节，不自动保存文件（`job_commands.py:47, 262-277`）；`apps` 走 `device_call`，不保存文件（`app.py:516-519`）；路线题第一对是 4 GiB／4 核、后四对是 6 GiB／4 核（`ST/efficiency/core-results.md:604`）【原件】。
35. 第七节"节点的名字和行标签不随它在不在屏幕上而变"：第 13 条之后仍成立，因为名字、角色、行标签的计算都不读可见位（第 15 条）。变化只在显示层：父节点不可见时，子节点的名字不再被它"吞掉"（`UiSemantics.kt:138`），会在默认输出里单独出现。"回执只说主动作，退出码汇总整条命令"与第 18、22 条相容：第 22 条推荐项 A 正是"主结果完成、退出码表达清理问题"。

---

## 7. 没能核实的内容

- 第 1 条里"Kotlin 侧也有前后依赖"：没有 Android 构建环境，没有逐提交编译。
- 所有 Android 运行时行为：`singleTask` 在设备上的实际表现；A2 的同步抛错在 Android 14 到 16 上的具体异常类型；两路死亡回调的先后；Shizuku 13.1.5 是否会自动重启用户服务（83 号记录用的是上游主干，不是同一版本）；常驻进程死亡时后台屏上任务的去向。
- A1 统计里，`acts.py` 在约 24 万个旧格式节点上调用失败，所以"约 4.8 万次显示提示"只是量级；精确数字以候选保存文件里的 254 次为准。
- 第 9 条"7 个采集专用的隐藏成员"、第四节第 9 处"执行框架差约 1,150 token"、第四节第 1 处"`snapshot --json` 约 1.6 KB"：没有重核。
- 82 号记录的 10 类下一步动作分类、26 个和 27 个的计数、21 个混合码、第 8 到第 12 组的细节：只抽查了与决定文件相关的部分，没有逐码复核。
- 第五节乙组第 2 问的崩溃循环行为、第 18 条"本机后台服务会不会占着会话"：需要设备或真实的进程中断，无法离线判断。
