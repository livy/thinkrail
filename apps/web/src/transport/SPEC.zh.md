# transport —— 通往宿主的 WS 客户端

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

通往宿主的唯一 WebSocket 客户端、它的应用级单例，以及那条有序投递边界：它把高频 Pi 事件成批处理，
同时不允许后到的 wire 消息越过它们。

## 边界（Boundary）

- **拥有：** `transport.ts`（`WsTransport`：按 id 关联的 `request` —— 除非调用方提高 `timeoutMs`，
  应答在 60s 后超时；一个宿主*只有在人类操作之后*才会应答的请求必须这么做（打开文件夹的对话框：
  一旦触发超时也会丢掉随后到达的应答）——，它追加到 socket URL 上的 **`?client=` 页面身份**与
  **`?protocol=` 当前 wire 版本**（该身份惰性生成，且*不是*经由仅安全上下文可用的
  `crypto.randomUUID`，因此纯 http 的远端源仍能启动；它跨重连但不跨重新加载，用于关联重放的请求与
  终端流路由，而宿主持有的 PTY 标签／shell 依然存活，重新加载的页面靠持久 `tabKey` 接管它们；版本号
  让较新的宿主能避免为一个独立发布的、无法渲染该功能的旧客户端声明某个被寻址的功能），
  **重连安全的未解析请求** —— 一个在 socket 死亡时仍在途的 frame 会回到队列并在同一 request id 下
  重放，而宿主对 `(clientKey, requestId)` 去重，因此一个已被接受的变更不会变成假失败，也不会执行
  两次 ——，构成这一契约中本方一半的两个 frame —— **`{ ack: [id] }` 回执**（每次响应读取都会被确认，
  在 microtask 上批量发送；在某个回执到达之前，宿主必须假定该应答随 socket 一起死亡并保持它可重放）
  与在每次（重新）连接时、于重放*之前*发送的 **`{ resume: [ids] }`** 对账（完整的仍未解析集合，
  于是宿主释放其他一切）。回执刻意是尽力而为且永不重传 —— 它会像响应一样死在 socket 缓冲区里，而它
  所指的请求已从 `pending` 中消失，因此没有任何东西会重放或重确认它；`resume` 通过重申事实而不是确认
  确认，一次性修复所有这些 ——，通道 `subscribe` 对快照带最后值重放；仅追加的终端数据，以及一次性的
  终端退出／分离 + 会话创建／删除 + `provider.changed` 失效 + 被寻址的 `feedback.interview` 通道，
  加上限定范围的 `session.resourcesChanged` 失效与按会话的 `mcp.status` 快照，永不缓存，也永不向
  迟到的订阅者重放。资源元数据水合属于已挂载的 chat 集成，而不是重放的失效载荷。重连／退避；
  `inferUrl` 默认为同源；**`httpBase()`** 从 WS 的 `url` 推导宿主的 HTTP 源 —— 用于构造宿主 HTTP URL，
  例如 markdown 查看器把相对 `<img>` 指向的 `/files/<workspaceId>/<path>` worktree 文件端点，目标正是
  transport 所拨号的同一宿主）；`piEventBatcher.ts`（浏览器侧针对连续 `pi.event` frame 的有界队列：
  精确到达顺序、不丢弃事件、以约 30 Hz 做一次原子投递、128 事件的强制冲刷上限，以及 `flush`／
  `dispose` 生命周期）；`wireTransport.ts`（`initTransport`／`getTransport` 单例；把 `server.welcome`、
  **`host.updateAvailable`**、**`project.updated`**、`pi.event`、`pi.extensionUi`、**`session.created`**、
  **`session.deleted`**、**`provider.changed`**、被寻址的 **`feedback.interview`**、
  **`workspace.created`/`updated`/`removed` 生命周期三件套，以及 `workspace.fsChanged`** 路由进 store
  —— 并让每次连接转换都经过 `setStatus`，其 connected 代次为活动工作区水合在每次重连时提供不同的
  触发；完整 welcome（协议 + 打开／最近项目视图 + 可选 config）经由原子 `installWelcomeSnapshot`，
  其独立的 `welcomeGeneration` 就是冷启动导航的就绪边沿；每次 welcome 之后，它用
  `includeDiffStats: false` 重新读取本界面已持有工作区列表的每个项目，对结果做代次围栏，并只把已
  已知的行经 `updateWorkspace` 折叠进去，因此一个在断开期间错过的推送式完整工作区快照（包括重命名）
  不会停留在陈旧状态，同时也不会把这次元数据修复误表述为成员关系归约；完整的宿主更新生命周期快照
  经 `applyHostUpdate`，项目快照经 `applyProjectUpdated`，连续的 `pi.event` frame 经批处理器进入一次
  `handlePiEvents(payloads)` store 提交，`pi.extensionUi` 经 `applyExtUi(request)`，
  `workspace.created` 经 `addWorkspace(workspace)`，`workspace.updated` 经
  `updateWorkspace(workspace)`，`workspace.removed` 经 `applyWorkspaceRemoved(projectId, id)`，
  `session.created` 经 `noteClosedChats`（同伴创建的领域状态只进入历史，绝不进入本地 placement），
  `session.deleted` 经幂等的 `deleteChat(workspaceId, sessionId)` 墓碑折叠（一条在线快路径；由于该
  事件通道刻意不重放，workbench 水合会从下一次权威 `session.list` 修复断开期间错过的任何删除），
  `provider.changed` 经原子的 store 失效 `noteProviderChanged()`，加上一次经 store 单调 provider 版本
  守卫安装的 `model.list` 重读（model 目录 hook 对每次 list／refresh 使用同样的受守卫写入，因此较旧的
  应答无法恢复某个已移除代次的 model；provider 设置观察同一版本并重读 `provider.status`），每个有效
  的 `server.welcome` 先清除宿主重启留下的任何弹窗投影，然后 `feedback.interview` 经幂等的
  `showSurveyInvitation()`，且只来自 v97+ 的宿主（较旧的宿主仍在该通道上发送已退役的访谈邀请）——
  shell 把该邀请挡在 onboarding 与同意对话框之后，而额外分析可能处于关闭状态（一个仍然存活的宿主
  声明会在 welcome 之后立即重新投递该被寻址的事件）。`loadInAppSurvey()` 在 welcome 之后、Settings
  打开时、调查窗口打开时以及一次 `SURVEY_CHANGED` 拒绝之后，把 `feedback.survey`（以 `currentLocale()`
  询问，因此当 PostHog 有对应 UI 语言的译文时就显示它；语言切换在下一次加载时生效）安装进 store；
  它永不拒绝，且过期应答会按代次丢弃，因此成功 Send 之后的 `forgetInAppSurvey()` 不会被一次仍在途的
  加载撤销，`workspace.fsChanged` 经 `noteFsChanged(payload)`，**`mcp.status`** 经受代次守卫的
  `applyMcpStatus(snapshot)`，**`settings.changed`** 经 `applyConfig(config)` —— 即启动之后由服务器
  同步的应用配置广播；welcome 的 config 落在上面那次原子安装中。

  **会话状态在每次受支持的 welcome 上水合。** `session.stateList` 按连接代次加戳，并在完整快照返回
  之前缓冲 `session.state` 推送，把这些完整记录替换按到达顺序折叠到各自的快照行之上，然后一次性安装
  得到的权威映射。当前代次的失败会保留之前的映射，同时以有上限的退避重试；溢出有界推送缓冲会重启
  完整读取，而不是无限增长，而过期代次的结果会丢弃其缓冲。待定的对话框记录会重放其精确请求，而添加／
  打开工作区会重启受代次守卫的完整读取，从而包含此前已存在于磁盘上的会话。不受支持的宿主会清空该
  映射。这个客户端永不请求旧的 `session.activityList` 墓碑。在 `WsTransport` 派发一次状态推送之前，
  任何已排队的 Pi 事件都会同步冲刷，因此运行时转录／渲染状态先行于引用它的宿主状态；连接状态转换
  保持同一屏障。该派发屏障保住跨消息顺序与 store 的转录修订围栏，同时仍然合并连续的流式 frame。
  所有订阅都在 init 时一次性完成，绝不在组件 effect 中）；
  `errorText.ts`（**`errorText(err, fallback?)`** —— 把一个被拒绝的 `request`（宿主的错误字符串／
  超时／抛出的非 Error）归一化成一行简短、可直接展示的文字，用于错误轮次／提示）；
  `requestError.ts`（**`RequestError`** + **`wsErrorCode(err)`** —— 一个携带宿主具名
  `WsResponse.errorCode` 的拒绝。带 code 的响应以 `RequestError` 拒绝，其他一切（超时或无名宿主
  错误）以普通 `Error` 拒绝，因此*拥有*一个 code 正是调用方区分“这个具体失败”与“读取失败了”的
  依据）；`skillLoad.ts`（会话资源加载的唯一应用集成协调者：每个工作区单飞 `workspace.watchReady`；
  除非监视器已知就绪，否则在本地折叠保守的 `skillChange: "unknown"` 通配作为可重放安全的回退；
  之后才捕获 store tick。它窄小的 `prewarmWorkspaceSkillLoad` 入口让工作区导航器能在选择之前启动
  同样的准备，而无需复制就绪／回退策略；失败仍可由最终的加载重试。预热式准备在线上被标记
  （`prewarm: true` —— 宿主把仅预热的监视器保留在有界、可淘汰的池中），且**永不成为真实加载的基线**：
  第一次真实加载总是运行它自己的 real 标记准备 —— 监视器仍温热时立即得到应答，而被淘汰时则重新
  创建／提升它（包含一次全新的保守 nudging）—— 因此一次被淘汰的预热永远无法让会话停在过期的
  新鲜度基线上，同时预热可以自由搭载任何在途准备工作，而一次已结算的预热会重新发出（在项目被重选
  时重新温暖一个被淘汰的监视器依旧便宜）。`session.create`／`session.reloadResources` 包装器只在
  那次准备之后才发出它们的加载，因此没有任何调用点会意外颠倒就绪与基线的顺序。
  `session.getMessages` 包装器是一次转录读取，永不等待就绪（一个新监视器的 ~750ms 启动否则会拖慢
  每一次冷恢复）：宿主用一个实时摘要应答它，其调用方会丢弃基线，因此它发送同样的真实准备但不等待它
  （读取仍会启动并提升监视器；失败仍可由下一次加载重试），并汇报读取之前捕获的 tick —— 万一有只来自
  磁盘的摘要到达时采取保守。除非返回的摘要同时精确匹配所请求的工作区与会话，它还会拒绝，从而使那次
  对不可信响应的身份校验成为单一的共享安装边界，而不是调用方约定）。
- **`mcp.ts`** 是 Settings 与 chat 的 Resources 视图共用的唯一 MCP 读取路径：
  **`requestMcpList(method, params)`** 按工作区与连接代次串行化 `mcp.list` 以及每个返回列表的变更，
  包括在派发下一个请求之前经由 `installMcpList` 的安装。这防止较旧的列表或变更应答覆盖较新的配置或
  实时 chat 成员关系。只有位于队列尾部的普通读取会被共享：一个在已排队变更之后请求的读取必须等那个
  变更完成并发出一次全新读取，绝不重用变更前的读取。变更从不共享。store 的 `selectMcpRead` 安装令牌
  在真正派发时捕获，而不是在入队时，从而既保留权威的会话省略，又保留在请求期间到达的推送。排队请求
  在派发前会重新检查 MCP 可用性、工作区移除以及它们捕获的连接代次；过期的排队工作拒绝而不发送，
  而已经派发的应答保留 store 的连接与删除围栏。新连接独立于较旧代次的队尾开始，工作区之间不互相阻塞，
  失败的请求会拒绝其调用方而不毒化队列或不阻碍重试。这个队列是临时的 transport 协调，不是第二个
  状态所有者。**`watchMcpWorkspace(workspaceId, onRead?)`** 先读一次，然后只在某条属于该工作区的
  实时 chat 仍报告服务器侧 `starting` 时，以有上限的退避（1 → 13 s，每条连续尝试六次）重读。宿主的
  `mcpSessions.capture` 在其调用方超时之后继续进行，并最终发布已结算的状态，包括 connected 边沿；
  浏览器侧的有界轮询仍是刷新助手，而不是唯一的完成路径。一旦没有任何东西处于 starting，连续尝试预算
  就重置；释放会取消定时器并抑制迟到的监视器回调。独立的监视器只在上面的队尾规则之下共享读取。
  工作区移除会在退役本地 modal 之后通过 `provider.loginCancel` 取消其活动中的 MCP 登录。为已立墓碑
  工作区迟到的登录 frame 会被取消而不是安装；若移除先于其确认，MCP 启动响应路径也如此处理。取消是
  在既有重连安全请求 transport 之上的尽力而为；宿主有界的探测生存期才是最终清理。
  **客户端错误上报**（`errorReports.ts`）：`installClientErrorReporting()` 监听窗口的 `error`／
  `unhandledrejection`，而 `reportClientError(clientErrorReport(kind, error, boundary?))` 通过
  fire-and-forget（发后不理）的 `client.reportError` 转发原始错误类与栈（并从栈中切掉消息 —— V8 的
  栈头会重复消息，其行看起来可能像 frame）—— 只发给具备 `CLIENT_ERROR_REPORT_PROTOCOL_VERSION`
  （v90）的宿主；welcome 之前产生的上报会等待它（≤ 10 条排队），且每个页面至多发送 50 条。每条上报
  携带 PostHog 的 chunk id（≤ 20），即 release 构建注入的 `_posthogChunkIds` 注册表为其自身栈中的
  URL 分配的那些，从而让上传的 map 能解析它的 frame。浏览器永不擦洗或持有第三方客户端：宿主是唯一
  的清洗者。`main.tsx` 安装这些监听器，并把转发器注入 `components` 的错误边界接缝。

- **公开接口（barrel）：** `initTransport`、`getTransport`、`prewarmWorkspaceSkillLoad`，三个 skill
  加载安全的会话请求包装器，`errorText`、`RequestError`、`wsErrorCode`、`requestMcpList`、
  `watchMcpWorkspace`、`ConnectionStatus`、`TransportOptions`、`runHostUpdate`、
  `supportsHostUpdateRun`、`supportsPlanReview`、`supportsPlanSummaryGeneration`、
  `supportsChangeMutations`、`supportsRichAnchors`、`supportsTurnChanges`、`supportsUserThemes`、
  `supportsToolResponses`、`respondToTool`、`installClientErrorReporting`、`reportClientError`、
  `clientErrorReport`。`respondToTool(sessionId, toolCallId, response)` 是工具响应离开客户端的唯一
  位置（chat 与 plan 页面都用它）：它向 v93+ 宿主发送 `session.respondToTool`，否则发送下面已退役的
  `session.answerQuestion`（`supportsToolResponses`），覆盖一个兼容窗口 —— [[module-contracts]]。
  `runHostUpdate` 是带类型的空宿主动作，`supportsHostUpdateRun` 让 `Shell` 只在协议 v70+ 时注入它；
  `supportsUserThemes`（v92）被导出，因为一旦一个无法提供主题文件的宿主向其发出 welcome，`Shell`
  就必须丢弃用户主题（缓存的或来自更早宿主的）；`supportsPlanReview` 被导出，因为兄弟面板
  (`PlanPane`) 以它为门禁来决定计划评审 UI —— 较旧的宿主不提供 `todo.startReview`／`reviewAll`，
  因此客户端不得提供它们；`supportsPlanSummaryGeneration`（v69）被导出，因为 `chat/useChatTodos`
  以它为门禁来决定自动摘要的 `todo.generateSummary` 请求 —— 较旧的宿主没有这样的方法；
  `supportsChangeMutations` 被导出，因为在该协议落地之前 `DiffPane` 必须扣下所有 `change.*` 能力；
  `supportsRichAnchors`（`REVIEW_RICH_ANCHORS_PROTOCOL_VERSION`）被导出，因为
  `panels/useReviewCommenting` 拒绝向早于它们的宿主发送 `region`／`structural` 草稿 —— 这类宿主会
  不经验证地存储选择器，并把它当作文本重新锚定，于是评论虽然存活却含义不同；该拒绝发生在草稿成为
  `review.commentAdd` 请求的唯一位置，是一个普通错误 toast，而行级／整文件草稿不受影响；
  `supportsTurnChanges`（`TURN_CHANGES_PROTOCOL_VERSION`，v80）被导出，因为
  `panels/useWorkspaceTurns` 不得向没有该方法的宿主发出 `workspace.turns` —— 没有它时，Changes 的
  范围菜单干脆不显示 agent 轮次。transport 还订阅 `turn.changed`，并通过 `applyTurnChanged` 把每次
  被记录的运行折叠进 `store.turnsByWorkspace`。仅字节的资源形状不需要门禁：早于
  `RESOURCE_META_PROTOCOL_VERSION` 的宿主应答 `fs.readFile` 与 `git.diffFile` 时不带 `meta`，
  各面板把它读作 "text" —— 正是该宿主自己的客户端所显示的那个界面 —— 而 `/blob` 只为宿主自己
  报告为仅字节的那一侧被寻址。
- **允许的依赖：** `contracts`（方法映射、`WS_CHANNELS`、用于 welcome + `project.updated` 的
  `Project`、用于 `pi.event` 的 `SessionEventPayload`、用于 `pi.extensionUi` 的 `ExtUiRequest`、
  用于 `workspace.created`/`updated` 的 `Workspace`、用于 `workspace.removed` 的
  `WorkspaceRemoved`、用于 `session.created` 的 `SessionCreatedPayload`、用于 `session.deleted` 的
  `SessionDeletedPayload`、用于 CLI 宿主更新动作的 `HOST_UPDATE_RUN_PROTOCOL_VERSION` + `Ack` +
  带类型的 `host.update` 方法、用于 `supportsPlanReview` 门禁的
  `PLAN_REVIEW_SUBAGENT_PROTOCOL_VERSION`、用于 diff 变更门禁的 `CHANGE_MUTATIONS_PROTOCOL_VERSION`、
  用于丰富锚点撰写门禁的 `REVIEW_RICH_ANCHORS_PROTOCOL_VERSION`、用于 `client.reportError` 的
  `CLIENT_ERROR_REPORT_PROTOCOL_VERSION` + `ClientErrorReport`、`provider.changed`、空的被寻址
  `feedback.interview` 邀请、用于 `server.welcome` + `host.updateAvailable` 的 `HostUpdateNotice`、
  用于 `workspace.fsChanged` 的 `WorkspaceFsChangedPayload`，以及用于 `server.welcome` 的 config +
  `settings.changed` 的 `AppConfig`）；`store`（welcome + 事件路由 —— 一条由父图拥有的运行时边）；
  `lib`（纯 HTTP 安全的随机页面身份）；`@thinkrail/locales`（用户可见的请求失败消息）；浏览器
  `WebSocket`。
- **禁止：** `server`／`shared`／任何 `pi` 包；导入 `panels`／`shell`；或请求、订阅、折叠当前布局状态。
  浏览器布局持久化只把 `httpBase()` 用作其前端本地存储身份的一部分；原生的稳定持久化没有 transport 边。

## 必须做对（Get right）

- **`DEFAULT_TIMEOUT_MS` 是宿主侧预算必须落在其下界之内的上限，且没有任何东西强制它。**
  服务器把网络的 `git` 限制在 55s，正是为了让自己的错误 —— 指明 ref、携带 git 的 stderr —— 赢得与
  本方在 60s 抛出的、无因的 `request "…" timed out` 之间的竞争（issue #209；
  `packages/server/src/git/SPEC.md`）。把这个数字降到该预算之下，每一次卡住的 fetch 都会静默退回到
  通用超时，而没有任何测试或类型会指出这一点。这两个常量位于独立发布的产物中，靠约定而正确；把它们
  互相推导出来属于 `contracts`。提高某个请求的 `timeoutMs` 是安全的，降低默认值则不是。
