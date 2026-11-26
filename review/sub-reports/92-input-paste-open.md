# 第七轮专题核查：输入、借用粘贴、`open`，以及"动作已派发但后态没确认"时事实的保留

对象：产品 `f401deb`（本轮差量 `fb75afc..f401deb`，相关提交 `545d095`、`41c25c6`、`f401deb`）。源码行号默认指 `f401deb`；Kotlin 文件在 `android/app/src/main/java/com/rainng/androidctl/`，主机文件在 `androidctl/src/androidctl/`。
记号：`S` = 本次评审的 scratchpad 根；`E` = 实施方材料包里的 `ultragoal/2026-11-11-next-improve/stages/42-r4-decisions/efficiency`；`RT` = 同一阶段目录下的 `runtime/api34`。我的脚本在 `S/agents11/92/`（`cmds.py` 列出每次运行的命令与回执首行，`render_paste.py`／`render_cmd.py` 用候选主机渲染契约样本，`noise.py` 统计输出噪声）。没有设备；所有设备行为来自实施方原件或源码推断。

## 摘要

1. **借用粘贴按第 22 条做对了**：粘贴已核对一致时，恢复失败报"完成＋退出码 1"，恢复未知或尚未结束报"完成＋退出码 3"；粘贴本身没成功的组合保持原状。恢复排在动作后观察之后，恢复前先比对剪贴板。契约样本、两份指引和实现一致（主机渲染与定向测试都已核对）。没有发现会让调用者重打已成功文字、或在粘贴没成功时报"完成"的路径。
2. 粘贴还差一处小问题：恢复失败或未知时，剪贴板里很可能还留着刚输入的文字（可能是密码），回执和指引都没有直说，也没给下一步。这是一句指引加一句原因文字的事，建议进下一轮。
3. **`open`"落地之后带上界面树"（第 15 条）在真实使用中基本没做到**：本轮能找到的 11 次真实 `open`，有 9 次回执是 `done`，但附带的树不可用。网页题准备阶段 5 次在主屏打开 Chrome，5 次都是这样，而且回执让调用者去 `snapshot --window` 导航栏和状态栏；随后那次快照看到的才是延后弹出的"设为默认浏览器"系统对话框。后台屏题 5 次里有 3 次没有树，模型每次都只能多跑一次 `snapshot`。实施方计划表把这一项记成"已闭"，与证据不符。修法很小，建议进下一轮，并更正"已闭"的说法。
4. **`545d095` 给 `open` 带来一处新的时序风险**（源码推断）：首次确认落地之后又做了一次确认，要求目标仍在顶层。落地后 1～2 秒才弹出的别的包的对话框（默认浏览器问询、权限框）会让 `done` 变成 UNKNOWN，并且一直等到请求预算用完。实测中这类对话框出现在约 2.5 秒，第二次确认在约 1.5～2 秒，差距很小。还有一种情形：启动时立刻弹权限框，落地确认会吃掉全部预算，结果 UNKNOWN 而且没有新树，比上一轮差。建议进下一轮，整理提交时顺手修。
5. 输入类命令（`fill`／`type`／`key`／`fill --submit`）的读回规则、中途失败时保留已做的步骤、A4 的成功短回执，都已落实。后台屏题 5 次填表流程里，没有一次 `fill` 之后为了确认又去 `snapshot`。剩下的噪声不到总输出的 1%，不必处理。
6. "已派发、后态没确认"的一般规则前后一致：已经记下的结果不会因为观察失败而改；派发开始但没有结果时报 UNKNOWN；没有找到把"已派发"写成"未发出"的地方，反过来也没有。本块没有需要挡合入的项。

---

## 一、借用粘贴：全部结果组合

流程（`BorrowedPaste.kt:101-122`）：读原剪贴板并检查（`:142-163, 205-222`）→ 借写入（`:150-161`，写入前先下发一份"结果未知"的预备回执）→ 回读确认（`:165-175`）→ 现查 PASTE 是否宣告（`UiActions.kt:301-308`）→ 发一次 PASTE → 在动作后观察窗口里轮询读回核对（`UiActions.kt:321-343`）→ 动作后观察（`UiActions.kt:271-284`）→ `finally` 里先比对再恢复（`BorrowedPaste.kt:260-312`）。主结果和退出码在每次 `save()` 时统一算一遍（`:334-360`），所以早期回执和最终回执用的是同一套规则。

下表全部是源码推断，契约里有样本的标出样本名（我用候选主机渲染过，见 `render_paste.py`）。

| 阶段情形 | 主结果 | 退出码 | 默认文本里能看到的 | 契约样本 |
| --- | --- | --- | --- | --- |
| 原剪贴板读不到、含 URI/Intent | not sent | 2 | 原因＋`code=clipboardUnavailable`／`clipboardTransfer`；`restore=not required` | original-unknown |
| 借写调用抛错、没返回 | UNKNOWN | 3 | `borrow` 未知；随后照常比对恢复 | — |
| 借写已返回，回读与借写内容不一致 | failed | 1 | "PASTE was not sent"；恢复多为 `skipped`（剪贴板已被别人改） | — |
| 借写回读不可观察 | UNKNOWN | 3 | 同上；恢复多为 `unknown` | — |
| 目标未就绪或 PASTE 没宣告 | failed | 1 | `paste=not sent; restore=restored`，`dispatchStarted=true`（借写算已派发） | advertisement-rejected |
| PASTE 平台明确失败 | failed | 1 | `paste=attempted` | — |
| PASTE 已发，回执未知 | UNKNOWN | 3 | 附"不要自动重复"的固定提示 | — |
| PASTE 成功，读回一致 | done | 0 | `effect=matches; paste=attempted; restore=restored|cleared|skipped` | restored／cleared／copy-skipped |
| PASTE 成功，读回不同（例如自动补全改了字） | UNKNOWN | 3 | `effect=differs`、原样的 `actualValue`、"不要自动重复" | effect-differs |
| PASTE 成功，读回不可观察（字段没变、密码框） | UNKNOWN | 3 | `effectReason`、"不要自动重复" | effect-unchanged |
| 读回一致，**恢复失败** | **done** | **1** | `restore=failed; restoreReason=…`；首行没有原因文字 | restore-failed |
| 读回一致，**恢复未知或仍在进行（早期回执）** | **done** | **3** | `restore=unknown|pending`；首行没有原因文字 | restore-unknown |
| 读回未一致＋恢复失败 | UNKNOWN（PASTE 已发）或 failed（PASTE 没发） | 3／1 | "Borrowed paste did not complete: effect=…, cleanup=failed" | — |
| PASTE 平台失败＋恢复未知 | UNKNOWN | 3 | 同上 | — |
| 动作后观察抛错 | 不变 | 不变 | `snapshotError=…` | — |
| 常驻进程在中途死掉 | 主机报 UNKNOWN（传输失败） | 3 | 没有恢复事实；`input.md` 写了"可能留下借写内容、丢失原内容" | — |

逐项对照：

- **第 22 条**：只改了"效果已核对一致"这一种（`:335` 的 `verified` 要求平台回执是 done，并且读回 `matches`）；粘贴本身没成功的组合仍按旧优先级（恢复未知优先于 failed，恢复失败优先于原结果）。第四节第 9 处提到的"未发出被恢复失败改成失败"实际不会发生：主结果为 not sent 只出现在借写之前，那时根本不进恢复（`:114-118, 326`）。【源码推断】
- **指引**：`input.md` 第 61-66 行改成"已核对的粘贴即使恢复失败（退出码 1）或未知（3）也报完成；读 `restore`，不要为了清理重发粘贴"；`reference.md` 第 60-61 行补了一句"完成加退出码 1"。两处不再矛盾。`clipboard.md` 只有一处链接，没有冲突。【源码】
- **契约**：`restore-failed` 和 `restore-unknown` 两个样本已经重新导出（`contract/wire-contract.json` 中 result 和 exit 已改，reason 已去掉）。主机定向测试：`pytest androidctl/tests/test_text_input.py androidctl/tests/test_activity_intents.py` 57 项通过，`test_wire_contract.py -k "restore or paste or open"` 21 项通过。【主机端本地复现】
- **会不会引导重打已成功的文字**：不会。done 加退出码 1 或 3 的首行写的是 `type: done`，指引里两处都说不要重发。上一轮"恢复失败整条报 failed"的诱因已经去掉。【源码＋主机渲染】
- **会不会在粘贴没成功时看到"完成"**：不会。done 同时要求平台回执 done 和读回 matches；读回会先排除"字段原样未变"（`BorrowedPaste.kt:48-50`），所以替换值恰好等于原值时也是 UNKNOWN。【源码】
- **恢复与动作后观察的先后**：观察在 `execute` 的 try 里，恢复在 `finally` 里，恢复一定晚于观察；观察抛错时也会恢复（`:101-119`）。【源码】
- **并发的别的剪贴板写入**：如果在借写和回读确认之间有别人写入，回读会不一致，PASTE 不发，结果 failed，恢复 `skipped`。如果在确认和 PASTE 之间写入，粘进去的是别人的内容，读回 `differs`，结果 UNKNOWN 并附 `actualValue`，恢复 `skipped`。比对和恢复两步不是原子的，指引已写明。我没有找到能把别人的新内容覆盖掉的路径。【源码】
- **密码框**：读回永远是"不可观察"（`UiActions.kt:346`），所以往密码框粘贴永远不会 done，固定报 UNKNOWN 加退出码 3。`fill` 往密码框写则是"done 加 verification=unobservable"（`UiActions.kt:747-751`、`actions.md:130-131`）。两条通道对密码的结论不对称，但调用者写密码本来就该用 `fill`，影响很小，见 F6。

### F1 恢复失败或未知时，剪贴板里可能留着刚输入的文字，回执和指引都没有直说

- 现象：恢复为 `failed`／`unknown`／`pending` 时，借写的内容很可能还在剪贴板里，可能是密码。首行是 `type: done`，没有原因文字，因为设备在主结果为 done 时会删掉 reason（`BorrowedPaste.kt:352-353`），`f401deb` 在主机端的补丁也就无从生效。后面只跟着 `restore=failed; restoreReason="Restore member is unavailable"`。`input.md` 只说"不要为了清理重发粘贴"，没有说剪贴板里可能还留着输入的内容，也没有给下一步。实际上 `clipboard clear` 是现成的命令。
- 触发：恢复准备失败、恢复调用被拒、恢复后回读不一致或读不到；或者请求预算在恢复之前用完，回执先发出、恢复还没做完。
- 真实任务：20 次运行里一次 `type --via paste` 都没有（上一轮 35 次也没有）。没有发生过。
- 影响：调用者不知道剪贴板里留着敏感文字，也就不会告诉用户。不会造成重复输入。
- 最小修正：恢复不是成功状态时，设备端给一个固定原因，例如"Text pasted and verified; clipboard restoration failed/unknown; the clipboard may still hold the typed text; do not paste again"（`f401deb` 已经让 done 加非零退出码时首行保留原因）。`input.md` 加一句：内容敏感时告诉用户，等界面资源释放后可以用 `clipboard clear` 清掉。维护代价：两三行代码，契约里两条样本的 reason 字段。
- 分类：可选建议；进下一轮。

---

## 二、`open`

### 现在的流程

`SystemActions.open`（`SystemActions.kt:187-196`）→ `ActivityLaunch.perform`（`ActivityLaunch.kt:180-218`）：

1. 解析并检查（找不到具体的处理者时报 not sent／failed；后台屏上先做任务预检）。
2. 下发一份"结果未知、落地待定"的预备回执，然后发启动（`:257-280`）。
3. 平台返回的启动码可接受时，结果是 UNKNOWN，落地状态为待定。**先**循环确认落地（`confirmLanding`，`:328-366`）：用 `getTasks` 看目标显示上是否有可见任务，其顶层活动的包名和完整 UID 都对得上。对得上就是 done；落到别的显示上就停在 UNKNOWN；查询失败也停在 UNKNOWN；对不上就等一会再查，**直到请求预算用完**（`:354-357`）。
4. 然后观察（`postAction.settle(run, null)`，默认 1.5 秒窗口）。
5. 观察之后，主屏上落地状态原本待定的、以及所有后台屏的情形，**再确认一次**（`observeLanding`，`:221-236`）。已经确认"落在别处"、出现查询错误、外来任务被搬进后台屏的，不再改结论；预算用完时保留已知结果（这是内部评审 Must-fix 1 和 3 修的）。

### 回执（候选主机渲染契约样本，`render_cmd.py ... open`）

- 成功：`open: done; display=0`，有任务搬动时再加 `taskMovement=…`。启动码、处理者、预检、落地等细节都省掉了，`--json` 里保留。符合第 15 条。
- 失败或 UNKNOWN：`requestedIntent`、`launchCode`、`launchCodeName`、`resolvedType`、`handler`、`landing`、`landingError`、`preflight`、`taskMovement` 全部打印，后面跟"不要自动重复"的固定提示（`open-unmatched`、`open-landing-error`、`open-pending`、`open-unexpected-task-move`）。成功时省掉的细节，失败时都在。【主机端本地复现】

### 各情形

| 情形 | 主结果／退出码 | 依据 |
| --- | --- | --- |
| 目标已在前台（`START_DELIVERED_TO_TOP`／`TASK_TO_FRONT`） | 第一次确认就对得上 → done／0；窗口已经存在，树一般可用 | 源码推断 |
| 启动成功，但到预算用完都没确认落地 | UNKNOWN／3，附 `landing.visibleTasks`（列出当时在顶层的别的包），启动细节齐全 | 源码＋样本 open-unmatched |
| 落到别的显示上 | UNKNOWN／3，立即停 | 源码＋测试 `ActivityLaunchTest.kt:393` |
| 后台屏上启动 | done／0，`display=2`；树经常不可用（见 F2） | 实测 |
| 启动后弹出别的包的对话框 | 取决于对话框出现得早还是晚，见 F3 | 源码推断＋实测时序 |
| 外来任务被搬进后台屏 | UNKNOWN／3，`code=unexpectedTaskMove`，附"已经启动，不要重复，告诉用户"的提示 | 样本 |

### F2 `open` 落地之后带树（第 15 条）在真实使用中多半没有做到

- 实测：
  - 网页题准备阶段，5 次在主屏 `open com.chrome.beta`（`RT/commands/36xx-r6-c2-candidate-0N-normal-chrome-open/stdout.bin`）：5 次都是 `done`、落地已确认，但附带的树 `scopeUnknown`，`settleMs` 都是 1501，取样 3～5 次。默认快照文件（`RT/verification/r6-c2-candidate-01/.androidctl/snapshots/000001-…txt`）里只有两个 systemui 窗口（导航栏、状态栏），而原因文字写的是 "inspect a listed window: snapshot --window 17; snapshot --window 10"，也就是让调用者去读导航栏和状态栏。约 2.5 秒后的下一份快照（`000002-…-com.android.permissioncontroller-…txt`）才看到 permissioncontroller 的 "Set Chrome Beta as your default browser app?" 对话框。实施方另一处准备记录 `RT/commands/r6-delivery-014-main-beta-open` 与 `-015-…` 是同样的情形，实施方自己在 `r6-runtime-preparation.md:115` 也记了"014 首次 Chrome open 为 done／landing observed，但附树 scopeUnknown"。
  - 后台屏题 5 次（`E/r6-v1-candidate-0N`）：02、04、05 三次的 `open` 回执是 `done; …; display=2; snapshotProblem={"code":"scopeUnknown","message":"…no windows enumerated…"}`，正文写 `screen … unreadable`；01、03 两次有树。树缺失的三次，模型下一条命令都是 `snapshot`。上一轮同一题是 5 次里 4 次没有树（`S/pkg9/.../efficiency/r5-v1-candidate-0N`）。
  - 合计：本轮可见的 11 次真实 `open`，9 次落地已确认但没有可用树。
- 原因（源码推断）：落地确认看的是 ActivityManager 的任务表，这比无障碍窗口列表早；观察只在 1.5 秒窗口里等（`UiPostAction.kt:46-56`），对"一个应用窗口都还没有"的情形不会多等；`UiScope.kt:86-98` 在没有可操作窗口时把 systemui 窗口也列成"可选窗口"。`545d095` 把观察挪到落地确认之后，方向是对的，但真实结果几乎没有改善。
- 对调用者的影响：几乎每次 `open` 之后都要多一次 `snapshot`，多一个回合（按本轮成本页，每回合固定前缀约 1.7 万 token）。主屏上"去读导航栏"的提示是误导，照做会白费一次调用。这次准备阶段里，延后弹出的默认浏览器问询从 `open` 回执完全看不出来，要靠下一次快照才发现。不过 `open` 本身也无从知道 2.5 秒后会发生什么，这一点不是缺陷。
- 是否违背裁决：不是方向相反，是没有达到目的。`ST/r6-plan.md:33` 把第 15 条记成"实现/窄测/review已闭"；实施方的设备检查只写了"不能倒写 014 的附树失败"，没有把它当成未达成。实施方的说明（`r6-delivery-implementation.md:128`）预先写了"落地很晚、预算不足时仍可能交不出可用树"。但证据显示这不是偶发的边角情形，而是常态。
- 最小修正：（1）只对 `open`：观察时如果还没有任何应用窗口（零窗口，或者只有 system 类窗口），在请求预算内继续取样，不受 1.5 秒窗口限制，看到第一个应用窗口后再按原规则观察到稳定。（2）`UiScope.selected` 列可选窗口时去掉 system 类窗口；全部是 system 类窗口时，改用"no App window yet; run snapshot"那条文字。维护代价：十来行代码，外加一个 `ActivityLaunchTest`／`UiPostAction` 用例。
- 分类：确定缺陷（裁决目的未达成）；进下一轮。不挡合入，因为它不会让调用者做错事，代价是每次多一次调用，而且与上一轮相比没有变差。但要明确更正"已闭"的说法，用户确认分栏时需要知道这件事。

### F3 首次确认之后再确认一次，会把已经落地的 `open` 改成 UNKNOWN 并等到预算用完；启动时立即弹框则没有新树

- 现象 A（落地之后才弹框）：主屏上首次确认已经得到 done 之后，观察结束时 `observeLanding` 会因为 `pending` 仍为真而再调一次 `confirmLanding`（`ActivityLaunch.kt:232-235`）。这次如果任务的顶层活动变成了别的包（应用自己调起的系统对话框，例如默认浏览器问询、运行时权限框，都由 permissioncontroller 提供），循环就会进入"未确认"分支：结果改成 UNKNOWN，原因 "Activity launch was sent; target visibility was not confirmed"，一直重查到请求预算（默认 5 秒）用完（`:354-357`）。最终回执是 UNKNOWN 加退出码 3，并附"不要自动重复，先用只读观察核对"的提示。
- 现象 B（启动时立即弹框）：如果应用一启动就弹权限框，首次确认始终对不上，会用掉全部预算，之后的观察拿不到时间（`UiPostAction.kt:44` 在预算为 0 时不取样），回执是 UNKNOWN，树要么没有，要么是启动之前的旧树并标 `timing=before-action`。`fb75afc` 是先观察 1.5 秒、后确认，这种情形下至少能带回一份看得到对话框的树。所以 B 是本轮的小倒退。
- 触发：A 需要对话框恰好出现在首次确认和第二次确认之间。网页题准备阶段的实测时序是：`open` 回执约在 14:18:10.3（快照文件时间戳），对话框约在 14:18:12.8 出现；`open` 的观察 `settleMs=1501`。两者只差约 1 秒，5 次里都没有撞上。B 在 Android 13 以上首次启动、会申请通知权限的应用上很常见。
- 平台前提（未知）：两种现象都依赖"应用调起的对话框活动进入应用自己的任务、成为任务顶层"。这对 `startActivityForResult` 调起的权限和角色请求活动是常见行为，但我没有在设备或上游源码里核实。
- 是否已在真实任务里发生：没有（11 次 `open` 都是 done）。
- 影响：A 会让调用者看到 UNKNOWN 和"不要重复"的提示，多等约 3 秒，然后按提示去观察。不会诱发重复启动，主要是困惑和耗时。B 会让调用者拿不到能说明"为什么没确认"的树，只能多一次 `snapshot`。
- 最小修正：主屏上首次确认已得到 done 之后，观察完不必再要求目标仍在顶层（主屏不存在任务搬动，`recordMovement` 在主屏直接返回 `null`，`:401`）。后台屏上的第二次确认只保留搬动检查，不因"顶层换成了别的包"而降级。另外，落地确认最多只用到"总预算减去一个观察窗口"，给观察留出时间。维护代价：几行代码，改两个测试的期望（`ActivityLaunchTest.kt:173` 的"recheck actual top"是有意的设计，需要一并改名）。
- 分类：设计取舍（源码推断）；进下一轮。建议在整理提交时一并改，因为 F2 的修正也要动同一段流程。

### `open` 的其余结论

- 延后弹出的默认浏览器问询：现在的 `open` 回执看不出来，这次准备阶段要靠下一次 `snapshot` 才发现（实测）。修了 F2 以后，如果对话框在观察窗口里出现，会直接体现在附带的树里；修了 F3 以后，不会因为它把 `done` 改成 UNKNOWN。
- 附录一"首页示例把 `key enter` 改成 `fill --submit`"已落实（`SKILL.md` 第 46 行）。【源码】

---

## 三、`fill`、`type`、`key`、`fill --submit` 的输入后验

- **读回规则**（`UiActions.kt:700-785, 812-862`、`actions.md:142-153`）：写入被平台接受之后，对同一个原节点在观察窗口内轮询读回。一致是 done；不同（例如自动补全改了字）是 UNKNOWN 加退出码 3，并附原样的 `actualValue`；读不到也是 UNKNOWN。`fill` 的"不同"不报 failed（`UiVerification.kt:139`），因为 App 可能做了格式化，这样处理是合理的。【源码】
- **节点在屏外或被第 13 条改标为不可见**：写入之前，不可见的节点不会被接受（`UiAction.kt:54, 95`）。写入之后读回时，如果节点不再可见，结果是 `verificationError="The original filled field is no longer visible"` → UNKNOWN 加退出码 3（`UiActions.kt:841-842`），默认文本里有 `verificationError=…` 和"不要自动重复，先用只读观察核对"。调用者的下一步是 `find` 或 `snapshot`（`find` 能搜到屏外文字）。第 13 条（`NodeFacts.kt`，可见改为"系统认为可见，并且与所属窗口相交"）会让这个分支稍微容易触发，但前提是写入之后字段被移出窗口，在后台屏没有输入法的情况下少见。其实节点的文字在不可见时仍然读得到，可以照常比对，见 F5。【源码推断】
- **密码框**：`fill` 写入被接受即 done，`verification=unobservable`（`UiActions.kt:747-751`），与 `actions.md:130-131` 一致。`--submit` 对密码框跳过值比对，直接按原目标提交（`:791-807`）。【源码】
- **输入法弹出遮挡**：读回走无障碍节点，不受遮挡影响。`click --touch` 本轮新增"更高层窗口盖住中心点就以 `covered` 拒绝"（`AppInput.kt:157-175`），输入法窗口盖住时会被拒，不会点到输入法上。【源码】
- **中途失败时保留前几步**：每一步开始前都会 `run.remember`，异常只改当前一步（`FillProgress.fail`，`FillProgress.kt:51-58`），汇总规则在 `:94-107`。例如聚焦成功、写入成功、提交失败时，回执为 `fill: failed; …; fill=done, verification="matches"; submit=…`；提交回执未知时，整条是 UNKNOWN，各步都保留（样本 `enter-up-failure`）。`fill` 是整字段替换，调用者看到 failed 再执行一次也不会重复插入。【源码＋样本渲染】
- **A4**：成功时 `type: done; via=keys`、`key: done`、`back: done; change=…`，不再打印按键进度 JSON。`fill` 成功时只打印 `fill=done, verification="matches", actualValue="…"` 和 `submit=done`，不打印 `sent`、键位等过程信息（`ui_output.py:487-506`）。A4 已落实。【源码＋原件】

### F4（旁见，噪声）成功回执里仍有少量冗余

- `fill` 成功时首行回显 `actualValue`，与刚输入的文字相同，树里也有 `value="…"`。20 次运行 24 次 `fill`，合计约 2.1 KB。
- `scroll-until` 成功时，因为"请求的动作名"（`scroll-until`）和"选中的动作"（`scroll-down`）名字不同，`selection_lines`（`ui_output.py:547-597`）会打印 `Requested`／`Resolved`／`Selected` 三行。15 次合计约 5.3 KB。`fill --submit` 退回 Enter 时也会多打 6 行，内容与首行的 `submit=done, action=key, key=enter` 重复（样本 `enter-done`）。
- 借用粘贴干净成功时仍打印 `effect=matches; paste=attempted; restore=restored`。
- 规模：以上合计不到 20 次运行默认输出（约 947 KB）的 1%（`noise.py`）。【重算】
- 分类：可选建议；不必处理。`scroll-until` 那三行可以由负责滚动查找的同事顺手判断：把"`scroll-until` 选中 `scroll-<方向>`"视为同一动作即可。

### F5 读回时节点"不再可见"一律报不可观察

- 现象：`awaitValue` 只要节点不可见，就不比对文字（`UiActions.kt:841-842`），其实这时文字照样读得到。
- 影响：极少触发；触发时报 UNKNOWN，结论是诚实的，只是多一步观察。
- 分类：可选建议；不必处理。如果要改，改成"不可见时照常比对文字，并附一条不可见的说明"。

### F6 密码字段上 `fill` 与粘贴的结论不对称

- `fill` 报 done 加 `verification=unobservable`；粘贴报 UNKNOWN 加退出码 3。
- 分类：设计取舍；不必处理。粘贴之所以更保守，是因为"PASTE 是否真的生效"只能靠读回证明，而 `fill` 的整字段写入有平台的接受回执。

---

## 四、"已派发但没确认后态"的一般规则

- **设备端**（`UiRun.kt:104-148`）：`remember` 记下的事实优先。动作结果一旦记下，之后观察失败、超时、交付失败都只加 `snapshotError` 这类字段，不改主结果（`failure()` 先取已记下的事实）。派发已开始但还没有事实时，`missingFactOutcome` 给 UNKNOWN；没开始给 not sent。早期回执（`response(expired=true)`）用的是同一份事实。【源码】
- **动作后观察**（`UiPostAction.kt:32-95`）：取样异常只记成 `missing`，不抛出、不改主结果。`reference.md:55-57` 写的"普通的观察缺口不改主结果"与实现一致。【源码】
- **主机端**：设备连接中途断开（`TransportFailure`）报 UNKNOWN，退出码 3（`ui_client.py:170-176`、`coordinator.py:195-197`）；发出之前就失败报 not sent（`coordinator.py:198-205`）。UNKNOWN 首行之后固定加"不要自动重复，先用只读观察核对"（`ui_output.py:284-296`）。常驻进程死亡属于传输失败，按 UNKNOWN 处理。【源码】
- **有没有把"已派发"写成"未发出"、或者反过来**：没有发现。最接近的是 `fill --submit` 退回 Enter 时目标 UID 对不上的情形（样本 `enter-first-mismatch`），首行写 `fill: not sent; dispatchStarted=true; …; submit=not sent, …, sent=true`。平台按 UID 拒收了这次注入，主结果 not sent 是对的；但 `sent=true` 和 `dispatchStarted=true` 表示的是"尝试过"，读起来自相矛盾。这是旧行为，本轮没有改动。分类：可选建议，不必处理；以后如果要改，把步骤里的 `sent` 改名为 `attempted`。【样本渲染】
- **`open`**：首次确认之前，以及启动码可接受但还没确认时，都是 UNKNOWN，不会说成"未发出"；平台明确拒绝时是 failed，启动调用丢失回包时是 UNKNOWN（`ActivityLaunch.kt:263-270`）。【源码】

### 后台屏题 5 次的输入类回执（`E/r6-v1-candidate-01…05`）

- 5 次都是同一条流程：`session open --display virtual` → `apps` → `open` →（3 次多一次 `snapshot`）→ `click`（新建）→ `fill`（名字）→ `click`（更多字段）→（3 次 `find Notes`）→ `scroll-until` → `fill`（备注）→ `click`（保存）→ 返回列表、查看 → `status` → `session close`。
- `fill` 10 次全部 done，首行形如 `fill: done; change=changed at content,e…:text,showingHintText,focused,actions; fill=done, verification="matches", actualValue="Local validation only"; snapshotPath=…`，后面是树。**没有一次 `fill` 之后再 `snapshot` 确认。**
- 能省掉的调用：3 次 `open` 之后的 `snapshot`（F2）。另外 3 次 `find Notes` 拿到 `matches=2; offscreenMatches=2` 之后接着做了 `scroll-until`，那次 `find` 可以省掉。这属于查找与滚动的专题，这里不展开。
- 对调用者没用的内容：上面 F4 那几项，以及 `change=… at e…:text,showingHintText,focused,actions` 这类字段列表。量都很小。

---

## 五、这一块有没有把简单的事做复杂

- **借用粘贴**：本轮的改动其实是一次简化。原来的主结果汇总只在 `execute` 末尾算一次（`fb75afc`），早期回执用的又是另一套状态；现在统一放进 `save()`，每次记下事实都用同一个函数（`BorrowedPaste.kt:334-360`，净增约 25 行）。内部评审的 Must-fix 2 正是靠这样才修好的。契约里粘贴共 9 条样本，本轮只改了 2 条。代价可以接受。
- 不过整条借用粘贴通道（`BorrowedPaste.kt` 402 行、`ClipboardAccess.kt` 211 行、`BorrowedPasteTest.kt` 534 行，外加 9 条契约样本）在两轮共 55 次真实任务里一次也没用过，维护量与用量明显不成比例。为罕见的恢复组合维护的分支主要是 `save()` 里那 6 个优先级分支和 `cleanup()` 的失败与未知判定（`:280-283`），已经是最少的写法，再合并就会丢掉"恢复失败"和"恢复未知"的区别。我不建议在这一批里动它。它是"按位置插入非 ASCII 文字"唯一的通道（`type --via keys` 只能映射键盘能打出的字符），删掉需要用户另行决定，不属于本轮的反证。
- **可以合并的地方**：
  1. 粘贴失败加恢复未知时，现在报 UNKNOWN（`:340`）；按"主结果说主动作、退出码说整条命令"的总则，应当是"failed 加退出码 3"，与第 22 条对称。合并以后 `save()` 少一个分支，规则也只剩一句："主结果看 PASTE 和读回，退出码再叠加恢复"。不过第 22 条明说"粘贴本身没成功的情形保持现状"，所以这只是下一轮的可选建议。
  2. `open` 的两次 `confirmLanding`（F3）：主屏上第二次可以删掉，后台屏上的第二次只留搬动检查。这样 `observeLanding` 里的四个提前返回条件可以减到两个。
- **旁见**：`41c25c6` 在 `UiPostAction.kt:255-284` 给"导航后可能仍在加载"加了一份固定的文字表，里面有 `"正在查找最佳路线"`、`"finding the best route"`，这是路线题（Maps）专用的文字（第六轮 B2 授权"界面树里有明显加载文字"，具体判定由实施方定）。功能上无害，只影响 `change=loading` 的标注，但它是对固定测试任务的特化，会让路线题的结果比真实使用好看一点。建议改成通用的"loading"类词，或者交给负责结果判定的专题评估。分类：设计取舍；进下一轮。

---

## 六、已核实无问题

1. 第 22 条：效果已核对一致时，恢复失败报"完成加退出码 1"，恢复未知或进行中报"完成加退出码 3"；恢复完成以后会清掉临时的退出码（`:345`）；粘贴没成功的组合保持原状。【源码＋主机测试】
2. 恢复晚于动作后观察；恢复前先比对剪贴板是否仍是本次借写的内容；用户在中途复制的新内容会被保留（`copy-skipped`）。【源码】
3. 早期回执与最终回执规则一致：核值完成前是 UNKNOWN，核值一致后、恢复完成前是"完成加退出码 3"（`BorrowedPasteTest.kt:268` 的三阶段用例）。【源码】
4. `input.md`、`reference.md` 的说法互相一致，并与实现一致；契约样本 `restore-failed`／`restore-unknown` 已经重新导出。【源码＋主机测试】
5. 粘贴没成功时不会报"完成"；字段原样未变时不会被当成粘贴成功。【源码】
6. `open` 成功时只打印 `display=`（有搬动时加 `taskMovement=`）；失败或 UNKNOWN 时启动细节完整，并附"不要重复"的提示。【主机渲染】
7. 内部评审的三项 Must-fix（观察耗尽预算时保留已确认的 done；早期回执里已核值的粘贴报"完成加退出码 3"；首次确认"落在别处"之后不会再被改成 done）在源码里都已修好（`ActivityLaunch.kt:221-236, 368-376`；`BorrowedPaste.kt:334-360`）。【源码】
8. `fill` 中途失败时保留已做的步骤；写入后读回的三种结果（一致、不同、读不到）与 `actions.md` 一致；密码框写入报 done 加 unobservable，与指引一致。【源码】
9. A4 在输入类命令上已落实（`type`、`key`、`back`、`fill` 各步成功时不打印过程计数），真实回执可证（`E/r6-v1-*`、`E/r6-c1-*`、`E/r6-c2-*`）。【原件】
10. 首页示例已从 `key enter` 改成 `fill … --submit`。【源码】
11. "已派发"与"未发出"的区分在设备端（`UiRun`）、主机端（传输失败与发出前失败）和 `open` 里一致，没有写反的地方。【源码】
12. 后台屏题 5 次填表流程里，`fill` 之后都没有为了确认而再 `snapshot`，`fill` 的回执足够调用者直接进入下一步。【原件】

## 七、未能核实的事项

- 应用调起的权限、角色请求对话框是否会成为应用任务的顶层活动（F3 的前提）。需要设备，或者核对 permissioncontroller 的 manifest 和 ActivityStarter 的任务归属。
- 后台屏上应用窗口在无障碍窗口列表里出现前的真实延迟分布（F2 修正后要多等多久）。原件只能看出"超过 1.5 秒"。
- 恢复剪贴板失败的真实触发频率：两轮原件都没有用过借用粘贴，所有结论来自源码和 JVM 测试，没有设备上的实例。
- `click --touch` 的"被覆盖"判定在带全屏透明系统窗口的厂商 ROM 上会不会误拒：源码按层级和窗口边界判断，没有设备可核。
- 我没有运行 Gradle 或 JVM 测试。Kotlin 侧的行为只有源码阅读和实施方提交的测试日志作依据。
