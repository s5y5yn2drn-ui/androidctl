# 第六轮专题核查：截图、标注、区域图、坐标触摸、多指手势与"不碰 AndroidCtl 自己"

核查对象：产品 `fb75afc`（树与被实测的 `a27fe5aa` 相同），独立 docs `66bb380a`。设备证据全部来自实施方在 Android Studio API 34 模拟器（720×1280 屏）上的原件，没有真机。我的脚本在 `S/agents9/69/`（`S` = `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad`）：`runs.py`（35 次计分运行的命令与图像统计），`imgtok.py`（看图那一步的输入增量）。下文 `ST` = `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`。

---

## 1. 结论摘要

**整体状态：这一块现在是干净的，不构成合入障碍。** 第五轮第 2 题（坐标触摸不再按接收方 UID 定向、删除预测和 `--expect`、派发前查一次任务表）落实正确，删得彻底；上一轮的桌面坐标被拒（A3）有真实原件证明已解决；标注标签位置（A5）已修，真实图可读、编号与树一致；六项能力冒烟中与本专题有关的五项都是"功能真的生效"，不是只看回执。

最重要的发现：

1. **A3 已解决，有原件。** 桌面上 `tap 274 860` 打开了 Gallery（`ST/runtime/api34/commands/2343`、`2344`），`drag` 把 Gallery 图标从原格拖到右上空格再拖回（`2347`–`2350`，我看了前后两张图，图标确实移动了）。回执里 `inputTarget.uid=-1`。【原件】
2. **自身保护如实做到了它声称的，但不是"任何一层"。** 检查读的是目标屏上 `isVisible` 任务的 `baseActivity`、`topActivity`、`realActivity`、`origActivity`、`baseIntent.component`、`topActivityInfo.packageName` 这六个字段（`AppInput.kt:73-82`，`ActivityTasks.kt:39-41`）。AndroidCtl 自己的任务、以及从管理页打开的无障碍/电池设置页都能挡住（设备原件 `2353`、`2360`、`2367`，`dumpsys` 证实任务根是 AndroidCtl）。挡不住的是"外部 App 任务中间夹着 AndroidCtl"这一种栈，实施方自己找到了这个反例，指引和内部文档都照实写了限制。我到 AOSP 14/15/16 核过字段含义，结论一致；另外确认 shell 有 `REAL_GET_TASKS`，所以不会触发系统的 `trimIneffectiveInfo` 字段抹除。【源码＋原件】
3. **这个缺口可以用一行清单改动堵上**：`MainActivity` 设为 `launchMode="singleTask"`（或在 `onCreate` 里不是任务根就带 `NEW_TASK` 重开自己）。这样 AndroidCtl 永远是自己任务的根，从它打开的设置页也都在这个任务里，"任务根是 AndroidCtl"就成了真保证，文档里那段例外说明可以删掉。实施方以"改变交互"为由拒绝了改清单，我认为理由不充分，但这只是可选项。【源码＋推断】
4. **`click --touch` 现在有一个真实的新风险**：不定向以后，如果节点中心被输入法、悬浮窗或通知横幅盖住，触摸会落到上层窗口（例如按到键盘上的某个键）。图像手势没有这个问题，因为调用者看过图，而且派发前会和截图时逐窗比较；`--touch` 不要求截图，调用者看不到遮挡。用已经读到的无障碍窗口列表（层级和边界）检查"中心点是否被更高层窗口覆盖"，大约 15 行就能补上。可选，但比自身保护更常见。【源码＋推断】
5. **派发前的逐窗比较（上一轮 `VisualInput.kt:295-307`）完整保留**，现在是"截图之后屏幕变了"的唯一防线，代价低，应当保留。【源码】
6. **这一轮的 35 次计分运行里，候选一次坐标手势都没用**（20 次候选运行中 `tap/swipe/drag/pinch/timeline/click --touch` 为 0），看图共 9 次（GitHub 0、Feeder 7、后台屏 Contacts 0、Maps 2）；基线 15 次运行看图 59 次、坐标点击 12 次。图像和坐标在候选里是兜底能力，成本占比很小。【重算】

**是否构成合入障碍：不构成。** 上面第 3、4 条和第 3 节几项是可选整理，不改对外契约。

---

## 2. 确定缺陷

没有必须修的缺陷。

### 可选

**O1 `click --touch` 不检查落点是否被别的窗口覆盖（中低）**

- 触发：`click REF --touch` 的节点中心被更高层的窗口盖住，例如输入法、悬浮窗、通知横幅、权限对话框。
- 位置：`VisualInput.kt:325-334`（`touch` 计算形状、查包名，然后 `inject`），`AppInput.kt:135-158`（`nodeTouchPoint` 只检查节点 visible/enabled、尺寸为正、中心在屏内）。无障碍的 `visible` 和 `boundsInScreen` 不考虑其它窗口遮挡。
- 影响：触摸按普通注入送给落点处最上层的窗口。原来的 UID 定向会在这种情况下拒绝（遮挡窗口属于别的 UID），按第 2 题删掉以后就没有了。图像手势没有这个问题，因为调用者看过图，派发前还会逐窗比较。`--touch` 不要求截图，调用者看不到遮挡。
- 最小修正：`touch` 里已经调用 `geometry(run)`，拿到了目标屏全部无障碍窗口的 `layer` 和 `bounds`。中心点落在 layer 比节点所在窗口更高的窗口内就拒绝（`notReady` 或 `occluded`，未发出）。约 15 行加一个 JVM 用例。
- 确定程度：读源码推出；没有设备复现。这一轮的计分运行里 `--touch` 用了 0 次，频率低。

**O2 外部任务中间层缺口（低），一行清单改动可以堵上**

- 触发：别的前台 App 用显式组件、不带 `NEW_TASK` 启动 AndroidCtl 的 `MainActivity`（清单 `AndroidManifest.xml:16-21` 是 exported，没有设 launchMode），用户再从管理页打开设置。栈变成"外部根 → AndroidCtl → Settings"，六个受检字段都不是 AndroidCtl。实施方在 `ST/r5-self-protection-boundary.md` 里用 AOSP 源码论证了这条路径，我核对后认为成立。
- 实际影响：很小。管理页带 `FLAG_SECURE`（`MainActivity.kt:93`），截图是黑的（我看了 `ST/runtime/api34/images/r5-input-e6a/management-guard-before.jpg`，整屏只有导航栏）；`open` 拒绝启动 AndroidCtl；这条路径要外部 App 主动嵌入。
- 最小修正：`MainActivity` 设 `android:launchMode="singleTask"`（或在 `onCreate` 判断 `!isTaskRoot` 时带 `NEW_TASK` 重启自己并 finish）。管理页于是永远是自己任务的根，从它打开的设置页仍压在同一任务里（`MainActivity.kt:178,186` 不带 `NEW_TASK`），`baseActivity` 和 `realActivity` 必定是 AndroidCtl。代价是从桌面图标重新进入时，管理页上面的设置页会被清掉，这对管理页无所谓。做了以后，`images.md:137-142` 和 `reference/ui.md:290` 里关于中间层的例外可以删掉。
- 确定程度：AOSP 启动规则的推断，没有设备验证。

**O3 单独执行 `screenshot --marks` 仍然接受、但永远不标（低，上一轮 51 号报告 D2 的残留）**

- 【实跑】`androidctl screenshot --help` 现在写明 "Standalone screenshot has no node tree to label"，指引 `images.md:25-27` 也写了，说明已经如实。但这个选项在这条命令上没有任何用处，只是一个陷阱。在 host 端对不带树的 `screenshot` 直接拒绝 `--marks` 并给出 `snapshot --screenshot --marks` 的写法即可，约 5 行。

**O4 默认文本输出里印出了内部的 `shape` 几何（低）**

- 【原件】`snapshot --screenshot` 的默认文本来源行带完整的 `image` 元数据，其中 `shape`（逐窗 id、类型、层、边界、变换矩阵、放大状态）约 450 字符，整个 image 段约 690–860 字符（`ST/runtime/api34/commands/2342`、`2373` 的 stdout；计分运行 `r5-s1-core-candidate-01` 同样）。`shape` 只供 worker 自己做旧图判断，调用者不需要。默认文本里省掉 `shape`、`at`、`finishedMs`、`connection`、`mappingError:null`，每张图约省 150 token，JSON 保留不动。

---

## 3. 设计与简化

### 3.1 坐标输入全链（逐命令）

| 环节 | 现状 | 评价 |
| --- | --- | --- |
| 坐标属于哪张图 | 本会话租约下**最近一次实际交付的完整图**（`VisualInput.kt:285-294`）。`--image ID` 只是"必须仍是这张"的守卫，不能选旧图。区域图（`r` 开头）不能作输入帧，区域坐标也不是手势坐标（`images.md:57-61`）。 | 清楚。调用者不会把旧图坐标用到新屏上。 |
| 什么时候算过期 | ①超过 120 秒（`:291`，`IMAGE_TTL_MS=120000`）；②任何可能已派发的动作之后（`UiBackend.kt:169,173,257,261`）；③新采样的树签名变了（`:52-57`）；④派发前重读的显示（尺寸、旋转、用户、亮屏、锁屏）、全部无障碍窗口（id、类型、层、active、focused、边界、变换）、放大状态和截图时逐字不同（`:295-307`）；⑤连接代次变了。 | 第④条就是上一轮提到的窗口比较，**完整保留**。删掉 UID 定向以后，它是"截图后又弹出窗口"的唯一防线，而且只用公开数据、代价低。偏严（焦点切换也算），但动作带 `--screenshot` 就会拿到新的合格图，额外成本接近零。保留。 |
| 缩放换算 | 默认 JPEG 最长边超过 1280 才等比缩小（`ImageCoordinates.kt:14-22`），PNG 为原尺寸。手势点先在交付图范围内验证，再按 `x*sourceWidth/width`、`y*sourceHeight/height` 换算成浮点显示坐标（`PointerPlan.kt:41-64`）。 | 正确。取的是交付像素的左上角而不是中心，在 1080 宽的屏上最多偏 1.9 个显示像素，可以忽略。主机端只接受整数（【实跑】`tap 10.5 20` 被拒）。 |
| 旋转 | 截图前后的形状必须一致，位图尺寸必须等于逻辑显示尺寸，`mappingKnown` 才为 true（`:239-248`）；派发时如果旋转变了，形状比较会判旧图。 | 宁可拒绝也不点错，方向对。横屏没有设备证据，【需设备】。 |
| 后台屏 | 截图取自工作屏的流（`:181-187`）；形状、窗口、放大状态都按目标 displayId 读；每个 MotionEvent 都设 displayId（`PointerGesture.kt:77`）。 | `pinch`/`timeline` 在 display 2 上实际生效（见 §4）。 |
| 派发前检查 | 当前图合格 → 形状一致 → `inputReady`（亮屏、解锁；普通 swipe 允许划开不安全的锁屏）→ 计划换算 → **任务表自身检查**（`:341-344`）→ 每个事件前 `targetInputReady` 和 `beginDispatch`。 | 查询失败或命中自身都在 `beginDispatch` 之前抛出，回执为未发出（`2353` 原件 `dispatchStarted=false`）。 |
| 注入方式 | 隐藏接口 `IInputManager.injectInputEventToTarget`，`targetUid=Process.INVALID_UID`（`AppInput.kt:64-70,81`）。按下和抬起用 `WAIT_FOR_RESULT`，移动用 `ASYNC`；异常或未确认时补发一次 CANCEL（`PointerGesture.kt:146-148,269-296`）。 | 等价于普通注入。保留隐藏接口只是因为按键路径还要用定向版本，触摸复用同一个发送器，合理。 |
| 之后的观察 | 可能已派发就跑 `postAction.settle` 并附上观察变化（`:317-321`）。 | 与其它动作一致。 |
| `click --touch` | 新鲜节点身份 → 节点包名经 PM 查 UID，不能是 AndroidCtl（`:332`）→ 中心点 → 同一个 `inject`（同样做任务表检查）。不要求截图。 | 缺遮挡检查，见 O1。 |

### 3.2 自身保护的实际能力（用户关心的"有限字段"）

**AOSP 核对**（`S/agents3/work14/src/android-{14,15,16}.0.0_r1/.../wm/Task.java`，`S/agents3/work15/aosp/android-{14,16}.0.0_r1/RunningTasks.java`）：

- `fillTaskInfo` 里：`topActivity` 和 `topActivityInfo` 来自栈顶未 finishing 的 Activity；`realActivity` 和 `origActivity` 是任务保存的启动身份；`baseIntent` 是根 Intent；`baseActivity` 是最底下的 Activity；`isVisible = hasVisibleChildren()`，即任务里有任何一个 `ActivityRecord.mVisible` 为 true（14：`Task.java:3445, 4374-4380`；15：`3471`；16：`3418`）。三个版本的字段含义相同。没有任何字段列出中间的 Activity。
- `RunningTasks.createRunningTaskInfo` 只在调用方**没有** `FLAG_ALLOWED` 时才调用 `trimIneffectiveInfo`，后者会在顶部 Activity 与任务 UID 不同时把 `topActivity` 和 `topActivityInfo.packageName` 抹成空串（14：`RunningTasks.java:175-180`；`Task.java:3474-3516`）。worker 是 shell，先检查了 `REAL_GET_TASKS`（`ActivityTasks.kt:52-54`），所以拿到的是未抹除的字段。这一点实施方没写，我补核后确认它不构成缺口。
- 16 把可见任务的排序依据改成 `isVisibleRequested`（`RunningTasks.java:160`），不影响 `isVisible` 字段本身。

**能挡住什么、挡不住什么：**

| 情形 | 结果 | 依据 |
| --- | --- | --- |
| 管理页本身在前台 | 拒绝 | `2353` 原件 |
| 从管理页打开的无障碍设置、电池优化设置 | 拒绝（任务根是 AndroidCtl） | `2360`、`2367` 原件；`2357` 的 `dumpsys` 显示 task 70 的 Hist #0 是 `MainActivity`，#1 是 Settings |
| 管理页授权弹出的系统权限对话框（与请求方同任务） | 拒绝 | 推断 |
| AndroidCtl 任务存在但不可见 | 放行 | `2351` 显示 AndroidCtl 任务在 `2343` 桌面点击之前已经存在（"its current task has been brought to the front"），而 `2343` 正常派发 |
| 外部任务中间夹 AndroidCtl、顶上是 Settings | **漏** | 实施方反例；O2 可以堵上 |
| 对话框主题或半透明的 AndroidCtl Activity 叠在别的任务上 | 它会在自己的任务里（启动方式决定），任务可见就拒绝；嵌入外部任务时同上一行 | 推断 |
| 画中画 | AndroidCtl 不支持画中画，不适用 | 清单 |
| 分屏或自由窗口里 AndroidCtl 占一侧 | 整个显示的坐标输入都被拒（**误拒**，即使点的是另一侧） | 源码；可以接受 |
| 通知栏展开盖在 AndroidCtl 任务上 | 任务仍可见，被拒（误拒，无害） | 推断 |
| AndroidCtl 的通知（点开管理页） | 通知栏是 SystemUI 窗口，不被挡；点开以后图已失效，下一次手势需要新图，新图上的任务检查会拒绝 | `GatewayService.kt:45,112`；推断 |
| Shizuku 管理界面、它的授权对话框 | 不在保护范围（是另一个包） | 实施方明确不纳入，合理：worker 运行时已经授权 |
| AndroidCtl 刚退到后台、切换动画还没结束 | `mVisible` 可能仍为 true，短暂误拒 | AOSP；调用节奏下几乎遇不到 |
| 检查之后、长手势进行中 AndroidCtl 才出现 | 不再检查（只查一次） | 源码；按用户裁决"派发前查一次" |

**"读落点处窗口的包名"会不会更准？** 不会更好。它看不到从管理页打开的设置页（那是 Settings 的包），而用户明确要挡这些页面；在"外部根 → AndroidCtl → Settings"的栈里它同样漏。任务表方案加上 O2 的 `singleTask`，比窗口方案更贴合用户的目标，查询成本也只是每次手势一次 Binder 调用。

**指引和内部文档是否如实：如实。** `images.md:137-142` 写的是"通过 base/top、real/original、base Intent 或 top ActivityInfo 识别 AndroidCtl 的可见任务"，"无法识别别的 App 任务里任意中间层的 Activity"，"这不是接收方或命中保证"；`reference/ui.md:290` 和 `architecture.md:162` 措辞一致；没有把有限字段写成"任意一层"。

**删除 UID 定向后点到别的应用的风险：** 对图像手势来说，截图到派发之间的窗口变化已经被逐窗比较拦下，剩下的只有"比较完到第一次按下"这几毫秒，调用者点到自己在图上看到的悬浮窗、输入法或系统栏，是调用者自己的选择，这本来就该由调用者承担，和 Playwright 按坐标点击一样。边缘滑动可能触发系统返回或回桌面手势，这和基线行为相同。真正多出来的风险只在 `click --touch`（见 O1）。这项保护也**不是安全边界**：调用者随时可以用 `androidctl shell` 执行 `input tap`（`ShellJob.kt:58` 以 shell 身份起进程）。它的定位是防手滑，按这个定位，现在的成本（约 10 行加一次查询）是相称的。

### 3.3 图像交付

| 声称 | 核对 | 结论 |
| --- | --- | --- |
| 默认 JPEG 质量 80，最长边 1280，不放大 | `VisualInput.kt:428`；`ImageCoordinates.kt:14-22`（`longest <= 1280` 原样返回） | 成立【源码】 |
| PNG 原尺寸 | 同上（`this == PNG` 原样返回） | 成立【源码】 |
| 标注用局部点阵字形，不改全局字体 | `ImageMarks.kt:172-220`（`RefGlyphs` 5×7 点阵，只覆盖 `e` 和数字，`Canvas.drawRect` 逐点画） | 成立【源码】 |
| 标签小目标优先 | `ImageMarks.kt:65-98`：按框面积升序选角，四个角都冲突时退回左上角，反序绘制让小目标的标签在最上面 | 成立【源码＋原件】 |
| 区域图是新采的原生像素 | `VisualInput.kt:131-179`：先核图的租约、连接、布局，再重新 `captureBitmap` 并裁剪，编码为 PNG | 成立【源码】 |

**标注的可读性（我实际看了图）：** `ST/runtime/api34/images/r5-input-e6a/gallery-photo-marks.jpg`（21 个标签）和 `r5-surface-c3/gallery-grid-marks.jpg`。

- 网格容器的标签（`e443563585`、`e819264818`）已经移到容器左下角，不再占第一张照片的左上角。A5 已修。
- 每张照片的标签在自己的左上角，黑底白字约 14 像素高，我能逐个读出。
- 工具栏里相邻按钮的标签会错开到下角（如 `e443563580`）。
- 编号与同次快照的树一致：`000029-…Gallery.txt` 里 `e443563592` 是第 0 行第 2 列的 `media_item_holder`，图上第三格幼牛也标着 `e443563592`。
- 副作用：编号是 10 位的真实 REF，标签约 120×20 像素，压住每张照片左上约 1/8 的内容；左下角的容器标签压住了最后一行第一张照片的底边。上一轮建议的"图内短序号"没有采纳，这是实施方的选择。指引写了"Labels can overlap or cover content"，可以接受。
- 真实图都是 720 宽。在 1080×2400 的真机上交付图是 576×1280，标签相对目标会更大。这只是推断，没有设备图。

**区域图会不会被误用：** 指引写得很明白（"captures new pixels … does not crop retained old pixels"，`images.md:50-55`）。真正的风险是动作之后用旧图坐标截区域，看到的是新内容；回执里的 `coordinateImageId` 和新的 `at` 能提示这一点。在 720 宽的屏上，区域图不比整图多任何细节（JPEG 本来就是原尺寸）；它的价值只在高分辨率屏缩到 1280 以后看细节，或者省 token 只看一块。上一轮 51 号报告已经指出，它是便利能力。REF 形式的区域图（按 REF 重新读取当前边界）比坐标形式多一段身份核对代码，属于可接受的复杂度。

**token 成本【重算，`runs.py`、`imgtok.py`】：** 35 次计分运行中，候选 20 次共看图 9 张（GitHub 0，Feeder 7，后台屏 Contacts 0，Maps 2），基线 15 次共 59 张（GitHub 44，Feeder 5，Maps 10）。候选看图那一步的输入增量是 1.7k–3.0k token，包含图和同一步的 CLI 文本；基线 GitHub 每步约 0.95k。720×1280 的一张图首次进入上下文大约不到 1k token。相对单次运行 36 万到 73 万的累计输入，图像占比只有几个百分点。四道题都不是"非看图不可"：候选在 GitHub 和 Contacts 上一张图都没看就完成了。

### 3.4 `timeline`、`pinch`、`drag` 是否合并

三者在设备端是同一个计划器和执行器（`PointerPlan.parse`），`drag` 和 `pinch` 只是主机端把参数拼成轨迹的约 45 行糖（`ui_commands.py:1500-1590`）。这一轮没有新证据：候选计分运行里三者使用次数都是 0，冒烟里各自正常工作。上一轮 C6 用户裁决保留 `drag`，**不重提**。

### 3.5 回看前几轮的决定

- **第五轮第 2 题（选 E 的变体）现在看是对的。** 删掉了约 480 行预测代码、编译桩和保留规则，桌面能力也恢复了（原件证实）。用户在追问第 4 问里选了"任务里任何一层"（A），而不是评审方建议的"读落点窗口"（C）。实施方遇到字段做不到"任何一层"时，没有为了字面遵守去造跟踪机制，而是如实收窄，并得到了用户"不把有限字段写成任意中间层的保证"的授权。这是正确的处理，不是钻牛角尖。剩下唯一的小遗憾是拒绝了一行清单改动（O2）。
- **第四轮第 6 项"每项能力配一个非它不可的任务"是这一块此前最大的牛角尖来源。** `ST/maps-gesture-gap.md` 为 Maps 旋转角度做了大量复核，`ST/capability-followups.md` 在 WPS、Launcher 和 Gallery 之间反复找"非拖拽不可"的题。第五轮第 3 题把它改成单次功能冒烟以后，这一轮的冒烟简洁、有效（见 §4），证明改得对。
- 上一轮评审方关于"派发前逐窗比较已经覆盖绝大多数变化"的判断，这一轮源码仍然成立。

### 3.6 与"类似 Playwright"定位对照

| 能力 | 判断 |
| --- | --- |
| 截图（JPEG 预览和 PNG 原图） | 必要的视觉兜底 |
| 按图坐标单指点击和滑动 | 必要（桌面、画布、无障碍信息缺失的界面） |
| 双指 `pinch` 和 `timeline` | 必要（地图、图片缩放），实现复用同一执行器，成本低 |
| 标注 | 有用的桥梁：把看到的东西对应到 REF，再走语义动作 |
| 区域图 | 便利能力，复杂度中等，可以保留 |
| 图的资格、120 秒 TTL、逐窗比较 | 必要的防误点，代价低 |
| 任务表自身检查 | 防手滑，成本相称；不要再往上加机制 |
| 手势回执里的细节（`keyIntervals`、`sampledOffsetMs`、`omitted.from/through`、`syncReturned`、`asyncQueued` 等） | 只在 JSON 里，默认文本已经精简（第五轮 B12 已做）。偏重，但不影响调用成本，不必再动 |

没有需要整块删除或重做的部分。

---

## 4. 证据审计

所有设备冒烟各做一次，API 34 Studio 模拟器，屏幕 720×1280。版本：桌面点击、拖拽、自身保护、标注、区域、结构化剪贴板在 `e6a26668`；`pinch`、`timeline`、`scroll-until` 在 `c3d78e64`。我核对了 `git diff e6a26668 a27fe5aa`：`VisualInput.kt`、`PointerPlan.kt`、`ImageRegion.kt` 只改了错误码命名（`staleImage` 改为 `stale`+`subject` 等），`AppInput.kt`、`PointerGesture.kt`、`ImageMarks.kt`、`ActivityTasks.kt` 没有改动。因此这些冒烟对最终版本的输入、图像路径行为有效。【源码】

| 实施方声称 | 原件 | 结论 |
| --- | --- | --- |
| 桌面坐标点击打开 App | `2342` 图（mappingKnown=true）；`2343` `tap 274 860` 结果 done，`uid=-1`；`2344` 独立快照显示 `com.android.gallery3d` | **成立**，功能生效 |
| 桌面拖动图标 | `2347` `drag 274 860 446 680 --duration 1200ms`，76/77 个事件返回，跳过 1 个 MOVE；我看了 `desktop-drag-before.jpg` 和 `desktop-drag-after.jpg`，Gallery 图标从 (274,860) 移到右上；`2349`/`2350` 拖回 | **成立**，功能生效 |
| 管理页和两个设置子页上坐标输入被拒 | `2353`、`2360`、`2367`：not sent，`code=protected`，`dispatchStarted=false`，前面都有 mappingKnown=true 的新图（`2352`、`2359`、`2366`）；`2357`/`2364` 的 `dumpsys` 显示任务根是 `MainActivity`、顶部是 Settings | **成立**。没有拿过期图冒充保护生效。实施方也没有把它推广到中间层 |
| `pinch` 放大、`timeline` 缩回 | `2624` 两指 53/53 事件，display 2，`uid=-1`；`2626` 52/53，跳过 1 个 MOVE。我看了 `calf-after-pinch.jpg`（幼牛放大成局部）和 `calf-after-timeline.jpg`（回到整张） | **成立**，像素层面生效。回执 `settle=unsettled`、无障碍内容 `unchanged` 被如实保留，没有拿 done 充当效果 |
| 按标注编号点开没有文字名的图片 | `2373` 标注图 21 个；`2374` `click e443563592`（语义点击，不是坐标点击）；`2375` 打开的正是第三格幼牛 | **成立**。注意这验证的是"标注到 REF 再语义点击"，不是"标注到坐标" |
| 对真实 App 的一块区域截图并读出内容 | `2376` `--region 500 430 700 650 --image i837193049` 得到 200×220 PNG；我看了 `calf-face-region.png`，幼牛的耳、眼、鼻清晰 | **成立**。但屏幕是 720 宽，区域图与整图分辨率相同，**没有验证到它的主要价值**（高分辨率屏缩图后看细节）。缩放映射在第五轮 MuMu（1080×1920 缩到 720×1280）有 559 张图的使用记录，映射代码此后没变 |
| 结构化剪贴板 | `2488` 在系统分享页点 Copy；`2489` `clipboard get` 得到 `text/plain`、1 项、`https://www.rfc-editor.org/info/rfc10021/`，`complete=true` | **成立**（不属本专题，顺带核对） |
| `scroll-until` 冒烟 | `2612` 在 100 项列表上 2 步找到第 13 行 | **部分成立**：冒烟跑在 `c3d78e64`，之后 `a27fe5a` 又改了 `ScrollContainer.kt`（U34 的"看到文字即成功"）。最终语义的设备证据来自固定任务中的自然使用（Feeder 5 次、Contacts 4 次、GitHub 2 次运行，共 13 次调用），不是那次冒烟。目标只在第 13 行，对"50 项以上长列表"的考验偏弱。细节留给相应专题 |
| 内部 review "MF0/O0" | 见上 | 我找到的 O1–O4 都是可选项，与它的结论不矛盾。但它没有检查 `--touch` 的遮挡，也没有评估清单改动 |

**其它审计点：**

- 失败和未知是否如实保留：`timeline` 跳过的 MOVE、`unsettled`、无障碍 `unchanged` 都保留了。`2354` 的 uiautomator 被 kill、`2469` 的 UNKNOWN 也写进了流水。【原件】
- 测试是验证行为还是复述实现：`AppInputTest` 的三个用例分别覆盖"任一受检字段命中即拒绝""查询失败零派发""不设 UID"，是行为断言；`ImageMarksTest` 断言生产代码实际使用的标签坐标和绘制顺序。【源码】host 端 `test_pointer_gestures.py` 11 项通过。【实跑】
- **缩放路径（交付尺寸不等于原生尺寸）这一轮没有设备证据**：本轮所有原件里 225 处都是 720×1280 对应 720×1280。【重算：在 `ST/runtime` 和 `ST/efficiency` 里 grep】

---

## 5. 需要用户裁决的新取舍

没有。O1–O4 都由实施方自行决定即可。O2 推翻了实施方"不改清单启动模式"的表态，但不改变对外契约，只让一条已经披露的例外消失。如果用户希望对这条例外表态，候选项是：

- A：保持现状，接受例外（零成本）。
- B：`MainActivity` 改 `singleTask`，删掉例外说明（1 行清单改动，加文档删减，从桌面重新进入时清掉管理页上面的设置页）。
- C：删除整个自身检查（不推荐：它挡住设置子页有真实价值，而且代价很低）。

我推荐 B，交给实施方定。

---

## 6. 有判别价值的补验

1. **一台 1080×2400 的真机或模拟器上，做一次 `snapshot --screenshot --marks` 加 `tap`。** 这验证交付图 576×1280 时的坐标换算和标签可读性。如果标签遮挡严重，"图内短序号"这条建议就值得重提；如果点偏，说明换算有问题，这一块就不能算干净。
2. **（只有在 O1 被质疑时才做）**输入法弹出、目标按钮被盖住时执行 `click REF --touch`，看是否打到键盘。打到了就把 O1 升为"应当修"。
3. 实施 O2 以后，用 `am start -n` 从别的 App 任务里启动管理页（不带 `NEW_TASK`），确认它另起任务。

---

## 7. 没能核实的内容

- 所有设备行为都来自 API 34 模拟器；横屏、真机、厂商 ROM（尤其是设置页是否另起任务）都未验证。
- `click --touch` 的遮挡问题（O1）是源码推断，没有设备复现。
- `TaskInfo.isVisible` 在切换动画期间的实际误拒时长未测。
- 图像 token 的精确数值无法从累计 usage 中完全分离，§3.3 的数字是包含同一步文本输出的上限估计。
- 我没有逐项重跑 JVM 测试（没有 Android 构建环境）。
