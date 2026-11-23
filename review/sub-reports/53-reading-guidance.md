# 第五轮子评审 53：调用者实际读到的东西（默认输出、命令帮助、公开指引、常见误用）

评审对象：产品 `cab84b8`（主机端与 `skills/androidctl/`），对照第四轮候选 `4235af9` 和主分支基线 `7d5be70`。
工作目录与脚本：`S/agents8/53/`（`S` 为评审暂存根目录），脚本在 `S/agents8/53/scripts/`，中间结果为同目录下的 `cli-r5.json`、`cli-r4.json`、`roll-r5.json`、`roll-r5-summary.json`、`replay-cab.json`、`replay-4235.json`。下文 `ST` 指实施方材料包内的 `ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`。

样本和版本说明：本轮真实任务原件共 70 次运行、950 条 CLI 调用。其中基线臂 16 次（主机 `7d5be70`）。候选臂 54 次，分属四个主机版本：`93fd40d` 1 次，`a6859c1` 34 次，`7693794` 13 次，`cab84b8` 10 次；另有 4 次没有 CLI 调用。四个候选版本的呈现代码与 `cab84b8` 相同（`git diff 93fd40d cab84b8 -- androidctl/src` 只改了 `key` 的帮助一句），所以候选臂的 stdout 可以代表当前的默认输出。指引文件在这几个版本之间有差别，下文用到的地方会说明。执行模型统一是 gpt-6.1-sol / medium，每次运行都是新会话。

---

## 1. 结论摘要

**整体状态**：默认观察输出的精简确实做到了，改动方向正确，本轮最值得保留的成果就是它。但在调用者的总输入里，**指引阅读的占比已经超过了 CLI 输出**。另外，少数几种拒绝在几乎每次运行里重复出现，每次都要多付一轮。现在最便宜的收益在这两处，不在继续压缩呈现代码。

最重要的发现：

1. **精简在真实样本上成立，但"每份都不超过一半"不完全成立。**【重算】我用 `4235af9` 和 `cab84b8` 两版真实的呈现函数，重放了实施方包内留存的全部 22 对 tree/windows 原始样本（去重后是 10 个不同页面）。合计字节比为 43.1%；实施方挑的三份页面分别是 45.9%、48.9%、28.0%，和他们报的数一致。另有一份没被选中的真实网页样本是 **53.4%**（`web/006`），超过一半。（`S/agents8/53/scripts/replay_all.py`）
2. **候选的总输入里，指引阅读占 21.2%，CLI 输出占 17.9%，`--help` 只占 0.4%。**【重算】这里按"内容进入上下文以后，在之后每一轮请求里都被重新计入输入"的方式累计（下文称"驻留占比"），样本是 54 次候选运行、929 轮请求、3,576 万输入 token。其余约 48% 是执行环境每轮固定的约 18.5k token 系统提示。**单是 `actions.md` 就占 11.8%**：54 次里有 45 次在读完首页后马上读它，中位 7,653 token。`observations.md`（492 行）和 `connection.md`（308 行）在 54 次运行里一次都没被读过。
3. **动作回执默认附带整页观察，占候选全部 stdout 字节的 41%；`snapshot --all` 一共只调用了 32 次，却占了 39%。**【原件】【重算】在默认文本里，即使把显然没用的东西删掉，也不损失任何能力，还能再省大约 35%（离线估算）。主要几项是：网页上每个节点都重复的 `action={"id":16908348,"name":"context-click"}`（占默认输出 6.8%）、每次观察开头那行溯源 JSON（8.4%）、`chromeRole`、行列序号、自定义动作的数字 ID，以及很长的绝对 `snapshotPath`。
4. **常见误用的真实频率比实施方记录的高，每次被拒至少多一轮。**【原件】
   - 时长不带单位：`a6859c1` 的 10 次 S1 运行**全部**中招，不是"五次"。模型写的值一半按秒（30、60），一半按毫秒（20000、30000、60000）。
   - 按键大写 `ENTER`：B2 的 5 次里有 2 次，整批被拒，每次多 3 轮。模型被拒前读过 `key --help`。
   - `@e…`：1 次。根因其实是 PowerShell 把 `@e674098770` 当作 splatting 吃掉了，CLI 收到的只是一个光秃秃的 `click`。
   - `reveal` 被拒 8 次、`open --package` 被拒 2 次，报错文字都会误导模型。
   - 候选运行里所有"被拒后重发"合计约 37 轮，约占 929 轮的 4%。
5. **"只改文档、不改解析器"的做法，在按键大小写、`@` 前缀、`open --package` 这三处属于钻牛角尖。**这几处都可以宽容接受，而且没有歧义。时长是唯一需要权衡的一处：可以用"按选项量程判定"的规则无歧义地接受，本轮观察到的 10 个真实值全部能被正确解释。
6. **首页不足以让新模型独立完成"打开 App、看、点、输、等、读结果"这个基本循环**：首页没有 `open`、`back`、`scroll`、`key` 的示例，反而写了不少证明性措辞。专题文档里大段是实现内部的机制，估计能从 2,419 行减到约 900–1,100 行。

本专题不直接涉及两道待决题（后台屏的搬动归因、M6 坐标）。

---

## 2. 确定缺陷

下面按对调用成本的影响排序。

### 必须修

**D1. `open --package PKG` 单独使用时被拒，而且报错让模型照着已经做过的事再做一遍。**【原件】【源码】
- 触发：V1 任务 5 次里有 2 次，见 `ST/efficiency/v1-candidate-02`、`v1-candidate-05` 的第一次 `open`。
- 回执原文是 `open: not sent; No concrete handler resolved; specify --component or --package`，可模型已经给了 `--package`。
- 原因：位置参数 `open PKG` 走启动器入口，而 `--package` 只是"把一个类型化 intent 限定在这个包里"（`ui_commands.py:1065-1129`；`actions.md:345-349`）。两个写法看起来等价，语义完全不同。
- 影响：每次多 1 轮（约 27k 输入）。并且被拒回执里附了 7 个候选组件的长 JSON。
- 最小修正：没有 `--action`、`--data`、`--component` 等任何 intent 字段、只有 `--package` 时，按 `open PKG` 处理。至少要改报错，直接写出 "use `open PKG` to launch the app"。

**D2. `reveal` 在目标不支持 SHOW_ON_SCREEN 时，报错写的是 `Requested action ID is not currently advertised`，首页却要求"点击屏外目标前先 reveal"。**【原件】
- 触发：Feeder 长列表，S1 系列一共 8 次（`s1-core-candidate-01..05`、`s1-with-05`、`s1-without-02`、`s1-without-05`）。
- 模型随后改用 `click`，直接成功了（如 `s1-core-candidate-01` 紧接着 `click e876009345` 得到 done）。也就是说，首页那条建议在这个 App 上平白多花一轮。
- 合计约 12 轮、约 42 万输入 token。
- 最小修正：报错改成说清"该节点不支持 reveal，可直接 click，或用 `scroll CONTAINER DIR`"。首页那句改为"click 失败再 reveal 或 scroll"。

**D3. `scroll-until` 的失败首因写错，且只支持精确匹配，在本轮主场景上第一次调用 0/10 成功。**【原件】
- S1 的 10 次首发（`a6859c1`）参数修正以后仍全部 exit 1。典型回执（`s1-with-03`）：完成了 3 步，然后第 4 步发现"已经没有 scroll-down 路线"，也就是滚到底了；`matches=0`。
- 但首因写的是 `Original container admission failed`，读起来像是容器本身有问题。
- 根本原因是 `--text` 要求完全相等，而 Feeder 行的名字是"标题换行摘要"的合并文本。实施方在 core-results 里也确认了这一点。
- 影响：每次运行在这里多 1–3 轮，并且是 `reveal` 失败（D2）的前因。
- 最小修正分两步：
  1. 失败汇总按语义写成 `reached end after 3 steps; no node matched --text`。
  2. 给 `scroll-until` 加上和 `wait`、`find` 同义的 `--contains`（可见文本子串）。这是能力改动，列在第 6 节 A/B。

### 可选（便宜，但收益明确）

**D4. 按键名大小写敏感，且报错不列出合法值；指引自己就用大写。**【原件】【源码】【实跑】
- 触发：`b2-with-01`、`b2-with-02` 的 batch 第 2 步写的是 `ENTER`，整批 `not sent`，每次多 3 轮、约 8.5 万输入。
- 合法键是固定的小写集合（`input_keys.py:7-25`），报错只说 `Use one supported editing key...`。
- `images.md:202` 写的是 "fill's ENTER fallback"，`actions.md:319` 写的是 "Enter down/up"，`cab84b8` 还没改。
- 我实跑 `key Enter`、`key Ctrl+A`、`key ctrl+A`，三个都返回 exit 2。
- 最小修正：命名键和修饰键按大小写不敏感处理（这是封闭集合，没有歧义）；单字母键只在有修饰键时才归一化。报错附上合法键表。

**D5. 节点前面显示的 `@` 在 PowerShell 里会被当作 splatting。**【原件】
- `c1-candidate-01` 第 3 轮的命令是 `click @e674098770`，CLI 收到的是不带参数的 `click`（报 `Provide a ref...`）。
- 第 4 轮模型改成加引号的 `'@e674098770'`，CLI 却报 `staleRef: Ref is absent from the current observation; ... run snapshot`。这条报错有误导：那个 ref 其实在当前观察里。
- 之后模型又读了 `actions.md`（7.6k token），重新 snapshot，第 6 轮用裸 `e674098770` 才成功。合计 3 轮。
- `a6859c1` 在首页加了一句说明后，其后 49 次候选运行没有再出现这个错误，说明文档修正有效。
- 剩余问题有两个。一是 `@` 在 PowerShell 下仍是陷阱。二是我实跑 `click @e15` 时，主机原样把 `@e15` 发给设备，没有在本地拦下来。
- 最小修正：主机解析参数时去掉开头的 `@`（ref 本身不会以 `@` 开头，所以零风险）。显示标记是否换掉，列在第 6 节。

**D6. 观察里显示 `scroll="forward down page-down"`，`scroll REF page-down` 却被拒。**【实跑】
- 实跑结果是 `Direction must be forward|backward|up|down|left|right`。首页又要求"按容器的 `scroll` 事实选方向"。
- 本轮原件里没见到模型踩这个坑，但这是一个显然的陷阱。
- 修正：接受 `page-*` 作为对应方向的别名，或者显示时不打印 `page-*`。

**D7. 时长不带单位被拒（详见第 3、4 节）。**
- 报错 `Use an integer with ms, s, m or h; only 0 may omit the unit` 没有给出可以直接照抄的改法。

**D8. 小的不一致。**【实跑】
- 设备不存在、根本没有发出任何东西时，动作回执里也写 `dispatchStarted=unknown`。这符合 `reference.md` 的规则，但会误导。
- 文本首行的命令路径，有的用点（`notifications.list`），有的用空格（`session close`），JSON 的 `command` 字段同样如此。`reference.md` 说命令路径是固定的，举的例子是 `job get`。

---

## 3. 设计与简化

### 3.1 默认观察格式

【源码】`ui_view.py:728-750`：每个节点一行，格式是 `角色 @REF "名字" 键=JSON值 …`，取值为 true 的属性只写键名，按层级缩进一个空格。我对比了真实输出：

- 原生页已经相当干净，例如 Settings 页的一行 `item @e746333903 "Yuxuan Li" clickable detail="Google 服务和偏好设置" rowIndex=1 columnIndex=0`。
- 网页仍然啰嗦，例如 GitHub 页的一行 `item @e582994549 "Homepage" clickable action={"id":16908348,"name":"context-click"} roleDescription="link" targetUrl="https://github.com/"`。

在 651 份候选默认 stdout 里（排除 `--all` 和 `--raw`，共 2.80 MB），按字节看（`scripts/composition.py`、`scripts/lean_estimate.py`）：

| 成分 | 占比 | 调用者用得上吗 |
| --- | ---: | --- |
| 网页上每个节点都带的 `context-click` 动作 | 6.8% | 几乎不用。Chrome 的每个节点都声明这个动作；确实需要时可以用 `action --action-name context-click`。`ba5d456` 的 review 修正专门把它恢复显示，属于为了"不丢事实"而保留的低价值字段。 |
| 每次观察开头的溯源 JSON 行（capture UUID、connection、sequence、finishedMs、scope） | 8.4% | 只有 UNKNOWN 后的"资源释放核对"流程会用到（`reference.md:95-131`）。默认文本可以只留 `seq` 和 age，partial 时再展开。 |
| `window … selected` 行和 `edge-windows` 行 | 约 2% | 单窗口时几乎没用。 |
| `rowIndex`、`columnIndex` 等行列信息 | 2.4% | 极少用到。 |
| `chromeRole` | 3.1% | 当 `roleDescription` 已经存在、或者能从角色推出时，属于冗余。 |
| 自定义动作写成 `{"id":2131296296,"name":"action-7f090028","label":"…"}` | 约 5.8% | `action --action-label` 本来就可以按标签调用，显示 `+"Open in reader"` 就够了。Feeder 每一行重复 8 个。 |
| 绝对路径 `snapshotPath="D:\\…"`（中位 238 字节，占首行的一半以上） | 5.1% | 这部分依赖实验环境，因为评测的工作区路径特别长。改成相对工作区的短路径就够。 |
| 10 位数字 ref | 4.6% | 改成会话内的短编号（`e15`）能再省约 2%，但这会动到设备端的身份设计，属于可选。 |

依次应用上面这些转换以后，累计降到原来的 64.8%，**不丢任何节点、名字、值、状态或可操作性**。这是离线估算，没有在设备上验证（【重算】`lean_estimate.py`）。

**与 Playwright aria snapshot 的比较**【推断】：Playwright 一个节点的写法类似 `- link "Homepage" [ref=e5]`，链接地址另起一行写 `/url:`。它有几个特点：
- 用标准 ARIA 角色，链接就是 `link`，所以不需要 `roleDescription` 和 `chromeRole`；
- 不列动作，角色本身已经说明能怎么操作；
- ref 很短；
- 没有每屏的溯源头。

AndroidCtl 的角色只有十个通用词（`item`、`text`…），网页语义只能靠 `roleDescription`/`chromeRole` 补，所以每个节点都多出两个属性。**把网页角色映射到更具体的角色词（link、heading），能同时去掉这两个属性**，这比继续删字段更治本。Playwright MCP 每次动作后同样返回整页快照（据我所知是默认行为，本轮没有核实最新版本），所以"动作后附带整页"本身不算异常。

### 3.2 动作回执附带整页

【重算】候选 stdout 的构成：动作回执 41%，`snapshot --all` 39%，普通观察 20%，被拒回执 0.2%。在 333 次有正文的成功动作里，有 110 次报告 `change=unsettled … Estimated next sample exceeds the observation deadline`。这 110 次之后，模型紧接着重新 `snapshot` 了 37 次（34%）；动作已经稳定的 223 次之后只有 16 次（7%）。

可见对模型来说，"页面还没稳定"的回执里那一整页内容不可信，它会再取一次。可以考虑：回执是 unsettled 时不附整页，只给一行"页面仍在变化，用 wait/snapshot"。这样省字节，也避免模型读到过渡中的页面。是否这样做属于产品取舍，列入第 6 节。

### 3.3 `snapshot --all` 是隐形的大头

【原件】`--all` 一共调用了 32 次。其中 8 次在网页上产出 255 KB 左右的输出，约 64k token，执行环境把它截到约 2 万 token 再给模型；另有 2 次在 Maps 上约 43 KB，约 1.1 万 token。

截断后模型只好再用 `Select-String` 去搜保存下来的文件，候选运行里这类文件读取一共 14 次。`--all` 同时打开三件事：不可见节点、全部标准动作（每个节点都带 `select`、`clear-selection`、`{"id":64}` 等）、额外字段。

建议：`--all` 默认不打印标准动作，或者给 `--all` 加一个默认的字符上限，超出时写一条省略记录。

### 3.4 公开指引

【源码】`cab84b8` 的 `skills/androidctl/` 一共 12 份、2,419 行、160 KB：首页 100 行，`observations.md` 492 行，`actions.md` 452 行，`connection.md` 308 行，`images.md` 275 行，`reference.md` 213 行，其余较短。

**首页够不够用**：首页有开 session、`find`、`click`、`wait`、`snapshot --cached`，也提到了 `fill`。下面这些基本循环要用的东西都没有示例：
- 打开 App（`open PKG` 和 `apps`），D1 就是因此踩坑；
- `back`；
- `scroll REF DIR`；
- `key enter`；
- 读一个结果。

同时，首页 100 行里约有 25 行是证明性或边界措辞，例如：
- "Partial sampling can establish positive facts but cannot prove absence or selector uniqueness"
- "Idle, an absent loading indicator or a complete sample do not establish business readiness"
- "establish non-occurrence and operation end before new input"
- "Same-name callers share a lease, including unnamed `default`"

结果是 45/54 的运行都要再读 `actions.md`（7.6k token）。它开头是一段 40 行的 "Ref Identity And Repair"，讲三份历史样本、重新发布不老化之类的内部机制，调用者照着做不了任何事。

**专题文档里可以删掉或移到 docs 仓库的内容**（只列典型）：
- `actions.md`："Ref Identity And Repair" 全段，可以压成 3 行："ref 只对最近的观察有效；被拒时重新 snapshot"。scroll 的映射细节、`completedSteps` 的语义、诊断截断等，调用者都不需要。
- `observations.md`：capture 的发布与读取、工作区权限修复、`--print`/`--json` 组合规则、"可见性只是无障碍报告"之类的大段说明。
- `reference.md`："UNKNOWN And Resource Release" 里要求核对 capture UUID 前缀、`snapshotCached=false` 的整段流程；各种退出码的覆盖规则；"Measured JSON fields"；8192 节点、8 MiB、192 KiB 这类上限。
- `connection.md`（308 行，零次被读）、`images.md` 中关于坐标映射和输入接收方的证明细节。

粗略估计（【推断】，按段落逐份估）：首页保持在 100–130 行，同时补上上面缺的示例、删掉证明句；`actions.md` 452 → 约 150 行；`observations.md` 492 → 约 150 行；`reference.md` 213 → 约 80 行；`images.md` 275 → 约 120 行；`connection.md` 308 → 约 120 行；其余小幅压缩。**合计约 2,419 → 900–1,100 行（减少 55–60%）。** 只看实际被读到的部分，如果 `actions.md` 压到三分之一，候选总输入大约能降 8%，这和前面全部呈现压缩带来的收益（CLI 输出驻留占比 17.9%）是同一量级。

### 3.5 第四轮决定和建议中现在看需要调整的地方

- **第 8 项的完成标准"真实样本重放后不超过原来的一半"量的对象太窄。** 它只量观察正文的字节，没有覆盖三样东西：动作回执的首行（中位 406 字节）、指引阅读（驻留占比 21%）、被拒造成的额外轮次（约 4%）。按这个标准，实施方在呈现代码和逐节点事实审计上做了大量工作（`ST/reading-replay/` 下有 item1–4 的四轮日志和审计）。而占比更大的指引长度，以及几处一行就能修好的宽容规则，被以"不加兼容"为理由放下了。下一轮的度量应当改成"每次任务的模型输入 token 和轮数"，并且拆出指引、输出、被拒三项。
- **"不丢事实"被执行得过于字面。** `ba5d456` 为了不丢 context-click 这条事实，在每个网页节点上多写 45 字节。默认文本的原则应当是"不丢调用者可用的能力"：能通过 `--all` 或 `--raw` 读到的低价值事实，不需要在默认文本里出现。

---

## 4. 证据审计

| 实施方声称 | 核对结果 |
| --- | --- |
| 三份重放分别为 46.55%、49.25%、28.72%（`ST/reading-replay/result.md`） | **成立**【重算】。我用 `4235af9` 和 `cab84b8` 两版真实的 `render_observation` 重放同一批原件，得到 45.9%、48.9%、28.0%。差异来自我用了较短的假 source 字段。**有边界**：包内另一份真实网页样本 `web/006` 是 53.4%；WPS 系列的另外 4 份是 27.6%–41.7%。合计 43.1%。"每份都不超过一半"在更宽的样本集上不成立，不过超出很小。 |
| 精简发生在模型真正读到的内容上 | **部分成立，不能精确对照。**【原件】第四轮候选臂（16-delivery 的 new 臂，用的是 JSON 对象行格式，带 `view_file.py`、`ui_diff.py`，接近 `4235af9`；具体提交在本克隆里解析不到）在 GitHub 任务上：snapshot 中位 18.4 KB，单次 `find` 231.8 KB，每次 CLI 调用约 3.4k 模型可见 token（37.1k / 11 次）。本轮 C2 候选：snapshot 中位 8.4 KB，`find` 9.1 KB，每次调用约 1.9k token。但两边设备不同（1080×1920 对 720×1280），屏内节点数也不同，只能说明方向。和基线 `7d5be70` 比没有意义：基线在 Chrome 上根本不采网页内容（`c2-baseline-01` 的 snapshot 只有 14 行，"Web View" 下面是空的），所以基线的观察反而更小（中位 2.4 KB 对候选 8.4 KB）。 |
| "五个全新模型都先传了无单位数值"（`ST/README.md:49`） | **低估了。**【原件】`a6859c1` 的 S1 两组（`s1-core-candidate-01..05`、`s1-with-01..05`）**10/10** 都先被拒。改文档以后只有 `b2-with-01` 一次运行用到 `--timeout`（60s、30s，都带了单位），n=1，不能证明修正有效。 |
| "两个模型读过 key 帮助仍用 ENTER/Enter" | **成立。**【原件】`b2-with-01` 第 3 轮、`b2-with-02` 第 2 轮先调了 `key --help`，下一轮 batch 才写 `ENTER`。当时的帮助没有小写示例；`cab84b8` 补上了，但 `cab84b8` 主机下的 10 次运行没有用到 key，所以效果没有验证。 |
| `@e15` 的根因是模型把显示标记带进了参数 | **部分成立。** 第一次失败实际上是 PowerShell splatting，见 D5。 |
| `a873616` "align scroll guidance" | 这个提交只删了已经没有消费者的 "diff options" 等文字（5 行），**不是针对误用的修正**。本轮真正的 scroll 问题（D2、D3）没有被处理。 |
| 第 8 项"lostFacts 为空" | **未逐节点复核**，只抽查了 context-click、rowIndex、targetUrl 都保留着。 |
| 失败与例外是否如实保留 | **成立。**【原件】`core-results.md` 把所有 exit 2 和 exit 1 的回执都列了出来，算成本时也包含了它们。但它们只是逐条列出，没有汇总成"误用类别 × 次数 × 浪费轮数"，所以"只改文档"的决策缺少量化依据。 |

---

## 5. 关于常见误用：按类别的判断

站在"调用者是 LLM，每次被拒就多一轮（本轮一轮约 3 万输入 token）"的立场，按"宽容接受 / 改报错 / 只改文档"给出判断。证据全部是【原件】，统计脚本是 `scripts/waste.py`。

| 误用 | 次数（候选） | 每次浪费 | 判断 | 接受规则与风险 |
| --- | ---: | --- | --- | --- |
| 时长不带单位 | 10（S1 全部） | 1 轮，约 3 万输入 | **宽容接受，并改报错** | 只针对 `--timeout`（量程 1ms–60s）：1–60 的裸整数按秒，1000–60000 按毫秒，61–999 继续拒绝，报错里写出 "use 500ms or …"。按这个规则，本轮 10 个真实值（30、60、20000、30000、60000）全部会被正确解释。风险是有人真想用 1–60 毫秒的 UI 预算，但这种情况实际不存在。`--settle`（0–60s）用同样的规则；手势的 `--duration` 只改报错。 |
| 按键大小写 | 2/5 批次 | 3 轮，约 8.5 万输入，整批不执行 | **宽容接受** | 命名键和修饰键不区分大小写。单字母只在有修饰键时归一化。无歧义。 |
| `@e15` 前缀 | 1 | 3 轮，外加读一遍 actions.md | **宽容接受，并改显示**（第 6 节） | 去掉开头的 `@`，零风险。光靠接受规则救不了 PowerShell 的 splatting，所以显示标记要改。 |
| `open --package` 单独使用 | 2/5 | 1 轮 | **宽容接受，并改报错** | 没有其他 intent 字段时等同于 `open PKG`，无歧义。 |
| `reveal` 目标不支持 | 8 | 1–2 轮，共约 12 轮 | **改报错和首页** | 不建议让 reveal 自动退回去滚动，那等于偷偷换了一个动作。 |
| `scroll` 方向不可用 | 2 | 1 轮 | 改报错就够 | 现在的报错已经列出当前可用方向，质量不错。 |
| 精确 `--text` 匹配不上合并名字 | S1 全部 | 1–3 轮 | **加 `--contains`**（第 6 节） | 子串匹配的唯一性已经有 `wait --contains` 现成的规则可以沿用。 |
| unsettled 以后重新 snapshot | 37 次 | 通常与其他调用同轮，约 0.5 轮 | 见 3.2 | — |

**对"不改解析器、不加兼容"这个立场的评价：** 对按键大小写、`@` 前缀、`open --package` 这三项坚持不改，属于钻牛角尖。它们都是封闭集合或者不会产生歧义的写法，宽容接受只要几行代码，不需要新机制，也不会让调用者变得不可预期。拒绝它们换来的是"每份指引多一句说明，每次被拒多一轮"。时长这一项，实施方的谨慎有一定道理（30 按秒还是按毫秒确实会被误解），但用量程判定规则可以做到无歧义。至少报错里应当给出可以直接照抄的写法。文档修正对 `@` 是有效的，但每加一句说明，所有调用者都要为它付阅读 token；被拒的调用者为宽容规则付出的代价则是零。

---

## 6. 需要用户裁决的新取舍

**Q1. 参数宽容政策**
- A. 维持现状：严格解析，只改文档。
- B. 严格解析，但所有参数错误都给出可以直接照抄的改法。
- C. 在 B 的基础上，宽容接受封闭集合里的写法：按键大小写、`@` 前缀、`open --package` 单独使用、`page-*` 方向。
- D. 在 C 的基础上，按量程接受裸时长。

推荐 **D**（至少 C）。依据是第 5 节：约 37 轮被拒里，有 16 轮是 C/D 可以直接消掉的。剩余未知：宽容接受以后，模型会不会在别处对 CLI 的严格程度产生错误预期。这一点没有证据，但风险很低。

**Q2. 指引结构**
- A. 维持现状：首页 100 行，专题 2,300 行。
- B. 只裁剪专题，把内部语义移到 docs 仓库，总量降到约 1,000 行。
- C. 在 B 的基础上，把基本循环要用的命令（`open`、`back`、`scroll`、`key`、`fill`，以及读结果）写进首页，让大多数任务不必再打开 `actions.md`。

推荐 **C**。依据：`actions.md` 一份就占候选总输入的 11.8%，被读了 45/54 次；`observations.md`、`connection.md` 零次被读。代价是维护者要接受"公开文档不完备是可以的，细节放在 docs 仓库"。剩余未知：裁剪以后，少见能力（例如 range、自定义动作）的首次使用成功率需要用一两次任务复测。

**Q3. 默认文本进一步精简**（第 3.1 节那张表）
- A. 不再改。
- B. 只做零风险的几项：去掉 context-click、溯源行改成短行、去掉单窗口的 window 行和 edge-windows 行、自定义动作只写标签、路径改成相对路径。
- C. 在 B 的基础上，网页改用具体角色（link、heading），去掉 `chromeRole`/`roleDescription`。
- D. 在 C 的基础上，改用会话内的短 ref，这需要动设备端。

推荐 **B**，C 可以作为下一步。依据是离线估算 B 约省 30%、C 再省 3–5%；D 牵动设备端的身份设计，收益只有约 2%，不值得。

**Q4. ref 的显示标记**
- A. 保持 `@e15`，同时由主机去掉 `@`。
- B. 改成 `[e15]`，或者干脆不加标记。

推荐 **B**。依据是 PowerShell splatting 这个真实案例，而且首页本身就在指导 PowerShell 用户。代价是测试和指引里的示例要一起改。

**Q5. unsettled 的动作是否附带整页**
- A. 照旧附带。
- B. 不附带整页，只附一句"页面仍在变化"，加上已知的变化摘要。

暂不推荐 B，需要先补证据：这 37 次重新 snapshot 里，有多少次是模型确实需要新页面，有多少次是信不过旧页面。目前的数据不足以裁决。

**Q6. `scroll-until --contains`**
- A. 不加。
- B. 加，语义沿用 `wait --contains`（可见文本子串），唯一性规则也沿用。

推荐 **B**。依据：S1 首发 0/10 成功，`find` 和 `wait` 都已经有子串语义，调用者很自然会以为 `scroll-until` 也有。

---

## 7. 没能核实的内容

- 没有设备，所有"再省 X%"都是对真实 stdout 做文本转换的离线估算。网页角色映射（Q3 的 C）需要设备端的角色数据才能确认可行。
- 第四轮候选臂的确切提交号在本克隆里解析不到，只能从格式和模块推断它接近 `4235af9`。和本轮的真实输出比较受设备分辨率影响，不能当成精确比值。
- `cab84b8` 和 `7693794` 的文档修正（按键小写示例、时长示例）在真实运行里几乎没有被检验：key 没有用到，时长只有 1 次运行用到。
- 没有逐节点复核实施方的 lostFacts 审计。
- "驻留占比"按"之后每一轮请求都完整重读"计算，前提是执行环境不压缩历史。本轮原件里没有看到压缩事件，但没有逐条确认。另外，缓存 token 的价格远低于未缓存 token，所以按费用算，指引的占比会比按 token 算的 21% 低。
- 关于 Playwright 的比较来自我对它公开格式的了解，没有联网核对最新版本。
- 我实跑了约 60 条命令的本地解析，覆盖首页和专题文档里的选项名、默认值、必填参数、退出码、示例命令。除第 2 节列出的不一致之外，其余都与文档一致。其中实跑 `pair 192.168.1.20:18843 …` 时 CLI 尝试连接了这个私有地址，连接被拒，没有其他影响。
