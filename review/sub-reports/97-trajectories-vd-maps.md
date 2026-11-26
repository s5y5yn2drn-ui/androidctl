# 第七轮专题核查：逐条阅读后台屏题与路线题的真实任务轨迹

核查对象：候选 `f401deb` 的 10 次真实任务原件（后台屏 Contacts 题 5 次、Google Maps 路线题 5 次），对照上一轮候选 `fb75afc` 的同题 10 次原件。全程只读，没有设备。解析脚本在 `S/agents11/97/tools/`（`steps.py`、`compact.py`、`table.py`、`stats.py`、`noise.py`、`render_fail.py`），都用 `python3 -I` 运行；中间产物在 `S/agents11/97/dump/`。

## 摘要

1. **路线题变贵，全部花在"读完路段之后"那一段。** 按阶段拆分：建立步行路线这一段，本轮反而比上一轮**少 2 个回合**（中位数 8 → 6）；读前三段持平（2 → 2）；读完路段之后，核对两端地址并回到 Museum 页这一段，从中位数 9 个回合涨到 16 个（上一轮 4–10，本轮 13–20，本轮 5 次全部高于上一轮 5 次中的最大值）。每回合新增的输入下降了约 21%（约 1,430 → 1,120 token），所以总成本几乎完全由回合数决定。
2. **涨出来的回合主要不是产品造成的。** 有几种解释找不到证据支持：可见性改动（第 13 条）、旧引用判定变严、瘦身删了有用信息（第 15 条）。滚动方向映射还省了约 0.4 回合。也不是纯随机。可以确认的系统性原因有两条：(i) 第六轮评审建议、经附录一落实的首页示例改动（`key enter` 改成 `fill --submit`）。改动后 5/5 次都直接提交，Maps 不再显示带地址的起点建议列表；上一轮 5/5 次是从这个列表里顺手读到起点地址"Trafalgar Square, London, UK"的。本轮有 2 次为此另外绕路找起点地址，共多花约 10 个回合。(ii) 模型找 Museum 完整地址时更执着：多次滚动、截图看图 7 次（上一轮 2 次）、翻"About"标签，平均每次多花约 5 个回合。地址行在 Overview 页的门票和评论下面。题目"核两端地址"没有写明读到什么程度算完成，这是这道题成本波动的最大来源。
3. **第 10 条"名字已变"的那一行在它最初针对的场景里注定出不来**（源码推断）。步行标签本身就是列表项，它的行标签取自它自己的子文字或它自己的名字，名字变了，行标签也跟着变。严格判定要求行标签相同，所以永远命不中。本轮 1/1 次、上一轮 3/3 次同类拒绝都会这样。要求"行标签不能省"的是第六轮评审方（我们）和用户裁决，这一条确实没有达到裁决的目的。建议进下一轮。
4. **第 5 次那条 `scroll` 退出码 1，并不是"观察超时"**：命令总共只用了 1.8 秒，结果是平台拒绝执行这次滚动（源码推断）。回执首行没有写失败原因，只有一句"预计下一次采样超过观察时限"，实施方自己的成本页也因此读错了。我在主机端用同形数据复现了这一行。建议进下一轮修。
5. **后台屏题：5 次、93 条命令全部退出码 0，流程干净。** 回合数持平（中位数 17），单条输出平均缩小约 45%，总输入中位数从 454,100 降到 425,366（-6%）。`open` 的回执现在可以附带界面树，但在后台屏上 3/5 次因"找不到窗口"没有附上，需要补一次 snapshot。打开、关闭和状态的回执都写明了显示和任务去向，模型据此正确报告了释放结果。
6. 这些问题**都不挡合入**。最短合理路径：路线题约 13–15 回合（现中位数 26），后台屏题约 13 回合（现中位数 17）。改进清单见第 8 节。

---

## 0. 口径与方法

- **数据**：`ST/efficiency/r6-c1-candidate-01..05`、`r6-v1-candidate-01..05`（本轮），`S/pkg9/.../efficiency/u35-c1-candidate-01..05`、`r5-v1-candidate-01..05`（上一轮候选）。两轮的执行框架都是 Codex 0.171.0，模型都是 `gpt-6.1-sol`、medium（从 `session_meta` 和 `turn_context` 读出）。
- **回合**：数 `token_usage_record`。一个回合里的工具调用是一段 JavaScript，可以连发多条命令。命令按 `call_id` 归到发出它的回合（`tools/compact.py`）。我早先按事件顺序归属的统计（`stats.py` 里的 multiCmdTurns）在旧记录上会错位，已经弃用。
- **题目原文**：在原生记录里是加密的。题目定义取自上一轮包里的 `real-tasks.md` 第 43 行："从 British Museum 地点页查询 National Gallery, Trafalgar Square, London 到这里的步行路线；**核两端地址**、当次距离／时长、前三条路段说明，回 British Museum 地点页。"实施方说本轮"沿用原题"，我没法独立核对本轮的提示原文（见"未能核实"）。
- 回合数和总输入都与实施方 `core-results.md` 的表一致：本轮 24/27/26/25/30 回合，731,760/829,986/819,607/776,265/1,010,956 输入。

---

## 1. 每次运行的步骤表

完整的逐条表（回合、本回合输入、原样命令、退出码、输出字节、输出首行）放在附录 A（路线题）和附录 B（后台屏题），由 `tools/table.py` 从原生记录直接生成。"模型的反应"就是表里的下一行。这里只写每次运行的要点。

### 1.1 路线题（本轮）

阶段划分：A 开场（报线程号、读首页）；B 从第一次 snapshot 到步行路线的详情可读；C 读前三段；D 核对地址、回到 Museum 页；最后一回合是最终回答。

| 运行 | A | B | C | D | 总回合 | 非零退出 | 截图看图 | 读过的指引 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 第 1 次 | 1 | 7（T2–T8） | 2 | 13（T11–T23） | 24 | T5 旧引用 | 2 | 只有 SKILL.md |
| 第 2 次 | 1 | 7（T2–T8） | 2 | 16（T11–T26） | 27 | 无 | 1 | 只有 SKILL.md |
| 第 3 次 | 1 | 6（T2–T7） | 2 | 16（T10–T25） | 26 | T13 方向不可用 | 2 | SKILL.md，另有 `collapse --help` |
| 第 4 次 | 2 | 6（T3–T8） | 2 | 14（T11–T24） | 25 | T6 旧引用；T23 方向不可用 | 1 | 只有 SKILL.md |
| 第 5 次 | 2 | 5（T3–T7） | 2 | 20（T10–T29） | 30 | T17 scroll 失败；T24 scroll-until 失败 | 1 | 只有 SKILL.md |

**第 1 次。**
- T3 点 Directions，回执是 unsettled、partial。
- T4 `fill … --submit`，回执是 `change=loading at windows`，树里步行标签名为 `"Walking mode: "`，同时显示"Finding best route..."。
- T5 模型直接点这个标签，被拒为旧引用。T6 补 snapshot，T7 点新引用 `e608125823`（`"Walking mode: 18 minutes"`）。T7 的回执里出现 `No primary accessibility node reply…`，所以 T8 又补了一次 snapshot。
- T9–T10 滚动读完三段。
- T11 点 Close 回到 Museum 页。T12–T19 为了找 Museum 地址，滚动 4 次、截图看图 2 次。T20 把标签条往右滚，T21 点"About"，T22 滚回左边，T23 点 Overview 并滚回顶部。
- 最终回答写的是"未取得两端包含邮编的完整地址"。D 段的 13 回合里，有 10 回合花在找地址上，而且没有找到。

**第 2 次。**
- T3 点 Directions，回执没有树（721 字节，partial），T4 补 snapshot。T5 `fill --submit` 的回执同样没有树，T6 补 snapshot。T7 点步行，回执是 loading，T8 补 snapshot。T9 点路线卡片，T10 滚动读完三段。
- 从 T11 起找地址：T11 滚回顶部，T12 点路线第 1 步"The National Gallery"，想看起点地址，结果进了 Route preview。T13、T14 连按两次 back，回到 Museum 页。T15–T20 滚动、截图，没有找到地址。
- T21 再按 back，**离开了地点页**，回到 Maps 首页。T22 点搜索框，历史记录里出现了 Museum 的完整地址。T23 填 National Gallery，建议列表给出了起点地址。T24 填 British Museum，T25 点建议，回到地点页。T26 snapshot 核对。

**第 3 次。**
- B 段：T4 `fill --submit` 回执是 loading，`No primary … reply`，T5 补 snapshot。T6 点步行，回执是 loading，T7 补 snapshot。
- D 段：T10 点 Close。T11–T12 滚外层，T13 再往下滚时被拒（已到底）。接着 T14 snapshot，T15–T16 截图看图，T17–T18 把内层列表往下滚两次，T19–T20 再截图看图，T21–T22 再往下滚两次，仍然没有找到 Address 行。
- T23 滚回顶部，T24 读 `collapse --help`，T25 `collapse` 把底部面板收回原状。

**第 4 次。**
- T5 `fill --submit` 的回执里，树还是提交前的建议列表。T6 点了其中的建议项 `e404859888`，被拒为旧引用（见第 3 节）。T7 补 snapshot，T8 点步行。
- D 段与第 1 次几乎逐条相同：Close、滚动两次、截图、再滚两次、翻到 About、回到 Overview。T22 滚回顶部，T23 再向上滚时被拒（已在顶部），T24 snapshot 核对。

**第 5 次。**
- B 段最短，只有 5 回合。
- D 段最长：
  - T10 点路线第 12 步"The British Museum"，进了路线预览，T11 back，T12 Close。
  - T13–T16 滚动并截图。T17 滚的是外层底部面板，退出码 1（见第 3 节），T18–T19 改滚内层列表。
  - T20 `scroll-until … --contains 'Great Russell'`，一步就读到 `Address: Great Russell St, London WC1B 3DG, United Kingdom`。
  - T21 back，离开地点页。T22–T23 从搜索历史进入 National Gallery 页。T24 对外层容器 `scroll-until`，滚了 2 步之后第 3 步没有方向可滚，退出码 1。T25 改用内层列表，一步就找到起点地址。
  - T26–T28 back，再从搜索历史回到 Museum 页。T29 snapshot 核对。
- 这是唯一一次两端完整地址都读到的运行。

### 1.2 路线题（上一轮候选，对照）

| 运行 | A | B | C | D | 总回合 | 截图 |
| --- | --- | --- | --- | --- | --- | --- |
| 第 1 次 | 2 | 8 | 2 | 6 | 19 | 0 |
| 第 2 次 | 2 | 8 | 2 | 9 | 22 | 0 |
| 第 3 次 | 2 | 8 | 2 | 10 | 23 | 1 |
| 第 4 次 | 1 | 7 | 2 | 4 | 15 | 0 |
| 第 5 次 | 2 | 7 | 2 | 10 | 22 | 1 |

上一轮的 B 段是"填起点（不提交）→ 点建议项 → 点步行（第 1–3 次被拒为旧引用）→ snapshot → 再点步行 → snapshot"。D 段典型的走法是"Close → 滚两次 → 滚回顶部（已到顶被拒）"。终点地址都接受 Directions 按钮上的"Great Russell St"；起点地址都来自填写时的建议列表。

### 1.3 后台屏题（本轮）

| 运行 | 回合 | 命令 | 读的指引 | `open` 带树了吗 | 关闭前 status | 关闭后 status |
| --- | --- | --- | --- | --- | --- | --- |
| 第 1 次 | 17 | 18 | SKILL、displays、reference | 带了（直接点"Create new contact"） | 有 | 有 |
| 第 2 次 | 19 | 20 | SKILL、displays | 没带（`scopeUnknown`），补 snapshot | 有 | 有 |
| 第 3 次 | 17 | 19 | SKILL、displays、reference | 带了 | `--json status` | `--json status` |
| 第 4 次 | 17 | 18 | SKILL、displays、reference | 没带，补 snapshot | `--json status` | `--json status` |
| 第 5 次 | 16 | 18 | SKILL、displays、reference | 没带，补 snapshot | 有 | 有 |

5 次走的都是同一条路，各次之间只差两三条命令：
1. 首页、`displays.md`（4 次同时读了 `reference.md`）。
2. `session open --display virtual` 加 `apps Contacts`。
3. `open com.android.contacts`。
4. 点"Create new contact"。
5. 填名字，同一回合点"More fields"。
6. 有 3 次先 `find Notes`，5 次都用 `scroll-until … --contains Notes`。
7. 填备注，同一回合点 SAVE。回执里直接带有已保存的名字和 Note。
8. `back` 回到列表，点开联系人核对。
9. 删除：More options → Delete → DELETE。
10. `session close`，再 `status`。

上一轮第 2 次的 22 回合，是因为把可以合在一回合的命令拆开发了。本轮没有出现这种情况。

---

## 2. 路线题为什么变贵

### 2.1 回合差全部来自 D 段

| 阶段 | 上一轮回合 | 本轮回合 | 中位数差 |
| --- | --- | --- | --- |
| A 开场 | 2,2,2,1,2 | 1,1,1,2,2 | -1 |
| B 建立步行路线 | 8,8,8,7,7 | 7,7,6,6,5 | **-2** |
| C 读前三段 | 2,2,2,2,2 | 2,2,2,2,2 | 0 |
| D 核对地址并回到原页 | 6,9,10,4,10 | 13,16,16,14,20 | **+7** |
| 合计（含最终回答） | 19,22,23,15,22 | 24,27,26,25,30 | +4 |

把 D 段再细分（回合数，按第 1.1、1.2 节逐条归类）：

| D 段里的类别 | 上一轮 | 本轮 | 平均差 |
| --- | --- | --- | --- |
| 找 Museum 地址：滚动、截图、看图、翻标签 | 2,2,7,2,7（平均 4.0） | 10,6,12,8,8（平均 8.8） | +4.8 |
| 找起点地址：点路线步骤、重新搜索、进 National Gallery 页 | 0 | 0,5,0,0,5（平均 2.0） | +2.0 |
| 回到原页并核对：滚回顶部、collapse、误离地点页后重进、最后的 snapshot | 3,6,2,1,2（平均 2.8） | 2,6,3,5,4（平均 4.0） | +1.2 |
| Close 回到地点页 | 1 | 1 | 0 |

**每回合的输出变小了。**
- 每回合新增的输入（最后一回合输入减去第一回合输入，再除以回合数减一）：上一轮 1,350/1,297/1,426/1,612/1,464，本轮 1,283/1,034/1,191/1,115/1,057，**平均下降约 21%**。
- CLI 原始输出：上一轮平均每条 4.5 KB，本轮 3.3 KB。5 次合计的字节数几乎持平（约 387 KB 对 374 KB），因为本轮多发了约 20% 的命令。
- 用"每回合固定前缀约 17.2K token，加上逐回合累积的输出"粗算：如果本轮回合数与上一轮一样是 22，总输入约 63 万，**比上一轮中位数还低约 5%**（估算）。

### 2.2 逐一检验六种解释

**(a) 第 13 条的可见性改动让以前能直接点的节点现在被拒：不成立。**
- 6 次非零退出里没有一次与可见性有关。原因码分别是 `stale` 两次、`routeUnavailable` 两次、平台拒绝一次、`scroll-until` 方向用尽一次，回执里也没有出现"not visible"。
- Museum 地点页的树形状两轮相同：Overview、Tickets、Reviews、Photos、Directory 五个标签可见，另有一个第 7 列的"About"标签在屏外；Overview 列表的前几项都是没有名字的 `item … clickable`。
- 本轮看不到 `offscreen` 标记，是因为 Maps 的 RecyclerView 本来就不创建屏外的行，和第 13 条无关。
- 证据：`dump/*.full.txt` 里的统计（`not visible` 0 次），以及两轮同位置输出的对照（上一轮第 1 次 T14，本轮第 1 次 T12）。

**(b) 引用过期判定变严，或页面签名因可见位变化更容易变：不成立。**
- 旧引用被拒，本轮 2 次（第 1、4 次），上一轮 3 次（第 1–3 次）。
- 第 1 次和上一轮那 3 次是同一种情况：步行标签名字从 `"Walking mode: "` 变成 `"Walking mode: 18 minutes"`。
- 第 4 次是新出现的情况，由 `--submit` 引起：点的是提交前的建议项，见第 3 节。
- 没有发现因可见位变化而被拒的个案。

**(c) 第 15 条瘦身删掉了模型以前靠它少走一步的信息：不成立。**
- 删掉的内容有：`baselineAgeMs`、"original target unknown…"之类的诊断短语、事件行、`connection/sequence/finishedMs`、窗口几何，以及以前每个节点上挂的 `action-name="action-40"`（上一轮第 1 次一份输出里有 517 处）。
- 逐条读了上一轮的 D 段，模型从来没有用这些信息做过决定。本轮也没有一次补查是因为缺了这些字段。
- 瘦身的实际效果是每回合的输入减少了约 21%（见上一节）。

**(d) `open`、滚动、滚动查找的行为变化：小幅有利。**
- 上一轮第 3、5 次在"只公布向前方向"的竖向列表上写 `down` 被拒，各多花 1 回合。本轮的方向映射把 `down` 映成了 `scroll-forward`：例如第 1 次 T9 的回执里有 `Selected: … "action":"scroll-forward"`，本轮没有再出现这种拒绝。
- 本轮的两次 `routeUnavailable`（第 3 次 T13、第 4 次 T23）都是真的滚到了尽头，不是方向词的问题。
- `scroll-until` 只有第 5 次用了，3 次里成功 2 次，每次都是一步读到地址。
- `open` 的改动只影响后台屏题。

**(e) Maps 当天的内容或网络不同：部分成立，影响有限。**
- 路线本身完全相同：0.8 mi、18 分钟，Irving St 160 yd、Charing Cross Rd 0.2 mi、Shaftesbury Ave 0.3 mi。
- 有三处不同：
  - 时段。上一轮在 11:40–14:18Z 运行，博物馆"Open · Closes 5 PM"；本轮在 17:36–18:00Z 和 03:50–04:00Z 运行，显示"Closed · Opens 10 AM"。
  - Overview 列表在评论之后的内容。第 5 次是"More info → About → Address"，第 3 次是"See all reviews → Rate and review → 30 must-visit…"。第 3 次往下滚了 4 次都没有到地址行，可能和这个顺序不同有关（推断）。
  - 第 3 次开始之前出现过网络错误页，操作者在模型启动前已经恢复，不计入成本。
- 这些不同解释不了 D 段整体多出的 7 回合。

**(f) 模型随机波动：不能单独解释。**
- 本轮 5 次的 D 段（13–20）全部高于上一轮 5 次的最大值（10）。双侧秩和检验 U=0，p≈0.008（5 对 5 的精确值，估算）。
- 这个模型的确定性又很高：本轮第 1、4 次的 D 段几乎逐条相同（Close、滚两次、截图、再滚两次、翻 About、回 Overview）；第 1–3 次的 B 段也几乎相同。
- 所以 D 段变长，应当理解为"在本轮的条件下，模型稳定地多做了这些事"。

**(g) 实施方没有列出的系统性原因：首页示例从 `key enter` 改成 `fill --submit`。证据充分。**
- 改动在 `skills/androidctl/SKILL.md`：`git diff fb75afc f401deb` 里，`fill FIELD_REF "New value"` 加 `key enter FIELD_REF` 两行被换成 `fill FIELD_REF "New value" --submit`。
- 它来自第六轮评审报告附录一的建议（`androidctl-r6-candidate-review-fb75afc.md:547`，理由是"候选模型 8 次提交全部选了 `fill --submit`"，那 8 次都是网页题），用户在 `androidctl-r6-decisions.md:540` 把它列入逐条处理。
- 结果：
  - 本轮 5/5 次都用了 `fill … --submit`，上一轮 0/5。
  - Maps 收到提交后，直接采用第一个结果，开始计算驾车路线，**不再显示建议列表**。
  - 上一轮 5/5 次的输出里都有 `item … "National Gallery" … detail="Trafalgar Square, London, UK"`，起点地址是顺手得到的。本轮只有第 4 次碰巧看到（那份回执的树还是提交前的样子），第 2 次是事后重新搜索才看到。第 1、3 次的最终回答里没有起点地址，第 5 次专门进了 National Gallery 页去读。
  - 统计：`dump/*.full.txt` 里 `detail="Trafalgar Square, London, UK"` 出现的次数，上一轮 1,2,1,1,1，本轮 0,1,0,1,0。
- 它的好处是 B 段省了"点建议项"这一步：B 段的中位数从 8 降到 6，有一半是它的功劳。
- 净效果：B 段省 1 回合，起点地址的绕路平均多 2 回合，大致相抵，略亏。
- 这是第六轮评审方建议的一个副作用：建议的依据只来自网页题，没有在 Maps 这类"建议列表里本身就带有任务所需事实"的页面上检验过。

### 2.3 结论

- 路线题回升的直接原因是 D 段多了约 7 个回合。其中约 5 个回合是模型找 Museum 完整地址时更执着（滚动、截图、翻 About），约 2 个回合是因为 `--submit` 不再展示建议列表、起点地址要另找，约 1 个回合是误离地点页之后重新进入。
- 产品在每一步上的表现没有变差：建立路线少了 2 回合，单回合输出少了 21%。
- 题目"核两端地址"没有说明读到什么程度算完成，模型的完成标准每次不同。上一轮多接受了"Great Russell St"这种街名级别的地址，本轮有 3 次继续往下找。这是本题成本波动的最大来源，也是实施方只拿旧基线比较、没有解释回升的地方。
- **建议**：成本页要和上一轮候选比较，按阶段拆开写；把题目里的地址要求写清楚（例如"以地点页或建议列表上显示的为准，不要求邮编"），或者把 D 段单列统计。
- **分类**：性质是证据不足／设计取舍（评测口径）；去向是不挡合入，进下一轮（只改成本页和题目说明，不改产品）。

---

## 3. 6 次非零退出逐个剖开

### 3.1 第 1 次 T5：`click e608125688`，退出码 2

**上一份回执**（T4 `fill --submit`，节选）：
```
fill: done; change=loading at windows (page may still be transitioning); fill=done, verification="matches", actualValue="National Gallery, Trafalgar Square, London"; submit=done; snapshotPath=…
   list rows=1 columns=3 selectionMode=1
    item roleDescription="Tab" selected rowIndex=0 columnIndex=0
     text "Driving mode: " selected
    item e608125683 "Transit mode: " clickable roleDescription="Tab" rowIndex=0 columnIndex=1
    item e608125688 "Walking mode: " clickable roleDescription="Tab" rowIndex=0 columnIndex=2
   text "Finding best route..."
```
**本次回执**：
```
click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action
```

- **发命令前能不能看出引用已旧？** 能。首行写着 `loading`，树里有"Finding best route..."，三个标签的时长都是空的。首页也写了"Collect a fresh `snapshot` for a transition"。模型还是直接点了。
- **两份样本之间页面发生了什么？** 对比 T6 snapshot：同一个位置（第 0 行第 2 列）的标签变成了 `e608125823 "Walking mode: 18 minutes" detail="18 min"`。
- **拒绝对不对？** 拒绝是安全的，但其实不必要：如果按原位置派发，点的就是同一个步行标签，结果是对的。
- **回执给的下一步能照做吗？** 回执没有给下一步。第 10 条要求的"名字已变：原为 X；同位置现在是 eNNN 'Y'"那一行没有出现。
- **原因（源码推断）**：
  - `UiRefRepair.kt:50-68` 的 `nameChange` 要求新旧身份除名字外完全相同（`current.copy(name = original.name) == original`）。
  - 身份里含行标签（`SelectionIdentity.kt:8-15`）。行标签由 `UiSemantics.kt:204-226` 求出：从目标所在的列表项开始，取子树里第一段非空的 `text`；取不到时，用该项自己的名字。
  - 步行标签本身就是列表项（它带 `rowIndex=0 columnIndex=2`）。加载时，时长那段文字是空的，行标签就退回到它自己的名字 `"Walking mode: "`；加载完以后，行标签变成子文字 `"18 min"`。名字一变，行标签也跟着变，严格判定永远命不中。
  - 实施方 `r6-maps-ref-diagnosis.md` 的结论是"未定位到代码错误"，但它已经指出行标签"可采用节点自己的选择名字"。我认为这正是原因：**凡是目标本身就是列表项、名字来自自身文字的情况，第 10 条那一行都出不来**，而这正是第 10 条最初针对的 Maps 步行标签。
- **模型怎么恢复？** T6 补 snapshot，T7 点新引用。多花 2 条命令，约 4.3 万输入（T5 和 T6 两回合的输入）。

**分类**：确定缺陷（相对于裁决的目的；代码按裁决的字面执行没有错）。去向：进下一轮。

**最小修正**：只在"名字已变"这一行诊断里，当行标签取自目标自己的子树或自己的名字时，不比较行标签，只比较来源节点编号、窗口、角色和完整的行列链。这一行只是提示，不派发，放宽的风险很低。代价：几行代码加两条测试（对应名字由空变非空、由短变长）。

**对第六轮的检讨**：决定文件第四节第 7 处"行标签是区分同位置已换内容的字段，判定时不能省"，来自评审方的风险提示（工作记录 83、85）。这条提示没有考虑到"行标签取自目标自身"这种情况，结果把裁决想要的效果完全抵消了。

### 3.2 第 4 次 T6：`click e404859888`，退出码 2

**上一份回执**（T5 `fill --submit`，节选）：
```
fill: done; change=unsettled at windows,e404859879:text,showingHintText,focused,actions (page may still be transitioning); fill=done, verification="matches", …; submit=done; observationWarning="partial; missing=3"; …
observation {"partial":true,"missing":3}
screen app="com.google.android.apps.maps" title="Maps" keyboard="shown" unreadable
list rows=3 columns=1
 item e404859888 "National Gallery" clickable detail="Trafalgar Square, London, UK" rowIndex=0 columnIndex=0
 item e404859901 "Osteria Locatelli at the National Gallery" clickable detail="Sainsbury Wing National Gallery, Trafalgar Square, London, UK" rowIndex=1 columnIndex=0
textbox e404859879 "Choose start location" value="National Gallery, Trafalgar Square, London" … covered="e404859888"
```
**本次回执**：与 3.1 相同的一行 `code=stale`。

- **两份样本之间发生了什么？** T7 snapshot 显示，Maps 已经进入驾车路线页（`"Selected route 7 minutes via Charing Cross Rd/A400"`），建议列表消失了。也就是说，T5 那份回执里的树是**提交还没生效时**的样子。
- **拒绝对不对？** 正确。原节点已经不存在，如果改投到同位置的某个节点，只会点错。
- **发命令前能不能看出来？** 能看出风险：首行写着 `submit=done`、`unsettled at windows`、`partial`。但树看上去像一份可以直接操作的建议列表，容易误导模型。
- **模型怎么恢复？** 补 snapshot，再点步行，多花 2 回合，约 4.6 万输入。
- **核对实施方的说法**：实施方在 `core-results.md` 第 285 行写"前后目标不同，不能归为名称变化"，属实。

**分类**：拒绝本身已核实无问题。可选建议：对 `fill --submit` 这类导航动作，在下列两种情况下继续取样，直到时限用完：首行是 `unsettled at windows` 且树里还保留着提交前的输入框焦点；或者键盘还在弹出状态。否则就在首行写明"树可能早于提交生效"。去向：进下一轮。

### 3.3 第 3 次 T13 和第 4 次 T23：`routeUnavailable`，退出码 2

**本次回执**：
```
scroll: not sent; Original target is not ready for scroll-down: target has no advertised route for scroll-down; current scroll directions: backward, up; use androidctl scroll e719654421 backward; use androidctl scroll e719654421 up; code=routeUnavailable; cached; dispatchStarted=false; timing=before-action
```

- **发命令前能不能看出来？** 不能。上一份回执（第 3 次 T12、第 4 次 T22）是在滚动途中取的样（`unsettled`），树里同一个容器仍然写着 `scroll="forward backward up down"`。拒绝用的是之后到达的一份更新样本（首行有 `cached`），这份样本里的事实调用者在发命令前看不到。
- **拒绝对不对？** 正确。容器已经到底或到顶，再滚也不会有效果。
- **回执给的下一步能照做吗？** 字面上能照做，但对任务没有用：模型想继续往下，回执给的是"往回滚"。第 3 次之后用了 snapshot 加截图（3 回合），第 4 次用了 snapshot 核对（1 回合）。
- 上一轮第 1、2 次是同样的"已到顶再往上滚"。

**分类**：设计取舍。去向：不必处理。可选建议：滚动之后的回执如果是 unsettled，等容器公布的方向稳定下来再取样，收益约 0.4 回合/次。

### 3.4 第 5 次 T17：`scroll e743705522 down`，退出码 1

**回执**：
```
scroll: failed; change=unsettled baselineAgeMs=19046 (page may still be transitioning); dispatchStarted=true; Estimated next sample exceeds the observation deadline; snapshotPath=…
Requested: {"target":{"ref":"e743705522"},"action":"scroll-down"}
Resolved: {"method":"original","target":{"ref":"e743705522","role":"scroll","name":"","windowId":31,"bounds":[0,0,720,1280]}}
Selected: {…,"action":"scroll-forward"}
```

- **这不是观察超时。**
  - 这条命令从启动到退出只用了 1.8 秒（`events.jsonl`：spawnBefore 7204.79 s，waitReturned 7206.58 s）。
  - 源码推断：普通 scroll 在派发之后结果为 `failed`，只有一条路径，就是平台的 `performAction` 返回 false，或者回调为 false（`ActionReceipt.kt:20-28`）。这条路径产生的结果没有 `reason`（`UiActions.kt:900`，`outcome(command, callback.classify(), data=…)`）。
  - 主机端的首行只在有 `reason` 时打印原因，平台的接受与否不会打印（`ui_output.py:55-68`、`479-481`）。
  - 于是首行只剩下 `samplingStopped` 那一句（`ui_output.py:102-105`），读起来像是观察超时。
  - 实施方的成本页（`core-results.md` 第 291 行）和本专题的任务说明都写成了"派发后观察超时"，说明这个误读是真实存在的。
  - 被滚的目标是外层的整屏底部面板（`[0,0,720,1280]`，只有 `+collapse`、`scroll="forward backward"`），平台拒绝向前滚，这在页面上说得通。
- **主机端复现**：`tools/render_fail.py` 构造了 `result=failed`、`dispatchStarted=true`、`platform={"accepted":false}`、`samplingStopped=…` 的结果，交给 `ui_output.result_line` 渲染，输出与原件一致：`scroll: failed; change=unsettled (page may still be transitioning); dispatchStarted=true; Estimated next sample exceeds the observation deadline`。
- **调用者的影响**：模型没有重试同一个容器，T18 改滚内层列表 `e743705782`，成功了。这次多花 1 回合，约 3.5 万输入。风险在于：调用者可能误以为滚动已经执行、只是没观察到，从而按"已滚一页"去推断位置。
- **是不是 Maps 这类一直在动的页面必然会这样？**
  - "预计下一次采样超过观察时限"在 Maps 上很常见：本轮 Maps 的动作回执约有一半是 unsettled 或 loading。但它只是一个附带说明，不是失败的原因。
  - 失败本身来自平台拒绝，和页面有没有动画无关。

**分类**：确定缺陷（输出）。去向：进下一轮。

**最小修正**：平台拒绝时，首行写 `platform rejected the action (accepted=false)` 或对应的原因码；失败的原因另有来源时，把 `samplingStopped` 移到首行之后。代价：在设备端或主机端各补一个原因字段，加一条测试。

### 3.5 第 5 次 T24：`scroll-until e743706398 down … --contains 'Address:'`，退出码 1

**回执首行（节选）**：
```
scroll-until: failed; After scrolling 2 steps, the next scroll was not sent: Original target is not ready for scroll-down: target has no advertised route for scroll-down; current scroll directions: backward, up; use androidctl scroll e743706398 backward; use androidctl scroll e743706398 up; change=changed at content baselineAgeMs=996; dispatchStarted=true; Estimated next sample exceeds the observation deadline; match={…"matches":0…}; completedSteps=2; stop=routeUnavailable; step={"command":"scroll-down","result":"not sent","reason":"After scrolling 2 steps, …（同一句原因又重复一遍）
```

- **有没有准确交代"已经滚了几步"？** 有，`After scrolling 2 steps` 和 `completedSteps=2` 都写了。结果判为 `failed`（已有效果，但条件没满足）也正确（`ScrollProgress.kt:178-179`）。
- **回执给的下一步能照做吗？** 照做没有用。正确的做法是改滚外层容器里面的内层列表（树里就有 `list e743706578 id="recycler_view" scroll="forward"`），回执没有提示这一点。模型 T25 自己改了容器，一步就找到了起点地址。
- **这次也不是观察超时**：停止原因是 `routeUnavailable`，"预计下一次采样超过观察时限"仍然只是附带说明。
- **噪声**：同一句原因在首行里出现了两次（一次在开头，一次在 `step={…}` 里），首行约 900 字节。

**分类**：可选建议。去向：进下一轮。两处可改：外层容器滚到尽头时，提示"树里还有可以滚动的内层容器 eNNN"；首行里的 `step` 不再重复原因。收益约 0.2 回合/次，代价小。

### 3.6 汇总

| 运行与回合 | 原因 | 发命令前能否看出 | 拒绝或失败是否正确 | 恢复多花 |
| --- | --- | --- | --- | --- |
| 第 1 次 T5 | 名字变化导致旧引用被拒 | 能（loading） | 安全但不必要；第 10 条那一行没出现 | 2 回合 |
| 第 4 次 T6 | 提交后建议项已消失 | 部分能 | 正确 | 2 回合 |
| 第 3 次 T13 | 已到底 | 不能 | 正确 | 1+2 回合 |
| 第 4 次 T23 | 已到顶 | 不能 | 正确 | 1 回合 |
| 第 5 次 T17 | 平台拒绝滚动 | 不能 | 结果对，原因没打印 | 1 回合 |
| 第 5 次 T24 | 外层容器方向用尽 | 不能 | 结果对，建议无用 | 1 回合 |

6 次合计约 11 回合，平均每次运行 2.2 回合。上一轮 7 次非零退出合计约 9 回合。**这一项不是回升的主要原因。**

---

## 4. 后台屏题

**会话打开。**
- 5/5 次都写了 `session open --display virtual`。后台屏必须写 `--display`；第 25 条"可以不写"只针对主屏。
- 回执是一行：`session open: done; target=virtual/display2; Target established`，65 字节，上一轮是 84 字节。显示写在首行里。
- 5/5 次都用了 `apps Contacts`，回执 45 字节。上一轮是 `apps` 不带查询词，622 字节。这是首页示例新增 `apps QUERY` 的效果。

**后台屏上的 `open`。**
- 首行写明了 `display=2`。
- 第 1、3 次附带了界面树，模型直接点"Create new contact"，省了 1 回合。
- 第 2、4、5 次的树是：
```
artifact kind="tree" state="unavailable" code="scopeUnknown" deliveryError="One complete task window is required; no windows enumerated; run snapshot when the screen is available"
```
  这 3 次随后的 snapshot（593 字节）都正常拿到了树。
- 上一轮 `open` 从来不带树，而且回执有 2,020 字节，全是启动细节。所以第 15 条"落地后带上界面树"在后台屏上只做到了 2/5。
- 3 次失败的原因推断是：刚启动时后台屏上还没有获得焦点的窗口，而后台屏取的是有焦点的非输入法窗口（`displays.md`："virtual uses focused non-IME"）。未在设备上核实。
- 可选建议：`open` 在后台屏上等到有焦点的窗口出现，或者等到时限用完，再采样。收益是每次运行 0.6 回合。

**填表、保存、核对、删除。**
- 填写回执自带 `verification="matches"`；点 SAVE 的回执直接带着已保存的名字和 `Note … detail="Local validation only"` 以及 toast。
- 重新打开联系人那次点击的回执已经足以核对，5/5 次都没有为核对另发 snapshot。
- 删除走了 3 次点击：More options → Delete → DELETE。最后一次的回执带着 `toast text=["R4 Local Contact deleted",…]` 和"Your contacts list is empty"。
- 多余的确认调用只有两类：
  - 第 1、2、3 次在 `scroll-until` 之前先 `find Notes`。`find` 已经给出了屏外的 `textbox e973002576 "Notes"`，模型仍然先滚动再填写。
  - 第 1、2、5 次在关闭前查了一次 `status`，第 3、4 次关闭前后都用了 `--json status`。
- 这两类都是模型自己的谨慎，没有一次是被产品输出逼出来的。

**关闭回执。**
```
session close: done; Session ended; UI cleanup confirmed
taskDisposition={"state": "clear", "displayId": 2, "destroyed": true, "openedTasks": [{"taskId": 59, …"baseActivity": "com.android.contacts/…PeopleActivity", "topActivity": "…同上", "realActivity": "…同上", "origActivity": null, "baseIntentComponent": "…同上", "topActivityInfo": {…同上…}}], "movedTasks": [], "foreignTasks": [], "unknownTasks": [], "returnedTasks": [], "unreturnedTasks": []}
```
- 有用的信息：`destroyed: true`，任务号，以及 moved、foreign、unknown 三类都为空。
- 噪声：同一个组件重复了 5 次，约 500 字节。
- 5/5 次的最终回答都准确引用了这些事实。模型正确处理了后台屏上的任务：没有搬屏，没有外来任务，没有触发收尾保护。

**与上一轮比较。**
- 回合数持平（中位数 17）。
- 命令数：上一轮 17/18/19/19/17，本轮 18/20/19/18/18。
- 单条输出平均从约 1,600 字节降到约 880 字节；每回合新增的输入从约 970 token 降到约 780 token。
- 总输入中位数从 454,100 降到 425,366。
- 本轮没有一次像上一轮第 2 次那样把命令拆开发。

**分类**：已核实无问题。`open` 不带树（3/5）和关闭回执里的重复内容属于可选建议，进下一轮。

---

## 5. 模型实际看到的输出：典型摘录与逐行评论

### 5.1 路线题

**摘录 ①：第 4 次 T5，`fill --submit` 的回执**（全文 23 行，节选关键行）
```
fill: done; change=unsettled at windows,e404859879:text,showingHintText,focused,actions (page may still be transitioning); fill=done, verification="matches", actualValue="National Gallery, Trafalgar Square, London"; submit=done; observationWarning="partial; missing=3"; snapshotPath="ultragoal\\…\\workspace\\.androidctl\\snapshots\\000003-…-Maps.txt"
androidctl capture="d0f8cddd-…:9"
observation {"partial":true,"missing":3}
screen app="com.google.android.apps.maps" title="Maps" keyboard="shown" unreadable
window id=25 type="inputMethod" layer=1 title="English (US) (QWERTY)" content="notRead"
window id=30 active focused content="partial" selected
edge-windows count=1 content="notRead" windows=[{"id":26}]
item e404859440 clickable
list rows=3 columns=1
 item e404859888 "National Gallery" clickable detail="Trafalgar Square, London, UK" rowIndex=0 columnIndex=0
  item e404859899 "Activate to enter suggestion National Gallery into search bar" clickable
 …
textbox e404859879 "Choose start location" value="National Gallery, Trafalgar Square, London" clickable +long-click +fill covered="e404859888"
```
- 第 1 行：`verification="matches"`、`submit=done` 有用，`unsettled at windows` 有用（警告）。路径约 200 字节，对这一步的决定是噪声。
- `capture`：按第 15 条必须保留，可以接受。
- `observation` 和 `partial` 两行有用（缺口标记）。
- 两行 `window` 加上 `edge-windows`，对调用者是噪声。
- 树：这里会误导人。它显示的是提交前的建议列表，模型据此去点，结果被拒（见 3.2）。

**摘录 ②：第 1 次 T17，在 Museum 的 Overview 页向下滚动**（7,275 字节，节选）
```
scroll: done; change=unsettled at content (page may still be transitioning); snapshotPath=…
Requested: {"target":{"ref":"e608126234"},"action":"scroll-down"}
Resolved: {"method":"original","target":{"ref":"e608126234","role":"list","name":"","windowId":21,"bounds":[0,284,720,1110]}}
Selected: {"target":{…同上…},"action":"scroll-forward"}
…
list e608126234 id="recycler_view" scroll="forward backward" rows=0 columns=0
 item e608126392 "Report a problem" clickable
 item e608126398 clickable
 item e608126399 clickable
 item e608126400 clickable
 item e608126401 "See more" clickable
 item e608126416 "Abdullatif Ahmed" clickable detail="5.0 stars | 2 months ago | The British Museum is truly a must-visit destination … （全文）| The British Museum is truly … （截断重复）| ​An exceptional experience … （再截断重复）| more | Thumbnail #1 | Thumbnail #2"
 item e608126433 "riel" clickable detail="4.0 stars | … （同样三遍）"
```
- `Requested`、`Resolved`、`Selected` 三行：只有 `Selected` 里的 `scroll-forward` 有信息量（说明方向映射发生了），`Resolved` 和 `Selected` 的目标完全重复。
- 3 个没有名字的 `item … clickable`：调用者完全没法用，是模型去截图看图的直接诱因之一（推断）。
- 评论的 detail 把同一段文字拼了三遍，两条评论共约 3.3 KB，占这份输出的 45%。本轮第 3、4 次每次有 6 条这样的行（约 10 KB）。
- 输出上限（64 KB）从未触发，最大一份 7,275 字节。模型从来没有去读保存的文件。

**摘录 ③：第 5 次 T20，`scroll-until` 成功**（4,267 字节，节选）
```
scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":2,"unknown":0,"refs":["e743706053","e743706057"],"field":"description","matchedText":"Address: Great Russell St, London WC1B 3DG, United Kingdom","windowId":31}; stop=found; snapshotPath=…
…
 item e743706053 "Address: Great Russell St, London WC1B 3DG, United Kingdom" clickable +long-click
  text "Great Russell St, London WC1B 3DG, United Kingdom"
 item e743706062 "Phone Number: +44 20 7323 8000" clickable +long-click
```
- 首行的 `matchedText` 直接就是答案，这是写得好的一例。
- 一条命令就完成了另外 4 次运行用 6 到 12 个回合都没做成的事。所以 D 段最大的节省点在**引导模型用 `scroll-until`**，而不是改输出。

**截图看图**：本轮共 7 次，上一轮 2 次，每次占 2 个回合（带截图的 snapshot 一回合，看图一回合）。都是为了看清楚地点页上没有名字的列表项是什么内容，没有一次用坐标点击，也没有一次靠截图绕过产品缺口。所有事实最终都来自界面树。

### 5.2 后台屏题

**摘录 ④：第 2 次 T5，后台屏上 `open` 没带树**
```
open: done; change=unsettled (page may still be transitioning); New worker: previous refs are invalid; display=2; snapshotPath=…
androidctl capture="4342367d-…:1"
screen keyboard="hidden" unreadable
artifact kind="tree" state="unavailable" code="scopeUnknown" deliveryError="One complete task window is required; no windows enumerated; run snapshot when the screen is available"
```
- `display=2`、"New worker: previous refs are invalid"：有用。
- 最后一行说得清楚，也给了下一步（run snapshot），模型照做，一回合就恢复了。
- 对比第 1 次：同样位置附带了 7 行树，直接就能点。

**摘录 ⑤：第 1 次 T8，`find Notes`**（21 行，节选）
```
find: done; cached; matches=2; offscreenMatches=2; snapshotPath=…
find query="Notes" matches=2 offscreen=2 shown=2
text e973002459
 text e973002460
  … （6 层只有编号的祖先）
       omission reason="find" nodes=5 read="snapshot --cached --all" ref="e973002465"
       text e973002470 id="contact_editor_fragment"
        scroll e973002471 id="contact_editor_scroller" scroll="forward backward up down"
         …
          omission reason="find" nodes=83 read="snapshot --cached --all"
          text e973002570 offscreen
           image e973002571 "Notes" id="kind_icon" offscreen
            … textbox e973002576 "Notes" value="" clickable +long-click +fill offscreen
```
- 有用：命中节点及其 `offscreen` 标记，滚动容器 `e973002471` 及其方向。
- 噪声：6 层只有编号、没有名字的祖先节点，以及 3 行 `omission`。可以只保留"最近的可滚动祖先"加命中节点。
- 模型拿到了 Notes 的引用，仍然先滚动再填写。指引写的是"Click ready targets directly … or scroll the container to expose a target"，模型的选择是合理的。

**摘录 ⑥：第 1 次 T16，`session close` 加 `status`**：见第 4 节。`status` 的 `gateway={…12 个字段…}` 和 `coordinates=…` 对这道题是噪声。另外，`pairing=` 打印了主机名。

**整体字节构成**（`tools/noise.py`，本轮路线题 5 次全部 CLI 输出约 374 KB）：

| 部分 | 占比 |
| --- | --- |
| 界面树 | 80.0% |
| 首行 | 9.4%（其中路径 6.6%） |
| 头部行（capture、screen、window、edge-windows） | 8.3% |
| `covered=` 属性 | 2.8% |
| Requested、Resolved、Selected | 1.8% |

后台屏题：首行 25.8%，其中路径 17.4%。路径这么长，是因为评测时工作区目录很深（`ultragoal\…\workspace\.androidctl\snapshots\…`），普通用户的路径会短很多。

---

## 6. 指引的实际使用

- **路线题**：
  - 5/5 次只读了 `SKILL.md`（7,173 字节），没有读任何专题。
  - 只有第 3 次读过一次 `collapse --help`。原因是树里有 `+collapse` 属性，而首页没讲这个命令怎么用，属于正常查询。
  - 没有一次因为指引没讲而反复试错。
  - 唯一的偏差是首页示例 `fill … --submit` 对路线题的影响（见 2.2 (g)）。首页其实写了"add `--submit` only when submission is intended"，但示例本身就带着 `--submit`，模型照着示例用了。
- **后台屏题**：
  - 5/5 次按首页的要求读了 `displays.md`，4/5 次顺带读了 `reference.md`，没有读 `--help`。
  - 没有一次"照着指引做却失败"。
- **没有被用到的指引**：首页关于 `scroll-until` 的段落写的是"For lazy content, use a bounded search"。路线题的 D 段正是这种场景，但只有第 5 次用了。可以考虑在首页补一句"要找的是首屏以下的某段文字时，优先用 `scroll-until --contains`，不要反复滚动加截图"（估算收益见第 8 节）。

---

## 7. 业务完成度

- **路线题**：
  - 5/5 次的距离、时长和前三段都正确（与 `core-results.md` 中操作者的独立核对一致）。
  - 5/5 次都回到了 Museum 地点页，没有开始导航，也没有保存或分享。
  - 地址完整度：
    - 第 5 次两端都是带邮编的完整地址。
    - 第 2 次起点是"Trafalgar Square, London, UK"，终点是完整地址。
    - 第 4 次起点同第 2 次，终点只有"Great Russell St"。
    - 第 1、3 次起点只有输入的文字，终点只有"Great Russell St"。
  - 所有缺失都在最终回答里如实说明了，没有虚报，也没有补写猜测的邮编。
- **后台屏题**：5/5 次完成了业务、确认了屏的归属并完成了关闭后的释放，最终回答与回执一致。
- 两题都没有靠截图绕过产品缺口。

---

## 8. 最短合理路径与改进清单

### 8.1 路线题

**最短合理路径**，约 13–15 回合：
1. 线程号加首页
2. snapshot
3. 点 Directions
4. 填起点（不提交），读建议列表里的地址
5. 点建议项
6. 如果回执显示仍在加载，补一次 snapshot
7. 点步行
8. 点路线卡片
9. 滚动读完三段
10. Close
11. 对 Overview 内层列表 `scroll-until --contains "Address:"`（必要时先滚一次外层）
12. 滚回顶部
13. 最终回答

**本轮中位数 26，差约 11–13 回合**，按中位数那次运行粗分：
- 找 Museum 地址时手动滚动加截图：约 6
- 回执还在加载或采样不完整，补 snapshot：约 2
- 旧引用和方向不可用的恢复：约 2
- 起点地址绕路（平均）：约 2
- 误离地点页后重新进入：约 1

### 8.2 后台屏题

**最短合理路径**，约 13 回合：
1. 线程号加首页
2. displays
3. `session open` 加 `apps` 加 `open`
4. 新建
5. 填名字加 More fields
6. `scroll-until Notes`
7. 填备注加 SAVE
8. back
9. 点开联系人（同时核对）
10. More options
11. Delete
12. DELETE 加 `session close` 加 `status`
13. 最终回答

**本轮中位数 17，差 4 回合**：`open` 没带树要补 snapshot（0.6）；先 find 再 scroll-until（0.6）；关闭前的 status（0.6）；模型拆分回合的习惯（约 2）。

### 8.3 改进清单（按"修一处能省多少回合"排序）

| 排名 | 改什么 | 类型 | 估计收益 | 代价 | 去向 |
| --- | --- | --- | --- | --- | --- |
| 1 | 成本页按阶段拆分，和上一轮候选比较；在题目里写清"核两端地址"的完成标准 | 评测口径（不改产品） | 路线题去掉约 4–10 回合的波动，否则无法把回升归因到产品 | 只改文档 | 进下一轮 |
| 2 | 首页补一句：要找首屏以下的某段文字时，优先用 `scroll-until --contains`，不要反复滚动加截图 | 改指引 | 路线题 D 段估计省 3–6 回合（参照第 5 次一步命中） | 一行 | 进下一轮 |
| 3 | `fill --submit` 这类导航动作：首行显示 loading 或 unsettled at windows 时继续取样到时限用完；否则在首行说明"树可能早于提交生效" | 改产品 | 路线题 5/5 次在提交后多了 1–2 回合，估计省约 1 回合/次 | 动作耗时变长；要改阈值和测试 | 进下一轮 |
| 4 | 第 10 条那一行：行标签取自目标自身时不比较行标签 | 改产品 | 名字变化导致的旧引用拒绝每次省 1 回合（本轮 1/5，上一轮 3/5） | 几行代码加两条测试；只是提示，不派发 | 进下一轮 |
| 5 | 后台屏上的 `open` 等到有焦点的窗口出现再采样 | 改产品 | 后台屏题省 0.6 回合/次 | 启动耗时略增 | 进下一轮 |
| 6 | 平台拒绝的动作，首行写明原因；`samplingStopped` 不要冒充原因 | 改输出 | 约 0.2 回合/次；主要价值是防止误读（实施方已经误读过） | 很小 | 进下一轮 |
| 7 | `scroll-until` 在外层容器到尽头时，提示可以改滚的内层容器；首行里的 `step` 不再重复原因 | 改输出 | 约 0.2 回合/次 | 小 | 进下一轮 |
| 8 | 输出去噪：detail 里重复的段落去重（Maps 评论）；`edge-windows` 是 notRead 且没有变化时不打印；首行路径改为相对 `.androidctl` 的短路径；`taskDisposition` 改成一行摘要 | 改输出 | 不省回合；每回合输入估计再减 5–10% | 改默认文本契约（第 15 条的延续），需要用户点头 | 进下一轮 |

不在清单里、但值得记下的一点：第六轮附录一"首页示例改成 `fill --submit`"这条建议，**不建议撤回**。它让路线建立少了 1 回合，也符合网页题的用法。它的副作用由第 1、2 条处理就够了。

---

## 9. 已核实无问题的清单

- 本轮 6 次非零退出，没有一次是"已经执行却报未发出"或"未执行却报已完成"。所有 `not sent` 都带 `dispatchStarted=false`；两次 `failed` 都带 `dispatchStarted=true`，其中 `scroll-until` 那次准确写了 `completedSteps=2`。没有出现 UNKNOWN。
- 两次旧引用拒绝都是安全的：第 4 次那次必须拒绝，第 1 次那次即使派发也不会点错，但拒绝不造成任何损害。
- 方向映射（附录一）在真实任务里生效了，上一轮那 2 次方向词拒绝本轮没有再出现。
- 输出上限（第 14 条）没有误伤：最大的单份输出只有 7.3 KB，模型从来不需要去读保存的文件。
- 后台屏的打开、关闭和状态回执足以让模型自主报告释放结果；首行都写了显示；5/5 次没有搬屏，也没有外来任务。
- `apps QUERY` 示例和瘦过的 `open` 回执，让后台屏题的单条输出缩小了约 45%。
- 本轮 10 次运行没有发现虚报。

---

## 10. 未能核实的事项

- 本轮题目的提示原文在原生记录里是加密的，没法逐字核对"沿用原题"。
- 第 1 次旧引用拒绝时，第 10 条那一行到底没命中在哪个字段：我的推断是行标签变了（`UiSemantics.kt:204-226` 加上树的形状），但原件没有保存含 `sourceId` 和 `row.label` 的原始树，没法实测确认。来源节点编号是否也变了，同样未知。
- 第 5 次 T17 的失败，究竟是平台返回 false 还是回调返回 false：两者在源码里都会得出 `failed` 且没有原因，而原件没有保存 JSON 结果。
- 后台屏上 `open` 出现 `scopeUnknown` 的确切时序原因，没有设备可以核实。
- D 段模型行为变化的心理原因（为什么本轮比上一轮更执着地找地址）无法从记录里确定；加密的推理内容不可读。
- Maps 的 Overview 内容顺序不同（第 3 次与第 5 次），是 A/B 实验还是时段造成的，未知。

---

## 附录 A：路线题逐条步骤表（本轮 5 次）

（由 `S/agents11/97/tools/table.py` 从 `native.rollout.jsonl` 直接生成。命令按原样列出，只去掉了 `--device`、`--session`、`--workspace-root` 和 PowerShell 的退出码包装；首行里的长路径以"…"代替。）

#### r6-c1-candidate-01

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,140 | `(读线程号)` | 0 | 38 | 01a213c8-1bcd-74c3-93b6-0851bea6d0dc |
| 1 | 17,140 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 2 | 19,120 | `snapshot` | 0 | 2567 | snapshot: done; snapshotPath=… |
| 3 | 20,442 | `click e608125440` | 0 | 1584 | click: done; change=unsettled (page may still be transitioning); observationWarning="partial; missing=9"; snapshotPath=… |
| 4 | 21,016 | `fill e608125571 'National Gallery, Trafalgar Square, London' --submit` | 0 | 2374 | fill: done; change=loading at windows (page may still be transitioning); fill=done, verification="matches", actualValue="National Gallery, Trafalgar Square, London"; submit=done; snapshotPath=… |
| 5 | 21,809 | `click e608125688` | 2 | 141 | click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action |
| 6 | 21,954 | `snapshot` | 0 | 3506 | snapshot: done; snapshotPath=… |
| 7 | 23,090 | `click e608125823` | 0 | 2603 | click: done; change=unsettled (page may still be transitioning); No primary accessibility node reply was available by the collection deadline; the app may be unresponsive; observationWarning="partial; missing=2"; snapshotPath=… |
| 8 | 23,928 | `snapshot` | 0 | 3339 | snapshot: done; snapshotPath=… |
| 9 | 24,999 | `scroll e608125845 down` | 0 | 4037 | scroll: done; change=changed at content,e608125845:bounds; snapshotPath=… |
| 10 | 26,329 | `scroll e608125845 down` | 0 | 4381 | scroll: done; change=changed at content,e608125845:actions; snapshotPath=… |
| 11 | 27,781 | `click e608125669` | 0 | 2652 | click: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 12 | 28,775 | `scroll e608126131 down` | 0 | 3471 | scroll: done; change=unsettled at content,e608126131:bounds,actions (page may still be transitioning); snapshotPath=… |
| 13 | 29,889 | `scroll e608126131 down` | 0 | 3543 | scroll: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 14 | 31,033 | `snapshot --screenshot` | 0 | 4107 | snapshot: done; snapshotPath=…; imagePath=…; imageSize=720 x 1280 px; imageMime=image/jpeg; imageSourceSize=720 x 1280 px; imageMappingKnown=true |
| 15 | 32,369 | (看截图) | | | |
| 16 | 33,603 | `scroll e608126234 down` | 0 | 3991 | scroll: done; change=changed at content,e608126234:actions; snapshotPath=… |
| 17 | 34,946 | `scroll e608126234 down` | 0 | 7275 | scroll: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 18 | 36,916 | `scroll e608126234 up` | 0 | 3971 | scroll: done; change=changed at content; snapshotPath=… |
| 18 | 36,916 | `snapshot --screenshot` | 0 | 4258 | snapshot: done; snapshotPath=…; imagePath=…; imageSize=720 x 1280 px; imageMime=image/jpeg; imageSourceSize=720 x 1280 px; imageMappingKnown=true |
| 19 | 39,564 | (看截图) | | | |
| 20 | 40,798 | `scroll e608126208 right` | 0 | 3634 | scroll: done; change=changed at content,e608126208:actions; snapshotPath=… |
| 21 | 42,000 | `click e608126228` | 0 | 3694 | click: done; change=changed at content; snapshotPath=… |
| 22 | 43,201 | `scroll e608126208 left` | 0 | 3690 | scroll: done; change=changed at content,e608126208:actions; snapshotPath=… |
| 23 | 44,400 | `click e608126467` | 0 | 3644 | click: done; change=changed at content; snapshotPath=… |
| 23 | 44,400 | `scroll e608126131 up` | 0 | 3486 | scroll: done; change=changed at content,e608126131:actions; snapshotPath=… |
| 24 | 46,658 | （最终回答） | | | |

#### r6-c1-candidate-02

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,140 | `(读线程号)` | 0 | 38 | 01a213cf-6b5c-73d0-8505-786b4c2f4bf1 |
| 1 | 17,140 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 2 | 19,121 | `snapshot` | 0 | 2567 | snapshot: done; snapshotPath=… |
| 3 | 20,429 | `click e863963156` | 0 | 721 | click: done; change=unsettled at windows (page may still be transitioning); observationWarning="partial; missing=2"; snapshotPath=… |
| 4 | 20,766 | `snapshot` | 0 | 1282 | snapshot: done; snapshotPath=… |
| 5 | 21,253 | `fill e863963366 'National Gallery, Trafalgar Square, London' --submit` | 0 | 1004 | fill: done; change=unsettled (page may still be transitioning); fill=done, verification="matches", actualValue="National Gallery, Trafalgar Square, London"; submit=done; observationWarning="partial; missing=2"; snapshotPath=… |
| 6 | 21,677 | `snapshot` | 0 | 3542 | snapshot: done; snapshotPath=… |
| 7 | 22,821 | `click e863963440` | 0 | 2697 | click: done; change=loading (page may still be transitioning); observationWarning="partial; missing=1"; snapshotPath=… |
| 8 | 23,686 | `snapshot` | 0 | 3339 | snapshot: done; snapshotPath=… |
| 9 | 24,749 | `click e863963673` | 0 | 3709 | click: done; change=changed at content,e863963673:bounds; snapshotPath=… |
| 10 | 25,955 | `scroll e863963464 down` | 0 | 4417 | scroll: done; change=unsettled at content,e863963464:actions (page may still be transitioning); snapshotPath=… |
| 11 | 27,408 | `scroll e863963464 up` | 0 | 4037 | scroll: done; change=changed at content,e863963464:actions; snapshotPath=… |
| 12 | 28,807 | `click e863963749` | 0 | 908 | click: done; change=changed at content; snapshotPath=… |
| 13 | 29,207 | `back` | 0 | 3614 | back: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 14 | 30,394 | `back` | 0 | 2695 | back: done; change=unsettled (page may still be transitioning); observationWarning="partial; missing=1"; snapshotPath=… |
| 15 | 31,281 | `scroll e863964083 down` | 0 | 3440 | scroll: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 16 | 32,395 | `scroll e863964083 down` | 0 | 3448 | scroll: done; change=unsettled at content,e863964083:actions (page may still be transitioning); snapshotPath=… |
| 17 | 33,509 | `snapshot --screenshot` | 0 | 4006 | snapshot: done; snapshotPath=…; imagePath=…; imageSize=720 x 1280 px; imageMime=image/jpeg; imageSourceSize=720 x 1280 px; imageMappingKnown=true |
| 18 | 34,811 | (看截图) | | | |
| 19 | 36,045 | `scroll e863964186 down` | 0 | 3890 | scroll: done; change=changed at content,e863964186:actions; snapshotPath=… |
| 20 | 37,330 | `scroll e863964186 down` | 0 | 7138 | scroll: done; change=changed at content; snapshotPath=… |
| 21 | 39,256 | `back` | 0 | 2141 | back: done; change=loading (page may still be transitioning); snapshotPath=… |
| 22 | 40,138 | `click e863964503` | 0 | 1715 | click: done; change=changed at windows,content; snapshotPath=… |
| 23 | 40,764 | `fill e863964554 'National Gallery, Trafalgar Square, London'` | 0 | 2373 | fill: done; change=changed at content,e863964554:text,showingHintText,actions; fill=done, verification="matches", actualValue="National Gallery, Trafalgar Square, London"; snapshotPath=… |
| 24 | 41,535 | `fill e863964554 'British Museum London'` | 0 | 2254 | fill: done; change=changed at content,e863964554:text; fill=done, verification="matches", actualValue="British Museum London"; snapshotPath=… |
| 25 | 42,306 | `click e863964772` | 0 | 2715 | click: done; change=unsettled (page may still be transitioning); observationWarning="partial; missing=1"; snapshotPath=… |
| 26 | 43,175 | `snapshot` | 0 | 2622 | snapshot: done; snapshotPath=… |
| 27 | 44,028 | （最终回答） | | | |

#### r6-c1-candidate-03

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,140 | `(读线程号)` | 0 | 38 | 01a213dc-aaed-75a2-b437-0eb9383b16c3 |
| 1 | 17,140 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 2 | 19,113 | `snapshot` | 0 | 2567 | snapshot: done; snapshotPath=… |
| 3 | 20,427 | `click e719653698` | 0 | 1314 | click: done; change=changed at windows,content; snapshotPath=… |
| 4 | 20,945 | `fill e719653839 'National Gallery, Trafalgar Square, London' --submit` | 0 | 2200 | fill: done; change=loading at windows (page may still be transitioning); No primary accessibility node reply was available by the collection deadline; the app may be unresponsive; fill=done, verification="matches", actualValue="Na… |
| 5 | 21,677 | `snapshot` | 0 | 3503 | snapshot: done; snapshotPath=… |
| 6 | 22,817 | `click e719654062` | 0 | 2718 | click: done; change=loading (page may still be transitioning); observationWarning="partial; missing=1"; snapshotPath=… |
| 7 | 23,705 | `snapshot` | 0 | 3339 | snapshot: done; snapshotPath=… |
| 8 | 24,783 | `click e719654235` | 0 | 3709 | click: done; change=changed at content,e719654235:bounds; snapshotPath=… |
| 9 | 26,005 | `scroll e719654084 down` | 0 | 4381 | scroll: done; change=changed at content,e719654084:actions; snapshotPath=… |
| 10 | 27,468 | `click e719653909` | 0 | 2693 | click: done; change=changed at content; snapshotPath=… |
| 11 | 28,516 | `scroll e719654421 down` | 0 | 3633 | scroll: done; change=changed at content,e719654421:bounds,actions; snapshotPath=… |
| 12 | 29,692 | `scroll e719654421 down` | 0 | 3731 | scroll: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 13 | 30,905 | `scroll e719654421 down` | 2 | 309 | scroll: not sent; Original target is not ready for scroll-down: target has no advertised route for scroll-down; current scroll directions: backward, up; use androidctl scroll e719654421 backward; use androidctl scroll e719654421 u… |
| 14 | 31,093 | `snapshot` | 0 | 3477 | snapshot: done; snapshotPath=… |
| 15 | 32,247 | `snapshot --screenshot` | 0 | 4115 | snapshot: done; snapshotPath=…; imagePath=…; imageSize=720 x 1280 px; imageMime=image/jpeg; imageSourceSize=720 x 1280 px; imageMappingKnown=true |
| 16 | 33,604 | (看截图) | | | |
| 17 | 34,838 | `scroll e719654525 forward` | 0 | 3669 | scroll: done; change=changed at content,e719654525:actions; snapshotPath=… |
| 18 | 36,046 | `scroll e719654525 forward` | 0 | 3324 | scroll: done; change=changed at content; snapshotPath=… |
| 19 | 37,139 | `snapshot --screenshot` | 0 | 3937 | snapshot: done; snapshotPath=…; imagePath=…; imageSize=720 x 1280 px; imageMime=image/jpeg; imageSourceSize=720 x 1280 px; imageMappingKnown=true |
| 20 | 38,434 | (看截图) | | | |
| 21 | 39,668 | `scroll e719654525 forward` | 0 | 6875 | scroll: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 22 | 41,519 | `scroll e719654525 forward` | 0 | 7237 | scroll: done; change=changed at content; snapshotPath=… |
| 23 | 43,523 | `scroll e719654421 up` | 0 | 6803 | scroll: done; change=unsettled at content,e719654421:actions (page may still be transitioning); snapshotPath=… |
| 24 | 45,397 | `collapse --help` | 0 | 1991 | Usage: python -m androidctl collapse [OPTIONS] [ref] |
| 25 | 46,003 | `collapse e719653885` | 0 | 2715 | collapse: done; change=changed at content,e719653885:actions; snapshotPath=… |
| 26 | 46,903 | （最终回答） | | | |

#### r6-c1-candidate-04

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 18,480 | `(读线程号)` | 0 | 38 | 01a215fa-2e8a-7600-8611-1295c8c5e56d |
| 2 | 18,610 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 3 | 20,563 | `snapshot` | 0 | 2563 | snapshot: done; snapshotPath=… |
| 4 | 21,815 | `click e404859793` | 0 | 1204 | click: done; change=unsettled (page may still be transitioning); observationWarning="partial; missing=7"; snapshotPath=… |
| 5 | 22,275 | `fill e404859879 'National Gallery, Trafalgar Square, London' --submit` | 0 | 1997 | fill: done; change=unsettled at windows,e404859879:text,showingHintText,focused,actions (page may still be transitioning); fill=done, verification="matches", actualValue="National Gallery, Trafalgar Square, London"; submit=done; o… |
| 6 | 22,982 | `click e404859888` | 2 | 141 | click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action |
| 7 | 23,124 | `snapshot` | 0 | 3291 | snapshot: done; snapshotPath=… |
| 8 | 24,178 | `click e404859991` | 0 | 3094 | click: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 9 | 25,186 | `click e404860127` | 0 | 3705 | click: done; change=changed at content,e404860127:bounds; snapshotPath=… |
| 10 | 26,400 | `scroll e404860015 down` | 0 | 4394 | scroll: done; change=unsettled at content,e404860015:actions (page may still be transitioning); snapshotPath=… |
| 11 | 27,857 | `click e404859970` | 0 | 2648 | click: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 12 | 28,823 | `scroll e404860310 down` | 0 | 3594 | scroll: done; change=unsettled at content,e404860310:bounds (page may still be transitioning); snapshotPath=… |
| 13 | 29,984 | `scroll e404860310 down` | 0 | 3549 | scroll: done; change=unsettled at content,e404860310:actions (page may still be transitioning); snapshotPath=… |
| 14 | 31,132 | `snapshot --screenshot` | 0 | 4107 | snapshot: done; snapshotPath=…; imagePath=…; imageSize=720 x 1280 px; imageMime=image/jpeg; imageSourceSize=720 x 1280 px; imageMappingKnown=true |
| 15 | 32,468 | (看截图) | | | |
| 16 | 33,702 | `scroll e404860413 down` | 0 | 3998 | scroll: done; change=changed at content,e404860413:actions; snapshotPath=… |
| 17 | 35,025 | `scroll e404860413 down` | 0 | 7275 | scroll: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 18 | 36,993 | `scroll e404860387 right` | 0 | 6901 | scroll: done; change=changed at content,e404860387:actions; snapshotPath=… |
| 19 | 38,859 | `click e404860407` | 0 | 3694 | click: done; change=changed at content; snapshotPath=… |
| 20 | 40,052 | `scroll e404860387 left` | 0 | 3690 | scroll: done; change=changed at content,e404860387:actions; snapshotPath=… |
| 21 | 41,243 | `click e404860642` | 0 | 6911 | click: done; change=changed at content; snapshotPath=… |
| 22 | 43,091 | `scroll e404860310 up` | 0 | 2687 | scroll: done; change=unsettled at content,e404860310:actions (page may still be transitioning); snapshotPath=… |
| 23 | 43,998 | `scroll e404860310 up` | 2 | 307 | scroll: not sent; Original target is not ready for scroll-up: target has no advertised route for scroll-up; current scroll directions: forward, down; use androidctl scroll e404860310 forward; use androidctl scroll e404860310 down;… |
| 24 | 44,182 | `snapshot` | 0 | 3291 | snapshot: done; snapshotPath=… |
| 25 | 45,243 | （最终回答） | | | |

#### r6-c1-candidate-05

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 18,480 | `(读线程号)` | 0 | 38 | 01a21601-471c-73e3-99d2-1189839a18af |
| 2 | 18,608 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 3 | 20,560 | `snapshot` | 0 | 2563 | snapshot: done; snapshotPath=… |
| 4 | 21,809 | `click e743704883` | 0 | 1314 | click: done; change=changed at windows,content; snapshotPath=… |
| 5 | 22,317 | `fill e743705013 'National Gallery, Trafalgar Square, London' --submit` | 0 | 1675 | fill: done; change=unsettled (page may still be transitioning); fill=done, verification="matches", actualValue="National Gallery, Trafalgar Square, London"; submit=done; observationWarning="partial; missing=5"; snapshotPath=… |
| 6 | 22,915 | `snapshot` | 0 | 3291 | snapshot: done; snapshotPath=… |
| 7 | 23,966 | `click e743705143` | 0 | 3395 | click: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 8 | 25,052 | `click e743705279` | 0 | 3705 | click: done; change=changed at content,e743705279:bounds; snapshotPath=… |
| 9 | 26,267 | `scroll e743705167 down` | 0 | 4358 | scroll: done; change=changed at content,e743705167:actions; snapshotPath=… |
| 10 | 27,713 | `click e743705458` | 0 | 1035 | click: done; change=changed at content; snapshotPath=… |
| 11 | 28,239 | `back` | 0 | 3610 | back: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 12 | 29,419 | `click e743705582` | 0 | 2497 | click: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 13 | 30,248 | `scroll e743705679 down` | 0 | 3322 | scroll: done; change=changed at content,e743705679:bounds,actions; snapshotPath=… |
| 14 | 31,323 | `scroll e743705679 down` | 0 | 3442 | scroll: done; change=unsettled at content (page may still be transitioning); snapshotPath=… |
| 15 | 32,438 | `snapshot --screenshot` | 0 | 4006 | snapshot: done; snapshotPath=…; imagePath=…; imageSize=720 x 1280 px; imageMime=image/jpeg; imageSourceSize=720 x 1280 px; imageMappingKnown=true |
| 16 | 33,745 | (看截图) | | | |
| 17 | 34,979 | `scroll e743705522 down` | 1 | 3845 | scroll: failed; change=unsettled baselineAgeMs=19046 (page may still be transitioning); dispatchStarted=true; Estimated next sample exceeds the observation deadline; snapshotPath=… |
| 18 | 36,236 | `scroll e743705782 down` | 0 | 3897 | scroll: done; change=changed at content,e743705782:actions; snapshotPath=… |
| 19 | 37,523 | `scroll e743705782 down` | 0 | 7138 | scroll: done; change=changed at content; snapshotPath=… |
| 20 | 39,455 | `scroll-until e743705782 down --max-steps 8 --contains 'Great Russell' --timeout 20s` | 0 | 4267 | scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":2,"unknown":0,"refs":["e743706053","e743706057"],"field":"description","matchedText":"Address: Great Russell St, London WC1… |
| 21 | 40,905 | `back` | 0 | 2141 | back: done; change=loading (page may still be transitioning); snapshotPath=… |
| 22 | 41,669 | `click e743706152` | 0 | 1710 | click: done; change=changed at windows,content; snapshotPath=… |
| 23 | 42,281 | `click e743706321` | 0 | 1832 | click: done; change=unsettled at windows,content (page may still be transitioning); snapshotPath=… |
| 24 | 42,932 | `scroll-until e743706398 down --max-steps 8 --contains 'Address:' --timeout 20s` | 1 | 4998 | scroll-until: failed; After scrolling 2 steps, the next scroll was not sent: Original target is not ready for scroll-down: target has no advertised route for scroll-down; current scroll directions: backward, up; use androidctl scr… |
| 25 | 44,472 | `scroll-until e743706578 down --max-steps 8 --contains 'Address:' --timeout 20s` | 0 | 4323 | scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":1,"unknown":0,"refs":["e743706785"],"field":"description","matchedText":"Address: Trafalgar Square, London WC2N 5DN, United… |
| 26 | 45,912 | `back` | 0 | 2357 | back: done; change=loading (page may still be transitioning); snapshotPath=… |
| 27 | 46,731 | `click e743706848` | 0 | 1894 | click: done; change=changed at windows,content; snapshotPath=… |
| 28 | 47,395 | `click e743707060` | 0 | 2540 | click: done; change=unsettled at windows,content (page may still be transitioning); snapshotPath=… |
| 29 | 48,241 | `snapshot` | 0 | 2660 | snapshot: done; snapshotPath=… |
| 30 | 49,126 | （最终回答） | | | |

## 附录 B：后台屏题逐条步骤表（本轮 5 次）

#### r6-v1-candidate-01

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,344 | `(读线程号)` | 0 | 38 | 01a21386-f823-7fc3-8a9a-c0ee89aa0074 |
| 2 | 17,470 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 3 | 19,421 | `(读指引 displays.md, reference.md)` | 0 | 13813 | # Main And Work Displays |
| 4 | 23,135 | `session open --display virtual` | 0 | 65 | session open: done; target=virtual/display2; Target established |
| 4 | 23,135 | `apps Contacts` | 0 | 45 | apps: done |
| 5 | 23,326 | `open com.android.contacts` | 0 | 691 | open: done; change=unsettled (page may still be transitioning); New worker: previous refs are invalid; display=2; snapshotPath=… |
| 6 | 23,643 | `click e973002458` | 0 | 1294 | click: done; change=unsettled at windows,selection (page may still be transitioning); snapshotPath=… |
| 7 | 24,152 | `fill e973002491 'R4 Local Contact'` | 0 | 1370 | fill: done; change=changed at content,e973002491:text,showingHintText,actions; fill=done, verification="matches", actualValue="R4 Local Contact"; snapshotPath=… |
| 7 | 24,152 | `click e973002514` | 0 | 1318 | click: done; change=changed at content; snapshotPath=… |
| 8 | 25,152 | `find Notes` | 0 | 1512 | find: done; cached; matches=2; offscreenMatches=2; snapshotPath=… |
| 9 | 25,705 | `scroll-until e973002471 down --max-steps 10 --contains Notes --timeout 15s` | 0 | 1951 | scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":1,"unknown":0,"refs":["e973002571"],"field":"description","matchedText":"Notes","windowId":24}; stop=found; snapshotPath=… |
| 10 | 26,449 | `fill e973002576 'Local validation only'` | 0 | 1623 | fill: done; change=changed at content,e973002576:text,showingHintText,focused,actions; fill=done, verification="matches", actualValue="Local validation only"; snapshotPath=… |
| 10 | 26,449 | `click e973002469` | 0 | 830 | click: done; change=changed at windows,selection; snapshotPath=… |
| 11 | 27,426 | `back` | 0 | 724 | back: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 12 | 27,757 | `click e973002633` | 0 | 677 | click: done; change=changed at windows,selection; snapshotPath=… |
| 13 | 28,079 | `click e973002653` | 0 | 753 | click: done; change=changed at windows,selection; snapshotPath=… |
| 14 | 28,426 | `click e973002677` | 0 | 478 | click: done; change=changed at windows,selection; snapshotPath=… |
| 15 | 28,691 | `click e973002705` | 0 | 680 | click: done; change=changed at windows,selection; snapshotPath=… |
| 15 | 28,691 | `status` | 0 | 565 | status: done |
| 16 | 29,287 | `session close` | 0 | 863 | session close: done; Session ended; UI cleanup confirmed |
| 16 | 29,287 | `status` | 0 | 449 | status: done |
| 17 | 29,903 | （最终回答） | | | |

#### r6-v1-candidate-02

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,344 | `(读线程号)` | 0 | 38 | 01a21392-8f13-72d2-9d92-297fba8a8654 |
| 2 | 17,472 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 3 | 19,434 | `(读指引 displays.md)` | 0 | 6219 | # Main And Work Displays |
| 4 | 21,300 | `session open --display virtual` | 0 | 65 | session open: done; target=virtual/display2; Target established |
| 4 | 21,300 | `apps Contacts` | 0 | 45 | apps: done |
| 5 | 21,511 | `open com.android.contacts` | 0 | 745 | open: done; change=unsettled (page may still be transitioning); New worker: previous refs are invalid; display=2; snapshotPath=…; snapshotProblem={"code":"scopeUnknown","message":"One complete task window is required; no windows e… |
| 6 | 21,839 | `snapshot` | 0 | 593 | snapshot: done; snapshotPath=… |
| 7 | 22,138 | `click e283703187` | 0 | 1294 | click: done; change=unsettled at windows,selection (page may still be transitioning); snapshotPath=… |
| 8 | 22,653 | `fill e283703220 'R4 Local Contact'` | 0 | 1378 | fill: done; change=changed at windows,content,e283703220:text,showingHintText,actions; fill=done, verification="matches", actualValue="R4 Local Contact"; snapshotPath=… |
| 8 | 22,653 | `click e283703243` | 0 | 1318 | click: done; change=changed at content; snapshotPath=… |
| 9 | 23,667 | `find Notes` | 0 | 1512 | find: done; cached; matches=2; offscreenMatches=2; snapshotPath=… |
| 10 | 24,227 | `scroll-until e283703200 down --max-steps 10 --contains Notes --timeout 15s` | 0 | 1951 | scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":1,"unknown":0,"refs":["e283703300"],"field":"description","matchedText":"Notes","windowId":23}; stop=found; snapshotPath=… |
| 11 | 24,978 | `fill e283703305 'Local validation only'` | 0 | 1623 | fill: done; change=changed at content,e283703305:text,showingHintText,focused,actions; fill=done, verification="matches", actualValue="Local validation only"; snapshotPath=… |
| 11 | 24,978 | `click e283703198` | 0 | 1542 | click: done; change=unsettled (page may still be transitioning); observationWarning="partial; missing=4"; snapshotPath=… |
| 12 | 26,220 | `snapshot` | 0 | 693 | snapshot: done; snapshotPath=… |
| 13 | 26,557 | `back` | 0 | 725 | back: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 14 | 26,896 | `click e283703363` | 0 | 678 | click: done; change=changed at windows,selection; snapshotPath=… |
| 15 | 27,229 | `click e283703383` | 0 | 753 | click: done; change=changed at windows,selection; snapshotPath=… |
| 16 | 27,583 | `click e283703406` | 0 | 478 | click: done; change=changed at windows,selection; snapshotPath=… |
| 17 | 27,915 | `click e283703434` | 0 | 680 | click: done; change=changed at windows,selection; snapshotPath=… |
| 18 | 28,242 | `status` | 0 | 566 | status: done |
| 18 | 28,242 | `session close` | 0 | 863 | session close: done; Session ended; UI cleanup confirmed |
| 18 | 28,242 | `status` | 0 | 449 | status: done |
| 19 | 29,093 | （最终回答） | | | |

#### r6-v1-candidate-03

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,344 | `(读线程号)` | 0 | 38 | 01a2139f-1a64-7a60-b71b-ff5f11a6de8e |
| 2 | 17,476 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 3 | 19,434 | `(读指引 displays.md, reference.md)` | 0 | 13813 | # Main And Work Displays |
| 4 | 23,144 | `session open --display virtual` | 0 | 77 | session open: done; target=virtual/display2; Target established |
| 4 | 23,144 | `apps Contacts` | 0 | 57 | apps: done |
| 5 | 23,387 | `open com.android.contacts` | 0 | 703 | open: done; change=unsettled (page may still be transitioning); New worker: previous refs are invalid; display=2; snapshotPath=… |
| 6 | 23,725 | `click e297196354` | 0 | 1306 | click: done; change=unsettled at windows,selection (page may still be transitioning); snapshotPath=… |
| 7 | 24,252 | `fill e297196387 'R4 Local Contact'` | 0 | 1382 | fill: done; change=changed at content,e297196387:text,showingHintText,actions; fill=done, verification="matches", actualValue="R4 Local Contact"; snapshotPath=… |
| 7 | 24,252 | `click e297196410` | 0 | 1330 | click: done; change=changed at content; snapshotPath=… |
| 8 | 25,291 | `find Notes` | 0 | 1524 | find: done; cached; matches=2; offscreenMatches=2; snapshotPath=… |
| 9 | 25,865 | `scroll-until e297196367 down --max-steps 8 --contains Notes --timeout 15s` | 0 | 1962 | scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":1,"unknown":0,"refs":["e297196467"],"field":"description","matchedText":"Notes","windowId":24}; stop=found; snapshotPath=… |
| 10 | 26,629 | `fill e297196472 'Local validation only'` | 0 | 1635 | fill: done; change=changed at content,e297196472:text,showingHintText,focused,actions; fill=done, verification="matches", actualValue="Local validation only"; snapshotPath=… |
| 10 | 26,629 | `click e297196365` | 0 | 775 | click: done; change=unsettled at windows,selection (page may still be transitioning); snapshotPath=… |
| 11 | 27,589 | `back` | 0 | 736 | back: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 12 | 27,982 | `click e297196529` | 0 | 689 | click: done; change=changed at windows,selection; snapshotPath=… |
| 13 | 28,327 | `click e297196549` | 0 | 765 | click: done; change=changed at windows,selection; snapshotPath=… |
| 14 | 28,695 | `click e297196572` | 0 | 490 | click: done; change=changed at windows,selection; snapshotPath=… |
| 15 | 28,983 | `click e297196600` | 0 | 692 | click: done; change=changed at windows,selection; snapshotPath=… |
| 15 | 28,983 | `snapshot` | 0 | 708 | snapshot: done; snapshotPath=… |
| 16 | 29,625 | `--json status` | 0 | 1060 | {"command":"status","result":"done","data":{"version":"0.4.0","uid":2000,"workerId":"25b4500b-c252-47ce-a675-02dc31517f31","worker":"ready","session":{"name":"r6-v1-candidate-03","pairingName":"（主机名略）","target":{"kind":"v… |
| 16 | 29,625 | `--json session close` | 0 | 1222 | {"command":"session close","result":"done","reason":"Session ended; UI cleanup confirmed","data":{"target":null,"previousSession":"r6-v1-candidate-03","taskDisposition":{"state":"clear","displayId":2,"destroyed":true,"openedTasks"… |
| 16 | 29,625 | `--json status` | 0 | 1000 | {"command":"status","result":"done","data":{"version":"0.4.0","uid":2000,"workerId":"25b4500b-c252-47ce-a675-02dc31517f31","worker":"ready","session":{"name":null,"pairingName":null,"target":null,"idleSeconds":0,"idleRemainingSeco… |
| 17 | 30,899 | （最终回答） | | | |

#### r6-v1-candidate-04

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,344 | `(读线程号)` | 0 | 38 | 01a213aa-1353-72b1-813c-38f535919e6c |
| 2 | 17,471 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 3 | 19,430 | `(读指引 displays.md, reference.md)` | 0 | 13813 | # Main And Work Displays |
| 4 | 23,143 | `session open --display virtual` | 0 | 65 | session open: done; target=virtual/display2; Target established |
| 4 | 23,143 | `apps Contacts` | 0 | 45 | apps: done |
| 5 | 23,335 | `open com.android.contacts` | 0 | 745 | open: done; change=unsettled (page may still be transitioning); New worker: previous refs are invalid; display=2; snapshotPath=…; snapshotProblem={"code":"scopeUnknown","message":"One complete task window is required; no windows e… |
| 6 | 23,648 | `snapshot` | 0 | 593 | snapshot: done; snapshotPath=… |
| 7 | 23,936 | `click e876998461` | 0 | 1294 | click: done; change=unsettled at windows,selection (page may still be transitioning); snapshotPath=… |
| 8 | 24,440 | `fill e876998494 'R4 Local Contact'` | 0 | 1370 | fill: done; change=changed at content,e876998494:text,showingHintText,actions; fill=done, verification="matches", actualValue="R4 Local Contact"; snapshotPath=… |
| 8 | 24,440 | `click e876998517` | 0 | 1318 | click: done; change=changed at content; snapshotPath=… |
| 9 | 25,426 | `scroll-until e876998474 down --max-steps 8 --contains Notes --timeout 15s` | 0 | 1950 | scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":1,"unknown":0,"refs":["e876998574"],"field":"description","matchedText":"Notes","windowId":23}; stop=found; snapshotPath=… |
| 10 | 26,166 | `fill e876998579 'Local validation only'` | 0 | 1623 | fill: done; change=changed at content,e876998579:text,showingHintText,focused,actions; fill=done, verification="matches", actualValue="Local validation only"; snapshotPath=… |
| 10 | 26,166 | `click e876998472` | 0 | 1040 | click: done; change=unsettled at windows,selection (page may still be transitioning); observationWarning="partial; missing=1"; snapshotPath=… |
| 11 | 27,139 | `back` | 0 | 724 | back: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 12 | 27,464 | `click e876998637` | 0 | 678 | click: done; change=changed at windows,selection; snapshotPath=… |
| 13 | 27,828 | `click e876998657` | 0 | 753 | click: done; change=changed at windows,selection; snapshotPath=… |
| 14 | 28,171 | `click e876998681` | 0 | 478 | click: done; change=changed at windows,selection; snapshotPath=… |
| 15 | 28,433 | `click e876998709` | 0 | 680 | click: done; change=changed at windows,selection; snapshotPath=… |
| 16 | 28,752 | `--json status` | 0 | 1047 | {"command":"status","result":"done","data":{"version":"0.4.0","uid":2000,"workerId":"5618ddd6-33ae-425b-a3d0-235aa9668328","worker":"ready","session":{"name":"r6-v1-candidate-04","pairingName":"（主机名略）","target":{"kind":"v… |
| 16 | 28,752 | `--json session close` | 0 | 1210 | {"command":"session close","result":"done","reason":"Session ended; UI cleanup confirmed","data":{"target":null,"previousSession":"r6-v1-candidate-04","taskDisposition":{"state":"clear","displayId":2,"destroyed":true,"openedTasks"… |
| 16 | 28,752 | `--json status` | 0 | 988 | {"command":"status","result":"done","data":{"version":"0.4.0","uid":2000,"workerId":"5618ddd6-33ae-425b-a3d0-235aa9668328","worker":"ready","session":{"name":null,"pairingName":null,"target":null,"idleSeconds":0,"idleRemainingSeco… |
| 17 | 29,948 | （最终回答） | | | |

#### r6-v1-candidate-05

| 回合 | 本回合输入 | 命令 | 退出码 | 字节 | 输出首行 |
| --- | --- | --- | --- | --- | --- |
| 1 | 17,344 | `(读线程号)` | 0 | 38 | 01a213b1-3243-7852-a3dd-018dcabcfaa1 |
| 1 | 17,344 | `(读指引 SKILL.md)` | 0 | 7173 | --- |
| 2 | 19,318 | `(读指引 displays.md, reference.md)` | 0 | 13813 | # Main And Work Displays |
| 3 | 23,098 | `session open --display virtual` | 0 | 65 | session open: done; target=virtual/display2; Target established |
| 3 | 23,098 | `apps Contacts` | 0 | 45 | apps: done |
| 4 | 23,290 | `open com.android.contacts` | 0 | 745 | open: done; change=unsettled (page may still be transitioning); New worker: previous refs are invalid; display=2; snapshotPath=…; snapshotProblem={"code":"scopeUnknown","message":"One complete task window is required; no windows e… |
| 5 | 23,608 | `snapshot` | 0 | 593 | snapshot: done; snapshotPath=… |
| 6 | 23,898 | `click e163171636` | 0 | 1258 | click: done; change=changed at windows,selection; snapshotPath=… |
| 7 | 24,398 | `fill e163171669 'R4 Local Contact'` | 0 | 1378 | fill: done; change=changed at windows,content,e163171669:text,showingHintText,actions; fill=done, verification="matches", actualValue="R4 Local Contact"; snapshotPath=… |
| 7 | 24,398 | `click e163171692` | 0 | 1318 | click: done; change=changed at content; snapshotPath=… |
| 8 | 25,397 | `scroll-until e163171649 down --max-steps 8 --contains Notes --timeout 15s` | 0 | 1950 | scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":1,"unknown":0,"refs":["e163171749"],"field":"description","matchedText":"Notes","windowId":23}; stop=found; snapshotPath=… |
| 9 | 26,140 | `fill e163171754 'Local validation only'` | 0 | 1623 | fill: done; change=changed at content,e163171754:text,showingHintText,focused,actions; fill=done, verification="matches", actualValue="Local validation only"; snapshotPath=… |
| 9 | 26,140 | `click e163171647` | 0 | 763 | click: done; change=unsettled at windows,selection (page may still be transitioning); snapshotPath=… |
| 10 | 27,057 | `back` | 0 | 724 | back: done; change=unsettled (page may still be transitioning); snapshotPath=… |
| 11 | 27,389 | `click e163171812` | 0 | 678 | click: done; change=changed at windows,selection; snapshotPath=… |
| 12 | 27,713 | `click e163171832` | 0 | 753 | click: done; change=changed at windows,selection; snapshotPath=… |
| 13 | 28,107 | `click e163171856` | 0 | 478 | click: done; change=changed at windows,selection; snapshotPath=… |
| 14 | 28,374 | `click e163171884` | 0 | 747 | click: done; change=changed at windows,selection; snapshotPath=… |
| 15 | 28,716 | `status` | 0 | 566 | status: done |
| 15 | 28,716 | `session close` | 0 | 863 | session close: done; Session ended; UI cleanup confirmed |
| 15 | 28,716 | `status` | 0 | 449 | status: done |
| 16 | 29,539 | （最终回答） | | | |
