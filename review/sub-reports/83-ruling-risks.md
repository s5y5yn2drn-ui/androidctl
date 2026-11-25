# 第六轮裁决落地的风险提示（对照 `fb75afc` 源码）

写给按第六轮裁决修改 `fb75afc` 的实施方。核查对象是产品仓库 `fb75afc`（下文的 `文件:行` 默认指它；Kotlin 文件在 `android/app/src/main/java/com/rainng/androidctl/`，Python 文件在 `androidctl/src/androidctl/`，指引在 `skills/androidctl/`）。裁决原文见 `S/agents10/USER-RULINGS-R6.md`，下文"第 N 条"都指那份裁决的编号。

证据类别：【源码】读 `fb75afc` 源码得出；【原件】读实施方材料包里的原始输出；【重算】用我写的脚本离线统计（脚本在 `S/agents10/83/`，只读材料包，不执行包内任何脚本）；【实跑】在主机环境 `S/r6/venv6`（装的就是 `fb75afc` 的主机端）里运行产品自己的渲染函数；【推断】；【需设备】。没有设备，也没有跑 Gradle，凡是 Android 侧行为都只来自源码。

本文用到的词：
- **调用者**：按指引操作这个工具的 LLM agent。
- **常驻进程**：设备上以 shell 身份运行、通过 Shizuku 用户服务启动的执行进程（代码里叫 worker）。
- **网关**：设备端 App 进程里的前台服务（`GatewayService`），它持有到常驻进程的绑定，并对主机提供网络接口。
- **后台屏**：工具创建的虚拟显示屏。
- **正文**：一条命令打印在结果首行之后的观察文本（头部几行加节点行）；`--raw` 时是整份原始 JSON。
- **首行**：结果的第一行，如 `click: not sent; ...; code=stale`。
- **引用**：节点编号，如 `e124387726`。
- **采样**：常驻进程一次采集到的界面树；"当前样本"是设备端记住、`--cached` 读取的那一份。

---

## 1. 摘要

1. **输出上限放在"打印正文"这一层不会波及 `--json`**：`--json` 现在只打印结果对象，从不包含正文或节点树（【源码】`ui_client.py:1047-1049`；【原件】`snapshot --json` 实际输出约 1.6 KB）。但上限的判断必须同时进入 `finish_delivery_result` 的"打印即补存"和退出码逻辑（`ui_client.py:976-1032`），只改 `print_delivery` 会让"文件没存成、正文也没打印"时的结果和警告自相矛盾。`--json` 自己也能变大（等待和滚动查找的 `refs` 列表不设上限、通知列表、`shell` 的 base64 输出），上界是设备回复的 192 KiB，与正文上限无关。
2. **"文件里永远是全文"现在有四种做不到的情形**：工作区 `.androidctl` 不可写、历史配额或 20 MB 单件上限、请求时限在保存前用完、显式 `--out` 写失败。这些情形下现有代码会把正文直接打印作为补救；加上上限后要改成"只打摘要、命令按交付失败报"。
3. **第 13 条不碰引用身份**：身份六元组不含可见位和边界（【源码】`SelectionIdentity.kt:8-15`、`UiSemantics.kt:320-334`），旧引用不会因可见位翻转被判为"已变化"。可见位进入的是整页签名（`UiSnapshot.kt:136-181`）和十几处行为判断，逐处见第 3 节。
4. **零面积判定有现成的"正确答案"可以对照**：同一个 Chrome 在主屏上已经把"自身边界面积为零"的节点标为不可见，包括"零高度的行内链接包着有面积的卡片"这种结构（98 次全部不可见）。后台屏改成同样的规则，等于让后台屏与主屏一致，而不是引入新的限制。【重算】在 601 份真实树上，后台屏 Chrome 有 58,957 个节点会从可见变成不可见；主屏原生 App 上按"自身面积为零"只影响 2 个无名节点，按"与显示区域交集为零"另外影响约 97 个整个在屏外的节点（多为转场中的页面）。
5. **零面积父节点被改标后，有面积的后代不会从默认输出消失**：主机渲染遇到不可见节点时把它的子节点提上来，不是整棵剪掉（【源码】`ui_view.py:363-365`）；稀疏编码不继承父节点的可见位（`NodeEncoding.kt:39`）。但设备端采集顺序会把这类后代推迟（`TreeCollector.kt:61,185`），选择器会把命中的可见子节点解析到不可见的父节点上并拒绝（`UiSelector.kt:47,54`）。这两点在主屏上本来就如此。
6. **设备端现在分不清"应用自己停的"和"意外退出"**：两类退出走同一个回调 `lost()`（`WorkerConnection.kt:115-136`），代码里没有"这是我主动停的"标记；"强制退出执行器"之后的状态与崩溃完全相同。要先加一个按目标 binder 记录的"预期退出"标记，再挂重绑，否则主动停止会触发重绑。"重新绑定一次"的读法决定崩溃循环时的行为，需要用户定（第 3 节 C2）。
7. **版本号只需改 `VERSION` 一行**：主机版本来自安装元数据，Android 的 versionName/versionCode 由构建脚本从 `VERSION` 派生，契约样本和测试都不写死版本（【源码】`git grep '0\.4\.0'` 只命中 `VERSION`）。唯一的坑是可编辑安装的元数据不会随 `VERSION` 自动更新，改完必须重装主机包再跑质量检查。
8. **需要用户裁决的有五项**（第 3 节）：显式 `--print` 是否越过上限；上限覆盖哪些非界面输出；"重新绑定一次"的读法；粘贴成功但恢复失败时的退出码；第 13 条在主屏上的波及面。其余都是实施方能自己定的细节。

---

## 2. 逐处风险提示

### 2.1 输出上限与 `--json`（第 14 条），以及 `--raw` 默认只保存（第 23 条）

#### 现状：每种模式打印什么、保存什么

主机端一条界面命令的路径是：`execute` 发请求并取回附件 → `deliver` 渲染正文、保存文件、修正结果 → `print_delivery` 打印（【源码】`ui_client.py:643-675, 695-769, 1035-1069`）。正文只在 `render_observation` 里生成一次（`ui_client.py:772-865`），保存和打印用的是同一个字符串，这就是"打印与保存同文"。

| 模式 | 打印 | 保存 | 依据 |
| --- | --- | --- | --- |
| 默认文本 | 首行，加整份正文 | 正文存到工作区 `.androidctl/snapshots/NNNNNN-时间-包名-标题.txt` | `ui_client.py:1050-1055`；`history.py:149-214` |
| `--no-print` | 只有首行 | 同上 | `ui_client.py:759`、`1053` |
| `--print` | 同默认（显式写法） | 同上 | `ui_commands.py:201-208` |
| `--out PATH` | 首行加正文 | 只写 `PATH`（必须是新文件、父目录已存在），**不再另存历史副本** | `ui_client.py:369-389, 896-901` |
| `--json` | 只打印结果对象 `result_wire(result)`，**不含正文、不含节点树** | 正文照常保存，路径在 `data.snapshotPath` | `ui_client.py:1047-1049, 759` |
| `--json --print` | 参数阶段拒绝 | — | `ui_client.py:576-581` |
| `--raw` | 首行加**整份原始 JSON**（窗口、树、图片元数据、附件状态） | 存成 `.json` | `ui_client.py:747, 782` |
| `--all` / `--max-chars` / `--depth` / `--interactive` | 首行加按选项投影后的正文 | 存的也是投影后的正文，不是整棵树 | `ui_client.py:803-812`；`ui_view.py:795-932` |
| `find` | 首行加"命中摘要行＋最深命中及其祖先"（默认最多 20 条命中，`--all` 不限） | 同默认 | `ui_view.py:103, 939-998` |
| 动作命令附带的观察 | 首行加动作后观察的正文（被拒且未发出时不取正文） | 同默认 | `ui_client.py:689-692` |
| `wait` 成功 | 选择器或文字条件：首行加"命中节点及祖先"；其他条件只有首行 | 有正文时同默认 | `ui_client.py:689-692, 794-802`；`ui_view.py:386-414` |
| `wait` 失败 | 首行加当前整页 | 同默认 | 同上 |
| `scroll-until` | 首行（含整个 `match=` 谓词）加最终观察 | 同默认 | `ui_output.py:153-168` |
| `notifications` | 首行，加每条通知一行 `notification={JSON}`；**这些行不是正文，不保存任何文件** | 无 | `ui_output.py:241-258` |
| `screenshot` | 首行（含图片路径和尺寸）；正文只有窗口行，很小 | 图片存到 `.androidctl/shots/`，或 `--image-out` | `ui_client.py:212-223` |
| `snapshot --screenshot` | 首行加正文 | 正文和图片各存一份 | 同上 |

【原件】实际体量：
- 后台屏上维基百科长列表页的那次默认 `snapshot --screenshot`，标准输出 440,511 字节、4,034 行，保存的 `.txt` 是 436,130 字节（`ST/runtime/api34/commands/2712-r5-surface-collection-C-provider-page/stdout.bin`；`ST/runtime/api34/verification/r5-collection-C-api34/.androidctl/snapshots/000003-*.txt`）。
- 同一页的 `--raw` 文件是 3,731,426 字节（同目录 `000004-*.json` 到 `000008-*.json`）。按现状，不加 `--no-print` 的 `snapshot --raw` 会把这 3.7 MB 全打出来。
- 材料包里 576 条带 `--json` 的命令中，`snapshot --json` 的标准输出最大约 1.6 KB，最大的是一次 `notifications --json`，13,114 字节（`ST/runtime/api34/commands/2528-*`）。
- 计分运行里候选 297 次调用的标准输出：中位数 3.8 KB，第 99 百分位 14 KB，只有 1 次超过 32 KB，是网页题的一次 `snapshot --cached --all`，180,175 字节（`ST/efficiency/r5-c2-final-candidate-01/evidence/cli/1795491004568345400-*/child.stdout`）。

【实跑】用 `fb75afc` 的渲染函数 `render_observation` 重新渲染那份后台屏长页的原始 JSON（脚本 `S/agents10/83/render_fix.py`、`render_all.py`）：
- 默认正文 435,415 字节、4,031 行；
- 模拟第 13 条修复（把与显示区域交集为零的 4,441 个节点标为不可见）后，默认正文 49,298 字节、438 行；
- 同一页 `--all` 是 1,374,499 字节；`find a` 默认 14,550 字节，`find a --all` 1,050,064 字节；
- 同一页 `snapshot --json` 打印的只是结果对象，几 KB，与正文大小无关。

结论：第 13 条修好之后，这一页的默认正文约 49 KB，低于 64–80 KB 的阈值，上限在这页的默认输出上不会触发；它真正会触发的是 `--all`、`find --all`、大页面上的 `wait` 选择器命中以及 `--raw --print`。

#### 上限应放在哪一层

- **放在"打印正文"这一步，并且只在文本模式下生效。** 判断条件是"不是 `--json`、没有 `--no-print`、正文存在、正文字节数超过阈值"。`--json` 走的是 `output.json(result_wire(result))` 这条分支（`ui_client.py:1047-1049`），从来不碰正文，所以不会被波及。
- **不能只改 `print_delivery`。** 现有的 `finish_delivery_result` 在打印之前就决定了"自动保存失败时用打印的正文补救"（`printed_body_fallback`，`ui_client.py:976-984`），会写入警告"正文跟在标准输出里"，并据此决定 `snapshot`/`find` 是否算交付成功、退出码是否变 1（`ui_client.py:985-1032`）；`print_delivery` 的 `finally` 也会在"正文没打印且没有 `snapshotPath`"时把 `snapshot`/`find` 改成失败（`ui_client.py:1056-1064`）。如果超限时只在打印处改成摘要，就会出现"警告说正文在标准输出里、实际只打了摘要"以及"结果仍是 done"的矛盾。建议在 `deliver` 里算出一个 `body_capped` 布尔值，传给 `finish_delivery_result` 当作"不打印"处理，再传给 `print_delivery` 决定打摘要还是全文。
- **首行和非正文的行不受这一层约束。** 首行里会整段打印 `match=` 谓词（`ui_output.py:153-154`）、滚动查找的步骤明细、选择区间等；`notifications` 的每条通知是单独的行（`ui_output.py:256-258`）。第 16 条让滚动查找返回全部匹配引用之后，`match=` 会随匹配数无限增长（见 2.8）。这些需要各自截断，不能指望正文上限兜住。

#### `--json` 自己会不会打爆

会，但与正文无关，也不是这次裁决引入的：
- 设备端单次回复的 JSON 上限是 192 KiB（【源码】`Wire.kt:13` 的 `REPLY_JSON_BYTES`），这就是 `--json` 结果对象的大致上界，再加主机补的十几个字段。
- `wait` 的选择器和文字条件会把全部命中引用放进 `data.predicate.refs`（`UiCondition.kt:153, 173, 205`）。【重算】在那份后台屏长页上，`--role text` 现在能命中 2,838 个可见节点，`refs` 约 40 KB；第 13 条之后可见节点降到约 530 个，这个数字会小很多。
- `shell --json` 会把两个流的输出各以 base64 放进 `data.outputBytes`（`job_commands.py:218-222`），每个流在设备端保留上限 128 KiB（`Wire.kt:23`），最坏约 350 KB。
- 建议：第 14 条不处理 `--json`，在决定文件里写明"`--json` 打印结果对象，不受正文上限约束，上界由设备回复上限决定"。

#### 与 `--no-print`、`--out`、`--raw` 的组合

| 组合 | 建议行为 | 理由 |
| --- | --- | --- |
| 默认，正文超限 | 首行；头部的溯源行和 `observation` 行；一行摘要；文件路径 | 头部带采集编号和缺口信息，是"结果未知后的恢复"和"采样不完整不能证明不存在"两条规则的依据（`reference.md:66-76`、`SKILL.md:76`），不能随正文一起省掉 |
| `--no-print` | 不变，只有首行 | 已经是"只保存" |
| `--out PATH`，超限 | 摘要里给 `PATH` | 显式 `--out` 时不另存历史副本（`ui_client.py:896`），摘要必须指向 `PATH` |
| `--out` 写失败，超限 | 只打摘要和错误，退出码 1 | 现状会把正文全打出来作补救；加上限后没有补救文件，只能如实报交付失败 |
| 自动保存失败（`.androidctl` 不可写、配额、20 MB 单件上限、时限用完），超限 | 只打摘要和 `historyError`；`snapshot`/`find` 报 failed，动作命令主结果不变、退出码 1 | 与现有"既没保存也没完整打印就算交付失败"的规则一致（`ui_client.py:1056-1064`）；摘要里给出 `--workspace-root` 或 `--out` 的改法 |
| `--raw`（第 23 条之后） | 默认只保存，首行给路径 | 第 23 条 |
| `--raw --print` | 见第 3 节 C1 | 与"不新增选项"有张力 |
| `--raw --no-print` | 与默认相同，建议仍接受，不报错 | 兼容已有写法 |
| `--raw --json` | 不变：打印结果对象，原始 JSON 存文件 | 指引 `observations.md:101` 正在推荐这个写法 |
| `--raw --out PATH` | 写 `PATH`，不打印 | 与第 23 条一致 |
| `--raw` 加 `--all`/`--depth`/`--max-chars`/`--interactive` | 建议在参数阶段拒绝，或明说被忽略 | 现状 `--raw` 时正文就是整份原始 JSON，这些投影选项对正文不起作用（`ui_client.py:782, 803`），却不报错 |

"文件里永远是全文"的"全文"要写清楚：它是**这条命令按所给选项渲染出的完整文本**，不是整棵树。带 `--depth 2` 的命令，文件里也只有两层。

#### 摘要至少要有什么

1. 原首行，不动。
2. 正文的头部行（溯源行 `androidctl capture=... sequence=...`、`observation {...}`，有图片时加 `image` 行）。这几行一般不到 1 KB。
3. 一行说明，例如：`body not printed: 1374499 bytes, 4976 nodes (4441 offscreen), 2089 refs; full text: PATH`。
4. 一行缩小范围的提示，按命令给可照抄的写法：`snapshot --cached --interactive`、`snapshot --cached REF`、`find TEXT`、`snapshot --cached --max-chars 20000`。网页题里调用者用 `--all` 读屏外内容，是因为"按引用读缓存时屏外子树为空"（评审报告 B3）；B3 修好后，提示里优先给 `snapshot --cached REF --all`。
5. `find` 超限时额外保留命中摘要行（`find query=... matches=N offscreen=M shown=K`），否则调用者连命中数都看不到。

阈值依据可以直接用这次的数：计分运行 297 次调用中第 99 百分位 14 KB、最大的默认输出（非 `--all`）不到 32 KB；后台屏长页修好后 49 KB。取 64 KB 时，这些都不触发，只有 `--all` 一类会触发。

#### 非界面命令的现状（只报告事实，范围由主评审向用户确认）

- `shell`：文本模式每个流最多显示 4,096 字节，超出时打印 `[display truncated]` 和一条导出命令（【源码】`job_commands.py:47, 264-277`）。它不自动保存文件，完整输出留在设备内存里，最多 8 份、2 分钟（`Wire.kt:23-25`）。这已经是一个更小的、另一种形态的上限，而且"不自动保存"与第 14 条"文件里永远是全文"的前提不同。
- `job output`：必须带 `--out`，不打印内容（`job_commands.py:653-686`）。
- `pull`/`push`：写文件，只打印状态。
- `apps`：每个匹配的应用一行，没有上限，实际一般几 KB。
- `notifications`：通知行不是正文、不保存文件，条数和文字长度受设备端限制，实测最大 13 KB。
- `status`、`devices`、`daemon status`：很小。

#### 必须上设备确认的
- 无。这一节全部是主机端逻辑。

#### 建议的测试
- 超限的默认 `snapshot`：首行、头部行、摘要、路径都在；文件内容与未加上限时渲染的正文逐字节相同；退出码 0。
- 超限加自动保存失败：只打摘要，`snapshot` 失败，没有"正文跟在标准输出里"的警告。
- 超限加 `--out`：摘要里是 `--out` 路径，历史目录没有新文件。
- `--json` 同一份大观察：输出与修改前逐字节相同。
- `--raw` 默认：不打印 JSON，文件存在；`--raw --json` 不变。
- 用本节那份 3.7 MB 原始 JSON 作夹具，覆盖 `--all` 和 `find --all` 两种超限。

---

### 2.2 引用身份与可见位（第 13 条）

#### 现状：身份和签名里有什么

- 引用身份是六个字段：来源节点编号、窗口、角色、名字、行标签、行列链（【源码】`SelectionIdentity.kt:8-15`，由 `UiSemantics.identity` 生成，`UiSemantics.kt:320-334`）。**不含可见位、不含边界、不含屏内面积。**
- 名字的计算（`UiSemantics.selection`，`UiSemantics.kt:81-179`）不看可见位；可见位只影响"父节点是否吞掉子节点的名字"这个显示标记（`UiSemantics.kt:138`），不进身份。
- 跨采样保留引用：最近 3 份已出版样本各存一张"身份到引用"的映射，新树里身份相同且来源地址唯一的节点沿用旧引用（`UiRefs.kt:37-98`，`RECENT_SAMPLES = 3`）。映射不依赖可见位。
- 动作前的身份核对 `requireOriginalIdentity` 只比较身份（`UiVerification.kt:22-33`）。
- 修复打分 `UiRefRepair.match` 用包名、角色、名字、资源编号、类名、行标签、共有动作和边界相近（8 像素以内）打分（`UiRefRepair.kt:85-125`），不看可见位。
- **整页签名包含可见位和边界**：`UiSnapshot.contentSignature` 把 `NodeFacts.observationFields`（含 `visible`、`bounds`）加 `drawingOrder` 等写进签名（`UiSnapshot.kt:136-181`），用于"动作后是否已稳定"、`wait --idle`、`wait --screen-change`、截图失效判断和变化报告。

结论：一个节点从可见变成不可见，**旧引用不会因此失效，也不会被判为"身份已变"**。会变的是：
- 整页签名不同，所以"页面已变化"的判断会更敏感。这是正确的：节点滚到屏外本来就是变化。新旧两份采样都在同一个新版本进程里按同一规则计算，不会出现"同一页面前后两次签名不同"的假变化。
- 升级之后是新的常驻进程，旧引用本来就作废（`UiSnapshot.kt:64-68` 的"New worker: previous refs are invalid"），不存在"旧版本算的引用被新规则判废"的问题。

#### 现状：可见位是怎么来的

- 设备端每个节点的可见位直接取平台的 `isVisibleToUser`（【源码】`NodeFacts.kt:111`），没有自己计算；边界取 `getBoundsInScreen`（`NodeFacts.kt:141`）。
- 采集顺序另外读一次 `isVisibleToUser`（`TreeCollector.kt:185`），不经过 `NodeFacts`。
- 动作前的新鲜读取走 `FreshNodes.read`，再次调用 `NodeFacts.read`（`FreshNodes.kt:39, 72`）。
- 显示尺寸在采集**之后**才放进树里：`collected.collect().put("display", targetFacts(run))`（`UiBackend.kt:397-413`）。采集过程中 `TreeCollector` 手里只有窗口列表和每个窗口的边界。

**落点风险**：如果只在采集后处理（例如在 `UiSnapshot.assemble` 里改写 `visible`），快照和动作前的新鲜读取就会用两套规则：快照说"屏外"，动作时新鲜读取仍按平台值说"可见"，于是点击照常发出；反过来也可能。**建议写一个纯函数 `onScreenVisible(platformVisible, bounds, windowBounds)`，在 `NodeFacts.read` 里调用（多加一个窗口边界参数），`TreeCollector.enqueue` 和 `FreshNodes.read` 都传入所在窗口的边界。** 这两处手里都有 `AccessibilityWindowInfo`，取边界不需要额外的跨进程调用。纯函数还能在 JVM 测试里直接测，`AccessibilityNodeInfo` 在 JVM 上构造不了。

#### 设备端和主机端消费可见位的地方，以及改后行为

| 位置 | 现在怎么用 | 改后变化 |
| --- | --- | --- |
| 采集顺序 `TreeCollector.kt:61, 185` | 不可见的节点排在所有可见节点之后再展开 | 后台屏长页上屏内约 530 个节点会先采；这是第 13 条要的效果 |
| 选择器 `UiSelector.kt:54` | 只匹配可见节点 | 后台屏上屏外节点不再被按名字选中 |
| "可见部分读全" `UiSelector.kt:96-101, 113-119` | 可见节点缺子节点就不算读全 | 截断的长页更容易判为"可见部分读全"；风险见 2.3 |
| `wait` 文字和选择器 `UiCondition.kt:143-159, 227` | 只看可见节点 | 后台屏上，"等待出现"不再被屏外的同名文字提前满足。这是改正，但原先能"等到"的写法现在会等不到 |
| `wait --gone` `UiCondition.kt:162-176` | 完整树上零个可见匹配 | 后台屏上节点滚出屏幕后能判为"消失" |
| 滚动查找文字模式 `ScrollContainer.kt:26-50`、`UiCondition.kt:227` | 只看可见节点 | 后台屏上"文字已出现"不再被屏外文字满足，会真的去滚动 |
| 滚动查找选择器模式 `ScrollContainer.kt:91` | 只算可见目标 | 同上 |
| 动作就绪 `UiAction.kt:53-54, 74, 95`；`NodeActions.kt:110` | 除 `reveal`（SHOW_ON_SCREEN）外，所有动作要求可见 | 后台屏上点击屏外的引用会被拒为"目标不可见"，要先 `reveal` 或滚动。主屏上本来就如此 |
| `reveal` `NodeActions.kt:110`；`UiActions.kt:145-165` | 豁免可见要求；等到"可见且边界非零"才停 | 不变 |
| `click --touch` 取点 `AppInput.kt:139-148` | 要求可见、边界宽高为正 | 零面积节点本来就被拒，不变 |
| `fill` 验证 `UiActions.kt:830-831` | 输入框不可见时不读回 | 后台屏上输入框被滚出屏外时改报"不可观察" |
| 加载判断 `UiSemantics.kt:397-402` | 有可见的转圈进度条就算加载中 | 屏外进度条不再让页面显得"仍在加载" |
| 覆盖判断 `UiSnapshot.kt:186-193` | 可见、可点、边界为正的节点才能覆盖别人 | 不变（零面积本来就不算） |
| 标注 `ImageMarks.kt:146` | 只标可见目标 | 不变（零面积框本来就画不出） |
| 动作效果和建议 `UiEffect.kt:91`、`UiAdvice.kt:100, 119` | 可见才读文字、才建议 | 与上同向 |
| 主机默认打印 `ui_view.py:363-365, 255-256` | 不可见节点不打印，子节点上提；`--all`/`find` 标 `offscreen` | 后台屏长页默认正文从 435 KB 降到约 49 KB（【实跑】） |
| `find` `ui_view.py:939-998` | 包含屏外节点，计 `offscreenMatches` | 后台屏上 `offscreenMatches` 变得有意义 |
| `snapshot --cached REF` `ui_view.py:311-377` | 根节点不可见时正文可能为空（评审报告 B3） | **后台屏上 B3 会第一次出现**：现在后台屏屏外节点都"可见"，所以不出空正文；改后会。B3 必须与第 13 条一起做 |

#### "屏内面积为零"的几种读法和真实树统计

【重算】脚本 `S/agents10/83/vis_stats.py`，扫描材料包 `pkg6` 到 `pkg9` 里所有带节点和边界的 JSON，按节点内容去重后得到 601 份树（后台屏 35 份，其中 Chrome 25 份；主屏 566 份，覆盖约 30 个 App；共 685,774 个节点；来自多个版本和两种模拟器）。结果在 `S/agents10/83/vis_all.json`。四种读法：

- **甲：边界与显示区域的交集面积为零**（裁决字面）。
- **乙：边界自身面积为零**（宽或高不大于零；平台给的边界已经被裁过）。
- **丙：边界完全在显示区域之外**（含刚好贴边）。
- **丁：边界与所在窗口边界的交集面积为零。**

"翻转"指"现在标为可见、改后不可见"。

| 范围 | 甲 | 乙 | 丙 | 丁 |
| --- | --- | --- | --- | --- |
| 后台屏 Chrome（25 份） | 58,957（有名字 44,027；可操作 9,420） | 58,957 | 58,701 | 58,957 |
| 主屏 Chrome（307 份） | 10（无名字、不可操作） | 0 | 10 | 0 |
| 主屏京东（5 份，MuMu 1080×1920） | 58（有名字 14；可操作 11） | 0 | 58 | 58 |
| 主屏系统设置（36 份） | 39（有名字 11；可操作 7） | 0 | 39 | 39 |
| 主屏哔哩哔哩横屏（15 份） | 2（无名字） | 2 | 0 | 2 |
| 主屏通讯录（28 份） | 0 | 0 | 0 | 2（无名字） |
| 其余 25 个主屏 App | 0 | 0 | 0 | 0 |

样例（【重算】）：
- 后台屏 Chrome 的翻转节点，典型边界是 `[460,1280,535,1280]`，也就是被平台裁成贴在屏幕下沿的零高度；还有 `[0,112,0,112]` 这种贴左边的零宽节点。
- 京东的 58 个是搜索页正在滑入时的帧，整页在 x=1128 到 2112，平台仍报可见。
- 设置的 38 个是页面滑出时在 x=-755 的帧；另有 1 个开关在 `[1083,1402,1239,1546]`，屏宽 1080，完全在右边屏外，平台报可见、可点击。

读法的取舍：
- 乙在主屏上几乎零波及，在后台屏 Chrome 上与甲完全相同，因为平台已经把屏外节点的边界裁成零面积。
- 甲和丁另外把主屏上"整个在屏外、面积为正"的节点（转场帧、那个开关）也标为不可见。这在语义上是对的，但会让那个开关从"能按名字点"变成"要先显示出来"。
- 丁用窗口边界，天然覆盖分屏和多窗口（窗口边界已按旋转给出），在全屏时与甲相同；采集和动作前读取时手里都有窗口边界，不依赖采集后才放进树的显示尺寸。
- **建议**：用丁（与所在窗口边界的交集为零）；在第 3 节 C5 里请用户确认可以接受"主屏上整个在屏外的节点也改标"。如果用户只想动后台屏的问题，就用乙。

显示区域的尺寸：
- 树里的 `display.width/height` 已经是旋转后的逻辑尺寸（【原件】哔哩哔哩横屏样本是 1920×1080，rotation=1）。
- 后台屏尺寸是创建时定的，样本是 720×1280。
- 分屏和多窗口时，显示尺寸比节点所在窗口大，用显示尺寸会漏掉"在本窗口外、在别的窗口里"的节点，所以建议用窗口边界。

#### 必须上设备确认的
- 后台屏上同一长页：改后默认输出的可见节点数，应与同一页在主屏上的可见节点数同量级（第 5 条裁决要求的"后台屏加长网页"检查可以顺带做）。
- 后台屏上 `reveal` 一个屏外引用后，节点的边界和可见位是否在设定的稳定时间内变为正面积、可见。
- 主屏上一次 Activity 转场中途采样，看翻转节点是否只出现在转场帧里。

#### 建议的测试
- `onScreenVisible` 纯函数：贴下沿零高度、贴左边零宽、屏内零高度、负坐标、部分在屏外的正面积节点（必须仍可见）、平台已报不可见（保持不可见）。
- 引用保持：同一身份的节点在两次采样之间从可见变成不可见，引用不变，`requireOriginalIdentity` 通过；动作被拒为 `notReady`（目标不可见），不是 `stale` 或 `changed`。
- 契约样本：`contract/wire-contract.json` 里有 2 个"零面积但标为可见"的节点样例（【重算】），如果生产端测试用的是真实规则，样本会变，需要按 `AGENTS.md` 的流程重新导出并核对。

---

### 2.3 零面积父节点被标为不可见后，有面积的子节点会不会一起被隐藏

#### 现状
- **设备端采集**：优先队列按"不可见、深度、发现顺序"排序（【源码】`TreeCollector.kt:61`）。每个节点的不可见标记按它自己计算（`TreeCollector.kt:185`），不从父节点继承。但子节点只有在父节点出队时才入队（`TreeCollector.kt:300-345`），所以不可见父节点下的可见子节点要等所有可见节点都处理完才会被发现。截断时，它们比别的可见节点更容易被截掉。
- **稀疏编码**：每个字段有固定默认值，可见位默认为真，没有任何从父节点继承的缺省值（`NodeEncoding.kt:7-43`，主机端 `node_facts.py:36, 75-76`）。
- **主机渲染**：默认视图遇到不可见节点时 `destination.extend(nested)`，把它已投影的子节点接到它原来的位置上，自己不出现（`ui_view.py:363-365`）。不加省略标记，也没有"此处有一个屏外容器"的提示。`--all` 和 `find` 不隐藏，只标 `offscreen`。
- **选择器的"可见部分读全"**：只要有一个**可见**节点缺子节点就不算读全；不可见节点缺子节点不影响（`UiSelector.kt:96-101, 113-119`）。
- **选择器解析**：命中的是可见节点，但目标取它的 `selectionTarget`，也就是吞掉它名字的那个可点击父节点；不检查父节点是否可见（`UiSelector.kt:47`）。
- **`find`**：返回最深命中及其祖先（`ui_view.py:417-436, 795-932`），用 `search=True` 投影，不隐藏不可见祖先，只标 `offscreen`。
- **`wait` 成功的正文**：同样用 `search=True` 渲染命中及祖先（`ui_view.py:386-414`），不可见祖先照常打印。

#### 真实树统计
【重算】脚本 `vis_stats.py`（"正面积后代"一栏）、`subtree_rule.py`、`inline_check.py`。"正面积后代"指：节点与显示区域的交集面积为正，并且某个祖先会被改标为不可见。

- **后台屏 Chrome**，按甲：32 个，其中 28 个有名字、5 个可操作；按丙：20 个。只有三种结构：
  - 维基百科页顶部的跳转链接"Jump to content"：父节点零宽 `[0,112,0,112]`，可点击；子节点是文字，有面积。每份树 1 次，共 19 次。
  - regex101 页上一张卡片：可点击的父节点零高度 `[48,1232,672,1232]`，子节点文字"regex101 v14.6.2 is here!"有面积 `[48,1008,672,1232]`。这是"行内链接包着块级卡片"的常见网页写法，2 份树共 8 个。
  - 一个"Go to top of page"按钮：固定定位，挂在一个贴下沿的零高度容器下，自身在屏内，1 次。
- **主屏原生 App**：按乙为 0；按甲只有 1 份 Chrome 旧版本样本里 19 个（标签页切换时一个 x=756 的屏外帧，子节点在屏内）。
- **主屏上 Chrome 怎么标同样的结构**：【重算】`inline_check.py` 统计"自身零面积、但有正面积后代"的节点——主屏 Chrome 186 次（其中可点击 98 次），**平台全部标为不可见**，包括同一个 regex101 卡片 `[48,1136,672,1136]`；后台屏同样的结构 22 次，平台全部标为可见。Firefox 主屏 1 次，也是不可见。
- 如果把规则改成"自身和所有已采到的后代都没有正面积才算不可见"（子树规则），后台屏 58,957 个翻转里只会保留 22 个（21 个可点击）。

#### 判断
- 默认输出：**有面积的后代不会消失**，主机已经把它们提上来了。会消失的只是零面积父节点自己那一行，以及它的引用和 `clickable` 标记。
- 可操作性：按名字点"regex101 v14.6.2 is here!"会命中可见的文字节点，再解析到零高度的可点击父节点，然后被拒为"目标不可见"。**这在主屏上现在就是如此**，第 13 条只是让后台屏与主屏一致，不是新回归。子树规则能让这种卡片可点，但它是超出第 13 条的改进，而且动作前的新鲜读取拿不到后代，两边规则会不一致（2.2）。建议不做，在指引里写一句"点击被拒为不可见时，试试点它可见的子节点或用 `click --touch`"。
- 截断：零面积父节点缺子节点时，选择器会认为"可见部分读全"。如果缺的恰好是那张卡片的子节点，按名字选择可能把"唯一"判错。主屏上同样存在，频率低。

#### 实施时必须保证的不变量
1. 平台报可见、并且与所在窗口交集面积为正的节点，改后仍然可见。用纯函数保证，测试穷举边界。
2. 默认视图里，一个可见节点不会因为祖先不可见而不被打印，它会被提到祖先的位置。已有行为，补一条回归测试锁住。
3. 采集顺序和 JSON 里的可见位用同一个函数，否则"可见优先"在后台屏上仍不起作用。
4. 快照和动作前新鲜读取用同一规则（2.2）。
5. `find`、`wait` 成功正文和 `--all` 照旧显示不可见祖先并标 `offscreen`。

#### 建议的测试
- 主机：零面积不可见父节点加一个可见有名字的子节点，默认渲染出现子节点行、不出现父节点引用；`--all` 两者都出现且父节点带 `offscreen`。
- 设备：父节点零高度、子节点正面积的两层夹具，采集截断在 N 个节点时，子节点的入队位置可以预期（写成断言，防止以后被改成"按父节点剪枝"）。
- 选择器：可见子节点的 `selectionTarget` 指向不可见父节点时，按名字点击返回 `notReady`（目标不可见），并带一句建议。

#### 必须上设备确认的
- 后台屏和主屏上，同一张"行内链接包卡片"，点卡片文字的结果是否一致。

---

### 2.4 设备端能否区分应用自己发起的停止和其它退出（第 19 条）

#### 现状：谁绑定、怎么知道它死了
- 网关前台服务 `GatewayService` 在 App 进程里创建 `WorkerConnection`（【源码】`GatewayService.kt:135-159`）。后者通过 Shizuku 绑定用户服务 `WorkerService`，设置为 `daemon(true)`（App 进程退出后常驻进程仍可存活），并带 `version(VERSION_CODE)`（`WorkerConnection.kt:50-57`）。
- 连接成功后，App 自己对常驻进程的 binder 注册死亡通知，回调 `lost(candidate, "Worker died")`（`WorkerConnection.kt:82-89`）；Shizuku 客户端另有一条 `onServiceDisconnected`，回调 `lost(null, "Worker disconnected")`（`WorkerConnection.kt:115-117`）。【上游源码】Shizuku-API（GitHub 上 master 分支 `5ab1d6f`，**不是**项目用的 13.1.5 标签，该仓库没有 13.x 标签）的 `ShizukuServiceConnection.died()` 在服务 binder 死亡时，在主线程对每个连接调用 `onServiceDisconnected`；服务端 `UserServiceRecord` 在 binder 死亡时删除记录，**不自动重启**用户服务。
- 两条回调都进入 `lost()`：清空连接、写 `lastFailure`、通知界面（`WorkerConnection.kt:119-136`），**不重绑**。注释写着"Shizuku and Binder failures require explicit user retry"（`WorkerConnection.kt:150`）。
- 已有的自动绑定入口有两个：Shizuku 的 binder 到达（粘性监听，Shizuku 启动或重启时触发）和 Shizuku 授权通过（`WorkerConnection.kt:139-148`）。用户点"刷新状态"时，`refreshWorker` 在常驻进程不活时调用 `bind()`（`GatewayService.kt:199-209`）。网关每 2 秒轮询一次，但只在活着时刷新状态（`GatewayService.kt:33-43`）。

#### 各条退出路径和 App 一侧看到的信号

| 路径 | 代码 | App 一侧的信号 | 现在的状态 |
| --- | --- | --- | --- |
| 管理页"停止空闲执行器" | `GatewayService.kt:288-291` → `WorkerConnection.stopExecutor`（`213-230`）：`prepareStop`（`WorkerService.kt:129-137`，有活动资源时拒绝）→ `destroy()` 让进程 `exitProcess(0)` → 等死亡确认 → 清空连接、`lastFailure = null` | 死亡通知和 `onServiceDisconnected` 都会触发 `lost()` | 若死亡通知先于 `stopExecutor` 的清理块，`lost()` 会短暂写入"Worker died"，随后被清成 null；若后于，`lost()` 因连接已空直接返回 |
| 管理页"强制退出执行器" | `forceExecutor`（`232-241`）→ 只调用 `destroyAndConfirm`，**不清连接** | 同上 | 完全靠 `lost()` 收尾，结果是 `lastFailure = "Worker died"`，**与崩溃无法区分** |
| 关闭网关（停止前台服务） | `GatewayService.onDestroy` → `detach()`（`WorkerConnection.kt:275-289`）先置 `detached`、清 `attempt` | 常驻进程不一定死（以非删除方式解绑，daemon 模式） | `lost()` 因 `attempt !== this` 返回 |
| 空闲自动退出 | **不存在**。全文只有两处 `exitProcess`：系统进程死亡和 `destroy()`（`WorkerService.kt:25, 147`）。"空闲"只让会话租约到期（`Sessions.kt:321-327`），进程照常运行 | — | — |
| 系统进程死亡 | 常驻进程自杀（`WorkerService.kt:25`）；App 进程也随系统进程一起没了 | 无人接收 | 不涉及重绑 |
| Shizuku 被停 | 用户服务进程不一定随之退出（【推断】按上游源码，客户端的死亡连接挂在服务 binder 上，不挂在 Shizuku 服务端上） | 如果服务也死了：`lost()`；Shizuku 再起来时，粘性监听调用 `bind()` | 已有自动绑定 |
| Shizuku 授权被撤 | 未核实 Shizuku 是否会因此结束用户服务 | 如果结束：`lost()` | 重绑会因"Authorize AndroidCtl in Shizuku"失败 |
| App 被更新或强杀 | App 进程消失，网关也消失（`START_NOT_STICKY`，`GatewayService.kt:161-165`）；更新后重新绑定时，Shizuku 发现版本号不同会删掉旧记录（上游 `UserServiceManager` 第 173-186 行） | 无人接收 | 用户重新打开 App 才会绑定 |
| 被系统回收、崩溃 | 进程消失 | 死亡通知和 `onServiceDisconnected` | `lastFailure = "Worker died"`，等人点刷新 |

#### 有没有可靠的"这是我主动停的"标记
**没有。** `stopExecutor` 事后把 `lastFailure` 清成 null 算是一种隐含标记，但它出现在死亡通知之后，存在竞态；`forceExecutor` 连这个都没有。如果直接在 `lost()` 里加"重绑一次"：
- 停止空闲执行器时，死亡通知可能先于清理块，于是触发重绑，违反"应用自己发起的停止不重绑"；
- 强制退出一定会触发重绑。

建议：
- 在 `WorkerConnection` 里加一个"预期退出的目标"字段，记录将要被结束的那个 `IExecutor` 的 binder，在 `stopExecutor` 和 `forceExecutor` 调用 `prepareStop`/`destroy` **之前**写入，持锁访问。
- `lost()` 比较死亡的目标是否就是它：是就不重绑并清掉这个字段，不是（或未记录）才重绑。
- 不要用全局布尔值，否则"停止旧进程后又起了新进程、新进程崩溃"会被误认为主动停止。
- `onServiceDisconnected` 的回调不带目标（`lost(null, ...)`），要用"当前连接的目标"来比较，或者只让死亡通知那一路负责重绑。两路都会进 `lost()`，第二次因连接已空而返回（`WorkerConnection.kt:127`），已经去重。

#### 网关在常驻进程死后是否还在、哪里能重绑
- 网关在另一个进程，常驻进程死了它照常运行。`lost()` 在 binder 线程或主线程执行；`bind()` 自己会投递到主线程（`WorkerConnection.kt:151-152`），可以在 `lost()` 末尾直接调用。
- `bind()` 已经处理了"上一次尝试仍在进行中"（5 秒内不重复，`WorkerConnection.kt:157-160`）和"先解绑旧尝试"（`172`）。
- 重绑的前提：Shizuku 在运行并已授权，网关没有被关（`detached` 为假），常驻进程版本与 App 一致。前两个条件 `bind()` 会检查并写进 `lastFailure`。

#### 重绑之后各类状态（"旧状态全部作废"现在是否已经如此）
- **会话和租约**：在新进程里是空的，`Sessions` 是新对象。主机下一条会话命令会收到 `sessionRequired: No current session; run session open --display main|virtual explicitly`（`Sessions.kt:369-371`），看不出"常驻进程换过了"。
- **引用和图片**：作废。新进程第一次出版观察时附注"New worker: previous refs are invalid"（`UiSnapshot.kt:64-68`）；捕获编号前缀是新的 workerId（`WorkerService.kt:30`），指引里"结果未知后核对捕获前缀"的规则会判为"常驻进程已换，无法确认"（`reference.md:74`）。
- **作业**：设备端作业表在内存里，作废；指引已写"A new worker loses refs/images/device jobs"（`connection.md:97-98`）。
- **配对**：在 App 一侧持久化，连接成功时 `reconcile` 下发允许的配对（`WorkerConnection.kt:78-81`），不受影响。
- **后台屏**：虚拟显示属于死掉的进程，系统会回收；上面的 App 任务被移到主屏还是被结束，【需设备】确认。第五轮决定已把"常驻进程退出时直接销毁后台屏"列为已知局限（`androidctl-r5-decisions.md:25`），与第 19 条不冲突。但自动重绑后，调用者既看不到后台屏没了，也看不到哪些任务被移走，第 12 条的"记下任务"也覆盖不到崩溃这条路。
- **界面清理的封锁**：清理状态在旧进程内存里，新进程没有。系统在登记方进程死亡时会自动注销它的 UiAutomation【推断，基于平台常识，未在本轮源码里核】，所以崩溃后自动重绑会顺带解开"界面清理未确认就封锁"。
- 建议：网关在重绑成功后记一条"常驻进程在某时因某原因意外退出，已自动重新连接"，在下一条命令收到 `sessionRequired` 或 `workerUnavailable` 时附上。否则调用者会把一次崩溃读成"会话过期"。

#### 与 A2 修复和"界面清理未确认就封锁"的门
- A2 修复后，"别的工具占着登记"不再进入封锁，与重绑无关。
- 用户因为封锁提示（`Sessions.kt:399-400`，"ask the user to use Stop idle executor on the device"）去点"停止空闲执行器"，是**应用发起的停止**，按第 19 条不重绑；指引说停止后要"refresh after it ends to attach a new worker"（`connection.md:104`）。也就是说，封锁场景下用户要点两次（停止、刷新）。建议把封锁提示改成"停止空闲执行器后再点刷新状态"，这是实施方能定的文字。
- 空闲自动退出不存在，不需要归类。会话到期只结束租约，不涉及重绑。

#### Shizuku 会不会自己重启用户服务
按上游源码不会（见上）。Shizuku 自身重启时，App 已有的粘性监听会调用 `bind()`；如果此时第 19 条的重绑也排队了，`bind()` 的"5 秒内已有尝试"判断会挡住第二次，不会重复（【源码】`WorkerConnection.kt:155-160`）。

#### "重新绑定一次"的几种读法（只报告后果，见第 3 节 C2）
- 甲：每次意外退出都重绑一次。新进程一起来又崩时，会无限循环：连接、死亡、重绑。如果崩溃发生在服务构造期间（例如某个隐藏接口在新系统上失效），Shizuku 等启动超时后删掉记录，App 一侧只会看到"绑定超时"，不进 `lost()`，不会循环；循环只发生在"连上之后很快死"的情形。
- 乙：每个 App 进程生命周期内总共一次。第二次崩溃起就回到"等人点刷新"。
- 丙：重绑失败就不再试。如果重绑本身成功、新进程随后崩溃，按丙仍会再重绑，等于甲。
- 实施建议（无论选哪种）：只在上一个进程存活超过一个最短时长（例如 60 秒）时才自动重绑，并在用户手动刷新成功后重置计数。

#### 必须上设备确认的
- 对常驻进程执行 `kill -9`（模拟崩溃）：两路回调的先后，重绑耗时，主机下一条命令看到什么。
- 停止空闲执行器和强制退出：确认不触发重绑。
- 常驻进程死亡时后台屏上的 App 任务去了哪里。
- 撤销 Shizuku 授权、停止 Shizuku 时，常驻进程是否退出、App 收到什么。

#### 建议的测试
- `WorkerConnection` 的状态机抽成可注入的纯逻辑后测试：预期退出不重绑；意外退出重绑一次；预期退出之后新进程崩溃仍重绑；`onServiceDisconnected` 与死亡通知先后两种顺序；关闭网关后不重绑；短命进程不重绑。

---

### 2.5 版本号（第 2 条）

#### 现状
- 【源码】`git grep -n '0\.4\.0' fb75afc` 只命中 `VERSION:1`。
- 主机端：`pyproject.toml:7, 60-61` 声明版本从 `VERSION` 动态读取；运行时用 `importlib.metadata.version("androidctl")`（`protocol.py:11`），也就是**安装时**写入的元数据。
- Android：`build.gradle.kts:22-41, 162-163` 从 `../VERSION` 读取，`versionName = VERSION`，`versionCode = 主×1,000,000 + 次×1,000 + 修订`。0.4.0 是 4000，0.5.0 是 5000，可以覆盖升级。
- 常驻进程：Shizuku 用户服务参数带 `version(VERSION_CODE)`（`WorkerConnection.kt:57`），版本号一变，Shizuku 会在下次绑定时结束旧进程（上游 `UserServiceManager` 第 173-186 行）；连接时 App 还会核对常驻进程报的 versionName（`WorkerConnection.kt:75-77`）。
- 主机和 App 握手要求版本完全相等（`transport.py:261, 274`；`local_server.py:64`；`local_call.py:129-134`）。
- 契约样本 `contract/wire-contract.json` 不含版本字符串。测试都从 `VERSION` 或元数据取值（`test_global_options.py:14, 358`；`test_resource_export.py:118, 142, 202`）。`ManagementStateTest.kt:52, 136` 里的 `"0.3.0"` 是解析夹具，与真实版本无关。
- 打包进主机包的 APK 文件名带版本（`app.py:546-548`），由发布工具生成；源码树里 `resources/` 只有 `__init__.py`。
- 发布说明 `tools/release/README.md:113-118` 要求发布时把 README 的相对链接改成指向 `vX.Y.Z` 的绝对地址；那是发布步骤，不是合入步骤。

#### 风险
- **只改 `VERSION` 一行就够**，不需要改其他文件，也不需要重新生成契约样本。
- **可编辑安装的元数据不会自动更新**：在已有的 `.conda` 环境里改了 `VERSION`，`androidctl --version` 和握手仍报 0.4.0，直到重新执行 `pip install -e .`。`Taskfile.yml` 的质量检查不负责重装。最终质量检查之前必须先重装，否则主机测试拿到的版本与新构建的 APK 不一致；纯主机测试仍会通过，因为它们读的是同一份元数据，问题会被掩盖。CI 每次全新安装，不受影响。
- 设备证据和最终头部的关系建议这样写：所有设备证据都在版本串为 0.4.0 的构建上取得；最终头部与被实测的树相比只差 `VERSION` 一行（用 `git diff --stat` 核对）；版本串只影响握手比较、APK 的 versionCode 和 Shizuku 用户服务的版本号，这三处没有在 0.5.0 构建上实测；用户在发布前的真机检查里覆盖安装一次即可补上。

#### 必须上设备确认的
- 0.4.0 已配对的设备覆盖安装 0.5.0：原配对还在；旧常驻进程被替换；旧主机连新 App 报版本不匹配（评审报告 §8 第 4 步已列）。

---

### 2.6 `find` 命中 0 条时自动新采一次（第 11 条）

#### 现状
- `find` 的匹配在**主机端**做：主机把 `find` 改成 `snapshot` 请求，参数是 `cached: true, find: true`（【源码】`ui_client.py:167-169`；`ui_commands.py:691-725`）。设备返回当前样本，主机在 `ui_view.observation_nodes` 里对整棵树（含屏外节点）做匹配（`ui_view.py:939-998`）。
- "上一条动作未稳定就重采"的标记在**设备端** `UiSnapshot.refreshBeforeFind`：动作结束时，设置条件是"结果不是 changed/unchanged，或者动作后的树里有可见的加载指示"（`UiSnapshot.kt:98-113`）；装入新样本时清掉（`UiSnapshot.kt:59`）。设备读请求时，如果是 `find` 且标记为真，就改成用上一份样本的 `selectionScope` 新采一次（`UiSnapshot.kt:91-96`）。
- 回执里有 `snapshotCached`（`UiPublication.kt:153`）；首行在读缓存时打印 `cached; sequence=...; sample=...; age=...`（`ui_output.py:60-68`）。
- `find` **没有**范围参数（没有 `--in`、`--window`）；新采时沿用缓存样本的 `selectionScope`。`find --all` 只影响显示多少命中，不影响采样。
- 没有缓存样本时，设备报 `noSnapshot`（`UiSnapshot.kt:85`）。

#### 落在哪一层最简单
- **主机端，不改协议。** 主机拿到第一次结果后，如果 `snapshotCached` 为真且命中数为 0，就在同一个时限内再发一次 `snapshot`，参数改成 `cached: false, find: true`，`scope` 取第一次返回树里的 `selectionScope`。原始树里带着这个字段，【原件】`r5-samples/*/artifacts.0001.tree.json` 里有 `"selectionScope": {}`。这样不需要新的参数，也不用改契约样本。
- 要改的是 `ui_client.execute` 的结构：现在一次请求就走完取附件、渲染、保存（`ui_client.py:643-675`）。重试必须在保存之前做，否则第一次 0 命中的正文会先存进历史，计数也多一次。
- 时限：两次请求共用命令的截止时间（`deadline = 开始时间 + timeoutMs`，默认 5 秒）。读缓存很快，新采在大页面上要 1.5 到 3 秒。时间不够时，第二次会以时限失败告终。

#### 新采之后
- 新样本装入为当前样本（`UiBackend.kt:397-413` 的 `snapshots.install`），之后的 `snapshot --cached` 读到的是新采的那一份。
- 旧样本里身份不变的节点沿用旧引用（`UiRefs.kt:50-98`），所以重采不会让调用者手里的引用无故失效。

#### 新采失败或不完整时怎么报
- 新采失败（时限、范围不可用）：建议主结果按 `find` 失败报，原因写"缓存样本 0 条，新采失败：…"。不要报 `done` 加 0 条，否则会被读成"确实没有"。
- 新采不完整、仍是 0 条：报 `done`，首行带现有的 `observationWarning=partial`（`ui_output.py:169-178`），命中摘要的 `read` 提示照旧（有可滚动容器时提示用 `scroll-until`，`ui_view.py:984-991`）。指引已写"采样不完整可以证明存在、不能证明不存在"（`SKILL.md:76`）。
- 回执：建议首行加一个短标记，例如 `refreshed after 0 cached matches`；JSON 里加一个布尔字段。否则调用者从首行只能看到"这次没写 cached"，看不出是重试的结果。

#### 指引要改的地方
- `SKILL.md:73-74`（"the next `find` refreshes once in the same scope; otherwise it reuses the current sample"）和 `find --help`（`ui_commands.py:686-690`）都要补上"命中 0 条时自动新采一次"。

#### 必须上设备确认的
- 无硬性要求。可以在第 5 条的那次实测里留意 `find` 0 命中后的行为。

#### 建议的测试
- 主机：读缓存 0 命中后发第二次请求，第二次参数里 `cached=false`、`scope` 等于第一次树的 `selectionScope`；历史目录只多一个文件；首行有重试标记。
- 读缓存已经是新采（`snapshotCached=false`）时不重试。
- 第二次超时：主结果失败。
- 第二次不完整：首行带 `partial`。

---

### 2.7 引用因名字变化被拒时附一行（第 10 条）

#### 现状
- 【原件】路线题里的实际回执是：`click: not sent; No current target matches the original ref signature; code=stale; subject=ref; dispatchStarted=false; timing=before-action`（`ST/efficiency/u35-c1-candidate-01/evidence/cli/1795520494773501400-*/child.stdout`）。
- 【源码】过程如下：
  - 第一次尝试读新鲜节点，`requireOriginalIdentity` 发现身份不同，抛 `changed`，主体是 `selection`（`UiVerification.kt:22-33`）。
  - `UiRefRepair.perform` 判断可以修复，调用 `capture()` 新采一棵树（`UiRefRepair.kt:32-41`）。
  - `capture` 经 `sample` 把新树**装为当前样本**（`UiBackend.kt:373-413`），所以拒绝发生时，新树里的引用已经可以直接用。
  - `select` 在新树里打分：两边名字都不空且不同的节点直接排除（`UiRefRepair.kt:98`），没有候选就抛 `stale`，主体是 `ref`，原因"No current target matches the original ref signature"（`UiRefRepair.kt:72`）。
  - 新树不完整时先抛 `identityUnknown`，原因"Ref repair observation is incomplete"（`UiRefRepair.kt:62`）。
  - 多个候选分数接近时抛 `ambiguous`（`UiRefRepair.kt:74-80`）。
- 回执文字在**设备端**生成（`Rejected` 的消息），主机只是原样打印原因并加上 `code=…`（`ui_output.py:38-56`）。

#### "同位置"可以用哪些已有字段判定
- 最强的是**同一窗口、同一来源节点编号**（`windowId` 加 `sourceId`）：步行模式标签页的名字从"Walking mode: "变成"Walking mode: 18 minutes"，是同一个 View，来源编号不变。新树里 `sourceId` 在同一窗口内唯一时（`UiRefs.kt:62-64` 已经算过唯一性），可以精确找到它。
- 其他可用的字段：角色、行标签、行列链（身份的其余部分）、边界。
- 列表复用时，同一个来源编号可能已经绑定了另一行内容（例如关注按钮被复用到下一行）。这时行标签或行列链会变，**它已经不是"只有名字变了"**，不应附这一行。

建议的判定：只有在新树里存在一个节点满足"窗口相同、来源编号相同、角色相同、行标签和行列链相同、只有名字不同"时，才附这一行。多个满足（不应发生）或没有满足时，不附，照现状报。新树不完整（`identityUnknown`）时照现状，与裁决一致。

#### 字段设计建议（不会被读成"同一个目标"）
- 设备端在拒绝的数据里放结构化事实，例如：
  ```
  data.nameChanged = {
    "originalName": "Walking mode: ",
    "currentRef": "e124390001",
    "currentName": "Walking mode: 18 minutes",
    "basis": "same window and source node"
  }
  ```
  主机首行渲染为：`name changed: was "Walking mode: "; at the same position now e124390001 "Walking mode: 18 minutes"; not sent`。
- 原因码保持 `stale`，主体保持 `ref`，结果 `not sent`、`dispatchStarted=false` 都不变。不要叫 `sameTarget`、`replacement`、`repairedRef` 这类名字，也不要放进 `resolved` 字段，因为 `resolved` 在别处表示"实际执行用的目标"（`ui_output.py:82-86, 479-530`），会被读成已经替换了目标。
- 不自动派发。

#### 必须有的测试
- 名字变、来源编号相同：附一行，给出新引用；该引用在当前样本里可执行；结果 `not sent`，原因码 `stale`。
- 来源编号相同但行标签变了（列表复用）：不附。
- 新树不完整：原因码 `identityUnknown`，不附。
- 名字由空变成非空（`UiRefRepair.kt:98` 不排除这种情况，可能被修复成功）：现有修复行为不变。
- 措辞测试：首行不含"same target""replaced"之类的词。

---

### 2.8 滚动查找选择器不再要求唯一（第 16 条）与截断规则（第 17 条）

#### 现状
- 选择器模式的成功条件：取可见目标（`selectionTarget` 解析后的目标，不是命中节点本身），**恰好一个、树完整、引用可用、身份完整**，才写 `ref`（【源码】`ScrollContainer.kt:87-113`）。
- `ScrollProgress.checked` 的判断顺序是：文字模式有命中即成功；选择器模式要有 `ref`；不完整就暂停（`observationIncomplete`/`observationPending`）；多个命中报 `ambiguous`；都没有报 `notFound`（`ScrollProgress.kt:51-72`）。
- 文字模式已经是"至少一个可见命中即成功、返回全部 `refs`、允许不完整"（`ScrollContainer.kt:26-50`）。
- `wait` 选择器模式：有任一可见**命中节点**即成功，不要求唯一、不要求完整，`refs` 是全部命中节点（`UiCondition.kt:143-159`）。第四轮第 15 项定的就是这样（`androidctl-r4-decisions.md:128-130`）。

#### "与 `wait` 一致"要改的地方
- `ScrollContainer.predicate`：改成与 `inspectText` 同形，返回 `matches`（命中节点数）、`unknown`、`complete`、`refs`（全部可见命中节点的引用），删掉单一 `ref`、`root`、`candidates` 的分支。
- `ScrollProgress.checked`：两种模式统一为"`matches > 0` 即成功"；`ambiguous` 这一类停止原因随之删除；不完整且 0 命中时照旧暂停或继续。
- 命中节点和目标的差别：`wait` 返回的是命中节点；现在的滚动查找返回的是解析后的目标。统一后按 `wait` 返回命中节点。指引里要写明 `refs` 是"观察到的匹配节点"，不保证可操作（文字模式已有这句，`actions.md:74-76`），`actions.md:77-78` 的"Exact selectors still require a complete search with a unique..."要删。
- 主机端：首行用 `match=` 整段打印谓词（`ui_output.py:153-154`）。返回全部 `refs` 后，这一段会随匹配数增长；评审工作记录 67 的 D3 也指出首行引用没有上限。建议文本首行只列前 20 个引用加总数（与 `find` 的默认 20 条一致，`ui_view.py:103`），JSON 保留全部。上界是设备回复的 192 KiB，最多 8,192 个节点，约 115 KB。第四轮第 15 项说"回执里列出全部匹配到的引用"，指的是数据；文本截断而 JSON 保留全部，不违背它。

#### "等待出现不因截断提前失败"
- 提前失败在 `UiWait.observe`：连续两次采样碰到同样的节点或字节上限、并且没有命中，就以上限原因码失败（【源码】`UiCondition.kt:424-431`）。它**对所有条件都生效**，包括 `gone`、`idle`、`screenChange` 和 `--any-of`。
- 按第 17 条，正向条件（文字、选择器、应用、画面变化）删掉这一提前失败；"等待消失"维持严格。`gone` 的成功条件本来就要求完整树（`UiCondition.kt:175`），在截断的树上永远不能成功，所以建议**保留 `gone` 的提前失败**（截断时它注定等不到，提前失败有用）。`--any-of` 里只要有一个正向分支，就不提前失败。`idle` 要求两次完整相同的树，截断时同样注定等不到，建议与 `gone` 同等处理；这一点裁决没提，实施方可以自己定，但要写明。
- 删掉之后，正向等待在截断的树上会等到超时。超时回执建议带：
  - 谓词里的 `limits`（现在只在提前失败时写，`UiCondition.kt:428`）；
  - `complete=false`（文字条件的谓词在 0 命中时是 null，`UiCondition.kt:196-206`，所以超时回执里没有任何截断信息，需要补）；
  - 一句"目标可能在未采集的部分；缩小范围或滚动后再等"。
  - 失败的 `wait` 本来会附整页观察（`ui_client.py:689-692`），首行会带 `observationWarning=partial`。

#### 指引里三处截断例外
- `SKILL.md:77`："Selector uniqueness has the [limited truncation exception](actions.md#selectors-and-scope)."
- `actions.md:30-34`："A sample capped only by `nodeLimit`/`treeByteLimit` can support ordinary unique selection when no visible node is missing children. ... This exception does not relax `gone`, `idle`, screen-change baselines or ref repair."
- `observations.md:87-91`："...[limited truncation](actions.md#selectors-and-scope) is an exception for ordinary action selection."
- 另外，"连续两次截断即停止"写在 `observations.md:161-162`："An unmet wait stops after two consecutive samples hit the same node/byte limit; a known positive match wins first."，随第 17 条改写。

裁决说"删到只剩报错"：三处例外的说明可以删到一句，或者干脆不写，让拒绝回执自己说明（`UiActions.kt:911-918` 的回执已经写了"Observation hit ... before selector uniqueness could be established"）。注意 `actions.md:31-32` 有一句"Visible descendants of invisible parents still participate"，与 2.3 的行为相符，删的时候不要留下相反的说法。

#### 第 13 条与第 17 条的耦合点
1. 采集顺序：后台屏长页的截断之所以连屏内节点都采不全，是因为屏外节点都"可见"，可见优先失效。第 13 条之后，截断只会落在屏外部分，"等待出现"在截断树上的成功率会大幅上升。
2. "可见部分读全"（`UiSelector.kt:96-101`）和动作选择的截断例外依赖可见位：第 13 条之后，截断的长页更容易被判为"可见部分读全"。
3. 后台屏上的"等待出现"：现在会被屏外文字提前满足（假成功）。第 13 条之后不会了，但这些等待会碰到截断；如果此时还有"连续两次截断提前失败"，就会以上限原因码快速失败；第 17 条之后则改为等到超时。
4. 所以先做第 13 条，才能看清第 17 条删掉提前失败之后，正向等待在长页上究竟是"很快等到"还是"一直等到超时"。顺序反过来，长页上的正向等待会从"假成功"直接变成"等满时限"，看起来像回归。

#### 必须上设备确认的
- 第 5 条的"后台屏加长网页"检查里，加一次对屏内文字的 `wait --contains` 和一次对屏外文字的 `wait --contains`，看耗时和结果。

#### 建议的测试
- 滚动查找选择器模式：多个可见命中时成功，返回全部 `refs`；不完整但有命中时成功；首行只列 20 个引用。
- `wait --contains` 在连续截断的树上不提前失败、等到超时，超时回执里有 `limits`；`wait --gone` 在连续截断时仍提前失败；`--any-of` 里同时有 `gone` 和文字分支时不提前失败。

---

### 2.9 其它几条的落点速查

**第 12 条（拒绝回执和 `status` 给剩余秒数；到期时记下后台屏上的任务）**
- 剩余秒数：租约里有 `touched` 和 `idleMs`（默认 30 分钟，`Sessions.kt:393`）。`status` 现在只给已空闲的秒数 `idleSeconds`，有活动资源时为 0（`Sessions.kt:344-357`）。剩余秒数等于 `idleMs` 减去已空闲时长，在 `Sessions` 内持锁计算即可。
- 拦下关闭的拒绝在 `DisplayTasks.kt:125`（原因码 `externalTasks`），拒绝时会把空闲计时重置一次，所以那一刻的剩余秒数约等于整个空闲时长。
- 到期路径：常驻进程的维护线程每秒调用一次 `sessions.expire`（`WorkerService.kt:40, 49-56`）。到期时 `invalidateLease` 在 `Sessions` 的锁内调用 `ui.clear`（`Sessions.kt:321-327`），而 `ui.clear` 把真正的清理丢到另一个执行线程异步去做（`UiBackend.kt:567-578`）。任务查询应放在那个异步清理里、销毁虚拟显示之前，不要放进 `Sessions` 的锁里，否则一次系统调用卡住会拖住所有会话命令。查询失败就记"任务未知"，不能因此不销毁。
- "之后的报错"：同一会话下一条命令收到的是 `sessionRequired`，原因"Session expired after its idle timeout"（`Sessions.kt:369-371`）。记录建议放在常驻进程内存里，挂在已有的 `expired` 集合旁边（会话重新打开时会移除，`Sessions.kt:180, 219`），随 `sessionRequired` 一起报。异步清理可能晚于下一条命令完成，要允许报"任务记录尚未完成"。常驻进程重启后记录丢失（与 2.4 有关）。

**第 18 条（本机连接没关好只给提示）**
- 现在改退出码的地方是 `Result.local_close_failed` 里的 `self.exit = UNCERTAIN_EXIT`（【源码】`protocol.py:80-82`），调用点是 `ui_client.py:661`、`app.py:383`、`job_commands.py:418, 499`。提示文字在 `output.py:50-55`。改法是删掉那一行赋值，保留提示和 `localCloseError` 字段。
- 指引 `reference.md:52` 的退出码 3 说明里提到了"cleanup/persistence"，要相应收窄。
- 模拟器上要确认的是"命令行进程和本机后台服务之间的连接没关好时，本机后台服务会不会继续占着会话"。真实的关闭失败很难自然触发，建议这样近似：发一条长等待（`wait --contains 不存在的文字 --timeout 30s`），在等待中强杀命令行进程，立刻用同一会话发 `status` 和 `snapshot`，记录是否报忙、忙多久；再用 `shell sleep 20` 重复一次。立即可用，第 18 条成立；报忙且持续到原时限，说明本机后台服务会占着，按裁决"确认不了就保持现状"。

**第 21 条（管理页 `singleTask`）**
- 清单里只有一个 Activity，`MainActivity`，没有 `launchMode`、没有 `taskAffinity`（【源码】`AndroidManifest.xml:16-21`）；任务亲和性默认是包名。
- 管理页会在自己的任务里启动"忽略电池优化"和"无障碍设置"两个系统设置页（`MainActivity.kt:178, 186`，没有新任务标志，所以落在 AndroidCtl 的任务里）。
- 前台通知点开管理页用的是 `PendingIntent.getActivity(..., Intent(this, MainActivity::class.java))`（`GatewayService.kt:44-46`）。
- 改成 `singleTask` 后：管理页永远是自己任务的根，不会再被放进别的 App 的任务中间。从桌面或通知再次进入时，叠在上面的设置页会被清掉，管理页收到 `onNewIntent`；`MainActivity` 现在没有重写 `onNewIntent`，也不读启动参数，所以没有逻辑受影响。从 Shizuku 授权界面返回不受影响，授权界面在 Shizuku 自己的任务里【推断】。
- 自身检查读六个字段（`ActivityTasks.kt:39-44`；调用在 `AppInput.kt:79-80`）。改后"AndroidCtl 页面夹在别的 App 任务中间"不再可能，`images.md:139-141` 的例外说明（"Metadata cannot identify an arbitrary middle Activity in another App's task"）可以删掉。代码里六个字段的检查可以保留，不必为了删几行而改；如果要收，至少保留根和顶两个字段。

**第 22 条（粘贴成功但恢复剪贴板失败）**
- 现状：`BorrowedPaste` 在恢复状态为 `failed` 时把主结果改成 `failed`，为 `pending`/`unknown` 时改成 `UNKNOWN`（【源码】`BorrowedPaste.kt:119-134`）。也就是说，粘贴已核对一致、只是恢复失败，整条报失败，退出码 1。
- 指引自相矛盾：
  - `reference.md:52` 说退出码 3 的情形里"A known main result can remain done"；
  - `input.md:61-63` 却说"Done needs confirmed PASTE, matching local effect and completed conditional cleanup"。
  - 评审报告附录一引用的是前一种说法。
- 建议改法：
  - 主结果按粘贴本身：已核对一致就是 `done`。
  - 退出码：恢复 `failed` 用 1（已知失败），恢复 `unknown`/`pending` 用 3（不确定）。
  - 诊断只留一个字段，例如 `restore` 加 `restoreReason`（现在首行已有 `restore=` 和 `restoreReason=`，`ui_output.py:383-409`）。
  - 同时改写 `input.md:61-63`。
- 退出码取值和"恢复未知是否也改"见第 3 节 C4。

**第 15 条（默认文本再瘦一轮，保留采集编号和采样缺口标记）**
- 头部现在是：溯源行 `androidctl capture=... connection=... sequence=... finishedMs=... [snapshotCached ...] [snapshotBeforeAction ...] snapshotAgeMs=...`；`observation {...}`（范围、显示、目标、开始时间、`missing`/`missingSummary`、`partial` 等）；有时还有 `windows-source` 和 `image` 行（【源码】`ui_view.py:1001-1020, 650-732`）。之后是 `screen app=... title=... keyboard=...` 和窗口行。
- 指引里用到的字段：
  - `reference.md:66-76`（结果未知后的资源释放核对）：要用 `capture` 冒号前的常驻进程编号判断"是不是同一个常驻进程"，要求新快照不是缓存（`snapshotCached=false`，文本里本来就省略）。
  - `observations.md:100-106`：用 `data.capture` 和原始树的 `sequence` 对照溯源行。
  - `SKILL.md:76` 和 `observations.md:87-91`：`partial`、`missing`/`missingSummary`、上限原因。
  - `observations.md:64-66`：对头部格式本身的描述。
- 删掉会让规则失去依据的：`capture`（裁决说的"采集编号"必须理解为 `capture`，因为恢复规则靠它的前缀识别常驻进程是否换过；只留 `sequence` 不够）、`sequence`、`partial` 和缺口摘要。可以删的：`connection`、`finishedMs`、`startedMs`、`snapshotAgeMs`（读缓存时首行已经有 `age=`），以及非 `--boxes` 时的窗口几何。删之前要同步改 `observations.md:64-66`。

---

## 3. 需要用户裁决的冲突或歧义

**C1. 显式 `--print` 是否越过输出上限（第 14 条与第 23 条）**

背景：第 14 条说上限"适用于所有默认文本输出（含 `--all`），不新增选项"；第 23 条把 `snapshot --raw` 改成默认只保存。`--print` 是现有选项（`ui_commands.py:201-208`），现在与默认行为相同。问题是：调用者明确写了 `--print`（包括 `--raw --print`），要不要照样只打摘要？

- A（推荐）：显式 `--print` 越过上限，照打全文；默认和 `--all` 受上限约束。
  - 得：不新增选项就有一个"我确实要全文"的出口；`--raw --print` 有意义；"打印与保存同文"的例外只落在"默认"上，符合裁决措辞。
  - 失：调用者若养成总写 `--print` 的习惯，上限就形同虚设。指引里要写"只在需要时使用"。
- B：上限对一切文本打印都生效，`--print` 只表示"打印（受上限约束）"。
  - 得：上下文永远不会被一条命令撑爆。
  - 失：`--raw --print` 永远只出摘要，等于失效，可以考虑直接删掉这个组合；要读全文只能读文件。
- C：只让 `--raw --print` 越过上限，其余显式 `--print` 受约束。
  - 得：原始 JSON 有出口。
  - 失：规则多一个特例。
- D：删掉 `--print`，只留 `--no-print`。
  - 得：选项更少。
  - 失：改动命令界面，违背"不新增选项"的精神；已有写法被拒。

剩余未知：调用者实际会不会滥用 `--print`。本轮 297 次调用里，没有一次因为 `--print` 打出超过 32 KB 的内容。

**C2. "重新绑定一次"的"一次"怎么读（第 19 条）**

背景：见 2.4。新进程一连上就崩的情形下，三种读法行为不同；裁决还说"分不清原因的退出也重绑一次"。

- A：每次意外退出都重绑一次。
  - 得：多次偶发崩溃都能自愈。
  - 失："连上就崩"时无限循环，持续消耗电量、刷系统日志。
- B：每个 App 进程生命周期内总共一次。
  - 得：绝不循环。
  - 失：第二次偶发崩溃起又要人到设备上点刷新。
- C（推荐）：每次意外退出重绑一次，但只在上一个进程存活超过一个最短时长（例如 60 秒）时才这样做；用户手动刷新成功后计数清零。
  - 得：偶发崩溃自愈，崩溃循环自动停止。
  - 失：多一个时长常数，测试要覆盖。
- D：重绑失败就不再试，成功之后的再次崩溃仍然重绑（字面"重绑失败不再试"）。
  - 失：实际上等于 A，同样会循环。

**C3. 输出上限覆盖哪些非界面输出（第 14 条）**

背景：见 2.1 末尾。`shell` 已经有一个 4 KB 的显示上限，而且不自动保存文件；`apps`、`notifications` 的行不是正文，也不保存文件。

- A（推荐）：第 14 条只管界面观察的正文（`snapshot`、`find`、`wait`、动作附带的观察、`scroll-until`），`shell` 保持现有的 4 KB 显示上限和导出命令，其他命令不动。
  - 得：改动集中在一处，"文件里永远是全文"的前提（正文本来就自动保存）成立。
  - 失：`notifications` 列表理论上可以到上百 KB，但实测最大 13 KB。
- B：所有命令的文本输出都套同一个上限，超限时自动保存到文件。
  - 得：规则统一。
  - 失：要给 `shell`、`notifications`、`apps` 新建自动保存，等于新增功能；`shell` 现有的 4 KB 显示上限要重新定。
- C：A 加上 `notifications` 和首行谓词各自的条数截断（JSON 不截）。
  - 得：覆盖实际可能变大的两处。
  - 失：多两处小规则。

**C4. 粘贴成功但恢复剪贴板失败时的退出码，以及"恢复未知"是否一起改（第 22 条）**

背景：见 2.9 第 22 条。指引两处说法相反；裁决说"改成与指引一致"，但没有说跟哪一处一致，也只提到了"恢复失败"。

- A（推荐）：主结果 `done`；恢复失败时退出码 1，恢复未知时退出码 3；同步改写 `input.md:61-63`。
  - 得：与 `reference.md:52` 的退出码表一致；首行写着 `done`，调用者不会重打；退出码仍提醒剪贴板没复原。
  - 失：外层的执行框架如果只看退出码，可能把 1 当成"失败、重试"。指引已写"已完成的动作不要因为非零退出而重做"（`SKILL.md:100`）。
- B：主结果 `done`，退出码 0，只在首行和 JSON 里留 `restore=failed`。
  - 得：最不会引起重试。
  - 失：剪贴板没复原这件事只剩一个字段，容易被忽略；与"清理问题用退出码表示"不一致。
- C：只改"恢复失败"，"恢复未知"仍让主结果为 `UNKNOWN`。
  - 得：严格按裁决字面。
  - 失：两种相近情形规则不同。

**C5. 第 13 条在主屏上的波及面**

背景：见 2.2。裁决的起因是后台屏上的 Chrome，但设备端改动对所有显示都生效。按"与所在窗口（或显示区域）交集为零"，主屏上另有约 97 个"整个在屏外、面积为正、平台报可见"的节点会改标为不可见，多为 Activity 转场中途的帧，还有一个完全在屏外的开关。按"自身边界面积为零"，主屏上只影响 2 个无名节点，后台屏 Chrome 的效果完全相同。

- A（推荐）：用"与所在窗口交集为零"，接受主屏上的这部分变化。
  - 得：符合裁决字面；转场中途的屏外节点不会被按名字选中；分屏时也正确。
  - 失：主屏上少数"在屏外但平台报可见"的节点要先显示出来才能操作。
- B：用"自身边界面积为零"。
  - 得：主屏几乎零变化，后台屏问题同样解决。
  - 失：将来若有 App 在后台屏上报"屏外、正面积、可见"的节点，不会被纠正。
- C：只对后台屏生效。
  - 得：主屏完全不变。
  - 失：两套规则，测试和指引都要分开写；与"一个产品一种可见含义"相悖。

---

## 4. 没能核实的内容

- 所有 Android 侧行为都只来自源码，没有设备、没有编译。特别是：常驻进程死亡时后台屏上任务的去向；Shizuku 授权被撤或 Shizuku 停止时用户服务是否退出；两路死亡回调在真机上的先后；`singleTask` 在从通知进入时的实际表现。
- Shizuku 行为依据的是 GitHub 上 `Shizuku-API` 的 master（`5ab1d6f`），不是项目使用的 13.1.5；该仓库没有 13.x 标签，没能按版本核对。
- "系统在登记方进程死亡时自动注销 UiAutomation"是平台常识，本轮没有对照 AOSP 源码核实。
- 可见位统计的样本面：601 份树来自多个历史版本和两种模拟器，主屏覆盖约 30 个 App，后台屏只有 Chrome、通讯录、Termux 和一个测试 App。没有真机，没有厂商系统，没有 Compose 重度 App 的大样本。"平台在主屏把零面积节点标为不可见"的结论只在 Chrome 和 Firefox 上看到。
- 第 18 条"强杀命令行进程"的近似实验是否等价于"本机连接关闭未确认"，没有验证。
- `apps` 命令在装了大量 App 的真机上的输出体量没有原件。
