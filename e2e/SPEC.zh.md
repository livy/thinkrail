# 浏览器 E2E harness

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

ThinkRail 宿主/UI 集成的真实浏览器系统门禁：构建实际发布的 web 客户端、启动隔离宿主、播种真实的 git
与持久化夹具、通过传输线驱动 Chromium，并清理它用过的每一项机器全局资源。默认套件排除由 provider
支撑的 `@agent` 测试；那些保持显式、已认证、按需运行的形态。

## 执行模型（Execution model）

`bun run e2e` 是完整的无 agent 门禁。它构建一次 web bundle，并运行机器自适应的进程级 Playwright 分片。
一个分片拥有一个宿主和一个 Playwright worker；该 lane 内部的串行执行保留了套件的破坏性重置语义，
而按 lane 限定的状态与端口让各 lane 相互独立。Playwright 把单个测试拆分到各 lane，父 runner 把它们的
blob 报告合并成一份普通结果。它还把分片失败 id 合并进 Playwright 的根 last-run 文件，因此
`--last-failed` 仍然是有效的串行修复循环。无论计数是一还是多，无 agent 覆盖完全相同。

自动计数是可用的 CPU 并行度的一半，钳制在 1–8。开发者可以显式选择 1–16 个 lane；`e2e:serial` 是稳定的
调试回退。携带 Playwright 参数的聚焦调用默认使用一个 lane，除非其分片数是显式的，因此对单个 spec 的
迭代保持廉价。CI 用 `THINKRAIL_E2E_JOB_SHARD=k/N` 把套件拆分到多台机器：该调用运行 `N` 片中的第 `k` 片，
其本地 lane 再把该片细分为 `N·L` 个全局 Playwright 分片 `(k-1)·L+i`，因此所有 job 的并集恰好是整个套件。
runner 拥有 `--shard`；把该变量与显式的 `--shard` 参数一起使用会被拒绝。
每个公开浏览器 E2E runner 在 macOS 上都在 setup 或构建工作开始之前拥有一个进程生命周期的 idle-sleep
断言；source、agent、full、binary 与 desktop 模式都会得到它。该断言使用系统 `caffeinate` 可执行文件，
作用域为 idle-system-sleep，并携带所属 runner 的 pid，因此显示器休眠仍然可用，且所属进程突然退出会释放它。
一个私有的继承式 owner 标记让组合式 full 运行的各阶段复用父级的断言，而不是成倍增加辅助进程。
其他操作系统上是空操作，而无法建立该断言的 macOS 宿主会在花费测试时间之前失败。
直接以无 agent 方式使用 Playwright 配置仍然自成一体，并在分片 runner 尚未构建时构建 web 应用。
真实 Central 执行通过公开的 `e2e:agent` 或 `e2e:full` runner 进入；直接以 Central 模式执行测试会被拒绝
（而 `--list` 仍可用），因为公开 runner 必须在把剥离凭据的环境交给 Playwright 之前拥有构建。
独立的 agent 运行总是在 `--list` 之外构建；环境中的 skip-build 不能抑制该阶段。只有 full runner 在构建后
的内部就绪标记才能选择已构建的 agent 计划。随后该计划把 skip-build 与一个专门的 Central runner 授权标记
都加入 Playwright 环境；Central 执行需要两者，因此单独的 skip-build 无法绕过公开 runner。聚焦的
`e2e:full` 调用分别在无 agent 与 agent 模式下独立列出所选内容，只运行有已选测试的阶段，并在两种模式都没
选中任何东西时失败；无参数的 full 门禁及其两阶段 `--list` 行为保持不变。无 agent、agent 与 full runner
拥有它们启动的进程树。当 full 组合某个阶段 runner 时，一个内部 parent-owner 标记让 full 成为唯一的信号
管理器；独立的无 agent 与 agent runner 保留所有权。在 POSIX 上，所有者用安全的非 runner 进程组快照每个
后代 PID，把 SIGINT/SIGTERM 精确转发给每个被跟踪的组一次，并且只单独转发给没有此类组的 PID，
然后在有界宽限期之后强制杀死那些同样的不重叠目标，即使根进程已退出。Windows 先通过 PowerShell 的
`Get-CimInstance Win32_Process` 快照并保留后代 PID，在不可用时优雅回退，并在根回退之前使用
`taskkill /T`；强制终止则对每个保留的 PID 使用 `taskkill /T /F`。受管根只在 POSIX 上是进程组组长。
在 Windows 上它们保持非分离并使用 `windowsHide: true`，因此它们普通的 Playwright、宿主与构建后代继承
一个非可见控制台，而不是分配可见的控制台窗口；Windows 的树终止不依赖分离。单元覆盖只钉住这个
“平台到分离”的选择；原生窗口可见性用一次手工 Windows 控制台探针验证。该保证不延伸到单独的二进制或
桌面产物 runner。主修饰键组合的测试通过一个夹具辅助函数读取页面浏览器报告的平台，在 Apple 上注入 Meta、
在其他平台注入 Control；硬编码 runner 宿主的修饰键会在浏览器/平台仿真下演练错误的产品分支。

由 provider 支撑的浏览器测试（`e2e:agent`）使用专门的串行真实 Central 模式；单独的无头工作流套件保留其
本地 PI 认证模式。并发的 provider 轮次会改变速率限制、成本与确定性，因此两者都不分片。agent runner 在
调用者的正常环境下构建 web 产物，然后在启动 Playwright 或任何宿主之前移除 PI 完整的环绕
provider/云凭据环境面。这些宿主还会收到与默认套件相同的完整封闭 `PATH`。这个划分是承重的：
构建工具链可能需要开发者安装的可执行文件与调用者环境，而精确模型预检必须在没有 API key、token、
Google ADC/project/location，或 AWS profile/key/token/container/web-identity/config 来源使另一个 provider
可用的情况下证明 Central。一个漂移金丝雀从 pi-ai 锁定的凭据发现分发包中推导每个大写环境字面量，
并要求它出现在拒绝列表中；防御性的云来源额外项即使该分发包当前未命名它们也保留。任何宿主都不得发现真实的
`central`、`pi` 或编辑器可执行文件。编译二进制与打包桌面套件仍是不同的产物门禁。每个在每台机器上都有
一个无 lane、互不重叠的命名空间；任何产物运行与 `e2e:serial` 仍在同一个 worktree 中顺序运行。
产物配置分发单个测试，因此透传的 Playwright `--shard=k/N` 会把它们均匀拆分到各 CI 机器，
每台机器启动自己的产物宿主。未来的启动器或部署会为同一套件添加另一个宿主适配器，
绝不复制特性 spec；因此共享行为在每个组合根上都被证明。

聚焦的 `compaction.live` 探针通过真实 provider 演练手工压缩、续接与恢复的上下文。它只降低一次性 lane 的
`keepRecentTokens` 并禁用自动压缩，以保持请求小且确定，并在 `finally` 中恢复设置字节。它既不改变
provider/effort 的选择，也不证明固定输出上限已移除；聚焦的 SDK 预留无关性金丝雀拥有该断言。

## 仓库克隆（Repository cloning）

`clone-project.spec.ts` 通过 Welcome 与 rail 菜单克隆一次性的本地 `file:` 仓库，检查浅/完整历史、
无需 agent 的项目主页采纳、目标冲突保护、URL 纠正、父级选择器、预览门控以及旧宿主隐藏。
取消 UI 覆盖会持有一个传输线回复，以证明 ack 在清理完成之前不会解锁表单；真实的进程拆除与清理由
projects 与子进程单元测试单独演练。不使用生产 Git 凭据或远程仓库。

## 聊天资源控制（Chat resource controls）

资源传输线覆盖通过真实 UI 打开持久化的聊天夹具，并检查权威目录的会话/工作区作用域、缺失输出结果、
不可用控件与空的 stop-all。它的传输线连接使用所启动页面的端口，而不是源码宿主的固定端口：
binary 与 packaged-desktop 适配器为同一个测试启动不同的端点。
历史夹具在进入其工作区之前播种，因此发现在浏览器重载期间不会与刚创建的 placement 的持久化竞争。
它从不播种运行中的资源，也不把转录文本变成执行权威。空目录 UI 探针使用真实响应覆盖 inspector 的
键盘焦点与焦点返回、缺席的 dock、窄布局、welcome/重连水合以及旧宿主能力隐藏；旧宿主探针只改变所公布的
协议版本。退役焦点探针在传输线接缝提供已完成的子项与持久化表单转录，在 inspector 中选中它，
然后降级下一次 welcome；UI、重连、能力退役与 Radix 焦点拆除都保持真实。这个仅终端夹具不是
agent 资源执行的证据。带标签的 provider 支撑探针通过 agent 工具启动真实命令与子项，
在 inspector 中检查有界纯文本日志/转录，验证聊天隔离、重载与已关闭 inspector 的完成，
并演练单个 Stop 与已确认的 Stop all。它们从不选模型，也不播种运行中资源的目录。SDK faux-provider
单元测试不是浏览器 agent 执行的证据。`attention-background-work.live` 端到端钉住后台工作的未读保持：
当真实的后台命令或分离的子 agent 在父级稳定后仍在运行时，宿主绝不发布未读完成，worktree 行显示运行中，
而不是一个点；当窗口未聚焦时（已看到的结果会立刻被读），该点只在被唤醒的轮次之后才亮起，
且运行中的徽标清除。

**MCP servers** 覆盖（`mcp-settings.spec.ts`）通过 runner 的 `bun` 的绝对路径运行服务端包的 stdio 夹具，
因为宿主的封闭 `PATH` 里没有它。它在工作区已持久化的聊天打开时通过设置表单添加一个用户服务端
（进入工作区会自动打开其最近的聊天，因此测试先等待该标签页，而不是与它竞争），在该聊天重载后看到它
Connected · 4 tools，并从 Resources inspector 看到它对该聊天禁用，然后移除它；它从一个未运行的服务端行
打开聊天，在该服务端失败后按聊天重连并显示其脱敏日志，并把一个不可读的用户 `mcp.json` 命名为带编辑提示；
它还证明仓库定义的服务端会标记该聊天的注意标记、等待 Review & approve，并在获批后启动。每个测试都在
`finally` 中移除它写入的 `mcp.json`（lane 的 agent 目录或夹具仓库的 `.pi/`），因此没有后续场景会启动服务端。

原生滚轮探针在点击浮动的 Latest 按钮之后重新指向真实的转录视口；留在被移除覆盖层上的指针不是手势已抵达
滚动容器的证据。中断 Latest 返回的探针在一个任务内完成点击与中断，从当前位置出发：重置 `scrollTop` 而
不发出滚动事件，可能让 hook 观测到的差值为零——当返回的第一帧恰好覆盖了探针的步长时，就看不出分离。
展开锚点探针把完整的头部放在真实粘性面包屑之下，然后再断言固定定位行为。仅与滚动容器相交并不证明头部
未被遮挡。程序化展开锚点探针在采样的 evaluate 内部激活，因此变化后的第一帧总会被测量，而断开的锚点会
作为重新挂载失败，而不是报告已分离节点的零矩形。
Pierre 表面先绘制纯文本，在 worker 的高亮落地后再替换其行，这在较慢的宿主（例如编译二进制与打包桌面产物）
上会很晚；指针探针在高亮 token（`pierreHighlightedTokens`）出现之后才测量或拖拽行。
之后的探针若要让可操作性把屏幕外的转录控件滚入视野，会先等待 `data-scroll-moving="false"`：
那次滚动不是读者输入，因此进行中的 placement 运动会恢复最新边缘，在慢 runner 上它正好落在 mousedown 与
mouseup 之间并丢掉了点击。进入工作区会自动打开其最近的聊天，而那里待答的问卷会立刻索取焦点与显示。
之后按下的 workbench 快捷键属于用户输入，卡片会让位于它们。因此问卷注意覆盖会在安排完 workbench 之后
关闭并重新打开聊天。Monaco 滚轮探针通过经可操作性检查的 `hover()` 抵达编辑器，绝不使用测量出的盒子：
在编辑器布局稳定之前读取的盒子可能已过时，其中心落在左侧面板的缩放手柄之下。每个 CDP 滚轮事件无论其
delta 大小都会滚动 Monaco 50 px（一个合成 tick），因此必须把元素滚出视野的探针会一直滚动直到观测到元素
消失，而不是发送一个巨大的 delta。

## 渲染性能分析 harness（Render profiling harness）

`bun run perf:render [--runs N] [--out file] [--cpu N] [--scenario a,b] [playwright args]` 是一次可选的
测量运行，绝不是门禁。它构建 `apps/web/dist-profile`（见 [[module-web]]），通过 `playwright.perf.config.ts`
在其上启动普通的隔离宿主，并运行 `e2e/perf/*.perf.ts`——这是默认配置不匹配的模式。它使用 worktree 的
串行 lane（端口块与状态目录），因此不要在同一 worktree 中与另一次串行 E2E 运行同时运行它。默认值：
运行 5 次，输出到 `$TMPDIR/thinkrail-render-profile.json`。未知的 `--scenario` 名字会在构建之前失败。
失败的 Playwright 运行不写输出，因此最后一份成功的文件会存活。

- **采集。** 一个初始化脚本在 React 加载之前安装最小的 `__REACT_DEVTOOLS_GLOBAL_HOOK__`，从而把根置于
  profile 模式。每次提交都按 DevTools 的方式遍历 fiber 树（只下潜到重新渲染的子树），并按组件名记录提交数、
  自身时间（`selfBaseDuration`）与包含时间（`actualDuration`，嵌套的同名实例不重复计数）。未命名组件以最近的
  具名所有者标注。hook 在其宿主组件下报告。标签是 bundle 函数名：`$N` 后缀标记打包器为避免冲突而重命名的
  名字（`Markdown$1` 是 react-markdown，而不是应用的 `Markdown`）。一个具名子树
  （`markdown`：根为 `AssistantMarkdown,Markdown`，可用 `THINKRAIL_PERF_MARKDOWN_ROOTS` 覆盖）汇总其最外层
  根之下每个组件的自身时间与渲染次数，因此任何 markdown 引擎都会被计入。
- **计时精度。** harness 通过 `page.route` 为文档与脚本添加 COOP/COEP 头，使页面处于跨源隔离状态
  （5 µs 计时器，而不是 100 µs）；测量浏览器禁用 Chromium 的本地网络访问检查，否则它会阻止来自已填充文档的
  WS。如果页面未被隔离，或自加载以来没有记录到持续时间为非零的 profile 提交（这意味着宿主提供的是 `dist/`
  而不是 `dist-profile/`），运行就会失败。
- **场景。** Chat streaming 播种持久化转录，并以固定节奏在传输线接缝把确定性的 Pi 事件流
  （`text_delta` 块、工具调用、`partialResult` 更新、`agent_settled`）重放进浏览器；这只测量客户端渲染，
  不是 agent 行为的证据。Region toggles 在带活动终端的已播种转录上折叠并展开右侧工具组、隐藏并显示右侧、
  隐藏并显示底部；它的存在是为了捕获会让中心或相邻窗格重新挂载的形态变化（回归表现为
  `ChatView`/`Markdown`/tool 面板的挂载，而不是少量 chrome 更新）。Live file edits 在打开的文件标签页下
  重写 worktree 中的文件。Large diff 打开一个 3,000 行的 Pierre diff 并滚轮滚动它。各次运行交错执行场景；
  runner 报告中位数与运行间离散度。
- **重型场景**（只在 `--scenario` 下；默认集合是上面四个）。`long-stream` 每 15 ms 以 10–60 字符的 delta
  重放同一个约 26k 字符的已播种 markdown 回答（标题、嵌套列表、ts/py/bash/json 围栏、一个 GFM 表格、
  mermaid）；`long-stream-xl` 是约 100k 字符的同一场景。`parallel-agents` 在一个工作区中以标签页打开 20 个
  持久化聊天并交错重放 20 条；`background-agents` 在可见聊天保持空闲时流式传输 19 个隐藏标签页，
  因此它记录的每次提交都是可见 UI 为后台工作付出的成本。随后两者都打开一个隐藏标签页，并断言其流式文本已
  抵达。
- **弱机器。** `--cpu N` 只对测量窗口应用 CDP `Emulation.setCPUThrottlingRate`。
- **额外指标。** 每次运行：长任务（数量、总计、最大）、rAF 帧间隔（p50/p95/max、掉帧的 60 Hz 帧）、
  CDP 主线程 task/script/layout/style 毫秒数与堆增量、场景计数器（delta、已挂载聊天），以及 markdown 子树的
  根渲染次数、自身毫秒数与每 delta 自身毫秒数。主线程时间包含 Playwright 的 `routeWebSocket` 中继，
  因此只跨同一场景的变体比较它。
- **变体。** 从某个 worktree 根运行该命令即测量那个 worktree；输出会记录 `rootDir`、`gitHead`、`dirty` 与
  `dirtyTreeHash`（`git status --porcelain` 加 `git diff HEAD` 的 sha256；未跟踪文件的内容不参与哈希）。

## 桌面支撑模式（Desktop-backed mode）

`bun run e2e:desktop` 针对内嵌在打包 Electrobun 进程中的宿主运行完整的无 agent 套件。一个仅测试用的
环境接缝让 Electrobun 必需的 native 窗口保持隐藏并停在中性本地页面上，并通过 ready 文件公布动态宿主
origin。因此 Playwright 是唯一已水合的应用客户端：当测试页面正在断言排他终端挂载或写入共享 placement 时，
原生 webview 无法接管。desktop 适配器只在 Playwright 完成后才写控制文件，然后要求应用正常优雅退出。

这与 `smoke:desktop` 是分开的：原生冒烟在系统 webview 中加载实际打包的 ThinkRail UI，要求 DOM 就绪加
宿主健康，并通过真实的 Electrobun 生命周期退出。Linux 在 Xvfb 下运行该冒烟，且仅在测试环境中启用软件渲染。
这个拆分同时证明原生窗口路径与广泛的浏览器行为，而不引入两个相互竞争的客户端。

JetBrains Central 覆盖使用一个带状态、独立编写的假可执行文件，它只实现 ThinkRail 调用的
argv/退出/后置条件面（`--version`、`status`、`quota --json`、`add pi`、`remove pi`、`login`、
`update --install`）。它的控制文件保存**以空格分隔的 token**，因为探针报告的版本、凭据是否存在、
某个操作如何失败都是关于宿主的独立事实：单值控制使真实的组合无法表示，而无法抵达的状态是一种没人断言的
失败模式（当宿主低于最低版本时 `update --install` 拒绝需要两者同时成立）。它物化一个仅针对 PI 公开 API
编写的、测试自有的合成 PI 扩展；不复制任何 Central 产物、源码片段、输出字符串、路由、常量、二进制或密钥。
与真实产物一样，它既引入一个新的 provider（绝不是一行），又重新注册内置的 `anthropic`
（一行带 JetBrains AI 标签的已配置行，没有 Sign-out，产物被移除后再次消失）。浏览器场景覆盖
缺失/过时/畸形的探针，加上高于最低版本时保持就绪、更新、登录/重试、原生 add/remove、同步操作串行化、
被监视的外部 add/change/remove、新聊天成功的当代切换、Disconnect 之后旧活动聊天的共存，以及一次已关闭的
合成扩展加载失败之后的启动/运行时保持。外部替换等待已稳定的已配置判定，并验证替换后的模型可见而旧模型已
消失。一个快速的候选可能在浏览器轮询观测到 `configuring` 之前就完成；那个待处理工作状态用服务端单元夹具中
被持有的候选检查，而不是浏览器中的最小可见时长假设。单元覆盖拥有操作 single-flight、watcher 去抖/合并、
过时候选拒绝、带与不带不透明扩展的启动，以及 Central 被移除后新聊天或重新附加聊天的精确模型无回退。
没有遗留迁移、忙碌轮次排空、活动聊天重新附加、补偿、受影响聊天阻塞或恢复封印需要测试。合成子输出、
扩展诊断与 provider 路由字段中的哨兵值被断言不出现在关闭的结果与渲染的设置面中；结构化 DTO 允许列表与
通用宿主映射让这些类别不进入 WS 帧、分析、日志与持久化。

配额覆盖添加测试自有的结构化重复值以及封闭的失败 token——绝不是真实的 Central 输出。它证明顶栏读数只在
Central 健康 + 启用时出现、只在可见时按同步的整秒间隔轮询、对宿主读取去重、在 Retry 时保留陈旧值、
在禁用/断开时隐藏，并在移动宽度下降级；Providers 卡片钉住默认/开关/间隔校验与持久化。已评审 argv 的断言
拒绝任何表现文本形式的配额回退，而忽略的配额字段中的哨兵证明账号/计划/已用/充值/补充/诊断从不跨过传输线
或渲染。

Central spec 共享一个夹具模块，用于面板导航、生命周期状态等待、argv 日志，以及带外宿主变更——
通过把假程序移入和移出 lane 的 PATH 目录来安装/卸载它，以及像用户自己的 shell 那样直接运行它。
宿主侧调用刻意不继承宿主的任何 PATH，因此 spec 永远无法抵达真实的 `central`。因为同一份日志记录双方，
注入宿主调用的 spec 断言的是*计数*而不是仅仅存在——这正是“应用对外部变更作出反应”与“应用重跑了该操作”的
区别。

第二个 spec 覆盖用户实际走过的生命周期，每种情况一个测试：Central 缺失、已安装但未登录、连接时被卸载、
PI 在应用内断开又在宿主上再次断开，以及宿主侧登出。它承重的断言是状态名无法表达的那些——卸载会从新聊天中
撤回 Central 的模型而全局产物存活，因此重新安装通过重新探测修复，而不是再执行一次 `add pi`；
未登录的宿主会提供 Sign in 而**完全没有 Connect**，就绪声明被替换而不是附加注释；以及登出让底下的连接保持
完整，而卡片只渲染一个状态：不带与之矛盾的“Connected”行的未登录行，只有一个 Sign in 且 Disconnect 同样
被扣留，并且一旦用户重新登录两者都恢复。响应式引导保留自己的 spec，由探针看不到的用例驱动：凭据存在，
但 `add pi` 仍被拒绝。

假程序只建模 `Auth` 行的形状——一个带样式的指示器、一个带内边距的标签、一个带样式的值——并在旁边打印一行
哨兵，那行绝不能出现在 UI 中，因为真实命令会打印用户的许可证与服务器。缓存的判定两次塑造套件，
而两处适配都刻意采用*用户*会做的事，而不是深入宿主。翻转宿主凭据状态的 spec 等待 TTL 过去并刷新，
因此该断言属于卡片的文案而不是缓存。而且由于判定为未登录时 Connect 被扣留，每个由连接驱动的场景都刷新直到
按钮出现，而不是假定它——否则更早场景遗留的判定会隐藏它，正如有人在该时间窗内回到面板所经历的那样。
每个状态还会被捕获为 `e2e/screenshots/<group>/` 下的评审 PNG（gitignored、稳定路径、每个状态一张元素截图、
retina、CSS 动画被快进，因此飞行中的弹簧绝不会进入图片）。截图是证据，绝不是断言——只有图片才能捕获的状态
意味着缺少 `data-testid`。跨场景完全相同的文件是一个发现，而不是缺陷：它们正是套件展示两种不同的宿主情况
渲染出同一张无法区分的卡片的方式。

真实 Central 的 agent 模式是显式且只读的。它的 setup 把已授权的全局 Central 扩展当作不透明来源，
并在 lane 的隔离 HOME 下暂存一份权限受限的副本；宿主仍只解析测试自有的 Central 可执行文件，
而该可执行文件在此模式下拒绝每个变更操作。隔离的 PI agent 目录只收到 `settings.json`——绝不是开发者的
`auth.json` 或 `models.json`——因此 Central 是唯一的 provider 来源，无关的本地模型无法进入 picker。
setup 通过公开传输线等待被监视的运行时代次报告已配置，然后要求 `model.default` 等于确切的
`THINKRAIL_E2E_MODEL` 对（变量缺席时使用确定性的套件默认值）。缺失产物、失败的代次或不可用的确切对都会在
任何 provider 轮次之前中止；PI 普通的首个可用回退绝不被接受为测试配置。同一份副本与封闭环境也为私有
重启宿主播种。

**会话状态**覆盖播种完整的转录状态，并驱动归一化的宿主阻塞/运行来钉住 Projects rail 的表现：
需要输入与所有者全局未读结果渲染同一个静态绿色注意点（“Needs attention”），而真正在运行的会话则把
工作区/项目身份图标换成共享的 `RunningIcon` 品牌徽标，颜色为行自身的颜色。rail 不得用问号/对勾/结果字形
或通用 spinner 替代。仅运行没有注意点；排队、隐藏/后台与显式停止的会话保持安静；活动中的被阻塞问题保留其
点，而其正交的运行徽标可以共存。重连/重启快照、直接与被动激活、所有者全局清除，以及快照重试都被覆盖；
活动 agent 覆盖钉住 `data-running`/`running-icon`（其笔画有动画，并在 reduced motion 下保持完整绘制且静止）
以及与 `data-attention`/`attention-dot` 的共存。这些是稳定的视觉钩子，而不是替代状态模型。退役的
`session.activityList → []` 兼容方法保留为一个聚焦的 handler 测试。

**顶栏 chrome**（`topbar-chrome.spec.ts`）在没有桌面端的情况下证明桌面标题栏契约的 web 一侧：
它从测试注入三个宿主公布的 CSS 属性（`--window-chrome-inset-left|right`、`--window-chrome-drag-region`）
并断言占位块宽度、logo/操作簇偏移、不变的 40px workbench 原点、头部解析后的 `-webkit-app-region`，
以及 Update、配额 Retry 与 Settings 都位于 `no-drag` 操作簇内。在普通浏览器中这些属性未设置，
因此第一个场景钉住零内缩、`no-drag` 基线与主题 token 的头部颜色。

Workbench 场景演练归一化的前端本地 frame，而不只是纯模型：frame 的几何/工具放置在工作区切换后存活，
而资源标签页与注意状态不同；关闭最后一个资源保留其空组；显式移除组会重新安置隐藏工作区的资源；
重载恢复按端点/表面限定的本地状态；以克隆会话存储初始化的标签页会重新铸造其活动表面 id，
且两个标签页在重载过程中都保留独立的 frame；同时打开的第二个页面既不采纳对端的文件/终端/聊天 placement，
也不遗漏对端创建的聊天的仅历史域事件；自定义预设 CRUD 同步，本地默认驱动显式的 frame Reset，
而 Apply 只影响其页面。全新或无效的本地文档直接从 Balanced 开始，且套件断言不存在任何当前布局请求。

底部 workbench 覆盖保留全部四种对齐，连同真实侧栈对排除的下角的所有权、侧栏缩放与窄宽压缩期间的实时对齐、
仅由分隔条拥有的侧比例的指针/键盘持久化、不会碰触前一面板滚动条边缘的分隔条命中区域、独立的高度/组缩放、
零主体空间折叠并可通过边缘 rail 与 `Ctrl+F6` 恢复、模态感知的可见性组合键、隐藏时的 PTY 连续性，
以及无进程的默认终端预留。终端创建现在演练宿主 pending-marker 握手加独立的本地 placement，
而不是一次布局修订或对端几何同步。`tool-rails.spec.ts` 覆盖既有窗格中的多个会话、独立的终端组、
窗格簇 rail 语法（簇镜像窗格顺序，边界投放创建窗格，而入口半边投放则加入其中一个，工具在终端条上不绘制
目标，窗格头部标题拖拽其工具，未放置的工具在尾部簇中变暗）、终端窗格所选会话在折叠与重载后存活、
链接的折叠区域、没有隐式激活的键盘焦点、单例拖拽期间的一次浏览器历史工作区切换，以及形态变化期间的 DOM
连续性：它在每次折叠/隐藏/显示之前给中心标签页、聊天输入、终端、Projects 树与 Review 面板盖章，
并断言同样的节点存活，任何面板组被重新挂载都会失败。压缩缩放矩阵使用带显式压缩断言的独立用例，
避免因 rail 占据宽度而意外折叠。`rail-layout-regressions.spec.ts` 覆盖底部测量之前的节点替换、
显式投放到部分/完全折叠的目标（终端落在折叠工具窗格旁会落入自己的窗格），以及跨每个缩放消费方的
缩放中途切换回滚。它在被取消的手势期间检查实时几何、不变的持久化、保留的 DOM、重载，以及随后的缩放可用性，
而不是把仅有一条取消 toast 当作成功。`project-home-chrome.spec.ts` 覆盖无工作区分支：全新起始屏幕与
Project Home 带有左侧工具 rail 与普通 Projects 窗格头部而没有右/底 rail、rail 条目 / Hide / `Mod+B` 切换
及其焦点端点，以及在一个工作区往返过程中与 workbench 匹配的 rail/头部几何。

## 隔离契约（Isolation contract）

无 agent 的 setup 播种一个测试自有的空 `models.json`，并绝不读取或复制开发者的 `auth.json` 或
`models.json`；provider 场景使用合成夹具，而单独授权的真实 Central 模式保留上面所述仅不透明产物的凭据契约。
环绕凭据环境变量仍会抵达无 agent 宿主，因此依赖“是否有 provider 已连接”的检查会先通过传输线读取宿主的
`provider.status`，绝不是仍在加载它的 UI 的一次性快照。模型上下文覆盖通过真实宿主演练
Default / 1M / Custom 选择器：共享与按 provider 的保存、活动聊天保留的限制与新聊天元数据的对比、
Custom 校验与 Apply 焦点、带草稿失效的跨客户端收敛、带 Retry 的脱敏读/写失败、移动键盘使用，
以及 v76 之前的隐藏。

通用与私有重启夹具播种额外分析为关闭且同意已确认，因此无关场景保持不被阻塞。它们还把自动访谈邀请播种为
永久关闭：串行 agent lane 会刻意在各场景间累积已接受的提示，而专门的反馈覆盖直接注入并断言被寻址的邀请
channel。冷启动 onboarding 向导同样被播种为关闭（`onboardingDismissed: true`，在配置文件中一次，
并通过 settings 传输线再写一次，因为宿主可能已先加载其配置），且只有 `onboarding.spec.ts` 通过
settings 传输线重新启用它，并在之后恢复。向导门禁每次页面加载只判定一次，因此该 spec 的冷启动在首次加载
之前先让宿主稳定：重置之后它们等待仍在进行中的 provider 运行时替换（`jbcentral` 的 `configuring` 会持续
报告前一个运行时的 provider）、登出每个已存储的凭据，并除非最多只剩环绕环境 provider 否则失败（那些会跳过）。
它的拆除即使在某一步失败时也尝试每个清理步骤。在任何夹具重置删除转录存储之前，它会通过公开传输线列出并
处置每个活动会话；处置会先中止并稳定未完成的 provider 轮次，因此中途结束的测试无法在下一个场景背后重新
创建无头 JSONL。同意测试通过真实 settings 传输线重置选择，并覆盖默认开启的偏好预置、
Done/普通关闭的接受、立即关闭的拒绝、持久化失败与重试、已确认的重载、跨客户端收敛、提示优先级，
以及 v65 之前的回退。

每个场景的应用重置还会移除 lane 夹具仓库的 `origin` 远程：顶栏的 Remote / Pull request 片段读取共享的仓库
配置，因此推送场景添加的 origin 绝不能重塑后续场景的顶栏。

应用重置不动宿主设置（主题、系统主题对、消息顺序、扩展开关），而 lane 共享一个宿主。改变它们的测试在结束
之前恢复它们；否则后续场景，或 `--repeat-each` 下的同一测试，会继承它。

**GitHub 评审状态**（`topbar-review.spec.ts`）需要看起来真实的 PR 而不使用网络。Origin 是
`git@github.com:<repo>.git`——宿主读取的已配置 URL 与 `git remote get-url` 都看到它——而
`core.sshCommand` 指向 fake-bin 的 `github-ssh`（`ssh.variant simple`），它从本地裸仓库提供 upload-pack /
receive-pack，因此推送、抓取与删除远程分支都是真实运行的。`insteadOf` 重写无法替代：`get-url` 会展开它，
宿主的评审查询随后会看到一个本地路径而永远找不到 PR。lane 的假 **`gh`**（`fixtures/fakeGh.ts`，由 runner 的
bun 下生成的 fake-bin shim 启动，因为宿主 PATH 中没有它）从测试写入的场景文件应答 `pr list|view` 与
`repo view`，对其应用 `pr ready|merge|reopen`，并记录每个 argv，以便 spec 断言确切的变更。
没有场景时它像未认证的 gh 那样失败，因此其他每个 spec 都保持无 gh 的行为；`THINKRAIL_GH_OFFLINE` 仍让
`pr.open` 停留在 compare 交接上。重置会删除场景与日志，并取消 ssh 配置。宿主在整个运行期间按 origin URL
缓存仓库合并设置，因此设置自己仓库设置的 spec 必须命名自己的仓库。

每个并发 lane 都派生出不同的数据目录、HOME、pi-agent 目录、夹具仓库、二进制缓存、desktop 缓存/状态加
ready/control 文件、Playwright transform 缓存、重启产物、picker/editor/provider 控制文件、
host/restart/binary/desktop 端口，以及 Central 夹具产物。transform 缓存是 lane 本地的，因为 Playwright 的
共享缓存假定只有一个 runner 进程；共享它会让一个冷分片消费另一个分片部分写入的 transform。
每个宿主（source、binary、desktop，以及 restart spec 自己的宿主）都会收到指向 lane 夹具 `login-shell` 的
`SHELL`，该脚本位于 Central/editor stub 旁边：`resolveShellEnv()` 总是在启动时探测登录 shell 并合并其 PATH，
因此夹具用该标记以及宿主被给予的同一封闭 PATH 应答该探测（由 shared 的 `LOGIN_ENV_MARKER` 识别），
否则就 exec runner 的真实 shell，让终端 spec 保持在一个真实 shell 上。没有它，探测会把开发者机器的目录合并
进封闭 PATH（并导入开发者的登录环境），spec 就可能抵达真实的 `central`、`pi` 或编辑器。
（`.bun/bin` 这个目录名是历史遗留：它曾经把 PATH 标记为完整以短路一个现已不存在的逻辑。）
文件夹选择来自一个跨 source、binary 与 desktop 宿主的 picker 控制文件：纯内容返回一个路径，
而 `error:<message>` 指令强制一次与平台无关的失败。source 宿主还会收到空的 `DISPLAY` 与 `WAYLAND_DISPLAY`，
因此损坏的控制文件无法抵达开发者的 Linux 显示；原生 Linux 预检仍由单元覆盖。端口分配在各 worktree 之间
保持稳定且防碰撞：注册表声明区分 lane 的逻辑键，同时按真实 worktree 路径检查陈旧性。旧的纯路径声明仍然有效。

跨窗口 placement 探针在对端初始聊天水合之后才改变其布局；创建终端可以合法地取代进行中的被动恢复，
那不是 placement 泄漏。

不同 worktree 可以并发运行。同一 worktree 中的两次完整 E2E 调用仍保持串行；lane id 在多次运行之间刻意保持
稳定，因此被中断的状态会被回收而不是泄漏。任何路径都不得回退到 `~/.thinkrail`、开发者的 HOME/配置树，
或真实的 pi agent 目录。显式的 agent 模式 setup 是狭窄的例外：它把一个已知的全局 Central 产物读入 lane 自有的
副本，绝不从源运行或向源写入，并在 setup 失败或拆除时删除该副本。沙箱化的 home 作为**同时的 `HOME` 和
`USERPROFILE`** 交给每个宿主：`homedir()`——pi 自己的 home 解析——在 Windows 上读取 `USERPROFILE` 并忽略
`HOME`，因此只给 `HOME` 会让 Windows lane 静默泄漏进真实 profile（见 `module-shared`）。每个 spawner——
Playwright web server、产物与活动重启宿主、工作流 harness，以及产物探针——都通过
`@thinkrail/shared/toolHomes` 的 `toolHomeEnv` 取得该 home，因此每个 agent 工具的配置 home（账号检测读取的
东西）落在沙箱中，而新的检测来源会一次性在所有地方被钉住。

## 边界（Boundary）

- **拥有：** `e2e/` 下的浏览器场景与夹具、它们的 Playwright 配置/runner 入口点、隔离与端口分配规则、
  报告编排，以及公开的 `e2e*` 包命令。`e2e/fixtures/git.ts` 是 spec 调用 `git` 的唯一位置
  （`git`、`gitQuiet`、`gitText`、`gitAs`、`commitFile`）；`gitAs`/`commitFile` 钉住一个一次性的 e2e 身份，
  因此被播种提交的作者绝不依赖开发者机器的真实 git 配置。`e2e/fixtures/pierre.ts` 是 spec 触碰
  `@pierre/diffs` DOM 的唯一位置（`pierreLines`、`pierreLineNumber`、`selectPierreLine`、
  `pierreCollapsedContext`、`pierreDeletionsSide`）：一个重命名 `data-line`、`data-column-number`、
  `data-unmodified-lines` 或 `data-deletions` 的 Pierre 发布只需改一个文件，且没有 spec 编码行如何被选中。
  `e2e/phone-review.spec.ts` 是评审面的手机级门禁——Pierre 文件渲染、仅统一视图的 diff、
  在 `PHONE_VIEWPORT` 下点击评论与内联放置——以今天桌面 shell 抵达手机的方式抵达：以桌面宽度打开，
  然后调整大小并隐藏辅助栈。
- **消费：** 构建后的 web 产物、宿主的公开启动/传输线行为、获准的服务端测试夹具导出、CLI 二进制、
  [[module-artifact-tests]] 的 locator、shared 的重试拆除辅助函数、git、Chromium 与 Playwright。
  独立原生/安装器冒烟与共享产物探针归该测试工作区所有，在 Playwright 发现之外。
- **禁止：** 假应用后端、生产启动路径中的 provider 假实现、把浏览器 import 进产品模块、依赖开发者状态的
  默认/无 agent 测试、读取显式授权的 Central 产物之外任何东西的 agent 测试，或让并行 worker 共享一个可变宿主。

## 验证策略（Verification policy）

迭代期间，运行受影响的 spec 并使用 Playwright 的 last-failed 模式。抖动修复用等价的夹具状态替换无关的昂贵
setup，并等待可观测的就绪；笼统重试、任意 sleep 与弱化断言不是同步策略。竞态保护强制危险的交错
（被持有的动画帧、被持有的回复），而不是在 CPU 节流的循环里采样它——那种采样成本随 runner 速度增长，
并在 CI 上超出测试预算。断言某事发生但没有稳定等待时，会在手势之前冻结页面时钟（`page.clock.pauseAt`），
因此回归的计时器绝不会触发，而不是与墙上时钟的 `timeout` 赛跑；随后 `page.clock.runFor` 确定性地播放一个
稳定窗口。真实 provider 的完成在响应证据出现之后等待会话的流式状态；可选的已渲染 `Done` 行不是终态契约。
主题是客户端发送变换的场景断言确切的传出 `session.prompt` 帧，而不是把已挂载的乐观转录行当作投递证据：
快速的 provider 拒绝可能添加一条更高的错误、滚动到最新行，并合法地虚拟化前面的用户行。聊天顺序覆盖播种
多轮转录，并断言两个最新边缘、它们物理上的 **Latest** 目的地、按宿主限定的浏览器本地持久化，以及跨浏览器
隔离，且不涉及 provider。已水合历史覆盖播种一个规范的巨型 Markdown 块，并驱动真实的粗粒度滚轮输入，
因此初始虚拟几何无法在该行挂载之前被钳制。聊天标题覆盖使用持久化的 Pi 转录来证明原生 `/name` 路径、
打开标签页与已关闭历史的行内重命名路径、键盘焦点恢复、有界的长历史滚动，以及重载持久性，且不涉及 provider。
首个提示标签覆盖（`chat-title-fallback.spec.ts`）播种未命名转录以钉住标签条上宿主投射的标签、
最近关闭与会话切换器（`data-title-source="prompt"`、整行作为 tooltip、按词边界截断）、从空的真实标题开始并
替换标签的行内重命名，以及乐观首次发送带来的即时标签；它在没有 provider 时无法持久化已发送的提示，
因此活动路径的持久性归服务端单元测试。Visualize 覆盖（`visualize.spec.ts`）在进入工作区之前通过获准的
会话夹具独立播种简短的历史工具调用/结果转录。它钉住已完成/默认展开的图表、全屏 SVG 增长、
平移/重置/Escape 与声明的 WebKit 双指缩放手势、精确的对比优点/缺点与建议，以及助手 mermaid 围栏与工具图表
在真实设置中选择一个被发现的对侧外观后重新渲染为宿主的 `--container-elevated-bg`。畸形源码被播种为成功的
历史工具结果（`isError: false`），因此工具与聊天都演练渲染器的纯源码回退，而不是工具错误分支。
文件预览把 mockup 写入共享夹具仓库（在 `finally` 中移除），并钉住卡片中的 HTML 渲染器、前后 2-up diff、
区域评论的 Send 通过传输线携带卡片的聊天 `sessionId`、HTML 设计宽度缩放与宽度切换、全屏、Open in tab，
以及一个以对比 diff 标签页打开并在重载后存活的 before/after。巨大的对比会通过 Show diff anyway 选择加入，
同时一个无关文件每 400 ms 变化（每次写入都是自己的 watcher 帧，越过 300 ms 的安静刷新），因此更新持续落在
昂贵的首次挂载之内。这些回归不需要 provider、提示提交、路由 mock 或截图。扩展开关覆盖
（`extensions-settings.spec.ts`）通过 `settings.update` 驱动每个已交付扩展（`visualize`、
`ask-user-question`）的真实“设置 → 扩展”开关，证明它在重载后存活，并播种一个必须仍渲染其图表的历史
`visualize` 调用与一个必须仍渲染其记录的已作答 `ask_user_question` 调用，而此时扩展是关闭的；
每个测试都在 `afterEach` 中恢复每个开关，因为分片共享一个宿主。`environment-settings.spec.ts` 通过被拒绝与
被接受的保存以及一次重载驱动“设置 → 环境”；宿主是以 `PI_CODING_AGENT_DIR` 启动的，因此它还证明启动值
仍在使用并被报告为启动时已设置。它在 `afterEach` 中清除已存储的记录，因此没有后续基于重启的 spec 会带着
它启动。`visualize.live.spec.ts` 中由 provider 支撑的对应项先运行：扩展关闭时 agent 绝不会被提供
`visualize`，而随后既有的图表/对比测试会在同一聊天中重新证明已恢复的工具；最后一个测试要求一个不命名该
工具的 before/after UI 提案，并期望 `.thinkrail/context/mockups/` 下的 mockup 文件以渲染 diff 的形式呈现，
这钉住了该工具的提示准则。MCP 工具卡片覆盖（`mcp-tool-cards.spec.ts`）以同样方式播种历史 `mcp__*`、资源与
`tool_search` 结果，并钉住单一共享图片条、折叠的 JSON 树、通过真实 `mcp.readOutput` 的 Full output
（一个已记录文件与一个已过期文件）、按 pi 报告的认证类型深入“设置 › MCP servers”或 Providers 的 Sign in
链接（provider 身份保持可见），以及针对被截断列表的纯文本回退；它在 `e2e/screenshots/mcp-tool-cards/` 下的
元素截图只是评审证据。问卷分页使用规范的持久化工具调用/ack 夹具钉住超过四个的六问题轮次、长页显示、
新聊天恢复页显示、可见评审焦点，以及粗指针焦点，且没有 provider 波动；桌面包测试单独钉住动态端口重启所需
的稳定 backend-profile/窗口适配器。流式带覆盖保持 `@agent`，因为只有 Pi 真实的行增长才能演练那个生命周期。
流挂载覆盖在 CPU 节流下把合成 `pi.event` 帧注入页面传输线（`fixtures/streamReplay.ts`），以证明工作中的聊天
在挂载时保持显示；它不是 provider 行为的证据。交接之前，每个影响应用的变更都运行完整的 `bun run e2e`
无 agent 门禁。纯产物回归仍由 `e2e:binary`、`e2e:desktop` 及其共享宿主探针覆盖：在 `PATH` 上没有 `pi`
可执行文件的情况下，一个合成的不透明外部扩展对默认与自定义 `PI_CODING_AGENT_DIR` 都能加载；desktop 还额外
证明其暂存的 `.ts` PI 运行时与物理资源。真实 Central 验收保持显式授权且隔离：专门的 `@agent` 套件暂存
不透明产物、校验确切的模型，并证明一次真实轮次，而不是接受另一个 provider 或假 agent。
