# 专题 30 核查报告：结果与退出码、阅读视图与保存、差异、区域截图、等待

核查对象：实施方第三轮（r3）整体建议的主线第 6、7、8 条，第 9 题（新 region 坐标契约）、第 10 题（Result 与退出码），建议提交 7、14、15、16，完整处置底表 G4、G5 前半、§6 的 C16（窗口缺席运行验证）与 A1（真实效率验证）。

文中路径缩写：`SRC` = 候选源码只读副本（HEAD 3f45af6），`ROOT`、`T`、`P`、`R` 同 COMMON.md。我写的离线脚本与输出都在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents6/30/`。全程没有设备，下面所有运行结论都来自实施方在模拟器上留下的原件。

---

## 1. 结论摘要

1. **第 10 题（Result 与退出码）：改选一个选项之外的简单规则。** 规则是：Result 只说明主动作是否发出、是否核实，交付失败或清理失败都不改写它；退出码汇总整条命令，只有"主动作 DONE、请求的交付物全部交付、清理已确认"时才是 0。listener 注销失败仍只作诊断，按用户 U27 的原话保留这个例外。
   - A 的内部矛盾：同一步"动作 DONE、请求的截图没保存成"，单独执行时 exit 0，放在 batch 里却停止并返回 exit 1。stdout 写失败也一样，单独执行 0，batch 里 1。源码现状就是这样，A 只是把它写成了合同。
   - B 把 Result 改写成 UNKNOWN 等整命令状态，会诱导调用者重发已经完成的动作。
   - 简单规则只处理不了一种情形：已经打印出去的字节后面又出错。这一点 A、B 同样无解。
   - 这题要问用户，但应改成两选一来问：简单规则（推荐）还是 A。
2. **退出码类别**：0/1/2/3/4 五类都有用处，不需要增加。该删的是"done 配 exit 1"这类重载（virtual launch 落点未确认、通知清理未确认），以及 batch 里另外一套交付停止判断。
3. **单一 View 与 `--diff-from`：同意方向。** 修正两点：
   - 证据：`P/r2-delivery/probe.py` 验证的是**已被放弃的 B 方案**（版本化文本按行 diff，8 个离线例子）。推荐的 C 方案（结构记录往返）在包里**没有留下原件**。我用候选源码复现成功：真实 Contacts 35 条记录、Chrome 页 99 条记录，往返后渲染逐字相等，`changes` 为空。这仍然只是离线投影。
   - 格式：保存文件应该一行一条记录（JSONL）。否则实施方自己验证过的"rg 定向续读"（425 token）会失效。按我的重算，记录 JSON 比正文多 15–30% 字节。
4. **封套字段**：diff 正确性真正必需的只有格式版本、View 选项、partial 与截断标记、记录本身。来源/connection/target/scope 防的是"标签误导"，不是重建出错；直接复用现有 `comparison_key` 即可，不必新增字段。"diff 更长就给全文""不匹配就给全文加原因"都合理，与评审方上一版 §2.10 一致。
5. **紧凑正文与 find：方向成立，但这是三轮下来最大的未兑现项。**
   - 用户规则下"可从已打印状态推出"的动作只剩无标签滚动；收益几乎全部来自"短名代替动作对象"，上一轮原型本来就是这么做的，所以离线降幅（Contacts −39%，Chrome −53% token）基本保得住。
   - 本轮**没有任何**新的真实任务测量。
   - 原型的 Chrome 输出里动作数组仍占 42.9%，主要是 show-on-screen、无障碍焦点、context-click、HTML 和移动粒度。可以把"本页通用动作集"在头行打印一次，节点只列差异。这在字面上满足用户规则；离线重算 Chrome 首屏再减约 28%。
6. **第 9 题（region 坐标）：这题不该问用户，选 A**（用完整交付图的坐标，与 tap 坐标和现有 zoom 同一单位；WPS 真实用例用的就是这个单位）。建议做两处简化：
   - 只读裁片所需的图尺寸 metadata 留到下一张完整图或租约结束，不随输入资格一起失效。
   - REF 裁片改用既有的 Fresh 身份准入取当前边界，删掉"同域观察签名比较"。
   - 旧同帧 zoom 的真实损失在本轮证据里是零：WPS 是静态 PDF，zoom 时距原图已经 30.5 秒，原 JPEG 本身也能读出那句规则。
7. **窗口缺席（提交 7）：同意。** 原件与 U27 第 3 条逐项对应（1 个 App、API34、1 次）。同 ID 窗口回来没测，已如实标注。窗口被临时遮挡（而不是真正关闭）的情形也没测。原型在 host 里按命令和谓词做了一处特判，正式实现应改成通用的附件状态。
8. **提交 15、16 排在最后不合理。** 它们不依赖稀疏 canonical，也不依赖名字规则；第 10 题的合同更应该放到最前，否则后面 13 个提交都按旧合同写结果。建议的阶段顺序见 §2.6。

---

## 2. 逐题分析

### 2.1 第 10 题：Result × 交付 × 清理 → 退出码

#### 2.1.1 现状矩阵（候选 3f45af6，源码核对）

基础映射 done→0、failed→1、not sent→2、UNKNOWN→3、still running→4，允许用显式 `exit` 覆盖。【源码】`androidctl/src/androidctl/protocol.py:45-64`

**单个 UI 命令**（`ui_commands.py:30-42` 以 `result.exit_code()` 退出）：

| 主 Result | 交付/保存 | stdout | 清理 | 输出 | exit | 依据 |
|---|---|---|---|---|---|---|
| done | 全成功 | 成功 | 正常 | done | 0 | 【源码】 |
| done（动作命令） | 请求的图/树下载或保存失败 | 成功 | 正常 | done + imageError/snapshotError | **0** | `ui_client.py:1387-1404` 只改写 snapshot/find/screenshot/zoom；batch.md:49-50 也举了这个例子 |
| done（snapshot/find） | 树没交付或文件没保存 | — | — | 改写为 failed；正文**不打印** | 1 | `ui_client.py:1387-1399`、`1421-1425`（打印要求已有 snapshotPath） |
| done（screenshot/zoom） | 图没保存 | — | — | failed | 1 | `ui_client.py:1400-1404` |
| done（wait/locate 等只读命令） | 树失败 | — | — | done + snapshotError | **0** | 同上，只读命令里只有四个会被改写 |
| done | 成功 | **写失败** | 正常 | data.deliveryError + stderr 一行 | **0** | `output.py:96-121`（注释写明不替换退出码） |
| done | diff 基线/历史错误 | — | — | diffBaselineError/historyError | 0 | `ui_client.py:1101-1118`；observations.md:113 |
| done（virtual launch） | — | — | — | landing 未确认 | **1** | `ActivityLaunch.kt:254-258`（落点确认后才删掉 exit） |
| 通知读/写任一结果 | — | — | listener 注销未确认 | cleanup=unconfirmed | 原为 0 时改 **1** | `NotificationOperations.kt:111`；notifications.md:33-35 |
| shell/文件任务 | — | — | 清理不确定 | 保持原 Result | 3 | `ShellProtocol.kt:145-166`；`file_jobs.py:135-136`；reference.md:68 |
| UI 目标建立失败 | — | — | 设备 stream close 失败 | 保留原结果 + cleanupError | **不改** | `TargetManagement.kt:86-92` |
| host 本地配置写入（已发布后同步失败） | — | — | — | done | 3 | `coordinator.py:57-58,390,678,749` |
| 任一结果 | — | — | **host 本地连接 close 抛异常** | 原 Result 丢失 | 2（Fault→not sent）或进程异常 | 见 §3 缺陷 D1 |

**batch**（`ui_batch.py:292-389`）：

| 情形 | 是否停止 | 外层 Result | exit |
|---|---|---|---|
| 某步非 done 或 exit≠0 | 停 | failed，原因"Step N stopped" | 该步 exit（为 0 时取 1） |
| 某步 done/0，但请求的、停止步或末步附件有 snapshotError/windowsError/imageError | 停 | failed | **1** |
| 中间步未请求附件 | 不下载，不判 | — | — |
| 文本模式 stdout 失败 | 停 | failed | 1 |
| 全部完成 | — | done | 0 |
| 全部完成后 batch 本地连接 close 失败 | — | failed + cleanupError | **1**（不是 3） |
| 冻结显示种类的预检失败 | 不执行 | not sent | 2 |
| JSON 最终打印失败 | — | 改为 failed | 1 |

#### 2.1.2 r3 推荐 A 的合同矩阵（按主稿第 10 题原文整理）

| 情形 | 单动作 | batch |
|---|---|---|
| 动作 DONE、附属观察/截图/保存失败 | DONE / **0**（不加 exit 1） | 该步 DONE；batch 停止，外层 FAILED / **1** |
| snapshot/find/screenshot/region 主文件失败 | FAILED / 1，已取得的正文仍打印 | 停止，FAILED / 1 |
| 主文件成功、stdout 晚失败 | 保持主结果 + 输出诊断（exit 不变，即 **0**） | 停止，**1** |
| 普通资源清理失败 | 3（即使动作 DONE 或更早已失败） | 3 优先 |
| listener 注销失败 | 只作诊断，exit 不变 | 不停批 |
| 非 DONE 或非 0 | 原样 | 停止，exit 取停止类别 |

#### 2.1.3 A 内部不一致的地方

- **核心矛盾。** "动作 DONE、请求的截图没保存成"在单独执行时是 0，在 batch 里是停止加 1。【源码】与 A 合同一致：现状如此，A 也把它定为合同。调用者要记两条规则；只看退出码的调用者，在单步时会误以为截图已到手（虽然拿不到 imagePath 时多半会发现）。
- **stdout 失败同样分裂。** 单步 0，batch 1。
- **原则不一致。** A 让清理失败改 exit（3），却不让请求的交付失败改 exit（单步）；batch 又让交付失败改 exit。三条规则没有一个共同原则可以解释。
- **"普通清理 exit 3"在现状里并不成立。** 至少有两处不是 3：UI 目标建立失败的清理（`TargetManagement.kt:86-92`，exit 不变）；batch 连接 close（`ui_batch.py:383-385`，exit 1）。A 是迁移，不是"保持现契约"，提交里要列出这些迁移点。

#### 2.1.4 选项之外的简单规则

> Result 只描述主动作是否发出/是否核实，永不因交付或清理失败改写。exit 是整条命令的汇总：0 当且仅当主动作 DONE、请求的交付物全部交付、清理已确认；否则非零，并在输出写明是哪一项。listener 注销失败按用户例外只作诊断，不计入。

| 情形 | 简单规则 | 与 A 的差别 | 与 B 的差别 |
|---|---|---|---|
| 动作 DONE + 请求的截图失败 | DONE / 非零（建议 1） | A 单步为 0 | B 把外层 Result 改成 failed/UNKNOWN |
| 同上，在 batch 里 | 该步 exit 非零 → 停止；外层 exit = 该步 exit | 与 A 的 batch 相同，**单步与 batch 一致** | — |
| stdout 失败 | 非零 | A 单步为 0 | — |
| 动作 DONE + 普通清理未确认 | DONE / 3 | 与 A 相同 | B 为外层 UNKNOWN / 3，Result 被改写 |
| snapshot 正文打印成功、文件保存失败 | 非零（1）；Result 是 done 还是 failed 属实施细节，建议保留"文件是主产物" | 与 A 相同 | — |
| batch 停止判断 | 只看"该步 exit≠0" | 去掉 `ui_batch.py` 里另一套 delivery_error 判断 | — |

处理不了的情形：

- **已经写出的字节无法修改。** 先打印了 JSON 或结果行，之后才失败（例如历史锁关闭失败、stdout 中途断开），已输出的 exit 字段或结果行与进程退出码会不一致。A、B 也都解决不了。对策是把"进程退出码为准"写进指引，并把打印之后的工作压到最少：改用 diff-from 后不再有"打印之后提交基线"这一步，只剩锁关闭。
- **进程被杀或主机崩溃**，没有退出码。三案相同。
- **异步晚到的设备清理**（如 listener）。按用户例外，本来就不计入。

**对 LLM 调用者哪个最不容易出错？**

调用者最常见的误判有两种：一是"以为没发出而重发"，二是"以为要的东西都拿到了"。

- 简单规则让 Result 回答第一个问题，exit 回答第二个问题，两个维度各管一件事，在单步和 batch 里含义一致。
- A 在单步里让 exit 0 也可能意味着"截图缺失"，第二个问题回答不清。
- B 让 Result 混合了两件事；外层 UNKNOWN 会被 SKILL 的 UNKNOWN 检查清单触发"确认原操作是否结束"，实际上动作已经 DONE，容易导致多余确认甚至重发。

推荐简单规则，指引写一句："exit≠0 时先读 Result；Result=done 表示动作已发生，不要重做，只补缺失的交付。"

【推断】没有任何真实调用者在这种失败分支下的行为数据；三案都只是设计判断。

#### 2.1.5 退出码类别

现在共五类：

- 0 成功；
- 1 已知失败（可能有部分效果）；
- 2 未发出（修正后可以安全重试）；
- 3 不确定，不要自动重复（UNKNOWN，或清理/同步不确定）；
- 4 仍在运行（轮询）。

每一类对应的下一步动作都不同，不需要增删。【源码】reference.md:58-70

要消除的是重载：

- "done + 1"出现在 virtual launch 落点和通知清理两处。前者由第 7 题与落点规则处理；后者按用户 U27 改为只作诊断。
- batch 连接清理用了 1。按简单规则，清理不确定应为 3。

---

### 2.2 单一 View 与 `--diff-from PATH`

**实施方主张**：打印、保存、diff 共用一份最终 View；保存成结构 JSON 封套；显式 PATH 比较；不匹配或截断时给全文加原因；diff 更长时给全文。

**核到的事实：**

- `P/r2-delivery/probe.py` 与 `results.json` 证明的是**方案 B**（每个文件首行放版本化 metadata，后接同一份文本正文，用 `difflib` 做行 diff）。8 个例子：相同、改一处显示值、产品截断、物理截掉 100B、不同 session、缺失、View 扩大、格式不符。全部是离线合成或改写自一份 Contacts 真实树。
  - 【原件】`P/r2-delivery/probe.py:10,117-174`；`results.json` 的 `diffFromViewOption`
  - 输入 hash `fdfbf50c…`，与 `stages/28-current-editor/runtime-api34/oracles/contacts-seed.raw.json` 一致【重算 sha256】
  - 另有动作命名 8 例和坐标算术 1 例。
- 推荐的方案 C（"35/98 条记录往返后渲染逐字相等、changes 为空"）在 `R/r2-delivery.md` §3.1 写明"不写新文件"，包内**没有脚本也没有结果**。
- **我的复现**【重算】：脚本 `agents6/30/roundtrip.py`，输出 `agents6/30/roundtrip-out.json`。用 `SRC` 的 `ui_view.observation_nodes→screen→ui_diff.records`：

| 输入 | 记录数 | 只用 parent 重建后渲染 | changes 为空 | 正文字节 | 记录 JSON 字节 |
|---|---|---|---|---|---|
| API34 Contacts seed | 35 | 逐字相等 | 是 | 7,968 | 10,370（+30%） |
| MuMu Chrome 4226 节点页（`collector/runtime-mumu-clone/preparation-page.raw.json`） | 99 | 逐字相等 | 是 | 43,837 | 50,187（+14%） |

- 无 key 的记录只有 screen/window/events/event/artifact 这些行，所有节点都有 key。所以 `changes` 不会漏掉节点变化。窗口和事件行的变化不进入结构 diff，只能整体重印；这一点需要写进文案。
- 实施方报的 Chrome 是 98 条，我用的树可能不是同一份，差 1 条（多一条 artifact 行），不影响结论。
- 以上全部是离线投影：没有 CLI 迁移，没有预算或省略，没有 batch，也没有真实任务成本。

**封套字段逐项判断**（【推断】基于 `ui_diff.py:27-148` 的源码）：

| 字段 | 对 diff 正确性 | 判断 |
|---|---|---|
| format/schema 版本 | 必需（解析与投影语义） | 保留 |
| View 选项（depth/query/interactive/all/maxChars…） | 必需。不同 View 下的"added"会把"原来没显示"误报成"新出现" | 保留 |
| partial / 产品截断（omission）事实 | 必需。否则省略的尾部会被报成新增 | 保留 |
| records | 必需 | 保留 |
| 基线采样时点/序号 | 不影响重建，但读者需要知道"相对哪次" | 保留（评审方 §2.10 也要求） |
| device/session/connection/target/scope/selectionScope | 不影响"基线+delta=当前"的重建；只防跨上下文的语义误导。跨设备的 diff 通常因 diffLonger 自动退回全文 | 直接复用现有 `comparison_key`（`ui_diff.py:27-55`），零新增维护；**不要再加字段或多级 reason** |
| bodyUtf8Bytes | C 以 JSON 解析失败代替，已删 | 同意删除 |

"diff 更长给全文"沿用现有 diffLonger；"缺失、坏格式、回收、截断、来源或 View 不匹配都给全文加原因"，与评审方 §2.10 的四条实现要点和三条验收一致，**合理**。

**补充建议（实施细节）：** 保存文件用 JSON Lines，第一行放封套 metadata，其后每行一条记录，大致是 `role {attrs}` 的 JSON 形式。理由：

1. 实施方实测过定向续读：在保存文件里 rg 读 Title，只用 425 token【原件】`P/presentation/README.md`"Title的真正续读"。单行紧凑 JSON 会让 rg 返回整个文件，这条恢复路径就失效了。
2. 记录 JSON 比正文大 14–30%，按行存不会更差。

**长期维护**：一个文件 schema 与读取器，加上 `comparison_key` 的输入源从 history 基线改为文件。删除的是基线预留、提交和 batch 收尾（`history.py` 的 BASELINE、`ui_batch.finish` 中的 candidate/protected、`ui_client.commit_baseline`）。这些删除是确定的净简化。

---

### 2.3 紧凑正文与 `find`

**用户修订**：只省略确实可从已打印状态推出的动作，长按存在就列。

**核到的事实：**

- 按候选的打印属性（不打印 `clickable`），能从已打印状态推出的只有带精确方向的无标签 scroll/page 动作。click 不能省，除非另外打印 `clickable:true`。【源码】`R/r2-delivery.md` §2.1 引 `ui_view.py:109-213`；我核到 `ui_view.py:53-86` 的动作集合定义，结论一致。
- 上一轮原型（`P/presentation`）**本来就没有省略任何动作**，只是把标准动作改成短名，无标签的保留 `{"id":N}`。【原件】`P/presentation/README.md`"原型的实际范围"第 1 条。所以用户规则下的方案 ≈ 原型 + 滚动方向省略 + NAME 入口，原型的主要收益基本保留：

| 原件集合 | 原 tokens | 原型 tokens | 降幅 |
|---|---:|---:|---:|
| Contacts 全部 16 条 | 44,077 | 26,793 | −39.2% |
| Chrome 全部 18 条 | 54,887 | 26,033 | −52.6% |

  【原件】`P/presentation/README.md` 表；`results/reply-fix/measurement.json`（我没有重算 token）。
- 两对真实运行（各 1 次，不同 App/镜像，路线不同）：

| 臂 | 总 input | 相对基线 |
|---|---:|---:|
| Contacts 原版 | 1,395,045 | 1.72 |
| Contacts 原型 | 629,770 | 0.77 |
| Chrome 原版 | 2,326,407 | 2.23 |
| Chrome 原型 | 1,821,534 | 1.75 |

  - 我重算的比值：1.716 / 0.775 / 2.235 / 1.750。
  - 但 Chrome 原型的 uncached input **多 46,679**，工具交付**多 25,374 B**，CLI 次数多 5 次。主因是默认 `find` 零匹配后滚了 4 屏。
  - 【原件】`P/presentation/runtime/comparison-chrome.md`、`comparison-contacts.md`；基线数在 `R/delivery-cost.md` §1。
- **本轮没有任何新测量。** 底表 A1 自己写着"本次无新最终呈现任务"。【原件】`P/r2-delivery/integration-draft.md:305`
- 原型 Chrome 首屏 stdout 里动作数组仍占 **42.9%**（8,342/19,432 B）。最多的是 show-on-screen（82）、`{"id":64}`（81）、context-click（76）、HTML 前后（各 76）、移动粒度（各 50）。【重算】`agents6/30/actions_share.py`。用户规则下，这些都不能"从节点状态推出"。

**选项之外的补充（建议作为实施细节，必要时向用户确认一次文案）：** 在 View 头行打印一次"本页通用动作集"，节点只列与通用集的差异（多出的照列，缺少的写成 `-name`）。动作因此仍可从已打印状态推出，字面满足用户规则，长按照样会出现在某处。【重算】`agents6/30/factor_actions.py`，对原型输出离线估算：

| 原型输出 | 原字节 | 估算后字节 | 降幅 |
|---|---:|---:|---:|
| Chrome 首屏 | 19,432 | ~13,889 | ~−28% |
| Chrome 第 3 条 | 4,497 | ~3,553 | — |
| Contacts fill | 3,623 | ~3,206 | ~−12% |

这是确定性变换的字节估算，不是 token，也不是任务收益。

**find**：Chrome 原型臂的额外成本里，可以逐条指认的最大一项就是默认 `find` 只搜可见投影（【源码】`ui_view.py:253-293`；评审方 F2）。改成默认搜全部已采节点、离屏命中直接给 REF，是本专题证据最直接的一条改进。

**判断**：方向成立，离线收益保得住；真实任务上的净效果仍然**未兑现**。三轮下来，这是本专题证据和产出之间差距最大的一项：离线 −40~−53%，真实只有 2 次单例，其中一次交叉。冻结 View、find、NAME 和首页指引后，用同一后端各跑一对（即 A1），是下一步最有判别力的验证。

---

### 2.4 第 9 题：region 坐标

**核到的事实：**

- 现有 zoom 与坐标输入都用"完整交付图的像素"作单位。
  - 【原件/文档】`SRC/skills/androidctl/images.md:62-66`（zoom 矩形用完整交付图像素）、`:174-180`（tap/swipe 用交付图像素，AndroidCtl 负责映射）
- WPS 真实用例（MuMu，原生 1080×1920，JPEG 720×1280）：039 号命令 `zoom --region 65 1015 670 1100` 就是用 JPEG 坐标圈的，原生区域 `[97,1522,1005,1650]`。
  - 裁片 `producedMs=711196`，比源图 `at=680697` **晚 30.5 秒**。
  - 【原件】`P/exploration-20261119/mumu/commands/039-wps-native-duplicate-rule/stdout.bin`、`command.json`
- 我亲眼看了两张图：原 JPEG（`wps-evidence/000023-…jpg`）里"Note that when duplicate Dictionary keys are encountered, all but the last instance are ignored." 已经清晰可读；PNG 裁片（`000024-…png`）只是更清楚。内容是静态 PDF 页。
- 所以本例**既没有用到同帧能力，也没有证明 zoom 是读懂所必需**。【原件】这与实施方自己在 r3 表中"旧 zoom 的成功不能证明新 region 已保住同帧能力"的表述一致，但更进一步：本例根本不依赖同帧。

**A 与 B 的比较：**

- **A**：调用者看缩小的 JPEG 直接圈，单位与 tap 一致，一套坐标规则。代价是需要知道"这组坐标属于哪张图"，也就是图的尺寸 metadata。
- **B**：调用者要自己乘缩放比；CLI 会出现第二种坐标单位，并且与 tap 不同。对"看缩小 JPEG 圈一块放大看小字"这个主要用法，B 明显更容易出错。
- **C（两种单位都给）**：长期复杂度更高，没有消费者。
- **这题不该问用户**：选 A 是与已发布单位保持一致的实施决定。

**A 额外引入的步骤和状态**：r3 让 metadata 跟着动作、新图、观察变化、租约结束一起失效，缺失时要求先补一张完整图。只读裁片唯一需要的是"该图的交付尺寸 ↔ 当前原生尺寸"的映射，外加当前几何（旋转、尺寸、显示）可比。

**建议**：图尺寸 metadata（不含 bitmap，约百字节）留到下一张完整图或租约结束；**与输入资格的失效脱钩**。几何不可比时拒绝。这样"截图 → snapshot 发现动画变化 → region"不会被无谓拒绝，状态也不比 r3 多（r3 本来就保留 metadata）。【推断】

**REF 裁片的"比较同域已观察签名"**：意思是要求一份与 REF 来源同 connection/target/scope 的当前观察，比较整份观察签名，再核原 REF 的身份与边界。【原件】`integration-draft.md` G4 末两段。

- 对只读截图来说，用旧边界裁错的后果只是"看到一块不对的图"，调用者看图就能发现。
- 整份签名比较需要新采一次树（Chrome 页可能要数秒），还会因为任何无关变化（时钟、动画）误拒。
- 已有的 Fresh 身份准入（节点动作本来就用）就能给出"这个节点现在在哪"。
- **建议删掉签名比较**：REF 裁片 = Fresh 取当前身份和边界，再从新截图裁出；身份失败就拒绝；回执标注树和图分属两个时点。评审方上一版 §2.11 的"签名不一致即拒绝"在只读场景下要求过严，应撤回。【推断】

**旧同帧 zoom 被替换后的真实损失**：瞬时内容（toast、倒计时、视频帧）的事后细读，以及同一时刻多块区域的交叉核对。三轮真实流程里没有找到依赖它的业务；WPS 不是这样的例子（见上）。损失真实存在，但目前没有消费者，按当前方向接受。【原件 + 推断】

---

### 2.5 窗口缺席与等待（建议提交 7、C16）

**我核的原件**：`P/r2-window-absence/runtime-api34/commands/contacts-back-wait-old-ref/stdout.bin`（4,870 B）与 `command.json`（exitCode 2）。我解析 JSON 后逐字段核对：

| 步骤 | 实际 |
|---|---|
| 1 `back --settle 0` | done，samples=0，snapshotSequence=5，cached 且为动作前样本 |
| 2 `wait --gone … --window 225` | done；predicate：basis=windowAbsent、windowId=225、complete=false；observations 为 seq7（窗口 221、216）、seq8（221、216、226），相隔约 82 ms；reason 写明"absent from consecutive window directories; node query was not started"；samples=3 |
| 3 `focus e592373416 --window 225` | not sent，staleRef，dispatchStarted=false，samples=0 |
| 外层 | failed / exit 2 |

另一次单独 CLI 用 selector 访问同一 W：not sent / scopeUnknown，samples=2。【原件】`result.md` 及 `commands/contacts-absent-window-selector`（我只核了 result.md 的转述，没有逐字节读这份 stdout）。

**与 U27 第 3 条的对应**（原话见 `T/decisions.md:38`：等消失可因连续两次窗口缺席完成并续 batch；注明当前观察不到/窗口缺席；两次仅防抖；无固定成功后复采；后续指定窗/引用的自身准入须核而非假定）：

- 完成并续批：是，第 3 步实际执行了。
- 注明依据：是，basis、complete=false、reason 都在。
- 两次防抖、无固定复采：是，第 3 步 samples=0。
- 后继引用自身准入：是，REF 与 selector 各自被拒，且零派发。

结论：**完全对应。** 样本量：1 个 App（Contacts）、API34 模拟器、1 次。

**遗漏，已如实标注的：**

- 同 ID 窗口回来没测。`result.md`"后继 selector"段、`integration-draft.md:304`、r3 主稿表格（"未测同 ID 回归"）都写了。
- seq6 的内容没有旁录，也写了。

**遗漏，未专门标注的：**

- 本次缺席是 Back 真正关闭了 editor，**不是**遮挡造成的暂时缺席。两次防抖存在的理由正是遮挡（评审方 §2.13 依据 AOSP 14 `AccessibilityController`）。被权限框等盖住超过约 80 ms 时，gone 会成立。按用户规则这是"当前观察不到"并已注明，所以不算缺陷；但没有真实遮挡样本。
- 评审方 §2.13 还要求"等出现时 W 被暂时盖住再露出 → 成功；一直缺席 → 超时并写明窗口缺席"。U27 没有要求，r3 只写了"正向条件不回归"，验收不够具体。

**实现上的问题**【原件】`window-absence.patch` 的 ui_client 部分：

- 原型在 host 的 `attachments()` 里按 `command=="wait"`、`predicate.basis=="windowAbsent"`、observations 数为 2、tree 状态等于 `{"unavailable","scopeUnknown"}` 跳过树的下载。
- 原因是现行 batch 规则把"只读命令请求的树不可得"当成交付失败而停批（`ui_batch.py:191-206,356-366`）。这是 §2.1 那套交付停止规则的一个具体代价。
- 正式实现应改为：设备把该树发布为通用的"不适用/未采"状态，host 统一跳过，不在 host 里解析某个命令的谓词。

---

### 2.6 提交 15、16 排在最后是否合理，以及阶段顺序

**依赖核查：**

- **单一 View、diff-from、compact、find**：都是 host 投影层（`ui_view.ProjectionNode` 与 `ui_diff.records`），不读 canonical 树的字段（【源码】`ui_diff.py:72-148`）。稀疏 canonical（提交 4）改的是 wire 和读取默认值，View 层只需跟着改读取函数，不用推倒重来。
- **名字规则**（提交 6/8）只改变节点行的内容；compact 规则、NAME 入口、diff 机制都与"名字怎么来"无关。之后改名字规则，旧保存文件会因 format/View 版本不同而退回全文，这本来就是设计内的行为。
- **NAME 入口**依赖设备端的标准动作名表（Kotlin）。与会话、稀疏都独立。
- **第 10 题合同**：r3 把它放进 15。但每个后续提交（会话、通知、launch、文本）都会产出结果和 exit。合同越晚定，前面写的结果就越要返工。r3 自己也说"若选 B/C，第一次触及就同步"，A 同样适用。
- **首页指引**（16）需要最终格式，但可以先随 15 出一版，后续提交各自同步对应段落。r3 已经要求"各项契约文档随该项交付"。

**判断**：排在最后不合理。这几项是证据最硬、风险最低、主要在 host 一侧的改进；放到最后，A1（唯一能回答"值不值"的测量）就要等全部 14 个提交之后。

**建议的阶段顺序：**

| 阶段 | 内容 | 目的 |
|---|---|---|
| 0 | 1 文件早关、3 配置父路径（独立小修） | — |
| 1 | 结果合同（第 10 题，按所选规则统一单步/batch/清理，含去掉 done+1 与 batch 交付特判）→ 15（单一 View + diff-from + compact + find + NAME + 保存失败照印）→ 16 首版首页 | 随后跑 A1：同一后端，新旧呈现各一对真实任务 |
| 2 | 2 会话 → 4 稀疏 canonical → 5 采集截止 → 6 身份 → 7 等待 → 8 目标规则 | 8 落地后在 Chrome 上复测一次 View 成本（target 规则会改变网页行数） |
| 3 | 9–13 输入、剪贴板、通知、activity，14 region | 16 终版整合 |

---

## 3. 确定缺陷与证据不足的断言

### 3.1 确定缺陷

| 编号 | 位置 | 后果 | 证据 |
|---|---|---|---|
| D1 | `SRC androidctl/src/androidctl/ui_client.py:1001-1007`（`run_request` 在 finally 里 `await client.close()`）；`cli_options.py:136-141` | 设备已返回结果之后，若本地连接 close 抛 `Fault`，`return` 的结果被丢弃，CLI 报 **not sent / exit 2**。动作可能已经派发，这正是会诱导重发的错误类别。若抛 `OSError` 等非 Fault，则是未捕获异常。batch 对同样的情况有专门处理（`ui_batch.py:378-388`）。 | 【源码】。触发频率未知：`transport.py:54-71` 对 TimeoutError 已吞掉，其它 BaseException 会重新抛出；未见实测 |
| D2 | `ui_batch.py:383-385` | batch 本地连接 close 失败为 exit 1；r3 A 要求"普通清理 exit 3"。现状与 A 不一致，r3 没有列为迁移点 | 【源码】 |
| D3 | `TargetManagement.kt:86-92` | UI 目标 stream close 失败只加 cleanupError，exit 不变；与 A 的"普通清理 exit 3"不一致，同样没列为迁移点 | 【源码】 |
| D4 | 单步 vs batch：附件失败（`ui_client.py:1387-1404` 对比 `ui_batch.py:356-366`）、stdout 失败（`output.py:96-121` 对比 `ui_batch.py:367-372`） | 同一事实，单步 exit 0、batch exit 1。r3 A 把它固化成合同 | 【源码】 |
| D5 | `ui_client.py:1421-1425` | 保存失败时内存里的正文不打印（评审方 F5，r3 已计划修） | 【源码】 |
| D6 | 窗口缺席原型 host 特判（`window-absence.patch` 的 ui_client hunk） | 在 host 解析单个命令的谓词才能续批；维护面会随命令种类扩散 | 【原件】 |
| D7 | `P/r2-delivery/probe.py` 与 README | 被引为"diff-from 原型"，实际验证的是被放弃的 B（行 diff）；推荐的 C 没有原件 | 【原件】；C 已由我复现（§2.2） |

### 3.2 证据不足的断言

- "紧凑正文与单一 View 能降低真实任务成本"：只有上一轮 2 次单例，其中 Chrome 交叉；本轮零新测。
- "region 能读懂小字"：WPS 的 JPEG 已经可读；新采样链、峰值和小字收益都没有实测。
- "REF 裁片需要同域签名比较"：没有业务证据说明比 Fresh 边界更必要。
- "两次缺席防抖足以区分遮挡与消失"：只有真正关闭的样本，没有遮挡样本。用户已接受"仅是当前观察不到"的语义，所以不影响合同，只影响文案。
- 原型方案 C"98 条记录"：包内没有原件；我得到 99 条，可能用的不是同一份树。

---

## 4. 值得删除或简化的机制

1. **batch 里另一套交付停止判断**（`explicit_delivery`、`delivery_error` 和 stdout 分支）。改成"每步按统一 exit 规则，非 0 即停"。
2. **窗口缺席的 host 谓词特判**。改为设备发布通用的"不适用"附件状态。
3. **REF 裁片的整份观察签名比较**。改用 Fresh 当前边界。
4. **region metadata 随输入资格失效**。改为留到下一张完整图或租约结束，几何不可比时拒绝。
5. **diff 持久基线全套**（BASELINE、预留/提交、batch 收尾、diffBaselineError、打印后提交），由 diff-from 取代。r3 已计划；这里确认它是净删除。
6. **封套里新增的身份字段**：直接复用 `comparison_key`，不新增字段或多级 reason。
7. **"done + exit 1"的两处重载**（launch 落点、通知清理）。

---

## 5. 需要用户裁决的事

**唯一一项：单动作的退出码是否反映"请求的交付物缺失"。**

- **简单规则（推荐）**：Result 只描述动作；exit 0 当且仅当动作 DONE、请求的交付全部到手、清理已确认；listener 例外保留。单步与 batch 一致。
  - 代价：附带截图失败时，单步也会非零。习惯"exit≠0 就重试"的调用者必须先读 Result，所以指引里要写一句"Result=done 不要重做"。
- **A**：单步附件失败仍为 0，batch 为 1。
  - 代价：两套规则；exit 0 不保证请求的东西都在。
- **剩余未知**：没有调用者在这些失败分支下的真实行为数据。

**不需要用户裁决、归实施细节的：**

- 第 9 题选 A，坐标单位与 tap 一致，按 §2.4 简化。
- diff-from 采用 C，保存为 JSONL。
- 通用动作集写进头行（若担心偏离用户原意，可在首页示例里展示后请用户确认一次，不必单独出题）。
- 退出码保持五类。
- 提交顺序按 §2.6。
- 提交 7、14、15、16 的修订验收：
  - **7**：增加"同 ID 窗口一次缺席后返回 → 不提前 gone"的 JVM 用例；增加"等出现时被遮挡后露出 → 成功；一直缺席到截止 → 超时并写明窗口缺席"；去掉 host 特判。
  - **14**：REF 用 Fresh 边界；metadata 寿命按 §2.4；验收"截图 → 无关变化 → region 仍可用；旋转后拒绝；裁片不续期输入帧"。
  - **15**：加入第 10 题统一规则；单步与 batch 同例对比；保存为 JSONL 并验证 rg 定向续读；find 离屏命中验证用 Chrome 的 Addis 例。
  - **16**：首版随 15 交付，用 A1 验收。

---

## 6. 未能核实的内容

- `T/reviews/route-r2-overall-design.md` 与 `route-r2-window-absence-plan/runtime/source.md` 我只读了 contract-integration 一份，以及 window-absence 的 README/result 和原件。窗口缺席的 44 项 JVM 测试是否实际通过，没有核原始日志。
- selector 后继那次的 stdout 原件没有逐字节读，只核了 result.md 的转述。
- 上一轮原型 token 数（26,793 / 26,033）没有用 tokenizer 重算；`P/presentation/runtime/*.json` 的逐项数值没有独立复核，只核了 md 表格与评审方表的一致性。
- r3 引用的 Bilibili 206–209 batch 与图像资格原件没有核。
- 现有 zoom 的 Kotlin 实现（`VisualInput.kt`、`ImageRegion.kt`）没有逐行读，相关说法依据 `images.md` 与实施方研究。
- `U27-U28` 用户原话文件不在 `ROOT/context`（只在另一个外发包目录），我用的是 `T/decisions.md:38` 的 U27 摘要。
- 所有运行结论都只来自 MuMu 或 API34 模拟器，没有真机或厂商 ROM 证据。
