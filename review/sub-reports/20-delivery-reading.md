# 20 观察交付与使用成本：第三轮复审

评审者编号 20。只读。路径约定：`T` = `NEW/ultragoal/2026-11-11-next-improve`，`R` = `T/prototypes/route-review/presentation/runtime`，`O` = `T/stages/final-verification/paired-runtime`，源码行号对应 `3f45af6`（`review-copy/androidctl/src/androidctl/`）。
自写脚本与输出全部在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents4/20/`：`arms.py`、`tok.py`、`delta.py`（按响应间原生 input 增量精确分摊，闭合到原生总数）、`actfrac.py`、`spec_e.py`/`spec_e_lib.py`（更严格默认规格的离线重渲染，o200k，tiktoken 0.14.0）、`project_total.py`（固定轨迹总输入投影），输出 `arms.out`、`arms-chrome.out`、`delta.out`、`spec_e.out`、`project.out`。

## 1. 结论摘要

1. 原型真实效果：Contacts 原型臂总输入 629,770，低于旧基线 812,987（0.77×）；Chrome 原型臂 1,821,534，仍是旧基线 1,041,124 的 1.75×（设备不同、基线读不到网页正文，只能看量级）。Contacts 回退在这一次已收回，Chrome 没有。【原件核对＋离线重算】
2. Chrome 原型多 5 次 CLI，主因不是“短格式导致多读”。默认 `find` 只搜当前视口里的投影节点，离屏目标返回 `matches=0`，续读提示又指向同样看不到离屏节点的 `snapshot --cached`。执行者因此连滚 4 屏，每屏打印 5–7k token 的整页正文。这是一个确定的可用性缺陷，实施方和内部 review 都没有指出。【源码核对＋原件核对】
3. 原型只做了最浅的一层。Chrome 原型 stdout 里仍有 42% 字节是动作数组：`{"id":64}`、`{"id":16908348}`、`1024/2048/256/512` 几乎出现在每一行。网页上每个容器仍因 CONTEXT_CLICK 成为目标，带 REF 和动作。原生页去掉动作以后，结果行、头行、窗口行和事件行占了 58% 字节。【离线重算】
4. “非缓存输入更高”这个指标不能拿来判断格式：它约等于最终上下文大小加上服务端缓存未命中的噪声。Chrome 原型和原版的最终上下文只差约 1 万 token（126,027 对 115,598），46.7k 的差额里约 3.6 万来自缓存未命中。【离线重算】
5. 我给出的更严格默认规格（下称 E）在同四臂真实 stdout 上实算：Contacts 原生页每次动作约 250–450 token（基线约 600–750，原型约 1,000）；Chrome 视口约 1.2–2.2k（原型约 4.3–8.4k）。保持实际轨迹不变时，投影总输入为：Contacts 原型路线 542k（0.67× 基线），原版路线 877k（1.08×）；Chrome 原型路线 1.07M（1.02× 基线，且包含网页语义）。【离线重算，固定轨迹假设】
6. 题 3：我不同意“默认保留标准 numeric 动作 ID”。10 份 cli-index 记录里 `action --action-id` 一次都没有用过。标准动作都有专用命令或可由已交付的 name 唯一定位。推荐 E：省略隐含动作，只留 `name:label`/custom `{id}`，用 geometry 规则收起状态栏和导航栏，事件只在有文字或有缺口时打印，保存体与打印体用同一份紧凑 View。【设计建议】
7. 指引没有真正解决。四臂都整读了 actions.md（6.4k）；SKILL 里“Unicode 插入读 input”一句很可能把两个 Contacts 执行者都引去读 input.md。原型臂里，指引的携带量仍占总输入的 17–23%。但它已经不高于基线（基线 Contacts 指引携带 342k，原型 146k）。【原件核对＋推断】
8. diff：按能力价值，我推荐把“工作区持久基线”换成“显式指定比较对象文件”的无状态 diff（E 方案），或者删除。实施方反驳“unchanged 省正文不等价”，成立。zoom：实施方指出我方“仅 720×1280”是事实错误，成立；我仍推荐改成“新截图带 ROI”，理由是同源驻留的收益未证，而它和坐标资格共享状态的责任已证。【原件核对＋源码核对＋设计建议】
9. 最大未知：E 规格下执行者是否会因看不到隐含动作或窗口细目而走错，以及指引首屏重组后是否仍整读 actions.md。两者都需要新的模型对照，不能离线定。

## 2. 独立判断

### 2.1 紧凑阅读原型的真实效果（问题 1）

**四臂重算（截至各自首次 final；总 input 含 cached，output 含 reasoning）**【原件核对：`R/*/attempt-01/native.rollout.jsonl`；脚本 `arms.py`、`delta.py`】

| 臂 | 总 input | cached | 非缓存 | 最终上下文 | output（reasoning） | 响应/外层工具 | CLI | 工具交付 B | 截断 | 续读保存体 | wall s |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Contacts 原版 3f45af6 | 1,395,045 | 1,326,976 | 68,069 | 85,572 | 4,287 (1,236) | 26/25 | 20 | 237,192 | 1（`--all`） | 1（rg） | 252.9 |
| Contacts 原型 | 629,770 | 576,640 | 53,130 | 44,872 | 3,167 (521) | 19/18 | 15 | 96,165 | 0 | 0 | 174.6 |
| Chrome 原版 3f45af6 | 2,326,407 | 2,212,864 | 113,543 | 115,598 | 7,396 (1,581) | 30/29 | 15 | 337,855 | 2 | 10 | 676.6 |
| Chrome 原型 | 1,821,534 | 1,661,312 | 160,222 | 126,027 | 5,478 (1,220) | 25/24 | 20 | 363,229 | 0 | 0 | 604.4 |
| 旧 Contacts 基线 7d5be70 | 812,987 | — | 52,155 | 50,891 | — | 20 | 16 | — | — | — | 171.4 |
| 旧 Contacts 候选 99e31a | 1,461,258 | — | 83,210 | 97,187 | — | 24 | 16 | — | — | — | 207.0 |
| 旧 Chrome 基线 | 1,041,124 | — | 74,980 | 55,662 | — | 27 | 20 | — | — | — | 255.6 |
| 旧 Chrome 候选 | 2,447,717 | — | 122,853 | 122,611 | — | 29 | 18 | — | — | — | 300.1 |

实施方 comparison-*.md 里的数字我全部复算一致。截断按工具外层 `Warning: truncated output` 计：Contacts 原版 1 次，Chrome 原版 2 次，原型 0 次。

**总输入的精确分摊**：把每个响应的原生 input 增量归到它前面那次工具输出，乘以此后仍携带它的响应数，结果闭合到原生总数。【离线重算，`delta.out`】

| 臂 | 固定前缀（≈18k×响应数） | 指引 | CLI 正文 | 读保存体 |
|---|---:|---:|---:|---:|
| Contacts 基线 | 359k | **342k** | 112k | 0 |
| Contacts 原型 | 342k | 146k | 140k | 1k |
| Contacts 原版 | 468k | 226k | **648k** | 52k |
| Chrome 基线 | 485k | 251k | 202k | 104k |
| Chrome 原型 | 451k | 304k | **1,064k** | 2k |
| Chrome 原版 | 541k | 647k | 448k | **690k** |

几点直接可见的事实：

- 固定前缀随响应数线性增长。每少一个回合，省约 18k 加上当时的上下文长度。
- 这个 harness 把工具输出 JSON 转义后放进上下文，CLI 正文（引号密集）的实际增量是工具自报 token 的 1.17–1.39 倍，Markdown 指引只有 0.92–0.97 倍。所以 `role {JSON}` 行在这类 harness 下比按 o200k 计算的更贵。这属于 harness 特性，不作为产品要求。【离线重算】
- 非缓存输入约等于最终上下文，偏差来自服务端缓存未命中：Chrome 原型多出约 +34k，原版约 −2k。实施方用非缓存输入说明“原型更贵”，这条论据不可靠。【离线重算】

**(a) 原型实际做了哪些呈现改变**【源码核对：`T/prototypes/route-review/presentation/source/androidctl/ui_text.py`（103 行），patch 只动 4 个 host 文件】

- 动作：无 label 的标准动作改成短名；名字形如 `action-HEX` 的（64 无障碍焦点、256/512 移动粒度、1024/2048 HTML 导航、16908348 CONTEXT_CLICK、16908356 显示提示）保留成 `{"id":N}`；带 label 的保留完整对象。结果是这些 numeric 噪声项照样出现在每一行（Chrome 原型 stdout 中 `{"id":64}` 1,014 次、`16908348` 926 次），动作数组仍占 42% 字节。Contacts 原型也有 `64/256/512` 共 175 次。【离线重算，`actfrac.py`】
- 事件：保留 events 头行，只增加 detailsOmitted/contentStates 计数，去掉无文字的逐条明细。没有文字的时候头行仍约 90 token。
- 窗口：完全不动（状态栏、导航栏、IME、选中应用窗口各一行）。
- 头行：删 command/args 和重复的 windowsSource，保留 source.capture、snapshotFacts 全部字段。
- 结果行：删三个 baseline 时点和 effect 的 source、actions 预览；windows/selection/content 三段 JSON 全保留（这是通知回执修复后恢复的）。
- 没有默认上限；`--all`/`--diff` 绕过短显示；保存体仍是旧格式，形成“短 stdout + 详细保存体”两份正文。

与我方上轮 §2.2 十条相比：第 1 条只做了一半（隐含动作没省，numeric 噪声还在）；第 2 条没做（网页容器仍是目标）；第 3 条只做了一半（头行保留）；第 4 条没做；第 5 条做了一部分（capture 还在，绝对路径还在）；第 7 条做了一部分（三段维度 JSON 还在）；第 8 条没做；第 9 条只在“值完全相同”时去重；第 10 条一致。双正文仍在，实施方自己也承认“不是推荐架构”。

**(b) Chrome 原型 20 次 CLI 对原版 15 次：逐调用**【原件核对】

| 原型 | 原版 |
|---|---|
| devices（多 1） | — |
| session open、snapshot（4,957 tok，不截断） | session open、snapshot（被截断） |
| `find 'Addis Ababa' --context 2` → `matches=0`，提示 `omission {"reason":"find","nodes":84,"read":"snapshot --cached"}` | `find --all` → 命中离屏的第 7 行 |
| 4 次 `scroll down`（每次 4.1–5.7k tok 正文）→ click | `reveal --interactive`（被截断）→ `click --no-print` |
| snapshot → 2 次详情 scroll（6.5k、7.0k） | `snapshot --no-print` → 2 次 `snapshot REF --all --no-print` + 4 次读保存体（各 7.7k） |
| `back --help` | — |
| 2 次 click（URL）+ 3 次 back（最后一次 5.8k）+ snapshot（5.9k） | 2 次 click + 3 次 `back --no-print` + 每次读文件首尾 + snapshot |

结论：多出的 5 次 CLI 中，2 次来自“find 假阴性 → 4 屏滚动”替代“find --all + reveal”，1 次来自 devices，1 次来自 `--help`，其余是路线差异。主要成本差不在格式本身，而在两点：一是**默认 find 假阴性**；二是**每次动作都默认打印整页正文**。原版执行者大量使用 `--no-print` 加定向读文件，同样昂贵（读保存体携带 690k）。原型把每页正文缩小了约 53%（实施方离线计数，与我核对一致），但网页视口正文仍有 4–8k token。

默认 find 假阴性的源码依据：`ui_view.py:236-237` 在非 `--all` 时丢弃 `visible=False` 节点；`find` 只在投影后的元素上匹配（`ui_view.py:269-284`、`observation_nodes:600-623`）；find 的省略提示是固定的 `"read":"snapshot --cached"`（`omission`，`ui_view.py:533-541`）。也就是说，在 4,226 节点的树里，默认 find 只搜索视口中的 84 个节点。

**(c) 相对基线 `7d5be70` 的位置**【原件核对；Chrome 不同设备、基线无网页语义，仅给量级】

- Contacts（同一 API34 AVD、同任务、同模型）：原型 0.77× 基线，原版 3f45af6 1.72×，旧候选 1.80×。原型臂最终上下文 44.9k，低于基线的 50.9k。
- Chrome：原型 1.75× 基线，原版 2.23×，旧候选 2.35×。最终上下文：原型 126k，基线 56k。
- 每次 CLI 平均增量：Contacts 原型约 1,050（基线约 865，原版约 2,580）；Chrome 原型约 4,585（基线约 850，但基线没有网页正文）。

**(d) 回退收回了吗？差在哪？**

- Contacts 这一次已收回，但有偶然因素：Title 直接可见，少走一次失败加恢复，回合也更少。按每次正文看，原型仍是基线的 1.2–1.6 倍。【离线重算】
- Chrome 没有收回。剩余差额依次来自：(1) 网页每个容器都是目标，带 REF 和动作；(2) 保留的 numeric 噪声动作；(3) 每次 scroll/back 默认打印整个视口；(4) find 假阴性带来的额外滚动；(5) 指引仍整读 2–3 个 topic；(6) 头行、窗口、事件行每次约 250–350 token。

### 2.2 题 3：对 LLM 调用者最优的默认正文规格（问题 2）

我提出的 E 规格如下，可以和实施方的 A–D 并列比较。

**结果行**：`命令: 状态; 关键核值; 变化摘要; body=相对路径`。
- 正常 done：保留 fill/check 核值（`verification`、`actualValue`）和目标值变化（`value "A"->"B"`）；windows 只写 `windows=changed+N`，unchanged 不写；selection 和 content 并入 `changed`/`unchanged`。
- content unknown 必须写出来（例如 `content=unknown(partial)`）。实施方担心 `changed` 一个词会盖住“窗口变了、内容不可读”，这个担心成立，处理方式就是只在非 changed/unchanged 时展开该维度。
- failed、not sent、UNKNOWN、done 但 exit≠0：沿用原样，不压缩（与实施方一致）。

**头行**：只保留 `{"seq":N}`，以及确实不同于默认值的事实：cached 时的年龄、beforeAction、scope 不是单窗 full、partial/missing。capture 和 connection 只在与上一条不同或出现在 JSON 里时保留。实施方认为“单序号不区分 worker/connection/session”，但这种变化发生时 host 本来就会打印 `New worker: previous refs are invalid`（`contacts-original` #3 原件可见），所以默认行不需要每次重复。

**screen 行**：app、title、keyboard，加上 unreadable/partial 等缺口标志。缺口必须留在这一行（同我方旧第 6 条）。

**窗口**：采用几何规则，不按标题猜：`type=system`、`content=notRead`、未选中、贴显示边、高度不超过显示高的 1/8 的条带窗不打印；IME 窗在 screen 已写 keyboard=shown 时不打印；唯一的选中应用窗为 full 时不打印。其余一律保留（弹窗、浮层、他应用窗、未知类型）。实施方说“system 类型不等于系统栏”，对，所以规则用几何而不用类型。代价是：一个贴边、窄条、不可读的 system 浮层会被隐藏，而它可能挡住坐标点击。我推断这种情况在 REF 动作上没有后果，在坐标点击上有后果，所以截图或坐标路径应该强制显示全部窗口。

**事件**：有文字的事件逐条打印；无文字时只在 incomplete、evicted、deliveryOmitted、未监听、受保护等缺口状态下打印一行缺口摘要，否则什么都不打印。ring、cached 冻结和 raw 不变。这比我方上轮“删 ring 元数据、cached 不重放”收敛，接受了实施方关于不应删设备事实的反驳。

**节点行**：`role {"ref","name","value","detail",关键状态...}`。
- 状态只写偏离默认的：focused、selected、checked、disabled、offscreen、unreadable、scroll 方向、range。
- 动作只列“非隐含”项：custom 无 label 的 `{id}`；带 label 的写 `name:label`；还有 `long-click`（无 click 时）、`expand`、`collapse`、`dismiss`、`set-progress`。
- `click` 是否保留：在“目标由 click 以外的原因产生”（只可滚、只可编辑、只有 custom）时必须可区分。最简单的做法是保留短名 `click`，每行只多约 2 token（E 实算采用此做法）。我方上轮“有 REF 即可点”不成立，接受实施方反驳。
- 不列：focus、clear-focus、select、clear-selection、set-selection、show-on-screen、scroll-* / page-*、fill（textbox 上已有 value 表达）、64/256/512/1024/2048/16908348/16908356 等。
- chromeRole 不打印；roleDescription、tooltipText 与 role/name 相同时不打印；无名节点不打印 targetUrl。

**numeric action ID 是否必须默认出现：不必须。**
- 证据：10 份 cli-index（旧 4 臂、新 4 臂及其它）里 `action --action-id` 零次【原件核对】。我知道实施方要求不按少量任务里的调用频率决定去留，所以这里不以零次为删能力的依据，只用它说明“默认正文里放 ID”的阅读成本没有对应的使用收益。
- 能力不受影响：`action` 命令已经支持 `--action-label`（`ui_commands.py:1022-1061`）。click、long-click、focus、check、uncheck、expand、collapse、dismiss、ime、scroll-* 都有专用命令（`ui_commands.py:1233-1242`，以及 LOCATE_ACTIONS `:90-107`）。
- 剩下的标准动作（select、copy、show-tooltip 等）可以 `--all` 后用 `--action-id`；或者用一个很小的查找：`action REF --action NAME`，在当前缓存节点已交付的 `actions[]`（每项已带 device 给出的 `id`+`name`，`NodeFacts.kt:199`、`UiSemantics.kt:48`）里找唯一同名项。这不是新的名字协议，name 本来就是设备交付的事实。
- 实施方的“不新增动作名解析”这个约束不合理：它把 ID 的阅读成本放到每一次观察上，去换一个从未被使用的抄写路径。【设计建议】

**默认上限**：默认 View 已经按视口裁剪，E 规格下的实测最大值是 Chrome 2.2k token，不需要靠上限来省钱。上限只作为保底（例如 16k 字符），只作用于默认 View 和 find；显式 `--all` 不受默认上限约束。续读要真正前进，例如 `snapshot --cached --after REF`（新参数）；或者在省略行里直接给出被省部分的首个 REF，并说明用 `snapshot REF` 读子树（现有 `omission` 已部分支持，`ui_view.py:539-541`）。

**Title 反例核实**【原件核对：`O/contacts-2/evidence/cli/1794875072193693100-…/child.stdout`；`agents2/work10/variants.py:211-241`、`tables.md`】：成立。旧 Contacts 第 9 条 `snapshot --cached --all` 中，Title `e834208151` 位于节点行累计 19,969 字符处，C6@12k 会把它截掉。上轮 11,080 里有 8.4k（占 33k 节省的约 25%）来自这一条截断。但还有一个关键细节双方都没说：这个反例的成因是我方规则 1 让 `--all` 保留完整动作数组。按无动作数组计，Title 位于累计 4,969 字符处，在 12k 内。也就是说，如果 `--all` 也用紧凑动作，并且默认上限不作用于显式 `--all`，这个反例就不存在。影响大小：它使上轮 Contacts 的“1.13 倍”偏乐观，按同表 C1-5 不截断应为 19,458 token（1.99×）；但在 E 规格下（`--all` 也紧凑），该条为 4,198 token（`spec_e.out` 中 Contacts 原版 #11），全臂 9,915 token 是 20 次 CLI 的总和，与基线 9,775（16 次）相当。【离线重算】

**E 规格离线实算**（同一份真实 stdout，o200k，`spec_e.out`；E1 = 只改呈现、REF 全保；E2 = 再加网页容器目标收窄：只有隐含动作、无名无值的节点不给 REF 并折叠）

| 臂（3f45af6 真实 stdout） | 原始 | E1 | E2 | 每次动作典型值（E1） |
|---|---:|---:|---:|---:|
| Contacts 原版 20 条 | 46,090 | 9,915 | 9,012 | fill 313–357，click 141–253 |
| Contacts 原型 15 条 | 11,443 | 3,755 | 3,755 | fill 306–357 |
| Chrome 原版 15 条 | 33,647 | 7,644 | 5,927 | snapshot 2,209 / 1,578 |
| Chrome 原型 20 条 | 75,398 | 30,059 | 21,857 | scroll 1.8–3.2k / 1.3–2.2k |

固定轨迹总输入投影（`project.out`；把每次 CLI 增量按该次 E/原始比例缩放，其余不变；指引系数为 0.5 时表示首屏重组后少读一半）：

| 臂 | 实际 | E1 | E2 | E2 + 指引×0.5 | 相对旧基线（E2） |
|---|---:|---:|---:|---:|---:|
| Contacts 原型路线 | 629,770 | 542,070 | 542,070 | 468,841 | 0.67× |
| Contacts 原版路线 | 1,395,045 | 887,419 | 876,965 | 763,859 | 1.08× |
| Chrome 原型路线 | 1,821,534 | 1,181,832 | 1,066,823 | 914,690 | 1.02× |
| Chrome 原版路线 | 2,326,407 | 1,995,127 | 1,972,356 | 1,782,889 | 1.89×（主要是读详细保存体，见下） |

Chrome 原版路线降得少，是因为它的大头是读详细保存体（690k）。如果保存体本身也是同一份紧凑 View（实施方 C 的“统一一份 View”），这部分也会按约 4–5 倍缩小。这一点支持实施方“打印、保存、diff 统一一份 View”的方向。【离线重算＋推断】

E 的已知风险：从文本反解析的离线变换不一定等价于在 renderer 里实现；E2 的目标收窄只是近似；隐含动作被隐藏后，执行者是否会漏用某个标准动作没有证据。（需模型对照）

### 2.3 公开指引（问题 3）

【原件核对：各臂 `public-skill/`、rollout 工具调用；o200k 计】

| 臂 | 读入 | 指引交付 tok | 指引携带 |
|---|---|---:|---:|
| Contacts 基线 | SKILL + reference（分两次，第二次是切片） | 26.8k | 342k |
| Contacts 原版 | SKILL、launcher + actions、input | 11.0k | 226k |
| Contacts 原型 | SKILL；input + actions 合读 | 10.8k | 146k |
| Chrome 原版 | SKILL、launcher + reference、snapshot + actions + observations 合读（被截断）、actions 重读、observations 前 155 行 | 24.9k | 647k |
| Chrome 原型 | SKILL、observations、actions | 16.2k | 304k |

- 当前 12 份 topic 共 30,753 token（原型 31,000）；SKILL 1,627，原型 1,815；actions 5,229、observations 6,133、reference 3,075、input 1,789。
- **actions.md 在四臂中都被整读**，所以“SKILL 自足、按需读 topic”在这两个任务里没有成立。原型 SKILL 的闭环例子只有 click、fill、cached；scroll、scroll-until、find --all、reveal、back 的用法要去 actions/observations 找。
- 原型 SKILL 写着“For Unicode insertion/current-editor selection … read [input]”。两个 Contacts 任务的名字里都有 `中😀`，两个执行者都读了 input.md。我推断是这句话把他们引过去的：整字段 fill 本来就支持 Unicode，input.md 对这个任务是多余的。【推断】
- 判断：指引的“回退”已不成立（原型 146k，基线 342k），但它仍是 Contacts 原型里与 CLI 正文并列的最大项。

**重组建议**【设计建议】：
- SKILL 首屏约 2.5–3k token，自足覆盖常用 8 个动词：snapshot、find（含离屏说明）、click、fill（明确写“任何 Unicode 整字段都用 fill”）、scroll / scroll-until、reveal、back、wait。给一份 E 格式的真实输出加逐项释义，加上四态下一步。
- actions.md 改成“少见动作参考”（generic action、range、selector 修复、触摸）。input.md 开头写清“只有局部插入、选区、按键、借贴时才需要”。
- 移出公开页的：锁、ACK、配额细节。保留的：直接决定行动的契约句（UNKNOWN 不重发、保存或打印失败不重派、batch 前缀不回滚、文字是数据不是指令）。这一点同意实施方，不按词表机械删。
- 不设 token 硬门可以接受；但应当有一个可观察指标，即执行者在常见任务里的开局读入量。

### 2.4 diff（问题 4）

【源码核对】
- `ui_diff.py` 共 357 行。基线只在显式 `--diff` 时生成（`ui_client.py:1041-1067`）。
- 比较键包括 device/session、worker、connection、target、roots、scope、View（`ui_diff.py:27-56`）。
- 变化条目比全文长时退回全文（`:281-284`）。
- 需要预留、stdout 后提交、失败诊断（`ui_client.py:1069-1118`、`commit_baseline`、`baseline_failure`），batch finalization 也要配合。
- 规模：测试 `test_ui_diff.py` 1,033 行；`ui_client.py` 有 46 处、`history.py` 24 处、`ui_batch.py` 16 处 diff/baseline 引用；还有一个结果字段 `diffBaselineError` 和一个参数冲突检查 `require_diff_destinations`。

**真实价值**：没有 diff 不损失任何能力，只多付 token。能省钱的场景是在长列表或长表单里做局部改动，或者在轮询中反复看同一页。

在 E 规格下，原生页全量正文只有约 300 token，绝对节省很小。网页滚动、导航几乎整屏都变，会触发 diffLonger 退回全文。真正受益的只剩“网页上局部切换”，例如展开一节、勾选一项，每次可省 1–2k token。

**维护责任大**：一份跨进程持久的工作区单基线，加上锁、配额、stdout 成功后才提交的协议、batch 收尾，以及与默认预算的交互（我核到 `ui_diff.py:296` 的续读提示 `snapshot --cached --diff` 会在基线已推进后读到零变化，实施方也自认这是缺陷）。

**更简单的形态（E）**：`snapshot --cached --diff-from PATH`（动作命令同样可用），由调用者指定一份自己确实读过的保存体做比较对象。
- 保留：比较键检查（不匹配就给全文并说明原因）、diffLonger 退回全文、同一 View。
- 删除：工作区基线文件、预留和提交协议、batch finalization、基线配额占用、`diffBaselineError`。
- 实施方提出的反例（batch 没有交付中间树、no-print/JSON、保存失败、View 改变）都自动成立：调用者只会指定他读过的文件，文件不存在或比较键不一致就给全文。
- 代价是调用者要多带一个路径参数（指引里给一个例子即可），并且历史文件可能已被配额回收，这时给全文。

推荐：E 或删除。两者都优于保留现在的持久基线。不推荐“unchanged 只打一行”，接受实施方的反驳。【设计建议】

### 2.5 zoom（问题 5）

**谁对**：实施方对。【原件核对】
- `T/stages/36-native-zoom/runtime-mumu/images/settings-full.jpg` 是 720×1280 JPEG；`full-source-roi.png`、`settings-marked.png` 是 1080×1920；`roi-a.png` 是 668×75（原生分辨率裁片）。
- `measured.json` 含 `sourceWidth=1080`、`sourceHeight=1920`。
- 我方“全部证据 720×1280、JPEG 没有缩放”是事实错误。
- 但我亲看了 `settings-full.jpg` 与 `roi-b.png`：完整 JPEG 已清楚可读，裁片在这个页面没有带来阅读增益。实施方自己也承认“新增细字收益未证”。

**同源的价值**：同一帧内多次 ROI、瞬态内容（toast、动画中途）、被 marks 遮住的原像素、与树和 REF 同一时点。代价是：每个 lease 驻留一帧原生位图（1080×1920 ARGB 约 8.3MB；720×1280 时 malloc 峰值实测 7.49MB）；`ImageState` 同时持有源图和坐标输入资格，失效规则耦合（实施方 §5.2 自述，我未逐行核 Kotlin）；硬件回读和派生图的释放路径也要维护。

**替代**：`screenshot --region x,y,w,h|REF --png`，即新采一帧、在设备端按原生分辨率裁剪后交付。没有驻留，没有资格耦合。损失是瞬态和历史同帧，以及树与图的严格同时点：新图有自己的时点，树要按需重采。另外，一张全幅 1080×1920 PNG 交给模型时多半会被服务端缩小，所以“ROI 原生裁剪”这个能力本身比“同源”更有用。【推断：基于常见视觉模型的长边上限】

推荐：改成“新截图带 ROI”（C）。如果用户看重瞬态或同帧取证，就保留 zoom，但应把源图寿命与坐标资格拆开（实施方说这是额外长期责任，同意）。我不推荐“不改并保留现状”，因为收益未证而耦合已证。【设计建议】

### 2.6 其余机制逐项简评（问题 6）

| 机制 | 简评 | 建议 |
|---|---|---|
| batch（静态预检、失败停前缀） | 复用真实 parser 和 Result，前缀语义清楚，几乎没有新的维护面；8 臂零自然使用，不构成删除理由。早期自动文件可被回收（`batch-retention/README.md`），契约已写明。 | 保留。指引里只给一句适用条件。 |
| JPEG 默认与交付坐标 | 坐标帧身份清楚，字节小；模型图片 token 与字节无关（同意实施方）。 | 保留。 |
| REF marks | 显式开启，默认零成本。 | 保留。 |
| sample/publish 分离、目录/树/图独立交付 | 内部机制，调用者不感知；树失败仍能拿到目录或图，属于真实价值。 | 保留；不进公开指引。 |
| 64/192/32KiB 块 | 内部参数，无调用者负担。 | 保留。 |
| help/export 同源 | 单一来源正确。 | 保留；改内容不改机制。 |
| D4（未请求图像占位、保存失败影响打印） | 原件确认 `artifact {"kind":"image","state":"not requested"}` 每次都出现（Contacts 原版 #9 等）。打印不应依赖保存成功。实施方同意修。 | 修。并入 E 的“只打印非默认事实”。 |
| 默认 find 只搜视口 | 见 §5 确定缺陷。 | 修：find 默认搜全部已采节点，离屏结果标 `offscreen` 并给 REF（可直接 reveal），或至少报 `offscreenMatches=N` 和 `find --all`。 |
| 双正文（研究原型） | 双方都不推荐长期保留。 | 统一 View。 |

可以合并或删除的：diff 的持久基线（改为 E 或删除）、zoom 的同源驻留（改为 ROI 新截图）、默认正文里的 numeric 标准 ID、每次都打印的 events 和窗口条带。

### 2.7 效率回退是否已被本轮方案解决（问题 8）

按实施方题 3 的 C（保 numeric 标准 ID、保完整窗口目录、保 events 摘要、仅 host 收窄 interactive、统一 View），再加指引调整：

- **Contacts 类原生页**：原型（不保 ID）每次约 1,000 token；保 ID 后会更大（实施方承认没有测）。我用原型 stdout 粗估，保 ID 每行约多 15–30 token，每页多 300–600 token。预期每次正文约 1.3–2× 基线；总输入取决于路线，大约 0.8–1.3× 基线（推断）。
- **Chrome 类网页**：C 不改变默认正文的目标集，“仅收窄 interactive”只影响显式 `--interactive`。实施方自己写了“普通未启 interactive 正文不会因此自动更短”。所以默认 scroll、back、click 仍是每次 4–8k token 的视口正文，预期仍是基线的约 1.5–1.8×（本轮原型臂实测 1.75×），find 假阴性也还在。
- **结论：C 不足以解决 Chrome 类网页的效率回退。** 剩余差额依次来自：网页容器目标（CONTEXT_CLICK）、隐含动作和 numeric ID、头行/窗口/事件每次 250–350 token、find 假阴性带来的多余滚动，以及指引首屏不自足。
- E2 加指引首屏重组的固定轨迹投影是 Contacts 0.6–1.0×、Chrome 约 0.9–1.0×（Chrome 基线不含网页语义）。要达到这个水平还需要：E 的动作和头部规则；网页目标在默认 View 中收窄（只影响 host 阅读筛选，设备 publicTarget 和 REF 分配不动，见 §6 选项）；修 find；SKILL 首屏覆盖常用动词；保存体与打印体统一用紧凑 View。

## 3. 对照实施方

**同意**：
- 打印、保存、diff 统一一份 View，不保留双正文。
- 保留 ring、raw、cached 冻结；failed/UNKNOWN 结果行原样。
- “有 REF 即可点”不成立。
- unchanged 省正文不是等价替代。
- zoom 分辨率事实。
- batch、JPEG、marks、块参数、help 同源保留。
- D4 修复。
- `ui_diff.py:296` 提示缺陷。
- Chrome 对照不能判格式因果（但见下：主因是可指认的 find 缺陷和默认整页打印，不是“无法判断”）。

**不同意或补充**：

1. “Chrome 指标交叉、不能归因”：过于保守。逐调用核对可以指认出一个确定的产品缺陷（find 假阴性加误导性的续读提示），以及原型保留的 42% 动作噪声。非缓存输入的差额主要是缓存未命中噪声，不应作为“原型更贵”的证据。【原件核对＋离线重算】
2. “标准 numeric ID 在最终正文中保留；不新增动作名解析”：不同意，理由见 §2.2。
3. “保完整窗口目录，不折叠系统栏”：不同意保持现状。几何规则不需要猜标题，原生页每次可省约 70–100 token。
4. “events 默认摘要”：同意摘要；不同意在无文字、无缺口时仍打印一行。
5. “题 3 C 仅收窄 host interactive”：对网页默认正文无效。应增加选项 C′：host 默认 View 也按“实际可操作”收窄 REF 显示，设备事实和 publicTarget 不动，被收起的节点仍可用 `--all` 拿到 REF。这样能拿到 D 的大部分阅读收益，又不动设备的命名、selector 和 marks。
6. “支持示例先行、不设 token 门”：同意不设硬门；但实施方的原型 SKILL 没有达到“首屏自足”，四臂都整读了 actions.md。
7. diff“保留”：不同意保留持久基线，推荐 E（显式比较文件）或删除。
8. zoom“保已选 C”：倾向改成 ROI 新截图，见 §2.5。

**实施方对上轮外评“§4、§11–12 证据修正”表逐行判定**：

| 行 | 判定 | 依据 |
|---|---|---|
| 旧四臂 1.80/2.35 倍、动作数组为大项 | 成立（实施方同意我方原结论） | 我复算 usage 一致。 |
| “多出时间全部模型侧、CLI 不变差” | 成立 | 旧 Chrome child 24.56→27.32s（+2.76s），我方原句过强。 |
| “恒差 10” | 部分成立 | `agents2/work10/carry.json`：Contacts 两臂 42/42 段恒差 10；chrome-2 有 7/26 段不是 10（6 段 +1115、1 段 −445），chrome-1c 有 3/28 段为 +1115。“恒差 10”不能概括全部，但分摊方法本身闭合，结论不受影响。 |
| 11,080 的等价限制、Title 反例 | 成立 | 我核到 Title 位于 19,969 字符处；补充：反例源于我方规则让 `--all` 保留完整动作并对其施加上限，规则改了反例就消失（见 §2.2）。“26 份中 12 份合成”我未核。 |
| “diff 每次持久化/跨 session” | 部分成立 | 我方原文是“每次 `--diff` 要持久化一份 MB 级整树”，限定在 `--diff`，没有说每次观察都持久化。“MB 级”只对大树成立（实施方 4.9MB 为特定基线）；多 session 互相重置基线，实施方也承认（contextChanged）。 |
| zoom “仅 720×1280” | 成立（我方事实错误） | 见 §2.5。 |
| “1884 passed” | 成立但非更正 | 我方原文本来就写“只在 Windows 成立”（主报告 :866、:1047）。实施方补充“我方拆出只跑了 host 检查、不含 Kotlin 构建”，这一点成立。 |
| dense 8,377,092/127 REF 来源 | 未核 | 不在本专题范围内。 |
| 原 22 案关系 | 成立 | 我方原文已注明逐案表不在包内。 |

## 4. 对上轮外评（我方）的更正

1. zoom“全部证据 720×1280、JPEG 没有缩放”是错的；MuMu 原生 1080×1920，JPEG 720×1280。
2. §2.2 规则 1（`--all` 保留完整动作）与规则 8（默认上限 8–12k）组合，会截掉真实下一步需要的 Title REF；11,080/1.13× 中有 8.4k 依赖这一截断。应改为：`--all` 也用紧凑动作，默认上限不作用于显式 `--all`。
3. “有 REF 即可点”不成立，应保留短名 `click` 或用其它可区分标记。
4. “多出的时间全部在模型侧，CLI 没有变差”过强（Chrome child +2.76s）。
5. “恒差 10”应改为“绝大多数段恒差 10，Chrome 有约 10% 段例外”。
6. “事件 ring 缩为只记带文字条目、cached 不重放”过度：属于删设备事实，换不来阅读收益，应只改呈现。
7. diff：“unchanged 只打一行”不等价于读者已有正文，撤回这一替代；删除理由应改为能力和维护的权衡，并给出无状态 E 作为替代。
8. 我方上轮没有发现“默认 find 只搜视口”的缺陷；Chrome 旧候选臂是否也受它影响，我未核。

## 5. 确定缺陷（只列我核过的）

1. **默认 find 对离屏节点假阴性，续读提示误导。**
   - 位置：`ui_view.py:236-237`（非 `--all` 丢弃 `visible=False`）、`:269-284`（find 只在投影后的元素上匹配）、`:533-541`（find 省略的 read 固定为 `snapshot --cached`）、`observation_nodes:600-623`。
   - 可观察后果：`R/chrome-prototype` 在 4,226 节点的列表页上执行 `find 'Addis Ababa' --context 2`，得到 `matches=0`，`omission nodes=84 read="snapshot --cached"`；执行者随后滚动 4 次（每次 4.1–5.7k token 正文）才找到目标。原版执行者用 `find --all` 一次就命中离屏行。
   - 修法：find 默认在全部已采节点上搜索，并标出 offscreen；或者报 `offscreenMatches` 并给出 `find --all` 提示。
2. **`--diff` 预算省略提示无法续读**：`ui_diff.py:290-298` 给出 `"read":"snapshot --cached --diff"`，但基线已推进，重读只得到零变化。（实施方已指认，我核到代码。）
3. **D4 占位行**：每次默认正文都有 `artifact {"kind":"image","state":"not requested"}`（`R/contacts-original` 原件中出现）。保存失败时不打印内存正文这一点，我只看到实施方原型 README 的描述，没有核源码。

研究原型的缺口（不是产品缺陷）：`ui_text.actions()` 把 `action-HEX` 形式的标准动作保留为 `{"id":N}`，导致 Chrome 原型 stdout 42% 仍是动作数组（`actfrac.py`）。

## 6. 需要用户裁决的事项

**6.1 默认阅读规格**

| 候选 | 能力得失 | 调用者负担 | 维护 | 预期成本（固定轨迹投影） |
|---|---|---|---|---|
| 实施方 C（紧凑，保标准 ID，保窗口和事件摘要，仅收窄 `--interactive`） | 不丢任何显示事实 | 每页读噪声；网页默认正文仍大 | 低 | Contacts 约 0.8–1.3×，Chrome 约 1.5–1.8×（推断） |
| E1（§2.2 规格，REF 全保） | 默认看不到标准 numeric ID、条带窗、无缺口事件头（可用 `--all`/JSON 拿到） | 少见动作要 `--all` 或 `--action NAME` | 低（纯 host 呈现） | Contacts 0.67–1.09×，Chrome 1.14× |
| E2（E1 + host 默认 View 收窄网页容器 REF，设备不动） | 无名无值、只有隐含动作的容器默认不给 REF（`--all` 仍有） | 极少数用 CONTEXT_CLICK 的任务要 `--all` | 低到中（host 规则 + 测试） | Contacts 同上，Chrome 1.02× |
| D（两端改 target） | 改变命名、selector、marks | 同 E2 | 高 | 与 E2 相近 |

推荐 **E2**。未知：隐藏隐含动作以后执行者是否会出错（需模型对照）。

**6.2 diff**：保留持久基线（现状），改为 E `--diff-from PATH`，或删除。推荐 E 或删除，看用户是否看重“长页局部变化”。删除时没有能力损失，只在少数场景多付 token。

**6.3 zoom**：保留同源（驻留加耦合），改为 `screenshot --region` 新截图，或删除。推荐改为 ROI 新截图。用户如果看重瞬态或同帧取证，就保留并拆开资格。

**6.4 动作调用地址**：保留默认 ID（实施方），或者用 `action REF --action NAME`，按已交付的 actions[] 唯一匹配。推荐后者。

**6.5 指引**：SKILL 首屏是否吸收常用动词并降级 actions.md。推荐吸收。

## 7. 会改变取舍的验证问题

1. **E 规格下执行者是否因看不到隐含动作或标准 ID 出错**：用同一后端、两种呈现（C 对 E2），跑 Contacts 和 Chrome 各一对或几对。如果 E2 没有出现错误命令，也没有因找 ID 读 `--all`，就选 E2；如果出现因缺 ID 的 `--all` 重读或错误命令，就在 E2 上恢复“只对非隐含标准动作给 ID”。
2. **find 修复后 Chrome 路线**：如果原型路线的 4 次滚动变成 find 加 reveal，Chrome 差额主要来自正文大小，E2 的价值确定；如果执行者仍然逐屏滚动，就要在 SKILL 首屏强调 find 用法。
3. **指引首屏重组后是否仍整读 actions.md**：如果不再读，携带量约降一半（E2 加指引×0.5 投影）；如果仍读，说明需要把常用动词写进 SKILL 并精简 actions 开头。
4. **diff 的真实需求**：如果用户有长页局部编辑或轮询任务，在一个真实任务上比较“全量 E 正文”与“E diff-from”，节省明显就保留 E，否则删除。
5. **高分辨率细字**：在一张默认 JPEG 读不清的真实页面上比较“新截图 ROI”与“同源 zoom”。如果两者读出相同、且没有瞬态需求，选 ROI；如果瞬态内容只有同源能拿到，就保留 zoom 并拆开资格。

## 8. 未核实与局限

- 所有投影都是固定轨迹假设：删掉正文后，执行者的路线、回合数和缓存命中都可能变。E 规格是从文本反解析的离线变换，不是 renderer 实现；E2 的目标收窄只是近似（用“只剩隐含动作”判断），没有核设备 publicTarget 的连带影响。
- 新旧轮次的 Chrome 设备不同（MuMu 1080×1920 对旧 API34 AVD），旧基线没有网页语义；跨轮只能比量级。wall 时间跨轮不可比（新 Chrome 两臂 600s 以上，环境不同）。
- 初始 prompt 是加密的，无法核任务文字是否逐字相同。
- 未核：保存失败时不打印的源码路径；zoom 的 `ImageState` 耦合（只依据实施方 §5.2）；26 份 raw 中 12 份为合成；旧 Chrome 候选臂是否受 find 缺陷影响；schema 原型数字。
- 模型图片缩放上限是一般性推断，没有在本包中实测。
- 没有设备，也没有运行任何包内脚本；所有数字来自我自写脚本对包内原件的只读解析。
