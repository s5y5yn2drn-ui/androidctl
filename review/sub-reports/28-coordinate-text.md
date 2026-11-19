# 专题 28 核查：坐标接收者的几何、文本输入通道与选区

核查者：独立只读核查。所有运行事实只来自 Android Studio API34 模拟器与 MuMu（API35）原件，没有真机；本人没有任何设备。
本文用到的路径缩写：`ROOT=…/scratchpad/pkg6/root`，`T=ROOT/ultragoal/2026-11-11-next-improve`，`P=T/prototypes/route-review`，`R=T/research/route-review-3f45af6`，`SRC=…/scratchpad/review-copy`（候选 3f45af6），`W=…/scratchpad/agents6/28`（本人工作目录，含下载的 AOSP 文件 `W/aosp/` 和重算脚本 `W/geom_check.py`，输出 `W/geom_check.out`）。

术语：**AX**＝无障碍节点动作（如 ACTION_SET_SELECTION）；**IC**＝通过输入法连接 InputConnection 直接提交或选区（候选里的 `editor commit/select`，计划并入 `type --via editor`）；**offset**＝IC `getSurroundingText` 返回的“这段文本在整个字段中的起点”，-1 表示未知；**VD**＝后台虚拟屏；**expect**＝坐标命令的 `--expect 包名`，显式指定接收 App。

---

## 1. 结论摘要

- **第 2 题（坐标几何）：这题不该问用户。** 平台给输入系统的显示变换只由 90° 倍数的旋转、轴向缩放和平移组成，永远“保矩形”【源码 A15 `ProjectionSpace.h:38-78`、`Output.cpp:170-193`、`DisplayDevice.cpp:134-155`】。B（遇到不保矩形就拒绝）与 C（遇到不保矩形就用整体外接矩形）只在走不到的输入上有区别。A（只认恒等变换）已经有真实代价：横屏 Chrome 被拒【原件 P5】。直接定为“按平台逐矩形语义支持旋转和缩放”。不保矩形这条走不到的分支，用一行“预测不可用，请用 --expect”拒绝即可。这属于实施细节。
- 实施方所说的 C 确实照抄平台这一段：逐矩形变换，角点取 floor(v+0.5)，左上闭右下开，用 floor 后的点做命中测试【源码 A14/15/16 `InputDispatcher.cpp` windowAcceptsTouchAt；`Transform.cpp:301-319`】。三个版本这段逻辑逐字相同。
- “只把点逆变换回去再测”这个更简单的做法**不等价**。重算显示，旋转后整数坐标点落在边上、或有非整数缩放时，结果差 1 像素【重算 `W/geom_check.py`】。这个偏差不影响安全，因为最终 UID 由平台核对，最坏是发送前被平台拒绝；但它会让边缘像素的预测和平台判法不同。逐矩形变换只需大约 30 行，照平台做即可。
- 坐标原型的说明是诚实的：P3（输入法打出字母 A）和 P5（横屏 Chrome）都是“独立探针 → 离线预测 → 原来的显式 expect 输入”的组合，主稿第 31 行写明了“不是新默认路径”。回调耗时 5/8/12/15ms（API34）和 31ms（MuMu）都能在原件里找到【原件】。原型对三个已知风险都做了处理：回调里只做一次原子赋值、不消费注册时返回的缓存、在 finally 里注销且不把无返回值当成成功。
- **第 3 题（IC 独立选区）：改选 A，删除 IC 选区，只保留 AX 选区。** 新证据都支持删除，没有一条支持保留：
  - Compose 文本框的 offset 永远是 -1【源码 AOSP `InputConnection.java:379-400` 的默认实现；androidx 的两个 Compose IC 都没有覆写】，所以 IC 选区在 Compose 上一律不会发出【源码 SRC `CurrentEditorInput.kt:99`】，而 AX 选区在 Bitwarden 上成功了【原件】。
  - regex101 证明调用者在快照里看到的 AX 偏移（48..51）由浏览器正确换成了模型偏移（46..49）。反过来，调用者看不到 IC 自己的偏移域；拿 AX 数值去调 IC 选区，会被拒或静默选错位置【推断，基于原件中 51 字与 49 字的差异】。
  - 主稿写的“已证 HTML 直接修改”是 IC **commit** 的能力，不是 IC **select** 的能力。
  - IC 选区从未进入任何发布标签，现在删除没有兼容成本。
- **第 4 题（AX 选区何时读回确认）：选 C，属于实施细节，不该问用户。** 证据是：网页上“动作后立即读回”3 次全部失败；把同一次读回挪到已有的动作后观察之后，3 次全部成功（Wikipedia 选区、Wikipedia 城市名修正 batch、regex101）【原件】。这样不增加 RPC，只调整顺序。不选它的理由只有一条：它不是同步屏障。但这一点在 C 自带的“与动作前相同则判 UNKNOWN”规则下已经得到诚实处理。
- **第 5 题（IC commit 能否凭局部匹配得 DONE）：需要用户定，但题目要改写成“Compose 上的 `--via editor` 能否报 DONE”。** 按现在的绝对锚点门，所有 Compose 文本框一律 UNKNOWN，batch 一律停【源码】。我同意推荐 B（以光标为锚的精确局部匹配），并给出了具体匹配规则和残余风险（§2.4）。
- **剪贴板（建议提交 11）：没有新证据。** VD 上的 paste 主稿如实标为未测，上一版结论不变。只修正一处：VD 上 paste 失败不等于“只能整字段 fill”，实施方的这一修正成立。
- 评审方上一版定稿有三处被推翻：
  1. “AX 选区偏移以编辑器真实文本为准”是错的，实际以 AX 文本为准。
  2. “注册时的初始数组为空”是错的，注册返回的是进程缓存。
  3. “首版只支持恒等变换”已被 P5 的真实代价推翻。

---

## 2. 逐题分析

### 2.1 第 2 题：默认坐标接收者的几何范围

**实施方主张**：推荐 C，即“按平台完整仿射 Region 语义”。B 已有转换主体，C 只多一个外接矩形分支，比再加一条拒绝规则更贴合平台。“非保矩形业务仍未验”（主稿 §2，第 125–134 行）。

**我核到的事实**

1. **平台命中测试**【源码，下载到 `W/aosp/`，标签 android-14.0.0_r1 / 15.0.0_r1 / 16.0.0_r1】
   - `windowAcceptsTouchAt` 在 A14 第 516 行、A15 第 576 行、A16 第 582 行，函数体逐字相同。做法是：先跳过不可见窗口，再跳过不可触摸窗口（触控笔拦截除外）；用 `displayTransform.transform(touchableRegion)` 把原始显示坐标下的区域换到逻辑坐标；点 `p = displayTransform.transform(x, y)`；最后测 `contains(floor(p.x), floor(p.y))`。注释写明，这样做是为了让“右、下边界不算在内”在旋转后依然成立。
   - `findTouchedWindowAtLocked`（A14 第 1236 行；A16 改名为 `DispatcherWindowInfo::findTouchedWindowAt`，第 1452 行）按从前到后的顺序遍历窗口，跳过 spy 窗口，取第一个命中的窗口。
2. **注入事件的坐标换算**【源码】`transformMotionEntryForInjectionLocked`（A14 第 4807 行、A15 第 5117 行、A16 第 5098 行）计算 `transformToDisplay = displayTransform.inverse() * injectedTransform`。也就是说，注入的逻辑点先经过 T⁻¹ 进入原始显示空间，命中测试时再乘 T 回到逻辑空间。
3. **Region 变换**【源码 `W/aosp/android-14.0.0_r1-Transform.cpp:301-319`；15 与 14 字节相同；16 只改了注释和 `set()` 的参数名】类型高于平移时：如果 `preserveRects()`，就逐个矩形取四角 min/max，再 floor(v+0.5) 后取并集；否则对整个区域外接矩形做变换。纯平移时先取整再平移。`EPSILON=0.0f`，第 44 行。
4. **结论：实施方的 C 就是照抄这一段语义。** 实施方的公式说明（`R/r2-coordinate.md` §4、`P/r2-coordinate/runtime-p5/result.md` 末节）与源码一致，包括“图上取得的点已经是逻辑坐标，不能再乘 T”，以及 T(T⁻¹p) 这一往返。

**显示变换会不会不保矩形？**【源码，A15，下载到 `W/aosp/`】
- `DisplayDevice::getFrontEndInfo`（`P/r2-coordinate/sources/a15-DisplayDevice.cpp:134-155`）把 `undoPhysicalOrientation * getTransform()` 的逆交给输入系统。`undoPhysicalOrientation` 由 `toRotationFlags(ui::Rotation)` 构成。
- `getTransform()` 来自 `ProjectionSpace::getTransform`（`android-15.0.0_r1-ProjectionSpace.h:38-78`），只由三部分相乘：源平移、轴向缩放 `scale.set(scaleX,0,0,scaleY)`、`toRotationFlags(rotation)` 的旋转。`Output::setProjection`（`android-15.0.0_r1-Output.cpp:170-193`）中的方向同样是 `ui::Rotation` 枚举，只有 0/90/180/270 四种。
- 这些分量的乘积在 b、c 或 a、d 位置上是精确的 0（0 乘任何数仍为 0），所以 `type()` 不会得到 ROT_INVALID，`preserveRects()` 恒为真。
- 因此在 A14–A16 的 SurfaceFlinger 上，**显示变换不可能不保矩形**。我没有逐一核 OEM 改动，这一点是【推断】，但要改出斜角显示投影，就得改掉 `ui::Rotation` 这个枚举。

**判断**
- B 和 C 的差别只存在于走不到的输入上。A 有真实代价：P5 原件显示，矩阵为 `[0,1,0,-1,0,720,0,0,1]` 时，冻结的预测器给出 `"decision":"refused","reason":"This probe supports only identity display transform"`，而同一张新图上显式 `--expect com.chrome.beta` 的滑动能正常滚动【原件 `P/r2-coordinate/runtime-p5/commands/p5-offline-prediction/stdout.bin`、`p5-actual-callback/stdout.bin`、`result.md`；样本：1 个角度、1 次回调、1 次操作】。
- D（全部要求显式 expect）属于“要不要默认预测”这个上层选择，用户此前已经倾向于预测，不在几何题范围内。
- 所以几何本身不需要用户裁决：**照平台做逐矩形变换（含缩放）**。不保矩形的分支选“拒绝并提示用 --expect”或“外接矩形”都行。我建议拒绝：它走不到，也无法测试，拒绝最省事，而且不会给出错误预测。
- 主稿第 130 行说 B 有“缩放/翻转业务未验”的代价。但缩放和翻转同样是保矩形变换，B 与 C 在这里完全相同，这条不能算作两者的差别。

**更简单的等价做法？**
我写了 `W/geom_check.py`，用 float32 并按 Transform.cpp 的运算顺序模拟，在 8 种变换下各取 72,000 个**刻意贴近边界**的点，与平台判法比较【重算，输出见 `W/geom_check.out`】：

| 做法 | 结果 |
| --- | --- |
| 只把点逆变换回原始坐标，再在原始区域里测 floor | 恒等变换下 0 处不一致。90°/180°/270° 旋转下，整数坐标点有 6110–9798 处不一致（均在旋转后的右、下边界上）。1.5 倍和 2/3 倍缩放下也有不一致。 |
| 逐矩形变换，但省掉 T(T⁻¹p) 这次浮点往返 | 纯旋转和 1.5 倍缩放下几乎 0 处不一致。2/3 倍缩放下，整数点有 3629–6471 处不一致，原因是浮点往返不精确。 |

结论：要与平台完全一致，必须照抄“逐矩形变换并按 floor(v+0.5) 取整”，并保留一次 T⁻¹ 再 T 的 float32 往返。偏差只出现在区域边缘一像素，也不影响安全：预测错了，平台会在首个 DOWN 就以 UID 不符拒绝，结果是“未发出”。照平台做的代价大约 30 行，不值得为了省事而换来边缘行为不一致。

**P2 原件与 WindowInfosListener 的已知风险**【原件＋源码】

| 风险 | 原型（`P/r2-coordinate/source.patch` 中的 `WorkerWindowProbe.kt`）里的处理 |
| --- | --- |
| 确认（ack）背压：本进程回调返回后才发 ack，慢回调会拖住全系统窗口信息更新 | 回调只计数并对 `AtomicReference` 做一次比较交换，扫描和序列化都在关闭消费之后进行。源码 `a14-WindowInfosListenerReporter.cpp` 中 `onWindowInfosChanged` 确实先同步调用本地所有监听器，再 ack。处理到位。 |
| 初始数组 | 注册返回的是进程缓存（同一文件的 `addWindowInfosListener` 返回 `mLastWindowInfos`），不一定为空。上一版定稿“初始数组是空的”是错的。原型从不消费初始值，只认真实回调。处理到位。 |
| 注册/注销没有返回值 | 只有远端 `remove` 返回 OK 时才从本地集合擦除，Java 一侧拿不到这个状态。原型在 finally 里注销一次，消费门之外另计回调次数，不把 void 当作成功。处理诚实。 |
| 静止页面会不会来新回调 | 只有当本进程原本没有本地监听器时，才会向远端 add 并触发一次输入信息更新。**产品必须保证 worker 进程里不再有别的常驻 WindowInfosListener**，否则每条命令新注册后，在静止页面上可能等不到回调，只能得到 noSample。这一点应写成实现约束【源码推断】。 |
| 回调耗时 | API34 上 P1/P2 为 5/8ms、P3 为 12ms、P5 为 15ms；MuMu API35 为 31ms（从注册尝试到收到回调）。我从 `p3/p5-actual-callback/stdout.bin` 的阶段时间戳独立重算过，P1/P2 和 MuMu 的数字引用自 `result.md`。样本共 5 次。 |

**主稿是否如实说明了这是组合验证**：是。P3 的接收方 UID 10109（LatinIME）出自离线预测 `p3-offline-prediction/stdout.bin`，真实输入是另一条显式 `tap --image --expect`（`p3-public-tap-once`，`source:"explicit"`）。主稿第 31 行和 `r2-coordinate.md` §9–11 都写明“不是新默认同请求/全手势/VD”。没有把组合包装成新默认路径的问题。

### 2.2 第 3 题：是否删除 IC 独立选区

**实施方主张**：B，两个都保留。理由有三：保已证 HTML 直接修改和当前 editor 定位；“没有证据支持为统一入口删它”；后删会改变公开命令。

**四项原件逐一核对**

| 实施方的事实 | 核查 |
| --- | --- |
| (a) Wikipedia 输入框上 AX 选区返回成功，立即读回却未变，之后才生效 | 【原件】`P/r2-text/runtime-api34/commands/wikipedia-caret-batch/stdout.bin`：`selectionAttempted:true`、平台 `accepted/callback:true`，但 after 仍为 13/13，`verification:"differs"`，batch 失败，后续 commit 一次 RPC 都没有发。换成把读回挪到动作后观察之后的版本（after-observe APK `1fced…`）后，Wikipedia 选区 6..9 得 DONE，城市名修正 batch（select 8..9 再 commit）两步都 DONE（`T/reviews/route-r2-text-after-runtime.md` 汇总；四个时间点表明观察耗时 494/994ms，读回耗时约 60ms）。属实。 |
| (b) Compose（Bitwarden）上 AX 选区匹配，IC commit 因 offset=-1 得 UNKNOWN | 【原件】`bitwarden-caret-batch/stdout.bin`：AX 选区 8/8 得 DONE；第 3 步原因是 `"Surrounding text has no interpretable absolute selection"`。【源码】这不是 Bitwarden 特有的问题：AOSP `InputConnection.getSurroundingText` 默认实现返回 `new SurroundingText(…, -1)`（`W/aosp/android-14.0.0_r1-InputConnection.java:379-400`）。androidx-main 的 `RecordingInputConnection`（ui 与 foundation 两份）和 `StatelessInputConnection` 都直接实现 InputConnection，都没有覆写 getSurroundingText（`W/aosp/androidx-main-*.kt`，grep 无结果）。View 体系的 `BaseInputConnection` 会返回真实起点（同目录 `BaseInputConnection.java:638-676`）。所以**所有标准 Compose 文本框的 offset 都是 -1**。注意 androidx-main 不等于 Bitwarden 实际使用的版本，这一点属于【推断】，但与原件一致。 |
| (c) regex101：AX 文本 51 字（含两个 ↵），原生文本 49 字；AX 选 48..51 对应 IC 的 46..49 | 【原件】`P/exploration-20261119/api34-ax-ic/commands/031-…/stdout.bin`：`select: done`，矩阵匹配。`032-…/stdout.bin`：IC 的 before 为 `length 49, absoluteSelection 46..49`，提交后末尾 503 变为 502、`caret 49`，结果 DONE。属实。样本 1 次、1 个站点、Chrome Beta158。 |
| (d) 同一多行框里 AX 写入 UNKNOWN、IC 写入 DONE | 【原件】`api34-editor-pair/commands/029-a-ax-fill-once`：UNKNOWN，原因是 `"Local UI read budget exhausted"`。`040-b-ic-commit-once`：DONE，after 为完整 49 字。属实，但这是 **AX fill 与 IC commit 的比较，与选区无关**。它支持保留 IC commit，不支持保留 IC select。另外 A 臂的 UNKNOWN 是读回预算耗尽造成的，不是值不匹配。 |

**AX 选区与 IC 选区是否等价**：不等价，但区别对 IC 选区不利。

| 维度 | AX 选区（`select REF START END`） | IC 选区（`editor select START END`） |
| --- | --- | --- |
| 偏移域 | AX 文本，也就是调用者在快照里能看到的文本。浏览器负责把它换成 DOM/模型位置（regex101 证明了这一换算）。 | IC 的 surrounding 文本。调用者事先**看不到**，只能从上一条 editor 命令回执的 preview 里得到。 |
| Compose | Bitwarden 上成功，样本 2 次。 | offset=-1，`contains()` 为假，**一律不发**（`SRC …/CurrentEditorInput.kt:99-100`，`EditorSurrounding.kt` 中 `interpretable` 要求 offset≥0）。 |
| 网页 | 改为观察后读回之后，Wikipedia 2 次、regex101 1 次都成功。 | Wikipedia 有一次早先的正式成功（`final-api34-capabilities.md` 引用，我未重读该原件）；regex101 单行 pattern 上用过，那里 AX 文本与 IC 文本相同。 |
| 普通 EditText | Contacts 上 2 次成功。 | 能用。 |
| VD | 理论上可用，但**从未实测**。 | 不可用（只限主屏）。 |
| 不需要 REF 或快照 | 不行。 | 可以。大约省 0.4s：IC 选区约 0.45s，AX 选区连同观察约 0.88s（`mapping-result.md`）。 |

**删除 IC 选区会丢什么真实能力**：在原件里找不到一个“AX 选区试过且失败、而 IC 选区成功”的字段。WPS 上两者都不行（AX 没有动作，IC 选区请求被忽略）；Termux 上 IC 的 surrounding 不足，IC 选区同样不会发出。剩下的只是理论上的情形：有 IC 但没有 AX 节点、或没有 SET_SELECTION 动作的自绘编辑器，目前没有样本。另一面，IC 选区有一个**新发现的使用隐患**【推断，基于原件中 51 与 49 的差异】：在带装饰字符的编辑器里，调用者从快照算出的偏移会被 IC 选区按另一套域解释。超出已知范围时不发送，但像 20 这样的落点会被 IC 静默选到错位的地方，而且 IC 自己读回“与请求匹配”，报 DONE。

**保留的维护量**【源码 SRC】
- 生产代码约 100 行：`CurrentEditorInput.kt` 中的 select 分支和参数校验约 25 行，`EditorSurrounding.contains/hasSameSelection` 和 `EditorVerification.select` 约 30 行，`EditorConnection.select` 和 `BrowserAccessibilityService.kt:55-58`，host 侧 `editor_input.py` 与 `ui_commands.py:684-692` 约 25 行，`skills/androidctl/input.md` 一段约 12 行。
- 测试约 100 行：`CurrentEditorInputTest.kt` 4 个用例，`test_editor_input.py` 若干，外加 wire contract 的一项。
- 代码量不大。**真正的成本在公开面**：两个选区命令、两套偏移域，要教 LLM 调用者分清。
- `editor select` 在 `d5ae012` 才加入，不在任何发布标签里（`git tag --contains d5ae012` 为空；基线 `7d5be70` 是 v0.3.0 之后的提交）。所以“后删会改变公开命令”不成立，现在删没有兼容成本。

**入口能否合并而能力不丢**：
- 现在的候选名是：AX 选区 `select REF START END`（只存在于研究原型，8 个文件）；IC 选区 `editor select START END`。`editor commit` 计划并入 `type --via editor` 之后，只剩一个孤立的 `editor select`。
- 如果保留 IC 选区，应改为 `select START END --via editor`，并且不带 REF，与 `type [REF] TEXT --via editor` 同构；文档写明“偏移按 editor 文本域”。但同一个动词底下两套偏移域，反而更容易误用。在没有 IC 独有样本的前提下，我更倾向于只留 AX 版。

**“先 AX 选区、再 `--via editor` 插入”实测覆盖**（用户 U27 第 8 条，`R/r2-work.md:22`）

| 类别 | 实测 | 结果 |
| --- | --- | --- |
| 普通框 | Contacts First name，caret 与 range 各 1 次，即时读回版 APK `e7d…` | 两笔 batch 都 DONE，晚读的业务值正确，相邻字段不变【原件 `contacts-*-batch`】。 |
| 网页框 | Wikipedia：即时版 2 次选区失败，后缀未发；观察后读回版 range 1 次 DONE，城市名 batch 1 次两步都 DONE。另有 regex101 多行框 1 次（select 再显式 commit） | 观察后读回版全部成功；regex101 的偏移换算正确。 |
| Compose | Bitwarden caret 与 range 各 1 次（只有即时版） | AX 选区 DONE，commit 实际写入正确，但报 UNKNOWN（offset=-1），batch 停【原件】。 |

**偏移域不同时调用者怎么办**：如果只留 AX 选区，调用者一律用快照里节点文本的 UTF-16 偏移，规则只有一条。如果两者都保留，调用者要先跑一次 editor 命令拿到 IC preview 才知道 IC 偏移，这是额外的往返和出错机会。上一版定稿写的“偏移以编辑器的真实文本为准”**必须改正**：AX SET_SELECTION 的参数由 Chrome 按 AX 文本位置换算（`R/regex101-ax-ic-selection-20261119.md` 中 Chromium 158 源码链，以及原件 031/032）。按“真实文本”的 46..49 去做 AX 选区，会选错位置。

### 2.3 第 4 题：AX 选区怎样确认即时后态

**实施方主张**：选 C，把同一次读回挪到已有的动作后观察之后；判定规则仍是：匹配为 DONE，与动作前相同为 UNKNOWN，其它已观察到的值为 FAILED。

**核查**：即时读回版 3 次网页选区全部失败（Wikipedia caret、range、城市名，`r2-text-runtime.md` 与原件）。观察后读回版 3 次全部 DONE（Wikipedia range、城市名 batch、regex101）。四个时间点证明读回发生在观察之后（`after-runtime` 评审，`mapping-result.md`：postObserve 4119970→4120276，readBack →4120320）【原件，API34，Chrome Beta158，3 次】。Contacts 和 Bitwarden 在即时版下已经匹配。

**判断：这是实施细节，不需要用户裁决。**
- 公开合同不变：“DONE 当且仅当最终那一次读回匹配”。
- 不增加 RPC。动作后观察本来就会跑，原件里即时版也照样跑了观察（settle 164ms）。
- 网页上的证据一边倒。

唯一的保留意见：观察稳定不等于选区已同步。如果站点让选区落后于观察结束，结果会落到“与动作前相同则 UNKNOWN”，不会误报。这个分类规则（即 B 的那部分）应当并入 C，主稿已经这样写了。另外有两个边角要在实现里明确：
- 观察被关闭或预算不足时，仍在剩余预算内读一次；预算不足时给 UNKNOWN，不能跳过读回。
- D（显式“期望选区”等待条件）是新的公开机制，没有必要。

### 2.4 第 5 题：IC commit 能否凭相对局部形态得到 DONE

**源码事实**【SRC `EditorSurrounding.kt`（文件内第 11–12 行 `interpretable`；`EditorVerification.commit` 文件内第 72–95 行）】
- 绝对锚点门有两层：before 与 after 都必须 `offset >= 0` 且选区端点在文本内，否则 UNKNOWN（“Surrounding text has no interpretable absolute selection”）；其后用 `before.offset + low` 推算绝对插入点和光标，再要求 after 落在预测范围内且文本逐字相同。
- 由 §2.2(b)，offset=-1 出现在所有不覆写 getSurroundingText 的 IC 上，也就是**所有标准 Compose 文本框**。所以在 A 下，`--via editor` 在 Compose App 里永远得不到 DONE，batch 一律停。Bitwarden 两笔都是这样，而实际写入都正确。

**B 的具体风险**
1. **重复上下文**：只以光标为锚，看不出“插在了另一个局部内容相同的位置”。例如在一串相同字符或相同行中间插入。只有当 App 或输入法在读取 before 与 commit 之间移动了光标时，这个问题才会出现。在绝对模式下，这种移动会被 offset 暴露；B 下会被放过。
2. **窗口裁切**：默认实现给出光标前 N 个、选区本身、光标后 N 个字符。候选请求 `max(128, len(TEXT))` 个前文，所以完整的 TEXT 总能看到，但看不到窗口外的字段内容。这一点与绝对模式相同，不是新风险。
3. 局部匹配都不证明整字段、因果或持久化。这一点主稿已写明。

**建议的匹配规则**（实施细节，供 B 采用）：
- after 的光标前缀必须以 `before.text[:low] + TEXT`（取窗口能覆盖的那部分）结尾，光标后缀必须以 `before.text[high:]`（同样按窗口截取）开头；
- after 的选区必须是折叠的光标；
- before 窗口两侧全为空时，要求 after 文本恰好等于 TEXT；
- 其它情况一律 UNKNOWN；
- 回执里注明 `proof: "relative"`，与 `"absolute"` 区分。

**更简单的替代 E**：带 REF 时，worker 本来就要读焦点节点做身份核对，可以在提交后像 fill 那样读该 AX 节点的整段文本和选区，用 AX 域做绝对核对（Bitwarden 的 AX 文本与真实文本一致）。代价有三：
- 要把 editor 通道和 AX 读取耦合起来；
- 网页上 AX 有时延，需要像 §2.3 那样放到观察之后读；
- 遇到占位符或标签又会碰上 fill 已知的问题。

E 比 B 重，不建议作为首选，可以作为“REF 形态下的增强”留到以后。

**这题是否需要用户定**：需要，但措辞要改。现在的写法（“能否凭相对局部形态得到 DONE”）用户看不出后果。应改为：“在 Compose 等 offset 未知的编辑器里，`type --via editor` 是否允许报 DONE 并让 batch 继续：A 一律 UNKNOWN 停批；B 以光标为锚做局部精确匹配，DONE 证明面更窄”。我同意推荐 B。证据只有 1 个 App、2 笔、API34 模拟器。

### 2.5 剪贴板与借贴（建议提交 11）

- 本轮没有新的设备证据。三条写路径（新 set、借写、恢复）都缺少抑制标记，候选中可见（SRC `ClipboardAccess.kt:114` 是唯一的 `setPrimaryClip` 调用点，全文无 `SUPPRESS`）【源码】。SystemUI 只在模拟器上、或来源为 shell 且带标记时抑制浮层（`agents5/23/up/ClipboardListener.android-14.0.0_r1.java:51-54, 87-93, 119-124`）【源码】。
- VD 上的 paste 主稿和底表都标为未测（`integration-draft.md:72, 299`；`r2-text-runtime.md` 末段“本窗没有新跑 VD PASTE”），标注如实。
- 上一版定稿 C11 写“不成立则文档写明后台屏任意文字只能整字段 fill”，这个推论过强。VD 上还有 keys（仅限可映射字符）以及“clipboard set＋App 快捷键或菜单”两条路。实施方的修正成立。

### 2.6 被新证据推翻的上一版评审定稿要点

1. §2.6.3“偏移以编辑器的真实文本为准”→ 应为“按 AX 节点文本，即快照所见的 UTF-16 偏移；文本可能含标签或装饰”。
2. §2.5“每次注册拿到的初始数组是空的”→ 拿到的是进程缓存，可能非空，一律不消费。
3. §2.5 规则 7“首版只支持恒等的显示变换”→ 改为照平台逐矩形语义。
4. §2.5“注销返回后仍可能收到一次在途回调”→ 没有“一次”的上限，关闭消费即可。
5. §2.5 验收“连续 1000 条坐标命令无残留”→ Java 一侧无法计量原生集合，这条验收不可执行，同意删除。

---

## 3. 确定缺陷与证据不足的断言

**确定缺陷（实施方方案）**
1. 主稿第 2 题把 B 与 C 的差别（“非保矩形业务未验”）当成真实取舍。但显示变换在 A14–A16 上恒保矩形，这道题本不该交给用户【源码 A15 ProjectionSpace/Output/DisplayDevice】。
2. 主稿第 3 题推荐 B 的依据把 IC **commit** 的成就（多行 49 字 DONE、HTML 直接修改）算到 IC **select** 头上。它没有提 IC select 在 Compose 上结构性不可用（offset=-1，不会发送），也没有提偏移域对调用者不可见的隐患。“后删会改变公开命令”与仓库事实不符：该命令不在任何发布标签里。
3. 主稿第 5 题的措辞没有点明“绝对门等于 Compose 上 editor 通道永远停批”。这是因为 Compose IC 不覆写 getSurroundingText，属于结构原因，不是一次样本的偶然。

**候选源码里的事实（不算 bug，但文档要写清）**
- `skills/androidctl/input.md` 第 15–40 行说“unknown offset … sends no selection”，但没有告诉调用者这等于“Compose 上永远不可用”。

**证据不足的断言**
- AX 选区“主屏与后台屏共用”：VD 上从未实测【需设备】。
- “Compose 上实际输入正确”：只有 1 个 App、2 笔、API34。
- 回调耗时：只有 5 个样本，两个镜像，没有真机。
- 坐标预测在 VD 上、在全手势（drag/pinch 跨窗口）上、以及作为默认路径集成后：都没有实测，主稿已如实标注。
- regex101 偏移换算：只有 1 次。

---

## 4. 值得删除或简化的机制

1. **IC 独立选区（`editor select`）**：删除。生产代码约 100 行、测试约 100 行、一段文档、一项 wire contract；公开面只剩一个选区命令、一套偏移域。保留 IC commit。
2. **几何中的“不保矩形”分支**：不实现外接矩形近似，也不为它准备测试，只留一行“预测不可用”的拒绝。
3. **只认恒等变换的门**：删除，换成照平台的逐矩形变换。
4. 研究原型里的 `retained` 队列（保留最近 2 个监听器）和 2s quiet：只用于研究，产品里不要带。
5. “期望选区”等待条件（第 4 题 D）和“严格/相对证明开关”（第 5 题 C）：都不做。
6. 1000 次连续命令的验收：删除。

---

## 5. 需要用户裁决的事

**(1) 删除 IC 独立选区吗？（原第 3 题）**
- A，删除，只留 `select REF START END`（AX）。得：一个命令、一套与快照一致的偏移域，Compose 可用，避免装饰字符导致的静默错选。失：无 REF、无快照的快速选区（约省 0.4s），以及理论上“有 IC 无 AX 节点”的编辑器，目前没有样本。
- B，两个都保留，IC 版改名为 `select START END --via editor`。得：保留上述理论能力。失：两套偏移域要教调用者，IC 版在 Compose 上永远不会发送。
- **推荐 A。** 剩余未知：有没有真实 App 的编辑器只能靠 IC 选区。以后真碰到，再以 `--via editor` 的形式加回来，成本低。

**(2) Compose 等 offset 未知的编辑器里，`type --via editor` 能否报 DONE？（原第 5 题，改写）**
- A，绝对门：一律 UNKNOWN 停批。
- B，以光标为锚做局部精确匹配（规则见 §2.4），回执注明是相对证明。
- **推荐 B。** 风险：局部内容重复时插错位置看不出来；不证明整字段。剩余未知：除 Bitwarden 外的其它 Compose App。

**归为实施细节（不需要用户裁决）**
- 第 2 题：照平台逐矩形语义，支持旋转和缩放；遇到不保矩形则拒绝。
- 第 4 题：选 C，观察之后读回；同时保留“与动作前相同则 UNKNOWN”；预算不足时 UNKNOWN。
- WindowInfosListener 的实现约束：worker 内不得有其它常驻监听器；不消费注册返回的缓存；回调只做原子赋值。

**建议提交的修订版边界与验收**
- **提交 9（坐标）**
  - 范围：逐矩形变换，含 float32 往返；首个 DOWN 的 UID 预测；命中自身或无样本时，在发送前拒绝；零发送时跳过动作后观察。
  - 验收（离线）：用 P5 回调里的真实区域（导航栏 `[1184,0,1280,720]`）做右、下边界单测；90/180/270 度和非整数缩放下，用平台公式核对边界像素。
  - 验收（设备）：在 API34 和 MuMu 上，不带 `--expect`，横屏 Chrome 滑动和输入法字母点击各 1 次；点自身管理界面时发送前拒绝；VD 普通 App tap 1 次；release 构建。
- **提交 10（文本）**
  - 范围：`type [REF] TEXT --via keys|paste|editor`；`select REF START END`（AX，偏移以快照文本为准，观察后读回）；删除 `editor select`；按用户对 (2) 的选择实现 commit 证明。
  - 验收：Contacts、Wikipedia、Bitwarden 各 1 次 `select`→`type --via editor` batch 完整跑通（Bitwarden 是否 DONE 取决于 (2)）；regex101 装饰字符映射 1 次；非焦点 REF 走 editor 时 0 次 commit 且被拒；**VD 上 AX 选区 1 次（新增，至今未测）**。
- **提交 11（剪贴板）**
  - 范围不变。
  - 验收：单测断言三条写路径都带抑制键、原有 extras 不丢、来源为 shell；VD paste 实测 1 次单列；真机来源分支保留为缺口。

---

## 6. 未能核实的内容

- OEM ROM 是否可能改出不保矩形的显示投影（只核了 AOSP A15 的 CompositionEngine）。
- Bitwarden 实际打包的 Compose 版本是否与 androidx-main 一样不覆写 getSurroundingText（只核了 androidx-main，与原件中的 offset=-1 一致）。
- `final-api34-capabilities.md` 引用的 Wikipedia IC 选区正式成功原件（我没有重读该原件）。
- P1/P2 的 5/8ms 和 MuMu 的 31ms 只读了 `result.md`；P3/P5 的耗时由我从原件重算。
- 所有运行结论都只在模拟器上；真机、VD 上的 AX 选区、VD 上的 paste 都没有任何实测。
