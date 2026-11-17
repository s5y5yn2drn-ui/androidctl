# 专题 01：节点架构与采集（方向 1、11、12、13，及与 2/29/38 的组合）

评审对象：候选 HEAD `3f45af6`（基线 `7d5be70`）。行号均取自 `3f45af6`。
证据标注：【实测】=包内原件记录的真实运行（注明版本）；【源码推断】=读源码/AOSP 得出；【设计建议】=本评审提议。
上游核对：本评审实际下载了 AOSP `android-14.0.0_r1 / 15.0.0_r1 / 16.0.0_r1` 及 `main` 的 `AccessibilityInteractionClient.java`、`AccessibilityCache.java`(15)、`UiAutomation.java`、`IAccessibilityServiceConnection.aidl`、`IAccessibilityInteractionConnectionCallback.aidl`（存于 `agents/01-aosp/`）。Chromium 的 Chrome 自动关闭无障碍逻辑**未**由本评审核对上游，只引用包内实测日志。

---

## 1. 结论摘要（重要在前）

1. **架构 A（worker 内 UiAutomation 作唯一节点宿主）本身是对的，B（普通服务唯一宿主）不该复活。** 普通服务对 Chrome 无论如何都必须存在（UiAutomation 不在“已启用服务列表”中，Chrome 约 5 秒后关掉网页无障碍树，包内 E10 日志实测），所以“候选同时跑两者”不是 A 的额外成本。B 真正能省的是 UiA 生命周期、约 10 个节点相关 hidden 成员和 E9。代价是把节点/输入/观察/交付拆到两个进程，再改写身份基线；同时还会失去 tool=true 才能看到的 dataSensitive 窗口。Bitwarden 一例的机制推理成立。按 Android 14 的自动推导，这类窗口覆盖所有对根视图开 `filterTouchesWhenObscured` 的 App，也就是常见防点击劫持做法的金融、密码、2FA 类 App，价值不小。
2. **“A 必须带 E9”的依据比交付稿说的弱很多。** 交付方据以判定“single 不够、必须 E9”的只有 J67 一份样本：HEAD single 用 3157ms 取得 3869/partial。但同一台 MuMu、同一 Chrome 长页上，J54 用同一条 AIC 公开单节点路径（cache 关、无预取），4213 次 child 查询累计只花 1040–1075ms（event0）和约 800–900ms（relevant），每节点约 0.25ms，比 J67 的约 0.8ms/节点快 3 倍。这个 3 倍差异没有任何内部材料解释，内部 review 也没有对比。如果真实 single 在约 1s/4.2k 节点，那么不带 E9 的 A 在 3000ms 内就能拿到 full 树，E9 只是约 1.5–2.5 倍的性能优化，不是“完整性前提”。**E9 的处置应为“证据不足”，不是“保留”。**
3. **E9 是一条完整的自建 Binder 回调链，也是本专题最大的长期平台风险。** 它绕过 `AccessibilityInteractionClient`（下称 AIC），直接调用 hidden `IAccessibilityServiceConnection.findAccessibilityNodeInfoByAccessibilityId`。查询回调由自建 `Binder.onTransact` 手工解析 AIDL Parcel 布局，并对多余字节严格拒收；AIC 的 finalize 步骤（setConnectionId、包名校正、setSealed）也在应用层重写了一遍。14/15/16/main 的 AIDL 签名一致，包内 API37 也实际跑通，所以短期没有问题。但只要平台在回调末尾增加一个参数或一个 finalize 步骤，采集就会退化成“只有根节点、所有子节点 unavailable”，且不会报错。动作回执 `ActionCallback` 的风险性质相同，但它在基线中已有。
4. **还有一个更简单的预取方案没有被公平测过。** 方案是：只在采集期间启用平台 cache，开头 clear，用公开 `getChild(i, FLAG_PREFETCH_DESCENDANTS_DEPTH_FIRST)`，结束时再关 cache。作者以“不能保证每个节点只来自本次采集”否决了它。我核了 AOSP 15：`AccessibilityCache.add` 在禁用时直接 return，所以残余风险只剩“禁用→下次启用”交界处在途的毫秒级数据。这些数据在语义上与“采集开始时读到的”没有区别，何况 3 秒的树本来就不是原子快照。这是用户警告过的那类为极窄区分增加机制的情形。
5. **方向 11（ee841ea 节点可选事实）让 Chrome 页面的默认行输出几乎翻倍，与方向 7 的阅读效率目标直接冲突。** 我实测了包内 S0 默认行文件（`stages/09-node-facts/runtime-combined/mumu/oracles/long-s0.lines.txt`）：共 10,097B，`chromeRole` 出现在 80 行，其中 32 行节点**只因** `chromeRole`（或其它新元数据）才没有被折叠成空布局。去掉 7 个新字段并折叠这些行后约 5,535B（-45%）。`tooltipText` 多与 name 相同，`roleDescription` 是本地化后的角色名（如“链接”“标题 2”），与 role 重复。内部 review 没有发现这个问题。
6. **方向 1（最低 Android 14）确实减少了兼容代码**：删掉 NodeAccess、API30–32 leash 反射、recycle 记账、旧 trust/connect/坐标门，节点相关 hidden 成员少了约 15 个。但 E9 随后又加回了 6 个节点 hidden 成员（`LEASHED_NODE_ID` 私有反射、`setConnectionId`、`setSealed`、两个 `TRANSACTION_*`、`findNode`），节点 hidden 面大致回到原规模。残留分支方面，`AccessibilityConnection.kt:36` 的 14/15+ cache 分支多余，因为 15/16 的 `UiAutomation.getCache()` 内部就是调用 `AIC.getCache(connectionId)`，统一用后者即可；另外 `VisualInput.kt:49` 留下了恒为 true 的 `apiSupported` 字段。
7. **8192 节点上限可以保留，但它不是收益来源，也不是主要成本来源。** 小页面（Settings 约 51–119 节点）不受影响，默认行阅读（只投影可见节点）也基本不随节点数增长（4214 节点页约 1K tokens）。不过在 Chrome 上真正起约束作用的是 8MiB，不是 8192：ee841ea 之后每节点约 1.31KB，8MiB 只容纳约 6.4k 节点，collector 却会采到 8192 个再由 `UiTreeBudget.fit` 截掉。用户对“3.3s 不快于 1024+ref 3.1–3.2s”的理解是对的：8192 换来的是完整事实和不需要滚动查找的定位，不是速度。我不建议改回“默认小、显式子树全量”。那会引入新的采集 scope 语义、子树内唯一性语义和调用者判断，复杂度高于固定上限。
8. **跨进程链（WorkerConnection/EditorBridge）只为 IC 存在，而且是必要的**：InputMethod 只能在 App 进程的 AccessibilityService 里获得。Chrome 声明这件事本身不需要任何 IPC，因为服务是被动的，只要 Bound 就行。若 IC 被删除，这条桥可以整体删掉，服务保留即可。
9. **长期最大维护风险依次是**：(a) E9 和 ActionCallback 的手工 Parcel 回调链及重写的 AIC finalize；(b) Chrome 依赖普通服务时，靠的是 Chrome 内部的服务判定启发式（当前 cap0 加相关 event mask 能在 Beta 158 上工作，Chrome 每 4 周发版都可能改变）；(c) UiAutomation 构造与收尾所用的私有/deprecated 成员（`UiAutomation(Looper, …)`、`mRemoteCallbackThread`），其中 `mRemoteCallbackThread` 位于清理关键路径上，没有做隔离。

---

## 2. 负责方向处置表

| 方向 | 处置 | 依据 | 依赖/真实能力损失 | 维护代价 |
| --- | --- | --- | --- | --- |
| 1 最低 Android 14 | **保留**；小调整 | 确实删除 11–13 的 leash/recycle/refresh/connect/coordinate 分支；`dataSensitive`、`injectInputEventToTarget` 本就需要 34。 | 失去 11–13 设备（用户已决）。 | 调整：删掉 `AccessibilityConnection.kt:34-41` 的版本分支，统一用 `AIC.getCache(id)`；删除 `VisualInput.kt:49` 的常量 `apiSupported` 及 `docs/reference/connection.md:105` 的描述。注意 Android 14 下限不降低**向前**（15/16/17）的 hidden 兼容责任。 |
| 11 节点可选事实 | **调整** | 7 个字段只有阅读投影（`ui_view.py:134-145`）和 `UiEffect.kt:140` 变化维度两类消费者，没有 selector/find/REF 消费者。在 Chrome 上 `chromeRole/tooltipText/roleDescription` 大多与 role/name 重复，并让空布局行无法折叠。 | 保留 `paneTitle`、`heading`、`targetUrl`（链接目的地对 Web 任务有实际价值）以及 `dataSensitive=true`。 | 行投影：去掉 `chromeRole`；`roleDescription`/`tooltipText` 只在与 role/name/detail 不同时显示；`targetUrl` 只在 target/`--all` 行显示；新元数据**不得**单独阻止空布局折叠。raw 可考虑省略 null/false，单节点约 -120B。 |
| 12 Browser service | **保留** | Chrome 无已启用服务列表时约 5s 关树【实测：`research/observation/device-results.md` E10 段日志 services=[]】；UiA 不进这个列表，任何 UiA flag 都改变不了。cap0、tool=false、不读内容，代码小（`BrowserAccessibilityService.kt` 86 行）。 | 用户必须手动启用；侧载 App 需先解除“受限设置”；启用后全局 AccessibilityManager 状态变化，部分风控 App 可能因此改变行为（未验证）。 | 主要责任在 Chrome 启发式的外部漂移。IC 也挂在这个服务上（方向 29），命名“browser support”略不准确，可接受。 |
| 13 A+E9、8192/3000ms/8MiB | **A 保留；E9 证据不足（倾向替换）；8192 保留** | A vs B 见 §3.2；E9 的必要性只有单样本支撑，且与同设备同 API 的 J54 实测相矛盾（§3.3）。 | 去掉 E9：超大页（>约 4k 节点）在 3s 内可能 partial，但 partial 只出现在**不可见**区域（collector 按 hidden→depth 优先，`TreeCollector.kt:63`）。 | E9：186 行 `CollectionNodes.kt` + 6 个 hidden 成员 + 手工 Parcel 协议 + 每次 collect 的 gate/封口。 |
| 组合 2（FLAG_SECURE） | 一致 | 节点侧把自身包窗口标为 `protected` 且不读取（`TreeCollector.kt:240`），与像素侧一致。普通服务 cap0 不读内容，不扩大自身暴露面。 | — | 无新增。 |
| 组合 29（IC） | 桥必要 | InputMethod 只能在 AccessibilityService 内取得（`BrowserAccessibilityService.kt:18-30`）；worker 通过 `IExecutor.setEditorBridge`（`WorkerService.kt:51`）拿到 App 侧 Binder，`EditorBridge.kt:125` 校验 SHELL_UID 后切到主线程捕获。 | 删除 IC 即可删桥。 | 约 96 行加 AIDL 加 attach/invalidate，成本与能力相称。 |
| 组合 38（显示 kind） | 一致，有一处注意 | 窗口目录按 lease target displayId 过滤（`AccessibilityConnection.kt:53-67`），绕开了 `UiAutomation(Looper,…)` 构造器隐含的 DEFAULT_DISPLAY（AOSP16 `UiAutomation.java:305` 日志“assumes DEFAULT_DISPLAY”）。 | — | 依赖 deprecated 的 hidden 构造器（`@UnsupportedAppUsage(maxTargetSdk=P)`），14–16/main 仍在。 |

---

## 3. 发现（按重要性）

### 3.1 当前节点链依赖的 hidden / non-SDK 接口清单【源码推断，已对照 AOSP 14/15/16/main】

| 层 | 成员（`HiddenApi.kt` 行） | 用途 | 14–16/main 状态 | 失效后果 |
| --- | --- | --- | --- | --- |
| UiA 构造/连接 | `UiAutomationConnection()` 构造（73,85）、`UiAutomation(Looper, IUiAutomationConnection)`（86-88）、`connectWithTimeout`（91）、`getConnectionId`（89）、`disconnect`/`UiAutomationConnection.disconnect`（94-95） | 在 shell app_process 中自建 UiA | 均存在；Looper 构造器已 `@Deprecated`（AOSP16:302-304） | 整条节点链不可用。 |
| UiA 私有字段 | `mRemoteCallbackThread`（96-101） | disconnect 后 join 内部线程 | 存在（14:372 / 16:377） | **在清理关键路径上**：`UiBackend.kt:505-508` 在 `try` 之前反射取它。字段一改名，`disconnect()` 就会抛异常，signals 未关、UiA 未注销，`clear()` 记录 cleanupFailure，后续 UI 准入受阻。属基线已有，但应改为 best-effort。 |
| cache | 14：`AIC.getCache(int)`（83）；15+：`UiAutomation.getCache()`（90）+ 反射 `setEnabled(false)` | 关闭共享 cache | 15/16 的 `UiAutomation.getCache()` 内部就是 `AIC.getCache(connectionId)`（AOSP16:566-572），`AIC.getCache(int)` 在 14/15/16/main 均为 static（:305/:311） | 版本分支多余，见方向 1。 |
| 服务连接 | `AIC.getConnection(int)`（82）→ IASC：`getWindows`（102）、`getMagnificationConfig`（103）、`performAccessibilityAction`（132-143）、**`findAccessibilityNodeInfoByAccessibilityId`**（117-128，E9 新增） | 窗口目录、放大、动作回执、E9 查询 | 四个签名在 14/15/16/main AIDL 中逐字一致 | 签名变化时报 NoSuchMethod（显式失败）。 |
| 回调 Stub | `IAccessibilityInteractionConnectionCallback$Stub.asInterface`（110）及 `TRANSACTION_setFindAccessibilityNodeInfoResult / setPrefetch… / setPerformAccessibilityActionResult`（111-116,129-131） | 自建 Binder 接收回调 | 回调接口 14：5 个方法；15/16：多了 `sendAttachOverlayResult`（在末尾，不影响已有 code）；三个方法签名不变 | **手工 Parcel 解析**：`CollectionNodes.kt:128-130` 依次 `readTypedObject` / `createTypedArrayList` / `readInt`，再 `dataAvail()!=0 → return false`；`AccessibilityConnection.kt:172-174` 同理。平台一旦在参数尾部增加字段，所有回调都会被静默拒收：采集退化为“只有根节点”，动作退化为 UNKNOWN。 |
| ANI 内部 | `getSourceNodeId`（77）、`getChildId(int)`（78）、私有 `LEASHED_NODE_ID`（79，从 `getField` 改成了 `getDeclaredField+setAccessible`）、`setConnectionId`（80）、`setSealed`（81） | sourceId 身份；E9 child 寻址与 finalize | 均存在 | E9 在应用层重写了 AIC 的 finalize（AIC15 `finalizeAndCacheAccessibilityNodeInfo` :1451-1466）；平台新增的 finalize 步骤不会自动跟进。 |
| 几何 | `AccessibilityManager.getWindowTransformationSpec` + `transformationMatrix/magnificationSpec` 字段（104-109；`AccessibilityConnection.kt:82-104`） | 坐标映射 | 未逐版核（本评审未下载 WindowTransformationSpec） | 只影响 mappingUnknown 拒绝。 |

**A 相对“纯公开”的必要 hidden 下限**：UiA 自建（构造/连接/注销）加上 cache 控制，是任何 shell-UiA 方案都绕不开的。`getWindows` 可以换成公开的 `UiAutomation.getWindowsOnAllDisplays()`（AOSP14 `UiAutomation.java:811`），只损失“RemoteException 与空目录可区分”这一窄诊断。`performAccessibilityAction` 及 `ActionCallback` 的保留有回执语义上的依据（属基线，不在本轮）。**真正因本轮新增、又可以讨论去留的，只有 E9 那 6 个成员加手工 Parcel 协议。**

### 3.2 A vs B：维护差额与能力价值【源码推断 + 实测】

- **两者都必须有普通服务**（Chrome），B 也省不掉它。B 的真实删减是：UiA 构造、连接、注销及内部线程处理（`UiBackend.kt:446-539`，约 95 行），`AccessibilityConnection` 中节点查询与 cache 的 hidden 部分，以及 E9。代价是：节点采集、REF 原对象、fill/settle、观察和交付全部搬到 App 进程，再与 shell worker 的输入、像素、租约进行类型化 IPC（`maintenance/report.md` §2/§4 的表格分析我同意）；身份基线从 sourceId 改为 `ANI.equals`；public action 会把 accepted、callback 和异常折叠为一个 false。在“节点、输入、观察、租约同在 worker”的现有代码上，A 的总维护面明显更小。**同意 A。**
- **Bitwarden 推理基本扎实**：U1/P/U2 是同窗口、同时序的三次采样（`next/capabilities/bitwarden-reviewed.json`）。P 的目录非空且选中了同一窗口，root 为 null，这排除了“找错窗口”和“预算耗尽”两种解释。机制有 14–16 源码链支撑（View 自动推导 dataSensitive，AIC.getRootView 过滤 tool=false）。不足之处只有一点：没有做 tool 位的单变量对照（同一个普通服务只把 XML 改成 tool=true 再跑一次）。作者“技术上可声明 true”的说法也成立：平台在 14–16 只读静态 XML，没有校验。但把一个自动化工具声明为“辅助残障用途”属于用途误报，和 UiA 作为测试自动化通道天然获得 true 不是一回事。我同意不以此作为 B 的修补手段。【未验证的补充】有资料显示 Android 16 Advanced Protection 会限制非 tool 服务，若属实，tool=false 的 Browser service 在该模式下会被禁用；这是 Browser service 的额外外部风险，不影响 UiA。
- **能力价值**：根视图开 `filterTouchesWhenObscured` 是 OWASP 等建议的通用防点击劫持做法，银行、支付、密码管理、2FA 类 App 采用得多。Android 14+ 的自动推导会让这些窗口对 tool=false 服务**整窗**不可见。对一个代 LLM 操作真实 App 的工具来说，这类 App 正是高价值场景。证据只有 Bitwarden 和 Settings 证书警告两处源码采用点，覆盖面属【源码推断】，但方向清楚。
- **“只用 UiA、不要普通服务”**：会失去 Chrome 网页树（E10 实测网页树掉到 31 节点）和 IC。不可取。**“只用普通服务 tool=true”**：等于 B 加用途误报。不可取。**“用 UiA 的 flags 让 Chrome 保持树”**：从包内日志看，Chrome 依据的是已启用服务列表，UiA 不在其中，所以没有 flag 能解决；Chromium 上游本评审未核。
- 【设计建议，低优先】shell worker 有 `WRITE_SECURE_SETTINGS`，可以在 Chrome 任务时按需写入 `enabled_accessibility_services`，从而免去用户手动启用和受限设置的流程，也免去服务常驻带来的全局影响。但这会改写用户的系统设置（包内保护检查正是以“secure setting 不变”为基线），用户也已表示不做初配改造。只作备选提出。

### 3.3 E9 的机制、复杂度，以及“必要性”证据的缺口【源码推断 + 实测复读】

**机制**。平台层面（AIC15 :604-722、:1269-1302），`findAccessibilityNodeInfoByAccessibilityId` 只等待 primary 单节点结果。预取结果通过另一个 oneway 回调 `setPrefetchAccessibilityNodeInfoResult` 送达，只在 interaction 等于“最后一个等待中的交互”时才 finalize 并写入 cache。cache 禁用时（:631-633）或 bypass 时（:646），预取标志会被清掉；窗口滚动时（:649-651）也会跳过预取。候选把 cache 关掉（`AccessibilityConnection.kt:34-41`），以免旧事件或旧预取污染观察，因此公开路径失去预取。E9 的做法：每次 child 查询新建一个 `Query : Binder`（`CollectionNodes.kt:107-181`），直接调用 hidden IASC 查询并带 `FLAG_PREFETCH_DESCENDANTS_DEPTH_FIRST`（:45-54），primary 用 latch 等到 collect 截止时间；预取列表作为“机会”先进入 gate 保护的 inbox，等下一次 `child()` 时再由 collector 线程 finalize 并 `putIfAbsent` 进本次 collect 的 `(windowId, sourceId)` 表（:74-83）。packages 早于或晚于回调到达都要处理（:148-158）；collect 结束时 `close()` 在同一个 gate 下封口，之后到达的回调丢弃（:98-105）。

**复杂度与脆弱点**：
1. 手工 Parcel 加严格的 `dataAvail` 检查（见 §3.1），平台 AIDL 在尾部演进时会**静默**全量失效，且没有任何告警或降级路径。
2. 应用层重写的 finalize 会与 AIC 后续版本脱节。
3. 绕过了 AIC 的 `isWindowScrolling` 预取抑制：滚动或 settle 期间仍会请求预取，加重目标 App UI 线程的负担。这是平台行为偏差，未实测其影响。
4. 【确定缺陷，轻】预算到期被误报为 unavailable。`CollectionNodes.kt:38/66/67` 在超时（包括“primary 已收到、但随后过了 deadline”这种情况，:67 会丢弃已到的节点）时返回 null，`TreeCollector.kt:319-321` 把它计为 `unavailable++` 并以 `unavailableChildren` 报出，而不是 `collectionLimit`。时间截断因此被说成提供方不可用。J68 中 timeouts=0，所以尚未观察到。
5. 每节点分配一个 Binder 加一个 latch；预取列表最多 50 节点、可能互相重叠（J68 prefetchReceived=3669，实际只用 4225），全部在 collector 线程上 finalize。

**必要性证据**（这是本专题最能改变判断的一点）：

| 样本（同一 MuMu `2a058081…`，同一 Wikipedia 首都长页） | 路径 | 节点 | 采集耗时 |
| --- | --- | --- | --- |
| J54 event0 P1/P2（普通服务探针，`e17-j54-reviewed.json`） | 公开 `getChild(i,0)`，cache 关，**无预取** | 4214 full | child 累计 1075 / 1040ms；workWall 1137 / 1098ms |
| J54 relevant Q1/Q2 | 同上 | 4214 full | workWall 967 / 865ms |
| J54 UiA U0/U1（研究版 E9） | E9 | 4214 full | 1114/600ms（event0），834/352ms（relevant） |
| **J67 HEAD single**（`j67-single-reviewed.json`） | UiA 公开 `getChild`，cache 关，无预取 | **3869 partial** | **3157ms** |
| J68 E9 | E9 | 4226 full | 1378ms |

J67 与 J54-public 走的是**同一条 AIC 单节点代码路径**（性能报告 §3、§5 自己也说两者“落同一 AIC”）。差别只在宿主（UiA shell 进程 vs App 内普通服务）和时间点，耗时却相差约 3 倍。没有任何材料解释这一差异：内部 review `architecture-next.md:57` 只说“child 时间门……可解释此结果”，没有与 J54 比较。同时，E9 自身连续两个样本也相差 2.4 倍（834 vs 352），说明这个页面上单样本的方差很大。因此“single 在 3s 内拿不到 full”**只有 n=1 支撑，并且与同设备的另一组实测矛盾**。如果 single 真实约为 0.25ms/节点，那么 8192 节点约 2s，3000ms 预算足够；即便偶尔超时，partial 也只发生在不可见区域。

**对“临时启用 cache”被否决理由的复核**。`research/observation/opportunistic-prefetch.md` 末节的否决理由是：clear 不会重置 `mInteractionIdWaitingForPrefetchResult`，事件 refresh 又在锁外回写。我核了 AOSP15 `AccessibilityCache.add`（`if (!mEnabled) return`）：只要“采集开始时 enable 加 clear，结束时 disable”，两次采集之间到达的迟到预取或 refresh 都会被丢弃。剩下的竞态只有“禁用→下次启用”交界处恰好仍在途、又恰好匹配旧 interaction 的那批数据，它们是采集开始前几毫秒的真实数据。采集期间由事件触发的 refresh 读到的是**更新的**数据，而不是旧数据。树本来就跨越最长 3 秒，不是原子快照。为排除这个窗口而自建整条 Binder 回调链，正是“为极窄区分增加机制”。另外，平台只保留“最后一个等待交互”的预取，因此吞吐可能低于 E9。我推断（未实测）在 oneway 同序投递下，大部分预取仍能被收进 cache，这一点需要补证。

### 3.4 ee841ea：可选事实的消费者与体积【实测复读 + 源码推断】

新增字段：`roleDescription`、`paneTitle`、`tooltipText`、`heading`、`dataSensitive`、`chromeRole`、`targetUrl`（`NodeFacts.kt:97-101,138-139`），外加 UTF-8 截短修正（`NodeFacts.kt:231-260`，有真实缺陷依据，应保留）。

| 字段 | 消费者 | Chrome S0 样本（4225 节点，`09-node-facts/runtime-combined/result.md`） | 判断 |
| --- | --- | --- | --- |
| chromeRole | 行属性、UiEffect | 非空 4194（几乎每个节点） | 与派生 role 重复；在默认行中出现 80 次，**32 行空布局只因它而保留**。从行中删除；raw 也可不采。 |
| tooltipText | 行、UiEffect | 非空 785、`""` 3413（raw 中保留空串） | Wikipedia 中多与 name 相同（如 `"name":"Abidjan","tooltipText":"Abidjan"`）。只在与 name/detail 不同时显示。 |
| roleDescription | 行、UiEffect | 非空 834 | Chrome 给出本地化角色名（“链接”“标题 2”“区域”），与 role 重复。只在与 role 不对应时显示，或直接删除。 |
| targetUrl | 行、UiEffect | 非空 818 | 对 Web 任务有真实价值（链接去向），但 token 很重。只在 target 或 `--all` 时显示。 |
| paneTitle | 行、UiEffect | Chrome 全 null | 原生对话框/面板有用，保留。 |
| heading | 行 | true 10 | 有用且便宜，保留。 |
| dataSensitive | 行 | 全 false | 提示性；LLM 难以据此改变行动，只显示 true 即可（当前已如此），成本很低。 |

**体积**：J68 E9 raw 为 4,930,351B/4226 ≈ 1167B/节点，S0（ee841ea 之后）为 5,553,077B/4225 ≈ 1314B/节点，约 +12.6%（只做了跨样本对比，可能混入其它字段变化）。默认行方面，同页旧测量是全文 4,476B（`cli/followup/large-page/report.md`），S0 为 10,097B；按本评审对 S0 的剥离计算约 5,535B。默认会在每次动作后打印观察（方向 39），所以这部分增量**每一步都要付**。

### 3.5 8192 / 3000ms / 8MiB【实测复读 + 源码推断】

- 支持 8192 而非 4096 的证据，只来自一个 Chrome Wikipedia 长表（4214–4237 节点）：4096 不完整，8192 完整。E10 的小米页有 1838 节点。原生 App 页通常只有几十到几百节点，上限对它们没有影响。所以“固定 8192”实质上是“Chrome/WebView 长文档不截断”的开关，按页面付费，小页面免费。这比“默认小、显式扩大”更简单（后者是用户已否决的旧 D 选项）。
- **8MiB 才是 Chrome 上的有效上限**：1314B/节点 × 6384 ≈ 8MiB。`TreeCollector.kt:275-276` 的早停只计算组装前的事实字节，组装后由 `UiTreeBudget.fit`（`UiTreeBudget.kt:16-40`）再截。在接近上限的大页上，会白采约 1.8k 个最终被丢弃的节点。【设计建议】早停时按组装后开销估算，或接受这一点并在文档中写明“节点上限在 Web 页上通常先受字节约束”。
- **LLM 阅读成本**：默认行只投影可见节点，与节点总数基本无关（4214 节点页约 1K tokens，ee841ea 之后约 2.7K）。真正危险的是 `--all`：4214 节点约 117K tokens，8192 节点约 230K。`--max-chars` 默认不设上限（`skills/androidctl/observations.md:50-53`）。这属于方向 39/40 的组合问题，建议 `--all` 在大树上给出默认字符预算或显式提示，由负责该方向的评审员裁决。
- **重复采样成本**：settle 和 wait 的每次采样都是整树。大页上每份 1.4–3s 加约 5MB，J68 中 click 的 CLI 耗时 4.4s。这是 8192 在“动作”上的真实代价，与事件唤醒（方向 14）相关。
- “1024+ref 3.1–3.2s vs 8192 3.3s”（`context/user-messages.md:103`）说明的是速度相当。8192 的收益在于定位不可见目标、selector 唯一性和 gone 判断的可证明性，以及免去滚动查找。我认为对“固定上限、可见优先、超时只缺不可见部分”这一模型，不需要重新裁决。

### 3.6 其它小项

- 【确定缺陷，轻】`VisualInput.kt:49` `"apiSupported" to true` 是常量字段，仍出现在 status 输出和 `docs/reference/connection.md:105` 中，应删除。
- 【确定缺陷，轻】`NodeFacts.kt:133-136` 给**每个节点**写入 `checkedModel`（设备级常量，由 SDK 推导），包括不可勾选的节点。行视图已省略 triState，但 raw 上每节点多约 25B。可以只给可勾选节点写，或改为设备级事实。
- `UiBackend.kt:454` 的 `FLAG_DONT_SUPPRESS_ACCESSIBILITY_SERVICES` 是 Browser service 能与 UiA 共存的前提（默认会压制其它服务），是正确且必要的耦合，建议在代码注释中写明理由。

---

## 4. 建议重新裁决的事项

### 4.1 节点采集链（在 A 内部）

| 选项 | 内容 | 优点 | 缺点 |
| --- | --- | --- | --- |
| A：维持 A+E9 | 现状 | 已有 J68 full 及两条业务闭环；约 1.5–2.5 倍采集提速（J54/E9-Chrome1024） | 手工 Parcel 加重写 finalize 的长期平台风险；186 行加 6 个 hidden 成员；必要性证据只有 n=1 |
| B：A 去掉 E9 | 回到 UiA 公开 `getChild`（cache 关、单节点），保留 8192/3000ms | 删除 `CollectionNodes.kt`、6 个 hidden 成员和全部查询回调协议；节点 hidden 面回到“UiA 自建 + cache 开关 + 现有动作回执” | 超大页可能慢 2 倍；若 J67 的 3s 属实，>约 4k 节点的页在不可见区域会 partial |
| C：A 加平台 cache 采集窗 | 每次 collect：`setEnabled(true)`（会 clear）→ 公开 `getChild(i, FLAG_PREFETCH_DESCENDANTS_DEPTH_FIRST)` → `setEnabled(false)` | 保留预取收益，删除手工 Parcel 与重写的 finalize，hidden 成员与当前的 cache 开关重合 | 接受交界处毫秒级的来源窗口；平台只保留最后一个等待交互的预取，吞吐需要实测 |
| D：普通服务唯一宿主（原 B） | — | 删除 UiA 与 E9 | 失去 dataSensitive 窗口；要求全面跨进程重写；不推荐 |

**推荐**：先做 §6 的补证 1（成本低），再在 B 和 C 之间选择；只有 B、C 都在约 4.2k 节点页上稳定超过 3s，才保留 A。若无法补证，在 A 与 B 之间，我倾向 B，理由如下：partial 只缺不可见区域，可见区域的 REF 与读取不受影响，而 E9 的失效方式是静默的全量退化。

### 4.2 节点可选事实的行投影

| 选项 | 内容 | 推荐 |
| --- | --- | --- |
| A | 维持现状 | 否：Chrome 默认行约增大 1.8 倍 |
| B | raw 保留；行中删除 chromeRole，roleDescription/tooltipText 去重后显示，targetUrl 只在 target/`--all` 显示，元数据不阻止空布局折叠 | **推荐** |
| C | B，并在采集端不再读 chromeRole/tooltipText（raw 也删除） | 可接受，进一步减小 8MiB 压力 |
| D | 整体回退 ee841ea（保留 UTF-8 修正） | 否：paneTitle/heading/targetUrl 有真实价值 |

---

## 5. 对既有内部 review 与交付方声明的对照

- `reviews/architecture-next.md`：结论是“最终支持推荐 A+E9 一条采集链”，并称 J67 阴性“可由 child 时间门解释”。**不同意其中的 E9 部分**：它没有把 J67（3157ms/3869）与同设备、同页、同 AIC 路径的 J54 公开单节点（约 1s/4214）对照。这两组数据相互矛盾，足以否定“single 必然超 3s”。A 优于 B 的部分我同意。
- `research/observation/opportunistic-prefetch.md` 否决“临时启用 cache”：事实陈述正确（clear 不重置等待 interaction，refresh 在锁外），但**风险定级过严**。它漏掉了 `AccessibilityCache.add` 在禁用时直接丢弃这一点（AOSP15 中可核），因此“采集窗”方案的残余风险只剩交界处毫秒级、且不比采集本身更旧的数据。
- `stages/11-prefetch/review.md`：“没有 must-fix，也没有 optional”。这次 review 只针对实现正确性，其结论成立（gate 与封口设计严谨）。但它没有评估“这个机制是否值得存在”，也没有发现超时被误报为 unavailable（§3.3 第 4 点）。
- `reviews/final-maintainability.md` 第 34 行称 Browser support、E9 等“分工清楚”，`final-android-contracts.md:39` 也是类似描述。两者都没有触及 E9 的平台演进风险，也没有发现 ee841ea 对默认行体积的影响。
- `deliverables/candidate-final.md:119` 已承认“平台升级须维持这一条节点链及 Chrome 普通组件前提”，方向正确，但没有说明失效时是**静默**退化。
- 用户消息 L8870 中，助手称 E9 “支持预取的完整性与业务价值”。准确的说法应是：J68 证明了 E9 **能**在 3s 内取得 full，但没有证明**不带 E9 就不能**。

---

## 6. 未覆盖范围与必要补证

**会改变判断的补证（按性价比排序）**：
1. **同一 worker 内 ABAB 对照**：在研究构建里加一个运行时开关，在 J67/J68 同一页（约 4.2k 节点）上交替采 single 与 E9，每组至少 4 次；条件允许时加 C（cache 采集窗）作为第三臂。判定标准：single 中位数 ≤1.5s，选 B（删除 E9）；single 约 3s 而 C ≤1.5s，选 C；B、C 都约 3s，保留 A。这是本专题唯一能直接翻转结论的实验。
2. Browser service 在 tool 位上的单变量对照：同一服务改为 tool=true，在 Bitwarden 声明页采一次。它只用于确认机制归因，不改变“不做用途误报”的结论，优先级低。
3. 在非 Chrome 的 Chromium WebView App 上，验证不启用 Browser service 时网页树是否同样会自动关闭。结果决定该服务对 WebView 类 App 是否同样必需。

**本评审未覆盖**：没有核对 Chromium 上游的自动关闭与服务判定代码；没有逐版核对 `WindowTransformationSpec` 字段；没有设备可供重跑；没有逐行审 `FreshNodes`、`UiActions`、`UiRefs` 的身份逻辑（属其它方向）；ee841ea 的 raw 体积增量是跨样本比较得出的，不是同样本单变量结果。
