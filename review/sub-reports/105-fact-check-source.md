# 第七轮评审报告初稿：源码、行为与裁决原文的事实核对

核对对象：`/home/user/androidctl/review/androidctl-r7-candidate-review-f401deb.md`（下称"报告"）。
核对依据：产品源码 `f401deb`（只读克隆）及其历史版本 `fb75afc`、`545d095`；docs `dfe5e45`；第四、六轮决定文件；第六轮评审报告；在主机上实际运行的命令（见各条）。数字和真实任务原件由另一位核对员负责，本文件只在顺带看到时提及。

工作目录：`/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents11/105`（复现脚本 `lockscreen.py`、`findhint.py`，`545d095` 的导出树 `c545/` 和测试日志 `c545-pytest.log`）。

## 摘要

- 总表列了 131 条陈述，另有第一节 15 条重点条目：有出入、说得过强或引用不完整的 9 条（第一节第 1 到 8 条和第 12 条），属实但应补充或写准的 6 条，内部不一致 4 处；总表里约 12 条因需要原件、设备或未重算而无法核对，其余属实。
- **最重要的一处**：下-5 和 §1.1、§5 里"按构造不可能触发""名字一变行标签必变""永远不成立"说得过强。按 `UiSemantics.kt` 的取法，名字取的是"目标自己的 text，否则自己的描述，否则子树里第一段文字或描述（跳过可点击等子节点）"，行标签取的是"子树里第一段非空 text（跳过按钮、可编辑、可勾选的子节点），否则自己的名字，否则第一段描述"。两者只在"取自同一段文字"时必然同变。路线题那个标签页属于这种结构（推断成立），但名字取自目标自己的描述、而子树另有文字等情形下，诊断仍可能命中。建议改成"在它针对的那类结构上不会触发"。
- **第二处**：下-3 第 1 点说 `find` 命中 0 条时才有 `read="snapshot --cached --all"`。实际上 `ui_view.py:896-898` 给 `find` 输出里**每一行**省略记录都加这个提示，命中成功时也有，而且盖掉了原来指向具体引用的 `snapshot e16`（主机端本地复现）。报告说窄了，影响比写的大。
- **评审方自评的两处遗漏**：Maps 专用的"加载中"文字表，源头是第六轮 B2 的修法里"或树里有明显加载文字时"，B2 的事实里正是"正在查找最佳路线"；方向词映射里的"高大于宽"判据，是第六轮附录一原文建议的。报告把这两处写成实施方的做法，§5 的自评表里也没有列。
- §6 第 2 题：`545d095` 的主机测试实测 73 项失败（不是"六七十项"里的笼统数），分布在 7 个测试文件，不是"漏了一个测试文件"这么简单；其中多数是后来那个回归修复时一起更新的输出断言。
- 内部不一致四处：下-19 说指引和帮助"5 处"，展开列了 7 处；§6 第 8 题"20 个混合码里 3 个值得动、其余 15 个"加起来是 18；§1.4 下-12 的"新采超时 1 次"与 §3.3 自己说的"那次不是这条路径"相冲；§6 第 2 题的原因与实测分布不符。
- 挡-1 核实属实：我在 `S/r7/run` 重跑那个测试，`assert 0 == 1` 失败；原因分析、CI 描述、`task ci` 的说法都对。macOS 上按 XNU 管道在大块写入时会扩到 64 KiB 的公开行为推断，大概率同样失败（推断）。
- 下-18 用两版渲染函数实际复现成立：`fb75afc` 头部有 `"display":{"displayState":1,"interactive":false,"keyguardShowing":true,…}`，`f401deb` 只剩 `capture=`。
- §3.6 的 11 条"已核实没有问题"抽查了 7 条（可见性单一定义、64 KiB 判定、原因码 150 个及登记测试、引用编号随机起步、`already registered` 包含匹配、最后一个提交只改主机、`mypy`／`ruff`、`--raw` 组合拒绝），全部属实。
- 对裁决原文的引用：第 1、4、5、6、7、8、10、11、14、15、17、19、22、23、24、25 条和第四轮第 11、18、21 项的引用都准确；"取消内部评审""`find` 总是新采"确实是评审方当时的推荐、用户没有采纳。

---

## 一、有出入、说得过强的条目（按重要性）

| 编号 | 报告位置 | 报告的说法 | 核对结果 | 依据 | 建议改成什么 |
| --- | --- | --- | --- | --- | --- |
| 1 | §1.1 第 6 点；§1.4 下-5；§3.2 下-5"原因"；§5 第一张表第 1 行 | "第 10 条的改名诊断按构造不会在它针对的场景触发"；"目标自己就是列表项时，行标签取自它自己子树里的文字或它自己的名字。所以这类目标的名字一变，行标签必然跟着变，'行标签相同'永远不成立" | **说得过强** | 源码推断。名字：`UiSemantics.kt:88-102`，先取目标自己的 `text`、再取自己的 `description`；两者都空且目标是可点击等独立节点时，取子树里第一段 `text` 或 `description`，并跳过可点击、可编辑等"公开目标"子节点（`UiSemantics.kt:161-178`）。名字是**单个值**，其余值进 `detail`，不存在"多段拼成的名字"。行标签：`UiSemantics.kt:182-228`，从目标往上找最近的列表项；对这一项先用 `RowLabel` 深度优先取第一段非空 **`text`**（只看 `text`，不看描述；跳过类名以 `Button` 结尾、可编辑、可勾选、密码的节点，不进嵌套列表项，`UiSemantics.kt:238-270`），取不到才用这一项自己的名字，再取不到用子树里第一段描述。反例（目标本身是列表项）：①名字来自目标自己的 `description`，子树里另有一个文字子节点，行标签取的是那个子节点的 `text`；只改描述时行标签不变，诊断能命中。②名字来自子树里排在前面的一个图标的描述，行标签来自后面的文字；只改图标描述时同上。③子树第一段文字在一个可点击（非 `Button` 类）子节点上：名字跳过它取后面的文字，行标签却取它；后面的文字变化时同上。路线题那个标签页：同一页上 "Driving mode" 行打印成子节点 `text "Driving mode: 17 minutes"` 加 `text "17 min"`（记录 89 第 4.3 节），说明名字和行标签都取第一段子文字，所以在这个具体场景里结论成立（推断）。 | §1.1："第 10 条的改名诊断在它要解决的场景里不会触发"。§3.2："目标自己是列表项、而名字和行标签取自同一段文字时（目标自己有 `text`，或名字和行标签都取子树里第一段文字；路线题的步行标签页正是这种结构），名字一变行标签必然跟着变，判定不可能成立。列表行和标签页大多是这种结构。名字取自目标自己的描述而子树另有文字等情形下，仍可能命中。"把"永远不成立"改为"在这类结构上不成立"。§5 表第 1 行同样改为"目标是列表项、名字和行标签取自同一段文字时"。 |
| 2 | §3.2 下-3 第 1 点；§1.4 下-3 | "`find` 命中 0 条时，输出里有一行 `omission reason="find" nodes=654 read="snapshot --cached --all"`（`ui_view.py:896-898`）" | **说得过窄**（行号对，触发条件写窄了） | 源码：`ui_view.py:896-898` 对 `find` 输出里**所有**省略记录统一改写 `read`，不看命中数；`reading_view` 对每个不在命中集合里的节点生成 `reason="find"` 的省略（`ui_view.py:770-771`）。主机端本地复现（`agents11/105/findhint.py`）：命中 1 条时输出仍有 `omission reason="find" nodes=1 read="snapshot --cached --all" ref="e16"`——带引用的省略本应提示 `snapshot e16`（`ui_view.py:689-692`），被这一行盖成了 `--all`。 | "`find` 输出里的每一行省略记录（不论命中几条）都提示 `read="snapshot --cached --all"`，连带把原本指向具体引用的 `snapshot 引用` 也改成了整页 `--all`（`ui_view.py:896-898`）。命中 0 条时整页只剩这一行。"修法里相应写成"删掉这一行改写，恢复按引用的读法"。 |
| 3 | §3.2 下-1"路线题上的改善靠的是写死的文案"；§5 第一张表第 2 行 | 把 Maps 专用的"加载中"文字表写成实施方"对固定任务过拟合的直接痕迹"；§5 只把 B2 的错写成"只看到边界变化而标题没变" | **遗漏评审方自己的来源** | 第六轮评审报告 B2（`androidctl-r6-candidate-review-fb75afc.md` §3.2）的修法原文："对可能引起导航的动作，只看到边界变化而标题没变、**或树里有明显加载文字时**，标成'可能还在变化'"；同一条的事实写的是"路线题 5 次里有 3 次在'**正在查找最佳路线**'阶段就拿到'已变化'"。实施方的文字表正是照这两句做的。源码：`UiPostAction.kt:256-268, 283`。 | §3.2 下-1 加一句："加载文字这一半也是我第六轮 B2 修法里要求的（'或树里有明显加载文字时'），例句就取自路线题，实施方照此写死了 Maps 文案；过拟合的源头在我。"§5 表第 2 行改为"B2 的规则写成'只看到边界变化而标题没变、或有明显加载文字'"，后果栏补"加载文字一半被落实成 Maps 专用字符串"。 |
| 4 | §3.3 下-10 | "附录一要求'只公布前后方向的竖向容器接受 `down`、`up`'。实现判断'竖向'的条件之一是'高大于宽'" | **引用不完整**（把评审方建议的判据写成了实现的选择） | 第六轮评审附录一原文："可以对**高大于宽**、只公布前后方向的容器做映射。"源码 `UiScroll.kt:52-55`：`ScrollView`、`GridView`、`collectionInfo` 单列、或 `verticalBounds`（宽大于 0 且高大于宽，`UiScroll.kt:66-71`）四者任一成立即按竖向映射；"高大于宽"与 `collectionInfo` 是"或"的关系，所以 `collectionInfo` 显示多列时形状判据照样生效。 | "附录一（我写的）建议'对高大于宽、只公布前后方向的容器做映射'，实现照做了：`ScrollView`、`GridView`、`collectionInfo` 单列或高大于宽，任一成立就把 `down`／`up` 映成前进／后退（`UiScroll.kt:52-60`）。"并把这一条加进 §5 第一张表：附录一的"高大于宽"判据 → 竖屏上的横向翻页列表会误映射 → 下-10。 |
| 5 | §6 第 2 题背景 | "`545d095`、`1973106`、`642799b` 三个的主机测试是红的（各有六七十项失败），原因是交付那一组的窄范围测试漏了一个测试文件，之后没人跑全量" | **部分有出入** | 主机端本地复现：`git archive 545d095` 导出到工作目录，用 `S/r7/venv7` 跑 `pytest androidctl/tests`（已确认导入的是导出树的源码）：**73 失败、1,866 通过、21 跳过**（`agents11/105/c545-pytest.log`）。失败分布在 7 个文件：`test_observation_files.py` 53、`test_ui_arguments.py` 12、`test_node_actions.py` 4、`test_activity_intents.py`、`test_notifications.py`、`test_text_input.py`、`test_ui_delivery.py` 各 1。其中 `test_observation_files`、`test_node_actions`、`test_activity_intents`、`test_text_input` 四个文件正是 `f401deb`（修回归的提交）一起改的；也就是说，大部分红是测试**正确地**抓到了 `545d095` 引入的回归，只是没人跑。另两个提交没有跑。 | "本轮 5 个提交里，`545d095` 的主机测试实测 73 项失败（`1973106`、`642799b` 按同一原因应同样是红的，未逐个跑），分布在 7 个测试文件；其中大部分是那次回归在输出断言上的表现，其余是没跟上有意改动的断言。原因是各组只跑了窄范围测试，之后没人跑全量。" |
| 6 | §1.4 下-16；附录一最后一条 | "'需要会话'的未发出回执后面跟着 `dispatchStarted=unknown`" | **说得过窄** | 源码 `ui_output.py:95-101`：只要不是只读命令，结果不是 `done`、且数据里没有布尔型 `dispatchStarted`，就打印 `dispatchStarted=unknown`。主机端本地复现：`androidctl --device x click ref15` 输出 `click: not sent; No unique paired device: x; …; code=deviceNotFound; dispatchStarted=unknown`。 | "主机端在发出之前就失败的未发出回执（例如'需要会话'、找不到设备）后面都跟着 `dispatchStarted=unknown`，与'未发出'自相矛盾（`ui_output.py:95-101`）。" |
| 7 | §5"裁决带来的结果"第 14 条 | "作为保险是对的；省成本的目的没有达到" | **略强** | 第 14 条原文的定位是"上限是不依赖成因的防线"；省成本只出现在背景里（"被执行框架截断后仍占了那次运行总输入的约 17%"）。 | "作为防线是对的；背景里期待的省成本没有体现。" |
| 8 | §3.3 下-19 | "`screenshot --help` 仍列出总是被拒的 `--marks`" | **略强** | 主机端本地复现：`screenshot --help` 里 `--marks` 的说明是"Label public REF targets with snapshot --screenshot --marks. Standalone screenshot has no node tree to label."——帮助已经说明不能单独用。 | "`screenshot --help` 仍列出 `--marks`，虽然注明了单独截图不可用，列出来本身没有意义。" |
| 9 | §3.2 下-2 | "`back`、`key`、`home`、`recents`、`open` 取动作后观察时不传入动作前的样本（`SystemActions.kt:56`），连续两次采样相同就算稳定" | **属实，但漏了更关键的一点** | `SystemActions.kt:56`（`back`、`key`、`type`、`home`、`recents`）和 `SystemActions.kt:193`（`open`）都是 `settle(run, null)`。没有基准时 `stableWithoutBaseline` 在两次相同采样后停止（`UiPostAction.kt:223`），而 `classification()` 在 `baseline == null` 时**只能**是"可能还在变化"或"加载中"，永远不会是"已变化"（`UiPostAction.kt:216`）。所以"16 条 `back` 回执全部标可能还在变化"是结构必然，不是偶然。 | 补一句："没有基准时分类只能是'可能还在变化'（`UiPostAction.kt:216`），所以这类回执的这个标记没有信息量；`open` 的同一处在 `SystemActions.kt:193`。" |
| 10 | §3.1 挡-1"后果" | "macOS 上的表现没有环境验证，未知" | 属实；可以补推断 | 公开事实：Linux 管道默认 64 KiB；macOS（XNU）管道起始 16 KiB，单次大块写入时会扩到 64 KiB。测试正文约 55 KB，以一行写出（`buffering=1` 的文本流，大块写直接下到系统调用）。 | "macOS 上没有环境验证；按 XNU 管道在大块写入时扩到 64 KiB 的行为推断，大概率同样一次写完、同样失败。" |
| 11 | §3.3 下-13 第 3 点 | "正常的时限截断在首行写成'the app may be unresponsive'" | 基本属实，触发条件可写准 | 源码：这句话出自 `CollectionReplies.kt:146-150`，条件是"采集截止时还在等一个主节点回复"；再由 `UiPublication.kt:200-206` 在缺口原因为 `collectionLimit` 时拼进 `notice`。长网页按 3 秒采集上限截断时，通常正好卡在等待回复上，所以会出现。 | "采集按时间上限截断、而截断那一刻正在等某个节点的回复时（长网页上的常态），首行附上'the app may be unresponsive'。" |
| 12 | §3.2 下-5"这是谁的错" | "第六轮决定文件第四节第 7 处写了……你据此选了严格判定" | 实质属实，来源可写准 | 决定文件第四节第 7 处原文"行标签是区分'同位置已经换了内容'的字段，判定时不能省"属实。严格判定本身最早出自工作记录 `83-ruling-risks.md` 第 2.7 节"建议的判定"，用户在第二次答复里确认（第 10 条"答复：按严格判定"；`86-final-check.md:219` 对照）。 | 可改为"我在第六轮给出的判定建议（记录 83 第 2.7 节，决定文件第四节第 7 处概括为'行标签……判定时不能省'）要求行标签相同，你在第二次答复里按它定了严格判定"。 |
| 13 | §3.3 下-10 | "`collectionInfo` 显示'一行多列'时不走形状判据。一行。" | 修法可行，但要注意代码结构 | `UiScroll.kt:52-55` 是四个条件的"或"，要加的是"`collectionInfo` 多列时排除 `verticalBounds`"，并且 `GridView` 本身也被当成竖向；改动仍是一两行。 | 可保留，补"（`GridView` 同样无条件按竖向处理，可一并看）"。 |
| 14 | §6 第 1 题 C；§3.2 下-3 | "首页第 87 行""`SKILL.md:87` 写'Read/search that file after host-tool truncation'" | 属实 | `SKILL.md:87` 原文 "Read/search that file after host-tool truncation. `--no-print` keeps saving;" | 无需改。（列在这里是因为题目点名要核） |
| 15 | §6 第 4 题 C | "被拒时新树本来就已经采好并装成了当前样本" | 属实，但有前提 | `UiRefRepair.kt:32-36`：只有首次尝试是"未发出"、错误可修复（`stale` 主体 `ref`、`changed` 主体 `selection`、`identityUnknown`）且预算未用完时，才调 `capture`；`capture` 走 `UiBackend.sample`，在 `UiBackend.kt:410-413` 里 `snapshots.install`。预算已用完或首个错误不可修复时没有新树。 | "旧引用被拒、并且请求还有剩余时间时，新树已经采好并装成了当前样本（`UiRefRepair.kt:36`、`UiBackend.kt:410-413`）。" |

## 二、报告内部不一致

| 编号 | 位置 | 不一致之处 | 建议 |
| --- | --- | --- | --- |
| N1 | §1.4 下-19 与 §3.3 下-19 | §1.4 说"指引和帮助里的 5 处小错"；§3.3 展开的是 7 处：`jobs-files.md:67` 截断、`DaemonVersion` 大小写、首页漏 `apps`、停止原因漏"没有路线"、`fill` 报错示例固定 `e15`、`screenshot --help` 的 `--marks`、`README.md` 的"简短结果"。内部 docs 3 处、实施方记录 6 处是对的。 | 改成"7 处"，或把 `README.md` 和 `--marks` 另列。 |
| N2 | §6 第 8 题 A | "剩下的 20 个混合码里只有 3 个值得动（已含在下-17），其余 15 个可以永久不动"：3 + 15 = 18，不是 20。记录 94 第 1 节写的是"其余 16 个"。 | 按记录 94 重数后统一，例如"20 个里 3 个值得动，其余 17 个……"或照记录 94 的口径。 |
| N3 | §1.4 下-12"真实任务"一列 与 §3.3 下-12 第 2 段 | 一列写"新采超时 1 次（设备检查）"，正文说"设备检查里那次 `find` 超时不是这条路径，而是上一条 `back` 未稳定引起的重采"。读者会以为 0 条自动新采在设备上失败过一次。 | 一列改为"0 条自动新采 0 次；另一条重采路径超时 1 次（设备检查）"。 |
| N4 | §6 第 2 题 与 实测 | "漏了一个测试文件"与 `545d095` 上 7 个文件失败的实测分布不符（见第一节第 5 条）。 | 同第一节第 5 条。 |

§1.4 与 §3 的对应：挡-1 在 §3.1；下-1 到下-9 在 §3.2；下-10 到下-19 在 §3.3；下-20 到下-22 在 §3.4。22 条一一对应，编号没有错位、没有漏条。§1.5 的 9 道题与 §6 第 1 到 9 题题目、分组（3 + 6）一致。附录一"下-16 的六处"正好 6 条，与 §1.4 下-16 一致。附录二 88 到 103 共 16 份，与 `sub-reports/` 目录实有文件一致。

## 三、引用裁决的核对

| 报告提到的裁决 | 报告的引用 | 原文 | 结果 |
| --- | --- | --- | --- |
| 第 1 条 | "整理成 6 到 10 个提交，接受只有最后一个提交能运行" | "不压成一个提交；接受只有最后一个提交能运行……重新整理成 6 到 10 个提交" | 准确 |
| 第 4 条 | "真机能跑通"改为发布条件，由用户在发布前做；清单在内部 docs | 同 | 准确 |
| 第 5 条 | 只跑候选，四道题各 5 次，基线沿用 | 同 | 准确 |
| 第 6 条 | 第一页分"挡合入""进下一轮"；挡合入的改完后评审方针对性复核；改了默认输出格式才重跑受影响那道题 | 同（"答复"第 2 点） | 准确 |
| 第 7 条 | "上一轮我把这叫'冻结'，你没有采纳" | "报告建议把六块设计'冻结'，用户没有采纳" | 准确 |
| 第 8 条 | 封闭清单 | 同；附录一"被第 8 条整体并入封闭清单"也对 | 准确 |
| 第 10 条 | 窗口、来源节点编号、角色、行标签、行列位置都相同、只有名字不同；改动第四轮第 11 项 | 同 | 准确 |
| 第 11 条 | 读缓存、0 条时新采；评审方推荐"总是新采"，用户没采纳 | 第六轮报告 §6 第 5 题 A（推荐）"`find` 总是新采"；用户选 B | 准确 |
| 第 13 条 | 可见性改在设备端 | 同 | 准确 |
| 第 14 条 | 输出上限 | 原文定位是"防线"（见第一节第 7 条） | 基本准确，"省成本的目的"略强 |
| 第 15 条 | "落地之后带上界面树"；评审方列的可删项是"连接号、起止时间戳、样本年龄和非 `--boxes` 时的窗口几何" | 第 15 条落实要点"`open` 的启动细节只在失败时打印；落地之后带上界面树"；第四节第 9 处"可以删的是连接号、起止时间戳、样本年龄和非 `--boxes` 时的窗口几何" | 准确 |
| 第 17 条 | 风险提示默认"截断"是节点数或字节上限 | 第六轮报告第 10 题背景"界面树碰到节点数或字节上限时"；第四节第 8 处同 | 准确 |
| 第 19 条 | 每次意外退出都重绑，最短存活时长由实施方定；记录附在下一条"需要会话"的报错上 | 同 | 准确 |
| 第 22、23、25 条 | 各组合 | 同；本机复现 `snapshot --raw --all`、`find --raw --context 2` 在参数阶段被拒，`session open --replace` 无 `--display` 被拒 | 准确 |
| 第 24 条 | 156 减到 150；"这一条是我推动的" | 156→150 属实；背景"评审报告 §2.3、§2.4 第 3 条" | 准确 |
| 第 25 条 | "我当时推荐不做" | "评审方当时推荐不做，用户选了做" | 准确 |
| 第 3 条 A2 | "接管无效，等对方退出"是你定的文字 | "文字明确写'接管无效，等对方退出'" | 准确 |
| 第四轮第 11 项 | "被拒绝的动作不再重印整页" | 第四轮第 11 项第 1 点原文同 | 准确 |
| 第四轮第 18 项 | 三条底线 | 原文三条；"真机能跑通"由第六轮第 4 条改为发布底线 | 准确 |
| 第四轮第 21 项 | "不强求中间提交可用" | "不强求中间的提交各自可用" | 准确 |
| 第四轮第 24 项 | 题目未直接引用 | — | 不涉及 |
| 决定文件第四节第 7 处 | "行标签是区分'同位置已经换了内容'的字段，判定时不能省" | 逐字相同 | 准确（来源补充见第一节第 12 条） |
| §5"你没有采纳"两处 | 取消逐改动内部评审；`find` 总是新采 | 第六轮报告 §6 第 7 题 A（推荐）含"取消逐项内部评审和取证文档"，用户"过程规格维持现状"；第 5 题 A（推荐）"总是新采"，用户选 B | 准确 |
| §3.5 成本页第一句 | 你要的是"多出的时间在模型一侧，与产品无关"；实施方按你后来的指示改写 | 决定文件第三节丁原文同；`PKG/context/USER_SCOPE.md:25`"额外耗时落在 CLI 之外不能直接证明与产品无关"；原话出自第六轮报告 §4.1 | 准确 |
| 第六轮附录一 | 首页示例改 `fill --submit`"依据只来自网页题" | 附录一原文"候选模型 8 次提交全部选了 `fill --submit`"，没有写出自哪道题 | 无法核对（要回查上一轮原件） |
| 第六轮 B2、附录一方向词 | 见第一节第 3、4 条 | — | 引用不完整 |

## 四、逐条核对总表（属实与无法核对）

依据性质：源码＝读 `f401deb` 源码；复现＝在主机上实际运行；裁决＝对照决定文件或第六轮报告。

| 编号 | 报告位置 | 报告的说法 | 结果 | 依据 |
| --- | --- | --- | --- | --- |
| 1 | §1.1 第 1 点 | 25 条裁决都有对应实现，十处风险提示照顾到 | 无法逐项核对（抽查第 7、9、10 处属实） | 风险提示第 7 处：严格判定已实现（`UiRefRepair.kt:49-69`）；第 9 处第 15 条可删项：头部已删连接号等（复现）；第 10 处：`--replace` 缺 `--display` 拒绝（复现） |
| 2 | §1.1 第 1 点 | 三项"做了但没达到目的"：B2、`open` 带树、改名诊断 | 属实（源码层面） | `UiPostAction.kt:244-284`；`ActivityLaunch.kt:208-236`；`UiRefRepair.kt:49-69` |
| 3 | §1.1 第 2 点 | 本轮 Kotlin 和 Python 主源码差量没有挡合入的错误 | 无法全量核对；我看过的部分未见反例 | 差量 41 个文件 +986/−554 |
| 4 | §1.1 第 5 点 | 净增约 430 行源码、约 1,850 行测试 | 属实 | `git diff --stat fb75afc f401deb`：主源码 +986/−554＝432；主机测试 +1208/−268、Android 测试 +1005/−94，合计净增 1,851 |
| 5 | §1.1 第 5 点 | 出过一个回归（已修） | 属实 | `545d095` 把 `ui_output.py:58` 写成 `result.result != "done"`，`f401deb` 改为 `… or result.exit_code() != 0` |
| 6 | §1.2 | 第四轮第 18 项三条底线，真机一条已由第 4 条改为发布条件 | 属实 | 裁决原文 |
| 7 | §1.4 挡-1 | `"large body " * 20000` 改为 `* 5000`（220 KB → 55 KB） | 属实 | `git diff fb75afc f401deb -- androidctl/tests/test_ui_delivery.py` |
| 8 | §1.4 挡-1、§3.1 | 测试在 Linux 上必然失败 | 属实 | 复现：`pytest androidctl/tests/test_ui_delivery.py -k test_print_broken_os_pipe…` → `assert 0 == 1` |
| 9 | §3.1 | 读端读到正文头后关管道，期望写端出错、退出码 1 | 属实 | `test_ui_delivery.py:832-859` |
| 10 | §3.1 | Linux 管道缓冲区 64 KiB，55 KB 能整段写入 | 属实（公开事实） | 默认 16 页＝65,536 字节 |
| 11 | §3.1 | 实施方完整质量检查只在 Windows 上跑 | 无法核对（实施方环境） | — |
| 12 | §3.1 | `AGENTS.md` 里 `task ci` 对 Linux 和 macOS 只做类型检查 | 属实 | `AGENTS.md`"`ci` also checks Linux/Darwin type branches"；`androidctl/Taskfile.yml:27-32`：`task quality` 加 `mypy --platform linux`／`darwin` |
| 13 | §3.1 | `python-posix` 在 Ubuntu（3.12、3.14）和 macOS 上跑 `python -m pytest`，推送到主分支时触发 | 属实 | `.github/workflows/quality.yml:3-33`；另外拉取请求和打标签也触发 |
| 14 | §3.1 | 修法：只改测试，不改产品 | 属实（失败与产品无关） | 产品代码路径在 Linux 上行为正确，只是测试前提不成立 |
| 15 | §1.4/§3.2 下-1 | 点击后回执标"已变化"，B2 规则只认"只有边界变化" | 属实 | `UiPostAction.kt:244-252`、`270-281`：导航类命令在分类为"已变化"且 `onlyGeometryChanged`（窗口目录去掉边界后相同、内容签名不含边界相同）时才改成"可能还在变化"；焦点、动作字段在签名里 |
| 16 | §3.2 下-1 | 导航类命令包括哪些 | 属实 | `UiPostAction.kt:97-98, 158`：`click`、`long-click`、`action`、`key`、`back`、`home`、`recents`、`open`、`ime`，以及 `fill --submit` |
| 17 | §3.2 下-1 | 加载中文字表五个字符串，`UiPostAction.kt:284` | 属实 | `UiPostAction.kt:283`：`loading`、`正在加载`、`正在查找最佳路线`、`finding the best route`、`finding best route`（比较前去空白、去尾部句点和省略号、转小写） |
| 18 | §3.2 下-1 | 后三个是 Maps 专用 | 属实（判断） | 字面即路线文案 |
| 19 | §3.2 下-1 | 与 `UiSemantics.loading` 是两套定义 | 属实 | `UiSemantics.kt:397-402` 只认可见的不定进度 `ProgressBar`，已在 `UiPostAction.kt:192, 215` 使用 |
| 20 | §3.2 下-1 | 上一轮 5 次里 4 次 | 属实 | 第六轮 B2 事实原文 |
| 21 | §3.2 下-2 | 系统按键回执没有动作前样本 | 属实 | `SystemActions.kt:56`、`193` |
| 22 | §3.2 下-2 | 连续两次采样相同就算稳定 | 属实 | `UiPostAction.kt:64, 223` |
| 23 | §3.2 下-3 第 2 点 | 超限摘要固定写 `--depth 2`（`ui_client.py:888`） | 属实 | `ui_client.py:887-889` |
| 24 | §3.2 下-3 第 3 点 | `SKILL.md:87` 原文 | 属实 | 见第一节第 14 条 |
| 25 | §3.2 下-3 第 4 点 | `--all` 下无名平台标准动作全部打印 | 属实 | `ui_view.py:117-126`：省略判断带 `not all_nodes`；`all_nodes` 时未命名的动作以 `action-xx` 打印（`ui_view.py:127-136`） |
| 26 | §3.2 下-3 | `find --all` 也走这条路径 | 属实 | `observation_nodes(all_nodes=…)` → `project` → `attributes` → `compact_actions(all_nodes=…)`（`ui_view.py:216-220`） |
| 27 | §3.2 下-3 | 默认输出里这类字段清零 | 属实（源码） | 默认模式下平台标准动作未命名的一律跳过 |
| 28 | §3.2 下-4 | 动作后观察只在 1.5 秒窗口里等 | 属实 | `UiPostAction.kt:157` `DEFAULT_WINDOW_MS = 1500` |
| 29 | §3.2 下-4 | 实施方把观察挪到落地确认之后 | 属实 | `ActivityLaunch.kt:208-218`（`f401deb`）对比 `fb75afc` 的 `perform`（先 `observe` 后确认） |
| 30 | §3.2 下-4 | 确认落地看的是任务表 | 属实 | `ActivityTasks.kt:104-133` 用 `getTasks` |
| 31 | §3.2 下-5 | 实现照做了严格判定（`UiRefRepair.kt:49-69`） | 属实 | `current.copy(name = original.name) == original` 且恰好一个、整棵树完整 |
| 32 | §3.2 下-5 | 行标签取法在 `UiSemantics.kt:181-237` | 属实（行号） | `row` 在 182-212，`nearestRowLabel` 214-228，`RowLabel` 230-271 |
| 33 | §3.2 下-5 | 测试里行标签是手写的 `"Row"` | 属实 | `UiRefRepairTest.kt:14`（另有 `"Other row"` 作反例，24 行） |
| 34 | §3.2 下-5 | 这一行只陈述、从不自动派发 | 属实 | `UiRefRepair.kt:38-43` 直接返回拒绝 |
| 35 | §3.2 下-5 | 前两轮 7 次 | 属实 | 第六轮 B1 原文"共 7 次" |
| 36 | §3.2 下-6 | 平台返回 false 时设备端不带原因（`UiActions.kt:900`） | 属实 | `UiActions.kt:900` `outcome(command, callback.classify(), data=…)` 不传原因；`ActionReceipt.kt:20-28` 返回 `false` 归为失败 |
| 37 | §3.2 下-6 | 主机端把采样说明印在首行 | 属实 | `ui_output.py:102-105`：结果不是 `done` 且数据里有 `samplingStopped` 就印，不看失败是否来自观察侧 |
| 38 | §3.2 下-6 | 平台返回 false 通常意味着没执行 | 属实（平台语义） | — |
| 39 | §3.2 下-6 | 滚动查找"往回滚"的建议 | 属实（来源记录 90；源码里建议拼接在 `UiActions.kt:1017-1018` 的 `directionAdvice`） | — |
| 40 | §3.2 下-7 | `show-on-screen` 在默认省略集合里 | 属实 | `ui_view.py:55-69` 含 `16908342`（`node_facts.py:59` 对应 `show-on-screen`） |
| 41 | §3.2 下-7 | 首页"Use `reveal REF` when advertised" | 属实 | `SKILL.md:62` |
| 42 | §3.2 下-8 | 附着进行中回执"Worker is unavailable; check the device App" | 属实 | `WorkerConnection.kt:274`；附着中未超时时 `readFailure()` 返回 `lastFailure`，新尝试开始时被清空（`WorkerConnection.kt:37-46, 133-137`） |
| 43 | §3.2 下-8 | 码表让调用者请用户点刷新 | 属实 | `reason-codes.json:150` "Ask the user to use Refresh status in AndroidCtl to attach a worker." |
| 44 | §3.2 下-8 | 自动重连期间同样回执 | 基本属实 | 在断开回调到新尝试开始之间，文字是"Worker disconnected; use Refresh status on the device"（`WorkerConnection.kt:103`），同样指向用户 |
| 45 | §3.2 下-8 | 产品知道进程正在绑定 | 属实 | `Attempt.attaching` |
| 46 | §3.3 下-10 | `UiScroll.kt:40-71` 的映射 | 属实 | 见第一节第 4 条 |
| 47 | §3.3 下-10 | 非标准 ViewPager 类、只公布前进后退、没有 `collectionInfo` 单列的整屏高横向列表会把 `down` 映成前进 | 属实 | `UiScroll.kt:46` 只认 `androidx.viewpager.widget.ViewPager`；其余走 52-60 |
| 48 | §3.3 下-10 | 有方向动作时不走映射 | 属实 | `UiScroll.kt:25`（请求的方向已公布直接用）、`34`（对应翻页已公布用翻页）、`35`（公布了任何上下左右或翻页动作就返回空） |
| 49 | §3.3 下-10 | 回执 `Selected` 行写出实际动作 | 属实（源码推断） | `UiRun.kt:71-74` 写 `selected`；`ui_output.py:571-593` 在请求动作与所选动作不同时打印 `Selected` |
| 50 | §3.3 下-10 | 上一轮路线题 2 次因方向词被拒 | 属实 | 第六轮附录一原文 |
| 51 | §3.3 下-11 | 首次确认之后观察结束时又确认一次，要求目标仍在顶层 | 属实 | `ActivityLaunch.kt:221-236` 再调 `confirmLanding`；`ActivityTasks.kt:110-123` 要求可见任务的顶层 Activity 属于目标包和 UID |
| 52 | §3.3 下-11 | 对话框出现时从完成改成结果未知并等满时限 | 属实（源码推断） | `confirmLanding` 循环里非 `observed` 时置 `UNKNOWN` 并 `pause()`，直到预算用完（`ActivityLaunch.kt:335-357`）；对话框是否改变任务顶层取决于它在哪个任务里（未知） |
| 53 | §3.3 下-11 | 一启动就弹权限框时首次确认耗尽时限，比上一轮差 | 属实（源码推断） | `fb75afc` 先 `observe` 后确认；`f401deb` 先确认，耗尽后 `observe` 的采样循环不进入 |
| 54 | §3.3 下-11 | 结果未知回执带"不要自动重复" | 无法核对（未查指引原句） | — |
| 55 | §3.3 下-12 | 采集时间上限 3 秒 | 属实 | `UiBudget.kt:68, 111` `COLLECT_MS = 3000L` |
| 56 | §3.3 下-12 | 提前失败只认节点数／字节上限 | 属实 | `UiObservation.kt:51-63` `treeLimits` 只返回 `nodeLimit`、`treeByteLimit`；时间截断的原因是 `collectionLimit`（`CollectedChildren.kt:11`、`TreeCollector.kt:289-290`），在 `UiCondition.kt:445`、`UiActions.kt:922`、`UiSelector.kt:90` 都不被识别 |
| 57 | §3.3 下-12 | `find` 默认时限 5 秒，两次请求共用 | 属实 | `find --help` "[default: (5s)]"（复现）；`ui_client.py:612-634` 用同一个 `UiClient(context, deadline)` |
| 58 | §3.3 下-12 | 新采失败时回执写明"缓存 0 条，新采失败" | 无法核对到原句 | `findRefreshed` 标记在 `ui_client.py:637` |
| 59 | §3.3 下-13 | 首行 `missing=2` 只数缺口记录条数 | 属实 | `ui_output.py:220-228` 用 `missingCount`，即 `len(tree["missing"])`（`ui_client.py:842-846`）；头部 `ui_view.py:916-920` 同样计条数 |
| 60 | §3.3 下-13 | "the app may be unresponsive" 的出处 | 属实 | 见第一节第 11 条 |
| 61 | §3.3 下-14 | `ui_view.py:372` 的条件 | 属实 | `if not all_nodes and not search and not ref and fact(node, "visible") is False`：只要给了引用就保留不可见节点，不看引用本身是否可见 |
| 62 | §3.3 下-14 | 摘要里的缩小写法不带那个引用 | 属实 | `ui_client.py:888` 固定文字 |
| 63 | §3.3 下-15 | 重连提示只在"需要会话"的回复上取出并清掉 | 属实 | `WorkerConnection.kt:155-175`，`recoveryNotice` 只在 `sessionRequired` 时调用 |
| 64 | §3.3 下-15、§2.4 | 最短存活 5 秒，单调时钟 | 属实 | `WorkerConnection.kt:20, 104, 293`（`SystemClock::elapsedRealtime`，`MIN_CONNECTED_MS = 5000L`） |
| 65 | §3.3 下-15 | 没有重连次数上限 | 属实 | `onServiceDisconnected` 只看 `expectedStop` 和存活时长，无计数 |
| 66 | §3.3 下-15 | 每圈至少约 6.5 秒 | 估算，无法从源码精确核对 | 5 秒加绑定耗时 |
| 67 | §3.3 下-15 | 自动化登记被占时回执"接管无效，等对方退出"是用户定的文字 | 属实 | 第 3 条 A2 答复 |
| 68 | §2.4 | 预期退出标记挂在每个附着对象上 | 属实 | `Attached.expectedStop`（`WorkerConnection.kt:226, 288`） |
| 69 | §2.4 | 回调和绑定都在主线程串行 | 基本属实 | `bind` 走 `binding.mainExecutor`（`WorkerConnection.kt:116-117`）；断开回调由 Shizuku 在主线程回调（未逐行核 Shizuku 13.1.5） |
| 70 | §2.4 | 这部分代码本轮变短 | 属实 | `WorkerConnection.kt` +66/−95 |
| 71 | §3.5 | 退出检测只走"服务断开"回调，去掉了死亡通知 | 属实 | `fb75afc:WorkerConnection.kt:83-88` 每个附着对象 `linkToDeath`；`f401deb` 只在强制停止的等待里用（`WorkerConnection.kt:229-254`） |
| 72 | §3.3 下-17 | `Sessions.kt:307` 仍写"take a new snapshot"，与码表相反 | 属实 | `Sessions.kt:307`；`reason-codes.json:112` 写"Inspect status after a session changes…" |
| 73 | §3.3 下-17 | `screenshot 引用` 空边界或越界报 `arguments`（`ImageRegion.kt:154-181`） | 属实 | `fromRef` → `rectangle`／`within`，抛 `Rejected("arguments", …)` |
| 74 | §3.3 下-17 | 设备端已有 `certificateClock`、`certificateExpired` | 属实 | `reason-codes.json:20-21`；`tlsIdentity` 一行混三种下一步（134 行） |
| 75 | §3.3 下-17 | `connectionBusy` 码表说成"另一个会话" | 属实 | `reason-codes.json:30` "Wait for the other session's request…" |
| 76 | §3.3 下-17 | `wait` 最长 60 秒 | 属实 | `--timeout` 帮助"1ms–60s"（复现） |
| 77 | §3.3 下-18 | 锁屏熄屏时上一轮头部显示状态，本轮只剩采集编号 | 属实 | 复现：`agents11/105/lockscreen.py` 分别用 `S/r6/venv6`（`fb75afc`）和 `S/r7/venv7`（`f401deb`）渲染同一棵 `displayState=1`、`interactive=false`、`keyguardShowing=true`、`deviceLocked=true` 的树：前者输出 `observation {…"display":{"displayState":1,"interactive":false,"keyguardShowing":true,"keyguardSecure":true,"deviceLocked":true}…}`，后者头部只有 `androidctl capture="w:7"`；`f401deb` 主机端源码里已没有 `keyguard`／`deviceLocked`／`displayState` 字样 |
| 78 | §3.3 下-18 | 不在评审方给第 15 条列的可删项里 | 属实 | 决定文件第四节第 9 处 |
| 79 | §3.3 下-19 | `jobs-files.md:67` 句子截成"Local t..." | 属实 | 原文 "Blocked kernel I/O is not forcibly interrupted. Local t..." |
| 80 | §3.3 下-19 | `connection.md` 写成 `DaemonVersion` | 属实 | `connection.md:125`；主机码为 `daemonVersion`（`local_call.py:146`） |
| 81 | §3.3 下-19 | 首页"开会话前"的列表漏 `apps` | 属实 | `SKILL.md:21` "Open explicitly before UI, notifications, clipboard, shell or file starts." |
| 82 | §3.3 下-19 | 滚动查找停止原因漏"没有路线" | 属实 | `actions.md:90-91` 列"step limit, incomplete observation, deadline, platform refusal or changed container" |
| 83 | §3.3 下-19 | `fill` 缺参数报错示例固定 `e15` | 属实 | 复现：`androidctl --device x fill e123456789` → "…for example, androidctl fill e15 hello…" |
| 84 | §3.3 下-19 | `README.md`"每条命令只打印简短结果" | 属实 | `README.md:27-28` "each command prints a short result" |
| 85 | §3.3 下-19 | 迁移清单漏 `wake` 的两个选项 | 无法独立核对（采信记录 98 的 `--help` 对比） | — |
| 86 | §3.3 下-19 | `reference/ui.md:343` 还写"full" | 属实（字面） | "只在full/current user0运行"；公开指引里原句"Both need full mode, current user0"已改为"Both require the current user to be user 0"（`notifications.md:17`） |
| 87 | §3.3 下-19 | `architecture.md:137`"正文摘要" | 属实 | "正文摘要只在真实worker外层超限时产生" |
| 88 | §3.3 下-19 | 实施方记录里 6 处要更正的说法 | 无法核对（属材料包） | — |
| 89 | 附录一 | `find` 判断是否新采的代码没有异常保护（`ui_client.py:618-634`） | 属实（行号与结构） | 618-626 的 `ui_view.project`／`find` 不在 `try … except (Fault, KeyError, TypeError)` 内；结果从失败变未知的后果采信记录 103 的复现 |
| 90 | 附录一 | 带标签的平台区段动作被丢（`ui_view.py:117-126`） | 属实 | 第二个条件 `platform_action and identifier not in ACTION_NAMES` 不看标签 |
| 91 | 附录一 | 交付错误被"Node observation is unavailable"盖住（`ui_client.py:799-800`） | 属实（行号） | 树为空且带引用或查询时直接赋值 |
| 92 | 附录一 | `fill` 完成退出码非零仍吞清理指引（`ui_output.py:60-67`） | 属实（行号与结构） | `fill` 数据存在时附加原因要求 `reason == error.message` 且不等于各步原因 |
| 93 | 附录一 | `--raw` 冲突时写法一律是 `snapshot --cached --raw` | 属实 | 复现：`find foo --raw --context 2` → "use androidctl snapshot --cached --raw" |
| 94 | 附录一 | 直接敲 `session`、`clipboard` 把帮助压成一行约 930 字节 | 属实 | 复现：930、926 字节，1 行 |
| 95 | 附录一 | `--print` 帮助没提 64 KiB | 属实 | `snapshot --help` 的 `--print` 一行未提上限 |
| 96 | 附录一 | 写错形状的引用要到设备端才被拒 | 属实 | 复现：`click ref15` 在本机越过参数检查，报的是设备查找错误 |
| 97 | 附录一 | 目标不可见时 `notReady` 等满整个时限 | 属实 | `UiActions.kt:1005-1026`：`routeUnavailable` 只在目标可见时提前抛，其余循环到预算用完 |
| 98 | 附录一 | `test_ui_reading_delivery.py:167` `"fresh" in stdout` | 属实（行号） | 167 行 |
| 99 | 附录一 | 原因码登记测试会漏掉几种写法 | 无法核对（采信记录 94 的实测） | 测试本身 7 项在本机通过 |
| 100 | §2.2 | 可见性只有一处实现、6 个读点共用 | 属实 | `NodeFacts.kt:185-192`；`NodeFacts.read` 在 `NodeFacts.kt:113` 调它，`TreeCollector.kt:191` 直接调用；读点 `TreeCollector.kt:191, 280`、`FreshNodes.kt:26, 42, 75` 等 |
| 101 | §2.2 | 后台屏长网页上平台报可见但在屏外的节点上一轮四千多个 | 属实（裁决背景 4,426 个） | 第 13 条背景 |
| 102 | §2.3 | 第六轮并掉 6 个码 | 属实 | 第 24 条：156 → 150 |
| 103 | §2.3 | 改一条要在设备端、契约、主机、首行、测试、指引六处同步 | 属实（判断） | — |
| 104 | §2.4 | 借用粘贴两处说法相反已统一 | 属实 | `input.md:61-62` "Confirmed PASTE with a matching local effect reports done even if restoration fails (exit 1) or remains unknown (exit 3)" |
| 105 | §2.4 | 输出上限判断进了补存、主结果、退出码三处 | 无法逐处核对；判定处属实 | `ui_client.py:738` |
| 106 | §2.4 | "你上一轮决定保留这道工序是对的，我当时建议取消是错的" | 属实（裁决） | 第 6 条、第六轮 §6 第 7 题 |
| 107 | §3.5 | `open` 先确认、再采树、最后再确认 | 属实 | `ActivityLaunch.kt:208-236` |
| 108 | §3.5 | 坐标触摸前要求目标窗口在窗口列表里 | 无法核对（未读该处） | — |
| 109 | §3.6 | 身份字段不含可见位和边界 | 属实 | `UiSemantics.kt:320-334` 六项 |
| 110 | §3.6 | 输出上限 65,536 字节，按 UTF-8，恰好等于照打 | 属实 | `ui_client.py:45, 738`（`> DEFAULT_BODY_BYTES`） |
| 111 | §3.6 | 150 个原因码双向一致；旧码无残留 | 属实 | `reason-codes.json` 150 行、150 个不同码；`test_reason_codes.py` 7 项本机通过；`identityChanged`、`noPublishedSnapshot`、`internalError`、`messageLimit` 在产品仓库里 0 处 |
| 112 | §3.6 | 3 处归位做了 | 部分属实 | `scroll` 无路线改报 `routeUnavailable`（`UiActions.kt:1021-1022`）；其余两处未核 |
| 113 | §3.6 | 每个进程的引用编号随机起步 | 属实 | `UiRefs.kt:8` `100_000_000L + SecureRandom().nextInt(900_000_000)` |
| 114 | §3.6 | 靠消息包含 `already registered` 识别 | 属实（源码）；Android 14–16 文案未重核 | `AutomationCleanup.kt:36` `IllegalStateException` 且 `contains("already registered")` |
| 115 | §3.6 | 最后那个提交只改主机侧文件 | 属实 | `git show --stat f401deb`：只有 `androidctl/` 下 6 个文件 |
| 116 | §3.6 | `mypy`、`ruff` 通过 | 属实 | 复现：`mypy --cache-dir=/dev/null` "Success: no issues found in 33 source files"；`ruff check`、`ruff format --check` 通过 |
| 117 | §3.6 | 约 60 种参数错误不漏回溯 | 部分复现 | 我跑的 6 种都是一行回执，退出码 2 |
| 118 | §3.4 下-20 | `result_line` 从 245 行到约 280 行 | 属实 | AST 计数：`fb75afc` 245 行，`f401deb` 280 行 |
| 119 | §3.4 下-20 | 回归从 `545d095` 起存在，修它只改 1 行源码、约 400 行测试 | 属实 | `git show --stat f401deb`：源码 1 行，测试 +235/−172（约 407 行） |
| 120 | §3.4 下-21 | docs 从 290 KB 涨到 302 KB；`reference/ui.md` 约 100 KB | 属实 | `git ls-tree -l` 合计：`66bb380` 290,239，`dfe5e45` 301,632；`ui.md` 100,509 字节 |
| 121 | §3.4 下-21 | 主机参数预检少了 39 行 | 无法核对（未重算） | — |
| 122 | §5 | 第 25 条评审方推荐不做 | 属实 | 裁决原文 |
| 123 | §5 | 附录一二十来条小项被第 8 条并入封闭清单 | 属实 | 第 8 条；附录一 13 段、约二十项 |
| 124 | §6 第 1 题背景 | 挡合入条目改完后评审方复核；改默认输出格式才重跑 | 属实 | 第 6 条答复 |
| 125 | §6 第 2 题 | 第 1 条"接受只有最后一个提交能运行" | 属实 | 裁决原文 |
| 126 | §6 第 2 题 | 版本号重装主机包的坑"已写在决定文件" | 属实 | 第四节第 5 处 |
| 127 | §6 第 4 题 C | 会改动第四轮第 11 项 | 属实 | 第四轮第 11 项第 1 点 |
| 128 | §6 第 4 题 A | Maps 是否保留来源节点编号未知 | 属实（未知） | 记录 89 第 4.4 节同样列为未知 |
| 129 | §6 第 6 题 A | `--all` 现在同时打开"隐藏和屏外节点、超过 20 条的匹配、无名动作" | 属实 | `ui_view.py:372`、`857-858`、`117-126` |
| 130 | §9 | 主机侧在 Linux、Python 3.13 上运行；CI 用 3.12、3.14 | 属实 | `S/r7/venv7/bin/python -V` 3.13.18；工作流矩阵 |
| 131 | 附录二 | 88–103 共 16 份，专题对应 | 属实 | `sub-reports/` 文件名 |

## 五、没能核对的事项

- 依赖设备或原件的现象部分（各条"实测"的次数、首行原文、`snapshot --window 17; snapshot --window 10` 等），交给另一位核对员。
- Android 14–16 上 `already registered` 文案、Shizuku 13.1.5 的回调线程，没有重新对照上游。
- `1973106`、`642799b` 两个提交的主机测试没有跑；按失败文件在 `f401deb` 才修正推断同样是红的。
- 第六轮附录一"首页示例改动的依据只来自网页题"，要回查上一轮原件。
- 3 处原因码归位中的 `mappingUnknown → userScope`、后台屏截图改报 `screenshotUnavailable` 没有逐处读源码。
