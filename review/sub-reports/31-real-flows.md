# 专题 31：真实 App 与真实网站流程的证据审计

核查对象：r3 主稿（`T/deliverables/route-review-r3-recommendation.md`）中"U28 新真实流程及具体增量"一节，及其所依据的 MuMu / API34 原件。所有流程都是用**冻结候选**跑的：Android 端 APK SHA256 以 `c6695f3e…` 开头（下称"冻结 APK"），主机端是非 editable 安装的冻结 wheel `1e453395…`，对应候选提交 `3f45af6`。r3 方案本身没有实现。例外是三处研究 APK：文件早关 APK `99b36…`、AX 选区研究 APK `1fced…`、身份原型 APK `8d15…`，下文会单独标出。

运行环境只有两种。一是 MuMu 模拟器 index1（Android 15 / API35，boot `01170df9…`）。二是 Android Studio 模拟器 API34 x86_64，用软件 GLES（SwiftShader）、2 核、2 GiB。没有真机。

我写的脚本和中间结果都在 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents6/31/`：

- `tab.py`：逐条列出每次命令，结果在 `mumu.tsv`、`api34*.tsv`
- `flows.py`：按流程汇总，结果在 `flows.txt`
- `bloat.py`：统计 stdout 的构成
- `actnames.py`：统计动作名称的分布
- `ff020.png`：从原件中取出的独立 ADB 截图

下文的【重算】都指这些脚本，脚本都用 `python3 -I` 运行。

---

## 1. 结论摘要

1. **记账：评审方成本表与原件一致。** 我用 `command.json` 自己重算了 WPS、Maps、MDN、GitHub、GOV.UK、OSM、京东、Bilibili 八段的 CLI 次数、wall 和 stdout 字节，与实施方 review 的成本表逐项相同。WPS 的差 1 次只是 036 号 `swipe --help` 的归类不同。【重算】
   - 12 条业务流程共 199 次 CLI。
   - 完整完成 9 条：WPS、Maps、MDN、GitHub、GOV.UK、OSM、Bilibili、NIST（WebView 测试宿主）、Chrome regex101（其中多行 AX 填值仍为 FAILED）。
   - 未完成 3 条：同程、京东、Firefox。
2. **摩擦分类：最常见、也最费调用的是 (a) 类，r3 没有对应项。** (a) 指"动作后的即时观察仍是旧页、加载中或空白"，约 11 次，每次多花 1–2 次调用。其中两处给出了**假的就绪信号**：
   - GitHub 101：回执为 `click: done; changed`，没有 unsettled 标记，但页面仍是旧列表。
   - NIST 017：`wait --contains Ethanol` 命中的是地址栏 URL，不是结果正文。
3. 其次费调用的类别：
   - (e) 图像资格与预算：Bilibili 一段多花约 5 次。
   - (i) CLI 用法错误与查帮助：10 次帮助、4 次参数或命令错误。
   - (g) App 自身门槛：同程、京东、WPS 营销页、Chrome 首启引导。
4. **r3 着墨最多的身份题（第 12、13 题）在真实流程里几乎不产生额外调用。** MDN 两次 reveal 局部 unobservable 后，**同一次输出里已经交付了新 REF**，下一步直接用，额外调用为 0。【原件】
   - 13 道题里，第 1、3–8、11 题在本批 12 条真实流程中一次也没出现为调用者摩擦。
   - 它们的依据来自更早的定向实验或源码，不是纯推演，但也不是真实使用中的高频问题。
5. **对主稿关键说法的核对**
   - WPS 035、MDN 081/087、Bilibili 206–209、NIST 021 的事实准确。
   - Maps 058 的"业务 name 真变"属实，但只是同一个"步行"标签页在路线加载后从"步行模式："变为"步行模式：18 分钟"，主体没变，主稿把它当作"正确拒绝"的论据说得偏重。
   - NIST 021 漏了"所交树是 partial（222 节点、collectionLimit）"，而且调用者实际传了 `--all`。
   - GitHub 101 被说轻了：那是一个假的 settled 回执。
6. **效率：本轮没有任何"新方案更省"的实测。** 这一点符合预期，也由 r3 自己确认。
   - 人类可读输出中 51%（MuMu）到 60%（API34 三组）的字节是逐节点的 `actions` 数组，其中 `click` 只占动作字节的 5.7%。【重算】
   - CLI wall 只占流程首末跨度的约 6.6%，调用轮次和输出体量才是成本主体。
   - 第一轮测得的"候选是基线 1.7–2.2 倍"这个问题，r3 只有排在最后、未测的第 15 个提交在处理。
7. **证据质量**
   - 操作者是实施方体系内的 explorer agent。它通过包装脚本 `ops.py` 调用，该脚本**只在 stdout 小于 7000 字节时才打印**（`mumu/ops.py` 末段）。大多数输出它是读文件看的。GitHub 104 还借助了离线 raw 来选 REF。所以这批记录代表的是"专家加特权通道"，不是普通调用者。
   - "独立 review"由同体系另一个新上下文的 agent 完成。它确实核了 hash 和原件，也抓到两处 must-fix，但谈不上外部独立。
   - 省略的图片不影响我核的关键结论，文本原件已足够。
8. **Firefox / system_server**
   - 证据倾向模拟器图形通道。首次渲染 regex101 就像素停滞，独立 ADB 图证实。之后依次出现 Gecko GPU IPC 超时、RT throttling。看门狗杀进程时，CPU0 正在 `vendor.hwcomposer-3` 的 binder 线程里读 `goldfish_pipe`。实施方的诊断没有用到这条内核 NMI 原件。
   - AndroidCtl 当时的负载是普通用户级操作，没有加重的证据，但也不能排除。单次样本。
   - "worker 存活、旧 UI 连接失效"在源码上成立，值得列为低频缺陷。修法应当最小：DeadObject 时重置连接并给出明确错误码，不要做世代监控。
9. **综合判断：真实流程支持 r3 的部分方向，但削弱它的优先级排序。** 支持的方向有：保留图像阅读、默认 find 覆盖全部已采节点、保留唯一性与 partial 局部 REF、文件早关。最值得先做的三件事：
   1. 输出瘦身（第 15 个提交提前），并按第一轮方法实测 token。
   2. 处理"导航未完成却报已稳定"。
   3. CLI 可发现性和首页指引（第 16 个提交），优先于身份和输入证明题。

---

## 2. 逐题分析

### 2.1 逐流程建账

口径说明：

- "CLI"只计正常 CLI 调用，不含 ADB 准备。
- "wall"是各 `command.json.elapsedSeconds` 之和，包括进程启动和旁录归档。
- "跨度"是首条命令开始到末条命令结束的时间，包括 agent 思考、读图和记录。
- "stdout"是各命令 stdout 字节之和。
- "额外调用"是我按原件判断，因失败、拒绝、旧观察或重新观察而多出来的次数，属于估计值。

原件都在 `P/exploration-20261119/<目录>/commands/NNN-*/`。【原件＋重算】

| 流程（宿主） | 命令号 | CLI | wall | 跨度 | stdout | 非一次成功的步骤 | 额外调用 | 完成 |
|---|---|---:|---:|---:|---:|---|---:|---|
| WPS 公开 PDF（MuMu，WPS 26.10.2） | 012–042，不含 013/015/017/040/041 | 25 | 24.5s | 7.2min | 330,643 | 016、019 营销页；018 `open` 因 Resolver 8 候选零派发；020 打开后空白，到 021 才可读；026–032 共 7 次"下一处/改查询"都没到正文；035 click DONE 但没退出页面管理，靠 037 Back | ≈10 | 是，正文靠像素加 zoom 读到 |
| 同程 10.7.4 | 043–045 | 3 | 3.5s | 1.5min | 27,290 | 强制升级弹窗无法关闭 | — | 否 |
| Google Maps 步行路线 | 048–062 | 15 | 19.8s | 3.8min | 203,017 | 057 返回时路线仍在加载；058 staleRef 零派发 | 1 | 是 |
| MDN（Chrome Beta） | 074–090，另有 072/073 建 tab | 17 | 22.6s | ≈5min | 385,601 | 081、087 reveal 主 DONE，但局部 unobservable/selectionChanged；新 REF 已在同一输出中给出 | 0 | 是 |
| GitHub issue | 097–106 | 10 | 11.7s | 2.7min | 415,496 | 101 click 后即时仍是旧列表；102 cached find 命中旧页；103 fresh 才到详情 | 2 | 是 |
| GOV.UK 邮编 | 107–120 | 14 | 14.4s | 4.1min | 344,592 | 113 提交后 loading，114 才有结果；118 scroll-until 已执行 1 步后报父子同名歧义（exit1） | 2 | 是 |
| OSM 路线 | 121–129 | 9 | 12.7s | 2.6min | 451,874 | 122 发布观察 partial；127 提交后加载中，128 才有结果 | ≈2 | 是 |
| 京东 | 184–190 | 7 | 7.6s | 2.1min | 86,423 | 184 附属树预算耗尽，185 重读；186 热词从"烧烤炉"变"飞机"，staleRef；188 进入风控验证页 | 2 | 否 |
| Bilibili 讲座 | 191–212 | 22 | 27.5s | 8.9min | 273,778 | 192 rootUnavailable；202 staleRef（控件已隐藏）；203、204 补截图；206 notReady 加附图预算耗尽；207 staleImage 停批；208 补截图 | ≈7 | 是 |
| 文件 push（MuMu Download） | 013、015 | 2 | 1.4s | — | 693 | `publishFailed / Text file busy` | — | 否；研究 APK 155/157 另行成功 |
| Firefox 160（API34） | 010–029 | 17 | 40.1s | 4.8min | 156,052 | 013 `keys BACK`、014 `key BACK` 参数错；017–021 像素停在 logo，021 click DONE 但树未变；025 wait 等 19.8s 失败；027、029 图像为 null | — | 否，随后 system_server 被看门狗杀死 |
| NIST（API34 WebView 测试宿主） | 010–026 | 17 | 35.7s | 5.2min | 506,919 | 010 scopeUnknown，011 重读；016 提交后即时仍是旧表单；017 wait 命中的是 URL；021 wait 因 cutoff 失败；025 Back 后即时仍是旧明细，026 重读 | ≈4 | 是，但宿主是测试壳 |
| Chrome regex101（API34） | 029–068 | 36 | 39.6s | 12.3min | 642,676 | 030–036 Chrome 首启引导；037/041 为启用浏览器支持而关、开会话（加 3 条 ADB）；053 多行 AX fill 已派发，结果 FAILED（读回多两处 ↵）；055 `select` 命令不存在，056/057 查帮助 | ≈12，其中约 9 次是准备 | 业务是；多行 AX fill 否 |
| 多行 AX/IC 对照（研究） | editor-pair 011–050 | 40 | 45.8s | 13.2min | 771,450 | A029 fill UNKNOWN（本地读预算耗尽，exit3）；B036 清空遇 placeholder，FAILED | 研究 | 研究 |
| AX→IC 映射（研究 APK `1fced`） | ax-ic 014–045 | 32 | 37.8s | 11.4min | 596,461 | 038 打开管理 App 被拒（protected） | 研究 | 研究 |

几点说明：

- 我的重算与 `T/reviews/route-r3-realflows-incremental.md` 的三张成本表对得上（例如 WPS 25 次 / 24.451s / 330,643B，Bilibili 22 次 / 27.513s / 273,778B）。该表没有覆盖 API34，API34 的数字是我补算的。
- 全部五个目录共 327 次 CLI。命令使用分布：
  - `click` 97、`snapshot` 55、`open` 25、`fill` 23、`status` 15、`back` 12
  - 帮助 10、`find` 8、`reveal` 7、`scroll` 4、`tap` 4
  - `editor select` / `editor commit` 各 4、`screenshot` 3、`batch` 3、`wait` 3
  - 其余各 1–2 次
- 选项使用：`--screenshot` 93 次，`--interactive` 57 次，`--all` 15 次。8 次 `find` **全部带了 `--all`**。【重算】

### 2.2 摩擦分类

只计本批真实流程，研究对照另列。"额外调用"是保守估计。

| 类别 | 实例（命令号） | 次数 | 额外调用 | 代表原件 |
|---|---|---:|---:|---|
| (a) 动作后即时观察仍是旧页、加载中或空白，需要再观察 | WPS 020→021；Maps 057；GitHub 101→102→103；GOV 113→114；OSM 122→123、127→128；京东 184→185；NIST 010→011、016→017、025→026 | ≈11 | ≈12 | `mumu/commands/101-github-open-issue/stdout.bin`（首行 `click: done; changed`，无 settle 标记，screen 仍是 Issues 列表）；`api34-recovery/commands/017-*`（wait 命中 URL） |
| (b) 动作已发，但局部结果 unobservable 或 UNKNOWN | MDN 081、087；Maps 060、Bilibili 205/209/211 的 effect unknown（业务后态另有正证）；regex 053 FAILED（读回值不同）；对照 A029 UNKNOWN | 7（真实流程 5） | ≈1 | `mumu/commands/081-mdn-reveal-examples/stdout.bin`（`reveal={"state":"unobservable"…"reads":11}`） |
| (c) REF 因名字或行变化被拒 | Maps 058（"步行模式："→"步行模式：18 分钟"）；京东 186（热词真变）；Bilibili 202（控件已隐藏）；MDN 081/087 的局部身份 | 3 次拒绝，另有 2 次局部 | 3 | `mumu/commands/057-*/stdout.bin` 与 `058-*`；`raw/057-maps-pick-origin.tree.json` 中 `view.name="步行模式："` |
| (d) 选择器歧义 | GOV 118（链接 843 与其子文本 844 同名） | 1 | 1 | `mumu/commands/118-gov-return-discovery/stdout.bin` |
| (e) 预算耗尽、附图失败、staleImage 连锁，以及坐标前置截图 | Bilibili 203/204、206、207、208；京东 184 附属树预算 | 5–6 | ≈5 | `mumu/commands/206-*`（`imageError="Request budget exhausted"`）、`207-*`（`Take a new screenshot first`） |
| (f) 动作回执 DONE，但业务没发生 | WPS 035；Firefox 021（click DONE，树 unchanged） | 2 | 1 | `mumu/commands/035-wps-leave-page-editor/stdout.bin`（`effect unchanged`，仍有"返回 / 页面设置"） |
| (g) 环境或 App 自身门槛 | 同程升级层；京东风控；WPS 营销 016/019；Chrome 首启引导（3 个 API34 窗口各约 6 次）；浏览器支持的启用与关闭准备；Firefox 图形与系统故障 | 7+ | ≥20 | `mumu/tongcheng-evidence/000027*.jpg`；`jd-evidence/000108*.jpg` |
| (h) 文件发布失败 | 013/015 | 1 | 1 | `mumu/commands/013-push-pdf/stdout.bin`；`probe-output.txt`（Download 目录的 `f_type=0x786f4256`，即 vboxsf，写端未关时 renameat2 返回 errno26） |
| (i) 其他：Intent 解析、根不可读、CLI 用法 | WPS 018 Resolver；Bilibili 192 rootUnavailable；API34 013/014/055 参数或命令错误；038 protected；10 次帮助；WPS 026–032 正文不在 AX 中，只能靠像素 | ≈20 | ≈15 | `api34/commands/013-cancel-default`（`No such command 'keys'`）、`014`（`key` 只接受编辑键）、`api34-recovery/commands/055` |

结论如下。

- **按出现次数**，最常见的是 (a)，约 11 次。其次是 (i) 和 (g)。
- **按额外调用**，最多的是 (g)，但这类不是 AndroidCtl 能解决的。在可归于产品的类别里，(a) 最多，约 12 次；其次是 (i) 中的帮助与用法错误，约 14 次；再次是 (e)，约 5 次。
- (c) 的 3 次拒绝里，有 2 次（京东、Bilibili）是目标真的变了或消失了，拒绝正确；Maps 一次有争议，见 2.4。
- **(b) 和 (c) 中 MDN 的部分几乎不费调用**：新 REF 在 reveal 的同一份输出里就给了。
- 题目示例里的 "Compose" 不在本批流程中，它来自更早的 r2 研究轮次。

### 2.3 对照 r3 方案

| 类别 | r3 中的对应项 | 我的判断 |
|---|---|---|
| (a) 旧页、假就绪 | **没有直接对应。** 主线 6 只说"事件不证明稳定"；第 1 题是采集路线（E9 回调与公开 cache 之争），处理截止归因，不处理导航未完成；第 7 个提交只针对窗口缺席；GitHub 一行写的是"不重发已接受点击追即时后态"。 | **高频且未处理。** GitHub 101 在内容出现微小变化（链接焦点）后就判为稳定，回执给出 `changed` 而无 unsettled 标记，这是假的稳定信号【原件】。skill 的定义是"两次相同样本即停"（`skills/androidctl/observations.md:292-295`），所以加大 `--settle` 也无济于事。【源码】 |
| (b) 局部 unobservable | 第 12、13 题，第 6、8 个提交；第 4、5 题与第 10 个提交（输入证明） | 只覆盖 reveal 身份（MDN）。A029 的"本地读预算耗尽"在大网页上出现，没有任何题目针对它，只能间接依赖第 4 个提交的 sparse 表示，而那项尚无"更快"的实测。 |
| (c) REF 拒绝 | 第 12、13 题 | Maps、京东、Bilibili 三例在 r3 下仍会被拒，r3 也认为应该拒；第 12、13 题只改变 MDN 这一类，而那一类额外调用为 0。 |
| (d) 歧义 | 第 8 个提交（NAME 只在已广告动作的节点中唯一匹配）；"网页专项折叠"列为明确不做 | 只出现 1 次，回执列出了候选，恢复成本是 1 次调用。不处理可以接受。 |
| (e) 图像资格 | 主线 6，第 9 个提交（"不用跳后采换取旧图兜底"），第 10 题 | r3 只保证照实报告，不降低成本。Bilibili 一流多花约 5 次。属中频、未改善。 |
| (f) DONE 未发生 | 主线 6（"动作回执与业务完成各有依据"） | 只是表述原则，无产品变化；这类问题本来也难由工具解决。可以接受。 |
| (g) 门槛 | 明确不做（不绕门） | 合理。 |
| (h) 文件 | 第 1 个提交（写端早关） | 修法正确且小。但触发条件是 MuMu 的 vboxsf 共享目录【原件】；真机 `/sdcard` 是 FUSE，触发概率未知【需设备】。重要性被高估，但做它成本低，可以做。 |
| (i) 用法与可发现性 | 第 16 个提交（skill 自足） | 部分覆盖。`key BACK` 被拒、`keys` 不存在、`select` 在冻结 APK 中不存在，这些都是真实误用。 |

**真实流程中高频、r3 完全没处理的问题：**

1. **导航未完成却回执已稳定**，即 (a)。
2. **输出体量。** r3 有第 15 个提交（compact），但它排在最后，没有量化目标，也没有实测。
3. **图像资格连锁的调用成本**，即 (e)。

**13 道题中，本批 12 条真实流程从未作为调用者摩擦遇到的：**

- 第 1 题（采集路线）、第 3 题（是否删 IC 选区）、第 4 题（AX select 确认时点，只在研究 APK 中出现）、第 5 题（IC 相对 DONE，涉及 Compose）、第 6 题（通知监听器寿命）、第 7 题（virtual Activity 通知）、第 8 题（后台屏 task 迁入）、第 11 题（不可确认清理）。
- 第 2 题：Bilibili 209 确实旋转了，但旋转后没有坐标输入，没遇到。
- 第 9 题：只有 WPS 039 一次旧 zoom，正常工作。
- 第 10 题：出现了退出码 1/2/3，调用者处理时没有可见困惑。
- 第 12 题：MDN 两次，额外调用 0。
- 第 13 题：问题来自 Maps 051→052 原始树的离线比较，不是业务失败。

需要说明，第 2、5、6、7、8 题在 r2 有定向实验原件，不是纯源码推演；第 11 题主要依据源码。但就"调用者最常卡在哪"而言，r3 的大部分题量落在本批流程里低频的问题上。【推断，样本：MuMu 9 条加 API34 3 条，2 个系统镜像】

### 2.4 核对主稿关键说法

- **WPS 035"点击正回执但未退出"：准确。**
  - 035 首行为 `click: done; changed … effect={"state":"unchanged"…}`，输出中仍有"返回"按钮（e748157416），"页面设置"变成 `disabled`。037 Back 后屏幕标题为 WPS-PDF。【原件】
  - 对应图 000021 已被省略，但文本足以支持结论。
  - 补充一点：回执里的 `changed` 指整屏内容变化（"页面设置"被禁用），调用者可能误读为成功，这正是 (f) 类的陷阱。
- **Maps"加载后业务 name 真变，旧 REF 零派发 staleRef"：字面准确，解读偏重。**
  - 057 中 e748159068 的 `name="步行模式："`；058 中当前 e748159201 的 `name="步行模式：18 分钟"`，同为 `rowIndex 0 / columnIndex 2` 的标签页。【原件】
  - 主体（"步行"标签页）没有变，变的是加载后补上的时长。拒绝很保守，代价是 1 次调用（058 的输出已附新 REF）。
  - 主稿把它当作"不能按同 source 自动接受旧号"的正例，这只是一种设计立场，不是"业务主体真变"的证据。第 12、13 题选 B 时它照样被拒。
- **MDN 两次 reveal"已派发而局部 unobservable"：准确，但影响被高估。**
  - 081：`reads=11`；087：`reads=8`；084 同类操作为 observed。【原件】
  - 新 REF e748160918 / e748161509 就在 081 / 087 自己的 stdout 中，082 / 088 直接使用，额外调用为 0。【原件，grep 计数各 1】
  - 真实代价在于：在静态 batch 里写"reveal REF 后 click 同一 REF"会失败。本批只用过 3 次 batch，都不涉及 reveal。
- **GitHub"即时 click 后仍旧列表"：准确，但说轻了。**
  - 101 首行 `click: done; changed`，**没有** `settle=unsettled`；本批其它 28 次未稳定的命令都标了 `settle=unsettled/loading`。【重算】
  - 102 cached find 又在旧缓存上命中旧页的"3.12"标签。
  - 这是回执给出假稳定信号。主稿只写"不重发已接受点击"，没有指出这是产品侧问题。
- **Bilibili 206–209 连锁：准确。**
  - 206：等待 4.985s 后 `notReady / dispatchStarted=false / imageError="Request budget exhausted"`。
  - 207：batch 第 1 步 `Take a new screenshot first`，第 2 步未执行。
  - 208：截图。209：batch 成功。【原件】
  - 主稿没说的是根因链条：播放器控件会自动隐藏，所以 202 和 206 都失败。仅暂停和全屏两项，201→209 就用了 9 次调用、约 5 分钟跨度。
- **NIST 021"cutoff 失败，所交树不含该数值"：准确，但有遗漏。**
  - 021 的观察是 partial：只有 222 个节点（022 有 3712 个），`collectionLimit`，`max-chars` 截掉 152 个节点，屏幕 `unreadable:true`。所交树确实不含 352.2。【原件，`raw/021*.tree.json` 309,739B 中计数为 0】
  - 调用者实际传了 `--all --max-chars 30000`。按 skill，`--contains` 只匹配可见文本，`--all` 只影响打印范围。调用者显然以为它能扩大匹配范围，这是一个用法陷阱。主稿把原因归为"现有 `--contains` 本来只匹配 visible"，对 021 那一刻并不完整：当时页面本身就没被完整采到。
  - 017 `wait --contains Ethanol` 命中 URL（假就绪）也属同一类，nist-result.md 自己写明了这一点，但主稿表格没有列出。
- **主稿没有提到的失败**
  - API34 013、014、055 三次 CLI 误用，加上 10 次帮助。
  - WPS 018 Intent 解析零派发；Bilibili 192 rootUnavailable；京东 184 附属树预算耗尽。
  - Chrome 上启用浏览器支持需要关会话、借 ADB、在设置里点约 8 步（regex 037–046）。
  - 各窗口的 Chrome 首启引导每次都要约 6 次调用。

### 2.5 效率

- **体量（只计人类可读 stdout）**【重算，`bloat.py`】
  - MuMu 149 条命令共 2,866,316B，其中 50.7% 是 `actions` 数组，10.5% 是首行加 `androidctl{…}` 头加 `events` 行。
  - API34 三组 135 条共 2,532,069B，`actions` 数组占 59.8%。
  - 按动作名分：`show-on-screen` 16.9%、`action-40` 13.3%、`action-102003c` 11.1%、`action-400` / `action-800` 各 9.4%，`click` 只占 5.7%。【重算，`actnames.py`】
  - 单条 stdout 最大：216 号 111,710B，129 号 96,018B，104 号 `find --all` 80,589B。
  - 解码后的树：NIST 022 为 5.6MB（3712 节点）。
  - 图像 payload：WPS 1.39MB，Bilibili 1.24MB（取自 review 表）。
  - 这说明第一轮缺陷 F4（"默认行对每个目标输出完整动作数组"）在本轮原件里**原样存在**，而且是最大的单一体量来源。
- **时间**
  - MuMu 八段业务的 CLI wall 合计约 144s，首末跨度约 36 分钟。CLI 自身只占约 6.6%，其余是调用者轮次（含 explorer 的记录时间）。
  - 所以对调用者来说，成本几乎完全由"调用次数 × 每次读入体量"决定，与此前评审方用模型 token 计量的结论一致。
- **"新方案更省"的实测：没有。** 这符合预期，也已核实：
  - r3 第 70 行明确说不能用这些数字论证加速。
  - 身份 B 的组合反而让最终 wire 增加 837 / 759B。
  - r2 稀疏两臂"CLI 更慢"。
  - compact 规格只有"仅省可推出动作"的定性描述，r2 处置表写明"旧比例不代新收益"。
  - 含义是：r3 至今没有证据回答第一轮最核心的效率问题（1.72–2.23 倍）。十条主线中能直接降低这个倍数的只有主线 7（compact、全已采 find、单一 View），它排在第 15 个提交，验收里没有 token 或调用次数指标。
- **使用分布（只作事实，不代表该删）**
  - 用到的：`click`、`snapshot`、`open`、`fill`、`back`、`find`、`reveal`、`scroll(-until)`、`expand`/`collapse`、`check`、`tap`、`swipe`、`zoom`、`screenshot`、`batch`、`wait`、`editor select`/`commit`、`shell`、`push`/`pull`/`job`。
  - 本批零使用：`locate`、`action`、`range`、`uncheck`、`focus`、`long-click`、`dismiss`、`ime`、`type`、`home`、`recents`、`wake`、`timeline`、`drag`、`pinch`、`clipboard`、`notifications`、`display`（后台屏）、`apps`、`diff`。
  - 本批没有选通知、后台屏、剪贴板类任务，零使用部分源于选题，不能据此删除。

### 2.6 证据质量

- **操作者**
  - MuMu 由 `r3_mumu_explorer`、API34 由 `r3_api34_explorer` 操作，都是实施方 root 派出的 agent。它们按协议执行"不重发、不换通道、失败照实保留"。
  - 业务命令经 `ops.py observe` 调用正常 CLI，同时旁录 trace 与解码后的 raw。
  - `ops.py` 只在 stdout 小于 7000 字节时才打印，大于时 agent 要另读文件。这批命令的 stdout 中位数在 1–3 万字节，所以 agent 读输出的方式与真实调用者不同。【原件 `mumu/ops.py`】
  - GitHub 104→105 写明"研究离线 raw 确认父评论 e748163960"再去读，这用了普通调用者没有的通道。【原件 README】
  - 静态 batch 计划（`bili-*-plan.json`）的写法显示操作者熟悉内部语义。
  - 结论：摩擦计数代表的是**专家加特权下限**，新手的摩擦只会更多。帮助调用 10 次、误用 4 次，说明即使是专家也要查。
- **"独立 review"**
  - 由同体系、"未参与实验的新上下文" agent 完成，核对了 freeze 和 hash 清单、看了图，并抓出 M1（WPS"下一处无 CLICK"的误述）、O2、MF-AX1 等文案问题。
  - 它对事实层是有效的二次核对；对"该不该做、优先级"没有外部视角，不能当作外评。
  - 我对它的成本表复算一致，对事实描述的抽查（035、058、081/087、101、118、206/207、021）也一致。
- **被省略的图片**：`OMITTED_IMAGES.tsv` 共 219 项。被省略的关键图包括：
  - Maps 000041/43/44；GitHub 000075/78；GOV 000090/91；OSM 000100–102
  - WPS 000007/10/17/21/25；Bilibili 000119/129
  - NIST 全部 8 张；regex 11 张；Firefox 4 张
  
  保留下来的有：WPS 正文图 000023 和 zoom 图 000024（我看过，确实出现 4.2.2 和"duplicate Dictionary keys…all but the last instance are ignored"），Bilibili 124/128/130/131，京东 108，同程 027。凡是我核的关键结论都能由文本原件（stdout、raw 树）独立支持；Firefox"像素停在 logo"由 API34 020 号 ADB 截图的 stdout 原件（33,363B PNG）直接可核，我已取出查看。**图片省略不影响关键结论的可核性**，只影响"人眼读图"类的细节，例如 NIST 024"亲看首行"。
- **只有模拟器，影响最大的结论**
  1. Firefox / system_server 归因：软件 GPU 加 2 核本身就是嫌疑。
  2. 文件 013：vboxsf 是 MuMu 特有。
  3. 所有时延和体量：模拟器 CPU 与真机不同。
  4. 浏览器支持、IME、剪贴板的视觉分支。
  
  对 (a)(c)(d) 这类语义事实的影响较小：同一 Chrome、Maps 版本在真机上大概率重现，但这是推断。

### 2.7 Firefox 与 system_server 被看门狗杀死

已核原件：`api34/README.md`、`commands/0NN-*/command.json`、034 号全量 logcat（4,043,978B），以及我取出的 020 号 PNG。【原件】

**时间线**

| UTC | 事实 |
|---|---|
| 21:36:57 起（018/019） | AndroidCtl 的图和 20 号独立 `screencap` 都停在 regex101 启动 logo（`ff020.png`），而 AX 树是 full。这时 AndroidCtl 只做过 open、3 次带截图的 snapshot/key/click。 |
| 21:38:28 | logcat 24526 行出现 Gecko `Killing GPU process due to IPC reply timeout`。 |
| 21:38:53–21:39:13 | 025 号 wait 轮询了 20 秒。 |
| 21:39:37、21:40:07 | `UiAutomation takeScreenshot() returned null` 两次（25112、25531 行）。 |
| 21:40:06 | 029 号 `open` 启动 WebView，触发系统过渡动画。 |
| 21:40:07.9 | 内核 `RT throttling activated`。 |
| 21:41:00 | 看门狗 WAITED_HALF。 |
| 21:42:00 | 看门狗杀死 system_server 500（27170–27246 行）。`android.anim` 线程停在 `ScreenCapture.nativeCaptureLayers ← TaskSnapshotController.recordSnapshot ← Transition.onTransactionReady`，并持有 `WindowManagerGlobalLock`。 |

**实施方诊断没有用到的原件。** 27096–27165 行是看门狗触发的 sysrq NMI 回溯：CPU1 是看门狗自身；**CPU0 是 `binder:368_4`，368 是 `vendor.hwcomposer-3`（2375 行），正在 `goldfish_pipe_read_write`**，也就是在和宿主模拟器的图形管道通信。这是单点采样，不能证明它一直卡在那里，但它与"captureLayers 等待 SurfaceFlinger/HWC → 宿主渲染器"的链条一致，指向模拟器图形通道。诊断写的"缺 SurfaceFlinger/GPU native 栈"应修正为"有一帧内核级 HWC 采样，仍缺用户态栈"。

**判断：是模拟器图形问题，还是 AndroidCtl 加重？证据走到下面这一步。**

1. 像素停滞在 AndroidCtl 产生明显负载之前就出现，独立 ADB 截图同样如此。
2. Gecko 先于系统阻塞出现 GPU 超时。
3. 阻塞点在平台的任务缩略图捕获，触发它的是一次普通 app 启动（029）。真人切换应用也会触发同一路径。
4. 内核显示 HWC 在宿主图形管道读等待，同时 RT 线程被节流。
5. AndroidCtl 的负载是约 6 次截图请求，加 20 秒 accessibility 轮询；它的两次截图失败是**症状**，不是起点。

综合来看，倾向于"软件 GLES、2 核的模拟器图形通道，叠加 Gecko 渲染"。AndroidCtl 截图和采集**没有加重的证据**，但样本只有 1 次、1 个 AVD，也没有 ANR 正文，**不能排除**。【原件加推断；需设备才能闭合】

**"worker 还活着，但旧连接已失效"是否算产品缺陷：算，属于低频但真实存在的缺口。**

- 证据【源码，3f45af6】：
  - `UiBackend.kt:440-444`：`connect()` 在 `platform != null` 时直接复用。
  - `UiBackend.kt:503-521`：只有 `clear` / `disconnect` 才会把 `platform` 置空。
  - 仓库中没有任何针对 UiAutomation 或 system_server 的 `linkToDeath`；`linkToDeath` 只出现在 `WorkerConnection.kt:90,258`，监视的是 worker Binder。
  - `WorkerService.kt:149`：`worker` 字段只看 `sessions.stopped`，正常时就报 `ready`。
- 推断：system_server 重建后，第一次 UI 调用可能抛 DeadObject，也可能被 UiAutomation 内部吞掉后返回 null（例如表现为 rootUnavailable、截图 null）。这时调用者看到的是"读不到根"之类的误导性错误，而 status 仍报 ready。如果关会话时 `unregister` 抛异常，`cleanupFailure` 会让之后所有 UI 请求都被拒并提示"restart the worker"（`Sessions.kt:144`）。后者虽然粗糙，但至少明确。【源码推断，需设备】
- 本轮在系统重建后没有再调用 status 或 UI，所以**没有实测反例**。
- 真机上看门狗或 soft reboot 很少见，但会发生，而 Shizuku 拉起的 worker 通常能存活（本例中 2148 和 2268 都活着）。
- 我不同意 r3"只记为未知、什么都不做"。最小修法不需要世代监控：UI 调用遇到 DeadObject 或连接异常时，就地清掉 `platform` 并返回明确错误码（例如 `uiConnectionLost`），下一次请求自然重连；status 的 `ready` 不应暗示 UI 连接健康。这项可放进第 2 或第 11 个提交，需要设备验证。

### 2.8 综合

**支持 r3 的部分**

- 默认 find 应覆盖全部已采节点：8 次 find 全部手动加 `--all`。
- 保留图像阅读：WPS 正文完全不在 AX 中。
- 保留唯一性检查和已执行前缀：GOV.UK 118。
- partial 状态下可靠的局部 REF 可以继续用：Maps 059→060、NIST 019。
- 文件写端早关：做法正确、成本低。
- 不绕过 App 门槛。

**削弱 r3 的部分（主要是优先级）**

- 第 12、13 题（加上第 6、8 个提交里的身份组合成本工作）在真实流程中额外调用为 0。
- 第 3–8 题和第 11 题在本批真实使用中没有出现。
- 而真实高频的 (a) 假稳定/旧页、输出体量、(e) 图像资格成本、(i) 可发现性，要么没有对应项，要么排在最后且没有量化验收。

结论是方向基本对，但实施精力放偏了：r3 把大量笔墨放在"证明语义精确"的边角上，对"调用者每一步要读多少、要多走几步"的主成本缺少量化目标。

**最值得优先做的三件事（从调用者体验出发）**

1. **输出瘦身前置**（第 15 个提交的 compact 部分，加上 find 默认全部已采），并用第一轮的同任务、同模型 token 方法实测，以"不高于基线"作为验收。依据：actions 数组占 51–60%，F4 原样存在。
2. **处理"导航未完成"**：click、fill、submit、back 之后，如果页面标识（标题、URL、窗口根）没变而且仍处在加载窗口内，不应报为已稳定（GitHub 101）；skill 也要指导调用者用 `wait --screen-change` 或 `wait --app`，避免 `--contains` 命中地址栏（NIST 017）。依据：本批最高频的产品侧摩擦。
3. **CLI 可发现性和首页指引**（第 16 个提交前移）：`back` 和 `key`、`keys` 的区分；`--all` 对 wait 不扩大匹配范围；浏览器支持的启用路径；Resolver 零派发时直接给出 `--component` 候选（018 已给出，可作范例）。依据：专家操作者也查了 10 次帮助、误用 4 次。

身份类的第 12、13 题建议暂按 A（维持现状）处理，等 batch 真实使用暴露出成本再重开。

---

## 3. 确定缺陷与证据不足的断言

**确定缺陷（候选 3f45af6）**

| 编号 | 缺陷 | 证据 |
|---|---|---|
| D1 | 动作后观察在内容出现微小变化（链接焦点）后就判为稳定，回执 `done; changed` 不带 unsettled，而页面导航尚未发生 | 【原件】`mumu/commands/101-*`；【源码/文档】`observations.md:292-295` 的"两次相同样本即停"。样本：1 次明确，另有 GOV 113、OSM 127、NIST 016/025 正确标了 loading 或仍为旧页 |
| D2 | 默认人类可读输出约 51–60% 是 `actions` 数组（即第一轮 F4），本轮依旧 | 【重算】`bloat.py`，样本 284 条命令 |
| D3 | UI 连接只在会话清理时重置，worker 的 `ready` 不反映 UI 连接健康 | 【源码】`UiBackend.kt:440-444,503-521`，`WorkerService.kt:149`；运行后果【需设备】 |
| D4 | `wait --contains` 加 `--all` 的组合容易被误解为能匹配离屏文本 | 【原件】`api34-recovery/commands/021` 的 argv；【文档】`observations.md:307-312`。属于可用性缺陷，文档本身没错 |

**证据不足或需修正的断言（实施方材料）**

- 主稿称 Maps 058 是"业务 name 真变"后的正确拒绝。实际只是同一标签页的标签在加载后变长，"正确"是设计立场，不是证据。
- 主稿把 MDN 列为第 12 题的核心反例，但没说明它的额外调用为 0。
- NIST 021 的归因漏了 partial 和 collectionLimit。
- API34 诊断写"缺 SurfaceFlinger/GPU native 栈"，实际有内核 NMI 下的 HWC goldfish_pipe 一帧采样（034 号 27130–27165 行），可以补充，但仍不能闭合。
- 文件第 1 个提交的必要性来自 MuMu vboxsf 环境，真机触发率未知。
- "独立 review 已关闭"只能说明事实层二次核对通过，不能说明外部独立。

---

## 4. 值得删除或简化的机制

只从真实流程证据出发：

- **身份组合成本的深度工作**：第 12、13 题的 B 方案、C13 组合下 detail 摘要缺口这类题目，在本批调用者体验中收益接近 0。建议按 A 维持，删去为它们预建的验收矩阵。
- **对 WPS 035 这类"DONE 但业务没发生"的情况，不要追加机制。** r3 也没加，这一点正确。
- **研究流程中的长串"准备"成本**：浏览器支持在 Chrome 上需关会话、借 ADB、点约 8 步。这不是删除对象，但可以简化成单条 CLI 引导，或在 status 中直接给出"浏览器支持未启用 + 设置路径"。这需要用户确认是否允许 CLI 打开系统设置页。

---

## 5. 需要用户裁决的事

1. **实施顺序是否改为"效率先行"。**
   - A：沿用 r3 的 16 个提交顺序，compact 在第 15 个。
   - B：把第 15 个提交中的 compact、find 默认全部已采两项和第 16 个提交前移到第 2 个提交之后，并设"同任务 token 不高于基线"作为硬验收。
   - 我推荐 B。代价是在会话与目标重构之后、身份工作之前，先承担一次输出格式迁移。剩余未知是 compact 的实际节省比例。
2. **是否把"导航未完成不报已稳定"列入下一轮范围。**
   - 这会改变动作回执的 settle 语义，涉及第 10 题的回执合同。
   - A：不改，只在 skill 中教调用者显式 wait。
   - B：click、fill、submit、back 之后，如果窗口根、标题或 URL 在 settle 窗口内未变、且页面在加载，就报 `settle=loading` 或 `navigationPending`。
   - 我推荐先 A 再按需 B。原因是 B 的判据在不同 App 里难以统一，未知较多，需要设备样本。

以下属于**实施细节**，不必问用户：

- D3 的最小修法：DeadObject 时重置连接并给出明确错误码。
- `wait` 帮助中注明 `--all` 不影响匹配范围。
- `key BACK` 被拒时提示改用 `back`。

---

## 6. 未能核实的内容

- 第一轮"1.72–2.23 倍"的方法在本轮没有重跑；本轮也没有任何 token 记录，所以 stdout 字节不能换算成 token。
- 被省略的图片（Maps、GitHub、GOV.UK、OSM、NIST、regex）我只能用文本原件替代核对。
- Firefox 事件的 ANR 正文、SurfaceFlinger 用户态栈、宿主 RenderThread 栈不在包内。
- 系统重建后旧 worker 的实际行为（报错文本、是否误报成功）没有原件。
- 真机上 vboxsf 以外的文件发布行为、(a) 类在真机 Chrome 上的频率，都需要设备。
- 身份原型运行（`identity-runtime/mumu` 中的 Maps 两臂和 MDN）我只列了命令表，没有逐条复核，属其它专题范围。
