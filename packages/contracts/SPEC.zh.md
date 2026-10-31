# 线上契约

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

浏览器↔宿主的线上主干：该协议的唯一权威来源。包含类型，以及一个无依赖的小型运行时：WS 方法
与通道常量、协议与特性版本、少量配置默认值、收窄的跨环守卫，以及两个客户端共同驱动的退出
确认规则。它是 `apps/web` 唯一可以依赖的包——正因如此，UI 才能独立于宿主发布。

## 边界（Boundary）

- **拥有：** 线上契约——实体类型、`pi` 的事件/消息类型（再导出）、WS 方法与通道注册表，以及
  协议版本。其中包括 **`WsErrorCode`**——*宿主命名*的封闭失败集合（`WsResponse.errorCode`，
  目前为 `UNKNOWN_COMMIT`、`PUSH_AUTH_FAILED`、`NOT_GIT` 和 `ALREADY_OPEN`（两个 `project.open`
  拒绝：文件夹不是仓库，以及文件夹已被某个工作区持有）、`PROJECT_CLONE_INVALID`、
  `PROJECT_CLONE_FAILED` 和 `PROJECT_CLONE_CLEANUP_FAILED`（输入纠正、Git 失败，以及需要
  文件系统介入的不完整清理）、`RESOURCE_UNAVAILABLE` 和 `SUBAGENT_TRANSCRIPT_NOT_FOUND`
  ——最后一个是 `subagent.getTranscript` 的**永久性**未命中，也就是让转录对话框停止轮询的信号。
  若某个已知子会话的第一个转录文件尚未写入，则改为返回空消息连同其当前状态，这样仍在运行的
  一次运行依然可轮询；传输层抖动仍是普通的 `error` 瞬时错误，值得重试——此外还有评论者 UI 必须
  区分的四种 `change.*` 结果：**`STALE_VIEW`**（某个 `expect` 哈希已不再匹配宿主读到的内容 →
  重新读取 diff 并重新提供选择）、**`SCOPE_IMMUTABLE`**（该范围的修改侧是一个提交，因此工作树中
  的任何内容都不在被描述之列）、**`RANGE_INVALID`**（某个区间落在它所声明的侧之外，或对一个
  纯字节资源请求了范围回退）、**`RECEIPT_UNKNOWN`**（撤销一个宿主已不再持有的回执——该环只有
  20 条深，随进程一起消失），以及 **`UNSUPPORTED_CHANGE`**（所选变更是符号链接或仅模式变更，
  这条面向字节的写入路径刻意拒绝它）），因此客户端可以针对某个具体失败做出反应，而不必对错误
  消息做模式匹配。只有当客户端对某个失败的处理方式不同时，它才配拥有一个错误码；其他一切仍是
  普通的 `error` 字符串。预期的方法特定结果仍是有类型的方法结果，而不是通用 WS 失败；不存在
  当前布局协议。
- **公开接口（`index.ts`）：** `piProtocol` + `domain` + `nativeClient` 的 `export type *`
  （`NativeUpdateState` / `NativeUpdateBridge`，`NativeWindowState` /
  `NativeWindowControlsBridge`，`NativeQuitHint` / `NativeCommand` / `NativeShortcutsBridge`，
  以及 `NativeNotificationInput` / `NativeNotificationBridge`，还有 `NativeMenuBridge`（已翻译的
  原生菜单标签），这些可选的 shell 本地桌面能力）；
  来自 `quitConfirmation` 的 `QUIT_CONFIRMATION` + **`createQuitConfirmation`** +
  `QuitConfirmationDependencies`；值导出包括 `DEFAULT_CONFIG`、`DEFAULT_WORKSPACE_NAME`
  （宿主为项目默认工作区所用的名字；在它被重命名之前，客户端用自己的语言表述它）、
  `SYSTEM_LOCALE_PREFERENCE` + **`isLocalePreference(value)`**（`"system"` 或规范 BCP-47 标签
  ——即 `AppConfig.locale` 契约；本地语言文件形状是带封闭错误码的 `LocalLocalesSnapshot` /
  `LocalLocaleFile`，由 `locales.list` 从 `LOCALES_PROTOCOL_VERSION`（v89）开始提供；用户主题
  文件是 `UserThemesSnapshot` / `UserThemeFile`，由 `themes.list` 从
  `USER_THEMES_PROTOCOL_VERSION`（v92）开始提供，其中每个成功文件的 `manifest` 刻意保持
  `unknown`，因为主题 schema 由 [[submodule-web-themes]] 拥有）、`THEME_MODES`、`isThemeMode`、
  `isSystemThemePair`、`normalizeThemePreference`、`JBCENTRAL_QUOTA_REFRESH_SECONDS`、
  `isJbcentralQuotaRefreshSeconds`、`isJbcentralConnected`、`SESSION_RENAME_PROTOCOL_VERSION`、
  `SESSION_TITLE_MAX_LENGTH`、`normalizeSessionTitle`、`FIRST_PROMPT_MAX_LENGTH` +
  **`firstPromptHead(text)`** + **`firstPromptOfTexts(texts)`** +
  **`clampToCodePoints(text, maxUnits)`**（对一条聊天首个提示词的唯一共享读法，用于未命名聊天的
  标签：第一个既不是控制消息、又有头部的用户文本；头部是第一行非空行，空白折叠，钳制到 200 个
  UTF-16 单元且不拆分代理对；一个前导 skill 块*或*它展开而来的 `/skill:<name> <request>` 命令读作
  该请求，否则读作 `/skill:<name>`；若第一行单独是一个小写标签（例如 `<review …>` 这样的包头部）
  或没有文本，则没有头部，转而尝试下一个用户文本——仅仅以 `<Button>` 开头的提示词仍保留其头部；
  一行中只有前 800 个单元会被规范化——宿主和客户端对各自的用户文本调用同一批函数，因此标签
  永不跳动）、`LINE_WIDTH_COLUMNS` + **`isLineWidth(value)`**（同步聊天/文件折行列数的共享
  40–240 整数契约）、`MAX_HISTORY_LIMIT`、`MAX_HISTORY_QUERY_LENGTH`、`TODO_NUDGE_PREFIX` +
  **`isControlMessage(text)`**（对该标记的唯一共享读法——客户端在注水时隐藏这类发送，宿主在历史
  索引中跳过它们，且不把它们计作 `message_sent`；双方在此达成一致，而不是各自重新推导
  `startsWith`）+ **`isRetriedAttempt(messages, index)`**（对 pi 已持久化但被取代的自动重试尝试的
  唯一共享读法——客户端注水时隐藏这些轮次，宿主的历史索引器跳过它们的文本；两者都占用索引槽位，
  因此跳转锚点保持对齐），这些来自 `domain`；来自 `piProtocol` 的
  **`isTranscriptMessageRole(role)`** 和 **`isSessionWorking(state)`**（运行中或空闲但有后台工作
  ——每个徽标和一览所用的唯一工作判定）（`isTranscriptMessageRole` 是转录携带哪些角色的唯一定义：
  宿主用它过滤 `session.getMessages`，*也*用它统计 `history` 的 `messageIndex`，因此两份相差一个
  角色的定义会静默地移动之后每一个跳转锚点）；`wsProtocol` 的 `export *`（值导出）（`WS_METHODS`、
  `WS_CHANNELS`、各类型化映射、`PROTOCOL_VERSION`，以及特性引入版本）。
- **允许的依赖：** 运行时为零。**仅类型**的 devDeps：`@earendil-works/pi-ai` +
  `@earendil-works/pi-agent-core`，且**从它们的包根**导入（仅类型 → 构建时擦除）。
- **部署义务：** 线上契约描述宿主行为和兼容性，永不描述提供该行为的启动器或部署方式。一个特性
  的线上形状由浏览器、桌面端和未来客户端共享。分离的仅类型原生客户端能力描述可选的 shell 本地
  桥，而不是宿主 WS 方法：`NativeUpdateState` 和 `NativeUpdateBridge` 承载更新呈现与显式本地动作；
  `NativeWindowState` 和 `NativeWindowControlsBridge` 承载最大化/全屏快照，以及无边框原生窗口
  委托给 HTML 控件的 最小化 / 切换最大化 / 关闭 动作；`NativeQuitHint`、`NativeCommand` 和
  `NativeShortcutsBridge` 推送已确认退出的提示与转发的原生菜单命令，并为自行确认退出手势的客户端
  提供 `quit()`；`NativeNotificationInput` 和 `NativeNotificationBridge` 承载
  `show({ title, subtitle?, body, silent? })`（即发即忘的 OS 通知，无回调、无 id）以及可选的
  `isWindowFocused()`，后者报告 OS 级别窗口焦点（即使 DevTools 持有渲染器焦点也为真）。同一个
  web 包在不导入原生 SDK 的情况下发现这些能力。可选的 `HostUpdateNotice` 承载一个封闭的 CLI 宿主
  更新生命周期；受协议门禁的 `host.update` 是一个空请求，只能启动启动器预先绑定的更新器。浏览器
  永不提供命令、路径、通道、版本、URL、重启或来源权威。
- **禁止：** 任何对 `pi` 包的*值*导入；对 `@earendil-works/pi-coding-agent` 的**任何**导入（哪怕
  是 `type`）（会引入 `node:fs`）；pi-ai 的**provider / API 子路径**（`/providers/*`、`/api/*`、
  `/bedrock-provider` 等——它们会静态加载 Node provider SDK）；以及导入 `server` / `shared` /
  `web`。

## 内容（Contents）

- **piProtocol.ts**——来自 pi 包根的 `import type` 再导出（仅类型 → 构建时擦除）：
  - `@earendil-works/pi-ai`：`Model`、`Message`、`UserMessage`、`AssistantMessage`、
    `ToolResultMessage`、`TextContent`、`ThinkingContent`、`ImageContent`、`ToolCall`、
    `AssistantMessageEvent`、`Usage`、`StopReason`；
  - **`WireModel`** = `Pick<Model<string>, "id"|"name"|"provider"|"contextWindow"|"reasoning">`
    **+ 一个计算字段 `thinkingLevels`**（pi-ai 的 `getSupportedThinkingLevels`，在宿主侧由
    `toWireModel` 映射；客户端→宿主的参数携带它但不生效）**+ 三个可选的挑选器元数据投影**——
    `cost`（按 Mtok 计的 `{input, output}` 标价，是 pi 的 `ModelCost` 的子集）、`input`（可接受的
    模态）和 **`auth`**（`WireModelAuth`：provider 的 `ProviderAuthKind` 加上 pi 的 `detail` 标签
    ——与 `provider.status` 报告的词汇相同，按模型投影，使挑选器可以显示“套餐”而不必做沉重的
    status 调用、也不必给出按 token 计价）——之所以可选，是因为早于
    **`MODEL_PICKER_PROTOCOL_VERSION`**（v77）的宿主会省略它们；这就是模型**在线上**所取的形状
    （`model.list`/`model.refresh`/`model.default`、`session.create` 的结果与参数、
    `session.setModel` 参数、`SessionSummary.model`）。它是一份恰好列出 UI 所渲染内容的
    **允许列表**，而*不是* `Omit`：扩展/provider 的 `Model.baseUrl` 和 `headers` 可能携带路由
    凭据，而允许列表**默认失败即封闭**——未来新增的任何 `Model` 字段（无论保密与否）默认被排除。
    宿主从 `{provider,id}` 重新解析出真正的 `Model`——因此客户端既读不到秘密，也无法注入一个
    `baseUrl` 让 agent 去调用（见 `agent` 模块 SPEC）。**`sameModel(a, b)`** 是每个环都在用的唯一
    身份判定——`{provider, id}`；其他字段都不是身份，因为名称、上下文、价格和层级都是可能落后于
    一次刷新的目录快照；
  - `@earendil-works/pi-agent-core`：`AgentEvent`、`AgentMessage`、`ThinkingLevel`（包含 `off` 的
    那个）；
  - 本地渲染联合 **`PiEvent`**——真正的超集 `AgentSessionEvent` 位于仅 Node 的
    `pi-coding-agent` 中，所以这里对它做了**镜像**（`agent_end.willRetry` 以及 `agent_settled` /
    `queue_update` / `compaction_*` / `auto_retry_*` / `summarization_retry_*` /
    `session_info_changed` / `thinking_level_changed` / 宿主投影的 `entry_appended { entryType }`
    等成员，再加 `bash_execution_update`——纯粹为保持联合忠实度而镜像；宿主从不调用
    `executeBash`，因此 UI 永远不会收到它）。事件携带的每条消息（`message_*`、`turn_end`、
    `agent_end.messages`）都类型化为 **`WireAgentMessage`**——即 `TranscriptMessage` 加上
    `WireBranchSummary` / `WireBashExecution` 镜像——而绝不是 pi-agent-core 的 `AgentMessage`。
    那个类型是 `Message` 加上编译时恰好包含的任何 `CustomAgentMessages` 增强：pi-coding-agent 只在
    Node 构建中加入自定义角色，而 pi-agent-core 本身已不再加入它们，因此同一个 `PiEvent` 在服务端
    和在浏览器中（那里的自定义消息守卫被收窄为 `never`）否则会意味着两个不同的联合。
    `agent_settled` 是一个宿主投影，携带最后一次尝试报告的终止元数据（`stopReason` + 可选的
    `errorMessage`）：`agent_end.willRetry` 只覆盖 provider 自动重试，而在压缩或排队的续接随之而来
    时，它并不是自动工作的终点。
    `compaction_end.result` 类型化为 **`CompactionEndResult`**——pi 仅 Node 的 `CompactionResult`
    的允许列表镜像，恰好携带压缩提示所渲染的内容（`tokensBefore` + 可选的
    `estimatedTokensAfter`）；宿主构造这个投影，而不是整体强转 pi 更丰富的对象，而线上数据始终
    不可信，因此 reducer 会守卫字段形状而不是假定它们；
  - **`SessionEventPayload`**（`{ sessionId, event: PiEvent }`）——`pi.event` 推送帧。
  - 廉价胜出的镜像（声明在仅 Node 的 `pi-coding-agent` 中）：**`SessionStats`** +
    **`ContextUsage`**（token/成本/上下文条——仅供显示）以及 **`SlashCommandInfo`** +
    **`SlashCommandSourceInfo`**（命令自动补全行，由运行时的 `session.getCommands`、仅技能的会话前
    `skill.list`，以及映射后的提示词模板列表返回），还有 **`SkillCatalogEntry`** +
    **`SkillDecision`**（`load`/`untrusted`/`pending-ack`/`disabled`）——工作区 Skills 管理器的
    `skills.state` 行。
  - **`ExtensionCatalogEntry`**（`{ name, title, description?, enabled, tools,
    hasChildExtensions }`）——一个宿主注册表 ThinkRail 扩展，即 `extension.list` 报告的样子；
    `name` 是 `AppConfig.disabledExtensions` 列表所用的身份，`tools` 是其父工厂注册的名字。
  - **`SessionSummary`**——宿主为注水而报告的聊天会话（读侧）；`live` 区分内存中的会话与仅存在于
    磁盘的会话。前端在本地注水已放置的会话，其余的在聊天历史中列出以便显式重新打开。可选的
    **`openTodos`**（该聊天 TODO 计划中非 `done` 项的数量）只由 `session.list` 填充（宿主经 todos
    模块装饰），用于历史/状态呈现；缺省 = 未知，按 0 处理。运行中的摘要可选的 **`lastSettlement`**
    保留宿主观察到的终点（`null` = 该次运行处于活动状态，或已在没有助手的情况下结算），以便重连
    时能呈现 Pi 从其重建上下文中移除的最终失败；缺省意味着本宿主进程未观察到结算，持久化转录
    才是权威。可选的 **`queue`**（**`SessionQueueState`**：pi 待处理的 `steering`/`followUp` 文本
    外加 `hasImages?: true`，即宿主对排队浏览器发送的保守聚合）仅在非空时随运行中的摘要一起传输
    ——它是客户端待发条带的注水种子，因为 `queue_update` 只在变化时触发，而在运行中途接入的客户端
    否则永远不会知道在其连接之前排队的消息。同一聚合也用于丰富投影出的 `queue_update` 事件；图像
    字节永不随这份读侧队列状态传输。在规范化状态协议下，可选的 **`state`** 携带与全工作区快照/
    推送所安装的完全相同的 `SessionState`，使转录注水能够证明它渲染的是哪一次完成，而无需依赖
    注意力专用的纪元。破坏性操作使用单独的 **`SessionQueueContent`** / **`QueuedMessageContent`**
    形状，它们恰好一次性地返回每条被排空消息的文本和可选图像，使输入器能恢复完整内容，而不必让
    普通的队列广播变得沉重。`session.getMessages` 返回 `{ summary, messages }`（转录是
    **`TranscriptMessage[]`**——即 pi 规范的 `Message` 联合，用 **`WireCustomMessage`** 拓宽（它是
    pi-coding-agent 仅 Node 的 `CustomMessage` 的仅类型镜像，使扩展注入的消息如 ask 回复能跨过
    线上），并用 **`WireCompactionSummary`** 拓宽（仅 Node 的 `CompactionSummaryMessage` 的镜像：
    `summary`/`tokensBefore`/`timestamp`），即一次压缩的已解析上下文记录——pi 把它放在保留的尾部
    之前并丢弃被摘要的消息，因此转发它才使压缩边界在重载/重新打开后依然存在，而不是渲染出一段从
    对话中途开始的转录。摘要反映磁盘重新打开后的当前运行中会话）。宿主可发送的角色全集由运行时
    **`isTranscriptMessageRole(role)`** 守卫钉住：它是服务端转录过滤与历史索引的唯一来源，二者的
    对齐保证历史命中的 `messageIndex` 在客户端的 `turnIdByMessageIndex` 面前有效——若只在一侧新增
    角色而不在另一侧新增，就会静默地移动之后每一个跳转锚点。
  - **聊天标题**——`SessionSummary.title` 是 pi 持久名称的读投影，会话尚无名称时为 `""`；客户端用
    自己的语言表述该默认值（没有英文占位符跨过线上）。追加式变更 `session.rename` 接受
    `{ workspaceId, sessionId, title }`，拒绝去除空白后单行形式为空或超过
    `SESSION_TITLE_MAX_LENGTH`（80）的标题，并返回一个 ack；`SESSION_RENAME_PROTOCOL_VERSION`
    把该变更和相关控件钉在 v66，使较新的客户端在旧宿主上隐藏它们。既有的 Pi 事件
    `session_info_changed { name?: string }` 是手动与自动变更的唯一运行时域更新，而
    `session.list`/`session.getMessages` 会修复丢失的事件。标题来源/出处、唯一性、工作区耦合或新的
    推送通道都不跨过线上；会话 id 仍是规范身份。**`SessionSummary.firstPrompt`** 和
    **`SessionStateRecord.firstPrompt`**（可选、追加式、无版本门禁）在标题旁携带转录首个用户提示词
    的 `firstPromptHead`：它是没有名称的聊天的显示回退，永远不是名称——没有变更接受它，也没有任何
    东西把它写进会话，较旧的宿主干脆省略它（客户端显示其通用标签）。
  - **扩展 UI 帧** **`ExtUiRequest`** / **`ExtUiResponse`**——对 pi 进程内 `uiContext` 调用
    （`select`/`confirm`/`input`/`editor` 往返；`notify`/`setStatus`/`setWidget`/`setTitle`/
    `dismiss` 为即发即忘）的线上形状，承载于 `pi.extensionUi` 通道。往返请求可选的 `toolCallId`
    指明它所涉及的工具调用；缺省则不属于任何调用（宿主只在知道该调用时才锚定对话框：显式指定，
    或恰好只有一个调用在运行时）。显式锚点可以附加 `toolInput`——该调用将要使用的输入，如发问宿主
    所见——客户端在提示中完整列出它。两个字段都是追加式的：旧客户端忽略它们，旧宿主永不发送它们。
    待处理对话框以客户端的 `session.extUiReply` 或宿主的 `dismiss` 帧结束；宿主在 abort、dispose
    或接受该聊天的用户消息时把它关掉（一条被扩展处理或被预检拒绝的消息会使其保持待处理），因此
    客户端永远不会把自己发出的消息当作回答。
  - **交互式工具**线上形状——**`ToolResponsePendingDetails`** `{ kind: "ack" }` + 守卫
    **`isToolResponsePending`**（工具结果的 `details` 表示“人的回应仍待处理”：由重启修复写入，
    由延迟的工具返回）、常量 **`TOOL_RESPONSE_CUSTOM_TYPE`**、**`ToolResponseMessage`** /
    **`ToolResponseDetails`** `{ toolCallId, response: unknown }`（当没有运行中的工具在等待时宿主
    追加的迟到回应）以及 **`readToolResponseMessage`**（共享读取器；它也会读已退役的
    `ask-user-answers` 形状——`details.result`——使既有转录中已回答的问题保持已答；该遗留字面量
    对读取器私有）。全部位于 `wsProtocol`。这些形状**不指名任何工具**：哪些工具是交互式的由
    server 扩展描述符和 web 渲染器注册来声明（[[module-extension-api]]）；工具自身的参数/结果类型
    位于其 pi 包（`@thinkrail/pi-ask-user-question/core`），而不在此处。`WireCustomMessage.customType`
    本身保持 `string`——该命名空间是开放的（任何 pi 扩展都可以铸造自定义消息，它们全都会跨过
    线上），因此严格性落在生产者 + 守卫上，后者校验 details 的*形状*（线上数据不可信——另一个
    进程，可能是另一个协议版本）。第一个交互式工具是 ask-user-question 扩展的
    （[[module-ext-ask-user-question]]），通过宿主的邮箱应答（server `agent/toolResponses`——设计
    理由见其 SPEC）；聊天**内联**渲染卡片并通过 `session.respondToTool` 回复（以工具调用 id 关联；
    该值对线上不透明，由工具校验；当调用未知/已回应/已被取代时大声拒绝）。
    `TOOL_RESPONSE_PROTOCOL_VERSION`（93）标记从仅 ask 的 `session.answerQuestion` 以及
    `needsInput` 分支 `question` → `tool-response` 的重命名。**一个兼容窗口，双向：**请求注册表保留
    `session.answerQuestion`（`result: unknown`），转发到同一条投递，因此已加载的旧客户端仍能应答；
    而客户端向低于 v93 的宿主（`supportsToolResponses`）发送 `session.answerQuestion`，并把该宿主的
    `needsInput.kind: "question"` / `question:<toolCallId>` 读作等待中的调用，因为 UI 独立于宿主
    发布（决策 3）。窗口关闭时两者都移除。
  - **todo 计划评审修复**的线上类型——常量 **`TODO_REVIEW_FIX_CUSTOM_TYPE`**、
    **`TodoReviewFixMessage`**（tag↔details 形状）+ 其守卫 **`isTodoReviewFixMessage`**（位于
    `wsProtocol`），以及 **`ReviewFixDetails`** / **`ReviewFixComment`**（位于 `domain`）：计划评审
    裁决的修复请求以**结构化自定义消息**（customType `todo-review-fix`）而非合成的用户轮次送达
    工作者（#363）。消息 `content` 仍是 agent 所读的渲染后包文本；`details`（项 id/标题、可选说明，
    以及精简的路径/行解析发现）是聊天卡片所渲染的内容——宿主在发送时从其锚点解析每条发现的
    `path`/行，因此客户端不重新解析任何东西。见 [[submodule-server-todos]] +
    [[submodule-web-chat]]。

- **domain.ts**——应用实体：`Project`（git 仓库 + 唯一 `slug` + 可选的 **`closed: true`**——持久化的
  打开侧栏成员位；缺省表示打开以保持向后兼容，而关闭永不改变项目的 id，也不删除它的工作区关联——
  加上技能信任字段 **`trusted`**（逐项目授权）、**`acknowledgedSkills`**（re-confirm-new：哪些已提交
  别名已获认可）和 **`disabledSkills`** / **`disabledGroups`**（项目基线层面的逐技能与逐组关闭——一个
  组是一个插件、一个来源层级，或特殊的 `@plugins`），它们门禁其技能所贡献的内容；工作区在该基线
  之上叠加 **`Workspace.skillOverrides`**（逐技能开关）；
  **`SubagentOverride`**（`"on" | "off"`）+ 可选的 **`Workspace.subagentsOverride`** 让工作区强制
  开/关子代理，缺省则继承宿主的 `AppConfig.subagentsEnabled` 默认值；可选的
  **`Workspace.subagentMaxConcurrentOverride`** 同样覆盖 `AppConfig.subagentMaxConcurrent`；
  “它有没有规格？”**不是**字段——而是惰性查询 `project.hasSpecs`，因为它要遍历整棵树）、
  **`ProjectPathStatus`**（候选路径的种类——`repo` / `initable` / `missing` / `notDirectory`——使 UI
  可以打开、提供 `git init`，或显示错误）、`Workspace`（git worktree；其可选的 **`renamed`** 标志即
  命名生命周期——缺省 = **尚未命名**（新建的 `workspace-N`，或旧宿主一次临时处理留下的未加锁记录），
  因此 agent 的 `set_title` 仍可为其命名；`true` = 已命名（自动或由用户命名），之后永不再被自动
  触碰；其可选的 **`kind: "default"`** 标记内置的逐项目 **Default 工作区**——即把项目文件夹本身
  当作工作区，每个项目恰好一个，固定排在 `workspace.list` 首位，服务端不可移除也不可重命名；
  **`kind: "external"`** 标记一个显式附加、用户自有的 worktree，ThinkRail 可以忘记它，但绝不可
  重命名或回收；缺省 = 由 ThinkRail 管理的 worktree 工作区；可选字面量
  **`initialTerminalPending: true`** 是宿主持有的供给标记，仅在工作区仍需要宿主预留时携带：宿主先
  预留确定性的终端，然后清除它；缺省意味着不再有供给工作——显式线上字段，永不用 id 约定；
  **结算生命周期**由另外四个宿主持有的可选字段承载——**`lastActiveAt`**（毫秒——宿主所见的最近一次
  *真实*活动：一次聊天/agent 轮次、终端输入，或该 worktree 的 HEAD 移动；选中或查看工作区刻意
  **不算**活动）、**`settledOverride`**（`"settled"` = 用户把它搁置，`"active"` = 用户把它固定为
  活跃；宿主在真实活动出现时清除任一值，使覆盖永不失效）及其 **`settledAt`** 戳，以及
  **`review`**（宿主对该分支最近已知的 `OpenBranchReview`，在宿主侧重刷新，使每个客户端看到同一个
  PR 状态而无需各自去取）——划分出活跃行与 Settled 搁架是客户端在这些事实加会话状态之上做的
  选择器，永不是线上字段；**`WORKSPACE_SETTLE_PROTOCOL_VERSION`**（v78）把这些字段与
  **`workspace.settle`** / **`workspace.unsettle`**（`{ id }`——Default 在服务端被拒绝；响应是更新
  后的记录，`workspace.updated` 推送让每个客户端收敛）钉在一起。结算永不触碰磁盘；
  `workspace.remove` 仍是唯一的拆除方式，一次一行）、
  **`OpenBranchReview`**（活跃分支的评审引用：PR 还是 MR + 编号，外加可选的 **`state`**——
  `"open" | "merged" | "closed"`，在 v78 之前的宿主上缺省并按 open 读取——以及 **`changedAt`**，
  即 provider 合并或关闭它的毫秒时间；v88 为开放的 GitHub PR 增加仅用于查询的 **`details`**——
  `draft`、`base`（它现在合入的分支，GitHub 可能已改靶）、`checks`（通过 / 待定 / 失败计数加上有界的
  失败名称）、`reviewDecision`、批准 / 请求变更的 `reviewers`、`mergeState`（GitHub 的
  `mergeStateStatus`，小写化）、仓库的 `mergeMethods` 和 `deleteBranchOnMerge`；details 只在
  `workspace.branchStatus` 上随行（`workspace.openReview` 的回答会把它们剥掉），且永不进入宿主保留
  的 `Workspace.review` 快照——客户端只会针对同一评审把它们叠加到该快照的状态上）、
  **`BranchStatus`**（`workspace.branchStatus` 的回答，v87 `BRANCH_STATUS_PROTOCOL_VERSION`：
  `hasOrigin`、`provider`（`"github" | "gitlab" | null`）、`published`（推送仓库拥有该分支；当分离
  origin 的推送仓库无法读取时为 `null`）、`pushesElsewhere`（origin 的推送 URL 与其抓取 URL 不同，
  因此 `ahead` / `behind` 描述的是推送仓库）、已发布时相对 `origin/<branch>` 的 `ahead` / `behind`、
  `aheadOfBase`、`onBase`（该分支是自己的基线——Default 位于默认分支上）以及
  `review: OpenBranchReview | null`——顶栏 Remote 与 Pull request 段据此推导下一步动作的事实，与
  是否存在评审无关）、**`ExistingWorktreeCandidate`**（一行 `workspace.listExisting`：绝对 `path` +
  `branch`，或是选择器会禁用的 `detached` 行）、
  `FileNode`（文件树节点；worktree 相对的 `/` 路径，当 git 忽略它时由宿主设置 `gitignored`）、
  **`ResourceMeta`**（一个资源的字节*究竟是什么*，由宿主决定并由 `RESOURCE_META_PROTOCOL_VERSION`
  = v75 钉住：sha-256 `hash` + `byteLength`，资源不存在时两者均为 `null`；`text` = 合法 UTF-8、
  感知 BOM，且未被已识别的二进制魔数占用（一个 ASCII PDF 是纯字节文档；一个 SVG 是文本）；可选的
  `mime`，先按魔数字节嗅探、再按文件名嗅探。它随 `fs.readFile` 以及 `git.diffFile` 的两侧传输，
  后者的 `content` 在 `text` 为假时是 `""`，其 `originalOid` 是用于寻址不可变 `/blob` 侧的已解析
  范围起点——宿主拒绝发送解码后的二进制，因此客户端用 `/files` + `/blob` HTTP 路由渲染纯字节资源，
  并用 `hash` 作为评论锚点或回退期望所命名的身份）、
  **变更操作类型**——**`LineSpan`**（从 1 开始、含两端；`count: 0` 指明 `start` *之前*的一个插入点）、
  **`RevertTarget`**（`file` = 该路径在范围内的整体变更，或 `range` = 一个 hunk，以**两侧的行区间**
  表示，永不用补丁或 `@@` 头：两个客户端 diff 引擎切分 hunk 的方式不同，客户端绝不能规定字节）以及
  **`ChangeReceipt`**（对 `change.*` 调用的回答*兼*其撤销令牌：`kind` 为 revert/undo，`before`/
  `after` 的 hash+长度+模式身份（`null` 表示不存在），以及可选的 `trashed`——整文件移除时垃圾
  助手收到的、绝对的同目录临时占用路径；该文件因此在 OS 垃圾箱中带此临时名称）、
  `Git*`/diff 类型——包括 **`GitDiffScope`**（Changes 面板在 diff 什么：`branch` → 该工作区自偏离
  其 diff 基线以来的工作（范围从二者的 merge-base 起，绝不是基线的末端）/ `uncommitted` → 工作树
  对 `HEAD` / `commit` → 单个提交，`sha^` 对 `sha` / `turn` → 宿主围绕一次 agent 运行取的两份
  worktree **树快照**（`{ id, baseTree, headTree, startedAt }`，两侧不可变，v80）；线上缺省 =
  `branch`，因此旧客户端不变）、**`TurnChangeSet`**（一次被记录的 agent 运行：
  `{ id, workspaceId, sessionId, startedAt, settledAt, baseTree, headTree, changes }`——该次运行实际
  留下的东西，由 **`workspace.turns`** 提供并在 **`turn.changed`** 上推送，
  `TURN_CHANGES_PROTOCOL_VERSION` = v80）、
  **`GitCommit`**（范围菜单列表中一行提交）、**`BranchList`**：`remote` 仍是规范的完整 ref 字符串
  列表，而可选的宿主撰写 `remoteGroups` 携带每个已配置远端及其 `{ ref, branch }` 行，用于两层呈现。
  该字段是追加式的，因此较新的 UI 在旧宿主上回退到完整 ref；一个 `null` 远端组保留那些配置已被删除
  的跟踪 ref，而不会在浏览器中凭空猜出归属。一个工作区基线的两个含义是**两个字段**：
  `Workspace.baseBranch` 是*创建出处*（worktree 被切出的 ref——回执的 `branch · from baseBranch` 所
  显示的内容；对于**用户自有**工作区，其出处不属于 ThinkRail 可主张的范围，它就是作为*初始*评审
  目标的仓库默认值，UI 不显示 `from`），而可选的 **`Workspace.diffBase`** 是*评审目标*
  （`workspace.setDiffBase`）；每次读取都在**服务端、在同一处**解析 `diffBase ?? baseBranch`——
  把它们合并成一个字段会让重新指向的目标谎报分支的来处；**`ProviderStatus`/`ProviderStatusReport`**
  ——Welcome 条带所渲染的 auth-provider 状态行（逐 provider 的 `configured` + auth `kind`：oauth /
  api-key / env / central / other——绝不是凭据值；`central` 标记一个内置 provider，其注册已被
  JetBrains AI（Central）扩展替换，仅由注册身份推导，绝不来自 Central 的配置；另加
  `canOAuth`/`canApiKey`/`canLogout`，它们门禁条带内的应用内 Sign-in / Sign-out 入口——`canLogout`
  仅对可移除的 auth.json 凭据为真，对宿主无法取消的 env / runtime / models.json 认证为假）；
  **应用内登录线上契约**——**`LoginFrame`**（流式流程更新：`authUrl` / `deviceCode` / `select` /
  `prompt` / `progress` / `success` / `error`，它们在客户端**累积**，绝不是凭据值）、
  **`LoginPush`**（`provider.login` 帧，`{ loginId, providerId, frame }`）和 **`LoginReply`**
  （`{ loginId, value }`——浏览器对 `select`/`prompt` 的回答）；JetBrains AI 线上契约（协议 v43）——
  **`JbcentralStatus`**，嵌套在 `ProviderStatusReport` 上，是宿主撰写的封闭生命周期：`absent`、
  `outdated`、`supported`、`configured`、`malformed-version`、`probe-failed`、`configuring` 或
  `load-failed`。认证作为 **`supported`/`configured` 上的 `signedOut` 标志**随行，而 configured 状态
  还携带封闭的 **`proxyStopped`** 观察；两者都不是独立状态，因为凭据、代理进程健康与配置是相互
  独立的轴。两个标志都是*正面观察到的否定事实*——不可用或读不到的探测报告 `false`，因此客户端
  永不渲染宿主并未证实的恢复要求。代理端口、PID、URL、状态文本或诊断都不跨过线上。只有可解析的
  安全版本、封闭的探测/失败原因，以及当前动作会出现在相关处。`configuring` 同时覆盖一次经过审查的
  CLI 动作和针对最新监视工件状态的合并候选重建；`configured` 意味着**新工作的当前运行时**应用了
  该工件。`load-failed.configured` 说明最近观察到的全局状态是否请求了 Central，使客户端可以在不
  接收工件路径的情况下提供封闭的 Retry/Disconnect 动作。历史运行中的会话可能保留较旧的运行时，被
  刻意排除在此状态之外。**`JbcentralInstall`** 携带宿主按 OS 的 `{platform,shell,command}` 官方安装
  计划。**`JbcentralActionResult`** 是封闭的 `applied` / `failed` 联合；失败原因区分安装、版本探测/
  支持、Central 动作、工件后置条件，以及封闭的运行时加载失败，且不携带消息。没有 pending、restart、
  blocked-session、recovery、migration、compensation 或 reattachment 结果。原始 stdout/stderr、
  生成的扩展内容或路径、代理 URL/秘密、诊断、受影响会话 id，以及原始 PI 模型在结构上缺失；server
  与 web 各自把码映射为自己的通用文案）。协议 v59 增加 **`JbcentralQuotaSnapshot`**，即独立的封闭
  配额读取：`hidden`、`available`、`stale` 或 `unavailable`；只有 available/stale 携带有限的周期性
  `remaining` / `total` 数字和一个观察时间戳。不存在账户/套餐/已用/充值/补充字段或原始失败文本；
  **主题/配置选择**——**`ThemeId`** 在线上是开放字符串，因为宿主持久化不透明选择，而独立发布的 web
  客户端拥有可用清单目录；**`ThemeMode`** 是封闭的 `"fixed" | "system"` 行为，**`SystemThemePair`**
  携带一个不透明亮色 id 加一个不透明暗色 id。`AppConfig.theme` 在两种模式下都保持为可逆的固定选择；
  `AppConfig.systemThemePair` 在系统模式首次配置之前缺省，之后在回到固定模式后依然保留。生效外观
  永不跨过线上：每个系统模式客户端都用自己的操作系统配色方案解析同一对值。
  `THEME_SYSTEM_PROTOCOL_VERSION` 把它钉在 v58。这些字段有意设计为追加式：把 `theme` 替换成嵌套
  对象会破坏旧客户端（或造出两个固定主题权威），而把一对值编码进不透明 id 则会把一个简单 id 变成
  未经校验的迷你协议。较新的 web 客户端用该特性引入常量在旧宿主上隐藏这些控件；
  **`ComposerGrowthLimit`**（`"compact" | "roomy" | "half-chat"`）是封闭的、服务端同步的输入器高度
  偏好：分别为 6 个视觉行、10 个视觉行，或已挂载聊天面板的 50%；`"half-chat"` 是默认值，而把这类
  语义 id 翻译成几何量由 web 负责；**`LINE_WIDTH_COLUMNS`** 拥有独立受界的聊天/文件视觉折行列数
  共享 `{ min: 40, max: 240, default: 120 }` 整数契约；**`isLineWidth`** 是线两侧共同消费的唯一
  校验器；**`SUBAGENT_SETTINGS_PROTOCOL_VERSION`** 把全局/工作区控件钉在它们 v57 的线上引入版本，
  使较新的 web 客户端在旧宿主上隐藏它们而不必与不断移动的最新协议比较；
  **`JBCENTRAL_QUOTA_PROTOCOL_VERSION`** 同样钉住 v59 的配额读取 + 设置；
  **`WINDOWS_SHELL_SETTINGS_PROTOCOL_VERSION`** 钉住 v62 的 Windows shell 设置，使较新的 web 客户端
  在只能保留而无法应用该配置字段的宿主上隐藏它；**`PLAN_REVIEW_SUBAGENT_PROTOCOL_VERSION`** 钉住
  v67 的评审重塑——评审者聊天已消失，因此 `TodoPlan.reviewerSessionId` 和
  `ReviewComment.reflection` 离开线上，`todo.startReview` 返回裸 ack，其分离的失败出现在追加式的
  `review.failed` 推送（`ReviewFailedPayload`）上，因为评审没有聊天来承载它。旧客户端把被移除的
  字段读作缺省，因此这个钉是让客户端能区分“该宿主没有评审者聊天”与“该宿主更旧”的依据，而不是去
  推断；**`AppConfig`**（`{ theme, themeMode, systemThemePair?, analyticsEnabled,
  analyticsConsentConfirmed, terminalReplayKb, terminalWindowsShell, composerGrowthLimit,
  chatLineWidth, fileLineWidth, chatLineWidthBounded, fileLineWidthBounded, customLayoutPresets,
  defaultModel?, defaultEffort?, reviewModel?, reviewEffort?, favoriteModels, recentModels,
  reviewAutoFix, agentReviewEnabled, subagentsEnabled, subagentMaxConcurrent,
  jbcentralQuotaEnabled, jbcentralQuotaRefreshSeconds, disabledExtensions, onboardingDismissed }`
  ——一个可扩展的包；折行列数字段在协议 v61 加入线上，`terminalWindowsShell` 在 v62 加入。
  `onboardingDismissed`（v94，默认 `false`）记录用户已完成或跳过了冷启动引导向导，因此它不再出现。
  **`DEFAULT_MODEL_PROTOCOL_VERSION`** 钉住 v72 的 AppConfig `defaultModel`/`defaultEffort` 与宿主侧
  默认解析；Settings 控件在旧宿主上被隐藏。`defaultModel` 是完整的允许列表 `WireModel`，
  `defaultEffort` 是可选的 `ThinkingLevel`，而 `settings.update` 接受 `null` 来清除任一可选值。
  **`MODEL_PICKER_PROTOCOL_VERSION`**（v77）钉住挑选器由宿主保留的列表：**`favoriteModels`**（按
  用户显示顺序排列的完整 `WireModel`——客户端通过 `settings.update` 写入整个列表，身份由
  `sameModel` 判定）和 **`recentModels`**（最新优先，最多 **`RECENT_MODELS_LIMIT`** = 5，**由宿主
  维护**：它被排除在 `AppConfigUpdate` 之外，宿主在每次显式选择模型时追加——指明模型的
  `session.create` 以及每次 `session.setModel`）。两者默认 `[]`，因此客户端只按版本门禁，绝不按
  字段存在性。任一列表中的快照在渲染前会被客户端重新指向运行中的目录；离开目录的模型干脆不提供。
  `terminalWindowsShell`（`"auto" | "pwsh" | "powershell" | "cmd"`，默认 `"auto"`）只在 Windows 上
  由 `server/terminal` 读取，其他地方忽略——每个值启动什么见
  `submodule-server-terminal` 的 shell 选择决策。**`PROJECT_TEMPLATE_PREVIEW_PROTOCOL_VERSION`** 钉住
  v63 追加式的、按项目定位的 `template.list` / `template.get` 读取，使 Create Workspace 能预览全局
  模板加当前检出中的项目模板而不发送宿主路径；旧宿主保留仅全局的回退。`themeMode` 默认
  `"fixed"` 且无配对，既保留旧配置也保留显式的 Dark 默认；`subagentsEnabled` 是宿主级子代理默认值
  （当前行为下为 `true`），仅被 `Workspace.subagentsOverride` 覆盖；`subagentMaxConcurrent`
  （**`SUBAGENT_MAX_CONCURRENT`** = 1–16 区间内的整数，默认 4，由 `isSubagentMaxConcurrent` 校验；
  在线上自 **`SUBAGENT_CONCURRENCY_PROTOCOL_VERSION`** = v79 起）是一个聊天同时运行多少
  子代理——更多的派生请求排队——仅被 `Workspace.subagentMaxConcurrentOverride` 覆盖；**`settleIdleDays`**（默认 `7`，
  `null` = 永不；在线上自 `WORKSPACE_SETTLE_PROTOCOL_VERSION` = v78 起）是空闲窗口，超过后安静的
  工作区结算进其项目搁架——这是宿主级数字，因为“多久算安静”是一种习惯而非项目事实；
  `agentReviewEnabled`（默认 `false`，在线上自 `AGENT_REVIEW_SETTING_PROTOCOL_VERSION` = v68 起）
  门禁工作者的会话内 `request_review` 工具，并对打开的会话即时生效——Review 按钮是独立的（见
  [[submodule-server-host-plan-review]]）；`disabledExtensions`（默认 `[]`，在线上自
  `EXTENSION_CATALOG_PROTOCOL_VERSION` = v82 起）指名用户关掉的宿主注册表扩展——未知名字被保留并
  忽略，宿主把它应用到新会话与打开的会话（见 [[submodule-server-extensions]]）；
  `customLayoutPresets` 是有界的、不含资源的目录，且是宿主同步的**唯一**布局值；当前/默认预设与
  分组上限是 web 本地的）；`analyticsEnabled` 是额外数据偏好与宿主门禁，默认 `false`，而
  `analyticsConsentConfirmed` 默认 `false`，记录初始选择的完成。
  `ANALYTICS_CONSENT_PROTOCOL_VERSION` 钉住这个 v65 契约，使较新的客户端不会在无法持久化它的旧宿主
  上显示同意流程。初始对话框先单独写入启用偏好，然后一起写入完成状态与当前偏好；Privacy Settings
  则两者一起写入。旧客户端仅写偏好的操作无法造出新的确认。安装 id 完全留在服务端；基础事件不受
  两个标志控制，见 [[submodule-server-analytics]]），它与 **`DEFAULT_CONFIG`** 回退一起承载该配置
  （宿主侧持久化为 `config.json`，在 `server.welcome` 中投递，经 `settings.update` 变更）。每个新的
  用户聊天都显式收到解析后的模型与 effort，因此会话与 `model.default` 一致；显式请求字段各自胜出，
  而仅含 effort 的请求使用已配置或回退的模型。计划评审子代理保留自己的模型策略。
  **`InterviewResponse`** 是从自动反馈弹窗接受的封闭 `"book" | "postpone" | "never"` 动作。没有使用
  次数、资格、忽略状态或客户端身份跨过线上。Contracts 有意不导出任何主题目录枚举/列表/标签：未来
  的清单可能铸造一个构宿主时未知的 id，而缺少它的客户端会解析到外观相同的随包回退；
  **`SpecGraphNode`/`SpecGraphSnapshot`**——Specs 查看器的读 DTO，**镜像**自（与 `PiEvent` 一样）、
  永不从 `pi-spec-graph` 导入——线上只携带面板所渲染的内容（`type`/`status` 保持 `string`：容忍
  磁盘上的任何东西）；**`TodoItem`/`TodoGroupItem`/`TodoPlan`/`TodoArtifact`** + 联合
  **`TodoStatus`/`TodoOrigin`/`TodoArtifactKind`**——聊天内计划 DTO，**镜像**自 `pi-todos/core`
  （永不导入），携带该聊天逐会话的 TODO 列表。`TodoGroupItem` 还携带
  **`status: TodoGroupStatus`**——该组的*任务*生命周期（`pending`/`active`/`done`），**由宿主**从
  步骤推导（`pi-todos` 的 `groupStatus`）而非存储：把它发出来意味着真值表只有一个家，也没有客户端
  需要重新推导。一个 `commit` 工件还携带 **`files?: GitFileChange[]`**（路径 + 状态 + `+/−`——与
  Changes 面板在提交范围下渲染的相同行）——由宿主经 `todo.list` 的装饰从 git 推导（同样的单一归属
  理由），永不存储；缺省 = 该 sha 已无法解析，静默降级。`TodoItem.summary` / `TodoPlan.summary` 是
  agent 的完成备注（按步骤 / 整个计划，按存储原样），而 `TodoItem.verification` 是单独的自我报告
  检查行（确切命令 + 结果，或“未验证”——客户端把它渲染成带标签的徽标，标明这是 agent 自己的主张，
  永不是宿主门禁）。`TodoItem.commitSubject` 是第三个被存储的完成时字段：宿主在该标题下提交该项的
  增量（`title` 是面板看到的计划步骤，这一行才是落到仓库历史里的内容——见
  [[submodule-server-todos]]）。它在线上只是因为 DTO 镜像了存储的项，而不是因为今天有客户端渲染它；
  **`TodoItem.review?: TodoReviewInfo`**（+ 联合 **`TodoReviewState`**）是宿主推导的评审装饰，只出现
  在可评审的项上（即拥有宿主变更集的项）：`state`
  （`unreviewed`/`reviewed`/`changes_requested`——`unreviewed` = 无存储记录）、`revision`（提交计数
  ——1 个 TODO = N 个提交）、`unreviewedShas`（自用户水位线以来的提交——“自评审以来已变更”的增量）、
  `feedback` + `at`。评审状态存在于宿主 sidecar，永不在 agent 可写的计划里；见
  [[submodule-server-todos]]。**`TodoPlan.unattributed?: GitFileChange[]`** 是同一 `todo.list` 装饰
  发出的宿主推导余量，仅在非空时出现：工作树中不属于该计划任何项的未提交行——否则在评审图上不可见
  的那些变更（推导与理由：[[submodule-server-todos]]）。
  **`TodoPlan.adoptedCommits?: TodoItem[]`** 是已提交的对应物，由同一装饰发出、仅在非空时出现：
  不属于任何项的 `base..HEAD` 提交，以**仅线上存在的 `done` 项**呈现（`origin: "adopted"`、
  `id: "commit:<sha>"`、一个 `commit` 工件），使一个没有规划就提交的聊天依然能显示——并能评审——
  那部分工作。它们永不存储；`TodoOrigin` 的 **`"adopted"`** 成员只存在于线上（推导：
  [[submodule-server-todos]]）。
  **`DelegationRunDetails`** + 联合 **`DelegationRunStatus`**——子代理 Agent 卡片 DTO，**镜像**自
  `pi-delegation`（永不导入）：随 `tool_execution_update.partialResult`（REPLACE）、最终的 `Agent`
  工具结果，以及 `subagent-completion` 自定义消息传输；子转录本身经 `subagent.getTranscript` 读取，
  以 `(workspaceId, parentSessionId, childSessionId)` 为键——其结果还携带该次运行当前的注册表
  `status`（宿主不再认识该运行后缺省），即客户端的“运行期间轮询”信号。完成消息的 tag + 配对位于
  `wsProtocol`（承载值的那一半），完全照搬 tool-response 的姿态：常量
  **`SUBAGENT_COMPLETION_CUSTOM_TYPE`**（镜像 `pi-subagents` 的 `SUBAGENT_COMPLETION_MESSAGE`，
  永不导入——又是 DTO 姿态）、**`SubagentCompletionMessage`**（编译期固定的 tag↔details 形状，其
  details 为 **`SubagentCompletionDetails`**：运行详情加上镜像自 `pi-subagents` 的可选
  **`CompletionDelivery`** 标记）以及共享守卫 **`isSubagentCompletionMessage`**——线上数据不可信，
  因此 details 要经 **`isDelegationRunDetails`**（domain）校验：**封闭的状态联合**、每个必需的
  **数值用量字段**、`durationMs`、每个出现的可选显示字段为字符串，以及出现的 `thinkingLevel` 为 pi
  封闭层级之一，绝不只验“存在一个对象”（PR #303 评审发现）。该校验器是形状检查的唯一归属处——web
  的 Agent 卡片读取器也通过它收窄——外加 **`customMessageText`**——对 `WireCustomMessage.content`
  （字符串 | 块）的唯一文本提取，由 web 的事件 reducer 与注水共享，使后台结果行的报告文本只推导
  一次。
  **历史搜索读 DTO**——**`HistoryScope`**（覆盖层的循环：本聊天 → 工作区 → 项目 → 全部）；
  **`PromptHit`**（一条被召回的提示词；携带可选的 `messageIndex` + `anchorText`——保留的最新一次出现
  的跳转锚点）和 **`MessageHit`**（全文对话命中；仅助手——用户角色的命中只会与其自身 `PromptHit`
  的文本重复，因此跳转入口放在那里；`messageIndex` 把跳到消息锚定到 `session.getMessages` 的顺序，
  `anchorText` 使锚点能容忍漂移），以及 **`HistorySearchResult`**（提示词 + 全文消息两段，带总数
  和索引状态）；**评审 DTO**——**`Review`**（每个工作区一个打开的评审；`baseSha`——被评审 diff 的
  原始侧（分支范围的岔点，也就是该 diff 实际显示的内容——绝不是目标的末端，后者可能带上评审从未
  显示的上游提交），在**创建时**钉为完整提交 oid，在评审生命期内不可变——加上 **`fileSessions`**，
  键 → 该键的评审聊天：每个文件一个聊天，在首次发送时固定，**空键**是无锚点的整体变更集桶，以同样
  方式固定，使第二条整体评论延续同一讨论——以及 **`doneFiles`**，同样的键：用户标记评审完成的文件，
  使完全解决的文件只在用户明确表示时才离开列表；可选的 **`guide`**——一个 **`ReviewGuide`**：最新的
  计划步骤裁决的 `summary`、其有序 `readingOrder`（`ReviewReadingStep { path, why }`）、裁决与出处，
  由宿主撰写，使 Changes 评审标签能走读评审者的叙事）、**`ReviewComment`**（`kind` 为
  inline/diff/file/review；`status` 为 draft/sent/resolved/dismissed——与 **`anchorState`**
  anchored/moved/outdated 正交；逐评论的 `sessionId`——它被发送进的聊天）、**`ReviewAnchor`**
  （`path` + `side` + `contentHash` + 有序的 **`ReviewSelector`** 回退链：`lineRange` / `textQuote` /
  `diffHunk` / `structural` / `region`——`structural` 是给拥有稳定身份（`json-pointer`、`ipynb-cell`、
  `table-cell`、`md-heading`，外加渲染器铸造的任何方案：未知方案原样跨过线上，只校验形状）的文档
  模型准备的按方案分型槽位，而 `region` 是规范化的 `0..1` 几何，带可选的分页媒体 1 起始 `page`，
  因此评论可以指名一个图像区域或一个 notebook 单元而不是一行；Ask agent 的 hunk 评论用精确显示的
  hunk 头填充 `diffHunk`，而 `contentHash` 是对该资源**字节**的 sha-256（对合法 UTF-8 与原本的文本
  哈希逐字节相同）。**`REVIEW_RICH_ANCHORS_PROTOCOL_VERSION`** 把追加式的 `region` 成员与如今被填充
  的 `structural` 槽位钉在 v74，使渲染器丰富的客户端能区分出会保留它们的宿主与早于它们的宿主，
  而不是让一个区域锚点被静默丢弃；`side: "base"` 的锚点还携带 **`baseRef`**，即其行与片段被捕获时
  所依据的 ref（因为两侧 diff 是两个行空间），外加上它被捕获时所在的 **`scope`**——即重新打开渲染
  该 blob 的那一个界面的 diff 身份）、**`ReviewSnapshot`**（`{ review, comments }`——`review.get`
  的读取结果，并连同 `workspaceId` 构成 `review.changed` 推送载荷 **`ReviewChangedPayload`**——
  全快照，因此重放是幂等的）；
  **提示词模板 DTO**——**`TemplateScope`**（`"global"` | `"project"`——模板所在处）、
  **`TemplateInfo`**（仅元数据：名称、可选的 `description`/`argumentHint`、`scope`、`filePath`
  ——`template.list` 返回的内容；刻意不含正文，使一次列表永不必传输每个文件的全文）以及
  **`Template`**（`TemplateInfo` + 完整 `content`——frontmatter + 正文——即按名称的
  `template.get`/`template.save` 形状）；
  **布局预设 DTO**——可移植的 **`LayoutPreset`**，即 `AppConfig.customLayoutPresets` 中同步的有界、
  不含资源的框架语法：中心拓扑、左/右/底分组几何、可见性/折叠、底部对齐，以及单例工具（`plan`，
  即 TODO List，自 `LAYOUT_PLAN_PRESET_PROTOCOL_VERSION` = v94 起被接受），但不含工作区、文件、diff、
  聊天、文档、终端、预览、注意力或当前/默认选择身份。每个当前布局类型——包括投影出的
  `WorkspaceLayoutDocument`、`WorkbenchFrame` 和 `WorkspaceViewState`——都是 web 本地的，刻意不在
  contracts 中。不存在当前布局方法或推送通道。
- **nativeClient.ts**——宿主线上契约之外的仅类型可选原生客户端能力。桌面端的
  更新桥暴露一个单调状态快照、检查/下载/安装等提示动作、失败操作身份，以及状态订阅，而不暴露 feed
  选择。可用、字节传输、准备、就绪和安装中是彼此不同的状态。安装期间，快照携带一个粗粒度的安装步骤
  和一段有界的结构化事件日志（只有 `diagnostic` 携带原始 SDK 文本）。重启之后，它以数据形式携带持久
  的最后安装结果，而绝不是显示文本：结果、from/to 版本、原生失败步骤、原始原因，以及保存的日志。
  只有 `dismissInstallResult()` 会清除它。窗口控件桥为窗口状态暴露同样的
  快照/动作/订阅形状。快捷键桥推送退出提示变化（`hidden` / `armed` / `release` / `quitting`）和
  `NativeCommand` id（目前是 `close-item`，关闭当前聚焦的应用内条目，以及 `open-settings`，在没有
  模态打开时打开 Settings），其唯一动作 `quit()` 请求宿主执行一次普通的协同退出。
  通知桥暴露 `show({ title, subtitle?, body, silent? })`（即发即忘的 OS 通知，无回调、无 id）以及
  可选的 `isWindowFocused()`，后者报告 OS 级别窗口焦点——即使 DevTools 持有渲染器焦点也为真，这与
  `document.hasFocus()` 不同。浏览器连接没有这些桥，也不渲染它们的任何入口。
- **quitConfirmation.ts**——退出确认手势的唯一归属处，由桌面主进程（macOS Cmd+Q，按键状态经
  CoreGraphics 轮询）和 web shell（Linux Ctrl+Q，按键状态来自 keydown/keyup）共享。
  `QUIT_CONFIRMATION` 持有 1200 ms 长按、500 ms 双击窗口（从首次释放开始计）和 40 ms 轮询。
  `createQuitConfirmation` 是运行在注入的 `readHeld` / `canShowHint` / `now` / `every` / `onHint` /
  `quit` 之上的状态机：`press(held)`、`sync()`（立刻重读 held）、`cancel()`（已确认的手势则退出，
  未确认的则隐藏提示）、幂等的 `quitNow()` 和 `reset()`（回到空闲、提示隐藏；用于退出请求失败的
  客户端）。只有在组合键被释放后才会触发退出。它放在这里而不是 `shared`，因为 `apps/web` 只能导入
  contracts，而手势的两份副本会漂移；它不依赖框架，也不触碰任何全局，因此仍在不含运行时依赖的
  规则之内。每个客户端只保留一个薄薄的按键状态适配器。
- **`HostUpdateNotice`**——可选的宿主线上契约 CLI 生命周期：当前版本、可用的较新版本、通道，以及
  可选的封闭 `available | running | succeeded | failed` 状态（缺省意味着 v64 的旧式通告）。
  `host.updateAvailable` 发布完整替换。协议 v70 增加无参数的 `host.update`；没有输出、任意诊断、
  feed URL、工件、平台路径或 shell 命令跨过线上。
- **wsProtocol.ts**——`WS_METHODS`（`project.*`——包括 **`project.close`**（把稳定记录标记为已关闭而
  不删除关联状态）、**`project.rename`**（`{ id, name }` → 更新后的 `Project`；只改显示用 `name`
  ——路径、slug 和 id 保持不变——并广播完整的 `project.updated`；`PROJECT_RENAME_PROTOCOL_VERSION`
  把它 v85 的引入钉住，使较新的客户端在旧宿主上省略该动作）、**`project.inspect`**（分类一个路径）
  + **`project.init`**（`git init` + 提交，然后打开）+ **`project.create`**（`{ parentPath, name,
  displayName? }` → 已打开的 `Project`：在 `parentPath` 下创建文件夹 `name`（缺失的父级一并创建），
  然后执行与 init 相同的引导流程，并在 `displayName`（用户所输入的名字）与文件夹名不同时用其为项目
  命名；`ONBOARDING_PROTOCOL_VERSION` 把它 v94 的引入连同 `AppConfig.onboardingDismissed` 和
  `provider.detect` 一起钉住）。**`PROJECT_CLONE_PROTOCOL_VERSION`**（v95）门禁
  **`project.cloneDestination`**（`{ parentPath, name }` → `{ path }`，经宿主校验后解析出的绝对
  目标）、**`project.clone`**（`{ operationId, url, parentPath, name, depth? }` → 已打开的
  `Project`，或在取消并清理后为 `null`）和 **`project.cancelClone`**（`{ operationId }` → ack）。
  操作 id 作用于请求它的客户端；取消一个不存在的操作是无害的。Clone 会重新校验目标，因此预览永不
  预留它。重连会重放原始请求；宿主 30 分钟的截止时间短于客户端 31 分钟的等待。
  + **`project.hasSpecs`**（逐项目惰性的“是否包含已注册的规格？”，供 Welcome 界面使用——它需要遍历
  整棵树，因此只对被展示的项目请求，绝不急切地为每个项目请求）/ `workspace.*`——特别是
  **`workspace.list { projectId, includeDiffStats? }`**，其中省略或为 true 保留现有的带计算聚合的
  完整行，`false` 返回同样的权威成员关系/顺序但不做导航恢复从不使用的同步逐工作区 diff 统计
  扇出 / `fs.*`（**`fs.readFile`** 回答 `{ content, meta: ResourceMeta }`）/ `git.*` /
  **`spec.graph`**（Specs 查看器的整图读取，逐工作区）/ **`todo.*`**——**`list`**/**`add`**/
  **`update`**/**`remove`**，即该聊天逐会话的 TODO 计划（以 `workspaceId` + `sessionId` 为键；
  `add` 把项标记为 `origin:"user"`），外加评审操作 **`review`**（批准：记录 `reviewed` + sha 水位
  线）、**`requestFix`**（记录 `changes_requested` + 反馈，然后宿主把修复包发进该项自己的聊天——
  分离执行，若在轮次前被拒绝则回滚）和 **`startReview`** / **`reviewAll`**（AGENT 评审：一个隐藏的
  评审子代理拿到该项的包并返回结构化裁决；发现以 `author: "agent"` 的评审评论形式到达；
  `TodoItem.review` 在裁决待定期间携带 `reviewing`，在 agent 批准时携带 `reviewedBy`。`reviewAll`
  报告它启动的数量，并在计划的串行链仍然繁忙时额外报告 `alreadyRunning`）/ **`terminal.*`**——
  **`reserve`**（幂等地建立一个宿主目录标签页而不启动其 PTY；`INITIAL_TERMINAL_TAB_KEY` 命名那个
  每个前端都可被动放置的宿主种子标签页）/ **`attach`**（以 `(workspaceId, tabKey)` 为键的幂等
  get-or-create，返回 `created` + 用于重绘的 `replay`；这是 PTY 诞生的唯一方式，它取代了
  `create`+`alive`）/ **`list`**（宿主拥有标签页列表）/ `write` / `resize` /
  **`close`**（按 `tabKey`，除非 `force` 否则拒绝繁忙的 shell）/ `model.list` + **`model.refresh`**
  （等待宿主的单飞目录刷新并返回 **`RefreshedModels`**——刷新后的列表加上 **`complete`**，表示该次
  处理是否在宿主的有界等待内结算，因为只有已结算的列表才是权威的；`force` 绕过 pi 的 4 小时新鲜度
  节流，使由用户发起的刷新真正去抓取）/
  **`model.default`**（宿主解析出的模型与兼容 effort：可用时为 AppConfig 的默认模型，否则为宿主
  已结算可用列表中的第一个模型，再否则为 `null`；effort 是 `defaultEffort ?? "medium"`，当存在模型
  时用 Pi 的 `clampThinkingLevel` 钳制）/
  **`model.contextSettings`** / **`model.setContextWindow`**（v76；两者都返回
  **`ModelContextSetting[]`**——合格 OpenAI 模型的 `provider`/`id`/`name`、pi 的生效 `contextWindow`，
  以及 models.json 中的显式 `override` 或表示 pi 默认值的 `null`。该变更接受
  `{target: "available" | {provider,id}, contextWindow: number | null}`：`null` 移除覆盖；数字必须
  满足 **`isModelContextWindow`** 且落在 **`MODEL_CONTEXT_WINDOW_LIMITS`**（272,000–1,000,000）之内，
  这是应用的编辑策略而非经验证的 provider 容量，两侧都会强制执行。此范围之外的外部值会被读取而
  不钳制，而 `available` 目标只覆盖 **`isSharedModelContextTarget`** 接受的设置（无覆盖，或覆盖在
  范围内）：应用不可能写出的覆盖只能通过指名其 `{provider,id}` 来更改。保存会发出既有的
  `provider.changed` 失效通知；没有 AppConfig 状态或原始配置跨过线上）/
  **`model.clampThinking`**（对一个 `{model, level}` 对使用 pi 的 `clampThinkingLevel`——会话前
  挑选器的 effort 调整，使没有客户端需要重新推导 pi 的策略）/ **`provider.status`**
  （auth-provider 状态报告；每次读取都在宿主侧重新校验）/ **`provider.detect`**（v94：
  **`DetectedAccount`** 列表——其他本地工具已登录的账户，每项指名其自身登录所建议的 pi provider 及
  可选的套餐标签；只是提示，绝不是凭据）/
  **`provider.*` 应用内登录**
  （**`loginStart`**——铸造一个 `loginId` 并**分离地**运行 pi 的登录流程（`type` 为 `"oauth"` |
  `"api_key"`，issue #97——两条认证路径共用一条通道；一次流程可能耗时数分钟，绝不能压在请求上，
  也不能阻塞 WS 泵）/ **`loginReply`**——回答运行中的 `select`/`prompt`，以 `loginId` 关联 /
  **`loginCancel`** / **`logout`** /
  **JetBrains AI** 系列 **`jbcentralConnect`** / **`jbcentralDisconnect`** /
  **`jbcentralStartProxy`** / **`jbcentralUpdate`** / **`jbcentralLogin`**（返回
  `JbcentralActionResult` 的原生全局 Central 动作；它们都不接受来自客户端的可执行文件、工件路径、
  输出、URL 或秘密）/ **`jbcentralQuota`**（v59；可选的 `force` 绕过已完成缓存的老化但依然加入
  进行中的读取；禁用/不健康时返回 `hidden` 且不调用配额））/
  **`workspace.listExisting`**（所选项目中未附加的 Git worktree，分离行按状态禁用）/
  **`workspace.openExisting`**（重新校验并把一个由分支支撑的检出注册为 `kind: "external"`，发出
  普通的 `workspace.created`，而不改动 Git 或磁盘）/
  **`workspace.rename`**（`{ id, name }` → 已加锁的更新后 `Workspace`；仅限受管理的 worktree；
  更改显示名而保留 Git 分支 + cwd，并广播普通的全快照 `workspace.updated`；
  `WORKSPACE_RENAME_PROTOCOL_VERSION` 把它 v55 的引入钉住，使带有该动作的较新客户端在旧宿主上省略
  它）/ **`workspace.openReview`**（活跃分支的可选 `OpenBranchReview` 元数据；可选的
  `allowCached: true` 允许命中的已结算宿主缓存，而省略则保留原本的强制新鲜行为，以照顾较旧的独立
  发布客户端；两种形式都仍会加入已在进行的查询）/ **`workspace.branchStatus`**（v87，同样的参数与
  新鲜度规则 → `BranchStatus`；对 v87 客户端取代 `openReview`，后者为更旧的客户端保留；v88 增加
  `skipFetch: true`——一次 CI 轮询：做一次新鲜的评审查询但不抓取 origin）/
  **`workspace.push`**（v87，`{ workspaceId, branch, confirmBase? }` → `{ dirtyFiles }`：把分支发布
  或推送到 `origin`，不碰任何 PR；以自身为基线的分支会被拒绝，除非 `confirmBase: true`，而
  `branch`——用户看到并确认的那一个——必须仍是当前分支；`PUSH_AUTH_FAILED` 与 `pr.open` 相同）/
  **`pr.preview`**（v87 增加可选的 `source: "plan" | "commits" | "template" | "empty"` 和
  `template`，使 `sessionId` 变为可选，并返回发现到的 `templates` 和 `planAvailable`；旧客户端的
  `{ sessionId }` 调用保留计划正文——`host` 显式给它发送 `source: "plan"`，因为 v87 的缺省可能
  是 `commits`）/ **`pr.generate`**（v87，`{ workspaceId, sessionId?, template? }` →
  `{ body: string | null }`，一次尽力而为的单次草稿；没有模型作答时为 `null`；当调用方提供
  `body` 时，v87 的 `pr.open` 不接受 `sessionId`，并接受可选的 `branch`——草稿为其撰写的那一个——
  它必须仍是当前分支，与 `workspace.push` 相同）/ v88 的**评审动作**
  （`REVIEW_ACTIONS_PROTOCOL_VERSION`），每个都指名用户看到的 `branch`，对 PR 还指名它显示的
  `number`：**`pr.ready`** / **`pr.reopen`**（`{ workspaceId, branch, number }` → `Ack`）、
  **`pr.merge`**（`… , method: "squash" | "merge" | "rebase", deleteBranch` →
  `MergePrResult { merged, branchDeleted, deleteError? }`——当 GitHub 只是排队了它（合并队列 / 自动
  合并）时 `merged: false`，此时分支永不删除；已落地但分支删除失败的合并算作成功并带上原因），以及
  **`workspace.deleteRemoteBranch`**（`{ workspaceId, branch }` → `Ack`；`PUSH_AUTH_FAILED` 与推送
  相同）/
  **`project.setTrust`**（持久化一个项目的信任授权 → 更新后的 `Project`；门禁其已提交的跨 agent 技能
  别名；`resources: true` 还会授予 pi 级别的资源信任，见 Project trust）/
  **`skill.list`**（面向某个 `projectId` 的会话前、仅技能 `SlashCommandInfo[]` 预览，从该项目当前
  检出解析，其**项目作用域别名受信任门禁**；最终的 worktree 会话才是权威）/
  **Skills 管理器**系列——**`project.aliasSkills`**（列出已提交别名名字，供存在性门禁提示的计数）/
  **`project.acknowledgeSkills`**（确认信任之后出现的技能）/ **`project.setSkillEnabled`**（项目
  基线）/ **`project.setGroupEnabled`**（在基线上开/关一个插件 / 来源层级 / `@plugins`）/
  **`workspace.setSkillOverride`**（逐工作区 开/关/清除 → `Workspace`）/
  **`workspace.setSubagentsOverride`**（`"on"` / `"off"` / `null` 表示继承 → 更新后的 `Workspace`）/
  **`workspace.setSubagentMaxConcurrent`**（一个 `SUBAGENT_MAX_CONCURRENT` 数字 / `null` 表示继承 →
  更新后的 `Workspace`，v79）/ **`workspace.setDiffBase`**（重指向 diff 目标，`null` 清除回创建基线
  ——回显更新后的 `Workspace` **并**广播 `workspace.updated`，使每个客户端在推送后收敛）/
  **`workspace.watchReady`**（在加载技能的客户端捕获其新鲜度基线之前，等待新监视器的保守启动微调；
  除非监视器已被认为就绪，否则 `{ startupNudge }` 为真，因此重放的响应可以在事件推送丢失或启动失败
  时为客户端提供保守回退；可选的 `prewarm: true` 把已启动的监视器标记为仅预热——宿主把它们放在全局
  有界、可淘汰的池中，任何真实的预检/读取都会把它们提升出池，见 server `watch` SPEC）/
  **`git.status`** + **`git.diffFile`**（两侧随 `ResourceMeta` 传输，其 `originalOid` 冻结该范围的
  原始侧供 `/blob` 使用），两者都接受可选的 **`scope: GitDiffScope`**（无法解析的范围——例如被
  rebase 移除的提交——会被*拒绝*，面板把它读作“重置范围”，而不是卡在一个死掉的 sha 上）/
  **`git.listCommits`**（该工作区分支自身的提交，`<diff base>..HEAD`，最新优先，在宿主侧有上限——
  范围菜单的惰性抓取列表）/ **`workspace.turns`**（`{ workspaceId }` → 该工作区已记录的 agent 运行，
  最旧优先，在宿主侧有上限）/ **`change.*` 写入路径**（`CHANGE_MUTATIONS_PROTOCOL_VERSION` = v75）
  ——**`change.revert`**（`{ workspaceId, path, scope, target, expect: { originalHash, modifiedHash } }`
  → `{ receipt }`：客户端指名*它所看到的*，宿主在逐工作区锁下从自己的读取中重新推导变更，任一侧哈希
  不匹配即 `STALE_VIEW` 且不写入任何东西——agent 在评审期间仍在工作，因此比较并交换就是全部的
  保护；符号链接与仅模式输入为 `UNSUPPORTED_CHANGE`）和 **`change.undo`**（`{ workspaceId,
  receiptId, expect: { modifiedHash } }` → `{ receipt }`：回执就是逆操作，这也是 UI 对每个 hunk 提供
  *Undo* 而非每次确认的原因；它回应的 `undo` 回执本身还可撤销一次，因此重做不需要第三个方法）。
  回执只存在于宿主内存中（逐工作区：最多 20 个，最多保留 64 MiB 字节，最旧的先被淘汰，最新的始终
  保留；随工作区一起丢弃）：git 和 OS 垃圾箱已经提供了恢复能力，因此数据目录格式毫无收益——所以是
  `RECEIPT_UNKNOWN` 而不是对持久性的线上承诺 / **`git.prefetch`**（对远端基线的一次尽力而为的后台
  抓取——New-Workspace 对话框的新鲜度预热；始终 ack `{ ok }`，当抓取确实移动了本地远端跟踪 ref 时，
  宿主随后向以该 ref 为 diff 基线的工作区发送无路径的 `workspace.fsChanged` 帧，使其由 git 推导的
  读取重新收敛）/ **`skills.state`**（对某个 `workspaceId` 返回 `SkillCatalogEntry[]`——完整目录 +
  逐技能 `decision` + `group`）/ **`extension.list`**（`ExtensionCatalogEntry[]`，无参数——宿主注册表
  与 `disabledExtensions` 合并；v82，**`EXTENSION_CATALOG_PROTOCOL_VERSION`**；没有写入方法——
  Settings 开关写 `settings.update { disabledExtensions }`）/ **`project.skills`**（同样的内容，按项目
  作用域，供会话前的管理器使用）/ **`session.reloadResources`**（为一个运行中的会话重新扫描技能 +
  重建系统提示词；流式期间被拒绝）/
  `session.*`——`create`/`prompt`/`steer`/`followUp`/**`clearQueue`**（排空 Pi 的 steering+followUp
  队列，返回完整的 `SessionQueueContent`；Pi 自身会发出表示清空的 `queue_update`；可选的
  `requireTextOnly` 在宿主观察到排队图像时不排空而直接拒绝，这是手动压缩的前置条件）/
  **`removeQueued`**（`{ kind, index }` → `RemovedQueuedMessage`：连同完整内容丢弃或取出**一条**
  排队消息——即待发条带的编辑/移除；按位置寻址是因为 Pi 的队列条目是没有 id 的裸字符串，而宿主在
  Pi 全有或全无的 `clearQueue` 之上模拟逐项移除，见 server agent SPEC）/ **`abort`**（普通 abort 为
  Interrupt 保留排队通道；`{ restoreQueue: true }` 在发出 abort 信号前原子地排空完整内容，并在会话
  到达空闲后返回它，这是 Stop 的无损路径）/`dispose`/**`delete`**/`setModel`/
  `setThinkingLevel`/`compact`/`getStats`/`getCommands`/`extUiReply`/**`respondToTool`**（对交互式
  工具调用的内联回复，以工具调用 id 关联）/**`list`**/**`getMessages`**（读侧）/
  **`settings.update`**（合并 + 校验 + 持久化顶层部分 `AppConfig`；拒绝非对象载荷或未知键，并丢弃
  已退役的键；存在时，`customLayoutPresets` 与 `systemThemePair` 是完全替换；进入系统模式需要完整
  的既有或来件配对。不带显式 `themeMode` 的旧式 `{ theme }` 变更意味着固定主题选择，并退出系统模式；
  返回合并后的配置）/
  **`environment.get`** / **`environment.update`**（`HOST_ENVIRONMENT_PROTOCOL_VERSION`，v91；被存储
  的宿主环境——可选的 pi 配置文件夹加上有序的 `{ name, value }` 变量——位于 `AppConfig` 之外，永不
  广播、也不在 `server.welcome` 中，因为值可能是秘密；`get` → `HostEnvironmentStatus`，带
  `restartRequired`、`launchOverridden` 名字和 `effectiveAgentDir`；`update` 替换整条记录，在无效时
  按字段/行索引返回 `{ ok: false, issues }` 而不保存任何东西）/
  **`feedback.respond`**（`{ action: InterviewResponse }` → ack；持久化自动邀请的 book、postpone 或
  永久忽略结果；Settings 里的链接永不调用它；`FEEDBACK_INTERVIEW_PROTOCOL_VERSION` 钉住所寻址通道
  v56 的引入，使宿主不会声称一个省略了当前 `?protocol=` 套接字能力的、独立发布的旧客户端）/
  **`client.reportError`**（`ClientErrorReport { kind: render | window | rejection, type, stack?,
  boundary?, chunkIds? }` → ack；`chunkIds` 把栈 URL 映射到 PostHog source-map chunk id；一个即发
  即忘的 UI 错误，用于宿主错误跟踪——载荷是原始且不可信的，宿主是唯一的清洗者；
  `CLIENT_ERROR_REPORT_PROTOCOL_VERSION` 把它 v90 的引入钉住，使较新的客户端永不把它发给旧宿主）/
  **`history.search`**（提示词召回 + 对话搜索的读取；结果有上限、按新近度排序；消息段仅助手——用户
  角色的命中改为呈现为可跳转的 `PromptHit`，绝不是单独的 `MessageHit`）/ **`review.*`** 系列——
  **`get`**（打开的评审 + 评论，惰性创建；读取时重新锚定）/ **`commentAdd`**/**`commentUpdate`**/
  **`commentDelete`**（撰写 + 手动解决/忽略；删除仅限 DRAFT——已发送的评论是一条记录，而已解决是
  终态：线上没有重新打开，也没有工作树回滚；`commentAdd` 接受 diff 标签的 `scope`，这正是让宿主能
  解析基线侧锚点 `baseRef` 的依据——并被持久化到锚点上）/ **`sendComment`**（一条评论 → 其*文件*的
  评审聊天，在该文件首次发送时创建，之后 `followUp`）/ **`sendBatch`**（全部/选中的草稿，按键分组
  进入每个键的聊天；回答**每一个**它触及的会话，按组顺序——只指名第一个会让其他聊天不可见，而它们
  的评论却已读作已发送）——两者都携带 **`ReviewSendResult`**：`session.create` 的形状加上
  **`reused`**，即只有宿主知道的那一个事实（这个聊天是被 followUp 进去的，还是刚创建的？）。被复用
  的聊天可能是客户端从未见过的——另一个客户端，或本客户端在重载之后，因为评审状态和 pi 转录都比
  宿主活得更久——因此它必须被注水，而不是当作新会话打开；当作新会话打开会为已标记为发送的评论显示
  一段空白对话 / **`fileDone`**（把完全解决的文件评审标记为完成；尚有未解决项时拒绝——一条新评论会
  重新打开该文件）/ **`close`**（原子 Clear：归档当前评审的非草稿记录、丢弃草稿、替换活跃评审，并向
  每个客户端发布新的打开快照）——外加
  **`template.*`**——提示词模板 CRUD（**`template.list`**、**`template.get`**——读取接受一个可选的
  位置参数，`workspaceId` 或 `projectId`，拒绝同时给两个；不给位置则仅全局，而任一定位形式都把项目
  合并到全局之上；**`template.save`**、**`template.delete`**）——全部读写 pi 的提示词目录（全局 +
  项目），因此模板保持 CLI 可移植，
  `WS_CHANNELS`（`server.welcome`——携带初始 `config: AppConfig` 以及 **`projects`**（打开的记录）和
  **`recentProjects`**（所有已知记录，打开 + 关闭，减去那些文件夹确实已消失的关闭记录），外加可选的
  **`hostUpdate: HostUpdateNotice`** 和 **`hostPlatform`**（`darwin | linux | win32`，对旧宿主可选）
  ——即*宿主*运行所在的 OS，使提供宿主执行命令的客户端（PR 设置对话框）能选对命令而不是从浏览器
  猜测 / **`host.updateAvailable`**——向后兼容的完整 CLI 更新快照：启动器发现较新版本时首先发布，
  随后随 v70 宿主侧运行在 running/succeeded/failed 之间推进而被替换；发现失败/无更新时缺省 /
  **`project.updated`**——打开/重新打开/关闭之后完整的持久化 `Project` 快照，包括 `closed` 成员位，
  使每个客户端原子地收敛其侧栏 + Recents，而无需乐观移除 / `pi.event` / `pi.extensionUi` /
  **`session.created`**（初始 `SessionSummary`，在新的宿主拥有会话注册时广播，使其他前端能在历史中
  列出它而无需打开本地放置）/ **`session.deleted`**（工作区 + 会话 id；一个不可重放的域事件，在永久
  删除后广播，使每个客户端移除该聊天并阻止过期的注水）/ **`settings.changed`**（完整的
  `AppConfig`，包括自定义预设定义，广播以使每个客户端收敛）/ **`feedback.interview`**（一个空的、
  有寻址的邀请，只发给宿主所声明的那个前端；不广播、不订阅、不重放）/ **`provider.login`**——
  无会话的应用内登录流（每帧一个 `LoginPush`，以 `loginId` 为键；它是 `pi.extensionUi` 的兄弟，
  因为登录在任何会话存在之前就运行在 Welcome 界面上）/ **`provider.changed`**——一次无数据的失效
  广播，发生在被监视的 Central 状态/重建结果改变了宿主权威的 provider 状态或当前模型代，或某次
  provider 登录（不是 MCP 登录）成功之后；客户端重读 `provider.status` 并使 `model.list` 失效，因此
  没有原始 provider/模型数据随推送传输 /
  `terminal.data` + **`terminal.exit`** + **`terminal.detached`**（有寻址的终端通道——发给唯一
  *已附加*的客户端而非广播，使一个 shell 的字节永不抵达另一个浏览器；当宿主不得不丢弃保留的输出时
  `terminal.data` 可携带 `truncated`，`terminal.detached` 表示另一个客户端接管了该标签页）/ **工作区
  生命周期三元组**——**`workspace.created`** / **`workspace.updated`** / **`workspace.removed`**
  ——注册表成员关系变化扇出到每个客户端，使其保持为共享域状态（架构 #9），全部由服务端的
  `workspaces` 发布器发出（绝不是逐客户端的乐观变更）。`created`/`updated` 携带**完整的持久化
  `Workspace` 快照**（在传输层的末值重放下幂等，因此例如一次 agent 重命名与一次手动重命名竞争时
  按 `id` 合并——绝不是增量）；`removed` 携带**`WorkspaceRemoved`** id 对（`{ projectId, id }`
  ——记录已消失）/ **`review.changed`**——某工作区的评审状态发生变化（由服务端的 `reviews` 发布器
  在每次变更时发出——UI 编辑、agent 的 `resolve_comment` 调用、重新锚定——使所有客户端收敛，与三元
  组同一模式）/ **`turn.changed`**——宿主刚为一次已结算 agent 运行记录的 `TurnChangeSet`（由服务端的
  `turns` 发布器发出；每个客户端把它追加到该工作区的列表）/ **`workspace.fsChanged`**——worktree
  变更通知器推送（**`WorkspaceFsChangedPayload`**：`{ workspaceId, paths, truncated, skillChange }`，
  去重后的 worktree 相对路径，有上限——`truncated` 意味着通用路径列表不完整，必须当作通配处理；
  `skillChange: "none" | "detected" | "unknown"` 是一个**独立的语义事实**，在该上限之前累积，因此
  一次具体的非技能溢出仍为 `none`，一个在上限之后被省略的技能路径仍为 `detected`，只有无路径的
  平台/启动不确定性才是 `unknown`；一个无路径、非截断且为 `none` 的帧是整工作区失效，例如仓库元数据
  漂移）；它是**失效提示，而不是数据**：客户端通过既有读取方法重读，因此重复/重放的帧是无害的。
  `WsMethodMap` 类型化请求/结果映射 +

  `WsParams`/`WsResult` 辅助类型，以及 `PROTOCOL_VERSION`。一个私有类型检查在两个方向上把
  `WS_METHODS` 与 `WsMethodMap` 绑定，因此某个方法只在一侧存在就无法编译。请求 id 同时也是重连
  幂等键：未解析的客户端会重放同一帧/同一 id，宿主返回 `(clientKey, requestId)` 的唯一缓存结果，
  而不是再次执行处理函数。两个*非*请求的客户端→宿主帧闭合该循环（因此有以键区分的
  **`WsClientMessage`**）：**`WsAck`**（`{ ack: string[] }`）指名客户端已*读过*的响应——这是区分
  “页面收到的回复”与“死在套接字缓冲里的回复”的唯一依据，因而也是让宿主释放保留结果的唯一依据
  ——而 **`WsResume`**（`{ resume: string[] }`）在每次（重）连接时先于重放发送，指名页面仍认为
  未解析的完整集合，使宿主可以释放其他一切。`resume` 之所以存在，是因为回执的可靠性只取决于承载它
  的套接字，而丢失的回执永远不会被重发：重述当前活跃集合比确认那些确认更可靠。这种行为是受协议
  版本约束的——重放中的 UI 绝不能对着一个去重之前的宿主运行。

## 聊天资源（Chat Resources）

当前聊天的资源视图是两个既有能力所有者之上的投影，而不是通用进程 API。
`CHAT_RESOURCES_PROTOCOL_VERSION`（71）门禁其方法与入口；v70 及更早的宿主绝不能收到这些请求。这里
不隐含任何原生 Pi 后台任务协议。命令 DTO 镜像 [[module-pi-background-commands]] 而不导入该包。子代理
保留其 Pi 子会话 id 和委派状态。

- `session.resources({workspaceId, sessionId})` 返回分开的命令与子代理摘要：所有当前活跃记录加上
  有界的近期终态记录。快照回显工作区与会话身份。命令镜像所有者的有界快照；子代理摘要只携带
  子/父 id、角色、任务、状态、创建时间、子会话实际的 `model`（`provider/id`）与生效
  `thinkingLevel`（二者可选——旧宿主缺省）、可选的中止原因、后台运行的 `detached: true`（来自子会话
  的 `info`；前台运行和旧宿主缺省），以及一旦到达终态——[[module-pi-delegation]] 测得的运行时长
  （`durationMs`），而不是用量或报告。子代理角色/任务摘要上限为 200/2,000 字符，投影中按创建顺序
  保留最近二十个终态子会话。这些是运行时目录；历史工具 ack 与持久化的 PID 在重启后都不提供活跃
  权威。
- `backgroundCommand.output({workspaceId, sessionId, commandId})` 在校验其父会话后，返回当前有界的
  `{available: true, command, output: {text, truncated}}`，或对未知、外来或已淘汰的命令返回
  `{available: false}`。输出替代先前的快照；截断是显式的。它不接受任何文件系统路径或 PID。
- `backgroundCommand.stop({workspaceId, sessionId, commandId})` 和
  `subagent.stop({workspaceId, parentSessionId, childSessionId})` 请求取消一个自有资源。已知的终态
  资源是幂等成功；未知、已过期或外来的 id 共用 `RESOURCE_UNAVAILABLE` 错误码。不可用的父会话使用
  同一错误码。成功的 `Ack` 承认的是意图，而不是观察到的终止。
- `subagent.stopAll({workspaceId, parentSessionId})` 捕获当前活跃的直接子会话，在返回
  `Ack & {targeted: number}` 之前请求取消每一个，且永不等待子会话结算。单个子代理 stop 遵循同样的
  仅意图确认规则。两个操作都不停止父会话、不移除子记录，也不禁用未来的启动；生命周期失效通知报告
  最终结算。
- `session.resourcesChanged` 是成员关系或生命周期变化时带作用域的 `{workspaceId, sessionId}` 失效
  通知。客户端重读快照；在启动 ack 之后，输出字节不会随广播或转录工具更新传输。既有的
  `subagent.getTranscript` 保持不变。

`DelegationRunDetails.abortReason` 镜像 [[module-pi-delegation]] 拥有的可选字符串。没有该字段的历史
详情仍然有效；非字符串值会被其守卫拒绝。用户 Stop 使用 `"user"`，使可移植的完成所有者可以在没有
线上拥有的取消注册表的情况下抑制空闲父会话唤醒。

`background-command-completion` 是一条被显示的 Pi 自定义消息，而不是目录更新。其受守卫的线上详情
包含不含完整命令的终态快照元数据，外加所有者有界的纯文本诊断摘录与截断标志，以及与子代理完成相同
的可选 `delivery` 标记（`wake` | `run` | `appended`——结果如何抵达 agent；在它之前写入的转录中缺省，
那些转录沿用旧的轮次规则）。守卫不校验它；共享的 `completionDelivery(details)` 读取器返回该标记或
`undefined`，因此未知值在任何地方都读作缺省。对话可以渲染那条历史通告，但绝不能从中推断当前资源
权威；日志和控件仍然使用带作用域的读取。

每次读取/控制都校验工作区成员关系与父系传承，而不只是语法上合法的 id。父与子会话 id 使用 Pi 的
规范文法，包括内部点号；工作区、命令与转录权威仍来自其所属注册表，而不是某个不透明 id 的拼写。
启动能力仍面向 agent；不存在浏览器 shell 执行方法。重连会使旧的控制权威失效，直到注水完成；瞬时
读取失败绝不能变成一个成功的空快照。不支持的宿主清除早先的资源投影。近期命令输出是短暂的；完成
消息摘录和子转录使用它们各自既有的 Pi 转录持久化，而不是第二个由线上拥有的历史存储。

## 项目信任（Project trust）

`PROJECT_TRUST_PROTOCOL_VERSION`（81）门禁泛化后的项目信任接口：信任摘要与 pi 级资源授权。与旧宿主
通信的客户端两者都不发送。

- `project.trustSummary { projectId }` 为泛化信任提示返回 `ProjectTrustSummary`：项目检出中的别名
  技能名字，以及 pi 级授权会加载的它自己的 pi 资源种类（`piResources: ProjectResourceKind[]`，没有
  时为空），使该提示恰好指名存在哪些东西，并只在 `extensions` 或 `settings` 位于其中时声明它会运行
  代码，另外还有多少个 `.pi/mcp.json` 条目随后会等待批准（`mcpServers`）。旧宿主只回答
  `project.aliasSkills`，客户端把它读作“仅有别名”。它读取项目自己的检出，不接受 `workspaceId`：
  信任是逐项目的，而展示它的两个界面（Welcome 提示和 New Workspace）在它们所导向的工作区存在之前
  就会运行；Skills 对话框在客户端侧额外加上其 worktree 目录中的不受信别名。
- `project.setTrust` 接受可选的 `resources: true`：此时授权还会信任项目自己的 pi 资源
  （`Project.piResourceTrust`——settings 和 extensions 会运行代码）。不带它时，授权只设置别名信任，
  因此一个没有说明 pi 级信任会加载什么内容的界面（旧客户端）永远不会启用它；`trusted: false` 撤销
  两者。

## MCP 服务器（MCP servers）

`MCP_PROTOCOL_VERSION`（83，在扩展目录的 82 之后）门禁 `mcp.*` 方法、`mcp.status` 通道以及可选的
`SessionResources.mcpServers` 列表；与旧宿主通信的客户端一个都不发送，并把缺失的列表当作“没有
MCP”。引擎是 pi 内建的 MCP（[[submodule-server-agent]]）；这些 DTO 镜像宿主从 pi 以及两个
`mcp.json` 文件读到的东西，绝不是 pi 自己的类型——web 只依赖 contracts。

- 项目记录携带宿主在聊天加载其服务器时应用的逐项目策略：`Project.mcpApprovals` 把仓库定义的服务器名
  映射到其整个 `.pi/mcp.json` 条目的指纹（该条目只有在该指纹仍成立时才会加载，因此一次编辑会使批准
  失效），`Project.mcpOverrides` 持有项目对用户级服务器的启用与暴露状态，永不写入 `.pi/mcp.json`；
  `Project.mcpToolGrants`（`McpToolGrant`：“始终允许”的 pi 工具名，加上它们被授予时的命令 + 参数或
  URL 的 `target` 哈希）同样是仅记录项；`ProjectTrustSummary.mcpServers` 统计仓库的条目数（见
  Project trust）。该计数是可选的，因为早于 MCP 的项目信任宿主会省略它；客户端把缺省解释为零。
- 每个请求都指名其目标：配置用 `workspaceId`（项目作用域 = 该工作区的 `.pi/mcp.json`；`user` = 全局
  文件），会话级动作（`reconnect`、`setSessionOverride`、`readOutput`）再加 `sessionId`。宿主没有
  “活跃工作区”的概念。
- `mcp.list` 返回已配置的 `McpServerSummary` 集合（作用域、定义文件、定义文件条目按渲染后的
  `fingerprint`、脱敏端点、已配置与生效暴露、启用状态、项目覆盖、仓库批准状态、pi 等价配置错误——
  每个具名条目一条，非对象条目也会被列出一条）。`fingerprint` 是该行在任作用域下被重写或移除时的
  身份；`approval.fingerprint` 是被跟踪批准的仓库内容——在项目行上是同一个值，在用户行上是被折叠
  的 `.pi/mcp.json` 覆盖的值——也就是 `mcp.approve` 所接受的东西。追加式的可选 `effective` 标志
  标识宿主加载器实际接纳的定义，与所显示的覆盖诊断无关；登录/登出/测试动作只属于那一行。旧宿主
  可能省略它，此时客户端绝不可为同名歧义行推断权威。与已配置集合一起，响应还携带逐会话的运行时
  `McpStatusSnapshot`；快照携带逐会话单调的 `generation`，使过期的推送永不覆盖更新的推送。一个
  运行中的聊天从打开那一刻起就有快照：在 pi 回答之前，以及每次重载之后，它启动的服务器读作
  `starting`。`configErrors`（`McpConfigFileError`：文件路径与 pi 等价消息，已脱敏）列出整份文件的
  问题——无法解析的 JSON、`mcpServers` 不是对象、顶层键拼写错误——这些是任何服务器行都无法承载的；
  旧宿主省略它。`handledElsewhere` 标记宿主：其中用户安装的扩展或 `-builtin:mcp` 拥有 MCP，管理是
  只读的；即使没有聊天打开也能读取 pi 设置（`by: "pi settings (-builtin:mcp)"`——Settings 识别这个
  确切值以说明该移除哪个条目，因此它保持稳定），替代扩展则通过一个运行中的聊天被看到（`by` = 其
  路径）。
- `MCP_TOOL_GRANTS_PROTOCOL_VERSION`（84）增加可选的 `McpServerSummary.alwaysAllowed`
  （`{ tool, label }`：对生效条目仍然有效的授权，只列在该行上）以及
  `mcp.revokeToolGrant { workspaceId, name, tool }`，一个像 `setProjectOverride` 一样的仅记录变更。
  旧宿主永不列出授权，因此客户端不需要门禁来隐藏撤销。
- 变更（`add`、`update`、`remove`、`setProjectOverride`、`approve`、`shareWithRepo`、
  `revokeToolGrant`）在与写入相同的同步步骤中重新检查所有权与当前项目策略，并在 `mcp.list` 会设置
  `handledElsewhere` 的地方以 `MCP_HANDLED_ELSEWHERE` 失败，使过期客户端无法在另一个管理器拥有 MCP
  时写入；它们返回新列表而不等待打开的聊天重载；`mcp.status` 推送会报告每次重载，因此返回的快照
  可能早于它。`setProjectOverride` 写项目记录，永不写被跟踪的 `.pi/mcp.json`；`shareWithRepo` 才是
  向它的显式写入。`setProjectOverride` 把它携带的字段原子地合并进最新的已存覆盖（客户端只发送变化
  的部分，因此并发客户端不会互相覆盖设置）。启用状态必须是布尔值，暴露必须是 `deferred`、`direct`、
  `hidden` 之一；畸形的运行时值以 `MCP_CONFIG_INVALID` 失败。`setSessionOverride` 是聊天资源的
  “在本聊天中禁用”：它只作用于 `mcp.json` 服务器——一个 `registered: true` 的
  `SessionResources.mcpServers` 行是某扩展注册的服务器（其 `transport` 属于该注册），禁用它以
  `MCP_CONFIG_INVALID` 失败；在 MCP 由别处处理的地方以 `MCP_HANDLED_ELSEWHERE` 失败。
  `mcp.reconnect` 在 pi 重新连上服务器后解析，并以 pi 的失败（已脱敏）拒绝——pi 对此的提示永不出现
  在聊天中——同样以 `MCP_HANDLED_ELSEWHERE` 失败。
- 写入仓库（`approve`、`shareWithRepo`，以及项目作用域的 `add` / `update` / `remove`）需要项目的
  pi 级信任，没有则失败为 `MCP_CONFIG_INVALID`，因为不受信项目的仓库条目根本不会被列出；
  `setProjectOverride` 仅记录，仍然可用。
- 一次写入永不批准用户未审查过的仓库内容。项目作用域下对新名字的 `add` 是用户自己的条目，按所写
  内容批准。项目作用域的 `update` 只有在磁盘上的条目已在其当前指纹下被批准时，才把批准带到所写
  条目（否则重写仍处于待批准状态）。每一次 `update` 或 `remove`，无论用户作用域还是项目作用域，
  都以其目标为 `McpRenderedTarget`：客户端渲染该行所用的 `McpServerSummary.fingerprint` 是必需的
  （`expectedFingerprint`），而当该作用域文件中的条目已不再拥有它时，宿主会拒绝且不动文件——因此
  过期的 Edit 或 Remove 永不重写或删除用户未见过内容，无论是一次检出改变了仓库条目，还是另一个
  客户端或编辑器在打开的对话框下替换了用户条目。该字段在线上是必需而非可选的：MCP 接口在
  `MCP_PROTOCOL_VERSION` 整体发布，因此没有需要容忍的更旧宿主或客户端。宿主自己校验 `scope`；
  除 `user` / `project` 之外的任何值都会在选择任何路径之前被拒绝。当项目文件已定义该名字，或持有
  一个未在其当前指纹下被批准的覆盖时，`shareWithRepo` 会拒绝。
- `mcp.login` / `mcp.testConnection` 在与启动探测相同的同步步骤中检查 MCP 所有权，与变更一致；它们
  返回一个 `loginId` 并复用 `provider.login` 帧族（`LoginPush.target` 标识该服务器）；这些推送只
  抵达启动该操作的连接，来自另一连接的回复或取消以 `LOGIN_NOT_OWNER` 失败。它们和 `mcp.logout`
  在设置了 `handledElsewhere` 的地方以 `MCP_HANDLED_ELSEWHERE` 失败，而当该服务器的登录正在任何
  地方运行（另一个 Settings 操作，或某聊天的 `/mcp login <name>`）时以“已在运行”错误失败。超过
  10 分钟仍打开的操作，或其客户端在重连窗口之后被宿主回收的操作，以 `error` 帧结束（“Sign-in
  timed out.” / “Sign-in cancelled.”）。`mcp.testConnection` 判断的是实际被接纳配置的传输，而不是
  加载器已跳过的、虽被批准但无效的项目行。如果在异步所有权解析期间工作区身份、cwd 或项目信任发生
  变化，变更与探测都会拒绝；没有任何过期的项目授权能接纳新操作。
- `McpToolDetails` 镜像 pi 对 `mcp__*` 及三个 MCP 资源工具的结果详情；`McpResultSummary` 是宿主
  撰写的、有界的、不含 base64 的呈现摘要，位于 `details.thinkrail` 之下，使卡片在实时、重载后以及在
  另一个客户端上渲染得完全一致。`mcp.readOutput` 只提供某结果已记录的 `fullOutputPath`，当临时文件
  消失时报告 `expired`。
- `mcp.readLog { workspaceId, name }` 返回 `McpServerLog`：该服务器最新的行（最多 200 行，最旧
  优先），来自 pi 共享的 `mcp.log` 及其轮转副本——即服务器发给 pi 的 `notifications/message` 日志
  消息，按每个条目的 `[<server>]` 头部归属——像 `detail` 一样脱敏；当服务器在那里没有记录任何内容时
  `text` 为空。它不需要会话：pi 每个 agent 目录保留一份日志。
- 状态（`McpServerState`）是宿主快照与客户端推导行共用的一套词汇：`starting`、`connected`、
  `needs-sign-in`、`failed`、`disconnected`、`disabled`、`disabled-in-project`、`disabled-in-chat`、
  `pending-approval`、`invalid-config`、`pending-reload`、`not-running`、`handled-elsewhere`、
  `unknown`；宿主永不发送 `not-running` / `handled-elsewhere`。`detail` 是 pi 对该服务器的状态文本，
  `configError` 是 pi 等价的校验消息；pi 对两者都不做编校（连接错误或 stdio 服务器的 stderr 尾部
  会原样抵达），因此宿主在它们抵达任何客户端之前，对两者——以及 MCP 登录的 `error` 帧——都遮蔽
  看起来像凭据的值。
- 错误码：`MCP_CONFIG_INVALID`（pi 等价校验失败，或上述某个写入前置条件不成立）、
  `MCP_PATH_UNSAFE`（符号链接或非普通文件的项目配置文件路径）、`MCP_HANDLED_ELSEWHERE`、
  `LOGIN_NOT_OWNER`。

## 规范化会话状态（Normalized session state）

`SessionState` 携带相互正交的宿主事实：`execution`（`idle | running`）加稳定的 `runId`、可为空的
`needsInput`（稳定的 `interactionId`、`tool-response | dialog`；等待中的工具调用携带其
`toolCallId`，待处理对话框携带其确切的可重放请求）、可为空的最近一次 `completion`（稳定 id 与显式
`succeeded | failed(error|length) | interrupted | cancelled` 结果）、Pi 的队列计数、运行中的
`backgroundWork` 标志（该会话仍拥有正在运行的后台命令或子代理），以及所有者全局的
`completionUnread` 回执投影（空闲会话仍有后台工作时保持为假）。`isSessionWorking` 是唯一的*工作*
判定：`execution` 为运行中或 `backgroundWork`；旧宿主省略该标志，读作假。`SessionStateRecord` 增加
会话/工作区/项目归属；`SessionSummary.state` 携带所附加会话的确切状态。

`session.stateList` 是全工作区每个顶层运行中或磁盘会话的权威快照，失败而不是返回不完整的扫描。
`session.state` 在引发它的 Pi 事件之后推送完整记录。`session.acknowledgeCompletion` 对一个确切的
完成 id 做比较并设置；需要输入永不可确认。`session.nudge` 对需要输入原子地空操作，在运行中的工作
期间排队，并在空闲会话上提示。这些协同方法自 `SESSION_STATE_PROTOCOL_VERSION` 起；旧宿主干脆不提供
跨工作区状态。

`workspace.markUnread` / `workspace.markRead`（`{ id }`，v86，
**`WORKSPACE_READ_STATE_PROTOCOL_VERSION`**）是在同一批回执之上的显式已读手势。*未读*会重新点亮该
工作区中最近活跃、空闲、未阻塞、无后台工作、且其最近完成已读且未被取消的会话，并回答那条
`record`（没有合格项时为 `null`）；*已读*确认该工作区中每一条未读完成，并回答变化后的 `records`。
两者都覆盖仅磁盘会话，永不触碰需要输入，并通过 `session.state` 推送让其他客户端收敛。v86 之前的
宿主两者都不提供，因此客户端隐藏它们。

请求注册表还保留 `session.activityList` 在一个兼容窗口内返回字面量 `[]`，使已加载的旧客户端在重连
后清除已退役的标记。不再返回任何活动推送或类型。

## 必须做对（Get right）

- **镜像在注释中不钉版本。** 因为真正的家是仅 Node 的而在本处重新声明的形状，只携带它镜像了*什么*，
  绝不携带*它上次对照哪个 pi 版本核对过*：那些标记每次升级都必须在多个文件里手工编辑，而且没有
  任何东西验证它们。与 UI 相关的生命周期成员必须在此显式（尤其是宿主增强的 `agent_settled`，以及
  `entry_appended`，宿主把它投影为 `{ entryType }`，使 UI 能刷新统计）；与 UI 无关的会话事件可以
  保持未建模并被忽略。当某次升级的变更日志触及某个镜像时才重新审计它，而不是因为某个注释写了一个
  版本号。
- **始终仅类型，且从包根导入**（仅类型导入会被 `verbatimModuleSyntax` 擦除，因此 web 包保持不含
  provider；pi-ai 的 provider/API 子路径会静态导入 Node SDK——永不触碰它们）。`/base` 条目只存在于
  0.79.8–0.79.9。
- **路径字段携带拼写契约**（架构决策 22）。`Project.path` 和 `Workspace.worktreePath` 是宿主解析后的
  绝对拼写。每个 worktree 相对的 `path`（`FileNode`、`GitFileChange`、`ReviewAnchor`、
  `ReviewReadingStep`、`WorkspaceFsChangedPayload.paths`）都是 POSIX `/` 且位于 worktree 内部。
  pi 报告的绝对路径（工具参数、转录中的 `cwd`）可以使用任一分隔符；消费者只在规范化之后比较它们。
- `Model` 是泛型的——以 `Model<any>` 暴露。
- `AssistantMessageEvent`（流式增量）嵌套在 `message_update.assistantMessageEvent` 之下，永不是顶层
  事件 `type`。
- 内部相对导入**不带扩展名**（`./domain`），而不是 `./domain.ts`——`composite` 会产出声明文件，这与
  `allowImportingTsExtensions` 不兼容。
- **打包门禁：** 对 web 应用执行 `bun build`，并确认**没有**出现 `@anthropic-ai/sdk` / `openai` /
  `node:fs`。

## 消费方（Consumed by）

`web`（类型 + 上文列出的运行时导出）和 `server`（相同，外加把 `session.*` 映射到 `AgentSession` 方法）。
shell 面板需要 `domain` + `wsProtocol`；`pi` 类型 + `PiEvent` 是 agent 会话的线上契约。
