# 专题 29 核查：通知、会话与后台屏、资源生命周期与清理

核查对象：实施方 r3 主稿（`T/deliverables/route-review-r3-recommendation.md`，下称"主稿"）主线第 1、2、9、10 条，第 6、7、8、11 题，建议提交 1、2、12、13，完整处置底表（`P/r2-delivery/integration-draft.md`）的 G3、G5 后半、G6。

路径简写：`S` = `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad`；`SRC` = `S/review-copy`（候选 HEAD 3f45af6），Kotlin 路径省略到文件名；`ROOT` = `S/pkg6/root`；`T`、`P`、`R` 同 COMMON.md。我的脚本和下载在 `S/agents6/29/`。

证据标签：【原件】我亲自读过的回执、日志、heap；【源码】给出 `文件:行` 和版本；【重算】我自己写脚本复算；【推断】；【需设备】。所有运行结论只来自模拟器：Android Studio API34 AVD（`sdk_phone64_x86_64 … 14/UE1A.230829.036.A1 userdebug`）或 MuMu（API35，fingerprint 伪装成 `vivo/PD2362 … user/release-keys`），没有真机数据。

---

## 1. 结论摘要

- **第 6 题（监听器注册寿命）：改选 C（worker 首次使用时注册，worker 死亡时由平台自动回收），B 不值得做。** API34 上每注册一次，system_server 的强 JNI 全局引用就多一条，2→6 我独立重算成立。system_server 全局引用表上限是 51,200 条，溢出时 `LOG(FATAL)`，整机软重启。按每命令注册，正常 LLM 用量要几个月才会撞上限，但持续轮询的脚本一两天就能到。C 只注册一次，代码最简单，也不需要"注销结果未知"的整套状态。主稿给 B 的"权限窗口"理由，对本来就是 shell UID 的 worker 分量很小。保留 A 作为用户可选的保守方案。
- **第 7 题（后台屏 Activity 通知的主结果）：这题不该问用户。** 用户已有方向（同一把钥匙，加上不新增内部接口）直接给出答案：发送前用"创建者包＋UID 在别屏有任务就拒绝"，发送后在**动作后观察结束时**核对"创建者包＋UID 在目标屏有可见任务"；对不上就返回 UNKNOWN/3，并列出实际可见的包。这和启动 App 时"落点未确认就停下"的既定规则一致。B 要加 `PendingIntent.getIntent` 和 `queryIntentComponents` 两个隐藏接口，只能作为用户明确批准的例外。A（只报一次 send）与启动 App 的规则矛盾，不推荐。
- **第 8 题（后续动作是否监测迁入）：同意 A，这题不需要问用户。** VLC 原件核实无误，样本只有 1 次、1 个 App。评审方上一版把"每次动作后查任务归属"列为备选，现在应彻底放弃。
- **第 11 题（不可确认的清理失败怎样恢复）：A 的方向对，但有两处要修正。** 一是"其余本机 Stop"在候选里走不通：失败后留下的那份清理资源会让正常 Stop 也被拒，只能用本机"强制结束"。二是 UiAutomation 阶段其实可以确认：系统里只有一个 UiAutomation 注册位，新建一次连接成功，就证明旧注册已释放，而且不需要新接口。采用这个办法后，A 的死路只剩极窄的情形，不需要 C（远程终止 worker）。
- **会话（主线第 1 条）：** 显式 open、同种复用、异种显式 replace 都同意。`wake` 不应借用或自建 UiAutomation 连接。改用候选已有的 `IInputManager.injectInputEventToTarget`，加上轮询公开的 `PowerManager.isInteractive`，就能删掉"wake 短期 owner"和"首次 virtual 不得复用 wake 连接"这两项状态。"最近结束的 kind"记录不是必需的。
- **system_server 重启后的陈旧 Binder：应列为确定缺陷，不是"未知"。** 原件证明 worker 和 Shizuku 在 system_server 被替换后仍然存活。按源码推断，后果是 UI 命令失败，或者会话到期后触发全局封锁，最后只能本机强制结束 worker。最便宜的修法是 worker 监听系统服务的死亡、随即自行退出。
- **建议提交 1（提前关闭 writer）：同意，可以先独立做。** errno 26 是 ETXTBSY；在 vboxsf 上，这是宿主端共享冲突映射过来的错误码。小建议：把 close 放到 dev/ino 身份核验之前。
- **建议提交 2、12、13：** 按以上结论收窄，见第 8 节。

---

## 2. 逐题分析

### 2.1 第 6 题：通知监听器注册寿命

**实施方主张。** API34 上同一 Binder 累计注册 2→6 次，强 JNI 全局引用也 2→6，注销不解除。推荐 B：在现有 Lease（设备租约）内首次使用时注册，租约结束时注销；注销结果未知时责任归 worker，不重复注册。API35 heap 导出被拒，强弱数量未测。

**核到的事实。**

1. **2→6→0 成立，测法可信。** 【原件＋重算】我没有用包里的解析器，自己写了一个最小 HPROF 解析器（`S/agents6/29/hprof_check.py`，用 `python3 -I` 运行），读四份 sparse 摘录：

   | 摘录 | 全部 JNI_GLOBAL | 全部 ManagedServiceInfo | 目标组件的 info | 目标 info 的 JNI_GLOBAL | 指向的 BinderProxy |
   |---|---:|---:|---:|---:|---|
   | method（另一次系统启动，不是零基线） | 1777 | 9 | 0 | 0 | — |
   | reg2 | 1754 | 11 | 2 | 2 | 都是 0x140e5bc0 |
   | reg6 | 1758 | 15 | 6 | 6 | 都是 0x140e5bc0 |
   | exited（probe 进程退出后） | 1757 | 16 | 7 | 0 | 0x140e5bc0 |

   探针原始输出（`P/r2-notifications/runtime-api34-recovered/probe-process/stdout.bin`）与之对应：同一 PID 2238、同一 binderIdentity；注册返回数依次为 1、2、（注销后 cycle）3–6、7；注销后 read 被 `SecurityException: Disallowed call from unknown notification listener` 拒绝。heap 用 `am dumpheap` 导出，没有加 `-g`。
   - 能证明到什么程度：这批摘录本身内部一致，独立解析器得到同样的数。但摘录不是完整 heap，接收方无法证明完整 heap 里没有遗漏，README 已如实说明。一次运行，一个 AVD 镜像。
   - 机理与源码吻合。【源码 AOSP14】`ManagedServices.java:1706-1716`：每次注册都新建 info 并 `linkToDeath`，全文没有 `unlinkToDeath`。`android_util_Binder.cpp:566`：JavaDeathRecipient 用 `NewGlobalRef` 持有 info；`:599-603`：`binderDied` 之后改成弱引用。exited 一行"7 个实例、0 条强引用"正是 worker 死亡后强引用降为弱引用的样子。
2. **每次泄漏的大小和释放时机。** 【重算＋源码】从 reg6 摘录量到：info 实例 41B 字段，ComponentName 16B，`INotificationListener$Stub$Proxy` 包装 12B，另有两个字符串（17 字符和 61 字符）；ComponentName 和包装对象每次注册都新建（`S/agents6/29/sizes.py`）。加上对象头、native 的 JavaDeathRecipient 和链表节点，估计每次注册约 0.3–0.4 KB，外加 1 条全局引用槽位【推断】。释放时机：worker 进程死亡时 `binderDied` 把强引用降为弱引用（A14 `android_util_Binder.cpp:578-605`），再随 BinderProxy 回收。exited 摘录证明强引用归零。**worker 活着期间不会释放。**
3. **全局引用表上限。** 【源码 ART android-14.0.0_r1，包内固定副本 `P/r2-notifications/upstream/android-14.0.0_r1-art-runtime_jni_java_vm_ext.cc`】`:63` 定义 `kGlobalsMax = 51200`；`:526` 用它初始化；`:689-705` 的 `AddGlobalRef` 在表满时 `LOG(FATAL)`。也就是说，system_server 会 abort，整机软重启。本 AVD 的基线约 1,760 条，余量约 49,400 次注册。真机上应用和服务更多，基线会更高【推断】。
4. **现实用量会不会出问题。** 【推断】每命令注册（A），一次通知交互大约 2–3 次注册（list/get/写），worker 是常驻的 Shizuku daemon（`WorkerConnection.kt:55`），可以连续运行多天：
   - 普通 LLM 使用（每天几十到几百次）：几个月到数年才会到上限，实际上不会撞到。
   - 每 2–3 秒轮询一次通知的循环（等验证码之类）：约 1.5–2 天撞上限。皮肤文档没有推荐轮询（`skills/androidctl/notifications.md`），但没有任何东西阻止调用者这样做。
   - 结论：A 有一个低概率但后果严重的整机崩溃路径，而且只发生在最低支持版本 Android 14 上。这一项足以让"注册次数"成为真实考虑，但也不该夸大成常见故障。
5. **API35 没测到属实。** 【原件】`P/r2-notifications/runtime-mumu-lifetime/native/heap-a-dump-once/stderr.bin` 记录的是 `SecurityException: Process not debuggable: android`（`ActivityManagerService.enforceDebuggable`），exit 255；该镜像是 user 构建。按 A15 源码，弱引用分支在构建开关打开、目标 SDK≥V 时生效，注销加 GC 之后可能不再累积，但这一点没有测到。

**三种寿命各自要维护的状态和接点。**【源码 SRC】
- `Sessions.kt:48` 只有一个 `current` 租约。所有结束路径（close、replace、takeover、本机 release、revoke、到期）都经过 `invalidateLease`→`clearLeaseUi`（`:281-284`），落到 `UiBackend.clear`（`UiBackend.kt:477`）。所以 B 的"挂注销"实际上只需要一个接点，不是主稿说的六处。worker 停止时不需要注销：进程死亡后，NMS 的 `binderDied` 会移除登记（A14 `ManagedServices.java:1846-1852`）。
- B 真正的复杂度在"注销失败或注册结果未知"这一支：要转交 worker 持有、保留来源、不能重复注册、换了新租约还要区分"正常登记"和"遗留登记"、`status` 要分开表述（`R/r2-notifications.md` §4 的表）。这是一套跨租约的小状态机，但它只为了把一个本来就是 shell 能力的监听窗口收窄到租约内。
- C：一个 worker 级标志"可能已登记"。第一次通知命令时注册；注册抛错就保持"可能已登记"并报告；下一次先读，读被"unknown listener"拒绝再注册（最坏多一次注册）。不需要注销，也不占 UI 资源。
- A：保持现状的"单对象、每命令注册/注销"，注销失败只诊断，不封锁。

**长期持有的实际代价（C）。**【源码＋推断】
- 回调：`INotificationListener` 是 oneway。候选的 Binder 对所有回调只做 `enforceInterface` 就返回（`NotificationAccess.kt:68-83`），不取 SBN、不解码。每条通知事件多一次 oneway 事务，代价很小。
- URI 授权：注册期间，NMS 会把活动通知附件 URI 的读权限授给 `com.android.shell`。worker 本来就能在任何时候注册，这项授权并不扩大远程调用者的能力；受影响的只是同 UID 的其它进程（adb shell），而它们本来就属于开发者。
- 隐私姿态：`dumpsys notification` 会一直显示一个 shell 监听器。这是观感问题，不是能力变化。
- A16 特有：`NotificationManagerService.java:13131`（android-16.0.0_r1）在 `lifetimeExtensionRefactor` 开关下把持有 STATUS_BAR_SERVICE 的 shell 也标成 `isSystemUi`；`:13534-13537` 对直接回复的生命周期更新只发给**第一个** `isSystemUi` 监听器，发完即 `break`。只有 SystemUI 在我们注册之后重启、排到我们后面时才会出问题，注册窗口越长，碰上的机会越大【源码＋推断】。A 和 B 也有同样的风险，只是窗口短。
- "权限窗口"论点的分量：worker 本身就是 shell UID，随时能注册；远程调用者的每条命令仍逐次校验当前 owner。窗口长短不改变远程调用者能拿到什么，只改变系统里登记存在多久。所以分量小。

**判断。** 推荐 C，A 作为保守备选；B 的收益（窗口收窄到租约内）配不上它的未知状态复杂度。是否接受"一直显示一个 shell 监听器"，可以交给用户，所以第 6 题保留，但候选应改成 A/C 二选一，推荐 C。

### 2.2 第 7 题：后台屏会话里 Activity 类通知动作的主结果

**实施方主张。** 推荐 B：用 `getIntent`/`queryIntentComponents` 取预期目标，与当次可见任务比对；同时承认这要新增两个内部接点。

**核到的事实。**
- **B 新增的两个接口。** `PendingIntent.getIntent()` 和 `PendingIntent.queryIntentComponents(int)`，两者都是隐藏接口；系统端 `ActivityManagerService` 都要求 `GET_INTENT_SENDER_INTENT` 权限。【源码】shell 在 A14/15/16 的清单里都声明了这个权限（`S/agents5/24/aosp/android-14.0.0_r1-AndroidManifest.xml:359`、15:370、16:381）。【原件】API34 上对 Termux 的一次实调成功（`P/r2-notifications/runtime-vd-api34/native/getter-once`，实施方 result.md 记录：显式组件 TermuxActivity、一个候选、UID 10145）。B 与用户"原则上不新增内部接口"的方向直接冲突，主稿自己也承认这一点。
- **Termux 例子是同包。** 【原件】`c-positive-send-once/stdout.bin`：creatorPackage=com.termux，落点 com.termux/TermuxActivity，UID 10145。也就是说，用创建者包当钥匙，这个例子同样会判为 DONE。实施方没有任何跨包样本。
- **三个时间点。** 【原件】
  - send 开始于 17:34:47.30；send 返回后约 16 ms 的 afterTasks 里，task201 在 display3，`visible=false`。
  - 17:34:47.92 开始的 `dumpsys activity`：task201 `visible=true`，`topResumedActivity` 是 TermuxActivity。
  - 17:34:48.14 开始的 `dumpsys window`：启动画面窗口已经 `HAS_DRAWN`、`isVisible=true`，TermuxActivity 窗口还是 `DRAW_PENDING`、`isVisible=true`。
  - 17:35:39 的截图才看到 Termux 画面。
  - 含义：任务级的"可见"在一秒内就成立，像素要更晚。所以核对应在**动作后观察结束时**做，用任务的可见性加顶层包/UID，而不是 send 刚返回时的快照，也不是窗口绘制状态。这与启动 App 时用"观察结束时的完整任务样本"判断落点的规则一致（`R/r2-sessions-display.md` §3.4）。注意：候选现在的通知写操作没有动作后观察（`NotificationActions.kt` 里没有 settle），要加上，并计入成本。
- **hidden-live 冲突零 send 属实。** 【原件】`e-conflict-zero-send/stdout.bin` 记录 `preflight.state=conflict`，冲突任务是 task202（display0，visible=false，TermuxActivity，UID 10145），`sendCallCount=0`，`displayOptionAdded=false`。这是研究探针的结果，不是产品代码路径。

**选项之外的做法（建议采用）：用创建者包＋UID 当预期，发送前后用同一把钥匙。**
- 发送前：创建者包＋UID 在别屏有活任务就拒绝（已定）。发送后：观察结束时，检查目标屏上有没有顶层为"创建者包＋UID"的可见任务。有就 DONE，并报告实际匹配到的任务。没有就 UNKNOWN/3，停 batch，列出目标屏上实际可见的包和 UID。只用公开 API：`getCreatorPackage`、`getCreatorUid`、`isActivity`；任务来源复用启动 App 已有的接口。
- **会误报 UNKNOWN 的真实情形**【推断，无样本】：
  1. 跨包 PendingIntent：应用通知打开浏览器网址（创建者是应用，落点是 Chrome）；系统通知（创建者 `android`/SystemUI，UID 1000）打开设置；`createChooser` 先落到系统选择器。系统通知这一类不算少见，但调用者在后台屏里操作系统通知本来就少。
  2. 跨包 trampoline（创建者的 Activity 转发给别的应用）：少见。Android 12 以后，面向 S+ 的应用不能用 broadcast/service 做通知 trampoline。
  3. alias：`activity-alias` 必须指向同包，所以按包核对不受影响；这一类不会误报。
  4. 多 Intent 栈（getActivities）：通常同包。
  - 发生频率没有数据。按常识，绝大多数应用通知打开的是自己的界面。这些情形下 UNKNOWN 是诚实的："已发送，落点不是创建者包，实际看到的是 X"。调用者拿到这个信息足以决定下一步。
- **误报 DONE 的风险**【推断】：创建者早就在目标屏可见，而这次 PI 是跨包、且新界面没有盖住它（例如透明 Activity）。B 也有类似的"复用即 DONE"问题（主稿："允许有效 task 复用"）。缓解办法：把目标屏上的全部可见任务都放进回执。
- **与"未确认就停下"的规则一致**：UNKNOWN/3 停批。A（只报 send）会在落点未确认时让 batch 继续，与启动 App 的既定规则不一致。
- **与 B 的比较**：B 在跨包和系统通知上能给出 DONE，代价是两个隐藏接口，加上多 Intent、alias、候选歧义、取值失败等解释代码，而且 B 同样不证明最终 handler（getter 审计 `P/r2-notifications/intent-getter-audit/result.md` 已写明）。B 也不在发送前检查"目标包在别屏的任务"，跨包时发送前检查的盲点两种方案一样。

**判断。** 用户已有的两条方向已经决定了做法，这不是需要用户重新裁决的取舍。B 只能作为"新增内部接口例外"单独申请；在没有跨包需求证据前不建议申请。

### 2.3 第 8 题：后台屏会话后续动作是否监测别屏任务迁入

**核到的事实。**【原件，`P/r2-sessions/runtime-attempt-2/commands/`】
- `verified-prestate-media`：VLC 媒体会话 `PAUSED(2)`，position=63654。
- `after-click-media` 和 `after-late-media`：`PLAYING(3)`，position=58906。位置回退，与重新加载一致。
- 三个时点的 `dumpsys activity` 里，task198（VideoPlayerActivity，主屏 A）和 task199（MainActivity，后台屏 B）记录完全相同，没有迁移。
- 样本：1 次点击、1 个 App、API34 AVD。实施方没有夸大，承认未触达 singleTask 内跳。

**判断。** 同意 A。这个反例里，最重要的副作用（共享播放服务重新播放）根本不涉及任务迁移，B/C 的任务归属检查发现不了，却要引入每动作查询和跨动作台账。评审方上一版把"每次动作后查任务归属"列为备选（以及 C9 里"主屏任务被搬走即确认"的说法），现在应彻底放弃，改为在后台屏文档里写明：任务、进程、播放服务、数据都是共享的，后台屏不提供隔离。用户此前"不新增普遍监控"的方向已经覆盖这一点，所以这题不需要再问用户。

### 2.4 第 11 题：不可确认的清理失败怎样恢复

**"第二次 disconnect 可能只是本地 no-op"：成立，14/15/16 一致。**【源码，自行下载 android-14/15/16.0.0_r1 到 `S/agents6/29/aosp/`】
- `UiAutomation.disconnect()`（A14:433、A15:437、A16:438）：先把状态置为 DISCONNECTED，再调用 connection.disconnect。第二次调用时命中 `useAccessibility() && DISCONNECTED`，直接 `return`。
- `UiAutomationConnection.disconnect()`（A14:128、A15:126、A16:124）：先把 `mOwningUid` 置为 -1，再做远端注销。直接对 connection 重试会被 `throwIfCalledByNotTrustedUidLocked` 拒绝；这对 shell UID 成立，如果 Shizuku 以 root 运行，root 例外可以通过【源码推断】。
- 候选里还有一个更具体的陷阱：`UiBackend.disconnect()` 只有在成功时才执行 `automation = null`（`UiBackend.kt:503-521`，赋值在 finally 之后，`:518`）。失败后 `automation` 仍然非空，"再跑一次 clear/disconnect"就会命中本地 no-op，被当成成功。主稿的警告是对的。

**封锁门在什么失败后锁住什么。**【源码 SRC】
- 置位来源：`UiBackend.clear` 中任一步失败（`:477-499`）、连接失败后的断开失败（`:466-472`）、未发布 VD 关闭失败（`TargetManagement.kt:83-95`）、通知注销失败（`NotificationOperations.kt:155`）。
- 锁住的范围：
  - 所有 UI 命令，包括只读的（`Sessions.kt:144`）。
  - 会话切换（`TargetManagement.kt:46`）。
  - 失败时留在 `resources` 里的 closing UI 资源，还会让新的 shell 和文件任务被拒为 busy（`Sessions.kt:188-202`、`ShellJobs.kt:33-34`、`FileJobs.kt:24-25`），并让到期永不执行（`:241-244`）。
  - **正常 Stop 也会失败**：`prepareStop`→`sessions.stop()`→`requireIdle()`（`WorkerService.kt:115-123`、`Sessions.kt:232-235`）。只剩本机管理界面的"强制结束"（`WorkerConnection.kt:238-247`，经 `MainActivity.kt:195-200`）。主稿第 11 题 A 写的"本机 Stop worker"需要改成"本机强制结束"。

**UiAutomation 阶段其实可以确认（选项之外的做法）。**【源码 android-14/15/16 `UiAutomationManager.java`】
- 系统里只有一个 UiAutomation 注册位：位置被占时，注册直接抛 `"already registered!"`（A14:103-106，15/16 同）。注册时对 owner 和 client 两个 Binder 都 `linkToDeath`（A14:109、:129）。注销时，未登记或不是同一个 client 就抛 `"not registered!"`（A14:146-153）。
- 由此可以确认：新建一个 `UiAutomationConnection`，注册成功，就**证明旧注册已经不存在**（没有"假成功"的可能）。之后立即断开它，或者直接把它作为下一个会话的连接。注册失败并报"already registered"，说明位置仍被占用（可能是我们，也可能是外部的 uiautomator），这时如实保留封锁，报"需本机强制结束 worker"。这条路复用候选已有的连接代码，不新增接口。
- 现实中注销失败的来源大多是系统已死（远端状态随之消失）或"not registered"（本来就不在了）。真正"远端仍登记、worker 和系统都活着、注销却失败"的情形，在源码里找不到常见触发路径【推断】。
- VD 阶段同样可以用公开 API 确认：`DisplayManager.getDisplay(id) == null` 即表示已释放。

**对没人守设备的 LLM 调用者的含义。** 按主稿原样实施 A：碰到 UiAutomation 不可确认，就是死路，而且要的是"强制结束"，不是"Stop"。采用上面的确认办法后，死路只剩"注册位确实仍被占用"，这时远程本来也无能为力（除非杀 worker）。

**C（远程终止 worker）的破坏面。**【源码】设备上同一时刻只有一个租约（`Sessions.kt:48`）。终止会结束：该租约；所有 pairing 的 shell 和文件后台任务以及内存里的输出（`ShellJobs`/`FileJobs` 在 worker 内存里）；后台屏和其中的内容（现行 VD 设置下，销屏会销掉内容）。候选里没有任何远程重启 worker 的途径：host 侧没有对应 RPC；`IExecutor.destroy()` 只接受 App UID 或 shell UID 调用（`WorkerService.kt:130-133`），只有本机管理界面会触发。封锁一旦触发，新工作本来就全被挡住，C 额外破坏的主要是正在运行的 shell 和文件任务。

**判断。** 推荐"A＋可确认的 UiAutomation 阶段"：显式的远程重试清理，对每个阶段按可观察事实确认（VD 已不存在、新注册成功）；确认不了才保留封锁，并给出"本机强制结束"出路。不需要 C。另外建议把封锁范围收窄：清理失败不应挡住 shell、文件任务、到期和正常 Stop。现在是因为 closing 资源占着 `resources` 才连带挡住的，可以把未完成的清理所有权从 `resources` 拆出来单独保存。实施者可以自行决定这一点，但应写进提交 2 的验收。

### 2.5 会话（主线第 1 条）

**主稿相对评审方上一版新增的状态**：wake 的短期 owner（借用或自建连接、收尾、不被首次 virtual 复用）；"最近一次实际结束的 kind"（只存在 worker 内）。

**wake 为什么需要 UI 连接（候选现状）。**【源码 SRC】
- `SystemActions.kt:101-117`：wake 调 `policyKey(KEYCODE_WAKEUP)`；`:163-199` 经 `ui.automation(run)` 用 `UiAutomation.injectInputEvent` 注入。等待亮屏时用 `ui.platform(run).signals` 的无障碍事件唤醒等待循环。
- 候选已有另一条不经 UiAutomation 的注入通道：`AppInput.sender`→`IInputManager.injectInputEventToTarget(event, mode, targetUid)`（`AppInput.kt:64-117`、`HiddenApi.kt:149-155`）。`SystemActions.kt:130` 的 key 命令已经在用 `INVALID_UID` 走这条通道，等价于 `adb shell input keyevent`。shell 在 14/15/16 都有 INJECT_EVENTS（清单 A14:171）。亮屏判断用的 `PowerManager.isInteractive` 是公开 API（`DeviceFacts.kt:19`）。

**更简单的做法。** wake 用已有的 InputSender 注入 KEYCODE_WAKEUP，每隔几十毫秒轮询 `isInteractive`，直到预算用完。准入只需要一个无状态检查：别的 owner 持有当前租约时报 busy。这样做：
- 不建租约、不占 UI 资源、不连 UiAutomation、没有任何需要收尾的东西；
- "wake 短期 owner""首次 virtual 不得复用 wake 遗留连接""wake 也进清理封锁"这几项全部删掉。
- 用的是候选已有的内部接口，不算新增。
- 需要设备确认：这条通道注入的 WAKEUP 在 Android 14+ 上能唤醒屏幕【推断，依据是 `input keyevent` 走同一入口；需设备】。
- "wake 要求已有会话"不好：熄屏时要建后台屏，就得"开主屏会话→wake→换后台屏"三步，评审方上一版已论证过。

**"最近结束 kind"。** 它的作用只是让"无会话"的报错带一句"上次是 virtual"。open 已强制要求 `--display KIND`，调用者必须自己选。这条记录能帮到上下文丢失的 LLM，但不是正确性所必需。建议不做；如果要做，把它并进现有的 `released` 集合，改成一个按 pairing＋name 索引的小 map，生命周期相同（revoke 时删除），不要另起结构。

### 2.6 system_server 重启后的陈旧 Binder

**实施方主张。** "有源码依据的未知"，不加自动恢复。

**核到的事实。**
- 【原件】`P/exploration-20261119/api34/commands/035-final-native-identity/stdout.bin`：system_server 已从 500 先后换成 3668、4058，zygote64 换成 4000，但 `shizuku_server` 2148（ppid 1，UID 2000）和 `com.rainng.androidctl:executor` 2268（UID 2000）**仍然存活**；Gateway App 进程不在。只有 1 个样本（API34 AVD，Watchdog 杀掉 system_server）。
- 【源码＋推断】worker 是 Shizuku 用 app_process 启动的 daemon 进程（`WorkerConnection.kt:55`），不是 zygote 子进程，所以不会随 zygote 重启而被杀，与原件一致。它在进程内缓存的系统服务代理（`ActivityTaskManager.getService()` 的单例、`PowerManager` 等 Manager 里的 Binder、UiAutomation 及其连接）在系统重启后全部是死代理。`ServiceManager.getService` 的新查询能拿到新代理，但框架单例不会刷新。
- 调用者会看到什么【推断，需设备】：
  1. Gateway App 进程已死，WebSocket 断开，在人重新打开 Gateway 之前，host 请求都到不了设备。
  2. 人重开 Gateway 后，它重新连上**还活着的旧 worker**，`status` 显示 ready（`WorkerService.kt:150` 只看是否在停止中）。
  3. 租约还在时，UI 命令复用旧的 `platform`（`UiBackend.kt:440-444`），调用会以 DeadObjectException 一类错误失败；启动 App 因为单例缓存的死代理而失败。
  4. 租约到期或调用者 close 时，`clear`→`disconnect`→新系统端回"not registered!"（IllegalStateException）。`unregister` 在 `platform != null` 时直接抛出（`UiBackend.kt:525-531`），于是置位全局 `uiCleanup` 封锁。
  5. 最后只能本机强制结束 worker。
- 所以这不是"未知"，而是一条源码上确定会走到死胡同的路径，只是没有实测。触发条件是 system_server 崩溃（Watchdog、原生崩溃），不常见但真实；本轮 API34 就碰上了一次。

**判断。** 列入缺陷清单。最小修法：worker 启动时对一个系统服务 Binder（例如已有 `HiddenApi.serviceBinder("activity")` 取到的 Binder）`linkToDeath`，回调里直接退出进程。Gateway 下次绑定时会拉起新 worker。这不是"自动恢复、重试、迁移会话"：系统重启后 VD、无障碍和会话状态本来就已经失效，worker 进程内又无法刷新所有单例，让它快速失败是最干净的。代价是丢掉仍在运行的 shell 和文件任务，系统重启时这本来就难以保证。需要设备确认 Shizuku 在系统重启后还能不能正常创建新的 UserService【需设备】。

### 2.7 建议提交 1：提前关闭 writer

**核到的事实。**
- 【原件】`P/exploration-20261119/mumu/probe-output.txt`：writer 打开时 `renameat2(RENAME_NOREPLACE) rc=-1 errno=26 message=Text file busy`；`close writer before control rc=0`；同一 inode 再次 `renameat2` 成功。`013-push-pdf/stdout.bin` 含 `publishFailed` 和 `Text file busy`。
- errno 26 是 ETXTBSY（Linux `errno-base.h`）。Download 目录的 statfs 类型 `0x786f4256` 是 vboxsf；按 Linux 6.1 的 `vboxguest_utils.c`，宿主返回的 `VERR_SHARING_VIOLATION` 会映射成 -ETXTBSY（实施方引用，我没有另行下载）。
- 为什么写入端没关就失败：宿主（Windows）上这个文件还有一个打开的写句柄，宿主拒绝重命名，于是报共享冲突【推断，与 vboxsf 机理一致；宿主侧锁持有者没有实测】。
- 修法就是把 writer 的唯一一次 close 挪到 rename 之前。【源码 SRC `TransferHandle.kt:128-157`】现在的顺序是 flush→sync→另开只读 fd 核对 dev/ino→beforePublish→native 发布；writer 要到 `close()`（`:159-183`）才关。实施方的四行研究改动和一次 PDF push/pull 字节一致（实施方 README 有记录，我只核了 probe 原件）。

**副作用。**
1. writer close 失败现在会变成"发布前失败，publicationAttempted=false"，不再是"发布后 cleanupError"。这种语义更准确。
2. close 之后到 rename 之间，临时文件不再被 fd 钉住。公开合同本来就排除了"有权主体主动替换临时文件"，所以这不是新退让。
3. 小建议：顺序改成 flush→sync→**close**→只读 fd 核对 dev/ino→beforePublish→rename。这样 dev/ino 核对是 rename 前最后一条身份事实，比"先核对后关闭"少一段窗口。实施者可以自定。
4. `DurableFile.kt:18` 的 `Os.rename` 只用于应用私有文件，不受影响【源码】。

**判断。** 同意它独立于其它选择、可以先做。样本：一个 MuMu 实例、一次 probe、一次 PDF 链路。真机共享存储（FUSE）上通常不会出这个问题【推断】，修了也没有坏处。

---

## 3. 确定缺陷与证据不足的断言

**确定缺陷（候选源码，或主稿方案本身）**
1. 【源码】清理失败后，closing 资源留在 `resources` 中，连带拒绝新的 shell 和文件任务、阻止到期，并让**正常 Stop 也失败**（`Sessions.kt:188-202,232-235,241-244`；`WorkerService.kt:115-123`）。主稿第 11 题 A 的"本机 Stop"实际要用"强制结束"。
2. 【源码】`UiBackend.disconnect()` 失败后 `automation` 不清空（`:503-521`）。任何"重试 clear"的天真实现都会命中 `UiAutomation.disconnect()` 的本地 no-op，造成假成功。
3. 【源码推断＋1 份原件】system_server 重启后，worker 存活但持有死代理；`status` 报 ready，到期或关闭会话时必然置位 `uiCleanup` 封锁。应该修（见 2.6），不应只记为"未知"。
4. 【源码】通知注销失败置位永久 `cleanupFailure`，并占住 UI 资源（`NotificationOperations.kt:155`、`UiBackend.kt:46,139`）。评审方上一版的 F6 继续成立，主稿也已去掉，这里只记为确认。
5. 【源码推断】按主稿 B 实现第 7 题，需要两个新的隐藏接口，与用户方向冲突；主稿把它列为推荐，属于方案层缺陷。
6. 【源码】候选的通知写操作没有动作后观察（`NotificationActions.kt`）。若采用"发送后核对落点"，必须新增观察窗口，否则只能拿到即时 hidden 的样本（原件已证即时 `visible=false`）。

**证据不足或需要收窄的断言**
- "B 要在 close、replace、takeover、到期、撤销、worker stop 各处挂注销"：实际上 Sessions 的所有结束路径汇总到一个接点；worker stop 不需要注销。复杂度被高估了，而真正的复杂度在未知状态那一支。
- "B 按租约数仍可能累积"：成立，但设备上同时只有一个租约，累积量等于"用过通知的租约数"。
- "C 跨 close/expiry/revoke 持续接收回调和 URI 权限，范围最宽"：从字面看成立。但远程调用者的能力没有扩大，分量被高估了。
- "PI 候选给出预期 handler"：只有 1 个同包样本；跨包、alias、多 Intent 都没有样本。
- API35 弱引用路径：未测（heap 被拒），不能写成"已修好"或"同样泄漏"。
- 文件修订中"宿主锁持有者"和"原 013 的确切 syscall"：未知，实施方已如实保留。
- `wake` 改用 IInputManager 后能否唤醒屏幕：未在设备上验证（我的建议同样需要验证）。

## 4. 值得删除或简化的机制

1. **wake 短期 owner 及其收尾、"首次 virtual 不复用 wake 连接"、wake 进入清理封锁**：改用已有的输入注入通道加轮询 `isInteractive`，全部删掉。
2. **"最近结束 kind"记录**：可以不做；要做就并进 `released`。
3. **通知监听器的"注销未知→责任转交 worker→保来源→不重复注册→status 分列"状态**：选 C 后简化为一个"可能已登记"标志；选 A 也只需要"未确认时下次复用、不重复注册"，不涉及租约。
4. **第 7 题 B 的 getIntent/query 解释层**（多 Intent、alias、候选歧义、取值失败）：用创建者包＋UID 的同一把钥匙替代。
5. **第 8 题 B/C 的跨动作任务台账**：放弃。
6. **第 11 题"UiAutomation 不可确认就永久封锁"**：改成"用一次新注册确认"，只在确认失败时封锁。
7. **主稿"listener 注销只诊断"的专门例外**：选 C 后，正常路径根本不注销，例外自然消失。

## 5. 需要用户裁决的事

只有一项真正需要用户定：

**通知监听器寿命（原第 6 题，改为二选一）**
- **C（推荐）**：worker 第一次处理通知命令时注册，一直保持到 worker 退出，由平台回收。
  - 得：Android 14 上不再随命令次数累积 JNI 全局引用，消除"轮询一两天可能让 system_server 崩溃"的路径；状态最少；不需要注销失败处理。
  - 失：worker 活着期间，系统里一直有一个 shell 通知监听器，每条通知事件都会回调进 worker（被丢弃），shell 一直持有通知附件 URI 的读授权；Android 16 上 SystemUI 重启后直接回复界面可能受影响（源码推断，未复现）。
- **A**：每条通知命令注册、用完注销，失败只诊断。
  - 得：登记窗口最短。
  - 失：Android 14 每次注册永久多一条全局引用（直到 worker 退出），上限 51,200；需要"注销未确认时下次复用、不重复注册"的小状态。
- 剩余未知：Android 15+ 是否已不累积；真机上 system_server 的基线全局引用数。

其余几题都可以按已有方向直接定，属于实施细节：
- 第 7 题：用创建者包＋UID 做发送前拒绝和发送后核对；核对放在动作后观察结束时；对不上返回 UNKNOWN/3 并附实际可见包。B 只有在用户主动提出跨包需求时，才作为"内部接口例外"另行申请。
- 第 8 题：A，文档写明后台屏不隔离共享状态。
- 第 11 题：可确认阶段显式重试，UiAutomation 阶段用新注册确认；把封锁从 shell、文件、到期、正常 Stop 上拆开。远程终止 worker 不做。用户若坚持要远程终止，可以另立一题，但现在没有必要。
- system_server 死亡时 worker 自行退出：修缺陷，不是新能力。

## 6. 建议提交 2、12、13 的修订版

- **提交 2 `refactor(session)`**
  - 范围：显式 open、同种复用、异种 replace，删除主机偏好、请求 kind 和 batch 冻结。
  - wake 改用已有输入注入通道，不建租约、不连 UiAutomation。
  - 远程"重试清理"：VD 用 `getDisplay==null` 确认，UiAutomation 用一次新注册确认。
  - 把未完成的清理所有权从 `Sessions.resources` 拆出，不再挡 shell、文件、到期和正常 Stop。
  - worker 监听系统服务死亡、自行退出。这一点也可以拆成单独的 `fix(worker)` 提交。
  - 依赖：无。
  - 验收（设备上）：熄屏时 wake 后建 VD；注入 VD 释放失败后重试成功；注入 UiAutomation 断开失败后，新注册成功即解封、注册被占时保持封锁并给出"强制结束"出路；清理失败期间 shell 和文件任务仍然可用；模拟 system_server 重启后 worker 退出、Gateway 重开得到新 worker。
- **提交 12 `refactor(notifications)`**
  - 按用户的选择（推荐 C）实现监听器，删除永久 cleanupFailure 和占用 UI 资源的例外。
  - 选择串（来源、标题、按钮、消息类内容）、自身包拒绝、不可清除预拒、控制者通知去重。
  - 不与租约绑定。
  - 验收：同一 worker 连续 N 条通知命令后，`dumpsys notification` 里只有 1 个本组件的登记（选 C），或 0 个（选 A）；注册抛错和读被拒后的路径行为正确；其余同主稿。
- **提交 13 `fix(activity)`**
  - 启动 App 和后台屏 Activity 通知共用完整的活任务来源。
  - 后台屏 Activity PI：加目标屏选项，创建者包＋UID 在别屏有任务则发送前拒绝；加动作后观察，结束时核对"创建者包＋UID 在目标屏有可见任务"，否则返回 UNKNOWN/3 并附实际可见集合。
  - 不加 getIntent/query，不加跨动作迁移检查，文档写明共享状态边界。
  - 依赖：提交 2 的会话目标。
  - 验收：Termux 同包正例、hidden-live 冲突零发送；一个跨包例子（例如打开网址的通知）报 UNKNOWN 并列出 Chrome；main 会话和非 Activity 维持原准入。

## 7. 没能核实的内容

- 完整 heap（只有摘录），以及 native 死亡接收者计数；API35 的强/弱引用数量。
- 真机 system_server 的 JNI 全局引用基线。
- Shizuku server 在 system_server 重启后能否继续创建 UserService。
- `IInputManager.injectInputEventToTarget(INVALID_UID)` 注入 WAKEUP 在产品 worker 里的实际效果（只依据源码，以及 `input keyevent` 走同一入口的常识）。
- 跨包通知 PendingIntent 的实际比例。
- vboxsf 宿主侧锁持有者（只读到实施方引用的 Linux 源码，我没有另行下载）；实施方 PDF 研究 APK 的 push/pull 原件只核到 README 层面，没有逐一打开 155–167 各条命令。
- 实施方内部 review（`T/reviews/route-r2-notifications-*.md`、`route-r2-q6-p4-*.md`、`route-r2-contract-integration.md`、`route-r3-file-probe.md`）没有逐份通读，只按需核对原件。

## 附：我写的文件

- `S/agents6/29/hprof_check.py`：独立 HPROF 摘录解析器，用于 2→6→0 复算。
- `S/agents6/29/sizes.py`：每次注册的对象大小。
- `S/agents6/29/aosp/`：android-14/15/16.0.0_r1 的 `UiAutomation.java`、`UiAutomationConnection.java`、`UiAutomationManager.java`，从 android.googlesource.com 下载。
