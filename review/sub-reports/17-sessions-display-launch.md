# 专题 17：会话、显示目标、后台屏、启动落点、跨屏已有任务（第三轮复审）

评审对象：候选 `3f45af6`（`review-copy`，只读）、实施方 U26 材料（`T` = `NEW/ultragoal/2026-11-11-next-improve`）、上轮我方主报告 §2.7/§2.8/§5 与专题 13/15。行号均对应 `3f45af6`。证据标签：**原件核对 / 源码核对 / 离线重算 / 推断 / 设计建议 / 需设备**。

---

## 1. 结论摘要

1. **D 的共享种类偏好在"全显式会话"下几乎失去了原有用途。** U23 设计偏好是为了挡"隐式取得 main"（`Sessions.kt:94` `keepLease`）；全显式后这条路径本身已删，D10 由"显式"消除，与偏好无关。剩下偏好比 B 多拦的只有两类序列：①调用者收到 `noSession` 后习惯性执行不带 `--display` 的 `session open` 再发 UI（自伤序列）；②同名另一调用者只做了 `session open`、没做 `display select main`（半步序列，此时后来者自己也会被拒）。同名调用者按 D 的正规两步（select main + open）回 main 时，旧调用者照样跟随。**源码核对 + 推断**
2. 序列①可以用 **`session open --display main|virtual` 必填、无缺省** 零状态消除，而且拦得比 D 早（在 open 处就要求选择，而不是在之后的 UI 请求处 mismatch）；序列②属于同名误用，任何"按名字"的方案都挡不全（D 也挡不住 virtual→另一 virtual，也挡不住正规两步）。
3. D 自己带来四类误拒或额外状态：跨任务遗留 virtual 偏好（报错提示还引导去 `display create`）、create done 但保存失败、create UNKNOWN 但实际已发布（D14）、回 main 需两步。实施方自己的 Contacts 实验也走了 `display select main` + `session open` 两步收尾。**原件核对**
4. **推荐 E′**：设备 lease 是唯一权威；只有 `session open --display main|virtual`（必填）和 takeover 建立 lease；主机不存任何显示状态，请求不带种类字段，删除 `display select/create`；`noSession` 文案带上本 worker 生命周期内该会话最后的目标；回执和 status 首行显示目标。纯设备命令同样要求先 open。可选附加项：batch 级一次性 `expectDisplay` 断言（设备端代码已有），默认不做。把握：中高（约 70%）。
5. 协调方的 F（open 返回 lease 句柄，主机按 device+session 共享保存）：**跟随问题与 D 相同，而且更彻底**（同名者每次 open 都会改写句柄），不值得做；只有调用者自己持有句柄才有实例级保护，但这对 LLM 负担大。
6. 启动落点（题 6）：VLC 原件与实施方描述一致；而且**原 settle 内最后一次观察就已在 VD2 读到 `org.videolan.vlc` 的 application 窗口（focused）**，所以按包确认用现有观察就能在原时点内通过。同意实施方 C（按包和 UID、在目标屏确认、main 与 VD 统一、未确认就停批）；我方上轮"other 时 exit0""读不到就跟平台码"的放宽不成立。isFocused 撤回有原件支持。
7. 跨屏已有任务（题 7）：同意实施方 C（默认保守拒绝，显式选项加 `NEW_TASK|MULTIPLE_TASK`）；我方上轮"默认按启动模式自动另开实例"挡不住 App 内跳板（VLC StartActivity→singleTask VideoPlayer 就是这种结构），原件、源码都是这个结论。建议选项名直接用 `--multiple-task`，带上这个选项时，按解析出的 launchMode/documentLaunchMode 先拒绝必然无效的情形。
8. D12/D13/D14 确认为缺陷；D15 实施方"不成立"正确，我方上轮错；D10 确认，但它由显式生命周期消除，不需要偏好。

---

## 2. 独立判断

### 2.1 会话模型（题 1 + 协调方补充 E、F）

#### 2.1.1 源码事实（源码核对）

- owner = pairing + 会话名（`Sessions.kt:5-13`），设备上只有一个 `current`（`:48`）。lease 已带单调 `id`（`:15-21,94,117`）和 `target`（`:20`）。
- 隐式建立：`acquire`/`acquireUi` 在无 lease 时都走 `keepLease` 新建 MAIN（`:56-64,90-97,136-150`）。shell/files 调 `acquire(owner)`，不带种类（`ShellJobs.kt:33-34`、`FileJobs.kt:24-25`）；通知、剪贴板、apps、wake 走 `UiBackend.execute`，`expected=null`（`UiBackend.kt:105-108`，`RpcMethods.kt:48-56`）。
- 种类检查：`requireTarget`（`Sessions.kt:152-170`）。无 lease 且期望 virtual → `displayRequired`；有 lease 但种类不符 → `displayMismatch`，提示为 "run display create or display select main" 或 "run session open or display select virtual"。
- 切换：`prepareChange` 先 `requireIdle`，**再**让旧 lease 失效（`:116-120`）；建屏成功后才 `publish`（`TargetManagement.kt:49-54`）。所以 busy 或校验在 prepare 前失败时旧 main 还活着；prepare 之后失败则没有 lease。
- 主机：偏好在 `state_directory()/display.json`，以 deviceId→session→kind 索引，跨工作区共享（`workspace.py:23-56`）。只有 `display select` 和 `display create`（done 后）写入（`app.py:780-805,839-863`）；`session open` 走 `device_call`，**不改偏好**（`app.py:337-350,741-760`）。UI 请求每次读文件并注入 `expectedDisplayKind`（`ui_client.py:594-607`），batch 在开始时冻结一次（`ui_batch.py:307-319`）。`uses_display` 只排除通知、剪贴板、apps、wake（`ui_client.py:70-71`）。
- open main 不连接 UiAutomation、不要求亮屏（`TargetManagement.kt:405-407`：MAIN 没有 stream，直接 return）。所以纯设备任务先 open main 几乎没有代价。

#### 2.1.2 各候选的状态、字段与步数

| 候选 | 主机持久状态 | 每请求字段 | 主屏任务 | 后台屏任务 | 纯 shell | 回 main | 保存失败类状态 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A 现状 | display.json + 锁（device+session 共享） | `expectedDisplayKind`（CLI 自动填） | 0 次额外调用（隐式） | `display create` 1 | 0 | `display select main` + `session open` 2 | 有（create done 但保存失败；写已发布但同步失败） |
| B 上轮我方 | 无 | 无 | `session open` 1 | `open --display virtual` 1 | open 1 | `session open` 1 | 无 |
| C 实施方 | 同 A | 同 A | open/main 1 | create 1 | 0（建立无 target 的 owner） | 2 | 有；另加设备"有 owner、无 target"状态 |
| D 实施方推荐 | 同 A | 同 A | open 1 | create 1（要挡 M4 需先 select virtual，共 2） | open 1 | select main + open 2 | 有 |
| E 协调方 | 无 | 可选断言（全局 `--display`，batch 写一次） | 1 | 1 | 1 | 1 | 无 |
| **E′ 我推荐** | 无 | 无（可选的 batch 断言默认不做） | `open --display main` 1 | `open --display virtual` 1 | `open --display main` 1 | `open --display main` 1 | 无 |
| F 实例代际（共享保存） | lease 句柄文件（device+session 共享） | 句柄（自动带） | 1 | 1 | 1 | 1 | 有（open done 但句柄保存失败） |

设备端状态数：A/B/D/E/E′/F 都是 {无, main, virtual}，C 多一个"有 owner、无 target"。**源码核对 + 设计建议**

#### 2.1.3 误操作序列逐条比较（推断，以源码语义推演）

记 X 为当前调用者，Y 为同 host、同 pairing、同会话名的另一调用者。✓ 表示被拦（not sent），✗ 表示落到非预期目标。

| # | 序列 | A | B | C | D | E（未断言/已断言） | E′ | F |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| M1 | 首次 create 失败（prepare 后），X 忽略回执继续发 UI（D10） | ✗ 隐式 main | ✓ noSession | ✓ | ✓ | ✓/✓ | ✓ | ✓ |
| M2 | VD lease 因 expiry/close/worker 重建消失，X 发 UI | ✓（偏好为 virtual 时）；偏好因保存失败或 UNKNOWN 仍为 main 时 ✗ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| M3 | M2 后 X 习惯性 `session open`（不带 --display），再发 UI | ✓ mismatch | ✗ 落 main | ✓ | ✓ mismatch | ✗/✓ | **✓ 在 open 处就因缺参数被拒**，X 必须自己选 | ✗ |
| M4 | X 有 main，create 在 prepare 前失败（绑定失败、busy），旧 main 仍在，X 继续 | ✗（事先 select virtual 才 ✓） | ✗ | ✗ | ✗（同 A） | ✗/✓ | ✗（回执写明"当前仍为 main"） | ✗ |
| M5 | Y 按正规两步回 main（D：select main + open；B/E：open） | ✗ X 跟随 | ✗ | ✗ | **✗ X 跟随** | ✗/✓ | ✗ | ✗（Y 的 open 改写共享句柄） |
| M6 | Y 只 `session open`（main），没 select | ✓（Y 自己也被拒） | ✗ | ✓ | ✓（Y 自己也被拒） | ✗/✓ | ✗ | ✗ |
| M7 | Y 另开一块 virtual | ✗ | ✗ | ✗ | ✗ | ✗/✗ | ✗ | ✗（只有调用者自持句柄才 ✓） |
| M8 | batch 中途 lease 丢失 | 偏好 main 时 ✗ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| N1 | 默认会话名跨任务，上一个任务留下 virtual 偏好，新任务想用 main | 误拒，提示引导 `display create` | 无 | 误拒 | **误拒** | 无 | 无 | 无 |
| N2 | create done 但保存失败 | 误拒，直到 select | 无 | 误拒 | 误拒 | 无 | 无 | 句柄失配 |
| N3 | create 回 UNKNOWN 但设备已发布（D14 竞态或传输） | mismatch | status 后继续 | mismatch | mismatch | status | status | 句柄未更新 → 拒 |
| N4 | 回 main 时忘了 select | mismatch，需再一步 | 不适用 | 同 A | 同 A | 不适用 | 不适用 | 不适用 |

**D 相对 B 多拦的：** M3、M6。**D 相对 E′ 多拦的：** 只有 M6（同名半步序列）。M6 本身是误用：两个调用者共用一个会话名，后来者还没走完 D 规定的两步。D 拦住 X 的同时也拦住了 Y；Y 一旦补上 select main，X 就会跟随。所以 D 并没有给 X 提供可依赖的同名保护。

**D 付出的代价：** N1–N4 四类误拒或多步；主机文件、锁、每命令读文件（文件损坏会让所有 UI 命令失败，`workspace.py:27-52`）；batch 冻结；`display select/create` 两个命令；`displays.md` 中大段 LLM 必读说明（公开 skill 共 90 行，偏好与保存语义约占三分之一）；`test_display.py` 567 行。**源码核对**

**零状态替代：**
- 自伤序列 M3：`--display` 必填即可消除，并且是在 open 时就挡住，比 D 在后续 UI 请求时才 mismatch 更早、更清楚。
- `noSession` 文案带上原种类：设备在内存中保留每个 owner 最后结束的 `{kind, displayId, reason}`，不持久化。worker 重建后没有这条记录，但必填参数已经覆盖。两者叠加，不需要任何主机状态。**设计建议**

#### 2.1.4 F（实例代际）

- lease 已有 `id`（`Sessions.kt:15-17,94,117`），在回执里暴露它几乎没有成本。
- 如果主机按 device+session 共享保存句柄：Y 的每次 open 都改写句柄，X 的下一请求会带上新句柄，于是在 M5/M6/M7 都跟随，比 D 更彻底。剩下能拦住的变化只有：worker 重建、expiry、close（这些情况已经是 noSession）、被设备上真人释放（sessionReleased）、被同 pairing 的其它会话名 takeover（X 已经得到 busy）。**共享保存的 F 相对 B 没有新增保护**，反而多出"open done 但句柄保存失败"这类状态。**推断（源码语义）**
- 只有调用者自己持有句柄（显式 `--lease N`，或存在工作区私有文件）才有实例级保护（M7）。代价是 LLM 每条命令都要复制一个数，过时句柄又会制造新的误拒。不推荐。
- 实例代际确实比种类保护更强：它能拦 virtual→另一 virtual、close 后重开、worker 重建；种类只能拦 virtual↔main。但实例保护要有价值，句柄就必须不被同名者共享，这和"按会话名共享"的前提相矛盾。上轮外评专题 13 **没有**跨请求代际机制：`sessionChanged` 只覆盖请求执行期间（专题 13:143；`Sessions.kt:210-214`）。所以实施方说"B 下同名另一调用者切到 main 后，旧调用者的新请求会跟随"**成立**，我方当时也承认这一点并交给"每任务一个名字"的指引（专题 13:18）。

#### 2.1.5 题 1 小结

- U23 选的"种类约束"，前提是隐式生命周期（U23 原话："期望 virtual 而当前没有后台屏时，在隐式取得 main 租约之前返回 not sent"）。全显式之后，这个前提消失了。D 是把为旧前提设计的保护叠到了新生命周期上。
- 我判断，对 LLM 最不容易出错、状态最少、能力不减的是 E′。E 的可选断言要求调用者每次记得带上，而会犯 M3 错误的调用者通常也会忘记断言，所以它主要只对 batch 有价值。

### 2.2 纯设备命令是否要求先 open（题 1 子问题）

- 现状：shell、files、通知、剪贴板、apps、wake 都隐式取得 main lease（`ShellJobs.kt:33`、`FileJobs.kt:24`、`UiBackend.kt:108`）。上轮外评 §5.2"后台屏空闲到期"一行指出：VD 过期后 wake/clipboard 会隐式建立 main owner，wake 还会点亮主屏。**源码核对**
- C 引入"有 owner、无 target"状态。为此需要分流两类入口、处理 status/close/资源组合，所有设备通道都要覆盖。收益只是每个任务少一次调用。
- 推荐：要求先 open（B/D/E′ 一致）。代价是每个任务一次 `session open --display main`（open main 不连接 UiAutomation、不要求亮屏，`TargetManagement.kt:405-407`）。job 和传输的后续操作仍按原 owner 免 lease（`ShellJobs.kt:55-79`、`FileJobs.kt:31-45`），这一点保持。把握：高。**设计建议**
- 需要在文档里写明：virtual 会话下的 shell、clipboard、wake 仍作用于整台设备，不受后台屏隔离（实施方 §2 正确）。

### 2.3 启动落点确认（题 6）

**原件核对**（`T/prototypes/route-review/launch/runtime-api34/`）：

- 主回执 `commands/vlc-view-once/stdout.bin`：`result=done`、`launchCode=0/START_SUCCESS`、`landing.state=unknown`、`matchingTaskCount=0`、`settleMs=1112`、`samples=5`、`exit=1`；`command.json` 记录 `exitCode=1`、`elapsedSeconds=1.625`、`--timeout 15s`。与实施方描述一致。
- 同请求 sidecar `artifacts/2242-…json`：`originalLiveCall2`（1228214→1228215）中 task174/display2 为 `StartActivity`、`visible=false`；`settleFinishedAtMs=1229327`；`afterSettle` 调用从 1229328 开始，即 **settle 结束后 1ms**，读到同一 task174 的 top 为 `VideoPlayerActivity`、UID 10142、`visible=true`，`realActivity` 仍是 StartActivity，`baseActivity` 已变为 VideoPlayer。`workDeadlineMs=1242460`。"即时不可见、末读可见"成立。
- **实施方没有强调的一点：** 同一回执 settle 内的最后一次观察（`snapshotSampleMs=1229314`，早于 settle 结束 13ms）已经是 VLC 页面。`artifacts/vlc-view-once.txt` 中有 `window {"id":82,"displayId":2,"type":"application","focused":true,"packageName":"org.videolan.vlc"}`，`screen {"app":"org.videolan.vlc"}`。也就是说，**按包确认在原 settle 时点内、用现有观察就已经能通过，不需要 settle 之后再读一次**。这把我方上轮"4 次都应得 target"的反事实估算推进到 1 个真实正例（仍只有 1 次，需设备扩充）。
- isFocused 撤回：`commands/contacts-after-b-edit-windows/stdout.bin:111-118` 中 VD3 `mCurrentFocus=Window{b16358 …ContactEditorActivity}`、`mFocusedApp=null`（`contacts-after-b-edit-activities` 第 545-552 行相同）。撤回有原件支持。

**判断：**

- 判定对象改为"目标屏上目标包（加 UID）可见"、main/VD 统一、时点放到动作后观察末尾，这三点都对。
- 未确认时（目标屏是其它已知包、没有可见 task、查询失败）不应让 batch 继续：后续计划步骤的前提不成立，停下让 LLM 看一眼，代价只是一次往返；继续执行则可能把输入打进权限框或其它 App。**新增"main 未确认也停批"值得**，前提是确认条件已经放宽，正常跳板不再误拒（上面的 VLC 样本说明能做到）。
- 数据来源：候选的 VD 落点在 `start()` 后立即读 roots + live（`ActivityLaunch.kt:254-258,272-282`），main 走另一套诊断（`:315-325`）。建议删除两来源交叉校验。目标屏前景首选**最后观察里目标屏的 application 窗口包名**：它就是 LLM 将要操作的那棵树，同一时点，零额外 IPC。再用一次未过滤 live tasks 读取提供 UID、elsewhere 和"他屏 task 移入"的后验。实施方 C 用"观察末尾读 live tasks"，也可以，差别不大；我的变体好在判定和观察同源、同时点。**设计建议**
- 我方上轮"other（前景是别的包）exit0"不成立：batch 会继续把输入打进别的 App。"读不到 task 不写 landing、exit 跟平台码"同样过松。实施方 §4.3 对这两条的批评成立。

### 2.4 跨屏已有同 App 任务（题 7）

**原件核对：** A 由 `am start --display 0 -f 0x10000000` 在 main 启动，B 由 `am start --display 3 -f 0x18000000` 启动（两者都是 adb shell，**不是产品的 `open` 命令**，`commands/contacts-main-a-insert-once`、`contacts-virtual-b-multiple-task-once/command.json`）。`contacts-after-b-edit-activities`：Display #0 中 task176，Display #3 中 task177，两者的 `app=ProcessRecord{28f6bf7 3134:com.android.contacts/u0a95}` 相同，Intent flg 分别为 `0x10000000` 和 `0x18000000`。`contacts-after-vd-close-activities` 中已无 #177，#176 仍为原 ActivityRecord `2045ab8`。实施方的事实描述成立；但要注意，这证明的是**平台 flag 能力**，没有经过候选的 preflight 或 open 路径。

**上游核对：** `agents3/work15/aosp/android-15.0.0_r1/ActivityStarter.java:3071-3101`：singleTask/singleInstance 遇到 NEW_DOCUMENT 时同时清掉 MULTIPLE_TASK；`documentLaunchMode=never` 仅在 singleInstancePerTask 且带 NEW_DOCUMENT 时清除。实施方指出我方"never ⇒ 一定去掉 MULTIPLE_TASK"不完整，**成立**。

**VLC 结构（源码核对）：** `sources/vlc-3.7.1-AndroidManifest.xml:1026-1030`：VideoPlayerActivity 是 `singleTask`；`vlc-3.7.1-StartActivity.kt:321` 由 StartActivity 跳转到 VideoPlayer。这正是"直接启动的组件是 standard，App 内跳到 singleton"的结构。我方上轮的三分支只看直接组件，会对 StartActivity 照常启动；如果主屏已有 VideoPlayer task，这次跳转可能把主屏的播放任务搬到 VD，并在 close 时销毁（**推断，需设备**）。候选的包级拒绝对这种情况是保守而有效的。

**判断：**

- 默认值：保留包级已知冲突拒绝（A/C 默认）。这对"真人正在用主屏"的场景损害最小，失败语义清楚（not sent，可重试）。代价是常用 App 在 recents 里有 task 时会被拒，LLM 需要多一次往返。
- 显式选项（实施方 C）：建议命名 `--multiple-task`，直接对应平台 flag，避免"new instance / safe"之类暗示隔离的词。语义："加 NEW_TASK|MULTIPLE_TASK；与主屏实例共享进程和数据；App 内跳转仍可能移动他屏 task；结束时随屏销毁本屏 task"。
- 带上该选项时，按已解析的 ActivityInfo 先拒绝必然无效的情形：launchMode 为 singleTask/singleInstance；documentLaunchMode=never 且 launchMode 不是 singleInstancePerTask。这些在 resolve 阶段就已知，几行代码，不复制 findTask。
- 拒绝文案直接给两条出路：`--multiple-task`（附共享和销毁说明），或改用 main。
- 不做"移入/借用归还"（D）。把握：中高。**设计建议**

### 2.5 后台屏本身

| 项 | 实施方 | 我的判断 |
| --- | --- | --- |
| 建屏 flag 7 项 + 读回三位 | 保留 | 同意（`WorkDisplay.kt:37-70`，**源码核对**）。尺寸取当前方向（`:36` `getRealMetrics`），是否改成自然方向属小事，需设备。 |
| 逐输入检查 target ON（`UiBackend.kt:73-77`） | 保留 | 同意保留。只读一次状态，代价近零；A16 上 VD 跟随显示组，与主屏门大致冗余但无害。我方上轮"删除"理由只是精简，撤回。 |
| main 亮且解锁门（`DeviceFacts.kt:69-81`） | 保留 | 同意。 |
| 不加 HIDE、保活、锁屏豁免 | 不加 | 同意。 |
| 不做 doctor | 不默认做 | 基本同意，但建议把**静态前提检查并进 `open --display virtual` 的最前面**（在 `prepareChange` 让旧 lease 失效**之前**）：检查 shell 是否持有 `ADD_TRUSTED_DISPLAY`、是否有 `activities_on_secondary_displays` 特性。不支持的设备得到准确原因，也不会白白结束已有会话。现状下 `prepareChange` 先让旧 lease 失效（`Sessions.kt:120`），之后 `createVirtualDisplay` 抛 SecurityException（D13），调用者原有的 main 会话就丢了。动态能力（窗口可读、输入、帧）由第一次真实使用暴露，再配合落点 elsewhere 报告，已经足够。独立的 live doctor 对 LLM 调用者没有不可替代的场景。**设计建议** |

### 2.6 D10、D12–D15（源码核对）

- **D10 成立**：`display create` 在 prepare 之后失败 → `current=null`（`Sessions.kt:120`、`TargetManagement.kt:82-95`）；主机只在 done 时保存（`app.py:839`），偏好仍为缺省 main → 下一条 UI 请求 `expected=main` → `keepLease` 新建 main（`Sessions.kt:94,146-148`）。消除它的是"删除隐式建立"，与偏好无关。实施方引用的 API37 原件本轮未重读（见 §8）。
- **D12 成立**：`DeviceFacts.kt:39` 把 `displayId` 传给 `isDeviceLocked(userId, deviceId)`。A15 `TrustManagerService.java:1832-1836` 在 deviceId≠0 时直接 `return false`。影响限于 VD 的 `targetFacts` 报告值；输入门用的是 main 的 facts（`UiBackend.kt:72-75`）。实施方"主输入门未被绕过"成立。
- **D13 成立**：`WorkDisplay.kt:52-61` 不捕获 SecurityException，`Wire.kt:75-78` 把非 Rejected 异常映射为 `workerFailure`。
- **D14 成立**：`TargetManagement.kt:49-54` 在 `synchronized(run)` 内完成 publish，`:59` 的 `remember(done)` 在锁外；中间如果 `response()` 先执行（`UiRun.kt:126`），回执是 begin 时写的 "Work display creation pending" UNKNOWN，而设备已经发布。D 模型下主机因此不保存 virtual，后续 mismatch（N3）；E′ 下只需 status。
- **D15 不成立（实施方正确，我方错）**：`UiRun.failure`（`UiRun.kt:116-124`）在已有 fact 时返回原 fact 的副本，只有原结果为 NOT_SENT 时才改写 reason/error；`changeTarget` 的 catch 只记 FAILED 或 UNKNOWN（`TargetManagement.kt:67-86`），cleanup 失败只追加 `cleanupError`（`:96`）。原失败不会被覆盖。

### 2.7 组合一致性与最干净的整体模型

候选链路"会话 → 显示目标 → 窗口 → task"上的重复事实来源：

1. 种类有两个来源：主机偏好 vs 设备 `lease.target`。前者只是意图，后者才决定路由；两者不一致时产生 N1–N4。
2. task 有两个来源：roots + live 非原子交叉校验，不一致就报 unknown（实施方 §4.2）。
3. 落点有两个时点：`start()` 后立即读 task vs 观察末尾的窗口树；VLC 原件中前者 unknown、后者已是目标包。
4. main 与 VD 落点规则不同（`ActivityLaunch.kt:252-262`）。
5. 冲突判定（包级 preflight）与平台实际行为（findTask 理想匹配 + App 内跳转）不同。这是有意的保守近似，可以保留，但要写明。

**E′ 整体模型（设计建议）：**

- 设备 lease = owner + target{main | virtual#id} + id。只有 `session open --display main|virtual` 和 `session takeover --display …` 能创建或替换；所有新的设备命令在无 lease 时返回 `noSession`（job/传输后续操作除外）；`sessionReleased`、`busy`、`sessionChanged` 保持现状。
- `open --display virtual`：先做静态前提检查（不碰旧 lease），再 prepare、建屏、在同一临界区发布并记 done（D14），失败时写明"当前无会话"或"当前仍为 main"。
- 主机：不存显示状态，请求不带种类；删除 `display select/create`、`display.json`、batch 冻结。回执首行在非 main 时带 `display=virtual#N`；status 显示目标。
- 启动：一次完整 live tasks 读取用于启动前包级冲突判定和启动后的 elsewhere/moved 后验；本屏前景由最后观察的窗口给出。main/VD 统一规则，目标未确认就 exit≠0。可选 `--multiple-task`。
- 销毁：随 lease 结束销毁屏；不做归还、保活、HIDE。

**相对实施方 D 的收益：** 删除主机偏好、锁、读文件和三类保存失败语义；删除两个管理命令和一个每请求字段；回 main 只需一步；消除 N1–N4；自伤序列 M3 在 open 处就被挡住；文档更短。
**代价：** 失去 M6（同名半步序列）上的拦截；`session open` 现有不带参数的写法变为参数错误（破坏性，但生命周期本身已在改）；与用户 U23/U25 的选择相反，需要用户重新裁决。

---

## 3. 对照实施方

| 实施方结论 | 判定 | 证据 |
| --- | --- | --- |
| A/B/C/D 四案描述（`sessions-display.md` §整体会话候选补查，表 208-213） | 描述与源码一致 | §2.1.1 |
| "D 保异种类拒绝，B 撤掉 U25 保护" | **部分成立**：D 多拦的只有 M3、M6；M3 用必填 `--display` 即可零状态消除，M6 是同名误用且 D 也只能拦到 Y 补上 select 之前 | §2.1.3 |
| "B 下同名另一调用者切到 main，旧调用者新请求跟随" | **成立**；但 D 在正规两步下同样跟随（实施方表 244 行自己也写了"后续新调用共同跟随"） | 专题 13:18,143；`sessions-display.md:242-244` |
| "删除 expected 不等于删除隐式建立" | 成立，但在全显式前提下，expected 剩下的价值只有 §2.1.3 那几行 | — |
| D 中"open main 不改偏好、回 main 要两步" | 事实成立（`app.py:741-760`）；实施方自己的 Contacts 收尾就走了两步（`commands/contacts-cleanup-select-main`、`contacts-cleanup-open-main`）。这是 D 的负担，不是优点 | 原件核对 |
| 推荐 D | **不同意**，推荐 E′ | §2.1 |
| 纯设备命令要求先 open | 同意 | §2.2 |
| 实例代际"新增生命周期/存储消费者，不是免费" | 成立；共享保存时没有收益 | §2.1.4 |
| 题 6 推荐 C，及"main 未确认也停批" | 同意；建议本屏前景改用最后观察的窗口（同源同时点） | §2.3 |
| VLC 原件描述 | 成立；补充：settle 内观察已见 VLC 窗口 | §2.3 |
| isFocused 撤回 | 成立 | §2.3 |
| 题 7 推荐 C（默认拒、显式新 task） | 同意；补充选项命名和 resolve 阶段拒绝必然无效的情形 | §2.4 |
| Contacts 双草稿事实 | 成立；但它是 adb `am start` 的平台能力正例，没有走候选的 open/preflight | §2.4 |
| 不默认 doctor | 同意；补充"静态检查前置到 open virtual、早于让旧 lease 失效" | §2.5 |
| 保留逐输入 target ON | 同意 | §2.5 |
| D10 "既有契约取舍，交整体模型" | 成立（由显式生命周期消除） | §2.6 |
| D12/D13/D14 | 成立 | §2.6 |
| D15 "外评覆盖原失败不成立" | **成立** | §2.6 |
| 对我方的反驳：§5.2"结束后要么 virtual 要么什么都没有，不会是 main"过强 | **成立**：busy 或校验在 prepare 前失败时，旧 main 仍在（`Sessions.kt:112-116`） | 源码核对 |
| 对我方的反驳：documentLaunchMode=never 表不完整 | **成立** | A15 ActivityStarter:3090-3101 |
| 对我方的反驳：专题 14 保 roots、专题 15 保 live，"一个来源"没给理由 | **成立** | `sessions-display.md` §4.2 |
| 对我方的反驳：other exit0、读不到就跟平台码过松 | **成立** | §2.3 |
| 对我方的反驳：默认 live doctor、"20% 回退阈值"没有依据 | **成立** | §2.5 |
| 对我方的反驳：默认自动新实例 | **成立**（App 内跳板的反例结构） | §2.4 |

---

## 4. 对上轮外评（我方）的更正

1. §5.8/§7 D15"清理错误覆盖原失败"：错误。`UiRun.failure` 保留原 fact。
2. §5.2"open virtual 结束后设备上要么 virtual 要么无，不会是 main"：过强。prepare 前的 busy、校验或主机侧失败都会保留旧 main。
3. §5.2 `--display` 缺省为 main（§2.8 第 2 项）：建议改为必填。这是消除 M3 自伤序列的零状态手段；缺省 main 加上"回执上能看出来"属于事后可见，不是事前防护。
4. §5.6 落点表"other → exit0""读不到 task → exit 跟平台码"：过松，应改为未确认即停批。
5. §5.6"只用一个任务事实来源"没有指明是哪个；专题 14 与 15 的选择互相矛盾。
6. §5.7 三分支默认自动 MULTIPLE_TASK：挡不住 App 内跳到 singleton 的情形；never 规则写得不完整。改为默认拒绝加显式选项。
7. §5.9 默认 live doctor 和回退阈值：没有必要，也没有依据；改为把静态检查前置到 open。
8. §5.3"删除逐输入 target ON"：撤回，收益可以忽略。
9. "4 次 VLC 都应得 target/0"原是估算；新原件提供 1 个真实正例（settle 内观察已是目标包），仍需设备扩充。
10. 我方对删偏好的论证（"那一行改掉，偏好就没有存在理由"）方向对，但没有逐序列列出偏好剩余的保护（M3、M6），也没有给出零状态的替代（必填 `--display`）。本报告补上。

---

## 5. 确定缺陷（本轮核过）

| 位置 | 缺陷 | 可观察后果 |
| --- | --- | --- |
| `Sessions.kt:94,146-148` + `app.py:839` + `workspace.py:55-56`（D10） | create 在 prepare 后失败，偏好仍为 main，下一条 UI 隐式建立 main | 本想在后台屏执行的 selector 动作落到真人主屏 |
| `DeviceFacts.kt:39`（D12） | `isDeviceLocked` 第二参数传了 displayId | VD 的 targetFacts 中 `deviceLocked` 恒为 false（仅影响报告） |
| `WorkDisplay.kt:52-61` + `Wire.kt:75-78`（D13） | SecurityException 被映射为 `workerFailure` | 不支持的设备（如 MuMu）得不到 `displayUnsupported` 和具体原因 |
| `TargetManagement.kt:49-59`（D14） | publish 与 remember(done) 不在同一临界区 | 设备已发布 virtual，回执却是 UNKNOWN；在 D 模型下还会导致后续 mismatch |
| `Sessions.kt:116-120` + `TargetManagement.kt:405-409` | 在任何建屏前提检查之前就让旧 lease 失效 | 不支持后台屏的设备上，一次失败的 create 就会结束调用者现有的 main 会话 |
| `Sessions.kt:160-168` 提示文案（设计缺陷） | 期望 virtual 而实际 main 时，提示 "run display create or display select main" | 遗留偏好（N1）会把只想用主屏的 LLM 引向建屏 |

D15 不是缺陷。D11（落点确认过窄）已由原件确认，见 §2.3。

---

## 6. 需要用户裁决的事项

### 6.1 会话与显示意图（题 1）

| 候选 | 能力 | 调用者负担 | 维护 | 评价 |
| --- | --- | --- | --- | --- |
| D（实施方） | 拦 M1–M3、M6；同名正规两步时跟随 | 回 main 2 步；N1–N4 误拒；文档长 | 主机文件、锁、每请求字段、冻结、两个命令、保存失败语义 | 偏好的主要用途已被显式生命周期取代 |
| B（上轮我方） | 拦 M1、M2；M3 不拦 | 每任务 1 次 open | 最少 | M3 是 LLM 实际可能犯的错 |
| E（协调方） | 同 B，带断言时更强 | 断言要自己记得带 | 少 | 主要对 batch 有价值 |
| **E′（推荐）** | 拦 M1–M3（M3 在 open 处）；M4–M7 不拦 | 每任务 1 次 open，必须写 `--display` | 最少 | 状态最少，最不容易被误导 |
| F（共享句柄） | 不比 B 多拦 | 同 B | 多一个文件 | 不推荐 |
| E′ + batch 断言（可选） | 额外拦 batch 内的 M4–M6 | batch 头一行 | 设备端代码已有 | 用户看重同名并发时再加 |

推荐 E′，把握约 70%。最大的不确定在用户偏好：用户是否把"同一会话名的两个调用者"当作需要产品保护的真实场景。如果是，就应当提供**调用者自持的实例句柄**，而不是 D 的种类偏好；但我不推荐做。

### 6.2 纯设备命令

推荐要求先 open（不引入 C 的无 target 状态）。把握：高。

### 6.3 落点确认（题 6）

推荐实施方 C，加上"本屏前景取自最后观察窗口"的变体，未确认即停批。把握：中高。

### 6.4 跨屏已有任务（题 7）

推荐实施方 C：默认包级拒绝，显式 `--multiple-task`，并在 resolve 阶段拒绝必然无效的组合。把握：中高。

### 6.5 doctor

推荐不做独立 doctor；把静态前提检查前置到 `open --display virtual`，并修 D13 的原因码。把握：中高。

---

## 7. 会改变取舍的验证问题

1. **LLM 在 noSession 之后实际怎么做（M3 频率）。** 影响 §6.1。用干净 agent 跑 VD 任务并制造 idle 到期：如果大多数会不带参数直接 `session open`，必填 `--display` 的价值得到确认；如果都能按文案带回原参数，则缺省 main 加文案（B/E）也可以接受。不需要设备，可以用假 daemon 做。
2. **用户是否需要同名并发保护。** 影响 §6.1。需要 → 提供调用者自持句柄（不是 D）；不需要 → E′。
3. **其它跳板 App 在 settle 内最后观察中能否见到目标包。** 影响 §6.3 中前景取窗口还是取 tasks。多数能见到 → 用窗口；常见窗口晚于 task → 改用实施方的"观察末尾读 live tasks"，或在剩余预算内重读。需设备。
4. **用户真实常用 App 在 recents 中有 task 的比例，以及 `--multiple-task` 后的行为。** 影响 §6.4 的默认值。几乎每次都被拒、且加 flag 后普遍正常 → 可以考虑把默认改为"按 launchMode 自动加 flag"；出现明显的共享状态问题 → 保持默认拒绝。需设备。
5. **主屏有 VideoPlayer task 时，在 VD 用 VIEW 启动 VLC。** 影响 §2.4 中跳板会搬 task 的推断。若被搬，包级默认拒绝的理由坐实；若不被搬，我方旧三分支的风险被高估。需设备。
6. **静态前提（权限和 feature）能否预测真机上 create 的成败。** 影响 §2.5 的前置检查。能预测 → 前置检查足够；常见"静态通过、动态失败" → 才考虑显式 live 诊断。需设备。

---

## 8. 未核实与局限

- 没有运行任何设备、模拟器或构建；所有"需设备"项都未验证。
- D10 引用的 API37 原件（`runtime-release-api37/commands/virtual-create-once`、`vlc-virtual-view-once`）本轮未重读，依据是源码推演和上轮记录。
- `ActivityTasks.kt` 中 `virtualActivityPreflight`、`virtualActivityLanding` 的细节未逐行重读，依据是实施方 §4.2 和上轮审计；`ActivityLaunch.kt` 的调用顺序已核。
- findTask 跨显示的理想匹配规则沿用实施方与上轮专题 14/15 引用的 A14 行号，本轮只亲核了 `adjustLaunchFlagsToDocumentMode`（A15）和 `isDeviceLocked`（A15）。
- §2.1.3 的序列表是按源码语义推演的，没有用实现的 E′ 实际跑过。
- 窗口 `packageName` 由无障碍树推得；把它用作前景判定的可靠性（例如悬浮窗、系统窗口）需设备确认。
