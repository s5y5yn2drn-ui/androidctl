# 专题 26：元素采集路线（E9）与稀疏表示 —— 独立核查

核查范围：r3 主稿主线第 3 条、第 1 题（动作后主采集路线）、建议提交 4/5、底表 G2 前半。
术语：**E9** = 候选里手写的无障碍回调采集器（`CollectionNodes.kt`），不经公开 `AccessibilityNodeInfo.getChild`，而是反射调用隐藏的 `IAccessibilityServiceConnection.findAccessibilityNodeInfoByAccessibilityId`，自己实现回调 Binder、解析回包；**公开 cache** = 打开平台 `AccessibilityCache` 后用公开 `getChild(index, PREFETCH_DESCENDANTS_DEPTH_FIRST)`；**cache-off single** = 关缓存、公开 API 逐节点取（平台此时会把预取标志清掉）；**F21** = "本地截止被计成节点不可用"的诊断缺陷；**M2** = 实施方为 F21 修法写的设备端机制探针。
所有运行数据只来自 MuMu 模拟器（Android 15/API35，4 核/4GiB），Chrome Beta 158，一个 Wikipedia 页面族；没有真机、没有厂商 ROM。
本人写的脚本：`agents6/26/sparse_recount.py`；下载的 AOSP 文件：`agents6/26/aosp/`（android-14.0.0_r1 / 15.0.0_r1 / 16.0.0_r1 的 `IAccessibilityServiceConnection.aidl`，14/16 的 `IAccessibilityInteractionConnectionCallback.aidl`）。

---

## 1. 结论摘要

1. **第 1 题（动作后主采集路线）：同意推荐 A（保 E9、修 F21），但选项应收窄为 A 对 B 两项。** C（cache-off single）已被现有证据排除：动作后采集窗口实测只有约 1.39 s，而 single 在同一 4226 节点页上静态采集就要 1.9–2.2 s。D（暂缓）并不是一种实现。E9 的实测优势是系统性的，不是噪声：动作后滚动这个场景下，平台公开路径在收到滚动事件后约 150 ms 内会停止预取，所以 cache 臂的节点查询事务多了约 4 倍（1437 对 361），采集多花 285 ms，已用掉窗口的 71%。
2. **按评审方上一版事先写好的判据，本次结果其实落在"建议换公开 cache"一侧**：差距小于一次组装耗时，两臂都完整、新鲜。实施方保 E9 的理由（截止控制）只部分成立：根节点、Fresh 和 refresh 在两条路线里本来都走公开 API，平台对它们有固定 5 s 等待。我仍选 A，理由是上面所说的滚动余量，加上 E9 增加的隐藏 ABI 不多（约 5 个成员，和基线已有的动作回调同一模式；AOSP 14–17 签名都没变）。这是一次样本上的判断，用户若更看重删代码，选 B 也可接受。
3. **有比"尾兼容或降级"更便宜的必要补丁：E9 遇到回包不兼容时要快速失败，并报出明确诊断。** 现在回包布局一旦不符，回调会被静默丢弃：第一次孩子查询等到截止，此后每个孩子都记成"不可用"，结果里没有任何"协议不兼容"的字样。用户已把降级延后，但"能被发现"不该延后。
4. **F21：缺陷确定，修法简单明确。** 位置是 `CollectionNodes.kt:38,66,67` 加 `TreeCollector.kt:318-322`，约 40 行，研究副本已有可用实现。M2 的 exit139 是 MuMu 上 app_process 退出阶段的环境崩溃，同一窗口里 uiautomator 每次也都 exit139，不影响机制结论。
5. **稀疏数据并不说明"稀疏以后更慢"。** CLI 慢了 1.35 s，几乎全部能由两项研究假象解释：研究 host 把稀疏体重新展开成 12.5 MB 稠密 JSON 再交给旧消费者（0.891 s）；稀疏臂又是新 worker 的第 1 次采样，基线臂是热 worker 的第 11 次，设备端采样区间本身反而更短（2666 对 2774 ms）。首份 windows 附件"预算耗尽"也是这次展开把 host 的 5 s 截止吃掉造成的（重算见 §2.5）。
6. **稀疏后，稀疏臂首先撞上的是 8192 节点上限，而产品没有为这个上限写明原因**：缺口只表现为 2003 条 childGap 和 7117 个未读孩子。三个限额都应保留，但要补一个独立的节点上限原因码。8192 不宜上调，因为默认 5 s 请求里设备 RPC 已占 3.4–4.0 s。
7. **"不保 dense 展开层"要改写。** 设备内部继续用稠密 NodeFacts（现状就是这样），只让早期预算计量、最终定稿、签名、wire 和 Fresh 单节点共用同一个稀疏编码函数；host 在唯一的入口按共享默认值表补默认值（或用一个统一的读取函数）。只禁止把稠密形态发出去或持久化。评审方上一版"设备端严格读取全部改成带默认值读取"的要求没有必要，应撤回。
8. **建议提交 4/5 要重排：** 把"F21 加节点上限原因码"拆成一个独立的小提交，排在最前；路线替换只在用户选 B 时才有；稀疏提交的验收要去掉研究展开的影响，并加上"windows 附件在默认预算内交付"。

---

## 2. 逐题分析

### 2.1 E9 到底手写了什么；基线怎么采；不兼容时会怎样

**事实。**

- 【源码 3f45af6】`CollectionNodes.kt:33-72` 先取孩子的 sourceId（隐藏 `getChildId`），已收到的预取命中直接返回；否则反射调用 `IAccessibilityServiceConnection.findAccessibilityNodeInfoByAccessibilityId(int,long,int,callback,int,long,Bundle)`（`HiddenApi.kt:117-128`），带 `FLAG_PREFETCH_DESCENDANTS_DEPTH_FIRST`。回调是自写的 `Binder`（`CollectionNodes.kt:107-146`），按 Stub 的事务常量（`HiddenApi.kt:111-116`，反射读 `TRANSACTION_setFindAccessibilityNodeInfoResult/…Prefetch…`）分流，回包按以下顺序读：`enforceInterface` → `readTypedObject(AccessibilityNodeInfo.CREATOR)` 或 `createTypedArrayList(CREATOR)` → `readInt` 交互号 → **严格要求 `dataAvail()==0`，否则 `return false`**（第 130 行）。然后 `finalizeNode` 照抄平台的定稿步骤：隐藏的 `setConnectionId`、包名纠正、隐藏的 `setSealed`（第 85-94 行）。根节点仍走公开 `window.root`（`TreeCollector.kt:232`），leashed 节点退回公开 `getChild`（第 40 行）。
- **节点本体的 Parcel 布局不是手写的**：解析用的是设备自己的 `AccessibilityNodeInfo.CREATOR`。所以厂商若改 `AccessibilityNodeInfo` 的字段布局，只要系统两端一致，E9 不受影响。真正手写的部分有三处：回调事务的外层布局（节点或列表加一个 int）、服务方法的 7 参数签名、Stub 事务常量名。【源码】
- 【源码，与 AOSP 对照】`setFindAccessibilityNodeInfoResult(in AccessibilityNodeInfo, int)` 和 `setPrefetchAccessibilityNodeInfoResult(in List<…>, int)` 在 android-14.0.0_r1、15（包内副本）、16.0.0_r1、17.0.0_r1（包内副本）中完全相同；服务方法签名在 14/15/16 相同（`agents6/26/aosp/`）。候选的最低版本是 Android 14。
- 【源码，git】基线 `7d5be70` 不是这样采：`TreeCollector` 调 `NodeAccess.child`，API 33+ 走公开 `getChild`，缓存关闭，等于 cache-off single，`COLLECT_MS=1500`。E9 是候选在 `ab7ff63 feat(observation): collect larger trees with scoped prefetch` 里新加的，新增约 186 行 `CollectionNodes.kt`；`HiddenApi` 新增 `setNodeConnection`、`setNodeSealed`、`nodeTransaction`、`prefetchTransaction`、服务端 `findNode` 约 5 个成员，复用了已有的 `childId/leashedNodeId/sourceNodeId/callbackInterface`。**基线本来就有同模式的手写回调**：`AccessibilityConnection.kt` 里的 `ActionCallback`，执行动作时同样手解 `readInt/readInt` 并做严格尾检查。所以 E9 新增的是"同一类 ABI 再多一处"，不是引入一类全新的责任。
- 【源码推断】各种不兼容时的表现：
  - 服务方法签名变了：反射 `NoSuchMethodException` → `addNode` 的 catch 记 `collectionFailed` 再抛 → 外层写 `treeProblem`。整次采集失败，但**有明确错误**。
  - 回调外层布局变了（例如在末尾追加参数），或事务常量改名（常量在 Binder 线程里的 `onTransact` 中惰性取，抛出的异常会被 oneway 事务吞掉）：回调被**静默丢弃**。第一次孩子查询等满剩余截止，之后的孩子查询都因截止已到立刻返回 null，结果只剩各窗口根节点，外加 `unavailableChildren` 计数，**没有任何指向协议不兼容的诊断**，读者看到的像是"App 不回应"。这正是值得补的地方。
  - `AccessibilityNodeInfo.CREATOR` 本身失败：反序列化异常同样在 Binder 线程被吞，表现同上。
- 【需设备】厂商 ROM 是否改过这两个 AIDL，没有任何数据。全部验证只在 MuMu 和 Android Studio 模拟器上做过。

**判断。** E9 的长期责任是：一个服务方法签名、两个回调事务的外层布局、两个事务常量名、定稿三步照抄平台，外加 186 行并发和截止逻辑。从 AOSP 14 到 17 都稳定，和基线已有的动作回调属于同一类风险。真正的短板是**不兼容时静默**，而不是缺少降级。

### 2.2 E9 对公开 cache 的实测对比

**核到的原始数字。** 原件是 `P/r2-nodes/runtime-mumu/commands/{scroll-e9,scroll-cache,expand-cache,expand-e9}{,-log}/`，我直接读了 logcat 里的 `AndroidCtlCollector` JSON 和 `command.json`。【原件】

| 命令 | collect（collectedMs−preparedMs） | 事务 started/completed | 采集截止−开始 | 完整 CLI |
| --- | ---: | ---: | ---: | ---: |
| scroll-e9 | 1592615−1591910=**705** | 361/361 | 1390 | 2.938 s |
| scroll-cache | 1645933−1644943=**990** | 1437/1437 | 1406 | 3.109 s |
| expand-cache | 1717434−1716550=**884** | 980/980 | 1390 | 2.891 s |
| expand-e9 | 1771618−1770715=**903** | 883/883 | 1386 | 2.860 s |

主稿和底表里的 705/990、903/884、361/1437、2.938/3.109 s 与原件逐一相符。展开对的事务数（883 对 980）主稿没有写，E9 在这一对里也略少。

- **样本量**：每个动作、每条路线各 1 次，共 4 次；一个页面（Chrome Beta 的 `List_of_national_capitals`，4226 节点，展开前态 210 节点）；一台 MuMu API35；worker 是热的（序号 9/11/16/22）。没有更大的页面、更慢的 App、真机，也没有动作前（selector/Fresh）路径的对比，因为两臂的动作前路径都固定为公开 cache-off。【原件】
- **"四条均 unsettled"** 的意思是：动作后的稳定检查在 1500 ms 的 settle 窗口内一直没看到页面安静下来，`settleMs` 依次是 1939/2125/1904/1876。回执交付的是窗口末尾那一份样本，不能证明页面已经稳定。影响：两臂受同样影响，不改变相对比较；但"新鲜"只能靠最终树加独立截图核实，不能推到中间样本。Chrome 一直在发事件，这一点对 cache 臂其实更"真实"，因为缓存会被事件持续逐出。【原件】
- **实施方没有强调的关键数字**：动作后的采集截止只有约 1.39 s（`collectionDeadlineMs−startedMs`），不是静态快照的 3000 ms。cache 臂滚动采集用掉窗口的 71%，E9 用掉 51%。四份整份样本都越过了采集截止（75/323/191/184 ms），cache 滚动越得最多。【原件】
- **滚动差异的机制**【源码 AOSP15 `AccessibilityInteractionClient.java:116,648-651,1594-1606`】：公开路径在窗口收到 `TYPE_VIEW_SCROLLED` 后 `getSendRecurringAccessibilityEventsInterval()*1.5`（默认约 150 ms）内会清掉预取标志；`AccessibilityCache` 也会因滚动和内容变化事件逐出子树。E9 直接调连接，不经过客户端这层判断，所以滚动后仍能拿到批量预取。这能解释 4 倍事务差，但事件时序没有实际测过，只能算源码推断。【源码＋推断】
- **更早的静态对比**【原件 `P/collector/runtime-mumu-clone/result.md`、`P/collector-cache/runtime-mumu-clone/result.md`】：同一页 4226 节点，E9 采集 1024/399 ms，cache-off single 1909/2209 ms；另一轮 4243 节点，E9 606/580 ms，cache 655/1047 ms。各格 1 次，首份 raw 有缺口。结论：静态时 E9 比 single 快约 2–5 倍，与 cache 接近。
- **"公开查询等待不完全服从本地剩余预算"具体是什么**【源码 AOSP15 `AccessibilityInteractionClient.java:114` `TIMEOUT_INTERACTION_MILLIS = 5000`，`waitForResultTimedLocked` 第 1414-1438 行】：公开 `getChild/refresh/getRoot` 每次查询最长同步等 5 s，调用方无法缩短。研究 cache 臂只能在返回之后检查截止。E9 用 `ready.await(until-now)`（`CollectionNodes.kt:160-170`），只等剩余预算。
  - **这是实质优势，但只覆盖孩子查询。** 根节点（`TreeCollector.kt:232`）、Fresh 的 root/refresh/parent/getChild（`FreshNodes.kt:21,38,53,67,81`）在两条路线里都走公开 API，同样可能一次卡 5 s。所以 E9 不能宣称"整次采集受截止控制"，只能说"大量孩子查询在 App UI 线程繁忙时不会每个都卡满"。动作后恰恰是 App UI 线程最可能繁忙的时刻，所以这一点有价值，但没有任何实测（四次都没有出现卡顿）。【源码＋推断，需设备】

### 2.3 判断：证据够不够选路线；收益和责任怎么权衡

- **C 可以排除**：single 在这个页面上静态就要 1.9–2.2 s，而动作后窗口只有 1.39 s，会变成 partial。这是能力损失，不是成本问题。【原件＋重算】
- **A 对 B**：
  - B 的好处是删掉回调解析、服务方法反射和定稿照抄（约 150 行加 5 个隐藏成员）。但 B **并不脱离隐藏 API**：开关缓存要用 `UiAutomation.getCache`（API35+）或 `AIC.getCache` 加 `setEnabled`，候选现在为了关闭缓存已经在用。B 还新增一个必须守住的不变式：缓存只在整树采集的短窗内打开，并且一定在 finally 里关闭；同一连接上其它公开读取（Fresh、wait 检查）不得在这个短窗内命中旧缓存。B 也继承了 5 s 固定等待和滚动后不预取。
  - A 的好处是滚动后余量更大，事务约少 4 倍，孩子查询受截止控制。代价是上文 §2.1 列出的 ABI 责任。
  - **按评审方上一版定稿事先写好的判据**（`/home/user/androidctl/review/androidctl-candidate-route-review-3f45af6.md` §3 表）：差距小于一次组装（约 665–758 ms），完整率相同，新鲜度没有问题，就"建议换公开缓存"。这次结果满足该判据，实施方自己在 `R/r2-nodes-waits.md` §1.3 写的判据也一样。实施方随后改成推荐 A，但没有正面承认这和事先判据相反。我认为那个判据有缺陷：它只拿组装耗时当尺子，忽略了动作后窗口只有 1.39 s 这个更紧的约束，以及滚动差异有机制性原因。按"余量占窗口比例"衡量，E9 的优势是实在的。
- **我的推荐：A，加一个"快速失败诊断"**，不做尾兼容，也不做降级。理由：①收益集中在最常见的"滚动后看"路径，而且有机制解释；②新增 ABI 与已有动作回调同类，AOSP 14–17 稳定；③静默失败的缺口用约 20 行就能补上：`onTransact` 里事务码匹配但布局校验失败或反序列化抛异常时，记一个"回调布局不兼容"标志，`child()` 立刻抛出明确的 `collectionFailed`（问题码例如 `nodeCallbackLayout`），不等到截止。这样整次采集失败并带诊断，厂商 ROM 的问题能被用户和维护者马上认出来，以后决定要不要降级也有了依据。用户在 U27 里明确说"E9 尾兼容/降级延后""E9 去留由用户据结果定"（`ROOT/ultragoal/2026-11-11-next-improve/decisions.md:38`），这个补丁和该指示不冲突。【设计建议】
- **选项之外的方案**："公开 cache 为主、E9 不要"就是 B。"E9 为主、确定不兼容时同一连接内退到公开路径"要承担两条路径的长期测试负担，目前没有一例真实不兼容，不值得先做。先做快速失败诊断，等真有设备报出 `nodeCallbackLayout` 再决定降级到 cache-off single 还是 cache。
- **哪些实验结果会改变结论**：
  - 一台真机或厂商 ROM 上 E9 回调不兼容 → 立即需要降级或改 B。
  - 一台慢真机、5k 以上节点的页面，滚动后 cache 臂 partial 而 E9 full → A 更稳。
  - 若真机上两臂在滚动后也接近（例如事件时序不同导致预取不受抑制）→ B 的维护优势胜出。
  - 一个 UI 线程繁忙的 App 在动作后让公开路径单次卡满 5 s、请求整体超时，而 E9 交付 partial → 加强 A。

### 2.4 F21：截止归因缺陷

- 【源码 3f45af6】`CollectionNodes.kt:38`（查询前已到截止 → null）、`:64-66`（`awaitPrimary` 等到截止仍没有回调 → null）、`:67`（有效 primary 在截止后才返回 → null），这些 null 与真正的"App 回了 null"、`packages==null`（:59-62）混在一起。`TreeCollector.kt:318-322` 把所有 null 都记成 `unavailable++`，最终写进 `unavailableChildren`。另外 `Query.packages()` 第 150 行在过了截止时调用 `reject()`，会把已经收到的 null 回包也清掉（这就是 M2 那个边角情形）。**缺陷确定。**
- 修法【原件 `P/r2-nodes/research.patch` 第 119-354 行、`review-fixes.patch`】：确知是本地截止的三个出口抛 `LocalCollectionDeadline`；`addChildren` 捕获后 break，当前及之后的孩子记为未读，并补 `collectionLimit` 原因；`reject(preserveDeliveredNull=true)` 保留已收到的 null。总共约 40 行，没有新状态，不重试。**修法简单明确。** 小提醒：`collectionLimit` 同时用于时间截止和 `nodes.size` 达到上限（`TreeCollector.kt:94,139`），见 §2.5，建议一起区分。
- **M2 与 exit139**【原件 `commands/m2-primary-deadline/stdout.bin`、`m2-crash-log-tail/stdout.bin`】：探针在 app_process 中用真实 Parcel 和本地 Binder 跑了 null/valid/absent 三种情形，`queryCalls` 都是 1，输出 `passed=true` 后，进程在 Shutdown thread 的 `__cxa_finalize` 处 SIGSEGV，退出码 139。同一日志里 uiautomator（PID 3203、4755）也是同样的崩溃栈，本窗 4 次 `management-dump` 全部 exit139。这是 MuMu 上 app_process 退出阶段的环境问题，**不影响 F21 机制结论**，也和 E9 路线选择无关。它也没有覆盖常驻 worker 的退出行为，那条没有测过。【原件＋推断】

### 2.5 稀疏表示：5485→8191 是否说明"稀疏更慢"；限额是否合理

**核到的原始事实**【原件 `P/r2-sparse/runtime-mumu/raw/*.tree.json`，用 `agents6/26/sparse_recount.py` 重算】：

| | 基线 c669 | 稀疏 52b3 |
| --- | --- | --- |
| 树字节 | 8,382,705（离 8 MiB 只差 5,903） | 6,185,243 |
| 节点 | 5485 | 8191 |
| 平均每节点字节 | 1505 | 738 |
| partial / missing | true / 1902（childGap 1900、treeByteLimit 1、collectionLimit 1） | true / 2003（全部是 childGap） |
| 未读孩子合计 | 7002 | 7117 |
| 样本区间（startedMs→finishedMs） | 2774 ms | 2666 ms |
| 主 RPC / deliveryMs / CLI | 3.406 s / 1264 / 5.234 s | 3.984 s / 2249 / 6.579 s |
| worker / capture | b4838d80…:6，序号 11（热） | 6a8516db…:1，序号 1（新 worker 首采） |

各项与主稿和底表一致。【原件＋重算】

**"稀疏以后反而更慢"不成立，慢在哪：**

1. **研究 host 的稠密展开**【原件 `P/r2-sparse/host_cli.py:19-46`、`capacity-facts.json`】：wrapper 在 `UiClient.blob` 内把稀疏体展开、重新编码成 12,478,962 B 的稠密 JSON，交给旧消费者再解析一遍，实测 `expansionSeconds=0.891`。deliveryMs 多出 985 ms，其中 891 ms 来自这里；扣掉后约 1358 ms，与基线的 1264 ms 接近。这一步是**原型的双表示假象**，产品里不应存在。【原件＋重算】
2. **首份 windows 附件预算耗尽，也是这次展开造成的**【源码 `androidctl/src/androidctl/ui_client.py:651-668`，原件 `commands/sparse-california-expanded/trace.json`】：host 按 tree→windows 的顺序取附件，每次取之前检查 `self.deadline`。trace 显示 snapshot RPC 在 62154.062 开始、用时 3.984 s，tree 的 7 次 ui.read 在 62158.156 结束（约 4.09 s），再加上展开的 0.891 s，到约 4.98 s 时才轮到 windows，已经越过 5 s 截止。没有这次展开，windows 会在约 4.1 s 取到。【重算＋推断】
3. **设备 RPC 多出的 578 ms 不在采样区间里**：采样区间反而短了 108 ms。差额落在采样之外，稀疏臂是新 worker 的第 1 次采样（连接建立和冷启动开销），基线是热 worker 的第 11 次。这个混杂因素实施方和内部 review 都没有指出。【原件＋推断】
4. 多采的 2706 个节点会让组装、发布和 host 处理按节点数线性增加，这部分成本是真的。但两臂的采样区间相近，说明在 8191 节点规模下它不占主导。【推断】

**稀疏后谁先封顶**【原件＋源码推断】：稀疏臂没有 `treeByteLimit` 也没有 `collectionLimit`，只有 2003 条 childGap 和 7117 个未读孩子；采集时间远低于 3000 ms（整个样本区间 2666 ms，里面还含组装）。对照 `TreeCollector.kt:315` 的 `discovered < Limits.NODES` 循环条件，**先封顶的是 8192 节点发现上限**。实施方写的是"8191 不足以判断早门先后"，这过于保守：时间和字节都能排除，剩下的只有节点上限。但**产品没有为节点上限写原因码**：孩子发现循环在这里停止时只留下未读计数；主循环在 `nodes.size` 达到 8192 时又和时间截止共用 `collectionLimit`。这是一个真实的诊断缺陷。【源码】

**三个限额是否还合理：**

- **8192 节点**：保留，不上调。默认 5 s 请求里设备 RPC 已占 3.4–4.0 s，上调只会让附件交付更早撞上截止。它同时限制设备端常驻的 `AccessibilityNodeInfo` 句柄和 JSONObject 内存。【推断】
- **3000 ms**：保留。动作后窗口实际约 1.39 s，时间是真正的约束。
- **8 MiB**：保留，作为长文本页的安全阀。实施方构造的 8192 节点 × 600 字符长标签样例，早期预算下界 6.5 MB，完整稀疏 payload 13.6 MB【原件 `P/r2-sparse/README.md`，未亲自重算】，说明字节上限在长文本页上仍可能先到。这个检查成本很低。
- 结论：不改数值，**补一个节点上限原因码**（例如 `nodeLimit`），把 `collectionLimit` 收窄为只表示时间截止。【设计建议】

**"长期不保 dense 展开层"现实吗：**

- 【源码统计】host 端直接读节点字段的代码集中在 `ui_view.py`（约 48 处）、`ui_diff.py`（约 8 处）、`ui_client.py`（约 5 处），另有 6 个测试文件用到节点字段。设备端有 21 个 Kotlin 文件用严格的 `getBoolean/getJSONArray` 等读 JSON，其中相当一部分读的是 NodeFacts。
- 更简单、也更干净的做法：**设备内部继续用稠密 NodeFacts**（现状如此，实施方 §2.2 也说早门只用同一编码函数做非破坏性计量）；稀疏只出现在这个唯一的编码函数里，早期预算、最终 fit、签名、wire、Fresh 单节点都调它。host 在附件解析的唯一入口按共享默认值表把缺失的键补上（在内存里补，不再重新编码成 JSON），后续消费者不改。真正要禁止的是两种格式同时上线、或保存文件里出现稠密体，不是内存里的默认值补全。这样设备端那 21 个文件不用改。评审方上一版"设备端严格读取全部改为带默认值读取"的要求可以撤回。【设计建议】

### 2.6 过度复杂化、可删可并的机制

见 §4。另外主线第 3 条的措辞堆满了内部术语（NodeFacts、Fresh、canonical、unknown/unavailable），对用户来说实际只有两个决定：路线 A 还是 B；稀疏化做不做（这一点其实不需要问）。

### 2.7 建议提交 4、5 的边界与修订版

实施方原版：4 是一次性全量的"canonical 稀疏迁移"，覆盖 wire、host 和全部消费者；5 是"按第 1 题选定的路线保截止"，不做尾兼容。问题在于：F21 和节点上限原因码与路线选择无关，却被绑在 5 里等用户裁决；4 的验收没有排除研究展开和冷 worker 的混杂；4 要求"全部消费者"迁移，代价被放大。

修订版（按顺序）：

| 提交 | 内容 | 依赖 | 验收 |
| --- | --- | --- | --- |
| 4a `fix(collection): attribute local cutoffs and node limits` | F21（沿用研究修法，含保留已收到的 null）；新增节点上限原因码，`collectionLimit` 只表示时间；E9 回调布局不兼容时快速失败并带明确问题码。 | 无，可最先做 | JVM：本地截止、已收到的 null、节点上限、布局不符四类分别出不同的 missing 或 problem；不重试。设备：动作后一次真实滚动，`missing` 与 `unavailableChildren` 的语义正确。 |
| 4b `refactor(observation): encode nodes with one sparse function` | 共享默认值表（契约生成，Kotlin 和 Python 同源）；动作先排序，再生成标签下标；一个编码函数同时服务早期预算、最终 fit、签名、wire、Fresh 单节点；host 在唯一解析入口补默认值；`unavailable` 保留未知值。设备内部 NodeFacts 不改。 | 4a（原因码） | 契约样本覆盖等于默认值、unknown、空串和 null 标签、重复动作 ID；同一界面两次采样签名相同；整树与 Fresh 读出的同一节点编码相同。设备：在原先撞 `treeByteLimit` 的真实页面上保留全部旧 source，并给出明确的首个封顶原因；**在热 worker 上、无研究展开时，windows 附件在默认 5 s 内交付**；不宣称速度提升。 |
| 5 `refactor(collection): read child nodes through the public cache`（**只在用户选 B 时才有**） | 删除 E9 回调，缓存只在整树短窗打开并在 finally 关闭；短窗期间其它公开读取不命中旧缓存。 | 4a | 一对真实滚动和展开，动作后完整、新鲜；记录超出窗口的比例。选 A 时这一项不存在。 |

---

## 3. 确定缺陷与证据不足的断言

**确定缺陷（候选源码）：**

1. F21：本地截止被记成 `unavailableChildren`，过了截止的 `reject()` 还会清掉已收到的 null（`CollectionNodes.kt:38,64-67,150`；`TreeCollector.kt:318-322`）。【源码】
2. 节点发现上限没有原因码；`nodes.size` 达到上限时又与时间截止共用 `collectionLimit`（`TreeCollector.kt:94,139,315-337`）。稀疏臂 8191 节点的封顶在回执里只能靠推断看出。【源码＋原件】
3. E9 回调布局不兼容或事务常量缺失时静默丢弃，整次采集退化为只剩根节点、每个孩子都记"不可用"，没有协议层诊断（`CollectionNodes.kt:125-130`）。基线的 `ActionCallback` 也是同一模式。【源码推断】

**证据不足或表述有偏的断言（实施方材料）：**

1. 推荐 A 的理由"保已有逐请求截止"说得过宽：根节点和 Fresh 在两条路线上都受公开 5 s 等待影响。【源码】
2. 推荐 A 时没有说明结果其实满足事先写好的"建议换 cache"判据；也没有用"动作后窗口只有约 1.39 s"这个更有力的理由。【原件】
3. "稀疏臂 CLI 更慢、首份 windows 预算失败"被列为稀疏方案的剩余风险，却没有指出这两项主要来自研究展开（0.891 s）和冷 worker 首采这两个混杂因素。【原件＋重算】
4. "8191 不足以判断早门先后"过于保守：时间和字节都可排除，先到的是节点上限。【原件＋源码推断】
5. 所有关于路线的数字都是每格 1 次、一个页面、一台模拟器。滚动差异的机制解释（150 ms 不预取）没有实际事件时序的直接证据。【原件】

---

## 4. 值得删除或简化的机制

1. **第 1 题的 C、D 选项**：C 已有排除证据，D 不是一种实现。题目收窄为 A/B 两项。
2. **研究计量绝不能进产品**：`ResearchNodeTransport` 的事务计数代理、`researchMode` 分流、`AndroidCtlCollector/AndroidCtlSparse` 研究日志、cache 启停事件数组（`P/r2-nodes/research.patch`、`P/r2-sparse/research.patch`）。提交 4/5 的验收不要要求复现"1/1/1/0/1 校准"这类研究仪表。
3. **host 稠密展开 adapter**（`P/r2-sparse/host_cli.py`、`sparse.py`）：只是研究用的双表示，产品用"入口处补默认值"替代，不再重新编码成 JSON。
4. **设备端全量改为读稀疏记录**（实施方"不保 dense 展开层"和评审方上一版"严格读取全改为带默认值"的组合）：不需要。内部保持稠密，只统一编码函数。
5. **尾兼容加自动降级**：用快速失败诊断代替，降级等出现真实不兼容再说。
6. （小项，实施细节）host 取附件的顺序可以改成先取约 1 KB 的 windows、再取 tree，避免大树把小附件挤出截止（`ui_client.py:651-655`）。

---

## 5. 需要用户裁决的事

**唯一需要问用户的：动作后整树采集用 E9（A）还是公开 cache（B）。**

- **A 保 E9（推荐）**：滚动后采集余量更大（实测事务少约 4 倍，占动作后 1.39 s 窗口的 51% 对 71%），孩子查询受剩余截止约束。长期维护约 186 行手写回调、5 个左右隐藏成员，以及服务方法和回调事务布局的版本核对（AOSP 14–17 未变）。要配快速失败诊断，不做降级。
- **B 公开 cache**：删掉大部分手写回调（不能脱离全部隐藏 API，开关缓存仍要用），要维护缓存短窗的开关不变式；滚动后预取会被平台抑制，孩子查询单次最长等 5 s。
- 剩余未知：真机和厂商 ROM 上的兼容性与事件时序、更大页面和慢 App 下的分布、UI 线程繁忙时公开 5 s 等待的实际发生频率。现有证据只有每格一次。

**不需要问用户、由实施者直接定的（实施细节）：** F21 修法；节点上限原因码；快速失败诊断；稀疏"同一编码函数"（这是正确性要求，不是偏好）；三个限额保持不变；host 入口补默认值的方式；附件读取顺序。C 和 D 两个选项删除。

---

## 6. 未能核实的内容

- 滚动后公开路径预取被抑制、缓存被逐出的**实际事件时序**：只有源码依据，原件里没有事件时间线。
- 厂商 ROM 上 `IAccessibilityInteractionConnectionCallback` 和 `IAccessibilityServiceConnection` 是否改过：【需设备】。
- 稀疏 8192 × 600 字符构造样例的 6.5 MB / 13.6 MB：只读了实施方 README，没有自己重算。
- 稀疏臂主 RPC 多出的 578 ms 是否全部是冷 worker 开销：日志里没有连接建立的分段计时，只是推断。
- 基线臂 803 个"最终 fit 裁掉"缺口的判定规则（`baseline-gates.md`）：没有逐条复核，只核了 missing 总数和原因分布（我重算得到 805 条无 ref 的 missing，实施方筛出的 803 条另外还要求 window=74 并带孩子计数，口径不同，没有深究）。
- 常驻 worker 进程在 MuMu 上正常退出时是否也会出现 exit139：没有数据。
