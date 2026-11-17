# 专题 03：输入通道及其组合（方向 16、20、23–29、35、36）

评审员：独立子评审（只读）。源码行号均取自候选 `3f45af6`，路径相对 REPO 根；`PKG/UG` = `PKG/ultragoal/2026-11-11-next-improve`。
证据类别：【实测】包内原件记录的真实运行（注明版本）；【源码推断】；【AOSP/上游】已在线核对固定 tag；【设计建议】。
没有设备，没有重跑任何东西。

---

## 1. 结论摘要（重要的在前）

1. **IC（方向 29）的独有价值，基本可以用候选里已有的命令组合替代，应当重新裁决。** 当初支撑 IC 的两个案例：
   - Termux 0.118.3 稳定版有硬件键盘快捷键 `Ctrl+Alt+V → doPaste()`【上游：termux-app@5b657c6 `TermuxTerminalViewClient.java:236-260,751-758`】。所以 `clipboard set "中文"` + `key ctrl+alt+v` 就能把 Unicode 写进 Termux，用不到 IC。候选已实现这两条命令，`KeyPlan.meta` 也正确设置了 CTRL/ALT meta。这条只做了源码推断，未实测。
   - Chrome 里的 HTML 字段，`fill` 本来就能写，包内收尾时就对同一字段 fill 成功过【实测】。
   - IC 剩下真正独有的是"按绝对 UTF-16 偏移选区"。这件事可以用带 REF 的 AX `ACTION_SET_SELECTION` 做：成本和 `range` 相当，不需要用户启用服务，也不受 main-only 限制。
   - 反过来，IC 在自己的旗舰场景表现很差：Termux 上 commit 结构上永远是 UNKNOWN，并且会停批【实测：28 MuMu】；Openreads 即时报 done，实际值却是 742【实测：28 API37】。
   - 建议冻结 IC，补两条小证据后大概率删除。
2. **借用剪贴板 PASTE（方向 36）是 354 行状态机加约 200 行接线和文档，换来的是"恢复用户原剪贴板"这一项。这项价值没有任何真实冲突证据。** 不恢复的等价组合今天就能用：`clipboard set TEXT` + `action REF --action-id 32768`（generic action 不排除 PASTE，见 `NodeActions.kt:13-33`）。建议在"删除并文档化组合"和"简化为无条件恢复的约 100 行实现"之间二选一，不维持现状。
3. **默认坐标 tap/swipe/drag/pinch 改成按"当前操作窗口根包"的 UID 定向（`VisualInput.kt:234-240,263-274`），这是对基线的行为回归，防护收益却很小。**
   - 回归：凡是点到另一个 UID 的窗口都会 targetMismatch 并 not sent，例如软键盘、嵌入式 photopicker、非操作窗口的系统 UI。Messages 的 photopicker 就必须靠 `--expect com.google.android.photopicker` 才点得进去【实测：21 API37】，而这个包名在 AX 树根上看不到。
   - 收益小：同一函数在派发前已经比较截图时与当前的完整窗口集合（`VisualInput.kt:223`，窗口集合包含 IME），窗口一变就报 staleImage。UID 条件额外防住的只有毫秒级竞态。
   - 建议坐标手势默认不定向，`--expect` 保留为显式定向；节点 touch 和 REF keys 维持定向。
4. **typed Intent（方向 28）真正的增量有三项：SEND + EXTRA_STREAM 的 ClipData 授权、具体 handler 解析（避免弹选择框）、以及免去 shell 转义。** `am start` 对 SEND 不会迁移 ClipData。但候选自己重写了平台 `Intent.migrateExtraStreamToClipData` 的一个变体，语义有偏差（见 §3 D4）。`landing`（ActivityTasks 303 行中的主要部分）对 LLM 的决策价值低，建议收缩。
5. **指针、节点类输入（方向 23–26）各有真实职责，并有真实业务正例，应保留：**
   - 节点 touch：Chrome Tokyo 链接；
   - action/reveal：Contacts 智能体自然用了 reveal；
   - range：VLC 进度条；
   - drag：Contacts 跨词选区；
   - pinch：Gallery 放大。

   问题在回执和文档过度展开：KeySequence 与 PointerGesture 用两套编号和字段约定记账；默认文本行整段 dump `gesture`/`keys` JSON（一次普通 tap 的结果行里约 800B 是内部计数）。
6. **`type --via keys` 和编辑组合键（方向 27）是基线缺失的真实能力**（基线只有 fill 和 4 个键），应保留。它也是第 1 条中替代 IC 的关键。
7. **结果语义在各通道之间不一致，并存在只为极窄区分而设的规则：**
   - IC select 的"未匹配且等于 before → UNKNOWN"；
   - IC commit 永远不会 FAILED，而 select、paste、fill、range 会；
   - 借贴在"借写后 PASTE 根本没发、原剪贴板已恢复"时报 FAILED。

   对 LLM 真正改变下一步的只有三类：没发出去 / 已确认完成 / 发了但没确认。建议各通道对外统一到这三类，加一个小的统一 `input` 摘要，细节放 JSON。
8. **有界滚动查找（方向 20）有自然采用证据。** 两个候选业务臂中，智能体都自发使用了 scroll-until：Chrome 一步找到，Contacts 因名字猜错（界面标为 "Title"）而合理失败。应保留，只需精简输出。
9. **局部核值与末观察分离（方向 16）是一次小重构（主代码 +122/−123），方向正确。** 唯一值得注意的是 fill 精确比较对"值 + 标签"一类 AX 文本必然误报 failed（Google Contacts `…, 名字`），建议在失败原因中显式提示"请求值是 actualValue 的前缀"。
10. **整体量：** 本专题 11 个提交主代码约 +6,150 行，测试约 +6,300 行，契约 JSON 另有数千行。若按第 1–4 条处置，可删减约 1,800–2,300 行主代码和对应测试、文档，同时减少 LLM 需要选择的通道数，能力损失只剩"恢复用户富文本剪贴板"和"无服务情况下 Termux 以外的 IC-only 编辑器"这两项未被证明的场景。

---

## 2. 方向处置表

| 方向 | 处置 | 依据 | 真实能力损失 / 维护代价 |
|---|---|---|---|
| 16 fill/check 局部核值与末观察分开 | **保留** | 主代码 +122/−123，把核值从末观察中解耦，末观察失败时核值仍可保留；Contacts 实测 matches 与末观察分列成立。 | 无新状态负担。建议：fill differs 时若 actualValue 以请求值开头，在 reason 中说明（Google Contacts 标签文本，§3 D9）。 |
| 20 原容器内有界滚动查找 | **保留，精简输出** | 两个候选业务臂中智能体均自然使用（chrome-1c 一步 found；contacts-2 名字猜错，2 步后 routeUnavailable，失败合理）。省掉中间树交付和 LLM 轮次。 | 主代码 +541。`step`/`finalCheckUnavailable`/`errorMessage` 等字段对 LLM 是噪声，只需 found/notFound/stop 和 REF。 |
| 23 UID 定向注入 + 节点 touch | **调整** | 节点 touch 有实测正例（Chrome Tokyo，无截图）。默认坐标手势的 UID 定向收益小、回归明确（§3 D1）。按节点包名推导 UID 会误拒 RemoteViews（§3 D2）。 | 改为坐标默认不定向，`--expect` 显式定向；节点 touch 和 REF key 保留定向，并允许 `--expect`。代码量基本不变。 |
| 24 显式 action/reveal | **保留** | reveal 在 contacts-2 中被智能体自然使用两次；按 label 调用 action 有分享正例；generic action 也覆盖 PASTE/COPY 等无参动作。 | 主代码 +465，可接受。 |
| 25 range（SET_PROGRESS） | **保留** | VLC 进度条精确 seek 有实测，拖动无法替代。 | 主代码 +284，小。 |
| 26 定时手势 drag/pinch/timeline | **保留 drag/pinch，`timeline` 原始 JSON 降为内部或高级用法** | drag 跨词选区、pinch 缩放均为实测正例；首次 pinch 幅度不足无效果属于正常业务反例。 | 主代码 +776。CANCEL 有价值但回执过细；与 KeySequence 的记账应统一（§3 D10）。 |
| 27 type/keys 编辑组合键 | **保留** | 基线没有插入式文本，也没有组合键。Contacts ctrl+a 加 keys 替换、Termux ASCII 均为实测。也是 IC 的替代路径。 | 主代码 +502。全局键 INVALID_UID 有 J178 实测依据（嵌入 BACK），同意。 |
| 28 typed Intent | **调整（收缩）** | 真实增量：SEND 的 ClipData 授权（fixture 四格实测）、handler 解析避免 Resolver、免 shell 转义。VLC SEND 未播放，不是授权问题（fd 已打开，随后 EOF）。 | 改用平台迁移语义（§3 D4）；`landing` 收缩为一行"前台包/Activity"或删除，由动作后观察承担。约可删 300–400 行。 |
| 29 IC commit + UTF-16 局部 selection | **替换 / 证据不足 → 建议冻结后删除** | 见摘要第 1 条和 §4 R1。 | 删除约 780 行主代码、685 行测试、AIDL 和 IEditorBridge。select 改为 AX `select REF START END`，约 100–150 行。损失：不支持 SET_TEXT/PASTE/组合键粘贴、只吃 IC 的未知编辑器（WPS 反例中连 IC 也写不进去）。 |
| 35 结构剪贴板 | **保留，简化 null 准入** | get/set/clear 用途比输入更广，命令简单；set + `action PASTE` 是借贴的替代。 | 每次 null 都做一次 AppOps `noteOperation` 采样（`ClipboardAccess.kt:127-168`），本身会写入隐私访问记录。可改为如实报告 `returnedClip=false` 加一个 `readAllowed` 事实。主代码 +600。 |
| 36 借用剪贴板一次 PASTE | **替换或大幅简化**（§4 R2） | 唯一独有价值是恢复原 ClipData；实测只覆盖空剪贴板或自建 HTML 种子，从未出现真实用户 Copy 冲突。Termux 场景借写后才发现没有 PASTE，失败。 | 删除可省约 556 行主代码和 483 行测试；简化后约保留 100 行。 |

---

## 3. 发现（按重要性）

### D1 默认坐标手势按操作窗口 UID 定向：回归大于防护

- 类型：设计取舍（行为回归），接近确定缺陷。证据：源码推断 + AOSP + 实测类比。
- 位置：
  - `android/app/src/main/java/com/rainng/androidctl/VisualInput.kt:234-240`：`coordinatePackage(args){ currentPackage(run) }`；
  - `VisualInput.kt:263-274`：取唯一 operatingWindow 的根包；
  - `AppInput.kt:60`：转成 PM UID；
  - 基线 `7d5be70:VisualInput.kt` 用的是 `UiAutomation.injectInputEvent`，不定向。
- 触发路径（源码推断，未实测）：
  - LLM 截图后点软键盘上的某个键：IME 窗口属于 IME 的 UID，不是 operating window → InputDispatcher 返回 TARGET_MISMATCH（AOSP14 `InputManagerService.java:894-902`）→ not sent。
  - 点嵌入式跨 UID 内容。实测：Messages 内嵌的 photopicker 属于 UID10232，必须加 `--expect com.google.android.photopicker`。AX 根包是 Messages，LLM 无法从树中得知这个包名。
  - 系统对话框不是 active window 时，同理。
- 防护收益：`VisualInput.kt:223` 在派发前比较截图时与当前的完整 `shape`，其中 `windows` 包含该 display 上的全部窗口（含 IME，见 `AccessibilityConnection.kt:53-66`）。任何新窗口出现或消失都已经报 staleImage。UID 条件额外防住的只是几何检查到 DOWN 之间的毫秒竞态，以及"同一组窗口内容换了 UID"这种几乎不存在的情形。
- 组合问题（D3）：首个 DOWN 的 mismatch 也会先调用 `dispatch()`，使当前图失效（`PointerGesture.kt:202`）。LLM 想改用 `--expect` 重试时，必须先重新截图。
- 建议：坐标手势默认 `INVALID_UID`（回到基线语义），保留 AndroidCtl 自身窗口的预检；`--expect` 作为显式定向。节点派生的 touch 和 REF key 继续定向。

### D2 节点 touch / REF key 用节点 packageName 推 UID，会误拒 RemoteViews 一类内容

- 类型：设计取舍。证据：源码推断（与上次外评"输入-3"一致）。
- 位置：`VisualInput.kt:258`、`SystemActions.kt:160`。
- 触发：桌面小部件、通知栏条目等 RemoteViews。节点 package 是发布方 App，接收输入的窗口属于 launcher 或 SystemUI → mismatch，not sent，结果里也不给出可用的包名。
- 建议：节点 touch 也允许 `--expect`；mismatch 时附上节点所在窗口根包作为候选。约 10 行改动。

### D3 not sent 的输入也带 `dispatchStarted=true`，并使当前图失效

- 类型：确定缺陷（轻）。证据：源码（同上次外评"输入-4"）。
- 位置：
  - `PointerGesture.kt:197-205`：先 `dispatch(result(UNKNOWN))`，再真正 send；
  - `KeySequence.kt:91-93`；
  - `VisualInput.kt:287-290`：dispatch 前还要再判一次 targetInputReady。
- 影响：首个 DOWN 被确认 mismatch 时，结果是 not sent，同时 `dispatchStarted=true`，图像资格丢失，还会跑一轮 settle。LLM 会看到互相矛盾的字段；与 D1 叠加时，代价是多一次截图。
- 建议：首个 DOWN 被确认 mismatch 时，不推进 image 失效，并把 dispatchStarted 设为 false。

### D4 SEND 的 ClipData 迁移是自造变体，与平台语义不一致

- 类型：设计缺陷。证据：源码 + AOSP 固定 tag + 实测回执。
- 位置：`android/app/src/main/java/com/rainng/androidctl/ActivityIntent.kt:44-48`：
  `if (ACTION_SEND) clipData = ClipData.newRawUri(null, stream)`。只在显式 `--grant-read-uri-permission` 时才加读权限。
- 平台：AOSP14 `Intent.migrateExtraStreamToClipData`（`core/java/android/content/Intent.java:12306-12360`，android-14.0.0_r1）的做法：
  - 用 `getType()` 作 ClipData MIME；
  - 把 EXTRA_TEXT / EXTRA_HTML_TEXT 放进同一个 item；
  - 有 stream 时**自动加** `FLAG_GRANT_READ_URI_PERMISSION`；
  - 另行处理 SEND_MULTIPLE。
  - 普通 App 发起分享时，Instrumentation 会自动调用它；`am start` 不会。
- 实测：fixture stream-on 回执中 ClipData MIME 为 `text/uri-list`，Intent MIME 为 `image/png`。
- 影响：
  - 接收方如果检查 ClipData 描述或 item 文本，看到的东西和真实分享不同；
  - LLM 必须记住单独加 `--grant`；
  - 不支持 SEND_MULTIPLE。
  - VLC 失败不能归因于此（日志显示 fd 已打开，随后 EOF），但这条偏差会让"与真实分享一致"的结论站不住。
- 建议：用反射调用平台 `migrateExtraStreamToClipData()`。候选已有 HiddenApi 基础设施。这样能删掉自造分支，`--grant` 也只剩 data URI 场景需要。

### D5 IC 在 Termux 上结构性只能报 UNKNOWN 并停批；done 也不等于字段值

- 类型：设计取舍。证据：实测 + 源码。
- 位置：`EditorSurrounding.kt:73-74`：surrounding 的 offset=-1 时不可解释，直接 unobservable。Termux 的 TerminalView 在 commit 后清空 buffer（研究报告已指出），所以每次都会落到这里。
- 实测：
  - 28 MuMu 的 Termux H4：第二步 commit 报 UNKNOWN，aggregate failed，exit3，后续步骤零 RPC，但截图上文本确实写进去了；
  - 28 API37 Openreads：commit `R7IC-FL-42` 即时 done/matches，稍后和重开后都是 `742`（digitsOnly 过滤）。
- 影响：IC 恰好在"唯一需要它的 App"里不能给出 done；在能给出 done 的 App（Contacts、Chrome、Openreads）里，fill、keys 或 paste 也都能写，而 done 的信息量又有限。

### D6 IC commit 与 select 的结果映射不对称，select 带一条极窄的三态规则

- 类型：设计取舍（过度区分）。证据：源码。
- 位置：
  - `CurrentEditorInput.kt:126-146`：select 中 `differs && before.hasSameSelection(after)` 映射为 UNKNOWN，其他 differs 映射为 FAILED；
  - commit 在 `EditorSurrounding.kt:81-93` 中所有不一致都返回 unobservable，永远不会 FAILED。
- 包内所有真机 select 都是 matches，这两条分支从未在真实环境中触发。
- 对 LLM 来说，"选区没变"和"选区变成别的"下一步动作相同：重新观察。建议整条随 IC 删除；如果保留 IC，统一为"matches→done，否则→UNKNOWN"。

### D7 借写和 `clipboard set` 未抑制 SystemUI 剪贴板浮层

- 类型：确定缺陷（真机可见副作用）。证据：AOSP + 源码。上次外评已提出，候选未处理。
- 位置：
  - `ClipboardAccess.kt:98-104`：`prepareBorrow` 用 `ClipData.newPlainText(marker, value)`，没有 extras；
  - AOSP14 SystemUI `ClipboardListener.java:51-54,118-130`：来源为 `com.android.shell` 且带 `com.android.systemui.SUPPRESS_CLIPBOARD_OVERLAY=true` 时才抑制浮层。
- 影响：在真机上每次借贴会弹出两次浮层（借写一次、恢复一次）。浮层可能遮挡界面，也可能进入随后的观察窗口集合。P9 API34 的 Name 正例中，主 observedChange 为 `windows changed appeared:1`，是否就是这个浮层未核实，属于证据缺口。
- 建议：如果保留借贴或 clipboard set，给借写的新 clip 加这个 extra（恢复时不能改动原 clip）。如果按 R2 删除借贴，只需处理 `clipboard set`。

### D8 借贴在"PASTE 从未发出、原剪贴板已恢复"时报 FAILED（exit1）

- 类型：过度区分（轻）。证据：实测 + 源码。
- 位置：`BorrowedPaste.kt:304-312`。实测为 35 API34 的 Termux 反例：主结果 failed/exit1/actionUnavailable，PASTE attempted=false，restore 状态为 restored。
- 对 LLM 而言，这和 not sent 等价：界面没变，剪贴板已还原。用 FAILED 只是为了表达"借写本身是副作用"，这是一个读者无法据以行动的区分。

### D9 fill 的精确比较对"值 + 标签"型 AX 文本必然误报 failed

- 类型：设计取舍。证据：实测。
- 情况：Google Contacts 的 AX text 是 `P9M-…-AB😀Z, 名字`（P9 MuMu，候选 8eef 物料），fill 精确比较只会得到 differs/failed，LLM 重填也没用。借贴能成功，是因为它按选区做插入核对。
- 建议：保持精确语义不放宽，但在 `actualValue.startsWith(requested)` 时，在 reason 中给出"请求值是 actualValue 的前缀，可能附带标签"。成本几行，能改变 LLM 的下一步决定（判为完成而不是重试）。

### D10 KeySequence 与 PointerGesture 两套记账，格式刻意不同

- 类型：维护性。证据：源码 + 公开文档。
- 位置：
  - `KeySequence.kt:187-211`：successfulPrefix 从 0 计、按计划索引；字段 `cleanup`/`returned`；
  - `PointerGesture.kt:306-351`：从 1 计、按调用号；字段 `cancel`/`accepted`、`asyncQueued`、`skippedMoves`、`lastSkippedOffsetMs`、`keyIntervals`；
  - `images.md:244-246` 明说两者"retain their existing separate format"。
- 影响：两份文档、两套测试、两套 wire 夹具，而 LLM 能用到的只有"发出了几步/计划几步、是否需要清理、清理结果"。
- 建议：统一对外摘要 `input={sent:"none|partial|all", events:"n/m", stop, cleanup:"none|ok|failed|unknown"}`，详细事件只在 JSON 中保留；同时统一编号起点（上次外评"合并-10"）。

### D11 默认文本结果行整段 dump 内部回执

- 类型：效率。证据：实测 + 源码。
- 位置：
  - `androidctl/src/androidctl/ui_client.py:522-534`：`input_facts` 把 `inputTarget`、`gesture`、`keys`、`cancel` 用 compact JSON 全部写进结果行；
  - `ui_client.py:297-306`：clipboard 和借贴各段以结构化 JSON 输出；
  - `ui_client.py:307-322`：IC 的 before/after 预览，每段最多 512 个 UTF-16 单元。
- 实测：
  - chrome-1c 中智能体的一次普通 `tap`，结果行包含完整 `gesture={…lastReturned…omitted…}`，约 800B；
  - P9 借贴 stdout 为 3130B，其中两段 `emptyBasis` 各约 12 个字段。
- 建议：默认行只显示 result、stop 和 sent 摘要，细节放 `--json`。

### D12 全局键 INVALID_UID 与 mismatch 文案匹配（接受，记录边界）

- 类型：设计取舍，同意现状。
- 位置：`SystemActions.kt:129-130,151-161`；`AppInput.kt:119-126`。
- 依据：
  - J178/J179 的实测中，嵌入 BACK 的接收 owner（10232）与根包不一致，无 REF 的全局键改为不定向有依据；
  - mismatch 判定依赖英文异常文案，已核实 android-14.0.0_r1、15.0.0_r1、16.0.0_r1 三个 tag 的文案相同；即使文案变化，也只会退化成 UNKNOWN 加 CANCEL，方向安全。
- 剩余边界：全局键对 AndroidCtl 管理窗的保护只靠预检（FLAG_SECURE 不阻止按键注入），文档已承认存在 TOCTOU。不建议为此加机制。

### D13 手势 CANCEL：保留，但证据只到"已发出"

- 类型：同意现状，说明价值边界。
- 位置：`PointerGesture.kt:147,273-300`。
- 实测（24 MuMu，失窗）：系统自己的 CANCEL 比产品 CANCEL 早 548ms 到达 App，产品 CANCEL 是否被接收未知。
- 判断：产品 CANCEL 真正防的是"产品自身异常或预算到期，导致只发了 DOWN 而没发 UP"。这种情况包内没有实测，但发一次 ASYNC CANCEL 成本极低，平台语义也正确。保留，但不需要 `cancel` 的十几个字段。

---

## 4. 问题 1：文本进入一个字段的路径

| 路径 | 平台前提 | 失败模式 | 真实证据 | 不可替代职责 |
|---|---|---|---|---|
| `fill REF TEXT`（AX SET_TEXT，整字段） | 节点可编辑并广告 SET_TEXT；需先获得焦点；支持 Unicode | 不 editable 的节点（Termux 的 `terminal_view`）→ notReady；AX 文本带标签时报 differs/failed（D9）；未聚焦的 Flutter 字段（Openreads Author 只广告 Click）→ not sent，需先 click | Contacts（含中文和 emoji）、Chrome Find（智能体自然使用）、Openreads | **是**：整字段替换的主通道，两个候选业务臂中智能体只用了它 |
| `type [REF] TEXT --via keys` / `key CHORD` | KCM 能映射全部文本；REF 必须已聚焦；无 REF 时为全局键 | 不可映射（中文）→ not sent；App 不按 KCM 产出 | Contacts DEL、ctrl+a + type；Termux ASCII（含全局键） | **是**：插入、编辑键、组合键；终端；以及"组合键粘贴" |
| `editor commit/select`（IC） | 用户在设置中启用 AndroidCtl Browser support 服务；仅 main 屏；当前有 a11y IC | Termux 必然 UNKNOWN（D5）；done ≠ 值（Openreads）；WPS 单元格 commit 未写入（历史反例） | Termux Unicode（截图正例）；Contacts、Chrome、Openreads 局部 | **基本没有**：Termux 可用 `clipboard set` + `key ctrl+alt+v`（源码推断）；选区可用 AX SET_SELECTION；其余 App 有 fill/keys/paste |
| `type REF TEXT --via paste`（借贴） | 原剪贴板可读，或空剪贴板准入通过；剪贴板非 URI/Intent；目标在借写后广告 PASTE | Termux 无 PASTE → 借写后才失败（D8）；READ 被拒 → not sent；未知所有权时不恢复 | AOSP Contacts（空剪贴板、HTML）、Google Contacts 标签字段 | **部分**：Unicode 插入可由 `clipboard set` + `action PASTE` 替代；只有"恢复原富文本剪贴板"不可替代 |
| `clipboard set` + 手动 PASTE（generic action，或 Termux 的 ctrl+alt+v） | 同上，不恢复 | 覆盖用户剪贴板（D7 浮层） | 包内未实测这一组合 | 是借贴与 IC 的共同替代 |

**LLM 能否根据文档选对？** 不能稳定选对。
- `SKILL.md:83-99` 只按命令列出主题，没有"先 fill；不可编辑则 keys（ASCII）；Unicode 插入用 paste…"这样的路由表。
- `input.md` 的 IC 与借贴两节是密集的法律式条文，`dispatchStarted`、`attempted`、`callReturned`、`captureValid` 四个概念并列出现。
- 四个候选业务臂中（不只是 Contacts），智能体从未用过 IC、keys、paste、touch、action、range、drag、pinch 或 typed open【实测：paired-runtime cli-index】。新通道的可发现性和使用率都没有得到证明。

**是否合并？** 建议对外只保留两个"写文本"命令，外加 `key`：
- `fill REF TEXT`：整字段替换，不变。
- `type [REF] TEXT [--via keys|paste]`：插入。可选方案是在**任何副作用之前**按确定性规则预选：整串可由 KCM 映射就用 keys；否则若带 REF，就用 paste（覆盖剪贴板，或简化版恢复）。这不违反"不 fallback"：规则在派发前确定，派发后不切换。输入研究（`research/input/report.md` 第 35 行）本来就允许这种做法。用户如果坚持由 LLM 显式选择，保留必填 `--via` 也可以。
- IC 删除；`select REF START END` 走 AX。

---

## 5. 问题 2：指针与节点动作及其守卫

| 守卫 | 防的真实事故 | 证据 | 判断 |
|---|---|---|---|
| fresh 身份/几何/可见 + 中心点在屏内（节点 touch，`AppInput.kt:128-152`） | 引用过期后点到另一个元素 | 历史 22 案中 REF 过期和窗口丢失真实发生过；节点 touch 正例（Tokyo）实测 | 保留，成本低 |
| 节点 touch / REF key 的 UID 条件 | 焦点或窗口被另一个 App 抢走后输入落错 | E1 AVD 正负例；21 中错焦拒绝为 notReady（这是焦点检查，不是 UID） | 保留，但加 `--expect`（D2） |
| 坐标手势默认 UID 条件 | 几何检查到派发之间的毫秒竞态 | 无事故证据；反例为 photopicker | **改为默认关闭**（D1） |
| 首个 DOWN mismatch 算 not sent | 让 LLM 知道"可以安全换个方法" | E1 实测 | 保留，修 D3 |
| 已执行前缀 / possibleDown / CANCEL / 逐键 UP | 中断后指针或修饰键卡在按下状态；批处理在 UNKNOWN 后停下 | 失窗 CANCEL 实测（系统先行取消）；修饰键中断后的 UP 释放**无设备证据**（25 结果自述） | 机制保留，回执压缩（D10、D11） |
| 原始 `timeline` JSON | 通用多段手势 | 实测都通过 drag、pinch 完成 | 可以不作为公开主命令 |

能否简化：可以，而且不丢能力。主要动作有三项：D1 的默认值改变；两个执行器只保留各自的调度部分，记账与对外摘要合并成一套；默认文本行不再 dump 内部回执。

---

## 6. 问题 3：typed Intent

- **相对 `open PACKAGE|--url` 和 `shell am start`，真实增量有四项：**
  1. SEND + EXTRA_STREAM 能让接收方读到流。`am start` 不迁移 ClipData；fixture 的 stream-on 与 stream-off 对照为实测。
  2. 在派发前解析出具体 handler；遇到 Resolver 时返回 not sent 和候选列表，不会把选择框留在屏幕上。
  3. 没有 shell 引号和类型转义问题，extra 类型带校验。
  4. 在 VD 上能做 task 预检，这部分属于方向 38。
- **低价值部分：**
  - `landing`：通过 before/after 的 root task 差分推断落点。实测多次为 unknown（两台 VIEW、四格中三格 DELIVERED_TO_TOP），而动作后观察本来就能给出前台包。规则说明在 `actions.md:315-326` 用了 12 行文档，代码集中在 `ActivityTasks.kt:192-303` 和 `ActivityLaunch.kt:338-…`。建议删掉 landing，或只保留"前台 root 的包和 Activity"一行。
  - `launchCodeName` 名称表：可以保留，成本小。
- **VLC SEND 未播放**：fd 已读取并 opened，随后 EOF / no demux，不是授权问题，应保留为业务缺口，不能算产品成功。原因未查明，可能与 MediaStore 流的 seek 能力有关，这是推测。
- **结论：** 保留 typed 字段、解析和 SEND 迁移（改用平台实现，D4）；收缩 landing。预计删 300–400 行。

---

## 7. 问题 4：结果语义

| 通道 | done 的含义 | 不一致时 | 只为极窄区分而设的规则 |
|---|---|---|---|
| fill/check | 新读到的 AX 值精确相等 | FAILED | 写入后允许改名的身份例外 |
| range | 新读到的 current 等于发出的 Float | FAILED | min/max/current 用字符串表示 NaN/∞ |
| IC commit | 局部镜像与插入位置吻合 | **UNKNOWN**（永不 FAILED） | — |
| IC select | 区间吻合 | FAILED，或"与 before 相同"时为 UNKNOWN | **是** |
| 借贴 | PASTE 回执 + 选区插入吻合 + 收尾完成 | FAILED；收尾未知时 UNKNOWN；PASTE 未发时也是 FAILED | **是**（D8） |
| keys/gesture | 平台全部返回 true | 有前缀时 UNKNOWN | — |
| open | 0..99 | FAILED；未解析时 not sent | 另附 landing 分列 |

**会改变 LLM 下一步的区分：**
- not sent 还是"可能已有副作用"：决定能否直接换通道。
- done 还是"没有确认"：决定是继续还是先观察。
- 局部核值的 `actualValue` 或 `after` 预览：能直接据此判断成功。

**基本是噪声的：**
- failed 与 UNKNOWN 的差别（两者都停批，下一步都是重新观察）；
- `attempted`、`callReturned`、`captureValid`、`dispatchStarted` 四者并列；
- `successfulPrefix` 的起点、`omitted`、`lastSkippedOffsetMs`、`syncReturned`、`asyncQueued`；
- `emptyBasis` 的 12 个字段；
- landing 的 root 列表；
- `verificationCutoffMs`。

建议：项目层面的四态可以保留，但每个通道必须遵守同一条映射：发出后只有精确匹配才是 done；已知不同为 failed；读不到为 UNKNOWN。IC commit 和借贴（D8）要对齐这条规则。默认文本输出只给出上面三类决策事实。

---

## 8. 问题 5：有界滚动查找

- 与 `scroll` + `find` 的关系：LLM 循环每一步都要交付整棵树（Chrome 页面约 4,225 节点）并消耗一轮对话；scroll-until 在设备端循环，只交付最后的观察和 REF。8192 节点预算决定了单次采样的上限，但不能替代滚动。
- 仍然必要。证据：两个候选业务臂中智能体自发使用。Algiers 负例中，默认 5s 预算只够 1 步（每步采样 1.3–4.7s），说明默认预算偏紧。建议在文档中给出"长页面加大 `--timeout`"的提示，不必改机制。
- 一个可选改进（设计建议）：容器子树里已有、只是不可见的唯一匹配目标，可以返回 REF 并标 `visible=false`，交给 `reveal` 处理，从而少滚动几步。实现时要保持"未发现不等于不存在"的现有语义。

---

## 9. 建议重新裁决的事项

### R1 IC（现裁决 U18 "IC C"）

- **A 维持现状。** 优点：已经实现并在 3 台设备上跑过。缺点：约 780 行主代码、AIDL、需要用户手动启用服务、main-only、Termux 结构性 UNKNOWN、done 信息量低。
- **B 删除 IC；`select REF START END` 改走 AX `ACTION_SET_SELECTION`；Termux 等终端用 `clipboard set` + `key ctrl+alt+v`；BrowserAccessibilityService 只保留 Chrome 声明职责。** 优点：减少一个跨进程桥、一种生命周期、两个命令和一套结果规则；select 可带 REF、可用于 VD。缺点：丧失只吃 IC 的编辑器（目前没有正例），失去"不用剪贴板"的 Unicode 路径。
- **C 只保留 commit，删除 select 的三态规则；select 改走 AX。** 是 A 和 B 的折中，但维护面依旧在。
- **D 冻结（不纳入首批合入），先补证，再在 B 与 C 之间选。**
- **推荐 D → B。** 剩余未知：Termux 在 AndroidCtl 注入的 ctrl+alt+v 下能否 doPaste；Chrome 和 Contacts 的 AX SET_SELECTION 能否被采纳。

### R2 借用剪贴板 PASTE（现裁决 U23，剪贴板 D/B）

- **A 维持现状**（354 行状态机，含当次 null 准入、所有权判定、条件恢复/清除/跳过）。
- **B 简化：** 借写 → PASTE → 局部核值 → 原剪贴板非 null 时写回原 ClipData；原剪贴板为 null 或未知时不再写，文档写明"剪贴板保留本次文本"。去掉空剪贴板准入、所有权三分支和清除路径，约 100 行。
- **C 删除：** 文档化 `clipboard get`（可选）→ `clipboard set TEXT` → `action REF --action-id 32768` →（可选）`clipboard set 原文本`。
- **D 并入 `type` 的派发前预选**（keys 不可映射时用 paste），实现采用 B。
- **推荐 C；如果用户看重"不破坏手机上的剪贴板"，选 B 或 D。** A 的额外复杂度防的是"借贴的约 1 秒窗口内用户恰好复制"和"读权限被拒时的 null"。前者包内零证据，后者在 B、C 下也只是"不恢复"，不会造成错误写入。

### R3 坐标手势默认 UID 定向（来自 B §7.1/7.2 的 U21 实现选择）

- **A 维持现状。**
- **B 坐标默认不定向，`--expect` 才定向；节点 touch 和 REF key 维持定向。**
- **C 维持定向，但 mismatch 时给出候选包。** 平台没有公开 API 能取得窗口 owner UID，实现成本不低。
- **推荐 B**（依据见 D1）。剩余未知：真机上 IME 和系统 UI 被误拒的频率。包内未单测点击软键盘，但从 AOSP 语义可以确定会误拒。

### R4 结果语义收敛（跨通道）

建议在一次小提交中完成：IC 删除或对齐（D6）、借贴 D8 改为 not sent 语义、keys 与 gesture 统一 `input` 摘要、默认文本行瘦身（D11）。这些不改变任何能力。

---

## 10. 如果只能保留一半输入面

**保留（按价值）：**
1. `fill`（含局部核值，方向 16）；
2. `type --via keys` 与 `key` 组合键（27）；
3. `click --touch`（23）；
4. `reveal` 与 generic `action`（24）；
5. `scroll-until`（20）；
6. `range`（25）；
7. `drag`/`pinch`（26，`timeline` 不公开）；
8. `clipboard get/set/clear`（35）。

**删除或冻结：**
- IC `editor commit/select`（29）；
- 借贴 `--via paste`（36，改用组合）；
- typed Intent 中的 landing（28 的一半）。

理由：被保留的每一项都有真实 App 正例，且是其他命令无法组合出来的；被删除的每一项都能用保留下来的命令组合覆盖到可证明的场景，或者包内没有它独有价值的正例。

---

## 11. 对既有内部 review 与交付方声明的对照

- **`reviews/final-maintainability.md`** 称"不采纳按行数去掉 clipboard/IC/VD"，并认为 KeySequence 与 PointerGesture "分别持真实前缀，比泛化执行器更清楚"。
  - 不同意第一点：本报告的依据不是行数，而是替代路径（Termux ctrl+alt+v、AX SET_SELECTION、generic action PASTE），这些替代在该 review 中都没有被考察。
  - 部分同意第二点：调度不必合并，但对外记账格式不该刻意分成两套（D10）。
- **28 的 runtime-review** 称"没有根据新增等待、重试"，这一点同意。但它没有讨论"Termux 结构性 UNKNOWN 意味着 IC 在旗舰场景不可批处理"这一方向性后果。
- **21 global-key-fix review**：同意恢复全局键不定向。但该 review 认为"运行中未发现要求扩充职责的证据"，同时又接受了 photopicker 必须用 `--expect` 的事实，没有反思默认坐标定向的成本（D1）。
- **上次外评 `upload/androidctl-candidate-review-final.md`**：
  - 同意其输入-1、3、4，通知-2、3，合并-10。
  - 它对 IC 的建议偏保守（"先修状态映射，路线等补证"），未发现 Termux ctrl+alt+v 这条替代，也没有把借贴与 `clipboard set` + `action PASTE` 做比较。
  - 它的"通知-5：URI grant 对 EXTRA_STREAM 可能不生效"，候选已经用 ClipData 迁移解决，但解决方式偏离平台（D4）。
- **研究报告 `research/input/report.md`** 第 39 行把 Termux 作为 IC 输入增量的证明，前提是"fill 和所选 keys 都不行"。它没有考察 Termux 自身的剪贴板快捷键；在第 37 行它也只检查了 AX PASTE 和长按菜单，没有看 Termux 的硬件键盘快捷键。

---

## 12. 未覆盖范围与必要补证

**未读或未细读：**
- `UiActions.kt` 只细读了 1–380 行（touch、paste、key、range、reveal 入口），fill 序列与行点击只略读；
- `ScrollContainer.kt`、`ScrollProgress.kt`、`PointerPlan.kt`、`ActivityTasks.kt`、`ClipboardOperations.kt`、`ClipboardProjection.kt` 未逐行读；
- host 端只读了 `ui_commands.py` 的命令定义和 `ui_client.py` 的结果渲染，`activity_intents.py`、`editor_input.py`、`input_keys.py`、`pointer_plan.py` 未读；
- `docs/reference/ui.md` 未读；
- `reviews/final-android-contracts.md`、`final-api34-capabilities.md`、`external-final-integration.md` 未读全文。

**会改变判断的补证（每项约一条命令）：**

1. Termux 0.118.3：`clipboard set "ACP11中文"` → `key ctrl+alt+v`（不带 REF），然后截图核对。
   - 成功 → R1 选 B，IC 失去 Termux 依据。
   - 失败（例如注入事件的 meta 没被 Termux 认作 Ctrl+Alt）→ R1 选 C，保留 commit。
2. AOSP Contacts 字段和 Chrome 网页 input 上 AX `ACTION_SET_SELECTION(start,end)`，随后读回 `textSelectionStart/End`。
   - 两者都能采纳 → select 改走 AX。
   - Chrome 不采纳 → 仅 Chrome 场景保留 IC select 的理由。
3. 真机（非模拟器）上一次 `clipboard set`，截图确认是否出现 SystemUI 剪贴板浮层。决定 D7 的优先级。
4. 默认 `tap` 点击软键盘某个键（无 `--expect`），确认 targetMismatch。决定 R3 的紧迫度。
