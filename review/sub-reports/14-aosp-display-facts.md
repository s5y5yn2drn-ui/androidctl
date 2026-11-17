# 后台屏平台事实核对（AOSP 14/15/16 一手源码）

范围：shell UID（com.android.shell）用 DisplayManager 创建的虚拟屏（候选称 work display / virtual target）。
上游源码已实际下载（googlesource `?format=TEXT`），本地副本在
`/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents3/work14/src/<tag>/<path>`。
除特别注明外，路径均相对 `platform/frameworks/base`，行号取自对应 tag。`InputDispatcher.cpp` 来自 `platform/frameworks/native/services/inputflinger/dispatcher/`。
另取 Android 16 QPR2 分支提交 `45034f06…`（研究包引用的同一提交）的 `LogicalDisplayMapper.java` / `DisplayGroupAllocator.java`，用于核对研究包 E7 的解释。
候选源码行号取自 REPO `3f45af6`。
证据标签：【上游】已下载核对的 AOSP 源码；【源码】候选源码；【实测】包内运行原件（注明设备与版本）；【建议】设计建议；【需设备】只能上设备确定。

---

## 0. 结论摘要（先给答案）

1. **候选的建屏 flag 组合与平台规则一致，无需改动**：`PUBLIC|OWN_CONTENT_ONLY|SUPPORTS_TOUCH|DESTROY_CONTENT_ON_REMOVAL|TRUSTED|OWN_FOCUS|STEAL_TOP_FOCUS_DISABLED`。这些 flag 的处理分两类：`TRUSTED` 缺权限时抛 `SecurityException`；`OWN_FOCUS` 和 no-steal 依赖不满足时被静默去掉（14/15 在 VirtualDisplayAdapter 里，16 移到了 DMS）。所以建屏后读回 flag 是必要的，候选也做了。三个版本的规则相同，只是检查的位置变了。【上游】
2. **后台屏上任何编辑框都拿不到 IME，键盘也不会弹到主屏**。IMMS 只给“顶层焦点屏”的客户端建立输入会话，no-steal 屏永远不会成为顶层焦点屏，所以 `startInput` 返回 `INVALID_DISPLAY_ID`。IME policy（LOCAL/FALLBACK/HIDE）在这条路径上不起作用。真人在主屏的输入法和输入会话不受影响。【上游】AOSP16 AVD 实测与此一致。【实测】
3. **在后台屏上插入中文（无法由按键映射的文本）可以做到，只有两条路**：(a) `ACTION_SET_SELECTION` 定位光标，再用 `ACTION_PASTE`（或 Ctrl+V 按键）粘贴剪贴板内容；(b) `ACTION_SET_TEXT` 写入重组后的整段文本。这两条在源码上都不依赖 IME。另外两条走不通：按键注入只接受 DOWN/UP，没有 `ACTION_MULTIPLE`；无障碍 `InputMethod` 的 InputConnection 只绑定顶层焦点屏（也就是真人正在用的主屏编辑器）。候选把“当前编辑器”通道限定为主屏，与平台一致，而且是唯一可行的做法。具体 App 的覆盖面需设备。
4. **后台屏的电源完全跟随默认显示组，而且三个版本行为不同**：
   - 14/15：只要有 Surface，虚拟屏的 `Display.getState()` 一直是 ON，主屏熄灭也不变。真正让屏上 Activity 停下来的是 **keyguard sleep token**：锁屏会施加到所有不是 ALWAYS_UNLOCKED 的屏。
   - 16：在 `correct_virtual_display_power_state` 开启时，虚拟屏状态跟随所在组。刚建屏时是 UNKNOWN，组睡眠时变 OFF，WM 随即给它加 off sleep token。这就是 API 37 上“建屏后短暂 UNKNOWN”的来源。【上游】
5. **候选“主屏亮且解锁”的前提在平台上是必要的**。主屏锁屏时，虚拟屏会拿到 keyguard sleep token（Activity 被停掉），PUBLIC 屏上还会显示 SystemUI 的 KeyguardPresentation。16 上主屏熄灭时虚拟屏也随之 OFF。【上游】
   - 放宽只能靠 `OWN_DISPLAY_GROUP + ALWAYS_UNLOCKED`，外加一把绑定虚拟屏 displayId 的屏幕 WakeLock。shell 在 AOSP 上有这两项权限。
   - 但 16 QPR2 的 `separate_timeouts` 分组路径会无视 OWN_DISPLAY_GROUP，把虚拟屏放回 group0（研究包 E7 实测与此吻合）。部分 ROM 还会删掉这些权限（MuMu 实测）。
   - 所以“放宽”不是可靠的跨版本能力，不建议首版做。
6. **默认组里的虚拟屏会帮主屏“续命”**：注入到虚拟屏的输入按 displayId 给所在组记 user activity，无障碍动作则固定记到默认组。只要 agent 在后台持续操作，主屏就不会因超时熄灭。这是真人可感知的副作用，平台层面避不开（除非建独立组），文档应如实写明。【上游】是否明显需设备。
7. **任务迁移**：
   - 以 `launchDisplayId=虚拟屏` 启动时，`findTask` 会**跨所有屏**找“理想匹配”的已有 task，即 `realActivity` 与启动组件相同的 task。找到就把它**整体 reparent 到虚拟屏**，并返回 `START_TASK_TO_FRONT` 或 `START_DELIVERED_TO_TOP`（这两个返回码都不能当作“新建”的证据）。
   - 只是 affinity 相同的 task，只会在虚拟屏本屏内匹配。
   - 候选按包名做的启动前预检比平台的迁移条件更宽，方向正确。但它用 `getTasks` 做第二来源，再加“两来源不一致→unknown”分支，这是多余的。
8. **销屏**：
   - 带 `DESTROY_CONTENT_ON_REMOVAL`（或 private 屏）时，屏上所有 task 会走 `destroyIfPossible`：App 正常收到 onDestroy，未保存的界面状态丢失，进程不被杀。
   - 不带这个 flag 的 PUBLIC 屏，task 会被 reparent 到主屏**最上层**，直接出现在真人面前。
   - 候选选择销毁，与平台一致。包内 E4 实测“迁入的旧 task 随销屏被销毁”与源码一致。
   - 16 新增了 DisplayWindowSettings 覆盖（显示设置文件可以改 removeMode）。
9. **确定缺陷（低影响）**：`DeviceFacts.kt:39` 把 displayId 传给 `ITrustManager.isDeviceLocked(userId, X)`。这个参数在 15/16 是 **deviceId**，非 0 时恒返回 false，所以虚拟屏 target 的 `deviceLocked` 事实在 15/16 上恒为“未锁”。门控用的是主屏事实，不受影响，但对外发布的 target 事实是错的。修法：第二个参数永远传 0。【上游+源码】

---

## A. 建屏与 flags

**结论**：候选请求的 7 个 flag 与平台规则一致，读回的 3 位（TRUSTED/OWN_FOCUS/no-steal）也正好是会被静默去掉的那几项。shell 在 AOSP 14/15/16 上都持有所需权限，但关键的两项（ADD_TRUSTED_DISPLAY、ADD_ALWAYS_UNLOCKED_DISPLAY）在 manifest 里被注释为“CTS 测试所需”，ROM 删掉它们并不意外。持续 drain 的 ImageReader 是为了“有 Surface 才算 ON”这条硬规则。API 37 上短暂的 UNKNOWN 来自 16 引入的虚拟屏电源状态修正。

### A1. Shell 权限（`packages/Shell/AndroidManifest.xml`）【上游】

| 权限 | 14r1 | 15r1 | 16r1 | 用途 |
|---|---|---|---|---|
| INTERNAL_SYSTEM_WINDOW | :170 | :173 | :175 | 允许启动到任意屏（ATS）、`setDisplayImePolicy`、`moveRootTaskToDisplay` |
| INJECT_EVENTS | :171 | :174 | :176 | `injectInputEventToTarget` |
| CAPTURE_VIDEO_OUTPUT / READ_FRAME_BUFFER | :194/:195 | :197/:198 | :200/:201 | 镜像屏 / `captureDisplay` |
| DEVICE_POWER / WAKE_LOCK | :197/:292 | :200/:299 | :203/:306 | 电源 |
| REAL_GET_TASKS / MANAGE_ACTIVITY_TASKS | :141/:258 | :143/:265 | :143/:272 | `getTasks` / `getAllRootTaskInfos` |
| ADD_TRUSTED_DISPLAY、ADD_ALWAYS_UNLOCKED_DISPLAY、CREATE_VIRTUAL_DEVICE | :714–716 | :746–748 | :974–976 | 可信屏、锁屏豁免屏 |
| CAPTURE_SECURE_VIDEO_OUTPUT | 无 | 无 | 无 | 所以不能建 SECURE 屏 |

关于 ADD_TRUSTED_DISPLAY 和 ADD_ALWAYS_UNLOCKED_DISPLAY：
- 14r1 `:713` 的注释是 `<!-- Permissions required for CTS test - CtsVirtualDevicesTestCases -->`，15 和 16 相同。也就是说，它们是为测试加给 shell 的，不是正式合约。
- 两项的保护级别都是 `signature|role`（`core/res/AndroidManifest.xml` 14r1 :7286–7291）。
- 包内 MuMu（Android 15）实测：shell 包清单里没有这两项，建屏抛 `Requires ADD_TRUSTED_DISPLAY`。【实测】

### A2. `DisplayManagerService.createVirtualDisplayInternal` 的 flag 规则【上游】

位置：14r1 `:1384`，15r1 `:1534`，16r1 `:1804`；对应 Adapter 为 `VirtualDisplayAdapter`（VDA）。

| flag | 规则 | 缺条件时 | 位置 |
|---|---|---|---|
| packageName | 必须属于调用 UID | 抛 SecurityException | DMS14 :1388；16 :1808 |
| PUBLIC | 自动加 AUTO_MIRROR | — | DMS14 :1417–1424；16 :1841–1852 |
| OWN_CONTENT_ONLY | 去掉 AUTO_MIRROR | —（候选需要它，否则 PUBLIC 屏变成镜像屏） | 14 :1426–1428；16 :1854–1859 |
| AUTO_MIRROR | 需要 CAPTURE_VIDEO_OUTPUT；同时去掉 OWN_DISPLAY_GROUP | 抛异常 / 静默去掉 | 14 :1429–1431、:1464–1471 |
| SECURE | 需要 CAPTURE_SECURE_VIDEO_OUTPUT | 抛异常 | 14 :1473–1479 |
| TRUSTED | 需要 ADD_TRUSTED_DISPLAY | **抛异常** | 14 :1481–1488；15 :1641；16 :1928–1936 |
| OWN_DISPLAY_GROUP | 需要 ADD_TRUSTED_DISPLAY（非 VDM） | **抛异常** | 14 :1490–1498；15 :1659–1667；16 :1946–1952 |
| ALWAYS_UNLOCKED | 需要 ADD_ALWAYS_UNLOCKED_DISPLAY | **抛异常** | 14 :1500–1508；15 :1669–1677；16 :1962–1970 |
| ALWAYS_UNLOCKED | 还要求 OWN_DISPLAY_GROUP 或 DEVICE_DISPLAY_GROUP | **静默去掉**（带日志） | 14/15 VDA :521–532 / :524–534；16 移到 DMS :1954–1960 |
| OWN_FOCUS | 要求 TRUSTED | **静默去掉** | 14 VDA :536–543；15 VDA :539–545；16 DMS :1972–1977 |
| STEAL_TOP_FOCUS_DISABLED | 要求 TRUSTED 且 OWN_FOCUS | **静默去掉** | 14 VDA :544–553；16 DMS :1979–1984 |
| SHOULD_SHOW_SYSTEM_DECORATIONS | 非 trusted 时去掉；否则需要 INTERNAL_SYSTEM_WINDOW | 去掉 / 抛异常 | 14 :1510–1526；16 :1986–2004 |
| DESTROY_CONTENT_ON_REMOVAL | 无权限要求 | — | 14 VDA :512–514 |
| SUPPORTS_TOUCH | 无权限要求；影响 `DisplayDeviceInfo.touch` | — | 14 VDA :557–558 |
| 不带 PUBLIC | 成为 FLAG_PRIVATE（14/15 还带 NEVER_BLANK） | — | 14 VDA :474–477 |

**与候选对照**：
- 候选请求的 flag 在 `WorkDisplay.kt:37–46`，读回的 3 位在 `:65–70`，即 `Display.FLAG_TRUSTED=1<<7`、`FLAG_OWN_FOCUS=1<<11`、`FLAG_STEAL_TOP_FOCUS_DISABLED=1<<12`（`core/java/android/view/Display.java` 14r1 :296/:336/:364）。读回值用的是 Display 位，没有与 `VIRTUAL_DISPLAY_FLAG_*`（`DisplayManager.java` 14r1 :218–476）混用。
- 实测读回 `actualFlags=6272=0x1880`（API34 AVD 和 API37 都是如此，requested=83273=0x14549），位值吻合。【实测】
- 可以更简单（很小的一点）：no-steal 只有在 TRUSTED 和 OWN_FOCUS 都在时才会保留，所以只读回 `FLAG_STEAL_TOP_FOCUS_DISABLED` 一位就够了；TRUSTED 缺权限时在创建阶段就已经抛异常。【建议】
- 不加 SHOULD_SHOW_SYSTEM_DECORATIONS 是对的：不会出现桌面、导航栏，IME policy 默认值也不受影响。
- 候选没有请求 TOUCH_FEEDBACK_DISABLED（研究原型请求了）。它只影响“显示触摸”开发者选项在该屏的绘制，可加可不加。

**三版差异**：权限要求不变。依赖检查在 16 从 VDA 前移到 DMS，静默去掉的语义不变。16 的 VDA 把 FLAG_OWN_CONTENT_ONLY 改为按非镜像无条件设置（16 VDA :611–618）。读回结论在三个版本上相同。

### A3. 没有 Surface 时的显示状态，以及为什么要持续 drain 的 ImageReader【上游】

- **14/15**：`mIsDisplayOn = surface != null`（14 VDA :296；15 :297），`mInfo.state = mIsDisplayOn ? ON : OFF`（14 :560；15 :563）。
  - `requestDisplayStateLocked`（14 :384–395）只发 paused/resumed 回调，**不改** `mInfo.state`。
  - 所以 14/15 上：有 Surface 时虚拟屏一直是 ON，**主屏熄灭也不变**；没有 Surface 就是 OFF。
- **16**：
  - `isNeverBlank` 的判断是“非 PUBLIC 且非镜像”（16 VDA :325–329）。PUBLIC 屏不是 never-blank。
  - 在 `Flags.correctVirtualDisplayPowerState()` 开启时，非 never-blank 屏的初始状态是 `STATE_UNKNOWN`，等第一次 `requestDisplayStateLocked` 才确定（:390–400）。之后状态由所在显示组驱动（:489–527）。
  - Surface 为 null 时，状态一律是 OFF（:671–676）。
  - 该 aconfig 标志在 `core/java/android/companion/virtual/flags/flags.aconfig`（16r1 :123，`purpose: PURPOSE_BUGFIX`）。标志在量产机上的取值需设备；AVD 上的行为与“开启”一致。
- **WM 侧**：非默认屏的 DisplayInfo 状态为 OFF 时，会加 display-off sleep token（14 `wm/DisplayContent.java:6017–6030` `onDisplayChanged`；16 :6105–6116），`shouldSleep()` 随之为真（14 :6392），屏上 Activity 被停掉。

所以：
- **必须有 Surface**，否则屏是 OFF，Activity 被停。
- **Surface 必须被持续消费**。ImageReader 只有 `maxImages` 个缓冲（候选设为 3，`WorkDisplay.kt:49`、`:147`）。不 acquire/close，生产端拿不到空缓冲，SurfaceFlinger 对该屏的合成会受阻。这一点是 native 行为，未逐行核实。
- 候选的截图走 `IWindowManager.captureDisplay`（需要 READ_FRAME_BUFFER，14 WMS :9429–9433；16 :10134–10138）。这是对图层的抓取，与 ImageReader 的内容无关。因此 ImageReader 只是一个“维持 ON 的消费端”。
- 【建议，需设备】这个消费端可以考虑用很小的缓冲尺寸来省内存（约 1080×2400×4×3≈31MB）。是否影响该屏的布局或合成，必须上设备验证。不验证就保持现状。

API 37 上“建屏后短暂 UNKNOWN”就对应 16 的初始 UNKNOWN（包内 `api37-readiness/runtime-api37/result.md` 记录 `added UNKNOWN→.884 ON`）。候选在 `WorkDisplay.kt:71–77` 在预算内等待 UNKNOWN 结束，这个处理是对的。14/15 上建屏即为 ON，不会进入这个循环。

---

## B. 输入法落在哪块屏，以及四条文本通道

**结论**：no-steal 屏上的编辑框请求输入法时，IMMS 直接拒绝绑定（`INVALID_DISPLAY_ID`）。IME 不出现在任何屏上；主屏键盘的状态和真人的输入会话不变。IME policy 只决定“获准后键盘显示在哪”，在这里不起作用。四条文本通道中，SET_TEXT 和 PASTE 在后台屏上成立；按键只能输入可映射的字符；无障碍 InputMethod 只能拿到主屏（真人）的编辑器。

### B1. IME policy 默认值与设置权限【上游】

- 默认值：
  - `DisplayWindowSettings.getImePolicyLocked`：默认屏为 LOCAL，其他屏没有设置时为 **FALLBACK_DISPLAY**（14 :229–239；15/16 同构）。
  - `DisplayContent.getImePolicy`：非 trusted 屏一律 FALLBACK；“强制桌面模式”的外接屏把 FALLBACK 提升为 LOCAL（14 :4203–4214；16 :4313–4325）。
- trusted 本身不会让默认值变成 LOCAL；system decorations 也不改变这个默认值。
- `IWindowManager.setDisplayImePolicy`：需要 INTERNAL_SYSTEM_WINDOW，且目标屏必须是 trusted，否则抛异常（14 WMS :7607–7630）。shell 满足这两个条件，可以通过 binder 调用。
- `cmd window` 没有对应子命令（14/16 `WindowManagerShellCommand` 的命令列表里没有）。
- `computeImeDisplayIdForTarget` 的映射：LOCAL→本屏，HIDE→不显示，其他→默认屏（14 IMMS :2739–2755）。

### B2. 为什么 IME 根本不启动【上游】

1. `WindowManagerInternal.hasInputMethodClientFocus` 要求请求的 displayId 等于 `getTopFocusedDisplayContent()` 的 id，否则返回 `INVALID_DISPLAY_ID`（14 WMS :8011–8035；15 :8263–8284；16 :8223–8245）。
2. IMMS 的 `startInputOrWindowGainedFocusInternalLocked` 遇到这个结果直接返回 `InputBindResult.INVALID_DISPLAY_ID`，不会改动当前客户端（14 IMMS :3763–3782；15 :3780–3799；16 :3811–3829）。
3. 顶层焦点屏的计算方式：按 z 序从上往下，第一个有焦点窗口或焦点 App 的屏（14 `RootWindowContainer.java:448–483`）。新屏一律以 `POSITION_BOTTOM` 加入（:1275、:1368）。no-steal 屏的 `mDontMoveToTop` 为真（14 `DisplayWindowSettings.java:294`），`moveDisplayToTopInternal` 也拒绝把它移到顶层（14 WMS :3138–3155）。
4. `Display.java` 对 FLAG_STEAL_TOP_FOCUS_DISABLED 的说明（14r1 :356–359；16r1 :374–377）原文写着：“The framework only supports IME on the top focused display… Enabling this flag on a display implicitly disables showing any IME.”

**对真人的意义**：
- 主屏的 IMMS 客户端、键盘可见性都不变；不会出现键盘被收起或突然弹出。
- 虚拟屏上的 App 调用 `showSoftInput` 也无效，因为它的 IMM 没有被服务。
- 包内 AOSP16 AVD E1b 实测：VD 上没有 IME 窗口，主屏 `mCurTokenDisplayId=0`、`mInputShown=true` 保持，主屏 oracle `focusLost=0`。【实测】

**唯一的边角**：当主屏上既没有焦点窗口、也没有焦点 App 时（RWC :462–470），顶层焦点屏会落到下一块有焦点的屏，也就是虚拟屏。此时它的编辑框可以拿到 IME，而 FALLBACK 会把键盘放到主屏。正常使用中主屏总有桌面或锁屏通知栏窗口作为焦点，所以这种情况很罕见。不建议为此预先加 HIDE；如果设备上真的观察到，再用 `setDisplayImePolicy(vd, HIDE)` 兜底。【建议】

### B3. 每屏独立焦点、per-display focus 与 InputConnection【上游】

- `DisplayContent.hasOwnFocus()` 的条件是 `mPerDisplayFocusEnabled || FLAG_OWN_FOCUS`（14 :6499–6502）。`config_perDisplayFocusEnabled` 在 AOSP 默认是 false（`core/res/res/values/config.xml` 14r1 :2454；16r1 :2730）。
- 有自己焦点的屏会单独算出 `mCurrentFocus`（14 :3790–3793）。所以虚拟屏可以有自己的焦点窗口；这个窗口会收到 window focus，View 也能取得焦点。
- 但这块屏上的编辑框**拿不到 InputConnection**（见 B2）。
- 真人同时在主屏编辑框里打字：输入会话一直绑定主屏，两边互不干扰。

### B4. 四条文本通道在后台屏上的可用性【上游】

| 通道 | 平台路径 | 后台屏可用？ | 中文局部插入 |
|---|---|---|---|
| (1) 无障碍 `ACTION_SET_TEXT`（fill） | TextView 只检查 `isEnabled()` 和 EDITABLE，然后直接 `setText`（14 `TextView.java:14315–`；16 :15005–） | **可用**；不需要焦点，不需要 IME。AVD 实测【实测】 | 可以，但只能整段替换：读出当前文本，在应用侧拼接后写回，再用 `ACTION_SET_SELECTION`（14 :14276）恢复光标。密码框读不到原文；自绘编辑器的 SET_TEXT 语义各不相同【需设备】 |
| (2) `ACTION_PASTE` + 全局剪贴板 | TextView 要求 `isFocused() && canPaste()`（14 :14262–14266）。App 进程读剪贴板时，ClipboardService 要求 `isUidFocused`（14 `ClipboardService.java:333`）；WMS 的 `isUidFocused` **遍历所有屏**的 `mCurrentFocus`（14 WMS :7997–8008；16 :8209–8220）。OWN_FOCUS 屏上的焦点窗口满足这个条件 | **源码成立** | **可以**：先用 SET_SELECTION 定位光标，再粘贴，是真正的局部插入。副作用：粘贴提示 toast 显示在**默认屏**（14 `ClipboardService.getToastContexts` :1442–1476，非 VDM 设备时用系统 context）；Android 13+ SystemUI 在剪贴板变化时会在主屏弹出剪贴板浮层，除非来源是 shell 且 ClipDescription extras 带 `com.android.systemui.SUPPRESS_CLIPBOARD_OVERLAY=true`（SystemUI `ClipboardListener.java` 14r1 :53–54、:118–131）。它还会覆盖真人的剪贴板 |
| (3) 向虚拟屏 displayId 注入 KeyEvent | 按键事件以 `KeyEntry.displayId` 选择该屏的焦点窗口（`InputDispatcher.cpp` 14 :2017–2024、:2076–2136）；`targetUid` 的校验在 :575–583。注入只接受 DOWN/UP（`checkKeyAction` 14 :133–141；16 :186–194），**`ACTION_MULTIPLE`（字符串）被拒绝** | **可用**；文本输入只限 KeyCharacterMap 能映射的字符。TextView 在没有 IME 时由 KeyListener 处理【需设备】 | **不能**输入中文。但 Ctrl+V 组合键会走 TextView 快捷键粘贴，可以作为 (2) 的按键版本【需设备】 |
| (4) AccessibilityService `InputMethod`（候选的 editor IC 通道） | IMMS 只在当前客户端建立后调用 `AccessibilityManagerInternal.startInput(mCurRemoteAccessibilityInputConnection, mCurEditorInfo…)`（14 IMMS :2489–2502；16 :2034）。当前客户端只能是顶层焦点屏上的编辑器 | **不可用**：它拿到的永远是主屏，也就是真人正在用的编辑器 | 不适用 |

**回答“后台屏能否插入中文”**：能。首选 SET_SELECTION + PASTE（或 Ctrl+V），备选 SET_TEXT 写入整段重组文本。两者在源码上都成立；对具体 App（WebView、Compose、Flutter、自绘编辑器）的覆盖需设备。按键注入和无障碍 IC 在平台上就走不通。

**候选**：
- `UiBackend.kt:239–247` 把 editor 通道限定在主屏，与平台一致。
- `BrowserAccessibilityService.kt:18–61` 的 InputMethod 在后台会话里本来就不可能指向虚拟屏，所以“主屏限定”无需额外校验。
- 用户已决定“改用 `clipboard set` + 通用 PASTE 动作”，平台支持这条路线。【建议】实现 `clipboard set` 时给 ClipData 的 extras 加上 SUPPRESS 标记，避免主屏弹出浮层，前提是设备验证有效（见末节）。

---

## C. 电源组、熄屏与锁屏

**结论**：不带 OWN_DISPLAY_GROUP 的虚拟屏属于默认组。
- 主屏锁屏时，虚拟屏会拿到 keyguard sleep token，Activity 停止；PUBLIC 屏上还会显示 KeyguardPresentation。
- 主屏熄屏时：16 上虚拟屏随组 OFF 并 sleep；14/15 上屏状态仍是 ON，是否停止取决于 keyguard 是否出现。
- 所以候选“主屏亮且解锁”的前提是必要的。
- OWN_DISPLAY_GROUP + ALWAYS_UNLOCKED + 绑定 displayId 的 WakeLock 在 14/15/16r1 源码上可以实现锁屏和熄屏时继续工作，但 16 QPR2 的分组路径会让它退回 group0，ROM 也可能删掉权限，不建议首版做。

### C1. 显示组与电源组【上游】

- **分组**：
  - 14/15/16r1 的 `LogicalDisplayMapper.assignDisplayGroupLocked`：有 FLAG_OWN_DISPLAY_GROUP（或 groupName）时分到新组，否则进入 `DEFAULT_DISPLAY_GROUP`（14 :854–；16 :1052–1095、:1310–1328）。
  - 16 QPR2：`separate_timeouts` 开启且 groupName 为空时，先由 `DisplayGroupAllocator` 决定组类型；普通虚拟屏归为 PRIMARY，也就是 group0（QPR2 `LogicalDisplayMapper.java:1094–1100`、`:1351–1354`；`DisplayGroupAllocator.java:67–73`）。**这时 OWN_DISPLAY_GROUP 失效**，但 Display flag 读回仍然带着它。这与包内 E7 实测“flags=0x1d80 但 groupId=0”一致。【实测：AOSP16 AVD】
- **电源组**：
  - 每个显示组对应一个 PowerGroup，新建时为 AWAKE（14 PMS :738–757）。
  - 用户活动超时按组计算，各组使用同一个 screen_off_timeout（14 :2894–2910）。
  - 电源键的 `goToSleep` 只作用于默认组（14 :5891；16 :6219）。
- **WakeLock**：
  - 按 `getPowerGroupId()` 映射：displayId 为 INVALID 时得到 INVALID_DISPLAY_GROUP，**对所有组生效**（14 :5353–5364、:2700–2735）。
  - 带 displayId 的 WakeLock 只作用于该屏所在的组；需要使用隐藏的 `PowerManager.newWakeLock(int, String, int displayId)`（14 :1349）。
  - **包内研究“持锁会连主屏一起保亮”的源码原因**：E7 里锁绑定的 VD 实际在 group0，所以和主屏是同一组。
- **用户活动**：
  - 注入事件经 InputDispatcher 发出的 user activity 带 displayId，按组计入（14 PMS :1991–2018）。
  - 无障碍动作调用 `PowerManager.userActivity`，使用 system context 的 displayId，即 **默认组**（14 `AbstractAccessibilityServiceConnection.java:2107`；`PowerManager.java:1436–1442`）。
  - 推论：在默认组里，agent 的任何后台操作都会推迟主屏熄灭。【需设备确认量级】
- **输入门控**：
  - 注入的触摸在 14 上按**全局** interactive 门控（`com_android_server_input_InputManagerService.cpp` 14 :1370–）；16 改为**按屏**判断（`isDisplayInteractive` 16 :1921–1937）。
  - 注入的非唤醒按键在非 interactive 时仍然放行（`PhoneWindowManager.java` 14 :4033；16 :5228）。

### C2. 锁屏【上游】

- `ActivityTaskManagerService.setLockScreenShown` 对**所有屏**调用 `KeyguardController.setKeyguardShown`（14 :2960–2963；16 :3049–3052）。唯一跳过的是 `isKeyguardAlwaysUnlocked()` 的屏，即 FLAG_ALWAYS_UNLOCKED（14 `KeyguardController.java:168–174`；`DisplayContent.java:6484–6486`）。
- 屏上 keyguard 可见且未被遮挡时，该屏获得 keyguard sleep token（14 :530–545）。
- SystemUI `KeyguardDisplayManager.isKeyguardShowable`：默认屏、FLAG_PRIVATE、FLAG_ALWAYS_UNLOCKED 以外的屏都会显示 **KeyguardPresentation**（14 :120–152；16 :143–182）。候选的 PUBLIC 屏属于会显示的一类，所以锁屏时这块屏上会出现锁屏 Presentation 窗口。
- `ITrustManager.isDeviceLocked(userId, X)`：14 的第二个参数是 displayId（14 `TrustManagerService.java:1715`）；**15/16 改为 deviceId，非 0 直接返回 false**（15 :1832–1836；16 :1867–1871；AIDL 15/16 :37/:38）。这导致候选 `DeviceFacts.kt:39` 的缺陷（见 E2）。

### C3. 状态表（主屏状态 × 虚拟屏组合 → 后台屏行为）

记号：✓ 源码直接支持；✗ 源码上不成立；?= 需设备。“截图”指 `captureDisplay` 图层抓取；“树”指无障碍窗口与节点；“注入”分触摸和按键。

**组合 1：候选现行（默认组，无 ALWAYS_UNLOCKED）**

| 主屏 | 14/15 | 16（power-state 修正开启） |
|---|---|---|
| 亮且解锁 | 运行 ✓；截图 ✓；树 ✓；触摸 ✓、按键 ✓【实测 API34/36 AVD】 | 同左 ✓【实测】 |
| 亮但锁屏 | 屏状态 ON，但有 keyguard sleep token，Activity pause/stop ✓；KeyguardPresentation 覆盖该屏 ✓；截图内容是锁屏 Presentation 还是黑屏 ?；树中应用窗口是否还在 ?；注入落到 Presentation 或被丢弃 ? | 同左 |
| 熄屏，keyguard 已出现 | 同“亮但锁屏”；另外全局非 interactive，**注入的触摸被丢弃** ✓，按键仍放行 ✓ | 虚拟屏 OFF，off token 加 keyguard token，Activity 停止 ✓；触摸按屏判定后丢弃 ✓ |
| 熄屏，keyguard 未出现（无锁屏，或处在锁定延迟内） | **虚拟屏仍为 ON，没有 sleep token，内容继续运行** ✓（源码推断）；截图、树 ?；触摸被全局门控丢弃 ✓；按键、无障碍动作可达 ? | 虚拟屏 OFF，停止 ✓ |

**组合 2：OWN_DISPLAY_GROUP + ALWAYS_UNLOCKED，并持有绑定 displayId 的 SCREEN WakeLock（14/15/16r1）**

| 主屏 | 行为 |
|---|---|
| 亮且解锁 | 运行 ✓ |
| 亮但锁屏 | 没有 keyguard token，也没有 Presentation，继续运行 ✓；树、截图、注入 ✓（源码）? |
| 熄屏 | 虚拟屏组独立保持 AWAKE ✓；14 全局 interactive 仍为真，触摸放行 ✓；16 按屏判定为 interactive ✓；整体 ? |
| 16 QPR2（separate_timeouts）| 退回 group0，表现同组合 1，只是没有 keyguard 覆盖；E7 实测主屏熄灭时两屏都 OFF【实测】 |

**组合 3：只加 OWN_DISPLAY_GROUP（不加 ALWAYS_UNLOCKED）**
- 锁屏行为同组合 1。
- 熄屏时独立组可以保持亮，但锁屏一出现照样被停。实际收益只有“主屏熄灭但未锁”的那段时间。

### C4. 对候选前提的裁定

- **“主屏亮且解锁”是必要条件**：锁屏会覆盖所有非 always-unlocked 屏；16 上默认组熄屏，虚拟屏就 OFF。`DeviceFacts.kt:69–81` 与 `UiBackend.kt:70–79` 的门控方向正确。
- 可以更简单：
  - `targetInputReady` 还会额外读一次虚拟屏 `displayState==ON`（`UiBackend.kt:77`）。在 14/15 上它恒为 ON，没有信息量；在 16 默认组上，它等价于主屏组是否 AWAKE，已经被主屏条件覆盖。可以删掉，或只保留建屏时那一次检查。【建议】
- 放宽（组合 2）：
  - 需要 shell 的 ADD_TRUSTED_DISPLAY 和 ADD_ALWAYS_UNLOCKED_DISPLAY，并常持一把绑定 displayId 的 WakeLock。
  - 后果一：锁着的手机上可以远程操作 App，授权语义变了。
  - 后果二：这把锁让系统不能 suspend，耗电。
  - 后果三：16 QPR2 及以后无法保证有独立组，flag 读回不能作为证据，必须读 `dumpsys display` 中的实际 groupId。
  - 建议首版不做；如果将来要做，作为单独的能力，在设备上读出实际组号后才开放。【建议】

---

## D. 任务迁移与销毁

**结论**：
- shell 启动到虚拟屏的权限检查总是通过（因为持有 INTERNAL_SYSTEM_WINDOW）。
- 平台不检查可调整大小；只有设备不支持副屏时会静默改回主屏。
- 目标 App 在主屏已有 `realActivity` 相同的 task 时，这个 task 会被**搬到虚拟屏**，不会新建。
- `moveRootTaskToDisplay` shell 可以调用，但会把 task 放到目标屏的最上层。
- 销屏时，带 DESTROY 的屏上所有 task 被销毁；不带的 PUBLIC 屏把 task 倒回主屏最上层。
- 候选的选择（DESTROY、预检拒绝、落点核验）与平台一致。

### D1. 启动到虚拟屏的权限与支持【上游】

- `ActivityTaskSupervisor.canPlaceEntityOnDisplay`（14 :398–433）：
  - 默认屏总是允许。
  - `!mSupportsMultiDisplay` 时拒绝。
  - 然后调用 `isCallerAllowedToLaunchOnDisplay`（14 :1167–1244）：调用方持有 **INTERNAL_SYSTEM_WINDOW 时直接放行**（:1190–1195），shell 就走这条。
  - App 自己在屏内再启动 Activity 时：trusted 的 PUBLIC 屏检查的是 `isUserVisible`，普通 user0 下通过；private 屏只允许 owner 或已在屏上的 UID。这也是必须用 PUBLIC 的另一个原因。
- `mSupportsMultiDisplay` 取决于 `FEATURE_ACTIVITIES_ON_SECONDARY_DISPLAYS`，并且要求多窗口形态启用（14 ATMS :904–905、:941–954）。
- `TaskLaunchParamsModifier`：options 里的 launchDisplayId 优先于已有 task 所在的屏（14 :425–436，在 :462–466 读取 task 的 root 之前）。不支持多屏时**静默改回默认屏**（:486–489）。
- **可调整大小**：`canPlaceEntityOnDisplay` 不检查 resizeable（只有 VDM 屏的 DWPC 会检查）。不可调整大小的 App 会在虚拟屏上全屏运行；配置与它期望的不一致时进入 size-compat，需设备。
- 补充：无障碍只跟踪非 private 的虚拟屏（`AccessibilityManagerService.isValidDisplay` 14 :4581–4597；16 :5641–5657）。

### D2. 目标 App 已有 task 在主屏时【上游】

`ActivityStarter.getReusableTask`（14 :2846–2900）：
- `NEW_TASK` 且没有 `MULTIPLE_TASK`，或 launchMode 为 singleTask/singleInstance 时，尝试放进已有 task。
- singleInstance 用 `findActivity` 跨所有屏查找。
- 其他情况用 `RootWindowContainer.findTask(r, preferredTDA)`（14 :2217–2258）：
  - **在首选 TDA（虚拟屏）上**接受 ideal match 和 affinity candidate。
  - **在其他 TDA 上只接受 ideal match**。ideal match 的条件是 `task.realActivity` 等于启动的组件，或 affinityIntent 的组件等于启动组件，且 documentData 相同（14 :391–404）。

`setTargetRootTaskIfNeeded`（14 :2912–2999；16 同构，另有 BAL 的 `avoidMoveToFront()`）：
- 已有 task 不在首选 TDA 上时，`differentTopTask=true`，于是 `intentTask.reparent(mTargetRootTask, ON_TOP, REPARENT_MOVE_ROOT_TASK_TO_FRONT…)`，`mMovedToFront=true`。**也就是整体搬到虚拟屏**。

`recycleTask`（14 :2184–2270）的返回值：
- 被搬或被提到前台：`START_TASK_TO_FRONT(2)`。
- 只把 intent 交给顶部 Activity：`START_DELIVERED_TO_TOP(3)`。
- 新建：`START_SUCCESS(0)`。
- 包内 E4 的迁移实测返回 3【实测：AOSP16 AVD】。所以**返回码不能区分“迁移”与“新建”**，只能用 task 事实判断。

与 launch flag 的关系：
- standard/singleTop 加 `NEW_TASK|MULTIPLE_TASK` 会跳过复用，新建 task，主屏 task 不动。代价是同一个 Activity 出现第二个实例，App 未必接受。
- singleTask/singleInstance 不受 MULTIPLE_TASK 影响，仍会复用或迁移。
- documentLaunchMode 会改变匹配的 documentData。
- 候选总是加 `NEW_TASK`（`ActivityLaunch.kt:79`）。

### D3. 显式搬 task【上游】

- `IActivityTaskManager.moveRootTaskToDisplay(taskId, displayId)` 需要 **INTERNAL_SYSTEM_WINDOW**，不是 MANAGE_ACTIVITY_TASKS（14 ATMS :3049–3061；15 :3113；16 :3163）。shell 可以调用。
- 内部固定 `ON_TOP`（14 RWC :1962–1970），所以搬回主屏就会出现在真人面前的最上层。
- 不打扰真人的“放到底层”需要走 WindowContainerTransaction 和 task organizer，复杂度高，不建议做。

### D4. 虚拟屏销毁时 task 去哪【上游】

- removeMode 的来源：
  - 14/15：`LogicalDisplay` 默认 `REMOVE_MODE_MOVE_CONTENT_TO_PRIMARY`；FLAG_PRIVATE 或 FLAG_DESTROY_CONTENT_ON_REMOVAL 时为 DESTROY（14 :408–422）。
  - `DisplayContent.shouldDestroyContentOnRemove` 在 14 读的是 `Display.getRemoveMode()`（14 :6388–6390）。`DisplayWindowSettings.getRemoveContentModeLocked` 在 14/15 只服务于 `IWindowManager.getRemoveContentMode`（14 WMS :7474）。研究包把它当作 14 的决策点，表述不精确，但结论相同。
  - 16 改为读 `DisplayWindowSettings`：private、显示设置覆盖、或 Display removeMode 为 DESTROY 时销毁（16 DisplayContent :6481–6488；DWS :192–204）。所以显示设置文件可以覆盖默认行为。
  - 三个版本的默认结果相同：PUBLIC 屏不带 flag 时迁回主屏，带 flag 时销毁。
- `TaskDisplayArea.remove`（14 :1815–1860；16 `removeAllTasks` :1791–）：
  - 需要销毁、非标准类型、或由 organizer 创建的 task：调用 `task.remove(false, "removeTaskDisplayArea")`，对每个 Activity 执行 `destroyIfPossible`（14 `TaskFragment.java:2801–2819`）。App 走正常的 finishing→onDestroy，**不保存实例状态，进程不被杀**。
  - 其他 task：reparent 到默认 TDA 的 `POSITION_TOP`，最后一个被调用 `moveToFront("display-removed")`，所以**会突然出现在真人面前**。进程和 Activity 实例保留；如果新配置与原来不同，可能触发 relaunch（需设备）。
- worker 死亡时，Binder 死亡会触发释放（14 VDA :302–316），走同一条销毁路径，没有机会先归还 task。

**候选的选择与包内 E4**：
- 候选带 DESTROY（`WorkDisplay.kt:42`），skill 明确写了“可能销毁迁入的 task”（`skills/androidctl/displays.md`）。
- E4 实测：同一个 task15 从 display0 迁入 VD8，正常 close 后不复存在【实测：AOSP16 AVD】。与源码一致。

### D5. 读 task 所在屏的接口【上游】

- `getAllRootTaskInfos()`、`getAllRootTaskInfosOnDisplay(displayId)` 需要 MANAGE_ACTIVITY_TASKS（`enforceTaskPermission`，14 ATMS :3265–3268）。
- `getTasks(maxNum, filterOnlyVisibleRecents, keepIntentExtra, displayId)` 需要 REAL_GET_TASKS 才能读完整。
- AIDL：14 :162/:207/:219/:221；15 :153/:197/:209/:211；16 :152/:201/:213/:215。三个版本签名不变。
- `RootTaskInfo.childTaskNames[i]` 等于各个叶 task 的 `origActivity`，否则取 `realActivity`（14 RWC :2506–2509；16 :2687–2690），**正好是 findTask 用来匹配 ideal match 的键**。
- 根 task 的 `displayId` 就是叶 task 的 displayId，因为叶 task 是它的子节点。

**可以更简单**：
- 候选的预检和落点同时读 `getAllRootTaskInfos` 与 `getTasks`，并在两者不一致时返回 unknown（`ActivityTasks.kt:85–115`、`:108–113`；`ActivityLaunch.kt:99–105`、`:273–310`）。
- 但迁移风险只存在于 WM 层级里的活 task，`findTask` 也只搜这个层级，`getAllRootTaskInfos` 对这个用途是完整的。
- 第二来源、不一致分支和 REAL_GET_TASKS 自检都可以删掉。预检只需要“其他屏上有任一 child 名称或 top 属于该包”，这个按包判断是保守的超集。【建议】

---

## E. 交叉核对

### E1. 总表

| 事实 | AOSP 位置（14/15/16） | 候选做法（file:line） | 一致？ | 包内实测 | 需设备？ |
|---|---|---|---|---|---|
| TRUSTED 缺权限时抛异常；OWN_FOCUS 和 no-steal 依赖不满足时静默去掉 | DMS :1481 / :1641 / :1928；VDA14 :536–553；DMS16 :1972–1984 | 请求 `WorkDisplay.kt:37–46`，读回 `:65–70` | 一致 | API34/37 actualFlags=0x1880 | 否 |
| PUBLIC 必须配 OWN_CONTENT_ONLY | DMS14 :1417–1428；16 :1841–1859 | 两者都请求 | 一致 | — | 否 |
| private 屏不被无障碍跟踪 | AMS14 :4581–4597；16 :5641–5657 | 使用 PUBLIC | 一致 | — | 否 |
| shell 持有可信屏权限，但属于 CTS 段 | Shell manifest 14 :713–716；15 :746–748；16 :973–976 | 不降级，直接拒绝 | 一致 | MuMu15 缺权限而失败 | ROM 需设备 |
| 无 Surface 即 OFF；16 初始 UNKNOWN | VDA14 :296、:560；16 :390–400、:671–676 | 持续 drain 的 ImageReader `WorkDisplay.kt:49–51`、`:80–87`；UNKNOWN 等待 `:71–77` | 一致 | API37 UNKNOWN→ON | 否 |
| no-steal 屏没有 IME | WMS14 :8011–8035；IMMS14 :3763–3782；16 :8223 / :3811 | 不设置 IME policy；editor 只限主屏 `UiBackend.kt:239–247` | 一致 | AVD16 E1b | 否 |
| 无障碍 IC 只对应顶层焦点屏 | IMMS14 :2489–2502 | 只限主屏 | 一致 | API34 virtual editor 被拒 | 否 |
| PASTE 在虚拟屏上可读剪贴板 | ClipboardService14 :333；WMS14 :7997–8008 | 用户决定改用 PASTE | 一致 | 无 | **是** |
| 按键注入没有 ACTION_MULTIPLE | InputDispatcher14 :133–141；16 :186–194 | 按键只做可映射的键 | 一致 | — | 否 |
| 锁屏会停掉虚拟屏 | ATMS14 :2960；KeyguardController14 :168–174、:530–545；KDM14 :120–152 | 要求主屏解锁 `DeviceFacts.kt:69–81` | 一致 | API34 熄屏后输入被拒 | 截图和树的表现需设备 |
| 16 熄屏时虚拟屏 OFF；14/15 仍为 ON | VDA16 :489–527；VDA14 :384–395、:560 | 每次输入都检查虚拟屏 ON `UiBackend.kt:77` | 方向一致，但该检查多余 | E7 AVD16 两屏均 OFF | 14/15 需设备 |
| 绑定 displayId 的锁只作用于该屏所在组；group0 会连主屏一起保亮 | PMS14 :5353–5364、:2700–2735 | 不持锁 | 一致 | E7 own-wake 主屏也被保亮 | 否 |
| QPR2 下 OWN_GROUP 失效 | QPR2 LDM :1094–1100、:1351–1354 | 不请求 | 一致 | E7 groupId=0 | 其他版本需设备 |
| 默认组时，虚拟屏输入推迟主屏熄灭 | PMS14 :1991–2018；A11y14 :2107；PowerManager14 :1436 | 未说明 | **未覆盖** | 无 | **是** |
| 跨屏 ideal match 的 task 会被搬到虚拟屏 | RWC14 :2217–2258、:391–404；AS14 :2846–2999 | 按包预检拒绝 `ActivityTasks.kt:85–115` | 一致（偏保守） | E4 task15 被迁移，返回码 3 | 否 |
| 返回码不能区分迁移与新建 | AS14 :2184–2270 | 落点按 task 事实核验 `ActivityLaunch.kt:312–323` | 一致 | API34 VLC：START_SUCCESS，落点 unknown | 否 |
| 不支持多屏时静默回主屏；不检查 resizeable | TLPM14 :486–489；ATS14 :398–433 | 落点核验 | 一致 | — | ROM 需设备 |
| 销屏：DESTROY 销毁 / 否则倒回主屏顶部 | TDA14 :1815–1860；LogicalDisplay14 :408–422；DC16 :6481 | 带 DESTROY | 一致 | E4 被销毁 | 否 |
| `moveRootTaskToDisplay` 需要 INTERNAL_SYSTEM_WINDOW，且固定 ON_TOP | ATMS14 :3049；RWC14 :1962 | 未使用 | — | — | 否 |
| `isDeviceLocked` 第二个参数在 15/16 是 deviceId | TrustMS15 :1832；16 :1867 | 传入 displayId `DeviceFacts.kt:39` | **不一致（缺陷）** | — | 否 |
| captureDisplay 需要 READ_FRAME_BUFFER，按图层抓取 | WMS14 :9429；16 :10134 | `WorkDisplay.kt:89–111` | 一致 | API37 新截图 | 否 |

### E2. 候选现有做法中与平台不符、或可以更简单的地方

1. 【确定缺陷，低影响】`DeviceFacts.kt:37–40`：`isDeviceLocked(user, displayId)` 在 15/16 上把 displayId 当作 deviceId，虚拟屏 target 的 `deviceLocked` 恒为 false。修法：固定传 0（14 上 0 是 DEFAULT_DISPLAY，15/16 上 0 是 DEVICE_ID_DEFAULT）。在只读主屏锁态的模型下，可以直接删掉 target 的锁态字段。
2. 【简化】task 预检和落点只用 `getAllRootTaskInfos`：删除 `getTasks` 第二来源、`inconsistent→unknown` 分支（`ActivityTasks.kt:108–113`）以及 REAL_GET_TASKS 自检（`ActivityLaunch.kt:101–102`）。依据见 D5。
3. 【简化】`UiBackend.kt:77` 每次输入前读虚拟屏 ON：14/15 恒为 ON，16 上被主屏条件覆盖。建屏时检查一次即可。
4. 【简化】读回只需检查 no-steal 一位；`requestedFlags/actualFlags` 不必出现在对外结果里（`TargetManagement.kt:59`）。它们对 LLM 使用者不影响任何决策，放到诊断日志即可。
5. 【文档补充】skill 没有写“agent 在后台操作会推迟主屏熄屏”；也没有写“锁屏时虚拟屏上会出现锁屏 Presentation”。
6. 【可选】HIDE IME policy：不需要（no-steal 已禁用 IME）。只有在设备上观察到“主屏无焦点时虚拟屏成为顶层焦点屏”时才加。
7. 【一致，保留】editor 只限主屏；DESTROY；按包预检拒绝；用 captureDisplay 截图；ImageReader 只做 drain；建屏时等待 UNKNOWN。

### E3. 报告 A、报告 B、包内研究中与源码不符的主张

| 来源 | 主张 | 核对 |
|---|---|---|
| B §3.3-12、§3.4“默认 HIDE、`--ime local` 给特殊编辑器” | 输入法默认会弹在主屏，必须调用 setDisplayImePolicy | **在 no-steal 下不成立**：IME 根本不启动，LOCAL 也解除不了顶层焦点门槛（B2）。A §4.5 和研究包的判断正确 |
| B §3.4“虚拟屏输入不再要求主屏亮着” | 按屏判断 inputReady | **默认组下不成立**：锁屏会给所有非 ALWAYS_UNLOCKED 屏加 token，16 熄屏时虚拟屏 OFF；独立组在 QPR2 失效 |
| B §3.4 生命周期“先把用户 task 送回主屏，再销毁” | — | 平台接口只能 ON_TOP 地搬回（D3），会打扰真人，而且 worker 死亡时做不到。研究包和 A 不采用，正确 |
| B §3.3-7“A14/A15 上若主屏也灭，注入的触摸被 native 丢弃” | — | 对触摸成立（14 按全局 interactive）；**按键不成立**，注入的非唤醒按键仍会放行（PWM14 :4033）。“无障碍动作的 userActivity 只给默认组续命”成立 |
| B §3.3-7、§3.4 保活 | 持有绑定 displayId 的锁就能独立保活 | 只在虚拟屏实际处于独立组时成立；QPR2 及 E7 下不成立 |
| B §3.3-10 | DESTROY flag 不需要权限 | 成立 |
| B §3.3-8 | NEW_TASK 时跨屏查找并搬移 | 成立，但要细化：其他屏上只接受 ideal match（realActivity 相同），affinity 匹配只在本屏生效 |
| 研究包“PUBLIC 默认 MOVE_TO_PRIMARY（DisplayWindowSettings.java:160–173）” | — | 结论对；14/15 的实际决策点是 `LogicalDisplay.java:408–422` 加 `DisplayContent.shouldDestroyContentOnRemove`，16 才改用 DisplayWindowSettings |
| 研究包 E7 的 QPR2 解释 | separate_timeouts 让 OWN_GROUP 失效 | 已下载同一提交核对，代码路径存在。镜像实际使用的版本未从二进制核实 |
| 研究包 “OWN_FOCUS 只依赖 TRUSTED，ALWAYS_UNLOCKED 依赖分组” | — | 成立 |
| A §4.5“需要真实 InputConnection 的…任意粘贴路径不能由上述两项推导为可用” | — | 措辞偏保守：源码上 ACTION_PASTE 只要求 View 有焦点加 `isUidFocused`，在虚拟屏上成立（B4）。仍需设备确认 |
| 候选 skill“Unsupported permissions … fail without a lower-privilege fallback” | — | 一致 |

### E4. 国产 ROM 可能改动的点（只列源码上可预期的差异类型）

1. **Shell manifest 删减**：ADD_TRUSTED_DISPLAY、ADD_ALWAYS_UNLOCKED_DISPLAY 位于“CTS”段，最容易被删；INTERNAL_SYSTEM_WINDOW、READ_FRAME_BUFFER 也可能被删。
2. **特性和配置开关**：`FEATURE_ACTIVITIES_ON_SECONDARY_DISPLAYS`、`config_supportsMultiDisplay`、`config_perDisplayFocusEnabled`、强制桌面模式（会把 IME 提升为 LOCAL、窗口模式变为 freeform）。
3. **显示设置覆盖**（`display_settings.xml` 按 uniqueId 设置）：removeContentMode、imePolicy、shouldShowSystemDecors。16 起 removeMode 读取这个来源。
4. **aconfig 开关取值**：`correct_virtual_display_power_state`、`separate_timeouts` / DisplayGroupAllocator、`perDisplayWakeByTouch` 一类按屏 interactive 的开关。
5. **WM 启动路径的厂商钩子**：投屏、PC 模式、多屏协同常常改写 `TaskLaunchParamsModifier` / `ActivityStarter` 的落点，或改变 findTask 的跨屏复用。
6. **SystemUI**：锁屏 Presentation、剪贴板浮层和粘贴 toast 的策略。
7. **后台冻结**：对“非前台屏”的进程做 cached freeze 或省电冻结，可能停掉虚拟屏上的 App。
8. **安全或防录屏应用**：拒绝创建虚拟屏，或拦截 FLAG_SECURE 内容。
9. **TrustManager / Keyguard 的多屏判定**被改写。

---

## 必须上设备验证的清单

每项都在至少一台 Android 14/15、一台 16（最好有 QPR2 以后的版本）、一台国产 ROM 上执行。“VD”指候选 `session open --display virtual`（或当前的 `display create`）建出的屏；displayId 从 status 读取。

1. **shell 权限与多屏特性**
   - 执行：`adb shell dumpsys package com.android.shell | grep -E "ADD_TRUSTED_DISPLAY|ADD_ALWAYS_UNLOCKED_DISPLAY|INTERNAL_SYSTEM_WINDOW|READ_FRAME_BUFFER|INJECT_EVENTS|MANAGE_ACTIVITY_TASKS"`；`adb shell pm has-feature android.software.activities_on_secondary_displays`。
   - 缺少前两项：该机型不支持后台屏，直接拒绝。缺少后几项：相关能力（截图、注入、task 读取）不可用。feature 为 false：启动会落回主屏。
2. **熄屏但未锁时，虚拟屏是否继续运行（14/15 与 16 的分歧）**
   - 执行：设置“无锁屏”，或把锁定延迟设为 5 分钟；VD 内打开一个秒表或计时页；按电源键；10 秒后读 `dumpsys display | grep -A3 "Display <vd>"` 的 mState，以及 `dumpsys activity activities` 中该 Activity 的状态；亮屏后截 VD 的图看秒表是否连续。
   - 14/15 上 VD 仍为 ON 且 Activity RESUMED：熄屏但未锁时可以继续后台工作，主屏 ON 的门槛可以按版本放宽。16 上 OFF/STOPPED：门槛必须保留。
3. **锁屏时虚拟屏显示什么**
   - 执行：主屏亮着时锁屏（安全锁）；对 VD 执行 `screenshot` 和 `snapshot`；读 `dumpsys window displays` 中该屏的窗口。
   - 看到 KeyguardPresentation 或黑屏，说明 skill 需要写明“锁屏时 VD 内容被替换”；树里应用窗口仍在，说明锁屏时读树会给出过期内容，需要在锁屏时拒绝观察。
4. **后台操作是否推迟主屏熄屏**
   - 执行：把 screen_off_timeout 设为 15 秒；VD 每 5 秒执行一次节点 click，持续 60 秒；观察主屏是否熄灭，读 `dumpsys power | grep -E "mLastUserActivityTime|Power Group"`。
   - 主屏不熄灭：skill 必须写明这一点。熄灭了：该副作用不存在。
5. **IME 与真人同时输入**
   - 执行：真人在主屏 EditText 上打字、软键盘显示；agent 在 VD 上对 EditText 执行 focus 和 click；读 `dumpsys input_method | grep -E "mCurClient|mCurFocusedWindow|mInputShown|mCurTokenDisplayId"`。
   - mCurClient 仍为主屏且键盘不动：与源码一致。键盘消失或转移：说明 ROM 改了 IMMS，后台屏的文本输入必须禁用或改用 HIDE。
6. **PASTE 局部中文插入（决定 fill 和 paste 的设计）**
   - 执行：`clipboard set "中文"`；对 VD 上的 EditText 执行 SET_SELECTION(3,3) 和 PASTE；回读节点文本。再对 WebView 输入框、Compose TextField 各做一次。观察主屏是否出现剪贴板浮层或“已粘贴”toast；再带上 SUPPRESS extra 重做一次。
   - 文本正确插入：PASTE 成为后台中文插入的主路径。某类控件失败：对该类控件退回 SET_TEXT 整段写入。浮层能被 extra 抑制：`clipboard set` 默认带上这个 extra。
7. **按键注入（ASCII 与 Ctrl+V）**
   - 执行：对 VD 上已聚焦的 EditText 注入 `abc`，以及 Ctrl+A、Ctrl+V。
   - 成功：key 通道在 VD 上可用。失败：VD 上的 key 只保留 BACK/ENTER/方向键。
8. **任务迁移与 MULTIPLE_TASK**
   - 执行：在主屏打开某个 standard launchMode 的 App 并停在二级页；用 shell 以 launchDisplayId=VD 启动它的 launcher Activity；读 `dumpsys activity activities` 中的 taskId 和所在 display，记录返回码。再加上 MULTIPLE_TASK 重复一次。
   - 第一次 task 被搬走、第二次新建且主屏 task 不动：可以考虑对 standard App 提供“隔离启动”。第二次被 App 拒绝或行为异常：保持按包拒绝。
9. **销屏后的 task 结局**
   - 执行：只用自有测试 App，在 VD 上打开后正常 close；再 kill worker 一次；看 logcat 中的 `am_destroy_activity`，以及 `dumpsys activity activities`。
   - 两种方式都被销毁且没有出现在主屏：当前语义成立。出现在主屏：ROM 改写了 removeMode，带 DESTROY 不足以防止，必须在建屏后读 `dumpsys window` 的 removeMode，或拒绝该 ROM。
10. **独立组与锁屏豁免实验（只为裁定“放宽”）**
    - 执行：用 OWN_DISPLAY_GROUP|ALWAYS_UNLOCKED 建屏（原型即可）；读 `dumpsys display` 中的实际 groupId；主屏锁屏并熄屏；在 VD 上持有一把绑定 displayId 的 SCREEN_DIM WakeLock；60 秒后看 VD 上 Activity 的状态和主屏是否亮着。
    - groupId≠0、VD 运行且主屏熄灭：该机型可以作为可选能力。groupId=0（QPR2 一类）或主屏被一起保亮：不做。
11. **16 的 UNKNOWN 时长**
    - 执行：建屏 20 次，记录 UNKNOWN→ON 的时长。
    - 都在百毫秒级：保持在预算内等待。偶尔数秒：创建流程的预算需要说明。
12. **isDeviceLocked 修复**
    - 执行：15/16 上锁屏时读 VD target 的 deviceLocked（修复前后各一次）。
    - 修复前为 false、修复后为 true：确认缺陷并已修复。
13. **（可选）小尺寸 ImageReader 作为 drain 消费端**
    - 执行：ImageReader 设为 64×64 而 VD 保持全尺寸；对比 captureDisplay 截图的尺寸和内容，以及 App 的布局。
    - 截图和布局都不变：可以节省约 30MB。有变化：保持全尺寸。

---

## 未覆盖或未核实

- InputDispatcher 中 `pokeUserActivity` 对注入事件的具体条件；SurfaceFlinger 在 VirtualDisplaySurface 缓冲用尽时的行为；global interactive 与各组 wakefulness 的完整关系（只做了推断，未逐行追）。
- 16 QPR2 只核对了分组两个文件；没有核对 QPR2 的 VDA/DMS 电源路径。
- 研究包 E1–E7 的原件只读了报告摘要，没有逐个打开 stdout 原件。
- `UiAutomation.takeScreenshot` 各版本的 display 参数（报告 A §4.4）不在本题范围内，没有核对。
- 所有关于 App 覆盖面（WebView、Compose、自绘编辑器）的结论都需要设备验证。
