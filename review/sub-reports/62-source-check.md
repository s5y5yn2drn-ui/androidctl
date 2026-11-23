# 第五轮决定文件的源码核对

核对对象：`/home/user/androidctl/review/androidctl-r5-decisions.md`（下称"决定文件"）。产品源码默认指 `cab84b8`，位于 `S/r5/prod`；Kotlin 文件省略前缀 `android/app/src/main/java/com/rainng/androidctl/`，主机文件省略前缀 `androidctl/src/androidctl/`。"原型分支"指公开采集路线的对照实验分支 `import-r5/review/r4-public-823`（它从 `82318e9` 派生，比 `cab84b8` 落后 25 个提交）。"手写采集路线"指 worker 自己实现无障碍节点查询回调的现行做法；"公开采集路线"指改用 `AccessibilityNodeInfo.getChild` 等公开接口的做法。

证据类别：【源码】给出 `文件:行`；【实跑】是在 `S/r5/venv5` 里实际运行的结果；【原件】是读过的实施方原始材料；【AOSP】是已下载的 Android 14 源码；【推断】是由源码推出、没有设备验证的结论。本次没有设备，凡是涉及系统行为的结论都只到源码层面。

---

## 一、与源码不符或需要改写的说法（按重要性排）

### 1.1 第 1 题：关闭前判断"外来任务"所需的记录，被决定文件一并取消了

决定文件第 1 题把第四轮的"会话里记住上一次任务表"整句列为"取代"。但关闭前要分清后台屏上哪些任务是外来的，必须有一个参照，现行代码里没有：

- 会话打开时不查任务表。`TargetManagement.execute` 和 `WorkDisplay.preflight/create` 都没有调用任务查询（`TargetManagement.kt:14-52`，`WorkDisplay.kt:55-100`，`DeviceFacts.kt:76-81`）。【源码】
- `open` 启动前那次查询只在后台屏上做，而且只保留"这个包在别的屏上的任务"（`ActivityLaunch.kt:260-290`，`ActivityTasks.kt:79-93`）；启动后的落点确认只保留可见任务、最多 8 条（`ActivityLaunch.kt:292-316`，`ActivityTasks.kt:95-124,136-147`）。两次查询拿到的完整列表都只存在于这条命令的内存里，命令结束就丢了。【源码】
- 会话（`Lease`）里没有任何任务相关的字段（`Sessions.kt:15-21`）。【源码】

所以关闭时查到后台屏上的任务，现有数据连"这是本会话 `open` 出来的"都无法判断，更谈不上"外来"。决定文件第五节第 1 问只讨论"来源不明的一类"，没有指出连本会话自己打开的任务也认不出来。这需要用户定一个最小的跨命令记录，见 2.1。

### 1.2 第 1 题："`open` 的落点确认里发现外来任务，仍报结果未知"——现在没有这个行为

决定文件用了"仍"字，第六节也写"沿用第四轮'没有要求却发生的搬动报结果未知'"。这是第四轮的决定，不是 `cab84b8` 的现状：源码里没有任何搬屏检测，落点确认只看目标包有没有出现在目标屏上（`ActivityTasks.kt:95-124`，`ActivityLaunch.kt:292-316`），全仓搜不到比对任务所在屏变化的代码。【源码】

数据上可以做：启动前那次查询拿到的是全部运行中任务（`ActivityTasks.kt:53` 的 `getTasks(Int.MAX_VALUE, false, false, INVALID_DISPLAY)`），只是 `virtualActivityPreflight` 只留下了冲突项。改成保留整张表，再和落点确认时的整张表按 `taskId` 比较 `displayId`，就能准确认出"启动前在别的屏、启动后到了后台屏"的任务。任务被搬屏时 `taskId` 不变（`ActivityStarter.android-14.0.0_r1.java` 的 `setTargetRootTaskIfNeeded` 走 `intentTask.reparent`，不新建任务）。【源码】【AOSP】

要改写成"新增"：在 `open` 里新增这项比对，而不是沿用。

### 1.3 第四节第 1 条：标签"不完整"是按节点判断的，不是按动作判断的

决定文件写"节点的动作标签不完整（被截断）时，按标签会被拒绝"，第 13 题据此定为"标签为空、重复或被截断时改写短名"。源码里的规则更宽：

- 设备按标签解析时，只要这个节点的 `unavailable` 里有任何一条以 `actions[` 开头的记录，就拒绝这个节点上**所有**按标签的调用（`NodeActions.kt:84-89`）。【源码】
- 这类记录有两种来源：标签超过 64 KB 被截断（`NodeFacts.kt:220-232`，`Wire.kt:22`），以及读取标签时抛出异常（`NodeFacts.kt:206-218`）。不只是截断。【源码】

所以"只写标签"的边界要改写为：**这个节点上任何一个动作标签不完整时，这个节点的全部自定义动作都写短名**；另外标签为空、或与这个节点上其它动作（包括不打印出来的标准动作）的标签相同时也写短名。按这个规则打印，调用者照抄标签就能调用成功；按决定文件现在的写法（逐个动作判断）打印，会出现"标签本身完整、却因为同一节点另一个标签被截断而调用失败"的情形。

其余部分属实：短名形如 `action-7f090011`，由编号生成（`UiSemantics.kt:51`，主机 `node_facts.py:79-80`），同一节点上编号唯一，所以短名总能唯一匹配；空标签在主机和设备两端都会被拒（`ui_client.py:1007-1011`，`NodeActions.kt:66-70`）。实跑 `action --help` 显示 `--action-label` 的说明是 "Exact complete current label; must be unique."。【实跑】

### 1.4 第四节第 3 条：会销毁后台屏的非调用者途径列得不全

决定文件列了"会话空闲到期；常驻进程退出"。源码里还有三条会使租约失效、进而销毁后台屏的途径，都由设备上的用户触发：

- 用户在 AndroidCtl 界面上释放会话：`WorkerService.release` 调 `Sessions.releaseLocal`（`WorkerService.kt:99-108`，`Sessions.kt:223-228`）。
- 用户在设备上撤销配对：`MainActivity.kt:210` → `GatewayService.revoke`（`GatewayService.kt:225-252`）→ `WorkerService.revoke`（`WorkerService.kt:110-115`）→ `Sessions.revoke`（`Sessions.kt:234-237`）。worker 重新绑定时的 `reconcile` 对不再允许的配对做同样的事（`WorkerService.kt:117-127`，`WorkerConnection.kt:80`）。
- 用户在设备上停止执行器（`GatewayService.kt:217-223,288-291`），worker 进程退出（`WorkerService.kt:129-148`），属于"常驻进程退出"的一种，但不是系统进程重启。

这些都不是调用者的命令，结论"调用者命令只有三条"不受影响，但写进指引的"已知局限"应当包括它们。几条途径最终都走 `invalidateLease` → `UiBackend.clear` → `WorkDisplay.close`（`Sessions.kt:333-336`，`UiBackend.kt:575-586,606-642`，`WorkDisplay.kt:153-175`）。【源码】

另外，`session takeover` 对**调用者自己**的会话同样可用（同一配对即可，`Sessions.kt:140-147`），而接管从不复用目标（`Sessions.kt:109`），所以 `session takeover --display virtual` 对自己持有的后台屏也会销毁并重建。决定文件的说法"被接管的会话持有后台屏时"字面上包括这种情形，指引里最好点明。【源码】

### 1.5 第五节第 1 问的描述：方向对，但漏了一个现成的区分依据

"任务信息里没有'原先在哪个屏'的字段"属实：`LiveTaskFact` 只有 `taskId`、`userId`、`displayId`、`visible` 和几个组件名（`ActivityTasks.kt:13-24`）。但有一个现成的依据可以把一部分任务确定为外来：任务编号按用户单调递增分配，到上限才回绕（AOSP `ActivityTaskSupervisor.getNextTaskIdForUser`，上限每用户 10 万）。后台屏是会话打开时新建的，所以关闭时后台屏上凡是编号不大于"会话打开时已存在的最大编号"的任务，一定是从别的屏搬来的。只有会话期间新建、又不是本会话 `open` 出来的任务才真正来源不明。前提是会话打开时记下这个编号，见 2.1。【AOSP】【推断】

### 1.6 第四节第 2 条："换路线不连带改引用"成立，但换路线会改变节点缓存的工作方式

引用身份这一点属实：稳定引用的身份由来源编号、窗口编号、角色、名字、行标签和行位置链构成（`SelectionIdentity.kt:8-15`，`UiSemantics.kt:320-334`），来源编号来自对节点对象调用隐藏的 `getSourceNodeId`（`HiddenApi.kt:63`，`NodeFacts.kt:174-178`），与查询路线无关；原型分支在 `NodeFacts.kt:175`、`AccessibilityConnection.kt:109` 同样使用它。唯一性判断（`UiRefs.kt:46-49`）和重复来源的截断记录（`TreeCollector.kt:319-329`）在原型里也一致。【源码】

需要补充的是：

- 现行代码在连接建立时用两个隐藏成员关掉了系统节点缓存（`AccessibilityConnection.kt:31-39`）。原型删掉这段，缓存恢复开启，改为在每次采集开头、局部重读前、找输入焦点前调用公开的 `UiAutomation.clearCache()`。这改变的不是引用，而是"读到的节点是不是新的"：在 Android 14 里 `findFocus` 优先读缓存（AOSP `AccessibilityInteractionClient` 约 940 行），`getParent`、`getChild`、`window.root` 也走缓存。原型只在四处加了清缓存（`TreeCollector.collect`、`UiActions.requireCurrentInputFocus`、`UiActions` 的原节点重读、`VisualInput` 的原节点重读），`SystemActions.kt:173-185` 里无引用按键前读焦点窗口根节点那一处没有加。【源码】【AOSP】
- 原型基于 `82318e9`，之后的 25 个提交里有两次改引用保留规则（`cfc16b0`、`ab13a08`，现行 `UiRefs.kt:22-31,41-45` 跨最近 3 次采样保留）。换路线要在 `cab84b8` 上重新移植，不能直接合并原型分支。【源码】
- 截止处理也变了：现行代码超时抛 `snapshotBudget`（`TreeCollector.kt:360-363`），原型抛内部的截止异常、整体标为 `collectionLimit` 并返回已采到的部分（原型 `TreeCollector.kt` 的 `catch (_: LocalCollectionDeadline)`）。结果都是"树不完整"，`wait`、选择器、滚动容器核对按不完整处理的逻辑不受影响，但原因码会变。

决定文件写"换路线不连带改引用"可以保留，建议加一句"缓存从关闭改为开启加按需清空，所有不经采集器读节点的地方都要核对"。

### 1.7 第 7 题：命令名是 `scroll-until`，不是 `scroll --until`

`cab84b8` 里没有 `scroll --until` 选项；对应命令是独立的 `scroll-until CONTAINER_REF DIRECTION --max-steps N`，目标用 `--role/--name/--text/--id` 精确匹配（实跑 `scroll-until --help`；`ui_commands.py:1240-1300`；`UiScrollUntil.kt:6-24`）。决定文件、第五轮评审正文（14 处）和用户裁决原文都用了 `scroll --until`。如果不是有意改名，决定文件应改成 `scroll-until`，以免实施方以为要新增一个选项。【实跑】【源码】

### 1.8 第 6 题保存规则："文件是唯一交付"的情形列得不全

决定文件列了"不打印的模式、`--out`、截图"。源码里另外几种交付形态没有被覆盖，其中一种与"正文已打印就只警告"直接冲突，见 2.6：

- 动作带 `--screenshot` 时，正文打印出来，但图片只以自动保存的文件交付（`ui_client.py:1441-1456`）。
- `snapshot --raw`、`find --raw` 另存整棵树的 JSON 文件（`ui_commands.py:481-484,648-652`）。
- `--json` 模式不打印正文（`ui_client.py:1481-1484`），是"不打印的模式"的一种，宜写明。
- `screenshot --image-out`、`snapshot --out` 是明确指定的文件。

### 1.9 第三节"六种冒烟"与第五节的依赖不自洽（文档层面）

第三节规定"图像标注"的冒烟"要先完成"发现 A5 的修正，第五节又把 A5 列为"待确认、做不做由用户定"。如果用户决定不做 A5，这项冒烟就没有前提。这不是源码问题，顺带指出。

---

## 二、需要用户再定的冲突或缺口

### 2.1 关闭前判断外来任务，需要一份跨命令的最小记录

**背景。** 见 1.1 和 1.5。第 1 题要求关闭前发现外来任务就默认不关，但现行代码在关闭时没有任何参照来区分"本会话 `open` 出来的任务""用户明确要求搬来的任务""从主屏搬来的任务""后台屏上的 App 自己新开的任务"。决定文件取消了第四轮的"会话里记住任务表"，这一条与第 1 题的检查在源码层面互相冲突。第五节第 1 问只覆盖了其中一部分。

| 候选 | 做法 | 得 | 失 |
| --- | --- | --- | --- |
| A | 会话打开后台屏时记一个数：当时已存在的最大任务编号。关闭时，后台屏上编号不大于它的任务判为外来；其余一律当作本会话的 | 状态只有一个整数；能确定地认出"会话开始前就存在、后来被搬来"的任务，这正是"主屏开着浏览器、后台屏里点链接"的情形 | 会话期间用户在主屏新开、之后被搬来的任务认不出，会被直接销毁 |
| B | 在会话里记下本会话 `open` 落点确认到的任务编号，以及用户明确搬来的任务编号。关闭时，后台屏上不在这两份名单里的任务都拦下 | 不会漏掉外来任务 | 后台屏上 App 自己新开的任务（分享面板、新文档任务等）也会拦住关闭，调用者要多带一次"同意丢弃"；状态是一小份名单 |
| C | A 和 B 都记。编号不大于基线的判为确定外来、拦下；会话期间新出现又不在名单里的单独列为"来源不明"，按第五节第 1 问的裁决处理 | 分类最清楚，回执能说清每个任务是哪一类 | 状态和代码最多（仍只是一个整数加一份名单） |
| D | 不记任何状态，关闭时把后台屏上包名不同于本会话 `open` 过的包的任务都拦下 | 不需要编号 | 本会话 `open` 过的包也同样需要记录，等于 B 的弱化版；同包多任务分不清 |

剩余未知：任务编号回绕（每用户 10 万）在一次会话内发生的可能性极小，但没有设备验证。

### 2.2 "送回主屏"的实现方式与第四轮"不新增隐藏接口"的约束

**背景。** 第 1 题只取代了第四轮"送回主屏"一节里"不抢到主屏最前面"这一条，第四轮同一句里的"不新增隐藏接口"没有被提及，按决定文件的规则仍然有效。用现有的启动通道（`ActivityLaunch.kt:129-155` 的 `prepareActivityStart`，经已有的隐藏成员 `startActivityAsUser`，`HiddenApi.kt:172`）带上"目标屏是主屏"的选项再启动一次，系统会把可复用的任务搬回主屏并放到最前（AOSP 14 `ActivityStarter.setTargetRootTaskIfNeeded` 里的 `intentTask.reparent(...ON_TOP...)`）。但这条路有几个源码层面看得到的限制：

- 任务表只给出基础组件名，不带原始意图的数据和附加值（`ActivityTasks.kt:53` 传 `keepIntentExtra=false`，`:69`）。按组件名或启动器入口重新启动时，如果任务的根活动与这次的意图不相等，标准启动模式下系统可能在该任务上再压一个新活动，单任务模式下会给 App 投递一次新意图，都可能改变 App 的状态。【AOSP】【推断】
- 同一个 App 有多个任务时（文档模式），重新启动无法指定是哪一个。【推断】
- 基础活动没有导出的任务，shell 身份启动不了，送不回去。【推断】
- 不传显示屏选项会原地拉到前台而不搬屏；现有代码对主屏目标传的是 `null`（`ActivityLaunch.kt:107-113`），送回时必须显式传 `setLaunchDisplayId(0)`。【源码】

按任务编号精确搬动需要新的隐藏成员，例如 `IActivityTaskManager.startActivityFromRecents(taskId, options)` 或 `moveRootTaskToDisplay`；Android 14 的 shell 声明了 `START_TASKS_FROM_RECENTS`、`MANAGE_ACTIVITY_TASKS`、`INTERNAL_SYSTEM_WINDOW` 三项权限（已下载的 AOSP 14 Shell `AndroidManifest.xml:170,258,259`），但这些接口对 shell 是否真的放行没有在设备上验证。

| 候选 | 做法 | 得 | 失 |
| --- | --- | --- | --- |
| A | 只用现有启动通道按启动器入口重新启动，送不回的任务如实报告、不销毁 | 不新增隐藏成员，符合第四轮约束 | 可能改变 App 状态；多任务、未导出入口送不回 |
| B | 新增一个按任务编号移动的隐藏成员，取代第四轮"不新增隐藏接口"在这一处的约束 | 精确、无副作用（推断） | 多一个隐藏接口；需要设备验证权限 |
| C | 先 A，A 送不回时再用 B | 覆盖面最大 | 两套代码 |
| D | 本轮不做"送回"，只保留"同意丢弃"一条出路，送回留待设备验证后再定 | 代码最少 | 与第 1 题"两条出路"的裁决不一致，需用户同意 |

### 2.3 默认输出里带标签的**标准**动作怎么写

**背景。** 第 13 题只说"自定义动作只写标签"。现行输出里，凡是带标签的动作——包括标准动作，例如带"固定"标签的长按、带标签的点击——都以 `action={"id":…,"name":…,"label":…}` 打印，而且会越过对常见动作的省略规则（`ui_view.py:106-131`，第 113 行在省略判断之前）。不带标签的标准动作打印为 `+long-click` 这类短写，调用者用专门命令调用。改了自定义动作的写法之后，带标签的标准动作怎么写没有规定，不同写法对调用者的影响不同。

| 候选 | 做法 | 得 | 失 |
| --- | --- | --- | --- |
| A | 标准动作一律按原规则写命令名（`+long-click`），标签不打印 | 最短；调用者用专门命令即可 | 丢掉 App 给长按等动作的语义说明（例如"固定""删除"） |
| B | 写命令名再附标签，例如 `+long-click="固定"` | 信息不丢，调用方式不变 | 比 A 长 |
| C | 与自定义动作相同，只写标签，调用者用 `action --action-label` | 规则统一 | 本来可以用 `long-click` 的地方变成 `action`，命令更绕；标签不唯一时还要退回短名 |

### 2.4 坐标输入的自身检查：只看任务顶部，还是看整个任务

**背景。** 第 2 题定为"派发前查一次任务表，AndroidCtl 的界面在目标屏上可见就拒绝"。任务表能给出 `visible`、`displayId` 和顶部、基础、真实、原始活动组件（`ActivityTasks.kt:13-42`），足以实现。但 AndroidCtl 打开系统设置页的方式让"AndroidCtl 的界面"有两种读法：管理界面从自己的活动里不带新任务标志启动"无障碍设置"和"电池优化设置"（`MainActivity.kt:178,186`），这些设置页因此落在 AndroidCtl 自己的任务里，任务顶部是设置 App，任务底部是 AndroidCtl。现成的 `LiveTaskFact.containsPackage`（`ActivityTasks.kt:39-41`）会同时比对顶部和底部。

| 候选 | 做法 | 得 | 失 |
| --- | --- | --- | --- |
| A | 任务里任何一层是 AndroidCtl（沿用 `containsPackage`）且可见就拒绝 | 调用者不能在 AndroidCtl 带出来的设置页上按坐标点击，例如替 AndroidCtl 打开无障碍开关 | 用户从 AndroidCtl 进入设置后，同一任务内的设置页也不能用坐标操作 |
| B | 只看任务顶部是不是 AndroidCtl | 字面上最贴近"AndroidCtl 的界面" | 上述设置页可以被坐标点击；节点操作本来也允许 |
| C | 不用任务表，沿用按键已有的做法：读目标屏的无障碍窗口，检查落点处或焦点窗口的根包名（`SystemActions.kt:173-185`） | 能覆盖不属于任务的窗口 | 与裁决的"查任务表"不同；对壁纸等不可读窗口仍需另定 |

`click --touch` 的现状：接收方 UID 取自节点的包名（`VisualInput.kt:357-366`，`AppInput.kt:63-73,119-129`），节点包名不可能是 AndroidCtl，因为采集器把 AndroidCtl 的窗口标为受保护、不读内容（`TreeCollector.kt:255`）。改成不定向后，上面的任务表检查同样适用，选 A 还是 B 的差别也一样。【源码】

### 2.5 按键名不分大小写时，单字母键怎么处理

**背景。** 第 4 题接受"按键名和修饰键不分大小写"。可用的按键里有六个单字母：`a c v x y z`（`input_keys.py:7-25`）。`Enter`、`CTRL+A` 这类写法转成小写没有歧义；但单独的 `A` 有两种合理读法：当作 `a`，或者当作"大写 A"，即 `shift+a`。两种读法发给 App 的按键不同。

| 候选 | 做法 | 得 | 失 |
| --- | --- | --- | --- |
| A | 一律转小写，`A` 等于 `a` | 规则最简单 | 想输入大写字母的调用者得到小写，且无提示 |
| B | 修饰键和多字母键名不分大小写；单独的大写字母拒绝，报错给出 `a` 和 `shift+a` 两种可照抄写法 | 没有歧义 | 规则多一条 |
| C | 单独的大写字母按 `shift+字母` 处理 | 符合"大写字母"的直觉 | 与"不分大小写"的字面相反 |

### 2.6 保存规则：正文已打印、但图片只有文件可交付时

**背景。** 见 1.8。第 6 题规定"正文已经打印出来时，任何自动保存失败都是警告"，又规定"文件是唯一交付时任何保存失败都按失败处理"。动作带 `--screenshot` 时两条同时适用：正文已打印，图片却只能通过自动保存的文件交付。现行代码把"要了图片却没交付"按失败处理（`ui_client.py:1441-1469`）。`--raw` 的整树文件也是同样情况。

| 候选 | 做法 | 得 | 失 |
| --- | --- | --- | --- |
| A | 按每件交付物分别判断：哪件东西只能以文件交付，它保存失败就算失败；正文打印成功、只是正文的自动存档失败才算警告 | 调用者明确要的东西没拿到时一定知道 | 规则要写成"按交付物"，比"只看交付方式"多一句 |
| B | 只要正文打印了，图片、`--raw` 文件保存失败都只警告 | 与裁决字面一致 | 调用者要了截图却拿不到，退出码仍是 0 |

### 2.7 `page-*` 滚动方向：别名还是精确动作

**背景。** 第 4 题接受 `page-*` 方向。现行设备端把 `down` 先映射到"向下滚"，容器没有公布"向下滚"时才退到"向下翻页"（`UiScroll.kt:18-36`）。如果一个容器两种都公布了，`scroll REF page-down` 可以有两种对外行为不同的读法。另外 `scroll` 的主机解析里，不认识的第一个位置参数会被当成引用（`ui_commands.py:1215-1227`），所以 `scroll page-down` 现在会被当作一个名为 `page-down` 的引用发给设备，报"引用不在当前快照"。

| 候选 | 做法 | 得 | 失 |
| --- | --- | --- | --- |
| A | `page-down` 只是 `down` 的别名，沿用现有映射 | 改动最小 | 调用者看到容器公布 `page-down`、写了 `page-down`，实际可能执行的是小步滚动 |
| B | `page-down` 精确调用"向下翻页"；容器没有公布时拒绝并给出可照抄的方向 | 所写即所得 | 多一套方向 |

两种都要同时决定 `scroll-until` 是否也接受这些方向（`UiScrollUntil.kt:8-11` 现在不接受）。

---

## 三、实施时要注意、但不需要用户定的点

### 第 1 题（搬屏检测）

1. **检查必须放在租约失效之前。** `session close` 现在先在锁内使租约失效（`Sessions.kt:160-171`），失效会立即把清理排进界面线程（`Sessions.kt:333-336`，`UiBackend.kt:575-586`），然后才在执行线程里收尾（`TargetManagement.kt:154-189`）。任务查询函数本身只需要 shell 上下文（`ActivityTasks.kt:44-75`），但 `open` 传进去的检查回调是"租约仍是当前"（`ActivityLaunch.kt:96`，`UiBackend.kt:324-327`），失效后会抛 `sessionChanged`。所以"查表、发现外来任务就报未发出"要放在 `prepareClose` 之前，并在查询期间占住界面资源，防止并发命令。`replace` 和 `takeover` 的失效发生在 `prepareChange` 的第二段锁内（`Sessions.kt:119-129`），其间已有一个在锁外执行的 `preflight` 回调（`Sessions.kt:117-118`），查表和送回可以放在那里，之后的二次核对（`Sessions.kt:120-123`）会发现期间会话是否变过。
2. **送回之后再查一次。** 决定文件第六节"有任务没有送回成功就不销毁"需要送回后再查一次任务表确认它们已在主屏（`displayId == 0`）。
3. **`replace`、`takeover` 的销毁先于新目标创建**，新目标创建失败不会恢复旧的（`TargetManagement.kt:54-83`，`displays.md:35-38`）。所以送回或丢弃的决定必须在销毁前完成，不能等新目标建好再做。
4. **`--multiple-task` 的删除点**：`ActivityLaunch.kt:185-189,270-272,318-336,399-407`；`ActivityIntent.kt:57`；主机 `ui_commands.py` 的选项定义、`ui_client.py:82`、`activity_intents.py:82-83`；指引 `actions.md`、`displays.md`；契约样本。拒绝文字在 `ActivityLaunch.kt:274-277`。
5. **"搬过来"参数**：跳过 `ActivityLaunch.kt:264-280` 的冲突拒绝即可，但不能带 `FLAG_ACTIVITY_MULTIPLE_TASK`（`ActivityLaunch.kt:406-407`）；搬来的是不是预期的那个任务，用 1.2 的整表比对确认。
6. **空闲到期与进程退出的位置**：空闲到期由每秒一次的定时任务触发（`WorkerService.kt:40,45-57` → `Sessions.kt:290-296`）；系统进程重启时退出在 `WorkerService.kt:25`，被停止时退出在 `WorkerService.kt:145-148`，虚拟屏随进程消失由系统回收。后台屏带有"移除时销毁内容"的标志（`WorkDisplay.kt:67-76`），所以上面任务直接销毁，不会回到主屏。

### 第 2 题（坐标触摸不定向）

7. **发送通道不用换。** 现有发送器调用隐藏的 `injectInputEventToTarget(event, mode, uid)`（`AppInput.kt:75-81,131-158`，`HiddenApi.kt:134-139`）。传 `Process.INVALID_UID` 就是不定向注入，按键无引用时和唤醒已经这样用（`SystemActions.kt:98`，`UiBackend.kt:185`）。所以坐标输入只要把 `VisualInput.kt:373` 的 `target.uid` 换成 `INVALID_UID`。
8. **按键路径仍需要这个隐藏成员。** 带引用的按键和 `fill` 的回车兜底仍按节点包名定向（`SystemActions.kt:96-132`），`isTargetMismatch` 和回执里的 `targetMismatch` 字段（`AppInput.kt:160-167,195-211`）要为按键保留，不属于"相关的失败码"。
9. **删除清单**：`WindowInputListener.kt`（163 行）、`WindowInput.kt`（133 行，失败码 `inputPredictionUnavailable`）；`VisualInput.kt:331-355` 的预测分支和 `WindowInputCleanupFailure`（失败码 `windowInputCleanup`）；`AppInput.kt:83-117` 的 `windowTarget`、`coordinateTarget`、`windowInputTarget`，以及 `InputTarget` 里只为预测服务的字段（`AppInput.kt:16-26`）；两个桩文件 `android/platform-stubs/android/window/WindowInfosListener.java`、`android/platform-stubs/android/view/InputWindowHandle.java` 和 `android/app/build.gradle.kts:233-245,265` 的编译配置；测试 `WindowInputTest.kt`、`WindowInputCaptureTest.kt`、`AppInputTest.kt` 的相关部分、`WireContractTest.kt` 与 `contract/wire-contract.json` 的样本；指引 `images.md:203-252` 一带。
10. **`--expect` 的位置**：主机 `ui_commands.py:117-121`（定义）、`1373,1428,1473,1508,1544`（`timeline`、`drag`、`pinch`、`tap`、`swipe`）、`ui_client.py:93-94,1018-1023`；设备 `AppInput.kt:45-49,90-94`，`VisualInput.kt:337-341`。

### 第 4 题（参数宽容）

11. **`@` 的去除点**：主机所有引用位置参数——共用的 `Ref` 类型（`ui_commands.py:239`），以及不走它的 `type` 的位置参数（`ui_commands.py:1023`）、`scroll`（`:1215-1220`）、`scroll-until`（`:1282`）、`snapshot` 的引用（`:1344`）。现在主机不去 `@`，设备找不到引用时报 `staleRef` "Ref is not in the current snapshot"（`UiBackend.kt:277`），这就是 B6 那条误导的报错。没有发现与 `@` 冲突的其它语法：`type` 只有一个位置参数时它是正文，两个时第一个才是引用。原件里调用者确实把 `@e674098770` 原样传给了主机（`ST/efficiency/c1-candidate-01/cli-index.json` 第 2 条）。【原件】
12. **`open --package` 单独使用**：现在主机把它组装成只含 `package` 的类型化意图（`ui_commands.py:1109-1134`），设备用无动作、只限定包名的意图去解析（`ActivityIntent.kt:37-43,56-85`），解析结果多半不是唯一入口。"没有其它 intent 字段"应理解为：没有 `--action/--data/--mime-type/--category/--component/--extra/--grant-read-uri-permission`，也没有 `--url` 和位置参数包名。`open --package X --url Y` 是"把这个网址限定给 X 打开"，必须保持类型化语义。规范化放在主机即可。
13. **时长单位**：仍然拒绝，报错已给出单位说明；不涉及源码冲突。

### 第 6 题（三处修正）

14. **`find` 现在永远读缓存。** 主机把 `find` 发成 `snapshot cached=true`（`ui_commands.py:657-672`，`ui_client.py:667`），设备在 `UiBackend.kt:268` 读缓存观察。"上一个动作的观察没有稳定"这个状态现在**两端都不保留**：它只出现在动作回执的 `settle` 字段里（`UiPostAction.kt:17-24,215-235`），不随已发布的观察保存（`UiSnapshot.kt`、`UiPublication.kt` 里没有），主机也没有跨命令记录（`ui_client.py:406-440` 只在本次回执里用）。实施时要在设备的已发布观察上加一个"发布它的动作未稳定"的标记，`snapshot cached` 读到这个标记时改为重新采集。注意 `find` 没有范围选项，重新采集用的是默认范围，可能与上一个动作的范围不同。
15. **读回比较的位置**：`fill` 在 `UiVerification.kt:47-56`（比较）、`:78-87`（附实际值）、`:114-121`（不同即失败）；借用粘贴在 `BorrowedPaste.kt:33-46`（比较，不同时没有附实际值）、`:190-196`（不同即失败），等待循环在 `UiActions.kt:296-318`。占位提示文字可用节点的 `hint` 字段（`NodeFacts.kt:96`）；`showingHintText`（`NodeFacts.kt:104`）已经在比较里按空串处理（`UiVerification.kt:54`）。新规则要处理的是 App 没设这个标志、但读回文字等于 `hint` 的情形。
16. **保存规则的现有分支**：`ui_client.py:1369-1376`（只有"权限收紧失败"单独处理）、`:1398-1469`（按命令、按是否打印分支）。

### 第 7 题（`scroll-until`）

17. `wait --contains` 的匹配在 `UiCondition.kt:186-205`：对可见节点的 `text`、`description`、`stateDescription` 做归一化后的子串匹配（`UiCondition.kt:222-227`），返回命中节点本身的引用。`scroll-until` 的选择器在 `UiSelector.kt:46-72`，匹配投影后的名字，并通过 `selectionTarget` 映射到可操作的父节点（`UiSelector.kt:40`）。两者可以共用归一化函数，但子串命中的往往是不可点击的文字节点，`scroll-until` 找到后应同样映射到可操作目标，否则调用者拿到的引用点不了。
18. 树不完整时现在直接以 `observationIncomplete` 停止（`ScrollProgress.kt:42-43`），"下一次采样完整后继续"需要在 `UiScrollUntil.kt:77-89` 的检查处改为再采一次。

### 第 8 题（采集路线）

19. **"原来的两个页面"和"暖机"**：指 MuMu 上同一个已加载的 Chrome 页面"各国首都"（约 4977 个节点，原型数据 3 对 3，耗时比 0.698）和"加州"长文（接近 8192 节点上限，5 对 5，耗时比 1.1994）。暖机是每次换路线先跑一份不计分的样本，旁录结束后固定等 10 秒再取第一份。完整率在材料里按每份的 `full/partial`、`missing` 条目和节点数记，另做了"去掉来源编号和窗口编号后逐节点比较事实"的核对；加州页两边都因节点上限而 `partial`，缺口也逐条相同（`ST/collection/normal-analysis.md:178-224`）。【原件】建议实施方在报告里把"完整率相同"写成"节点事实和缺口相同"，因为加州页上两边都不是完整的。
20. **耗时比用哪个数**：材料里比的是采集阶段耗时（含关闭）。主调用耗时和整条命令耗时的比值分别约 1.22 和 1.07（同一份材料）。三个口径在已有数据上都低于 1.5，但实施方应在报告里写明用的是哪一个。【原件】
21. **"第一次查询超时就停"在原型里已基本成立。** 单次采样的时限是 3 秒（`UiBudget.kt:52,95`，原型相同），系统单次查询的等待上限是 5 秒（AOSP `AccessibilityInteractionClient.TIMEOUT_INTERACTION_MILLIS`）。原型在每次 `getChild` 和读根节点之后检查截止时间，过了就停止并返回已采到的部分，所以一次采样里最多只有一次 5 秒阻塞。公开接口无法区分"超时"和"节点不存在"（都返回空），只能按时间判断，这一点实施方要知道。多次采样的命令（动作后的观察、`wait`）里每次采样仍可能各阻塞一次，验收应写明按"一次采样"还是"一条命令"计。已有的根节点阻塞实验两条路线都是一次约 5 秒，没有叠加，但它卡的是两条路线共用的根节点读取，对路线没有区分力。【源码】【原件】
22. **删除手写路线时必须留下的**：执行动作的回调整套（`AccessibilityConnection.kt:104-140`，`HiddenApi.kt` 的 `performAction`、`callbackInterface`、`actionTransaction`、`CALLBACK_DESCRIPTOR`，`ActionReply.kt`）；`sourceNodeId`；`clientConnection`、`automationConnectionId`、`windows`（`HiddenApi.kt:63,68,75,87`）。可删的是 `CollectionNodes.kt`、`CollectionReplies.kt`，以及 `childId`、`leashedNodeId`、`setNodeConnection`、`setNodeSealed`、`clientCache`、`automationCache`、`nodeTransaction`、`prefetchTransaction`、`findNode` 九个成员和关缓存的那段反射（原型差量）。`TreeCollector` 里捕获的截止异常要随之移到 `CollectedChildren.kt`（原型就是这样做的）。【源码】

### 第 13、14、15、17 题

23. **第 13 题各项位置**：网页节点上的 `context-click` 在 `compact_actions` 里以 `action={…}` 打印（`ui_view.py:106-131`，编号 16908348 不在默认省略集合 `ui_view.py:55-69` 里）；溯源行是 `header()`（`ui_view.py:974-975`），内容由 `metadata()` 组装（`ui_view.py:628-710`），在 `ui_client.py:1288-1296` 拼到正文开头；窗口行在 `screen()`（`ui_view.py:466-508`）；动作写法在 `ui_view.py:113-114,741-747`；路径在 `ui_client.py:1321,1328,1358,1360`。
24. **溯源行改短不会破坏重读**：主机不回读已保存的正文，`snapshot --cached` 和同一采样的再读都走设备（`UiBackend.kt:268`），不依赖这一行。但这一行里带着截断摘要（`missing`、`missingSummary`，`ui_view.py:666-692`）和锁屏状态，改短时要保留"树不完整"和"锁屏"这两类信息（第四轮第 8 项要求"采集不完整时在首行写明"）。
25. **单窗口不打印窗口行**：窗口行还带 `content` 状态（受保护、根节点不可读等，`ui_view.py:486-489`）。只在这一行除编号外全是默认值时省略，否则仍要打印。窗口编号只在多窗口时才需要用于 `--window`，单窗口时省略不影响调用。
26. **相对路径**：快照帮助写明相对路径以命令的工作目录为基准；工作区根目录可以由环境变量指到别处（`--workspace-root`），文件不在工作目录之下时应保留绝对路径，不要输出一串 `../`。
27. **第 14 题去掉 `@`**：一行节点的格式是"角色、引用、带引号的名字、键值对"（`ui_view.py:728-750`）。名字一律是 JSON 字符串，其它字段是 `键=值` 或 `+动作`，所以第二个不带引号、形如 `e` 加数字的词只可能是引用，去掉 `@` 后仍能区分。另外，实际引用是 `e` 加 9 到 10 位数字（`UiSnapshot.kt:22`，`UiRefs.kt:8`），决定文件里的 `e15` 只是示意。
28. **第 15 题**：设备内部按细分码分支的地方是引用修复（`UiRefRepair.kt:44-49`，认 `staleRef`、`selectionChanged`、`identityUnknown`）；主机按码分支的有 `daemonStopped`、`deviceNotFound`、`publishUnsupported`、`temporaryCleanup`、`privatePermissions(Repair)`、`protocolMismatch`（`app.py:815,836`，`coordinator.py:298`，`file_jobs.py:417,542`，`private_files.py:227`，`ui_client.py:1372`，`transport.py:270`）。合并"已变化"类时，`sessionChanged`（会话已结束）不能与可修复的 `selectionChanged` 合到同一类，否则内部会在会话已结束时尝试修复引用。指引里点名的码只有 `busy`、`notReady`、`protected`、`staleImage`、`sourceChanged`、`arguments`、`staleRef` 几个。
29. **第 17 题能判断**：采集结果里每个节点有 `visible`，缺子节点的父节点带 `childrenIncomplete` 和 `missingChildIndex`（`CollectedChildren.kt:39-55`，`TreeCollector.kt:190-198`）。"没有可见节点缺子节点"可以用"不存在 `visible=true` 且 `childrenIncomplete=true` 的节点"来判断。另外要把三种情形算作不满足：没有 `ref` 的缺口记录（窗口级读失败，`TreeCollector.kt:279-281`）、范围内窗口状态不是 `full`、`repeatedSource` 截断（`UiRefs.kt:46`）。采集队列优先处理可见节点（`TreeCollector.kt:63,186-187`），所以在节点上限截断时这个条件多半成立。缺失的子节点本身是否可见无法知道，只能按父节点判断。

### 第 1 题以外的小项

30. 原型分支要在 `cab84b8` 上重新移植（见 1.6）；移植时 `SystemActions.kt:173-185` 读焦点窗口根节点之前也要清缓存。

---

## 四、已核对属实的清单

- `action` 接受 `--action-id/--action-name/--action-label` 三选一，主机和设备两端都校验（`ui_commands.py:761-801`，`ui_client.py:992-1011`，`NodeActions.kt:47-71`）；按名字和按标签都要求与当前公布的动作恰好一个相等（`NodeActions.kt:81-91`）；带参数的标准动作不能用 `action` 调用（`NodeActions.kt:97-99`）。【源码】【实跑】
- 主机和设备的动作名映射一致（`node_facts.py:42-72`，`UiSemantics.kt:32-51`，`UiAction.kt:10-35`），所以主机打印的短名在设备上能匹配。【源码】
- 来源编号取自隐藏的 `getSourceNodeId`（`HiddenApi.kt:63`，`NodeFacts.kt:174-178`，`CollectionNodes.kt:92`，`AccessibilityConnection.kt:114`）；原型分支同样保留和使用（原型 `HiddenApi.kt:96`，`NodeFacts.kt:175`，`AccessibilityConnection.kt:109`）。【源码】
- 调用者命令里会销毁后台屏的确实只有三条：`session close`（`Sessions.kt:160-171`）；`session open --display main --replace` 在持有后台屏时（`Sessions.kt:99-130,149-156`）；`session takeover`（`Sessions.kt:109,140-147`）。`session open --display virtual --replace` 在已持有后台屏时复用、不重建（`Sessions.kt:109-114`）；在持有主屏时只结束主屏租约，没有屏可销毁。不带 `--replace` 换类型报 `displayMismatch`，不销毁（`Sessions.kt:149-154`）。`wake` 只占资源不动租约（`Sessions.kt:208-217`，`UiBackend.kt:152`）。主机 `daemon stop`、`device forget` 不触达设备会话（实跑 `device --help` 显示 "does not revoke or stop device work"）。【源码】【实跑】
- `session takeover` 被接管方持有后台屏、接管目标也是后台屏时：旧后台屏连同其上全部任务销毁，再新建一块（`Sessions.kt:109,127`，`TargetManagement.kt:61-65`，`WorkDisplay.kt:67-76` 的"移除时销毁内容"标志）。
- 坐标输入现在全部定向到预测出的 UID 或 `--expect` 指定的包（`VisualInput.kt:315-355,368-384`）。
- 决定文件第二节"第 25 项：f 项和缺陷 S4 里与手写节点查询有关的修正已经做了"属实：提交 `56a43d7`（回调尾部多余字节容忍、未知事务码报协议错误，`CollectionNodes.kt:117-137`，`CollectionReplies.kt:21,74`）；动作回调那一处在 `566a950`（`ActionReply.kt`）。【源码】
- `--multiple-task` 仍在（实跑 `open --help`），默认拒绝的报错仍推荐它（`ActivityLaunch.kt:274-277`）。【实跑】【源码】
- 已有的公开路线数据：两个页面耗时比 0.698 和 1.1994，节点事实和缺口相同（`ST/collection/normal-analysis.md:178-224`），与决定文件第六节"1.20 倍和 0.70 倍、两个页面完整率相同"一致。【原件】
- 手写路线删除后执行动作的回调不受影响（`ActionReply.kt`、`AccessibilityConnection.action` 不依赖 `CollectionReplies`，全仓搜索确认）。【源码】

---

## 五、没能核实的

- 带"目标屏是主屏"的选项重新启动时，系统是否对每种启动模式都把后台屏上的任务搬回主屏，以及会不会给 App 投递新意图或压上新活动。只读了 AOSP 14 的 `ActivityStarter`，需设备验证。
- `startActivityFromRecents`、`moveRootTaskToDisplay` 对 shell 身份是否放行。只确认了 shell 声明了相关权限。
- 不定向注入在桌面（有壁纸窗口）上能否点开图标、拖动图标。源码上不定向注入不做同 UID 检查，但没有设备。
- 原因码总数。评审报告写 185 个；我用"`Rejected(`/`Fault(` 第一个参数是字面字符串"去重只数到 138 个（结果在 `S/agents8/62/codes.txt`），差额应来自非字面构造的码，没有逐一核对口径。
- 任务编号在一次会话内回绕的实际概率；Android 15、16 上任务编号分配是否不变（只看了 AOSP 15 的 `ActivityTaskSupervisor`）。
- 公开路线开启系统缓存后，晚到的预取回调会不会在下一次采样清缓存之后写入旧节点。AOSP 14 只缓存"当前正在等待预取"的那次交互的结果（`AccessibilityInteractionClient.setPrefetchAccessibilityNodeInfoResult`），风险看起来小，但没有设备验证。
