# pi-thinkrail-workflow —— 提供工作流系统的 pi 扩展

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

`pi-thinkrail-workflow` 是一个 pi 扩展，提供 ThinkRail 的**工作流系统** ——
一组把 agent *如何执行一项工作*固化下来的 skill。（对照：`pi-spec-graph` 定义规格模型*是什么*；
`visualize` 是一个渲染工具。）它恰好贡献两样东西，由 `package.json` 的 pi manifest 接线
（`pi: { extensions: ["./index.ts"], skills: ["./skills"] }`）：

- **`index.ts`** —— 一个 `ExtensionFactory`，注册一条始终启用的 `before_agent_start` 规则：
  在一项新工作开始时，把项目上手、PR 生命周期工作，以及每一处对产品行为产生的改动
  （一个特性、一处行为变更或一个缺陷修复）送到根路由 skill（`choosing-a-workflow`）；
  入口模型及其理由见 [[submodule-workflow-skills]]（规则 4）。已经路由的工作恢复其活动工作流；
  不改变任何行为的工作直接继续，既不加载也不声明工作流。
- **`skills/`** —— 工作流 skill 家族：对符合工作流条件的工作进行分类的根路由，
  以及从它可达的 worker 与概念 skill。**权威名册是 [[submodule-workflow-skills]] 中的家族表**，
  与系统的设计 —— 概念模型、skill 角色、元规则、逐 skill 理由 —— 并列。本规格自身不保留名册
  （第二份列表会漂移 —— 本条目替换掉的那份部分枚举已经过期，漏掉了概念 skill），
  只在某个包级决策针对某个具体 skill 时才点名它（上文的根路由；Boundary 下的撰写检查清单）。

本包通过在 `skills/<name>/` 下新增子模块来成长（若将来需要新工具，它会放在 `tools/`
子模块下）；添加下一个 skill 既不改变布局，也不改变系统的形状
（[[submodule-workflow-skills]] 中的元规则 12）。

## 知识投递（Knowledge delivery）

机制与 `pi-spec-graph` 相同（[[module-spec-graph]]）：每个工作流住在自己的 skill 里，
通过 `pi.skills` manifest / `additionalSkillPaths` 自动发现。`before_agent_start` 规则
镜像 `pi-spec-graph` 的 `SPEC_RULE` —— 包括其机制：处理器就地修改
`pi-thinkrail-workflow` 标签下的 `systemPromptOptions.sections` 并且不返回任何内容，
因此 pi 只修补那一个区段，而不是强制整段提示词（[[module-spec-graph]] 拥有该理由）。
简短且字节稳定，因此它在每次运行时都能通过而不扰动 provider 的提示词缓存，
而且它是一个指针，不是重述 —— 它只在新工作开始时生效，已路由的续作则恢复其活动工作流，
在该工作流的文本不再处于上下文时重新读取它的 skill（压缩会丢弃已加载的 skill 正文；
宿主的 `skillCarryover` 会在预算内重新附加它们，所以这里是正文被截断或丢弃时的兜底）。
路由规则只写一次，在路由 skill 里；每个工作流的步骤只写一次，在它自己的 skill 里。
setting-up-a-project 家族本身不携带规则 —— 根路由把上手工作交给 dispatcher
（其 `description` 也会自触发），而在应用内，欢迎屏幕的 "Set up project" 卡片
会种入 `/skill:setting-up-a-project` 命令 —— pi 的 skill 命令语法，它**强制**
dispatcher 加载，而不是依赖描述匹配（见 [[module-web]]）。

## 边界（Boundary）

- **允许的依赖：** `@earendil-works/pi-coding-agent`（**仅类型** ——
  `ExtensionAPI`/`ExtensionFactory`），作为 `peerDependency`。没有 `typebox`：
  本包不注册自定义工具，只有一条 `before_agent_start` 规则和 skill 内容。
- **禁止：** 任何 `@thinkrail/*` 包、`apps/web`、`packages/server` 内部 ——
  只通过工具*名称*（`ask_user_question`、`spec_*`）触达，绝不通过 import。
- **工作区内部，不再绑定到仅宿主可用的工具。** skill 内容只通过工具*名称*触达能力：
  来自可移植 `pi-spec-graph` 的 `spec_*`，以及来自可移植 `@thinkrail/pi-ask-user-question`
  包的 `ask_user_question`（[[module-pi-ask-user-question]]）—— 由
  [[module-ext-ask-user-question]] 组合进 ThinkRail，也可以从 npm 装进原版 `pi`。
  当该工具不存在时（在 Settings → Extensions 中关闭，或从未安装），
  `asking-user-questions` 把这一轮问答作为普通聊天消息发出并结束轮次，
  因此没有工作流会阻塞在它上面。本包仍然不声称能在原版 `pi` 下运行 ——
  它未发布，只由 ThinkRail 暂存 —— 并且它保持独立包而不折进 `packages/server`，
  理由与 `packages/shared` 不折进 `server` 相同：一个 `SKILL.md` 与 WS/会话层没有运行时耦合 ——
  它只需要一个交给 `additionalSkillPaths` 的路径。
- **撰写检查清单随产品一起交付 —— 这是刻意的。** `writing-workflow-skills` 面向开发
  （它编辑本包），却像家族其他成员一样被暂存进每个 ThinkRail 项目：排除某一个 skill
  会让暂存路径变复杂而收益甚微，且它的触发条件很窄。该 skill 正文携带相应的工作区守卫 ——
  当 `packages/pi-thinkrail-workflow` 不在工作区中时（即 ThinkRail 管理的项目，
  这些 skill 在此是只读的暂存缓存），它用一行说明这一点并停止；
  家族只在它的源头，即 thinkrail 仓库，扩展。

## thinkrail 集成（thinkrail integration）

`packages/server/src/agent/extensions.ts` 以与 `pi-spec-graph` 相同的方式添加本包：
`require.resolve("pi-thinkrail-workflow/index.ts")` 放入 `additionalExtensionPaths`，
其 `skills/` 目录放入 `additionalSkillPaths`。

## 测试（Testing）

`index.test.ts` 钉住本包唯一的运行时行为：工厂注册的 `before_agent_start` 处理器
把 `WORKFLOW_RULE` 放进 `systemPromptOptions.sections` 并且不返回任何内容 ——
对由用户提示词启动的运行和由自定义消息启动的运行都一样（pi 1.2 对两者都发出该事件，
以 `messageType` 区分）—— 而 pi 自己的渲染器（`buildSystemPromptSections` /
`buildSystemPrompt`）把这些被修改的选项变成一个 `<pi-thinkrail-workflow>` 块，
排在 `cwd` 之后并结束提示词 —— 即旧版自由文本追加所在的尾部位置。pi 不从其包根或
`core/index.ts` 重新导出该渲染器（只有它的
`BuildSystemPromptOptions`/`NormalizedBuildSystemPromptOptions` 类型是根导出的），
所以测试通过解析出的路径触达 `dist/core/system-prompt.js`；一个挪动它的 pi
会大声地破坏该 import，而不是悄悄削弱断言。选项字面量的类型是
`NormalizedBuildSystemPromptOptions`，所以形状变化就是编译错误。该规则的*措辞*是散文，
不是契约，因此不被钉住。

skill 行为由 **workflow-test harness** 无头测试 —— 设计、判定模型、套件与覆盖率见
[[module-workflow-tests]]（`bun run test:workflows`；按需运行 —— 需要 pi 鉴权、
消耗真实 token，绝不是提交/CI 门禁）。逐 skill 的观测状态见家族表
（[[submodule-workflow-skills]]）。切片 3（worker 流程端到端）已部分落地：
`importing-a-codebase` 在 harness 的 importing 套件中运行（采纳 + 回归场景，
随文档采纳工作一起加入）。剩余的后续工作是只写场景定义，不新增任何机制：
通过用户模拟器跑 `starting-a-new-project` / `brainstorming` 的完整运行
（设计记录：[[module-workflow-tests]]）。

import 分支还通过一个带标签的 `@agent` 浏览器 e2e **经应用**得到覆盖
（`e2e/setting-up-a-project.live.spec.ts`）：它把一个工作区 worktree 变成纯代码项目，
驱动欢迎卡片的确切 `/skill:setting-up-a-project` 命令，并断言该流程起草了
`goal-and-requirements.md`（在 Specs 侧栏中渲染）—— 证明按钮的 `/skill:` 种入
在 `session.prompt` 路径上驱动了该流程，而这是任何无头场景都没有覆盖的。

## 非目标（Non-goals）

- 一个原版 pi 可移植的工作流包 —— 那需要用一个最低公共分母的提问机制替换
  `ask_user_question`；对一个只属于 thinkrail 的宿主特性不值得（见上文边界）。
- 工作流的任何运行时/引擎层 —— 见 [[submodule-workflow-skills]] 中的 "no runtime machinery"。
- 重实现旧 thinkrail 的 `claude-plugin` **工单/看板工程工作流**（它的 ticket-orchestrator、
  ticket-implement、bug-fix、spec-review 等 skill —— thinkrail 团队用来构建 thinkrail 本身）。
  那是用于另一个基于工单的产品、且这里没有看板或工单系统可运行的开发工具链。
  移植单个*面向产品*的 skill 仍然没问题 —— `setting-up-a-project` 就是这样来的
  （[[submodule-workflow-skills]]）。
