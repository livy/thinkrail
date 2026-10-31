# 面板 —— 功能视图

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

与布局无关、由 store 驱动的功能视图。面板填满其容器，
且从不感知自己的排布方式（因此移动端 shell 是叠加层，而不是重写）。

Files（两种模式）与 Review 把固定工具栏留在面板自有的 `components/QuietScrollArea`
之外（Files 经由 `RememberedScrollArea`）；Projects 与 Specs 则为 `shell/SPEC.md`
中描述的、由 shell 拥有的滚动容器暴露内容。`TerminalInstance` 用 `QuietScrollFrame`
包装 xterm，它为 xterm 的后代自定义滚动控件做外观处理，却不缩小其命中区域；
顶部/底部状态来自 xterm 公开的 `buffer.active.viewportY/baseY` 以及
`onScroll`/`onWriteParsed` API，而不是把它的非原生视口假装成拥有 DOM 滚动度量。
这些边界同样权威地表明是否存在垂直回滚缓冲，使该 frame
能够在不发明第二套滚动模型的前提下，为本地意图与无障碍模式暴露 xterm
原本不可见的控制器。因此同一种中性的、意图触发的滑块 + 方向性幕帘会跟着终端走，
无论它被放在哪里。功能视图从不接收或推导左/右/底部的摆放位置来达成这种处理。

## 边界（Boundary）

- **拥有：** `ProjectTree`。每个顶层项目行都是紧凑的 IDE 树行，高度用共享的
  `--panel-row-height`（`web-spacing`）：**始终可见的 chevron** + 文件夹/名称 +
  仅折叠时显示的纯文本工作区计数 + **始终可见、位于固定右缘列的 Create workspace `+`**。
  那个 `+` 与 **Projects 头部 Add project `+` 是同一个控件** ——两者都是 `IconButton`
  ghost `s`，因此渲染完全一致，字形也对齐在同一垂直轴上（二者都位于行的 `pr-xs` 右缘）。
  Create workspace `+` 带有 tooltip 与无障碍名称，名称通过 `lib`
  的 `platformShortcutLabel` 指明 shell 的 `Mod+N` 组合键及其 `Mod+Alt+N` 浏览器别名（见
  `submodule-web-shell` 全局组合键）；上下文菜单项则保持朴素。
  长名称在计数/操作之前截断；这里刻意**没有可见的 Close 或 overflow 图标**。
  悬停会高亮整行，且高亮在其**项目上下文菜单**打开期间保持。右键在指针位置打开该
  PR-#167 风格的菜单，且不选中/不导航；约 700ms 的长按在滚动取消后是其触摸等价物。
  当项目名按钮获得焦点时，标准 Context Menu 键或 Shift+F10 会为纯键盘使用打开同一菜单；
  方向键/激活/Escape 键正常工作。菜单是中性的：**Plus Create workspace**、**FolderOpen
  Open existing worktree…**、分隔符、**Pencil Rename**（仅当宿主的协议至少为
  `PROJECT_RENAME_PROTOCOL_VERSION`）、**X Close project**。Rename
  把项目名替换为与工作区重命名相同的行内输入框（Enter/blur 通过 `project.rename`
  提交变更后的裁剪名称，Escape 取消），并等待 `project.updated` 推送 —— 没有乐观标签；
  拒绝则抛出错误 toast。Create 与直接的 `+` 流程完全一致。Open existing worktree 会打开
  `ExistingWorktreeDialog` 选择器，数据来自 `workspace.listExisting`（每行含分支 +
  绝对路径；detached-HEAD 行可见但禁用）；选中后调用 `workspace.openExisting`，
  随后展开该项目并激活所附加的行，但不启动聊天。Close 打开一个居中、中性的
  `ConfirmDialog`，标题为 **“Close {name}?”**，描述为**“Removes this project from the
  open projects list. Its repository, workspaces, chats, and running activity are kept.
  Reopen it from Add project → Recents.”**，初始焦点在 Cancel，以及**Close project**；
  Cancel、背景点击与 Escape 均可关闭。确认后触发 `project.close`
  并等待完整的 `project.updated` 推送 —— 没有乐观移除；成功即行消失且无 toast，
  拒绝则保留该行并抛出错误 toast。菜单/对话框关闭后恢复源项目名焦点；
  成功关闭则聚焦回退项目名或 Projects 视图的 Add project 控件。`ProjectTree`
  还拥有每个项目 `+` 打开的 `NewWorkspaceDialog`，**以及**每个工作区行悬停显示的**kebab
  菜单**（`MoreVertical`，受控 `DropdownMenu`）—— 在行上任意位置右键都在 kebab
  处打开同一菜单，且不选中/不激活该工作区，而 kebab 仍是触摸与键盘聚焦路径。其动作包括
  `DropdownMenuSub` **“Open in”**（仅在检测到至少一个编辑器时渲染）、**Copy path** 与
  **Reveal in file manager**。由 ThinkRail 托管的工作树在连接宿主的协议至少为
  `WORKSPACE_RENAME_PROTOCOL_VERSION` 时额外获得**Rename**，外加 **Remove workspace**；
  外部行只有 **Remove from ThinkRail**，其确认承诺检出目录及其分支保持不动。Default
  两种变更都没有。“Open in” 来自宿主级的 `editor.list`；GUI 条目调用 `workspace.openIn`，
  而 terminal 类型的 Vim 会激活该工作区并通过 `addTerminal` 的一次性 `initialCommand`
  运行。Copy 写入 `worktreePath`；Reveal 调用 `workspace.reveal`。这些请求、
  `editor.list`、项目工作区列表加载、
  移除确认（`RemoveWorkspaceDialog`）以及下面的行内重命名控制器（`createRenameController
  ` / `useInlineRename`，与项目重命名共用）都只存在于 `workspaceActions.ts` 中一处，使
  shell
  的顶栏工作区菜单（[[submodule-web-shell-location-bar]]）无需第二份实现即可提供相同动作
   ——包括能力感知的待处理重命名。控制器把请求派发到编辑开始时捕获的那个工作区，并在
  hook 的工作区标识变化时重置，因此未按工作区加 key 的消费者也无法把待处理名称派错。
  **Settle** / **Keep active**（`workspace.settle` / `workspace.unsettle`，
  以 `WORKSPACE_SETTLE_PROTOCOL_VERSION` 为门禁）走同一个文件。**Settled 置物架。**
  仅当宿主声明支持 `WORKSPACE_SETTLE_PROTOCOL_VERSION` 时才渲染；
  旧宿主保留原始的扁平列表与顺序，即使回滚持久化携带了更新的陈旧字段，也没有置物架或
  settle 控件。在支持该能力的宿主上，展开的项目先按 `selectWorkspacePartition`
  的最新创建优先渲染其**活跃**行（绝不按活动时间重排，因此 agent 工作时行也不会移动），
  然后是一个 **`Settled · N`** 展开头（默认折叠，按浏览器存于 store），
  其正文以**紧凑**方式列出已结算行 ——单行、只有名称、一个小**原因 chip**（`merged` /
  `closed` / `idle 2w` / `by you`，长格式在 tooltip 中）、kebab 旁悬停显示的 **↩ Keep
  active** —— 一次十条，并带一个 *Show 25 more* 行。活跃行在 kebab 旁获得悬停显示的**✓
  Settle**（菜单与右键中同样有）；当该行的 agent 正在工作或有未读结果时禁用，Default
  上不存在。在 v86+ 宿主上，同一菜单区还带一个来自 `selectWorkspaceReadAction`
  的已读切换：当该工作区中任一聊天有未读结果时为**Mark as read**，
  否则当存在可重新点亮的已完成、未取消结果时为 **Mark as unread**，
  再否则什么都不显示（`workspaceActions.ts` 中的 `markWorkspaceRead` /
  `markWorkspaceUnread`，与顶栏共用）。未读只点亮最近活跃的那一个聊天，
  因此一次访问即可清除；已读从不解除 needs-input 阻塞。该 partition 会在共享的 30 秒
  `useNow` 时钟上重算，使空闲行无需点击就能跨过时间窗口，并让 rail 与顶栏保持同步；
  选中某个已结算行会展开置物架并翻页到能显示它的位置。
  **第一次自动移动在每个浏览器中只通告一次**（`settledShelfNotice`，
  与同类持久化模块一样是带宿主限定前缀的 localStorage 标志）：
  当 `selectAutoSettledCounts` 首次报告由置物架自行收纳的行 —— 合并/关闭的 PR
  或空闲窗口，绝不是手工停放的行 ——
  且该计数稳定保持两秒（升级回填会以一阵推送的形式落地，
  通告应统计这阵推送而不是它的第一行）时，一个 info toast 显示 *Moved N quiet
  workspaces to Settled*，说明规则，并提供 **Show**，它会展开那些项目及其置物架。
  否则升级后的首次启动会一次清空长 rail 的大部分内容且毫无解释。结算完全不碰磁盘：
  置物架只是一个列表状态，每行自己的 *Remove* 仍是唯一的拆除方式 ——目前还没有批量移除，
  因为安全实现需要先有一个非阻塞、崩溃安全的宿主拆除机制（#688）。
  为什么是这种形态（一次性记录）：只允许显式归档的 rail 把家务留给了用户（作者机器上有
  48 行），纯按最近使用的折叠无法说“已完成”，而隐藏的 Archived
  层会让行在没有可见归宿的情况下消失 ——置物架让每一次自动移动都只差一次展开，
  并把原因写明。原因是一个描边的 `tr-text-caption` 徽标：merged 用 info 色调，
  settled-by-you 用 warning 色调，closed 与 idle 作为中性的 `border-default` /
  `text-subtle` 元数据。重命名会就地用无外壳的单行输入框替换该行的名称 span，
  并沿用相同的排版、颜色与几何尺寸；它已预填、获得焦点并被选中。Enter 或 blur 提交，
  Escape 取消，空白或与编辑起始标签相同的文本会直接退出而不发请求，
  因此关闭未改动的编辑器不会被传入的对端快照回滚。当当前 socket
  的能力未知时所做的变更提交会保持待处理，只有在 v55 或更新的 welcome 恢复 `canRename`
  之后才派发；旧宿主永远不会收到该方法。提交不会在客户端留下乐观域状态：
  该行回到先前由宿主拥有的标签，直到所有界面都采纳完整的快照 `workspace.updated` 推送；
  拒绝则保留该快照并抛出错误 toast。Git 分支与工作树文件夹从不改变。Remove
  使用破坏性样式并打开居中的 `ConfirmDialog`；确认后触发 `workspace.remove`，
  并让每个客户端通过 store 的 `applyWorkspaceRemoved` 响应宿主的 `workspace.removed`
  推送；被拒绝的请求（不会有事件到来）会暴露错误 toast，行保持原位。
  每个**工作区行**是**两行**：上面是显示 `name`，其下第二行是 git **分支**（弱化、
  等宽），仅当它与名称不同时渲染（因此 pristine/legacy 的 `workspace-N`
  行保持单行紧凑）——显示名称与 git 分支解耦（见 [[submodule-server-workspaces]]）。

  工作区/项目的会话呈现只来自规范化后的宿主状态。rail 恰好有两种视觉处理：针对具体的
  needs-input 阻塞或所有者级未读结果的静态绿/强调色**注意点**，
  以及在顶层会话确实正在工作时用共享 `RunningIcon` 品牌徽标**替代**工作区/项目身份图标。
  注意力是二元的：needs-input 与未读结果状态使用同一个点，无障碍标签为**“Needs
  attention”**，且不带问号/对勾/结果字形、spinner、计数或状态专属 tooltip。Working
  保持该行既有的激活/非激活颜色（徽标使用与被替换字形完全相同的 `text-primary` /
  `text-text-muted`），并以无障碍方式暴露**“Agent working”**；它绝不添加第二个标记。
  排队、隐藏/后台、已停止与安静会话显示其身份字形；needs-input
  会话在其正交的执行事实仍是运行中时也仍可显示徽标，因此注意点与 working 处理可以共存。
  减少动效时徽标保持静态。折叠项目的汇总使用与工作区行相同的选择器，
  而展开项目在工作区行上显示细节。这些组件仍是对规范化宿主状态选择器的纯 props 驱动。

  `ProjectTree` 从规范化宿主状态选择器渲染共享的轻依赖 `AttentionDot`。它是静态强调色，
  不携带计数或状态专属字形/tooltip，并在身份按钮与悬停显示的 kebab 之间占据独立的 flex
  列。工作区行与折叠项目行仅在为正时暴露 `data-attention`。另外，共享 `RunningIcon`
  在规范化的 working 状态下替代身份图标；工作区行与折叠项目行仅在为正时暴露
  `data-running`。这些属性是测试钩子，而不是第二套状态模型。

  **项目行只在折叠时携带工作区计数**；展开时由工作区行直接提供细节。**Default
  工作区**（`kind === "default"` —— 项目文件夹本身）**固定渲染在最前**（服务器在
  `workspace.list` 中将其置顶；`addWorkspace` 把创建出的工作树行追加在其后），用
  **`House` 图标**替代 `GitBranch` 字形，并且**没有 Rename 或 Remove
  项**（不可重命名/不可移除 —— 服务器同时强制这两点；菜单只是不显示）——
  它仍像每个工作树一样获得 “Open in” / Copy path / Reveal。
  其分支行显示该文件夹真实的当前分支。当**所选项目的**权威工作区列表落地时，
  `ProjectTree` 会以 fire-and-forget 方式为最多前八行调用 transport
  的 `prewarmWorkspaceSkillLoad`：常见的可见集合会在用户点击工作区之前开始保守的 watcher
  就绪窗口。这个按选择的配额限制了请求扇出；*全局*上限在宿主侧 —— 仅预热的 watcher
  存在于一个有上限、可淘汰的池中（服务器 `watch` SPEC），
  因此在一次宿主生命周期内点击浏览许多项目会复用该池，而不是累积 watcher。
  列表从不等待预热，失败保持静默且可在最终聊天加载时重试，
  而且仅仅展开后台项目不会预热它（预热以*所选*项目为门禁，
  因此下面那个延迟的恢复展开抓取也保持该不变量）。**Rail 的展开状态是按浏览器存储于
  store 的视图状态**（`store.expandedProjectIds`），不是组件状态：它能跨过
  Project-Home/工作区重挂载边界，并通过 `projectExpansion` 持久化模块（localStorage，
  带宿主限定前缀的 key，启动时从 `main.tsx` 水合，尽力写入，读取不受信任）跨过页面重载
  ——重载后 rail 看起来一样（置物架的展开状态刻意不持久化 —— 折叠才是正确的默认）。
  那些持久化展开状态超出本客户端已抓取列表的行（全新重载）会惰性抓取其缺失的
  `workspace.list`；已抓取的列表会在显式展开手势时刷新，并由 transport 在新的
  welcome/重连代际之后刷新，绝不在循环中重复抓取。活动工作区也必须保持可见：当
  `ProjectTree` 带着活动工作区挂载时，
  或活动工作区推导出的所属项目发生变化或首次变得可解析时，它会展开那个父项目 ——
  这种揭示作用于持久化基线*之上*（持久化的折叠绝不隐藏活动工作区）。在所属项目未变时，
  手动折叠仍被尊重；普通的 `workspace.updated`
  快照与同项目内的工作区切换不会强制再次展开它。导航恢复是中性的：
  重载会重新选中路由中的项目而不触碰展开状态（持久化状态*就是*视图）。
  创建工作区会显式展开其项目。选中或创建工作区也会选中其所属项目，
  即使创建对话框的项目选择器指向另一个项目，项目主页与活动工作区上下文也保持连贯。
  **打开项目会落在该项目的 Welcome** —— 刻意**不自动进入**任何工作区：Welcome 是分叉点，
  把两种工作模式（隔离工作树 vs 项目文件夹的 Default 工作区）作为显式选择呈现（见
  `WelcomePanel`），因此打开与“项目主页”手势收敛到同一个入口界面。
  打开操作经过共享的**`useOpenProject`** hook（被 `ProjectTree` **与** `WelcomePanel`
  复用，因此 Projects 视图与 Welcome 屏幕中的流程完全一致）：`project.open` 会在相同 id
  下重新激活一个已关闭的已知路径（或打开新路径），随后发起客户端选中 Project Home，
  而每个客户端都会收到 `project.updated`；失败时 `project.inspect` → 要么在其
  `initable` 时提议把该文件夹引导成仓库 —— 一个模态 **`ConfirmDialog`**（确认 →
  `project.init`）—— 要么在 **`NoticeDialog`** 中暴露错误 ——因此非 git
  文件夹绝不会静默无操作。原生选择器仍是本地快速路径，并保留其调高的超时，
  因为它等待的是人；若宿主无法呈现它，拒绝会转而打开一个**从宿主路径打开项目**对话框，
  携带原因和一个自动聚焦的路径字段。该对话框说明路径属于运行 ThinkRail 的那台电脑，
  接受宿主绝对路径或 `~` / `~/…`，并通过同一个 open/inspect/init 流程提交。**Enter host
  path…** 也始终存在于 `AddProjectMenu` 的 Open project 旁：
  远程客户端无法判断一次成功的原生选择器是否在某个看不见的宿主显示器上打开，
  因此恢复手段不能只依赖失败。每个打开手势都启动一个客户端级的最后意图代际，
  两个已挂载的 `useOpenProject` 实例共享它。该流程会在每次 picker、open、inspect、init
  与采纳等待之后复查该代际，
  因此任一界面上的手动路径或近期选择都会在旧流程选中项目或抛出陈旧对话框之前取代它。
  这些是 `@thinkrail/ui/dialog` 上的模态框（init 提议没有屏幕锚点，不像 Remove
  popover）；`NoticeDialog` 仍是不带站内恢复的单按钮失败界面。该 hook 返回一个 `dialogs`
  节点，由每个消费者渲染。**选中项目**（点击其行 —— chevron
  单独展开/折叠）**会取消选中任何活动工作区**，使 shell 回到该项目的 Welcome ——
  一个刻意的“项目主页”手势。两个选中项目的手势 —— rail
  行点击与采纳刚打开的项目（`ProjectTree` *与* `WelcomePanel`）——
  也会**揭示该项目的工作区**（`selectProject(id, { reveal: true })`）：
  进入项目的手势承诺其工作区列表，因此从 Welcome 屏幕打开时绝不会落在折叠的 rail 行上；
  工作区的前端本地视图通过 shell 布局持久化存活下来，
  因此重新选中它会在未变的框架内恢复该窗口的资源标签页。这次往返会卸载工作区界面，
  但终端没有客户端侧生命周期会丢失：宿主拥有每个标签页与 PTY，卸载不会杀死任何东西。
  多个不同终端可能在不同的 workbench 组中可见；shell
  布局的可见性门禁为每个本地选中的终端身份挂载一个 body，且不挂载非活动 body。
  `TerminalWorkbenchBody` 从 shell 接收其 New-terminal 回调，因此它保持排布无关，
  而居中放置可以捕获其所属组。宿主附着仍按身份全局独占，
  因此在另一个客户端选中同一终端会触发既有的 takeover/detached/reclaim 流程。
  终端目录水合带连接代际戳，其完整快照推送订阅在 `terminal.list` 之前建立：
  在读取开始后落地的推送会胜出，而 transport 同步重放的缓存推送则被正确视为读取基线。
  只有显式 `terminal.close` 会杀死 PTY，并带既有的 busy-shell 确认；
  确认强制关闭会保留活动请求直到它结算（对话框可以关闭，但第二个请求无法让它变成孤儿），
  失败会暴露给用户，而权威目录移除会关闭已变陈旧的确认，
  而不是留下一个指向已被其他客户端关闭的终端的模态框。另外 `FilesPanel`（**Files**
  工具：固定的面板头条行，带一对 **All | Changed** `ToggleSegment` —— 计数非零时
  `Changed` 携带变更文件数 —— 下面是 `FileTree` 或 `ChangesPanel`；该模式是应用级、
  仅会话的 `store.filesMode`，默认 All，且 `requestChangesView` / `revealChanges`
  用一次 store 写入同时揭示 Files 并选中 Changed。`FilesPanel` 拥有唯一的 `git.status`
  读取（`useGitStatus`，按 scope 加 key，包括 unknown-commit 的 scope 回退），并向
  `ChangesPanel` 与 **All** 的标记提供数据，因此计数、标记与 Changed 列表不可能不一致。
  两个 `git.status` 读取方 —— `useGitStatus` 与 `ChangesReviewPane` ——
  都通过 `gitStatusCache.readGitStatus` 读取，它以 *(workspace, scope, target ref)* 为
  key 记录每个结果；review 标签页从该条目开始，然后用自己的一次读取刷新它。这样，
  打开一个面板已读取过的 scope 会在首帧渲染其分区，而不是等一个 skeleton 往返。
  该缓存是有界 LRU（32 条），回复绝不覆盖来自更晚请求的条目，
  被移除工作区的条目会被丢弃，而 `UNKNOWN_COMMIT` 拒绝会忘记该条目，因此消失的 scope
  绝不会被播种。播种的状态在标签页自己的读取结算前是暂定的：一次 reveal、
  存储的活动文件，以及路径不在种子中的 Stacked 恢复，都会等待那次读取，
  而不是被丢弃或覆盖。**All** 遵循 VS Code 的 Explorer：朴素树加上由纯函数
  `changesModel.buildChangeMarks` 一次性推导的标记 ——变更文件显示其名称加上尾随的
  `M`/`A`/`U`/`R` 字母，使用状态色（modified 琥珀色、added/untracked 绿色、renamed
  蓝色；唯一的 `chat/planView.statusLetter`，与 Changes review 标签页和 plan 行共用，
  因此 untracked 处处读作 `U`、added 读作 `A`）；每个祖先文件夹，无论展开还是折叠，
  都显示其名称和一个处于同一种琥珀色“包含变更”色的点（整行作为紧凑行）；
  已删除文件既不显示也不标记文件夹，与 VS Code 一致。Git 忽略的条目（宿主的
  `FileNode.gitignored`—— 对紧凑行来说，即它所代表的最深目录的那个属性 ——
  及其内部的一切）以 `text-subtle` 渲染。活动文件或 diff
  标签页（`selectActiveFilePath`）获得选中行的观感，并且当它在 All 显示期间变化时，
  其文件夹展开（并保持展开）且它滚动 `nearest` 进入视野；返回 All 时，
  仅当活动文件与持久化的 `seenActivePath` 不同才揭示，且此时跳过滚动恢复，
  使两者不会互相争夺。自动揭示绝不切换侧边标签页或模式。点击仍然打开编辑器。
  首次状态读取失败时不显示任何标记；刷新失败则保留上一批标记，与 Changed 列表一致。
  树的观感按工作区跨重载记住（`store.filesTreeByWorkspace`：All 的已展开文件夹、Changed
  树在全部 scope 间共享的已折叠文件夹、通过 `RememberedScrollArea` 的按视图滚动 —— 一个
  Changed 视图的滚动属于其 scope 与比较目标，因此改变任一项都会从顶部开始），由
  `filesTreeState` 模块持久化（优先原生稳定偏好，否则使用带宿主限定 key 的 localStorage；
  从 `main.tsx` 水合，防抖写入外加一次 `pagehide` 冲刷；窗口之间共享，最后写入者胜出，
  无实时同步）。切换行右侧的一个 **Collapse all folders** `HeaderIconButton` 折叠 All，
  或者 —— 仅在 Changed 的 Tree 视图且至少有一个变更时 ——折叠所有 changed-tree
  文件夹（`changeTreeFolderPaths`）），`FileTree`、`SpecsPanel`、`ReviewPanel`、
  `ChangesPanel`（变更文件位于固定的**面板头条行**之下 ——
  `h-panel-header-row`（`--panel-header-row-height`，当前 28px），与 workbench Group
  Headers 和聊天头部共享结构几何，而不是在此固定某个值 —— 该行通过
  **`ChangesScopeMenu`** scope 药丸 + 共享 **`BranchPicker`** 目标分支药丸说明 diff
  的是**什么**，外加 **List | Tree** 切换（`store.changesView`，应用级），
  在扁平列表与文件夹**`ChangesTree`** 之间切换；在任一视图中单击文件都会在该 scope 的
  **Changes review 标签页**中打开/揭示它（双击或 *Open as tab*：它自己的 diff 标签页 ——
  即下面的两种呈现规则），并且每个文件行都携带共享的**`ChangeRowActions`** 菜单。
  行包装器绘制完整的悬停/选中色带，包括尾随菜单槽位；其内部打开文件的按钮保持透明，
  因此该色带看起来不会在菜单之前被裁切），`FilePane` 与 `DiffPane`
  作为资源注册表分派器，指向 [[module-resource-renderers]]（打包的惰性渲染器、Monaco
  的桌面文件管道与 Pierre 的 provider 都在那里），外加惰性的 `TerminalInstance`。
  细头部的视图切换段（有序资源候选、`Split|Inline`、`List|Tree`）是共享的
  `@thinkrail/ui/toggle-segment` `ToggleSegment` —— 其活动段复用标签页语法的
  `control-bg-selected`（见下文），而绝不是容器表面，因此在高对比主题下、当
  `container-elevated-bg` 与工具栏表面重合时，选中填充仍然可见。`ChangesPanel`
  的次级工具栏**不绘制自己的表面**：与右侧面板标签条一样，它显示面板的
  `container-sidebar-bg`，使两行外壳看起来是一块连续表面。
  **文件风格树行**（chevron/占位引导、文件夹/文件图标、截断标签、尾随槽位；`min-w-0`
  使行在与尾随控件共享 flex 行时仍可收缩）是共享的 **`TreeRow`**，被 `FileTree` 与
  `ChangesTree` 同时使用，因此两棵树保持完全一致。`TreeRow` 可选的 `onWarm` 报告
  `hover`（指针进入）或 `focus`。`FileTree` 的文件行在 focus 时立即预热，并通过
  `warmOnHover` 在悬停时预热 —— 它只保留一个待处理行，并在指针停留 150 ms 后触发，
  因此扫过整棵树不会加载任何东西。预热通过 **`useEditorWarm`**（`editorWarm.ts`，也用于
  shell 的空闲预加载）预加载代码视图与该行的 grammar：它返回 `usePreloadResourceView`
  的回调，或在手机级视口和 `saveData` 连接上返回 `undefined`。`editorWarm`
  还拥有启动门禁：一个 `FileTree` 在其首次列表落地前持有 `holdEditorWarm()`，而
  **`whenFilesTreeIdle`** 仅在没有树在加载时于空闲期运行 shell 的预加载。
  被搁置的任务会等待释放，然后等待一个新的空闲期，因此 Monaco chunk
  绝不会在树的首次绘制中求值。两棵树都会**把单目录链压缩成一行、
  以斜杠连接**（`apps/web/src`）：只有当某目录恰好有一个子项、
  且该子项也是目录时链条才继续，紧凑行把最深目录作为一个单元展开/折叠。`ChangesTree`
  针对变更文件树评估这一点；`FileTree` 仅通过其既有的客户端目录读取解析可见的紧凑链，
  因此线上契约仍是朴素的即时目录列表。**`+N −M` diff 计数徽标**是共享的
  **`DiffStatBadge`**，仅在 Changes 内部使用：扁平列表的文件行、树的文件行，
  以及树文件夹的聚合（仅在其折叠时；展开文件夹的子项已显示这些计数）。`ChangesTree`
  的树构建 + `+/−` 聚合 + 共享状态字形位于纯 **`changesModel.ts`** 中（有单元测试；不含
  store/transport ——`ChangesTree` 是展示型的，由 `ChangesPanel` 提供 `changes` +
  `onOpen`/`isActive`），同时还有**diff 标签页身份 + scope 词汇**：`scopeKey` /
  `diffTabId(workspaceId, scope, path)` / `diffTabName` / `scopeLabel` 以及
  `splitPath`（用于扁平列表的路径行与 diff 头部的路径 chip）。**分支组合框**是共享的 **`BranchPicker`**（可搜索、按
  Remote/Local 分组、当前选择带对勾、每次打开都刷新且另有一个显式 Refresh 控件；
  没有远程分支的仓库 —— 一个全新的**Start a new project** 仓库 —— 会把触发器标为 “·
  local”，并在头部说明其分支只存在于本机，因此裸的 `main` 不会被误认为 GitHub 分支）——
  一个组件同时用于 New-Workspace 对话框的*基础*分支和 Changes 头部的*目标*分支。
  除非调用方为自己的药丸族传入 `triggerClassName`（New-Workspace 对话框的 composer
  风格药丸），其触发器是 `Button ghost m`（分支图标、标签、chevron），因此 Changes
  头部的 scope 与目标触发器看起来是一对。**Remote 分两层**：一个 `Remote` 父项，
  下面按宿主标识的 remote（`origin`、`upstream`）各一个子组，其行显示分支名而不重复
  remote 名。完整 ref 仍是每行的选择身份、搜索值与 `data-branch`；浏览器从不拆分
  `remote/branch`，因为 Git 允许 remote 名中包含 `/`。未配置的跟踪 ref 位于 `Other` 下，
  并保留完整 ref 作为行标签。`BranchList.remoteGroups` 是增量字段：面对省略它的旧宿主，
  选择器回退为一个由完整 ref 组成的扁平 Remote 组。分组路径使用嵌套的 cmdk 组；
  强制挂载的父项只在 cmdk 隐藏了每个子组时才隐藏。一个 fork 会使用两个 remote，
  因此只显示 `origin` 的列表会隐藏它分叉自的 ref。围绕它的整个状态 —— 列表、
  `refreshing`、`refresh()` —— 是共享的**`useBranchList(projectId,
  onLoaded?)`**（`branches.ts`，基于可离线降级的 `listBranchesOrEmpty`；
  它还会为降级的首次读取报告 `failed`，因此选择器绝不会把不可读的仓库说成“仅本地”），
  两个选择器因此**在构造上**完全一致：列表**以项目为 key**（项目切换时清空，
  且两次读取都带代际戳，因此切换时绝不会提供或落到上一个项目的分支）、
  **只有首次读取会降级**（*refresh* 在瞬时失败时保留上一次的良好列表，
  而不是把选择器清空），并且 `refreshing` 始终驱动 spinner。
  首次加载的预取总是把非空默认值提供给宿主，宿主才是判断它是否命名了已配置 remote
  的权威；这样既能避免陈旧或缺失的默认跟踪 ref 卡在 create 的关键路径上，
  又无需读取尚未渲染的分支状态。手动选择只预取已加载 remote 列表中的行。`null`
  projectId 不读取任何东西 ——已关闭对话框就这样暂停。其降级默认值是 `defaultBranch:
  ""`，**绝不是字面量 `HEAD`**：一个命名了某个 ref 的哨兵会被采信 ——
  对话框会预选它并将其持久化为工作区的 `baseBranch`，而那个工作树将永远与其自己的 head
  做 diff。空表示“未知”，因此 `create` 省略 `baseRef`，由宿主解析真实分支。
  **`WelcomePanel`** 是 shell 在没有活动工作区时挂载的首触界面（居中，
  左侧导航在其旁边）。**一个主标题**（`welcome-title`，顶栏的品牌样式 —— 强调字体、
  `text-primary` —— 放大）：**所示项目的名称**，或没有项目时用 `PRODUCT_NAME` ——
  字标是空状态身份，项目打开后其自身名称即身份（因此没有单独的项目眉标）。
  唯一的例外是连接后的交接：在没有项目时从 Welcome 横幅连上 provider 之后，主标题显示
  **“You're set”**，其下是向导结束时使用的同一条 `ConnectedLine`（“Anthropic connected ·
  the agent can run now”；toast 携带计数），因此屏幕明显变化，而不是把用户退回同一面墙。
  **此外没有推销文案** —— 营销段落因无人阅读而被移除；屏幕顺序是标题 → 横幅
  →**一到三张卡片**（图标左上、标签 + 说明左下；主卡是带稳定 `welcome-cta`
  钩子的填充主色卡片，其余是安静的 `welcome-action`）。Welcome 是**模式分叉点**：
  有项目时它总是把 **“Start building”**（隔离工作树）与 **“Work in project
  folder”**（Default 工作区）配对，使两种工作模式成为可见选择而不是隐藏默认。
  按状态的卡片：**无项目** → **“Start a new project”**（主，`welcome-cta`）+ **“Open
  project”** ——或面对低于 `ONBOARDING_PROTOCOL_VERSION` 的宿主时 “Open project”
  单独作为主项；**有项目 + `hasSpecs`** → **“Start building”**（主）+ “Work in project
  folder”；**有项目 + 无规格** →规格先行的 **“Set up project”**（主）+ “Start building”
  + “Work in project folder”。**“Open project” 只出现在无项目状态** —— 在那里，它与
  “Start a new project” 是唯一可能的动作；一旦有项目显示，打开另一个就是项目 rail 的
  **“+”**（同一个下拉），因此 Welcome 始终是*在此项目中工作* 的界面。那张卡片挂载共享的
  **`AddProjectMenu`** 下拉（与项目 rail 的 “+” 同一菜单：Start a new
  project…（当调用方传入 `onStartNew`，即宿主支持 `project.create`）/ Open project /
  Enter host path… / Clone repository…（来自 `PROJECT_CLONE_PROTOCOL_VERSION`）/
  Recents）。Recents 是 store 的 `recentProjects`：一个最近打开路径列表，包含打开 +
  关闭记录，无状态徽标（服务器会省略文件夹已消失的关闭记录）；
  选中任一项都会运行共享打开流程并落在 Project Home，关闭记录保留其 id 与工作区状态。
  `Card` 是 `forwardRef`，可用作 Radix `asChild` 触发器。**“Work in project
  folder”**（`House` 图标，与 rail 的 Default 行一致）**直接进入** Default 工作区 ——
  无对话框：共享的 `enterDefaultWorkspace` 助手列出项目的工作区、存入 store，并激活
  `kind === "default"` 行；没有 Default 行的旧宿主降级为错误 toast。**“Start
  building”** 是创建并启动流程的意图优先表述 ——它打开预选到 **Isolated workspace**
  目标的 `NewWorkspaceDialog`；*工作区* 是机制，不是标签。**“Set up project”**
  打开同一对话框，带一个 `initialPrompt` 种子**和一个 `promptNote`** ——
  该注释是卡片自己的文案（对话框保持 skill 无关），说明种子命令做什么：agent
  在构建之前先从目标出发起草项目的规格 ——刻意**不**枚举产物，
  因为分派器的路由各不相同（starting-a-new-project 停在 goal-and-requirements；只有
  importing-a-codebase 才起草架构 + 模块 SPEC），而卡片事先无法知道路由。
  种子是 `/skill:setting-up-a-project` 命令**带一个尾随空格** ——
  与斜杠命令补全写入的插入格式相同（`chat` 的 `selectedSlashCommandValue`），
  因此种子后的主区读起来像一条*已完成*的命令，补全菜单在其上保持关闭（pi
  的解析器把参数尾部视为可选）。该命令**强制**加载 setting-up-a-project 分派器
  skill（pi 的 skill 命令语法；在 `session.prompt` 路径上展开），
  而不是指望模型自动匹配它；分派器随后检测新仓库还是既有仓库，并据此起草规格（见
  [[module-thinkrail-workflow]]）。**每个 Welcome 入口都预选 Isolated workspace 目标**
  —— 包括 setup，因此规格起草与任何其他工作一样可在自己的分支上评审，
  模式叙事也保持统一；Project-folder 备选仍在对话框中一键可达。（这种统一使由 opener
  选择目标成为死 API —— 对话框拥有自己的目标状态，且总在工作树一侧打开；没有
  `initialTarget` prop。）驱动 has-specs 状态的项目 = `selectedProjectId ??
  projects[0]`，以响应式方式读取（因此可见导航的选择会更新它）。其 `hasSpecs` 通过
  `project.hasSpecs` 为该单个项目**惰性抓取**（一次全树遍历，避开连接握手）——
  在解析前保持 pending，因此卡片会等待它。打开项目的编排位于共享的 **`useOpenProject`**
  hook 中（见上文），因此 Welcome 的 “Open project” 卡片获得与 rail 相同的非 git
  init/通知处理。在卡片之上，`WelcomePanel` 组合 **`ProviderWarningBanner`** ——
  一条纤细的金色横幅，**仅在未连接任何 provider 时**显示（“No model provider connected
  — the agent can't run”），其 **Connect a provider** CTA
  打开聚焦的**`ConnectProviderDialog`**（见下文），而不是完整的 Settings 接管。Welcome
  传入 `hasProvider`，并在向导门禁仍为 `pending` 时不渲染它，
  因此它绝不会闪现在向导之前。**Provider 状态只有一个所有者**：store 的
  `providerStatus` 快照，并带有它所回答的 `providerStatusVersion` 戳。store 在每次
  `provider.changed`、每次
  welcome（断连期间完成的登录会在重连时被拾取）以及每个成功的非定向登录帧 ——
  包括其他客户端的，即使本客户端自己的登录仍占用对话框（早于推送的宿主仍会刷新）——
  时递增该版本；本地登出或 Retry 也会递增它。`useProviderStatus` 在快照落后时重读，
  按版本单飞，而 `useHasProvider`（当 `reportHasModelSource` 时为已连接：任何标准
  provider `configured` **或** JetBrains Central 已连接，因此仅有 Central
  不会触发警告）从中派生，所以横幅、向导门禁、连接界面与 Settings 永不矛盾。
  `useHasProvider` 还在挂载时失效一次，因此回到 Welcome
  能看到宿主从不推送的凭据变化（`pi` CLI 的登录或登出）。
  横幅在第一个答案落地前保持隐藏，传输层错误降级为*不*唠叨（未知或离线 ≠ “没有
  provider”）。所有 provider **管理**都在 Settings 中，不在这里（常驻条已移除）。

**首次连接。** `ConnectProviderDialog` 是覆盖在 `ConnectProviderOptions` 之上的引导遮罩
——与引导向导内嵌的正文相同：一块带边框的表面，其中 **Detected** 区块 ——
仅当账户检测发现内容时（见下文）——位于搜索框之上，然后是**一个可搜索列表**，
列出其他所有未配置 provider，置于 240px 高的滚动区中，按视口收缩到 160px 下限，
因此矮窗口先损失列表高度再让页面滚动（约 40 个 provider 从不撑大遮罩，
过滤也从不改变其尺寸）：订阅（支持 OAuth）provider 在前，然后仅 key 的在后，每行是
Settings 的 `ProviderRow`，按钮与 Settings 相同：provider 支持处给 Sign in，接受处给
API key，因此文案里的“或添加 API key”对所有 provider 成立。JetBrains AI 不是 pi
provider，而是配置多个 provider 的 `central` CLI，因此它保留真正的 `JetBrainsAiCard`（其
Central 状态机从不重复实现）—— 像其他选项一样放置：只要装了 `central` 就放在
**Detected** 中（按状态给 Connect、登录、启动 proxy 或更新），否则在列表末尾作为普通
**JetBrains AI** 行，其 **Set up**展开卡片的安装指南。未过滤列表跳过已检测的 provider；
搜索则覆盖所有 provider，包括它们。用单个列表而不是分开的订阅 / API-key 桶，
因为用户按名称找 provider，而不是按认证方式。**Detected 是靠证据赢得的，绝不硬编码。**
该界面首先询问宿主 `provider.detect`（`detectAccounts`；
旧宿主或失败视为未发现任何东西）—— 向导在 **Get started** 时、遮罩在打开时 ——
并在它回答前保持 skeleton，因此没有跳变。每个 pi provider 未配置且支持 OAuth
的已检测账户都会成为一个 Detected 登录项（例如 Claude Code 已登录时的 “Sign in with
Claude”，以该工具的计划作为徽标），它会启动该 provider 的普通 OAuth 流程，而那些
provider 会离开列表。当没有可用内容被发现时，完全没有 Detected 区块 ——
点名一个我们一无所知的 provider 就是把猜测装扮成建议。它是建议，绝不是静默登录：
宿主报告哪些工具已登录，用户仍需通过 pi 自己的流程授权 ThinkRail。这里没有跳转到
Settings 的链接 ——该界面已列出所有可连接 provider。此处不出现配额或其他全局设置；完整的
`ProvidersSettings` 仍是高级用户界面。数据与登录接线是共享的 **`useProviderAuth`**
hook，`ProvidersSettings` 也使用它：store 的 provider 状态快照（上文）、
任一界面打开时重读一次 —— Central 的安装状态与类似宿主本地事实会在无推送的情况下变化 ——
`startLogin` / `logout`，以及 store 驱动的 `LoginDialog` 节点。恰好一个界面挂载该节点：
Settings 打开时为 Settings → Providers，否则为连接界面（`ownsLoginDialog`），
因此登录绝不会渲染两个叠放的对话框。**连接后交接：** 当一个在该界面打开时未配置的
provider 变为已配置 —— OAuth / API-key 登录或 Central 连接都一样；
该界面的首次报告就是那个基线，即使其重读仍在进行中（等待一个当前值会吞掉先落地的登录），
除非它传入 `configuredAtStart`，向导以空值这么做（冷启动没有 provider），因此在 intro
或 privacy 步骤期间通过 Settings 连上的 provider 在到达 Connect 时仍会被验证 ——
`ConnectProviderOptions` 通过 `onConnected` 报告一次，且只来自当前、
成功的快照（绝不在重读进行中或失败之后，因此在 Get started 之前一刻登出的 provider
不会被验证），然后由宿主界面决定：`ConnectProviderDialog` 清除该 provider
已完成的登录（绝不是更新的那个），关闭自身，并抛出一次成功 toast，命名该 provider 与
`model.list` 计数（“JetBrains AI connected — 23 models available”），随后让 Welcome
把主标题切换为 `ConnectedLine`；向导既不 toast 也不关闭 —— 它就地验证（见下文）。
Settings 保持自身行为（不自动关闭），因为那里的高级用户可能正在添加第二个 provider。

**从零开始。** “Start a new project” 打开 **`CreateProjectDialog`**（由
`useOpenProject` 的 `startNewProject` 驱动，因此 rail “+” 与 Welcome 共用它）：
一个项目名、其推导出的文件夹名，以及一个**Location** —— 父文件夹，默认 `~/thinkrail`，
可作为宿主路径编辑或用原生目录选择器挑选 ——以最终路径预览，然后
`project.create`（文件夹 + `git init` + 初始提交 + 打开；没有“跳过 git”选项，
因为非仓库不能成为项目）。一个 **Draft the specs with the agent**
开关（默认**开**）让调用方（`startProjectSetupChat`）进入新项目的 **Default 工作区**，
并打开一个全新的聊天，其 composer 已预填 —— 但不发送 ——`/skill:setting-up-a-project`：
全新仓库没有可隔离的东西，因此第二个 Create-workspace
对话框只会在任何东西存在之前增加点击。该续接以导航为栅栏：只有当新项目的 home
仍是用户正在看的内容（已选中、无活动工作区）时它才继续，且只有当那个 Default
工作区仍是活动工作区时才启动聊天，因此在 `workspace.list` 期间走开就会丢弃它；
列表读取失败会 toast 而不是拒绝。既有仓库上的 “Set up project” 卡片仍走
`NewWorkspaceDialog`。失败保持内联在对话框中。

**克隆仓库。** `CloneProjectDialog` 由 `useOpenProject` 从任一 add-project 菜单打开，
接受 Git URL、可编辑的由 URL 推导的文件夹名、宿主父路径（默认 `~`，可编辑或用 Browse
选择），以及可选的正数历史深度（空 = 完整历史）。`project.cloneDestination`
提供绝对路径预览并拒绝已存在的目标；只有与当前字段匹配的预览才启用提交，
克隆在写入前会再次校验。该对话框使用宿主 Git 凭据，而不是 GitHub 登录。
克隆期间字段被锁定、状态被朗读，Cancel 请求 `project.cancelClone`；
原请求只在进程退出并清理完成后才结算。Cancel 返回到可编辑表单并保留已填值。
错误保持内联；清理失败明确要求手动移除。关闭空闲对话框会关闭它，而
Escape/背景点击无法关闭进行中的克隆。卸载会请求取消。重连会重放待处理操作。
成功通过与打开本地项目相同的最后意图栅栏采纳 Project Home，且不启动聊天。
迟到的结果无法抢走更新的打开项目选择。

**冷启动引导向导。** `OnboardingWizard` 仅在真正的冷启动时替换 Welcome 卡片：
宿主支持 `ONBOARDING_PROTOCOL_VERSION`、`AppConfig.onboardingDismissed` 为 false、
没有打开或近期项目，且 provider 状态快照报告无 provider。该决定**锁存于 store
门禁**（跨重挂载存活），因此在向导中途连接 provider 不会拆掉它。它以 **intro**
开场（品牌标题、落地页标语及其三个“why”要点 —— 工作树、规格、pi 作为引擎 ——作为小卡片，
以及 **Get started**；文案跟随 thinkrail.ai，使首次运行与落地页说法一致）——
首次使用的用户先见到产品，而不是一堵 provider 墙。然后，仅在分析同意未确认时，有一个
**privacy** 步骤（见下文 `AnalyticsConsentDialog` 部分）；然后
**Connect**（`ConnectProviderOptions`）；一旦某 provider 变为已配置，登录对话框关闭，
一个加载中的**验证**就地覆盖选项（同一个框，无尺寸变化）—— 不消耗 token，只做
`model.list` 加 `model.default` 读取。`model.list` 在后台刷新目录的同时提供缓存快照，
因此空的首次读取会回退到一次等待中的 `model.refresh`（同一刷新，此时被等待）再作决定 ——
刚连上、目录仍在预热的 provider 会验证通过而不是警告。非空列表（或不可读列表：
连接本身已成功）把向导推进到 **done**，没有 Done 按钮，作为屏幕切换而非尺寸变化：
整个连接屏幕（标题、选项、Skip）在原地 200 ms 内淡出，然后 done 屏幕淡入 —— “You're
set” 主标题、一条 `ConnectedLine`（“Anthropic connected · Claude Fable 5 is the default
model”）以及 Welcome 自己的两张卡片（Start a new project / Open project，作为
`projectCards` 传入，使其菜单与对话框仍归 Welcome）。没有任何高度动画，
因此没有列表在折叠中途被裁切，居中列只在其不可见时重新居中；减少动效时切换是即时的。
到达 done 会持久化 `onboardingDismissed`；向导（连同隐藏的 rail
与被暂缓的提示）保留到有项目打开。仍然为空的刷新会警告并提供 **Check
again**（重跑两次读取，例如在别处修好账户之后）以及 Back 回到选择 ——仅 Back 无法返回，
因为 provider 已配置因此不再被列出。向导没有自己的项目步骤，也不显示进度条。**Skip
setup** 出现在 done 之前的所有屏幕，除 privacy 外（其 Continue 是唯一前进方式）；
跳过还会取消对话框中仍在待处理的无目标登录（`cancelUnownedLogin`，与
`ConnectProviderDialog` 共用）。登录由其请求发起的 start 拥有（`startReservedLogin`）：
当界面在 `provider.loginStart` 回答前关闭时，迟到的 `loginId` 会被取消并清除，
而不是隐藏运行。门禁位于 store（`onboardingWizard`：pending → shown | hidden），使
shell 在未决时暂缓首次运行提示。完成或跳过会写入 `onboardingDismissed: true`，
因此回访用户 —— 即使之后移除了所有项目与 provider —— 得到常规 Welcome；
写入失败仍会在本地关闭向导。

在其下方，**`ProjectTrustNotice`**
是工作区前的信任界面（因此即使尚无工作区也能到达信任）：**存在性门禁** ——
除非所选项目带有信任授予会加载的内容，否则不渲染任何东西。
存在性来自 `chat/useProjectTrustSummary`（在 `PROJECT_TRUST_PROTOCOL_VERSION`
及以上的宿主上为 `project.trustSummary`，由 store 的 `selectSupportsProjectTrust` 门禁：
别名 skill 名、pi 自身需要信任的资源类型与项目子 agent 定义，以及 `.pi/mcp.json`
条目计数；旧宿主只回答 `project.aliasSkills`，读作“仅别名”），
以及纯函数 `chat/projectTrust` 推导 —— 与 `chat/SkillsDialog` 的信任横幅和 New Workspace
共用，因此三个界面对一次授予会加载什么不可能有分歧 —— `deriveProjectTrustNotice`
**分别**评估别名授予（`Project.trusted`）与 pi 级信任（`Project.piResourceTrust`），
因此通知只列出仍未开启的内容：“ships N skills and its own Pi resources → *Trust
project*”，外加 —— 当 pi 级资源被门禁时 —— 授予会加载的确切类型（摘要的 `piResources`：
skills、prompts、themes、system prompt、subagent definitions），并且仅在 `extensions` 或
`settings` 处于其中时说明它们会在本机运行代码，以及其 N 个 MCP
服务器仍需逐个批准（一个唯一被门禁内容为 `.pi/mcp.json` 的仓库仍显示该通知，其授予请求
pi 级信任）。*Trust project* 发送由 `trustGrantParams` 构建的 `project.setTrust`：
仅当通知列出 pi 级资源时才 `resources: true`（因此也说明了它们加载与运行什么）；
仅别名的通知只授予别名信任。信任之后，它为新出现的 skill 显示“N new → *Review &
enable*” 状态（`project.acknowledgeSkills`），否则显示一行安静的 “trusted”。*Stop
trusting*（位于破坏性 `ConfirmDialog`
之后）在任一授予被持有的所有状态下显示（`canStopTrusting`），不仅是完全受信之时 ——
添加时 pi 已信任的项目，或只部分受信的项目，都保留撤销路径 —— 并发送 `project.setTrust({
trusted: false })`，撤销两个授予；其确认文案（`stopTrustingText`）说明 pi 资源停止加载、
运行中的子 agent 停止，并且仅当 `piResourceTrust` 为 `granted`
且宿主支持项目信任时打开中的聊天才会重载，否则说明该项目 skill 在新聊天中停止加载。
它在信任前从不渲染 skill 的（攻击者可控的）名称。完整管理器（`chat/SkillsDialog`
的**项目模式** —— 信任 + 组/skill 开关，尚无会话）从 **New Workspace** 到达，其 opener
是共享的 `chat/SkillsButton` 原语（因此不会与聊天头部的 Skills 触发器漂移）。这是用户
skill 设置的会话前一半；聊天头部在 workspace 模式下打开同一对话框（带 Reload）。

**`NewWorkspaceDialog`** 是开始工作的界面：**一个目标控件**（两选项分段 —— 原生单选组，
`fieldset` + 视觉隐藏单选输入之上的 sr-only `legend`，因此辅助技术听到的是一个互斥选择
—— 两者始终可见：一眼看到双模式模型）选择工作**在哪里**运行，而头部是**模式感知**的，
因此总能如实命名操作：**Isolated workspace** → 标题**“Create workspace”**，描述 **“A
separate checkout on its own new branch. Files, chats, changes, and terminals stay
scoped to it.”**；**Project folder** → 标题 **“Work in project folder”**，描述**“Runs
directly in your project folder — no isolation. Changes land on the current
branch.”**在 folder 模式下，基础分支选择器与命名提示被隐藏（不创建任何东西 ——
提交通过共享的**`enterDefaultWorkspace`** 助手**进入**项目的 Default
工作区（`defaultWorkspace.ts`：`workspace.list` → 折入 store → 激活 `kind ===
"default"` 行，一次性原子进入 —— rail 的自动展开跟随激活；旧宿主没有该行时错误 toast +
`null` —— 与 Welcome 分叉卡片背后是同一助手，因此进入 + 降级路径只存在一处；
**`onCreated` 不触发** —— 没有创建任何东西，且助手的列表已是新的）），提交按钮读作
**Start** 而不是 **Create**；分支列表抓取 + 后台 base 预取仍会运行（fire-and-forget，
保持切回工作树时即时）；聊天启动尾部在两种模式下相同。可选的 **`promptNote`** 渲染为
prompt 上方的小 info 条（被 “Set up project” 用于说明种子 skill 命令做什么）。
工作树模式的基础分支触发器读作 **“From {base}”**，而不是未解释的 ref。可选的
**`initialPrompt`** 播种 prompt 主区（仍可编辑；默认空）；当 prompt
非空时（工作树模式），一条次级提示说明 ThinkRail 将根据请求为工作区与分支命名。
其余保持紧凑：基础分支组合框（`git.listBranches`，离线时降级为本地分支；Refresh
重新列出；`origin/HEAD` 被过滤以免出现游离的 `origin`）、项目选择器、prompt 主区，
以及复用的 `chat/ModelEffortPicker` 的**会话前**模式。它按**宿主默认**打开：
药丸读作 `Default · ‹model› · ‹level›`，来自宿主的 `model.default`
结果（可用时为保存的默认值，否则为宿主偏好的可用模型），且 popover 的 Default 行被勾选。
任何显式选择（模型、级别或两者）都会把对话框翻转为**显式配对**；Default 行 ——
以及下面不可用模型的对账 —— 会**同步地**让它回到跟随，因此刚选完 Default 就按 Create
已经省略该配对；随后的 `model.default` 读取只刷新显示的配对。
任何进行中的此类读取都会被显式选择取消，
因此用户选择之后落地的回复绝不覆盖选择（它仍刷新 Default 行所显示的内容）。Create
仅在显式配对时发送 `{model, thinkingLevel}`，在**跟随时两者都省略**，
因此宿主解析器在创建时决定，显示也永远不会快照一个期间被 Settings 改过的默认值。
没有可用模型时宿主返回 `model: null`，药丸显示裸 Default。对话框不选择竞争性的默认值：
其显示与新建会话共享宿主解析器（见 `submodule-server-agent`）。收藏/最近项通过
`chat/useModelPreferences` 到达，即 composer 使用的同一接缝。选择器的 popover 通过
portal 进入对话框节点（因此其列表在 Dialog 滚动锁下滚动）。它们的目录是共享的那一个 ——
`chat/useModelCatalog`，因此对话框与聊天 composer 不会漂移 ——这也意味着它是**实时的**：
选择器的 Refresh 行可以在持有选择之下替换列表。
因此对话框在每次变化时通过纯函数**`reconcileModel`** 对账所持模型（仅模型 ——
力度由下面宿主的 clamp 决定）：重指向同一个 `{provider,id}`（刷新后的对象，其
`thinkingLevels` 可能不同）。当目录中没有该模型时它的行为取决于**`catalogFresh`** ——
store 的 `modelsFresh`，仅对宿主报告 **`complete`** 的、被等待的强制刷新的安装结果才为
true（有上限的等待可能回答一个当前但未结算的列表，那不是下判断的依据），
在下一次任何消费者的 `model.list`
安装时丢弃（该消费者的处理器从它所启动的分离刷新之前的状态回答）*并且*在任何消费者激活时
提前丢弃。在新鲜目录上它返回**`"unavailable"`** —— 一个判定，而不是替换：
对话框随后通过**一个** `applyHostDefault`
请求**`model.default`**（宿主保存的默认值或首个可用回退，加上其一致的力度），
与预选时完全一样 ——因此这里不存在宿主默认策略的客户端副本。每次打开最多询问一次，
因此仍缺失的模型无法让 effect 打转。力度是另一件事：一个 effect 通过向宿主请求 pi 的
clamp（**`model.clampThinking`**）而不是本地判断，使所持力度在所指模型上仍可运行，
因此显式切换与让模型集合缩小的刷新会以与 pi 相同的方式解决。`model.default` 无需调整：
宿主已返回自洽的配对。在打开与项目选择器变化时，对话框读取
**`skill.list({projectId})`**；每当出现前导斜杠 token
时它还会读取**`template.list({projectId})`**。它把两者送入共享的 `prompt` 模块，因此
Create Workspace 与实时聊天使用相同的过滤、菜单、键盘导航、避免竞态的模板选择，以及 Tab
遍历占位符状态机。skill 来自所选项目的**当前检出**加上个人/打包来源；模板合并全局 +
当前检出项目作用域，项目优先。选择 skill 插入 `/skill:<name> `；选择模板读取
`template.get({projectId, name})`，用其正文替换完整草稿，并激活其占位符。Up/Down 导航，
Enter/Tab 选择，Escape 关闭菜单；在菜单未打开时 Tab/Shift+Tab 循环活动模板槽位，Escape
结束该会话。提交会镜像编辑过的重复槽位并在最终文本成为首个 prompt 前移除未触碰的标记。
当所选模板正文仍在加载时，提交被暂缓但 prompt 仍可编辑；编辑会取消延迟应用。
切换项目也会使进行中的选择失效，因此来自上一个检出的响应无法填充下一个项目的 prompt。
首个 prompt 在异步工作区创建开始前被快照。
列表/获取失败会保留草稿并降级到仍然可用的那个来源。扩展命令与 `/compact` 保持缺席，
因为没有实时会话。prompt 下方的一行说明把目录标为**来自当前检出**（若所选基础分支不同，
则创建出的工作树会话才是权威）。当所选项目带有任何仍未**受信**的内容时（与 Welcome
通知相同的 `useProjectTrustSummary` + `deriveProjectTrustNotice` 组合 ——计数与类型，
绝不含名称），一个**信任通知**显示 *Trust project* 按钮 —— 仓库的 skill 与 pi
资源在授予前保持扣留（来自同一 `trustGrantParams` 的 `project.setTrust`，
因此仅当通知列出 pi 级资源时才携带 `resources: true`；它把更新后的项目折回 store
并重新预览）；个人 + 打包 skill 无论如何都会显示。当菜单关闭时，**Enter
提交**（与提交按钮的 `↵` 提示一致），并且**Shift+Enter** 插入换行。工作树模式的提交 =
`workspace.create({ projectId, baseRef })` → 设为活动，对话框自身展开项目并刷新其权威
`workspace.list`（fire-and-forget；不依赖 `workspace.created` 推送，
因为未加载的项目列表会丢弃它）—— 没有 `onCreated` 回调，因此每个挂载点（`ProjectTree`、
`WelcomePanel`、shell 键盘打开的实例）都得到相同的创建后折叠 →
**总是打开一个全新聊天**（`session.create({ workspaceId, model?, thinkingLevel? })` ——
所持模型 + 力度即使没有 prompt 也会应用，并且一起传递；
没有所持模型时宿主仍应用保存的/回退的默认值，只要模型可用就显式传入解析出的配对）→
输入的 prompt **以及任何附加图片**另外作为首条消息发送（fire-and-forget `prompt`，把
`images` 与 `text` 一起转发，并带一个携带相同附件的乐观用户轮次）。prompt
主区通过共享的 `usePromptImages` 控制器接受**图片粘贴/拖放**（与聊天 `Composer` 相同的
`imageAttachment` 解码/降采样 + chip 表面）；只要还有图片在解码，提交就**被暂缓**，
而**只有图片没有文本**的启动也是有效的 kick-off。**空 prompt 且无图片会让刚打开的
composer 保持就绪** ——提交开始工作界面总是把用户落在聊天中，绝不是一个裸回执（folder
模式：进入 Default 后同样的尾部）。被**拒绝**的 kick-off `prompt`（错误的模型 / 缺失的
API key —— 例如选了不存在的模型）会通过 `store.appendErrorTurn` 作为 `error`
轮次出现在刚打开的聊天里（带 `transport` 的 `errorText`），而不是消失。
两个**没有聊天可承载轮次**的拒绝改为抛出 `store.toast.error`：失败的
**`workspace.create`**（保持对话框打开以便重试）和失败的
**`session.create`**（对话框已关闭、工作区已存在 —— toast 是唯一还剩的上报位置）。
（`gh` 状态位于 `SettingsDialog`，不在创建对话框中。）**`SettingsDialog`** 是 shell
顶栏齿轮打开的应用设置界面 —— 一个**store 驱动的双栏 shell**：
一个导航列表（只渲染一次）旁边是可滚动的内容栏。在 `md`+ 上该列表是分组的 rail：
**Agent**（Models、Providers、MCP servers、Prompt templates、
Environment）·**Interface**（Chat、Appearance、Language、Layout、Terminal）·
**System**（Workspaces、Integrations、Extensions、Notifications、Privacy、Updates）。
Integrations 保留区段 id `github`：目前它是 GitHub CLI 页面，也是后续集成的归宿。在
`md` 以下，同一列表是不带组标签的水平滚动条。Feedback 卡片（`settings-nav-feedback`，
“Share feedback”）在任何宽度都属于该列表，在 `md`+ 上固定在 rail 底部。纯导航数据 ——
分组、协议与注入内容门禁，以及把请求的区段解析为可用区段（被门禁或已退役的 id，例如旧的
`review` / `line-width` 深链，回退到 Appearance）—— 位于 `panels/settingsNav.ts`，
因此对话框只负责渲染它。`settingsOpen`/`settingsSection` 存在于 store，因此齿轮**与**
Welcome 横幅都能把它深链打开到某个区段。Escape 关闭对话框，除非焦点位于带
`data-settings-draft-input` 的输入框内，此时 Escape 改为还原该草稿。**共享 Settings
原语**让每个页面保持同一形态（shell 注入的 Layout 区段也复用它们）：
`SettingsPageContainer` 以间距堆叠页面的各组，
并只在真正渲染的子项之间绘制分隔线（每个子项保留其元素 key，
因此条件组绝不会重挂载其兄弟）；`SettingsGroup` 是 `SettingsRow` 的有标题区块 —— 标签、
可选描述、一个控件，在窄宽度下把控件绕到标签下方；`SegmentedControl` 是原生单选组，
`value: T | null`（`null` 表示什么都不选，用于宿主值未命名任何选项时）并支持按选项禁用；
`SettingsNotice` 是唯一的内联中性/info/成功/警告/错误通知。
自由文本设置共用一个**自动保存输入模式**：Enter 或 blur 一次保存有效的变更值（没有
Save/Apply 按钮）；无效草稿保持可见并带 `aria-invalid` 与可见范围说明，且绝不会被发送；
Escape 还原为权威值；输入框带 `data-settings-draft-input`；被拒绝的保存会 toast，
且权威值继续生效。**`McpSettings`** 是 **MCP servers** 区段（在 `MCP_PROTOCOL_VERSION`
以下隐藏）。它描述活动工作区的项目（无活动工作区 → “Open a workspace to manage its MCP
servers”），并通过 `transport` 的 `watchMcpWorkspace` 读取 `mcp.list` —— 在挂载时、每次
welcome、项目信任变化时，以及在有实时聊天仍报告某服务器 `starting` 时以有界退避重试，
因为宿主不推送 connected 状态变化。行来自 store 的 `deriveMcpServerRows`。其行的
`effective` 标志门禁仅按名称的 Sign in、Sign out 与 Test connection 动作，
无论在状态动作还是 overflow 菜单中：宿主可选的 `summary.effective` 是权威的，
即使一个有效的用户服务器带有无效的折叠仓库覆盖诊断。
只有那个被承认的定义才获得实时会话状态。对旧宿主，状态保留 scope/批准启发式，
但仅按名称的动作只允许用于唯一的有效用户定义或已批准的项目定义，绝不允许有歧义的同名行。
Review、Edit 与 Remove 仍按 scope 区分。当项目未受信时，项目 scope
的行会在客户端立即被丢弃，
因此撤销绝不让缓存的仓库行在刷新落地前仍可操作（宿主本来也会省略它们，横幅也这么说明）；
撤销还会关闭打开的 Edit、Review & approve 或项目 scope 的 Remove 对话框，
丢弃仍在进行中的 Edit / Review 读取，并且打开的 Add 对话框回退到 User scope ——
包括它已为 Project scope 走到的 review
步骤（无论如何宿主都会拒绝对未受信项目的仓库写入）。
用户服务器的启用/暴露变化只发送变更的字段（`mcp.setProjectOverride` 合并），而
Remove（任一 scope）与 Edit 携带渲染时的 `summary.fingerprint`（`mcpRenderedTarget`，
由对话框打开时的行构建，绝不重读），因此陈旧行会被拒绝，而不会重写或删除已变更的内容 ——
无论是一个检出改过的仓库条目，还是另一个客户端或编辑器在 Remove
对话框打开期间替换掉的用户条目；两者在失败时都会刷新列表，
因此用户看到当前条目并能从中重试。行显示状态点 + 标签 + 一行原因、名称、User/Project 与
HTTP/stdio chip（stdio：“Runs on host — inherits the host environment”）、
暴露菜单（`deferred | direct | hidden`；configured codemode 读作 “treated as deferred
(codemode not available yet)”）、启用开关、至多一个主动作与一个次动作 —— 状态表
`mcpRowView` 编码：Sign in（需要登录）· Review & approve（待批准，或待处理的仓库覆盖）·
Edit（无效的仓库条目）· **Reconnect**（失败，旁边带 **Show log**；已断开）·**Reload
now**（待重载）· **Open a chat**（未运行，HTTP 时旁边带 Test connection）—— 以及一个
overflow 菜单（OAuth 服务器的 Sign in / Sign out、HTTP 的 Test connection、Edit 或
Review & approve、用户服务器有项目设置时的 Share with repo、Details、确认后的 Remove）。
Reconnect 与 Reload now 按聊天执行：
它们为报告该行状态的实时聊天（`McpServerRow.sessions.reporting`）逐一切换聊天运行 ——
`mcp.reconnect`，以及通过 skills 基线的 `session.reloadResources`（因此该聊天的 skills
徽标也会清除）。失败以宿主文本留在行上（pi 的重连失败由宿主遮蔽；忙碌聊天的 “Can't
reload while the chat is busy”，它保持待处理并在空闲后自行重载），
当针对多个聊天时每个聊天一行、以标签页标题命名，且仅在该行保持该状态时显示 —— 绝不
toast。Open a chat 关闭 Settings 并通过 `openChat` 的 `startChatInTab`
在该工作区启动聊天，与 workbench 的 New chat 是同一条路径。Show log 打开
`McpLogDialog`：pi 对该服务器的最后报告（`detail`，含 stdio 服务器的 stderr
尾部）以及其在 pi 的 `mcp.log` 中的行（`mcp.readLog`，由宿主遮蔽）。Details 不列出工具：
除了始终允许的那些，目前没有载荷携带工具名（pi 的 `/mcp` 文本与 `SessionResources`
报告的是计数）。需要关注的行排在前，禁用的行仍列在最后；`handledElsewhere`
使该区段变为只读并带一条通知（`mcpHandledElsewhereText`）：“MCP is handled by <path> in
this workspace; ThinkRail's MCP management is read-only.”（针对替换型扩展），而对宿主的
`by: "pi settings (-builtin:mcp)"` 则显示 “MCP is turned off in pi settings
(-builtin:mcp); remove that entry to manage servers here.”。整个 `mcp.json`
的问题（`McpListResult.configErrors`：无法解析的 JSON、`mcpServers` 不是对象、拼错的
key）按文件渲染为一条错误通知（`McpConfigFileErrors`），命名其路径、pi 的消息以及 “Edit
the file directly to fix it”，无论位于列表之上还是空状态之上 —— 因此 pi
无法读取的文件绝不会被当作“没有服务器”。状态详情（pi 的报告）到达时已由宿主遮蔽。
**启用与暴露遵循同一条路由规则（`mcpSettingWrite`）：** 用户级服务器仅为此项目变更，
通过 `mcp.setProjectOverride`（宿主原子合并字段，因此只有变更字段被传递），全局
`mcp.json` 从不写入；仓库定义的服务器在 `.pi/mcp.json` 中用 `mcp.update` 重写，
其整条目替换从条目被写入时的样子开始，通过 `fs.readFile` 读出。每次 `mcp.update`（开关、
暴露、Edit）都由 `mcpEntryUpdate` 构建，并携带该行渲染时的 fingerprint 作为
`expectedFingerprint`（`summary.fingerprint`，与 Remove 在任一 scope 发送的身份相同），
因此宿主会拒绝基于自那以后已在磁盘上变化的条目重建的写入；
宿主只从与磁盘当前内容一致的已批准条目继承批准。被拒绝的 update 或 share 会在行上（或在
Edit 对话框中）显示宿主错误并重读 `mcp.list`。待批准、无效与已替换的行会锁定两个控件，
因此开关绝无法走后门批准仓库条目。读取原始条目仅用于 update 与 fingerprint：
没有对话框会显示其中的明文 env、header 或 OAuth client-secret 值（以 `<literal value
hidden>` 代替；`${VAR}` / `!command` 引用会显示，因为它们才是实际运行的东西）。**Review
& approve** 列出将要运行的内容 —— 宿主遮蔽后的命令行或 URL（`summary.endpoint`，
貌似凭据的参数读作 `***`）、工作目录、每个 `!command`，以及带引用或隐藏标记的
env/header 名称 —— 然后用摘要的 fingerprint 批准（宿主会拒绝自那以后变化的文件）。
被拒绝的批准也会刷新 `mcp.list`；打开的 review 保留其原始 fingerprint，
绝不自动重试已变更的内容。关闭并重新打开 review 会读取刷新后的摘要，
并在新的确认前重读条目。**Edit** 用于仓库条目：原始条目以 JSON 呈现，明文值被隐藏；
留在原地的标记保留该名称下文件的值（`mcpEditedEntry`），新值必须是引用。用户级服务器的
Edit 是禁用的菜单项，命名其源文件（`McpServerSummary.source`）：`mcp.update`
替换整个条目且宿主不返回原始用户条目，因此 Settings 只能有损重建 ——用户直接编辑
`~/.pi/agent/mcp.json`，而开关与暴露菜单继续写入本项目的记录覆盖。**Add** 有三个层级：
来自 `mcpPresets.ts` 的预设 —— 五个，经核实并固定版本（不用 `@latest`）：Context7
与 DeepWiki（`direct`，无需认证）、Linear 经由其厂商只读端点（`/mcp/readonly`）与
Sentry（两者均 OAuth —— 添加后登录），以及 Playwright（`npx -y
@playwright/mcp@<pinned>`），
它作为禁用的用户条目加本项目的记录覆盖（`optInPerProject`）添加，
因此只在用户选择加入处运行，且绝不触碰仓库；`modifiesData`（Sentry、
Playwright）会添加“this server can modify data in <service>” 确认，
而名称已配置的预设读作 **Added**。GitHub 保持为配方而非预设（`gh auth token` 的 scope
未经验证）：在预设下方，*Use recipe* 用 `GITHUB_MCP_RECIPE_JSON` 填充 Paste JSON 层级 ——
GitHub 的只读端点（`/mcp/readonly`）和来自 `!t=$(gh auth token) && echo "Bearer $t"` 的
`Authorization` 头，在 gh 未登录时它会失败而不是发送空 token（`managing-mcp-servers`
skill 携带同一条目）；还有短表单（pi 的名称字符集、按文件去重与 `-`/`_` 双胞胎、
每行一个参数的 stdio 命令或 http(s) URL、env/header 行、暴露、描述，以及一个 scope，
其中 Project 需要受信项目）；以及粘贴的 `mcpServers` JSON（包装器或裸映射，
按服务器预览；有效条目被添加，SSE 与畸形条目被标记）。**Settings
只存引用**（`mcpSecretIssues`：表单中的机密值从不存储）：表单中每个 env/header
值都必须是 `${VAR}` / `$VAR` 模板或 `!command`（pi 的语法，`isConfigReference`：`$$` /
`$!` 转义下一个字符，一个完整但畸形的 `${...}`
作为字面量被吞掉且不重新扫描其中内部的美元符，后续引用仍计数）—— 其他任何内容，
包括空值，都是带重写提示的校验错误，而添加后完全留空的行被报告为未命名行而不是被丢弃 ——
且其 URL 不得携带用户信息或以凭据命名的查询值（pi 按原样发送 URL，因此建议用 header）；
粘贴条目若带明文 env、header 或 OAuth client-secret 值或此类 URL，
会以同样的提示被标记且不被添加。没有明文确认。任何会先运行某些内容的保存都会先显示一个
review 步骤，列出每个命令与 `!` 值；当其中有 `!command` 时，它和 Review & approve
会加一行：“!commands run on the host each time a chat connects; keep them fast (e.g.
read a cached token).”（pi 在每次连接时同步解析它们）。`mcp.add` 按服务器运行；
添加失败留在对话框中，粘贴只保留失败的添加。
成功的添加独立于预设随后的项目选择加入而被提交：只有成功的添加才调用
`onEnableInProject`，其 `Promise<void>` 契约把启用失败的呈现留给 Settings。Add
等待该回调结算（即使被拒绝），绝不把已提交的添加转成添加失败，也绝不重试该添加。
如果无法确认选择加入，Add 会关闭（除非还有其他添加失败），Settings 既有的 `user:<name>`
行错误显示“Added, couldn't confirm enabling for this project; use its switch to
retry.”它不会在响应含糊时断言服务器已禁用。重试行开关只发送 `mcp.setProjectOverride`；
不引入新流程或状态所有者。每次变更后，一行状态说明变更在何处生效：“when a chat starts
in this workspace”（没有聊天打开时），否则 “to open chats as they reload; a busy chat
waits until it is idle” —— 宿主在重载聊天之前回答，因此返回的快照无法说明哪些已完成；
行上的实时状态可以。Sign in / Test connection 启动 `mcp.login` / `mcp.testConnection`，
并像 Providers 一样复用 `auth/LoginDialog`：登录状态保留推送的 `target`，
因此只有本工作区的 MCP 登录才打开该区段的对话框（Providers 忽略它，
为另一个工作区显示的区段也忽略），而登录页仅从对话框的按钮打开。一条横幅读作：
“ThinkRail asks in the chat before MCP calls that may change data. Always allow trusts
a tool in every chat of this project; those tools are listed on their server below.”
摘要携带 `alwaysAllowed` 的行显示一行 **Always allowed in this project:** 工具
chip（宿主的 `label`，pi 工具名作为 title）—— 明确写出 scope，因为该行的 USER chip
描述的是服务器，而不是授予；每个 chip 的 ×（“Ask again before <tool> in this
project”）通过 `requestMcpList` 发送 `mcp.revokeToolGrant`，并在该行只读或忙碌时禁用。
这是撤销授予的唯一位置。空状态提供预设、表单、JSON 导入与一条宿主通知。活动区段：
**`ProvidersSettings`**（应用内 provider 认证界面 —— 头部一行带页内
**Refresh**（`providers-refresh`，状态读取进行中时禁用），下面是分组
**Connected**（每行仅在 `canLogout` 时有 **Sign-out** ——env / models.json 认证改为显示
“Managed” 标签，因为宿主无法取消它；`kind: "central"` 的行标为 “JetBrains AI”，其
Managed 标签指向拥有该连接的 JetBrains AI 卡片）、**JetBrains AI** 卡片、
**Subscriptions**（未配置的 `canOAuth` provider）与 **Available (N)**（仅 `canApiKey`
的其余项，用 “Show N more” 展开器设上限）。每个未配置 provider 都是一行：`canOAuth`
时给 **Sign in**，`canApiKey` 时给 **API key** ——
**两条路径都启动 `provider.loginStart`**（`type` `"oauth"` / `"api_key"`，issue #97）
进入同一个 store 驱动的 `auth/LoginDialog`（打开 URL / 粘贴 code / 回答 provider
自己的 key 提示，`provider.loginReply` —— 没有内联 key 字段）；两个标志都没有的行显示
“configured outside the app” 说明；以及 **`JetBrainsAiCard`** —— 通过用户的
JetBrains 订阅路由 Central 支持的模型，同时保留 ThinkRail 内嵌的 PI —— 一个基于类型化
`JbcentralStatus` + `provider.jbcentral*` 的状态机：absent（官方宿主 OS 安装指南 +
Recheck）、outdated —— 低于宿主的最低支持 Central（引导 Update）、invalid/unverifiable
version（安全引导，无原生动作；*高于*最低要求的版本就是就绪，绝不被门禁）、**signed
out** —— 卡片**陈述该事实且只提供 Sign in**：主动作*替换* Connect 而不是与之并列，而在
`supported` 时已登出的行替换 “Central is ready” 断言而不是为其加注。
规则是卡片绝不宣传无法成功的动作 ——无凭据连接会失败 —— 因此前置条件成为所提供的动作，
一旦宿主报告凭据，Connect 就回归。**无论底层配置如何，已登出都渲染为一个状态：**
正文只说 Central 已登出 —— 绝不与会与之矛盾的 “Connected”行并列 —— 且**Sign in
是唯一动作**，Disconnect 连同 Connect 一并扣留。认证之后，一个 proxy
被明确观察到已停止的已配置状态同样用 “Central's proxy is not running” 替换成功断言，
且只提供 **Start proxy**；启动之后，Connected + Disconnect 回归。因此前置顺序是 Sign in
→ Start proxy → 常规已连接控件，绝不出现竞争性动作。未知的 proxy 健康状态不会制造要求。
损坏的会话只请求能解决其当前前置条件的那一件事，而不是把修复与无关选择或成功消息配对。
**登录是一个按钮，绝不是菜单：** ThinkRail 在宿主上启动 Central 的流程，而 `central
login` 命令*仅*在该启动失败处出现 ——
把命令打印在可用按钮旁边会让用户在同一条路的两条路径之间做选择。
由于该流程在**宿主**的浏览器中打开，启动后的确认如此说明并指明 Refresh 为下一步，因为
Connect 尚未出现在屏幕上。*反应式*引导为探针看不见的情形保留：凭据存在但动作仍被拒绝 ——
需要登录（启动 Central 登录 + Retry）、就绪（Connect）、配置中（某个 Central
动作或受监视的候选重建正在进行）、已连接（新工作的当前运行时已应用 Central；
Disconnect）、加载失败（最后运行时或启动时的朴素回退仍可用；Retry 或 Disconnect），
以及通用动作错误（Retry/Recheck）。没有重启提示、受影响聊天列表、阻塞状态或恢复模式。
既有实时聊天可能保留较旧的运行时 —— 包括 Disconnect 之后的 Central ——
卡片说明其状态适用于新聊天。更新/连接/断开状态由宿主权威并在客户端间共享；
每次变更都会重读 `provider.status`，而来自受监视外部变化的 `provider.changed`
失效会触发同样的重读加模型列表失效。状态读取按请求排序，
因此较旧的响应无法替换较新的受监视/动作结果。文案绝不承诺只有 Claude/GPT，绝不要求独立
PI，绝不渲染子进程输出/诊断/工件内容/路径/proxy 数据/机密/原始模型，
并且只把封闭的原因码映射为 ThinkRail 撰写的文本。在协议 v59+ 上，同一张卡片在每种
Central 生命周期状态下都始终显示同步的 **Show quota in top bar** 开关与 **Refresh every
_ seconds** 字段。该标志默认开启；间隔默认 30，接受 `1–3600` 的整数。
关闭会禁用（但保留）该间隔。字段本地编辑，在 blur/Enter 时提交，内联报告无效范围，
并等待 `settings.changed` 而不是安装乐观权威。旧宿主两个控件都没有。
**`GithubSettings`**（Integrations 区段的 “Local GitHub” 组 ——把 `github.authStatus()`
显示为 `settings-gh-status[data-connected]`：Connected + login，或一条点名 `gh auth
login` 的未连接通知；两种状态下都提供 Refresh（`github.refresh`））；
**`LanguageSettings`**（以 `LOCALES_PROTOCOL_VERSION` 为门禁；一个整合 store 的容器，
包在 props 驱动的 `LanguagePage` 之上，其推导来自 [[submodule-web-i18n]]）：
一个单选列表 —— **System**（默认；命名 System 解析到的语言），
然后按各语言自身名称列出每个可用语言并带来源徽标（*Built-in*、*Local file*、*Built-in,
customized by local file*），然后是已选中但文件缺失的不可选行（*File not found*；
渲染英文，选择被保留）以及每个损坏文件的行（其翻译后的错误码加上未翻译的解析器细节）。
选择会写入 `settings.update { locale }` 并收敛到广播。*Add a language* 给出步骤，带指向
`v<appVersion>`（已发布版本；否则为 `main`）的 `packages/locales/en.json` 链接、
宿主文件夹路径及 **Copy path** 与 **Open folder**（`locales.reveal`，
像工作区揭示一样在每个客户端都提供 —— 远程浏览器用该路径），以及指向 CONTRIBUTING
的 Translations 小节的 **How to contribute** 链接）；
**`AppearanceSettings`**（目录驱动的主题设置：一个 **Theme mode**分段控件，**System /
Light / Dark**，System 仅在 `THEME_SYSTEM_PROTOCOL_VERSION`
起提供（旧宿主保持仅固定行为），以及一个 **Contrast level** 控件，**Default / High**。
Light/Dark 写入固定主题：从 System 切换时它们采纳该外观下配对的那一半 ——
即控件当时显示的 —— 绝不是陈旧的保留固定主题；在固定模式下它们移动到当前主题族的对应项。
Contrast 切换到同族的另一对比度主题（System 模式下两半同时切换），
因此无论哪次切换主题族都存活；目录中没有目标的选项被禁用，而不是映射到无关主题。
具体主题位于折叠展开区之后 —— 固定模式下是 manifest 列表，System 模式下是经外观过滤的
`Light theme` / `Dark theme` 选择器，任一槽位独立地是普通或高对比 ——
因为模式与对比度回答了常见问题。模式与 manifest 列表保持分离：把 System
变成另一个主题行会把配置嵌进一个类单选选项。System 模式添加一行 `Current on this
device`（`system-theme-current`），读作 `<device icon> <Light|Dark> → <palette icon>
<resolved label>` ——图标表明哪一半是设备外观、哪一半是主题，因为两半常是同一个词；
模式与配对跟随用户，而每个设备读取自己的系统设置。首次启用 System 会原子地发送模式 +
由主题推导的同对比度配对；之后的槽位编辑替换完整配对，返回固定只改变模式，保留两个选择。
本面板同一时刻只允许一个主题变更在进行；其控件使用真实的禁用状态直到请求结算，
防止快速的完整配对写入用陈旧的兄弟槽位互相覆盖。每个动作都触发 `settings.update`
并**收敛到 `settings.changed` 广播**，没有乐观应用；拒绝则控件/主题不变并抛出 toast。
不可用或外观错误的已配置 id 由模式控件下方的警告通知披露 ——位于展开区之外，因此始终可见
—— 点名生效的同外观回退，并且绝不会被静默写回。面板从不拥有主题列表、媒体查询逻辑、
配对推导或回退 —— 全部来自 `themes`；它只推导两个控件会移动到的族/对比度对应项。一个
**Custom themes** 组（`CustomThemesSettings.tsx`，像 `LanguagePage` 一样 props 驱动，在
`USER_THEMES_PROTOCOL_VERSION` 以下隐藏）给出复制打包文件的步骤，带固定版本的
`dark.theme.json` 链接、`<dataDir>/themes` 文件夹及 **Copy path** / **Open
folder**（`themes.reveal`），以及每个不可用文件一条警告通知（`themes` 的问题列表）。
该页面通过 `onThemesChange` 读取目录与问题，因此放入的文件无需重开即会出现在列表中。
一个 **Editor** 组承载文件 `LineWidthControl`）；
**`LineWidthControl({kind})`**（`LineWidthSettings.tsx`，没有独立页面 ——`chat` 控件在
Chat 中，`file` 控件在 Appearance → Editor —— 一个 40–240 的整数 **Line width** 字段，
使用自动保存输入模式，外加一个独立的、宿主同步的 **Fit to pane**
开关（`${kind}-line-width-bounded`）；默认 120/开。来自 `settings.changed`
的权威宽度变化会替换陈旧草稿；变更只收敛到该广播，被拒绝的调用会 toast
且不改变几何））；**`ChatSettings`**（一个 **Appearance and behavior** 行组 ——
**Message order** 基于 `store.chatMessageOrder` 分段（Oldest first，兼容默认 / Newest
first，选择加入）、**Composer size** 基于 composer 增长上限分段（Compact / Roomy / Half
chat）、chat 的 `LineWidthControl`，以及 **Streaming response movement** 行，
一个基于 `store.streamingResponseMovement` 的双手柄范围。movement 行的文案是 “When the
chat moves while an answer grows,and where its newest edge lands”；一个轴是 Top →
Message box，Settle 为 25–90，Trigger 为 35–100，都以 5 为步长且最小间隔 10，
显示的默认为 75%→100%。它不暴露 runway/tail/lifecycle 控件。消息顺序与 movement
都立即生效，并只通过聊天偏好接缝持久化在本客户端：浏览器使用带当前宿主限定的 key，
而原生 shell 可注入其稳定的后端配置/窗口适配器。另一个浏览器、原生窗口或宿主都不受影响。
Composer 增长仍是顶层 `AppConfig` 字段并收敛到 `settings.changed`，拒绝时 toast。
解释性文案说“message box”，而不是内部名 “composer”，指用户输入的位置。最后的
**Subagents** 组（`SubagentSettings.tsx`）是行而不是表格：全局 **Allow subagents**，
以及自 `SUBAGENT_CONCURRENCY_PROTOCOL_VERSION` 起的 **Per chat at once**；然后，
在有活动工作区时，一个 **This workspace** 子标题点名它，下面是同样两行。整个组要求
`SUBAGENT_SETTINGS_PROTOCOL_VERSION`，
因此独立发布的客户端绝不会对旧宿主提供不支持的变更。工作区行显示*生效*值并带
Global/Custom 标签：切换写入 `on`/`off`，输入的限制写入覆盖，空的限制字段（占位 =
全局）或 Custom 标签（重置按钮）发送 `null`，因此后续的全局变化仍会流通。
限制字段遵循自动保存输入模式，由 `isSubagentMaxConcurrent`
校验（无效草稿保持可见并带范围错误且不被保存），且从不钳制。当该工作区的子 agent
实际关闭时工作区限制被禁用；全局限制即使全局关闭仍可编辑，
因为被强制开启的工作区仍会继承它。全局变更通过 `settings.changed` 收敛，工作区变更通过
`workspace.updated`；没有一个是乐观的）；**shell 拥有的注入 Layout
区段**（Balanced/Focus/Review 外加命名的自定义预设卡片。
预设与实时框架匹配的卡片读作已应用 ——通过比较几何推导，因为没有发布当前布局 —— 其 Apply
被禁用。生效默认值携带 **Default** 徽标（存储的默认值已不存在时为 **Fallback**），其
tooltip 说明 Reset 会使用它，且它按窗口记住。一个 **Add new layout** 卡片打开
popover（名称 + Save preset），捕获实时框架。
自定义的捕获/重命名/删除更新宿主同步的目录并通过 `settings.changed` 收敛；
默认选择与独立的侧/底组上限是前端本地的，上限在折叠的 **Advanced** 展开区内自动保存。
在有活动工作区时，每张卡片的 **Apply now…** 与 **Reset to default**
行（`layout-reset-default`）都经过一次确认，然后请 shell 替换本窗口的框架，
并在每个保留的工作区视图内原子地保留/重排打开的资源身份）；可选的**shell 拥有的注入
Update 区段**（Settings shell 仅在提供内容时才包含其行；`panels`
既不发现原生也不发现宿主更新能力。如果之后的 welcome 在 Updates 被选中时移除了注入内容，
则渲染并高亮 Appearance，而不是留下没有活动行）；
**`WorkspacesSettings`**（**Workspaces** 区段，仅在 `protocolVersion >=
WORKSPACE_SETTLE_PROTOCOL_VERSION` 时列出）——一个 `SettingsRadioCards` 组，**Settle
idle workspaces after**：1 / 3 / 7（默认）/ 14 days / Never，写为 `settings.update {
settleIdleDays }`（Never 为 `null`）并通过 `settings.changed` 收敛；
宿主配置的合法非预设值会让预设卡片都不被选中，而不是错误地显示默认值。
默认是一周而非一个长周末：合并/关闭的 PR 已经立即结算，因此空闲规则只捕获休眠的工作，
并倾向于让暂停的工作区保持活动（周四的工作到周一就在置物架上读起来像丢失而非整理）。
文案点名算作活动的三件事与永不结算的三件事，因为该设置只有连同那些规则一起才好理解；
**`TerminalSettings`** —— 一个 **Scrollback** 分段控件（`store.terminalReplayKb`，从
Off 到 1 MB 的五个预设；合法的非预设宿主值不会选中任何预设，而不是选错一个；
`settings.update { terminalReplayKb }`，适用于此后打开的终端），以及在 Windows 宿主上
`protocolVersion >= WINDOWS_SHELL_SETTINGS_PROTOCOL_VERSION` 时的一个 **Windows shell**
选择器（Auto / PowerShell 7 (pwsh) / Windows PowerShell / Command Prompt，
`settings.update { terminalWindowsShell }` ——每个选项启动什么见
`submodule-server-terminal` 的 shell 选择决策）。
协议门禁防止较新的独立发布客户端呈现一个旧宿主会保留但不生效的设置。Windows-shell
部分被拆成 **props 驱动**的 `WindowsShellSettings` 组件，而不是像 replay
选择器那样内联读取 store：zustand 的 React 绑定会把其冻结的 `getInitialState()`
快照（`useSyncExternalStore` 的 `getServerSnapshot` 参数）喂给 `renderToStaticMarkup`，
而不是测试的 `setState`，因此任何在该渲染路径下仍需可断言的设置区段都改为以 props 接收
store 值 —— 与 `ChatSettings` 已对 `SubagentSettings` 使用的形态相同；以及
**`TemplatesSettings`** —— 两个组，**Project templates** 然后是**Global templates**，
每组一个带柔和 **New** 按钮的头部加上其行（无活动工作区时项目组只是一条打开项目的通知，
没有 New），通过**两次独立的 `template.list` 调用**抓取（每当 store 的
`templatesVersion` 递增就都重抓，各自带失败标志，
因此一个的成功绝不会覆盖另一个仍然真实的失败）：**Global** 用无 scope 的（`{}`），
项目组用 `{ workspaceId }` 并过滤到 `scope === "project"`。无 scope 调用之所以重要，
正是因为服务器的 `template.list { workspaceId }` 响应是**影子合并**的（`templates.ts` 的
`listTemplates`：项目模板胜过同名的全局模板）——对 composer 的 `/` 菜单是对的，但如果
Settings 的 Global 组也使用同一个按工作区限定的调用，
被遮蔽的全局模板就会完全从视野消失，无法找到、编辑或删除（`data-testid="template-row"`：
名称 + 描述，且 —— 仅项目行 ——一个 **Open as file** 动作，通过文件树使用的完全相同的
`openTabs.ts` `openFileInTab` 打开 `.pi/prompts/<name>.md` —— 使用 **`keep`** 意图，
因为刻意的“在编辑器中打开”不得落进一个会被后续点击静默替换的预览槽位 —— 然后关闭
Settings，以及一个 **Edit** 动作；全局模板没有可打开文件标签页的工作树，
因此全局行只有对话框）。**New**/**Edit** 打开共享的 `chat/TemplateEditorDialog`（见
`chat/SPEC.md` 的 Save-as-template 条目 ——它位于 `chat/` 是因为 `HistoryOverlay` 的
save-as-template 动作需要完全相同的表单，而 `chat/` 不能导入 `panels/`）。**Delete**
是锚定在该行自身 Delete 按钮上的 `ConfirmPopover`，直接调用 `template.delete` ——
对话框本身从不参与删除。**R4 —— 起始模板提议：** 当 **Global**
组的抓取已解析为零行且无错误时，其空状态是一条通知，带 **Add starter templates**
按钮（`data-testid="template-starters"`）—— 点击它按顺序 `template.save`
五个逐字起始模板（scope `"global"`，正文在客户端通过共享 `prompt` 模块的
`assembleTemplate` 组装，即 `TemplateEditorDialog` 使用的同一助手），然后递增一次
`templatesVersion`，即行列表已在其上重抓的同一失效 ——
该提议在列表非空后的下一次渲染自行消失，无需追踪关闭状态。
那五个（review/explain/tests/commit/rename）**与本仓库检入自身 `.pi/prompts/`
的是同一组**：那些发布在*项目* scope，因此只有 ThinkRail 检出的仓库看得到它们，
而“ThinkRail 发布的模板”必须只意味着一件事而不是两件 —— 改一个就改另一个。composer 的
`/` 菜单承载可发现性的一半（`chat/SPEC.md`：当任何地方都没有模板时，一个
`slash-templates-empty` 页脚提示深链到这里），因为这个提议否则就藏在对话框两层深处。
项目组的空状态是一条朴素、无图标的通知，点名该项目的 `.pi/prompts/` ——该提议仅限
Global，因为它只播种全局文件。没有服务器改动。**`NotificationsSettings`**
是应用外注意通知的单一总开关（默认开），通过 `settings.update` 持久化 `{
notificationsEnabled }`。启用时它还通过[[submodule-web-notifications]] barrel
暴露实时浏览器权限：权限仍为 `default` 时一条 **Allow in browser**
通知（该点击就是调用通道 `requestPermission` 的用户手势；从 `default`
打开总开关也会询问）、`denied` 时的警告并带可折叠的 **How to enable** 步骤、浏览器没有
Notification API 时只有警告没有步骤，而一旦授予或在 OS 通道上则什么都不显示 ——
这是可见的应用内获取路径，不依赖失焦事件。通知管道、抑制与离开路径的权限前言位于
[[submodule-web-notifications]]。**`PrivacySettings`** 一并管理额外数据偏好与确认；
事件契约属于 [[submodule-server-analytics]]。该栏以指向项目 readme 分析小节的链接结束。
**`AnalyticsConsentDialog`** 在支持该能力的宿主未确认配置水合后由 shell 挂载一次，
在冷启动引导向导仍可能显示或正在显示时（`selectOnboardingActive`）被暂缓（连同调查邀请）
：新用户先见到产品再回答隐私问题，而在对话框挂载前共享保持关闭，
因此暂缓它绝不会未经询问就收集。向导本身在其自然的接缝处询问 —— **Get started**
通向它自己的 **privacy** 步骤（同一个 `AnalyticsConsentChoice` 正文，通过
`useAnalyticsConsent` 预热并保存；**Continue** 确认且只在该保存落地后才前进，
因此写入失败会保留该步骤及其草稿与错误，
而不是带着仍然预置的共享继续前进）在模型列表之前，且仅在同意未确认时 —— 因此既不是
intro 也不是第一个项目在隐私提示上打开；在 intro 上跳过则把问题交回 shell 的对话框。
代价是 intro 只携带基础事件。它把草稿开关初始化为开，并立即持久化 `{ analyticsEnabled:
true }`；持久化与广播在对话框保持打开时激活宿主门禁。**Done**、Close、Escape
与背景点击都以 `analyticsConsentConfirmed: true` 持久化当前草稿，因此普通结果就是开启。
关闭开关会立即持久化关闭/已确认，随之而来的配置广播关闭对话框。
已确认的开/关配置从不挂载或预热。持久化失败会让草稿与错误保持可见以便重试。
在用户于该界面翻转开关之前，期间在别处确认的决定（Settings → Privacy、
另一个窗口）会被采纳而非覆盖：开关显示保存的值，Continue / Done 前进而不写入 ——
只有在这里做出的选择才被持久化。已保存的决定跨重启存活，并在之后通过 Settings 更改。
对话框只保留简短的产品使用文案与共享开关；Settings 添加可选结果、报告维度与排除内容。
旧宿主保留其遗留隐私控件，没有新的同意对话框。**`FeedbackSettings`** 是始终存在的
Feedback 栏：**Take the survey**（仅在调查已启动且未完成时）打开调查窗口，旁边是
Discord 链接与 `info@thinkrail.ai`。Settings 从不自行绘制问题，也从不调用
`feedback.respond`。**`SurveyInvitationDialog`** 是自动邀请：**Never show again** 记录
`never`，**Not now** 与关闭记录 `postpone`，**Take the survey** 打开窗口并记录
`postpone`，因此离开调查且未发送会在之后再次邀请。每个动作都立即关闭对话框；响应失败是
toast，绝不是被困住的模态框。**`SurveyWindow`** 是唯一的调查 UI。
其第一页在第一个问题之上携带数据通知（发送什么、关联到随机安装 ID、
不含机密/代码/个人数据、可通过 `info@thinkrail.ai` 删除；没有链接）；`SurveyForm`
随后每步绘制一个问题（评分点是真实单选输入；选择 Other 会聚焦其文本框），带
**Continue**，最后一步带 **Send**。发送前不发送任何内容，关闭从不调用宿主，
分析设置也不参与。`SURVEY_CHANGED` 会重载调查并以简短通知重新开始；其他拒绝保持内联；
成功显示 PostHog 的致谢头部或应用自己的。表单以调查 id、迭代与 `language` 为 key，
因此另一种语言的重载会重新开始而不是混杂语言，Send 会回显 `survey.language`。v98
之前的宿主通过 `surveyDraft` 的 `inlineOthers` 得到较旧的提交形态（Others 文本内联、无
`language`）；在没有宿主被 welcome（断开）时 Send 被禁用，
因此不会有请求被排队并以一个在不知宿主情况下选定的形态重放。**`ModelsSettings`** 是
**Models** 区段，仅在协议 v72 或更新时可见。它在打开时、
实时目录变化时（包括空目录）以及每次保存后重读 `model.default` ——
宿主针对其已结算目录解析保存的模型及其力度级别，因此消失或改变的模型绝不会提供陈旧级别；
只有最新读取生效，读取失败则用重试替换控件 —— 并通过 `settings.update` 写入
`defaultModel` / `defaultEffort`，持久化失败时抛错误 toast。宿主的 `model.default`
结果是显示的生效选择，包括保存的模型缺失时的首个可用回退；
支持的力度级别与显示的力度来自同一个已解析、经 Pi clamp 的模型。在保存或重读进行中时，
两个触发器都被禁用，已打开选择器中的选择也被忽略。在 v76，**`ModelContextSettings`**
在宿主作为 `ModelContextSetting[]` 返回的每个合格 GPT 模型上添加一个 Default / 1M /
Custom 选择器；折叠的 **By each model**展开区为每个 provider/model 对显示一个选择器，
因此同一模型在不同 provider 上保持独立可编辑。选择以显式 `override` 为 key（Default =
`null`，pi 报告的目录值）。共享控件只汇总契约的 `isSharedModelContextTarget` 承认的行 ——
当它们的覆盖不同时显示 “Customized by model”，统计它不动其外部行，
在不会治理任何行时省略 —— 选择共享预设会在一次 `model.setContextWindow`
调用中替换那些行的覆盖。Custom 显示一个有界于契约 272K–1M 范围的整数字段，在 Enter 或
blur 时保存 —— 但 blur 到同一控件的兄弟预设除外，因此选 Default/1M
绝不会先提交一个半输入的值；文案把它标为应用策略，而不是经验证的 provider 限制，
超出范围的外部值保持可见但无法重新应用。草稿是 UI 本地的，并在其权威覆盖变化时被丢弃；
输入不被重挂载，禁用保存后焦点只回到发起它的控件。Pi 的共享配置是权威 —— 没有乐观值或
AppConfig 字段；读取跟随目录/provider 失效，栅栏掉陈旧回复，待处理时禁用控件，失败时用
Retry 替换控件。props 驱动的 `ModelContextControls` 拥有呈现；
旧宿主既得不到该区块也得不到其请求。`ModelsSettings` 把控件组织为三个视觉组：

  **DEFAULT MODEL** —— 一个 **Chat model** 行（`defaultModel`、`defaultEffort`）与
  **Visible models** 展开区。**REVIEW**（`models-review-group`）—— 在一处配置 review
  的所有内容，因为用户按这个词去找它：**Review model** 行（`review-model-row`；
  `reviewModel`、`reviewEffort`），独立于 Chat model 行的加载状态，
  然后是下面两个自动化开关。两个模型行都在 `useModelCatalog` 之上使用
  `ModelSelector`/`ThinkingSelector`；review
  选择器携带一个**显式的默认模型行**（`model-option-default`，以宿主的 `model.default`
  结果标注），它写入 `{ reviewModel: null, reviewEffort: null }` —— null 清除的线上形态，
  见 `submodule-server-settings` ——因此选定的 review 模型可以恢复到宿主的新聊天默认值，
  无需手改宿主状态；未设置时，力度控件运行在默认模型支持的级别上（从 `model.default`
  抓取一次），而不是空列表。**`VisibleModelsSettings`**
  是一个折叠展开区（其摘要统计隐藏的模型），列出每个目录 provider/model 对，
  带搜索字段与开关；隐藏模型通过 store 的 `selectPickerModels` 离开每个模型选择器（固定、
  最近、provider 组、搜索），它始终保留调用方当前的选择即使已被隐藏，新模型默认显示。
  隐藏是**客户端本地**偏好，不是宿主配置：`chat/modelVisibility`
  的 `initModelVisibilityPersistence`（从 `main.tsx` 调用一次）水合 `store.hiddenModels`
  并回写变更 ——在注入了原生 shell 稳定的 `clientPreferences` 适配器时使用键
  `hidden-models`，否则使用浏览器 localStorage
  `${STORAGE_PREFIX}hidden-models:<httpBase>`，其 `storage` 事件同步其他标签页。
  完整目录在此仍可用于配置。

  REVIEW 组的开关：一个 **agent-review 开关**（`agent-review-toggle`、
  `agent-review-row`，基于 `store.agentReviewEnabled` 的开关 → `settings.update {
  agentReviewEnabled }`）—— 关闭会扣留 worker 会话内的 `request_review` 工具，因此
  review 只能通过 Review 按钮发生（宿主据此实时切换工具的活动集合，
  见 `submodule-server-host-plan-review`）；该行在
  `AGENT_REVIEW_SETTING_PROTOCOL_VERSION`（v68）以下隐藏，
  在那里宿主可能存储该字段却仍注册该工具。以及一个 **auto-fix
  开关**（`review-autofix-toggle`、`review-autofix-row`，基于 `store.reviewAutoFix`
  的开关 → `settings.update { reviewAutoFix }`）—— 关闭意味着 `request_changes`
  判定只记录发现并等待（宿主据此门禁其 auto-fix 循环，见 `submodule-server-todos`）。

  **GPT CONTEXT WINDOW** —— 上文描述的 `ModelContextSettings` 区块。

  没有**独立的 Review 导航区段**；所有 review 相关控件都在 Models 的 REVIEW 组中。
  **`ExtensionsSettings`**（**Extensions** 区段，导航 `settings-nav-extensions`，
  仅在 `protocolVersion >= EXTENSION_CATALOG_PROTOCOL_VERSION`
  时列出）在该区段每次挂载时用 `extension.list` 抓取宿主注册表目录 ——
  目录按宿主进程静态，因此它是区段本地状态而非 store 切片 —— 并通过 props 驱动的
  `ExtensionsCatalog` 渲染：每行标题、描述、工具名 chip 和一个
  `@thinkrail/ui/switch`（`extension-toggle-<name>`）。开关状态来自
  **`store.disabledExtensions`**，而不是目录的 `enabled` 快照，
  因此它跟随来自任何客户端的 `settings.changed`；切换通过 `settings.update {
  disabledExtensions }` 写入整个列表，
  并且像每个设置开关一样收敛到该广播而不是乐观渲染（`submodule-server-settings` § Get
  right）。列表中已有的未知名称原样携带通过。空的注册表渲染 `extensions-empty`；
  抓取失败渲染 `extensions-failed`。
  禁用绝不注销工具渲染器（[[submodule-web-extensions]]），因此过往转录仍会渲染。
  **`EnvironmentSettings`**（**Environment** 区段，导航 `settings-nav-environment`，
  仅在 `protocolVersion >= HOST_ENVIRONMENT_PROTOCOL_VERSION`
  时列出）编辑存储的宿主环境（[[submodule-server-environment]]）：pi 配置文件夹、
  正在使用的文件夹（`effectiveAgentDir`），以及有序的名称/值行，其值在按行揭示前被遮蔽。
  它在挂载时把 `environment.get` 抓取进区段本地状态 —— 从不是 store 切片，
  因为值可能是机密 —— 并保持显式草稿（Save / Discard，而非逐字段写入），
  因为重启前什么都不生效。保存在进行中时整个表单是一个 `<fieldset disabled>`，
  因此响应绝不会覆盖在其发送之后所做的编辑，也不会把其问题挂到它从未见过的行上。Save
  通过 `environment.update` 发送整个记录；`{ ok: false }`
  把每个问题映射回其字段或行（空行先在客户端被丢弃，使服务器索引对齐）。
  待处理的重启显示 `settings-environment-restart`；启动时设置的名称为带 “Set at launch”
  徽标。纯草稿助手位于 `panels/environmentDraft.ts`；测试渲染的是 props 驱动的
  `EnvironmentForm`。没有文件夹选择器：`dialog.selectDirectory`
  是项目设置的选择器并记录设置分析。
  `ProvidersSettings`/`McpSettings`/`AppearanceSettings`/`LineWidthControl`/`ChatSetting
  s`/`TemplatesSettings`/
  `WorkspacesSettings`/`PrivacySettings`/`ModelsSettings`/`FeedbackSettings`/`Extensions
  Settings`/`EnvironmentSettings`、引导用的
  `ConnectProviderDialog`/`CreateProjectDialog`/`OnboardingWizard`，以及应用级的
  **`SurveyInvitationDialog`** / **`SurveyWindow`** 是面板拥有的**集成件**（store +
  transport）。`SettingsDialog` 从 shell 组合根接收 Layout 区段，因此没有面板会侧向伸入
  shell，而 `LoginDialog` 保持展示型（`auth` 模块）。

  面板组合自己的子面板（例如侧边工具 → `FilesPanel` → `FileTree`/`ChangesPanel`，资源栏
  → 已注册的打包渲染器）—— 一个内部层级。当中心组没有资源标签页时，workbench
  向面板索取该空界面，作为常驻的创建/定向回执而不是通用占位：**“Workspace ready”**、
  显示名称、`branch · from baseBranch`，以及 **“Files, chats, changes, and terminals
  are scoped to this workspace,”**，随后是既有的 **New chat** 动作。对 **Default
  工作区**，回执陈述事实而非承诺隔离：**“Default workspace”**、项目名、`on <branch>`，
  以及 “Chats, changes, and terminals run directly in your project folder.”
  一个**外部工作区**读作 **“Existing worktree”** 并带 `on <branch>`，理由相同 ——
  ThinkRail 不是切出它的，因此没有可声称的 `from <base>`。它既不是一次性也不可关闭，
  因此也能在最后一个标签页关闭后提供帮助，而不引入引导状态。workbench
  资源渲染器通过惰性 **`PlanPane`** 处理已注册的 **`plan`** 标签页（`PlanTab`）——
  聊天计划的**实时 review-map 页面**。前端本地放置只存 `todo-plan` 解析器种类 +
  会话身份，绝不存内联计划内容；
  另一个客户端可以显式重开同一个宿主拥有的页面而不继承放置。

  **TODO 列表是右栏工具。** `LAYOUT_TOOLS` 携带 `"plan"`；
  `WorkspaceWorkbench.toolBodies.plan` 渲染惰性的**`PlanRailPane`**（新文件，
  `apps/web/src/panels/PlanRailPane.tsx`），它**不是** `PlanPane` ——
  它导入聊天视图所依赖的同一组紧凑 `TodoAddRow` +
  `TodoRows`（`apps/web/src/chat/TodoList.tsx`），因此一个界面的视觉变更会落到另一个上。
  `data-testid="plan-rail-pane"` 在其外层 div 上（与中心标签页的 `plan-pane` 不同），使
  e2e 能无歧义地定位任一面。rail 通过
  `selectRailPlanSessionId` 选择该工作区当前活动的聊天并显示该聊天的 TODO；
  当工作区根本没有聊天标签页时，`plan-rail-empty` 占位提示“Open a chat to see its plan
  here.”。`TodoAddRow` 内的 `todo-open-plan` 按钮打开文档尺度的**中心标签页
  Plan**（`openPlan` → `openDoc({kind:"plan", ...})`）；rail 是常驻一瞥界面，中心标签页
  Plan 是深度评审界面（`PlanSessionLive`、PR 动作、下一步横幅、已采纳提交、未归属）。
  二者互补 —— rail 用于任务列表，页面用于评审轨迹。没有布局迁移：既有的中心标签页 Plan
  与新 rail 工具并存工作。

  **尚未完成的 rail 项带悬停 Remove 按钮。** 用户可以从 rail 取消排队中或进行中的步骤，
  通过一个出现在行悬停状态的 `data-testid="todo-remove"` × 按钮；点击调用
  `plan.remove(id)`，它触发宿主的 `todo.remove` 线上调用。宿主的 `removeTodo`
  以解释性错误拒绝 `done` 项（已完成步骤是计划的审计轨迹），因此该按钮在客户端也对 done
  行隐藏。失败路径由 `useChatTodos.remove` 自身拥有：拒绝的宿主抛错，hook
  回滚乐观补丁并通过 store 的错误通道 toast “Couldn't remove the step”，
  因此一次陈旧点击（行在渲染与点击之间刚变为 done）绝不会表现为未处理的拒绝。中心标签页
  Plan 页面**不**暴露该便利操作 —— 它是评审轨迹界面，队列操作位于 rail。agent 仍通过
  pi-todos 的 `todo_remove` 工具管理自己的移除，该工具绕过宿主守卫。

  中心标签页 Plan 以文档尺度渲染会话的 TODO 计划，**按状态分组**（`planSections`）：
  一个 **`Now` 区块**（`plan-now-executing`）持有当前工作 —— 活动组/零散项，
  随后是待处理项（没有单独的 To-do 区段；项状态字形区分进行中与待处理）。
  其**实时状态是头部中、`Session` 标题右侧的一个可点击 chip**（`plan-now-status`、
  `data-glance`，来自 `sessionGlance`），它**打开聊天**（`openChatInTab`），
  因此你能从计划跳到对话：`working` → 一个 `Working on this` spinner，
  `waiting_question` → 一个 `Question` chip。待答问题也**可就地作答**：Session
  正文的**实时槽位**（`PlanSessionLive`，它订阅会话运行时，使其重渲染不落在沉重的
  PlanPane 上）承载与聊天相同的卡片（`plan-ask`，通过 `planView.pendingInteraction`
  找到，并通过工具注册表的 `getToolRenderer(toolName)` 渲染 —— 任何交互式工具，
  不只问卷；该栏不导入任何扩展）。它位于一个最小 `ChatActionsContext` 与派生的
  `InteractionStatesContext` 之中：
  - 动作携带真实的 `session.respondToTool`；
  - 仅聊天的动作（reveal、focus、subagent）是空操作。

  `PlanInteractionCard` 通过聊天使用的同一 `useToolInteraction` 适配器把这些变成卡片的
  `interaction`，因此从计划提交的响应与聊天走完全相同的路径。没有待答问题时，
  同一槽位改为显示**agent 的最新消息**（`plan-agent-message`，
  `planView.lastAgentText` 在有界滚动区内渲染 Markdown —— `max-h-[12rem]
  overflow-y-auto`，使完整消息保持可读而不把计划项往下挤 ——流式期间实时）—— **始终如此，
  包括某步骤正在进行时**，因此计划对 agent 此刻在做什么保持诚实；
  仅当待答情形占用槽位或尚无 agent 消息时它才不渲染。在项目下方，Session 以
  **chat/steer composer** 结束（`plan-session-chat`，一个 `PlanComposer` 文本域，
  工作方式像聊天 composer —— Enter 发送，Shift+Enter 换行），其发送行为适配运行状态：
  agent 流式时它**steer**（`session.steer`，“Steer the agent…”），
  否则它**开始一轮**（`session.prompt`，“Message the agent…”）。没有其 Chat 而被恢复的
  Plan 标签页没有本地运行时，而聊天对账只水合已放置的 Chat 标签页，因此 `PlanPane`
  自身在连接但没有运行时时调用 `hydrateSessionRuntime`（后台，无标签页放置）——
  实时槽位、状态与计划内 ask 卡片都需要它。composer 保持 `ChatView.performSend` 的语义：
  它先等待 `hydrateSessionRuntime`，因此模式来自**已水合**的运行时。**prompt**
  在发送前被记录（`appendUserMessage`）且不被等待 —— `session.prompt`
  仅在运行结束时解析 —— 因此拒绝会作为 `appendErrorTurn` 出现在该聊天中，
  不会丢失任何东西。**steer** 不被记录（它随送达消息一起到达），因此被等待：
  水合失败或被拒绝的 steer 会 toast 并重新抛出，草稿留在计划中。
  **从计划发送仍留在计划上** —— 它绝不自动切换到聊天标签页（早期版本在成功路径上通过
  `openChatInTab` 这么做；已移除，因为计划本身就是一个工作界面，且 agent
  的回复会流进正下方的实时 `plan-agent-message` 区块）。要跳到聊天就点击 Now 状态
  chip（`Working on this` / `Question`）或显式的 Open-chat 便利操作 ——
  这些是现在计划中仅有的 `openChatInTab` 调用点。`PlanComposer`
  在上一次提交仍在进行中时忽略提交，因此重复按 Enter 无法把同一草稿添加或发送两次。
  因此已完成的计划（没有未完成步骤）把其 Session 变成聊天入口，而不是一行死掉的 “all
  steps done”，运行中的计划则获得就地转向字段。唯一的例外是**真正空的**计划（无项、
  空闲）：那里正文显示 `plan-now-idle` 行（`No steps yet…`），
  它本身是打开添加任务输入（一个悬停 `+ Add a task` 提示）以引导计划的点击目标。一个
  **`Done` 区段**（`plan-done-section`，始终展开 —— 该页面就是评审轨迹）持有已完成的组，
  然后是完成的零散项。两个区块共用一个 `PLAN_CARD_CLASS` 卡片形态，带 `glyph + title`
  头部 ——Session（`CircleDot`）、Done（`CircleCheck`，通过 `PlanCardSection`）——
  因此计划读起来是一叠一致的卡片。**标题尺度（自上而下，无反转）：** 页面标题
  `tr-heading-sm` → 卡片 + 组任务标题 `tr-title-dialog`（14/600）→项标题
  `tr-title-section`（14/500）→ 元数据 `tr-text-metadata` ——
  组标题绝不比它所持的项目更小。Now 区块头部携带计划页面的 **add-task
  控件**（计划页面唯一的页内添加方式）：一个 `+ Task`
  按钮（`plan-add-task`）切换一个内联的**自动增高文本域**（`plan-add-input`）—— 朴素
  **Enter 添加**、**Shift+Enter** 插入换行（像 composer 一样多行）、Esc 关闭 ——接在与
  rail 的 `TodoAddRow` 相同的 `useChatTodos.add` 上 —— 一个零散的**用户**项加上 agent
  轻推 ——因此两个入口保持同一流程。项目保持**扫描优先的项目解剖**：项目 TITLE
  是唯一全尺寸文本（`tr-text-ui font-medium`），每个细节都降一级（`tr-text-metadata`，
  弱化/静音）—— 因此标题绝不混入散文。标题**换行**（`break-words`，绝不截断）——
  长标题完整呈现，不被裁切。**done 项不携带前导状态字形**（`hideStatusGlyph` ——
  其区段已说明 “done”；只有活动的 review 或 `changes_requested` 警告保留前导字形；
  该槽位持有幽灵占位符，使标题保持对齐）。**done 项折叠成紧凑区块，当 agent 写了
  `summary` 时始终内联携带其预览**（`plan-item-meta` / `plan-item-summary-preview`）：
  第 1 行是前导 chevron（与变更集展开区的解剖一致；不可折叠的行用幽灵占位符保留 chevron
  的宽度，使列表中每个标题对齐）+ 标题，**review 槽位在其右缘**；第 2 行（当项有
  `summary` 时）是摘要的第一行，以 Markdown 渲染并 `line-clamp-1`，
  使扫描即可知道做了什么而无需点击；第 3 行（当存在 `verification`
  和/或变更集时）是安静的元信息条 —— 验证字形（`ShieldCheck` / `CircleAlert`，来自
  `verificationStatus`，完整验证文本作为其原生 hover tooltip，
  因此鼠标悬停读到确切检查项 ——页内 tooltip 组件会困住焦点并与行的点击展开冲突）、`N
  files`，以及 —— 在 `sm` 以下隐藏，使标题在手机上保持宽度 —— 短 sha + `DiffStatBadge`。
  两行元信息都通过同一幽灵 chevron 占位符与标题对齐。元信息单独占行，
  正是为了让标题行的右缘空出来给 review 槽位 —— 悬停显示的动作绝不与标题、
  摘要预览或元信息碰撞。常驻预览 + 可悬停的验证去掉了为看交付了什么而展开每个 done
  项的成本。**展开是点击，不是悬停**（`plan-item-toggle` → `data-expanded`）：
  详情块始终挂载但被 CSS 隐藏（`hidden` →`group-data-[expanded=true]:flex`），
  因此点击能持久化它（并且在没有悬停的触摸设备上也能用），
  元信息行让位给它（`group-data-[expanded=true]:hidden`）；chevron 也以同样方式旋转。
  没有 JS 悬停状态 ——一个带 mouse/focus 处理器的静态 `<div>` 是 lint
  正确拒绝的无障碍坏味道。详情块是一个缩进的左轨（`border-l`）块，持有备注、agent 的
  `summary`（Markdown）、完整的 `VerificationBadge`（字形 + 以 Markdown 呈现的验证，
  因此多项检查读作要点）、changes_requested 的 `feedback` 备注、变更集，**以及 ——
  当该项累积了 2+ 提交（修复循环）时 —— 一条 REVISIONS
  迷你时间线**（`plan-revisions`/`plan-revision`，来自 `planView.itemRevisions`）：
  每个提交按顺序一行（`#n` + 路由 Changes 面板的 sha chip +
  仍可解析时的 `DiffStatBadge`），最后一行标为 *current*，而 review 的 `unreviewedShas`
  差异中的任何 sha 标为 *unreviewed*（`data-unreviewed`）—— 这是诚实的 agent
  如何走到这里的故事（提交 → review → 修复 → 提交），任何最终 diff 视图都讲不出来。
  任何携带细节的项都可折叠 —— 包括**只有 `note` 的非 done 项**：该备注是面向 agent
  的工作细节，因此留在展开区之后，默认的人类视图是标题 + 状态，绝不是 agent 的内联备注。
  在详情内部，变更集保持其自己的**可折叠**展开区 —— 一行摘要（sha chip + `N files` +
  `DiffStatBadge`）切换该提交的 `GitFileChange[]` 行；chevron/摘要是开关，而 sha chip
  保持独立按钮（路由 Changes 面板，绝不切换）。展开后，文件行在该项的 `commit:{sha}`
  scope 打开注册表分派的 diff 标签页（`openDiffInTab`，preview 意图；路径列表回退在分支
  scope 打开，没有计数因为它们会漂移），**并且 review 判定就在项行本身**：
  该行的右缘是一个 review 槽位，按优先级顺序恰好渲染以下之一：不可点击、脉动的
  `Checking…` 状态（`plan-item-reviewing`，来自宿主推导的 `review.reviewing` —— review
  作为隐藏子 agent 运行，因此没有聊天可打开）、警告性的 `Needs a fix · N` chip，
  或针对未结算可评审项的、主色填充的 `Check it` 按钮（`plan-start-review` ——
  标准的**小号**动作按钮：`h-6`/`tr-text-action`/ `control-primary-bg`，与
  `SendReviewButton` 同样尺寸，而不是过大的 `min-h-8` 块）。
  两个**状态**读数始终可见（是状态，不是动作）；**`Check it` 动作在行的悬停 /
  键盘聚焦时显现**，与 ProjectTree kebab 完全一样：`[@media(hover:hover)]:opacity-0` +
  `[@media(hover:hover)]:group-hover:opacity-100` + `focus-visible:opacity-100` ——
  因此在桌面上，一墙的主色按钮绝不会涂满每个可评审行，
  而在**触摸**设备上（无悬停）它保持可见，并且它绝不会像 `group-focus-within` 那样粘住。
  它是标题行上的**流内**按钮（第 2 行的元信息腾出了那个右缘，因此标题只是为它收缩 ——
  无重叠、无空的保留槽位）。仍是一个槽位，没有重复 —— 变更集展开行**不**携带 review
  便利操作。`Check it` 触发 AGENT review（`todo.startReview` —— 一个隐藏的 review 子
  agent）并**留在**计划页面上：行的 `Checking…` 脉动加一个 toast 是唯一信号，
  成功与失败皆然 —— review 没有自己的聊天，因此 toast 必须携带错误。判定通过
  `review.changed` 广播落地（`useChatTodos` 在其上重抓计划），而不是本会话的 `pi.event`
  ——子 agent 的事件被隐藏；确认后的失败通过 `review.failed` 广播落地（`useChatTodos`
  把它作为错误 toast 抛出，按所属 `sessionId` 过滤，并在分屏视图间按 toast 正文去重）。
  **Plan-review 的 STATE 始终从计划推导；只有 ACTIONS 以
  `transport.supportsPlanReview`（v67）为宿主版本门禁。**
  `reviewables`/`unsettledReviewables`/`planReady` 无论宿主版本都来自 `TodoItem.review`
  ——把它们门禁为空会让 `planReady` 在某个未评审步骤上读作可交付。面对不提供
  `todo.startReview`/`reviewAll` 的旧宿主，`PlanPane` 只禁用变更型便利操作（每行 `Check
  it`、两个 `Check All` 触发器），
  因此独立发布的较新客户端绝不对宿主无法履行的能力*发起调用*，
  同时仍反映宿主确实报告的评审状态。行控件（`plan-item-toggle`、变更集开关、sha chip、
  review 槽位、`FileRow`）都带 `min-h-8` ——密集的元信息行在触摸上仍可点。
  `planView.changeSetCounts` 是唯一的计数/统计推导（路径 → 仅计数；提交
  →`changeSetStat`），被行的元信息条与展开行共用。**没有页内手动判定 UI** —— 原先的
  `manually` 开关+ `ReviewActions` 对（Approve / Ask to fix）已随 `PlanReview.tsx`
  移除；`todo.review` / `todo.requestFix` 线上方法与宿主处理器保留，
  因此手动覆盖界面可以在不做协议工作的情况下回归；agent 撰写的发现出现在 Review
  面板中并标有 `agent` 徽标（`review-comment-agent`），agent 结算的卡片读作 `Reviewed ·
  agent`；**changes_requested** 判定会大声标记该项：状态字形翻为警告
  `CircleAlert`（`StatusIcon changesRequested`、`data-changes-requested` —— rail
  行与计划页面皆然），计划页面的标题行长出一个警告**`Needs a fix · N`**
  chip（`plan-item-changes-requested`；N = `planView.itemOpenFindings`，
  评审人未关闭的评论按 `origin` 来源匹配（无来源的用路径连接回退）—— Review
  标签页才是真相；该 chip `requestToolView`s 到 Review 标签页）且判定的 `feedback`
  备注内联渲染（`plan-item-review-feedback`）；批准会结算该项 —— 计划页面在该行的
  review 槽位中、标题右侧显示 **`Checked` 标签**（`plan-item-verified`，`CircleCheck` +
  文本，成功色调），并在行上带 `data-reviewed`（rail 保留圈起的 `StatusIcon reviewed`
  字形）；`planView.reviewSettled` 是唯一的推导 —— 已批准且无未评审差异，
  因此新修订会让该项同时退出标签与已评审计数。**头部是标题 + 生命周期 STEPPER 和一个
  kebab 菜单**。stepper（`plan-progress`）渲染计划的交付漏斗 ——**Build（`d/t done`）→
  Review（`r/k reviewed`，`plan-review-progress`，仅当计划有可评审项时）→
  PR（`plan-pr-stage`，`data-state`）** —— 每个阶段为其状态佩戴一个字形：done（对勾）、
  active（计划当前所处的阶段）、pending（弱化）。PR 阶段读取与按钮相同的
  `useBranchStatus` 查询，并在 PR 打开后显示 `PR #N`；“merged” 不可知（查询只看到 OPEN
  评审），因此漏斗诚实地止于 PR 打开。stepper 下方是**工作 CONTEXT
  行**（`plan-context`）：`baseBranch ← branch · N commits · +A −R` —— 箭头指向合并
  TARGET（base ← head，GitHub PR 惯例：变更从工作区分支流入其基础分支）。`N commits` 是
  **`PlanCommitsMenu`**（`plan-commits-trigger`）—— 一个镜像 Changes scope
  菜单提交列表的下拉：`git.listCommits`（急切加载，
  每当计划的提交计数跳动就重载）是计数与列表的唯一来源，因此二者绝不背离 —— 而且，作为
  `base..HEAD` 枚举，它已经涵盖已采纳提交（没有步骤拥有的分支提交）与每步提交；
  每行（`plan-commits-item`、`data-sha`）通过每步提交 chip 使用的同一 `openChanges({
  sha })` 在 Changes 面板中打开该提交的 diff。该 chip 在加载中以及分支无提交时自行隐藏。
  总 diff 来自工作区记录的 `diffStats`；每部分在未知时隐藏。
  头部与摘要之间是**下一步动作横幅**（`plan-next-action`、`data-kind`）——
  报告的唯一“现在做什么”，按紧急度渲染第一个匹配状态：`fix`（N 个步骤携带
  changes_requested →**Show step** 滚动到第一个被标记的项并通过 `focusRequest` token
  自动展开它 —— `{ id, tick }`，tick 每次点击递增，并由目标 `ItemBlock` 每个 tick
  消费一次，因此再次点击会重新展开手动折叠的行，
  而陈旧请求无法在之后重开它）→`review`（N 个未结算可评审项 → 一个内联 **Review All**
  按钮，与 kebab 项相同的 `todo.reviewAll` 流程，kebab 项保留）→`ship`（全部完成 +
  已评审、无打开的 PR → 一个内联 **Open PR**，与头部按钮相同的 `pr.open`
  流程）→无任何要求动作时隐藏。计划级完成备注佩戴 `Summary` 眉标，
  使报告读起来是带标签的区段；当备注较长时它钳制为两行，
  而其**展开/折叠开关位于卡片头部**（可点击 `plan-overall-summary-toggle`
  头部上右对齐的 chevron，随 `open` 旋转）——不是尾随按钮 ——
  因此折叠永远无需滚过已展开的散文。下一步动作横幅、Summary 区块与 Now-executing
  区块共用一个卡片形态（`PLAN_CARD_CLASS` —— 相同的高架背景、边框、圆角与内边距），
  使计划顶部读作一叠一致的卡片。在项区段之后，页面渲染 **`Committed outside the
  plan`**（`plan-adopted-commits`，仅当 `TodoPlan.adoptedCommits` 非空时 ——
  包括在否则为空的计划上）：宿主推导的、没有项拥有的 `base..HEAD` 提交（推导：
  [[submodule-server-todos]]），每个用与计划步骤相同的 **`ItemBlock`** 渲染，
  因此它携带完全相同的变更集、Start-review 与修订便利操作 ——它们与某项的提交一样可评审。
  **仅 Review 阶段：** 它们被*排除*在 Build `d/t done` 计数之外，也从不门禁 `ship`，但
  `planView.reviewableItems` 包含它们，因此 Review stepper、`review` 下一步动作与
  Review All 都覆盖它们。然后页面渲染 **`Outside the plan`**（`plan-unattributed`，仅当
  `TodoPlan.unattributed` 非空时 ——包括在否则为空的计划上）：宿主推导的、
  没有任何项声称的未提交行（推导：[[submodule-server-todos]]），渲染为打开**未提交
  scope** diff 的 `FileRow` —— 这个诚实区段让未计划的工作在 review map 中保持可见，
  而不是读作“没有其他变更”；`chat/planMarkdown` 把它导出为自己的区段。kebab
  菜单（`plan-menu`，一个 `DropdownMenu`）持有 **Copy**（剪贴板）/ **Save
  .md**（浏览器下载）—— 二者都通过 `chat/planMarkdown` 编译 ——以及，当计划有可评审项时，
  **Review All**（`plan-review-all`）：触发 `todo.reviewAll`，
  它在计划的串行链上对每个*未结算*可评审项做 agent review，一次一个（无未结算项时禁用；
  一个 toast 报告启动了多少个，每行的 `Reviewing…` 脉动跟踪进度），外加 **Open draft
  PR**（`plan-open-draft-pr`，PR 存在后隐藏）。**头部还拥有计划的终点线 —— Open
  PR**（`plan-open-pr`、task-open-pr）：一个确定性的宿主侧流程（push + `gh`，绝不是
  agent prompt），**仅在首次创建时**（`openReview` 缺席）经过**compose
  对话框**（`PrComposeDialog.tsx`、`pr-compose-dialog`）：点击抓取 `pr.preview`
  并打开可编辑的 Title（`pr-compose-title`，从聊天的**真实**名称预填 —— 绝不是其首个
  prompt 显示标签；未命名聊天留给宿主的分支回退）+ Description（`pr-compose-body`，
  从计划预填）字段；只有提交（`pr-compose-submit`，标签跟随动作 ——Open PR / Open draft
  PR）用编辑过的 `title`/`body` 运行 `pr.open`。对话框在成功时关闭，
  在一般失败时保持打开（编辑经 toast 存活），并在 `PUSH_AUTH_FAILED` 时移交给
  `PrSetupDialog` —— 其 Try again 重新提交最后编辑的 title/body（保存在 ref 中），
  绝不是重新渲染的草稿。当计划*就绪*（全部完成 + 全部评审结算）时头部按钮是主色填充，
  否则安静；一旦存在打开的 PR（在 v78 上宿主保留的快照拥有身份/状态，匹配的
  `workspace.openReview` 回答覆盖实时细节；旧宿主单用查询，都通过面板拥有的
  `resolveBranchReview`，使 shell 与 Plan 不可能有分歧）标签翻为 **Push updates**，
  且该按钮**完全绕过 compose 对话框** —— 按下它（或下一步动作的 `push` 分支）直接调用
  `pr.open` 且不带 `title`/`body`，因此宿主推送到同一分支/PR，
  并静默从计划刷新其正文（`renderPrBody`），同时不动 PR 标题（无 `titleEdited`）。
  每次推送都重新编辑 PR 描述读起来像“再设置一次 PR”；模态框只是创建便利操作。当查询报告
  **`unpushedCommits`** 时标签追加计数（`Push updates (N)`），按钮变为主色填充，
  下一步动作横幅长出一个 `push` 分支（“N new commits aren't in PR #N yet” + Push
  updates），因此 PR 之后的新工作绝不静默留在本地 ——
  成功推送会重读权威状态并在远程跟踪分支追平后清除两者。当查询改为报告
  **`behindCommits`**（origin 有 HEAD 缺少的提交 —— 分支**已分叉**，
  因此朴素推送是非快进且会失败）时，一个独立的**分叉**状态优先于 push 分支，
  并把它当作**同步冲突，而不是 force-push 提示**：头部按钮读作**Branch
  diverged**（`data-diverged`），下一步动作横幅（`data-kind="diverged"`）说明 origin
  有检出缺少的提交、必须先集成 —— 朴素推送无法落地，而 force-push 会**丢弃远程的提交**。
  两者都复制安全的 `git pull --rebase origin <branch>`（`plan-integrate-command`），
  而不是 force 命令 ——它点名分叉所对比的确切 ref，而不是所配置的上游。
  分支仅在它在所有受支持 shell（POSIX、PowerShell、cmd：
  `[A-Za-z0-9][A-Za-z0-9._/-]*`）中都是 shell 惰性时才被插值；
  其他任何名称得到纯文本指示且没有可复制命令，因为没有单一引号方式在这些 shell
  间都安全。这是刻意的：`behind > 0` 只证明分叉，
  **不**证明该检出改写了历史（另一个检出可能只是推送了），而宿主自己的新抓取已移动了
  `--force-with-lease` 基线 ——因此从分叉推断 force-push 可能静默删除另一个检出的工作。
  真正的改写仍是显式终端动作，应用绝不发起，也绝不提供已加载的命令。`behind`
  仅在新鲜查询上被信任（宿主此时抓取 origin；见 [[submodule-server-git]]）；
  该字段是增量且向后兼容的（旧宿主省略它 → 先前行为）。分叉逻辑在 `countPushDivergence`
  有单元覆盖；分叉 UI 状态不被浏览器 E2E 测试装置覆盖，因为它无法伪造一个被检测到的打开
  PR（真实 `gh` 查询）。还有一个 **`PR #N` chip**（`plan-pr-chip`）在 URL 已知时外链 ——
  现在每次读取都如此，因为 `workspace.openReview` 携带评审自己的
  `url`（[[submodule-server-branch-review]]）；带 key 的状态优先用它，
  回退到来自同一评审更早回答的 url，因此 chip 绝不会在报告没有 url 的刷新中丢失链接。该
  hook 拥有唯一的带 key PR 状态：`noteOpenReview(review, url?)` 在 `pr.open`
  之后立即播种它（页面中没有单独的影子状态），并取代同一 key 的任何进行中读取，
  因此变更前的回答无法覆盖变更结果。带 key 的状态与请求代际被每个已挂载 hook
  消费者共享，使计划与 shell scope 标签保持在同一个变更结果上。在该 hook
  移到另一分支之后，陈旧的变更闭包无法写穿。聚焦重抓会覆盖状态。工作区激活通过
  `allowCached: true` 显式选择加入宿主的 60 秒已结算回答缓存，
  而聚焦时的省略为旧客户端保留线上契约原本的强制新鲜行为，因此在 GitHub 上关闭/合并的
  PR 会在那次重抓时从 chip、标签与 stepper 中消失，而不是粘到重挂载。
  断开期间收到的聚焦会锁存该新鲜意图，并在重连时使用它，
  而不是回退到符合缓存条件的激活读取。工作区的 `fsChanged` 跳动以 `allowCached: true`
  重读（`reload`）：宿主在 `.git` 元信息轻推时丢弃其缓存回答，因此从终端打开或推送的 PR
  仍会浮现，而新鲜读取会 `git fetch`、写 `.git`、再次轻推、成环。只要评审编号匹配，URL
  就跨多次重抓保留。`compare` 结果打开预填的 GitHub compare 页面（`window.open`）；
  每种结果都 toast，未提交文件得到单独的 info toast。`pr.open` 请求以 **180s
  超时**运行（push + gh 变更可能超过传输层 60s 默认值），且任何 PR
  工作在进行中时头部按钮佩戴 spinner —— **Pushing…** 标签只在真正提交期间（preview
  抓取不是推送，其失败 toast“Couldn't prepare the PR”，绝不是 “Open PR failed”）。
  未提交文件的 info toast 在结果分支之前触发一次。每次成功提交在播种 `pr.open`
  返回的任何评审之后，启动一次新的共享 open-review 读取。
  它绝不把变更载荷当作最终的未推送计数：
  推送后的读取决定计数已清除还是更新的本地提交又让它非零。compose
  提交还报告标题是否被触碰（`titleEdited`），因此宿主绝不会用重新生成的预填覆盖 GitHub
  侧的重命名。**点名可修复设置缺口的失败会打开 `PrSetupDialog`（`PrSetupDialog.tsx`、
  `pr-setup-dialog`）而不是 toast**：一个 `PUSH_AUTH_FAILED` 拒绝（通过传输层的
  `wsErrorCode` 匹配）说明宿主在没有终端的情况下推送，并显示 git
  的 stderr（`pr-setup-detail`）加可复制的修复（SSH 用 `ssh-add --apple-use-keychain …`，
  HTTPS 用 `gh auth login`）；一个携带 `ghProblem` 的 `compare` 结果说明缺失/未认证的
  GitHub CLI 并给出安装/登录命令，并作为对话框内链接（`pr-setup-compare` —— 真实锚点，
  因此没有弹窗拦截风险）提供 compare 页面，而不是盲目的 `window.open`。
  两个变体都保留一个 **Try again**（`pr-setup-retry`），它重跑同一流程 ——
  始终用最后编辑的 title/body（runner 的 `lastSubmit` ref，每次提交都设置，
  仅在流程完全成功、用户显式取消 compose 对话框，或当他们接受 compare
  页面移交（客户端无法观察的外部完成）时才清除 —— 否则 `ghProblem` 结果保留它，因此 gh
  对话框的 Try again 确实重新提交；失败后重开 Open PR 复用那些编辑（包括已编辑标题基线，
  因此重开的草稿不丢其 `titleEdited` 标志）而不是重抓一个重新生成的草稿）；命令行通过
  `copyText`（`pr-setup-copy`）复制，并携带一个 **Run**（`pr-setup-run`），
  它关闭对话框并通过 store 的 `addTerminal(workspaceId, initialCommand)`
  在一个新的工作区终端中执行该命令 —— pty 是真实的交互式 shell，
  因此口令/登录提示就在那里回答，而不是让用户去找外部终端。
  命令集是**宿主平台感知的**（welcome 的 `hostPlatform`，在接收时校验并在每次 welcome
  时重置，因此切换宿主不会留下陈旧平台；**没有 darwin 回退** ——
  null/未知平台得到通用命令：朴素 `ssh-add`、不猜包管理器、一个 cli.github.com
  安装提示）：macOS 上 `ssh-add --apple-use-keychain` / `brew`，Linux 上朴素 `ssh-add` +
  发行版提示，Windows 上 `$env:USERPROFILE` + winget ——命令在 HOST 上运行，
  因此浏览器自己的平台从不被咨询。匹配 `Host key verification failed`
  的推送认证细节会添加一行批准主机密钥（交互式运行 `ssh -T git@github.com`）——
  ssh-add/gh 补救措施修不了 known_hosts。一个*没有* `ghProblem` 的 `compare`（离线接缝、
  瞬时 gh 失败）保留 window.open + toast 路径。该对话框在服务器侧被单元/e2e
  钉住（`isPushAuthFailure`、`ghSetupProblem`）；浏览器侧分支需要真实损坏的推送 /缺失
  gh，因此它们靠约定保持。**`Summary` 卡片**（`plan-overall-summary`）只显示 agent
  的计划级备注 —— 当每个步骤都完成时通过 `planCompletionSummary` 新鲜获取，
  或在计划被重做期间通过 `planStaleSummary` 获取陈旧备注。
  它**不携带任何步骤/文件/评审事实**（那些位于头部 stepper + 上下文行，
  在这里重复只是噪音），且**当没有散文时完全省略**（agent
  未写备注的已完成计划不渲染空卡片）。agent 的备注仍通过 `planCompletionSummary` 进入
  Copy / Save-.md 导出。**没有页内 “Review mode”** ——发现位于右侧面板的 **Review**
  标签页；当评审 agent 有未关闭评论时（`selectAgentReviewCommentCount` —— open、
  `author: "agent"`）头部显示一个 **`N comments`** chip（`plan-review-comments`），它
  `requestToolView(ws, "review")` 以聚焦该标签页。一旦某项在完成后重开，Summary 备注通过
  `planStaleSummary` 保持可见，标为陈旧并带 `Updating…`
  徽标（`plan-summary-stale`）而不是消失，直到 agent 在下一次完成时重写它 —— 导出仍以
  `planCompletionSummary` 为门禁。`FileRow`（`planFileRow.tsx`，独立模块，
  使计划界面共享同一行而不成环）是共享的变更集行。按构造即实时，它通过与 TODO rail
  相同的 `useChatTodos` hook 读取（每次挂载抓取 + `pi.event` 重抓），
  因此不可能显示陈旧快照。`TerminalWorkbench`
  为每个语义终端身份拥有一个可见性门禁的终端 body，以及宿主原子的关闭流程。
  忙碌关闭在确认与强制重试之间仍是一个相关联的请求；对话框自动关闭无法释放该请求，
  权威目录移除会关闭陈旧确认，而被拒绝的强制关闭精确清除该请求并带一个错误，
  使之后的关闭可以干净开始。聊天的 workbench
  关闭命令路由到 `store.closeChatToHistory`（保持会话存活），并显示一个 **chat-history**
  下拉（最近关闭的 + 仅磁盘的聊天，仅非空时显示）；每行有一个一键 trash
  动作（`session.delete` → 幂等 `store.deleteChat`，无确认）；`session.deleted`
  广播在每个已连接客户端驱动同样的折叠。在工作区激活与每次重连时，
  `session.list` 先对账读取开始时捕获的客户端成员快照（运行时/缓存身份加上已放置的
  chat/TODO 文档引用），因此一个现已从权威结果中缺席的基线会话走正常 tombstone
  与本地放置修剪，而读取期间创建的聊天得以存活。本界面本地工作区视图所引用的聊天通过
  `session.getMessages` → `messagesToRuntime` → `store.hydrateSession` 水合。
  每个其余会话进入本地历史而不打开或选中标签页：另一个前端创建或使用聊天是领域活动，
  而不是放置指令。转录读取失败会抛出错误 toast 并使该摘要在历史中可重试；`session.list`
  失败也会抛错，而不是呈现一个无解释的空工作区。一旦对账被取消、断开或归档，两个 toast
  都归于沉默。实时水合刻意不携带当前磁盘的 skill 基线；只有仅磁盘的附件接收其捕获的
  `syncedTick`。`session.deleted` 在每个客户端驱动同样的幂等运行时/历史/本地放置折叠；
  不存在当前布局推送。重开历史行会把其既有会话身份添加到从该 Group Header
  捕获的请求时中心目标（包括空组）；被拒绝的读取让该行留在历史中并抛出错误 toast。
  workbench shell 集成还解析历史搜索的**`chatLocationRequest`** 深链（见
  `store/SPEC.md`）：一旦其工作区是活动的，它会聚焦已打开的标签页、`reopenChat`
  一个活动但已关闭的、或抓取 + 水合一个仅磁盘的 —— 即上面重开流程的两种情况，
  加上针对已打开标签页的第三种 ——把请求留给 `ChatView` 消费以做滚动 +
  闪烁（`chat/SPEC.md` 的 Jump-to-message 条目）。**`Toaster`** 是 shell
  挂载一次的应用级 toast 宿主：它订阅 `store.toasts` 并通过 `@thinkrail/ui/toast`
  原语渲染每一个，让 Radix 拥有自动超时 + 滑动/悬停暂停，并把每次关闭路由回
  `store.dismissToast`（因此 store 仍是单一真相来源）。错误会持续到被关闭；成功/info
  会超时。它是**集成件** —— 原语保持展示型。
- **公开接口：** 布局无关的功能渲染器（`ProjectTree`、`WelcomePanel`、
  文件/diff/文档/聊天栏、单例侧边工具、终端 body、Settings，以及 `Toaster`），
  **按文件**导入，使 Monaco/shiki/xterm 保持惰性。标签条、组头部、
  侧边栈与中心拓扑不是面板接口；shell 布局模块包装这些渲染器。
- **允许的依赖：** `store`、`transport`、`components`（`SkeletonRows` ——
  每个异步面板的待处理状态渲染内容形状的 skeleton 行，绝不是裸的 “Loading…” 行）、
  `@thinkrail/ui/*`（含对话框用的 `popover`/`command`/`textarea`）、
  `chat`（`ModelEffortPicker` + 喂给它的 `useModelCatalog`/`useModelPreferences` hook，
  被 `NewWorkspaceDialog` 复用；`ModelSelector`/`ThinkingSelector`，仍由
  `ModelsSettings` 挂载；`modelPicker` 的 `AUTH_KIND_LABEL`，即唯一的连接种类词汇（目录
  key）`ProvidersSettings` 与选择器共享，以及其 `formatContext` 加 `ProviderGlyph`，被
  `VisibleModelsSettings` 的行复用；`TemplateEditorDialog`，被 `TemplatesSettings`
  复用）、`auth`（`LoginDialog`，通过 `useProviderAuth` 与 `McpSettings` 挂载）、
  `@thinkrail/resource-renderers`、`lib`、`themes`（目录 + 通用应用契约）、`contracts`；
  `@remixicon/react`；以及每个惰性面板自己拥有的重型库（`shiki`、`@xterm/*`），通过
  `import()` 加载。
- **禁止：** `server`/`shared`/`pi`；导入 `shell`；跨入无关面板。

- **Ship —— 分支的交付状态与动作，顶栏与 Plan 栏一份实现。**
  `useBranchStatus(workspace, status)`（带 key 的 `branchStatusState`、请求代际、
  `noteOpenReview`、上面的聚焦/fs/重连新鲜度规则原样沿用）在 v87 宿主上读取
  `workspace.branchStatus`，在其下回退到 `workspace.openReview`。仅在 welcome
  已命名协议后才算已连接，因此重连绝不会向新宿主询问旧方法。它暴露查询的评审 ——
  调用方用 `resolveBranchReview` 对照快照解析它 ——并把 v87 的 ahead/behind 折到其上，
  外加无评审的 `BranchFacts`。`shipActions(t, context)` 是每段 `{ primary, secondary }`
  的纯推导（标签用调用方的 `useT()` 翻译器渲染）—— 远程分支（Publish / Push N /
  Sync…）与 review 分支（Open PR / Create MR / Open draft PR / 被禁用的 Merge-locally
  桩 / Settle；在 v88 宿主上为打开的 GitHub PR 提供 Fix with agent / Ready / Merge PR，
  以及其后的 Reopen / Delete remote branch）——在完整场景表上单元测试过，它构建的 agent
  prompt 文本（`fixChecksPrompt`、`integratePrompt`）亦然。`useShipRunner`
  执行它们并拥有其对话框：共享的 `PrComposeDialog`（现在也用于从顶栏首次创建）、
  `PrSetupDialog`、分叉 Sync 对话框、基础推送确认、`MergePrDialog`，
  以及远程分支删除确认。合并对话框及其 toast 从 `details.base` 命名 PR 的当前基础分支，
  绝不是工作区的基础分支（PR 可在 GitHub 上被重定向），并在未知时不做目标断言。Review
  动作作用于调用方显示的 `review`（其编号）以及 runner 所绑定到的分支。**Merge PR**
  只提供仓库允许的方法（宿主无法判断时提供全部方法），默认使用该项目上次使用的方法 ——
  `mergePrefs`，按宿主 + 项目存在于客户端（一个项目即一个仓库），回退顺序为 squash →
  merge → rebase；*delete the branch on GitHub* 与之一起被记住并默认开启，在 GitHub
  自己删除已合并分支时隐藏。**Fix with agent** 与 Sync 对话框的 **Ask agent to
  integrate** 通过 `openChat` 的 `promptWorkspaceChat` 把 prompt 交给 runner 的聊天 ——
  该聊天流式时为后续消息，否则为一条 prompt；没有聊天时则与之一起新开一个 ——
  并显示该聊天。集成可以 rebase 或 merge，但绝不 force-push 或丢弃提交，
  并把推送留给用户。Plan 栏的 Open PR / *Push updates* / 分叉处理走同一 runner，
  因此上面的 `lastSubmit` 复用、`PUSH_AUTH_FAILED`、`ghProblem` 与 compare
  移交规则只有一个归宿。**Try again** 重跑真正失败的那个变更 ——PR 提交或推送，
  在失败时记录、在设置对话框关闭时丢弃 —— 绝不是用户最后撰写的那次 PR；
  `lastSubmit` 只恢复 compose 对话框的编辑，包括其描述来源，
  且只在它们被撰写时所针对的分支上。每次变更都携带它为之进行的分支 ——compose 对话框在
  preview 时的分支、基础推送确认中点名的分支、重试时失败尝试的分支 —— 一旦检出已移动，
  宿主就拒绝它，因此无论终端切换还是重试都无法推送一个用户没看到或未确认的分支。Plan
  栏单独显式请求其计划正文（`bodySource: "plan"`，
  因此只有已采纳提交的计划仍会在其上打开），并保留其计划专属强调与刷新正文的*Push
  updates*（不带 body 的 `pr.open`）；顶栏的 Push 是 `workspace.push`，绝不触碰 PR
  描述。**Compose 来源：** 对话框的 *Description from* 开关用 `source` 重抓
  `pr.preview` ——*Chat plan*（会话有计划时的默认；从顶栏取活动聊天的会话）、*Commits*、
  *Template*（宿主发现的 PR/MR 模板，一个选择器）、*Empty* —— 而 **Generate with
  agent** / **Fill template with
  agent**（`pr.generate`）把字段文本替换为宿主的一次性草稿；提交前不推送任何东西，而
  `null` 草稿会 toast 而不是清空字段。

## 必须做对（Get right）

- **Workbench 标签页外壳不是功能面板。** shell 布局模块为每个组提供一套选中标签语法：
  整个可选标签背后的 `control-bg-selected`、`text-default`，以及底缘上**横跨标签全宽的
  2px `primary` 标记**（`after:inset-x-0`，与选中填充齐平 —— 无水平内缩）。
  非活动标签保持透明并带弱化文本；悬停使用 `control-bg-hovered`；
  键盘聚焦保留其独立的焦点环。标记是形状线索，而不只是文本颜色变化，
  因此当高对比主题让相邻表面变得相等时，选中仍显而易见。
  该语法还提供有界单行溢出与完整的 WAI-ARIA 标签页模式，含 roving focus 与带标签的
  tabpanel。面板渲染器提供标题/图标/状态/关闭元信息并填充选中的 tabpanel；
  它们从不读取组顺序，也不绘制自己的停靠条。共享的 `ToggleSegment`（List|Tree、
  Split|Inline，以及解析后的资源候选）为其活动段借用相同的 `control-bg-selected` 填充 +
  `text-default`（无底缘标记 —— 纤细切换而不是标签页），因此“选中”处处读法一致，
  绝不派生平行的表面 token。
- **文件与 diff 栏只分派；它们不分类格式。** 每个都描述宿主元信息、
  为其意图与手机类别解析注册表、惰性挂载所选候选，并在标签页上保留 `rendererId`
  加不透明视图状态。惰性实现身份包含 renderer id 与手机类别，
  因此跨越断点会切换代码实现并丢弃不兼容状态。滚动支撑的视图状态经过渲染器唯一的
  `useScrollViewState` hook（[[module-resource-renderers]]）：偏移在滚动器分离时保存，
  并在滚动器附着时重新应用。渲染器只能在挂载后填充其滚动器 ——Pierre 界面在其共享 worker
  池初始化前不渲染任何东西 —— 因此滚动器尚无法容纳的恢复会保持待处理，
  直到其内容增长到适配或用户滚动，而如果标签页先离开，保存的就是那个待处理偏移。
  没有这个保持，切回源 diff 的标签页会落在顶部并随后持久化 0。
  两个或更多候选变成一个有序列切换，其 id 是测试钩子。若所选渲染器无法为该栏的 view 或
  diff 意图放置某些线程，它们仍在一个未放置条中可见；
  其动作切换到第一个声明匹配该意图锚点几何的候选。打包注册在 shell 的 `AppResourceHost`
  中发生一次，而渲染器实现的导入保持惰性。
- 单例侧边工具渲染器是 **Projects | Specs | Files | Review**（变更文件列表是 Files 的
  **Changed** 模式，不是独立工具）。其当前位置与本地选择由 shell 提供；Review 把其
  store 推导的待处理草稿计数作为标签元信息暴露。当单例移动到相反一侧时，渲染器保持相同。
- **`ReviewPanel`** 是 review 侧栏（模型见 [[submodule-server-reviews]]）—— **一个屏幕，
  一个按文件的手风琴**：每行一个路径 + 草稿/已发送/已结算计数，带一个折叠 chevron；
  **点击行就地展开其评论并打开该文件的标签页**（折叠是第二次点击且不导航 ——
  行是唯一的开关；行上唯一的另一个动作在下方）。所有评论都已结算的文件**仍被列出**，
  直到用户显式完成它：行本身长出一个 **Done 对勾字形**（计数之后内联 ——
  无论折叠与否都可见；下方一个只放一个字形的条读起来像 stray space），它调用
  `review.fileDone`，只有它才移除该文件（`Review.doneFiles`；新评论会重开它）。
  展开的区段以 TODO 计划的确切区段流显示该文件的评论，
  由共享的计划原子构建（`chat/planKit`：`SectionLabel` + `PlanStatusIcon` —— 与
  `TodoList` 渲染的是同一批组件）。**评审 agent 的评论与用户的评论走同一套生命周期** ——
  唯一差别是一个作者徽标（`review-comment-agent`：一个 `Bot` + “ThinkRail” chip）：
  它们共享相同的区段、字形、导航、
  每行发送/删除/结算与草稿计数（`fileDraftIds`/`allDraftIds` 与宿主的隐式
  `sendableComments` 都与作者无关；判定的 fix 包仍会扫过用户尚未发送的任何 agent
  草稿）。区段按状态 —— **Drafts** 在前（用户可操作、未发送的 remarks；
  行动号召位于该文件的 `Send review (N)` 条下，而不是埋在已发送行下面）→ **In
  progress**（已发送的 ——聊天接管了它们；字形与 TODO 的进行中项完全一样是 GLANCE-AWARE，
  通过 `sessionGlance` + `TodoList.glanceIcon`：working 点 / 会话等待
  `ask_user_question` 时的 **(?)** / 在用户处空闲时的暂停 ——没有加载的运行时读作等待）→
  **Resolved**（弱化的 Done 样式：主色对勾 + 删除线提示文本；聊天动作在悬停时显现 ——
  resolved 是终态，不重开）。没有逐行状态词 —— 区段即命名状态；行只携带字形、钳制文本与
  `L3` ref（外加锚点失效时的 `outdated` 眉标，其原生 tooltip —— `outdatedReason`
  来自[[module-resource-renderers]]，与线程卡片的标签共享 ——
  通过锚点自己的选择器说明什么无法重新找到：未能再次匹配的文本、发生变化的一个字节位置，
  或一个整体消失的文件。线上契约不携带原因，只携带状态，
  因此文本与字节两种措辞都承认“或文件已消失”，而不是声称客户端并不具备的精确度；
  规则本身是[[submodule-server-reviews]] 的重新锚定第 4 步）。
  本地选中的中心资源若是一个已评审文件，其区段会**自动展开**，
  而展开绝不自动折叠（折叠只是用户的手势 ——
  一次发送打开其聊天标签页绝不能折叠用户正在读的区段）；**Drafts 行被编号**（1.、2.、
  …）而不是佩戴待处理字形 —— 而 workbench 工具路由器在此类标签页被 ACTIVATED 时**揭示
  Review 工具**（以本地选中资源变化为 key，
  因此在已选中资源中保存的草稿绝不会猛拉注意力；`selectActiveReviewedPath`
  是共享的推导）。每个评论行都是一个**导航手势**，对所有作者一条规则：
  带链接聊天的行（`comment.sessionId`）打开**该讨论**（其聊天标签页）；没有链接的 ——
  每个草稿都如此，因为草稿从不被发送 ——打开文件并**聚焦到该评论**。
  文件仍只差一个悬停动作（`FileText` 字形运行文件 + 聚焦导航；
  聊天字形已从打开的行中移除）。文件聚焦通过 store 的 `reviewFocusRequest` 生效，
  它命名该行打开的界面（`inline` 文件，或带其 scope 的 `diff`），
  因此已挂载的另一种栏或另一个基础分支无法消费它。该栏恰好消费它一次：Monaco
  揭示锚点行，包括在全新挂载时（从挂载的 layout effect 立即滚动；见 resource-renderers
  SPEC 的挂载规则），预览把流内卡片滚动到视野内。**此处不编辑** ——
  文件内卡片才是编辑器；该行的动作图标（各自一层，绝不触发导航）是逐行 **Send**（→
  `review.sendComment`，通过与新工作区相同的 `openChatSession`
  尾部打开创建出的聊天标签页）、**DRAFT 行的 Delete**（ConfirmPopover →
  `review.commentDelete` —— 未发送的 remark 是用户自己的草稿纸）、已发送行的 **Open
  chat**（复用历史重开流程），以及手动 Resolve 覆盖（`review.commentUpdate`）。
  **一旦发送，评论就是记录 —— 不删除、不回滚、不重开**，而 resolved
  是终态（服务器强制）：对某个变更的反驳写在评论里，新的意见就是新评论。**一个普通列表
  —— 无页脚**：批量发送位于栏工具栏（`SendReviewButton`）和面板本身 ——
  每个展开区段的条携带相同的按文件 `Send review (N)`（`testid: review-panel-send`；
  `path: null` 覆盖无锚点的整个变更集桶），面板头部是一个跨所有文件的**`Send all
  (N)`**（`SendAllReviewsButton`、`testid: review-send-all`，基于 `allDraftIds`；不传
  id ——宿主的“所有草稿”就是该批次，因此计数无法与并发编辑竞态）。**头部（及其
  Clear）跟随 review 的 RECORDS，而不是其文件行**：只要 review 持有任何评论它就显示，
  因此完成每个已评审文件 —— 这会清空手风琴而 resolved/sent 记录仍在 —— 仍留下关闭 review
  的途径（早先按文件门禁的头部会让一个完全完成的 review 搁浅而没有 Clear）。`Send all`
  仍以草稿为门禁；**Clear**（`testid: review-clear`）是一个破坏性 `ConfirmPopover`，
  调用服务器原子的 `review.close` Clear；宿主归档非草稿记录、丢弃草稿、替换活动 review，
  并发布全新的空快照，因此发起客户端与兄弟客户端都通过 `review.changed` 收敛。
  空正文区分两种空：**记录仍存在但每个文件都已完成**（“…finished — Clear to
  archive…”）vs **真正空**的 review（“No review comments yet…”）。没有归档浏览器。
  review 级（整体备注）composer 目前被移除（`review` 评论类型保留在模型中，无 UI）。
  `review.get` 水合读取**由 workbench 工具集成拥有**，位于条件挂载的 Review
  正文之外（`useWorkspaceReview`，即 `useWorkspaceSpecs` 模式 ——
  该读取也会在服务器侧重新锚定）：即使面板正文未挂载，标签页标志与 Review
  徽标也需要该快照。每个客户端都收敛到折入 store 的 `review.changed` 推送；
  这里没有任何东西乐观变更。评论撰写是**选择触发，没有模式开关**。渲染器实现投射
  `SurfaceReview` 锚点，绝不重写它们。共享的 React `ReviewComposer` 拥有文本域、Save
  draft / Send now / Cancel 动作、忙碌状态、焦点与光标位置；`PreviewCommenting`、Pierre
  `FileDiff` 与 Pierre `File` 把同一组件放在各自的几何中。
  保存的草稿携带原始文件 `lineRange`；无法定位的渲染预览选择产生空选择器集，
  因此是整文件评论。`useFileReview` 把草稿与路径、scope 以及该界面的 **side**
  结合（宿主填充 `contentHash` 与容忍漂移的 `textQuote`）；Send now
  另外触发 `review.sendComment` 并打开创建出的聊天。

  **Pierre 拥有每个 diff 与每个手机类代码文件。** 带 `lineRange`
  的线程在其范围结束行变成 `lineAnnotations`（`base` → `deletions`，`worktree` →
  `additions`）并渲染 `ReviewThreadCard`；注解元数据与数组在无关 review
  推送之间保持评论 id 身份。因为 Pierre 1.5.1 按数组索引为 React 注解包装器设 key，
  每次挂载都用 tombstone 保持只追加的评论 id 槽位，并为 composer
  保留始终存在的第一个槽位。行号选择与 gutter 工具会在那一侧打开 composer。
  跨越两个侧面空间的选区（unified 视图，从已删除行拖到新增行）**不打开 composer**：
  composer 槽位显示 `review-selection-blocked` —— “a comment anchors to one side of the
  diff” —— 同时用户自己的选区仍高亮，并提供一个 Dismiss，
  因为锚定在用户未选择的一侧是静默重指向，而两个侧面空间之间的坐标转换并不存在。
  聚焦请求在 `onFocusHandled` 之前把匹配的注解滚动到视野内；没有 `lineRange` 的线程，
  或其端点在 Pierre 初始折叠的未变更上下文中被隐藏的线程，绝不进入 Pierre，
  而留在栏级未放置条中，该条消费针对它的任何聚焦请求。Pierre `File`
  在手机上应用相同的注解、选择、聚焦与未放置规则，
  恰好把端点在当前文本中存在的线程报告为已放置（`filePlacedThreadIds`）——Pierre
  只为存在的行发出注解行，因此指向被缩短文件的线程否则会从界面和条中同时消失。Review
  只在修改侧是工作树（`branch`、`uncommitted` 或 `pinned`）处附着；commit scope
  是历史的，既不接受 review 撰写也不接受 hunk 变更。diff
  的删除列与新增列仍是两个权威锚点空间：原始侧选择创建 `side: "base"`，修改侧选择创建
  `side: "worktree"`，二者都不会被重映射到另一侧的行号。

  **Pierre diff 文本选择崩溃（WebKit）—— 一个 vendored `@pierre/diffs` 补丁。** 在
  review diff 中拖选代码会让 WebKit 的网页内容进程崩溃（Electrobun 桌面：“A problem
  repeatedly occurred”；`EXC_BREAKPOINT`
  于 `WebCore::EventHandler::handleMouseDraggedEvent`）。Pierre 的 `InteractionManager`
  在悬停行变化时把 gutter 工具移入悬停行的编号单元，并在 `pointerleave` 时移除它；
  在我们的 `renderGutterUtility` 下，该节点持有一个 `<slot>`。WebKit 在每次 slot
  插入或移除时重建整个影子宿主的渲染器，而在文本拖拽的指针处理器中同步这么做会使它崩溃：
  一个不含 Pierre 的页面能复现该签名，而移动一个普通元素、把同一 slot 移动推迟到一帧，
  或约 100 行以下的 diff 则不会。不变量：Pierre 界面绝不在指针或鼠标处理器中同步插入、
  移除或移动 `<slot>`。`patches/@pierre%2Fdiffs@1.5.1.patch`（根
  `patchedDependencies`）携带上游公开提案 pierrecomputer/pierre#1185（issue
  pierrecomputer/pierre#1184）加上它缺少的两项补充：在鼠标按钮按住在 Pierre
  自己的选择会话之外时放置是空操作 —— 由 `pointermove` 与 `pointerleave` 武装，
  因此第一次移动就已离开 diff 的拖拽无法移除该 slot ——
  而第一次无按钮的移动会重新放置该工具，即使在拖拽结束的那一行上。Bun
  只把补丁应用到其确切版本，并静默跳过不匹配的版本（安装成功且 lockfile 丢掉该条目），
  因此 review e2e 测试“a text-selection drag leaves the comment button on the line it
  started from” 和 “… that leaves the diff on its first move keeps the comment button”
  就是守卫：只要修补后的行为缺失，它们在 Chromium 上就失败，包括只携带 #1185 的 Pierre
  版本。升级 Pierre 会让它们保持通过，而一旦某个版本包含该修复，
  就删除补丁文件及其 `patchedDependencies` 条目。同一补丁还修复
  `FileDiff.trimColumnRows`：当虚拟化窗口的末端收缩时，1.5.1
  会把最后一个保留行之后的注解行与无换行行连同被丢弃的行一起修剪，
  而之后追加的块从不重新发出它们，因此该行上的 composer、
  线程或工具栏会消失直到完整渲染（在 composer 的聚焦滚动移动窗口时可见）。
  保留行之后的尾随行现在保留；位于其行之前的 separators 仍被移除。守卫是
  `e2e/changes-review-drafts.spec.ts` 中的“Changes keeps unsaved composer selection…”。
  Turbo 会哈希 `patches/**`，因此编辑过的补丁会重建 web bundle，而不是从缓存恢复陈旧的。
  测试的对照项（工具位于起始行；选区持有被拖拽的文本）阻止空布局通过 ——早先的
  `display:grid` 二分“修复”正是只以那种方式生效。崩溃本身只在 WebKit 中复现（一个临时的
  Playwright `webkit` 项目）。跟踪于 #683。

  **Monaco 只渲染桌面文件。** `reviewWidgets.ts` 为那一个界面保留其内容 widget
  选择便利操作、上下文菜单动作、装饰与视图区对账。它暴露以评论 id 为 key
  的稳定区域节点；`MonacoReviewZones` 把共享的 React `ReviewThreadCard` 与
  `ReviewComposer` portal 进去，而 `ResizeObserver` 把它们测得的高度回喂给 Monaco。
  因此无关推送会保持同一个带 key 的草稿文本域，包括共享卡片的 Escape
  取消与快捷键保存行为。没有 `lineRange` 的线程留在同一个未放置条中。
  `monacoMenuIcons.ts` 装饰 Monaco 的独立文件编辑器菜单；不再保留任何 diff 编辑器或
  diff 侧分支。**渲染预览**：`MarkdownPreview` 在每个插入的锚点处切分剥离后的文档，
  并把它拼接到 markdown 片段之间（`splicedSegments` ——inline-edit 的切分模式；
  切分**绝不划分多行构造**：位于围栏代码块或 GFM
  表格内的锚点会吸附到该构造的最后一行（`sourceLines` 的 `indivisibleSpans` +
  `snapSplitLine`），因此卡片落在它所评论的区块*之后*，两半都仍是完整文档 ——
  半个围栏不是文档，其未闭合的开头会在评论存活期间把文件余下部分全部渲染为代码；
  列表与块引用分成两个良构构造，这正是两张项目之间的卡片应有之义；
  无法定位的行追加到文档之后，绝不丢失）——
  插入物是保存的卡片**以及**打开的 composer（位于所选区块下的流内，通过
  `PreviewCommenting` 的 children-as-function 契约；只有瞬时图标保持浮动）。**与 Monaco
  的区域对等**：每个未结算评论下的区块 —— 以及打开时 composer 目标下的区块 —— 佩戴
  `.review-region`（`markReviewRegions` —— 仅一条细 LEFT RAIL，即 Monaco 装饰的
  gutter-rail 一半；**绝不是背景涂抹**：整块涂抹读作损坏的文本选择 ——
  在一个要点中选中三个词就把整个要点涂满；只对叶子 BLOCK 元素）。预览锚定是**精确的**：
  `sourceLines.ts`（采用自 inline-edit）以 RAW 文件坐标给元素盖 remark
  源位置戳（`sourceLineRehype` 元组形式取每个片段的偏移 —— 片段从第 1 行重新解析；通过
  `chat/Markdown` 的 `rehypePlugins` prop），composer
  通过戳解析选区（仅边界处的结束块由其前一个带戳的兄弟替换），对未盖戳的内容回退到
  `previewAnchor` 的短语搜索。侧栏仍是全细节界面。**Review
  存在感是自我宣告且按文件的**：一个中心资源标签页（文件或 diff），其路径仍在 review
  中时佩戴 `Review` 标志，带**两个状态**（`ReviewTabFlag`，基于唯一的 `reviewFlags`
  推导）—— 当该文件持有**未发送草稿**时为强调色（`tr-text-eyebrow text-primary`），
  一旦只剩**已发送**评论则为弱化色（`text-text-subtle`）；resolved/dismissed 完全去掉它。
  是两个状态而非有/无，因为*“在 review 中”*与*“有东西要发送”*是不同事实，而 review
  词汇的其他部分早已把草稿**或**已发送计为在 review 中（`fileSummaries`、
  `selectActiveReviewedPath`、`fileThreads`）——
  仅草稿的标志会让聊天正在积极处理的一个文件在标签条中看起来与从未评审过的文件完全相同，
  而 rail 却坚称它在 review 中。**`Send review (N)` 严格仅限草稿且按文件** —— 文件或
  diff 栏的资源工具栏携带该文本按钮（`SendReviewButton`，基于唯一的 `fileDraftIds`
  推导）：计数与发送恰好是本文件的草稿，批处理进该文件自己的 review
  聊天（每个文件一个聊天 ——宿主在 `Review.fileSessions` 中钉住它并之后在那里发送
  `followUp`），该聊天**立即打开**（宿主把包分离地射入会话 —— 见 reviews SPEC
  的发送延迟说明）。其他文件的草稿不动；每个栏携带自己的按钮，而 Review
  面板在每个展开区段的条中显示同一按钮（面板头部添加跨文件的 `Send all (N)` ——
  见上文）。在无可发送内容时提供它会是谎言，
  因此进行中的文件保持其弱化标志且不长出工具栏。覆盖未评论文件的栏两者都不显示。
  没有可进入的手动 review 模式。每个发送便利操作（composer 的 Send now、线程卡片、
  侧栏行/页脚、标签页 Send all）都经过唯一的 `reviewSend.ts`
  对（`sendReviewComment`/`sendReviewBatch`：请求 → 显示聊天标签页 → 失败时 toast），
  而各栏通过唯一的 **`useFileReview`** hook 集成，把其以锚点为 key 的 `worktree`/`base`
  界面通过注册表 props 传入。该 hook 保持一个文件的线程数组，
  从而在它们的线程未变时保持两个界面（`sameThreads`）：每次 review
  推送都会替换工作区的评论列表，
  而由它重建每个打开文件的线程会给每个已挂载渲染器一个全新的 `review`。编译器跳过的
  Monaco 曾因另一个文件上的一条评论重渲染两次（`e2e/perf/reviewThreadIsolation.perf.ts`
  把它钉在零）。该对、该 hook 与 `SendReviewButton` 接受一个可选的聊天会话 id：
  没有它时发送指向最后打开的聊天（否则指向该文件钉住的 review 聊天）；有它时 ——
  一个显示在聊天工具卡片中的文件，其 `ResourcePreview` 读取该聊天的 scope —— 宿主的显式
  `sessionId` 胜过钉住的聊天，因此反馈落在显示该文件的聊天中。shell 的
  `AppResourceHost` 恰好把这些（按调用绑定）作为渲染器的 `FileReview` / `ReviewSend`
  宿主能力暴露。一个批次用它触及的每个会话回答（每组一个），
  因此多文件批次会打开它启动的每个聊天并聚焦第一个 ——
  一个用户从未见过的聊天仍会是一个在其评论上工作的 agent。
  **显示每个聊天的行为依据结果的 `reused` 标志分叉：
  **本次发送**创建**的聊天直接从结果打开（`openChatSession` —— 无往返，
  且其运行时在第一个流式事件之前就存在），而**复用**的聊天走 `openChatInTab`
  的标签页→运行时→磁盘升级，因为它可能是本客户端从未见过的聊天（第二个客户端，
  或本客户端重载后 —— review 状态与 pi 转录都活得比宿主久）；
  把它当作新的打开会为已标记为发送的评论显示一个空白对话。
  **侧栏导航前往锚点可读的界面**（一个推导，`reviewModel` 的 `ReviewSurface`：行的
  `commentSurface`、文件行的 `reviewFileSurface` ——
  后者仅当该文件上*每一个*未结算评论都是 base 侧时才选 diff）：`base`
  锚点的行索引变更前的 blob，只有 Pierre 的删除侧渲染它，也只有它挂载 `base` 线程，
  因此它会重开一个**以锚点自己 `baseRef` 的 pinned diff**（`GitDiffScope.kind:
  "pinned"`，wire v30：工作树 vs 一个不可变提交）——绝不是它被捕获时的 scope，
  后者会针对当前分叉点/`HEAD` 重新解析，并在工作树提交或 review
  目标被重指向时从评论底下移走（旧卡片会以陈旧行号挂载到不同的 blob 上）。在 `baseRef`
  被盖戳之前保存的评论回退到其捕获的 scope，然后回退到工作区的当前 scope。
  把每一行都路由到文件标签页会把 base 侧的 remark 放到说着别的内容的工作树行上，
  没有卡片，聚焦请求也无人消费。
- **实时刷新（工作树面板跟随磁盘）。** 每个工作区范围的读取都经过一个 hook ——
  **`useWorkspaceRead(workspaceId, read, handlers, readKey?) → { reload }`** ——
  它拥有*何时*读取（工作区切换、该工作区的 `fsChangesByWorkspace` 跳动、**`readKey`**
  变化，或手动 Refresh 的 `reload()`），而调用方拥有*如何对待*结果（`onResult` /
  `onFailure` / `onSwitch`）。集中于此是因为每个站点否则都要重新实现**陈旧响应守卫**：
  调用方转移时仍在飞行中的回答绝不能落到新工作区的视图里（读取带代际戳 —— 最新者胜，
  被放弃者保持沉默）。`null` workspaceId 不读取任何东西，
  这也是组件表达*暂停*读取的方式。一个可见的 `FileTree`
  目录在折叠时只探测到足以识别其紧凑单目录链为止；
  链最深目录之下的后代只在该紧凑行展开时才挂载并读取。无需把任何 tick 作为 prop
  穿线下来。它的使用者 —— `FileTree`（根 + 每条可见目录链）、
  `FilesPanel`（`git.status`，通过 `useGitStatus`，喂给两种模式）、
  `useWorkspaceSpecs`（`spec.graph`）—— 外加 `FilePane`/`DiffPane`，
  它们按每个打开的标签页遵循同一 tick 契约。agent 编辑、终端命令与 Finder
  变更都无需手动步骤即会落地。三种形态让其 effect
  的依赖列表保持**诚实**（其中任何地方都没有 exhaustive-deps 豁免）：fs tick
  被作为**事件**消费（`useAppStore.subscribe`）而不是选择进组件 ——
  因此它在不是渲染输入的情况下触发重读，消费者也不再因无关的工作树抖动而重渲染；
  **重置是 effect 的清理**，它闭包捕获被*离开*的工作区（重置真正需要的那个 id ——以
  `workspaceId` 为 key 的朴素 effect 运行时，*新* id 已在作用域内）；
  而手动刷新是**命令式的 `reload()`**，不是 nonce 依赖。`readKey`
  是读取的**第二个身份维度**，用于以工作区之外的东西参数化的读取 —— `ChangesPanel`
  传 `${scopeKey}:${targetRef}`，因此切换 diff scope
  或重指向目标分支会像工作区切换一样重置并重读，一个 scope
  的列表绝不会在另一个之下滞留。`onFailure` 接收**该拒绝**，而不只是工作区 id：
  对某个*具名*失败作出反应的调用方（见下文消失提交规则）必须能把它与超时或未命名宿主失败
  区分开。唯一刻意**不**经过此 hook 的读取是 `ChangesScopeMenu` 的惰性对 ——
  它们是*打开*触发而非 tick 触发 ——因此该菜单改为**以其完整身份 `(workspaceId,
  targetRef)` 为 key**：其提交行是 `git log <base>..HEAD`，
  所以重指向目标会改变存在哪些提交，而重挂载会清除属于上一对的行，
  同时中和仍为其在飞行中的任何响应。在一次挂载内该对也是**带代际戳**的，
  因此连续两次打开不会让较早的回答覆盖较晚的。它**只是身份** ——是什么让重读发生，
  绝不是读取*用什么*读（参数位于调用方的 `read` 闭包中，hook 每次渲染都重新捕获它，
  因此重读所用的值按构造就是 key 命名的那个）。它被作为参数穿线给 `read`（与
  `reload`），供宁愿据此分支而不闭包捕获参数的调用方使用；忽略它 —— 如 `ChangesPanel`
  所做，其 `scope` 只是一个 key 命名的对象 —— 是预期行为。重抓**保留视图状态**：
  `FileTree` 重读根 + 支撑每个可见紧凑行与展开分支的目录探测。
  展开状态位于单个行之上（在 store 中，因此也跨重挂载与重载存活），
  并以紧凑行所代表的每个目录路径为 key，因此缩短或加长一条链都无法隐藏重抓前可见的后代；
  消失的目录经其父项退出。`FilesPanel` 重读 `git.status`（仅列表 —— diff
  作为中心资源渲染，不在列表之下），`SpecsPanel` 重抓而不重挂载（展开状态存活），而
  `FilePane`/`DiffPane` 在工作区 tick 越过其已加载的
  tick 时重读打开资源的内容（可见时实时；后台标签页在本地选择时追平 ——
  只挂载每个组的选中 body；重读失败 ——文件被删除 —— 保留最后内容，不自动关闭；
  文件离开变更集的 diff 标签页同样保留其最后内容 ——消失之处显示在 Changes 列表中）。
  `FilePane` 与 `DiffPane` 运行**同一个**标签页内容实时刷新契约 ——共享的
  **`useLiveTabContent(tab, {read, applyFresh, keepCurrent}, reloadKey?)`** hook ——
  只差在读取方法（`fs.readFile` vs `git.diffFile`）与 store
  的工作区限定写入（`updateFileTabContent` vs `updateDiffTabContent`，
  各自接收所捕获的工作区，因为不透明的缓存 id 可能跨工作区重复）。
  其单批跳过（“此文件不在其中 —— 只推进 tick”）要求该批**点名**了文件：
  一个**无路径**帧（`paths: []`，宿主的 ref 移动轻推）总是重读，
  因为路径归属对一次未触及任何文件的变更什么也说明不了 —— 这正是终端 `git commit` 移动
  `HEAD` 时让打开的 `uncommitted` scope diff 保持诚实的原因。`reloadKey` 是该 hook
  的**第二个实时维度**，用于内容依赖文件之外某物的标签页：`DiffPane` 传
  `selectDiffTabTargetRef`，因此重指向 review 目标会立刻重读**branch scope** 标签页，
  而不是拖到下一次 fs tick（commit scope 没有这个维度 —— 其两侧无法移动）。
  该重读保留标签页既有的 tick：它回答“这个标签页现在意味着什么”，它不观察文件变化。
  两个维度是两个 effect，因此**两次读取可以同时在飞行中**（一次缓慢的 tick 重读，
  然后一次重指向）；二者从**每标签页一个排序器**（`createReadSequencer`，
  有单元测试）轮流，而响应**只在之后没有更晚读取开始时**才被写入。否则就是网络挑出胜者：
  乱序解析时，较旧的读取最后落地并覆盖较新目标的内容，同时携带其自己诚实 —— 但现已陈旧
  —— 的戳，于是两个 effect 都看不到任何漂移，栏会在新目标的标签下无限保留旧目标的 diff。
  丢弃被取代的读取没有成本：取代它的那次读取才是用户正在等的。面板只为活动工作区挂载，
  因此作用域是自然的。降级的宿主 watcher 会暂停自动失效，直到下一次工作区读取重建它；
  可编辑文件的冲突处理等待 `fs.writeFile`（目前查看器是只读的）。
- **`useWorkspaceSpecs` 拥有 `spec.graph` 读取**（一个抓取器，
  一个“此文件是规格”的定义）：快照落入 store（`specsByWorkspace`）而非面板状态，
  因为聊天的轮次分隔需要同一答案来路由其 chip。它由 **workbench 工具集成**调用，
  而不是由 `SpecsPanel` 调用 —— 面板正文只在其标签页显示时存在，
  因此在那里拥有读取会让坐在 Changes 上的用户阻止图跟踪工作树，而 agent
  写的每个规格都会被计为变更文件（这种拆分会被一次标签页选择静默撤销）。由于按工作区设
  key，切换时会显示该工作区最后已知的树而重读在飞行中（没有需要重置的东西），
  而失败读取标志是工作区范围的，因此无法把某个提示泄漏到兄弟的良好树上。它返回 `{
  failed, reload }` ——`SpecsPanel` 的仅错误 Retry 直接调用 `reload`，
  因此无需在面板状态中持有重试计数器。
- `SpecsPanel` 是只读的规格图谱查看器 —— 该快照的纯读取者。每次工作区激活抓取一次，在
  fs tick 时自动重抓，渲染为 **`parent` 树**（根 = 无/悬挂父项；默认展开）。**没有常驻
  Refresh 控件或面板工具栏行**：例行同步是自动的。
  抓取**失败渲染一个独立的行内错误提示并带 Retry**，绝不是 “No specs” 空状态 ——
  离线与空是不同的答案。有上一次快照时，提示位于保留的树之上；没有时，它替换加载状态。
  树构建（`specTree.ts`）假定良构图 ——**父环是 `spec_validate` 的问题，
  不是查看器的**（环成员从任何根都不可达，因此根本不渲染）——但遍历是**访问守卫**的，
  因此畸形图绝不会让 UI 挂起或成环。本切片只有树 —— 无跨边显示、无编辑、无校验徽标、
  无图画布。
- `SpecsPanel` 是紧凑的**文档优先树**：规格节点既是容器**也是**文档，
  因此控件把两种角色都显式化。层级使用固定的逐深度缩进 + chevron，
  刻意**不用连接轨或分支肘线**（常驻线条让狭窄的 rail 过载）。只有带内边距的
  **chevron** 展开/折叠，而行其余部分是原生文档按钮，其**单击预览**渲染后的规格 ——
  其**双击保留**它 —— 经过与 `FileTree` 相同的 `fs.readFile` → `openTab` 流程（见
  Preview 标签页条目；沿规格图谱往下读正是可复用槽位存在的场景）。每行保持单行：缩进 →
  chevron → 形状编码的角色图标 → 截断标题 → 尾随角色（`ARCH` / `MODULE` / `SUBMODULE` /
  `TASK`；未知类型紧凑降级）。角色在**行悬停/聚焦时揭示**，不截断，而 `aria-label`
  无条件携带它。标题通过 `specDisplayTitle` 渲染，它把标题的 ` — ` / ` – ` 分隔符折叠为
  **` · `**。顶层 `goal-and-requirements` 行改为携带确切的**`Main spec`**
  标签与独立的根图标；本地选中的文件资源的行有常驻选中处理。**生命周期状态完全不呈现**
  ——未来的 lint 健康随真正的 linter 特性到来，而不是靠投机性的点或复用的状态外壳。
  这仍是一个克制的层级 ——没有主图、重复根、预览栏或图画布。`FileTree`
  共享同一文件手势模型（preview/keep），但保留自己的目录行为 ——整行点击切换目录，
  那里没有冲突。
- **聊天深链保持排布无关。** 一个 shell 拥有的 **`LayoutIntent`** 命名单例工具；shell
  解析其当前侧/组、就地揭示它，并在本地选中它。`changesRequest` 与 `specRequest`
  添加要聚焦/打开的那一个路径，而不命名布局目的地。因此只揭示工具的分隔 chip
  无需伪造路径或假定固定右侧面板。`ChangesPanel` 监视
  `changesRequest`（由聊天轮次分隔的 “files changed” chip 设置），
  **高亮**所请求文件的行（用 `matchesWorktreePath` 对照 `git.status`
  解析）**并在目的中心组的预览槽位中打开该文件的 review 标签页**（`path: null`
  的请求打开标签页而不揭示，也不等待面板自己的 `git.status`，
  因此刚挂载的面板从不因被修剪轮次而回退并重定向它；携带 scope 的请求打开该 scope
  的标签页）—— chip/列表行的点击*就是*用户显式要求看到该变更，
  因此停在仅高亮读起来像坏了，而跟随 chip 是浏览，与点击它所指向的行相同，
  所以它复用该槽位而不是为每个 chip 累积一个保留标签页。不再位于当前 diff
  中的路径（几天前的一轮）降级为仅高亮：没有 diff 可显示。
  **用户已经导航越过的深链也是如此** ——这次打开是那种在发生时*无法*标记自己导航的情形，
  因为路径只有在 `git.status` 落地后才可解析，而通常正是该 chip 揭示此视图（全新挂载、
  完整往返）。它对比的是点击时盖戳的目的组 id 与本地导航时钟，
  因此用户在列表加载时选的标签页是更晚的导航并保持焦点。
  该意图在处理后被**消费**（`clearChangesRequest`）—— 它打开一个中心资源，因此重放它的
  git-status 重读会把用户的标签页拽回去。`SpecsPanel` 监视 **`specRequest`**（“N specs”
  chip）并**打开渲染后的规格**，同样在目的组的预览槽位（`openFileInTab`，
  它把上报路径规范化 —— pi 可能上报绝对路径或带 `./` 前缀 ——
  为工作树相对的**标签页身份**，
  因此深链绝不会为一个已在其相对路径下打开的文件打开第二个标签页；
  这位于咽喉点而不是每个调用方，也意味着几秒前创建、尚未进入图谱的规格同样能打开）——
  规格除了其内容没有可预览的东西，而树行从本地选中资源身份点亮。
  该意图在处理后被**消费**（`clearSpecRequest`）：与 Changes 链接一样，
  它打开中心标签页，因此在重挂载或图重抓时重放它会在编辑中途把用户的标签页拽回去。
  两个意图，两个 effect：规格 chip 绝不能落进 git 推导的 Changes 视图，
  后者在结构上无法显示 gitignored 的 `.thinkrail/context/` 草稿规格 ——
  正是推动这次拆分的空 Changes 缺陷。两个意图都携带**恰好一个路径**：
  写入多个产物的一轮在聊天中解决歧义（chip 在那里展开为列表 —— 见 chat/SPEC.md），
  因此没有面板需要标记一个*集合*。这是刻意的 ——
  在这些工作区范围的树之上再来一套按轮次范围的标记词汇，
  会重新引入上面的单选规则存在所要防止的“两行都读作选中”歧义。
- **diff scope 在 Changes 头部选择，并进入标签页的身份。** 两个头部控件说明在 diff
  什么：**`ChangesScopeMenu`** 药丸 —— *All
  changes*（自分叉出目标分支以来该工作区的工作 —— 从 merge base 度量，
  因此落到目标上的上游提交绝不会在此成为幻影行；默认值）/ *Uncommitted changes* /
  来自分支列表的一个**提交** ——以及目标分支的共享 **`BranchPicker`**
  药丸（`workspace.setDiffBase`；面板收敛到广播 `workspace.updated`，绝不乐观）。
  在两行固定内容下方，菜单提供 **Last turn · N files** —— 最新的宿主记录的 agent
  运行（`store.turnsByWorkspace` 经 `useWorkspaceTurns` 读取；shell 在
  `TURN_CHANGES_PROTOCOL_VERSION` 及以上的宿主上按连接加载它；
  在某个运行改变过东西之前禁用并显示 “No agent turn changed files yet”）——
  以及当存在多个运行时，一个 **Agent turns** 列表（最新在前，时间 · 文件 ·
  相对结算时间）。选择一个设置 `turn` scope `{ id, baseTree, headTree, startedAt }`；
  review 标签页与逐文件标签页从快照读取两侧（`/blob` 提供树限定路径），因此 scope
  在工作树继续前进时保持冻结，且由于其修改侧是历史的，它不提供 revert 或 keep 控件。
  被修剪的快照会像被改写的提交一样被拒绝。菜单的其他内容**每次打开时惰性加载**，
  绝不在面板挂载时加载：提交行用 `git.listCommits`（subject + `shortSha · author ·
  relative time`），未提交 scope 下用一个 `git.status` 探测，这正是让 *Uncommitted*
  行说 “No uncommitted changes”（禁用）而不是打开一个无解释的空列表的原因；
  各自独立降级。菜单内容是**高度有界且可滚动的**（基于共享 `DropdownMenuContent` 原语，
  因为任何长菜单都有此问题）——200 个提交行绝不能跑到视口边缘之外无法触及。药丸以**短
  sha** 命名 commit scope，绝不用其 subject（`scopeLabel`；subject 是触发器的 `title`，
  通过 `scopeTitle`，菜单行完整显示它）——rail
  头部的一句话会把兄弟目标分支药丸挤成省略号。一个命名仓库已不再拥有的提交的
  scope（rebase、分支重置）会让宿主以**具名**代码 `UNKNOWN_COMMIT`（`wsErrorCode`）拒绝
  `git.status`，而*那个*拒绝 ——且只有那一个 —— **带 toast 重置到 branch scope**，
  而不是卡在一个死 sha 上。review 标签页的 scope 就是其身份，
  因此*它*的读取上出现同一代码绝不重指向该标签页：它丢弃陈旧列表、
  点名什么消失了（提交或轮次快照）并提供*Review all changes*，经由 `requestChangesView`
  前往侧栏重置所落的 branch 标签页 —— 绝不是无用的 *Retry*。其他每种失败（超时、
  长时间网络中断、git 错误）都不动用户选定的 scope、保留最后一次良好列表，
  并在每个失败连续段中说明一次：在网络抖动时静默切换 scope 比陈旧列表更糟。
  该代码之所以存在，正是因为“读取失败”无法区分这两种情况。
- **“从未回答”、“失败”与“回答为空”是三种状态，绝不是两种。** 面板分别持有 `GitStatus`
  *与*失败：尚无状态读作 **Loading…**，有失败但没有可保留的列表时渲染错误加
  **Retry**（`changes-error` / `changes-retry`，`reload()`），只有已落地且 `changes`
  为空的回答才可说 “No changes in this scope.”（`changes-empty`）。
  失败的首次读取绝不能走空状态分支 —— “干净”是*关于工作树的断言*，
  而没落地的读取没有做任何断言；一个在不干净时显示干净的 review 界面是本产品最糟的失败。
  宿主侧同一规则：非零 `git diff` 退出会**抛错**而不是产出空变更集（见
  `server/src/git/SPEC.md`）。**目标分支位于 scope 菜单旁边，
  而不是其内部**（如最初设计）：可搜索列表属于组合框，而嵌套的 Radix
  子菜单会在那些惰性读取落地、菜单重渲染时自行关闭。
- **diff 是中心资源标签页，不是 Changes 工具内的内嵌内容，而 Changes 有两种呈现。** 在
  Changes 行上单击打开（或在内部揭示）该工作区针对面板 scope 的 **review 标签页** ——
  `ChangesReviewPane`，每个 *(workspace, scope)*一个 `ChangesTab`（`changesTabId`），
  preview 意图 —— 该 scope 的所有变更文件作为**区段**呈现在一个滚动表面中：
  粘滞区段头（状态、路径、`+N −M`、注册表返回多个候选时的渲染器段、*Open as tab*、
  *Revert file*、*Viewed*），正文在滚入视野时惰性读取 `git.diffFile`，并在与 `DiffTab`
  相同的实时刷新契约下刷新。双击，或区段的 *Open as tab*，通过 `openDiffInTab` 以
  `keep` 意图打开逐文件的 **`DiffTab`** *(path, scope)* —— 深层单文件界面是一种提升，
  绝不是第一次点击。双击携带其前导单击的预览槽位主张；显式 *Open as tab*
  则保留既有预览。review 标签页有两个**视图模式**，由其工具栏中的 `Stacked | One file`
  段选择并在应用级持有（`store.changesLayout`，像 `changesView`）：*Stacked*
  是连续列表；*One file* 一次显示一个区段，带 `Prev / Next` 与 `n / N` 计数器（`V` 切换
  viewed 并在标记时前进，`Alt+↓` / `Alt+↑` 步进文件）。两种模式读同一个模型 —— scope
  的有序文件列表、`viewed`、`activePath`、折叠覆盖、`Split | Inline`、¶ ——
  因此切换绝不丢失 review 进度；为每种模式设第二种标签页正是出于这个原因被拒绝。
  切换复用该栏的区段内容缓存与渲染器渲染缓存，因此新模式首帧即画出完成好的 diff。返回
  Stacked 是双缓冲的：新挂载的列表测量其视口并在仍显示的 One-file 视图下方 `invisible`
  状态下滚动到活动文件，只有当该文件的头部在两个帧内保持列表顶部后才丢弃 One-file（上限
  30 帧）。直接显示时，列表在滚动 0 处绘制其第一个文件与页脚，然后可见地滚动到活动文件。
  **瞬态撰写属于标签页区段，而不是其已挂载的渲染器。** 一个可选的 scope 范围 review
  草稿上下文在虚拟化、折叠、渲染器以及 Stacked/One-file 切换之间保留带戳选区、composer
  文本/光标、待提交，以及未保存的线程编辑。共享的 `useStampedComposer`
  与线程卡片对代码与富表面使用此策略；渲染器、侧/页、文件、scope
  与工作区身份隔离各自的暂存状态。笔记本选区保留带戳的侧与单元索引，
  绝不是解析后单元对象的身份。Cancel 与成功的 Save/Send 退役它；
  迟到的完成无法关闭更新的选区。恢复暂存绝不聚焦文本域或发送宿主变更。
  既有的线程编辑保留 blur 保存与快捷键保存；发送会等待任何已编辑正文被保存，Escape
  只丢弃本地编辑。线程卡片在每个界面上都有一条编辑状态路径 ——
  没有暂存上下文时它回退到本地状态，因此逐文件标签页无法与 review 标签页漂移。
  内容戳变化仍会使陈旧选区失效。暂存仅限标签页生命周期，不跨重载持久化，
  且其第一次写入保留预览。**Changes 侧栏是 review 标签页的导航器**：
  其行高亮跟随标签页的 `activePath`（头部越过工具栏中线的那个区段，
  或所显示的单个文件），已 viewed 的文件佩戴对勾字形，
  点击行揭示该区段（若被折叠则展开它）——在异步区段确定其高度期间，该揭示仍以路径锚定，
  直到视图模式变化或用户的滚轮、触摸、指针或键盘输入接管。超时无法约束远程读取。
  查找导航只在区段揭示之后把滚动交给线程，并禁用后续头部修正，使线程无法被滚出视野。
  该栏把**区段**带入视野；**评论**在区段内的揭示是一个 `reviewFocusRequest`，由
  `consumeChangesTabReveal` 抛出，由目标区段的 `usePierreFocus`
  消费（`virtualizer.revealLine`）——
  这是唯一能显现尚未在有界虚拟化区段中渲染的某行及其线程卡片的路径。用户接管的揭示、
  更新的揭示，或该栏卸载，都会撤回其待处理 `reviewFocusRequest`，
  因此之后挂载的线程绝不会把视口猛拽到一个没人在等的发现上。
  每个区段只接收标签页中属于自己路径的那一片（暂存、保留 keys、标志）且在 `memo` 之后，
  因此一个区段的 composer 中的击键只重渲染该区段，而不是其兄弟 ——且在其内部只有
  composer，因为渲染器表面与其注解按槽位读取暂存（[[module-resource-renderers]]）；
  正文高度自然的 stacked 区段不保存视图状态 —— 只有 One-file 与有界渲染器拥有滚动器；
  `changesView` 默认 **Tree**，而文件行在两种视图中都是 `change-item`，
  因为它是同一个东西。当所显示的文件离开 scope（被 revert，或 agent 移除了其变更）时，
  One-file 模式停在该位置，而不是跳回第一个文件。列表以一个视图口高度的测量尾部结束，
  使最后一个文件可以滚到顶部并变为活动，与 GitHub 一样。scroll-spy 绝不自动揭示 Review
  工具：`selectActiveReviewedPath` 刻意忽略 review 标签页，
  因为由滚动触发的揭示会在阅读中途替换其共享侧组中的 Changes 导航器。**Hunk
  分诊是同一标签页上的覆盖层，不是第二套变更模型。** 在可变 scope 中，每个 hunk
  工具栏在既有 Revert 与 Ask-agent 动作旁获得 **Keep**；被保留的 hunk 以**内容 key**
  记录在标签页上（`kept[path]`）—— 该 key 是该 hunk 被移除 + 新增文本的哈希，
  内容完全相同时带上重复组大小与原侧位置，因此 revert
  一个较早的双胞胎不会把其决定转移给未评审的双胞胎。歧义组在其大小变化时保守地丢失决定。
  唯一的 key 在相邻 revert 引起的行位移中存活，并在 agent 再次改动该 hunk 时立即失效。
  被 *reverted* 的 hunk 无需状态：ThinkRail 的 revert 恢复基础文本，hunk 离开 diff，而
  toast 的 Undo 是回去的路。区段头显示 `k/n kept`，保留某文件的最后一个 hunk
  会原子地把它标为 **viewed** ——这是一个动作边，而不是常驻渲染规则，因此显式未 viewed
  的文件在虚拟化与模式切换中保持未 viewed。Hunk 与文件进度是同一个模型；stacked
  视图的底栏显示 `n of N reviewed`、**Next unreviewed**（`J`，环绕）与**Mark all
  viewed**，而 `V` 切换活动区段。刻意没有 *Revert all*：宿主没有原子的多文件丢弃，而
  review 界面上的批量破坏性动作只有在服务器侧存在之后才配得上它的击键。逐文件的
  `DiffTab` 不传分诊且保持不变。**评审人的故事是一段行走，不是一份报告。** 当工作区
  review 携带 `guide`（最新计划步骤判定的摘要与建议阅读顺序，见
  [[submodule-server-reviews]]）或任何未关闭的 agent 发现时，stacked 视图长出一条左侧
  **review guide rail**（`ChangesReviewGuide`，从工具栏切换，绝不在 One-file
  模式显示）：判定徽标 + 摘要、编号阅读顺序（scope 外文件的步骤被禁用而非隐藏，
  使顺序保持可读），以及发现。`guideSteps` 是唯一的推导 —— 先阅读步骤，
  然后按路径排列的未关闭 agent 发现 —— 而 **Start / Next / Restart**（`N`，`P`
  后退）行走它，跳过文件不在本 scope 的步骤（`nextGuideStep`；禁用步骤仅为上下文显示，
  绝不停在其上）：步骤通过标签页的一次性揭示显现其区段，发现步骤还会触发 store 的
  `reviewFocusRequest`，因此线程卡片通过与 Review 面板相同的路径被滚动到。*Fix this
  one* 与 *Apply fixes* 是 Review 面板的发送路径（`sendReviewComment`、`sendReviewBatch`
  作用于未关闭的发现 id），在一个 busy 标志之后，不是新的变更。
  游标是步骤的身份（阅读步骤在顺序中的位置 —— 契约允许同一文件出现两次 —— 或发现 id），
  因此从其下方被解决的发现会让位给其后继，而不是跳过其中一个。rail 绝不在手机类视口渲染
  ——390px 屏幕上的 280px 会不留 diff 可引导。纯字母快捷键（`V`、`J`、`N`、
  `P`）属于中心正在看的 review 标签页（`ownsReviewShortcut`：attention 组的选中标签页、
  没有打开的模态或菜单层、没有文本输入目标，且没有更早的处理器消费该事件），
  因此并排拆分的两个 review scope 无法都响应一次击键。**规模规则**：文件变更超过 400
  行的区段，或路径是锁/生成文件的区段，折叠挂载在 *Expand* / *Open as tab* 之后；scope
  超过 50 个文件时显示一条可关闭的通知，提供 *One file*；没有任何东西自行切换模式，
  因为一个在 review 中途自我重排的 review 界面会丢失读者的位置。该列表是扁平的
  `react-virtuoso` 列表，每个文件一项，头部粘滞在其项内部（不用 `GroupedVirtuoso`：
  活得比其项内容更久的组头部无法承载逐文件控件）；Pierre `CodeView`
  曾被评估用于容器并被拒绝，因为其项只有 `file` / `diff`，因此无法承载区段必须像
  `DiffPane` 一样分派的渲染 markdown、图片、SVG、CSV、JSON、笔记本与 PDF 渲染器。
  **区段在拿到 diff 之前就持有其形状。** 每个区段在其正文首次可见时惰性读取
  `git.diffFile`，经由该栏的 `SectionContentCache`：飞行中的读取只在相同路径、diff
  目标与文件系统代际间共享。最新请求可以在其区段滚出后填充缓存，
  但较早的初次读取无法覆盖更新的目标或一次实时刷新。失败的读取不存任何东西，因此
  *Retry* 会再读一次。加载期间，stacked 区段的占位符预留
  `estimatedSectionHeight(change)` —— `232 + 40 × changed lines` px，上限 20 000 ——这与
  Pierre 对同一统计所渲染的大小相差约 2 倍以内。没有该预留时，一个只有其 diff
  十分之一大小的 skeleton 会让 Virtuoso 一次挂载十几个区段（那么多并发读取），
  然后随着每个增长重新估算整个列表，因此跳到远处文件会落在几十个文件之外。`DiffPane`
  与区段都渲染共享的 **`DiffSurface`**：它描述返回的 `ResourceMeta`、为 `diff`
  解析注册表、惰性挂载所选渲染器、托管该文件的 review 线程与 hunk 动作，并显示未放置条。
  仅有字节的原始侧使用响应解析出的原 oid 加 `/blob`；缺失的一侧是显式的，绝不是带伪造
  URL 的字节值。固定工具栏保留路径与逐文件 review 发送，
  然后仅在所选渲染器声明相应能力时暴露 ¶ 空白字符、修改侧复制与 **Split | Inline**。
  当注册表返回多个候选时，它为每个候选渲染一个 `view-toggle-<renderer suffix>` 段。
  渲染器选择替换了旧的仅 markdown 的 `rendered` 状态；布局与空白字符仍是独立的 diff
  状态。review 标签页的工具栏一次性为每个区段携带相同的空白字符与 `Split | Inline` 控件
  —— 但因为该切换是跨所有区段的栏级而非逐渲染器，它们集体遵循 `DiffPane` 的能力门禁：
  只有当 scope 中至少一个文件（按路径，通过
  `diffRendererCapabilities`）解析到声明该能力的渲染器时，每个控件才显示，因此全
  Markdown/图片/PDF/丰富内容的 scope 会隐藏它们，而不是提供无效控件。
  该门禁只读取逐路径的渲染器选择（`selectChangesTabRendererIds`），
  因此对某区段的草稿与视图状态写入不会重新解析该 scope。解析在内容加载之前运行，
  因此门禁从路径的 MIME 推断文本/二进制划分 —— 已知二进制类型（图片、
  PDF）解析到其无布局渲染器；一个无可推断 MIME
  的未知扩展名二进制在其字节加载前仍读作文本。它还携带 scope 与 target 作为只读
  chip（scope 在 Changes 头部选择，是标签页身份的一部分）、文件 / `+ −` 总数与 viewed
  计数、*Collapse all* / *Expand all*，以及在 该工作区草稿上的 `Send review (N)`。

  **宽幅改写会等在一条通知之后。** `DiffSurfaceBody` 通过 [[module-resource-renderers]]
  的 `GuardedRendererDiff` 挂载其渲染器（策略及其阈值在那里），
  并在存在工作树支撑的修改侧时向该通知添加 **Open file**（进入桌面编辑器）的动作。

  打包候选及其行为（代码、markdown、二进制、LFS、图片、SVG、CSV、JSON、笔记本、PDF、
  HTML、Pierre 虚拟化）由 [[module-resource-renderers]] 拥有；各栏只分派、
  持久化视图状态、拥有未放置条并提供 `hunkActions`。

  可变 scope（`branch`、`uncommitted`、`pinned`）仅当当前 welcome 声明
  `CHANGE_MUTATIONS_PROTOCOL_VERSION` 且携带两个哈希的 diff 元信息落地之后才获得
  `hunkActions`。变更块独立于 Pierre 的 hunk 模型：零上下文、同一空白字符策略的
  `diff@8` `structuredPatch` 产出发送给 `change.revert` 的原始/修改 `LineSpan` 对。
  每个块在其第一行变更之上获得一个纤细的注解工具栏，带 **Revert**（`hunk-revert`）与
  **Ask agent**（`hunk-ask-agent`）；纯删除把该工具栏锚定在删除侧，
  而缺失的修改侧会抑制所有块工具栏，只留下头部级 Revert file。
  运行中的会话会添加一句安静的“the agent is working in this workspace”通知，
  但保护该动作的是 CAS，而不是禁用。Ask agent 用修改侧 `lineRange`、一个确切的
  `diffHunk` 头部，以及光标处的 `Please revise this change: ` 打开工作树 composer；
  纯删除锚定到之前已有的工作树行（顶部则为第 1 行）并说明它指被移除的行，
  而空工作树只发出 `diffHunk`，因此是整文件评论。随后它使用普通的
  `review.worktree.commenting.onSend`。固定头部添加 **Revert
  file**（`diff-revert-file`）。

  `DiffPane` 在每次范围/文件 revert 时发送两个已渲染哈希。成功抛出一个八秒 toast，其
  **Undo** 动作用回执 id 与 `receipt.after.hash` 发送 `change.undo`；
  被送入回收站的整个文件会说明它已移到回收站。`STALE_VIEW`
  立即使用 `useLiveTabContent.reload()` 并说 “This file changed since you opened it —
  review the new diff”；`RECEIPT_UNKNOWN`（回执被淘汰、其工作区被遗忘，
  或宿主在一个仍在屏幕上的 toast 背后重启了）以同样方式重载并作为信息而非错误说“This
  change can no longer be undone — the host no longer holds it”；
  其他每种具名或未命名失败都使用 `errorText`。commit scope 从不接收 `hunkActions`，
  因此工具栏与 Revert file 都无法渲染。
- **Changes：List | Tree。** 一个头部切换（`store.changesView`，应用级 —— 持久化在
  store 中而不按工作区，因此跨工作区切换存活）在扁平 **List** 与文件夹
  **Tree**（`ChangesTree`）之间切换，二者都从同一个 `git.status` 列表构建。Tree
  的样式与 Files 树完全一样（共享 `TreeRow`）；文件夹**默认展开**（变更集都很小），
  用户折叠的文件夹按工作区为所有 scope 记住（`filesTreeByWorkspace.changedCollapsed`），
  而单目录链是一行以斜杠连接的紧凑行（基于变更文件树，不论磁盘上未变更的兄弟项），与
  `FileTree` 一致。**状态显示在文件名上，而不是字母字形**（git 装饰惯例 ——
  `changesModel.statusNameClass`，两种视图共用）：added / untracked → 绿色，deleted →
  红色 + 删除线，renamed → 蓝色，modified → 朴素。每个文件显示 `+N −M` 徽标（共享
  `DiffStatBadge`），文件夹**仅在其折叠时**显示聚合徽标 —— 展开后其子项已携带计数 ——
  逐文件计数来自 `git.status`（`GitFileChange.added/removed`，来自 `git diff
  --numstat`；untracked 文件把其整个内容计为 added ——但二进制或超大 untracked 文件不计，
  镜像受跟踪二进制从 `--numstat` 中消失的方式），文件夹计数在客户端求和。两种视图共享
  `ChangesPanel` 的 `openDiff` + `isActive`。**List 显示完整的工作树相对路径** ——
  弱化的目录前缀（行溢出时它先让步）+ 状态色的基名，因此用户扫读的名字保持可见。
- **浏览在每个中心组复用一个标签页：预览 vs 保留。**
  每个工作区视图为每个框架组有一个本地预览身份；其标签为斜体并携带
  `data-preview="true"`。单击文件/规格/变更行，或跟随渲染文档/聊天产物链接，
  会作为预览打开进浏览器最后聚焦的目的组。双击保留；点击已活动的预览也保留，
  作为触摸路径。显式的 Settings/open-as-file 动作以保留态开始。聊天与已注册的
  plan/document 标签页绝不进入预览。标签条与上下文/命令界面还暴露一个可用键盘操作的
  Keep Preview 命令。

  预览只替换该组在同一索引处的槽位，因此浏览绝不重排标签条。单击立即打开其预览，
  来自打开标签页缓存或读取，没有结算等待。双击组合预览然后提升：
  行的第二次点击（`event.detail > 1`）不发送预览（每个浏览行都用 `lib` 的
  `firstClickOnly` 包裹其预览打开），因此只有前导预览与 `dblclick` 的保留出去。
  如果读取已落地，那是两次本地转换：预览，然后对同一放置的就地保留。如果读取仍在飞行中，
  `openTabs.ts` 单飞它并把前导点击的槽位主张带入一次最终的保留转换。无论哪种方式，
  网络延迟都无法颠倒意图。Changes review 标签页预览是唯一仍等待结算窗口的行打开：
  其双击打开的是不同资源（保留的逐文件 diff），因此即时的 review
  标签页会闪一下然后被替换。新鲜度戳（`loadedTick`，以及 diff 的
  `loadedTarget`）在读取离开前捕获，绝不在响应时从更新的状态取。
  本地逐组导航时钟在请求时捕获：陈旧的预览完成会输给更晚的注意力；刻意的保留仍提交。
  如果目的组消失，shell 重路由到当前最后焦点；如果更新的本地转换已放置该规范资源，
  完成时选择那个放置而不是重复它。预览放置与注意力在本地提交。单元与 E2E
  测试钉住双击合并、陈旧读取拒绝、逐组隔离、本地身份收敛，以及键盘/触摸提升。
- **行动作：一个菜单，两个触发器。** 每个**文件**行（两种视图）都包在
  **`ChangeRowActions`** 中：一个悬停/聚焦揭示的 `⌄`
  按钮*与*在行上右键都打开同一个下拉。`⌄` 不是装饰 —— 它是**触摸路径**，
  那里没有右键（移动优先）。项：**View**（与朴素点击相同的动作）与 **Copy
  path**（工作树相对）。刻意没有别的：面板是**只读**的 —— 没有丢弃文件/-文件夹/-全部 ——
  也没有 “Open in ‹external app›”，宿主侧 `open` 会对每个远程/手机客户端静默出错（Copy
  path 是可移植的逃生口）。**文件夹行没有菜单** ——该列表对文件夹不适用任何东西。
  基于既有的 `@thinkrail/ui/dropdown-menu`（不新增 `context-menu` 原语）；
  右键处理器通过渲染 prop 回传，因此它落在行真正的交互元素上而不是裸 div，而 `⌄`
  触发器是行按钮的*兄弟*（按钮套按钮是无效的）。
  三条布局规则让该包装器不可见而不是成为接缝 —— 每条都由一次几何 e2e 断言钉住，
  因为每条都是初稿发布过的真实缺陷：**(1) 包装器拥有行的高亮**（悬停 / 选中 /
  菜单打开），因为色带必须也跨过尾随槽位，否则行读起来在它自己的菜单之前被切断 ——
  内部元素**完全不**绘制背景（扁平列表的按钮不携带 `hover:`/选中类，而 `TreeRow`
  接受 `highlight="wrapper"`，其 `"self"` 默认值是 Files 树想要的）。始终恰好一个绘制者：
  两个会隐藏“包装器已停止绘制”的情况，这就是为什么 e2e
  钉住把*包装器*的计算色带与*内部按钮*的（透明）颜色带比较，而不是包装器对包装器；**(2)
  没有菜单的行保留同一 gutter**（`ROW_MENU_SLOT`，从 `ChangeRowActions`
  导出并由树的文件夹行佩戴），否则 `+N −M` 列在文件夹上会比文件上右移 24px；以及 **(3)
  行与该槽位共享其 flex 行，因此它必须能在其标签之下收缩** ——`TreeRow` 携带 `min-w-0`，
  而每条路径都渲染为*两个可截断的一半*（目录 + 基名），因此长基名绝不能把计数（或在
  `DiffPane` 的孪生 chip 中，把 ¶/复制/布局控件）挤出框外。两半**并非**同等可截断：
  目录前缀在基名让出一个像素之前就**完全**让步，因为名字才是用户扫读的东西。
  该顺序是*结构性的* ——目录是唯一可收缩项（`shrink`），基名是 `shrink-0` ——
  而不是收缩*比例*。比例（这里曾是 `shrink-[20]` vs `shrink`）只是近似它：flex 按
  factor × basis 的比例分摊亏空，因此基名总会被切走一片，在小字号下是亚像素，在 14px
  下约 2px ——这正是 UI 缩放到更大时一个 12 字符的 `shortName.ts` 长出省略号的原因。e2e
  钉住分别测量两个 span，因此“目录先让步”保持为一个测试可以证伪的断言。`shrink-0`
  **单独**会让 chip **对布局不可见地**溢出，同时在屏幕上漫过按钮，因此基名把它与
  `max-w-full` 配对：flex 绝不偷走名字的宽度，但 max-width 仍把它钳制到行，这也是为什么
  e2e 钉住测量*chip 的* `scrollWidth`，而不是头部的。
- **diff 标签页可以比较两个工作区文件。** `DiffTab` 是一个联合 ——
  `GitDiffTab`（带其已加载两侧的 git scope）或 `FileComparisonTab`（一个 `before` 路径，
  完全没有 git 字段）—— 其持久化的 `LayoutDiffTab` 亦然；`isFileComparison(tab)`
  是唯一的谓词。`openFileComparisonInTab` 打开一个比较（身份 `compare:<before>:<path>`，
  命名为 `before → after`），而 `DiffPane` 据此分叉：比较渲染
  [[module-resource-renderers]] 的 `ResourcePreview`（两个文件都从磁盘实时读取，无
  revert/hunk 动作，评论挂在两个文件的工作树锚点上，Split/Inline、
  渲染器与视图状态在标签页上），而不是 git 支撑的 `DiffSurface`。比较没有可能误读的
  scope，因此编译器找出每个仅 git 的读取者（`GitDiffPane`、Changes 面板的活动 diff
  匹配、`updateDiffTabContent`、空白字符、重载水合 —— 后者安装比较而不做 `git.diffFile`
  读取）；更早的版本给它一个占位 `uncommitted` scope，让每个读取者自己记得做 `before`
  检查。其 review 标志与 Review 面板的活动路径覆盖两个文件（`tabReviewPaths`）。
- **文件标签页使用与 diff 相同的渲染器分派。** `FilePane` 描述首个 `fs.readFile`
  元信息（落地前暂定为文本 ——且对早于 `RESOURCE_META_PROTOCOL_VERSION`、从不发送 `meta`
  的宿主永久为文本，那是该宿主自己的遗留表面，因此仅字节渲染无需协议门禁），解析
  `view`，并挂载所选惰性候选。其 Suspense 回退是 `file-pane-loading`；
  预加载的实现从不显示它。`thinkrail/markdown` 仍是 `.md`/`.mdx` 的更高排名匹配，
  因此文档在 `MarkdownPreview` 中打开；`thinkrail/code` 是文本回退，在桌面上用 Monaco，
  在手机类视口用 Pierre `File`；`thinkrail/binary` 显示身份与宿主支撑的下载。
  候选列表单独决定资源切换是否存在。栏或标签页中不再保留任何格式谓词或 preview/source
  字段。

  `MarkdownPreview` 保留上文描述的文档排版、frontmatter 剥离、alert、Mermaid 渲染、
  源行戳、review 评论与有界阅读宽度。这些是渲染器行为，不是分派策略。
- **终端渲染器 + 字体测量。** `TerminalInstance` 刻意运行 xterm 的**默认 DOM 渲染器** ——
   `addon-webgl` *不*加载，加载它会是回退（见 `architecture.md` Decision #11：DOM
  渲染器是触摸的前提，且 `WebglAddon.dispose()` 会泄漏其 WebGL2 上下文，
  而我们逐工作树的终端 churn 会撞上它）。插件恰是 `fit`、`clipboard`、`unicode11` 与
  `web-fonts`；其他任何被固定但未导入的都是死重并会成为下一个读取者的陷阱。`web-fonts`
  是承重而非装饰：我们的代码字体以按字母表的 woff2 子集发布，因此西里尔/中日韩文件在
  xterm 已测量字符单元*之后*才落地（它在构造时测量一次，之后再不测 —— 不像 Monaco，
  会重测不受信任的早期读数）。没有重新测量，非拉丁字形会渲染进为回退字体设定尺寸的单元，
  而 PTY 持有错误的 cols/rows。因此初次附着等待 `relayout()`，执行最后一次 `fit()`，
  然后才捕获 PTY 网格。该等待**由截止时间约束**，因为 pinned addon 中的 `relayout()`
  会等待 `document.fonts.ready` 加上每个已注册字面的 `FontFace.load()` ——
  一个卡住的字体响应会让它无限期保持 *pending*（而不是拒绝），
  而无界等待会让该栏空白且没有 shell。relayout 失败或截止时间到期会回退到构造时测量，
  而不是让该栏搁浅；到期时先中和陈旧 relayout（销毁该 addon 会跳过其重新测量的
  `fontFamily` 切换），因此晚加载完成的字体无法重新布局一个已附着的终端。
  该顺序还防止“以回退宽度附着 + 纠正性 resize”产生快照后的 shell 重绘，
  从而抹掉已重放的行。其预绑定输出缓冲是一个有界等待状态：成功绑定把它过滤到已采纳的
  PTY，而创建失败会清空它并停止接受页面范围的终端帧。
  该失败把宿主的稳定引导渲染为转义后的 DOM 文本而不是可执行的终端输出，并带 Terminal
  Settings 与 Retry 动作。失败的 xterm 子树是惰性的；
  显式重试会让该恢复界面保持挂载且其动作禁用直到请求结算，保住焦点而不偷走无关的
  workbench 焦点。初次 xterm 焦点推迟到成功附着，
  且仅在请求它的终端标签页仍拥有焦点时应用；Retry 让它的覆盖层挂载着撑过成功交接，
  并仅在焦点仍位于该终端区域时恢复 xterm 焦点。聚焦交接在 `focus()`
  之前同步提交已附着状态：失败帧在该提交之前保持惰性，而惰性子树会静默拒绝焦点，
  因此帧时机（在负载较高的机器上，下一动画帧可能先于 React
  的提交）绝不能成为解除它的事物。竞争的分离会使该交接失效并在再次暴露 Take Back
  之前清除其重试状态。**历史重放是输入惰性的：** PTY id 在 xterm
  的重放回调之前保持未采纳，该回调在绑定并排干真正实时的帧之前复查附着新鲜度；因此
  xterm 为已记录终端查询合成的回复绝无法进入实时 shell。PTY 尺寸区分期望的、
  飞行中的与宿主已确认的网格；只有成功的 `terminal.resize` 推进确认，
  因此重连重放无法让一个全屏应用永久定格于宿主从未应用的请求。16 个 ANSI 槽位来自主题的
  `--ansi-*` 域调色板（绝不是语义 UI 文本 token）；在其之上 xterm
  运行一个由主题对比度元信息驱动的**`minimumContrastRatio` 可读性下限**（普通 `4.5`，
  高对比 `7`，位于 `panels/terminalContrast.ts`）。xterm 默认的 `1` 禁用修正，
  这让接近终端背景的颜色（近乎全黑的深色画布上的 `black`）没有下限；
  该比例在不编辑调色板的情况下把解析后的前景色相对实时背景抬升 —— 全部 16 个 HC ANSI
  颜色渲染 ≥ 7:1 且保留色相。该下限**无法**修复 ANSI **dim**（SGR 2）：xterm 以 50%
  不透明度渲染 dim 前景，而对已经高对比的默认前景（Vite 的 `(client)`
  标签是*默认前景色*上的 dim，不是 ansi 颜色）修正从不触发，且浅色画布上的 50% 封顶约
  3.3:1。因此在**高对比主题中 dim 属性会从终端输出中剥除**（`stripAnsiDim`），
  以全前景对比度渲染该文本（≥ AA）。`terminalContrast.test.ts` 门禁复现 xterm
  的颜色数学，以把两个 HC 主题都保持在阈值。**12px 内容内缩**位于 xterm
  **挂载宿主自身的盒子**上（绝对定位，四边 `inset-12`），而不是作为其内边距 —— FitAddon
  从该宿主的测量尺寸推导 cols/rows，因此内边距会多算网格并裁掉最后一行/列；
  内缩该盒子让测量面积等于可见内容面积。
- **IME 控制组合键救援。** xterm 6.0.0 在 CJK 输入法活动时直接丢弃 `Ctrl+<letter>` 与
  `Escape`（上游 #6065）：其组合键表按 `keyCode` 分支，而活动 IME 对每个键都报告哨兵值
  229，因此什么都不匹配且*不发出任何字节* ——中日韩用户无法中断失控进程或退出 vim。
  `TerminalInstance` 的键处理器在 `keyCode === 229` 处拦截 keydown，并通过
  `imeControlBytes` 从 `event.code`（在 IME 下仍准确）推导控制字节，自行写入 PTY；
  任何不是被救援组合键的东西都留给普通文本输入。
- 重型依赖（Monaco / shiki / xterm）通过 `React.lazy(() => import())` 加载，
  以留在急切包之外。加载失败的惰性 chunk（或渲染抛错）由 **shell** 包裹每个区域的
  `components/ErrorBoundary` 容纳（见 `shell/SPEC.md`），
  因此单个面板会降级而不是让应用空白；面板本身不拥有该边界。
- 流式不变量（当聊天落地时）：`text_delta`/`thinking_delta` **APPEND**；
  `tool_execution_update.partialResult` **REPLACE**。
