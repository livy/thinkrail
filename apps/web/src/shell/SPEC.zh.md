# shell —— 响应式帧

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

响应式组装根：顶层应用 chrome、活动项目/工作区路由、主题应用、全局快捷键、区域错误隔离，以
及把布局无关的面板组装成一个前端本地的桌面工作台帧。未来的移动 shell 可以用不同方式投影同
一批面板；它绝不能意外继承桌面停靠。

## 边界（Boundary）

- **拥有：** `Shell` 作为唯一的组装根；topbar 与常驻的位置上下文；活动项目/工作区路由；单
  次挂载的 Settings、分析同意、问卷邀请、问卷窗口、注意通知权限前言与 Toaster，外加键盘打
  开的 Create workspace 对话框和 `Mod+K` 会话切换器（以及来自
  [[submodule-web-notifications]] 的一次性挂载的注意通知观察者）；主题应用与全局快捷键；注
  入的 Layout 设置段与可选的 application Update 设置段；以及工作台引擎与 store、持久化、面
  板、传输层支撑的领域状态和错误边界之间的集成。
- **公开接口：** `Shell`。
- **允许的依赖：** 子布局模块；`updates`；`panels`；`chat` 的应用集成水合/渲染；`store`、
  `transport`、contracts（类型、线上常量，以及共享的 `createQuitConfirmation` 规则）、
  `@thinkrail/ui/*`、`@thinkrail/locales`（UI 字符串）、`i18n`（`useLocaleSync`，挂载一次）、
  `components/ErrorBoundary`、`components/QuietScrollArea`、`constants`、`lib` 与
  `themes`。
- **禁止：** 引入 server/shared/pi；被 panels/store/transport 导入；把排布知识放进功能面板；
  或通过传输层发送当前的帧/视图状态。

## 内部模块（Internal modules）

每个子项都是以 `index.ts` 为公开接口的目录模块：

- `layout/`（[[submodule-web-shell-layout]]）是纯粹的帧/视图变更、投影与渲染引擎。它从不导
  入功能面板、store/transport 运行时或持久化。
- `layoutState/`（[[submodule-web-shell-layout-state]]）拥有本地水合、校验、持久化、原始
  Balanced 初始化，以及纯布局结果的原子装入。
- `layoutIntents/`（[[submodule-web-shell-layout-intents]]）拥有一次性消费的排布意图路由，
  把它们送进纯布局转换。
- `chatReconciliation/`（[[submodule-web-shell-chat-reconciliation]]）拥有宿主会话/本地放
  置/缓存/历史的收敛与 chat 深链编排。
- `terminalReconciliation/`（[[submodule-web-shell-terminal-reconciliation]]）拥有宿主终端
  目录/本地放置的收敛，但不拥有 PTY 生命周期。
- `legacySelection/`（[[submodule-web-shell-legacy-selection]]）是唯一临时的适配器，把工作
  台注意映射到迁移期的活动编辑器/终端/预览镜像。
- `locationBar/`（[[submodule-web-shell-location-bar]]）拥有 topbar 的带标题位置段及其项目
  / 工作区 /分支控件；它按文件触达 `panels` 与 `store`/`transport`，绝不触达布局引擎。

同级依赖图是：`layoutState → layout`；`chatReconciliation → layout + layoutState`；
`terminalReconciliation → layout`；
`layoutIntents → layout + chatReconciliation + terminalReconciliation`；`legacySelection`
只触达 store selectors/actions；`WorkspaceWorkbench` 把每个活动编排 barrel 与 `layout`、面
板和渲染回调组合起来。chat 资源的可用性被隔离在一个按会话的 selector 组件之后；父级工作台
从不订阅整个 `sessions` 记录，因此一个流式运行时无法让排在它之后的每个标签页渲染器和侧边工
具都失效。同级之间只通过 barrel 导入。测试与拥有该行为的编排模块放在一起，而不是让 store
测试去导入 shell 的运行时副作用。

## 组装（Composition）

topbar 保留 ThinkRail 身份、连接状态、Settings 和紧凑的位置上下文，并在原生宿主移除自己的
标题条时兼作**窗口标题栏**（[[module-desktop]]，*Native window chrome*）。它是一个固定的
`h-topbar-row`（`--topbar-row-height`，40px —— macOS 标题栏的比例，因此原生红绿灯按钮居中
落在其中），带 `px-16`，并与宿主无关：唯一与宿主形状相关的输入是 `<html>` 上的三个 CSS 自
定义属性。`--window-chrome-inset-left` / `--window-chrome-inset-right` 通过
`w-window-chrome-inset-*` 间距 token 被两侧各一个 `aria-hidden` 边缘占位块消费
（`window-chrome-inset-left|right`）；未设置时（任何浏览器）它们解析为 `0px`，头部看起来与从
前一样，而桌面端在 macOS 上发布 `64px`，于是内容从 80px 处开始，并在全屏时收缩到 `0px`。
`--window-chrome-drag-region` 驱动头部的 `window-drag` 工具类，除非宿主发布 `drag`，它解析
为 `no-drag`，因此浏览器标签页或带原生装饰的窗口绝不会多出一条拖动条。内缩刻意没有用
padding 实现：`spacingUsage` 门禁只允许 padding 工具类命名规范刻度，而 CSS
`padding-left: max(…)` 需要门禁豁免。头部是 `select-none`；其整个尾部动作簇
（`topbar-actions`）是 `window-no-drag`，因此放进其中的任何按钮 —— Update 提示、配额 Retry、
Settings —— 从构造上就被排除在拖动之外；`Shell.tsx` 自身不在头部放置其他按钮
（`topbarChrome.test.ts` 把守这一点）。位置条（[[submodule-web-shell-location-bar]]）是唯一
刻意的例外：它的 pill、chip 以及 Remote / Pull request 分体按钮各自带 `window-no-drag`，而
它们的标题与细线仍可拖动，因此用户仍能在控件之间抓住这条位置条。诸如连接标签这样的纯文本仍
可拖动。另一处按钮组是 `NativeWindowControls`：当宿主安装可选的
`NativeWindowControlsBridge`（`__THINKRAIL_NATIVE_WINDOW_CONTROLS__`，[[module-contracts]]）
时 ——目前只有 Windows 桌面端 —— Shell 挂载 `useNativeWindowControls`，它镜像 `updates`
的能力钩子（只读一次全局、`getState`、订阅、浏览器里为 `null`；该状态不带 revision，因此一
旦有推送到达就丢弃初始的 `getState` 快照），并渲染三个由 props 驱动的 46×40 标题按钮（最小
化、最大化或还原、关闭），作为固定在头部右上角、位于宿主通过右侧内缩预留区域内的
`window-no-drag` 覆盖层；当状态显示 `fullScreen` 时它们消失。这些控件不拥有任何窗口状态，
也绝不进入 store；钩子对被拒绝的动作或快照读取用 `console.warn` 报告而不抛异常。Windows 标
题按钮的惯例（无 tooltip、整高点击目标、通过 `feedback-error-subtle` 的偏红关闭悬停）是有
意的；macOS 保留 AppKit 的红绿灯按钮，永远看不到该 bridge。`SettingsDialog` 从头部 portal
出去，不受影响。shell 里没有任何东西命名或导入桌面宿主。`Shell` 挂载可选的更新钩子，注入受
版本门控的空宿主动作，并把归一化状态传入 props 驱动的控件。它的紧凑状态提示打开注入的
Update 设置段；没有相应能力时，两者都不渲染。面板收到的是可选的 React 内容，绝不是启动器或
原生运行时的检查。topbar 的身份是纯图标的 ThinkRail 标记 —— 与 `public/favicon.svg` 同一份
矢量图，内联为 32×32，并通过语义化的 `text-primary` 渲染 —— 位置之前没有分隔线。有上下文项
目时，标记之后跟着 `locationBar` 条：带标题的段**Project · Workspace · Branch · from base
· Remote · Pull request**，其取值是控件（项目切换器、工作区动作 + 同级切换器、分支卡片、带
分支下一步动作的 Remote 与 Pull request 段），其下拉顺序与语法由
[[submodule-web-shell-location-bar]] 拥有。Shell 传给它上下文项目、活动工作区（Project
Home 时为 `null`）、来自 `useBranchStatus` 的分支状态，以及打开 shell 自有的 New workspace
对话框的回调。没有选中的项目时，只留 logo。

紧挨在宿主连接状态之前，topbar 有条件地渲染 **JetBrains 周期配额读数**（协议 v59）：中性的
Coins 图标
+ 按 locale 格式化的 `remaining / total credits`。它只在同步设置开启且宿主报告 Central 健
  康时存在；零值没有警告策略。加载中/已加载是非按钮内容，带新鲜度 tooltip。首次失败是
  `Quota unavailable · Retry`；后续失败把最后的值作为过期值保留，附带一个小的警告新鲜度标
  记与 Retry。数字更新与轮询循环都不是 aria-live。窄宽度会先隐藏 `credits` 与已有的连接标
  签，然后才轮到图标或配额数字。

`Shell` 拥有可见客户端的轮询控制器：在激活/重连/provider 失效/可见恢复/配置变更时立即请求，
然后按配置的 `1–3600` 秒间隔（默认 30）发出一次不重叠的请求。隐藏的文档以及被禁用/被宿主
隐藏的结果会取消计时器；Retry 强制把已完成的缓存视为过期，但仍然并入宿主的 single-flight。
请求序号守卫阻止旧响应恢复已被取代的状态。健康、缓存与去重由宿主拥有；shell 从不直接调用或
解释 Central。

有活动工作区时，`Shell` 直接挂载工作台；只有懒加载的 Pierre 资源体会挂载 worker-pool
provider，因此普通工作区既不会请求 Pierre 的 chunk，也不会启动它共享的池。切换工作区只改变
资源内容与注意，绝不改变帧拓扑、Projects/Specs/Files/Review 的放置、侧/底几何、折叠、可见
性、对齐，或某个组显示哪个单例工具（选中的工作区资源，例如终端，仍按工作区各归其位）。这也
是一条 React 不变量：`WorkspaceWorkbench` 在切换期间保持挂载，靠它的 `workspaceId` prop 重
新指向，绝不以工作区为 key。帧的 chrome —— 面板组、常驻边缘栏、侧/底堆栈、中心条带与
Projects 树 —— 保留各自的 DOM；资源体与按工作区的工具内容被换掉。辅助头部可以按工作区本地
的选择在工具标题与终端会话条带之间切换，而不替换 pane 外壳。资源体既按资源 id、也按工作区
做 key，因为某些布局 id 是确定性的（每个工作区的初始终端都是 `terminal:thinkrail-initial`）：
同一个 id 绝不能把一个工作区的活动 xterm 或 pane 带进另一个工作区的提交。因此一次切换不
会为 chrome 绘制任何淡出、骨架或空帧，工作台调用的每个钩子都必须容忍 `workspaceId` 变化：属
于某个工作区的状态由该 id 限定，或存放在按工作区做 key 的 store 分片里，且为上一个工作区算
出的就绪标志绝不能在针对下一个工作区的那一次 effect 里被读取。首次访问某个本地视图尚未物化
的工作区同样不会重挂载：帧就绪后，工作台在 `layoutState` 装入那个完全相同视图之前的那一次
提交里，渲染该帧的空视图投影，因此帧的 key 永不改变。"Restoring workspace layout" 占位符只
在本地帧水合之前存在。进入或离开 Project Home 是切换 Shell 分支，不算工作区切换。Shell 自
有的 Projects 与 Specs 包装使用 `components/QuietScrollArea`，Project Home 导航器也是如此；
Files、Review 与 xterm 在 `panels` 里拥有各自的内部 quiet-scroll 表面。这些原语从不接收或
推断放置。改变形状的帧命令（折叠/展开、侧边或底部可见性、对齐、组的增删、预置应用/重置）绝
不重挂载工作台的 `ResizablePanelGroup` 或其中的资源体：面板就地增删，引擎以命令式方式强制
已提交的尺寸（[[submodule-web-shell-layout]]，*Shape changes without remounts*），因此当相
邻 pane 出现或消失时，chat 转录、编辑器、终端或工具 pane 保留其 DOM 与状态。只有在外层列与
对齐行之间迁移的侧堆栈（对齐变更，或隐藏/显示拥有底角的侧边）才会重挂载，因为它换了父级。
没有活动工作区时，Shell 挂载 `ProjectHome`：Welcome 与项目导航器并排，使用独立的本地几何
（`thinkrail-shell-welcome` 自动保存外加记忆的展开尺寸），绝不用工作台帧；它与帧的唯一联系是
原始默认值 —— 该 pane 以内置 Balanced 预置的左侧宽度打开，因此全新表面上 Home 与工作台显示
同样的 Projects 边缘。它的 chrome 就是工作台的 chrome：同样常驻的 40 px 左侧工具栏，带一个
Projects 条目（显示时用选中填充与 Fill 图标、`aria-pressed`、`aria-controls` 指向该 pane），
以及同样带 Hide 控件的朴素 pane 头部，这些都由 `layout` 导出的原语
（[[submodule-web-shell-layout]]）围绕 shell 自有的 `useCollapsibleRegion` 组合而成。当引导流
程处于活动状态时（`selectOnboardingActive` —— 冷启动向导已显示，或在冷启动形状的 store 上
仍待定），工具栏与 Projects pane 完全不渲染，因此既无项目也无模型的首次用户只看到向导；
Welcome 面板就地保持挂载（移除的是面板，不是组），因此完成向导后仍保留它的交接对话框。那里
不渲染右侧或底部工具栏：Specs、Files、Review 与终端都是工作区资源，空的工具栏只会宣传无法
生效的工具。工具栏条目、Hide 与 `Mod+B` 是同一个切换：隐藏时，若焦点原本在该 pane 内任意位
置（包括头部）或在其分隔条上，则把焦点还给工具栏控件，否则不动焦点；显示时聚焦最后聚焦的树
控件，否则活动行，否则第一个控件 —— 头部的 Hide 按钮绝不是落点。在工作区内隐藏 Projects 不
会在 Project Home 上隐藏它：Home 是导航表面，保留自己的状态。Settings 对话框、分析同意窗口、
定向问卷邀请、问卷窗口、注意通知权限前言与 Toasts 各自在两个分支之上挂载一次（引导向导活
动期间，它改为通过自己的隐私步骤征求同意，shell 则扣住自己的窗口）；注意通知观察者
（`useAttentionNotifications`）同样只在这里运行一次。同意窗口是一个布局无关的面板，由浏览器
与桌面客户端共享，受已水合的宿主配置与协议支持门控；shell 只拥有它的放置。同意优先于自动问
卷邀请，因此启动时绝不会把两个提示叠在一起；邀请会等待，在同意关闭后再显示。

在 `main.tsx` 的同步首绘应用之后，Shell 是唯一挂载的主题副作用所有者。当
`welcomeGeneration === 0` 时它保留带版本的首选项提示；此后它把 store 的不透明固定 id + 固定
/系统模式 + 可选配对经 `themes` 投影出去，并写入协调后的提示。固定模式没有媒体监听器。系统
模式恰好拥有一个 `prefers-color-scheme` 监听器，在变化时重新应用本地解析出的槽位，并在首选
项变化/卸载时清理它；该本地事件绝不修改 store、调用宿主或改变其他客户端。没有其他组件修改
`[data-theme]`。

## 工作台行为（Workbench behavior）

`WorkspaceWorkbench` 拥有空闲编辑器预加载（`useIdleEditorPreload`）：当
`panels/editorWarm` 的 `useEditorWarm` 允许时（非手机档、非 `saveData`），它安排一次
`whenFilesTreeIdle` 的代码视图预加载（2 秒空闲超时，1.5 秒兜底延迟），该预加载还会等待正在
加载的 Files 树完成绘制。预加载每页只运行一次，不按工作区切换运行，且 `lib` 的
`whenPageActive` 会把它扣住，直到页面可见且在线，并在真正运行前再检查一次：隐藏的标签页会
白白付出约 3.9 MB 与一次 50–70 ms 的任务，而在离线状态下失败的预加载会让 Chromium 在整个会
话内缓存那个失败的 chunk。预加载完成后，从任何表面（Files、快速打开、chat 链接、评审）首次
打开桌面代码视图都不再出现加载兜底；其他渲染器仍在首次挂载时加载。

持久的帧语法与纯操作属于 [[submodule-web-shell-layout]]。Zustand 承载一个 `WorkbenchFrame`、
本地布局偏好，以及按 key 的 `WorkspaceViewState` 值；挂载的文档是派生的，绝不作为第二个
权威被持久化。帧加视图的转换通过 `layoutState` 原子提交。

辅助工具使用常驻边缘栏，其簇镜像各个 pane；终端组始终是独立的 pane（工具与终端绝不共用一个），
并在其 pane 内暴露具名的会话标签页。中心资源保留既有的条带。工具栏的激活只通过布局引擎
改变选择/可见性；New terminal 仍是唯一的创建动作，放置由布局拥有（进入某个终端 pane，或进
入工具 pane 下方的新 pane）—— shell 不注入任何侧边菜单动作。工作台显式接收工作区身份，以便
在不重挂载的前提下栅栏住待处理的拖动/焦点工作。

资源的打开会路由到该工作区最后聚焦的存活中心组。重新打开一个规范资源是选中它已有的本地放置，
而不是复制一份。资源关闭不会在帧组变空时移除该组。显式的拆分/新增/移除/合并命令拥有拓扑；
组移除在一次状态提交之前，确定性地把每个本地保留的工作区视图里的资源重新安家。应用预置遵
循同样的全视图规则。移动一个单例工具，或调整/折叠/显示某个区域，改变的是那唯一的帧；在已有
组之间移动文件/chat/diff/文档/终端，只改变活动工作区视图。指针/尺寸草稿留在运行时的本地，
完成时发布一次本地转换。

布局持久化的边界是 `layoutState`，不是 store。浏览器按后端 endpoint 与前端表面身份限定状态；
原生窗口使用注入的稳定字符串适配器的 profile/window 作用域，键固定且与宿主的动态端口无关。
两条路径持久化并解码同一份有界文档。状态在加载时做 schema 校验，并在重载或受支持的窗口会
话恢复时还原。同时存在的窗口不会观察到彼此的存储写入。没有有效本地文档的表面从 Balanced 帧
开始；旧的宿主快照与旧的浏览器注意键从不读取。

项目/文件/变更/评审/chat/终端视图只接收资源身份、可见性与容器边界。移动一个视图不会改变模
块依赖，也不会让它去审视帧。终端体只在该终端被本地选中于某个可见、未折叠的组时挂载；隐藏的
终端标签页保持未挂载，而它们的宿主 PTY 继续运行。

每次异步的资源/会话/目录水合，在装入数据或后续放置之前都会检查连接代数、工作区寿命与当前本
地帧/视图身份。对等端创建的 chat 仍可通过宿主历史被发现，但不会打开本地标签页。宿主终端目
录成员关系是共享的：对账移除已死的本地引用，并把新发现的目录标签页放进兼容的本地终端槽位，
而不改变帧几何或抢走注意。显式关闭终端仍然属于宿主域的寿命，并在每个表面收敛移除。

默认终端的创建不再依赖宿主布局修订版。工作区创建流程携带一个宿主拥有的 pending 标记；宿主
预留那条确定性的、无进程的终端目录条目，并且只在持久化成功后才清除该标记。随后每个前端把目
录标签页放到本地，通常放进它的底部槽位；PTY 附加仍要等待可见性门禁。

## 布局设置（Layout settings）

内置预置仍归 web 所有。Layout 段展示内置项加上宿主同步的自定义预置目录，而默认预置选择与独
立的侧/底上限属于这个前端表面本地。选中的默认值就是显式 Reset 帧的目标；它不会在工作区切换
时重新应用，因为每个工作区共享当前的帧。捕获/重命名/删除只改变共享的自定义定义。应用了哪个
预置从不被存储：只要实时帧匹配该预置的几何（`frameMatchesPreset`），卡片就读作已应用，因此
手动调整尺寸会直接清除这个标记。Apply 或 Reset（各自后面跟一次确认）替换这个窗口的帧，并重
排所有保留的工作区视图，保持资源身份不变，然后在本地持久化；另一个前端不受影响。预置卡片把
动作放在卡片内部：较长的译文标签会让动作行换行，而不是溢出到相邻卡片上。

## 长操作反馈（Long-operation feedback）

启动一个 agent 会话需要数秒（watcher 就绪 + `session.create`），因此它绝不静默：每一条
chat 启动路径—— New-chat 按钮（空中心与标签栏）与 Settings › MCP servers 的 *Open a
chat*，它们共用 `panels/openChat` 的 `startChatInTab`，`NewWorkspaceDialog` 的创建并启动流
程，以及重新打开已关闭的 chat（`openChatInTab`）—— 都用 store 的按工作区 chat 启动计数器
（`beginChatStart`/`endChatStart`，用计数器是因为启动可能重叠）把请求括起来；worktree 创建按
项目做同样的事（`beginWorktreeCreation`/`endWorktreeCreation`），`ProjectTree` 把它渲染为
该项目下的一行 pending —— 列表原地不动，新 worktree 落在该行原来的位置。消费方在结果将出现
的位置以行内 pending 状态展示它：空中心按钮翻转为禁用的 spinner（"Starting chat…"，同时也
是双击守卫），而在重新打开的 chat 水合期间 chat 历史触发器会旋转。工作区移除会把该计数器连
同其余按工作区状态一起丢弃。

## 错误韧性（Error resilience）

每个独立挂载的工作台资源体 —— 包括文档、终端与单例工具 —— 都有自己的 keyed 区域边界，因此
一个出错的懒加载面板无法让工作台 chrome、同级组或 shell 变空。切换工作区或资源会重置卡住的
区域错误。失败的动态 chunk 提供页面重载，而不是重试同一个过期模块。`main.tsx` 保留围绕
`Shell` 的最后兜底边界。

无效的本地布局状态回退到 Balanced 安全帧，且不污染领域状态。本地持久化失败会让实时帧保持可
用，并报告一条可操作的错误。自定义预置的设置失败会让已实例化的当前帧与目录都保持不变。

一个会话尚未进入本地运行时缓存的 chat 标签页，会渲染与所有其他正在恢复的资源相同的内容骨架
—— 绝不一上来就给出需要手动点的 "Retry"，因为 `chatReconciliation` 的放置/目录收敛在绝大多
数情况下已经在一两秒内自动水合了它，而立即出现的重试按钮会让正常的加载读起来像
"this failed"。只有当水合在短暂的宽限期（`CHAT_RETRY_DELAY_MS`）之后仍然停滞时，
`ChatResourceBody` 才会把骨架换成显式的重试消息，因此重试提示只为它真正存在的那个卡死场景
出现。

## Chat 标题控件（Chat title controls）

chat 标签页已有的上下文菜单新增 **Rename chat**，工作区 **Recently closed** chat 菜单的每
一行也新增一个可见的铅笔动作。与工作区重命名一样，每个动作就地用它自己的标签替换为一条无
chrome 的单行输入框，排版与几何完全一致；该字段预填**真实**标题（未命名 chat 为空，绝不是
它的首提示标签）、聚焦并选中。Enter 或失焦提交，Escape 取消，空白、超过 80 字符与未改动的
值从不发请求。键盘提交/取消会把焦点还给被替换的标签页或历史行控件；指针失焦则保留用户新的
焦点目标。具名的、受视口约束的交互式历史浮层在该行被编辑期间保持打开且可滚动。只有在
welcome 协议支持 `session.rename` 时这些控件才渲染。

**未命名的 chat 读作它的首提示。** 每个展示 chat 标题的 shell 表面都渲染同一个 `lib` 的
`chatLabel`（面板则用 store 的 `selectChatTitle`）：真实名字优先；agent 尚未命名的 chat 以
与具名 chat 相同的样式显示其首提示的开头，否则显示通用标签 —— 切换器也一样，它不再对未读标
题另用一套措辞。该元素携带标签的 `data-title-source="name" | "prompt" | "default"`，并且对
提示标签而言，把完整的第一行作为它的 tooltip。标签条通过 Workbench 的 `chatFirstPrompts`
prop（`selectUnnamedChatFirstPrompts`，按会话 id，也覆盖计划文档的 chat）拿到这些提示，与
重命名回调并列 —— 布局文档本身从不被重新贴标签，因为 Workbench 的每一次变更提交的都是它所
拿到文档的派生物，而被重新贴过标签的副本会把兜底值持久化为标签页的名字；`openPlan` 同样存
储真实的 chat 名（标题 fold 也会随其 chat 重命名计划标签页），因此计划标签页在渲染时定措辞，
并在真实标题落地后跟随它。Recently closed 行与会话切换器读取同一个
`selectChatFirstPrompt`（先读宿主的状态记录，因此某行若由对等端的 `session.created` 添加而
没有提示，会在下一次状态推送时补上）；切换器在 `chatTitle` 旁边携带 `firstPrompt`，这样即
使 chat 以提示展示，宿主的标题回填仍然生效。没有任何东西写这个标签：agent 的 `set_title`
或手动重命名都会通过同一个 `session_info_changed` fold 在所有地方替换它。

shell 把仅限 chat 的变更回调注入到本来领域中立的 Workbench 标签页菜单里，而不是去教纯布局
引擎会话是如何被持久化的。一次提交没有乐观的领域写入：内联编辑器会先回到先前宿主拥有的标签，
直到既有的 `session_info_changed` store fold 在所有地方更新打开的标签页与已关闭的历史；被
拒绝时保留该快照并抛出标准的错误 toast。重命名已关闭的 chat 不会打开或选中它；重命名打开的
chat 不改变放置或焦点。标题自动到达走同一个标签 fold，但不打开编辑器、不发通知、不做焦点转
移。ChatView 的 `/name` 命令是通往同一次线上变更的独立键盘入口。

## 全局快捷键（Global chords）

`useGlobalHotkeys` 仍是应用级快捷键唯一的捕获阶段所有者。它通过工作台命令接口路由命令，而
不是命令式地调用功能面板的 ref：

- `Ctrl+R` 为本地选中的 chat（否则回退到工作区最近的 chat）打开 chat 历史；
- `Mod+B` 切换左侧，恢复本地组/标签页注意，或恢复某个符合条件的单例工具；
- `Mod+J` 对右侧做同样的事；
- `Mod+Shift+J` 切换底部，恢复本地底部注意、一个以底部为目标的单例，或终端创建表面；
- `Mod+N` 及其别名 `Mod+Alt+N` 为上下文项目（活动工作区所属项目，否则选中的项目）打开
  `NewWorkspaceDialog`。shell 以本地状态拥有这唯一一个由键盘打开的实例 —— 它不是 store 状
  态，也不是 `ProjectTree`/`WelcomePanel` 的实例，后者保留各自的返回焦点与提示播种语义。
  `Mod+N` 是规范快捷键，也是快捷键 chrome 首先展示的那一个；别名之所以存在，是因为浏览器为
  新建窗口保留了 Cmd/Ctrl+N 并且从不把它投递给页面，而桌面 webview 会投递。这是唯一接受
  Alt 的快捷键。
- `Mod+K` 打开 `SessionSwitcher`，一个列出活动会话的模态面板 —— 只列出需要注意或正在运行的
  会话，按 `ProjectTree` 顺序按项目分组，组内按 needs-input > error > unread > running 排序。
  它由全局归一化的注意快照（`sessionStateByWorkspace`）驱动，因此能呈现本客户端尚未加载
  的工作区里的会话；它在打开时用 `workspace.list`/`session.list` 回填这些会话的工作区名与
  chat 标题，并通过 `activateWorkspaceFromRoute` 跳转到所选项。它的打开状态存放在
  [[submodule-web-store]]（`sessionSwitcherOpen` +`openSessionSwitcher`/
  `closeSessionSwitcher`/`toggleSessionSwitcher`），这样来自
  [[submodule-web-notifications]] 的聚合注意通知可以在点击时呈现该面板；shell 通过
  `toggleSessionSwitcher` 绑定快捷键。由于面板本身就是一个模态，全局快捷键在它打开期间被抑
  制，因此它自己拥有 `Mod+K` 的关闭切换（与 `Esc`/背景点击并列）。它既不接受 Shift 也不接
  受 Alt。
- 行与头部的字形复用 `ProjectTree` 的词汇：前导的 `RiChat2Line`（运行时包在 `RunningIcon`
  里），以及为需要注意的会话准备的尾部 `AttentionDot` —— 没有按结局定制的字形。

字母快捷键匹配物理 `KeyboardEvent.code`，绝不使用依赖布局的 `key`。布局与创建类快捷键在
xterm 内部仍归应用所有，不会重复触发，并在模态对话框打开时被抑制。没有活动工作区时，右/底
快捷键既不起作用也不吞掉浏览器的快捷键；`Mod+B` 按上面的焦点规则切换 Project Home 的
Projects pane。没有上下文项目时，`Mod+N` 同样既不起作用也不吞掉。终端的 `Ctrl+R` 仍归
xterm；`Ctrl+Shift+R`、macOS 的 `Cmd+R`、F5 与浏览器重载保持原样。其他所有排布操作由
[[submodule-web-shell-layout]] 的布局命令/菜单系统暴露。

**原生快捷键。** Shell 总是挂载 `AppShortcuts` 叶子。`useAppShortcuts` 只读一次
`NativeShortcutsBridge`（`__THINKRAIL_NATIVE_SHORTCUTS__`，[[module-contracts]]；仅桌面宿
主）。没有该 bridge 时它不加任何监听器也不渲染任何东西。有 bridge 时该叶子渲染提示，因此提
示变化只会重渲染那个叶子。每个应用级关闭/退出/设置快捷键都通过同一张命令表
`shortcutCommands.ts` 分发：一个命令 id（`close-item`、`quit`、`open-settings`）按快捷键平
台映射到 Ctrl 组合键，每个组合键携带它的终端规则，钩子提供 id → 处理器的映射。来自
`subscribeCommand` 的原生命令与 web 的 keydown 组合键运行同一个处理器。有 bridge 时快捷键
平台是 `platformFamily()`（`apple` / `windows` / `linux`），否则是 `browser`。

| 平台 | 退出 | 关闭条目 | 打开 Settings |
| --- | --- | --- | --- |
| apple | 原生 Cmd+Q（宿主确认） | 原生 Cmd+W | 原生 Cmd+, |
| linux | web Ctrl+Q（web 确认） | web Ctrl+W, Ctrl+F4 | web Ctrl+, |
| windows | 无（Alt+F4 是操作系统关窗） | web Ctrl+W, Ctrl+F4 | web Ctrl+, |
| browser | 无 | 无 | 无 |

web 组合键只用 Ctrl（无 Meta/Alt；除产生标点组合字符的情况外无 Shift），在窗口捕获阶段处理，
并且每次物理按下只生效一次：重复的 keydown 会被吞掉但绝不执行。在 xterm 内部，Ctrl+W 既
不被处理也不被阻止，因此 shell 能拿到它；Ctrl+F4、Ctrl+Q 与 Ctrl+, 仍归应用所有。这张表在
apple 上不绑定任何东西（原生拥有那些组合键），在浏览器里也不绑定任何东西，因此钩子只在
`hasWebShortcuts` 为真处添加窗口键盘监听器。字母组合键匹配打出的拉丁字母（`key`），当布局
打出非拉丁字母时回退到 `code`；标点组合键（Ctrl+,）匹配打出的 ASCII 字符，当 Shift 产生了
它时允许 Shift（俄语 Windows 的 Shift+/），当布局在那里打出非 ASCII 字符时回退到不带 Shift
的 `code` —— 因此 AZERTY 的 `Comma` 键（`;`）与 US 的 Shift+Comma（`<`）都不匹配；macOS 菜
单也按字符匹配。F4 匹配 `code`。这是对上面那条物理 `code` 规则的刻意例外：关闭与退出是破坏
性操作，而在 AZERTY 或 Dvorak 上 W/Q 位置打出的是撤销或全选。先匹配修饰键与组合键；只有组
合键匹配后才运行 xterm 检查。web 所有的组合键在焦点位于沙箱 HTML 预览 iframe 内时不触发：
它的 keydown 永远到不了父窗口。

`open-settings`（原生或 web）只在没有模态层打开时打开 Settings 的默认段：已打开的 Settings
保持原样（段落与草稿保留），另一个对话框则保留键盘。这道守卫在这里，而不在 store 的
`openSettings` 里 ——后者的调用方会深链到具体段落。浏览器刻意不绑定任何 Settings 组合键：
Mac 浏览器为自己的设置保留 Cmd+,，而桌面才是主要表面。Settings 齿轮的 tooltip 只在绑定处提
示该组合键（`Settings (⌘,)` /`Settings (Ctrl+,)`），来自同一次 bridge-and-platform 读取
（`readSettingsShortcutLabel`）。

退出确认在两个地方都运行共享 contracts 的 `createQuitConfirmation` 规则：macOS 上由宿主执
行，Linux 上由 web 执行。web 适配器从 Ctrl+Q 的 keydown 一直跟踪到同一物理 `code` 或
Control 的 keyup，在 keyup 时调用 `sync()`，并把窗口失焦或文档隐藏当作 `cancel()`：已确认
的手势（提示 `release`）就退出，否则就隐藏。已确认的退出调用一次 `bridge.quit()`；若它被拒
绝，适配器重置规则，于是 pill 隐藏、Ctrl+Q 重新可用。无论哪种情况，web 都把提示渲染为
`QuitHintOverlay`：一个居中的、`pointer-events-none` 的 `role="status"` pill，在 `hidden`
时淡出并在淡出期间保留最后一段文本；文本标明平台组合键（`⌘Q` /`Ctrl+Q`）。

`close-item` 绝不是关窗：若有可关闭的层打开着（任何位置的模态或打开的 Radix
dialog/alertdialog/menu；chat 历史浮层只在 DOM 聚焦的组内，或在没有组持焦点时的任意位置；
selector 与 xterm 测试与 `useGlobalHotkeys` 共用，位于 `shortcutLayers.ts`），钩子会把
`Escape` 派发给聚焦元素，让每一层保留自己的 Escape 语义。该 Escape 是合成的（不受信），因
此每个可关闭层必须在 document/window 级别或在焦点的祖先上捕获 Escape，且不得按 `isTrusted`
过滤；Radix `DismissableLayer`（document 捕获）与 `HistoryOverlay`（window 捕获）都如此，
e2e `close-quit-shortcuts` 把守这一点。否则它会在 shell 自有的 `closeRequestChannel.ts` 上
触发 `requestClose`（作为 `subscribeCloseRequest` 注入工作台，因此没有渲染级联），工作台通
过与被点标签页 X 相同的 `onRequestClose` 路径（含终端确认）关闭聚焦组当前选中的标签页，否
则关闭最后聚焦的中心组的。单例工具、折叠的组与整个区域从不被关闭；没有可关闭对象时是 no-op。
没有活动工作区时只有层这一步生效。每个组合键只有一个所有者：命令表里的组合键绝不同时绑在
`useGlobalHotkeys` 里，而宿主原生转发的组合键在该平台上绝不出现在表里。
