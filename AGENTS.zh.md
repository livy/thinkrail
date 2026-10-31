# ThinkRail

> 中文译文，原文见 [`AGENTS.md`](./AGENTS.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

越用越好的 agentic IDE，围绕 `pi` 编程 agent 构建。应用是一个薄宿主，运行 `pi` 并将其桥接到
富交互 UI；`pi` 拥有模型、skill、压缩、成本与会话状态。

## 按比例读取上下文（Read context proportionally）

- 使用 `goal-and-requirements.md` 了解产品的目标、原则、能力与非目标。
- 使用 `architecture.md` 了解系统拓扑、跨模块决策与仓库级不变量。
- 进行 pi 升级，或为 pi 添加宿主侧绕过方案时，阅读 `pi-compatibility.md`：针对已安装的 SDK
  逐一核对每条适用的移除门禁，更新登记表，并报告移除了什么、保留了什么。
- 当工作受某个模块边界、契约、不变量、已记录行为或架构决策约束，或可能改变它们时，阅读其
  所属的 `SPEC.md`。
- 局部改动不需要阅读无关的规格，也不需要完整的仓库地图。

## 模块结构与边界（Module structure and boundaries）

清晰、分形的模块边界是最高优先级的要求：

- 每个包以及每个有意义的目录级子模块都有一个 `SPEC.md`，写明其职责、公开接口、允许的依赖
  与禁止的触达。
- 子模块只以 `index.ts` barrel 作为其唯一的公开接口；同级模块通过 barrel 导入，绝不导入内部
  实现。只有当 barrel 会破坏代码分割或违背库约定时，才保留逐文件导入，例如
  `apps/web/src/panels` 与 `packages/ui`。
- 同级子模块之间的依赖边记录在父模块的 `SPEC.md` 中，而不是每个叶子模块里。
- 移动或模糊边界的改动要先更新所属规格。在可行时用测试覆盖公开接口与边界规则，但不要为
  局部改动强行制造覆盖率。

## 引擎与架构（Engine and architecture）

- 通过 `@earendil-works/pi-coding-agent`（`createAgentSession`）在进程内运行 `pi`，绝不作为
  子进程运行，也绝不通过第二个 agent 运行时运行。致命的 provider/agent 故障可能导致宿主
  崩溃；这种缺乏崩溃隔离的状态是被接受的。
- 三个环是引擎宿主（`packages/server` + `packages/shared`）、带类型的传输线
  （`packages/contracts`）与可独立发布的 UI（`apps/web`）。`apps/cli` 与 `apps/desktop` 是
  同一宿主之上的薄启动器。
- 只使用 `@earendil-works/*` 包作用域。`@mariozechner/*` 已废弃。

## 仓库级不变量（Repo-wide invariants）

- `apps/web` 只依赖 `contracts`、`locales`、`ui`、`extension-api/web`、`resource-renderers`
  和 `thinkrail-extensions/*/web`，绝不依赖 `server` 或 `shared`。
- 面向用户的 UI 文本是 `packages/locales/en.json` 中的一个 key，通过
  `@thinkrail/locales/react` 渲染；lint 门禁会拒绝硬编码的 JSX 文本与文本属性。
- 绝不要在浏览器打包的代码中对 `pi` 做值导入。只从 `pi-ai` / `pi-agent-core` 包根导入类型；
  `@earendil-works/pi-coding-agent` 仅限服务端。
- 单一 id 模型：UI 标签页 id 与 `session.sessionId`；不存在单独的 pi UUID。
- `pi` 拥有状态。宿主暴露 pi 报告的内容，而不重新计算成本、统计或状态。
- 流式语义：`text_delta` / `thinking_delta` 追加；`tool_execution_update.partialResult`
  替换。
- 会话正在流式输出时 `prompt()` 会抛错；请使用 `steer()` / `followUp()`。错误通过事件流与
  方法抛出的结果向外传递，而不要视作崩溃信号。
- 自动工作结束于 `agent_settled`，绝不是 `agent_end`；尝试级别的 `agent_end` 之后还可能
  跟着重试、压缩、恢复或排队的续跑。
- UI 面板与布局无关；由 shell 负责排布。
- 错误无条件上报到 PostHog Error Tracking，但只作为分类信息——类型、`code`、来源/作用域/
  请求方法，以及仅属于 ThinkRail 的栈帧——绝不包含 `message` 文本。值得区分判别的失败应
  赋予 `CodedError` 的 code，而不要依赖其 message。携带错误的宿主诊断走 `logger(scope)`，
  不走 `console.*`。允许列表位于 `packages/server/src/analytics/SPEC.md`。
- 传输层的宿主端点是参数，默认为同源；`server.welcome` 携带协议版本。

### Web UI 上下文（Web UI context）

做 Web UI 相关工作时，阅读 `apps/web/SPEC.md` 及所属子模块规格。样式使用 Tailwind v4 工具类，
映射到生成的语义化 CSS 变量 token：绝不使用内联 style 对象、裸十六进制色值、内部调色板名称
或未知的 token 工具类。配色相关工作阅读 `apps/web/src/styles/COLOR.md`，间距相关工作阅读
`apps/web/src/styles/SPACING.md`。图标使用 `@remixicon/react`（默认 Line，激活时 Fill），
基础组件使用 `@thinkrail/ui/<primitive>` 中自有的 shadcn/Radix 原语；`cn()` 位于
`@thinkrail/ui/utils`。这些规则同样适用于 `packages/ui`、`packages/resource-renderers` 以及
扩展的 web 部分。

做对话渲染或工具呈现时，阅读 `apps/web/src/chat/SPEC.md`。展示型渲染器始终由 props 驱动；
只有 `ChatView` 集成 store 与传输层。服务端能力与其 UI 渲染器通过工具名经
`registerToolRenderer` 关联；未注册的工具使用默认渲染器。

## 规格与注释（Specs and comments）

规格是意图、决策、不变量、权衡与复盘的持久归宿。保持简洁，避免复述代码或另一份规格。
注释趋近于零：lint/type 指令和极少数一行式的危险提示是可接受的；理由与叙述属于所属规格。

## 验证（Verification）

本地测试使用一次性夹具，不具备生产环境访问权限。运行受影响的测试，修复由所请求改动导致的
失败，并在每一步重跑，无需逐次请求批准。

- 用最小的相关单元或聚焦的 E2E 目标迭代。
- 对于已发布应用行为或集成改动，在合并后的实现完成之后、最终交接或提 PR 之前，运行一次
  完整的无 agent 浏览器套件（`bun run e2e`）——而不是每完成一个 TODO 项或每次提交都跑一次。
- 当改动触及真实 agent 行为时，还要运行覆盖它的聚焦 live 规格，例如
  `bun run e2e:full e2e/agent-naming.live.spec.ts`；这不需要批准。运行完整的
  `bun run e2e:full` 套件前先询问用户：它会消耗真实的 provider token，并且耗时多得多。
- 仅涉及文档/规格的改动与仅涉及测试夹具的改动使用针对性检查，除非它们可能影响已发布的
  运行时。
- 快速门禁：`bun run check:deps`、`bun run check:boundaries`、`bun run check:seams`、
  `bun run lint` 与 `bun run typecheck`。单元测试是 `bun run test`；
  `bun run check:spec-surface` 校验已登记的 spec/barrel 公开接口；
  `bun run check:compiler` 在某个 web 函数相对已提交的普查基线新退出 React Compiler
  （或开始编译）时失败。
- `bun run test:workflows` 按需运行：它使用真实 provider token，不是提交/CI 门禁。
- 二进制与桌面产物模式有各自的门禁；改动这些产物时使用它们。

所有 runner 模式、隔离保证、凭据处理、取消行为与调试命令都在 `e2e/SPEC.md` 中；workflow
夹具细节在 `e2e/workflows/SPEC.md` 中。

## 交接卫生（Handoff hygiene）

门禁全绿是必要而不充分的条件：

- 本地交接前，审查任务范围内的工作树与提交。开启或更新 PR 前，审查相对其基线的完整分支
  diff 以及工作树。
- 对于非平凡的实现，做一轮减法：移除可避免的抽象、状态所有者、依赖、兼容层与回退路径。
  报告实质性的移除，并为保留的层给出理由；不要为局部修改增加仪式。
- 绝不要仅为让门禁通过而添加 `biome-ignore`、`@ts-expect-error`、`@ts-ignore`、
  `eslint-disable` 或 `as any`。把错误视为设计信号。如果确实需要抑制，先取得用户的明确
  批准。
- 在所审查的同一 diff 范围内审计新增的注释与抑制。不要让先前无关的分支历史污染任务局部的
  交接。
- 集中处理重复的非平凡推导。在 web 应用中，派生的 store 状态应放在 selector 中，总是同时
  发生的写入应放在一个原子 action 中。
- 替换某种模式或状态模型时，搜索每一处旧用法并迁移，或明确指出有意保留的幸存者。
- `README.md` 是面向用户的首页。对于新增、删除或重命名用户可见能力，或改变安装、分析行为
  的改动，审查其中的表述并在同一改动中更新受影响的说法；当截图所描绘的 UI 发生实质变化时，
  刷新 `.github/readme-assets/` 下的对应截图。
- 在已批准的范围内执行安全的清理。只有破坏性、超出范围或产品层面的决策才需要询问。
- PR 生命周期相关的工作使用 `shipping-a-pr` skill，包括截图要求与 PR 模板。以程序方式创建
  issue 时，复现所选的 issue 模板并传入其 frontmatter 标签。

## 技术栈（Stack）

Bun + Turbo monorepo · TypeScript strict · React 19 + Zustand + Tailwind v4 · 进程内 `pi`
（Node >= 22.19）。应用状态存放在 `~/.thinkrail` 下。

依赖锁定精确版本。横切依赖只在根 `workspaces.catalog` 中锁定一次，并通过 `catalog:` 引用；
peer 依赖与本地协议是唯一的豁免。理由由 `architecture.md` 的决策 #10 负责说明。
