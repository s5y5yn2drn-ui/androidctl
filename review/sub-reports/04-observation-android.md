# 04 设备端观察：采样/发布/传输、事件与等待、图像 —— 独立评审

评审范围：CURRENT_SCOPE 方向 3、4、5、14、15、21、22、37、40，以及「必须合起来审的接点」第 3、4、5 条。
源码以 REPO 当前检出 `3f45af6` 为准，行号取自该提交；基线 `7d5be70` 用 `git show` 对照。
证据标注：【实测】=包内原件记录的真实运行（注明是否当前候选）；【源码推断】；【设计建议】。
发现标注：【确定缺陷】【设计取舍】【证据缺口】。我没有设备，没有重跑 Android；只在 scratchpad 里用 `python3 -I` 对包内原始 JSON 树和 CLI 文本做了只读统计（脚本 `agents/04tools/nodesize.py`）。

---

## 1. 结论摘要（重要的在前）

1. **近期事件（15）默认交付给 LLM 是净负担，应删掉，或只保留带文字的提示。** 候选自己的成对业务运行（paired-runtime contacts-2、chrome-1c，均为当前候选物料）一共打印了 132 条 `event` 记录：**全部是 `kind=window`，没有一条带 text/description**；`events`/`event` 行占已保存正文字节的 9.7%（contacts-2，12,818/131,641B）和 14.1%（chrome-1c，23,685/167,564B）。这些行是原始整型（`type:4194304`、`windowChanges:2`、`contentState:"sourceUnknown"`），LLM 无法据此做出不同决定，而窗口变化已经由目录和 `observedChange.windows` 表达。为此付出的是约 780 行生产代码（`0e583db`）：ring、来源保护、UTF-8 二分裁剪、冻结重放、windows 附件二次编码。交付方列出的「默认摘要」选项 A 仍保留来源/区间/淘汰摘要，还要新增一个阅读选项，我认为不够。
2. **一次动作回执里有 3–5 个互相重叠的「变化」裁决，其中 `settle` 在两类常见场景下是恒定值，最容易让 LLM 误读。** 系统动作（back/home/key/open…）一律以 `before=null` 进入 settle（`SystemActions.kt:58`），因此**必然**报告 `settle=unsettled`，以及 `change=unknown; changeReason="No before or final observation"`（`UiPostAction.kt:193`、`ObservedChange.kt:11`），即使最终树已经采到并交付。【实测】contacts-2 的 back 回执就是这样。另一类场景：基线树为 partial 时（Chrome 长页几乎总是），settle 永远拿不到 CHANGED/UNCHANGED，于是每个动作都会耗满 settleMs 反复整树采样，最后仍报 `unsettled`。建议 settle 只回答「是否稳定」，「是否变化」只由 observedChange 回答（与上次外评「闭环-7」一致，内部 review 的反驳不成立，理由见 §5）。
3. **sample/publish 分离（3）是真实、简单的收益，应保留。** 基线在每次内部 sample 后都做 JSON 序列化、deflate 和 hash（`7d5be70:UiBackend.kt:297` `snapshots.deliveryTree()`），只为预留交付时间；settle/wait 的每次中间采样都白付这笔开销。候选只在最终交付时编码一次。
4. **「目录/tree/image 三个独立附件」（4）的方向对，但切法过细。** 树失败时仍交付目录和图片，这个价值是真实的：已有 scopeUnknown、protected 的自然样本（12/13 阶段 DISCARD、管理页）。但目录是几 KB 的小对象，没有必要作为第三个分块附件：现在每次观察都多一次 `ui.read` 往返，`recentEvents` 也因此要在出图之后把整个 windows 附件**再编码一次**（`UiDelivery.kt:261-307`）。建议把目录（以及仍保留的事件提示）内联进主回执（192KiB 绰绰有余），只把 tree 和 image 作为分块 blob。
5. **字节定稿（40）在修一个自找的问题。** 树的字节主要来自冗长的节点 schema，而不是页面内容。我统计了当前候选真实 Chrome 树（seq70，568 节点）：平均每节点约 1.5KB，`actions` 对象数组占 22%，大量 `null/false/""` 默认字段各占 1.5–2.5%；正文很少。8MiB 因此大约对应 6000 个 Chrome 节点，8192 的节点上限在网页上基本不可能达到。交付方用来验证的 8,377,092B「真实树」其实是**自造 fixture**：127 个节点、123 个 row，每个节点复制同一个 64KiB `row.label`（`UiSemantics.row` 把行标签整段复制进行内每个后代，`UiSemantics.kt:175-205`）。它证明的是投影会放大文本，而不是 8MiB 定稿机制有必要。该机制目前有三道字节闸（采集期原始下界、投影后 fit、发布时 `prepareJson` 硬拒），外加事件 room 计算。正确方向是先给 schema 瘦身，并把 row.label 改成引用或截断，再把 fit 降为罕见兜底。
6. **事件唤醒（14）去掉了原来 250ms 的节流，没有任何 A/B 证据证明更快。** 12 阶段原件明确写着「没有AB对照……不称事件提速」。【源码推断】在持续动画或内容事件风暴下，settle/wait 会变成背靠背的整树采样（`UiBudget.kt:49-52` + `UiSignals.kt:98-129`）。每次采样都占用目标 App 的 UI 线程，可能反而拖慢要等待的那个转场。建议保留事件唤醒，同时设一个最小间隔。
7. **传输常量（5）可以保留，不值得再裁决。** 改动本身只有约 30 行生产代码（`865d816`），收益确实很小：只有按需取原尺寸大 PNG 时约省 140ms；默认 JPEG 和压缩后的树通常只有 1–2 块。真正可省的维护面是第 4 点所说的「目录作为第三个附件」。
8. **图像（21/22/37）模型本身合理，但失效规则与 zoom 的耦合过紧，而且 zoom 的价值没有在任何证据里成立。** JPEG 预览加交付图坐标逆映射，是为高分辨率设备上「LLM 看到缩放图」配套的必要设计，应保留。但全部图像证据都来自 720×1280 设备，长边恰好等于 1280，**JPEG 根本没有缩放**，所以 zoom 在这些证据里只能消掉 JPEG 压缩伪影；结果页也自述「不宣称新细字收益」。每租约一帧原生源的模型可以保留，但应只在「新截图 / 派发 / 租约结束」时失效。现在任何树签名变化都会销毁它（`VisualInput.kt:39-44`），而签名包含 partial 和节点数波动，长网页上几乎每次 fresh snapshot 都会销毁 zoom 源。
9. 我在本专题内**没有找到并发或 Bitmap 释放类的确定缺陷**：所有 UI 工作都在单线程 executor 上，`ui.read` 走 `UiDelivery` 的同步锁，锁顺序一致。确定缺陷集中在误导性字段和成本上（见 §3）。

---

## 2. 负责方向处置表

| 方向 | 处置 | 依据 | 依赖 | 真实能力损失 | 维护代价变化 |
|---|---|---|---|---|---|
| 3 sample/publish 分离 | **保留** | 去掉了每次中间 sample 的序列化、压缩和 hash（基线 `UiBackend.kt:297`）；这是确定的 CPU 节省，实现很小（`0f5f535` 生产代码 +58/−38） | 无 | 无 | 降低 |
| 4 目录/tree/image 独立交付 | **调整** | 树失败时保留目录和图片有真实样本（scopeUnknown、protected）。但目录作为第三个分块附件，每次观察多一次往返，还导致事件要二次编码 | 15 的处置 | 无（目录改为内联） | 删除 windows artifact 的四态、一次 read 和 `completeEvents` 重编码 |
| 5 UI 64KiB / 回复 192KiB / 文件 32KiB | **保留**（可选：文件块也统一到 64KiB，只留一个块常量） | 改动小，收益小但为正；不再值得投入裁决成本 | 无 | 无 | 中性 |
| 14 事件促发条件检查 | **调整** | 方向成立（降低等待相位延迟），但去掉了节流，又没有 A/B 证据 | 无 | 无 | 加一个最小间隔常量 |
| 15 近期事件 ring/来源/区间/默认打印 | **删除**（或替换为「仅文字提示」） | 真实业务 132/132 条为无文字的 window 元数据，占正文 10–14%；约 780 行实现 | 4 | 失去瞬时 toast/pane 文本的被动捕获。在真实 App 证据里从未出现过（13 阶段：「无可证Toast或pane正文」） | 大幅降低 |
| 21 默认 JPEG + 交付图坐标，PNG 按需 | **保留**，精简字段 | 高分辨率设备上坐标映射是必要的；`imageMappingKnown`、`imageSourceSize` 等只在异常时才需要打印 | 无 | 无 | 略降 |
| 22 显式 REF 标注 | **保留** | 只在显式 `--marks` 时生效，默认零成本；R/P 原件显示副本已显式 free | 无 | 无 | 中性 |
| 37 每租约一帧原生源 zoom | **调整** | 只读裁剪同帧，并保留原时点，这是可辩护的；但「树签名变化即失效」与 zoom 无关，且 zoom 在证据里的阅读价值未成立 | 21 | 无 | 降低（删一条失效耦合） |
| 40 真实 UTF-8 树统一定稿 | **调整 → 先替换根因** | 定稿解决了「安装的树与发布的树不一致」这个真实缺陷（上次外评「采集-1」），但触发它的主要是 schema 冗余和 row.label 复制 | 11–13 的 schema、host 契约 | 无（瘦身不丢事实） | 先增后降：瘦身是一次性的契约改动，之后可以撤掉两道字节闸 |

---

## 3. 发现（按重要性）

### F1 近期事件默认打印：无信息量的正文税【设计取舍｜实测+源码】
- **位置**：`UiRecentEvents.kt`（全文 320 行）、`UiSignals.kt:77-85`、`UiBackend.kt:275-285`、`UiDelivery.kt:49-61,116-144,261-307`、`ui_view.py:361-375`。
- **实测（当前候选，paired-runtime）**：对 contacts-2 与 chrome-1c 的 `artifacts/*.txt` 做只读统计，`event` 记录共 132 条，`kind` 全部为 `window`，含 `"text"` 的记录为 0。`events`/`event` 行在 contacts-2 占 12,818/131,641B，在 chrome-1c 占 23,685/167,564B。13 阶段 API34 结果页也写明「真实App本轮投影记录均是window类型……无可证Toast或pane正文」。A 观察中 recentEvents 原字段有 7,704B，而整个 windows 附件只有 8,426B（28 节点小页）。
- **机制**：只读命令（snapshot/wait/locate）没有事件起点，每次都重放整个 16KiB ring（`UiRecentEvents.kt:173-212`，mode=`recent`）。动作则重放操作区间。cached 读取还要冻结重放（`UiDelivery.kt:136-143`）。
- **它防什么**：转场中一闪而过的 toast/pane 文本（树里采不到）。**真实发生过吗**：包内真实 App 运行里从未发生，只有自建机制探针（13 runtime-mechanism）构造过。
- **删掉损失什么**：失去被动捕获瞬时 toast/pane 文本的可能性。窗口出现/消失已经由目录和 `observedChange.windows` 给出。
- **附带成本**：每个被投影的事件都在无障碍回调线程上、持有 gate 锁，把整个 ring 解析再序列化 2–3 次（`recordBudget`→`payload`，`UiRecentEvents.kt:214-219`；淘汰循环每轮重新序列化整环，`:120-122`）。量级是毫秒，但这正是不必要的复杂度。
- **建议**：见 §4 R1。推荐只保留「有文字的 toast/pane」提示，或者整体删除；无论哪种，都删掉 source/range/retained/incomplete 元数据和 cached 冻结。

### F2 settle / observedChange / effect 多重裁决，settle 在常见场景下是恒定值【确定缺陷（误导字段）＋设计取舍｜源码+实测】
- **触发路径 1（系统动作）**：`SystemActions.kt:57-60` 调用 `settle(run, null)`。`SettledSamples.classification()` 在 `baseline == null` 时直接返回 UNSETTLED（`UiPostAction.kt:193`），`ObservedChange.compare` 在 before 为 null 时返回 `unknown("No before or final observation")`（`ObservedChange.kt:11`）。所以 back/home/recents/key/open/notifications **每一次**都打印 `settle=unsettled; change=unknown; changeReason="No before or final observation"`，即使最终树已经采到（「or final」这个原因文字在这时是错的）。【实测】contacts-2 back 回执原文就是如此，同一回执后面还附带了 seq12 的新树 snapshotPath。observations.md「Post-action Sampling」把这写成了契约，但写成契约并不让它有信息量。
- **触发路径 2（partial 基线）**：`classification()` 中 `before?.let{!completeTree(it)} == true -> UNSETTLED`（`UiPostAction.kt:193`）。循环的提前结束条件只有 `stableWithoutBaseline()`（仅 before==null）或 CHANGED（`:64-68`），所以基线为 partial 时**不可能提前结束**：每个动作都耗满 settleMs，期间反复整树采样，最终必报 `unsettled`。Chrome 长页几乎总是 partial（38 阶段 seq60/61/70/71/74 均 partial；seq71 click 回执即 `settle=unsettled`）。
- **触发路径 3（互相矛盾）**：【实测】contacts-2 首个 click 回执：`settle=unsettled; …; changed; windows={changed}; selection={changed}; content={unknown}; baselineSequence=1; baselineFinishedMs=…; baselineAgeMs=7844; baselineComputedAtMs=…; effect={"state":"unknown",…,"source":{beforeObservation…,afterObservation…}}`。一行里有 settle、observedChange.state、三个维度、effect 共六个变化相关的裁决，外加两个原始时钟数（`baselineFinishedMs`、`baselineComputedAtMs`，`ui_client.py:274-281`），对 LLM 没有用。
- **影响**：LLM 无法从 settle 得到任何可行动信号，又可能被「unsettled + changed」误导而多等或重复读取；partial 页上的每个动作都额外付出约 1.5s 以及若干次整树采集。
- **建议**：见 §4 R2。

### F3 事件唤醒去掉了采样节流【设计取舍/风险｜源码推断，证据缺口】
- **位置**：`UiBudget.kt:49-52`（`nextCheckDeadline = min(now+250, …)`）、`UiSignals.kt:98-129`、`UiBackend.kt:292-307`、`UiPostAction.kt:63-81`、`UiCondition.kt:387-412`。
- **机制**：`beforeCheck` 在采集开始前记下序号。只要采集期间来了任何相关事件，`await` 就立即返回，没有下限间隔。默认 scope 下，同一 display 上任何窗口的 `WINDOW_CONTENT_CHANGED`（含 SystemUI、IME）都会唤醒。基线是固定 `pause(≤250ms)`。
- **后果（推断）**：在动画、进度条、视频页上，settle 窗口内（partial 时总是耗满，见 F2）和 wait 期间会变成背靠背的整树采集。每次采集都在目标 App 的 UI 线程上执行无障碍查询，并附带 O(n²) 的 `coverage`（`UiSnapshot.kt:147-169`）。
- **证据**：12 阶段只证明了「framework 至少一次送达 listener」（seq=7），自述「没有AB对照……不称事件提速、采样减量或普遍性能收益」。
- **建议**：保留唤醒，同时加最小间隔，例如 `max(100ms, 上次采集耗时)`；或者只在 `WINDOW_STATE/WINDOWS_CHANGED` 时立即唤醒，内容事件仍按计时器走。若要裁决，需要的补证见 §6。

### F4 字节预算：三道闸，防的是 schema 自身的膨胀【设计取舍｜实测+源码】
- **位置**：采集期 `TreeCollector.kt:275-281`（累加未投影 facts，不含 ref/parent/view/row）；投影后 `UiSnapshot.kt:43` → `UiTreeBudget.fit`（`UiTreeBudget.kt:16-40`，按先序前缀裁剪并修复关系）；发布时 `UiDelivery.kt:418-420` 硬拒；事件余量 `UiDelivery.kt:275`。
- **实测密度（当前候选真实树）**：seq70 Chrome 树 884,535B / 568 节点 ≈ 1,557B/节点。`actions` 平均每节点 322B，占 22%，每个动作都是 `{"id":…,"name":"action-40","label":null}` 对象；`view` 149B，`row` 82B；`showingHintText/checkedModel/stateDescription/contentInvalid/roleDescription/collectionInfo` 等默认值字段各约 23–24B/节点。抽样节点里几乎全是 `null/false/""`。旧 09 长表 4,225 节点约 5.55MB，与此一致。所以 **8MiB 在网页上约等于 5,500–6,400 节点**，8192 节点上限基本不会成为约束。上次外评也算出约 6,390 节点。
- **127 唯一 REF / 8,377,092B 是什么**：`stages/36-native-zoom/runtime-software-api34/dense-row-facts.json` 和 `38/dense-fixture-preparation.md` 写明，这是任务自建的 `org.androidctl.research.dense` `row-budget` 模式：一个 CollectionItem 行，首个 TextView 是 65,536B ASCII，后面接短文本后代。`UiSemantics.row` 把同一段 64KiB 标签复制进每个后代的 `row.label`，结果是 127 个节点撑满 8MiB（`allRowLabelUtf8Bytes:[65536]`、`rowLabelsEqualFirst:true`）。**它不是发行版页面**。它反证了根因：一个节点的文本被投影复制了 N 份。
- **判断**：8192/8MiB 组合本身不算荒谬，但在现有 schema 下名义上的 8192 是虚的；真正起作用的约束是字节数，而字节又被默认值和复制主导。限制单节点投影文本、去掉复制和默认字段，比事后裁树更对症。还有一处相关的不一致：采集按「可见优先、浅层优先」（`TreeCollector.kt:63`），fit 却按先序前缀裁剪（`UiTreeBudget.kt:42-54`）。超限时会先丢掉先序靠后的可见节点或后一个窗口的根，而保留先序靠前的不可见节点。触发频率是真实存在的：真实网页超过约 6,000 节点时就会进入 fit。
- **建议**：见 §4 R4。

### F5 目录作为第三个分块附件，以及事件二次编码【设计取舍｜源码+实测】
- **位置**：`UiDelivery.kt:224-233`（windows 先编码并压缩）→ `:270-291`（出图之后加入 recentEvents，`prepareJson(body)` 第二次编码）；host `ui_client.py:651-686` 按顺序 read tree、windows、image。
- **实测**：12/13 阶段每个普通观察都是 `main/read = 1/2`，带图是 `1/3`。windows payload 只有 0.5–8KB。
- **建议**：目录本来就小，可以内联进主回执（`REPLY_JSON_BYTES` 192KiB）。这样能删掉 windows artifact 的 NotRequested/Pending/Unavailable/Available 四态、一次往返，以及事件完成时的重编码。树失败时目录仍随主回执到达，所以第 4 方向的真实能力不受影响。

### F6 多个「已发布」权威分散在三个对象里【设计取舍｜源码】
- `UiSnapshot` 持有当前观察、targets、sequence；`UiDelivery` 持有 `current`（附件槽）、`publishedTree`（`:63-67,251-253`，3f45af6 为修复「zoom 冲掉 screen-change 基线」而加）、`publishedEvents`（`:56-61,302-304`）；`VisualInput` 持有 `image`；`UiSignals` 持有 `protectedWindows`，是当前观察的投影副本（`UiSnapshot.kt:58`）。`publishedTree` 和 `publishedEvents` 的键都是「同一 lease 下的某个 sample sequence」，其实就是「这份 sample 是否已发布、发布时附带了哪些事件」。
- **建议**：用 UiSnapshot 上的一个 `publishedCapture: String?` 表达，install 时清空；若按 F1 删除事件冻结，`publishedEvents` 随之消失。少两个跨对象、需要单独比对 lease/connection/sequence 的键。

### F7 图像源失效与树签名耦合；zoom REF 拒绝越界或零高节点【设计取舍＋小缺陷｜源码】
- `VisualInput.observed`（`:39-44`）：只要新旧树签名不同就回收原生源。签名包含 `partial`、scope 和全量内容（`UiSnapshot.kt:107-116`）。限时采集的 partial 树每次节点数都不同，所以长网页上每次 fresh snapshot 都会让 zoom 失效（`staleImage`），也会让坐标帧失效。对坐标输入来说，这是保守的取舍；但 zoom 只是读历史像素，而且 crop 元数据已经标明原时点，与树是否变化无关。
- `ImageRegion.kt:157-158`：`zoom REF` 要求节点 bounds 完全落在显示区内，`rectangle()` 还要求宽高非零（`:178-183`）。部分可见的节点、零高节点（例如 seq70 树里的 `[32,1184,720,1184]`）会被拒。这是【确定缺陷，低】：应当裁剪到显示区内，而不是拒绝。
- 内存：R/P 原件（720×1280）证明源像素、marks 副本、ROI 都有显式 free，同一时刻峰值是两块 3.69MB 像素。**没有说明的**：高分辨率设备（例如 1080×2400，单块约 10.4MB；`bitmap.scale` 在需要缩放时会产生第三块）、与 8MiB JSON 树并存时的 Java 堆、RSS/GPU。所以这些证据只能支撑「720p 上没有泄漏」。

### F8 一次 `click REF` 的事实层（回答问题 1）

| 层 | 设备/主机位置 | 是否重复 |
|---|---|---|
| 0 预算预留 | `UiBackend.kt:118` 按上一个 capture 的大小提前截止时间 | — |
| 1 事件起点 | `UiBackend.kt:114` cursor | 只服务 F1 的近期事件 |
| 2 REF 解析/修复 | `UiRefRepair.perform` → REF 过期时 `ui.capture`（**整树采样 #1**，可选） | 视情况 |
| 3 Fresh 原目标 | FreshNodes：原节点、父链、row 上下文（局部 IPC） | 必要 |
| 4 派发 | `dispatch` → `visuals.invalidate()` | — |
| 5 settle | 每轮**整树采样**，至少 2 次才能判稳（partial 时耗满 settleMs，见 F2）；事件或计时器唤醒（F3） | **主要采样成本** |
| 6 局部核值（fill/check） | 从 settle 样本取原目标 | 复用样本 |
| 7 observedChange | 端点比较：before＝**缓存旧树**（可能已有数秒，示例中 7,844ms），after＝最后一个样本 | 与 settle 共用样本，结论重叠 |
| 8 effect | 原目标字段前后对比，同一对 before/after | 与 7 重叠 |
| 9 发布 | 最后一个样本只编码一次（tree、windows），附图时出图后**再编码一次 windows**（F5） | windows 编码两次 |
| 10 传输 | 主回执＋read tree＋read windows（＋read image） | 多一次往返 |
| 11 host | 解码→JSON→行投影→保存 `.txt`→默认打印；结果行再打印 settle/change/维度/baseline 四数/effect | 同一变化事实打印 3–5 次 |

- **重复采样**：发布阶段不再重采，最后一个 settle 样本就是交付的树，这一点做对了。重复只发生在 settle 循环本身，partial 页和事件风暴会放大它。
- **多次编码**：windows 两次；变化事实三套（settle 签名、observedChange、effect）。
- **能否显著简化**：可以。(a) settle 只给「稳定与否」，变化只给 observedChange，effect 只在目标级字段确实变化时输出；(b) 删除事件 ring，或只留文字提示；(c) 目录内联；(d) 删掉对 LLM 无用的 baseline 时钟数，只保留 `baselineAgeMs`。
- **sample/publish 分离和独立附件实际解决了什么**：(1) 中间样本不再编码；(2) 树失败（protected、scopeUnknown、treeByteLimit）时仍有目录和图，12/13 阶段 DISCARD 后 scopeUnknown 就是自然样本；(3) 截图不再必须先有树。22 案报告首轮没有附件失败样本，这是事实。但 (1) 本身已足以支撑方向 3；方向 4 的价值是「树失败不连带丢掉其它东西」，用「目录内联＋tree/image 两个 blob」就能实现，不需要三个对等附件。

### F9 传输（回答问题 3）【实测，研究原型】
- E15（MuMu/API35，研究 APK，非当前候选）：1.51MB PNG 从 32KiB 块改为 64KiB 块，平均快 139.7ms；中位 PNG 约 20ms；JPEG 和树只差 3–10ms。最大 PNG 主要耗时其实在 Android 侧三次大 JSON 处理（各约 40ms），base64 只占约 5ms。
- 代价：`865d816` 生产代码 +31/−14，外加两端 chunk 校验参数和契约样本。三个常量各自有理由：文件作业块不必变；回复上限必须容纳 64KiB base64 的转义最坏情况（研究算出约 174,768B）。
- **判断**：值得保留，但不值得任何人再花时间。更简单的做法是只留一个块常量（文件也用 64KiB），可选，收益同样很小。真正有分量的简化是 F5。不建议再做 D（二进制/流）：研究显示 base64 不是瓶颈。

---

## 4. 建议重新裁决的事项（真实 A/B/C/D）

### R1 近期事件（方向 15；交付方的未批准选项「默认近期事件精简」）
| 选项 | 内容 | 优点 | 缺点 |
|---|---|---|---|
| A 删除交付 | 删除 ring、来源保护、冻结、windows 重编码和 host 渲染；只保留 F3 中的事件唤醒 | 约 780 行和 10–14% 正文一起消失；状态最少 | 失去被动捕获瞬时 toast/pane 文本（真实证据中从未出现） |
| B 仅文字提示（**推荐**） | 只记录带非空 text/description 的 toast、pane、通知状态，存最近 N 条纯文本；动作区间内出现才打印 `toast "…"` 一行；不带 seq/clock/type/source/range/retained 元数据；不冻结；cached 不显示 | 保住唯一可能有用的信号；几乎零正文成本；实现约 1/5 | 「区间不完整」这类语义不再表达。我认为 LLM 无法据此行动，所以不算损失 |
| C 交付方 A | 默认打印摘要（来源、区间、淘汰）加文字提示；明细走新阅读选项 | 默认正文减少 | ring、冻结、元数据全保留，还多一个选项和摘要投影 |
| D 现状 | — | 无改动 | 已证的正文税 |

推荐 B。如果补证 E1（§6）显示主流 App 在 toast/pane 上也几乎不产生可投影文本，就改选 A。

### R2 settle 与 observedChange（接点 3；涉及 16–19、39，非本专题独占）
| 选项 | 内容 | 优点 | 缺点 |
|---|---|---|---|
| A（**推荐**） | settle 改为 `stable: true/false`（连续两份签名相同，不要求基线完整，不区分有没有 before），loading 作为原因；「变化」只看 observedChange；系统动作也用缓存树作为 before（与 click 一致）或者干脆不报 change | 去掉恒定字段和矛盾组合；partial 页也能提前结束采样，省约 1.5s/动作 | 失去 A→B→A 闪变这类过程信息（LLM 不会据此行动）；契约和文档要改 |
| B | 保留四态，但在恒定场景（before==null、partial 基线）下不输出 settle/change | 改动最小 | 规则更多，partial 页仍然耗满 settleMs |
| C 现状 | — | — | F2 |

### R3 事件唤醒节流（方向 14）
| A（**推荐**）保留唤醒，加最小间隔 `max(100ms, 上次采集耗时)` | B 只有窗口级事件才立即唤醒，内容事件走计时器 | C 回退为纯 250ms 计时器 | D 现状 |
|---|---|---|---|
| 一行常量，消除忙循环风险，保住大部分相位收益 | 语义清楚，但仍要维护事件分类 | 最简单，但失去「窗口出现就立即检查」 | 风险未测 |

### R4 树字节预算（方向 40，与 11–13 的 8192/8MiB 选择相关）
| 选项 | 内容 | 优点 | 缺点/未知 |
|---|---|---|---|
| A 现状 | 采集下界＋投影后 fit＋发布硬拒 | 已实现，测试完整 | 网页上实际节点上限约 6k；三道闸；fit 裁剪顺序与采集优先级不一致 |
| B（**推荐**）schema 瘦身＋限制投影文本 | 省略默认/null 字段；actions 只出 id（label 非空时才带）；`row.label` 改为引用行节点 REF，或截断到 ≤256B 并加 hash 作身份；保留 fit 作为罕见兜底，删掉采集期原始下界或与之合并 | 估计每节点降到 300–500B，8192 节点约 3–4MiB，节点上限重新成为真实约束；对 LLM 和 host 都更省 | 跨 Kotlin/Python 契约改动（wire-contract 重新导出）；身份比较要改用 label hash。我没核实 host Pydantic 模型对缺省字段的容忍度 |
| C | 只限制单节点投影文本（row.label、name ≤ N），其余不变 | 改动小，直接消除复制放大 | 默认字段冗余仍在，约 6k 节点上限不变 |
| D | 把 8192 降回 4096、8MiB 不变 | 字节闸几乎不触发 | 违背用户已选的 C（8192）；丢掉长表完整性 |

### R5 zoom 模型（方向 37，用户 U23 已选「当前唯一完整原生源」）
| 选项 | 内容 | 优点 | 缺点 |
|---|---|---|---|
| A 现状 | 每租约一帧；新截图、派发、树签名变化、采样失败、结束时失效 | 已实现 | 长网页几乎每次 fresh snapshot 都会让 zoom 失效；价值未证 |
| B（**推荐**，仍在 U23 授权内） | 保留一帧，但只在新截图、派发、租约结束时失效；坐标帧另行在树变化时失效（一个布尔值，或沿用现有 shape 加 TTL 检查）；REF 区域裁剪到显示区内 | zoom 语义就是「读那一帧」，失效规则与之吻合 | 坐标帧与源寿命多一个位 |
| C | zoom = 新截一帧原生 PNG 再裁剪，明确标新时点；不保留源 | 零常驻内存，无失效状态 | 与 LLM 看过的那一帧可能不同（动画、toast） |
| D 删除 zoom | 需要细节时用 `screenshot --image-format png` | 删掉约 450 行（`cace585`） | 高分辨率设备上读小字的能力只能靠整帧 PNG（1.5MB 级） |

剩余未知：zoom 在高分辨率设备上的真实阅读收益（见 §6 E4）。若补证显示收益很小，D 是合理的。

> 传输常量（R 无需列）：建议维持 B（现状），不再裁决。

---

## 5. 对既有内部 review 与交付方声明的对照

- **`final-maintainability.md:47` 不采纳「settle 与 observedChange 合成一个状态」**，`external-observation-audit.md:23` 的理由是「A→B→A→A 端点 unchanged 而 settle=unsettled 的差异正确，合并会丢过程/截止信息」。我不同意：(1) 他们保护的过程信息在系统动作上是恒定值，在 partial 基线下也是恒定值（F2），恒定值不携带信息；(2) 「闪变后恢复」没有对应的 LLM 决策；(3) 我推荐的 A 并没有把两者合成一个，而是让两者各自只回答一个问题（稳定 / 变化），并消除恒定和矛盾组合。他们的推理只在「两份完整树、有基线」这种少数情形下成立。
- **`final-maintainability.md:32`「目录/tree/image 的独立状态有各自失败消费者，不能为合并代码退回一个总成功标志」**：成立的是「不能退回总成功标志」；不成立的是「因此必须三个对等分块附件」。目录内联同样保留独立失败语义（F5）。
- **`final-maintainability.md:39` 认为 `UiTreeBudget` 解决了投影后的真实 8MiB 限制**：解决了一致性，这一点我同意（上次外评「采集-1」的安装/发布不一致是真缺陷，3f45af6 修对了方向）。但没有追问字节从哪里来。交付稿把 8,377,092B/127 REF 称为「实际dense树」，容易被读成真实页面；原件明确是自建 fixture，且其成因恰恰是投影复制。
- **`u25-pending-questions-review.md` 第 2 题推荐「摘要＋文字提示」**：方向对，但仍保留了来源、区间、淘汰元数据和一个新阅读选项。它的依据是「真实小页事件明细占比较大」，而它自己的证据已经足以支持更彻底的 B 或 A（132/132 无文字）。
- **12 阶段 `result.md`**：自述没有 A/B、不称提速，这很诚实。但交付稿（candidate-final 第 24 行）把「事件只促发真实条件检查」列为已完成能力，没有说明它去掉了节流。
- **R/P 内存原件与 `software-canvas-lifetime.md`**：证据本身严谨，并明确声明不是全进程峰值。我同意他们的边界声明；补充一点：所有图像证据都在 720×1280 上，JPEG 没有缩放，所以 zoom 的价值和高分辨率下的内存都还没有证据（F7）。
- **上次外评（`androidctl-candidate-review-final.md`）**：其「效率-1」（只读命令重放全部事件）、「闭环-7」（settle 与 observedChange 双权威）、「采集-1」（字节双口径）、「图像-1」（zoom 冲掉基线）我都独立复核过并同意。其中「采集-1」和「图像-1」已由 3f45af6 修复，另两项被内部 review 驳回，我认为驳回不成立。

---

## 6. 未覆盖范围与必要补证

**未覆盖或未核实**：`ImageMarks.kt` 的绘制细节；`UiCondition.kt` 的各条件语义（只核了循环和唤醒）；`FreshNodes`/`UiRefRepair` 的内部正确性；host `history.py` 的配额与 diff 基线（属其它专题）；Python wire 模型对缺省字段的容忍度（影响 R4-B 的改动面）；AOSP 上游未追溯（本专题结论不依赖上游行号）。

**只列会改变判断的补证**：
- **E1 真实 App toast/pane 文本出现率**：在 3–5 个会弹 toast 或 snackbar 的真实 App 动作上，看事件 ring 是否捕获到文字。若几乎为 0，R1 选 A（整体删除）；若常见且有决策价值，R1 选 B。
- **E2 事件风暴下的采样次数**：在一个有持续动画的页面（视频进度、加载动画）上，对比 eb78116 前后同一个 click 的 settle `samples` 和 settle 时长。若候选采样数明显更多（例如 ≥3 倍），R3 必须做（A 或 C）；若相近，R3 可降为可选。
- **E3 schema 瘦身后的字节**：用已保存的真实树（09 长表 5.55MB、seq70/72）离线模拟 R4-B 的序列化，得到每节点字节数。若能降到 ≤600B/节点，R4-B 可以撤掉采集期字节闸；否则保留 A，只做 C。这一项可以完全离线完成，不需要设备。
- **E4 高分辨率设备上的 zoom 与内存**：在 1080×2400 级 AVD 上做一次「默认 JPEG 读不清 → zoom 读清」的真实页面对照，同时记录 `dumpsys meminfo`。若 JPEG 已可读，R5 倾向 D；若 zoom 明显有用而内存可接受，选 B。
