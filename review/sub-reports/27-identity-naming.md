# 专题 27：元素引用（REF）的身份、名字与可点目标规则——独立核查

核查者：只读独立核查。源码 `SRC`=候选 `3f45af6`（基线 `7d5be70` 用 `git show/diff` 对照）。自写脚本全部在
`/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents6/27/`（`cmp.py`、`freq3.py`、`recycle.py` 及输出 `freq3-out.txt`），用 `python3 -I` 运行，包内原件只作参数读入。没有设备，所有运行事实来自实施方保存的 MuMu／API34 模拟器原件。

术语（首次出现说明）：
- **REF**：打印给调用者的元素编号（如 `e748160657`）。
- **身份**（`SelectionIdentity`）：worker 判断"同一个元素"的元组，见 §2.1。
- **Fresh**：操作 REF 时 worker 对原节点句柄 `refresh()` 后重读并重新派生身份（`FreshNodes.kt`）。
- **reveal**：把离屏元素滚到可见，并在本地重读确认（局部结果 observed／unobservable）。
- **row.label（行标签）**：元素所在列表行里第一个可机械归属的文字，用来区分"同名按钮属于哪一行"。
- **吸收（absorb）**：可点父节点从不可点后代取文字作自己的 name/detail，后代被标为 `nameConsumed`，并把 `selectionTarget` 指向父节点（选择器命中后代时改去操作父节点）。
- **C13**：实施方代号，指"只带上下文点击（context-click）动作的节点不再算可点目标"的规则改动。
- **Q12B／Q13B**：主稿第 12 题（行标签）／第 13 题（公开名字）的 B 选项，即"不按可见性筛选、采用已知完整的原始文字"。
- **c669／8d**：实施方对两份 APK 的代号：c669 = 候选原版；8d = 只改 `UiSemantics.kt` 五处条件的身份窄原型。

---

## 1. 结论摘要

1. **第 12 题（行标签是否随可见性变）不该问用户。** 它是确定缺陷：同一节点、同一原始文字，只因 `visible` 翻转就换 REF、reveal 局部报 unobservable。B 是唯一合理修法，实施者自行决定即可。A 是保留缺陷，D 是为绕开缺陷另造一套比较口径，都是凑数选项。C（删掉行标签）比 B 弱，省不下实现成本。
2. **第 13 题（公开名字是否共享原始语义）也不该单独问用户。** 用户已偏好"`find` 默认搜全部已采节点（含离屏）"。在这个前提下，离屏可点卡片的名字只能来自离屏后代，B 是唯一自洽的选择。C 会让离屏卡片在 `find` 里没有名字，与该偏好冲突；A、D 保留缺陷。建议把 12、13 合并成一条实施原则："名字与行标签不依赖可见性；visible/bounds 只用于状态判断"。窄原型本来就是把两者一起改的。
3. **频率不是"未知"，有离线数据。** 我对全部保存的连续树对做了静态扫描。MuMu 主语料 114 对中有 12 对出现"地址、自身原始字段、子树文字都不变，只有可见性变了，身份却变了"的节点，共 130 个，其中 44 个可点；涉及 Chrome（MDN、GOV.UK、标签栏）、Google Maps、Bilibili 三个 App。API34-recovery 45 对中 18 个（1 个可点），r2-nodes 20 对中 34 个（8 个可点）。【重算】实施方称"GOV.UK/Bilibili 窄查未找到合格名字对象"，这一说法不完整：Bilibili `210→211` 中一张可点视频卡滚出屏幕后名字从标题变成空串，换了 REF。【原件】
4. **行标签的独立保护价值，在全部语料中没有出现过一例真实个案；名字派生则有。** 在约 200 个连续树对中，"只有行标签能区分"的候选共 3 个，全部是可见性伪差（Maps 051→052；NIST 013→014、015→016）。"只有派生名字能区分"的同址内容替换有真实个案，例如京东 `185→186` 中同一可点 ViewGroup 从"烧烤炉"变成"飞机"。【重算＋原件】行标签仍有用：选择器消歧和签名修复都以它为一票否决，而且不论身份里留不留它都要计算。所以 C 省不下实现，只会丢掉一道防线。支持保留，采用 B。
5. **组合成本的字节数全部复核一致。** 例如 California 8191 节点 6,324,841 B、余 2,063,767 B；旧稠密表示超限 86,150 B。【重算】但"旧稠密 B 超限"不构成真正的反例：这份 5485 节点基线树本来就被裁到距上限只剩 5,903 B，任何增长都会超；现行最终裁剪在身份派生之后执行，会如实裁尾并标 partial。所以"第 6 项必须等稀疏 canonical 先完成"是排期偏好，不是正确性依赖。
6. **"`detailComplete=false` 新摘要缺口"与 Addis 493 字符吸收是同一个问题。** 问题出在吸收规则：大号可点网页容器把整段正文吸进 detail，正文碎片被标成 consumed，选择器（如 `--id firstHeading`）被归并到大容器。它不涉及身份或字节。建议实施者自定一条不加阈值的窄规则：吸收触及既有 64 节点／8 层上限时，不设 consumed 和 selectionTarget。不需要问用户。
7. **窄原型的描述基本如实。** 五处条件属实；真实运行只有 Maps 一次 reveal、MDN 一次 reveal 加一次原 REF 点击，均在 MuMu 单一镜像上，原 REF 保持、后续点击成功。主稿明确写出它不含 C13、稀疏表示和选择器新入口。
8. **提交 6、8 的修订：**第 6 项改为"可见性中立的名字与行标签"，解除对第 4 项（稀疏 canonical）的硬依赖。第 8 项只做 C13 和吸收边界，验收用离线重算加一次 Addis 真实点击，不要求"完整组合"矩阵。详见 §2.8。
9. 主稿把"切换规则会改部分旧身份"列为代价，这是过虑：REF 只存在于 worker 内存会话中，升级 APK 后旧 REF 本来就全部失效。【源码＋推断】

---

## 2. 逐题分析

### 2.1 读懂现状：身份由什么组成、各防什么、可见性在哪一步进入

**身份元组。**【源码】`SelectionIdentity.kt:8-15`；`UiSemantics.kt:315-329`：

| 字段 | 来源 | 防的误操作 |
| --- | --- | --- |
| `sourceId` | 隐藏 API `sourceNodeId`（`NodeFacts.kt:184-188`），即平台节点地址（View 的无障碍 ID，或网页／Compose 的虚拟节点 ID）。 | 节点被替换成另一个对象。 |
| `windowId` | 节点所在窗口。 | 跨窗口误用，例如弹窗盖住后旧号落到别的窗口。 |
| `role` | 由 editable／checkable／className／collection 等推出（`UiSemantics.kt:58-69`）。 | 同一地址变成另一类控件。 |
| `name` | `view.name`：先取自有 text/description；为空且节点本身可操作时，才从**可见、非独立目标**的后代吸收（`:78-99`、`:141-171`）。 | 列表或轮播复用同一个 View 显示另一条内容。下文京东"烧烤炉→飞机"、Maps 起点字段的真实个案都靠它挡住。 |
| `rowLabel` | 最近的列表行（collectionItemInfo）里第一个**可见** text；没有时退到 nameOnly 名字，再退到**可见** description（`:175-305`）。 | 同名按钮（如 `Delete`）在同一地址、同一行号，但所属记录变了，例如列表删项后用 notifyDataSetChanged 整体重绑。依据是单元测试 `UiSemanticsTest.kt:103 rowIdentityIgnoresInputs...`（Alpha→Bravo）。 |
| `positions` | 到根的所有 collectionItemInfo 行列号链。 | 视图回收后绑定到另一行位置。 |

身份不完整（sourceId 缺、className 不可读、`nameComplete=false`、`row.complete=false`）时不分配可用 REF（`UiRefs.kt:20-50`）。地址重复或展开被截断时也不分配。

**消费者。**【源码】
- `UiRefs.replace` 只在身份元组完全相等时沿用旧 REF 编号。
- `UiVerification.requireOriginalIdentity` 在派发前和 reveal 局部检查时严格比较；只有写值动作之后放宽 name（`UiVerification.kt:22-32`）。
- `UiEffect.kt:41-53` 身份不等就报 effect unknown。
- `UiRefRepair.kt:83-114` 是派发前的签名修复，以 name 和行标签作一票否决。
- 选择器用行标签消歧（`UiSelectorTest.kt:21-40`）。

**可见性进入派生的三处**（行号为 3f45af6）：
1. `Selection.visitChild` 遇到 hidden 后代就跳过整棵子树（`UiSemantics.kt:159-160`）。所以无自有文字的父节点，名字取决于后代当前是否可见。
2. `RowLabel` 只认 visible 节点的 text（`:238-243`）。
3. `RowDescription` 只认 visible 的 description（`:287-291`）。

另外，`consumed` 要求父节点可见（`:132`），这是阅读层的去重，不进身份。

**基线 7d5be70。**【源码】`git diff 7d5be70 3f45af6` 显示：`SelectionIdentity` 字段、`RowLabel`/`RowDescription` 的可见性门、`visitChild` 的 hidden 跳过在基线中都已存在。候选只把"命名动作"判定换成更宽的 `publicTarget`，`requireOriginalIdentity` 改为集中实现。**也就是说，可见性依赖是基线就有的老问题，不是候选新引入的。**

### 2.2 核实问题：MDN 与 Maps 是否属实，是缺陷还是合理设计，出现频率如何

**MDN（c669，MuMu，Chrome Beta）。**【原件】
- `mumu/raw/079→081`（source `2211908157736`）与 `085→087`（source `8697308774696`）：目标的原始字段只有 `visible:false→true` 和 bounds 不同。
- view 完全相同（`role=item`，`name=Examples` 来自自有 description），row 链 `{row 2, col 0}` 相同，targetUrl 相同。
- 唯一的派生差异是 `row.label: null→"Examples"`；行节点下的文字子节点 `"Examples"` 两端都在。
- 祖链上的 sourceId 全部相同。唯一的原始差异是祖先 WebView 的滚动 actions 变了，它不进身份。
- 回执 `mumu/commands/081-…/trace.json`、`087-…/trace.json`：主结果 done；`reveal.state=unobservable`，`error.code=selectionChanged`，reads 分别为 11 和 8；`effect.state=unknown`。
- 对照 `084`：正文中不在列表行里的目标 reveal 后为 observed。
- 结论：**属实**。值得一提的是，这里的行标签"Examples"与目标自身名字完全相同，没有提供任何额外区分信息，却造成了拒绝。

**Maps 无行标签的可点父节点。**【原件】`mumu/raw/051→052`，source `-4294966420`/`-4294966417`（"关于这些结果"、"最好提前预订"）：两端子节点文字相同；name 从 `''` 变为有名，REF 被换。属实。我的扫描还在同一对树中多找到两个可点节点："大英博物馆"（`-4294966759`）可见→隐藏后名字丢失，`-4294966410`、`-4294966398` 隐藏→可见后获得名字。

**缺陷还是设计？是缺陷。** 理由：
- 身份的用途是回答"还是不是同一个东西"。可见性回答的是"现在能不能看到、能不能点"，候选已经在 `ready`、reveal、effect 里单独处理它。
- 把可见性混进身份，在节点没变时制造"已变"，只产生误拒绝。
- 它是安全方向的失败（拒绝而不是误点），代价是效率：reveal 结果不可观察；调用者必须重新取树、换号；滚动之后旧号失效；effect 失去观察。

**频率（新数据）。**【重算】`freq3.py` 的筛选条件：连续两棵保存树中，同一 `(windowId, sourceId)` 节点的可见性翻转；它自身的原始字段（不含 visible、bounds、REF、派生字段）不变；它的子树（sourceId、text、description、className、clickable）不变；所在行节点及其子树不变；祖链原始字段（不含 actions）不变；但派生身份变了。

| 语料 | 连续对数 | 受影响对数 | 节点数（可点数） | 涉及 App 与变化字段 |
| --- | ---: | ---: | ---: | --- |
| `exploration-20261119/mumu/raw` | 114 | 12 | 130（44） | MDN 与 GOV.UK 的行标签；Maps 的行标签 18 个、名字 5 个；Chrome 标签栏名字 1 个；Bilibili 名字 1 个。 |
| `exploration-20261119/api34-recovery/raw` | 45 | 3 | 18（1） | NIST 行标签。 |
| `r2-nodes/runtime-mumu/raw` | 20 | 6 | 34（8） | Chrome 行标签。 |
| `api34-editor-pair`、`api34/raw` | 29 + 12 | 0 | 0 | — |

这只是静态的"身份会变"计数，并不等于每个节点都发生了一次失败；调用者要拿旧号跨越这次变化才会撞上。真实回执里：包内 16 次 reveal 中有 3 次报 selectionChanged，全部来自 c669；其中 MDN 两次发生在自然业务流里，Maps 那次是有意构造的。8d 对应的两次都是 observed。【原件】样本只有一台 MuMu（另有一份 API34 AVD 语料），不涉及真机或厂商 ROM。

### 2.3 第 12、13 题该不该问用户

**第 12 题：不该问。**
- B 修的是确定缺陷，不改变公开 name，只改 `row.label`。实施方提到的代价有两项：
  - 一是"隐藏先行字段缺口会收紧"。我统计了包内 271 份树、163,366 个节点，`unavailable` 一次也没有出现。【重算】这项代价在现有数据里从未发生，且仍是安全方向（拒绝）。
  - 二是"字节增加"，见 §2.5。
- A 等于保留缺陷。D 要维护一套只用于 reveal 局部检查的比较口径，最终 REF 仍然被换，不能修复"滚动后旧号失效"。两者都不是真实的产品取舍。
- C 能修掉误拒，但行标签仍要为选择器和修复计算，C 省不下代码，只丢掉身份里的一道防线。
- 这是实施者能自己定的事。

**第 13 题：在 `find` 搜全部已采节点的方向下，B 是唯一自洽的选择。**
- 我核到的事实：Bilibili `210→211`，source `-4294966267` 的可点视频卡在离屏时 name 为 `''`、detail 为 `''`，而 9 个子节点的标题、播放量等文字都完整存在（`cmp.py` 输出）。【原件】
- 现行规则下，`find "宋瓷"` 在离屏范围里只能命中不可点的 TextView 叶子。原因是隐藏的父节点不吸收，叶子的 `selectionTarget` 为空，调用者拿不到可点的卡片 REF。【源码推断：`UiSemantics.kt:87,159,331-355`】
- 在 B 下，卡片得到标题名字，叶子的 selectionTarget 指向卡片，`find` 能直接给出可 reveal、可点击的目标。这正是主线第 7 条"离屏命中给出引用，可直接 reveal"所需要的。
- **C**（身份用原始名字，阅读用可见名字）：阅读名字在离屏时为空，`find` 搜不到离屏卡片的标题，与用户偏好直接冲突。除非把"阅读名字"再定义成原始名字，那就退化成了 B。
- **A、D** 保留缺陷。
- 唯一真实的"产品层"变化是：可见父节点若有隐藏子节点（例如被底栏裁掉的半张卡），其 name/detail 会包含被裁部分的文字。这与"已采即可读"的方向一致，不需要单独裁决。
- **建议：**把第 12、13 题合并为一条实施原则，交给实施者执行，不再作为用户选择题。

### 2.4 有没有更简单的身份模型

候选替代方案：身份只用"窗口 ID + 节点 ID + 角色 + 自有文字（+ 行位置）"，不派生名字和行标签。

它会丢掉的、**已有真实个案的**保护：【重算 `recycle.py`，原件】连续树中同一可点地址、两端都可见、自有 text/description 相同、只有派生名字不同的共 6 例：
- 京东 `185→186`：同一 ViewGroup（`-4294967251`）从"烧烤炉"变成"飞机"，是同一地址展示了另一条内容。没有派生名字，旧 REF 会点到"飞机"。
- Maps `057→058`：起点字段从"National Gallery…"变成"国家美术馆"；`058→059`：路线摘要变化。
- Bilibili `210→211`：标题省略形式变化（同一视频，属于误拒）。
- Bilibili 信息流卡片没有 collectionItemInfo（row 链为空），行位置提供不了任何保护。复用的卡片只能靠派生名字区分。

**派生名字必须保留。**

**行标签的独立价值。**实施方说"只有既有行为依据，没有真实误点个案"。核实结果：**属实，而且更进一步**。"只有行标签能区分"的候选 3 例，全部是可见性伪差：
- Maps `051→052`：`视频` 按钮，行标签 `Immersive View→视频`。原因是 TextView "Immersive View" 被裁成 hidden。
- NIST `013→014`、`015→016`：行标签 `"1. "` 与 `"Enter a chemical species…"` 来回切换，原因是序号文字 visible 翻转。

也就是说，现行行标签在语料中只造成误拒，没有挡住过一次真实误操作。【原件】但它防的场景（列表删项后整体重绑、同名按钮换了记录）在 RecyclerView/ListView 用 notifyDataSetChanged 时确实会发生，属于【推断】，模拟器语料没有覆盖删除流程，仍是合理的防线。而且：
- 行标签还被选择器消歧和签名修复使用，不论身份里有没有它都要计算；
- 在 B 下它只在真实文字变化时才变。

所以保留它的边际成本接近零。**更简单的模型不值得**：B 就是"同一机制去掉可见性门"，代码量与 C 相当，比"只用地址"多保护京东类和 Bilibili 类真实场景。

**"更稳的信号"替代行标签？**collectionItemInfo 没有条目 ID，而行节点自身也是被复用的 View，平台没有给出比"行内文字"更稳的条目键。网页节点的 sourceId 跟随 DOM 节点，本来就稳定。我没有找到更好的信号。

**复杂度是否相称。**身份机制约 230 行（`UiSemantics.kt:78-329`），由 64 节点和 8 层上限约束。修掉可见性门后，它防住的风险（同址换内容）有真实个案，误拒来源被去掉，复杂度相称。主要的维护负担在吸收规则（§2.7），不在身份比较。

### 2.5 组合成本

**数字核对。**【重算】我读取了 `R/reveal-identity-cost/results.json`：
- California 8191 节点：A 6,185,243 B；B 重算可用门后 6,324,820 B；C13 加 B 为 6,324,841 B，余 2,063,767 B。
- Related：C13 加 B 为 6,307,741 B（主稿余量 2,080,867 B）。
- California baseline 稠密原始体 8,382,705 B；B 重算可用门后稠密 8,474,758 B，超出 86,150 B。

以上均与主稿一致。§7.5 的共享 selection 组合（6,325,678／6,308,500 B）我没有重跑，只复核了 §6 部分。B 使最大树增加约 139.6 KB（+2.26%）。

**稠密反例说明了什么。**基线稠密体本来就只比 8 MiB 上限低 5,903 B，因为最终裁剪会把树裁到贴近上限。所以任何派生字节增长都会"超限"，这不是 B 特有的风险。候选在 `UiSnapshot.kt:31-43` 的执行顺序是：先 `project`（派生 view/row）、再分配 REF、最后 `UiTreeBudget.fit`。超出部分会被如实裁尾，并标 `partial` 和 `treeByteLimit`（`UiTreeBudget.kt:16-40`）。【源码】所以在稠密表示下先上 B，后果只是超大树多裁掉约 1% 的尾部节点；这类树本来就是 partial。这证明不了 B 必须排在稀疏 canonical 之后。

**"`detailComplete=false` 的新增摘要缺口"是什么。**
- 在 California／Related 两棵大树中，同一个隐藏的可点 `android.view.View` 父节点（`e110157150`）同时失去"隐藏"和"上下文点击"两道阻隔后，吸收了标题"List of municipalities in California"，并继续收集正文，触到 64 节点／8 层上限，`detailComplete` 变成 false。
- **对调用者的影响：**host 会给该行加 `unreadable` 标记（`ui_view.py:150-154`），detail 被截断。名字完整，身份不受影响，可用 REF 不受影响。它与 Addis 的 493 字符吸收是同一现象（§2.7）。

**离线重算证明了什么：**在固定的保存节点集合上，B、C13、共享名字三者组合后的最终 JSON 低于 8 MiB；可用 REF 静态计数没有损失；MDN 和 Maps 的跨端点身份相等。
**没有证明什么：**真实采集的早期预算和先后顺序；Fresh 读隐藏子树的真实 provider 读取次数和截止时间；新 REF 的分配和代际；最终 fit 的实际运行；未来 SingleView 的字节；Android 端派生耗时和内存。主稿 §84 对这些边界的表述如实。

### 2.6 窄原型的证明范围

**改了什么。**【原件】`P/exploration-20261119/identity-prototype/source.patch` 中产品源码只有 5 处条件：
1. `visitChild` 去掉 hidden 跳过；
2. `RowLabel` 的 text 完整性判断不再依赖 visible；
3. `RowLabel` 接受 hidden text；
4. `RowDescription` 的 description 完整性判断不再依赖 visible；
5. `RowDescription` 接受 hidden description。

其余是测试改动。README 记录 JVM 410 项通过，我没有重跑（无构建环境）。

**真实运行（MuMu，8d APK）。**【原件】`identity-runtime/mumu/commands/`：

| 动作 | 结果 |
| --- | --- |
| Maps `035` reveal | `observed`，reads 1，原 REF `e770640517` 保持，effect changed（visible、bounds）。对照臂 c669 的 `014` 为 `unobservable/selectionChanged`。注意两臂是不同的 provider 对象（窗口 26 与 72），不是同一对象的 A/B 对比。 |
| MDN `050` reveal | `observed`，reads 12，原 REF `e770640889`。 |
| MDN `052` 用原 REF 点击 | done；stdout 中出现 `timeout_static#examples`；`mdn-code-evidence.json` 含完整示例代码节点。 |
| MDN `053` Back | 原 REF 和行标签保持。 |

**样本量：**真实 reveal 2 次、点击 1 次、App 2 个，只在一台模拟器镜像上，没有重复。

**主稿是否如实。**基本如实：明确写了"只有 raw selection＋Q12B，不含 C13、最终 sparse/canonical、NAME 或 SingleView"，wall time 不构成提速证明，reads 不等于 IPC 次数。我补充两点：
- 隐藏字段缺口在原型运行中没有出现，在全部语料中也是 0 次；
- Fresh 进入隐藏子树的实际读取代价只有 reads 轮数，没有 provider 读取计数。主稿把它列为未知，表述准确。

### 2.7 C13 与名字的相互作用

**事实。**【原件】`r2-nodes/target-results.json`：Addis 页面的可点 `android.view.View` `e664667262`（bounds `[0,160,720,1182]`）在 C13 后：
- name 从 `''` 变成 `Addis Ababa`；
- detail 成为 493 字符的碎片串，形如 `" ( | ; | :  | አዲስ አበባ | ,  | literal translation | … is the capital and largest city of  | . …"`；
- `detailComplete=false`；
- 标题叶子 `e664667281` 被设为 `nameConsumed=true`，`selectionTarget` 指向这个大容器；
- 多个正文片段节点也被标为 consumed。

**对阅读的影响。**【源码推断，`ui_view.py:116,225-262`】
- 被 consumed 的正文片段不再单独成行，而是作为大容器的 detail 输出。
- 链接文字因为本身是独立目标而被排除，所以 detail 里出现"is the capital and largest city of  | ."这种缺词碎句，链接另行列出，阅读顺序被打乱。
- 该行还带 `unreadable` 标记。这是阅读质量的退化。

**对身份的影响。**身份很小：名字取第一个值（"Addis Ababa"），detail 不进身份。

**对执行的影响。**`--id firstHeading` 等选择器会被归并到大容器（`UiSelector.kt:40`），执行目标从标题变成"整篇正文区域"。r2 真实运行显示 reveal 父节点和 reveal 标题都能回到标题（integration-draft G2），所以 reveal 能力没有丧失。但 click 的业务含义改变了，这一点没有真实验证。

**该不该重新设计吸收规则？**
- 现行吸收规则与 TalkBack "可聚焦节点读出其不可聚焦后代"的思路一致。原生 App 的卡片、Bilibili 的视频卡都依赖它。不能改成"含可点后代就不吸收"：Bilibili 卡片含"更多操作"子按钮，这样改会让卡片失名。
- 给名字或 detail 设字符上限，会引入新的阈值，而且解决不了 selectionTarget 归并错位。
- **建议（实施细节，不是用户题）：**以既有的 64 节点／8 层上限作为"这是区域，不是控件"的信号。吸收因触达上限而停止时，该父节点仍保留 name/detail（`detailComplete=false` 照实标出），但**不把后代标为 consumed，也不设 selectionTarget**。正文片段照常单独成行，`--id firstHeading` 仍指向标题本身。这不引入新阈值，只改 `project` 中的一个条件。【设计建议，需 Addis／California 离线重算加一次真实点击核验】
- 若不采纳这条，至少要在第 8 项的验收中把"选择器归并到区域容器"列为接受的已知行为。

**第 8 项验收要求"核 C13 与共享 selection 的完整组合"，是否过于复杂？**组合本身并不复杂。四项改动作用在同一个函数里的两个条件上（hidden 门、publicTarget 门）。§7.5 的"非加性"本质上就是同一个大容器同时失去两道阻隔。解决吸收边界之后，组合只需用现有 `recompute_targets.py` 风格的离线重算核对，不必另建"完整组合矩阵"。

### 2.8 建议提交 6、8 的边界、依赖与验收（修订版）

**原稿的问题：**
- 第 6 项硬依赖第 4 项（稀疏 canonical），依据是稠密超限反例，而 §2.5 已说明它不成立；
- 第 6 项把 12、13 题分开挂选项；
- 第 8 项要求"完整组合"验收但没有给出边界规则；
- 两项的验收都偏重"不能宣称"之类的表述，缺少可执行的判据。

**修订：**

**6 `fix(identity): derive names and row labels independently of visibility`**
- **范围：**`UiSemantics` 中 selection 和 row 的可见性门（即窄原型的 5 处条件）；`consumed` 仍要求父节点可见；`UiRefRepair`、选择器、effect、Fresh 复用同一派生，不加字段、缓存或开关。
- **依赖：**无。可以在稀疏表示之前或之后落地；最终 fit 已在派生之后执行。
- **验收：**
  - (a) 单元测试：纯 visible/bounds 变化时身份保持；Alpha→Bravo、Record A→Record B 真实变化仍被拒绝；隐藏先行字段 unavailable 或截断时为 incomplete；64/8 上限和 gap 不变。
  - (b) 从保存原件构造的固定输入测试：MDN 079/081、Maps 051/052、Bilibili 210/211、京东 185/186（京东用于确认真实变化仍被拒绝）。
  - (c) 复用已有 8d 运行证据；产品化之后在 MuMu 上重跑一次 MDN reveal 加原 REF 点击即可。
  - (d) 跨语言契约样本按 AGENTS.md 规定的流程核对；wire 形状不变。

**8 `refactor(target): treat context-click as ordinary and bound absorption`**
- **范围：**两端都把 context-click 归为普通动作；保留通用 action、无 label 的自定义动作、所有节点 REF；吸收触达上限时不设 consumed 和 selectionTarget（若实施者采纳 §2.7 的规则）；marks 名额。
- **依赖：**最好在第 6 项之后，两者改同一函数，可减少一次身份口径切换。不依赖稀疏表示。
- **验收：**
  - (a) 用八树离线重算核对 target、选择器归并、marks 的差异清单，Addis 和 California 的大容器单列；
  - (b) 一次真实 Addis 运行：`--id firstHeading` 的执行目标和 click 后态符合所选规则；
  - (c) host 与 Kotlin 的目标谓词集合一致。

不再要求"完整组合矩阵"，也不要求"不能直接相加旧差量"之类的说明性条款。

---

## 3. 确定缺陷与证据不足的断言

**确定缺陷（候选源码，基线就存在）：**
- D1 身份派生依赖可见性，在行标签（`UiSemantics.kt:238-243,287-291`）和无自有文字节点的名字（`:159-160`）两处都有。纯可见性变化会导致换 REF、reveal 局部 unobservable、effect unknown。【原件＋源码】
- D2 C13 之后（现行规则下对任何无 context-click 的大号可点容器同样成立），大容器吸收正文：consumed 的正文碎片化，选择器归并到区域容器。【原件＋源码】属于设计缺陷，严重程度为中：影响阅读质量和执行目标，不影响安全。
- D3（次要）`find` 搜全部已采节点时，离屏可点卡片因 D1 没有名字，无法作为命中给出可执行 REF。【源码推断＋Bilibili 原件】

**证据不足或不准确的断言：**
- "Maps 是公开名字问题的唯一真实反例；GOV.UK/Bilibili 窄查未见合格对象"（研究 §7.1）：**不完整**。MuMu 语料中还有 Bilibili `210→211`、Chrome 标签栏 `073→074` 两个可点节点，Maps 051→052 中还有另外 3 个可点节点。
- "真实频率未知"（主稿第 12 题）：**可以部分量化**，见 §2.2 的表。静态计数之外仍未知的是"调用者实际跨越变化使用旧号"的比例。
- "旧 dense 下 B 超 8 MiB 86,150 B，因此 B 依赖先完成 sparse"：数字正确，**推论不成立**（§2.5）。
- "切换规则会改旧身份"被列为采用 B 的代价：REF 只存在于会话内存，升级时本来就会失效，**不是真实代价**。
- "隐藏先行字段缺口可能收紧可用 REF"：机制正确；但 163,366 个保存节点中出现 0 次，应写成"理论上存在、尚未观测到"。

---

## 4. 值得删除或简化的机制

1. **第 12、13 题作为两道用户题**：删除，合并成一条实施原则。
2. **D 选项（reveal 派发后放宽比较）**：不应进入设计空间，它会多出一套比较口径。
3. **C 选项（第 13 题的双名字体系）**：与 `find` 方向冲突，删除。
4. **第 6 项对第 4 项的硬依赖**：删除。
5. **第 8 项"完整组合"验收**：改为明确的吸收边界规则加离线差异清单（§2.8）。
6. 不建议删除行标签或派生名字。前者在 B 下几乎零成本，并被选择器和修复复用；后者有真实保护个案。

---

## 5. 需要用户裁决的事

**本专题没有必须由用户裁决的事。**实施者可自定的事项：
- 名字与行标签可见性中立（原第 12、13 题，推荐 B）；
- 吸收触达上限时不 consume、不归并（§2.7）；
- 第 6、8 项的顺序与验收（§2.8）。

如果用户希望保留一项产品层确认，最多一句话即可："离屏元素与被裁部分的文字会出现在名字和 detail 中（`find` 全量方向的直接结果）"，作为告知，不设 A/B/C/D 选项。

---

## 6. 未能核实的内容

- 窄原型的 JVM 410 项、ktlint、R8 构建结果：没有构建环境，只看了 README 和 patch。
- 研究 §7.5 的共享 selection 组合字节（6,325,678／6,308,500 B）：未自行重算，只复核了 §6 的 `results.json`。
- 8d 的 Maps／MDN 截图：未查看图像，只核了 trace 和 stdout。
- 行标签防护的场景（列表删项后整体重绑）：在 Android 上是否真的会保持同一 sourceId 和行号【需设备】；Compose LazyColumn 在复用槽位时语义节点 ID 是否保持【需设备／上游源码】，本次未查。
- Fresh 在 B 下进入隐藏子树的实际 provider 读取次数和截止时间【需设备】。
- 我的频率扫描只比较相邻保存树，子树比较只看 sourceId、text、description、className、clickable 五个字段；并非所有可见性翻转都会被调用者撞上，所以它是"会变"的上界性计数，不是失败率。
- §2.7 吸收边界规则的离线效果和 Addis 真实 click 后态均未运行。
- 所有运行证据只来自 MuMu 和 API34 AVD 两种模拟器镜像，不涉及真机或厂商 ROM。
