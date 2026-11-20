# 第四轮评审专题：结果与退出码、batch、观察的打印／保存／比较、区域截图、文件传输

评审对象：产品仓 `4235af9`（主机源码与 `f146f32`、`af99a0b` 两个冻结截点逐字相同，只有 `skills/` 文字不同【源码 `git diff --stat f146f32 4235af9 -- androidctl/src`】）。工作目录 `/tmp/claude-0/-home-user-androidctl/89958852-96e0-588b-9740-adf34ce1e474/scratchpad/agents7/39/`，脚本在其 `scripts/`。下文 `PKG` 指实施方材料包，`ST` 指 `PKG/ultragoal/2026-11-11-next-improve/stages/41-route-rebuild`。

术语：
- "四臂"：实施方让两个独立模型会话分别用旧候选（old）和新候选（new）完成 Maps 路线查询、GitHub issue 查找两个真实任务，共四次；做过两轮（第 0 阶段一轮、最终一轮），共八次。
- "View 文件"：本轮新定义的观察保存格式 `androidctl.view/1`，一种 JSON Lines 文件（头部一行、每个节点一行、结尾一行）。
- "交付"：命令把观察正文或图片打印到 stdout、或写成文件。

---

## 1. 结论摘要

整体状态：第三轮定下的"结果只说主动作，退出码汇总整条命令"已经按验收标准落地，单步命令和 batch 用同一条规则，代码简洁；观察的打印、保存、比较也合并成了同一个 View，`041eaa7` 净删约 1,150 行。但"保存与比较"这一半，比它在真实任务里带来的价值重得多，还有两个确定缺陷。

最重要的发现：

1. **退出码合同在源码里自洽，与 `reference.md` 和第三轮 §2.2 的五条验收基本一致**【源码＋实跑】。"动作 done 但退出码非 0"这种情形，在八次真实任务、共 84 次 CLI 调用里一次也没出现过。非 0 退出共 5 次：3 次 `wait` 失败（退出 1），2 次 `scroll` 未发出（退出 2），模型的反应都正确【重算 `scripts/arms.py`】。所以"done＋非 0"给模型带来多少负担，目前没有任何实证。它不常见，而且结果字首先打印出来，我判断风险不高，**不建议推翻**。
2. **`--diff-from` 在实施方自己的真实验证里，一次有用的差量都没产出**【原件】。真正发生变化的 4 次比较里，2 次原生滚动都因为"差量比全文还长"退回全文，2 次网页滚动因为采集不完整（partial）退回全文；得到 `changes` 模式的 4 次全是"什么都没变"。八次真实任务里一次都没用过。它却决定了保存格式（JSON Lines）、解码器、校验和约 70 行指引，还会让第 3 道待决题里每种新的表示方案都得同步改比较规则。**建议删除 `--diff-from` 和 View 文件格式，自动保存改成与 stdout 相同的文本正文**（§3.1）。
3. **确定缺陷：带 `--diff-from` 的命令在"未发出"或"未取得观察"时，会往历史目录写一份空 View，并把它报告为 `snapshotPath`**。同一行里还印着 `Observation: unavailable; run snapshot.` 和 `snapshotError`。我在本机复现了，实施方自己的 `native007` 原件里也有这一情形，但他们的结果页和 review 都没有指出【实跑＋原件】。
4. **确定缺陷（误伤面大）：工作区缓存目录的私有权限检查过严**。`.androidctl/` 一旦已经存在、但权限不是"仅本人"，所有 `snapshot`、`find`、`screenshot` 都会在设备已经采样之后报 `failed`／退出 1；`use` 报 `not sent`／退出 2；动作只多一条历史错误。实施方真实运行里因此失败 2 次，准备期另有一次退出 2。在 WSL 的 `/mnt/c` 这类文件系统上，自己刚建的目录也过不了检查【推断，未在 WSL 实测】。这些是界面快照，不是凭据；只需要"新建时设为私有"，不需要"已存在时拒绝"。
5. 区域截图的坐标映射在 MuMu 720×1280 → 1080×1920 上与源码公式逐值吻合（2 个矩形、共 3 次调用），按节点（REF）裁剪只在 API34 上实跑过 1 次【原件】。push、查询、pull 的字节一致，我对 98,317 字节样本重算过 SHA-256，成立【重算】。
6. `ui_client.py` 有 1,520 行，承担了五种职责（§3.4），是这一块最需要拆分的地方。测试大多通过真实 CLI 解析器和本地假守护进程、写真实临时文件，属于行为层面的测试，不是复述实现；但相对实际用量偏多（§4.5）。

对待决题的看法：第 1 题（后台屏 Activity 通知）从结果合同的角度看，"派发与观察分列"（B／C／D）比 A 更符合"结果只说主动作"。第 3 题（动作字典）如果 `--diff-from` 被删除，B 和 C 就不再需要同步比较规则，成本会明显下降。另外，评审方自己第三轮 §2.2 第 9 条"启动落点未确认改为未知、退出 3"，与"结果只说主动作"的原则不一致，建议重议（§3.3）。

---

## 2. 确定缺陷

### D1（中）`--diff-from` 在没有观察时保存空 View，并报告为 `snapshotPath`

- **触发条件**：任何带 `--diff-from PATH` 的命令，只要设备没返回 capture。包括未发出（缺会话、缺设备、过期 REF 等）、UNKNOWN（回复丢失），以及 batch 中这类步骤。
- **位置**：`ui_client.py:1185-1199`。`deliver()` 的条件是 `observation is not None or comparison is not None`，有比较对象就渲染一个 `{"tree":None,"windows":None}` 的空观察。随后 `save_delivery`（`:1375-1404`）把它写进历史，并设置 `snapshotPath`。
- **实跑**：在无设备环境运行 `androidctl click e1 --diff-from nonexist.json`，结果是 `click: not sent; …; Observation: unavailable; run snapshot.; snapshotPath=".../.androidctl/snapshots/000002-…-untitled-untitled.json"; snapshotError="Requested observation was not delivered"`，后面还跟着 `androidctl {…}`、`diff {"mode":"full","reason":"treeUnavailable",…}` 和 `screen {"unreadable":true}` 三行伪正文，退出 2。连续运行三次，历史目录多出三个文件；文件内容是只有 `screen {"unreadable":true}` 的 View。
- **原件**：`ST/16-delivery/runtime/mumu-view/commands/native/007-safe-native-scroll-with-file-diff/stdout.bin`，一次 `not sent／sessionRequired` 的 scroll，`data.snapshotPath` 指向 `000002-…-untitled-untitled.json`。`view-result.md` 只把它记为"到期拒绝、exit2"。
- **影响**：调用者按指引"读 `data.snapshotPath`"时，读到的是一份空屏；这份空文件还占历史配额。拿它再做 `--diff-from` 会得到 `treeUnavailable`。同一行里"观察不可用"和"保存路径"互相矛盾。退出码不受影响。
- **最小修正**：只有 `observation is not None` 才渲染和保存；比较对象缺了观察，就只在 `data.diff` 里记原因。如果按 §3.1 删除 `--diff-from`，这个缺陷随之消失。
- **确定程度**：源码、实跑、原件三方面一致。

### D2（中，误伤面大）工作区缓存的私有权限检查把"权限不是仅本人"当成硬失败

- **位置**：`history.py:67-80` 对 `.androidctl`、`snapshots`、`shots` 三个目录都调用 `private_directory`。`private_files.py:221-236` 对已存在的目录执行 `check_private`。POSIX 上（`:149-160`）要求属主是当前用户，且组和其他人的权限位全为 0；Windows 上（`:161-218`）要求 DACL 只有"允许"类条目、主体只能是本人或 SYSTEM，且本人必须有完全控制。`workspace.save` 也走同一个函数（`workspace.py:47`）。
- **后果链**：读命令的主交付是保存文件（`ui_client.py:1447-1458`），保存失败就把 `snapshot`、`find` 改成 `failed`，`screenshot` 同理（`:1459-1461`）。这时设备已经采样，正文也已经打印。
- **实跑（Linux）**：把 `.androidctl` 设为 0755 后运行 `use -s foo`，结果是 `use: not sent; Private POSIX owner/mode required … mode=0o755; …; code=privatePermissions`，退出 2。
- **原件（Windows，实施方自己的真实运行）**：
  - `ST/16-delivery/runtime/mumu-view/commands/native/013-…/stdout.bin`：`snapshot` 实际采样成功，比较也成功（`changes` 模式），但 `"result":"failed","reason":"Unexpected Windows DACL access entry: …\.androidctl. Choose a writable --workspace-root…"`，退出 1。
  - `…/017-cached-body-with-original-save-failure`：同样 `failed`／退出 1，正文照常打印。
  - 准备期还有一次 `not sent/privatePermissions`／退出 2，那次是状态目录（`ST/16-delivery/efficiency/profile-preparation-correction.md:3`）。
  - 原因都是"用普通 mkdir 建的目录继承了父目录的 DACL"。
- **误伤面**【推断】：
  - Windows：资源管理器复制或还原的项目目录、同步盘目录、git 或其它工具预先建好的 `.androidctl`、FAT／exFAT 和网络共享。
  - POSIX：WSL 的 `/mnt/c`（drvfs 默认不带 metadata 选项时 chmod 无效，目录显示 0777；`mkdir(mode=0o700)` 之后立即检查也会失败，等于每次都失败）、vfat、CIFS、uid 不一致的容器挂载。
  - 后果：每个读命令都报失败、每个动作都带 `historyError`。指引给了修复办法（`images.md` 第 116-121 行），但调用者是模型，多半只会反复重试，或者换 `--workspace-root`。
- **必要性评估**：凭据目录（`state_directory()`，存放配对私钥和证书）需要这项检查，应该保留。工作区缓存存的是界面正文和截图，敏感程度与项目目录里的其它文件相当；产品用私有权限创建它已经够了。"已存在且权限更宽就拒绝写入"防的是"别人能读到用户自己放宽了权限的目录"，收益很小，误伤却是整个读路径失效。这一检查在基线 `7d5be70` 就有【源码 `git show 7d5be70:androidctl/src/androidctl/history.py`】，不是本轮引入的；但本轮把"保存失败"升级成了读命令失败（第三轮 §2.2 第 2 条），放大了它的后果。
- **最小修正**：工作区缓存和 `config.json` 只在新建时设为私有；已存在的目录只检查"是不是目录"，不检查属主和权限，最多附一条一次性提示。凭据目录维持现状。
- **确定程度**：源码、实跑（Linux）、原件（Windows）三方面一致；WSL 和 FAT 是推断。

### D3（低）batch 在所有步骤都完成、只是关闭连接失败时，把顶层结果改成 `failed`

- **位置**：`ui_batch.py:284-288`。`close_error` 存在且 `reason is None` 时，会设置 `reason="Cannot close batch connection…"`，于是 `aggregate()` 返回 `result="failed"`（`:189`），随后 `local_close_failed` 把退出码设为 3。
- **与合同不一致**：`batch.md` 写的是"Batch `done` means the plan completed; runtime stops use `failed`"。单步命令在同样情形下保留 `done`、退出 3（`protocol.py:67-69`）。测试 `test_close_failure_reports_the_executed_prefix` 只断言了退出 3 和每步 `done`，没有断言顶层结果字。
- **影响**：调用者看到 `batch: failed`，可能以为某一步失败了，去重跑后面的步骤。退出码 3 本身是对的。
- **最小修正**：关闭失败不设置 `reason`，保留 `done`，只附 `localCloseError`，退出 3。
- **确定程度**：读源码推出，未用假守护进程实跑。

### D4（低）没有结果行的退出 1：未处理异常、Ctrl-C、stdout 文件描述符已关闭

- **实跑**：用 `>&-` 关闭 stdout 后运行 `androidctl devices`，在 `app.py:890` 抛 `AttributeError: 'NoneType' object has no attribute 'isatty'`，退出 1，没有结果行。
- **源码**：测试 `test_batch_keeps_original_failure_and_reports_close_fault` 本身就把"动作回执之后抛 RuntimeError、整个 CLI 以 traceback 退出"当作预期行为。click 在 `standalone_mode=False` 下会把 Ctrl-C 转成 `Abort` 往外抛（推断，未实测）。
- **影响**：退出 1 在公开表里的意思是"已知失败或缺交付"；但动作发出之后的崩溃同样是退出 1，而实际状态是"未知"。频率很低。
- **最小修正**：`main()` 捕获 `Abort` 和未预期异常，用退出 3 表示"状态未知"；指引里补一句"没有结果行的非 0 退出按 UNKNOWN 处理"。

### D5（低）其它小问题

- `snapshot: not sent; …; Observation: unavailable; run snapshot.`：对 snapshot 自己说"去运行 snapshot"，没有意义【实跑】。修正：`ui_client.py:357-363` 对 `snapshot`、`find` 不追加这句。
- `screenshot REF` 也要求本租约（lease）里先有一张完整截图（`VisualInput.kt:134`），但 REF 形式用的是节点当前的原生边界，不需要完整图的坐标系。结果是只想放大一个节点时，也要多一次完整截图；显示旋转过还会被拒（`requireLayout`）。另外，REF 边界部分越出屏幕时直接拒绝（`ImageRegion.kt:159-160,175-176,189`），不做裁剪。原生 View 的无障碍边界一般已经裁到可见区域，网页虚拟节点未必【推断】。修正方向：REF 形式只检查当前布局，边界裁到显示范围，并在 `region` 里如实报告。
- `pull /sdcard/x exists.txt` 在无设备时先报 `deviceRequired`，而不是先报"目标已存在"【实跑】；`job get t123` 也要先选设备，与指引里"本地 `t…` 查询不联系设备"有出入【实跑】。都是检查先后顺序的问题，不影响安全。

---

## 3. 设计与简化

### 3.1 保存与比较：建议退回"保存即打印的文本"，删除 `--diff-from` 和 View 文件

现状（`041eaa7`）：打印、保存、比较共用一个 View。打印时渲染成文本；保存时编码成 `androidctl.view/1` JSON Lines（`view_file.py:62-70`）；比较时用解码器加校验（`view_file.py:109-256`）读回一个显式文件，比较器（`ui_diff.py`）按键比较新增、删除、变化、移动和顺序，满足下列任一条件就退回全文：文件缺失、文件无法读取、文件格式非法、采集不完整、阅读预算截断、设备／会话／worker／连接／目标／范围／选择策略／视图选项不同、差量比全文还长。

评价：

- **自洽性**：机制本身是自洽的，读回闭环我核对过，成立（§4.1）。比"固定基线"简单，也不需要锁和后提交。
- **价值**：在实施方自己的验证里，真正有变化的 4 次比较全部退回全文【原件，汇总自 `ST/16-delivery/runtime/mumu-view/commands/*/*/stdout.bin`，脚本输出见下表】；八次真实任务里没有一次使用【重算 `scripts/arms.py`，84 次调用】。

| 调用 | 模式／原因 | 计数 |
| --- | --- | --- |
| native005、013、015，web005（都是什么都没变） | changes／空 | {} |
| native010、011（原生滚动一屏） | full／diffLonger | removed 5–7，changed 1，added 5–7 |
| web006、007（网页滚动） | full／partial | — |
| native009（换了连接） | full／contextChanged | — |
| native012、016、007 | full／fileInvalid、fileMissing、treeUnavailable | — |

  滚动是最常见的变化。滚动之后，节点的增删本来就接近整屏，所以差量天然比全文长；网页又经常只采到一部分。比较的键里还包含 `connection` 和 `worker`，租约过期重开或 worker 重启之后，旧文件就一律不可比。"可比"的窗口很窄，而在这个窗口里最常见的答案是"没变"；"没变"用 `change=unchanged`（动作已经给出）或文件哈希就能回答。

- **代价**：
  - 源码：`ui_diff.py` 266 行，`view_file.py` 中解码和校验约 150 行，`ui_client.py` 中比较接线约 40 行。
  - 测试：`test_ui_diff.py` 573 行、46 个，另外 batch 和观察文件测试里也有一部分。
  - 指引：`observations.md` 的 "Explicit Diffs" 66 行，`batch.md` 两段。
  - 保存格式：JSON Lines 比打印文本大约 1.5 倍（`native-ordinary.json` 4,307 字节，同一份打印正文约 2,790 字节【原件】）。真实任务里模型确实会回读保存文件：GitHub new 臂用 `rg` 和 `Get-Content` 搜索历史文件，一次 `rg` 输出约 2.2 万 token，被工具截断【原件 `ST/16-delivery/efficiency/github-new/native.rollout.jsonl` 第 72-89 行】。如果文件里就是打印的文本行，同样的搜索更短，也和模型已经见过的行一一对应。
  - 演化成本：待决第 3 题（动作字典）的实施方原话是"B/C 需要同 View 打印、保存、续读和 diff 同步采用"。每改一次表示，比较规则和文件格式都要跟着改。
- **建议的最小可行形态**：
  1. 自动保存 `snapshot`、`find` 和动作观察的**打印文本**，后缀用 `.txt`；`--out` 同样写文本。配额、锁和"新文件不覆盖"保留。这一部分有真实价值：模型在输出被截断时回读文件，github-new 和 github-old 两臂都这样做过。
  2. 删除 `--diff-from`、`view_file` 的解码和校验、`ui_diff`。
  3. `--raw` 保留，作为结构化出口。
  4. `--json` 模式下，如果保存失败，就把正文放进 `data.body`；不要让调用者两手空空（现在 JSON 模式加保存失败，正文在哪里都拿不到）。
- **能力损失**：失去"同一屏上的结构化变化列表"。目前没有任何真实任务用到它；动作结果里的 `change=… at …`（`ui_client.py:450-505`）已经覆盖"有没有变、变在哪个维度"。

### 3.2 结果与退出码合同：保留，但收紧文字

从源码重建的退出码真值表【源码，行号如下】：

| 退出码 | 产生条件 |
| ---: | --- |
| 0 | 结果为 `done`，没有显式 `exit`，也没有 `deliveryError`（`protocol.py:53-65`） |
| 1 | ① 结果为 `failed`，包括 `snapshot`、`find`、`screenshot` 主交付失败被改写成 `failed`（`ui_client.py:1446-1461`）；② 结果为 `done`，但命令行明确要求的观察或图片没交付（`ui_client.py:1473-1492`，显式 `exit=1`）；③ 结果为 `done`，但 stdout 或 stderr 写入失败（`protocol.py:65`）；④ batch 中某步退出 1；⑤ 未处理异常（D4） |
| 2 | 结果为 `not sent`，包括所有参数错误、计划校验失败（`cli_options.py:136-142`、`app.py:893-906`） |
| 3 | ① 结果为 `UNKNOWN`；② 本地连接关闭失败（`protocol.py:67-69`，结果保留）；③ 设备侧 UI 清理未确认（`AutomationCleanup.kt:59-66`）；④ WindowInfos 监听器注销未确认，此时同时为未发出（`WindowInputListener.kt:143-158`）；⑤ shell 清理未知或信号失败（`ShellProtocol.kt:156,165-167`）；⑥ 配置或凭据"已发布但同步或清理失败"（`workspace.py:47-54`、`coordinator.py:378-390,672-678,743-749`）；⑦ 文件作业发布已确认但清理未确认（`file_jobs.py:136,568`） |
| 4 | 结果为 `still running`（shell 或传输作业未结束、`daemon stop` 未确认） |

优先级：显式的 3 不会被交付失败改成 1，因为 `exit_code()` 只在原值为 0 时才改成 1。`finish_delivery_result` 也只在退出码为 0 时设 1。batch 取停止那一步的退出码，关闭失败再覆盖成 3。这几条与 `reference.md:68-91`、`batch.md:43-56`、`images.md:148-154` 一致。通知监听器注销失败已经不进主回执（`NotificationOperations.kt:91-96`、`notifications.md:39-42`）。

无设备下的实跑（`scripts` 外直接运行；每条都核对了 stdout 和退出码）：

| 命令 | 结果 | 退出 |
| --- | --- | ---: |
| `snapshot`（无配对设备） | not sent／deviceRequired | 2 |
| `snapshot --bogus`、`click`（缺目标）、`screenshot --image x`、`screenshot --region 0 0 10`、`--region 10 10 5 5`、`screenshot e1 --region …`、`--image-format gif`、`click e1 --image-format png` | not sent／arguments | 2 |
| `snapshot --print --json` | not sent | 2 |
| `snapshot --out 已存在文件` | not sent／fileExists | 2 |
| `snapshot --out 不存在目录/a.txt` | not sent／snapshotPath | 2 |
| `snapshot --raw --diff-from x` | not sent | 2 |
| batch：未知命令、重复输出路径、已存在输出、计划内全局选项、`--help`、JSON 下的 `--print`、空计划、计划文件缺失 | 整个计划 not sent | 2 |
| batch：第 1 步运行时 not sent | `batch: failed; Step 1 stopped: not sent (exit 2)`，只含一步前缀 | 2 |
| `devices >/dev/full`（文本或 JSON） | `delivery: done; stdout: [Errno 28]`，输出到 stderr | 1 |
| `devices >&-` | traceback | 1 |
| `use -s foo`（`.androidctl` 为 0755） | not sent／privatePermissions | 2 |

**与第三轮 §2.2 验收标准逐条对照**：

1. 动作完成、要求的截图没存成：单步和 batch 都是非 0 且结果字为 done。成立【源码；测试 `test_single_and_batch_share_required_delivery_exits`】。
2. 读类命令没交付文件：失败、退出 1、正文仍打印。成立【原件 native017】。
3. 动作完成、清理未确认：退出 3、结果 done、首行写明清理未完成和出路。成立。设备侧的 reason 前缀是 "UI cleanup unconfirmed; run session close to retry, or use Force executor exit"（`Sessions.kt:356`、`AutomationCleanup.kt:65`）；主机侧在行尾追加 "local connection cleanup unconfirmed; inspect with androidctl daemon status"（`output.py:49-55`），不在行首，但同在一行。
4. 通知监听器注销失败不改主回执。成立。
5. 公开表与行为一致。基本成立；不一致处见 D3（batch 结果字）和 D4（无结果行的退出 1）。

N3（关闭失败覆盖已得结果）和 N6（batch 关闭失败退出 1、后台屏流关闭失败不改退出码）都已修好【源码 `ui_client.py:1126-1138`、`ui_batch.py:276-288`、`TargetManagement.kt:79`、`UiRun.kt:153-158`】。

**从 LLM 调用者角度看是否好用**：

- 真实记录：84 次调用里没有出现 "done＋非 0"。5 次非 0 退出，模型都做了正确的下一步：未发出的 scroll 改了方向；失败的 wait 改用 find 或换了条件【原件 `ST/16-delivery/efficiency/maps-new/native.rollout.jsonl` 第 52-69 行；`ST/efficiency/*/cli-index.json`】。harness 按 SKILL 的要求把 `$LASTEXITCODE` 传了出来（`maps-new` 第 52 行）。所以这套合同在常见路径上没有带来负担，在罕见路径上也没有数据。
- 候选合同 B："退出码只表达主动作已执行、未执行或不确定（0／1／2／3），交付和清理问题只写一行警告"。
  - 得：结果字和退出码一一对应，`exit` 字段可以删掉。
  - 失：stdout 写失败也会退出 0，管道和包装脚本会以为拿到了输出；batch 要么在缺了明确要求的截图之后照常继续，要么得为"缺交付"另立一套停止判断，而第三轮正是为了删掉那套双规则才定了现在的合同；清理未确认（下一次会话可能被封锁）退出 0，也违背"退出 0 代表可以放心继续"的直觉。
- 候选合同 C："把清理未确认从 3 拆成另一个码（例如 5）"。得：3 只表示"不知道动作是否发生"。失：多一个码，而两种情形的正确反应都是"不要重发"，第三轮已经论证过不必拆。
- **判断**：第三轮方案不是错的，维持。可以做的简化是**文字层面**：`reference.md:68-91` 的两段 24 行，可以收成一张五行表（上面的真值表），加一句"结果字是 done 而退出码非 0：动作已经发生，只补缺的交付或处理清理"。其余例外移到各自领域的文件里。

### 3.3 评审方前几轮建议的自我修正

- **第三轮 §2.2 第 2 条"读命令没交付文件就是失败"，在文本模式下过严。** 对 LLM 来说，交付的主渠道是 stdout 正文；文件只是续读用的副本。D2 已经说明，一个与读取无关的权限问题因此变成"每次读取都失败"。建议改成：**保存文件是唯一交付渠道时**（`--json`、`--no-print`、`--out`），保存失败才算失败；默认文本模式下正文已经打印，结果维持 done，保存失败作为显式警告，退出 1 还是 0 可以再议，我倾向退出 0 加警告，与"自动历史不算明确要求的附件"一致。
- **第三轮 §2.2 第 9 条"启动落点未确认改为未知、退出 3"，与"结果只说主动作"自相矛盾。** `click` 的 done 只表示平台接受；`open` 的 done 却要求目标在目标显示上可见。真实验证里有 2 次 `open` 是 START_SUCCESS 之后报 UNKNOWN／退出 3，而目标 App 随后确实出现了（WPS 打开 PDF；API34 上的 VLC）。可见运行原件里共 11 次 open，8 次 done、1 次未发出、2 次 UNKNOWN【原件，见 `ST/16-delivery/runtime/mumu-view/commands/reading/004-…`、`ST/12-sessions/03-authority/preflight-fix/runtime/api34/commands/093-…`】。在 batch 里这会无谓地停下。建议：START_SUCCESS 报 done，落点作为 `landing=pending|visible|notVisible` 事实；只有平台返回不确定的代码（例如 `code1`）才报 UNKNOWN。这与待决第 1 题的"派发与观察分列"是同一原则。
- **第三轮 P9 曾承认比较功能没有原件**，结果第四轮实施方把它做成了完整的文件格式。评审方当时应该直接建议"不做比较"，而不是"换一种比较实现"。

### 3.4 `ui_client.py`（1,520 行）的职责与拆分

五种职责：

1. 请求事实（`requested_action`，约 50 行）；
2. 参数预检：截图区域、select、range、action、touch、expect、打印与 JSON 冲突（`prepare_ui`，约 200 行）。其中大部分在设备端还有一份，例如 `ImageRegion.validateRequest`。保留主机这一份是为了 batch 能在执行前整体拒绝，是合理的；
3. 结果行文本格式化：`result_line` 及其辅助函数约 500 行，按命令硬编码字段清单；
4. 附件分块下载协议：`blob`，含哈希、zlib、元数据一致性检查，约 100 行；
5. 交付、渲染、保存和退出码收尾：`deliver`、`render_observation`、`save_delivery`、`finish_delivery_result`，约 330 行。

建议按 3、4、5 拆成三个模块，单个模块就能读懂退出码逻辑。职责本身没有多余，但第 3 项的字段清单会随每个新命令增长，值得考虑"按领域文件注册字段"的方式。

模块规模（这一块 12 个文件）：基线 `7d5be70` 3,810 行，旧候选 `3f45af6` 7,389 行，本轮 `4235af9` 7,446 行。整个主机源码分别是 8,008、11,959、12,067 行。本轮这一块基本持平：`ui_batch` 少 100 行，`ui_diff` 少 91 行，`workspace` 少 96 行，`history` 少 58 行；新增 `view_file` 261 行，`ui_client` 多 84 行。公共数据结构有 `Result`、`Problem`、`Fault`、`View`、`UiInvocation`、`PreparedUi`、`DeliveryPaths`、`RenderedObservation`、`UiExecution`、`ReadingView`、`SavedHistory`，按 §3.1 删除比较后，`ReadingView` 可以退化成"渲染好的文本加记录"。

### 3.5 头部溯源行（与"阅读面"专题重叠，只从结果合同角度说）

最终两个新臂的 22 次调用里，头部行（`androidctl {…}`、`screen`、`window`、`edge-windows`、`events`）共约 28.7 KB，占 478 KB 输出的 6%；其中 `androidctl {…}` 一行平均约 530 字节【重算 `scripts/headers.py`】。真正的体积在正文：一次 `find May --all` 就是 5.8 万 token。做决定需要的只有 `screen`（App、标题、键盘）、是否采集不完整、窗口是否被遮挡。`androidctl` 行里的 `capture`、`connection`、`sequence`、`startedMs`、`windowsSource`（与采样时间重复）只服务于 UNKNOWN 资源核查和比较键。前者的流程本身就过度（`reference.md:93-113`），后者建议删除。建议这一行只保留 `cached`／`age`，其余放进 JSON 和保存文件。

### 3.6 batch

计划静态校验走真实解析器，重复输出路径、已存在的文件、全局选项、help 和 version 都会在连接前整体拒绝【实跑，见 §3.2 表】。停止规则只有一条：退出码非 0（`ui_batch.py:227,265-274`）。JSON 模式只在最后输出一个结果，包含已执行的前缀（`steps[]`、`planned`）。"动作 done 但退出码非 0"的那一步仍记为 done，batch 本身报 `failed`，取那一步的退出码。合同清楚。

但 batch 是旧候选新增的能力（基线没有），八次真实任务一次都没用过【重算】，代价是 317 行源码、1,599 行测试（128 个）和 86 行指引。我不建议删除（已经是用户确认过的能力），建议冻结范围，不再加新规则。

### 3.7 文件传输与作业

本轮只改了 `6d18ebc`：发布前先关闭写入器，并转移关闭所有权，避免清理时重复关闭同一个 fd。另外在 `job_commands.py` 里加了关闭失败转退出 3 的处理。改动小，方向正确【源码 `TransferHandle.kt:128-136`】。`jobs-files.md` 131 行，信息密度极高（例如 −1 不是退出码、清理报告不改执行值），对 LLM 来说偏重，但不在本轮范围。`shell -- echo hi` 会报"多余参数"，脚本必须是一个参数【实跑】，这一点对模型不够直观，可以在 help 里给出加引号的例子。

---

## 4. 证据审计

### 4.1 Settings 和 RFC 的 View 文件闭环

| 声称 | 核对 | 结论 |
| --- | --- | --- |
| 普通、find、interactive、200 字符预算四份文件，用生产解码器读回，统一 CRLF 之后与 stdout 正文完全一致；原始字节不相等 | 读了 `native-View-roundtrip.json`、`files/native-ordinary.json` 和 `commands/native/001` 原始 stdout；文件头、记录和尾部结构与源码一致；原始字节不等的原因（Windows 文本模式）也写明了 | 成立。样本：MuMu 1 台，Settings 4 份，网页 4 份 |
| 缺尾部的文件退回 `fileInvalid` | `native012` stdout：`full/fileInvalid`，结果 done／退出 0 | 成立 |
| 配额回收后同一路径变为 `fileMissing` | `native015` 得到 `changes`、`native016` 得到 `full/fileMissing`，退出均为 0；150 份是同一个 View 的副本，实施方自己写明了 | 成立，但边界清楚：这是人工预置的配额，不是自然积累 |
| 换连接后退回 `contextChanged` | `native009` | 成立 |
| 可逆滚动的差量 | `native010`、`011` 实际都是 `full/diffLonger` | 实施方如实记录了。但"闭环"只证明了退回机制正常，**没有证明差量有用**（§3.1） |
| 保存失败与正文分离 | `native013`、`017` 退出 1，正文照常打印 | 成立；同时它也是 D2 的证据 |
| `native007` 只是"过期拒绝" | 原件里还有空 View 的 `snapshotPath` | **部分成立**：漏掉了 D1 |

### 4.2 MuMu 720×1280 → 1080×1920 区域映射

`commands/formal/008` 和 `012` 原始 stdout：请求 `[23,401,691,681]`，原生 `[34,601,1037,1022]`，1003×421，`mappingKnown=false`。按 1.5 倍、左上取 floor、右下取 ceil 重算完全相同。WPS 细读的 `[75,778,649,966]` → `[112,1167,974,1449]`、862×282 也吻合。**成立。** 样本：MuMu（API 35）上 3 次矩形区域、API34 上 4 次矩形区域；`screenshot REF` 只有 API34 上 1 次（`api34-regions/commands/main/015`），我没打开它的图。"细节可读收益"实施方自己就没有声称（"不能写成普遍的小字收益"），措辞恰当。

### 4.3 WPS PDF 细读

`reading-result.md` 如实保留了：`open` 是 UNKNOWN／退出 3、先读时写错了许可证名、原生区域图纠正了这个错误。它也明确说"没有证明 JPEG 中该词不可读"。**成立，结论措辞克制。** 样本 1 次。

### 4.4 push、查询、pull 字节一致

我重算了 `ST/runtime/file-publication/source.bin` 和 `received.bin`：都是 98,317 字节，SHA-256 都是 `44bf8cca…`，`cmp` 相同；`result.json` 里的 push、job get、pull 三个回执的大小和哈希也一致。数据超过 32 KiB 分块大小，所以走了多块传输。**成立。** 运行时 `TransferHandle.kt` 已经包含 `6d18ebc`（此后再没改过）。只有正常路径，关闭异常和取消没有实跑（实施方写明了）。另外 WPS 段还有一次 331,986 字节的 push。

### 4.5 Linux 窄验和测试

- `linux-final/result.md`：WSL ext4 上跑了 27 个模块，1,428 个通过、15 个 Windows 专属跳过；源码是 `f146f32`，其主机部分与 `4235af9` 相同【源码】。**成立**，而且边界写得清楚（不是全部测试，也不是 macOS）。值得注意的是它跑在 ext4 上，**没覆盖 D2 的 drvfs 情形**。
- 总数：交付稿写 Windows 上 1,936 个通过、2 个跳过；我在 Linux 收集到 1,887 个。差额可能来自平台参数化，我没有核对。
- 抽查测试性质：`test_ui_batch.py`（128 个）、`test_ui_diff.py`（46 个）、`test_observation_files.py`（100 个）都用 `CliRunner` 跑真实 app，接本地假守护进程（`connection_test_support.local_server`），文件写在真实的 `tmp_path`。`monkeypatch` 主要用于注入失败，例如替换 `UiClient.close`、`ReadingView.render`，也有少量替换内部函数（`history.save`、`ui_view.project`）。**整体上是在验证可观察的行为**，不是复述实现。我在本机跑了其中 4 个文件：275 个通过、6 个跳过，29.9 秒。
- 测试有 25,573 行，源码有 12,067 行。这一块里，batch 测试 1,599 行，对应源码 317 行，比例约 5 比 1，而真实用量为 0；比较测试 573 行，用量也是 0。删除比较可以去掉约 600 到 800 行测试。测试名里混有内部代号（如 `…_stop_h4`），对后来的维护者不友好。

### 4.6 实施方 review 写着"0/0 关闭"的地方

`16-delivery/02-view/review.md` 以及集成 review 都没有发现 D1，也没有把 D2 当作产品问题；后者被归类为"实验准备成本"（交付稿第 95、140 行）。从产品角度看，"测试人员用普通 mkdir 建了目录就整条读路径失败"，正好说明真实用户也会踩到。

---

## 5. 对相关待决题的评价

- **第 1 题（后台屏 Activity 通知）**：只从结果合同看，A 让"平台已接受派发、页面也对了"的情形报 UNKNOWN／退出 3 并停下 batch，等于用退出码表达"验证门没通过"，与"结果只说主动作"相违背。B、C、D 都把派发和观察分开，更一致。实施方推荐 D，我在合同层面没有异议；保护强弱的取舍属于另一专题。
- **第 3 题（动作字典）**：实施方把"B 和 C 需要打印、保存、续读、比较同步改"列为成本。如果按 §3.1 删除 `--diff-from` 和 View 文件，这项成本只剩"打印和保存同一份文本"，B 和 C 的代价会下降。建议先决定 §3.1，再回答第 3 题。
- **第 2、4 题**：不在本专题范围。

---

## 6. 需要用户裁决的新取舍

1. **是否删除 `--diff-from` 和 View 文件格式**（§3.1）
   - 候选 A：删除，自动保存改为打印文本（推荐）。少约 500 行源码、约 600 到 800 行测试和约 70 行指引；回读的文件和看到的行一致。损失是结构化变化列表，目前零使用、零有效产出。
   - 候选 B：保留现状。
   - 候选 C：保留 JSON Lines 保存格式，只删比较。保存格式仍比文本大约 1.5 倍，而且仍要维护解码器，意义不大。
   - 剩余未知：有没有别的调用者（非 LLM 脚本）依赖 JSON Lines 格式。目前是 alpha 版本，按理没有。
2. **工作区缓存的权限策略**（D2）
   - 候选 A：新建时设为私有，已存在的只检查是否为目录（推荐）。
   - 候选 B：维持严格检查，但默认文本模式下保存失败不再让读命令失败。
   - 候选 C：维持现状。
   - 凭据目录不受影响。
   - 剩余未知：WSL `/mnt/c` 的实际表现需要实测一次。
3. **读命令在默认文本模式下，保存失败时结果字和退出码怎么定**（§3.3 第 1 条，修订第三轮 §2.2 第 2 条）
   - 候选 A：done 加警告、退出 0（推荐）。
   - 候选 B：done 加警告、退出 1。
   - 候选 C：维持 failed、退出 1。
4. **`open` 在 START_SUCCESS 但落点未确认时的结果**（§3.3 第 2 条，修订第三轮 §2.2 第 9 条）
   - 候选 A：done 加 `landing` 事实（推荐，与 click 一致）。
   - 候选 B：维持 UNKNOWN／退出 3。
   - 与待决第 1 题一起定。

实施细节（实施方自己可以定）：D1、D3、D4、D5 的修正；`ui_client.py` 的拆分；`reference.md` 退出码段落改成一张表；`androidctl` 头部行删去比较键字段。

---

## 7. 没能核实的内容

- 没有设备，没有实跑任何设备路径。设备侧行为全部来自源码和实施方原件（只有 MuMu 和 Android Studio API34 模拟器）。
- WSL `/mnt/c`、FAT、网络共享上的权限检查行为是推断。
- `screenshot REF` 的图像内容（API34 `015`）没有打开查看；REF 边界部分越出屏幕被拒绝，是从源码推断的。
- Ctrl-C 的退出路径没有实测。
- D3（batch 关闭失败时的结果字）只读了源码，没有用假守护进程实跑。
- Windows 的 1,936 个测试与 Linux 收集到的 1,887 个之间的差额，没有核对。
- 没有评估文件传输的取消、清理和替换分支（实施方也没有实跑）。
- 一次运行期间，本机为无设备探测启动了一个守护进程，`HOME` 设在 `agents7/39/home`，已经用 `daemon stop` 停止。对源码克隆跑测试时只生成了被 git 忽略的 `__pycache__`，`git status` 是干净的。
