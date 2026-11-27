# 任务 120：启动、动作后观察、落点和屏幕——行为链深审（`409e853`）

## 结论（先读这里）

1. **确定缺陷，实测 5/5**：Chrome 新标签页上 `fill <首页假搜索框> URL --submit` 每次都报 UNKNOWN、submit 没发。原因是 fill 校验只认原节点，原节点一隐藏就立即判"不可观察"并跳过 submit（`UiActions.kt:868-877`、`:659`），可同一回执附的新树里，真正的地址栏已经聚焦，值也正好是请求值。每次多花 1～2 条命令（snapshot + ime）。
2. **确定缺陷，实测 3 次**：深色主题开关 `check/uncheck` 报 UNKNOWN、退出码 3，同一回执的新树里开关已经是目标状态。原因是主题切换重建了 Activity，窗口和 sourceId 都换了，产品只能从原节点句柄读值，读不到就报 `rootUnavailable`，这是终局状态（`UiActions.kt:881-882`）。在同一命令内用新树下确定结论有条件，但要另立"继任节点"规则，有误判风险，需要用户定（问题 Q1、Q2）。
3. **确定缺陷，实测 2/2**：`fill` 发出 focus 之后，确认输入焦点的等待用满整个请求预算（`awaitInputFocus`，`UiActions.kt:1063-1071`），动作后观察一份也拿不到。回执是 UNKNOWN、`timing=before-action`，用时约 5 秒。`open` 这一轮已经给观察留了时间，fill 的焦点步骤没留。
4. **延迟缺陷，有计时证据**：结果画面没有应用窗口时（例如通知栏展开），`open`、`click` 等"等首棵树"的命令会一直采样到命令时限才停（`UiPostAction.kt:39-44`）。实测 `open` 被通知栏盖住时用了 4.9 秒，时限 5 秒；同组不受影响的 `open` 是 2.2～2.7 秒。在通知栏里点快捷开关，按源码推断会同样等满时限（需设备）。
5. **本轮引入的小矛盾**：显式 `--window` 的窗口消失后，等首棵树的命令会继续在默认范围采样，但 `reportAbsentWindow` 最后无条件写入 `settle=changed`，把刚算出的 loading/unsettled 覆盖掉（`UiPostAction.kt:146`）。
6. **误导性措辞**：没有基准时，即使已经连续两份相同的完整样本，`settle` 也报 unsettled，主机端写成"page may still be transitioning"。后台屏题 5/5 次的首条 `open` 都是这样。
7. **`open` 遇到"立刻弹权限框"仍报 UNKNOWN 并用满预算**：落点只认任务的顶层包。实测 `r7-new-07-retry1#24` 中，任务表里唯一可见的任务顶层是 permissioncontroller，目标任务不在可见列表里。这一轮已经能带回对话框的树（比上一轮好），但退出码 3 和约 4.9 秒的耗时还在（问题 Q3）。
8. 截止时间各组合、`UiBudget` 各出口、后台屏移入移出、显示偏好与显式 `--display`，逐条核对后没有发现矛盾（见第五节）。本轮 `UiPostAction.kt` 净增约 200 行，其中约 90 行的"导航补全规则"重复了 `UiEffect` 和 `ObservedChange` 的节点比较，收益证据薄（第六节）。

说明：命令编号统一按 `cli-index.json` 从 0 开始计数（`#N`）。协调方说的"第 9、24 条""第 10 条"是从 1 开始的计数，分别对应 `r7-new-07#8`、`#23` 和 `r7-new-07-retry1#9`。证据路径前缀 `EV = S/r8/ev/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`；源码均为 `409e853`，前缀 `android/app/src/main/java/com/rainng/androidctl/`。我的中间脚本和清单在 `S/agents14/120/`（`all2.txt` 是 593 条动作命令的耗时和首行）。

---

## 一、整条链（文字图）

**入口**：`UiBackend.execute`（`UiBackend.kt:108-152`）先建 `UiBudget(deadline)`。设备端截止 `deadline` 取"请求截止减 min(500 ms, 总长/5)"；会发布观察的命令再按上一份采集的大小改写一次（`reserveDelivery`，`UiBudget.kt:27-31`）。执行线程做事；响应线程最多等 `responseRemaining()`，到点就交 `run.response(expired=true)`。

**派发前**：节点动作走 `UiRefRepair.perform` → `withOriginalTarget`，再检查范围、读原节点（`readOriginal`）、等原节点就绪（`awaitAction`/`awaitInputFocus`，用整个预算，这时还没发出，没问题）。系统动作（`SystemActions.perform`）在派发前取 `defaultBaseline`，要求是完整、同一连接、同一屏、默认范围的当前样本，否则为 null（`UiSnapshot.kt:63-72`）。节点动作的基准一律是 `snapshots.cached()`，也就是引用所在的那份样本。

**派发**：`invokePlatform` → `ui.dispatchInput` → `run.begin`，从这里起 `dispatchStarted=true`。open 是 `ActivityLaunch.launch`：先 dispatch，再 `startActivity`，拿到 launchCode。

**派发后的本地确认**（只看原节点）：
- 值类动作（check/uncheck/fill/range）：`awaitValue` 只读原节点句柄。读到不可见（fill）、读不到或抛异常，立即返回 `terminal=true`（`UiActions.kt:881-882`）。
- fill 的 focus：`awaitInputFocus`，用整个预算（第三节 D3）。
- reveal：`awaitReveal`，用到 `start + settleMs`。
- open：`confirmLanding` 轮询任务表到 `firstCutoff = deadline - max(settleMs, 1500) - 250`（`ActivityLaunch.kt:213`）。

**动作后观察**：`UiPostAction.settle`（`UiPostAction.kt:61-136`）。
- `PostActionWindow` 维护四个量：观察截止 `until = min(operationUntil, start + settleMs)`；单次读取截止 `collectionUntil`；重试截止 `retryUntil`；能否结束 `canSettle`。"等首棵树"的命令是 open、click、long-click、action、带引用的 key、`fill --submit`。对这些命令：还没有可用树时，单次读取可以用到操作截止；还没有应用窗口时，等待和重试都延到操作截止，应用窗口一出现就从那一刻重新给 1.5 秒。
- 每次采样：`beginSample` 先等到距上次开始满 250 ms；单次采样上限是 3 秒、观察截止、设备截止三者中最早的（`UiBudget.kt:64-82`）。
- 分类：`SettledSamples.accept/classification`（`UiPostAction.kt:265-298`）。连续两份相同的完整签名才算稳定；有 loading → LOADING；稳定且不同于基准 → CHANGED；稳定、等于基准、中途也没变过 → UNCHANGED；其余都是 UNSETTLED。循环停止条件是：没有基准时稳定，或者 CHANGED，或者到截止，或者估算下一份采样赶不上（`samplingStopped`）。
- 报告：`report()` 再叠三层改写。(a) 系统动作有基准、却整段没看到变化 → UNSETTLED 加 `noChangeObserved`；(b) 导航类命令：最终树有 loading 文字或进度条 → LOADING；"只有几何变化"或"只有目标焦点/动作/边框变化"的 CHANGED 降为 UNSETTLED；(c) 窗口消失 → 无条件写 `changed`。`observedChange` 另由 `ObservedChange.compare(before, final)` 独立计算；节点动作再由 `UiEffect.apply` 比原节点字段。

**落点**（只有 open）：`observeLanding`（`ActivityLaunch.kt:223-244`）。主屏已确认 → 直接返回。未确认 → 观察截止留 250 ms，观察完再 `confirmLanding` 到设备截止。后台屏已确认 → 再查一次搬入和"出现在别的屏"（`inspectConfirmedLanding`）。

**回执**：`finish` → `UiPublication.finish` 附当前样本（`snapshotBeforeAction` 标明是否为动作前的样本）。主机端 `change_summary`（`ui_output.py:446-518`）的规则是：`settle` 为 unsettled 或 loading 时，它压过 `observedChange.state`；本地校验为 differs 或 unobservable 时改写成 unknown。

---

## 二、协调方点名的两类 UNKNOWN：确切路径，以及能否当场下结论

### D1 深色主题 check/uncheck 报 UNKNOWN（实测 3 次）

**证据**：
- `EV/efficiency/r7-new-07/…#23`（`uncheck e241580843 --settle 3s`，2860 ms）。首行 `uncheck: UNKNOWN; Current window root is unavailable`；`Resolved` 的原开关在窗口 142；`Selected` 是行 `e241580838` 的 `click`。附树的窗口是 144，`switch e241580874 … checked="false"`，正是目标状态。
- `r7-new-07-retry1#9`（`check e241581262`）同理：原开关在窗口 153，附树的窗口 155 里开关 `checked="true"`。
- `r7-new-07#8`（`check e241580321`）：同样的 UNKNOWN，但附树 `scopeUnknown`（读的时候还没有应用窗口），这一次确实没有可用的新树。下一条 `recents`（`#9`）的树显示开关已是 ON。

**源码路径**：
1. 开关本身不可点，行可点。`performAction` → `awaitAction` 判定"未就绪"后调用 `captureSuggestedRow`，拿到 `RowSelection` → `rowCheck`（`UiActions.kt:1039-1043`、`:253-254`、`:568-613`）。
2. `rowCheck` 点行：`invoke(..., UiAction.CLICK)`，平台返回 true，记成 UNKNOWN "Checked-state verification pending"（`:594-598`）。
3. `awaitValue(run, checkNode, control, action)`（`:600`）只用原开关的句柄 `checkNode` 和原窗口 id 读：`readOriginal` → `FreshNodes.read`。深色主题会让 Settings 的 Activity 因配置改变而重建，旧窗口的根已经取不到，`FreshNodes.kt:24` 抛出 `rootUnavailable "Current window root is unavailable"`；如果旧窗口已经从列表里消失，则在 `requireOperatingWindow` 抛 `changed`。
4. `awaitValue` 的 catch 不区分错误种类，一律返回 `Verification(verificationError=…, terminal=true)`（`:881-882`）→ `verifiedResult` 把 UNOBSERVABLE 记成 `Outcome.UNKNOWN`（`UiVerification.kt:135-141`）。
5. `settle(..., checkedIdentityEnded = true)`（`:603`）只取一份样本就停。如果那份是 `rootUnavailable`，还会再做一次 `observeAfterRootLoss`（`UiPostAction.kt:93-95`、`:156-170`），这两次正是拿到新窗口 144/155 树的来源（`rootRecoveredWithinRequest=true`）。
6. `UiEffect.apply` 按"同窗口、同 sourceId"找原节点，找不到 → "Original target is not identifiable in the final observation"（`UiEffect.kt:46-51`）。主机端的规则是"本地校验 unobservable 压过整页比较"，所以首行是 `change=unsettled…`，退出码 3。

**能不能当场给结论**：条件在同一命令里已经齐了：新树就在 `run.observation` 里，`UiAction.CHECK/UNCHECK` 的判定也只是读 `checked` 字段。但产品的身份规则 `UiSemantics.identity` 包含 `sourceId` 和 `windowId`（`UiSemantics.kt:321-334`），Activity 重建后，按定义这已经是"另一个节点"。要下结论，就得新立一条"继任节点"规则。可以这样写窄：原窗口已从目录消失；新树与原树同包、同显示屏；新树里恰好一个节点与原开关的角色、资源 id（这里是 `switch_widget`）、行标签和行链位置全部相同；它的 checked 值可读。

风险：
- (a) 点击本身没生效，但新页面上同 id 的开关默认就是目标状态。例如点行进入了一个新页面，新页面里的开关恰好同 id、同行标签，结论就会是假的 DONE。
- (b) 列表页里每行的开关都用同一个 id。靠行标签区分；行标签取不到完整值时，规则必须不命中。
- (c) 这是给"原目标核验"这条历史决定开口子。原则上只要规则不命中就保持 UNKNOWN，不会比现在差，但每多一条命中路径，都需要设备上的对照测试。

→ **问题 Q1**（见第七节）。

### D2 Chrome 新标签页 `fill … --submit` 报 UNKNOWN（实测 5/5）

**证据**：`r7-new-08#3`、`r7-new-08-retry1#3`、`r7-new-09#3`、`r7-new-09-retry1#3`、`r7-new-10#3`，五次都是：
`fill: UNKNOWN; change=unsettled at windows,content,eXXX:visible,bounds…; focus=done, sent=true; fill=UNKNOWN, reason="The original filled field is no longer visible", sent=true, verification="unobservable"; submit=not sent`。

以 `r7-new-08#3` 为例，附树里 `textbox e241582113 value="https://tfl.gov.uk/plan-a-journey/" … +ime="Enter" id="url_bar" focused`；原目标 `e241582029 "Search Google or type URL"` 是信息流里的假搜索框。之后的命令序列是 `#4 snapshot`、`#5 ime e241582113`（另两题是 `ime` 加 snapshot），每题多 1～2 条命令。

**源码路径**：
1. `FillSequence.focus`：原假框能拿到输入焦点，所以 focus=done（`UiActions.kt:714-735`）。
2. `fill()`：`awaitFocusedFill` 读原节点仍然就绪，接着在原节点上执行 SET_TEXT，平台返回 true（`:743-757`）。Chrome 随即把编辑交给地址栏 `url_bar`，假框隐藏。
3. `awaitValue(..., UiAction.FILL, until)`：第一次读原节点就发现 `!visible`，立即返回 "The original filled field is no longer visible"（`:868-869`）。状态是 UNOBSERVABLE，`:877` 的 fill 特例让它不重试就返回。
4. `verifiedResult` → UNKNOWN（`:775`）；`performSteps` 里 `progress.step(FILL)` 不是 DONE，所以 submit 不发（`:659`）。
5. `finish` → `settle`（`fill --submit` 等首棵树）拿到新树；`UiEffect` 报原目标的 visible 和 bounds 变了。

**能不能当场给结论**：分两层。
- "值已经写进了某个输入框"：新树里同一窗口恰好有一个输入聚焦、可编辑的节点，文本与请求值完全相等，而它在基准里不是这个值。这可以作为一条事实写进回执，不改 fill 的结果。风险很小，只是在陈述新树里看得到的东西，不冒充原目标。
- "替调用者把 submit 发到这个新节点"：等于把动作扩到一个调用者没有准入的节点上，违背"只作用于原目标"的既有决定。在 Chrome 地址栏上它正是调用者想要的，但在别的 App 里，"原框隐藏、焦点转移到另一个框"也可能是弹出了别的编辑器（例如搜索建议页的另一个输入框），这时发回车会触发调用者没看到的提交。

→ **问题 Q2**。

---

## 三、其余确定缺陷

### D3 fill 的焦点确认吃掉整个预算，动作后零观察（实测 2/2）

- **证据**：`r7-new-06#13`、`r7-new-06-retry1#12`（Collabora Calc 单元格），用时 4990 和 4983 ms。首行 `fill: UNKNOWN; code=notReady; … No post-action observation; cached; dispatchStarted=true; timing=before-action; focus=UNKNOWN, reason="Original target is not ready for input focus: target is not input-focused", sent=true`，并带 `snapshotError="No post-action snapshot within the budget"`。下一条 `snapshot`（`#14`）显示 `keyboard="shown"`，说明焦点动作其实已经起了作用。
- **路径**：`focus()` 发出 ACTION_FOCUS 后调用 `awaitInputFocus`（`UiActions.kt:730`）。它的循环条件是 `run.budget.remaining() > 0`（`:1064`），一直等到设备截止才抛 notReady → `progress.fail(FOCUS)` → `finish` → `settle(start = now)`。这时已过截止，而无 submit 的 fill 不在等首棵树的名单里，`canStart` 为假，一份都不采。
- **影响**：调用者多花约 5 秒加一条 snapshot，回执里看不到焦点之后的画面。同一类问题也出在 `submit()` 的 `awaitInputFocus`（`:796`）：fill 已经发出，如果之后焦点丢了，同样会吃满预算（源码推断，未见实例）。
- **建议**：已发出动作之后的本地确认，截止统一为"发出时刻加 settleMs"（与 `awaitValue`、`awaitReveal` 一致），剩下的时间留给观察。代价：要 1.5 秒以上才给出输入焦点的 App，fill 会在 focus 步骤停下，报 UNKNOWN，由调用者重试。现在这种 App 本来也是 UNKNOWN，只是更慢。
- **验证**：脚本化时钟测试 `FillSequence`；在 Calc 上复测，回执应带焦点之后的新树，总耗时应约为 2～3 秒。

### D4 没有应用窗口时，等首棵树的命令等满时限

- **证据**：`r7-new-07#4 notifications` 之后，`#5 open --action android.settings.DISPLAY_SETTINGS` 用时 4893 ms，附树是 systemui 的通知栏（只有系统窗口 17 和边缘窗口 16，没有应用窗口）。同组的 `#2`、`#3` 次 open 分别是 2739 和 2259 ms。
- **路径**：`PostActionWindow.observed` 看到目录里没有 `type=application`，就置 `waitingForApp=true`，于是 `canSettle=false`，`retryUntil` 和 `collectionUntil` 都等于 `operationUntil`（`UiPostAction.kt:39-51`）。即使连续多份样本完全相同，也不会停。
- **影响**：结果画面本来就没有应用窗口时，例如通知栏或快捷设置展开、系统对话框，`click`/`open`/`long-click`/`action`/带引用的 `key` 都会用满时限（默认 5 秒；调用者给 `--timeout 30s` 就是 30 秒），最后报 "unsettled"。在通知栏里 `click` 快捷开关的情形没有实测，按源码推断如此（需设备）。上一轮决定文件第六节第 4 处只考虑了"后台屏点按钮关掉最后一个界面"的代价，没覆盖通知栏。
- **建议**（选项见 Q4）：没有应用窗口、但已经连续两份相同完整样本时，允许结束；或者给"等应用窗口"设一个上限，例如 settleMs 的两倍。代价：真正慢启动、要几秒后才出窗口的 App，会更早报 unsettled。

### D5 窗口消失后继续采样，结果又被无条件改写

- **路径**：本轮起，等首棵树的命令在 `absentWindow` 之后会继续在默认范围采样（`UiPostAction.kt:77`、`:100` 的 `!firstTree`）；但 `reportAbsentWindow` 最后执行 `data.put("settle", "changed")`（`:146`），覆盖掉 `report()` 算出的 LOADING 或 UNSETTLED。
- **影响**：显式 `--window`/`--in` 的点击把对话框关掉，下面的页面还在加载时，首行不会有 "page may still be transitioning"，只剩下进度条触发的 `loading detected`；加载文字类的 loading 不会提示。少见。
- **建议**：只在 `report()` 的结果是 UNCHANGED 或 UNSETTLED、且没有 loading 时才写 changed；或者干脆不改写 settle，只留 `absentWindow` 和 notice。代价：一两行代码加一个测试。

### D6 没有基准时"已稳定"报成"可能还在过渡"

- **证据**：`r7-v1-candidate-01…05#2 open com.android.contacts`，五次首行都是 `change=unsettled (page may still be transitioning)`，附树是静态的空联系人页。
- **路径**：`classification()` 在 `baseline == null` 时一律返回 UNSETTLED（`UiPostAction.kt:293`），哪怕 `stableWithoutBaseline()` 已经成立、循环已因此提前结束（`:98`）。测试 `UiOutcomeTest.kt:574-582` 专门断言这一点。主机端把它渲染成"可能还在过渡"（`ui_output.py:458-462`、`:514-517`）。
- **影响**：措辞与设备掌握的事实相反。后续 `find` 还会因此多采一次（`UiSnapshot.actionFinished`，`:113-115`）。调用者在实测里没有因此出错，代价主要是误导和那一次多余的采集。
- **建议**（Q5）：主机端在没有基准时（`observedChange.reason` 以 "No before observation" 开头），把这句改成 "no baseline; observation stable"；或者设备端用一个事实字段表达"稳定"。

### D7（设计取舍）open 立刻弹权限框：UNKNOWN 并用满预算

- **证据**：`r7-new-07-retry1#24 open com.android.deskclock`，4943 ms。`landing={"state":"unknown",…"matchingTaskCount":0,"visibleTasks":[{"taskId":123,…"package":"com.android.permissioncontroller",…"GrantPermissionsActivity"}]}`，附树是 "Allow Clock to send you notifications?"。可见任务只有这一个；如果权限框是独立任务，下面的时钟任务也应可见，所以权限框多半是在时钟自己的任务顶层。上一轮记录 92 的"平台前提"在这里得到了间接实测。
- **路径**：`activityLanding` 只按 `topActivityInfo` 的包名和 UID 匹配（`ActivityTasks.kt:110-116`）→ `confirmLanding` 一直轮询到 `firstCutoff`，观察 1.5 秒，再轮询到截止（`ActivityLaunch.kt:237-243`）。
- **对比**：同一 App，如果首次轮询先于弹框，就是 done，而且按决定不会再被改报。所以结果取决于时序竞争。Android 13 以上，凡是首启就申请通知权限的 App 都会碰到。
- **选项见 Q3**。

### D8（设计意见）后台屏"出现在别的屏"在第一次轮询就判定

`confirmLanding` 一旦看到 `elsewhere` 就立即返回 UNKNOWN（`ActivityLaunch.kt:375-381`），不等目标屏上的任务出现。`activityLanding` 只要目标屏上有匹配就判 observed，所以只有在"别的屏上恰好有目标包的顶层活动"（例如主屏上别的 App 任务里开着同包的 Custom Tab），而新任务还没在后台屏上变成可见时，才会误判。罕见，需设备确认可见性出现的时机。建议：轮询到截止都没看到目标屏上的任务，才报 elsewhere。

---

## 四、小的不一致（可选）

- `usesCurrentBaseline` 把 `type --via keys` 一律算成系统动作（`UiPostAction.kt:205-211`），带引用时也一样；但 `key REF` 不算，`waitsForFirstTree` 也不含 `type REF`。带引用的按键输入因此会走 `noChangeObserved` 这一支，基准用的却是引用所在的缓存样本。影响很小，建议按"是否带引用"对齐。
- 节点动作的基准是引用所在的旧样本，可能已经很久。成功回执（concise）不显示 `baselineAgeMs`，所以在有时钟、滚动条一类常变内容的页面上，"点了没反应"也可能报 `change=changed`。这是历史决定"节点动作沿用现有 before"的结果，指引已写明变化不证明因果。不算缺陷，记作已知代价。
- `reveal` 和值校验（check/range）先用掉 settle 窗口，`settle` 用的是同一个 start（`UiActions.kt:153-158`、`:431-447`），所以校验一直不匹配时，动作后零采样，回执附的是动作前的树（会标 `timing=before-action`）。恰恰是最需要看页面的时候没有新树。可以与 D3 一起，给校验和观察分配不同的截止。

---

## 五、核对过、没有问题的地方

- **截止组合**（源码逐条推演）：
  - `settle 0` 时，等首棵树的命令仍允许一次读取（`canStart` 的 `firstAttempt`），其余命令零次，与指引相符。
  - 首份不可用但有应用窗口时，不再放宽，符合新-5（上一轮的第 5 个新问题：读取失败后要不要重试到命令时限）评审方推荐的 A。
  - 单次 3 秒上限只由 `beginSample` 一处计算。
  - `canSample` 早停要满足"上次采样耗时 ≤ 剩余时间"，且只在可用树之后才估算；没有应用窗口时不估算，避免误停。
  - open 的 `firstCutoff` 在 settleMs 很大或时限很短时会小于当前时间，但 do-while 至少查一次。
  - 观察截止的 250 ms 余量只扣在"未确认"和"后台屏"两种情形；主屏已确认不扣。
- **`UiBudget` 各出口**：
  - 构造时预算已尽 → timeout，在 execute 之前，回执是 not sent。
  - `beginSample` 抛 `snapshotBudget` → 由 `SettledSamples.observe` 记作 missing，循环继续，由 `canStart` 判定停止。
  - `check()` 区分局部截止和请求截止的文字。
  - `checkDelivery` 抛出只发生在交付阶段，主结果已经 `remember`，不会被改写。
  - `pendingRootReadNotice` 在响应超时时由 `run.response(expired)` 写入（实测 `r7-new-02#3` 显示 "still pending after 4579 ms"）。
  - 548 条动作命令中，没有一条在默认 5 秒时限下超过 5.2 秒（含主机进程启动）。
- **异常路径资源**：`prepareCollection` 失败时回收窗口；`readOriginal` 在 finally 里回收；`UiBackend.execute` 的 finally 总会 `sessions.finished`。没有发现泄漏。根读取挂起时执行线程仍被占着，但下一条命令（`r7-new-02#4 back`）能正常运行。
- **后台屏任务**：`recordMovement` 只统计"之前在别的屏、现在在目标屏"的 user0 任务。授权搬入和外来搬入分开计入 history，外来搬入优先于落点（`unexpectedTaskMove`），已确认的落点查询失败只附 `landingError`，不改写结论（`inspectConfirmedLanding`），都符合上一轮第 8 问的 B 和新-6 的推荐 B。
- **显示偏好与显式 `--display`**：`Sessions.prepareChange` 的取值是 `target ?: 当前租约的屏 ?: 主屏`（`Sessions.kt:113`）。显式请求一个不同种类的屏且没有 `--replace` 时，由 `checkTargetChange` 拒绝；同种类时复用，不走 preflight；`takeover` 必须显式给 `--display`（`app.py`）。三者没有矛盾。
- **"连续两次截断早停"** 只存在于 `wait`（`UiCondition.kt:463-470`），动作后观察本身有 1.5 秒的上限，不需要它。两者没有冲突。

---

## 六、重复机制、只为测试存在的分支、本轮差量评价

**同一事实算多遍**："页面变没变"由四处各自计算：
1. `SettledSamples` 的签名比较（`UiSnapshot.signature`）；
2. `ObservedChange.compare`（目录、选择、内容三个维度）；
3. `UiEffect.fieldDifferences`（原目标的字段）；
4. 本轮新增的 `onlyNavigationChanges`/`removeNavigationChanges`/`lostFocus`（`UiPostAction.kt:361-447`），又做了一遍逐节点、逐字段的 diff，外加 `onlyGeometryChanged`。

主机端最后再按优先级合并（`settle` 压过 `observedChange`，本地校验压过两者）。四套结论可能互相矛盾：例如 `observedChange=changed`、`settle=unsettled`、`effect=changed`，首行显示 `change=unsettled at content,eREF:focused,actions`。

**`UiPostAction.kt` 本轮约 +200 行，逐块**：

| 块 | 行数（约） | 解决什么 | 值不值 |
| --- | --- | --- | --- |
| `PostActionWindow` 与循环改写 | +40 | 首棵树放宽、等应用窗口（上一轮第 13 项，用户已定） | 值，有上一轮 19 次 open 无树的实测依据；但见 D4 和 D5 的副作用 |
| `observeRoots` 与 `rootUnavailableObserved/Recovered` | +25 | 回答新-5 要数的"根读不到、后来是否恢复" | 作为收集证据的手段可以接受；这两个字段每次出现都上首行，等第八轮定了新-5 后可以删 |
| `noChangeObserved` 与 `usesCurrentBaseline` | +20 | 系统动作有基准却没看到变化时，改用更准确的措辞 | 值，代价小 |
| `onlyNavigationChanges`、`removeNavigationChanges`、`lostFocus`、`directoryWithoutBounds` | +90 | "点了只换了焦点"不报 changed | 证据薄：实测首行 `at content,eREF:focused,actions` 一类只有约 5 次，且结论只是把 changed 降成 unsettled。规则精确到动作 id 的成对替换，维护成本高。删除的能力代价：这类点击会报 `change=changed`，调用者需要看 effect 字段，而 effect 本来就会列出 focused 和 actions |
| 删掉 Maps 专用文字，loading 定义并入 `UiSemantics` | −20 | 上一轮决定 | 值 |

总体上，循环本身更清晰了（截止集中到一个对象），报告层更复杂了（三层改写，加上 D5 的覆盖）。

**可删建议**（每条附能力代价）：
- `onlyNavigationChanges` 一组（约 90 行）。代价见上表。
- `rootUnavailableObserved/rootRecoveredWithinRequest` 的首行呈现。新-5 定下来之后，可以只留在 JSON 里。代价：调用者少一个"App 根暂时读不到"的提示，但 `snapshotError` 仍在。
- `SettledSamples.report` 的 `completeWindow` 参数只把"窗口未满时的 UNCHANGED"降成 UNSETTLED。系统动作已经有 `noChangeObserved` 处理同样的意思，节点动作也几乎总能满窗。可以评估删除，删之前需要测试数据。

只为测试存在的分支：`SettledSamples.observe` 的 `budget = null` 默认值只有测试和 `observeAfterRootLoss` 之外的调用者在用；生产调用都传了 budget。可以改成必填参数。

---

## 七、需要用户定的问题

**Q1 主题切换一类"原窗口被重建"的值校验，要不要用新树里的继任节点下结论？**
- A：维持 UNKNOWN（现状）。好处：不碰"只核验原目标"的决定。代价：深色主题、语言、字号一类开关每次都是退出码 3，调用者要多读一次。
- B：维持 UNKNOWN，但回执写一条事实，例如 `successorCandidate={ref, checked}`，并在 reason 里说"原窗口已被重建，附树里同位置开关为 X"。好处：不改结果，又能让调用者少一条命令。代价：加一个字段，需要规则只陈述事实。
- C：按第二节的窄规则命中时报 DONE，`verificationSource="successor in post-action tree"`。好处：退出码正确。代价：存在假 DONE 的风险，需要设备对照测试。
- **推荐 B**。

**Q2 Chrome 首页假搜索框 `fill --submit`：**
- A：现状。
- B：回执写出"请求值出现在聚焦输入框 eXXX"的事实，并给出具体下一步，例如 `ime eXXX`，submit 仍不发。好处：5/5 的情形能省一条 snapshot。代价：加一条事实规则。
- C：B，另外在继任节点上自动发 submit。好处：一条命令完成。代价：把动作扩到调用者没有准入的节点上，在别的 App 有误提交的风险。
- **推荐 B**；也可以在指引里写明"Chrome 首页先 click 假框再 fill 地址栏"，零代码。

**Q3 open 立刻弹权限框：**
- A：现状（UNKNOWN，用满预算）。
- B：落点允许"目标屏上可见任务的基础活动属于目标包，而顶层是 permissioncontroller"，报 done 并附 `coveredBy`。代价：多一个特例包名。
- C：只缩短预算：看到这种形态就停止轮询，仍报 UNKNOWN。好处：省约 2 秒。代价：结果不变。
- **推荐 B**。这种形态在 Android 13 以上首启时常见，而且当前结果取决于时序竞争，同一情形可能是 done 也可能是 UNKNOWN。

**Q4 没有应用窗口时的等待上限（D4）：**
- A：现状。
- B：没有应用窗口、且连续两份相同完整样本时，允许结束。
- C：给"等应用窗口"设固定上限，例如 2 倍 settleMs。
- **推荐 B**：真正在启动中的界面不会连续两份相同，影响面最小。

**Q5 没有基准时的稳定（D6）：**
- A：现状。
- B：只改主机端措辞。
- C：设备端新增一个 settle 值。
- **推荐 B**，零协议改动。

---

## 八、需要设备确认的事项

- 通知栏里 `click` 快捷开关是否真的等满时限（D4）。
- 权限框是否位于目标 App 自己的任务（D7；目前的证据是间接的）。
- 后台屏 `elsewhere` 判定的时序竞争是否真实存在（D8）。
- D3 修复后，在 Calc 上的实际耗时。
