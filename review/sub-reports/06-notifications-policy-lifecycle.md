# 06 通知、管理保护/权限边界、控制者通知、资源生命周期

评审员：子评审 06。范围是 CURRENT_SCOPE 方向 2、30–32、33，“必须合起来审的接点”第 5、6 条，以及所有新增长寿命资源的横向生命周期。
源码行号取自候选 `3f45af6`（REPO）。AOSP 引用来自本人在线下载的 `android-14.0.0_r1`（另有 14/15/16 的 `ActivityOptions`/`ComponentOptions`），文件保存在 `scratchpad/agents/aosp06/`。我没有设备，没有重跑任何东西。

证据类别标注：【实测】=包内原件；【源码】=候选源码推断；【AOSP】=上游源码核实；【设计】=设计建议。

---

## 1. 结论摘要

1. **通知 D 的底层路线可靠，但写路径的“地址/守卫/收尾记账”层做过头，而且有一个会实际卡住可用性的守卫。**
   - 路线：shell UID 带 `STATUS_BAR_SERVICE`，用 `INotificationManager.registerListener` 注册临时系统 listener，再用 `getActiveNotificationsFromListener`、`cancelNotificationsFromListener` 和 `PendingIntent.sendAndReturnResult` 完成读写。每一步我都在 AOSP 14 上核过，没有注入 system_server。【AOSP】
   - 问题：写操作要求回传 6 个字段（key、postTime、action、title、creator、creatorUid）。NMS 每次 enqueue 都把 postTime 重置为 `currentTimeMillis()`（`NMS.java:6704-6706`）。进度通知、媒体通知、有新消息的聊天通知，以及 AndroidCtl 自己每 2 秒刷新一次的 Gateway 通知，在 get 到 action 这段跨越 LLM 思考的时间里，很可能已经变了版本，结果是反复 `notificationChanged`。
   - 现有证据只覆盖 Termux 自造的静态通知，没有覆盖这种情况。【AOSP+源码，证据缺口】
2. **通知写操作建议“保留能力、裁剪表面”（下文 C 案）。** 具体做法：
   - 地址缩到 key + action 索引（title 作可选核对），postTime 改为可选，删除 creator/creatorUid 守卫。
   - ongoing 通知在 dismiss 前直接拒绝。
   - 删除 open 永远为 unknown 的 landing 任务列表。
   - 把 cleanup 三态从每个回执里拿掉。
   - 补读 MessagingStyle 会话正文：没有它，reply 的使用场景基本读不到要回复的内容。
   - 只读 list/get 的价值最确定，应保留。
   - 如果用户想把维护面压到最小，B 案（只读 + 用现有通知栏节点操作）是诚实的替代，损失也可以接受。
3. **【确定缺陷，可达性极低但后果重】通知 listener 注销失败时封锁过宽，还可能诱发重复外发。**
   - 后果：保留的 UI resource 会挡住所有 UI、新 shell/file、租约到期和正常 Stop（`UiBackend.kt:139`、`Sessions.kt:144,198-201,232-235,241-244`）。
   - 同时 send 已经 `done` 的回执会被改成非零 exit（`NotificationOperations.kt:185-195`），对 reply 而言等于诱导 agent 重发一条真实消息。
   - 但 AOSP 的 `unregisterListener` 不查权限，未命中也不抛（`NMS.java:4544-4546`、`ManagedServices.java:1084-1088,1670-1684`）；listener 还 `linkToDeath`，worker 一死就自动移除（`MS.java:1706-1712,1846-1853`）。
   - 也就是说，整套“保留 access / 保留 busy / cleanup 三态 / exit 覆写 / 最坏预留”防的是一个几乎到不了、即使到了也会随 worker 死亡自愈的状态。建议整体删除，只把诊断写进 status。
4. **【确定缺陷，低】dismiss 遇到 `FLAG_ONGOING_EVENT` 会被平台静默忽略（`NMS.java:4697-4704`），产品却要等满超时，才报 UNKNOWN 并停批**（`NotificationActions.kt:387-425`）。这个 flag 在派发前就能读到。上次外评已经提出过，内部以“未授权”为由搁置；我认为它是几行代码的缺陷修复，不是新范围。
5. **FLAG_SECURE 和自身管理 UI 保护（方向 2）应当保留。它在 UI 通道内基本自洽，但本质是防误操作和防屏幕诱导，不是安全隔离。**
   - 公开文档大体诚实（`skills/androidctl/observations.md:173-177` 写明 shell/file 权限不变），但 README 的保护段（`README.md:115-118`）没有这一句。
   - 有一个新的通道缺口：`notifications open` 可以打开 AndroidCtl 自己的 Gateway 通知 contentIntent，从而启动 MainActivity。typed open 对此有专门拒绝（`ActivityLaunch.kt:187-188`），通知写路径没有。
6. **控制者通知（方向 33）不构成第二状态权威**：它与管理页读同一份缓存的 `WorkerStatus`，唯一的 2 秒轮询只是从 MainActivity 搬到了 Service。建议保留，修两处：
   - 每 2 秒无条件 `notify()`（`GatewayService.kt:104-115`），应在文本变化时才发。
   - 轮询失败写入的 `error` 不会被之后的成功轮询清掉（`GatewayService.kt:280-308`）。阶段 33 自己的更正页已承认这一点。
7. **资源生命周期总体所有权清楚**：单一 UI executor 串行化了图源、VD、事件和 listener 的释放，没找到双重释放或泄漏。具体问题有三个：
   - “清理失败 → 永久 closing/busy”这一模式被复用到三个来源。
   - `TargetManagement` 在 VD 清理失败时，用清理错误覆盖了原失败原因（`TargetManagement.kt:84-93`）。
   - Gateway OFF 之后，租约和后台屏可以继续存活到 idle 上限 24 小时（`TargetManagement.kt:26,117`），而唯一可见的指示（通知）已经消失。
8. **把整套权限边界放在一起看，没有发现被夸大成安全保证的表述，也没有发现新增的实质安全面。**
   - 通知读取在隐私上不是新能力：shell 本来就能 `cmd notification get`，且 redact=false。新的只是让它变得顺手，并且锁屏时也能读。
   - reply/action 才是新的外发能力，但它不比“解锁后在 App 里打字发送”多出风险类别。
   - 文档应该直白写一句：reply 会真实发出消息。

---

## 2. 负责方向处置表

| 方向 | 处置 | 依据 | 真实能力损失（若按建议） | 维护代价变化 |
| --- | --- | --- | --- | --- |
| 2 管理窗 FLAG_SECURE + 自身 UI 来源保护 | **保留，改表述，补一个缺口** | 防第三方录屏/截图泄露配对码，也防 agent 经 UI 通道被诱导开码、读码、撤销、停机（研究已在基线实测远程开码读码）。UI 通道内覆盖基本一致。 | 无 | 加 1 行（通知写拒绝 creatorPackage==自身），README 加 1 句 |
| 30 通知 list/get | **保留，调整** | 结构化读取有实测增量（API34 release 实测 list/get 正常），不依赖树、VD 或解锁 | 无；建议补 MessagingStyle | 删最坏预留，改固定余量；删 cleanup 三态 |
| 31 通知 action/open/dismiss | **调整（C 案）** | 机制在 Termux/Clock 上成立；postTime 和 creator 守卫对频繁更新的通知会持续误拒；ongoing 等满超时；landing 永远 unknown | 失去“PI creator 变化”这条核对（研究 E14 已证明它不能绑定业务对象） | 净删除约 150–250 行产品代码和对应契约/文档 |
| 32 通知 reply | **调整，与 31 同步；证据不足部分标明** | 只有 Termux 本地回复的证据；没有 MessagingStyle 正文就看不到对话内容 | 无 | 加一个有界 messages 投影 |
| 33 控制者通知 | **保留，修两处** | 同源缓存，不是第二权威；成本是后台 2 秒一次 Binder 加 notify | 无 | 加文本变化比较，轮询成功时清除轮询自己造成的错误 |
| 接点 5（图源/编码/Canvas/大树共存） | **保留** | 单 executor 串行，所有权唯一；全分辨率 ARGB 源在 TTL 过期后仍常驻到下次失效（设计如此，为 zoom 服务） | — | — |
| 接点 6（full + 自身 UI + main 解锁 + 通知设备级 + 剪贴板准入） | **保留，文档统一一句边界说明** | 见 §3.5 | — | — |
| 横向：清理失败永久 busy 模式 | **调整** | 通知 listener 这个来源应移出该模式；UiAutomation/VD 来源可保留，但应只封住 UI | — | 删除 `NotificationOperations` 的保留状态 |

---

## 3. 发现（按重要性）

### 3.1 通知写操作的 6 字段地址与 postTime 守卫【设计取舍 → 可用性缺陷】【AOSP+源码；证据缺口】

**位置**：
- `NotificationActions.kt:12-32`：validate 要求 postTime；action/reply/open 还要求 creator 和 creatorUid。
- `NotificationActions.kt:238-269`：perform 依次核对 key、userId、postTime、action 索引、title、creator 和 creatorUid。
- `androidctl/src/androidctl/ui_commands.py`：notification_action、reply 和 open 的必填参数。

**平台事实**：
- AOSP 14 `NotificationManagerService.java:6704-6706`：`new StatusBarNotification(..., System.currentTimeMillis())`。每次 `notify()`（包括内容更新）都会产生新的 postTime。
- 研究报告 §3 自己承认：key、postTime 和动作描述的核对只证明“本次选择字段一致”，不能证明业务对象原子一致。E14 的实测是：选 a，共享 token 却把回复交给了 b。

**影响**：
- LLM agent 的典型节奏是 get → 思考（数秒到数十秒）→ action/reply。以下几类通知在这段时间里大概率已被更新：
  - 下载/上传进度、媒体播放进度和曲目、导航。
  - 聊天会话通知（同一会话来了新消息）。
  - 带秒表/倒计时更新的 App。
  - AndroidCtl 自己的 Gateway 通知：每 2 秒 notify，见 §3.6。阶段 33 原件里 `mUpdateTimeMs` 每约 2 秒变一次。
- 这时 postTime 守卫会稳定地拒绝，agent 只能 get 后立刻 action，再一次赌时间窗。batch 是静态的，无法把 get 的输出串进同一批的 action，所以帮不上忙。
- 包内所有写路径证据（阶段 30/31 的 Termux 和研究中的 Clock）都是不再更新的静态通知，完全没有覆盖这种情形。
- creator/creatorUid 守卫没有独立价值：PI 的 creator 几乎只在“换了一个 App 的 PI”时才变，而 key 已经包含了 package；E14 又证明它不能防错投。

**建议（C 案的核心）**：
- 必填地址只留 key + action 索引；title 作为可选核对，用来防动作数组重排。
- postTime 改成可选的严格模式。默认不核，回执里报告实际作用对象的 postTime。
- 删除 creator/creatorUid 参数，并删除对应的 host 参数、契约样例和文档段落。
- 这样会损失“自 get 之后通知已更新”的拒绝。对聊天 reply 而言这是取舍：默认不拒，回执给出当前 postTime 和是否变化即可。

**会改变判断的补证**：在真实聊天 App（模拟器上的 Google Messages + `adb emu sms send` 即可）和一个进度通知上，各跑一次 get → 等 10 秒 → reply/action。
- 若拒绝率高：C 案必要。
- 若真实 App 很少更新：postTime 可以保留为默认，只删 creator 守卫。

### 3.2 通知 listener 注销失败的“保留责任”机制【确定缺陷/过度防御】【AOSP+源码】

**触发路径**：
- `NotificationOperations.kt:147-161`：unregister 抛错时，保留 `access`，设置 `cleanupFailure`，回执写 `cleanup:"unconfirmed"`。
- `NotificationOperations.kt:185-195`：只要 cleanup unconfirmed，就把 exit 0 改成 1。
- `UiBackend.kt:139`：`cleanupPending` 时不调用 `sessions.finished(resource)`，UI resource 永久留在 `Sessions.resources`。
- `UiBackend.kt:46` 加 `Sessions.kt:144`：之后所有 UI 请求都被拒，返回 `uiCleanup … restart the worker`。
- `Sessions.kt:198-201`：resources 里有 UI 时，新的 shell/file 一律 busy。
- `Sessions.kt:241-244`：resources 非空时，租约永不过期，其他配对一直拿到 busy。
- `Sessions.kt:232-235` 加 `WorkerService.kt:115-123`：本机“Stop idle executor”失败，只能用 Force（`WorkerConnection.kt:238-247`）。
- 远端 agent 没有任何自救手段，提示里的“restart the worker”它做不到。

**AOSP 可达性**：
- `unregisterListener` 不做权限检查（`NMS.java:4544-4546`），`ManagedServices.unregisterService` 的注释写明 “no need to check permissions”（`MS.java:1084-1088`）。
- `removeServiceImpl` 找不到时返回 null，不会抛（`MS.java:1670-1684`）。
- 剩下的失败来源只有 Binder 传输错误。那意味着 system_server 已经死亡，此时整机运行时重启，worker 也随之结束。
- 即使 listener 真的残留，`registerServiceImpl` 已经 `linkToDeath`（`MS.java:1706-1712`），`binderDied` 会自动移除（`MS.java:1846-1853`）。残留代价只是 NMS 向一个空 Binder 发 oneway 回调。
- 结论：这套机制防的是一个几乎不可达、会自愈、残留代价很低的状态。它的代价却很实在：
  - 每个通知回执都多一个 `cleanup` 字段；
  - 每个回执的预算里都要预留 `cleanupError`（`NotificationProjection.kt:301-305`）；
  - exit 被覆写；
  - architecture.md 里有专门段落；
  - skill 文档里有一整段说明（`skills/androidctl/notifications.md` “data.cleanup …” 段）。

**更严重的组合**：reply/action 已经 `done`（平台已发送）时，如果 unregister 抛错，exit 会被改成 1。agent 和 batch 都按非零处理，agent 很可能重发，结果就是**真实消息被发两次**。这比 listener 残留严重得多。

**建议**：
- 在 finally 里尝试 unregister。失败时只写 worker status 的诊断（例如 `diagnostics.notificationCleanup`），不保留 resource，不改 exit，不进回执。
- 删除 `NotificationAccess.cleanupPending` 状态机、`NotificationOperations.access` 字段和 cleanup 三态。
- 估计可删产品代码约 60–80 行，外加相应的测试、契约和文档。
- 没有真实损失：泄漏会在 worker 结束时自愈，而且几乎不会发生。

### 3.3 dismiss 对 ongoing 通知等满超时【确定缺陷，低】【AOSP+源码】

- 平台：`NMS.java:4697-4704` 的 `cancelNotificationFromListenerLocked` 带 `mustNotHaveFlags = FLAG_ONGOING_EVENT`，cancel 会静默无效。
- 产品：`NotificationActions.kt:387-425` 没有预检。cancel 返回后循环 read，直到 deadline，最后以 UNKNOWN 结束并停批。
- `flags` 在 `notifications get` 的 identity 里已经输出（`NotificationOperations.kt:224`），写路径的 `NativeNotificationCandidate.identity()`（`NotificationActions.kt:151-161`）却没有带上。
- 建议：派发前遇到 `FLAG_ONGOING_EVENT` 直接 not sent，code 写 `notificationOngoing`。约 5 行。
- 内部处置（`external-final-integration.md` “通知1”）以“新增预拒未在 U25 授权”为由不改。我不同意：这是平台确定不会执行的调用，修正它不扩大任何能力。

### 3.4 通知投影的预算与回执机械【过度工程】【源码】

**现状**：
- `NotificationProjection.kt:290-345` 的 `notificationEnvelopeBytes` 为每种命令拼出一个最坏情况回执，包括：
  - 256 个 `\u0000`（JSON 转义后每个 6 字节）的 reason/code/message/cleanupError/operationError；
  - 所有计时字段取 `Long.MAX_VALUE`；
  - open 的 landing 两组错误。
- 这个最坏回执被用于三处：list 预算、选择记账（`NotificationActions.kt:315-322`）和派发前检查（`NotificationActions.kt:427-431`）。
- `NotificationObject.create` 还要再为每个字段预留最坏的 omission 记录（`NotificationProjection.kt:244-268`）。
- 这与 `ClipboardBytes` 是同一模式写了两遍（上次外评合并-3）。

**判断**：192KiB 的回复预算下，通知 identity 都是几百字节量级。用固定余量（例如 8KiB）加“对象级整体省略”就够了。逐字段 omission（字段名 + UTF-16 长度 + from/count）对 LLM 的决策没有可见用途：它只需要知道“这条被截了，去 get”。

**建议**：
- 固定余量，删掉最坏回执构造。
- list 只做对象级省略；get 只截断长正文。
- 删掉选择阶段反复的 `rememberSelection` 预算检查：选择对象是有界的几百字节。

### 3.5 open 的 landing 前后任务列表【过度/无消费者】【源码】

- `NotificationActions.kt:335-346,433-467`：open 前后各读一次 `getAllRootTaskInfos`，最多各 8 条，外加 count、omitted 和 error。但 `landing.state` 恒为 `"unknown"`（`:448`），reason 也是固定文案。
- 对 agent 来说，它的下一步本来就是 snapshot，这份数据不会改变任何决策。上次外评已经提出（通知-5 第 3 条）；内部处置是“保有界 open 落点事实”。
- 建议删除，回执只留平台 code/codeName。预算预留（`NotificationProjection.kt:338-344`）也随之删掉。

### 3.6 MessagingStyle 缺失使 reply 价值大打折扣【能力缺口】【源码】

- 详情字段只有 title、text、bigText、subText、textLines 和 remoteInputHistory（`NotificationProjection.kt:86-98`），没有 `EXTRA_MESSAGES` 和 `EXTRA_CONVERSATION_TITLE`。
- 主流聊天 App 的会话通知都用 MessagingStyle，正文在 messages 里。EXTRA_TEXT 往往只有最后一条，或者是“N 条新消息”。
- 结果是 agent 用结构化读取看不到它要回复的内容，往往还得 open App 去读；那样 reply 的效率优势基本就没了。
- 内部处置（“通知4”）认为这是新正文范围。我的判断：如果保留 reply，补 messages 比保留 creator 守卫和 landing 更值得。复用 3.4 简化后的截断即可，代价是一个字段。

### 3.7 自身管理 UI 保护：一致性、缺口与表述【设计取舍 + 低缺陷】【源码+实测】

**防的是什么**：研究（`research/policy/report.md` §2）在基线实测过，经 UI 通道可以远程点开配对码，并在树中读到 6 位有效码。威胁有两类：
- (a) 被屏幕文字诱导或误操作的已授权 agent，经 UI 通道开码、读码、撤销配对或停止 executor；
- (b) 第三方录屏或投屏泄露配对码，以及 AndroidCtl 自己的截图把配对码送到云端 LLM。

FLAG_SECURE（`MainActivity.kt:95`；配对对话框显式 `SecureFlagPolicy.SecureOn`，见 `ManagementScreen.kt:325`）解决 (b) 的像素部分。【实测】`final-release-runtime-api34.md:20` 记录了管理页截图主体为黑。

**UI 通道内的覆盖**（全部按实际来源或接收者判定）：

| 通道 | 位置 | 判定依据 |
| --- | --- | --- |
| 树 | `TreeCollector.kt:240` | 根包名 |
| fresh 节点 / 节点动作 | `FreshNodes.kt:22` → `UiObservation.kt:81-87` | 根包名 |
| 近期事件正文 | `UiSignals.kt:162-168`，加 `protectedWindows`（`UiSnapshot.kt:58`、`UiObservation.kt:65-79`） | 窗口或包名 |
| 触摸（节点和坐标） | `AppInput.kt:78-87` + `injectInputEventToTarget` | 平台按 UID 强制 |
| 无 REF 按键 | `SystemActions.kt:201-214` | 派发前查一次焦点窗，有时间窗（上次外评已指出） |
| typed open | `ActivityLaunch.kt:187-188` | 解析出的 handler 包名 |
| IC | `BrowserAccessibilityService.kt:135` | EditorInfo 包名 |

结论：大体集中在 `requireExternalRoot` 和按 UID 注入两处，另有 TreeCollector、UiSignals 和 ActivityLaunch 三处各写一份判断。可以收成一个 `sourceState(pkg)`，但收益不大，不是必须。

**缺口**（均为防误操作意义上的缺口，不是安全漏洞）：
- 【确定缺陷，低】`notifications open` 和 `action` 不检查 PI 的 creator 是否为自身。Gateway 通知的 contentIntent 就是 MainActivity（`GatewayService.kt:39-41,100`），release 实测的 list/get 也确实返回了这个 PI 的元数据。`NotificationActions.perform`（`:238-269`）没有任何自身包判断，可以绕过 `ActivityLaunch` 的 `protected` 拒绝，把管理页拉到前台。
  - 之后读树和坐标点击仍被挡住，所以实际危害小。讽刺的是，因为 Gateway 通知每 2 秒刷新 postTime，这条路径通常会被 postTime 守卫“意外”挡住。一旦按 3.1 放宽 postTime，就必须补上这条判断。修法是 1 行：`creatorPackage == ownPackage` 时拒绝。
- shell：已配对主机可以 `am start` 管理页、`input tap` 盲点。在 worker 当前租约还没连 UiAutomation 时（UiAutomation 按租约惰性连接，见 `UiBackend.kt:425-430`，租约结束时断开），也可以 `uiautomator dump` 读到配对码。这在 full 模型下无法也不该修。

**误拒风险**：
- 根包名为 null 时，所有路径都按 `sourceUnknown` 拒绝或不交付（`UiObservation.kt:84`、`TreeCollector.kt:239`、`UiSignals.kt:166`）。
- 少数 SystemUI 或 overlay 窗口可能出现 null 包名。包内没有看到这类误拒样本，【证据缺口，影响小】。

**表述**：
- `observations.md:173-177` 写明“Shizuku's UI is outside … Shell and file commands keep their existing permissions”，这是准确的。
- 但 `README.md:115-118` 和 `images.md:158` 只列出保护措施，没有说明 shell 仍是全权。建议在 README 这段末尾加一句：“这是对 AndroidCtl UI 通道的防误操作护栏；已配对主机的 shell 仍拥有完整 shell 权限。”
- 我没有发现把它称作“安全隔离”或“沙箱”的措辞。

**处置**：保留。整个方向只有 8 行加若干判断，而它防的场景（屏幕诱导 agent 开码、读码）在基线有实测，性价比高。

### 3.8 控制者通知（方向 33）【设计取舍 + 两处小缺陷】【源码+实测】

**不是第二权威**：
- `foregroundNotification()`（`GatewayService.kt:74-102`）读取 `worker.readAttachedWorker().status`，与 `publishManagementStatus`（`:315-342`）是同一个缓存对象。
- 唯一的刷新来源是 `observeWorker`（`:32-38`）：它在 management 单线程上排队，完成后再排下一次，不会堆积。
- 原先 MainActivity 里的 `pollWorkerStatus` 被删除（bba1416），所以轮询来源仍然只有一个。

**成本**：
- Gateway 开着期间，每 2 秒一次 `status()` Binder 调用，外加一次 `notify()`。
- `Handler.postDelayed` 基于 uptime，不会唤醒深睡，电量影响有限。

**缺陷 A（低）**：每次 `publishStatus` 都无条件 `notify()`（`:104-115`），即使文本没变。这会让 NMS 每 2 秒处理一次更新，并刷新 postTime（阶段 33 原件里 `mUpdateTimeMs` 每约 2 秒变化）。修法：缓存上次的文本，变化时才 notify。

**缺陷 B（低）**：
- 轮询走的是 `enqueueManagement`，不是 `enqueueManagementCommand`（`:280-284` 对比 `:287-308`）。轮询失败会写入 `error`，之后成功的轮询不会清掉它；错误一直挂在管理页上，直到用户做一次显式命令。
- 阶段 33 的 `runtime-evidence-correction.md` 已承认这是源码事实，只是没有实测时序。
- 修法：轮询成功时，清除由轮询自身写入的那类错误。或者干脆让轮询失败只表现为 status=null（`refreshStatus` 本来就会这么设，见 `WorkerConnection.kt:212-214`），不写 `error`。

**表述与边界**：README 已写明“Gateway 关闭会移除通知；worker、租约和作业可能继续”。准确。但见 §3.9 的生命周期后果。

**处置**：保留。它只增加约 100 行，就让用户在后台也能看到“谁在控制”，是对 full 权限模型真正有用的透明度措施。上次外评建议在无租约时也显示 shell helper，我同意，成本是一行。

### 3.9 资源生命周期横向审查【源码；部分 AOSP】

| 资源 | Owner | 获取 | 释放触发 | 失败/强杀/worker 死亡/Gateway OFF | 发现 |
| --- | --- | --- | --- | --- | --- |
| UiAutomation 连接 + 回调 HandlerThread + 事件 ring | `UiBackend`（单 UI executor） | 租约内首个需要平台的 UI 请求，惰性连接（`UiBackend.kt:425-458`） | 租约失效 → `clear()`（`:477-500`）→ `disconnect()`（`:502-519`）；signals/ring 在 `platform.signals.close()` 里一并关闭 | 连接失败后清理再失败：`sessions.closing(run.lease)` 永久占用并写 `closeError`（`:466-473`）。清理失败：`closing` 不释放，`recordCleanupFailure`。worker 死亡由平台回收。Gateway OFF 时持续到租约 idle 到期 | 失败后会挡住 shell/file 和租约到期（与 3.2 同一模式）。这个来源的失败比通知 listener 更可能真实发生，保留 fail-closed 可以理解，但应只封 UI，不封 shell/file |
| 普通 AccessibilityService（Browser support，含 IME 能力）+ EditorBridge | App 进程 / 系统绑定；EditorBridge 由 `WorkerConnection` 创建 | 用户在设置里启用（`AndroidManifest.xml` 新增）；EditorBridge 在 worker 附着时下发（`WorkerConnection.kt:82-96`） | 用户关闭；EditorBridge 在 worker 死亡、detach 或重绑时 `invalidate`（`:83-87,132,166,226,286`） | 与 Gateway 无关：服务在启用期间常驻，订阅 6 类事件但 `onAccessibilityEvent` 为空（`BrowserAccessibilityService.kt:159`、`xml/browser_accessibility_service.xml`）。这是为了让 Chrome 开启渲染无障碍，代价是全机的事件 IPC，以及 Chrome 持续构建无障碍树 | 无泄漏。产品边界上应在 Browser support 页面告诉用户“不用时可关闭以免性能开销”（属方向 11–13 评审员的范围，这里只点出生命周期） |
| 通知 listener | `NotificationOperations`（每次调用） | `register()`（`NotificationAccess.kt:19-22`） | 同一调用的 finally `unregister()`（`NotificationOperations.kt:147-161`） | NMS `linkToDeath`，worker 死亡时自动移除（`MS.java:1706-1712,1846-1853`）。注销失败路径见 3.2 | 过度保留，并有 exit 覆写诱发重发的风险 |
| VirtualDisplay + ImageReader + 消费线程 | `WorkDisplay`，挂在 `Lease.target.stream` 上 | `display.create`（`TargetManagement.kt:100-105`、`WorkDisplay.kt:33-78`） | 未发布：`changeTarget` 的 finally（`TargetManagement.kt:82-97`）。已发布：租约失效时 `clear()` 调 `stream.close()`（`UiBackend.kt:492`）。`close()` 幂等，按“先停 listener、释放 VD、join 消费线程 2s，再关 reader”的顺序执行（`WorkDisplay.kt:113-144`） | worker 死亡时平台回收 VD，按 `DESTROY_CONTENT_ON_REMOVAL` 销毁其上 task（包括被搬进去的用户 task，见上次外评资源-2）。Gateway OFF 时 VD 存活到租约 idle 到期，而 create 的 idle 上限为 86400s（`TargetManagement.kt:26,117`） | (a) 【确定缺陷，低】未发布清理失败时，`run.remember(failure)` 用清理错误覆盖原失败结果（`TargetManagement.kt:84-93`）：比如 create 因 `displayUnsupported` 失败、VD 清理又失败，agent 只看到清理错误。建议把清理错误放进原结果的 `cleanupError`。(b) 【设计】Gateway OFF 后，后台屏连同上面的 App 可能隐形存活长达 24 小时 |
| 原生图源 Bitmap（软件 ARGB 全分辨率） | `VisualInput.image`（每个 worker 一帧，按租约核验） | screenshot（`VisualInput.kt:59-121`） | 新截图、派发（`UiBackend.dispatch` → `visuals.invalidate()`）、观察签名变化、`clear()`；全部在 UI executor 上执行，不与 zoom 竞争 | worker 死亡时随进程释放 | 120s TTL 只限制“能否用于坐标”，不释放内存；源 Bitmap 可常驻到租约结束（按 1080×2400 约 10MB）。这是为 zoom 的设计取舍，可以接受 |
| 截图 HardwareBuffer / 硬件 Bitmap | 局部作用域 | `softwareSource`、`WorkDisplay.screenshot` | `use {}` 加 finally `recycle`（`VisualInput.kt:123-132`、`WorkDisplay.kt:100-110`） | — | 正确 |
| 派生编码 Bitmap / 标注 Canvas 副本 | `encode()` 和 `screenshot()` 局部 | scale/copy | finally `recycle`（`VisualInput.kt:115-116,199-202`） | — | 正确，没找到双重释放（`takeIf { it !== bitmap }` 已防止） |
| 剪贴板借用 | `BorrowedPaste`（单次命令） | 借写 | 同命令 finally 中按条件恢复、清除或跳过（`BorrowedPaste.kt:241-291`） | worker 强杀时没有持久恢复（已在文档说明） | 不在本专题深审 |
| 租约 / 会话 | `Sessions` | session open、隐式 acquire | idle 到期（worker 端每秒维护一次，与 Gateway 无关）、close、release、revoke | resources 非空时不到期 | 见 3.2 和 UiAutomation 行 |
| 控制者轮询 | `GatewayService.statusHandler` | onCreate | `beginStopping` 中 `removeCallbacks`（`GatewayService.kt:358`）；Runnable 自带 `current/stopping` 门 | — | 正确 |
| host daemon / history / diff base / 锁 | host | `LifetimeLock`（fcntl/msvcrt）、history 配额 150 文件 / 20MB（`history.py:20-21,74-102`） | 进程退出时 OS 释放锁 | — | 只粗读，未发现问题；diff base 的生命周期属方向 8 评审员 |

没有找到双重释放；所有 `close`/`invalidate` 都是幂等的。“closing 永久占用”是有意设计的 fail-closed（`docs/architecture.md:135`），但它被三个来源共用，封锁的面也过宽，问题在于这两点而不是机制本身。

### 3.10 PendingIntent / BAL 的可靠性与升级风险【AOSP】

- **sender 选项**（`NotificationActions.kt:211-231`）：
  - API 34/35 用 `MODE_BACKGROUND_ACTIVITY_START_ALLOWED`，加上 @hide 的 `ComponentOptions.setPendingIntentBackgroundActivityLaunchAllowedByPermission`。我核过：该方法在 14 和 15 都是 @hide，16 中已不存在。
  - API 36 改用公开的 `MODE_BACKGROUND_ACTIVITY_START_ALLOW_ALWAYS = 3`（AOSP 16 `ActivityOptions.java:156`）。
  - Shell 持有 `START_ACTIVITIES_FROM_BACKGROUND`（`Shell/AndroidManifest.xml:260`）。
  - 代码按字段是否存在来分支，不按 SDK 版本，这是合理的。【实测】API34 走旧分支、API36/37 走 mode 3，都返回了 START_TASK_TO_FRONT。
- **升级风险**：
  - 如果未来再收紧“sender 凭权限豁免”，或者删掉 @hide 方法，反射会在 `prepareSend` 里失败，结果是 NOT_SENT。失败方式是安全的。
  - broadcast/service 类 PI 不受 BAL 影响，reply 一般走 broadcast，因此 BAL 风险主要落在 open 上。
- **平台语义**：`sendAndReturnResult` 只代表派发返回，回执也是这么说的（“Platform dispatch returned; App outcome is not verified”），没有夸大。

---

## 4. 建议重新裁决的事项

### 裁决 N-1：通知 D 的写路径形态（用户 U19 已选 D）

| 选项 | 内容 | 优点 | 缺点 |
| --- | --- | --- | --- |
| A 维持现 D | 现状：6 字段地址、cleanup 三态、landing、最坏预留 | 已实现，有 Termux/Clock 证据 | 频繁更新的通知上持续误拒（3.1）；ongoing 等满超时；回执和文档冗长；看不到会话正文 |
| B 只读 | 保留 list/get（加 messages），删除 action/open/dismiss/reply；写操作走现有 `notifications` 加节点操作 | 删除产品代码约 600 行，外加约 1,300 行契约和测试；没有对外发送的新路径 | 失去锁屏结构化操作（写本来就要求解锁，所以实际损失小）；通知栏节点操作对折叠或分组通知较脆，需要多轮 |
| **C 保留能力，裁剪表面（推荐）** | 地址为 key + index（title 可选，postTime 可选严格）；删除 creator 守卫、landing、cleanup 三态和最坏预留；ongoing 预拒；补 messages；creator 为自身时拒绝 | 保留 D 的能力；修掉真实可用性问题；净删代码 | 需要重写约 1/3 的通知写代码和契约样例；放宽 postTime 后，默认不再拒绝“通知已更新” |
| D 去掉 reply | C 的基础上不提供 reply | 移除唯一的“对外发文本”新路径 | reply 恰是通知写操作里对 agent 最有效率的一项；风险类别与 UI 打字发送相同，删除理由不足 |

推荐 C。剩余未知：真实聊天 App 的 reply 是否被接受，以及 postTime 更新频率。补证见 §6。若补证表明主流 App reply 不可用，降级为 D 或 B。

### 裁决 N-2：清理失败的封锁策略（跨方向，基线机制被候选扩展）

- (a) 维持现状：任一来源失败，就封住全部 UI、shell 和 file，并阻止到期。
- **(b) 通知 listener 移出该机制（只记诊断）；UiAutomation/VD 失败只封 UI 和 display，不封 shell/file。推荐。**
- (c) 全部来源只记诊断、不封锁。不推荐：UiAutomation 清理失败可能导致下一次连接冲突，保留 fail-closed 有道理。

---

## 5. 对既有内部 review 与交付方声明的对照

- `reviews/final-android-contracts.md:46` 称“unregister 失败保主事实、cleanupError、非零 exit 与 busy，不悄悄释放资源”，并视为正确。我不同意这是优点：
  - 它把一个 AOSP 上几乎不可达、且由 `linkToDeath` 自愈的状态，升格成了全 worker 封锁；
  - 还让已发送的 reply 回执变成非零 exit，可能诱发重复外发。
  - 该 review 没有对照 AOSP 的 `unregisterService` 和 `binderDied`。
- `reviews/final-android-contracts.md:51` 和 `final-maintainability.md:41` 称“没有发现需要删除的无消费者 owner/状态”，并说“各通道有界 JSON 投影……不建议提取通用框架”。
  - 前半句不成立：`NotificationOperations.access` 在失败后被保留，但没有任何代码会再用它（没有重试）。它只充当一个“保留 busy”的标志；open landing 也没有决策消费者。
  - 后半句（不必做通用 JSON 框架）我同意，但应该用固定余量替代最坏回执构造，而不是把最坏回执抽成通用框架。
- `external-final-integration.md` 的“通知1”以“未授权”为由不修 ongoing 预拒，“通知5”保留 creator、title 和 postTime 守卫以及 landing。
  - 我认为前者是缺陷修复，不需要新授权。
  - 后者没有回应“postTime 每次 enqueue 刷新”这一 AOSP 事实对可用性的影响；上次外评也只说守卫“可能冗余”，没有指出它会误拒。
- 上次外评（`upload/androidctl-candidate-review-final.md`）的资源-1、保护-1、保护-2、通知-1、通知-4 和合并-3、合并-7，我独立复核后基本同意，补充如下：
  - (1) 资源-1 的可达性已用 AOSP 坐实，并新增“已发送回执 exit 被覆写可诱发重发”这一更重的后果；
  - (2) 保护的新缺口：通知 open 会走到自身 MainActivity；
  - (3) 控制者通知的无条件 notify 和粘滞 error；
  - (4) postTime 误拒。
- 阶段 30/31 runtime review 的“无 must-fix/optional”，限定在它们自己的证明范围内（Termux 静态通知）是成立的；但不能外推到真实 App。阶段 31 自己也写了“不扩展到其它 ROM、新 BAL 分支或大帧全链”。

---

## 6. 未覆盖范围与必要补证

**未覆盖**：
- 剪贴板借用的细节（只核了生命周期入口）；
- host 端 diff base 的生命周期；
- `UiRecentEvents` 投影的内部预算；
- `NetworkGateway` 和配对协议本体（候选未改动，`git diff 7d5be70 HEAD` 显示 NetworkGateway/Pairings/DeviceIdentity 没有变化，transport.py 改了 4 行）；
- OEM ROM 上 NMS 和 BAL 的差异。

**会改变判断的补证**（各一次即可）：
1. **真实聊天 App 的 get → reply**：模拟器上用 Google Messages，并用 `adb emu sms send` 制造会话。
   - 若 reply 被 App 接受，且回执和后验一致：支持 C。
   - 若主流 App 拒收：降级为 D 或 B。
   - 同时可以观察 MessagingStyle 的 messages 是否就是 agent 回复所需的正文。
2. **进度类通知的 get → 等 10 秒 → action**，在当前候选上执行。
   - 稳定得到 `notificationChanged`：坐实 3.1，postTime 必须改为可选。
   - 不会误拒：postTime 可以保留为默认。
3. **（可选）`notifications open` 指向 Gateway 自身通知**：只用来确认 3.7 的缺口，不影响方向判断，可以直接按 1 行修复处理。
