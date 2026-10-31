# store —— Zustand 应用状态

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

唯一的 Zustand store：连接状态、项目/工作区、一个前端本地的工作台帧外加按工作区视图/注意、
终端目录，以及为每个活动 `AgentSession` 各一个**按会话的 chat 运行时**（因此多个 chat 可以
并发流式输出）。

## Chat 资源（Chat Resources）

Resources 视图把宿主快照与 Pi 对话运行时分开保存，按 chat 身份与当前连接权威做 key。装入/
失效/清空一个资源快照是一个原子的 store 动作。单调的失效修订版与连接代数栅栏住每一次读取装
入与失败；墓碑与被清空的条目拒绝迟到的应答。连接状态变化会把保留的快照标记为过期，而不受支
持的 welcome 会清空它们。`selectors.ts` 是 Resources selector 与谓词的规范所有者，包括活动
计数与活动/已结束分组（经由 contracts 的 `isLiveResource`，因此被中断的运行绝不算活动）以及
已结束总数：宿主的按种类 `history` 统计，或在更旧宿主上对快照自身已结束记录所做的统计。宿
主把快照限制在最近已结束记录的窗口内；用户翻页取来的更旧内容与 `older` 处在同一个投影上，每
种类紧接该窗口的一条连续运行（`chatResources.ts`）：新快照只在它的窗口仍触及旧链条时才携带
某种类的运行（否则丢弃它，而不是展示一个带静默空洞的列表），新的连接代数会丢弃它（一次重启
可能在其中露出被中断的运行），而 `appendChatResourceHistory` 只在链条仍结束于该页被请求的
位置时才保留该页。`selectChatResourceGroups(snapshot, includeMcp, older)` 交回每种类持有的
全部已结束记录（先窗口后运行，最新的在前）、单独作为 `recentFinished` 的窗口（唯一可能是新
的记录），以及宿主在这些之外的持有量（`remaining`）；`selectChatResourceOldest` 指出下一页
的锚点。组件不各自独立推导这些。`chatResources.ts` 是 store 的私有实现文件，负责资源投影/
状态/作用域/读取类型与 `staleChatResources` 状态变换。父级结算不会清空它仍在运行的资源，也
不改变 Projects 工具栏既有的活动契约。可选的 `mcpServers` 列表是同一个资源快照的一部分；
`selectChatResourceGroups(snapshot, includeMcp)` 为它排序（注意优先），并把它的活动
（connected/starting）与注意（needs sign-in、failed、pending approval）server 与工作计数分开
统计，对更旧的协议交回 `null`。

资源快照不按浏览器持久化。重连或不受支持的宿主会移除控制权威，直到一次全新的读取成功；失败
的读取会保留看起来已过期的数据，绝不凭空造一个空目录。chat 删除/工作区移除会清空对应的投影。
浮层与选中日志状态属于 chat 集成，不属于领域持久化。呈现见
[[submodule-web-chat-resources]]，线上契约见 [[module-contracts]]。

## MCP server（MCP servers）

`mcp.ts` 是 MCP server 状态的私有分片实现：**`mcpByWorkspace`** 按工作区持有最后一次
`mcp.list` 的答案 —— `servers`、文件级的 `configErrors`（来自更旧宿主时为空）与
`handledElsewhere` —— 以及持有的按会话状态快照，旁边是一个单调的 **`mcpRevision`**。两个原
子动作写入它。**`applyMcpStatus(snapshot)`**（`mcp.status` 通道）只在已连接、MCP 可用协议、
作用域存活（被移除的工作区与被删除的 chat 会被拒绝）时装入，且只经由**代数守卫**：当快照
的代数更新，或它到达于更新的连接时，才替换持有的那个 —— 重启的宿主会重置它自己的计数器，因
此纯数字比较会让视图冻结。**`installMcpList(read, result)`**（每次 `mcp.list` 与返回列表的
变更）由请求之前捕获的 `selectMcpRead`token 栅栏：一次列表就是该次读取时权威的活动 chat 集
合，因此它省略的持有会话会被丢弃，除非读取开始之后有推送落地，更新代的持有项会存活，已删除
的 chat 绝不重新装入。每个持有的快照还携带某个 server 在该连接上首次被看到 `starting` 的时
间。不受支持的 welcome 会清空该分片，工作区移除会丢弃它的条目，chat 删除会丢弃它的快照。
**`deriveMcpServerRows(projection, now)`** 是唯一的聚合：每个配置条目一行；没有活动 chat 时
状态来自配置（invalid、pending approval、disabled / disabled in this project，否则 not
running）；否则任一 chat 中的问题按固定优先级胜出，`connected` 统计报告它的 chat 数
（"connected in 2 of 3"），`sessions.reporting` 命名报告该行状态的活 chat（Settings 的按
chat Reconnect 与 Reload 现在指向它们），未被报告的 server 读作 unknown，而 `starting` 超
过一分钟读作 unknown（stalled）。宿主可选的 `summary.effective` 指出被接纳的条目，它接收会
话状态并可提供仅按名字的 login/logout/test 动作；被替换的用户条目读作 `replaced`。对更旧的
宿主，状态投影保留有效的 approved-project/user 回退，但对有歧义的同名行会扣住运行时动作。
被折叠的无效项目覆盖不会吊销有效用户行被显式报告的权威。待定的 repo 覆盖会把它对应的用户行
标记为需要注意。注意行排在最前，disabled/replaced 行排在最后。`selectMcpWorkspaceStarting`
在某个 server 启动期间驱动有界重读。状态词汇与注意/活动谓词放在 `lib`（`mcpState`），因为
仅 props 的 chat resources 模块共享它们。

## 边界（Boundary）

- **拥有：** `appStore.ts` —— 连接/项目/工作区状态 + setter。连接状态有两条语义不同的单调
  边：`setStatus("connected")` 为重连水合推进 **`connectionGeneration`**，而
  **`welcomeGeneration`** 只在一份完整的 `server.welcome` 快照落地时推进。每次连接状态转换都
  会清空 `protocolVersion`，因此 UI 能力在 socket 之间、以及当前 socket 的 welcome 装入之
  前始终不可用。**
  `installWelcomeSnapshot(protocolVersion, projects, recentProjects, config?, hostPlatform?, hostUpdate?)`
  **在一次 Zustand 写入里装入协议 + 两个排好序的项目视图 + 可选 config、宿主平台、可选的宿
  主更新生命周期快照 + 导航修复，然后推进那条就绪边；路由校验与能力读取绝不会观察到半个
  welcome。`installProjectSnapshot` 仍是给聚焦调用方用的仅项目原语。**`projects`** 是打开
  着的工具栏，而**`recentProjects`** 是每个已知打开 + 已关闭项目的按最后打开排序集合；服务
  器会省略文件夹已不存在的已关闭记录。**`applyProjectUpdated(project)`** 是
  `project.updated` 推送与权威项目变更响应的唯一全快照更新器：它 upsert/排序 Recents，并要
  么 upsert/排序工具栏，要么在 `closed === true` 时移除该行。两个动作也会对账过期的导航：
  只有当本客户端选中的项目或活动工作区属于一条不再打开的记录时，它们才清空活动工作区并选中
  第一个剩余项目的 Home（没有剩余时为 `null`），同时刻意保留每一份本地的工作区视图/注意/资
  源渲染/终端/会话映射，以便无损重开。其他客户端的打开绝不抢走导航，后台关闭也绝不移动它。
  所有项目响应的调用点都使用同一个更新器，因此打开与最近的副本不会漂移。显式的本地转换是
  **`selectMain()`**、**`selectProject(projectId, opts?)`** 与
  **`activateWorkspace(workspace)`**；各自原子地更新它耦合的作用域 id，且不存在能拆开该不变
  量的通用活动工作区 setter。**`workspaceSelectionHistory: string[]`**是这个页面的最近优先
  工作区注意：每一次普通的、路由驱动的与历史搜索的激活都会把目的地移到最前，而项目/main 的
  选择保持这份新近性不变。它不是宿主状态、URL 历史或重载持久化。
  **`expandedProjectIds: Record<string, true>`** 是 Projects 工具栏的按浏览器展开状态 —— 由
  store 持有（它必须挺过工具栏的重挂载，并且能被非工具栏手势写入），配
  **`toggleProjectExpanded(projectId)`**（箭头）、**`expandProject(projectId)`**（幂等揭示：
  工作区创建 / worktree 附加 / 活动工作区可见性规则），以及 `selectProject` 的
  **`{ reveal: true }`** 选项 —— 用户手势变体，一次写入即进入项目 home 并展开其行（工具栏行
  点击、Welcome 屏的打开采纳）；导航恢复与工作区移除的兜底裸调用它，保持展开中性。项目快照
  装入会把展开裁剪到打开工具栏（已关闭项目的条目被丢弃；未变化时身份稳定）。
  **`hydrateExpandedProjects(projectIds)`** 在启动时从 `panels/projectExpansion` 持久化模块
  为集合播种—— store 自身仍不触碰任何存储：该模块拥有 localStorage 镜像（按宿主限定的
  key、尽力而为的写入、不受信的读取）并订阅变化。**已结算搁架。** `selectors.ts` 拥有
  **`selectWorkspacePartition(state, projectId, now)`** —— 把一个项目的行推导为 `live` 与
  `settled` 的唯一来源（每条已结算行携带它的原因：`override`、`review merged|closed`，或
  `idle since`），由工具栏、topbar 的 *Switch to* 菜单与搁架共同消费，因此三者绝不会不一致；
  **`selectAutoSettledCounts(state, now)`** 复用它的逐行规则，为每个已加载项目只统计搁架
  自行接收的行（绝不算 `override`），这正是工具栏一次性首次移动提示所播报的内容。低于
  `WORKSPACE_SETTLE_PROTOCOL_VERSION` 时它把宿主的原始列表作为 `live` 返回、不排序，并忽略
  连过期的持久化 settle 字段；激活使用同一道门禁，保留旧宿主的顺序与行为。
  `workspaceSettlingSupported` 记录最后一次完成的 welcome 的判决，而不是在短暂断连期间清空，
  因此重连不会重挂载工具栏并丢弃就地重命名；稍后到达的更旧 welcome 会权威地关掉它。在支
  持的宿主上，规则依次是：有 agent 正在工作或需要注意的行，或 Default 工作区，是 live；
  `settledOverride: "settled"` 结算，`"active"` 保持 live（用户的话压过下面的闩锁 ——结算你
  正在其中的那一行必须移动它）；**`activeWorkspaceLiveLatch`** 保持时的活动工作区是 live；
  `review.state` 为 `merged`/`closed` 且有有效的 `changedAt` 且没有更新的 `lastActiveAt`
  则结算（不完整的已结算快照保持 live 且可刷新）；打开的 review 保持 live；否则空闲时间超
  过 **`settleIdleDays`**（镜像自 `AppConfig`，`null` = 从不）则结算。调用方选择 **`order`
  **（`WorkspaceOrder`）：`"created"`（默认）反转宿主的列表 —— 它的注册插入顺序，最早创建
  的在前 —— 因此两个列表都是最新创建在前：工具栏的顺序，新工作区落在顶部，而且你工作时行绝
  不跳动；`"recent"` 按最新 `lastActiveAt` 排序（最新创建者打破平局）供 topbar 的切换器；
  无论哪种，Default 行都钉在最前。可选排序被尝试过并否决了：自动重排工具栏会在指针下移动行，
  而切换器想要新近性但不需要选择器。闩锁是 "选择永不移动行" 规则的状态化：
  `workspaceActivationPatch` 把它设为目的地当时是否 live（因此你打开的一个已结算工作区会留
  在搁架上，被强制揭示），并且**在目的地已经是活动工作区时原样保留它**（一次重击、*Open in
  → terminal*，或 Back/Forward 回到同一行都不是新的选择，绝不能重新判定一行其窗口在你身处
  其中时已经过去），而推进活动行 `lastActiveAt` 的 `workspace.updated` 推送会重新武装它（打
  开工作区里的真实工作让它保持 live）。**`settledShelfExpanded`** / **`settledShelfShown`
  **（按项目、仅此浏览器、不持久化 —— 每次加载默认都是折叠的搁架）持有搁架的展开与分页。分
  区把 `now` 作为参数，因此调用方决定 tick，selector 保持纯。经过校验的路由恢复使用
  **`activateWorkspaceFromRoute(workspace, sessionId?)`**：它应用同样的作用域 id，推进兼容的
  工作区导航 tick 与当前目的组的时钟，并要么装上一个同时盖上两个时钟的瞬态
  **`routeChatTarget`**，要么清除一个更旧的精确目标。仅工作区的路由不携带中心标签意图，因此
  它保留既有的浏览器本地注意；普通的位置推导随后可能把它规范化到已选中的 chat。
  **`routeChatTargetGeneration`** 只在目标装入时推进（包括同工作区深链），因此消费无法复制
  shell 的对账遍。`selectCurrentRouteChatTarget` 是唯一检查目标工作区与导航戳记是否仍然成
  立的判定。shell 的 chat 对账模块在普通的本地已放置 chat 水合之前校验/水合那个精确目标；
  成功的权威缺失会消费它，而失败则为重连保留它。store 不拥有任何 URL/历史/存储访问 ——
  `navigation` 拥有序列化与驱动器。它还拥有每个客户端在 `workspace.created`/`updated`/
  `removed` 推送上都一致执行的**工作区生命周期反应**（没有按客户端的乐观更新 —— 后端是权威）：
  **`addWorkspace(ws)`** 按 `id` upsert 一份 `workspace.created` 快照（项目尚未列出时
  是 no-op —— 在它下一次 `workspace.list` 时对账，而不是播种一个只有一行的局部列表；否则不
  存在则添加 / 存在则合并，与创建客户端自己在创建后重列的幂等）；**`updateWorkspace(ws)`**
  折叠一份 `workspace.updated` 快照：在 `workspaces[ws.projectId]` 中按 `id` **替换**该记
  录，只沿用列表算出的 `diffStats` 聚合（该快照是持久化记录，本身没有它）。推送是权威的，
  因此是*替换* —— 绝不是合并：合并无法清掉**宿主丢掉的某个可选字段**（`diffBase` 重新指向
  创建基线、最后一条 `skillOverrides` 被移除，或 `subagentsOverride` 回到继承），从而让客户
  端继续按宿主已不再持有的值贴标签并做读取键；一个从未取回的项目，或某个 id 不在它的列表里，
  都是 **no-op** —— 下一次 `workspace.list` 会对账；
  **`applyWorkspaceRemoved(projectId, id)`** 是**整个**移除反应（`removeWorkspace` 丢掉该行
  +`clearWorkspaceState` 丢掉它的本地视图/注意/终端映射与 chat 运行时 + 最近记录丢掉这个已
  死的 id，且**如果它正是本客户端的活动工作区** → 激活最近选中的、项目仍然打开的已加载工作
  区，即使跨项目；当没有剩余任何工作区时，`selectProject(projectId)` 回退到被移除工作区的
  Project Home；两条活动兜底都会得到同一个中性 toast，对发起者与观察者都读得通）。后台移除
  绝不移动焦点。因为生命周期事件是共享的而选择历史是浏览器本地的，每个观察客户端恢复的是它
  自己先前的上下文。原语 **`removeWorkspace(projectId, id)`**只丢掉该行（未知的项目/id 是
  no-op）；**前端本地的工作台状态** —— 一个 `workbenchFrame` 携带这个前端表面的无资源拓扑、
  几何、单例工具与恢复目标；`workspaceViewsByWorkspace` 携带每个工作区的资源放置、顺序与
  预览身份；`layoutAttentionByWorkspace` 携带它的选择、最后焦点与导航时钟；本地预置/上限偏
  好位于帧旁边。这些是来自 `shell/layout` 的 web 本地类型，不是线上快照。
  `layoutDocumentsByWorkspace` 是一个身份稳定的**派生投影缓存**，只为既有 selector/effect
  从帧 + 视图重建；它既不被持久化，也不能作为权威被写入。这里没有接受/投影的成对、修订版、
  变更 id、待写入、回滚队列或冲突状态。

`applyLocalLayoutTransition(result)` 是纯布局结果的唯一原子装入边界。仅资源的结果更新一个
工作区视图与注意。帧结果在替换那唯一帧的同时替换每一份保留的工作区视图重映射，因此显式的组
移除或预置应用绝不可能让某个隐藏工作区继续引用一个已死的组。组件从不拼接组/标签页数组。空
组是合法的帧状态，关闭最后一个资源不会删除它们。只有当本地转换使某个不受控的指针/尺寸草稿
失效时，`layoutProjectionEpoch` 才推进。

store 拥有值与动作，绝不拥有持久化。`shell/layoutState` 校验并水合一份按浏览器 endpoint/表
面或原生 profile/window 限定的本地文档，订阅相关的状态边，并持久化归一化的帧/视图/注意/偏
好。`hydrateLocalLayoutState` 装入该文档一次；缺失或无效的文档表示为一个没有工作区视图的
Balanced 帧。工作区消失时 `clearWorkspaceTabs` 移除该工作区视图、注意与相关本地状态。页面
寿命的 `removedWorkspaceIds` 墓碑会拒绝过期的目录/会话/缓存/工作区到达，因此在途读取无法重
建它。

**浏览器本地的资源渲染状态**按工作区 + 规范资源 id 做 key，绝不嵌入帧。已加载的文件/diff
内容与元数据、渲染器选择、不透明的渲染器视图状态、diff 布局、活动 chat 运行时与已解析的文
档 markdown，仍是各自领域来源之上的缓存。放置 id 在一个工作区视图内是稳定的；已经被另一个
语义缓存占用的 id 会得到一个防碰撞的缓存 id。只有当它的本地资源引用命名了一个已注册
resolver 加上持久化来源身份时，虚拟文档才合法；`todo-plan` 按会话解析到活动的 `PlanPane`。
任意内联 markdown 不能进入持久化布局状态。水合之前的空缓存绝不等于缺失：领域读取必须对当前
连接代数是权威的，之后对账才可以裁剪某个本地引用。

**工作区本地的注意**是每个稳定帧组选中的标签页、最后聚焦的中心组、最后聚焦的辅助组、组导航
时钟，以及每个辅助组可选的最后终端召回。召回是在同一所有者内的经过校验的历史，不是另一个活
动选择，且绝不跨工作区复制。选择/焦点绝不修改帧。但一个组的*工具*选择是共享的：单例工具是
帧放置，因此显示某个工具的组在这个窗口的每个工作区里都显示同一个工具。选中一个工具（点击、
揭示，或工具移动的焦点）会被写入每一份保留的注意 —— 只要它在该组的选择是工具或为空 —— 而一
个新物化的视图继承最近活动工作区的工具选择。选中的工作区*资源*（混合辅助组里的终端）保持本
地，绝不被覆盖。`shell/layoutState` 在与本地写入同一个事务里拥有这个扇出。帧替换会把每个工
作区的注意对账到存活的组/标签。导航时钟在本地改变焦点的打开与显式重选时于请求时推进；戳记
随意图一起走，因此接受时不会计两次。当更新的导航超过一次慢完成时，该完成被丢弃。预览身份是
每个中心组的工作区本地：`preview` 替换该组的槽位，`keep` 单向提升。一次合并的预览→keep 手
势把 `claimPreview` 带在它唯一的最终意图上，因此没有中间状态被持久化；在已落地预览之后到来
的 keep 就地提升那处既有放置。与排布无关的打开进入 store，但只有 shell 布局集成才会把它们
对账到该工作区的最后焦点，并提交本地放置。重新打开一个已有的规范资源只改变注意，除非非身份
元数据变了。迟到的 chat 标题事件可以修复缓存/历史标签或一个仍排队的打开，但不能重建本地已
关闭的放置或抢走焦点。**持久标题的 fold** 让 `session_info_changed` 走扩展 UI `setTitle` 已
使用的那同一个单点 chat 标签变更：把缺失的名字规范化为空的未命名名（在渲染时由 `lib` 的
`chatDisplayName` 定措辞，正如 `selectChatTitle` 所做；`selectChatName` 返回存储的名字，而
`workspaceName(t, ws)` 为宿主命名的默认工作区定措辞）；更新缓存的 `ChatTab` 与同一会话的
`PlanTab`（各自带它既有的本地放置与任何排队的打开，按资源身份匹配），和/或它的
`closedChatsByWorkspace` 行；并且不改变任何新近性、运行时、放置、导航时钟或注意。持久的 Pi
事件可以取代扩展仅用于呈现的标题。对这个表面从未列出过的会话，事件仍是 no-op ——
`session.list`/`session.getMessages` 提供重连与后续水合的真相，因此不存在标题专用的事件缓
冲。`syncLegacySelection` 把选中的资源镜像进临时的编辑器/终端兼容状态，而不成为放置权威；
它的 selector 包含匹配的缓存/目录 key，因此身份修复会重新触发镜像。

**`terminalsByWorkspace` 仍是终端领域状态的镜像，绝不是放置权威。** 宿主拥有终端的存在性，
按 `(workspaceId, tabKey)` 做 key；工作区视图在本地引用那个 key。`setWorkspaceTerminals` 采
纳 `terminal.list` / `terminal.tabs`，只在预留仍在途中时保留一个被省略的本地标签页，移除已
死的引用，并把尚未被表示的目录标签页提供给本地放置对账，而不改变帧几何或注意。
`addTerminal` 铸造一个持久 key 并发出一次本地放置意图；被捕获的组目的地保留上下文创建，而
未捕获的请求解析到底部。默认终端预留是宿主拥有的工作区创建握手，表面接收由此产生的目录条目。
已确认的关闭移除领域成员关系与每一处本地引用；被拒绝则保留放置。不存在工作区全局的
`activeTerminal`：工作区注意决定哪些主体挂载，而宿主的独占附加/接管决定哪个客户端控制某个
PTY。**按会话的 chat 状态** —— `sessions: Record<sessionId, SessionRuntime>`，其中一个
`SessionRuntime` 持有一个 chat 的 `turns`（pi 规范）/ `toolResults` / `toolResponses`（交
互式工具的迟到响应 —— `tool-response` 自定义消息，以及旧转录里的 `ask-user-answers` —— 按
工具调用 id 做 key、值为不透明，由 reducer 与水合建立索引，绝不变成气泡；兄弟的
`subagent-completion` 自定义消息反而被**追加为一个 `subagentCompletion` 轮次** —— 分离式
subagent 的终态报告是按转录定位的，渲染为 `chat` 的后台结果行；它、
`backgroundCommandCompletion` 与 `reviewFix` 轮次都保留消息的 `timestamp`，以便 chat 能在
那里打开一轮 —— 两者都由共享 contracts 守卫收窄）/ `currentAssistantId` /
`attemptAssistantId`（把溢出移除限定到实际观察到的尝试）/ `isStreaming` /
**`settlementTick`**（浏览器本地，每次 `agent_settled` 都单调递增，因此批处理的 start+settle
仍向 chat 布局暴露完成边）/`model` / `thinkingLevel` / **`eventRevision`**（浏览器本地，每
收到一个 Pi 事件就递增；对权威转录读取的比较并装入栅栏）/ **`syncedConnectionGeneration`**
（该运行时的转录最后读取自哪个已连接宿主代数）/`stats` / **`statsRefreshTick`**（浏览器本
地的失效信号，用于已挂载 chat 的权威统计读取）/ `commands` /`draft` 及其**扩展 UI 状态**
（`pendingExtUi`（由 `chat` 的 `ExtUiDialogRequest` 定型）+ `extUiQueue`（重叠对话框的
FIFO，使任何一个都不会让它对应的 server promise 成为孤儿）+ `extUiStatus` /`extUiWidget`）。
`openChatSession` 创建一个运行时；`closeChatRuntime` / `clearWorkspaceState` 丢弃它；按
会话的变更器（`appendUserMessage` / **`appendErrorTurn`** / `setStats` / `setCommands` /
`setCurrentModel` / `setThinkingLevel` / `setChatDraft` / `clearPendingExtUi`）接收一个
`sessionId`。**`appendErrorTurn(sessionId, text)`** 为一次**被拒绝**的驱动轮次的线上调用
（`session.prompt`/`steer`/`followUp`/`create`）追加一个 `error` 轮次 —— 例如 `prompt()` 抛
出"no API key" / 某模型不可用 —— 因此失败的发送落进 chat 而不是被吞掉；它不携带恢复动作，
因为 Pi 从未接受那个缺失的轮次。它不清空 `isStreaming` 或活动的 assistant id：一次被拒绝的
steer/follow-up 无法结算宿主仍在运行的工作。*流式*故障反而在 `agent_settled` 处通过
**`reduceSessionEvent`** 结束运行，使用宿主投影的最终终态元数据：`stopReason: "error"` 携带
Pi 的 `errorMessage`，而 `stopReason: "length"` 变成可操作的截断错误 —— 两者都不能变成
"✓ Done"。收口者由 `chat` 的 `insertRoundCloser` 放置（在那些没有唤醒任何人的尾部结果之前，
与水合使用的规则相同）。只有那个由结算产生的错误轮次携带 web 本地的
`recovery: "try-again"` 标记；水合只把它给自己合成出来的当前最终失败。`appendUserMessage`
在添加后续用户轮次的同一次乐观写入里消费每一个先前的标记，而 `agent_start` 为另一个客户端
或自定义消息开始的工作消费它。因此渲染器可以提供一次普通的 `Try again.` 发送，而无需解析错
误，也不会留下一个可操作的历史失败。`agent_end` 是尝试级的，绝不清空 `isStreaming`；只有结
算才结束重试、压缩与排队的续跑。同一个 `agent_settled` fold 独立于最终的 `isStreaming` 值
递增 `settlementTick`，当 Pi 事件批处理器只给 React 该批次的 false→false 端点时仍保留那次
转换。**压缩的生命周期是一个一等轮次**：`compaction_start` 追加一个 `compaction` 轮次
（`running`），`compaction_end` 就地结算尾部正在运行的那个（成功 → `done` + 来自已定型的
`CompactionEndResult` 的 tokens-before/after，并过守卫—— 线上数据不可信；`aborted` →
`cancelled`；`errorMessage` → `failed` 并携带该消息，例如 pi 的一次性溢出恢复上限 —— 失败
的压缩必须可见，绝不能被吞掉），或在没有正在运行的轮次时追加那条已结算轮次（压缩中途重连）。
一次 `willRetry: true` 的成功 `compaction_end` 还会把该轮次标记为 `resuming`（pi 继续同
一次运行；结算会清除该标志 —— 已结算的转录绝不声称工作仍在进行），并且仍然移除被取代的
assistant 尝试。一次成功且没有持久 `summary` 的实时轮次，也是 chat 集成去读取 Pi 规范压缩
后转录的信号；reducer 自己绝不猜测切分边界。reducer 依赖 pi 的保证：每个发出的
`compaction_start` 都配有一个 `compaction_end`（成功与失败路径都发出它），这与其他每一对事
件得到的信任相同。手工命令的传输层拒绝使用一次原子的 store 变更：给定请求之前观察到的压缩
轮次 id，只有当当前 id 中没有新的时，才追加一个失败的压缩轮次。因此一个已由
`compaction_end` 表示的 Pi 失败绝不被重复，而在生命周期开始之前的拒绝仍会在同一个压缩表面
上可见。**`auto_retry_start` 镜像 pi 对活动上下文的外科手术**：pi 的 `_prepareRetry` 在重
跑该轮次之前从活动上下文中裁掉失败尝试的 assistant 消息（重试会把它作为一条新消息重新流式
输出），同时*把它留在会话文件里*，因此 reducer 丢弃被取代的失败 assistant 轮次
（`removeSupersededAssistant`，与溢出压缩路径相同的规则）—— 否则客户端会把这条回复渲染两次
（冻结的失败分部 + 重试的副本）。水合把同样的呈现规则应用到持久化副本（`chat/hydrate.ts` 隐
藏被重试的尝试 —— 一次出错的 assistant 后面跟着另一条 assistant 消息），因此实时与重载后的
客户端一致。
  - **归一化的会话状态**（`sessionStateByWorkspace`）是每个工作区/会话由宿主撰写的完整状态
    记录。快照水合只在一次完整的当前代数读取时替换该映射；有序推送折叠一条记录，删除则移除
    它。共享 selector 推导 needs-input、working（contracts 的 `isSessionWorking`，因此一个
    在后台命令或 subagent 上空闲的 chat 仍读作正在运行）、unread-finished，以及项目/工作区
    汇总 —— 不存储优先级，也没有第二份运行中分片。记录的 `firstPrompt` 是
    **`selectChatFirstPrompt`** 的首选来源（其次是 `ClosedChat` 行，然后是已加载运行时的用户
    轮次，经 contracts 的 `firstPromptOfTexts` —— 与宿主所用的同一读法，因此一次乐观的
    `/skill:` 发送或前导的包会给出宿主将报告的那个开头；运行时只在宿主尚未报告之前作数，因
    为已水合的运行时可能起始于一次压缩之后）；`selectChatTitle` 把它经 `chatDisplayName`
    折叠，`selectUnnamedChatFirstPrompts` 按会话 id 为工作区未命名的 chat *与计划*标签页收
    集它，而 `selectSwitcherGroups` 把同一个 selector 读进 `SwitcherSession.firstPrompt`，
    与真实的 `chatTitle` 并列（它的钩子只在面板打开期间订阅 `sessions`）。
    `SessionRuntime.hostState` 随转录水合装入，也由 Pi 事件之后投递的状态推送装入。一个结
    果是**被看到时即被读取**，而不是靠手势：`selectReadyCompletionAck` 只在连接是当前的、
    运行时已同步到这个代数、它的 `hostState` 命名同一个完成、精确的结果行已渲染，且
    `selectChatAttended` 成立时返回那个精确的未读完成 id —— 该 chat 是活动工作区中最后聚焦
    中心组的选中标签页，且没有历史浮层遮蔽它或为它而被请求（一个待定的
    `historyOpenRequest` 绝不能让聚焦跳转在浮层挂载之前就确认）。`ChatTranscript` 补上
    store 无从知道的事实（窗口有焦点且可见；没有 Resources 检查器或模态层覆盖转录；它跟随
    最新边），并在全部成立时确认一次。该条件是连续的，因此不需要记录激活、不需要待定的冷连
    接手势或 tick 顺序：用户正在跟随期间到达的结果会立即清除，回滚到最新边会稍后清除，而未
    聚焦的窗口会保持它点亮（并通知）直到用户回来。宿主对精确 id 做比较并设置，因此过期的视
    图绝不可能清掉更新的结果。唯一的例外是本窗口**手工标记为未读**的结果
    （`workspace.markUnread`）：`beginMarkWorkspaceUnread` 快照该工作区可重新点亮的
    `candidates`（会话 → 完成 id）以及其中哪些被关注；在应答落地之前（宿主的状态推送可能先
    到），只有那些精确的候选完成被抑制，因此真正新的结果绝不被吞掉。每个工作区同一时间只有
    一个标记在途：`beginMarkWorkspaceUnread` 在一个待定时返回 false，调用方就不再发第二个
    请求，否则它的应答可能过早清空关注快照。`finishMarkWorkspaceUnread` 记录
    `markedUnreadBySession[sessionId] = { completionId, held }`，其中 `held` 表示被重新点
    亮的 chat 在标记时被关注，**并且在整个请求期间一直保持被关注**。被持有的精确完成永不可
    确认；一旦该 chat 不再被关注，一个 store 订阅就同时清掉待定关注与 `held`，因此它在下一
    次访问时清除 —— "一直点亮直到你回来"。`selectWorkspaceReadAction` 从快照推导菜单项（有
    任何结果未读时 `markRead`，否则当 contracts 的 `canMarkCompletionUnread` 允许时
    `markUnread`，否则隐藏），而 `selectCompletionMarkedUnread` 是共享的自标记判定。已关闭
    的 chat 可重新打开：工作台关闭命令原子地移除本地放置并调用 **`closeChatToHistory`**，
    它**保持运行时 + 宿主会话存活**，把它记进 **`closedChatsByWorkspace`**（`ClosedChat[]`，
    按工作区，最近优先），并清空待定的跳转/历史打开请求 —— 但绝不清空更新的
    `routeChatTarget`，它的生命周期属于导航取代与对账。文件、diff 与已注册文档的渲染缓存遵
    循同样的本地转换；排队的打开结算之后，shell 只回收不在本地放置与排队意图中的缓存。任何
    对等端都无法远程恢复放置。**`reopenChat(workspaceId, …)`** 即使在另一个工作区成为活动
    之后，也会把运行时/历史成员关系恢复到它被捕获的工作区；shell 通过那唯一一条转换路径把
    放置加入该工作区本地选中的中心组。**`noteClosedChats`** 也把仅磁盘的会话（来自
    `session.list`）与对等端创建的会话（来自 `session.created` 摘要）记在那里 —— 幂等，跳
    过活动/打开的会话，同时用更新的摘要刷新既有的历史行 —— 因此一个挺过了宿主重启或在另一
    个前端创建的 chat 可以重新打开，而不会获得本地放置。一个 `ClosedChat` 在它的 `title` 旁
    保留摘要的 `firstPrompt`，而关闭转换会复制打开 chat 的（`selectChatFirstPrompt`）；历
    史行仍通过 `selectChatFirstPrompt` 读取，其宿主记录压过那份副本。
    **`deleteChat(workspaceId, sessionId)`** 是已确认本地 `session.delete` 与
    `session.deleted` 广播的幂等 fold：它原子地丢掉该 chat 拥有的每个标签页 —— 它的转录、
    活动的计划页，以及任何依赖它的遗留文档缓存 —— 外加它的历史行/运行时 + skill 基线，记录
    一个页面寿命的墓碑，移除该 chat 或其依赖文档的排队打开，并排队一次资源移除意图。shell
    布局集成通过它的纯变更路径移除每一处匹配的 chat 放置与由会话支撑的计划引用，然后在同一
    次转换中对账本地注意。在那之前墓碑不渲染任何主体，因此过期的本地引用无法重建或水合该会
    话。**`noteClosedChats`** 与 **`hydrateSession`** 拒绝有墓碑的会话 id，因此已经在途的
    过期 `session.list` / `session.getMessages` 结果无法重建一个已删除的 chat；墓碑能挺过工
    作区拆除，因为一个更旧的读取仍可能在其后结算。活动工作区的水合遍在每次 `session.list`
    之前快照**`selectWorkspaceSessionIds`**；当那次权威读取落地时，
    **`reconcileWorkspaceSessions`** 对宿主结果中缺失的每一个基线 id 应用同样的墓碑 fold，修
    复断连期间漏掉的删除事件，而不删除读取开始之后创建的会话，也不推进用户导航时钟。否则
    **`hydrateSession`** 在连接时从宿主 `SessionSummary` + 转换后的转录重建浏览器本地的运
    行时/渲染状态；放置只来自这个表面的本地工作区视图、一次本地创建意图或一次显式重开 ——对
    等端创建/活动会话仍在历史中可被发现而不会打开标签页 —— 活动摘要的 `lastSettlement` 存
    在时是权威的；否则只有持久化转录最后一条对话消息上的失败才是当前的（历史性的 `length`
    尝试后面跟着后续工作，绝不能变成过期警告）。如果运行时已存在，水合是 no-op，因此活动/
    领先的 chat 绝不被覆盖。
    **`reconcileSession(summary, hydrated, expectedEventRevision, connectionGeneration)`**
    是成功压缩或重连之后，面向既有运行时的独立权威路径。它只在期望的 Pi 事件修订版处比较并
    装入，拒绝被移除的工作区/会话或跨工作区身份，替换轮次/工具结果/ask 回答/队列/模型/思考
    + 流式状态，并保留草稿/统计/statsRefreshTick/命令/扩展 UI/放置/历史/焦点。它标记已连接
    的代数并推进修订版，因此两次读取无法互相回退。当最新的实时压缩匹配持久记录时，它的 id
    + estimated-after 计数会存活，而 `resuming` 只在返回的摘要仍在流式时存活。纯函数
    **`reduceSessionEvent`** 把一个 `PiEvent` 折叠进某个运行时。每个 `message_end`、
    `compaction_end`、`agent_settled`，以及带花费或上下文条目的 `entry_appended`（`usage`、
    `compaction`、`branch_summary`、`context_edit`）也都会推进 `statsRefreshTick`，即使
    该事件不改变任何已渲染轮次，因此定稿的 Pi 用量/上下文边界不会在呈现过滤或批处理的
    `isStreaming` false→false 端点之后丢失。**`handlePiEvents` 在一次原子 store 写入里折叠
    一个有序批次，同时为每个受影响运行时按事件各递增一次修订版，即使是 UI 忽略的事件**，因
    为被忽略仍意味着它越过了快照定序边界。单事件的 **`handlePiEvent`** 委托给同一路径，因
    此测试与非线上调用方无法漂移。传输层在投递任何后续响应或非 Pi 推送之前，先冲刷待定批次，
    从而保持修订版栅栏的已接收消息顺序。**只有空闲发送才乐观进入转录**
    （`ChatView.onSubmit` → `appendUserMessage`）；下面的末轮回显去重之所以足够，正是因为回
    显之前没有别的东西介入。**流式发送（`steer`/`followUp`）绝不追加轮次**：它的文本存在
    `queue` 里（逐字折叠自宿主投影的 `queue_update`，水合时从摘要播种），旁边是可选保守的
    `hasImages` 聚合，用于守卫破坏性的纯文本队列恢复；轮次只通过 pi 规范的用户
    `message_start` 落地 ——在它真正的位置，实时与已水合达成一致。（镜像 pi 自己的交互模式；
    取代了"一切乐观追加"模型，后者的末轮去重只要在追加与回显之间有 assistant 内容落地就
    会失手 —— 已在实时中复现为重复且错位的排队气泡。）对空闲回显而言：相等的 Pi
    `message_start` 回显会被忽略，而 Pi 规范展开的 `<skill>` 回显会**替换**紧邻其前那条匹
    配的原始 `/skill:<name> …` 轮次（同一轮次 id），因此实时与已水合转录都只包含一次规范的
    skill 调用；畸形或不匹配的块正常追加。**`handlePiEvent(event, sessionId)`** 与
    **`applyExtUi(request)`** 通过 `withRuntime` 辅助函数按 id 路由（对未知会话是 no-op）。
    `error` 级别的 `notify` 变成一个 **`error`**轮次（与会话错误得到的同一条横幅 —— 扩展崩
    溃绝不能读作闲聊）；其他任何级别都保持为 `system` 轮次。级别是*唯一*的输入，因此宿主自
    己的错误通知走同一条路：`host/handlers` 在一次回滚的发送之后发布的
    `Review send failed: …` 同样渲染为那条横幅。这是有意的 —— 一次静默回滚的发送正是用户绝
    不能滚过去的东西。一个**发射即忘**的 ext-UI 帧（`notify`/`setStatus`/`setWidget`/
    `setTitle`）如果**没有可落地的运行时**，会被扣在 **`extUiOrphans`** 里（有界，最新 64
    条）而不是被丢弃，而**两个**安装运行时的动作 ——`openChatSession` *与* `hydrateSession`
    —— 在该会话运行时一存在时，就通过共享的 `replayExtUiOrphans` 排空它的帧。准入测试是 **
    “没有运行时”**，不是“未知会话”：除 `setTitle` 之外每种帧都由 `withRuntime` 归约，因此一
    个仅有标签页或 `closedChats` 条目的会话会通过“我们知道它吗”的测试，随后被
    `withRuntime` 丢弃—— 既不应用也不缓冲。**今天没有任何路径会构造那种状态** ——
    `closeChatToHistory` 保留运行时，`closeChatRuntime` 没有生产调用方，而两处放置 chat 标
    签页的 `openTab` 站点都以其运行时已存在为守卫 ——因此这条规则是守卫而非在修的线上问题，
    并且它通过直接给 store 播种（而不是走一条用户无法执行的动作用序列）来钉住。一旦有任何
    东西在转录水合之前放置 chat 标签页，它就开始发挥作用。`setTitle` 是唯一例外，它自己作
    判断：`renameChat` 重命名标签页或 `closedChats` 条目，并报告它**到底**有没有找到该会话，
    因此只有两个映射里都不存在的会话的标题才会被缓冲。准入从重命名中顺带得出，而不是在旁
    边另算一遍 —— 一次遍历 `tabsByWorkspace`/`closedChatsByWorkspace`，而不是把同两个谓词
    跑两遍。**对话框**（`select`/`confirm`/`input`/`editor`）绝不缓冲：它是一次往返回路的
    一半，另一半可能已被另一个客户端回答，或因宿主重启而丢失，因此重放它会让 chat 卡在一个
    没人在等的问题上。为什么要缓冲：宿主在回答安装运行时的 RPC *之前*就发布了扩展在
    `session_start` 里做的事，因此未缓冲的客户端会静默丢掉扩展在启动时设置的每一个状态、标
    题、widget 与 notify。这**不只是创建路径** —— 每当宿主从磁盘附加一个会话时，pi 都会重
    新发出 `session_start`（`reason: "resume"`/`"startup"`），也就是宿主重启之后打开的每个
    chat，而那些路径通过 `hydrateSession` 而不是 `openChatSession` 安装运行时。**已知缺口：
    ** 这个 64 槽环是一个全局列表，而宿主把 ext-UI 帧广播给每个客户端，因此一批会话同时
    附加时可能驱逐某个客户端仍需要的帧；并且没有任何东西会为被删除或其工作区被移除的会话清
    除帧。按会话分桶，加上在这两个事件上清除，就是当任一件事变成现实时的修法。宿主范围的
    **`models`** 列表保持全局（不按会话），旁边是 **`modelsRefreshing`** —— 被等待的
    `model.refresh` 在途标志 —— 与 **`modelsFresh`**，即该列表的*来源*：只有当它持有一次被
    等待的强制刷新的已装入结果时才为真，`NewWorkspaceDialog` 在可以替换一个目录里没有的模
    型之前需要它。它就在这里、紧挨着列表，正是因为 **`models` 是应用范围的**：
    `setModelsForProviderVersion`（一次带守卫的 `model.list` 快照，其处理器用它所启动的分
    离刷新之前的数据作答）在同一次写入里**丢掉**它，因此权威随任何消费者所替换的列表一起落
    地 —— 若作为某个消费者的本地标志，它会活得比它的对象更久，并确认一个随后被 `create()`
    拒绝的已移除模型。`beginModelsRefresh` 捕获并返回当前 provider 版本；
    `finishModelsRefresh(version, RefreshedModels|null)` 只落地匹配的应答（列表 + 来源 +
    清空的在途标志在一次写入里；`null` = 刷新失败 —— 保留当前列表*及其*来源，因为什么都没
    装入）。来源来自**宿主的**`complete`，绝不是"有应答到达"：宿主限制它等待 pi 的时长，因
    此一次应答可能携带注册表当前的样子，而本会结算它的那一遍仍在运行 —— 这样的列表会被装入
    （它*确实*是当前的）但丢掉权威，因为据此断定某个模型已消失正是那个错误。
    **`dropModelsFreshness`** 是第三个写入者：放弃权威而*不*替换列表，这正是激活的消费者必须
    **同步**做的事 —— 否则更早的消费者设置的标志会跨过这次激活，让一份继承来的列表在自己
    `model.list` 应答落地之前冒充本次打开自身的真相。**`providerVersion`** 是从宿主观察到
    的、单调且无数据的 `provider.changed` 代数；**`noteProviderChanged()`** 原子地递增它，
    并清空 `models`、新鲜度与任何旧的刷新 spinner。**`providerStatus`** 是唯一一份
    `provider.status` 快照（`{ version, report, failed }`）；**`providerStatusVersion`**
    是它的失效代数，由 `noteProviderChanged`、每次 welcome、每个成功的无目标登录帧（也包括
    另一处 `applyLoginFrame` 挡在本客户端打开对话框之外的登录），以及
    `invalidateProviderStatus()` 推进。**`installProviderStatus(version, report | null)`**
    只装入对当前版本的应答（`null` = 读取失败：保留最后一份报告，置 `failed`）；
    single-flight 读取位于 `panels/useProviderStatus`。`model.list` 与 `model.refresh` 的应
    答都通过带版本守卫的 store 动作装入，因此任何 picker 或更旧的异步应答都无法提供一个已
    移除的运行时代。传输层拥有带守卫的重读；shell 的配额控制器把它当作一次立即的关闭快照失
    效。其他目录传输工作位于 `chat/useModelCatalog`，不在这里（store→transport 这条边保持
    仅类型）。**应用内登录**状态 **`activeLogin: LoginState | null`**（类型来自 `auth`）是
    **扁平的 + 无会话的**（登录在任何会话存在之前就跑在 Welcome 屏上 —— 把它路由进某个会话
    运行时会丢掉它的帧）：纯函数**`foldLoginFrame`** reducer 放在这里（正如 `reduceExtUi`/
    `reduceSessionEvent` 那样 —— `auth` 保持展示型）。**`reserveLogin()`** 只在既无
    `loginStarting` 也无对话框被持有时同步抢占全局启动槽；**`finishLoginStart()`** 在完成
    或失败时释放它。Providers 与 MCP 共享那个槽；只有属主请求的完成才释放它，而不是迟到的
    `beginLogin` 应答。**`beginLogin(loginId, providerId, target?)`** 打开登录，保留一个匹
    配的由帧创建的对话框（帧可能抢在应答之前），并拒绝覆盖另一个不同的活动登录。工作区移除
    会退掉它定向的 MCP 对话框；迟到的启动应答与有墓碑工作区的帧无法重建它，也无法让全局登
    录槽被占住。**`applyLoginFrame(push)`** 折叠一个入站的 `provider.login` 帧（若帧先到则
    创建 `activeLogin`；忽略针对另一个活动登录的帧）；两者都保留 MCP 推送的 `target`，这正
    是 Providers 与 MCP 设置各自只打开自己登录的方式，**`clearLoginInput()`** 在应答发出的
    瞬间丢弃活动输入（不双提交），而**`clearLogin(loginId?)`** 关闭它 —— 给了 id 时只关那
    个登录，因此过期表面绝不会丢更新的那个；**`clearSettledLogin(providerId)`** 是连接后的
    交接，只清除该 provider 已结束（success/error）的无目标登录。针对本客户端未持有的登录
    的终态帧不打开任何东西（被取代登录的错误会到达每个客户端）；它作为
    **`unheldLoginFrame`** 保留，以便 `beginLogin` 在它正是本客户端自己的快速失败抢在启动应
    答之前时重放它。**设置表面**状态 —— **`settingsOpen`** + **`settingsSection`**（一个
    const 对象枚举：`Providers`/`Models`/`Mcp`/`Github`/`Appearance`/`Chat`/`Layout`/
    `Updates`/`Terminal`/`Workspaces`/`Templates`/`Notifications`/`Privacy`/`Feedback`/
    `Extensions`；`LineWidth` 与 `Review` 两个键被保留，以便旧深链仍能通过类型检查，但它们
    不是导航段 —— `panels/settingsNav` 把它们解析到某个可用段，其控件现在位于
    Chat/Appearance 与 Models），配 **`openSettings(section?)`**（深链到某段，默认
    Providers）/**`closeSettings()`** / **`setSettingsSection()`** —— 放在这里，是为了让顶
    栏齿轮、Welcome 的 provider 警告与更新就绪的 shell 提示都能深链而无需逐层传 props。可
    选的 Update 键仅用于导航。原生更新器的快照/动作仍留在 `updates` 的 shell 本地钩子状态
    里；可选的 **`hostUpdate`** 生命周期快照是宿主领域状态，从 `server.welcome` 原子装入，
    或被那唯一一次 `host.updateAvailable` 推送替换。它在临时断连期间保持可见；稍后的
    welcome 会替换或清空它。短命的 **`notificationPromptOpen`** 加
    **`openNotificationPrompt()`** / **`closeNotificationPrompt()`**是注意通知权限前言的客户
    端本地渲染标志（节奏/稍后提醒与触发器位于 [[submodule-web-notifications]]，绝不在
    store）。短命的 **`surveyInvitationOpen`** 加 **`showSurveyInvitation()`** /
    **`hideSurveyInvitation()`** 是宿主定向问卷邀请的渲染投影，在每次有效 welcome 之前清空。
    **`surveyWindowOpen`** 是邀请与 Settings → Feedback 共同打开的那唯一问卷窗口；
    **`openSurveyWindow()`** 打开它并在同一次更新中关闭邀请与 Settings，
    **`closeSurveyWindow()`** 关闭它。**`inAppSurvey`**（由 **`setInAppSurvey`** 写入）是
    Feedback pane 与该窗口所绘制的、收窄后的已启动问卷。计数、关闭与完成留在宿主的
    `feedback.json`。**主题**偏好状态 —— **`theme: ThemeId`**（保留的不透明固定选择）、
    **`themeMode: ThemeMode`**，以及可选的 **`systemThemePair: SystemThemePair`** —— 通过
    **`applyConfig(config)`** 从 `server.welcome` /`settings.changed` 广播到达。来自前特性宿
    主的 welcome（或任何畸形的运行时模式/配对）会在这个读取侧被归一化为固定模式，保留其有
    效的不透明 `theme`，以及仅当是完整字符串配对时才保留的配对。这些纯粹是宿主值：生效的本
    地外观/id 与媒体监听器绝不进入 Zustand；shell 通过 `themes` 驱动它们。默认值仍是固定
    `DEFAULT_CONFIG.theme` 且无配对，但当既有的 `welcomeGeneration === 0` 时，shell 保留
    React 之前的偏好提示，而不是把那些占位值盖在上面。复用那条就绪边避免了第二个派生的
    config 水合标志；首次 welcome 之后，断连/重连保留最后一份权威 config，而稍后的 welcome
    可以替换它。**`terminalReplayKb: number`**、
    **`terminalWindowsShell: TerminalWindowsShell`**、
    **`composerGrowthLimit: ComposerGrowthLimit`**、**`chatLineWidth` / `fileLineWidth`**、
    它们各自独立的 **`chatLineWidthBounded` /`fileLineWidthBounded`** 开关、
    **`customLayoutPresets: LayoutPreset[]`**、用于新 chat 的可选**`defaultModel: WireModel`
    / `defaultEffort: ThinkingLevel`**、picker 的 **`favoriteModels` /
    `recentModels: WireModel[]`**（默认 `[]`，因此 v77 之前的宿主读作 "none"）、**
    `analyticsEnabled: boolean`**、**`analyticsConsentConfirmed: boolean`**、**
    `notificationsEnabled: boolean`**（注意通知总开关）、**`subagentsEnabled: boolean`**、
    **`subagentMaxConcurrent: number`**、**`disabledExtensions: string[]`**（默认 `[]`，因
    此 v82 之前的宿主读作 "all on"）、**`jbcentralQuotaEnabled: boolean`**、
    **`jbcentralQuotaRefreshSeconds: number`**，以及 **`onboardingDismissed: boolean`**（缺
    失 ⇒ `false`）全部走同一个 `applyConfig` fold（宿主拥有，逐字段按 contracts 辅助函数给
    默认值/做校验，因此更旧或畸形的宿主快照无法毒化 store）。`terminalWindowsShell` 经
    `isTerminalWindowsShell` 收窄，否则使用 `DEFAULT_CONFIG.terminalWindowsShell`；
    `TerminalSettings` 消费两个终端字段，而终端启动仍归服务端所有—— Terminal、Line width、
    Chat、共享 Layout 目录、Privacy、provider 控件与 shell 配额这些读取侧。分析偏好与确认
    各自独立默认为 false；store 绝不把偏好升级为确认。一个 selector 组合宿主能力、已水合配
    置与缺失的确认，以驱动那唯一一次首次启动提示；`selectOnboardingActive` 在客户端本地的
    **`onboardingWizard`**门禁（`pending | shown | hidden`，由 Welcome 通过
    `setOnboardingWizard` 设置）处于显示状态时按住它（以及问卷邀请）—— 包括它的完成屏，即
    在 `onboardingDismissed` 已持久化之后，直到有项目打开 —— 或在冷启动形状的 store 上仍待
    定（宿主支持、未关闭、没有打开或最近的项目）。Project Home 读取同一个 selector 来隐藏
    它的 Projects 工具栏与 pane，因此引导流程拥有整个窗口。已持久化偏好的预置与最终选择更
    新都通过 `applyConfig` 收敛；对话框的视觉草稿保持组件本地。
    **`chatMessageOrder: ChatMessageOrder`** 与
    **`streamingResponseMovement: StreamingResponseMovement`**反而是客户端本地的呈现偏好，由
    chat 偏好接缝一起水合，来源是按宿主限定的浏览器 localStorage 或原生 shell 注入的
    backend-profile/window 作用域适配器。消息顺序默认最早优先；移动默认
    `{ settle: 75, trigger: 100 }`。`setChatMessageOrder` / `setStreamingResponseMovement`
    无需服务端往返即可改变它们，而 `applyConfig` 绝不可能覆盖任何一个。`ChatView` 按顺序投
    影每个运行时而不重写规范轮次，并把移动窗口传入它唯一的滚动控制器。
    **`hiddenModels: ModelRef[]`**（`{ provider, id }` 配对）是客户端本地的模型可见性偏好，
    带纯函数 **`hydrateHiddenModels(list)`** / **`setModelVisibility(model, visible)`**/
    **`toggleModelVisibility(model)`** 动作；store 不触碰任何存储 ——
    `chat/modelVisibility` 负责水合与持久化它，而 `applyConfig` 绝不可能覆盖它。
    **`selectPickerModels(models, hidden, keep)`** 是每个模型 picker 的调用方都会应用的唯一
    过滤：它保持目录顺序，返回可见模型外加任何 `keep` 条目（当前选择），因此隐藏绝不会让某
    个 picker 丢掉它正在显示的模型。完整目录在 `ModelsSettings` 的可见性配置里仍然可用。已
    实例化的工作台帧、当前/默认预置 id 与组上限是独立的本地值，由 `shell/layoutState` 水合；
    `applyConfig` 绝不可能覆盖它们。**toast 队列** —— **`toasts: Toast[]`**（最早优先）配
    **`pushToast(toast) → id`** /**`dismissToast(id)`** 以及顺手的
    **`toast.error/success/warning/info(message, title?)`** 辅助函数（包装 `pushToast`，因此
    非 React 调用点 —— 一次发射即忘线上调用里的 `.catch` —— 也能发一条）—— 放在这里，任何
    表面都能升起一次瞬态通知。一个 toast 还可以额外携带 `durationMs` 与一个
    `{ label, onClick }` 动作；`panels/Toaster` 通过 Radix `ToastAction` 渲染该动作，并在
    其他方面拥有超时/滑动关闭（错误会保留直到被关闭）。`pushToast` **合并一条完全相同的无
    动作活动 toast**（同 variant/title/message/duration —— 一次重试的失败返回既有 id 而不
    是叠一个孪生），绝不合并那些回调命名不同反向动作的可操作回执，并且**只通过驱逐最旧的无
    动作通知来应用 5 条 toast 的上限**。可操作回执一直保留，直到它们自己的有界时长关闭它们
    或用户关闭它们；如果可操作项已经超过五条，新的无动作 toast 就是该上限的驱逐候选，而不
    是一条 Undo 回执。它是一次**没有更好去处被拒绝的线上调用**的家（没有 chat 标签页来承载
    错误轮次），补足 `appendErrorTurn`（后者处理 chat 内的情况）。宿主范围的
    **`templatesVersion: number`**计数器 + **`bumpTemplatesVersion()`**（递增）是一个裸失效
    信号，形状与下面的 `fsChangesByWorkspace` 的 `tick` 相同 ——
    **`panels/TemplatesSettings.tsx`** 与**`chat/TemplateEditorDialog.tsx`** 在
    `template.save`/`delete` 之后调用它，而 Templates 设置面板自己的列表据此重取（它的
    `useTemplateList` 取数代数）。它**刻意不是** composer `/` 菜单的新鲜度来源 —— 那次取数
    在每次菜单打开时都不带缓存地运行，因为文件也会在应用之外改变，那是任何应用内计数器看不
    到的地方（见 `chat/SPEC.md` 的 Template slots 段）；store 只持有计数器，绝不取数。**实
    时刷新信号** ——**
    `fsChangesByWorkspace: Record<workspaceId, { tick, paths, truncated }>`** 配**
    `noteFsChanged(payload)`**（折叠一个 `workspace.fsChanged` 推送：`tick` 逐帧递增；
    `paths`/`truncated` 是最后一批）—— 面板选择自己工作区的条目，并在 `tick` 变化时重取
    （store 只持有信号，绝不取数；`applyWorkspaceRemoved` 丢弃被移除工作区的条目）。
    **`fsChangedPaths(batch)`** 把一批读作它的路径，或在它被截断/无路径时读作 `null`（任何东
    西都可能变了）—— 对 `useLiveTabContent` 与资源宿主是同一条规则。
    **`selectFileChangeSince(state, previous, workspaceId)`** 以同样方式回答一次 store 转换，
    并对新连接也回答 `null`，因为断连期间丢失的推送从不重放；资源宿主的
    `subscribeFileChanges` 使用它，因此已挂载的 `ResourcePreview` 会在重连之后重读。
    **review 分片** —— **`reviewsByWorkspace: Record<workspaceId, ReviewSnapshot>`** 配
    **`setWorkspaceReview`**（一次 `review.get` 读取落地）与 **`applyReviewChanged`**（折叠
    一个 `review.changed` 推送 —— 全快照、幂等；每个客户端，包括某次变更的发起者，都在这里
    收敛 —— 没有乐观更新）；`applyWorkspaceRemoved` 丢弃该条目；待定草稿计数是一个
    selector（`selectReviewDraftCount`），绝不在组件里重复。**Skills 重载徽标**搭同一个
    tick，不需要单独的信号：`noteFsChanged` 还折叠
    **`skillChangeTickByWorkspace: Record<workspaceId, tick>`** —— 最近一次*与 skill 相关*批
    次的 tick，来自宿主撰写的 `payload.skillChange` 语义（具体项目 skill 路径为 `detected`，
    真正无路径的不确定性为 `unknown`，具体的非 skill 变动为 `none`）。它与有上限的通用
    `paths`/`truncated` 配对无关，因此一次大型构建无法冒充 skill 变更，而路径上限之后的
    skill 事件也不会丢失；它保持*累积*，因此稍后的非 skill 批次绝不清空它。新 watcher 的合
    成启动轻推仍是保守的 `unknown`。传输层集中化的 skill 加载准备会等待
    `workspace.watchReady`，除非 watcher 已被知晓为就绪，否则折叠一次重复的 unknown 兜底
    （该事件推送可能在重连期间已经死掉），然后捕获该次加载的基线 tick。新加载的会话保持干净；
    就绪之后的真实 skill 帧仍比基线更新。每个 chat 记录
    **`skillsSyncedTickBySession: Record<sessionId, tick>`** = 它加载 skills 时的 tick。它
    **只在资源针对当前磁盘实际（重）加载时**推进：一次新的 `openChatSession`、一次仅磁盘的
    `hydrateSession` 附加，以及成功重载时的 **`markSkillsSynced(sessionId, syncedTick)`**
    （`markSkillsSynced` 是**单调**的 —— `Math.max`，因此乱序的重载完成无法把基线往回移 ——
    并且对一个已被处置的会话是 **no-op**，因此迟到的完成无法复活被 `closeChatRuntime`/
    `clearWorkspaceState` 丢弃的条目）。**活动**的 `hydrateSession` 恢复复用服务端会话已加
    载的 skills（`getMessages` 只返回转录，不重载），客户端无法为它定日期，因此它**什么都
    不推进** —— 若已观察到 skill 变更，该 chat 保持*保守地过期*，绝不虚假地清除。那个
    `syncedTick` 是 skill 加载往返**开始时**捕获的工作区 tick，紧接在共享的
    `workspace.watchReady` 准备之后（`selectWorkspaceTick`，由调用方在 `session.create`/
    `reloadResources` 之前快照；`getMessages` 转录读取在自己的请求之前快照，不等就绪），而
    **不是**在完成时 —— 因此一次 `fsChanged` 帧在加载在途期间折叠进来的 skill 变更（该次加
    载并未看到它）会留在基线之后并保持徽标点亮，而不是被静默吸收。selector
    **`selectSkillsStale(state, workspaceId, sessionId)`** = `skillChangeTick > syncedTick`
    —— 由 store 派生（能挺过 `ChatView` 的标签切换重挂载）且按会话（加载了当前 skills 的兄
    弟/更新 chat 不会被标记；一次重载只清除它自己的）。`updateFileTabContent` 把内容、
    `ResourceMeta` 与捕获的 fs tick 一起写入。它的 diff 孪生 `updateDiffTabContent` 还额外
    写入两侧的元数据、解析出的原始 oid 与 `loadedTarget`；一个 `DiffTab` 在 fs-tick 或
    review-target 维度任一上过期，而两个身份都不得活得比它所描述的内容更久。短命的
    **`reveal-tool` `LayoutIntent`** 是把某个单例侧工具揭示/聚焦的、与排布无关的请求；shell
    布局集成消费它并解析该工具当前保存的位置。**`changesRequest`** 与**`specRequest`** 给
    那一次揭示加上可选路径/条目目标，并携带一个浏览器本地的请求时中心目的地，而不把布局关
    注点暴露给功能视图。异步解析携带点击时捕获的本地目的组的导航时钟戳记：如果稍后的注意超
    过了它，工具仍可能高亮该条目，但过期的完成无法抢走焦点。两个意图在处理后都被消费，因此
    重挂载/重读无法重放一次结构性打开。之所以仍需要两个字段，是因为一个被 gitignore 的
    spec 属于规格图谱，而不是 git 派生的 Changes 视图。**`specsByWorkspace`** +
    **`setWorkspaceSpecs`** 持有每个工作区的 `spec.graph` 快照（由 `panels` 的
    `useWorkspaceSpecs` 取回，并在工作区 fs tick 上保持新鲜），因此 chat 的轮次分隔线可以
    根据 Specs 面板所渲染的同一份快照，把一条被写入的路径分类为 spec —— "这个文件是 spec"
    只有一处定义，经由 **`specPathMatcher(nodes)`** selector；随工作区在
    `applyWorkspaceRemoved` 中被丢弃。`setWorkspaceSpecs` **在重读未发现变化时保留先前的数
    组身份** —— 大多数 fs tick 都不触及任何 spec，而一个新身份会让 `ChatView` 的匹配器
    memo 失效，并以约每秒一次的速度重新推导每个打开 chat 的整份转录。**`openDoc(tab)`** 缓
    存并放置一个已解析的 **`DocTab`** 或一个 **`PlanTab`**（`kind: "plan"`，id 为
    `${workspaceId}:plan:${sessionId}` —— 每个 chat 一页，重开即聚焦）。本地放置持久化只保
    留 resolver 种类
  + 持久化会话身份，绝不保留缓存内容。`PlanPane` 实时读取宿主拥有的计划，因此该页没有会过
    期的快照。**`DiffTab`** 是在**单个 diff scope** 上由某个渲染器分派的一个已变更文件（id
    为 `${workspaceId}:diff:${scopeKey}:${path}` —— 每个 *(file, scope)* 一个标签页：
    **scope 是标签页身份的一部分**，因为标签页的内容绝不能因为 Changes 工具的 scope 在它下
    面翻转就改变含义；标签页携带自己的 `scope`，这也是 `DiffPane` 重读时所用的，绝不是面板
    当前的 scope）。那是 `DiffTab` 联合的 `GitDiffTab` 一半；另一半 `FileComparisonTab` 比
    较两个工作区文件（`before` + `path`，没有 scope 也没有已加载的两侧—— 见
    `panels/SPEC.md`），而 `isFileComparison(tab)` 区分它们。**标签页身份固定的是它显示*哪
    个 scope*—— 种类，外加 commit scope 时的那个 sha。** 分支 scope 的标签页意味着
    "这个文件对比该工作区的**当前**评审目标"，而那个目标移动 —— 因为提交落到了分支上，或因
    为用户重新指向了它 —— 是与标签页下方 worktree 改变相同的实时刷新契约，而不是含义改变；
    因此目标 ref **不**属于标签页 id（分支名钉不住任何东西 —— 只有 commit sha 是不可变的，
    而它已在 id 里）。它*确实*要求的是：目标移动时标签页重读：`selectDiffTabTargetRef` 就是
    那个第二实时维度（见 `panels/SPEC.md` 的实时刷新契约）—— 以及标签页**记录它内容实际针
    对读取的那个目标**（`DiffTab.loadedTarget`，必需，由每次内容写入写入）。pane 只在其资
    源被本地选中时挂载，因此没有那条记录，一个在后台期间目标已移动的 diff 会带着已经握在手
    里的新目标挂载，断定什么都没变，并在新目标的标签下显示*旧*目标的 diff；缓存值正是挂载
    时用来比较的东西。`FileTab` 与 `DiffTab` 共享 `rendererId` 加上不透明的 `viewState`，
    由工作区显式的 `setTabRenderer` 与 `setTabViewState` 写入，因此一次在工作区切换之后才结
    算的卸载回调仍会更新它所属的标签页。注册表拥有视图状态解释；改变渲染器或手机/桌面实现
    会丢弃先前的不透明状态，而每种实现会拒绝它无法解释的状态。仅 diff 的呈现通过
    `setDiffTabView` 保持 `view` split|inline（默认 split），通过
    `setDiffTabIgnoreWhitespace` 保持 `ignoreWhitespace`。文件文档上遗留持久化的 `view` 与
    diff 文档上的 `rendered` 不在被接受的缓存形状之内，读取时被忽略；不存在迁移状态。由
    `ChangesPanel` 在 `keep` 意图处打开（双击 / *Open as tab*）；单击则打开 **`ChangesTab`
    **。**`ChangesTab`**（`kind: "changes"`，id 为 `changesTabId` 算出的
    `tupleKey("changes", workspaceId, scopeKey)` —— 每个 *(workspace, scope)* 一个）是覆盖
    某一个 scope 的**每一个**已变更文件的评审表面。与 `DiffTab` 一样，它的 scope 是它身份
    的一部分，而目标 ref 不是；与 `DiffTab` 不同的是它**不携带内容** —— 该 pane 读取
    `git.status`，每个 section 自己读 `git.diffFile`，因此持久化的布局标签页只是 kind + id
    + name + scope，重载水合无需请求即可重建该标签页。该标签页*确实*拥有的是评审进度：
    `viewed` 路径、`activePath`（视口顶部的 section，或 One-file 模式下显示的文件；
    `ChangesPanel` 从它高亮）、覆盖刻板默认值的逐路径折叠覆写、一次性的 `reveal`
    （`{ path, tick, commentId? }`，由 `openChangesTab` 写入，并由该 pane 在一次
    `consumeChangesTabReveal` 写入中消费，那次写入同时落地 `activePath`、该 section 的展开，
    以及一个瞄准该标签页 scope 的 finding 的 `reviewFocusRequest`）、所有 section 共享的
    `view` split|inline + `ignoreWhitespace`，以及通过工作区显式的 `setChangesTabSection*`
    动作写入的逐路径 `rendererId` / 不透明 `viewState`（完全相同的视图状态不算一次写入）。
    每个 section 还持有不透明的 `reviewDrafts`，由工作区/标签页/路径/key 显式的函数式动作
    更新。它们能挺过渲染器/视图状态变化，但挺不过标签页关闭或重载；写入无法重建已关闭的标
    签页。第一次草稿写入保留预览而不导航，且没有任何草稿写入是宿主评审变更。渲染器拥有草稿
    形状与过期选择策略。堆叠对单文件模式**不**在标签页上：`changesLayout`（`"stacked"` 默
    认 | `"single"`）与 `changesView` 一样是应用范围的，是一种阅读偏好而不是某次评审的属性。
    该标签页与 `FileTab`、`DiffTab` 一样有资格成为预览 —— 直到它持有进度：第一次
    `viewed` 或 `kept` 写入（`recordChangesTabProgress`）也在同一个 `set` 里保留它的规范布
    局放置，因此恢复的与拆分组的预览无法仅仅因为遗留预览镜像为空或指向另一个组就驱逐读者的
    标记。保留最后一个 hunk 会原子地记录文件与 hunk 进度；批量 viewed 更新是一次写入，而不
    是每个文件一次 store 通知加一次数组复制。进度是**会话作用域**的：`viewed`/`kept` 只活
    在标签页对象上，重载会重建为空的标签页，而被 viewed 的文件不会因 agent 重写它而失效
    （kept 的 hunk 会，经由它们的内容 key）—— 跨重载按文件哈希持久化 viewed 是已知的后续工作，
    而不是意外。**`turnsByWorkspace`** 持有每个工作区由宿主记录的 agent 运行
    （`TurnChangeSet[]`，按 `startedAt` 最早优先，按 id 去重并像宿主那样封顶 30 条）。
    `applyTurnChanged` 既为一个未加载的工作区播种，也替换既有的回执。`setWorkspaceTurns`
    接收 `workspace.turns` 的应答、那次读取之前捕获的不可变列表，以及它的连接代数。它用快
    照替换基线，然后叠加对象身份不在基线中的当前回执：初始/重连水合期间的推送胜出，包括对
    同一 id 的替换，而宿主省略的未变化基线条目被裁剪。两条写入路径应用相同的排序、去重与封
    顶。不需要单独的推送缓冲或修订版映射；唯一的加载器只在有界的 store 列表旁保留有界的基
    线。断连/旧代数的应答是 no-op；工作区移除会丢掉该列表，它的墓碑会拒绝稍后的快照与推送。
    该列表被 chat 的轮次分隔线（该轮的回执）与 Changes scope 菜单（*Last turn*）读取。
    shell 是唯一的加载器（`useLoadWorkspaceTurns`，在每个连接代数上重读 `workspace.turns`，
    因此在 socket 断开期间结算的运行仍会到达）；其他每个消费者只通过 `useWorkspaceTurns` 读
    取。**`requestChangesView(workspaceId, path | null, scope?)`** 为 chat 的*Review turn*
    动作增加了两个可选维度：一个 `scope` 与揭示意图原子地落在 `diffScopeByWorkspace` 上，
    而 `null` 路径意味着"打开该 scope 的评审标签页"而不是"揭示这个文件"。它揭示 **Files**
    工具并在同一次写入中设置 **`filesMode: "changed"`**，**`revealChanges(workspaceId)`**
    （chat 的提交回执）也是如此；`filesMode`（`"all"` 默认）与 `changesView` 一样是应用范
    围且仅限本次会话。**`filesTreeByWorkspace`**（通过 **`selectFilesTreeView`** 读取，缺
    失时是一个稳定的空视图）是 Files 树按工作区记住的样子：All 模式的 `expanded` 文件夹、
    Changed 树的 `changedCollapsed` 文件夹（所有 scope 共用一套）、每个视图的 `scroll`
    （`all` / `changedList` / `changedTree`，各自标记它所属的 scope 与对比目标，因此另一个
    scope 或目标分支会从顶部开始），以及 `seenActivePath`。**`revealInFiles`** 在一次写入
    里展开一条路径的每个祖先并把它记为已见，因此自动揭示在返回时能判断活动文件期间是否变过；
    每次写入都是幂等的（未变化的视图保持其身份），被移除工作区的视图在
    `applyWorkspaceRemoved` 中被丢弃。持久化在 store 之外，位于 `panels/filesTreeState`。
    **`selectActiveFilePath`** 是活动文件或 diff 标签页的路径，否则是 `null`。
    **`diffScopeByWorkspace`** + **`setDiffScope(workspaceId, scope)`** 持有每个工作区的
    Changes 面板正在对比*什么*（通过 **`selectDiffScope`** 读取，默认是共享的、引用稳定的
    `BRANCH_SCOPE`）；按**工作区**做 key，而不像 `changesView` 那样应用范围，因为一个
    scope 属于那个分支的评审 —— 一个 commit sha 在另一个 worktree 里毫无意义 —— 并随工作区
    在 `applyWorkspaceRemoved` 中被丢弃。短命的**`chatLocationRequest`** —— 历史搜索跳转的
    深链；请求方激活目标项目 + 工作区，工作台 shell 集成打开/水合目标 chat，`ChatView` 消
    费并清除 —— 是**
    `ChatLocationRequest { workspaceId, projectId, sessionId, messageIndex, anchorText, navigation? }`
    **，由 **`requestChatLocation(req)`** 设置（它在切换工作区*之前*捕获并推进某个已水合目
    的组的本地时钟，并**原子地**设置 `selectedProjectId` + `activeWorkspaceId`，这与
    `activateWorkspace` 维护的不变量相同，因为目标 chat 可能位于与搜索发起处不同的项目/工
    作区 ——当目标项目缺席时，调用方 `useHistorySearch.openMessage` 先加载目的项目的工作区），
    并由**`clearChatLocation()`** 清除；目标的锚点对运行时的 `turnIdByMessageIndex` 解
    析（见 `chat/SPEC.md` 的水合条目），缺失时回退到最新的 `anchorText` 匹配。兄弟的短命
    **`historyOpenRequest { id, sessionId }`**—— 由 **`requestHistoryOpen(target)`** 设置、
    由 **`clearHistoryOpen()`** 清除 —— 把 shell 的应用级 `Ctrl+R` 带给某个 chat，后者打开
    （或已打开时重设作用域）它的历史浮层；它之所以经过 store，正是因为该组合键完全在 chat
    子树之外触发（见 `shell/SPEC.md` 的 "Global chords"）。目标来自**`selectHistoryTarget`
    **（本地选中的 chat，否则该工作区最新的 chat），而动作把工作台选择与请求原子地一起排队；
    请求 id 用于关联被取代的清理，意图携带 chat 资源，因此即便发生缓存/放置 id 别名（包
    括由仅放置铸造解决的 id 碰撞），选择仍然是语义的。那次选择刻意不聚焦标签页，因为挂载的
    历史查询拥有焦点。shell 更新该组的本地注意，使目标主体挂载并消费该请求，而不发布结构性
    快照。`EditorTab`（`FileTab`
  | `ChatTab` | `DocTab` | `DiffTab` | `ChangesTab` | `PlanTab`）+ `TerminalTab` + `ClosedChat` +
`SessionRuntime` 类型。（chat 的*渲染*类型 + 渲染器位于 `chat` 模块。）`selectors.ts` 里的
纯上下文 selector 从这些规范 id 与集合解析活动 `Workspace`、它所属的项目 id，以及 shell 的
上下文项目；派生的活动项目状态绝不单独存储。
- **公开接口（barrel）：** `useAppStore`；`selectActiveWorkspace`、`selectWorkspaceById`
  （"这个 id 的工作区"的唯一查找 —— `selectActiveWorkspace` 就是它作用于活动 id，而
  `openFileInTab`/`ChatView` 通过它读取 worktree 根）、`selectCanRenameChat`（ChatView 与
  shell 控件共享的唯一协议能力推导）、`selectWorkspaceTerminals`（宿主拥有的终端目录；布局
  可见性门禁从它收到的文档 + 本地注意推导挂载身份，而宿主附加仍按终端独占）、
  `selectActiveWorkspaceProjectId`、`selectHistoryTarget` + `HistoryTarget`（shell
  `Ctrl+R` 的路由目标：本地选中的 chat 资源，否则该工作区最新的 chat）、
  `selectContextProject`、布局放置 selector（递归中心加左/右/底辅助组）、
  `selectAttentionCenterTab`（本地最后中心焦点中的选中资源）、
  `selectCurrentRouteChatTarget`（仅当它的工作区与带戳导航仍然当前时生效的精确 chat 意图）、
  `selectSkillsStale`、**`selectDiffScope` + `BRANCH_SCOPE`**（某个工作区的 Changes 面板
  正在对比什么，默认是共享的 branch-scope 常量）、**`selectDiffBaseRef`**（它被度量的那个
  ref —— 宿主唯一一次解析的客户端镜像）、**`selectDiffTabTargetRef`**（那个 ref *作为一个
  打开的 diff 标签页的实时维度*：分支 scope 标签页的目标，对两侧都不会动的
  commit/uncommitted 标签页为 `""` —— 在这里派生，绝不在面板里重新拼装）、
  `selectChangesTabSection`（按工作区、标签页与路径取一个 Changes 标签页 section —— 评审草
  稿持久化通过它读取草稿槽）、`selectChangesTabRendererIds`（某个 Changes 标签页各 section
  的 path → 选中渲染器 id，用 `useShallow` 读取，因此评审工具栏的能力门禁会忽略草稿与视图
  状态写入）、`selectWorkspaceTick`（同步基线快照）、`selectWorkspaceSessionIds`（去重后的
  本地 chat 放置 + 历史成员关系，用作重连对账基线）、归一化的会话状态 selector/action（精
  确会话 + 工作区/项目 needs-input/working/unread-finished 汇总、直接激活、就绪的精确完成
  确认）、`matchesWorktreePath`（把 agent 报告的路径 —— 相对或绝对 ——与 worktree 相对路径
  对齐；由 Changes 深链与 spec 分类器共享。后缀规则**仅适用于绝对报告**，并锚定在分隔符上：
  若不锚定，`/wt/src/a-foo.ts` 会匹配 `src/foo.ts`；若应用于相对报告，`module-b/SPEC.md`
  会匹配*根*的 `SPEC.md`）+ `specPathMatcher`（一条被写入的路径是否规格图谱节点？）；
  `selectCatalogModel`（把模型 ref 对**实时** `models` 列表解析 —— 会话自己的 `model` 是它
  被创建时的快照，因此其上由宿主算出的事实，今天即 `thinkingLevels`，要通过它读取；当该
  ref 已离开目录时，调用方回退到快照）；`selectPickerModels` + `ModelRef`（上面的隐藏模型
  picker 过滤）；MCP 推导 `deriveMcpServerRows` +`McpServerRow` / `McpRowState`、
  `MCP_STARTING_BOUND_MS`、`selectMcpRead` + `McpRead`、`selectMcpWorkspaceStarting`、
  `McpWorkspaceProjection`、`selectSupportsMcp`，以及资源辅助函数 `orderMcpResources`；
  `selectSupportsProjectTrust`（信任摘要与 pi 级资源授予所用的
  `PROJECT_TRUST_PROTOCOL_VERSION` 门禁）；`toast`（随处可发的辅助函数）、`Toast`（类型）、
  web 本地的帧/工作区视图/注意 selector 与原子动作、资源渲染状态类型（文件/diff/虚拟文档
  /计划/chat）、`TerminalTab`、`ClosedChat`、`SessionRuntime` + `EMPTY_RUNTIME`（ChatView
  创建前的兜底）、`ChatLocationRequest`（类型）、`reduceSessionEvent`。
- **允许的依赖：** `contracts`（`Project`/`Workspace`/`Model`/`ThinkingLevel`/
  `SessionStats`/`SlashCommandInfo`/`ExtUiRequest`/`LoginPush`/`WorkspaceFsChangedPayload`
  /`LayoutPreset`/`AppConfig`/`ThemeId`；`DEFAULT_CONFIG` 用于 welcome 之前的默认值；
  `PiEvent`/`LoginFrame`，**仅类型**）；`lib`（共享的路径 + 数组 + 规范消息原语 ——
  `normalizePath`/`isAbsolutePath` 供 `matchesWorktreePath`，`shallowEqualArrays` 供快照身
  份守卫，`userText` 加上 skill 调用解析器/匹配器供用户消息回显对账，MCP 注意/活动谓词；是
  一个叶子，因此这条边不引入环）；`chat`（`ChatTurn`/`ToolResultState`，**仅类型**）；
  `auth`（`LoginState`，**仅类型**）；`transport`（`ConnectionStatus`，**仅类型**）；
  `@thinkrail/locales`（默认 chat/终端名、结算原因文案与 toast，在创建时解析）；`zustand`。
- **禁止：** `server`/`shared`/`pi`；导入 `panels`、shell 运行时（web 本地布局状态那条边仅
  类型），或 transport 运行时。
