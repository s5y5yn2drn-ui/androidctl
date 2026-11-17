# 反驳式复核：主报告〔单方〕条目与两处分歧（C1–C7、J1、J2）

复核对象：`/home/user/androidctl/review/androidctl-candidate-route-review-3f45af6.md`（下称主报告），以及底稿 01/02/03/06/08。
方法：每条都先尝试证明它是错的。依据只来自三处：候选源码（REPO = 3f45af6，基线 7d5be70，只用 `git show` 读取）、评审包原件（PKG/ultragoal/2026-11-11-next-improve，下称 U）、上游一手源码。上游源码下载到 `agents2/work09/`：AOSP android-14.0.0_r1、15.0.0_r1、16.0.0_r1 的 `InputDispatcher.cpp`、`NotificationManagerService.java`、`ManagedServices.java`、`ClipboardService.java`，以及 termux-app tag v0.118.3 的 `TermuxTerminalViewClient.java`、`TerminalView.java`。全程只读，没有执行包内任何脚本，也没有设备。

证据标签：【实测】指包内原件里的真实运行；【源码】指候选或基线源码；【上游】指上面列出的一手上游源码；【推断】；【需设备】。

---

## 1. 总表

| # | 原主张（主报告位置） | 裁定 | 证据类别 | 主报告应改写为（可直接替换） |
|---|---|---|---|---|
| C1 | E9 必要性只有 J67 一份样本；J67（3157ms/3869/partial）与 J54 公开单节点路径（约 1s/4214/full）同设备同页面，相差约 3 倍且没有解释（R5、E2） | **需收窄**。"证据不足"仍成立，"没有解释"不成立 | 实测＋源码；**需设备**才能定论 | "J67 是新 worker 的**首个**采样。同一批原件里，E9 的首个采样也比随后的热采样慢 1.8–4.5 倍（J68：1378ms，之后 305–761ms；J54：1114/834ms，之后 600/352ms）。J54 公开探针测的是热状态，每个节点做的 IPC 与 HEAD single 相同（各 1 次 getChild，cache 关），读的 30 个字段也相同。所以约 3 倍的差距最合理的解释是冷/热差加上两组实验相隔一次重启，不是 single 本身每个节点多做了工作。缺的是 HEAD single 的**热**样本，以及冷首采在真实使用中出现得有多频繁。" P4 要改成"同一 worker 内：冷首采 1 次，然后热采样 single/E9 交替各 ≥3 次"，P4 仍是唯一出路 |
| C2 | IC 可以由 `clipboard set` + `key ctrl+alt+v` 替代（R3、E3） | **需收窄**。替代路径在源码层面逐环成立，但端到端从未跑过；"Chrome 同一字段 fill 实测能写"只测过 fill 空串 | 源码＋上游＋实测（局部）；**需设备** | "Termux 0.118.3 的 Ctrl+Alt+V→doPaste 不依赖物理键盘，只受 `disable-hardware-keyboard-shortcuts` 属性控制（默认关）。候选的 `key ctrl+alt+v` 能发出正确的 meta 序列，并经全局键路径送达 Termux：同一台 MuMu、同一版本 Termux 已用全局键实测输入过 ASCII。Termux 有焦点，按平台规则可以读剪贴板。端到端仍需一次设备验证。Chrome 那个 HTML 字段只实测过 `fill ''`（清空，done/matches）。" |
| C3 | 坐标手势默认按操作窗 UID 定向，会拒绝点软键盘；photopicker 实测必须加 `--expect`（T4） | **软键盘部分成立**（源码加上游逐行可证）；**"实测必须 --expect"夸大**；但包内有一条主报告没引用、性质相同的**真实**拒绝 | 源码＋上游＋实测（同类）；软键盘本身**需设备** | "operatingWindow 明确排除 inputMethod 窗口（`UiScope.kt:50-52`），所以键盘弹出时，无 `--expect` 的坐标 tap 会以 App 的 UID 定向。AOSP14 InputDispatcher 在触点落到 IME 窗口（owner 是 IME 的 UID）时返回 TARGET_MISMATCH，结果是 not sent。photopicker 那次运行一开始就带了 `--expect`，没有做过不带的对照。不过同一阶段里，按 input-focus UID 定向的 BACK 确实因为嵌入窗 owner 不同被平台拒绝了，项目随后把全局键改成了不定向（global-key-fix），坐标手势却保留了同样的定向。" |
| C4 | 平台每次更新通知都会重置 postTime，写操作守卫因此反复误拒（T6） | **成立** | 上游＋源码 | 原文保留，补上依据："AOSP 14/15/16 的 `enqueueNotificationInternal` 每次 notify 都以 `System.currentTimeMillis()` 新建 SBN；候选比较的是 `sbn.postTime`（`NotificationActions.kt:147,245`），不是 `Notification.when`。候选自己的 Gateway 通知每 2 秒 notify 一次，所以它自身的 postTime 每 2 秒变一次。" |
| C5 | listener 注销失败会挡住全部 UI、新 shell/file、租约到期和正常 Stop；会把 done 回执改成非零 exit；AOSP 上几乎不会失败，worker 死亡后自动清除（D5） | **方向成立，但要补两处、改一处行号** | 源码＋上游 | "……挡住所有 UI、新 shell/file、租约到期和正常 Stop；本机 Force executor exit 仍可用，已在跑的 shell/传输可读、可取消。这套 fail-closed 模式是基线已有的（UiAutomation 断开失败，`closeError`＋`sessions.closing`），候选只是把通知 listener 接了进来，并新增'done 回执 exit 改为 1'（`NotificationOperations.kt:111`，不是 :185-195）。AOSP `unregisterService` 不做权限检查，listener 注册时已 linkToDeath，worker 死亡后由 binderDied 移除。" |
| C6 | "8,377,092B / 127 唯一 REF"是任务自建的 fixture，交付稿措辞容易被读成真实页面（§8 第 4 点） | **成立**（措辞判断偏重，可以略收） | 实测原件＋源码 | "这是任务自建包 `org.androidctl.research.dense` 的 row-budget 模式：一个 64KiB 行标签被复制进 123 个后代的 row.label。内部 review（final-host-evidence:60）写明了'不是发行 App 业务'，但交付稿 candidate-final.md:34 那一句没有写。它暴露的是投影把 row.label 逐节点放大的真实机制，更适合直接限制 row.label 的长度，而不能证明真实页面需要最终定稿机制。" |
| C7 | B 处置表写了"采用/调整采用"，但 3f45af6 里找不到实现：next:、staleRef 候选、不可信内容提示、应用笔记、WebView 折叠（§8 第 6 点） | **需收窄**：五项里三项属实；`next:` 和 WebView 两项需要改写；"用户可能以为已落地"的定性偏重 | 源码＋包内文档 | 见 §2 C7 的逐项表。总述改为："b-disposition 开头就声明'处置建议本身不是用户批准'，它也不是交付清单。问题在于没有一份处置与落地的对账表：不可信内容提示、staleRef 候选、应用笔记三项为 0；非 done 的下一步提示以 reason 尾句的形式存在（多数是基线已有），没有统一的 next 字段；WebView 没有专门的折叠，只有通用的 nameConsumed 与仅可见投影带来的部分等价效果。" |
| J1 | 后台屏"协议必填种类"是否保留（R2 裁断采纳 02：保留设备侧必填，只改主机写入时点） | **维持主报告裁断，补一条**：显式 `session close`（以及 `display select main`）时清除主机偏好；`status` 显示偏好。若采纳 R2-C，首次合入不带 kind 字段，这个问题就延后 | 源码推演＋设计建议 | 在 R2 的"关于机制本身"段末加一句："主机偏好在显式 `session close` 时清除（close 已销毁后台屏，继续后台工作本来就要重新 create，所以不损失保护），由此去掉 T7 里大部分遗留 virtual 的情形。" |
| J2 | IC 与借贴删哪个（R3 推荐：删借贴；IC 不进首次合入，倾向删除；ctrl+alt+v 不成立则改 C） | **同意主报告的 B→C 条件结构**，改正两处依据 | 实测＋源码＋上游；**需设备** | R3 的依据里："选区可用 AX SET_SELECTION"改为"现有 `key REF shift+left/right` 可以做相对选区；做绝对偏移选区需要新增一个类型化命令，因为 generic `action` 拒绝带参数的标准动作（`NodeActions.kt:89-91`）"；"Chrome 同一字段 fill 实测能写"改为"只实测过 fill 空串"。 |

---

## 2. 逐条详述

### C1　E9 证据不足：J67 与 J54 的约 3 倍差距

**两条链每个节点实际做什么。**

- **HEAD single（J67 实际运行的版本）**：J67 跑的不是 3f45af6，而是**基线 7d5be70** 加 `single-8192.patch`。补丁只改了 COLLECT_MS 1500→3000、NODES 1024→8192、树字节 4→8MiB，另加了计数日志（`U/prototypes/observation/architecture-single-8192/single-8192.patch`）。
  - 基线 `TreeCollector.addChildren` 对每个孩子调用一次 `NodeAccess.child`。在 API≥33 上，这就是 `node.getChild(index)`（`git show 7d5be70:…/NodeAccess.kt:11`）。基线 `AccessibilityConnection` 把 UiAutomation 的 cache 设为 disabled（`…/AccessibilityConnection.kt:29-37`）。性能报告 §3 已核 AIC A14–A16：cache 关闭时会清掉 PREFETCH 标志，所以每个孩子只有 1 次 Binder 往返。
  - `fresh()` 只在 API<33 时调用 refresh，API≥33 不调用。
  - `NodeFacts.read` 读的全是已经 parcel 到本地的 ANI 字段：text、actionList、collectionInfo/ItemInfo、rangeInfo、bounds 等。没有 extras、availableExtraData、parent 查询或窗口变换。唯一的反射 `sourceNodeId` 是本地调用。
  - HEAD 每个节点多出来的只有 CPU 工作：`facts.toString().toByteArray()` 用来计字节，外加 JSON 构建。
- **J54 公开探针**：`PublicProbeService.java:146,181` 用 `getRoot(0)` 和 `getChild(i,0)`，cache 关，也是每个孩子 1 次 IPC。`e17-j54-reviewed.json` 的 `commonFields` 正好就是 NodeFacts 那 30 个字段。探针的 Meter 自己也算进了 JSON 构建与旧索引映射（性能报告 §1）。

**结论：每个节点的 Binder 往返数相同，字段集合相同。** 差异只在宿主（UiA shell worker 对比 App 内的普通服务）、采样时点，以及是否冷启动。【源码】

**被忽略的解释：冷首采。** 下面的数都来自原件 `e17-j54-reviewed.json` 和 `j68-e9-reviewed.json.allE9Collections`。

| 样本 | 是否新 worker 首采 | 节点 | 采集 ms |
|---|---|---|---|
| J54 event0 U0（E9 研究链） | 是（回执 notice "New worker"） | 4214 | 1114 |
| J54 event0 U1 | 否 | 4214 | 600 |
| J54 relevant U0 / U1 | 首采 / 非首采 | 4214 | 834 / 352 |
| J54 公开探针 P1/P2/Q1/Q2（排在 U0 之后） | 否（Chrome 已被 U0 预热） | 4214 | workWall 1137/1098/967/865 |
| **J67 single** | **是**（cd556542，connection1，唯一一次采样） | 3869 partial | **3157** |
| J68 E9 预算门 | 是（50d39bb6，connection1） | 4226 | 1378 |
| J68 E9 同一 worker 随后各次 | 否 | 4226/4227/4237 | 743、305、463、504、1167、761 |

- E9 自己的冷/热比是 1.8–4.5 倍。J67 对 J54 公开探针约 3.2 倍，正好落在这个区间里。
- 同处冷态时，single 对 E9 是 3157/1378≈2.3 倍；同处热态时，J54 公开探针对 E9 是 865–1137 / 352–600≈1.8–2.7 倍。两个时代的模式比例一致。
- 所以说"3 倍无解释"不对。更合理的说法是：J67 只测到了冷首采；公开探针的数字是热态；single 本身比 E9 慢约 2–2.7 倍，这与其它样本一致。
- 冷态代价从哪里来（Chrome 侧首次建立 a11y 树、worker 侧首次连接，还是重启后的设备状态），原件分不出来。【推断】

**n>1 的其它 single 样本**：
- `research/observation/e9-reviewed.json` 中 e9-single-chrome1/2 为 511/355ms（1024 节点封顶）。
- `chrome-a/b-comparison.json` 中 single 为 204/189/177/241ms（1024 封顶）。
- 折算每个节点 0.17–0.50ms，外推到 4225 节点约 0.7–2.1s，低于 3000ms。J67 每个节点 0.82ms，高于这个区间，同样与冷态解释相符。
- 但这些样本的页面和设备与 J67 不同，我也没有逐一核实设备，只能当旁证。【实测，旁证】

**性能报告那句"J54 公开子集约 1s 不能替代 HEAD 完整链的 J67 阴性"背后有没有具体解释？**
- 它在 §1 和 §5 列了宿主不同、计时边界不同（public 的 work 不含 UiSnapshot 组装与交付），却同时承认 J67 与公开探针"落同一 AIC"。
- 它没有提到冷/热，也没有指出 J67 是唯一冷样本。所以这句话是一个边界声明，不是对差距的解释。
- 底稿 01 注意到了"E9 自身两样本相差 2.4 倍"，但没有把它和首采联系起来。

**裁定**：
- "E9 证据不足"保持，但理由要换：不是"3 倍悬空"，而是"只有冷首采的 single 样本，没有热态 single，也不知道冷首采在实际使用中占多少"。
- 如果热态 single 约 1–1.5s，E9 只换来"新 worker 首次读超大页时完整"这一项，partial 也只发生在不可见区域。删除 E9 的理由会明显变强。
- P4 仍是唯一能定论的路径，设计改为：同一新 worker 先冷采 1 次，然后 single、E9（可再加上 cache 临时启用方案）各热采 ≥3 次交替进行；冷采也按 single/E9 各换一个新 worker 做。【**需设备**】

### C2　IC 可由 `clipboard set` + `key ctrl+alt+v` 替代

**(a) 候选能否正确发出 ctrl+alt+v。【源码】**
- 键名：`v` 在允许集合里，主机侧 `input_keys.py:7-25` 与 Android 侧 `KeyPlan.kt:19-38` 都有。可用键是固定的 17 个（编辑键加 a/c/v/x/y/z），不是任意字母。修饰键是 ctrl/alt/shift。
- 事件序列：`KeyPlan.chord`（`KeyPlan.kt:66-87`）依次发 CTRL_LEFT DOWN(meta=CTRL)、ALT_LEFT DOWN(meta=CTRL|ALT)、V DOWN/UP(meta=CTRL|ALT)、ALT UP(meta=CTRL)、CTRL UP(meta=0)。meta 位包含 `META_CTRL_ON|META_CTRL_LEFT_ON` 和 `META_ALT_ON|META_ALT_LEFT_ON`（`:106-122`）。
- 事件属性：deviceId 为 `VIRTUAL_KEYBOARD`，source 为 KEYBOARD，flag 为 FROM_SYSTEM（`KeyPlan.kt:7-16`）。失败时 `KeySequence.release` 会补发所有已按下键的 UP（`KeySequence.kt:155-181`）。
- 无 REF 时：`UiBackend.key` 交给 `SystemActions.perform`，`keyInputTarget` 走 `requireExternalInputWindow`（`SystemActions.kt:199-212`）。条件只有两个：恰好一个有焦点的窗口，且它的根包不是 AndroidCtl。满足后用 `INVALID_UID` 不定向发送（`:130`）。
- 带 REF 时：要求节点 visible、enabled、focused（`UiAction.kt:87-92`），**不要求 editable**。Termux 的 `terminal_view` 有焦点时可以作为 REF（stage 35 记录里它是 focused/visible/enabled，editable=false），此时按 Termux 的 UID 定向发送。
- 全局键路径对 Termux 的可用性**已实测**：`U/stages/25-text-keys/runtime-mumu/result.md:15-23`。同一台 MuMu、Termux 0.118.3（code1002），无 REF 的 `type --via keys` 共 18 个事件全部返回，整图核验输入正确。

**(b) Termux 上游。【上游：termux-app v0.118.3】**
- `TerminalView.onKeyDown:753-762` 先交给 `mClient.onKeyDown`。
- `TermuxTerminalViewClient.onKeyDown:227-269`：
  - `handleVirtualKeys` 只处理音量键。
  - 条件是 `!areHardwareKeyboardShortcutsDisabled() && e.isCtrlPressed() && e.isAltPressed()`，然后 `e.getUnicodeChar(0)=='v'` 时调用 `doPaste()`（`:256-257`）。
- 不检查事件是否来自物理键盘、不检查 InputDevice 类型，也不受软键盘或 extra-keys 设置影响。唯一的开关是 termux.properties 的硬件键盘快捷键禁用项，默认关。
- `getUnicodeChar(0)` 按事件 deviceId 对应的 KCM 求值；VIRTUAL_KEYBOARD 的 KCM 把 KEYCODE_V 映射为 'v'。【推断，属于平台常识，未单独下载 KCM 核对】
- 前面 CTRL、ALT 的 DOWN 事件进入 Termux 时：ALT DOWN 带着 ctrl+alt，会进同一分支，但不匹配任何字母，返回 true 被吞掉，无副作用。
- `doPaste`（`:751-759`）需要当前 session 仍在运行，然后用 `ShareUtils.getTextStringFromClipboardIfSet` 取文本粘贴。

**(c) 剪贴板读取限制。【上游：AOSP14 ClipboardService:1261-1281】**
- `OP_READ_CLIPBOARD` 允许默认 IME 或 `isUidFocused(uid)` 的应用读取。Termux 在前台且有焦点，可以读。
- 写入方是 shell UID，同一个 user 0 的 primary clip，不受影响。
- 读取时 Android 13+ 可能弹出"已粘贴"提示；`clipboard set` 也可能弹出 D7 说的浮层，可能遮挡后续截图。

**仍然只能上设备验证的部分：**
- 注入的修饰键组合会不会在 PhoneWindowManager 被拦截。我没有找到拦截 Ctrl+Alt+V 的代码，但没有逐行核对。
- 端到端粘贴效果，包括中文和 emoji。
- 这条命令的结果是 done（"派发已返回，App 结果未核实"）。它和 IC 在 Termux 上的 UNKNOWN 同样不能证明业务效果，区别是 done 不会停批。

**Chrome 那条"同一字段 fill 实测成功"。【实测】**
- 出处是 `U/reviews/final-api34-capabilities.md:22` 与 `U/stages/36-native-zoom/runtime-u25-followup-api34/commands/ic-wiki-normal-clear-task-text/`。
- argv 是 `fill e106264361 '' --in com.chrome.beta`，结果 done/matches，actualValue 为空。
- 属实，但只是把字段清空。同一字段写非空值没有单独实测。
- IC 在 Chrome 上的增量是 HTML 字段里的局部插入和局部选区。整字段 fill 只能覆盖"整体替换"这一种用法，不能说它替代了局部插入。

### C3　坐标手势默认 UID 定向

**源码层面逐步成立。【源码】**
- `VisualInput.perform:232-238`：没有 `--expect` 时，`coordinatePackage` 取 `currentPackage()`。
- `currentPackage`（`:258-269`）要求恰好一个 `UiScope.operatingWindow`。这个函数在 `UiScope.kt:50-52` 明确写着 `type != "inputMethod"`，并要求 active 或 focused。
- 所以键盘弹出时，定向的 UID 是宿主 App，而不是 IME。
- `AppInput.sender(target.uid)` 走 `injectInputEventToTarget`。首个 DOWN 是 WAIT_FOR_RESULT（`PointerGesture.kt:193`）。抛出的异常经 `isTargetMismatch`（`AppInput.kt:120-125`）判定后，结果为 not sent。
- 基线 7d5be70 的坐标手势走 `UiAutomation.injectInputEvent`，不定向（`git show 7d5be70:…/VisualInput.kt:160,263-271`）。主报告"基线不定向"属实。

**上游一致。【上游】**
- AOSP14 `InputDispatcher.cpp`：`verifyTargetedInjection`（`:573-589`）在 `targetUid != window->ownerUid` 时返回错误。
- 触摸路径在找到新的触摸窗口后马上校验（`:2326-2331`），之后还会对所有已触摸窗口再校验一次（`:2585-2603`）；不匹配时结果为 `TARGET_MISMATCH`。
- IME 窗口由 IME 进程添加，owner 是 IME 的 UID。所以点软键盘会被拒。这已经是源码加上游的完整推理链，剩下只缺一次设备上的确认。【**需设备**做最终确认，但推理链没有断点】

**包内有没有"截图后点软键盘"的真实记录：没有。**
- chrome-1c 的 `tap 140 934`：回执里 `screen.keyboard="hidden"`，窗口只有导航栏、状态栏和 Chrome（window14），触点落在 Chrome 内。
- chrome-2 是基线，不定向，与此无关。

**photopicker "实测必须加 --expect"：夸大。**
- `U/stages/21-input-targets/runtime-api37/result.md:13` 和 global-key-fix 两轮都是一开始就用 `tap 120 1080 --expect com.google.android.photopicker`，没有做过不带 `--expect` 的对照。
- 但同一份原件 `:19` 有一条更有力的**真实**同类拒绝：清理时按 input-focus 解析宿主 Messages（UID10153）的 BACK，第一个 DOWN 就是 targetMismatch。原因是嵌入窗口的 owner 是 UID10232。
- 项目随后在 `stages/21-input-targets/global-key-fix/` 把无 REF 的键改为 `INVALID_UID` 不定向，坐标手势却保留了"当前操作窗包"定向（global-key-fix/implementation.md:7）。
- 这正是 T4 想表达的回归，主报告应该改引这条。

**收益一侧**：`geometry()` 把全部窗口（含 IME）的形状和变换纳入 `shape`（`VisualInput.kt:305,330-358`），派发前与截图时比较，不一致就报 staleImage（`:220-223`）。主报告"UID 条件只额外防住毫秒级竞态"属实。【源码】

### C4　通知 postTime

- 【上游】`NotificationManagerService.enqueueNotificationInternal` 每次都执行 `new StatusBarNotification(…, user, null, System.currentTimeMillis())`。
  - 位置：A14 r1 在 `:6704-6706`，A15 r1 在 `:7428-7430`，A16 r1 在 `:8456`。
  - 更新已有通知时，postTime 不会从旧记录继承。`getPostTime()` 在 NMS 里只用于复制 SBN（`:4453`）和计算 timePosted。
  - `Notification.when` 是 App 设置的显示时间，与 postTime 是两回事。
- 【源码】候选取 `sbn.postTime`（`NotificationActions.kt:147`、`NotificationOperations.kt:133`），写操作要求严格相等（`NotificationActions.kt:245` 抛 notificationChanged）。比较的不是 `when`。
- 【源码】`GatewayService`：`observeWorker` 每 `WORKER_STATUS_MS=2000`（`:436`）运行一次，经 `enqueueManagement`→`publishStatus`（`:310-313`）无条件执行 `notify()`（`:104-115`）。所以候选自己的 Gateway 通知 postTime 每 2 秒刷新一次。
- **裁定：成立。** 顺带：这也让 D6（通知 open 能拉起自身 Gateway）的实际可达性变低。agent 要在 2 秒内读到地址并发出 open 才会触发，但 D6 的修法照样应该做。"频繁更新的通知上反复被拒"本身是推断，包内没有实测。【推断】

### C5　listener 注销失败的封锁范围

逐行核对：

- `NotificationOperations.kt:64-78`：finally 里调用 `unregister()`。失败时**保留 `access`**（`cleanupPending` 仍为 true），设置 `cleanupFailure`，并记一份 `cleanup:"unconfirmed"` 的回执。没有任何重试路径；`NotificationAccess.kt:28-32` 只在本次调用的 finally 里执行一次。
- `UiBackend.kt:139`：`cleanupPending` 时不调用 `sessions.finished(resource)`，UI 资源永久保留。
- `UiBackend.kt:46`：`cleanupFailure = notifications.cleanupFailure ?: closeError`。
- 因此 `Sessions.acquireUi:144` 对**所有 UI 请求**（包括 ui.read）一律拒绝，返回 uiCleanup "restart the worker"。
- `requireResources:198-201`：只要存在 UI 资源，新的 shell/file 一律 busy。
- `expire:241-244`：有资源就不会到期。
- `stop:232-235` 要求空闲，所以正常 Stop 被拒。
- **未被挡住的**：`WorkerConnection.forceExecutor:238-247` 不经空闲检查，本机管理页的"Force executor exit"仍可用。这与上次外评附录 B 一致。
- **"已 done 的回执被改成非零 exit"属实，但位置在 `NotificationOperations.kt:111`**（`notificationCleanup`：unconfirmed 且 exit 为 0 时改成 1）。主报告和底稿 06 写的 `:185-195` 对应的是 `notificationAction` 构造，行号错了。
  - 结果字段仍是 `result:"done"`，只改了 exit。"诱导重发"取决于 agent 看 exit 还是看 result，属于【推断】。
- **基线已有，还是候选新增？**
  - "清理失败就 fail-closed，并用一个永不释放的资源封住整个 worker"在基线就有：`git show 7d5be70:…/UiBackend.kt:36-37,67,377,394` 的 `closeError`，加上连接失败后的 `sessions.closing`。
  - 候选做了两件事：把通知 listener 接进这个机制；新增 exit 覆写与回执里的 cleanup 三态。
- **上游。**
  - AOSP14 `NMS.unregisterListener:4544-4546` 直接调用 `ManagedServices.unregisterService:1084-1089`，注释写明"no need to check permissions"，唯一可能的异常是 `checkNotNull`。
  - 注册路径 `registerServiceImpl:1704-1715` 会 `linkToDeath`；`ManagedServiceInfo.binderDied:1846-1853` 调用 `removeServiceImpl`。所以 worker 死亡后会自动移除。
  - 从 shell 进程调用时，失败只剩 Binder 传输错误或反射错误。
- **与上次外评是否矛盾**：不矛盾。外评"资源-1"列出的挡住项与本次一致；附录 B 把"整台 worker 完全不可用"收窄为"Force exit 仍可用；已在跑的 shell/传输可读可取消；基线既有机制"。底稿 06 和主报告 D5 没有写这两点收窄，读者会以为 worker 只能重启设备才能恢复，所以需要补上。
- **裁定**：问题本身成立（候选扩大了一个几乎不可达的失败路径的封锁面，并加了 exit 覆写），修法也成立（不保留资源，诊断写进 status）。改写见总表。

### C6　8,377,092B / 127 REF 的 dense 树

- 原件 `U/stages/36-native-zoom/runtime-software-api34/dense-row-facts.json`：rawBytes 8377092，nodes 127，uniqueRefs 127，rowCount 123，`allRowLabelUtf8Bytes:[65536]`（每个 row.label 都是同一个 64KiB），missing 为 treeByteLimit。
- `fixture-install-binding.json`：`denseApk build\research-dense.apk` 12,696B，包名 `org.androidctl.research.dense`。
- 设计说明 `U/stages/38-observation-consistency/dense-fixture-preparation.md`（row-budget 模式）写明：一个可见的 65,536B TextView 后面跟 256 个短文本后代，并给行容器设置 CollectionItemInfo，"每个后代保同一已读 label，约 256 份 row.label 即 16MiB"。
- 【源码】`UiSemantics.row:173-204` 对每个节点向上找最近的行祖先，用 `nearestRowLabel` 取标签，所以同一行内每个后代都带着完整的 row.label 副本。
- **交付稿措辞**：candidate-final.md:34 只写了"实际 dense 树为 8,377,092B、127 唯一 REF 且无悬空 parent"，同一文件里没有"fixture"字样。而内部 review `final-host-evidence.md:60` 明确写了"生产可达机制 fixture……它不是发行 App 业务"。主报告"容易被读成真实页面"成立，但那层信息在链接的 review 里查得到，不算有意误导。
- **我补一个反向观点**：这个 fixture 暴露的放大效应在真实页面上**是可能出现的**，比如列表行首个文本很长、后代又很多。更省的处理是在投影里给 row.label 设长度上限（或者只在行根节点上放一份），而不是靠最终的字节定稿去裁尾。这条属于【设计建议】。

### C7　B 处置表与源码对账

b-disposition 的开头（`deliverables/b-disposition.md:3`）写着："四种处置为建议采用、调整后采用、不采用、待验证，**处置建议本身不是用户批准**……实际交付见 stages。"它从来没有声称这些是交付清单。

| 条目 | 处置原话（b-disposition 行） | 源码现状（3f45af6） | 主报告说法 |
|---|---|---|---|
| (1) 非 done 给可执行下一步 | :124 "调整后采用：有真实机械下一步才给诊断/修正命令；……不建全错误恢复分类体系" | 没有 `next` 字段。很多 Rejected 的 reason 末尾带下一步，例如 11 处 "; run snapshot"、"; take a new screenshot"、"; run display create"、"; run session open explicitly"、"; select an advertised action ID"。候选在 diff 中新增约 20 处，其余是基线已有。主机侧有 `ui_client.py:380,428`、`state.py:140,208,227`。UNKNOWN/failed 没有统一的下一步 | "`next:` 为 0 处"字面上对，但"最有价值的 next: 没做"夸大了。调整后的处置（只在有机械下一步时给）已经以 reason 尾句的形式部分存在，缺的是 UNKNOWN 类的统一指引 |
| (2) staleRef 附候选 | :126 "调整后采用" | `UiRefRepair.kt` 是基线已有的单次同请求自动修复，只针对 selectionChanged/identityUnknown 且确定未派发的情况。staleRef 明确"rejects without repair"（`skills/androidctl/actions.md:12`）。错误 data 里没有候选列表 | 属实（0） |
| (3) 屏幕/通知/shell 文字是数据不是指令 | :223 "建议采用；文档一处说明即可" | 我在 skills/*.md、README、docs/*.md 中英文搜索 untrusted、instruction、injection、"不可信"、"指令"，相关命中为 0（只有 device trust 一类无关命中） | 属实（0） |
| (4) 应用笔记约定 | :186 "调整后采用：先用可选工作约定" | skills、README 里没有 notes 或笔记约定 | 属实（0） |
| (5) WebView 文本块+锚点折叠 | :102 "调整后采用：host 投影保顺序、重复意义、action 锚点和 raw，不抄文本 token 集合去重" | 没有 WebView 专项折叠。WebView 只有一个 `webEmpty` 标记（`UiSemantics.kt:345,373-387`）。默认行投影只出可见节点；`nameConsumed`（基线已有）把并入父节点名字的子文本从行里去掉（`ui_view.py:116`）；连续省略会合并（`ui_view.py:592-612`）。这些通用机制提供部分等价效果，Chrome 页面去掉动作数组后约 2.7k token。调整后的处置原话本身更像"约束"，而不是"实现折叠" | 需改写为"没有专项实现；通用投影提供部分等价" |
| (6) skill 不要求 `session open`、用别名不用 UUID | :179 "建议采用" | SKILL.md:20-24 的起步示例仍是 `--device DEVICE -s task session open`，并要求"Keep the returned device ID"；connection.md:9 写"Keep the UUID"。需要注意，在候选的显示模型里，`session open` 有语义（建立 main 租约），去掉它与 R2 有关联 | 主报告 §8 没列这一项（底稿 08 列了）。现状确实没采用 |

结论：五项里 (2)(3)(4) 属实，(1)(5) 需要改写。主报告"用户可能以为这些已经落地"偏重，准确的说法是"没有对账表"。不可信内容提示只需一行文档，应该补上，这一点同意。

---

### J1　后台屏的协议必填种类（独立推演）

三种模型：

- **X（现状）**：主机持久偏好（device+session），每条 UI 请求带 kind，设备侧检查。
- **Y**：session open 时固定目标，主机不记忆，没有租约时隐式取得 main。
- **Z**：设备端墓碑，worker 内存里记住"会话 S 曾是 virtual"。

| 情形 | X 现状 | X'（发出 create 前写偏好 + close 时清除） | Y | Z |
|---|---|---|---|---|
| worker 重建 | 拒（displayRequired） | 拒 | **静默落主屏** | **静默落主屏**（墓碑丢失） |
| VD idle 到期 | 拒 | 拒 | **静默落主屏**（除非 VD 租约不做到期） | 拒 |
| create failed/UNKNOWN 后续发 | **静默落主屏**（包内 API37 真实发生过一次） | 拒 | **静默落主屏** | 拒（create 尝试时写墓碑） |
| 显式 session close 后再 open | open 建 main，偏好仍是 virtual：多一次 displayMismatch 往返 | 偏好已清，正常进 main，**不误拒** | 正常 | 视设计而定 |
| 跨工作区 / 默认会话名 `default` 的遗留 virtual | **误拒**，提示还引导 agent 去建屏（T7） | 正常收尾（close）的任务不再遗留；放弃而未 close 的任务仍会遗留一次误拒，需提示里加 `display select main` | 不存在 | 同一 worker 生命周期内有遗留 |

第四种方案我评估了三个：

- **(a) 每条命令显式带 `--display virtual`，或用环境变量**：去掉主机文件、锁和 select，保留设备侧字段。agent 漏写标志、同时租约已失效时，会静默落主屏，需要两个条件同时发生。LLM 偶尔会漏写参数，而后台屏的用途恰好是"人正在用主屏"。我不推荐。
- **(b) 无租约时 UI 命令不再隐式建租约**：不需要任何状态，也不会静默落主屏。代价是所有只用主屏的用户每次空闲到期（默认 10 分钟）后都要多一次 `session open`，这与 B 报告"不强制 session open"相反，影响面比后台屏本身大得多。不推荐。
- **(c) 把 virtual 编码进会话名或租约类型**：本质上就是 X 的偏好换了个载体，而且引入魔法命名。不推荐。

**裁断：主报告采纳 02 的结论是对的**（设备侧必填种类是唯一能扛住 worker 重建的保护，不到 40 行）。但应该在"改写入时点"之外再加"显式 close 时清除偏好、status 显示偏好、displayRequired 提示补上 `display select main`"。

- close 时清除不损失保护：close 已经销毁了 VD，继续后台工作必须重新 create，而 create 会重写偏好。
- 这样可以消掉 T7 的大部分遗留，也正好消掉底稿 02 的 F3（close→open 的多余 mismatch 往返）。

底稿 08 主张的"session 固定、无主机记忆"在 worker 重建和 create 失败两种情形下都会静默落主屏，不应采用。如果按 R2-C 把后台屏拆出首次合入，这一整套都留在分支里，问题延后。

### J2　IC 与借贴

各自的**独有能力**（已证实的部分）：

- **IC commit**：不碰剪贴板，在当前编辑器插入 Unicode。
  - 唯一一个其它通道都写不进去的实测正例是 Termux：fill 对 `terminal_view` 报 notReady，keys 映射不了中文，借贴因为没有 PASTE 而失败（`stages/35-clipboard/p9-borrowed-paste/runtime-api34/result.md`）。
  - 但在 Termux 上，IC 结构上只能报 UNKNOWN，并且会停批（`stages/28-current-editor/runtime-mumu/result.md` H4）。
  - Contacts、Chrome、LongEditor 上的正例，借贴、fill 或 keys 也能覆盖。
  - 代价：用户要启用服务，只能用于主屏，还有一座 AIDL 桥（`IEditorBridge.aidl`，`CurrentEditorInput/EditorBridge/EditorSurrounding.kt` 共约 400 行，加上服务和主机部分）。
- **IC select**：按绝对 UTF-16 偏移设置选区。
  - 替代路径一：现有的 `key REF shift+left/right`，做相对选区，次数多。
  - 替代路径二：新增一个 AX SET_SELECTION 的类型化命令。注意 generic `action` 会拒绝带参数的标准动作（`NodeActions.kt:11-24,89-91`），所以"直接用 AX SET_SELECTION"并不是现成的。
- **借贴**：在广告了 PASTE 的字段里按光标插入或替换 Unicode，并恢复原剪贴板。实测正例：AOSP Contacts 空剪贴板与 HTML 剪贴板的恢复、Google Contacts 带标签字段。
  - 这项插入能力完全可以由 `clipboard set` + `action REF --action-id 32768`（PASTE 不在 parameterized 集合里，可以走 generic action）替代，只是失去自动恢复。
  - "恢复原剪贴板"的需求包内没有出现过任何真实冲突证据。

**替代条件是否成立。**
- 借贴的替代：组件都在，PASTE 走 generic action 在源码层面可行，组合本身没有实测。
- IC 在 Termux 上的替代：ctrl+alt+v 在源码和上游层面逐环成立，全局键送达 Termux 已有实测。只差一次端到端验证。

**总复杂度最低、又不丢已证实能力的组合：**

1. 删借贴（约 354 行状态机加接线），文档写明组合用法，并说明它会覆盖剪贴板。
2. 做一次设备验证：`clipboard set "ACP中文😀"` → `key ctrl+alt+v`（无 REF）→ 截图。
   - 成功：IC 的 commit 和 select 一起删除。选区先用 shift+方向键；有真实需求时再加一个约百行的 `select REF START END`（AX SET_SELECTION）。
   - 失败：只保留 IC commit（R3-C），删掉 IC select 和三态里的 UNKNOWN/FAILED 细分规则。
3. 不建议"保留 IC、删借贴"作为默认结论。IC 唯一的独有正例里它恰好不能报 done，而它的依赖面（服务启用、主屏、桥）比借贴还大。

这与主报告 R3 的 B→C 条件结构一致，所以主报告裁断不需要改方向，只需要改正上面两处依据。

---

## 3. 顺带发现的主报告其它错误或夸大

1. **D5 行号错误**：exit 覆写在 `NotificationOperations.kt:111`，不在 `:185-195`。底稿 06 的 `:147-161` 也对不上 3f45af6，实际 finally 在 `:64-78`。D5 还漏写了"Force executor exit 仍可用"和"基线已有机制"，这两点上次外评附录 B 已经收窄过。
2. **T4 引错了证据**：photopicker 不是"实测必须 --expect"，那次运行一开始就带了 `--expect`。应该改引 stage 21 那次真实的 BACK targetMismatch，以及随后的 global-key-fix：项目已经在键上认可了"按当前窗包定向会误拒嵌入或跨 UID 窗口"，坐标手势却没有同步。
3. **R3 依据里"选区可用 AX SET_SELECTION"**会让读者以为这是现成能力，实际要新增命令。现成的替代是 shift+方向键。
4. **E3 里"Chrome 的同一字段 fill 实测能写"**只测过清空。
5. **E2 里"同一台设备同一页面"**要补一句：J54 在重启前，J67/J68 在重启后（post-reboot）；J67 是新 worker 唯一的冷首采。主报告 E2 只说"没有做同一 worker 内的交替对照"，没有指出冷/热这个最主要的混杂因素。
6. **D6 与 T6 的交互**：候选自己的 Gateway 通知每 2 秒刷新 postTime，客观上让 D6 很难被触发。主报告把 D6 当作独立风险，可以加一句可达性说明，修法不变。
7. **J67 用的是基线 7d5be70 的 single 链**（加大预算），不是 3f45af6 的采集链。主报告称"HEAD single"沿用了包内"HEAD=7d5be70"的说法，没有错，但读者容易误认为是候选 HEAD。建议写成"基线 single（7d5be70 + 8192/3000ms 研究补丁）"。

## 4. 没能核实的范围

- 一切设备行为：Termux 上 ctrl+alt+v 的端到端效果、PhoneWindowManager 是否拦截注入的 Ctrl+Alt+V、软键盘 tap 实际被拒、热态 single 的耗时。这些只能做到源码加上游推理。
- VIRTUAL_KEYBOARD KCM 对 KEYCODE_V 的映射，我没有下载 `Virtual.kcm` 原文核对。
- 1024 封顶的 single 旁证样本（chrome-a/b、e9-single-chrome）的具体设备和页面，我没有逐一核实。
- J1 只做了源码推演。底稿 02 引用的 API37 "VIEW 落 task84"原件，我没有重读。
- AOSP15 及以后的 InputDispatcher、ManagedServices 只核了 14（NMS 核了 14/15/16）。
- 没有通读 decisions.md，也没有逐提交审查其它方向。
