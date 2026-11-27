# 任务 127：固定四题 30 次尝试逐次读记录

评审对象：`EV/efficiency/` 下 Maps 路线题（`r7-c1-*`，候选 5 + 基线 5）、Chrome Beta 里读 GitHub（`r7-c2-*`，候选 5 + 基线 5）、Feeder 长列表（`r7-s1-core-candidate-*`，5）、后台虚拟屏 Contacts（`r7-v1-candidate-*`，5）。候选是 `82f3a82` 的主机端、APK 和指引，基线是 `7d5be70`。下文 `EV` 指 `S/r8/ev/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`，`PKG` 指实施方材料包。"命令 n" 指该尝试 `cli-index.json` 里的第 n 条 AndroidCtl 命令；"L n" 指 `native.rollout.jsonl` 的行号。我的中间产物（逐次精简会话、命令表）在 `S/agents14/127/tx/`。

## 结论（先读这里）

1. **业务判定：30 次 PASS 我都同意。** 每次最终答复里的业务事实都能在命令输出里找到出处，没有把 A 的值读成 B 的。只有三处不影响业务的小毛病，实施方的判定都没记：GitHub 候选 04 和基线 01 写了"Guido van Rossum"，命令输出里没有这个全名（模型自带知识）；Contacts 候选 02 说被拒那次点击"退出码 1"，实际是 2；Maps 候选 03 第三段没写距离（题面没要求）。判定标准总体合理，Maps 和 GitHub 的"返回"标准偏松但说得过去；**标准缺一条"答复中不得有输出以外的事实"**。
2. **新发现的产品缺陷（实测 + 源码推断）**：GitHub 题 5/5 次候选里，点击结果链接后回执给的是 `change=changed`，没有标"仍在过渡"，但回执里的页面其实还是旧结果页，5 次都只能再补一次 snapshot。我的读法是：click 的比较基线是 REF 所在那次采样，上一条 `fill --submit` 留下的延迟加载被算成了这次点击的变化。
3. **Maps 回执普遍提前收尾**：候选 90 条动作里约 60 条带 `unsettled` 或 "Estimated next sample exceeds the observation deadline"，约 24 条是 partial。5 次候选一共 26 次 snapshot，其中约 19 次是为这些回执补拍的。
4. **Maps 界面树缺内容（需设备）**：National Gallery 地点页上截图看得见的 "Know before you go"、"Hours"、"Directory / Ochre Brasserie" 等文字，在默认界面树里只剩没有名字的 `item eN clickable`，或者整块没有。模型因此 5 次里看了 7 张图。
5. **实施方举的四个正例都核实属实**（Maps C03 旧引用被拒后附新树、GitHub C02 第 31 步拒绝、Feeder"根未恢复须下一条 snapshot"、Contacts 删除确认同命令恢复）。不过整整 30 次里 `ref-repair` 只自然触发了 2 次；`find` 只用了 1 次；`wait`、`reveal`、`--marks`、`batch` 一次都没用。和基线比，旧引用被拒后省掉一轮这件事**没有被证明**：基线 Maps B01 被拒后也是直接用旧树里的另一个引用接着点。
6. **候选省成本的主因**：GitHub 题候选能完整读到 Chrome 网页内容的界面树。基线读不到网页正文，靠 61 张截图加坐标点按完成。Maps 两边差别很小：轮数中位 25 对 27，CLI 中位 21 对 23。
7. **浪费排序**：① Maps 核实起点街道地址并回到地点页，占候选 CLI 的约 66%，这部分由题面造成；② 动作回执提前收尾后补拍 snapshot；③ `scroll-until` 用的文字和页面对不上，加上长命令被宿主轮询切成多轮；④ Feeder 为确认"整卡可见"再滚一次、截两张图；⑤ 读指引，后台屏题每次都读 displays.md，约占 input 的 11%。
8. **这四道题还能不能发现新问题**：能，但收益在下降。本次仍找出第 2、4 条两个新问题；不过回执类新机制在这四题上几乎触发不到，建议换题或加题，专门覆盖 `ref-repair`、`wait`、`find` 和长列表跳读。

---

## 一、逐次业务判定

判定方法：对照题面（题面原文在会话记录里是加密的 `agent_message`，见 L9 一带，读不出来，所以按冻结提示 `pkg9/.../efficiency-preparation.md` 第 76–148 行，加上 `PKG/.../r7-runtime-preparation.md` 第 104 行 Maps 的"街道一级"补充），逐项在命令输出里 grep 最终答复的每个事实。依据种类都是"实测"。

### Maps（题面：两端街道级地址、距离、时长、前三条路段说明，返回 British Museum 地点页，不导航、不用当前位置）

| 尝试 | 判定 | 依据与备注 |
| --- | --- | --- |
| B01–B05 | 同意 PASS | 0.8 mi、18 min、Irving 160 yd、Charing 0.2 mi、Shaftesbury 0.3 mi 都在 XML 树里。起点街道来自建议项 `National Gallery detail="Trafalgar Square, London, UK"` 或地点页 Directions 标签，不是输入框回显。 |
| C01 | 同意 | 起点取自 NG 地点页的 `Directions to The National Gallery, Trafalgar Square`（tx 第 1049 行），步骤见命令 7、8 的回执。 |
| C02 | 同意 | 终页标题 `The British Museum, Great Russell St, London WC1B 3DG...` 在输出里有（tx 第 2019 行）。 |
| C03 | 同意 | 第三段没写 0.3 mi，题面只要求"说明"，不算缺项。 |
| C04、C05 | 同意 | C05 起点写成 "Trafalgar Square, London"，其中 "London" 来自回显或建议项，不影响街道级要求。 |

所有尝试都没有点 "Your location"、Save、Share 或 Start。**标准的松紧**：五次候选和多数基线都是重新搜索 "British Museum London" 回到地点页，不是退回原来那一页（例如 C02、C05 的终页标题已带邮编）。按"回到同一地点的地点页"算可以接受；如果题意是"回到原页面实例"，这个标准偏松。建议明确写成"同一地点的地点页即可"。

### GitHub（题面：编号、状态、维护者 2023 年 5 月关于修复版本和回移的说明，附作者、日期、评论链接，返回同一筛选）

| 尝试 | 判定 | 备注 |
| --- | --- | --- |
| 10 次 | 同意 PASS | #86296、Closed、gvanrossum、`May 24, 2023, 12:10 AM GMT`、`#issuecomment-1560279122`、gh-96764 与 3.11/3.10 的说明，在候选的界面树中逐字存在（例如 C04 tx 第 993–1004 行）；基线来自截图加 XML。 |
| C04、B01 | PASS，但有输出外事实 | 写了"gvanrossum（Guido van Rossum）"。全名在所有命令输出中 0 次出现（`grep -c` 结果为 1，就是 final 自身）。题面要的"作者"已经满足，多出来的全名来自模型记忆，属于轻度编造。 |
| C03 | 同意 | 评论正文来自 `find May --all` 之后对屏外 REF 的 `snapshot e156593835`，是缓存读取，实施方已披露。 |

"同一筛选"：C01 是往地址栏填 URL 回去的，基线也多用 URL，并改成了带引号的 `is:closed "..."`。实施方以"Closed 1、唯一结果 #86296"作判据，可以接受。

### Feeder（题面：使整张目标卡可见并打开，报告标题、官方链接、摘要里的字段/发送者/接收者，返回同一列表，不分享）

5 次都同意 PASS。标题和摘要来自阅读页（例如 C02 命令 6 的回执），URL 来自 WebView 地址栏 `https://www.rfc-editor.org/info/rfc10080/`，C05 取自 Share 预览。"Read/Unread" 取自列表项的 `detail="R4 RFC list | Oct 30, 2026 | Read"`。C05 打开了 Share 面板，没有选择收件人就按 Back 退出。这已经贴近"不分享"的边界，实施方单列了，判 PASS 合理。

### 后台屏 Contacts

5 次都同意 PASS。重新打开后读到 `Note detail="Local validation only"`，删除后出现 `Your contacts list is empty` 和 toast `R4 Local Contact deleted`（C01 删除确认那条 child.stdout）；`session close` 返回 `destroyed=true; openedTasks=1`，其余计数都是 0；之后 `status` 显示 `session=none`。C02 最终答复写"退出码 1"，`cli-index` 记的是 2。原因是这次的 exec 包装没有把 `$LASTEXITCODE` 传出去（L71–L75 两条命令放在同一个 exec 里，宿主看到的是 PowerShell 的 1）。这是调用者没照 SKILL 的"保存并传播 `$LASTEXITCODE`"去做，与业务无关。

**读包提醒**：Contacts C02–C05 的 `session open`、`apps`、`close`、`status`，以及 C02 被拒那条 click，在 `EV` 里没有 child.stdout。材料包按 SHA 共用重复文件（`PKG/EVIDENCE_GUIDE.md` 第 37 行），解包时没有还原；内容在 native 会话里能读到，不算证据缺失。

---

## 二、过程读记录：弯路、非零退出、产品行为

### 2.1 非零退出全表（候选 5 条，基线 4 条）

| 尝试/命令 | 情况 | 回执是否引导出正确下一步 |
| --- | --- | --- |
| Maps C03 命令 4 | `click e412482302` 返回 exit2，`stale`、`timing=ref-repair`，附 `freshRepair=true` 的新树（路线页） | 是。命令 5 直接用新树里的 `e412482400`（Walking 标签），中间没有补 snapshot。 |
| Maps C05 命令 7 | 同上 | 是。命令 8 直接用新引用。 |
| GitHub C02 命令 7 | `scroll-until ... --contains "May 2023"` 返回 exit1：`completedSteps=30`、`stop=routeUnavailable`，第 31 步 `not sent` | 部分是。回执把已执行 30 步和第 31 步被拒分得很清楚，但没有说"没匹配上"的原因。页面上的写法是 "May 24, 2023"，模型随后向上用 "2023" 查 15 步才找到。 |
| GitHub C04 命令 7 | 同类，`stop=maxSteps` | 同上，又用 "May" 向上查一次。 |
| GitHub C05 命令 5 | 同类 | 末次正文里已经有目标评论，模型直接用上了。 |
| Contacts C02 命令 13 | 旧菜单引用被拒，`Ref is absent from the current observation; run snapshot`，`cached`、`timing=before-action`，没有附新树 | 正确拒绝了，但提示"run snapshot"引来一次多余的 snapshot（命令 14）。同一 exec 里刚打印过的上一条回执（命令 12，含 `More options e902992356`）已经够用。 |
| 基线 Maps B01、B03（2 条）、GitHub B03 | 旧引用、`scroll down` 不支持、`--json` 位置写错 | 基线 B01、B03 被拒后都是直接换引用继续，没有补拍。 |

### 2.2 产品行为问题

**P1 确定缺陷：Chrome 链接点击回执过早下"已变化"的结论**
- 位置：GitHub C01 命令 3、C02 命令 4、C03 命令 3、C04 命令 4、C05 命令 3。
- 触发条件：上一条是 `fill --submit`，回执为 unsettled，页面还在加载；接着点击结果链接。
- 可观察的错误：回执是 `click: done; change=changed at content,eX:focused,actions`，没有 unsettled 标记，但 `screen title` 仍是 "Issues · python/cpython"，也就是旧页面。下一条 snapshot 才变成 #86296。把 fill 回执和 click 回执的树做 diff，变化只有 Open/Active 计数补齐（fill 留下的延迟加载）和目标链接 `focused`。
- 读法（源码推断）：`UiPostAction.kt:214–298` 的 `SettledSamples` 用 `before`（click 时就是 REF 所在那次采样）作基线；`usesCurrentBaseline`（`UiPostAction.kt:205–210`）不含 click。所以基线采样之后才到的旧变化被算到点击头上；`onlyNavigationChanges` 只会剔除目标自身的 focused/actions/bounds。
- 影响：5 次各多 1 轮。更大的风险是，信任 `changed` 的调用者会把旧页面当成点击后的结果。
- 建议：click 等动作在派发前取一份新基线（或复用 ref 解析时的新采样）；做不到时至少输出 `baselineAgeMs`，并在基线早于上一动作结束时降为 unsettled。代价：派发前多一次采样，约 0.3–1 秒。验证：用"fill --submit 之后立刻点链接"做一个 JVM 用例，再加一次真实 Chrome 复现（需设备）。

**P2 设计取舍：Maps 上动作回执大量提前收尾**
- 实测：Maps 候选每次 9–16 条回执带 `unsettled`，其中多条附 "Estimated next sample exceeds the observation deadline"（`samplingStopped`，`UiPostAction.kt:112–114,133`）；partial 2–8 条。紧跟其后的 snapshot：C01 6、C02 6、C03 4、C04 3、C05 2。
- 例子：C03 命令 5 点 Walking，回执里是 "Finding best route..."，命令 6 才拿到路线。
- 影响：每次 Maps 尝试多出 2–6 轮。模型从没用过 `wait --contains`，指引也没有把"回执 unsettled 时用 wait 而不是 snapshot"讲清楚。
- 建议（需用户取舍）：A. 维持现状，只在指引里加一句"unsettled 后用 `wait --contains 目标文字`"；B. 在 navigation 类动作上把默认观察期限按设备的实测采样耗时放宽；C. 回执 unsettled 时附一个"建议等待的文字"。推荐 A 加 B 的轻量版。B 的代价是每条动作平均多等约 1–2 秒，需要在 U35 上复测。

**P3 证据不足（需设备）：Maps 地点页界面树缺可见文字**
- 位置：Maps C01 命令 18–21。截图（L132、L149 中解码出的两张 jpg，我放在 `S/agents14/127/img/`）里有 "Know before you go"、3 条要点、"Hours Closed · Opens 10 AM Fri"、"Directory / Ochre Brasserie / The Hunt for the Cheshire Cat"。同次树里只有 `item e141929059 clickable`、`e141929081`、`e141929083`、`e141929088 "Collapsed"`，Directory 列表完全没有。
- 可能原因：节点被报告为 `visible=false`，默认投影把它丢掉（`ui_view.py:384`）；也可能是 Compose 合并语义没有带出文本。
- 影响：模型 5 次候选中有 4 次看了截图。建议用 `snapshot --all` 或原始树在设备上对照确认原因；如果是可见性误判，就在投影里对"在视口内却报 invisible"的节点保留文本。

**P4 可选建议：`scroll-until` 文字失配时没有近似提示，长命令被宿主轮询切成多轮**
- GitHub C02、C04 命令 7 各跑满 30 步（30–40 秒），宿主 exec 10 秒就让出，模型用 `write_stdin` 轮询了 3 次（C05 2 次），合计 8 轮。页面实际文字是 "May 24, 2023"，模型的 "May 2023" 是按子串匹配的，自然匹配不上。
- 建议：失败回执里列出"同一节点内各词都命中"的近似候选（例如 `nearMatches=["May 24, 2023"]`），或支持词集匹配；指引里提醒长 `scroll-until` 会超过宿主默认等待时间。

**P5 可选建议：没有"整卡可见"的事实**
- Feeder 5 次都在 `scroll-until` 命中之后，又截图、滚动、再截图，用 2 张图（C01 用 1 张）确认目标卡完整。原因是树里没有"部分在视口外"的标记（例如 C02 命令 2–5）。
- 建议：对部分越出视口的节点加 `clipped` 属性。这样每次能省 2–3 条 CLI 和 2 张图。

**P6 可选建议：列表项重复的动作标签很占体积**
- Feeder 每行重复 8 个 `action-label`（Toggle read status、Open in reader 等），16 行就是 128 个。可以在同级节点动作集合相同时折叠成一个。

**P7 可选建议：被拒提示措辞**
- Contacts C02 的 `run snapshot` 提示可以改成"使用最近一次观察（capture …:10）的引用，或 run snapshot"，与 ref-repair 分支的措辞保持一致。

核对过确实没问题的地方：
- `fill` 的 `verification="matches"` 每次都和后续页面一致。
- `back` 回执带 `baselineAgeMs`，内容如实。
- 根读取失败（`rootUnavailable`）的回执如实报告，见 Feeder C03、C04 命令 9（`screen ... unreadable`，附 `artifact state="unavailable"`）。
- Contacts 删除确认同命令恢复后，树里直接就是空列表。
- 没有发现"动作后说没变而实际变了"的情况（`noChangeObserved` 没有出现）。

### 2.3 每题的弯路

- **Maps**：候选读完路线时，工具调用分别在第 8–13 次。之后的 10–23 条 CLI 都花在核实起点街道（点 Step 1、搜 National Gallery、滚动、截图）和重新搜 British Museum 返回上。按每次的 CLI 数算，候选一共 118 条，其中约 78 条（66%）属于这一段。原因是路线页只显示 "The National Gallery"，题面要求街道级地址；这部分成本由题面造成，不应算到产品头上。C01、C02 比基线贵（30 轮、36 轮），主要也在这一段（C02 退了 5 次 back）。
- **GitHub**：候选的固定路线是 snapshot、fill 搜索、点击、补拍 snapshot（P1）、定位评论、back。C03 用 `find --all` 加缓存 REF 读取最省（10 条 CLI）。C02、C04 栽在 P4。
- **Feeder**：路线一致；额外成本是 P5 的确认步骤，以及打开菜单后又关掉（C02 命令 7–8）。
- **Contacts**：C01、C03–C05 是 16–17 条 CLI，几乎没有弯路；C02 多出命令 4 和命令 14 两次 snapshot。

另外，每次尝试都固定花 2 轮在读线程 ID 和读 SKILL 上（题面要求）。

---

## 三、候选相对基线：哪些改动真的用上了

| 改动 | 30 次里的实际触发 | 是否省了回合 |
| --- | --- | --- |
| Chrome/WebView 网页内容进树 | GitHub 5/5、Feeder WebView 5/5 | **是，主要收益来源**：基线 GitHub 每次 11–13 张图、19–33 条 CLI；候选 0 张图、6–11 条。 |
| 动作回执直接附界面树 | 所有动作 | Contacts、Feeder 明显省了：Contacts C01、C03–C05 全程 0 次 snapshot。Maps 受 P2 限制，收益有限。注意基线 7d5 也有 `--print` 回执树，所以差别主要在默认就附树和格式更紧凑。 |
| `ref-repair`（旧引用被拒时附新树） | 2 次（Maps C03、C05），属实 | 单看是省了一次 snapshot；但基线 B01 被拒后同样没有补拍，所以相对基线的净收益**未被证明**。 |
| 运行中的 `scroll-until` 前缀与第 31 步拒绝 | GitHub C02，属实 | 回执分得清楚，但没有帮模型找到目标（P4）。 |
| 新-5 根读取字段（实施方的代号，指回执里的 `rootUnavailableObserved`/`rootRecoveredWithinRequest`） | Feeder 2 次（未恢复，模型下一条 snapshot 是对的）；Contacts 5 次删除确认（同命令恢复） | 属实。Feeder 那 2 次就算没有这个字段，`unreadable` 也足以让模型去补拍，字段本身的增量价值很小。 |
| `find`、缓存 REF 读取 | 1 次（GitHub C03） | 是，那次最省。 |
| `wait`、`reveal`、`--marks`、`batch`、`check` | 0 次 | 这四道题覆盖不到。 |

---

## 四、后台屏题专项

- **是否全程在后台屏上操作**：是（实测）。5 次都是先 `session open --display virtual`（C02 回执为 `target=virtual/display2`），`open` 的回执带 `display=2`；之后的命令都不带 `--display`，全部走这个会话；所有 argv 里都没有 main。`close` 回执为 `displayId=2; destroyed=true; openedTasks=1`，独立核验也看到主屏没有 Contacts 任务。不足之处：动作回执的 screen 行不显示 displayId，"每一步都在第 2 屏"只能从会话归属推出来，不能逐条直接看到。
- **SKILL 里那 8 行后台屏要点够不够**：不够。5 次都另外读了 displays.md（约 7 KB），按实施方的估算约占这题 input 的 11%（`PKG/.../core-results.md` 第 343–355 行）。题面要求"依据回执和公开状态核实释放"，这 8 行没讲 `session close` 回执各字段的含义，也没讲 `status` 关闭后应该是什么样子，模型只能去读专题。建议在这 8 行后补一行："`session close` 报告 `destroyed`/`openedTasks`/`unreturnedTasks`，之后 `status` 应为 `session=none`"。估计 5 次能各省约 4 万 input，也就是 10%（这是估算，不是实测）。

---

## 五、汇总：最常见的浪费与消除办法

| 排序 | 浪费 | 估计量（30 次） | 产品可以怎么消除 |
| --- | --- | --- | --- |
| 1 | Maps 核实起点街道并返回地点页 | 候选约 78 条 CLI、50 多轮 | 主要是题面造成的；产品侧只能靠 P3（树缺文字）减少看图。建议题面接受路线页起点或建议项的文字作为街道。 |
| 2 | 回执提前收尾或页面过时后补拍 snapshot | Maps 约 21 次、GitHub 5 次（P1）、Feeder 2 次 | 修 P1 的基线；P2 放宽期限，或在指引里推 `wait`。 |
| 3 | `scroll-until` 文字失配，加长命令被轮询切开 | GitHub 4 条 CLI、8 轮轮询 | P4：近似命中提示；指引说明宿主的等待时间。 |
| 4 | 确认"整卡可见"的截图 | Feeder 约 15 条 CLI、9 张图 | P5：`clipped` 属性。 |
| 5 | 读指引 | Contacts 每次额外约 11%；所有尝试每次 2 轮固定开销 | 第四节建议补的那一行。 |

**这四道题是否还能发现新问题**：能，但边际收益在快速下降。本次仍发现了 P1（回执基线）和 P3（Maps 树缺文字）这类真问题，但这一轮新加的回执机制在这些题上的触发率很低：ref-repair 2/118、root 字段 7/377、`wait` 和 `reveal` 为 0。用这四题的成本中位数去评价这类机制，几乎没有信息量。需要用户取舍：A. 继续固定四题，只作回归；B. 固定四题保留两题（GitHub 和 Contacts，最稳定），另外换两道题专门覆盖 ref-repair、wait、长列表跳读；C. 全部换新。推荐 B。好处是保留历史可比性，又能覆盖新机制；代价是需要准备新题的起点和独立核验。
