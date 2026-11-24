# 第六轮专题核查 74：给调用者的指引、`--help` 与默认文本输出

核查对象：产品 `fb75afc`（`skills/androidctl/` 11 个文件、主机端 `ui_view.py`／`ui_output.py`／`output.py`、`S/r6/venv6` 中的 CLI）。对照上一轮候选 `cab84b8` 和基线 `7d5be70`。
真实输出原件：实施方材料包 `ST/efficiency/` 下 20 次计分候选运行（GitHub 网页任务 5、Feeder 阅读器长列表任务 5、后台屏 Contacts 任务 5、Maps 路线任务 5），另有 2 次不计分的历史候选运行只在说明处引用。这些运行的模型读到的 `SKILL.md`、`actions.md`、`displays.md`、`reference.md` 我逐字比对过，与 `fb75afc` 完全相同【重算】，所以原件就是本版本的真实输出和真实指引。
工作目录：`S/agents9/74/`（`S` 为评审暂存根目录）。脚本在 `S/agents9/74/scripts/`：`run_examples.py`（逐条实跑指引示例）、`index.py`（CLI 调用索引，结果 `cliindex.tsv`）、`noise.py`（默认输出各成分占比）、`residency.py`（指引驻留占比）、`mdreads.py`／`dumpread.py`（模型实际读到的指引正文）、`timeline.py`（轨迹时间线）。建议的首页全文在 `S/agents9/74/SKILL-proposed.md`。
说明：下文"引用"指打印给调用者的节点编号（如 `e15`）；"租约"指会话对设备显示目标的占用；"后台屏"指 AndroidCtl 创建的虚拟显示；"partial"指界面树采样不完整；"UNKNOWN"指命令不知道动作是否已经发生。

---

## 1. 结论摘要

**整体状态**：指引这一块方向对了，第一次达到"够用且比较便宜"。默认输出却有一处本轮引入的回归，把上一轮做掉的大部分精简又吃回去了。这处回归是一行级别的修正，不构成方向问题，但应当在合入前修掉。

最重要的发现：

1. **指引总量从 160 KB 降到 69 KB（约 17k token），首页 130 行，覆盖了基本循环。**【重算】20 次计分运行里，指引驻留占全部输入的 **8.9%**（上一轮 21.2%）。按任务分：GitHub 5.3%、Maps 5.2%、Feeder 10.6%、后台屏 16.9%。读完首页后又读专题的运行有 9/20：后台屏 5/5 是首页要求的，Feeder 3/5，GitHub 1/5，Maps 0/5。`--help` 在 20 次里一次都没被调用。实施方按第 12 题应当报告的"指引占输入比例"，在交付里全部写成 N/A，理由是"分词器未知"。可是工具回执本身带有 `original_token_count`（首页 1,654），完全可以算出来。这是一个小的钻牛角尖。
2. **指引里 81 条命令示例在 `fb75afc` 上逐条实跑，没有一条在参数解析阶段被拒。**【实跑】被拒的只有两个本来就是占位写法的例子：`COMMAND --help`，以及正文里描述行格式时写的 `androidctl key=value`。实跑的数字常量（8192 节点、8 MiB、3 s、JPEG 80、150 文件／20 MiB、128 KiB、64／96 KiB、各预算）都与源码一致。`androidctl help 主题` 输出的就是 `skills/` 里同一份文件，逐字节相同。
3. **必须修：默认输出把应当省略的标准动作当成"自定义动作"逐节点打印，占候选全部 stdout 的 23.5%。**【源码】【原件】【实跑】每个节点都带 `action-name="action-40"`，Maps、网页节点还带 `action-100`／`action-200`／`action-400`／`action-800`。这些其实是无障碍焦点、按粒度移动、按 HTML 元素移动等普通动作。根因在 `ui_view.py:113-125`：先判断"不在命名表里就当自定义动作"，然后才判断"是否属于默认省略集合"。64、128、256、512、1024、2048 都不在命名表里，所以永远走不到省略分支。这是本轮提交 `65f6795` 引入的，`cab84b8` 没有这个问题。测试没有覆盖。指引 `observations.md:61-62` 写的"默认省略普通焦点／选择／导航动作"因此与实际输出不符。
4. **必须修（第 15 题未落实）：`back` 的默认文本打印完整的逐事件按键进度 `keys={"plannedEvents":2,...}`，每次约 550 字节，20 次 `back` 次次都有。**【原件】【源码】`ui_output.py:463-475` 只对手势做了精简，按键没有。
5. **默认输出里另有约 10% 是调用者几乎用不上的内部字段**，第 13 题的"短溯源"只做了一半。【原件】这些字段包括：事件块的头行和逐事件原始字段（4.2%）、`observation {...}` 行（2.3%）、行列序号（2.1%）、`edge-windows` 行（1.1%）、截图时整段 `image {...}` 形状 JSON、`open` 回执里的 handler／preflight／landing 任务 JSON、`session close` 的整份任务清单、被拒回执里的 `cached; sequence=…; sample=…ms; age=…ms; timing=before-action`。
6. **参数报错基本做到了"可照抄"（第 4 题），运行期的拒绝还没有做到。**【实跑】【原件】例如引用过期时报 `No current target matches the original ref signature; code=stale; subject=ref`，没有告诉调用者"重新 snapshot"，Maps 5 次里有 3 次中招；点击屏外目标时报 `target is not visible`，没有写出下一步。

**是否构成合入障碍**：指引本身不构成。第 3、4 条都是小改动，但属于"用户已裁决、实施方声称已完成、实际没有做到"，建议列为合入前必须修。其余各条是可选的精简，可以在主线上继续做。

---

## 2. 确定缺陷

### 必须修

**D1. 标准动作被当成自定义动作逐节点打印（默认输出回归）。**【源码】【原件】【实跑】
- 触发条件：任何公布了无障碍焦点（ID 64）、清除焦点（128）、按粒度移动（256／512）、按 HTML 元素移动（1024／2048）而又没有标签的节点。实际上几乎所有 Compose 节点和网页节点都满足。
- 位置：`androidctl/src/androidctl/ui_view.py:110-125`。`if identifier not in ACTION_NAMES:` 分支排在 `DEFAULT_OMITTED_ACTIONS` 判断之前，`node_facts.py:42-72` 的命名表里没有这几个 ID。引入提交是 `65f6795 refactor(ui): simplify observation text and reference input`。`cab84b8` 的同名函数先判断省略集合，没有这个问题。
- 实跑：`compact_actions({'actions':[64,16,32,128,1],'clickable':True})` 返回 `[{'name':'action-40'}, 'long-click', {'name':'action-80'}]`。
- 原件：20 次计分运行的全部 CLI stdout 共 1,487,127 字节，其中这类片段 348,780 字节，占 **23.5%**。按 ID 计：`action-40` 出现 5,524 次，`action-400`／`action-800` 各 2,304 次，`action-100`／`action-200` 各 2,010 次（`scripts/noise.py`）。Feeder 每一行都是 `item e… "…" clickable +long-click action-name="action-40" action-label="Toggle read status" …`。
- 影响：
  - 默认输出白白多出将近四分之一。
  - 模型可能以为节点有一个叫 `action-40` 的"自定义动作"可以调用。
  - 指引里"默认省略普通焦点／选择／导航动作"的说法与实际输出不符。
  - 实施方在本轮测得的成本数字也包含了这部分膨胀。
- 最小修正：在 `compact_actions` 里先判断"无标签且属于默认省略集合"再跳过；或者把这 6 个 ID 加进命名表。补一条测试，用含 64、256 的真实节点断言默认文本中没有 `action-name`。
- 确定程度：读源码、实跑函数、原件计数，三方面一致。

**D2. `back`（以及走按键通道的命令）在默认文本里打印逐事件进度。**【原件】【源码】
- 原件：`ST/efficiency/r5-s1-core-candidate-03/evidence/cli/…/child.stdout` 中 `back` 的首行带 `keys={"plannedEvents":2,"attempted":2,"returned":2,"successfulPrefix":[0,1],"syncReturned":2,"lastReturned":{…"downTime":298218,"eventTime":298229,…},"inFlight":null,…,"omitted":{…}}`。候选运行里 20 次 `back`，20 次都有。
- 位置：`ui_output.py:463-475` 的 `input_facts` 只对 `inputTarget`／`point`／`gesture` 做了精简（`concise_gesture`），`keys` 总是打印。
- 影响：违反第 15 题"过程计数从默认文本去掉、留在 `--json`"。单次多约 550 字节，量不大，但正是用户点名要去掉的那一类。
- 最小修正：成功完成（`stop="completed"`、没有 `possibleDown`／`cleanup`）时不打印 `keys`；未完成时保留，与手势一致。
- 同类可选项：`scroll-until` 首行的 `step={…"platform":{"accepted":true,"callback":true}…}` 和 `match={"sequence":…,"sampleMs":…}`，也属于过程事实。

### 可选（便宜，收益明确）

**D3. 运行期拒绝不给下一步写法。**【原件】
- `click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action`。Maps 5 次中 3 次出现（`u35-c1-candidate-01/02/03`），原因是点击路线标签页时页面还在"Finding best route..."。模型每次都多花一轮 `snapshot`。建议改成 `the screen changed; run androidctl snapshot and use the new ref`。
- `click: not sent; Original target is not ready for click: target is not visible; code=notReady; cached; sequence=30; …`（`r5-c2-final-candidate-05`）。模型随后去读了 `actions.md`（2,779 token），才改用 `scroll-until`。建议写成 `scroll its container (androidctl scroll-until CONTAINER down --contains "…") or reveal it`。
- `wait: failed; Observation failed: Condition observation completed after the request cutoff`（`r5-c2-final-candidate-03`）。没有提示加大 `--timeout`。
- 对照：滚动方向被拒时，回执已经给出了可照抄的命令（`use androidctl scroll e124388171 forward`），做法是对的，应当推广到上面几处。

**D4. 参数报错仍有几处不符合第 4 题。**【实跑】
- 由 Click 自己生成的报错，例如 `Missing option '--max-steps'`、`Missing option '--via'`、`Missing argument 'y2'`、`Missing argument 'query'`，只给出"去看 `--help`"，没有可照抄的写法。
- `session open` 缺 `--display` 时，报错里出现字面的 `\n\t`：`Choose from:\n\tmain,\n\tvirtual`。
- 可照抄的例子用的是固定占位 `e15`，没有换成调用者实际给的引用（`scroll e1 diagonal` 返回 `use androidctl scroll e15 down`）。照抄会作用到错误的目标上。最好用调用者自己的引用。
- `notifications get '-1|…'` 不加 `--` 时，报 `No such option: -1`，没有给出 `-- '-1|…'` 的写法。C8 只要求改帮助和指引，这两处都已经改了。
- `help skill`（小写）被拒，只接受 `SKILL`。

**D5. 未连设备时的回执自相矛盾。**【实跑】
- `click: not sent; No paired devices; …; dispatchStarted=unknown`：结果写的是"未发出"，后面又写"不知道是否开始发送"。
- 读命令在同样情况下附带 `Observation: unavailable; run snapshot.`，可此时 `snapshot` 同样会失败。
- 设备选择阶段就失败了，应当写 `dispatchStarted=false`，并去掉 `run snapshot` 的建议。

**D6. `find` 零匹配时的提示过时了。**【源码】`ui_view.py:988-993` 仍然写 `Use scroll-until … with a target selector`，而首页推荐的是 `--contains`。也没有代入真实的容器引用。

**D7. 帮助文字串用。**【实跑】
- 所有动作命令的 `--settle` 说明都带 "check/uncheck verification can continue beyond --settle"，`click`、`scroll`、`key`、`back` 上也有。
- `snapshot --all` 的说明里出现 "find: show all deepest matches"。
- 每个子命令的帮助都重复 4 个全局选项，约 650 字节。全部 63 页帮助合计 93 KB。
- 实际运行中模型从不读 `--help`，所以这几条只影响人。

---

## 3. 设计与简化

### 3.1 指引：逐文件阅读（以首次接触的 LLM 调用者身份）

**首页 `SKILL.md`（130 行，约 1.65k token）。**
- 读完首页能不能完成普通任务：能，有两个缺口。
  - 打开应用：缺 `apps QUERY` 查包名，它只在 `actions.md` 的 Navigation 一节。真实运行里，后台屏任务的模型自己调用了 `apps`。
  - 读懂树行：首页没有展示一行树的样子，也没有解释 `+long-click`、`action-label`、`scroll=` 是什么。这些解释只在 `observations.md:56-66`。
  - 其余几项（找到并点击、填表、滚动查找、等待、截图、收尾）首页都有示例。Maps 5/5 和 GitHub 4/5 只读首页就完成了，这是证据。
- 需要内部知识或有歧义的句子（建议改写或删掉）：
  - "Use matching Host/App versions and user-prepared shell-UID Shizuku/Gateway." 调用者做不了任何事，应移到 connection。
  - "Same-name callers share a lease, including `default`; respect `busy` and `connectionBusy`." 这里的"lease"没有解释。
  - "When the first result line says `page may still be transitioning`, the next `find` refreshes once in the same scope; otherwise it reuses the current sample." 这是机制描述。调用者需要的只是"这时先 snapshot 或 wait"。
  - "Partial sampling can establish positive facts, but cannot prove absence. Selector uniqueness has the limited truncation exception." 普通任务用不上，应移到 observations。
  - "Do not repeat a done action for an unchanged/partial observation or nonzero exit. Obtain missing delivery or resolve cleanup instead." 读不懂：什么叫"获取缺失的交付"？
  - UNKNOWN 一节要求"establish non-occurrence and operation end … use resource-release checks"，链接到 `reference.md` 的工人 UUID 比对规则：必须是同设备、同会话、非缓存的新 snapshot，并比较 `data.capture` 冒号前的部分。这要求调用者手工做一个内部一致性检查。见 3.3。
- 与专题重复的内容：
  - 首页的 unsettled-find 规则与 `observations.md:5-8`、`observations.md:105-107` 重复了三处。
  - 文件保存规则与 `observations.md:29-33`、`images.md:66-76` 重复。

**`observations.md`（173 行，约 2.4k token）。** 一次都没被读过（本轮 20 次，上一轮 54 次）。
- 应当留在公开指引里的：树行格式（第 56-66 行）、窗口范围的选法、`wait` 的几个条件。
- 应当挪到内部 docs 的：
  - "Raw keeps the original missing list; compact lines keep relevant displayed-node gaps …"，第 95-105 行全段。
  - 保留现场、比对 capture 与序号的取证步骤（第 101-105 行）。
  - "Screen-change requires the same session's current complete published tree and matching normalized scope policy …" 这段可以压成一句："`--screen-change` 需要先有一次完整的 snapshot"。
  - "An unmet wait stops after two consecutive samples hit the same node/byte limit"。
  - Recent Event Facts 全节。

**`actions.md`（195 行，约 2.8k token）。** 本轮被读 4 次，3 次发生在 Feeder 的 `scroll-until` 成功之后。原因是匹配项在屏幕边缘，模型不确定能不能直接点。首页加一句"边缘的匹配项可能需要再滚一次"，就能省掉这三次读取。
- 可以删或挪走的：
  - "Ref Identity And Repair" 两段（第 7-21 行）。修复是自动的，调用者只需要知道"引用过期就重新 snapshot"。
  - "A sample capped only by nodeLimit/treeByteLimit can support ordinary unique selection …"（第 31-35 行），属于内部判定规则。
  - Scroll-until 停止条件那一段（第 82-89 行）只需留一句。
  - Fill 一节的"↵ followed by a newline"、提示文本等于空值、`--settle 0` 仍做第一次本地读取（第 141-155 行），属于实现细节。
- 需要改写的：
  - "Physical directions can select the corresponding page action or a supported generic route; `selected.action` reports the choice. No alternate route follows dispatch." 这一句塞了三条规则。

**`connection.md`（138 行，约 1.7k token）。** 对调用者最有用的是：配对、`use`、ADB 转发、"重启后要用户重新开 Shizuku"、升级前先 `daemon stop`。
- 应当挪到内部 docs 的：
  - "Device Diagnostics" 一节（`dumpsys` 命令、锁文件路径）。
  - 证书续期与撤销一段（第 98-101 行）。
  - "DaemonVersion rejects a differently versioned running daemon … about five continuous minutes" 这一段，对调用者只需留"版本不同就先用旧版 `daemon stop`"。

**`reference.md`（126 行）。**
- 退出码表有用，预算表有用。
- "UNKNOWN And Resource Release" 和 "JSON Field Guide" 两节是给内部用的。
- 后台屏 5 次运行都把 reference 和 displays 一起读了（3.2k token），推测是首页 UNKNOWN 一节链接到这里所致。

**`displays.md`（99 行）。** 后台屏任务必读，内容大体必要：任务搬动、销毁、`--return-tasks`／`--discard-tasks`。"Closing And Cleanup" 一节可以压成两句。

**`images.md`、`input.md`、`jobs-files.md`、`clipboard.md`、`notifications.md`。** 本轮基本没被读（clipboard 1 次）。内容准确，但同样是电报体，一句里塞多条规则。
- 例：`input.md` 借剪贴板粘贴一节有 6 段，大约 20 条规则；`jobs-files.md` 的发布边界整节是安全论证。
- 这些主题的使用频率低，压缩它们不是当务之急。

**压缩是否过头。** 是的，主要问题在专题，首页还好。
- 典型句：
  - "Explicit `page-*` requires that exact advertised action."
  - "Fields resolve independently; invalid configured devices do not fall back."
  - "Reliable local refs/positive values can remain useful."
- 这些句子省略主语，用斜杠连接并列项，一句塞两三条规则，模型容易误读成"可以"或"必须"。
- 正确做法是少写规则，不是把同样多的规则压短。上一轮 160 KB 的内容大多被压缩了，删掉的不多。

**删掉的内容里有没有调用者其实需要的。** 我对比了 `cab84b8` 版的 `actions.md`（30 KB）、`observations.md`（35 KB）、`connection.md`（20 KB）。删掉的主要是：
- 引用保留与修复的时序；
- 滚动方向的映射规则（单列集合把 up/down 映射到 forward/backward，ViewPager 例外）；
- 保存权限修复；
- 缺口汇总算法。

没有发现调用者必需、现在只能靠试错得到的内容。滚动方向那一条，现在由被拒回执给出可用方向来兜底，这种做法更好。我没有逐句比对全部删减，属于抽样结论。

### 3.2 默认输出：逐行评价（真实原件）

以下摘自 `ST/efficiency/u35-c1-candidate-01/evidence/cli/1795520476637502800-…/child.stdout`（Maps，`click` 之后）：

| 行 | 评价 |
| --- | --- |
| `click: done; change=changed at windows,content baselineAgeMs=9763 (original target unknown: Original target is not identifiable in the final observation); snapshotPath="…"` | 需要的是 `done`、`changed` 和路径。`baselineAgeMs` 和括号里的说明对 `click` 没有用。 |
| `androidctl capture="f09d…:10" connection=1 sequence=24 finishedMs=248638 snapshotAgeMs=47` | 只有 UNKNOWN 取证时才用到 capture。可以只留 `capture` 和年龄。 |
| `observation {"scope":[{"window":22,"state":"full","package":"…maps"}],"target":{"displayId":0}}` | 全部完整时，这一行与下面的 `screen`／`window` 行重复。partial 时才有用。 |
| `screen app="…maps" title="Maps" keyboard="hidden"` | 有用。 |
| `window id=22 active focused content="full" selected`，`edge-windows count=1 content="notRead" windows=[…]` | 几乎没用。状态栏、导航栏没读，这本来就是常态。 |
| `events state="available" source={…"receivedClock":"elapsedRealtime","eventClock":"uptimeMillis",…} range={…} retained={…} incomplete=false` | 纯内部字段。 |
| `event seq=142 receivedMs=… eventTime=… type=32 displayId=0 windowId=22 contentChangeTypes=16 windowChanges=0 kind="pane" contentState="available" packageName="…" text=[…]` ×3 | 有用的只是 `pane: "Start location …, Destination …"`，而且三条是同一句话。可以压成一行并去重。 |
| `item e124387726 "Walking mode: " clickable action-name="action-40" roleDescription="Tab" rowIndex=0 columnIndex=2` | 引用、角色、名字、`clickable` 都有用；`action-name="action-40"` 见 D1；行列序号在多数页面上没有用。 |
| `text "Driving mode: " selected` / `text selected` / `text selected` / `image selected` | 空节点链没有被折叠。 |

其他命令（均为原件）：
- `snapshot --screenshot`：多出一整行 `image {"id":…,"shape":{"windows":[…],"magnification":{…}}}`。调用者需要的只是 `imagePath`、尺寸和 `id`（坐标输入要用 `id`）。首行已经有 `imageSize=…; imageMappingKnown=true`，所以这一行大部分是重复。
- `open com.android.contacts`（后台屏）：首行约 1,900 字节，带 `launchCode`、`launchCodeName`、`resolvedType`、`handler{…}`、`preflight{…}`、`landing{matchingTasks…, visibleTasks…}`。调用者需要的是 `done`、落在哪个显示上，以及（后台屏时）有没有搬动别人的任务。
- `session close`：`taskDisposition={…openedTasks:[{baseActivity, topActivity, realActivity, origActivity, baseIntentComponent, topActivityInfo…}]…}`。状态为 `clear` 时只需一句 "destroyed display 2; no foreign tasks"。
- `session open`：`session open: done; Target established`，`target={"kind": "virtual", "displayId": 2}`。紧凑，好。
- `status`：`gateway={…"tlsActive":0,"tlsQueued":0,"activeQueries":1…}` 属于内部字段。
- `fill`：`fill=done, verification="matches", actualValue="National Gallery, …"`。有用，正是调用者需要的。`focus=done, sent=false` 可以省。
- `scroll` 被拒：`current scroll directions: forward, down; use androidctl scroll e124388171 forward`。好，可以照抄。
- UNKNOWN（`ST/runtime/api34/commands/2681-…/stdout.bin`）：`fill: UNKNOWN; … fill=UNKNOWN, reason="Complete current selection identity is unavailable", sent=true, verification="unobservable" …; submit=not sent …; Do not repeat automatically; first check the effect with read-only observations.` 结尾那句是对的，但它前面是一长串字段。建议把这句提到首行紧跟 `UNKNOWN` 之后。
- `apps`：每行"包名 名字"，紧凑，好。

**第 13、14、15 题逐条落实情况**（原件 309 份带树输出）：

| 裁决 | 状态 |
| --- | --- |
| 13：网页节点去掉重复的 `context-click` | 做到。只在 `--all` 里出现（12 份，全是 `find --all`／`snapshot --cached --all`）。 |
| 13：溯源改短行 | 部分做到。`androidctl capture=…` 是短行，但又多了 `observation {…}`，截图时还有 `image {…}` 和 `windows-source {…}`，开头 3–5 行仍是元数据。 |
| 13：单窗口省窗口行 | 做到（71 份没有窗口行）。目录里有输入法或边缘窗口时仍打印。 |
| 13：自定义动作只写标签 | 标签部分做到（Feeder 的 `action-label="Share"`）。但 D1 让标准动作以 `action-name` 形式大量出现，与本题目的相反。 |
| 13：相对路径 | 做到，相对于 CLI 当前目录。测试环境的当前目录是仓库根，所以路径仍有约 170 字节，并且 JSON 引用让 Windows 反斜杠成对出现。这是测试环境造成的，不算缺陷。 |
| 13：`+long-click="show tooltip"` | 做到。 |
| 14：引用写成裸 `e15` | 做到，没有任何符号。但编号是 9 位随机起点（`e488674553`），来自基线提交 `dc9f569`，用意是让旧进程的引用不会被新进程误认。代价是每个引用多 6–7 个字符，引用总共占 4.1% 字节。是否改用会话内的小编号，见第 5 节。 |
| 15：过程计数从默认文本去掉 | 手势做到；按键（`back`）没有做到，见 D2。`scroll-until` 的 `step`／`match` JSON、被拒回执的 `sequence/sample/age/timing` 仍在。 |
| 4：报错给可照抄的写法 | 自写的参数报错大多做到。Click 生成的报错和运行期拒绝没有做到，见 D3、D4。 |
| C8 | 帮助与指引已改：`snapshot --help` 和 `screenshot --help` 都写了"Standalone screenshot has no node tree to label"，`notifications.md` 写了 `--`。`screenshot --marks` 仍被接受，并且仍然产出 0 个标注。 |

**树的行格式。** "角色、引用、带引号的名字、裸布尔值、`键=JSON`"这个格式紧凑，没有歧义，好读，应当保留。三处可改：
- 默认省略 `rowIndex/columnIndex`；
- 折叠只有 `selected` 的空节点链；
- 边界框只在 `--boxes` 时出现。当前已经是这样，不需要改。

### 3.3 与 Playwright CLI／MCP 的比较

公开资料：Playwright CLI（`github.com/microsoft/playwright-cli`）用 `-s=name` 选会话，引用是 `e15`，动作命令输出页面 URL、标题和一个快照文件链接，快照默认写到文件里，`--raw` 去掉页面状态；Playwright MCP 的 `browser_snapshot` 用 `[ref=e15]` 标出引用。

AndroidCtl 多出来的概念，以及我对去留的判断：

| 概念 | 是否必需 | 建议 |
| --- | --- | --- |
| 配对、设备选择 | 必需（远程设备，不是本机浏览器） | 留在 connection，首页一句带过。 |
| 会话与租约（`session open`、`busy`） | 必需：一台手机只有一块主屏，多个代理会互相干扰 | 留在首页，但不要用"lease"这个词。 |
| 后台屏、任务搬动、`--return-tasks`／`--discard-tasks` | Android 特有，有真实风险 | 留在专题。首页只写"后台屏先读 displays"。 |
| 结果四态与退出码 | 必需。UNKNOWN 是远程设备真实存在的情况 | 留，但首页给出操作性写法（见 3.4）。 |
| "结果与进程退出码各管一摊"、交付失败、`exit 4` | 普通任务几乎碰不到 | 移到 reference，首页只说"退出码与结果一致"。 |
| partial 与"不能证明不存在" | Android 无障碍树确实常不完整 | 首页一句即可，细节移到专题。 |
| 过期引用与自动修复 | 修复是自动的 | 首页只写"引用过期就重新 snapshot"，修复机制移到内部 docs。 |
| unsettled 与 find 刷新一次 | 是本产品自己的机制 | 首页改成"看到 transitioning 先 snapshot 或 wait"，刷新规则不必让调用者知道。 |
| 保存文件、`--no-print`、`--json` 读 `snapshotPath` | 有用 | 留一段。 |
| UNKNOWN 后按工人 UUID 比对 capture 来判断"资源已释放" | 不必让调用者手工做 | 做成默认行为：下一次成功的 `snapshot` 若确认前一个操作已结束，就在输出里写一句（例如 `previous operation ended`）。 |
| 动作后默认打印整页观察 | Playwright CLI 默认只给链接 | 对 Android 来说，动作后看到新屏幕通常省一轮，保留默认打印是合理的。前提是先修 D1 并压掉元数据行。 |

### 3.4 首页改写建议

建议全文见 `S/agents9/74/SKILL-proposed.md`（约 120 行）。相对现版本改了这些：

1. 开头一句说清产品是什么（"类似 snapshot 加引用的浏览器工具"），删掉 Shizuku／Gateway 版本前提（移到 connection）。
2. 在"开始"里加入 `apps maps` 查包名。
3. 新增"读屏幕"一节，给出 3 行真实格式的树样例，解释 `+NAME`、`action-label`、`scroll=` 和缩进。这是首次调用者最需要、而现在只能去读 observations 才能得到的内容。
4. 动作示例补上 `fill --submit`、`check`、`wait --timeout`。加一句"边缘的匹配项可能需要再滚一次"，针对 Feeder 3/5 次去读 actions 的原因。
5. 结果表加上退出码。新增"常见拒绝"四条：`stale`、不可见、无滚动路线、超时。每条都写下一步怎么做。
6. UNKNOWN 改成操作性规则：不重发；只读检查；看见效果就继续；确认没发生可以重试一次；仍不确定或第二次 UNKNOWN 就停下并报告；付款、发送、删除类要有明确证据。
   - 这一条放宽了现行的"先用工人 UUID 证明资源已释放"。代价是：如果前一个动作在设备上还没结束，重试可能叠加。我认为在指引层面可以接受，建议同时按 3.3 把这项检查做成默认行为。这一点需要用户确认，见第 5 节。
7. 删掉 unsettled-find 刷新机制、truncation 例外、"obtain missing delivery"等内部表述。
8. 专题索引改成"什么情况读哪篇"的表格。

---

## 4. 证据审计

| 实施方声称 | 核对结果 |
| --- | --- |
| "公开 11 份收口：首页 128 行，10 个专题 1,124 行，合计 1,252 行"（`ST/r5-guidance.md`） | 基本成立。`fb75afc` 是首页 130 行，专题 1,131 行，合计 1,261 行（最终提交又小改过）。【重算】 |
| "首页补齐 open/back/scroll/key/fill 和读结果的基本循环" | 成立。缺口只有 `apps` 和树行格式说明。【源码】 |
| "真实 skill 导出 11 份逐字节一致；help 专题原文字节一致" | 成立。我实跑 `help 各主题` 与 `skill --out`，结果一致。【实跑】 |
| "默认单窗省窗口行……网页无说明的 context-click 默认省略……自定义动作只有在全节点标签完整……时写 action-label"（`ST/r5-presentation.md`） | 部分成立。三条本身都实现了，但同一次改动把标准动作误归为自定义动作（D1），实际效果与"默认输出精简"的目标相反。实施方自报的 966 项测试通过，没有一项用到 ID 64 之类的标准动作。测试是在复述新格式，没有检验省略规则。【源码】【原件】 |
| "成功手势省完整输入目标、点和逐事件进度" | 成立。但没有覆盖按键（D2）。【源码】 |
| 第 12 题"每次报告两个数" | 部分成立。"首页后专题次数"如实报告了，与我重算一致：GitHub 候选 C05 读 actions 1 次，后台屏 5 次各读 2 篇，Maps 0 次。"指引占输入比例"全部写 N/A。实际上工具回执带 `original_token_count`，又有逐响应 usage，完全可以计算。我算出的结果是 8.9%（按任务 5.2%–16.9%），`scripts/residency.py`。【重算】 |
| 失败与拒绝是否如实保留 | 成立。原件里的 exit 1／2 回执都完整保留（8 次滚动或点击被拒、2 次 `scroll-until` 失败、1 次 `wait` 失败）。计分候选运行里没有 UNKNOWN，我另从 `ST/runtime` 找到 5 份 UNKNOWN 原件。【原件】 |

---

## 5. 需要用户裁决的新取舍

**Q1. 引用编号的长度。**
- A：维持 9 位随机起点（`e488674553`）。优点是旧进程的引用不会撞上新进程，这是基线 `dc9f569` 的用意。代价约占输出的 2–3%，模型抄写长编号也更容易出错。本轮原件里没有观察到抄错。
- B：会话内从小编号开始（`e1`、`e2`……），在执行器重启时由主机拒绝旧编号。工人 UUID 本来就在 capture 里，可以用它来判断。
- 推荐 A，暂不改动。收益小，而且需要另做一套防误认机制。如果以后要做输出瘦身的第二轮，再考虑 B。

**Q2. UNKNOWN 之后的指引写法。**
- A：维持现行规则：先用同一工人的新 snapshot 证明前一个操作已结束，再输入。
- B：改用首页建议稿的操作性规则：只读检查，确认没发生可以重试一次，第二次 UNKNOWN 就停下。同时让 CLI 在下一次 snapshot 里自动报告"前一操作已结束"。
- 推荐 B。现行 A 要求调用者理解 capture 的格式，而且 20 次计分运行里没有用到过。剩余未知：自动报告需要设备端知道上一个操作已结束。这个状态现在已经存在（资源释放检查用的就是它），但我没有核实主机能否不加 RPC 就拿到。

其余都是实施细节，不需要用户裁决：D1–D7、元数据行精简、事件块压缩、专题去电报体。

---

## 6. 有判别价值的补验

1. **修掉 D1、D2 后，在 Maps 和 Feeder 上各重跑 2–3 次，看 CLI 输出驻留占比和总输入。** 预期默认输出减少约 20%。如果总输入几乎不降，说明成本瓶颈在系统提示和轮数上，输出瘦身的第二轮（元数据行、事件块）就不值得再投入。
2. **用首页建议稿替换现首页，在 Feeder 上跑 3 次。** 看"首页之后读专题"能否从 3/5 降到 0–1 次。如果不降，说明问题不在首页措辞，而在 `scroll-until` 匹配项落在屏幕边缘时的行为，应当改命令本身（例如完成前再把匹配项滚进可见区），而不是改文字。
3. **在一个含无标签自定义动作的应用上实跑一次 `action REF --action-name action-XXXX`**，确认 D1 修好后，真正的自定义动作仍然按短名打印并且可以调用。

---

## 7. 没能核实的内容

- 设备端行为一律没有实跑（本环境没有设备）。所有示例只核到"参数被接受、在连接阶段失败"这一层。
- `ui_view.py` 的修改是否在其他场景（`--all`、`find --context`、网页 `chromeRole`）下还有别的漏省，我只做了函数级实跑和原件计数，没有逐分支覆盖。
- `cab84b8` 与 `fb75afc` 的指引删减只抽查了 actions、observations、connection 三篇的主要段落，没有逐句比对。
- `original_token_count` 用的是执行环境报告的分词数，我把它当作模型分词数的近似值。驻留占比按"读入后每一轮都重新计入输入"估算，没有考虑执行环境可能做的上下文压缩（本轮各运行都远小于上下文窗口，影响应当很小）。
- Playwright CLI 的比较只依据公开的 README 摘要，没有实际运行它。
