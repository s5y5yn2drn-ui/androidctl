# 任务 129：新十题逐次读记录（下半：第 7、8、9 题与通过的第 1、3、10 题，共 10 次）

记号：`EV` = `S/r8/ev/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`，`PKG` = 实施方材料包里的同一目录。"native 第 N 行"指 `native.rollout.jsonl` 的 ordinal（0 起）。"CLI[k]"指该次 `cli-index.json` 里第 k 条命令（1 起），原件在 `evidence/cli/<id>/child.stdout`。运行版本均为 `82f3a82`；`409e853` 那句新指引没有实测过。源码引用版本为 `409e853`。

## 一、结论（先读这里）

1. **第 7 题（主题往返）两次失败，直接原因是指引旧句 "Stop after another UNKNOWN" 被理解成"整题第二次 UNKNOWN 就停"，这一点实施方判断对（高把握）。但根子在产品：三次深色主题切换动作全都实际生效，产品三次都报 UNKNOWN；其中两次，产品在同一回执里已附上新树，开关就在原位置、原 resource id，值已经变过去了。** 原因是切换主题使 Settings 的 Activity 重建，窗口 id 变了（130→132、142→144、153→155），而产品的验证绑定原节点句柄和原窗口 id（`UiEffect.kt:44-49` `sameAddress` 要求 windowId 相同；`UiActions.kt:597-600` `rowCheck` 用原 `checkNode` 读值）。实施方以"模型自述的窗口重建不算已证机制"为由不归因、并判"无产品 Must-fix"，证据其实足够；该句改动也只是让模型少停一次，不减少假 UNKNOWN。
2. **第 7 题重试的第二个 UNKNOWN 是 `open com.android.deskclock`：Clock 一回到前台就弹出自己的通知权限询问，产品的落地确认只认"可见任务顶部 Activity 的包名"（`ActivityTasks.kt:111-116` `activityLanding`），权限页的包名是 permissioncontroller，于是判未知。** 产品手里有任务的 baseActivity（`hasTaskIdentity`，同文件 43-44 行）却没用，也没在 `visibleTasks` 里交给调用者（`visibleTaskFact` 只给顶部 Activity）。这在 API 33+ 首次启动 App 时很常见。
3. **新十题里 13 次退出码 3 中，有 7 次是同一个模式：Chrome 新标签页上 `fill <假搜索框> URL --submit`。** Chrome 把焦点交给真正的地址栏，URL 实际已填进去；但原输入框消失，产品报 fill UNKNOWN、submit 不派发。本批 5 次网页尝试全部命中，04 题另两次也命中，即**所有 7 次网页尝试都命中**。每次多花 1–2 条命令，且在旧的"第二次 UNKNOWN 就停"读法下，一道网页题只要再出现一次 UNKNOWN 就会停。这是下一轮最值得修的条目（按出现频率）。
4. **第 8 题（TfL）是站点对自动化的 Cloudflare 拦截，产品没有读不到 WebView。** 模型自己的快照（首轮 000009、重试 000008）把挑战页正文连同 iframe 里的 "Verifying… / Cloudflare" 都读到了。实施方强调的"像素有正文、full 树只有 WebView 壳"来自约 70 秒后 operator 另拍的 capture 549。那是一个独立现象：WebView 子树为空，却报 `partial=false`。它可能会误导网页题，但不是本题失败的原因（需设备确认机制）。
5. **第 9 题（Gutenberg）维护页属实**：两次的树和截图都是 "Temporarily Down for Maintenance"。属于站点/外部原因，高把握。
6. **第 3 题首轮失败是题目准备缺陷，不是产品或模型的问题**：AOSP Contacts 在空库时界面上确实没有搜索入口（000002/000003 树：只有抽屉、ADD ACCOUNT、IMPORT、Create new contact；抽屉只有 Contacts/Labels/Settings）。重试时加一个不匹配的控制联系人，属于修题，不是放水：验收点一条没少，重试的 PASS 扎实。但首轮应改记为"准备缺陷，不计入产品/指引"，而不是与其它 FAIL 混在一起。
7. **第 1、10 题的通过是扎实的**，最终答复里的每个事实都在快照/截图里找得到。过程中有两个不影响结果的产品行为：倒计时数字所在的节点每秒换 ref，导致点不到（第 1 题 exit 2）；`wait --contains Search` 失败是正确行为，因为 OSM 移动版的搜索框收在 "Main" 菜单里（第 10 题）。
8. **对"新十题 3/10"的读法（只就本人负责的 6 题）**：08、09 两题没有产生任何产品信号；03 首轮是题目准备问题；07 是"产品假 UNKNOWN + 指引停止句"共同造成的。真正能衡量产品的 4 题（01、03、07、10）里 3 题通过，07 的失败可以通过产品改进或指引改句消除。不应把 3/10 读成产品在没见过的任务上的真实水平。
9. **评测环境本身的一个问题**：10 次尝试中有 6 次，模型看到的退出码是 1，而 CLI 的真实退出码是 3 或 2。原因是 PowerShell `-Command` 不传递原生命令的退出码，模型自己的包装也没有写 `exit $LASTEXITCODE`。这不是产品问题，但让"退出码区分 UNKNOWN 与失败"在评测里失效，模型最终答复里也写错了（07 两次、08 首轮、03 重试）。

## 二、最重要的发现（每条可独立读）

### 发现 A（确定缺陷，产品判定偏保守）：配置变更导致 Activity 重建时，check/uncheck 必然报 UNKNOWN，即使回执自带的新树已显示目标值

- **位置**：`android/.../UiActions.kt:572-615` `rowCheck`。它用原节点句柄 `checkNode` 做 `awaitValue`。句柄所在窗口被销毁后，读取抛出 `rootUnavailable`（`FreshNodes.kt:24`），验证结果为 `unobservable`，结论是 UNKNOWN。`UiEffect.kt:44-49` 也以 windowId+sourceId 定位原目标，新窗口里自然找不到（"Original target is not identifiable in the final observation"）。
- **实测**（三次主题切换，全部实际生效）：
  - 07 首轮 CLI[9] `check e241580321`：exit 3。回执里没有新树（`scopeUnknown`，"no App window on the target display yet"）。紧随其后的 recents 回执 000009 显示 `checked="true"`，窗口 id 从 130 变为 132。
  - 07 首轮 CLI[24] `uncheck e241580843 --settle 3s`：exit 3。**同一回执**里的新树（窗口 144，原为 142）位置 `[536,617,648,713]` 相同、`id="switch_widget"` 相同，已是 `checked="false"`。
  - 07 重试 CLI[10] `check e241581262`：exit 3。**同一回执**新树（窗口 155，原为 153）已是 `checked="true"`。
  - 对照：07 重试恢复时改用 `click e241581780` 点同一行，CLI[23] exit 0、done，同样发生了 root 不可用后恢复。同一个物理动作，click 报 done，check 报 UNKNOWN。
- **触发条件**：切换后会触发 Activity 重建的设置，例如深色主题、语言、字体/显示大小、方向，以及会 recreate 的 App 内主题开关。
- **影响**：调用者每次都要多做一次只读核验；碰上旧指引的停止句，整题就停了（07 两次失败的直接起因）。对普通任务也会把 UNKNOWN 的信号稀释掉。
- **建议**（需用户取舍，见第六节问题 1）：至少在回执里给出"替换窗口中同包、同 resource id、同角色、同位置的候选节点及其 checked 值"，结论仍保持 UNKNOWN；更进一步是在条件严格满足时判 done 并标 `reidentified`。
- **代价**：B 方案一般，需要 Kotlin 改动、合同样本和设备测试；C 方案有误判风险，必须限定同包、同标题、同 resource id、同位置且唯一。
- **验证**：设备上做深色主题、字体大小两类开关往返，并做一个反例（新窗口是另一个页面，但有同 id 开关），需设备。

### 发现 B（设计取舍，频率最高）：Chrome 新标签页的 URL 输入固定产生 fill UNKNOWN

- **实测**：08 CLI[4]、08 重试 CLI[4]、09 CLI[4]、09 重试 CLI[4]、10 CLI[4]，回执都是 `focus=done; fill=UNKNOWN, reason="The original filled field is no longer visible"; submit=not sent`。同一回执的新树里，地址栏已聚焦，值正是所填 URL（例：08 CLI[4] 新树 `textbox e241582113 value="https://tfl.gov.uk/plan-a-journey/" ... id="url_bar" focused`）。按 `PKG/efficiency/core-results.md` 第 576、582 行，04 题两次同样如此。
- **机制**（源码推断）：`UiActions.kt:869` 在原节点不可见时直接给出 unobservable；`actions.md:158` 明确 submit "never chooses another field"。Chrome 把假搜索框的焦点移交给 `url_bar`，原节点随即消失。
- **模型反应**：7 次都正确处理，即只读确认后对新 ref 执行一次 `ime`，没有重填。但每次多花 1–2 条命令和约 10 秒。
- **影响**：所有网页任务的第一步都会出现一次 UNKNOWN（exit 3）。在 07 那种"第二次 UNKNOWN 就停"的读法下，网页题的容错余量为零。
- **可选处理**：
  - 指引里把 `open --url URL` 写成打开网址的首选（`actions.md:183` 已有这个能力，但不显眼）；
  - 或者产品识别"焦点移交"：原字段消失，同窗口里唯一的已聚焦可编辑框恰好等于所填值时，fill 报 done 并标出移交到哪个 ref，submit 仍不派发，但回执直接给出"可对 eXXX 执行 ime"。
- **代价**：指引一句话，代价低；产品识别代价中等，而且要守住"不替调用者选字段"的原则。

### 发现 C（设计取舍）：`open` 遇到目标 App 自己的运行时权限弹窗时报 UNKNOWN，回执也不说明弹窗属于目标任务

- **实测**：07 重试 CLI[25] `open com.android.deskclock` 的回执为 exit 3，`launchCode=2 START_TASK_TO_FRONT`，`landing.state=unknown, matchingTaskCount=0, visibleTasks=[{taskId:123, package:"com.android.permissioncontroller", activity:"...GrantPermissionsActivity"}]`，新树是 "Allow Clock to send you notifications?"。按 `core-results.md` 第 600、602 行，operator 在两次准备中 open Clock 也都遇到 UNKNOWN 加权限页。对照：07 首轮经 Recents 点 Clock 卡片（CLI[18] click）到达同一个权限页，结果是 done。
- **源码**：`ActivityTasks.kt:104-133` `activityLanding` 只比较 `topActivityInfo.packageName/uid`；`hasTaskIdentity`（43-44 行，比较 base/real/orig activity）已经存在，但没有用于落地判断；`visibleTaskFact`（136-143 行）不输出 baseActivity。
- **影响**：API 33+ 上首次打开任何需要通知权限的 App 都会这样。模型无法从回执看出 task 123 是 Clock 的任务（taskId 归属需设备确认，源码推断是 Clock 的任务）。
- **建议**（问题 3）：在 `visibleTasks` 里补充 base/real activity；或增加一个落地状态，例如"目标任务在前台但被别的包的 Activity 覆盖"。结论是否升为 done 由用户定。

### 发现 D（证据不足，需设备）：WebView 子树为空时快照仍报完整

- 08 重试由 operator 拍的 capture 549（`EV/runtime/api34/verification/r7-new-08-retry1/evidence/000002-...json`）共 30 个节点，`partial=false`，`missing=[]`。"Web View" FrameLayout（e241582479）和 Web 内容根 `android.view.View`（e241582268）都没有子节点，`childrenIncomplete=false`。约 72 秒前模型自己的 000008 同页完整读到正文和挑战组件。
- 同类空 WebView 的情况还出现在 09 首轮 000006、10 的 000007，但这两次都在页面加载中，产品标了 "loading detected / page may still be transitioning"。549 这次没有任何提示。
- 读法：可能是 Chrome 在一段时间没有无障碍查询后自动关闭了 Web 内容的无障碍树，第一次查询时还是空的。机制未证。
- 影响：模型可能把"页面没有内容"当成事实。本批没有因此失败。
- 建议：可选。产品在"Web 内容宿主节点无子节点且未在加载"时给出一个 warning，指引写"再拍一次"。先在设备上复现。

### 发现 E（证据层面，非产品）：评测包装吞掉了退出码 3/2

- 07 两次、08 首轮、09 两次、03 重试中，模型的 `exec_command` 看到的 `exit_code` 都是 1，而 `cli-index.json` 的 `childExit` 是 3 或 2。用 `$code=$LASTEXITCODE; exit $code` 的几次（01、10、08 重试）看到的则是正确值。
- 影响：评测里"退出码区分 UNKNOWN/失败"的作用失效，模型最终答复里有 4 次写错了退出码。实施方已注意到"不采用模型自称 exit1"，但没有指出原因在评测包装。
- 建议：下一轮评测脚本统一传递 `$LASTEXITCODE`。这不需要改产品。

### 发现 F（可选建议，低优先）：倒计时等实时变化的文字节点每次都换 ref，所以点不到

- 01 CLI[15] `click e241549385`（名称 "29:23"，`+click="Pause"`）exit 2 stale，`timing=ref-repair`，`freshRepair=true`，新树里同一节点已变成 e241549386 "29:17"。ref 签名含名称，修复也匹配不上。模型改点 "Stop" 按钮完成暂停。
- 影响：计时器、媒体进度、实时价格一类元素无法直接点击。频率低，模型通常能找到替代按钮。暂不建议投入，除非出现无替代按钮的真实任务。

## 三、逐题

### 第 7 题：四个界面的深色主题往返（XAPP-05）

**题面验收点**（来源：`PKG/r7-runtime-preparation.md:120`。模型收到的题面在 native 里是加密的，看不到原文，这里依据实施方的题面表和模型复述）：
1. Clock 原页、Files 指定目录三项、Collabora 稳定首页、SystemUI 通知栏，分别取基线图，并读内容、位置、数量；
2. 记录 Dark theme 原来的 checked 和 schedule；
3. 只把 Dark theme 改成相反值，并核对 checked；
4. 四个界面分别取变化后的图；
5. 通过 Recents 核对三个 App 的原页面位置；
6. 再读当前 checked，恢复原值；
7. 四个界面的恢复图，以及 Files 数量、Collabora 未编辑、Clock 对象保持；
8. 最后回到 Home；
9. 按跟随/部分跟随/未跟随/证据不足分类。

约束：不改 schedule，不编辑文档，不创建 Timer，按实际原值恢复。

是否苛刻：步骤多（约 20 步），但每一步都是普通操作，不算苛刻。题目要求不得回答 Clock 的权限询问（不改权限），而 Clock 每次回到前台都会弹这个询问（`POST_NOTIFICATIONS=false` 且 flags 为 USER_SENSITIVE，见 `EV/runtime/api34/commands/r7-new-07-independent-restored-night-and-clock-permission/stdout.bin`）。这使任何"进入 Clock"的步骤都先落在权限页。这是 App 行为加题目约束，不是苛刻，但它直接制造了重试的第二个 UNKNOWN。

**首轮时间线**（`EV/efficiency/r7-new-07`）：
- native 26–59：读 SKILL/actions，分别拍四个基线（000001 Clock、000002 Files、000003 Collabora、000004 通知栏），均 exit 0。
- 66–69：进入 Dark theme 页，读到 OFF，Schedule None。
- 75：**同一个 exec 里先 `check e241580321` 再 `recents`，没有先读 check 的结果**（执行偏差，实施方也记了）。check 为 exit 3，没有新树；recents 回执显示 `checked="true"`。
- 88–143：经 Recents 回到 Collabora（变深）、Files（变深）、Clock（先落在权限询问页，Back 后回 Timer 00），通知栏卡片变深。全部 exit 0。
- 143–148：`open --action android.settings.DARK_THEME_SETTINGS` 重开设置页，读到 ON。
- 150：`uncheck e241580843 --settle 3s` exit 3，同一回执新树已是 OFF。
- 157–164：只读 snapshot 加截图，确认 OFF/None。
- 170（最终答复）：明确写"公开 SKILL 规定 Stop after another UNKNOWN；第二次 UNKNOWN 出现在恢复操作，因此停止"。未做恢复后的四界面核验和 Home。

**重试时间线**（`EV/efficiency/r7-new-07-retry1`）：
- 基线四图：Collabora 的基线已经是深色。这是因为首轮恢复主题后 Collabora 没有跟着变回浅色，属于 App 行为，也正好说明题目为什么要求恢复后再核验。
- native 60：同样在一个 exec 里 `check` 加 `recents`；check 为 exit 3，同一回执新树 ON。
- 70–138：开启后四图、经 Recents 回三个 App（Clock 又落在权限页，Back），均 exit 0。
- 139：恢复改用 `click`，exit 0、done，新树 OFF。
- 144：`open com.android.deskclock` 为 exit 3，落在权限页；随后 snapshot 仍是权限页。
- 152：模型给上层发消息（加密）。
- 158：最终答复，同样以"Stop after another UNKNOWN"为理由，停在权限页。

**原因判断**：
- 主因：**指引问题**（高把握）。两次最终答复都明确引用这句，而当时都已有"可靠的业务成功"（SKILL 第 116 行 "Continue on reliable business success" 本该允许继续）。
- 次因一：**产品判定偏保守**（发现 A、C，中高把握）。4 个 UNKNOWN 里，3 个对应的动作实际已成功，第 4 个（open）也实际落进了目标任务（需设备确认 task 归属）。
- 次因二：**App 行为**（Clock 反复弹出权限询问；Collabora 不随主题恢复）。
- 次要：**执行偏差**（check 后不读结果就顺排 Recents；不过 Recents 不会改变开关状态，实际危害小）。

**实施方处置是否充分**：
- 对直接原因的分析是对的（`PKG/r7-cumulative-validation.md:620-630`），改句范围窄、有依据。
- 不够的地方：
  - (1) 把"窗口重建"当成未证机制，回避了产品层面。证据其实齐全：窗口 id 变化有实测，验证绑定原窗口有源码。
  - (2) 判"无产品 Must-fix，不需要产品自动 fallback"，没有讨论"在回执里提示候选节点"这种低风险方案。
  - (3) 新句 "If a retry is also UNKNOWN, stop retrying that action" 能消除本题的误停，但假 UNKNOWN 仍然存在。下一个更谨慎的模型，或别的停止规则，还会被它绊住。
  - (4) 新句未经实测，不能说 07 已修好。实施方自己也这样写了（第 700 行），这一点是对的。

### 第 8 题：TfL 行程规划

**题面验收点**（`r7-runtime-preparation.md:121`）：
1. 在本题新标签页打开 plan-a-journey；
2. 查询 London Waterloo→Tate Modern，确认所选地点对象；
3. 选择一项步行或无障碍条件，记录原文后提交；
4. 读一条结果的时点、起终点、方式/耗时、换乘摘要，区分"步行不适用换乘"和"页面未提供"；
5. 返回表单，核对地点和条件是保持还是重置；
6. 关闭本题标签页。

约束：不登录、不定位、不购票。从模型最终答复看，题面要求遇到安全验证就停止。第 4 点"区分不适用与未提供"要求较细，但合理。

**两次时间线**：
- 首轮：snapshot → 点 New tab → snapshot → CLI[4] fill URL `--submit` exit 3（发现 B）→ snapshot → `ime` exit 0 → snapshot 显示 "Just a moment..." → `wait --contains "Plan a journey"` 为 done，但命中的是地址栏里的 URL 文本，不是页面（实施方已记）→ snapshot 加截图 000009，树里有 "Verification required"、说明文字和空 iframe → 停止，保留标签页。
- 重试：同样路径，000008 的树完整读到 iframe 内的 "Verifying..."、Cloudflare/Privacy/Help 链接 → 停止。

**原因**：**站点/外部**（高把握）。Cloudflare 对模拟器加自动化流量做了人机验证，两次都是。没有产品原因：模型自己的快照完整读到了正文和挑战组件。

**关于"截图有正文、树只有骨架"**：这只出现在 operator 后来另拍的 549（发现 D），不是本题失败的原因。实施方把它写成"full 采集不等于业务信息充分"（`r7-candidate.md:71`），说法本身成立，但容易被读成"产品读不到 WebView 才失败"，应该澄清。

**对网页题的意味**：有反爬保护的站点在模拟器上随时会阻断，结果对产品没有信息量。网页题应事先在同一环境做一次只读可达性预检，或者准备备用站点。不要为此改产品。

### 第 9 题：Project Gutenberg 读作品

**题面验收点**（`r7-runtime-preparation.md:122`）：
1. 在本题标签页打开 gutenberg.org，搜索 H. G. Wells 的 The Time Machine；
2. 选中匹配作品，在版本页核对标题/作者/语言；
3. 打开在线 HTML，经目录进入第二章，读章首和章尾并概括；
4. 回到同一目录和同一搜索结果；
5. 关闭标签页。

不苛刻。

**两次**：fill 为 exit 3（发现 B）→ `ime` → 维护页。重试还用菜单刷新了一次，仍是维护页 → 关闭自己的标签页，回到 5 个标签页。

**维护页是否属实**：属实。树（`EV/efficiency/r7-new-09/evidence/cli/1795739759813875000-3046.../artifacts/0002-000007-...txt`）和截图（同目录 `0001-000007-...jpg`，本人已查看）都是 "Temporarily Down for Maintenance / Project Gutenberg is temporarily unavailable while we perform maintenance"。重试在 00:39 前后的 000007、000010 也是同样内容。

**原因**：**站点/外部**（高把握）。首轮模型没有看截图（原生图像 0），但树已足够。

**实施方处置**：充分。没有等待、没有换网络、没有外部代读，记录如实。

### 第 1 题：Clock 计时器与通知（通过）

**题面验收点**（`r7-runtime-preparation.md:114`）：
1. 建唯一一个 30:00 Timer，并打上 `<RUN>` 标签；
2. 读两个时刻的剩余值，确认递减；
3. Home 后展开通知，读剩余时间和 Pause/Add 1 min（Add 1 min 只读不点）；
4. 点通知主体回到同一 Timer，Pause 后读两次稳定值；
5. Resume 后读到递减，再 Pause；
6. Delete 后分别核对 Timer 空态和通知缺席。

不苛刻。

**核对**：最终答复里的事实逐项都有依据：
- 000009 为 29:44，000010 为 29:37；
- 000012 的通知 detail 为 "Alerted | 29:33 | … | R7-NEW01-20261127"，含 PAUSE、ADD 1 MIN；
- 000013 为 29:23；
- 000016/000017 为 29:09，`+click="Start"`（已暂停）；
- 000019 为 29:07（已恢复运行）；
- 000020 为 29:00（已暂停）；
- 000021 为 00h 00m 00s；
- CLI[23] `notifications list` 为 partial=false，只有 AndroidCtl 和 android 两条。

**扎实**。

**过程中的产品行为**：CLI[15] 的 stale 见发现 F。产品正确地没有派发，并给了新树，模型改用 Stop 按钮。这次模型在每条命令后都写了 `exit $LASTEXITCODE`，看到的退出码正确。

### 第 3 题：Contacts 七字段未保存草稿（首轮失败，重试通过）

**题面验收点**（`r7-runtime-preparation.md:116`）：
1. 先实际提交 marker 搜索，核对无结果；
2. 新建本地草稿，填七个字段；
3. Home→Recents 后逐项读回七个完整值及保存位置；
4. 正常舍弃；
5. 再提交一次同一搜索，结果为阴性；
6. 回到列表。

约束：不保存，保留原联系人。在空库上，第 1、5 点不可能做到，题目准备有缺陷。

**首轮**：
- CLI[1–3]：空列表上没有搜索入口，抽屉里也没有（000002/000003）。
- 模型两次给上层发消息，上层回复"继续做草稿，缺项保留"（native 51–86）。
- 草稿七个字段全部 `fill` done，`verification="matches"`。
- 中途两次 `back` 本意可能是收起键盘，实际弹出了 "Discard changes?"（CLI[10]）。模型在同一个 exec 里紧接着执行的 `scroll e241552610`（CLI[11]）被产品以 "Ref is absent from the current observation" 拒绝，未派发。**这是产品正确阻止了在对话框背后滚动**。随后模型点 CANCEL 继续。
- Home/Recents 读回七个值，舍弃，结束。

**重试**：operator 先建了一个不匹配、没有电话和邮箱的控制联系人 `R7ControlOnly20261127`，列表上出现 "Search contacts"（000001）。
- 初始搜索 000003 和末次搜索 000030 都是 `No contacts`；
- 000024–000026 读回七个值和 "Saving to Device"；
- 舍弃后列表只剩控制对象（000031）。

**扎实**。CLI[12] 的 stale 与首轮情形相同（back 弹出了舍弃对话框），产品正确拦截。模型在最终答复里把这次写成"退出码 1"，原因见发现 E。

**原因判断**：首轮是**题面/准备缺陷**（高把握），不是产品、指引或模型的问题。模型的处理是正确的：发现缺口、上报、继续做安全部分、如实报缺项。

**修题还是放水**：修题。七字段、不保存、前后两次真实搜索的要求都没变；控制联系人不匹配 marker，不会让搜索"更容易"出现阴性结果。唯一新增的信息是"有一个控制对象，要保护它"，这不算提示答案。

**实施方处置的不足**：首轮仍按普通 FAIL 计入（`core-results.md:506`），没有标成"准备缺陷，不计入产品/指引评估"。读"3/10"时会把这次误当成能力失败。

### 第 10 题：OpenStreetMap（通过）

**题面验收点**（`r7-runtime-preparation.md:123`）：
1. 搜索 Tower Bridge, London；
2. 选中实际的桥梁对象，读名称、类别和页面给出的位置事实；
3. 改变缩放并平移；
4. 经本题搜索结果回到同一对象，核对面板和地图都已回原位；
5. 关闭本题标签页。

不苛刻。

**核对**：
- 000013：搜索结果为 Bridge，位置 "Thames Path, East Smithfield, …, EC3N 4AE, United Kingdom"，`map=16/51.50554/-0.07534`；
- 000015：Way 378541210，Version #37，`man_made=bridge`，`bridge:structure=suspension`，比例尺 100 m；
- 000017：200 m，URL 为 `#map=15/51.50554/-0.07534`；
- 000019：URL 为 `#map=15/51.50555/-0.06896`；
- 000023：100 m，同一 Way；
- 000025：Closed Way…，5 standard tabs。

最终答复的每个事实都有依据，**扎实**。

**过程中的产品行为**：
- CLI[4] fill 为 exit 3（发现 B），模型先 snapshot 再 `ime`，处理正确。
- CLI[8] `wait --contains Search --timeout 30s` 为 exit 1。回执 `complete=true, unknown=0, matches=0`，同时附带的树里只有 "Main" 展开按钮，没有 Search 文字。**产品行为正确**，是模型猜错了页面文字，浪费 30 秒。
- 000007 是加载中的空 WebView，产品已经标了 loading。

## 四、本批 10 次尝试的全部非零退出（13 次）

| 尝试 / 命令 | 真实退出码（模型看到的） | 原因 | 模型反应 | 回执是否帮到模型 |
| --- | --- | --- | --- | --- |
| 07 首轮 CLI[9] `check e241580321` | 3（1） | Activity 重建，原窗口消失；观察时新窗口还没出现，没有树 | 同一 exec 已排好 recents（偏差），从 recents 回执读到 ON 后继续 | 部分：明确了"已派发、不要自动重复"，但没有给出新状态 |
| 07 首轮 CLI[24] `uncheck e241580843 --settle 3s` | 3（1） | 同上，但回执附了新树，显示 OFF | 只读 snapshot 确认 OFF，然后**按旧句停止** | 树有帮助，结论 UNKNOWN 加上指引旧句导致误停 |
| 07 重试 CLI[10] `check e241581262 --screenshot` | 3（1） | 同上，回执新树显示 ON | 模型明确说"返回的新页面已明确显示 ON"，继续 | 有帮助（靠模型自己读树） |
| 07 重试 CLI[25] `open com.android.deskclock` | 3（1） | Clock 自己的通知权限页盖在任务顶部，落地判定只看顶部包名 | 只读 snapshot 后**按旧句停止** | 部分：给了 visibleTasks，但没有 baseActivity，看不出这是 Clock 的任务 |
| 08 首轮 CLI[4] `fill … --submit` | 3（1） | Chrome 假搜索框把焦点交给地址栏 | snapshot，然后 `ime` | 有帮助：新树给出了地址栏 ref 和值 |
| 08 重试 CLI[4] | 3（3） | 同上 | 同上 | 同上 |
| 09 首轮 CLI[4] | 3（1） | 同上 | 直接用回执新树里的 ref 执行 `ime` | 同上 |
| 09 重试 CLI[4] | 3（1） | 同上 | snapshot，然后 `ime` | 同上 |
| 10 CLI[4] | 3（3） | 同上 | snapshot，然后 `ime` | 同上 |
| 10 CLI[8] `wait --contains Search --timeout 30s` | 1（1） | 页面确实没有 Search 文字（移动版收进了 Main 菜单） | snapshot 加截图，看图后点 Main | 有帮助：`complete=true`，附带的树能看出原因 |
| 01 CLI[15] `click e241549385` | 2（2） | 倒计时文字变化导致 ref 失效（发现 F） | snapshot，改点 Stop | 有帮助：给了 freshRepair 新树 |
| 03 首轮 CLI[11] `scroll e241552610 down` | 2（0，与后续命令同一 exec，被覆盖） | 前一条 back 弹出了舍弃对话框，ref 不在当前观察里 | 点 CANCEL，回到编辑器后同一 ref 再滚动成功 | 有帮助：正确拦截了在对话框背后的输入 |
| 03 重试 CLI[12] `click e241564167` | 2（1） | 同上 | 点 CANCEL 后，同一 ref 再点成功 | 同上 |

说明：
- "模型看到的"取自 native 中 `exec_command` 返回的 `exit_code`。03 首轮那条 exit 2 落在多条命令的 exec 里，被后面的命令覆盖成 0，具体见 native 第 109 行附近。
- 新十题全集是 13 次 exit 3、9 次 exit 2、3 次 exit 1。本批 10 次占其中 9 次 exit 3、3 次 exit 2、1 次 exit 1。
- 13 次 exit 3 里，7 次是 Chrome 假搜索框（本批 5 次，04 题 2 次），4 次是 07（3 次 Activity 重建，1 次权限页），2 次在 06 题（不在本批）。

## 五、对实施方分析与处置的总体评价

- **核对过、没问题的部分**：
  - 逐命令真实退出码，以及"不采用模型自称 exit1"；
  - 08、09 的阻断事实；
  - 01、03 重试、10 的 PASS 证据链；
  - 07 的恢复安全核验（原 night mode 为 no、custom 22–06，Clock 权限原状，`EV/runtime/api34/commands/r7-new-07-*`）；
  - 409e853 相对 82f3a82 只改了 SKILL 这一句（`git diff 82f3a82 409e853`，本人已核）。
- **不足之处**：
  1. 07 回避了产品层面，见发现 A、C。
  2. 没有把 7 次网页 fill UNKNOWN 合在一起看成同一个高频问题。它们在 `core-results.md` 里被逐条当成"真实失败前缀"处理，没有提出任何改进。
  3. 03 首轮没有标成准备缺陷。
  4. 549 空 WebView 的表述，容易被读成第 8 题失败的原因。
  5. 没有发现评测包装吞退出码。
- **过度投入的迹象**：`core-results.md` 和 `r7-cumulative-validation.md` 大量篇幅用在 root 字段计数、成本对账、"不回填"之类的声明上。对"哪里可以让下一次任务成功"的分析反而很少。

## 六、需要用户决定的问题

**问题 1：check/uncheck（以及 fill）的目标窗口被替换后，产品是否尝试重新识别？**
- 背景：发现 A。3 次主题切换全部成功，却都报 UNKNOWN。
- 选项：
  - A. 保持现状，只靠新句和模型自己读树。代价为零；假 UNKNOWN 照旧。
  - B. 结论仍为 UNKNOWN，回执增加 `replacementCandidate`：同包、同 resource id、同角色、同位置的唯一节点，及其 checked/value。代价中等；不会误判；模型可以少拍一次快照，指引可以直接写"候选值等于期望值算可靠业务成功"。
  - C. 满足 B 的严格条件时判 done，并标 `reidentified=true`。省得最多，但有误判风险，需要反例测试。
- 推荐 B。先在设备上验证 B 的候选是否稳定，再考虑 C。

**问题 2：Chrome 新标签页输入 URL 的 UNKNOWN 怎么处理？**
- 背景：发现 B。所有网页尝试都命中。
- 选项：
  - A. 只改指引：打开网址首选 `open --url`。代价低，但模型不一定照做，题面要求"在本标签页打开"时也不适用。
  - B. 产品识别焦点移交：原字段消失、唯一已聚焦可编辑框的值等于所填值时，fill 报 done 并标 `focusMovedTo`；submit 仍不派发，或在同一 IME 动作下派发到新字段（后者要突破"不选别的字段"原则，需要用户点头）。
  - C. A 和 B 都做。
- 推荐 A 立刻做；B 中只做"fill done 加 focusMovedTo、submit 不派发"的部分。

**问题 3：`open` 遇到目标 App 自己的权限页怎么报？**
- 选项：
  - A. 保持现状。
  - B. `visibleTasks` 增加 baseActivity，并增加落地状态 `coveredInTargetTask`，结论仍为 UNKNOWN。
  - C. 判 done，附加 `coveredBy`。
- 推荐 B。代价小，模型可以据此安全地 Back。是否升为 C 由用户定（权限页属于目标任务，C 也说得通）。

**问题 4：新十题结果怎么记？**
- 建议把 08、09 标成"外部阻断，无产品信号"，把 03 首轮标成"准备缺陷"，分开列出，不和其它 FAIL 混算。下一轮网页题加可达性预检或备用站点。
- 这是读法和评测设计上的建议，不需要改产品。

## 七、汇总：本批暴露的问题里，哪些值得下一轮修

按对真实任务成功率的影响排序：
1. **Chrome 新标签页 URL fill 的 UNKNOWN**（发现 B）：频率最高，所有网页任务都会遇到；本身不致败，但与任何停止规则叠加就会致败。先改指引，再考虑产品识别。
2. **配置变更导致窗口重建时 check/uncheck 报 UNKNOWN**（发现 A）：本批唯一真正导致失败的产品因素（与指引叠加）。
3. **open 被目标任务自己的权限页覆盖**（发现 C）：API 33+ 很常见，改动小。
4. **空 WebView 却报完整**（发现 D）：先上设备复现，再决定。
5. **评测包装传递退出码**（发现 E）：改评测脚本，不改产品。

不值得动的：
- TfL 和 Gutenberg（外部原因）；
- 第 10 题的 wait 失败（产品行为正确）；
- 舍弃对话框后的 stale 拒绝（产品正确拦截）；
- 倒计时 ref 失效（发现 F，频率低、有替代路径）；
- 07 首轮的"顺排 recents"偏差（危害小，不值得加规则）。

**对"3/10"的读法**：就本人负责的 6 题，可以评价产品的只有 01、03（重试）、07、10，其中 3 题通过；07 的失败可以通过发现 A、C 或指引新句消除，但新句未经实测。08、09 没有信号，03 首轮是准备问题。另外 4 题（02、04、05、06）由另一位评审负责，本人没有判断。因此，3/10 既不代表产品在没见过的任务上的真实水平，也不能反过来说"去掉外部因素后就全对"。
