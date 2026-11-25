# 原因码再合并一轮：候选清单（对 `fb75afc`）

工作记录 82。对应用户第六轮第 24 条裁决："原因码再合并一轮：全部重新审一遍，包括上一轮定为不合并的。标准是'调用者的下一步动作完全相同才合并'。同时立规矩：新增原因码必须对应不同的下一步动作。清单可能很短甚至为空，如实报告。"

源码一律指 `fb75afc`。Kotlin 文件省略前缀 `android/app/src/main/java/com/rainng/androidctl/`，主机文件省略前缀 `androidctl/src/androidctl/`。工作目录 `S/agents10/82/`（`S` 是评审 scratchpad 根目录）里有：`extract.py`（抽取构造点）、`rows.json`（609 处构造的原始记录）、`table.py`／`table.json`（逐码汇总与使用处）、`runs.py`／`runs2.py`／`runs.json`（真实运行统计）、`classify.py`（逐码人工归类）、`codes-full.md`（全表）。全程只读，没有改动产品仓库，也没有执行材料包里的任何脚本。

"原因码"在本文指回执 `error.code` 里那个机器可读的字符串（设备端 `Rejected(code, …)`、主机端 `Fault(code, …)`，以及少数在别处拼出的 `{"code": …}`）。"调用者"指照 `skills/androidctl/*.md` 操作这个命令行工具的 LLM agent。

---

## 一、总表摘要

- **现在有 156 个原因码**【重算】：设备端 114 个、主机端 62 个，两端都用的 20 个。其中由 `Rejected("…")`／`Fault("…")`／`TransportFailure("…")` 直接以字面量构造的 145 个（设备 106 个、377 处；主机 57 个、220 处），其余 11 个只在别处或经变量、条件表达式构造。与工作记录 70 的"144／153"逐个对上了，差异见 §二。
- **按调用者的下一步动作分成 10 类**（定义见 §二）：已满足无需动作 1 个；改参数重发 17 个；重新观察后重建命令 13 个；换动作或换通道 17 个；稍后原样重试 15 个；先做调用者自己能做的准备步骤 20 个；停下请用户处理 26 个；结果未知、先只读核对 3 个；无法自行恢复、只能报告 27 个；不是给调用者的（只在设备管理界面、内部、诊断字段）17 个。另有 21 个码内部混着下一步动作不同的情形。
- **建议合并（高把握）5 组，并掉 6 个码**：`snapshotPath`、`imagePath` 并入 `fileParent`；`identityChanged` 并入 `tlsIdentity`；`noPublishedSnapshot` 并入 `noSnapshot`；`internalError` 并入 `internal`；`messageLimit` 并入 `jsonLimit`。另有 3 处"同一情形用了两个码"的归位，不减个数（滚动"此方向无路"统一用 `routeUnavailable`；`mappingUnknown` 的"须 user 0"归 `userScope`；后台屏截图失败归 `screenshotUnavailable`）。
- **建议合并（中把握）1 组，减 6 个**：依赖当前状态、但下一步就是改参数重发的 `actionLabel`、`actionName`、`duration`，以及 `point`、`region`、`range` 的"取值越界"用法并入 `arguments`；`point`、`region` 的"引用的目标不在屏内"用法并入 `notReady`，`range` 的"最小最大值读不到"并入 `rangeUnavailable`。高加中合计减 12 个，**156 个变成 144 个**。
- **中低、低把握的候选另有 6 组，最多再减 14 个**（全做是 130 个），我倾向只做其中的"Shizuku 未就绪"一组或者都不做，各组理由见 §三。
- **上一轮不合并的 6 个**：`identityChanged` 改判为合并（并入 `tlsIdentity`，不是并入界面的 `changed`，"安全信号"的含义保留）；`sourceChanged`、`temporaryChanged` 可以互相合并（中低），仍不并入 `changed`；`sessionChanged`、`workerLockChanged`、`staleWorker` 仍不合并（前一个本身是混合码，后两个一个恢复路径核不到、一个只在设备管理界面出现）。上一轮"部分符合"的 4 个：`point`、`region` 剩下的用法这次可以全部归位；`deviceName`、`scriptFile` 并入 `arguments` 只有中低把握。
- **最重要的不确定点**：标准里"下一步动作"取多粗。我取的是"拿到 A 和拿到 B 后发的下一条命令相同（参数由回执给出）"；对"停下来告诉用户"一类，要求用户处理之后调用者的后续命令也相同。如果按字面最粗的读法，"停下请用户处理"的 26 个和"只能报告"的 27 个几乎都可以各并成一个码。我不建议这样做，理由在 §三丙；这一点需要用户定。
- 真实运行里调用者真正看到的码很少：37 份运行的全部 CLI 原件里，文本 `code=` 只出现 15 次（`notReady` 7、旧名 `staleRef` 4、`stale` 3、`arguments` 1），另有 `--json` 回执里 `scopeUnknown` 8 次（都在成功回执的附带诊断里）、`routeUnavailable` 1 次。`notReady` 的 7 次里有 6 次是"这个方向没有滚动路线"，这正是建议归位的那一处。

---

## 二、方法、计数与对账

### 2.1 抽取方法

`extract.py` 遍历 `SRC` 工作树（HEAD 为 `fb75afc`，`git status` 干净）下全部 `.kt` 与 `.py`：

1. 找出每一处 `Rejected(`、`Fault(`、`TransportFailure(`（排除类定义），按括号配对取完整参数表，第一个参数是字符串字面量就记为码，否则记下表达式；同时取 `argument=`、`subject=`、`uncertain=`、主机端的 `stage`、所在函数名。共 609 处，其中 597 处是字面量，12 处不是。
2. 另找 `"code" to …`（Kotlin）、`"code": "…"` 和 `code="…"`（Python）三种"在别处构造"的写法，共 24 处。
3. 人工逐个解开那 12 处非字面量和别处构造里的变量（见下）。

**12 处非字面量的解法**【源码】：
- `TransferHandle.kt:159`：条件表达式，产出 `publishUnsupported` 或 `publishFailed`。
- `UiActions.kt:913`、`UiCondition.kt:430`：`limits.first()`，取值只可能是 `nodeLimit`、`treeByteLimit`（`UiObservation.kt:63`）。
- `UiActions.kt:1010`：变量 `code`，`UiActions.kt:1009` 定为 `scroll-until` 时 `routeUnavailable`、否则 `notReady`。
- `UiSnapshot.kt:118`：观察问题里的码透传，缺省 `noSnapshot`。
- `ui_client.py:284`：透传设备的码，缺省 `captureUnavailable`。
- `WorkerConnection.kt:344`、`identity.py:134`、`protocol.py:132,139`、`transport.py:117,273`：透传或由调用方传入（`decode_chunk(code=…)` 的取值是 `chunk`、`output`、`captureMismatch`；`transport.py:81-111` 的变量是 `tlsIdentity`、`connectionFailed`、`unauthorized`；`transport.py:267` 透传设备握手拒绝里的 `protocolMismatch`／`unauthorized`）。

**别处构造的码**：`actionProtocol`（`ActionReply.kt:59`）、`uiCleanup`（`AutomationCleanup.kt:62`）、`checkPending`（`ScrollProgress.kt:106`）、`windowUnavailable`、`nodeLimit`、`collectionLimit`、`rootUnavailable`（`TreeCollector.kt`）、`treeByteLimit`（`UiTreeBudget.kt:24`）、`workerFailure`（`Wire.kt:80`，所有非 `Rejected` 异常的默认码）、`protocolMismatch`／`unauthorized`（`NetworkGateway.kt:71-72`）、`internal`（`file_jobs.py:554`）、`internalError`（`local_server.py:171`）、`query`（`job_commands.py:143`）、`localConfiguration`（`state.py:263`）。

**可能漏掉的构造方式与人工补查**：
- 通过辅助函数传码：只找到 `decode_chunk(code=…)` 一个（主机），已解开。Kotlin 没有接收 `code: String` 再构造 `Rejected` 的辅助函数（查了 `fun …(… code: String`）。
- 字符串拼接生成码：没有找到。
- 透传：设备把常驻进程回执里的码原样带回（`decodeWorkerReply`），主机把设备码原样包进 `transfer_fault`（`file_jobs.py:182-186`），都不产生新码。
- 观察数据里的"缺口原因"（`nodeUnavailable`、`collectionFailed`、`repeatedSource`、`jsonBudget`）不是回执错误码，没有计入；但 `nodeLimit`、`treeByteLimit`、`collectionLimit` 既是缺口原因又会成为错误码，计入了。
- `TreeCollector.kt:169` 在没有问题对象时把根的状态字（如 `failed`）当码用，按代码路径这里几乎走不到，没有计入；这是一个可能的漏数点。
- 设备管理界面里给人看的码（例如强制退出执行器）也是同一个 `Rejected`，计入了，但在表里标为"只在设备管理界面"。

### 2.2 计数

| 口径 | 设备端 | 主机端 | 去重合计 |
| --- | --- | --- | --- |
| 字面量直接构造 | 106 个（377 处） | 57 个（220 处） | 145 个 |
| 加上别处、动态构造 | 114 个 | 62 个 | **156 个**（两端共用 20 个） |

通用的 `arguments` 码有 216 处构造（设备 122、主机 94），`argument` 字段取值 23 种。

### 2.3 与工作记录 70 的 144／153 对账

工作记录 70 的脚本（`S/agents9/70/codes.py`）只匹配 `Rejected("`、`Fault("` 两种字面量，再用正则补"别处构造"。逐个对比结果：

1. 字面量部分 144 对 145：差的是 `replyUnavailable`。它用 `TransportFailure(…)`（`Fault` 的子类，`transport.py:438`）构造，70 的正则不认这个类名。
2. 别处构造部分 70 记 9 个，其中 `message` 是误报：它来自 `BorrowedPaste.kt:382` 和 `ClipboardProjection.kt:163` 的 `listOf("code", "message")`，不是码。所以 70 的真实结果是 144＋8＝152。
3. 我另外补出 4 个动态构造的码：`replyUnavailable`（上一条）、`connectionFailed`（`transport.py:81-85` 的条件表达式）、`publishFailed`（`TransferHandle.kt:159` 的条件表达式）、`workerFailure`（`Wire.kt:80` 的默认值）。152＋4＝156。

两次计数在 70 识别到的 152 个码上完全一致，没有 70 有而我没有的码。

### 2.4 "下一步动作"的分类定义

调用者是 LLM agent。每个码按"它拿到这个码之后，最合理的下一条命令是什么"归类。依据依次是：回执文字里写明的做法；`skills/androidctl/*.md` 里的说法；主机和设备里按码分支的代码；主结果词和退出码；真实运行里模型的实际反应。

| 类别 | 定义 | 个数 |
| --- | --- | --- |
| N0 已满足 | 要求的状态已经成立，什么都不用做 | 1 |
| N1 改参数重发 | 按回执改写参数后重发同一条命令，不需要新的观察 | 17 |
| N2 重新观察后重建 | 先 `snapshot`（或 `screenshot`，必要时缩小范围），用新观察里的引用、图像重新组织命令 | 13 |
| N3 换动作或通道 | 目标现在不支持这样做，改用另一种命令（改点行、换列出的方向、换输入方式、改用主屏） | 17 |
| N4 稍后原样重试 | 瞬时占用或时间不够，原样或加大 `--timeout` 重发 | 15 |
| N5 先做准备步骤 | 调用者自己能做的前置命令：`session open`、`pair`、`device select`、停旧守护进程、建目录 | 20 |
| N6 停下请用户处理 | 需要人在设备或主机上操作（启动 Shizuku、授权、解锁、换用户、导入受信证书、修权限） | 26 |
| N7 结果未知 | 动作可能已经发生，不能自动重发，先只读核对 | 3 |
| N8 只能报告 | 实现缺陷、协议不一致、平台不可用；停下报告，最多重试一次 | 27 |
| N9 不是给调用者的 | 只在设备管理界面出现（11 个）、内部捕获转换（5 个），或只作诊断字段（7 个，部分与上面重叠计） | 17 |

（受众统计：调用者可见 133 个，只在设备管理界面 11 个，内部 5 个，主要作诊断字段 7 个。设备管理界面和内部的判断有一部分是读调用链推断的，表里标了"推断"。）

**"完全相同"的判法**（我采用的）：拿到 A 和拿到 B 之后，调用者发的下一条命令是同一个子命令，参数由回执本身（`argument`、`subject` 或消息）给出；如果下一步是"告诉用户"，还要求用户处理完之后调用者的后续命令相同；至于转告给用户的话不同，只要消息里写清楚，不算不同。

---

## 三、建议合并的清单（供逐组批准）

按把握从高到低排。"处"指源码里要改的位置；测试和契约样本的数是按"断言了这个码"的行粗数，未逐行确认。

### 甲、高把握

#### 第 1 组：输出路径的父目录不存在 —— `snapshotPath`、`imagePath` 并入 `fileParent`

- **组内各码**【源码】：
  - `snapshotPath`：`ui_client.py:376`，"Observation destination parent must be an existing directory"。
  - `imagePath`：`ui_client.py:385`，"Image destination parent must be an existing directory"。
  - `fileParent`：`job_commands.py:441`（`shell` 输出 `--out`）、`user_files.py:303`（Windows 打开父目录）。
- **为什么下一步完全相同**：前三处都走同一个函数 `require_new_file(path, exists_message, parent_error)`（`user_files.py:68-72`）。这个函数里"目标已存在"已经统一报 `fileExists`，只有"父目录不存在"由调用方各传一个码。三处都在发出请求之前检查，结果都是"未发出"。调用者拿到任何一个，下一步都是"建好这个目录或换一个存在的目录，然后原样重发"。
- **合并后**：叫 `fileParent`。可以带 `argument` 写明是哪个选项（`out`／`imageOut`），不带也不影响调用者，因为消息已经写了是哪种目标。
- **连带改动**：构造 2 处；测试 1 处（`test_ui_arguments.py:564` 断言 `imagePath`）；指引没有点名这两个码（`SKILL.md:87`、`images.md:15` 里的 `imagePath` 是结果字段名，不是码，不受影响）；契约样本 0 处；没有代码按这两个码分支。约 3 处。
- **反方意见**：码名和结果字段同名（`snapshotPath`、`imagePath` 同时是成功回执里的字段），合并后反而消除了一处同名歧义。我想不出调用者对三者反应不同的情形。
- **建议**：合并，高把握。上一轮把这两个码列为"依赖当前状态、不并入 `arguments`"，这个判断仍然对；这次是并入 `fileParent`，不是并入 `arguments`。

#### 第 2 组：设备出示的证书与已存不同 —— `identityChanged` 并入 `tlsIdentity`

- **组内各码**【源码】：
  - `identityChanged`：`state.py:289`，配对时已有这台设备的记录，但新出示的证书与记录不同；"Use device trust with a trusted certificate"。
  - `tlsIdentity`：`coordinator.py:302`（配对发现阶段，同样是已存记录的证书与新证书不同）、`transport.py:217`（连接时证书与已存不同）、`transport.py:284`（欢迎消息里的设备名或编号与已存不同），以及 `transport.py:81` 的证书校验失败；消息"Device certificate changed; import trusted certificate"。
- **为什么下一步完全相同**：`coordinator.py:302` 和 `state.py:289` 判的是同一件事（已存证书不等于设备现在出示的证书），只是在配对流程的两个检查点；两处的提示都是"从可信途径拿到证书后 `device trust`"。调用者下一步都是停下来请用户核实设备身份，用户给出可信证书后发 `androidctl device trust DEVICE certificate.pem`（`connection.md:34`）。
- **合并后**：叫 `tlsIdentity`。不需要附加字段；`stage` 已经能区分配对与连接（`identityChanged` 目前没有 `stage`，合并时补一个即可）。
- **连带改动**：构造 1 处；测试 0 处断言 `identityChanged`；指引 0 处；契约 0 处；没有分支。约 1–2 处。
- **反方意见**：上一轮不合并 `identityChanged` 的理由是"它是安全信号，不能并进界面的 `changed`"。这个理由仍然成立，但并入 `tlsIdentity` 保留了安全含义（`tlsIdentity` 本身就是安全信号）。另一点：`tlsIdentity` 自己还混着"时钟不对、证书过期"两种情形（`transport.py:94-100`），那两种的下一步是"校准时钟"或"在设备上续签再 `device trust`"，与"证书变了"不同。这是 `tlsIdentity` 原有的问题，合并不会让它更混，单独列在 §五。
- **建议**：合并，高把握。

#### 第 3 组：没有当前观察 —— `noPublishedSnapshot` 并入 `noSnapshot`

- **组内各码**【源码】：
  - `noSnapshot`：`UiSnapshot.kt:85`、`UiSnapshot.kt:118`（缺省值），"No current snapshot; run snapshot"。用在引用类动作取缓存观察时（`UiActions.kt:76` 起十余处 `ui.snapshots.cached()`）。
  - `noPublishedSnapshot`：`UiDelivery.kt:98,100`，"No current published node observation; run snapshot"。用在 `wait` 的"页面变化"基线。
- **为什么下一步完全相同**：两者回执里写的做法都是 `run snapshot`，拿到之后调用者的下一条命令都是 `snapshot`，然后重发原命令（引用类动作要换成新观察里的引用，`wait` 原样重发）。结果都是"未发出"。
- **合并后**：叫 `noSnapshot`；可以带 `subject=published`，调用者用不上。
- **连带改动**：构造 2 处；设备端分支 1 处（`UiCondition.kt:335` 的集合 `noPublishedSnapshot, partialBaseline, scopeMismatch` 改成含 `noSnapshot`；要确认 `noSnapshot` 不会从别的路径落进这个 `catch`，读源码看 `published()` 里只会抛这三种，问题不大）；测试约 2 处断言（JVM）；指引 0；契约 0。约 5 处。
- **反方意见**：`partialBaseline`（基线必须完整）看起来也像，但它的下一步不是简单 `snapshot`：页面太大时再采也不完整，要缩小范围，所以不放进这一组。
- **建议**：合并，高把握。

#### 第 4 组：主机端未预料的异常 —— `internalError` 并入 `internal`

- **组内各码**【源码】：
  - `internal`：`file_jobs.py:554`，文件任务里未分类的异常，`stage=host.file.*`。
  - `internalError`：`local_server.py:171`，本地守护进程处理请求时未分类的异常，带 `uncertain`。
- **为什么下一步完全相同**：两个都是"主机自己的代码出了未预料的异常，消息是异常文字"。调用者能做的只有：结果是"未知"时先只读核对（结果词已经说明），然后停下报告。没有任何代码按这两个码分支。
- **合并后**：叫 `internal`，`stage` 区分来源。
- **连带改动**：1 处构造；测试 0 处断言 `internalError`（`internal` 的 3 处断言不受影响）；指引 0；契约 0。
- **反方意见**：没有。设备端同义的 `workerFailure` 能不能一起并，见丙组第 10 组。
- **建议**：合并，高把握。

#### 第 5 组：请求或回执超出大小上限 —— `messageLimit` 并入 `jsonLimit`

- **组内各码**【源码】：
  - `jsonLimit`：`UiReceipt.kt:11`（回执超出 JSON 预算，`uncertain`）、`WorkerConnection.kt:191`（请求超出）、`transport.py:489`（主机发出前整条请求超出 UTF-8 预算）。
  - `messageLimit`：`GatewayClient.kt:34`（回执超出网络预算，`uncertain`）、`transport.py:41`（主机发出前请求超出消息上限）。
- **为什么下一步完全相同**：两个码是同一个限制在两层（JSON 编码层和网络消息层）的体现，而且各自都同时有"请求超了"和"回执超了"两种用法，划分方式完全一样。请求超了：未发出，下一步缩短输入（更短的文本、脚本）后重发；回执超了：动作已执行，结果未知，下一步只读核对、改用更小的读取范围。调用者拿到哪个码都按同样的方式区分，靠的是结果词，不是码。
- **合并后**：叫 `jsonLimit`（或更直白的 `sizeLimit`）；可以带 `subject=request/reply`，结果词已经足够区分。
- **连带改动**：构造 2 处；测试约 1 处；指引 0；契约 0；没有分支。
- **反方意见**：调试时可能想知道是哪一层超了；消息文字仍然写着。
- **建议**：合并，中高把握。

#### 第 6 组：同一情形用了两个码，归到一个（不减个数）

这三处不是"两个码合成一个"，而是"同一件事在不同地方报了不同的码"，把少数用法移到已有的码下。按"下一步相同就用同一个码"的标准，它们比合并更没有争议。

- **6a. 滚动"这个方向没有路线"统一报 `routeUnavailable`。** `UiActions.kt:1009` 写着：同一个检查，命令是 `scroll-until` 时报 `routeUnavailable`，是 `scroll` 时报 `notReady`；两者的回执文字完全相同（"target has no advertised route for scroll-down; current scroll directions: …; use androidctl scroll eN forward"）。调用者的下一步都是换列出的方向，或者认定已经到头。真实运行里 `notReady` 出现 7 次，其中 6 次正是这个情形（`u35-c1-candidate-01/03/05`、`u35-c1-baseline-02`、`r5-c1-play-candidate-01` 等，`evidence/cli/*/child.stdout`），模型 4 次直接改用列出的方向、1 次先 `snapshot`、1 次 `back`；`scroll-until` 报的 `routeUnavailable` 1 次（`r5-c2-final-candidate-04`），模型改成反方向再滚。改法：把 `UiActions.kt:1009` 的条件去掉，两种命令都报 `routeUnavailable`。连带：1 处构造；测试未见断言滚动报 `notReady` 的（`notReady` 的测试断言在文件传输和引用修复里，与此无关）；指引不点名。好处是 `notReady` 去掉了它最常见的一种情形，剩下的含义更集中（目标不可见、被禁用、没有输入焦点）。高把握。
- **6b. `mappingUnknown` 的"坐标输入要求 user 0"改报 `userScope`。** `VisualInput.kt:389` 报 `mappingUnknown("Coordinate input requires user 0")`，而别处同一条件都报 `userScope`（`AppInput.kt:51` "App input requires user 0" 等 5 处）。下一步都是请用户切回主用户。1 处构造，没有测试、指引、契约。高把握。
- **6c. 后台屏截图失败改报 `screenshotUnavailable`。** 同一个函数 `VisualInput.captureBitmap`（`VisualInput.kt:181-186`）：主屏截图失败报 `screenshotUnavailable`（"Platform returned no screenshot""Software screenshot copy failed"），后台屏截图失败走 `WorkDisplay.screenshot` 报 `imageUnavailable`（"Work display capture was not confirmed""…returned no HardwareBuffer""…returned no Bitmap""Work display software copy failed"，`WorkDisplay.kt:136-146`），文字几乎一一对应。下一步都是再截一次，仍不行就不用图像继续或报告。例外是 `WorkDisplay.kt:130` 的"Work display image consumer failed"：这个错误一旦发生就一直存在（`consumerError` 只置不清，`WorkDisplay.kt:28,121-130`），再截多少次都一样，下一步应当是关闭并重开后台屏会话。所以建议只把那 4 处采集失败归到 `screenshotUnavailable`，`imageUnavailable` 留给"后台屏的图像通道坏了，要重开会话"这一种，并在消息里写明要重开。个数不变；4 处构造；测试、指引、契约 0。高把握。如果用户更看重个数，也可以把那一处也并进去、靠消息说明要重开，这样减 1 个，但不完全满足"下一步相同"。

### 乙、中把握

#### 第 7 组：依赖当前状态、但下一步就是改参数重发的码 —— 并入 `arguments`

上一轮的判据是"判断不依赖设备当前状态"才并入 `arguments`。新标准看的是下一步：下面这些码判断时要看当前状态，但调用者拿到之后的下一条命令都是"同一条命令、改一个参数"，与 `arguments` 相同。

- **组内各码与归位**【源码】：
  - `actionLabel`（`NodeActions.kt:89,91`）、`actionName`（`NodeActions.kt:83`）：标签或短名不完整、或者没有恰好匹配一个当前公布的动作；回执都写"select an (advertised) action ID"。下一步：改用 `--action-id N` 重发。并入 `arguments`，`argument=actionLabel`／`actionName`。
  - `duration`（`PointerGesture.kt:179`）："The whole gesture duration must fit the remaining UI work budget"。下一步：缩短手势时长或加大 `--timeout` 重发。并入 `arguments`，`argument=duration`。
  - `point` 的坐标用法（`PointerPlan.kt:52`）："Every gesture point must be inside the delivered image"。下一步：按交付图像的尺寸改坐标重发。并入 `arguments`，`argument=point`。
  - `region` 的坐标用法（`ImageRegion.fromCoordinates` 调 `within`，`ImageRegion.kt:147,189`）："Region exceeds the complete image boundaries"。下一步：改 `--region` 重发。并入 `arguments`，`argument=region`。
  - `range` 的取值越界（`NodeRange.kt:50`）："Platform value must lie within the original target's current comparable min/max range"。下一步：按回执和观察里的范围改值重发。并入 `arguments`，`argument=value`。
  - 设备端网关的超时取值检查（`NetworkGateway.kt:297`，"Timeout must be 1 to $maximum ms"，现在报 `timeout`）：纯参数校验，上一轮漏网。并入 `arguments`，`argument=timeout`。主机端应该先挡住了（未逐一核实），所以这一处多半走不到；`timeout` 码本身因为其它用法保留，个数不变。
- **同时归位到别的码（让 `point`、`region`、`range` 消失）**：
  - `point` 的引用用法（`AppInput.kt:155`，"Fresh target center is outside the display"）：同一函数 `nodeTouchPoint` 里"目标不可见""目标宽高为零"都报 `notReady`（`AppInput.kt:142,149`），"中心在屏外"是同一类情形，下一步都是先把目标滚到屏内或重新观察。并入 `notReady`。
  - `region` 的引用用法（`ImageRegion.fromRef`，`ImageRegion.kt:160`）：按引用截区域时目标的当前边界超出屏幕，下一步是把目标滚进屏内或改用整屏截图，不是改参数。并入 `notReady`。实现时 `within` 要区分调用方（加一个参数或拆成两个函数）。
  - `range` 的"最小值／最大值读不到"（`NodeRange.kt:45,48`）：与 `rangeUnavailable`（`NodeRange.kt:42`，"Original target has no current range information"）一样，下一步都是这个目标现在不能用 `range`，改用滚动或拖动。并入 `rangeUnavailable`。
- **合并后个数**：`actionLabel`、`actionName`、`duration`、`point`、`region`、`range` 六个码消失，减 6 个。
- **连带改动**：构造约 12 处（上面列的位置）；测试约 10 处断言要改（`NodeActionsTest.kt:72,76` 等处断言 `actionName`；`ImageRegionTest.kt:48` 正好断言上一轮的划分"越界报 `region`、格式错报 `arguments`"，`ImageRegionTest.kt:82` 断言引用越界报 `region`；`ImageCoordinatesTest.kt`、`PointerGestureTest.kt`、`duration` 测试各约 1 处）；主机端没有按这些码分支；指引没有点名；契约样本 0 处（`contract/wire-contract.json` 里出现的 `point`、`region`、`actionLabel` 都是参数字段，不是码）。主机文本输出会从 `code=point` 变成 `code=arguments; argument=point`。
- **反方意见**：
  1. `arguments` 的含义会从"写法错"变宽成"改参数就能好"。`reference.md:84` 写的是"`arguments` identifies parameter errors"，这句话仍然成立，但上一轮用户批准的判据（"判断不依赖设备当前的状态"）要改写。设备端只有一处消费者按 `arguments` 分支（`UiRun.kt:129`：`wait` 的 `arguments`＋`argument=condition` 报"未发出"、其它 `arguments` 报"失败"），这些新成员都不在 `wait` 里，不受影响。
  2. `actionLabel` 的"标签不完整"情形，调用者要从观察里读出动作编号；观察里已有，不需要新观察。如果实际观察的默认文本不打印动作编号（第六轮 A1 刚动过这部分输出），调用者就得先 `snapshot --json` 再重发，下一步就不止是改参数了。这一点要在 A1 改完之后核对一次【未核实】。
  3. `point` 坐标用法：如果调用者是用一张旧图算的坐标，"改坐标"之前其实要先截新图。不过旧图会先被 `--image` 守卫以 `stale, subject=image` 挡掉（`images.md:106-108`），走不到这里。
- **建议**：合并，中把握。这一组改的是上一轮用户亲自批准的判据，请用户明确认可"改参数就能好"这个新口径。
- **外围成员（中低把握，可单独批）**：
  - `deviceName`（`state.py:285`，"This device name is already paired"）：下一步通常是换 `--name` 重发；但也可能是用户本来就要把旧记录 `device forget` 掉，两种做法都合理，回执没说。上一轮已把 `pairingName` 并入 `arguments`。减 1。
  - `scriptFile`（`job_commands.py:79`，"Cannot read script file {path}: {error}"）：文件不存在时是改路径；权限不够时要先改文件权限，不只是改参数。主机测试 `test_cli_arguments.py:289,336` 断言它。减 1。
  - `scopeMismatch`（`UiActions.kt:446,456`、`UiCondition.kt:110`）："Ref belongs to window N; include its original window in the action scope"等，下一步是补 `--window N` 重发，也是改参数。但 `UiCondition.kt:335` 按它分支、契约样本里有 2 处，改动面比前两个大。减 1，低把握。

### 丙、中低和低把握（我倾向不做或只做第 8 组）

#### 第 8 组：Shizuku 未就绪 —— `shizukuUnavailable`、`shizukuPermission`、`shizukuMode` 合成一个

- **组内各码**【源码】：`shizukuUnavailable`（`WorkerConnection.kt:292,327`，"Start Shizuku on the device"）、`shizukuPermission`（`WorkerConnection.kt:170,294`，"Authorize AndroidCtl in Shizuku"）、`shizukuMode`（`WorkerConnection.kt:336`，root 模式或身份不明）。前两个在每次调用常驻进程时检查（`requireTarget`），会到达调用者；`shizukuMode` 主要在 App 启动和绑定时检查，多半只在设备管理界面出现（推断）。
- **为什么下一步相同**：指引把 Shizuku 定为用户准备的环境（`SKILL.md:8`、`connection.md:52,87`），调用者没有办法自己启动或授权。三者的下一步都是停下来把消息转告用户，用户处理完以后原样重发。
- **合并后**：叫 `shizuku`，可带 `subject=unavailable/permission/mode` 供人看。减 2。
- **连带改动**：5 处构造；测试、指引、契约都是 0；没有分支。
- **反方意见**：如果调用者手里有 ADB（指引的连接一节讲了 ADB 路线），它也许能自己通过 ADB 启动 Shizuku，但不能替用户点授权；那样两者的下一步就不同了。指引没有让调用者这样做，所以按现行指引仍相同。另外收益很小：37 份运行里一次都没出现。
- **建议**：可以合并，中低把握；不做也没有损失。

#### 第 9 组：传输途中文件变了 —— `sourceChanged`、`temporaryChanged`

- **组内各码**【源码】：`sourceChanged`（`TransferHandle.kt:92,118,128`；`file_jobs.py:284,293,297`：源文件比声明的短、长或元数据变了）、`temporaryChanged`（`TransferHandle.kt:144`、`user_files.py:424`：发布前临时文件的身份变了）。
- **为什么下一步相同**：都发生在发布之前，结果都是"失败、没有发布"；下一步都是重跑同一条 `push`／`pull`。
- **合并后**：叫 `sourceChanged`，带 `subject=source/temporary`。减 1。
- **连带改动**：8 处构造；测试 1 处（`test_file_jobs.py:201`）；指引 `jobs-files.md:61` 点名 `sourceChanged`，不用改。
- **反方意见**：源文件变了，往往是它还在被写，调用者应该等它写完（或问用户）再重传；临时文件变了，是有别的东西动了 AndroidCtl 自己的临时文件，可以立即重传。下一条命令相同，时机不同。上一轮不并入界面 `changed` 的理由（"要重传"而不是"重新观察"）仍然成立。
- **建议**：中低把握，不合并也可以。

#### 第 10 组（宽版）：实现缺陷类再并入 `internal`

- **组内各码**：`workerFailure`（设备端所有非 `Rejected` 异常的默认码，`Wire.kt:80`）、`invalidRequest`（`local_server.py:136,147`）、`artifact`（`UiDelivery.kt:399`）、`method`（`WorkerService.kt:96`、`coordinator.py:773`）。它们都表示"主机和设备之间或实现内部出现了不该出现的情形"，调用者下一步都是停下报告。减 4。
- **反方意见**：`workerFailure` 同时兜住了平台抛出的异常（权限异常、远端进程死掉等），其中一部分重试一次就好，和"实现缺陷"不完全一样；契约样本里有 9 处、JVM 测试约 8 处断言它，改动面最大。`method` 在版本不配套时出现，那时的下一步是"装配套版本"，与 `protocolMismatch` 相同而不是与 `internal` 相同（不过版本不配套时握手已经先报 `protocolMismatch`，多半走不到）。
- **建议**：低把握，不建议本批做。

#### 第 11 组：传输完整性 —— `chunk`、`offset`、`size`、`hash`

- 四个码都表示"传输的数据块、偏移、大小或摘要对不上"（`FileJob.kt:110,114,230`、`TransferHandle.kt:81,104,106`、`FileJobs.kt:23`、`Wire.kt:69`、`file_jobs.py:289,324,345`），结果都是失败、没有发布，下一步都是重跑传输，仍不行就报告。减 3。
- **反方意见**：四个名字同时是协议字段名（`offset`、`size`、`hash`、`chunk`），在源码、测试、契约里大量出现，挑出"作为码"的那几处要逐行看；调用者实际上见不到它们（真实运行 0 次）。收益几乎为零。
- **建议**：低把握，不建议做。

#### 第 12 组：观察上限 —— `nodeLimit`、`treeByteLimit`

- 两者作错误码时都来自 `limits.first()`（`UiActions.kt:913`、`UiCondition.kt:430`），指引 `actions.md:30` 把它们并列；下一步都是缩小观察范围（`--window`、先滚动、改用引用）。所有消费者都把它们当一个集合处理（`UiObservation.kt:63`、`UiSelector.kt:103`）。减 1。
- **反方意见**：调用者的反应确实相同，但这两个字符串同时是观察数据里"缺口原因"的词汇（`missing[].reason`、`treeProblem.code`、根问题），合并就要改观察数据格式，或者让错误码与数据词汇不一致（同一件事两个名字，正是这次要消除的东西）。连带改动是所有组里最大的（设备源码 8 处以上、JVM 测试约 30 处、指引 1 处、契约 1 处）。
- **建议**：满足标准，但不建议本批做；如果以后改观察数据格式，再顺便合。

### 合计

| 范围 | 减少 | 合并后 |
| --- | --- | --- |
| 甲（第 1–6 组） | 6 | 150 |
| 甲＋乙核心（第 7 组核心） | 12 | **144** |
| ＋第 7 组外围 3 个 | 15 | 141 |
| ＋第 8、9 组 | 18 | 138 |
| ＋第 10、11、12 组 | 26 | 130 |

我建议批准到"甲＋乙核心"，即 144 个；外围和丙组由用户按偏好取舍。

---

## 四、考虑过但不建议合并

1. **`stale` 与 `changed`**（上一轮合出的两个）：今天两者的下一步基本都是 `snapshot` 后换引用。但第六轮第 10 条裁决要让"引用因名字变化被拒"的回执附上"同位置现在是 eNNN 'Y'"——真实运行里这类拒绝报的是 `stale, subject=ref`（例如 `u35-c1-candidate-01` 的 `click e124387726`）——之后拿到这种 `stale` 的调用者可能直接核对并改用新引用，不再先 `snapshot`，而 `changed` 仍然要 `snapshot`。另外 `UiRefRepair.kt:49-51` 按 (`stale`, `ref`) 和 (`changed`, `selection`) 决定是否修复引用。不合并。
2. **`sessionRequired` 与 `sessionReleased`**：两条回执的做法文字完全相同（"run session open --display main|virtual explicitly"，`Sessions.kt:92,149,371`），按字面应当合并。但"已释放"是设备前的人主动收回了会话（`displays.md:83`），调用者合理的反应是先问用户为什么、要不要继续，而不是马上重开；"已到期"只是空闲超时，直接重开就对。我认为该改的是 `sessionReleased` 的提示文字（写明先确认用户意图），而不是合并。
3. **`busy` 与 `connectionBusy`**：指引把两者并列（`SKILL.md:23-24`、`connection.md:93`："requires respecting the current holder"）。但 `connectionBusy` 是主机上那条设备连接被另一个会话的请求占着（`transport.py:377,390`），通常几秒内就空出来，原样重试即可；`busy` 是会话租约被别人持有（`Sessions.kt:386`），可能要等到对方关闭或空闲 30 分钟到期，下一步更像是告诉用户。不合并。
4. **`snapshotBudget` 与 `timeout`**：都是"时间用尽"。但 `timeout` 自己混着"连不上设备"（主机 `coordinator.py:143` 等）和"取值越界"（见第 7 组），而 `snapshotBudget` 在采集阶段出现时往往表示界面树太大或 App 无响应（`UiBudget.kt:55-56` 有"the app may be unresponsive"的诊断），缩小范围比加时间更有用。不合并。
5. **`pairingExpired`、`pairingUnconfirmed`、`unauthorized`、`revoked`**（都和"重新配对"有关）：`pairingUnconfirmed` 的提示是"用当前配对码再 pair 一次，先查保存的凭据"（`state.py:314`），未必需要新配对码；`unauthorized` 混着"配对窗口被拒"和"已有凭据被拒"两种（`transport.py:106-116`），主机还按 (`unauthorized`, `device.authentication`) 分支做回退（`coordinator.py:430`）。只有 `pairingExpired` 与 `unauthorized` 的"配对窗口被拒"那一半下一步完全相同，见 §五。
6. **`jobUnavailable` 与 `outputExpired`**：前者可能是编号写错（改参数），后者是编号对但输出已过保留期。不合并。
7. **`magnificationActive` 与 `magnificationOverlay`**（同一函数 `VisualInput.kt:366,406,411`）：放大功能开着时都拒绝坐标输入。但"覆盖层还在"可能是关掉放大之后短暂残留，那时等一下重试就行，而"放大开着"要请用户关掉。没有设备，分不清，不建议合并【需设备】。
8. **`clipboardTransfer`、`clipboardUnavailable`、`clipboardUnconfirmed`**（借用粘贴的三种失败）：原剪贴板含 URI 时永远借不了，要改用别的输入方式；后两种可能是瞬时的，可以重试。不合并。
9. **`daemonUnavailable` 与 `daemonVersion`**：版本不符的下一步是停掉旧守护进程，端点不可用的原因不明（`local_call.py:130,157`）。不合并。
10. **`identityUnknown`、`sourceUnknown`、`scopeUnknown`**（"事实读不到"三兄弟）：`scopeUnknown` 的回执给出具体写法（"snapshot --window 24"），`identityUnknown` 再采可能还是读不到（App 不提供地址），`sourceUnknown` 里还混着"App 按键需要输入焦点窗口"。不合并。
11. **"停下请用户处理"整类（26 个）、"只能报告"整类（27 个）各并成一个码**：按最粗的读法下一步都是"停下、转告"。不建议，理由：（1）用户处理完之后，调用者的后续命令各不相同（重新配对、`device trust`、原样重试、换用户），我采用的判法要求后续也相同；（2）这些码调用者几乎见不到（37 份运行 0 次），合并省不了调用者什么；（3）它们是维护者排查时唯一机器可读的区分，消息是自由文字。是否采用这个更粗的读法，请用户定。
12. **`workerLockChanged` 与 `workerUnavailable`**：两者都要用户到设备上处理执行器，但锁变化之后的恢复路径（刷新状态是否足够，还是要停掉执行器）我在源码和指引里都没核到。不合并。
13. **`staleWorker`**：只在设备管理界面的"强制退出执行器"里出现（`GatewayService.kt:221`、`MainActivity.kt:196`），不在调用者标准的适用范围。
14. **`gatewayStopping` 与 `stopping`**：前者多数用法只在设备管理界面（续签证书、停止网关），只有 `NetworkGateway.kt:262` 一处会到调用者。不合并。

---

## 五、可能该拆或该归位的码（超出裁决范围）

用户这次没有要求拆分。但"新增原因码必须对应不同的下一步动作"反过来意味着现有的每个码应当能告诉调用者下一步。下面只列事实和看法，**是否处理需要用户另行决定**。21 个码内部混着下一步不同的情形（全表"说明"列以"混合"开头），比较值得注意的是：

1. **`notReady`**（10 处构造）：混着"目标不可见（先滚进屏内）""被禁用（等或换目标）""没有输入焦点（先聚焦）""这个方向没有滚动路线（换方向）"。最后一种已在第 6a 组归位；其余的回执文字都写明了原因，我认为不必再拆。
2. **`deviceRequired`**：`state.py:186` 是"一台配对设备都没有，要配对"，`state.py:193` 是"有多台、没选，用 `--device` 选"。后一种与 `deviceNotFound`（`state.py:206`，"run androidctl devices then … --device NAME"）下一步完全相同。可以把后一种归到 `deviceNotFound`，让 `deviceRequired` 只表示"要先配对"；`coordinator.py:298` 按 `deviceNotFound` 分支，要确认归位后不影响那里的内部回退（那里只捕获 `state.record` 的"没有这台设备的记录"）。
3. **`tlsIdentity`**：混着"证书变了"和"时钟不对、证书过期"（`transport.py:81-100`）。设备端已经有 `certificateClock`、`certificateExpired` 两个码（`DeviceIdentity.kt:49,51`），主机端可以复用它们报时钟问题，个数不变。
4. **`unauthorized`**：主机在配对时的"配对窗口被拒；请在设备上显示新配对码"（`transport.py:111-116`）与设备的 `pairingExpired`（`Pairings.kt:63`）下一步完全相同；用已有凭据连接时的"凭据被拒"是另一回事。可以把前一种归到 `pairingExpired`，但 `coordinator.py:430` 按 (`unauthorized`, `device.authentication`) 回退，要一起看。
5. **`sessionChanged`**：混着"会话在预检期间变了，先看 status"（`Sessions.kt:129,193`）、设备管理界面的"显示的会话已变，先刷新"（`Sessions.kt:255-256`、`WorkerService.kt:105`），以及"会话已被关闭或释放；take a new snapshot"（`Sessions.kt:293`）。最后一条的提示可能不对：会话已经没了，`snapshot` 会被 `sessionRequired` 或 `sessionReleased` 挡住，下一步应是 `status` 或 `session open`。这是提示文字的问题，建议顺手改。
6. **`timeout`**：混着"时间用尽"（重试或加大 `--timeout`）、"连接没在预算内完成"（检查网络）、"取值越界"（第 7 组）。
7. **`busy`**：混着"会话被别人持有""本会话的界面或文件资源还在用""上一个文件步骤还在跑，查它的编号"（`FileJob.kt:150`）。最后一种的下一步是查任务状态，不是等待。
8. **`inputUnavailable`**：混着"屏幕关闭或锁定"（调用者可以先 `wake`，锁屏要用户解锁）和"输入服务不可用"（只能报告）。
9. **`stale`**（上一轮合出的）：`subject=ref` 的下一步是 `snapshot`，`subject=image` 的下一步是 `screenshot`，子命令不同。`subject` 已经区分，指引也点名了 `code=stale, subject=image`（`images.md:108`），我认为不必拆；只是指出按新标准的最严读法，上一轮这个合并并不完全满足。
10. **`actionUnavailable`**：混着"动作编号不在公布列表里（换编号）"、"不支持 reveal，直接 click"（`UiActions.kt:113`）、"不支持 PASTE／SET_SELECTION（换输入方式）"。回执文字写明了做法，可以不拆。

---

## 六、规矩的落法建议

用户要立的规矩是"新增原因码必须对应不同的下一步动作"。建议落成下面两件事，都很轻：

**做法一（推荐）：一张码表，加一条只核对"在不在表里"的测试。**

- 在 `contract/` 下放一个小文件，例如 `contract/reason-codes.json`，每个码一行：码、下一步动作类别（用 §二 的 N0–N9 或更少的几类）、一句话写调用者该做什么、受众（调用者／设备管理界面／内部）。约 145 行。
- 加一条主机测试（放在现有 `test_wire_contract.py` 旁边即可）：用与 `extract.py` 同样的办法扫描两端源码里的 `Rejected("…")`、`Fault("…")`、`TransportFailure("…")`、`"code" to "…"`、`code="…"`，断言每个码都在表里、表里每个码都还有构造处。动态构造的码（§2.1 那十来处）在表里标记为"动态"，测试对它们只核对"在表里"。
- 开发文档（`AGENTS.md` 的 Coding Style 一节或内部 docs 的 `reference/README.md`）加一句话：**"新增原因码时，在 `contract/reason-codes.json` 加一行；如果它的下一步动作类别和那句做法与表里已有某个码相同，就复用那个码，用 `argument` 或 `subject` 区分。"**
- 能检验的部分是"每个码都登记了下一步"；"下一步是否真的不同"仍由评审判断，测试判断不了语义，这一点要说清楚，不要让测试显得在证明什么。
- 代价：一次性写表约 1–2 小时（本记录的 `codes-full.md` 可以直接作底稿）；以后每加一个码多写一行；扫描正则要随构造写法变化维护，约 50 行测试代码。表同时可以给指引用：`reference.md` 的错误说明可以从表里摘，不用另写。

**做法二（更轻）：只写一句话，不加测试。** 在 `AGENTS.md` 写上面那句规矩，评审时人工看新码。代价几乎为零，但"码表"不存在，下次再审一轮时又要从头数一遍（本轮和上一轮都花了不少工夫在计数上）。

不建议更重的做法，例如把码定义成两端共享的枚举、由生成器产出 Kotlin 和 Python 常量：要引入代码生成步骤，对一个人维护的 alpha 产品不值得。

---

## 七、全表

逐码一行的全表在 **`S/agents10/82/codes-full.md`**（156 行，列：码、端、构造处数、主结果与退出码、下一步动作类别、受众、指引里点名的位置、按码分支的位置、真实运行出现次数、说明）。生成脚本是同目录的 `classify.py`，人工归类写在脚本里的字典中，可以逐条复查。

几个要点摘在这里：

- **指引里点名的码**（15 个）：`arguments`、`busy`、`connectionBusy`、`covered`、`nodeLimit`、`treeByteLimit`、`stale`、`changed`、`unexpectedTaskMove`、`protocolMismatch`、`sessionRequired`、`sourceChanged`、`publishUnsupported`、`apkUnavailable`，以及作为诊断字段的 `uiCleanup`。建议合并的高、中把握各组里，没有一个码被指引点名。
- **主机按码分支的地方**（8 处）：`protocolMismatch`（`transport.py:270`）、`unexpectedTaskMove`（`ui_output.py:232`，结果未知时换一句提示）、`privatePermissions`（`private_files.py:227`）、`deviceNotFound`（`coordinator.py:298`）、`unauthorized`（`coordinator.py:430`）、`publishUnsupported`（`file_jobs.py:417`）、`temporaryCleanup`（`file_jobs.py:542`）、`daemonStopped`（`app.py:918,939`）。主机没有"按码给修复提示"的表；修复提示都写在各条回执的消息里，`UiAdvice.kt` 只为勾选和滚动方向生成提示文字。
- **设备按码分支的地方**：`collectionProtocol`、`scopeUnknown`、`timeout`、`snapshotBudget`、`noPublishedSnapshot`、`partialBaseline`、`scopeMismatch`、`sessionChanged`、`rootUnavailable`、`nodeLimit`、`treeByteLimit`、`collectionLimit`、`requestEnded`，以及 `arguments`＋`condition`、`stale`＋`ref`、`changed`＋`selection`（位置见全表）。
- **退出码**：主结果词和退出码由"是否已发出"决定（`UiRun.kt:124-130`；网关层由 `uncertain` 标记决定，`Wire.kt:86`），与码无关；只有 `wait`（按 `arguments`＋`condition`）和 `snapshot`（按 `collectionProtocol`）两处由码影响结果词。所以合并原因码不会改变任何命令的退出码。
- **真实运行**【原件】（37 份运行的 `evidence/cli/*/child.stdout` 与 `child.stderr`，脚本 `runs.py`、`runs2.py`）：文本 `code=` 共 15 次：`notReady` 7（6 次是滚动无路线）、`staleRef` 4（全在基线运行，旧码名）、`stale` 3（候选的路线题，都是名字变化导致的引用拒绝，模型每次都先 `snapshot` 再点新引用）、`arguments` 1（`screenshot --json` 把全局选项放错位置，模型下一条改成 `--json screenshot`）。`--json` 回执里另有 `scopeUnknown` 8 次（长列表题 4 次、后台屏题 4 次，都在成功回执附带的 `snapshotProblem` 里，模型下一条都是 `snapshot`）、`routeUnavailable` 1 次。其余 140 多个码在 37 份运行里一次都没出现。

---

## 八、没能核实的内容

1. 受众判断：哪些码只在设备管理界面出现、哪些会到达调用者，我是读调用链推断的（例如 `shizukuMode`、`certificateClock`、`certificateExpired`、`workerBusy`、`workerDirectory`、`credential`、`pairingOnly`、`transferState`、`backendCache`），没有在设备上触发过。
2. 设备端网关的 `--timeout` 取值检查（`NetworkGateway.kt:297`）是否真的被主机先挡住，没有逐条核对主机的参数解析。
3. 第 7 组 `actionLabel` 的反方意见第 2 点：A1 改完默认输出之后，默认文本里是否仍能读到动作编号，要在改完后核对。
4. `magnificationOverlay` 是否只是短暂残留（第四节第 7 条）【需设备】。
5. `workerLockChanged` 之后调用者或用户的恢复步骤，源码和指引里都没有找到明确说法。
6. 测试和契约的连带改动处数是按"断言了这个码"的行粗数，没有逐行确认；`offset`、`size`、`hash`、`chunk`、`point`、`region` 这类与字段同名的码尤其可能多算或少算。
7. `TreeCollector.kt:169` 用根状态字充当码的路径，按源码几乎走不到，没有计入总数；如果走得到，会多出 `failed` 之类的码。
8. 真实运行的版本：候选运行并非都在 `fb75afc` 上（网页、长列表、后台屏题的候选运行在更早的头上，基线运行在 `7d5be70`，所以出现旧码名 `staleRef`）；统计只用来看"调用者实际见到哪些码、之后做了什么"，不代表 `fb75afc` 上的频率。
