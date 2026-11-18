# 专题 18：输入通道复审（IC、借贴、结构剪贴板、坐标接收者、后台屏文本、typed Intent、D6/D9）

评审者 18。只读。路径约定：`T` = `NEW/ultragoal/2026-11-11-next-improve`；源码行号指 `review-copy`（3f45af6）的 `android/app/src/main/java/com/rainng/androidctl/`（下称 `A/`）。自写脚本与下载件在 `agents4/18/`。

## 1. 结论摘要

1. **IC**：实施方纠正我方的两点成立：E13 已真实跑过 Termux 的“剪贴板 + Ctrl+Alt+V”中文粘贴，WPS 正文与 Contacts 也有 IC 正例。我方说的“端到端从未跑过”和“IC 唯一正例是 Termux”都是错的。但 E13 原报告自己写的是“A：不纳 IC——合理替代”，所以“必须保留”这一结论不是证据推出来的。我的判断：**保留 `commit`**。它在主屏上等同于“像键盘一样打字”，是最通用的 Unicode 插入方式，不碰剪贴板；它依附的无障碍服务本来就要为 Chrome 保留，额外维护量适中。**独立 `select` 降级**：建议改成按 REF 的 AX `select REF START END`，主屏和后台屏都能用。IC select 目前只有 Contacts 一个正例，而 Contacts 本身也支持 AX 选区。
2. **借贴**：实施方说“Termux/WPS 还需快捷键或图菜单，所以两命令替代不等价”，这个论据**不成立**。借贴自己也只走 AX PASTE（`UiActions.kt:257-262`），同样服务不了 Termux 和 WPS。借贴真正独有的只有两点：恢复原剪贴板（仅限文本/HTML，原剪贴板含 URI/Intent 时整条拒绝，见 `BorrowedPaste.kt:197`），以及一次请求内完成插入核值。这两点在**后台屏**（真人同时在用手机）价值最大。所以我更正我方“删除”的结论，改为**保留，但必须同时加 D6 浮层抑制、精简回执**。可以再考虑让粘贴触发方式支持按键组合，把 Termux 这类 App 也纳进来。
3. **结构剪贴板准入**：P8 确实在真机上人为设置并实测了“读被拒、写被允许”，我方“包内没有实际发生过”的说法错了。但自然状态下多常见仍是推断。调用者需要据以改变下一步的只有三种状态：有内容 / 确认为空 / 读不到（附原因）。采样时戳、readMode、backgroundReadGranted 等字段只属于内部。
4. **坐标接收者**：已用原件确认默认 tap 点软键盘会被拒。我提出比 C 更简单的 **E**：用**已采样的无障碍窗口目录**做命中判断，取包含该点的最上层窗口，用其根包换算 PM UID 后定向。依据有三：A14 的无障碍窗口本身就由 `WindowInfosListener` 填充（`AccessibilityWindowsPopulator extends WindowInfosListener`，第 55 行）；无障碍窗口的 bounds 就是可触摸区域（A17 `AccessibilityWindowInfo.java:416-417`）；离线重算显示 IME 窗口 box 恰好等于 SF 的 touchableRegion，A 键 (72,902) 命中的最上层窗口就是 inputMethod。E 不需要隐藏类、回调生命周期或坐标变换，能拿到 C 的大部分收益，只缺嵌入式 SCVH 的 owner（这类情况会诚实地报 not sent，调用者改用 `--expect`）。**推荐 E；C 只在用户确实需要默认点中嵌入式跨 UID 接收面时再考虑。**
5. **D6**：实施方前后不一致。`inputs.md` §3.3 建议纳入，整体建议却写“暂不加”。我建议加（源码遗漏已确认）。**D9**：同意保留派发前 pending。但首个 DOWN 已明确 mismatch、结果为 not sent 时仍跑后采样（实测 2.016 秒、4 次采样），应当跳过。**typed Intent**：实施方正确，我方 §8.6 的“改走平台迁移”应撤回。
6. 最大未知：E 和 C 在真实 OEM 与嵌入面上的预测误拒率；借贴恢复的实际价值取决于后台屏模式是否成为主用法。

## 2. 独立判断

### 2.1 IC（当前 editor commit + 独立 UTF-16 选区）

**(a) Termux 的剪贴板 + 快捷键路径**（原件核对）

- E13 研究探针（不是正式 bridge），MuMu，Termux 0.118.3：
  - `commands/e13-termux-clipboard/stdout.bin`：ready（原剪贴板 present=false）→ borrow `ICF11-TX-P-中文`，回读确认 → restore 后与原空状态匹配。
  - `e13-termux-ctrl-alt-v/stdout.bin`：六个事件全部 `returned:true`，**`targetUid:10087`（定向到 Termux 的 UID，不是全局键）**。
  - 我亲自看了 `artifacts/e13-termux-paste.png`（SHA `32b439e1…` 已重算一致）：prompt 上恰好是 `ICF11-TX-P-中文`，并有“已将剪贴板中的内容粘贴到Termux”的提示，没有 Enter。
  - 两个包装进程分别以 135 和 139 退出（Bus error / Segmentation fault），但事件回执和图像独立成立。
- P8（`T/stages/35-clipboard/p8-structured/runtime-api34/result.md`）：正式候选的**全局** Ctrl+Alt+V，6/6 事件，主结果 done，内容为 ASCII 的 B，独立图确认被消费。
- 结论：中文经剪贴板 + App 快捷键插入 Termux 已有真实证据。成立条件：Termux 有焦点；硬件快捷键未被禁用；需事先知道这个 App 的专用快捷键；剪贴板会被改写（E13 用的是借写 + 恢复）。中文这次走的是定向 UID 的探针，正式全局键只测过 ASCII。但两者的差别只在键序列（源码 `KeyPlan.kt:65-85`，与文本内容无关），可以合理外推，标为**推断**。

**(b) WPS 和 Contacts 上 IC 做成了什么**（原件核对 + 报告核对）

- WPS 26.10.2 正文：`e13-wps-commit-insert/stdout.bin` 显示焦点节点 `text_editor`，`editable=false`、`text=null`，广告的动作只有 2/4/8/16/32/64/16908342，**没有 SET_TEXT(2097152)、PASTE(32768)、SET_SELECTION(131072)**。IC commit 后，`e13-wps-insert.png`（已亲看）显示 `ICF11-WP-I-中文`，“全文: 13”。
  - fill 路线：`e13-wps-fill-control` 未派发（5.1 秒后 not sent）。
  - 剪贴板路线：在另一个空文档上，借写 → 长按 → 看图 → 点图上的“粘贴”菜单（菜单不在 AX 树里）→ 成功。比 IC 多了长按、截图、点菜单至少三步，而且改写了剪贴板。
  - IC select(9,10)：调用返回，但选区仍是 13/13，**被忽略**。
- Contacts 4.90：I 臂（caret 9 处插入）和 S 臂（替换 9..12）都成功，保住了前后缀。替代路径：AX fill 能整值写入，但 Google Contacts 的 AX text 带“, 姓名”标签，按“读出 AX 原值→拼接→整值写回”的方式不可信；而 IC 读到的 surrounding 是干净正文。Contacts 也广告 AX SET_SELECTION（E13 报告称，未做对照）。
- 结论：IC 实测做成的**独有或明显更便宜**的事：①在 AX 不暴露文本和动作的富文本或自绘编辑器（WPS 正文）里，一步插入 Unicode，不碰剪贴板，不需要看图点菜单；②在 AX text 不可信的字段（Google Contacts 标签）里做局部插入；③Termux 上不需要知道 App 专用快捷键。IC select 的独有价值**未证**：WPS 两条路都失败，Contacts 两条路都可能成功。

**(c) 能否推到当前候选**：E13 是研究探针（content provider/DUMP 入口），正式 bridge 的证据是 28 号阶段（Contacts、Termux、Openreads）。两者调用的是同一个平台 `AccessibilityInputConnection.commitText/setSelection`。当前候选 `BrowserAccessibilityService.kt:53,58` 直接调用这两个方法，`CurrentEditorInput.kt:93-149` 在调用前后各读一次核值。平台层能力可以外推；正式 bridge 在 WPS 上未跑，标为**推断**。

**独立判断**

- IC commit 本质上是“像 IME 一样打字”。凡是真人能用键盘输入的编辑器，都有 InputConnection。所以在主屏上，它是**覆盖面最广**的 Unicode 插入通道，不依赖 App 的 AX 实现，也不依赖 App 的粘贴入口。替代路径的真实成本：
  - 剪贴板 + App 专用入口：多 1–3 次调用；要知道该 App 的快捷键，或者截图、长按、读菜单坐标；会污染剪贴板，或依赖借贴恢复；在主屏会弹“已粘贴”提示和复制浮层。
  - fill：只能整值替换，而且依赖 AX 可编辑。
- 维护责任：无障碍服务 `BrowserAccessibilityService` 本来就要为 Chrome 保留（我方 R3 当时也说保留）。IC 增量是 `flagInputMethodEditor`、`onCreateInputMethod` 与 generation、`EditorBridge.kt`（96 行）、`CurrentEditorInput.kt`（171）、`EditorSurrounding.kt`（129）、AIDL 和 WorkerConnection 里的 bridge 接线，约 450 行加测试和文档。只能用于主屏；wrapper 或 generation 失效时会诚实停止。这个量适中，换来的是唯一不碰剪贴板的通用 Unicode 插入。
- 我对 commit 的判断：**保留**（设计取舍，把握中高）。
- 我对 select 的判断：独立能力有价值，例如先定位或选中再 `key ctrl+x`。但 IC 的实现只在主屏可用、与当前 editor 绑定、不带 REF。更好的形态是 **AX `select REF START END`（SET_SELECTION）**：按 REF 定向、后台屏可用、不需要服务，覆盖所有广告该动作的标准 EditText 和 WebView 字段。IC select 只对“有 IC、但不广告 AX SET_SELECTION、而且真会执行 setSelection”的编辑器有增量，目前没有一例证据（WPS 是反例）。建议：新增 AX select；IC select 交用户决定是否保留，我倾向删除（省掉 `EditorSurrounding` 的范围准入、select 三态规则和 `input.md` 约 15 行条文）。
- commit 和 select 是否合并：同意实施方的意见。在 WPS 已证 setSelection 会被忽略的前提下，“先选再提交”必须在两步之间核对选区，否则会把文字写到旧 caret。合并换不来事务，所以**不建议合并**。如果选择 AX select，就更无须合并。
- 入口命名（设计建议）：把 `editor commit TEXT` 改为 `type TEXT --via editor`。这样“插入式输入”统一在 `type` 这个动词下（keys/paste/editor），调用者少记一个命令族。

### 2.2 借用剪贴板粘贴

**源码核对**

- 借贴只服务带 REF、已有输入焦点、借写后**广告 AX PASTE**的节点：`UiActions.kt:207-243`（`paste`）、`:257-262`（`ready` 检查 PASTE 广告）、`:268`（`ACTION_PASTE`）。
- 原剪贴板含 URI 或 Intent 时，`BorrowedPaste.kt:197-199` 直接拒绝。也就是说，用户剪贴板里有图片时，整个通道不可用。
- 状态：`original`（持有活的 ClipData）、`borrowed`、`borrow/paste/restore` 三段 JSON。
- 清理路径：`execute()` 的 finally 在 `borrow.attempted` 后必调 `cleanup()`（`:97-103`）。`cleanup` 先读一次核所有权，读到 null 时依空依据判 skipped 或 unknown；如果当前剪贴板已不是借写内容，就 skip，保留用户新复制的内容（`:255-258`）；否则恢复或清空，并再读一次核对（`:269-292`）。
- 每次读到 null 都会调用 `emptyBasis`（一次 AppOps `noteOperation`，`ClipboardAccess.kt:127-168`）。
- 其他行为：pending 状态派发前就写入（`:134-137`）；worker 被强杀时没有恢复路径，文档已声明“无 crash-recovery”。

**失败模式**

- 所有权核对到恢复之间不是 CAS。真人在这几十到几百毫秒里复制的内容，可能被恢复动作覆盖（**推断**，窗口很小）。
- 强杀或 Binder 死亡时，借写的内容会留在剪贴板。
- 恢复不还原 owner、时间戳和历史。
- 主屏会出现两次复制浮层（借写一次；恢复非空原剪贴板时一次），以及读剪贴板时的“已粘贴”提示，后者无法抑制。

**价值判断**：调用者是替真人操作设备的 LLM agent。剪贴板是真人的私有工作状态，被悄悄覆盖算一种用户可感知的伤害，尤其在后台屏模式下（真人正在主屏上用手机，可能刚复制了东西准备粘贴到别处）。借贴把伤害窗口压缩到一次请求之内，而且在检测到真人新复制时会让路。这个价值是真实的，不需要以“包内出现过 Copy 冲突”为前提。我方上轮以“没有冲突证据”否定它，属于用单次证据决定去留，正是本轮简报要求避免的。

**更简单的同等做法？**

- “`clipboard set --restore-after`”做不成：恢复必须发生在粘贴之后，而粘贴是另一条命令。要么做成跨命令的恢复令牌（更复杂），要么像现在这样放在单条命令里。现形态（`type REF TEXT --via paste`）已经是“粘贴一段文本且尽量不破坏剪贴板”的最小单命令形态。
- 可以简化的地方：
  - 默认回执只给“插入核值 / 粘贴是否发出 / 剪贴板已恢复、已跳过或未知”三行，JSON 保留细节（P9 的 stdout 为 3130 字节，两段 emptyBasis 各约 12 个字段，见我方专题 03 D11）。
  - `failureState` 的细分可以保留。
- 可以扩展的地方（设计建议）：让粘贴触发方式可选 `--paste-by action|keys:CHORD`，复用同一套借写和恢复状态机，把 Termux（ctrl+alt+v）和接受 ctrl+v 的 App 纳进来，在后台屏也能用。效果核值在 AX text 不可读时为 UNKNOWN。这比让调用者手工组合 `clipboard set` + `key` 并接受剪贴板被覆盖更好。不过这是新增能力，交用户选择。
- **推荐**：保留借贴（更正我方），必须加 D6 抑制，精简回执；可选扩展按键触发。对 URI/图片原剪贴板保持拒绝，文档要明说“此时请用 `clipboard set` + PASTE，并接受覆盖”。

### 2.3 结构剪贴板准入

- 原件核对（P8）：`read-denied-batch` 中第一次 get 返回 null，readMode=1，`accepted=false`，结果 UNKNOWN，第二步零 RPC；`denied-explicit-set-B` 写调用返回但回读 null，结果 UNKNOWN，之后 Termux 实际粘贴出 B。这个状态是 `appops set … READ_CLIPBOARD ignore` **人为制造**的。null-state 报告也写明“固定源码支持的可达配置，不是本次已实测故障”。
- 自然状态下的可达性：A14 `ClipboardService` 在 shell 的后台读取权限分支之后仍会 `noteOp`（null-state 上游副本第 1318-1324 行），所以用户或 OEM 的 AppOps 配置可以只拒读。哪些 ROM 默认如此，**需设备**。
- 调用者会据以采取不同下一步的区分：
  1. get 结果是有内容 / 确认为空 / 读不到。读不到时附原因（锁屏、非 user0、AppOps 拒绝、无后台读权限），因为“请解锁”和“权限被拒”对应的下一步不同。
  2. set/clear 的结果是已核、写了但核不了（READ 被拒）、失败。后者应提示“写入可能已生效，用业务效果确认”。
  3. 借贴是否因原剪贴板状态未知而拒绝。
- 只属于内部的窄区分：`noteStartedMs/noteReturnedMs`、`clipboardDeviceId`、`callerUid`、`backgroundReadGranted` 原值、`returnedClip` 与 `callReturned` 分层（保留在 JSON 里即可，默认不显示）。
- emptyBasis 本身约 45 行（反射调用 `IAppOpsService.noteOperation`，7 个参数），14–17 版签名稳定（null-state 报告核对过）。它保护的是“读被拒时，借写把看不见的原剪贴板覆盖掉”。**保留**，代价小。缺点是每次读到 null 都会记一次 AppOps 访问，可以接受。

### 2.4 坐标输入的默认接收者

**原件核对（ime-api34）**：

- `default-one-visible-letter-tap/stdout.bin`：结果 not sent，原因“Gesture targetMismatch”，inputTarget 为 `com.android.contacts/10095/window`，attempted 1、returned 0。平台报错：`…not directed at a window owned by uid 10095`。
- `actual-app-ime-window-owners/stdout.bin`：InputMethod 窗口 `mOwnerUid=10109`，frame `[0,48,720,1280]`，touchableRegion `(0,658,720,1184)`，transform 的 Y 为 −48。
- 默认 tap 点软键盘被拒的缺口**已确认**。

**离线重算（E 的可行性）**：

- 脚本 `agents4/18/ax_hit.py`，输入是同一工作区 tap 命令自身采到的窗口目录 `workspace/.androidctl/snapshots/000006-…Create_new_contact.txt`。该目录有 4 个窗口：
  - id 25，system，layer 3，[0,1184,720,1280]
  - id 31，system，layer 2，[0,0,720,48]
  - **id 22，inputMethod，layer 1，[0,658,720,1184]**
  - id 34，application，layer 0，[0,0,720,1280]
- 按 layer 从高到低取第一个包含 (72,902) 的窗口，结果是 **22 inputMethod**；点 (300,400) 则得到 34 application。
- 无障碍窗口的 box 与 SF 的 touchableRegion 逐像素相同；无障碍窗口集合与 dumpsys 里“4 visible windows”（NavigationBar、StatusBar、InputMethod、Contacts）一一对应。

**源码核对（上游）**：

- `agents4/18/a14-AccessibilityWindowsPopulator.java`（android-14.0.0_r1，从 googlesource 下载）第 55 行 `extends WindowInfosListener`；第 159-166 行只保留可见、非 clone、可触摸区域非空、frame 非空的输入窗口；第 700-704 行把 `inputWindowHandle.touchableRegion` 变换到屏幕坐标。
- A17 `AccessibilityWindowInfo.java:395-417`：`getRegionInScreen` 就是可触摸区域，`getBoundsInScreen` 等于该区域的外接矩形。
- 结论：**无障碍窗口目录就是平台从同一份 SF 输入窗口数据加工出来的公开视图**，坐标已经是屏幕坐标，只少了 ownerUid。

**C（隐藏 `WindowInfosListener`）逐项核对**：

- (a) 稳定性与权限：
  - Java 签名在 14/15/17 一致（`register()` 返回 `Pair<InputWindowHandle[],DisplayInfo[]>`，抽象方法 `onWindowInfosChanged`）。`InputWindowHandle.ownerUid/inputConfig/displayId/transform` 三版都是 public 字段，但 A14 的 frame 是四个 int，15/17 是 Rect（实施方已指出）。
  - 必须用编译期 stub 去**继承隐藏的抽象类**。Proxy 做不到，需要处理 R8 和隐藏 API 豁免，在 Shizuku UserService 的加载路径上没验证过。
  - SF 要求调用方有 `ACCESS_SURFACE_FLINGER`（AOSP 14 的 shell 有）。OEM 的 shell 是否也有，**需设备**。
  - 注册返回的是本进程的缓存，首次为空：探针实测初始 0 窗，要等回调才有数据。所以每条命令都要“注册 → 等回调 → 注销”，或者常驻监听。
  - JNI 忽略 native add/remove 的返回状态（实施方引用 A15 `android_window_WindowInfosListener.cpp:137-157`）；只要漏掉一次注销，SF 每次窗口变化都会把全量窗口 Parcel 推给 worker，直到进程退出。
- (b) 预测与 dispatcher 实际命中不一致的情形：
  - spy 窗口和 slippery、split 跨 UID：固定 UID 后会被拒。
  - 平台私有的 connection 表会剔除部分窗口。
  - display 变换与窗口变换容易用错（实施方已列）。
  - 采样与派发之间窗口变化：多为误拒，不会越权。
  - IME 在副屏：后台屏本来就没有 IME，不涉及。
  - **E 有完全相同的局限**，而且 E 的坐标已经由平台换算好，没有“display 变换用错”的风险。
- (c) 维护成本：隐藏类继承加构建与 R8 接线、每命令回调生命周期、第二个窗口真相源（与图像资格所用的无障碍 shape 并存，两者不一致时怎么解释）、结果表达。

**更简单的替代**

- **A（默认不定向）**：UID 定向真正防住的是：①派发瞬间落到 AndroidCtl 自己的窗口（来源预检能拦主要情形，拦不住检查之后才出现的自身窗口）；②形状检查到 DOWN 之间这毫秒级窗口里突然弹出的他 UID 窗口；③静态的跨 UID 嵌入面，而这一项多数是**误拒**，并不是保护。A 丢掉的是平台在派发时做的正向 UID 检查。实际风险低（**推断**），但这是一项保护强度的下降。
- **E（无障碍窗口目录命中，推荐）**：
  - 规则：在已经与图像 shape 核对一致的窗口目录里，按 layer 从高到低找第一个其区域（`getRegionInScreen`）包含首个 DOWN 点的窗口，读它的 root packageName，用 PM 换算 full UID，拒绝自身 UID 后固定给整条手势。读不到 root 包名、不在 user0、命中自身窗口时，一律在发送前拒绝。
  - 收益：软键盘、分屏另一侧的 App（现行“唯一操作窗”规则在分屏下其实也会误拒，**推断**）、SystemUI 和他 App 的悬浮窗，都能用默认坐标点中，同时**保留平台的正向 UID 条件**。
  - 局限：SCVH 嵌入面（比如 photopicker）的 owner 不等于 root 包，会被拒，表现与现行 B 一样，调用者用 `--expect`。拒绝信息里可以提示“命中区域可能是嵌入内容”。
  - 成本：约 40–80 行，复用 `currentPackage` 的 root 读取和 `AppInput.target`，不需要隐藏 API。
  - 实施方说无障碍窗口信息“没有 ownerUid，root 的 package 也不等于 input channel owner”。对嵌入面这话成立；但现行 B 本身就是“操作窗 root 包 → PM UID”，可见候选已经接受这种信任方式。把它从“唯一操作窗”推广到“点所在的窗”，并没有降低证据强度。实施方没有评估这条路线，**不公允**。
- **允许集合（App + IME）**：平台不支持。`injectInputEventToTarget` 的 targetUid 是单值（`AppInput.kt:100`，`InvalidUid` 表示不限）。“点落在 IME 区域就定向 IME UID，否则定向操作窗 UID”这种两分支规则，是 E 的一个特例，已经覆盖实测缺口；完整的 E 还顺带覆盖了分屏和悬浮窗。

**推荐**：E 作为默认，`--expect` 保留为显式覆盖。把握：中高。IME 情形有原件和离线重算支撑；OEM 上无障碍窗口与 SF 的一致性**需设备**。C 只在用户要求“默认就能点中嵌入式跨 UID 接收面”时再考虑；A 是更省事、但保护更弱的备选。

### 2.5 各通道分工与调用者决策表

各通道语义基本不重叠：AX 语义动作、整值替换、插入、键、几何。“不自动兜底”这条原则本身是对的，避免重复输入。真正的负担来自命令族多、条文密，而不是通道太多。`input.md` 中借贴和 IC 两节约 70 行“法律式”条文。

| 场景 | 首选 | 其次 / 何时换 | 可合并入口（设计建议） |
|---|---|---|---|
| 按钮、开关、链接 | `click REF` | 没反应或 AX 动作不生效 → `click REF --touch`（节点几何触摸） | 不变 |
| 非 click 的语义动作（展开、长按、自定义） | `action REF --action-id` | — | 不变 |
| 滑块 | `range REF VALUE` | 没有 RangeInfo → 坐标 `drag` | 不变 |
| 让目标进入可视区 | `reveal` | — | 不变 |
| 整字段写入（任意文字） | `fill REF TEXT [--submit]` | 节点不可编辑、无 SET_TEXT → 改用插入类通道 | 不变 |
| 在光标处插入 ASCII 或可映射字符 | `type [REF] TEXT --via keys` | 有不可映射字符 → paste 或 editor | **`type`** 统一：`--via keys\|paste\|editor` |
| 在光标处插入任意 Unicode，主屏 | `type TEXT --via editor`（现 `editor commit`） | 服务未启用 → `--via paste` | 同上 |
| 在光标处插入任意 Unicode，后台屏或无服务 | `type REF TEXT --via paste`（借贴） | 不广告 PASTE → `clipboard set` + App 快捷键（接受覆盖），或扩展后的 `--paste-by keys:` | 同上 |
| 定位光标或选区 | 新增 `select REF START END`（AX） | 不广告 → `key [REF] move-home/left/shift+…` | 替代 `editor select` |
| 编辑键、组合键 | `key CHORD [REF]` | — | 不变 |
| 图上才看得到的目标（画布、键盘、菜单） | `tap/swipe X Y`（默认 E 定向） | 嵌入面被拒 → `--expect PKG` | 不变 |
| 定时手势 | `drag/pinch/timeline` | — | 不变 |
| 打开带参数的组件 | `open --intent {…}` | — | 不变 |

- 合并要点：①`editor commit` 并入 `type --via editor`；②`editor select` 改为 AX `select REF`（或保留为 `select --via editor`）；③不新增发送后自动换通道。
- 可以考虑一个**发送前**确定性的默认：`type REF TEXT` 不带 `--via` 时，可映射就走 keys，否则交由调用者选。它不会重复发送，也不改变语义，因为都是光标处插入。属于设计建议，把握中等。
- 另一个缺口：`key` 不支持重复（“One chord is one command”）。在长文本里移动光标要 N 条命令或一个 N 步 batch，这一点支持新增 AX select。

### 2.6 后台屏文本输入

- 现有能力：
  - fill：整值、Unicode，要求可编辑且广告 SET_TEXT。
  - keys：可映射字符，以及 home/end、方向键、shift 选择、ctrl+a/c/v/x。
  - `clipboard set` + `action PASTE` 或 `key ctrl+v`：Unicode，覆盖剪贴板。
  - 借贴：Unicode，带恢复，要求广告 PASTE。
- 缺口：①不广告 PASTE 的编辑器里，Unicode 只能“set + App 快捷键”并接受覆盖；②主屏会出现复制浮层和“已粘贴”提示，打扰真人；③没有绝对光标定位，只能逐键移动；④AX 可读但带标签的字段，没法安全地读改写。
- 最小补法（设计建议）：①**D6 抑制浮层**，影响最大，成本只有一行 extra；②AX `select REF START END`；③借贴支持按键触发（可选）。我方上轮把 AX select 设为“等 V8 结果再加”，现在看，它在主屏也有用（替代 IC select），不必挂在 V8 上。
- 后台屏能否读剪贴板：读剪贴板要求接收方 UID 在某块屏上持有焦点窗口（`isUidFocused`），后台屏有自己的焦点。PASTE 路径在后台屏是否成立，仍**需设备**（我方 V8 依然有效）。

### 2.7 typed Intent、D6、D9

- **typed Intent**（源码核对）：`ActivityIntent.kt:44-48` 只在 SEND 且 `EXTRA_STREAM` 为 uri 时设置 `ClipData.newRawUri`；读授权只在调用方显式指定 `grantRead` 时加（`:48`）。启动走 `IActivityTaskManager.startActivityAsUser`（`HiddenApi.kt:188-190`，`ActivityLaunch.kt:113`），**绕过了 Instrumentation 里平台的 `migrateExtraStreamToClipData`**。所以候选的做法是有意的最小子集，不是自造缺陷。改走平台迁移会自动加读授权，并把 text/html 也带进去，属于契约变更。实施方处置成立。
- **D6**（源码核对）：`ClipboardAccess.kt:79,102` 新建的 plain clip 没有带抑制 extra，遗漏属实。实施方 `inputs.md` §3.3 建议纳入，`route-review-recommendation.md` 的“确定修正与不增加的机制”一节却写“暂不加 clipboard 浮层抑制标记”，**前后矛盾**。我的建议：借写和 `clipboard set` 默认带抑制。恢复原剪贴板时，可以给原 ClipData 的副本加上该 extra 后再写入：`sameClipboard` 只比较 label、mime 和各 item（`BorrowedPaste.kt:320-341`），不比较 description extras，所以所有权核对不受影响。这样恢复时也不会再弹浮层。
- **D9**（源码核对）：`PointerGesture.kt:202` 在 `send` 之前就调用 `dispatch`，`UiBackend.dispatch` 随即 begin 并让图像失效（`UiBackend.kt:309-316`）；`:249-252` 在首个 DOWN mismatch 时把 `possible` 清空并置为 NOT_SENT。`VisualInput.kt:244` 只要 `dispatchStarted` 就做后采样，**不看结果是否为 NOT_SENT**。同意实施方保留派发前 pending 和图像失效（防止在途 UNKNOWN 丢失）。但“首个 DOWN 已明确 mismatch、无副作用”时跳过后采样是零风险的小改：IME 实测这一步花了 2.016 秒，含 4 次采样。实施方把它列为“可选优化、本轮不提”，我认为应该做。保住图像资格的成本更高（bitmap 已释放），不建议做。采用 E 之后 mismatch 本来就会变少，D9 的影响随之缩小。

## 3. 对照实施方

| 实施方指认或结论 | 判定 | 证据 |
|---|---|---|
| 外评“没有读拒写允许实证”错误 | **部分成立**。P8 的确在真机上实测了这个状态（人为 appops）；我方“包内没有实际发生过”与原件不符。但自然状态下的发生率仍未证。 | P8 `result.md`，`commands/read-denied-batch` |
| Termux Unicode 剪贴板替代已实测，外评“从未跑过”错误 | **成立**。补充：中文那次是定向 UID10087 的研究探针，正式全局键只测过 ASCII。 | `e13-termux-ctrl-alt-v/stdout.bin`，`e13-termux-paste.png` |
| WPS/Contacts 证明 IC 有独立价值 → 保留 IC 和独立 select | commit **成立**。select **不成立**：WPS 上 select 被忽略；Contacts 也有 AX 选区，没做对照。另外 E13 报告本身把“不纳 IC”评为合理替代，“建议保留”是取舍，不是证据推出来的。 | `e13-wps-select-middle` / `selected-inspect`（选区仍 13/13）；`T/research/input/followup/report.md` 的 A/B/C/D 表 |
| 借贴“两命令替代不等价：Termux/WPS 还需快捷键或图菜单” | **不成立**（论据错位）。借贴也只走 AX PASTE，同样不能用于 Termux/WPS；这一点不构成二者差别。“失去恢复原对象和组合核值”成立。 | `UiActions.kt:257-262` |
| 合并 select+commit 会向旧 caret 写字 | **成立**（针对我方方案）。 | WPS select 负例；`CurrentEditorInput.kt:113-149` |
| 坐标推荐有限 C；“不需 AX owner 数据库” | “不需 owner 数据库”**成立**。但没有评估基于无障碍窗口目录的 E；而现行 B 本就信任 root 包 → UID，说“AWI 无 ownerUid 所以不行”对 IME 和普通窗口**不公允**。C 的隐藏类、回调和双真相源成本被低估。 | §2.4 离线重算；A14 Populator 第 55、159-166 行 |
| 外评“UID 只防毫秒竞态”是价值推断 | **部分成立**。我方说法过强：静态嵌入面和自身窗口的情形也在其中。但嵌入面那部分多为误拒，不是保护。 | `inputs.md` §4.1；我方专题 03 D1 |
| D6 可作为小调整 | 同意内容。但实施方整体建议写“暂不加”，自相矛盾。 | 见 §2.7 |
| D8 借写后 PASTE 未发出仍报 failed | **成立**（借写是真实副作用）。默认文本写成“未粘贴；剪贴板已恢复”即可。 | `BorrowedPaste.kt:304-312` |
| D9 保留 pending | **成立**。但 NOT_SENT 后跳过后采样不应归入“本轮不提”。 | `VisualInput.kt:244` |
| §8.5 统一映射会混同证明层级 | **成立**。 | 各通道 done 定义不同 |
| §8.6 平台迁移会改授权语义 | **成立**，而且候选绕过了 Instrumentation 迁移。 | `HiddenApi.kt:188-190` |
| “各通道不自动兜底” | 原则成立。但实施方没有处理调用者负担（命令族多、条文密）。用“不是行为等价”否定入口合并，**过度**：入口合并不改语义。 | §2.5 |

**反向检查**：实施方在借贴、IC select、坐标 A 上，多次用“不是行为等价”或“能力损失须明说”来回避判断。等价本来就不是标准。IC select 的损失（WPS 失败、Contacts 有替代）以及坐标 A 的损失（毫秒竞态加自身窗口时间窗）对调用者影响都小，应当明确说出来，而不是一律保留或者交用户。

## 4. 对上轮外评（我方）的更正

1. §3.1 说 Termux 剪贴板快捷键路径“端到端从未跑过”：**错**。E13 报告与原件在旧包中就有（`EVIDENCE_INDEX.tsv` 收录了 `research/input/followup/report.md`）。
2. §3.1 和专题 03 说“IC 唯一正例是 Termux”，并把“WPS 单元格 commit 未写入”当反例：**错或过强**。E13 的 WPS 正文 IC 写入成功（单元格 grid 与正文是不同对象）。
3. §2.3 说“读拒写允许包内没有实际发生过”：**错**（P8 已实测）。我方“价值低”的判断应改成“自然发生率未知”。
4. §2.3 说借贴“只多换来自动恢复”，进而主张删除：**过强**。恢复的价值在后台屏模式下是真实的，不能用“没有冲突证据”否定。更正为保留。
5. §3.2 说 UID“只防毫秒竞态”：**过强**（见上表）。
6. §8.6 建议“改为直接调用平台迁移”，并说不支持 MULTIPLE 是缺陷：**撤回**。
7. §8.5 的统一 done/failed 映射：**撤回**，只保留“统一显示摘要”。
8. §3.1 当时的合并方案（在同一次 capture 里先选后提交）：**撤回**，理由是 WPS 的 select 负例。
9. 我方的 D7（D6）说“真机每次弹、借贴必弹两次”：**过强**，与 OEM 和是否初始化完成有关。抑制建议本身保留。

## 5. 确定缺陷（核过）

1. **默认坐标点软键盘被拒**：`VisualInput.kt:234-240`（取当前操作窗的包）加 `UiScope.kt:50`（操作窗排除 IME）。后果：截图后点键盘字母返回 not sent。原件见 `ime-api34/commands/default-one-visible-letter-tap`。
2. **D6 浮层抑制遗漏**：`ClipboardAccess.kt:79,102`。后果：借写和 `clipboard set` 可能在主屏弹出复制浮层，恢复非空原剪贴板时会再弹一次；后台屏模式下打扰真人（上游依据：A14 SystemUI `ClipboardListener`，实施方已核）。
3. **首个 DOWN 已明确 not sent 后仍做后采样**：`VisualInput.kt:244`。后果：白白多出约 2 秒和多次采样（ime-api34 回包中 tap 自身有 4 次 post samples，CLI 共 2.016 秒）。属于效率缺陷，不影响正确性。
4. **实施方文档矛盾**：`T/deliverables/route-review-recommendation.md` 的“确定修正与不增加的机制”与 `T/research/route-review-3f45af6/inputs.md` §3.3 对 D6 的处置相反。后果：实施时口径不明。
5. 设计限制（不是缺陷，需写进文档）：原剪贴板含 URI 或 Intent 时，借贴直接拒绝（`BorrowedPaste.kt:197-199`），用户剪贴板里有图片时该通道不可用。

## 6. 需要用户裁决的事项

**6.1 默认坐标接收者**

| 方案 | 能力 | 调用者负担 | 维护 | 评价 |
|---|---|---|---|---|
| A 默认不定向 | 什么都点得到，包括嵌入面 | 最低 | 最低，但需要自身来源预检 | 失去平台正向 UID 检查 |
| B 现状 | 点不到 IME、分屏另一侧、悬浮窗 | 高（需显式 `--expect`） | 低 | 已证误拒 |
| C SF 隐藏监听预测 | 加上嵌入面 | 低 | 高：隐藏类继承、回调生命周期、R8、`ACCESS_SURFACE_FLINGER`、双真相源 | 收益只比 E 多嵌入面 |
| D 强制 expect | 同 B | 最高 | 低 | 不推荐 |
| **E 无障碍窗口目录命中** | IME、分屏、系统和悬浮窗；嵌入面诚实报 not sent | 低 | 低：公开 API，复用现有窗口 shape | **推荐** |

未知：OEM 上无障碍窗口 region 与 SF 是否一致；IME 窗口 root 包能否读到（**需设备**）。

**6.2 IC**

- 选项：①保 commit 加 IC select；②**保 commit，select 改为 AX `select REF`（推荐）**；③删 IC。
- 推荐 ②：保住无剪贴板的通用 Unicode 插入；选区能力扩展到后台屏，并按 REF 定向。
- 未知：有没有只认 IC select 的真实编辑器。

**6.3 借贴**

- 选项：①**保留，加 D6，精简回执（推荐）**；②在 ① 基础上加 `--paste-by keys:CHORD`；③删除，接受覆盖（我方旧结论，现撤回）；④无条件恢复（不建议，会覆盖真人新复制的内容）。
- 推荐 ①；若后台屏是主用法，选 ②。

**6.4 剪贴板 null 准入**：保留 emptyBasis 作为内部机制；对外只给三态加原因。没有实质分歧，不必再裁。

**6.5 D9 跳过后采样**：建议采纳，成本极低。

## 7. 会改变取舍的验证问题

1. **E 的一致性**（需设备）：在一台 OEM 真机上弹出软键盘，比较无障碍窗口的 inputMethod region 与 `dumpsys window` 的 touchableRegion，并检查能否读到 IME 窗口的 root 包名。一致且可读 → 选 E；无障碍窗口缺失或 region 不准 → 退回 A（加自身预检），或在用户接受成本时选 C。
2. **嵌入面的使用频率**：如果目标任务经常要点 photopicker、广告或 SCVH 小部件，C 或 A 的增量就有意义；如果很少，E 就够了。这项可以通过任务调研回答，不需要设备矩阵。
3. **后台屏 PASTE 是否成立**（我方 V8，需设备）：成立 → 借贴（加 D6）就是后台屏 Unicode 插入的主通道，值得考虑按键触发扩展；不成立 → 后台屏 Unicode 局部插入只剩 fill 读改写，借贴价值下降到主屏。
4. **IC select 的独有价值**：找一个不广告 AX SET_SELECTION、但 IC setSelection 生效的真实编辑器。找到 → 保留 IC select；找不到 → 删除。
5. **D6 抑制在目标 SystemUI 上的效果**（需设备）：有效 → 默认带上；无效 → 文档写明浮层副作用。

## 8. 未核实与局限

- 没有设备，E 的设备一致性只有一台 API34 模拟器的离线数据。
- 没有读完所有 E13/P9 命令原件：Termux、WPS 的关键原件和图像已亲核；Contacts I/S 臂采信报告与索引。
- 没有核对 IC 正式 bridge 在 WPS 上的表现；28 号阶段原件没有重读。
- A14 Populator 是从 googlesource 新下载的（`agents4/18/a14-AccessibilityWindowsPopulator.java`），没有逐行核对 A15/A16 是否仍继承 `WindowInfosListener`（**推断**仍是）。
- AccessibilityWindowManager 对嵌入窗口和遮挡区域的进一步处理没有逐行读过；“region 已扣除遮挡”属**推断**。
- 后台屏剪贴板读取和焦点规则沿用我方旧结论，本轮没有重新核对上游。
