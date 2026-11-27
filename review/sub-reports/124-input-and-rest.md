# 任务 124：输入、粘贴、剪贴板、通知、文件、图像、系统动作、原因码（第八轮，对 `409e853`）

记号：`S` = 本次评审 scratchpad 根；`K/` = `android/app/src/main/java/com/rainng/androidctl/`；`H/` = `androidctl/src/androidctl/`；`EV` = `S/r8/ev/ultragoal/2026-11-11-next-improve/stages/42-r4-decisions`。我的脚本和中间结果在 `S/agents14/124/`（`runs.py`/`runs.json` 统计 48 次正式尝试的命令与非零退出；`show.py` 打印某次尝试的命令与输出；`roll.py` 检索模型可见的回答文字；`extract.py`/`out-409.json` 是独立的原因码抽取；`probe/` 是对登记测试的注入试验）。没有设备；设备行为只来自源码推断或实施方原件，需要设备的地方单独标出。

"四块能力"指第七轮第 21 项裁决里"只删不补"的四块：借用粘贴（`type --via paste`，借用户的剪贴板发一次 AX 粘贴，事后把剪贴板恢复）的恢复规则、后台屏（虚拟屏）上的任务保护、"完成但退出码不是 0"的组合、原因码。

## 结论（先看这里）

1. **本专题范围内没有挡合入的确定缺陷**。三条文字输入路线（`fill` 走无障碍设置文字，`type --via keys` 走按键事件，`type --via paste` 走借用粘贴；`ime` 只发输入法动作）之间产品不会自动换路线，所以不会因为回退而重复输入；48 次实测里 7 次 `fill --submit` 报 UNKNOWN，模型都先看新树再只补发 `ime`，回执写明 `submit=not sent`，没有重复提交（实测）。
2. **借用粘贴的"残留输入"提醒（第七轮第 18 项）在它覆盖的两种情形下准确，但覆盖面窄于真实风险**：只在"粘贴已核对一致 + 恢复 failed/unknown"时出现；粘贴本身失败或 UNKNOWN、而恢复也失败或未知时（借写已经发生，剪贴板同样可能留着输入），没有这句；恢复仍在进行（`restore=pending`，退出码 3）的早期回执也没有。建议只补一句指引（零代码），见 F1。
3. **四块能力本轮没有新增违反裁决的规则**；实施方也**一次都没有动用"删或降级"的授权**（交付结论与小项表都写明"不加规则"，我在差量里也没找到删减或降级），所以不存在需要复核的能力代价。本轮在这四块里的改动全部能对应到已授权条目（第 18 项、下-16/17/18、小-72、退出码统一的"不改行为一半"）。
4. **原因码**：码表仍是 150 个码，与源码构造双向一致（独立抽取器复现）；本轮改了 9 行（diff 18 行），全是文字，没有增删码。删掉两组自检用例之后，登记测试仍能拦住新码未登记、码表留下无发出方的码，以及多种非常规写法（注入 14 种写法实测，只有"新定义 `Fault` 子类"一种静默漏过）。
5. **码表这个机制对调用者几乎没有作用**：码表不随产品发布，指引只点名 7 个码，模型从来看不到"actions"那一列；48 次尝试 1,096 条命令里 35 次非零退出，见到的码只有 `stale`、`notReady`（基线另有 `staleRef`、`arguments`），模型的下一步每次都由回执里那句话决定，最终答复里提到码只是当标签用（实测）。建议用户决定是否把码表缩成"码 + 一句话"或去掉没有消费者的"类别"列，见 Q1。

---

## 一、各主题的行为链

### 1.1 文字输入的三条路线：选择与回退

- **选择**完全由调用者显式决定：`fill REF TEXT` 整段替换；`type [REF] TEXT --via keys|paste` 用于插入位置或选区（`skills/androidctl/input.md:3-4`；`H/ui_commands.py:1105-1134` 要求 `--via`）。设备端没有任何"fill 失败转 keys/paste"的自动路径：`K/UiActions.kt` 里 fill、`keyOnTarget`、`NodePasteTarget` 各自独立，`fill --submit` 只在 IME 动作和 Enter 键之间二选一（`K/UiActions.kt:826-827,1128-1129`）。交付结论里"没有新增自动输入 fallback"与源码一致。【源码推断，核对无问题】
- **实测**：
  - Chrome 新标签页的假搜索框 7 次 `fill URL --submit` 都报 `fill: UNKNOWN`、`fill=UNKNOWN, reason="The original filled field is no longer visible"`、`submit=not sent`（`EV/efficiency/r7-new-04` 第 4 条、`-04-retry1` 第 3 条、`-08`、`-08-retry1`、`-09`、`-09-retry1`、`-10` 各第 3 条）。原因是 Chrome 把焦点和文字移到了另一个节点 `url_bar`。模型 7 次都没有重发 fill，而是对新树里的 `url_bar` 只发 `ime`（例如 `r7-new-08` 第 4、5 条）。没有重复输入，也没有重复提交。代价是每次多 1～2 条命令；这是回执如实说"不知道"的结果，不是缺陷。【实测】
  - Collabora 表格（`r7-new-06` 第 13 条、`-06-retry1` 第 12 条）：`fill` 报 UNKNOWN，`focus=UNKNOWN … sent=true; fill=not sent`。模型随后 `key ctrl+a` + `type '009731 --via keys`。fill 没有发出，所以换路线不会重复写入。这是调用者自己换路线，回执给的事实足够让它判断。【实测】
- 48 次尝试里 `type --via paste` 0 次，`clipboard *` 0 次（第七轮 57 次也是 0 次）。【实测，`S/agents14/124/runs.json`】

### 1.2 借用粘贴与"残留输入"提醒

流程见第七轮记录 92 第一节，本轮流程未变。本轮只在 `K/BorrowedPaste.kt:351-360` 的 `save()` 里加了一句固定 reason：主结果为 done（即平台回执 done 且读回 `matches`）而 `restore.state` 为 `failed` 或 `unknown` 时写 "The input text may remain on the clipboard; if it is sensitive, tell the user and get consent before clearing it"；退出码规则（failed→1，pending/unknown→3）不变。契约样本两条（`contract/wire-contract.json:7808,8060`）和主机渲染测试（`androidctl/tests/test_wire_contract.py:1392-1397`，断言这句出现在默认文本首行）同步。主机端定向测试我跑过：`test_text_input/test_clipboard/test_notifications/test_image_delivery/test_input_keys/test_jobs/test_reason_codes/test_file_jobs` 209 passed，`test_wire_contract.py -k "paste or restore or region or notification or clipboard"` 20 passed。【主机端实际运行】

逐个核对这句话在哪些状态下出现、是否准确（源码推断）：

| 恢复状态 | 怎么来的 | 剪贴板里实际可能有什么 | 有没有提醒 | 判断 |
| --- | --- | --- | --- | --- |
| `failed`（准备恢复失败、调用被拒） | `cleanup()` catch，`K/BorrowedPaste.kt:283-287` | 借写的输入文字 | 有 | 准确 |
| `failed`（恢复后读回与原内容不同） | `restoreOriginal()`，`:303-309` | 借写的文字、或别人新复制的内容 | 有 | "may remain"措辞恰当 |
| `unknown`（读不到当前剪贴板 / 恢复后读回不可观察） | `:265-272`、`:303-309` | 都可能 | 有 | 准确 |
| `skipped`（剪贴板已经不是借写的那份） | `:274-277` | 不是输入文字 | 无 | 准确 |
| `pending`（回执先于恢复完成发出） | `cleanup()` 开头，`run.response(expired=true)` 用最后记住的事实（`K/UiRun.kt:160-185`） | **此刻一定还是输入文字**；恢复多半随后完成，但调用者无从得知 | **无**，退出码 3 | 见 F1 |
| 粘贴 failed/UNKNOWN，且恢复 failed/unknown | `save()` 的 else 分支：`Borrowed paste did not complete: effect=…, cleanup=failed` | 输入文字（借写已确认时概率更高） | **无** | 见 F1 |

另外核对过：锁屏时借写之前就会被 `dispatchInput → targetInputReady` 拒绝（`K/UiActions.kt:282-283`，`K/BorrowedPaste.kt:153`），借写没有发生，`restore=not required`，不会误留输入。【源码推断】

### 1.3 坐标触摸（`click REF --touch`）与坐标手势

- 本轮小-34 把触摸点从"整节点中心"改成"节点与所属窗口交集的中心"（`K/AppInput.kt:151-155,179-192`），窗口缺席的检查提前到了显示范围检查之前。核对：交集为空报 `notReady`；窗口缺席报 `changed, subject=selection`；中心出屏报 `point`；被更高层窗口盖住报 `covered` 并给出窗口编号；窗口列表已经按目标屏过滤（`K/AccessibilityConnection.kt:51-65`），主屏的悬浮窗不会误判为盖住后台屏的点。没有扩大触摸资格。【源码推断，核对无问题】可见变化：窗口缺席且中心出屏的节点，以前报 `point`，现在报 `changed`，两者下一步都是重读，影响可忽略。
- 实测 2 次 `click --touch`（`r7-new-02` 第 9 条、`-02-retry1` 第 17 条），都 done。部分越出窗口的节点没有在实测里出现，交集中心是否点准**需设备**。
- 冗余：`nodeTouchPoint` 先检查节点矩形非空（`:150`），`windowTouchCenter` 又检查交集非空（`:188`）；前者是后者的特例，只是文字不同。可删一处，收益很小（可选，见 4.1）。

**锁屏、熄屏、后台屏的口径**（任务第 5 点）。所有会发出输入的路径在派发前都经过 `UiBackend.targetInputReady`（`K/UiBackend.kt:79-97`）：主屏要求可交互且解锁；后台屏要求主屏可交互、解锁并且亮屏，后台屏本身亮屏。AX 节点动作、按键、`notifications`（拉下通知栏）、坐标手势、`--touch`、借用粘贴都走这一条。唯一例外是主屏上的 `swipe` 可以越过"无密码的滑动锁"（`K/VisualInput.kt:308-313`、`K/UiBackend.kt:89-95`），后台屏上没有这个例外，与 `images.md:121-123` 一致。只读路径（截图、区域截图、`snapshot`、`notifications list/get`）不检查锁；`clipboard` 不拒绝，但锁屏时"剪贴板为空"不被接受为已知事实（`K/ClipboardAccess.kt:144-146`），所以锁屏时 `clipboard clear` 的结果会是 UNKNOWN（**需设备**确认锁屏时 shell 读剪贴板的实际返回）。主机端屏幕行的标记把锁屏和主屏不可交互标成主屏的事实、把熄屏标成目标屏的事实（`H/ui_view.py:505-511`）。**结论：输入、触摸、截图三处对锁屏的处理口径一致，没有矛盾。**【源码推断】唯一的小毛病：后台屏会话里主屏锁着时，拒绝文字是共用的 "Display must be interactive and unlocked for the current user"（`K/DeviceFacts.kt:69-74`，经 `mainReadyForWorkDisplay` 调用），没说是主屏；调用者读到它可能以为后台屏有问题。改一句文字即可（可选，F4）。

### 1.4 区域截图（`screenshot --region` / `screenshot REF`）

- 坐标区域：用"最近一张完整图"的交付像素校验，再按原生尺寸向外取整（`K/ImageRegion.kt:142-152,210-224`）；`floor/ceil` 保证宽高至少 1，右下界不超过原生尺寸。截的是新像素，不是旧图裁剪；裁剪前后都核对显示布局不变（`K/VisualInput.kt:143-169`）。与 `images.md:44-52` 一致。【源码推断，核对无问题】
- REF 区域（下-18 第二处）：目标不可见报 `notReady` 加可见性建议；边界为空、为负或超出整屏报 `notReady`（`K/ImageRegion.kt:155-183`）；显式 `--region` 写错仍报 `arguments`、越界仍报 `region`。第七轮指出的误报 `arguments` 已修好。【源码推断】
- 口径差异：触摸现在用"节点 ∩ 窗口"，REF 截图仍用节点原始边界，并且只要有一点越出屏幕就整个拒绝。平台给的节点边界通常已经按父视图裁过，这种情形多半罕见，**需设备**。不建议为此加规则。
- 新加的六个不等式（`:167-173`）与 `rectangle()`/`within()` 是两份平行写法，代码量很小，可以不动。
- 实测：48 次里 `screenshot REF`、`--region` 都是 0 次。

### 1.5 通知

现行产品只剩读取：`notifications.list`、`notifications.get KEY`（`K/NotificationOperations.kt:7-17`）；通知动作在前几轮已经删除，任务书说的"操作的身份匹配"已经不存在。`get` 用完整 key 精确匹配，只在用户 0 与 -1 的通知里找，找不到报 failed（`K/NotificationProjection.kt:39-42`）。监听器注册、一次"未知监听器"修复、租约结束注销都还在。实测 1 次 `notifications list`（`r7-new-01` 第 22 条）输出正常。本轮唯一改动是预算估算里写退出码改走 `setExit`（`:262`）。顺带一提：这一处只是估算回执大小用的"最坏情形"副本，真实通知回执从来不带 `exit`，所以把它算作"8 个设备端退出码写点"之一有点牵强，无害。【源码推断】

### 1.6 系统动作

`K/SystemActions.kt:44,57,189-199` 本轮让 back/home/recents/notifications/key/type 和 `open` 的动作后观察用"最近一份完整树"作基准（第七轮第 12 项），`open` 的观察截止改用 `operationUntil`（第 13 项，它的正确性归 `open` 专题）。后台屏上 home/recents/notifications 仍拒绝为 `displayUnsupported`。没有新的锁屏或任务规则。【源码推断】

### 1.7 文件传输

本轮只有两处退出码写法改成 `set_exit`（`H/file_jobs.py:136,568`），行为不变。抽查了部分完成和清理链（`H/file_jobs.py:503-571`）：先清设备端，再关流、删主机临时文件（只有拉取且未发布、并且身份仍是自己创建的那份时才删）、关目录；清理失败一律记 `cleanupError` 并把退出码设为 3；"完全没开始"才改报 `not sent`，并且前提是清理没有出错。保守之处：只要本地流或目录关不掉，这个目的地会一直被占用，直到守护进程退出（`:575-577`）。这是有意的设计，实测 0 次文件传输，不建议动。`job_commands.show` 删掉单独一行 `code=…`，改到首行打印（`H/output.py:39-45,71-79`），信息没有丢。【源码推断】

---

## 二、发现

### F1 残留输入的提醒覆盖面窄于真实风险（证据不足 / 可选建议）

- 位置：`K/BorrowedPaste.kt:351-367`（`save()`）；`skills/androidctl/input.md:61-66`。
- 触发：(a) 平台对 PASTE 报失败或 UNKNOWN，或读回不一致或不可观察，而恢复又是 `failed`/`unknown`；(b) 回执在恢复完成前发出，`restore=pending`、退出码 3。
- 现象：(a) 首行是 `type: failed|UNKNOWN; Borrowed paste did not complete: effect=…, cleanup=failed`，没有提醒剪贴板可能留着输入；(b) 首行是 `type: done`、退出码 3、没有 reason，指引说的是"These receipts warn"，读者会以为退出码 3 的回执都带这句话。测试 `anUnconfirmedPasteDoesNotReceiveTheVerifiedPasteCleanupNotice`（`BorrowedPasteTest.kt:244-254`）把 (a) 的"没有提醒"写成了预期。
- 影响：只在输入的是敏感内容（例如密码）时有影响，调用者不会想到要告诉用户。不会造成重复输入。真实运行 105 次里 0 次使用借用粘贴。
- 依据：第七轮第 18 项字面只说了"恢复失败为完成+1、恢复未知为完成+3"这两种情形，实施方按字面做，**不算违反裁决**。但第七轮记录 92 的 F1 原意是"恢复不是成功状态时"。
- 建议（最小代价，零代码）：在 `input.md` 那一段末尾补一句："Whenever `restore` is failed, unknown or pending, the clipboard may still hold the input text." 代价是一行指引。若要把代码也补上（把 `pending` 加进条件，或非 done 时把这句接到 reason 后面），会改契约样本，属于四块之内的"补"，需要用户点头，我不推荐。
- 验证：文档改动无需设备。

### F2 注入试验：登记测试删掉自检用例之后还保护什么（核对无问题，附一处已知漏检）

在 `S/agents14/124/probe/` 里复制源码，逐个注入写法后跑 `test_reason_codes.py`【主机端实际运行，`probe/run.sh`】：

| 注入写法 | 结果 |
| --- | --- |
| Kotlin `Rejected("brandNewA", …)` | 失败，列出未登记码 |
| Python `Fault(code="…")`、`protocol.Fault("…")`（本轮补的） | 失败，列出未登记码 |
| Kotlin `put("code", "…")`（本轮补的） | 失败，列出未登记码 |
| Python `{"code": "…"}`、`Problem(code=…)`、`Result(error={"code": …})` | 失败 |
| Kotlin `"code" to` 换行写值 | 失败 |
| 码表多一行没有发出方的码 | 失败，列出该码 |
| `put("code", …).put(…)` 链式、`"code" to` 放在参数末尾、`Rejected(code = …)` 命名参数、嵌套 `json("code" to …)` | 报"Review dynamic"失败（误报，但会明确失败，不会静默放过） |
| 新定义 `class LocalFault(Fault)` 再抛 `LocalFault("…")` | **静默通过**（设计上不做继承发现，第七轮已知） |

结论：删掉的两组自检用例只是检验测试本身；主测试是双向比对，抽取器如果坏到少收，会立即报"码没有发出方"，所以不需要自检。补的 5 到 7 行确实堵住了第七轮记录 94 指出的四种漏检。现存产品里只有 `TransportFailure` 一个子类，并且已经点名。不需要再改。

### F3 码表 9 行文字改动逐行核对（核对无问题，附一处遗留混用）

本轮 diff 的 18 行就是 9 行码表各改一次：`busy`、`certificateClock`、`certificateExpired`、`collectionLimit`、`connectionBusy`、`scopeUnknown`、`stale`、`tlsIdentity`、`workerUnavailable`。逐个对照了构造处：

- `certificateClock`/`certificateExpired`：主机端 `H/transport.py:93-102` 按 OpenSSL 的 verify_code 改用这两个已有的设备端码，消息写了核对两端时钟；码表的新文字同时覆盖主机端和设备端（`K/DeviceIdentity.kt:49-53`）。一致。
- `tlsIdentity`：码表去掉了时钟和过期两种情形，补上"欢迎消息里的设备不符时核对地址"（`H/transport.py:308`）。一致。
- `scopeUnknown`：与 `K/UiScope.kt:88-97` 的消息一致，只列 App 窗口。
- `stale`：与实测回执 "use the new observation or run snapshot" 一致（`r7-c1-candidate-03` 第 3 条）。
- `busy`：自动化工具占用那一句与 `K/AutomationCleanup.kt:37-41` 一致。**遗留**："For other holders, respect the current owner and take over only on user instruction" 套在 `K/Sessions.kt:228,296,323`（"Active ui must finish"，占用者往往就是调用者自己还没结束的请求）和 `H/coordinator.py:281,411,655,696` 上并不贴切，这些情形下一步是等待后重试，谈不上"接管"。它属于用户第 20 项明说"不动"的 20 个混用码之一，只记录，不建议本轮动。
- `connectionBusy`、`workerUnavailable`、`collectionLimit`：与构造处消息一致（`H/transport.py:146-160` 等），具体时序归连接专题。

### F4 后台屏会话里主屏锁着时，拒绝文字没有点名主屏（可选建议）

`K/DeviceFacts.kt:69-81`。改成 "Main display must be interactive and unlocked …"，或者由 `mainReadyForWorkDisplay` 自己给文字。一行，不改码、不改规则。实测 0 次。

### F5 模型把退出码报错了一次（观察，不属于产品缺陷）

`r7-new-03-retry1` 的最终答复写"stale 拒绝，明确 not sent、退出码 1"，实际 `childExit=2`（cli-index 第 11 条）。原因码和结果词它都读对了，退出码记错。说明调用者在报告里依赖首行的结果词，不怎么看退出码；不需要处理。

---

## 三、四块能力"只删不补"的复核（任务第 2 点）

| 块 | 本轮改动 | 是否新增规则 | 依据 |
| --- | --- | --- | --- |
| 借用粘贴恢复规则 | 只加了一句 reason（`K/BorrowedPaste.kt:352-360`）；退出码改用 `setExit`；删掉 editor 通道的墓碑测试（小-56） | 否（第 18 项明确允许加文字） | diff 见上 |
| 后台屏任务保护 | `open` 在后台屏上已确认之后只做搬入检查和"目标出现在别的屏"的改报，删掉了"顶层换包就改报 UNKNOWN"（`K/ActivityLaunch.kt:225-262`） | 否，是一次删减（小-72，第 8 问 B）；保留"出现在别的屏"属于评审方推荐的暂行新-6B，仍待用户确认 | 新-6 |
| 完成但退出码不是 0 | 两种语言各加一个只赋值的 setter，exit=3 常量合并成一个；`fill` 在 done+非零退出码时显示顶层清理原因（下-17） | 否（第 12 问 A；下-17 已授权） | `K/Wire.kt:9,56`；`H/protocol.py` `set_exit` |
| 原因码 | 9 行文字；主机端 TLS 两种情形改用已有码；`connectionBusy` 的请求票据带上截止时间（下-18 第四处） | 不新增码、不并码；票据带截止时间是授权范围内的一项新状态 | 第 20 项、下-18 |

**实施方没有使用"直接删或降级"的授权**（`r7-small-items.md:27`、`deliverables/r7-candidate.md:30`，我在差量里也没找到相应的删除），所以没有需要复核的能力代价记录。按第七轮第 21 项的本意（"出了缺陷优先删或降级"），本专题里唯一可以考虑用上这个授权的是 F1：与其把提醒补全，不如在指引里统一一句（见 F1），这不算"补规则"。

---

## 四、可删、可并与无人使用的路径（任务第 4 点）

### 4.1 小的冗余（可选，收益很小）

- `K/AppInput.kt:150` 与 `:188` 两次检查矩形非空，可以删掉前一处。
- `K/ImageRegion.kt:167-173` 与 `rectangle()`/`within()` 是平行的两份几何校验。
- `K/NotificationProjection.kt:262` 给预算估算写了一个真实回执从不出现的 `exit` 字段，可以删掉（省几个字节的估算），也可以不动。

### 4.2 零使用的能力（设计取舍，需要用户定）

105 次真实运行（本轮 48 次加第七轮 57 次）里，借用粘贴、`clipboard get/set/clear`、文件推拉、`shell`、`screenshot REF/--region` 都是 0 次；坐标手势只出现在基线组和 2 次 `pinch`、1 次 `swipe`。其中借用粘贴是这些里维护负担最重的：`K/BorrowedPaste.kt` 407 行，加上 `K/UiActions.kt` 里的 `NodePasteTarget` 约 80 行、`ClipboardAccess` 的借写部分，以及 `BorrowedPasteTest.kt` 544 行。它也有真实的能力空缺可以填：`type --via keys` 只能打虚拟键盘映射得到的字符（`K/KeyPlan.kt:102-117`），中文等字符在不能用 `fill` 的地方（需要插入到光标处、或控件不支持设置文字）只能靠借用粘贴。见 Q2。

---

## 五、原因码作为机制是否值得维护（任务第 3 点）

- **调用者看不到码表**：`contract/reason-codes.json` 只被 `androidctl/tests/test_reason_codes.py` 读取，不在 `skills/` 里，也不打包进主机端（`git grep reason-codes` 只命中测试、`contract/README.md` 和 `AGENTS.md`）。指引正文只点名 7 个码：`arguments`、`busy`、`changed`、`connectionBusy`、`covered`、`stale`、`unexpectedTaskMove`。【源码】
- **实测**（`S/agents14/124/runs.json`）：48 次尝试 1,096 条命令，非零退出 35 次。带码的：候选版本上 `stale` 5 次、`notReady` 6 次；基线上 `staleRef` 2 次、`arguments` 1 次（`--json` 写错了位置）。其余非零退出没有码（failed/UNKNOWN 的动作结果）。每一次的下一步都能由回执的那句话直接得到：`stale` 之后用附带新树里的引用，或者先 `snapshot`；`notReady` 之后换控件或换做法。模型最终答复里出现码名 7 次（`stale` 5 次、`scopeUnknown` 2 次），都只是当作标签写进报告，例如 "not sent / stale，实际退出码为 2"（`r7-c1-candidate-03` 第 145 行）。推理内容是加密的，看不到它在思考时有没有用码。【实测】
- **维护代价**：152 行码表，每加一个码都要写一行"下一步"，登记测试只核对码在不在，不核对文字对不对；第七轮的 `sessionChanged` 文字就是码表与回执各写一份、互相走样的例子。`category` 一列没有任何消费者（第七轮记录 94 的 F3，本轮仍是这样）。
- **判断**：码本身值得保留：设备端和主机端约 25 处按具体码分支，指引也用它作稳定的锚点。码表的"下一步"一列和"类别"一列对调用者没有作用，只是给维护者的底稿；登记测试防的是"码没登记"，这件事本身对调用者也没有后果。见 Q1。

---

## 六、给用户的问题

**Q1：码表要不要瘦身？**

- A：保持现状（码、类别、下一步三列，加登记测试）。好处：不动。代价：每个新码都要写一行没人读的文字；文字会和回执走样。
- B：删掉"类别"一列，登记测试不再检查类别。好处：去掉一列没有消费者的数据，测试少几行。代价：以后如果想给调用者输出类别，还得重建。
- C：码表缩成"码 + 一句话用途"，"下一步"统一以回执消息为准；登记测试照旧做双向比对。好处：只剩一份"下一步"的写法，不会再走样。代价：码表 150 行要改写一遍；与第六轮第 24 条"登记下一步"的要求相反，需要用户推翻自己的旧决定。
- D：删掉码表和登记测试，只在 `AGENTS.md` 保留"新码必须对应不同的下一步"这条规矩。好处：维护最轻。代价：失去"新码必须登记"的机械检查；码可能悄悄增多（对调用者影响仍然很小）。
- **推荐 B**：代价最小，去掉的是确定没有消费者的东西。C 的收益真实但改动面大，可以等下次真有码表文字走样再做。

**Q2：借用粘贴要不要保留？**

- A：保留并冻结（现状）。好处：保住"中文插入到光标处"这一能力。代价：约 1,000 行产品和测试代码继续维护，105 次真实运行里 0 次使用。
- B：删掉借用粘贴，`type` 只留 `--via keys`。好处：一次删掉四块能力里的一整块，以及剪贴板借写和恢复的全部状态。代价：在不支持设置文字的控件里，失去非 ASCII 文字插入；契约、指引、测试都要跟着改。
- C：保留借写和 PASTE，降级恢复：不再细分 failed/unknown/pending，一律报"恢复未确认"，并固定提醒剪贴板可能留着输入。好处：符合第 21 项"降级"的说法，F1 也随之消失。代价：仍然要做一次改动，契约样本也要重导。
- **推荐 A，加 F1 那一句指引**：真实任务没有暴露问题，删或降级都要先付一轮改动成本；等出现真实需要或真实缺陷时，再按第 21 项走 B 或 C。

---

## 七、核对过、没有问题的项（汇总）

- 三条输入路线之间没有自动回退；实测 9 次 UNKNOWN 填写之后都没有重复输入。
- 残留提醒在"已核对的粘贴 + 恢复 failed/unknown"时出现，默认文本和 `--json` 一致，契约和渲染测试都覆盖了（主机端定向测试 229 passed）。
- 锁屏、熄屏在输入、触摸、截图、剪贴板、通知之间口径一致；swipe 的例外只在主屏。
- 触摸交集中心、窗口缺席、遮挡、按屏过滤窗口：都正确。
- 区域截图的坐标换算、取整和布局核对都正确；REF 截图不再误报 `arguments`。
- 通知只剩读取，key 精确匹配。
- 文件传输的清理顺序和退出码规则本轮未变。
- 码表 150 码与源码双向一致；9 行改动与构造处一致；登记测试的保护范围见 F2。
- 四块能力没有违反"只删不补"；实施方没有动用删或降级授权。
