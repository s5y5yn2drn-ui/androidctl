# 第四轮专题 36：界面树采集、节点编码、身份与名字、节点动作、动作后观察与等待

评审对象：产品 `4235af9`（下文行号默认指它），对照旧候选 `3f45af6` 与主分支基线 `7d5be70`。只读核查；没有设备、没有跑 Gradle。主机侧只跑了三个测试文件（见 §5）和若干本地参数错误。本文用到的离线脚本在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents7/36/scripts/`；下载的 AOSP 文件在同目录的 `aosp/`。

术语：**手写回调采集器**指 `CollectionNodes.kt`＋`CollectionReplies.kt`。它不走公开的 `AccessibilityNodeInfo.getChild`，而是用反射调用隐藏的 `IAccessibilityServiceConnection.findAccessibilityNodeInfoByAccessibilityId`，自己实现回调 Binder 并解析回包。实施方称它 E9。**Fresh 读取**指动作前后对原目标节点做的现场重读（`FreshNodes.kt`），走公开 API。**完整树**指 `completeTree()` 为真，即 `partial=false` 且所有根都是 `full`（`UiObservation.kt:45-49`）。

---

## 1. 结论摘要

**整体状态。**代码已经成形，逐行读下来没有发现会点错目标的身份缺陷。REF 身份由“节点源地址、窗口、角色、名字、行标签、行位置”共同组成，同一时刻两个不同节点拿不到同一个身份。主要问题出在别处：产品把“整棵树是否完整”当成选择器、`gone`、`idle`、REF 修复和动作后稳定判断共同的前提，而长网页上的树按设计**总是**不完整。真正的成本落在这里，而不在采集路线怎么选。

最重要的发现：

1. **长网页上，选择器类命令基本不可用，失败的代价又很高。**California 维基长页的 15 份独立样本全部撞上 8192 节点上限。每份只有 60–81 个可见节点，其余八千多个都在屏幕外；23/26 份 partial 树里，没有一个可见节点缺子节点。可是 `click/locate/wait --role/--name/--id`、`wait --gone`、`wait --idle`、REF 修复和动作后稳定判断都要求完整树。产品不会因此立刻失败，而是反复重采到 5 秒预算耗尽。失败结果里也不带它其实已经找到的唯一候选 REF：实测 `wait --id firstHeading` 匹配 1、目标 1，仍然 5.7 秒后 exit1，连快照都没交付。这是本专题最该改的地方（缺陷 D1）。【源码＋原件＋重算】
2. **采集路线的取舍不是性能瓶颈。**8192 节点那一页，设备端采集用了 585–1816 ms，整个主 RPC 却是 1.6–2.9 s。多出来的约 1 秒花在语义投影、REF、编码和压缩上，此外 CLI 每次还有约 1 秒主机开销。小页面采集只要 34–90 ms。手写采集器和公开路线之间那几百毫秒的差，被这些成本盖过了。我的建议是：**保留但冻结**。它不比基线多隐藏成员（节点相关的隐藏成员从 16 个降到 10 个，因为最低 API 提到了 34），但多了一个手写 Binder、约 365 行产品代码和 342 行测试。另有一条完全公开、从未实测的替代路线：`UiAutomation.clearCache()` 从 API 34 起就是公开方法，我已对 AOSP 14 核实。今后如果真出现回调协议错误，改走这条路线，不要打补丁。【源码＋原件＋上游】
3. **动作后观察实际上是 250 ms 一次的轮询。**最小起点间隔落地后，所有在采样之后等待的循环（settle、wait、capture、选择器）里，无障碍事件已经不能让下一次采样提前；事件相关性过滤只剩 Fresh 重读循环还在用。普通 click 的耗时：
   - 最好情况：页面很快变完并静止，采 2–3 次，设备端 0.4–0.65 s，CLI 约 1.4–1.6 s。
   - 无变化或持续变化：采满 1.5 s 的观察窗，约 6 次。
   - 长网页：窗内只来得及采 1 次，必然 `unsettled`，CLI 3.5–4 s。

   【源码＋原件】
4. **动态文字的节点会失去 REF，这是设计如此，不是缺陷。**名字是身份的一部分，所以计时器、计数、价格一变，旧 REF 就会被零派发拒绝。名字这一项是对付 RecyclerView 复用 View 的最后一道防线，应当保留。实施方的变名实例是秒表的显示文字，那不是可点目标，所以这个实例只证明了“名字变了 REF 就失效”。【源码＋原件】
5. **快速失败只覆盖了一部分。**已知回调里的布局错误会立即以 `collectionProtocol` 失败，在 API34 上用本地回环探针证实（exit0；MuMu 上断言全过但 exit139）。但还有三个缺口：
   - 未知事务码会被静默忽略，最后被报成时间截止；
   - 动作回调 `ActionCallback` 遇到布局不符仍然静默丢弃，一直等到整个请求截止；
   - 对尾部多余字节零容忍，未来平台在回调末尾追加参数时会让每次采集都失败。

   回调接口在 AOSP 14 到 main 之间没有变（已核）。【源码＋上游＋原件】
6. **稀疏编码两端确实一致**：Kotlin 产出契约、Python 校验同一份契约，“缺省”和“不可用”靠 `unavailable` 列表区分，没有混淆。但它的实际收益只是解码体积：压缩后线上字节不变（约 43 万字节），只是让 2733 个屏幕外节点能塞进 8 MiB 上限，代价是每次快照多约 0.7 s。【源码＋原件】

**对待决题 2（动作后仍见旧页）**：我同意保持现有算法（实施方的 A）。但选项里漏了一个更值得做的方向：让长网页上的稳定判断、选择器和 gone 不再因屏幕外截断而必然失败（见 §5、§6）。

---

## 2. 确定缺陷

### D1（中高）选择器在“确定会重复出现的截断”上反复重采到超时，失败时不给候选

- **触发**：长网页（树撞到 `nodeLimit` 或 `treeByteLimit`）上使用 `click/long-click/... --role/--name/--text/--id`、`locate`、`wait <选择器>`、`wait --gone`、`wait --idle`。
- **源码**：
  - `UiActions.kt:878-913` 的 `selected()`：`matches.complete` 为假时返回“uniqueness is unknown”，然后 `awaitNextCheck` 再采一次，直到预算耗尽才抛 `notReady`。
  - `UiCondition.kt:133-160`：选择器条件不完整时 `reason` 不带候选 REF。
  - `UiCondition.kt:166-171`：`gone` 必须完整树才能成立。
  - `waitUiCondition`（`UiCondition.kt:441-466`）：不完整不算不可重试错误，一直采到预算耗尽。
  - `UiSelector.kt:21-24, 43`：完整性取整棵树，而匹配本身只看可见节点（`:47`）。
- **原件**：`ST/13-nodes/runtime/mumu/nodes/raw/010-california-firstheading-selector.trace.json` 的 predicate 是 `matches 1, targets 1, unknown 0, complete false`，结果 `failed`，`snapshotError: Request budget exhausted before snapshot delivery`，CLI 5.703 s，exit1。
- **重算**：`scripts/partialscan.py` 对材料包里的 187 份树做了统计：Chrome 126 份中 26 份 partial；Maps、Settings、WPS、Contacts、Bitwarden 等原生 App 61 份全部 full。`scripts/treestats.py` 显示，California 的 15 份样本全是 `nodeLimit` partial，可见节点 60–81 个，`missing` 约 2000 条。补充脚本显示，26 份 partial 树中有 23 份里**没有任何可见节点 `childrenIncomplete`**，也就是可见节点的子节点全都读到了。注意这不等于“可见内容一定完整”：同一批样本里有 32–46 个可见节点挂在一个零高度、不可见的祖先下面（`scripts/hiddenanc.py`）。
- **影响**：每次失败白花 5 秒以上，而且拿不到任何可以继续用的东西。调用者只能再跑 `find` 或 `snapshot` 取 REF，再发一次动作。对 LLM 来说，这把一次调用变成了三次。效率实验里“GitHub old 的 wait 18 次采样超时”就是同一类现象。
- **最小修正**：
  1. 当 partial 的原因只有 `nodeLimit` 或 `treeByteLimit`（同一页面下次采样会以同样方式截断）时，第一次采样后就停止重试，直接以 `uniquenessUnknown` 失败。
  2. 失败回执里附上现有匹配结果中的候选 REF（最多 5 个，复用 `UiSelector.ambiguity`），调用者可以直接改用 REF 动作。
  3. 只有 `collectionLimit`（时间截断）才值得重采。

  这些都不改变“不完整就不证明唯一”的原则，只是把失败变快、变得可以接着用。
- **确定度**：源码确定；原件 1 例。

### D2（中）动作后稳定判断和 REF 延续在长网页上必然失效

- **源码**：
  - `UiPostAction.kt:188, 209-212`：只要样本不完整，`same` 就清零，结果只能是 `UNSETTLED`。
  - `UiSnapshot.kt:37`：新 REF 只从**紧挨着的上一份**样本继承（`refs.replace(assembled, targets…)`）。某个节点只要在一份样本里没被采到，再次出现时就拿新号。
  - `UiBudget.kt:52`：单次采集截止取 `min(until, deadline, 起点+3000)`，所以动作后的采集会被 1.5 s 观察窗截断。
- **原件**：California 上 004、005、009、013 的 `settle` 全是 `unsettled`，窗内只有 1–2 次采样，主 RPC 2.2–2.9 s。API34 镜像上，网页滚动后的观察只采到 552 个节点就被 `collectionLimit` 截断（`ST/16-delivery/runtime/api34-regions/saved-bodies/main/010-…tree.json`、`014-…`）。
- **影响**：长网页上，动作后观察从来给不出 `changed` 或 `unchanged` 的结论。后续每次快照、wait 采样，都可能重新编号此前被截掉的屏幕外节点，调用者手里那些由 `find` 得来的离屏 REF 会因此失效（报 `staleRef`）。
- **最小修正方向**：
  - REF 部分可以完全在内部解决：`UiRefs` 保留最近几份样本发出过的“身份到 REF”映射（例如最近 4 份），节点再次出现且身份相同时沿用原号。约 20 行，不改任何协议。
  - 稳定判断部分需要用户取舍，见 §6 U1、U2。
- **确定度**：源码确定；原件只说明长页上必然 `unsettled`，REF 被重新编号属于推断，原件中没有直接成对的例子。

### D3（低中）动作回调遇到布局不符时静默丢弃，等满整个请求预算

- **源码**：
  - `AccessibilityConnection.kt:160-177`：`interaction` 不符时直接 `return`；`dataAvail()!=0` 时 `return false`。
  - `:192-193`：`awaitResult` 一直等到 `budget.deadline`，也就是整个请求的工作截止（默认 5 s 减去预留）。
- **影响**：如果某个平台或 ROM 修改了这个回调，每次动作都会先空等到截止，再报 `UNKNOWN`。动作此时其实已经派发，之后也没有时间做动作后观察。这和采集侧已经做好的快速失败不一致，正是第三轮 N4 指出的模式在动作这一侧没修。另外，即便协议正常，App 主线程卡住时动作回调也会吃掉整个预算，而公开 `performAction` 自带 5 s 上限。
- **修正**：复用 `checkCollectionCallbackLayout` 的思路，布局不符时立即唤醒并报 `actionProtocol`。约 10 行。
- **确定度**：源码确定，没有在任何设备上观察到触发。

### D4（低）快速失败漏掉“未知事务码”

- **源码**：`CollectionNodes.kt:120-121`：既不是 primary 也不是 prefetch 的事务码，会交给 `super.onTransact` 后被忽略。这时 `awaitPrimary` 等到截止，`TreeCollector.addChildren` 把它记成 `collectionLimit`（`TreeCollector.kt:344-345`），也就是时间截止，而不是 `collectionProtocol`。
- **影响**：如果某个 ROM 改用别的方法返回结果，现象会是“每次采集都只剩根节点，并报时间截止”。这恰好是第三轮 §2.3 想用诊断消除的症状。
- **修正**：对这个只用于查询的 Binder，收到非 `INTERFACE_TRANSACTION` 的未知事务码就 `fail(...)`。一行判断加一个测试。
- **确定度**：源码确定。

### D5（低，推翻评审方第三轮的一项决定）对尾部字节零容忍

- **源码**：`CollectionReplies.kt:157-163` 的 `remainingBytes == 0` 是硬性要求。
- **上游**：回调接口 `IAccessibilityInteractionConnectionCallback` 在 AOSP 14.0.0_r1、15.0.0_r1、16.0.0_r1 和 main 中，`setFindAccessibilityNodeInfoResult(in ANI, int)` 和 `setPrefetchAccessibilityNodeInfoResult(in List<ANI>, int)` 的签名相同。`findAccessibilityNodeInfoByAccessibilityId` 的 7 参数签名在 14 与 16 中也相同（`aosp/IASC-android-1{4,6}.0.0_r1.aidl`）。
- **理由**：交互 ID 是在检查尾部之前读出来的。如果平台在回调末尾**追加**参数，交互 ID 仍然读得对，忽略尾部完全安全；如果平台在前面**插入**参数，交互 ID 校验本身就会失败，照样能快速失败。尾部零容忍不提供任何额外保护，却会把“末尾追加参数”这种最常见的接口演进变成每次采集都失败。第三轮写的“不做尾部兼容”在这一点上过严。
- **修正**：删掉尾部检查，保留交互 ID 检查。
- **确定度**：源码与上游确定；对现实收益的判断属于推断。

### D6（低中，风险项）REF 动作的变化判断以旧快照为基线

- **源码**：`UiActions.kt:130, 186, 484`：`before = ui.snapshots.cached()`，取的是上一次安装的样本，不会为动作再采一次。`observations.md:357-361` 明说“does not collect a new pre-action sample, impose an age limit or establish causality”。
- **影响**：调用者先 `snapshot`，过了几十秒再 `click REF`。如果这期间页面自己变了（计时、轮播、消息），只要动作后有两次相同的完整样本，就会报 `changed`。默认文本只给一个变化结论，`baselineAgeMs` 要看 JSON 才有，LLM 很可能把它读成“点击生效了”。
- **修正**：
  - 最便宜的做法：基线超过某个年龄（例如 2 s）时，默认文本把结论标成“基线过旧，变化未必由本次动作引起”。
  - 或者：基线过旧时，先采一次再派发。小页面多 40–90 ms，长页面多 2 s。
- **确定度**：源码确定；材料包里没有直接的误判实例。

### 不构成缺陷、但需要写进指引的

**动态文字节点的 REF 会失效。**
- 名字取节点自己的 text/description；容器没有自己的文字时，取第一个后代的文字。行标签取所在行第一个文字。
- 这两者任一变化，旧 REF 都会报 `selectionChanged` 或 `staleRef`。
- 修复路径会拒绝两个都不为空却不相同的名字（`UiRefRepair.kt:163-165, 176-182`），而且要求完整树（`:55`），所以长网页上连修复都不会尝试。
- 结论：对计时器、计数徽标、价格这类节点，REF 只在两次变化之间有效。指引应当直接告诉调用者，这类目标用选择器或在动作前重新取快照。现有 `actions.md:9-30` 讲了规则，没讲这个后果。

---

## 3. 设计与简化

### 3.1 手写回调采集器：保留但冻结（不再加机制）

**这一块的隐藏接口与手写代码清单。**集中在 `HiddenApi.kt`、`CollectionNodes.kt`、`AccessibilityConnection.kt`：

| 类别 | 4235af9 中与采集和动作有关的成员 | 3f45af6 | 7d5be70 基线 |
| --- | --- | --- | --- |
| 节点的隐藏方法和字段 | `getSourceNodeId`、`getChildId`、`LEASHED_NODE_ID`（反射修改可访问性）、`setConnectionId`、`setSealed` | 相同 | `getSourceNodeId`、`getChildId`、`getParentId`、`getAccessibilityViewId`、各类 leash 字段与常量，共 11 个左右，用于 API 33 以下 |
| 交互客户端和服务连接 | `AIC.getConnection`、`AIC.getCache`（API 34）、`UiAutomation.getCache`（API 35 以上）、`IASC.getWindows`、`IASC.findAccessibilityNodeInfoByAccessibilityId`、`IASC.performAccessibilityAction`、`AccessibilityCache.setEnabled` | 相同 | 少 `findNode`；多 `AIC.findAccessibilityNodeInfoByAccessibilityId` 两种重载 |
| 回调存根 | `asInterface`，以及 3 个 `TRANSACTION_*` 常量：节点、预取、动作 | 相同 | 只有动作常量 |
| 手写 Binder 子类 | `CollectionNodes.Query`（`:105-135`，31 行）、`ActionCallback`（`AccessibilityConnection.kt:151-205`，55 行） | 2 个 | 只有 1 个（`ActionCallback`） |
| 手写 Parcel 读取 | `enforceInterface`、`readTypedObject`、`createTypedArrayList`、`readInt`、`dataAvail`；动作侧是 `readInt` 两次加 `dataAvail` | 同类 | 只有动作侧 |
| 采集专用的产品代码行数 | `CollectionNodes` 136＋`CollectionReplies` 173＋`CollectedChildren` 56＝365 | 186 | 0（`NodeAccess.kt` 53 行） |
| 采集专用的测试行数 | `CollectionRepliesTest` 145＋`CollectedChildrenTest` 94＋`CollectionPublicationTest` 103＝342 | 0 | 0 |

【源码，`git show` 三个版本的 `HiddenApi.kt` 和 `git grep ": Binder()"`】

**结论与理由。**
- **风险没有增加。**与基线相比，隐藏成员总数反而少了（最低 API 提到 34 后，删掉了 API 33 以下的路径）。新增的风险和基线早就有的手写动作回调属于同一类。依赖的接口在 AOSP 14 到 main 之间没有变（已核）。
- **性能论据很弱。**第三轮的对比每格只有 1 次。本轮数据显示，长页上真正的成本在采集之后的处理阶段：003 号样本采集 585 ms，主 RPC 1594 ms；008 号样本采集 1816 ms，主 RPC 2735 ms。小页上采集 34–90 ms，而 CLI 下限约 1.0–1.2 s。无论走哪条路线，用户感受到的差别都不大。【原件，`ST/13-nodes/runtime/mumu/nodes/raw/*.tree.json` 的 `source.startedMs/finishedMs`，与 `costs.json` 的主 RPC 时间对照；重算脚本在 `scripts/` 下】
- **它真正的优点**：子节点查询服从本次采集的截止时间；公开查询是同步的，单次最长阻塞 5 s，调用方无法缩短。但根节点和 Fresh 读取在两条路线上都走公开 API，所以这个优点只覆盖了子节点查询。
- **一条从未实测、完全公开的路线**：
  - 保持缓存默认开启，每次整树采集前调用公开的 `UiAutomation.clearCache()`（AOSP 14 中没有 `@hide`，见 `agents/01-aosp/android-14.0.0_r1-UiAutomation.java:519`），然后用公开的 `getChild`。
  - 这样既能删掉手写采集器，也能删掉“关缓存”这个隐藏调用（`AccessibilityConnection.kt:35-41`）。
  - 代价一：AOSP 14 到 main 的 `UiAutomation.onAccessibilityEvent` 都不更新缓存（已读源码），所以缓存只在两次清除之间可信。Fresh 读取虽然有 `refresh()`，但 `window.root` 可能命中旧缓存（`FreshNodes.kt:21`），需要在 Fresh 读取前也清一次。
  - 代价二：仍然受公开查询 5 s 阻塞的限制。
- **需要什么证据才能定**：
  - 在 MuMu 和 API34 两种镜像上，各取长网页、原生长列表、Compose 列表三类页面。
  - 每类页面在两条路线上各连续采 5 次以上（热 worker，配对交替）。
  - 记录采集毫秒数、完整率、动作后新鲜度，并单独做一次“App 主线程卡住”的场景。
  - 判据：如果公开路线的采集中位数不超过手写路线的 1.15 倍，且完整率相同，就换成公开路线，删掉约 700 行产品和测试代码。
  - 我不建议现在专门花设备时间去做这件事。只有出现真实的协议错误，或者这块代码需要再次改动时，才值得做。

### 3.2 真正的杠杆：屏幕外节点的体量

California 页上，8192 个节点里只有 60–81 个可见。每一次动作后样本、wait 样本、快照，都要为八千个屏幕外节点付出约 1–2 s 的设备时间和约 0.43 MB 的压缩载荷。换来的只是：

1. `find` 和 `reveal` 能找到屏幕外的内容。这是真实能力，County 和 MDN 的流程靠的就是它。
2. 让树变成 partial，从而触发 D1、D2。

稀疏编码在这一页的净效果，是多收 2733 个屏幕外节点、每次多花约 0.7 s（`paired/result.md`；4 次测量，每臂 2 次）。

我认为更简单、对调用者更有价值的整体做法，是按用途区分采集：

- 显式 `snapshot` 和 `find` 用现在的 8192 上限。
- 动作后观察和 wait 采样以可见节点为主，屏幕外只收很少，例如 1000 个节点。
- 同时把“可见部分是否完整”作为选择器和稳定判断的完整性依据。

这改变了第三轮以来“所有采样同一套规则”的约定，也会影响 `find` 在动作后的覆盖范围，所以列入 §6 由用户决定。

### 3.3 身份、名字、吸收和可点目标这套语义层

- **身份规则是对的，不会碰撞。**身份含 `sourceId`（窗口内唯一）和窗口 ID（`UiSemantics.kt:320-334`）。同一时刻两个不同节点不可能同身份。跨时间唯一可能“碰撞”的情形，是 RecyclerView 把同一个 View 复用给另一项数据；这时名字、行标签和行位置（适配器下标）三者会同时守住。三者都相同时，基本就是同一个业务项。f7a214d（名字和行标签不再看可见性）解决的是真实的滚动后身份丢失，我核过原件：County 节点在 003→004→005、MDN 节点在 017→021 的 REF、名字和行标签都保持不变，只有 `visible` 和 `bounds` 变了。**支持这一改动。**【源码＋原件】
- **吸收规则对 LLM 有帮助。**`selectionTarget` 让 `--name "Settings"` 能落到可点的父节点上。
- **略显过度、但不急于删的**：
  - 行标签的三级回退（`RowLabel`，然后自身名字，然后 `RowDescription`，`UiSemantics.kt:214-313`）。
  - `row.chain` 记录所有祖先集合项的位置。
  - `nameComplete` 和 `detailComplete` 两个标志。
  - `webEmpty`、`covered`。

  它们每一项都不大，但都会进身份判断或输出。如果要削，先削 `RowDescription` 回退（无文字行用描述做标签）和 `covered`（只是提示）。
- **真正的负担是指引。**`observations.md` 500 行、`actions.md` 420 行，大量是协议级陈述。例如：“Only the legacy single `--window` post-action path has `source.capture`, and raw `tree.source.sequence` against its `snapshotFacts.sequence`”（`observations.md:183-184`），还有 `observations.md:292-297` 那张省略规则表。LLM 调用者不需要这些。建议把规格内容移到独立 docs 仓库，公共指引只留：怎么读、怎么选目标、哪些错误该重取快照，以及 D1、D2 说的长页注意事项。

### 3.4 节点命令面

18 个节点命令各有 23–29 个选项，其中 11 个是阅读和截图选项（`--all/--boxes/--depth/--interactive/--max-chars/--print/--diff-from/--screenshot/--image-format/--marks/--image-out`），每个命令都重复一遍。【实跑，`androidctl click --help` 等】

建议：

- **把 `focus`、`long-click`、`expand`、`collapse`、`dismiss`、`ime` 并入 `action`，例如 `action REF expand`。**它们在设备端只是 `UiAction` 枚举里同一路径的不同平台 ID（`UiAction.kt:15-21`），唯一差别是现在的 `action` 只接受 REF、不接受选择器（`NodeActions.kt:52-54`）。合并时给 `action` 补上选择器即可。少 6 个命令和 6 段帮助文字，没有能力损失。
- **`locate` 与 `wait <选择器>` 高度重叠**：两者都等到唯一可见目标并返回 REF。`locate` 只是多检查一项“支持某动作”。可以合并成 `wait --ready-for click`。
- **`reveal` 保留**：它在派发后还会等目标变为可见并非零尺寸（`UiActions.kt:144-174`），`action show-on-screen` 不做这一步。
- **`click/fill/check/uncheck/range/select/scroll/scroll-until/find/wait` 保留**：它们各有验证或循环语义。
- **错误码种类太多**：`staleRef`、`selectionChanged`、`identityUnknown`、`containerChanged`、`scopeMismatch`、`covered`、`notReady`、`ambiguous`、`actionUnavailable`、`actionLabel`、`actionName`、`actionUnsupported`、`routeUnavailable`、`inputFocus`……对调用者来说，可以归成四类：“重取快照后重试”“不唯一，改用 REF”“目标未就绪”“参数错”。建议回执增加一个只有这四个值的 `next` 字段，细分代码保留在 JSON 里。

### 3.5 评审方前几轮现在看有问题的建议

1. **第三轮 §2.3“不做尾部兼容”**：过严，见 D5。
2. **第三轮 §2.3 保留手写采集器的主要理由“动作后采集窗口只有 1.39 s”**：本轮数据显示，窗口大部分被采集之后的处理和屏幕外体量占掉了，两条路线在长页上都完成不了。这条理由分量不足。结论（保留）可以维持，但理由应当改成“现在能用、风险与已有动作回调同类、换掉要花设备验证时间”。
3. **历轮一直沿用“不完整就不能证明唯一，所以拒绝”的原则**：原则本身没错，但没有配套“快速失败并给出候选”。结果在长网页上，这个原则成了调用者的成本，而不是保护（D1）。
4. **第三轮 §3.2 把稀疏编码列为放行默认**：代码小，可以保留；但它的价值只是解码体积，不应再被表述为效率改进。

---

## 4. 证据审计

| 实施方声称 | 我核对的原件 | 结论 |
| --- | --- | --- |
| County 长页：隐藏目标用原 REF 执行 reveal，再沿原 REF 点击，正文和返回都对 | `ST/13-nodes/runtime/mumu/nodes/raw/003/004/005-*.tree.json`：源 `137881335103971` 的 REF 始终是 `e733619154`，名字和行标签都是 “County statistics of the United States”；`visible` 从 false 变为默认 true，`bounds` 从 `[48,1920,1032,1920]` 变为 `[48,969,1032,1194]` | **成立**。边界：只有 1 次；MuMu API35、Chrome Beta；004 和 005 的动作后观察都是 partial/unsettled，文章正文要到 006 独立读取才看到（实施方如实写了）。 |
| MDN 长页：隐藏的 Examples 用原 REF reveal 后点击 | 017–020：源 `161323266605539` 的 REF 始终是 `e733644564`，行位置 2,0，行标签 Examples | **成立**，只有 1 次。 |
| 按动作名执行 context-click，打开了真实菜单 | `012-…trace.json`：请求 `actionName: context-click`，解析为 `16908348`，`accepted true / callback true`，`settle changed`；012 的树里有“返回、前进、重新加载、下载、打印、分享……” | **成立**，只有 1 次。 |
| 同源计时文字真正改名后零派发拒绝 | 031 和 033：同源 `167078522782179` 的名字从 `00:55.46` 变为 `00:56.48`，REF 从 `…684` 变为 `…686`；033 未发送、`staleRef`，修复采样 1 次 | **成立，但证明力有限**：被改名的是秒表显示文字，不是可点目标，也没有测试“可点按钮自身改名”。它只说明名字一变 REF 就失效。 |
| firstHeading 精确 wait 因 partial 不能证明唯一而失败 | 010 trace（见 D1） | **成立**，如实保留了失败。但实施方只把它当作事实记录，没有把它识别成需要修的产品问题。 |
| 薄探针六案：API34 退出 0，MuMu 退出 139 | `ST/runtime/api34/commands/024-callback-once/`：exit 0，3.109 s，stderr 0 字节，六案都通过；`ST/runtime/callback-probe-mumu.json` 和 `commands/051-…`：exit 139，stderr 为 “Segmentation fault”，断言全部通过 | **成立，且如实标出边界**。注意探针源码（`04-collection/diagnostics/probes/debug-variant/source/ProbeCollectionCallbacks.java:128, 166`）是本地 `query.transact` 加动态代理服务，属于**本地回环**测试，验证的是本进程的 Parcel 和节点解析，不涉及真实系统回调。实施方也写了“不认证远端 IPC”。“快速失败在真实不兼容设备上有效”目前没有任何证据，因为根本没有遇到过不兼容设备。 |
| 热 worker 稀疏对照：5459 到 8192 节点，CLI 更久，压缩载荷略大 | `13-nodes/runtime/mumu/{dense,sparse}/costs.json`：稠密臂解码 8,382,033 字节，压缩 429,544 和 431,637；稀疏臂解码 6,224,689 字节，压缩 434,310 和 435,195。CLI 4.11 和 4.11 s 对 4.86 和 4.77 s。我的扫描也确认：稠密臂撞的是 `treeByteLimit` 加 `nodeLimit`，稀疏臂只撞 `nodeLimit` | **成立**，每臂 2 次。新增的 2733 个节点都在屏幕外：稀疏臂可见节点 74 个，稠密臂 59 个。 |
| 采样起点间隔 250–251 ms，事件间隔约 160 ms | `timing/result.md` 列了 17 个差值。原始 logcat 和 `association-and-intervals.json` **不在材料包里**（`EVIDENCE_INDEX.tsv` 里没有） | **原始日志核不到**。旁证：正式运行 029 在主 RPC 4.515 s 内采了 18 次，平均约 250 ms，与源码 `UiBudget.kt:58-61` 一致。我按源码判断这条声称为真。 |
| 节点上限原因码已区分 | 我的扫描中出现了 `nodeLimit` 18 份、`collectionLimit` 9 份、`treeByteLimit` 3 份 | **成立**。 |

总体看，失败、UNKNOWN、partial 都如实保留了，没有发现用单次成功概括普遍结论的地方。问题主要在判读层面：实施方把 D1、D2 这类“规则在长页上必然失败”的现象当作边界如实记录，却没有把它们当成产品问题。

---

## 5. 测试抽查

- **主机测试**【实跑】：`test_wire_contract.py`、`test_node_actions.py`、`test_ui_view.py`，298 个通过，0.54 s。其中 `test_sparse_defaults_and_action_names_match_the_device_contract`（`test_wire_contract.py:487-511`）用 Kotlin 导出的契约对拍了 Python 的 `DEFAULTS` 和 `ACTION_NAMES`，还覆盖了“`heading` 不可用时输出 null”“`text` 不可用时省略并记入 `unavailable`”“重复 ID 和自定义动作”等情况。这是真正的两端对拍：Kotlin 侧的 `WireContractTest.kt:214-244` 由生产代码 `NodeEncoding.node` 生成样本。
- **JVM 测试**（只读源码，没有运行）：
  - `CollectionRepliesTest`：7 个用例，直接驱动生产收件单元，覆盖截止前收到的空结果、旧预取失败唤醒当前等待者、事务常量缺失等。是行为测试。
  - `UiRefsTest.visibilityAndBoundsKeepDerivedNamesRowsAndRefsButContentChangesReject`：用 JSON 层面的语义构造，检查“只变可见性则 REF 保持，变名字或行标签则拒绝”。是行为测试。
  - `UiRefRepairTest`：只有 29 行，但用的是录制的真实候选池，不只是复述实现。
- **测试空白**：
  - 没有 JVM 测试覆盖 `TreeCollector` 的队列顺序（可见优先、按深度）和 `FreshNodes`，两者都依赖 Android 对象。
  - 没有测试覆盖 D1 的“部分树上选择器反复重采”，也没有测试覆盖 D3、D4。
  - Fresh 读取每读一个节点就要做一次 `getChild` 加一次 `refresh` 跨进程调用（缓存已关），一次准入读取最多约 64 个节点；这部分开销没有任何计时证据。

---

## 6. 对待决题的评价

**待决题 2“动作后仍见旧页”**（实施方推荐 A：保持现有算法）。

- 四个选项都是真实候选，推荐依据（最终四臂各 1 次、没有严格的“旧业务条目仍保持”）成立，但样本很小。
- 漏掉的一个选项：**让现有稳定判断在大页面上能工作。**现在长网页的动作后观察必然 `unsettled`，与旧页无关（D2）。这个缺口比“旧页”影响面更大，改动也更小，可以在 A 的框架内做。
- 关于“CHANGED 但业务仍在 Loading”的 5 例：设备端的“加载中”只认可见、不定进度的 `android.widget.ProgressBar`（`UiSemantics.kt:397-402`），网页上显示文字 “Loading” 不算。这是“已经变化，但未必就绪”的根源。我不建议扩大加载识别（容易误判），只建议在指引里写一句：“`changed` 不代表新页已就绪；导航类动作接 `wait --contains <新页专有文字>`”。
- **结论**：支持 A，加上 D1 的快速失败，以及 D2 的 REF 延续修正（实施细节）。

**待决题 3（页级动作字典）**：不在本专题深核范围。我只注意到动作字节占比（31–60%）里，有一部分来自屏幕外节点也带着动作。如果按 §3.2 只在快照时收屏幕外节点，动作后正文里的动作体量会自然变小。

---

## 7. 需要用户裁决的新取舍

### U1：partial 树上的选择器唯一性

| 候选 | 得 | 失 |
| --- | --- | --- |
| a. 维持拒绝，但第一次采样就快速失败并附带候选 REF（D1 的修正） | 不降低任何保护，失败从 5 s 以上降到一次采样；调用者能直接改用 REF | 仍然要多一次 REF 动作调用 |
| b. 当所有可见节点的子节点都读全时，接受唯一的可见匹配，回执标注 `uniqueness: visibleOnly` | 长网页上选择器能直接成功 | 不是严格证明：零高度不可见祖先下面可能有未读的可见节点（样本中真实存在）。只有当两个完全相同的可见目标中恰好一个落在未读部分时，才会点错 |
| c. 现状 | 无改动 | D1 的成本照旧 |

**推荐 a。**它其实是实施细节，可以直接做。是否进一步采用 b，应在 a 落地后看长页上选择器失败是否仍然常见，再决定。剩余未知：真机和其它浏览器上，可见节点挂在不可见祖先下的频率。

### U2：动作后和 wait 采样是否限制屏幕外体量

| 候选 | 得 | 失 |
| --- | --- | --- |
| a. 现状：所有采样都用 8192 上限 | 每份样本都能 `find` 屏幕外内容 | 长页上每个动作多 1–2 s，稳定判断必然失效 |
| b. 显式 `snapshot` 和 `find` 用 8192；动作后和 wait 采样以可见节点为主，屏幕外只收约 1000 个，并用“可见部分完整”判断稳定 | 长页上动作有望降到小页水平，稳定判断恢复 | 动作后的正文里查不到远处的离屏内容，需要显式 `snapshot`；要定义并测试“可见部分完整”；采样规则从一套变成两套 |

**推荐**：先做 D2 的 REF 延续修正（不需要裁决），再用 California 和一张原生长列表做 b 的原型，比较 CLI 时间和 settle 结论。证据不足时维持 a。

### U3：手写回调采集器

**不需要现在裁决。**维持用户此前的“保留”；今后若出现协议错误或需要大改，改走 `clearCache()` 加公开 `getChild` 的路线，并按 §3.1 的判据测量。

**以下属于实施细节，不需要用户决定**：D1 的快速失败与候选、D2 的 REF 延续、D3 的动作回调快速失败、D4 的未知事务码处理、D5 删掉尾部检查、D6 的基线年龄提示、§3.4 的命令合并（合并会改公开命令面，建议和指引精简一起做）。

---

## 8. 没能核实的内容

- 采样间隔的原始 logcat 与关联表不在材料包里，只有实施方的 review 文本和正式运行 029 的平均值作旁证。
- 真机和厂商 ROM 上回调接口与服务方法是否被改过；`collectionProtocol` 在真实不兼容设备上是否真能快速失败（探针只是本地回环）。
- Fresh 读取的跨进程调用次数和耗时（材料中没有分项计时）。
- Compose 的 LazyColumn 复用节点时语义 ID 是否会被重用（影响 §3.3 的“复用 View 碰撞”分析；按我的理解较新的 Compose 会重新分配 ID，未核对源码）。
- `clearCache()` 加公开 `getChild` 路线的实际性能与预取行为，从未运行。
- JVM 测试没有运行（没有 Gradle）。主机只跑了上述三个文件，用了 `-p no:cacheprovider`。事后核对：SRC 里已有的 `__pycache__` 目录都早于本次运行，`git status` 中没有出现新文件。
- 所有设备结论都只来自 MuMu（API35）和 Android Studio API34 模拟器，每个场景 1–2 次。
