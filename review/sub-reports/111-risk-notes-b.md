# 第七轮决定的风险提示（第 14 到 19 项、第 3、7、9 项）与旧决定核对

写给按用户第七轮决定修改 `f401deb` 的实施方，也作为第七轮决定文件第四节和"改动了哪些旧条目"一节的底稿。"第 N 项"指用户第七轮消息的编号（原文在 `S/agents12/USER-RULINGS-R7.md`）；"第六轮第 N 条"指 `androidctl-r6-decisions.md`；"第五轮第 N 题"、"第四轮第 N 项"指更早的决定文件。源码位置默认指 `f401deb`：Kotlin 在 `android/app/src/main/java/com/rainng/androidctl/`，Python 在 `androidctl/src/androidctl/`，指引在 `skills/androidctl/`，内部 docs 指 `dfe5e45`。

依据的性质：【源码】读源码；【实跑】在本机用 `f401deb` 的主机端运行（脚本在 `S/agents12/111/`）；【重算】离线统计；【推断】；【需设备】。Android 一侧没有设备、没有编译。

## 摘要

1. 第 14 项按用户原话只是把 `ui_view.py:120-125` 的一个条件对 `--all` 也生效（约一行）。实测这样只让 `--all` 输出少 32% 到 39%，本轮网页题那两份文件和维基长页仍全部超过 64 KiB 上限；报告第 7 题 A 的宽读法能少 58% 到 73%。两种读法请用户确认（待定 1）。
2. 第 14 项的三处提示里，`find` 省略记录被改写的问题早在上一轮候选就有；超限摘要拿不到命令名，要按命令分别写提示，"65536"是写死的字符串。
3. 第 16 项不能改共用的 `treeLimits()`（`UiObservation.kt:51-64`）：按选择器的动作和选择完整性判断也用它，改了就会把"按选择器的动作"一起改掉，而代定写的是不改。只能在 `UiCondition.kt` 里单独加。按现有时限算，默认 5 秒的 `wait` 在每次采集都截在 3 秒的页面上永远凑不齐"连续两次"，用户能看到的变化只剩超时回执里的原因（待定 2）。
4. 第 17 项要在 `WorkerConnection.kt` 的"已连接"记录上加一个不会被清掉的"自动重连而来"标记，不能复用会被清掉的 `recoveryPending`。`WorkerConnectionTest.kt:138` 断言连续重连三次，必须改。
5. 第 18 项有现成的 `clipboard clear`，但恢复状态未知时清空可能抹掉用户新复制的内容，所以指引那句要写"征得用户同意后"。固定说明放在设备端还是主机端，影响契约样本要不要重新导出。
6. 第 19 项上一轮候选并不是"只在锁屏时打印"：设置了锁屏密码的手机上，每次观察都会多打一行 `observation {"display":{"keyguardSecure":true}}`（实跑）。后台屏会话里，屏幕状态字段说的是后台屏，锁屏字段说的是整机。
7. 第 3 项：本轮 20 次运行没有保存原始界面树，对照样本拿不到这些输入。可用的输入是契约样本 116 条、50 份原始界面树（本轮 3 份加前两轮 47 份，已实测能渲染），以及在测试里录下来的首行。`result_line` 现在 280 行。docs 仓库现在 301,632 字节，要删一半，其中有几处是旧决定要求保留的。
8. 第 9 项：三种改法在本机 Linux 上都已实测通过，原测试照旧失败。最省事的一种只要在测试里临时调高上限常量，加两行。
9. 旧决定核对：用户列的 8 处之外，另找到 17 处被改动或了结的旧条目，表在第十节。

---

## 一、第 14 项：`--all` 不再打印无名动作编号；改三处提示；上限阈值不动

### 结论

- 按用户原话，最小的改法是把 `ui_view.py:120-125` 里"平台标准动作且没有名字"这一半移到 `not all_nodes` 之外，约一行。这就是把第六轮第 3 条 A1 的"无名动作一律省略"扩到 `--all`。
- 只做这一项省不下多少。本轮网页题保存的两份 `--all` 文件分别少 39% 和 32%，维基长页少 39%，四份都还在 64 KiB 以上。报告第 7 题 A 的宽读法是"`--all` 只多显示隐藏节点和超过 20 条的命中，属性按默认规则打印"，能少 58% 到 73%。用户说过"与报告写法不同时以本消息为准"，所以按原话做是读法 1；要不要更宽，列为待定 1。
- 三处提示都是改字符串，但超限摘要是在不知道命令名的函数里拼出来的，要按命令分别写。

### 现状

**`compact_actions`（`ui_view.py:106-154`）**【源码】

- 平台标准动作指编号是单个比特且不大于 2^21，或者落在 `0x0102xxxx` 段（`:117-119`）。
- 默认模式（`all_nodes` 为假）在 `:120-126` 跳过两类动作：一是在默认省略集合里、又没有标签的（集合在 `:55-69`）；二是平台标准动作、又不在命名表 `ACTION_NAMES` 里的（命名表在 `node_facts.py:42-72`）。
- `--all` 时这两个判断都不生效，`:141-146` 的默认省略也不生效。于是：
  - 编号 64、256、512、1024、2048 这类动作打印成 `action-name="action-40"`、`"action-100"`、`"action-200"`、`"action-400"`、`"action-800"`；
  - 命名表里有、但默认不打印的动作打印成 `+focus`、`action-name="select"`、`action-name="clear-selection"`、`+show-on-screen`、`action-name="context-click"`；
  - 动作为空的节点打印 `actions=[]`（`:219` 加上 `node_line` 的 `:665`）。
- "无名动作编号"准确地说，就是第二类在 `--all` 下打出来的 `action-name="action-<十六进制>"`。原始记录里大量出现的是 `action-40`、`-100`、`-200`、`-400`、`-800`；"滚动到位置"（`action-1020037`）、"显示提示"（`action-1020044`）也属于这一类。
- 最小改法是把条件改成：`(not all_nodes and identifier in DEFAULT_OMITTED_ACTIONS and not label) or (platform_action and identifier not in ACTION_NAMES)`。

**`--all` 用在哪些命令上，`all_nodes` 在各处是什么意思**【源码】

- 有 `--all` 的命令只有 `snapshot`（含按引用读 `snapshot REF`）、`find`、`locate`（`ui_commands.py:538, 720, 775`）。`--all` 也作为 `args.all` 发给设备，但设备端不读它（Kotlin 里没有读取处），所以这是纯主机端的改动，不用重新导出契约样本。
- `all_nodes` 一个参数管着五件事：

| 位置 | 作用 | `snapshot` | `snapshot REF` | `find` |
| --- | --- | --- | --- | --- |
| `ui_view.py:372` | 显示不可见节点 | 有 | 无（带引用时本来就显示） | 无（搜索时本来就显示） |
| `ui_view.py:860` | 取消 20 条命中的上限 | — | — | 有 |
| `ui_view.py:196-318` | 属性：每个节点都给引用，被父节点吸收的名字、`id`、冗余的 `chromeRole`、与名字相同的提示文字照打 | 有 | 有 | 部分（搜索时已经打印引用、名字和 `id`） |
| `ui_view.py:106-154` | 动作全打 | 有 | 有 | 有 |
| `ui_view.py:343` | 按引用读时，跳过"被祖先吸收的名字要在本节点重新显示"的计算 | — | 有 | — |
| `ui_client.py:847-857` | 窗口行打印全部字段 | 有 | 有 | 有 |

- 所以 `all_nodes` 的含义在三个命令上并不一致：对 `find` 主要是"不限条数"，对 `snapshot` 主要是"显示隐藏节点"，对 `snapshot REF` 只剩"多打属性"。

**两种读法下 `--all` 输出各少多少**

本轮 20 次运行只保存了渲染后的文本，没有保存原始界面树（第九节有说明）。所以网页题那两份按文本规则近似；维基长页有原始界面树（实施方本轮的交付检查），用 `f401deb` 的渲染函数实际渲染。

| 文件 | 现在 | 读法 1：只省无名平台动作 | 读法 1b：`--all` 也套默认的动作省略 | 宽读法：只多显示节点，属性按默认 |
| --- | ---: | ---: | ---: | ---: |
| 网页题第 1 次 `--all`（168,039 B） | — | 102,430（−39.0%） | 70,103（−58.3%） | 约 52,800（约 −69%） |
| 网页题第 5 次 `--all`（259,096 B） | — | 176,598（−31.8%） | 135,938（−47.5%） | 约 109,700（约 −58%） |
| 维基长页 12 号 | 843,448 | 511,813（−39.3%） | 405,674（−51.9%） | 226,121（−73.2%） |
| 维基长页 22 号 | 782,308 | 474,282（−39.4%） | 375,633（−52.0%） | 209,928（−73.2%） |

- 依据：网页题两行【重算】，`S/agents12/111/allsize.py`。读法 1 的数字是精确的，因为被删的记号形式固定；读法 1b 和宽读法的数字是按文本规则近似的。
- 依据：维基两行【实跑】，`S/agents12/111/wiki_all.py`，在 `f401deb` 的 `render_observation` 上替换 `compact_actions` 或 `attributes` 后实际渲染。不带 `--all` 时这两页是 32,845 和 32,468 字节。
- 结论：只有网页题第 1 次那份在宽读法下会降到上限以内。其余各份在三种读法下都还会超限，所以三处提示无论如何都要改。

**三处提示**【源码】

1. `find` 的省略记录（`ui_view.py:896-898`）。
   - 原文：每一条省略记录都被改写成 `read="snapshot --cached --all"`。
   - 被盖掉的是 `omission()`（`:682-694`）原本给的读法：带引用的省略记录写 `snapshot eNNN`，合并后的记录写 `snapshot --cached`。
   - 这段在上一轮候选里就有（`git blame` 指向 `fb75afc`），不是本轮新加的。
2. 超限摘要（`ui_client.py:883-889`）。
   - 原文：`Observation exceeds 65536 UTF-8 bytes; full text requires a saved file.` 后面接 `Narrow the current sample: androidctl snapshot --cached --depth 2`。
   - "65536"是写死的字符串，阈值常量另在 `ui_client.py:45`（`DEFAULT_BODY_BYTES = 65_536`）。
   - 摘要在 `render_observation` 里拼出来，那里拿不到命令名，只能从 `invocation["command"]` 取。
3. 首页 `SKILL.md:85-87`。
   - 原文：`Text results save the full body in .txt. Bodies over 64 KiB of UTF-8 text print a summary and file path; smaller bodies print in full. Read/search that file after host-tool truncation.`

### 建议改成什么（草稿，措辞由实施方定）

1. **`find` 的省略记录**：删掉 `:896-898` 的改写，恢复 `omission()` 给的读法。
   - 带引用的记录写 `read="snapshot eNNN"`。按引用读本来就会显示屏外后代（`:372`），对 `find` 来说够用。
   - 合并后不带引用的记录，如果写 `snapshot --cached`，等于把调用者引去读整页。这里宜不给读法，或者写 `find QUERY --context N`。
2. **超限摘要**：按命令分三种写。

| 触发超限的命令 | 建议的提示 | 说明 |
| --- | --- | --- |
| `snapshot`（含 `--all`）、`wait`、动作命令、`locate` | `Narrow it: androidctl find TEXT (reads this sample), then androidctl snapshot REF; search the saved file instead of reading it whole.` | 动作和 `wait` 的观察就是当前样本，`find` 读缓存不碰设备；动作未稳定时 `find` 会重新采集一次，也合适。 |
| `find` | `Narrow this find: a more specific query, fewer --context, or without --all; then snapshot REF.` | 再写 `find 关键词` 会让调用者原样重发。 |
| `snapshot REF` | `Read a child: androidctl snapshot CHILD_REF, or find TEXT.` | 与报告下-14 有关：按引用读一个很大的容器也会超限。 |

   `65536` 改成引用 `DEFAULT_BODY_BYTES`。

3. **首页那句**：`Search that file for the part you need; do not read it whole.` 同时可以加 `find TEXT reads the cached sample without a device call.`，两行以内。

### 最容易出错的地方

- **按读法 1 只改动作，三处提示照样必须改。** 上表四份文件在读法 1 下全部还超限。
- **宽读法要把 `all_nodes` 拆成两个参数。** 一个管显示节点和命中条数，一个管属性。
  - `:343` 那处名字计算要跟着"属性"那个参数走。否则 `snapshot REF --all` 在不多打属性之后，被祖先吸收的名字会消失。
  - `ui_client.py:848` 的窗口行要明确归到哪一边。
- **合并后的省略记录不要退回 `snapshot --cached`**，那等于另一种"读整页"的引导。
- **摘要要保留第六轮第 14 条落实要点要求的内容**：结果首行、溯源行、缺口行、`find` 的命中摘要、文件路径，再加一行可照抄的缩小范围的写法。
- 指引里 `observations.md:33` 和 `:78`（`--all and raw expose more facts`）的说法要随所选读法改。

### 连带要改的

- **主机测试**：`test_ui_view.py`、`test_observation_files.py` 里断言 `--all` 动作和 `read="snapshot --cached --all"` 的地方；`test_ui_delivery.py`、`test_ui_reading_delivery.py` 里断言摘要原文的地方。
- **契约样本**：不用改（纯主机端）。
- **指引**：`SKILL.md:85-87`；`observations.md:33, 36-40, 53-58, 70-78`。
- **内部 docs**：`reference/ui.md:7`（摘要写法）、`:55`（"省略记录保……真实缓存读取命令"）、`:63`（"`--all`也显示其它节点的广告"）。
- **码表**：不涉及。

### 建议的测试

- 同一棵树渲染 `--all`，断言不出现 `action-name="action-40"`、`"action-1020044"`；`copy`、`paste`、`cut` 和带标签的自定义动作仍在。
- 选宽读法时另加一条：`snapshot REF --all` 下，被祖先吸收的名字仍然显示。
- `find` 的输出里，带引用的省略记录写 `snapshot eNNN`，合并后的记录不写 `--all`。
- 对 `snapshot`、`find`、动作命令各造一次超限，断言摘要里的提示与命令对应，并且数字来自常量。
- 用维基那两份原始界面树做一条体积回归：渲染 `--all` 后断言字节数低于某个值，让以后有人放回动作编号时能被发现。

### 需设备

不需要。这几处都只改主机端渲染。网页题正式实测会顺带检验效果。

### 读法（待确认）

- 读法 14-1：`--all` 只省"平台标准动作里没有名字的"（读法 1）；其余动作、属性照现在全打。
- 读法 14-2：这一项改默认输出，排在对照样本和首行重构之后（第 3 项定的顺序）。

### 待用户定

**待定 1：`--all` 少打到什么程度。** 背景：用户原话是"不再打印无名动作编号"，报告第 7 题 A 是"`--all` 只做第一件事"，后者更宽。

- **A（按原话）**：只省无名平台动作。
  - 优点：改一行，`--all` 的其余含义不变。
  - 缺点：只省三到四成，网页上的 `--all` 照样触发上限。
- **B（报告的读法）**：`--all` 只多显示隐藏节点和全部命中，属性和动作按默认规则。
  - 优点：省六到七成，部分页面回到上限以内。
  - 缺点：`all_nodes` 要拆成两个参数，测试和指引改得多。想看全部属性只能读 `--raw` 的 JSON（`--json` 不含界面树）。
- **C（折中）**：动作套用默认的省略，其余属性照全打（表中读法 1b）。
  - 优点：省约一半，`all_nodes` 不用拆。
  - 缺点：超限照旧。

---

## 二、第 15 项：后台屏的安全要点并进首页，专题改为出问题时再读

### 结论

- 首页现在 136 行（`wc -l`），已经比"约 130 行"多 6 行。
- 并入后台屏要点大约净加 5 行；删掉 `find` 重新采集的规则净减 2 到 3 行；本轮其它几处改首页的地方加起来净增 0 到 1 行。合计约 138 到 140 行。要回到 130 行左右，还得再压 8 行上下，建议从专题索引里压。
- 没有任何一轮决定文件要求过"用后台屏前先读专题"。这句是实施方在上一轮候选里写进首页的，它落实的是第四轮第 10 项"后台屏三五行"和第五轮第 12 题"首页约 130 行，多数任务不必再打开专题"。

### 现状

**首页怎样要求先读显示专题**【源码】

- `SKILL.md:26-28`：`For a background screen use / session open --display virtual after reading [display requirements](displays.md). / It shares App/account state; moving existing tasks and ending the screen need care.`
- 专题索引 `SKILL.md:128`：`Read [displays](displays.md) for leases, virtual requirements and task movement.`
- 显示专题自己在 `displays.md:44` 又写了一遍 `Read the movement/destruction rules before choosing it.`

**"解释 `find` 何时重新采集的内部规则"是哪几行**

`SKILL.md:77-79`：

```
After an unsettled or still-loading action, the next `find` collects once in the
same scope unless a newer sample has replaced it. Zero cached matches also
trigger one fresh collection in the same scope and request budget.
```

- `:76`（`Reuse a sufficient observation. Cached/REF snapshots re-read the same sample.`）和 `:80`（`Collect a fresh snapshot for a transition, changed target or missing fact.`）是给调用者的做法，宜保留。
- 删掉 `:77-79` 之后，调用者仍能从首行看出这次 `find` 用的是哪份样本：`ui_output.py:89-94` 会打印 `cached` 或 `fresh`。
- 删掉后丢的是"命中 0 条时已经对新样本查过一次"这件事。不知道这一点的调用者，可能会在 0 条之后再补一次 `snapshot` 加 `find`。建议压成一句保留：`find reads the current sample and re-collects once when it is stale or has no match (shown as fresh).`
- `find --help` 里的同一条说明（`ui_commands.py:713-716`）不动。
- **被改动的旧条目**：第六轮第 11 条落实要点原文是"自动新采之后，回执要让调用者看得出这次结果来自新采的样本……首页和 `find --help` 里要补上这条行为"。首页那一半被代定删掉，或者压成一句；`--help` 那一半和回执上的 `fresh` 标记保留。

**`displays.md` 里哪些算安全要点**【源码】

| 要点 | 位置 |
| --- | --- |
| 创建前提：真实的 shell 权限、支持第二块屏和多窗口、主屏亮着且已解锁；输入时主屏要亮屏且已解锁、后台屏也要亮着 | `displays.md:30-33` |
| 与主屏共用 App 进程、数据、账号、播放；分开的任务不隔离状态 | `:42-43` |
| 移除后台屏会销毁上面的内容，包括没保存的界面 | `:43-44` |
| 别的屏上已有任务的 App 默认被拦；`--move-existing` 只在用户明确要求时用 | `:48-52` |
| 关闭、`--display main --replace`、`takeover` 销毁前先检查；被搬来的和确定外来的任务挡住销毁、报未发出；两条出路 `--return-tasks` 和 `--discard-tasks` | `:61-77` |
| 送回前先征得用户同意（会把 App 调到主屏最前面） | `:79-81` |
| 空闲到期、设备端进程退出或重启、在设备上释放会直接销毁，不做保护；`snapshot` 续期，`status` 不续期 | `:87-93` |

### 并进首页的草稿

只是草稿，替换 `SKILL.md:26-28` 的 3 行；措辞由实施方拟，用户过目。

```
1 Background screen: `session open --display virtual` needs main ON and unlocked.
2 It shares App processes, data and accounts with main; tasks do not isolate state.
3 An App with a task on another display is refused there; use `open --move-existing`
4 only when the user explicitly asks to move it.
5 Close, main --replace and takeover destroy it, including unsaved UI. Moved or
6 foreign tasks block that as not sent; ask the user, then add `--return-tasks`
7 or `--discard-tasks`. Idle expiry and worker exit destroy it without this check.
8 Read [displays](displays.md) when a virtual open, move or close is rejected.
```

8 行替换 3 行，净加 5 行。

### 首页行数的预算

| 改动 | 行数变化 |
| --- | ---: |
| 现在 | 136 |
| 第 15 项：后台屏要点（上面的草稿） | +5 |
| 代定：删 `find` 重新采集的规则（`:77-79`）；压成一句是 −2，全删是 −3 | −2 到 −3 |
| 第 14 项：`:87` 改写；加一句 `find` 读缓存 | 0 到 +1 |
| 报告下-7：`:62` 的 `reveal` 那句改写（如果实施方选择改指引，而不是在输出里加标记） | 0 |
| 报告下-20：`:22` 的列举里补 `apps` | 0 |
| 专题索引 `:128` 改成"出问题时再读" | 0 |
| **合计** | **约 138 到 140** |

要回到约 130 行，可以把 `:120-129` 的 10 行专题索引改成每行列两个专题，约 −5 行；再把 `:33-36` 关于外壳和退出码的 4 行压成 2 行。这只是建议。

### 最容易出错的地方

- **漏写"关闭会销毁"和"被搬来的任务挡住关闭、要问用户"**，就会出现调用者自己带 `--discard-tasks` 丢掉用户任务的情形。这两句不能省。
- **不要把"先读专题"写回去。** 专题第 44 行那句"选之前先读"也要一起改掉，否则读首页的调用者会被专题引回来。
- **到期销毁和"`snapshot` 续期、`status` 不续期"**：在等用户决定时会用到。首页可以只写"到期会直接销毁"，续期的细节留在专题里。
- 第五轮第 12 题要求每次固定任务跑完报告"读指引占总输入的比例"和"读完首页之后再读专题的次数"。这两个数正好用来验证这一项。

### 连带要改的

- `SKILL.md:26-28, 77-79, 128`；`displays.md:44`；`reference.md` 里如果有"先读显示专题"的交叉引用也要改。
- 内部 docs：`reference/ui.md:33` 讲 `find` 重新采集的规则不动，那是内部规格。
- 没有代码、码表和契约样本要改。

### 建议的测试

- 指引没有自动测试。建议加一条主机测试：首页行数不超过某个值（例如 135）。另一条只检查首页里含有 `--move-existing`、`--return-tasks`、`--discard-tasks` 三个写法。它防止以后删减时把要点删掉；但它也是一条新规则，是否值得由实施方定。

### 需设备

用户要求在 Android Studio 模拟器上用后台屏题验证。建议看这几件事：

1. **成功和关闭**：5 次里成功几次；每次结束时 `session close` 是否完成。如果被拦下，调用者是先问用户，还是自己带了 `--discard-tasks` 或 `--return-tasks`。
2. **专题被读的次数**：从原生记录里数 `displays.md`、`reference.md` 的读取次数（上一轮是 5 次里 5 次读显示专题、4 次连带读参考专题），以及读指引占总输入的比例（上一轮约 16%）。
3. **误关、误搬**：有没有在用户没要求时用 `--move-existing`；有没有销毁了不是本会话打开的任务。
4. **补一种情形**：后台屏题本身未必会碰到"App 已在主屏开着"。建议另做 1 到 2 次"主屏先开着 Contacts 再开始任务"的检查，看调用者被拦之后是不是停下来问用户。这不是正式实测，只是验证首页要点够不够用。

### 读法（待确认）

- 读法 15-1："首页保持约 130 行"读作 130 到 135 行，到不了时写明实际行数和原因（与代定里"达不到时写明实际数字和原因"一致）。
- 读法 15-2：删 `find` 那几行时保留一句"`find` 读当前样本，过期或 0 条时重新采集一次（首行标 `fresh`）"，不全删。

---

## 三、第 16 项：按采集时限截断也算截断，等待消失和等待空闲提前失败并写明原因

### 结论

- 只能在 `UiCondition.kt` 里改，不能改共用的 `treeLimits()`。
- 现在"以同样方式截断"比较的只是截断原因的集合（例如 `{nodeLimit}`），不看截在哪个位置。所以每次截在不同位置也算"同样方式"，这一点对采集时限截断照样成立。
- 按现有时限算，默认 5 秒的 `wait` 在每次采集都截在 3 秒的页面上，凑不齐"连续两次"：第二次采样会被请求截止切断，截止后完成的采样又按规定不评估。真正会提前失败的，只有调用者写了约 7 秒以上 `--timeout` 的时候。默认用法下，调用者能看到的变化只剩超时回执里的原因。见待定 2。
- 回执复用现有的码 `collectionLimit`（它已经在码表里），码表里这一行的说明要改。

### 现状

**提前失败认哪些截断**【源码】

- `UiCondition.kt:31`：只有 `gone` 和 `idle` 两种条件会因截断提前失败。
- `UiCondition.kt:337, 445-452`：每次评估都取 `treeLimits(tree)`，与上一次取到的集合相同、并且非空，就算"连续两次以同样方式截断"。这时如果没有命中，并且所有条件都是 `gone` 或 `idle`（第六轮第 17 条第三次答复：多个条件里只要有一个"等待出现"类，就不提前失败），就以 `limits.first()` 作原因码返回失败，文字是 `Two consecutive observations hit … before the wait condition matched`。
- `:485` 的 `observationFailed` 会把连续性清零。
- `UiObservation.kt:51-64` 的 `treeLimits()` 只返回 `nodeLimit` 和 `treeByteLimit`。

**采集时限截断怎么表示、怎么触发**【源码】

- 每次采样的截止时刻是 `min(until, 请求截止, 开始时刻 + 3000ms)`（`UiBudget.kt:59-69`；`COLLECT_MS = 3000L` 在 `:111`）。
- 超过截止时，`TreeCollector.kt:287-292`（节点间）和 `:349-351`（读子节点时）记 `collectionLimit`，`CollectedChildren.kt:11` 同理。
- 它进入 `missing` 的 `reason`，也可能进入 `problem.code`。
- 注意：**截在请求截止上的采样也记 `collectionLimit`**，与截在 3 秒上限上的记法相同，代码里分不出来。

**还有两处用 `treeLimits()`**【源码】

- `UiActions.kt:922-929`：按选择器的动作，截断时第一次采样后就失败。
- `UiSelector.kt:87-104`：选择完整性判断，即第五轮第 17 题"没有可见节点缺子节点时按普通唯一性规则执行"。
- 如果把 `collectionLimit` 加进 `treeLimits()`，这两处会一起变：按选择器的动作在大网页上会第一次采样就失败。代定写的是"下-12 只做第 16 项"，所以这是**必须避免的误改**。

**默认 5 秒时的时间线**【源码推断】

- 0 到 3 秒：第一次采样，截在 3 秒上限上。评估，记下 `{collectionLimit}`。
- 约 3.25 秒：第二次采样开始，截止是 `min(3.25+3, 请求截止约 4.5 秒)`，于是截在请求截止上。
- 评估时 `budget.remaining()==0`，走 `UiCondition.kt:404-406` 的 `snapshotBudget`（`:365-367`、`:436-438` 是同类检查），也就是"截止后才完成的采样不评估"，并且清零连续性。
- 结果是超时，不会提前失败。
- 调用者写 `--timeout 10s`：约 6.3 秒时第二次采样完成并被评估，两次都是 `{collectionLimit}`，于是提前失败。写 60 秒也是约 6.3 秒就返回。

### 最容易出错的地方

1. **改 `treeLimits()` 本身**：会连带改按选择器的动作和选择完整性（见上）。正确做法是在 `UiWait.evaluate` 里另算一个本地集合，`treeLimits(tree)` 再加上"本次采样截在 3 秒上限上"。
2. **把截在请求截止上的采样也算进来**：多数情况下这种采样不会被评估，但 `reserveDelivery` 会挪动截止（`UiBudget.kt:22-26`），边界上可能被评估。建议采集时在 `missing` 那条里加一个区分字段（例如 `basis: "sampleCap"`），或者用 `budget.sampleWork() >= COLLECT_MS` 来判断，只认 3 秒上限造成的截断。
3. **普通页面偶尔一次慢**：要连续两次都截在 3 秒上，才会提前失败，所以偶尔一次慢挡得住。但在采集本来就要 2.9 到 3.1 秒的页面上，会经常两次连续截断。这类页面在完整的树上本来也很难证明"消失"，影响不大。
4. **`predicate.limits` 会变**：`UiCondition.kt:446` 会把本地集合写进 `predicate`。"等待出现"类条件在大页面上超时时，首行的 `match=` 会多出 `"limits":["collectionLimit"]`。这正好满足第六轮第四节第 8 处"超时回执要主动补上树被截断"，但也是一处默认输出的小变化。
5. **原因码**：用 `collectionLimit`（`limits.first()` 自然就是它）。
   - 码表里这一行现在是 `category: internal`，说明是"把它当作观察缺口"（`contract/reason-codes.json:28`），对等待提前失败不合适，要补一句等待提前失败时的下一步。
   - 不能新增码（第 21 项、第六轮第 24 条）。改这一行的说明属于"提示文字"，第 21 项允许。类别要不要从 `internal` 改成 `observe`，见读法 16-3。
6. **原因文字**：建议写成 `Two consecutive samples stopped at the 3 s collection limit before the condition matched; absence or idle cannot be proven on this page. Wait for content that appears (--contains, selector, --app, --screen-change), or narrow with --window/--in.`
   - Chrome 这类页面的整页就在一个窗口里，`--window` 不一定有用，所以把"改用正向条件"写在前面。

### 连带要改的

- **Kotlin**：`UiCondition.kt:445-452`。可选：在 `TreeCollector.kt` 里给时限截断加区分字段。`UiObservation.kt` 不动。
- **JVM 测试**：`UiCondition` 的测试补三条：两次连续 3 秒截断时 `gone` 提前失败；`any-of` 里含一个 `--contains` 时不提前失败；第二次采样截在请求截止上、被丢弃时不提前失败，连续性清零。另补一条：按选择器的动作在 `collectionLimit` 的树上行为不变（防误改）。
- **契约样本**：现有 `waitResults` 里有一条 `nodeLimit` 的提前失败样例（`contract/wire-contract.json:225-265`）。可以不加新样例；要加就按 `AGENTS.md` 的流程导出、核对、替换。
- **码表**：`collectionLimit` 一行补一句。
- **指引**：`observations.md` 里讲等待和截断的那句。
- **内部 docs**：`reference/ui.md:236` 现在明写"collectionLimit、provider故障及其它partial不当节点/字节限额"，要改；`:111` 讲按选择器的动作，不变。

### 需设备

- 在长网页上用 `--timeout 10s` 跑一次 `wait --gone`、一次 `wait --idle`，看是否约 6 秒返回、原因文字对不对。
- 再用默认 5 秒各跑一次，看超时回执里有没有截断原因。
- 在普通原生页面上跑 `wait --gone`，确认不会误报。
- 报告 §7 的补验（"在长网页上跑一次等待消失和一次 0 条的 `find`"）可以一起做。

### 只记录、不改的清单（代定：下-12 只做第 16 项）

- `find` 的 5 秒时限不改。大页面上缓存 0 条后重新采集要 4 到 4.6 秒，可能超时；回执写明"缓存 0 条，重新采集失败"，不会被读成"页面上没有"。
- "截止后才完成的采样不评估"不改（`UiCondition.kt:365-367, 404-406, 436-438`）。这也是默认 5 秒下凑不齐两次的直接原因。
- 滚动查找 `scroll-until` 的第一步：截断的树上会等满时限（第五轮第 7 题的"界面树不完整时在下一次完整后继续"），回执不说原因。
- 按选择器执行的动作（`UiActions.kt:910-945`）：`collectionLimit` 不算截断，大网页上会等满时限。
- 报告附录一那次 `find`：找到 1 条，因附带的窗口列表超时而整条报失败。

### 第六轮第 17 条和第四轮第 16 项原文，以及改了哪一句

- **第四轮第 16 项**原文："截断是节点上限或字节上限造成的时候，按名字的命令在第一次采样之后就失败……`wait` 要连续两次以同样方式截断才这样处理。"
  - 这一项改的是"截断是节点上限或字节上限造成的时候"这个前提：对 `wait` 的 `gone` 和 `idle`，加上采集时限。
  - 按名字的命令那一半（第五轮第 17 题收窄过）不变。
- **第六轮第 17 条**原文：裁决是"'等待出现'不因截断提前失败；……'等待消失'维持严格"；答复是"`idle` 与'等待消失'同样保留提前失败"；第三次答复是多条件规则。
  - 这一项扩大的是这里"截断"的含义。条件范围（只有 `gone`、`idle`）、"连续两次"、多条件规则都不变。
- 第六轮第 7 条的表里，"截时的唯一性例外"一块不在第 21 项冻结的四块之内，所以给它加这条规则不违反第 21 项。

### 读法（待确认）

- 读法 16-1："按采集时限截断"只指截在每次采样 3 秒上限上的，不含截在请求截止上的。
- 读法 16-2："连续两次"照用，"以同样方式"仍按截断原因的集合比较。
- 读法 16-3：复用 `collectionLimit`。码表这一行补一句等待提前失败时的下一步，类别不改（`internal`）；要改成 `observe` 的话属于码表规则的改动，请用户点头。

### 待用户定

**待定 2：默认 5 秒的等待在这类页面上怎么办。** 背景：按现有时限，"连续两次"要约 6.3 秒才凑得齐。默认 5 秒的 `wait --gone` 在每次都截在 3 秒的页面上，照样等满 5 秒，只是回执可以写明原因。

- **A（按原话，推荐）**：照用"连续两次"，另外让超时回执带上截断原因（`limits` 和一句说明）。
  - 优点：不加新规则，误伤最少。
  - 缺点：默认用法下不会"提前"，只是"写明原因"。调用者写长时限时才省时间。
- **B**：对 3 秒上限造成的截断，一次就提前失败。
  - 优点：默认 5 秒下约 3 秒就返回。
  - 缺点：只是偶尔慢一次的页面也会报失败，与"连续两次"的旧规则不一致。等于又加一条规则。
- **C**：把截止后完成、同样截在上限上的那次采样也算作"第二次"。
  - 优点：默认 5 秒下约 4.5 秒返回。
  - 缺点：改动"截止后才完成的采样不评估"这条代定要保持的规则，不推荐。

---

## 四、第 17 项：自动重连来的进程 60 秒内又退出，就不再自动重连

### 结论

- 要在"已连接"记录 `Attached` 上加一个不会被清掉的标记，表示"这个进程是自动重连来的"。在 `onServiceDisconnected` 里，这个标记为真、并且存活不到 60 秒时，不重绑，并写一条专门的失败原因。
- 用户在手机上点刷新、或者 Shizuku 重新就绪而触发的绑定，都会走 `bind()` 的默认参数 `recovering=false`。所以手动起来的进程天然重新获得自动重连的资格。

### 现状：状态机【源码】

- `WorkerConnection.kt:49-110`：每次绑定是一个 `Attempt`，带 `recovering`，表示是不是自动重连发起的。
- `:75`：附着成功（读状态、对账都通过之后），记 `connectedAt = now()`，并把 `recovering` 存成 `Attached.recoveryPending`（`:279-289`）。
- `:95-109`：断开时重绑的条件是：之前确实连上了，不是预期退出，并且 `now() - connectedAt > MIN_CONNECTED_MS`（5000，`:293`）。
  - 5 秒从附着成功算到断开回调到达，包含回调的延迟。
  - 不重绑时写 `lastFailure = "Worker disconnected; use Refresh status on the device"`；预期退出时写 `null`。
- `:116-147`：`bind(recovering)` 新建 `Attempt`，把 `lastFailure` 清空。所以自动重连进行中，`requireTarget` 报的是兜底文字 `"Worker is unavailable; check the device App"`（`:274`）。这就是报告下-8 那句话。
- `:177-181`：`recoveryPending` 只在下一条返回 `sessionRequired` 的命令上取出，取出后清掉。
- `:226`：`destroyAndConfirm` 把 `expectedStop` 置为真。"停止空闲执行器"和"强制退出执行器"都走这里。
- 谁会调用 `bind()`：
  - `start()`，即 Shizuku 的服务就绪和权限授予回调（`WorkerBinding.kt:37-41, 54-58`）；
  - 用户的"刷新"（`GatewayService.kt:199-203`）；
  - 自动重连（`:108`）。
  前两种都不带 `recovering`。

### 建议的改法

- `Attached` 加 `val autoRecovered: Boolean`，取 `Attempt.recovering`。它与 `recoveryPending` 分开，因为后者会被清掉。
- 断开时增加一个判断：`previous.autoRecovered && alive <= 60_000` 时不重绑，并写专门的原因。原因文字草稿：`Worker exited again within 60 s of an automatic reconnection; it will not be reconnected automatically. Ask the user to use Refresh status on the device.`
- 原因码仍是 `workerUnavailable`（`requireTarget` 用 `readFailure()` 的文字，`:274`）。

### 组合表

| 情形 | 5 秒规则 | 60 秒规则 | 结果 |
| --- | --- | --- | --- |
| 手动启动的进程活了 3 秒退出 | 不满足 | 不适用 | 不重连，原因是现有的 `Worker disconnected; use Refresh status`（与现在相同） |
| 手动启动的进程活了 10 秒退出 | 满足 | 不适用 | 自动重连；新进程记为自动重连来的 |
| 上一行的新进程活了 4 秒退出 | 不满足 | 也不满足 | 不重连；用哪句原因文字，见读法 17-3 |
| 上一行的新进程活了 30 秒退出 | 满足 | 不满足 | 不重连，原因是新文字"60 秒内又退出" |
| 上一行的新进程活了 2 分钟退出 | 满足 | 满足 | 再自动重连一次；第三个进程同样按 60 秒判断 |
| 自动重连来的进程被"停止空闲执行器"停掉 | — | — | 预期退出，不重连（与现在相同） |
| 不再自动重连之后，用户点了刷新 | — | — | 新进程是手动的，重新获得资格 |
| Shizuku 重启，应用收到服务就绪回调重新绑定 | — | — | 按手动处理，重新获得资格（读法 17-2） |

### 最容易出错的地方

- **复用 `recoveryPending` 判断**：它在第一次 `sessionRequired` 之后就被清掉，按下面的补充要求还会被成功的会话命令清掉。之后再判断就当成手动进程了。
- **60 秒的起点**：用 `connectedAt`（附着成功时），与 5 秒规则同一个起点。如果从开始绑定时算，附着本身的耗时会被算进去。
- **边界**：5 秒规则写的是 `>`。60 秒宜同样写成"存活大于 60 秒才重连"，测试覆盖 60000 和 60001。
- **原因文字要和报告下-8 区分开**：
  - 下-8 要求连接进行中时回执改成"正在连接，几秒后重试"，条件是 `attempt.attaching` 并且未超过 `BIND_TIMEOUT_MS`；
  - 本项是"不会再自动连，要找用户"。
  两者同用 `workerUnavailable`。码表这一行（`contract/reason-codes.json:150`，现在只写"请用户刷新"）要按文字分两句：写 connecting 的，几秒后重试；其它的，请用户刷新。
- **崩溃循环的上界变了**：现在每次至少间隔约 6.5 秒，无限重连；改后，存活超过 60 秒的进程仍会被一直重连，最快每分钟一次。这是"60 秒内又退出"这个条件的直接结果，见读法 17-1。

### 第一次成功的会话命令清掉重连提示（报告下-15 的第一处）

- **在哪里取、在哪里清**：只在 `WorkerConnection.call()` 里（`:149-181`）。回复是 `sessionRequired` 时取出、附上、清掉。
- **建议**：在 `call()` 成功解码之后，如果请求的命令在 `RpcMethods.SESSION_COMMANDS`（`session.open`、`session.close`、`session.takeover`，`RpcMethods.kt:24`）里，并且结果是 `done`，就清掉 `recoveryPending`。
- **"会话命令"的两种读法**：
  - 只算 `session open` 和 `takeover`（读法 17-4，推荐）；
  - 任何成功进入会话层的命令。但重连之后，任何界面命令要成功都得先有新会话，而开会话本身就会先清掉提示，所以两种读法实际差别很小。
- **多调用者时的已知缺口**：提示是每个设备端进程一份，不是每个调用者一份。别的调用者先开了会话，提示就被清掉了；本调用者之后碰到的 `sessionRequired` 不再带提示。现在的实现本来也是先到先得。

### 自动化接口被占时补一句（报告下-15 的第二处）

- **回执原文**：`AutomationCleanup.kt:36-42`。用 `busy` 加主体 `automation`，文字是 `Another automation tool holds the UiAutomation registration; takeover has no effect; wait for it to exit`。`UiBackend.kt` 只是调用它（`:532` 一带的注册流程）。
- **加一句之后**：`…; takeover has no effect; wait for it to exit, and if it does not exit, ask the user to stop it`。
- **码表** `busy` 一行（`reason-codes.json:15`）：在 `For subject=automation, takeover has no effect: wait for the other automation tool to exit and retry the unsent command.` 之后补 `If it keeps holding the registration, ask the user to stop that tool.`
- **指引**：`SKILL.md:24-25`、`connection.md:96-97` 各补半句。
- **JVM 测试**：`UiCleanupTest.kt` 断言了这句原文，要改。

### 连带要改的

- **Kotlin**：`WorkerConnection.kt:75, 95-109, 149-181, 279-289`；`AutomationCleanup.kt:39`。
- **JVM 测试**：
  - `WorkerConnectionTest.kt:138-160` 的 `everyMatureUnexpectedDeathQueuesOneBindAndNeverReplaysARequest` 现在让每个重连来的进程活 5001 毫秒、重连三次。新规则下第二次就不再重连。要改成让重连来的进程活 60001 毫秒，另加一条"30 秒内又退出就不重连"。
  - `:239-277` 的提示测试补"`session open` 成功之后提示被清掉"。
  - 再加"手动刷新后重新获得资格"。
- **码表**：`workerUnavailable`、`busy` 两行。
- **指引**：`connection.md:105-110`（"attached for more than five seconds gets one automatic bind attempt"，补 60 秒的例外）；`SKILL.md:24-25`。
- **内部 docs**：`reference/connection.md:205` 一带的重绑说明。
- **契约样本**：不涉及。重连是设备端应用内部的事，提示文字没有进契约样本。

### 需设备

- 能让设备端进程在附着后约 30 秒自己退出的手段：调试构建里加一个开关，或者用 `adb shell kill` 杀进程（杀进程不是"预期退出"）。用它确认不再重连、回执文字正确。
- 再确认用户在手机上点刷新后能连上，之后再被杀时仍会自动重连。
- Shizuku 重启时的表现（会不会被当成手动重新获得资格）也需设备。

### 读法（待确认）

- 读法 17-1：60 秒按每个自动重连来的进程各自判断。活过 60 秒的仍按"意外退出就重连"处理，所以每分钟一次以下的崩溃仍会被一直重连。
- 读法 17-2：用户点刷新，以及 Shizuku 重新就绪触发的绑定，都算手动，重新获得资格。
- 读法 17-3：自动重连来的进程活不到 5 秒就退出时，两条规则都不重连，原因文字用新的"60 秒内又退出"那句，因为它比旧句多说明了已经自动重连过一次。
- 读法 17-4："第一次成功的会话命令"指结果为完成的 `session open` 或 `session takeover`。

---

## 五、第 18 项：借用粘贴的回执加一句固定说明，指引加一句

### 结论

- 改动只限提示文字：不改恢复规则、主结果、退出码，不新增原因码。
- 固定说明有两个放法，影响要不要重新导出契约样本，由实施方选（读法 18-1）。
- 产品有 `clipboard clear`，但它会连用户新复制的内容一起清掉，所以指引那句要写成"征得用户同意后再清"。

### 现状【源码】

- `BorrowedPaste.kt:334-360` 的 `save()`：粘贴已核对一致（`verified`）时主结果是完成；恢复状态是 `failed` 时退出码 1，`pending` 或 `unknown` 时退出码 3（`:346-351`）。
- 随后 `:352-353`：主结果是完成，就删掉 `reason`。这就是"`reason` 被删掉的那两行"。所以这两种情形的回执里没有任何一句话。
- 主机端首行（`ui_output.py:56-69`）只在有 `reason` 并且"未完成或退出码非零"时打印原因，加上 `borrowed_paste_facts`（`:450-477`）打印的 `effect=matches; paste=attempted; restore=failed`，以及恢复的错误文字 `restoreReason=`。
- 恢复状态的来源（`:259-312`）：
  - `failed`：恢复调用被拒绝，或恢复后读回与原内容不同；
  - `unknown`：读不到，或调用之后出了异常；
  - `skipped`：剪贴板已被用户换成新内容，或确认为空。不会泄漏，不需要说明。
- 契约样本里有两条相关样例：`pasteResults` 的 `restore-failed`（完成，退出码 1，无 `reason`）和 `restore-unknown`（完成，退出码 3，无 `reason`）。

### 产品有没有"清空剪贴板"

有。`clipboard clear`（`clipboard.md:3-7, 25-28`）。它需要当前会话，结果靠读回判断完成、失败或结果未知，并且"不先读原内容"。

所以指引那句可以照做，但要注意两点：

- 恢复状态未知时，剪贴板里可能已经是原内容，或者是用户新复制的内容，这时清空会把它们抹掉；
- 清空本身也可能结果未知。

建议放在 `input.md:61-63` 那段之后：`If the text was sensitive and restore is failed or unknown, tell the user the clipboard may still hold it; run clipboard clear only with the user's agreement, because it also discards whatever is there now.`

### 固定说明放在哪里

- **A，设备端**：在 `:352` 的分支里，对 `verified` 并且恢复状态是 `failed`、`pending`、`unknown` 的情形，保留一句 `reason`，例如 `Paste matched; the clipboard may still hold the typed text because restoration failed/is unknown`。
  - 首行会自动打印它，因为退出码非零，`ui_output.py:58` 成立。
  - 契约样本那两条的 `reason` 会变，要按 `AGENTS.md` 重新导出并跑两端契约测试（第六轮第 22 条落实要点也要求过这两条随改动重导）。
- **B，主机端**：在 `borrowed_paste_facts` 里，对恢复状态为 `failed` 或 `unknown` 的情形追加一句固定文字。
  - 不改设备端，不重导契约样本。
  - `--json` 的使用者看不到这句，只能看到 `restore.state`。

### 最容易出错的地方

- **改到主结果或退出码**：这一句只能是文字。设备端做法里不能顺手改 `:338-344` 的判断。
- **让 `reason` 在完成时出现会不会被别处当成失败**：主机首行的规则已经允许"完成加退出码非零时打印原因"（`f401deb` 修的就是这个），所以不会。但测试 `test_wire_contract.py` 里对这两条样例的首行断言要随着改。
- **第 21 项的边界**：借用粘贴的恢复规则属于四块之一。这一项是用户明确点名照做的提示文字；小项清单里任何"顺便改恢复规则"的条目都不做。

### 连带要改的

- 选 A：`BorrowedPaste.kt:352-358`；`BorrowedPasteTest.kt`；契约样本两条（重新导出）；`test_wire_contract.py` 对应的首行断言。
- 选 B：`ui_output.py:450-477`；`test_result_output.py` 或 `test_text_input.py` 补断言。
- 两种都要改：`input.md:61-66` 加一句；内部 docs `reference/ui.md:197` 或 `:381` 补半句。

### 建议的测试

完成并且恢复失败、完成并且恢复状态未知这两种情形，首行出现固定说明；恢复成功（`restored`、`cleared`）和 `skipped` 不出现；退出码仍是 1 和 3。

### 需设备

不需要。借用粘贴在两轮 57 次运行里都没用过，设备上很难让恢复失败。

### 读法（待确认）

- 读法 18-1：固定说明放主机端（B），不重导契约样本。选 A 也可以，代价是一次契约重导。由实施方选并写明。
- 读法 18-2：说明只加在"粘贴已核对一致、恢复失败或状态未知"这两种情形，即报告下-16 说的那两种；主结果不是完成时，现有的 `Borrowed paste did not complete: …cleanup=failed` 不另加。

---

## 六、第 19 项：恢复成只在锁屏或熄屏时打印的短标记

### 结论

- 设备每次都把屏幕和锁屏状态放在界面树的 `display` 里发给主机（`UiBackend.kt:410`，`DeviceFacts.kt:13-42`），本轮主机只是不再打印。所以这是纯主机端改动，改 `ui_view.py` 的 `header()`，几行。
- 上一轮候选并不是"只在锁屏或熄屏时打印"，见下面的实跑。

### 现状

- **`f401deb`**：`metadata()`（`ui_view.py:620-634`）把 `snapshotFacts` 原样传下去；`header()`（`:903-935`）只打印 `partial`、`missing`、`unavailable`、`limits`、`treeProblem`，不看 `display`。【源码】
- **上一轮候选**：`metadata()` 打印 `display` 里与默认值不同的字段，默认值是亮屏、交互、用户 0、未锁、`keyguardSecure=false`、主屏、屏幕状态 2。【实跑，`S/agents12/111/hdr6.py`，用 `fb75afc` 的主机端】

| 设备状态 | 上一轮候选打印的 `observation` 行 |
| --- | --- |
| 主屏正常，手机设了锁屏密码 | `observation {"display":{"keyguardSecure":true}}`，**每次观察都有** |
| 主屏锁屏 | `{"display":{"keyguardShowing":true,"keyguardSecure":true,"deviceLocked":true}}` |
| 熄屏 | `{"display":{"interactive":false,"displayState":1,"keyguardShowing":true,…}}` |
| 后台屏正常 | `{"display":{"displayId":7,"keyguardSecure":true}}`，每次都有 |

- 模拟器通常没有设锁屏密码，所以实测时看不到这行噪声；真机上会每次都有。

### 字段的含义【源码】

| 字段 | 来源 | 说的是 |
| --- | --- | --- |
| `interactive` | `PowerManager.isInteractive`（`DeviceFacts.kt:19`） | 整机，近似主屏是否亮屏可交互 |
| `keyguardShowing`、`keyguardSecure` | 窗口服务（`:35-36`） | 整机锁屏 |
| `deviceLocked` | 信任服务，按当前用户（`:37-40`） | 整机，当前用户是否锁定 |
| `userId` | 当前前台用户 | 整机 |
| `displayId`、`displayState` | `getDisplay(目标屏)`（`:22-31`） | **目标屏**：后台屏会话里是后台屏 |

- 后台屏会话里，这份 `display` 不包含主屏的 `displayState`。但后台屏上的输入要求主屏亮屏且已解锁（`DeviceFacts.kt:76-81`，`UiBackend.kt:84-88`），主屏的亮灭只能从 `interactive` 近似看出。
- 输入的拒绝条件（`DeviceFacts.kt:69-74`）：`userId != 0`、`interactive` 为假、`keyguardShowing` 或 `deviceLocked` 为真或读不到。

### 短标记的建议形式（草稿）

- 正常时不打印。
- 异常时在第一行 `androidctl capture=…` 后面加一个短字段，或者并进已有的 `observation {…}` 缺口行，例如 `screen=locked`、`screen=off`、`screen=off,locked`。
- 后台屏会话里分开写：`main=locked`、`main=off` 来自整机字段；`target=off` 来自后台屏的 `displayState`。
- 判定：
  - 锁屏：`keyguardShowing` 或 `deviceLocked` 为真；
  - 熄屏：`interactive` 为假，或者目标屏的 `displayState` 不是 2；
  - 读不到（值为空或带 `…Error`）：写 `unknown`；
  - `userId != 0`：写 `user=N`，因为输入同样会被拒绝；
  - 目标屏不存在（`displayError`）：写 `display=unavailable`。
- `keyguardSecure`（是否设了密码）不打印。

### 最容易出错的地方

- **按上一轮的"与默认值不同就打印"恢复**：设了锁屏密码的真机上每次观察都会多一行，这正是本轮删掉它的理由之一。
- **后台屏会话里**：后台屏的屏幕状态与主屏不同，不能用 `displayState` 判断"主屏熄屏"。
- **`displayState` 的其它取值**（3 和 4 是低功耗显示，6 是挂起的亮屏）一律算熄屏，还是只算 1？建议只要不是 2 就标出来，标的是具体数值。

### 连带要改的

- `ui_view.py:903-935`。
- 主机测试：`test_ui_view.py` 或 `test_observation_files.py` 加锁屏、熄屏、后台屏三种头部。`ui_test_support.py:37-59` 的 `body_metadata` 解析器要认新字段。
- 指引：`observations.md:80-82`（`A short observation {JSON} line reports sampling gaps.`）补一句。
- 内部 docs：`reference/ui.md` 讲默认头部的地方，以及报告下-20 指出的那处"有意删除"的说明。
- 契约样本：不涉及。

### 需设备

- 主屏锁屏时跑一次 `snapshot`；熄屏时跑一次；后台屏会话里主屏熄屏时跑一次，看标记是否正确。
- 有一点要在设备上确认：熄屏后在后台屏上读界面树时，`interactive` 是否如实变成假。

### 读法（待确认）

- 读法 19-1：标记只在锁屏、熄屏、非用户 0、目标屏不可用时出现；`keyguardSecure` 不打印。
- 读法 19-2：后台屏会话里分别标主屏（整机字段）和目标屏（`displayState`）。

---

## 七、第 3 项：对照样本、两项重构和删减目标

### 结论

- 本轮 20 次真实运行**没有保存原始界面树**。保存的是渲染后的 `.txt` 和标准输出，不能作为重构的输入；只有 3 份 `--raw` 的 JSON。
- 能用的输入有三种：契约样本里的 116 条回执；50 份原始界面树；在现有测试里录下来的首行。
- 首行函数降到 120 行是可能的，前提是"字段表"和格式化助手不算在这 120 行里。docs 降到 150 KB 要删约一半，其中有几处是旧决定要求保留的。

### 对照样本从哪里取

| 来源 | 覆盖面 | 取法 | 坑 |
| --- | --- | --- | --- |
| 契约样本 `contract/wire-contract.json`（116 条请求和回执，分 23 组） | 各命令的成功和失败、退出码非零的组合、借用粘贴、等待、会话 | 对每条回执调用 `result_line`（文本），并调用 `print_delivery(..., as_json=True)` | 命令名有的取自 `result.command`，有的取自请求（`test_wire_contract.py:734-1165` 的写法不一）；部分样例要带观察才走得到正文分支 |
| 原始界面树 50 份 | 正文渲染、`--all`、`find`、`--depth`、按引用读、超限摘要 | `render_observation`，用一组视图组合 | 本轮 3 份（交付检查目录）；第六轮材料包 23 份、第五轮 24 份，都已实测能被 `f401deb` 渲染【实跑，`oldraw.py`】。第四轮那 51 份是多段 JSON，读不了 |
| 现有测试的输出 | 测试已覆盖的所有首行分支 | 在测试会话里临时替换 `Output.echo`，按测试名录下每条输出，存成一个文件 | 不改测试本身；重构后再录一次，逐字节比较 |
| 20 次真实运行 | 只能当"真实首行长什么样"的参照，不能重放 | — | 只有渲染后的文本 |

**共同的坑**

- 保存文件的路径带时间戳和序号（`snapshots/000004-20261125T…Z-…txt`），显示时又转成相对路径（`ui_output.py:323-328`），结果取决于当前目录。比较前要把路径换成占位符。
- `deliveryMs`、`snapshotAgeMs` 这类耗时，以及工作进程编号，每次都不同，要去掉。
- Windows 的文本模式标准输出把 `\n` 写成 `\r\n`，路径分隔符也不同。对照样本要么在一个平台上生成并比较，要么比较前统一换行和分隔符。建议 Windows 和 Linux 各生成一份。
- 对照样本是一次性的工具。建议不提交进产品仓库，放在实施方的工作目录，两项重构完成后删掉；要提交的话，会成为又一处要维护的东西（读法 3-1）。

### `result_line` 现在的结构

`ui_output.py:37-316`，280 行【源码】。按段落粗分：

| 行 | 内容 | 约行数 |
| --- | --- | ---: |
| 44-55 | `fill` 的分步、开头 | 12 |
| 56-75 | 原因和错误字段（四层条件） | 20 |
| 76-85 | 名字变化那一行（第 11 项要删） | 10 |
| 86-122 | 变化摘要、`cached`/`fresh`、派发、匹配、提示、加载 | 37 |
| 123-153 | 值、输入、剪贴板、借用粘贴、`select` | 31 |
| 154-185 | `open` 的 14 个字段 | 32 |
| 186-204 | `via`、触摸、分步、`match=` | 19 |
| 205-219 | `scroll-until` | 15 |
| 220-236 | 缺口标记、"Observation: unavailable" | 17 |
| 237-259 | 路径和图片 | 23 |
| 260-283 | 错误字段 | 24 |
| 284-296 | 结果未知的说明 | 13 |
| 297-316 | 通知、输出 | 20 |

此外，这个文件里还有 `change_summary`（`:331-405`，75 行）等 8 个辅助函数，全文件 598 行。

**改成按命令声明字段的表时最容易走样的地方**

- **同一字段在不同命令里条件不同**：
  - `taskMovement` 在 `open` 成功时也打印，其余 `open` 字段只在失败时打印（`:178-181`）；
  - `reason` 对 `clipboard.get` 从不打印，对 `fill` 只在与各步原因不同时打印（`:56-69`）；
  - `snapshotError` 与 `snapshotProblem.message` 相同时不重复打印（`:269-274`）；
  - 通知和剪贴板命令用 `structured_json`，其余用 `compact`（`:276-282`）。
- **字段顺序**：现在的顺序是代码顺序，表格化之后最容易悄悄换位。对照样本是逐字节比较的，能发现这一点。
- **由多个字段合成的那几段**：`change_summary`、`cached`/`fresh`、`dispatchStarted`（只在为假或未完成时打印）、结果未知的说明（按原因码和命令分三种），不适合放进表，宜保留为几个小函数。
- **120 行现实吗**：通用循环加上 `fill`、`clipboard.get`、结果未知说明这几个特例，函数本体降到 100 到 120 行是做得到的（工作记录 99 §2.1 也是这个估计）。但字段表本身会有约 60 到 100 行。"首行生成函数不超过 120 行"宜读作函数本体，不含表和辅助函数，见读法 3-2。

### 测试里解析输出的代码现在散在哪里【源码】

- **正文已经有一个集中的解析器**：`ui_test_support.py:37-59` 的 `body_metadata`（解析头部）、`:62-126` 的 `body_lines`、`:129-138` 的 `one_line`。有 6 个测试文件在用。
- **首行没有解析器**，写法分散：

| 写法 | 例子 |
| --- | --- |
| 整行相等 | `test_result_output.py:143, 227, 264, 754` |
| 前缀 | `test_result_output.py:124` |
| 子串 | `test_display.py:255-257`、`test_cli_arguments.py` 多处 |
| 按 `"; "` 切开 | `test_ui_execution.py:770` |
| 契约测试对各样例逐条断言首行 | `test_wire_contract.py:734-1165`，约 18 处 `result_line` 调用 |

- 按"对输出文字做断言"的行数粗数：`test_result_output.py` 64 处、`test_wire_contract.py` 60 处、`test_observation_files.py` 32 处、`test_ui_receipts.py` 14 处、`test_image_delivery.py` 12 处、`test_ui_reading_delivery.py` 11 处。
- 建议的集中方式：写一个 `parse_result_line(text)`，返回"结果词、原因、键值字段、裸标记"的结构。按 `"; "` 切，`key=JSON` 解析，其余当裸标记。测试改成断言结构，而不是断言整行。
- 注意：`"; "` 也可能出现在原因文字里（`reason` 原样打印）。所以解析器要先认出结果词和原因，再切其余部分；或者测试只断言字段，不断言原因的整句。

### docs 仓库：现在 301,632 字节，目标不超过 150 KB

| 文件 | 字节 | 建议 |
| --- | ---: | --- |
| `reference/ui.md` | 100,509 | 删到只留设计理由，约 30 到 35 KB（报告附录一"最长的一篇删到只留设计理由"） |
| `testing/cases.md` | 68,809 | 留"发布前真机检查"（第六轮第 4 条要求放内部 docs）和 22 个真实案例的清单（第 4 项的新任务要从这里挑）；每个案例压成目标和验收各几行，约 30 到 35 KB |
| `reference/connection.md` | 37,758 | 约 20 KB |
| `architecture.md` | 31,607 | 约 25 KB |
| `reference/jobs-files.md` | 25,024 | 约 12 KB |
| `reference/README.md` | 14,308 | 约 6 KB |
| `development.md` | 11,854 | 保留；其中"本批交付与版本时点"和"接口迁移清单"（`:66-91`）是第六轮第 1 条答复要求只放内部 docs 的 |
| `testing/prepare_assets.py` | 8,912 | 是否计入 150 KB，见读法 3-3 |
| `README.md` | 2,851 | 保留 |

按上表约 155 到 165 KB，到 150 KB 还要再压 `architecture.md` 或 `connection.md`。

**不能删的**（前几轮决定要求放在内部 docs）：

- 发布前真机清单（第六轮第 4 条）；
- "中间提交不可构建"的说明和破坏性变更清单（第六轮第 1 条答复，现在在 `development.md:73, 77-91`）；
- 第四轮第 23 项"只写当前的设计；测试用例那一份改成清单"（这本身就支持删减）。

### 工作记录 99 第 5 节的删减清单与报告附录一的对应

| 记录 99 §5 新清单 | 报告附录一"合入后删减的排序" | 第七轮的去向 |
| --- | --- | --- |
| 1 测试里的输出解析收拢到一个辅助函数 | 第 1 项 | 第 3 项，带数字目标（集中到一个辅助函数） |
| 2 首行改成字段表 | 第 2 项 | 第 3 项，目标不超过 120 行 |
| 3 内部 docs：`ui.md` 删到设计理由，`cases.md` 裁成发布清单 | 第 3 项 | 代定：不超过 150 KB。`cases.md` 不能只剩发布清单，因为新任务要从 22 个案例里挑 |
| 4 两套"加载中"判定合并，删路线题专用字符串 | 第 4 项 | 与第 12 项重合，并入"产品里不留只对某道任务有效的文字" |
| 5 设置退出码的 18 处走同一个助手 | 第 5 项 | 见下面的判断 |
| 6 原因码登记测试简化 | 未列 | 报告第 9 题 A 写了"补约 5 行"；第 20 项只修下-18 的四处，简化不在清单里 |
| 7 契约导出改成直接写回加 `git diff` 审查 | 未列 | 不在清单里 |
| 8 文字归一化加共享样本 | 未列 | 不在清单里 |
| 9 墓碑测试、`"65536"` 改引用常量、滚动的三行多余输出 | 未列 | `"65536"` 并入第 14 项；其余属于小项清单 |
| 明确不值得做：契约样本裁剪、继续收拢参数校验、回退重连等 | 同 | 不做 |

**"设置退出码的 18 处改成走同一个函数"算不算动了第 21 项那块能力**

- 记录 99 §2.2 的原方案有两部分：
  - 一是 18 处 `put("exit", N)` 改走一个助手。这是纯重构，主结果、退出码、输出都不变，不加规则。
  - 二是"强制同时写一个诊断字段""首行统一打 `exit=N`"。这会新增字段、改首行，等于给"完成但退出码不是 0"这一块加了一条规则。
- 我的判断：第一部分不违反第 21 项"只删不补"，可以做；第二部分违反，不做。
- 但第 21 项写的是"不再加规则，出了缺陷优先删或降级"，纯重构是不是"动这块"，用户没有说。所以列为读法 3-4。

### 读法（待确认）

- 读法 3-1：对照样本是一次性的，不提交进产品仓库，两项重构完成后删除。
- 读法 3-2："首行生成函数不超过 120 行"指 `result_line` 函数本体，不含字段表和辅助函数。
- 读法 3-3："docs 仓库不超过 150 KB"按仓库里全部受版本控制的文件计，含 `testing/prepare_assets.py`；`cases.md` 保留 22 个案例的清单。
- 读法 3-4：退出码统一助手只做不改行为的那一半，不加新诊断字段、不改首行。
- 读法 3-5："测试里解析输出的代码集中到一个辅助函数"指首行；正文已经有 `body_lines` 和 `body_metadata`，两者可以合在同一个支持模块里。

---

## 八、第 7、9 项：Linux 测试的修法和 WSL

### 第 9 项：挡-1 的测试现在就修

**现状**【源码，实跑】

- `androidctl/tests/test_ui_delivery.py:797-863`：正文由 `"large body " * 5000` 构成，约 55 KB。读端读到正文头行 `androidctl ` 之后关闭（`:832-840`）。写端如果早已把 55 KB 全部写进管道缓冲区，就不会出错，退出码是 0，断言 `exit_code == 1` 失败（`:859`）。
- 我在本机 Linux（Python 3.12，`S/r7/venv7`）上复现：`assert 0 == 1`。

**三种改法，都已在本机 Linux 上实测**（测试文件复制到 `S/agents12/111/tests_copy/`，不动仓库；连跑 3 遍，每遍三种改法都通过、原测试都失败）：

| 改法 | 做法 | 优点 | 缺点 |
| --- | --- | --- | --- |
| A，临时调高上限 | `monkeypatch.setattr(ui_client, "DEFAULT_BODY_BYTES", 1 << 20)`，正文恢复为约 220 KB（`* 20000`） | 两行；回到上一轮在 Windows 上一直通过的形态；220 KB 大于三个平台默认的管道缓冲区（Linux 64 KiB，macOS 最多 64 KiB，Windows 更小），写端必然被卡住 | 依赖常量名；这个测试不再走"真实阈值"，但它测的本来就不是阈值 |
| B，缩小管道缓冲区 | `fcntl.fcntl(writer, F_SETPIPE_SZ, 4096)` | 一行，不改正文 | 只有 Linux 有 `F_SETPIPE_SZ`；macOS 和 Windows 上要跳过或另写，不满足"各平台都必然触发" |
| C，写端等读端先关 | 包一层输出流：写出正文头行之后，等读端关闭的信号，再写余下部分 | 与缓冲区大小无关，最确定 | 多约 15 行测试代码，依赖输出是按什么块写出的（例如一次写入里同时含头行和正文） |

- 建议用 A：最简单，三个平台的行为都能从缓冲区大小直接推出。C 更彻底，但测试代码更复杂。
- macOS 没有环境验证。A 和 C 在原理上都与平台无关，B 不行。

### 第 7 项：完整检查覆盖 Linux，用本机 WSL 跑主机端全部测试

**仓库现在的结构**【源码】

- `Taskfile.yml`：
  - `quality` 等于 `android:quality` 加 `python:quality`；
  - `ci` 等于 `android:quality` 加 `python:ci`。
- `androidctl/Taskfile.yml`：
  - `python:quality` 是 ruff 检查、ruff 格式检查、mypy，再加 `python -m pytest`；
  - `python:ci` 在此之上加 `mypy --platform linux` 和 `--platform darwin`，**只做类型检查，不在 Linux 上跑测试**。
- `pyproject.toml:63-70`：pytest 的测试目录是 `androidctl/tests` 和 `tools/release/tests`。
- `.github/workflows/quality.yml` 的 `python-posix` 任务：Ubuntu 24.04 上 Python 3.12 和 3.14，macOS 上 3.12；只装 `-e . pytest`，跑 `python -m pytest`。

**Linux 上跑主机端全部测试的最小命令**（与仓库自动检查的 `python-posix` 相同）

```sh
# 在 WSL 自己的文件系统里，不在 /mnt/c 下
git clone /mnt/<盘符>/<仓库路径> ~/androidctl-wsl && cd ~/androidctl-wsl
git checkout <要测的提交>
python3.12 -m venv .venv-linux
.venv-linux/bin/python -m pip install -c constraints-ci.txt -e . pytest
.venv-linux/bin/python -m pytest
```

- 想同时在 Linux 上跑 ruff 和 mypy，就装 `-e ".[dev]"`，再跑 `python -m ruff check …`、`python -m mypy`。不必在 WSL 里装 Task。
- 参照：本轮评审在 Linux 上跑全部主机测试，1,958 个通过、21 个跳过（只适用于 Windows 的）、1 个失败（挡-1）。装法不同时总数不同（报告 §3.1 是 2,009 个通过）。

**在 WSL 里跑的坑**

- **仓库放在 Windows 盘上、经 `/mnt/c` 访问**：
  - 跨文件系统的读写慢很多（WSL2 的 Windows 盘经网络文件协议访问），全部测试会慢数倍；
  - 默认挂载不支持 Linux 文件权限，`chmod` 不生效，所有文件显示为 777。测试里有检查工作区目录权限、凭据目录严格权限的（第四轮第 14 项、第五轮第 6 题第三处），在 `/mnt/c` 上可能失败或走错分支；
  - 在 Windows 盘上创建 Unix 域套接字，部分 WSL 版本不支持。
  - 结论：**在 WSL 自己的 ext4 里克隆一份**，用 `git clone` 或 `git fetch` 从 Windows 那份取同一个提交。
- **换行符**：Windows 那份如果开了 `core.autocrlf`，工作区里是 CRLF。在 WSL 里直接用它，有的测试（脚本文件要求 LF、逐字节比较）会失败。在 WSL 里重新克隆可以避开。
- **`.conda` 环境不能共用**：那是 Windows 的二进制。WSL 里要单独建 Linux 环境。Ubuntu 24.04 自带 Python 3.12，与仓库自动检查的主版本一致；缺 venv 模块时装 `python3.12-venv`。
- **两项要读 Kotlin 或 C 源码的测试**（`test_wire_limits_agree_across_boundaries`、`test_reason_codes`）需要完整的仓库，不能只导出 `androidctl/` 目录（工作记录 99 §3.2）。
- **内存和虚拟化**：
  - WSL2 是一台常驻内存的轻量虚拟机，默认可以占用主机内存的一半，空闲后才慢慢释放。路线题的模拟器要 6 GiB，上一轮用户为了给模拟器腾内存关过一台 Hyper-V 虚拟机。建议写进流程：**每次测完执行 `wsl --shutdown`；不在设备实测和性能测量的时段跑 WSL**。也可以在 `%UserProfile%\.wslconfig` 里设 `memory=4GB` 封顶。
  - 如果这台机器原来没有启用 WSL2：启用它要打开"虚拟机平台"，也就是系统的 Hyper-V 虚拟机监控程序。这时模拟器只能走 Windows 自带的虚拟化加速（WHPX），不能用 Android 模拟器自己的加速驱动。用户以前开过 Hyper-V 虚拟机，说明这一层大概率已经开着。启用后先用 `emulator -accel-check` 确认一次【需确认】。

### 第 7 项其余

- "每次评审前跑全部测试"：建议读作主机端全部测试在 Windows 和 WSL 上各跑一遍；改了 Android 代码时再跑 Android 端的 JVM 测试和静态检查。日志交给评审（读法 7-1）。
- macOS 记为已知未覆盖（用户原话）。挡-1 的改法选 A 或 C，在原理上与平台无关。

### 读法（待确认）

- 读法 7-1：见上。
- 读法 7-2：WSL 里的仓库是 ext4 里的单独克隆，与 Windows 那份按提交号同步，不经 `/mnt/` 直接测。

---

## 九、几处与报告写法不一致、以用户消息为准的地方

| 位置 | 报告写的 | 用户消息 | 按哪个 |
| --- | --- | --- | --- |
| 第 14 项 | 第 7 题 A："`--all` 只做第一件事" | "`--all` 不再打印无名动作编号" | 按消息（读法 14-1）；更宽的做法列为待定 1 |
| 第 15 项 | 第 8 题 B 未提首页行数 | 代定：约 130 行，删 `find` 重新采集的规则 | 按消息；预算见第二节 |
| 第 16 项 | 下-12 写"建议只记录" | 做第 16 项；`find` 时限不改 | 按消息 |
| 第 17 项 | 下-15 只列现象，没有出方向 | 60 秒规则 | 按消息 |
| 第 7 项 | 第 10 题 A：取消三份累计评审 | 全部保留，加两条 | 按消息 |
| 第 3 项 | 第 3 题 A：删减限两周、到期就停 | 全做，带数字目标，达不到写明原因 | 按消息 |
| 报告第 2 题附带事 | 推到临时分支跑自动检查 | 不做；Linux 用 WSL 覆盖 | 按消息 |

---

## 十、旧决定的核对

### 10.1 用户第四部分列的 8 处

| 用户所列 | 旧条目原文（摘） | 改了哪一句 | 没改的部分 |
| --- | --- | --- | --- |
| 第 11 项改第四轮第 11 项 | 第四轮第 11 项第 1 点："被拒绝的动作不再重印整页"；第六轮第 10 条已开过一个例外："附一行，不附整页" | 对"引用已过期"这一类拒绝，附上已读好的新界面树（受 64 KiB 上限约束）；第六轮第 10 条整条被取代（删掉名字变化那一行和它的判定） | 其它拒绝仍不重印整页；`wait` 成功只印命中节点和祖先；加载指示写进回执；首页写清导航后怎么确认 |
| 第 17 项给第六轮第 19 条加例外 | 第六轮第 19 条答复："每次意外退出都重绑一次，但只在上一个进程存活超过一个最短时长时才这样做" | 加例外：自动重连来的进程 60 秒内又退出，不再重连 | 5 秒最短存活；应用自己停止不重绑；旧状态作废；重绑后记一条附在"需要会话"的报错上（这一条另被下-15 改为成功的会话命令也会清掉） |
| 第 16 项扩大第六轮第 17 条的"截断" | 第六轮第 17 条："'等待消失'维持严格"，`idle` 同样处理；它所依据的第四轮第 16 项写的是"截断是节点上限或字节上限造成的时候" | `gone`、`idle` 的提前失败把"按采集时限截断"也算作截断 | "等待出现"类不提前失败；连续两次；多条件规则；按选择器的动作和第五轮第 17 题的选择规则只认节点和字节上限 |
| 第 14 项把第六轮第 3 条 A1 扩到 `--all` | 第六轮第 3 条 A1 答复："没有名字的平台标准动作一律省略；`copy`、`paste`、`cut` 保留" | 这条省略对 `--all` 也生效 | 默认模式的规则；`copy`、`paste`、`cut` 保留；`fill`、`set-progress` 照打；第五轮第 13 题的标签写法 |
| 第 13 项改掉本轮 `open` 的第二次确认 | 不出自任何决定文件：是实施方落实第六轮第 15 条"`open` 落地之后带上界面树"时选的顺序（先确认到前台、读界面树、再确认一次） | 删掉主屏上的第二次确认；没有应用窗口时在时限内继续等；建议里去掉系统窗口 | 第四轮第 13 项"动作后的观察结束时没有看到目标，就在剩余时间里继续查任务列表"；第六轮第 15 条"启动细节只在失败时打印、落地后带界面树" |
| 第 21 项推翻第六轮第 7 条 | 第六轮第 7 条："不设'冻结'，逐块决定"，六块各有裁决 | 四块"只删不补"：借用粘贴的恢复规则、后台屏上的任务保护、"完成但退出码不是 0"的组合、原因码 | 另两块（按件保存规则、截断时的唯一性例外）仍逐块决定；第 16 项正是给后者加规则，不冲突；四块里已列出的缺陷和提示文字照修（下-16、下-17、下-18、下-15 的第二处） |
| 第 5 项结束第六轮第 5 条的基线例外 | 第六轮第 5 条："这一次网页题和路线题的基线也沿用，是用户明确批准的例外，不改变这两条规则对以后各轮的要求" | 例外结束：网页题、路线题基线同日交替重跑 | 长列表题沿用、比值超过 1.25 先重跑；两倍底线；5 次里成功 3 次；环境失败只重试一次 |
| 第 1、6 项把第六轮第 1 条的合入顺序顺延到第八轮之后 | 第六轮第 1 条："全部改动做完；第七轮完整评审通过；重新整理成 6 到 10 个提交……；用户最终批准；快进合入" | "第七轮评审通过"改为第八轮及之后；本轮继续追加提交，整理推迟到合入前（第 9 项） | 6 到 10 个提交、核对树一致、整理后跑完整检查、快进、版本号提交最后追加、中间提交可以不可构建 |

### 10.2 第六轮决定：用户没有列、但被改动或了结的条目

| 第六轮条目 | 现在的状态 | 依据 |
| --- | --- | --- |
| 第 2 条 版本号的时点 | 内容不变；0.4.0 一直保持到第八轮之后的最终批准 | 第 9 项 |
| 第 3 条 A2 `busy` 加主体，"接管无效，等对方退出" | 回执、码表、指引各加一句"对方不退出时请用户停掉它"；不新增码 | 补充第 1 点（下-15） |
| 第 6 条 过程规格与第七轮评审 | "第七轮评审对象……挡合入的条目改完后由评审方针对性复核，复核通过即评审通过"了结：不合入，挡-1 在本批修，由第八轮完整评审核对；过程规格保留并加两条（评审前跑全部测试；用 WSL 覆盖 Linux） | 第 1、7、8、9 项 |
| 第 8 条 封闭清单 | 这一批的清单是：23 条、约 40 条小项、删减、新任务。清单之外新发现的缺陷改按"直接依赖里的具体缺陷可自主修，有能力损失或不兼容的取舍带选项回来问"，不再一律等下一轮报告 | 第 1、2、3 项；代定第 3 点 |
| 第 10 条 名字变化那一行 | 整条被取代：删掉这个功能，旧引用被拒时附整份新界面树 | 第 11 项 |
| 第 11 条落实要点"首页和 `find --help` 里要补上这条行为" | 首页那一半删去（或压成一句）；`find --help` 和首行的 `fresh` 标记保留 | 代定第 5 点 |
| 第 12 条 后台屏到期 | 行为不变；"到期直接销毁"进首页要点 | 第 15 项 |
| 第 14 条落实要点"摘要里……一行怎样缩小范围的可照抄写法" | 阈值不变；那行写法改为"先 `find`、再按引用读"，并按命令区分 | 第 14 项 |
| 第 15 条 默认文本再瘦一轮 | 规则不变；本轮删掉的锁屏和熄屏状态不在它允许删除的范围内，恢复成短标记（属于落实上的纠正）；"`open` 落地后带界面树"的做法改为在时限内等第一个应用窗口 | 第 13、19 项 |
| 第 17 条 截断规则 | 见 10.1 | 第 16 项 |
| 第 19 条答复"重绑之后记一条，在下一条报'需要会话'的命令上附上" | 加：第一次成功的会话命令也清掉这条 | 补充第 1 点（下-15） |
| 第 22 条 借用粘贴 | 规则、主结果、退出码不变；回执加一句固定说明，指引加一句 | 第 18 项 |
| 第 24 条 原因码再合并一轮；"21 个混合码留到第七轮进下一轮" | 不再做合并式整理；混合码只改下-18 列的四处（其中 `tlsIdentity` 是一处），其余一直不动；"新增码必须对应不同下一步"之上再加"只删不补" | 第 20、21 项 |
| 第三节乙 B2 跳转后的标注 | 改为补全规则（窗口和标题没变、变化只在被操作节点自身时标"可能还在变化"）；删掉 Google Maps 专用文字；不做"继续等到标题变化" | 第 12 项 |
| 第三节"不在这一批里的"第 1 项 合入后减法清单 | 进这一批，带数字目标；内容按记录 99 的新清单，不含"参数校验收拢"和"契约样本裁剪" | 第 3 项，代定第 2 点 |
| 第三节"不在这一批里的"第 3 项 从 22 个案例挑题复测 | 进这一批：10 道新任务，从 22 个案例按形态挑，再补两三种 | 第 4 项，代定第 1 点 |
| 第三节"不在这一批里的"第 5 项 21 个混合码 | 了结：只改下-18 的四处 | 第 20 项 |
| 第三节"不在这一批里的"第 2、4、6 项（核心冒烟、"网页占位误判"的说法、原因码候选第 7 到 12 组） | 仍不在这一批 | — |
| 第七节"首页约 130 行" | 维持；并入后台屏要点的同时删 `find` 那几行 | 代定第 5 点 |

### 10.3 第五轮决定

| 第五轮条目 | 现在的状态 | 依据 |
| --- | --- | --- |
| 第 6 题第一处 `find` 重新采集 | 行为不变；首页上的说明删去或压成一句 | 代定第 5 点 |
| 第 9 题 成本底线 | 底线不变；成本统计另外与上一轮候选比较 | 第 5 项 |
| 第 11 题 固定任务集 | 规则恢复："依赖实时网络的任务在每次全量实测时与候选交替重跑"，长列表题比值超过 1.25 才重跑；另加 10 道只跑候选的新任务（不进固定任务集） | 第 4、5 项 |
| 第 12 题 公开指引（首页约 130 行，多数任务不必再打开专题；报告读指引比例和再读专题的次数） | 维持；显示专题改为"出问题时再读"，后台屏要点进首页；那两个数用来验证第 15 项 | 第 15 项 |
| 第 13 题 默认输出 | 不变；无名平台动作的省略扩到 `--all` | 第 14 项 |
| 第 16 题 过程规格"维持现状，不另加要求" | 加两条：评审前跑全部测试；用 WSL 覆盖 Linux | 第 7 项 |
| 第 17 题 截断时按名字操作 | 不变（只认节点和字节上限）；`wait` 的 `gone`、`idle` 另按第 16 项 | 第 16 项 |
| 第 1 题 搬屏检测和四分类 | 行为不变；属于"只删不补"的四块之一；要点进首页 | 第 15、21 项 |

### 10.4 第四轮决定

| 第四轮条目 | 现在的状态 | 依据 |
| --- | --- | --- |
| 第 8 项 默认输出（"完整事实仍可用 `--raw` 取回"） | 不变；如果第 14 项选宽读法，`--all` 的全部属性只能从 `--raw` 取，与这一项一致 | 第 14 项 |
| 第 10 项 首页（"后台屏三五行"，已被第五轮第 12 题放宽到约 130 行） | 后台屏一段从 3 行扩到约 8 行 | 第 15 项 |
| 第 11 项 动作后的观察 | 见 10.1 | 第 11 项 |
| 第 13 项 启动确认 | 不变；第 13 项的"在时限内继续等"与它同向 | 第 13 项 |
| 第 16 项 界面树被截断（"截断是节点上限或字节上限造成的时候"） | 对 `wait` 的 `gone`、`idle` 加上采集时限 | 第 16 项 |
| 第 19 项 实测（全量实测在首轮、每三轮、合入之前各做一次） | 这一批再做一次四道题各 5 次；另加 10 道新任务，只跑候选 | 第 4 项 |
| 第 21 项 提交历史（"如果之后又打磨了多轮，合入之前再整理一次"） | 正是这种情形：整理推迟到合入前 | 第 9 项 |
| 第 22 项 实施方的评审与取证 | 保留；加两条 | 第 7 项 |
| 第 24 项 评审方的角色（每轮完整评审；讨论结束打包） | 第八轮仍做完整评审，第一页两栏；打包等用户说了再打 | 第 8 项 |
| 第四节"回归环境：每一轮在 Android Studio 模拟器上跑后台屏固定任务" | 不变；另定设备验证以 Android Studio 模拟器为准，MuMu 只作主屏补充 | 第三部分补充 |

---

## 十一、全部读法（待确认）

| 编号 | 项 | 读法 |
| --- | --- | --- |
| 14-1 | 第 14 项 | `--all` 只省"平台标准动作里没有名字的"，其余照全打 |
| 14-2 | 第 14 项 | 改默认输出，排在对照样本和首行重构之后 |
| 15-1 | 第 15 项 | "约 130 行"读作 130 到 135 行，到不了写明实际行数 |
| 15-2 | 第 15 项 | `find` 那几行压成一句保留，不全删 |
| 16-1 | 第 16 项 | 只认截在每次采样 3 秒上限上的截断，不含截在请求截止上的 |
| 16-2 | 第 16 项 | "连续两次"照用，"同样方式"按截断原因的集合比较 |
| 16-3 | 第 16 项 | 复用 `collectionLimit`，码表补一句，类别不改 |
| 17-1 | 第 17 项 | 60 秒按每个自动重连来的进程各自判断，活过 60 秒的照旧重连 |
| 17-2 | 第 17 项 | 用户刷新、Shizuku 重新就绪触发的绑定都算手动，恢复资格 |
| 17-3 | 第 17 项 | 自动重连来的进程活不到 5 秒就退出，用"60 秒内又退出"那句原因 |
| 17-4 | 第 17 项 | "第一次成功的会话命令"指完成的 `session open` 或 `session takeover` |
| 18-1 | 第 18 项 | 固定说明放主机端，不重导契约样本（实施方可改选设备端并写明） |
| 18-2 | 第 18 项 | 说明只加在"粘贴已核对一致、恢复失败或未知"两种情形 |
| 19-1 | 第 19 项 | 只在锁屏、熄屏、非用户 0、目标屏不可用时标记；不打印 `keyguardSecure` |
| 19-2 | 第 19 项 | 后台屏会话里分别标主屏和目标屏 |
| 3-1 | 第 3 项 | 对照样本一次性使用，不提交，重构后删除 |
| 3-2 | 第 3 项 | 120 行指 `result_line` 函数本体，不含字段表和辅助函数 |
| 3-3 | 第 3 项 | 150 KB 按 docs 仓库全部受版本控制的文件计；`cases.md` 保留 22 个案例 |
| 3-4 | 第 3 项 | 退出码统一助手只做不改行为的那一半 |
| 3-5 | 第 3 项 | "解析输出的辅助函数"指首行解析，可与正文解析放在同一模块 |
| 7-1 | 第 7 项 | "每次评审前跑全部测试"：主机端在 Windows 和 WSL 各跑一遍，改了 Android 代码再跑 Android 端 |
| 7-2 | 第 7 项 | WSL 里用 ext4 中的单独克隆，按提交号同步 |

## 十二、待用户定

| 编号 | 问题 | 选项 | 推荐 |
| --- | --- | --- | --- |
| 待定 1 | `--all` 少打到什么程度（第 14 项） | A 只省无名平台动作（−32% 到 −39%，仍超限）；B 报告的宽读法：只多显示节点，属性按默认（约 −58% 到 −73%，要拆参数，全部属性只能从 `--raw` 取）；C 动作套默认省略、其余属性全打（约 −48% 到 −58%，仍超限） | 按原话是 A；如果用户的本意是让 `--all` 不再触发上限，只有 B 部分做得到 |
| 待定 2 | 默认 5 秒的 `wait --gone`/`--idle` 在每次截在 3 秒的页面上怎么办（第 16 项） | A 照用"连续两次"，超时回执写明截断原因（默认用法下不会提前）；B 3 秒上限截断一次就提前失败（快，但偶尔慢一次也会误报）；C 把截止后完成的采样也算第二次（要改"截止后不评估"，不推荐） | A |

附：本节和第十一节里，影响调用者可见行为的只有待定 1、待定 2 以及读法 15-2、16-3、17-1、18-1、19-1。其余是实施细节或过程口径。
