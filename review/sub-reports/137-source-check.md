# 任务 137：对照源码核对决定文件里的位置

核对对象：`/home/user/androidctl/review/androidctl-r8-decisions.md`（下称"决定文件"）。源码是产品 `409e853` 的只读检出，工作树干净。下文路径前缀与决定文件相同：设备端 `android/app/src/main/java/com/rainng/androidctl/`，主机端 `androidctl/src/androidctl/`，指引 `skills/androidctl/`。"实测"指我打开记录或运行主机端看到的结果，"源码"指读代码得出的判断。

## 结论

1. 决定文件列出的源码和指引行号大多准确。有 3 处不准：第八节第 7 条的 `ui_output.py:120-126` 实际在 `:128-134`；缺-14 列的 `WorkerConnection.kt:120-122` 恰好是唯一已经写对的那句，要改的是 `:43`、`:88`、`:334-335`；缺-13 的 `--any-of` 示例不在 `app.py`，而在 `ui_commands.py`，反斜杠转义来自首行渲染。
2. 有几处位置没列全，其中两处影响行为。第一处是缺-1：`ActivityLaunch.kt:404-420` 的 `recordOpenedTasks` 也按"最上面界面的包名"判断。如果不跟着改，后台屏上带权限框打开的任务不会记成"本会话打开的"，关闭后台屏时会被归为来历不明的任务。第二处也是缺-1：报完成时主机端首行不打印 `landing`（`ui_output.py:76-80`），所以"写明被什么盖住"必须改主机端。
3. 取-1 的新界面树要到 `settle` 之后才拿得到（`UiActions.kt:603`），而核对结论在 `:600-601` 已经定了，所以新事实只能附加，不能改结论。实测三次里，第一次（`r7-new-07` 第 9 条命令）回执没有界面树。另外两次的新开关都没有名字，资源编号都是 `switch_widget`，所在行没有标签，行里的子项叫 "Use Dark theme"，窗口编号都换了。
4. 取-2 只要不改各步骤的结果，按 `FillProgress.kt:97` 的合并规则整条命令仍是"结果未知"。如果把填写那一步改成完成而提交不发，结果会变成"失败"。
5. 缺-5：按"去掉位置分之后领先不到 2 分就拒绝"来实现，现有 21 组样本里报"选中"的 10 组都不受影响，也没有测试依赖位置分定胜负。
6. 取-5 有一个坑：放宽读取的截止时间在读第一份样本之前就要定，这时还不知道"同一个页面"。在 Chrome 里点链接，窗口不变但页面变了。所以能在读之前判断的只有"窗口没变"，而这样做会取消多数应用内跳转的放宽。
7. 工作记录 `125` 的删减清单里，`displays.md` 的行号全部不对：`:259-262` 实为 `:104-107`，`:218-220` 实为 `:63-64`，`:177` 实为 `:22`。其余行号都对。清单里压缩 `connection.md:112-120` 那一条，会删掉缺-15 要求修改的那几句。
8. 四段顺序里有三处要返工：缺-4 和取-5 改同一行代码和同一个测试文件，却分在第一段和第三段；缺-7 的措辞改动和第四段的首行清理改同一个函数；第二、三段为行为改动同步的指引句子，第四段会再删或再压缩。
9. 第八节第 5、6、7 条基本属实：源码里没有写死权限控制器包名，也没有现成的取包名函数；缺-4 和取-5 确实在同一段代码里；名单在 `baselineAgeMs` 移走后确实没用了。但行号要更正，而且等待命令失败时首行里还有第二份 `baselineAgeMs`。
10. 有 1 件事要用户定，见第 4 节的问题。另有两条意见写给主评审：第八节第 8 条的"不加测试"和缺-11 修缺陷有冲突；删减清单与缺-15 有冲突。

---

## 1. 位置核对表

### 1.1 不属实或有偏差的（先看这些）

| 决定文件里的位置 | 决定文件说是什么 | 核对结果 | 准确位置或要补的位置 |
| --- | --- | --- | --- |
| 第八节第 7 条 `ui_output.py:120-126` | 主机端"哪些命令显示基准年龄"的名单 | **行号不对。** `:120-126` 是 `fill` 步骤列表推导式的尾部和首行开头 | 名单在 `:128-134`（`show_baseline_age=` 的表达式），参数声明在 `:447`，唯一的使用在 `:510-511` |
| 缺-14 `WorkerConnection.kt:120-122`、`:333` 一带 | "请点刷新"写法不一；"连接被拒绝"不给下一步 | **偏差。** `:120-122` 是 60 秒例外那句，已经写了 "ask the user to use Refresh status"，是唯一写对的。`:333` 是 "Worker is connecting; retry in a few seconds"，没有问题 | 要改的是：`:43`（"binding timed out; use Refresh status"）、`:334`（DISCONNECTED_NOTICE）、`:335`（STOPPED_NOTICE）、`:88`（"Worker attachment rejected: …"，这就是"连接被拒绝"，没有下一步）。`:347`（Shizuku 身份）同类，可以一并统一。另外 `contract/reason-codes.json:150` 的 `workerUnavailable` 下一步文字也提到 Refresh，要对齐 |
| 缺-13 "主机端报错文字（`app.py` 的用法错误处理）" | 用法提示里的 "`.;`"；报错里的 `--any-of` 示例被首行转义成 `\"` | **一半对。** "`.;`" 确实在 `app.py:1080-1128`：Click 原文以句号结尾，又接了 "; …"。实测：`fill` 缺参数时打印 `Missing argument '[REF] TEXT'.; for example, …` | `--any-of` 示例在 `ui_commands.py:338`、`:346-347`、`:679-680`、`:693-695`。`\"` 是 `reason_facts` 用 `display_name(…, quoted=False)` 渲染原因时产生的（`ui_output.py:238`，`output.py:14-19`）。实测：`wait --any-of x` 打印 `--any-of '[{\"text\":\"Ready\"}]'` |
| 第二节第 1 条（取-1）`UiActions.kt:568-613`、`:843-883`；`UiEffect.kt:44-49` | 开关核对路径 | 列出的行都对：`rowCheck` 在 `:568-613`，`awaitValue` 在 `:843-886`，终局判断在 `:881-882`，`UiEffect` 按原窗口和原节点编号找目标在 `:44-49`。**但不全** | 补：①直接可点的开关走 `sendAction`（`:516-565`，核对在 `:545`，`UiEffect.apply` 在 `:557`）。实测三次走的都是 `rowCheck`，但别的设置页可能走这条。②事实要加在 `settle` 之后（`rowCheck` 里是 `:603-610`）。③主机端首行渲染在 `ui_output.py` 的 `result_line`。④契约样本，在 `WireContractTest.kt`、`contract/wire-contract.json` 和 `test_wire_contract.py` 里加一个样本 |
| 第二节第 2 条（取-2）`UiActions.kt:866-877`、`:659`、`:805-817` | 原框不可见判"不可观察"；核对没过不发提交 | 三处都对 | 补：动作后的界面树只在 `FillSequence.finish()` 里拿得到（`:692-712`，观察在 `:700`），事实应加在这里；另外还有主机端渲染和契约样本 |
| 第二节第 3 条（缺-1）`ActivityTasks.kt:104-133`、`:43-44` | 落点判断；按任务归属判断的函数 | 两处都对 | 补：①`ActivityLaunch.kt:354-392` 的 `confirmLanding`，完成时的原因文字在 `:373`，写的是 "package and UID are visible"。②`:246-262` 的 `inspectConfirmedLanding`，后台屏确认后的复查。③**`:404-420` 的 `recordOpenedTasks`**，同样只认最上面界面的包名和 UID，不改的话后台屏的任务记录会漏（见第 2 节第 3 条）。④主机端 `ui_output.py:76-80`（报完成时 `open` 只打印 `taskMovement`）和 `:312-327`（`launch_facts`）。⑤指引 `actions.md:196-199`（"visible task's actual top package and full UID confirm the target display"）和 `displays.md:57-58`，这两句会变得不对。⑥docs `reference/ui.md` 的对应段落 |
| 第二节第 5 条（取-5）`UiPostAction.kt:27-54`、`:198-203`；`SKILL.md:85` | 放宽首次读取的条件；取-6 那句指引 | 都对 | 补：`observations.md:149-152`。这几行正好描述"首份可用界面树可以用满命令时限"，行为改了就要同步 |
| 缺-4 `UiPostAction.kt:39-44` | 等应用窗口时用满时限 | 对 | 补：允许结束的判断在循环里（`:97-101`，`canSettle` 和 `finished`）；"连续两份完整样本相同"的计数在 `SettledSamples.accept`（`:270-278`），现在只在没有基准时通过 `stableWithoutBaseline`（`:300`）对外暴露 |
| 第二节第 7 条（首行）`UiPostAction.kt:223-243`、`ui_output.py:298-301`、`observations.md:153-156`、契约样本 | 两个统计字段 | 都对 | 补：`UiPostAction.kt:253`（调用 `observeRoots`）、`:327-329`（写进结果）；契约里的 `root-unavailable`、`root-recovered` 两个样本（`wire-contract.json:5927-5968`）及其设备端生成代码；`test_wire_contract.py:452-468`；`PostActionFactsTest.kt`（7 处）；docs `reference/ui.md:118`。**注意别连带删掉 `observeAfterRootLoss`（`:156-170`），取-1 那两次拿到新界面树靠的就是它** |
| 第二节第 7 条 `ui_output.py:105-217` 里的 `baselineAgeMs` | 移出首行 | 对（`:510-511`） | 补：等待命令失败时，`match=` 里的条件摘要只在成功时去掉 `baselineAgeMs` 等时间字段（`:16-34`），失败时仍会打印到首行。这些字段由设备端 `UiCondition.kt:499-503` 写入。决定文件没说这一份要不要一起移走（见第 4 节的问题） |
| 第二节第 7 条 `snapshotBytes`、`snapshotLines` | 只在超限摘要里保留 | 对（`ui_output.py:199-204`；`ui_client.py:943-945`） | 超限摘要本身已经在文字里写了字节数和行数（`ui_client.py:912-917`），所以从首行删掉不会丢信息。连带改 `test_ui_reading_delivery.py:330-331` 和 `test_result_output.py` 里的相关断言 |
| 缺-15 `connection.md:115-120` | "旧会话已作废"提示的指引 | 对，核心在 `:117-120` | 设备端只存一份提示的位置是 `WorkerConnection.kt:116`（置位）和 `:205-209`（取走即清）。按决定只改指引，设备端不动 |

### 1.2 核对属实的

| 位置 | 内容 |
| --- | --- |
| `UiActions.kt:1063-1071`（缺-3） | `awaitInputFocus` 的循环以 `run.budget.remaining()` 为条件，会等满命令时限。函数整体在 `:1057-1072` |
| `UiPostAction.kt:146`（缺-6） | `reportAbsentWindow` 无条件写 `settle=changed` |
| `ui_output.py:446-518`（缺-7） | `change_summary`；"可能还在变化"的措辞在 `:458-462`、`:491-494`、`:514-517`。设备端没有基准就判"未稳定"的地方在 `UiPostAction.kt:293` |
| `ScrollProgress.kt:138-150`（缺-8） | 没有路线时用 "container is at its end" 覆盖掉带方向建议的原文（`:148`） |
| `UiCondition.kt:452-455`（缺-9） | 评估之后时限归零，就算已命中（`winner >= 0`）也报失败 |
| `ui_output.py:224-237` 一带（缺-10） | 实测：构造一个"完成、退出码不是 0"的 `fill`，首行是 `fill: done; focus=done; fill=done; submit=done; fill=done, verification="matches"; submit=done`。重复来自 `reason_facts`（`:224-238`）把复合原因又打印了一遍 |
| `androidctl/tests/ui_test_support.py:12-58`（缺-10） | 解析函数遇到重名字段时，后者覆盖前者 |
| `BorrowedPaste.kt:353`（缺-11） | 集合是 `setOf("failed", "unknown")`，漏了 `"pending"`；`input.md:62-65`（要改的是 `:63` "These receipts warn…"） |
| 缺-12：`connection.md:135`、`README.md:27-28`、`SKILL.md:91`，以及 `displays.md` 与 `connection.md` 的同一字段 | `DaemonVersion`；README 写"每条命令只打印简短结果"；"Read/search that file after host-tool truncation"；同一字段在 `displays.md:22` 叫 `callerSession`，在 `connection.md:101` 叫 `callerState` |
| `UiRefRepair.kt:77-83`、`:124-126`、`:130-133`（缺-5） | 领先不到 2 分才算有歧义；位置分 2；空标签不算相同。另外"位置近"的判断在 `:120`、`:137-145`（四个坐标之差合计不超过 8 像素） |
| `UiPostAction.kt:133` | `samplingStopped` = "Estimated next sample exceeds the observation deadline" |
| `UiPostAction.kt:341-447`（第五节第 5 条，取-14） | 导航补全规则 |
| `SKILL.md:33`、`:39`、`:85`、`:91`、`:128` | 见第 3 节 |

---

## 2. 给实施方的风险提示

**第 1 条（取-1）。** 核对结论在观察之前就定了：`awaitValue` 在 `:600` 返回，`checked` 在 `:601` 定下，新界面树到 `:603` 的 `settle` 之后才进 `run.observation`。所以新事实只能在 `UiEffect.apply` 之后附加，不能回头改 `checked`。"同一位置、同一编号"对应节点数据里的这些字段：`bounds`（位置）、`resourceId`（编号）、`view.role`、`row.label`、`row.chain`（行列号）和 `packageName`。`sourceId`、`windowId` 和引用编号重建后都会变，不能用。实测三次：`r7-new-07` 第 9 条（`check e241580321`）回执没有界面树（`scopeUnknown`），这时不能写这条事实。第 24 条（`uncheck e241580843`）的窗口从 142 换成 144，新开关 `e241580874`，编号 `switch_widget`，`checked="false"`，正是目标状态。`r7-new-07-retry1` 第 10 条（`check e241581262`）的窗口从 153 换成 155，新开关 `e241581293`，编号 `switch_widget`，`checked="true"`。三次的开关自己都没有名字（`name=""`），所在行（第 1 行第 0 列）也没有名字，行里的子项叫 "Use Dark theme"。所以 `row.label` 多半就是 "Use Dark theme"，这是按 `UiSemantics.kt:215-229` 推断的，文本视图里看不到。文本视图里也没有新开关的坐标，"同一位置"只能在 JSON 里验证。`checkedIdentityEnded=true` 时观察只取一份样本，这一份能不能拿到新树，取决于 `observeAfterRootLoss`。

**第 2 条（取-2）。** 合并规则（`FillProgress.kt:95-101`）是：任一步是"结果未知"，整条就是"结果未知"。所以只要只在回执里加事实、不改 `fill` 这一步的结果，整条仍是"结果未知"（退出码 3）。如果把 `fill` 这一步改成完成，`submit` 仍是"未发出"，`possiblePrefix` 为真，整条会落到"失败"（退出码 1）。"已获得焦点的输入框"可以这样唯一确定：新树里、原窗口内恰好一个可编辑且 `focused` 的节点，文本与请求值完全相等，有引用编号，而且不是原目标。实测两次（`r7-new-08` 第 4 条、`r7-new-09-retry1` 第 4 条）都满足：同在窗口 63，`url_bar` 带 `focused` 和 `+ime="Enter"`。要防的情况有两种：Chrome 的行内自动补全会让值多出一段，这时完全相等不成立，不写事实就行；密码框不能比较值。

**第 3 条（缺-1）。** 现在的落点判断只看可见任务最上面那个界面的包名和 UID（`ActivityTasks.kt:110-116`）。`hasTaskIdentity`（`:43-44`）可以直接调用，但它只比较包名，不比较 UID。权限框那一层的 UID 是权限控制器自己的，所以这条路径上指引写的"完整 UID 确认"（`actions.md:196-199`）做不到，要改那句话；也可以再加一道保护，比如要求这个任务是本次打开新出现的（`beforeTasks` 里没有）。权限控制器的包名在源码里既没有写死，也没有任何取法。可用的系统接口有两个：`PackageManager.getPermissionControllerPackageName()`，它是隐藏的系统接口，代码里已有 `HiddenApi` 反射的先例；或者解析 `PackageManager.ACTION_REQUEST_PERMISSIONS` 这个 Intent 的处理者。两者都要在设备上确认。后台屏"目标出现在别的屏"的改报（`ActivityLaunch.kt:254-256`、`:375-381`）用的是同一个 `matching`。如果把新规则加进 `matching`，"别的屏"的判断也会变宽，而且失去 UID 保护。建议新规则只用于"目标屏上"这一判断，单独给一个落点状态，`elsewhere` 的逻辑不动。**一定要同时改 `recordOpenedTasks`（`:404-420`）**，否则后台屏上这个任务不会记进"本会话打开的"，关闭时会按来历不明处理（`DisplayTasks.kt:133-145`）。

**第 4 条（缺-5）。** 打分有五项（`UiRefRepair.kt:124-126`）：名字或编号的锚点每个 8 分，类名相同 3 分，行标签相同 3 分，共有动作每个 2 分、最多 4 分，位置相同 2 分。"领先只来自位置分"不能写成"领先正好 2 分就拒绝"，因为共有动作多一个也是 2 分。应写成"去掉位置分之后领先不到 2 分"。我用脚本按源码规则重算了 `ref-repair-pools.json` 的 21 组样本（脚本在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents15/137/pools.py`）。报"选中"的 10 组里，6 组只有一个候选；`Files36-returned` 三组去掉位置分后领先 11 分，`Files30-returned` 三组领先 3 分。所以按上面的写法改，都仍会选中。没有测试依赖"位置分定胜负"：`UiRefRepairExecutionTest` 和 `WireContractTest` 用的是单一候选或 7 个完全相同的候选。会被新规则拒掉的只有缺-5 描述的那种情形：同名、同编号、行没有标签、只靠位置区分。这正是要拒的。需要新加一组样本。

**第 5 条（取-5）。** 放宽读取的条件写在 `UiPostAction.kt:39` 的 `collectionUntil`：首份可用界面树到手之前，读取截止用命令时限。第 44 行的 `canStart` 和第 198-203 行的 `waitsForFirstTree` 也参与。坑在于：截止时间在读第一份样本之前就要交给 `ui.sample`（`UiBackend.kt:405-407`），这时手里只有动作前的 `before`（带窗口目录和 `selectionScope`）。能在读之前判断的只有"窗口没变"，比如比较 `before` 的窗口编号与当前窗口列表，`ObservedChange.sameDirectoryIgnoringBounds` 和 `UiScope.same` 都可以用。"同一个页面"读之前无法知道。而在 Chrome 里点链接，窗口不变、页面变了，这正是 Wikipedia 长页的情形。所以按"窗口没变就不放宽"去做，会把多数应用内跳转的放宽都取消，第七轮第 13 项想救的"拿不到首份界面树"会在这些跳转里重新出现。实施方要在那一轮的说明里写清取舍，并在长网页和主题切换上各验一次。它和缺-4 改的是同一行（`:39` 的 `firstTree && (!usable || waitingForApp)`）。

**第 6 条：小缺陷逐条。**

- **缺-3**：`awaitInputFocus` 有 5 个调用处（`UiActions.kt:309`、`:394`、`:730`、`:796`，`UiSelection.kt:40`）。其中 `:309`、`:394` 和 `UiSelection.kt:40` 是动作发出前的准入检查，不能缩短截止。新截止要做成可选参数，只用在发出之后的 `:730`，`:796` 可以一起用。"发出时刻"要在 `progress.started(FOCUS)` 时记下来。
- **缺-4**：`canSettle` 现在只看 `waitingForApp`；"连续两份完整样本相同"的计数 `same` 是私有的，对外只有没有基准时的 `stableWithoutBaseline`。要新开一个判断，并在 `:101` 用上。测试在 `PostActionFactsTest.kt`，取-5 和第四段也会改这个文件。
- **缺-6**：只在 `report()` 的结果不是 `loading` 时才写 `changed`，可行。现有测试里没有断言"窗口消失后仍写 changed"，要新加一个。
- **缺-7**：主机端只看得到 `observedChange.reason = "No before observation for change comparison"` 和 `settle=unsettled`。设备端"已稳定"（`stableWithoutBaseline`）这个事实没有发出来。所以只改主机端措辞时，只能写"没有比较基准"，不能写"已稳定"。连带改 `observations.md:138` 那句和 `test_ui_receipts.py:70` 的断言。
- **缺-8**：可行，约 3 行，只在 `completedSteps == 0` 时保留原文的方向建议。连带的测试 `UiScrollUntilTest.kt:12-31` 名字就叫 "…WithoutReverseAdvice…"，第 30 行对 0 步和 1 步都断言没有反向建议，要拆开。契约样本用的是 1 步（`wire-contract.json:2555-2594`），不受影响。
- **缺-9**：可行，判断时限归零时加上 `winner < 0`。现有测试里没有"评估阶段截止但已命中"的样例，要新加一个。`observations.md:210` 那句不用改。
- **缺-10**：可行。去重应改 `reason_facts`：`fill` 完成且退出码不是 0 时，不再打印复合原因。我在副本里把解析函数改成遇到重名就报错，跑了主机端全部测试（不含进程测试）：2080 个通过，2 个失败，失败原因是副本里缺 Android 源码，与改动无关。所以解析函数加了报错，现有测试不会受牵连。
- **缺-11**：前一种情形在 `:353` 的集合里加 `"pending"`。注意早回执的测试（`BorrowedPasteTest.kt:108-122`、`:300-312`）会因此多出一句提醒，断言要相应调整。后一种情形（粘贴失败或未知、恢复也失败或未知）走的是 `:361-365` 的原因文字，按决定改 `input.md:63`。见第 5 节关于"不加测试"的意见。
- **缺-12**：四处位置都对，见 1.2 表。`connection.md:101` 和 `displays.md:22` 统一成一个写法，例如 "`callerSession` (JSON `callerState`)"。
- **缺-13**："`.;`"在 `app.py` 里去掉句号即可。`\"` 只改文字修不好，因为 JSON 示例里必须有双引号，而首行渲染一定会转义它。可行的做法有两种：报错文字里不放 JSON 示例，改为指向 `androidctl wait --help`；或者把示例放在首行之外的第二行。改渲染规则会影响所有回执，不建议。连带改 `test_result_output.py` 和 `test_ui_arguments.py` 里断言这些原文的测试。
- **缺-14**：位置见 1.1 表。设备端没有测试断言这些原文，`contract/reason-codes.json:150` 要对齐。
- **缺-15**：按决定只改 `connection.md:117-120`。与删减清单的冲突见第 3 节。

**第 7 条（首行）。** `result_line`（`ui_output.py:105-217`）往首行加东西的来源，按顺序是：
- `reason_facts`：原因和错误码；
- `change_summary`：变化标记、`baselineAgeMs` 和括号里的各条说明；
- `cached`、`fresh`；
- `dispatch_facts`：`dispatchStarted`、`samplingStopped`、`step results=unknown`、`route`、`ref`；
- `observation_facts`：`timing`、`matches`、`offscreenMatches`、`notice`、两个统计字段、`loading detected`、`target still offscreen`；
- `value_facts`：14 个核对字段，其中有 `verificationSource`、`verificationCutoffMs`；
- `input_facts`、`action_callback_facts`；
- 剪贴板、借用粘贴、`select`、`open` 的声明字段，以及 `launch_facts`；
- `via`、`textLength`、`channel=touch`；
- 各步骤的 `step_facts`；
- `match=`（条件摘要，失败时带时间字段）、`cutoffStage`、`scroll_search_facts`；
- `completeness_facts`（`observationWarning`）；
- "Observation: unavailable"；
- `snapshotPath`、`imagePath`、`snapshotBytes`、`snapshotLines`；
- `image_facts`（`imageSize`、`imageMime`、`imageSourceSize`、`imageMappingKnown`，区域截图还带 `at`、`finishedMs` 等）；
- `delivery_problem_facts`、`unknown_facts`；
- 最后是 `output.context` 加上的 `device`、`session`。

按准入规则，决定文件点名移走的是两个统计字段、`baselineAgeMs`、`samplingStopped`、`snapshotBytes` 和 `snapshotLines`。按同一规则看，还可能该移走的有：等待失败时 `match=` 里的 `sequence`、`sampleMs` 和四个 `baseline*` 字段；区域截图的 `at`、`finishedMs`；`verificationCutoffMs`；`imageMime`、`imageSourceSize`（图片已映射时）。决定文件没说这些在不在"清一次"之内，见第 4 节的问题。

连带要改的文件：契约 `wire-contract.json` 的 `root-*` 两个样本及其设备端生成代码，之后要按 AGENTS.md 的 Windows 流程重新导出；`test_wire_contract.py:452-468`；`test_result_output.py`（23 处）；`test_ui_execution.py:755-793`；`test_ui_reading_delivery.py:330-331`；`PostActionFactsTest.kt`、`UiOutcomeTest.kt`、`UiReceiptTest.kt`、`UiDeliveryTest.kt`、`UiConditionTest.kt`；指引 `observations.md:153-156`；docs `reference/ui.md:106`、`:118`。

调用者指引里引用将移走字段的只有 `observations.md:153-156`（两个统计字段）。`observations.md:139` 的 "sample age remain independent" 说的是 JSON 里的事实，可以不动。`baselineAgeMs` 在 JSON 里保留（`UiObservation.kt:41`），契约里 `:155`、`:372`、`:409`、`:2136` 这几处不用改。

---

## 3. 第三节的指引行号与删减清单

**首页行号。** 五处都与决定文件说的一致，但有两处要注意"删掉一句"具体删多少：
- `:33` 是 "`status` reads idle time without renewal; see [displays](displays.md) for details."。指向显示专题的只是后半句，前半句是关于空闲时间的有用事实。
- `:128` 一整行同时含 `connection` 和 `displays` 两个链接。整行删掉的话，`connection` 只剩 `:35` 一处入口（那里已有）。
- `:39` 是 "Read each result and observation before the next command; known steps may run sequentially."。
- `:85` 是 "Partial sampling can establish positive facts, but cannot prove absence."。
- `:91` 是 "Read/search that file after host-tool truncation."。

问-1 选 A 时，建议写明 `:33` 只删后半句。

**英文初稿里的锚点。** `reference.md:69` 的标题是 "## UNKNOWN And Resource Release"，对应锚点 `#unknown-and-resource-release`，存在。现在的 `SKILL.md:118` 也用这个锚点。

**工作记录 `125` 第 4.2 节（首页），逐行核对。** `:21-24`、`:33`、`:128`、`:39-41`、`:65-67`（以及它引用的 `observations.md:38-40`）、`:83-84`、`:89-94`（它说 `--json`、`--raw` "那里已有"，在 `observations.md:49-55`，属实）、`:110-112`（要删的那句在 `:111`）、`:119-120`（以及 `:118`）。全部仍指向它说的内容。

**第 4.3 节（专题），逐行核对。**

| 清单位置 | 在 `409e853` 上 | 结论 |
| --- | --- | --- |
| `observations.md:94-97` | 锁屏和熄屏标记 | 对 |
| `observations.md:149-156` | 首份界面树采样和两个统计字段 | 对 |
| `observations.md:199-213` | 等待截断、`cutoffStage`、any-of 的 complete | 对 |
| `observations.md:13-18` | `fresh` 和 `findRefreshed`、缓存读失败 | 对 |
| `actions.md:38-40` | 上限下唯一性的例外 | 对 |
| `actions.md:72-76` | 形状映射和单行多列的例外 | 对 |
| `actions.md:204-206` | 确认落地之后的观察（实际文字在 `:204-205`） | 对；但它给的删除理由"`displays.md:218-220` 讲的是同一件事"**行号不对**，同一件事在 `displays.md:63-64` |
| `connection.md:112-120` | 重绑规则 | 对 |
| `connection.md:141-155` | 设备诊断、锁文件、凭据路径 | 对 |
| `displays.md:259-262` | 关闭摘要的格式 | **行号不对**，`displays.md` 只有 120 行，实际在 `:104-107` |
| `images.md` 历史清理诊断 2 行 | — | 在 `images.md:105-106` |

另外第 1.2 节 A2 的 `displays.md:177` 实际在 `:22`。缺-12 要用到这一处，决定文件只写了文件名，没有写行号，所以不受影响。

**删减清单与决定文件别处的冲突。**

1. **`connection.md:112-120` 压成两行，与缺-15 冲突。** 缺-15 要求"只改 `connection.md` 里已有的那几句"（`:117-120`："The next sessionRequired error reports … A successful session open or takeover clears that notice"）。清单给的两行替代文字里没有这层意思，压缩之后缺-15 就没有句子可改了。按第八节第 10 条"冲突以本文件为准"，应当在压缩后的文字里保留"作废提示只送一次，可能被别的会话先取走"。建议主评审在第八节第 10 条里点名这一处。
2. **`observations.md:149-156`。** 清单给的替代句写的是两个统计字段，而这两个字段第二节第 7 条要删。第八节第 10 条已经处理了这一点。但 `:149-152` 描述的放宽读取正是取-5 和缺-4 要改的行为，删或移之前要先确定新行为的写法（见第 4 节）。
3. **`SKILL.md:39-41` "按 3.3 改写"。** `125` 第 3.3 节的写法多了一句"只连发互不依赖结果的命令"，用户的原话只有"某条命令退出码不是 0 就停下后面的"。以决定文件为准，不加前半句。
4. **重叠但不冲突的：** `:89-94` 的压缩包含 `:91`（缺-12），两者改法一致，一次做完即可；`:65-67` 包含 `:66`（选-2，在问-2 里）；`:33`、`:128` 已有问-1；`:119-120` 已被第三节第 1 条取代；`actions.md:204-205` 要删，而紧挨着的 `:196-199` 要为缺-1 改写，最好一并处理。

---

## 4. 第四节四段顺序的意见（写给主评审，不改顺序）

有三处先后依赖会造成返工：

1. **缺-4（第一段）与取-5（第三段）改同一行。** 都是 `UiPostAction.kt:39`（`collectionUntil`），以及 `canStart` 和 `PostActionFactsTest.kt`，第三段会把第一段刚改的条件和测试再改一遍。决定文件第八节第 6 条已经说"两处要一起看"。最省事的排法是把缺-4 挪到第三段和取-5 一起做，或者把取-5 提到第一段。
2. **缺-7、缺-10（第一段）与首行清理（第四段）改同一个函数和同一批测试。** 缺-7 改 `change_summary` 的措辞，缺-10 改 `reason_facts`。第四段要把"某某未知"的子句合成一句、移走 `baselineAgeMs`，改的还是 `change_summary`，测试集中在 `test_result_output.py`。第二段新加进首行的三条事实，到第四段也可能按规则再调一次。省事的做法是把缺-7 挪到第四段。缺-10 可以留在第一段，它只是去重，加上解析函数的报错，正好能护住后面的改动。
3. **指引同步与删减清单。** 第二、三段的行为改动按"行为改动照常同步"要改 `actions.md:196-205`、`displays.md:57-58`、`observations.md:149-152`；第四段的删减清单又要删或压缩其中两处。省事的做法是第二、三段的指引同步一律推到第四段，和删减一起做。第八节第 3 条已经把指引类小缺陷放到第四段，这里是同一个原则。

另外，契约样本会在第二段（新事实）和第四段（删统计字段）各重新导出一次，导出只能在 Windows 上做。这不算返工，但实施方要知道。

**待用户定（1 问）。**

**问：首行"清一次"是只清点名的几项，还是按准入规则把首行全过一遍？**

背景：决定文件第二节第 7 条点名的是两个统计字段、`baselineAgeMs`、"预计下次采样超过观察期限"那句、`snapshotBytes` 和 `snapshotLines`，以及把几个"某某未知"的子句合成一句。按同一条规则，首行里还有几项同类的东西：等待失败时条件摘要里的 `sequence`、`sampleMs` 和四个 `baseline*` 字段（`baselineAgeMs` 在这里还有第二份）；区域截图的 `at`、`finishedMs`；`verificationCutoffMs`；图片已映射时的 `imageMime`、`imageSourceSize`。用户的原话是"立准入规则并清一次"，没说范围。

- **A：只清点名的几项。** 好处：范围最明确，测试改动最少。代价：首行里还留着第二份 `baselineAgeMs`，规则立了却没有执行完。
- **B：点名的几项，加上等待失败时条件摘要里的时间字段。** 好处：`baselineAgeMs` 不会在首行出现两种情形；改动只多了 `predicate_summary` 里的一个判断和几条测试。代价：比 A 多一点范围。
- **C：按规则全过一遍，逐项列出移走的理由。** 好处：一次清干净。代价：图片相关的字段可能有调用者在用（坐标映射），判断有争议，范围会变大，也更难对着真实记录逐条论证。

**我推荐 B。** 它让"移出首行"在两处一致，范围仍然封闭。C 里那几项最好等有真实记录再说。

---

## 5. 第八节第 5、6、7 条的核对

- **第 5 条（权限控制器包名）。** 属实，而且源码里现在什么都没有：主代码、测试、主机端和指引里都没有 `permissioncontroller` 字样，也没有任何取包名的函数。实测记录里出现的包名是 `com.android.permissioncontroller`（`r7-new-07-retry1` 第 25 条命令），其他厂商的设备上可能不同。系统接口的两种取法见第 2 节第 3 条，都需要设备确认。"主屏和后台屏用同一个判断"可以做到：两条路径都经过 `activityLanding`（`ActivityLaunch.kt:253`、`:367`）。但要同时改 `recordOpenedTasks`。
- **第 6 条（`UiPostAction.kt:27-54` 与缺-4 是同一段）。** 属实。两者都改 `PostActionWindow`，而且是同一行 `:39`。不过缺-4 的完整改动还会超出这一段，伸到循环（`:97-101`）和 `SettledSamples`（`:270-300`）。
- **第 7 条（名单删掉后没用了）。** 结论属实，行号要更正为 `:128-134`。`show_baseline_age` 只在 `:510` 控制 `baselineAgeMs` 的打印，移走之后这个参数和名单都没有别的用处。"两端各算一遍"也随之消失：设备端的 `usesCurrentBaseline`（`UiPostAction.kt:205-210`）还在，但它另有用途（`:131`，判断"没有观察到变化"），不再和主机端的名单重复。补充一点：`baselineAgeMs` 在等待命令失败时还会经条件摘要出现在首行（`ui_output.py:16-34`），这一份不受名单控制，见第 4 节的问题。

**写给主评审的另一条意见。** 第八节第 8 条把"冻结不动"解释为"不加测试"，又说"出了缺陷照修"。缺-11 的修法会改变早回执的原因文字，至少要调整 `BorrowedPasteTest.kt` 里现有的断言，最好再加一条断言覆盖 `pending`。建议在第八节第 8 条写明：修缺陷时可以改现有测试、加针对这个缺陷的断言，但不为冻结的能力加别的测试。这是评审方自己补的理解，可以直接改，不必问用户。
