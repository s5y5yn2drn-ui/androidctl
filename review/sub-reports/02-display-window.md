# 专题 02：显示目标、窗口选择、租约与身份一致性（方向 17/18/19/38，接点 1，顺带 33）

评审对象：候选 HEAD `3f45af6`（基线 `7d5be70`）。行号均取自 3f45af6。没有设备，没有运行任何包内脚本。证据类别标注：【实测】=包内原件记录的真实运行（注明是否当前候选）；【源码】=源码推断；【设计】=设计建议。

---

## 1. 结论摘要（按重要性排序）

1. **必填种类这套机制，唯一一次在包内真实发生的「本想上后台屏、结果打到主屏」事故，它没有拦住。**【实测，候选 99e31a9 ≈ 当前 38 号提交修正前】API37 上 `display create` 返回 failed，操作 agent 没检查 exit 就续发了 VLC VIEW，VIEW 落到 **main display0 / task84**（`stages/final-verification/runtime-release-api37/result.md`）。原因在主机：`app.py:839` 只在 create `done` 之后才写 `virtual`，失败或 UNKNOWN 都不写，偏好仍是 main，于是设备「合法地」隐式取得了 main 租约。交付方把它归为「操作者错误」。可这个产品的使用者本来就是 agent，而必填 kind 要防的正是这种疏忽。**建议改为 create 发出前先写 virtual（写失败就 not sent）**。这样能补上这个洞，同时还能删掉「已建屏但偏好没保存」这一分列状态和对应的非零退出分支。属于净简化。
2. **方向 38 的能力增量，以现有证据看适用面很窄，维护面却很宽。**【实测 + 源码】
   - 已证实的：2 台 AOSP AVD（API34/37）上能建屏、在后台屏上节点/坐标操作、主屏隔离、zoom、close/create/open。
   - 已证实不行的：MuMu 不支持。
   - 3 次真实 `open` 全是 `landing unknown` + exit 1，会让 batch 停下。
   - 没有真机、没有 ROM 证据。
   - 「目标包在别的屏已有 task 就拒发」在日常手机上会挡掉多少常用 App，没人量过。
   - 设计前提是主屏亮且解锁、后台屏同电源组。所以实际上它只服务「真人此刻正在用手机、agent 在旁边并行干活」这一场景；模拟器上几乎没有收益。
   - 一个提交约 +2950 行，跨约 25 处 displayId 依赖；每条 UI 请求、`ui.read` 和 batch 都多了一个身份维度。
   - 建议：要么拆出首次合入（推荐），要么保留但做下文的小修。
3. **对 LLM 来说，「主机级 deviceId+session 偏好」最常见的失败模式是遗留的 virtual，而不是隐式回 main。**【源码】
   - 会话名默认是 `"default"`（`app.py:97-106`）。偏好按主机共享，`session close`、到期、worker 重建都不清它。
   - 所以一个任务用过后台屏之后，同设备、同会话名（尤其是 default）的后续任务，每条 UI 命令都会被 `displayRequired` 拒绝。
   - 拒绝文案只提示 `run display create`（`Sessions.kt:157`），不提 `display select main`。
   - `status` 也不显示主机偏好（`app.py:153-180` 只显示设备实际 target）。
   - 结果是 agent 很可能「按提示」去建一块它根本不需要的后台屏。修法很便宜：status 显示偏好及其来源，`displayRequired` 文案补上 `display select main`。
4. **必填 kind 本身（设备侧）值得保留，代价低、语义清楚。** `Sessions.requireTarget`（`Sessions.kt:152-170`）加上 `DisplayKind.requested`（`DisplayTarget.kt:13-16`）不到 40 行。它是唯一能扛住 worker 重建的「不静默落 main」保护。替代方案里，(a) session 属性和 (b) 逐条 `--display` 都会在默认值处重新引入同一事故；(c) 设备端墓碑会在 worker 重建时失效。所以我**不建议推翻 U23 的「协议必填」**，只建议改主机写入时点（见第 4 节 R1）。
5. **「是否变化」现在至少有 5 种编码**：`settle`（含 changed/unchanged）、`observedChange.state`、`observedChange.{windows,selection,content}`、`effect.state`、`verification`，加上 `absentWindow` 时被强制写成的 `settle=changed`（`UiPostAction.kt:102`）。
   - 默认文本摘要每个动作都会打印 3 个维度 JSON 和 4 个基线数字（`ui_client.py:270-288`）。真实 agent 的 fill 摘要行实测就是这样（`paired-runtime/contacts-2/evidence/cli/.../child.stdout`）。
   - 建议：`settle` 只表达稳定性（`stable/loading/unsettled`）；去掉 `absentWindow` 对 settle 的覆盖；文本模式只印 `observedChange` 的结论维度和 `baselineAgeMs`，JSON 保全。
6. **显式窗口缺席后的行为前后矛盾。**
   - 动作后，同连接目录里查不到窗口就足以切换 scope，并报 `settle=changed`（`UiBackend.kt:415-428`、`UiPostAction.kt:97-110`）。
   - `wait --window W --gone/--contains/selector` 遇到 W 已不在时，却会一直等到超时再报 failed（`TreeCollector.kt:107-118` 判 `windowUnavailable`；`UiCondition.kt:64-75` 视为缺样本；`waitUiCondition` 循环，`UiCondition.kt:395-411`）。
   - 窗口 ID 不会复用，缺席就是永久的，空等没有意义。
   - 建议：wait 遇到「同连接完整目录中请求窗口缺席」时**立即**返回 failed 并给 `windowAbsent`（不判 gone，以尊重 U25「缺席≠gone」）。
   - 至于交付方未批准的「缺席后续等 scopeUnknown」：建议**不新增这项特殊政策**，改为删掉 `absentWindow` 的提前退出分支，让缺席后直接走现成的默认范围 settle 循环。这比 B 方案更简单，也更一致。
7. **后台屏上的 `open` 落点判定过严，产生了可复现的错误非零退出。**【实测 3/3 + 源码】
   - `ActivityLaunch.kt:254-258` 先置 exit 1，`recordVirtualLanding`（`:312-321`）在 startActivity 返回后**立即只读一次** task 表，要求顶部 activity 等于解析出的组件。
   - 跳板/启动页（VLC 的 StartActivity→VideoPlayerActivity）注定判 unknown → exit 1。主屏对同样情况不罚 exit。
   - 建议：只有 `elsewhere`（已正面观察到落在别的屏）才非零退出；`unknown` 与主屏同等处理，或者把判定放进已有的 settle 末样本。
8. **WorkDisplay 的 reader/drain/consumer/线程结构整体不算过度**（150 行，`captureDisplay` 已替代长持 Image）。两处可以收窄：
   - 创建时的 ON 门与每次输入时的 ON 门重复（`WorkDisplay.kt:71-77` 对比 `UiBackend.kt:75-79`）。API37 的 UNKNOWN 误拒正是这道冗余门造成的；修正方案（UNKNOWN 时按原预算等）可以接受，但最简单的做法是删掉创建门。
   - consumer 2s 不退出就转成全局 `uiCleanup`（`WorkDisplay.kt:128-135`），对一个只做 `acquireLatestImage().close()` 的线程过于防御。
9. **控制者通知（33）没有引入第二个状态权威**：它读的是同一份 worker status（`GatewayService.kt:32-38,74-91`）。有一点需要知道：status 读取会调 `Sessions.expire()`（`Sessions.kt:251-254`），而 worker 自己已有 1s 维护定时器（`WorkerService.kt:30,39-46`）。读状态带副作用，好在是冗余而非错误。2s 轮询本身可以接受。

---

## 2. 负责方向处置表

| 方向 | 处置 | 依据 | 依赖 | 真实能力损失（按建议改） | 维护代价变化 |
| --- | --- | --- | --- | --- | --- |
| 17 显式窗口集合 | **保留，小调整** | `window/windows/package/allWindows/default` 五种 scope 各有真实用途（多窗、系统窗、IME 场景）；默认单窗成本对。问题集中在缺席处理和 wait 的超时空等。 | 18、19、post-action | 无 | 删掉 `absentWindow` 提前退出分支（`UiPostAction.kt:43-44,67`），wait 增加一个「缺席即失败」判断：净持平或略减 |
| 18 任一条件共享样本 | **保留** | `UiWait` 在同一样本上算全部条件、取首个匹配（`UiCondition.kt:325-347`），模型简单，正确。 | 17 | 无 | 无 |
| 19 observedChange 分维度 / settle | **调整** | 分维度本身有价值（窗口变了而内容未知是常见真实情形）。但 settle 的 changed/unchanged、top-level state 和 effect 重复表达「是否变化」，`absentWindow` 还改写 settle。文本默认输出噪声大。 | 16、39 | settle 不再给 changed/unchanged：A→B→A 的「中途变过又回来」信息会降为 `unsettled`（现在也大多如此，见 `SettledSamples.classification`） | 字段与文档减少；host 摘要逻辑简化 |
| 38 显示目标 / 后台屏 | **证据不足 → 推荐拆出首次合入；若保留则调整** | 见结论 1–3、7；真机适用面、task 冲突拒发率、落点误报均未量化或已实测不佳。 | 1、21/22 图像、27/28 open、29 IC、33 | 拆出：失去「真人用主屏时 agent 在后台屏工作」能力（目前只在 AOSP AVD 上证实） | 拆出可去掉 kind 字段、主机偏好文件/锁、`display select/create`、WorkDisplay、TargetManagement 的 create 分支、virtual task 预检/落点及约 90 行公开文档 |
| 33 中与 lease/worker 状态相关的部分 | **保留** | 单一来源；显示 target 有用。 | 38 | 无 | 可选：status 不再调 `expire()` |

---

## 3. 发现

### F1【确定缺陷·设计层】create「成功后才保存」让真实发生过的误路由不受保护

- 证据类别：【实测】API37，候选 `99e31a9`（UNKNOWN 修正前），misroute 原件 `stages/final-verification/runtime-release-api37/commands/vlc-virtual-view-once/stdout.bin` 及 `after-misrouted-view-activities`；【源码】当前 `app.py:839-866`（只有 `result.result == "done"` 才调用 `save_display`），`Sessions.kt:94`（无租约时 `keepLease` 新建默认 MAIN 租约），`Sessions.kt:156-158`（只有期望 virtual 时才拒绝）。
- 触发路径：偏好为 main（首次使用，或上次 select main）→ `display create` 返回 failed、UNKNOWN 或 not sent → agent 继续发 `open/tap/...` → 主机填 `main` → 设备在 `TargetManagement` 已经使旧租约失效（`prepareChange` → `invalidateLease`）的情况下隐式新建 main 租约 → 动作作用在用户主屏。当前候选修好了 API37 的 UNKNOWN 误拒，但 MuMu 不支持、ROM 缺权限、超时 UNKNOWN 这几条路径依然存在。
- 影响：U23 想防的「本想 virtual 却落 main」只在「建屏成功之后再丢失」时被拦住，「建屏没成功」这一支没拦住。而后一支恰恰是唯一在包内真实发生过的。
- 建议（R1）：`display create` 在发 RPC **之前**写 `virtual`。写失败就 not sent（满足 U23「保存失败须报告」）；写成功后建屏失败、UNKNOWN 都保持 virtual，后续 UI 得到 `displayRequired`（安全且明确），agent 再 `display create` 重试或 `display select main` 退回。由此可以删掉 `displayPreference.saved=false`、`displayPreferenceError`、notice 和「建屏 done 但 exit 非零」的分支（`app.py:840-865`）。这与 U25 第 3 条「UNKNOWN 不更新」冲突，需要用户重新裁决。
- 对上次外评（裁决-3「先远端 create、成功后再写本地偏好」）的不同意见：它只比较了两种顺序下「部分失败」的状态数，没有考虑「失败后续发」这一更常见的 agent 行为。

### F2【设计取舍·高影响】遗留 virtual 偏好与默认会话名组合：误导性拒绝，并被引导去建一块不需要的屏

- 证据类别：【源码】`app.py:97-106`（默认 `session="default"`），`workspace.py:55-56`（缺省 main，否则读主机文件），`displays.md`（close/到期/worker 重建都保留 virtual），`Sessions.kt:157`（`displayRequired` 文案只说 `run display create`），`app.py:153-180`（`emit_status` 不显示偏好）。【实测】API34 `expired-virtual-ui-no-hidden-main`、MuMu `virtual-empty-snapshot-once`：拒绝路径按设计工作。
- 失败模式：
  - (a) 任务 A 在 default 会话里用过后台屏，正常 `session close` 收尾；数小时后，任务 B（另一个工作区，没指定会话名）的第一条 `snapshot` 被 `displayRequired` 拒绝。B 不知道有偏好这回事，按提示执行 `display create`。
  - (b) 同名会话被不同任务复用时同理。
  - (c) 偏好文件里的条目只增不减。
- 判断：这是「宁拒勿误」的有意取舍，安全方向正确。但诊断信息不足，会把用户（LLM）推向错误的修复。
- 建议：
  - `status` 增加 `displayPreference=virtual (host, since <time>)`。
  - `displayRequired` 文案改为「run display create to recreate a work display, or display select main to work on the main screen」。
  - 可选：`display create` 在 `session` 为 `default` 时给出提醒。以上都不增加机制。

### F3【设计取舍】`session open` 不写 main，必然产生一次 `displayMismatch` 往返

- 证据类别：【源码】`TargetManagement.kt:12-35`（open 总是建 MAIN），`displays.md`；【实测】API34 `explicit-main-session-open` 之后的 virtual snapshot 被 `displayMismatch` 拒绝。
- 权衡：
  - 改成写 main 可以省一轮。但 LLM 常把 `session open` 当「重置」用；在后台屏模式下误用时，现在的设计会拒绝（安全），改了以后会直接在主屏动作。
  - 由于 `session open` 已经销毁了后台屏，损失其实已经发生。这时主屏动作的危害只在「真人正在用主屏」时才实质存在，而那正是后台屏唯一的目标场景。
- 结论：**维持 U25（open 不改偏好、保留独立 select）**，只做 F2 的文案修正。我不同意上次外评「session open 写 main、删除 select」的建议。

### F4【确定缺陷·输出】后台屏 `open` 的落点判定：单次即时读 + exit 1，3/3 真实运行误报

- 证据类别：【实测】API34 两次 VLC open（`stages/37-work-display/runtime-api34/commands/vlc-typed-open-once`、`immediate-vlc-reopen`，旧物料 6f5ee3a），API37 一次（`api37-readiness/runtime-api37/commands/readiness-vlc-virtual-view-once/stdout.bin`，当前候选物料）。三次都是 `START_SUCCESS` + `landing unknown` + `exit 1`；API37 那次还因 VD 上 1.5s 内「no windows enumerated」导致 `tree=unavailable/scopeUnknown`。【源码】`ActivityLaunch.kt:254-258`（virtual 成功先 exit 1），`:312-321`（只调一次 `virtualActivityLanding`，没有等待），`ActivityTasks.kt:116-150`（要求顶部 activity ∈ {解析组件, actual}，跳板必失败）。
- 影响：在后台屏上「打开 App」这个最基本的动作，会对绝大多数带启动页的 App 返回非零，batch 因此停下。主屏没有这条规则，两屏语义不一致。
- 建议：只有 `landing.state == "elsewhere"`（真正有害：App 被拉到了别的屏）才 exit 1；`unknown` 保留事实、exit 0。或者把判定移到已有的 settle 末样本（看默认操作窗的包名和 display）。这两种都不加机制。

### F5【确定缺陷·小】显式窗口缺席时 wait 空等到超时

- 证据类别：【源码】`ObservedChange.kt:218-227` 的 `windowAbsent` 要求 `sent=true`，wait 永不派发，所以永远不会走缺席分支；`TreeCollector.kt:107-118` 判 `failed/windowUnavailable`；`UiCondition.kt:64-75` 遇到无节点就 `missing` 并不匹配；`UiCondition.kt:162-169` gone 要求 complete；`waitUiCondition` 一直循环到预算用完（`UiCondition.kt:395-411`）。【实测】外评引用的 `stages/16-wait-conditions/runtime-api34/commands/wait-absent-once` 读起来像工具故障（我没有逐字节复核）。
- 影响：`wait --window W --gone` 在 W 关闭这种最常见的「等弹窗消失」场景下，必定耗尽 timeout 后报 failed，与事实相反。`--windows [A,B]` 中有一个缺席时也一样（partial → gone 永远不成立）。
- 建议：同连接目录完整、请求窗口缺席时，wait **立即**返回 failed，`reason=windowAbsent`，并提示「用默认范围或 `--app` 等待」。不判 gone，符合 U25。是否进一步「缺席即 gone」（外评裁决-8），我的看法是：窗口 ID 不复用，对 `--gone` 来说缺席在语义上就是该范围内找不到，判成立并注明 `windowAbsent` 也是合理的。但用户已经明确保留「缺席≠gone」，快速失败已经能消除主要代价。

### F6【设计取舍】`absentWindow` 改写 settle，且只采一份默认样本

- 证据类别：【源码】`UiPostAction.kt:43-44`（缺席后不进循环）、`:67`、`:101-109`（强制 `settle="changed"`）；【实测】`reviews/u25-pending-questions-review.md` 汇总的 stage 12/13/16/17 多次「原后树 scopeUnknown，需要独立 snapshot」。
- 问题：
  - (1) settle 是稳定性分类，这里被用来再编码一遍「窗口缺席」，与 `absentWindow`、`observedChange.windows.disappeared` 重复。
  - (2) 只采一份样本，转场中大多拿不到正文。
- 对未批准选项的判断：B「只续等 scopeUnknown」能用，但它是又一个局部特例。更简单的做法是：缺席后把本次请求的有效 scope 视为默认，直接复用 `settle()` 已有的「默认 scope 遇 scopeUnknown 就 awaitNextCheck」分支（`UiPostAction.kt:48-58`），并删掉 `absentWindow` 的三处特例和 settle 覆盖。效果介于 B 和 C 之间，截止仍是原截止。代码是减少的，也不新增政策词汇。

### F7【设计取舍·中】「是否变化」的多重编码与默认文本噪声

- 证据类别：【源码】`UiPostAction.kt:155-218`（SettledSamples 既判稳定，又与 baseline 比出 CHANGED/UNCHANGED），`ObservedChange.kt:71-99`（三维 + 总态），`UiEffect.kt`（原目标字段差），`ui_client.py:225-288`（文本摘要印 settle 差异、总态、三维 JSON、四个基线数字、effect）。【实测】API37 两步 batch 中 click 的一行结果：`settle=changed`、`observedChange.state=changed`、`content=changed`、`effect=unchanged`；candidate Contacts 真实 agent 的 fill 摘要行开头约 250 字节全是这些字段（`stages/final-verification/paired-runtime/contacts-2/evidence/cli/1794875053903236600-.../child.stdout`）。
- 判断：
  - 维度拆分（windows/selection/content）有真实价值。
  - `effect`（原目标对象的字段差）回答的是另一个问题，也有价值。
  - 重复主要出在两处：settle 的 changed/unchanged 与 observedChange 总态重复（交付方以 A→B→A 为由保留，但该情形 settle 已经归为 unsettled，用不着 changed/unchanged 来表达）；文本模式打印 `baselineFinishedMs`、`baselineComputedAtMs` 这种原始 uptime 毫秒，对 LLM 没有用。
- 建议：settle 收为 `stable|loading|unsettled`；文本摘要只印 `changed/unchanged/unknown(维度)` 和 `baselineAge`；JSON 不变。U25 第 5 条（基线序号/年龄）的事实保留在 JSON 里，不违背用户意图。

### F8【设计取舍·低】WorkDisplay 的创建 ON 门与 API37 修正

- 【源码】`WorkDisplay.kt:71-77` 加上 `UiBackend.kt:75-79`（每次 target 输入都检查 VD ON）、`DeviceFacts.kt:76-81`。【实测】API37 原失败：建屏后约 9ms 读到 UNKNOWN 即释放；探针显示 61ms 后变为 ON（`reviews/api37-display-readiness-audit.md`）。
- 判断：修正本身最小且正确（复用 UiRun/UiBudget，只在 UNKNOWN 时等）。但它暴露出创建门是第二个权威：输入时已经检查 ON，截图/快照在非 ON 时本来就会失败或为空。删掉创建门（只保留 flags 校验）能从根上避免同类平台时序差异。维持现状也可以接受，所以列为低优先级。
- consumer 线程 2s 未退出就记 `uiCleanup` 并封锁全部 UI（`WorkDisplay.kt:128-135` → `UiBackend.recordCleanupFailure`）：防的是一个只 `close()` Image 的 HandlerThread 卡死，包内没有发生过。可以降为记录 + 放手（泄漏一个线程比锁死整台 worker 便宜）。

### F9【证据缺口】适用面

- 已实测（当前或近似候选）：AOSP API34/37 AVD 上建屏、节点/坐标/键、主屏隔离（主屏 editor 保持焦点、零 touch）、zoom、close→create→open、到期后不隐式落 main、batch 前缀停止。MuMu 明确缺 `ADD_TRUSTED_DISPLAY`。
- 未实测：任何真机/ROM；有 task 冲突时的拒发率；锁屏/超时熄屏下的实际体验（同电源组，熄屏后 VD 也熄；有凭据锁时 `wake` 无法恢复，只能等真人）；跨屏 App 内跳转迁移旧 task（研究 E4 已证明会被销毁，产品只有文档说明）。
- 结论：对「模拟器上跑 agent」的用户，后台屏几乎没有收益（主屏本来就没人用）；对「真机」用户，前提条件很多，而且全部未验证。

### F10【设计取舍·低】后台屏模式下主屏事件进入结果

- 【源码】`UiSignals.kt:78-84`：`recent.project` 对所有 display 的事件都做投影，只有唤醒才按 display 过滤。在后台屏模式（真人正在用主屏）下，真人的 pane 标题、toast 文本、包名会进入 agent 的动作结果。这不是安全边界（shell 本来就全权），但属于噪声和隐私暴露，而且只在 VD 场景出现。交付方明确不采纳（`external-final-integration.md` 屏4）。如果保留 VD，建议至少在 `displays.md` 写明一句；不建议为此加过滤机制。

### F11【观察】身份一致性矩阵（接点 1）

| 身份 | 权威 | 失效时机 | 一致性判断 |
| --- | --- | --- | --- |
| 种类偏好 | 主机 `display.json`（device+session） | 仅显式 select / create done | batch 起始冻结（`ui_batch.py:307-315`），单条命令冻结一次（`ui_client.py:596-607`）。**缺口：F1、F2** |
| 租约实际 target | 设备 `Lease.target`，不可变 | close/open/takeover/create/到期/revoke/worker 重建 | 一致；kind 不符在任何派发前拒绝（`Sessions.kt:136-150`，诊断顺序先 owner/资源/cleanup 再 kind） |
| 窗口 scope | 请求 args，平台 window ID | 窗口消失 | 按 target display 过滤目录（`AccessibilityConnection.kt:52-66`），跨屏 ID 自然缺席。**缺口：F5、F6** |
| 节点 REF | `UiSnapshot.targets`，按观察序列 | 新安装观察、租约清理 | 租约失效 → `clear` → `snapshots.close`（`UiBackend.kt:477-501`）；新 VD 实例不会继承旧 REF。一致 |
| 坐标帧 | VisualInput 当前完整源 + meta | 派发/新截图/租约结束/120s | 布局含 displayId，`ImageRegion.kt:95,148` 核 target。一致 |
| App UID（`--expect`） | 注入时 PM UID + displayId | 每次派发 | 与种类无关，分工清楚（`displays.md` 已说明）。一致 |

另外两处冗余但无害：`ui.read` 也带 kind（`RpcMethods.kt:48-49`），但 capture 本来就绑定租约，kind 检查不会提供额外保护；status 读取会触发 `expire()`（F9 已述）。

---

## 4. 建议重新裁决的事项

### R1：create 写偏好的时点（覆盖 U25 第 3 条「UNKNOWN 不更新」）

| 选项 | 内容 | 优点 | 缺点 |
| --- | --- | --- | --- |
| A 维持 | done 后写 virtual，UNKNOWN/failed 不写，保存失败单列 | 偏好与成功事实严格对应 | 建屏失败后续发会落 main（包内已真实发生一次）；需要「建成未保存」分支与文案 |
| **B 发前写（推荐）** | 发 RPC 前写 virtual，写失败就 not sent；建屏任何结果都不回滚 | 封住 F1；删掉分列状态与非零分支；表达的是「意图」 | MuMu 等不支持设备上，失败后需要 `display select main` 才能回主屏（会被明确拒绝，不会误动作） |
| C 发前写 + 失败时回写 main | 已知 failed/not sent 时回写 main | 不支持设备上少一步 | 又多一个写入点和一种失败，复杂度回升；UNKNOWN 仍需选择 |

剩余未知：无，这是纯主机逻辑。

### R2：方向 38 是否进入首次合入

| 选项 | 内容 | 能力 | 维护 |
| --- | --- | --- | --- |
| A 全部合入（现状 + F1/F2/F4 小修） | — | 保留在 AOSP AVD 上已证实的后台屏 | 最大：协议必填字段、主机偏好、约 25 处 displayId 依赖、task 预检/落点、文档约 90 行 |
| **B 拆分（推荐）** | 先合入 `Lease.target`/按 target 取目录/输入 display 的内部贯穿（只有 MAIN），后台屏的建屏、kind 字段、主机偏好、virtual 预检落点留在分支 | 暂不提供后台屏 | 首次合入去掉 kind 字段和偏好文件；后续若补入 VD，基础已经就位 |
| C 删除 | 连内部贯穿一起回退 | 同 B | 最小，但丢掉外评认可的「默认窗判断收敛」 |
| D 合入但标记实验 | 隐藏在显式开关后 | 同 A | 同 A，再加一层开关；不推荐 |

推荐 B。理由：真实能力增量只在 AOSP AVD 上证实；目标场景（真人用主屏 + 后台屏）需要真机证据；而 kind 字段和偏好文件一旦公开发布，就变成长期协议负担。拆分有成本：9321313 是一个不可拆的大提交，交付方以「不为行数拆临时 main-only 接口」拒绝过拆分（`external-final-integration.md` 屏7）。我认为这里拆分的理由是证据不足，不是行数。

会改变判断的补证：在用户实际使用的那台手机上做一次探针：建屏权限、OWN_FOCUS/no-steal 是否保留、1 个日常 App（在 recents 中有 task）的 open 是否被拒、熄屏锁屏后的恢复路径。全部通过就倾向 A + 小修；建屏或常用 App 不可用就坚持 B。

### R3：wait 遇到显式窗口缺席

| 选项 | 内容 |
| --- | --- |
| A 维持（空等到超时，failed） | — |
| **B 立即 failed + windowAbsent（推荐）** | 不判 gone，不改 U25 原则 |
| C 判 gone 成立并标 windowAbsent | 外评裁决-8；需要用户推翻 U25「缺席≠gone」 |

### R4：动作后显式窗口缺席（交付方「未批准选项」）

推荐**第五个方案**（不在交付方 A–D 中）：删除 `absentWindow` 的提前退出与 settle 覆盖，缺席后把有效 scope 视为默认，复用现有 settle 循环（含现成的 scopeUnknown 等待）。截止不变。代码减少。

---

## 5. 对既有内部 review 与交付方声明的对照

- `reviews/api37-display-readiness-audit.md`：平台源码分析可信，修正最小。不同意的地方：它和 `external-final-integration.md` 末段都把「create 失败后 VIEW 落 main」定性为与产品无关的操作者错误。对一个以 agent 为使用者、并为此专门设计了「必填种类」的产品来说，这是该机制的覆盖缺口（F1），应作为产品问题处理。
- `reviews/final-host-evidence.md:28`：「create 只在主 done 后保存……测试收益对应实际持久化/并发责任」，作为对实现的描述是准确的。但它只核了实现与契约是否一致，没有质疑契约本身留下的缺口。
- `reviews/final-android-contracts.md:44`：「平台成功先记 done，再以非零 exit 保 pending/elsewhere」把 `unknown` 与 `elsewhere` 一律算作非零，没有讨论跳板 activity 造成的 3/3 误报（F4）。
- `external-final-integration.md`：
  - 闭环7「settle 保过程/截止，observedChange 比端点；A→B→A 有真实不同含义」：A→B→A 在 `SettledSamples` 中已经是 unsettled，settle 的 changed/unchanged 仍与端点比较重复。我部分不同意（F7）。
  - 闭环3 / u25-pending：交付方推荐 B「续等 scopeUnknown」。我推荐删掉特例、复用默认循环（F6/R4）。
  - 屏5：ui.read 仍带必填 kind，属于冗余但无害。
- 上次外评 `upload/androidctl-candidate-review-final.md`：
  - 同意「设备侧必填种类便宜且正确」和「只约束种类不绑实例足够」。
  - 裁决-3 建议「成功后写 virtual + open 写 main + 删 select」：我不同意「成功后写」（F1）和「open 写 main」（F3）。
  - 屏-1 指出的「工作区偏好的洞」已由 U25 改为主机级而消除，但主机级引入了 F2 的遗留问题。上次外评与交付方都没有讨论这一点。
- `candidate-final.md` 称三项可选改进「保持现合同不阻塞」：同意它们不阻塞。但 F1、F4、F5 不属于那三项，是独立的小缺陷，建议在合入前修掉。

---

## 6. 未覆盖范围与必要补证

- 没有逐行审 `ActivityLaunch` 的 SEND/ClipData、`VisualInput` 坐标在 VD 上的完整路径、`TreeCollector` 预算细节、`UiActions` 中 covered/scope 准入（属于其它专题）。
- 没有复核 stage 15/16/17 各运行原件的字节内容，只读了它们的 result.md 摘要和外评引用；`wait-absent-once` 的具体输出未亲读。
- 没有核实上游 AOSP 中「null Surface 的 VD 能否 ON / 能否用 captureDisplay」，因此不主张删掉 ImageReader drain（交付方有两帧探针支持保留）。
- 会改变判断的补证（只列这些）：
  1. 用户真机上的后台屏探针（见 R2），决定 38 是合入还是拆出。
  2. 在 API34 AVD 上对 1–2 个带启动页的 App 做 virtual open，统计 `landing` 分布：若 `unknown` 普遍，F4 必须改；若多数是 `observed`，可以降级为文档问题。
  3. 一次真实 agent 在后台屏上完成任务的对照（目前成对任务只覆盖主屏）：观察 agent 是否误读 `displayRequired`、是否因 `exit 1` 中断。这能直接检验 F2/F4 的实际代价。
