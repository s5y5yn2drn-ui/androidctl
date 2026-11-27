# 第八轮专题 125：公开指引、命令行帮助和独立 docs

对象：产品 `409e853` 的 `skills/androidctl/*.md`（11 个文件，78,355 字节），`S/r8/venv8` 里实际安装的命令行的 `--help` 和参数报错，独立 docs 仓库 `090b1a2`。对照版本是上一轮候选 `f401deb` / `dfe5e45`。
工作目录 `S/agents14/125/`（`S` 是评审暂存根目录）。其中 `help/`（63 页 `--help` 原文）、`reads.json`（48 次尝试逐次读了哪些指引、在第几次工具调用读、读之前有没有出过错）、`new07.txt` / `new07r.txt`（新十题第 7 题两次尝试的可读化记录）、`dump.py`、`dump2.py`、`reads.py`、`batch.py`、`idents.py`（自写脚本，都用 `python3 -I` 运行，包内路径作为参数传入）。
说明："首页"指 `SKILL.md`；"专题"指另外 10 个 `.md`；"后台屏"指 AndroidCtl 创建的虚拟显示；"尝试"指一次正式实测（`EV/efficiency/r7-<组>-<n>/`）。38 次候选尝试用的是 `82f3a82` 的指引，与 `409e853` 只差首页 UNKNOWN 一节的最后一句。

---

## 结论（先看这里）

1. **第 15 项（后台屏要点并进首页、专题改为出问题时再读）只做到了一半。**（实测）8 行要点确实进了首页，内容也够用。但后台屏题 5/5 仍然在第 2–3 次工具调用、还没出任何错的时候整篇读了 `displays.md`。原因很可能是首页还有两处链接邀请去读它（`SKILL.md:33` "see displays for details"，`:128` "displays for display/task safeguards"）。连带读 `reference.md` 的情况从上一轮的 4/5 降到 0/5，所以指引占这道题输入的比例从约 16% 降到约 11%。
2. **新句 "If a retry is also UNKNOWN, stop retrying that action." 方向对，旧句确实让模型提前停了工。**（实测）第 7 题两次尝试的最终答复都原话引用了 "Stop after another UNKNOWN."，并以此为理由停止输入。两次的第二个 UNKNOWN 都是另一个动作，前一个 UNKNOWN 也已经用后续观察核清了。新句有两处缺口：一是"that action"可以读成"同一条命令"，换个输入通道再做同一件业务改动不算"重试"；二是只说停止，没说停下之后该做什么。这句话没有实测过，只能算消歧，不能写成已修好第 7 题。
3. **首页一句话和实际做法冲突，3/38 次在出错之后照样执行了同一批里后面的输入。**（实测）首页 `:39` 是 "Read each result and observation before the next command; known steps may run sequentially."。第 7 题两次都把 `check` 和 `recents` 放在同一次调用里，`check` 已经是 UNKNOWN，`recents` 照样执行了。第 2 题是 `click` 已经失败，后面的 `back` 和 `click` 照样执行了。建议改成"同一次调用里只连发互不依赖结果的命令；某条命令退出码非零就停下后面的命令"。
4. **第 14 项的第三处提示没改。**（源码）首页 `:91` 仍是 "Read/search that file after host-tool truncation."。超限摘要的另外两处已经按裁决改成"搜索，不要整份读"（`ui_client.py:926-941`），首页这一句漏改了。这轮 48 次尝试没有触发过摘要，所以没有造成实际影响。
5. **指引涨的 6.0 KB 里约四分之三是给少见情形写的字段语义。** 其中 `observations.md` 涨 2.2 KB，`actions.md` 涨 1.9 KB，`connection.md` 和 `displays.md` 各涨约 0.8 KB。这些字段在 38 次候选尝试的命令输出里出现 0–2 次的占大半（第 4 节）。第 4 节给了一份逐条删减清单，每条附风险：首页约可再减 12 行，专题约可再减 3 KB，都不碰安全规则。
6. **准确性整体好。** 逐条核过的说法里只找到 4 处小错（第 1.2 节），没有一处会让调用者重复输入或丢掉用户数据。上一轮查出的帮助和报错问题多数已修（第 1.4 节）。
7. **独立 docs 从 301,632 字节减到 149,809 字节，达到裁决要求的 150 KB 以内。** 22 个案例的原文逐字节相同（52,478 字节，SHA-256 与实施方给的一致），六条发布前真机检查也逐行相同。抽查的源码数字全部对得上。但"删除重复说明"只说得上 docs 内部去了重：`reference/ui.md` 有相当一部分是把公开指引逐段译成压缩中文，成了与源码、指引并列的第三份说明（第 6 节）。

---

## 1. 准确性

### 1.1 核对过、与实现一致的说法（抽查）

| 指引说法（位置） | 依据 |
| --- | --- |
| REF 只能是 `e` 加 ASCII 数字，可带一个 `@`（`actions.md:11`） | 主机端实际运行：`click E15`、`click @@e15`、`click ref15`、`click @e15x` 都在参数阶段被拒，报 `REF must be e followed by ASCII digits…`，退出码 2 |
| 自动重绑的两个阈值：手动连上的常驻进程活过 5 秒、自动连上的活过 60 秒才再重绑（`connection.md:112-116`） | `WorkerConnection.kt:331-332`（`MIN_CONNECTED_MS = 5000L`、`MIN_AUTOMATIC_CONNECTED_MS = 60000L`），`:115-122` |
| 首行 `timing=ref-repair` 和 "Rejected REF was not verified" 只在 `freshRepair=true` 时出现（`SKILL.md:83-84`、`actions.md:22-25`） | `ui_output.py:287-289` |
| find 首行的 `fresh` 只在真读到新树时出现；JSON 里的 `findRefreshed` 记录的是"尝试过"（`observations.md:13-14`） | `ui_output.py:139-145`，`ui_client.py:646-654` |
| `target still offscreen`（`actions.md:117`） | `ui_output.py:305-308` |
| 状态文本打印 `callerSession=…`，JSON 字段叫 `callerState`，取值 active/released/expired/none（`displays.md:177`） | `app.py:180-181`，`Sessions.kt:380-386` |
| 成功关闭后台屏的文本是一行摘要加六类任务计数（`displays.md:259-262`） | 实测 `r7-v1-candidate-01` 关闭回执：`taskDisposition=state=clear; displayId=2; destroyed=true; openedTasks=1; movedTasks=0; …` |
| `locate --action scroll-down` 是合法写法（`actions.md:79`） | `help/locate.txt` 列有 `scroll-down` |
| 已核实的 `fill … --submit` 报错现在会回显调用者自己的引用；`wait` 的两种组合错误分开说明；`snapshot --cached --in com.x` 回显 `--in com.x`；`notifications get -1|a` 提示加 `--`；直接敲 `session` 会列出子命令 | 主机端实际运行，原文见第 1.4 节 |
| 指引里出现的 camelCase 标识符（字段名、原因码）在源码或码表里都存在 | 自写 `idents.py`：唯一的"缺失"是 `macOS` |
| 超限摘要里按整页、find、按引用读分开给出读法，并写了"搜索保存的文件，不要整份读" | `ui_client.py:926-941` |
| 后台屏要点（`SKILL.md:26-33`）每一条都与 `displays.md` 和源码一致：创建需要主屏亮屏且已解锁；关闭会销毁屏上内容；搬来的或别的 App 的任务会挡住关闭；来源不明的任务没有丢弃选项也可能被销毁；空闲到期（默认 30 分钟）、常驻进程退出、设备端释放／吊销／Stop 都会直接销毁 | `displays.md:187-190,229-255`；`Sessions.kt` |
| `help skill`（小写）被拒并列出可用主题；`help SKILL` 可用（`reference.md:5`） | 主机端实际运行 |

### 1.2 写错、过时或残缺的

| # | 位置 | 现象 | 依据 | 影响 | 分类／建议 |
| --- | --- | --- | --- | --- | --- |
| A1 | `SKILL.md:91` | "Read/search that file after host-tool truncation." 没按第 14 项改，与摘要本身的"Search the saved file … instead of reading the whole file"语气相反（一个把"读"放在前面，一个明说别整份读） | 决定文件第 14 项落实要点、第六节第 5 处第 3 条；源码 `ui_client.py:939-941` | 低。38 次候选尝试没有一次触发摘要（`grep "Observation exceeds"` 为 0）。主机工具截断时模型读文件用的是 `Get-Content -Tail`（`r7-new-04`），没有整份读 | 确定缺陷（落实遗漏）。改成 "Search that file or narrow with `find`/`--depth`; avoid reading it whole."，代价一行 |
| A2 | `connection.md:101` 对 `displays.md:177` | 前者写状态里分 "callerState active/released/expired/none"，后者写 "status reports callerSession=expired"。文本里是 `callerSession=`，JSON 里是 `callerState`，两处各说对一半，按文本搜索 `callerState` 会搜不到 | `app.py:180-181` | 很小 | 确定缺陷（措辞）。`connection.md` 写成 "`callerSession` (JSON `callerState`)" |
| A3 | `connection.md:135` | 仍写 `DaemonVersion`，码表里是 `daemonVersion`。上一轮 E2 已提过 | `contract/reason-codes.json` 第 38 行 | 很小 | 确定缺陷，一个字母 |
| A4 | `jobs-files.md:67` | 截断的 "Local t..." 改成了 "Local t-job"，但指引别处没有"t-job"这个说法。原意是"本机传输任务的查询" | 文件原文 | 很小 | 写成 "Local transfer-job queries" |

### 1.3 帮助与指引互相矛盾的

没有发现说法相反的地方。剩下的小问题：

- `screenshot --help` 仍列出 `--marks`（`help/screenshot.txt:18-20`），而这个选项总会被拒。上一轮 5.1 第 2 条提过，未处理。
- `find x --raw --all` 的报错建议用 `androidctl snapshot --raw`（新采），而 `observations.md:54-55` 让人用 `snapshot --cached --raw`。实施方记录说这是有意区分（find 要用新样本）。不算错，但两处读起来不一致。
- Click 自己生成的报错仍带 ".;"，例如 `device: not sent; No such command 'default'.; inspect…`。小-52 只处理了产品自己写的三句。
- `help/` 里 63 页帮助合计 51.9 KB，上一轮是 98.7 KB。重复的全局选项已经在叶子命令里隐藏，只留一行 "Global options: androidctl --help"，与 `reference.md:28` 一致。

### 1.4 上一轮报错问题的复核（主机端实际运行）

```
fill: not sent; Use fill REF TEXT or fill TEXT with selector flags; for example, androidctl fill e42 hello …            （回显 e42，已修）
scroll-until: not sent; Missing option '--max-steps'.; for example, androidctl scroll-until e42 down --max-steps 10 …    （已修）
wait: not sent; Provide only one wait condition; use --any-of for alternatives, …                                         （已修）
wait: not sent; --gone requires at least one selector field; for example, androidctl wait --gone --name Loading          （已修）
snapshot: not sent; Take a scoped snapshot first …: androidctl snapshot --in com.x; then androidctl snapshot --cached     （已修）
notifications get: not sent; No such option: -1; put -- before a notification KEY beginning with '-': …                  （已修）
session: not sent; Missing command; choose open, close, takeover; …                                                       （已修）
```

---

## 2. UNKNOWN 最后一句的改动

### 2.1 模型当时是不是因为旧句而停（实测）

**是。**

- `r7-new-07`：`check e241580321` 返回 UNKNOWN（`new07.txt` 第 115 行，"Current window root is unavailable"）。同一次调用里的 `recents` 回执已经显示 ON，之后打开设置页又读到 ON。后面恢复原值的 `uncheck e241580843 --settle 3s` 也是 UNKNOWN（第 185 行），但随命令返回的新树已经是 OFF，之后的只读 `snapshot` 又确认了一次。最终答复（第 170 行附近）原话是："公开 SKILL.md 明确规定 'Stop after another UNKNOWN.'；第二次 UNKNOWN 出现在恢复操作，因此只追加只读恢复核验后停止输入"。没做的是恢复之后的四页截图和回 Home，本题因此判失败。
- `r7-new-07-retry1`：`check` 是 UNKNOWN，但随命令返回的新树已经是 ON（`new07r.txt` 第 66-67 行）。恢复用的 `click` 成功。之后另一个动作 `open com.android.deskclock` 因为通知权限弹窗返回 UNKNOWN（第 168 行）。最终答复写的是"依公开 SKILL 的'Stop after another UNKNOWN'停止后续输入"（第 181 行）。
- 两次的第二个 UNKNOWN 都不是在重试前一个动作，而且前一个动作的业务结果已经用后续观察可靠地读到了。首页本来就写了 "Continue on reliable business success."。模型把整道题里先后出现的、互不相关的 UNKNOWN 累加成了硬性停工。实施方 `r7-cumulative-validation.md:622` 的读法与原始记录一致，我核实过。

### 2.2 新句会不会引入相反的问题（设计意见）

- **不会鼓励"无条件重试一次"。** 新句前面那句 "Otherwise establish non-occurrence and operation end before new input" 仍然管着：只有确认没发生、操作已结束，才能再做输入。"支付、发送、删除要有权威记录"那句也还在（`SKILL.md:121`）。对"结果未知"的动作做重试，前提已经写清楚了。
- **缺口一：以动作为单位还是以业务改动为单位。** "stop retrying that action" 可以读成"同一条命令"。模型可能改用 `click --touch`、坐标 `tap` 或者换一个按钮，去完成同一个业务改动，而这不算字面上的"重试"。前文 "not … another input channel" 讲的是用什么来证明操作已结束，不是禁止换通道重做。建议写成 "If a retry is also UNKNOWN, do not attempt that change again by any input; report it."。
- **缺口二：停下之后做什么。** 新句没有写。旧句至少让模型停下并把情况交回给人。建议补"报告未确定的状态"。上面那句建议写法已经包含了这一点。
- **执行偏差与这句无关，但相邻。** 第 7 题两次都把 `check` 和 `recents` 放在同一次调用里（见结论第 3 条、第 3.3 节）。这是首页"可连发"那句的问题，不是 UNKNOWN 这句的问题。

### 2.3 没有实测，该怎么对待

- 这句话的收益只是"消除一处已观察到两次的误读"，还没有证据表明改后第 7 题能做完。报告里应当写"未经实测的消歧"，不能把 `82f3a82` 那两次的失败说成已修复。
- 风险很低（只收窄了停止的范围，其它前提都在），不值得为它单独重跑整套实测。如果下一轮还有实测，把第 7 题重跑一两次，就能同时检验这句话和上面"出错就停下同批命令"的改法。
- 如果采纳 2.2 的写法，这一行会变长。可以把首页 `:119-120` 并成一行来抵消。

---

## 3. 调用者实际怎么用指引（48 次尝试，`reads.json`）

### 3.1 读了什么、什么时候读（实测）

| 组 | 首页 | 专题（括号里是第几次工具调用；"!"表示读之前已出过非零退出码） |
| --- | --- | --- |
| 路线题候选 5 次 | 5/5，都在第 2 次调用 | 无 |
| 网页题候选 5 次 | 5/5，第 2 次 | 无 |
| 长列表题候选 5 次 | 5/5，第 2 次 | **无**（上一轮 4/5 读了 `actions`） |
| 后台屏题 5 次 | 5/5，第 1–2 次 | `displays` 5/5，第 2–3 次，都在出错之前；`reference` 0/5（上一轮 4/5） |
| 新十题 18 次 | 18/18，第 1–2 次 | `input` 6、`images` 7、`actions` 6、`reference` 4（4 次都在 UNKNOWN 之后，顺着首页 UNKNOWN 一节的链接去读）、`notifications` 1；`--help` 只有 1 次（`r7-new-03` 的 `home --help` 和 `recents --help`） |
| 基线 `7d5be70` 10 次 | 10/10 | 网页题基线 5/5 读了 65 KB 的旧 `reference.md`，指引约占那道题输入的 34–41%（估算） |

- 每次都是用 `Get-Content -Raw` 把整篇读进来，没有一次用搜索去找某一段。
- 读专题的时机分两种。一种是开头按任务类型一次性读完：后台屏读 `displays`，输入类读 `input`，状态切换读 `actions`。另一种是出错之后按链接回头查：UNKNOWN 之后读 `reference`，看图前后读 `images`。第二种正是首页链接所设计的用法。
- 38 次候选尝试里，**`observations`、`connection`、`jobs-files`、`clipboard` 一次都没被读过**，`notifications` 只读过 1 次。这五篇合计约 29 KB。上一轮 20 次里没被读过的专题是 7 篇。

### 3.2 指引占输入的份量（估算）

按"读到的 token 数 × 之后的回合数 ÷ 总输入"粗算（`reads.py`，口径与实施方 `core-results.md` 的 A 估计相近）：

- 只读首页的题：约 5–6%（路线题 4.7–5.6%，网页题 5.3–6.1%，长列表题约 6.0–6.4%）。
- 后台屏题：约 13%（实施方的口径是 11.06%）。上一轮约 16–17%。
- 新十题：3%–21%，读了 `input`、`actions`、`images`、`reference` 的那几次在 15–21%（`r7-new-02`、`-06`）。
- 首页本身约 1.8k token（7,031 字节）。

### 3.3 指引导致的低效或错误动作

- **连发命令遇到失败不停，3/38。**（实测，`batch.py`）`r7-new-07` 第 11 次调用：`check` UNKNOWN 之后照样执行了 `recents`。`r7-new-07-retry1` 第 8 次调用：同样的情形。`r7-new-02` 第 6 次调用：`click` 已失败（"Window root accessibility read is still pending"），之后照样执行了 `back` 和 `click --name Search`。119 次多命令调用里还有 3 次出了错，但后面只跟只读命令，没有害处。首页 `:39` 把"逐条读结果"和"可以连发"写在同一行，模型把后半句当成了许可。→ 设计取舍，建议改写成"同一次调用里只连发互不依赖结果的命令；某条命令退出码非零就停下后面的命令"。代价一行；验证可以靠下一轮实测里多命令调用的统计。
- **旧 UNKNOWN 句造成提前停工，2/18**，见第 2 节。
- 没有发现模型因为照抄某条指引示例而做错事。

---

## 4. 指引的增长与删减

### 4.1 本轮涨了什么（`f401deb` 72,311 → `409e853` 78,355 字节，+6,044）

| 文件 | 增量 | 新增内容 | 在 38 次候选尝试的命令输出里出现过几次（`evidence/cli/*/child.stdout`） |
| --- | ---: | --- | --- |
| `observations.md` | +2,228 | REF 视图默认／屏外规则；`fresh` 与 `findRefreshed` 的区别；缓存树读失败；摘要带字节／行数；`mainLocked`/`mainInteractive`/`displayOff` 4 行；`missingRecords`/`incompleteParents` 3 行；动作后首树采样与 `rootUnavailableObserved`/`rootRecoveredWithinRequest` 8 行；wait 的 `collectionCutoff`、`cutoffStage`、any-of complete 7 行 | `mainLocked`/`displayOff` 0 次；`cutoffStage` 0 次；`fresh` 2 次；`rootUnavailableObserved=true` 23 次（14 次尝试）；`collectionCutoff` 25 次（9 次尝试）；`missingRecords` 142 次（20 次尝试） |
| `actions.md` | +1,932 | `freshRepair` 4 行；节点／字节上限下唯一性的例外 3 行；`covered` 两种含义 4 行；被拒之后怎么读 5 行；形状映射的单行多列例外 3 行；`reveal` 和最近可点祖先 7 行；启动确认之后的观察 3 行；触摸取窗口交集；"done 但退出码非零看清理原因" | `timing=ref-repair` 4 次；`covered` 0 次；`target still offscreen` 1 次；最近祖先建议 2 次；`routeUnavailable` 1 次尝试 |
| `connection.md` | +827 | `connectionBusy` 的剩余预算 2 行；status 字段；证书时钟／TLS 身份 3 行；重绑规则从 6 行扩到 10 行（加了 60 秒阈值、手动与自动之分、Shizuku-ready 事件） | `busy`/`connectionBusy`/`sessionRequired` 0 次 |
| `displays.md` | +787 | `previousSession`；确认落地之后的再检查 3 行；关闭摘要格式 4 行 | `previousSession` 0 次；关闭摘要 5 次（每次正常关闭都有） |
| `images.md` | +327 | 隐藏目标的 REF 区域报 notReady；`freshRepair` 交付一行；历史清理诊断 2 行 | 0 次 |
| `input.md` | +199 | 剪贴板残留需要征得同意再清空（第 18 项要求） | 0 次 |
| `SKILL.md` | -4 | 后台屏 8 行进来；删了 `find` 重采三行、版本前提一行；专题列表从 10 行压到 4 行 | — |
| `reference.md` | -253 | JSON 字段导读删掉四组 | — |

结论：裁决明文要求写进指引的只有剪贴板一句（第 18 项）、`--all` 定义（第 14 项）、后台屏要点（第 15 项）、`covered` 分开说（小-67）、停止原因补"没有路线"（下-20）。其余约 4.5 KB 是实施方为新加的字段和边界逐条补的说明，大多对应 0–2 次的少见情形。

### 4.2 首页：只留"完成一个普通任务必须知道的"（逐条）

| 行 | 内容 | 建议 | 风险 |
| --- | --- | --- | --- |
| 21-24 | 同名共享租约，`busy`/`connectionBusy`，接管，`busy subject=automation` | 压成 1 行："Respect `busy`/`connectionBusy`; take over only on user instruction." `automation` 那半句删掉 | 低。报错原文（`AutomationCleanup.kt:39`）和码表里已有完整做法。38 次里 0 次触发 |
| 33 | "`status` reads idle time without renewal; see displays for details" | 删掉 "see displays for details" 半句 | 低。这是 5/5 预读 `displays` 的可疑原因之一；排障时还能从 `:128` 找到 |
| 128 | "displays for display/task safeguards" | 改成 "displays when a work display blocks or fails" | 低。用词从"安全保障"改成"出问题时"，与第 15 项的本意一致 |
| 39-41 | 读结果、连发、PowerShell 退出码 | 改成 2 行：连发规则按 3.3 改写；PowerShell 一句保留 | 无。这是修正，不是删减 |
| 65-67 | `--all` 的细节（"node attributes/actions keep their default rules"） | 只留 "`--all` adds hidden/offscreen nodes and removes the 20-match limit"，其余放回 `observations.md:38-40`（那里已有） | 无 |
| 83-84 | `timing=ref-repair` | 留着。38 次里出现 4 次，回执本身也带一句说明 | — |
| 89-94 | 正文保存、64 KiB 摘要、`--no-print`、`--json`、`--raw` | 压成 2 行："Bodies over 64 KiB print a summary and the saved path; search that file instead of reading it whole." `--json` 和 `--raw` 两句移到 `observations.md`（那里已有） | 低。38 次里没有一次用 `--json` 或 `--raw` |
| 110-112 | "Obtain missing delivery or resolve cleanup instead." | 上一轮已指出读不懂，可以删，结果表里已有同样的意思 | 低 |
| 119-120 | UNKNOWN 最后一句 | 按 2.2 改写，与 `:118` 并成一行 | — |

合计约减 8–12 行，首页回到约 125 行，同时修掉两处实测里出过问题的写法。**不能删的**：防注入那句（`:9`）、后台屏 8 行、结果表、UNKNOWN 一节、收尾一节。

### 4.3 专题：可以删或移到 docs 的（逐条）

| 位置 | 内容 | 建议 | 风险 |
| --- | --- | --- | --- |
| `observations.md:94-97` | `mainLocked`/`mainInteractive`/`displayOff` 的含义 | 压成 1 行："The screen line marks known lock/off states; unknown adds no marker." | 低。0 次出现 |
| `observations.md:149-156` | 首树采样、`rootUnavailableObserved`、`rootRecoveredWithinRequest` | 压成 1 行："root-unavailable fields describe this request's sampling, not business success." 预算细节移到 docs | 低。出现 23 次，但模型从不需要读懂预算边界；回执里已有 UNKNOWN 或 verification 字段 |
| `observations.md:199-213` | wait 截断与 `cutoffStage`、any-of complete | 留"连续两次撞上限就停，`collectionCutoff` 区分来源"2 行，其余删 | 低。`cutoffStage` 0 次 |
| `observations.md:13-18` | `fresh` 对 `findRefreshed`、缓存树读失败 | 只留"`fresh` means a new tree was read"一句 | 无 |
| `actions.md:38-40` | 节点／字节上限下唯一性的例外 | 删。被拒时回执会报告缺口 | 低 |
| `actions.md:72-76` | 形状映射与单行多列例外 | 只留 "the receipt shows a differing route as direction → action" | 低 |
| `actions.md:204-206` | 确认落地之后的观察 | 删。`displays.md:218-220` 讲的是同一件事 | 无 |
| `connection.md:112-120` | 重绑规则 9 行 | 压成 2 行："After a worker restart, old sessions, work displays and refs are gone; reopen. If it does not reconnect, ask the user to use Refresh status." 两个阈值移到 docs | 低。设备端报错原文（`WorkerConnection.kt:121-122`）已经写了"请用户点 Refresh status" |
| `connection.md:141-155` | 设备诊断、锁文件路径 | 移到 docs | 低。0 次读取 |
| `displays.md:259-262` | 关闭摘要的格式 | 删掉格式说明，只留 "JSON keeps full task details" | 无 |
| `images.md` 历史清理诊断 2 行 | — | 删 | 无 |

合计约 -3 KB。专题在这轮 38 次里只被读过 29 次，删专题省下的成本有限。**优先顺序应当是首页（4.2），其次是实际被读过的 `displays`、`input`、`images`、`actions`、`reference`。**

---

## 5. 后台屏 8 行要点（第 15 项）

- **确实并进了首页**（`SKILL.md:26-33`）。决定文件要求不能漏的两句都在：关闭会销毁内容（`:29`）；搬来的或别的 App 的任务会挡住关闭，要先问用户，再选送回还是丢弃（`:30`）。另外补了"`--move-existing` 只在用户明确要求时用"、空闲到期也会销毁。`displays.md:201-202` 那句"选用前先读这些规则"也已经改成"排障用"。
- **对这道题够用。**（实测）5/5 次都依次做了 `session open --display virtual`、`apps Contacts`、`open`、增改删、`session close`、`status`。关闭回执是 `state=clear; destroyed=true`；没有一次在用户没要求时用 `--move-existing` 或 `--discard-tasks`。
- **但"专题改为出问题时再读"没做到。** 5/5 次都在第 2–3 次工具调用、出错之前整篇读了 `displays.md`（`reads.json`）。推理内容是加密的，原因只能推测：最可能的是首页 `:33` 和 `:128` 两处链接，加上安全导向的模型看到"safeguards"会主动去读。按 4.2 改掉这两处，代价两行，就可以检验。
- **难的情形没有用模型测过。** 后台屏题碰不到"App 已经在主屏开着、打开被挡"或"关闭被挡、要问用户"。决定文件第六节第 6 处建议另做一两次，实施方只有操作员手工的 `--move-existing` 验证（`r7-runtime-preparation.md:231`），没有让模型在只读首页的条件下做这两种情形。→ 证据不足；首页要点是否足以让模型在这两种情形下正确停下来问用户，需设备。

---

## 6. 独立 docs（`dfe5e45` → `090b1a2`，一个提交）

- **规模**：`git diff --stat` 显示 8 个文件，+392／-859 行。按受版本控制的字节数，从 301,632 降到 149,809，满足读-5 的 150 KB 口径（我自己重算过，与实施方的数字一致）。删得最多的是 `reference/ui.md`（100.5 KB → 28.0 KB）、`reference/connection.md`（37.8 KB → 12.3 KB）、`architecture.md`（31.6 KB → 10.7 KB）。
- **必留内容**：22 个案例从"真实业务清单"开始的原文与 `dfe5e45` 逐字节相同（52,478 字节，SHA-256 `4cb58215…`，与实施方的记录一致；能数出 22 个不同的案例编号）。"发布前真机检查"的六个勾选项逐行相同，只把"第七轮评审"改成了"第八轮评审"。小-5 没有拆，实施方的理由是"拿不准时保持原样"，符合决定。
- **与源码不符的地方**：抽查的都对。Selection 名字归并 64 节点／8 层、RowLabel 只有 64 节点上限（`UiSemantics.kt:31-32,240`）；近期三份样本（`UiRefs.kt:86`，`RECENT_SAMPLES = 3`）；5 秒／60 秒重绑；`freshRepair` 的条件；`apps` 需要租约。没有发现错。
- **"集中现行行为契约并删除重复说明"是否属实**：docs 内部确实去了重，比如编码与字节限额只在 `reference/README.md` 维护，其它页链接过去。但**与公开指引之间的重复反而更明显**：`reference/ui.md` 的"投影字段""行省略与缺口""ref、selector 与 guard"几段，基本是 `observations.md:28-43` 和 `actions.md:11-48` 的压缩中文转写。比如 `covered` 的两种含义、`--all` 的定义、`freshRepair` 的条件，三处（源码、指引、docs）各写一遍，措辞还各不相同。真正只在 docs 里有、对维护者有用的是："为什么这样设计"的取舍（如"检查不能让 App 静止：列表换业务对象但身份分量相同，产品无法识别"）、跨端职责划分、破坏性迁移表和旧码映射（`development.md`），以及案例和真机清单。
- **写法**：`reference/ui.md` 用的是极密的中文缩写，例如"撤旧节点权限""首次快退""保名字与完整性，但该父不继续贡献 nameConsumed/selectionTarget"，不读源码很难懂。对唯一的维护者来说，这一页的价值低于它的维护成本：行为一改，就要同时改源码、指引和这一页。
- **建议（设计取舍，需用户定）**：
  - A：保持现状。好处是不用再动。代价是三份说明长期同步。
  - B：`reference/*.md` 只保留"设计理由、跨端职责、平台风险"，行为细节一律链接公开指引和源码函数名。估计还能删 30–40 KB。代价是一次重写，且下一轮要复核链接。
  - C：删掉 `reference/ui.md`，把它的设计理由并进 `architecture.md`。省得最多，但会丢一部分只在这里写过的边界理由。
  - 推荐 B，可以放在发布之后做，不挡合入。

---

## 7. 发现清单（按去向）

**确定缺陷（都是一行级）**：
- A1 首页 `:91` 漏改，第 14 项的第三处提示。
- 首页 `:39` 连发规则改写（3/38 次实测出现）。
- A2 `callerState` 与 `callerSession` 两处说法。
- A3 `DaemonVersion` 大小写。
- A4 "t-job"。

**设计取舍（需用户定）**：
- UNKNOWN 新句按 2.2 再收紧成"任何输入都不要再试那项改动，报告出来"。
- 首页去掉两处邀请预读 `displays` 的链接（4.2），验证第 15 项的本意。
- 独立 docs 的 A/B/C（第 6 节）。

**可选建议**：
- 4.2 的首页删减和 4.3 的专题删减（合计首页约 -10 行，专题约 -3 KB）。
- `screenshot --help` 去掉 `--marks`。
- Click 报错的 ".;"。

**证据不足**：
- 新 UNKNOWN 句的效果没有实测。
- 后台屏"打开被挡／关闭被挡"两种情形下只读首页够不够，没有用模型测过（需设备）。
- 模型为什么预读 `displays`：推理加密，属于推断。

**核过没问题的**：第 1.1 节全部；22 个案例和六条真机检查原样保留；docs 抽查的数字；上一轮报错问题已修复的 7 条。

## 8. 未能核实的事项

- 需要设备的行为都没有实跑。第 1.1 节里的设备端说法是源码推断或实施方的原件。
- 第 3.2 节的比例是按回合数加权的估算，不是 token 分项实测。
- 我没有逐句通读独立 docs 的 150 KB，第 6 节是按段抽查的。
