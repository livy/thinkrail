# reviews —— 文件/diff 上的草稿评论 + 评审会话

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

评审层：GitHub 风格的**草稿评论**，锚定到工作区的文件与 diff，在不启动 agent 的情况下收集，然后作为
**结构化上下文包**发送——按**文件**分组，每个文件的评论进入该文件的那一个评审会话，或全部进入某次
发送所点名的那一个会话。拥有逐工作区评审 store、锚点重新锚定以及包渲染。

## 模型（Model (mirrors the wire DTOs in `contracts`)）

- **每个工作区一个打开的 `Review`**（在第一次读取/评论时惰性自动创建）。惰性创建会等待 git（被钉住
  的基点经异步作用域解析器解析），因此它是**每个工作区单飞的**：否则未加锁的 `review.get` 读取与
  一个加锁变更在这段窗口内竞争，会各自保存一份不同的新评审，后者悄悄替换掉另一方的（可能已被变更
  过的）快照。Clear 在解析其替代者之前加入该创建飞行，因此一次初次的 `review.get` 永远不会把一个
  未经宣告的快照保存在 Clear 所发布的那份之上。`freshSnapshot` 在其 git await **之后**重读工作区：
  移除会在任何东西持久化之前抛出，而一个变化的 diff 身份（`worktreePath`、`baseBranch`、
  `diffBase`）会重试基点解析，而不是把新评审钉在旧目标上。创建是一次快照趟可以跨越的**唯一**
  await：所有针对打开快照的 load→mutate→persist 都同步运行。变更辅助函数在有已打开快照时立即使用
  它，只在必须加入惰性创建时才 await；它们绝不把已加载的快照包进 `Promise.resolve` 并带着过期状态
  让出。未加锁读取的重新锚定趟（`getReviewSnapshot`）遵循同一规则。`addComment` 在取得快照*之前*
  解析其不可变的基点侧 ref，但只在最终同步的 mutate/persist 趟内捕获可变的 worktree 锚点；
  `clearReview` 在加载它要归档的活跃快照*之前*解析新基点——因为评审锁只覆盖变更，而一个跨越 await
  持有快照的趟会保存覆盖掉某个并发写入者（一次加锁变更、`rollbackSend`、一次 agent 解决）在这段
  空隙中持久化的内容，悄悄删掉它。这条纪律仍允许的唯一损失是一次加锁变更（其快照早于它的若干
  await）覆盖一次重新锚定持久化——良性：锚点状态是从工作树内容推导的，并在下次读取时重新计算。
  线上 **`review.close` 就是 Clear 操作**：在宿主的 workspace review 锁之下，`clearReview` 先把
  当前评审的非草稿记录作为一份关闭快照持久化到
  `reviews/archive/<workspaceId>/<reviewId>.json`，然后用一份全新的打开评审替换活跃快照，并且只
  发布那份新快照——客户端永远不会收敛到一份中间的已关闭副本。草稿被丢弃；已发送/已解决/已驳回的
  记录存活。没有归档浏览器，但在途的 agent 仍可按 id 解析一条已发送的归档评论。
  `Review.fileSessions` 把每个评审 KEY 钉到它的聊天（key → sessionId）：在评审的生命期内每个文件
  一个会话——该文件的首次发送创建它，之后每次发送（单条或批量）都跟进到其中。key 是评论的路径，
  或对无锚点的整体变更集备注而言是**空字符串**，像文件一样被精确钉住（`reviewSessionKey`），因此
  第二条整体备注继续那段讨论，而不是开启一个没人能跟进的会话。`Review.doneFiles`（相同的 key）
  标记用户已完成评审的文件：一个完全解决的文件会留在评审中，直到 `markFileDone`（线上：
  `review.fileDone`，在还有任何未解决内容时被拒绝）表示“这里的活干完了”——而该文件上的一条新评论
  会清除这个标记（`addComment`），重新打开它。`Review.baseSha` 钉住**被评审 diff 的原始一侧**——
  分支范围的 `originalRef`，即**分叉点**（diff 目标与 `HEAD` 的 `merge-base`）——**在创建时一次性
  钉到一个完整提交 oid**。不是 `diffBaseRef` 的尖端：分支 diff 展示的是分叉点对工作树（merge-base
  语义），因此一个已经越过某个分叉工作区而前进的目标，其尖端携带本次评审从未展示过的上游提交。
  刻意是分支范围，无论 Changes 面板显示哪个作用域——一次作用域切换绝不能重新定义评审*是什么*
  （在另一个作用域做的评论仍引用它自己的 `anchor.baseRef`）。同样出于不可变的理由：目标在评审中途
  可被重新指向，且它的分支可能移动。当该 ref 无法解析时它降级为原始 ref，因此评审界面在一个不可读
  的基点上存活下来而不是随之消失。`reviewBaseRef` 是 `host` 读取它的方式。
- **路径是工作树相对的 POSIX 路径**（架构决策 22）。`addComment` 通过 fs 的
  `workspaceRelativePath` 存储 `anchor.path`，因此一条引用绝对路径或原生路径的 finding 会落到与该
  文件 diff 及其 `doneFiles` 条目相同的 key 上；工作树之外被拒绝。`setReviewGuide` 对阅读顺序步骤
  做同样的事，并把外部路径按原样保留（guide 把它显示为禁用）。文件读取共享这一包含关系，因此像
  `..env` 这样的名字是一个文件，而不是逃逸。
- **`ReviewComment`**：`kind` 为 inline/diff/file/review；`status` 为 draft → sent → resolved（或
  dismissed）；**`author`**——人类（默认，缺省）或 `"agent"`：计划的评审人 agent 通过同一个
  `addComment`（工作树侧锚点）提交 finding，在面板上加徽章，并被同一套发送机制接手（TODO 修复包
  走 `buildSendPackage`，见 [[submodule-server-todos]] 的 agent reviewer 一节）。线上
  （`review.commentUpdate`）只能落地终态的人工结果（resolved/dismissed，可从 draft 或 sent 出发）；
  `draft`↔`sent` 的迁移专属于发送路径（`markCommentsSent`/`rollbackSend`）——一个能撤销发送的
  客户端就能改写或删除一条某个 agent 会话已经引用的备注；`sessionId` 链接评论被发送进去的那个
  会话——即其文件的评审会话。**一条评论一旦被发送就是记录**：草稿——用户自己未发送的草稿——仍可被
  删除（`review.commentDelete`，仅限草稿，否则拒绝），但已发送的评论永不被删除——Clear 在替换活跃
  评审之前把那条记录移进关闭归档——并且评审不提供工作树改动的回滚（旧的 `git.revertFile` Reject
  已移除）；对一个变更提出反对的方式就是在评论里说出来。
- **`ReviewAnchor` = `path` + `side` + `contentHash` + 一条有序的 `selectors` 回退链**
  （`lineRange`、带 exact/prefix/suffix 的 `textQuote`、对文档节点的 `structural` scheme+ref、对
  归一化 `0..1` 几何的 `region`；Ask-agent 用实际展示的 hunk 头填充 `diffHunk`，而 `textQuote`
  承载重新锚定）。`anchorState` 轴（`anchored`/`moved`/`outdated`）与 `status` **正交**：“它是否被
  讨论过”与“锚点是否还活着”永不互相覆盖。**`contentHash` 是资源字节的 sha-256**
  （`fs.hashBytes`），且捕获跟随资源而非评论：一个**文本**侧（`fs.classifyBytes`——无识别的二进制
  魔数、前 8 KiB 无 NUL 字节，且严格 UTF-8 解码；BOM 算文本，SVG 算文本）像以前一样从 `lineRange`
  推导 `textQuote`，而一个**纯字节**侧对字节做哈希并**按给定原样**保留渲染器的 selectors，绝不推导
  quote——因为没有文本可引用。`addComment` 在两种捕获之前**收窄线上的 selectors**——每个元素都是
  已知 `kind` 的对象，且每个声明的字段都做类型检查（`region` 的分量为有限值且在 `[0, 1]` 内，
  `page` 是正整数；`structural.ref` 是非空字符串且 `scheme` 匹配 `/^[a-z][a-z0-9-]*$/`；
  `lineRange` 为整数、1 起始、不反向）——并存储*那个*收窄后的值，因此未声明的字段无法随线上数据
  进入 store。它除形状之外不检查任何东西：一个未知的 structural scheme 属于本宿主从未听说过的渲染器，
  会被保留，而不是拒绝。
- **两个 diff 侧是两个锚点空间。** 一个 `side: "worktree"` 锚点从工作树文件捕获；一个
  `side: "base"` 锚点从 diff 的 ORIGINAL 编辑器所展示的那个 blob 捕获——宿主从标签页的 `scope`
  解析它（`resolveDiffRange(...).originalRef`），**把该 ref 钉到一个完整提交 oid**
  （`git.resolveCommitOid`）并作为 **`baseRef`** 盖在锚点上，使该片段及其上下文在评审的生命期内
  保持可读——并且保持相同。钉住是承重的那部分：作用域的 `originalRef` 对 `uncommitted` 是符号性的
  （字面量 `HEAD`），并在 `merge-base` 失败时降级为原始基点 ref，所以逐字存储它意味着用户的下一次
  提交就把它重新指向，包会读着今天的内容却配着昨天的行号。一个不指向任何提交的 ref 会被直接拒绝，
  而不是锚定到某个会移动的东西上。两侧在捕获时都按**字节**读取（基点侧用
  `git.readBlobBytesAt`，工作树侧本来就是这样），因为一次解码读取会为一张图片哈希替换字符，并丢掉
  工作树侧保留的 BOM。基点侧选择**永不翻译成工作树行号**：两侧在相同的行号上讲不同的事，因此对被
  删除或被重写行的备注会落到如今占据该位置的任何东西上——而*那*正是发送包会展示给 agent 的内容。
  一个路径根本不在基点中的基点锚点会在一开始就被拒绝，绝不悄悄重新指向。

## 重新锚定（Re-anchoring (the file changed under a comment)）

每次快照读取时以及任何发送之前（`reanchorWorkspace`）重新计算，针对工作树：
1. `contentHash`（该文件字节的 sha-256）未变 → `anchored`。
2. 否则搜索 `textQuote.exact`（用 prefix/suffix 消歧）→ 恰好一个匹配 → 更新 `lineRange`，状态
   `moved`（静默重新钉住）。**`moved` 是粘性的**：重新钉住会刷新锚点的 `contentHash`，因此紧接的
   下一趟否则会看到哈希匹配并悄悄把它降级为 `anchored`——丢掉该状态所记录的“自创建以来已漂移”这一
   事实。
3. 无匹配/匹配歧义或文件消失 → `outdated`；评论保留其创建时快照（`textQuote.exact`），因此它仍
   有意义且仍可发送（包会把它标记为过期）。
4. **根本没有 `textQuote` 可搜索**（一个纯字节资源，或一个不基于文本锚定的渲染器）：一个**有位置
   的**锚点——任何 `lineRange`/`structural`/`region` 选择器——是 `outdated`，因为重新钉住需要证据，
   而一张变化的图片或 notebook 提供不了任何证据；一个**整文件**锚点（无选择器，即 `file` 评论
   类型）保持 `moved` 并刷新哈希，因为该备注是关于文件的，而不是关于文件中某个位置的。这是刻意的，
   而且刻意严格：一张重新编码或元数据被动过、尺寸相同的 PNG *大概*仍让该备注的主题位于相同的归一化
   区域内，但“尺寸相同”并不是那些像素相同的证据，而 `outdated` 的全部意义就是“在信任这个位置之前
   再看一眼”。曾考虑过一条“尺寸保持 → `moved`”的规则并被否决，因为宿主那时将断言一个它无法检查的
   视觉事实——diff 查看器（2-up、swipe、onion、difference）才是那次检查该待的地方，而一条
   `outdated` 线程仍在其区域上被绘制、仍可发送，并仍携带其创建时快照。客户端的 UI 文案点明原因
   （`outdatedReason`）：无法再次找到的文本、位置之下发生变化的字节，或一个已消失的文件。
`side: "base"` 锚点永不重新锚定：`baseRef` 是一个提交 oid，因此它命名的 blob 不可变，没有什么可
漂移——而对*工作树*重新锚定它们正是逐侧捕获所要防止的那种重新指向。

## 发送流程与上下文包（Send flows & the context package）

`review.sendComment` / `review.sendBatch` 在**`host` 的处理器中组装**（本模块从不导入 `agent`）：
重新锚定 → 渲染包（一条结构化用户消息，带稳定的评论 id，逐评论的片段 + 周边上下文——或者，对一个
由几何或文档节点而非行来命名的位置（一个栅格或 SVG 区域、一个 notebook 单元格），一行 `<locator>`
命名该区域几何或 `<scheme> <ref>` 节点，因为没有行跨度可引用；每条评论都携带 `anchor-kind`
（`region`/`structural`/`line`/`file`），因此 agent 把几何读作几何——绝不是完整 diff。**每个动态
属性和 locator 值都被实体转义**（`& < > "`，CR/LF 用数字引用），因为一个仓库路径或渲染器的节点 ref
是不可信文本，否则可能关闭一个标签并伪造一条用户从未写过的评论；片段、上下文与正文保持逐字，因为
它们正是 agent 需要按原样阅读的内容——而 Web 气泡卡把相同的转义解析回来（见
`apps/web/src/chat/SPEC.md`）。agent 用自己的工具读取工作树；**每一侧读自己的内容**——工作树锚点读
工作树，基点锚点读锚点的 `baseRef` blob，因为基点行号索引的是变更前的文件）→
`agent.createSession`（或在一次发送点名了某个会话时 `followUp` 进客户端的**最后一个打开的会话**
——即用户屏幕上已有的对话——否则进入该 KEY 已钉住的会话，**当它不活跃时从磁盘重新附加**，因为评审
状态与 pi transcript 都能在宿主重启后存活；一次跨越多个 key 的批量发送会分别发送每一组并全部应答，
因此没有任何一组被留在无人看见地运行；无论谁收到这个包，都成为该 key 的钉子）→ `markSent` →
prompt。`markSent` 要求每个被请求的 id 仍指向活跃快照中的一个草稿——在生命周期漂移之后，它绝不悄悄
标记一个部分集合。整个序列**与每一次评审变更一起按工作区串行化**（`host` 的 `withReviewLock`）：
草稿/会话检查发生在被等待的会话创建*之前*，因此在那段空隙中两次并发发送都会看到“有草稿、无会话”
并把评审分叉——而一次并发的 `review.close` Clear 会使已经构建好的包失效，留 agent 拿着一些任何
打开的评审都不包含的评论 id。**prompt 以分离方式触发**（`fireReviewPrompt`）：处理器在会话存在的
那一刻就返回，好让客户端立刻打开聊天——等待 ack 意味着每次发送都要坐等 pi 的 10 秒接受窗口。因为
`markSent` 是在锁内、在轮次被确知接受之前被等待的——它必须如此，好让该 key 的钉子在锁内存在，且
并发发送无法分叉聊天——一次轮次前拒绝（模型不对、key 缺失/过期）既会在刚打开的聊天**内部**以
extension-UI 通知的形式出现，**又会把评论回滚为 `draft`**（`rollbackSend`，以 `ackSend` 的
接受-对立-拒绝窗口为键）：一次 agent 从未收到的评审保持可重试，而不是搁浅为 `sent` 并丢掉其
send/edit/delete 动作，而为那次失败发送专门启动的会话会被解除钉住，除非还有另一条评论支撑它。接受
**之后**的故障是真正的轮次故障（包*确实*已送达），走事件流，让 `sent` 状态保持正确。回滚以分离
方式运行（在发送的锁释放之后）且完全同步，因此——像 `reanchorWorkspace` 一样——它在无锁下仍然正确；
并且它用 `load`（绝不是 `ensureSnapshot`）读取：一次先落地的 `review.close` Clear 会让它对新的评审
成为干净的空操作，而不是复活被清除的评论。**`resolve_comment`** 能力是 agent 模块的一个自定义工具
（`agent/reviewTool.ts`，像 `ask_user_question` 一样在每个会话上注册），其执行通过一个宿主安装的
接缝被委派回这里——agent 模块保持无依赖。**与会话绑定**：该工具把
`ctx.sessionManager.getSessionId()` 一路线程化到 `resolveCommentFromAgent`，而
`applyAgentResolution` 只解决 `status === "sent"` **且** `sessionId` 等于调用方（即
`markCommentsSent` 记录的实际接收者）的评论。一个 `draft` 评论通过这个工具无条件不可解决（评论只能被真正
收到它的那个聊天解决；任何东西——agent 或人——都不能解决自己未发送的草稿）——正是这一点让评审人
agent 无法提交一条 finding 后立即自己清掉它（见 [[submodule-server-host]] 的 approve 门禁）。解决
先搜索活跃快照，再搜索关闭归档，因此一次在 Clear 落地时已在途的工具调用仍能完成其记录；归档更新会
持久化，但不发布非活跃快照。

## 边界（Boundary）

- **拥有：** 数据目录下的活跃 `reviews/<workspaceId>.json` 以及位于
  `reviews/archive/<workspaceId>/<reviewId>.json` 的关闭记录归档（经 `persistence.dataDir`；id 会
  成为路径段，因此每次文件触碰都拒绝带路径段的 id——`/^[\w-]+$/`——否则一个线上提供的
  `../config` 式字符串会让读取/写入/删除指向 reviews 目录之外：这是处理器自身查找背后的纵深防御）、
  评论 CRUD + 状态/生命周期迁移、锚点捕获 + 重新锚定（纯函数、有单元测试的 `anchoring.ts`）、包渲染
  （纯函数 `packageRender.ts`），以及 `review.changed` 发布者接缝（`setReviewPublisher`，由 `host`
  安装——全快照推送，在最后值重放之下幂等）。
- **公开接口（barrel）：** `getReviewSnapshot`、`addComment`、`updateComment`、`deleteComment`
  （仅草稿）、`clearReview`、`markCommentsSent`、`rollbackSend`（在一次轮次前发送拒绝时撤销
  `markCommentsSent`）、`markFileDone`、`fileReviewSession` + `reviewSessionKey`/
  `REVIEW_LEVEL_KEY`（逐 key 的会话钉子）、`resolveCommentFromAgent`、`reanchorWorkspace`、
  `sendableComments`、`buildSendPackage`、`removeWorkspaceReviews`、`setReviewPublisher`、
  `setReviewGuide`（最新计划步骤裁决的叙述——摘要、建议阅读顺序、裁决、来源——存储在打开评审上的
  `review.guide`，并随快照扇出，因此 Changes 评审标签页可以走它而无需第二次读取；由 `host` 的裁决
  记录器写入，绝不由客户端写入）（+ 纯锚定/渲染辅助：`reanchor`、`buildTextQuote`、`hashContent`、
  `lineRangeOf`、`textQuoteOf`、`renderPackage`）。
- **允许的依赖：** `contracts`（类型）、`persistence`（数据目录）、`log`、`workspaces`（工作树路径
  查找）、`git`（评审的 `baseSha` 解析、基点锚点 `baseRef` 背后的 diff 范围，以及基点侧的 blob
  读取）、`fs`（`classifyBytes`/`decodeText`/`hashBytes`——文本性与锚点 `contentHash` *所是*的
  sha-256 在那里一次性决定，因此一条评论的哈希与一次 revert 的期望对同一批字节永不产生分歧）、
  Node 的 `fs`。
- **禁止：** 导入 `host`/`agent` 或任何 pi 包；除通过该接缝外发布。

## 必须做对（Get right）

- **状态通过推送收敛，绝不乐观**——每次变更（UI 编辑、agent 解决、一次改变了状态的重新锚定）都发出
  一份完整 `review.changed` 快照；客户端折叠它。Clear 恰好发出新的打开快照（绝不是被归档的评审），
  因此每个已连接客户端一起清空。关闭归档先被写入；一次失败无法在不保留其非草稿记录的情况下替换活跃
  评审，而重试按评审 id 幂等。
- 发送**先重新锚定**，因此包中的行号在发送时是真的——并且**与每次变更一起按工作区串行化**，所以没有
  东西能在一次先检查后标记之下关闭或重发。一个引用了某评论 id 的包，是对该 id 仍然存在的承诺。
- **片段从它被捕获的那一侧读取。** 基点锚点永不读工作树，工作树锚点永不读 blob：给 agent 看错误的
  一侧，从它的视角看与用户说错了话无法区分。基点锚点还携带它被捕获时的 **`scope`**（紧挨着解析出的
  `baseRef`）——即 Review 侧栏重开该备注自身界面所依据的 diff 身份（见 `panels/SPEC.md`）。
- **持久化永不因文件损坏而丢掉一份评审。** 写入是**原子**的（临时文件 + rename，因此写入中途被杀
  的宿主不会留下截断的评审），而读取只把 **`ENOENT`** 视为“没有评审”：一个无法解析——或根本无法
  读取——的文件会**抛出**并被留在磁盘上，因为唯一对“缺失”采取行动的调用方（`ensureSnapshot`）的
  响应是在其上写一份全新的空评审。故障在 `review.get` 上呈现（面板会说明），而不是悄悄丢弃每一条
  评论。唯一的例外是 agent 解决背后的跨工作区扫描：一个损坏的*兄弟*文件会被记录并跳过，绝不被允许
  让一次属于健康评审的解决失败。
- 一个**未知/重复的 agent 解决会响亮失败**（错误文本回到模型），绝不静默。一次在 Clear 之后到达的
  解决会原地更新归档记录，且不发出活跃评审推送。agent 解决接受 `sent` 评论**以及 agent 撰写的
  `draft`**：在自动修复关闭时，评审人的 finding 被记录但从不发送，而重新评审包仍指示评审人解决每条
  修复所针对的 finding——一个仅限已发送的门禁会让那条指令无法满足，并让这些 finding 搁浅在一个已
  批准的条目之下。人类草稿对 agent 仍不可解决（它们是用户未发送的草稿）。
- 工作区移除会清除活跃评审及其归档目录（`removeWorkspaceReviews`，由工作区归档处理器调用）。
