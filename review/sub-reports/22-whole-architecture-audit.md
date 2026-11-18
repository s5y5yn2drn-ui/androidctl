# 22 整体架构与组合一致性对抗审查

范围：三件事，分别是整体方向、实施方对上轮外评的事实反驳逐条核查、内部 review 质量。
路径简称：`T` = `NEW/ultragoal/2026-11-11-next-improve`；`RC` = review-copy（3f45af6）；`W` = `scratchpad/agents4/22/`。
统计脚本和中间数据：`W/surface.py`（命令、选项、隐藏 API、skill 规则句数）、`W/codes.py`（拒绝码、结果键）、`W/jpgdim.py`、`W/titleref.py`，输出在 `W/surface.json`。输入是用 `git archive` 从 RC（3f45af6、7d5be70）和拆分实验仓库 `agents2/work12/repo`（split/* 分支）导出的树，放在 `W/trees/`。全部是静态正则计数，没有执行包内代码。

## 1. 结论摘要

1. 两边的偏差方向相反，而且都是系统性的。实施方的处理方式是：凡是不能证明等价的就保留，每个窄的区分都单独留一份状态，用"已选/已关闭 review"代替"现在还该不该有"的论证。结果是 U26 研究最后没有删掉任何一个能力级入口，反而新增了 4 项机制。外评的处理方式是：以采用频率和代码体量作删除理由，低估了借贴、通知写在后台屏并行场景下的能力损失，同时又自加 doctor、SET_SELECTION 等新面。
2. 概念面实数（原件统计，脚本 `W/surface.py`）：基线到候选，CLI 叶命令 47→71，长选项 38→68，Kotlin 文件/行 57/9,360→97/18,226，隐藏 API 成员 34→57，`Class.forName` 字面量 14→29，skill 文件/词 2/11,534→12/20,995，规范性句子 331→594，设备拒绝码 95→123，结果键 233→509。实施方方案落地后各项不减，只增：隐藏成员约 +12、类 +3、生命周期对象 +1、选项 +1。外评方案按拆分实测，命令 −7（另加 doctor +1），选项约 −8，隐藏成员 −6，Kotlin −1.4k 行，规则句 −78。
3. 组合一致性：实施方的"一条一致产品链"里有 4 行，实际是把不一致写成了"各有职责"。具体是：显示意图 4 个来源；"是否变化"5 套编码；窗口缺席 2 套语义（动作后判 settle=changed，wait 等到截止）；清理失败 2 套退出码（通知 exit 1，其余 exit 3）。
4. 我推荐的整体方案 E 是：能力基本保留，入口和状态收紧。具体做法：
   - 会话全显式，删偏好文件、`display select/create`，由 `session open --display` 统一入口；
   - IC 和借贴统一放到 `type --via` 下；
   - 删除持久 diff 基线；
   - zoom 并入 `screenshot --region`，不再驻留原生源；
   - 不加 WindowInfosListener，不加每 lease 持有 listener；
   - 倾向用公开 cache 替换 E9 的手写回调。

   实施方推荐的启动统一门、动作排序 B、阅读 C 和显式 new-task 可以接受。
5. 实施方对外评的事实反驳中，成立或基本成立的有：zoom 分辨率、P8 读拒写允、Termux 粘贴已跑、D15、D16 的"无意义"、D9 的缺陷定性、恒差 10、子进程不变差、Title REF 截断。部分成立的有：next、callback 告警、E9 阈值。不成立或偷换对象的有：settle"不会提前结束"、diff"跨会话"、dense、1884。其中关键一点是：Termux 的更正事实上削弱了实施方自己保留 IC 的论据。
6. 内部 review 只审了表述、证据边界和选项完整性，没有一次挑战"该不该有"。它们的修订都在加条件、加选项。isFocused 事件说明：review 会从上游源码出发加强门，只有碰巧有原件反证时才撤回。D5 的"全局封锁"在推荐的 C 下只是移到了 lease 结束时，但被当作关闭处理。
7. 提交策略方面，我建议先把最终树做出来并验证，再从最终树重新切片，不在 40 个提交上回退式重排，也不长期保留"40+11"的追加历史。

## 2. 独立判断

### 2.1 第一件：整体方向

#### Q1 用同一把尺子评两边的总方向

| 维度 | 实施方：全保留＋修缺陷＋精简呈现＋新增机制 | 上轮外评：瘦身＋显式会话＋三项待验 |
| --- | --- | --- |
| 能力 | 最强。所有入口都在，另外加了 new-task、坐标预测。 | 损失真实。删借贴后，后台屏和真人并行时只剩 `clipboard set`（永久覆盖），或者用 keys（只能 ASCII）。通知写在后台屏模式下的价值（不用拉主屏 shade）被低估。zoom 的"同帧"价值被一句"未缩放"带过，而这句话本身是错的（见 §3）。 |
| 调用者负担 | 负担最高。71 个命令、68 个选项、21k 词的 skill。新增规则包括显式 open 和 select 偏好的关系、坐标预测失败时什么时候必须 expect、new-task 的 singleton 例外。题 1 选 D 后要求调用者同时理解 open、select、create 三个动作。 | 负担较低，但它另加了 doctor 和 SET_SELECTION，又提出 key/index+postTime 地址协议，部分抵消了收益。 |
| 维护责任 | 不减反增。新增一个隐藏抽象类子类化（WindowInfosListener）、跨命令的 listener/URI grant 寿命、一个新启动选项及其 singleton 语义。 | 实测的拆分规模是 Kotlin −2,899/+262（含后台屏整块，不是外评目标态）。主机偏好删除后，D1 的 22 项 POSIX 失败随之消失（`W/trees` 对照：split_rv38 只剩 1 failed）。 |
| 系统性偏差 | ① 把"不等价"当成保留的充分理由（例如：借贴替代"不是行为等价"、IC"不因 Termux 成功重开删除"）。② 每个窄区分都留状态（会话 D 同时保 lease、偏好、每请求 kind、batch 冻结）。③ 用"已选""已关闭 review"代替现时论证；处置表第 13 行写明"已选能力不要求用户再次证明价值"。④ 新增机制时只和"现状"比，不和"更简单的公开等价物"比（见 §2.1 Q4 坐标）。 | ① 用采用次数（"四次任务零调用"）和行数作删除依据，这违反了用户"不按调用频率"的明确偏好。② 用单一场景的失败或成功做二选一的决策门（Termux 粘贴成功就删 IC；1.5s 门）。③ 事实断言有错（zoom、Termux 端到端从未跑、D16、D15），部分删除论据因此失去支撑。④ 删除时没有按"后台屏并行真人"这个核心场景重新评估每个能力。 |

总评（设计建议）：
- 实施方在"修确定缺陷、统一阅读 View、统一启动门、单一 live 来源、动作排序"这些点上，是真实的收敛。但它把 U26 用户原话里"积极提出合并、重写或删除"落实成了"删内部旧路径"，没有在能力入口和状态层面做任何合并。
- 外评的方向（减少面）是对的，但删错了对象，事实依据也有缺口。

#### Q2 概念面实数

统计方法：`python3 -I W/surface.py W/trees/<tree>...`。CLI 叶命令取 typer 的 `.command("x")`、`add_action/add_global("x")`、`for command in (...)`、`name="help"` 和裸 group，与 `skills/androidctl/reference.md` 的命令表对照，差 1 以内。选项按 host CLI 模块中不重复的 `"--xxx"` 字面量计。隐藏 API 按每个文件中 `get(Declared)Method/Field("名")` 的不重复名计，另计 `Class.forName("…")`。规则句是 skill/*.md 中含 must/never/only/cannot/requires/unless 等词的句子，只作代理指标。拒绝码按 `Rejected("code"` 计，结果键按 `put("k")`/`"k" to` 计（`W/codes.py`）。

| 指标（原件统计） | 7d5be70 基线 | 3f45af6 候选 | 拆 diff | 拆借贴 | 拆通知写 | 拆 zoom | 拆 IC | 拆后台屏整块 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| CLI 叶命令 | 47 | 71 | 71 | 71 | 67 | 70 | 69 | 69 |
| 长选项 | 38 | 68 | 67 | 68 | 62 | 67 | 68 | 68 |
| 隐藏成员 / forName 类 | 34/14 | 57/29 | 57 | 57 | 51 | 57 | 57 | 50 |
| Kotlin 行 | 9,360 | 18,226 | — | −494 | −631 | −303 | −501 | −707 |
| host Python 行 | 8,007 | 11,958 | −632 | −28 | −184 | −89 | −90 | −251 |
| skill 词 / 规则句 | 11,534/331 | 20,995/594 | −782/−25 | −392/−15 | −606/−23 | −442/−15 | −420/−15 | −1,066/−28 |
| 拒绝码 / 结果键 | 95/233 | 123/509 | 0/0 | −2/−8 | −4/−18 | −1/−5 | −2/−11 | −6/−15 |

新增选项 31 个（`surface.json` flag 差集），其中包括 `--diff --via --expect --region --hold --post-time --creator-uid --result-key --grant-read-uri-permission` 等。

输入通道（源码核对 `RpcMethods.kt` 和各通道文件），基线 4 个，候选 10 个：
- 基线：AX 动作、坐标 tap/swipe、按键、open/系统键；
- 候选新增：节点 touch、IC、借贴、clipboard 写、通知写、range、drag/pinch/timeline。

每个通道的接收者定向规则不同：坐标按操作窗根包 UID（`VisualInput.kt:232-240`），节点 touch 按节点 UID，无 REF 的按键不定向，IC 只用 main 的焦点 editor，通知走 PendingIntent，剪贴板是全局。

"是否变化"的结果编码有 5 套：`settle`、`observedChange`（state 加 windows/selection/content 三维）、`effect`（UiEffect）、局部 `verification`、host `diff`。此外 open 另有 `landing`。

设备端和 host 端的独立状态机或生命周期对象（源码核对），基线约 9 个，候选 19 个：
- 基线已有：lease（`Sessions.kt`）、UiA 连接（`UiBackend`、`AccessibilityConnection`）、REF/快照缓存（`UiRefs`、`UiSnapshot`）、坐标帧（`VisualInput`）、shell job、file job/传输句柄、worker lock、Gateway、daemon。
- 候选新增 10 个：VD 目标（`WorkDisplay`、`DisplayTarget`）、host 显示偏好、E9 每次 collect 的回调和机会表（`CollectionNodes`）、原生图源驻留（`VisualInput` 的 ImageState，zoom）、事件 ring（`UiRecentEvents`）、通知 listener token（`NotificationOperations`）、IC 桥（`EditorBridge`、`CurrentEditorInput`、`WorkerConnection` Attached、`BrowserAccessibilityService`）、借用剪贴板（`BorrowedPaste`）、剪贴板空准入采样（`ClipboardAccess` emptyBasis）、host diff 基线（`history.py:17`，`ui_diff.py`）。
- Kotlin 顶层 `var` 数从 181 增到 288。

各方案落地后的变化：

| 方案 | 命令 | 选项 | 隐藏成员/类 | 生命周期对象 | 依据 |
| --- | ---: | ---: | ---: | ---: | --- |
| 实施方推荐组合（题 1D、2C、4C、7C） | ±0 | +1（new-task） | 约 +12/+3（compile-only 桩 `T/prototypes/route-review/coordinate-window/compile-only/` 中有 WindowInfosListener 的 register、unregister、onWindowInfosChanged 和 DisplayInfo 的 3 个字段，InputWindowHandle 的 7 个字段）。单一 live 来源最多 −1（`getAllRootTaskInfos` 仍有其它 roots 消费者，实施方声明不全局删）。 | +1（窗口信息注册/释放）。listener 寿命从每命令延长到每 lease，对象数不变但存续期变长。 | 推断（来自其题面和桩），尚无实现 |
| 外评方案（删 diff、借贴、通知写、zoom，会话显式且删偏好，加 doctor） | 71→65（−4、−1、−2，+1） | 约 −9 | −6 | −4（diff 基线、借用剪贴板、图源驻留、host 偏好） | 拆分分支实测，加上设计推断 |
| 本报告方案 E（§2.1 Q4） | 71→66 | 约 −5 | −7（若 E9 换 cache）或 0 | −3（host 偏好、diff 基线、图源驻留） | 推断 |

#### Q3 组合一致性：沿链走一遍

| 链段 | 多个事实来源 / 一个概念多套语义 / 失败语义不一致 | 位置（3f45af6） | 实施方表格怎么写 |
| --- | --- | --- | --- |
| 会话到显示 | "我在哪个屏"有 4 个来源：设备 lease target、host 偏好文件、每请求 expectedDisplayKind、batch 起始冻结 kind。另外 `display create` 只在 done 后写偏好（D10）。 | `app.py:839`；`Sessions.keepLease` / `requireTarget`；`state.py` | "共享偏好只表示意图"。等于是把多来源写成了"各有职责"。题 1D 保留了全部 4 个来源。 |
| 显示到窗口 | 窗口缺席有 2 套语义。动作后观察：`run.absentWindow` 一旦设置就停止采样，并把 `settle` 写成 `"changed"`（`UiBackend.kt:424`；`UiPostAction.kt:43-44,67,101-108`）。`wait --window`：继续等到截止（D16）。 | 同左；`TreeCollector.kt:111-118` | 只写了"显式窗口暂缺仍等原截止"。实施方用来否定 D16 的理由是"目录缺席可能只是不可见"，同一理由同样否定动作后路径的"changed"，但它没有处理。这是确定的不一致，源码核对。 |
| 节点到变化 | settle 分类里也有 CHANGED（`UiPostAction.kt:189-197`），与 observedChange、UiEffect、diff 重叠。partial 基线的 settle 永远是 UNSETTLED，并且不会提前结束（见 §5）。 | `UiPostAction.kt`、`ObservedChange.kt`、`UiEffect.kt`、`ui_diff.py` | "各有消费者，不能合成一个布尔值"。真正不同的消费者只有"局部核值"和"屏幕变化"两类，其余是重复编码。 |
| 输入通道 | 接收者定向有 5 种规则（见 Q2）；坐标的定向和 IME 冲突已经实证。 | `VisualInput.kt:232-269`，`UiScope.kt:50-52` | "保各自准入"。准入各自不同是合理的，但"默认接收者"这个概念只该有一份规则。 |
| 图源到坐标资格 | 同一个 ImageState 同时持有只读源（zoom）和输入资格。树签名变化会同时失效两者，zoom 又不刷新年龄。 | `VisualInput.kt:39-44,60,120,132`；`ImageRegion.kt:26` | 实施方在 `delivery-cost.md` §5.2 承认两者共用状态，拆开需要额外责任。表格里写成了"失效规则保持"。 |
| 权限与清理 | 清理失败有 2 套退出码：通知清理未确认时，done 被改成 exit 1（`NotificationOperations.kt:111`）；总规则和其它路径是 exit 3（`reference.md:68`；`coordinator.py:57-58`；`file_jobs.py:136`；`ShellProtocol.kt:156`）。通知注销失败会封住全部通道（`Sessions.kt:198`），其它清理失败只走 `uiCleanup` 拒绝。 | 同左 | "清理报告实际事实"。题 4C 不改退出码，封锁也只是移到 lease 结束时。 |
| 交付 | 一份 View 加 JSON/raw，前后一致。 | `ui_view.py` | 自洽，同意。 |
| 结果到退出码 | VD 上 open 的平台 START 成功但落点未确认时，result 是 done、exit 是 1（`ActivityLaunch.kt:254-256`）。result 和 exit 两套编码不一致，调用者要额外记住。 | 同左 | 题 6C 统一停批，但没有统一编码。 |

结论：表中"节点与即时对象""定稿与交付"两行是自洽的。"控制占用与显示意图""事件与观察""输入通道""权限与清理"四行，是把多来源或多语义写成了"各有职责"。

#### Q4 更好的整体方案 E（设计建议）

**必须保留（不可替代的场景）**
- UiA 节点链加 REF 动作，fill/check 局部核值，`scroll-until/reveal/find`。这是核心。
- 后台屏，因为真人可以同时用主屏。这是已实测的独有能力。
- typed Intent 加落点。
- 通知 list/get。
- 通知写（action/open/dismiss/reply）。后台屏任务处理通知时不用拉下主屏 shade；shade 替代路线会占用真人主屏，而且丢失 PI 地址。
- 借贴。后台屏和真人并行时，它是唯一能插入任意 Unicode、又尽量还原真人剪贴板的通道。`clipboard set` 是永久覆盖，更伤真人。这一点更正我方旧结论，见 §4。
- IC。在 main 上不碰共享剪贴板，可以替换选区。E13 已经证明 Termux 和 WPS 有剪贴板替代路线（`T/research/input/followup/report.md:9-12,68`），所以 IC 的独有价值只剩"不碰剪贴板＋选区替换"，属于中等价值。

**合并入口（不减能力）**
1. `session open [--display main|virtual]` 作为唯一的建立和选择入口。删除 `display select`、`display create`、host 偏好文件和每请求 kind。效果：命令 −2，host 状态 −1，wire 字段 −1；D1 的 22 项 POSIX 失败随之消失（拆分实测 rv38 失败数 23→12→1）。代价是同名共享会话失去种类保护，但实施方自己也写了"每任务独立名字是使用约定"。
2. 文本插入统一为 `type [REF] TEXT --via keys|editor|paste [--range S E]`。删除 `editor commit/select` 这一组（命令 −2，组 −1），选区设置和提交放在同一请求里做，并且先核选区再提交（未生效就报 not sent）。代价是失去"只 select 不 commit，再接其它通道"的组合；目前没有发现这种组合的真实消费者（推断）。
3. zoom 并入 `screenshot --region L T R B [--ref]`：取一次新的原生分辨率截图并裁剪，不驻留每 lease 一帧的原生源，crop 也不当坐标帧。这样命令 −1，生命周期对象 −1，"zoom 不刷新年龄、签名失效"这组规则也随之消失。代价是失去同帧历史和"已过期图的只读 crop"（MuMu 原件中有 121s 的例子）。细字收益在原件里本来就没有证明（实施方 `delivery-cost.md` §5.1 同意）。是否保留同帧，需要设备验证（§7 V3）。
4. "变化"的默认呈现只给两类：局部核值，以及一行屏幕变化（来自 observedChange）。settle 只回答"是否稳定"，不再出现 `"changed"`。窗口缺席在动作后和 wait 里统一成一种语义。

**删除或集中负责**
- 删除持久化的 diff 基线（`history.py` BASELINE、`ui_diff.py`）。若要保留变化列表能力，改为同一命令内的动作前后比较，无状态。"读者已收到的 View"这一基线语义的代价（配额、batch 交叉、跨会话 contextChanged），比它的收益大，属于证据不足下的设计建议。
- 清理失败统一为 exit 3，并且只保留所属资源，不封锁无关通道。这是题 4 的 B 的简化版，不需要局部 pending 系统：注销未确认时保留 token，在 lease 结束时再处理。
- E9 倾向换成公开 cache 窗口。实测 collect：E9 是 606/580ms，cache 是 655/1,047ms（`collector-cache/runtime-mumu-clone/result.md` 表）。可以删掉手写的回调 Binder，以及只被 `CollectionNodes.kt` 使用的 7 个隐藏句柄（childId、findNode、leashedNodeId、nodeTransaction、prefetchTransaction、setNodeConnection、setNodeSealed；源码核对，`W` 映射脚本输出）。代价是接受平台 cache 的来源契约。细节交给节点专题。

**不该加的新机制**
- WindowInfosListener 坐标预测（题 2C）。它带来隐藏抽象类子类化、长驻 worker 中的注册/注销、变换数学，代价约 +12 个成员。已经实证的失败只有 IME 这一类，而公开路径大概率就能覆盖：`AccessibilityWindowInfo` 有 IME 类型和 `getRegionInScreen`，默认 IME 包可从 `Settings.Secure.DEFAULT_INPUT_METHOD` 取得并映射到 UID，所以只需在 IME 区域内换成 IME UID，其余窗口保持操作窗 UID 或要求 `--expect`。这是推断，需要设备验证（§7 V1）。如果不可行，就选 A：默认不定向，`--expect` 时显式定向。
- 每 lease 持有通知 listener（题 4C）。它延长了跨 user 回调和 shell URI grant 的寿命，换来的只是每条命令省掉一次注册和注销；这笔成本没有测过。保持每命令注册，同时修正失败语义。
- 主机偏好加"全显式"（题 1D）。同上，选 1B。

**接受的实施方新增**：题 5B（排序后再建事实）；题 6C（main 和 VD 用统一门，单一 live 来源，删掉 roots/live 互证）；题 3C 加一份 View；题 7C 的 `--new-task`（+1 选项，能力真实）。

相对收益与代价：
- 相对实施方：命令 −5，生命周期对象 −3，隐藏成员 −7 到 −19（不加 WindowInfosListener，cache 替换 E9），规则句大约少 60 到 80。代价是失去同名种类保护、zoom 同帧、持久 diff，以及在 C 能覆盖、公开路径不能覆盖的少数系统窗口上的自动定向。
- 相对外评：保留通知写和借贴；不加 doctor 和 SET_SELECTION；IC 的去留改为只合并入口。

#### Q5 追加修正还是重排/重写

依据：
- 拆分实验原件存在。`agents2/work12/repo` 的 split/* 分支可以核对。例如 `72fe5d6 fix: correct silent UiRun merge (restore missingFactReason path)` 证实了 UiRun 静默合并；all-six 分支有 `4fec8b0 test: restore local_call import`。这些和专题 12 第 12、46、63、75 行的 5 处静默合并一致。实施方说"本包无可审临时 patch/日志"，从字面看是成立的（我们没有随包提供），但这些原件真实存在。
- 测试结果（原件 `work12/results/*.sum`）：head 是 23 failed/1,799 passed，all-six 是 1 failed/1,550 passed，统计的是 `androidctl/tests`。我方主报告写的"1,850/1,601"多了 51，应当是把 tools 测试也算进去的推算值，未在拆分分支实测（见 §4）。

判断（设计建议）：
- 只在 40 个提交上追加 11 个修正，最终历史会留下"先加后改"：偏好写入、listener 寿命、launch 互证、动作次序等都会被改两次。合入主仓后无法回收，审阅者要读完 51 个提交才能理解最终契约。
- 先在旧序列上回退式重排，同样有风险。实测 6 块里出了 5 处静默错误，而 Kotlin 部分只做了静态检查。
- 更好的做法：用户先定能力集；在候选顶上前向实现最终树，做全量 CI 和设备验收；然后从最终树按依赖重新切片，每片构建测试，最后 `git diff` 切片 HEAD 与最终树应为空。这样避免了回退合并的静默错误，也不保留 churn。

### 2.2 第二件：实施方事实反驳逐条核查（见 §3 表）

### 2.3 第三件：内部 review 质量

- **是否挑战方向**：`route-review-whole-contracts.md` 的 M1–M3 和 `route-review-whole-maintenance.md` 的 M1–M2 都是关于表述或题面完整性的：落点唯一性措辞、numeric ID 两处说法不一致、main 新停批代价、移入和借用归还分开。`route-review-decisions.md` 的 M1 是"会话缺了保现状的选项"，M2 是"listener 跨 user 要写明"，M3/M4 是"动作次序题要补设备比较消费者"。这里没有一项是"这个能力或机制不该存在"或"能否合并入口"。维护 review 甚至直接背书："保 IC、通知写……的建议也没有只诉诸投入"，又说"没有必要另造第五方案"。合同 review 的末节写"七题都有不能由实验替代的……"，对总面积没有做任何计数。这是原件核对。
- **review 的修订方向是加法**：decisions-M1 加一个选项，M3/M4 加比较消费者，contracts-M1 先提出加 isFocused 强门，之后才撤回。没有一条 must-fix 让方案变得更小。
- **isFocused 事件说明了什么**：reviewer 读上游 `Task.java` 后提出了一个新的强制门。之后依靠既有原件 `contacts-after-b-edit-windows/stdout.bin` 才发现 VD3 的 `mCurrentFocus` 是 Contacts、`mFocusedApp=null`（我核对过第 117–118 行），于是撤回。这说明两点：(a) review 倾向于从源码推导出更强的门；(b) 撤回依赖巧合存在的原件。没有原件的门（例如 C 的 user0、profile、非 identity 变换规则）就没有同样的反证机会。
- **关闭了但实质未解决的问题**：
  - D5 在推荐的 4C 下，注销失败"仍可经现门阻塞"（推荐稿题 4）。全局封锁没有消失，只是从每命令挪到了 lease 结束时。维护 review 写了"不能称已消除 D5"，却没有列为 must-fix。
  - 通知清理的 exit 1 与总规则 exit 3 不一致，没有任何 review 提出。
  - 动作后窗口缺席判 `settle="changed"`，与 D16 的处置矛盾，没有被提出。
  - 处置表内部还有过期陈述没有清理：第 25 行说有限 C"全部独立 review 均闭合"，第 107 行却说"runtime 增量 review 未闭合"；第 186 行仍写 Contacts"增量 review 正进行"。合同 review 的 Optional-1 只处理了"五题和七题"的不一致。
- **证据边界审得认真**：Contacts 对照的数字和因果限制、cache 首份 raw 缺失，这些都核得细。所以问题不在严谨度，而在 review 的提问层级：它只问"这样写准不准"，不问"这样设计该不该"。

## 3. 对照实施方：事实反驳逐条

外评原文位置：主报告用 `M§`，专题用 `S编号`。实施方位置：处置表用 `D`（`external-disposition.md` 的行号），整合稿用 `I`，专题用其文件名。

| # | 实施方指认 | 外评原文 | 实施方位置 | 我核对的原件/源码 | 结论 | 是否改变方向 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | zoom"仅 720×1280、从未缩放"是错的 | M§2.6 第 205 行；S04 第 19、84、168 行 | D 第 70、172 行；`delivery-cost.md` §5.1 | `T/stages/36-native-zoom/runtime-mumu/`：`settings-full.jpg`、`apps-independent.jpg` 为 720×1280（`W/jpgdim.py`），`full-source-roi.png` 为 1080×1920，trace 中 `sourceWidth=1080`。旧包同路径存在。 | 成立（原件核对） | 推翻了"zoom 无缩放"这个前提。但细字收益仍未证；E 方案以"新截图裁剪"替代，理由是状态，不是分辨率。 |
| 2 | 借贴/剪贴板：P8 已证 READ ignore/WRITE allow | M§2.3 第 142 行："包内没有实际发生过" | D 第 68、98 行；`inputs.md` §3.1 | 旧包 `stages/35-clipboard/p8-structured/runtime-api34/result.md:21-33`：人为把 shell 的 READ_CLIPBOARD 设为 ignore，WRITE 不变，之后 set 生效、Termux 实际粘贴了 B。 | 部分成立：组合在包内真实发生过，我方说法错误；但它是用 appops 人为诱发的，不能证明自然使用中常见（推断）。 | emptyBasis 有真实消费者。这不能证明借贴必要；借贴的价值在后台屏并行（§2.1）。 |
| 3 | IC：Termux 路线已有分层实证 | M§3.1 第 246 行："端到端从未跑过" | D 第 105、190 行 | 旧包 `research/input/followup/report.md:12`：borrow 写入、目标 UID 的 Ctrl+Alt+V、图上 13 字符 Unicode、restore。同一报告第 68 行把"不纳 IC"评为"合理替代"。 | 成立，我方事实错误。但这条证据削弱的是 IC 的唯一性，与实施方保留 IC 的论证方向相反。 | IC 降为"中等独有价值"，即不碰剪贴板加选区替换。E 方案把它合并进 `type --via` 入口，不删。 |
| 4 | D15"覆盖原失败"不成立 | M§7 D15 | D 第 161 行；I 第 19、78 行 | `TargetManagement.kt:61-89`：catch 先 remember FAILED/UNKNOWN；finally 中 `run.failure(error)` 对已有 fact 返回副本，只在 data 中加 cleanupError（`UiRun.kt:116-124`）。只有 NOT_SENT 时才改 reason，而这里的 catch 不会产生 NOT_SENT。 | 成立（源码核对） | 不改变方向；删除 D15。 |
| 5 | D16"无意义等待"证据不足 | M§7 D16："窗口 ID 不会复用，空等没有意义" | D 第 162 行 | AOSP 的无障碍 windowId 按窗口 token 分配，窗口隐藏后再显示仍是同一 id（推断，依据上游 AccessibilityWindowManager 的语义，本轮未重新取源）。目录缺席不等于 gone。 | 成立（推断） | 但实施方没有处理动作后路径对同一缺席判 `settle=changed`（§2.1 Q3、§5）。这一点改变方向：应当统一两处语义，而不只是保留等待。 |
| 6 | D9 是可解释的组合，不是缺陷 | M§7 D9 | D 第 155 行；I 第 72 行 | `PointerGesture.kt:197-213`：先 `dispatch(UNKNOWN)`（`UiBackend.kt:309-317` 中 begin 加图失效），再 `event.send()`。拒绝是在 IPC 之后由平台返回的，所以 dispatchStarted=true 符合事实。 | 成立（源码核对）：我方"确定缺陷"定性过强。 | 不改变方向。"首拒后强制重截图"作为成本保留。 |
| 7 | "多出时间全在模型侧"过强 | M§4.1 第 325 行 | `delivery-cost.md` §1 | `paired-runtime/*/native-metrics.json` 的 `cli_seconds_sum.child`：Contacts 24.23→19.01，Chrome 24.56→27.32（+2.76s）；wall 减 child 的余量分别增加 40.8s 和 41.8s。 | 成立（原件核对） | 不改变方向，主因仍是 wall 余量。 |
| 8 | "恒差 10"不能概括 | M§4.3 第 357 行；S10 第 266、274 行 | `delivery-cost.md` §1 | S10 第 164–167 行自己写了 chrome-2 残差范围是 −445 到 1,115。 | 成立（原件核对），是我方自相矛盾。 | 不改变方向。 |
| 9 | next"0 处"错，已有约 20 处 | S08 第 151 行"0 处实现" | D 第 197 行 | S09 第 185 行和 M§8.8 第 929 行已经改成"约 20 处已有"。 | 部分成立：对 S08 成立，主报告已经自我更正。 | 无影响。 |
| 10 | 离线重渲染中 Title REF 被默认上限截掉 | M§4.5 第 386、392 行（"默认上限只截短了 --all 那一条，−8.4k"） | `delivery-cost.md` §3 | `contacts-2` 第 9 条 `snapshot --cached --all` 的 child.stdout：`e834208151 "Title"` 位于第 66 行，节点行累计约 20.2k 字符（`W/titleref.py`），超过 12k 上限；第 10 条正是 `reveal e834208151`。 | 成立（离线重算）。我方把默认上限施加到了显式 `--all` 上，因此 1.13× 不可执行。去掉这项后，Contacts 约为 19.5k，相当于基线的 2.0×。 | 精简方向不变，但我方给出的"1.13×/0.74×"标题数字不能再用。 |
| 11 | dense 8,377,092 是 fixture | M§8.4 第 907 行、M§12 第 1043 行 | D 第 173 行 | 我方原文已写"任务自建测试 App"。 | 不是更正，双方一致。分歧只在"改 row"这个设计。 | 无影响。 |
| 12 | diff"每次持久化/跨 session"不实 | M§2.5 第 186 行："每次 --diff 要持久化……多个会话互相重置基线" | D 第 174 行 | `history.py:17` 每个 workspace 只有一份 `diff-baseline.json`；`ui_diff.py:35-42,247` 按 device/session 判断 contextChanged。 | 不成立。我方原文是"每次 `--diff`"和"互相重置"，两者都是事实。实施方驳的是"每次调用持久化/跨会话误比"，这些不是我方的说法。 | 无影响。 |
| 13 | "1884 passed"和拆出通过的范围 | M§7 D1、M§12.8、M§1 第 72 行 | D 第 175 行 | 我方已写"只在 Windows 上成立"、"Kotlin 只做静态检查"。`work12/results`：all-six 是 1,550 passed，不是主报告的 1,601（见 §4）。 | 实施方的说法与我方一致，不算更正；但它附带暴露了我方数字的问题。 | 无影响。 |
| 14 | E9 三分支阈值是未验证外推 | M§3.3 第 300–306 行 | D 第 108 行 | 新原件：single 热采 collect 1,909ms，E9 为 399–1,024ms，cache 为 655–1,047ms。 | 部分成立：1.5s/3s 门确实是任意设定的。但按新数据，cache 和 E9 的 collect 在同一量级，可以删掉 7 个隐藏句柄。实施方"保 E9"的论据并不比 cache 强。 | 改变方向：应当正面比较换 cache（交节点专题）。 |
| 15 | callback 告警是推断 | M§3.3 第 292 行："静默退化成只有根节点、不报错" | D 第 109 行；`nodes-waits.md` §2 | `CollectionNodes.kt:42-71`：primary 解析失败时 `child` 返回 null，节点被计为 unavailableChildren，即 partial，并非不报错，而是错报成 provider 不可用；prefetch 失败只会退回直接查询。 | 部分成立：我方"只有根、不报错"过强，"错误归因、难以察觉"成立。 | 不改变方向。 |
| 16 | settle：系统动作可提前结束、partial 有保守理由 | M§8.2 第 896 行 | D 第 110 行；I 第 20 行 | `UiPostAction.kt:63-67,189-200`：before==null 时，`stableWithoutBaseline` 可以提前结束，但报告恒为 UNSETTLED，这与我方"必然报告未稳定"一致。before 为 partial 时，classification 恒为 UNSETTLED，stableWithoutBaseline 为 false，所以不会提前 break，会跑到 until。 | 不成立。实施方驳的"不会提前结束"是针对 null 基线的；我方的"无法提前结束"说的是 partial 基线。两个断言都成立。 | 维持：partial 页的每个动作都耗满 settle（§5）。 |
| 17 | b-disposition"采用"未落地不等于违约 | S08 第 35、151 行；M§12.6 | D 第 198 行 | S09 第 20 行已经收窄成"不是交付清单"。 | 部分成立：对 S08 的"违约式"定性成立，主报告已经收窄。 | 无影响。 |
| 18 | 拆出和静默合并只是作者报告 | M§2.1 第 74 行 | D 第 81 行 | `work12/repo` 中 `72fe5d6`、`4fec8b0` 等提交真实存在，results/*.sum 也在。 | 实施方的说法字面成立（包内没有），但原件存在，可以核对。 | 支持 §2.1 Q5 的判断：回退式重排有静默风险。 |

对实施方其它整体主张的同意与不同意：
- **同意**：单一 live 来源和启动统一门（题 6C）；一份 View；题 5B；D12、D13、D14、D7 修正；不新增 wire codec；不加 doctor 和 MessagingStyle。
- **不同意**：题 1D（应选 B）；题 2C（应选公开 IME 区域方案 E，或 A）；题 4C（应保持每命令注册并统一失败语义）；"E9 不需再问"（应正面比较 cache）；D16 只保等待、不统一动作后语义。

## 4. 对上轮外评（我方）的更正

1. M§2.6 和 S04 的"全部图像 720×1280、JPEG 未缩放"是错的。旧包已经有 MuMu 1080×1920 的原件。
2. M§3.1 的"Termux 剪贴板加 Ctrl+Alt+V 端到端从未跑过"是错的。旧包 E13 已经跑通 Unicode 版本（`research/input/followup/report.md:12`），P8 也有 ASCII 正例。按我方自己的规则，应当得出"Termux 不需要 IC"；IC 的去留应当转到"不碰剪贴板＋选区"这个独有价值上评估。
3. M§2.3 的"读拒写允组合包内没有实际发生过"是错的，P8 人为诱发过这个组合。同时，"借贴只多换来自动恢复"低估了后台屏并行场景：借贴是唯一尽量不伤真人剪贴板的 Unicode 插入通道。删除借贴的建议应当撤回。
4. M§4.5 的 1.13×、0.74× 依赖了把默认上限施加到显式 `--all` 上。这会截掉下一步要用的 Title REF，因此不可执行；去掉这项后 Contacts 约为 2.0×。
5. M§4.1 的"CLI 子进程耗时没有变差"不对，Chrome 增加了 2.76s。M§4.3 的"恒差 10"与 S10 自己的数据矛盾。
6. M§7 中，D15 不成立；D9 定为"确定缺陷"过强；D16 的"空等没有意义"过强（正确的方向是统一两处缺席语义）。
7. M§1 和 M§11 的"pytest 1,601 通过"在拆分原件中没有对应。原件是 1,550（androidctl/tests）；1,601 和 1,850 都是加了 51 项 tools 测试的推算值，没有在拆分分支实测（`agents2/12-split-cost.md:34`）。
8. M§3.3 的"静默退化成只有根"过强，实际是"错报成 provider 不可用的 partial"。
9. 外评自加的新面（doctor、SET_SELECTION、MessagingStyle、key/index 地址）与它自己的瘦身方向冲突，应当去掉。

## 5. 确定缺陷（本人核过）

| 位置 | 后果 | 类别 |
| --- | --- | --- |
| `UiBackend.kt:424`；`UiPostAction.kt:43-44,67,101-108`，对比 wait 路径（D16） | 动作后请求窗口缺席时，立即停止采样，`settle` 被写成 `"changed"`；同样的缺席在 wait 中却继续等到截止。同一事实两套语义，而且 settle 字段被借用来表达缺席。 | 源码核对 |
| `UiPostAction.kt:189-200` | 动作前基线为 partial（Chrome 长页常见）时，分类恒为 UNSETTLED，并且不会提前 break，每个动作都耗满 settle 窗口并反复整树采样。实施方 I 第 20 行的反驳针对的是 null 基线。 | 源码核对。实际耗时影响需设备验证。 |
| `NotificationOperations.kt:111`，对比 `reference.md:68`、`coordinator.py:57-58`、`file_jobs.py:136`、`ShellProtocol.kt:156` | 通知清理未确认时，done 被改成 exit 1；其它所有清理失败或不确定都是 exit 3。调用者按总规则会误判。`notifications.md:33-35` 只写了"nonzero"。 | 源码核对。属于契约不一致，已部分文档化。 |
| `ActivityLaunch.kt:254-256` | VD 上 open 的 result 为 done、exit 为 1。result 和 exit 两套编码相互矛盾，reference 中"done 通常 0"需要例外记忆。 | 源码核对。属于设计不一致。 |
| `T/research/route-review-3f45af6/external-disposition.md` 第 25 行对比第 107 行，以及第 186 行 | 权威处置表对有限 C review 的状态自相矛盾，第 186 行还保留着"Contacts review 正进行"的过期陈述。 | 原件核对。属于文档缺陷。 |

## 6. 需要用户裁决的事项（总方向级）

### U1 首版范围

| 候选 | 能力 | 调用者负担 | 维护责任 | 未知 |
| --- | --- | --- | --- | --- |
| A 实施方：全保留＋新增 4 机制 | 最强 | 最高（71 命令，68 选项，规则句 594 后还会增加） | 最高。+12 隐藏成员，listener 跨命令寿命。 | C 的接入、R8、变换；per-lease 隐私 |
| B 外评：瘦身（删 diff、借贴、通知写、zoom） | 损失后台屏并行下的 Unicode 插入和通知直达 | 低 | 低 | 需要新增 doctor 等，部分抵消 |
| **E 折中（推荐）**：能力保留，入口合并，状态收紧（§2.1 Q4） | 失去同名种类保护、zoom 同帧、持久 diff，以及 C 能覆盖的少数系统窗口的自动定向 | 中低（约 66 命令，规则句约少 60–80） | 中。不新增隐藏面；若换 cache，再少 7 个句柄。 | IME 公开路径是否够用；同帧是否有消费者 |

推荐 E。理由是它最符合用户的"干净、能力强、高效"：保留不可替代的能力，删除只是重复编码或额外状态的东西。

### U2 新增隐藏 API 的门槛

建议作为总原则：新增隐藏 API 或长驻注册之前，必须证明不存在公开等价物，或者证明公开等价物有实测缺口。这一条直接决定题 2C 和 E9/cache 的去向。

### U3 提交策略

| 候选 | 说明 |
| --- | --- |
| a | 在 40 个提交上追加修正（实施方）。保留 churn。 |
| b | 先回退式重排（外评）。有静默合并风险。 |
| **c（推荐）** | 先前向实现到最终树并验收，再从最终树重新切片。 |

## 7. 会改变取舍的验证问题

- **V1（题 2）**：在 API34 和 API37 上，用 `AccessibilityWindowInfo` 的 IME 窗口区域加上 `DEFAULT_INPUT_METHOD` 对应的 UID 做定向，点击软键盘键能不能成功、会不会误拒普通 App 区域。成功，就选公开 E，不加 WindowInfosListener；失败或有遗漏窗口类别，就在 A（不定向）和 C 之间按保护强度选。
- **V2（E9）**：换一种页面（原生长列表）、在真机上比较 cache 和 E9 的完整度与 collect 时间。完整、量级相同，就换 cache，删 7 个句柄和手写回调；cache 出现缺失或明显变慢，就保留 E9。
- **V3（zoom）**：有没有任务需要"同一帧"的细节，例如瞬时 toast 或动画中的文字。没有，就用 `screenshot --region` 替代；有，就保留原生源，但把只读源和输入资格拆开。
- **V4（会话）**：真实的 agent 编排中，是否存在多个调用者共用同一个会话名。不存在，选 1B；存在而且无法改约定，才保留种类（1D）。
- **V5（listener）**：每命令注册和注销的实际耗时。在毫秒级，就保持每命令注册；在数百毫秒，再考虑每 lease 持有。
- **V6（借贴）**：后台屏任务进行中，真人在主屏复制，借贴恢复时会不会覆盖真人的新内容。会，就加上"恢复前比对当前剪贴板是否仍是借出值"，不需要 CAS 这类原子保证；不会，就维持现状。
- **V7（settle）**：在 Chrome 长页上，partial 基线下每个动作的实际 settle 耗时。接近窗口上限，就说明 §5 第 2 条确实影响效率，应当修。

## 8. 未核实与局限

- 计数都是静态正则。CLI 叶命令和 reference 表相差 1。"规则句"只是代理指标。"隐藏成员"按每文件不重复名计，跨类同名（如 `asInterface`）会合并。
- 实施方方案和 E 方案的增减量是推断，没有实现。外评方案的增减来自拆分分支实测，但拆分分支不等于外评的目标态（后台屏是整块拆掉的）。
- D16 的 windowId 稳定性基于对上游语义的记忆，本轮没有重新取 AOSP 源码。
- E 方案的 IME 公开路径、cache 在真机上的表现、借贴在并行场景下的表现，都需要设备验证。
- 没有逐条重算 Chrome 和 Contacts 新对照的成本，只复用了实施方数字和其 review 的边界。
- 五个专题的细节（通知、输入、节点、交付、会话）以其它子 agent 的结论为准；本报告只做跨专题判断。
