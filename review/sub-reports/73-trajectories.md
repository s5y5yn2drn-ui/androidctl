# 第六轮评审子报告 73：真实任务模型轨迹的行为阅读

**评审对象。** 实施方本轮交回的 35 次计分运行，另有 2 次不计分的历史运行。本文用两个缩写：`PKG` 指材料包根目录；`E` 指 `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions/efficiency`。

**执行模型。** 全部是 gpt-6.1-sol，medium 推理档，每次运行都是全新会话（见各运行目录的 `registration.json`）。

**版本。**
- 候选 20 次用的是 `u34-contains-materials` 目录里的物料：主机端、指引和 APK。这批物料对应产品提交 `a27fe5aa`，它的源码树与 `fb75afc` 相同（`ST/u34-contains-materials/result.md`）。
- 我把模型实际收到的 `SKILL.md`、`actions.md`、`displays.md` 正文从轨迹里取出来，和 `fb75afc` 的 `skills/androidctl/` 逐字比对过，只差文件末尾一个换行。【原件】
- 基线用的是 `7d5be70`。Feeder 题的基线 5 次是复用旧运行的结果。

**工作目录。** `S/agents9/73/`（`S` 为本轮 scratchpad 根目录），内容如下：
- 时间线脚本 `timeline.py`，生成的逐次时间线 `tl/<运行>.md`、`.full.txt`、`.json`，汇总表 `tl/_summary.tsv`；
- 输出噪声统计 `noise.py`（结果在 `noise.txt`）和 `carry.py`；
- 命令分布统计 `stats.py`。

所有脚本都用 `python3 -I` 运行，材料包路径作为参数传入。

**阅读范围。**
- 候选 20 次逐轮通读：GitHub 网页题 5 次、Feeder 长列表题 5 次、后台屏 Contacts 题 5 次、Maps 路线题 5 次。
- 基线逐轮精读 6 次：Maps 第 2、3 次，GitHub 第 1 次（另外把第 2 与第 5 次、第 3 与第 4 次各自逐条对比过命令序列），Feeder 第 2 次。其余基线用脚本统计。

---

## 1. 结论摘要

**整体状态。** 拿到现在这份指引和工具的新调用者，能顺利完成普通任务：
- 候选 20/20 业务成功（基线 14/15）；
- 一次 `--help` 都没有读；
- 没有任何参数写法错误；
- 每次读完 6.6 KB 的首页指引后，第 2 到 3 轮就开始操作设备。

上一轮最大的两类摩擦已经基本消失：一是"先读 30 KB 专题"，二是"入口陷阱"。剩下的浪费主要来自**输出内容**和**动作后观察的取样时机**，已经与参数无关。

最重要的发现：

1. **入口陷阱清零。** 时长不带单位、引用带 `@`、键名大小写、`open --package` 这几类，在 20 次运行里都是 0 次。
   - `actions.md` 从 30 KB 缩到 11 KB，只有 4/20 次被读；上一轮是 5/5 次都读。
   - `scroll-until --contains` 在 Feeder 题和后台屏题上 9/9 次第一次调用就成功。
   - 上一轮的"必踩坑"只剩方向词一类：在只宣告 `forward` 的列表上写 `down`，出现 2 次。另有 2 次是在已到顶的列表上写 `up` 被拒。【原件】
2. **默认输出出现回归：本轮把标准无障碍动作编号重新印了出来。**
   - 现象：网页和 Maps 的几乎每个节点上都挂着 `action-name="action-40" action-name="action-100" action-name="action-200" action-name="action-400" action-name="action-800"`。
   - 占比：占 GitHub 题候选全部输出的 33%，Maps 题 18%，后台屏题 17%，Feeder 题 8%；在 `--all` 输出里占到 50%。
   - 原因：`ui_view.py` 的 `compact_actions` 先判断"不在已知名字表里"，于是 `DEFAULT_OMITTED_ACTIONS` 里列的 64、128、256、512、1024、2048 永远走不到省略分支。这是本轮提交 `65f6795` 引入的，`cab84b8` 上这些编号是被省略的。我在主机环境里实跑过，确认会这样输出。【源码、实跑】
3. **动作回执里的观察常常取得太早，或者把仍在加载的页面判为 `changed`。** 由此出现三种多余步骤：
   - 候选 20 次里共有 35 次"动作之后紧接着补一次 snapshot"，其中 25 次跟在 `unsettled`、`partial` 或"无树"回执之后；
   - GitHub 题 4/5 次点开 issue 后，回执报 `changed`，但标题仍是列表页；
   - Maps 题 3/5 次在"Finding best route..."阶段拿到 `changed`，随后点击出行方式标签被判 `stale`。【原件】
4. **引用过期被拒时，回执不带任何新事实，比基线多花一轮。**
   - Maps 题的 3 次过期全部是同一种情形：步行标签的名字从 `Walking mode: ` 变成 `Walking mode: 18 minutes`，节点还在原来的行列位置。
   - 基线的同类拒绝附带了一棵新树，模型下一条命令就直接点新引用；候选的拒绝只有 141 字节，模型必须先补一次 `snapshot`。【原件】
5. **用 `snapshot --cached REF` 读屏外节点时，返回的正文是空的，也没有任何提示。** GitHub 题 4/5 次因此先读一次空的，再加 `--all` 重读。【原件】
6. **后台屏 Contacts 题 5/5 顺利。** 会话、显示、关闭和资源核实都没有出现困惑；`session close` 加 `status` 的回执足以让模型如实报告释放结果。

**是否构成合入障碍：不构成。** 第 2 条是一个几行就能修的回归，必须在合入前修；第 3 到 5 条是体验和成本问题，不影响正确性。从行为上看，候选整体比基线好：GitHub 题大幅改善；Feeder 题和 Maps 题略好或持平，轮数中位数分别是 14 对 15、22 对 25.5。

---

## 2. 确定缺陷

### 必须修

**F1. 标准无障碍动作编号被印进默认输出（本轮回归）**
- **触发条件。** 节点宣告了 ACCESSIBILITY_FOCUS（0x40）、CLEAR_ACCESSIBILITY_FOCUS（0x80）、按粒度移动（0x100、0x200）、HTML 元素导航（0x400、0x800）、SCROLL_TO_POSITION（0x1020037）等动作。Chrome 网页、Maps 和 Contacts 的几乎每个节点都是这种情况。
- **源码位置。**
  - `androidctl/src/androidctl/ui_view.py:106-145`：`compact_actions` 的第一个分支 `if identifier not in ACTION_NAMES:` 会把未命名的编号一律加成 `{"name": "action-40"}`。
  - `ui_view.py:54-68` 的 `DEFAULT_OMITTED_ACTIONS` 虽然列了这些编号，但省略判断在后面的 `elif` 里，永远轮不到。
  - `node_facts.py:42-80` 的 `ACTION_NAMES` 里没有 64、128、256、512、1024、2048。
  - `cab84b8` 的同一函数先判断是否省略，所以没有这个问题（`git show cab84b8:androidctl/src/androidctl/ui_view.py`）。引入这个问题的是 `65f6795 refactor(ui): simplify observation text and reference input`。
- **实跑。** 在 `venv6` 里执行 `compact_actions({'actions':[16,64,128,256,512,1024,2048,16908342],'clickable':True})`，返回 `[{'name':'action-40'},{'name':'action-80'},{'name':'action-100'},{'name':'action-200'},{'name':'action-400'},{'name':'action-800'}]`。主机测试里没有覆盖这种情况。
- **影响（原件、重算，`S/agents9/73/noise.txt`、`carry.py`）。** 下表的"约合 token"按 3.5 字节对 1 token 粗算，只能看量级，精确数字以负责重算的核查员为准。

  | 题目 | 占全部输出的比例 | 按"之后被重复读入的轮数"加权后的比例 | 每次运行约合 token |
  | --- | --- | --- | --- |
  | GitHub 网页题 | 32.7% | 31%～33% | 6.7 万～9 万 |
  | Maps 路线题 | 17.7% | 约 17% | 1.9 万～4.5 万 |
  | 后台屏 Contacts 题 | 17.2% | 19%～21% | 1 万～1.5 万 |
  | Feeder 长列表题 | 7.7% | 约 6% | 0.3 万～0.6 万 |

  GitHub 第 1 次那条 180 KB 的 `snapshot --cached --all` 输出里，`action-name=` 一项就占 49.7%。

  上一轮评审单独点名过 `context-click` 的噪声，第五轮第 13 题也裁决过"去掉网页节点上重复的 `context-click`"。现在换成了更多、更密的编号噪声。
- **最小修正。** 把省略判断提到"未命名"分支之前，或者把这些编号补进名字表。再补一条主机测试："普通视图里不出现这些编号"。`--all` 模式也应当省略它们，`--raw` 和 `--json` 保持完整。
- **确定程度。** 有原件，有源码，也实跑过。

**F2. `back` 回执的默认文本仍印出按键过程计数，与第五轮第 15 题的裁决不符**
- **原件示例**（后台屏第 1 次）：`back: done; ... keys={"plannedEvents":2,"attempted":2,"returned":2,"successfulPrefix":[0,1],"syncReturned":2,"lastReturned":{...,"downTime":255395,"eventTime":255404,...},"inFlight":null,...}`，每次约 600 字节。
- **源码。** `ui_output.py:463-476` 的 `input_facts` 对 `keys` 字段不做任何省略。
- **裁决原文**（`androidctl-r5-decisions.md` 第 15 题）："过程计数……从默认文本里去掉，在 `--json` 里保留"。
- **影响。** 每次运行 1 到 6 次 `back`，每次约 600 字节，量不大，但属于已裁决却没落实的项目。
- **修正。** 默认文本里成功时不印 `keys`，只在失败或未完成时打印。

### 可选（体验和成本，按"每次运行浪费的轮数乘以出现频率"排序）

**F3. 动作后自动观察取得太早，或者 `changed` 判早了，导致补查或用错观察**
- **原件统计**（`S/agents9/73` 下的分类脚本，候选 20 次）：
  - 成功的动作回执里，`change=unsettled` 占的比例分别是：GitHub 17/23、Maps 33/59、后台屏 22/61、Feeder 16/39。其中 56 条带有"Estimated next sample exceeds the observation deadline"。
  - "成功动作后紧接着补 fresh snapshot"共 35 次，按前一条回执的状态分：

    | 前一条回执的状态 | 次数 |
    | --- | --- |
    | `unsettled`、`partial` 或无树 | 25 |
    | 报 `changed`，但标题仍是旧页 | 3 |
    | Feeder 题为确认整卡可见而截图（题目要求） | 7 |
- **最典型的两例。**
  - GitHub 题 4/5 次点开 issue 链接：回执首行是 `click: done; change=changed at content,e…:bounds`，`screen title` 仍是 `Issues · python/cpython`（第 1 次第 8 轮，第 2 次第 5 轮，第 3 次第 8 轮，第 4 次第 5 轮）。
    - 第 2、3、4 次模型察觉到问题，补了一次 snapshot（各多 1 轮）。
    - 第 1 次同一轮接着执行 `find "May" --all`。按指引，回执不是"page may still be transitioning"时 `find` 复用当前样本，所以它读的是导航前的缓存，结果 0 命中。随后又补 snapshot、重新 `find`，多 2 轮。
  - Maps 题第 1 到 3 次：点出发地之后，回执报 `change=changed`，树里却还挂着 `text "Finding best route..."`，三个出行方式标签的名字都还没有时长。模型据此点击步行标签，结果被拒（见 F4）。第 4、5 次的同一步回执是 `unsettled`，模型先补 snapshot 或 `find`，就避开了这次拒绝。
- **影响。** 平均每次运行约 1.2 次补查；GitHub 题 4/5 次、Maps 题 3/5 次因此多出 1 到 2 轮。
- **最小修正方向**（实施方可定）：
  - 对 `click`、`back`、`fill --submit` 这类可能导航的动作，如果只看到 `bounds` 变化、标题没变，或者树里有明显的加载文字，不要标 `changed`，改标 `unsettled`；
  - 或者把"本次观察是否早于页面就绪"的判断交给一个统一的、偏保守的规则。

  指引不用改，现有的"首行写 `page may still be transitioning` 时 `find` 会刷新"这一句已经是对的。问题在于回执的标注不准。

**F4. 引用过期被拒时，回执不带任何可用的新事实**
- **触发条件。** 引用签名里的名字是非空的旧值，新名字也是非空但不同的值。规则在 `UiRefRepair.kt:97-99`：两个名字都非空且不相同时直接排除。
- **原件。** 三次被拒前后的情况如下（实施方在 `E/core-results.md` 末节"U35 三次旧 REF click"里也逐条核过）：

  | 运行 | 被拒的命令 | 被拒前的节点 | 补 snapshot 后的节点 |
  | --- | --- | --- | --- |
  | Maps 候选第 1 次 | `click e124387726` | `item e124387726 "Walking mode: "`，`roleDescription="Tab" rowIndex=0 columnIndex=2` | `item e124387857 "Walking mode: 18 minutes" detail="18 min"`，位置相同 |
  | Maps 候选第 2 次 | 同类 | 同上 | 同上 |
  | Maps 候选第 3 次 | 同类 | 同上 | 同上 |

  三次的拒绝回执都只有一行 141 字节：`click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action`。
- **与基线对比。** 基线 4/5 次撞上同一个拒绝。基线的拒绝回执长 4.5 KB，附带了动作前采到的新树，里面直接有 `Walking mode: 18 minutes` 的新引用，模型下一条命令就点中了（基线第 2 次第 7 到 8 轮）。候选在这一步要多 1 轮。
- **评价。**
  - 拒绝本身是对的：名字变了，不能静默替换。
  - 实施方的说法"旧基线同现此行为，未显示新增能力损失"只说对了一半。拒绝行为确实相同，但恢复成本多了一轮，这一点没有写出来。
- **最小修正**（实施方可定）：拒绝回执里附上修复时已经采到的那份观察；或者只列出"角色、行列位置都相同而名字不同"的候选引用，比如 `candidates: e124387857 "Walking mode: 18 minutes" (same row/column)`。是否放宽身份规则（例如允许"旧名是新名的前缀"），见第 5 节。

**F5. `snapshot --cached REF` 读屏外节点时返回空正文**
- **原件。** GitHub 题第 1、2、3、5 次都先执行了 `snapshot --cached e…`，输出只有 905 到 913 字节，全是头部，没有任何节点行；下一轮再加 `--all` 才读到评论（例如第 2 次第 9 到 10 轮）。
- **原因。** 默认视图省略 `offscreen` 节点，而 `find` 命中的评论节点都在屏外（`offscreenMatches=4`）。
- **影响。** 4/5 次 GitHub 运行各多 1 轮。上一轮评审把"`find` 加 `snapshot --cached REF`"列为读长页的首选路径，本轮模型也确实这样用了，但这条路的第二步正好在屏外节点上失效。
- **修正。** 显式给出 REF 时，包含它的屏外子树；或者在空结果的首行写"目标子树在屏外，加 `--all`"。

**F6. 方向词和"已到顶"的拒绝**
- **原件。** 候选共有 4 次被拒，另有不计分的历史运行 1 次：
  - Maps 第 3、5 次：在 `list … id="recycler_view" scroll="forward" rows=0 columns=0` 上写 `scroll … down`，被拒为 `notReady`，回执建议用 `forward`，模型下一轮改用 `forward` 成功；
  - Maps 第 1、2 次：在 `scroll="forward down"`（也就是已经到顶）的列表上写 `up`，同样被拒。
- **源码。** `UiScroll.kt:41-66`：只有 ScrollView、GridView 或 `columns == 1` 的节点才把 `down` 映射成 `forward`。
- **影响。** 每次 1 轮，平均每次运行 0.2 轮。回执里给出的可用方向和建议命令都对，恢复率 100%。
- **修正。** 对只宣告 `forward`/`backward`、而且节点高大于宽的容器，允许 `up`/`down` 映射；`up` 被拒而只剩 `forward` 时，首行直接写"已在开头"。

**F7. 后台屏上 `open` 的回执没有树**
- **原件。** 5/5 次 `open com.android.contacts` 都成功，回执约 2 KB，其中约 1.6 KB 是启动细节（`requestedIntent`、`handler`、`preflight`、`landing` 等 JSON）。末尾是 `snapshotError="One complete task window is required; no windows enumerated; run snapshot when the screen is available"`。
- **影响。** 4/5 次下一轮紧跟一次 `snapshot`，各多 1 轮。
- **修正。** `open` 落地之后再等一个采样周期取观察；启动细节只在失败时打印。

**F8. 其它可读性问题（单项量小，合起来值得做）**
- **事件段。** Maps 题快照里的事件行占全部输出的 14.3%。每条事件都印出 `receivedMs`、`eventTime`、`type`、`contentChangeTypes`、`windowChanges` 等字段，还有 80 秒前输入法的"normal keyboard"事件，每次快照都重复（`ui_view.py:549-568`）。真正有用的只有 toast 文字，例如后台屏题里的"R4 Local Contact deleted"，模型在最终报告里引用了它。
- **每条命令的头部。** 每条命令都有 `androidctl capture=… connection=… sequence=… finishedMs=… snapshotAgeMs=…`、`observation {json}`、`edge-windows …` 三行，再加约 170 字节的 `snapshotPath`，合计约占后台屏题输出的 22%、Feeder 题的 13%。第五轮第 13 题要求"溯源行改成短行；只有一个窗口时不打印窗口行"，现在只落实了一部分。
- **无用的说明短语。** `(original target unknown: Original target is not identifiable in the final observation)` 和 `content unknown: Actual roots differ or are unknown` 出现在约 80 条回执里。模型从没用过，后台屏题里基本是逐条略过。
- **`wait` 失败的措辞难懂。** 回执是 `wait: failed; Observation failed: Condition observation completed after the request cutoff`（GitHub 第 3 次第 4 轮），读不出"5 秒内没看到这段文字"这层意思。
- **点击屏外目标被拒时没有建议。** 回执只说 `click: not sent; … target is not visible`，没有建议用 `reveal` 或 `scroll-until`（GitHub 第 5 次第 6 轮）。模型为此读了 11 KB 的 `actions.md`，多 2 轮。

---

## 3. 设计与简化

### 3.1 开场：本轮最成功的改动

| | 候选（20 次） | 基线（15 次） |
| --- | --- | --- |
| 首页 | `SKILL.md` 6,615 B，20/20 次读完整 | `SKILL.md` 16,001 B，15/15 次 |
| 首页后的专题 | `actions.md` 11 KB 共 4 次（GitHub 第 5 次；Feeder 第 2、3、5 次）；`clipboard.md` 1 次（读后没用）；`displays.md` 加 `reference.md` 共 12.9 KB，后台屏 5/5 次（首页要求读） | GitHub 5/5 次读 `reference.md`，被宿主截断在 64 KB |
| `--help` | 0 | 0 |
| 第一次操作设备在第几轮 | 第 2 到 3 轮；后台屏第 3 到 4 轮 | 第 2 到 3 轮 |

说明：
- 第 1 轮报线程号是题目规定的，所以两边开场都至少要 2 轮。
- 模型在 Feeder 题读 `actions.md`，是为了弄清"怎样使整张卡片可见"。这是题目措辞引出来的，不是首页缺了什么。
- 结论：首页加专题的分层方式这一轮真正起作用了。第五轮第 12 题定的"只报告，不设门槛"做法是对的，**不要动**。

### 3.2 每次非零退出

候选 20 次共 10 次非零退出（`cli-index.json` 的 `childExit`）：Maps 7 次、GitHub 3 次、Feeder 0 次、后台屏 0 次。四个题组都没有出现 UNKNOWN、搬屏，也没有出现收尾保护被触发。

| 运行和轮次 | 命令 | 关键输出 | 分类 | 恢复方式 | 多花的轮数 |
| --- | --- | --- | --- | --- | --- |
| Maps 第 1、2、3 次第 7 轮 | `click e…`（步行标签） | `not sent; No current target matches the original ref signature; code=stale` | 引用过期。根因是前一条回执的 `changed` 判早了，见 F3、F4 | snapshot 后点新引用 | 2（基线 1） |
| Maps 第 1、2 次第 17 轮 | `scroll e… up` | `current scroll directions: forward, down; use androidctl scroll e… forward` | 方向词（已在顶端） | 放弃这一步，改用 `back` 或直接结束 | 1 |
| Maps 第 3、5 次 | `scroll e… down` | `current scroll directions: forward; use … forward` | 方向词（只宣告 `forward`） | 改用 `forward` | 1 |
| GitHub 第 3 次第 4 轮 | `wait --contains "rare race"` | `failed; Observation failed: Condition observation completed after the request cutoff; observationWarning="partial; missing=1"` | 超时加措辞难懂；真实原因是搜索结果里没有这段文字 | snapshot 后改搜索词 | 1 |
| GitHub 第 4 次第 9 轮 | `scroll-until e… down --max-steps 40 --contains "May 2023" --timeout 60s` | `After scrolling 30 steps, the next scroll was not sent … stop=routeUnavailable` | 模型的检索词写错：页面原文是"May 24, 2023"，不含子串"May 2023" | 改为向上 `scroll-until "May"` | 约 4（含宿主 10 秒挂起后的 3 次轮询） |
| GitHub 第 5 次第 6 轮 | `click e657930845`（屏外的标题链接） | `not sent; … target is not visible; code=notReady` | 平台拒绝，回执没给建议 | 读 `actions.md`，再用 `scroll-until` | 2 |

上一轮的入口陷阱还剩多少：

| 陷阱 | 上一轮 | 本轮候选 |
| --- | --- | --- |
| 时长不带单位 | 10/10 次被拒 | 0 次。13 次 `scroll-until` 都写了 `15s`、`30s` 或 `60s`。首页示例就是 `--timeout 15s`；主机仍然拒绝裸数字，但报错里加了 `for example 1s`【实跑】 |
| 引用带 `@` | 1 次 | 0 次。输出不再印 `@`，主机也会剥掉前缀（`ui_client.py:397`） |
| 键名大小写 | 2 次 | 0 次。没人用 `key`；主机已经改成先转小写（`input_keys.py:30`） |
| `open` 写法 | 2/5 次 | 0 次。5/5 次都写成 `open com.android.contacts` |
| 方向词 | Maps 1 次 | 2 次，另有 2 次是"已到顶"（见 F6） |

### 3.3 引用被判过期的全部个案

候选 20 次里只有上表那 3 次，全部来自 Maps 的步行标签。共同点如下：
- 节点一直存在；
- 名字从 `Walking mode: ` 变成 `Walking mode: 18 minutes`，同时多了 `detail="18 min"`；
- 行列位置 `rowIndex=0 columnIndex=2` 没变，引用号从 `e124387726` 变成 `e124387857`。

模型为此多做的事是"读一行拒绝，补一次 snapshot，再点新引用"，每次 2 轮。Feeder 题和后台屏题没有出现过期。GitHub 题没有出现过期，但第 1 次出现了"读到旧缓存"的情况，属于 F3。

### 3.4 多余调用：按原因分类

| 类别 | 次数（20 次合计） | 原因归属 |
| --- | --- | --- |
| 动作后在 `unsettled` 或无树回执之后补 snapshot | 25 | 工具输出不足（F3、F7），按指引补是对的 |
| 回执报 `changed` 但页面没变，补 snapshot | 3，另有 1 次旧缓存 `find` | 工具标注错误（F3） |
| 空的 REF 读取 | 4 | 工具输出不足（F5） |
| 读保存文件（`Select-String`） | 1 | GitHub 第 1 次 `snapshot --cached --all` 输出 180 KB，被宿主截断。其中一半是 F1 的噪声 |
| 动作后截图看图 | Feeder 7 次，Maps 2 次 | 题目要求"整卡可见"或要读页面完整地址，属正当 |
| 宿主轮询（`write_stdin`） | 3 | 宿主执行器 10 秒后挂起；起因是长 `scroll-until` 加错误的检索词 |
| GitHub "Jump to bottom" | 2 | 模型的习惯，为了确保评论加载完 |

这一轮没有看到"同一页反复 `find`"或者"无目的地反复 snapshot"。模型几乎每一次补查都有明确理由，而且这些理由多数指向工具输出。

### 3.5 默认输出的可读性：原件片段

原件都在 `E/<运行>/evidence/cli/*/child.stdout`。

**① 点击回执的头部**（Maps 第 1 次第 6 轮）。噪声出现在每次调用的头部：
```
click: done; change=changed at windows,content baselineAgeMs=9763 (original target unknown: Original target is not identifiable in the final observation); snapshotPath="ultragoal\\…\\000004-…-Maps.txt"
androidctl capture="f09d1def-…:10" connection=1 sequence=24 finishedMs=248638 snapshotAgeMs=47
observation {"scope":[{"window":22,"state":"full","package":"com.google.android.apps.maps"}],"target":{"displayId":0}}
screen app="com.google.android.apps.maps" title="Maps" keyboard="hidden"
window id=22 active focused content="full" selected
edge-windows count=1 content="notRead" windows=[{"id":18,"box":[0,0,720,48]}]
events state="available" source={"connection":1,"sequence":"receipt",…} range={…} retained={…} incomplete=false
event seq=142 receivedMs=247713 eventTime=247710 type=32 displayId=0 windowId=22 contentChangeTypes=16 windowChanges=0 kind="pane" … text=["Start location, …"]
```
评价：
- 调用者真正用到的只有首行的 `done` 和 `changed`、`screen title`，以及下面的树；
- `capture`、`observation`、`edge-windows` 三行和长路径从来没被用过；
- 同一条 pane 事件在三次连续调用里重复出现。

**② 网页评论节点（`--all` 读取，GitHub 第 2 次第 10 轮）。** 正文只有"gvanrossum commented on May 24, 2023"，其余都是噪声：
```
text e784477380 "gvanrossum commented on May 24, 2023" action-name="action-40" action-name="action-100" action-name="action-200" action-name="action-400" action-name="action-800" +show-on-screen action-name="context-click" roleDescription="heading 3" chromeRole="heading" heading offscreen
```

**③ Feeder 列表项。** 每行挂 8 个动作标签（第五轮第 13 题"自定义动作只写标签"的结果）。信息有用，但每个列表项都重复：
```
item e810640127 "RFC 10080: …" clickable +long-click action-name="action-40" action-label="Toggle read status" action-label="Open in reader" action-label="Open in custom tab" action-label="Open in default browser" action-label="Save article" action-label="Mark the above as read" action-label="Mark the below as read" action-label="Share" detail="R4 RFC list | Oct 30, 2026 | Unread" rowIndex=15 columnIndex=1
```

**④ 引用过期：候选与基线对照。** 候选只给一行，没有新事实；基线附带新树，新引用就在里面：
```
候选：click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action
基线：click: not sent; … code=staleRef; … snapshotPath=…  (随后附动作前的新树，含 <item ref="e705072772" name="Walking mode: 18 minutes" …/>)
```

**⑤ 方向拒绝。** 这是写得好的一例：直接给出可用方向和可以照抄的命令，模型 4/4 次一轮内恢复。
```
scroll: not sent; Original target is not ready for scroll-down: target has no advertised route for scroll-down; current scroll directions: forward; use androidctl scroll e726584882 forward; code=notReady; …
```

**⑥ `back` 回执里的按键遥测**（F2）：
```
back: done; Platform key dispatch returned; App outcome is not verified; change=unsettled (…No before observation for change comparison); keys={"plannedEvents":2,"attempted":2,"returned":2,"successfulPrefix":[0,1],…,"lastReturned":{"index":1,"keyCode":4,…,"downTime":255395,"eventTime":255404,…},…}
```

**⑦ 会话关闭和状态。** 这也是写得好的一例：模型直接据此报告"屏已销毁、无外来任务、资源为 0"。
```
session close: done; Session ended; UI cleanup confirmed
taskDisposition={"state": "clear", "displayId": 2, "destroyed": true, "openedTasks": [{…}], "movedTasks": [], "foreignTasks": [], "unknownTasks": [], …}
status: done … session=none callerSession=none ui=0 shell=0 file=0
```

**⑧ 空的 REF 读取**（F5）。只有头部，没有任何节点行，也没有说明原因：
```
snapshot: done; cached; sequence=35; …
androidctl capture=… snapshotCached=true …
observation {…}
screen app="com.chrome.beta" title="… Issue #86296 …"
window id=29 active focused content="full" selected
edge-windows count=2 …
```

**哪些事实调用者需要，却要另外去查：**
- 节点是不是完整可见。Feeder 题 5/5 次都靠截图加看图来确认；默认输出只有 `offscreen` 一个标记，没有"部分可见"。
- 引用过期之后，新的引用是什么（F4）。
- 页面是否还在加载（F3）。

### 3.6 用到的命令和参数分布

由 `S/agents9/73/stats.py` 统计，候选 20 次：

| 题组 | 命令次数 |
| --- | --- |
| GitHub | snapshot 28（其中 `--cached` 13、`--all` 9），find 10，fill 8（全部带 `--submit`），click 8，back 5，scroll-until 4，wait 1 |
| Feeder | click 23，snapshot 18（`--screenshot` 8），back 6，scroll-until 5，scroll 5 |
| 后台屏 | click 35，fill 10，snapshot 6，session open 5，open 5，back 5，session close 5，status 5，apps 4，scroll-until 4，find 4，scroll 2 |
| Maps | scroll 34，click 26，snapshot 19（`--screenshot` 2），fill 5，back 1，find 1 |

从未用过的命令有：`reveal`、`key`、`ime`、`type`、`select`、`check`、`uncheck`、`expand`、`collapse`、`long-click`、`action`、`range`、`locate`、`tap`、`swipe`、`drag`、`pinch`、`timeline`、`screenshot`（单独使用）、`clipboard`、`notifications`、`shell` 和文件命令。`--marks`、区域截图、`--boxes`、`--json`、`--raw` 也从未用过。

新能力在自由选择下的使用情况：

| 新能力 | 使用次数 | 结果 |
| --- | --- | --- |
| `scroll-until --contains` | 13 次（GitHub 2 次运行、Feeder 5/5、后台屏 4/5） | 12 次成功，1 次失败（模型检索词写错）。Feeder 和后台屏第一次调用就成功。但 Feeder 在文字刚出现时就停下，卡片只露出一部分，5/5 次还要再滚一下并截图确认。这是第五轮用户裁决"看到文字即成功"的直接后果，属于预期，不是缺陷 |
| `wait --contains` | 1 次 | 失败（措辞问题见 F8） |
| 图像标注、区域截图、坐标手势、剪贴板 | 0 次 | 普通任务用不上；基线 GitHub 题用了 10 次 `tap` 加 2 次 `swipe`，候选因为网页内容已经进了树而完全不需要 |
| 截图看图 | 候选 9 次，基线 GitHub 44 次 | 候选只在题目要求视觉确认时才用 |

### 3.7 结束阶段

- **会话关闭。**
  - 后台屏题 5/5 次 `session close` 后接 `status`，这是题目要求。
  - 其它三题都没有关闭会话，也是题目要求的（保留现场给独立核验）。
- **业务核验。**
  - GitHub 题回到筛选结果后补一次 snapshot 核对，或者改填 URL 回到结果页（第 3 次）。
  - Maps 题回到地点页核对。
  - Feeder 题回到列表核对。
  - 后台屏题从列表重开联系人，核对名字和备注之后才删除。
- **最终回答。** 都如实交代了没读到的地址范围，例如"终点仅核实到 Great Russell St"。两处小瑕疵：
  - Maps 第 5 次说某条命令"退出码 1"，真实的子进程退出码是 2。原因是模型自己的 PowerShell 包装没有传递退出码，不是产品问题。
  - GitHub 第 3、4 次没有提到自己那条失败的 `wait` 或 `scroll-until`，题目也没有要求列出。

  20 次里没有任何虚报。

---

## 4. 基线与候选的做法差异

每题挑一次最接近中位数的运行，并排列出。

**GitHub 网页题**（候选第 1 次 15 轮，基线第 1 次 27 轮）：

| 步骤 | 候选 | 基线 |
| --- | --- | --- |
| 开场 | 读 6.6 KB 首页，snapshot | 读 16 KB 首页，snapshot，再截图看图，读 64 KB `reference.md` |
| 搜索 | `fill --submit` 搜索，`find`（0 命中），`snapshot --cached --all`（180 KB），在保存文件里检索，改搜索词重搜 | 直接填 URL 搜索，截图看图，按坐标 `tap` 点开 issue |
| 进入 issue | 点 issue（回执是旧页），`find "May" --all`（读旧缓存，0 命中），snapshot，再 `find "May" --all`（4 命中） | 打开 Chrome 菜单，滚动，点"页内查找"，填 May，`back`，截图 |
| 读评论 | `snapshot --cached REF`（空），`--all REF`，`--all 父REF` | "下一个"按钮点 4 次，每次截图并看图 |
| 取评论链接 | 节点 `id="issuecomment-1560279122"` 直接给出 | 长按日期 |
| 收尾 | `back` 加 snapshot | `back` 两次，截图看图 |

候选省掉的是截图看图、坐标点击、页内查找和长按取链接，约 12 轮。候选多出来的是空的 REF 读取、旧缓存 `find` 和 `--all` 大输出，约 4 轮。

**Feeder 长列表题**（候选第 2 次 14 轮，基线第 2 次 15 轮）：

| 步骤 | 候选 | 基线 |
| --- | --- | --- |
| 找目标 | snapshot，`scroll-until --contains "RFC 10080"` | snapshot，`scroll down` 两次 |
| 确认整卡可见 | 读 `actions.md`，截图看图，再 `scroll down`，再截图看图 | 截图看图 |
| 打开和核实 | 点卡片，菜单，"Open in Web view"，snapshot | 点卡片，菜单，`back`，再点，snapshot |
| 收尾 | `back`，点 `Go back` | `back`，点 |

结论：`scroll-until` 省下了一次普通滚动，但"刚出现就停下"又补回了一次滚动和一次截图。两边基本打平。

**Maps 路线题**（候选第 2 次 22 轮，基线第 2 次 24 轮）：

两边路径几乎一样：点搜索框，填起点，点建议项，点步行标签（被拒），换用新引用，滚动读路线，点 Close，滚动读地址，回到顶部。差别在于：
- 候选在被拒后多补一次 snapshot（F4），在已到顶时多一次 `up` 被拒；
- 基线每条命令都要加 `--print`，被拒后少 1 轮，另外多截图看图 2 次。

总体持平或候选略好。

**后台屏 Contacts 题**：只有候选，典型 17 轮。读首页和 `displays.md`，`session open --display virtual`，`apps`，`open`，snapshot，新建，填姓名，`scroll-until Notes`，填备注，保存，`back`，重开核对，删除（点击 3 到 4 次），`session close`，`status`，最终回答。

**模型的确定性很高。** 基线第 3 与第 4 次的 22 条命令逐条同序，只有坐标和引用号不同；基线第 2 与第 5 次也几乎一样；候选 Maps 第 1 到 3 次前 10 轮逐条相同。所以"每 5 次中有几次"更像是"这个模型在这一步会不会犯"，不是随机波动。陷阱一旦出现就是系统性的，只看 5 次的样本不能把它当成偶发。

---

## 5. 后台屏 Contacts 五次：会话与显示

- 5/5 次的步骤都是：先按首页的要求读 `displays.md`，然后 `session open --display virtual`，回执是 `target={"kind": "virtual", "displayId": 2}`。没有一次搞混 main 和 virtual，没有一次 `takeover`，也没有一次 `busy`。
- 回执里的 `preflight` 和 `landing` 显示，任务 59 落在 `displayId 2`，没有冲突。5 次的 `movedTasks`、`foreignTasks`、`unknownTasks` 全部为空，所以"任务被系统搬到别的屏"这件事没有发生，搬屏归因机制在这批运行里没有被检验到。
- 收尾保护：5/5 次 `session close` 一次成功，回执是 `UI cleanup confirmed`、`destroyed: true`；随后 `status` 显示 `session=none ui=0 shell=0 file=0`。最终回答都准确引用了这些事实。保护从来没有被触发，所以"关闭时有外来任务、拒绝销毁"这条路径没有接受过真实任务的检验。
- 唯一的摩擦是 F7：`open` 之后没有树。
- **结论：这一块对调用者很友好，不要动。**

---

## 6. 前五个最值得改的问题

按"每次运行平均浪费的轮数或 token，乘以出现频率"排序：

| 排名 | 问题 | 浪费量 | 最小改法 | 谁来定 |
| --- | --- | --- | --- | --- |
| 1 | F1 标准动作编号噪声（回归） | 20/20 次，每一轮都在；GitHub 约占加权输入的 3 成（每次运行约 7 万到 9 万 token），Maps 和后台屏约 2 成 | `compact_actions` 先做省略判断，补一条测试；`--all` 也省略 | 实施方，属于回归修复 |
| 2 | F3 动作后观察取早了，或 `changed` 判早了 | 每次运行约 1.2 次补查；GitHub 4/5 次、Maps 3/5 次多 1 到 2 轮 | 导航类动作在标题没变、只有 `bounds` 变化或有加载文字时标 `unsettled`；`open` 落地后再取一次（F7） | 实施方 |
| 3 | F4 引用过期回执没有新事实 | Maps 3/5 次各多 1 轮 | 拒绝时附上修复观察，或附上同角色、同行列的候选引用 | 实施方；放宽身份规则需要用户定（见第 8 节） |
| 4 | F5 REF 读屏外节点为空 | GitHub 4/5 次各多 1 轮 | 显式给出 REF 时包含屏外子树，或者给出提示 | 实施方 |
| 5 | F8 头部、事件、遥测的冗余（含 F2 的 `back` 遥测） | 每轮都有；Maps 的事件行约占 15%，后台屏的头部约占 22% | 头部压成一行；事件段只印 toast 和窗口标题变化的文字，并去重；成功时不印 `keys`；`open` 成功时不印启动细节 | F2 是落实已有裁决，实施方定；事件段和头部的进一步压缩会改默认文本契约，第五轮第 13 题写的是"只做这几项"，所以需要用户点头 |

次一级的问题有：方向词（F6，每次运行 0.2 轮）；`wait` 失败措辞和点击屏外目标时缺少建议（F8，各 1 次）。

**做得好、不要动的地方：**
1. **首页 6.6 KB 加专题的分层。** 本轮 0 次读 `--help`，专题只读了 4/20 次，开场 2 到 3 轮。这是相对上一轮最大的进步。
2. **动作默认附带观察，引用不带符号，入口宽容**（剥掉 `@`、键名不分大小写、`open PACKAGE`、首页示例写好时长单位）。入口陷阱 0 次。基线每条命令都得手写 `--print`。
3. **网页内容进入树，`find` 能搜到屏外内容，节点直接带 `targetUrl`。** GitHub 题从 27 轮降到 15 轮，全靠这一项；评论永久链接直接就在 `id` 里。
4. **"未发送"类拒绝的写法。** `not sent`、`dispatchStarted=false`，再加上可用方向和可以照抄的命令，10 次非零退出全部在 1 到 2 轮内恢复，没有一次误判成"已执行"。
5. **后台屏会话的打开、关闭和状态回执。** 足以让模型自主报告释放结果，不需要读别的东西。

---

## 7. 证据审计

| 实施方的声称 | 核对结果 |
| --- | --- |
| 四题候选 5/5 成功；GitHub 轮数中位数 15 对 27；Feeder 14 对 15；Maps 22 对 25.5（基线 4/5）（`E/core-results.md`"正常收尾与本轮固定任务结果"一节） | **成立**。轮数与我从 `token_usage_record` 数出来的一致（`S/agents9/73/tl/_summary.tsv`）。业务成败我没有逐次独立核对，只核对了最终回答与轨迹是否一致：没有发现虚报。 |
| 指导交付：GitHub 候选只有第 5 次读了 `actions`；Maps 候选和基线读专题 0 次；`--help` 0 次 | **成立**（第 3.1 节）。Feeder 3 次读 `actions` 和 1 次读 `clipboard` 的情况我没有在该页找到对应表格，未能核对它是否报告了。 |
| U35 三次旧引用点击："步行 Tab 的可观察名称从无时长变成含 18 minutes""旧基线同现此行为，目前未显示新增能力损失""补查成本不变" | **部分成立**。名字变化和"节点仍在原行列"成立。但它自己的表格写明基线"拒绝本身已交付 fresh 树……没有另发 snapshot"，候选却要另发；这正是一轮的恢复成本差，它没有被认定为退步（见 F4）。 |
| 失败、UNKNOWN、partial 照记 | **成立**。10 次非零退出、14 次以上 partial 观察和 3 次宿主轮询都在原件里；没有 UNKNOWN。 |
| 第五轮第 13、15 题已落实（默认输出精简） | **部分成立**。`context-click` 已去掉，自定义动作已改成只写标签，路径已改成相对路径。但同时引入了 F1 的回归；`back` 的过程计数仍在默认文本里（F2）；溯源行仍有三行（F8）。 |
| 版本匹配 | **成立**。指引正文与 `fb75afc` 逐字一致；主机和 APK 来自 `a27fe5aa` 的物料目录（这一点依据的是实施方的记录加上路径，我没有重新计算哈希）。 |

另外两点：
- 测试全过，并不能说明默认输出符合设计。主机端没有任何测试覆盖"标准动作编号不进入默认文本"。
- 实施方的成本页只记轮数和总输入，没有把"多出来的轮数从哪里来"拆开。本报告第 2、3 节的分类可以直接作为修复清单。

---

## 8. 需要用户裁决的新取舍

**问题 1：标签或按钮名字"补长"时（旧名字是新名字的前缀），是否允许自动沿用原引用？**
- A：维持现状，拒绝。实施方在拒绝回执里附上同角色、同行列的候选引用（F4 的修法）。
- B：当旧名字非空、是新名字的前缀，并且角色、行列、包名都相同时，视为同一节点继续执行，回执里注明名字变了。
- C：维持现状，回执也不附任何东西。

推荐 **A**。它只花一轮里的几十字节，就把恢复成本降到与基线一样，而且不放宽保护。B 能再省一轮，但"标签补上时长"和"换成另一个标签"在通用规则里很难稳妥区分。B 只有在 A 落实之后，过期仍然频繁出现时才值得再议。

**问题 2：默认文本是否在第五轮第 13 题之外继续压缩？** 压缩的对象是：头部三行合成一行；事件段只留 toast 和标题文字；`open` 和 `back` 成功时的诊断细节只在 `--json` 中出现。
- A：做。`--json`、`--raw` 和保存文件保持完整。
- B：不做，只修 F1、F2。

推荐 **A**。按原件估算，后台屏题能再省 2 成多，Maps 题再省 1 成多，没有任何能力损失：在这 20 次里，这些字段一次也没被调用者用过，唯一被用到的 toast 文字会保留下来。

剩余的未知：真机和厂商 ROM 上事件的种类可能更多，"只留 toast"是否够用，需要设备才能确认。

其余各条（F1 到 F8 的修法）都属于实施细节。

---

## 9. 有判别价值的补验

1. **修完 F1 和 F3 后，只重跑 GitHub 候选 3 次。**
   - 如果中位数总输入下降超过 25%、轮数下降 1 到 2 轮，说明默认输出加回执时机是当前最主要的剩余成本，就应该接着做第 8 节问题 2 的 A。
   - 如果下降不到 10%，说明剩余成本主要在模型策略，输出不必再压缩。
2. **修完 F3、F4 后，重跑 Maps 候选 3 次。**
   - 如果过期从 3/5 降到 0，或者过期后 1 轮就恢复，说明问题 1 选 A 已经足够，不需要放宽身份规则。
   - 如果仍然频繁过期并且要多轮恢复，再议 B。
3. **用 `fb75afc` 的主机，对 `E/r5-c2-final-candidate-01/evidence/cli` 里那条 180 KB 的 `--all` 原件做一次离线重投影。**
   - 原件里只有渲染后的文本，需要先确认是否保存了 raw，这一点我没有核实。
   - 修复前后的字节对比可以直接给出 F1 的收益，不需要设备。

---

## 10. 未能核实的内容

- 我没有独立核对每次运行的业务成败，只核对了最终回答与轨迹事实是否一致。
- 任务正文在轨迹里是加密的，我是用 `ST/efficiency-preparation.md` 里冻结的提示文本对照的，假定实际执行时用的就是这份提示。
- 轨迹里的图像被替换成了占位，无法判断 Feeder 和 Maps 的截图是否显示了模型所说的内容。
- token 数字用"字节除以 3.5"粗算，只能看量级。精确重算另有核查员负责。
- F3 到 F7 的修法都没有在设备上验证；F6 里"高大于宽即可映射"在横向列表或嵌套滚动容器里是否会误判，需要设备才能确认。
- 不计分的两次运行只做了统计，没有逐轮精读：
  - Maps 历史运行 30 轮，其中 1 次方向拒绝；
  - GitHub 旧候选 10 轮，其中 1 次 `scroll-until` 因"多个目标"失败，那是第五轮用户裁决"看到文字即成功"之前的语义。
