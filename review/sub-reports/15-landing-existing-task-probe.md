# 后台屏三件事：落点判定、目标 App 已有任务、能力探针

范围：候选 3f45af6 的 `open` 落点、跨屏已有 task 的处理、后台屏能力探测。前提是用户已定的显式会话模型（只有 `session open --display main|virtual` 建租约，没有会话就拒绝）。
证据标注：【实测】=包内原件；【源码】=候选源码（REPO 相对路径，3f45af6 行号）；【上游】=本次下载核对的 AOSP（frameworks/base，tag 与行号已写明，缓存在 `scratchpad/agents3/work15/aosp/`）；【建议】=设计。

---

## 0. 结论摘要

1. **落点只回答一个问题：动作结束时，我这块屏的前台是不是我要的那个包；有没有东西跑到别的屏上去。** 判定改在动作后 settle 结束时进行，只读一次 `getTasks`，按**包**而不是按 activity 比较。结果只有 `target | other | elsewhere` 三个值。非零退出只给两种情况：`elsewhere`，以及别屏原有的 task 被搬进本屏（`movedTask`）。主屏和后台屏用同一条规则。按这条规则回放包内 4 次真实 VLC open，都应得到 `target`、exit 0（现状是全部 `unknown`、exit 1）。
2. 平台返回码证明不了落点：上游把被拦截（aborted）的启动对外报成 `START_SUCCESS`（ActivityStarter 14:1344-1347）。startActivity 返回那一刻的 task 顶部也证明不了：跳板、启动页、权限对话框都会在之后替换它。现状这两个依据都用上了，所以两头都错。
3. 删掉 `getAllRootTaskInfos` 这一路，以及 root/leaf 交叉一致性、before/after root 迁移、alias/UID 关联、`observedRoots`、`landingError.before/after`、`preflight` 字段。`ActivityTasks.kt` 从 303 行降到约 70 行，`ActivityLaunch.kt` 从 400 行降到约 290 行，主代码净减约 340 行。
4. **已有任务**：上游跨屏复用 task 只看"理想匹配"（task.realActivity 等于目标组件），以及 singleTask/singleInstance。命中后会把那个 task **整个搬到**后台屏（ActivityStarter 14:2993-2997 `reparentToTargetRootTask`）。销屏时，因为设了 DESTROY_CONTENT_ON_REMOVAL，它会被销毁（TaskDisplayArea 14:1818-1843）。E4 在 AOSP16 上实测过这条路径。
5. 推荐的默认行为是一张三分支表：没有冲突就照常启动；冲突且启动模式是 standard/singleTop/singleInstancePerTask 时，加 `FLAG_ACTIVITY_MULTIPLE_TASK` 开新实例，主屏 task 不动；冲突且启动模式是 singleTask/singleInstance/documentLaunchMode=never（平台必然搬走主屏 task）时拒绝。首版不提供"借用"选项。"借用并归还"在接口层做不干净：归还只能放到主屏**顶部**，worker 一死就没人归还。
6. 包内原件**无法**估计"常用 App 已在主屏有 task"的比例：全是新装模拟器，存活的 task 只有桌面、systemui 和测试 App。这个比例需要上设备量，量法见 §2.6。
7. **能力探针**：新增顶层命令 `androidctl doctor`，不需要会话，默认做静态检查加一次约 1–2 s 的临时屏实测（建屏 → 启动自带探针 Activity → 核落点、帧、无障碍、注入、主屏焦点 → 销屏）。`--static` 只做无副作用检查。它和 `session open --display virtual` 的失败用同一套原因码：顶层只有 `displayUnsupported`（本机做不到）和 `displayUnavailable`（现在做不到）两个，下面挂 `cause`。
8. 两条会改变判断的上游事实，探针和文档都要反映：①没有 `android.software.activities_on_secondary_displays`，或多窗口被关掉（低内存、config），平台会**静默**把启动改到主屏（TaskLaunchParamsModifier 14:485-489、ATMS 14:904-955）；②ADD_TRUSTED_DISPLAY 的保护级别是 `signature|role`，`pm grant` 授不了，MuMu 这类 ROM 补不上。

---

## 一、落点判定的重新设计

### 1.1 agent 真正要回答的问题

【建议】agent 在 `open` 之后要决定的只有两件事：
- **能不能继续在这块屏上操作这个 App**：看本屏前台是不是目标包；
- **有没有打扰到别的屏（真人的主屏）**：看目标包有没有在别的屏新出现，以及别的屏原有的 task 有没有被搬过来。

它和现有两个依据的关系：

| 依据 | 能说明什么 | 不能说明什么 | 依据出处 |
|---|---|---|---|
| startActivity 返回码 | ATMS 接受或拒绝了请求；2/3 表示复用了已有 task | 是否真的启动：被拦截的启动对外返回 `START_SUCCESS`；不能说明落在哪块屏 | 【上游】android-14.0.0_r1 `ActivityStarter.java:1169-1177`（abort 仍"假装成功"）、`:1344-1347` `getExternalResult` 把 `START_ABORTED` 改成 `START_SUCCESS`；`ActivityManager.java:480-524` 码表 |
| startActivity 刚返回时的 task 顶部 activity | 那一瞬间的状态 | 跳板/启动页之后会替换顶部；返回时新 task 可能还没建好 | 【实测】VLC 解析出 `StartActivity`，真正落地的是 `VideoPlayerActivity`（`stages/37-work-display/runtime-api34/result.md` "typed open 与落点事实"行） |
| **settle 结束时目标屏前台 task 的包** | 正好回答"我要的 App 现在在不在我这块屏前台" | — | 本设计 |

### 1.2 判定的事实、时点和取值

**事实来源**：只用 `IActivityTaskManager.getTasks(Int.MAX_VALUE, false, false, INVALID_DISPLAY)`，候选里已有，即 `HiddenApi.liveTasks`（`HiddenApi.kt:159-167`）。每条记录有 `taskId / displayId / isVisible / topActivity / baseActivity / realActivity / origActivity / baseIntent`。
- 【上游】`RunningTasks.java`（14）`:76-111`：返回顺序是"各屏的焦点且可见 task"在前，然后是可见 task（逐屏、从上到下），最后是不可见 task。所以 **"D 屏前台" = 结果里第一个 `displayId==D && isVisible` 的条目**。
- 【上游】`Task.java`（14）`:3372,3384,3444-3445`：`displayId`、`topActivity`、`isFocused`、`isVisible` 都由 `fillTaskInfo` 填写。
- 不用无障碍窗口目录做主判据。【实测】4 次后台屏 VLC open 中有 3 次，动作后 1.5 s 内目标屏窗口目录是空的或预算用尽（`vlc-typed-open-once/001-windows.decoded.bin`：`"windows":[]`，但 recentEvents 里有 display2 上 VLC 的 window 32/33；`readiness-vlc-virtual-view-once` 同样如此；`api37-readiness/runtime-api34/.../vlc-virtual-view-once` 是 `snapshotBudget`）。task 表不受影响。

**时点**：并入已有的动作后观察。`SystemActions.perform` 在 `ui.postAction.settle(run, null)` 返回之后（`SystemActions.kt:57-61`）再读一次 task 表，算出 landing 并写进结果。settle 本来就是"等画面稳定"的窗口，跳板在这段时间里会完成。启动前那次 task 读取（§2 的冲突判断也要用）同时作为 `before`。整个 open 只读两次 task 表；去掉现在 `launch()` 里的即时读、`getAllRootTaskInfos` 和 `liveTasks` 的重复读。

**跳板/启动页**：按包比较，`pkg(t) = package(topActivity) ?: package(baseActivity)`。同包的跳板、启动页、真正的页面都算 `target`。跳到别的包（链接路由、权限控制器、选择器），只要还在本屏，就是 `other`，并报告前台包名。agent 看到 `foreground=com.google.android.permissioncontroller` 就知道下一步该处理权限框。

**判定表**（输入：B = 启动前的 task 表，A = settle 后的 task 表，D = 租约目标屏，H = 解析出的 handler 包；只看 user 0）【建议】：

| 顺序 | 条件 | result | landing | exit | 附加字段 |
|---|---|---|---|---|---|
| 1 | 启动抛 SecurityException/ActivityNotFound，或返回码 <0 或 ≥100 | failed | 不写 | 1 | `launchCode` |
| 2 | 启动调用其它异常 / 回包丢失 | unknown | 不写 | 1 | `launchError` |
| 3 | A 中存在 t：`t.displayId==D`，且 B 中同 taskId 的条目 `displayId≠D`（别屏原有 task 被搬进本屏） | done | 按 4/5/6 取值 | **1** | `movedTask:{taskId, fromDisplay}` |
| 4 | fg(A,D) 的包 ≠ H，且 A 中存在 t：`t.displayId≠D ∧ t.isVisible ∧ pkg(t)==H`，并且 B 里没有"同 taskId、同屏、可见"的记录（启动后才在别屏出现或变可见） | done | `elsewhere` | **1** | `elsewhere:{displayId, taskId}` |
| 5 | fg(A,D) 存在且 pkg==H | done | `target` | 0 | `foreground:{package, taskId}` |
| 6 | 其它情况（本屏前台是别的包，或本屏没有可见 task） | done | `other` | 0 | `foreground:{package\|null, taskId\|null}` |
| — | settle 后读 task 表抛错 | done | 不写 | 0 | `landingError:"<一句话>"` |

- 取值只有三个。`unknown` 不再作为取值：读不到就不写 landing，exit 跟启动码走。启动前那次读取刚成功过，之后的读取几乎不会失败；为这一种情况再加一个状态和一套退出规则不值。
- **主屏与后台屏统一用这张表。** 主屏会话下 D=0，别的屏通常没有 task，规则 3/4 基本不会触发，效果就是按包确认前台。现状主屏用 before/after root 迁移（`ActivityTasks.kt:219-290`）、后台屏用顶部 activity 精确匹配（`:117-150`），两套语义，这里合并成一套。
- 去掉现状"后台屏启动成功就先置 exit 1、确认后再去掉"的做法（`ActivityLaunch.kt:254-258`、`:320`）。回包在 settle 中途过期时，结果里就没有 landing；agent 本来也得再看一次画面。

### 1.3 用包内原件回放

| 原件 | 现状输出 | 原件中可供判定的事实 | 新规则结果 |
|---|---|---|---|
| `stages/37-work-display/runtime-api34/commands/vlc-typed-open-once/stdout.bin`（旧物料 6f5ee3a） | `START_SUCCESS`、`landing unknown matchingTaskCount 0`、exit 1 | settle 1503 ms；事件 seq6/8 显示 display2 上的 window 32、33 属于 `org.videolan.vlc`（`001-windows.decoded.bin`）；事后独立读到 task134/display2/VideoPlayerActivity/UID10142（`result.md`） | fg(A,2)=VLC task134 → **target，exit 0** |
| `.../runtime-api34/commands/immediate-vlc-reopen/stdout.bin`（同物料） | 同上，exit 1 | 动作后窗口目录本身就有 display3 的 VLC 焦点窗 46，tree 可用；独立读到 task135/display3 | **target，exit 0** |
| `.../api37-readiness/runtime-api37/commands/readiness-vlc-virtual-view-once/stdout.bin`（当前候选物料） | 同上，exit 1，窗口目录为空 | 事件 seq3/4：display4 上的 VLC window 136/137；独立读到 task85/VideoPlayerActivity/display4，主屏无 VLC | **target，exit 0** |
| `.../api37-readiness/runtime-api34/commands/vlc-virtual-view-once/stdout.bin`（当前候选物料，第 4 次） | 同上，exit 1，`snapshotBudget` | 独立读到 task166/VideoPlayerActivity/display2 | **target，exit 0** |
| `stages/final-verification/runtime-release-api37/commands/vlc-virtual-view-once/stdout.bin`（create 失败后误落主屏） | `landing observed displayId 0` | 显式会话模型下没有会话，请求在派发前就被拒绝，不会落到主屏；若 agent 显式开的是主屏会话，D=0 → target | 由会话模型解决，与 landing 无关 |

注意：前 4 行的事实来自事后几秒的独立读取，加上 settle 窗口内已经收到的 display 事件。settle 结束那一刻 `getTasks` 返回什么属于【推断】，需上设备确认（§4 第 1 条）。

### 1.4 删除与代码量

【源码】可删：
- `ActivityTasks.kt`：`ChildTaskFact`、`RootTaskFact`、`activityTasks()`、`leafRoots`、`virtualActivityPreflight`（`:85-115`，由 §2 的冲突表代替）、`virtualActivityLanding`（`:117-150`）、`activityLanding`、`matchesResolvedActivity`、`rootTransition`（`:219-290`）。`LiveTaskFact` 收缩成 `TaskFact(taskId, displayId, visible, top, base, real, orig, baseIntent)`，不再输出 `topActivityInfo` 和 UID 关联。
- `ActivityLaunch.kt`：`ActivityPlatform.tasks()/liveTasks()` 合并成一个 `tasks()`；删 `checkVirtualTasks`（`:273-310`）、`recordVirtualLanding`（`:312-324`）、`recordLanding`（`:338-351`）、预置 exit 1 与 `pending` landing（`:254-258`）。新增 `launchPlan()`（约 25 行，§2.5）和 `land(before, after)`（约 35 行）。
- `HiddenApi.rootTasks`（`HiddenApi.kt:158`）。它唯一的另一个使用者是 `NotificationActions.kt:132,462` 的通知 open，按用户决定通知首版只保留只读，所以也删。
- `boundedActivityFacts` 只剩 resolver 候选在用（`ActivityLaunch.kt:205`），保留。
- 输出字段删掉：`preflight`、`landing.state/source/matchingTaskCount/matchingRootCount/association/root/childTask/observedRoots/observedRootsOmitted/reason`、`landingError.before/after`。
- 错误码删掉 `taskPreflightUnknown`。

规模估计【建议】：主代码 Kotlin 约 −340 行（`ActivityTasks.kt` 303→≈70，`ActivityLaunch.kt` 400→≈290）。`ActivityLaunchTest.kt` 里 landing/preflight 相关约 170 行（`:118-187`、`:378-469`）换成约 120 行判定表测试。契约里约 8 个 landing 样本缩成 3 个（target/other/elsewhere+movedTask）。文档方面，`actions.md:340-349` 的 10 行缩成 4 行，`displays.md:67-75` 缩成 5 行。

---

## 二、目标 App 在主屏（或别的屏）已有任务

### 2.1 上游实际怎么复用 task（推演的基础）

【上游】android-14.0.0_r1，15/16 行号括注：
- `ActivityStarter.getReusableTask`（14:2846-2900）：只要有 `NEW_TASK` 且没有 `MULTIPLE_TASK`，或者启动模式是 singleTask/singleInstance，就会尝试复用（14:2861-2863；15:2752；16:2972-2974）。singleInstance 用 `findActivity` 在**所有屏**找（14:2873），其余用 `RootWindowContainer.findTask`。
- `RootWindowContainer.findTask`（14:2211-2254；15:2386 起；16:2358 起）：先在首选屏（launchDisplayId 指定的后台屏）里找理想匹配或亲和匹配；**其他屏只接受理想匹配**（14:2235-2245）。理想匹配 = `task.realActivity == cls` 或 `affinityIntent.component == cls`（14:390-404；16:408-415）。只有亲和性相同的候选只在首选屏内有效（14:405-416）。
- 命中别屏的 task 后，`setTargetRootTaskIfNeeded`（14:2912-3000）认定 `differentTopTask = true`（14:2955），然后执行 `intentTask.reparent(mTargetRootTask, ON_TOP, …, "reparentToTargetRootTask")`（14:2993-2997；15:2884；16:3110），**把整个 task 搬到后台屏**。
- documentLaunchMode="never" 会去掉 `MULTIPLE_TASK`（14:3182-3188；16:3336-3340）。
- 销屏：设了 DESTROY_CONTENT_ON_REMOVAL 时 `task.remove(...)`（TaskDisplayArea 14:1818-1843；15:1799-1824；16:1795-1819）；没设时把 task reparent 到默认屏 **POSITION_TOP**（14:1852；16:1828）。
- worker 进程死亡时，DMS 因 app token 死亡释放虚拟屏（VirtualDisplayAdapter 14:302-316 `binderDied → destroyLocked`），走同一条销屏路径。
- 【实测】E4（AOSP16 AVD，原件 `e4-own-task-*` 不在包内，只有报告转述）：主屏 task15 经 `NEW_TASK + launchDisplay` 启动后返回码 3，task15 被移到 VD8；close 之后 task15 不复存在（`research/display/report.md` "E4" 节）。

候选现状是在启动前按**包**拦截所有别屏 task（`ActivityTasks.kt:85-115`；`displays.md:67-69`）。这比平台实际的复用范围更宽：同包但 realActivity 不同的 task，平台并不会跨屏复用。

### 2.2 四种做法逐一推演

**做法 1：拒绝（现状）**
- 接口：不发启动请求。
- 对主屏：没有影响。
- 会话结束：不涉及。
- 问题：目标包只要在主屏有 task（包括后台的）就打不开。日常手机上，用户按 Home 或进程被回收后 task 仍挂在主屏，只有从最近任务里划掉才会消失【上游推断，需设备】。所以常用 App 多半会被挡住，比例需要实测（§2.6）。

**做法 2：强制新开实例（`NEW_TASK | MULTIPLE_TASK`）**

| 启动 activity 的 launchMode | 平台结果 | 依据 |
|---|---|---|
| standard / singleTop | `putIntoExistingTask=false`，在后台屏**新建 task 和新实例**，主屏 task 不动 | 14:2861-2863 |
| singleInstancePerTask（API31+） | MULTIPLE_TASK 生效，新建 | 14:2603-2607、3176-3183 |
| singleTask | MULTIPLE_TASK 对复用判断**无效**（启动模式本身就让 putIntoExistingTask=true）。别屏 task 的 realActivity 等于它时，会被**搬到**后台屏；不等时，在后台屏新建 task，于是出现第二个实例 | 14:2861-2863、2235-2245、2993-2997 |
| singleInstance | `findActivity` 查遍所有屏，已有实例所在 task 被**搬到**后台屏 | 14:2870-2873 |
| documentLaunchMode=never | MULTIPLE_TASK 被去掉，退回默认复用，理想匹配时**搬走** | 14:3182-3188 |
| documentLaunchMode=always / intoExisting | 新建文档 task | 14:2626-2631、3164-3172 |

- 对主屏的可见影响：新实例的情况下没有。搬走的情况下，如果主屏那个 task 正在前台，真人会看到 App 消失。
- 会话结束：新实例随屏销毁，主屏 task 完好。
- 风险：两个 task 共用一个进程，单例、内存缓存、"当前页面"一类静态状态，以及媒体会话、相机这类独占资源都是共享的。有的 App 会用 `isTaskRoot()` 或静态标记主动 finish 重复实例。App 内部的跳板如果对主屏 task 的 realActivity 用 NEW_TASK 发起启动，仍会把主屏 task 搬过来（这种模式少见）。这些需设备逐个 App 验证（§4 第 5 条）。

**做法 3：借用并归还（搬过来，结束时搬回）**
- 接口：`IActivityTaskManager.moveRootTaskToDisplay(taskId, displayId)`，需要 INTERNAL_SYSTEM_WINDOW【上游】ATMS 14:3049-3061 / 16:3163，Shell manifest 14:170 有这个权限。也可以直接让平台复用时搬过来。
- 归还：只能调用同一个接口，而它会 `reparent(onTop)` 再 `resumeNextFocusAfterReparent()`（RootWindowContainer 14:1947-1950；16:1997-2000），所以 **task 回到主屏时放在最上面并拿到焦点**，压在真人正在用的 App 之上。要放到底部只能用 WindowContainerTransaction 加 TaskDisplayArea token，这要求注册 DisplayArea organizer，和 SystemUI 已经注册的那个冲突【上游推断，未逐行核】。
- 状态：进程不死时数据保留。跨屏会产生配置变化，Activity 可能重建，靠 savedInstanceState 恢复，需设备确认。
- worker 死亡：VD 被系统释放。有 DESTROY 标志时借来的 task 被销毁，没有归还的机会。去掉 DESTROY 标志的话，**所有** VD task（包括 agent 自己开的）都会被放到主屏顶部（TaskDisplayArea 14:1838-1852）。
- 结论：两条路都会在真人主屏上产生可见的突兀变化，而且 worker 死亡时兜不住。不采用。

**做法 4：搬过来不归还 / 销屏时交给系统**
- 这就是去掉预检后平台的默认行为（E4 实测）。主屏 task 立刻从主屏消失，如果它在前台，真人马上能看到。
- 带 DESTROY 标志时，会话结束会销毁它，丢掉未保存的界面状态和返回栈；不带时它被放到主屏顶部。
- 不能作为默认。可以作为显式选项（`--take-task`），但首版不做，理由见 2.4。

### 2.3 推荐的默认行为：三分支表

【建议】启动前用同一次 `getTasks`（B）判断。P = 解析出的规范组件（考虑 alias 的 targetActivity），LM/DM 取自 `ResolveInfo.activityInfo.launchMode/documentLaunchMode`：

```
ideal(t) := P ∈ {t.realActivity, t.origActivity, t.baseIntent.component}
           (LM==singleInstance 时再加 t.topActivity==P)
E := { t ∈ B : user0 ∧ t.displayId ≠ D ∧ ideal(t) }
T := { t ∈ B : user0 ∧ t.displayId == D ∧ ideal(t) }

E 为空                                   → 照常启动（NEW_TASK）
E 非空 且 T 非空                          → 照常启动（首选屏的理想匹配优先，14:2224-2231）
E 非空 且 LM∈{standard,singleTop,singleInstancePerTask} 且 DM≠never
                                          → NEW_TASK|MULTIPLE_TASK（新实例）
其它（singleTask / singleInstance / DM=never）→ 不发送，错误码 taskOnOtherDisplay
```

- 只在后台屏会话下有意义。主屏会话时别的屏没有 task，第一行就成立。
- 同包但不是理想匹配的 task 不再拦截，因为平台不会跨屏复用它（14:2235-2245）。
- 启动后仍由 §1.2 规则 3 兜底：App 内部跳板把别屏 task 搬过来时，报 `movedTask` 并 exit 1，提醒"会话结束会销毁它"。
- 新实例的结果里加 `newInstance:true`，以及 `existingTask:{taskId, displayId}`，让 agent 知道主屏还有一个实例共享同一进程。

**拒绝时的错误信息**（文本模式）：
```
open: not sent (taskOnOtherDisplay)
org.example/.MainActivity (launchMode singleTask) already runs in task 57 on display 0 (main, background).
Starting it here would move that task off the main screen; closing this session would destroy it.
next: work on the main screen (session open --display main), or ask the user to remove the app from Recents and retry.
```
JSON 的 `error.data` 带 `conflict:{taskId, displayId, visible, activity, launchMode}`。visible=true 时把 "background" 换成 "in front"。

### 2.4 为什么不做"借用"选项

- 新实例和主屏实例共享进程与应用数据，登录态、账号、文件都在。agent 拿到的已经是"已登录的 App"。借用额外得到的只有真人那条返回栈和当前页面。
- 借用的代价：真人的 task 当场从主屏消失，会话结束后被销毁；worker 一死就无法挽回（§2.2 做法 3/4）。
- 如果设备实测（§4 第 3、5 条）显示被拒绝的大多是 singleTask 启动页的常用 App，再加 `open --take-task`。语义是"移入本屏、会话结束随屏销毁"，结果字段复用 `movedTask`。到那时只需要在第三分支加一个开关，约 10 行。

### 2.5 实现要点

- `ActivityInfoFact` 增加 `launchMode`、`documentLaunchMode`（`ActivityLaunch.kt:125-126` 取自 `info.activityInfo`）。
- `launchPlan(before, handler, D)` 返回 `Plain | NewInstance(existing) | Refuse(conflict)`。`NewInstance` 时在 `prepareActivityStart` 之前执行 `prepared.intent.addFlags(FLAG_ACTIVITY_MULTIPLE_TASK)`。
- 只读一次 `getTasks`（需要 REAL_GET_TASKS，保留现有检查 `ActivityLaunch.kt:101-103`，原因码改为 `missingPermission`）。

### 2.6 包内原件能否估计比例：不能

- 【实测】API34 AVD 初态的存活 task 只有 systemui#131、launcher#130 和分屏根 #3/#4/#5（`stages/37-work-display/runtime-api34/commands/initial-activities/stdout.bin`）。recents 里 13 条大多是测试 App（`final-recents/stdout.bin`）。MuMu 只有桌面（`runtime-mumu/commands/initial-activities/stdout.bin`）。这些都是新装测试机，不代表日常使用的手机。
- **需设备量**，量法：
  1. 选 ≥5 台日常在用的真机，再选 agent 典型目标 App 30 个（或用 `dumpsys usagestats` 取使用最多的前 30 个）。
  2. 三天内随机抽 5 个时点，只读执行 `adb shell am stack list`（失败则用 `dumpsys activity activities`），统计每个目标包在 display 0 上是否有存活 task。
  3. 对每个包执行 `adb shell cmd package resolve-activity -a android.intent.action.MAIN -c android.intent.category.LAUNCHER <pkg>`，读启动 activity 的 launchMode/documentLaunchMode（需设备确认输出里有这两个字段）。
  4. 算两个数：a = 有主屏 task 的比例（现状的拦截率）；b = 其中 launchMode 为 singleTask/singleInstance 或 DM=never 的比例（推荐方案下的拦截率 ≈ a×b）。
  5. 用法：a×b 高（比如 >20%），就把 `--take-task` 提进首版；a×b 低，维持 §2.3。

---

## 三、能力探针命令

### 3.1 命令形态

【建议】
- 名字：顶层 `androidctl doctor`。新模型下 `display` 命令组（select/create）会被删掉，没有合适的组可放；顶层和 `status` 并列，发现成本最低。首版只含"后台屏"一节，以后其它能力检查也能挂进来。
- 选项：`--static` 只做无副作用检查；默认做静态检查加临时屏实测（3.3）；全局 `--json` 照常可用。
- **不需要会话**：设备端 RPC `display.check` 和 `status` 一样不要求租约（`RpcMethods.kt:5-15` 的 QUERY 类，以及不受 `requiresDisplay` 约束）。实测部分进入 UI 执行器排队，不读也不改当前租约。UI 执行器正被别的请求占用时返回 `busy`，不等待。
- 设备端 RPC：`display.check {live: bool}` → `{verdict: "ok"|"unsupported"|"unavailable", checks:[{id, state:"ok"|"fail"|"skip", detail, cause?, fix?}], now:{...}}`。

### 3.2 原因码，与 `session open --display virtual` 共用

顶层错误码只有两个，agent 按它们分支：
- `displayUnsupported`：本机做不到，改用主屏。
- `displayUnavailable`：现在做不到，可以等或让真人处理后重试。

`cause` 给人看，也给 agent 看：

| cause | 顶层码 | 一句话原因 | 补救 |
|---|---|---|---|
| `apiTooOld` | Unsupported | Android < 14 没有 own-focus/no-steal 虚拟屏 | 补不了 |
| `missingPermission:<NAME>` | Unsupported | shell 没有 NAME | ADD_TRUSTED_DISPLAY / INJECT_EVENTS / INTERNAL_SYSTEM_WINDOW 是 signature 级，`pm grant` 授不了，补不了（root 设备以 root 跑 worker 不在本产品范围） |
| `noSecondaryDisplayActivities` | Unsupported | 系统会把后台屏启动静默改到主屏 | 没有 feature 就补不了。低内存或 config 关闭时可开开发者选项"强制允许活动可调整大小"再重启（需设备确认） |
| `flagsDropped` | Unsupported | 建屏后 trusted/own-focus/no-steal 没保留 | 补不了 |
| `launchRerouted` | Unsupported | 探针 Activity 落到了别的屏 | 补不了 |
| `windowsNotVisible` | Unsupported | 无障碍读不到该屏窗口 | 补不了 |
| `inputRejected` | Unsupported | 按屏注入被拒 | 补不了 |
| `noFrame` | Unsupported | 截不到该屏的非空帧 | 补不了 |
| `stealsFocus` | Unsupported | 建屏或启动使主屏失焦 | 补不了 |
| `mainNotReady` | Unavailable | 主屏熄屏或锁定 | `wake`，再请真人解锁 |
| `displayNotOn` | Unavailable | 新屏没到 ON | 重试；仍然失败就按 Unsupported 处理 |

`session open --display virtual` 在建屏前先跑静态检查，失败时直接返回同样的码和 cause，**不再把 SecurityException 原文包成 `workerFailure`**（现状见【实测】`runtime-mumu/commands/unsupported-create-once/stdout.bin`：`"code":"workerFailure"`）。建屏过程中的 flags 和 ON 判定也映射到 `flagsDropped`/`displayNotOn`。错误信息末尾附一句 `run androidctl doctor for all checks`。两边共用一个 `WorkDisplaySupport` 对象，候选的 `WorkDisplay.create` 和 `mainReadyForWorkDisplay`（`WorkDisplay.kt:33-78`、`DeviceFacts.kt:76-81`）挪进去。

### 3.3 检查项清单

**静态（无副作用，`--static` 也会做）**

| id | 怎么查 | 依据 |
|---|---|---|
| api | `Build.VERSION.SDK_INT >= 34` | `DisplayManager.VIRTUAL_DISPLAY_FLAG_OWN_FOCUS/STEAL_TOP_FOCUS_DISABLED` 从 API34 起存在（候选 `WorkDisplay.kt:37-46` 反射取字段） |
| perm.trustedDisplay | `shellContext.checkSelfPermission("android.permission.ADD_TRUSTED_DISPLAY")` | 【上游】DMS 14:1481-1487 / 16:1928-1934；core manifest 14:7286 为 `signature\|role`；Shell manifest 14:715、15:747、16:975 |
| perm.launchOnDisplay | INTERNAL_SYSTEM_WINDOW | `isCallerAllowedToLaunchOnDisplay`（ActivityTaskSupervisor 14，"allow launch any on display"分支）；Shell manifest 14:170 |
| perm.capture | READ_FRAME_BUFFER | WMS 14:9432 `captureDisplay` |
| perm.inject | INJECT_EVENTS | InputManagerService 14:864-869 `injectInputEventToTarget` |
| perm.tasks | REAL_GET_TASKS | `getTasks` 跨 UID 可见；候选 `ActivityLaunch.kt:101` |
| activities | `pm.hasSystemFeature("android.software.activities_on_secondary_displays") && (ActivityTaskManager.supportsMultiWindow(ctx) \|\| Settings.Global development_force_resizable_activities != 0)` | 【上游】ATMS 14:904-955 计算 `mSupportsMultiDisplay`；`ActivityTaskManager.supportsMultiWindow` 14:252-260（低内存或 `config_supportsMultiWindow=false` 即为 false）；TaskLaunchParamsModifier 14:485-489 / 15:496 / 16:498 不满足时静默改到主屏 |
| now.main | 现有 `deviceFacts()`：interactive、keyguard、deviceLocked、display ON | `DeviceFacts.kt` |

这是"当前状态"，单列一行，不算能力判定。

**实测（默认执行；需要主屏亮且解锁，否则跳过并说明原因；任何一项静态检查失败也跳过）**

建一块与生产完全相同的临时屏，即 `WorkDisplaySupport.create` 同一条路径，同样的 flags 和尺寸，做完立即销毁。全程约 1–2 s，主屏上没有可见变化。前提是 activities 检查通过，否则启动会落到主屏，这正是先做静态检查的原因。

| id | 怎么查 | 失败 cause |
|---|---|---|
| live.flags | `created.display.flags` 含 `FLAG_TRUSTED\|FLAG_OWN_FOCUS\|FLAG_STEAL_TOP_FOCUS_DISABLED`（0x1880）。上游会静默去掉不满足条件的 flag（VirtualDisplayAdapter 14:536-552；DMS 16:1972-1983） | flagsDropped |
| live.on | 预算内等 `state` 从 UNKNOWN 变为 ON（候选 `WorkDisplay.kt:71-77`） | displayNotOn |
| live.launch | 用生产启动路径 `prepareActivityStart` + `setLaunchDisplayId` 启动 AndroidCtl 自带的 `DisplayProbeActivity`（exported、`taskAffinity=""`、纯色背景加一个按钮），带 `NEW_TASK\|MULTIPLE_TASK`，不碰管理页 task。用 §1.2 的规则核落点：探针 task 的 displayId==VD | launchRerouted |
| live.windows | 无障碍 `getWindows` 的 SparseArray 中该 displayId 有探针窗口（包名为 AndroidCtl）。在预算内重试，因为新连接存在首读为空的现象（§1.2 的实测） | windowsNotVisible |
| live.input | 通过 `injectInputEventToTarget(event, mode, appUid)` 在按钮中心注入 DOWN/UP，带 displayId；再经无障碍读到按钮文字变为 `touched` | inputRejected |
| live.frame | `captureDisplay(displayId)` 得到 Bitmap，探针纯色区域像素等于预期色 | noFrame |
| live.focus | 实测前后 display 0 的焦点窗口（无障碍 active 窗口 id，以及 getTasks 中 display0 的 focused task）没有变化 | stealsFocus |
| live.cleanup | 销屏后 `getTasks` 里没有探针 task | 只记一行 warning，不影响 verdict |

可选：电源、锁屏方面的检查不放进探针，那是会话期间的状态，不属于能力。

默认做实测的理由：包内真实失败几乎都只有实测才能暴露，例如 API37 的 state UNKNOWN（`reviews/api37-display-readiness-audit.md`）和新连接窗口目录为空。单做静态检查只能抓到 MuMu 这一类问题。

### 3.4 输出样例

**MuMu（Android 15 / API 35）**。【实测】依据：`runtime-mumu/commands/unsupported-create-once/stdout.bin`（缺 ADD_TRUSTED_DISPLAY）、`initial-status/stdout.bin`（androidApi 35）、`initial-keyguard`（showing=false）、`initial-power`（Awake）、`initial-displays`（Display 0 ON）；研究报告记录 shell 包有 INTERNAL_SYSTEM_WINDOW/INJECT_EVENTS，副屏 feature 为 true（`research/display/report.md` "MuMu平台能力门失败"节）。READ_FRAME_BUFFER、REAL_GET_TASKS 和 supportsMultiWindow 包内没有原件，样例按 AOSP 默认填写。
```
work display: unsupported on this device
  ok    Android 15 (API 35)
  FAIL  ADD_TRUSTED_DISPLAY: the shell user on this system does not hold it; it cannot be granted with pm grant
  ok    INTERNAL_SYSTEM_WINDOW, READ_FRAME_BUFFER, INJECT_EVENTS, REAL_GET_TASKS
  ok    activities on secondary displays
  skip  live display test: blocked by ADD_TRUSTED_DISPLAY
now: main screen on and unlocked
use: session open --display main
```
`--json`：`{"verdict":"unsupported","cause":"missingPermission:ADD_TRUSTED_DISPLAY","checks":[...]}`。同一台机上执行 `session open --display virtual` 得到 `error.code=displayUnsupported, cause=missingPermission:ADD_TRUSTED_DISPLAY`，并且不建屏。

**API34 AVD**。【实测】依据：`runtime-api34/commands/first-create-once`（requestedFlags 83273、actualFlags 6272 = 0x1880）、VD2 720×1280、VLC 截图与输入成功（`result.md`）。
```
work display: ok
  ok    Android 14 (API 34)
  ok    ADD_TRUSTED_DISPLAY, INTERNAL_SYSTEM_WINDOW, READ_FRAME_BUFFER, INJECT_EVENTS, REAL_GET_TASKS
  ok    activities on secondary displays
  ok    live: flags trusted+own-focus+no-steal kept (0x1880), ON after 12 ms
  ok    live: probe activity landed on display 7; windows visible; touch delivered; frame captured
  ok    live: main focus unchanged; probe display removed
now: main screen on and unlocked
use: session open --display virtual
```
（"12 ms"、"display 7" 只是示意，实际数值以运行为准。）

### 3.5 实现规模与测试清单

规模【建议】：
- 设备端 `DisplayCheck.kt` 约 170 行（静态约 50 行，实测约 100 行，结果组装约 20 行）。`DisplayProbeActivity.kt` 约 40 行加 manifest 条目。`WorkDisplay.create` 和 `mainReadyForWorkDisplay` 移入共用的 `WorkDisplaySupport`，行数持平。
- 主机端 `doctor` 命令约 25 行，渲染器约 30 行。
- 合计约 +270 行。

测试：
- JVM：静态判定（各权限、feature、多窗口组合 → cause）；实测各步失败 → cause 映射，使用假平台；`session open --display virtual` 在静态失败时不建屏、返回同一 cause；探针启动带 `MULTIPLE_TASK` 且 `taskAffinity=""`。
- Python：`doctor` 文本和 JSON 渲染；无会话时可运行；`--static` 不发 live 参数。
- 契约：新增 `display.check` 的 ok 与 unsupported 两个样本。

---

## 四、必须上设备验证的清单

| # | 跑什么 | 看什么 | 结果含义 |
|---|---|---|---|
| 1 | API34 和 API37 AVD：`session open --display virtual` 后对 VLC VIEW 执行 open 3 次，并对两个带启动页的 App（自写一个 Splash→Main 的测试 App 和一个发行 App）各 open 一次 | settle 结束时的 `landing`/`foreground`/exit | 全部是 `target`/0：规则成立。出现 `other` 且 foreground=null：settle 窗口对某些 App 太短，要给 open 的 settle 加"本屏出现可见 task"这个早停或延长条件，并在预算内重读一次 |
| 2 | 同一台 AVD：测试 App 在主屏保留一个可见 task，它的跳板 Activity 用 `getApplicationContext().startActivity(NEW_TASK)` 启动另一个 Activity；在 VD 上 open 这个跳板 | `landing` | `elsewhere`/exit 1：确认 TaskLaunchParamsModifier 的进程级回落（14:518-545）会把启动带回主屏，规则 4 有效。仍是 `target`：规则 4 保留但优先级可以降低 |
| 3 | 测试 App 做 5 个启动 activity 变体（standard、singleTop、singleTask、singleInstance、documentLaunchMode=never），每个先在主屏留一个后台 task，再在 VD 上 open | 主屏 task id 和屏号是否不变；VD 上是否新建 task；拒绝时的信息 | 前两种应新建实例且主屏不变，后三种应被拒绝。另外用原始 `MULTIPLE_TASK` 对 singleTask 强发一次，确认主屏 task 被搬走（验证 §2.2 的上游推演）；结果不符就修正判定表 |
| 4 | 同第 3 条，但主屏 task 正在前台，真人用自写的输入 App 持续输入 | 主屏 App 的生命周期日志（onPause 次数）、焦点、输入是否中断 | 新实例路径下应为 0 次 pause、焦点不变。不是这样就说明共享进程或焦点有副作用，要在文档里写明 |
| 5 | 真机：常用 App 20 个，在主屏已有 task 时走新实例路径 | 新实例是否被 App 自行 finish、崩溃、要求重新登录、抢主屏的媒体或相机 | 出问题的比例高（如 >20%）：默认改回"拒绝"，新实例改成显式选项 |
| 6 | 第 3 条之后执行 `session close` | VD 上的新实例是否被销毁；主屏 task 是否完好（同一 task id，返回栈保留） | 不符：DESTROY 语义或 reparent 判断有误，需重做 §2 |
| 7 | 按 §2.6 的量法统计 | a、b、a×b | 决定是否把 `--take-task` 放进首版 |
| 8 | 在 MuMu、API34 AVD、API37 AVD 上执行 `androidctl doctor` 和 `--json` | 是否与 §3.4 样例一致；`session open --display virtual` 的 cause 是否与 doctor 相同 | 不一致说明共用对象没真正共用 |
| 9 | worker 重启后立即执行 `doctor`，以及重启后立即在 VD 上执行 open | live.windows 首读是否为空、多久后出现 | 稳定在 1.5 s 内出现：探针按这个时长重试即可。更久或一直为空：属于观察通道的问题，要单独修（建议同时检查连接后是否需要主动触发一次窗口刷新） |
| 10 | 真人在主屏连续输入时执行 `doctor`（实测级） | 主屏有无闪烁或失焦；最近任务里有没有探针 task | 有任何可见影响：实测改为显式 `--live`，默认只做静态 |
| 11 | 低内存 AVD（`ro.config.low_ram=true`）或没有副屏 feature 的镜像 | 静态检查报 `noSecondaryDisplayActivities`；若强行跳过静态直接实测，会得到 `launchRerouted` | 证明"静默落主屏"在本机真实存在，以及静态门的必要性 |
| 12 | ≥5 个品牌真机（Pixel、Samsung、Xiaomi、OPPO、vivo）执行 `doctor --static`，必要时加实测 | ADD_TRUSTED_DISPLAY 是否存在、flags 是否保留、能否输入和截帧 | 这就是后台屏转正式能力的支持矩阵；失败的品牌在文档中列为不支持 |

---

## 五、未核实与边界

- E4 的原件（`prototypes/device/commands/e4-own-task-*`）不在包内，只采信报告的转述。
- RunningTasks 的排序、`isVisible` 在跳板过渡期间的取值，只读了上游源码，未在设备上核对（§4 第 1 条）。
- 做法 3 里"把 task 放到主屏底部需要 organizer、会与 SystemUI 冲突"只是推断，没有逐行核对 DisplayAreaOrganizerController。这个结论不影响"不采用借用"，因为 worker 死亡时无法归还这一条已经足以否定它。
- 无障碍窗口目录在新连接后为空的原因没有在上游定位，只作为设备检查项列出（§4 第 9 条）。
