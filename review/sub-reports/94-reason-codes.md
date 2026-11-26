# 专题核查 94：原因码（第七轮，对 `f401deb`）

"原因码"指回执 `error.code` 里那个机器可读的字符串（设备端 `Rejected(code, …)`、主机端 `Fault(code, …)` 及少数拼出的 `{"code": …}`）。"主体"指它旁边说明"是哪样东西"的 `subject` 字段。"调用者"指按 `skills/androidctl/*.md` 操作 CLI 的 LLM agent。下文 Kotlin 路径省略前缀 `android/app/src/main/java/com/rainng/androidctl/`，主机路径省略 `androidctl/src/androidctl/`；版本默认 `f401deb`。`S` 为评审 scratchpad 根目录，我的脚本和中间结果在 `S/agents11/94/`。全程只读。

## 摘要

1. **数目和第 24 条的落实都对**（主机端本地复现）：我用一个比登记测试更宽松的独立抽取器重数，`f401deb` 两端实际构造的码正好 150 个，和 `contract/reason-codes.json` 双向一致。5 组合并、3 处归位都做了；6 个旧码在源码、测试、指引、契约样本、docs 仓库里都没有残留。主机端和设备端按码分支的地方合并后仍然正确。
2. **码表质量总体可用**：我回到构造处核了约 70 个码，大多数"下一步"写得准确。发现两处小问题：`sessionChanged` 一处回执文字（`Sessions.kt:307` "take a new snapshot"）与码表相反，上一轮已经指出过，这次没改；`screenshot REF` 在目标边界为空或越界时报 `arguments`（`ImageRegion.kt:154-181`），违背了指引里"`arguments` 只表示与设备状态无关的写法错"的规则。两处都建议进下一轮，不挡合入。
3. **登记测试建议保留，做两处小修**。它能防住"加了码没登记""删了码表里还留着""换成认不出的动态写法"这三种情况。我实测了它漏掉的写法：Python 的 `Fault(code="x")`、`protocol.Fault("x")`、`Fault` 的子类，以及 Kotlin 的 `put("code", "x")`，这几种都会被静默放过，各补一两行就能堵上。注释里写的 `Rejected("x")` 会误报，但误报时测试会明确失败，不会静默。改成两端各定义一个枚举，要动 600 多处构造，不值得；"从契约样本生成"做不到，样本里只有少数几个码。
4. **方向判断：原因码的规模基本只跟维护和内部分支有关，跟调用者关系不大**。20 次真实运行共 322 条命令，非零退出 8 次；模型见到的码只有 6 个（`stale`、`routeUnavailable`、`notReady`、`actionUnavailable`，另有成功回执附带诊断里的 `scopeUnknown`、`rootUnavailable`）。指引点名的码 13 个，与真实出现的只重合 `stale` 一个。每一次的下一步都跟着回执里那句话走，没有一次依赖码本身。所以再合并一轮对调用者几乎没有收益。把码压成十来个"类别"、文本里只给类别，同样收益很小，代价却大。建议从现在起停止"合并码"式的整理，以后只按真实运行里看到的回执文字是否准确来改。
5. 第六轮列的"内部混着不同下一步的 21 个码"，对着 `f401deb` 重列后是 20 个：`mappingUnknown` 经归位后已经只剩一种下一步。值得进下一轮的按重要性是：`sessionChanged` 的文字、`screenshot REF` 误用 `arguments`、`tlsIdentity` 的时钟／过期两种情形改用已有的设备端码。其余 16 个可以永久不动。
6. **总体判断**：这一块不挡合入，没有发现需要推翻用户第 24 条的证据。我也要指出：评审方第六轮推动的这轮合并，并掉的 6 个码在 57 份真实运行里一次都没出现过，对调用者是零收益的整理。代价不大，但不宜再来一轮。

---

## 一、数：源码里实际构造的码与码表

### 1.1 独立重数（主机端本地复现）

做法：`S/agents11/94/extract.py` 不复用产品测试的规则，故意多收。它把下面几类位置上的全部 camelCase 字符串字面量都收进来：Kotlin 的 `Rejected(` 第一个参数（按括号深度跨行截取，一个参数里有多个字面量就全收）、`"code" to …` 和 `put("code", …)` 的值；Python 的 `Fault`／`TransportFailure` 第一个参数，任意调用的 `code=` 关键字参数，`{"code": …}` 字典，以及对 `code`／`error_code`／`reason_code` 的赋值。运行命令：

```
python3 -I S/agents11/94/extract.py S/r7/prod S/agents11/94/out-f401.json
```

结果：多收进来 154 个字面量。逐个看后，`code`、`state`、`done`、`failed` 4 个不是原因码，而是取字段的键名（`getString("code")`、`text("state")`）或退出码映射表的键（`protocol.py:68`）。去掉这 4 个后**正好 150 个，与码表双向一致**：表外无码，表内每个码都有构造处。动态构造共 17 处，我逐处看过，全部属于透传已有问题码（如 `Wire.kt:80` 的 `rejected?.code ?: "workerFailure"`、`identity.py:134`、`UiSnapshot.kt:118`）或序列化预算占位（`ClipboardProjection.kt:145`、`NotificationProjection.kt:263`），没有一处造出新码。

产品自带的登记测试也通过了：在 `S/r7/run` 下运行 `PYTHONDONTWRITEBYTECODE=1 S/r7/venv7/bin/python -m pytest -p no:cacheprovider -q androidctl/tests/test_reason_codes.py`，7 passed。

附带对比基线（主机端本地复现，同一脚本作用于 `git archive 7d5be70`）：基线约 144 个码。到候选，去掉了 30 个（其中大部分是基线里按参数名起的码，如 `url`、`key`、`path`、`selector`，已并入 `arguments`），新增了 36 个（多数来自新能力：后台屏任务、复选框、剪贴板借用、观察上限等）。所以经过三轮合并，码数仍比基线多 6 个。这本身不是问题，因为能力变多了，但可以说明"码数"这个指标本身没有太大意义。

### 1.2 5 组合并、3 处归位（源码推断，`git show 642799b`）

| 项 | 落实 | 依据 |
| --- | --- | --- |
| `snapshotPath`、`imagePath` → `fileParent` | 已做；成功结果里的同名字段 `data.snapshotPath`／`data.imagePath` 保留不变（这两个名字作为字段名在 `ui_client.py`、`ui_output.py`、指引里仍大量出现，属正常） | `ui_client.py:381,390` |
| `identityChanged` → `tlsIdentity` | 已做，并补了阶段 `tls` | `state.py:290` |
| `noPublishedSnapshot` → `noSnapshot` | 已做；消费者集合同步改 | `UiDelivery.kt:98,100`；`UiCondition.kt:356` |
| `internalError` → `internal` | 已做，保留 `uncertain: True` | `local_server.py:171` |
| `messageLimit` → `jsonLimit` | 已做，两端 | `transport.py:41`；`GatewayClient.kt:34` |
| 滚动"此方向无路线"统一 `routeUnavailable` | 已做：`awaitAction` 对 `scroll` 和 `scroll-until` 都报 `routeUnavailable`；旧的按命令选码的分支（`fb75afc` 的 `UiActions.kt:1009`）已删除 | `UiActions.kt:1020-1023`；`UiScrollUntil.kt:106` |
| 坐标输入"要求用户 0" → `userScope` | 已做 | `VisualInput.kt:389` |
| 后台屏截图四种失败 → `screenshotUnavailable`；`imageUnavailable` 只留给图像通道坏了 | 已做；`imageUnavailable` 的消息补了"close and reopen the virtual session" | `WorkDisplay.kt:131,138,141,145,148` |

**残留检查**（主机端本地复现）：对 `f401deb` 整棵树运行 `git grep` 查 `identityChanged`、`noPublishedSnapshot`、`internalError`、`messageLimit`，零命中。查 `snapshotPath`／`imagePath` 作为码的用法（形如 `code=…`、`Fault("…`），零命中。docs 仓库 `dfe5e45` 里也是零命中。契约样本 `contract/wire-contract.json` 本来就不含这 6 个码，实施方没有为改名重新导出样本，这样做是对的。

---

## 二、码表的质量

### 2.1 抽查范围

我把每个码的全部构造处抽出来（`S/agents11/94/sites.txt`，约 1000 行），逐条对照码表 `actions` 一栏。实际核过的码约 70 个：`actionLabel`、`actionName`、`actionProtocol`、`actionUnavailable`、`actionUnsupported`、`activityUnavailable`、`ambiguous`、`apkUnavailable`、`appNotLaunchable`、`arguments`、`backendConnection`、`busy`、`captureMismatch`、`captureUnavailable`、`certificateClock`、`certificateExpired`、`changed`、`clipboardUnavailable`、`connectionClosed`、`covered`、`deviceName`、`deviceRequired`、`displayMismatch`、`displayUnsupported`、`externalTasks`、`fileCleanup`、`fileParent`、`gatewayStopping`、`history`、`identityUnknown`、`imageUnavailable`、`inputFocus`、`inputUnavailable`、`internal`、`jobOwner`、`jsonLimit`、`knownDevice`、`mappingUnknown`、`noSnapshot`、`notReady`、`output`、`partialBaseline`、`point`、`protected`、`query`、`queueFull`、`range`、`region`、`replyUnavailable`、`revoked`、`rootUnavailable`、`scopeMismatch`、`scopeUnknown`、`screenshotUnavailable`、`sessionChanged`、`sessionReleased`、`sessionRequired`、`snapshotBudget`、`sourceUnknown`、`stale`、`stopping`、`taskOnOtherDisplay`、`timeout`、`tlsIdentity`、`uiCleanup`、`unauthorized`、`userScope`、`windowUnavailable`、`workerFailure`、`workerUnavailable`、`workspace`。

结论：绝大多数行与构造处的消息一致，能指导调用者。内部评审指出的 `actionUnsupported`、`scopeUnknown` 两行已经改好（`reason-codes.json:6,108`）。下面是我发现的问题。

### 2.2 发现

**F1　`sessionChanged` 的一处回执文字与码表相反（上一轮已指出，未改）。**
- 现象：`Sessions.kt:304-307` 的 `requireCurrent` 在当前会话已不是这次请求的租约时报 `sessionChanged`，消息是 "The session was closed or released; take a new snapshot"。码表第 112 行写的是"先看 status……只在用户意图允许时重开会话"。按消息去执行 `snapshot`，多半会被 `sessionRequired` 或 `sessionReleased` 挡住，然后才拿到正确指引。另外一种可能：同名会话被另一个调用方替换过，这时 `snapshot` 能成功，但调用者手里的引用已经全部作废。
- 触发：UI 命令执行过程中，会话被关闭、释放或替换（`UiBackend.kt:329,339,347`、`TargetManagement.kt:30,128` 调用 `requireCurrent`）。
- 真实任务里：57 份运行中没有出现过。
- 影响：调用者多发一条命令；在"被替换"的情形下，可能在别人的会话上继续操作，前提是用户允许多个调用方共用同名会话。
- 修正：把消息改成 "…; inspect status first"，与同码另两处（`Sessions.kt:138,207`）一致。一行改动加对应测试断言，维护代价约为零。
- 依据：源码推断。分类：确定缺陷（提示文字）／**进下一轮**，也可以放进合入前的整理提交。

**F2　`screenshot REF` 对目标边界的设备状态报 `arguments`。**
- 现象：`VisualInput.kt:153` → `ImageRegion.fromRef`（`ImageRegion.kt:154-165`）把节点的实时边界交给 `rectangle()` 校验（`ImageRegion.kt:167-181`）。节点边界为空（右不大于左）或左上为负时，报 `code=arguments, argument=region`，消息是 "Region right and bottom must exceed left and top"。可调用者根本没写 `--region`。这违反了 `reference.md:86`（"`arguments` identifies syntax or parameter errors independent of current device state"）和 docs `reference/ui.md:15` 的规则。`freshRegionTarget`（`VisualInput.kt:198-224`）在调用 `fromRef` 之前不检查可见性。
- 触发：对屏外、已收起或零尺寸的节点执行 `screenshot REF`。平台对屏外节点的边界是否裁成空矩形，我没有设备，**未知**；按 Android 的常见实现推测，不少屏外节点的边界会是空的。
- 真实任务里：20 份运行没有一次 `screenshot REF`（命令分布：click 95、snapshot 71、scroll 45、back 27、fill 24 等），所以没有发生。
- 影响：调用者会去改一个自己没写的参数，大概率改不对，最后放弃或改用整屏截图。不会造成误操作。
- 修正：在 `fromRef` 里对 `fresh` 的可见性或非空边界先做检查，失败时报 `notReady`，并附上 `UiAdvice.visibilityAdvice` 的提示（与 `AppInput.kt:142` 一致）。约 3 行加 1 个 JVM 测试。
- 依据：源码推断。分类：确定缺陷（码选错）／**进下一轮**。

**F3　码表"类别"一列有个别与做法不符，且这一列没有消费者。** `externalTasks` 的类别是 `adjust`，做法却是"Ask the user whether to return or discard"（`reason-codes.json:48`）。`jsonLimit` 的类别是 `adjust`，但它的结果未知那一半应属 `verify`。混合码只能填一个类别，这一列对混合码本来就不准。除登记测试检查取值合法之外，没有任何代码、指引或输出读这一列。分类：可选建议／不必处理。如果以后要用这一列（见第四节），再统一校一遍。

**F4　`tlsIdentity` 多了一种情形，码表没写。** `transport.py:284-286` 在欢迎消息里的设备编号或名字与记录不符时，报 `tlsIdentity`，消息是 "Welcome names a different device"，阶段是 `authentication`。下一步应是核对连接地址是否指向了别的设备，而不是导入证书。码表第 134 行只写了"证书变了"和"时钟／过期"两种情形。触发很少（证书已经匹配、但设备身份不符）。分类：设计取舍／并入第五节 `tlsIdentity` 一起处理，进下一轮。

### 2.3 `busy` 的两句（源码推断）

码表第 15 行写的是 "For subject=automation, takeover has no effect: wait for the other automation tool to exit and retry the unsent command. For other holders, respect the current owner and take over only on user instruction; if the message identifies a running file step, query that original job." 这与构造处一致：`AutomationCleanup.kt:37-40` 带 `subject = "automation"`，消息是 "takeover has no effect; wait for it to exit"；`FileJob.kt:150` 是 "query its ID"。也与指引一致（`SKILL.md:22-24`）。文本回执会输出 `subject=`（真实运行里见过 `code=stale; subject=ref` 的格式），调用者看得到主体。**已核实无问题。**

### 2.4 下一步完全相同却没合并的码

按用户维持的判法（下一条命令相同；如果要停下请用户处理，用户处理完之后的命令也要相同），码表本身就暴露出几对文字几乎一样的码：`nodeLimit` 与 `treeByteLimit`（第 74 行与第 137 行逐字相同），`offset`／`size`／`hash`／`chunk`，`shizukuUnavailable`／`shizukuPermission`。它们正是候选第 8、10、12 组，用户已明确裁决不做。这里只确认：码表如实反映了"下一步相同"，没有因为不合并而写出虚构的区别。没有发现第 7 到 12 组之外、新的完全相同的码对。

合并之后有没有丢掉调用者需要的区分：`jsonLimit` 在合并前后都同时含"请求过大，未发出"和"回执过大，结果未知"两种情形（合并前的 `messageLimit` 也是这样），调用者靠主结果词（`not sent`／`UNKNOWN`）区分，合并没有让它更混。`tlsIdentity` 并入"配对时证书变了"之后，下一步与原有的"证书变了"相同。`noSnapshot` 合并后，`UiCondition.kt:356` 的捕获集合从 `noPublishedSnapshot` 换成 `noSnapshot`。我核了 `publishedBaseline` 和 `published` 两条路径（`UiDelivery.kt:91-105`、`UiCondition.kt:115-126`），里面不会抛出 `UiSnapshot` 的那个 `noSnapshot`，捕获范围没有被意外放宽。**已核实无问题。**

---

## 三、登记测试

### 3.1 怎么实现的（源码推断，`androidctl/tests/test_reason_codes.py`）

- 第 10-30 行是两张"已知透传"白名单，按文件名列出允许的非字面量表达式，例如 `transport.py: {"code"}`、`WorkerConnection.kt: {"error.code", …}`。
- 第 33-104 行处理 Python：用 `ast` 遍历产品源码，收集 `Fault`／`TransportFailure` 第一个位置参数、`Problem`／`decode_chunk` 的 `code=`、字典里的 `"code"` 键；只在 `transport.py` 里额外检查对 `code` 的赋值。非字面量必须在白名单里，否则测试失败并要求人工复核。
- 第 106-157 行处理 Kotlin：正则匹配 `Rejected(` 第一个参数和 `"code" to …`，对少数已知的条件、回退表达式做完整匹配，对 `NetworkGateway.kt` 的 `when` 分支逐行解析。认不出的写法必须在白名单里。
- 第 160-188 行是双向集合比较，并检查每行类别合法、做法非空。
- 第 191-245 行是测试自身的负向用例：2 个漂移用例，4 个"未知动态写法要求复核"的用例。

### 3.2 能防住什么、防不住什么（主机端本地复现）

我把两端源码复制到 `S/agents11/94/probe/tree`，逐一加入一种写法，再调用测试里的 `verify_reason_code_registration()`（脚本 `S/agents11/94/probe.py`，用 `S/r7/venv7/bin/python -I` 运行，不导入产品模块）：

| 写法 | 结果 |
| --- | --- |
| Python `PROBE = "x"; raise Fault(PROBE, …)` | 检出（要求复核） |
| Python `raise Fault(code="probeKw", message="x")` | **静默放过** |
| Python `raise protocol.Fault("probeAttr", "x")` | **静默放过** |
| Python `class LocalFault(Fault)` 后 `raise LocalFault("probeSub", …)` | **静默放过** |
| Kotlin `JSONObject().put("code", "probePut")` | **静默放过** |
| Kotlin `Rejected(code = "probeNamed", message = "x")` | 检出（要求复核） |
| Kotlin `const val P = "x"; Rejected(P, …)` | 检出 |
| Kotlin `val c = when {…}; Rejected(c, …)` | 检出 |
| Kotlin 注释里的 `// throw Rejected("probeComment", …)` | **误报**（要求登记一个并不存在的码，但失败是明确的） |
| Kotlin `json("state" to "probeState")` | 放过 |

最后一行要单独说明。`TreeCollector.kt:175` 用 `failed?.entry?.text("state") ?: "rootUnavailable"` 把窗口根的状态词直接当码发出去，状态词来自 `TreeCollector.kt:254-262` 的 `when`（`rootUnavailable`、`sourceUnknown`、`protected`、`notRead`）。测试只从回退值里认出 `rootUnavailable`。另外两个码之所以通过，是因为它们恰好在别处也有构造，是巧合。以后如果有人在那个 `when` 里加一个新状态词，它就会变成调用者可见、但没登记的码（真实运行里就见过这条路径发出的 `rootUnavailable`：`r6-s1-core-candidate-04` 的 `1795622670890299400…/child.stdout`）。实施方在说明里承认了这条边界。

依赖字符串模式的脆弱点：Kotlin 一侧完全靠正则。`Rejected(` 前面排除了 `class `，`"code" to` 要求后面跟逗号才算完整，有限表达式靠 4 条完整匹配的模式。代码格式化（ktlint 改换行）或重构表达式，都可能让测试报"要求复核"。这种失败是明确的，不会静默，修起来要把新写法加进白名单或模式。这是可以接受的维护负担。

### 3.3 判断：保留，做两处小修

- **相称性**：测试在 2 秒内跑完，不需要额外依赖，不碰运行时代码；真正的抽取逻辑约 150 行，自测约 55 行。码表 152 行、24 KB，除测试外没有任何东西读它，它是给维护者和评审看的底稿。对一个人维护的 alpha 产品，这个规模偏大但可以接受；用户第 24 条明确要求了码表和登记测试，我没有找到推翻它的证据。
- **小修 1**（建议，约 5 行）：Python 一侧把 `Fault`／`TransportFailure` 的 `code=` 关键字参数，以及 `ast.Attribute` 形式的调用（`protocol.Fault`）也收进来；Kotlin 一侧把正则扩成 `"code"\s+to|put\("code",`。这样就堵住了上表四种静默放过的写法。子类问题现在不存在，不必处理。
- **小修 2**（可选）：第 191-245 行的自测是内部评审补出来的"测试的测试"。如果维护者想减少代码量，可以删掉 4 个动态写法用例，只留 2 个漂移用例。不删也无害。
- **不建议换做法**。两端各定义一个枚举（Kotlin `object ReasonCode`、Python `Enum`），要改约 600 处构造，还需要另一个跨语言测试来核对两边一致；而且"下一步"一栏仍要单独维护。从契约样本生成也行不通：样本里只有少数几个码。第六轮工作记录 82 的"做法二"（只在 `AGENTS.md` 写一句规矩、不加测试）更轻，但用户已经选了码表加测试。
- 分类：可选建议／进下一轮（小修 1），不必处理（小修 2）。

---

## 四、调用者实际怎么用原因码

### 4.1 真实运行（实测，原件）

脚本：`S/agents11/94/runs.py` 统计每次调用的 `evidence/cli/*/child.stdout|stderr`（结果在 `runs.json`）；另对 `native.rollout.jsonl` 里模型实际收到的全部工具返回重新统计，两种统计结果一致。20 份运行共 322 次 CLI 调用，其中非零退出 8 次。

| 运行（`ultragoal/2026-11-11-next-improve/stages/42-r4-decisions/efficiency/…`） | 命令 | 退出码 | 码 | 回执里给的做法 | 模型的下一条命令 | 与码表一致吗 |
| --- | --- | --- | --- | --- | --- | --- |
| `r6-c1-candidate-01/evidence/cli/1795628261452159100…` | `click e608125688` | 2 | `stale` `subject=ref` | 无 | `snapshot`，然后点新引用 | 一致 |
| `r6-c1-candidate-04/…/1795665096342835300…` | `click e404859888` | 2 | `stale` `subject=ref` | 无 | `snapshot`，然后点新引用 | 一致 |
| `r6-c1-candidate-03/…/1795629673603476900…` | `scroll … down` | 2 | `routeUnavailable` | "use androidctl scroll … backward / up" | `snapshot`、`snapshot --screenshot`，之后才 `scroll … up` | 基本一致（先观察一次，属安全动作） |
| `r6-c1-candidate-04/…/1795665233010191900…` | `scroll … up` | 2 | `routeUnavailable` | 同上 | `snapshot` | 基本一致 |
| `r6-c1-candidate-05/…/1795665696380035200…` | `scroll-until … down` | 1 | `stop=routeUnavailable` | 同上 | 换容器再 `scroll-until` | 一致 |
| `r6-c2-candidate-01/…/1795616710850257100…` | `click e950416952` | 2 | `notReady` | "reveal it with androidctl reveal …, then retry" | `reveal` 后再 `click` | 一致 |
| `r6-s1-core-candidate-04/…/1795622612375108200…` | `reveal e668236894` | 2 | `actionUnavailable` | "click it directly … or use snapshot to find a container to scroll" | 滚动容器 | 一致（选了第二种做法） |
| `r6-c1-candidate-05/…/1795665632062176200…` | `scroll … down` | 1 | （无码：已执行，之后的观察超时） | — | 再滚一次 | 不涉及 |

成功回执附带的诊断：`scopeUnknown` 出现在 6 次运行里（长列表题 3 次、后台屏题 3 次），`rootUnavailable` 出现在 1 次运行里。模型的下一条命令都是普通的 `snapshot`，全部成功。长列表题的回执写的是 "inspect a listed window: snapshot --window 14"，模型没照做，直接 `snapshot` 也成功了（界面变化还没结束时的短暂状态，过一会儿自己就消失了）。这说明对这种短暂状态，"再采一次"往往就够了，回执和码表第 108 行都可以加一句"变化中时先重新 `snapshot`"。分类：可选建议／进下一轮。

模型的推理内容是加密的（`reasoning.summary` 为空），所以看不到它是否"读了码"。但每一次的下一步都能由回执的那句话直接得出。没有一次是码本身提供了句子之外的信息。

### 4.2 指引里讲到的码（源码推断）

`skills/androidctl/*.md` 共 1308 行，作为码点名的有 13 个：`busy`、`connectionBusy`（`SKILL.md:23-24`），`arguments`、`stale`、`changed`（`reference.md:86-87`），`covered`、`unexpectedTaskMove`（`actions.md:39,181`），`stale` `subject=image`（`images.md:111`），`protocolMismatch`、`sessionRequired`（`connection.md:104,109`；`displays.md:21,93`），`uiCleanup`（作为诊断字段，`displays.md:105`），`sourceChanged`、`publishUnsupported`（`jobs-files.md:61,63`），`apkUnavailable`（`reference.md:36`）。

| | 个数 |
| --- | --- |
| 码表 | 150 |
| 指引点名 | 13 |
| 本轮 20 份运行中出现（含成功诊断） | 6 |
| 两者都有 | 1（`stale`） |
| 第六轮 37 份运行加本轮 20 份，共 57 份出现过的 | 7 个现行码（`notReady`、`stale`、`arguments`、`scopeUnknown`、`routeUnavailable`、`actionUnavailable`、`rootUnavailable`），另有旧码名 `staleRef` |

### 4.3 方向性判断

- **原因码体系的规模，主要跟维护和内部分支相称，跟调用者的使用不相称，但这不构成问题**。150 个码里，调用者实际碰到的不到 5%。原因码在文本回执里只占 `code=xxx` 约 20 个字符，非零退出又只占调用的 2.5%（8／322），对成本的影响可以忽略。真正决定调用者下一步的是回执里那句话，而那句话写得好不好，与码有多少个无关。
- **第六轮合并的实际收益约为零**。本轮并掉的 6 个码（`snapshotPath`、`imagePath`、`identityChanged`、`noPublishedSnapshot`、`internalError`、`messageLimit`）在 57 份运行里出现 0 次。3 处归位里只有 `routeUnavailable` 在真实运行里出现（本轮 3 次）。可是对比上一轮，那时同样的情形报 `notReady`，模型 6 次里有 4 次直接改用列出的方向；本轮报 `routeUnavailable`，模型 3 次里 2 次先 `snapshot`。没有证据表明换码改善了调用者的行为。这是评审方上一轮推动的整理，我认为它没有害处，但也几乎没有用处。**建议不要再做"合并码"一类的轮次**，以后的改动以"真实运行里某句回执把调用者带错"为证据来驱动。
- **"把码压成十来个下一步类别，文本只给类别和一句话，其余进 `--json`"：不推荐**。
  - 收益：调用者本来就只读那句话，类别对它是多余的信息；把 `code=` 挪进 `--json`，每次非零退出省约 20 个字符，可以忽略。
  - 代价：设备端和主机端有约 25 处按具体码分支（例如 `UiCondition.kt:356,487`、`UiRefRepair.kt:76-80`、`TargetManagement.kt:169`、`coordinator.py:298,430`、`file_jobs.py:417,542`、`app.py:938,959`），这些必须保留细码；指引里点名的 `busy subject=automation`、`stale subject=image` 也会失去稳定的锚点；契约、测试、指引都要大改。还有一点：码表已经有 10 个类别，混合码在类别上填不准（见 F3），说明"按类别"本身并不比"按码"更能表达下一步。
  - 如果一定想给调用者一个类别，代价最低的做法是不改码，只在回执里多输出一个由码表查出来的类别字段。但现在没有证据表明调用者需要它，我不建议做。
- 分类：设计取舍／不必处理（维持现状），并建议用户确认"停止合并类整理"。

---

## 五、"内部混着不同下一步"的码：对着 `f401deb` 重列

第六轮的 21 个里，`mappingUnknown` 的"要求用户 0"已归位到 `userScope`（`VisualInput.kt:389`）。剩下的 9 处构造（`VisualInput.kt:385,401`、`AccessibilityConnection.kt:70-100`）都是"坐标映射事实缺失，改用引用"，**已不再混合**。所以现在是 20 个。`notReady` 去掉了"无路线"这种情形，但仍然混合。

调用者现在靠什么区分：除注明外，都靠回执消息里写明的原因和做法。下表"真实出现"一栏统计的是 57 份运行。

| # | 码 | 混着的下一步 | 现在靠什么区分 | 真实出现 | 建议 |
| --- | --- | --- | --- | --- | --- |
| 1 | `sessionChanged` | 预检期间变了，先看 status（`Sessions.kt:138,207`）／管理界面刷新（`Sessions.kt:269-270`、`WorkerService.kt:105`）／请求执行中会话已被关闭或替换（`Sessions.kt:307`） | 消息；但第三种情形的消息写错了（F1） | 0 | **进下一轮**：只改 `Sessions.kt:307` 的文字，不拆码 |
| 2 | `region`（以及 `arguments`） | 坐标区域越界，改参数／引用目标边界超出屏幕，先让它露出来或改用整屏截图；引用目标边界为空时报的是 `arguments`（F2） | 消息；`arguments` 那种情形连消息都对不上 | 0 | **进下一轮**：引用路径先查可见性，报 `notReady` |
| 3 | `tlsIdentity` | 证书变了，核实后 `device trust`／时钟不对（`transport.py:94-95`）／证书过期，在设备上续签（`transport.py:96-100`）／欢迎消息里的设备不符，核对地址（`transport.py:284-286`） | 消息后缀和阶段（`tls`／`authentication`） | 0 | 进下一轮（低）：主机端对时钟、过期两种情形复用设备端已有的 `certificateClock`／`certificateExpired`（`DeviceIdentity.kt:49,51`），码数不变；码表补上第四种情形 |
| 4 | `busy` | 别的会话持有租约，接管须用户指示（`Sessions.kt:410`）／`subject=automation`，接管无效、等对方退出／本会话资源未结束，等待后重试（`Sessions.kt:228,296,323`、`coordinator.py:281,411,620,655,696`、`local_server.py:141`、`file_jobs.py:588`、`JobTable.kt:46`）／文件步骤还在跑，查任务（`FileJob.kt:150`） | 自动化那种靠主体；其余靠消息 | 0 | 用户已裁决复用；可选：给 `FileJob.kt:150` 加主体 `job`。不拆 |
| 5 | `timeout` | 发出前预算用完，原样重试／发出后用完，结果未知，先核对／连接未完成，查网络（`coordinator.py:144`）／超时取值越界（`NetworkGateway.kt:297`，实为参数错误） | 主结果词（`not sent`／`UNKNOWN`）加阶段 | 0 | 不拆；可选：把 `NetworkGateway.kt:297` 改报 `arguments` |
| 6 | `notReady` | 不可见，`reveal`／被禁用，等待或换目标／没有输入焦点，先聚焦／边界为零／选择器无匹配或观察不完整（`UiActions.kt:948`） | 消息里带可照抄的命令 | 本轮 1 次，上轮 7 次（其中 6 次已归位） | 永久不动 |
| 7 | `inputUnavailable` | 屏幕关闭，`wake`／锁屏，请用户解锁（`DeviceFacts.kt:72,79`）／输入服务不可用，报告（`AppInput.kt:65,67`）／后台屏未点亮（`UiBackend.kt:87`） | 消息 | 0 | 永久不动 |
| 8 | `unauthorized` | 配对窗口被拒，要新配对码／已存凭据被拒，查配对（`transport.py:111-116,267`）／网关握手时既无配对也无窗口（`NetworkGateway.kt:72`） | 消息；主机按（`unauthorized`、`device.authentication`）分支（`coordinator.py:430`） | 0 | 永久不动（拆开会牵动主机端的回退逻辑） |
| 9 | `deviceRequired` | 没有配对设备，去配对／有多台、没选，用 `--device` 选（`state.py:187,193`） | 消息 | 0 | 永久不动（后一种可以归到 `deviceNotFound`，收益很小） |
| 10 | `actionUnavailable` | 换动作编号／不支持 `reveal`，直接 `click`／不支持 PASTE 或 SET_SELECTION，换输入方式 | 消息里带命令 | 本轮 1 次（模型照做） | 永久不动 |
| 11 | `sourceUnknown` | 重新截图或 `snapshot`／App 按键需要输入焦点窗口（`SystemActions.kt:178`） | 消息 | 0 | 永久不动 |
| 12 | `point` | 坐标在图像外，改参数／引用中心在屏外，滚进屏内（`PointerPlan.kt:52`、`AppInput.kt:155`） | 消息 | 0 | 永久不动 |
| 13 | `range` | 取值越界，改参数／最小或最大值读不到，换动作（`NodeRange.kt:45-50`） | 消息 | 0 | 永久不动 |
| 14 | `clipboardUnavailable` | 借用粘贴时，换输入方式（`BorrowedPaste.kt:211`）／剪贴板命令时服务不可用，报告（`ClipboardAccess.kt:205`） | 命令上下文和消息 | 0 | 永久不动 |
| 15 | `displayUnsupported` | 平台不支持后台屏，改用主屏／该命令只支持主屏（`SystemActions.kt:41`） | 消息；两者实际都是"用主屏" | 0 | 永久不动（实际上接近不混合） |
| 16 | `history` | 缓存目录有问题，需要用户处理／历史忙，重试／预算用完，重试（`history.py:70-88`） | 消息 | 0 | 永久不动 |
| 17 | `output` | 输出完整性问题，报告／取输出时间不够，再查一次（`job_commands.py:157-190`） | 消息 | 0 | 永久不动 |
| 18 | `workspace` | 工作区目录不存在／文件读不了（`workspace.py:22-65`） | 消息；两者都是"修工作区" | 0 | 永久不动 |
| 19 | `fileCleanup` | 清理未确认／目标身份变了（`user_files.py:45,50`） | 消息；属诊断 | 0 | 永久不动 |
| 20 | `uiCleanup` | 挡住新工作（`Sessions.kt:265`）／仅作诊断字段（`AutomationCleanup.kt:77`） | 出现的位置不同：`error` 还是 `cleanupError` | 0 | 永久不动 |

**按重要性排序、值得进下一轮的短清单**：1）`sessionChanged` 的一句文字（F1）；2）`screenshot REF` 误报 `arguments`（F2）；3）`tlsIdentity` 的时钟／过期两种情形复用已有的设备端码，码表补上第四种情形（F4）。第 4、5 行（`busy` 加主体 `job`、`NetworkGateway.kt:297` 改报 `arguments`）属可选。其余 15 行可以永久不动：消息都写明了做法，真实运行里出现过的两个（`notReady`、`actionUnavailable`）模型都照着做对了。

---

## 六、主机端（及设备端）按码分支的地方在合并后是否正确（源码推断）

主机端（`grep` 出的全部按码比较）：

- `app.py:938,959`（`daemonStopped`）、`coordinator.py:298`（`deviceNotFound`）、`coordinator.py:430`（`unauthorized` + `device.authentication`）、`file_jobs.py:417`（`publishUnsupported`）、`file_jobs.py:542`（`temporaryCleanup`）、`private_files.py:227`（`privatePermissions`）、`transport.py:270`（`protocolMismatch`）、`ui_output.py:288`（`unexpectedTaskMove`）：都不涉及本轮合并或归位的码，行为不变。
- `identity.py:134` 透传码并改阶段为 `tls`，与 `tlsIdentity` 合并无关。
- 在 `fb75afc` 上查过：主机端原本就没有按 `internalError`、`messageLimit`、`identityChanged`、`snapshotPath`、`imagePath` 分支的代码，所以合并不会漏改消费者。

设备端：`UiCondition.kt:356` 已同步为 `noSnapshot`（见 2.4）。`UiCondition.kt:487`、`TargetManagement.kt:169`、`ClipboardOperations.kt:87`、`UiRefRepair.kt:76-80`、`UiRun.kt:128-129`、`FileJob.kt:249`、`WorkerConnection.kt:156,165` 都不涉及被改的码。没有任何代码按 `imageUnavailable`、`screenshotUnavailable`、`mappingUnknown`、`notReady`（滚动情形）分支，所以归位没有消费者需要跟着改。**已核实无问题。**

---

## 七、已核实无问题的点

1. 150 个码与码表双向一致（独立、更宽松的抽取器复现）；17 处动态构造都是透传或占位，没有造出新码。
2. 5 组合并、3 处归位全部落实；6 个旧码在产品仓库（源码、测试、指引、契约样本）和 docs 仓库里都没有残留；成功字段 `snapshotPath`／`imagePath` 保留不变。
3. `busy` 的两句与构造处、指引一致，主体在文本输出里可见。
4. 合并没有让 `jsonLimit`、`tlsIdentity`、`noSnapshot` 更混，也没有放宽 `UiCondition.kt:356` 的捕获范围。
5. 两端按码分支的地方合并后都正确。
6. 内部评审指出的 `actionUnsupported`、`scopeUnknown` 两行已经改准；登记测试对 Kotlin 跨行拼接、条件拼接、回退后拼接和 Python 未知赋值，都会明确失败。
7. `AGENTS.md` 的规矩一句和 `contract/README.md` 的说明，与用户裁决的措辞相符，并如实说明测试判断不了"下一步是否真的不同"。
8. 没有为改名而重新导出契约样本，这是对的：样本本来就不含被改的码。

## 八、其它小项

- docs `development.md:89` 写着"旧原因码按本批合并表迁移"，但产品仓库和 docs 仓库里都没有这张"旧码 → 新码"的迁移表（旧码名在两仓零命中）。发布说明要列破坏性变更时，需要这份对照：本轮 6 个，加上前几轮从基线去掉的约 30 个，见 1.1 的基线对比。分类：可选建议／进下一轮（发布前补）。
- 调用者可见的还有两套平行的词汇，不在码表和规矩的范围内：`scroll-until` 的 `stop=`（`found`、`notFound`、`maxSteps`、`budgetExhausted`、`platformPending` 等，`ScrollProgress.kt:56-165`；其中 `stop=found` 在本轮运行里出现 14 次），以及观察缺口原因（`nodeUnavailable`、`collectionFailed` 等）。`ScrollProgress.kt:146` 会把错误码当作 `stop=` 的值。这是实施方有意划定的边界，我认为合理；只是指出"新增原因码须对应不同下一步"这条规矩管不到它们。分类：设计取舍／不必处理。

## 九、未能核实的事项

- F2 在设备上是否真会发生：取决于平台对屏外节点的边界是否裁成空矩形，以及 Chrome、WebView 节点的边界行为。没有设备，未知。
- 后台屏截图四种失败（`screenshotUnavailable`）和 `imageUnavailable` 的设备行为：没有设备证据，实施方也说明没有。
- 模型是否"读了码"：推理内容加密，只能从下一条命令推断，无法直接核实。
- `magnificationActive` 与 `magnificationOverlay` 是否下一步相同：仍需设备才能判断（第六轮已列为需设备）。
- 我没有运行 Android 端测试或 Gradle；Kotlin 一侧的判断都来自源码。
