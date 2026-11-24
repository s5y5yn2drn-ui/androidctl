# 专题 67：选择器、find／locate／wait／scroll／scroll-until／reveal 这条链

核查对象：产品 `fb75afc`（树与被实测的 `a27fe5aa` 相同）。只读。没有设备、没有跑 Gradle。
记号：`SRC` = 产品只读克隆；`K` = `SRC/android/app/src/main/java/com/rainng/androidctl`；`H` = `SRC/androidctl/src/androidctl`；`PKG` = 实施方材料包；`ST` = `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`；`EFF` = `ST/efficiency`。
任务名称：GitHub 网页任务（Chrome 里查一个 issue 的评论）、Feeder 长列表任务（在 RSS 阅读器列表里找一篇 RFC 并打开）、后台屏 Contacts 任务（在后台虚拟屏上的 Contacts 里填联系人）、Maps 路线任务。
我写的统计脚本在 `S/agents9/67/cmds.py`、`S/agents9/67/count.py`，逐条命令清单在 `S/agents9/67/allcmds.txt`（600 条 CLI 调用，来自 `EFF/*/cli-index.json` 与 `evidence/cli/*/child.stdout`）。

---

## 1. 结论摘要

整体状态：这条链现在**能用、行为基本如实，不构成合入障碍**；但它的规则数量明显多于实际被用到的部分，"省调用"这一收益在真实任务里还没有被证明，有几处可以收窄或合并。

最重要的发现：

1. **`scroll-until --contains` 在用户第 34 号消息（文字出现即成功）之后工作正常**：计分运行里 13 次调用，12 次成功、1 次如实失败（滚到列表末端，所给文字 `May 2023` 本来就不是页面上 `May 24, 2023` 的子串）；首次调用因参数被拒 **0/13**（上一轮同类首调用因时长缺单位被拒 10/10）。返回的引用在 GitHub 任务里被直接拿去点击成功。【原件】【重算】
2. **但它没有证明省了调用**：Feeder 任务候选 5 次全部是"scroll-until 找到 → 再手动 scroll 一次让整张卡片露出"，CLI 中位数 12 对基线 11，模型总输入比值 0.96；GitHub 任务候选的大幅节省来自 `find` 能读到 Chrome 给出的整页屏外文字，scroll-until 只在 2/5 次里出现，其中一次一共滚了 53 步。【原件】【重算】
3. **选择器这一整套（精确 `--role/--name/--text/--id`、唯一性、第 17 题的"有限截断例外"、`locate`、scroll-until 的选择器模式）在 20 次候选计分运行里被模型调用 0 次**；全部 600 条命令里只有基线 Maps 的 2 次 `wait --gone --text`。也没有任何一次输出触到 `nodeLimit`／`treeByteLimit`，截断例外从未被真实任务走到。【重算】
4. **`find` 的"不稳定后重采一次"标志在一个真实案例里漏掉了**：GitHub 任务候选第 1 次点开 issue 后动作报 `changed`（已稳定），随后 `find "May" --all` 读缓存得到 0 条，模型再 `snapshot` 后同一查询得到 4 条。"变化已稳定"不等于"网页已加载完"。【原件】
5. **几处确定的小缺陷**：首个采样不完整时 scroll-until 直接"未发出"而不像滚动之后那样继续等完整采样（字面遵守裁决原文的痕迹）；匹配到的全部引用不设上限地印在首行；`wait` 最后一个采样只要晚于截止就被整个丢弃不评估；文字规范化只把 ASCII `-` 当空格，en dash 不行（实施方自己的冒烟里已出现过一次不匹配）；滚到末端的失败文字仍写成"目标未就绪"，没有达到第 7 题要求的"滚动 N 步后到底"那种直白说法。都是"可选／建议修"，没有必须修。
6. **大小写折叠的 ICU 测试桥值得保留**：生产代码用 Android 自带的 `android.icu.lang.UCharacter.foldCase`（最低 API 34 即 ICU 72），测试端只加了一个 12 行的转调类和一个测试依赖，代价很小，换来主机 `find`（Python `casefold`）与设备端 `wait/scroll-until` 规则一致。

是否构成合入障碍：**否**。建议在合入前或紧随其后处理第 2 节的几条可选项，并请用户就第 5 节两三个收窄方案拍板（主要是删掉 scroll-until 的选择器模式、`find` 的重采规则改为更简单的形式）。

---

## 2. 确定缺陷

没有"必须修"。以下按影响排序，全部为"可选／建议修"。

### D1 `find` 的重采标志会漏掉"动作已稳定但网页仍在加载"（建议修或改设计，见第 5 节 Q2）

- 触发：动作的稳定判定为 `changed` 或 `unchanged`，但页面其实还在增量加载（Chrome 里点开链接很常见：骨架先出现并短暂静止）。此时标志不置位，下一次 `find` 读旧采样。
- 源码：`K/UiSnapshot.kt:98-112`，标志只在 `settle` 不是 `changed/unchanged` 或动作后样本被识别为 loading 时置位。
- 原件：`EFF/r5-c2-final-candidate-01`，第 7 条 `click e222293158` 首行 `change=changed at content`；第 8 条 `find "May" --all` 首行 `cached; sequence=38; ... matches=0`；第 9 条 `snapshot`；第 10 条同一查询 `cached; sequence=39; matches=4`。模型多花 1 次 snapshot 和 1 次 find。
- 影响：`find` 的 0 条结果会被读成"页面上没有"。首行写了 `cached` 和 `age=1497ms`，事实如实，但调用者很难据此判断要不要重采。
- 最小修正方向：要么让 `find` 一律新采（见第 5 节），要么把 Web 内容的 loading 识别补上（后者不可靠，不推荐）。
- 确定程度：原件确认一次；原因（加载未完成）是推断，第二次 snapshot 里出现了第一次没有的评论文字可以支持这一推断。

### D2 scroll-until 首个采样不完整时直接"未发出"，与滚动之后的处理不一致（可选）

- 触发：命令开始时第一份采样不完整（例如 Maps 里约 10% 的采样带 `nodeUnavailable` 或 `collectionLimit` 缺口），且没有可靠正匹配。
- 源码：`K/ScrollProgress.kt:58-63`：`completedSteps > 0` 时停止码为 `observationPending`，主循环会继续等下一份完整采样；`completedSteps == 0` 时停止码为 `observationIncomplete`，循环退出，结果为"未发出"。`K/UiScrollUntil.kt:78-86`。
- 影响：同一个"树暂时不完整"的状况，滚过一步就耐心等，没滚就立即放弃；调用者拿到退出码 2，要自己重发。测试 `UiScrollUntilTest.kt:233` 把这一行为固定了下来。
- 来源：第 7 题原文写的是"**滚动后**界面树不完整时，在下一次采样完整后继续"。这是字面遵守裁决的痕迹，初始状态同样适用这条理由（还没滚就更不该滚）。
- 修正方向：初始不完整也进入 `observationPending`，复用 `recoverObservation()`。改动在一行判断内。
- 确定程度：源码确定；真实任务里没有出现过（因为 13 次调用的首采样都完整或已有正匹配）。

### D3 匹配引用不设上限地印在首行（可选，影响调用成本）

- 触发：`wait --contains` 或 `scroll-until --contains` 用了一个常见短词，在长网页上命中几百个节点。
- 源码：`H/ui_output.py:153-154` 把整个 `predicate` 原样 `json.dumps` 进首行，包括 `refs` 数组和完整 `matchedText`。设备端 `K/UiReceipt.kt:44-75` 只在回复超过 192 KiB 时才摘要 `matchedText`，`refs` 从不裁剪（实施方 U34 补丁明确"全部 refs 不裁剪"）。
- 影响：首行可以长达几十 KB，全部进入模型上下文。真实运行里 `May` 只命中 3 个，没有出现；但 `matchedText` 已经把 Feeder 一整张卡片（标题加摘要约 250 字）印在首行（`EFF/r5-s1-core-candidate-04` 的 scroll-until 回执）。
- 修正方向：首行只印前若干个 refs 加总数、`matchedText` 截到固定长度；完整内容留在 `--json`。
- 确定程度：源码确定；成本影响是推断。

### D4 `wait` 晚于截止的那份采样不做评估（可选）

- 触发：默认 5 秒预算内，最后一份（常常也是唯一一份）采样在截止之后才完成。
- 源码：`K/UiCondition.kt:344-346`、`383-385`：预算为零直接返回"Observation failed: Condition observation completed after the request cutoff"，不评估条件；但这份采样照样作为当前页交付给调用者。
- 原件：`EFF/r5-c2-final-candidate-03` 第 3 条 `wait --contains "rare race"`，首行 `failed; Observation failed: ...; observationWarning="partial; missing=1"`，没有 `match=`。这次文字确实不在页面上（我在交付的正文文件里搜索为 0 次），所以结果正确；但若文字已在页面上，调用者会看到"失败"且正文里就有那段文字。
- 影响：结论自相矛盾的风险；文字"Observation failed"也把"没等到"说成了"观察失败"。
- 修正方向：对正向条件（`--contains`、选择器、`--app`）评估这份迟到的采样，命中即成功；文案改为"条件在截止前未满足"。
- 确定程度：源码确定；矛盾情形未在原件里出现。

### D5 文字规范化只把 ASCII 连字符当空格（可选）

- 源码：`K/UiCondition.kt:241-248` 与 `H/ui_view.py:380-383` 都只 `replace('-', ' ')`。
- 【实跑】用 `S/r6/venv6/bin/python` 调主机 `folded`：`"ScreenTinker - open"` 不是 `"ScreenTinker – open-source"`（en dash）的子串；`"strasse"` 能匹配 `"Straße"`，`"Wi-Fi"` 能匹配 `"WI FI"`。
- 原件：实施方的长列表冒烟记录 `ST/real-tasks.md:856` 自己记了"ASCII 标题连接符 `-` 不匹配实际 en dash `–`"。
- 修正方向：两端都把 Unicode 破折号类（`Pd`）映射为空格。各一行。

### D6 滚到末端的失败文字仍是"目标未就绪"（可选；与第 7 题文字要求部分不符）

- 原件：`EFF/r5-c2-final-candidate-04` 第二次 scroll-until 首行：`After scrolling 30 steps, the next scroll was not sent: Original target is not ready for scroll-down: target has no advertised route for scroll-down; current scroll directions: backward, up, page-up; use androidctl scroll e576924742 backward; ...`，`stop=routeUnavailable`。
- 事实是如实的（完成 30 步、容器不再公布向下滚动），但第 7 题要求的是"滚动 N 步后到底，没有节点匹配"这种直白说法；现在的文字说"未就绪"，还建议"scroll backward"，对一次搜索来说是误导性建议。
- 源码：`K/UiActions.kt:1007-1011` 把 scroll-until 的"无路线"复用为普通动作的未就绪文案；`K/ScrollProgress.kt:142-152` 原样拼接。
- 修正方向：scroll-until 遇到容器不再公布该方向时单独写"After N steps the container no longer offers DIRECTION (likely its end); no match"。

### D7 滚动后容器丢失时，停止码不是 `changed`（可选，文档与实现不一致）

- 源码：初始检查时容器变了，`failure()` 走最后一支，停止码为 `changed`（`K/ScrollProgress.kt:153-159`）；滚过一步之后容器变了，`failure()` 走第一支，停止码为 `finalCheckUnavailable`（`:127-139`），原因文字里才含 "Original container is absent"。`skills/androidctl/actions.md:89-90` 说调用者应读到"changed container"这一停止原因。
- 影响：调用者按停止码分支时会漏掉这种情况。轻微。

### D8 截断例外有一个已知的不严密处（可选，低概率）

- 源码：采集队列按"可见优先、再按深度"排序（`K/TreeCollector.kt:62`），不可见节点的子节点只有在该不可见节点出队时才入队。若节点上限在此之前用完，"不可见父节点下的可见子节点"会丢失，而缺口的所属节点不可见，`K/UiSelector.kt:107-120` 会接受这个缺口，于是唯一性可能被错误确认。指引 `actions.md:31` 自己写了"Visible descendants of invisible parents still participate"，说明这类节点确实存在（Chrome 的虚拟节点常见）。
- 影响：只在"长页面触到上限 + 恰好有同名第二目标藏在不可见容器下"时误选。真实任务里从未触到上限。
- 修正方向：缺口所属节点不可见、但其祖先链上有可见节点时也拒绝；或者干脆接受这一残余风险并在内部文档写明。

### 其它我查过、没有发现问题的点

- **步数上限不会多滚**：`completedSteps` 只在回执为 done 时加一（`K/ScrollProgress.kt:104-108`），主循环在等于 `maxSteps` 时停（`K/UiScrollUntil.kt:82-83`），最后一步之后先检查匹配再判断上限。
- **已滚步数不漏报**：失败、UNKNOWN、平台拒绝、截止各分支都保留 `completedSteps` 和当前 `step`（`K/ScrollProgress.kt:98-172`）；原件里 30 步失败的回执也保留了第 31 步"未发出"的完整记录。
- **到底判定**：只靠平台不再公布该方向或平台返回失败，不用"内容没变"推断末端（`actions.md:87`）。GitHub 运行里 30 步后正确停在页尾。没有发现误判；代价是对"一直返回成功却不动"的容器只能耗尽步数或时间，这属于设计取舍，不是缺陷。
- **`page-*` 方向**：主机与设备用同一组 10 个方向；`page-*` 只走公布的翻页动作，物理方向可回退到翻页动作或已知类的通用前进/后退（`K/UiScroll.kt:20-38`）；scroll 与 scroll-until 一致。
- **正匹配优先**：`wait` 先算命中再看"连续两次同类截断"（`K/UiCondition.kt:406-433`）；`scroll-until --contains` 先看 `matches > 0` 再看完整性（`K/ScrollProgress.kt:55-63`）。
- **`--gone` 要求完整**、显式单窗口连续两次缺席判"窗口缺席"、成功只交匹配及祖先（`H/ui_view.py:386-414`）、失败交当前页：实现与 `observations.md:121-165` 一致。

---

## 3. 设计与简化

### 3.1 七个入口的语义对照

| 命令 | 读哪份采样 | 匹配规则 | 要求可见 | 要求唯一 | 采样不完整时 | 成功返回 | 失败返回 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `find TEXT` | 默认读缓存；上一个动作被判为不稳定或 loading 时先新采一次（只一次） | 主机端：NFKC、完整大小写折叠、`-` 当空格、空白合并后做子串匹配；字段为 name（可含子孙文字）、value、detail、stateDescription、资源 ID | 否，屏外节点也算，标 `offscreen` | 否 | 照常返回，事实里标 partial | 最深的匹配加祖先，默认 20 条 | 0 条时也是 done，退出码 0 |
| `locate 选择器 --action A` | 每轮新采，直到截止 | 精确选择器 | 是 | 是，且要求可直接执行动作 A | 非截断类不完整：继续等直到截止；只是节点/字节上限截断：没有可见节点缺子节点时照常判断，否则第一份就失败 | `ref` | not ready／ambiguous 等，附候选 |
| `wait --contains` | 每轮新采 | 设备端，与 find 同一种规范化，但字段只有 text、description、stateDescription，排除提示文字，**不含资源 ID 和合成 name** | 是 | 否 | 可靠正匹配先成功；否则继续等；连续两份同类上限截断提前失败 | 匹配节点及祖先，`predicate.refs` 全部列出 | 当前页 |
| `wait 选择器`（`--role/--name/--text/--id`） | 同上 | 精确（相等），`--id` 可用 `:id/` 后缀，接受一次 XML 实体解码 | 是 | **否**，至少一个即可 | 同上 | 同上 | 同上 |
| `wait --gone 选择器` | 同上 | 精确选择器（**不能配 `--contains`**） | 只数可见节点 | — | 必须完整且零命中、零未知；显式单窗口连续两次缺席也算 | 状态 | 当前页 |
| `wait --app／--idle／--screen-change／--any-of` | 同上 | 包名相等／两份完整且相同的树／与已发布观察比较／多条件取第一个命中 | — | — | `--idle`、`--screen-change` 需要完整 | 状态（非节点条件不交正文） | 当前页 |
| `scroll REF [方向]` | 动作前读原容器新事实 | — | — | — | — | 动作后观察 | 未发出时给可照抄方向 |
| `scroll-until REF 方向 --max-steps N --contains` | 每步后新采，只看原容器的严格后代 | 同 `wait --contains` | 是 | 否 | 有可靠正匹配就成功；没有时首采样不完整即未发出，滚过之后等下一份完整采样 | 默认整页正文；首行 `refs`、首个 `matchedText`、`completedSteps`、`stop` | 当前页加停止原因、已滚步数、末步回执 |
| `scroll-until … 选择器` | 同上 | 精确选择器 | 是 | **是**，且身份可用，返回单个 `predicate.ref` | 同上；完整且多于一个时报 ambiguous | 同上 | 同上 |
| `reveal REF` | 动作前读原目标 | — | — | — | — | 发一次 SHOW_ON_SCREEN，回读原目标是否可见 | 目标未公布该动作时未发出 |
| `snapshot --cached [REF]` | 只读缓存，不重采，不消耗 find 的重采标志 | — | — | — | — | 同一份采样 | 无缓存时报错 |

### 3.2 调用者容易混淆的地方（都有源码依据）

1. **`find` 能找到，`wait --contains` 却等不到**：`find` 搜屏外节点、合成 name（可能来自不可见子孙）和资源 ID 后缀（`H/ui_view.py:417-433`）；`wait --contains` 只搜可见节点的三个字段（`K/UiCondition.kt:223-239`）。`find Settings` 命中一个 `id=".../settings_button"` 的图标后，`wait --contains Settings` 可以永远等不到。指引 `observations.md:19-24` 和 `:135-139` 分别写了两套字段，但没有把差别并列出来。
2. **同一个词在三处是三种意思**：`--text` 在 `wait`／`locate`／`click`／`scroll-until` 里是"节点自身文字完全相等"；`--contains` 是规范化子串；`find` 的位置参数又是第三套字段的子串。上一轮 Feeder 消融实验里模型 5 次用 `--text 标题` 精确匹配失败，原因是节点文字是"标题\n摘要"合并值（`EFF/core-results.md:114-129`）。这一轮指引把 `--contains` 放在示例首位后，模型 13/13 都改用 `--contains`，问题消失。
3. **`wait 选择器` 与 `locate` 的区别**：前者"至少一个可见匹配"，后者"唯一、可见、启用、可执行该动作"。两者参数几乎一样，结果语义完全不同。
4. **`scroll-until --contains` 成功不等于目标可操作**：返回的是"看见了"的节点，可能只露出一部分。Feeder 任务 5/5 次在成功后又手动 `scroll` 一次才让整张卡片露出（任务要求卡片完整可见；模型的说明原文见 `EFF/r5-s1-core-candidate-0{1,2,3,5}/native.rollout.jsonl` 的 assistant 消息"已使完整目标卡可见"）。另一方面 GitHub 候选第 5 次的 `click e657930845` 因目标不可见被拒，scroll-until 找到后同一引用点击成功，说明引用在多数情况下可用。
5. **几乎每次 scroll-until 成功都带"page may still be transitioning"**：12/12 次成功的首行都有 `change=unsettled`。原因是找到后立刻停止观察（`K/UiPostAction.kt:46` 的 `if (!keepObserving) break`），没有攒够两份相同采样。这条提示因此对 scroll-until 几乎是常量，降低了它的信号价值，同时会让下一次 `find` 必定重采。

### 3.3 命令数量与可合并之处

真实使用（20 次候选计分运行、297 条 CLI）：`find` 15、`scroll` 46、`scroll-until` 13、`wait` 1、`locate` 0、`reveal` 0、任何选择器参数 0。【重算】基线 15 次运行里选择器只出现在 2 次 `wait --gone --text`。我不以使用次数决定去留，但这些数字说明：调用者（LLM）的主路径是"看观察 → 用 REF"，选择器这条路基本是给脚本式用法留的。

可以合并或收窄、而不丢能力的方案（会改命令界面，列入第 5 节由用户定）：

- **删掉 scroll-until 的选择器模式，只留 `--contains`。** 选择器模式承担"完整 + 唯一 + 身份可用的规范目标"整套判定（`K/ScrollContainer.kt:87-113`、`ambiguous` 停止分支、角色校验 `K/UiScrollUntil.kt:52-60`），本轮 0 次使用，上一轮多次因精确匹配合并文字而失败。需要可操作引用时，调用者拿 `--contains` 找到后再用 REF 或选择器动作，后者本身就做唯一性和就绪检查。得：少一种模式、少一个停止原因、指引少一段；失：一次调用里直接得到唯一 REF 的便利。
- **`locate` 并入 `wait`**：例如 `wait --role button --name Save --ready click`。得：少一个命令、"等待"只有一个入口；失：无（能力等价），代价是改名。也可以直接删 `locate`：所有选择器动作（`click --name Save`）本来就先等唯一就绪目标（`K/UiActions.kt:892-937`），`locate` 只多了"不执行"。本轮 0 次使用。
- **`wait --gone` 允许 `--contains`**：现在"等 Loading 消失"只能写精确文字（基线 Maps 的 `--text "Finding best route..."`），与正向等待的规范化子串不对称。改动小，纯增益。
- **`scroll-until` 并入 `scroll --until-contains TEXT --max-steps N`**：功能等价，方向、路线、超时规则本来就一样。得：少一个命令名；失：一次改名成本。我不强推，现在的名字也清楚。
- `reveal`：代码只有约 40 行（`K/UiActions.kt:107-169`），与 `action --action-id` 共用路径，保留无妨；但本轮 0 次使用，上一轮在 Feeder 上 3 次都是"未公布该动作"。

### 3.4 关于第 17 题"有限截断例外"

- 规则原文（`actions.md:30-34`）：只因节点数或字节上限截断、且没有可见节点缺子节点时，普通选择器可按唯一性执行；可见节点缺子节点时第一份截断采样就失败；不放宽 `gone`、`idle`、screen-change 基线和引用修复。
- 实现（`K/UiSelector.kt:88-120`、`K/UiActions.kt:910-921`）与原文一致。
- 是否易懂：对调用者几乎没有可操作的信息——调用者不能控制截断，看到的只是成功或一条"Observation hit nodeLimit before selector uniqueness could be established"的报错。可是这条规则在公开指引里出现了三次（`SKILL.md:77`、`observations.md:89-91`、`actions.md:30-34`）。
- 原件：本轮 600 条命令里 0 次出现 `nodeLimit`／`treeByteLimit`，选择器动作 0 次。没有误拒或误选的原件。
- 不一致：它放宽的理由（采集可见优先，选择器只数可见节点）对 `wait --gone` 同样成立，`gone` 也只数可见节点（`K/UiSelector.kt:54`），却不放宽；对 `wait --contains` 而言，"连续两次同类截断提前失败"（`K/UiCondition.kt:424-432`）在可见优先采集下也没有必要——可见文字不会因为屏外节点被截掉而看不到。三条规则用三种标准对待同一种截断。
- 判断：这是一个为极端情形准备、长期要维护文档的规则，价值有限。代码已写好且有测试，删掉省不了多少；建议**把它从公开指引里拿掉**，只保留报错；内部统一成一条：截断只影响"证明不存在"类判断（`gone`、`idle`、screen-change），不影响"看到了"类判断和只数可见节点的唯一性。

### 3.5 关于第 6 题第一处（`find` 重采）

- 生命周期（`K/UiSnapshot.kt:23,59,91-96,98-112,130`）：动作可能已派发时，按本动作的 `settle` 和动作后样本是否 loading 置位；安装任何新采样或清空快照时清除；`find` 读到标志时把请求改为新采，范围取当前观察的 `selectionScope`；新采失败则标志保留；未派发的拒绝不改标志；`snapshot --cached` 不消耗标志。
- 作用域是整个会话的一份快照，不分窗口；重采沿用当前观察的范围，不会悄悄换范围。实现与实施方说明 `ST/r5-unsettled-find.md` 一致。
- 漏重采：见 D1。多重采：scroll-until 成功几乎必然置位（3.2 第 5 条），所以 scroll-until 之后的第一次 `find` 必定重采，这本身无害。
- 调用者看到的提示：动作首行的"page may still be transitioning"，`find` 首行有无 `cached`。
- 判断：这是为"不改观察算法"这一前提打的补丁，引入了一个跨命令的隐藏状态和三段指引。更简单的形式是"`find` 总是新采，要读旧的就用 `snapshot --cached` 加主机端搜索文件"，代价是每次 `find` 多一次设备采样（真实运行里一次采样约 1–3 秒，不增加模型 token）。列为第 5 节 Q2。

### 3.6 文字匹配实现（ICU）

- 设备端实际用的是 Android 平台自带的 `android.icu.lang.UCharacter.foldCase(…, true)`（`K/UiCondition.kt:241-248`），即 Unicode 完整大小写折叠；最低 API 34（`SRC/android/app/build.gradle.kts:72`），对应平台 ICU 72。
- JVM 单元测试里 `android.jar` 的 ICU 类是空桩，所以实施方加了测试依赖 `icu4j 72.1`（`SRC/android/gradle/libs.versions.toml:9-10,22`、`app/build.gradle.kts:268`）和测试源码里一个同名转调类（`SRC/android/app/src/test/java/android/icu/lang/UCharacter.kt`，12 行）。生产代码没有注入点。
- 值不值得：值得。代价是一个只在测试类路径上的依赖和 12 行代码；收益是主机 `find`（Python `str.casefold`）与设备端规则一致，而且生产算法被真实执行。更简单的替代是生产代码改用 `lowercase(Locale.ROOT)`，则可以完全不要测试依赖；差别只在 `ß/ss`、希腊词尾 sigma 等少数字符（我用主机实跑确认 `"strasse"` 能匹配 `"Straße"` 依赖的正是完整折叠）。两种都可以，现状不需要改。
- 一个小的残余差异【推断】：空白切分用正则 `[\s\u001C-\u001F]+`，JVM 测试跑的是 OpenJDK 正则，设备上跑的是 Android 基于 ICU 的正则，两者对 `\s` 的定义未必相同；Python `split()` 还包括 U+0085、U+2028/2029 等。经 NFKC 后绝大多数特殊空白已变成普通空格，剩余差异只影响极少数字符。【未能核实】Android 正则 `\s` 的确切集合。

### 3.7 这一块的规模与价值

- 规模：设备端直接相关约 1,300 行（`UiCondition.kt` 502、`UiSelector.kt` 189、`ScrollProgress.kt` 185、`ScrollContainer.kt` 142、`UiScrollUntil.kt` 142、`UiScroll.kt` 63、`UiActions.kt` 选择与等待约 70、`UiSnapshot.kt` 标志约 20），JVM 测试约 1,560 行，主机约 400 行，公开指引约 60 行。
- `ScrollProgress` 有十几个停止码（found、notFound、observationPending、observationIncomplete、ambiguous、admissionPending、platformPending、checking、platformUnknown、platformRejected、finalCheckUnavailable、maxSteps、budgetExhausted，再加 routeUnavailable、changed 等透传码），其中一半是中途被打断时保留 UNKNOWN 用的过渡态。指引只列了五类。这是"已执行前缀如实保留"决定的必要成本，我认为可以接受，但调用者可见的停止码应收敛到：找到、步数用完、到末端、观察不完整、截止、容器变了、结果未知。
- 判断：能力本身有效（scroll-until 在 GitHub 长网页上一次调用滚 6–30 步并正确停止；在 Feeder 和 Contacts 上每次只需 2 步），保护也真实（原容器绑定、不点击、前缀保留）。过重的部分是选择器分支和若干为极端情形写的规则。**不需要整块重做**；建议按 3.3、3.4、3.5 收窄。

### 3.8 前几轮决定中现在看应调整的

- **第 7 题"语义沿用 `wait --contains`"写得不够清楚**，直接导致实施方先做了"唯一规范目标"版本、真实任务里出现 `stop=ambiguous`（`EFF/r5-c2-candidate-01`，同一条评论的标题、日期链接、日期文字三个节点被判为多目标），再由用户第 34 号消息纠正。评审方第五轮的建议是这句话的来源之一（实施方分歧记录 `ST/r5-scroll-contains-triage.md` 引用了评审方专题 56 的括注）。教训：给"沿用某语义"类建议时要写明成功条件的基数。
- **第 7 题"滚动**后**不完整则等完整"**的字面写法造成了 D2 的不对称。
- **第 17 题保留"`wait` 连续两次同类截断提前失败"**：在可见优先采集下，这条对正向条件没有理由，建议只对需要完整树的条件保留（第 5 节 Q3）。
- **默认 5 秒预算**对 `wait` 偏短：GitHub 页面一次采样就可能吃掉大半预算（D4 的原件）。模型在 scroll-until 上已普遍显式给 `--timeout 15s`～`60s`，`wait` 上没有。可以考虑把 `wait` 默认提到 10 秒；这是实施细节。

---

## 4. 证据审计

| 实施方或材料的声称 | 核对结果 |
| --- | --- |
| scroll-until 的 contains 在至少一个可靠可见匹配时成功，包括不完整采样（`ST/u34-contains.md`） | **成立**。源码 `K/ScrollProgress.kt:55-56`、`K/ScrollContainer.kt:27-52`；原件 `EFF/r5-s1-core-candidate-01` 首行 `done; ... "complete":false,"matches":1`，带 `partial; missing=1`。 |
| 精确选择器仍要求完整、唯一、可见、身份可用 | **成立**（`K/ScrollContainer.kt:87-113`）。真实任务里 0 次使用，因此只有 JVM 测试支撑。 |
| 已执行前缀、末步回执、UNKNOWN 如实保留 | **成立**（源码）。原件里有"30 步后第 31 步未发出"的完整回执；UNKNOWN 分支没有真实原件，只有测试 `UiScrollUntilTest.kt:257`。 |
| 第 7 题要求的 50 项以上长列表冒烟已完成（`ST/real-tasks.md:856`） | **部分成立**。列表确有 100 行，但目标在第 13 行，2 步即找到，没有真正考验长距离滚动。更强的证据其实来自计分运行：GitHub 候选第 4 次单次调用滚 30 步到页尾并正确停止，第 5 次 6 步找到。 |
| 首调用参数被拒问题已解决 | **成立**。本轮 600 条命令里 `Invalid value for '--timeout'` 出现 0 次；13 次 scroll-until 首调用全部被接受。改善来自指引示例写了 `--timeout 15s`，不是放宽了参数规则（`H/cli_options.py:148-160` 仍要求单位，我实跑 `--timeout 15` 仍被拒）。 |
| Feeder 候选与沿用基线的模型总输入比值 0.96，不能归因于 scroll-until（`EFF/core-results.md:192,687-691`） | **成立**，并且我从命令序列补充了原因：候选 5/5 次在 scroll-until 之后又 `scroll` 一次；基线有 2/5 次用 App 自带搜索直接找到，0–2 次滚动。实施方没有把这个比值说成 scroll-until 的收益，表述克制。 |
| GitHub 任务候选 CLI 中位数 12 对基线 20 | **数字成立**（我重算一致），但节省主要来自 `find` 读整页屏外文字（候选第 1–3 次没有用 scroll-until）。实施方在 `core-results.md:420` 也写明了第 4 次三次 scroll-until 共 53 步、CLI 少不代表设备动作少。 |
| `wait` 失败如实保留 | **成立**。GitHub 候选第 3 次的 `wait` 失败被计入成本、未被重试或删除（`core-results.md:424`）。但文案"Observation failed"不准确（D4）。 |
| ICU 桥"不改生产算法、不引入注入点"（`ST/u34-contains.md`） | **成立**。测试类只做转调；生产代码无测试钩子。首次 JVM 失败、Java 桥被拒等过程原件也如实保留了。 |
| 测试是在验证行为还是复述实现 | 大体在验证行为（例如"最后一步也检查匹配""重复完整样本不宣布到底""首采样不完整即未发出"）。但最后这一条把 D2 的不对称当作期望固定了下来；`containsCannotTreatUnreadOrTruncatedFieldsAsACompleteNegativeSample` 固定了"被截断的长文字前缀即使含目标也不算匹配"，这一点是可讨论的设计，而不是唯一正确行为。 |
| 单次成功是否被说成普遍结论 | 没有发现。实施方对 Maps 方向拒绝、Feeder 消融未达效率门都如实写了。 |

补充统计（全部【重算】，脚本见开头）：

- 计分运行里 scroll-until 13 次：Feeder 5 次全部成功，每次都是 2 步；后台屏 Contacts 4 次全部成功，每次也是 2 步（另有 1 次运行改用两次 `scroll`）；GitHub 4 次，分别 12 步成功、30 步到末端失败、11 步成功、6 步成功。也就是说 13 次里 9 次只滚了 2 步，真正的长距离滚动只出现在 GitHub 网页上。失败原因分布：到末端 1。基线同一任务的替代做法：Feeder 基线用 App 搜索框或 0–2 次 `scroll`；GitHub 基线用 1 次 `scroll` 加 1 次 `swipe`，主要靠多次 `snapshot`。
- `wait`：候选 1 次（失败，文字确实不在）；基线 3 次（`gone` 成功 1、`gone` 因 Maps 采样连续不完整 40 秒失败 1、`contains` 成功 1）。`wait` 不是模型的常用工具，模型主要依赖动作后的观察。
- `find`：候选 15 次，其中 9 次在不稳定动作后自动新采、6 次读缓存；读缓存的 6 次里 1 次给出了假阴性（D1）。
- 方向拒绝（与本专题相邻）：Maps 候选 5 次里有 3 次请求 `down`，而列表（`recycler_view`，`rows=0 columns=0`）只公布 `forward`，被拒后按建议改用 `forward`；另 2 次在列表顶端请求 `up` 被拒，这是正确拒绝。物理方向只对 `ScrollView`／`GridView`／单列集合回退到通用前进后退（`K/UiScroll.kt:40-62`），是有意的保守选择，代价是每次运行约 1 次调用。

---

## 5. 需要用户裁决的新取舍

### Q1 scroll-until 是否只保留 `--contains`

- A：保持两种模式（现状）。
- B：删掉选择器模式，只留 `--contains`；需要唯一可操作目标时，调用者找到后用 REF 或选择器动作。
- C：保留选择器模式，但不再要求唯一（与 `wait` 选择器一致，返回全部匹配）。
- 推荐 **B**。理由：本轮 0 次使用，上一轮选择器模式是失败的主要来源；删掉后少一种成功条件、少一个停止原因、少一段指引，`ScrollContainer.predicate` 与角色校验可以删除。失去的只是"一次调用直接拿到唯一 REF"，后续动作本身就做这个检查。剩余未知：脚本式用户是否依赖它（目前没有外部用户的证据）。

### Q2 `find` 读缓存还是新采

- A：现状（读缓存，不稳定或 loading 后重采一次）。
- B：`find` 一律新采；要读旧采样用 `snapshot --cached`（或给 `find` 加 `--cached`）。
- C：现状，但把动作后 `changed` 的网页（Chrome/WebView 窗口）也视为需要重采。
- 推荐 **B**。理由：去掉一个跨命令的隐藏标志和三段指引，消除 D1 那类假阴性；代价是每次 `find` 多一次设备采样（约 1–3 秒，不增加模型 token）。C 是更窄的补丁，但还是要靠识别"网页仍在加载"，可靠性未知。剩余未知：新采是否会让模型此前看到的 REF 失效——按 `actions.md:7-14`，同一节点在新采样里保持同一 REF，应当不会，【需设备】确认。

### Q3 截断与"证明不存在"的规则统一

- A：现状（选择器唯一性有截断例外；`gone` 不放宽；`wait` 正向条件连续两次截断提前失败）。
- B：统一为一条：节点或字节上限截断只影响需要完整树的判断（`gone`、`idle`、screen-change）；正向等待不因截断提前失败；只数可见节点的判断（选择器唯一性、`gone`）在"没有可见节点缺子节点"时都可以成立。公开指引只留报错，不再讲例外。
- C：B 的前半（正向等待不提前失败、指引删例外），`gone` 维持严格。
- 推荐 **C**，其次 B。C 只删规则、不新增放宽，风险最小；B 让 `gone` 在长网页上可用，但要接受 D8 那种残余误判。剩余未知：真实任务里从未触到上限，取舍的实际影响目前只能推断。

### Q4（可选）`locate` 是否保留为独立命令

- A：保留。B：并入 `wait`（例如 `wait 选择器 --ready ACTION`）。C：删除，依赖选择器动作自带的等待。
- 推荐 A 或 C，看用户是否在意"不执行地探测就绪"这一用法；B 增加 `wait` 的参数，不一定更简单。本轮 0 次使用，维护成本很小（`K/UiActions.kt:19-40` 加共用的 `selected()`）。这一项优先级低，可以不在本轮定。

属于实施细节、不需要用户定的：D2–D7 的修正、`wait --gone --contains`、Unicode 破折号规范化、首行引用条数上限、`wait` 默认预算、scroll-until 停止码收敛。

---

## 6. 有判别价值的补验

1. **在 GitHub 长 issue 页上，点开后立即 `find` 与先 `snapshot` 再 `find` 的对照，各 5 次**（【需设备】）。若缓存读取的假阴性频繁出现，Q2 应选 B；若只是偶发，A 或 C 也可接受。
2. **`find` 一律新采的时间成本**：在三类页面（Feeder、Contacts、GitHub）各测 10 次 `find` 新采耗时。若中位数超过 3 秒，Q2 的 B 要配 `--cached` 开关。
3. **初始不完整的 scroll-until**：在 Maps 列表上连发 10 次 `scroll-until … --contains`，统计"未发出（observationIncomplete）"的比例。若明显大于 0，D2 应当修。
4. **触到节点上限的长网页**（例如上千条评论的 issue）上跑一次 `wait --contains 屏内可见文字`：若在 2 份采样后提前失败而文字其实可见，Q3 应至少选 C。

---

## 7. 没能核实的内容

- 所有设备行为都只来自模拟器原件；没有真机或厂商 ROM 的证据。
- Android 上 `java.util.regex` 对 `\s` 的确切字符集，与 JVM 测试是否一致。
- `--contains` 命中节点后引用"不可操作"的比例：只有 GitHub 一次正面例子（点击成功）和 Feeder 五次"露出不完整需再滚"的例子，没有系统统计。
- 容器在滚动途中被替换（例如 Compose 重建、WebView 重载）时的真实行为：只有 JVM 测试，没有原件。
- Feeder 任务的原始任务文字不在原生记录的可读部分里；"任务要求整张卡片可见"是根据五次运行的模型说明推断的。
- 不可见父节点下可见子节点在真实页面里的出现频率（D8 的实际风险）。
