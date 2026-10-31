# host —— 浏览器↔宿主的线上契约

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

线上契约与组合根：`Bun.serve` 的 HTTP+WS、静态 SPA 服务、WS 方法→处理器注册表、通道扇出，
以及两个启动器共用的进程启动包装。

## 边界（Boundary）

- **拥有：** `server.ts`（异步 `createServer` 先请 auth 启动 Central 制品监听并发布初始的当前 PI
  runtime，需要时回退为带闭合 `load-failed` 状态的普通 PI，然后用 `Bun.serve` 创建 `/health`、
  `/ws` 升级、一条 **`GET /files/<workspaceId>/<relpath>`** 路由——该路由在只对不超过 8 KiB 的头部
  做分类后（经 `fs` 的 `resolveWorktreeFile`——路径受限；坏 id/逃逸/缺失 → 404；
  `Cache-Control: no-store`，因为 worktree 会在该 URL 下移动），用 `Bun.file` 流出 worktree 文件的
  原始字节，从而让 markdown 查看器的相对 `<img>` 可以解析；与之配套的
  **`GET /blob/<workspaceId>/<oid>/<relpath>`** 路由在**单个提交或树**上提供该路径的字节
  （`git.readBlobStreamAtAsync`，前置一个 40/64 位十六进制的 `oid`——即 diff 范围的
  `resolvedOriginalOid`，于是该 URL 命名的是不可变内容，并回答
  `Cache-Control: public, max-age=31536000, immutable`；Git 原语要求对象是 blob，因此树/提交/gitlink
  与坏 id/oid/逃逸/缺失路径一样返回 404）。在打开 blob 响应体之前，该路由执行一次有界的
  `git cat-file -s`；超过 `BLOB_SIZE_LIMIT`（64 MiB——评审界面一次渲染的最大图片、PDF 或 notebook
  的大小，也是一个 `change.revert` 回执最多可钉住的内容上限）的对象以 413 拒绝。被接受的 blob 是
  **流式**发送的：只有 8 KiB 嗅探头部会被 await 以生成响应头，其余部分逐块转发 `git cat-file` 的
  stdout，因此并发的图片或 PDF diff 消耗的是管道缓冲而非宿主内存中的 blob；响应体以 chunked 方式发出
  （流式响应体不带 `Content-Length`），中途断开的消费者会杀掉它的 `git`——进入流式后通过响应体的
  cancel，仍在等待嗅探头部时通过请求的 `signal`——转发自身的时限是五分钟，因此慢读者会把 `git`
  堵在管道上，而不会与 55 秒的网络预算赛跑。两条路由都存在，是因为 `fs.readFile` 与 `git.diffFile`
  对它们不得解码的字节回答 `""`，所以纯字节资源改由 HTTP 获取。两条路由都通过共享的字节分类器加
  文件名回退（最终回退到 `application/octet-stream`）得出 `Content-Type`，并发送
  `X-Content-Type-Options: nosniff`；活跃的同源类型（`text/html`、`application/xhtml+xml`、
  `image/svg+xml`）另外收到 `Content-Security-Policy: sandbox; default-src 'none'`，
  因此直接导航无法执行仓库脚本；静态服务并带 `index.html` 回退；`server.welcome` 推送；
  在升级时从 socket URL 读出的 **`?client=` 页面标识**（以 `RequestContext` 传给每个处理器；
  它寻址终端输出，但不再*拥有* PTY——见 [[submodule-server-terminal]]），以及
  `clientKey → socket` 注册表和**重放命名空间保留计时器**（它比一次重连活得更久，终端刻意为它所不及）；
  **请求重放缓存**以 `(clientKey, requestId)` 为键（第一帧拥有一个处理器 promise 及其序列化响应，
  重连重放会 await/返回同一结果，不匹配的重复请求被拒绝，回收客户端会清空其缓存——但**只在其上没有任何
  在飞请求之后**：未解决的请求会比 socket 的宽限期活得更久，因为页面会把那一帧持有到它*自身*的时限
  （文件夹选择器为 30 分钟）并在重连时重放，所以 `clearClient` 会拒绝回收而让回收重新计时，
  而不是让重放第二次执行一个尚未完成的处理器）。**该缓存中的任何内容都不会被驱逐**，
  因为一次成功的 `send` 只说明字节已入队，不说明页面读到了它们，而一个持着回复死掉的 socket
  与一个已把回复刷出的 socket 无法区分——所以任何由宿主自行决定丢弃的结果，都可能是重放正要索取的那一个。
  结果只在客户端自己发话时离开，通过这里处理、永不路由到处理器的两种帧：`{ ack: [id] }` 列出它已**读过**
  的响应（稳定状态），每次重连的 `{ resume: [id] }` 列出它仍视为**未解决**的一切，从而释放其他所有
  已结算的结果。`resume` 正是让回执可以丢失的东西——ack 会和响应一样死在 socket 缓冲里，
  永远不会有谁重发它，所以每次重连都重述完整真相，而不是去确认那些确认。代价改由**两个硬上限**约束，
  每个都在其大小变得可知的地方执行：*入*向的条目数——满命名空间会拒绝新 id
  （`RequestReplayOverflowError` → 正常的 `ok: false`），同时仍回答它持有的每个 id——以及处理器*出*向的
  保留字节数，因为响应大小在准入时不可知（`fs.readFile` 返回整个文件）而在飞的工作不占份量，
  所以准入时的字节检查只会约束计数，别的什么也约束不了。会突破字节预算的结果不被保留：
  条目仍留作该工作确实运行过的证据，因此它的重放失败（`RequestReplayUnretainedError`）而不会重新执行，
  已经发给调用方的那个响应不受影响。两个上限都不会牺牲 exactly-once——一个拒绝尚未开始的工作，
  另一个保留已完成工作的记录而只丢弃其答案），
  **`provider.login`** 通道发布（`auth` 模块的无会话登录帧桥，像 `pi.extensionUi` 一样接好），
  **`provider.changed`** 失效广播——在 auth 改变 Central 状态或当前 runtime 代数之后、在每次成功的
  应用内 provider 登录（不是 MCP 登录）之后、以及在每次 `provider.logout` 之后发出，让其他客户端与
  Welcome 界面看到新的 provider（客户端会重读状态/模型），以及 `provider.*` 处理器——包括
  `provider.jbcentralQuota`，其处理器把 `settings` 的 enabled/interval 值与 `auth` 的闭合缓存读组合起来，
  以便兄弟模块之间从不互相 import——**`watch` 接线**（把 `workspace.fsChanged` 发布回调注入 `watch`，
  并注入 `agent` 的项目 skill 路径分类器，使每个有上限的批次都携带独立的
  `skillChange: none|detected|unknown` 证据；把 **`workspace.watchReady`** 暴露为类型化预检，
  它在一个 web skill 加载流程捕获基线之前 await 新监听器保守的启动轻推，并报告监听器是否已被认作就绪
  （客户端可重放安全的保守回退；其可选的 `prewarm` 标志被转发进 `watch` 有界的仅预热层级，
  因此预选预热永不无界增长监听器注册表）；加上 **repo-metadata** 回调（`setRepoMetaPublisher`），
  在被监听 worktree 中一次 git 元数据写入时扇出到**三个**汇合点：`recordWorkspaceHead`
  （已结算生命周期的提交代理——只在 sha 真正移动时打时间戳，见
  [[submodule-server-workspaces]]）、`refreshUserOwnedWorkspace`（**重新同步用户自有工作区的
  folder-truth 分支**——由宿主中介，因为 `watch` 没有 `workspaces` 边，并通过工作区生命周期 tee 自发布）
  **以及**一个无路径、与 skill 无关的 `fsChanged` 帧（`paths: []`、`truncated: false`、
  `skillChange: "none"`），使客户端相对 `HEAD` 的读取（`git.status`、`uncommitted` 范围的 diff 标签页）
  在终端 `commit`/`reset` 移动 ref 时重读；同一次发布还喂给 **fsNudge 接缝**（`fsNudge.ts`：
  `setFsNudgePublisher` + `nudgeBaseRefWorkspaces`），这是 `git.prefetch` 处理器在应用自身后台
  fetch **移动**了某远程跟踪 ref 时触发的宿主中介——这种写入只有项目仓库共享的 `.git` 能看到，
  对每个 worktree 监听器都不可见——把无路径帧发给该项目中 diff 基准正是被移动 ref 的每个工作区
  （它们分支范围 merge-base 可能已移动——没移动时重读是幂等的；其他所有工作区保持沉睡），
  且不触碰任何 worktree 文件；从工作区读取类处理器（`fs.*`、`git.status`/`git.diffFile`、`spec.graph`）
  调用 `ensureWatch(workspaceId)`——读就是“有客户端在看”的信号；在 `workspace.remove` 的快速路径中
  于 `evictSpecIndex` 旁边调用 `stopWatch`；在 `stop()` 中调用 `stopAllWatches()`），
  在 socket 关闭之前于 `stop()` 中调用 `stopJbcentralRuntime()` 与 `cancelAllLogins()`，
  一个可选的启动期 `openProject(projectPath)`（尽力而为——启动器便利），
  **分析接线**（启动时以启动器来源、目的地和每次运行的附加数据抑制来初始化 `initializeAnalytics`；
  启动只在 `analyticsEnabled && analyticsConsentConfirmed` 时授予附加层。启动之后，settings 发布器
  仍广播每个合并后的配置，但只在其成功应用的更新显式携带 `analyticsEnabled` 时改变分析授权，
  因此无关写入会保留当前授权，而对话框的偏好预热可以启用它。`analyticsConsentConfirmed` 控制
  web 提示的生命周期。其未确认状态下的 on-prime 可以启用普通的附加采集，但浏览器归因只从
  confirmed-on 状态开始：直接在某次最终应用的对话框更新之后（UI 已就绪），或对于已保存的选择，
  只在启动器于正常 UI 就绪后显式发出 `RunningServer.startAttributionClaim()` 就绪信号之后。
  撤销会取消该 claim 代次。人类运行中基础事件保持开启。同意状态变更会清空附加事件关联，
  使重新启用无法重构同意之前的工作；仅活动型富集在关闭时无效，开启时从其严格的服务端记录恢复，
  且不重试已消耗的 claim。`shutdownAnalytics()` 仍是 `stop()` 中的尽力排空，并被优雅关停 await；
  每个采集点都在这里，包括已有的基础事件：`session.create` 中的 `chat_started`（对一个没有可用模型的
  聊天，宿主的 new-chat 解析判定为 `none`，而不是 pi 的占位符；否则报告所创建会话的实际模型）、
  `message_sent`（通过本地 `trackSend(mode, text)`）在一次**被接受**的
  `session.prompt`/`session.steer`/`session.followUp` 之后（`prompt`/`steer`/`follow_up`；
  当 contracts 的 `isControlMessage(text)` 命中时跳过——客户端的 TODO 唤醒轻推走同一批方法，
  并不是用户消息；`session.respondToTool` 是工具回复，也不是消息）、
  `provider_login`，取自登录发布 tee 的终态 `success` 帧，方法（`oauth`/`api-key`）从
  `loginAnalytics.ts` 查出——即 `provider.loginStart` 处理器记录的 loginId→method 映射
  （并由 `provider.loginCancel` 清除；未知 loginId 不跟踪任何东西，fail closed）——
  加上一次成功的 `provider.jbcentralConnect`→`applied`（失败的动作永不计数）——按
  `submodule-server-analytics`，特性模块从不自行跟踪。基础登录/聊天/发送事件从其自身的会话/登录代次
  附加闭合的 auth 分类，永不取自更晚的全局 runtime；发送在派发前快照 provider/auth。
  不透明加载器的 provider 成员关系识别 Central，而不打开其 auth/配置面。
  附加的 setup/run/task/review/PR 观测使用 [[submodule-server-analytics]] 中的闭合触发器，
  并带瞬态的同意范围关联与任务制品对账。只有宿主中介这些事件；分析初始化会发出打包安装生命周期事件，
  而 provider 变更采集并不存在。
  Setup 只观测已有的读取结果，永不触发 provider 工作；只有显式的 setup 变更才计数
  （`dialog.selectDirectory` 是其中之一：路径为 null 时以 `cancelled` 记录 `directory_pick`；
  `project.open` 把 projects 模块的 `NOT_GIT` / `ALREADY_OPEN` 编码错误映射到闭合原因，其他一切为
  `unknown`）。运行计时从规范的 `agent_start` 开始，本地发送意图在调用 pi 之前记录（不是
  `ackSend` 之后）；未经证明的来源保持未知，重试保持一个周期直到 `agent_settled`。
  被接受的排队来源在发送解析之后仍存续，直到规范周期观测；被拒绝的发送不贡献。
  同意清空会保留授予前的周期标记，而结算、队列清空、会话删除和完整宿主重置则释放其对应的瞬态意图状态。
  任务观测在具备完成能力的变更之前，先通过既有的无 pi 依赖的 `TodoStore` / `groupStatus` 边界
  同步读取归约组状态，然后在 reconciliation 之后读取制品；线上编辑会等待已有制品工作，
  而不是为遥测新建 Git 写入。完成证据与去重等待工作区既有 reconciliation 队列收敛，
  包括 plan 漂移后的替换轮次，并保留该变更原始的同意授予。
  空/刚 hydrate 的组以及通过删除完成的组不计数。这些仅内存的观测在异步工作之前保留某次授予的采集，
  且永不跨撤销回放。
  `stop()` → 立即清理 agent 会话，然后在 `closeAllTerminals()` **之前**
  `persistTerminalSessions()`，然后释放监听器/socket；`shutdown()` 记忆化一条异步优雅路径：
  先有界的 `settleSessionsForShutdown()` + await 的 `shutdownAnalytics()`，然后
  `turnTracker.drain()` 等待那些结算所启动的回执，然后 `stop()`）。有界结算包含隐藏的委派子会话，
  即使其父会话空闲，也包含并发移除已启动的子级级联，因此优雅退出不会让后台子级丢失其终端 abort/
  工具结果；`crashLog.ts`（`installCrashLog`——把 `uncaughtException`/`unhandledRejection` 报告
  追加到 `<dataDir>/logs/crash.log` 并回显到 stderr，然后——仅对第一个故障——await 一次 ≤ 1 秒的
  分析 `reportFatalError`（带上崩溃类别标记）再 `exit(1)`；该窗口内更晚的故障仍会追加，
  但既不重新上报也不重新退出：进程内 pi 意味着这种故障就是整个宿主的故障，而一个没有终端启动的
  启动器否则会丢掉它唯一的线索。永远不是恢复，且永不安装在 `NODE_ENV=test` 下——单元测试进程
  自己上报自己的故障。它通过 `log` 模块的 `describeError` 渲染抛出内容，使崩溃报告与日志行一致，
  但保留自己的同步追加——死亡路径不得依赖日志器的状态）；
  `boot.ts`（`bootHost` → await `initLogging`——启动器传入 `verbose` 时为 debug 级别——然后安装
  崩溃报告，启动器传入 `resolveShellEnv: true` 时 await 登录 shell 环境（在 `initLogging` 之前启动），
  应用已存储的宿主环境（`applyHostEnvironment`，在任何 `getAgentDir()` 读取者之前），
  在选择服务端口之前预热同一套 Central 监听器/runtime 初始化，await `createServer`
  （它为底层嵌入者幂等地强制 runtime 引导），并写入 `listening on` 信息行（见
  `submodule-server-log`）；`reportDesktopError(error, scope)` 是桌面启动器把已处理错误送进分析的
  入口（来源 `desktop`）。它的 SIGINT/SIGTERM 处理器在进程退出前 await 同一个 shutdown。
  结算会中止流式会话并有界等待，使 pi 持久化它们的 "Operation aborted" 工具结果且转录成对落地，
  但阻塞在 `ask_user_question` 上的会话例外：shutdown 刻意让该调用悬空，下一次 attach 会把它修复为
  可回答的 ack；显式用户 Stop 仍是终端 abort 路径）；
  `handlers.ts`（WS 方法→处理器注册表，类型化的
  `WsHandlers = { [M in WsMethodName]: WsHandler<M> }`，因此缺失、多余或类型错误的处理器都会让
  typecheck 失败；`handleRequest` 在派发时一次性收窄线上的 `unknown` 参数，而参数不在运行时做形状校验。
  注册表包含 `workspace.rename` 作为进入 `renameWorkspace(id, name)` 的直接手动入口
  （没有 `branch` 选项）——workspaces 模块只改显示标签、持久化并发布，因此该处理器从不改 Git、
  不单独 emit、不单独 patch 客户端；`project.rename` 同样委派给 projects 模块的 `renameProject`，
  由它拥有校验与推送。`client.reportError` 把不可信的浏览器报告交给分析的 `reportClientError`
  （唯一的净化器）并总是 ack。错误跟踪还走 `server.ts` 的两处接缝：每个失败的 WS 请求都连同其
  已注册方法上报（含 CodedError），而 `observeLoggedErrors` 把宿主的 `log.warn/error(msg, err)`
  调用路由为 `host_logged`——因此携带错误的宿主诊断走日志器，永不走 `console.*`。
  宿主的 `resolveNewChatModel` 把 AppConfig 设置与 agent 已结算的可用模型列表及 Pi thinking 夹取
  组合起来；`model.default`、`session.create` 与新建的 review 聊天共享该解析器，且创建时显式传入
  其模型与 effort。一次**指明了**模型的 `session.create` 以及每次 `session.setModel` 也通过
  `settings` 的 `noteRecentModel` 把解析出的模型记为 recent——由默认解析出的创建是宿主的选择
  而非用户的，因此不予记录。`model.contextSettings` 与 `model.setContextWindow` 委派给 agent 的
  `modelContext` 适配器并返回其 `ModelContextSetting[]`；该变更命名 provider/model 目标或所有合格配对，
  永不命名任意模型元数据，成功保存复用 `provider.changed` 使各客户端目录失效。既有 review 聊天与
  plan-review 子 agent 保留各自的策略——以及 **Skills 管理器集合**：`skill.list` / `skills.state` /
  `project.skills` 从 `projects`（工作区范围内时再加该工作区的 `skillOverrides`）构建准入上下文，
  并传入 agent 的 `listSkillCommands`/`listSkillCatalog`；**MCP 集合**（`mcp.*`、`host/mcp.ts`）
  把 `agent` 的 MCP 服务与 `projects`（批准、覆盖、pi 级信任）组合起来，发布 `mcp.status` 快照
  （只有 `MCP_PROTOCOL_VERSION` 或更新版本的客户端订阅该 topic）以及 `session.resourcesChanged`
  失效。每个请求解析其工作区与该工作区的项目。管理变更与探测在异步所有权解析之后重读二者，
  然后在同一个同步调用栈里检查所有权并准入工作。项目身份、cwd 或 pi 级信任发生改变会拒绝该请求以便
  重试；批准与覆盖取自最新记录，而不是 await 之前的快照。项目覆盖补丁针对最新存储的覆盖原子合并。
  线上启用值必须是布尔值，记录暴露必须是 `deferred`、`direct` 或 `hidden`；非法值在记录或按聊天
  禁用集变更之前被拒绝。没有写入会批准用户尚未评审的仓库内容：项目文件中的 `mcp.add`
  （文件尚未定义的名字）就是用户自己的条目，因此其指纹在同一请求中被批准；`mcp.update` 与
  `mcp.remove` 在该范围自己的文件（用户范围为全局 `mcp.json`，项目范围为 worktree 的
  `.pi/mcp.json`）中，与写入处于同一同步步骤读取磁盘条目的指纹，并在请求的 `expectedFingerprint`
  与之不同时拒绝且不动文件（任一范围内缺失指纹都被拒绝）；只有当记录曾批准该磁盘指纹时，
  `mcp.update` 才把批准带到写入的指纹——对待处理或已变更条目的重写仍保持待处理
  （从 checkout 的新条目重建的开关不得批准它）；来自仓库的编辑保持待处理；`mcp.approve` 拒绝
  已不再是磁盘上那个的指纹；`mcp.shareWithRepo` 在项目文件已定义该名字、或持有未在其当前指纹上
  被批准的覆盖时拒绝，否则把记录覆盖写为 pi 的覆盖条目、批准它并清除记录副本。
  `mcp.revokeToolGrant` 从记录中移除一个始终允许的工具且不重载任何聊天：调用守卫实时读取授权。
  变更之后，受影响的活会话（用户文件为全部，项目文件与记录变更为该项目的）被对账——只有其有效配置
  确实改变的才通过门禁重载。该对账是启动而非 await（失败会记日志）：agent 逐个重载空闲聊天，
  因此请求立刻用新列表作答，而 `mcp.status` 推送报告每次重载。`createServer` 启动 agent 的
  `watchUserMcpConfig`（随服务器停止）并在某工作区的变更轻推触及 `.pi` 或被截断时对账该工作区的
  活聊天。按聊天调用（`setSessionOverride`、`reconnect`）要求会话属于被命名的工作区；
  `mcp.reconnect` 运行 agent 的 `reconnectMcpServer`（pi 的通知不进聊天；pi 的失败被遮蔽后成为该请求
  的错误）并无论如何刷新该聊天的状态；`mcp.readOutput` 读取 agent 的 `readMcpToolOutput`，
  且仅限于该工作区的会话；`mcp.readLog` 解析工作区、按 pi 的字符集检查名字并返回 agent 的
  `readMcpServerLog`（pi 每个 agent 目录保留一个 `mcp.log`，因此不需要会话）；
  `mcp.login` / `mcp.testConnection` 以调用方的 `clientKey` 为属主启动 agent 的探测并返回其
  `loginId`（`mcp.logout` await 该探测）。探测准入用当前信任与策略从 agent 的宿主配置加载器解析出
  有效的有效服务器；缺失或非法目标被拒绝，而 Test connection 要求该准入配置为 HTTP。
  一个已被批准但非法的项目条目绝不会仅因其摘要看起来已批准就遮蔽有效的用户目标。三者在
  `mcp.list` 会报告 `handledElsewhere` 的任何地方都以 `MCP_HANDLED_ELSEWHERE` 拒绝
  （该探测的加载器强制 `builtin:mcp`）；`provider.loginReply` / `provider.loginCancel` 把
  `mcplogin_*` id 连同调用方的 `clientKey` 路由到该探测，而 `createServer` 只把探测帧发给属主的
  socket，在客户端的保留计时器最终回收它时取消该客户端的探测（`cancelMcpProbesOwnedBy`，
  位于 `releaseInvitation` 旁边；绝不在仅仅 socket 关闭时取消——窗口内的重连会保留该 key 与其登录）
  并在停止时取消每个探测——`mcp.login` / `mcp.testConnection` 的推送只发给发起连接
  （`ws.send`，不是广播 topic），外来回复以 `LOGIN_NOT_OWNER` 失败；`session.list` 用
  `openTodos: countOpenTodos(…)` 为 agent 的 `listSessions` 摘要逐会话装饰（`agent` + `todos` 的
  宿主专属组合——`agent` 保持无 todos 依赖；计数失败则省略该字段，永不使列表失败）；
  **`todo.requestFix`** 是同类组合（`todos` 记录并渲染 fix 包，`agent` 投递）：该包以**分离**方式
  触发到该项自己的聊天，作为**结构化的 `todo-review-fix` 自定义消息**——`sendReviewFixToSession`
  （`fireTodoFixPrompt`）调用 `AgentSession.sendCustomMessage`，把渲染后的包文本作为 `content`
  （agent 读到的东西），把 `ReviewFixDetails`（`buildReviewFixDetails`：条目 id/标题、反馈备注，
  以及精简的路径/行号解析后的 finding）作为 `details`（聊天卡片渲染的东西），并带
  `deliverAs: "followUp"` + `triggerTurn`——**不是**合成的用户轮次（#363）。像旧的
  `followUpSession` 路径一样包在 `ackSend` 中，因此轮次前的拒绝会回滚 review 记录
  （`rollbackTodoFix`）并作为扩展 UI 通知浮现，从而未投递的 fix 请求永不搁浅为 `changes_requested`。
  手动 fix 包**携带该项未关闭的 agent finding**，与自动周期完全一致（`itemFixFindings`——
  按 `origin` 取该项未失效的 agent 撰写草稿，`markCommentsSent` + `buildSendPackage` 在
  `withReviewLock` 下）：auto-fix 关闭时，裁决路径什么都不发，因此没有这一点 worker 就永远看不到
  评审人的 finding，它们会搁浅为草稿直到后来某次 approve。包渲染发生在这些 finding 被标记为已发送
  之前；每个准备失败都会做身份校验并精确回滚该请求写入的 `changes_requested` 记录，以及任何标记，
  因此它既不让失败请求搁浅，也不覆盖更新的决策。同样的身份安全补偿也在分离的轮次前拒绝时运行；
  手动或自动 fix 在其第一次 await 之前认领一个内存中的按条目闸锁（`claimItemFix`），并在整个包准备与
  fix 发送接受/失败期间持有它，因此重叠的手动发送会被拒绝。`isItemUnderActiveReview(sessionId, id)`
  把该闸锁与 `planReviewQueue` 的活动集一并暴露给任何需要知道某条目是否有 review 在飞的调用方。
  **`todo.startReview` / `todo.reviewAll` + `host/requestReview.ts`** 组合 agent 评审人。一个 plan
  步骤由该 plan 会话的**隐藏、短暂委派子会话**评审（`agent` 的 `runReviewSubagent`），携带我们的
  评审人角色——`host/reviewerRole.ts` 拥有系统 prompt、只读工具集与围栏 JSON 输出契约；`todos`
  渲染的 review 包只是变更集**引用**，永不点名工具。子会话返回最终文本，宿主解析结构化裁决
  （`parseVerdict`，对 finding 宽松、对裁决词严格）并拥有每个状态迁移。没有评审人聊天，
  没有评审人撰写的工具（`review_verdict` / `add_review_comment` / `reflect_finding` 随之消失），
  因此也没有需要监控、注册或解卡的评审人会话——它所需的那整套崩溃恢复面塌缩为一个被 await 的
  promise：出错、被中止或返回不可解析文本的子会话会 reject，而 `catch` 当场清除该项的 `reviewing`
  标记（`cancelTodoReview`）。**两个入口，一条记录路径。** worker 自己的 `request_review` 工具
  （`handleRequestReview`）await 裁决并将其作为工具结果返回——worker 读取 `composeText` 并就地修复。
  Start review / Review All 按钮（`startPlanReview`）**同步**把该项标记为 `reviewing`
  （因此客户端一重读 plan，面板就显示脉冲）并通过记录 + Review 标签页投递结果。**两者**都在 plan 的
  串行链上运行 review 主体（`planReviewQueue.onPlanChain`——按钮路径 fire-and-forget，工具路径
  await），因此工具请求永不与同一 plan 上另一步的按钮 review 重叠。两者都汇入 `recordVerdict`：
  `approve` 结算该项（`approveTodoReview(…, "agent")`）；`request_changes` 把每个 finding 归档到
  Review 标签页（`fileFinding`——当 `reviews.anchorProblem` 接受该位置时为行内评论，否则为
  review 级；锚点解析是尽力而为的，因此坏锚点回退为 review 级评论，但存储写入失败会传播并取消本次
  review，而不是静默丢弃该 finding 却仍消耗该周期），然后花费 fix 预算。**1 周期上限在两条路径上
  相同。** `canAutoFix = reviewAutoFix !== false && spent < 1`（`todoReviewAutoCycles`），
  记录以 `autoCycles: canAutoFix ? 1 : 2` 写入——`1` 表示“worker 确实被要求修复，该项处于周期中”，
  `2` 是终态（“现在由人决定”）。记录 `1` 却不要求任何人修复会让该项搁浅：`maybeAutoReReview`
  的触发器正是读这个值，而没有任何东西会产出它等待的新 delta。**只有 worker 接受该 fix 之后才消耗
  一个周期。** 在按钮路径上，`deliverFixToWorker` 拥有整个临界区——归档 finding、记录乐观的周期 `1`、
  选择并标记它们 `sent`、然后发送——从归档到标记被持有在同一个 `withReviewLock` 中，因此任何插入的
  Review 发送都无法在预留之前抢走刚归档的草稿。失败按归档是否完成而分流：*归档*失败在任何记录之前
  抛出，因此本次 review 取消（`fileFindings` 已补偿其部分持久化），而任何*归档后*失败——发送被拒
  （worker 已分离、忙碌、轮次前拒绝）**或任何准备失败**（快照/包/标记）——在乐观的 `1` 之上把该项
  重新记录为终态（`autoCycles: 2`），同时用 `rollbackSend` 把任何已标记的 finding 归还为 `draft`。
  归档后失败若以抛出逃逸，会让该项停留在 `autoCycles: 1`——这正是统一 catch 的全部意义。
  把它留在 `1` 会让该步骤永久搁浅：没有任何人要求 worker 改任何东西，因此不会有新 delta 到达
  `maybeAutoReReview`，而之后的手动 review 会把这个周期读作已消耗并拒绝发送。fix 闸锁拒绝的认领
  （一次手动 Ask-to-fix 已在飞）也以同样方式结算。在按钮路径上，活预算还会**把 fix 投递到 worker
  聊天**（`deliverFixToWorker`）：该项的 origin 范围草稿 finding（`itemFixFindings`——该项、
  该 worker 会话、未失效；无范围的清扫会把其他步骤的 finding 带进这个 worker 并使它们错误地显示为
  已发送）用 `renderFixPackage` + `buildSendPackage` 渲染，并在 `ackSend` 下作为结构化
  `todo-review-fix` 消息发送，轮次前拒绝时由 `rollbackSend` 把它们归还为 `draft`。工具路径从不发送
  ——worker 已在工具结果中有裁决，再作为消息发一份会重复指令。预算已耗尽或 auto-fix 关闭时，
  两条路径都把 finding 留在 Review 标签页给用户，并说明此事（`composeText`）。**每个 plan 一次只允许
  一个 review**（`host/planReviewQueue.ts`）：`onPlanChain` 把每次运行串到该 plan 的 promise 上——
  `enqueuePlanReview` 为分离的按钮路径使用它，`handleRequestReview` 为工具路径 await 它——
  因此 Review All 会启动 N 个步骤但串行运行它们，而工具请求排在针对不同步骤的在飞按钮 review 之后。
  该链存在就是为了防止 N 个并发 provider 流。同一模块持有两个入口都检查的按条目认领，
  因此已在 review 中的步骤被拒绝，而不是被评审两次、两个裁决竞相写进同一记录。`todo.reviewAll`
  报告 `{ total }`——它实际启动的数目——并只在它什么也没启动而该 plan 的链仍忙碌时报告
  `alreadyRunning`。该模块是纯机制，带注入的 runner（无 agent 依赖），这也是
  `planReview.test.ts` 在没有 provider 的情况下驱动整条裁决 → 记录 → 投递路径的方式。
  **自动重审**（`maybeAutoReReview`，从 `isTodoToolEnd` 上的 reconciliation 钩子 tee 出来）：
  在一次 fix 落地且 worker 重新把该步骤标为 done 之后，恰好仍在其唯一自动周期内的条目
  （`autoCycles === 1`、非 `reviewing`、`status: "done"`）会被重审。触发器接受
  `changes_requested` 记录上的新提交 delta **或**记录被重置为 `unreviewed`——后者是路径列表回退
  （`todos/artifacts.ts` 在重做无法提交时丢弃记录），此时残留的已消耗周期本身就是“一次 fix 已落地”
  的信号，因为没有 sha 可供做水位标记。符合条件的条目**即使另一个 review 已在运行**也会被入队到该
  plan 的串行链上（第二次 fix 落在 Review All 正在评审另一步的时候），而按条目认领会去重——链忙碌时
  它绝不能被丢弃，否则一个在 review 中途修好的步骤会永远停在 `autoCycles: 1` 的
  `changes_requested`，因为链排空后没有任何东西会重试。**`approve` 永不结算仍有未关闭 finding 的
  条目。** fix 周期的两轮是独立运行，因此没有任何结构性东西把第 2 轮的 approve 与第 1 轮的 finding
  连起来：门禁是显式的，即在 `approveTodoReview` 之前检查 `itemOpenFindings`。它的集合刻意比 fix
  候选**更宽**——`itemFixFindings` 只含 `draft`（`sent` 的 finding 不得搭上第二次 fix 请求），
  而 review 模型把 **`draft` 与 `sent` 都**算作未解决，因为一个修好代码却没有调用
  `resolve_comment` 的 worker 让该 finding 保持未关闭。没有该门禁，plan 会读作可发布，
  Open PR 会为一条 Review 面板仍显示为阻塞的评论亮起。**反向也成立：`changes_requested` 裁决不得
  比其 finding 活得更久：** 关闭 finding 的两条人工路径都通过 `clearChangesRequestedIfResolved` 对账
  ——`review.commentDelete`（删除）与 `review.commentUpdate`（解决/忽略）。当被关闭的 finding 是该条目
  **最后一个**未关闭项（`itemOpenFindings` 为空）且该条目仍是 `changes_requested` 时，其裁决记录被丢弃
  （`dropTodoReviewVerdict`）回到 `unreviewed`——**只有记录**：在飞 review 的 pending 标记（其启动
  SHA 水位）与自动周期计数都保留，因此在 review/fix 中途关闭最后一个 finding 既不会给评审人从未见过的
  提交打水位，也不会重置单周期上限；任何 finding 仍开着时该检查是空操作，因此一个从未有过行内 finding
  的整体变更裁决永不被动。agent 的 `resolve_comment` 路径（`applyAgentResolution`）刻意不在这里对账
  ——它落在 fix 周期内，其随后的重审会重新推导裁决，而且记录在两种情况下都是惰性的（Open-PR 门禁数的
  是未关闭 finding，不是记录）。该 plan 的 review 装饰由该记录派生，`apps/web` 的 `useChatTodos`
  在任何 review 快照变更时重读 plan。被阻塞的 approve 清除 `reviewing` 标记、不动记录、告诉 worker
  对它已处理的调用 `resolve_comment`（`composeText` 的第三种形式）并作为
  `PlanReviewResult.blockedByOpenFindings` 走线上，使卡片无法声称该步骤已完成。`resolve_comment`
  仍是 WORKER 的工具——`reviews.applyAgentResolution` 只解析 `sent` 评论，且只在调用会话等于
  `markCommentsSent` 记录的投递目标聊天时；`project.trustSummary` 用 agent 的
  `projectTrustSummary(project.path)` 回答信任通知；`project.setTrust` 在授予别名信任时确认当时存在的
  别名（agent 的 `listProjectAliasSkillNames`；对已获别名信任的项目（pi 级授予）再次授予会保留已确认
  集合，因此此后到达的别名仍待评审），设置别名授予，只在请求携带 `resources: true` 时授予 pi 级信任
  （撤销会同时清除二者；较老客户端的授予仅为别名），并把该项目的各工作区交给 agent 的
  `applyPiResourceTrust`，使活聊天重载进新信任（空闲者立即、忙碌者在结算时）；`createServer` 把
  `setMcpPolicyResolver`（工作区 → 其项目的 `mcpApprovals` / `mcpOverrides` / `mcpToolGrants`，
  fail-closed 为无）与 `setMcpToolGrantWriter`（“Always allow” 回答 → 工作区项目上的
  `grantProjectMcpTool`）接在准入解析器旁边，并把 agent 的 `piTrustsProjectPath` 注入为 projects 的
  信任种子（`setPiTrustSeed`），使新加的项目从 pi 自己的决定开始；`project.acknowledgeSkills` /
  `project.setSkillEnabled` / `project.setGroupEnabled` / `project.aliasSkills` /
  `workspace.setSkillOverride` 变更/读取持久化的开关；`session.reloadResources` 重新扫描一个运行中的
  会话——组合留在这里；`agent` 从不 import 其兄弟模块。`createServer` 还接线
  **`setSkillAdmissionResolver`**，把会话的 `workspaceId` 映射到其项目的信任/已确认/已禁用加上该工作区
  的覆盖（fail-closed），因此 `agent` 无需 import `projects`/`workspaces` 即可门禁 skill）。
  兄弟的 subagent 策略以同样方式在这里组合：`createServer` 接线 `setSubagentsEnabledResolver`，
  存在显式 `Workspace.subagentsOverride` 时映射它，否则使用 `AppConfig.subagentsEnabled`
  （未知工作区 fail closed），settings 发布器在携带 `subagentsEnabled` 的全局更新之后请求
  `refreshSubagentTools()` 重新评估所有活会话（同理 `refreshAgentReviewTool()` 对应
  `agentReviewEnabled`、`refreshSubagentConcurrency()` 对应 `subagentMaxConcurrent`、
  `refreshExtensionTools()` 对应 `disabledExtensions`）——不是每次发布之后，因为 `recentModels`
  现在每次模型切换都会发布，而每个空闲会话重建工具集会纯属抖动——并且
  `workspace.setSubagentsOverride` 通过 `workspaces` 持久化后只刷新该工作区。按聊天的 subagent 上限
  以完全相同方式组合：`setSubagentConcurrencyResolver` 映射有效的
  `Workspace.subagentMaxConcurrentOverride`，否则有效的 `AppConfig.subagentMaxConcurrent`，
  否则 contracts 默认值（`resolveSubagentMaxConcurrent`；未知工作区使用全局值——上限没有 fail-closed
  的含义），而 `workspace.setSubagentMaxConcurrent` 持久化后只调整该工作区的服务大小。扩展开关是
  全局的：`setDisabledExtensionsResolver` 把 `AppConfig.disabledExtensions` 作为集合返回
  （未知名字是惰性的），而 `extension.list` 用该同一设置连接 `extensions` 目录作答——开关本身经
  `settings.update` 写入，因此没有扩展写入方法。两个权威发布器仍是客户端的收敛路径；宿主到 agent 的
  刷新改变的是运行时能力，不是前端状态；**Review 标记是内存加磁盘，且只有磁盘那一半能跨越重启。**
  串行链与按条目闸锁都在内存中；`pending` 标记是磁盘 sidecar。`createServer` 在服务器接受连接之前
  调用一次 **`reconcilePendingReviewsOnBoot`**：它遍历每个项目的每个工作区并调用 `todos` 的
  `clearAllPendingReviews(worktreePath)`，后者清扫每个会话的 sidecar 并无条件丢弃每个 `pending` 条目
  ——这是安全的，因为全新进程里尚无任何东西入队，所以发现的每个标记必然早于它。没有这一步，
  上次关停时在飞的 review 会永远转 `Reviewing…`，而 Review All 会永远跳过该项（它自己的
  `reviewing !== true` 过滤）；
  `ackSend.ts`（发送 ack 策略——见“必须做对的点”）；`titleTool.ts`（**`set_title` 宿主**，
  通过 agent 的 `setTitleToolHost` 安装——只有宿主可以做的 `agent` + `workspaces` 组合）。
  主 agent 命名；宿主只执行写入策略（旧的主机驱动多轮——先取几个词，然后在首个结算轮次上用廉价模型
  润色，再在发送时用廉价模型生成聊天标题——总是产出用户想改掉的名字）：
  - 调用会话必须是活的受管会话（`getSessionWorkspaceId`）；其他任何东西——例如子 agent 子会话——
    得到错误结果。
  - `chat_title` → `renameSession(…, { onlyIfUnnamed: true })`。`workspace_name` → 仅当工作区受管
    （不是 `default`/`external`）且未 `renamed` 时，经 `renameWorkspace(id, name, { branch })`
    （锁住它，移动分支一次）。没有分支形状门禁，因此旧宿主临时轮次留下的未锁定记录仍可命名。
  - 结果文本按目标说明它是被应用还是因已有名字而保留，使 agent 停止。没有可用 `workspace_name`
    的 `branch` 在任何写入之前被拒绝（否则 slug 会被静默丢弃而 agent 还以为已完成）。
    在工作区仍可命名时，没有可用 `workspace_name` 的调用以同样方式被拒绝：一个要求 agent “再调一次”
    的成功结果在实机被忽略了（Opus 命名了聊天，然后继续），而一次调用让聊天名与工作区名保持一致。
    首个名字是最终名，手动重命名总是获胜。
  - `workspaceNeedsName(sessionId)` 使用同一资格规则（受管、未 `renamed`），因此 agent 轮次开始的
    `pending-naming` 提醒与写入策略不会互相矛盾。
  - agent 从未命名的目标保持无名；没有回退。
  - **工作区归档拆除**——`agent` + `terminal` + `workspaces` 的另一处只有宿主可以做的组合。
    `workspace.remove` **在任何副作用之前大声拒绝 `kind: "default"` 工作区**
    （该记录的 `worktreePath` 就是项目文件夹——回收的 `rm -rf` 回退绝不能看到它；UI 隐藏了 Remove，
    这道守卫是给有 bug/失控的客户端准备的）。否则它会收割以该 worktree 为根的一切（对用户自有的
    `kind: "external"` 工作区，除 checkout 本身之外的一切），但它是**非阻塞**的：同步做快的那部分
    ——`forgetWorkspace`（丢弃记录 → 立刻从 `workspace.list` 消失）→ `evictSpecIndex`（丢弃 spec
    缓存）→ `closeWorkspaceTerminals`（杀掉它的 PTY）——**ack**，然后在**后台**运行慢的回收
    （`archiveTeardown`，fire-and-forget）：`removeWorkspaceSessions`（中止流式轮次、销毁活会话，
    **并**清除 pi 针对该 cwd 的磁盘转录）→ `reclaimWorktree`（`git worktree remove`；对 external
    是硬性空操作）。因此用户永不等待 git 子进程 + 会话中止。**顺序成立：** 终端（同步）与会话
    （后台，在回收之前）在目录被删除之前已关闭，因为它们以它为 cwd，且该工作区的 todo 变更队列
    在二者之间已结算（`settleChangeArtifacts`）——在飞 reconciliation 的 plan/基线写入在清扫它们的
    回收之前落地，永不落到它之后进入复活的目录。按契约是尽力而为——失败的后台拆除会记 warn 日志，
    永不抛进虚空（没有谁 await 它）。**归档保留分支但不保留聊天：** git 分支保留（代码可恢复），
    但聊天历史随 worktree 一起清除——这是刻意的范围选择，不是泄漏。
- **变更操作按工作区串行，走它们自己的链**（`reviewLock.ts` 的 `createKeyedLock` 铸造两者）：
  `change.revert`/`change.undo` 在 **`withChangeLock(workspaceId)`** 下运行，使两次 revert 无法交错
  其加载→校验→写入，而 review 与变更——彼此独立的资源——永不互相排队。agent 刻意**不**被暂停：
  `changes` 模块的 compare-and-swap 使竞态安全，且 fs 监听器的 `fsChanged` tick 在写入之后重读打开的
  标签页，与 agent 编辑之后完全一样（因此两个处理器都先 `ensureWatch`）。具名失败作为
  `WsResponse.errorCode` 传递
  （`STALE_VIEW`/`SCOPE_IMMUTABLE`/`RANGE_INVALID`/`RECEIPT_UNKNOWN`/`UNSUPPORTED_CHANGE`），
  走 `UNKNOWN_COMMIT` 使用的同一个 `CodedError` 映射——派发本身不命名任何码，因此在 `contracts` 中
  新增并由某特性抛出的码会到达客户端，无需更新宿主侧白名单。
- **Review 状态由宿主组合并按工作区串行**（`reviewLock.ts`）：`review.send*` 是 `reviews`
  （草稿 + 包）加 `agent`（会话）再加 `reviews`（标记已发送 + 链接）——一次跨
  `await createSession(…)` 的检查后标记，是 review 层唯一的非原子间隙。
  **`withReviewLock` 覆盖线上暴露的每个 review 变更，不只是发送**，因为有两样不同的东西落进该间隙：
  第二次*发送*读到同样的“有草稿、尚无会话”并把 review 分叉，而一次*变更*会让已构建的包失效——
  一个落在那里的 `review.close` Clear 会让包搁浅：标记看到一个全新的空 review 并把聊天链接到*那个*，
  留下 agent 永远无法 `resolve_comment` 的评论 id。每个工作区一个队列，因此发送中途发出的变更就发生在
  它之后。包 prompt 在标记之后以**分离**方式触发，因此锁只持有会话创建，而失败的操作释放它而不是
  毒化队列。plan-review 裁决路径加入同一把锁：被接受的 guide 持久化与 `deliverFixToWorker` 的
  归档→记录→选择→渲染→标记段落都留在锁内，因此 Clear 无法把 guide 与其 finding 分开，插入的发送
  也无法在这些阶段之间抢走候选 id。批准在同一把锁下检查未关闭 finding 并记录；被阻塞或失败的裁决
  不会替换 guide。刻意不加锁：`review.get`（其加载 → 重新锚定 → 持久化是一次同步过程，且 hydrate
  不得排在发送之后）——加上两个完全保持同步的变更：`reviews.resolveCommentFromAgent`
  （worker 工具接缝）与 `reanchorWorkspace`（fs 监听 tee）：二者都在写入之前从磁盘重读快照，
  且都不删除评论也不关闭 review，因此落在发送间隙内也无法使包的 id 失效。
- **一次 review 发送落在屏幕上已有的对话里，否则落在该键的聊天里。** 两个发送处理器都通过
  `sendToFileChat` 路由：评论按 `reviews.reviewSessionKey` 分组（锚点的路径，或无锚点评论的
  review 级桶——像文件一样被钉住，使第二条整体评论延续同一讨论），每组按优先顺序落在客户端的
  **最后打开的聊天**（发送携带的可选 `sessionId`——用户已在其中的对话），否则落在该键钉住的聊天
  （`reviews.fileReviewSession`），否则新建一个聊天；无论谁收到该包都成为该键的钉住目标
  （`markCommentsSent`），因此侧栏的“打开讨论”总是跟着评论走。**指名其聊天的发送是一个包**：
  当客户端传入 `sessionId`（某聊天卡片自己的聊天，或最后打开的聊天）时，每条评论都以一条结构化消息
  进入那个聊天，因为分组的存在是为了把每个键路由到它自己的聊天，而把按键的多个包发进同一聊天，
  会让 agent 把一次 review——一份前后对照 mockup 的两个文件——分成不同的轮次。
  **`review.sendBatch` 以它触及的每个会话作答**，按组顺序：跨两个文件的批次启动两个聊天，
  只命名第一个会让另一个在无人看见的情况下运行，而它的评论已经读作已发送（客户端会打开全部，
  聚焦第一个）。一个仅处于**分离**状态的已链接聊天被视为存在：它从持久化转录
  `agent.ensureSessionAttached` 并被 follow up 进去。review 状态与 pi 会话都能跨宿主重启存活，
  因此只按活度门禁（`hasSession`）意味着任何客户端未重新打开的 review 聊天都会得到*第二个*聊天
  和被覆盖的链接。只有该文件从未有过会话时才创建新会话——或者，作为显式恢复记录在日志中，
  当转录确实已从磁盘消失时（没有 UI 可以关闭 review 并重来，因此把它卡死会更糟）；其他每种重新打开
  失败都抛出，而不是静默分叉该对话。
- **聊天开始时的 scratch 目录播种：** `session.create` 处理器在创建会话之前调用 `workspaces` 的
  `ensureWorkspaceScratchDir`——Default 工作区被 gitignore 的 `.thinkrail/context/` 只在聊天真正
  在那里开始时落入用户仓库（而被删除的 worktree scratch 目录会自愈）。宿主组合——没有新的模块边。
- **Project 生命周期扇出：** `createServer` 安装 `projects` 模块的发布器，把每份权威的
  open/reopen/close 快照映射为 **`project.updated`**。WS `open` 处理器订阅该通道并在 `server.welcome`
  中 hydrate 两个视图：`projects`（仅打开的记录）与 `recentProjects`（所有已知记录，文件夹确实消失的
  关闭记录被省略）。`/ws` 升级先 await `refreshRecentProjectFolders` 最多 250 ms，因此无响应的挂载
  最多按该预算延迟一次连接，而未完成的检查让其记录保持在上一次完成的结果；`open` 本身从不 yield，
  因此 `server.welcome` 仍是 socket 的第一帧，两个视图都在发送它们的同一 tick 从当前注册表读出。
  这条单一全快照通道是幂等的，避免了分开的 opened/closed 流乱序重放。每个客户端由它收敛其 rail +
  Recents；只有发起方的打开流程选中 Project Home，而关闭回退仍是各客户端自己的视图状态。
- **Workspace 生命周期扇出：** `createServer` 安装 `workspaces` 模块的发布器
  （`setWorkspacePublisher`），把每个域事件 `kind` 映射到其 `WS_CHANNELS.workspace*` 通道
  （`created`/`updated` → 完整记录；`removed` → `{ projectId, id }`）并 `server.publish` 它。
  这是工作区成员关系变更到达线上的**唯一**地方——创建/重命名/归档都流经它，因此每个客户端
  （包括发起方）都靠反应收敛，从不靠各客户端的乐观。两个新通道在 WS `open` 处理器中与
  `workspace.updated` 一起被 `ws.subscribe`。
- **会话删除扇出：** `createServer` 安装 agent 模块的删除发布器，并在 `session.deleted` 上广播每个
  工作区范围的 `SessionDeletedPayload`；WS `open` 处理器让每个客户端订阅，使永久的域删除在发起页面
  之外也收敛。它仍是低延迟事件，不是持久队列：重连客户端针对活动工作区的 `session.list` 是其 socket
  掉线期间错过的事件的权威读侧修复。
- **Chat Resources：** 范围化资源读取、命令输出/停止与直接子级停止/全部停止的处理器，在解析工作区
  成员关系并把 id 加注册表拥有的 cwd 传给 agent barrel 之前，强制每个请求的确切键集合。父/子 id
  遵循 Pi 的规范会话语法（含内部点号）；工作区与命令 id 保留其所有者专属语法与权威。缺失的父/资源
  使用 `RESOURCE_UNAVAILABLE`；输出的缺失命令结果只有在校验其父级之后才是 `available:false`。
  不接受任何客户端的路径/PID 字段。agent 的资源发布器映射到 `session.resourcesChanged`，
  在 WS open 处理器中订阅；这是目录失效，绝不是输出广播。资源所有权与拆除留在 agent。
- **Session 状态组合：** 在开始服务之前，宿主提供每个工作区的 `{id,cwd}`、初始化生命周期/回执元数据，
  并安装工作区→项目解析器。`session.stateList` 返回完整的全工作区快照；`session.state` 广播完整记录；
  完成确认与轻推处理器通过同一注册表校验工作区/会话身份。WS open 处理器在 welcome 之后让每个客户端
  订阅。`session.activityList` 在一个窗口期内仍是惰性的 `[]` 兼容方法，且没有推送通道。
- **CLI 更新生命周期：** 启动器可以提供一个可选的异步通知生产者、固定间隔和无参数的更新运行器。
  `createServer` 在监听之后启动检查而不 await 它们，且永不重叠检查。更新的发布成为后续 welcome 中
  保留的 `available` 快照。v70 的 `host.update` 请求在启动一次分离的、服务器级 single-flight 运行之后
  ack；running、succeeded 与 failed 是对每个客户端的 `host.updateAvailable` 的完整替换。成功会锁存到
  宿主重启，且不关停也不重启该无监督进程；失败允许重试，且不暴露子进程输出或任意诊断。发现失败保留
  已知状态，关停让晚到的结果变得惰性，而宿主永不知道 feed、安装器、命令、路径、通道或版本选择。
- **调研邀请投递：** 三个用户发送处理器共享一条 `ackSend` 之后的路径，它一次性过滤控制流量、
  跟踪匿名 `message_sent`，并记录本地反馈计数。feedback 模块注入的发布器把符合条件的 claim 映射为
  针对该请求不透明 client key 的、带地址的 `feedback.interview` 投递，且仅当其 socket 通告的协议版本
  支持该通道；投递失败或重连宽限之后的最终客户端回收会释放该 claim，而瞬时重连会保留它并在
  `server.welcome` 之后重新投递。宿主重启没有可重新投递的 claim，而 welcome 会清除前端过期的弹窗投影。
  弹窗的 `feedback.respond` 动作是普通可重放请求，永不改变 Settings 链接。
- **调研提交：** 在 `initializeAnalytics` 之后，宿主从分析的 `surveyEndpoint()` 配置 feedback 调研源，
  因此无 key、CI 与测试运行永不抓取，而 `THINKRAIL_POSTHOG_HOST` 生效。源码开发宿主只在
  `THINKRAIL_DEV_POSTHOG=1` 与 `THINKRAIL_POSTHOG_PROJECT_KEY` 都设置时抓取。
  `feedback.survey.submit` 重读调研列表（无缓存），以 `SURVEY_INVALID`、`SURVEY_UNAVAILABLE`
  （没有已启动的，或已完成）或 `SURVEY_CHANGED`（另一个 id 或迭代，或用户看到的翻译已消失）拒绝，
  把构建好的负载交给 `captureSurveyResponse`，并只在该采集被接受时记录完成
  （否则为 `SURVEY_NOT_SENT`，以及列表无法读取时）。一旦被接受，该答案就算已发送：完成写入失败会记
  日志，永不报告为未发送，因此重试无法把答案发两次。
- **聊天标题：** `session.rename` 把 `workspaceId` 解析为其 cwd，并把标题校验与无条件的持久写入委派
  给 `agent`；它从不直接 patch 某个客户端。自动标题只来自 agent 的 `set_title`（见上）；发送不携带
  命名工作。手动与自动写入都通过既有的 `pi.event`/`session_info_changed` 通道与 `session.list` 修复
  收敛每个客户端；不存在新的推送通道。

`RunningServer.startAttributionClaim()` 是显式的启动器就绪信号，它在进入分析归因之前重新检查
已保存的 enabled/confirmed 选择。

- **公开接口（barrel）：** `createServer`、`CreateServerOptions`、`RunningServer`、`bootHost`、
  `BootHostOptions`、`BootedHost`、`BuildKind`、`reportDesktopError`。
- **允许的依赖：** `contracts`（`PROTOCOL_VERSION`、特性引入版本、`WS_CHANNELS`）；`shared`
  （`freePort`、`shellEnv`——供 `boot.ts`；`codedError`）；`persistence`（`dataDir`——`crashLog.ts`
  写往何处）；`pi-todos/core`（归约的同步任务快照，组状态仍归 core 拥有）；它组合的各个特性模块
  （按父依赖图，含 `fs` 的 `resolveWorktreeFile`/`resourceMeta` 与 `git` 的
  `readBlobStreamAtAsync`，用于 `/files` + `/blob` 路由）；Bun/Node。
- **禁止：** 被任何特性模块 import；import `web`/`cli`/`desktop`。

- **已结算生命周期接线（`settledLifecycle.ts`）。** 宿主是三个活动来源与 `workspaces` 写入者交汇的
  地方：规范化会话状态发布器在记录报告 `execution: "running"` 时打
  `recordWorkspaceActivity(record.workspaceId)`（用户 prompt 会翻转它，因此用户与 agent 轮次都计数
  ——而写入者每分钟一次的合并让流式 agent 成本很低），一次被接受的 `terminal.write` 给所属工作区打
  时间戳（被位移/空操作写入不打），而 repo-metadata 回调调用 `recordWorkspaceHead`，`watch` 的监听
  启动轻推接线到 `seedWorkspaceHead`，因此首次准入做种子、监听器重建可检测未监听期间发生的 HEAD 移动。
  读取结果、选择工作区或打开文件永不打卡——“看不算工作”是用户可见的规则。`workspace.settle` /
  `workspace.unsettle` 是该模块上的薄处理器；结算永不触碰磁盘，而 `workspace.remove` 仍是唯一的
  拆除方式。启动 `pr.open` 本身在其第一次 await 之前打活动时间戳（打开 PR 是工作，因此它会让停放的
  行重新活跃），并按 `openPr` 报告的、脏读之后的确切分支持久化任何返回的 PR（随后从线上剥掉该
  宿主专属上下文），因此在变更期间的外部 checkout 切换既不会丢弃也不会错挂已创建的 PR；接受一次
  按钮/自动 plan review 也会同步打活动时间戳。**Review 刷新：** 在一次 `workspace.list` 回复之后和
  每五分钟的计时器上，宿主刷新那些 PR 状态仍可能改变分区的行的 `review` 快照——活行每次刷新，
  **空闲已结算行最多每 30 分钟一次**（从 ThinkRail 之外为休眠分支打开的 PR 仍须把它带回来，但休眠
  分支是长尾且必须保持廉价），永不包括用户停放的行（覆盖无论如何都获胜）或带有效终态时间的已合并/
  已关闭快照（不完整的已结算答案仍算活且可刷新；重新打开的完整 review 在激活时被捕获）——并限制为
  少数并发 provider 调用，走该模块的 60 秒缓存。空闲刷新节奏是分支感知的，因此切换分支永不继承旧
  分支的 30 分钟延迟；对活动工作区的一次新鲜 `workspace.openReview` 或 `workspace.branchStatus`
  也写入快照，因此活动行总是最新的。两个写入者都传入该查询所针对的分支，而 `workspaces` 会丢弃分支
  已在其下方移动的答案。**协议错位：** `RequestContext.protocolVersion` 是 socket 通告的协议
  （进程内调用方缺失，它们说当前协议）；`workspace.openReview`（以及 `workspace.branchStatus`，
  其调用方总是更新）对早于 `WORKSPACE_SETTLE_PROTOCOL_VERSION` 的客户端把已合并或已关闭的 review
  回答为 `null`，因为这种客户端会把任何 review 读作打开的 PR，其 plan 面板会提供 *Push updates*
  从而创建新 PR——宿主快照本身仍记录真实状态。**分支状态 / push / 生成（v87）：**
  `workspace.branchStatus` 像 `openReview` 一样组合同一 review 查询 + 分叉，另外加上 `git` 的
  `commitsAheadOf`（相对基准领先）、`pr` 的 `onBaseBranch` 以及 origin 推送 URL 的 provider
  （`originUrls`，与 `pr.open` 读法相同）；对于推送 URL 与抓取 URL 不同的 origin，分叉是对**推送**
  仓库计算的 `countPushTargetDivergence`，因此计数描述的是 Push 动作投递到的仓库——`published` 就是
  分叉答案本身（其远程 ref 存在），所以两者永不矛盾；测量失败时，只对单 URL origin 回退到跟踪 ref，
  而分裂 origin 报告 `published: null`（未知）而不是抓取仓库的答案；`workspace.push` 像 `pr.open`
  一样打活动时间戳并通过 `pr` 丢弃该 worktree 的 review 缓存；`pr.generate` 在这里组合——`pr`
  收集有界输入（`prDraftInput`），`assist` 起草——因此 `pr` 不新增 `assist` 边。**Review 动作
  （v88）：** `pr.ready` / `pr.merge` / `pr.reopen` / `workspace.deleteRemoteBranch` 委派给 `pr`
  并像 push 一样打活动时间戳；快照跟随客户端下一次新鲜的 `workspace.branchStatus`（已合并的 review
  需要 provider 的合并时间，只有该查询拥有它）。`workspace.branchStatus` 上的 `skipFetch` 只跳过
  origin 抓取，永不跳过 review 查询。**回填：** 同一次列表后遍历为没有 `lastActiveAt` 的记录打上
  来自最新聊天 `updatedAt`（`listSessions`）的时间戳，否则取 worktree 的 `.git` gitfile 的 mtime
  （受管与外部 worktree 一视同仁——只有 Default 行，其 `.git` 就是仓库本身，跳到 now），通过
  `backfillWorkspaceActivity`——每条记录一次，因为此后时间戳就存在了。按行的回填/review 失败会记
  日志，分离的列表/计时器遍历总是捕获其终态拒绝；生命周期维护可以降级，但永不能变成未处理的
  宿主致命 promise。

## 必须做对（Get right）

- 每个注册的 WS 命令都按**仅方法名**做 debug 跟踪（`ws <method>` / `ws <method> failed`）；
  不在闭合处理器注册表中的名字改以固定的 `ws unknown method` 跟踪。绝不跟踪原始未注册方法名、参数或
  处理器错误文本，它们可能反映凭据与用户提供的值；见 `submodule-server-log` 的隐私规则。
- WS 命令直接返回值；只有事件 + 扩展 UI + **`project.updated`**（从 `projects` 模块注入的发布器发布）
  + 工作区生命周期三件套（`workspace.created`/`updated`/`removed`，从 `workspaces` 模块注入的发布器
  发布）+ **`session.deleted`**（从 agent 模块注入的发布器发布）+ **`provider.changed`**（从 auth 的
  Central/runtime 失效接缝以及成功的 provider 登录时发布）+ **`host.updateAvailable`**（每次保留的
  CLI 更新生命周期替换时发布）使用推送通道。每个客户端应当听到的**广播**推送通道都必须在 WS `open`
  处理器中被 `ws.subscribe`——对无人订阅的 topic 发布不会到达任何人，且静默无声。四个通道刻意**不**
  订阅也不广播：`feedback.interview`、`terminal.data`、`terminal.exit` 与 `terminal.detached` 用
  `ws.send` 发给一个带地址的客户端。新增带地址通道意味着接一个发布器，而不是一个订阅。
- **终端背压永不只相信 `drain` 事件。** 被背压的 `ws.send` 会把客户端锁存进
  `terminalBackpressured` 并阻塞其批处理器，而批处理器刻意只在 `resume()` 时重试——因此锁存的解除
  必须保证。Bun 的 `drain` 是快速路径，但一次跨系统睡眠丢失的 drain（在 macOS 休眠后被观察到）会让
  锁存永久卡住：标签页冻结、pty 仍活着、记录器仍然是最新的。一个 1 秒的对账器直接询问 socket：
  标志置位时，`getBufferedAmount() === 0` 意味着 OS 从未投递那次 drain（`terminalSend.ts` 中的
  `drainedClientKeys`），于是宿主解除锁存并恢复该客户端的终端。仍有缓冲字节的 socket 保持锁存，
  而 socket 已消失的锁存留给 `close`/`open`。重载客户端总能恢复（open 会清标志并恢复），
  这也是它隐藏了这么久的原因。
- 宿主是特性被接在一起的唯一地方——特性永不反向伸进它。
- **登录 shell 环境导入在启动时是选择加入的**（`resolveShellEnv: true`，由 dev、CLI 与桌面启动器
  传入）。复盘：默认开启时，`boot.test.ts` 把开发者的登录环境——包括 provider API key——复制进共享的
  `bun test` 进程（`bun test` 会丢掉 `TERM=dumb`，因此终端守卫从未触发）。之后 faux 模型套件解析到
  真实 provider、超时，并在花真 token 的同时挂住服务器套件。任何不传该标志的嵌入者或测试保留它被
  给予的环境。
- 分开的宿主进程不协调可变状态或事件。它们可以共用同一数据目录，但各自拥有独立的内存会话、终端、
  监听器与已连接客户端；持久化冲突被接受，而不是由宿主串行化。
- `shutdown()` 在并发信号/原生退出调用下是安全的：调用方收到同一个 promise，生命周期工作只运行一次，
  而资源释放在会话结算与轮次回执排空之后仍保持有序。
- **一次发送（prompt/steer/followUp/respondToTool）在 ACCEPTED 时被 ack，而不是在轮次结束时**
  （`ackSend`）：pi 的发送方法只在轮次结束时 resolve，而一个轮次可能超过客户端的请求超时
  （长工具轮次与数分钟推理轮次是常态）——await 完成会在一个健康的轮次上冒出幽灵般的“请求超时”。
  ack 窗口内的拒绝仍会让请求失败（坏模型 / 缺 key；对 `respondToTool` 还有未知/已回复/已被取代的
  调用——`assessRespondability` 的大声裁决，或没有值的响应）；之后的故障通过事件流到达客户端。

### 漂移与覆盖（Drift & overwrite）

一个 finding 锚定在 `side:"worktree"`（活文件——它随代码行内跟随）。当*后续* plan 步骤覆盖被评审的
行时，`reanchor` 把它降级为 `outdated`（`reviews.ts` 的 reanchor），但 `maybeAutoReReview` 由 SHA
水位增长触发，或（路径列表回退的等价物，没有 sha 可作水位）记录读作 `unreviewed` 而消耗过的自动
周期仍留在记录上——见 `todos/SPEC.md` 的自动周期持久性。**派生的 `stale`** 条件守住这一点——
`anchorState === "outdated"` 且该 finding 的 origin sha 在其步骤上被取代
（`todos.reviewedShaSuperseded`）。每个 agent finding 都携带 `origin`（步骤 + 会话 + 被评审的 sha，
由 `fileFinding` 从该裁决所属的 review 打上）；`isFindingStale` 把失效 finding 从自动 fix 集合中
剔除。客户端徽标走**服务端派生、非持久化的 `stale` 标志**：`markClientStale` 富化每个穿越到客户端的
快照（`review.get` + `review.changed` 广播）——宿主是唯一能这么做的环，因为失效性把 finding 的
`origin`（reviews）接到其步骤的提交（todos）。`stale` 从不是持久化字段。

保留被评审的*原始*代码曾被考虑并否决：事后从 `reviewedSha` 重构它并不可靠（finding 的 textQuote
来自添加时的 worktree，而那并不保证等于该 blob），而添加时快照是真实存储成本换低价值（代码已被
覆盖的 finding 通常是 moot 的；它的散文正文无论如何都会存活）。若真需要，唯一可靠的路径是添加时
快照，绝不是漂移时冻结。

### 持久化 delta（整个阶段）（Persisted delta (whole phase)）

一个可选的持久化 `ReviewComment` 字段携带 finding 来源：
`origin?: { todoId, reviewedSha, sessionId }`——**已落地**。线上的 `stale?: boolean` 由宿主按客户端
快照派生，永不存储。没有新的工具参数，没有新的 `status`/`anchorState` 枚举值。
