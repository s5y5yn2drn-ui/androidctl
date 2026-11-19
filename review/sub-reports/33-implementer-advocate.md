# 实施方辩护人报告：对裁决草案 P1–P18 的反驳与支持

只读核查。范围：`DRAFT.md` 的立场 P（重点 P11、P8、P5、P7/P12/P15、P3/P14、P13、P17）。没有读 `agents6/26–32` 的报告。
证据标签：【原件】【源码】【重算】【推断】【需设备】。源码版本：候选 `SRC`=3f45af6；AOSP 用 `agents3/work14/src/android-1x.0.0_r1`、`agents6/29/aosp`（UiAutomationManager）、`agents5/24/aosp`（ManagedServices），以及我自己下载的 `agents6/33/aosp/a14-PendingIntent.java`（android-14.0.0_r1）。Compose 源码取自 `agents6/34/dl/ax/`，那里是别的 agent 下载的上游文件，我只把它当上游原文读。
没有设备，所有运行性结论都只引用实施方的模拟器原件（MuMu / API34 AVD），每类样本都只有 1 次。

术语：AX = 无障碍（AccessibilityNodeInfo）通道；IC = 输入法连接通道（候选里 BrowserSupport 服务的 `AccessibilityService.InputMethod`）；E9 = 候选用手写 Binder 回调批量取节点的采集路线；PI = PendingIntent；VD = 后台虚拟屏；REF = 观察里分配给节点的引用号。

---

## 一、逐条

### P11 结果与退出码的"简单规则"

**评审方立场**：Result 只描述主动作，永不因交付或清理失败改写；整条命令的 exit 为 0 当且仅当主动作 DONE、请求的交付物都已交付、清理已确认；batch 只看"该步 exit 非 0"。评审方还说实施方 A 有内部矛盾："单独执行退出 0，放进 batch 停批并退出 1"。

**反证与支持**

1. **所谓"内部矛盾"站不住。**【源码】`ui_batch.py:210-225` 的 `aggregate`：batch 一旦停止，它自己的 Result 就是 `failed`，exit 是 `exit_code`。`ui_batch.py:362-376` 里，步骤 Result 保持 done，停批理由是"Step N delivery failed"。所以 batch 的 exit 1 对应的是 **batch 自己的 Result=failed**（计划没跑完），在 batch 这一层是一一对应的。单条命令没有"后续步骤被跳过"这件事。`skills/androidctl/batch.md:43-50` 也明文写了 `back --screenshot` 在 batch 里"done + imageError，batch failed/exit1"。两个命令各自声明的条件不同，这不是矛盾。
2. **P11 照字面执行，会让读命令变差。**【源码】`ui_client.py:1387-1404` 的 `finish_primary_result`：snapshot/find 没交付树文件、screenshot/zoom 没交付图文件时，结果改成 `failed`。`images.md:144-146` 写明"missing that artifact fails an otherwise successful observation"。读命令的"主动作"就是交付物本身。按 P11"永不因交付失败改写 Result"，`snapshot: done` 会没有文件，调用者照 `SKILL.md:41-44`"读 data.snapshotPath"就会扑空。P11 必须明确豁免这四个读命令，草案没有写。
3. **"请求的交付物"没定义会误伤所有动作。**【源码】动作命令默认会自动保存观察正文（`images.md:112-113`："actions… still save their bodies automatically"）；历史配额不够时，这次保存直接被拒（`images.md:132-134`）。如果自动保存也算"请求的交付物"，配额、磁盘或权限这类常见问题就会让每个已经 DONE 的动作都非零退出。必须沿用 batch 已有的口径：只有 `explicit_delivery`（`ui_batch.py:186-195`：读命令、`--out`、`--image-out`、`--print`、`--diff`、`--screenshot`）算"请求的"。
4. **非零退出取哪个值，草案没说，这关系到会不会重发。**【源码】`reference.md:62-66` 定义 exit 1 是"known failure"，exit 3 是 UNKNOWN，而且"Do not repeat this action automatically"（`output.py:72`）。如果"动作 DONE、截图没存成"给 1，"非零就重试"的外层包装、或只看 exit 的 agent 会把已经发出的点击、发送当成失败再做一次。不过同类风险现在就有：清理未确认时是 done+exit3（`reference.md:68`、`app.py:854,863`）。所以 P11 并没有引入新的风险类别，前提是文档写清楚"Result=done 而 exit≠0 时不要重发"（现在 `images.md:120,146-149` 已经这样写）。结论：退出值可以取 1，与 batch 的现行做法一致，但 `reference.md` 那张表必须改成"exit 1 也可能是 done 加交付缺失"。
5. **"清理 3"和"UNKNOWN 3"混用现在已经存在，而且方向安全。**【源码】`reference.md:68`。两者都意味着"不要盲目重发"，Result 字段能区分。不需要拆。
6. **实施方提到的"stdout 晚失败"约束，范围比实施方说的窄。**【源码】单条命令的连接关闭在打印之前（`ui_client.py:1001-1007`）；文件保存在打印之前（`delivery_scope`）。打印之后只剩 stdout 本身和 diff 基线提交（`ui_client.py:1015-1021`），而 r3 已经决定删掉持久基线。stdout 失败时调用者多半根本没收到完整 JSON，所以用进程 exit 表达失败反而是唯一可靠的通道。按 P11，只有一种窄情况会让"JSON 已完整写出、`exit` 字段缺省为 0"与进程 exit 不一致：写成功了，随后 flush 或关闭失败。规则里要写明"进程 exit 为准"。现有测试 `tests/test_result_output.py:164-182` 断言"动作 done + deliveryError → exit 0"，采用 P11 要改这条测试，属于有意的契约变更。

**结论：部分对，需要这样改。** 简单规则可以采用，而且比 A 好讲；但必须补四点：(a) snapshot/find/screenshot/zoom 照旧以主文件交付为准，保持 failed；(b) "请求的交付物"= `explicit_delivery` 口径，自动历史保存和配额回收不计；(c) 写明 done 且 exit≠0 的含义和"勿重发"；(d) 进程 exit 为准。评审方关于"A 内部矛盾"的论据应当删掉。

### P8 通知落点用创建者包，不加两个隐藏接口

**评审方立场**：只用创建者包加 UID 做发送前拒绝和发送后核对；跨包 PI 得到诚实的 UNKNOWN。

**反证**

1. **跨包时，"发送前按创建者拒绝"确实形同虚设，而且这里的危害是不可逆的搬迁。**【源码】AOSP 14 `ActivityStarter.java:2944-2996`：可复用的任务不在目标显示区域时 `differentTopTask = true`，随后 `intentTask.reparent(mTargetRootTask, …"reparentToTargetRootTask")`，也就是把别屏上已存在的接收方任务整体搬到 VD。PI 目标是另一个包时（最常见的是隐式 `ACTION_VIEW` 交给浏览器或地图，或者创建者是 `android` / SystemUI 而目标是 Settings），按创建者预拒查不到任何任务，于是放行；用户主屏上的浏览器或设置任务会被搬走。P8 事后给的 UNKNOWN 很诚实，但搬迁已经发生。【推断】哪类通知是跨包，我没有逐 App 实测：同包是主流（App 自己的 Activity，含 FCM 自动展示；Android 12 起禁止通知跳板，进一步推向直接 Activity）；跨包集中在系统通知、"打开链接"类、代其它 App 发的通知。下载完成通常是广播 PI，不在本门范围。
2. **系统通知在 P8 下是 100% 没用的 UNKNOWN。**【推断+源码】创建者是 `android`（UID 1000）时，它自己几乎没有可见 Activity 任务。按 P8 的落点核对永远对不上，每次都停批。
3. **"两个隐藏接口"的维护成本被高估了。**【源码】`a14-PendingIntent.java:1365-1370`：`queryIntentComponents` 标注了 `@SystemApi(client = MODULE_LIBRARIES)`、`@TestApi`、`@RequiresPermission(GET_INTENT_SENDER_INTENT)`。它是带兼容约束的系统 API，不是普通 `@hide`。`getIntent()`（`:1329-1333`）才是 `@hide @UnsupportedAppUsage`，而且它并非必需：`queryIntentComponents` 返回的 ResolveInfo 已经带包名、组件和 UID（`intent-getter-audit/result.md` 第 2 点）。shell 有这项权限【原件】（同文 `notifications-shell-permissions/stdout.bin:768`，API34，1 次）。
4. **实施方自己也漏了同一点。**【原件】r3 主稿第 7 题 B 写"也不增加 receiver 别屏任务新门"（主稿 :189）。用这个接口最大的价值其实不是把 DONE 判得更准，而是**发送前按预期接收方预拒**：候选包或 UID 在别屏有活任务就 not sent；没有候选或有歧义时，在 virtual 会话里拒绝或要求显式确认。评审方和实施方都没提。

**结论：评审方部分错。** 建议改为：只保留 `queryIntentComponents` 一个接口（不要 `getIntent`），用它同时做"按预期接收方的别屏预拒"和落点核对；拿不到候选时退回创建者规则，结果为 UNKNOWN。这是 P8 和第 7 题 B 之外更安全的一档。【需设备】跨包 PI 的真实搬迁目前没有原件（实施方只有同包 Termux 正例和一个零发送冲突例）。

### P5 删除 IC 独立选区

**评审方立场**：只留 AX `select REF START END`；IC 选区在 Compose 上一律不会发出；没有"AX 失败、IC 成功"的样本；命令没发布过。

**核查**

- 【源码】Compose 不覆写 `getSurroundingText`：foundation 和 ui 的 `RecordingInputConnection`、`StatelessInputConnection`（androidx 提交 2cda5a0d…及 refs/heads 两版，`agents6/34/dl/ax/`）只覆写 `getTextBefore/AfterCursor`、`getSelectedText`、`getExtractedText`，不覆写 `getSurroundingText`。平台默认实现的 offset 是 -1（这一点草案 F5 已写）。候选通过 `AccessibilityInputConnection` 只调用 `getSurroundingText`（`BrowserAccessibilityService.kt:46-51`），拿不到 `getExtractedText`。`EditorSurrounding.interpretable` 要求 offset≥0（`EditorSurrounding.kt:11-12`），`CurrentEditorInput.kt:98-99` 因此返回 NOT_SENT。"Compose 上一律不发"属实。
- 【源码】评审方没提、却支持删除的一点：**IC 选区只能落在光标前后 128 个 UTF-16 单位以内**（`CurrentEditorInput.kt:96,169` `CONTEXT_UNITS = 128`，`input.md:30-31`"Both must be inside this command's complete known surrounding range"）。实施方所说的"当前 editor 定位"能力，在长文本里只能做局部移动，比主稿第 3 题 B 描述的窄。
- 【原件】我在材料里没有找到任何"AX 选区失败而 IC 选区成功"的样本。A029 对 B040（`api34-editor-pair/pair-result.md:3,22`）比的是 AX fill 对 IC commit，不是选区；而 IC commit 会保留。IC 的 063 select 2..6 是单行 pattern，AX 同样能做。成本：IC select 0.531 s 对 AX select 0.875 s（`api34-ax-ic/mapping-result.md:30`，以及 pair 的 039），各 1 次，差别不大。
- 【推断】IC 能做而 AX 不能的唯一像样的场景：调用者手里没有可用 REF。例如大页面被 8192 节点上限或字节上限裁尾，聚焦的编辑框（长 issue 末尾的回复框）不在交付树里；或者该节点 `refAvailable=false`。没有原件。
- 【推断】regex101 的业务位置（Match Information 46–49）属于原生文本域，IC 选区可以直接用；用 AX 要在 AX 文本里重新找子串（48–51）。对按 AX 文本工作的 LLM 来说，后者其实更自然。

**结论：评审方基本对。** 可以删，但建议用一个更便宜的办法补上"无 REF"这个缺口：让 AX 选区接受"当前输入焦点节点"作为目标（UiAutomation 可直接 `findFocus(FOCUS_INPUT)`，不需要整树和 REF）。这样就不需要第二套偏移域。另外，评审方"以后再以 `select START END --via editor` 加回"的说法，应同时写明 128 单位窗口这一限制。

### P7 监听器寿命：简化版每租约

**评审方立场**：只保留一个"可能已登记"标志；租约结束时注销一次，失败就保留标志；下一条通知命令先读，读被拒再注册；不要"责任转交、保来源、status 分列"。

- 【源码】租约结束确实汇聚到 `Sessions.kt:281-284` 的 `invalidateLease→clearLeaseUi`。简化版成立。
- **一个缺口**：P7 写"读被拒再注册"，但没限定是哪种拒绝。【源码】AOSP 14 `ManagedServices.java:1069` 未登记时抛 `SecurityException("Disallowed call from unknown …")`；`registerServiceImpl`（`:1706-1711`）对同一个 Binder 不去重，每次都会 `linkToDeath` 并加入 `mServices`。实施方 API34 的 heap 原件也显示同一 Binder 每注册一次强引用就多一份（F17）。如果读是因为别的原因被拒（权限、用户被切换、参数错误），而原登记其实还在，重新注册就是重复登记，正是这项设计要避免的累积。所以应当只在看到"unknown listener"这种拒绝时才重新注册，其它拒绝照实报告。不过评审方的规则也修了实施方方案的一个盲点：system_server 重启后旧登记全部失效（F24），而实施方"不自动再 register"会让通知一直不可用，直到 worker 重启。
- **status 分列成本极低，不该删。**【源码】`WorkerService.kt:149-152` 的 status 已有 `diagnostics.uiCleanup` 槽位。注销失败之后，监听器仍以 shell 身份接收所有通知回调和 URI 授权，这关系到用户隐私，在 status 里加一行"注销未确认、登记可能仍在、附错误"就够了。

**结论：评审方大体对。** 需要两处修改："只在 unknown listener 拒绝时重新注册"；status 保留一行"登记可能仍在"。"保来源、责任转交"可以删。

### P12 清理恢复

- (a)【源码】`agents6/29/aosp/android-14.0.0_r1-UiAutomationManager.java:101-106` 只有一个注册位，被占时抛"already registered!"；`:143-155` 注销时比对 Binder。新注册成功，确实证明旧注册已经不在。但需要补两点：(1) 注册失败**不能**证明是我们自己的旧注册卡住了，系统里别的 UiAutomation 用户（如 `uiautomator`、仪器测试）也会占这个位；(2) 不要先"探测注册"再断开，否则断开再失败就回到原状。应当把下一次正常连接（`UiBackend.connectPlatform`，`UiBackend.kt:445-470`）本身当作探测，成功就直接作为当前连接沿用。这样就不需要单独的 retry 动作，也不会和正在使用的连接冲突：worker 里只有一个 `automation`。
- (b)【源码】`Sessions.kt:232-235` 的 `stop()→requireIdle()` 和 `WorkerService.kt:115-123` 证明正常 Stop 也会被挡，评审方对。实施方写的"本机 Stop"在措辞上不准确，实际只能走 `connection.md:163` 的 Force executor exit。
- (c) 拆开所有权是对的。但拆开以后，新会话的 UI 命令在旧注册仍占位时会失败，需要一个专门的错误码（例如"UiAutomation 仍被占用"），并指向本机强制结束；否则调用者只会看到一个泛化的注册异常。

**结论：评审方对，按上面补充。**

### P15 wake 改走输入注入

- 【源码】AOSP 14/15/16 `PhoneWindowManager.interceptKeyBeforeQueueing`：KEYCODE_WAKEUP 置 `isWakeKey`（14 `:4296-4300`，15 `:4928-4933`，16 `:5520-5526`）。非交互、非注入的分支保留 wake 键（14 `:4054-4063`），随后 `wakeUpFromWakeKey`（14 `:4434-4436`）。没有任何按"注入/非注入"或注入通道拦截 wake 的分支。`InputManagerService.injectInputEventToTarget`（14 `:864-887`）只检查 INJECT_EVENTS 权限；targetUid 用 `INVALID_UID` 时不做目标 UID 校验。候选的 `keys()` 已经在这样用（`SystemActions.kt:136`）。UiAutomation 的注入最终也进同一个 native `injectInputEvent`，策略层看到的是一样的事件。
- 【源码】要注意，wake 现有的轮询用的是 UiAutomation 的事件信号（`SystemActions.kt:108-115`：`ui.platform(run).signals`、`awaitNextCheck`），改路线时要换成 PowerManager 轮询。`ready()` 对 wake 只读 `deviceFacts`（`:82`），不需要连接。
- 我没找到候选当初选 UiAutomation 的硬理由；同文件的 home/recents 也走同一个 `policyKey`，看起来只是复用。

**结论：评审方对。**【需设备】确认熄屏和锁屏下确实能唤醒（每个系统镜像至少 1 次）。

### P3 保留 E9 并加快速失败

- 【源码】可行，而且候选现在的缺陷比 F/P16(d) 写的更重。`CollectionNodes.kt:130`：布局校验失败时 `return false`，既不打标记也不唤醒等待线程。`awaitPrimary` 于是一直等到采集截止（`:160-163`），之后每个子节点都因 `>= until` 返回 null。结果是**白白耗光整个采集预算，再退化成只剩根节点**，而不只是"静默退化"。`readTypedObject/createTypedArrayList` 如果抛异常，oneway 事务里的异常会被 Binder 框架记日志后吞掉，同样到不了等待线程。
- 修法完全在进程内：给 `onTransact` 包一层 try/catch，校验失败或抛异常时置"布局不兼容"并 `ready.countDown()`；收到这个回调对象上未知的事务码也记为不兼容。出站的 `findNode` 是同步调用（`:43-55` 有返回值），签名变化导致的异常本来就能捕获。唯一捕获不到的是"系统根本不回调"，这和"App 慢"分不开，只能按超时处理。

**结论：评审方对**，而且快速失败是可行的、便宜的。

### P14 稀疏表示

- 【原件】实施方 `r2-nodes-waits.md:71` 自己写了"选择纯稀疏内部记录还是定稿前稠密 builder 应按完整消费者链决定"，设备端两边其实一致。早期预算和最终定稿用同一个编码器，两种做法都满足 U27 第 1 条（同一算法）。同时两种做法都**不**把派生字段（row/view/refReason）计入早期预算，实施方已承认这一点（`reveal-identity…md` §6.1）。
- **分歧在主机端，评审方的做法有一个具体陷阱。**【源码】`ui_client.py:1204`：`--raw` 正文是 `json_value(observation)`，即把解析后的对象**重新序列化**。如果主机在"唯一解析入口"把默认值补进字典，raw 输出和其它重新序列化的路径就会变回稠密（California 页 12.5 MB），等于在产品里重现评审方归咎于研究主机的那 0.891 s，同时多占历史配额（20 MiB）。实施方"不保长期 dense 展开层"防的正是这个。
- **结论：部分对。** 可以按评审方做，但必须规定：补默认值只发生在只读视图层；raw、保存和传出一律使用收到的稀疏字节或稀疏字典。否则就照实施方的做法。

### P13 身份 B/B 不必等稀疏

- 【源码】评审方对机制的描述正确：`UiSnapshot.kt:31-49` 先 `UiSemantics.project` 和 `refs.replace`，再 `UiTreeBudget.fit`；`fit` 裁掉尾部，标 partial/treeByteLimit，修 `childrenIncomplete`；只有被保留节点的 REF 被安装（`:44-47`）。所以实施方"B 在旧稠密下超限、必须先完成稀疏"是错的：超出 86,150 B 会被如实裁尾，不会发出超限的树。
- 【重算/推断】代价量级：旧稠密 California 每节点约 8,382,705/5485 ≈ 1,528 B，超出部分约 56 个节点，也就是一个**本来就是 partial** 的页面再从尾部少掉约 1%（按文档顺序，在最后一个窗口的最末端）。这是边际收缩，不是新类型的能力回退。【推断】哪个窗口排在最后我没有核实，如果系统栏排在最后，被裁掉的可能根本不是 App 内容。
- **评审方漏掉的一点**【原件+源码】：`UiTreeBudget.repair`（`:62-96`）裁尾后**不重新派生**保留节点的 view，`nameComplete/detailComplete` 仍是按未裁剪的全集算的。实施方在 §6.2 里实际观察到 `e412316540` 就是这样。B 会让接近上限的稠密页更常触发裁尾，从而更常暴露这个旧缺陷。应当和 B 一起修：裁尾后重派生受影响的祖先，或者把它们标为不完整。

**结论：评审方对（不必等稀疏）**，但验收里要加上"裁尾后派生是否一致"。

### P17 把紧凑正文、find、指引提前

- 【原件】实施方把它们放到第 14、15 项的理由（主稿 :297）是"消费稳定的目标、观察、输入与交付合同"，也就是依赖名字规则（第 6、8 项）、稀疏记录（第 4 项）和结果合同（第 10 题）。
- 判断：**紧凑正文**（不再逐节点打印动作数组，占输出字节的 51–60%，F26）是主机端呈现，和名字、目标规则几乎无关；**find 默认搜全部已采节点**只是改一个过滤默认值。这两项先做不会返工，还能马上拿到效率对照。**结构化 View 文件 / diff-from** 依赖稀疏记录和 NAME，先做就要返工。**首页指引**更不能先做：第 2 项（显式 open）、第 10 项（`type --via`）、第 13 项（通知）都会改命令面，先写首页等于写两遍。
- **结论：部分对。** 两全的顺序是：早关文件写端、缺陷修复、紧凑正文、find 默认、结果合同（若已裁决）、效率对照基线 → 会话、表示、身份 → View 文件与 diff → 最后才是首页指引。

### 其余

- **P4**【源码，沿用 F3/F4，未独立重读 InputDispatcher】：`windowAcceptsTouchAt` 只用显示变换，可触摸区域本来就在逻辑显示坐标里。若 F4"显示变换只有平移、轴向缩放、90° 倍数旋转"成立，C 的非保矩形分支就是死代码，B 更简单。同意不问用户。
- **P6**：与实施方推荐一致，同意。
- **P9、P10**：同意。用 Fresh 单节点重取当前边界，比整份签名更直接。
- **P16(b)**【源码】`ui_client.py:1001-1007` 属实，而且 batch 在同一情况下保留结果并写 `cleanupError`（`ui_batch.py:378-386`），两边不一致，支持把它列为缺陷。
- **P18**：未复算。

---

## 二、评审方最该改的 5 处

1. **P8**：不要整体拒绝新接口。只保留 `queryIntentComponents`（`@SystemApi(MODULE_LIBRARIES)/@TestApi`，不是普通隐藏接口），用它做**按预期接收方的发送前别屏预拒**。只按创建者预拒，在跨包 PI 上拦不住 `ActivityStarter` 把主屏任务 `reparentToTargetRootTask` 搬到 VD；事后给 UNKNOWN 救不回来。
2. **P11**：补齐读命令豁免、"请求的"按 `explicit_delivery` 口径、done 且 exit≠0 的"勿重发"文案、进程 exit 为准；删掉"A 内部矛盾"这条论据（batch 的 exit 对应的是它自己的 failed）。
3. **P14**：主机端补默认值只能用在视图层，raw、保存和传出必须用稀疏原文，否则 `ui_client.py:1204` 会在产品里重现稠密展开的成本。
4. **P7**：只在"unknown listener"拒绝时重新注册（AOSP 对同一 Binder 不去重）；status 保留一行"登记可能仍在"。
5. **P13/P3** 写准缺陷：E9 布局不兼容会**耗光整个采集预算**再退化成只剩根节点（`CollectionNodes.kt:130`）；B 落地要同时修裁尾后不重新派生 view 的问题（`UiTreeBudget.repair`）。

## 三、实施方主稿里评审方漏批的问题

1. **第 7 题 B 自己排除了"按接收方预拒"**（主稿 :189），把 getter 的主要安全价值丢掉了；同时把 `getIntent` 和 `queryIntentComponents` 并列为必需，其实只需要后者。
2. **第 3 题 B 夸大了 IC 选区的"当前 editor 定位"**：IC 选区受光标前后 128 单位窗口限制（`CurrentEditorInput.kt:169`），主稿没说。
3. **"B 必须等稀疏"**（主稿 :78、:249）论据不成立：最终裁剪在派生之后（见 P13）。
4. **第 11 题 A 写"其余本机 Stop"**，但正常 Stop 会被 `requireIdle` 挡住，实际只能强制结束。
5. **F21 本地截止缺陷之外，E9 的布局校验失败会等到采集截止**，主稿第 1 题没有列出（与 P16(d) 相关，但严重程度不同）。
6. 【推断，供实现参考】Compose 的 IC commit 若取回的光标前文本短于请求长度，说明已到文本开头，可以推出绝对 offset，从而在短文本框上拿到更强的 DONE 证明；但 `getTextBeforeCursor` 的契约允许返回少于请求的字符，所以只能作为启发式，不能当作证明。双方都没讨论，优先级低。

## 四、未能核实

- 跨包通知 PI 真实触发任务搬迁【需设备】；各类常见通知的 PI 目标分布（没有逐 App 实测）。
- 用输入注入唤醒在熄屏和锁屏下的实际效果【需设备】。
- 旧稠密页裁尾时最后一个窗口是哪个（窗口排序没有核）。
- F4"显示变换恒保矩形"我没有独立重读 SurfaceFlinger 源码，沿用了草案的结论。
- androidx `getTextBeforeSelection` 的实现不在本地文件里，第三节第 6 条是推断。
