# AndroidCtl 三个设计问题的独立方案（干净上下文）

材料范围：只用了 `clean3/` 下的 `user-messages.md`、`source-candidate/`、`source-baseline/`、`paired-runtime/`。没有设备，也没有运行 Android。
标注约定：【实测】指原件里直接可见或由原件算出的数；【源码推断】指读代码得出的行为；【设计建议】是我的主张；"需设备"指只有真机或模拟器才能确定。
token 数用 `tiktoken o200k_base` 计，作为 gpt-6.1-sol 分词器的近似。校验：contacts-2 第一次读 SKILL.md 共 8,576 B，o200k 计 1,908 tok；原生 usage 在相邻两次响应之间增加 2,036（其中含 118 输出）。两者相符。
统计脚本在 `out/steps.py`、`out/rollout.py`、`out/compose.py`、`out/propose2.py`、`out/simulate.py`。

---

## 问题一：默认输出应该长什么样

### 结论

1. **候选版的默认输出比基线更重，而且加重的部分基本没有被用到。** 同一任务里，CLI stdout 是基线的 4.3 倍（Contacts）和 5.1 倍（Chrome），模型的累计输入是 1.80 倍和 2.35 倍。约一半字节来自每个可操作节点都打印的完整 `actions` 对象数组（`{"id":…,"name":…,"label":null}`）。其余主要是结果行里的 JSON 诊断块（windows/selection/content/baseline*/effect）、每步重复的 window/event 行，以及长绝对路径。
2. **目标设计**：stdout 由一行结果加一个紧凑的屏幕正文组成。正文每个节点一行，格式为 `角色 [REF] "名称" [=值] [状态词] [特殊动作]`，不用 JSON。同时：
   - 不打印通用动作、事件流、window 目录（异常窗口除外）、采样元数据和绝对路径；
   - 正文默认上限约 8,000 字符，超出部分打印省略行，指明可展开的 REF；
   - 保存的文件和 `--json` 保留全部事实。
   - 在 Contacts 的 fill 页上，这一行加正文为 554 B / 169 tok，现状是候选 7,255 B / 2,131 tok、基线 2,476 B / 715 tok。
3. **用全部真实步骤回放估算**：候选两臂 stdout 合计从 44,077 和 54,887 tok 降到约 3,300 和 4,350 tok，累计模型输入估计降 27% 和 34%。剩下的差距主要来自开局读文档。
4. **公开指引**：只保留一份 SKILL.md，约 3–4k tok，覆盖 90% 的操作循环；其余主题按需读。实测 agent 在第一个动作之前读了 19–25k tok 文档，还因为单次输出 10k tok 的截断重复读。由于这些文档在之后每次响应里都被重新计入上下文，它们是累计输入的最大单项。

### 1.1 现状量化

**整体对比【实测，native-metrics.json 与 rollout 逐响应 usage】**

| 臂 | 版本 | 响应数 | 累计 input tok | 末次上下文 | CLI stdout | stdout(o200k) | 墙钟 |
|---|---|---|---|---|---|---|---|
| contacts-1 | 基线 | 20 | 812,987 | 50,891 | 34,879 B | 9,791 | 171 s |
| contacts-2 | 候选 | 24 | 1,461,258 | 97,187 | 149,803 B | 44,093 | 207 s |
| chrome-2 | 基线 | 27 | 1,041,124 | 55,662 | 36,131 B | 10,636 | 256 s |
| chrome-1c | 候选 | 29 | 2,447,717 | 122,611 | 184,116 B | 54,905 | 300 s |

**stdout 构成【实测，`out/compose.py` 按行类别统计】**

| 类别 | contacts-1 基线 | contacts-2 候选 | chrome-2 基线 | chrome-1c 候选 |
|---|---|---|---|---|
| 节点行 | 46% | 70.5% | 33% | 67.5% |
| 其中 `actions` 属性 | 16% | **56%** | 4% | **51%** |
| 结果行（首行） | 18% | 11% | 28% | 8.5% |
| 元数据行（`<?androidctl`/`androidctl {…}`） | 17% | 4.7% | 20% | 5.9% |
| window 行 | 15.6% | 3.9% | 14.8% | 3.8% |
| events/event 行 | — | 8.6% | — | 12.9% |

**同一步对比：Contacts 中 First name 字段 fill 之后【实测】**

- 基线（`contacts-1` 第 4 步，2,476 B / 715 tok）的内容：
  - 结果行 `fill: done; changed; focus=done, sent=false; fill=done, verification="matches", actualValue=…; snapshotPath="D:\\…长路径…"`；
  - 1 行 XML 元数据处理指令；
  - 4 行 window；
  - 约 15 行 XML 节点。可编辑节点带 `actions="clear-focus select … action-1020036 ime"`。
- 候选（`contacts-2` 第 3 步，7,255 B / 2,131 tok）的内容：
  - 结果行内嵌 `windows={…}; selection={…}; content={…}; baselineSequence=6; baselineFinishedMs=…; baselineAgeMs=…; baselineComputedAtMs=…; effect={…"actions":{"before":{"jsonPreview":"[{\"id\":2,…"…}`；
  - 1 行 `androidctl {…}` 元数据；
  - 4 行 window JSON、1 行 events JSON、1 行 artifact；
  - 15 个节点。每个可操作节点带 8–12 个 `{"id":…,"name":…,"label":null}` 对象，例如 Cancel 按钮这一行有 10 个动作对象。

**产生原因【源码推断】**

- `source-candidate/androidctl/src/androidctl/ui_view.py:120-121`：只要节点是 target，就把 `node["actions"]` 整个对象数组放进属性。基线 `source-baseline/androidctl/src/androidctl/ui_view.py:52-77` 只给可编辑节点打印动作名，其余节点只打印 expand/collapse/dismiss/long-click。
- `ui_view.py:72-86` 的 `is_target` 把"有任何不在 ORDINARY_VIEW_ACTIONS 里的动作"都算作 target。`ORDINARY_VIEW_ACTIONS`（`ui_view.py:53`）不含 `16908348`（CONTEXT_CLICK）和 `16908342` 以外的一些通用 ID。Chrome 的网页节点几乎都带 CONTEXT_CLICK、`action-400/800`（HTML 元素导航），所以都成了 target。
  - 【实测】chrome-1c 首次 snapshot 正文只有 93 行，但有 40,403 B：82 个 REF 中有 76 个带 `action-102003c`，平均每节点约 470 B。
- 结果行的 JSON 诊断块：`ui_client.py:271-292`。
- 默认打印 window/event 行：`ui_view.py:333-375`。
- 节点行固定为 `role {JSON}`：`ui_view.py:525-530`。
- 默认打印正文且没有上限：`SKILL.md:41`。`--max-chars` 默认是 None（`ui_commands.py:225`）。

**大输出导致的绕路【实测】**

- 该 harness 对单次工具输出的截断点是 40,153 B（10k tok）。候选里被截断的有：
  - contacts-2 的 `snapshot --cached --all`（41,888 B）；
  - chrome-1c 的首次 `snapshot`（40,403 B）；
  - chrome-1c 的 `click --interactive`（39,979 B）：`--interactive` 没有起到压缩作用，因为几乎全部节点都是 target。
- 被截断后 agent 用 `Get-Content` 重读产物文件，又被截断。chrome-1c 在这几次上共花了约 34.6k tok：11,266 + 11,235 + 12,149。
- 两个候选臂各多出 4 次和 2 次模型响应。
- 能力层面的对照：
  - 基线 Chrome 根本拿不到网页树，只有 `text name="Web View"`，全程靠截图和坐标完成。
  - 候选拿到了网页树，但 `find Addis Ababa` 因采集 `partial`（collectionLimit）返回 `matches=0`，最后仍然截图并用坐标点击（chrome-1c 第 7–9 步）。
  - 所以这对任务里，候选新增的网页事实主要用在读详情页首段（`snapshot e408552907`，3.8 KB）。默认输出的体积没有换来相应的收益。

### 1.2 目标设计【设计建议】

**结果行（第一行）**：`命令 目标: 结果; 一个原因; 关键事实`

- 结果只有 `done / failed / not sent / UNKNOWN / still running` 五个词，语义与问题三统一。
- 变化只用一个词描述，由 observedChange 和 effect 派生：`new screen`、`content changed`、`no visible change`、`unsettled`、`change unknown`。不打印 before/after JSON、baseline* 序号或时间。
- fill/type 类打印读回值 `value="…"`。读不到时写 `value unobservable`，此时结果不能是 done。
- 失败时打印 `code=`、`dispatchStarted=`，以及已执行前缀（仅 batch）。

**屏幕行（第二行）**：`screen 包名 "标题" keyboard=shown|hidden [partial(原因 N)] [unsettled] [display=virtual#id] #序号 saved=短文件名`

- `partial` 和 `unreadable` 这类"缺口事实"必须出现在这一行，因为它决定 agent 能否把"没找到"当作"不存在"。
- 序号和短文件名替代绝对路径和整行元数据。绝对路径只放在 `--json` 和保存的文件里。

**节点行**：`角色 [REF] "名称" [=“值”] [~"detail"] [状态词…] [scroll方向] [{特殊动作}]`

- 只给"可用的目标"出 REF，包括 click/long-click/编辑/勾选/滚动/range 和非通用动作。只有 CONTEXT_CLICK、HTML 导航、accessibility focus、SHOW_ON_SCREEN、select/clear-selection 这些通用动作的节点不算目标。
- 状态词：`focused selected checked=… disabled offscreen password heading covered unreadable`。
- 特殊动作只列 expand/collapse/dismiss、不可 click 时的 long-click、带 label 的自定义动作，以及聚焦编辑框的 `ime=Enter`。click/fill/focus/select 由角色隐含，不再打印。
- 无名、无值、非目标的容器节点直接省略，子节点上提。与父节点名称重复的子 text 省略。
- `#resourceId` 只在节点没有名称且是目标时打印。默认不打印 `targetUrl`、`chromeRole=genericContainer` 和 collection 行列信息。

**不默认打印的内容**：window 目录（只在出现应用主窗口、IME、系统栏以外的窗口时打一行 `also: popup/dialog/overlay 包名`）、recent events、artifact 状态、采样元数据、动作 ID 列表、boxes。

**上限**：正文默认约 8,000 字符。超出时以 `… N nodes omitted: snapshot --cached REF | find TEXT` 结尾，列出被截断的顶层子树 REF。这复用现有 `omission`（`ui_view.py:533-545`），只是把可选的 `--max-chars` 改成默认值。

**显式出口**：
- `snapshot --cached REF`：该子树的全属性视图，含动作 ID、label、box、URL、collection，供 `action --action-id` 使用；
- `--all`：包含 offscreen 节点；
- `--json`：完整结构；
- 保存的文件：全属性版本，可以直接"展开同一文件"。

我建议删掉 `--depth`、`--interactive` 和 `--max-chars` 选项。默认视图已经足够紧凑，这三个选项在实测中也没有让输出变小（chrome-1c 的 `--interactive` 仍有 39,979 B）。

**手写样例：Contacts "Create new contact" 页，First name 字段 fill 之后**（`out/sample-fill.txt`，554 B / 169 tok）

```
fill e834208100: done; value="ACV-20261117-AB中😀Z" (read back)
screen com.android.contacts "Create new contact" keyboard=shown #8 saved=000003.txt
button e834208075 "Cancel"
text "Create new contact"
button e834208078 "SAVE"
scroll e834208080 #contact_editor_scroller [up down]
  item e834208084 "Add contact photo"
  text "Saving to | Device"
  image "Name"
  textbox e834208100 "First name" ="ACV-20261117-AB中😀Z" focused ime=Enter
  textbox e834208101 "Last name" =""
  item e834208103 "Show more name fields"
  textbox e834208111 "Phone" =""
```

| 同一步 | 字节 | o200k tok |
|---|---|---|
| 基线 contacts-1 #4 | 2,476 | 715 |
| 候选 contacts-2 #3 | 7,255 | 2,131 |
| **目标** | **554** | **169** |

完成任务需要的事实都在里面：保存位置 "Saving to | Device"、各字段 REF、读回值、焦点、键盘状态。

**全量回放估算**：`out/propose2.py` 按上面的规则机械地重写候选两臂每一步的 stdout，不补充任何信息，也不做 web 段落合并。

| 臂 | 现 stdout tok | 目标估计 | 首个 snapshot | `snapshot --all` 或首屏 |
|---|---|---|---|---|
| contacts-2 | 44,077 | ≈3,290 | 1,234→76 | 12,425→486（`--all`） |
| chrome-1c | 54,887 | ≈4,350 | 12,229→498 | — |

`out/simulate.py` 按 rollout 顺序，把每步节省量从之后每次响应的 input 里扣除：
- contacts-2 累计 input 1,461,258 → 约 1,060,000（−27%）；
- chrome-1c 2,447,717 → 约 1,615,000（−34%）。

这只是上下文规模的估算，没有计入 agent 因此少走的弯路（不再被截断、不再重读文件），实际可能更低。结论仍要靠成对重跑确认。

### 1.3 丢掉了什么、如何拿回、风险

| 丢掉的信息 | 拿回方式 | 风险 |
|---|---|---|
| 每个节点的动作 ID/label 全集 | `snapshot --cached REF` 全属性视图、`--json`、保存的文件 | agent 想用 `action --action-id` 时多一次本地读，不访问设备 |
| window 目录、events | `snapshot --windows`（现有）、`--json` | 弹窗或悬浮窗只剩一行 `also:` 提示。需要靠 `also:` 规则覆盖到位，"需设备"验证各类系统窗口 |
| before/after/baseline 诊断 | `--json` 的 `observedChange`、`effect` | 低。agent 本来就不该把变化当作业务成功（SKILL.md 自己的规则） |
| 网页 URL、行列信息 | REF 视图 | 判断链接去向时多一步；也可以直接看地址栏 |
| 超过上限的正文 | 省略行给出 REF 和 `find` | 长列表的第一屏之外需要显式展开。这本来就是 `find` 和 `scroll` 该做的事 |
| 通用动作 target 的 REF（如 Chrome 容器） | `--all` 或全属性视图 | 极少数只响应 CONTEXT_CLICK 的网页控件拿不到默认 REF。需设备验证 |

主要风险：规则过滤得太多，导致 agent 漏看。所以缺口事实（`partial`、`unreadable`、`covered`、`offscreen`）一律保留，并且出现在 screen 行。

### 1.4 公开指引组织【实测 + 设计建议】

**第一个 CLI 动作之前读的文档**【实测】：

| 臂 | 读取次数 | 文档 tok | 备注 |
|---|---|---|---|
| contacts-1 | 3 | 18,708 | reference.md 被截断后重读 |
| contacts-2 | 6 | 25,259 | 2 次截断 |
| chrome-2 | 1 | 3,948 | 只读了 SKILL.md；后来按需读 reference，9,052 tok |
| chrome-1c | 4 | 23,823 | 1 次截断；后来又读了 images.md，3,380 tok |

- 候选把文档拆成 12 个主题文件，共约 150 KB。SKILL.md 本身（8 KB）用大量限定句写成，例如"Partial collection can establish reliable positive facts; gaps cannot prove absence…"，但没有一个"读输出"的最小例子。agent 于是倾向于把 actions/input/observations 都读一遍。
- 这些文档 token 在之后约 20 次响应中都会重复计入。粗算 contacts-2 约 25k × 18 ≈ 45 万，接近累计 input 的三分之一。
- 如果开局只需约 4k tok，再加上 1.2 的输出改动，Contacts 累计 input 估计约 0.7M，低于基线的 0.81M；Chrome 约 1.1M，接近基线。这是估算。

建议：
1. SKILL.md 控制在 3–4k tok：
   - 一个完整的最短会话：open → snapshot → click/fill → 读结果 → close；
   - 一个输出样例及逐项释义；
   - 五种结果词各一句处置；
   - 主题索引，每个主题一句"什么时候需要它"。
2. 主题文件按能力拆分：网页与长页、图像/坐标、显示器、通知、剪贴板与文本输入、batch、jobs/files、连接与故障。每个文件开头放 5 行以内的"何时读"，正文先写例子再写规则。
3. 把只对实现者有意义的保证性语句移回内部设计文档，这与 AGENTS.md "detailed internal design stays in the separate docs repository" 一致。例如 "ownership check and restore are not a CAS operation"。
4. `androidctl help TOPIC` 与这些文件同源，保持现状即可。

---

## 问题二：显示目标的最简模型

### 结论

- 显示种类本来就记录在设备端的租约上（`Lease.target.kind`）。真正可能"静默落到主屏"的路径只有一条：**UI 请求在没有租约时隐式新建一个 MAIN 租约**（`Sessions.kt:90-97`）。
- 现状的做法是保留隐式租约，在主机上持久保存一份"期望种类"，每个请求都带上它让设备比对。这个做法新增了：
  - 主机文件 `display.json` 和它的锁；
  - 协议必填字段 `expectedDisplayKind`；
  - `display select` 命令；
  - `display create` 成功但保存失败时拆开报告的结果；
  - batch 期间冻结期望种类。
- 即便如此，现状仍有一个静默主屏的洞：首次 `display create` 失败或 UNKNOWN 时，偏好保持 main，下一条 UI 请求会隐式拿到 main。另外，过期偏好会跨任务、跨工作区持续生效，让之后的任务卡在 `displayMismatch`。
- **我的模型【设计建议】：取消隐式租约，主机端不保存任何显示状态。**
  - 只有显式的 `session open [--display main|virtual] [--idle D]` 能建立租约，它取代 `display create`；
  - 删除 `display select`、`display.json` 和 `expectedDisplayKind`；
  - 租约因过期、worker 重建、close 或被接管而消失后，任何命令都返回 `not sent; code=noSession; run session open [--display virtual]`；
  - 结果的 screen 行在虚拟屏时标注 `display=virtual#id`。
  - 状态从"主机偏好 × 设备租约"两层组合减为设备租约一层，静默主屏的路径不复存在。
- **能力本身**：现有证据不足以作为正式能力发布。建议本次发布先移除虚拟屏；若保留，只按上面的极简模型，并标为"未在任何真机验证"。所缺证据见 2.4，均"需设备"。

### 2.1 现状模型【源码推断】

**主机端**
- `workspace.py:27-111`：`state_directory()/display.json` 的结构是 `{deviceId: {session: main|virtual}}`。读坏了会让全部 UI 请求失败；写操作带锁，遇到竞争直接失败；"已发布但 fsync 或清理失败"作为单独状态处理。
- `ui_client.py:594-607`：凡是 `uses_display(command)` 的请求（除 notifications/clipboard/apps/wake 以外），首次调用时读取偏好并冻结，然后作为 `expectedDisplayKind` 加进参数。
- `ui_batch.py:307-309`：batch 开始前冻结一次。
- `app.py:774-805`：`display select` 只写本地文件。
- `app.py:808-863`：`display create` 成功后写 virtual；写失败时设备上的屏照样保留，返回非零退出码并附 `displayPreferenceError`。
- `session open/takeover`（`app.py:732-771`）总是 main，且不改偏好。

**设备端**
- `DisplayTarget.kt:13-15`：缺少该字段就报 `arguments` 错误。
- `RpcMethods.kt:48-56`：判断哪些命令需要显示目标。
- `Sessions.kt:56-64, 136-150`：`acquire`/`acquireUi` 在没有租约时 `keepLease` 新建 MAIN 租约。期望 virtual 时先在 `requireTarget`（`Sessions.kt:152-170`）里报 `displayRequired`；种类不符时报 `displayMismatch`。
- `TargetManagement.kt:59-61, 113-126`：`display.create` 先 `prepareChange`，使旧租约失效，再创建 `WorkDisplay`；失败或超时则不发布，并关闭新屏。
- `UiRun.kt:92-95`：响应返回后 `checkActive` 抛 `requestEnded`，因此 UNKNOWN 之后不会再发布。
- `WorkDisplay.kt:37-77`：请求 TRUSTED/OWN_FOCUS/STEAL_TOP_FOCUS_DISABLED 标志，缺任何一个就失败；截图走隐藏的 `IWindowManager.captureDisplay`（`WorkDisplay.kt:89-111`）。
- `DeviceFacts.kt:76-81`：要求主屏 ON 且可输入。

### 2.2 情形推演

"静默主屏"指 agent 本意是操作后台屏，结果没有收到任何拒绝就操作了用户主屏。

| 情形 | 现状 | 我的模型 |
|---|---|---|
| 正常使用 | `display create` → 写 virtual → 每个请求带 virtual → 设备核对种类 | `session open --display virtual` → 租约种类为 virtual → 请求直接落到该租约 |
| 后台屏到期 | 偏好仍是 virtual → UI 请求报 `displayRequired`。但 notifications/clipboard/apps/wake 会隐式建 MAIN 租约（`acquire(owner, null)`），之后 UI 请求报 `displayMismatch`。安全，但多一种状态 | 任何命令都报 `noSession`。agent 自己决定重开哪种 |
| worker 进程重建 | 同"到期"。屏随进程一起销毁，迁入的 task 可能丢失 | 同"到期" |
| `display create` 失败 | 旧租约已失效，偏好不变。**如果之前从没选过 virtual（偏好 main），下一条 snapshot 会隐式拿 MAIN 租约：静默主屏** | 没有租约 → `noSession` |
| `display create` UNKNOWN | 偏好不变（U25 规定），后果同上，有静默主屏风险；设备侧不会再发布 | `noSession`；若在响应前已发布，则是 virtual 租约。两种都安全 |
| `session open` | 建 main 租约，偏好不变。**若偏好是 virtual，下一条 UI 报 `displayMismatch`，提示 "run session open or display select virtual"，照提示再 `session open` 会原地循环**，必须改用 `display select main` | 建立所请求种类的租约，没有其它状态 |
| `session close` | 租约消失、偏好保留。偏好为 main 时，下一条 UI 隐式重建 main | `noSession` |
| 换工作目录 | 按 U25，偏好是主机级 deviceId+session，跨工作区共同生效 | 设备租约本来就按配对+会话名区分，行为天然一致，主机侧没有可写坏的文件 |
| 使用默认会话名 `default` | 所有没写 `-s` 的任务共享一份偏好。前一个任务留下的 virtual 会让后续任务卡住，见上面"`session open`"一行 | 每个任务自己 `session open` 时决定种类，不会遗留 |
| 之后不相关任务复用设备 | 会话名相同时同上，被遗留偏好卡住；会话名不同时正常（main） | 不受影响。租约还在时报 busy，与现状相同 |
| 两个进程同时用同一会话名 | 偏好共享，种类核对只防"种类不同" | 剩余风险：B 显式 `session open`（main）后，A 的下一次全新 `snapshot` 会落到 B 的 main 租约。带 REF 的动作会因 REF 过期失败。这本质上是会话名冲突问题，现状同样存在 refs 混用。可选的小措施是结果行打印 `lease#N`，换号时提示 `new session` |

**比较**

| 维度 | 现状 | 我的模型 |
|---|---|---|
| 持久状态 | 主机 `display.json`、锁文件，加设备租约 | 只有设备租约（内存） |
| 种类状态的组合 | 偏好{无,main,virtual} × 租约{无,main,virtual} = 9 种，各自对应不同错误和处置 | 租约{无,main,virtual} = 3 种 |
| 协议面 | 每个 UI RPC 必填 `expectedDisplayKind`；`display.create`；`display select`（本地）；`displayRequired`/`displayMismatch` 两个错误码；create 结果中的 `displayPreference`、`displayPreferenceError` | `session.open` 加一个 `display` 参数；一个 `noSession` 错误码 |
| 失败模式 | 文件损坏会阻断全部 UI；保存失败要拆开报告；遗留偏好会循环卡住；首次 create 失败或 UNKNOWN 后有静默主屏 | 只有"租约没了就重开"一种 |
| 要删除的代码（约） | — | `workspace.py` 显示部分约 100 行、`app.py` display 命令约 90 行、`ui_client`/`ui_batch` 冻结逻辑约 20 行、`DisplayKind.requested` 和 `requireTarget` 约 30 行、`test_display.py` 567 行中的大部分 |

代价：主屏用户在空闲 10 分钟到期或 worker 重建之后，必须多执行一次 `session open`，不再自动续上。由于这两种情况下旧 REF 本来就全部失效（实测输出 "New worker: previous refs are invalid"），agent 总要重新观察，多一条命令的成本很低，换来的是两种显示目标都"永不隐式取得目标"。jobs/files/notifications/clipboard 如果需要会话互斥，也要先 open。这一点在指引里一句话就能说清。

这个模型满足 U23 的核心诉求：期望 virtual 但没有后台屏时，在取得 main 之前返回 not sent。它放弃的是 U23/U25 指定的实现手段（主机偏好加协议必填字段）。用户已声明历史决定可以推翻，这里给出替换理由：状态更少，并且堵住了上表里的静默主屏洞。

### 2.3 什么场景有价值

- **前提**：主屏必须亮且解锁（`DeviceFacts.kt:76-81`、`displays.md:36-39`），并且没有保亮机制。因此"手机放着无人看管、agent 在后台干活"不成立，这种情况直接用主屏更简单。
- **真正的场景**：用户正在用手机做 A，agent 在虚拟屏操作另一个 App B，互不抢焦点（OWN_FOCUS、STEAL_TOP_FOCUS_DISABLED）。
- **限制**：
  - 同一包已有 task 在别的显示器上时禁止启动（`displays.md:67-69`），所以不能操作用户正在用的 App；
  - 关闭屏会销毁迁入的 task 和未保存的状态；
  - IC 输入仅限主屏（`input.md:16-17`），后台屏上只剩 fill 和按键；
  - 截图依赖隐藏接口 `captureDisplay`。

### 2.4 发布证据判断

**我能看到的证据**：
- 主机和设备端单元测试：`DisplayTargetTest.kt` 有 8 个测试，覆盖种类核对、到期不隐式替换、wake 保留租约等；`SessionsTest`、`test_display.py` 覆盖偏好文件。
- 用户消息记录：MuMu 缺少可信后台屏权限；AOSP 模拟器能跑后台流程，但"独立电源未成立"，且销屏可能销毁迁入的旧 task（`user-messages.md` L67）。
- 没有真机证据；源码和测试里也看不到任何真实 App 在虚拟屏上完成业务的记录。

**判断**：不足以作为正式能力发布。

**需要的证据（均需设备）**：
1. 至少一台 Android 14+ 真机（Pixel 类），再加一种国产 ROM：shell UID 能否保留 TRUSTED/OWN_FOCUS/STEAL_TOP_FOCUS_DISABLED 三个标志。
2. 3 个以上真实 App 在虚拟屏完成"启动 → 读树 → fill/click → 返回"，同时用户在主屏打字，验证没有输入或焦点串扰。
3. 虚拟屏上 IME 弹出到哪块屏，fill 和按键是否受影响。
4. 截图隐藏接口在真机可用，内存峰值可接受。
5. 主屏锁屏或熄屏时虚拟屏的行为与报错。
6. close 时迁入 task 丢失的实际范围。

在这些证据出来之前，最省维护成本的做法是本次发布不含该能力。由于我的模型把显示种类收敛成 `session open` 的一个参数，以后补回来也不需要改动其它基础设施。

---

## 问题三：文本输入应保留几条路

### 结论【设计建议】

保留三条机制，每条对应一个动词，取消 `--via`：

| 命令 | 机制 | 用途 |
|---|---|---|
| `fill REF TEXT` | AX SET_TEXT，整字段替换，读回校验 | 默认首选，后台屏可用 |
| `type TEXT [--replace START END]` | 当前编辑器的 InputConnection（现 `editor commit`；`--replace` 在同一次捕获内先 setSelection 再 commit） | Unicode 插入或局部替换，适用于不支持 SET_TEXT 的编辑器；仅主屏，需要 Browser support 服务 |
| `key CHORD [REF]`，可加 `--text ASCII` | 注入按键事件（现 `key` 与 `type --via keys`） | Enter/Tab/Del/方向键/组合键，以及没有 IC 时输入可映射字符；后台屏可用 |

另外保留 `clipboard get|set`，作为显式的数据操作。

**删除**：
- 借用剪贴板粘贴 `type --via paste`（BorrowedPaste）。被删的能力由 `clipboard set TEXT` 加 `action REF --action-id 32768`（PASTE）或 `key ctrl+v` 覆盖，只是不再自动还原剪贴板。
- 独立的 `editor select`，并入 `type --replace`。
- 剪贴板 null 的 AppOps 准入采样（emptyBasis）。它主要的消费者就是借用粘贴的"空原值还原"。

**选择规则**一句话：先 fill；fill 报 SET_TEXT 不可用或读回不符时，用 `type`；需要按键语义，或者 `type` 不可用（后台屏、服务未启用）时，用 `key`。不做自动回退。

**结果语义统一为四种**：
- `not sent`：证明什么都没发出；
- `done`：本通道自己的后置条件被读回证实；
- `failed`：已发出，读回证明没达成，或平台明确拒绝；
- `UNKNOWN`：已发出，但后置条件不可观测。

文本模式只打印这四个词、一个原因和读回值。attempted/callReturned/successfulPrefix/possibleDown 等过程细节只放在 `--json`。

### 3.1 各途径对比

【源码推断，行数为 `wc -l`】

**fill（AX SET_TEXT）**
- 平台前提：节点公开 SET_TEXT，如标准 EditText、Compose、Chrome 网页输入框。
- 独有能力：整字段原子替换并读回；不依赖焦点或 IME；虚拟屏可用。
- 失败方式：节点不支持（终端、自绘编辑器、部分 WPS 类）；App 的 TextWatcher 或格式化可能改值，此时读回不符会报 failed。
- 实现与维护：在 `UiActions.kt`（1,035 行）里，另有 `FillProgress.kt` 135 行；ENTER 提交回退复用按键。基线已有，属于成熟路径。
- 结果状态：focus/fill/submit 三段。

**key / `type --via keys`（注入 KeyEvent）**
- 平台前提：shell UID 注入权限，经隐藏接口 `IInputManager`、`InputEvent.setDisplayId`（`HiddenApi.kt:145-155`）。
- 独有能力：Enter/Tab/Del/方向/组合键，以及 ctrl+a/c/v 这类编辑器快捷键；能作用于只吃按键的控件；虚拟屏可用。
- 失败方式：文本只能是 KeyCharacterMap 可映射的字符，CJK 和 emoji 一律 not sent（`input.md:95-102`）；焦点可能在派发前变化；有部分前缀生效和按键卡住（possibleDown）的可能。
- 实现与维护：`KeySequence.kt` 212 行、`KeyPlan.kt` 123 行（文本映射只占约 20 行）、`AppInput.kt` 171 行（与触摸共用）、测试 314 行、主机 `input_keys.py` 62 行。结果字段很多：plannedEvents、successfulPrefix、cleanup 等。

**`editor commit/select`（AX 服务的 InputConnection）**
- 平台前提：API 33+ 的 `AccessibilityService.InputMethod`，服务配置 `flagInputMethodEditor`（`res/xml/browser_accessibility_service.xml`）；用户须在系统设置里启用 "AndroidCtl browser support"，这个服务 Chrome 网页树本来就需要；仅主屏和 user 0。
- 独有能力：任意 Unicode；对任何"软键盘能输入的编辑器"都能插入或替换（软键盘本身就走 IC）；可读周边文本做校验。
- 失败方式：当前没有编辑器、EditorInfo 或连接；App 不接受（WPS 选区未生效，见 `user-messages.md` L182）；捕获期间焦点变化会让结果变成 UNKNOWN。
- 实现与维护：`CurrentEditorInput.kt` 171 行、`EditorSurrounding.kt` 129 行、`EditorBridge.kt` 96 行、AIDL、`BrowserAccessibilityService.kt` 中 InputMethod 部分约 45 行、`WorkerConnection` 中的桥接生命周期、测试 345 行、主机 `editor_input.py` 32 行。
- 跨进程链路：shell worker → Binder（UID 闸门，`EditorBridge.kt:39-43`）→ App 主线程 → InputConnection。
- 结果状态：select 有 DONE/UNKNOWN/FAILED 三路判定。

**`type --via paste`（借写剪贴板 → PASTE → 还原）**
- 平台前提：隐藏接口 IClipboard（`ClipboardAccess.kt:98-125`）、AppOps 准入（`ClipboardAccess.kt:127-170`）；节点在聚焦后公开 PASTE。
- 独有能力：不启用 AX 服务也能把 Unicode 插入"支持 PASTE 但不支持 SET_TEXT"的编辑器，并自动还原用户的剪贴板。
- 失败方式：与用户或 App 的复制竞争；还原不是 CAS；App 延迟读取剪贴板会拿到错误内容；剪贴板历史和隐私提示；原值为 null 时需要准入（`input.md:63-90`）。
- 实现与维护：`BorrowedPaste.kt` 354 行、`UiActions.kt:206-320` 约 115 行、`ClipboardAccess` 中借用、还原、准入部分约 80 行、测试 351 行，以及结果行渲染。结果分 borrow/paste/effect/restore 四组，每组又有 cleared/skipped/failed/unknown，是五条途径里最复杂的。

**`clipboard set`，加 action PASTE 或手动粘贴**
- 平台前提：IClipboard 隐藏接口。
- 独有能力：明确地写剪贴板，供 App 自带的粘贴按钮或菜单使用，也供用户自己使用。
- 失败方式：覆盖用户剪贴板，这是显式副作用；读回失败报 UNKNOWN。
- 实现与维护：`ClipboardOperations.kt` 165 行、`ClipboardProjection.kt` 170 行、`ClipboardAccess` 中读写部分、测试 294 行。

### 3.2 最小集合与覆盖分析

- **fill**：默认通道，必须保留。
- **IC（改名为 `type`）**：保留。它覆盖了"fill 不行，又需要 Unicode"的全部可用情形；它依赖的服务因 Chrome 已经存在，额外代价只有 InputMethod 回调和一座桥。
  - 把 `editor select` 并入 `--replace`：在同一次捕获里先 setSelection 再 commitText，最后用周边文本读回比较"期望全文片段"，得到 done/failed/UNKNOWN。这样去掉了独立 select 的三路判定，也消除了 select 和 commit 之间"不是事务"的告诫（`input.md:38-39`）。
  - 这推翻了 U25 "保留独立 select"。理由：独立 select 唯一的独有用法是"先选区，再用别的通道作用于选区"，而 `--replace` 和按键 `shift+方向` 都能覆盖。
- **按键**：保留，用于组合键和功能键，以及后台屏或未启用服务时的 ASCII 文本。文本模式挪到 `key --text`，让 `type` 只有一个含义。由于文本映射只有约 20 行，保留它几乎没有额外成本。
- **借用粘贴**：删除。`clipboard set` 加上 `action REF --action-id 32768`（`actions.md:139-147` 已有通用动作入口）就能完成"写入并 PASTE"，只是不会自动还原用户剪贴板。这个缺口是真实的，但还原本身做不到可靠（不是 CAS、App 可能延迟读取），那一层保证本来就很弱。删除可省约 900 行（含测试）。
- **剪贴板 null 准入**：删除。`clipboard get` 对 null 直接报 `returnedClip=false`（表示空，或没有读取权限），结果为 done。需要区分时，agent 可以先 `clipboard set` 再 `get` 读回。这推翻了 U23 第 2 项的 D 方案；理由是该机制主要服务于借用粘贴的空原值还原。

**删减后真实的覆盖缺口**（均需设备验证）：
1. 不支持 SET_TEXT、不接受 IC commit、又公开 PASTE 的编辑器，在用户不愿剪贴板被覆盖时无路可走（可接受先 `clipboard set` 再手动处理）。这种编辑器是否存在需设备验证。
2. 后台屏上的 Unicode 局部输入：fill 只能整字段替换，按键不能输入 CJK。
3. WPS 一类选区不生效的 App，`--replace` 会报 failed（与现状相同）。
4. IC 在各 App 中的接受度：Contacts 和 Termux 有记录（`user-messages.md` L65、L182），其它需设备。

### 3.3 选择与结果语义

- **选择**：指引里只放一张表：
  - 整字段写入 → `fill`；
  - fill 报 `actionUnavailable` 或读回不符 → `type`；
  - 需要按键语义 → `key`；
  - 后台屏或服务未启用 → `key --text`（仅 ASCII）。
- 不做自动路由，因为三条机制语义不同，自动选择会掩盖"整字段替换"和"插入"的差别。
- **统一结果**：

| 结果 | fill | type（IC） | key |
|---|---|---|---|
| not sent | 没发出 SET_TEXT | 没调用 commit | 没有任何 DOWN 返回 true |
| done | 字段读回等于目标值 | 周边文本读回符合期望（插入或替换后） | 全部事件返回 true；带 REF 时附读回值，但不据此判断 |
| failed | 读回为其它值，或平台拒绝 | 读回为其它值 | 部分前缀生效后被拒，附已生效前缀 |
| UNKNOWN | 已发出但读不到 | 已调用但捕获失效或读不到 | 返回状态不明 |

文本模式打印一行，例如 `type: done; value="…Q中😀Z"` 或 `key: failed; sent 3/5 events`。完整过程事实留在 `--json`。

---

## 我没看到 / 无法确定的

- 没有设备，任何 Android 行为都没有运行验证。其中虚拟屏三项标志在真机上能否保留、IC 在各 App 中的接受度、"只剩 CONTEXT_CLICK 的网页节点"是否确实无须默认 REF，都需要设备。
- 每臂只有一次运行，样本太小。墙钟和响应数的差异可能有随机成分。
- token 用 o200k 近似，与 gpt-6.1-sol 的真实分词有偏差，量级经抽样核对一致。图片 token（基线 6 张、候选 3 张）没有计入我的估算。
- 1.2 的回放估算是对现有输出做机械改写，没有验证 agent 读这种紧凑格式时的决策质量。需要用新格式成对重跑同两个任务。
- rollout 里的推理内容是加密的，我无法确认 agent 选择读哪些文档、做哪些绕路的真实动因，只能从调用序列推断。
- 没有逐行通读全部 Kotlin（18,226 行）。问题三的代码量是文件级统计，按功能切分的行数是估算。
- 用户消息里提到的外部评审报告和研究记录不在材料内，按要求没有查找。U23/U25 背后的实测依据我看不到，所以对它们的推翻只基于本材料中的源码和原件。
