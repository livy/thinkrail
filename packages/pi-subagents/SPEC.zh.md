# pi-subagents —— 委托核心之上的 Agent 工具

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

`pi-subagents` 是 **[[module-pi-delegation]] 的第一个消费者**，也是该框架可扩展性的证明：
面向 LLM 的 subagent 能力。它只拥有 subagent 专属的东西 —— `Agent` + `get_subagent_result`
工具、agent 定义（发现 / 优先级 / 信任）、定义 → spawn 的映射，以及后台完成投递。
它包含**零私有子会话装配代码**：一切与会话形状相关的东西都经核心的 `SessionOptions`。
每个选择是如何定下来的：见下方的决策日志。

## 公开接口：barrel，`index.ts`（Public surface (the barrel, `index.ts`)）

- **default export** —— 零配置的纯 pi 扩展入口（`pi` manifest 指向这里）；
  它用默认绑定构建自己的 `DelegationService`，在 spawn 时从工具 `ExtensionContext`
  投影出父级。投影包含公开的 `modelRegistry`，使核心自建的运行时能在每次子会话 spawn 之前
  镜像其他扩展所做的 provider 注册，而不检视它们的配置，也不依赖 pi 的私有运行时字段。
  它还携带 `isProjectTrusted`，因此纯 pi 子会话继承其所在会话的项目信任，
  而不是 pi 的"默认受信"。
  **回退服务的生命周期是会话作用域的**：在 `session_shutdown` 时扩展会 await
  `disposeChildrenOf(parent)` 并丢弃该服务，因此原版 pi 的后台子会话绝不会比其父会话
  活得更久、白白消耗 token 却拿到无法投递的完成（PR #302 评审发现）。嵌入方注入的服务
  刻意不被触碰 —— 它的生命周期属于嵌入方（ThinkRail 在 `removeSession` 中级联）。
  默认工厂与 legacy 工厂按每次工厂调用来拥有自己的完成生命周期：重载结束该生命周期，
  而父轮次中止不会。迟到的工具更新会被 Pi 在工具返回后丢弃。
- `createSubagentsExtension(options?)` —— 兼容的、runner 作用域的扩展工厂。选项包括
  `service?`、`delegationRoot?`、`scope?`、`isEnabled?` 和 `canDeliverCompletion?`。
- `createSubagents(options: SubagentsExtensionOptions & { service: DelegationService })` ——
  一个显式保留的、可移植的父级所有者，只暴露 `extension`、`flushCompletions({ atTurnEnd? })`
  和同步的 `dispose()`。它注入的服务仍归嵌入方所有。一个不可变的父级身份在首次
  `session_start` 时确立；用该所有者服务另一个父级会被拒绝。当前发送者、待处理完成认领、
  flush 守卫与永久关闭标志都住在被重新调用的扩展回调之外。重载关闭只以身份安全的方式解绑
  旧的发送者；下一次 `session_start` 绑定新的发送者并 flush 保留的结果。
  其他关闭原因与显式处置会永久关闭投递、清除待处理认领并忽略未来的结果；
  重新绑定无法重新打开它。嵌入方必须显式处置该所有者，因为 Pi 的
  `AgentSession.dispose()` 不发出扩展关闭事件。前台错误详情暂存仍是 runner 本地的。
- `isEnabled` 独立于完成投递控制工具可用性与启动。它默认为启用，为 false 时在
  `session_start` 时把两个工具从活动集合中移除，并在子会话装配之前、异步准备之后
  以及 `runQueued` 之前即刻重新检查。迟到的禁用会在拒绝之前处置那个未启动的子会话。
  `canDeliverCompletion` 默认为 true，临时把会话记录追加与自动唤醒都门控住，
  而不关闭所有者或丢失待处理结果。嵌入方在临时删除回滚或注册变得可寻址之后执行 flush。
- `SUBAGENT_COMPLETION_MESSAGE` —— web 的后台结果行据以取键的自定义消息类型；
  `SubagentCompletionDetails`（该运行的 `DelegationRunDetails` 加
  `delivery: CompletionDelivery` —— `wake` | `run` | `appended`，见结果投递），
  镜像进 `@thinkrail/contracts`。
- 定义：`AgentDefinition`、`discoverAgentDefinitions`、`parseAgentDefinition`、`BUILTIN_AGENTS`。
- 映射：`toSpawnMapping`、`resolveModelRef`、`buildChildSystemPrompt`、`RECURSION_GUARD_TOOLS`。

## 工具（The tools）

| 工具 | 行为 |
| --- | --- |
| `Agent({ subagent_type, task, model?, run_in_background? })` | 在每个父模型轮次之前快照定义（会话中可编辑），然后把点名的定义加上可选的调用模型映射为 `SessionOptions`，并通过 `createChild` + `runQueued` 启动；description 与执行闭包使用同一份快照，因此会话中的编辑无法让对外宣称的模型策略与实际执行不一致。模型优先级是定义钉定 → 调用模型 → 实时父级；定义钉定会静默胜出而覆盖提供的调用值，可用的 agent 描述会在使用前公布每个定义的模型钉定或 call/parent 选择以及任何显式 thinking 策略。描述还声明 subagent 无法调用 MCP 工具 —— 子会话从不加载 MCP 引擎（精选子集里没有，pi 和 ThinkRail 中都没有），所以父级自己调用并把结果放进任务里 —— 以及后台报告会自行到达，因此模型结束自己的轮次来等待，而不是轮询或休眠。前台：await 结果并随工具信号走；`error` 结果会抛出（工具错误，原因在前）—— 而该错误工具结果**仍然携带本次运行的最终 `details`**：pi 会把抛出的工具错误结果替换为 `{content, details: {}}`，因此扩展在抛出之前按 `toolCallId` 暂存结果详情，并通过 `tool_result` 钩子覆盖把它们重新注入（该暂存在 `turn_end` —— 定稿之后 —— 与 `session_shutdown` 时被清扫，因此在工具定稿之前被中止的轮次不会留下孤立条目）；失败运行的卡片保留其子会话 id，会话记录仍可打开（PR #304 评审发现）。后台：**绝不随父轮次的中止信号**（分离式运行能在父级中止后存活 —— 核心规格语义，有测试钉住）；记录 `info.detached`，使嵌入方的资源摘要能把它与前台运行区分开；立即返回 `{childSessionId}` 文本；终态结果变成一条 `subagent-completion` 自定义消息，按下文结果投递规则投递一次。实时 `onUpdate` 详情流入 `partialResult`（替换）。结果有界至 50k 字符 —— 完整文本留在子会话记录里。 |
| `get_subagent_result({ session_id })` | **检查血缘**：若某个子会话的 `record.parentSessionId` 不是调用方会话，则走未知 id 的错误路径 —— 在共享（工作区作用域）服务下，一个标签页不得读取或标记收集另一个父级的子会话（PR #302 评审发现）。通过 `findChild` + `collectResult` 读取核心 registry：终态 → 最终文本 + 详情，经与前台结果**相同的、原因在前、50k 有界**的整形（标记为已收集，从而抑制待处理推送 —— 结果投递；出错的运行先报告其 `errorMessage` —— 核心决策 #24 —— 被中止的运行报告其状态与原因，因此拉取绝不会隐藏推送本会说出的话）；运行中 → 状态快照，其文本说明报告会自行到达；未知 id → 错误，点名重启丢失的情形 + 派生出的会话记录路径。 |

两个工具都在 `session_start`（由 `bindExtensions` 发出）内注册。`Agent` 还会在首个
provider 轮次的 `before_agent_start` 中、以及在每个 `turn_end` 为下一轮次重新注册；
pi 会立即替换同名工具，因此每一轮次都获得一份定义快照，同时用于其 description 与执行，
而编辑仍然实时生效，无需 `/reload`。当嵌入方停用这些工具时注册保持不变：
同一会话内重新启用与分离式完成投递都不需要扩展重载或替换。

## 结果投递（Result delivery）

分离式结果在**父级的下一步、恰好一次**到达父级 —— 经由先到的那条通道：
推送（`subagent-completion`）或拉取（`get_subagent_result`）。所有者自己持有结果
（仅由分离式 `run.then` 续作加入的 `pending` 集合 —— 不扫描子会话，没有第二份会话记录
或 registry），并且绝不把它交给 pi 的 steer/follow-up 队列：pi 只在运行本将停止时才排空
follow-up，不提供逐消息出队，且中止后仍保留已排队消息，这就是拉取到的结果会在几分钟到
几小时后以"已处理"轮次回来的原因（2026-10 日志样本中占 89 % 的完成）。投递点：

- **忙碌的父级，在步骤边界** —— 一个合格的 `turn_end`（assistant `stop`/`toolUse`，
  信号未被中止；`length`/`error` 轮次留给 pi 的恢复）或 `agent_before_settle`：
  每个被持有的结果都用 `sendMessage({ triggerTurn: false })` 发进 pi 的待处理自定义列表，
  pi 会在该轮次的工具结果之后、下一个请求之前追加它，而该边界在有一个结果应当唤醒父级时
  返回 `continue`。`turn_end` 监听器打开一个门禁的嵌入方在那里调用
  `flushCompletions({ atTurnEnd: true })`；以该方式投递的唤醒在下一个
  `agent_before_settle` 时被请求，除非更晚的轮次已经携带了它。
- **空闲的父级**（随发送者绑定的 `ctx.isIdle()` 探测；探测过期时保持不变）：
  被持有的结果作为**单批**发出，最后一个唤醒型结果最后发送并单独携带 `triggerTurn`，
  这样一个批次最多启动一次运行，而触发发送失败会把唤醒留给一条仍待处理的结果。
  这覆盖到达、在 `agent_settled` 之后一个宏任务调度的 flush（在宿主的轮次收尾器之后）、
  在 `session_compact`/`session_compact_failed`/`session_tree` 之后（pi 在这些事件内部
  仍算作压缩中），以及嵌入方的 `flushCompletions()`。父级忙碌时的宿主 flush 什么也不做；
  边界会投递。
- **拉取认领推送**：每个投递点都会丢弃其子会话快照为 `collected` 的结果。

`delivery` 记录发生了什么，供 web 的轮次布局使用：`wake` —— 该结果启动了父级的运行
（唤醒批次中的第一个）；`run` —— 它加入了已经在进行的运行；`appended` —— 它被显示出来
但没有唤醒任何人。消息整形保持原因在前、50k 有界。每次发送都在同步的 Pi 接受之前被认领，
可重入的 flush 无法复制它，而同步发送失败会恢复该认领，除非发生了处置；
恰好一次意味着 Pi 的即发即弃 API 的同步接受，而不是最终的 provider 响应。
不涉及宿主的完成队列。

## 定义：发现、优先级、信任（Definitions: discovery, precedence, trust）

按信任顺序先到者胜 —— **builtins → personal（`<agentDir>/agents/*.md`）→ project
（`<cwd>/.pi/agents/*.md`、`<cwd>/.agents/agents/*.md`）** —— 因此 worktree 定义
永远不会遮蔽内置或 personal 名称（下文决策 6），而项目定义只在 `ctx.isProjectTrusted()`
时才加载。内置项是 **TS 常量**（`builtins.ts`：scout / planner / worker / reviewer ——
用户定案），不是 `.md` 文件，因此它们能在 `bun build --compile` 后存活并获得类型检查；
只有 `worker` 能修改文件，且它的提示词禁止历史/索引类 git 操作
（`commit`/`add`/`stash`/`reset`/`push`）：子会话共享父级的 worktree 与分支，
因此自我提交会切断父级历史的连续性，并使工作与跟踪它的东西脱钩
（在 ThinkRail 里是按计划步骤的提交映射）；提交归父级/宿主所有，git 读取保持开放。
用户撰写的定义沿用社区的 `.md` + frontmatter 约定
（`name`、`description`、`tools`、`model`、`thinking`、`max_turns`、
`inherit_project_context`、`skills`、`extensions`；正文 = 系统提示词）。格式错误的文件
被跳过，绝不致命。四个内置项都设置 `extensions: true`（嵌入方的精选子会话集合 ——
在纯 pi 零配置下不生效），其中只读角色把规格读取工具
（`spec_grep`/`spec_get`/`spec_graph`）列入允许列表，scout 则列入 web 工具
（`web_search`/`fetch_content`）。只有 reviewer 选择加入项目上下文：它的判断取决于仓库指引，
而它的提示词只承认实质性、可达、未被缓解且归因于具名目标的发现。
它必须证伪候选发现、补全目标、按根因去重，并以一个明确裁决结尾；
GitHub review 与线程语义不在这个可移植角色之内。

## 映射 —— 策略，而非机制（The mapping (policy, not mechanism)）

`toSpawnMapping`：定义 + 可选的调用模型 → `SessionOptions` + `RunOptions.maxTurns`。
生效的引用遵循定义钉定 → 调用模型 → 父级继承。钉定会丢弃调用引用（包括其 thinking 后缀），
且不解析它。引用按会话可用的模型解析（`provider/id` → 精确 id → 唯一前缀）；
对未解析的选择不允许 provider 替换、自定义模型合成或父级回退。

定义与调用模型引用都接受末尾一个 `:<thinking>` 后缀：`off`、`minimal`、`low`、`medium`、
`high`、`xhigh` 或 `max`。整串引用解析总是先行：既有含冒号的字面 id 与唯一前缀保持其含义，
歧义直接失败而不会去尝试剥离后的引用。只有整串引用缺失时才允许后缀解析；其非空基底必须
按同样的规则解析，且不做递归剥离。未知后缀保持为未解析的字面引用，
给出语法提示而不是静默采用默认 thinking。缺失与歧义的选择有不同的诊断；
只有在歧义时才建议 provider 限定。导出的 `resolveModelRef` 仍然只做身份解析，
不解释后缀。

thinking 优先级是显式定义 `thinking` → 胜出引用上的后缀 → 实时父级。
Pi 会把请求的等级钳制到所选模型的能力；扩展既不复制该策略，也不改变父级状态。
这是语法兼容，而不是采纳 Pi 的 CLI 解析器：它的模糊平局裁决与自定义 ID 回退会违反本包的
歧义/可用性契约。逐次调用的独立 thinking 参数不在本接口之中。

子会话系统提示词 = 定义正文 → subagent 桥接 → env 块（cwd、git 分支、平台）——
**稳定材料在前**，以复用 KV 缓存前缀。递归守卫
（`excludeTools: ["Agent", "get_subagent_result"]`）是无条件的。
`extensions: true` 透传给核心的精选集合选择加入（核心决策 #25）—— 该集合包含哪些扩展
是**嵌入方**的选择，绝不是本包的。

## 边界（Boundary）

- **允许的依赖：** `pi-delegation`（仅经其 barrel）；pi SDK + `typebox` 作为
  **peerDependencies**；`node:*`。
- **禁止：** 任何 `@thinkrail/*` 包（必须在原版 pi 下可用）；`@earendil-works/pi-tui`
  （纯 pi 门槛是默认渲染）；`pi-delegation/src/*` 内部。
- web 卡片 / 会话记录视图是*展示*面，通过工具名称连接
  （`registerToolRenderer("Agent", …)`）—— 它们住在 `apps/web`（`chat/tools/subagent`），
  绝不在这里。

## 用户停止（User stop）

分离式完成使用 [[module-pi-delegation]] 中最终结果的 `details.abortReason`，
而不是第二份已停止子会话 registry 或 UI 生成的消息。当它是 `"user"` 时，
该结果被投递而不唤醒父级（`triggerTurn: false`，边界不返回 `continue`）：
它既不唤醒空闲父级，也不中止工作中的父级，也不禁用后续委托。

停止**父级**对当时被持有的结果同理：被中止的 `agent_settled` 把它们标记为非唤醒，
在门禁允许后显示一次（`appended`），该标记会经受住门禁关闭或一次重载，
直到下一次被接受的用户提示词。停止之后完成的结果照常唤醒父级
（用户定案，2026-10：Stop 停止当前运行，而不是停止后续结果）。
provider 错误结束一次运行之后，被持有的结果会通过 `agent_before_settle` 一起唤醒它一次。
永久会话关闭完全抑制投递；保留所有者的重载只是暂停投递。
原版 Pi 的行为不变，除非其调用方显式提供用户原因。

## 验证（Verification）

包内单元套件（`bun test`）钉住字面 ID/后缀优先级、歧义、定义 thinking、Pi 钳制的子会话
thinking、父级隔离与子会话创建之前的失败，包括一个双扩展回归：其中一个扩展注册合成的
provider，零配置 subagents 扩展经它委托。公开 Pi 会话测试钉住门控的重放、真实的重载间隙、
永久关闭、同步发送失败、可重入 flush 与默认/legacy runner 隔离，以及结果投递：
拉取过的结果绝不再推送（跨 Stop 与下一次提示词亦然）、单次运行内工具与最终轮次上的
下一步投递、一个请求中的多个结果、每个空闲批次一次运行、父级停止后不再唤醒、
provider 错误后只唤醒一次。端到端：`e2e/subagents.live.spec.ts`（`@agent` —— 真实宿主
驱动真实子会话：前台扇出、后台完成、会话记录读取）。核心规格的纯 pi 门槛：
**`bun run smoke:subagents`** —— 该扩展带默认绑定，在仓库钉定的原版 pi CLI 下，
于隔离的一次性 agent 目录中运行；仅按需运行（需要 pi 鉴权、消耗真实 token），
绝不是提交/CI 门禁。

## 决策日志 —— 与用户共同定案，2026-08（Decision log (settled with the user, 2026-08)）

0. **自建而非采纳第三方。** 第三方的 pi subagent 扩展被调研过两次，也尝试过一次采纳并回滚；
   被调研的选项反复与我方多会话内嵌宿主、编译二进制和 web 渲染器不匹配
   （其中一些用子进程 spawn、进程级 registry/发现、TUI 管理）—— 尽管其中两个证明的进程内
   spawn 模式（pi SDK 作为 peerDependency）正是核心复用的（核心决策 #17）。
   我方的按会话发现修复以上游 tintinweb/pi-subagents PR #223 的形式存活
   （已不再是依赖）。
1. **可移植的工作区包** —— 一个消费 `pi-delegation` 的纯 pi 扩展（pi SDK 作为
   `peerDependency`）。ThinkRail 的服务端按会话嵌入该工厂，把宿主绑定的
   `DelegationService` 交给它；在原版 pi 下该扩展用默认绑定构建服务。
   它必须在纯 pi 中加载并工作（核心规格的纯 pi 门槛）。*取代原先"宿主拥有的捆绑扩展"
   决策 —— 在 PR #261 评审轮中反转；调研 + 理由：核心决策日志 #17–21。*
2. **工具命名：Claude Code 风格** —— `Agent`（spawn；模型受过该约定的训练）+
   `get_subagent_result`（收集分离式结果）。没有 `steer_subagent`。
3. **范围：前台 + 并行扇出 + 后台运行。** 并行扇出 = 一条消息中的多个 `Agent` 调用
   （pi 并发执行一个批次的工具调用；核心的信号量给它们配速）—— 没有 `tasks[]`/链式 DSL，
   模型自己排布链的顺序。
4. **会话记录持久化，随时可打开** —— 子会话是磁盘上隐藏的 pi 会话；web 卡片链到一个
   只读的会话记录视图，它在运行期间、完成之后、以及宿主重启之后都可用。
   后台完成投递（上述结果投递与用户停止策略）在宿主重启时丢失，并在其会话进入关闭时
   被抑制 —— 接受；会话记录无论如何都存活。
5. **内置 agent：一个小的精选集合**，为 ThinkRail 的规格优先工作流调校
   （名册 + TS 常量形式：见上文定义）；personal + worktree 定义则进一步扩充。
6. **优先级/信任：worktree 定义绝不能遮蔽内置或 personal 名称**（对应 skill 的不变量；
   社区库的顺序相反 —— 那造就了"仓库自带 `Explore.md`"攻击）。worktree 定义沿用项目既有的
   信任姿态（与 pi 原生 `.pi` 资源相同）；没有单独的准入门禁（更早的用户决策）。
7. **子会话上下文：默认窄，按定义选择加入。** 子会话看到定义正文 + env 块
   （内容：上文映射）+ 任务；选择加入项：`inherit_project_context`（worktree AGENTS.md）、
   `skills:`（显式列表）。没有父会话继承；fork 模式继承是记录在案的成长路径
   （核心 `origin: fork`）。依据（2026-08，源码核实）：参考实现分为默认窄 + 选择加入
   （nicobailon —— 采纳最多，3.1k★ / 56k dl/wk —— gotgenes-replace、tintinweb）
   与项目感知（pi example、gotgenes-append）两类；nicobailon 的窄模型被选为多模式系统
   最干净的基底，而 gotgenes 的 KV 缓存提示词排序（稳定材料在前）被采纳进我方自己的布局。
8. **逐次调用的模型是未钉定定义的回退。** `Agent.model` 让父级在定义没有 `model` 时，
   从其保留的运行时世代中选择任一明确的可用模型。定义钉定会静默胜出而覆盖提供的调用值
   （用户定案），而不是失败或变成可覆盖；工具的可选 agent 列表暴露该钉定，使优先级可见。
   定义发现在每个父模型轮次做快照，防止实时编辑把公布的钉定与执行闭包拆开。
   两个来源都未设置时，核心保留父模型继承。
9. **分离式结果由所有者持有，并在下一步投递**（2026-10，经跨 provider 设计评审后
   用户定案）。follow-up 投递让模型能在运行期间轮询，然后在运行结束时（或中止后的
   下一次提示词之后）再次投递已拉取的结果。被否决的方案：只在运行结束时去重
   （模型会继续用 `sleep` 轮询）、在 `turn_end` 时 steering（一次一条的通道，能经受中止，
   但留下拉取窗口）、边界 `custom_message` 草稿（任何扩展的草稿非法时 pi 会丢掉整条链，
   因此一个已认领的结果可能消失）、纯提示词指引与仅在 web 隐藏
   （多出的轮次与上下文增长仍在）。
