# plan review —— 隐藏的委派子 agent，而不是评审人聊天

> 中文译文，原文见 [`planReview.SPEC.md`](./planReview.SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

独立于撰写它的 agent，评审一个已完成的 plan 步骤，并把结果转成状态：该条目的 review 记录、Review
标签页中的 finding，以及在 fix 预算允许时投递给 worker 聊天的 fix 请求。

文件：`requestReview.ts`（流程）、`reviewerRole.ts`（评审人的 prompt + 输出契约）、
`planReviewQueue.ts`（按 plan 串行 + 按条目认领）。

## 为什么是子 agent 而不是聊天（Why a subagent and not a chat）

最初的实现把评审人跑成**第二个可见的 pi 会话**，按 plan 钉住，持有自己的工具
（`add_review_comment`、`review_verdict`、`reflect_finding`）。这种形态的代价大于收益：

- **一个会话就是一份要持续看管的存在期。** 崩掉、被中止，或者只是结束轮次却没调用
  `review_verdict` 的评审人，会永久留下该条目的 `pending` 标记，永远转 `Reviewing…` 并让
  Review All 死锁。从这种状态恢复需要 settle tee、评审人→worker 注册表、终止分类器，以及一个按结算
  而非按结果推进的队列（`reviewerSessionMonitor`、`reviewQueue`）——约 400 行代码，唯一职责就是察觉
  某个会话已经停了。
- **工具让裁决变成轮次中途的副作用。** `review_verdict` 在评审人仍在流式输出时就清掉了持久标记，
  因此下游每道守卫都必须区分“裁决已记录”与“轮次已结束”，而来源必须钉在 `currentReview` 映射里，
  并且会被下一个包覆盖。
- **用户从来就不想要那个聊天。** 它唯一的用处就是看评审人思考。

委派子会话把这一切反转：`runReviewSubagent` 返回一个 **promise**。完成就是 resolve，失败就是
reject，而整个恢复面塌缩为一个调用 `cancelTodoReview` 的 `catch`。评审人不需要我们的任何工具——
它用 `read`/`grep`/`find`/`ls`/`bash` 读代码，返回一个围栏 JSON 裁决，由宿主解析并应用。

## 不变量（Invariants）

- **评审人使用宿主的新聊天默认值，加上仓库的评审人配置。** `runReview` 把子会话的模型解析为钉住的
  `reviewModel`，否则 `resolveNewChatModel({})`：AppConfig 的 `defaultModel`，回退到首选的可用模型
  （都不可用时则无模型）——绝不是 worker 静默继承来的模型。评审人的 **effort** 同样解析：钉住的
  `reviewEffort`，否则 AppConfig 的 `defaultEffort`，回退到 `medium`；有可用模型时解析器会为所选模型
  夹取它。`runReview` **无条件**传入 `thinkingLevel`——省略级别会让 pi-delegation 继承父 worker 的
  effort，因此未设置的项会静默地让评审人以 worker 的 effort 运行，而不是默认值。`runReviewSubagent`
  以 `contextFiles: true` 与 `extensions: true` 生成子会话，而 `REVIEWER_TOOLS` 在
  `read`/`grep`/`find`/`ls`/`bash` 之外携带 `spec_grep`/`spec_get`/`spec_graph`，复现内置
  `reviewer` agent 的上下文/工具策略，使 plan review 遵循仓库指引并能审计不变量。只有系统 prompt +
  JSON 输出契约不同（宿主解析裁决）。
- **宿主拥有评审人的角色，包只拥有事实。** `reviewerRole.ts` 持有系统 prompt（评审顺序、什么算
  finding、JSON 契约）；`todos.renderReviewPackage` 渲染变更集*引用*与 worker 的声明，且不点名任何
  工具。一个指示评审人所没有的工具的包，就是自相矛盾的 prompt——这个拆分正是为防止该 bug。
- **模型输出不可信。** `parseVerdict` 对裁决词严格，并通过 contracts 的 `isPlanReviewResult` 校验每个
  finding：每个都需要非空 `id` + `body`、一个已知枚举中的 `kind`（若存在），以及一致的正行范围
  （行需要 path，`endLine` 需要 `startLine`，`endLine >= startLine`）——否则畸形位置会在
  `fileFinding` 的锚点解析中抛出并静默丢弃该 finding。基数也被强制：没有 finding 的
  `request_changes` 被拒绝（它会让 worker 无所可修地搁浅），而 `approve` 可以不带 finding。
  任何非法输出都是失败的 review，而不是静默 approve：标记被清除，条目回到 unreviewed。
  `parseVerdict` 还**显式构造结果**（只有 verdict/summary/readingOrder/findings——`readingOrder` 是
  评审人可选的有序 `{ path, why }` 列表，由契约校验器限制在 `REVIEW_READING_ORDER_LIMIT`；任何
  finding 或 reading-order 路径中的反斜杠被读作分隔符并在此转为 `/`，因为模型可能在任何宿主上写出
  Windows 路径（架构 Decision 22）；**每个被接受的裁决**都在与其 finding 和成功裁决记录相同的
  `withReviewLock` 持有期内，通过 `reviews.setReviewGuide` 替换该工作区 review 的 `guide`——评审人
  未给出时 summary 与 order 为空。因此排队的 Clear 会同时移除 guide 与 finding，绝不会只移除 guide
  而归档仍在继续。被拒绝、归档/记录失败以及被未关闭 finding 阻塞的 approve 都让上一份 guide 保持不变，
  绝不会出现新的成功徽标。guide 持久化在接受之后仍是尽力而为；fix 准备/发送失败保留被接受的
  request-changes guide 以及既有的周期/标记补偿。worker 投递保持在锁外），而不是展开模型对象，
  因此模型幻觉出的宿主专属字段——例如 `blockedByOpenFindings`——永远无法存活到误导卡片；
  宿主只从自己的未关闭 finding 检查设置它。
- **`reviewing` 标记在启动/入队时同步设置**，因此客户端一重读 plan，面板就在任何 await 之前闪动。
  被接受的按钮或自动 review 也在同一个同步步骤记录工作区活动——review 是对工作区的工作，和一次聊天
  轮次一样，因此它让该行离开 Settled 架。
- **分离的失败会被发布，而不只是记日志。** `todo.startReview`/`todo.reviewAll` 在 review 入队的
  那一刻 ack；随后运行在分离路径上失败（provider 错误、非法输出、中止），而它没有自己的聊天可展示。
  `startPlanReview` 清除 `reviewing` 标记并调用 `reviewFailedPublisher`
  （`ReviewFailedPayload`，在 `review.failed` 上广播），使 plan 页面能弹出 toast——这是面板规格对
  toast 携带失败的要求。载荷携带所属 `sessionId`，使只有该 plan 的视图弹 toast（而不是工作区里的
  每个 plan）；同一会话的分屏视图按 toast 正文去重。被 await 的工具路径不需要发布：它向 worker reject。
  由 `planReview.test.ts` 中 ack 后失败测试钉住。
- **`autoCycles` 必须与实际发生的事一致。** 只有 worker 真正接受了 fix 请求时才写 `1`；被拒绝的 fix
  ——发送**或**其任何准备步骤（快照/包/标记）失败——或被拒绝的 fix 闸锁，都会在把已标记 finding
  回滚为 `draft` 之后重新记录 `2`（终态）。`deliverFixToWorker` 把准备+发送包在一个 catch 里，
  正是为了让准备失败无法以抛出溜走并把乐观的 `1` 留在那儿。写入 `1` 却没有投递的请求会让条目搁浅：
  自动重审触发器等待的 delta 无人产出，而之后的一次 review 会把该周期读作已消耗。由
  `planReview.test.ts` 中发送被拒与准备被拒的测试钉住。
- **approve 是裁决，不是结算。** `recordVerdict` 只在 `itemOpenFindings` 为空时结算 `reviewed`；
  否则它清除标记、不动记录，并报告 `approve-blocked`，使 worker 被告知去解决它已修好的东西。
  fix 周期的第 2 轮是与第 1 轮分开的运行，因此没有任何结构性东西阻止 approve 落在未解决的 `sent`
  finding 之上——只有这道检查会。
- **工具路径在快照变更集之前 await 制品对账。** `request_review` 紧跟 `todo_update` 触发，而
  `maybeAttachChangeArtifacts` 可能仍在提交该步骤的工作。`handleRequestReview` 在 `startTodoReview`
  之前 await `settleChangeArtifacts`（`listTodos` 使用的同一屏障），因此评审人永不会看到一个没有
  变更集的步骤，也不会快照已被取代的制品。按钮路径不需要这样的 await——用户在一个已对账的 plan 上
  启动它。由 `planReview.test.ts` 中 held-reconcile 顺序测试钉住。
- **认领不得比取得它的调用活得更久。** `handleRequestReview` 认领该条目，因此每个认领后的出口——
  包括 `startTodoReview` 在没有变更集的步骤上抛出——都必须执行释放。泄漏它会把该步骤卡成
  “已在被评审”直到宿主重启。两者都在 `planReview.test.ts` 中钉住。
- **两个入口共享上限。** worker 的 `request_review` 工具与 Start review 按钮计算同一个 `canAutoFix`；
  工具路径在工具结果文本中报告它，按钮路径按它行动并发送 fix。没有共享上限，工具路径会永远循环
  fix → review → fix。
- **静态工具指引是策略中立的；工具结果携带下一步动作。** `composeText` 依据 `canAutoFix` 可以写
  “修复并再次 request_review”，也可以写“现在不要修复——报告给用户”。因此 `requestReviewTool` 的
  description/promptGuidelines 绝不能硬编码“总是修复并重审”——它们告诉 worker 遵循工具结果陈述的
  下一步动作，使更强的系统级 prompt 无法覆盖配置出的停止。
- **面向 worker 的 finding 身份是持久化的评论，而不是模型的 id。** `fileFinding` 把每个 finding
  持久化为一条新的 `rc_*` 评论；模型自己的 id（`f1`…）是瞬态的。在工具路径上工具结果*就是*投递，
  因此当 worker 被要求修复（`canAutoFix`）时，`recordVerdict` 把这些 `rc_*` 评论作为一次
  `withReviewLock` 事务归档并标记 `sent` 给发起会话（在消耗周期之前），结果文本/卡片点名它们的
  规范 id。归档与标记必须原子：否则一次并发的 Review clear 或中途的非草稿冲突会留下未关闭 finding，
  其规范 id worker 从未收到，从而把后来的 approve 卡在它无法点名的草稿之后——因此标记失败会删除刚
  归档的草稿并 reject 该请求。否则 `resolve_comment`——只按持久 id 关闭分配给该 worker 的 `sent`
  评论——永远无法满足未关闭 finding 门禁，把 request_changes → fix → approve 卡死。终态路径
  （“不要修复”）把它们留为 `draft` 给用户。归档本身在每条路径上都是全有或全无：若后续写入抛出，
  `fileFindings` 会删除它已持久化的草稿，且它总是在 `withReviewLock` 内运行（按钮与 auto-fix-off
  路径在归档周围取锁；工具路径已持有它），因此并发的 Review 发送无法在两次写入之间把某个 finding
  标成 `sent` 并挫败该补偿——已发送的 finding 无法被删除，会未被投递地搁浅。在**按钮路径**上归档走得更远：
  `deliverFixToWorker` 在一次 `withReviewLock` 持有内归档、记录周期、选择草稿并标记它们 `sent`，
  然后才是（必然无锁的）聊天发送，因此任何插入的 Review 发送都无法在归档与预留之间标记刚归档的草稿
  ——那会让 `itemFixFindings`（只含草稿）为空，并在仍消耗一个周期的同时交给 worker 一个没有规范 id
  的通用请求。失败按归档是否完成而分流：*归档*失败抛出（什么都没记录 → review 取消），任何
  *归档后*失败记录周期 2 以归还自动周期并把已标记 finding 回滚为草稿。周期记录写入本身在每条路径上
  都是归档事务的一部分：若它在 finding 持久化之后抛出，它们会在同一把锁下被撤销——任何 `sent` 分配
  回滚，然后草稿被删除（`unfileFindings`）——之后取消才传播，因此 sidecar 失败永不留下一个 id 从未
  到达 worker 的未关闭 finding。记录写入本身也是原子的：`recordAgentChangesRequested`
  （以及 `approveTodoReview`）把该条目的记录、其自动周期计数与其 pending 标记清除持久化为**一次**
  快照写入（`commitReviewTransition`），绝不写成三次，因此部分失败无法把条目停在已消耗周期却 pending
  仍置位的 `changes_requested`。由工具路径 finding 身份测试、`planReview.test.ts` 中的插入发送/
  部分持久化/投递前竞态/记录失败回归测试，以及 `reviewFlow.test.ts` 中的单快照原子性测试钉住。
- **agent 的会话内入口是门禁的；按钮不是。** worker 的 `request_review` 工具总是注册，但只有
  `agentReviewEnabled` 配置标志开启时才保留在会话的活跃工具集里。宿主注入
  `setAgentReviewEnabledResolver(() => getConfig().agentReviewEnabled !== false)`，使 agent 模块
  永不获得 settings 边（见 `agent/SPEC.md`），而切换该设置会调用 `refreshAgentReviewTool()` 把变更
  **实时**应用到打开的会话（空闲会话同步应用，流式会话在其下一个 `agent_settled`）。因为
  `setActiveToolsByName` 只根据活跃工具的指引重建系统 prompt，丢掉该工具也丢掉了它的指引——
  禁用后不会有残留的“总是 request_review”指令存活。它只门禁该工具：Review 按钮
  （`startPlanReview`）及其下游的自动 fix / 自动重审是独立的，且总是可用。
- **每个 plan 同时只有一个 review，每个步骤一生只有一个。** 两个入口都在该 plan 的链上串行
  （`planReviewQueue.onPlanChain`）：按钮路径经 `enqueuePlanReview`（fire-and-forget），worker 的
  `request_review` 工具则 await `onPlanChain` 取结果。该链让 Review All——以及一次工具请求与另一步
  按钮 review 的竞态——不会同时打开两个 provider 流；按条目认领则让两个裁决不会竞相写进同一记录。

## 边界（Boundary）

- **拥有 / 公开接口：** `startPlanReview(workspaceId, sessionId, itemId, runSubagent?)`、
  `maybeAutoReReview(workspaceId, sessionId)`、`installRequestReviewSeam()`、
  `setReviewFailedPublisher(fn)`，以及纯函数 `parseVerdict` / `composeText`；`planReviewQueue` 的
  `enqueuePlanReview` / `onPlanChain` / `claimItemReview` / `releaseItemReview` /
  `itemReviewActive` / `planReviewRunning`。
- **允许的依赖：** `agent`（`runReviewSubagent`、`sendReviewFixToSession`、`getSessionWorkspaceId`、
  `notifyExtUi`）、`todos`、`reviews`、`settings`，以及宿主兄弟模块 `ackSend` / `reviewLock` /
  `todoReview`（fix 闸锁 + finding 定范围）。
- **禁止：** 从 `todoReview` import `requestReview`（依赖是单向的，因此完全没有 import 的
  `planReviewQueue` 才是双方读取在飞状态的地方）；越过 barrel 伸进 `todos/reviews.ts` 或
  `reviews/*` 内部。
- **测试：** `planReview.test.ts` 通过 `startPlanReview` 的最后一个参数注入桩 runner，并针对
  faux 模型 worker 会话驱动真实的记录 → 归档 finding → 投递路径，因此裁决语义在没有 provider 的
  情况下也被覆盖。一个集成测试省略桩，让真实的 `runReviewSubagent` 创建委派子会话（faux 模型、
  脚本化裁决），并端到端演练子会话创建 → request_changes → 规范 finding 解析 → 重审 → approve。
  `requestReview.test.ts` 钉住纯解析/组合规则。
