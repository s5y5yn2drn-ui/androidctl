# 独立复核：评审方报告的缺陷主张与事实依据（24）

复核对象：`/home/user/androidctl/review/androidctl-followup-route-review-3f45af6.md`（下称“报告”）§7.1 N1–N15，以及用户据以改判的事实依据 B1–B12、坐标接收者接入事实 C。
一手材料：候选只读克隆 `review-copy`（`3f45af6`，下文 Kotlin 路径省略前缀 `android/app/src/main/java/com/rainng/androidctl/`，主机路径省略 `androidctl/src/androidctl/`）；实施方包 `T=…/ultragoal/2026-11-11-next-improve` 内原始回执、树、日志、图；包内固定上游副本；本轮自行下载的 AOSP 文件（`agents5/24/aosp/`，标签 `android-14.0.0_r1`、`android-15.0.0_r1`、`android-16.0.0_r1`，来源 android.googlesource.com `?format=TEXT`）。
自写脚本均在 `agents5/24/`，用 `python3 -I` 运行，包内路径作参数传入。没有设备、模拟器、Android 构建，也没有重跑 pytest（环境缺 typer 等依赖）。

---

## 1. 摘要

**不成立或部分成立（最重要）**
1. **N1 部分成立**：签名确实按原序哈希动作数组，成对原件 830 节点只有次序不同也属实；但次序分组是“每个 worker 的第 1 次采集 vs 第 2 次采集”（与 E9/single 无关），原件只覆盖新 worker，没有覆盖“同一 worker 内新 UiAutomation 连接”；“页面真实变化后 settle 多采一次”无源码或原件支持。
2. **N7 部分成立**：两套语义属实，但 `wait --window` 缺席时并非“整树重采”，而是每轮只枚举窗口目录、因 `scopeUnknown` 重试到截止。
3. **N8 部分成立**：基线不完整时确实永不提前结束；但基线完整而“无变化”的动作同样耗满窗口（`UiPostAction.kt:63-68` 只在 CHANGED 或无基线稳定时提前退出），且“网页长页常见”在 Chrome 原件里没有出现（四份 4226 节点树全部 full）。
4. **N10 部分成立**：逻辑成立；原件中白花的是 settle 1,368 ms/4 次采样，tap RPC 1.422 s，“约 2 秒”是整条 CLI 子进程耗时 2.016 s。
5. **N13 部分成立**：静态可指认 22 个 POSIX 必败用例（`test_display.py` 20 个参数化实例 + `test_ui_batch.py:338` 2 个），第 23 个与 `workspace.py:127` 的关系我没能静态确认。
6. **N4 基本成立但有一处措辞偏差**：`notifications.md:59-61` 写了“without … display selection”，不是完全无文档；“落在主屏”是平台默认行为的推断，App 已有任务在别屏时可能不同。
7. **报告 §3.3 与 §7.2 D5 有误**：“其它清理失败不封”不成立——`UiBackend.closeError`（租约 UI 清理、连接失败后的断开失败）走同一个 `cleanupFailure` 门，同样封锁全部 UI 命令和会话切换且永不复位；“读取不受影响”也不成立——`snapshot`、`notifications list` 等只读 UI 命令同样被 `Sessions.kt:144` 拒绝，只有 `ui.read`/`status` 等查询方法不受影响。
8. **B4 部分成立**：图源与坐标资格共用一个 `ImageState` 和同一组失效触发，但 120 s TTL 只约束坐标输入，不约束 zoom；API34 软件模拟器上原生源就是 720×1280，zoom 没有分辨率增益，增益只在 MuMu（1080×1920）上存在。
9. **B1 补充**：产品 UI 命令面确实没有后台屏会话下操作通知栏的路径；但 `shell` 在后台屏会话可用，`cmd statusbar expand-notifications` 加 `input tap` 这类 shell 绕行理论上存在（不经节点、不受产品护栏），“根本走不通”要限定为“产品 UI 命令面内”。
10. **B6 细化**：“标题检查已存在”指的是**动作按钮标题**（`NotificationActions.kt:284`），通知本身的 `EXTRA_TITLE` 没有比对；创建者包与 UID 只在 action/open/reply 比对，dismiss 不比对。

**新事实**
- WindowInfosListener 的 JNI `nativeRegister` 忽略 `addWindowInfosListener` 的返回状态，权限被拒时 `register()` 静默返回空数组；`nativeUnregister` 返回 void，也忽略状态（A15 `android_window_WindowInfosListener.cpp:143-158`）。“初始数组为空”在常驻 worker 里每次都会发生（进程级 Reporter 在最后一个监听者注销时清空缓存），且无法区分“正常”与“被拒”。
- 通知 listener：同一 Binder 重复注册会**重复登记**（`ManagedServices.registerServiceImpl` 只 `mServices.add`，无去重、不抛异常），注销按 Binder+userid 一次删光；且 `linkToDeath` 使 worker 进程死亡时登记自动清除。

---

## 2. 逐条：A 部分（N1–N15）

### N1 动作次序进入设备端签名 —— 部分成立
**源码【源码】**
- `NodeFacts.kt:142-151`：`actions` 由 `node.actionList.mapIndexed` 按平台原序生成，没有排序（报告写 `:145`，在此范围内）。
- `NodeFacts.kt:15-39` `observationFields` 含 `"actions"`；`UiSnapshot.kt:104` `signatureFields = observationFields + …`；`UiSnapshot.kt:107-116` `signature()` 含 `contentSignature`；`UiSnapshot.kt:118-141` 逐节点 `signatureFields.map(node::opt)` 后整体 sha256，数组原序进入哈希。
- 全部消费者：
  1. `UiPostAction.kt:158,168,179,193-194`（SettledSamples：基线签名、样本签名、changed/CHANGED 判定）；
  2. `UiBackend.kt:368` → `VisualInput.kt:39-44`（`observed`：签名变即 `invalidate()` 截图/坐标资格）；
  3. `ObservedChange.kt:11-14`（动作回执 `observedChange.content`）；
  4. `ObservedChange.kt:17-23` + `UiDelivery.kt:112`（已发布基线的 `contentSignature`，供 `comparePublished`/wait 已发布基线比较）；
  5. 另有 `UiEffect.kt:67-71` 局部核值逐字段比较 `json(value).toString()`，`actions` 次序同样敏感（不是签名但同一缺陷面）。
**重算【重算，`agents5/24/n1_order.py`】**（键 = windowId+sourceId）
| 对 | 共同节点 | 只次序不同 | 多重集不同 |
|---|---:|---:|---:|
| a-e9-first vs a-single-warm | 4226 | 830 | 0 |
| b-single-first vs b-e9-warm | 4226 | 830 | 0 |
| a-e9-first vs b-single-first（两 worker 首采） | 4217 | **0** | 0 |
| a-single-warm vs b-e9-warm（两 worker 次采） | 4217 | **0** | 0 |
| preparation-page（准备 worker 首采）vs a-e9-first | 4217 | 6 | 0 |

含 16 与 64 的节点：两份首采均为 `64<16` 831 个，两份次采均为 `16<64` 841 个。结论：次序与采集方式（E9/single）无关，只与“该 worker 的第几次采集”相关，三个 worker 的首采一致、两个 worker 的次采一致【原件】。Chrome 构建器先加 64 后加 16（包内 `analysis-action-order/beta158-AccessibilityNodeInfoBuilder.java:357-365`），次采的 `16,64` 来自 Chrome 自身另一条更新路径，机制在 Chrome 侧，未在包内找到。

**因果链逐步核对**
- “首采与之后次序不同”：原件支持，但只覆盖**新 worker** 的首采。报告写“新连接后的首个快照”——租约清理会 `disconnect()` UiAutomation（`UiBackend.kt:480-493,503-519`），所以每次新会话都会新建连接；同一 worker 内新连接是否同样触发 Chrome 次序重置，原件没有覆盖【需设备】。
- “之后做任何动作，包括无效点击，都报有变化”：动作 `before = ui.snapshots.cached()`（如 `UiActions.kt:346`），若它是首采、动作后样本是次采，`signature != baseline` → `changed`，两次相同后 `CHANGED`（`UiPostAction.kt:179,193-194`），`observedChange.content` 亦为 changed【源码＋推断】。只对“首采之后的第一个动作”成立；第 3 次及以后采集是否保持次采次序，原件每 worker 只有两次，无法确认。
- “保留的截图失效”：成立（`VisualInput.kt:43`）【源码】。
- “页面真实变化后 settle 多采一次”：**无支持**。真实变化时 CHANGED 本来就要求连续两份相同样本，次序只在首/次采之间翻转，不会额外增加一份样本；报告没给出机制【源码】。

### N2 默认 `find` 只搜视口 —— 成立
- `ui_view.py:253-255`：非 `--all` 时 `visible is False` 的节点被丢弃（其可见子节点上提）；`ui_view.py:276-293` `find` 只遍历投影后的元素；`ui_view.py:703-704` 先 `project` 后 `find`。行号与报告 `236-237` 不符，正确是 `253-255`（报告的 236-237 落在 ref 局部命名段）。
- 提示：零匹配时所有元素按 `reason=find` 省略并合并为无 ref 的 omission，`read` 固定为 `snapshot --cached`（`ui_view.py:533-545,595-610`）；有 ref 的单个省略提示 `snapshot REF`（`:543-544`）。两者非 `--all` 时同样丢弃离屏节点。故“固定指向 `snapshot --cached`”只对零匹配/合并省略成立。
- 原件【原件】：Chrome 原型臂 `cli-index.json` 第 4 条 `find 'Addis Ababa' --context 2` → `matches=0`，输出 `omission {"reason":"find","nodes":84,"read":"snapshot --cached"}`；同一缓存树（RPC `31612e2f…/001-tree.decoded.bin`）4226 节点、105 可见，`Addis Ababa` 三个节点 `visible=false`（`agents5/24/n2_tree.py`）。随后第 5–8 条是 4 次 `scroll e953117431 down`，第 9 条 `click e953117600`。“matches=0 后连滚 4 屏”成立。

### N3 标注名额被只带上下文点击的节点占用 —— 成立（数字与措辞需修正）
- `ImageMarks.kt:117`（MAX 40）、`:145-155`（按 `nodes` 原序取 `publicTarget && refAvailable && visible` 前 40 个）；`UiSemantics.kt:55-56` `publicTarget`；`NodeActions.kt:11-35,44`：`ordinaryViewActions` 不含 `16908348`（accessibilityActionContextClick），所以任何带它的节点都是目标。主机 `ui_view.py:39-53,72-86` 同规则。
- 重算【重算，`agents5/24/n3_marks.py`】：Chrome 原型臂 13 份网页树中，前 40 名额里“只因上下文点击才算目标”的节点 **22–31 个**；真正可点目标（把 16908348 视作普通动作后仍是目标）未得到标注的 **9–29 个**。collector 原件 a-e9-first：29 个占位、15 个可点目标落选。报告的 “25–33 / 14–21” 范围偏窄，方向成立。
- 措辞：占位节点并非都是“文字节点”，角色包括 list、listItem、cell、heading、image、region 等容器。
- `--interactive` 作用弱：首份树可见 105 个节点中 84 个是目标【重算】，“不起筛选作用”应改为“只滤掉约两成”。

### N4 后台屏会话里 `notifications open` 在主屏启动 —— 基本成立
- `NotificationActions.kt:187-207`：Activity 型 PI 只设置 BAL 模式（`:209-231`），没有 `setLaunchDisplayId`；全仓库唯一 `setLaunchDisplayId` 在 `ActivityLaunch.kt:119`。
- `RpcMethods.kt:48-56` `requiresDisplay` 排除全部通知命令 → `UiBackend.kt:107` 不带种类期望；写入门 `NotificationActions.kt:124-128` 只检查主屏亮且解锁（`inputReady`），没有按会话目标拒绝。
- 文档：`notifications.md:59-61` 写“without tree, screenshot, Browser support or display selection”，没有说明会落在主屏。“无文档”应改为“文档未写明落点”。
- “在主屏启动”是平台默认（无显示参数、调用方在 display 0）的推断；目标 App 已有任务在别屏、singleTask 等情况可能不同【推断/需设备】。

### N5 通知注销失败后封锁、远程无法恢复 —— 成立（并比报告更宽）
逐跳【源码】：
1. `NotificationOperations.kt:20-21` `cleanupFailure` 为 `private set`，唯一写入点 `:72`（unregister 抛异常），全仓库无复位。
2. `NotificationOperations.kt:65-74`：失败时 `access` 保留、`cleanupPending` 仍为 true；以后没有任何代码再调 `unregister`（下一次 `execute` 在 acquire 阶段就被拒）。
3. `UiBackend.kt:46` `cleanupFailure = notifications.cleanupFailure ?: closeError`；`:108` 传入 `Sessions.acquireUi`；`Sessions.kt:144` 抛 `uiCleanup`——**所有 UI 命令**，包括只读的 snapshot、wait、screenshot、`notifications list/get`、`clipboard get`。
4. `UiBackend.kt:139`：通知命令且 `cleanupPending` 时不调 `sessions.finished(resource)`，UI 资源永久留在 `resources`。后果：shell/file `Sessions.reserve(…, changing=true)`（`ShellJobs.kt:34`、`FileJobs.kt:25`）→ `requireResources` busy；`prepareChange` → `requireIdle` busy（`Sessions.kt:117`）；`expire()` 要求 `resources.isEmpty()`（`:241-244`），租约永不到期；远程停止 `prepareStop` → `sessions.stop()` → `requireIdle` busy（`WorkerService.kt:115-121`、`Sessions.kt:232-235`）。
5. `TargetManagement.kt:46` 也检查 `cleanupFailure`。
6. 恢复入口：`stopExecutor`/`forceExecutor` 只由设备本地管理界面调用（`MainActivity.kt:137-138,199`、`GatewayService.kt:207,276`）；`destroy()` 只接受 App UID 或 shell（`WorkerService.kt:131-134`）。`connection.md:169`（报告写 170）原文“There is no remote force command.”；`notifications.md:35-36` 写明“requires the existing executor restart procedure”。
- 仍可用的：`ui.read`（`UiDelivery.kt:370` `acquireExistingUi` 不查 `cleanupFailure`）、`status`、`job.get`、`output.read` 等查询。

**报告 §3.3 / §7.2 D5 的错误**：`closeError` 由 `recordCleanupFailure`（租约清理失败，`UiBackend.kt:47-50,480-498`；建屏流失败后关闭流失败，`TargetManagement.kt:83-95`）和连接失败后断开失败（`UiBackend.kt:466-472`，同时 `sessions.closing` 留下资源）写入，同样永不复位、同样走 `cleanupFailure` 封锁。因此“其它清理失败不封”不成立；D5 的“读取不受影响”只对 `ui.read` 类查询成立。

### N6 通知清理未确认退出码 1 —— 成立
`NotificationOperations.kt:102-112`：`cleanup == "unconfirmed"` 且当前退出码为 0 时置 `exit=1`；`notificationExit`（`:114-119`）。`reference.md:66-67`：“Cleanup failure, uncertainty or signal failure overrides exit to 3”。`notifications.md:34` 只写“nonzero exit”。

### N7 窗口缺席两套语义 —— 部分成立
- 动作后：`UiBackend.kt:419-427` 在 `windowAbsent`（`ObservedChange.kt:218-227`：需已派发、动作前存在、同连接、本次不在）时置 `run.absentWindow` 并改用默认范围；`UiPostAction.kt:43-44,67` 不再采样，`:97-110` 写 `settle=changed` 与说明。
- wait：`windowAbsent` 要求 `sent=true`，wait 不派发，永远不会置位；请求窗口缺席时 `UiScope.kt:65-85` 抛 `scopeUnknown`，`TreeCollector.kt:76-101` 只做窗口目录、不读任何节点；`UiCondition.kt:309-321,366-371` 视为可重试，`:387-413` 循环到预算耗尽。
- 修正：wait 侧不是“整树重采到超时”，而是“每个检查点重新枚举窗口目录、零节点、到超时报失败”。`--gone` 与其它条件处理相同（都等到超时）。

### N8 基线不完整时每个动作耗满观察窗口 —— 部分成立
`UiPostAction.kt:63-68` 只有两种提前结束：`stableWithoutBaseline()`（`before==null && same>=2`，`:200`）或 `classification()==CHANGED`（`:188-198`）。
| 基线 | 行为 |
|---|---|
| 空（`before==null`，如系统动作 `SystemActions.kt:58`） | 两份连续相同完整样本即结束 |
| 不完整（`completeTree(before)==false`，`UiObservation.kt:45-49`） | `:193` 恒 UNSETTLED，永不 CHANGED，跑到 `until` 或 `canSample` 预估超限（`:78-81`） |
| 完整且有变化 | 两份相同且≠基线即 CHANGED 结束 |
| 完整且**无变化** | 只能 UNCHANGED，不在提前结束条件里，同样跑满 |
修正：耗满窗口不是“不完整基线”独有，任何无效动作都会跑满（ime-api34 tap 原件即 `settleMs 1368`、`samples 4`）。“网页长页常见”无原件支持：Chrome 原件四份 4226 节点树均 full；原型臂滚动 `settle=unsettled`+`samplingStopped` 的原因是单次采样昂贵而非基线不完整【原件】。

### N9 建屏前提检查晚于旧租约失效 —— 成立
`Sessions.kt:98-121` `prepareChange`：`:119` `invalidateLease(held)` 在建屏之前。失效之前的失败：`accepting`、`validate`、released（仅 takeover）、他人占用 busy、`requireIdle`；以及 `TargetManagement.kt:18-26` 的 idle 参数校验、`WorkDisplay` 构造（无检查）。失效之后的失败：`TargetManagement.kt:46` uiCleanup、`:47` 授权、`:102` `mainReadyForWorkDisplay`（主屏亮、解锁、ON，`DeviceFacts.kt:76-81`）、`WorkDisplay.kt:33-77` 创建失败/标志缺失（`displayUnsupported`）/非 ON。补充：失效后当前无租约，调用者下一条不带 virtual 期望的命令会经 `keepLease` 隐式建立新的主屏租约（`Sessions.kt:90-97,146-148`），旧 ref 全失效。报告行号 116-120 应为 98-121（失效点 119）。

### N10 首个按下已明确未发出仍做后采样 —— 部分成立
`VisualInput.kt:242-246`：`if (run.dispatchStarted)` 即 settle；`dispatchStarted` 在送出首个事件前由 `backend.dispatch` 置位（`VisualInput.kt:276-290`）。`PointerGesture.kt:240-253`：首个 DOWN `targetMismatch` 时 `state=NOT_SENT`，但不影响 `dispatchStarted`。原件 `ime-api34/commands/default-one-visible-letter-tap/stdout.bin`：`result:"not sent"`、`stop:"targetMismatch"`、`settleMs:1368`、`samples:4`、`settle:"unsettled"`；trace `elapsedSeconds 1.422`；`command.json` exit 2、CLI 2.016 s【原件】。“白花约 2 秒”应为“约 1.4 秒（settle）”。

### N11 回包尾部严格检查 —— 成立（细化）
`CollectionNodes.kt:125-130`：读完节点和 interactionId 后 `dataAvail()!=0` 即 `return false`，主结果与预取都不收。`awaitPrimary`（`:160-170`）等到 `until`（采集截止，样本起点 +3000 ms），返回 null，`child()` 返回 null；之后每次 `child()` 在 `:38` 因已过截止立即返回 null。所以是“第一个子节点等满截止，其余立即放弃”，每次采样都多花约 3 s，树只剩各窗口根（根来自公开 `window.root`）。当前平台未触发，触发条件为假设【推断】。

### N12 通知写命令没有自身包检查 —— 成立
`NotificationActions.kt:238-269`、`NotificationRequest.validate`（`:12-31`）均无 own-package/UID 判断；对照 `ActivityLaunch.kt:187-190`（`protected`）、`AppInput.kt:86`、`TreeCollector.kt:240`、`UiObservation.kt:86`。可观察风险：Gateway 自己的前台通知的 contentIntent 若指向管理界面，`notifications open` 会绕过 ActivityLaunch 的拒绝【推断】。

### N13 Linux 上 pytest 失败 —— 部分成立
- 机制【源码】：`private_files.py:21-24` POSIX 下状态目录是 `HOME/.local/state/androidctl`；`conftest.py` 的 `isolated_state` 把 HOME 设成 `tmp_path`；测试 `workspace.display_path().parent.mkdir()`（无 `parents=True`）在 POSIX 上因 `tmp/.local/state` 不存在而 `FileNotFoundError`，Windows 下父目录是 `LOCALAPPDATA/AndroidCtl`，可以建。
- 可静态指认：`test_display.py:76-78`（8 个参数）、`:163-166`（仅 `selection=bad` 1 个）、`:383-388`（3 个）、`:433-436`（1 个）、`:507-523`（7 个）＝20；`test_ui_batch.py:333-338`（2 个）＝22。`test_workspace.py:30,56,82` 用的是工作区 `config_path`，父目录可建，不在此列。第 23 个与 `workspace.py:127`（`read()` 的 `read_text`）的对应关系没能静态确认。
- CI【源码】：`.github/workflows/quality.yml:14-33` `python-posix` 在 ubuntu-24.04（3.12、3.14）和 macos-15-intel 上直接 `python -m pytest`（testpaths 含 `androidctl/tests`）；`python-windows` 跑 `task python:ci`。三个 POSIX 任务都会遇到上述失败。“1884 passed 只是 Windows 结果”未核。

### N14 种类不符的提示文案 —— 成立（细化）
`Sessions.kt:152-169`：无租约且期望 virtual 时只提示 “run display create”（`:154-156`）；有租约且期望 virtual、实际 main 时提示 “run display create or display select main”（`:157-168`）。对“偏好残留 virtual、只想用主屏”的调用者，前者只指向建屏，后者把建屏放在第一位。报告行号 160-168 只覆盖后一种。

### N15 diff 续读提示 —— 成立
`ui_diff.py:290-298` 截断时提示 `snapshot --cached --diff`；`:229,243` 只要可比即 `eligible`，`:343-346` 预留候选基线，`ui_client.py:1018,1101-1111` 提交——截断不阻止基线推进，续读比对的是已推进的基线，只能得到零变化。

---

## 3. 逐条：B 部分

### B1 后台屏会话下拉通知栏被拒 —— 成立（限定在产品 UI 命令面）
`SystemActions.kt:41-43` 对 VIRTUAL 目标拒绝 `home/recents/notifications`。其它产品路径：`key` 只允许编辑键（`KeyPlan.kt:19-44`，无通知键）；坐标/节点命令目标是后台屏；树只枚举目标屏窗口（`TreeCollector` 用 `windowDirectory(sampled, displayId)`）。但 `shell` 不需要显示目标（`RpcMethods.requiresDisplay` 不含 shell；`ShellJobs.kt:33` 走 `Sessions.acquire`），shell UID 可执行 `cmd statusbar expand-notifications` 与 `input tap`（shell 持有 STATUS_BAR_SERVICE、INJECT_EVENTS，A14 Shell manifest `:171,285`）。这条绕行不经节点、无引用、不受产品护栏，说“完全不存在路径”过强【源码＋推断】。

### B2 借贴 —— 成立
- 只走无障碍粘贴：`UiActions.kt:258-271` `ready()` 要求节点当前宣告 `ACTION_PASTE`，`paste()` 调 `invokePlatform(..., ACTION_PASTE, ...)`。报告引 `257-262`，实际 257-271。
- 恢复条件（`BorrowedPaste.kt`）：只有借写已尝试（`borrow.attempted`）才清理（`:97-103`）；先读当前剪贴板：读不到 → 若可接受为空则 `skipped`，否则 `unknown`（`:244-252`）；与借写内容不同 → `skipped`“Current clipboard changed”（`:254-257`）；相同 → 恢复原剪贴板或原为空时清空，并回读核对（`:267-290`）。
- 原剪贴板含 URI 或 Intent：借写前直接拒绝 `clipboardTransfer`，不写不贴（`:185-202`）。
- 强杀：原剪贴板只存在 worker 内存字段 `original`（`:76`），借写内容是标签为 `AndroidCtl <uuid>` 的纯文本（`ClipboardAccess.kt:98-104`），全仓库没有启动时恢复逻辑 → 强杀后借写内容留在剪贴板，原内容丢失【源码】。

### B3 旧原型证据 —— 成立（均为研究探针，非产品代码）
- Termux【原件】：`commands/e13-termux-clipboard`（`BorrowClipboard` 借写 `ICF11-TX-P-中文`，exit 135 为 MuMu 进程退出码，stdout 有 borrow/restore 记录）＋`e13-termux-ctrl-alt-v`（`TargetedInputProbe` 向 UID 10087 发 Ctrl+Alt+V 六个键事件，均 returned）；`artifacts/e13-termux-paste.png`（1080×1920，亲看）提示符后出现 `ICF11-TX-P-中文`，系统 toast“已将剪贴板中的内容粘贴到Termux”。路径跑通。
- WPS【原件】：`e13-wps-commit-insert` commit `ICF11-WP-I-中文`；`e13-wps-after-insert` surrounding text `ICF11-WP-I-中文\n`；焦点节点 `editable=false`、`text=null`、动作只有 2/4/8/16/32/64/show-on-screen（无 SET_TEXT、PASTE）。`e13-wps-select-middle` 请求 9..10 后 `e13-wps-selected-inspect` 选区仍 13,13，`e13-wps-selected.png` 亦无高亮 → 独立选区被忽略。
- Contacts【原件】：插入后 `ICF11-CT-ICF11-CT-I-中文abcdef`；select 9..12 后 commit 得 `ICF11-CT-ICF11-CT-S-中文def` → 插入与替换都成功；节点宣告 SET_SELECTION（131072）。

### B4 原生图源与坐标资格的耦合 —— 部分成立
- 同一状态：`VisualInput.kt:17-22,30` 单个 `image: ImageState(lease, bitmap, meta, layout)`；`zoom`（`:132-148`）用其 `bitmap`；坐标输入（`:205-241`）用其 `meta`。
- 同一失效：`invalidate()`（`:32-36`）同时丢弃两者；触发点：树签名变化（`:39-44`）、新截图（`:63`）、采样失败（`UiBackend.kt:316,347,385`）、租约清理（`:491`）、带截图的其它命令（`UiPublication.kt:33`）、坐标输入发现几何变化（`VisualInput.kt:217-224`）。
- 不共用的规则：120 s TTL 只在坐标输入检查（`:210-214`），zoom 只检查同租约和图 id，不检查 TTL、不要求 `mappingKnown`。所以是“状态与失效触发耦合，时效规则不耦合”。
- 证据图分辨率【重算，`agents5/24/imgdims.py`】：`stages/36-native-zoom/runtime-mumu/` 源 1080×1920（`full-source-roi.png`、`settings-marked.png`），预览 720×1280；`runtime-software-api34/`、`runtime-u25-followup-api34/` 全部 ≤720×1280，裁剪宽 460–720，即源本身 720×1280，zoom 在此无分辨率增益。

### B5 通知 listener 上游行为 —— 成立
- (a) Shell manifest 不含 `ACCESS_NOTIFICATIONS`（A14/A15/A16 全文 grep 无）；含 `STATUS_BAR_SERVICE`（A14 `:285`、A15 `:292`、A16 `:299`）与 `MANAGE_NOTIFICATION_LISTENERS`（`:554/:571/:582`）。`registerListener` 只要求 `enforceSystemOrSystemUI`（A14 包内 `a14-NotificationManagerService.java:4534-4538`，A16 `:5629-5633`，A15 `:4941-4945`）。【源码】
- (b) 注销：`unregisterListener` → `ManagedServices.unregisterService`（A14 `:1084-1089`）→ `removeServiceImpl`（A14 `:1670-1685`、A15 `:1750-1765`、A16 `:1922-1937`）从尾到头删除**所有** `asBinder()==` 且 `userid==` 的登记。
- (c) 重复注册：`registerSystemService`（A14 `:1091-1099`）→ `registerServiceImpl`（A14 `:1699-1716`、A15 `:1779-1796`、A16 `:1951-1968`）每次 `linkToDeath` 后 `mServices.add`，不去重、不抛异常 → 重复登记，每次 `onServiceAdded`；随后一次注销全部删除。`linkToDeath` 还意味着 worker 死亡时登记自动清除。
- (d) 注销未登记对象：`removeServiceImpl` 返回 null，`unregisterServiceImpl` 无操作，不报错（`checkNotNull` 只拒绝 null）。
- 耗时【重算，`agents5/24/b5_timing.py`】：`T/stages` 下 29/30/31/final-verification 四处回执中 `notificationRegisterMs` 101 个值，0–1 ms；`notificationCleanupMs` 101 个值，0–3 ms（含同一回执 stdout/trace 重复计数）。仅 API34/API37 模拟器。

### B6 发布时间与地址比对 —— 成立（细化）
- 上游：每次入队 `new StatusBarNotification(..., System.currentTimeMillis())`（A14 `a14-NotificationManagerService.java:6704-6706`、A15 `:7428-7430`、A16 `:8454-8456`），更新同一通知也重置；未找到恢复旧 postTime 的代码【源码，grep】。
- 产品严格比对：`NotificationActions.kt:243`。
- 已存在的检查：键（`:240-241`）、动作序号与**动作按钮标题**（`:278-286`）、PI 创建者包与 UID（`:260-264`，仅 action/open/reply；dismiss 在 `:246-250` 提前返回）。通知自身标题 `EXTRA_TITLE` 与内容没有比对。

### B7 VLC 启动模式与 ActivityStarter —— 成立
- VLC 3.7.1（`prototypes/route-review/launch/sources/vlc-3.7.1-AndroidManifest.xml`）：入口 `.StartActivity`（`:96-110`）无 launchMode（标准）、透明无 UI、exported；`.gui.MainActivity` singleTask（`:967-970`）；`.gui.video.VideoPlayerActivity` singleTask（`:1026-1030`）；APK 解析 `vlc-3.7.1-apk-manifest.txt:1878,1936` launchMode=2 一致。`vlc-3.7.1-StartActivity.kt:251-253,299` 由入口转发到 VideoPlayer（REORDER_TO_FRONT）或 MainActivity。【源码】
- A14 `ActivityStarter.java`（本轮下载）：`getReusableTask` `:2861-2863` singleTask/singleInstance 无论 MULTIPLE_TASK 都尝试复用已有任务，`:2882-2883` `findTask(mStartActivity, mPreferredTaskDisplayArea)`；`adjustLaunchFlagsToDocumentMode` `:3153-3161` NEW_DOCUMENT 与 singleTask 冲突时剥除 NEW_DOCUMENT|MULTIPLE_TASK，`DOCUMENT_LAUNCH_NEVER` 时剥除 MULTIPLE_TASK（`:3172-3188`），`:2627-2629` DOCUMENT_LAUNCH_ALWAYS 加 MULTIPLE_TASK。结论：给入口加 `NEW_TASK|MULTIPLE_TASK` 只能让 StartActivity 落新任务，它转发的 singleTask 页面仍复用已有任务。“任务是否被搬到后台屏”取决于 `findTask` 跨屏结果与后续重挂父，源码未逐行核到底【需设备，即报告 V9】。

### B8 VLC 启动原件 —— 成立
`launch/runtime-api34/commands/vlc-view-once/stdout.bin`：`result:"done"`、`launchCodeName:"START_SUCCESS"`、`landing.state:"unknown"`（matchingTaskCount 0）、`settleMs:1112`、`samples:5`、末尾 `"exit":1`；`command.json` exitCode 1、1.625 s。同次 `002-windows.decoded.bin`（sequence 5，`finishedMs 1229314`，在 settle 1228215→1229327 之内）：displayId 2 唯一应用窗口 `org.videolan.vlc`、content full。sidecar `artifacts/2242-533d…json`：`originalLiveCall2`（1228214–1228215）VD2 task174 `StartActivity` visible=false；`afterSettle`（1229328–1229333，settle 结束之后）同任务 `VideoPlayerActivity` visible=true。注意：settle 内末次采样只证明**包**是 VLC，不证明是哪个 Activity；Activity 级证据是 settle 之后的 sidecar 读取。

### B9 E9 与公开缓存路线 —— 成立
- 采集时间【重算，`agents5/24/b9_logs.py`，`collectedMs-preparedMs`】：collector 运行（4226 节点）E9 1024/399、single 1909/2209；collector-cache 运行（4243 节点）E9 606/580、cache 655（次采）/1047（首采）。报告数字一致；注意 single 与 cache 来自两次不同运行，且 E9 自身在两次运行间 399–1024 波动。
- 上游（包内 `research/observation/upstream/android15-release/…`，分支名而非标签）：
  - 只收最后一次交互的预取：`AccessibilityInteractionClient.java:726-732` 单字段 `mInteractionIdWaitingForPrefetchResult` 每次请求覆盖；`:1269-1300` 只在 id 相等时接收。
  - 滚动后 150 ms 不预取：`:116-117`（`getSendRecurringAccessibilityEventsInterval()*1.5`，100 ms×1.5）、`:648-652`、`:1594-1607`、事件入口 `:1153-1156`。
  - 内容变化逐出子树：`AccessibilityCache.java:259-268`（SUBTREE 型清子树，其余移除单节点）、`:272-274`（滚动清子树）、`:722-760`。
  - UiAutomation 转发：本轮下载 A15 `UiAutomation.java:1848` `IAccessibilityServiceClientImpl extends IAccessibilityServiceClientWrapper`；`AccessibilityService.java:2976-2982` wrapper 把每个事件交给 `AccessibilityInteractionClient.onAccessibilityEvent`，后者转 cache（`AccessibilityInteractionClient.java:1166-1173`）。三项均属实。

### B10 Chrome 节点字节与 8 MiB —— 成立
【重算，`agents5/24/b10_bytes.py`】collector 原件 4226 节点 nodes 数组 6,528,667 B，每节点 1,544.9 B（含树头约 1,545.8 B，与 `decodedSize 6,532,479` 一致），8 MiB（`Wire.kt:20`，`UiTreeBudget.kt:42-54` 按编码字节截断）可装约 5,429 个，低于 `Limits.NODES=8192`（`Wire.kt:27`）。稀疏估计依删减口径而异：实施方 `schema/README.md` 的“动作简写＋默认省略”省 3.26 MB → 约 774 B/节点（报告采用此数）；我更激进的口径（另删 true 默认的 visible/enabled/refAvailable 和完整性标记）约 538 B/节点。两种口径下 8192 节点均可装入。动作数组占原始树字节 21.4%（与 B11 的 41% 是 CLI 输出口径，不同）。

### B11 四臂 usage 与旧基线 —— 成立
【重算，`agents5/24/b11_usage.py`，末个 `token_count.total_token_usage.input_tokens`，与逐条 `token_usage_record` 求和一致】
| 臂 | 总输入 | CLI 次数 |
|---|---:|---:|
| Contacts 旧基线 `stages/final-verification/paired-runtime/contacts-1` | 812,987 | 16 |
| Contacts 候选原版 | 1,395,045 | 20 |
| Contacts 紧凑原型 | 629,770 | 15 |
| Chrome 旧基线 `…/chrome-2` | 1,041,124 | 20 |
| Chrome 候选原版 | 2,326,407 | 15 |
| Chrome 紧凑原型 | 1,821,534 | 20 |
Chrome 原型臂全部 stdout 254,556 B 中动作数组 105,473 B，占 **41.4%**；`{"id":64}` 出现 **1,014** 次【重算，`b11_actions.py`】。均与报告一致。旧 paired-runtime 另有 contacts-2（候选，1,461,258）、chrome-1c（候选，2,447,717）。

### B12 会话相关源码事实 —— 成立
- 主屏 `session open`：`TargetManagement.kt:100-105` `createTarget` 对无 stream 的主屏直接返回，不连接 UiAutomation、不调用 `mainReadyForWorkDisplay`；只有建后台屏才要求主屏亮且解锁。
- `session open` 不改偏好：`workspace.save_display` 只有两个调用点 `app.py:799`（display select）和 `app.py:841`（display create 成功后）。
- 偏好文件：`workspace.py:23-24` `state_directory()/display.json`，索引 deviceId → session → main|virtual（`:27-52`），缺省读 `main`（`:55-56`）；读取点 `ui_client.py:594-607`（每个 UiClient 首次冻结一次）。
- 无租约时隐式建立主屏租约：`Sessions.acquireUi`（`:136-149`，经 `keepLease` `:90-97`，`Lease.target` 缺省 MAIN `:15-21`）覆盖全部 UI 命令（含 notifications、clipboard、apps、wake）；`Sessions.acquire`（`:56-64`）覆盖 shell（`ShellJobs.kt:33`）与 file（`FileJobs.kt:24`）。例外：期望 virtual 时报 `displayRequired`（`:154-156`）；会话键在本地 released 集合时报 `sessionReleased`（`:70-72`）。

---

## 4. C 部分：坐标接收者内部接口的接入事实

**接口形态【源码】**
- `android.window.WindowInfosListener` 在 A14/A15/A16/A17 都是 `@hide` 的 `public abstract class`（A14/A15 `:30-32`，A16 本轮下载同 A15，A17 `:35-37`），唯一抽象方法 `onWindowInfosChanged(InputWindowHandle[], DisplayInfo[])`；构造器内 `nativeCreate(this)`。必须子类化，`java.lang.reflect.Proxy` 不能代理抽象类。A15 与 A16 Java 文件无差异；JNI 只多一行宏。
- JNI（A15 `android_window_WindowInfosListener.cpp`）：`nativeCreate` 保存 `NewWeakGlobalRef(listener)`（`:92-93,130-134`）——Java 对象若无强引用会被回收，回调静默丢弃（`:100-104`）。`nativeRegister`（`:143-152`）**忽略** `addWindowInfosListener` 返回值；`nativeUnregister`（`:155-158`）返回 void，同样忽略。
- 权限：`SurfaceComposerAIDL::addWindowInfosListener/removeWindowInfosListener` 要求 uid system/graphics 或 `ACCESS_SURFACE_FLINGER`（A14 `SurfaceFlinger.cpp:9077-9091`、A15 `:10432-10446`、A17 `:10717-10731`）。Shell 持有 `ACCESS_SURFACE_FLINGER`（A14 manifest `:193`、A15 `:196`、A16 `:199`）。被拒时 Java 侧拿到空数组、无异常。
- 初始数据：`WindowInfosListenerReporter`（进程单例，A15 `:28-58`）只在本进程第一个监听者时向 SF 注册，初始结果是本进程**缓存的上一份**窗口；最后一个监听者注销时清空缓存（`:61-83`）。SF 侧注册后 `setTransactionFlags(eInputInfoUpdateNeeded)`（A14/A15/A17 `SurfaceFlinger::addWindowInfosListener`），新数据经回调送达。所以“每命令注册、用完注销”的常驻 worker 每次 `register()` 初始都是空，必须等回调。
- 注销后回调：Reporter `onWindowInfosChanged`（A15 `:85-100`）在锁内复制监听者集合后在锁外逐个回调，注销与在途回调并发时，注销返回后仍可能收到一次回调【源码＋推断】。注销未登记对象直接返回 OK（`:66-68`）。
- `InputWindowHandle` 字段：探针用的 `name/token/ownerUid/displayId/inputConfig/touchableRegion/transform` 在 A14、A15、A16、A17 同名同型（A14 `:81-159`、A15/A16 `:84-158`、A17 `:84-161`）。差异：A14 用 `frameLeft/Top/Right/Bottom` 四个 int（`:106-109`），A15 起改为 `Rect frame`，并新增 `contentSize`、`canOccludePresentation`；若实现用到 frame 需分版本。`inputConfig` 位定义来自 `android.os.InputConfig`，各版本位值本轮未逐一比对。

**探针怎样编译和加载【原件】**
- `coordinate-window/compile-only/` 手写 `WindowInfosListener`、`InputWindowHandle` 桩，仅参与 javac；`build/build.ps1` 用 JDK21 javac（`--release 11`，android-36 android.jar）编译，D8 `--min-api 34` 只打包 probe 类。dexdump 实为 4 个类（三个 probe 类加 D8 合成的 `CoordinateWindowProbe-IA`），初次构建脚本的“恰好三类”检查因此失败（`build/initial-build.json`、`result.json` 的 `initialGateFailed`），产物中无 `android.*` 类。
- 运行：`adb shell CLASSPATH=…jar app_process / dev.androidctl.research.CoordinateWindowProbe`，shell UID 2000，独立进程，0.235 s 退出 0（`runtime-api34/commands/probe-once/command.json`）。

**探针实测证明了什么【原件，`probe-once/stdout.bin`】**
- 证明：API34 软件模拟器上，以 shell 身份、boot classloader 父加载器的普通 app_process 进程，子类化隐藏抽象类可链接运行；`initialWindowCount 0`、`initialDisplayCount 0`；约 10 ms 后（deadline 3808233−3000＝起点 3805233，sample 3805243）首个回调给出 10 个窗口、1 个显示；InputMethod ownerUid 10109、touchable `[0,658,720,1184]`；Contacts 窗口 uid 10095 `[0,0,720,1280]`；`unregisterCallReturned=true`；进程退出后 `probe-after-ps` 无残留。
- 没证明：SF 侧注册/注销是否成功（JNI 不回报状态，`register` 返回只说明 Java 方法返回）；注销后不再回调；Shizuku UserService 进程、R8 发行包、A15/A16/真机或 MuMu 上的行为；后台屏（VD）窗口与显示变换；预测结果是否与 InputDispatcher 实际派发对象一致；没有做任何注入。样本中还有 `ActivityRecordInputSink`（uid 1000，inputConfig 5/15，区域覆盖全屏外扩）、壁纸（区域空、缩放变换）等窗口，命中算法必须复现 InputDispatcher 对 `NOT_TOUCHABLE`、`NO_INPUT_CHANNEL`、变换和 z 序的处理，探针未涉及。

**常驻 worker 里同样方式加载的未验证环节【源码＋推断】**
- worker 是 Shizuku UserService（`WorkerConnection.kt:52-55,184`），由 Shizuku 用 `createPackageContextAsUser(…CONTEXT_INCLUDE_CODE…)` 得到 App 的 classloader 再 `loadClass`（包内 `research/notifications/upstream/shizuku-UserService.java:71-82`），父加载器链到 boot classpath，理论上能解析 `android.window.WindowInfosListener`，未实测。
- 候选目前对隐藏 API **只用反射**（`HiddenApi.kt`），没有 compileOnly 桩、没有 hidden-API 豁免代码；子类化需要新增桩 jar（compileOnly）与直接链接隐藏构造器/方法。worker 里反射隐藏成员已在多个阶段可用，说明该进程未强制 hidden-API 策略，但“直接链接”路径未在 worker 实测。
- 发行构建 `isMinifyEnabled = true` + `proguard-android-optimize.txt`（`build.gradle.kts:167-171`），仓库无自定义 proguard 规则文件。R8 若看不到桩（未作为 library 提供）会报缺类或需 `-dontwarn`，此时可能把 `onWindowInfosChanged` 重命名或移除（JNI 按名字 `GetMethodID(..., "onWindowInfosChanged", …)` 在基类上取 ID，`:180-181`），覆写被改名会导致回调落到抽象方法。需要 keep 规则并在 release 包上验证。
- 回调线程是 binder 线程，worker 需保持强引用与 binder 线程池；与 UiAutomation 回调线程的并发未设计。

**对“便宜验证清单”的评估与补充**
| 项 | 判定通过的标准（建议） |
|---|---|
| ① 内部接口能否在常驻 worker 里加载 | 用**发行配置（R8 开启）**构建的 APK，在 Shizuku worker 内：类加载与实例化成功；`register()` 后 ≤500 ms 内收到非空回调；在 API34 模拟器和 MuMu（若为 A12 以下需另论）各一次；dex 中能看到保留下来的 `onWindowInfosChanged` 名称 |
| ② 能否干净注销 | 同一对象“注册→注销→再注册→注销”各两轮；注销后在主屏制造窗口变化（弹/收键盘），连续 2 s 无回调（允许注销瞬间 1 次在途回调并记录）；`dumpsys SurfaceFlinger` 中本进程监听者计数归零（若该版本 dumpsys 有此项）；worker 被杀后同样归零 |
| ③ 预测与实际接收者一致 | 至少四类点：软键盘区、普通 App 区、状态栏/导航栏、分屏或悬浮窗边界（各在主屏与后台屏）；每点先预测 UID，再以该 UID 定向注入，记录平台是否 `targetMismatch`；一致率 100% 才通过，任何不一致记录窗口列表原件 |
| 缺项④ 权限被拒与空数据的区分 | 因 JNI 不回报状态，需定义“注册后 N ms 无回调即视为不可用并退回窗口目录方案”的超时，并在无 `ACCESS_SURFACE_FLINGER` 的场景（例如非 shell 身份）验证退回路径 |
| 缺项⑤ 版本字段 | 在 A14 与 A15+ 各跑一次读取所用字段（避免用 `frame`），并核 `inputConfig` 位定义 |
| 缺项⑥ 后台屏 | VD 的 displayId、`DisplayInfo.mTransform` 与 VD 坐标换算在后台屏会话里实测一次 |

---

## 5. 无法在无设备条件下判定的条目与所需实验
1. N1 同一 worker 内重建 UiAutomation 连接后 Chrome 动作次序是否重置，以及第 3 次以后采集是否稳定：同一 worker 内 `session close/open` 两次，每次连续采 3 份，比较动作次序。
2. N4 后台屏会话 `notifications open` 的实际落屏：VD 会话下对一条 Activity 型通知执行 open，读 VD 与主屏任务栈。
3. N5 注销失败实际可触发的条件（Binder 异常类型），以及失败后本地“Force executor exit”能否恢复：用注入失败的研究构建验证。
4. N11 平台追加回调参数的情形：只能在出现此类平台版本时验证，或用研究构建人为在回包后追加 int 模拟。
5. B1 shell 绕行在后台屏会话的可用性与对真人的干扰：shell 展开通知栏后 `input tap`。
6. B7/B8 VLC singleTask 任务是否被搬到后台屏（报告 V9）。
7. C ①–⑥ 全部需设备。
8. N13 的第 23 个失败与 Linux 总数：在 Linux 上实跑一次 `pytest androidctl/tests`。

## 6. 未核实与局限
- N13 的“1884 passed 只是 Windows 结果”与第 23 个失败未核；未运行 pytest。
- N12 的“dismiss 可能清掉控制者通知”只核了缺少检查，未核前台服务通知在 A14+ 是否可被 listener 取消。
- B7 只核到 `getReusableTask`/文档模式剥离，`findTask` 跨屏与重挂父的完整路径未逐行核。
- C 的 `inputConfig` 位值、A16 `SurfaceFlinger.cpp`、A14 `WindowInfosListenerReporter.cpp` 未下载比对（A14 依赖包内 A15 推断初始缓存语义，A14 JNI 同样返回 Pair）。
- B9 上游引用的是包内 `android15-release` 分支副本，非固定标签；A17 副本（`research/selected-integration/platform17/…`）未逐行比对。
- 报告 §4.4 关于“AOSP 14 无障碍窗口填充器继承窗口信息监听器”不在本次清单内，未核。
