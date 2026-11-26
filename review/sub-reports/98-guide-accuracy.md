# 第七轮专题核查 98：调用者指引与命令行帮助的准确性和可用性

核查对象：产品 `f401deb` 的 `skills/androidctl/*.md`（11 个文件，72,261 字节，1,308 行）、`S/r7/venv7` 中实际安装的 CLI 的 `--help` 和参数报错、内部 docs `dfe5e45`。真实运行证据是实施方材料包里本轮 20 次计分运行（`ST/efficiency/r6-*-candidate-0N/`）。
工作目录 `S/agents11/98/`（`S` 是评审暂存根目录）。其中：`help/`（67 页 `--help` 原文）、`errors.txt`（38 种参数错误的真实报错）、`examples.tsv`（指引里 83 条命令示例的实跑结果）、`seq.txt`（20 次运行逐次读了哪些指引、调了哪些命令、首行是什么）、`surface-diff.txt`（0.4.0 基线与候选的选项面对比）、`scripts/`（自写的抽取脚本，均以 `python3 -I` 运行，包内路径作参数传入）。
说明：下文"引用"指打印给调用者的节点编号（如 `e15`）；"后台屏"指 AndroidCtl 创建的虚拟显示；"常驻进程"指设备上以 shell 身份执行界面操作的进程；"摘要"指默认文本超过 64 KiB 时打印的缩略正文。

---

## 摘要

1. **准确性总体好，没有发现会让调用者做出错误、不可逆动作的陈述。**（源码推断、主机端复现）我抽查了 40 来条可验证陈述，包括数值常量、默认值、回执词、原因码、退出码含义和各条"遇到 X 就做 Y"，只找到 5 处小错。指引里 83 条命令示例逐条实跑，除 4 个本来就是占位写法的例子外，没有一条在参数阶段被拒。
2. **本轮 25 条裁决里要求改指引的每一条都已经落到指引里**，包括第 11、12、13、14、16、17、19–25 条和附录一的各小项。上一轮查出的借用粘贴两处说法相反已经统一，`\n\t` 原样打出的问题也已修好。
3. **最值得改的一处在首页，是写法问题，不是写错。**（实测）首页"Read/search that file after host-tool truncation"一句，在网页题 2/5 次运行里让模型把整份 168 KB 和 259 KB 的全文文件一次读进来，分别是 42,011 和 64,775 个原始 token，被执行框架截到 2 万。输出上限本来要省的那部分成本因此只是推后了一轮。建议改成"按关键词搜索这个文件，或用 `find`、`--depth` 缩小范围，不要整份读"。
4. **首页基本够用，有两处缺口。**（实测）路线题和网页题 10/10 只读首页就完成了。长列表题 4/5 又读了 `actions` 专题，比上一轮的 3/5 还多一次。读完之后做的都是首页已经写了的"再滚一次容器"。后台屏题 5/5 按首页要求读了 `displays` 专题，其中 4 次连带读了 `reference`。这两种读取分别占那次运行总输入的大约 5 个和 10 个百分点（估算）。首页加一行就能省掉长列表题这次读取。
5. **规模**：本轮增加的 3.3 KB 全部对应裁决要求写的新行为，没有一处是凭空多出来的。但附录一"专题写法"那一项几乎没有动：专题里的长难句和实现细节仍在，实施方计划页写的"专题完整句"只做到了很小一部分。按逐文件估算，指引可以从 72 KB 删到 48 KB 左右。不过专题在 20 次运行里只被读过 13 次，删专题能省的成本有限，排在首页改动之后。
6. **帮助文本与指引一致**，有 4 处小问题。参数报错大多给了可照抄的写法，并回显调用者自己的引用。仍不理想的有：`fill e42 --submit` 的示例回显成 `e15`；直接敲 `session`、`clipboard` 会把整页帮助压成一行近千字节打出来；`notifications get '-1|…'` 没有提示加 `--`；Click 自己生成的"缺参数"报错只说"去看 --help"。
7. **内部 docs**：发布前真机清单的条目齐全。接口迁移清单基本完整，只漏了 `wake` 去掉的两个选项。`reference/ui.md` 还留着已在指引里删掉的"full mode"一词。

总体判断：**指引和帮助这一块不挡合入。** 没有一条够得上"挡合入"的门槛。建议合入前顺手改 3 处一行级的文字：首页那句读文件的写法、`jobs-files.md` 里那半句截断的话、`fill` 报错里的示例引用。其余放进下一轮。

---

## 1. 逐句核对准确性

### 1.1 核对过且与实现一致的（已核实无问题）

| 指引陈述（位置） | 依据 |
| --- | --- |
| 超过 64 KiB（65,536 UTF-8 字节）打印摘要，文件存全文（`SKILL.md:85-86`、`observations.md:234-236`、`images.md:71-73`） | 源码 `androidctl/src/androidctl/ui_client.py:45,733-739`：阈值是 `> 65_536`，只在非 `--json`、未显式 `--print`、非 `--raw` 时生效 |
| 摘要含采集编号、缺口、`find` 命中计数、文件路径和一行缩小范围的写法（`observations.md:234-235`） | 源码 `ui_client.py:864-888`（摘要里含 `Narrow the current sample: androidctl snapshot --cached --depth 2`）。实测原件 `r6-c2-candidate-01/evidence/cli/1795616656204020000-…/child.stdout`（590 字节） |
| 摘要情形下全文没存成文件：`snapshot`／`find` 报失败；动作、`wait`、`locate` 结果不变，退出码 1（`images.md:85`） | 源码 `ui_client.py:1019-1031`（只有 `snapshot`、`find` 改成失败），`1057-1065`（其它命令退出码改 1） |
| `find` 缓存零命中时在同一范围、同一预算里新采一次，首行带 `fresh`；新采失败报失败（`SKILL.md:78-79`、`observations.md:207-209`、`find --help`） | 源码 `ui_client.py:612-648`，`ui_output.py:91-94`。实测 `r6-c2-candidate-01` 首行 `find: done; fresh; matches=0` |
| `--raw` 默认只保存，与 `--all/--depth/--max-chars/--interactive/--boxes`、`find --context`、`snapshot REF` 组合在参数阶段拒绝（`observations.md:239-241`） | 主机端复现（`errors.txt`）：`snapshot --raw --all` → `--raw reads the whole observation and cannot combine with --all; use androidctl snapshot --cached --raw, or omit --raw for a projected view; code=arguments` |
| 单独的 `screenshot --marks` 被拒（`images.md:25-26`、`SKILL.md:94-95`） | 复现：`Standalone screenshot has no node tree to label; use androidctl snapshot --screenshot --marks` |
| 自动化登记被别的工具占着时报 `busy subject=automation`，接管无效，等对方退出（`SKILL.md:24-25`、`connection.md:96-97`） | 源码 `android/.../AutomationCleanup.kt:34-41`（消息本身也写了 "takeover has no effect; wait for it to exit"），码表 `contract/reason-codes.json` 的 `busy` 一行分主体写了两句 |
| 常驻进程意外退出、而且已连上 5 秒以上，才自动重绑一次；重绑后在下一条"需要会话"报错里附一句说明，旧会话、后台屏和任务作废，不重放（`connection.md:106-110`） | 源码 `WorkerConnection.kt:104,293`（`MIN_CONNECTED_MS = 5000L`、`> MIN_CONNECTED_MS`），`:180` 是那句说明的原文 |
| `status` 和被拦下的关闭回执给剩余空闲秒数；`status` 不续期，`snapshot` 续期（`displays.md:165-166,236-237`） | 源码 `Sessions.kt:367-374`、`TargetManagement.kt:323`；主机测试 `test_display.py:272` 断言文本里有 `idleRemainingSeconds=1800` |
| 后台屏到期后的报错带"要继续用后台屏请写 `--display virtual`"；不写 `--display` 时新开主屏并附同一句提示（`displays.md:166-169`） | 源码 `SessionExpiry.kt:10` |
| `--replace` 和 `takeover` 必须写 `--display`（`displays.md:158`） | 复现：`--replace requires --display. Use session open --display main --replace or session open --display virtual --replace.; code=arguments; argument=display` |
| 竖长容器在没有方向动作时把 up/down 映射到 backward/forward（`actions.md:61-62`） | 源码 `UiScroll.kt:52-60,65-70`（另有 `ScrollView`、`GridView`、单列集合几种情况，指引没有列全，但不影响调用） |
| 文本首行最多列 20 个引用加总数，`--json` 给全部（`actions.md:78-79`、`observations.md:374-375`） | 源码 `ui_output.py:32` |
| 屏外点击被拒时给下一步：有 SHOW_ON_SCREEN 就建议 `reveal`，没有就建议 `snapshot` 后滚动容器 | 源码 `UiAdvice.kt:11-18`。实测 `r6-c2-candidate-01`：`click: not sent; … target is not visible; reveal it with androidctl reveal e950416952, then retry` → `reveal` → `click` 成功，只多了一轮 |
| 借用粘贴：粘贴已核对成功时，恢复剪贴板失败报完成且退出码 1，恢复状态未知报完成且退出码 3（`input.md:61-62`、`reference.md:216-217`） | 两处说法现在一致，与第 22 条相符（契约侧由其它专题核） |
| 数值常量：8192 节点、8 MiB 树、3 s 采集预留、预览长边 1280、JPEG 质量 80、最多标 40 个目标、历史 150 个文件／20 MiB、每路输出 128 KiB、本机后台服务空闲约 5 分钟后退出、`scroll` 默认方向 `forward` | `Wire.kt:20-27`、`UiBudget.kt:111`、`ImageCoordinates.kt:25`、`VisualInput.kt:428`、`ImageMarks.kt:109`、`history.py:17-18`、`daemon.py:24`、`ui_commands.py:1324` |
| 预算表的默认值（普通 5 s、截图 15 s、settle 上限 1500 ms、shell 10 s／10 s、后台 shell 10 m／0、传输 10 m／10 s、`job get --wait` 默认 0） | 与 `help/*.txt` 里各命令的 `[default: …]` 一致 |
| 指引里提到的 30 个原因码都在 150 码的码表里；第 24 条并掉的旧码（`identityChanged`、`noPublishedSnapshot`、`internalError`、`messageLimit`）在指引和 docs 里都已经不再出现 | 自写脚本对照 `contract/reason-codes.json` |
| `androidctl help SKILL` 的输出与仓库文件逐字节相同；小写的 `help skill` 被拒，与 `reference.md:161`"Use filename stems, including SKILL"一致 | 主机端复现 |
| 实际运行时读到的指引就是 `f401deb` 的版本 | 材料包 `r6-final-materials/skill/` 与仓库文件除行尾的 CRLF 外完全相同；原生记录里 `SKILL.md` 的读取命中 86/91 长行 |

### 1.2 写错、过时或残缺的

| # | 位置 | 现象 | 依据 | 对调用者的影响 | 分类 |
| --- | --- | --- | --- | --- | --- |
| E1 | `jobs-files.md:67` | 句子被截断，正文就是 "`Blocked kernel I/O is not forcibly interrupted. Local t...` / `queries read host facts …`"。原意应是"本机传输查询读主机上的事实，不联系设备"。这处错从提交 `8adb58d` 起一直存在，本轮没有发现 | 源码（文件原文），`git log -S` | 小：读者看不懂"不联系设备就能查传输状态"这条规则。普通任务读不到这个文件 | 确定缺陷；**建议合入前顺手改**（一个词），否则进下一轮 |
| E2 | `connection.md:125` | 写成 "`DaemonVersion` rejects…"，大写开头，也没加反引号；码表里的码是 `daemonVersion` | 码表 | 很小：按码精确匹配时对不上 | 确定缺陷；进下一轮 |
| E3 | `SKILL.md:22` | "Open explicitly before UI, notifications, clipboard, shell or file starts" 漏了 `apps`；`displays.md:160` 写了 `apps` 也需要会话 | 两处指引互相对照 | 很小：首页示例本来就先开会话再调 `apps` | 可选建议；进下一轮 |
| E4 | `actions.md:90` | 滚动查找的停止原因列了"步数上限、观察不完整、截止、平台拒绝、容器变化"，没有列真实出现过的"这个方向没有路线"（`stop=routeUnavailable`） | 实测 `r6-c1-candidate-05`：`scroll-until: failed; After scrolling 2 steps, the next scroll was not sent: … no advertised route for scroll-down …; stop=routeUnavailable` | 很小：回执本身已经给出可用方向和可照抄的写法 | 可选建议；进下一轮 |
| E5 | `SKILL.md:87`："Read/search that file after host-tool truncation." | 写法不准，不是事实错误。真实运行中它被理解成"整份读进来" | 见第 3.3 节 | 中：网页题 2/5 一次多吃 2 万 token | 设计取舍或可选建议；**建议合入前改写成一行** |

### 1.3 指引讲了而实现不支持的

没有发现。83 条示例里，被拒的只有 `androidctl COMMAND --help`、`androidctl key=value` 这两个占位写法，以及 `device address DEVICE HOST:PORT`（占位端口）和 `shell --script-file task.sh`（文件不存在）（主机端复现，`examples.tsv`）。

### 1.4 实现支持、调用者会需要、而指引没写的

- **引用来自"可能还在变化"的观察时可能过期，过期后应当重新 `snapshot`。**（实测）路线题 2/5 都是这样：`fill … --submit` 的首行是 `change=loading … (page may still be transitioning)`，模型直接点了这份观察里的引用，被拒为 `click: not sent; No current target matches the original ref signature; code=stale; subject=ref`（`r6-c1-candidate-01`、`-04`）。首页 `SKILL.md:80` 写了"遇到过渡就新采 snapshot"，但模型没有照做。这条拒绝回执本身也没有给下一步，而同类的 `UiBackend.kt:279` 写的是 "Ref is not in the current snapshot; run snapshot"。`UiRefRepair.kt:99` 这句只要加上 "; run androidctl snapshot and use the new ref" 即可。上一轮 74 号报告 D3 已经提过，不在第六轮的封闭清单里，所以属于遗留项。→ 可选建议，进下一轮。维护代价是改一个字符串。
- **滚动查找命中后，匹配项常常贴在容器边缘，看全它需要再滚一次。** 见第 3 节。
- 不支持而调用者可能误以为支持的：`find` 没有 `--cached` 选项（复现：`No such option: --cached`）。指引没说它有，不算错。

### 1.5 指引内部互相矛盾的

- 上一轮查出的借用粘贴两处说法相反（`input.md` 写"报完成需要清理完成"，`reference.md` 写"已知的主结果可以保持完成"），**已经统一**：现在两处都写成"完成，退出码 1／3，不要再粘贴"。
- 本轮没有发现新的直接矛盾。有几处同一规则写了三遍，措辞各不相同，以后改一处容易漏另外两处。详见第 4.2 节。

---

## 2. 本轮改动在指引里的落点

| 裁决 | 指引现在怎么写（位置） | 调用者能否照做 | 写法评价 |
| --- | --- | --- | --- |
| 11 `find` 零命中自动新采 | `SKILL.md:78-79`；`observations.md:207-209`；`find --help` 第一段 | 能。首行出现 `fresh`，实测里看得到 | 首页两行可以压成一行。`observations` 里那句"The result says `fresh`"不完全准：未稳定之后的重采首行也写 `fresh`（`ui_output.py:91-93`），两种情形区分不出来，不过调用者也不需要区分 |
| 12 到期提示、剩余秒数、`snapshot` 保持 | `displays.md:165-169,236-239` | 能 | 第 236-239 行讲"记录可能 pending 或 unavailable、常驻进程退出后丢失"，属于少见情形，可以删掉后半句 |
| 13 屏外节点判为不可见 | `observations.md:226-228`；`SKILL.md:62-63`（`reveal` 或滚动）；拒绝回执里给 `reveal` 的写法 | 能，实测一次成功（见 1.1） | 好。风险提示里点名的"后台屏输入框滚出屏外后 `fill` 读回报无法观察"没有单写，但 `actions.md:143`"读回不可得是 UNKNOWN"已经覆盖 |
| 14 输出上限 | `SKILL.md:85-89`；`observations.md:234-238`；`images.md:71-73,85` | 部分能。知道文件在哪，但不知道该怎么读：真实运行里整份读了（见 3.3） | 同一规则写了三遍。"例外"和"`--print` 只在需要时用"按裁决写了 |
| 16 滚动查找选择器模式 | `actions.md:77-81` | 能 | 好；`scroll-until --help` 第一句仍写 "find an exact selector target"（`ui_commands.py:1345`），容易读成要求唯一，进下一轮 |
| 17 截断规则 | `actions.md:33-34`、`SKILL.md:82`、`observations.md:302`（只剩"拒绝回执会报告缺口和候选"）；`observations.md:371-375` 写了 `wait` 的新规则 | 能 | 三处例外说明按裁决删到了只剩报错。`wait` 那段 5 行，可以压成 2 行 |
| 19 重绑后旧状态作废 | `connection.md:106-110` | 能。下一条报错本身就带说明 | "attached for more than five seconds" 是实现阈值，调用者用不上；保留"重绑后旧会话、后台屏、任务、引用都作废，重新 `session open`"就够了 |
| 20 其它系统用户的任务、送回可能搬错 | `displays.md:215,225-226` | 能 | 好 |
| 21 删掉中间 Activity 的例外 | `images.md:141-144` 已删；`displays.md:196` 讲任务列表覆盖面的那句按裁决保留 | — | 正确 |
| 22 借用粘贴退出码 | `input.md:61-63`；`reference.md:216-217` | 能 | 好，两处一致 |
| 23 `--raw` 默认只保存 | `SKILL.md:89`；`observations.md:238-244` | 能 | `observations` 里列 6 个不能组合的选项那一段占 6 行，报错本身已经给出可照抄的写法，可以压成"`--raw` 不能与改变正文形状的选项同用，报错会给出写法" |
| 25 `session open` 默认主屏 | `connection.md:92`；`displays.md:157-158`；首页示例保留 `--display main` | 能 | 好 |
| 24 原因码与 `busy` 分主体 | `SKILL.md:24-25`；`connection.md:96-97` | 能 | 首页这两行属于极少见情形，报错原文就是完整指示，建议从首页删掉，见 3.4 |
| 附录：首页补 `apps` 和树行格式 | `SKILL.md:41,52-53` | 能。后台屏题 5/5 用了 `apps Contacts` | 好 |
| 附录：首页 `key enter` 改 `fill --submit` | `SKILL.md:45` | 能。`fill … --submit` 在 20 次运行里用了 24 次 | 好 |
| 附录：通知专题的 full mode | `notifications.md:17-19` 已删 | — | 好。内部 docs `reference/ui.md:343` 仍写"只在full/current user0运行"，源码里已经没有这个模式（`grep` 无结果） |
| 附录：方向词映射、单独 `screenshot --marks` 拒绝、报错回显引用 | 见上 | 能 | `screenshot --help` 仍列出 `--marks` 选项，而这个选项总是被拒，见 5.1 |
| 附录：指引专题的写法 | 实施方计划页（`ST/r6-plan.md:60`）写"专题完整句、删除无定义 full mode、删过度解释但保具体下一步，已整合审结" | — | **只做了一小部分**。本轮差量（`skills-delta.diff`）里专题的删改只有三类：截断例外、full mode、中间 Activity，没有一段是为了文风或去掉实现细节而重写的。上一轮举例的长难句（"Explicit `page-*` requires that exact advertised action."、"Fields resolve independently; invalid configured devices do not fall back."、"`↵` followed by a newline" 等）原样保留 |

结论：**每条裁决都让调用者知道了新行为，也知道该怎么做。** 新写的句子大多简短可照做。长出来的解释性段落集中在 `observations.md` 的 `--raw` 一段和 `wait` 截断一段、`displays.md` 的到期记录、`connection.md` 的重绑阈值，各自可以删一半。

---

## 3. 首页

### 3.1 20 次运行实际读了什么、用了什么（实测，`seq.txt`）

| 题 | 读的指引 | `--help` | 不成功的命令 |
| --- | --- | --- | --- |
| 路线题（Maps）5 次 | 只读首页 | 1 次（`collapse --help`，`r6-c1-candidate-03`） | 2 次引用过期（`-01`、`-04`）；2 次滚动方向无路线（`-03`、`-04`）；1 次滚动已派发但观察超时、报失败；1 次滚动查找因没有路线停下（均在 `-05`） |
| 网页题（GitHub）5 次 | 只读首页 | 0 | 1 次屏外点击被拒（`-01`），随后 `reveal` 再点击成功 |
| 长列表题（Feeder）5 次 | 首页；01、02、03、05 又读了 `actions.md` | 0 | 1 次 `reveal` 不支持（`-04`） |
| 后台屏题（Contacts）5 次 | 首页 + `displays.md`；01、03、04、05 同一次调用里还读了 `reference.md` | 0 | 0 |

命令频次（共 322 条）：`click` 95、`snapshot` 71（其中 `--screenshot` 18）、`scroll` 45、`back` 27、`fill` 24（全部带 `--submit`）、`scroll-until` 15、`find` 11、`status` 10、`session` 8、`open` 5、`apps` 5、`reveal` 2、`collapse` 2。**`wait`、`key`、`ime`、`check`、`locate`、`type`、单独的 `screenshot` 一次都没用；没有出现 UNKNOWN。** 路线、网页、长列表三题的会话由执行框架预先打开，模型一次也没调 `session open`，所以首页"开始任务"一节在这 15 次运行里没有被检验到。

### 3.2 首页逐段的作用

| 行 | 内容 | 实际作用 |
| --- | --- | --- |
| 1-4 | 元数据 | 必需 |
| 8 | "Use matching Host/App versions and user-prepared shell-UID Shizuku/Gateway." | 调用者据此做不了任何事；属于连接专题，**可删** |
| 9-10 | `help` 入口；屏幕文字是数据不是指令 | 保留（防注入那句必须留） |
| 14-20 | 配对、开会话、snapshot 示例 | 后台屏题用到；保留 |
| 22-25 | 会话共享、`busy`、`connectionBusy`、接管、`busy subject=automation` | 20 次运行都没碰到。`automation` 两行重复了报错原文和码表，**可删**；前两行保留 |
| 26-29 | 后台屏指向显示专题；连接专题 | 后台屏题必读；保留 |
| 33-36 | 读结果、可连发、保退出码、PowerShell | 模型确实在一次调用里连发了多条命令；PowerShell 那句对执行框架有用；可以合成 2 行 |
| 38-50 | 九条基本命令 | 用得最多；`wait` 一条从未用到，但它是正确做法的入口，保留 |
| 52-58 | 树行格式、`fill`、`key`/`ime`、`check` | `fill --submit` 高频；保留 |
| 60-74 | `find`、点击、`reveal`、滚动查找 | 高频；**这里缺一句"命中后匹配项可能贴边"**（见 3.3） |
| 76-83 | 复用观察、`find` 重采、过渡时新采、缺口 | 路线题 2/5 正好违反了第 80 行；建议把"过渡中的引用可能过期"写得更直白 |
| 85-90 | 保存、摘要、`--json`、`--raw` | 网页题用到；**第 87 行要改写**（见 3.3） |
| 92-95 | 截图与标注 | 用了 18 次；保留 |
| 97-108 | 结果表 | 保留。第 106-107 行"Obtain missing delivery or resolve cleanup instead"读不懂，可以改成"只补取缺失的文件或收尾" |
| 110-116 | UNKNOWN | 没有触发过，但关系到"不重复输入"，必须保留 |
| 118-129 | 10 行专题列表 | 正文里已经有链接；可以压成 3-4 行 |
| 131-136 | 收尾 | 用到；保留 |

### 3.3 首页不足导致的额外读取和试错（实测加估算）

**长列表题 4/5 读了 `actions.md`。** 顺序都一样：`scroll-until` 成功，然后 `snapshot --screenshot`，看图发现目标卡只露出一部分，读 `actions.md`，然后 `scroll 容器 down` 一次，再点击（`seq.txt` 里 `r6-s1-core-candidate-01/02/03/05`）。读完之后用的都是首页已有的写法。没读专题的 `-04` 先试了 `reveal`（被拒），再滚动，结果一样。成本页按它自己的分摊法估出：首页加这次专题读取，合计约占长列表题总输入的 10.6%–12.5%（`core-results.md:148,150`）；只读首页的 `-04` 是 6.0%（`:156`）。所以这次专题读取约占 **4.5 到 6.5 个百分点**（估算，沿用成本页的口径）。
→ 首页第 70-72 行后面加一行即可，例如："The match is often at the container edge; to see the whole item, scroll the container once more (or `reveal` it when advertised), then act." 可选建议，进下一轮；收益明确，代价是一行。

**后台屏题 5/5 读 `displays.md`，4/5 连带读 `reference.md`。** 这是首页要求的。指引驻留合计约占 16.2%–17.1%（`core-results.md:225,233`），只读首页时约 5.5%，差约 **10 个百分点**（估算）。5 次运行没有一次出错，用到的命令（`session open --display virtual`、`apps`、`open`、`session close`、`status`）首页都有。`displays.md` 里真正为安全所需的是四件事：已在别处运行的 App 会挡住打开，要搬动需用户同意并用 `--move-existing`；关闭会销毁屏上内容；关闭被拦下时要问用户，再选 `--return-tasks` 或 `--discard-tasks`；后台屏与主屏共享 App 数据和账号。这四件事可以写成首页里 4-5 行，`displays.md` 改为出问题时再读。`reference.md` 被连带读取没有直接原因，推测是首页"结果表"和 UNKNOWN 两处都链接到它。这是对"首页要求先读显示专题"这一已定事项的重议，属于设计取舍，进下一轮，由用户决定。

**网页题：摘要之后整份读文件。** `r6-c2-candidate-01` 执行 `snapshot --cached --all` 得到摘要（168,039 字节的全文），下一步就是 `Get-Content -Raw` 读整份文件，原始 42,011 token，被截到 2 万。`r6-c2-candidate-05` 执行 `snapshot --all` 得到 259,096 字节的全文，先整份读（原始 64,775 token，截断），接着才改用 `Select-String` 按关键词搜索，之后又按行段读（原始 31,524 token，也被截断）。输出上限在这两次里没有省下成本，只是把成本推后了一轮。原因之一正是首页第 87 行的"Read/search that file"把"读"放在"搜"前面。
→ 改成："Search that file with the host's text search, or narrow with `find`/`--depth`/`--max-chars`; avoid reading it whole." 摘要里给的缩小范围写法是 `snapshot --cached --depth 2`，对 `find` 的摘要不太对路，可以按命令给 `find QUERY` 一类的写法（`ui_client.py:887`）。可选建议；首页这一句的改写**建议合入前做**，代价一行。

**路线题 2/5 点击了过渡中观察里的引用，被拒为过期**（见 1.4），每次多一轮 `snapshot`。

### 3.4 首页删什么、加什么（保持约 130 行）

- **删**（约 -12 行）：第 8 行；第 24-25 行（`automation`）；第 33-36 行合成 2 行；专题列表 10 行压成 4 行；第 106-107 行改写成 1 行。
- **加**（约 +7 行）：滚动查找命中贴边时再滚一次（1 行）；过渡中的引用可能过期，被拒为过期就重新 `snapshot`（1 行）；摘要文件要搜不要整读（第 87 行改写，±0）。如果用户同意把后台屏要点放进首页，再加 4-5 行。
- 结果大约 131 行，在约束之内。

---

## 4. 规模与写法

### 4.1 本轮增加的 3.3 KB 加在哪

| 文件 | `fb75afc` → `f401deb` | 内容 | 是否必要 |
| --- | --- | --- | --- |
| `observations.md` | 9,698 → 10,878（+1,180） | 可见性定义、`find` 零命中、输出上限、`--raw` 组合拒绝、`wait` 截断新规则、事件只留提示条 | 都对应裁决。`--raw` 一段（约 450 字节）和 `wait` 一段可以各删一半 |
| `displays.md` | 5,479 → 6,217（+738） | 不写 `--display`、剩余秒数、到期提示与任务记录、其它系统用户、送回可能搬错 | 必要；到期记录"pending／unavailable"那半句可删 |
| `connection.md` | 6,827 → 7,365（+538） | 不写 `--display`、`busy` 分主体、自动重绑 | 必要；"5 秒"可删 |
| `SKILL.md` | 6,612 → 7,035（+423） | `automation`、`apps`、树行格式、零命中、输出上限 | 除 `automation` 两行外都必要 |
| `images.md` | +196 | 单独标注被拒、摘要交付一行 | 必要 |
| `reference.md` | +195 | 借用粘贴退出码、`arguments` 的定义 | 必要 |
| 其余 | `notifications` +38、`actions` +24、`input` -12 | — | — |

### 4.2 专题里重复首页的内容

- 输出上限写了三遍：`SKILL.md:85-89`、`observations.md:234-238`、`images.md:71-73`。
- `find` 重采写了三遍：`SKILL.md:76-79`、`observations.md:204-209`、`find --help`。
- `busy subject=automation` 写了四遍：`SKILL.md:24-25`、`connection.md:96-97`、码表、报错原文。
- 树行格式写了两遍：`SKILL.md:52-53`、`observations.md:261-263`。
- 滚动查找语义写了两遍：`SKILL.md:70-73`、`actions.md:73-81`。
- 标注写了两遍：`SKILL.md:94-95`、`images.md:24-27`。
- 不写 `--display` 写了两遍：`connection.md:92`、`displays.md:157-158`。

建议专题只留首页没有的细节，并链回首页。

### 4.3 过度解释实现原理、只为极少见情形写的长段（举例）

- `actions.md:7-22` 引用保留与修复。调用者只需要知道"过期就重新 snapshot；名字变了的拒绝会给出同位置的新引用，先确认再用"。
- `actions.md:142-153` `fill` 读回的 `↵` 归一、提示文本当空值、`--settle 0` 仍做一次本地读取。
- `observations.md:304-315` 缺口取证步骤。
- `observations.md:317-329` 动作后采样的独立事实。
- `observations.md:359-363` 画面变化的基线。
- `observations.md:380-386` 事件。
- `connection.md:117-121` 证书续期。
- `connection.md:131-145` 设备诊断、锁文件路径。
- `input.md:33-66` 借用粘贴整节约 30 行、二十来条规则。
- `jobs-files.md:240-247` 发布边界的安全论证。
- `reference.md:224-235` 资源释放的常驻进程编号比对。首页 UNKNOWN 一节链接到这里，不能整段删，但可以写成三步可照做的检查。
- `reference.md:237-262` JSON 字段导读。

### 4.4 可以删到多少（估算）

| 文件 | 现在 | 估计可到 | 删法 |
| --- | ---: | ---: | --- |
| SKILL | 7.0 KB | 6.8 KB | 见 3.4（删和加基本相抵） |
| actions | 11.1 KB | 7.0 KB | 引用修复压成 2 行；`fill` 读回细节、勾选行点击、`range` 细节压成各 1-2 行；启动意图的选项说明交给 `open --help` |
| observations | 10.9 KB | 6.5 KB | 删取证段、动作后采样段；`--raw` 与 `wait` 截断各半；事件 2 行；只留首页没有的阅读选项和窗口范围 |
| images | 8.5 KB | 5.5 KB | 区域截图三段压成一段；交付表留 3 行 |
| reference | 7.6 KB | 5.0 KB | JSON 字段导读删到 3 行；资源释放写成可照做的三步 |
| connection | 7.4 KB | 4.5 KB | 设备诊断、证书续期、本机后台服务版本细节移入内部 docs，留各一句"做什么" |
| displays | 6.2 KB | 4.5 KB | 收尾一节压成两句；到期记录半句删掉 |
| input | 5.0 KB | 3.0 KB | 借用粘贴保留"要先聚焦；结果看 `effect`／`paste`／`restore`；恢复失败不要重粘" |
| jobs-files | 5.2 KB | 3.5 KB | 发布边界一段移入内部 docs |
| clipboard、notifications | 3.3 KB | 3.0 KB | 基本不动 |
| **合计** | **72.3 KB** | **约 49 KB** | 约 -32% |

**取舍**：20 次运行里专题只被读过 13 次：`actions` 4、`displays` 5、`reference` 4，其余 7 个专题一次都没读过。删没人读的专题，对这四道题的成本没有影响。真正影响成本的是首页的两处改动（3.3），以及 `displays`、`actions`、`reference` 这三个被读的文件。所以建议的顺序是：首页 → `displays`/`actions`/`reference` → 其它。整体删减属于可选建议，进下一轮，不挡合入。另外，上一轮已经指出的问题（"正确做法是少写规则，不是把同样多的规则压短"）仍然成立，本轮没有改善。

---

## 5. 帮助文本与参数报错

### 5.1 `--help` 与指引的一致性（主机端复现，`help/`）

全部 67 页帮助（60 多个命令加各命令组）都能正常打出。逐页对照指引，没有发现两边说法相反的地方。小问题有 5 处：

1. `snapshot`／`find`／`locate` 的 `--print/--no-print` 说明是"Print the received observation body by default in text mode"，没有提超过 64 KiB 时只打摘要，与第 14 条不完全一致。进下一轮，改一行。
2. `screenshot --help` 仍列出 `--marks`（说明是"Label public REF targets with snapshot --screenshot --marks. Standalone screenshot has no node tree to label."），但这个选项现在总是被拒。可以从这条命令上移除，或在说明里写明"总是拒绝"。进下一轮。
3. `scroll-until --help` 第一句 "Observe matching text or find an exact selector target" 是第 16 条之前的说法。进下一轮。
4. 所有动作命令的 `--settle` 说明都带 "check/uncheck verification can continue beyond --settle"；`snapshot --all` 的说明里出现 "find: show all deepest matches"；每页重复 4 个全局选项。67 页合计 98.7 KB，其中重复的全局选项约 40 KB。上一轮 D7 已提过，没有改。20 次运行只调过 1 次 `--help`，所以只影响人工阅读。进下一轮。
5. `open --help` 的 `--action`、`--data`、`--mime-type`、`--category`、`--component`、`--extra` 没有说明文字。指引 `actions.md:163-174` 写了。可选建议。

### 5.2 参数报错（主机端复现，全文在 `errors.txt`）

**做得好的**（给了可照抄的写法，大多回显调用者自己的引用）：

```
click: not sent; Provide a ref or at least one selector flag; for example, e15 or --name Save; code=arguments
click: not sent; Use a ref or selector flags, not both; for example, e42 or --name Save; code=arguments
click: not sent; No such option: --tuoch (Possible options: --touch); inspect accepted syntax with: androidctl click --help; code=arguments
snapshot: not sent; --raw reads the whole observation and cannot combine with --all; use androidctl snapshot --cached --raw, or omit --raw for a projected view; code=arguments
snapshot: not sent; --raw reads the whole observation and cannot combine with REF; use androidctl snapshot --cached --raw, or omit --raw for a projected view; code=arguments
screenshot: not sent; Standalone screenshot has no node tree to label; use androidctl snapshot --screenshot --marks; code=arguments
snapshot: not sent; --marks requires --screenshot or --image-out; use androidctl snapshot --screenshot --marks; code=arguments
session open: not sent; --replace requires --display. Use session open --display main --replace or session open --display virtual --replace.; code=arguments; argument=display
session takeover: not sent; session takeover requires --display. Use session takeover --display main or session takeover --display virtual.; code=arguments; argument=display
scroll: not sent; Direction must be forward|backward|…|page-right; use androidctl scroll e42 down; code=arguments
scroll-until: not sent; Provide a container REF and --contains or selector flags: androidctl scroll-until e42 down --max-steps 10 --contains Settings; code=arguments
key: not sent; A is ambiguous; use androidctl key a or androidctl key shift+a; code=arguments; argument=key
{"command":"snapshot","result":"not sent","reason":"--print and --json cannot be used together; use androidctl snapshot --print or androidctl snapshot --json",…}
snapshot: not sent; Invalid value for '--timeout': Use an integer with ms, s, m or h; add a unit, for example 1s. Only 0 may omit the unit; …
notifications: not sent; --timeout before a subcommand is a bare notifications option; put subcommand options after its name, for example: androidctl notifications list --timeout 5s; code=arguments
session open: not sent; --return-tasks and --discard-tasks are mutually exclusive. Choose --return-tasks or --discard-tasks.; code=arguments
snapshot: not sent; Cached observations cannot request a new screenshot; use androidctl snapshot --screenshot; code=arguments
swipe: not sent; Invalid value for '--duration': Duration must be between 50ms and 5s; for example, 400ms; …
```

`@e15` 照常接受（`ui_client.py:400-405`）。

**上一轮的 `\n\t` 问题已修。** 上一轮候选 `session open` 缺 `--display` 时报 `Missing option '--display'. Choose from:\n\tmain,\n\tvirtual`（用 `S/r6/venv6` 复现）。现在不写 `--display` 是合法用法；其它由 Click 生成的报错也不再含转义字面：`session open --display` 不给值时报 `Option '--display' requires an argument.`，`session` 命令组原先带 `\n\n` 的输出现在也合成了空格。

**仍不理想的**（都属于可选建议，进下一轮）：

1. `fill e42 --submit` → `Use fill REF TEXT or fill TEXT with selector flags; for example, androidctl fill e15 hello …`。这里没有回显调用者的引用，仍是固定的 `e15`。照抄会作用到另一个节点上，而 `e15` 恰好可能是当前页上一个真实存在的引用。附录一"示例回显调用者自己的引用"在这条上没有做到。确定缺陷，改一处字符串，**建议合入前顺手改**；不改也不挡合入（没在真实运行中出现过）。
2. 直接敲 `androidctl session`、`androidctl clipboard`：把整页帮助压成一行打出来，约 930 字节，以 `session: not sent; Usage: androidctl session [OPTIONS] COMMAND [ARGS]... Own this device for a task. Options: --device <str> …` 开头。而 `device`、`daemon`、`job` 报的是 `Missing command.; inspect accepted syntax with: androidctl device --help`。两种写法不一致；前者应当改成"缺子命令，可用 open|close|takeover"。
3. `notifications get '-1|a|7|null|1'` → `No such option: -1; inspect accepted syntax with: androidctl notifications get --help`，没有提示加 `--`（帮助和指引里都写了）。
4. Click 自己生成的缺参数报错只给"去看 --help"，没有可照抄的写法：`Missing argument '[REF] TEXT'.`、`Missing option '--max-steps'.`、`Missing option '--via'.`、`Missing argument 'x2'.`、`Got unexpected extra argument(s) (3)`。
5. `wait --contains a --idle`、`wait --gone --contains x` 都报 `Provide one nonempty wait condition or --any-of; for example, androidctl wait --contains Ready`。前者没说"只能一个，组合请用 `--any-of`"，后者没说"`--gone` 需要选择器字段"。
6. `snapshot --cached --in com.x` → `Take a scoped snapshot first, then render its cache: androidctl snapshot --window 1; then androidctl snapshot --cached`。示例换成了 `--window 1`，没有沿用调用者写的 `--in com.x`。
7. 写错形状的引用（`click ref15`、`click E15`、`click @@e15`）在主机端不拒，要等到设备那边才报 `Ref is not in the current snapshot; run snapshot`（`UiBackend.kt:279`）。这条报错有下一步，结果可以接受；在主机端用 `e[0-9]+` 先挡一下会更早、更便宜。
8. 主机端各命令的报错格式不一致：界面命令把 `code=…` 接在首行末尾，`help` 等本地命令把 `code=arguments` 另起一行（`output.py:72-78`）。不影响使用。

---

## 6. 内部 docs 抽查

- **发布前真机清单**（`testing/cases.md:15-24`，`development.md:75` 链接到它）：配对、主屏基本循环（第 19 行）；后台屏开关、读网页、填表（第 20 行）；1080 像素宽以上带标注截图并按图点击（第 21 行）；0.5.0 覆盖安装保留 0.4.0 配对（第 22 行）；可选国产 ROM（第 23 行）；写公开发布说明（第 24 行）。**用户要求的 9 项全部在**。唯一的小毛病是第 20 行把"后台屏开关、读网页、填表"合成一句，读起来像是"网页和填表都要在后台屏上做"。如果本意是网页和填表放在主屏、后台屏另测，宜拆成两条。可选建议。
- **破坏性变更清单**（`development.md:77-89` 接口迁移清单）：我用 0.4.0 基线 `7d5be70` 的源码配同一 Python 环境跑 `--help`，与候选的选项面对比（`surface-diff.txt`）。被删的有：动作命令的 `--print` 和 `--image-out`、`wait --print`（清单写了）、`wake --screenshot` 和 `wake --settle`（**清单没写**，只有一句"各命令完整可用选项以其 `--help` 为准"兜底）。行为上的改变有：`find` 现在会新采（基线说明是 "without collecting a new…"）；截图默认 JPEG；独立截图不再带树；`--raw` 改成只保存；最低系统升到 Android 14；必须显式开会话。除第一条外清单都写了。`find` 会新采不算破坏性变化，可以不列。→ 建议在"动作与阅读选项"一行补上 `wake`。进下一轮，或者在写发布说明时补。
- `reference/ui.md:343` 仍写"只在full/current user0运行"，已在公开指引里删掉的"full mode"在内部 docs 里还留着，源码里已经没有这个模式。进下一轮。
- `architecture.md:137` 写"正文摘要只在真实worker外层超限时产生"。这里说的其实是字段值的摘要（`actualValueSummary` 一类，见 `reference/ui.md:379`），不是 64 KiB 正文摘要，容易与第 14 条混淆。宜改成"字段值摘要"。进下一轮。
- 第 19 条的自动重绑：`reference/connection.md`（提交 `509c23e`）写了"已连接严格超过5秒时，排队一次bind"，与源码一致。同一文件第 205 行"worker 不可用时用 Refresh status 发起重绑"讲的是重绑失败或进程存活太短时的手动路径，两处不矛盾。
- 第 21 条：`reference/ui.md:292` 已改为"管理页使用singleTask，保持为自己任务的根"；第 312 行讲任务列表覆盖面的"也不能枚举任意中间层Activity"按裁决保留，正确。
- `reference/ui.md:7` 关于输出上限、`--raw`、`--print` 的描述与源码和指引一致。

---

## 7. 发现清单（按去向）

**会让调用者做错事的**：没有发现会造成误操作或重复输入的错误陈述。最接近的是 5.2 第 1 条：`fill` 报错里的固定示例 `e15` 被照抄时会作用到另一个节点上。它没有在真实运行中出现过。

**建议合入前顺手改**（都是一行级，不改也不构成挡合入）：
1. `SKILL.md:87` 改成"搜文件或缩小范围，不要整份读"（实测，网页题 2/5）。
2. `jobs-files.md:67` 补全截断的 "Local t..."（源码）。
3. `fill` 参数报错的示例改成回显调用者的引用（复现）。

**进下一轮**：
- 首页加"滚动查找命中贴边时再滚一次"一行（实测，长列表题 4/5，约省 5 个百分点输入）。
- 首页把"过渡中的引用可能过期"写直白；`UiRefRepair.kt:99` 的过期拒绝加上 "run androidctl snapshot"（实测，路线题 2/5）。
- 首页删 `automation` 两行、第 8 行、压缩专题列表（3.4）。
- 后台屏要点并入首页、`displays.md` 改为出问题时再读（设计取舍，需用户裁决，约省 10 个百分点输入）。
- 专题整体删减到约 49 KB（4.4），以及"专题写法"一项的实际落实。
- 帮助：`--print` 说明补摘要；`screenshot --marks` 从这条命令上移除；`scroll-until` 第一句；`--settle` 串用（5.1）。
- 报错：命令组缺子命令时整页帮助压成一行；`notifications get -1|…` 提示加 `--`；Click 缺参数报错给示例；`wait` 两种组合错误的措辞；`--in` 示例回显（5.2）。
- `connection.md` 的 `DaemonVersion` 大小写；`actions.md` 停止原因补"没有路线"；首页"开会话前"的列表补 `apps`。
- 内部 docs：`wake` 选项补进迁移清单；`reference/ui.md:343` 的 full；`architecture.md:137` 的"正文摘要"；真机清单第 20 行宜拆开。

**不必处理**：`fresh` 同时用于两种重采，区分不出来（调用者不需要区分）；指引没写 `find --cached`（不存在的选项，不写是对的）。

**评审方此前结论的复核**：
- 上一轮 74 号报告说"首页足以完成普通任务，缺 `apps` 和树行格式"，本轮已补，后台屏题 5/5 用了 `apps`，这个判断成立。
- 上一轮建议"边缘匹配需要再滚一次，首页加一句"，实施方没有做，本轮长列表题读专题的次数从 3/5 变成 4/5。这条建议仍然成立，而且有了更具体的成本估计。
- 上一轮关于专题写法过度压缩的判断依旧成立。实施方计划页写的"专题完整句已整合审结"与实际差量不符，算是把"已完成"写得过满。
- 第 14 条输出上限的方向是对的：本轮网页题的输出不再出现 180 KB 的一次性打印。但它的收益依赖调用者"搜文件而不是整读"，指引的措辞需要配合，否则一部分成本只是推后了一轮。这一点不是裁决本身的问题。

---

## 8. 未能核实的事项

- 需要设备的行为都没有实跑，包括屏外判定、自动重绑、到期提示、`busy subject=automation`、借用粘贴退出码、摘要交付失败时的退出码。上面引的都是源码推断或实施方的原件，不是我复现的。
- 第 3.3 节的百分点按实施方成本页的分摊口径估算，那个口径本身是近似值（见 `core-results.md:148` 的 A／B 两种情景），不是 token 分项实测。
- 模型为什么在长列表题里读 `actions.md`、在后台屏题里连带读 `reference.md`：原生记录里的推理内容是加密的，只能从前后命令推断，判断依据是间接的。
- 路线、网页、长列表三题的会话由执行框架预先打开，首页"开始任务"一节和不写 `--display` 的默认行为在这 15 次运行里没有被模型实际检验到。
- 我没有逐句通读内部 docs（共 293 KB），第 6 节只抽查了与本轮裁决和指引直接相关的段落。
- 原因码表"每个码一句做法"写得对不对，不在本专题范围内，只核对了指引里提到的码都在表里。
