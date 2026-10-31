# Pi 兼容性——宿主适配与升级移除门禁

> 中文译文，原文见 [`pi-compatibility.md`](./pi-compatibility.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

这是为补偿缺失的 pi API、pi 缺陷或打包约束而存在的宿主代码的升级检查清单。Pi 仍是引擎与状态
所有者。模块规格拥有详细理由；本索引记录一次升级必须重新核对什么，以及什么条件允许我们删除
该适配。已安装版本只来自根依赖 catalog，而不是本文件中的第二个锁定。

## 最近一次重新核对（Last re-check）

压缩输出上限的升级（[upstream #11036](https://github.com/earendil-works/pi/pull/11036)）已针对
已安装的 catalog 版本核对过。**没有任何移除门禁被满足。**

- MCP loader/writers 仍是私有的；管理/状态仍依赖命令、通知与日志。`openUrl` 仍是公开的
  登录 URL 接缝。唯一一处对等性变化是接受单个前导 BOM，现在由宿主 loader、writers 以及
  原始条目占用/指纹读取共同遵循。配置与宿主测试覆盖两种作用域、已批准的覆盖、格式错误的
  占用名称以及拒绝写入时的保留。
- `AgentSession.dispose()` 仍不发出扩展关闭事件；重载调度是顺序的，且 MCP 在等待关闭之前
  先清空其服务器列表。保留有序清理与晚启动准入。Codemode 仍需要一个我们不打包的 worker/WASM
  产物面。原始 `structuredContent` 仍是临时的，不是有界持久的记录呈现。
- 已投递队列的移除与 `clearQueue()` 仍只处理文本。SDK 创建仍不初始化无头终端主题。在产物中
  扩展加载仍需要 Jiti/虚拟模块适配、静态 OAuth/Bedrock 注册以及图片缩放覆盖。产物探针还
  通过外部扩展 loader 演练一次来自 `pi-ai/models` 的值导入。
- 公开的信任检测并不取代资源种类发现/排除式重载；会话发现与 prompt 模板 API 仍不覆盖宿主的
  读取/编辑界面。保留它们的对等性测试。
- Pi 现在不再依赖由 reserve 推导的输出上限来确定历史、轮次前缀与分支摘要的大小。推理继承与
  拒绝不完整的 `length` 摘要仍然存在。一个公开的压缩金丝雀测试在不同 reserve 下检查轮次
  前缀的答案空间，而不锁定 provider 选项策略；生命周期测试保留取消与不完整摘要拒绝的覆盖。
  已恢复的截断调用恢复现在也会省略配对的工具结果，但不能取代 `sessionRepair` 的宿主重启与
  问卷修复。没有引入宿主摘要器、努力默认值改动、reserve 调优、传输线迁移或会话迁移。
- Pi 1.3 的恢复式工具装载还原增加了一个宿主接缝，即下面 attach 处的 registry 工具对账。
- 内置聊天目录从 1,554 对 provider/模型变为 1,599 对：移除 71 对，新增 116 对。被移除的对
  包括 `openai-codex/gpt-5.5` 与 `openai-codex/gpt-5.3-codex-spark`；`openai/gpt-5.5`
  仍在。这是内置目录的变化，并不表示 provider 已下线这些 API；已存储、已刷新与自定义目录
  可能不同。发布说明必须提醒用户在继续之前检查其默认模型与恢复聊天的模型。不可用的默认/
  恢复模型遵循 [[submodule-server-agent]] 中现有的、不同的宿主/Pi 选择规则；宿主仍不暴露
  Pi 的 `modelFallbackMessage`。这次升级没有增加任何回退或警告 UI。
- 自 Pi 1.3 起，`kimi-coding` 在 `https://api.kimi.com/coding/v1` 上使用 OpenAI Chat
  Completions。使用自定义 `models.json` 覆盖 base URL 的用户必须包含 `/v1`；宿主不会重写
  这些覆盖。把这条迁移提示写进发布说明。

## 升级契约（Upgrade contract）

每次 pi 升级都要针对**已安装包的导出与实现**核对每一条，而不仅是发布说明或上游 issue 的
状态。升级的验证摘要会列出被移除、带证据保留或被改变的适配，以及运行过的相关测试。新增的
绕过方案在此增加一条，并在其所属规格中增加理由；移除它时既删除绕过方案，也删除这一条记录。
对等性失败是漂移的证据，而不是弱化测试的许可。

公开 SDK 的替代必须在宿主实现被移除之前保持会话隔离、取消、项目信任、持久记录，以及编译/
桌面行为。在生产中导入私有 dist 路径不是替代；这类导入仍限于对等性测试。

## MCP 适配（MCP adaptations）

### 配置 loader 与 writers（Configuration loader and writers）

**所有者：** [[submodule-server-agent-mcp]]，`agent/mcp/config.ts` 与 `writers.ts`。
Pi 公开的 `loadConfig` 钩子整体替换它的 loader；它的 loader、校验器与 writers 不是公开导出。
宿主移植其文件语义，以应用项目策略并执行安全的多客户端写入。**移除门禁：** 出现一个公开的
loader/校验器/writer 面，能保留条目来源，并允许在项目覆盖合并之前完成准入。针对已安装的
`core/mcp-servers` 与 `extensions/mcp/config` 验证该移植；运行 `agent/mcp/config.test.ts`
与 `host/mcp.test.ts`。宿主侧的遏制、陈旧写入检查与私有策略仍是宿主的义务。

### 运行时状态、管理与登录结果（Runtime status, management and sign-in outcomes）

**所有者：** [[submodule-server-agent]]，`mcpSessions.ts` / `mcpSignIn.ts`，以及
[[submodule-server-agent-mcp]]，`status.ts` / `log.ts`。
Pi 通过 `/mcp` 与通知暴露管理能力，而不是一个带类型的控制器。宿主捕获这些命令、解析状态与
结果文本、读取 pi 的日志格式，并用一个隔离的 pi 会话做 Settings 探针。OAuth 与凭据留在 pi
中；授权 URL 可通过 pi 公开的 `openUrl` 回调获得，无需文本解析器。**移除门禁：** 出现公开的
按会话状态读取/订阅与可取消的管理/认证 API，以及用于日志读取的结构化诊断。运行
`mcpEngine.test.ts`、`mcpSignIn.test.ts` 以及 MCP 状态/日志/脱敏测试；当 pi 的措辞变化时，
同时验证现代状态推送与旧式通知脱敏。

### 有序的引擎关闭与重载准入（Ordered engine shutdown and reload admission）

**所有者：** [[submodule-server-agent]] 的会话生命周期，以及
[[submodule-server-agent-mcp]]，`engine.ts`。
`AgentSession.dispose()` 不发出关闭事件，而 pi 顺序化的扩展关闭可能被一个无关的处理器拖住。
宿主捕获内置的 MCP 关闭处理器，并在普通拆除/重载之前启动它，同时保留一条有界的宿主退出
路径。被捕获的关闭与 runner 发出的关闭共享同一个按加载的清理 promise；重载在准入替代者之前
等待实际清理完成。Pi 在等待关闭前先清空其服务器列表，因此幂等性不足以保证完成顺序。宿主还
在关闭之后守卫实际的 MCP 启动/变更回调：`beforeSessionStart` 先于整个异步调度，单靠它无法
阻止晚启动。**移除门禁：** 出现一个公开的生命周期，它独立于无关处理器关闭 MCP、在重复关闭时
汇入同一份实际清理、只在清理完成后准入替代者，并且即使先前的处理器在 dispose 之后恢复也能
阻止启动。重新核对完成顺序与 `AgentSession.reload` 钩子，而不只是幂等性；运行
`sessionShutdown.test.ts`、`reloadGate.test.ts` 以及 MCP 删除/重载/进程生命周期的回归测试，
其中包括一个顽固的旧 PID 和一个在关闭过程中被异步 `resources_discover` 暂停的启动调度。
仅凭 `AgentSessionRuntime.dispose()` 不能证明这些宿主义务已经消失。

### codemode 未被打包时的暴露（Exposure while codemode is not packaged）

**所有者：** [[submodule-server-agent-mcp]] 的 `normalizeExposureForHost` 与工具搜索激活。
文件配置的 `codemode` 暴露变为 `deferred`；已注册的服务器仍可通过 `tool_search` 触达。
**移除门禁：** 在 CLI 与桌面之间交付 pi 的 codemode worker/WASM，然后在不做归一化的情况下
保留原生暴露。运行配置/暴露测试与共享产物探针；仅公开 factory 可用并不构成打包支持。

### 持久的呈现细节（Durable presentation details）

**所有者：** [[submodule-server-agent-mcp]]，`results.ts`，以及 [[submodule-web-chat]]。
Pi 的原始 `structuredContent` 会到达钩子，但不会保留在定稿的记录消息中。宿主存储有界、
不含 base64 的呈现细节，使实时卡片与恢复后的卡片一致。**移除门禁：** pi 提供一个等价的
有界持久表示。重新核对结果形状与完整输出路径；运行 `mcp/results.test.ts`、
`mcpEngine.test.ts`、web MCP 卡片测试与 `e2e/mcp-tool-cards.spec.ts`。绝不为了删除适配器
而持久化完整的原始结果。

## 其他对 pi 敏感的宿主接缝（Other pi-sensitive host seams）

**attach 处的 registry 工具对账**——[[submodule-server-agent]]，`agentSessionManager.ts`。
Pi ≥ 1.3 会还原恢复聊天的已记录工具装载，并保持已记录的移除处于非活动状态，因此一个在其
registry 扩展被禁用期间被扣留的工具，在重新启用后会仍然关闭。Attach 通过镜像 Pi 私有的
`_isActivatedOnRegistration`（可声明的暴露，`defaultActive !== false`）重新激活已启用
registry 扩展的工具；升级时重新核对这条规则。如果宿主将来调用 `navigateTree`（它会重跑
还原），也要在其后对账。**移除门禁：** Pi 提供公开方式，把宿主扣留的工具与已记录的选择区分
开。升级时运行管理器的重启回归测试。

- **Anthropic SSE 分帧补丁**——[[submodule-server-agent]]（Provider 流分帧），
  `patches/@earendil-works%2Fpi-ai@1.4.0.patch` 作用于 `dist/api/anthropic-messages.js`。
  不打补丁时，两种流会以不可重试的 “Could not parse Anthropic SSE event” 失败：一种是
  有效的 CRLF 流，其 `\r\n` 被网络分块切开；另一种是事件中途被切断的流。打上补丁后，第一种
  成功，第二种以可重试的截断错误结束，例如 “stream ended before message_stop”。
  **移除门禁：** 已安装的 pi-ai 版本同时修复这两者（跟踪
  [upstream #10390](https://github.com/earendil-works/pi/issues/10390)）。Bun 会静默跳过
  版本不匹配的补丁，因此每次 pi 升级都要针对未打补丁的版本运行
  `agent/anthropicStreamFraming.test.ts`。它的 “defects the patch fixes” 组说了算：通过时
  删除补丁及其 `patchedDependencies` 条目；失败时把补丁移植到新版本。若只有
  “behavior the patch keeps” 组失败，说明 pi 改变了该行为：在移植任何东西之前先决定是否
  跟随它。
- **Registry 工具优先级**——[[submodule-server-extensions]]，会话 loader 的
  `extensionsOverride` 中的 `preferRegistryTools`。它依赖三条 pi 事实：
  - pi 保留每个扩展，并把一个工具名交给它的第一次注册
    （`ExtensionRunner.getAllRegisteredTools`）；
  - 路径扩展（用户包）先于内联 factory 加载；
  - 工具冲突只在扩展之间报告，且发生在覆盖运行之前。

  该覆盖把一个 ThinkRail registry 扩展移到第一个注册其某个工具名的用户包之前，因此像 rpiv
  的 `ask_user_question` 这样同名的包无法接管以该名字为键的卡片、邮箱与修复逻辑。当 pi 提供
  内联扩展的优先或替换选项时移除它。在已安装的 dist 中重新核实这三条事实，并在每次升级时
  运行 `agentSessionManager.test.ts` 中的冲突测试（重载前后的同名问卷，以及一个在关闭我们的
  扩展时创建、随后又在包之上重新启用它的聊天）。
- **仅图片的排队输入**——[[submodule-server-agent]]，队列投射。宿主补偿 pi 只处理文本的
  已投递消息移除（[upstream #8612](https://github.com/earendil-works/pi/issues/8612)），
  并保留被破坏性队列 API 丢弃的图片内容。当已安装的实现原生处理该情形时移除对应的绕过方案；
  验证队列/图片回归，而不只是 issue 状态。
- **内置扩展加载与图片处理**——[[submodule-server-agent]]、[[module-cli]]、
  [[module-desktop]]。强制 Jiti 变换、虚拟模块身份、静态 provider 注册与图片缩放覆盖，
  桥接单文件产物无法发现或加载的资源。只有当 pi 提供等价的产物接缝时才逐一替换；运行
  `check:seams` 以及相关的二进制/桌面产物探针，包括扩展值导入与图片。
- **按账户的模型可用性**——[[submodule-server-agent]]，`modelAvailability.ts`。Pi 的
  `openai` provider 在 Sign in with ChatGPT 下列出其整个 API 目录（其 `filterAllModels`
  只丢弃分类器），因此宿主把相关 id 保留在 pi 的 `openai-codex` 目录中；Central 代理宣传
  的目录比 pi 的更窄，因此宿主把不透明 provider 与其 `/models` 列表求交。当已安装的
  `providers/openai.js` 自行过滤 OAuth 聊天模型时移除 ChatGPT 规则，当 pi 把被重路由的
  provider 目录收窄到其端点实际提供的内容时移除列表规则；运行
  `modelAvailability.test.ts` 与 `runtimeGenerations.test.ts`。
- **默认模型偏好**——[[submodule-server-agent]]，`modelDefaults.ts`。Pi 的
  `defaultModelPerProvider` 未导出，因此宿主为新聊天、评审者与一次性默认值镜像它。当存在
  公开导出或初始模型 resolver 时移除该镜像；在那之前，`modelDefaults.test.ts` 中的对等性
  测试会在该映射或其顺序发生任何上游变化时失败。
- **无头主题**——[[submodule-server-agent]]，`plainTextTheme`。Pi 的终端主题不会由 SDK
  会话创建初始化。当 pi 提供一个可用的非 TUI 主题契约时移除该子类；针对整个继承面运行
  `webUiContext.test.ts`。
- **信任发现与排除式重载**——[[submodule-server-agent]] 的 `projectTrust.ts` 与
  `extensions.ts`。资源种类发现镜像 pi 私有的需要信任的路径，而排除式 loader 保留可重载的
  扩展路径与内置排除。有公开的发现/过滤 API 时优先使用它；在每次资源加载改动时运行
  `projectTrust.test.ts`、`extensions.test.ts` 与 `piRuntime.test.ts`。
- **会话发现与 prompt 模板解析**——[[submodule-server-history]]、
  [[submodule-server-templates]]、[[submodule-web-chat]]。在公开 API 无法服务其读取/编辑
  界面的地方，宿主镜像 pi 的会话目录/JSONL 发现与 prompt 模板/frontmatter 规则。当公开的
  发现/解析 API 满足这些需求时优先使用它们；在那之前，在历史、模板与输入框测试中保留对等
  性覆盖。
- **来自子记录的 subagent 用量合计**——[[submodule-server-agent]]，`subagentUsage.ts`。
  Pi 的会话合计循环（`AgentSession.getSessionStats`）未针对裸会话文件导出，因此宿主镜像哪些
  条目携带花费（`usage`、压缩/分支摘要的 `usage`、assistant 与工具结果消息的 usage）来对账
  崩溃的子运行。当 pi 导出针对条目或文件的合计辅助函数时移除该镜像。在那之前，
  `delegation.test.ts`（“the subagent totals mirror matches pi's own totals for every
  spend source”）在一份携带所有花费来源（包括分支外花费）的记录上锁定与 pi 的
  `getSessionStats` 的对等性；每次 pi 升级都重跑它。
- **失败 usage 追加的回滚**——[[submodule-server-agent]]，`recordSessionUsage`。
  `SessionManager._appendEntry` 在 `_persist` 之前就把条目加入内存并移动叶子，而一次失败的
  `appendFileSync` 可能留下撕裂的半行。因此宿主恢复先前的叶子，并把已存在的记录截断回追加前
  的大小；`delegation.test.ts` 锁定这两者（“a usage write pi kept only in memory never cuts
  the reopened history”、“a torn usage append is cut back…”）。当失败的持久化不再改变 pi
  的内存与叶子时移除叶子恢复；只有当 pi 也保证失败的追加不留下撕裂字节时才移除截断。

- **脱离式结果保留**——[[module-pi-subagents]]（结果投递）与
  [[module-pi-background-commands]]，由 [[submodule-server-agent]] 的 `turn_end` 与
  `compaction_end` 冲刷来应用。Pi 只在一次运行本来会停止时才排空后续消息，不给扩展任何方式
  检查或撤回一条已排队消息（`hasPendingMessages()` 是布尔值，`clearQueue()` 仅限宿主且
  清空一切），并且在中止后仍保留排队消息，因此一个被推送的结果会与 agent 已经拉取的结果
  重复。所有者持有结果，只在步骤边界发送它们，或发送给空闲的父会话。**移除门禁：** 出现
  扩展可撤回的按消息队列句柄，并在下一步而非运行结束时投递——即便如此，也要保留拉取时声明
  的规则。运行 `pi-subagents` 与 `pi-background-commands` 套件以及
  `agent/chatResources.test.ts`。
- **跨压缩的 skill 正文**——[[submodule-server-agent]]，`skillCarryover.ts`。Pi 的压缩会
  把已加载的 skill 文本摘要掉（`read` 结果到达摘要器时被截到 2000 字符），而且没有任何东西
  重新提供它，因此一个宿主 `context` 处理器在压缩后把被压缩掉的 skill 正文重新附加到每个
  请求上。当 pi 的压缩原生保留或重新附加已加载 skill 时移除它；用
  `skillCarryover.test.ts` 与 `bun run e2e:full e2e/skill-after-compaction.live.spec.ts`
  验证。它通过匹配 pi 未导出的 `"pi-virtual"` 模型 API 跳过虚拟选择，因为 `context` 处理器
  只能看到选择，看不到被路由的模型；当 pi 向 `context` 处理器暴露被调度的模型（或
  `isVirtualModel`）时替换该判断。

## 上下文预算限制（Context-budget limitations）

**所有者：** [[submodule-server-agent]]。这些是上游限制，不是宿主适配。在已安装的 SDK 中，
`findCutPoint` 会完整保留尾部的一对工具调用/结果，即使它单独就超出 `keepRecentTokens`；
而 `generateSummary` 序列化整个被摘要的区间时没有输入预算（自 Pi 1.4 起它会请求到模型输出
上限）。因此有界的工具结果并不保证整个请求可被准入。

**升级检查：** 在已安装的压缩实现中检查那些公开函数，看是否保留了批次与摘要输入预算。使用
一对大于最近 token 目标的工具调用，以及大于摘要模型输入窗口的用户文本，来评估准入，而不是
作为“这些限制必须持续存在”的 CI 断言。不要用宿主重试或压缩默认值覆盖来补偿。

## 不是临时 SDK 绕过方案的项（Not temporary SDK workarounds）

项目服务器批准、私有的项目/聊天覆盖、确认策略、凭据遮蔽、传输线能力检查与 React 呈现都是
ThinkRail 的刻意行为。它们的所有者是 [[submodule-server-agent-mcp]] 与
[[submodule-server-agent]]。pi 升级不会静默地移除或弱化它们。复用上游策略是一个独立的
兼容性/产品决策，而不是假定名称相似的 API 具有相同语义。
