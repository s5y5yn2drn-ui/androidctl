# 第六轮专题核查：语义动作、文字输入、按键、剪贴板与动作后的结果判定

评审对象：产品 `fb75afc`（树与被实测的 `a27fe5aa` 相同），对照第五轮候选 `cab84b8`。源码引用默认指 `fb75afc`。真实任务统计用的是实施方交回的 37 份模型运行原件（其中 35 份计分）。没有设备，所有设备行为只来自实施方的模拟器原件。

记号：`S` = `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad`；`PKG` = `S/pkg9/androidctl-review-20261124T161212Z-fb75afc`；`ST` = `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`；Kotlin 文件都在 `android/app/src/main/java/com/rainng/androidctl/` 下，主机文件都在 `androidctl/src/androidctl/` 下。我的脚本在 `S/agents9/68/`（`tally.py`、`rollout_noise.py`、`noise2.py`）。

---

## 1. 结论摘要

**整体状态**：输入这一块的方向是对的，按第五轮裁决落实得基本到位，没有需要整块删除或重做的部分。写入通道分成三条（无障碍整字段写入、按键注入、借用剪贴板粘贴），每条都有清楚的"先检查、发一次、读回核对、四种结果"流程，在 35 次计分任务里 44 次 `fill`（16 次带 `--submit`）和 1 次 `ime` 全部 done，没有出现非 done 的输入结果。这一块本身不构成合入障碍。但是**和它相邻的默认输出有一处本轮引入的回退，应当在合入前修掉**。

最重要的发现：

1. **【必须修，本轮回退】默认观察把标准无障碍动作当成自定义动作打印**。网页和大多数原生节点都带"无障碍焦点""按粒度移动""按网页元素移动"这几个标准动作（编号 64、256、512、1024、2048），主机的动作名表里没有它们，`compact_actions` 先走"不在表里就按自定义动作处理"的分支，于是每个节点都多出 `action-name="action-40" action-name="action-100" …` 四五个记号，而且其中 4 个是需要参数、按 `action --action-name` 调用必然被拒的动作。实测在最终候选的 GitHub 网页任务里，这些记号占默认输出字符的 **31.5%**，Maps 17.8%，后台屏 Contacts 17.2%，Feeder 7.7%。第五轮候选 `cab84b8` 没有这个问题，是本轮提交 `65f6795` 调整判断顺序造成的，测试没有覆盖这几个编号。修正只需几行。【源码＋实跑＋重算】
2. **第五轮 A4 的"清空后读回占位文字算已清空"没有修到当初被误判的那个场景**。当初 3 次"清空后误判失败"全部发生在同一个网页编辑框上，我读了该页面 15 份原始树：这个编辑框的 `hint` 一直是空串，读回的"占位文字"其实是页面正文，不是平台意义上的提示文字。新规则"读回等于已知 hint 就算空"在这里不起作用；现在这类清空会报 UNKNOWN（退出码 3），而不是原来的 failed。评审方上一轮把它诊断成"提示文字"是错的，用户的裁决建立在这个错误诊断上。新规则在设备上也没有一个必须靠它才判对的案例。好在 UNKNOWN 是诚实的结果，不会把没写入报成成功。【原件＋源码】
3. **第五轮第 15 题"过程计数从默认文本去掉"只落实了手势，没有落实按键**。每次 `back`、`key`、`type --via keys` 成功时，默认文本仍打印约 400–700 字节的逐事件进度 JSON（`keys={"plannedEvents":2,…}`），定向按键还附带 `inputTarget`。实施方把裁决收窄成了"成功手势短回执"。【源码＋原件】
4. 借用粘贴的流程正确：恢复剪贴板在动作后观察结束之后、并先核对剪贴板仍是本次借写的标记内容，用户中途复制的内容会被保留；空字段粘贴不再依赖光标位置；换行装饰只把"`↵` 紧跟换行"归一。剩下一个分类不一致：粘贴已经核对成功、只是剪贴板恢复失败时，整条命令报 failed（退出码 1），这会诱导调用者重打，造成重复插入；参考页自己说的是"已知主结果可以保持 done、清理问题用退出码 3"。【源码】
5. 动作命令族本身偏宽但不冗余：`fill --submit`、`key enter REF`、`ime REF` 三种提交方式走的是三条不同的平台通道，各有不可替代的场景；真实任务里模型全部选了 `fill --submit` 且全部成功，首页的"基本操作"示例却写的是 `key enter`，建议换成 `--submit`。35 次计分任务里 `key`、`type`、`select`、`clipboard`、`action`、`check`/`uncheck`、`focus`、`long-click`、`range`、`dismiss`、`reveal`、`home`、`recents` 一次也没用过。不以此定去留。【重算】
6. 为网页读取保留的那个最小无障碍服务只有 10 行，不处理事件、不读窗口内容，和 UiAutomation 共存（连接时带了"不压制其它无障碍服务"的标志），是必要复杂度；删除输入法通道后，在 Compose 输入框上没有找到任何真实损失的证据。【源码】

**是否构成合入障碍**：输入、按键、剪贴板本身不构成。第 1 条是合入前应修的小回退（属于默认输出，几行代码加一个测试，修完能明显降低网页任务成本）；第 3 条是裁决落实偏差，也应在合入前补齐。其余是可选的整理。

---

## 2. 确定缺陷

### 必须修

**D1 默认观察把标准动作打印成不可用的"自定义动作短名"（本轮回退）**

- 触发：任何带"无障碍焦点"（64）、"清除无障碍焦点"（128）、"按粒度前进／后退"（256／512）、"下一个／上一个网页元素"（1024／2048）等标准动作的可操作节点。Chromium 网页节点几乎全部带这几个，原生节点普遍带 64。
- 位置：`ui_view.py:117`（`if identifier not in ACTION_NAMES:` 分支先于 `:132` 的 `DEFAULT_OMITTED_ACTIONS` 检查），`node_facts.py:42-72`（名表缺这些编号）。`cab84b8` 的同一函数先做省略检查，不会打印它们；改动来自提交 `65f6795 refactor(ui): simplify observation text and reference input`。
- 实跑：用 `S/r6/venv6/bin/python` 调 `compact_actions({'actions':[16,64,256,512,1024,2048,16908348],'chromeRole':'link','clickable':True})`，返回 `[{'name':'action-40'},{'name':'action-100'},{'name':'action-200'},{'name':'action-400'},{'name':'action-800'}]`。【实跑】
- 原件：最终候选的四组任务默认（不带 `--all`）输出里，这类记号的字符占比为 GitHub 31.5%（475,582 字符中 149,964）、Maps 17.8%、后台屏 Contacts 17.2%、Feeder 7.7%。在模型实际收到的工具返回里（含转义）GitHub 五次各占 28–30%。同类的 `action-1020037`（"滚动到位置"，需要参数）也被打印。脚本 `S/agents9/68/noise2.py`、`rollout_noise.py`。【重算】
- 影响：一是成本，网页任务观察正文约三成是这类噪声；二是误导，`action-100`/`200`/`400`/`800`/`1020037` 按 `action --action-name` 调用会被设备以"标准动作需要带参数"拒绝（`NodeActions.kt:120-134, 206-208`），`action-40` 能发但只是移动无障碍焦点，对调用者没有用处。这与第五轮第 13 题"默认输出变瘦、自定义动作只写标签或短名"的本意相反。
- 最小修正：在自定义分支之前先判断"是否标准动作编号"（单比特编号或 `android.R.id.accessibilityAction*` 范围），标准动作走原来的省略规则；需要参数的标准动作在默认输出里一律不打印。补一个包含 64/256/1024 的测试。
- 确定程度：源码、实跑、原件三方一致，确定。

**D2 按键的逐事件进度仍在默认文本里（裁决落实偏差）**

- 位置：`ui_output.py:463-477`（`input_facts` 只对 `tap/swipe/drag/pinch/timeline` 且 done 时省略），`ui_output.py:24-27`。
- 原件：最终候选四组任务里 20 次 `back` 的结果行全部带 `keys={"plannedEvents":2,"attempted":2,…,"omitted":{…}}`，合计约 8.3 KB；例如 `ST/efficiency/r5-c2-final-candidate-01/evidence/cli/` 下首个 `back` 的结果行长 934 字节。定向按键（`ST/runtime/api34/commands/2684-…`）还打印 `inputTarget={…}`。【原件】
- 依据：第五轮第 15 题写明"过程计数（例如手势的逐事件进度）从默认文本里去掉，在 `--json` 里保留"；实施方计划表把它写成"成功手势短回执"（`ST/r5-plan.md:91,97`），收窄了范围。
- 影响：成本不大（`back` 是第三常用命令，每次多 400 多字节），但它属于用户已定的事项。
- 最小修正：done 的按键结果不打印 `keys`、`inputTarget`；未完成时照旧打印。

### 可选（建议修）

**D3 借用粘贴：效果已核对成功、只是恢复剪贴板失败时报 failed**

- 位置：`BorrowedPaste.kt:120-126`：`restore=failed` 时整体结果为 FAILED（退出码 1），`restore` 为 pending/unknown 时整体为 UNKNOWN，即使 `effect=matches`。
- 影响：调用者看到 `type: failed` 容易重打，粘贴是插入，重打就是重复文字。这正是第五轮把"文字不同"从 failed 改为 UNKNOWN 的理由。参考页 `reference.md:52` 说"已知主结果可以保持 done、清理不确定用退出码 3"，与这里的做法不一致。真实原件里恢复都成功（`clear`/`restored` 且读回一致），所以这是低频路径。
- 最小修正：主结果只由派发和效果决定；恢复失败或未知作为单独的警告事实，退出码用 3。确定程度：源码推断，行为确定，频率低。

**D4 `clipboard get` 读到空但不能确认是真空时报 UNKNOWN**

- 位置：`ClipboardOperations.kt:145-155`。这是只读命令，没有任何派发，UNKNOWN 的含义（"不知道动作是否发生"）和首页的 UNKNOWN 处置规则（"先确认未发生再做新输入"）都套不上。建议改成 `failed` 或 `done` 加 `clipboard=unavailable`。成本小，频率极低（35 次任务未使用）。

**D5 `key` 的参数顺序与其它输入命令相反，报错也不提示**

- `fill [REF] TEXT`、`type [REF] TEXT`，但 `key KEY [REF]`。实跑 `androidctl key e15 enter` 返回"Use one supported editing key … for example, androidctl key enter"，没有给出 `androidctl key enter e15`。第五轮第 4 题要求"报错给出可照抄的写法"。【实跑】修正：第一个参数像引用、第二个参数是合法键名时，报错给出交换后的写法。

---

## 3. 设计与简化

### 3.1 命令族逐个说明

下表的"结果"一栏：not sent（退出码 2）表示确定没有发出输入；done（0）、failed（1）、UNKNOWN（3）只在已经开始派发后出现。所有命令在派发前都做同样的公共检查：引用的原始身份要能重新读到并且一致、目标在本次操作窗口（或显式范围）内、显示和锁屏状态允许输入；引用失效时在派发前最多修复一次（`UiRefRepair`）。【源码】

| 命令 | 发出什么 | 派发前另外检查 | 派发后核对 | 何时 failed / UNKNOWN |
| --- | --- | --- | --- | --- |
| `click`、`long-click`、`expand`、`collapse`、`dismiss` | 对应的无障碍动作，一次 | 可见、启用、节点公布了该动作；在预算内等待就绪 | 只看平台回执，再做动作后观察（不核业务） | 平台同步接受且回调为 false，或同步拒绝且回调非 true：failed；回调超时、回调解码错、接受时抛异常：UNKNOWN（`ActionReceipt.kt:245-253`） |
| `click --touch` | 节点中心坐标触摸，不指定接收方 | 新鲜位置、目标显示上可见任务里没有 AndroidCtl 自己（`AppInput.kt:73-82`） | 观察 | 注入未确认：UNKNOWN |
| `focus` | 无障碍 FOCUS | 已聚焦则不发、直接 done | 观察 | 同 `click` |
| `check`/`uncheck` | 无障碍 CLICK；控件本身不可点时点"合格的集合行" | 已满足则不发；已选中单选不能直接取消 | 在预算内读回勾选状态 | 读回不符：failed；读不到：UNKNOWN |
| `action` | 按编号/短名/标签选中的、节点当前公布的无参动作 | 选项唯一；标签要求本节点所有标签完整；需要参数的标准动作拒绝 | 观察 | 同 `click` |
| `range` | SET_PROGRESS（浮点值） | 在公布的范围内 | 读回 rangeInfo | 不符：failed；读不到：UNKNOWN |
| `select` | SET_SELECTION（UTF-16 偏移） | 原身份、输入焦点；已是该区间则不发 | 读回选区 | 区间符合 done；旧区间不变 UNKNOWN；其它区间 failed |
| `ime` | 无障碍 IME 动作 | 原字段已有输入焦点 | 观察 | 同 `click` |
| `fill` | 必要时 FOCUS，再 SET_TEXT 整字段替换 | 可编辑；当前值已等于请求值时两步都不发 | 读回原字段文字（只归一"`↵`加换行"） | 读回相符 done；读回不同或读不到 UNKNOWN 并附实际值；密码框按平台肯定回执 done、核对记为不可观察 |
| `fill --submit` | 上面之后：字段公布 IME 动作就发 IME，否则仅当已知单行字段时发一次定向 Enter | 只有写入核对相符（或密码肯定回执）才提交；提交前再核一次值 | 提交后另取观察 | 写入非 done 时提交为 not sent，前缀保留 |
| `type --via keys` | 用虚拟键盘字符表映射整串，逐事件注入；带引用时只发给原包的 UID | 带引用要求原字段已有输入焦点；不带引用要求当前焦点窗口不是 AndroidCtl；整串不可映射则整条不发 | 不核对文字 | 中途未确认：UNKNOWN 并保留已发前缀；首个事件被系统判定发错对象：not sent |
| `type --via paste` | 借用剪贴板写入标记内容，确认，再发一次无障碍 PASTE | 原剪贴板含 URI/Intent 拒绝；原剪贴板读到空且不能证明真空就不借 | 按原文和选区算期望值读回；观察；然后条件恢复剪贴板 | 见 3.3 |
| `key` | 一个键或组合键（键集很小） | 同 `type --via keys` | 不核对 | 同上 |
| `back` | BACK 键事件，不指定接收方 | 当前焦点窗口不是 AndroidCtl | 观察 | 同上 |
| `home`、`recents` | 通过 UiAutomation 注入系统策略键；只限主屏 | — | 观察 | 两个事件都返回 true 才 done，否则 UNKNOWN |

**重叠与容易选错的地方**：

- `fill`、`type`、`key`：`fill` 是整字段替换且核对；`type` 是在光标处插入，`--via keys` 只能打字符表里有的字符（中文整条不发），`--via paste` 会动用户剪贴板；`key` 只是一个编辑键。区分清楚，且 `type` 强制写 `--via`，避免了悄悄动剪贴板。`type` 和 `key` 带引用时要求字段**已经**有输入焦点，不会替你聚焦，粘贴前常需要先 `focus` 或 `click`，这是最容易踩的坑，指引写了。
- `click` 与 `action`：没有实质重叠，`action` 只用于自定义动作和没有专门命令的标准动作。
- 三种提交方式：`fill --submit` 自动在 IME 动作和 Enter 之间选；`ime REF` 只发 IME 动作；`key enter REF` 只发定向 Enter。三者在网页、Compose、普通 EditText 上行为确实不同（IME 动作可能插入换行，Enter 可能被吞或触发两次），保留三条是合理的。指引 `actions.md:136-139` 讲清楚了；但首页"基本操作"示例写的是 `fill` 后接 `key enter FIELD_REF`（`SKILL.md:42-43`），而真实任务里模型 16 次都用了 `fill --submit` 且全部 submit=done、从未用过 `key`。建议首页示例改为 `fill FIELD_REF "text" --submit`。
- `key` 的大写字母规则：`Enter`、`CTRL+A` 可以，单独 `A` 拒绝并给出 `key a`/`key shift+a`（`KeyPlan.kt:243-251`，主机 `input_keys.py:29-37`）。键集只有 enter/tab/del/escape/方向/行首行尾/forward-del 和 a/c/v/x/y/z，单独大写字母实际只会出现在误用里，报错可照抄，不算别扭，不值得再花用户时间。键集里没有空格和数字，需要时用 `type " " --via keys`。

**35 次计分任务的实际使用**（`S/agents9/68/tally.py`，按每条 CLI 调用的原始 stdout 统计）：

| 臂 | `fill`（不带提交） | `fill --submit` | `ime` | 其它输入命令 |
| --- | ---: | ---: | ---: | --- |
| 候选（18 次） | 15，全部 done | 8，全部 done，submit 全部 done | 0 | 0 |
| 基线（17 次） | 12，全部 done | 8，全部 done | 1，done | 0 |

非计分的两次历史运行另有 3 次 `fill`，也全部 done。也就是说，真实任务里没有任何一次输入命令的结果不是 done，A4 的修正、借用粘贴、按键、剪贴板都没有被任务路径用到；它们的设备证据全部来自实施方自己执行命令的功能检查。

### 3.2 文字写入的后态判定（`fill`）

流程（`UiActions.kt:597-797`）：同值跳过（不带提交时）→ 必要时 FOCUS 并确认输入焦点 → 再读一次，已等于请求值就不写 → SET_TEXT → 在"动作后观察的上限"（默认 1.5 秒）内反复读回原字段 → 可选提交 → 动作后观察。

- 换行装饰：`UiVerification.kt:73-76` 只把"`↵` 紧跟换行"归一。实施方在可见的网页三行输入框上实测 `verification=matches`、原始读回保留 `↵`，并有独立截图（`ST/r5-overall-validation.md:56` 所述的 2687/2688，我未逐张核图）。这一条修到了上一轮的 2 次误判。
- 读回不同报 UNKNOWN 并附 `actualValue`：`UiVerification.kt:139`（只对 `fill`；`check`/`range` 仍是 failed）。这满足第五轮第 6 题。
- 清空后读回等于 hint：只在"已接受的清空写入"之后启用（`UiVerification.kt:56-61`，`UiActions.kt:834`），写前不用它跳过清空，内部 review 的 MF1 已修。**但如摘要第 2 条所说，它没有覆盖当初真正误判的那个网页字段**：第五轮包里该页 15 份原始树（`S/pkg8/…/runtime/api34/verification/regex-visible-769/.androidctl/snapshots/`）中，那个多行编辑框的 `hint` 全部为空串、`showingHintText` 未设，"Insert your test string here, drop a file, or \nbrowse files\n. …"是正文。实施方的设备证据（`ST/real-tasks.md:806` 的 2336–2339）里清空后 `showingHintText` 从 false 变为 true，是旧规则判对的，新规则没有被用到。【原件】
- 会不会把没写入报成 done：只有两条路径不读回就 done。一是当前值已等于请求值，这是正确的空操作；二是密码框，按 SET_TEXT 的平台肯定回执报 done，指引写明了（`actions.md:129-130`）。没有发现把没写入报成 done 的路径。
- 会不会把成功写入报成 UNKNOWN：会，而且是设计上接受的。已知来源：App 改格式（电话号码、空格）、浏览器地址栏改写、页面"伪占位正文"、写入后字段消失或身份读不全。实施方原件里有一例：鸭鸭浏览器地址栏 `fill https://regex101.com/ --submit` 报 UNKNOWN，原因是写入后"当前选中身份不完整"，提交随之未发；实施方随后独立确认地址后用 `key enter REF` 继续（`ST/runtime/api34/commands/2681`–`2684`）。这说明 UNKNOWN 后有可行的出路。
- UNKNOWN 后的指引：首页说"先只读检查效果；业务上可靠成功就继续；否则先确认未发生……再出现 UNKNOWN 就停"（`SKILL.md:104-110`）。对 `fill` 而言，回执里已经带了读回值，只要值可接受就属于"可靠成功"，不会走到"再出现一次就停"。按字面读并不过度保守；但 UNKNOWN 在这里实际表示"写了，值和请求不同"，与"不知道发没发"共用一个标签，调用者要靠读 `actualValue` 区分。35 次任务里一次都没出现，没有证据支持再改裁决。
- 成本：成功时结果行会回显完整 `actualValue`（等于请求值），长文本时等于重复一遍；可以只在不相符时打印。小事。

### 3.3 借用粘贴

`BorrowedPaste.kt` 的顺序（`:101-137, 139-218, 275-327`）：读原剪贴板（读到空时用一次 AppOps 记账调用证明"确实允许读、确实是空"，否则不借）→ 写入带随机标记标签的纯文本 → 读回确认是本次借写 → 确认目标仍聚焦且公布 PASTE → 发一次 PASTE → 在观察上限内等字段变化并核对期望值 → 动作后观察 → `finally` 里读当前剪贴板，仍是本次标记内容才恢复原内容（原来为空就清空），恢复后再读回核对。

- "恢复不早于后观察且先比对"：成立（`:105-119` 观察在前，`:275-302` 先比对后恢复）。【源码】实施方原件里两次空字段粘贴都是"后观察结束后 restore cleared、读回相符"（`ST/real-tasks.md:814`）。【原件，实施方记录，我未逐件核】
- 并发复制：借写期间用户复制的内容会因标签不同而被保留（`restore=skipped`）。剩余窗口有两个，都很短：比对与写回之间（指引已写"非原子"）；读原剪贴板与借写之间（用户此时复制的内容会被借写覆盖，然后被恢复成更早的原内容，指引没写，影响可忽略）。
- 空字段：原文已知为空或明确显示提示文字时，期望值就是请求值，不需要光标位置（`:65-72`），修掉了上一轮的 D3；同一网页伪占位字段上仍会是 UNKNOWN。
- 结果分类：见缺陷 D3。

### 3.4 按键

- 键名解析在主机和设备各做一次（`input_keys.py`、`KeyPlan.kt`），规则一致。组合键按"修饰键按下 → 主键按下/抬起 → 修饰键逆序抬起"生成；中途失败会把仍可能按下的键逐个补发抬起（`KeySequence.kt:155-181`）。这套"可能按下集合"和补抬机制是必要的，否则一次失败可能让 Ctrl 卡住。
- 带引用与不带引用：带引用时要求原字段已有输入焦点，并且注入时指定接收方 UID 为原包的 UID，系统发现接收窗口不属于该 UID 会拒绝，首个事件被拒时可以确定没发出（not sent）；不带引用时只检查当前焦点窗口不是 AndroidCtl，然后不指定接收方发送，检查和发送之间换了焦点就可能发给别人（指引如实写了）。这是合理的分级。
- `fill --submit`、`key enter REF`、`ime REF` 的区别见 3.1，指引讲清楚了。

### 3.5 为网页读取保留的最小无障碍服务

- 做什么：`BrowserAccessibilityService.kt` 两个空回调；配置 `canRetrieveWindowContent=false`、`isAccessibilityTool=false`。它只是让 Chrome 认为"有无障碍服务在用"，从而保持网页无障碍树。删除输入法通道后它已经退回到最初的样子，与输入无关。
- 用户要做的事：在 AndroidCtl 管理页点"浏览器支持"跳到系统无障碍设置手动开启；侧载安装还要先在应用信息里允许受限设置（`connection.md:68-71`）。生命周期由系统管，开启后常驻，Gateway 不控制它。
- 与 UiAutomation 的关系：UiAutomation 连接时带 `FLAG_DONT_SUPPRESS_ACCESSIBILITY_SERVICES`（`UiBackend.kt:534`），两者共存。
- 没有它时：评审方早先引用的实测是 Chrome 在没有已启用服务时约 5 秒后关掉网页树（440 个节点降到 32 个），基线的网页正文几乎读不到（`/home/user/androidctl/review/sub-reports/01-node-architecture.md:29`、`08-reports-coverage.md:320`）。这一轮我没有重核那批原件。基线 GitHub 任务仍能 5/5 成功，是靠在地址栏填写带查询参数的 URL 绕过网页读取。
- 判断：必要复杂度，代码极小；代价是用户要手动开一个全局无障碍服务（可能让个别风控 App 改变行为，未验证）。保持现状。
- 删除输入法通道的损失：通道原来的独有价值是"不碰剪贴板、在任意编辑器里插入任意 Unicode"，主要正例是终端类 App。Compose 输入框支持 SET_TEXT 和 PASTE，`fill`/借用粘贴都覆盖；Maps 和 Contacts 任务里 23 次 `fill` 全部相符。我没有找到 Compose 上真实损失的证据。终端类 App 可以用 `clipboard set` 加 `key ctrl+alt+v`（键集允许）。删除是对的。

### 3.6 结构化剪贴板

- 能力：读出标签、MIME 顺序和每项的文字、HTML、URI、Intent 表示，不打开 URI、不执行 Intent；`set` 只写纯文本，`clear` 清空；两者都用读回判定 done/failed/UNKNOWN（`ClipboardOperations.kt:54-78`）。
- 后台读取：worker 以 shell UID 运行，以 `com.android.shell` 身份调用 `IClipboard.getPrimaryClip(pkg, tag, userId, deviceId)`；shell 持有后台读剪贴板权限，所以不需要前台。读到 null 时用 `IAppOpsService.noteOperation(OP_READ_CLIPBOARD)` 确认"允许读"，再加上未锁屏、用户 0、权限已授，才把 null 当作空（`ClipboardAccess.kt:131-172`）。最低 API 34，四参数签名与之匹配。厂商限制 shell 读剪贴板时会在借写之前拒绝，用户剪贴板不动（第四轮第 5 项）。【源码；厂商行为需设备】
- 写入时附加 SystemUI 的"不显示复制浮层"标志；按我对 AOSP 的记忆，该标志只对 shell 来源和模拟器生效，这里正好是 shell 来源。【推断，未下载上游核对】
- 与借用粘贴的互相干扰：两者都在同一个控制租约内串行执行，不会并发；`clipboard set` 之后再借用粘贴，粘贴结束会把 `set` 的内容恢复回来。没有状态干扰。

### 3.7 自定义动作

- 调用：`action REF --action-id N | --action-name NAME | --action-label LABEL` 三选一（`NodeActions.kt:156-210`）。按标签时要求本节点所有标签完整且标签唯一；按短名时要求唯一；需要参数的标准动作拒绝。
- 默认输出：自定义动作在标签完整、非空、唯一且不需要参数时写 `action-label="Share"`，否则写短名 `action-name="action-7f090011"`；带说明的标准动作写 `+long-click="show tooltip"`（`ui_view.py:106-144`）。设备端的接受条件和主机端的打印条件一致，符合第 13 题和第 6 问的裁决。Feeder 节点上实际打印了 8 个可直接照抄的标签。
- 问题只在缺陷 D1：标准动作被误送进了这个分支。

### 3.8 整体判断：规则数量和结果语义对 LLM 是帮助还是负担

- **结果语义是帮助**。not sent（确定没发，可以放心修正后重试）、done、failed、UNKNOWN 四种，加上"已执行前缀"和固定退出码，正好对应 Android 上真实存在的几种情况：Binder 调用超时、回调丢失、注入被系统拒绝、App 改写了值。Playwright 不需要这些，是因为它在同一进程内同步操作 DOM。对一个会重试的 LLM，"not sent 可重试、UNKNOWN 不要盲目重做"这条区分能直接避免重复下单、重复粘贴。
- **与 Playwright 相比，Android 平台必需的概念**：三条写入通道（字符表无法打中文、部分控件没有 SET_TEXT、后台屏不能用输入法）；写后读回（App 会改格式，平台回执不代表值）；按键的接收方限定（避免把按键打进 AndroidCtl 自己或别的 App）；剪贴板借用与恢复。
- **可以收掉或减轻的**：默认文本里的过程计数（D2）、成功时的 `actualValue` 回显、标准动作噪声（D1）；指引的密度。`input.md` 95 行、`actions.md` 的 fill 段，以及内部设计文档 `docs/reference/ui.md` 的输入段，几乎每句都是边界条件（"Restoring content does not restore owner, timestamps or history"、"Late App consumption may see restored/empty content"）。这些是诚实的，但 LLM 只在打开专题时读，日常成本有限；内部设计文档的输入段（例如第 147–197 行）密度已经高到难以维护，建议下一轮按"规则一句、理由一句"重写，不影响合入。
- **"清空后读回等于 hint 算空"这条新规则**：在设备上没有一个必须靠它才判对的案例，在唯一观察到的误判场景上又不起作用，却带来了 `afterAcceptedTextWrite` 标志、提交前核对的豁免分支、一轮内部 Must-fix 和三处文档段落。这是一处小的"钻牛角尖"：源头是评审方上一轮把网页伪占位正文误诊为提示文字，用户据此裁决"要做"。代价不大（十几行），保留无害；如果要简化，删掉它只会让少数"正文恰等于 hint 且未标记显示提示"的字段在清空后从 done 变成 UNKNOWN。我倾向保留但在指引里不再把它说成修掉了网页占位误判。
- **没有整块应当删、并、重做的部分。**

### 3.9 前几轮哪些结论现在看是错的

- 评审方第五轮子报告 50 的 D2 把网页清空后的读回诊断为"占位文字"，建议"回读值等于提示文字时视为已清空"。原件显示那是页面正文而非平台提示文字，建议本身对该场景无效。错误不大，但应更正。
- 第四轮第 5 项"多行网页输入框里带换行的文字会判为不符，指引里写明"已被第五轮取代，现在归一化有设备正例，取代是对的。

---

## 4. 证据审计

| 实施方声称 | 核对结果 |
| --- | --- |
| `fill`/粘贴只把 `↵\n` 归一成换行，没有 trim 或去标签（`ST/r5-text-postconditions.md`） | **成立**。`UiVerification.kt:73-76`；`actualValue` 保留原始读回（`:99-108`）。 |
| 已接受写入读回不同报 UNKNOWN 并附实际值；check/range 仍为 failed | **成立**。`UiVerification.kt:139, 147-151`；`BorrowedPaste.kt:209-216`。 |
| 清空后读回等于 hint 按空值，只在已接受写入后启用 | **代码成立**（`UiVerification.kt:56-61`，`UiActions.kt:834`、`:765`）。**但"修好了第五轮 A4 的占位误判"不成立**：当初误判的字段 hint 为空，见 3.2。实施方自己的总结里也只说"真实 Contacts hint/空 paste 证据"，没有声称网页清空重测过（`ST/r5-overall-validation.md:20,66`），措辞是诚实的。 |
| 空字段粘贴不依赖选区 | **成立**。`BorrowedPaste.kt:65-72`；实施方 Contacts 原件记录为 matches（`ST/real-tasks.md:814`，我只读了流水，未逐件核原始回执）。 |
| 恢复剪贴板在后观察之后，且先比对借写归属 | **成立**。见 3.3。 |
| 文字后验的改动"未进行设备验证"（`ST/r5-text-postconditions.md` 末段） | **如实**。之后的设备补点只覆盖了网页多行写入和 Contacts 写入/清空，网页清空没有重做。 |
| 第 15 题"默认过程计数收敛"（`ST/r5-plan.md:91`） | **部分成立**：手势成功时收敛；按键没有（D2）。 |
| 第 13 题默认输出变瘦、自定义动作只写标签或短名 | **部分成立**：自定义动作的写法正确；但同一次改动把标准动作送进自定义分支，默认输出反而变重（D1）。实施方的成本页用的是最终候选的实测数据，所以这部分成本已经计入"候选约为基线 0.48 倍"的结论里，结论本身没有失真，只是还有明显的下降空间。 |
| 按键名大小写不敏感，单独大写字母拒绝并给两种写法（`ST/r5-key-duration.md`） | **成立**。实跑 `androidctl key A` 返回 `A is ambiguous; use androidctl key a or androidctl key shift+a`，`code=arguments; argument=key`。 |
| 真实任务结果如实保留 | **成立**。35 次计分运行里 44 次 `fill` 与 1 次 `ime` 的原始 stdout 首行都是 done，退出码 0，与成本页一致；实施方功能检查里的 UNKNOWN（地址栏 `fill`）和失败（就绪等待超时）都原样保留在流水和原件里，没有被后来的成功覆盖。 |
| 测试是在验证行为还是复述实现 | 文字后验的 JVM 测试直接构造组合状态（实施方自己也承认，`ST/r5-overall-validation.md:20`），需要结合生产调用顺序读；主机 `compact_actions` 的测试只用了 16/32/4096/8192/16384 和一个自定义编号，没有覆盖最常见的 64/1024，所以 D1 漏过了。 |

---

## 5. 需要用户裁决的新取舍

没有必须由用户定的新取舍。D1、D2、D3、D5 都是实施细节或已定裁决的落实。

只有一处需要让用户知情：第五轮 A4 后半"清空后读回等于提示文字算已清空"是按评审方的错误诊断定的。可选做法：

- A：保留现状（规则无害，十几行），只把指引和内部文档里"修掉网页占位误判"的含义说准确。**推荐。**
- B：删掉这条规则，只保留"平台明确标记正在显示提示文字"的旧判定。省掉一个标志和一处豁免分支，少数字段的清空会从 done 变为 UNKNOWN。
- C：为网页伪占位正文另造识别规则（例如比较写前读到的"空状态正文"）。不推荐：没有足够样本，容易把真实正文误当空。

剩余未知：Chromium 普通 `<input placeholder>` 清空后是否标记"正在显示提示文字"，决定了这条规则在网页上是否有用武之地（见第 6 节第 2 条）。

---

## 6. 有判别价值的补验

1. **修掉 D1 后重跑一次 GitHub 网页任务的候选臂（1–2 次即可）**：预期默认观察字符下降约三成。如果下降明显，说明这一项值得在合入前修；如果模型轮数也随之下降，说明噪声影响了阅读，而不只是成本。这不改变"合入方向对"的判断，只影响成本结论的强度。
2. **在 Chrome 普通表单 `<input placeholder="…">` 上做一次 `fill REF ""`**：看读回是 done 还是 UNKNOWN，并记下 `hint`、`showingHintText`。若是 UNKNOWN，第 5 节的 A 方案需要在指引里说明"网页清空常为 UNKNOWN，看截图判断"；若是 done，说明旧判定已覆盖常见网页表单，新规则可按 B 删除。
3. **在一台真机或厂商 ROM 上做一次借用粘贴和 `clipboard get`**：看 shell 读剪贴板是否被限制、复制浮层是否真的被压制、恢复是否成功。结果会决定借用粘贴在真机上是"可用"还是"常在借写前拒绝"，影响 D3 的优先级。

---

## 7. 没能核实的内容

- 所有设备行为都只来自实施方模拟器原件；真机、厂商 ROM 上的剪贴板限制、复制浮层压制、隐私提示都未知。
- 网页三行输入的 matches 和 Contacts 空字段粘贴，我只读了实施方流水和部分回执，没有逐张核对截图。
- 鸭鸭浏览器地址栏 `fill` 写入后"身份不完整"的根因（是 App 改树，还是名字派生读不全）未查明，也不知道它在浏览器地址栏上是否普遍。
- `select`、`check`/`uncheck` 的行点击回退和 `range` 的核对，我只读了指引和主干源码，没有逐行审。
- SystemUI 复制浮层压制标志只对 shell 来源生效，这是我对 AOSP 的记忆，没有下载上游源码核对。
- 早先评审"没有浏览器服务时 Chrome 约 5 秒关树"的实测，本轮没有重核原件。
