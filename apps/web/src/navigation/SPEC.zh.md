# navigation —— 客户端本地路由与恢复

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

客户端本地的位置层：一个后端相对、可序列化的路由，用于 main／Project Home／工作区／chat，外加浏览器
fragment 驱动器和恢复协调器 —— 后者在改变被渲染的 store 之前，依据宿主拥有的状态校验传入路由。

本模块让位置可移植，却不把它变成共享状态。浏览器标签各自拥有独立的 fragment；Electrobun shell 按
后端 profile／窗口持久化观察到的 fragment，之后的移动端 shell 按 profile／设备持久化它。跨客户端的
继续是一个显式的链接／书签动作，绝不是一个自动的、后端拥有的活动位置。

## 边界（Boundary）

- **拥有：** `NavigationLocation`；带版本的 fragment 编解码器；`NavigationDriver` 接缝
  （`read`／`replace`／`push`／传入位置订阅）；浏览器 fragment 驱动器；启动／直接链接的恢复与规范化
  兜底；push 与 replace 的历史决策；序列化意图与已校验 store 状态之间的循环／幂等守卫。
- **公开接口（barrel）：** 路由类型 + 编解码器，以及 `initNavigation(driver?)`；浏览器驱动器是默认值。
  原生适配器可以消费／产出同一个后端相对路由，而不改变 store 或 transport。
- **外部依赖：** `contracts`（project／workspace／session 的 DTO 类型，仅类型）；浏览器 History／
  Location API。
- **禁止：** server/shared/pi；拥有 project／workspace／session 快照；持久化凭据或原始后端密钥；
  导入 panels/chat/shell；写入一个由后端拥有的“活动位置”。

同级依赖边只存在于 `module-web` 中。`main.tsx` 初始化该集成，而 `shell/chatReconciliation` 经其既有
水合集成消费 store 的精确 chat 目标。

## 路由契约（Route contract）

带版本的浏览器 fragment 恰好表示以下之一：

- `#/v1` —— main／Welcome；
- `#/v1/projects/<projectId>` —— Project Home；
- `#/v1/projects/<projectId>/workspaces/<workspaceId>` —— 工作区；
- `#/v1/projects/<projectId>/workspaces/<workspaceId>/chats/<sessionId>` —— 精确 chat。

每个 id 是一个编码后的路径段。空 id、多余段、畸形编码与未知版本都无效，并规范化为 main。该路由是
后端相对的：同源 web 从其源获得后端身份；独立托管／原生客户端把它与选定的后端 profile 配对。其中不
应有任何凭据。

由 store 驱动的导航经下面的历史契约写入 fragment：用户意图的位置变化 push 一条浏览器历史条目，被动
的则原地 replace。直接打开的 fragment 与之后传入的 fragment 变化仍要经过校验。驱动器在写入之前比较
推导出的位置，因此非导航的 store 抖动 —— 尤其是流式 Pi 事件 —— 不会引发任何 History API 调用。

## 历史契约（History contract (browser Back/Forward, issue #152)）

Back/Forward 必须逐步走过用户的应用内导航。条目是**位置，不是标签状态**：每条都是恢复契约能够校验并
重新进入的路由，因此 Back/Forward 只是一次传入 fragment —— push 只改变 hash，而仅 hash 的遍历会触发
`hashchange`，驱动器早已把它作为传入路由上报。新的机制只决定*一次导航何时创建条目*。

**Push 与 replace。** 协调器按 store 边沿决策（`subscribe(state, previous)`）：

- 一个边沿上的**用户意图** = 项目／工作区范围移动、活动工作区的 `navTickByWorkspace` 推进，或活动
  工作区布局 attention 中任何按组 navigation 时钟的推进（`navigationClockByGroup`）—— 除非协调器
  自己正在应用一条传入路由（见采纳）。意图边沿**武装一次 push**。协调器在 store 订阅的**顶部**武装
  —— 在会运行同步的、清理过期精确 chat 目标之前 —— 因此一个取消了未解析 chat 目标的点击仍把自己
  的位置记录为一次 push（由精确 chat 目标的单元测试钉住）。
- 推导出的位置常常在比点击*更晚*的边沿上变化：`setActiveTab`／`openTab` 在点击时推进
  `navTickByWorkspace`，而 `deriveLocation` 读取布局 **attention**，后者由 layout-intent 处理器在一次
  后续的本地转换中更新。attention 边沿自带意图信号 —— `selectTab(..., countNavigation)` 为用户
  选择／打开／关闭推进目标组的 navigation 时钟，而对被动的本地水合或请求时已加戳的消费
  （`shouldAdvanceAcceptedNavigation` = false）保持不变。
- 一次 fragment 写入**消费这次武装**：已武装 → `push`，否则 → `replace`。该武装在位置未变的同步之间
  **持续存在**，直到一次写入消费它或一条传入 fragment 被采纳。这是承重的，而非松懈：**被延后的加戳
  打开**（新 chat 按钮、历史重新打开、跨工作区跳转）在点击时通过
  `beginCenterNavigation`／`requestChatLocation` 计入导航，并且只在一次异步往返之后才落地位置变化，
  刻意*不*重新计数（`shouldAdvanceAcceptedNavigation` = false）—— 更早清除武装会把每个这类流程降级为
  replace。被接受的权衡：一次位置中性的意图（重选当前标签）之后跟着一次*被动*位置变化，会多记录一条
  条目 —— 罕见（多客户端竞争）且自愈。没有先前意图的被动位置变化（本地水合或规范化兜底）则 replace ——
  并入当前条目：**一次用户点击 = 一条历史条目**，其 fragment 自愈到最终位置。
- **采纳：** 一条传入 fragment（启动、Back/Forward、地址栏编辑）*就是*当前条目 —— 接受它会清除武装，
  协调器在应用该路由期间所做的规范化与每次写入（`selectMain`／`selectProject`／`setWorkspaces`／
  `activateWorkspaceFromRoute`，它们本身会推进 tick 与时钟）都被屏蔽于意图测试之外并采用 replace，
  永不 push。协调器自己被屏蔽的边沿也永不取消其待定路由 —— 解析会显式清除它。解析之后的同步写入
  用 replace。
- **采纳是用户导航，在 store 上可见且立即：** 接受一条 fragment 会在校验读取开始*之前*推进（被屏蔽
  的）活动工作区 `navTickByWorkspace`。被延后的文件／chat 读取会把自己的请求时 tick 与该当前值比较，
  因此一次在途的 Back/Forward 会胜过更早的完成。
- fragment 未变 → 无 History API 调用（既有的 `lastWritten` 守卫）。

**由这些规则派生出的 Back/Forward 语义：**

- 一条 chat 的历史条目在其标签被关闭后仍然存在：Back 会通过正常的已校验恢复重新打开／重新聚焦它
  （用户决定；不做历史清洗）。原子的本地关闭会移除 attention 选中的标签并推进一次导航，因此关闭活动
  chat 会 push 邻居位置；Back 回到已关闭的 chat。
- 回到一个此后已被删除的 chat／工作区／项目会经既有权威缺失规则落在规范化兜底上；瞬时失败则保持 URL
  不变。
- 被接受的粗粒度（路由契约不变）：文件／diff／文档标签序列化为工作区路由，因此 Back 无法重新聚焦某个
  具体文件标签；从一个 chat 条目退到同工作区的、工作区级条目时，可能立即规范化回选中的 chat（一次
  “死”的 Back 按压，经 replace 自愈）。一次跳进布局 attention 尚未水合的工作区的跨工作区跳转可能创建
  两条条目（先是工作区，再是 chat）。一次*被动*范围迁移（一次远端项目关闭，或工作区移除恢复本客户端
  最近剩余的工作区，且仅在没有剩余任何工作区时回退到 Project Home）会读作一次范围移动并记录一条条目
  —— Back 回到那个已死位置会经缺失规则规范化。精确文件路由与同工作区快路径（在兄弟 chat 之间的 Back
  上跳过权威 `workspace.list` 读取）在任务规格中记录为后续工作。

驱动器接缝携带 `push(fragment)`（在浏览器驱动器中是 `history.pushState`）；原生适配器把它映射到自己的
历史模型上。

## 恢复契约（Restore contract）

传入路由是意图，绝不是领域真相：

1. 等待 store 的 **`welcomeGeneration`**，它只由那次原子完整 welcome 安装推进 —— 连接状态、协议版本与
   一个空项目列表都不是就绪信号；
2. 依据该已打开项目快照校验项目；
3. 在验证／激活工作区之前，拉取／安装该项目权威的 `workspace.list({ includeDiffStats: false })` ——
   成员关系／顺序保持完整，而按工作区同步扇出的 diff 统计保持关闭、不进入自动启动；
4. 读取之后重新检查路由代次与已打开项目，然后原子地激活一个有效工作区，推进其中心导航 tick（使更早
   的延后读取被取代），并为精确 chat 路由安装一个盖上结果 tick 的精确 chat 目标、或为工作区级路由清除
   任何更旧的精确目标；工作区路由不携带标签意图，因此既有的浏览器本地 attention 保留，普通 store 同步
   可能把 fragment 规范化到一个已选中的 chat；
5. 让 `shell/chatReconciliation` 校验／水合那个精确会话；精确目标的安装会推进一个专用代次，因此当工作区
   已经处于活动状态时这一步也会运行，而目标的消费不会重复这一遍。普通领域水合永不自动打开另一个前端的
   chat，且当精确目标未解析时，任何后台工作都不得激活另一个 chat 或推进其 navigation tick；
6. 应用每个已解析层级，包括 main 本身，然后只在**成功的权威缺失**之后规范化 id：已完成的会话列表缺少
   该 chat → 工作区，已完成的工作区列表缺少该工作区 → Project Home，已完成的 welcome 缺少该项目 →
   main。从一个缺失 chat 的回退不会抹掉已有的本地 placement 或本窗口的 attention；精确目标被消费之后，
   推导出的位置如实反映仍然被选中的内容。超时、断开、不可读响应或普通服务器错误都不能说明存在性：
   导航保持路由／目标不变，而数据加载器执行其标准的重试／重连或错误 UI。

每条传入路由推进一个单调的恢复代次；每次异步继续都会检查它。任何项目／工作区范围移动**或中心导航
tick**都会取消一次仍在待定的权威读取，因此一次同工作区的文件／chat 点击也能胜过其迟到的应答。精确
chat 目标只在其工作区与所盖的 navigation tick 仍然最新时才能聚焦。当一条传入的精确路由未解析时，
store→驱动器的写入暂停，因此临时的“工作区／无标签／错误”状态无法抹掉 chat fragment。重复的
初始化／welcome 投递与 React Strict Mode 都是幂等的；重连会重试未解析的意图，但不会重放一条已完成的
启动路由。

## 平台适配器（Platform adapters）

Electrobun 按 `{ backendProfileId, windowId }` 在 webview 存储之外持久化一个有界 fragment，然后在启动
动态端口的本地宿主之后把它追加到真实源上。它的 preload 在应用代码之前就观察浏览器驱动器的
`history.replaceState`／`pushState` 以及初始／hash／pop 变化；web 编解码器仍是权威，无效的已恢复意图
经普通恢复路径规范化。这条原生桥接永不暴露宿主／领域能力。共享的后端选择仍被推迟。

移动端之后按后端 profile／设备持久化同一路由，并把通用／自定义链接映射到它上面。这些平台适配器都不会
把路由挪进后端拥有的状态。
