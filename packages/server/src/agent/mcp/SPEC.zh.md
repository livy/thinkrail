# agent/mcp —— 宿主自有的 MCP 层：pi 对等的配置加载器、写入器、指纹、按会话的宿主扩展

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

ThinkRail 在 pi 内置 MCP 周围自有的部分：在配置层，像 pi 那样精确读取两个 `mcp.json` 文件、
从一个长期存活的多客户端进程中安全地编辑它们，以及两个只有宿主才需要的派生——代替 codemode
的暴露方式归一化，以及支撑仓库服务器审批的按条目指纹；在会话层，守护调用的按会话宿主扩展，
以及防止同一台服务器的两次登录同时进行的按服务器锁；还有从 pi 共享的 MCP 日志中读取单个
服务器的行。引擎（连接、OAuth、工具）仍归 pi 所有；本模块从不打开连接。

升级时的移除门禁见 [[pi-compatibility]]；下文的契约拥有实现细节。

## pi 事实（pi facts (`@earendil-works/pi-coding-agent`, the catalog-pinned version)）

pi 不导出 `loadMcpConfig` / `validateMcpServerConfig` / 它的写入器，而宿主提供的 `loadConfig`
会整体替换 pi 的加载器——所以 `config.ts` 是 `dist/core/mcp-servers.js` 与
`dist/extensions/mcp/config.js`（MIT）的**移植**。与安全相关的规则在构造上完全相同，并由
`config.test.ts` 固定：该测试用同一批夹具分别跑 pi 已安装的加载器（由于其包的 `exports` 映射
隐藏了它，因此按 dist 路径访问）与我们的实现：服务器名的字符集、`exposure` 取值与
`codemode-deferred` 别名、`toolExposure` 形状、`type: "sse"` 的拒绝、URL/`headers`/`args`/
`env`/`cwd` 的类型校验、OAuth（`callbackUrl` 仅限回环、`callbackPort` 一致性、CIMD 约束、
`authServerMetadataUrl`）、`auth.provider` 只能配合 https 或回环地址且**绝不出现在项目文件**、
项目覆盖条目（仅 `enabled` / `exposure` / `toolExposure`，必须落在已存在的全局服务器上，报告时
与 `scope: "global"` 和一个 `override` 路径合并），以及 `-`/`_` 的命名空间冲突。每次 pi 升级都
重新验证；对等性失败是移植缺陷，绝不是放宽某条规则的理由。开头的一个 UTF-8 BOM 被加载器、
写入器以及原始条目占用/指纹读取一致地忽略。成功的编辑写出不带 BOM 的普通 JSON；被拒绝的编辑，
或移除一个不存在的名称，保留原始字节。这样 Windows 上创建的文件仍然可编辑，同时既不绕过审批，
也不会把已占用的名称当成不存在。

## 这里有什么（What's here）

- `summaries.ts` —— `summarizeMcpServers()`：每个条目按其在文件中书写的样子产生一个
  `McpServerSummary`（仅覆盖的项目条目会折叠进它的用户级服务器，同时带上它的 `approval` 和
  它的校验错误；不是对象的条目保留其行并带上 pi 的 `must be an object` 错误），包含作用域、
  定义文件、该行自身原始条目的 `fingerprint`（`fingerprintMcpEntry` 作用于定义文件中按原样
  书写的条目，两个作用域都是如此——这是重写或移除时必须出示的身份）、传输层、掩蔽后的端点
  （`maskMcpEndpoint`）、该行是否为被准入的有效定义、配置的与生效的暴露方式与启用状态（记录中
  的覆盖与已准入的仓库覆盖都已应用）、OAuth 适用性、是否替换了某个用户级服务器、审批状态
  （相对当前指纹为 `approved` / `pending` / `changed`）、配置的描述与等同 pi 的配置错误（两者都
  经过 `redactMcpText`），以及在生效行上那些授权仍与其目标匹配的 `alwaysAllowed` 工具。
  `summarizeMcpConfigErrors()` 是文件级的其余部分（不可解析的 JSON、`mcpServers` 不是对象、
  类型写错的 `autoEnableCodemode`），以 `{ source, message }` 形式、掩蔽后供
  `McpListResult.configErrors` 使用：对前两种情形，pi 的加载器不会列出该文件的任何服务器，所以
  没有它 Settings 会读成“没有服务器”。
- `redact.ts` —— 宿主对离开自身的文本使用的唯一凭据掩蔽器。pi 在其状态文本里不做任何脱敏：
  连接错误、HTTP 错误响应体片段和 stdio 服务器的 stderr 尾部（最多 2 000 字符）都按原样到达。
  `redactMcpText` 以确定性且幂等的方式掩蔽 URL 用户信息与以凭据命名的查询值（`maskUrl`；其他
  URL 按原样返回）、`Bearer` / `Basic` 凭据（任意大小写）、以凭据命名的 `key=value` 与未加引号的
  `key: value` 对（就像服务器的 stderr 打印的那样；`Bearer`/`Basic` 的值保留其方案）以及带引号的
  `"key": value` 对、已知的令牌前缀（`gh[pousr]_`、`github_pat_`、`sk-`、`xox[abprs]-`、`glpat-`、
  `AKIA`），以及长度 32 以上、属于十六进制或混合大小写且含数字的 base64url 字母表字符的连续串——
  `/` 与 `.` 会终止一个连续串，因此路径、URL 和包名得以保留（被 `/` 切成短片段的标准 base64
  密钥属于被接受的漏网情形）。它应用在 pi 的文本变成宿主 DTO 的地方：两个状态解析器（每个
  `detail` 与解析出的配置错误）、`summarizeMcpServers`（`configError`）、
  `summarizeMcpConfigErrors`、`readMcpServerLog`、面向旧版客户端的启动通知，以及登录探测的错误
  帧。`maskMcpEndpoint` 是命令行形式：URL，或带掩蔽的命令——以凭据命名的旗标或赋值之后的值、
  形如 URL 的参数（用户信息、以凭据命名的查询值）、头部参数（`-H` / `--header` 之后的任何值，
  以及任意位置以凭据命名的 `Name: value`）、带令牌前缀的参数都被掩蔽；`${VAR}` 引用保留，含空格、
  引号或为空内容的参数会以 JSON 加引号，使该行不含歧义（审批对话框把它显示为实际运行的内容）。
- `status.ts` —— pi 两种状态文本的解析器（`formatStatus` 的行
  `name: state[, N tools] (exposure)` 带缩进的错误，以及启动关注通知）与
  `deriveMcpServerStatuses()`；这些格式由夹具固定，并在每次 pi 升级时重新检查；多行的配置诊断不会
  丢弃有效的服务器行，解析出的细节在别的任何东西看到之前先脱敏。
- `log.ts` —— `readMcpServerLog(name)` → `McpServerLog`：pi 把服务器发送的每条
  `notifications/message` 追加到 agent 目录中的一个 `mcp.log`（超过 5 MiB 轮转为 `mcp.log.1`），
  每条消息一个条目，形式为 `<ISO time> [<server>] <level>[ <logger>:] <text>`，续行缩进四个空格
  （`formatMcpLogMessage`，因此一台服务器无法伪造另一台的开头）。`mcpServerLogLines` 保留头部
  指明该服务器的条目及其续行——绝不做子串匹配——并取最近 200 行；两个文件都会被读取（各取最后
  8 MiB，缺失的文件视为空），结果再经过 `redactMcpText`。该格式由针对 pi 自身格式化器的测试固定；
  每次 pi 升级都重新检查。
- `results.ts` —— `summarizeMcpResult()`：会话宿主的 `tool_result` 处理器为 `mcp__*` 和三个
  resource 工具写入 `details.thinkrail` 的 `McpResultSummary`，它根据 pi 的机器可读结果
  （`structuredContent` = 服务器的原始 `CallToolResult`，或 resource 载荷）构造：块类型、URI、
  MIME 类型与大小——绝不包含图像/音频/blob 数据——外加服务器自己的 `structuredContent`。一份
  字节预算（`MCP_SUMMARY_BYTES`，64 KiB）约束整个摘要：块按顺序保留直到预算用尽（`omittedBlocks`
  统计剩余数量，因为一台服务器可能返回成千上万个 URI 很长的 resource 链接），而
  `structuredContent` 仅在剩余额度容得下时保留（否则 `structuredContentTruncated`）；resource 读取
  只保留其内容的 URI 与类型。pi 会从最终消息中丢弃 `structuredContent`，所以正是这个摘要让卡片
  能实时渲染相同内容、重载后如此、在另一客户端上也是如此。
- `management.ts` —— `effectiveMcpServerConfig` 通过会话所用的同一个加载器读取已准入的定义，
  因此探测从不从展示行去推断它。宿主组合出 `writeMcpServerEntry`（pi 的校验外加项目级的 `auth`
  禁令、新增与更新的区分、项目范围包含；返回项目条目的指纹）。新增要求名称在定义文件的
  `mcpServers` 中不作为自有属性存在，无论该条目是否有效：标量、null 和数组值在用户级与项目级
  两个作用域都占用该名称，即使它们出现在客户端列表读取之后。拒绝既不改动文件字节，也不改动审批。
  更新则有意仍要求存在对象条目；它不会替换格式错误的非对象值。其余助手是
  `removeMcpServerEntry`、`mcpEntryFingerprint(scope, worktree, name)`（该作用域文件中当前原始条目
  的指纹——全局 `mcp.json` 或工作树的 `.pi/mcp.json`——不存在时为 `undefined`；宿主在任何重写或
  移除之前用它比对渲染出的 `expectedFingerprint`），以及 `shareMcpOverrideWithRepo`（从记录中写出
  pi 的覆盖条目；当它所覆盖的用户级服务器已不存在时拒绝——没有定义的覆盖条目会是无效条目——
  当项目文件含有任何不是覆盖对象的自有值时拒绝，或当它含有一个未在其当前指纹上获批的覆盖时
  拒绝，因此一次分享绝不会合并进未经审查的仓库内容）。分享会独立于有效性检查原始仓库占用；
  拒绝一个格式错误但已占用的条目会保留文件、审批与记录覆盖。它的用户级定义读取与写入覆盖校验
  有意仍只做对象检查，而不是缺失检查。
- `config.ts` —— 移植加宿主新增部分：`loadHostMcpConfig()` 是会话交给 pi 的东西——项目文件条目
  （服务器与仅覆盖条目一样）只有在项目记录的 `mcpApprovals[name]` 等于其当前指纹时才被准入
  （未获批的覆盖会让全局服务器保持配置中的样子），记录中的 `mcpOverrides` 设置用户级服务器的
  `enabled` / `exposure`，可选的聊天本地禁用集合优先于两者，随后曝光方式被归一化；
  `loadMcpConfigFiles()`（以及它所基于的可选 `admitProjectEntry` 谓词）返回 pi 的
  `LoadedMcpConfig` **和** `entries`（每个具名 `mcpServers` 值按书写的样子——对象与无效值一视同仁
  ——附带文件、作用域以及是否为覆盖），因为 pi 的合并结果会丢失项目来源，**还**返回 `fileErrors`
  （`errors` 中文件级的子集，拆成路径与消息）；`fingerprintMcpEntry(name, raw)` = 对名称与整个原始
  条目的规范 JSON 做 SHA-256（键递归排序，`toolExposure` 除外，其顺序有语义）——绝不基于解析后的
  值；`normalizeExposureForHost()` 在 codemode 尚未交付期间，把未设置与 `codemode` 的暴露方式
  （服务器级与按工具级）在内存中映射为 `deferred`。`mcpPolicyOf(project)` 从项目记录中读出该策略
  （审批、覆盖、工具授权）（缺失字段读为空）；`mcpTargetOf` / `grantedMcpTools` / `mcpToolLabel`
  （去掉 `mcp__<server>__` 的工具名）支撑工具授权。
- `sessionHost.ts` —— `createMcpSessionHost({ policy, onCommandSettled?, askForToolCall, grantTool })`，
  每个父会话一个（管理器把它挂在条目上；子会话没有）：一个针对每个 `mcp__*` 调用的 `tool_call`
  处理器，前提是该调用的工具没有声明 `readOnlyHint: true`。对在本聊天中被允许或持有有效项目授权的
  工具跳过询问；否则通过注入的 `askForToolCall` 询问（即管理器的 `selectForToolCall`，于是对话框
  指明 `event.toolCallId`、渲染在该调用的卡片上，并把 `event.input` 作为 `toolInput` 携带——即到达
  守护处的输入，提示中完整列出；标题绝不嵌入预览），选项为 **Deny · Allow once · Allow in this
  chat · Always allow**，顺序如此，使得在堆叠的移动端布局里持久的授权不会紧挨着 Deny。当提供
  Always allow 时，标题第二行说明其作用域（“Always allow trusts this tool in every chat of this
  project. Manage it in Settings › MCP servers.”）；web 端把后续标题行渲染为辅助文本，因此这段
  文案不需要契约字段。任何非肯定的回答（Deny、关闭对话框、被一条用户消息抢先回答、中止、没有 UI）
  都以一个理由阻塞。它唯一接线的中止是运行的 `ctx.signal`，传给该端口；pi 接受的用户消息通过
  桥接的通用 `cancelExtUiForSession` 取消对话框（[[submodule-server-agent]] › `webUiContext`），
  因此宿主既不持有自己的取消集合或 `AbortController`，也不暴露取消方法。“Allow in this chat” 存在
  宿主对象里，所以它能挺过资源重载，并随会话消亡。**Always allow** 只在 `loaded()` 认识该工具所属
  服务器时提供（pi 的 `ToolInfo.namespace.aliases[0]`）；它调用
  `grantTool(server, mcpTargetOf(config), toolName)`，绝不触碰聊天内集合，所以在 Settings 中的撤销
  会在开放聊天的下一次调用时生效。授权在每次调用时通过 `policy().toolGrants` 读取
  （`grantedMcpTools`）：授权只有在它的 `target`——有效条目的 `command` + `args`，或 `url` 按书写
  样子的 SHA-256——仍然匹配时才算数，因此 env、header 或 exposure 的编辑保留授权，而新的 command
  或 URL 会重新询问。工具以 pi 的工具名为键，这是守护处唯一可见的名称（pi 的 `ToolInfo` 不携带
  原始 MCP 工具名）。授权存放在项目记录中，绝不在 `.pi/mcp.json` 里，因为仓库可能会预先附带授权。
  它的 `extensions` 还带上引擎（`engine.ts`）以及会话的 `loadConfig`（`loadHostMcpConfig` 作用于
  `getAgentDir()`、会话 `cwd`、`ctx.isProjectTrusted()` 和注入的项目策略）。宿主扩展是一个
  始终加载的内联工厂，因此它的工厂在每次加载时启动：它清空 `loaded()`（每台服务器按 pi 收到的
  样子记录的有效配置，只有在内置扩展启动时才再次设置——若某次加载中另一个扩展拥有 `/mcp`，
  或存在 `-builtin:mcp`，则保持为 `null`），并为 `registered()` 保留那次加载的 `ExtensionAPI`
  （pi 的 `getMcpServers()`：扩展注册的服务器，加载过期后为 `[]`）。`shutdownEngine(event, ctx)`
  运行引擎捕获的关闭逻辑。管理器在一次受门禁的重载之前捕获 `loaded()` 是否非空：失败的替换可能
  清空或更新该快照，却没有成功应用运行时。结合任何新加载的引擎，这一来源信息让父级 `mcpSessions`
  跟踪对象能为即时、延迟、手动和聊天本地覆盖的重载保留一次可重试的失败，即使已无剩余的配置差异。
  只有成功的重载才会清除它；本宿主不持有第二个失败标记。可选的命令完成回调由会话管理器提供；
  本子模块从不导入状态协调器。
- `engine.ts` —— `createMcpEngine({ loadConfig, lockSignIns?, openUrl?, onCommandSettled? })` →
  `{ extensions, shutdown }`：pi 的两个内置描述符（`mcp` = `createMcpExtension`、`tool-search`），
  `replaceable: true, builtin: true`，与 pi 的 CLI 完全一致。可选的宿主回调通过 `openUrl` 交付
  Settings 授权 URL，并在变更类命令之后使状态失效。普通聊天绝不在宿主机上打开浏览器。`mcp` 工厂
  先注册自己的 `session_start` / `mcp_servers_change` 处理器，在 `pi.getMcpServers()` 非空时激活
  `tool_search`，于是它们先于 pi 的可达性检查运行：由扩展注册的服务器绕过加载器的归一化（pi 默认
  把它们当作 codemode，而 codemode 尚未交付），`tool_search` 也能到达它们。由于活在可替换的内置
  扩展里，当另一个扩展拥有 MCP 时，该激活也随之消失，而在那里处理 `mcp_servers_change` 不会扭曲
  pi “没有人连接已注册服务器”的报告。`mcp` 工厂交给 pi 一个 `ExtensionAPI` 的 `Proxy`；除两个成员
  外的所有成员都用 `Reflect.get` 转发（pi 的 API 对象从不被展开，因此没有任何东西被提前读取）。
  每次工厂加载都拥有一个关闭标志和一个清理 promise。`on` 捕获 pi 的 `session_shutdown` 处理器；
  `shutdown(event, ctx)` 与 runner 的关闭分发都会同步关闭那次加载，并加入同一个清理 promise。pi 在
  等待传输层关闭之前就清空其服务器列表（每次升级都重新验证过），所以第二次原始分发会提前返回：
  仅靠幂等并不能保持完成顺序。重载必须等待真正的清理之后再准入替代品，而不是那个有界的退出等待。
  proxy 守护真正的 `session_start` 与 `mcp_servers_change` 回调（包括 tool-search 的激活）以防
  某次加载已关闭：更早的异步处理器可能在关闭完成之后才恢复分发。这些守护只拥有那次加载的引擎
  资源，而不拥有持久的会话生命周期状态。每次 pi 升级都要重新检查这些顺序保证。除非调用方传入
  `lockSignIns: false`（只有 Settings 探测会这样，它自己持有锁），否则 `registerCommand` 会包装
  `/mcp` 的处理器：`login <name>` / `logout <name>` 按 pi 的解析方式解析后，在命令持续期间获取
  `acquireMcpSignInLock(name, "chat")` 并在 `finally` 中释放；当另一个持有者持锁时，该命令以警告
  形式发出 “A sign-in for "<name>" is already running in Settings.”（或 “in a chat”），绝不进入 pi。
  状态、`reconnect`、裸的 `login` / `logout` 以及格式错误的参数保留 pi 的分发行为。变更类命令的
  完成会触发所提供的状态失效，包括失败；状态读取则不会，从而避免捕获递归。
- `signInLock.ts` —— `acquireMcpSignInLock(serverName, holder)`：进程级锁，以 pi 归一化后的服务器
  命名空间为键（`-` 与 `_` 别名共享同一把锁，与 pi 的凭据身份一致），由 Settings 探测
  （`"settings"`）与聊天（`"chat"`）共用（原因与持有者：[[submodule-server-agent]] › MCP servers）。
  它返回 `{ release }`（只释放自己的持有，因此迟到或重复的调用无害）或 `{ heldBy }`，调用方据此
  形成拒绝。
- `writers.ts` —— `add` / `update` / `remove`，沿用 pi 的编辑语义（未知键与缩进保留；全局服务器的
  `enabled: true` / `exposure: "codemode"` 默认值被移除而非写入；覆盖条目保留显式值），原子地替换
  文件（临时文件 + 重命名）并保留其权限。用户文件的符号链接仍是符号链接：写入器解析并原子地替换
  它的真实目标，使用该目标的权限，绝不使用链接本身宽松的权限。`assertProjectMcpConfigWritable(worktree, path)`
  是项目文件的包含规则：目标留在工作树内，已存在的 `.pi/` 是真实目录，已存在的 `mcp.json` 是
  普通文件——绝不为符号链接，因为 pi 的读取器会跟随链接，而一次读-改-写会把外来文件复制进仓库
  （`McpConfigPathUnsafeError`，线上代码 `MCP_PATH_UNSAFE`）。写入是同步的，因此单个进程绝不会
  交错两次编辑；原子重命名让并发的 `pi mcp add` 永远读不到半截文件（两个进程之间丢失更新是被接受的）。
  `add` 会创建缺失的文件，并报告是否替换了同名条目；`remove` 报告该文件是否定义过该服务器；
  带 `override` 的 `update` 会添加缺失的覆盖条目。

## 边界（Boundary）

- **公开接口（barrel）：** `loadMcpConfigFiles`、`validateMcpServerConfig`、`mcpNamespace`、
  `isOverrideEntry`、`fingerprintMcpEntry`、`normalizeExposureForHost`、`globalMcpConfigPath`、
  `projectMcpConfigPath`、`addMcpServerConfig`、`updateMcpServerConfig`、`removeMcpServerConfig`、
  `assertProjectMcpConfigWritable`、`McpConfigPathUnsafeError`、`createMcpSessionHost` +
  `McpSessionHost`，以及选项标签 `MCP_CONFIRM_DENY` / `MCP_CONFIRM_ONCE` / `MCP_CONFIRM_CHAT` /
  `MCP_CONFIRM_ALWAYS`、`createMcpEngine`（`{ loadConfig, lockSignIns?, openUrl?, onCommandSettled? }`）、
  `acquireMcpSignInLock`、`loadHostMcpConfig` + `McpProjectPolicy` + `mcpPolicyOf`、
  `summarizeMcpServers`、`maskMcpEndpoint` / `redactMcpText`、`readMcpServerLog`、
  `effectiveMcpServerConfig`、`management.ts` 的变更操作（`writeMcpServerEntry`、
  `removeMcpServerEntry`、`mcpEntryFingerprint`、`shareMcpOverrideWithRepo`）、
  `summarizeMcpConfigErrors`、状态解析器（`parseMcpStatusText`、`parseMcpAttentionNotice`、
  `MCP_ATTENTION_PREFIX`、`deriveMcpServerStatuses`、`ParsedMcpStatus`）、`summarizeMcpResult` /
  `isMcpResultTool` / `MCP_SUMMARY_BYTES`，以及类型 `LoadedMcpFiles`、`McpConfigFileEntry`、
  `McpConfigScope`、`McpServerConfigPatch`。
- **允许的依赖：** `@earendil-works/pi-coding-agent`（类型、`CONFIG_DIR_NAME`、`getAgentDir`，以及
  引擎工厂 `createMcpExtension` / `createToolSearchExtension`）；`@thinkrail/contracts`（类型）；
  `@thinkrail/shared/codedError`；Node 的 `fs`/`path`/`crypto`。
- **禁止：** `host`、`projects`、web；不得包含连接、OAuth 或传输层代码——那些留在 pi。
