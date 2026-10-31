# ThinkRail UI 术语参考

> 中文译文，原文见 [`UI-TERMINOLOGY.md`](./UI-TERMINOLOGY.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

ThinkRail Web UI（`apps/web`）的规范词汇，供设计师、ChatGPT 与 pi agent 在设计讨论中使用。

**本文档的范围与规则：**

- **仅前端。** 记录 `apps/web` 当前实现的样子 —— 唯一事实来源。
- **描述性，而非规定性。** 不重新设计、不提建议、不发明名字。当代码库没有清晰的名字，或命名不一致
  时，会用 **⚠ 命名说明（Naming note）** 明确指出。
- 每个区域在适用处列出：**规范名称**（标题）、**实现名称**（React 组件 / 文件）、
  **`data-testid`** 钩子（应用的稳定身份锚点）、**父级**、**子级**、**位置**与**职责**。
- 活跃工作区布局把本窗口前端本地的框架与工作区视图投射为一个桌面 workbench，带递归中心组与可移动
  的侧组。移动端单视图 shell 已设计但尚未实现。

文档自上而下推进：应用布局 → 每个区域 → 共享基础组件 → 术语表。

---

# 应用布局（Application Layout）

整个应用由一个根组件组合而成，并拆分为一个固定的顶栏，覆盖在具有两种互斥状态的主体之上。

- **App Shell** —— 根框架（`Shell`）。
  - **Top Bar**（`<header>`）—— 始终存在。
  - **Body** —— 两种状态之一：
    - **Workspace Workbench** —— 本窗口本地框架与活跃工作区视图的投射。
    - **Welcome Layout** —— 没有活跃工作区时，Welcome 屏幕旁的项目轨道。
  - **Toaster** —— 应用级通知宿主，在任一状态之上挂载一次。

| 规范名称 | 实现 | `data-testid` | 说明 |
|---|---|---|---|
| App Shell | `shell/Shell.tsx` → `Shell` | `shell` | 组合根；拥有主题 DOM 副作用 + 全局热键 |
| Top Bar | `Shell` 内的 `<header>` | — | 见下方命名说明 |
| Workspace Workbench | `shell/WorkspaceWorkbench.tsx` + `shell/layout/Workbench.tsx` | `workbench` | 一个窗口本地框架；工作区专属的资源投射 |
| Welcome Layout | `ResizablePanelGroup`（`autoSaveId="thinkrail-shell-welcome"`） | — | 项目轨道 + Welcome |
| Toaster | `panels/Toaster.tsx` → `Toaster` | — | 见共享基础组件 |

**⚠ 命名说明（Top Bar）：** 代码中没有为这个头部提供组件或具名 prop —— 它是 `Shell.tsx` 中的一个
内联 `<header>` 元素。本参考称其为 **Top Bar**。不要把它与 **Chat Header**（每个聊天标签的栏）或
**Group Header**（workbench 标签条）混淆。

---

# 顶栏（Top Bar）

跨越最顶部的应用级栏。它与布局无关（在两种主体状态下都存在）。

- **父级：** App Shell。
- **位置：** 全宽、顶部；主体之上的固定行（`grid-rows-[auto_1fr]`）。
- **实现：** `shell/Shell.tsx` 中的内联 `<header>`。

子级（从左到右）：

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Logo | `<BrandLogo />`（提供的完整矢量素材） | `brand-logo` | 随主题变化的 ThinkRail 品牌标记 |
| Scope Context | `Shell.tsx` 中的内联块 | `scope-context` | 持久的位置面包屑；有活跃工作区时显示两行 |
| — Scope Project | 内联 `<span>` | `scope-project` | 所属项目名称 |
| — Scope Name | 内联 `<span>` | `scope-name` | 活跃工作区的显示名称，或 `"Project home"` |
| — Scope Branch | 内联 `<span>` | `scope-branch` | 活跃工作区的 git 分支 |
| — Scope Base | 内联 `<span>` | `scope-base` | `· from <baseBranch>`（Default 工作区隐藏） |
| Connection Status | 内联 `<span>` | `connection-status`（`data-status`） | Connected / Connecting… / Disconnected 胶囊，带颜色圆点 |
| Settings Button | 内联 `<button>`（齿轮，`@remixicon/react` `Settings`） | `open-settings` | 通过 `store.openSettings()` 打开 Settings Dialog |

**⚠ 命名说明（Scope Context）：** `data-testid` 是 `scope-context`，规格文本称它为「location
context」。本参考采用 **Scope Context** 作为规范名；「location context」是散文中的另一种说法。

---

# 项目工具（Projects Tool）

单例的 Projects 工具，初始位于左侧。它的 Projects Rail 视图列出项目，并在每个项目下方展开其
工作区。

- **规范名称：** Projects Tool；**Projects Rail** 命名它的导航视图。
- **实现：** `panels/ProjectTree.tsx` → `ProjectTree`，由 shell 的 Projects 工具渲染。
- **`data-testid`：** 工具主体包装是 `left-nav`。
- **父级：** Workspace Workbench（一个可移动的单例侧工具），或 Welcome Layout。
- **位置：** 初始位于左侧；其所属组可以调整大小、折叠、隐藏或移动。
- **职责：** 打开仓库、选择项目（一种「project home」手势，会取消选择任何活跃工作区）、关闭项目、
  展开/折叠以显示工作区、创建/选择/移除工作区，以及在外部编辑器 / 文件管理器中打开工作区。

**⚠ 命名说明（Projects Tool）：** 遗留名称仍然共存 —— 组件是 `ProjectTree`，它的导航视图是
**Projects Rail**，兼容性测试 id 是 `left-nav`。「Left Nav」与「Left Sidebar」都无法描述持久的
位置，因为该工具可以移动。

子级：

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Add-Project Button / Menu | `panels/AddProjectMenu.tsx` → `AddProjectMenu`（轨道上的「+」） | `add-project-menu` | Open project / Enter host path… / Clone repository… / Recents 下拉菜单 |
| Project Row | `ProjectTree` 中的内联行 | `project-item` | 一个项目（git 仓库）；点击即选中它（project home） |
| — Project Expander | 折叠箭头控件 | `project-expand` | 展开/折叠该项目的工作区列表 |
| — Project Name | 内联 `<button>` | `project-name` | 选中该项目（project home） |
| — Workspace Count | 内联 `<span>` | `project-workspace-count` | 折叠行上该项目 worktree 工作区的数量 |
| — Add-Workspace Button | 内联「+」 | `add-workspace` | 打开 New Workspace Dialog |
| — Project Actions Menu | 行上的 Context Menu | `project-actions` | Create workspace（`project-menu-create-workspace`）/ Open existing worktree（`project-menu-open-existing-worktree`）/ Rename（`project-menu-rename` → 内联 `project-name-input`）/ Close project（`project-menu-close`） |
| Workspace Row | `ProjectTree` 中的内联行 | `workspace-item` | 一个工作区（git worktree）；两行：名称 + 分支 |
| — Workspace Name | 内联 `<span>` | `workspace-name` | 显示名称 |
| — Workspace Branch | 内联 `<span>` | `workspace-branch` | git 分支（弱化、比例字体的元数据；等于名称时隐藏） |
| — Working Badge | `components/RunningIcon.tsx` → `RunningIcon` | `running-icon` | 该行的会话运行期间，代替其身份图标显示的实时 TR 品牌徽标（项目行上也有）；标签「Agent working」 |
| — Attention Dot | `components/AttentionDot.tsx` → `AttentionDot` | `attention-dot` | 表示需要输入 / 未读结果的静态强调色圆点；标签「Needs attention」 |
| — Workspace Actions Menu | `MoreVertical` Dropdown Menu | `workspace-menu` / `workspace-actions` | Open in（`workspace-open-in`）/ Copy path / Reveal / Remove workspace |
| — Remove-Workspace Item | 动作菜单中的菜单项 | `workspace-remove` | 打开 Confirm Dialog；在 Default 工作区上不显示 |

**Default Workspace** 行（`kind === "default"` —— 项目文件夹本身）固定在最前，用 `House` 图标代替
`GitBranch` 字形，并且没有 Remove 项 —— 但它获得与任何 worktree 相同的 Open in / Copy path /
Reveal 菜单。

---

# 欢迎屏幕（Welcome Screen）

当没有活跃工作区时（全新安装，或归档最后一个工作区之后），显示在 Welcome Layout 的右列。与
Workspace Workbench 互斥。

- **规范名称：** Welcome Panel（又称 Welcome Screen）。
- **实现：** `panels/WelcomePanel.tsx` → `WelcomePanel`。
- **父级：** App Shell（Welcome Layout，`id="welcome"` 面板）。
- **位置：** Projects Rail 旁宽右列中的居中内容。
- **职责：** 首次接触表面 + **模式分叉** —— 把「Start building」（隔离 worktree）与
  「Work in project folder」（Default 工作区）配对呈现。

子级：

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Welcome Heading | 内联 hero 标题 | `welcome-title` | 项目名称，没有项目时为 `PRODUCT_NAME` |
| Provider Warning Banner | `panels/ProviderWarningBanner.tsx` → `ProviderWarningBanner` | — | 仅在未连接任何 provider 时显示的金色横幅 |
| Project Trust Notice | `panels/ProjectTrustNotice.tsx` → `ProjectTrustNotice` | `project-trust-notice` | 工作区创建前的信任表面，说明一次授权会加载什么，带 Stop trusting |
| Primary Card (CTA) | `WelcomePanel` 中的 `Card` | `welcome-cta` | 填充式主要动作 |
| Action Card | `WelcomePanel` 中的 `Card` | `welcome-action` | 安静的次要动作 |

---

# 中央工作台（Center Workbench）

中央工作区是一棵由一到四个标签组构成的递归树。每个组拥有自己的 **Group Header**（标签条加组
控件）、选择、预览槽与主体；水平或垂直分隔条调整相邻组的大小。

- **规范名称：** Center Workbench（又称 Center Tabbed Area / Editor Area）。
- **实现：** `shell/layout/Workbench.tsx`（拓扑与容器装饰），由 `shell/WorkspaceWorkbench.tsx`
  集成（功能主体与本地状态编排）。
- **`data-testid`：** 兼容性包装是 `center-tabs`；叶子是 `center-group`，Group Header 是
  `center-tab-strip`（一个遗留名称），分隔条是 `center-split-resize`。
- **父级：** Workspace Workbench。
- **职责：** 承载 **File**、**Chat**、**Diff**、**Changes review**、可重水化的 **Document** 与
  **Terminal** 标签；拥有递归摆放、按组的预览/保留语义、移动、溢出与焦点恢复。

**⚠ 命名说明（tab element）：** file/chat/diff/changes/document 标签携带
`data-testid="editor-tab"`；terminal 摆位携带 `terminal-tab`。资源种类也可以通过 `data-kind`
获取。「Editor tab」是兼容性测试钩子，不是种类。

子级：

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Center Group | `Workbench` 中的 `CenterGroupView` | `center-group` | 一个叶子：标签条 + 本地选中的主体 |
| Group Header | `TabStrip` 加上 `Workbench` 中注入的组控件 | `center-tab-strip` | 固定单行装饰，包含 ARIA tablist、滚动、可搜索溢出与组动作 |
| Tab | `WorkbenchTab` | `editor-tab` / `terminal-tab` | 一个规范摆位；`data-preview` 标记预览 |
| — Tab Close | 内联 `X` 动作 | `editor-tab-close` | 关闭该摆位（不是底层的持久资源） |
| New-Chat Button | 注入的中央动作 | `new-chat` | 打开一个全新的聊天标签 |
| Chat-History Menu | `shell/WorkspaceChatHistory.tsx` | `chat-history` | 重新打开或删除已关闭/仅存在于磁盘的聊天 |
| Editor Pane | 注入的选中主体 | `editor-pane` | 本地选中的中央主体 |
| Workspace-Ready Receipt | 注入的空状态 | `workspace-ready` | 最终中央叶子为空时的定位回执 |
| — Start-Chat Action | 内联动作 | `start-chat` | 从空回执发起新聊天 |

标签主体组件：

| 规范名称 | 实现 | 标签种类 | 职责 |
|---|---|---|---|
| File Pane | `panels/FilePane.tsx` → `FilePane` | `FileTab` | 注册表分发的文件查看器；更丰富的格式获得渲染器切换 |
| — Code File | `resource-renderers/support/MonacoEditor.tsx` / `resource-renderers/renderers/code/PierreFile.tsx`（懒加载） | — | 桌面上的只读 Monaco 源码，手机上的 Pierre 源码 |
| — Markdown Preview | `resource-renderers/support/MarkdownPreview.tsx` → `MarkdownPreview`（懒加载） | — | 渲染后的 markdown（文档皮肤） |
| Diff Pane | `panels/DiffPane.tsx` → `DiffPane` | `DiffTab` | 文件 diff；Split\|Inline 或 Source\|Rendered 切换 |
| — Source Diff | `resource-renderers/renderers/code/PierreDiff.tsx`（懒加载） | — | 只读的 Pierre 拆分/统一源码 diff，带评审与 hunk 动作 |
| — Rendered Diff | `resource-renderers/support/RenderedDiff.tsx` → `RenderedDiff`（懒加载） | — | 富 markdown diff（`<ins>`/`<del>`） |
| Changes Review Pane | `panels/ChangesReviewPane.tsx` → `ChangesReviewPane` | `ChangesTab` | 一个范围内的每个变更文件作为区块；**Stacked** \| **One file** 模式（见其自己的章节） |
| Chat View | `chat/ChatView.tsx` → `ChatView`（懒加载） | `chat` | agent 对话（见其自己的章节） |
| Document Pane | `WorkspaceWorkbench` 引用渲染器 | `document` | 可重水化的虚拟文档，例如 TODO 计划 |
| Terminal Body | `panels/TerminalWorkbench.tsx` → `TerminalWorkbenchBody` | `terminal` | 按可见性门控的 xterm 表面 |

---

# 聊天视图（Chat View）

agent 对话，渲染在 Center Tabbed Area 的 Chat 标签内。

- **规范名称：** Chat View。
- **实现：** `chat/ChatView.tsx` → `ChatView`（唯一的应用集成部分；连接 store + 传输层）。
  它下面的所有渲染器都是展示性的/props 驱动的。
- **父级：** Center Workbench（一个选中的 `chat` 标签主体）。
- **位置：** 填满 Editor Pane。纵向：Chat Header（顶部）→ Message List（中部，滚动）→
  Composer（底部）。
- **职责：** 把 pi 的规范消息 / 内容块模型渲染为折叠行；拥有 composer、历史浮层与对话框。

## 聊天头部（Chat Header）

- **规范名称：** Chat Header。
- **实现：** `chat/ChatHeader.tsx` → `ChatHeader`。
- **父级：** Chat View。
- **位置：** Chat View 的细顶栏。
- **子级 / 槽位：**
  - **Status Entries** —— 内联弱化的 `statusEntries` 文本（扩展状态）。
  - **Session Stats Bar** —— `chat/SessionStatsBar.tsx` → `SessionStatsBar`（token/成本统计）。
  - **Resources Trigger** —— `chat/resources/ResourcesButton.tsx` → `ResourcesButton`
    （`data-testid="resources-trigger"`）；活跃资源计数 —— 实时工作加上已连接/正在启动的 MCP
    服务器（或在计数不权威时显示 `—`）—— 在工作运行期间呼吸；切换 **Resources Inspector**。
    它旁边是 **MCP Attention Marker**（`ResourcesAttention`、`resources-mcp-attention`），当某个
    服务器需要登录、失败或等待批准时打开 Settings › MCP servers。
  - **Skills Button** —— `chat/SkillsButton.tsx` → `SkillsButton`（`data-testid="open-skills"`）；
    打开 **Skills Dialog**（`chat/SkillsDialog.tsx` → `SkillsDialog`）。

## 资源（Resources）

当前聊天的后台命令与子代理，通过一套行语法渲染在由 `chat/resources/` 拥有的三个表面上。

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Resources Trigger | `resources/ResourcesButton.tsx` → `ResourcesButton` | `resources-trigger`（`data-active-count`） | inspector 的头部切换；实时计数 |
| Resources Dock | `resources/ResourcesDock.tsx` → `ResourcesDock` | `resources-dock`（`resources-dock-toggle`） | Composer 上方的实时行；inspector 打开时隐藏；超过四行时折叠 |
| Resources Inspector | `resources/ResourcesInspector.tsx` → `ResourcesInspector` | `resources-inspector` | 覆盖在 Message List 上的非模态就地对话框：名册（Active / Finished / MCP servers）+ 选中行的详情 |
| MCP Server Row | `resources/mcpRow.tsx`（私有） | `resource-mcp`（`data-name`、`data-state`；`resource-mcp-state`、`resource-mcp-disable` / `-enable` / `-reconnect` / `-settings`），旁边是区块头部的 Settings 链接 `resources-mcp-settings` | inspector 第三区块中该聊天的一个 MCP 服务器：状态标签、工具数量、传输层、按聊天的动作 |
| Resource Row | dock 与 inspector 内共享的行 | `resource-command` / `resource-subagent`（`data-resource-id`、`data-status`、`data-state`） | 一条命令或一个子代理：字形 · 名称 · 活动 · 状态 · 已用时间。在 dock 中名称与活动是一个 Inspect 按钮（`resource-inspect`）；在 inspector 中行包装（`data-selected`）持有 listbox 选项（名称 · 状态 · 已用时间，位于被截断的活动行之上；可访问名 `name: activity, state`）及其同级的 Stop。Stop 在 hover 与聚焦时出现（`resource-stop`） |
| Command Log View | `resources/CommandLogView.tsx` → `CommandLogView` | `command-log-output` / `command-log-unavailable` | 有界的纯文本输出；inspector 中命令的详情主体 |
| Subagent Transcript Pane | `chat/SubagentTranscriptDialog.tsx` → `SubagentTranscriptPane` | `subagent-transcript` | 只读的子级转录；inspector 中子代理的详情主体（也是 Subagent Transcript Dialog 的主体） |
| Stop-All Action | inspector 中的名册头部 | `resources-stop-all` → `resources-stop-all-confirm` | 在一次点名数量的确认之后，停止该聊天的活跃子代理 |
| Still-Running Chip | `chat/turns.tsx` 中的 `TurnDivider` | `turn-divider-running` | 最新轮次分隔条上的「N still running」；打开 inspector |

## 消息列表（Message List）

- **规范名称：** Message List（又称 Transcript）。
- **实现：** `ChatView` 中的 `react-virtuoso` `Virtuoso`，通过 `chat/rows.ts`（`deriveRows`）
  渲染**派生行**，由 `chat/turns.tsx` → `ChatTurnView` 分发。
- **父级：** Chat View。
- **位置：** 头部与 composer 之间的滚动中部区域。
- **职责：** 把 pi 的轮次渲染为折叠行，带渐进披露。

**⚠ 命名说明（Message vs Turn vs Row）：** 代码区分三个层级。**Turn**（`ChatTurn`）是 pi 的消息级
单位；**Row**（`ChatRow`）是派生的渲染单位（折叠跨越轮次边界）；每个渲染出的消息元素携带
`data-testid="chat-message"` 与 `data-role`。渲染单位用 **Row**，pi 消息用 **Turn**；
「Message」是通用表面术语。

行 / 消息渲染器（除非另有说明，均在 `chat/turns.tsx` 中）：

| 规范名称 | 实现 | `data-testid` | 行种类 | 职责 |
|---|---|---|---|---|
| Turn Dispatcher | `ChatTurnView` | — | — | 把派生行分发给它的渲染器 |
| User Message | `UserTurn` | `chat-message`（`data-role="user"`） | `user` | 一个用户提示词气泡 |
| Assistant Markdown | `chat/Markdown.tsx` → `Markdown` | — | `markdown` | 助手文本（GFM + shiki） |
| System Notice | `SystemTurn` | `chat-message` | `system` | web 本地的系统通知 |
| Error Turn | `ErrorTurn` | `chat-message` | `error` | 持久的带色调失败通知（从不折叠） |
| Retry Indicator | `RetryIndicator` | `retry-indicator` | `retry` | 重试倒计时（轮次 / 摘要） |
| Tool Card | `chat/ToolCard.tsx` → `ToolCard` | `tool-card`（`-toggle`） | `tool` | 一次主要工具调用（可折叠框架） |
| Activity Group | `chat/ActivityGroup.tsx` → `ActivityGroup` | — | `activity` | 折叠的常规步骤串（「N steps · …」） |
| Ext-UI Prompt | `chat/ExtUiPrompt.tsx` → `ExtUiPrompt` | `ext-ui-prompt` | — | 扩展的聊天内提问（`pi.extensionUi` 对话框）：在其工具的卡片上，否则停靠在 composer 上方 |
| Turn Divider | `TurnDivider` | `turn-divider` / `turn-divider-<id>` | `divider` | 轮次结束摘要 + 产物芯片 |
| — Artifact Chip | `ArtifactChip` | `turn-divider-specs` / `turn-divider-files` | — | 「N specs」/「N files changed」的深链/披露；带宿主轮次回执时，文件芯片读作 `N files changed · +a −r` |
| — Artifact List | `ArtifactList` | `<testid>-list` / `-list-item` | — | 展开的逐路径列表（每文件的 `+/−` 来自回执） |
| — Review-Turn Chip | `TurnDivider` 中的内联 | `turn-divider-review` | — | 「Review turn」：在该轮次的范围内打开 Changes Review Tab（仅在有回执时） |
| Stream Indicator | `chat/StreamIndicator.tsx` → `StreamIndicator` | `stream-indicator` | — | 实时流式状态：Working Badge + 阶段标签（「Thinking…」、「Running bash…」） |

## 工具调用（Tool Call）

- **规范名称：** Tool Card（框架）；Tool Renderer（主体）。
- **实现：** `chat/ToolCard.tsx` → `ToolCard`（可折叠框架），主体通过 `chat/toolRegistry.tsx` →
  `registerToolRenderer` 注册。未注册的工具回退到 `DefaultToolRenderer`。
- **父级：** Message List（一个 `tool` 行），或 Activity Group 的步骤行（常规工具）。
- **职责：** 渲染一次工具调用；「card」装饰使用 `ToolCard`，「bare」装饰拥有自己的框架。

内建工具渲染器（除非另有说明，均在 `chat/tools/` 中）：

| 规范名称 | 实现 | 显著度 | 职责 |
|---|---|---|---|
| Bash Card | `BashCard.tsx` → `BashCard` | 常规 | 终端命令块 |
| Read Card | `ReadCard.tsx` → `ReadCard` | 常规 | 文件读取（路径 + 高亮后的文件） |
| Write Card | `ReadCard.tsx` → `WriteCard` | 常规 | 文件写入 |
| Edit Card | `EditCard.tsx` → `EditCard` | 常规 | 编辑（删除/新增行 diff） |
| Ask-User-Question Card | `thinkrail-extensions/ask-user-question/web/` → `AskUserQuestionCard` | 主要，「bare」 | 内联问卷 |
| Visualization Card | `thinkrail-extensions/visualize/web/` → `VisualizationCard` | 主要，展开 | Mermaid 图表 / 对比卡片 |
| Web Card(s) | `tools/web/` | 常规 | 搜索/获取渲染器 |
| Default Tool Renderer | `DefaultToolRenderer` | 常规 | 未注册工具的回退 |

## Composer

- **规范名称：** Composer。
- **实现：** `chat/Composer.tsx` → `Composer`。
- **父级：** Chat View。
- **位置：** Chat View 的底部。
- **职责：** 提示词输入 + send/steer/followUp/abort、`@` 提及、`/` 斜杠命令、模板槽会话、图片
  粘贴/拖放、`↑` 回忆，以及打开历史的入口。

子级 / 关联表面：

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Model Selector | `chat/ModelSelector.tsx` → `ModelSelector` | — | 模型选择器（+ Refresh catalog） |
| Thinking Selector | `chat/ThinkingSelector.tsx` → `ThinkingSelector` | — | 思考/强度级别选择器 |
| Slash-Command Menu | `chat/SlashCommandCompletion.tsx` → `SlashCommandMenu` | `slash-templates-empty`（页脚） | `/` 命令 + 模板补全 |
| Send Button | 内联（`ArrowUp` / `Square` 中止） | — | 发送 / 引导 / 跟进 / 中止 |
| History-Open Button | 内联（`History` 图标） | `history-open` | 打开 History Overlay |
| Slot Hint Chip | 内联胶囊 | `slot-hint` | 模板槽会话进度（`slot n/m · ⇥ next · esc done`） |
| Slot Highlight | 内联背景 span | `slot-highlight`（`data-slot-state`） | 带色调的模板槽范围 |

## 历史浮层（History Overlay）

- **规范名称：** History Overlay。
- **实现：** `chat/HistoryOverlay.tsx` → `HistoryOverlay`，由 `chat/useHistorySearch.ts` 驱动。
- **父级：** Chat View（由 Composer / `Ctrl+R` 打开）。
- **职责：** 历史回忆/搜索；紧凑的单列，`Tab` 扩展为两栏放大布局（结果 + 预览）。

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Results List | `HistoryOverlay` 内 | `history-results` | 提示词 / 消息命中 |
| Preview Pane | `HistoryOverlay` 内 | `history-preview` | 选中命中的全文 |
| Scope Picker | `HistoryOverlay` 内 | `history-scope` / `-option` | This chat / Workspace / Project / Everywhere |
| Jump Action | `HistoryOverlay` 内 | `history-jump`（`-shortcut`） | 跳到聊天 |
| Save-as-Template Action | `HistoryOverlay` 内 | `history-save-template`（`-shortcut`） | 打开 Template Editor Dialog |

## 聊天计划（Chat Plan (TODO plan)）

- **规范名称：** Chat Plan。
- **实现：** `chat/TodoList.tsx` → `TodoList` + `panels/PlanRailPane.tsx` → `PlanRailPane`
  （轨道主体）+ `panels/PlanPane.tsx` → `PlanPane`（中央标签页，通过 `openPlan` 打开）。
- **父级：** 右侧轨道工具（`plan`）承载紧凑的 TODO 列表；中央编辑器标签承载文档规模的 Plan 页。
- **职责：** 呈现该聊天的 `pi-todos` 计划（分组优先、按状态排序）。TODO 列表是 Files / Specs /
  Review 旁边的侧轨工具。

---

# 变更评审标签（Changes Review Tab）

针对一个 diff 范围的连续评审表面：该范围的每个变更文件作为一个 **File Section** 呈现在中央标签
中。单击 Changes 行，或点击聊天的 **Review-Turn Chip**，即可打开它；Files 工具的 Changed 模式是
它的导航器。

- **规范名称：** Changes Review Tab；其主体是 **Changes Review Pane**。
- **实现：** `panels/ChangesReviewPane.tsx` → `ChangesReviewPane`，按文件渲染
  `panels/ChangesFileSection.tsx` → `ChangesFileSection`，以及
  `panels/ChangesReviewGuide.tsx` → `ChangesReviewGuide` 轨道。
- **`data-testid`：** 主体 `changes-review`；其标签是一个 `editor-tab`，带 `data-kind="changes"`。
- **父级：** Center Workbench（一个选中的 `changes` 标签主体；每个工作区 + 范围一个）。
- **位置：** 填满 Editor Pane。纵向：Review Toolbar（顶部）→ 区块（中部，滚动）→ Triage Bar
  （底部，仅 Stacked）；Review Guide 是区块左侧的轨道。
- **视图模式：** **Stacked**（默认；所有区块在一个虚拟化列表中）与 **One file**（在 Walk Bar 下
  一次一个区块），由工具栏的 Stacked\|One file 分段选择，并在应用范围内保持。
- **职责：** 按区块懒读取每个文件的 diff，保持该标签的已查看 / 已折叠 / 已保留状态，并引导评审者
  走过该范围。

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Review Toolbar | `ChangesReviewPane` 中的内联 `role="toolbar"` | `changes-review-toolbar` | 范围 + 目标（`changes-review-scope`）、`N files · +N −M · n/N viewed`（`changes-review-summary`、`changes-review-viewed-count`），以及下面的控件 |
| — Send Review | `SendAllReviewsButton` | `changes-review-send` | 「Send review (N)」 |
| — Collapse / Expand All | `HeaderIconButton` | `changes-review-collapse-all` / `changes-review-expand-all` | 一次处理所有区块（仅 Stacked） |
| — Review Guide Toggle | `HeaderIconButton` | `changes-review-guide-toggle` | 显示 / 隐藏 Review Guide |
| — Whitespace Toggle | `HeaderIconButton`（¶） | `changes-review-toggle-whitespace` | 隐藏空白变更 |
| — Diff View Toggle | `ToggleSegment` | `changes-review-toggle-split` / `changes-review-toggle-inline` | Split \| Inline（仅桌面） |
| — Layout Toggle | `ToggleSegment` | `changes-review-layout-stacked` / `changes-review-layout-single` | Stacked \| One file |
| File Section | `ChangesFileSection` | `changes-section`（`data-path`、`data-status`、`data-collapsed`、`data-viewed`） | 一个变更文件：头部 + 懒读取的 diff 主体，像 Diff Pane 一样分发 |
| — Section Header | 内联；Stacked 中粘性 | `changes-section-header` | 折叠箭头（`changes-section-toggle`）、状态、路径（`changes-section-path`）、`+N −M`、`k/n kept`（`changes-section-kept`）、渲染器分段、Revert file（`changes-section-revert`）、Open as tab（`changes-section-open-tab`）、动作菜单（`changes-section-menu`） |
| — Viewed | 内联切换按钮 | `changes-section-viewed`（`aria-pressed`） | 在该标签中把文件标记为已查看 |
| — Collapsed Notice | 内联 | `changes-section-collapsed` / `changes-section-expand` | 默认折叠的大文件或生成文件：Expand / Open as tab |
| — Section states | 内联 | `changes-section-loading` / `changes-section-error` / `changes-section-retry` | 保留高度的占位；读取失败 + Retry |
| Hunk Toolbar | `resource-renderers/renderers/code/PierreDiff.tsx` 中的 `HunkToolbar` | `hunk-toolbar`（`data-kept`） | 逐 hunk 的 Source Diff 动作：Revert（`hunk-revert`）、Ask agent（`hunk-ask-agent`） |
| — Keep | 内联切换按钮 | `hunk-keep`（`aria-pressed`） | 「Keep」/「Kept」hunk 分诊；仅可变更范围内的评审标签 |
| Triage Bar | `ChangesReviewPane` 中的内联（Stacked） | `changes-review-triage` | 底栏：`n of N reviewed` |
| — Progress Bar | 内联 | `changes-review-progress` | 已查看比例 |
| — Next Unreviewed | 内联按钮 | `changes-review-next-unreviewed` | 「Next unreviewed」文件（`J`，循环） |
| — Mark All Viewed | 内联按钮 | `changes-review-mark-all` | 「Mark all viewed」 |
| Walk Bar | `ChangesReviewPane` 中的内联（One file） | `changes-review-walk` | Prev / Next 文件（`changes-review-prev` / `changes-review-next`、`Alt+↑` / `Alt+↓`）、`n / N`（`changes-review-counter`）、Mark viewed（`changes-review-mark-viewed`、`V`） |
| Review Guide | `ChangesReviewGuide` | `changes-review-guide` | 「Review guide」左轨道（Stacked，手机上不显示），当评审有 guide 或存在未解决的 agent 发现时显示：结论 + 摘要（`changes-review-guide-summary`） |
| — Guide Step | 内联按钮 | `changes-review-guide-step`（`data-active`） | 一个「Suggested reading order」步骤；其文件在范围之外时禁用 |
| — Guide Finding | 内联卡片 | `changes-review-guide-finding`（`data-active`） | 一个未解决的 agent 发现；「Fix this one」（`changes-review-guide-fix`） |
| — Guide Prev / Next | 内联按钮 | `changes-review-guide-prev` / `changes-review-guide-next` | 走查步骤：Start / Next / Restart（`N`；`P` 返回） |
| — Apply Fixes | 内联按钮 | `changes-review-guide-apply` | 「Apply fixes」：把所有未解决的发现发给 agent |
| Large-Scope Notice | 内联 | `changes-review-large-notice` | 超过 50 个文件：建议使用 One file；可关闭 |
| End Note | `ReviewListFooter` | `changes-review-end` | Stacked 列表尾部的「End of changes · N files」 |
| Empty / Error states | 内联 | `changes-review-empty` / `changes-review-error` / `changes-review-retry` / `changes-review-all-changes` | 该范围内没有变更；读取失败 + Retry；范围的提交或轮次快照已消失 + Review all changes |

---

# 侧边工作台（Side Workbench）

左、右两侧是可独立调整大小的区域。每侧包含一个有序的垂直标签组栈；组可以调整大小、折叠成 27px 的
行、移动，并在为空时消失。隐藏一侧会留下一条紧凑的恢复轨道。侧边排列在工作区范围内共享；选择与
焦点是浏览器本地的。

- **规范名称：** Left Side、Right Side、Side Group、Hidden-Side Rail。
- **实现：** `shell/layout/Workbench.tsx`（`SideStack`、`SideGroupView`）。
- **`data-testid`：** `left-stack`、`right-stack`、`left-layout-rail`、`right-layout-rail`、
  `side-group-fold`，以及按侧分别的组调整大小把手。
- **父级：** Workspace Workbench。
- **职责：** 摆放单例 shell 工具与终端，而不把功能面板耦合到它们的位置。文件、diff、聊天与文档
  保持仅限中央；终端可以跨域。

每个 Side Group 都有一个 **Group Header**；折叠时该头部保留为 27px 的行，而其关联主体保持原生隐藏
且未挂载。初始的 Balanced 摆放把 Projects 放在左侧，Specs 与 Files 放在第一个右侧组，然后 Review
放在第二个；默认终端目录加入最后一个组。这是启动行为，不是固定层级。

## 侧边工具（Side Tools）

| 规范名称 | `data-testid` | 实现 / 职责 |
|---|---|---|
| Projects | `tab-projects`，主体 `left-nav` | `panels/ProjectTree.tsx`；项目与工作区 |
| Specs | `tab-specs` | `panels/SpecsPanel.tsx`；自动刷新的只读规格图谱，带仅错误的 `specs-retry` |
| Files | `tab-files` | `panels/FilesPanel.tsx`；**All \| Changed** 开关（`files-mode-all` / `files-mode-changed`，应用范围 `store.filesMode`，默认 All）压在 `panels/FileTree.tsx` 上（worktree 树；变更文件携带按状态着色的名称 + 字母 `file-status`，含变更的文件夹携带琥珀色名称 + 圆点 `file-folder-dot`；被 git 忽略的行 `data-status="ignored"` 使用 subtle 颜色；活跃文件/diff 标签的行被选中并自动展开）加上 **Collapse All Button** `files-collapse-all`，或 `panels/ChangesPanel.tsx`（见下） |
| Review | `tab-review` | `panels/ReviewPanel.tsx`；评审手风琴与发送动作 |

`WorkspaceWorkbench` 拥有长生命周期的 Specs 与 Review 读取，因此即使这些工具主体未被选中，徽标、
评审标记与产物分类也保持最新。

## Changes 模式（Files 工具）（Changes mode (Files tool)）

Files 工具的 Changed 模式在 All | Changed 开关行下面保留自己的功能工具栏：范围菜单、目标分支选择器，
以及 List\|Tree 切换（默认为 Tree）。单击一行会打开该范围的 **Changes review tab** —— 每个变更文件
作为一个堆叠区块，或在其 **One file** 模式下一次一个 —— 而侧边栏是该标签的导航器（视图内区块的行
为活跃，已查看文件带标记）；双击或 *Open as tab* 打开逐文件的 diff 标签。`ChangesPanel` 保持与
排布无关，并由 `FilesPanel` 供给其 git 状态。

| 规范名称 | 实现 | `data-testid` | 职责 |
|---|---|---|---|
| Changes Header | `ChangesPanel` 中的内联 `role="toolbar"` | `changes-view-toggle` | 范围菜单、目标分支选择器、List \| Tree |
| Changes Scope Menu | `panels/ChangesScopeMenu.tsx` → `ChangesScopeMenu` | `changes-scope-trigger`（`changes-scope-label`） | 范围胶囊：All changes（`changes-scope-all`）、Uncommitted changes（`changes-scope-uncommitted`）、Last turn、Agent turns、Commits（`changes-scope-commit`） |
| — Last Turn | 菜单项 | `changes-scope-last-turn` | 「Last turn · N files」：最新的 agent 运行；在某次运行产生变更之前禁用 |
| — Agent Turns | 「Agent turns」下的菜单项 | `changes-scope-turn`（`data-turn`） | 最近的 agent 运行，最新在前（仅当多于一个时） |
| Branch Picker | `panels/BranchPicker.tsx` → `BranchPicker` | `changes-target-picker` | 目标分支（`vs <base>`） |
| Changes View Toggle | `ui/toggle-segment.tsx` | `changes-toggle-list` / `changes-toggle-tree` | List \| Tree |
| Changes List Row | `ChangesPanel` 中的内联 | `change-item`（`data-active`、`data-viewed`） | 扁平的变更文件行 |
| Changes Tree | `panels/ChangesTree.tsx` | 文件行 `change-item`，文件夹 `change-tree-folder` | 变更文件的文件夹树 |
| Viewed Mark | `panels/ViewedMark.tsx` → `ViewedMark` | `change-viewed` | 评审标签标记为已查看的行上的对勾字形 |
| Diff-Stat Badge | `panels/DiffStatBadge.tsx` | — | 每文件 / 每文件夹的 `+N −M` |
| Empty state | 内联 | `changes-empty` | 该范围内没有变更 |
| Error state | 内联 | `changes-error` / `changes-retry` | 读取失败 + Retry |

**⚠ 命名说明（Changes Header）：** `changes-view-toggle` 标记整个头部工具栏，而不是 List\|Tree
分段（`changes-toggle-list` / `changes-toggle-tree`）。

---

# 终端摆位（Terminal Placement）

终端是一个可移动的资源标签，而不是永久的右下角面板。它可以占据中央组或任意侧组。每个可见、展开
组中选中的终端会挂载；非活跃、折叠或隐藏的终端摆位不附加。一个终端身份在每个浏览器中至多有一个
挂载的主体。

- **实现：** `panels/TerminalWorkbench.tsx` 集成宿主目录与关闭流程；`panels/TerminalInstance.tsx`
  是懒加载的 xterm 主体。
- **`data-testid`：** 摆位 `terminal-tab`，兼容性主体包装 `terminal-panel`，实例
  `terminal-instance`，添加动作 `terminal-add`。
- **职责：** 在布局控制摆位与可见性的同时保持宿主拥有的 shell 身份。关闭时保留显式的忙进程确认
  流程。

**⚠ 命名说明（Status Bar）：** ThinkRail **没有专用的状态栏组件**。最接近的表面是 Top Bar 的
Connection Status 与 Chat Header 的 Session Stats Bar。

---

# 设置对话框（Settings Dialog）

- **规范名称：** Settings Dialog。
- **实现：** `panels/SettingsDialog.tsx` → `SettingsDialog`（store 驱动；打开状态在 store 中，
  因此多个表面可以深链接打开它）。
- **父级：** App Shell（在 Top Bar 的 `<header>` 内挂载一次）。
- **位置：** 模态覆盖层；两栏外壳（左侧区块轨道 + 可滚动内容栏；移动端把轨道折叠为水平条）。
- **区块（每个都是自己的组件）：**

| 规范名称 | 实现 | 职责 |
|---|---|---|
| Providers | `panels/ProvidersSettings.tsx` → `ProvidersSettings` | 应用内 provider 认证（+ `panels/JetBrainsAiCard.tsx` → `JetBrainsAiCard`） |
| MCP servers | `panels/McpSettings.tsx` → `McpSettings`（+ `panels/McpServerDialogs.tsx`） | 活跃工作区的 MCP 服务器（`mcp-server-row`、`mcp-add-dialog`、`mcp-approve-dialog`） |
| GitHub | `panels/GithubSettings.tsx` → `GithubSettings` | 本地 GitHub 连接状态 |
| Appearance | `panels/AppearanceSettings.tsx` → `AppearanceSettings` | 主题选择器 |
| Layout | `shell/LayoutSettings.tsx` → `LayoutSettings`（注入到对话框中） | 默认/应用/捕获 workbench 预设与侧组上限 |
| Terminal | `panels/TerminalSettings.tsx` → `TerminalSettings` | 终端重放预算 |
| Templates | `panels/TemplatesSettings.tsx` → `TemplatesSettings` | 全局 / 项目提示词模板（`template-row`、`template-starters`） |
| Privacy | `panels/PrivacySettings.tsx` → `PrivacySettings` | 匿名使用分析开关 |
| General | 变暗的导航项 | 「Soon」占位 |

---

# 共享 UI 基础组件（Shared UI Primitives）

可复用的构件（shadcn/ui、Radix），由 `packages/ui/` 拥有，并用 ThinkRail token 主题化。按文件导入
为 `@thinkrail/ui/<primitive>`（不使用 barrel）。

| 规范名称 | 实现 | 说明 |
|---|---|---|
| Button | `packages/ui/button.tsx` | `default` / `destructive` / `outline` / `ghost` |
| Dialog (Modal) | `packages/ui/dialog.tsx` | **Modal** 基础组件；可选 `hideClose` |
| Dropdown Menu | `packages/ui/dropdown-menu.tsx` | 高度受限、可滚动的菜单；通过 `DropdownMenuSub*` 提供子菜单 |
| Context Menu | `packages/ui/context-menu.tsx` | 右键菜单；与 Dropdown Menu 共享 `menu-styles.ts` |
| Popover | `packages/ui/popover.tsx` | 可选 `container` 传送目标 |
| Command | `packages/ui/command.tsx` | cmdk 组合框主体 |
| Textarea | `packages/ui/textarea.tsx` | |
| Tooltip | `packages/ui/tooltip.tsx` | |
| Resizable | `packages/ui/resizable.tsx` | `ResizablePanelGroup` / `ResizablePanel` / `ResizableHandle` |
| Toast | `packages/ui/toast.tsx` | 展示性；store 拥有队列 |
| Error Boundary | `components/ErrorBoundary.tsx` → `ErrorBoundary` | 按区域的崩溃隔离 |

建立在这些基础组件之上的应用级对话框/浮层实例：

| 规范名称 | 实现 | 基于 | 职责 |
|---|---|---|---|
| New Workspace Dialog | `panels/NewWorkspaceDialog.tsx` → `NewWorkspaceDialog` | Dialog | 开始工作的表面（模式分叉：隔离 worktree / 项目文件夹） |
| Confirm Dialog | `panels/ConfirmDialog.tsx` → `ConfirmDialog` | Dialog | 没有稳定锚点的模态是/否（初始化仓库、关闭项目、移除工作区） |
| Notice Dialog | `panels/NoticeDialog.tsx` → `NoticeDialog` | Dialog | 用于失败的单按钮信息模态 |
| Confirm Popover | `panels/ConfirmPopover.tsx` → `ConfirmPopover` | Popover | 从专用动作控件触发锚定的是/否（模板删除） |
| Template Editor Dialog | `chat/TemplateEditorDialog.tsx` → `TemplateEditorDialog` | Dialog | 创建/编辑提示词模板 |
| Skills Dialog | `chat/SkillsDialog.tsx` → `SkillsDialog` | Dialog | Skills 管理器（聊天 + 项目模式） |
| Login Dialog | `auth/` → `LoginDialog` | Dialog | Provider OAuth / API-key 登录 |

**⚠ 命名说明（primitives）：**

- **Modal** = **Dialog** 基础组件。没有单独的 `Modal` 组件；「Modal」是通用术语，「Dialog」是实现。
- **Context Menu** —— 两种形状共存。**Context Menu** 基础组件（`packages/ui/context-menu.tsx`，
  Radix）支撑 Project Row 的右键菜单；更旧的右键表面（Change-Row Actions 菜单）仍是 **Dropdown
  Menu** 基础组件加一个共享的右键处理器 —— 称后者为「Row Actions Menu」。两者通过
  `packages/ui/menu-styles.ts` 呈现相同外观。
- **Drawer** —— **没有** drawer 基础组件或组件。移动端单视图 shell 已设计但未实现；不要对任何
  现存区域使用「Drawer」。**Resources Inspector** 是一个就地的非模态 **Dialog**（`DialogPanel`），
  不是 drawer。
- **Toolbar** —— 没有 `Toolbar` 组件；每个面板的细控件行（Changes Header、Diff Pane 头部、
  Review Toolbar、视图切换）都是内联的。描述性地使用 **Panel Header** / **Panel Toolbar**，而
  不是把它们当作组件名。

---

# 术语表 —— 规范名称（Glossary — Canonical Names）

在设计讨论中使用这些术语。存在多个名字时，列出**规范**术语，并在括号中列出其替代说法。

**顶层布局（Top-level layout）**

- **App Shell** —— 根框架（`Shell`）。
- **Top Bar** —— 应用级头部（无组件名；内联 `<header>`）。
- **Workspace Workbench** / **Welcome Layout** —— 两种主体状态。
- **Toaster** —— 应用级 toast 宿主。

**Top Bar**

- **Wordmark** —— ThinkRail 品牌标记。
- **Scope Context**（替代：location context）—— 持久的位置面包屑。
- **Connection Status** —— 已连接/连接中/已断开的胶囊。
- **Settings Button** —— 打开 Settings Dialog。

**项目（Projects）**

- **Projects Tool** —— 可移动的单例；**Projects Rail**（替代：Project Tree；遗留测试 id：Left
  Nav）是它的项目 → 工作区视图。
- **Project Row**、**Workspace Row**、**Default Workspace** —— 轨道行。
- **Add-Project Menu**、**Add-Workspace Button**、**Remove-Workspace Button**。
- **Diff-Stat Badge** —— `+N −M` 徽标。

**欢迎（Welcome）**

- **Welcome Panel**（替代：Welcome Screen）—— 无工作区的表面。
- **Welcome Heading**、**Primary Card (CTA)**、**Action Card**。
- **Provider Warning Banner**、**Project Trust Notice**。

**中央（Center）**

- **Center Workbench**（替代：Center Tabbed Area、Editor Area）。
- **Center Group**、**Group Header**、**Tab Strip**、**Tab**、**Tab Close**、**Split Separator**。
- 标签种类：**File tab**、**Chat tab**、**Diff tab**、**Changes review tab**、**Document tab**、
  **Terminal tab**。
- **Editor Pane**、**Workspace-Ready Receipt**。
- **File Pane**（**Code File** / **Markdown Preview**）、**Diff Pane**（**Source Diff** /
  **Rendered Diff**）、**Changes Review Pane**（**Stacked** / **One file**）、**Document Pane**、
  **Terminal Body**。
- **Chat-History Menu**、**New-Chat Button**。

**聊天（Chat）**

- **Chat View** —— 整个对话表面。
- **Chat Header** —— 它的顶栏。**Session Stats Bar**、**Resources Trigger**、**Skills Button**。
- **Resources** —— **Resources Trigger**、**Resources Dock**、**Resources Inspector**、
  **Resource Row**、**Command Log View**、**Subagent Transcript Pane**、**Stop-All Action**、
  **Still-Running Chip**。
- **Message List**（替代：Transcript）。单位：**Turn**（pi 消息）、**Row**（派生渲染单位）。
- 行渲染器：**User Message**、**Assistant Markdown**、**System Notice**、**Error Turn**、
  **Retry Indicator**、**Tool Card**、**Activity Group**、**Turn Divider**（带 **Artifact Chip** /
  **Artifact List** / **Review-Turn Chip**）、**Stream Indicator**。
- **Tool Card**（框架）/ **Tool Renderer**（主体）：**Bash Card**、**Read Card**、**Write Card**、
  **Edit Card**、**Ask-User-Question Card**、**Visualization Card**、**Web Card**、
  **Default Tool Renderer**。
- **Composer** —— 提示词输入。**Model Selector**、**Thinking Selector**、**Slash-Command Menu**、
  **Send Button**、**History-Open Button**、**Slot Hint Chip**。
- **History Overlay** —— **Results List**、**Preview Pane**、**Scope Picker**、**Jump Action**、
  **Save-as-Template Action**。
- **Chat Plan** —— **TODO List**（右轨工具，`PlanRailPane`）+ **Plan Page**（中央标签，
  `PlanPane`，通过 `openPlan` 打开）。

**变更评审（Changes review）**

- **Changes Review Tab**（主体：**Changes Review Pane**）—— 每个工作区 + 范围一个；模式
  **Stacked** / **One file**。
- **Review Toolbar**、**File Section**（**Section Header**、**Viewed**、**Collapsed Notice**）、
  **Hunk Toolbar**（**Keep**）。
- **Triage Bar**（**Progress Bar**、**Next Unreviewed**、**Mark All Viewed**）、**Walk Bar**。
- **Review Guide**（**Guide Step**、**Guide Finding**、**Apply Fixes**）、**Large-Scope Notice**、
  **End Note**。

**侧边（Sides）**

- **Left Side**、**Right Side**、**Side Group**、**Hidden-Side Rail**。
- 单例工具：**Projects**、**Specs**、**Files**、**Review**。
- **Specs Panel**、**Files Panel**（替代：File Tree；其 **Changed** 模式渲染 **Changes Panel**）、
  **Review Panel**。
- **Changes Header**（无组件）：**Changes Scope Menu**（含 **Last Turn** / **Agent Turns**）、
  **Branch Picker**、**Changes View Toggle**。
- **Changes List** / **Changes Tree**、**Change-Row Actions**、**Tree Row**、**Viewed Mark**、
  **Diff-Stat Badge**。

**终端（Terminal）**

- **Terminal Placement**、**Terminal Tab**、**Terminal Instance**、**Add-Terminal Button**。
  终端可以位于中央组或侧组；它不是固定的右下区域。

**设置（Settings）**

- **Settings Dialog**，区块为：**Providers**、**GitHub**、**Appearance**、**Layout**、
  **Terminal**、**Templates**、**Privacy**。

**共享基础组件（Shared primitives）**

- **Modal** = **Dialog** 基础组件。**Dropdown Menu**、**Context Menu**、**Popover**、**Command**、
  **Tooltip**、**Toast**、**Resizable**、**Error Boundary**。
- 应用对话框：**New Workspace Dialog**、**Confirm Dialog**、**Notice Dialog**、**Confirm Popover**、
  **Template Editor Dialog**、**Skills Dialog**、**Login Dialog**。（扩展对话框不是模态：见
  **Ext-UI Prompt**。）

**不映射到任何 ThinkRail 区域的术语**
**（Terms that do NOT map to a ThinkRail region (avoid or use only as noted)）**

- **Status Bar** —— 不存在；最接近的是 Connection Status 胶囊与 Session Stats Bar。
- **Context Menu** —— 现在是一个真实的基础组件（Project Row 右键）；Changes 行的右键仍是
  Dropdown Menu（**Row Actions Menu**）。命名表面，而不只是「context menu」。
- **Drawer** —— 不存在（移动端 shell 尚未实现）。
- **Toolbar** —— 没有组件；细控件行是内联的 **Panel Headers**。
- **Bottom Terminal** —— 不存在固定区域；请说 **Terminal Placement** 并指明其所属组。
