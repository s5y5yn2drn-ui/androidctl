# 专题 42：已确认范围的逐条落实、删减与契约一致性

核查对象：产品 `4235af9`（相对旧候选 `3f45af6` 新增 28 个提交，基线 `7d5be70`），独立 docs `9b8c5bd`（材料包内副本）。
需求来源：用户授权原话（材料包 `context/u29-user-messages.md`）、第三轮评审（下称"R3"）、更早的定稿（下称"旧定稿"）中未被 R3 修订的部分。
逐条符合性表（223 行）在工作目录：`/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents7/42/conformance-table.md`。本报告只给汇总和所有非"落实"的行。
自写脚本与中间产物都在同一工作目录：`dump_cli.py`（导出三版命令面）、`cmp_cli.py`（比较）、`cli-{base,old,new}.json`、`cli-compare.txt`、`wire-*.json`（三版契约样本）、`codes*-*.txt`（错误码集合）。

文中"材料包"指 `pkg7/androidctl-external-candidate-review-20261120T174713Z-4235af9`，"阶段目录"指材料包内 `ultragoal/2026-11-11-next-improve/stages/41-route-rebuild/`。

---

## 1. 结论摘要

**整体状态：** R3 已决定的事项和已放行的默认做法，在源码里基本逐条落地；没有一条被漏做。偏差集中在三处：一处越过了"只新增一个隐藏接口例外"的边界且没交给用户；三处指引或内部文档与源码矛盾；若干验收只做到一部分，实施方也如实写了。

**逐条判定总数（223 行）：** 落实 190，部分落实 20，未落实 0，做法不同 6，无法从源码判定 7。【源码＋重算，计数脚本见工作目录】需要说明：很多"落实"只核到源码这一层；设备证据列里我亲自打开原件核过的只有两处（通知跨 App 搬迁、监听器故障组），其余是实施方给的链接。

**最重要的发现：**

1. **新增了 7 个隐藏接口反射成员，没有报给用户。** 为了在让旧会话失效之前判断"平台是否支持后台屏上的 Activity"，实施方新增了 `ActivityTaskManager.supportsMultiWindow`，以及一整条"以外部身份取得 Settings 内容提供者、调用、再释放"的链（`IActivityManager`、`Stub.asInterface`、`getContentProviderExternal`、`removeContentProviderExternalAsUser`、`ContentProviderHolder.provider`、`IContentProvider.call`）。【源码 `HiddenApi.kt` 本轮 diff；`WorkDisplaySettings.kt:1-59`；`WorkDisplay.kt:49-56`】旧定稿写明窗口信息监听器是"不新增 Android 内部接口"的唯一例外，R3 沿用这条线，并把"新增两个隐藏接口"列为方案缺陷 P2。实施方把这件事记成"U29 实现判断：属于现有准入合同的缺陷修复"（`decisions.md:54`），交付稿的四道待决题里没有它。
2. **三处契约矛盾。**（a）首页指引 `SKILL.md:193-195` 让调用者在写通知前"保留完整的 key/postTime 和 creator"，而命令已只收一个选择串，通知页明写"postTime 不是写条件"；（b）独立 docs `reference/connection.md:167` 仍写启动"平台成功后先保 done/exit1"，源码和公开指引都是"UNKNOWN、退出 3"；（c）`SKILL.md:184-185` 仍有"select then commit"的旧说法。（见 §4）
3. **R3 §9.3 要报给用户的六项结果，交付稿回答了四项半。** 效率数字、跨 App 通知搬迁、系统进程重启、历史形态都回答了，而且答得具体。没回答的是：锁屏下 `wake` 是否有效（明写未实跑）；Android 15 以上通知引用是否累积（只写在阶段内部计划里，交付稿完全没提）。
4. **几项验收只做到一部分，实施方如实写明。** 包括：注销抛错之后"观察、shell、到期都正常"没有在同一组里证完；四个效率对照臂都没有用到默认 `find`；"把等待限定在页面正文内"产品做不到，指引只能写成"选更具体的条件"。
5. **本轮新增的对外概念大多有出处。** 没有明确出处、属于实施方自定的有：`screenshot --image ID`；默认保存文件从 `.txt` 行格式改成后缀仍为 `.json` 的逐行 JSON 记录；保存文件封套带设备、会话、来源等字段，全文回退原因分 10 种（R3 §7 要求的是"不新增字段和多级原因"）；同种重开会话时新给的 `--idle` 被静默忽略；`snapshot`/`find` 的保存失败算作主操作失败。
6. **删除清单基本完整**，但漏了几项可见的形状变化：通知写命令的入参从"KEY 加 4–6 个身份选项"变成一个选择串；默认保存文件格式和后缀变化；`--raw` 改为稀疏形状；`find` 默认只显示 20 组；基线到最终版去掉了隐式会话；`wake` 不再能附带截图和观察。它们都有授权，或者能从已授权事项推出（见 §3.3）。

**对待决题的看法（只涉及本专题的部分）：** 四道题与 R3 §3.1 四项一一对应，没有越界提问。第 1 题，R3 原本只给了"维持现状"和"加一个查询接口"两个选项；实施方用 Feeder 经代理打开 Chrome 的原件说明查询接口也挡不住，于是扩成四个选项。这样扩有依据，但推荐的 D 改变了"派发完成"的含义，属于新的产品合同，用户应当知道这一点再选。第 4 题从两种形态扩成四种，属于合理细化。

---

## 2. 确定缺陷

按严重度排序。全部来自源码阅读或文档对读，不涉及设备。

### D1（中）越过"只有一个隐藏接口例外"的边界，没有提交用户

- **触发条件：** 每次 `session open --display virtual` 或接管到后台屏，都会在让旧会话失效之前做这项预检。
- **位置：** `HiddenApi.kt` 本轮新增 7 个 `by lazy` 成员（总数从 65 个变成 70 个，同时删了 `disconnectConnection`、`rootTasks`）；`WorkDisplaySettings.kt:8-59`；`WorkDisplay.kt:49-56`。【源码】
- **实际影响：** 多了一条依赖 `IActivityManager`、`ContentProviderHolder`、`IContentProvider.call(AttributionSource,…)` 签名的隐藏路径。签名一旦在新版本或厂商 ROM 上变化，后台屏就开不了（预检抛错），而且这条路径只在模拟器上验证过。维护上，`IContentProvider.call` 的参数表在过去的大版本里改过（加入 `AttributionSource`，凭记忆，未对 AOSP 源码核对），以后还可能再变。
- **为什么算缺陷而不只是实施细节：** 旧定稿 §2.5 明文写"这是'不新增 Android 内部接口'的唯一例外，例外只限这一项"；R3 把"推荐方案新增两个隐藏接口"列为方案缺陷 P2，并要求只加一个查询接口时也要交用户定。按同样的标准，这 7 个成员应当进待决题。
- **最小修正方向：** 交给用户在下面几种做法里选：（a）保留现状；（b）只用公开的 `FEATURE_ACTIVITIES_ON_SECONDARY_DISPLAYS` 做预检，平台另有限制时在建屏阶段如实报"不支持"，代价是那种设备上旧会话会先失效；（c）只保留 `supportsMultiWindow` 一个成员，去掉内容提供者链，代价是开了"强制可调整大小"开关的设备会被误拒；（d）其它读取设置的方式，可行性要另核。
- **确定程度：** 是否新增隐藏接口，源码确定。这条预检有没有必要，实施方给了一个 API34 上 `force_resizable` 被强制打开的原件（`12-sessions/03-authority/preflight-fix-runtime.md`），我没有核。

### D2（中低）首页指引仍要求保留通知的 key/postTime/creator

- **位置：** `skills/androidctl/SKILL.md:193-195`："Before an explicit write, read fresh get facts and preserve the complete key/postTime and required action/creator selection."【源码】
- **另一端：** `skills/androidctl/notifications.md:76-83`："Copy the complete `selection` string……postTime remains a read fact and is not a write condition."；命令 `notifications action|open|dismiss|reply` 只收 `SELECTION`。【实跑】`androidctl notifications action k --post-time 1` 退出 2。
- **影响：** 调用者先读首页。照字面去找 `--post-time`/`--creator`，会得到一次参数错误（不发送，没有副作用），之后要再读通知页。代价是多一次调用。
- **修法：** 把这句改成"复制 `get` 返回的完整 selection 串"。
- **确定程度：** 确定。

### D3（中低）独立 docs 仍写启动是"先 done、退出 1"

- **位置：** docs `reference/connection.md:167`："平台成功后先保done/exit1，只有后读确认唯一匹配可见leaf在目标屏才移除本次临时非零。"【原件，材料包 `source/androidctl/docs`】
- **另一端：** `ActivityLaunch.kt:229-230`（先记 UNKNOWN、`landing.state=pending`）、`:256`、`:298-311`（观察后确认才改 DONE）；`skills/androidctl/actions.md:365-375`（"first records UNKNOWN/exit3……leaves UNKNOWN/exit3 and stops a batch"）。【源码】
- **影响：** 这份 docs 是给维护者看的契约参考，说的正是 R3 §2.2 第 9 条要删掉的"完成但退出 1"。实施方的累计 review 报告说两组旧文稿遗漏已经关闭，这一处没被发现。
- **修法：** 按源码改写这一句。
- **确定程度：** 确定。

### D4（低）首页残留 `editor commit` 时代的说法

- **位置：** `SKILL.md:184-185`："Current-editor commands capture anew each time; select then commit is not a transaction."
- **另一端：** `input.md:48-50`："a preceding select and a following type are separate steps"。这里的"commit"指已经删除的 `editor commit`，现在的写入命令是 `type --via editor`。
- **影响：** 只是措辞，调用者可能去找 `commit`。
- **确定程度：** 确定。

### D5（低，交付不完整）R3 §9.3 第 5 项没进交付稿

- 交付稿里没有"Android 15 以上通知引用是否累积"。实际结论只写在 `15-notifications/plan.md:81,87` 和 `15-notifications/review.md:38`：API35 上堆导出被系统拒绝、探针以 139 退出、无法计量。【原件，我读了这两份文本】R3 写的是"若有办法测到"，测不到本身就是要报的结果。

---

## 3. 设计与简化

### 3.1 删除清单的完整性（任务 3）

用三套主机环境导出完整命令树逐项比较（【实跑】`dump_cli.py`、`cmp_cli.py`，输出 `cli-compare.txt`）：

| 指标 | 基线 | 旧候选 | 最终 |
|---|---:|---:|---:|
| 叶命令 | 47 | 70 | 66 |
| 长选项（去重） | 40 | 72 | 70 |

**旧候选到最终，已知清单核对：** 5 个命令（`display create`、`display select`、`editor commit`、`editor select`、`zoom`，另有两个父组 `display`、`editor`）与 6 个选项（`--creator`、`--creator-uid`、`--diff`、`--post-time`、`--title`、`--title-null`）全部确认消失。没有发现清单外被删的命令或选项，也没有任何共享选项的默认值、必填性或取值集合发生变化（脚本对每个共同选项比较了 required、default、choices、nargs、is_flag、type 六项）。

**清单外、行为变窄或形状改变的项（旧候选到最终）：**

| 变化 | 出处判断 |
|---|---|
| 通知四个写命令的入参：从"KEY 加 `--post-time`/`--creator`/`--creator-uid`/`--title`/`--title-null`"变成一个 `SELECTION` 位置参数。现在只拿 `list` 给的 key 不能直接写，必须先 `get` | 有授权（旧定稿 §2.7 第 3 条） |
| RPC 方法删除 `zoom`、`display.create`、`editor.commit`、`editor.select`，新增 `select`（`RpcMethods.kt` diff） | 有授权 |
| 请求字段 `expectedDisplayKind` 删除；契约里删除 `displayRouting`、`editorResults`、`deviceUiMethods`、`targetMethods`、`expectedKinds` | 有授权 |
| 输出字段删除：启动落点的交叉校验字段（`observedRoots`、`matchingRootCount`、`association`、`childTask`、`position`、`rootTaskId`、`topIntentComponent(Source)`、`activitySource`、`nameSource`）；缩放字段 `sourceImageId`、`producedMs`；通知旧版的 `creator` 对象与 `afterCount`/`beforeCount` 等 | 有授权（单一任务来源、删缩放、选择串） |
| 结果原因文字有约 10 条删除、约 27 条新增（契约样本对比） | 随上面各项 |
| 默认保存文件：从与打印同形的 `.txt` 行格式，变成后缀为 `.json`、内容是逐行 JSON 记录的视图文件；不再是"打印即文件" | 实施方自行决定，值得商榷（低）：R3 只把"一行一条记录"列为可选；`.json` 后缀装的不是一个 JSON 文档，用 `json.load` 读会失败 |
| `--raw` 保存的形状：从 `{windows, tree, image, artifacts}` 的完整加工树，变成设备收到的稀疏对象 | 有授权（R3 §3.2 末行） |
| `find` 默认最多显示 20 组，`--all` 才显示全部 | 有授权（旧定稿 §2.8 第 10 条） |
| 同种重开会话时，新给的 `--idle` 被静默忽略（回执原因"Target reused; idle configuration unchanged"） | 实施方自行决定且合理，最好在帮助里写明（`session open --help` 只说 preserves its idle configuration，已写） |
| `snapshot`/`find` 保存失败时结果变成 failed、退出 1，即使正文已经打印 | 实施方自行决定，值得商榷。R3 §2.2 第 3 条说"自动的历史保存……不算要求的附件"；实施方把 snapshot 的保存文件当作主交付，公开表写明了（`reference.md:73-74`）。真实任务里出现过一次：目录权限不合格导致保存失败、退出 1，正文却可读 |
| 有缺口但没有文字的事件不再打印（旧定稿 §2.8 第 5 条是"有文字或有缺口时打印"） | 实施方自行决定且合理（R3 §9.2 的措辞只写了"有文字"），影响小 |
| `wake` 在有待完成清理时也被拦 | 实施方自行决定且合理（R3 §4.1 留为未定） |

**基线到最终，变窄的项：** 没有任何基线命令被删。变窄的有：没有会话时不再隐式取得主屏会话，所有设备命令都要先 `session open --display KIND`（有授权）；`session open` 不带 `--display` 成为参数错误（有授权）；`wake` 失去 `--screenshot`/`--settle`/`--image-out`/`--print`（旧候选已删；有授权：`wake` 不连 UiAutomation 就不可能附带观察）；`--diff` 自动基线变成必须自己保管文件路径（有授权）。

**结论：** 没有发现疑似无意的能力退化。清单外需要用户知道的是保存文件格式和后缀的变化，以及 snapshot 保存失败的判定。

### 3.2 本轮新增了什么（任务 6）

授权的主旨是删除、合并、简化。本轮新增的对外可见项如下（【实跑】命令面比较、【重算】契约键与错误码集合比较、【源码】）：

| 类别 | 新增项 | 出处 |
|---|---|---|
| 命令 | `select` | R3 §2.5 |
| 选项 | `session open --display`、`--replace`；`session takeover --display` | 旧定稿 §2.1 |
| 选项 | `open --multiple-task` | 旧定稿 §2.3 |
| 选项 | `action --action-name` | 旧定稿 §2.8 第 9 条（原文写 `--action NAME`） |
| 选项 | `--diff-from PATH`（31 个命令） | 旧定稿 §2.10 |
| 选项 | `screenshot [REF]`、`--region` | 旧定稿 §2.11、R3 §3.2 第 9 题 |
| 选项 | `screenshot --image ID`（要求区域对应某张完整坐标图） | **没有出处**，实施方自加。小，但多一个选项和一条拒绝路径 |
| 错误码 | `sessionRequired`、`collectionProtocol`、`nodeLimit`（原因码）、`inputPredictionUnavailable`、`multipleTaskUnsupported`、`taskOnOtherDisplay`（通知）、`notificationCancel`、`actionName` | 均有出处 |
| 错误码 | `windowInputCleanup`（窗口信息监听器注销失败：不发送并退出 3） | 推导出处：旧定稿要求每条命令注册注销，R3 规定清理未确认为 3 |
| 状态值 | `landing.state = pending/observed/unknown`；`preflight.state = clear/conflict`；树 `notApplicable`；`basis = windowAbsent`；`proof = absolute/relative`；附件状态 `not requested` | 均有出处或属于实现所需 |
| 输出字段 | `selection` 串；`landing.matchingTasks/visibleTasks(+Omitted)`；`preflight.conflicts`；`beforeSelection/afterSelection`；`coordinateImageId`、`mappingKnown`；`status.diagnostics.notificationCleanup`；`status.session.target`；`localCloseError`；`cleanupError.code=uiCleanup` | 均有出处，或是该功能必需的事实 |
| 输出记录 | 正文里的 `edge-windows`、`events` 摘要记录；结果行 `change=…` | 旧定稿 §2.8、§2.13 |
| 文件格式 | 视图文件（头、记录、尾，带格式版本）；10 种全文回退原因（`fileMissing`、`fileUnreadable`、`fileInvalid`、`projectionUnavailable`、`treeUnavailable`、`partial`、`readingTruncated`、`sourceUnavailable`、`contextChanged`、`diffLonger`） | 文件格式有出处；**封套字段与原因的数量超出 R3 §7"不新增字段和多级原因"的简化要求**，更接近旧定稿 §2.10 的写法 |
| 契约样本组 | `wakeResults`、`sessionResults`、`sessionManagementResults`、`textResults`、`nodeTargets`、`nodeEncoding`、`sessionKinds` | 随各功能 |
| 隐藏接口调用点 | 窗口信息监听器（编译桩 `WindowInfosListener`、`InputWindowHandle`，加保留规则） | 旧定稿 §2.5 唯一例外 |
| 隐藏接口调用点 | `supportsMultiWindow` 与内容提供者链共 7 个成员 | **没有出处**（见 D1） |
| 隐藏接口调用点 | `ITrustManager.isDeviceLocked(user, DEVICE_ID_DEFAULT)` 替换旧的按显示编号调用 | F15 修复，接口不变 |

**判断：** 新增项绝大多数直接对应已决定事项，没有出现"为了完备再造一层"的大块机制。需要质疑的只有三处：隐藏接口链（D1）；视图文件封套与回退原因的粒度；`screenshot --image`。后两项是维护成本问题，不影响正确性。

### 3.3 评审方前几轮的建议，现在看哪些代价偏高

- **R3 §3.1 第 2 项"等待范围限定在页面正文内"**：产品没有"正文子树等待"这种能力。指引只能改写成"选正文里特有的角色和名字"（`SKILL.md:167-170`）。R3 的写法比产品实际能做到的多，实施方的处理是对的。
- **R3 §7"保存文件封套不新增字段和多级原因"** 与旧定稿 §2.10"比较前核对设备、会话、worker、连接、目标、根和视图选项，不一致时写明原因"本身就有张力。实施方实际按旧定稿做了，所以多出设备、会话、来源等字段和 10 种原因。现在看，R3 那条简化如果严格执行，"拿别的会话的文件来比"就只能靠视图选项去发现，不够；实施方的选择更稳。维护面可以收缩：把 10 种原因合成"文件问题、来源不可比、本次不完整、差异更长"四类，细节放进 detail。
- **旧定稿"标准动作的数字编号默认不出现"**：实施方保留了无专用命令的标准动作编号，例如 `{"id":4,"name":"select"}`，理由是 `action --action-id` 要用（`SKILL.md:151-153`）。现在已有 `--action-name`，这些编号其实可以按名字调用。这与第 3 题（剩余动作体量）是同一件事，不另提。

---

## 4. 契约矛盾（任务 4）与证据审计

### 4.1 契约矛盾一览

| # | 主题 | 一端（原文与位置） | 另一端（原文与位置） | 判断 |
|---|---|---|---|---|
| K1 | 通知写操作的身份 | `SKILL.md:193-195` "preserve the complete key/postTime and required action/creator selection" | `notifications.md:79-83` "postTime remains a read fact and is not a write condition"；CLI 只收 `SELECTION`【实跑】 | 首页过时（D2） |
| K2 | 启动退出码 | docs `reference/connection.md:167` "平台成功后先保done/exit1" | `ActivityLaunch.kt:229-256`；`actions.md:365-375` "first records UNKNOWN/exit3" | docs 过时（D3） |
| K3 | 输入法写入命令名 | `SKILL.md:184-185` "select then commit is not a transaction" | `input.md:48-50` "a preceding select and a following type are separate steps"；命令面无 `commit` | 首页措辞过时（D4） |
| K4 | 正文范围的等待 | R3 §3.1 第 2 项"等待范围限定在页面正文内" | `SKILL.md:168-169` "there is no body-subtree wait option" | 需求超出产品能力，指引处理正确 |
| K5 | 自动保存是否影响退出码 | R3 §2.2 第 3 条"自动的历史保存和配额回收警告不算'要求的附件'" | `reference.md:73-74` "`snapshot`/`find` require their saved observation……missing that main delivery is failed/1"；`ui_client.py:1446-1455` | 不是文档之间的矛盾；实施方把 snapshot 的保存文件定义为主交付。公开表与源码一致，与 R3 的字面有张力 |
| K6 | 事件缺口是否打印 | 旧定稿 §2.8 第 5 条"事件只在带文字或有缺口时打印" | `observations.md:471-472` "Without such text, no event section is printed, even for a gap"；`ui_view.py:424-444` | 偏离旧定稿，文档与源码一致 |

**以下我核过，没有矛盾：** 退出码表（`reference.md:58-91` 与 `protocol.py:53-69`、`ui_client.py:1461-1492`、`AutomationCleanup.kt:58-66`、`ui_batch.py:262-288` 一致）；五种结果状态名（`protocol.py:47` 的字面量与 `SKILL.md:205-211`、`reference.md:60-66` 一致）；会话与目标屏规则（`SKILL.md:29-35`、`displays.md:16-50`、`session open --help`、`Sessions.kt:76-157`、docs `connection.md:161-165` 一致）；通知选择规则（`notifications.md` 与 `NotificationActions.kt`、`NotificationOperations.kt` 一致）；输入通道（`type --help`、`input.md`、`actions.md:414-420` 一致）；保存与比较（`SKILL.md:70-98`、`observations.md:97-145`、`view_file.py`、`ui_diff.py` 一致）。契约样本 `contract/wire-contract.json` 与源码的一致性由两端的契约测试负责，我没有运行 Gradle，只核了键和码的增删与源码相符。

### 4.2 实施方关键声称的核对

| 声称 | 核对结果 |
|---|---|
| "未发现已批准能力遗漏、未授权能力删除"（`final-scope-review.md`） | **部分成立。** 能力删除方面成立；但"未授权新增"没有被检查，D1 的 7 个隐藏成员没有被这份 review 指出 |
| 跨 App 通知把主屏 Chrome 任务搬到后台屏、结束后没回主屏 | **成立。**【原件】`15-notifications/runtime/api34-notifications/commands/formal/073…/stdout.bin` 里 `Task{6e0ba66 #266 … com.chrome.beta}` 位于 `Display #0` 段，`075…/stdout.bin` 里同一对象位于 `Display #4` 段；`raw/formal/074-feeder-cross-app-one-send.trace.json` 的结果是 `UNKNOWN`，`visibleTasks` 只有 `com.chrome.beta` uid 10148。API34 模拟器，1 次 |
| 注销故障下"主回执、退出码不变，后续可用" | **部分成立，实施方自己也这样写。**【原件】`15-notifications/runtime/listener-fault/runtime/api34/review.md` 明写"B/live1 下的完整观察、shell、files 可用性没有通过"：B 组的 snapshot 因停在 AndroidCtl 自身受保护页失败，shell/files 没有执行；C 组只在已确认未登记后测到 |
| 第 0 阶段效率对照数字 | **数字齐全，结论如实。** 第 0 阶段 GitHub 新臂输入 1,388,555，旧臂 585,387；最终 Maps 新臂 773,077，旧臂 381,588；四对里新呈现有两对更贵，实施方没有宣称净收益（`efficiency/result.md`、交付稿效率节）。我没有重算 usage 原件 |
| "四臂都没有默认 find 样本" | 成立（`efficiency/result.md:57,69`），所以"默认 find 搜全部已采节点"的效率收益没有实测 |
| 采集协议快速失败"API34 薄探针六案 exit0，MuMu exit139 保留" | 未核原件；源码有对应单元测试（`CollectionRepliesTest`、`CollectionPublicationTest`） |
| 系统服务死亡后 worker 退出 | 源码成立（`WorkerService.kt:25`）；**没有单元测试**；设备上只证明旧 worker 消失、新 worker 可观察，没有旧进程退出码 |

---

## 5. R3 §9.3 六项结果是否报了（任务 2）

| 项 | 交付稿是否报 | 是否回答了问题 |
|---|---|---|
| 1 效率对照、重新观察次数、假就绪次数、输出体量、动作占比 | 报了，两张表 | **回答了。** 第 0 阶段：动作后新增独立界面读取 1/1/1/11 次（含缓存投影），严格假就绪 0；new 默认正文里动作字节约占 39%–46%。最终：新采读取 1/1/3/3 次，"首行显示已变化但业务仍在加载"1/1/1/2 次；正文/动作字节 Maps 旧 54,815/32,755（59.76%）、新 102,177/43,517（42.59%），GitHub 新 135,318/42,541（31.44%）、旧 33,528/16,826（50.18%）。实施方写明两阶段的读取口径不同，不能直接相减 |
| 2 打开网址的通知是否搬动主屏任务 | 报了 | **回答了，答案是会搬。** 同一 Chrome 任务从主屏搬到后台屏，会话结束后没有回主屏（我核了原件，见 §4.2） |
| 3 `wake` 在熄屏、锁屏下 | 报了一半 | 熄屏唤醒成立；锁屏（真锁）分支明写未实跑 |
| 4 系统进程重启后的完整路径 | 报了 | 回答了；旧 worker 消失、新 worker 可观察，没有取得旧进程的退出码 |
| 5 Android 15 以上通知引用是否累积 | **没报** | 结论（堆导出被拒、无法计量）只在阶段内部计划和 review 里（D5） |
| 6 历史形态 | 报了，第 4 题 | 回答了，扩成 A/B/C/D |

---

## 6. 实施方自行做出的判断（任务 5）

`decisions.md` 的"Judgment calls"一节按倒序排列。U29 之后的条目是 J310、J311、J312，另外第 54 行有一条"U29 实现判断"。J309 及以前属于上一轮研究期。我还从源码和指引里找到几处没有登记的自定做法。

| 判断 | 内容 | 是否越权 | 是否应告诉用户 |
|---|---|---|---|
| U29 实现判断（`decisions.md:54`） | 用"一次外部取得、调用、释放"的内容提供者链替换普通解析器，读取 `force_resizable_activities`，算作"现有准入合同的缺陷修复" | **越过了隐藏接口的边界**（D1）。修的问题是真的（建屏前提要在让旧会话失效前查清），但修法引入了 7 个隐藏成员 | 应当，并给出替代做法 |
| J310 | 窗口缺席保留 screen-change 的正证据；同一事实同编码；有待完成清理时可以显式开一个不碰 UiAutomation 的主屏会话来跑 shell/文件，真实界面操作仍被封锁 | 不越权，是 R3 §3.2 第 11 题的接线解释 | 最后一点是可见行为，指引已写（`displays.md:50`），可以不单独报 |
| J311 | 不带 REF 的 `type --via editor` 保留"不先采树"的独立准入；显式观察请求必须交付 | 不越权，依据旧定稿"行为不变" | 不必 |
| J312 | 四臂结束后把首页两个滚动例子的 `down` 改成 `DIRECTION` | 不越权 | 已告诉用户：交付稿写明最后的文字修订没有实测 |
| 未登记：snapshot/find 保存失败判为失败、退出 1 | 见 §3.1 | 有张力（K5） | 应当，因为真实任务里出现过 |
| 未登记：有缺口但没有文字的事件不打印 | 见 K6 | 轻微偏离 | 可不报 |
| 未登记：保存文件改为 `.json` 后缀的逐行记录 | 见 §3.1 | 在可选建议范围内，后缀是自定的 | 应当顺带说明 |
| 未登记：同种重开会话忽略新 `--idle` | 见 §3.1 | 不越权 | 可不报 |
| 未登记：有待完成清理时 `wake` 也被拦 | R3 §4.1 留为未定 | 不越权 | 可不报 |
| 未登记：`screenshot --image ID` | 见 §3.2 | 小幅超出需求 | 可不报 |
| 未登记：启动冲突的报错文字不区分"别屏可见任务"和"只有后台任务"，只在 `data.conflicts[]` 里区分 | 旧定稿 §2.3 第 1 条要求报错区分 | 部分落实 | 可不报，属于实施细节 |

---

## 7. 对相关待决题的评价

- **第 1 题（后台屏通知落点）**：R3 只给了"维持"和"加 `queryIntentComponents`"两个选项。实施方的原件证明，Feeder 先由自家代理接收，之后才打开 Chrome，所以查询初始接收者也预见不到 Chrome。【原件，我核了搬迁那一段】因此扩成四个选项有依据。推荐的 D 保留"创建者别屏有任务就拒绝"，同时让"派发成功"不再因为落点核对失败而变成 UNKNOWN 并停下 batch。这改变了 R3 §3.1 第 1 项写定的"没有就是未知、退出 3、停 batch"，是新的结果合同，不是实现细节。用户选 D 时要知道：以后"done"只表示派发，不表示界面落在了目标屏。
- **第 2、3 题**：没有越出 §3.1 的范围。
- **第 4 题**：比 R3 多了 B 和 D，属于合理细化。
- **漏掉的一题**：D1 的隐藏接口预检应当作为第五题，或者至少作为一个"需要知情"的条目放进交付稿。

---

## 8. 需要用户裁决的新取舍

1. **建屏预检用的 7 个隐藏成员留不留**（D1）。候选：（a）保留，接受新增的接口面，只在模拟器验证过；（b）只用公开的特性检查，代价是少数设备上"支持与否"要等到建屏时才知道，那时旧会话已经失效；（c）只留 `supportsMultiWindow` 一个成员，开了"强制可调整大小"开关的设备会被误拒。**我的推荐是 b 或 c**：这个产品的立场是"能力够用、维护得动"；为了"强制可调整大小"这种开发者选项走一整条内容提供者链，不划算。剩余未知：有多少真实设备依赖这个开关才能开后台屏，没有数据。
2. **`snapshot`/`find` 保存失败要不要算失败**（K5）。候选：（a）维持，保存文件就是主交付，退出 1；（b）正文已经打印时算完成，保存失败退出 1 但结果字段为 done；（c）保存失败只给警告、退出 0。我推荐 b，与动作命令"结果反映主事实、退出码反映附件"的规则一致。不确定的地方：现在的调用者是否依赖 `snapshotPath` 一定存在。

其余发现（D2–D5、保存文件后缀、`screenshot --image`、回退原因的粒度）由实施方直接修，不需要用户裁决。

---

## 9. 我没能核实的内容

- 没有运行任何设备、模拟器或 Gradle。Android 测试、契约导出和两端验证是否通过，只有实施方的声称。
- 符合性表里设备证据列标"未查"或只有链接的，我没有打开原件。亲自核过的原件只有两处：通知跨 App 搬迁的 073、074、075，以及监听器故障组的 review 文本。
- 主机端测试我没有运行（全量由主评审在跑），测试名是 grep 出来的，没有逐个读测试内容判断它是在验证行为还是在复述实现。
- 符合性表里判为"无法判定"的 7 行：#5 第 0 阶段是否单独向用户汇报过；#31 长网页滚动后与展开后采集完整；#70 裁尾后名字完整性复核；#107 主机先取窗口列表；#129 契约重新导出；#155 建屏四种结局后的 `status`；#179 失败、未发出、未知的行是否原样不压缩。
- 实施方的设备原件来自 MuMu（不能建后台屏；实施方材料把它与 API35 并称，我没有核镜像版本）和 Android Studio API34 模拟器。所有设备结论都不能推广到真机或厂商 ROM。
