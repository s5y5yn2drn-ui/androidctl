# 第七轮专题核查 96：网页题与长列表题的真实任务轨迹逐条阅读

## 摘要

1. 10 次运行全部业务成功，答案逐项正确（我对照了原件里的评论原文和阅读页正文）。没有一次读 `--help`，也没有参数写法错误。2 次被拒都在 1 次调用内恢复。没有出现 UNKNOWN。【实测】
2. 网页题和上一轮同题相比：回合中位数 15→13，总输入中位数 557,402→387,934（约 −30%）；命令输出总字节 661 KB→314 KB（−53%）。原因有两个：标准无障碍动作编号不再印进默认输出；按引用读屏外节点不再返回空正文。长列表题几乎没变：回合中位数 14→14，输入中位数 393,353→395,456，命令序列基本照旧。【实测】
3. 最大的剩余摩擦：网页题 5/5 次点开 issue 链接后，回执写 `changed`，标题却还是列表页，于是 5/5 次补一次 `snapshot`。第六轮清单乙项 B2 是"必须做"，原因在于实现只把"仅边界变化"判为可能还在变化；本轮同一事件在回执里表现为目标的 `focused,actions` 变化，所以这条规则一次也没生效。实施方把 B2 记为"已闭"，真实任务的证据不支持。【实测＋源码】
4. 系统 `back` 动作后取观察时没有"动作前样本"（源码 `SystemActions.kt:56`），两次相同的取样就算稳定，所以回执可能把返回前的页面当成结果。16/16 条 `back` 回执都是 `unsettled`；长列表题有 3/5 次第二个 `back` 的回执仍显示阅读页，模型只好补 `snapshot`。【实测＋源码】
5. 网页题两次成本离群（677K、519K）都来自"超限摘要→读保存文件"这条路，约占各自总输入的 21% 和 26%【估算】。它由三处产品提示合力引出：`find` 省略行的提示 `read="snapshot --cached --all"`；超限摘要里给的 `snapshot --cached --depth 2`（对 Chrome 只能看到外层包装节点，没有用）；`--all` 会重新打开全部动作字段，保存文件里 41%–50% 是动作编号噪声，模型最后只好自己用 `-replace` 把它们滤掉。
6. 长列表题 5/5 次都截图看图，用来确认整张卡片可见。题目要求"完整可见"，而界面树没有"被裁切／部分可见"这项事实。另有 1 次照首页"advertised 时用 `reveal`"的写法去试 `reveal`，被拒了：默认输出根本不印 `show-on-screen` 这个动作（`ui_view.py:55-68`），所以"是否宣告了 reveal"这一条件调用者看不到。
7. 总体判断：这批轨迹支持合入，没有发现挡合入的问题。以上各点都只是多花调用或多花输入，没有一处误导调用者去重复执行或执行错误的动作。按"修一处能省多少调用"排的改进清单见第 7 节；前三条（B2 补全、`back` 的取样基准、两处提示文字）代价小、收益明确，建议进下一轮的第一批。

---

## 0. 范围、方法与依据

**材料。** 本轮 10 次运行：`ultragoal/2026-11-11-next-improve/stages/42-r4-decisions/efficiency/r6-c2-candidate-01..05`（网页题）和 `r6-s1-core-candidate-01..05`（长列表题）。上一轮同题 10 次：第六轮材料包同路径下的 `r5-c2-final-candidate-01..05` 和 `r5-s1-core-candidate-01..05`。上一轮阅读记录：`/home/user/androidctl/review/sub-reports/73-trajectories.md`。

**题面。** 模型收到的任务正文是加密的，我从第六轮包的 `efficiency-preparation.md` 第 106 行和第 136 行取了冻结题面：
- 网页题：从 CPython Issues 默认的 open 列表出发，搜索并打开已关闭的 "AsyncIO's wait_for can hide cancellation in a rare race condition"，报告编号、状态、维护者 2023 年 5 月关于修复版本和旧分支回移的说明，附作者、日期和真实评论链接，然后回到同一筛选结果。
- 长列表题：在 Feeder 的 `R4 RFC list` 顶部找到 RFC 10080，使整张目标卡片可见，打开阅读；报告标题、官方链接，以及摘要里涉及的邮件头字段、发送者和接收者各自能做什么；然后回到同一列表；不改其它文章。

**指引版本。** 我把模型实际收到的 `SKILL.md` 和 `actions.md` 从原生记录里取出来，与 `f401deb` 的 `skills/androidctl/` 逐字比对，只差文件末尾的换行。【实测】

**脚本。** 工作目录是 `S/agents11/96/`（`S` 为 scratchpad 根目录），所有脚本都用 `python3 -I` 运行，材料路径作为参数传入：
- `tl.py`：逐次时间线，产出 `tl/<运行>.md`、`.full.txt`、`.short.txt`；
- `compact.py`：步骤表，产出 `compact.txt`；
- `stats.py`：回合、输入、命令计数，产出 `stats.txt`；
- `noise.py`、`noise2.py`：输出构成统计。

"回合"等于原生记录里 `token_count` 事件的个数，与实施方成本页的轮数逐次一致。

**依据性质的标注。** 【实测】指真实运行的原件；【源码】指 `f401deb` 的 `文件:行号`；【估算】指按逐回合输入增量推算。

---

## 1. 逐次步骤表

**说明。**
- r 是回合号。第 1 回合报线程号、读首页，这是题目规定的动作，两轮都有。
- "字节"是该条命令交给模型的输出字节。
- 首行里的长保存路径缩写为 `…Path`。
- 原生记录里模型的推理摘要是加密的，评论性的中间发言也很少（每次 1 到 2 句）。所以"反应"一栏是按下一条命令推断的，凡是引用原话的我都标了出来。
- 标记：**[拒]** 被拒或失败；**[补]** 为确认上一步而多发的调用；**[读]** 读指引或读保存文件；**[看]** 截图或看图；**[弯]** 弯路。

### 网页题第 1 次（19 回合，15 条命令，总输入 677,133）

| r | 命令 | 退出 | 字节 | 首行（摘） | 反应和标注 |
|---|---|---|---|---|---|
| 1–2 | 读线程号；读 `SKILL.md` | 0 | 7,173 | — | 开场 |
| 3 | `snapshot` | 0 | 7,007 | `snapshot: done; …Path` | 取搜索框的引用 |
| 4 | `fill e950415716 "is:issue state:closed AsyncIO wait_for" --submit` | 0 | 6,064 | `fill: done; change=loading … (page may still be transitioning); fill=done, verification="matches"…` | 按首页规则下一步用 `find` |
| 5 | `find "hide cancellation"` | 0 | 832 | `find: done; fresh; matches=0` | 结果页共 103 条，目标不在第一页。**[弯]** 检索词是模型自己选的 |
| 6 | `snapshot --cached --all` | 0 | 590 | `Observation exceeds 65536 UTF-8 bytes…; Narrow the current sample: androidctl snapshot --cached --depth 2` | **[弯]** 照着 `find` 省略行的提示 `read="snapshot --cached --all"` 去做 |
| 7 | 读保存文件（`Get-Content`） | 0 | 75,345 | `Warning: truncated output (original token count: 42011)` | **[读][补]** 被执行框架截断；文件里 50% 是动作编号 |
| 8 | `fill … "is:issue state:closed hide cancellation" --submit` | 0 | 6,141 | `fill: done; change=loading …` | 换检索词 |
| 9 | `find "hide cancellation"` | 0 | 4,134 | `find: done; fresh; matches=2; offscreenMatches=1` | 目标链接行末带 `offscreen` |
| 10 | `click e950416952` | **2** | 205 | `click: not sent; … target is not visible; reveal it with androidctl reveal e950416952, then retry; code=notReady` | **[拒]** 首页没写"屏外目标先 reveal" |
| 11 | `reveal e950416952` | 0 | 5,889 | `reveal: done; change=changed at content,e950416952:visible,bounds` | 照拒绝回执的建议做 |
| 12 | `click e950416952` | 0 | 5,419 | `click: done; change=changed at content,e950416952:focused,actions` | 回执的标题仍是 `Issues · python/cpython` |
| 13 | `snapshot` | 0 | 5,686 | `snapshot: done` | **[补]** 回执判成 `changed`，但仍是旧页。模型原话："已找到并打开 #86296，页面状态为 Closed。接下来核对 2023 年 5 月的维护者评论" |
| 14 | `find "May" --all` | 0 | 8,611 | `find: done; cached; matches=4; offscreenMatches=4` | 只有 4 条命中，`--all` 的唯一作用是多印字段（约 45% 是动作编号） |
| 15 | `snapshot --cached e950417509` | 0 | 1,586 | `snapshot: done; cached` | 屏外子树正文正常返回（B3 已修好）；只拿到评论头部 |
| 16 | `snapshot --cached e950417508` | 0 | 2,632 | 同上 | **[补]** 改读父节点（列表项），拿到正文和永久链接。原因在页面结构：正文是头部的兄弟节点 |
| 17 | `back` | 0 | 5,459 | `back: done; change=unsettled (page may still be transitioning)` | 回执标题已是列表，地址栏里有查询串 |
| 18 | `snapshot` | 0 | 5,368 | `snapshot: done` | **[补]** 核对返回页 |
| 19 | 最终回答 | | | | 正确 |

### 网页题第 2 次（13 回合，10 条命令，353,674）

| r | 命令 | 退出 | 字节 | 首行（摘） | 反应和标注 |
|---|---|---|---|---|---|
| 3 | `snapshot` | 0 | 7,087 | | |
| 4 | `fill e499760759 "is:issue state:closed AsyncIO wait_for hide cancellation" --submit` | 0 | 7,517 | `fill: done; change=unsettled …; observationWarning="partial; missing=19"` | 正文仍是搜索前的旧列表（7.5 KB 旧页） |
| 5 | `find "rare race"` | 0 | 3,013 | `find: done; fresh; matches=1; offscreenMatches=0` | 一次命中 |
| 6 | `click e499761373` | 0 | 7,546 | `click: done; change=changed at content,e499761373:focused,actions` | 标题仍是列表页 |
| 7 | `snapshot` | 0 | 5,857 | | **[补]** 同第 1 次第 13 回合 |
| 8 | `click e499761442`（"Jump to bottom"） | 0 | 4,337 | `click: done; change=unsettled at content,…:bounds,roleDescription` | 模型选的路线 |
| 9 | `snapshot` | 0 | 4,620 | | **[补]** 回执是 `unsettled` |
| 10 | `scroll-until e499760689 up --max-steps 15 --contains "May" --timeout 30s` | 0 | 9,457 | `scroll-until: done; change=changed at content; match={…"matches":3…}; stop=found` | 一次拿到作者、日期、永久链接和正文。模型原话："gvanrossum 在 2023 年 5 月 24 日说明修复进入 3.12…评论永久链接已取得。现在返回原筛选结果" |
| 11 | `fill e499760670 "https://github.com/python/cpython/issues?q=…" --submit`（地址栏） | 0 | 2,826 | `fill: done; change=unsettled …; loading detected` | 用地址栏回到筛选结果，代替 `back` |
| 12 | `snapshot` | 0 | 7,615 | | 核对返回页 |

### 网页题第 3 次（12 回合，9 条命令，314,586，本组最短）

| r | 命令 | 退出 | 字节 | 标注 |
|---|---|---|---|---|
| 3 | `snapshot` | 0 | 6,477 | |
| 4 | `fill e361703251 "is:issue state:closed AsyncIO's wait_for can hide cancellation in a rare race condition" --submit` | 0 | 6,279 | |
| 5 | `find "AsyncIO"` | 0 | 5,235 | 5 条命中 |
| 6 | `click e361703870` | 0 | 7,742 | `changed at content,…:focused,actions`，标题仍是列表页 |
| 7 | `snapshot` | 0 | 5,857 | **[补]** |
| 8 | `find "May" --all` | 0 | 8,591 | 4 条命中；`--all` 只带来噪声 |
| 9 | `snapshot --cached e361704353` | 0 | 1,586 | 评论头部 |
| 10 | `snapshot --cached e361704352` | 0 | 2,632 | **[补]** 父列表项，拿到正文 |
| 11 | `back` | 0 | 7,722 | `unsettled`。回执已显示列表标题和查询串，模型直接据此作答，没有再核对 |

### 网页题第 4 次（13 回合，11 条命令，387,934，等于中位数）

| r | 命令 | 退出 | 字节 | 标注 |
|---|---|---|---|---|
| 1 | 读线程号和 `SKILL.md`，同一回合 | 0 | 7,211 | |
| 2 | `snapshot` | 0 | 6,628 | |
| 3 | `fill … "is:issue state:closed AsyncIO wait_for hide cancellation" --submit` | 0 | 6,266 | `loading` |
| 4 | `find "AsyncIO"` | 0 | 5,183 | |
| 5 | `click e919273712` | 0 | 7,558 | `changed … focused,actions`，旧标题 |
| 6 | `snapshot` | 0 | 5,750 | **[补]** |
| 7 | `click e919273780`（"Jump to bottom"） | 0 | 6,902 | `unsettled … :bounds` |
| 8 | `snapshot` | 0 | 4,632 | **[补]** |
| 9 | `scroll-until e919273029 up --max-steps 12 --contains "May" --timeout 30s` | 0 | 9,469 | 一次到位 |
| 10 | `back` | 0 | 9,422 | 仍是 issue 页：页内锚点在浏览历史里多留了一步。**[弯]**，由页面和模型选的路线造成 |
| 11 | `back` | 0 | 6,099 | `unsettled; observationWarning="partial; missing=20"`，已回到列表 |
| 12 | `snapshot` | 0 | 7,490 | **[补]** 核对返回页 |

### 网页题第 5 次（14 回合，10 条命令加 3 次读文件，519,095）

| r | 命令 | 退出 | 字节 | 标注 |
|---|---|---|---|---|
| 1–4 | 同第 4 次：读首页、`snapshot`、`fill … "is:issue state:closed wait_for hide cancellation" --submit`、`find "AsyncIO"` | 0 | | |
| 5 | `click e659418523` | 0 | 7,443 | `changed … focused,actions`，旧标题 |
| 6 | `snapshot` | 0 | 5,857 | **[补]** |
| 7 | `click e659418592`（"Jump to bottom"） | 0 | 3,370 | `unsettled at windows,content,…:visible,bounds` |
| 8 | `snapshot --all` | 0 | 642 | 超限摘要，并建议 `--depth 2`。**[弯]** |
| 9 | 读保存文件（`Get-Content`） | 0 | 75,574 | **[读]** 截断（原 64,775 token） |
| 10 | `Select-String -Pattern 'May.*2023\|backport\|3.12\|3.11' -Context 3,8` | 0 | 75,597 | **[读]** 仍被截断：每行动作字段太长 |
| 11 | `Get-Content … \| Select-Object -Skip 532 -First 90 \| ForEach-Object { $_ -replace ' action-name="[^"]*"','' }` | 0 | 21,016 | **[读]** 模型自己写过滤，去掉动作编号后才读到评论 |
| 12 | `back` | 0 | 10,940 | 仍是 issue 页（锚点历史） |
| 13 | `back` 加 `snapshot`，同一回合 | 0 | 14,908 | **[补]** |

### 长列表题第 1 次（14 回合，13 条命令，2 次看图，389,116）

| r | 命令 | 退出 | 字节 | 首行（摘） | 标注 |
|---|---|---|---|---|---|
| 3 | `snapshot` | 0 | 4,196 | | `list … scroll="forward down" rows=16`，顶部是 RFC 10090 |
| 4 | `scroll-until e638907335 down --max-steps 10 --contains "RFC 10080" --timeout 15s` | 0 | 5,460 | `scroll-until: done; change=changed at content; match={…"refs":["e638907461"]…}; stop=found` | 一次命中。目标是最后一行（`rowIndex=15`），列表仍可往下滚 |
| 5 | `snapshot --screenshot --marks` | 0 | 5,708 | | **[看]** 判断是否完整可见 |
| 6 | 看图；读 `actions.md`（11 KB） | | 11,334 | | **[看][读]** 图上卡片下沿被导航栏和悬浮按钮压住 |
| 7 | `scroll e638907335 down`；`snapshot --screenshot` | 0 | 8,784 | `scroll: done; change=changed at content,e638907335:actions` | 列表变成 `scroll="backward up"`，已到底 |
| 8 | 看图；`click e638907458` | 0 | 1,234 | `click: done; change=changed at content` | 阅读页正文一次到位。模型原话："已使目标卡完整可见并打开。摘要明确涉及 `Expires` 邮件头…正在从文章页面确认官方链接" |
| 9 | `click e638907496`（"Open menu"） | 0 | 740 | 弹出菜单：Share、Mark as unread、Save、Read aloud | **[弯]** 菜单里没有链接，属于页面探索 |
| 10 | `back`；`click e638907492`（"Open in Web view"） | 0 | 2,125 | 第二条：`click: done; change=unsettled at windows; snapshotProblem={"code":"scopeUnknown","message":"…inspect a listed window: snapshot --window 14; snapshot --window 18"}` | 回执没有树 |
| 11 | `snapshot` | 0 | 7,639 | WebView，`url_field` 值为 `https://www.rfc-editor.org/info/rfc10080/` | **[补]** |
| 12 | `back`；`back` | 0 | 2,518 | 两条都是 `unsettled`，**两条都显示阅读页** | 第二个回执取样过早 |
| 13 | `snapshot` | 0 | 4,010 | 列表，目标 `Read`，相邻项 `Unread` | **[补]** |

### 长列表题第 2 次（14 回合，12 条命令，2 次看图，395,456，等于中位数）

| r | 命令 | 标注 |
|---|---|---|
| 1 | 读线程号和 `SKILL.md` | |
| 2 | `snapshot` | |
| 3 | `scroll-until e235467380 down --max-steps 12 --contains 'RFC 10080' --timeout 20s` | 一次命中 |
| 4 | `snapshot --screenshot`；读 `actions.md` | **[看][读]** |
| 5 | 看图 | **[看]** |
| 6 | `scroll e235467380 down --screenshot` | |
| 7 | 看图 | **[看]** |
| 8 | `click e235467503`（目标卡） | 阅读页 |
| 9 | `click e235467537`（"Open menu"） | **[弯]** |
| 10 | `click`（Close menu）；`click e235467533`（"Open in Web view"） | `scopeUnknown`，没有树 |
| 11 | `snapshot` | **[补]** 拿到链接 |
| 12 | `back`；`back` | 两条回执都显示阅读页 |
| 13 | `snapshot` | **[补]** 列表 |

### 长列表题第 3 次（16 回合，11 条命令，2 次看图，438,282）

| r | 命令 | 标注 |
|---|---|---|
| 3–8 | `snapshot`；`scroll-until …`；`snapshot --screenshot`；看图并读 `actions.md`；`scroll … down --screenshot`；看图 | 与第 1、2 次相同 |
| 9 | `click e309769976` | 阅读页 |
| 10 | `click e309770011`（"Open menu"） | |
| 11 | `click e309770019`（"Share"） | `unsettled …; No primary accessibility node reply was available by the collection deadline; the app may be unresponsive; observationWarning="partial; missing=1"` |
| 12 | `snapshot` | **[补]** 分享面板预览里有 `https://www.rfc-editor.org/info/rfc10080/`，没有发送 |
| 13 | `back` | 回到阅读页 |
| 14 | `back` | `unsettled`，**仍显示阅读页** |
| 15 | `snapshot` | **[补]** 列表 |

### 长列表题第 4 次（14 回合，12 条命令，1 次看图，339,899）

| r | 命令 | 退出 | 标注 |
|---|---|---|---|
| 4 | `scroll-until e668236771 down --max-steps 20 --contains "RFC 10080" --timeout 30s` | 0 | 一次命中 |
| 5 | `reveal e668236894` | **2**（模型的包装层把它报成 1） | **[拒]** `reveal: not sent; This target does not advertise reveal; click it directly with androidctl click e668236894, or use androidctl snapshot to find a container to scroll; code=actionUnavailable` |
| 6 | `scroll e668236771 down` | 0 | 列表变成 `backward up` |
| 7 | `snapshot --screenshot` | 0 | **[看]** |
| 8 | 看图 | | **[看]** |
| 9 | `click e668236894` | 0 | 阅读页 |
| 10 | `click e668236928`（"Open menu"） | 0 | **[弯]** |
| 11 | `click`（Close menu）；`click e668236924`（"Open in Web view"） | 0 | `scopeUnknown` |
| 12 | `snapshot` | 0 | **[补]** |
| 13 | `back`；`back` | 0 | 第二条回执已是列表，模型据此作答 |

### 长列表题第 5 次（15 回合，11 条命令，2 次看图，410,891）

| r | 命令 | 标注 |
|---|---|---|
| 3–8 | 与第 1、3 次同序：`snapshot`、`scroll-until`、`snapshot --screenshot`、看图并读 `actions.md`、`scroll --screenshot`、看图 | |
| 9 | `click e216825557` | 阅读页 |
| 10 | `click e216825591`（"Open menu"） | **[弯]** |
| 11 | `click`（Close menu）；`click e216825587`（"Open in Web view"） | `scopeUnknown` |
| 12 | `snapshot` | **[补]** |
| 13 | `back` | 回到阅读页 |
| 14 | `click e216825577`（"Go back"） | `click: done; change=changed at content`，列表一次到位，不必补查 |

**小结。** 10 次里共有 2 条非零退出：网页题第 1 次的 `click`（屏外），长列表题第 4 次的 `reveal`（没宣告）。两条都是"未发出"，`dispatchStarted=false`，都在 1 次调用内恢复。没有任何 `--help`。指引的阅读情况：首页 10/10 次读；`actions.md` 长列表题 4/5 次读（全部和别的调用同回合发出，不多占回合），网页题 0/5 次。

---

## 2. 摩擦点归因

### 2.1 逐类统计（10 次合计）

下表的"多花的调用"指相对第 7 节最短路径多出来的命令；同回合批量发出的不另计回合。

| 摩擦 | 出现 | 多花 | 归因 | 依据 |
|---|---|---|---|---|
| 点开 issue 后回执判 `changed`、标题仍是旧页，补 `snapshot` | 网页 5/5 | 5 条命令、5 回合 | **产品输出给的信息有误**（B2 没覆盖这种形态） | 【实测】见 2.2 |
| 系统 `back` 回执取样过早或一律 `unsettled`，补 `snapshot` | 网页 2/5（第 1、4 次；第 5 次同回合连发），长列表 3/5（第 1、2、3 次） | 约 6 条 | **产品**：`back` 没有动作前样本，两次相同就算稳定 | 【实测＋源码】见 2.3 |
| 点击打开另一个 App（WebView）后回执没有树，补 `snapshot` | 长列表 4/5（第 3 次走分享面板，同样 partial 后补查） | 5 条 | **产品**：落地窗口还没枚举出来就交回执；附带的建议还指向系统窗口 | 【实测】 |
| 超限摘要后读保存文件 | 网页 2/5（第 1 次 2 条，第 5 次 4 条） | 6 条，约 27 万输入 token【估算】 | **产品提示引导**（省略行和超限摘要）＋**噪声**（`--all` 的动作字段）＋模型选择（第 5 次先发 `snapshot --all`） | 【实测＋源码】见 2.4 |
| 截图加看图，确认整卡可见 | 长列表 5/5（9 次看图） | 每次约 2 回合 | **题目要求**＋**产品没给"部分可见"事实** | 【实测】见 2.5 |
| `scroll-until` 命中后再滚一次 | 长列表 5/5 | 每次 1 条 | **页面本身需要**：命中时目标是最后一行，被导航栏压住。用户已裁决"看到文字即成功"，这是预期结果 | 【实测】 |
| `reveal` 被拒（目标没宣告） | 长列表 1/5 | 1 | **指引和输出不一致**：首页说"advertised 时用 reveal"，默认输出却不印这个宣告 | 【实测＋源码】见 2.6 |
| 点击屏外目标被拒，再 `reveal` | 网页 1/5 | 1 | **指引没写**"带 `offscreen` 的目标先 reveal"；产品的拒绝回执写得好，一步恢复 | 【实测】 |
| 第一次按引用读评论只拿到头部，再读父节点 | 网页 2/5 | 2 | **页面结构**（正文是兄弟节点）＋模型选择 | 【实测】 |
| 锚点历史，多一次 `back` | 网页 2/5 | 2 | **页面本身**，加上模型选了"Jump to bottom" | 【实测】 |
| 打开菜单找链接 | 长列表 5/5 | 每次 1 条 | **页面本身**：Feeder 菜单里没有链接 | 【实测】 |
| 第一次检索词太宽，换词重搜 | 网页 1/5 | 2 | **模型的选择** | 【实测】 |
| `find "May" --all` 只有 4 条命中仍加 `--all` | 网页 2/5 | 0 条命令，约多 4 KB | **指引写得不全**：首页说 `--all` 是"超过默认 20 条"，没说会打开全部字段 | 【实测＋源码】 |

### 2.2 B2 的实现漏掉了它要修的那个真实事件（网页 5/5）

**上一轮的原件。** 上一轮网页题点开 issue 的回执是这样的（第六轮包 `r5-c2-final-candidate-02` 第 5 回合）：
```
click: done; change=changed at content,e784476896:bounds baselineAgeMs=10222; …
screen app="com.chrome.beta" title="Chrome Beta: Issues · python/cpython · GitHub"
```

**本轮的原件。** 同一步变成这样（`r6-c2-candidate-01` 第 12 回合；第 2–5 次逐字同形）：
```
click: done; change=changed at content,e950416952:focused,actions; snapshotPath=…
screen app="com.chrome.beta" title="Chrome Beta: Issues · python/cpython · GitHub" keyboard="hidden"
```

**实现。** `UiPostAction.kt:245-253` 的 `navigationSettlement` 只在 `onlyGeometryChanged` 成立时改判 `UNSETTLED`。`onlyGeometryChanged`（`:269-281`）要求去掉边界之后，整份内容签名完全相同。点击之后链接自己获得焦点、动作表也随之变化，所以签名不同，结论仍是 `CHANGED`。【源码】

**与裁决的关系。** 第六轮决定清单乙项 B2 的原文是"只看到边界变化而标题没变……标成可能还在变化"，具体判定交给实施方。实施方的计划表 `ST/r6-plan.md` 第 49 行写的是"导航类只有几何变／标题不变或有明确 loading 不标 CHANGED……已闭"，但代码只实现了"只有几何变"。

**真实后果。**
- 5/5 次各多一次 `snapshot`。上一轮是 4/5 次。
- 更隐蔽的风险是：`changed` 回执之后紧跟的 `find` 读的是旧页缓存。第 11 条裁决的"0 条命中就新采"把多数情形兜住了，但如果旧页上恰好也有同样的字，`find` 会返回旧页的命中，而且不会新采。
- 本轮没有模型踩到：5 次都先看了标题再补 `snapshot`。

**最小修正。** 对导航类动作，如果窗口和标题都没变，而且变化只落在被派发的那个目标自身的 `focused`、`selected`、`actions`、`bounds` 上，就判 `UNSETTLED`，并在预算内继续取样，等标题或窗口变化。代价是一个 Kotlin 函数加一条行为测试。

**分类。** 确定缺陷（实现没有覆盖裁决所举的那个真实情形）。去向：**进下一轮**。它不误导执行，只多 1 次调用。但建议主报告指出，"B2 已闭"的说法与实测不符。

### 2.3 `back` 回执：没有动作前样本，两次相同就当稳定（长列表 3/5，网页回执 16/16 都是 `unsettled`）

**源码。**
- `SystemActions.kt:56`：`back`、`key`、`home`、`recents`、`notifications` 都调 `settle(run, null)`，`open` 在 `:193` 也一样。
- `UiPostAction.kt:63-65`：没有 `before` 时，`stableWithoutBaseline()`（`same >= 2`）就结束取样。
- `:216`：`baseline == null` 时一律判 `UNSETTLED`。

**后果。**
- 回执永远写 `change=unsettled (page may still be transitioning)`，所以"真的还在变"和"没有比较基准"无法区分。
- 当返回动画还没开始、头两次取样都是旧页时，回执就把旧页当结果交出来。长列表题第 1、2、3 次的第二个 `back` 都是这样，原件如下（`r6-s1-core-candidate-03` 第 14 回合，第 15 回合补的 `snapshot` 才是列表）：
```
back: done; change=unsettled (page may still be transitioning); snapshotPath=…
screen app="com.nononsenseapps.feeder" title="Feeder" keyboard="hidden"
text e309770079
 text "RFC 10080: Updated Use of the Expires Message Header Field"
 …
item e309770089 "Go back" clickable
```

**对照。** 长列表题第 5 次点阅读页里的 "Go back" 按钮，回执是 `click: done; change=changed at content`，列表一次到位。普通点击有动作前样本可比，系统返回键没有。

**指引。** 首页写的是"Collect a fresh `snapshot` for a transition"。模型照做是对的，代价是多一次调用。

**最小修正。** 系统按键类动作用会话里最近一次完整样本当基准（同一连接、样本足够新时）。这样"两次相同"就必须先和基准不同才算稳定，回执也能报出 `changed`。代价中等：涉及取样时机，需要补测试。

**分类。** 设计取舍兼可修缺陷。去向：**进下一轮**。

### 2.4 超限摘要和省略行把调用者引向最贵的读法（网页 2/5，两次成本离群）

**原件。** 网页题第 1 次：
```
find: done; fresh; matches=0; offscreenMatches=0; …
find query="hide cancellation" matches=0 offscreen=0 shown=0 read="Use scroll-until CONTAINER_REF DIRECTION --max-steps N with a target selector for uncollected content"
omission reason="find" nodes=654 read="snapshot --cached --all" ref="e950415517"
```
下一回合模型照着执行 `snapshot --cached --all`，得到：
```
snapshot: done; cached; snapshotPath=…
androidctl capture="3f02b61c-…:11" snapshotCached=true
screen app="com.chrome.beta" title="Chrome Beta: Issues · python/cpython · GitHub" keyboard="hidden"
Observation exceeds 65536 UTF-8 bytes; full text requires a saved file.
Narrow the current sample: androidctl snapshot --cached --depth 2
```

**源码。**
- `ui_view.py:897-898`：所有省略行都写死 `read="snapshot --cached --all"`。
- `ui_client.py:887-889`：超限摘要写死 `snapshot --cached --depth 2`。
- Chrome 的树在网页根节点之上有 9 到 10 层包装节点。`find` 的祖先链原件就是 `text e…517 → … id="compositor_view_holder" → …`，所以 `--depth 2` 只能看到包装节点。

**噪声。** 保存文件里的动作字段噪声（`noise.py` 的结果）：网页题第 1 次的文件 168,039 字节，其中标准动作 `action-name` 占 50.3%，`+show-on-screen` 占 6.2%；第 5 次的文件 259,096 字节，对应占 41.1% 和 5.1%。原因是 `--all` 让 `compact_actions` 跳过全部省略分支（`ui_view.py:120-123`，`all_nodes=True`）。第 3 条裁决的 A1 只管默认输出，所以这是按裁决范围做的，不算违约。但真实任务里调用者走到 `--all`，恰恰是因为产品自己的提示把他们引到那里。

**成本【估算】。**
- 第 1 次：第 7 回合读文件后，每回合输入增加约 11,860 token，此后还有 12 个回合，约 14 万 token，占该次 677,133 的约 21%。
- 第 5 次：三次读文件分别增加约 11.8K、14.5K、5.9K token，此后各留存 5、4、3 个回合，约 13.5 万 token，占 519,095 的约 26%。
- 去掉这部分，两次分别约 535K 和 384K。

**最小修正。**
- 省略行改成 `read="snapshot --cached eNNN"`：B3 修好之后，按引用读屏外子树是可用的。
- 超限摘要改成"用 `find TEXT` 定位，再 `snapshot --cached REF`"。
- `--all` 不再重新打开标准动作编号，这些编号留给 `--json` 和 `--raw`；这一项改动了 A1 的范围，需要用户点头。
- 前两项只改两处字符串，代价极小。

**分类。** 前两项是可选建议，去向**进下一轮**；如果整理提交前顺手改，风险很低。第三项是设计取舍，进下一轮，由用户定。

### 2.5 长列表题为什么每次都看图

模型能从树里读到的事实：
- `scroll-until` 后目标是 `rowIndex=15`（共 `rows=16`），而列表仍是 `scroll="forward backward up down"`；
- 再滚一次后列表变成 `scroll="backward up"`。

一个细心的调用者可以推出"已到底，最后一行应当完整可见"。但树里没有"这个节点被裁切或被遮住"的直接事实。`covered` 只检查中心点是否被同窗口的可点节点覆盖（`UiSnapshot.kt:188-214`），对导航栏这类跨窗口遮挡、对下沿被裁切都不报。截图证实了第一次滚动后卡片确实被导航栏和悬浮按钮压住（`r6-s1-core-candidate-01/workspace/.androidctl/shots/000003-…jpg`），第二张图则完整（`000005-…jpg`）。

所以 5/5 次看图的原因有两层：题目明说"使完整目标卡可见"，模型选择用视觉核实，这是合理的；产品可以给却没给"部分可见"的提示。上一轮同样 5/5 次截图（上一轮报告 3.5 节），本轮没有变化。

**可选改法。** 节点边界贴着容器可见边缘、而容器在该方向仍能滚动时，印一个短标记（例如 `edge`）。这是启发式判断，有误报风险，代价中等，**不建议在这一批做**。

**分类。** 可选建议，去向**进下一轮**或不必处理。

### 2.6 "advertised 时用 reveal"，但默认输出从不印这个宣告

**源码。** `ui_view.py:55-68` 的 `DEFAULT_OMITTED_ACTIONS` 包含 16908342（`show-on-screen`），所以默认视图里任何节点都不会出现 `+show-on-screen`。本轮 10 次的默认输出里一次都没出现，只在 `--all` 输出里出现。【实测】

**指引。** 首页第 62 行写的是 "Use `reveal REF` when advertised, or scroll the container to expose a target"。调用者无从判断"是否 advertised"，长列表题第 4 次就是这样试错的。

**拒绝回执。** 原文如下：
```
reveal: not sent; This target does not advertise reveal; click it directly with androidctl click e668236894, or use androidctl snapshot to find a container to scroll; code=actionUnavailable; cached; dispatchStarted=false; timing=before-action
```
它首先建议"直接点"，可模型的目的是让卡片可见；它还建议用 `snapshot` 去找容器，而容器是已知的。

**对比。** 网页题第 1 次点击屏外目标时，拒绝回执给出 `reveal it with androidctl reveal e950416952, then retry`（`UiAdvice.kt:11-18`，只在确有宣告时才建议），模型一步恢复。这是上一轮附录建议落实得好的地方：上一轮同类拒绝之后模型去读了 11 KB 的 `actions.md`，本轮没有。

**最小修正（二选一）。**
- 指引改一句："节点带 `offscreen` 时先 `reveal`；没有宣告会被安全拒绝（未发出）。"
- 只在带 `offscreen` 的节点上印 `+reveal`。

两种做法代价都极小。

**分类。** 确定缺陷（指引与输出不一致），程度轻。去向：**进下一轮**。

### 2.7 打开另一个 App 后回执没有树，而且建议看系统窗口（长列表 4/5）

**原件。** 第 2 次第 10 回合：
```
click: done; change=unsettled at windows (page may still be transitioning); snapshotPath=…; snapshotProblem={"code":"scopeUnknown","message":"One complete task window is required; inspect a listed window: snapshot --window 16; snapshot --window 17"}
screen keyboard="hidden" unreadable
window id=16 displayId=0 type="system" layer=1 title="Navigation bar" content="notRead"
window id=17 displayId=0 type="system" content="notRead"
artifact kind="tree" state="unavailable" code="scopeUnknown" …
```

**分析。**
- 建议去 `snapshot --window 16`，可 16 号是导航栏，照做毫无用处。4 次里模型都没照做，直接发 `snapshot`，所以没有被误导。
- 源码 `UiPostAction.kt:47-56` 在 `scopeUnknown` 时会等下一次检查，但截止时间不变，新 App 的窗口在截止前没出现。
- 第 15 条裁决只要求 `open` 落地后带树；由点击引起的跨 App 落地不在其中。

**最小修正。**
- `scopeUnknown` 并且是导航类动作时，在请求预算内多等一次；
- 建议文字在只剩系统窗口时改成"App 窗口尚未出现，稍后 `snapshot`"。

**分类。** 可选建议，去向**进下一轮**。

---

## 3. 模型实际看到的输出：典型样本逐行评

### 3.1 网页题

**① `snapshot`**（第 2 次第 3 回合，7,087 字节，节选）：
```
snapshot: done; snapshotPath="ultragoal\\…\\000001-…-Chrome_Beta__Issues___python_cpython___GitHub.txt"   ← 首行只说 done 和一条 170 字节的路径
androidctl capture="347498ef-…:9"                                       ← 溯源，按裁决保留
screen app="com.chrome.beta" title="Chrome Beta: Issues · python/cpython · GitHub" keyboard="hidden"   ← 决定性的一行
window id=21 active focused content="full" selected                     ← 很少用到
edge-windows count=2 content="notRead" windows=[{"id":13},{"id":16}]    ← 10 次里从没用过
scroll e499760689 focused scroll="forward down page-down"               ← 后来 scroll-until 用到它
 item e499760692 "Skip to content" clickable roleDescription="link" targetUrl="…#start-of-content"
 …
      textbox e499760759 "Search issues" value="is:issue state:open " clickable action-name="copy" action-name="paste" action-name="cut" +expand +fill +ime chromeRole="textFieldWithComboBox"   ← 决定性的引用
 …
       item e499761297 "Closed  (72,363)" clickable … targetUrl="…state%3Aclosed"
       text "Closed"
       text " ("
       text "72,363"
       text ")"                                                         ← 一个链接拆成 4 个文字子节点，又和父节点的名字重复
```

**评价。**
- 默认视图里已经没有标准动作编号（上一轮同位置每个节点挂 3–5 个 `action-name="action-40"…`）。首屏从上一轮的 11,186 字节降到 6,477–7,087 字节。瘦身见效。
- 剩下的冗余主要在网页文字被拆成子节点、又在父节点名字里重复一遍，例如 `"Closed  (72,363)"` 下面挂 4 个 `text`。这是忠实呈现无障碍树的代价，没有删过头。
- 首行不含页面标识。要看是哪一页，得读第 3 行的 `screen`。对网页题这正是判断"点击是否真的跳转了"的那一行，建议把 `title` 提到首行，或者在动作回执的首行里带上 `title changed`/`same title`。

**② 动作回执**（第 1 次第 12 回合，即 2.2 的样本）：
```
click: done; change=changed at content,e950416952:focused,actions; snapshotPath=…
androidctl capture="3f02b61c-…:16"
screen app="com.chrome.beta" title="Chrome Beta: Issues · python/cpython · GitHub" keyboard="hidden"
```

**评价。**
- 首行的 `changed` 是错误的暗示，第 3 行的标题才说了实话。5 次里模型都靠第 3 行识破了，但每次都要多一次调用。
- 和上一轮相比，`baselineAgeMs=`、`connection=… sequence=… finishedMs=…`、`observation {…}` 都已删去（第 15 条裁决），首行和头部明显变短。

**③ `scroll-until`**（第 2 次第 10 回合，9,457 字节，节选）：
```
scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":3,"unknown":0,"refs":["e499761857","e499761864","e499761865"],"field":"text","matchedText":"gvanrossum commented on May 24, 2023","windowId":21}; stop=found; snapshotPath=…
Requested: {"target":{"ref":"e499760689"},"action":"scroll-until","direction":"up","maxSteps":15}
Resolved: {"method":"original","target":{"ref":"e499760689","role":"scroll","name":"","windowId":21,"bounds":[0,160,720,1184]}}
Selected: {"target":{"ref":"e499760689","role":"scroll",…,"bounds":[0,160,720,1184]},"action":"scroll-up"}
…
     item chromeRole="listItem" rowIndex=16 columnIndex=0
      text "gvanrossum commented on May 24, 2023" roleDescription="heading 3" heading
      …
      item e499761864 "on May 24, 2023" clickable roleDescription="link" targetUrl="https://github.com/python/cpython/issues/86296#issuecomment-1560279122"
       text "May 24, 2023, 12:10 AM GMT"
      text "Member"
      item e499761868 "And fixed in 3.12 by " clickable detail=". (We're not 100.00% sure that the fix doesn't disturb some workaround, so we're not backporting the fix to 3.11 …"
```

**评价。**
- 这是本轮最好的一份输出，一条命令就交齐了作者、日期、永久链接和正文。
- 上一轮同一步的首行带 `Matching visible text observed…` 和 `change=unsettled … (page may still be transitioning)`；本轮改成了 `changed`，没有误导性的"还在变化"。这一项（B2 的答复部分）落实得好。
- 噪声：`Requested`、`Resolved`、`Selected` 三行 JSON 约 480 字节，从没被用过；`match` 里的引用指向不带引用号印在树里的叶子文字节点（`e499761857` 在正文里找不到），调用者得自己对应到可操作的父节点。

**④ `find --all`**（第 3 次第 8 回合，8,591 字节）。去掉动作字段和 `depth=` 后只剩约 4.7 KB【主机端用脚本重算】。原件节选：
```
find: done; cached; matches=4; offscreenMatches=4; …
window id=13 displayId=0 type="system" layer=2 title="Navigation bar" content="notRead"   ← --all 带出的窗口列表
text e361703052 action-name="select" action-name="clear-selection" action-name="action-40" +show-on-screen
 text e361703053 action-name="select" action-name="clear-selection" action-name="action-40" +show-on-screen
 …（9 层包装节点，每层都挂动作）
                text e361704354 "gvanrossum commented on May 24, 2023" depth=19 action-name="action-40" action-name="action-100" action-name="action-200" action-name="action-400" action-name="action-800" +show-on-screen action-name="context-click" roleDescription="heading 3" … offscreen
```
**评价。** 首页把 `--all` 只说成"超过默认 20 条"，`find --help` 才写了"Include hidden nodes and extra fields"。两次使用时都只有 4 条命中，`--all` 的唯一效果是把输出放大一倍。

### 3.2 长列表题

**① `snapshot`**（第 1 次第 3 回合，4,196 字节）：
```
snapshot: done; snapshotPath=…
androidctl capture="3f579196-…:3"
screen app="com.nononsenseapps.feeder" title="Feeder" keyboard="hidden"
window id=21 active focused content="full" selected
edge-windows count=2 content="notRead" windows=[{"id":14},{"id":18}]
list e638907335 scroll="forward down" rows=16 columns=1                  ← 决定性：容器引用、方向、总行数
 item e638907336 "RFC 10090: BGP Usage for … WAN port" clickable +long-click action-label="Toggle read status" action-label="Open in reader" action-label="Open in custom tab" action-label="Open in default browser" action-label="Save article" action-label="Mark the above as read" action-label="Mark the below as read" action-label="Share" detail="R4 RFC list | Nov 23, 2026 | Unread" rowIndex=0 columnIndex=1
 …（每行同样 8 个 action-label）
text "Refreshing"
item e638907376 "Open navigation drawer" clickable
```

**评价。**
- 决定下一步的只有 `list` 那一行。每个列表项都重复印同样 8 个 `action-label`，占长列表题全部命令输出的 18.8%（上一轮 16.2%，见 `noise2.py`），10 次里一次都没用到。
- `detail="… | Unread"` 是判断"相邻项保持未读"的关键事实，印得清楚。
- 可选改法：同一容器里相邻项的动作标签完全相同时，从第二项起只写"同上"。

**② `scroll-until` 回执**（第 1 次第 4 回合，5,460 字节）：
```
scroll-until: done; change=changed at content; match={"condition":"scroll-until","complete":true,"matches":1,"unknown":0,"refs":["e638907461"],"field":"text","matchedText":"RFC 10080: Updated Use of the Expires Message Header Field\nThis document allows broader use of the \"Expires\" … to handle","windowId":21}; stop=found; snapshotPath=…
Requested: {…"maxSteps":10}
Resolved: {…"bounds":[0,176,720,1280]}}
Selected: {…"bounds":[0,48,720,1280]},"action":"scroll-down"}            ← 两行边界不一致，没人用
…
list e638907335 scroll="forward backward up down" rows=16 columns=1
 …
 item e638907458 "RFC 10080: Updated Use of the Expires Message Header Field\n…" … detail="R4 RFC list | Oct 30, 2026 | Unread" rowIndex=15 columnIndex=1
```

**评价。**
- 7/7 次 `scroll-until`（两题合计）第一次调用就成功，`--timeout` 都带了单位，用法一次到位。
- 首行从上一轮的 `change=unsettled … (page may still be transitioning; content unknown: Node observation incomplete)` 改成了 `changed`，误导去除。
- `matchedText` 把 260 字符的整段名字又印一遍，正文里还会再印一次。
- 匹配到的引用是叶子文字节点 `e638907461`，正文里只印了可点的父节点 `e638907458`。模型 5/5 次都正确点了父节点，没有出错，但这个对应关系要靠调用者自己完成。可以在 `match` 里同时给出最近的可操作祖先。

**③ 动作回执：返回键**（第 3 次第 14 回合，见 2.3 原件）。首行 `back: done; change=unsettled (page may still be transitioning)` 本身属实，但 16/16 条 `back` 都是这一句，信息量为零；下面的正文还可能是返回前的页面。上一轮 `back` 首行带的 `keys={…}` 过程计数和 `Platform key dispatch returned; App outcome is not verified` 已删去（A4 落实），这一点是改好了的。

### 3.3 和上一轮比：瘦身效果

| 项目 | 上一轮 网页 | 本轮 网页 | 上一轮 长列表 | 本轮 长列表 |
|---|---:|---:|---:|---:|
| 5 次命令输出合计 | 661,256 B | 313,903 B | 241,907 B | 196,127 B |
| 标准动作编号 | 33.5% | 1.9%（全部来自 `find --all`） | 7.2% | 0 |
| `observation {…}` 行 | 1.2% | 0.1% | 2.8% | 0.0% |
| 头部 `capture` 行 | 1.2% | 1.1% | 2.6% | 1.7% |
| `keys=` 过程计数 | 0.3% | 0 | 1.0% | 0 |
| 保存路径（首行） | 2.3% | 4.1% | 5.6% | 7.2% |
| Feeder 的 8 个动作标签 | — | — | 16.2% | 18.8% |

（`noise2.py`，只统计命令输出，不含读文件。保存路径的占比上升，是因为别的内容少了，路径本身没变长。）

**结论。**
- A1、A4 和第 15 条落实得好，默认输出没有删过头：10 次里没有一次因为默认输出缺了什么而去用 `--json` 或 `--raw`。
- 剩下的可删项依次是：`--all` 下的动作字段、长保存路径、Feeder 重复的动作标签、`Requested`/`Resolved`/`Selected` 三行、`edge-windows` 行。

---

## 4. 指引的实际使用

| 文件 | 网页题 | 长列表题 | 用途 |
|---|---|---|---|
| `SKILL.md`（7,036 字节） | 5/5 | 5/5 | 每次都读全文，第 2–3 回合就开始操作设备 |
| `actions.md`（11,137 字节） | 0/5 | 4/5 | 都在截图后读，想弄清"怎样让整卡可见"。读完之后的动作（`scroll down`）和没读的第 4 次一样，**没有带来行为差异** |
| 其它专题、`--help` | 0 | 0 | |

**首页里 10 次都用到的部分。**
- 常用循环里的 `snapshot`、`fill --submit`、`find`、`click`、`back`、`scroll`；
- `scroll-until` 示例：参数原样照抄，7/7 次成功；
- "未稳定的动作之后 `find` 会新采"：网页题 5/5 次 `fill` 之后接 `find`，首行都是 `fresh`；
- `snapshot --cached REF`：网页 2/5 次；
- 超过 64 KiB 读保存文件：网页 2/5 次；
- `snapshot --screenshot`、`--marks` 和看图：长列表 5/5 次。

**首页里 10 次都没用到的段落。** "Start A Task"（配对、`session open`、`busy`、后台屏，约 1.2 KB）；`apps`、`open`、`wait --contains`；`key`、`ime`、`check` 那一段；`--no-print`、`--json`、`--raw`；UNKNOWN 一节（约 0.5 KB）；Topics 列表除 `actions` 以外的 9 条；Finish 一节（题目规定不关会话）。合计约 3 KB，占首页的 40%。这两道题本来就不需要这些内容，它们是给后台屏题、写操作和结果未知时用的，**不建议为这两题删减**。按实施方成本页的估算，首页约占网页题输入的 4.8%。

**因为指引没讲而试错的地方。**
- 点击带 `offscreen` 标记的目标（网页第 1 次）：首页没说屏外目标要先 reveal，产品的拒绝回执补上了这一句。
- 试 `reveal`（长列表第 4 次）：首页说"advertised 时用"，但默认输出不印宣告，见 2.6。
- `find --all`（网页第 1、3 次）：首页对 `--all` 的描述不全。

**照着指引或产品提示做却失败或变贵的地方。**
- 网页第 1 次照 `find` 省略行的 `read="snapshot --cached --all"` 去做，走进超限摘要，再读 75 KB 被截断的文件。
- 网页第 5 次照首页"Read/search that file after host-tool truncation"去读和搜保存文件，连续两次被执行框架截断，第三次靠自己写的过滤才读到（2.4）。

**写得好的地方。**
- 首页的 `scroll-until` 示例让参数一次写对；"Choose directions from its `scroll` fact"让方向词 0 次出错（上一轮还有 4 次方向被拒）。
- "Reuse a sufficient observation"一段让 `fill` 之后的 `find` 自然地新采，模型 5/5 次没有在 `loading` 回执上多补 `snapshot`。

---

## 5. 和上一轮同题轨迹的行为差异

| | 上一轮 网页 | 本轮 网页 | 上一轮 长列表 | 本轮 长列表 |
|---|---|---|---|---|
| 回合，逐次 | 15、14、17、15、16 | 19、13、12、13、14 | 13、14、14、13、17 | 14、14、16、14、15 |
| 回合中位数 | 15 | **13** | 14 | 14 |
| 命令条数中位数 | 12 | **10** | 12 | 12 |
| 总输入中位数 | 557,402 | **387,934** | 393,353 | 395,456 |
| 非零退出 | 3 | 1 | 0 | 1 |
| 看图次数 | 0 | 0 | 8 | 9 |

（`stats.py`，`stats.txt`。）

**网页题典型路径。**
- 上一轮：`snapshot` → `fill` → `find` → `click` → `snapshot` → `find "May" --all` → `snapshot --cached REF`（空）→ `snapshot --cached --all REF` → `snapshot --cached --all 父REF` → `back`（→ `snapshot`）。
- 本轮：`snapshot` → `fill` → `find` → `click` → `snapshot` → 两种走法之一：
  - `find "May"` → `snapshot --cached REF` →（`snapshot --cached 父REF`）；
  - "Jump to bottom" → `snapshot` → `scroll-until up "May"`。

  然后 `back`（→ `snapshot`）。

**消失的步骤。**
- 空的按引用读取，以及随后补的 `--all REF`：上一轮 4/5 次，本轮 0 次，这是 B3 的效果。
- 屏外点击被拒后读 `actions.md`：上一轮 1 次，本轮 0 次，靠的是拒绝回执里给的建议。
- `wait` 的难懂失败：上一轮 1 次，本轮 0 次。

**新增的步骤。**
- 超限摘要加读文件（2/5），由第 14 条的输出上限加省略行提示引出。上一轮同样情形是直接打印 75 KB 再被截断，调用次数大致持平；第 5 次因为摘要里的建议没用，多花了 2 次读文件。
- "Jump to bottom"加 `scroll-until up`（2/5）：上一轮第 4 次也用过，那次检索词写错失败了；本轮 2/2 一次成功。

**新增的拒绝类型。** `actionUnavailable`（长列表第 4 次的 `reveal`）。`notReady` 是老类型，但本轮回执带上了可照抄的 `reveal` 命令。

**长列表题。** 命令序列与上一轮几乎逐条相同：`snapshot`、`scroll-until`、截图看图、`scroll`、截图看图、点卡片、菜单、"Open in Web view"、`snapshot`、`back`、`back` 或 Go back、`snapshot`。本轮的改动（A1、B2 的 `scroll-until` 部分、B3）都不在这条路径的瓶颈上，所以回合和输入持平。这与实施方成本页的比值 0.96 一致。

---

## 6. 业务完成度

**网页题 5/5 正确。**
- 内容：issue 编号 #86296，状态 Closed（Completed），作者 gvanrossum，2023-05-24 00:10 GMT，永久链接 `#issuecomment-1560279122`；3.12 由 gh-96764 修复，不回移 3.11，3.10 需要另做完全不同的修复。我对照了原件里的评论正文（例如第 3 次第 10 回合的 `detail=". (We're not 100.00% sure … not backporting the fix to 3.11 … totally different fix for 3.10 …"`）。【实测】
- 第 2 次是在地址栏填入同一查询网址回到筛选结果，而不是用 `back`。查询串相同，结果页相同，可以算"同一筛选结果"。
- 第 3 次只凭一条 `unsettled` 的 `back` 回执就判定已经返回，没有再核对。回执里的标题和查询串已经是对的，实施方的独立核验也确认了，但这在指引上属于略冒险的做法。

**长列表题 5/5 正确。**
- 内容：标题；官方链接 `https://www.rfc-editor.org/info/rfc10080/`（4 次取自 WebView 地址栏，1 次取自分享面板预览，没有发送）；`Expires` 邮件头；发送者可标明何时过期；接收者可据此区别处理。回到同一列表，目标为 `Read`，相邻项为 `Unread`，取自树里的 `detail`。
- "使整卡可见"这一项 5/5 次都靠截图看图确认。树给不出这一事实，这是**靠截图补产品缺口**完成的（2.5）。不过题目本来就允许使用图像。

**没有任何虚报或靠运气得到的结果。** 最终回答都写了"无待确认信息"，与原件相符。

---

## 7. 最短合理路径、差距与改进清单

### 7.1 熟练调用者的最短路径

**网页题（按今天的产品）：**
1. 线程号和首页，同一回合；
2. `snapshot`；
3. `fill 搜索框 "is:issue state:closed AsyncIO wait_for hide cancellation" --submit`；
4. `find "rare race"`；
5. `click 链接`；
6. `snapshot`（今天必须有，因为回执判 `changed` 但仍是旧页）；
7. `find "gvanrossum commented"` 或 `find "May"`；
8. `snapshot --cached 列表项REF`；
9. `back`。

一共 **8 条命令、10 个回合**（含最终回答）。如果 B2 补全，第 6 步可以省掉，就是 7 条命令、9 个回合。
- 本轮中位数是 10 条命令、13 个回合，差 2 条命令、3 个回合。
- 最好的一次（第 3 次）是 9 条命令、12 个回合。

**长列表题：**
1. 线程号和首页；
2. `snapshot`；
3. `scroll-until … --contains "RFC 10080"`；
4. `scroll … down --screenshot`；
5. 看图（题目要求视觉确认时）；
6. 点卡片；
7. 点 "Open in Web view"；
8. `snapshot`；
9. `back`；
10. 点阅读页的 "Go back"（回执带列表，不必再核对）。

一共 **8 条命令加 1 次看图、11 个回合**；不看图的话 10 个回合。
- 本轮中位数是 12 条命令加 2 次看图、14 个回合，差 3 个回合。

### 7.2 差距由哪几类摩擦构成（按中位数运行折算）

**网页题（差 3 个回合）：**
- `back` 后的核对：约 0.6；
- 读文件或 `--all` 弯路：中位数运行没有，但贡献了两次最大的离群；
- 第二次按引用读取：0.4；
- 屏外点击被拒：0.2；
- 锚点路线：0.4；
- 地址栏返回：模型选择；
- 另外，按今天的产品，点击后补 `snapshot` 那 1 次已经算进最短路径里。

**长列表题（差 3 个回合）：**
- 菜单探索：1，页面本身；
- 第二次截图看图：约 1，产品缺事实加模型谨慎；
- `back` 取样过早后的核对：0.6，产品；
- "Open in Web view"之后没有树（已计入最短路径的第 8 步，如果产品修好可再省 1）；
- `reveal` 试错：0.2。

### 7.3 按"修一处能省多少调用"排序的改进清单

| # | 改什么 | 改哪里 | 预计收益 | 代价和风险 | 建议去向 |
|---|---|---|---|---|---|
| 1 | B2 补全：导航类动作窗口和标题都没变、变化只在派发目标自身的焦点、选中、动作或边界上时，判"可能还在变化"并在预算内续采到标题变化 | 产品（`UiPostAction.kt:245-281`） | 网页题 5/5 次各省 1 条命令和 1 个回合（约 −8% 回合）；同时消除 `changed` 之后 `find` 读旧页缓存的隐患 | 小：一个判定加一条测试；可能让个别点击多等一个取样周期 | 进下一轮（第一批） |
| 2 | 系统按键类动作以会话最近的完整样本作基准；与基准相同不算稳定 | 产品（`SystemActions.kt:56`、`UiPostAction.kt:63-65,216`） | 10 次里约 6 次各省 1 条（长列表 3/5、网页 2–3/5） | 中：取样时机，要防止原地不动的 `back` 白白等满预算 | 进下一轮 |
| 3 | 省略行提示改成 `snapshot --cached eNNN`；超限摘要改成"`find TEXT`，再 `snapshot --cached REF`" | 输出（`ui_view.py:898`、`ui_client.py:888`） | 走到超限的运行各省 2–4 条命令、约 13–14 万输入 token（本轮 2/5 次网页题，约占各自 21–26%）【估算】 | 极小：两处字符串 | 进下一轮；整理提交前顺手改也可以 |
| 4 | `--all` 不再重新打开标准动作编号（留给 `--json` 和 `--raw`） | 产品或输出（`ui_view.py:120-123`） | `--all` 文件缩小 45–55%，读文件时少一次截断；`find --all` 每次少约 4 KB | 小；但改了 A1 的范围，需要用户确认 | 进下一轮，由用户定 |
| 5 | 点击引起跨 App 落地、只剩系统窗口时多等一次；建议文字不再指向系统窗口 | 产品＋输出（`UiPostAction.kt:47-56`、`UiScope.kt:96`） | 长列表 4/5 次各省 1 条 | 小到中 | 进下一轮 |
| 6 | 首页写明"带 `offscreen` 的目标先 `reveal`；没有宣告会被安全拒绝"，或只在屏外节点上印 `+reveal` | 指引（`SKILL.md:62`）或输出 | 约 0.2 条/次（本轮 2/10 次各 1 条） | 极小 | 进下一轮 |
| 7 | 首页把 `--all` 的副作用说全（"包括隐藏节点和全部字段；命中不到 20 条时不要加"） | 指引（`SKILL.md:61`） | 0 条命令；网页 2/5 次各少约 4 KB，并减少走进第 3 条那条贵路线的机会 | 极小 | 进下一轮 |
| 8 | 节点贴着容器可见边缘、容器在该方向仍可滚时印短标记；同容器相邻项的动作标签重复时省略 | 输出 | 长列表题可能省下截图加看图约 1–2 个回合（题目若仍要求视觉确认则省不了）；输出少约 15% | 中：启发式有误报风险，会改默认文本契约 | 可选，不建议在这一批做 |

---

## 8. 发现与分类汇总

| 编号 | 发现 | 依据 | 性质 | 去向 |
|---|---|---|---|---|
| F1 | B2 的实现没有覆盖它要修的真实事件：点击导航后的 `focused,actions` 变化被判 `changed`，网页 5/5 次多一次 `snapshot`；"B2 已闭"与实测不符 | 实测＋源码 | 确定缺陷 | 进下一轮（主报告应更正"已闭"的说法） |
| F2 | 系统按键类动作没有动作前样本，两次相同就当稳定，回执可能是返回前的页面；`back` 回执 16/16 都是 `unsettled` | 实测＋源码 | 设计取舍（可修） | 进下一轮 |
| F3 | 省略行和超限摘要的提示把调用者引向 `--all` 和读文件，`--depth 2` 对网页没用；造成两次成本离群 | 实测＋源码＋估算 | 确定缺陷（提示文字） | 进下一轮（可顺手改） |
| F4 | `--all` 重新打开标准动作编号，保存文件里 41–50% 是噪声，模型只好自己写过滤 | 实测＋源码 | 设计取舍 | 进下一轮，由用户定 |
| F5 | 首页"advertised 时用 reveal"，默认输出却从不印 `show-on-screen` | 实测＋源码 | 确定缺陷（指引与输出不一致） | 进下一轮 |
| F6 | 点击引起跨 App 落地时回执没有树，建议看系统窗口 | 实测＋源码 | 可选建议 | 进下一轮 |
| F7 | 首页对 `find --all` 的描述不全 | 实测＋`--help` 对照 | 可选建议 | 进下一轮 |
| F8 | 树里没有"部分可见"事实，长列表题 5/5 次靠截图确认 | 实测 | 可选建议 | 进下一轮或不必处理 |
| F9 | Feeder 每行重复 8 个动作标签（占长列表输出 19%）；`Requested`/`Resolved`/`Selected` 三行和长保存路径 | 实测 | 可选建议 | 不必处理或进下一轮 |
| F10 | `scroll-until` 的 `match.refs` 指向正文里没有引用号的叶子节点 | 实测 | 可选建议 | 不必处理 |

**挡合入：无。** 理由：以上各项都只多花调用或输入，不造成误操作、重复执行或资源损失；F1 虽然是"必须做"清单里的一项，但裁决把判定交给了实施方，而实现覆盖了裁决字面上举的"只有边界变化"这一情形，所以不构成"与裁决明确相反"。

---

## 9. 已核实无问题（支持实施方的地方）

1. **A1 生效。** 10 次的默认输出里没有一个标准动作编号；网页题命令输出减半。【实测】
2. **B3 生效。** `snapshot --cached REF` 读屏外节点会返回子树（网页第 1、3 次各 1,586 和 2,632 字节，都带正文）；上一轮 4/5 次的"先读空、再加 `--all`"本轮 0 次。【实测】
3. **B2 的答复部分生效。** `scroll-until` 成功时不再带"可能还在变化"，7/7 条首行都是 `changed … stop=found`。【实测】
4. **屏外点击的拒绝回执**给出可照抄的 `reveal` 命令，模型一步恢复，不再去读专题。【实测】
5. **第 11 条（`find` 读缓存、0 条命中再新采）。** 首行的 `fresh` 和 `cached` 标得清楚；`fill --submit` 之后的 `find` 5/5 次都是新采。【实测】
6. **A4 落实。** `back`、`fill` 等成功回执里没有过程计数。【实测】
7. **入口零陷阱。** 时长单位、引用写法、方向词、`--help` 都是 0 次出错；方向词在上一轮还有 4 次被拒。【实测】
8. **退出码。** 两次"未发出"的子进程退出码都是 2，与回执一致。长列表第 4 次模型看到的是 1，原因在模型自己的 PowerShell 包装没有传递退出码，不是产品问题。【实测】

## 10. 未能核实的事项

- **模型的推理过程。** 原生记录里的推理摘要是加密的，"为什么读 `actions.md`""为什么选 Jump to bottom"等只能按下一条命令推断。
- **修正后的实际节省。** F1、F2、F5、F6 修正后能省多少，是按本轮原件推算的，没有设备可以复测。F2 修正后会不会让原地不动的 `back` 多等，属于未知。
- **长列表题截图前后的节点边界。** 保存的观察里没有原始边界，所以 F8 的"贴边"判据在这道题上是否成立，没有用数据验证过。
- **任务提示原文。** 模型收到的提示是加密的；我用的是第六轮包里冻结的题面，没有逐字核对本轮实际发出的提示。
- **成本估算的口径。** 第 2.4 节的 token 估算按回合输入增量乘以留存回合数计算，没有扣除缓存折扣，也不是精确分项。
