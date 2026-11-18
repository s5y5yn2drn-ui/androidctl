# 19 节点采集、语义与等待复审（E9 / target / 动作次序 / settle / schema）

评审对象：候选 `3f45af6`（`review-copy`）；实施方 U26 材料（`T` = `NEW/ultragoal/2026-11-11-next-improve`）。
我写的离线脚本都在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents4/19/`：`order.py`（逐节点比对）、`last64.py`（动作 64 的位置）、`targets.py`（target 成因）、`marks.py`（按 ImageMarks 规则重算标注）、`names.py`（命名归并），还有下载的 `AccessibilityService-15.java`（android-15.0.0_r1）。所有输入都按参数传入，用 `python3 -I` 运行。

## 1. 结论摘要

1. **E9 建议保留，把握中等（约 65%）。** 在静态页面上，cache 臂和 E9 接近：collect 655/1047ms 对 606/580ms。single 明显更慢：1909/2209ms 对 E9 的 1024/399ms。但实验只测了静态页面上的 snapshot。上游源码表明，cache 路线在"动作后 settle"路径上有三种 E9 没有的退化，实施方都没写成可观察差别：只接收最后一次交互的预取；滚动事件后 150ms 内不预取；内容/滚动事件会逐出子树缓存。外评 P-E9 真正想问的就是 settle 路径，这个实验没有回答。
2. **动作次序是确定缺陷，不只是"阅读噪声"。** 设备端 `signature` 和 `contentSignature` 按原数组次序哈希 `actions`（源码核对）。6 份同页原件全部显示：每个 worker 的首采是 Chrome 新建次序，第二采起是 provider 缓存次序。因此在新连接后的首个 Chrome 快照上做任何动作（包括无效动作），端点 `observedChange.content` 都会报 changed，已保留的截图也会失效。推荐实施方的 B：读事实前按 ID 规整。
3. **D3 只改 host 不够。** 设备端 `publicTarget` 同时决定截图标注（上限 40 个）。三份 Chrome 原件离线重算：40 个标注里有 25–33 个是只带 CONTEXT_CLICK 的文字节点，真正可点的目标被挤掉 14–21 个。两端共用一条规则、只把 CONTEXT_CLICK 归为普通动作，比实施方的 B 和我方上轮的白名单都干净。
4. **canonical 本身应改为稀疏表示，不应只加 wire codec。** Chrome 每节点约 1546B，8MiB 实际只能装约 5.4k 节点，8192 在网页上达不到。稀疏表示约 774B/节点，能装下 8192。assemble 阶段 600–785ms 已经和 E9 collect 同量级甚至更大。
5. D8 成立。D16 部分成立：缺席不等于永久消失，但 wait 耗满预算，而且和 settle 对缺席的处理不一致。settle 维持"非纯稳定"的语义：实施方成立。最小采样间隔只建议用固定下限，不按"上次耗时"成比例。
6. 最大未知：E9 和 cache 在"滚动或变化中的长网页 settle"上的每采样成本和 partial 率。这一项会翻转 E9 的取舍，值得做一次窄实验。

## 2. 独立判断

### 2.1 E9 去留

**我自己重算的时间（原件核对：两份 collector log tail）**

单位 ms。prepared→collected 是 collect，prepared→installed 是设备端整采样（含 assemble/fit），installed→publication 另计。

| 轮次 / 页面 | 臂（worker 内次序） | collect | assemble | 整采样 | publication | 查询计数 |
|---|---|---:|---:|---:|---:|---|
| R1 4226 | A E9 首 | 1024 | 677 | 1703 | 174 | direct 1036 / opp 3189 |
| R1 | A single 热 | 1909 | 672 | 2584 | 211 | public 4225 |
| R1 | B single 首 | 2209 | 658 | 2871 | 166 | public 4225 |
| R1 | B E9 热 | 399 | 628 | 1029 | 131 | direct 515 / opp 3710 |
| R2 4243 | A E9 首 | 606 | 600 | 1209 | 242 | root 2 / direct 360 / opp 3882 |
| R2 | A cache 热 | 655 | 785 | 1442 | 169 | root 2 / public 4242 |
| R2 | B cache 首 | 1047 | 407 | 1456 | 144 | root 2 / public 4242 |
| R2 | B E9 热 | 580 | 591 | 1174 | 163 | root 2 / direct 406 / opp 3836 |

来源：`T/prototypes/route-review/collector/runtime-mumu-clone/commands/all-four-collector-log-tail/stdout.bin` 和 `T/prototypes/route-review/collector-cache/runtime-mumu-clone/commands/all-four-cache-collector-log-tail/stdout.bin`。与两份 result.md 一致。

- **公平性。** 两组都是交叉设计（每个 worker 内首、热顺序相反），每格 n=1，所以臂效应和次序效应分不开。同一 worker 内两采相隔约 50s（R1）、约 75–120s（R2），中间有人工操作间隔，不是 settle 那样的背靠背采样。"热"只表示 worker 已连接、Chrome provider 缓存已建立；后者有原件证据，见 2.4。
- **能下的结论。** R2-A 中，cache 占了"热"的便宜，仍比 E9 首采慢（655 对 606）；R2-B 中 cache 首采 1047 对 E9 热 580。所以静态页面上 cache 的 collect 约为 E9 的 1.1–1.8 倍。single 是 E9 的 1.9–5.5 倍。CLI 总时间 3.2–3.6s 里约有 2s 是固定的主机进程和交付开销，所以 3.485/3.500 这类数字掩盖了 collect 的差别。
- **更重要的一点（离线重算）。** E9 的 collect（399–1024ms）已经和 assemble（591–785ms）同量级，甚至更小。在静态页面上，cache 和 E9 的差距（约 50–470ms）小于一次 assemble。
- **实验漏了一项计数。** cache 臂只记了公开调用次数 4242，没记 cache 命中数和实际 IPC 次数（`collector-cache/README.md:17` 自己承认）。所以无法知道这一页上 cache 臂"退化成 single"的程度有多大。

**把实施方那句话翻译成可观察差别（源码核对）**

先确认 UiAutomation 宿主会把事件转给 AIC/cache：`UiAutomation.IAccessibilityServiceClientImpl extends IAccessibilityServiceClientWrapper`（android-15 UiAutomation.java:1848），而 wrapper 的 `onAccessibilityEvent` 先调 `AccessibilityInteractionClient.onAccessibilityEvent`，再回调服务（我下载的 `AccessibilityService-15.java:2976–2986`）。所以下面这些平台机制对 cache 路线都生效。

| # | cache 路线独有现象 | 机制（AOSP15） | 触发条件 | 后果 | 能否简单规避 |
|---|---|---|---|---|---|
| C1 | 晚到的预取被丢弃 | `AIC.setPrefetchAccessibilityNodeInfoResult:1269–1290` 只接收 `mInteractionIdWaitingForPrefetchResult` 那一次交互的结果；而每次 cache 未命中都会在 primary 返回后把这个 ID 改成新的（`AIC:709–712`） | provider 的 UI 线程忙，预取比采集器下一次未命中还晚到。动作后最常见 | 未命中增多，向 single 退化（静态页 single 是 E9 的 2–5 倍） | 不能。要改 AIC 私有状态 |
| C2 | 滚动后不预取 | `AIC:648–651` 的 `isWindowScrolling`；阈值 `DISABLE_PREFETCHING_FOR_SCROLLING_MILLIS` = 1.5×100ms（`AIC:116–117`）；由 `TYPE_VIEW_SCROLLED` 更新（`AIC:1153–1160`） | scroll、swipe、scroll-until 之后的 settle；fling 期间持续发生 | 该窗口在滚动期间等同 single | 不能（静态私有状态） |
| C3 | 采集中途被逐出 | `AccessibilityCache.onAccessibilityEvent`：内容变化/滚动时 `clearSubTreeLocked`，窗口状态变化时 `clear()`（AC15 约 :260–288） | 页面正在变化（Chrome 变化时持续发 content-changed） | 更多 IPC。但读到的数据反而更新 | 不需要规避；这是正确性和速度的交换 |
| C4 | 跨窗口的旧来源 | 旧预取的交互 ID 在下一窗口的 root primary 返回之前仍是"等待中"。provider UI 线程按 FIFO 处理，所以上一采样延迟的预取恰好会在新窗口 enable 之后、新 root primary 之前送达，并被 `cache.add` 收下 | settle 背靠背采样，且 provider 忙 | 第 N+1 采样混入第 N 采样时刻的子树。"两份相同即稳定"可能被欺骗 | 用公开 API 没有干净的办法；clearCache 再 enable 不能取消在途回调。陈旧程度上限约一个采样周期 |
| C5 | child 调用只能等满 5s | `AIC:1415–1431` 的 `TIMEOUT_INTERACTION_MILLIS`=5000 | root 读完之后 provider 才卡住 | 超出 settle 的截止甚至请求截止，回执变成 timeout/expired，而不是带可靠局部 REF 的 partial；单 executor 后续请求排队（`UiBackend.kt:21,110,143`） | 只能把超时线程交出去；同步 Binder 本身取消不了 |

C5 的分量需要打折扣：E9 也用公开 `window.root`（`TreeCollector.kt:233`），leashed child 也走公开 `getChild`（`CollectionNodes.kt:40`）。provider 一开始就卡住时，E9 同样会在 root 上等最多 5s。E9 的优势只体现在"root 读完之后才卡住"这个较窄的情形。

**E9 独有的风险（源码核对）**

- `CollectionNodes.kt:130` 的 `data.dataAvail() != 0` 会拒收整个回包。AIDL 最常见的演进方式是在末尾追加参数，这种无害变化会被它变成"每个 child 都等到截止，然后只剩根、partial"。这是自找的脆弱性：只读 `node` 和 `interactionId`、忽略尾部数据，就能扛住追加式演进。
- AOSP 14/15/16/17 回调签名没有变化（实施方 §2 已核，我抽查了 `agents/01-aosp/*-IAccessibilityInteractionConnectionCallback.aidl`）。厂商 ROM 改这个 AIDL 会破坏它自己的无障碍框架，可能性低，但没有证据。
- E9 在一次采集内不会因事件逐出节点，所以树内部的时间一致性比 C3 差。但 single 同样如此，所以这算不上 E9 独有的缺点。

**推荐**

保留 E9（把握中等），同时做两件小事：

1. 放宽尾部检查（只校验已知前缀）。
2. 每个连接内，第一次出现"拒收或未知事务码"后，本连接改用公开 `getChild` 降级。这不是通用兼容层，只有一条降级分支；它把"新平台上永久只剩根"变成"慢但完整"。

理由：E9 维护的是自写回包 codec 加 finalize，约 6 个只给 E9 用的隐藏成员；cache 开关的隐藏句柄是两条路线都要的（`AccessibilityConnection.kt:35–41`），换 cache 并不能完全摆脱隐藏 API。而 C1–C3 会在 settle、滚动等最常见的动作后路径上把 cache 推向 single。single 在 4.2k 节点页上 collect 要 1.9–2.2s，已经超过默认 settle 窗口 1500ms，而 settle 里的采集截止就是 settle 窗口（`UiBudget.collectionDeadline:42`、`UiPostAction.kt:39,143`），结果是动作后的首采 partial。

**是否回答了 P-E9：没有。** 外评 P-E9 关心的是"热态交替"，以及能否放进默认动作后的预算；实验只做了静态页面的 snapshot，每格 1 次，间隔几十秒。没回答的部分（滚动或变化中的 settle）恰恰是 C1–C3 生效的地方，**会改变取舍**：如果那种场景下 cache 和 E9 仍然接近，就应该换成 cache，删掉手写 codec。

### 2.2 预算：8192 / 3000ms / 8MiB、partial、D8

- 三个约束各管一件事：`Wire.kt:20,27` 定义上限；`TreeCollector.kt:275–276` 做原始下界检查；`UiTreeBudget.fit` 最终定稿。这个分工合理（源码核对）。
- **网页上真正起作用的是字节上限。** decodedSize 6,532,479B / 4226 节点 ≈ 1546B/节点，8MiB ÷ 1546 ≈ **5,426 节点**（离线重算）。也就是说，这类 Chrome 页面永远到不了 8192；8192 只在稀疏的原生树上起作用。慢设备上时间先到：API34 软件模拟器 seq70 在 568 节点、3006ms 时 partial。
- partial 的语义（可靠局部 REF 照常可用，全 scope 的唯一性、gone、idle 被阻断）是合理的保守做法。
- **D8 确认（源码核对）。** 截止时间可以从三处变成 null，然后统一算进 `unavailableChildren`（`TreeCollector.kt:315–333`），不会记成 collectionLimit：
  - `CollectionNodes.kt:38` 查询前检查截止；
  - `:66` 是 `awaitPrimary` 等到截止，以及回包被拒收；
  - `:67` 是 primary 已到但已过截止。

  实施方的修法（只区分"确知的本地截止"和"一般未取得"）够用。

### 2.3 节点语义与 target（D2、D3）

先用原件核对（`targets.py`，输入 `a-e9-first.raw.json`）：4226 个节点里 4201 个是 target，其中 **3353 个仅因为 CONTEXT_CLICK**；可见 target 92 个，其中 66 个只有 CONTEXT_CLICK。

设备端 `publicTarget` 的消费者和保持现状的后果：

| 消费者 | 保持现状的后果 | 证据 |
|---|---|---|
| `ImageMarks.from:151–154`（上限 40，按树序取） | MuMu 页：40 个标注里 29 个是只带 CONTEXT_CLICK 的文字，26 个真目标漏掉 15 个。API34 页：33/40，漏 14/21。Addis 详情页：25/40，漏 21/36。`--marks` 在网页上基本失效 | 离线重算，`marks.py`，三份原件 |
| `Selection.visitChild:160`、`absorbs:364` 命名聚合边界 | Chrome 的链接节点自己带名字，受影响的未命名可点父节点只有 1 例（`e328647670`→"List"）。全页 `nameConsumed` 为 0：几乎每个文字节点都是边界，所以默认正文里文字重复出现，无法被父节点吸收 | `names.py` |
| `UiSelector.inSample` 按 selectionTarget 归并 | 现在用 `--text List` 选中的是文字子节点，不是可点的父节点；收窄后会选到父节点，点击对象更合理（推断） | 源码 |
| REF 是否可用 | **与 target 无关。** `UiRefs` 给所有采到的节点 REF，4226/4226 都可用。所以"REF 泛滥"只是 host 显示层面的问题 | 原件核对 |
| 树体积 | 与 target 无关，actions 总是全量写入 | 源码 `NodeFacts.kt:143–145` |

判断：

- 实施方 B（只改 host 的 interactive 筛选）会**留下标注缺陷**，还会制造第二个"目标"概念（"阅读资格"和"设备 target"）。调用者需要多学一个规则：interactive 不等于 target，被藏起来的节点要用 cached/all 再读一次。
- 我方上轮的白名单（只认带 label 的 custom action）会藏掉不带 label、却是唯一入口的自定义动作，太强。实施方在这点上是对的。
- **最干净的做法：** 两端共用一条谓词，只把 `ACTION_CONTEXT_CLICK`（16908348）放进 `ordinaryViewActions` 和 `ORDINARY_VIEW_ACTIONS`。未知的自定义动作仍然是 target；用 REF 加 ID 的通用动作执行能力不变。
  - 代价：命名、选择、标注三处行为变化（上表都是向好的方向），需要更新契约测试，契约样本需要重新导出。
  - 不新增任何状态或概念。

### 2.4 动作次序（题 5）

**原件核对（`order.py`、`last64.py`）。** 同一 worker 内两采只有 `actions` 一个字段不同：R1 两对各 830 个节点，R2-B 一对 833 个；每一处都是 64 从中间移到了末尾，ID 集合相同。

我还核对了每份原件里"带 click/expand/collapse 且 64 在末尾"的节点数：

| 原件 | 64 在末尾的节点数 | 判定 |
|---|---|---|
| preparation-page（上一个 worker 的唯一一采） | 0/842 | 新建次序 |
| a-e9-first | 6/842 | 新建次序 |
| b-single-first | 6/842 | 新建次序 |
| b-cache-first | 8/847 | 新建次序 |
| a-single-warm | 836/842 | 缓存次序 |
| b-e9-warm（R1） | 836/842 | 缓存次序 |
| a-cache-warm | 841/847 | 缓存次序 |
| b-e9-warm（R2） | 841/847 | 缓存次序 |
| API34 chrome-current-visible | 5/67 | 新建次序 |
| API34 chrome-addis-detail | 0/166 | 新建次序 |

**规律：** 新连接后的首采是 provider 新建次序；同一连接的第二采起是 provider 缓存次序。这与采集臂无关，三个 worker、6 对原件全部一致。

**上游机制（源码核对）**

- Chromium Beta158 `AccessibilityNodeInfoBuilder.java:357–372` 新建节点时先加 64、后加 click；`WebContentsAccessibilityImpl.java:1366–1385` 缓存命中时会 remove 再 add 64。
- AOSP `addAction` 会先删除同 ID 的动作再追加。
- AOSP parcel 对"标准且无 label"的动作走 bitmask，读回时按位序重建（`ANI15:4428–4450,4701–4709,4835–4843`）。观察到的数组不是按位序排的，说明 Chrome 经 AndroidX `AccessibilityActionCompat` 构造的是非单例动作（`mSerializationFlag=-1`），走的是保序路径。这一点是推断，与观测吻合。
- 结论：上游不保证动作次序；对标准动作来说，框架自己就会改写次序。

**设备端次序敏感是源码事实，不是推断（源码核对）**

- `UiSnapshot.kt:104` 的 `signatureFields` 包含 `NodeFacts.observationFields`，后者含 `"actions"`（`NodeFacts.kt:26`）。
- `contentSignature:130–141` 把 `node.opt("actions")` 原样放进 JSONArray 后哈希。
- `actions` 由 `node.actionList.mapIndexed` 生成（`NodeFacts.kt:145`），保留原序。
- 消费者：
  - `SettledSamples.accept`（`UiPostAction.kt:168`）；
  - `ObservedChange.compare`（`ObservedChange.kt:12`）；
  - `VisualInput.observed`（`VisualInput.kt:43`）；
  - host diff。

**由此得出的确定缺陷**

- 新连接后的首个 Chrome 快照 X（新建次序）之后，做任何动作，之后的采样都是缓存次序。`ObservedChange` 的 content 判 changed。页面较小、窗口内能放下两采时，settle 判 CHANGED；4k 节点页一般只放得下一采，settle 判 unsettled，但 `observedChange` 仍然是 changed。
- 无效点击会被报成"有变化"，LLM 会据此误判动作生效。
- 同理，`snapshot --screenshot` 之后再做一次新的 snapshot，保留的截图会因次序变化失效，坐标输入随之被拒。
- 页面真实变化之后，Chrome 新建的节点在第一、二采之间也会翻一次次序，settle 因此要多一采才能稳定。在 4k 节点页上约多 1–1.7s，或者直接变成 unsettled。
- 触发频率需设备，机制和前提已有原件和源码证据。

**评判 A/B/C/D**

- 推荐 B：读事实前按 ID 排序，排序之后再生成 `actions[i].label` 的 unavailable 下标。
- 次序没有任何消费者需要：`NodeActions.ids/resolve` 只按 ID 集合或 label 选动作（`NodeActions.kt:37–42`），框架自己也会改写次序。"raw 不再保收到的原序"不构成能力损失。
- label/custom action 的对应关系：只要排序在 `actionFacts` 之前（下标在排序后产生），ID、label、缺口三者就保持关联。同 ID 重复的情况，AOSP `addAction` 已按 ID 去重，四份原件里也是 0。
- C、D 留下设备端缺陷或双重表示，都不可取。

### 2.5 事件、等待、settle、变化

**消费者与下一步**

| 机制 | LLM 看到不同取值后会怎么做 | 判断 |
|---|---|---|
| #14 事件促发检查（`UiSignals`、`awaitNextCheck`） | 不可见，只影响延迟 | 保留 |
| #15 ring 与来源区间 | 默认几乎不用；排障时用 | 保留，默认隐藏 |
| #16 fill/check 局部核值 | 失败就重试或换方法；成功就继续 | 保留，最直接的结论 |
| #17 显式窗口与缺席 | 见 D16 | 改语义（见下） |
| #18 任一条件共享样本 | 不可见 | 保留 |
| #19 settle、ObservedChange、UiEffect | 实际只有一个决策："动作有没有效果，在哪里，页面还在不在动" | 计算可以分开，**呈现应合并为一个 change 结论**：`changed/unchanged/moving/unknown`，加 `where: target 字段 / 窗口 / 内容`。A→B→A 作为一个标志放进 JSON |
| #20 有界滚动查找 | 能找到就不必让 LLM 循环滚动 | 保留 |

- **settle 不改成纯稳定：实施方成立。** A→B→A 和 CHANGED 的区分有真实价值（`UiOutcomeTest.kt:461`）。系统动作借旧树做基线会引入陈旧比较，拒绝也成立。
- 但要注意：UNCHANGED 必须等满整个窗口（`UiPostAction.kt:64–68,207`），所以每个无效动作固定花 1500ms，这是"捕捉延迟效果"的代价。动作次序缺陷会把本应 UNCHANGED 的结果变成 CHANGED，在小页面上还会提前停止，掩盖延迟效果。
- **D16（源码核对）。** `waitUiCondition`（`UiCondition.kt:387–413`）不看 `absentWindow`。窗口不在时，根节点 failed（`TreeCollector.kt:111–118`），树变成 partial，`gone` 拿不到完整的负证据，于是一直整树重采到预算用完，最后报 failed。
  - 而 settle 对同一种缺席立即报 changed，并回退到默认 scope（`UiPostAction.kt:43–44,97–110`）。
  - "缺席不证明永久消失"成立：窗口 ID 按 token 稳定，被遮挡时可能暂时退出无障碍窗口列表。但 `gone` 的普通语义本来也只管"当前不可见"，不管永久性。
  - 所以对 `--gone` 来说缺席就应该算满足（并附 notice），其它条件则立即以 `windowAbsent` 失败（不可重试）。
  - 现行为属于**语义不一致加预算浪费**，比实施方说的"非缺陷"要严重。
- **采样风暴（源码核对加推断）。** 只要有相关事件，`awaitNextCheck` 立即返回（`UiBackend.kt:292–305`），250ms 计时器只在没有事件时起作用。小页面（约 30ms/采）遇到事件洪流（进度条、动画）时，每秒可能采几十次，每次都向 App UI 线程发 IPC。风险真实，但收益和伤害都没有量化。
  - 建议加一个**固定的采样起点间隔下限**（例如 100ms）：大页面上几乎没有影响，小页面上封住采样速率。
  - 不要用"上次耗时"之类成比例的下限：大页面 1.2s/采，1500ms 窗口本来只放得下 1–2 采，成比例的下限会让它只剩 1 采。我方上轮的"100ms 或上次耗时"需要更正。

### 2.6 schema：改 canonical，不加 codec

- 实施方实测（`T/prototypes/route-review/schema/README.md`）：只简写动作省 18.75%，再省略默认值共省 49.91%，即 6,532,479 → 3,272,319B。实施方说"只在 wire 层压缩不增加 fit 容量"，这是对的，**恰好说明应该改 canonical 本身。**
- 改成稀疏 canonical 后约 774B/节点，8MiB 可装约 10.8k 节点，**8192 才真正可达**（离线重算）。同一页面还会减少：
  - assemble 和 fit 的 JSON 处理量（目前 600–785ms/采，数值来自 log，可能是单次采样里最大的一项）；
  - signature 的序列化量；
  - 主机解码和内存。
- 代价：所有 `getBoolean/getString/has` 消费者要改用带默认值的读取，还有契约样本、Kotlin 与 Python 的默认值表。
  - 规则可以非常简单：**缺键等于默认值；"未知"只由 `unavailable` 表达**。现在 `readField` 读取失败本来就写 null 并另记 unavailable（`NodeFacts.kt:203–214`），所以缺键不会和未知混淆。
  - 动作改为 `actions: [ids]`，只在 label 非空时附 `actionLabels: {id: label}`。
- 这是一次性返工。按用户偏好，应该做。
- 效果为推断，需设备核实：Android 上 assemble 实际能省多少 CPU 没有量过。

### 2.7 普通无障碍服务做唯一节点宿主

- 不建议（推断加已有原件）。
- 已见差距：
  - 非 tool 服务读不到 `dataSensitive` 节点；
  - 需要用户在设置里手动开启，而且是可见状态；
  - 不少 App 会检测并限制非 tool 无障碍服务；
  - Android 13 起侧载应用受受限设置约束；
  - 节点、身份、settle 都要和 shell 的输入、截图、租约做跨进程对齐，动作回执和截止也要重定。
- 宣称的 SCVH 嵌入树收益，在 API14/37 上 shell 都取不到嵌入的 ANI（实施方 §3 有原件），这项收益没有落地。BrowserSupport 只用来满足 Chrome 的"需要有无障碍服务"前提，这个分工已经足够。

### 2.8 组合一致性与最干净的模型

现在这条链上的重复和不一致：

1. **同一棵树反复算 signature：** `VisualInput.observed` 对前后两棵各算一次，`SettledSamples.accept` 一次，`ObservedChange.compare` 对两端算 contentSignature，baseline 再一次。4k 节点时每次都要序列化数 MB 的 JSON。成本没量过，属于推断。
2. **缺席的失败语义不一致：** settle 判 changed 并回退；wait 等到超时。
3. **截止和 provider null 混在一起**（D8）。
4. **次序噪声**由 signature、diff、image 三处各自承受。

最干净的模型（设计建议）：

- 每次采样产出一份**定稿记录**，包含：规范化的节点（动作按 ID 排序、稀疏 canonical）、每个 root 的状态和原因（`full | localDeadline | nodeLimit | byteLimit | providerNull | windowAbsent`），以及**在定稿时只算一次**的 `signature/contentSignature`。
- 安装 REF 和全部下游消费者（settle、wait 条件、ObservedChange、UiEffect、截图失效、host diff）都只读这份记录。
- 窗口缺席是 root 的一种一等状态，wait 和 settle 用同一张解释表。
- 呈现层只输出一个 `change` 结论。

## 3. 对照实施方

| 实施方结论 | 判定 | 证据与差异 |
|---|---|---|
| 本轮保 A/E9 | 成立（理由需补） | 我同意保留，但关键理由是 settle 和滚动路径上的 C1–C3。实施方只写了"来源与 5s 等待交换"，没指出这三点 |
| "E9 保本 collect child 来源及剩余截止等待；cache 交换旧来源与公开 child 5s 等待" | 部分成立 | 5s：E9 的 root 和 leashed 同样受限，优势较窄（`TreeCollector.kt:233`、`CollectionNodes.kt:40`）。旧来源（C4）成立，而且在 settle 背靠背采样时 FIFO 会让它更容易发生。漏了 C1、C3 |
| "cache 有事件/滚动抑制责任"（collector-cache review:52） | 成立 | 我已核实 UiAutomation 经 wrapper 把事件转给 AIC（`AccessibilityService-15.java:2976–2986`、UiAutomation15:1848） |
| "无需为已知交换再造实验" | 对 E9 不成立；对动作次序成立 | settle 和滚动路径上的成本差会翻转 E9 的取舍，值得做一次窄实验（见 §7） |
| "callback 不建通用告警/fallback" | 部分成立 | 不需要通用系统；但"放宽尾部检查"和"连接内一次降级"成本很低，能避免最坏的失败形态 |
| D8 确定诊断合并 | 成立 | 源码核对 |
| D3：只改 host interactive | 不成立 | 留下标注缺陷（40 个里 25–33 个被占），并增加"阅读资格"这一概念 |
| D3：不应连 label-less custom 一起删 | 成立 | 更正我方 |
| 题 5 推荐 B | 成立，但定性应为缺陷 | 实施方 §8.3 称之为"比较语义取舍""设备影响是源码推断"。我核了源码：签名确实按原序包含 actions，前提条件也已由原件证实 |
| schema：保 canonical、不加 codec | 一半成立 | 不加 codec 成立；保 canonical 不成立。改 canonical 能提高容量、减少 assemble 成本 |
| D16 不是确定缺陷 | 部分成立 | 缺席不等于永久消失，这点成立；但对 gone 来说缺席就应算满足，现行为浪费预算且与 settle 不一致 |
| 拒绝 settle 纯稳定、拒绝系统动作借旧树 | 成立 | 源码加 `UiOutcomeTest` |
| 最小采样间隔"未测不称必须" | 部分成立 | 加一个固定小下限风险低，不必先测；成比例的下限会伤害大页面 |
| 变化字段"回答不同问题，宜先简化呈现" | 成立 | 我进一步建议在呈现层合并为一个 change 结论 |

## 4. 对上轮外评（我方）的更正

1. **P-E9 的阈值表设计不对。** 它只按静态热采的 collect 中位数（≤1.5s / 约 3s）分支，没有覆盖 settle 路径。按我方自己的表，本轮结果（single 热采 1.9s，cache 0.66s）会指向"换 cache"；但这个结论没有考虑 C1–C3。
2. **"不带 E9 约为 E9 的 1.8–2.7 倍"过于精确。** 同一 APK 的实测是 1.9–5.5 倍（受次序混杂影响）。
3. **"8MiB 约 6.4k 节点"偏高。** 这一页按每节点 1546B 计算约 5.4k。
4. **D3 的两条措辞要改。** "默认输出多出一批无操作意义的 REF"：REF 本来就对所有节点可用，问题只在 host 显示和标注。"两端改成白名单"太强。正确做法是两端只把 CONTEXT_CLICK 归为普通动作。
5. **D16 的理由写错了。** "窗口 ID 不会复用，空等没有意义"：ID 不复用并不等于缺席即永久。结论（不应空等）仍然成立，但理由应换成"gone 语义只管当前，以及与 settle 一致"。
6. **"最小采样间隔 100ms 或上次耗时"的后半句要删。**
7. **E9 失效的描述不准。** "失效方式是静默退化"：实际表现是 partial、只剩根、有 `unavailableChildren`，不是返回一棵 full 的假树；"静默"指的是没有专门的诊断。

## 5. 确定缺陷

| # | 位置 | 可观察后果 | 证据类别 |
|---|---|---|---|
| F1 | `NodeFacts.kt:145`（`actionList.mapIndexed` 保留原序）+ `UiSnapshot.kt:104,130–141` | Chrome 上，新连接后的首个快照之后的任何动作：`observedChange.content=changed`（小页面上 settle 判 CHANGED 并提前停止）；保留的截图失效；host diff 每对多出 17/830 条噪声；页面真实变化后 settle 需要多一采 | 原件核对（6 对原件，次序规律一致）+ 源码核对；触发频率需设备 |
| F2 | `NodeActions.kt`（`ordinaryViewActions` 缺 16908348）、`ui_view.py:53`；`ImageMarks.kt:151–154` | `--marks` 在 Chrome 页上 40 个名额被 CONTEXT_CLICK 文字占 25–33 个，14–21 个可点目标得不到标注；`--interactive` 不起筛选作用 | 离线重算（3 份原件，`marks.py`、`targets.py`）+ 源码核对 |
| F3 | `CollectionNodes.kt:38,66,67` → `TreeCollector.kt:315–333`（D8） | 本地截止被报成 `unavailableChildren`，没有 collectionLimit | 源码核对 |
| F4 | `UiCondition.kt:387–413`（不看 absentWindow），对照 `UiPostAction.kt:43–44,97–110` | `wait --window W --gone` 在 W 缺席时整树重采到超时再报 failed；同一种缺席在 settle 中立即判 changed | 源码核对 |
| F5（脆弱性，当前未触发） | `CollectionNodes.kt:130` 的 `dataAvail()!=0` | 平台在 callback AIDL 末尾追加参数时，所有 child 都等到截止，只剩根、partial | 源码核对加推断 |

## 6. 需要用户裁决的事项

1. **E9 / cache / single**

   | 方案 | 能力 | 维护 |
   |---|---|---|
   | E9（推荐，附放宽尾部检查和连接内一次降级） | settle、滚动路径预计最稳；树内部时间一致性一般 | 自写 codec 加 finalize，每个平台版本都要核 |
   | cache | 静态页面接近 E9；滚动和变化时退化，有跨窗口旧来源，child 只能等满 5s | 删除手写 codec，cache 开关的隐藏句柄仍需保留 |
   | single | 4.2k 节点页 collect 1.9–2.2s，超过 settle 窗口 | 最少 |

   未知：§7 V1。

2. **动作次序：B（推荐，按缺陷修复处理）还是 D。** 选 B 后，raw 不再保收到的原序，这没有能力损失。

3. **target 规则**

   | 方案 | 说明 |
   |---|---|
   | E：两端同时只把 CONTEXT_CLICK 归为普通动作（推荐） | 修好标注和 interactive，命名吸收更自然；代价是改契约和测试 |
   | 实施方 C（只改 host interactive） | 留下标注缺陷和第二个"目标"概念 |
   | 我方旧白名单 | 太强，否决 |

4. **canonical 稀疏化（推荐做）。** 使 8192 在网页上可达，并减少 assemble 和交付成本；一次性跨语言返工，需要一份默认值表。

5. **D16 语义。** gone 在窗口缺席时算满足（附 notice），其它条件立即以 `windowAbsent` 失败（推荐）；或者保持现状。

6. **采样节拍。** 加一个固定的采样起点间隔下限（例如 100ms，推荐），或不加。

7. **变化呈现。** 合并为一个 change 结论加 JSON 细节（推荐）；还是保留三套字段并列。

## 7. 会改变取舍的验证问题

- **V1（决定 E9 去留）。** 在约 4k 节点的 Chrome 长页上，分别执行"scroll 一屏后 settle"和"点击展开 List 后 settle"，E9 臂和 cache 臂交替进行，记录：
  - 每个 post 采样的 collect 时间；
  - full/partial；
  - cache 臂的实际 IPC 次数或命中数（要补埋点）；
  - 是否超出 settle 截止。

  如果 cache 和 E9 的差距仍小于一次 assemble，而且 partial 率相同：换 cache，删手写 codec。如果 cache 明显向 single 退化，或者出现 partial / 超出截止：保留 E9。
- **V2（决定 F1 的严重程度，不决定修不修）。** 在真实会话里，新连接后先 snapshot，再点一个无效目标，看端点是否报 changed，以及 Chrome 的 provider 缓存是否也会因页面变化或返回导航而重置。结果不改变"按 ID 排序"的推荐，只影响优先级。
- **V3（决定 canonical 稀疏化的优先级）。** 在设备上分别测稀疏与当前形态下 4k 节点树的 assemble、fit、signature CPU 时间。省下的超过约 300ms/采，就把它列为 settle 提速的第一优先；不足 100ms，就只按容量理由去做。
- **V4（决定节拍下限）。** 在小页面加持续事件（动画）的场景下，对比"无下限"和"100ms 下限"的采样数和目标 App 帧率。没有可见伤害，就把下限只当作防御。
- **不需要重做的：** 动作次序、D8、D3 的标注影响、D16 的控制流，现有原件和源码已经足够。

## 8. 未核实与局限

- 全部设备数据来自 MuMu 模拟器（API35）、单一 Chrome Beta158 页面，每格 n=1；R2 的首份 raw 缺失，A 对的节点级对比只有回执。API34 的两份是更早的软件模拟器历史原件。真机、Android 16/17、厂商 ROM 上的 E9 和 cache 行为都没有核。
- C1–C5 是源码推断出的机制，没有运行时 trace。C4 的发生频率未知。
- "新连接导致 Chrome provider 缓存重置"是根据 6 份原件规律做的推断，没有 provider 侧 trace；不排除有其它重置条件（页面变化之后可能同样发生，这只会加重 F1）。
- signature 的重复计算成本，以及稀疏化在 Android 上省多少 CPU，都没有量过。
- 标注重算用 `visible`、`refAvailable` 和 bounds 合法性近似 `ImageMarks.box` 的条件，没有做屏幕缩放的裁切，计数可能有 ±1–2 的误差。
- 没有运行任何包内脚本、Android 构建或设备。
