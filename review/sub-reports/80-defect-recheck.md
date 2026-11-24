# 第六轮关键缺陷独立复核（初稿 A2、A3、A4、A5、B1、B6）

复核对象：产品候选 `fb75afc`（下称"候选"），主分支基线 `7d5be70`（下称"基线"）。
做法：每条先从源码、AOSP 和原件独立推一遍，写下结论后再读专题记录 64、65、66、68、69、70、71、73、74、75 和初稿 `androidctl-r6-candidate-review-fb75afc.md`，最后比较出入。
记号：`S` 为评审暂存目录；`PKG` 为实施方本轮材料包；`ST` 为 `PKG/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`；我的脚本、导出树和 AOSP 下载都在 `S/agents9/80/`。没有设备，没跑 Gradle。

## 总表

| 初稿条目 | 我的结论 | 确定程度 | 与初稿的主要出入 |
| --- | --- | --- | --- |
| A3 自动化登记被占用后界面能力卡死 | **成立**，必须修，小 | 平台路径【源码】确定（AOSP 14/15/16 三版一致）；候选控制流【源码】确定；只差一次设备复现 | 来源说错了：基线的退路在第四轮候选里已被另一种"下次登记成功即确认"的自愈机制取代，**真正把它变成永久卡死的是第五轮候选按第四轮决定第 25h 项删掉自愈**，而 25h 来自评审方第四轮报告 §6 第 9 条的建议。修法还有一个更省的版本（不进入"待注销"状态，约 5–10 行，不需要隐藏成员） |
| B6 后台屏 Chrome 几乎全报可见 | **成立**，且比初稿说的更确定 | 现象【原件】【重算】确定；"后台屏特有"有同一模拟器、同一 Chrome Beta 的主屏对照支持，【推断】但证据强 | 初稿的"主屏约 526 个可见"来自另一台模拟器（MuMu，1080×1920）的旧数据，不是本轮原件；同机对照应改用 API 34 主屏的 Python 教程页。默认输出确实打印了 44 万字节——有真实的 stdout 原件，不只是重算 |
| B1 引用因名字变化被拒共 7 次 | **成立**，数字全部对上 | 【原件】【源码】确定 | 无实质出入；补一句"拒绝时新树已采好、新引用已发号" |
| A4 版本号没变 | **成立**；但问题不在"只比字符串" | 【源码】确定；混用症状【推断】 | 严格相等本身就是合理的锁步设计；缺陷只是没升号。没有独立协议版本号，契约文件也不带版本 |
| A5 中间提交不可运行 | **成立**（实跑：第 2–8 个提交主机侧导入即失败） | 【实跑】确定 | "不强求中间提交各自可用"是**用户自己在 U33 里写的**，不是实施方擅自；应从"缺陷"改写成"合入方式的约束" |
| A2 成功按键与滚动查找仍打印过程计数 | **成立**，是"没落实"而非裁决歧义 | 【原件】【源码】确定 | 量很小（17 次 `back` 共 7.1 KB，12 次滚动查找共 2.3 KB），作为"未落实裁决"保留在合入前顺手修即可，不应与 A3 并列为同等级障碍 |

---

## 1. 自动化登记被占用时界面能力卡死（初稿 A3）

### 1.1 独立结论

**成立。严重度：中高（远程调用者无法自行恢复，必须有人到设备上操作）；触发概率低但现实存在。必须修，改动小。**确定程度：平台行为和候选控制流都是源码确定的，唯一没有的是设备复现。

### 1.2 平台一侧（AOSP 14、15、16 三版逐行核对，代码路径完全一致）

文件：`S/agents6/29/aosp/android-{14,15,16}.0.0_r1-{UiAutomation,UiAutomationConnection,UiAutomationManager}.java`；`AccessibilityManagerService.java` 14 和 16 我新下载到 `S/agents9/80/aosp/`（标签 `android-14.0.0_r1`、`android-16.0.0_r1`）。

1. **`UiAutomation.connectWithTimeout`**（A14 第 358–409 行，A16 第 363–414 行；三版函数体逐字节相同，md5 一致）：先在锁内把状态设为"连接中"（CONNECTING），启动名为 `UiAutomation` 的远程回调线程，然后调用 `mUiAutomationConnection.connect(...)`，**只捕获 `RemoteException`**。其它异常直接抛出，状态停在"连接中"，不会回到"已断开"或"失败"。只有等待超时时才把状态设为"失败"（FAILED）。
2. **`UiAutomationConnection.connect`**（A14 第 111–125 行，A16 第 107–121 行）：在调用系统服务**之前**先设置 `mOwningUid`，然后调用 `registerUiTestAutomationServiceLocked`；`mClient` 只在系统服务**正常返回之后**才赋值（A14 第 577 行，A16 第 689 行），`isConnectedLocked()` 就是 `mClient != null`。
3. **`AccessibilityManagerService.registerUiTestAutomationService`**（A14 第 1405–1428 行，A16 第 1628–1649 行）：权限检查后直接调用 `UiAutomationManager.registerUiTestAutomationServiceLocked`，后者成功才会走到 `onUserStateChangedLocked`。
4. **`UiAutomationManager.registerUiTestAutomationServiceLocked`**（A14 第 101–106 行，A15 第 108–112 行，A16 第 108–111 行）：系统里只有一个槽 `mUiAutomationService`；非空就抛 `IllegalStateException("UiAutomationService … already registered!")`。这一步在 `owner.linkToDeath` 和任何状态写入之前。**所以被拒时系统侧什么都没有登记。**
5. `IllegalStateException` 能跨 Binder 原样传回调用方。候选的 `UiAutomationConnection` 是本进程内对象，异常从它直接抛进 `connectWithTimeout`，不被捕获。
6. **`UiAutomation.disconnect`**（A14 第 433–458 行，A16 第 438–466 行）：状态为"连接中"时第一行就抛 `IllegalStateException("Cannot call disconnect() while connecting …")`，在 `try/finally` 之前，所以连回调线程都不退出。即使绕过它直接调 `UiAutomationConnection.disconnect()`，也会因为 `mClient == null` 抛 `"Already disconnected."`（A16 第 124–135 行）。
7. **能否共存**：系统只允许一个登记，`uiautomator dump`、Appium 的 UIAutomator2 服务、其它测试框架和候选互斥。`FLAG_DONT_SUPPRESS_ACCESSIBILITY_SERVICES`（候选用的 0x1）只决定是否压制已开启的无障碍服务，不影响这个互斥。只有带 `FLAG_DONT_USE_ACCESSIBILITY` 的登记会在 `useAccessibility()` 处提前返回、不占槽，`uiautomator dump` 不带这个标志。

结论：别的工具先占着槽、候选后连时，候选第一次连接必然抛"already registered"，随后 `disconnect()` 必然抛"Cannot call disconnect() while connecting"，而且**此时候选在系统侧根本没有登记**。

### 1.3 候选一侧的控制流（`fb75afc`）

- 连接入口：`UiBackend.kt:475-480` `connect()` 先调 `sessions.requireCleanUi()`，再调 `connectPlatform`；`UiBackend.kt:502-540` `registerAutomation` 在调用 `connectWithTimeout` **之前**执行 `cleanup.registrationStarted()`（第 533 行），把"需要注销"置为真；第 534 行连接抛错后进入 `discardConnection`（第 557–565 行），再进入 `clean`，最后调用 `AutomationCleanup.close`。
- `AutomationCleanup.kt:32-41`：只尝试一次 `disconnect`（`disconnectAttempted`），失败就记 `"UiAutomation registration release is unconfirmed: Cannot call disconnect() while connecting …"`，`registrationPending` 永远保持为真。回调线程那一半能正常结束（`stopCallbacks` 通过反射拿到平台的远程回调线程并让它退出），所以留下的只有"注销未确认"。
- 门：`Sessions.kt:250-252` `requireCleanUi()` 只要 `cleanupFailure()` 非空就拒绝，返回 `uiCleanup`，文字是 `Sessions.kt:399-400` 的 "UI cleanup unconfirmed; ask the user to use Stop idle executor on the device"。它挡住所有界面命令（`UiBackend.kt:119,163,478`）、新建后台屏和替换目标（`Sessions.kt:164,233,246`，`TargetManagement.kt:135`）。
- **有没有自动恢复路径**：
  - 下一条命令：被门直接拒绝，不会再尝试登记。
  - `session close` 重试：`UiBackend.kt:591-596` `cleanupRetry` 会再调 `clean`，但 `AutomationCleanup.close` 因 `disconnectAttempted` 不再调 `disconnect`。无效。
  - 新会话、空闲到期：同一个 worker 的 `pendingCleanup` 不清空。无效。
  - worker 进程退出：只有系统服务死亡（`WorkerService.kt:25`）或设备上"Stop idle executor"／"Force executor exit"（`WorkerConnection.kt:213-241`）。主机端没有任何重启 worker 的命令；`destroy` 只接受 App 自身 UID 或 shell UID 调用（`WorkerService.kt:146`）。
  - 结论：**没有远程恢复路径。** 设备上点"Stop idle executor"后进程退出，状态随之消失；之后能否自动重新附着我没有核（专题 71 说需要在设备上刷新或 Shizuku 重绑）。
- 指引如实写了这个设计：`skills/androidctl/displays.md:89-91` "Unconfirmed UiAutomation unregister requires the user's ordinary Stop idle executor on the device; do not try another registration to prove release."
- 第一条失败命令的回执会带出平台原文"…already registered!"，但随后附加的指导是"请用户去设备上停止执行器"，与真实状态（我们根本没登记）不符。

### 1.4 历史：退路是怎么丢的（与初稿出入最大的地方）

我用 `git -C SRC log -S` 在原始提交上追了两步：

1. **`6fc57c3`（2026-11-19，第四轮候选 `4235af9` 之前）** `fix(session): confirm cleanup stages and narrow pending gate`：删掉基线的 `HiddenApi.disconnectConnection` 和"认 `Already disconnected.`"的退路，新建 `AutomationCleanup`。提交说明没有给理由。但同一提交**换上了另一种恢复**：`4235af9:UiBackend.kt:574-616` 在下一次正常登记成功后调用 `confirmReleased()` 清掉旧的"待注销"；`registrationConfirmationReady`（第 53–57 行）允许"只剩注销未确认"时放行下一次登记尝试（第 115–122、542–552 行）。也就是说，在第四轮候选上，本场景会在别的工具松开槽之后**自动恢复**。实施方当时的设备试验计划（`S/pkg7/…/stages/41-route-rebuild/12-sessions/01-cleanup/runtime-plan.md:90`）明写"正式实现允许 registration-only pending 先走正常注册；成功后才可继续 UI"。
2. **`897a300`（2026-11-22，第五轮候选 `cab84b8` 之前）** `fix(sessions): distinguish queued and failed UI cleanup`：删掉 `confirmReleased`、`registrationConfirmationReady` 和试登记恢复，指引改成"An unconfirmed UiAutomation unregister stays blocked without trying a new registration"。这是在落实**第四轮决定第 25h 项**"简化清理重试：注销没确认就保持封锁，请用户在设备上停止"（`androidctl-r4-decisions.md:244`）。
3. 25h 的来源是**评审方第四轮报告**：`androidctl-r4-candidate-review-4235af9.md:356` 第 9 条："清理重试时，用'下一次注册成功'证明旧的注册已经释放……它防的那种故障已经被'系统进程死亡时后台进程退出'消除；所有非故障注入的运行里没有出现过清理错误；这段编排没有 JVM 测试 → 简化成'注销没确认就保持封锁，请用户在设备上停止'"。这条理由漏掉了"别人占槽时连接失败、`disconnect` 必然抛错"这条**不需要故障注入、平台正常行为就会走到**的路径。

所以：永久卡死从第五轮候选 `cab84b8` 开始存在；直接原因是评审方第四轮的简化建议加用户的 25h 裁决，删 `disconnectConnection` 只是间接原因（它被删时已有替代的自愈机制）。第五轮评审没有发现，专题 47 只核对了"25h 已实现"。这条应进初稿 §8"哪里带偏了"。

### 1.5 基线的退路覆盖什么

`7d5be70:UiBackend.kt:419-434`：`disconnect()` 抛 `IllegalStateException` 且平台对象尚未建成（`platform == null`，即连接阶段失败）时，改调 `UiAutomationConnection.disconnect()`；如果抛 `"Already disconnected."`，说明本地连接对象的 `mClient` 从未赋值、系统服务从未成功返回，按"已清理"处理。依据的是平台自身的本地状态（1.2 第 2 点），判断确定。它恰好覆盖本场景；对超时（状态为"失败"）的情况，正常 `disconnect()` 就能真正注销，不需要它。

### 1.6 触发概率

- 候选在**一个租约的第一条界面命令**时才登记，租约结束（`clear`，包括空闲到期）时注销（`UiBackend.kt:567-578`）。租约期间候选一直占槽，此时别的工具来登记会失败，候选不受影响。所以只有"别人先占、候选后连"这个窗口会触发。
- `uiautomator dump`：一次占槽约 1–3 秒。调用者自己用 `androidctl shell`（尤其是后台作业）跑 dump，同时新租约发第一条界面命令，才会撞上。低概率，但 LLM 调用者在观察不顺时去跑 `uiautomator dump` 是常见的兜底动作。
- Appium、UIAutomator2 服务、Maestro 等：在它们的会话期间一直占槽。设备上有这类工具时，候选的第一条界面命令必然失败；**工具退出后候选仍然卡住**，这才是相对基线和第四轮候选的回归。
- 本轮全部原件里没有出现过这个错误（我在 `ST/efficiency`、`ST/runtime` 里搜过 "already registered"、`uiCleanup`；日志里的命中都是无关的广播接收器）。
- 判断：低频、但一旦发生就是远程硬停；对"给 AI agent 用的远程 CLI"而言属于必须修的那一类。

### 1.7 最小修法（三选一，按我的推荐顺序）

- **A（推荐，约 5–10 行加 1 个 JVM 测试，不需要隐藏成员）**：把 `registrationStarted()` 的语义改成"系统可能已登记"。`connectWithTimeout` 抛出的根因不是 `TimeoutException` 时（即同步抛出的 `IllegalStateException`／`SecurityException`），直接把这次登记标为"未登记"，不进入待注销；超时仍按现在的路径正常注销。依据是 1.2 第 2–4 点：同步抛错说明系统服务没有正常返回，`mClient` 未赋值，系统槽没有被本对象占用。失败回执改写成"另一个自动化工具（例如 uiautomator、Appium）正占用 UiAutomation；它退出后重试"，不要让调用者去请人停执行器。
- B（更彻底，净删代码）：取消"注销未确认就封锁全部界面命令"这道门，只保留后台屏未确认销毁的门。理由：UiAutomation 登记本身就是平台级互斥，如果旧登记真的残留，下一次连接会以"already registered"明确失败，封锁门没有额外保护价值。这等于部分推翻 25h，需要用户点头。
- C：恢复基线的 `disconnectConnection` 退路，约 10–15 行，加回 1 个隐藏成员。可行但不如 A 干净。

初稿"约 10 到 20 行"的估计对 C 合适，对 A 偏大。

### 1.8 与专题 71 和初稿的出入

- 平台推导、候选控制流、修法方向：与专题 71 的 D1 一致，我独立推出的结论相同。专题 71 第 135–136、146–147 行对"封锁门是在为证明而增加成本"的判断我同意。
- 专题 71 写 `disconnectConnection`"在 `8d0416c` 整理提交里"被删，这是整理后的历史；原始删除提交是 `6fc57c3`，而且当时有替代的自愈机制。初稿写"在第四轮或第五轮收缩隐藏接口时被删掉了（上一轮候选已经如此，前两轮评审都没发现）"，因果链说错了：
  - 不是收缩隐藏接口造成的，而是删自愈机制造成的；
  - 删自愈机制正是评审方建议、用户裁决的结果；
  - 第四轮候选不卡死，第五轮候选起才卡死。

### 1.9 初稿 A3 建议改写为

> **A3. 别的自动化工具先占着系统的 UiAutomation 登记时，候选会把自己锁死，只能有人到设备上停止执行器（相对基线和第四轮候选都是回归；源码确定，未在设备上复现）。**
> - **平台行为**（AOSP 14、15、16 源码一致）：系统只允许一个 UiAutomation 登记。已被占用时，系统服务在写入任何状态之前抛"already registered"；`UiAutomation` 停在"连接中"，随后的 `disconnect()` 必然抛错。此时候选在系统侧根本没有登记。
> - **候选的处理**：连接前就标记"需要注销"（`UiBackend.kt:533`），注销失败后只记一次、不再重试（`AutomationCleanup.kt:32-41`），此后所有界面命令、新建后台屏都被 `uiCleanup` 拒绝（`Sessions.kt:250`），提示请用户到设备上停止执行器。重试关闭、新会话、空闲到期都不能解除。占用的工具退出后也不能。
> - **怎么来的**：基线能识别"其实从未登记"。第四轮候选删了这层判断，但换上了"下次登记成功即确认释放"的自愈机制。第五轮按第四轮决定第 25h 项删掉自愈，才变成永久卡死。25h 来自我第四轮报告 §6 第 9 条，当时以为这种失败只来自故障注入，漏掉了这条平台正常路径。
> - **触发**：设备上有 Appium、UIAutomator2 服务等常驻占用，或调用者用 `androidctl shell` 跑的 `uiautomator dump` 恰好与新租约的第一条界面命令重叠。候选自己占着登记时，别的工具失败，候选不受影响。
> - **修法**：连接同步抛错（不是超时）时不进入"需要注销"状态，回执写明"另一个自动化工具占用了 UiAutomation，它退出后重试"。约 5–10 行加一个 JVM 测试，不需要隐藏接口。也可以借这个机会取消"注销未确认就封锁全部界面命令"这道门（见 §6 新增题）。设备复现（§7 第 1 条）可以并行做，但修法不依赖复现结果。

---

## 2. 后台屏上的 Chrome 把几乎所有节点报成可见（初稿 B6）

### 2.1 独立结论

**现象成立。**目前最可能的解释是"后台屏特有"：同一台 API 34 模拟器、同一个 Chrome Beta，主屏页面上屏外节点被正确标为不可见。确定程度：现象【原件】【重算】确定；归因【推断】，但有同机对照。默认 `snapshot` 在这种页面上**确实打印约 44 万字节**，有真实的 stdout 原件。

### 2.2 编码规则（不靠猜）

- 设备端稀疏编码的缺省表里 `"visible" to true`（`NodeEncoding.kt:39`），等于缺省值的字段不写出（`NodeEncoding.kt:53`）；值取自 `node.isVisibleToUser`（`NodeFacts.kt:111`）。
- 主机端缺省表同为 `"visible": True`（`node_facts.py:36`），通过 `fact()` 解码（第 77–78 行）。
- 所以树里**没有 `visible` 键就等于可见**；这份树里只有 4 个节点显式写了 `visible: false`。

### 2.3 重算（脚本 `S/agents9/80/scripts/vis.py`，`python3 -I`）

- `ST/collection/r5-samples/r5-C-studio-capitals-01/artifacts.0001.tree.json`（后台屏 displayId 5，720×1280）：4,976 个节点，4,972 个可见。
- 可见节点与屏幕的关系：
  - 面积为正、与屏幕相交的有 531 个；
  - 按"右下边大于 0、左上边小于屏宽屏高"的宽松算法是 545 个（专题 66 和初稿的 546 落在这个口径附近）；
  - **4,426 个节点的边界被裁成 `[x, 1280, x2, 1280]`**，也就是零高度地贴在屏幕下沿，却报为可见。
- 同组共 10 份样本，结果如下：

  | 样本 | 节点 | 可见 | 屏内 | 说明 |
  | --- | --- | --- | --- | --- |
  | 手写路线 01–03、两份暖机 | 4,976 | 4,972 | 531 | 全部相同 |
  | 公开路线两份暖机 | 4,976 | 4,972 | 531 | 同上 |
  | 公开路线 01 | 3,236 | 全部可见 | 473 | 正式样本，采集上限截断 |
  | 公开路线 02 | 3,850 | 全部可见 | 473 | 同上 |
  | 公开路线 03 | 4,321 | 全部可见 | 488 | 同上 |

- 公开路线三份正式样本都因 `collectionLimit` 截断（`partial: true`）。屏内 531 个节点只采到 473–488 个，"可见优先"的采集顺序在这里失效。
- **同机主屏对照**：`ST/runtime/api34/verification/u33-window-lifecycle/.androidctl/snapshots/000023-…The_Python_Tutorial….json`（displayId 0，同一台 API 34 模拟器，720×1280，Chrome Beta）有 751 个节点，可见 58 个；不可见的 693 个里有 667 个同样是零高度被裁到屏幕边缘的节点。主屏上 Chrome 会把屏外节点标成不可见，后台屏上不会。
- 同机主屏上其它长网页的默认快照也都很短：GitHub 问题列表 103–109 行，Python 教程 25–82 行。
- 初稿引用的"主屏 4,977 个节点、526 个可见"来自 `ST/r5-collection-preparation.md:75`，是 **MuMu 模拟器（1080×1920）上的旧数**，不是本轮原件，设备和分辨率都不同。可以作旁证，但不该写成"同一页面在主屏上"。
- 页面状态：截图 `ST/runtime/api34/images/r5-C-studio-capitals-ready.jpg` 显示页面在顶部、没有滚动，是桌面版布局（URL 带 `toggle_view_desktop`）。所以不是滚动造成的。

### 2.4 默认输出到底多大

- **重算**（脚本 `S/agents9/80/scripts/render.py`，`S/r6/venv6` 的主机代码，调用 `ui_view.project` 和 `ui_view.render`，设了 `PYTHONDONTWRITEBYTECODE=1`）：
  - 4,976 节点的样本，节点正文 4,028 行、434,652 字符；
  - 公开路线三份截断样本分别是 327,068、366,469、392,163 字符。
- **原件**：`ST/runtime/api34/commands/2712-r5-surface-collection-C-provider-page/stdout.bin`。命令是不带 `--no-print` 的 `snapshot --screenshot`，**stdout 实际 440,511 字节、4,034 行**；对应保存的观察文件 `…/r5-collection-C-api34/.androidctl/snapshots/000003-….txt` 有 4,033 行、436,130 字符。
- 缺省上限：`--max-chars` 默认 `None`（`ui_commands.py:237-244,518`）；`--depth` 默认 `None`；`reading_view` 在四个过滤条件都为空时原样返回（`ui_view.py:804-805`）；正文默认打印（`ui_client.py:759`）。主机投影只隐藏 `visible == false` 的节点（`ui_view.py:363`）。**没有任何缺省截断。**

### 2.5 判断

- 是不是后台屏特有：同机同 Chrome 的主屏对照支持"是"。没能排除的混杂因素是页面不同（主屏对照是 Python 教程，不是首都列表），也没有其它 Chrome 版本或真机的数据。【推断】
- 是不是 Chrome Beta 特有：无法判断，全部样本都是 Chrome Beta。
- 四道固定任务里没有"后台屏加网页"的组合，所以没测到真实影响面。但调用者在后台屏上读任何长网页，一次默认 `snapshot` 就是十万 token 量级，足以撑爆上下文。

### 2.6 修法提示

- 最小修法：把"边界在显示区域内的面积为零"也当作屏外。主屏上 Chrome 自己就是这样标的，两种显示上的行为可以因此统一。
- 改在哪里：如果只改主机渲染，只影响默认打印，约 5 行。如果改在设备端 `NodeFacts`，还会影响选择器、条件、"目标不可见"的拒绝和"可见优先"的采集顺序，范围更大，需要补测试。
- 不论哪种，都建议给默认 `snapshot` 加一个大输出保护：超过某个阈值时改为写文件并只打印摘要。这是独立于本缺陷的防线。

### 2.7 与专题 66、初稿的出入

- 计数和渲染结果一致（546 与 531 只是边界口径不同）。
- 出入有两点：
  - "主屏 526 个可见"的来源要改，应换成同机对照；
  - "可能收到十万 token"可以改成实测：已有一次真实默认输出 44 万字节的原件。
- 专题 66 已指出公开路线因此截断，我同意。

### 2.8 初稿 B6 建议改写为

> **B6. 后台屏上的 Chrome 把屏外节点也报成"可见"，默认 `snapshot` 一次打印约 44 万字节（原件；归因需设备确认）。**
> - **事实**：后台屏上一份维基百科长列表页（720×1280，页面在顶部）的树有 4,976 个节点，其中 4,972 个标为可见；有 4,426 个节点的边界被裁成贴在屏幕下沿的零高度，仍报可见。真正落在屏内的约 530 个。同一台 API 34 模拟器、同一个 Chrome Beta 在主屏上，屏外节点被正确标为不可见：Python 教程页 751 个节点只有 58 个可见。实验里一次真实的 `snapshot --screenshot`（没有加 `--no-print`）打印了 440,511 字节、4,034 行。主机没有任何默认的字符或深度上限。公开采集路线的三份正式样本也因此在采集上限处截断，屏内节点只采到约九成。
> - **影响**：在后台屏上读长网页，一次默认观察就是十万 token 量级。四道固定任务没有覆盖这种组合。
> - **修法**：把"边界在显示区域内面积为零"也当作屏外处理。只改主机渲染约 5 行；改设备端会同时影响选择和采集顺序，需要补测试。另外建议给默认输出加一个过大时转存文件的保护。设备确认（§7 第 3 条）用于确定是不是只在后台屏发生，修法本身对两种显示都无害。

---

## 3. 引用因名字变化被拒共 7 次（初稿 B1）

### 3.1 独立结论

**成立，所有数字都对上。**【原件】【源码】确定。

### 3.2 逐次列表（脚本 `S/agents9/80/scripts/stale.py`，扫描 `ST/efficiency/*/cli-index.json` 和每条命令的 `child.stdout`、`child.stderr`，37 个运行目录全扫）

| 运行 | 序号 | 命令 | 拒绝文字 | 回执字节 | 下一条命令 |
| --- | --- | --- | --- | --- | --- |
| 基线 Maps 02 | #4 | `click e705072637 --print` | `click: not sent; No current target matches the original ref signature; code=staleRef; dispatchStarted=false; timing=before-action; snapshotPath=…` | 4,589，附带新树 | `click e705072772 --print`，直接点新引用，成功 |
| 基线 Maps 03 | #4 | `click e481016079 --print` | 同上 | 4,839，附带新树 | `click e481016215` |
| 基线 Maps 04 | #5 | `click e391062384 --print` | 同上 | 4,827，附带新树 | `click e391062519` |
| 基线 Maps 05 | #5 | `click e186313298 --print` | 同上 | 4,840，附带新树 | `click e186313433` |
| 候选 Maps 01 | #4 | `click e124387726` | `click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action` | 141 | `snapshot`（7,037 字节），然后 `click e124387857` |
| 候选 Maps 02 | #4 | `click e947004935` | 同上 | 141 | `snapshot`（7,102 字节），然后 `click e947005043` |
| 候选 Maps 03 | #4 | `click e563611054` | 同上 | 141 | `snapshot`（7,117 字节），然后 `click e563611186` |

- 7 次全部是同一处：步行标签页的名字从 `"Walking mode: "` 变成 `"Walking mode: 18 minutes"`，`rowIndex=0 columnIndex=2`、`roleDescription="Tab"` 不变（候选第 3、5 条命令的输出里逐次核对过）。基线回执里的新树中就有 `e705072772 name="Walking mode: 18 minutes"`。
- 只出现在 Maps 路线题。GitHub、Feeder、后台屏 Contacts 三道题的两侧都没有，不计分的历史运行里也没有。
- 候选第 4、5 次没撞上：第 4 次在点击前先做了 `snapshot`，第 5 次用了新采的 `find Walking`。
- "候选 3 次、基线 4 次、只在路线题、候选每次多一次 snapshot、基线直接点新引用"全部成立。

### 3.3 拒绝条件和新树

- `UiRefRepair.kt:96-98`：两边名字都完整，且都非空又不相同时，直接排除这个候选。这里新旧名字都非空且不同，所以排除；没有剩下的匹配，第 72 行抛 `stale`。
- 拒绝时手里**已有新采的完整树**：第 36–39 行在拒绝前调用 `capture()`（`UiActions.kt:99,109,172` 传入 `ui.capture(run)`）。不完整时第 62 行报的是 `identityUnknown`，而不是 `stale`。`select` 是在这棵新树上做的，节点也已经编了引用。所以"在拒绝信息里附一行同位置的新节点"不需要额外采集，成本很低。新树有没有作为可读缓存交给主机，我没有逐行核对。

### 3.4 与专题 66、73 和初稿的出入

没有实质出入。初稿"实施方只统计了 3 次"我没有核对实施方的计数来源。初稿"不建议放宽名字规则"的判断我同意："关注"变成"已关注"这类变化确实应该挡住。

### 3.5 初稿 B1 建议

基本保留。在"修法"后补一句："拒绝前修复逻辑已经采到一棵完整的新树（`UiRefRepair.kt:36-39`），新节点的引用也已编好，所以附一行候选引用不需要再采一次。"

---

## 4. 版本握手与中间提交（初稿 A4、A5）

### 4.1 握手（A4）

**结论：成立，但症结是版本号没有升，不是"只比字符串"。**【源码】确定；混用后的症状【推断】。

- 主机：`transport.py:234` 在请求头里发送 `X-AndroidCtl-Version: VERSION`；`transport.py:274` 要求设备的欢迎消息 `version == VERSION`，否则报 `protocolMismatch`，并提示用 `androidctl apk --out` 取匹配的 APK（第 260–262 行）。
- 设备网关：`NetworkGateway.kt:66` 要求请求头里的版本与 `BuildConfig.VERSION_NAME` 相等，否则拒绝并发 `protocolMismatch`（第 98–100 行）。
- worker：`WorkerConnection.kt:75` 要求 worker 自报的版本与 App 相等，而且 UID 必须是 shell；`WorkerConnection.kt:57` 用 `VERSION_CODE` 绑定。
- 来源：`VERSION` 文件在基线和候选都是 `0.4.0`；Gradle 只接受 `MAJOR.MINOR.PATCH`（`android/app/build.gradle.kts:14-35`），`versionCode = major*1e6 + minor*1e3 + patch`，所以两边的 versionCode 也相同。
- **没有独立于产品版本的协议版本号**；`contract/wire-contract.json` 是样本集合，不带版本字段。
- 后果：
  - 新主机配旧 0.4.0 APK，或旧主机配新 APK：握手两边都通过，之后的方法名、参数和原因码不兼容，用户看到零散的参数或方法错误。具体症状没有设备，无法验证。
  - 升到 0.5.0 之后：两个方向都会在握手时明确拒绝，并给出取匹配 APK 的提示。
- 判断："严格相等加同版本锁步"对一个人维护的 alpha 产品是合适的设计，不需要另加协议版本号。修法就是改一行 `VERSION`（再加发布说明）。实施方没改是因为没有发布授权（专题 75 引 `decisions.md:736`），这是用户的合入步骤。
- 初稿 A4 的事实对。建议把"握手只比较版本字符串是否相等"改成"握手要求两端版本完全相等，这个设计本身没问题，但两边都还是 0.4.0，所以检测不出混用"，避免读者以为要重做握手。

### 4.2 中间提交（A5）

**结论：成立。**【实跑】确定。

- 做法：`git -C SRC archive <提交> androidctl skills pyproject.toml VERSION contract | tar -x -C S/agents9/80/c-<提交>`，没有动 SRC。然后用 `S/r6/venv6/bin/python`，设 `PYTHONDONTWRITEBYTECODE=1`，执行 `sys.path.insert(0, 导出目录/androidctl/src)`，确认导入的确实是导出树，再 `import androidctl.app`，并跑 `python -m androidctl --help`。
- 9 个整理提交（最早的在前）的结果：

  | 序号 | 提交 | `import androidctl.app` 和 `--help` |
  | --- | --- | --- |
  | 1 | `9d9d3b8` | 通过，`--help` 正常 |
  | 2 | `d01d885` | 失败：`NameError: name 'UiCommand' is not defined`（`ui_commands.py:504`） |
  | 3 | `8d0416c` | 失败：同上 |
  | 4 | `d351e43` | 失败：`NameError: UiCommand`（`ui_commands.py:304`） |
  | 5 | `8adb58d` | 失败：`NameError: name 'SCROLL_DIRECTIONS' is not defined`（`ui_commands.py:757`） |
  | 6 | `b29ea1f` | 失败：`NameError: name 'clipboard' is not defined`（`ui_commands.py:249`） |
  | 7 | `010b14f` | 失败：同上 |
  | 8 | `dc22cef` | 失败：`NameError: name 'TyperGroup' is not defined`（`ui_commands.py:129`，模块顶层） |
  | 9 | `fb75afc` | 即最终树，通过 |

- 任务点名的第 2、5、8 个都在导入阶段失败。和专题 75 的结果一致。Kotlin 侧我没有检查。
- **与初稿的出入**：初稿写"实施方说过'不要求中间提交可用'，但没有说到这个程度"。原话其实是**用户在 U33 写的**："从最终树按子系统集中组织提交，以 6–10 个为目标，但不强求固定数量，也不强求中间提交各自可用"（`PKG/context/authority/user-136340.json`）。所以这不是实施方违规，也不该算作"缺陷"。它的实际含义是：只能整体合入，用压成一个提交或 `--no-ff` 合并加 `bisect --first-parent`；不能逐个提交审合、二分或回退单个子系统。
- 初稿 A5 建议改写为："**A5（合入方式的约束，不是缺陷）**：用户在 U33 里允许中间提交不可用。实跑显示第 2 到第 8 个提交连主机命令行都导入失败，因此只能整体合入（压成一个提交，或用 `--no-ff` 合并、日后按第一父提交二分），9 个提交只作分章节的阅读材料。见 §6 第 1 题。"

---

## 5. 成功的按键与滚动查找仍打印过程计数（初稿 A2）

### 5.1 独立结论

**成立，属于"没落实"，裁决本身没有歧义。**【原件】【源码】确定。量很小。

### 5.2 统计

脚本是 `S/agents9/80/scripts/proc.py` 和后续内联统计，只统计 4 组计分的候选运行，共 20 次：

- **成功的 `back`**：17 次，17 次首行都带 `keys={"plannedEvents":2,"attempted":2,"returned":2,"successfulPrefix":[0,1],"syncReturned":2,"lastReturned":{…"downTime"…,"eventTime"…,"callTime"…},"inFlight":null,…,"omitted":{…}}`。这一段合计 7,072 字节，每次约 416 字节，约占这些首行 14,143 字节的一半。计分运行里没有出现 `home`、`recents` 和 `key`；源码上它们走同一路径。
- **成功的 `scroll-until`**：12 次，全部带 `completedSteps=N` 和整份 `step={"command":"scroll-down","result":"done","reason":null,"data":{…},"index":N}`，两者合计 2,280 字节，每次约 190 字节。另有 1 次未完成，带这两项理所应当。
- 如果把不计分的两次历史运行也算进去，`back` 是 20 次、8,324 字节（专题 68 的"8.3 KB"就是这个口径）。

### 5.3 源码

- `ui_output.py:24-27`：`concise_gesture` 只对 `tap`、`swipe`、`drag`、`pinch`、`timeline` 五个命令、且结果为 `done` 时生效。
- `ui_output.py:463-476` `input_facts`：`keys` 字段对其它所有命令无条件打印。
- `ui_output.py:155-168`：`scroll-until` 成功时也打印 `completedSteps` 和 `step`。

### 5.4 裁决原文与指引

- `androidctl-r5-decisions.md` 第 15 题："过程计数（动作回执里记'做到了第几步'的字段，例如手势的逐事件进度）从默认文本里去掉，在 `--json` 里保留……这与评审报告 B12'进度只在 `--json` 或未完成时打印'一致。"
  - 这里先给出一般定义，手势只是"例如"。
  - `keys` 里的 `plannedEvents`、`attempted`、`returned`、`successfulPrefix` 正是"做到了第几步"。
  - `scroll-until` 的 `step={…}` 是最后一步的过程回执，同属此类；`completedSteps` 处在边界上，成功时可以留一个整数。
- 公开指引自己也这么写：`skills/androidctl/reference.md:81-82` "Normal text abbreviates known successful dispatch/progress; JSON keeps complete available facts"。实现与指引不一致。
- 所以是没落实，而不是裁决有歧义。手势的例子可能诱导了窄读，但定义和指引都足以排除这种读法。

### 5.5 与专题 64、68、70、74 和初稿的出入

各专题的结论一致；初稿的"17 次、12 次"也与我的计分口径一致。初稿"每次约 400 到 600 字节"可以改成"约 420 字节"。

### 5.6 初稿 A2 的定位建议

- 事实成立，修法约 5 行：成功时省略 `keys`、`inputTarget`、`step`，`completedSteps` 可保留；未完成时照旧。
- 但影响很小：20 次计分运行一共约 9 KB，约 2,300 token，相对每回合 1.73 万 token 的固定前缀可以忽略。
- 建议在 §4.1 里注明"这是未落实的裁决，顺手修，不是阻碍合入的实质缺陷"，或者挪到 4.2。这样 §4.1 里真正有分量的只剩 A3 和 A1。

---

## 6. 需要补进初稿的新取舍（只列一项）

**UiAutomation 的"注销未确认就封锁全部界面命令"要不要保留？**（第四轮第 25h 项的局部重议）

- A（推荐）：保留封锁门，只修 1.7 的 A——连接同步抛错时不进入待注销。改动最小，不推翻 25h。
- B：取消 UiAutomation 这一项的封锁门，下一次连接自己去撞平台的互斥，失败时报平台原因；后台屏未确认销毁的封锁照旧。净删代码，远程调用者永远不会因为 UiAutomation 被锁死；代价是"注销确实失败、系统里真有残留"这种极端情况下，回执从"请到设备上停止"变成连接报"已被占用"。两者都需要人处理，但 B 的报错更准确。
- 剩余未知：真机和厂商 ROM 上 `disconnect` 是否另有失败方式。

## 7. 有判别价值的补验

1. 模拟器上循环跑 `adb shell uiautomator dump`，同时新开租约发第一条界面命令；再在 dump 停止后发界面命令。预期是第一条报"already registered"，之后全部 `uiCleanup`。如果平台行为不同，A3 才降级。修法 A 不依赖这个结果。
2. 同一个长网页（最好就是首都列表页）在同一台模拟器的主屏和后台屏各取一次 `--raw`，比较 `visible` 计数和零面积节点数。如果后台屏仍然几乎全为真，就确认 B6 是后台屏特有。

## 8. 没能核实的内容

- 任何设备行为，包括 A3 的复现、后台屏 Chrome 的可见位归因，以及新旧版本混用后的具体报错。
- 设备上"Stop idle executor"之后，worker 怎样重新附着。
- 各中间提交的 Kotlin 侧能否编译（只看了主机侧）。
- B1 修复时采到的新树有没有作为主机可读的缓存发布出去。
- 初稿"实施方只统计了 3 次"的出处。
