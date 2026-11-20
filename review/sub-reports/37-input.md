# 专题 37：坐标输入、文字输入与选区、按键、剪贴板、截图坐标资格（第四轮，`4235af9`）

核查者：独立只读核查。没有设备，所有设备事实只来自实施方在 Android Studio API34 模拟器和 MuMu（API35）上的原件。没有真机或厂商 ROM 结论。

路径缩写：
- `SRC`：产品仓库只读克隆，HEAD=`4235af9`。下文的 `K/` 指 `SRC/android/app/src/main/java/com/rainng/androidctl/`。
- `PKG`：实施方本轮材料包。`ST` 指 `PKG/ultragoal/2026-11-11-next-improve/stages/41-route-rebuild`。
- `W`：本人工作目录 `…/scratchpad/agents7/37/`。重算脚本和下载的 AOSP 文件都在这里，AOSP 文件在 `W/aosp/`，版本标签写在文件名里。另外复用了 `…/scratchpad/agents6/28/aosp/`（InputDispatcher、Transform，android-14/15/16.0.0_r1）和 `…/scratchpad/agents5/23/up/`（ClipboardService、AccessibilityController 等）。

术语：
- **AX**：无障碍节点动作，例如 SET_SELECTION、PASTE、SET_TEXT。
- **IC**：输入法连接 InputConnection。`type --via editor` 通过 App 内的无障碍服务（设置里的“browser support”）拿到它，再调用 `commitText` 和 `getSurroundingText`。
- **定向注入**：`IInputManager.injectInputEventToTarget(event, mode, uid)`。平台只把事件交给属于这个 UID 的窗口，否则在分发前丢弃并报“目标不符”。
- **预测**：没有 `--expect` 时，坐标命令先用窗口元数据内部接口 `WindowInfosListener` 算出首个按下点落在哪个 UID 的窗口上，再按这个 UID 定向注入。
- **VD**：后台虚拟屏。

---

## 1. 结论摘要

**整体状态**：坐标接收者这条线的实现是正确的，而且“出错方向安全”：预测错了只会在首个按下就被平台拒绝，结果是“未发出”，不会点到别的 App。文字输入的三个通道和 AX 选区在已有样本里写入正确，回执也如实。问题不在功能能不能用，而在于：
- 相对证明有一个会误报 DONE 的缺口；
- 借贴把“没变化”判成 FAILED；
- 公开面明显宽于真实任务的用量。

**最重要的发现**

1. **`type --via editor` 的相对证明会误报 DONE**（确定，中）。
   - 编辑器偏移未知时（标准 Compose 都是这种情况），匹配器只比较“读回里实际还在的前后文”。前文、后文消失，或者提交被 App 拒绝而原文恰好是周期重复的，都会报 DONE。
   - 典型例子：输入 `0` 到已满长度的 `00`；单字符验证码框把 `5` 换成 `7`。
   - JVM 测试还把其中一种情况写成了期望值：`CurrentEditorInputTest.kt:40`，before 为 `"ab|c"`，after 为 `"x"`，测试判 DONE。
   - 【源码＋重算 `W/relative_check.py`】
2. **借贴把“粘贴后字段没变化”判成 FAILED**（确定，中低）。
   - FAILED 之后会立刻恢复用户原剪贴板。如果 App 稍后才真正读剪贴板，粘进去的是用户自己的原内容；调用者看到 FAILED 还可能重试。
   - 这和 `select` 的规则不一致：`select` 读回与动作前相同判 UNKNOWN。【源码】
3. **主屏桌面上的定向点击很可能被平台以“目标不符”拒绝**。
   - 原因：InputDispatcher 会把壁纸窗口一并加入被触摸窗口，而定向注入要求所有被触摸窗口同属一个 UID。
   - 默认预测和 `--expect` 都会受影响，结果是“未发出”，不会误点，但相比基线的不定向注入是能力回退。
   - 证据包里没有任何一次在桌面上点击的样本。【上游源码推断，需设备】
4. **坐标几何“照抄平台”本身没问题，但第三轮评审方的论证偏重了**。
   - 安全由平台在分发时核对 UID 保证；照抄只决定边缘 1 像素会不会被误拒。
   - 而且 Android 14 起，无障碍窗口列表本身就是由同一份 WindowInfos 数据算出来的。用公开的窗口列表做预测，代价小得多，效果几乎相同。
   - 现在代码已经写好，并在 14/15 上实跑过；14–16 的签名和常量我逐项核过，都一致。建议**保留但冻结**：以后平台签名一旦失效，退到无障碍窗口列表，不再追加内部接口。【源码＋上游】
5. **用量证据**。
   - 本轮两阶段八臂真实任务（GitHub、Maps）共约 84 次 CLI，文字只用了 `fill`。`type`（三种通道）、`select`、`key`、`ime`、`clipboard`、`tap/swipe/drag/pinch/timeline` 全部是 0 次。
   - 上一轮 327 次真实流程里，`tap` 4 次、`editor` 选区和提交各 4 次（都在 regex101 多行框），`type`、`clipboard`、`drag/pinch/timeline`、`ime` 都是 0 次。
   - 这些通道的设备证据全部来自专门设计的验证用例。【重算 `W/count_cmds.py`，原件】
6. **证据审计**。
   - Contacts、Bitwarden 相对证明、regex101 装饰偏移、空串 UNKNOWN、IME 字母键、自身窗口拒绝，这几条原件都对得上。
   - VD 上的 Contacts 系列（含“点位落在 First 而不是 Last”和 VD select→paste）以及 MuMu 横屏和自身拒绝，原件**不在包里**，只有成本汇总，核不到。
   - 所有设备运行用的是阶段 release 构建：坐标线是 `2beaadd`，文字线是 `826c180`，**不是最终 `4235af9` 的 APK**。其后 `3ef0ec6` 又改过 `VisualInput` 的图资格逻辑。

**待决题**：四道待决题都不直接落在本专题（见 §5）。本专题需要用户定的只有一项：文字通道收敛（§6）。

---

## 2. 确定缺陷

### D1（中）`--via editor` 相对证明对“旧文消失”和“周期重复的空提交”误报 DONE

- **位置**：`K/EditorSurrounding.kt:87-115`（`relativeCommit`）。前文覆盖量取 `prefixUnits = minOf(prefix.length, insertion)`，后文同理，只比较读回里实际存在的那部分；只有两侧原本都为空时，才要求整段相等。
- **触发条件**：编辑器 offset 为 −1（标准 Compose 文本框都是这样，见上一轮核实），并且出现下面任一情况：
  - (a) 提交后光标前的旧文不见了。例如单字符验证码框在 onValueChange 里只保留最后一个字符：before `5|`，输入 `7`，after `7|`。结果报 DONE，字段实际是 `7`，不是 `57`。
  - (b) 提交被 App 拒绝（例如 Compose 里在 onValueChange 做长度上限），而光标前的旧文以 TEXT 为周期重复。例如 `00` 加 `0`、`haha` 加 `ha`：after 与 before 完全相同，也报 DONE。
- **证据**：
  - 【重算】`W/relative_check.py`（输出在 `W/relative_check.out`）逐行移植了 `relativeCommit`。上述两类情况都给出 MATCH，而“非周期的空提交”（`abc` 加 `c`）正确给出 UNKNOWN。
  - 【源码】`SRC/android/app/src/test/java/com/rainng/androidctl/CurrentEditorInputTest.kt:40` 把 `EditorSurrounding("abc",-1,2,2)`、输入 `"x"`、读回 `EditorSurrounding("x",-1,1,1)` 列为应当 relative DONE 的用例。也就是说，测试固化了 (a)。
- **影响**：
  - DONE 让静态 batch 继续往下走。
  - `input.md:54-67` 写的是“比较实际重叠的相邻上下文……更宽的窗口不臆造重叠之外的知识”。这是如实描述，但 LLM 调用者读不出“旧文丢了也会 DONE”这层后果。
  - 绝对模式（offset 已知）不受影响：上面两种情况在绝对模式下都会因为光标或范围预测不符给出 UNKNOWN。
- **最小修正**：在相对模式下，要求读回覆盖 before 窗口里实际存在的前文和后文，具体是 `insertion >= min(prefix.length, 128)` 且光标后文长度 `>= min(suffix.length, 128)`，否则给 UNKNOWN。读回请求的前文量本来就不小于 `128 + TEXT 长度`，这个要求只会让真正截断窗口的编辑器得到 UNKNOWN，方向保守。同时把测试第 40 行改成期望 UNKNOWN。大约 5 行。
- **确定程度**：逻辑缺陷可以确定（源码加重算）。真实 App 里出现的频率没有样本。

### D2（中低）借贴把“没变化”判成 FAILED，随后恢复剪贴板

- **位置**：
  - `K/BorrowedPaste.kt:33-43`：`PasteText.verify` 只要读回不等于期望就给 `differs`，包括读回与动作前完全相同的情况。
  - `:175-181`：`differs` 映射为 FAILED。
  - `K/UiActions.kt:321-335`（`awaitPasteValue`）：最多轮询到 `--settle` 截止。
  - 之后 `cleanup()` 立即恢复原剪贴板（`BorrowedPaste.kt:240-292`）。
- **触发条件**：PASTE 已被平台接受，但 App 在 settle 窗口内还没有完成粘贴。例如网页渲染进程忙，或者调用者给了 `--settle 0`。
- **影响**：
  - 结果是 FAILED，而 FAILED 通常被理解为“可以再来一次”，于是可能重复插入。
  - 更糟的是，恢复之后 App 才读剪贴板，粘进字段的是用户原来的剪贴板内容，可能是隐私数据。
  - `input.md:126-128` 承认了“App 稍后读剪贴板可能看到恢复后的内容”，但回执是 FAILED，而不是 UNKNOWN。
- **对照**：`select` 的规则是“与动作前相同则 UNKNOWN，其它已知值才 FAILED”（`input.md:29-31`，`NodeSelection.outcome`）。借贴应当采用同一规则。
- **最小修正**：读回与动作前的文本和选区都相同时给 UNKNOWN，只有出现第三种值才给 FAILED。BorrowedPasteTest 加一条对应用例。
- **确定程度**：映射逻辑可以确定。迟到粘贴的时序【需设备】，本包中网页和 VD 的粘贴样本都在 settle 内完成。

### D3（中，需设备）桌面等“带壁纸”窗口上的定向注入很可能目标不符

- **上游事实**【源码】：
  - `W/aosp/InputMonitor-14.0.0_r1.java:278-281`：只要窗口是壁纸目标、未锁屏、且没有关闭壁纸触摸，WM 就给它设 `DUPLICATE_TOUCH_TO_WALLPAPER`。
  - InputDispatcher 在首个按下时把它下面的壁纸窗口加入被触摸集合（A14 `InputDispatcher.cpp:2420-2433`，`findWallpaperWindowBelow` 在第 6855 行，不检查 UID 和可触摸性）。
  - 随后，定向注入要求被触摸集合里**每个**窗口都属于目标 UID，否则报 TARGET_MISMATCH（A14 第 2586-2603 行；A16 第 2743-2754 行同样存在）。
  - 壁纸一般属于 SystemUI（ImageWallpaper）或动态壁纸 App，与桌面 App 不是同一个 UID。
- **推论**：在主屏桌面上点图标，默认预测得到桌面 UID，`--expect 桌面包` 也一样，平台都会拒绝。产品会如实报“未发出”（`AppInput.kt:160-167` 精确匹配异常文本，`PointerGesture.kt:247-250`），不会误点，但这件事做不成。基线 `7d5be70` 用的是不定向注入，没有这个问题。
- **证据缺口**：我在 PKG 和上一轮材料里搜过所有带 `inputTarget` 的回执，没有一次落在桌面上（`grep` 结果为空）。Launcher3 是否主动关闭了壁纸触摸，我没有核实。
- **建议**：用一次桌面图标点击定案。如果确实被拒，有两个办法：
  - 在文档里写明“桌面图标用 `open` 或 `click REF`”；
  - 或者仅在预测命中带壁纸复制标志的窗口时改用不定向注入。代价是这一类窗口失去 UID 门。

### D4（低）空串 editor 在绝对证明已经完整覆盖时仍判 UNKNOWN

- **位置**：`K/EditorSurrounding.kt:79`。
- **原件**：`ST/14-input/runtime/text-api34/commands/formal/044-clear-owned-decorated-web-editor/stdout.bin`。before 是 offset 0、长度 23、选中 0..23，前后都没有剩余文本，说明 before 就是整个字段；after 是 offset 0、长度 0。这已经证明字段为空，但回执给的是 UNKNOWN，batch 也停了。
- **说明**：这是有意的保守，并非误报，代价是“清空”这类合法操作无法得到 DONE。
- **二选一**：
  - 在绝对模式下，before 覆盖了整个字段（offset 为 0，且返回长度小于请求长度）时，接受空读回；
  - 或者在参数校验阶段直接拒绝空 TEXT，并在文档里写“清空用 `fill REF ''`”。

  第二种更简单、更可预期。

### D5（低）editor 的 REF 绑定只核到“同包”

- **位置**：
  - `K/UiActions.kt:47-66`：worker 只核原 REF 是当前 AX 输入焦点；
  - `K/BrowserAccessibilityService.kt:28-51`：桥接端只核 `EditorInfo.packageName` 与 REF 的包名相同（`:43`）。
- **风险**【推断，需设备】：AX 焦点已经移到同一 App 的另一个框，而无障碍输入法的 `onStartInput` 还没跟上（这一步是异步的）。这时 commit 会写进前一个框，而且前后读的是同一个错误连接，照样报 DONE。
- **影响有限**：前一条命令的动作后观察通常已有几百毫秒，原件 006/009 也证明“同包另一个未聚焦 REF”会被正确拒绝。
- **建议**：不加新机制，只在 `input.md` 里写明“绑定粒度是同包加 AX 焦点”。如果以后要加强，可以在 AX 文本与 IC 文本相同（非装饰字段）时比对两边的选区。

### D6（低）`--via editor` 的观察开关取决于 `--settle` 是否在命令行出现

- **位置**：`SRC/androidctl/src/androidctl/ui_commands.py:1316-1319`。只有在命令行上显式写了 `--settle`，才会附带 `observe=true`；默认不做动作后观察。`keys` 和 `paste` 则总是观察。
- **问题**：同一个 `type` 命令，三个通道的默认行为不同，而且开关是“参数是否出现”，不是“参数的值”。对 LLM 调用者来说，这既难预期也难记。
- **建议**：统一为所有通道默认观察，或者统一为都不观察并显式加 `--settle`。我倾向于统一默认观察，因为 editor 省掉的约 0.5 秒不值得为它保留一条例外规则。

### 已核实为正确、不是缺陷的点（避免误报）

- **首个事件目标不符即判“未发出”**：可靠。A14/15/16 的 `InputManagerService.injectInputEventToTarget` 在 TARGET_MISMATCH 时抛出同一句 IllegalArgumentException（`W/aosp/IMS-14.0.0_r1.java:894-902`，15、16 只差注解）。A14 的分发器在第 2327 和 2586 行出错时直接返回空目标集合，**确实一个事件都没发出**。首个 DOWN 用 WAIT_FOR_RESULT（`PointerGesture.kt:193`），所以能同步拿到这个结论。
- **平台返回 FAILED（该点没有可触摸窗口）时判 UNKNOWN**：偏保守，但不算错。
- **零派发后不观察、保留图资格**（`01077a8`）：
  - `VisualInput.kt:321-327` 只在 `mayHaveDispatched` 为真时观察；
  - `UiBackend.kt` 的 `observedOperation` 也只在可能派发时撤销资格；
  - 预测阶段抛出的拒绝发生在派发之前，不撤销资格；
  - 监听器注销失败时走专门的结果，标“未发出”加 exit 3，同样不撤销。
- **几何与平台一致**：
  - `K/WindowInput.kt:32-131` 跳过的标志是 NOT_VISIBLE(1<<1)、NOT_TOUCHABLE(1<<3)、SPY(1<<14)，与 A14/A16 `InputConfig.aidl` 的取值相同（`W/aosp/InputConfig-*.aidl`）；
  - 逐矩形变换和 floor(v+0.5) 的取整与 Transform.cpp 一致（上一轮已核）；
  - 窗口顺序是从前到后，与分发器相同。
- **隐藏接口签名 14→16 稳定**：
  - `WindowInfosListener.register()` 返回 `Pair<InputWindowHandle[],DisplayInfo[]>`，`unregister()`，抽象方法 `onWindowInfosChanged`；
  - `DisplayInfo` 的 `mDisplayId`、`mLogicalSize`、`mTransform`；
  - `InputWindowHandle` 的 `ownerUid`、`displayId`、`inputConfig`、`touchableRegion`。

  这些在 android-14/15/16.0.0_r1 里逐一存在，类型相同（`W/aosp/WindowInfosListener-*.java`、`InputWindowHandle-*.java`）。平台字段是 `final`，桩里不是 `final`，这不影响按名字访问字段。IClipboard 的四个方法在 14/15/16 的参数表也相同（agents5 的 `ClipboardService.*`）。
- **预测与平台的已知差异，都只会导致误拒**：
  - 分发器会丢掉“尚未注册输入通道”的窗口（A14 第 5007 行附近），而回调里还在。
  - 回调是某一时刻的样本，分发器可能已经更新。
  - 不可信遮挡下，平台会以无目标判为不符。

  这些情况的结果都是“未发出”，不会误点。
- **其它几何场景**：画中画、分屏分隔条、圆角和刘海装饰、旋转都直接用平台的同一份可触摸区域，所以在源码层面没有额外差别。其中只有旋转在 MuMu 上实跑过一次，原件不在包里。

---

## 3. 设计与简化

### 3.1 坐标接收者：现形态可以接受，但不应再加码

| 项 | 判断 |
| --- | --- |
| 安全来自哪里 | 来自平台在分发时的 UID 核对，不来自预测本身。预测只决定“填哪个 UID”，以及“命中 AndroidCtl 自身时在发送前拒绝”。所以预测精度只影响误拒率。 |
| 第三轮“照抄平台逐矩形语义，含 float32 往返”的建议 | **技术上对，但论证过重**。偏差只在区域边缘 1 像素，而 LLM 选的点几乎都在控件中部。真正必需的只有“支持旋转”这一项，任何正确的变换都能做到。现在这部分代码约 70 行，测试用的是 P5 真实矩阵的具体值，属于行为测试。不必回退，但也不必再为它的精确性投入。 |
| 第一轮定稿“内部接口优先、无障碍窗口列表作退路” | **代价偏高的选择**。我核了 A14 的 `AccessibilityWindowsPopulator`（`agents5/23/up/…:150-185`）和 `AccessibilityController`（`:1580-1760`）：无障碍窗口列表本身就是系统服务从同一份 WindowInfos 回调里算出来的，按层排序，带可触摸区域。worker 在 `VisualInput.geometry()` 里本来就读了这份列表，用来判断图是否过时（`VisualInput.kt:422-451`）。拿它在按下点做层序命中，再用根节点包名换 UID，大约 30 行，不需要编译桩、Gradle 任务、保留规则、LinkageError 处理、注销失败的 exit 3 结果，也没有“worker 内不得有别的常驻监听器”这条约束。差异只在被它排除的窗口类型（例如 INPUT_CONSUMER、SECURE_SYSTEM_OVERLAY），而这些差异同样只会导致误拒。 |
| 现在要不要换 | **不建议现在换**。现实现已在 API34 和 MuMu API35 的 release 构建上实跑（IME 字母、静止页、横屏、VD、自身拒绝），签名在 14–16 稳定，换实现还要重做设备验证。建议定一条规则：**将来平台签名失效或行为漂移时，退到无障碍窗口列表，不再追加任何内部接口。** |
| “派发后用事件确认接收者”或“不做预判” | 平台没有提供“谁收到了”的回执。完全不做预判等于回到不定向注入，失去自身 UID 保护，而不定向的按键路径已经接受了这种“检查时与使用时之间的竞态”（`input.md:169-172`）。如果用户更看重简单，可以考虑“不定向注入，加上发送前用已采集的窗口列表核对按下点不在自身窗口”，这会与按键路径保持一致。代价是失去分发时的 UID 门，而这道门实际只挡住了“检查到发送之间突然弹出的别家窗口”。我不推荐，因为现实现已经做完且安全。 |
| 可小幅简化 | `WindowInputCleanupFailure` 为“注销抛异常”单独设计了一种结果外形和 exit 3（`WindowInputListener.kt:139-159`）。`unregister()` 是无返回值的原生调用，实际只可能抛 LinkageError。可以并入普通的“预测不可用、未发出”，同时把 worker 标成需要重启，从而少一种公开结果。属于实施细节。 |

### 3.2 截图坐标资格

- **规则清楚，失效条件齐全**：
  - 同一会话租约、同一 ID、120 秒以内、连接代际相同；
  - 显示形态完全一致：显示、尺寸、旋转、锁屏、放大和窗口列表；
  - 任何可能的派发都会撤销资格，观察到变化也会撤销；
  - 区域截图既不恢复也不延长资格。

  【源码 `VisualInput.kt:37-57, 281-307`】
- **映射**：`x*源宽/交付宽`，没有加半像素（`PointerPlan.kt:41-64`）。JPEG 预览缩放 2/3 时，系统性偏左上不到 1 个源像素，可以忽略。
- **残余风险**：同一窗口内的内容自行滚动（例如轮播、自动加载），在没有新观察时不会撤销资格，只靠 120 秒期限兜底。文档写的是“观察到变化需要新图”，这是如实描述。
- **注意**：`3ef0ec6` 在文字线和坐标线的设备运行之后重写了资格的存放方式（Bitmap 不再驻留，改为单独的 `inputQualified`）。这部分的设备证据只在 `ST/16-delivery/runtime/api34-regions` 有一次“同一完整图 ID 安全滚动”和两次拒绝，没有覆盖“零派发后同一张图再次点击”。

### 3.3 文字输入面：对 LLM 调用者过宽

| 入口 | 唯一不可替代的用途 | 真实任务用量 | 维护量（设备端行数） |
| --- | --- | --- | --- |
| `fill REF TEXT` | 整字段替换，任意 Unicode，主屏和 VD 都能用 | 本轮 12 次左右，上轮 23 次；文字类任务的绝对主力 | 已有 |
| `type --via keys` | 光标处插入可映射字符，加 Enter、Tab 等；主屏和 VD | 0 | `KeyPlan.kt` 123 加 `KeySequence.kt` 212 |
| `key KEY [REF]` | 编辑键和组合键（全选、移动、删除） | 0 | 与 keys 共用 |
| `select REF S E` | AX 选区；后接插入，实现局部替换 | 本轮 0；上轮在研究原型里有 | `UiSelection.kt` 133 加 `NodeSelection.kt` 81 |
| `type --via editor` | 光标处插入任意 Unicode，不碰剪贴板，有 IC 读回证明；装饰过的多行网页框（`fill` 在这里核对失败）；Compose 里给相对证明 | 本轮 0；上轮在 regex101 研究里 4 次 | 约 430 行（`CurrentEditorInput` 117、`EditorSurrounding` 134、`EditorBridge` 90、`BrowserAccessibilityService` 85、AIDL），另需用户在 App 设置里开启一个无障碍服务；只支持主屏 |
| `type --via paste` | 光标处插入任意 Unicode，**VD 也能用** | 0 | `BorrowedPaste.kt` 354，加 `UiActions` 中约 90 行，加文档约 50 行；借用用户剪贴板（覆盖层、隐私提示、worker 被杀时原剪贴板丢失） |
| `ime REF` | 单独请求 IME 动作；`fill --submit` 已经覆盖了常见用法 | 0 | 很小 |
| `clipboard get/set/clear` | 读写剪贴板本身就是业务需求（复制验证码等） | 0 | 约 546 行 |

**判断**：
- **是不是为单个 App 的特例长期背着**：editor 的“装饰多行框”用途来自 regex101 和 Chrome 这一个场景，Compose 相对证明来自 Bitwarden 这一个 App。paste 的“VD 任意 Unicode 局部插入”在任何真实任务里都没出现过。
- **两个“任意 Unicode 局部插入”通道是重复能力**。真实需求是“在光标处插入非 ASCII 文本”，现在有 editor 和 paste 两条实现，各带一套证明体系和一套文档。

**我独立给出的最小集合**：
- `fill`、`type`（缺省即 keys）、`key`、`select`、`clipboard get/set/clear`；
- 加**一条**任意 Unicode 局部插入通道；
- `ime` 可以并入 `fill --submit`，单独保留的代价也很低，无所谓。

两种取舍（需要用户定，见 §6）：
- **保留 editor、删除借贴**：
  - 失去的：VD 上的任意 Unicode 局部插入（VD 上仍有整字段 `fill` 和可映射字符的 keys），以及“带自动恢复的一次粘贴”。
  - 还能做的：确实需要粘贴时，调用者可以显式组合 `clipboard set TEXT` 加 `action REF --action-id 32768`（通用动作入口允许无参的 PASTE，`actions.md:147-160`），自己承担覆盖剪贴板的后果。
  - 少维护：约 500 行代码、一套四段回执和“早返回后仍要清理”的语义。
- **保留借贴、删除 editor**：
  - 失去的：不碰剪贴板的插入、IC 域的绝对证明、Compose 上的相对证明，以及多行网页框里带换行文本的正确核对（借贴按 AX 文本核对，遇到 Chrome 的 `↵` 装饰会判 differs）。
  - 少维护：约 430 行、一个 AIDL、一个需要用户开启的无障碍服务和“浏览器支持”设置步骤。

**选区偏移的可用性**：`select` 要求调用者给出 UTF-16 偏移（`input.md:16-20`）。LLM 数字符一般按码点或字形计，遇到 emoji 会算错，算错的结果是选错位置；读回会如实报 FAILED，但这仍是一次可以避免的往返。建议增加 `select REF --text SUBSTRING`，只在子串唯一出现时选中，由 worker 自己换算偏移。改动小，能明显减少误用。属于实施细节。

### 3.4 手势面

- `tap`、`swipe`、`drag`、`pinch` 都是同一执行器上的计划生成器（主机端 `pointer_plan.py` 只有 48 行），保留的成本主要在文档。
- `timeline` 是底层通用入口，真实用量为 0。它需要 JSON 轨迹，LLM 很难写对。建议从公开面隐藏或删除，只保留 drag 和 pinch 这两种语义糖；以后有需求再开放。能力上的损失是任意多段轨迹，例如画图、手写。
- `click REF --touch`（节点中心触摸）在 AX 点击无效的自绘控件上不可替代，应当保留。

### 3.5 `BrowserAccessibilityService` / `EditorBridge` 还有没有不可替代的用途

它们唯一的用途就是 `type --via editor`。UiAutomation 不提供无障碍输入法接口，所以 IC 通道只能经由 App 进程里的无障碍服务。它的价值完全取决于 §6 的取舍：如果选“保留借贴、删除 editor”，这两个文件、AIDL、`WorkerConnection` 里的桥接生命周期和“浏览器支持”设置可以整体删除。

### 3.6 文档

`input.md` 共 183 行，绝大部分在讲回执里的事实字段（attempted、callReturned、captureValid、successfulPrefix、possibleDown、omitted 等）。对 LLM 来说，真正需要的只有三件事：
- 每个通道的前提；
- DONE 证明了什么、没证明什么；
- UNKNOWN 时不要重发。

建议把字段细节移进参考页，主页保持在 40 行以内。属于实施细节。

### 3.7 评审方前几轮的哪些建议现在看是错的或代价过高

1. **第一轮定稿 §2.5 “内部接口优先，无障碍窗口列表只作退路”**：代价过高。理由见 §3.1：A14 起两者同源，精度只影响误拒。不建议现在推翻，但应当写明“到此为止”。
2. **第三轮 §3.2 和第 28 号工作记录“必须连 float32 往返一起照抄，否则边缘不一致”**：结论成立，但重要性被高估。边缘 1 像素的误拒对 LLM 调用者没有实际影响。
3. **第三轮 §2.5 和第 28 号推荐的相对证明匹配规则**：前文要以 `before.text[:low]+TEXT` 结尾，后文要以 `before.text[high:]` 开头，“按窗口截取”。这里“按窗口截取”被实现成“只比读回里还剩下的部分”，于是出现 D1。评审方当时没有写清“必须覆盖 before 窗口里实际存在的上下文”，要承担一部分责任。
4. **第三轮把 editor 和 paste 都当作“已决定保留”的能力**：在本轮“真实任务零用量”的证据面前，应当重新问一次。见 §6。

---

## 4. 证据审计

| 声称 | 核对 | 结论 |
| --- | --- | --- |
| Contacts 原生连续链：AX 选中 8..11，即 `😀中`，再用 editor 替换为 `Q🦊`，两步都 absolute DONE，字段全文正确 | 【原件】`ST/14-input/runtime/text-api34/commands/formal/008-…/stdout.bin`：select 读回 `selectedText:"😀中"`；editor 的 before 是 offset 0、选中 8..11，after 是光标 11、预览 `U29TA-ABQ🦊CD-Z`，`proof:"absolute"`。release APK `456efedc…`，源码 `826c180`。样本 1 次。 | 成立 |
| 同包未聚焦的 REF 用 editor 不会写入 | 【原件】API34 的 006 在 `raw/` 里有 trace；MuMu 的 `text-mumu/commands/formal/009` 回执为 not sent / inputFocus / exit 2。 | 成立（2 次） |
| Bitwarden（Compose）相对证明 DONE，batch 第三步真实继续 | 【原件】`029-…/stdout.bin`：两次 editor 都是 `offset:-1`、`proof:"relative"`，knownRange 和 absoluteSelection 为 null，没有 verificationRange，第三步执行了。字段文本完全在 IC 窗口内（长度 15/16）。实施方注明 batch 开始时 Email 已经聚焦，属实。 | 成立。边界：1 个 App、1 次 batch、API34。D1 的缺口没有被这次样本触发。 |
| regex101 装饰文本：AX 17..23 对应 IC 16..22 | 【原件】`042-…/stdout.bin`：select 的请求是 17..23，AX 文本长 23，选中 `line-2`；editor 的 before 是长度 22、选中 16..22，after 是 `…\ntail-🦊`，absolute DONE。 | 成立（1 次，Chrome Beta） |
| 空串 editor 删除得到 UNKNOWN，batch 停止，没有倒填 | 【原件】`044-…/stdout.bin`：第二步 UNKNOWN，原因是 `Empty read-back has no character witness`；after 长度 0，exit 3。 | 如实。我认为这个 UNKNOWN 本可以是 DONE（D4）。 |
| IME 字母键默认预测 | 【原件】`004-…/stdout.bin`：`inputTarget.uid=10109`、`source=windowInfos`、`packageCandidates=["com.android.inputmethod.latin"]`，前缀 [1,2]。 | 成立（API34，1 次）。MuMu 上因为 IME 没有显示，实施方如实写了“未验”。 |
| 自身窗口：默认路径与 `--expect` 自身包都在发送前拒绝 | 【原件】`063`：“App input cannot target the AndroidCtl App UID”，dispatchStarted=false；`064`：“management UI content is protected”。 | 成立。“拒绝后同一张图仍有资格”是实施方明说的推断，不是直接观测，标注如实。 |
| MuMu 静止页默认点击 | 【原件】`ST/14-input/runtime/mumu/commands/formal/005`：uid 10089，候选为 Chrome，前缀 [1,2]。构建是 `2beaadd` 的 release。 | 成立（1 次） |
| MuMu 横屏默认滑动（ROT270），以及 MuMu 自身拒绝 | 包里只有 `costs.json` 汇总：010 done/unsettled，013 not sent，原因是自身 UID。formal 009–014 的 stdout 和 raw **不在包内**。 | 核不到原件；汇总与声称一致 |
| “VD 点位实际落在 First，不是 Last”；VD 上 editor 只支持主屏；VD 原生 select→paste DONE | `text-api34/evidence-index.json` 列出了 formal 010–027，但 `PKG/EVIDENCE_INDEX.tsv` 里没有，包内也没有这些文件。`costs.json` 汇总显示 013 done 并已派发，015 not sent / displayUnsupported，017 两步 done。PKG/README 第 71 行承认“原 evidence-index 可能列出未附文件”。 | **核不到**。“落在 First”只能当作实施方自述。它的说法本身是诚实的（操作者用错了主屏坐标，承认不是 Last 命中），但没有原件可查。 |
| VD 网页 select→paste（regex101） | 原件 `055` 存在，实施方结果写两步 DONE。我没有逐字段展开，但 costs 与 result.md 一致。 | 部分核对 |
| 版本匹配 | 坐标线 release 构建绑定 `2beaadd`（`H_coordinate/*.json` 的 `productHead`），文字线绑定 `826c180`（`H_text/source-artifact-binding.json`）。最终 `4235af9` 在其后还改了 `VisualInput.kt`（162 行）和 `UiBackend.kt`（80 行），见 `git diff --stat 826c180 4235af9`。最终 APK 没有重跑文字和坐标矩阵。 | 应当如实标注为“阶段构建证据” |
| 真实任务里的通道用量 | 【重算 `W/count_cmds.py`】最终八臂和第 0 阶段八臂的 `cli-index.json` 中，文字类命令只有 `fill`，坐标类为 0（maps-new 有 1 次 screenshot）。 | 实施方没有拿这些数字支持输入面，这里只作为我自己的论据 |

**测试抽查**：
- `WindowInputTest.kt` 用 P5 真实旋转矩阵和手算的边界点做断言，是行为测试。
- `CurrentEditorInputTest` 大体在测行为，但第 40 行固化了 D1。
- `BorrowedPasteTest` 的第 159-163 行把“任意不同即 FAILED”固化了，没有“与动作前相同”这一类，对应 D2。
- 主机端 `test_text_input.py` 和 `test_input_keys.py` 我实跑了：51 passed（`venv4/bin/python -m pytest -p no:cacheprovider -q …`，5.4 秒）。它们覆盖参数、batch 停止和回执渲染，不涉及设备语义。
- 【实跑】`androidctl type hello --via paste` 返回 `not sent; Paste requires an explicit REF; code=arguments`，exit 2；`key ctrl+q` 和 `tap … --expect "bad pkg"` 都在本地被拒。

---

## 5. 对相关待决题的评价

实施方的四道待决题（virtual Activity 通知、动作后仍见旧页、页级动作字典、提交历史）都不在本专题内。

只有第 2 题与坐标输入有关：如果改变“动作后自动观察”，`tap/swipe` 之后的观察也会跟着变。本专题的零派发不观察（`01077a8`）与该题的任何选项都兼容，没有额外约束。

---

## 6. 需要用户裁决的新取舍

### 文字“任意 Unicode 局部插入”只保留一条通道

- **背景**：editor 和 paste 提供同一种能力。两者在真实任务里都是 0 次用量，各自带一套证明体系、一套回执和一套文档。
- **候选**：
  - **A. 保留 editor，删除借贴（推荐）**。
    - 得：不碰用户剪贴板；IC 绝对证明；修好 D1 后可信的 Compose 相对证明；少约 500 行。
    - 失：VD 上的任意 Unicode 局部插入，以及一次带自动恢复的粘贴。需要时可以显式组合 `clipboard set` 加 `action REF --action-id 32768`。
    - 仍需用户在 App 设置里开启无障碍服务，而且只支持主屏。
  - **B. 保留借贴，删除 editor**。
    - 得：VD 可用；不需要额外开启无障碍服务；少约 430 行和一个 AIDL。
    - 失：会借用剪贴板（覆盖层、隐私提示，worker 被杀时原剪贴板丢失）；多行装饰网页框里带换行的文本会核对失败。
  - **C. 两个都留（现状）**。能力最全，维护和文档最重。LLM 要先学会三选一。
- **推荐 A 的理由**：无人值守的 agent 默认不应动用户剪贴板。VD 是边缘场景，而且 VD 上 `fill` 仍然可用。editor 的证明质量更高。
- **剩余未知**：Compose 和网页之外的自绘编辑器（例如 WPS、Termux）上，两条通道都没有可用样本。

其它几项，包括坐标实现保留但冻结、`timeline` 隐藏、`select --text`、editor 观察开关统一、空串处理、D1/D2 的修正，都是实施方可以自行决定的实施细节。

---

## 7. 没能核实的内容

- D3（桌面壁纸导致定向注入目标不符）没有任何设备样本；Launcher3 是否关闭了壁纸触摸也没有核实。
- D5 的焦点与输入法连接竞态没有设备样本。
- VD 上 Contacts 的点击、editor 和 paste，以及 MuMu 的横屏和自身拒绝，原件不在包内。
- 最终 `4235af9` APK 上的坐标和文字行为没有设备运行；`3ef0ec6` 之后“零派发后同一张图再次输入”没有设备样本。
- Android 16 上没有任何设备证据。对 16 的判断只来自源码签名与常量比对。
- pinch、drag 和 timeline 在设备上跨窗口的表现（第二指落在另一个 UID 时目标不符）没有样本。
- 迟到粘贴（D2）的真实时序没有样本。
