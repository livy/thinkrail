# agent —— 进程内 pi 会话

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

进程内的 `pi` 引擎：一个用于会话前工作与未来聊天的当前共享模型/认证运行时世代，`AgentSession`
的生命周期（每个聊天标签页一个，根植于某个工作区的工作树，并保留其创建时所用的运行时世代）、
Pi 资源/skill 加载（含可移植的跨 agent skill 发现 + 会话前 skill 目录）、把 pi 进程内的
`uiContext` 对话框调用转成 WS 帧的**扩展 UI 桥**、ask-user-question 扩展背后的
**`ask_user_question`** 答案信箱与答案注入路径，以及让重新打开的 transcript 保持 provider
合法的**重启修复**。

## 边界（Boundary）

- **拥有：**
  - `piRuntime`（当前的共享 `ModelRuntime` 世代——pi 为目录、凭据、可用性、登录/注销和请求
    分发提供的规范模型/认证门面）。同级消费者使用 `usePiRuntime()` 回调来捕获当前世代，而不是
    接收一个可变单例；宿主启动在任何模型工作之前初始化它，而后续的 Central 工件变更会准备并原子地
    激活一个新世代。测试在初始化之前配置工厂。每个运行时都以**环境网络关闭**的方式创建——
    `allowModelNetwork: false` **加上构造期间一个作用域内的 `PI_OFFLINE`**（pi 0.81 在构造时从该
    环境变量推导运行时的环境网络默认值；该选项现在只门控创建时的刷新——在 0.80.x 中它同时供给
    两者；作用域内的值随后立即恢复，用户自行设置的值不受影响——由 `piRuntime.test.ts` 固定）：
    目录读取保持本地（内建 + models.json + 持久化的 models-store），因为启用网络的 `refresh()`
    （pi 0.82 把旧的 `reloadConfig()` 折叠进来）会等待远程 pi.dev 目录检查且没有超时——在
    `provider.status` 与宿主启动路径上，只要出口慢或被阻断就会卡住。对实时目录唯一刻意的选择加入
    是单飞的 **`refreshCatalogs(runtime)`**（issue #98，模仿 pi 自己的 `/model`），由两个触发点
    驱动：一个只来自 `model.list` 的分离任务（`listAvailableModels` 触发它，然后提供当前快照——
    选择器的读取从不等待网络；更广的触发点——`model.default`、新聊天默认值解析、宿主启动——曾被
    考虑并否决，因此它们通过 `listSettledModels` 读取快照），以及通过 `model.refresh` 的
    **等待式**调用（`refreshAvailableModels`，选择器的新鲜度入口：等待刷新，然后提供刷新后的快照
    **并带上 `complete`**——`refreshCatalogs` 解析出一个 `CatalogRefreshOutcome`，说明它所等待的
    那一轮是否结算，该判定作为 `RefreshedModels.complete` 传到客户端，因为一次有上限的等待只能
    承诺一份*当前*的列表，而不是已结算的列表，而目录权威必须以此差异为键）。每次调用
    `refresh({ allowNetwork: true, force })`，其中 **`force` 是调用方的意图，而不是常量**：
    *隐式*触发（`model.list`、打开选择器）不开启它，由 pi 的 **4 小时 provider 新鲜度节流**决定
    是否抓取任何东西；而*用户发起*的刷新（选择器的 Refresh 行 → `model.refresh({force:true})`）
    绕过它——在该窗口内 pi 会在**发出任何请求之前**提前返回（其 `If-None-Match` 重新校验也不
    例外），所以一次未强制的显式刷新将什么都抓不到。**按运行时实例单飞**（pi 的 `refresh()`
    不对并发调用去重）**并以种类为键**：隐式调用方加入任何一轮，强制调用方绝不加入被节流的一轮
    （否则它会继承那次空操作），而是排队在它之后。**15 秒预算**（pi 的模型选择器所用的那个）被
    应用**两次**，两次都落在 **unref 的**定时器上（绝不能拖住一个正在关闭的宿主或测试进程）：
    作为 `models.refresh` 的**中止**信号（挂起的刷新必须自行过期，否则单飞会被卡死）*以及*作为
    **调用方等待什么**的上限，因为该信号既不约束 pi 在其之后未受信号约束的
    `forceRefreshAvailability()` 扇出，也不约束排队在被节流轮次之后的强制轮次——没有它，一个慢
    provider 就会让每个选择器的刷新行一直转圈。超时的调用方按注册表现状提供（报告
    `completed: false`），同时单飞继续跟踪那次无界轮次（因此它无法启动第二次并发刷新）；失败只
    发出封闭的通用/计数警告日志（绝不包含 provider id 或错误）+ 被吞掉，绝不是选择器的问题；
    **`PI_OFFLINE`**（pi 的环境变量约定）会禁用它——解析为一次*已完成*的轮次，因为在没有任何可
    抓取内容时，注册表现状就是已结算的答案；e2e webServer 环境与管理器的单元测试套件为封闭性
    设置了它。这个运行时之上的 **provider 凭据接口**——`provider.status` + 应用内登录——位于同级
    的 `auth` 模块（它消费共享的 `usePiRuntime` 回调），**不**在这里。

    候选准备只接受经过审查的不透明 Central 路径集合，构建一个新运行时，应用组合根的不变量世代
    初始化器（源码模式的 e2e 宿主用它来实现其受门禁的假 provider），为 `provider.status` 记录该
    不透明化之前的 provider-id 允许列表，连同当时每个 id 的显示名称，然后通过 PI 的公开无头加载器
    一次性应用不透明扩展。该世代另外通过不透明注册身份比较（绝不通过配置值）记录该加载器引入或
    替换的 id（`opaqueProviderIds`）。允许列表保持完整的不透明化之前的集合：一个新的 Central id
    从不在其中，因此永远不会成为一行，而被替换的内建 provider（`anthropic`、`openai`、…）仍然
    可见，auth 通过求两个集合的交集把它归因于 Central。（从允许列表中减去不透明集合——曾短暂用于
    分析归因——使得每个经 Central 路由的 provider 从 Providers 区段消失；这两个事实必须保持分离。）
    因此 auth 从不检查或输出 Central 的 provider 配置，而新增/移除/替换也绝不会丢掉进程本地的
    provider 注册。初始化器必须在第一个世代之前配置，并对每个候选运行。路径是本模块收到的唯一
    工件事实；它从不读取、解析、哈希、快照、记录、复制或提供该文件。初始化器/扩展/加载器/provider
    失败会丢弃候选并折叠为封闭的 `load-failed` 结果；原始诊断绝不进入 `pi.extensionUi`、线上契约、
    日志、分析、持久化或快照。Auth 拥有文件监视、合并与陈旧候选拒绝；agent 只准备并激活一个世代。

    `getSessionRuntimeGeneration` 暴露一个活会话所保留的世代，供宿主的同步元数据读取使用。激活
    会改变用于会话前读取和未来会话创建的当前指针；它绝不修改、清空或重建已存在的会话。活会话保留
    其原始运行时世代。激活之后挂接的磁盘会话，在该会话保存的模型于当前运行时中可用时保留该模型。
    若不可用，Pi 先选它的 settings 默认值，再选它按序的 provider 默认值，然后选第一个可用模型；
    这不必匹配 ThinkRail 的 AppConfig 默认值，也不必匹配保存的 provider。例外是 pi 仍然列出、但
    账号并不提供的模型（`modelAvailability` 过滤掉了它），否则 pi 会逐字恢复它；此时宿主改传
    `preferredDefaultModel`。打开不改变保存的模型。一次普通的物理模型回退在 Pi 追加助手回复时变得
    持久，而不是在发送被接受时；显式选择模型则立即持久化。仅仅打开一个聊天会保留其原始模型，以备
    该 provider 恢复。宿主在正常的会话状态中暴露替代模型，但丢弃 SDK 的 `modelFallbackMessage`，
    因此没有单独的回退警告。没有任何可用模型时，transcript 仍可读，也不会把占位模型持久化为替代品。

    每一次模型**读取**都经过 **`settledAvailableModels(runtime)`**——pi 的 `getAvailableSnapshot()`
    经 `modelAvailability` 收窄，**绝不用 `getAvailable()`**：后者会等待 `refreshAvailability()`，
    而它要么返回待处理的按 provider 认证扇出*要么启动一次*，全都未受信号约束——所以通过它读取就
    等于把 `model.list`（其契约是在不碰网络的情况下作答）、`model.default` 和每一次入站模型引用
    检查都交给一次无界等待，并且在应用刷新截止时间的一行之后又绕过它。该快照是 pi 上一次*已结算*
    轮次得出的结论（在 `create()` 时、每次 `refresh()` 之后以及登录/注销时写入），再加上 pi 在某个
    带有已存凭据或已配置 key 的 provider 注册那一刻写入的一条临时条目——配置注册与原生
    （`registerNativeProvider`）注册一样，因此一个 Central 或 delegation 镜像来的 provider 在其
    可用性轮次落地之前就可读。因为它是唯一的读取入口，选择器、默认值与模型解析在同一世代内保持一致。
  - `modelAvailability` —— 把该快照收窄到**连接**实际提供的内容；pi 的目录是按 provider 的，而不是
    按账号的，所以一个 pi 认为已配置的 provider 仍可能在其目录的大部分上返回 400。两条规则，各自以
    `baseUrl` 为范围，因此自定义或改道的模型不受影响：**Sign in with ChatGPT**（`openai` + 对
    `api.openai.com/v1` 的 OAuth）只保留 pi 自己 `openai-codex` 目录中的 id——ChatGPT 后端以
    `model_not_found` 拒绝纯 API 的其余部分（`gpt-4`、`gpt-4o`、`o1`、…）；而一个**不透明
    （Central）provider** 只保留其代理在 `GET /models`（OpenAI 线上契约）或 `GET /v1/models`
    （Anthropic 线上契约）上公布的 id，以及能匹配某个已列出 `-YYYYMMDD` / `-YYYY-MM-DD` 日期化
    id 的别名（代理会把 `gpt-4o` 解析为 `gpt-4o-2024-11-20`）。列表与 `refreshCatalogs` 并排地
    用 pi 解析出的认证读取（在 `model.list` 上分离、在 `model.refresh` 上等待并强制），单飞，
    端到端 5 秒有界（一个截止时间覆盖凭据解析与抓取，因此卡住的 OAuth 刷新无法卡死单飞），每个
    运行时节流到 60 秒，变更会发布 `provider.changed` 以便选择器重新读取；未知的线上契约或从未
    读取过的列表不做任何过滤，重读失败则保留上一次列表。失败只记录一个封闭的计数，绝不记录
    provider id 或 provider 的错误文本。直接 API key 与其他订阅保留 pi 的目录：没有任何列表被
    验证过在那里与 pi 的别名一致。
  - `modelDefaults` —— `preferredDefaultModel(available)`：按 pi 的顺序取 pi 的
    `defaultModelPerProvider`，然后是任何 id 属于这些默认值之一的可用模型，否则取第一个可用模型。
    Pi 不导出那张映射表，所以宿主在一项针对已安装 dist 的对等测试之下镜像它；第一遍正是 pi 自己的
    `findInitialModel` 会做的选择。同 id 那一遍覆盖了自身默认值被账号过滤器移除的 provider：
    Sign in with ChatGPT 从 `openai-codex` 集合提供 `openai` 模型，所以它会落到 Codex 默认值上，
    而不是目录顺序。
  - `modelContext` —— 针对 pi 共享的 `getAgentDir()/models.json` 中 `contextWindow` 覆盖的窄
    settings 适配器，而不是第二个模型注册表。它从 `settledAvailableModels` 中列出 OpenAI
    Responses / Codex Responses 模型，条件是有效预算仍等于契约中 272K 的编辑下限（该设置正是为了
    解除这个上限），外加任何已带覆盖的模型，以便总能回退；每个条目把 pi 的有效 `contextWindow`
    与该文件的显式 `override` 配对（`null` = pi 默认值）。资格由 pi 的目录推导，绝不来自某个模型
    id 列表，因此超过 1M 的目录默认值不会被批量保存触碰。保存先校验契约的 272K–1M 范围，然后用
    `jsonc-parser` 为目标修补 `providers.<p>.modelOverrides.<id>.contextWindow`——对于
    `available` 则是契约的 `isSharedModelContextTarget` 准入的每一对——因此共享保存绝不会重写
    只有定向保存才能触碰的范围之外的覆盖——这样注释、BOM 和不相关的配置都能存活；它会修剪被清空的
    override 对象（绝不修剪 `providers`），对无效 JSON 或只读文件拒绝修补而非强行修补，然后在
    解析后的目标旁边写一个临时文件（沿用原文件的 mode，新文件为 `0o600`）并重命名就位——若是
    符号链接的 `models.json` 则跟随到其目标，并为悬空链接创建其目标而不是替换链接。保存是进程内
    串行的；文件发生变更后会跟随一次在当前世代上禁用网络的 `runtime.refresh` 和宿主的
    `provider.changed` 信号；空操作的保存两者都不做。每次读取也会先在本地刷新，因此外部 pi CLI 的
    编辑可见。活会话保留其创建时的模型；新聊天解析更新后的元数据。pi 拥有模式校验与有效值；错误
    消息是通用的，绝不携带文件内容。已接受的缺口：Central 重建时若其运行时在保存之前已加载该文件，
    而在其刷新之后才激活，则会继续提供旧预算，直到下一次读取刷新它。
  - `agentSessionManager` —— 会话以 `session.sessionId` 为键（每个 `Entry` 还跟踪自己的
    `workspaceId`），`createSession({ cwd, workspaceId, model?, thinkingLevel? })` →
    `createAgentSession(...)`，配有按会话的 `SessionManager` **和 `buildSessionSettings(cwd, projectTrusted)`**
    （用户的真实 settings + 一个内存中的 `images.autoResize:false` 覆盖——绝不持久化——因此 `read`
    工具以**原始**形式发送图像文件，绕过单文件二进制无法打包的 pi photon/WASM 缩放器；web UI 在
    挂接时自行缩小用户附加的图像——`apps/web` 的 `chat/imageAttachment` 把长边限制在 1568px——
    而下面的 `imageGuard` 扩展是上下文内的第二道防线）。该覆盖在**每次 `settings.reload()` 之后
    重新应用**：pi 的 `SettingsManager.reload()` 会从磁盘重建 settings 并丢弃 `applyOverrides`，
    而资源加载器在每次 `reload()` 时都会重载 settings——包括在 `createAgentSession` 内部——所以
    一次性覆盖永远到不了 prompt。自 pi 0.87 起，同一设置也支配 prompt 附加与工具结果图像，因此
    该覆盖正是阻止 pi 用 `[Image omitted…]` 提示重写用户文本的原因（那会破坏客户端的乐观回显
    去重）。被委派的子会话运行在同一批 settings 上：`delegation` 把 `buildSessionSettings` 绑定
    为核心的 `buildChildSettings`，它接收父上下文 `isProjectTrusted` 所回答的信任
    （`liveParentContext`；pi-delegation 决策 #31，issue #604）。一个共享的 `registerSession`
    发布每个事件，并打上其 id + `bindExtensions({ mode:'rpc', uiContext })`。事件投影保留最终
    `agent_end` 助手所报告的终止元数据并附加到 `agent_settled`，这样即便在这些事件之间发生压缩/
    重试，线上契约也只有一个权威的自动工作终止点；它转发而不是重新推导 pi 的结果。
    `compaction_end` 被单独投影为一个**新的允许列表事件**：其 `result` 只携带 `tokensBefore` 和
    可选的 `estimatedTokensAfter`，绝不包含 pi 的摘要、条目 id、用量或扩展细节。工具的
    `tool_execution_end` / `tool_execution_update` 帧从（部分）结果中丢弃 `structuredContent`：
    那是 pi 为程序化调用方提供的值（bash 最多在那里放 1 MiB 原始输出），从不持久化也从不渲染，
    所以转发它只会让每个实时帧和浏览器 store 变大。`entry_appended` 只投影为 `{ entryType }`：
    扩展与 pi 内部条目的载荷绝不跨过线上契约。活条目在 `SessionSummary.lastSettlement` 中保留该
    结算，以便在 Pi 从其重建的上下文中移除失败尝试之后重连使用；新的 `agent_start` 暴露显式的
    `null`（无当前终止点），使较早持久化的失败不会在运行中途重新出现，而磁盘会话仍以 transcript
    为权威。活摘要还携带 pi 的队列快照（`SessionQueueState`，仅在非空时）：`queue_update` 只在
    变更时触发，所以这是读侧种子，让在运行中途挂接的客户端能渲染在其连接之前已排队的消息。
    每条队列 lane 也仅在 Pi 报告对应文本条目待处理期间保留浏览器排队消息的完整内容。这份临时镜像
    的存在仅仅因为 Pi 的破坏性队列 API 返回文本却丢弃图像块；Pi 的队列事件对成员关系/顺序仍然
    权威。宿主只把保守的 `hasImages` 聚合投影进摘要/事件，因此图像字节不会走常规读取流。

    **一处成员关系覆盖用于补偿一个 Pi 缺陷：一条已投递的、仅含图像（文本为空）的排队消息永远
    不会被清除。** Pi 把投递时移除队列条目的键取为所投递用户消息的文本（`contentText`），所以一条
    带图像但无文本的排队消息会永远留在 Pi 的 `_steeringMessages` / `_followUpMessages` 中，其
    "Steering"/"Follow-up" 标签永不消失。因此管理器按 lane 统计这类投递
    （`stuckEmptyDeliveries`）：当收到一条文本为空、内容携带图像的用户 `message_start`，且 Pi 仍
    报告某条 lane 中的空文本条目多于已统计数量时，它递增该 lane 的计数器，并**合成 Pi 未发出的
    清空 `queue_update`**。每一处为展示而读取 lane 的地方——投影后的 `queue_update`、临时镜像同步
    以及 `queueStateOf`——都先从 Pi 的列表中丢弃相应数量的空文本条目，这样后来的真实
    `queue_update` 无法复活那个幻影。计数器仅按活条目存在（Pi 的临时队列从不跨进程重启存活），
    并在 `clearQueue()` 清空两条 lane 时重置。

    **在带来上游修复的 pi 升级中移除整个覆盖**（earendil-works/pi#8612，最后检查时仍未关闭）。
    一旦 Pi 原生清除空文本图像投递，就删掉 `stuckEmptyDeliveries`、`displayedLane`、合成的
    `queue_update` 和 `effectivePendingCount` 调整。移除门禁以已安装的代码为准，而不是 PR 状态：
    每次 pi 升级都在已安装的 `agent-session.js` 中 grep 围绕 `this._steeringMessages.indexOf` 的
    `if (messageText)` 守卫——该守卫存在时变通仍然必需；它消失时修复就已交付。

  - **归一化的会话状态**是一个建立在正交事实之上的宿主投影，绝不是第二个 agent 运行时。活推导
    读取 Pi 的执行/队列状态、问题阶段注册表的确切期望/等待 id、扩展对话框注册表、最终结算和
    显式停止意图。展示时，需要输入优先于进行中，而底层执行可能仍在运行。只有 `agent_settled`
    才产生一次正常完成；一个终止分类器区分成功、错误、长度、中止和缺失工作。显式停止在 abort
    之前把当前 run id 记为已取消并保持安静；重启后发现未完成的运行变为 interrupted。

    **后台工作持有未读。** 当一个空闲聊天仍拥有一个运行中/停止中的后台命令或一个排队/运行中的
    delegation 子会话时，其完成会被报告，但 `completionUnread` 保持 false，因此在 agent 的工作
    仍在进行时不触发注意点，也不发出应用外通知；需要输入绝不被持有。命令变更监听器与 delegation
    生命周期监听器在一个合并后的 `setImmediate` 上重新发布父会话状态，此时唤醒父会话的同 tick
    完成投递已把它翻成运行中——这样最后完成的子会话绝不会在唤醒轮次之前闪出注意点。不唤醒父会话的
    完成（用户停止）会在该重新发布时释放持有；被重载或回收站持有门控的投递同样先释放，其后续唤醒
    会再次清掉未读。仅有磁盘的会话没有活工作，从不持有。同一事实作为 `backgroundWork` 发布，
    因此客户端把空闲聊天显示为工作中（契约的 `isSessionWorking`）而不是悄悄完成。

    状态 id 来自 Pi 的活动会话条目链：run/中断来自最新的用户条目，完成来自决定性的助手条目，
    问题/对话框来自它们的交互 id。问题需要来自活信箱的输入；没有活阶段时，已挂接会话只把
    transcript 中一个已确认的调用读作需要输入（修复 ack），而磁盘状态仍把一个未应答的悬空调用
    读作需要输入，挂接时的修复会为它补 ack。磁盘状态从完整的活动分支重建；文件读取/解析失败会让
    整个全工作区快照失败，而不是省略某一行。回执初始化在元数据文件并非都能加载时运行（首次安装，
    或某个文件因不可读而被搁置，后者会记录日志）：它把已存在的完成 id 标记为已处理，但绝不抑制
    未解决的输入。因为它在开始提供服务之前运行，它是唯一的尽力而为读取者：不可读的 transcript
    或会话目录会被记录日志并保持未基线化，而不是阻塞宿主启动，而之后的快照会持续失败直到文件被
    修复。回执写入是串行且原子的。会话在语义变更时发布完整状态记录。待处理的扩展对话框保留其
    完整请求，以便重连客户端能渲染并回答确切的阻塞项，而不是看到一个不可用的需要输入标记。

    `listSessionStates` 返回每个顶层活/磁盘会话；`acknowledgeCompletion` 只对当前确切的那条未读
    完成做比较并设置；`markWorkspaceUnread` 丢弃该工作区中最近活跃会话（最新的分支条目时间戳）
    里 `canMarkCompletionUnread` 准入的那些回执——空闲、未阻塞、无后台工作、非取消的读取结果——
    而 `markWorkspaceRead` 为该工作区中每条未读完成写入回执；两者都覆盖仅有磁盘的会话，并发布
    每条变更记录。共享收集器*先*等待磁盘发现，然后在一个同步遍历中汇集活与磁盘来源（同 id 时
    活胜出），因此扫描中途挂接的聊天只被计一次，而不会在两个列表之间滑掉——`listSessionStates`
    同样如此；`nudgeSession` 原子地跳过需要输入、在运行时排队、在空闲时 prompt。
    `SessionSummary.state` 与 `session.state` 使用同一推导，因此客户端按 id 证明确切的已渲染完成，
    而不是自己发明生命周期。

    新会话与会话前入口点捕获当前世代；对活会话的操作使用该会话保留的运行时。`abort` 仍作为取消
    控制路径可用。`prompt`/`steer`/`followUp`（带图像）/
    — **`promptSession` 与 `followUpSession` 都依据会话**活**的 `isStreaming` 解析投递模式，
    而不是调用方对它的看法**：`prompt()` 在轮次中途抛错（因此回退到 `steer`），而 pi 的
    `followUp()` 只会*入队*到一条队列，由已在飞行中的 run 排空（所以在空闲会话上它回退到
    `prompt`，否则消息永远停在那里——就像一次 `review.sendBatch` 进入重新挂接的审查聊天时，
    把其评论标记为已发送给一个从未看到它们的 agent）/ **`clearQueueSession`**（Pi 的
    `clearQueue()`：排空两个队列但只返回文本，而宿主先快照其协调后的临时镜像，返回完整的逐消息
    文本 + 图像。Pi 发出清空的 `queue_update`）。其可选的仅文本前置条件在触碰 Pi 之前就拒绝——
    只要任一被跟踪的 lane 中有排队的图像；手动压缩使用该守卫，而 **`abortSession(..., true)`**
    在一次管理器操作中同步认领停止并排空队列。若某个被接受的问题结果仍在持久化，停止会在向 Pi 的
    abort 发信号之前给它一个有界的宽限期；这防止异步的 post-tool 钩子死锁停止，同时仍保留正常的
    快速路径。它在 abort 之前再排空一次，把这些迟到的到达追加到其原始 lane；随后等待空闲并返回
    完整有序的队列，因此延迟的停止不会与续跑竞争，也不会丢图像 /
    **`removeQueuedSession(sessionId, kind, index)`** —— 逐条队列移除，Pi 的 API 没有此能力
    （队列是裸字符串数组，`clearQueue` 是全有或全无）：通过完整内容路径排空，丢弃 `lane[index]`
    （越界 → `removed: null`，全部重新入队），并按顺序把每个保留项连同其图像重新入队（每条 lane
    用 `steer()`/`followUp()`——每次重新入队都发出自己的 `queue_update`，因此客户端仅凭事件即可
    收敛）。**无丢失保证：** 若该操作期间 run 已结算，重新入队的保留项会永远停在那里（Pi 的队列
    只在一次 run 内部排空），所以空闲情形通过与 `followUpSession` 相同的空闲投递回退来排空它们——
    第一个变成 `prompt`，其余 steer 进它启动的 run；投递时机可能在该竞争窗口内退化，但内容永不
    丢失（由空闲回退单元测试固定）——
    `setModel` / `setThinkingLevel` / **按会话门控的手动 `compact`**（第二个重叠请求在 Pi 能覆写
    它唯一的压缩控制器之前被拒绝；正在进行的 Pi 压缩也会阻止进入）/ `getSessionStats`
    （+ contextUsage）/ `getSessionCommands` / `listAvailableModels` / `listSettledModels`
    （同一快照但不启动刷新）/ **`clampThinkingForModel`**（pi 针对 `{model, level}` 对的
    `clampThinkingLevel`——`model.clampThinking`；宿主用它，使默认值与活会话的 effort 变更跟随
    Pi）。更早时，#394 显示 ThinkRail 的 `available[0]` 与 Pi 的选择不同并撞上代理 400。现在
    ThinkRail 在宿主侧从 `AppConfig` 解析新聊天默认值，当保存的默认值不可用时回退到
    `preferredDefaultModel`（pi 的 provider 默认值，然后第一个可用模型），而不重写该设置。仅
    `available[0]` 是目录顺序——对一个全新的 OpenAI 连接来说是 `openai/gpt-4`，一个 8K 上下文
    模型。显式请求的不可用模型会被拒绝，而不是静默替换。宿主显式传入选定的模型，使 UI 与会话一致。
    计划审查对未设置的审查者覆盖使用同一个 `resolveNewChatModel({})`：AppConfig 的
    `defaultModel`/`defaultEffort`，以首选模型和 `medium` effort 作为回退。

    **模型以 `WireModel` 跨过线上契约（绝不用 pi 原始的 `Model`）：** `toWireModel` 把 `Model`
    投影到线上契约的**允许列表**（见 `WireModel`）——因此 `baseUrl`、`headers`、扩展/provider
    路由数据以及任何其他字段默认被排除——而入站侧通过 `resolveWireModel` 针对
    **`settledAvailableModels`** 按 `{provider,id}` 重新解析该引用：`createSession` 使用当前世代，
    而 `setModel` 使用该活会话保留的运行时。因此一个刚出现在全局选择器中的模型可能对较旧的活聊天
    不可用，并以封闭的模型不可用错误失败，而不是跨越世代。Pi 逐字使用 `Model.baseUrl`，所以客户端
    的 baseUrl 从不被信任（既阻断泄露，*也*阻断任意 URL 注入）。**水合读取侧**——
    `listSessions(workspaceId, cwd)`（活会话**与** pi 持久化在 `cwd` 下的磁盘会话取并集，同 id 时
    活胜出 → 打上 `live` 标记的 `SessionSummary[]`；在把**分离的**磁盘列表当作权威之前，它会严格
    扫描每一份 transcript 头部并核实 pi 返回了每个文件，因此不可读/格式错误/被跳过的 transcript
    会让该读取被拒绝，而不是伪装成不存在并被重连协调标记为墓碑。首个解析出的条目不是会话头部的
    `.jsonl` 不是 transcript——pi 自己的列表会跳过这类文件，而扩展会在那里保留旁路日志（例如
    pi-automode 的 `<session>-pi-automode.jsonl`）——但一个完全解析不出任何条目的文件仍然会让
    读取被拒绝。已注册活会话自身确切的 `SessionManager.getSessionFile()` 路径被排除在该磁盘预检
    之外：它的内存条目已经是权威的，而 pi 可能在宿主列表期间截断/重写该路径，因此把这种瞬时物理
    状态当作分离的损坏聊天会让每次重载都清空所有聊天）+
    `getSessionMessages(sessionId, workspaceId, cwd)`（若尚未存活，则把磁盘会话重新打开进管理器，
    首先在活跃进程运行时中解析 transcript 所指的任何模型，并在其当前 id 下重试 pi 已改名的
    provider；当该模型不可用时，PI 选择当前默认值，且打开时不持久化模型变更；PI 在用户发送消息时
    记录该替换。没有持久化模型引用的旧 transcript 也使用 PI 的模型解析器——然后返回
    `{ summary, messages }` —— `TranscriptMessage[]`：pi 规范的子集**加上 `custom` 消息**（它们
    携带 `tool-response` 回复——以及旧 transcript 中已退役的 `ask-user-answers` 形式——交互式
    工具卡片按工具调用 id 与之配对），**加上 `compactionSummary`**，pi 为被压缩掉的消息保留的
    持久标记——之所以保留它，正是因为 pi 解析后的 transcript 是唯一幸存的东西，丢掉它就会交给
    客户端一个从对话中途开始的聊天，而没有任何东西解释这个空缺。哪些角色属于这些是**不在此处
    决定的**：该过滤器是契约的 `isTranscriptMessageRole`，与 `history` 的索引共享，因此两者不会
    漂移并错位 `messageIndex`），外加 **`ensureSessionAttached(sessionId, workspaceId, cwd)`** ——
    同样的单飞重新打开但不读取 transcript，供只需要会话*可再次 prompt* 的调用方使用（审查发送
    对既有聊天的后续投递）。它**仅当该 id 在该 cwd 中不指向任何 transcript 时**回答 `false`——
    这是调用方唯一可以改为启动新聊天的情形——而对其他所有重开失败都**抛错**，因此仅仅不可读的
    会话绝不会被误认为不存在而被静默分叉；磁盘这一半正是宿主**重启**后幸存的部分——而重新挂接
    会在 **`createAgentSession` 播种其上下文之前运行 `repairDanglingToolCalls`（`sessionRepair`
    兄弟模块）**：工具执行中途宿主死亡会留下最后一个可重放的助手工具批次未配对，每个 provider 都
    拒绝这样的叶子（聊天会变砖），而在活会话之后追加会让其内存状态失同步——所以缺失的结果在
    每个重启后会话都必经的唯一瓶颈点被配对。修复是**仅尾部且重放感知的**：pi 会按位置关闭一个
    待处理工具批次，然后才检查下一条助手消息，随后丢弃 `error` / `aborted` / `length` 尝试。Pi
    从不执行来自长度截断响应的工具调用，因为其参数可能不完整；这些调用因此用普通错误结果修复，
    绝不是一次可应答的 ask ack。失败的尝试从不贡献候选，但仍会关闭一个更早的候选；任何后续的
    用户、custom、压缩或助手消息都会使该空缺不再适合追加持久化叶子。只有调用 id **和工具名**都
    匹配最终候选批次的唯一结果才可紧随其后，而已经带有有效结果的并行调用保持不动。这防止一个迟到
    的结果在没有其被省略的调用时存活，并永久毒化 OpenAI 重放。一般的缺失调用得到 pi 的 abort 约定
    （`isError` "Operation aborted (host restarted…)"）；悬空的 ask 得到规范 ack
    （`details {kind:"ack"}`），因此其卡片仍可回答，同时 transcript 对 provider 合法；
    **`respondToTool(sessionId, toolCallId, response)`** —— 交互式工具的回复路径（见
    `toolResponses` 条目）；**`settleSessionsForShutdown(timeoutMs)`** —— 关闭的礼貌一半：同步关闭
    每个命令服务与 subagent 完成拥有者，并在 abort 任何流式父会话之前启动每个隐藏子会话的级联。
    保留具有可恢复活 ask 阶段（`expected`、`waiting` 或 `answer-accepted-uncommitted`）的父会话；
    处置每个隐藏子会话（包括其父会话已空闲的），纳入因并发移除而已经待处理的级联，并在同一边界
    之内等待全部完成。关闭在快照阶段之前原子地关闭答案准入：一个 expected/waiting 的 ask 仍保持
    悬空以便 ack 修复，而一个已被接受的答案会到达其原生持久化结果，随后续跑被中止。该快照之后
    竞态而来的回复会被拒绝，而不是被接受并在处置期间丢失。破坏性拆除在 abort/处置之前排空两条
    Pi 输入队列，包括可恢复的聊天删除、工作区归档、礼貌关闭与紧急处置。Pi 的 `abort()` 只向当前
    核心 run 发信号并等待会话空闲；run 后处理可能用一个全新的 abort 控制器继续处理已排队的输入，
    因此仅靠 abort 就可能重启工作并搁浅处置。移除/归档/删除复用 Stop 的基于条目的排空与有界
    已接受答案宽限期，在 abort 之前再次排空以丢弃该宽限期内排队的输入。公共命令保留其删除守卫；
    基于捕获条目的拆除绕过它，因此归档仍能结算已成墓碑的父会话。礼貌关闭则保留未应答的 ask 以供
    重启修复，在已接受结果的持久化边界再次排空，并保留其既有的共享关闭预算。同步处置（包括失败的
    准备）在断开 Pi 之前清空队列，且发生在宿主取消订阅之后，因此处置不会多发布一个队列事件。
    显式用户停止在向 Pi 发信号之前同步认领一个未应答的 ask；当 Submit 已经赢下时，它会给那个确切的
    结果边界一个有界宽限期，然后即使持久化仍待完成也发出 abort 信号。答案 RPC 仅当 `turn_end`
    包含被接受的原生结果时才解析；被替换或缺失的结果会拒绝它。每条处置路径都在取消订阅之前放弃
    注册表，因此已接受答案的 promise 不会搁浅。`abandon()` 是进程处置管道，而不是语义取消：它只在
    同步会话处置之前立即解除内存工具的阻塞，而真实 `SessionManager` 的重启覆盖固定了它返回的结果
    不会在挂接时修复之前被持久化。`disposeAllSessions` 仍是同步的紧急停止，但把它尽力而为的子会话
    级联注册进同一个待处理集合；`getSessionWorkspaceId(sessionId)`（宿主的 `set_title` 处理器用作
    键的活会话→工作区查找）；`removeSession`/`disposeAllSessions`；
    **`removeWorkspaceSessions(workspaceId, cwd?)`**（**归档拆除**：在第一次 await 之前关闭该工作区
    的会话准入，捕获每个已注册的父会话，同步关闭其资源拥有者，并在并发排空已退役工作区世代的准备
    工作的同时启动父会话 abort/移除。之后完成的准备会处置其未注册的拥有者/会话，而不是发布进已
    归档的工作区。在破坏性移除期间，未应答的问题不会被保留以供重启。两个屏障都结算之后，
    **无条件地**处置该工作区的每个活会话——绕过 `removeSession` 强制的逐聊天删除守卫，因此一个
    可恢复删除正在回收站中途的聊天无法中止拆除循环并搁浅其兄弟——然后删除根植于工作树 `cwd` 的
    pi 磁盘 transcript——pi 的 `SessionManager` 是仅追加的，所以清除 = `list(cwd)` 然后 `rm` 那些
    记录的 `cwd` 匹配的文件，绝不 `rm -rf` 编码后的目录，因为 pi 的 cwd→目录编码可能让不同的
    cwd 撞在同一目录上；在双重归档时省略 `cwd` 只跳过磁盘清除）；
    **`deleteSession(sessionId, workspaceId, cwd)`**（在任何 await 之前先标记为已删除，使在飞行中的
    磁盘挂接无法之后再注册；该墓碑还使一个保留的活条目在**整个删除事务期间对所有会话命令（包括
    `session.dispose`）都不可寻址**，因此另一个客户端无法在待处理的回收站移动之后追加一个轮次，
    也无法销毁回滚目标。**该事务按会话 id 单飞**：针对同一聊天的并发第二次回收站点击（另一个
    标签页/客户端）会加入正在运行的事务（或在外部工作区指明该 id 时作为未知被拒绝），而不是启动
    一个对手事务——共享墓碑的两个拥有者会让失败方在移动中途把它回滚并短暂重开聊天——并且**只有
    安装墓碑的那个事务才会在失败时清除它**，因此较早一次成功删除的永久墓碑能在后来一次虚假的
    重复删除中幸存。如需要则中止一个活轮次但保留活条目，从该会话自己的 `SessionManager` 解析活
    transcript（绝不用有损的目录列表），否则使用上面同样的严格磁盘查找，把 cwd 完全匹配的
    transcript 通过 `trashFile` 移到操作系统回收站，然后处置活条目并为客户端收敛发布
    `SessionDeletedPayload`；一个新建的空活聊天，其预留 JSONL 路径尚未实体化，则没有可恢复内容可
    回收，直接处置。任何查找或回收失败都会抛错、回滚它安装的墓碑、恢复对同一 transcript/活条目的
    命令访问，并且不发布任何东西；在一个可恢复的 UI 动作背后，有意没有永久 unlink 的兜底）；
    `setSessionPublisher` + `setSessionCreatedPublisher`（在 `createSession` 注册一个新的宿主自有
    会话之后广播初始 `SessionSummary`——不是在既有 transcript 重新挂接时——以便对等前端无需继承
    摆放位置就能发现它）+ `setSessionDeletedPublisher` + `setSessionManagerFactory` 接缝。
  - `oneshot` —— 一次性 LLM 补全，**不使用** `AgentSession`（无工具/扩展/磁盘）：
    `completeOnce(request)` 从共享运行时的已认证集合中挑选一个模型，并分发一次
    `runtime.completeSimple()`——pi 规范的、与 provider 无关的请求路径，它自行解析模型的认证
    （含 OAuth 刷新），也能服务只实现 `streamSimple` 的 provider（扩展注册的那些）。
    `pickModel(tier)` = 模型选择：`cheap` 优先取一个精选的小/快允许列表 ∩ 已认证集合，否则取按
    每 token 成本计算最便宜的定价模型（未定价的目录条目属于免费、预览或隐秘档位，会限流或消失），
    再否则取 `default` 选择；`default` = `preferredDefaultModel`；当没有任何已认证项时为 `null`。
    这是 `assist` 任务（计划摘要、PR 起草）运行的基元——会话之外唯一发生模型**分发**的地方。
  - `webUiContext` —— `createWebUiContext(sessionId, currentToolCall?)` 构建 pi 调用的
    `ExtensionUIContext`（对话框往返到浏览器，fire-and-forget 方法推送）。对话框指明它所涉及的
    工具调用（`toolCallId`），以便 web 在那次调用的卡片上询问：管理器传入 pi 自己的
    `agent.state.pendingToolCalls`，且仅在**恰好一个**调用正在运行时对话框才锚定（工具前的常见
    门控情形；并行调用保持不锚定，而不是猜测），而 `selectForToolCall` 显式锚定并携带该调用的
    `toolInput`（MCP 守卫）。`setExtUiPublisher`（server→client 推送接缝；一个可选的
    `ExtUiAudience` 把推送收窄到低于某协议版本的客户端）、`resolveExtUi`（浏览器回复）、
    `cancelExtUiForSession`（在处置时，以及 pi 接受一条用户消息之后——pi `"queued"` 的 steer 或
    follow-up，或一次空闲 prompt 其 `preflightResult` 为 `"started"`/`"queued"`——随后它回答该
    聊天的每个待处理对话框；被输入扩展 `"handled"` 的消息、或被预检拒绝的消息（无模型、无凭据）
    会让对话框保持打开，而队列在 `removeQueuedSession` 中的重新入队不取消任何东西）、
    `notifyExtUi`、`notifyExtensionError`（pi 的 `ExtensionError` → 一条客户端可见的 `error`
    通知，携带扩展 + 事件 + 原因——原因截断在 500 字符，因为 `error.error` 是远程形状的，而当
    扩展的文件是匿名入口点（`SKILL.md`、`index.ts`）时用其**目录**命名该扩展，绝不是
    "Extension SKILL.md failed"；一句干巴巴的 "An extension failed." 正是让 #277 仅从 UI 无法
    读懂的原因）。管理器的 `bindExtensions({onError})` 把它包在 `reportExtensionError` 中，后者
    做了通知做不到的**两件**事：对活条目，它向轮转的宿主日志写一条 `warn`，携带**完整**的
    `extensionPath` 和扩展自己的 `stack`（重新水合到一个 `Error` 上，以便落入结构化的 `err`
    字段——聊天拿到短名称，日志拿到无歧义的名字，崩溃在标签页关闭之后仍可查找），并且它把客户端
    推送**门控**在 `entry.registered` 上，即 `registerSession` 把条目放进映射时设置的显式标志。
    事件路径的 `sessions.get(id) === entry` 不能复用：`bindExtensions` 在 `prepareSessionEntry`
    内部运行，*早于*注册，所以更严格的形式会抑制 #277 所谈的 `session_start` 失败。映射中的*缺席*
    也不能代表“尚未注册”——`disposeSession` 删除时不留下墓碑，因此被处置的条目与未注册的条目无法
    区分，而一条迟到的错误会被推给一个永远无法处理它的客户端。对活条目，日志从不受门控，但宿主
    已**处置**的条目（`entry.disposed`，在每条拆除路径中都在 `session.dispose()` 之前设置）会把
    报告降级为单条 `debug` 行，不带 stack、不推客户端：pi 0.87 的 `finishTurn` agent 循环钩子活得
    比 `AgentSession.dispose()` 更久，仍会把 `turn_end`/`context` 边界分发给我们刚刚失效的 runner，
    因此它的 “stale ctx” 和 “could not resolve the persisted assistant entry ID” 报告是我们自己
    拆除的回声，而不是扩展崩溃；pi 0.86 先从 agent 断开，所以它们从未浮现。对活条目，它**仅当 pi
    提供了 stack 时**才附加 `Error`：pi 自己的若干 `emitError` 调用点省略了它（`runner.js` 的
    message_end、`agent-session.js` 的 command/`<runtime>`），在那里合成一个就会记录*宿主的*
    stack——把读者指向 `prepareSessionEntry` 而不是扩展，恰恰与那行日志存在的理由相反。
    **成员分三类，而不是两类。** *不可翻译*的是惰性空操作，也确实应当如此——它们接受一个 web 宿主
    无法渲染的 TUI `Component` 工厂（`setFooter`、`setHeader`、`setEditorComponent`、`custom`、
    `setWidget` 的工厂重载；字符串数组重载**是**被渲染的）。*可翻译*的必须真正生效：**`theme` 是
    一个真实的 `Theme`**（`plainTextTheme`），其每个装饰器都原样返回输入。*可翻译但未实现*是第三类，
    此处点名是为了让这个划分不被读成穷尽：`setEditorText` / `pasteToEditor` 由 pi 自己的 rpc 模式
    转发给宿主，web 输入框本可以遵循它们；在有人需要之前我们的实现保持惰性。把主题与那一类区分开的
    是惰性的代价——未实现的编辑器调用只丢一个功能，未实现的主题会在扩展的第一行就杀掉整个扩展。
    `getAllThemes: []` / `getTheme: undefined` 与 pi 自己的 rpc 模式一致；`setTheme` 的
    `{success:true}` 是已知的谎言，单独跟踪——web 宿主没有可切换的 TUI 主题，因此 pi 的 rpc 模式
    形式（`{success:false}`）才是诚实的答案。`plainTextTheme` 继承 pi 的 `Theme` 并覆盖
    `fg`/`bg`/`style`/`bold`/`italic`/`underline`/`inverse`/`strikethrough`/`getFgAnsi`/
    `getBgAnsi`；`getThinkingBorderColor` / `getBashModeBorderColor` 之所以保持朴素，只是因为 pi
    通过 `this.fg` 路由它们——这是一项继承来的保证，所以 `webUiContext.test.ts` 显式固定了它们。
    **还有三个继承成员仍在替终端作答，而且是作为数据而不是转义序列：** `getColorMode`
    （`truecolor`，来自构造函数——pi 的 `ColorMode` 是 `"truecolor" | "256color"`，没有“不渲染
    颜色”的取值），以及 `colors` / `appearance` getter，它们把每个 `""` 令牌解析为 pi 猜测的终端
    默认值（例如 `colors.accent` 是一个 RGB 值），并报告 pi 检测到的浅色/深色外观。对它们中的任何
    一个都不存在诚实的朴素答案。当扩展*通过*主题着色时它们零成本——每条这样的路径都返回纯文本——
    只会咬到那种自己读取模式或颜色然后自行发出 ANSI 的扩展，那正是本模块之外跟踪的未消毒桥接缺口。
    它的颜色表**仅**为满足构造函数签名而存在：每个会把颜色变成转义序列的方法都被覆盖，而那些仍替
    终端作答的成员读取的是构造函数的*模式*或 pi 的终端猜测，绝不是转义序列。一次改动画板的 pi
    升级会让构建失败，这是*主题接口移动了的通知*，而不是缺陷。
    **被否决的方案**（这些决策被记录的唯一位置）：(1) `{} as ExtensionUIContext["theme"]` —— #277
    缺陷本身。它假定 TUI 成员在 `rpc` 模式下不可达，但 `ctx.ui.theme` 是由**扩展**调用的，不是由
    pi 的渲染器调用的，而 pi 自己的 rpc 模式会交出一个活的主题。(2) 一个结构化实现 `Theme` 的对象
    字面量——不加强制类型转换就不可能：`Theme` 带有私有字段。(3) 用空白颜色构建一个真实 `Theme`、
    不做覆盖——pi 会把 `""` 映射为*默认颜色*的转义序列（`\x1b[39m`），而不是什么也不输出，因此
    状态文本会以字面转义字节到达浏览器。(4) 转发 pi 导出的 `theme` 单例——它是一个 `Proxy`，在
    `initTheme()` 运行之前会抛 `Theme not initialized`，而 `initTheme` 只从 pi 自己的 CLI 入口点
    调用，在通过 `createAgentSession` 嵌入 pi 时从不调用。每个把 pi 当库嵌入的嵌入方都会撞上这一点；
    上游修复要等到一次刻意的 pi 升级才会到达我们。
  - `toolResponses` —— **交互式工具**（其 `execute` 等待真人的工具）的宿主一半：按会话的回复信箱
    （`ToolResponseWaiters`）、可回复性、重启修复支持与迟到回复。信箱是**工具无关的**：每个条目以
    `toolCallId` 为键，回复是一个不透明值，宿主所掌握的唯一工具知识是已启用注册扩展的
    `interactiveTools` 映射（`extensions/interactive`）——哪些名称需要等待真人，以及一条迟到的
    回复对模型而言如何呈现。`ask_user_question` 是第一个客户
    （[[module-pi-ask-user-question]]，由 [[module-ext-ask-user-question]] 组合）：
    - 该包拥有模式、校验、`message_end` 归一化器（它让第一个 ask 成为回复唯一的工具调用）、
      顺序执行的 `execute` 和结果信封。
    - 它的注册表条目通过 `sessionExtensions` 注册该工具，并在 `interactiveTools` 中声明它。
    - 新建与重新打开的父会话在 `createParentSession` 中创建一个信箱，而
      `toolResponseSessionServices(waiters)` 把它作为 `expectToolResponse(toolCallId)` 与
      `awaitToolResponse(toolCallId, signal)` 交给该接缝。一次调用只有在扩展宣告它时才进入信箱，
      绝不按工具名进入：在扩展关闭时，一个 `ask_user_question` 调用（模型调用缺席的工具，或同名
      包的工具）不会产生需要输入、不持有完成投递，其回复 RPC 以“未在等待”失败。信箱活得比资源
      重载和扩展开关更久：在重新启用之后提出的问题通过同一信箱作答，而在扩展关闭时创建的聊天会
      通过该开关所请求的那一次重载接纳它，且先于任何同名包。
    - 回复 → `response`，取代 → `deferred`（工具返回待处理标记并结束轮次；pi 仍会排空排队的
      输入框消息），处置 → `abandoned`。停止类拒绝作为宿主的已停止错误透传。
    - `agent` 不从扩展导入任何东西：待处理标记及其守卫属于 `contracts`，待处理与已停止文本是宿主
      自己的，而迟到回复的措辞来自描述符的 `describeResponse`（否则用一段通用前言加上 JSON 形式的
      回复）。
    - 来自用户包的同名工具通过加载器的 `preferRegistryTools` 顺序让位给我们
      （[[submodule-server-extensions]]，在 [[pi-compatibility]] 中注册）。这保持了结构化卡片与其
      答案通道的配对：一个已安装的终端问卷绝不能在 web 客户端渲染我们卡片的同时用 `ui.input`
      取代执行。它的其他工具、命令和钩子仍然可用。

    一个合格的活调用是**顺序且阻塞的**：校验之后其 `execute` 等待 `session.respondToTool`，然后
    通过工具自己的信封把真人的真实回复作为原生工具结果返回。资格是共享的：被 `error`、`aborted`
    或 `length` 停止的助手无法执行交互式调用。依赖一个未决答案的问题会等待后面的轮次；多轮访谈
    这一规范本身位于工作流族的 `asking-user-questions` 概念 skill（[[submodule-workflow-skills]]）。

    注册表通过 `turn_end` 跟踪 `expected`（合格调用已宣告）、`waiting`、
    `response-accepted-uncommitted` 与 `stopped`。这覆盖了 Pi 从 `tool_execution_start` 经过
    pre-tool 钩子到 `execute` 的真实异步间隙：礼貌关闭冻结新的回复准入并保留一个 expected 调用；
    在该冻结之前被接受的回复仍然是权威的，并在关闭/停止中止其续跑之前被持久化。停止优先会同步把
    一个 expected/waiting 调用标记为已停止，因此后来的回复无法逆转胜者。语义校验仍然门控执行：
    在 pre-execute 窗口内被接受的回复，只有在真实工具返回且 `turn_end` 包含其结果时才被确认；
    一个校验错误（工具的询问方抛错）或缺失的结果会拒绝该回复 RPC，而不是挂起或声称成功。该证明
    不需要来自 `execute` 的回执——后者活在包里。`persistTurn(toolResults)` 按 `toolCallId` 把
    结果与等待者匹配，并只接受一个 `details` 与被接受回复深相等的非错误结果：`details` 就是回复
    本身，所以该证明从不依赖工具给模型的措辞。一个只重写内容的 post-tool 钩子会被确认，因为回复
    确实持久化了；一个替换 `details` 或把结果变成错误的钩子会拒绝该 RPC。已否决：同时比较重建的
    信封（那会把宿主绑到工具文本上，并在另一个扩展重写工具输出时把已持久化的答案报告为丢失）。
    Pi 未执行的每个 expected 调用在 `turn_end` 时被清除。回复 RPC 在匹配结果到达持久化的
    `turn_end` 边界之后才解析该阶段并确认。显式停止排空队列并以稳定的已停止错误中止一个未应答的
    阶段；Submit 赢下之后，停止把 Pi 的 abort 推迟到回复持久化，然后只结束续跑。两种顺序都会留下
    一个终止的、provider 合法的结果。

    **用打字代替回答会取代活卡片，与重启之后完全一样。** `steerSession` 与流式期间调用的
    `promptSession` 先把文本排进 Pi 的 steering lane，只有在 `steer()` 报告 `queued` 处置时
    （返回 `handled` 的扩展 `input` 处理器不会排队任何东西，因此 ack 之后不会有用户轮次）才对每个
    expected/waiting 调用调用 `supersede()`：工具返回与重启修复所写的相同的待处理标记
    （`details {kind:"ack"}`），并带 `terminate: true`，使 Pi 跳过工具批次自己的模型请求。Pi 把
    steering 消息作为下一个用户轮次排空，而卡片通过它已经理解的唯一形状（待处理标记 + 更晚的
    用户消息）从 transcript 推导出 `superseded`。之后的回复 RPC 以 `superseded` 可回复性错误失败；
    已经处于 `response-accepted-uncommitted` 的调用绝不会被取代（回复胜出，steering 在它之后投递）。
    范围有意很窄：只有用户自己的输入框发送才会取代——follow-up lane（Cmd+Enter）、在卡片出现*之前*
    排队的 steering、`removeQueuedSession` 内部的重新入队，以及 nudge（它们本来就在 `needsInput`
    处停下）都让卡片保持打开。

    进程重启有意只改变续跑机制：挂接时的修复会在 `createAgentSession` 之前，为一个合格的悬空调用
    写入待处理标记（`details {kind:"ack"}`，宿主的 `TOOL_RESPONSE_PENDING_TEXT`），前提是其工具名
    在已启用的 `interactiveTools` 映射中；可回复性同样把任何其他工具的报告为未知，让卡片可回答但
    会话空闲。不做参数形状检查：我们的实现在启用时赢下该名称，而被禁用扩展的工具根本不是交互式的，
    因此外来的同名调用得到普通的错误修复。被长度截断的调用得到与任何其他不可执行工具调用相同的
    错误修复。没有活阶段时，`respondToTool` 通过 `sendCustomMessage({triggerTurn:true})` 注入一条
    `tool-response` custom 消息（内容来自 `describeResponse`，`details { toolCallId, response }`）。
    因此问题得以在既不恢复 JavaScript promise 也不调用底层 `Agent.continue` 的情况下存活；一个未
    提交的 Submit 只是再次出现。队列条目仍归 Pi 所有，并且按显式范围为仅活。卡片从 transcript 推导
    两种形式（原生活结果与待处理标记 + custom 回复）。

    已否决的方案：对每个活调用立即 ack 会让 Pi 排空在卡片*之前*排队的输入并不可见地取代它；精确
    恢复一个悬空调用需要 Pi 并未提供的一种生命周期安全的 resume API；当只有问题必须存活时，一个
    持久化的宿主队列/SQLite 发件箱并不必要；客户端侧的“跳过再 steer”组合会与答案路径竞争，并在
    transcript 中留下一个拒绝，而恢复路径留下的是一个 ack；把信箱以工具名为键（那个接缝只会在
    类型上是通用的——第二个交互式工具，或第三方的，可能宣告一个 web 永远无法回答的调用）。
  - `sessionRepair` —— `repairDanglingToolCalls(sessionManager, interactiveTools)`：重启安全网
    （理由见上文管理器条目）。对 pi 的 `SessionManager` 是纯函数（通过 `buildSessionContext` 具备
    压缩感知；幂等；只追加活动尾部批次中缺失的结果）——针对 `SessionManager.inMemory` 做过单元
    测试，包含失败尝试与历史空缺的重放。
  - `imageGuard` —— 超大图像守卫：一个内联扩展（`oversizedImageGuard`，`buildResourceLoader` 的
    共享工厂之一，也是被委派子会话基础集合的全部），挂在 pi 的 **`context` 事件**上（每次 LLM
    调用之前触发，活会话也不例外）。**仅 Anthropic 家族**：上限是 Anthropic 模型级规则，所以该
    处理器按上下文的活动模型门控（`isAnthropicFamilyModel`——原生 `anthropic`/`anthropic-messages`，
    或经 Bedrock/Vertex/聚合器路由的 Claude 模型 id；未知模型 ⇒ 空操作），其他每个 provider 的
    图像上下文原样通过。它直接从 base64 头部字节嗅探每个图像块的像素尺寸（PNG/JPEG/GIF/WebP——
    不用编解码器，绝不清除它嗅探不出的东西；**每轮工作有界**：只实体化一个 256KiB 的解码前缀——
    SOF 位于其之后的 JPEG 嗅探为未知，而不是被清除——且每个块每轮只嗅探一次），并把任何违反
    provider 规则的块替换为一条文本提示，指明所违反的规则外加重新附加的提示。五条规则，按顺序：
    **provider 接受的媒体类型**（`ACCEPTED_IMAGE_TYPES`，通过 `contracts` 与输入框共享——pi 会逐字
    转发图像的媒体类型，所以一个旧的 `image/heic` 块会让整个请求 400；清除它能治愈在输入框拒绝
    这类文件之前就被毒化的会话）；**4.5MB 编码 base64 载荷上限**（`IMAGE_MAX_BASE64_BYTES`，通过
    `contracts` 与输入框共享——pi 自己在 Anthropic 5MB API 上限之下的余量，与 `data.length` 比较，
    因为线上契约携带 base64，因此它甚至适用于无法嗅探的格式）；**每边 8000px 硬上限**；一旦整个
    上下文携带超过 20 张图像，就启用**计数感知的 2000px 上限**——清除会改变选中该上限的那个计数
    本身，所以违反 2000px 的块**只按从大到小的顺序清除，直到幸存者重新落回阈值之下**（18 张小图
    + 3 张 2500px ⇒ 清除一张，其余两张在 8000px 之下仍合法）；以及**请求范围的
    `REQUEST_IMAGE_BASE64_BUDGET`**（24MB base64，Anthropic 每请求 32MB 上限之下的余量——若干个
    单图合法的块仍可能撑爆整个请求），通过**从大到小清除幸存者直到总量落回预算**来执行。这正是
    把被超大图像毒化的会话解锁出来（历史每轮都会重发，所以一张坏图会永远 400）：会话是仅追加的，
    而宿主没有图像编解码器（见上面的 autoResize 权衡），所以守卫只变换**出站上下文**——会话文件
    与 transcript 保持不动，卡住的聊天在其下一条消息上即可恢复。一旦会话超过 21 张图像，计数感知
    的上限还会把一张原始 >2000px 的 `read` 工具图像降级为一条提示，而不是变砖。纯核心
    （`guardOversizedImages`、`imageDimensions`）用手工构造的头部字节做过单元测试。
  - `skillCarryover` —— 压缩后 skill 恢复器：一个内联扩展，同时存在于 `buildResourceLoader` 的
    共享工厂与 `childBaseExtensionFactories` 中，因此被委派的子会话也携带它。Pi 的压缩会把已加载
    skill 的正文总结掉，而系统 prompt 只保留其名称和描述，且 `WORKFLOW_RULE` 禁止重新路由已经路由
    的工作。通过 `read` 加载的 `SKILL.md`——常规路径——处境更糟：pi 交给摘要器的每份工具结果都被
    截到 2000 字符，因此后面的规则永远进不了摘要。此后 agent 会继续“遵循”一个它再也看不见的
    skill。参照 Claude Code 压缩后重新附加已调用 skill 的做法：一个 **`context` 处理器**在分支上
    存在压缩期间，在每次请求中紧跟在 `compactionSummary` 之后插入一条
    `thinkrail/skill-carryover` custom 消息（`display: false`）。它对 pi 加载过的每个 skill
    （`before_agent_start` 的 `systemPromptOptions.skills`）在最新切口之前存在一次加载的情形进行
    恢复——一次 `parseSkillBlock` 用户块，或一次成功解析到该 skill `filePath` 的 `read`（在 pi 的
    `@`/`~`/`file://` 处理和符号链接之后）——除非切口处或之后的加载已经展示了整个文件；读取其他
    `SKILL.md` 文件不算数。切口之前的任何加载，包括摘录，都是调用证据。在保留的尾部中，每次加载
    按它展示的行数计算，算法与 pi 的 `read` 相同（1 起始的 `offset`、`limit`，以及被截断时的
    `details.truncation.outputLines`；一个 `/skill:` 块展示全部）：只有当它们的并集覆盖当前文件的
    每一行时正文才存活，因此摘录绝不会掩盖一次被压缩掉的加载，而合起来展示整个文件的若干页也不会
    被重复计费。skill 按其最近一次加载排序。正文从被准入的 skill 文件重新读取（去除 frontmatter，
    像 pi 的 `/skill:` 块那样包裹），预算为每个 skill 正文 5K token、整条渲染消息（前言、包裹、
    截断提示）min(25K, 上下文窗口的 10%)，按最新优先，从末尾截断并附上指向该文件的指针。窗口必须
    是被分发模型的窗口：`ctx.model` 是所选，而虚拟选择的限额（默认 0）并不能说明它路由到的物理
    模型——pi 的路由容量检查在本处理器之前运行——因此一个虚拟选择，或任何窗口未知的模型，都得不到
    结转。文本按压缩条目记忆化，因此注入的前缀保持缓存稳定。按请求注入是唯一能到达**运行中阈值**
    压缩与**溢出重试**压缩之后第一次请求的位置：`session_compact` 时的 `sendMessage` 在 run 活跃
    期间被推迟到 `turn_end`，所以那些请求发出时并没有它。不向会话写入任何东西，因此结转绝不会作为
    用户文本到达下一个摘要器，pi 的“Already compacted”守卫仍然成立。暂时否决：用一个自定义的
    `session_before_compact` 摘要记录当前工作流步骤（它会取代 pi 的摘要器调用并自己拥有凭据）。
    `skillCarryover.test.ts` 驱动一个假 pi 会话走完手动、运行中阈值和溢出重试三种压缩。实时的
    `skill-after-compaction` 规格固定了一点：压缩之后的阶段保留每条规则且不重新读取 `SKILL.md`：
    没有结转时 agent 靠每次压缩后重新读取来恢复（pi 的摘要也叫它这么做），所以仅凭合规性无法区分
    两者。
  - `delegation` —— ThinkRail 对可移植 **`pi-delegation`** 核心 + **`pi-subagents`** 层
    （[[module-pi-delegation]]、[[module-pi-subagents]]）的嵌入：绑定只有宿主才知道的东西——数据
    目录下的 delegation 根（`<dataDir>/delegation`）、`scope = workspaceId`，以及管理器的
    `liveParentContext` 投影（`ParentContext`，核心决策 #23），包括该父会话所保留的确切
    `ModelRuntime`。因此既有的父会话及其子会话在一次 Central 变更之后仍留在其运行时世代上，而之后
    创建的父会话投影新世代。宿主范围的 `getPiRuntime` 解析器作为核心的动态回退传入，而不是在服务
    创建时捕获。每个工作区缓存一个 `DelegationService`（`delegationServiceFor`，同步——绑定时没有
    任何东西 await）。有父会话的 run 的花费以 pi `usage` 条目的形式追加到活父会话
    （`kind: "subagent"`，`note` = 子会话 id；只知总成本），每次发布
    `entry_appended { entryType: "usage" }`：
    每次 `run-progress`（在每次子会话响应或工具启动之后）追加该 run 用量增量超过每 run 水位线的
    部分，而 `run-terminal` 追加剩余部分，这样工具栏能跟随一个运行中的 subagent，且 pi 自己的会话
    总计对每种结局和每个消费者（Agent、`request_review`）都包含被委派的花费，与完成投递无关。
    水位线（`subagentUsage` 的 run 记录器）存于内存，从 run 的第一条记录活到其 `run-terminal`，
    并在 pi 把该增量纳入其会话时推进（即使持久化失败也是如此），因此一次写入绝不会被计两次；pi 未
    纳入的增量由下一次携带，但在 `run-terminal` 处被丢弃的那份不会带入子会话的下一次 run——协调会
    把它找回。增量记录任何正的 token 计数或高于 1e-9 的成本。进度事件跳过监听器的资源/状态发布
    （资源摘要携带状态，不携带用量）。进度只是刷新读数；崩溃恢复仍来自子 transcript 与协调。这是
    被委派花费唯一的记账通道：delegation 工具绝不能返回 `AgentToolResult.usage`，否则 pi 会再求和
    一次。写入是尽力而为的，就像 pi 的缓存预热器：追加失败会被记录日志，绝不使子 run 失败。
    `recordSessionUsage` 只报告 pi 是否把用量纳入其会话；持久化仍是不确定的。pi 在持久化之前把叶子
    移到新条目，所以一次失败的追加会把上一个叶子恢复回来（更晚的条目绝不能链到一个未持久化的 id，
    否则重新打开的聊天会丢掉它之前的历史），并把既有 transcript 截回追加前的大小（撕裂的半行会吞掉
    下一个条目）；pi 的初始文件创建不在这次修复范围内。关闭一个聊天会让其条目保持可写到其拆除级联
    结算为止，因此该级联所停止的 run 仍会被记录；拆除期间拒绝挂接，所以 transcript 保持单一写入者。
    删除墓碑会抑制该写入，因为 pi 的追加会重建已进回收站的文件。礼貌的宿主关闭在其预算内记录；
    紧急处置（`disposeAllSessions`）会连同墓碑一起吊销拆除写入者，因此迟到的记录绝不能重建已进
    回收站的文件，而被它丢弃的记录在该聊天下次挂接时被找回。子会话不能生成子会话
    （`RECURSION_GUARD_TOOLS`），所以增量绝不嵌套。**`subagentUsage`** 让 transcript 而不是事件
    成为权威：每次挂接时，以及一次删除在其墓碑抑制了终止写入之后被回滚时，它会异步统计父会话
    delegation 目录下每个没有排队或运行中 run 的新子文件的用量总量，并把差额追加到父会话针对该
    子会话的 `subagent` 条目上。这找回了崩溃或被杀掉而从未完成的 run，为父会话记账之前的老聊天
    补上历史，并拾取 run 之外的子会话花费。解析出的子会话总量按进程以文件路径、mtime 和大小缓存，
    因此子文件版本未变时只读一次（重叠扫描可能重复读它）；每次协调都把这些总量与父会话当前的记录
    比较，没有记录时间戳的捷径，因此重开之后丢失的记录即使在同一个子会话的后续记录幸存时也能被
    找回。差额在追加之前立即从活条目重算，因此重叠的协调无法重复计数；只有在没有任何 token 计数为
    负、且缺失 tokens 或 cost（高于 1e-9，以忽略浮点噪声）时才追加，绝不作为负修正追加。被 fork 的
    子会话（头部 `parentSession`）会被跳过，因为它们复制的历史携带父会话自己的花费。由宿主注入的
    `setSubagentConcurrencyResolver` 在创建时提供每个服务的按聊天上限，而在全局或工作区上限变更
    之后，`refreshSubagentConcurrency(workspaceId?)` 会重新解析并在线调整缓存服务的大小（核心的
    `setMaxConcurrentPerParent`）；`subagentsFor(workspaceId, isEnabled, canDeliverCompletion)`
    为每个父会话创建一个保留的可移植 `Subagents` 拥有者；其扩展在每次资源加载时被注入。由宿主注入
    的 `setSubagentsEnabledResolver` 把该工作区 id 映射到当前生效策略，而不产生 `agent` →
    settings/workspaces 的边。该谓词到达扩展的启动时守卫以及初始/重载激活。对实时策略变更，
    `refreshSubagentTools(workspaceId?)` 通过 pi 的活动工具 API 移除/添加 `Agent` +
    `get_subagent_result`：空闲会话同步更新，流式会话只保留一个待重评估并在 `agent_settled` 时
    应用，重复变更那时解析最新策略。会话注册在异步扩展绑定之后、创建被发布之前再解析一次，因此
    策略变更不会落入“绑定先于注册”的空隙。策略变更绝不替换被保留的拥有者，因此已在运行的分离子
    会话会完成并保留完成投递；被禁用的启动仍会被活谓词立即拒绝，即便流式父会话的工具集还没来得及
    刷新。计划审查的 `request_review` 工具遵循同样的活开关形状：它总是被注册，但由
    `setAgentReviewEnabledResolver`（宿主注入，全局——没有 `agent` → settings 的边）决定它是否留在
    会话的活动集中，而 `refreshAgentReviewTool(workspaceId?)` 把变更按空闲同步 / 流式推迟到
    `agent_settled` 应用，与 subagent 工具完全一致。因为 `setActiveToolsByName` 会从活动工具的
    指南重建系统 prompt，丢弃该工具也就丢弃了它的指导。只有工具被门控——`startPlanReview` 的按钮
    路径是另一个宿主接缝。见 `submodule-server-host-plan-review`。
    **注册表扩展**遵循同样的形状，只是以扩展名而不是工作区为键。由宿主注入的
    `setDisabledExtensionsResolver(() => ReadonlySet<string>)`（全局——没有 `agent` → settings 的
    边；解析器抛错意味着没有禁用任何东西）由 `buildResourceLoader` 在**每次加载器重载**时读取
    （与项目信任一起快照化），因此被门控的注册表工厂会跳过一个已禁用的扩展，且 `extensionsOverride`
    省略其 `<inline:…>` 条目——新父会话从不加载它——而当一个隐藏子会话的加载器运行时，也会被
    `childExtensionFactories()` 读取。`refreshExtensionTools()` 更新活会话：
    `sourceInfo.path` 属于某个已禁用注册表扩展的工具离开活动集（按条目记住，以便重新启用时精确恢复
    被扣留的工具，绝不恢复那些默认就不活动的），空闲同步 / 流式推迟到 `agent_settled`，最新策略
    胜出，而 `setActiveToolsByName` 会连同这些工具一起丢弃它们的指导。已禁用扩展的钩子在打开的
    会话中继续运行，直到其下一次资源重载——pi 无法卸载扩展，而每次禁用都重载会击穿 prompt 缓存。
    有两种情形确实会重载，经由 `requestSessionReload` 门控：一种是会话从未加载过的已启用扩展（在它
    关闭时创建的）且它并未加载失败，另一种是**会话已加载其 skills** 的已禁用扩展
    （`loadsDisabledExtensionSkills`）——skills 活在系统 prompt 中，只有重载才能重建它，而留下的
    一个 skill 会把 agent 引向被扣留的工具。无 skill 的扩展保持不重载的禁用。当一个扩展拥有静态或
    按会话的父工厂时，它被视为可加载。注册应用同样的协调，而一次完成的重载会重新应用它，且不请求
    再次重载——除非扩展策略**在其运行期间移动了**：一次运行中的重载在开关之前快照了策略，所以其
    结果可能陈旧（在禁用重载期间重新启用，否则会让扩展保持未加载状态，而 Settings 显示它是开的）。
    `refreshExtensionTools` 只在禁用集合实际变化时才递增策略世代；每次重载记录它启动时的世代，其
    完成时以“仅当世代移动才允许重载”的方式协调，并判定**新的**加载器。因此一次未变更的重发、或
    重载期间一次无 skill 的开关，都不会带来额外代价。跨重启时 Settings 是事实来源：Pi ≥ 1.3 会恢复
    一个重新打开的聊天所记录的工具配置，使被记录的移除保持不活动，所以挂接会先把 Pi 的“注册即激活”
    规则会激活的、已启用注册表扩展的每个工具重新激活（即声明式暴露，而不是 `defaultActive: false`），
    然后扣留已禁用的。内建、MCP、subagent 与审查工具保留 Pi 恢复的选择。因此被 Settings 之外任何
    东西关掉的注册表工具会在重启后回来，就像在 Pi `reload()` 时一样。
    **`set_title`** 工具（`titleTool.ts`）是聊天与工作区获得名称的方式：主 agent 用 `chat_title?` /
    `workspace_name?` / `branch?`（英文 kebab slug）调用它。它总是被注册且活动，并通过
    `promptSnippet` 列出；在命名之后把它关掉会在会话中途重建系统 prompt 并击穿 prompt 缓存。它的
    `promptGuidelines` 携带命名规则：每段对话一次，作为第一个带有具体任务的轮次的**首个动作**，
    即便答案只有一行（或在读取一个被链接的 PR/issue/ticket 之后立即执行）。名称使用用户的语言，
    而 PR/issue/ticket 使用 `<Verb> #<n> <title verbatim>`。仅有指南被证明太弱（实况 e2e，Claude
    Opus：3 个真实任务轮次中只有 1 次命名；当 prompt 要求一句话回答时 3 次中 0 次）。因此只要该
    聊天（pi 会话名）或其工作区仍未命名，一个 `before_agent_start` 钩子就添加一段特定状态的
    **`pending-naming`** 系统 prompt 区段（“this chat has no title yet … call set_title with
    chat_title before your other tool calls … otherwise ignore this note”）；有了它，同一个真实
    任务轮次 3 次全部命名。该区段与指南都点明仍缺失的字段（工作区未命名时的 `workspace_name` 与
    `branch`），因为只说“一次”会让调用只命名聊天。其措辞有意低压：“first action … even when the
    answer is one sentence” 的早期版本会让模型在不相关的纯工具轮次上添加叙述性前言。一旦两者都
    命名，该区段就消失，代价是每个聊天在其早期一次 prompt 缓存未命中。写入策略以及该状态的工作区
    一半不在这里：宿主通过 `setTitleToolHost({ apply, workspaceNeedsName })` 注入两者，因为命名
    组合了 `agent` + `workspaces`。
    级联：`removeSession`/`disposeAllSessions` 触发 `disposeSessionChildren`——`removeSession`
    返回该级联，**删除事务在 `publishDeleted`/解析之前等待它**（安全：级联自带吞错，因此一个失败的
    子会话 abort 绝不能让一个 transcript 已进回收站的删除失败），而工作区归档**按会话等待它**——
    外加**为该工作区注册的每个待处理级联**（`disposeSession` 在级联*开始*时就把条目从 `sessions`
    中移除，否则并发的归档会在某次删除或移除的子会话级联仍在运行时看不到可等待的父会话；每个
    `disposeSession` 级联都在一个按工作区的注册表中登记，由归档排空——PR #303 评审发现 + 同一次
    排查中发现的并发那一半，两者都通过一个受测试门控的子会话轮次固定，两个方向都确定：红，因为修复
    前的归档在其同步前缀中 `rm -rf`，而门禁可证明仍关闭；绿，因为归档的完成被 await 链在级联之后。
    有意是**级联，而不是删除事务**：归档必须不被卡在回收站中途的删除阻塞——回收站这一步延迟无界且
    从不触碰 store；这种独立性本身就是被固定的行为）——在 `removeWorkspaceDelegation`（丢弃服务 +
    删除 `delegation/<workspaceId>`）之前，因此 store 绝不会在一个活子会话之下被删除——隐藏子会话
    绝不比其工作区活得更久。
    `readChildTranscript` 从 store 按 `(workspaceId, parentSessionId, childSessionId)` 服务
    `subagent.getTranscript`——这些 id 是会变成路径段的线上契约字符串，因此它拒绝形似路径的值
    （分隔符、`..`；该处理器还像每个同级读取一样校验工作区）——并连同消息返回该 run 当前的注册表
    `status`，后者由原始条目经 pi 的规范投影（`buildSessionContext`——与活会话的 `session.messages`
    所走的同一条条目→消息路径）构建，并完全像 `getSessionMessages` 那样经契约共享的
    `isTranscriptMessageRole` 过滤（此处曾有一个私有的消息条目循环发生漂移：压缩是一个条目*类型*，
    而不是消息角色，因此一个被压缩的子会话 transcript 丢掉了它的 `compactionSummary` 标记——PR
    #303 评审发现，已被测试固定；重启/处置之后缺席；线上契约含义见 [[module-contracts]]）。Pi 惰性
    地写入第一个 transcript 文件。因此一个属于所请求父会话的已注册子会话在该文件存在之前返回空消息
    列表加上其权威状态；排队/运行中仍可轮询，包括在第一次 provider 回复之前。这不是从任务文本重建。
    没有该自有句柄的缺失文件会抛 `CodedError("SUBAGENT_TRANSCRIPT_NOT_FOUND")`——这是 web 对话框
    停止轮询的**永久**未命中，在线上契约上具名而不是从消息做模式匹配（[[module-contracts]] 拥有
    代码集合；这里使用 agent 模块的窄 `@thinkrail/shared/codedError` 边，与 Chat Resources 共享，
    并镜像 `git` 的用法）。选择加入扩展的子会话（其定义中 `extensions: true`）得到**精选子会话
    集合**（`extensions` 中的 `childExtensionFactories`，即 pi `InlineExtension`）：
    headless-search 策略 + `pi-spec-graph` + 每个注册表扩展的 `childExtensions`（内联命名，来自
    [[submodule-server-extensions]]；今天是 `web-access` 的 `pi-web-access`）——有意不是父会话的
    完整集合（理由与“已列出子会话”的例外：核心决策 #25）。**每个**子会话，无论是否选择加入，也会
    加载核心的基础集合（`childBaseExtensionFactories`，同一决策）：`oversizedImageGuard`，因为子
    会话的 `read` 在共享 settings 下返回原始图像（见 `buildSessionSettings` 条目），以及
    `skillCarryover`，因为子会话在那些 settings 下也会自动压缩。一个使用由扩展注册的 provider 模型
    的子会话还会加载该 provider 的**拥有者扩展**（核心决策 #32：这样的 provider 可能只服务其扩展
    观察到的会话）：`liveParentContext.providerExtensionPaths(provider)` 从一个进程级拥有者映射作答，
    该映射由 `buildResourceLoader` 的 `extensionsOverride` 从 pi 加载时注册队列填充（来源：
    [[pi-compatibility]]），并与父会话自己已加载的扩展求交。之所以是进程级，是因为一个扩展可能只在
    它在某个进程中的首次加载时注册其 provider；求交则防止子会话加载其父会话没有的扩展。内联工厂
    永远不是拥有者。当子会话的信任（同一个 `isProjectTrusted`）为假时，项目作用域的拥有者也被排除：
    一个在繁忙父会话上被推迟的撤销会把这些扩展留在其加载器中，而显式路径无论如何都会加载它们。
  - `extensions` —— Pi 资源接线。候选世代通过一个无头的 `DefaultResourceLoader` 加载经过审查的
    外部 Central 路径一次，以应用 provider 注册，而不检查它。随后
    `buildResourceLoader(cwd, settingsManager, getAdmission, excludedPaths, extraFactories?)` 解析
    Pi 常规的 settings/包 + `.pi` / `.agents` 扩展集，在**加载之前**移除那个确切的不透明身份，并
    显式加载剩余的路径：会话使用其保留世代已经拥有的 provider 对象，因此任意的 Central 工厂/错误/UI
    无法到达 `pi.extensionUi`。Central 身份始终从会话发现中排除——即便全局工件发生变化——因此一个
    会话无法改变其世代。那个显式路径集合**不在构造时冻结**：加载器的公开 `reload()` 被包装，使每次
    重载都先通过宿主自己的 `DefaultPackageManager` 重新解析该集合（当前 settings 与信任，`builtin:*`
    条目按名称处理以便 `-builtin:*` 被尊重），并原地修改 pi 所读的宿主自有数组，然后在加载之后重新
    标注已发现路径的来源——不使用 pi 的 `extensionsOverride`，因为它在工厂已执行之后才运行。所有
    其他用户扩展保留常规发现。随后加载器添加自动的**可移植跨 agent skill 别名**，然后加载三个随附
    的 pi 包——**`pi-spec-graph`**（`spec_*` 工具 + 其 `before_agent_start` 规则）、
    **`pi-thinkrail-workflow`**（工作流路由规则 + 工作流 skills）以及 **`pi-todos`**（`todo_*`
    工具 + 其 skill）——以及**来自服务器注册表的 ThinkRail 扩展**（[[submodule-server-extensions]]；
    `visualize`、`ask-user-question`，以及带 `pi-web-access` 工具和 `librarian` skill 的
    `web-access`）作为具名内联扩展。既有的个人别名是 Claude
    （`${CLAUDE_CONFIG_DIR:-~/.claude}/skills`）、Codex（`${CODEX_HOME:-~/.codex}/skills`）、
    Copilot（`~/.copilot/skills`）和 Gemini（`${GEMINI_CLI_HOME:-~}/.gemini/skills`），**外加每个
    已安装 Claude 插件的 `skills/` 目录**（从 `~/.claude/plugins/installed_plugins.json` 读取——
    解析出的 `installPath`，绝不做缓存扫描，因此陈旧版本和传递依赖里的 `node_modules/**/skills`
    都被排除）；项目根别名是 `.claude/skills`、`.github/skills` 和 `.gemini/skills`。纯函数
    **`isProjectSkillPath(relativePath)`** 是工作树监视器（通过 `host` 注入）使用的唯一服务端定义：
    它识别这些别名加上 Pi 原生的 `.pi/skills` 与 `.agents/skills`，因此有上限的文件系统批次携带
    真实的 skill 变更证据，而不让 `watch` 依赖 `agent`。固定的项目/个人别名根被注册为候选 skill
    路径，**无论它们是否已存在**，因此一次 `loader.reload()` 就能拾取分支切换 / pull / clone 在会话
    中途创建的那个（插件目录是构造时已安装的集合——之后添加的插件需要新会话）；分类仍然只统计实际
    存在的目录。仍然绝不扫描任意点目录、插件缓存、命令，也不做向下嵌套发现。Pi 仍是解析器：仅厂商
    独有的宏/钩子/模型/子 agent/元数据不被模拟。首名优先的优先级是 Pi 原生/已配置/共享 →
    ThinkRail 随附 → 个人别名 → 项目别名，因此仓库永远无法遮蔽你自己的或 ThinkRail 的 skills；
    来源元数据保留真实的 `project` / `user` 作用域。
    **准入门禁（`skillAdmission`）：** 已提交的**项目作用域**别名对一次 clone 而言是攻击者可控的，
    且会被注入系统 prompt，因此按 skill 它们会从**准入上下文**解析为 `load` / `untrusted` /
    `pending-ack` / `disabled`——项目的 `trusted` + `acknowledgedSkills`（授予信任只确认当下存在的
    东西，因此之后 pull/切换分支带来的 skill 在被确认之前是 `pending-ack`）+ `disabledSkills` /
    **`disabledGroups`** 基线（组键 = 插件名、来源层级
    `project`/`personal`/`bundled`/`pi`，或特殊的 `@plugins`——由 `skillGroup` 按 skill 指派，与
    `SkillCatalogEntry.group` 匹配）——并与工作区的按 skill `skillOverrides` 分层叠加（信任门禁在
    开关层之前检查，因此一个“开”的覆盖绝不可能解除一个不可信别名的门禁，而按 skill 的 `on` 胜过
    组级禁用）。`skillsGate` 在一个 `skillsOverride` 中完成过滤 + 重新标注；只有 `load` 的 skills
    到达系统 prompt / `/skill:` 列表。一个**已禁用注册表扩展**的 skill 根（`extensionSkillRoots`：
    开发态的源根，打包态时是暂存根下该扩展的 `extensions/<name>` 文件夹）被排除在
    `additionalSkillPaths` 本身之外——在每次重载时从快照的禁用集合原地重建——绝不从 pi 的结果中
    过滤：pi 在任何覆盖运行之前就按首名优先解析同名 skill，所以事后丢弃一个胜者也会丢掉它遮蔽的
    个人或项目 skill。`listSkillCommands`（其缓存键包含禁用集合）与 `listSkillCatalog` 以同样方式
    构建输入，因此一个被关掉的扩展的 skills 也从 Settings → Skills 和 `/skill:` 中消失。随附的宿主
    skills 因此暂存在 `host/` 下，与 `extensions/<name>/` 并排。宿主通过
    **`setSkillAdmissionResolver`** 接缝解析上下文（以 `workspaceId` 为键，失败即关闭）；
    `buildResourceLoader` 把该解析器作为 thunk 接收，而 `skillsGate` 在每次 `loader.reload()` 时
    重新解析**两者**：准入上下文（`getCtx`）**以及**活跃的兼容来源集合（全新发现），因此
    `session.reloadResources` 能拾取会话中途的信任授予、skill/组开关，**或一个新出现的别名目录**
    ——而一个迟到的项目别名仍会被分类 + 信任门控，绝不会作为未分类的加载溜过去。个人与随附资源
    从不受信任门控（只受启用/禁用层），而该门禁的范围限于兼容别名。pi 原生的项目资源（`.pi/`
    settings、扩展、skills、prompts 与 themes、`SYSTEM.md`、祖先 `.agents/skills`）改由 pi 级信任
    支配：pi 用 ThinkRail 从 `Project.piResourceTrust` 推导出的项目可信标志来门控它们，并在每次加载
    之前重新应用（MCP servers › Trust）。
    `listSkillCommands(cwd, admission)` 通过一个短命的仅 skills 的 `DefaultResourceLoader`（无
    模型/会话/transcript，无扩展工厂）复用同一批受门控输入，用于工作区之前的自动补全，并按
    `(cwd, admission)` 短暂缓存；**`listSkillCatalog(cwd, admission)`** 是 Skills 管理器的未过滤
    变体（每个已发现的 skill + 其 `group` + `decision`）——由工作区驱动（通过 `skills.state`）
    或由一个项目驱动（通过 `project.skills`，当前检出，无覆盖）——而
    **`listProjectAliasSkillNames`** 是通知中的现有别名计数。完整会话加载器支持**两种模式**：
    - **从源码运行（默认）：** `additionalExtensionPaths` 指向 pi 包的原始 `.ts` 入口（pi 的加载器
      用 jiti 加载它们——不把值导入我们的类型检查图），**在首次使用时惰性**解析（绝不在模块加载
      时：解析需要 `node_modules`，而编译后的二进制没有）。工作区包的 `pi.skills` 清单不会为
      文件路径条目自动发现——它们的 `skills/` 目录（`pi-spec-graph`、`pi-thinkrail-workflow`、
      `pi-todos`）、宿主自己的 skills（`packages/server/skills`：仅 ThinkRail 的指导，例如
      `managing-mcp-servers`，它在原生 pi 下会是错的，因此不能放在可移植包里）外加注册表扩展的
      拥有者解析 skill 根，都通过 **`additionalSkillPaths`** 接线；`build-support` 的 `skillRoots`
      为两种打包都列出同一个宿主根。注册表扩展本身从不按路径加载：它们在两种模式下都是同样的具名
      内联工厂（一种组合模式；`{ name, factory }` 让诊断中保留 `<inline:visualize>`）。
    - **打包启动器（编译后的 CLI 二进制与打包的桌面运行时）：** 启动器在第一个会话之前等待
      **`registerBundledRuntime({ factories, skillsDir, trashHelpers })` 接缝**——同样三个 pi 包
      以**值导入的默认导出工厂**形式提供（pi 让 `extensionFactories` 与路径加载具备完整的 API 对等；
      损失的东西——文件相对的 `baseDir`、每次重载的重评估——它们都不使用）外加一个暂存的磁盘上
      skills 目录（pi 通过普通 fs 读取 `SKILL.md`，所以 skills 必须活在真实文件系统上；打包运行时
      **只**使用这个暂存根——它绝不从宿主的 `createRequire` 解析扩展自有的包）。注册表扩展不需要
      接缝字段：`@thinkrail/server` 对每个 `@thinkrail/ext-*/server` 的静态导入把它们带进二进制和
      桌面打包。该接缝还执行**打包工件的 pi 注册**：pi 把仅 Node 的 provider 代码藏在打包器不透明的
      变量说明符动态导入之后（这样浏览器打包无法到达 `node:http` OAuth 服务器 / AWS SDK），而单文件
      二进制在运行时无法解析它们——每次 OAuth 登录都死于 `Cannot find module './openai-codex.js'`。
      pi 正好为此提供静态注册接缝，而我们镜像 pi 自己的二进制入口（`pi-coding-agent`
      `dist/bun/cli.js`）：**`registerBunOAuthFlows()`**（`@earendil-works/pi-ai/bun-oauth`）+
      **`setBedrockProviderModule(bedrockProviderModule)`**（`…/compat` + `…/bedrock-provider`）。
      两者都通过**接缝内部的动态字面量导入**加载——字面说明符会被 `bun build --compile` 和桌面
      server 运行时构建静态打包，而 dev（从不调用该接缝）从不加载这些流程模块或 AWS SDK。注册落在
      pi 在登录时查询的同一个 `pi-ai` 实例上，因为目录在整个仓库中钉住了一个确切的 `pi-ai` 版本
      （一个 store 条目 → 一个打包模块实例）。Chat 回收站有两条位于同一注册之后的工件接缝，都由
      `trash` 模块拥有：它静态安装的 procfs 解析器，以及启动器暂存、由 `registerBundledRuntime`
      通过 `setBundledTrashHelpers` 注入的 `trashHelpers`（理由：[[submodule-server-trash]]）。没有
      平台会降级为永久 unlink。
    桌面 server/工厂打包使用 pi 的 `PI_BUNDLED_NODE=true` 编译期 define 构建。那是 pi 自己为
    “已打包但未编译”发行版提供的开关：它选择内嵌模块扩展加载器（jiti 的静态入口，内含 Babel，再加
    pi 的虚拟模块）。没有它，pi 会把该打包当作普通 Node 运行时，并按相对路径去找 jiti 惰性加载的
    `../dist/babel.cjs`，而该文件在单文件包内并不存在，于是 Central 候选加载失败（pi 0.86.0 让
    加载器选择变成运行时相关；0.84.x 总是使用静态入口）。候选加载器还强制 jiti 的转译
    （`JITI_TRY_NATIVE=false`，外加 `JITI_REBUILD_FS_CACHE=1`，使陈旧的转译缓存绝不跨过 pi 升级
    存活）：若允许原生导入，Bun 会自己解析一个外部扩展裸写的
    `@earendil-works/pi-coding-agent` 导入——自动安装第二份 pi 副本，因为
    `~/.pi/agent/extensions` 下没有任何 `node_modules`——而不是把它映射到 pi 的虚拟模块（即打包
    实例）。该 define 与强制的转译合起来就是被测试的工件接缝（共享工件探针的合成扩展会值导入 pi，
    没有它们就会失败关闭）；`server-runtime.ts` 这个文件名只是个名字。
    在每种模式下，可选的 Central 工件都仍是 PI 公开 Jiti 接缝加载的外部文件系统路径；它绝不被打包、
    暂存或复制进 ThinkRail。两种模式都会追加 `extensionFactories`：一个 **headless-search 策略**
    （`headlessSearch.ts`，一个 `tool_call` 钩子，把 `web_search` 的 `workflow` 默认为 `"none"`，
    因为否则 pi-web-access 会在宿主机上打开一个浏览器策展器并阻塞在那里）。它归宿主所有且无条件
    加载，绝不是 `web-access` 扩展的一部分，因为它补偿的是宿主的 UI，也必须覆盖用户自己安装的
    `pi-web-access`——那恰好在该扩展被关掉时运行；`oversizedImageGuard`（上下文级图像尺寸守卫，见
    `imageGuard` 条目）、`skillCarryover`（见其条目），**以及调用方的 `extraFactories`**——按会话
    的宿主绑定（工作区绑定的 subagents 扩展），以值导入方式提供，使 dev 与编译后的二进制走同一条
    路径。pi 自己的内置扩展（`llama.cpp`、`codemode`、`tool-search`、`mcp`）只由 pi 的 CLI 加载；
    SDK 宿主按工厂逐个选择加入。ThinkRail 选择加入 **`mcp` 与 `tool-search`**，使用 pi 自己的
    描述符（`{ name, factory, replaceable: true, builtin: true }`，因此用户安装的 `/mcp` 扩展会替换
    它们，而 pi settings 中的 `-builtin:mcp` 会禁用它们）——只在下文 MCP 一节中的保护措施落地之后才
    注册，因此没有任何构建会在缺少它们的情况下启动服务器；除 pi 自己的 `-builtin:mcp` 之外没有运行时
    开关。`codemode` 与 `llama.cpp` 不被加载（codemode 需要一个 QuickJS worker + wasm 接缝，编译
    后的二进制和桌面打包都没有）。见 [MCP 服务器](#mcp-servers)。
    两条会话路径都把它作为 `resourceLoader` 传入。`buildResourceLoader` 保持内部；接缝及其类型在
    barrel 上。
- **公开接口（barrel）：** 管理器操作（含 `respondToTool` + `settleSessionsForShutdown`）+
  `CreateSessionInput`/`CreateSessionResult` + `SessionEventPayload`；运行时世代门面
  （`usePiRuntime`、候选准备/激活、当前世代 id，以及封闭的 `load-failed` 结果——不含管理器内部）
  外加 `configurePiRuntime`/工厂测试接缝和引导前的 `configurePiRuntimeGenerationInitializer`
  组合接缝；`listModelContextSettings` / `setModelContextWindow` / `setModelContextPublisher`
  （`modelContext` 适配器及其 `provider.changed` 接缝）；`piLoginOptions`（携带惰性创建的安装
  设备 id 的 pi 登录选项，供 `auth` 使用）；`completeOnce`/`pickModel` +
  `OneShotRequest`/`OneShotResult`/`ModelTier`；`webUiContext` 接缝；`toolResponses` 信箱与纯助手
  （`createToolResponseWaiters`/`toolResponseSessionServices`/`assessRespondability`/
  `describeToolResponse`/`buildToolResponseMessage`/`awaitingToolCallId`）；归一化状态操作
  （`listSessionStates`/`acknowledgeCompletion`/`markWorkspaceUnread`/`markWorkspaceRead`/
  `nudgeSession` + 发布者/投影接缝）；`repairDanglingToolCalls`；`liveParentContext` +
  `readChildTranscript`（delegation 嵌入）；skill 目录助手
  `listSkillCommands(cwd, admission)`（已过滤，会话前自动补全）/
  `listSkillCatalog(cwd, admission)`（未过滤，管理器的 `skills.state`）/
  `listProjectAliasSkillNames(cwd)`（现有别名计数）/ `isProjectSkillPath(relativePath)`
  （监视分类谓词）；`reloadSessionResources(sessionId)`（活跃聊天重载；重载门禁被阻塞时抛错）+
  `requestSessionReload(sessionId)`（延迟变体：此刻 `"reloaded"`，或直到门禁打开为止 `"deferred"`
  ——门禁打开指结算、一个已准入 prompt 的释放，或一次压缩的结束；请求会合并）；**`setSkillAdmissionResolver`**
  接缝（宿主接线 `workspaceId` → 准入上下文，由 `admissionContextFor(project, overrides)` 一次性
  派生）；**`setMcpPolicyResolver`** 接缝（宿主接线 `workspaceId` → 项目的 MCP 审批、覆盖与工具
  授权，由 `mcpPolicyOf(project)` 派生）及其写入孪生 **`setMcpToolGrantWriter`**；MCP 管理接口——
  `listMcpServers`、`mcpOwnershipGuard`（解析 pi settings，然后同步回答谁在管理 MCP 而不是
  ThinkRail——即 `listMcpServers` 报告为 `handledElsewhere` 的那个——因此宿主在与每个管理变更相同的
  调用栈中检查它）、`refreshMcpStatus`、`setMcpStatusPublisher`、`mcpSessionView` /
  `liveSessionIdsOf`、`dispatchSessionMcpCommand`（带一个可选的 `notify`，接管该分发的通知而不是
  由聊天接收）、`reconnectMcpServer`、`readMcpServerLog`（一个服务器在 pi 共享 `mcp.log` 中的
  已掩蔽行）、`setSessionMcpServerEnabled`（聊天本地禁用，通过重载门禁应用）、配置的有效配置读取
  `effectiveMcpServerConfig` 与配置写入器 `writeMcpServerEntry` / `removeMcpServerEntry` /
  `mcpEntryFingerprint` / `shareMcpOverrideWithRepo`（pi 规则校验，`MCP_CONFIG_INVALID` /
  `MCP_PATH_UNSAFE`）以及登录探测（`startMcpProbe`、`replyMcpProbe`、`cancelMcpProbe`、
  `cancelMcpProbesOwnedBy`、`cancelAllMcpProbes`、`setMcpLoginPublisher`、`isMcpLoginId`）；
  pi 级信任接口——`applyPiResourceTrust(workspaceIds)`（对信任已过期的每个活会话做受门控的重载）、
  `piTrustsProjectPath(path)`（pi 是否已信任某个路径：按最近祖先取它的 `trust.json` 决策，否则取
  其全局 `defaultProjectTrust` 是否为 `always`；不可读的 store 回答 `false`——宿主为新添加项目做的
  播种）以及 `projectTrustSummary(cwd)`（信任通知的事实：别名名称与存在的
  `ProjectResourceKind`——pi 的 `hasTrustRequiringProjectResources` 门控的 `.pi/` 条目、祖先
  `.agents/skills`，以及来自 `pi-subagents` 的 `discoverAgentDefinitions` 的项目定义；该条目列表
  镜像 pi 私有的那份，去掉 `mcp.json`，其条目作为 `mcpServers` 单独计数，因为每个还需要自己的审批，
  所以一次门控了新条目的 pi 升级必须扩展它）；subagent 策略接缝
  **`setSubagentsEnabledResolver`** + **`refreshSubagentTools`**（宿主解析生效的全局默认值加工作区
  覆盖；管理器拥有活会话的激活时机）与并发接缝 **`setSubagentConcurrencyResolver`** +
  **`refreshSubagentConcurrency`**（同样的划分；`delegation` 拥有把上限应用到其缓存服务）；
  扩展开关接缝 **`setDisabledExtensionsResolver`** + **`refreshExtensionTools`**（宿主拥有集合；
  本模块拥有组合期过滤与活激活时机）；`set_title` 接缝（`setTitleToolHost` +
  `TitleToolHost`/`SET_TITLE_TOOL_NAME`/`SetTitleParams`）；打包工件接缝（`registerBundledRuntime` +
  `BundledExtensions`/`BundledExtensionFactory`）。
- **允许的依赖：** `@earendil-works/pi-coding-agent`（运行时）；`@earendil-works/pi-ai`（类型 +
  测试夹具 + **从包根值导入的纯目录助手**——今天恰好是 `getSupportedThinkingLevels` +
  `clampThinkingLevel`，对 `Model` 的纯数据投影；*分发*仍经过共享 `ModelRuntime`，绝不用 pi-ai 的
  stream/complete——外加 `/bun-oauth` + `/bedrock-provider` + `/compat` 子路径，只在
  `registerBundledRuntime` 的动态导入内部值导入）；`pi-delegation` + `pi-subagents`（可移植的
  delegation 运行时与 Agent 工具组合，由宿主嵌入值导入）；`pi-background-commands`（会话绑定的
  命令能力，同样由其宿主嵌入值导入）；`pi-spec-graph` + `pi-thinkrail-workflow` + `pi-todos`
  （随附的 pi 包——父会话通过资源加载器路径或启动器工厂加载该集合；被委派的子会话值导入
  `pi-spec-graph`）；同级 `extensions` 注册表模块（ThinkRail 扩展的内联工厂与 skill 根——`agent`
  从不直接导入 `@thinkrail/ext-*` 包或其 pi 包；barrel 保持 Node 可加载以服务工作流测试装置
  （[[module-workflow-tests]]），因为一个加载原始第三方 `.ts` 的注册表扩展会惰性加载它，见
  [[module-ext-web-access]]）；`typebox`（宿主工具的参数模式）；`jsonc-parser`（`modelContext` 中
  定向的 `models.json` 编辑）；`trash`（只通过同级 **`trash` 模块**到达——见
  [[submodule-server-trash]]：一条路径、禁通配、允许抛错、绝不降级为 `unlink`；启动器的暂存助手与
  procfs 解析器接缝也在那里，因为 `changes` 的整文件回退需要同一原语）；`contracts`
  （`PiEvent`/`Model`/`ThinkingLevel`/`ImageContent`/`SessionStats`/`SessionSummary`/
  `Session*Payload`/`SlashCommandInfo`/`ExtUi*`/`AskUserQuestion*`/`ProviderStatus*`）；`log`
  （诊断 + 会话生命周期 debug 轨迹）；`persistence`（delegation 的 `dataDir` 加窄的会话回执
  store）；`trash`（可恢复删除原语）；Node。
- **禁止：** `host`；除 `log`、`trash` 和那些窄持久化接口之外的兄弟功能（会话工作树 `cwd` 始终是
  输入，绝不是工作区注册表查找）；Central 进程/文件系统知识——调用方只为一个候选提供所需的不透明
  扩展路径。

## 工具输出与上下文恢复（Tool output and context recovery）

可移植工具拥有面向模型的有界输出（[[submodule-spec-graph-tools]]、[[submodule-pi-todos-tools]]）；
宿主不包装每个工具，也不重写用户输入。Pi 原生的 read/bash/MCP 限制仍然权威。结构化结果的 details
是展示元数据；度量的是面向模型的内容，而不是序列化后的 transcript 或 SDK 上下文对象。

Pi 拥有聚合预算、压缩与恢复；有界的工具输出并不保证整个请求被准入。宿主不添加压缩覆盖，也不添加
重试循环。已知的上游上下文预算限制及其升级检查见 [[pi-compatibility]]。

## Provider 流分帧（Provider stream framing）

Pi 拥有 provider 重试；宿主从不添加重试。Pi 只重试匹配其可重试模式的错误："Anthropic stream ended
before message_stop" 匹配，"Could not parse Anthropic SSE event" 不匹配。
`patches/@earendil-works%2Fpi-ai@1.4.0.patch`（根 `patchedDependencies`，上游
earendil-works/pi#10390、#564）修复了 pi-ai 的 `anthropic-messages` SSE 解析器中两个会以该解析
错误结束轮次的缺陷。这两处修复的结果不同：

- **分裂的 CRLF。** 一个 `\r\n` 在 `event:` 行之后被拆到两个网络块中，被读成两个换行。那个空行会在
  缺少 `data:` 行的情况下分发该事件，于是一个完整、有效的 CRLF 流失败。现在末尾的 `\r` 会等待下一个
  块，流便能成功。最后的排空仍会在孤立的 `\r` 处结束一行。
- **被截断的流。** 一次截断使其缺少空行的事件，过去会以空或部分数据被分发。现在它被严格解析、绝不
  “修复”，并在为空、不可解析或不是对象时被丢弃，因此截断会以可重试的截断错误结束，例如 "stream
  ended before message_stop"。一个确实可解析但未完成的事件仍会被产出，因此缺少 `message_stop` 之后
  最后一个空行的流仍然成功。

一个已结束但格式错误的事件保留不可重试的解析错误。观察到的失败携带空的 `data=`，`raw=` 中只有
`event:` 行。分裂的 CRLF 总是留下这种签名，而随机截断很少如此，所以分裂的 CRLF 很可能是用户撞到的
情形。未确认：是 api.anthropic.com 还是路径上的某个代理在发送 CRLF。守卫测试与移除门禁见
[[pi-compatibility]]。

## 会话标题（Session titles）

`agentSessionManager` 是持久聊天标题的唯一写入者。它的 `renameSession(sessionId, workspaceId, cwd,
title, { onlyIfUnnamed? })` 校验一个非空白、单行且在契约长度限制之内的标题，严格在所给
workspace/cwd 内解析该会话，并在归一化标题已是当前值时避免追加。活会话通过
`AgentSession.setSessionName` 写入；仅有磁盘的会话用
`SessionManager.open(...).appendSessionInfo(...)` 打开其确切的 transcript，而不挂接 agent，也不
解析模型。两条路径都发布同一个 `session_info_changed` Pi 事件，而 `SessionSummary.title` 仍是水合
投影。

受门控的写入在异步竞态中仍然权威。`onlyIfUnnamed` 紧挨着追加处做检查，是 `set_title` 处理器使用的
比较并设置（其 `false` 返回值正是处理器得知聊天已命名的方式）；手工的线上契约变更是无条件的。因此
agent 给出的标题绝不可能覆盖一个持久名称，而第一个名称就是最终的。这里不需要生成/手写的来源标记或
标题 sidecar：pi 名称的缺席与存在就足够了。架构中已接受的“不跨进程协调”规则仍然适用。

**标题旁边的首个 prompt。** 管理器投影的每个 `SessionSummary` 与 `SessionStateRecord`
（`summaryOf`、`listSessions` 的磁盘行、状态发布以及 `listSessionStates` 的来源）还携带
`firstPrompt`——契约的 `firstPromptHead`，取 transcript 中第一条有文本且不是控制消息的用户消息——
因此客户端可以为 agent 尚未命名的聊天打上标签。只有一个派生函数，`firstPromptOf(entries)`，它为
活会话与磁盘会话一样读取**完整**的 `SessionManager.getEntries()`：`AgentSession.messages` 是压缩
感知投影，压缩之后的第一条用户消息不再是聊天的首条 prompt，而 pi 的 `SessionInfo.firstMessage`
在没有文本时会替换成一个占位字符串。活条目无论找到与否都缓存该答案，并只重读自上次扫描以来追加的
条目，且仅在 `getEntryCount()` 增长时重读，因此按事件的状态发布绝不会重扫一个用户轮次全是包或图像
的 transcript。该值是只读的：没有写入路径接受它，`renameSession` 与 `onlyIfUnnamed` 不受影响，而
已命名的聊天仍携带它，以便客户端显示标题并把 prompt 保留为工具提示。

## Chat Resources 集成（Chat Resources integration）

管理器为每个父聊天在它的 Pi 会话旁边保留一个 [[module-pi-background-commands]] 服务，并通过常规的
资源加载器路径注入其扩展。同一绑定在源码宿主与打包宿主中都提供活会话上下文、生效的 shell settings
与生命周期。第一版只把该能力加载进父聊天，而不是精选的隐藏子会话扩展集；普通子会话的 Bash 仍在其
transcript 中可见。

一个小的 agent-barrel 门面提供 [[module-contracts]] 中定义的资源快照、历史分页、命令输出/停止以及
subagent 停止/全部停止操作。活行来自命令服务加上 `DelegationService.childrenOf(parent)`；没有聚合
注册表持有其生命周期的副本。已完成的行来自 [[submodule-server-resource-history]]，与拥有者仍持有的
内容合并——拥有者自己的终止记录胜过一条已归档的在飞记录，因此归档写入失败绝不会把已完成的工作变成
Interrupted。快照携带每条活记录加上每种类型最新的二十条已完成记录以及按类型的结局计数；更旧的内容
通过 `session.resourceHistory` 分页，而一旦拥有者驱逐或丢失该命令，`backgroundCommand.output` 就回退
到归档的日志尾部。subagent 摘要包含直接的前台与后台子会话（计划审查者也算），并省略尚无运行快照的
句柄；较旧的句柄与 transcript 从不因为该窗口而被处置。任务与角色摘要上限为 2,000 和 200 字符。
每个工作区服务一个生命周期订阅，外加每个命令服务的变更订阅，通过宿主注入的发布者发布有作用域的失效；
不添加按 token/输出的广播。

同一批订阅同步喂养归档：命令准入、停止请求与终止边界（带保留的日志尾部），subagent 的
排队/启动/终止边界（带终止时间）。命令的归档订阅活得比管理器自己的订阅更久，只有在命令服务的处置
结算（或其余额耗尽）之后才关闭，因此拆除本身造成的终止边界仍会被记录。归档写入失败会被记录日志，且
对同一状态绝不重试；拥有者的记录在其存活期间仍然胜出。**Interrupted** 在这里一次性决定：一条命令在
其聊天资源正在关闭时以 `stopped` 结束，且在关闭开始之前没有请求过停止，就会被归档为 `interrupted`，
而一条在级联放弃时仍未结算的命令同样如此（在飞，带着它至今的日志）。对 subagent，关闭级联自己标记
其取消：在 `disposeChildrenOf` 之前它以原因 `interrupted` 中止每个子会话，而由于
[[module-pi-delegation]] 保留一次 run 的第一个取消原因，只有先被拆除取消的 run 才被归档为
`interrupted`。拆除落地时已被取消的 run 保留自己的来源并读作 Stopped——用户停止（`"user"`）以及
被父会话自己的 Stop 通过其工具信号取消的前台子会话（该信号不携带原因）都是如此（PR #1004 上的一个
AIR 评审发现：从缺失的原因推断拆除，恰好把那次 run 标错）。仅仅在拆除期间结算的用户停止或工具停止
仍保持 Stopped。被中断的工作只呈现给用户：它绝不唤醒父会话，也没有任何东西告诉 agent 它已确认的 run
已经消失（用户结算，2026-10）。死掉的宿主留在飞行中的工作则是归档自己的派生。工作区归档在其
delegation store 之后移除该工作区的历史；已退役 delegation 服务的迟到生命周期事件不会被归档，因此
它们无法重建它。聊天删除会把历史留在原处。

每个操作在触碰句柄之前都会校验真实的工作区/会话成员关系、子会话谱系以及管理器的删除门禁。已持久化的
父会话使用既有的单飞挂接路径；未知的父会话绝不冒充空目录。缺失/外来/被驱逐的资源共享
`RESOURCE_UNAVAILABLE` 路径（只有命令输出在父会话校验之后返回 `available:false`）。资源控制绝不
依赖 UI 是否已看到某个工具事件，也不依赖启动/重启一个 provider 轮次。自然完成与请求的取消仍是来源
结局；已停止状态不会从一个已确认的 RPC 合成。

用户 subagent 控制提供 [[module-pi-delegation]] 定义的 `"user"` 取消原因；完成投递属于
[[module-pi-subagents]]，而不是某个宿主抑制集合。全部停止在返回其目标计数之前向所有已捕获的活子会话
发信号；单个与批量控制只确认意图，不等待 provider/工具结算。它们的分离 abort promise 总是带拒绝
处理器，生命周期失效仍是终止权威，两个控制都不把 `disposeChildrenOf` 当替代品使用。

命令服务活得比视图摆放和父轮次取消更久。实际的会话处置与工作区归档在既有宿主关闭预算之内关闭命令
准入，并在等待拆除之前向命令/子会话工作发信号。按会话的拆除墓碑阻止已持久化的父会话在其上一次资源
级联结算之前重新挂接。工作区归档关闭一个世代，并在移除 delegation/transcript 存储之前等待在飞的父
会话准备，因此陈旧的准备无法在拆除之后注册。资源关闭对所捕获的每个工作区父会话都在任何父会话 abort
被等待之前运行；单个移除同样在等待主轮次之前向资源发信号。流式的破坏性移除复用管理器的排空队列
Stop 路径，丢弃被排空的输入，同时在父会话被处置之前保留同样的有界已接受答案持久化宽限期。仅主轮次
的 Stop 绝不关闭 Resource 拥有者。
命令与分离 subagent 的完成投递尊重待处理的会话删除及其回滚；任何通知都不得在一个正被移入回收站的
transcript 之后追加或唤醒父会话。墓碑是临时的，且不处置任何一个拥有者。资源重载保留命令服务与可移植
的 `Subagents` 拥有者，并重新绑定其完成发送方。管理器不保存任何完成队列。`SessionManager` 在 SDK 会话
创建之前就已存在，为该服务提供不可变的身份。它的上下文在启动时读取当前会话的
cwd/model/thinking/session 文件以及生效的 SettingsManager shell 路径/前缀。完成需要那个确切的已注册
拥有者、没有删除墓碑、没有资源关闭、也没有可恢复的活问题。一条不唤醒的用户停止通知不得在一个未应答
的工具调用之后追加并破坏仅尾部重启修复。在投递于既有可移植拥有者中等待期间，资源检查与取消仍然可用。
原生 `turn_end` 结果边界清除问题阶段并以 `{ atTurnEnd: true }` 冲刷两个拥有者——这是结果仍能加入正在
运行的父会话下一次请求的最后安全点——而 `compaction_end` 在 pi 不再压缩时冲刷它们一次，注册、资源
重载和删除回滚同样如此；父会话繁忙时任何其他冲刷都是空操作，因为拥有者在步骤边界投递
（[[module-pi-subagents]] Result delivery）。`closeSessionResources` 在子会话取消或任何 await 之前
同步处置 subagent 拥有者，包括准备失败的情形。永久关闭会挺过关闭预算到期并抑制迟到的结局；Pi 的
处置不会发出扩展关闭事件，因此级联在同步拥有者处置之后自己发出 `session_shutdown`（见
[MCP 服务器](#mcp-servers)）。既有的缓存资源级联仍是唯一的拆除拥有者。SDK 创建与条目准备失败都会
关闭拥有者。注册之后的失败还会通过常规拆除路径移除那个确切条目，而不是把一个已处置的会话宣传为活。
重启后的宿主没有控制句柄，也没有保留的命令输出可用来从历史重建。

`getSessionResources`、`readBackgroundCommandOutput`、`stopBackgroundCommand`、`stopSubagent`、
`stopAllSubagents` 与 `setSessionResourcesPublisher` 只通过本模块的 barrel 公开。
`stopBackgroundCommand` 在结算之前就确认，因此一个停止命令的测试会在结束之前等到 `stopped`，而测试
发布者（进程全局）会忽略其他聊天的通知，并保持它启动的任何读取都被处理：一条迟到的 `stopped` 通知
曾经到达了下一个测试的发布者，其被拒绝的读取变成一个未处理的拒绝，在该发布者仍安装期间被 bun 记到
其后每个测试头上。它唯一新增的外部依赖是 `pi-background-commands`；没有 `agent` → `terminal`、
`subprocess`、settings 或 workspaces 的边。所属的父图记录这一包依赖。

## MCP 服务器（MCP servers）

pi 内置的 MCP 是唯一的 MCP 引擎：连接、OAuth、暴露、资源、结果截断和 `/mcp` 命令都仍归 pi；
ThinkRail 拥有一个多会话 GUI 宿主在其周围应当拥有的东西。

- **注册。** 上面提到的两个内置描述符，仅按父会话创建，由会话的 `createMcpSessionHost` 连同调用
  守卫与状态失效回调一起构建。管理器注入守卫的两个端口——`askForToolCall`（→
  `selectForToolCall(sessionId, { toolCallId, toolInput }, …)`，因此审批锚定到它的调用，即便在并行
  调用之间也是如此，并列出该调用的输入）与 `grantTool`（→ 授权写入器接缝）——而宿主注入该工作区的
  项目策略（`setMcpPolicyResolver` → `Project.mcpApprovals` / `mcpOverrides` / `mcpToolGrants`，
  失败即关闭为无）。子会话（subagent）两者都不加载——精选子会话集合不携带引擎——而 `Agent` 工具的
  描述说明 subagent 无法调用 MCP 工具。父聊天把 pi 的 `openUrl` 变成空操作（远程客户端绝不能让人
  在宿主机上打开浏览器）；pi 的 rpc 模式 `/mcp login` 仍显示 URL 和回贴对话框。在 Central 排除
  加载器模式下，两个内置扩展都经过 `extensions` 下描述的每次重载路径重新解析，因此 `-builtin:*`
  在那里同样生效。
- **配置**（[[submodule-server-agent-mcp]]）。pi 的 `mcp.json` 文件是事实来源
  （`~/.pi/agent/mcp.json`、受信任的 `.pi/mcp.json`）。ThinkRail 提供的加载器是 pi 的
  `loadMcpConfig` + 校验的移植（对等测试用同一批夹具跑 pi 已安装的加载器，并在每次 pi 升级时重新
  检查），因为 pi 不导出它们；它还应用项目记录的启用/暴露覆盖、丢弃用户尚未批准的仓库定义条目，
  并在 codemode 缺席期间于内存中把 `codemode` 暴露归一化为 `deferred`。配置写入器归 ThinkRail
  （同步临时文件重命名、保留未知键、只触碰定义文件、不允许符号链接的 `.pi/` 路径、每个 UI 中掩蔽
  机密字面量）。由第三方扩展注册的服务器绕过该加载器；`mcp` 描述符的工厂把其 `session_start` /
  `mcp_servers_change` 处理器注册在 pi 的处理器之前，因此只要 `pi.getMcpServers()` 非空，
  `tool_search` 就在 pi 检查可达性之前已激活（没有 “neither is active” 警告，而当另一个扩展替换了
  内置扩展时也无事可做）。
- **信任。** 宿主为某个项目构建的每个 `SettingsManager` 都传入其真实的 pi 级信任
  （`Project.piResourceTrust`，[[submodule-server-projects]]），而不是强制的 `true`。
  - **位置。** 会话（`buildSessionSettings`）、会话前加载器与子会话。只有别名名称列表传 `false`：
    别名根是显式路径，因此它从不需要项目 settings。
  - **每次加载。** 会话加载器的 `reload()` 在每次加载之前应用准入上下文的当前信任
    （`setProjectTrusted`），因此任何重载——授予、配置变更或手动重载——都恰好加载记录所允许的东西。
    所以一次授予或撤销只需请门禁重载那些信任已过期的会话（`applyPiResourceTrust`：空闲的立刻，繁忙的
    在结算时）。信任变更时仍在打开的会话不在该扫描中，因此注册会把其已加载的信任与记录重新核对，并
    在 `createSession` 返回之前重载它。
  - **子会话。** 子会话的信任来自其父上下文：`liveParentContext.isProjectTrusted` 回答父会话*已加载*
    的信任与当前记录的与。因此子会话所持的信任绝不多于其父会话（一次在繁忙父会话上被推迟的授予，在
    父会话重载之前不会到达其新子会话），也不多于记录（一次撤销会立即到达新的子会话）。
  - **撤销。** 子会话在创建时捕获其信任，而 `pi-subagents` 会跨父会话重载保留子会话，因此撤销还会
    停掉一个陈旧父会话的每个活子会话（`stopSessionChildren`，原因 `"user"`：撤销是用户自己的动作，
    且在不唤醒父会话的情况下告知父会话）。它等到那些 run 真正结束——abort 失败的子会话被处置——之后
    才请求父会话重载。**繁忙**的父会话不会被打断：它当前轮次用已经加载的资源跑完，并在结算时重载
    （`"deferred"`），而信任 Stop 的确认框会把这点披露出来（“open chats reload when idle”）。因此
    `project.setTrust` 返回时，已没有子会话、也没有空闲聊天在已撤销的信任下运行，而繁忙聊天在其轮次
    结束时停下。
  - **失败。** 失败的重载在返回的处置中报告为 `"failed"`，随后 `project.setTrust` 在持久化并发布
    记录之后失败，并点名仍处于先前信任下的聊天。之后失败的延迟重载会作为该聊天的 `error` 通知到达
    用户，请求关闭或重载它。授予会让运行中的子会话保持原样，因为它们只是缺少项目的资源。
  - **同意。** pi 级授予是它自己的同意，与别名信任分开：只有带 `resources: true` 的
    `project.setTrust`，由一个点明它加载了什么的界面发出，才授予它
    （[[submodule-server-projects]]）。仓库定义的 MCP 条目额外需要一个按条目的审批，键为整个原始条目
    对象的指纹（审批状态在项目记录中）。信任通知对别名 skill、pi 自有需要信任的资源、项目 subagent
    定义与项目 MCP 条目一视同仁地显示。
- **生命周期。** `AgentSession.dispose()` 从不发出 `session_shutdown`，因此资源级联
  （`closeSessionResources`）自己运行它——同步地，在同步拥有者处置之后，从而让处理器在同步拆除路径
  处置 pi 会话之前已经开始。pi 的 runner 按加载顺序逐个 await 处理器，而文件扩展先于 `builtin:mcp`
  加载，所以一个永不结算的处理器会让 pi 的 MCP 关闭永远无法开始（pi 的进程退出钩子只发 SIGTERM）。
  因此级联首先直接运行 `mcp` 工厂自己的 `session_shutdown` 处理器（`McpSessionHost.shutdownEngine`，
  带上事件与 runner 的 `createContext()`——pi 会传的东西），然后为每个扩展运行
  `session.extensionRunner.emit({ type: "session_shutdown", reason: "quit" })`。被捕获的 MCP 关闭与
  由 runner 发出的 MCP 关闭共享同一个按加载的清理 promise，因此过早清空 pi 的服务器列表不会让第二次
  分发丢掉第一次分发的完成（[[submodule-server-agent-mcp]]）。受门控的重载在调用 pi 的
  `session.reload()` 之前以原因 `reload` 等待那次真正的 MCP 清理；它们绝不使用有界的退出等待在旧
  资源仍然活着时准入替代品。一个挂起的无关关闭处理器可能延迟重载，但无法让 MCP 进程跨过一次信任或
  资源变更继续存活。关闭检查在重载之前、在公开的 `beforeSessionStart` 钩子中以及在分发之后进行，
  若条目已处置或其资源正在关闭则拒绝。仅有该钩子不够：更早的异步 `session_start` 处理器可能通过
  关闭暂停分发，因此引擎用该加载同步关闭的标志来守护其实际的启动/变更回调。关闭不等待无界的资源
  加载器或启动分发。若 pi 之后构建了替代 runner，重载拥有者会给那个 runner 一个自己的有界关闭并
  处置它。关闭按 runner 单飞，包括紧急处置在重载中途紧接礼貌关闭而来的情形；每个处置续段只能处置
  它所等待关闭的那个 runner。已关闭的重载绝不冲刷完成、不重启 MCP 状态、也不安排另一次重载。
  每条拆除路径都经过该级联：删除/移除、工作区归档/移除、退出（`settleSessionsForShutdown` +
  `disposeAllSessions`）以及绑定之后失败的一次准备——包括一次因工作区已关闭或宿主期间开始退出而注册
  被拒绝的准备。它们中的每一个都把 `session.dispose()`（`disposePiSessionAfterShutdown`）推迟到
  两者都结算或余额（`SESSION_SHUTDOWN_BUDGET_MS`，3 秒——pi 在 2.5 秒内逐级升级 stdin-close →
  SIGTERM → SIGKILL）耗尽为止，因为 `dispose()` 会使处理器仍在使用的扩展上下文失效；宿主关闭的
  结算余额就是同样的 3 秒。保留的会话保留其服务器（不驱逐）；它们作为 `SessionResources` 的第三个
  列表出现（`mcpServers`：名称、状态、工具数、来自会话最新状态快照的传输层，对于扩展注册的服务器
  则是 `registered: true` 加该注册的传输层；变更会发布 `session.resourcesChanged`），按聊天的
  “Disable in this chat” 在下一次空闲重载时应用（延迟的那次会让服务器立即显示为 `pending-reload`）。
  该覆盖只到达 `mcp.json` 服务器，因此 `setSessionMcpServerEnabled` 拒绝
  （`MCP_CONFIG_INVALID`）禁用一个聊天并未从其文件加载的服务器——对已注册的服务器，消息说明它由
  扩展注册、无法按聊天禁用——并在 `builtin:mcp` 不拥有 `/mcp` 的地方以
  `MCP_HANDLED_ELSEWHERE` 拒绝。所有权与文件来源在启用和禁用之前都会被校验，即便对空操作请求也是
  如此；一个在替换 `/mcp` 的重载中保留下来的已禁用条目无法通过宿主重新启用。失败的重载绝不回滚
  聊天本地的禁用意图；在其 MCP 重载失败仍然存在期间重复同一值会重试，而不是返回“未变更”。若该失败
  清空了已加载快照，重试会针对当前已准入的配置校验文件来源；未知或仅扩展的名称仍被拒绝，而替代的
  命令拥有者仍会让该控制变为只读。配置或信任变更只重载受影响的会话：会话的 `loadConfig` 记录 pi 在
  其上次启动时收到的每服务器有效配置（`McpSessionHost.loaded()`，在每次加载开始时清空），而
  `reconcileMcpSessions` 把它与加载器现在会产出的结果比较（`pendingMcpReload`）——例如一个未获批
  的仓库条目什么也没改变，因此什么也不重载。一个 `/mcp` 被另一个扩展拥有、或无人拥有的会话没有可比
  较的对象（内置扩展的加载器从不在那里运行），除非先前一次失败的 MCP 重载仍可重试，否则不参与协调。
  协调在每次宿主变更之后运行、在用户文件变更时运行（`watchUserMcpConfig`：pi 的 agent 目录上一个
  目录监视器，300 ms 去抖；缺失的 agent 目录会先创建——pi 本来也会在首次使用时创建它——因此用户
  之后添加的文件仍会被看到；重命名或监视器错误会退役旧监视并以有上限的重试退避重新布防，包括在
  agent 目录被重建之后；停止会取消重试）、在工作区变更 nudge 触及 `.pi` 时运行，以及在每个会话的
  `agent_settled` 时运行（因此 agent 自己做的编辑会在 run 结束时应用）；一次延迟的重载会让变更的
  服务器显示为 `pending-reload` 直到它运行。每个影响 MCP 的受门控重载（即时、延迟、手动、信任/
  settings，或聊天本地覆盖）都在 `mcpSessions` 中共享一个失败记录器：既有的被跟踪状态对象即便在
  失败的加载已经替换或清空其配置快照之后仍保留重试标记。门禁会捕获内置引擎在替换之前是否已加载；
  该来源信息、一个新加载的引擎，或一次既有失败，都会使该重载被视为影响 MCP。从未管理过 MCP 的宿主
  不会获得 MCP 失败标记。每次失败尝试记录日志并通知一次，细节已脱敏；延迟与协调的调用方不会再次报告
  它。失败不会重启权威状态，也不会立即排空另一个待处理重载；之后的协调或显式重试可以在没有任何配置
  差异的情况下恢复。只有成功的重载才清除标记并重启状态。正在关闭或已处置的条目既不报告失败也不重启。
  重载经过 `session.reload()`，位于一个按会话的重载门禁之后：重载运行期间，每个准入接缝
  （`promptSession`、`steerSession`、`followUpSession`、`nudgeSession`、队列重入、
  `respondToTool`、`sendReviewFixToSession`、`compactSession`）都以 "reloading" 拒绝，而完成投递
  在其拥有者中通过 `canDeliverCompletion` 等待并在之后冲刷；门禁只在 pi 为 `isIdle`、没有已准入的
  prompt 仍在预检中（宿主从准入到 `agent_start` 保留该窗口——pi ≤ 1.1 在整个窗口内都报告空闲，
  pi 1.2 也仍如此直到 `before_agent_start`，穿过 `input` 处理器与展开；一个被当作斜杠命令处理的
  prompt 从不启动 run）、没有排队消息移除正在把保留的消息放回（`removeQueuedSession` 从它的排空到
  最后一次重入持有一个 `agent_start` 不会释放的预留，因此在此期间结算的 run 无法启动一次会拒绝——
  并丢失——其余部分的重载），并且没有手动或自动压缩在飞行中时才打开；待处理的重载在
  `agent_settled` 时、在每个预留释放时以及压缩结束 / `compactSession` 的 `finally` 中被重新考虑。
- **状态**（`mcpSessions.ts`）。被暴露，而不是被重算：连接真相是 pi 的无头 `/mcp` 文本，通过分发
  会话自己的命令获得（`dispatchSessionMcpCommand` → `extensionRunner.getCommand("mcp").handler`）。
  捕获按来源进行，绝不按格式：该分发得到 `createCommandContext()`，其 `ui` getter 被重定义
  （`Object.defineProperty`，因此 pi 的其他惰性 getter 保持不动）为 pi UI 的一个转发代理，只有其
  `notify` 会向捕获报告。因此它只取自己那次分发的通知，它打印的任何东西都不会到达聊天，而用户自己的
  `/mcp`（pi 的 prompt 路径，pi 的上下文）总是在那里打印，即便在捕获中途也是如此。重连
  （`reconnectMcpServer`：`mcp.reconnect` 以及登录后的那些）以同样方式分发：成功是静默的，pi 的错误
  通知变成该调用的拒绝（经 `redactMcpText`），而 `builtin:mcp` 不拥有 `/mcp` 的聊天会以
  `MCP_HANDLED_ELSEWHERE` 被拒绝。pi 启动时的 “MCP servers need attention” 通知是未经请求的，所以
  只有它凭其固定前缀通过 `interceptExtUiNotify`（[`webUiContext`]）被识别，且在 `bindExtensions`
  之前安装：它变成一次部分状态更新加一次完整刷新，而不是一个 toast——但低于 `MCP_PROTOCOL_VERSION`
  的客户端例外，它们未订阅 `mcp.status`，仍按 toast 收到该通知（`notifyExtUi` 带一个
  `ExtUiAudience`，按 socket 投递并在投递前脱敏），因为 web 客户端与宿主各自独立发布。pi 的 `/mcp`
  在作答之前会等待每个启动连接，因此注册以及每次成功的受门控重载都会先用 pi 得到的东西
  （`McpSessionHost.loaded()` 加上扩展注册的服务器）播种快照：每个已启用的读作 `starting`，一个
  已禁用的经同一推导读作其禁用状态，而上一次加载的报告绝不跨重载存活。之后开始一次捕获；在此之前
  开始的那个会被丢弃（按会话的加载纪元），因此陈旧报告绝不会替换种子。刷新还在 `agent_settled` 时
  以及在一个聊天自己的 MCP 登录/注销/重连结算之后运行，包括失败时；状态读取从不自我失效。捕获在读取
  调用方的等待余额之后继续，并发布它最终得到的答案。每个会话至多一个捕获在飞行中，调用方等待一个有界
  时间。每个快照把 pi 的报告与配置合并（`deriveMcpServerStatuses`：被本聊天 / 项目禁用、待审批、
  配置无效；pi 未作答时上一次报告胜出），携带按会话单调递增的世代，并发布到 `mcp.status` 频道。
  pi 不脱敏其状态文本（连接错误、HTTP 错误响应体、stdio 服务器的 stderr 尾部），所以每个 `detail`
  和 `configError` 在离开宿主之前都经过宿主的 `redactMcpText`（[[submodule-server-agent-mcp]]）。
  配置问题会被报告而绝不丢弃：每个具名条目保留其摘要（一个不是对象的条目读作
  `server "<name>" must be an object`），而 `listMcpServers` 把两个文件级的问题（不可解析的 JSON、
  `mcpServers` 不是对象、类型写错的键；`summarizeMcpConfigErrors`，已掩蔽）作为 `configErrors`
  加入——pi 自己对它们的报告正是宿主所拦截的启动通知。只有 `/mcp` 属于 `builtin:mcp` 的会话才被查询；
  其他拥有者会让 `listMcpServers` 报告 `handledElsewhere`（`by` = 该扩展的路径，或对一个没有 `/mcp`
  的活聊天为 `pi settings`），而把 `/mcp` 交接出去的那次重载之后的刷新会一次性退役该会话的状态与
  资源行（一个空的 `mcp.status` 快照，一次 `session.resourcesChanged`）。关闭内置扩展的 pi settings
  在没有聊天打开时也已知：`listMcpServers` 像排除加载器那样解析它们（`DefaultPackageManager` 带
  内置扩展的名称，取 `builtin:mcp` 资源的 `enabled`；缺失的包被跳过，绝不安装）并报告
  `by: "pi settings (-builtin:mcp)"`。一个已启用的非内置扩展是否注册 `/mcp` 在不加载它的情况下
  不可知，因此替代品只能通过某个活聊天的拥有者看到。
- **调用。** 按会话的宿主扩展（[[submodule-server-agent-mcp]]）在任何没有 `readOnlyHint` 的
  `mcp__*` 调用之前，在该调用的卡片上询问（Deny · Allow once · Allow in this chat · Always allow；
  除肯定回答之外的任何东西都会阻塞，而一条用户消息像任何对话框一样回答它）。“Allow in this chat”
  是会话记忆；“Always allow” 是通过 **`setMcpToolGrantWriter`** 接缝写入的项目授权（宿主 →
  `grantProjectMcpTool`；写入失败会被记录日志，调用仍然运行），并在每次调用时通过策略解析器读回。
  它的 `tool_result` 处理器在 `details.thinkrail` 下添加有界、无 base64 的 `McpResultSummary`
  （`summarizeMcpResult`），而 `readMcpToolOutput` 从某个结果确切的 `fullOutputPath` 提供其记录的
  完整输出（`mcp.readOutput`）（pi 的仅拥有者可读临时文件；至多 1 MiB，超出则 `truncated`）——对
  没有记录输出的结果是 `unavailable`，文件消失后是 `expired`。Settings 的 *Show log* 读取
  `readMcpServerLog`（[[submodule-server-agent-mcp]]）：pi 把服务器记录的内容
  （`notifications/message`）按 agent 目录追加到一个 `mcp.log`，供所有会话共用，因此一个服务器的行
  按其条目头挑选并掩蔽。
- **登录**（`mcpSignIn.ts`；可取消登录需要 pi ≥ 1.1.0，#10565）。
  `startMcpProbe({ action: login | test | logout })` 打开一个短命、未注册、内存中的 pi 会话，其
  加载器只携带 `builtin:mcp` 与那一台服务器（强制启用），其 UI 上下文是探测自己的，分发 pi 的
  `/mcp login|logout <name>` 或无头 `/mcp`，然后发出 `session_shutdown`（它也会中止一次正在进行的
  登录）并处置它。帧走 `provider.login` 家族，带 `target: { kind: "mcp" }` 和 `loginId`
  `mcplogin_*`，只发给拥有的连接（`setMcpLoginPublisher(push, ownerClientKey)`）；来自任何其他连接
  的回复与取消以 `LOGIN_NOT_OWNER` 失败。Pi 公开的 `openUrl` 回调直接提供 `authUrl` 帧，仅限
  `https:` 或回环 `http:` 地址；它的重定向 prompt 变成回贴的 `prompt` 帧，而结局由 pi 捕获的通知
  推导（已登录 / 失败文本 / 已取消），绝不单靠命令 promise。pi 对浏览器或回贴的等待没有总超时，
  因此每个探测都有一个截止时间（`PROBE_DEADLINE_MS`，10 分钟；`startMcpProbe` 的可选 `deadlineMs`
  是测试接缝，宿主从不传它），到期后其结束方式与取消完全相同——`error` 帧 "Sign-in timed out."、
  `session_shutdown`（关闭 pi 的回环监听器）、会话关闭后释放锁——而宿主在回收一个客户端时取消该
  客户端拥有的探测（`cancelMcpProbesOwnedBy`），因此重载的页面或被丢弃的标签页绝不会一直占着一台
  服务器直到重启。成功登录之后，该工作区中把该服务器报告为*需要登录*的活聊天会得到
  `/mcp reconnect <name>`，而不是等到它们下一个轮次。测试连接仅限 HTTP（stdio 服务器在聊天启动它时
  被验证）并报告 *connected · N tools* 或原因；每个 `error` 帧的消息（pi 的原因或通知）都像状态
  细节一样脱敏。活聊天自己的 `/mcp login` 保留 pi 的 rpc 路径：登录地址作为一条 transcript 通知
  到达（绝不自动打开；`openUrl` 在宿主上是空操作），与 pi 的回贴对话框并排。探测的加载器强制
  `builtin:mcp`，因此在 `listMcpServers` 报告 `handledElsewhere` 的地方宿主不启动任何探测
  （`MCP_HANDLED_ELSEWHERE`，[[submodule-server-host]]）。
- **每台服务器一次登录，无论谁运行它。** pi 在共享的 `mcp-auth.json` 中为每台服务器保留一份客户端
  注册和待处理 PKCE 状态，所以针对同一台服务器的两个并发流程会互相破坏。探测从其启动到其会话关闭
  持有该服务器的锁（`acquireMcpSignInLock`，[[submodule-server-agent-mcp]]），并以 "already
  running" 拒绝（当某个聊天持有时为 "… in a chat"）；活聊天的 `/mcp login|logout <name>` 在命令
  持续期间持有同一把锁（引擎包装后的 `/mcp`），而在其他任何东西持锁时，它在聊天中说明这一点，而不是
  启动 pi 的流程。裸的 `/mcp login` / `/mcp logout` 不加锁——这是已接受的缺口：pi 在其处理器内部挑选
  服务器（唯一的 OAuth 服务器、唯一需要登录的那个，或它自己选择器的答案），因此在其流程写入状态之前
  没有名称可锁；pi 的状态文本与提示总会点明服务器（`run /mcp login <name>`）。

## 必须做对（Get right）

- 会话正在流式输出时 `prompt()` 会抛错 → `promptSession` 回退到 `steer()`。
- 错误通过事件流 + 抛出的方法到达，而不是崩溃信号——包装并转发。
- **重新打开的磁盘会话在播种之前先被修复**（`SessionManager.open` 与 `createAgentSession` 之间的
  `repairDanglingToolCalls`）——绝不在一个活 `AgentSession` 之后向会话文件追加，其内存上下文会
  失同步。
- **一个活交互式调用只在其会话绑定的阶段注册表中阻塞；重启状态仍由 transcript 推导。** 该注册表在
  处置时被遗忘且永不恢复。只有某个已启用交互式工具的可执行悬空调用会在挂接之前被修复为待处理标记；
  `length`/`error`/`aborted` 尝试是终止的。`assessRespondability` 仍是唯一的回复有效性权威；拒绝
  会让 WS 请求大声失败。
- 为会话前读取和新会话共享一个**当前的** `ModelRuntime`。每个会话作为 `createAgentSession` 的
  `modelRuntime` 接收并保留其世代；给每个会话自己的 `SessionManager`，并在移除时 `dispose()` 它。
  旧的运行时只能通过旧的活会话到达，并随它们一起变得可回收；`AgentSession.reload()` 仅限资源，绝不
  改变世代。
- **一个 `pi` `Model` 绝不能原样跨过线上契约**——provider/扩展配置可能在 `baseUrl`、headers、auth
  或 provider 闭包中携带机密。每个带模型的帧（`model.list`/`model.refresh`/`model.default`、
  `session.create` 的结果、`SessionSummary.model`）都经过 `toWireModel`——列表路径共享那唯一一次
  `readAvailableWireModels` 读取，使该投影无法因新增调用方而被绕过；每个入站模型引用
  （`session.create` / `session.setModel`）都在宿主侧按 `{provider,id}` **重新解析**
  （`resolveWireModel`），从不被信任。线上契约类型
  `WireModel = Pick<Model, id|name|provider|contextWindow|reasoning> + thinkingLevels +
  cost{input,output} + input + auth` 是一个**允许列表**——它失败即关闭，因此未来的 `Model` 字段
  默认无法泄露（一个单元测试固定了确切的键集合）。`thinkingLevels` 是计算字段：pi-ai 的
  `getSupportedThinkingLevels(model)` 在同一瓶颈点被映射，因此 effort 选择器无需客户端重新推导就能
  渲染 pi 的按模型支持真相。`cost` 只保留两个列表价（pi 的 `ModelCost` 档位与缓存费率留在宿主侧），
  而 `input` 保留模态列表。**`auth`** 是把每 provider 的连接种类投影到每个模型上（`providerAuth.ts`
  → `catalogProviderAuth`，来自 pi 的同步 `isUsingOAuth` / `getProviderAuthStatus` 加上该世代的不
  透明 Central id），以便选择器无需 `provider.status` 刷新就能说 *plan* / *API key* / *env* /
  *JetBrains AI*；`describeProviderAuth` 是唯一的种类/细节映射，而 `auth/providerStatus` 复用它，
  因此选择器与 Settings → Providers 不可能不一致。`toWireModel(model, auth?)` 把该投影作为值接收：
  目录读取按 provider 计算一次（`readAvailableWireModels`），而会话结果、摘要与 `setSessionModel`
  路径从它们自己的世代推导它（`sessionWireModel`）。
- 活斜杠命令列表由 **Pi 的 rpc 模式所用的同样三个来源**推导
  （`extensionRunner.getRegisteredCommands()` + `promptTemplates` +
  `resourceLoader.getSkills()`）。会话前目录只把 `resourceLoader.getSkills()` 经同一个 skill→命令
  助手映射，并应用**同样的项目信任门禁**，因此 New Workspace 预览与真实会话不可能不一致，除非是已
  接受的基线分支/当前检出时机差异。
- 对话框 promise 尊重 abort/超时，并在会话处置时被结算（+ 在 UI 中关闭）——一次被桥接的
  `uiContext` 调用绝不能挂起。
- **Prompt 模板的 `/name` 展开**——prompt 中键入的引用如 `/name args` 走 agent 默认的
  `expandPromptTemplates: true`（无 agent 代码改动）。它从会话的**创建时模板快照**展开——会话中途
  保存的模板**不会**被已打开会话的键入路径看到（pi 会逐字透传未知的 `/name` 文本）。输入框的 `/`
  菜单路径总是经由 `template.list` 保持新鲜（见 `templates/SPEC.md` 的新鲜度规则）。
