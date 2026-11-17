# 后台屏现成实现对比：scrcpy / Mika / dsh-go / Pinky 与 AndroidCtl 候选

## 0. 结论（先读这一节）

1. **AndroidCtl 候选的建屏方案已经是四家里最干净的一种，不需要大改。** 候选申请的 7 个 flag（`WorkDisplay.kt:37-46`）与 dsh-go 的 flag 列表完全一样（`PrivilegedDisplayCore.kt:2082-2084`）。缺任何一个就拒绝建屏、不降级（dsh-go 也这样做，Mika 则会降级）。候选还多做了一步：建屏后读回 `Display.getFlags()`，核对 TRUSTED/OWN_FOCUS/no-steal 三位（`WorkDisplay.kt:65-70`），四家上游都没有在每次建屏时这样做。这部分保留。可以再加一个 `TOUCH_FEEDBACK_DISABLED`：scrcpy 和 Mika 在 API 33+ 都加了它，不需要额外权限。
2. **最该向上游学的是"落点核实要等一会儿"。** dsh-go 启动后最多轮询 4 秒，每 120ms 查一次目标屏顶部任务的**包名**（`PrivilegedDisplayCore.kt:877-887`）。候选只在启动返回后立即读一次，并要求精确匹配解析出的 Activity（`ActivityTasks.kt:117-160`）。结果 API34 和 API37 两次真实 VLC 运行都是 `landing unknown / matching 0 / exit 1`，但事后独立查询确认 App 已经落在虚拟屏【实测，AVD，候选 3f45af6】。建议改成：在请求预算内，按包名和 UID 轮询目标屏顶部任务。
3. **scrcpy 的默认值是为"有人看着这块屏"设计的，后台屏不应照抄。** 具体指：开系统装饰、不设输入法策略（键盘默认弹到主屏）、不加 `STEAL_TOP_FOCUS_DISABLED`、33+ 默认加 `ALWAYS_UNLOCKED`、`--start-app` 不检查是否已有任务也不核实落点。唯一值得直接对齐的默认值是 **"关屏时销毁内容"**：scrcpy 从 v3.0 起默认就是销毁，v3.1 才加了 `--no-vd-destroy-content` 作为可选的"搬回主屏"。
4. **"App 在主屏已有任务"四家各有一套处理**：scrcpy 不处理，只提供 `+` 强停；Mika 只拒绝主屏上**可见**的任务；Pinky 先记账，关屏时再送回；dsh-go 用 `MULTIPLE_TASK` 另开一个任务，再核实落点。候选的做法是"已知任何屏上有该包任务就不发送启动"，比四家都严。它的代价是常用 App 的覆盖面变小，换来的是不需要台账和归还逻辑。建议首版保留这个做法，`MULTIPLE_TASK` 等真机数据再决定（见第 9 节）。
5. **输入法**：scrcpy 的 `--display-ime-policy` 只决定**获准弹出的键盘显示在哪块屏**（local/fallback/hide），不决定能不能拿到输入法。AOSP 14 在 `Display.java:356-359` 写明：no-steal 屏上隐含禁用 IME。所以 AndroidCtl 后台屏上的文本输入只能走节点 `SET_TEXT`/`IME_ENTER`、`ACTION_PASTE` 或按键注入，不能依赖软键盘。这与用户"`clipboard set` + 通用 PASTE"的决定方向一致。Mika 的注释称"真机 Android 14 上 LOCAL 能在虚拟屏弹出键盘"，这与 AOSP 和任务 AVD 的实测相反，必须上真机核实。
6. **电源与锁屏方面，候选的"主屏亮且解锁才工作、不保活"是对的。** Pinky 的源码注释记录了真机现象：Android 17（Mi 10 Pro LineageOS 24 与 A17 模拟器）上，虚拟屏不管带不带 `OWN_DISPLAY_GROUP` 都会被并入主屏电源组，按 displayId 持唤醒锁也无效（`dark.go:18-30`）。这与包内 AVD 实验 E7 的 group0 结果一致。Mika 只在读回的 `displayGroupId != 0` 时才替这块屏续命（`DisplayAwake.kt:52-53`）。上游的经验是：不要根据 flag 承诺"独立保活"。
7. **"先探测能力再建屏"在四家里都只是面向人的自检页**（Mika `AndroidSelfCheck.kt:168-185`，dsh-go `selfCheckProbe` 在 `PrivilegedDisplayCore.kt:677-720`），不是每次建屏前的步骤。scrcpy 只在 API<29 时拒绝（`Server.java:76-86`）。AndroidCtl 不需要另做 probe/doctor：建屏本身就是探测，读回 flag 就是判据。但要把失败映射成明确的 `displayUnsupported`：候选在 API 30–33 上会因 `NoSuchFieldException`、在缺 `ADD_TRUSTED_DISPLAY` 的 ROM 上会因 `SecurityException`，都落到通用的 `workerFailure`（`WorkDisplay.kt:46`，`Wire.kt:75-78`）【源码】。

---

## 1. 版本与来源

| 项目 | 核对的版本 | 说明 |
|---|---|---|
| scrcpy | **v5.0.1 = `a60891aea193d92e7e5c3942700eca63f9d19a5f`**（最新 tag，2026-10-08，等于 master HEAD）；功能首次出现的 tag 用 `git tag --contains` 核对 | 下载方式为 `git clone`；GitHub API 和 PR 网页返回 403，**PR/issue 讨论未能核实** |
| Mika (wangzan101/mika) | HEAD `f881d83fa120510509ecf40556a3d23a35abeee2`（2026-11-17）；报告 B 引用的 `c393f6455bc8…` 也已取回，抽查一致 | |
| dsh-go (ccch1mneyyy/dsh-go) | HEAD `6db08d2e2faafec2db8597587d424c2165196dad`（2026-11-17）；B 引用的 `3edc945fcdd9…` 已取回 | 行号以 HEAD 为准；B 版的同一段代码约前移 40 行 |
| Pinky (DIYgod/pinky) | HEAD `762d3995e6632e60e6bb7641b4f330b39a8007a7`（2026-11-17）；B 引用的 `960377d0ea5a…` 已取回，抽查一致 | |
| AOSP | `android-14.0.0_r1`：`Display.java`、`DisplayManager.java`、`DisplayWindowSettings.java`、`TaskDisplayArea.java`；`android-13.0.0_r1`：`DisplayManager.java` | Gitiles `?format=TEXT` 下载 |
| AndroidCtl 候选 | REPO `3f45af6` | |

本地副本位于 `scratchpad/agents3/work16/{scrcpy,mika,dsh-go,pinky,aosp}/`。

报告 B §3.2 说的"四家"，指的就是 scrcpy、Mika、dsh-go、Pinky。报告 A §4.3 和 `research/display/report.md` 比较的也是这四家（`refs.json` 记录了各自的 HEAD）。

---

## 2. 总表

| 维度 | scrcpy v5.0.1 `--new-display` | Mika | dsh-go | Pinky | **AndroidCtl 候选** |
|---|---|---|---|---|---|
| 进程 / 建屏身份 | app_process shell；用 FakeContext=`com.android.shell` 反射构造 DisplayManager | shell 助手；DisplayManager 反射构造 | Shizuku/ADB 共用 Core | 内嵌 scrcpy-server | Shizuku shell worker，shell Context |
| 基础 flag | PUBLIC\|PRESENTATION\|OWN_CONTENT_ONLY\|SUPPORTS_TOUCH\|ROTATES_WITH_CONTENT（+默认 DESTROY、默认装饰） | PUBLIC\|OWN_CONTENT_ONLY\|TOUCH\|ROTATES\|DESTROY | PUBLIC\|OWN_CONTENT_ONLY\|TOUCH\|DESTROY | 同 scrcpy，但关装饰 | PUBLIC\|OWN_CONTENT_ONLY\|TOUCH\|DESTROY |
| 33+ 追加 | TRUSTED\|OWN_DISPLAY_GROUP\|ALWAYS_UNLOCKED\|TOUCH_FEEDBACK_DISABLED | TOUCH_FEEDBACK_DISABLED；可信时再加 TRUSTED\|OWN_GROUP\|ALWAYS_UNLOCKED | （按名字反射，全部必需）TRUSTED | 同 scrcpy | TRUSTED |
| 34+ 追加 | OWN_FOCUS\|DEVICE_DISPLAY_GROUP（**无 no-steal**） | OWN_FOCUS\|STEAL_TOP_FOCUS_DISABLED | OWN_FOCUS\|STEAL_TOP_FOCUS_DISABLED | 同 scrcpy | OWN_FOCUS\|STEAL_TOP_FOCUS_DISABLED |
| 缺权限 / 缺 flag | 抛 AssertionError，进程退出；API<29 时配置错误 | SecurityException 后**去掉可信类 flag 重建**，继续用 | 反射取不到常量就报 `displayUnsupported`；**不读回** | 继承 scrcpy；SDK<29/30 时拒绝 | 读回三位 flag，缺则拒绝；但 API<34 和 SecurityException 的错误码是 `workerFailure` |
| 能力探测 | 无 | 自检页：建屏、读 OWN_FOCUS 位、开自有页 | 自检：探测屏、自有窗口、注入后等 DOWN/UP 回执 | SDK 门槛；读回 displayGroup；探测 H.265 | 无单独探测；建屏即探测，读回 flag，等待 STATE_ON |
| 尺寸 / dpi | 主屏自然方向尺寸与 dpi | 主屏 metrics | 短边 720、dpi 320 | 主屏自然尺寸 | 主屏 realMetrics、densityDpi |
| 启动到虚拟屏 | `ActivityOptions.setLaunchDisplayId` + startActivityAsUser；`+` 先 forceStop；**不检查返回值** | 同上；`am start --display` 作回退 | `NEW_TASK\|MULTIPLE_TASK` + PendingIntent.send(options) | 调 scrcpy start-app | startActivityAsUser（shell 包名）+ setLaunchDisplayId |
| 已有任务 | 不处理 | 主屏**可见**就拒绝；后台任务允许被搬入 | MULTIPLE_TASK 另开 | 启动前记下他屏任务，关屏时 `move-stack` 送回 | 任何他屏已知任务都拒绝（`taskOnOtherDisplay`） |
| 落点核实 | 无 | 有（开到别处就挪回并报告） | 轮询 4s，查目标屏顶包；报 `targetLeftDisplay`/`launchUnconfirmed` | 无（只有 nudge） | 启动后读一次，精确匹配 Activity+UID；`unknown` 时 exit 1 |
| IME 策略 | 默认不设（AOSP 默认 FALLBACK→主屏）；可选 local/fallback/hide | LOCAL | HIDE | local | 不设 |
| 文本通道 | 按键注入（KeyCharacterMap）；剪贴板 + `KEYCODE_PASTE`；UHID | SET_TEXT、ACTION_PASTE、按键 | SET_TEXT | 按键或 companion | 节点 SET_TEXT/IME_ENTER；（IC 仅主屏） |
| 输入注入 | `InputEvent.setDisplayId`，ASYNC | setDisplayId + `injectInputEvent(e, mode, targetUid)` | setDisplayId | scrcpy 或 companion | setDisplayId + 按 UID 定向注入 |
| 画面 | 视频编码器 Surface | 自持 ImageReader，带帧时间戳 | 自持 ImageReader | `screencap -d`（它自己注释说对虚拟屏不可用） | ImageReader 只排空；截图走 `IWindowManager.captureDisplay` |
| 锁屏 | 33+ ALWAYS_UNLOCKED（锁屏下照常） | 同 | 锁屏即拒 `locked` | 依赖 ALWAYS_UNLOCKED，33+ 默认走后台屏 | 要求主屏亮且解锁 |
| 保活 | `--keep-active`：每 4s 调 `userActivity(displayId)`（可选） | 仅当读回 group≠0 时每 4s 按屏 userActivity | 无 | 每 4s 注入 KEYCODE_UNKNOWN；A17 上"黑屏保醒" | 无；目标屏非 ON 时拒绝输入 |
| 销毁 | **默认 DESTROY**；`--no-vd-destroy-content` 改为搬回主屏 | 逐个任务决定留或关；留下的挪到主屏最底层 | DESTROY，释放后等任务清空（≤1s） | 先送回原属用户的任务，再 DESTROY；KeepContent 可选 | DESTROY，不归还 |
| 文档限制 | 见 §3.9（简短） | — | — | 类型注释写明三条 | `displays.md` 90 行 |

---

## 3. 逐项展开（附出处）

### 3.1 建屏 flag 与版本差异

**scrcpy**【上游，v5.0.1】
- `server/src/main/java/com/genymobile/scrcpy/video/NewDisplayCapture.java:31-43`：flag 全部用本地常量定义，不读 SDK 字段。这样做的原因见提交 `5e10c37f`《Define all DisplayManager flags locally》。
- `NewDisplayCapture.java:211-235`（`startNew`）：
  - 始终加 `PUBLIC | PRESENTATION | OWN_CONTENT_ONLY | SUPPORTS_TOUCH | ROTATES_WITH_CONTENT`；
  - 默认加 `DESTROY_CONTENT_ON_REMOVAL`（`:218-220`）和 `SHOULD_SHOW_SYSTEM_DECORATIONS`（`:221-223`）；
  - API ≥ 33 加 `TRUSTED | OWN_DISPLAY_GROUP | ALWAYS_UNLOCKED | TOUCH_FEEDBACK_DISABLED`（`:224-228`）；
  - API ≥ 34 加 `OWN_FOCUS | DEVICE_DISPLAY_GROUP`（`:229-231`）；
  - **从不加 `STEAL_TOP_FOCUS_DISABLED`**（1<<16 不在常量表里）。
- flag 按 Android 版本分档的原因：这些常量在 AOSP 中分批出现。核对结果：`android-13.0.0_r1` 的 `DisplayManager.java` 有 TRUSTED(:371) 和 ALWAYS_UNLOCKED(:391)，没有 OWN_FOCUS 和 STEAL_TOP_FOCUS_DISABLED；`android-14.0.0_r1` 的 `DisplayManager.java:454` 有 OWN_FOCUS、`:476` 有 STEAL_TOP_FOCUS_DISABLED。
- 各项何时加入：
  - 首个版本 `98ed5eb6`（v3.0）就带了 TRUSTED 等可信类 flag，以及始终开启的 DESTROY 和系统装饰；
  - `145b823b`（v3.0）加入 `--no-vd-system-decorations`，提交说明是"Some devices render a broken UI when this flag is enabled"；
  - `6c6607d4`（v3.1）加入 `--no-vd-destroy-content`；
  - `fd8bef68`（v3.2）加入 `--display-ime-policy`；
  - `9d56d26d`（v3.3.3）加入 PRESENTATION，理由是"apps with baked in two-screen support can see the virtual display as an external display"。
- 建屏方式：`wrappers/DisplayManager.java:176-182`，用 `FakeContext`（包名 `com.android.shell`，`FakeContext.java:19`）反射构造 `DisplayManager`，再调公开的 `createVirtualDisplay`。
- 缺权限时的表现：`NewDisplayCapture.java:260-263` 记日志后抛 `AssertionError("Could not create display")`，没有降级。版本门槛在 `Server.java:76-86`：API<29 时 `--new-display` 和 `--display-ime-policy` 都报 ConfigurationException。

**Mika**【上游，f881d83】
- `cap/phone/shell/ShellService.kt:1778-1791` 的 `displayFlags`：基础集合加 ROTATES 和 DESTROY；33+ 加 TOUCH_FEEDBACK_DISABLED，可信时再加 TRUSTED|OWN_GROUP|ALWAYS_UNLOCKED；34+ 且可信时加 OWN_FOCUS|STEAL_TOP_FOCUS_DISABLED。
- `:103-120` 加 `:1814`：遇到 `SecurityException` 就以 trusted=false 重建，日志写明"退回后这块屏没有自己的焦点"。
- `:122-132`：建屏后设成全屏窗口模式、开方向监听、设 IME 策略，并按读回的显示组决定是否保活。

**dsh-go**【上游，6db08d2】
- `phone/background/PrivilegedDisplayCore.kt:752-760` 加 `:2082-2084`：`FLAG_NAMES` 共 7 项（PUBLIC、OWN_CONTENT_ONLY、SUPPORTS_TOUCH、DESTROY_CONTENT_ON_REMOVAL、TRUSTED、OWN_FOCUS、STEAL_TOP_FOCUS_DISABLED），任意一项反射失败就报 `displayUnsupported`。注释是"never silently make a focus-stealing display on an old ROM"。
- 建屏后只检查 `displayId > 0`（`:767`），**不读回 flag**。尺寸是短边 720、dpi 320（`:741`、`:764`）。

**Pinky**【上游，762d399】
- 用 scrcpy 的 flag，关系统装饰：`internal/driver/android/background.go:169-182`。版本门槛：`:112`（≥30）、`:131`（≥29）。

**候选**【源码，3f45af6】
- `WorkDisplay.kt:37-46`：用 `DisplayManager.getField("VIRTUAL_DISPLAY_FLAG_$name")` 取 7 个 flag，与 dsh-go 完全相同。
- `:65-70`：读回 `Display.FLAG_TRUSTED|OWN_FOCUS|STEAL_TOP_FOCUS_DISABLED`，缺则 `displayUnsupported`。
- `:71-77`：等待显示状态从 UNKNOWN 变为非 UNKNOWN，非 ON 就拒绝。
- AVD 上读回结果是 requested=83273 (0x14549)、actual=6272 (0x1880)【实测，AVD API34/37，`stages/37-work-display/api37-readiness/runtime-api3{4,7}/result.md`】。
- **缺陷【源码】**：
  - API 30–33 上 `getField(...STEAL_TOP_FOCUS_DISABLED)` 会抛 NoSuchFieldException。A13 r1 没有这个常量，见上。
  - 缺 `ADD_TRUSTED_DISPLAY` 时 `createVirtualDisplay` 抛 SecurityException。包内 MuMu Android 15 实测就是这样（`research/display/report.md` "本轮设备结果：MuMu"）。
  - 这两种情况都会经 `problem()` 变成 `workerFailure`（`Wire.kt:75-78`），而不是文档承诺的"unsupported"（`displays.md:35-36`）。

### 3.2 能力探测

- **scrcpy**：没有探测，只有 API 门槛（`Server.java:76-86`）。建屏失败时进程直接终止。
- **Mika**：自检页 `selfcheck/AndroidSelfCheck.kt:168-185`。它实际建一块屏，读 `Display.FLAG_OWN_FOCUS`，结果分三档：pass、partial（附 `Advice.FocusShared`）、fail；之后再开一个自有页面，确认落在 Mika 屏上（`:187-198`）。这是给人看的诊断，平时建屏仍会降级。
- **dsh-go**：`selfCheckProbe`（`PrivilegedDisplayCore.kt:677-720`）。它建一块 360 短边、带 PRESENTATION 的探测屏，放一个自有接收窗口，注入 DOWN/UP，收到回执才置 `INJECT_OK=true`。每个 owner 只允许一次（`:684-688`）。
- **Pinky**：SDK 门槛；建屏后读回 `displayGroup` 判断是否与主屏同组（`background.go:210-213`）；首次在新设备上会探测 H.265 编码器（`:259-285`）。
- **结论**：没有一家在**每次建屏前**做独立探测。真正共同的做法只有一条："按名字取 flag，取不到就拒绝"，dsh-go 和候选都是这样。另外只有 Mika 和 Pinky 会读回某个事实（OWN_FOCUS 位、displayGroupId），并据此改变行为。没有任何一家检查 `android.software.activities_on_secondary_displays` feature 或 shell 是否持有 `ADD_TRUSTED_DISPLAY`。候选"建屏 → 读回 → 失败即释放"已经是最强的组合，缺的只是明确的错误码。

### 3.3 启动 App 到虚拟屏

**scrcpy**【上游】
- `control/Controller.java:774-814`（`startApp`）：`+` 前缀表示先强停；`?` 前缀表示按应用名搜索。
- `:817-835`：在 `--new-display` 下最多等 1 秒拿到虚拟屏 id，拿不到就报"No known display id"。
- `device/Device.java:298-321`：`getLaunchIntent`，加 `FLAG_ACTIVITY_NEW_TASK`；API ≥ 26 时设 `ActivityOptions.setLaunchDisplayId(displayId)`；需要时先 `forceStopPackage`（`wrappers/ActivityManager.java:158-165`，userId=USER_CURRENT），然后 `startActivity`。**返回值被丢弃**，不核实落点，也不检查已有任务。
- 用的是 binder 调用，不是 `am start --display`。
- `doc/device.md:174-179` 写明：`+` 前缀"forced-stop before being started"。强停会结束用户在任何屏上的这个 App 的全部任务和进程，等于用丢失用户状态来换"从干净状态启动"。

**Mika**【上游】
- `ShellService.kt:257-293`：在 `launchLock` 内依次执行：
  1. 目标 App 在主屏有可见任务就返回 `START_IN_USE_ON_MAIN`（判定逻辑在 `:633-638`，读 `TaskInfo.isVisible`，读不到时只看列表第一项）；
  2. 已被另一块 Mika 屏占用就拒绝；
  3. 读不到任务列表就拒绝（"看不清主屏状态时宁可不启动"）；
  4. 上次留下的任务整块挪回来（`resumeKept`）；
  5. 用 `startActivityAsUser`（shell 包名，带 options）启动，失败时退回 `am start --display`。
- 后台（不可见）任务**允许**被系统搬到虚拟屏。

**dsh-go**【上游】
- `PrivilegedDisplayCore.kt:863-888`：使用 `FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_MULTIPLE_TASK`，通过 `PendingIntent.send(..., options)` 启动。注释说明原因：shell UserService 不是 ActivityManager 附着进程，`Context.startActivity` 会被拒。
- 启动后最多 4 秒、每 120ms 检查一次 `foregroundPackage() == pkg`。超时后，如果屏上已有别的任务就交给下一次读屏报告；否则报 `targetLeftDisplay` 或 `launchUnconfirmed`。
- B 引用的 `3edc945` 版本里同样有 `MULTIPLE_TASK`（第 908 行），**报告 B §3.2 写 dsh-go "启动前保护用户的 App：无"不准确**：它用 MULTIPLE_TASK 避免搬动 standard 启动模式的旧任务。不过 MULTIPLE_TASK 对 singleTask/singleInstance 无效（报告 A §4.6 与 `research/display/report.md` §2 已指出）。

**Pinky**【上游】
- `background.go:651-679`（`startWith`）：启动前通过 `rootTasks` 记下所有不在本屏的任务 id（`b.theirs`），然后调 scrcpy start-app（`apps.go:154-171`）。
- 没有落点核实。启动后如果读不到窗口，会依次发 KEYCODE_UNKNOWN、再启动一次 App（"nudge"，注释 `:681-712`，实现 `:714-745`）。

**候选**【源码】
- `ActivityLaunch.kt:79`、`:107-123`、`:136-161`：`NEW_TASK`，用 `startActivityAsUser` 以 `com.android.shell` 启动，虚拟屏时带 `setLaunchDisplayId`。
- 预检在 `:273-310` 和 `ActivityTasks.kt:85-115`：`getAllRootTaskInfos` 与 `getTasks(INVALID_DISPLAY)` 中任何他屏任务含该包就判冲突；两份事实不一致时判 unknown。两种情况都不发送。
- 落点在 `ActivityLaunch.kt:312-325` 和 `ActivityTasks.kt:117-160`：启动返回后读一次；要求唯一一个可见叶任务，且 `actualActivity` 与解析出的组件、UID 都匹配。
- 【实测】API34 和 API37 的 VLC VIEW 都是 `matching 0 / landing unknown / exit 1`，而事后独立 `dumpsys` 确认 task 在虚拟屏（`runtime-api34/result.md`、`runtime-api37/result.md` 的"唯一VIEW"行）。可能原因有两个：VIEW 实际落在 `VideoPlayerActivity`，与解析出的组件不同（跳板 Activity）；读得太早。Mika 的注释也提到"包里有些 Activity 是跳板，直接启动会立刻跳走"（`ShellService.kt:252-253`）。

### 3.4 输入法与文本输入

**scrcpy**【上游】
- 文档 `doc/virtual-display.md:105-114`："By default, the virtual display IME appears on the default display. To make it appear on the local display, use `--display-ime-policy=local`"。
- `app/src/cli.c:373-382`：可选值 local、fallback（默认屏）、hide。
- 实现分两种情况：
  - 新屏：建好后调一次 `IWindowManager.setDisplayImePolicy`（`NewDisplayCapture.java:240-242`）；
  - 已有副屏（`--display-id`）：`CleanUp.java:104-115` 先保存旧值，退出时在 `:252-254` 还原。
- API 门槛：`wrappers/WindowManager.java:241-266`，API<31 用 `setShouldShowIme`（不支持 hide）；`Server.java:82-85` 规定 API<29 时报错；`cli.c:3374-3378` 规定只能用于副屏。
- AOSP 默认值【上游，A14】：`DisplayWindowSettings.java:229-238`，默认屏是 LOCAL，其他屏未设置时为 `FALLBACK_DISPLAY`。这解释了 scrcpy 文档中"默认键盘出现在主屏"。
- 文本通道：
  - `Controller.java:451-481`：`injectChar`/`injectText` 用 KeyCharacterMap 把字符转成按键事件，`setDisplayId` 后注入；
  - `:707-727`：`setClipboard(text, paste)` 设置剪贴板，再向操作屏注入 `KEYCODE_PASTE`；
  - UHID 键盘在 API ≥ 35 时可以绑到虚拟屏（`:182-213`）。
  - 这些通道都不依赖软键盘窗口。

**Mika**：`ShellService.kt:1639-1646` 与 `:1816-1818` 设置 `IME_POLICY = 0 (LOCAL)`，注释是"真机（安卓 14）实测 LOCAL：在虚拟屏聚焦输入框后输入法窗口出现在虚拟屏上，主屏焦点不变"。文本走 `ACTION_SET_TEXT`（`a11y/WheelDriver.kt:162`）和 `ACTION_PASTE`（`a11y/AndroidScreen.kt:955-956, 2393`）。

**dsh-go**：`PrivilegedDisplayCore.kt:769-771` 和 `:2028-2035` 设为 HIDE，失败也忽略（runCatching）。文本走 `ACTION_SET_TEXT`（`:560-563`、`:1937-1947`）。

**Pinky**：`DisplayIMEPolicy: "local"`（`background.go:179-181`）。

**AOSP**【上游，A14 `core/java/android/view/Display.java:356-359`】："The framework only supports IME on the top focused display (b/262520411). Therefore, Enabling this flag [STEAL_TOP_FOCUS_DISABLED] on a display implicitly disables showing any IME."

**任务 AVD 实测**（转述 `research/display/report.md` E1b，Android 16 AVD，研究探针而非候选）：
- LOCAL 和 HIDE 两种策略下，VD 上都没有 IME 窗口，主屏键盘不受影响；
- 节点 SET_TEXT 和 IME_ENTER 在两种策略下都生效。

**候选**：不调用 `setDisplayImePolicy`（全仓 grep 无结果）。`displays.md:63-64` 写明当前编辑器输入（IC）只在主屏可用。

### 3.5 输入注入

- **scrcpy**：
  - 位置类事件进入虚拟屏 id，按键在 `--new-display` 下也进入虚拟屏 id（`Controller.java:46-60` 的说明，`:750-763` 的 `getActionDisplayId`）；
  - `Device.java:62-72` 中 displayId≠0 时调用 `InputManager.setDisplayId`，注入模式是 ASYNC，不确认目标窗口；
  - API ≥ 29 才支持非主屏注入（`:57-60`）。
- **Mika**：`ShellService.kt:1649-1690`。`inject` 是 setDisplayId 后注入；`injectTo` 用 `injectInputEvent(event, mode, targetUid)`（API 33+ 隐藏重载），焦点不在目标 UID 时系统拒绝。按键用 WAIT_FOR_RESULT，触摸用 ASYNC（`:1798-1802`）。
- **dsh-go**：`PrivilegedDisplayCore.kt:2013`，setDisplayId。
- **候选**：`AppInput.kt:64-115`，按 UID 定向注入加 setDisplayId，`mismatch` 有专门结果。这与 Mika 的 `injectTo` 一致，比 scrcpy 严格。
- **主屏焦点**：只有带 no-steal 的实现（Mika、dsh-go、候选）能保证注入和无障碍动作不把虚拟屏顶成顶层焦点屏。依据是报告 B §3.3 第 4 条转述的 A14 `WindowManagerService.java:3138-3155`（本轮**未重下载核对**），以及 E1b 实测主屏 focusLost=0。

### 3.6 画面获取

- **scrcpy**：VD 的 Surface 就是编码器（或 OpenGL 滤镜）的输入（`NewDisplayCapture.java:266-285`）。文档 `virtual-display.md:90-91` 说："if no app is started, no content will be rendered, so no video frame will be produced at all"。
- **Mika**：自持 ImageReader，保存 `image.timestamp`（`cap/phone/display/AgentDisplay.kt`；报告 A §4.4 和研究报告 §5 已核）。
- **dsh-go**：自持 ImageReader，RGBA_8888，maxImages=2（`PrivilegedDisplayCore.kt:750`、`:1174`）。
- **Pinky**：`Screenshot` 调 `screencap -p -d N`（`background.go:474-487`）。它自己的类型注释在 `:50-53` 写道："Screenshot returns driver.ErrUnsupported, since screencap rejects virtual display ids; decode a frame from Stream instead"。
- **候选**：ImageReader 只用来排空帧、让 VD 保持有 Surface（`WorkDisplay.kt:80-87`）；截图走 `IWindowManager.captureDisplay(displayId, null, listener)` 同步抓取（`:89-111`）。【实测】AVD 上 VLC 绿帧 PNG 正确。这种做法的取舍是：多依赖一个隐藏 API（API 34+），换来"截图时刻 = 抓取开始"，不会拿到旧 buffer。研究 E1b 的观察是：动作后立即取 ImageReader 最新帧，可能仍是事件前的 buffer。

### 3.7 电源与锁屏

- **scrcpy**：
  - 33+ 带 `ALWAYS_UNLOCKED|OWN_DISPLAY_GROUP`，意图是锁屏和主屏灭屏时 VD 照常运行；
  - `--keep-active` 每 4 秒调 `IPowerManager.userActivity(displayId, …)`（`Controller.java:78`、`:250-262`，`Device.java:91-94`，`wrappers/PowerManager.java:54-80`，API 31+ 有按屏重载），文档见 `doc/device.md:7-14` 和 `virtual-display.md:33-38`；
  - 开机按 POWER 只针对主屏 mirror（`Controller.java:231`）。
- **Mika**：`shell/DisplayAwake.kt:5-10`、`:52-53`。只有建屏带 OWN_GROUP、并且**读回的 `displayGroupId` ≠ 0** 时才每 4 秒按屏 userActivity，注释是"报到默认组会让用户的主屏也一直不灭"。
- **dsh-go**：锁屏即拒（`PrivilegedDisplayCore.kt:317, 438, 617, 698, 1795` 的 `Fault("locked")`）；不保活。
- **Pinky**：
  - `background.go:91-106`：33+ 默认走后台屏，靠 ALWAYS_UNLOCKED；
  - `:286-326`：每 4 秒注入 `KEYCODE_UNKNOWN`，以免锁屏时 10 秒超时；
  - `dark.go:18-40` 记录了一个关键平台变化：A17（"Android 17's separate screen timeouts put every virtual display that does not belong to a virtual device into the main display's group … flag or not … A display-scoped wake lock does not help. Seen on the Mi 10 Pro (LineageOS 24) and on the Android 17 emulator"），所以它在 A17 上把主屏"黑屏保醒"；
  - `background.go:91-99` 又称"Up to Android 16 they also get a display group of their own (checked on the Android 16 emulator)"。这与包内 Android 16 AVD 的 E7（`separate_timeouts=true`，group0）不一致，说明分组取决于 QPR 版本和 aconfig 开关。
- **候选**：`DeviceFacts.kt:76-81` 的 `mainReadyForWorkDisplay` 要求主屏 inputReady（解锁可交互）且 STATE_ON；不加 ALWAYS_UNLOCKED/OWN_GROUP，不保活（`displays.md:36-40`）。

### 3.8 销毁与 task 去向

- **AOSP**【上游，A14】：
  - `DisplayWindowSettings.java:160-172`：PRIVATE 屏默认销毁内容；**其他屏（含 PUBLIC 虚拟屏）默认 `MOVE_TO_PRIMARY`**；
  - `TaskDisplayArea.java:1818`、`:1840`：带 `shouldDestroyContentOnRemove()` 时销毁。
- **scrcpy**：
  - 默认带 DESTROY（`NewDisplayCapture.java:218-220`），v3.0 首版就无条件带；
  - `--no-vd-destroy-content`（v3.1，`6c6607d4`）去掉它，提交说明是"the running apps are moved to the main display rather than being destroyed"；
  - 文档 `virtual-display.md:94-102`："By default, when the virtual display is closed, the running apps are destroyed"；
  - 退出时 `virtualDisplay.release()`（`NewDisplayCapture.java:296-310`）；进程死亡时由平台通过 binder death 释放，scrcpy 没有专门处理；
  - **默认值的理由**：提交说明和文档都没有写，PR #5615 网页返回 403，未能核实。从行为上推断，scrcpy 关窗即关屏，如果不销毁，屏上的 App 会突然出现在用户主屏最前面（AOSP 默认 MOVE_TO_PRIMARY）。
- **Mika**：`ShellService.kt:741-760` 的 `finishDisplay`。按 `AppsAfterTask` 逐个任务决定去留：留下的先 `setHidden`，再 `moveRootTaskToDisplay` 到主屏，然后 reorder 到最底层；其余随 DESTROY 销毁。还处理了"正在出声"的任务。被留下的任务记进 `kept`，下次打开同一 App 时挪回来（`:736-740` 的 `kept`）。
- **dsh-go**：`PrivilegedDisplayCore.kt:909-944` 的 `releasePhysical`。先 release，然后最多等 1 秒确认该屏任务清空，否则报 `releasePending`；任务列表读不到时不阻塞。
- **Pinky**：`background.go:799-823` 的 `returnTheirs` 用 `cmd activity display move-stack <root> 0` 把含原属用户任务的 root 送回主屏，失败只记日志；之后关屏销毁其余任务（`:899-910`）。`KeepContent` 可选（`:32-34`，`:235`）。
- **候选**：只做 DESTROY，不归还。`displays.md:73-75` 写明可能销毁被迁入的任务；close 结果"does not confirm that Android's task list has already removed every display task"（`:46-47`）。

### 3.9 文档里对用户写明的限制

- **scrcpy**（`doc/virtual-display.md`，114 行，包含 flex 等功能说明）：
  - 有的设备虚拟屏里有 launcher，没有时屏是空的，必须 `--start-app`（`:55-60`）；
  - 没有 App 时不产生视频帧（`:90-91`）；
  - 有的设备开系统装饰后 UI 损坏（`:87-88`）；
  - 关屏默认销毁 App（`:96`）；
  - 默认键盘出现在主屏（`:107`）；
  - 用 `--keep-active` 防熄屏（`:33`）。
  - 文档没有提已有任务被搬走、焦点、锁屏这三点。
- **Pinky**（`background.go:44-53` 的注释）：读树需要 companion；截图不支持；有的 App 拒绝副屏，会开到主屏。
- **候选**：`displays.md` 共 90 行，其中约 40 行在讲 kind、偏好文件、batch 冻结 kind。这些内容按用户新决定（显式会话）会被删掉。

---

## 4. 报告 B/A 关键描述的核实结果

| 报告说法 | 核实 | 出处 |
|---|---|---|
| B：scrcpy 33+/34+ 加可信和独立焦点，没有 no-steal | 准确 | scrcpy `NewDisplayCapture.java:224-232` |
| B：scrcpy 有 ALWAYS_UNLOCKED | 准确（33+） | 同上 |
| B：scrcpy 输入法"不设（键盘弹主屏）" | 准确，默认值来自 AOSP FALLBACK | scrcpy `Options.java:55`（-1）；AOSP A14 `DisplayWindowSettings.java:229-238` |
| B：scrcpy 保活为 `--keep-active` | 准确，按屏 userActivity，可选 | `Controller.java:250-262` |
| B：scrcpy 没有启动前保护，没有自检 | 准确；而且 `startApp` 不看返回值 | `Device.java:298-321` |
| B：Mika 抛异常时降级重建并继续用全部能力 | 准确 | Mika `ShellService.kt:103-120, 1814` |
| B：Mika 输入法 LOCAL、按屏 4s userActivity | 准确；**保活只在读回 group≠0 时启用**，B 没写这一条件 | `DisplayAwake.kt:52-53` |
| B：Mika 在用户正在主屏使用时拒绝 | 准确，判据是"可见" | `ShellService.kt:633-638` |
| B：dsh-go 取不到 flag 常量就拒绝，建屏后不读回 | 准确 | `PrivilegedDisplayCore.kt:752-767` |
| B：dsh-go 锁屏即拒，720 短边，dpi 320，HIDE | 准确 | `:317, 741, 764, 771` |
| B：dsh-go "启动前保护：无" | **不准确**：它用 `MULTIPLE_TASK`，另有 4s 落点轮询 | `:865-887`（B 版 `:908`） |
| B：Pinky 33+ 默认后台屏、4s KEYCODE_UNKNOWN、记账送回、`screencap -d` | 准确 | `background.go:100-106, 301, 651-679, 799-823, 474-487` |
| B §3.3 第 7 条：按 displayId 持唤醒锁可让 VD 独立保活 | **不成立**：Pinky 在 A17 真机上记录"display-scoped wake lock does not help"；包内 E7 在 A16 AVD 上也是 group0 | Pinky `dark.go:18-30`；`research/display/report.md` E7 |
| B §3.4：默认 HIDE，`--ime local` 留给特殊编辑器 | HIDE 合理；LOCAL 被 AOSP 的 no-steal 说明否定（Mika 的反例待真机核实） | AOSP A14 `Display.java:356-359` |
| B §3.6：开发者可用 `scrcpy --display-id=N` 镜像 PUBLIC 屏 | 未核实（研究报告也标为待验） | — |
| A §4.3：Pinky 捆绑的 scrcpy 5.0.1 有 OWN_FOCUS，没有 no-steal | 准确（以 v5.0.1 源码为准） | `NewDisplayCapture.java:211-232` |
| A §4.6：MULTIPLE_TASK 不是通用隔离 | 与 AOSP 一致（转述研究报告 `ActivityStarter.java:2856-2884`，本轮未重下载） | — |

---

## 5. 其它开源实现（检查结果）

逐个浅克隆并 grep 了 `createVirtualDisplay|VIRTUAL_DISPLAY_FLAG|setLaunchDisplayId|new-display`：

- `mobile-next/mobilecli`（`6f45e5a`）：`agents/android/java/DisplayUtils.java:63-68` 有 `createVirtualDisplay`，但只是对主屏做镜像用于推流，**不是独立工作屏**，与本题无关。
- `yume-chan/ya-webadb`（`d22e9dc`）：`libraries/scrcpy/src/3_0/impl/*.ts` 只是 scrcpy 客户端选项的封装（newDisplay 等），实现仍是 scrcpy-server。
- `microsoft/Hybrid-Computer-Use`（`00eaeeb`）、`nidhoggfgg/PhoneUse`（`b2bf4ab`）、`danielealbano/android-remote-control-mcp`（`2e2582f`）、`callstack/agent-device`（`e0ab0c2`）：都没有虚拟屏实现。

没有找到第五个能下载源码、在 shell 权限下建独立工作屏的实现，因此本报告不再列其它项目。

---

## 6. AndroidCtl 应采用的做法

| # | 做法 | 上游出处 | 对候选的具体改动 |
|---|---|---|---|
| A1 | 必需的 flag 按名字取；任一缺失、或读回缺 TRUSTED/OWN_FOCUS/no-steal，就释放并拒绝，不降级 | dsh-go `PrivilegedDisplayCore.kt:752-760`；候选读回比上游更强 | 保留 `WorkDisplay.kt:37-70`。**把 `NoSuchFieldException`、`SecurityException` 和 API<34 映射为 `displayUnsupported`**，可以在 `create()` 开头先判断 `SDK_INT >= 34` |
| A2 | 加 `TOUCH_FEEDBACK_DISABLED` | scrcpy `NewDisplayCapture.java:228`；Mika `ShellService.kt:1782` | 在 flag 列表加一项。它不在必需读回集合里。作用是在用户开了"显示触摸"时，避免截图上出现触点 |
| A3 | 尺寸和 dpi 取主屏自然方向 | scrcpy `NewDisplayCapture.java:115-127`（`443f315f`：natural orientation） | 候选用 `getRealMetrics`，取的是**当前**方向。主屏横置时会建出横屏的 VD。改为按 rotation 交换宽高（scrcpy `:119-121`） |
| A4 | 启动用 `startActivityAsUser`，shell 包名，带 `setLaunchDisplayId` | scrcpy `Device.java:307-320`；Mika `ShellService.kt:271-283` | 已一致，保留。不加 Mika 的 `am start` 回退 |
| A5 | **落点在有界时间内轮询，按包名和 UID 判定** | dsh-go `PrivilegedDisplayCore.kt:877-887`（4s/120ms，`targetLeftDisplay`/`launchUnconfirmed`） | `recordVirtualLanding` 改为在请求预算内轮询"目标屏顶部任务的包名和 UID"，不再要求 `actualActivity == 解析组件`（跳板 Activity 会让它失败）。只有超时仍不在本屏才报 `elsewhere` 或 `unconfirmed`。这能直接消除两次实测中的 `landing unknown/exit 1` |
| A6 | 输入注入用 setDisplayId，并按目标 UID 定向 | Mika `ShellService.kt:1665-1680` | 已一致，保留 |
| A7 | 销毁时默认 DESTROY_CONTENT_ON_REMOVAL | scrcpy 默认值（`NewDisplayCapture.java:218-220`，`virtual-display.md:96`）；AOSP 对 PUBLIC 屏默认 MOVE_TO_PRIMARY（`DisplayWindowSettings.java:168-169`） | 已一致。**不加** `--keep` 或归还选项（理由见 §8(a)） |
| A8 | release 后有界等待该屏任务清空，再回报 close | dsh-go `PrivilegedDisplayCore.kt:934-941`（1s，`releasePending`） | 候选目前明确不确认任务清空（`displays.md:46-47`）。如果要让 `session close: done` 意味着"屏和任务都已消失"，就照 dsh-go 加一个有界轮询；如果不加，就保持现状，这是一个取舍 |
| A9 | 不保活、要求主屏亮且解锁；以后如果要保活，必须按**读回的 displayGroupId** 决定 | Mika `DisplayAwake.kt:52-53`；Pinky `dark.go:18-30` | 已一致。文档只写一句"主屏熄灭或锁屏时后台屏停止" |
| A10 | 文本输入只承诺节点通道（SET_TEXT、IME_ENTER、PASTE）和按键注入，不承诺软键盘 | AOSP `Display.java:356-359`；scrcpy `Controller.java:707-718`（剪贴板加 PASTE 键）；Mika `AndroidScreen.kt:955` | 与"`clipboard set` + 通用 PASTE"一致。PASTE 在虚拟屏上用节点 `ACTION_PASTE` 还是 `KEYCODE_PASTE`（按 displayId 加 UID 注入），见设备清单 #2 |
| A11 | 显式设置 IME 策略 HIDE（一次调用） | dsh-go `:771`；scrcpy 提供该开关 | 理由：如果某些 ROM 放宽了 no-steal 的 IME 门槛（Mika 的注释暗示可能），AOSP 默认 FALLBACK 会把键盘弹到**用户主屏**。HIDE 能把这种可能直接关掉。建议调用失败就让建屏失败（可信屏上 shell 应当能调用），不做 best-effort 分支。该条依赖设备清单 #1 |
| A12 | 用户文档采用 scrcpy 式的短限制列表 | scrcpy `virtual-display.md` | 见 §7 |

## 7. 不应照抄的做法

| 做法 | 出处 | 不抄的原因 |
|---|---|---|
| 开系统装饰 | scrcpy 默认（`NewDisplayCapture.java:221-223`） | 那是给人看的桌面。后台屏不需要 home/recents，而且 scrcpy 自己承认部分设备 UI 会损坏（`145b823b`） |
| 不加 no-steal | scrcpy、Pinky | scrcpy 的使用场景是"人在 PC 上操作这块屏"，抢主屏焦点没有关系；AndroidCtl 的前提是"人同时在用主屏" |
| 33+ 默认 ALWAYS_UNLOCKED | scrcpy、Mika、Pinky（Pinky 还默认走后台屏） | 这改变了授权语义（锁着的手机被远程操作），而且 A17 起 VD 与主屏同组时它也保不住运行（Pinky `dark.go`）。用户没有要求锁屏工作 |
| `--start-app=+pkg` 强停 | scrcpy `Device.java:317-319` | 强停会结束用户在主屏上的同一个 App 和未保存状态，与"不打扰用户"相反。用户的规则是"先关掉主屏上的任务，或改用主屏会话" |
| 启动不看返回值，不核实落点 | scrcpy `Device.java:320` | scrcpy 有人看着画面；agent 需要结果 |
| `--keep-active`（按屏 userActivity）、KEYCODE_UNKNOWN 保活、黑屏保醒 | scrcpy、Pinky | VD 在主屏组时，这些会让用户主屏也不熄灭；Pinky 为 A17 专门写了 dark 逻辑，维护面大 |
| 缺可信时降级建屏 | Mika `ShellService.kt:103-120` | 会抢主屏焦点，失败语义不可比 |
| `am start --display`、`am display move-stack` shell 回退 | Mika、Pinky | 同一意图两条路径；shell 命令在部分 ROM 上行为不同 |
| 外跳自动拉回、逐个任务留或关、音频判定、`kept` 续用 | Mika `AndroidScreen.kt:1657-1760`，`ShellService.kt:741-760` | 维护面很大，依赖启发式；agent 只需要知道"落到了别处"即可 |
| 启动前记账、关屏送回 | Pinky `background.go:651-679, 799-823` | 只在正常关闭时生效，worker 死亡时无效；需要台账和 root/child 映射。候选"有冲突就不发送"能从源头避免这个问题 |
| 固定 720/320 尺寸 | dsh-go | 会改变 App 布局断点 |
| 用 scrcpy 视频会话承载 VD，`screencap -d` 截图 | Pinky | 静帧 CLI 不需要编码器；`screencap -d` 对 VD 不可用（Pinky 自己的注释） |
| 启动后 nudge（发无效键、重复启动） | Pinky `background.go:681-745` | 违背"不自动重发" |
| 把自检做成产品流程（探测屏、接收窗口、一次性 owner） | dsh-go `selfCheckProbe`、Mika 自检页 | 面向人的诊断页。对 agent 来说，`session open --display virtual` 的成功或失败本身就是判据 |
| 单独的 IME policy 开关（local/fallback/hide） | scrcpy `--display-ime-policy` | no-steal 下 local 无效，fallback 有害，只剩 hide 有意义，所以不需要开关 |

---

## 8. 四个特别问题

**(a) scrcpy 对"销屏时销毁内容"的默认值与理由，对 AndroidCtl 会话结束的启发**
- 事实：scrcpy v3.0 起默认销毁（`NewDisplayCapture.java:218-220`；首个提交 `98ed5eb6` 无条件带这个 flag）。v3.1 加了 `--no-vd-destroy-content`，改为把 App 搬回主屏（`6c6607d4`）。文档只写了行为，没有写理由（PR 讨论 403，未能核实）。理由只能从行为推断：AOSP 对 PUBLIC 屏的默认是 MOVE_TO_PRIMARY（A14 `DisplayWindowSettings.java:168-169`），不带这个 flag，关屏时 App 会突然出现在用户主屏最前面。
- 对 AndroidCtl 的启发：
  1. 会话结束（close、到期、接管、worker 重建）都应销毁屏上的内容。worker 死亡时平台释放 VD，DESTROY 同样生效，这是唯一在所有结束路径上行为一致的选项。
  2. "搬回主屏"在 scrcpy 中是**用户显式选择**的开关，而且 scrcpy 的使用者正看着那块屏。AndroidCtl 的使用者是 agent，"把 App 搬到真人主屏"是一个影响真人的副作用，首版不提供 `--keep`。
  3. 销毁的风险只存在于"被搬进来的用户旧任务"。候选的启动预检已经挡住了已知冲突。剩下 App 内部跳转时迁入旧任务的情况，在 skill 里写一句即可，不要为它建归还机制。

**(b) scrcpy 的 IME policy 选项说明后台屏上文本输入能做到什么**
- 这些选项只决定**键盘窗口在哪里显示**：local 是本屏，fallback 是默认屏，hide 是不显示（`cli.c:373-382`）。它们不决定 IME 能否连到虚拟屏上的编辑器。AOSP 明确规定 IME 只服务顶层焦点屏，no-steal 屏隐含禁用 IME（A14 `Display.java:356-359`）。scrcpy 不加 no-steal，所以它的 `local` 在"VD 抢到焦点"时有效，但代价是用户主屏失去焦点。
- 因此在 AndroidCtl 的后台屏上（必须带 no-steal）：
  - 能做：节点 `SET_TEXT`（直接 setText）、节点 `IME_ENTER`、节点 `ACTION_PASTE`；按键事件注入（setDisplayId 加 UID，需要 OWN_FOCUS 让 VD 有自己的焦点窗口，E1b 已实测 VD window focused=true）；剪贴板加 `KEYCODE_PASTE`（scrcpy 的通道）。
  - 不能做：需要真实 InputConnection 的输入，例如自绘编辑器、联想或组合输入、依赖 `commitText` 的控件。这类需求应该在主屏会话里完成（IC 已经只在主屏）。
  - 唯一需要真机确认的例外是 Mika 声称的"A14 真机上 LOCAL 能弹出键盘"（设备清单 #1）。

**(c) 有没有"先探测能力再建屏"可以借鉴**
- 没有一家在建屏之前做独立探测。可借鉴的是三个"建屏后读回"的模式：
  1. 读回 flag 位（Mika 自检 `AndroidSelfCheck.kt:177-183`；候选已做）；
  2. 读回 `displayGroupId`，再决定保活与否（Mika `DisplayAwake.kt:52-53`、Pinky `background.go:210-213`）；
  3. 注入后等回执（dsh-go `selfCheckProbe`）。
- 第 3 个对 AndroidCtl 没有必要，因为第一条真实 UI 命令就会得到结果。
- 建议：`session open --display virtual` = 建屏 + 读回 + 等 ON；失败统一报 `displayUnsupported`，并附上读回的 flag 或异常类名。不新增 `doctor --display`，也不在 status 中预报"支持与否"。

**(d) 有没有实现处理"App 已在主屏有 task"**
- 有三家处理，各不相同：
  - **Mika**：主屏**可见**任务就拒绝（`START_IN_USE_ON_MAIN`），后台任务允许被系统搬进来；结束时按策略挪到主屏最底层或销毁（`ShellService.kt:257-268, 633-638, 741-760`）。
  - **Pinky**：启动前记下所有他屏任务；正常关屏时 `move-stack` 送回，失败只记日志（`background.go:651-679, 799-823`）。
  - **dsh-go**：`NEW_TASK|MULTIPLE_TASK` 另开任务，再轮询确认落点；落到别处报 `targetLeftDisplay`（`PrivilegedDisplayCore.kt:865-887`）。
  - scrcpy 不处理，只提供 `+` 强停。
- 候选的规则是任何他屏已知任务都不发送，比四家都严。它少了一类损失（关屏销毁用户旧任务），多了一类失败（常用 App 常驻后台就无法开到虚拟屏）。是否值得引入 MULTIPLE_TASK，取决于设备清单 #4 的结果。

---

## 9. 与候选逐项对照

| 方面 | 与成熟实现一致 | 比上游重 | 比上游弱 |
|---|---|---|---|
| flag | 7 项与 dsh-go 相同；含 no-steal（优于 scrcpy/Pinky） | — | 缺 TOUCH_FEEDBACK_DISABLED（scrcpy、Mika 都有）；尺寸未按自然方向 |
| 失败处理 | 不降级（同 dsh-go） | — | 错误码：API<34 和缺权限都会落到 `workerFailure` |
| 读回 | 比四家都强（每次建屏都读回三位） | — | — |
| 启动 | binder 方式加 shell 包名（同 scrcpy、Mika） | 预检要求 root 与 live 两份事实一致，不一致报 unknown，比 Mika 的"读 isVisible"复杂得多 | 落点只读一次、精确匹配 Activity → 两次实测都是 unknown/exit1；dsh-go 的轮询更可用 |
| 已有任务 | 拒绝已知冲突（方向与 Mika 一致，更严） | — | 无 |
| 输入 | 按 UID 定向（同 Mika，优于 scrcpy） | — | — |
| IME | 不依赖软键盘 | — | 未设 HIDE（dsh-go 设了）。依赖平台的 no-steal 隐式禁用 |
| 画面 | 能截任意时刻 | ImageReader 只排空，另用 captureDisplay，等于两套资源 | 不提供帧时间戳（文档已写明） |
| 电源和锁屏 | 不保活，要求主屏亮且解锁，方向正确（与 A17 平台变化一致） | — | — |
| 销毁 | DESTROY（同 scrcpy 默认值、dsh-go） | — | close 不确认任务已清空（dsh-go 有 1s 有界确认） |
| 文档 | 写明销毁会丢迁入任务、IC 仅主屏 | `displays.md` 有 90 行，其中约一半是 kind 和偏好机制（按用户决定删除） | — |

**按显式会话模型重做时，给实施者的最小清单**：

1. 建屏入口改为 `session open --display virtual`。保留 `WorkDisplay` 的 flag、读回和等 ON；加 TOUCH_FEEDBACK_DISABLED 和 HIDE；尺寸按自然方向；错误统一为 `displayUnsupported`。
2. 落点改为"预算内轮询目标屏顶部任务的包名和 UID"。
3. 预检保留"任何他屏已知任务都不发送"，结果只给冲突的包名和所在屏，不输出 `conflictsOmitted` 等细节字段。
4. close 释放 VD。是否等任务清空由 A8 决定；如果等，就照 dsh-go 用 1s 上限。
5. `displays.md` 缩成 scrcpy 式的限制列表，大约 10 条：
   - 需要 Android 14+ 且 ROM 保留 shell 的可信显示权限，否则报 displayUnsupported；
   - 主屏必须亮且解锁；
   - 没有桌面、home 和 recents；
   - 没有软键盘，文本走 fill 或 paste；
   - 已在其他屏有任务的 App 不能打开；
   - 有的 App 会开到主屏（报 elsewhere）；
   - 会话结束时销毁屏上全部内容；
   - 截图按需抓取。

---

## 10. 必须上设备验证的清单（只列影响设计的）

所有命令都在真机上执行。理想覆盖是 Pixel A14/A15/A16 各一台，加至少一台国产 ROM。`<N>` 指 `session open --display virtual` 返回的 displayId。

1. **no-steal 屏上能否出现 IME，以及 HIDE 是否必要**
   - 操作：在虚拟屏打开一个带 EditText 的 App，用 `click` 聚焦输入框；分别在不设策略和设 HIDE 两种情况下执行。
   - 观察：`adb shell dumpsys input_method | grep -E 'mCurTokenDisplayId|mInputShown|mCurClient'`；`adb shell dumpsys window | grep -E 'mCurrentFocus|imeLayeringTarget|Display: mDisplayId'`；同时观察主屏是否弹出键盘。
   - 判读：
     - 两种情况下都没有 IME：符合 AOSP 说明，HIDE 只是保险，可以保留也可以去掉；
     - 不设策略时主屏弹出键盘：**HIDE 必须做**；
     - IME 出现在 VD 上且主屏焦点不变（Mika 的说法）：该 ROM 放宽了门槛，可以考虑以后开放 IC 后台输入，但不改首版。
2. **PASTE 通道**
   - 操作：`clipboard set "测试abc"`，然后对 VD 上的输入框分别执行节点 `ACTION_PASTE` 和按 `displayId=<N>` + UID 注入的 `KEYCODE_PASTE`。
   - 观察：节点文本回读。
   - 判读：决定通用 PASTE 动作在虚拟屏上用哪条通道；两条都失败，就说明后台屏只能用 fill。
3. **落点轮询的时长**
   - 操作：用当前 `open` 打开 VLC VIEW、设置、浏览器各 5 次。每次在启动返回后每 100ms 执行一次 `adb shell dumpsys activity activities | grep -E 'Display #|\* Task|topActivity'`，直到任务出现在 display N。
   - 观察：出现所需的时间，以及顶部 Activity 是否等于解析出的 Activity。
   - 判读：p95 < 2s 时用 4s 上限即可；顶部 Activity 常与解析结果不同，就证实必须按包名和 UID 判定（A5）。
4. **已有任务的覆盖面，以及 MULTIPLE_TASK 是否可用**
   - 操作 1：在日常使用状态下执行 `adb shell cmd activity get-...`（或 `dumpsys activity recents`），统计微信、浏览器、地图等常用 App 有后台任务的比例。
   - 操作 2：对有后台任务的 standard 启动 App 执行 `adb shell am start --display <N> -f 0x18000000 -n PKG/ACT`（NEW_TASK|MULTIPLE_TASK），检查主屏原任务的 taskId、display 和页面状态是否不变，VD 上是否出现新任务；对 singleTask 的 App 重复同样操作。
   - 判读：
     - 多数常用 App 有后台任务、而 MULTIPLE_TASK 能保留原任务：值得把 MULTIPLE_TASK 作为 standard App 的默认，预检只拦 singleTask/singleInstance；
     - MULTIPLE_TASK 仍会搬动原任务，或产生 App 不支持的双实例：保持"有冲突就不发送"。
5. **OEM 是否保留建屏权限**
   - 操作：`adb shell dumpsys package com.android.shell | grep -E 'ADD_TRUSTED_DISPLAY|INTERNAL_SYSTEM_WINDOW'`，然后执行 `session open --display virtual`。
   - 观察：权限是否 granted；actualFlags 是否包含 0x1880。
   - 判读：缺权限的 ROM 必须稳定返回 `displayUnsupported`（验证 A1 修复）；有权限但读回缺位，说明该 ROM 会剥 flag，也同样拒绝。
6. **主屏熄灭时 VD 的状态**
   - 操作：会话进行中让主屏自然超时熄灭。
   - 观察：`adb shell dumpsys display | grep -E 'mDisplayGroupId|Display Groups|mState'` 和 `dumpsys power | grep -i wakefulness`，然后在 VD 上执行一次 click。
   - 判读：如果 VD 与主屏同组并一起 OFF，"要求主屏亮且解锁"的规则就足够，错误码应为输入不可用；如果某 ROM 上 VD 独立组仍 ON，也不改设计，因为不承诺锁屏工作。
7. **横屏 App 和自然方向**
   - 操作 1：在主屏横置时开会话，检查 VD 尺寸（验证 A3）。
   - 操作 2：打开强制横屏的 App（例如视频全屏），截图并点击一个已知坐标。
   - 判读：
     - App 被 letterbox 但坐标正确：维持不加 ROTATES_WITH_CONTENT；
     - 截图方向与输入映射不一致：必须加 ROTATES 并处理几何，参考 Mika `ensureOrientationWatch` 的注释"STEAL_TOP_FOCUS_DISABLED…有的系统因此不再采纳后台 Activity 的转屏请求"。
8. **worker 死亡后的任务去向**
   - 操作：在 VD 上打开一个自有测试 App，`kill` 掉 worker 进程。
   - 观察：`adb shell dumpsys activity activities | grep -B2 -A5 '<pkg>'`。
   - 判读：任务被销毁、没有出现在主屏，说明 DESTROY 在 binder death 路径上生效，与正常 close 一致；任务出现在主屏，说明需要在文档中改写风险说明（这种情况下不建议补救机制）。
