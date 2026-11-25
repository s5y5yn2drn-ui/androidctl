# 第六轮决定文件终检（源码一侧）

核对对象：`/home/user/androidctl/review/androidctl-r6-decisions.md`（提交 `e445e0d`，下称"决定文件"）。源码一律指 `fb75afc`（`S/r6/prod`，只读）。Kotlin 文件省略前缀 `android/app/src/main/java/com/rainng/androidctl/`，主机文件省略前缀 `androidctl/src/androidctl/`，指引文件省略前缀 `skills/androidctl/`。内部文档指 docs 仓库 `66bb380a` 的导出。

证据类别：【源码】读 `fb75afc`；【实跑】在 `S/r6/venv6`（可编辑安装的 `fb75afc` 主机端）里调用产品自己的函数或命令，脚本在 `S/agents10/87/render.py`，用 `-I` 运行；【AOSP】读已下载的 AOSP 14/15/16 文件（`S/agents6/29/aosp/`、`S/agents9/71/aosp/`）；【推断】。没有设备，没有跑 Gradle，Android 一侧结论全部来自读源码。

---

## 1. 总评

- 与源码不符、照字面做会出错的有 6 处（第 2 节），需要收紧或补充的有 11 处（第 3 节）。上一遍 85 号记录的 28 条里，26 条已经改对或因用户答复而不再适用，2 条改了但仍不准（第 5 节）。
- 最严重的是第 3 条 A1：决定文件说"只要把默认省略的判断挪到前面，`action-1020037`、`action-1020044` 就不再打印"，这不成立。这两个编号不在默认省略集合里，挪位置之后照样打印，必须另加"平台标准动作、没有名字"这一判断。
- 第四节第 10 处（`session open` 默认主屏）的大方向可靠："跟随现有显示"放在设备端、在会话锁里判断，这个建议成立，改法也简单。但有三句不准：回执首行现在在新开、复用、替换三种情形下**都没有**写显示（显示在第二行），不只是复用路径要核；"同名会话属于另一个配对、需要接管"的说法与代码的归属规则不符；到期记录现在只存会话名，不存显示种类。另外漏了契约样本和两端测试的连带改动。
- 有一个需要用户裁决的新问题（第 4 节）：细则 e 的提示只加在"到期报错"里，可调用者到期后如果第一条命令就是不带 `--display` 的 `session open`，会直接在主屏新开并成功，到期记录随之被清掉，调用者永远看不到"要继续用后台屏请写 `--display virtual`"。`session close`、设备上的"释放会话"、常驻进程重启也有同样的路径。
- 替实施方核过：`action REF --action-name copy`（以及 `paste`、`cut`）在 `fb75afc` 上可以调用（第 6 节）。

---

## 2. 与源码不符或做不通（必须改）

### 2.1 第 3 条 A1：只挪判断顺序，`action-1020037`、`action-1020044` 仍会打印

- **决定文件原文**："把'默认省略'的判断放到'自定义动作'和'标签读不到'这两个分支之前。这样编号 64、128、256、512、1024、2048 的六个动作……'滚动到位置'（显示为 `action-1020037`）、'显示提示'（显示为 `action-1020044`……）等没有名字的平台标准动作都不再打印。"
- **源码事实**【源码】：
  - `compact_actions`（`ui_view.py:106-144`）的"默认省略"判断用的是集合 `DEFAULT_OMITTED_ACTIONS`（`ui_view.py:55-69`），内容是 1、2、4、8、64、128、16908342、256、512、1024、2048、131072、16908383。
  - 64 到 2048 这六个在集合里，又不在命名表 `ACTION_NAMES`（`node_facts.py:42-72`）里，所以挪位置之后确实会被省略。
  - 但 16908343（十六进制 0x1020037，滚动到位置）只在 `PARAMETERIZED_ACTIONS`（`ui_view.py:40-53`）里，16908356（0x1020044，显示提示）哪个集合都不在。挪位置之后它们仍然走"不在命名表"那一支，照样打印成 `action-1020037`、`action-1020044`。同样漏掉的还有 16908357（隐藏提示）、16908354（移动窗口）、16908362（按住）、16908373 到 16908376（拖放三个、显示文字建议）、16908382（按方向滚动）。
- **怎样区分"平台标准动作"和"应用自定义动作"**【AOSP】【推断】：
  - AOSP 16 的 `AccessibilityNodeInfo.java` 里，平台标准动作就是用私有构造器登记进 `sStandardActions` 的那些（`AccessibilityNodeInfo.android-16.0.0_r1.java:6122, 6855-6861`）：21 个单比特旧式编号（1 到 `ACTION_SET_TEXT`=2097152，`LAST_LEGACY_STANDARD_ACTION`，第 487 行），以及 23 个 `R.id.accessibilityAction*`（第 6544-6824 行），后者都在框架资源编号段 0x0102xxxx 里。
  - 公开构造器的文档要求应用自定义动作"必须用资源标识"（第 6845-6847 行），也就是应用自己的 0x7fxxxxxx 段。
  - 所以主机端可以这样判：编号是单比特且不大于 2097152，或者落在 0x01020000 到 0x0102FFFF，就算平台标准动作；其余都算自定义动作。用显式表加编号段兜底，未来版本新增的框架动作也能归对。这条判法只能读源码推出，应用硬塞框架编号当自定义动作的情形没有设备验证。
- **建议的改后文字**："默认文本只打印三类动作：有专门命令的（`NAMED_COMMAND_ACTIONS`；滚动方向另在 `scroll` 字段里打印）；平台标准动作里有名字的（`ACTION_NAMES` 里的，例如 `copy`、`paste`、`cut`、`context-click`，默认省略集合里的除外）；平台标准动作之外的应用自定义动作。平台标准动作按编号判断：单比特且不大于 2097152，或落在 0x0102xxxx 段。平台标准动作没有名字的，一律默认省略，不论是否需要参数。这个判断放在'自定义动作'和'标签读不到'两个分支之前。只把 `DEFAULT_OMITTED_ACTIONS` 挪到前面不够，16908343、16908356 等不在这个集合里。"

### 2.2 第四节第 10 处："回执首行'落在哪个显示'现在是否每次都有，要核一遍复用路径"

- **源码事实**【源码】【实跑】：`session open`、`session takeover` 的文本回执由 `render_summary`（`app.py:200-247`）生成。首行是 `命令: 结果; 原因; code=...`，**三种成功情形下首行都不含显示**；显示在第二行 `target={...}`（`emit_fields`，`app.py:212-219`）。用 `S/agents10/87/render.py` 喂设备端的三种回执，实际打印是：
  - 新开主屏：`session open: done; Target established`，第二行 `target={"kind": "main", "displayId": 0}`。
  - 复用后台屏：`session open: done; Target reused; idle configuration unchanged`，第二行 `target={"kind": "virtual", "displayId": 7}`。
  - 替换成后台屏：`session open: done; Target established`，第二行 `target={"kind": "virtual", "displayId": 9}`。
  - 设备端回执文字来自 `TargetManagement.kt:187-199`（`targetReceipt`），`data.target` 在三种情形下都有。
- **影响**：实施方照原文只去"核复用路径"，会以为新开和替换已经满足。
- **建议的改后文字**："现在三种成功情形（新开、复用、替换）的首行都不写显示，显示只在第二行 `target=` 里。要在主机的 `render_summary` 里把 `data.target` 的种类和编号提到首行，并标出是新开还是复用（设备回执的 `reason` 已经区分 `Target established` 和 `Target reused`）。只改主机渲染即可，不动契约。失败且没有目标时（例如 `displayMismatch`）首行写'未建立'或不写，由实施方定。`session takeover` 宜同样处理。"

### 2.3 第四节第 10 处："同名会话属于另一个配对、需要接管时，照现状报"

- **源码事实**【源码】：
  - 会话的身份是"配对加会话名"（`SessionOwner.sessionKey`，`Sessions.kt:5-13`）。另一个配对里的同名会话是**另一个会话**，不是"同名会话"。
  - 设备上同一时刻只有一个当前租约（`Sessions.kt:68`）。`session open` 在会话锁里先让到期的租约失效（`expire()`，`Sessions.kt:147`），然后：当前租约属于别的调用主体（另一个配对，或同一配对下的另一个会话名），一律报 `busy`（`Sessions.kt:151-155`，`busy` 的文字在 `Sessions.kt:385-390`）；属于自己且种类相同就复用（`Sessions.kt:111-116`）；属于自己但种类不同且没带替换就报 `displayMismatch`（`Sessions.kt:157-162`）；没有当前租约就新开。
  - 接管只在同一配对内（`Sessions.kt:153`），跨配对不能接管。
- **建议的改后文字**："'会话已存在'在代码里的含义是：设备上唯一的当前租约属于同一配对、同一会话名，并且没有到期（判断前先执行到期检查）。当前租约属于别的调用主体（另一个配对，或同一配对的另一个会话名）时，照现状报 `busy`，不因为没写 `--display` 而改变；接管只在同一配对内可用。被设备上的'释放会话'释放过的会话没有租约，按新开处理。"

### 2.4 第四节第 10 处与第 25 条细则 e：到期记录不含显示种类，提示有永远看不到的路径

- **决定文件原文**："到期报错里那句提示，只在到期的会话原来在后台屏上时才加；到期记录现在挂在常驻进程内存里，常驻进程重启后会丢。"
- **源码事实**【源码】：
  - 到期记录是 `expired: MutableSet<Pair<String, String>>`（`Sessions.kt:70`），到期时只记会话的身份（`Sessions.kt:321-327`），**不记显示种类**，也不记任务。要加提示，得把它改成"身份到记录"的映射，记录里至少有显示种类；第 12 条要求的"到期时后台屏上的任务"应放进同一份记录（见第 3 节第 4 条）。
  - 到期报错只有一处：`sessionRequired`，文字 `Session expired after its idle timeout; run session open --display main|virtual explicitly`（`Sessions.kt:369-372`）。它出现在所有需要会话的命令上：界面命令（`acquireUi`，`UiBackend.kt:114`）、`shell`（`ShellJobs.kt:37`）、文件传输（`FileJobs.kt:24`）、读交付物（`UiDelivery.kt:389`）。`status` 和 `wake` 不报它。
  - 记录在三处被清掉或看不到：`session open` 新开成功时清掉（`publish`，`Sessions.kt:219`）；`session close` 清掉（`Sessions.kt:180`）；`status` 的 `callerState` 只有 `released`、`active`、`none` 三种，到期后报 `none`，不说到期（`Sessions.kt:357-362`）。
- **由此得出的结论**：调用者到期后如果第一条命令就是 `session open`（不写 `--display`），设备端按细则 a 在主屏新开、成功、清掉记录，**这句提示调用者永远看不到**；它只能从细则 d 的首行"落在主屏"自己发现。先发 `session close` 再开，或者常驻进程重启，结果相同。只有"到期后先发一条界面、shell 或文件命令"时才看得到提示。这一点需要用户知道，见第 4 节。
- **建议的改后文字**（第四节第 10 处）："到期记录现在只存会话身份（`Sessions.kt:70, 324`），要改成带显示种类的记录，并与第 12 条'到期时记下任务'合成一份。提示出现在到期之后第一条需要会话的命令（界面、`shell`、文件）的 `sessionRequired` 报错里；`session open` 新开、`session close` 都会清掉记录，`status` 现在不显示到期。所以到期后直接 `session open` 的调用者看不到这句提示，处理办法见第五节（或用户裁决结果）。"

### 2.5 第 23 条读法："`--raw` 下不起作用的选项是 `--all`、`--depth`、`--max-chars`、`--interactive` 四个"

- **源码事实**【源码】：`--raw` 时正文就是整份观察的 JSON（`ui_client.py:782`），投影结果只在 `not view.raw` 时才替换正文（`ui_client.py:839` 起），查询和引用只用来算首行的命中数（`ui_client.py:803-811`）。逐个看：
  - `snapshot`（`ui_commands.py:507-578`）：`--all`、`--boxes`、`--depth`、`--interactive`、`--max-chars` 都不影响正文。位置参数 `REF` 会把请求变成读缓存并发 `renderRef` 给设备，但正文仍是整份 JSON，不是那棵子树。
  - `find`（`ui_commands.py:690-724`）：`--context`、`--all`、`--boxes`、`--depth`、`--interactive`、`--max-chars` 都不影响正文；查询词只影响首行的 `matches=`。
  - 起作用的：窗口选项（`--package`、`--window`、`--windows`、`--all-windows`，决定设备采哪些窗口）、`--cached`、截图类选项（另存图片）、`--timeout`、`--out`、`--print`/`--no-print`。
- **建议的改后文字**："'等'读作在 `--raw` 下不影响正文的选项：两个命令共有的 `--all`、`--boxes`、`--depth`、`--max-chars`、`--interactive`，`find` 另有 `--context`。`snapshot REF --raw` 的正文也不是那棵子树，宜一并拒绝，或明说它保存整份观察。窗口选项、`--cached`、截图选项、`--out`、`--print`/`--no-print` 照常接受。"这是评审方补的读法，改后仍要让用户确认（第六节末尾已列为待确认）。

### 2.6 第四节第 1 处："显式 `--print` 越过上限时正文已经全文打印，保存失败仍只是警告"

- **源码事实**【源码】：显式 `--print` 时，保存失败是否只是警告取决于 `body_file_required = view.out is not None or view.raw or not print_body`（`ui_client.py:975`）。不带 `--out`、`--raw` 时成立（`printed_body_fallback` 为真，只写 `snapshotWarning`，结果仍为完成）；带 `--out` 或 `--raw` 时不成立：`snapshot`、`find` 报失败，其它命令退出码变 1（`ui_client.py:985-1032`）。
- **影响**：第 23 条落实要点写"`--raw` 的文件……保存失败按失败处理"，而第四节第 1 处这句不加限定，就与它字面冲突（`--raw --print` 时到底是警告还是失败）。
- **建议的改后文字**："显式 `--print`、且不带 `--out`、`--raw` 时，正文已经全文打印，保存失败仍只是警告；带 `--out` 或 `--raw` 时文件是必需的交付物，保存失败照旧按失败。"

---

## 3. 需收紧或应补充（建议改）

1. **第四节第 10 处，判断的位置写错了文件。** 原文说"同种显示就复用、不同种且没带替换就拒绝"的判断在 `TargetManagement.kt:65-96`。那里只解析请求（`DisplayKind.requested`，`DisplayTarget.kt:13-15`，缺省时报 `arguments` "displayKind is required and must be main or virtual"）并调用 `prepareChange`；复用、拒绝、替换、接管的分支都在 `Sessions.kt:101-166`（复用 `111-116`，`displayMismatch` `157-162`，接管的归属 `153`，替换和接管走 `117-133` 新建租约、在锁外跑预检后再作废旧租约）。建议改引用。

2. **第四节第 10 处，设备端要改的地方和连带的测试、契约，没有写。** 建议补一段：
   - 设备端：`DisplayKind.requested` 对 `session.open` 允许缺省（`session.takeover` 和带 `replace` 的仍必填）；`TargetManagement.execute` 现在在进锁之前按种类构造后台屏对象（`TargetManagement.kt:81-82`），缺省时种类要在 `Sessions.checkChange` 的锁里决定：有自己的当前租约就复用它的目标，没有就新开主屏。缺省只可能落到"复用"或"新开主屏"，用不到后台屏对象，改动很小。
   - 契约：`contract/wire-contract.json` 的 `sessionManagementResults`（第 7837 行起）只有三条样本，都带 `displayKind: main`；主机 `test_wire_contract.py:107-110` 断言请求里 `displayKind == "main"`；设备 `DisplayTargetTest.kt:50-57` 断言空参数被拒。要加"缺省、跟随现有"和"缺省、新开主屏"两条样本，按 `AGENTS.md` 的流程导出、核对、替换，再跑两端契约测试。
   - 主机端：`--display` 改成可选，缺省时不发 `displayKind` 这个键（不要发 `null`，`DisplayTargetTest.kt:51` 把 `null` 当错误）；`test_display.py:48-77`、`test_cli_arguments.py:144-157, 611-621` 的期望要改。
   - 设备文字：`Sessions.kt:92, 149, 371` 三处写着 "run session open --display main|virtual explicitly"，`Sessions.kt:160` 的 `displayMismatch` 提示，都要按新规则改写。

3. **第 25 条细则 c 和"`\n\t` 那条报错"的落点。**
   - `\n\t` 的来源【源码】【实跑】：typer 0.27.2 的选项类型在缺参数时生成 `Choose from:\n\tmain,\n\tvirtual`，里面是真的换行和制表符（`venv6/.../typer/_types.py:64-69`）；主机的 `main()` 把它包成 `arguments` 错误（`app.py:999-1011`）；`render_summary` 用 `display_name(..., quoted=False)` 把控制字符转义（`app.py:203`、`output.py:14-19`），于是打出字面的 `\n\t`。改完之后只剩 `session takeover --display` 一个必填的选择型选项，但修法应放在 `main()` 里对框架消息统一压缩空白，免得以后再出现。
   - 细则 c：`--replace` 不带 `--display` 宜在主机参数阶段拒绝（`app.py` 的 `session_open` 里加一句，报 `arguments`、未发出，给可照抄的写法），设备端对"带 `replace`、缺 `displayKind`"也拒绝，作防线。现在 `session open --replace` 不带 `--display` 报的就是那条带 `\n\t` 的缺参数错误【实跑】。
   - 细则 b：`session takeover` 现在已经必须写 `--display`（`app.py:870-875`），不用改行为，只修那条报错的格式。

4. **第 12 条与细则 e 是同一条报错、应是同一份记录。** 两者都附在 `sessionRequired` 的"已到期"报错上（`Sessions.kt:369-372`），都需要扩展 `expired` 这份记录（`Sessions.kt:70`）。显示种类在到期时（锁内，`Sessions.kt:321-327`）就知道；任务列表按第四节第 9 处在异步清理里查。建议在第 12 条或第四节第 9 处写明"两者合用一份到期记录：显示种类在锁内记，任务列表在异步清理里补"。

5. **不带 `--display` 的 `session open` 跟随后台屏会话时，不会触发任何销毁检查。** 【源码】复用路径在 `prepareChange` 里直接返回（`Sessions.kt:111-116`），不调用预检，也就不调用 `releaseDisplayTasks`（`TargetManagement.kt:24-37` 只在预检里被调用）；`changeTarget` 对复用也跳过清理检查和建屏（`TargetManagement.kt:134-138`）。建议在第四节第 10 处写一句，免得实施方担心。顺带两点同样的现状：
   - 复用时 `--idle` 被忽略（回执写 "idle configuration unchanged"，`TargetManagement.kt:196`）；主机总是发 `idleSeconds`（默认 1800，`app.py:818-838`），设备分不清是不是显式写的。跟随现有显示之后，`session open --idle 2h` 不写 `--display` 也会静默保持旧的空闲时长。要改空闲时长只能先关再开。指引 `displays.md:16-17` 已经写了"即使写了不同的 `--idle`"，可沿用。
   - `--return-tasks`、`--discard-tasks` 不带 `--replace` 时被接受、不起作用。是否在参数阶段拒绝由实施方定。

6. **第 25 条细则 f 和"指引要改"的清单不全。** 现在出现 `session open` 或 `--display` 的地方：
   - 首页 `SKILL.md:18`（示例，保留 `--display main`）、`SKILL.md:24-25`（"For a background screen use `session open --display virtual`"，保留）；`README.md:145`（保留）。
   - 显示专题 `displays.md:3, 6-7, 16-21, 61-62`：第 3 行"open its target explicitly"、第 16-21 行的规则段要加"不写时默认主屏、已存在时跟随、`--replace` 必须写 `--display`"。第 6-7、61-62 行的示例本来就写了 `--display`，可保留。
   - 连接专题 `connection.md:92`（"Same-kind open reuses the lease"）要补"不写 `--display` 时跟随"。决定文件说"连接专题要改"，指的应是这一句，连接专题里没有别的 `session open` 写法。
   - `session open --help`（`app.py:798-800` "Open an explicit main or virtual session"，`app.py:809` 选项说明）。
   - 内部文档：`architecture.md:58, 88`；`reference/connection.md:13, 139, 146, 149, 153, 167`；`reference/ui.md:284, 318`。其中 `architecture.md:58` 写着"显式 open/takeover 必填 displayKind"，与新规则直接冲突。

7. **第 19 条答复的落点。**
   - "上一个进程存活超过最短时长"需要的时间戳现在没有【源码】：应用一侧只有绑定开始的时间 `Attempt.started`（`WorkerConnection.kt:59-62`），连上之后的 `Attached` 没有记连上的时刻（`WorkerConnection.kt:305-322`），常驻进程的状态里也没有启动时间（`ManagementState.kt:78-101`）。要在 `onServiceConnected` 接受连接时（`WorkerConnection.kt:89-95`）记一个时刻。
   - "下一条命令报需要会话"实际有两种码：重绑还没完成时，命令在应用一侧就被拒为 `workerUnavailable`（`requireTarget`，`WorkerConnection.kt:291-302`，文字取 `lastFailure`，例如 "Worker died"）；重绑完成后，新进程里没有会话，报 `sessionRequired` "No current session"（`Sessions.kt:369-372`）。记录两种都要附。
   - 记录必须存在应用进程里，新进程的内存是空的。所有转给常驻进程的请求都经过 `NetworkGateway.dispatch` 里的一处 `worker.call`（`NetworkGateway.kt:306-323`），常驻进程的错误在应用一侧以异常形式出现（`decodeWorkerReply`，`WorkerConnection.kt:330-343`），在这里附最省事。
   - 旧会话的主人和显示种类，应用一侧只有最近一次读到的状态（`SessionStatus` 的 `name`、`targetKind`，`ManagementState.kt:103-125`），可能过时。旧会话如果在后台屏上，这条记录也应带上"要继续用后台屏请写 `--display virtual`"，否则重绑之后不写 `--display` 的 `session open` 同样会静默落到主屏（并入第 4 节的问题）。

8. **第 3 条 A2 与第 24 条：`busy` 现在没有任何一处带主体。** 【源码】设备端 `busy` 构造在 `Sessions.kt:214, 282, 309, 385-390`、`JobTable.kt:46`、`FileJob.kt:150`；主机端在 `coordinator.py:281, 411, 620, 655, 696`、`file_jobs.py:588`、`local_server.py:141`；都没有 `subject`。主机和指引都没有按 `busy` 分支，只有 `SKILL.md:23-24` 写"respect `busy` and `connectionBusy`. Takeover needs user instruction"和 `connection.md:93-94`。所以"码表里 `busy` 按主体分两句"实际是"不带主体"一句加"主体为自动化登记"一句；指引里"接管要用户指示"那句要补"主体是自动化登记时接管无效，等对方退出后重试"。顺带一个现状：同一个同步抛错在 `wait` 上报失败（退出码 1），在其它命令上报未发出（退出码 2），因为 `wait` 缺事实时默认失败（`UiRun.kt:125-131`）。这不是新问题，实施方知道即可。

9. **第 3 条 A2 的识别细节**（补进第四节第 9 处）【AOSP】【源码】：
   - 异常类型和消息：Android 14、15、16 的 `UiAutomationManager` 都抛 `IllegalStateException`，消息是 `"UiAutomationService " + 客户端 + "already registered!"`，客户端和 "already" 之间**没有空格**（14：`UiAutomationManager.java:103-105`；15：`109-111`；16：`109-110`）。识别时用"包含 already registered"，不要按整句或按带空格的前缀匹配。
   - 传播路径：系统进程抛出的 `IllegalStateException` 经 Binder 原样带回；`UiAutomationConnection.registerUiTestAutomationServiceLocked` 只捕获 `RemoteException`（16：`UiAutomationConnection.java:688-693`），`UiAutomation.connectWithTimeout` 也只捕获 `RemoteException`（16：`UiAutomation.java:384-396`），所以它在等待之前同步抛出，经反射被包成 `InvocationTargetException`，产品的 `rootCause`（`Wire.kt:73`）能取出原异常。
   - 为什么会锁死：`connectWithTimeout` 在调用前已把状态设为"连接中"（16：`UiAutomation.java:369-376`），抛错后状态停在"连接中"，之后 `disconnect()` 必然抛 "Cannot call disconnect() while connecting"（16：`UiAutomation.java:438-443`），于是产品的 `registrationPending` 永远清不掉（`AutomationCleanup.kt:28-41`）。修法是在 `registerAutomation` 的 `catch`（`UiBackend.kt:535-538`）里，判定为同步抛错时先把"待注销"清掉再走 `discardConnection`。
   - 现在这条回执的码是默认的 `workerFailure`（`Wire.kt:76-82`），结果是未发出，另附 `uiCleanup` 和退出码 3（`AutomationCleanup.kt:59-65`）。

10. **第 22 条：契约样本要重导，现成的退出码通道可以直接用。**
    - 回执有一个 `exit` 字段，设备端可以设，主机 `Result.exit_code` 优先用它（`protocol.py:64-78`）。现成的"完成加退出码 3"例子是 `session close` 清理未确认（`TargetManagement.kt:279-283`）和 `withUiCleanup`（`AutomationCleanup.kt:59-65`）。"完成加退出码 1"在设备端还没有先例，但通道支持任意整数。
    - 恢复状态的取值有 `not required`、`pending`、`skipped`、`unknown`、`failed`、`cleared`、`restored`（`BorrowedPaste.kt:84-86, 117, 276-298, 319-326`）。现有规则把 `pending` 和 `unknown` 一起当"结果未知"（`BorrowedPaste.kt:122`），实施时 `pending` 应同样归"恢复状态未知、退出码 3"。
    - 契约样本 `pasteResults` 里的 `restore-failed`（现在 `failed`）和 `restore-unknown`（现在 `UNKNOWN`）正好都是"效果已核对一致"的情形，结果词会变成 `done`，要重导；主机 `test_wire_contract.py:1245-1275` 和设备 `BorrowedPasteTest.kt:495-508` 的期望要改。`test_text_input.py:106-107` 已经期望退出码 3 和 1，不用改。
    - 结果为完成时现在会删掉 `reason`（`BorrowedPaste.kt:128-129`），改后要保留一句说明恢复失败或未知，否则首行看不出为什么退出码非零。

11. **第 14 条答复的范围列举漏了两个命令，`notifications` 有歧义。** 打印观察正文的命令【源码】：`snapshot`、`find`、`locate`、`screenshot`、`wait`（成功时只对选择器和文字条件，失败时照打，`ui_client.py:688-691`）、所有走 `run_action` 的动作命令，其中包括 `open`、`back`、`home`、`recents`（`ui_commands.py:1097-1114`）和**不带子命令的 `notifications`**（展开通知栏，`ui_commands.py:438-462`）。答复里"即 `snapshot`、`find`、`wait`、滚动查找和动作命令附带的观察"是评审方当时的背景列举，漏了 `locate`、`screenshot`；后面"`notifications` 等其余命令不动"指的应是 `notifications list`、`notifications get`，展开通知栏那条是动作命令、在范围内。建议把列举改成"所有界面命令附带的观察正文（含 `locate`、`screenshot`、`open`、`back`/`home`/`recents`、不带子命令的 `notifications`）"，并把"不动"写成"`notifications list`、`notifications get`"。`shell` 的 4 KB 显示上限属实（`job_commands.py:47, 262-280`，超出时打印 `[display truncated]` 和导出命令）。

---

## 4. 需要用户裁决的新问题

### 问题：后台屏会话到期后，"要继续用后台屏请写 `--display virtual`"这句提示有调用者永远看不到的路径，怎么办

**背景。** 第 25 条细则 a 定了"不写 `--display` 时，会话已存在就跟随，不存在就在主屏新开"；细则 e 定了"同名会话在后台屏上到期的，到期报错里加一句提示"。按源码，到期报错只出现在到期之后**第一条需要会话的命令**（界面命令、`shell`、文件传输）上。下面几种情形里，调用者看不到这句提示，会话直接落到主屏：

- 到期后第一条命令就是 `session open`（不写 `--display`）。这在调用者"每段任务开头先开会话"的习惯下很常见。设备端在主屏新开、成功，并清掉到期记录（`Sessions.kt:219`）。
- 到期后先发了 `session close`，记录被清掉（`Sessions.kt:180`）。
- 用户在设备上点了"释放会话"（之后的报错是 `sessionReleased`，不是到期，`Sessions.kt:91-93`）。
- 常驻进程意外退出、自动重绑（第 19 条），新进程里没有到期记录。
- `status` 到期后只显示 `callerSession=none`（`Sessions.kt:357-362`）。

这些情形下，细则 d 的首行会写"落在主屏"，所以调用者**有机会**从首行自己发现；但它看不到"后台屏会话已经到期、要继续请写 `--display virtual`"这层因果。后台屏上的任务在到期时已被销毁（第 12 条维持"到期直接销毁"），所以落到主屏本身不会误操作后台屏，风险在于调用者接着在主屏上操作用户正在用的 App，而它以为自己还在后台屏上。

**候选。**

- **A：照现在的细则做，不扩展。** 提示只加在到期报错里；其余路径靠细则 d 的首行。得：改动最小，与用户原话字面一致。失：上面几种路径里调用者可能在主屏上操作用户的 App 而不自知，后台屏题这类任务最容易碰到。
- **B（推荐）：同一份到期记录，也在不写 `--display` 的 `session open` 上用一次。** 设备端在 `session open` 不带种类、同名会话的到期记录显示原来在后台屏时，照常在主屏新开（符合细则 a），但回执首行加同一句提示，然后清掉记录。第 19 条重绑的记录如果知道旧会话在后台屏上，也同样附这句。得：最常见的路径被覆盖，复用细则 e 的同一份记录，代码量很小（`checkChange` 里多读一次记录）。失：`session open` 的成功回执多一种文字；`session close`、设备端释放两条路径仍看不到。
- **C：同 B 的情形下改为拒绝。** 不带 `--display` 的 `session open` 遇到"同名会话在后台屏上到期"时报参数错误、未发出，要求显式写 `--display main` 或 `virtual`。得：调用者不可能在不知情时落到主屏。失：多一种失败路径；与细则 a"不存在就默认主屏"的字面有张力，需要用户明确同意这个例外。
- **D：同 B，再让 `status` 显示到期。** `callerState` 增加 `expired`（附原来的显示种类），`status` 的输出写一行提示。得：覆盖面最大。失：`status` 的输出和指引都要改，`status` 是否清掉记录也要再定。

**剩余未知。** 调用者实际多常在到期后直接 `session open`，没有数据；本轮后台屏题 5 次运行里会话都没有到期。

除此之外没有发现需要用户另行裁决的冲突。

---

## 5. 上一遍（85 号记录）各条的核销表

"已改对"指现在的决定文件与源码一致；"不再适用"指用户的答复让这一条不必再改。

### 5.1 必须改的 6 处

| 85 号编号 | 内容 | 现状 |
| --- | --- | --- |
| 2.1 | A1 修法与规则不一致 | **改了但仍不准**：现在写"挪到自定义动作和标签读不到之前"，并声称这样 `action-1020037`、`action-1020044` 不再打印，这一句不成立（本报告 2.1）。`fill`、`set-progress` 照常打印、`copy`/`paste`/`cut` 由用户答复处理，这两点已改对 |
| 2.2 | 摘要漏了溯源行和缺口标记 | 已改对（第 14 条落实要点列了首行、溯源行、缺口行、命中摘要行、路径、缩小写法） |
| 2.3 | `find --raw` 漏写 | 已改对（第 23 条答复） |
| 2.4 | 第 7 组另一半用法 | 不再适用（用户答复第 7 组不做） |
| 2.5 | `--print` 现状写得不准 | 已改对（第 14 条落实要点第一条） |
| 2.6 | "20 次运行没人出错" | 不再适用（那一问已答，决定文件里已没有这句） |

### 5.2 建议收紧的 12 处

| 85 号编号 | 内容 | 现状 |
| --- | --- | --- |
| 3.1 | `--json` 上界 | 已改对（第四节第 1 处，192 KiB 和约 350 KB） |
| 3.2 | "补救"的说法 | 已改对（第四节第 1 处写了只有文本模式自动保存失败时照打、按完成算）。但同一处新加的"显式 `--print` 保存失败仍只是警告"缺限定（本报告 2.6） |
| 3.3 | `print_delivery` 的判断 | 已改对 |
| 3.4 | 三个诊断字段 | 已改对（第 18 条落实要点） |
| 3.5 | A2 的反射、措辞、C3 | 已改对（第四节第 9 处；码由用户定为 `busy`）。可补识别细节（本报告第 3 节第 9 条） |
| 3.6 | A4 范围 | 已改对 |
| 3.7 | 可见位的 6 个调用点 | 已改对（第四节第 2 处） |
| 3.8 | 第 13 条按窗口算、约 99 个 | 已改对（第 13 条答复和"已知的波及面"） |
| 3.9 | 退出码表补"完成加退出码 1" | 已改对（第四节第 9 处第 22 条） |
| 3.10 | 第二节第四轮第 16 项那一行 | 已改对 |
| 3.11 | 诊断说明短语在首行 | 已改对（第 15 条落实要点） |
| 3.12 | 第 21 条的落点 | 已改对（第 21 条落实要点；自身检查保留由用户答复） |

### 5.3 应补的 10 条风险，以及 C7、C8

| 85 号编号 | 内容 | 现状 |
| --- | --- | --- |
| 5.1 | 契约样本 2 个零面积节点 | 已收（第四节第 2 处） |
| 5.2 | 第 13 条之后调用者看到的变化 | 已收（第四节第 2 处）；"点击被拒时试试可见子节点或 `click --touch`"那句建议没收，可选 |
| 5.3 | 选择器解析到不可见父节点 | 已收（第四节第 3 处） |
| 5.4 | 第 12 条的锁 | 已收（第四节第 9 处） |
| 5.5 | 第 18 条要改的退出码表 | 已收（第四节第 9 处） |
| 5.6 | `--raw` 静默忽略投影选项 | 已收（第四节第 9 处、第 23 条答复），但选项清单不全（本报告 2.5） |
| 5.7 | 第 19 条标记按目标进程记 | 已收（第四节第 4 处） |
| 5.8 | 第 17 条超时回执补截断信息 | 已收（第四节第 8 处） |
| 5.9 | 第 11 条要改的指引和帮助 | 已收（第 11 条落实要点） |
| 5.10 | 第 10 条两个边角 | 已收（第四节第 7 处） |
| C7 | 第 22 条只改效果一致的情形 | 已收（第 22 条落实要点、第四节第 9 处） |
| C8 | 第 11 条两次请求共用时限 | 已收（第四节第 6 处） |

---

## 6. 替实施方核过的事实

1. **`action REF --action-name copy`（以及 `paste`、`cut`）能调用。**【源码】
   - 主机：`action` 命令有 `--action-name`，原样作为 `actionName` 发给设备（`ui_commands.py:811-861`），没有按名字过滤。
   - 设备：节点的每个动作带 `name`，由 `UiSemantics.actionName` 给出（`NodeFacts.kt:196-204`），其中 16384、32768、65536 分别叫 `copy`、`paste`、`cut`（`UiSemantics.kt:31-49`）。`NodeActions.resolve` 按名字找唯一一个当前广告的动作（`NodeActions.kt:79-83`），这三个不在"需要参数"集合里（`NodeActions.kt:11-24`），所以不会被拒为 `actionUnsupported`；随后以空参数调用平台动作（`UiActions.kt:107-135`）。
   - 限制：目标必须可用且可见（`NodeActions.kt:103-114`）；`copy`、`cut` 要有选区才有实际效果，效果核对可能报未知。按用户答复，这三个应保留打印。
2. **会话"已存在"的精确含义**见本报告 2.3；**到期报错**的构造位置和出现的命令见 2.4。
3. **`--replace` 不带 `--display`、`session takeover` 不带 `--display`、`session open` 不带 `--display`** 现在都报同一条缺参数错误【实跑】：`session open: not sent; Missing option '--display'. Choose from:\n\tmain,\n\tvirtual; inspect accepted syntax with: androidctl session open --help; code=arguments`。
4. **主机端判断"跟随"的竞态。** 如果主机先查 `status` 再决定发 `main` 还是 `virtual`，两次请求之间隔一次往返（本地网关加局域网，通常几十毫秒到数百毫秒）。窗口里可能发生：会话恰好到期（主机按 `virtual` 发出，设备新开一个后台屏，违反"新开默认主屏"）；同名的另一个调用者（指引允许同名调用者共享租约，`SKILL.md:23`）关闭或替换了会话（得到 `displayMismatch` 或开错种类）。后果不至于破坏数据，但会让行为不可预期，还多一次往返。设备端在会话锁里判断没有这些问题，所以"宜在设备端判断"成立。
5. **第 10 条的五个字段都拿得到。**【源码】`UiSemantics.identity`（`UiSemantics.kt:320-334`）返回的 `SelectionIdentity`（`SelectionIdentity.kt:8-15`）正好包含：窗口 `windowId`、来源节点编号 `sourceId`、角色 `role`（取自 `view.role`）、行标签 `rowLabel`（取自 `row.label`）、行列位置 `positions`（取自 `row.chain`），另加名字 `name`。严格判定可以写成"两边身份去掉名字之后相等，名字不同，且恰好一个"。名字、角色或行信息不完整时 `identity` 返回空，这时按落实要点只报拒绝。
6. **第 11 条衔接。** `find` 命中 0 条时结果是完成；采样不完整时首行已经有 `observationWarning=partial; missing=N`（`ui_output.py:169-178`），来源是这次正文对应样本的完整性（`ui_client.py:829-834`）。答复要求的"首行带采样不完整的标记"现成就有，只要第二次请求的样本驱动这个字段即可；第 15 条把它改成短标记时一并处理。
7. **第 16 条。** `wait` 的引用列表由设备端生成（选择器 `UiCondition.kt:144-160`，消失 `162-175`，文字 `197-206`），滚动查找文字模式也生成（`ScrollContainer.kt:27-50`）；主机两者都用同一行打印整个条件对象（`ui_output.py:153-154` 的 `match=`）。所以"首行只列前 20 个加总数"在主机这一处改就能同时覆盖两个命令。滚动查找的选择器模式现在不生成引用列表，只有单个 `ref` 和歧义候选文字（`ScrollContainer.kt:87-113`），第 16 条要在设备端补上。`find` 的默认 20 条在 `ui_view.py:103`。
8. **第 17 条。** 等待条件只有六种：文字（`--contains`）、应用（`--app`）、消失（`--gone`）、`idle`、画面变化（`--screen-change`）、选择器（按角色、名字等），外加组合它们的 `--any-of`（`UiCondition.kt:21-28`；`ui_commands.py:583-645`）。答复把"消失"和 `idle` 留下提前失败，其余四种取消，这恰好对应"成功条件本身要求完整的树"的两种：消失要求完整、零命中、零未知（`UiCondition.kt:175`），`idle` 只接受完整的树（`UiPostAction.kt:183-197`）。答复覆盖了全部条件种类。现在的提前失败是对整次评估统一做的（`UiCondition.kt:424-431`），改成"所有条件都是消失或 `idle` 时才提前失败"即可实现评审方补的那条多条件读法。
9. **第 24 条。** 第 1 到 6 组的构造位置与 85 号记录一致。按旧码分支的地方只有一处设备内部分支：`UiCondition.kt:335` 捕获 `noPublishedSnapshot`，把 `wait --screen-change` 缺基线的情形报成未发出。合并后要把这个集合改成含 `noSnapshot`，否则会变成失败、退出码由 2 变 1。82 号记录已经写了这一处，决定文件引用了 82 号记录，不算遗漏。`identityChanged` 只在 `state.py:288-290`（配对时证书变化）构造，主机没有任何地方按它触发"重新 `device trust`"之类的处理，提示只在消息文字里。
10. **第 5 条"后台屏加长网页"的检查。** 用现有命令能直接做的：`session open --display virtual`、`open URL`（`open` 接受网址，`ui_commands.py:1124-1126`）、默认 `snapshot`（看是否超限、是否给了摘要和文件）、`find` 加 `click`。另外两件（"标为可见的节点数与实际在屏内的相符"、"屏内有面积的节点都在默认输出里"）没有命令能直接给出，要用 `snapshot --raw` 存下的 JSON 离线写脚本数。主屏对照要先换显示：如果 Chrome 原来在主屏上有任务、被搬到了后台屏，`session open --display main --replace` 或 `session close` 会被任务保护拦下（`displays.md:56-62`），需要 `--return-tasks`。

---

## 7. 没能核实的内容

- 所有 Android 运行时行为：A2 的 `IllegalStateException` 在真实设备上的消息（只读了 AOSP 源码，厂商 ROM 可能改字）；`action --action-name copy` 在真实 App 上的效果；第 19 条两路死亡回调的先后。
- 第 3 节第 7 条里"一次往返几十到数百毫秒"是按架构推断的量级，没有测。
- A1 的"0x0102xxxx 段都是平台标准动作"只按 AOSP 16 的源码和文档推出，没有在设备上找反例。
- 第四节第 1 至 9 处中本报告没有重新提到的内容，上一遍已经核过，这一遍只抽查了行号，没有逐句重核。
- 第二节状态表里第四轮第 12、15、16 项和第五轮各行，我对照了第四轮决定文件原文和源码现状，没有发现不符；第五轮决定文件原文没有逐行重读。
