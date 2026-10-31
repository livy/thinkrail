# 引擎宿主（服务端库）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

把引擎宿主作为可嵌入的库。为浏览器↔宿主 wire 提供服务（`Bun.serve` HTTP+WS、静态 SPA），
并通过 `createAgentSession` 在进程内运行 `pi` agent。由 `apps/cli` 和 Electrobun 的 `apps/desktop`
在进程内启动；它自己没有独立入口点（`dev.ts` 为开发 / e2e 启动它）。

## 边界（Boundary）

- **拥有：** HTTP+WS 服务器、静态服务、WS 派发注册表、服务端特性服务
  （project/workspace/git/fs/terminal 加以进程内的 `AgentSession` 管理器），以及 `~/.thinkrail` 持久化。
- **公开接口：** `createServer(options) → Promise<RunningServer>`
  （`{ port, startAttributionClaim, stop, shutdown }`）——保存选择的归因会等待启动器显式的
  UI-ready 调用；`stop()` 是面向低层测试的同步资源释放，而 `shutdown()` 是幂等的、
  有界的生产生命周期（收敛会话 + 排空 turn 回执/分析并释放 socket/PTY/watcher），
  每个启动器都必须 await 它——公开工厂在绑定 socket 或暴露处理器之前启动 Central 制品监视
  并应用初始的当前 PI 运行时——当配置的 Central 扩展失败时回退到 `load-failed` 状态的
  普通运行时——因此每个嵌入者都获得同一个引导不变量——以及
  `bootHost(options) → BootedHost`（进程引导包装器：安装崩溃日志、仅在启动器以
  `resolveShellEnv: true` 选择加入时才解析登录 shell 环境、应用已存储的宿主环境
  （[[submodule-server-environment]]）、在选择端口之前预热同一初始化、await `createServer`，
  并安装 SIGINT/SIGTERM 优雅关闭处理器），两者都从 `host/` 重新导出；
  加上 `registerBundledRuntime`（及其类型，从 `agent/` 重新导出）——这是一个编译后二进制接缝，
  让无法以路径方式加载内置 pi 扩展的启动器（`bun build --compile` 二进制或打包后的
  Electrobun 服务端运行时里没有 `node_modules`）把它们作为值导入的工厂 + 一个暂存 skills 目录注入
  （registry 扩展无需注入——它们的工厂通过静态 import 图抵达），注入暂存的 macOS/Windows
  操作系统回收站辅助程序路径，并注册 pi 静态打包的 provider 流程（OAuth 流程 + Bedrock 模块），
  这些流程 pi 原本通过二进制不友好的变量说明符动态 import 触达（见 agent SPEC）。仅供构建使用的
  **`@thinkrail/server/build-support`** 是内置扩展条目、skill 根、按平台的 `bun-pty` 库和回收站辅助程序
  的唯一清单，由两个启动器的打包器消费。制品夹具/断言运行器位于 [[module-artifact-tests]]，
  不在该库中。该包还暴露 **`@thinkrail/server/agent` 子路径导出**（`agent` barrel）：
  面向**无头工作流测试夹具**（`e2e/workflows/`）的服务端会话接口，它通过生产接线驱动真实的进程内会话，
  而不启动 HTTP 宿主——这是一个刻意的第二入口，避免在 node 运行的 e2e worker 下求值 `host`
  （仅 Bun：`Bun.serve`、`bun-pty`）。不供 `apps/*` 使用——web/CLI 的边界规则不变。
- **允许的依赖：** `contracts`（类型 + WS 常量）、`shared`（`shellEnv` 和 Central 适配器）、`bun-pty`、
  `@earendil-works/pi-coding-agent` + `@earendil-works/pi-ai`（运行时）、`extension-api`
  （`./server` 类型）+ 每个 `thinkrail-extensions/*` 的**仅 server 子路径**（registry 的组合边；
  宿主不再直接依赖某个扩展拥有的 pi 包）、`pino` 及其 pretty/rolling 目的地（宿主诊断）、
  `jsonc-parser`（定向共享 Pi 配置编辑）、Bun/Node。
- **部署义务：** 产品行为存在于拥有它的服务端特性模块中，并由 `host` 组合；启动器只提供引导选项和
  打包资源。当出现已证明的第二环境需要不同实现时，拥有该特性的模块定义一个窄的注入端口，
  而不是宿主级的平台适配器。
- **禁止：** import `web`/`cli`/`desktop`；被打包进浏览器；按启动器身份分支产品行为。

## 内部模块（Internal modules）

每个模块位于 `src/<name>/` 下，作为一个有界的子模块：一个 `SPEC.md`（它自己的边界）+ 一个
**barrel** `index.ts`，后者是它唯一的公开接口。兄弟模块**通过其 barrel、绝不通过其内部**import 模块。
它们之间的边在这里拥有（见依赖图），不在叶子规格中。

| 模块 | 拥有 | 规格 |
| --- | --- | --- |
| `host` | `Bun.serve` HTTP+WS、静态 SPA、WS 派发注册表、channel 发布 | [host/SPEC.md](src/host/SPEC.md) |
| `persistence` | 数据目录下的 JSON 领域/配置状态 | [persistence/SPEC.md](src/persistence/SPEC.md) |
| `log` | 显式分级诊断 → 美化 stderr + `<dataDir>/logs` 下面向 agent 的 JSONL（pino-roll 按日/10 MB 轮转，14 个轮转文件 + 当前文件）；任意 console 输出只留在终端 | [log/SPEC.md](src/log/SPEC.md) |
| `settings` | 服务端同步的应用配置，包括共享的自定义布局预设目录和 Central 配额偏好（绝不是当前/默认布局） | [settings/SPEC.md](src/settings/SPEC.md) |
| `environment` | 已存储的宿主环境（pi 配置目录 + 变量），在引导时应用到 `process.env` | [environment/SPEC.md](src/environment/SPEC.md) |
| `locales` | 用户在 `<dataDir>/locales/` 下的本地语言文件：校验、监视、`locales.list`/`changed` | [locales/SPEC.md](src/locales/SPEC.md) |
| `themes` | 用户在 `<dataDir>/themes/` 下的主题文件：读取、监视、`themes.list`/`changed`（由客户端校验） | [themes/SPEC.md](src/themes/SPEC.md) |
| `projects` | 稳定的已知仓库注册表：open/recent 视图 + 无损的 close/reopen（校验、去重、slug） | [projects/SPEC.md](src/projects/SPEC.md) |
| `workspaces` | 工作区 = 各自分支上的 `git worktree` | [workspaces/SPEC.md](src/workspaces/SPEC.md) |
| `git` | `git(cwd, args)` 运行器 + 相对 base 的 worktree 状态/diff + 分支列表 | [git/SPEC.md](src/git/SPEC.md) |
| `subprocess` | `runBounded(argv, …)`：一个子进程、一份预算，到期按进程组杀死 | [subprocess/SPEC.md](src/subprocess/SPEC.md) |
| `github` | 只读的本地 `gh` 认证状态（shell-out），供 New-Workspace 界面使用 | [github/SPEC.md](src/github/SPEC.md) |
| `branch-review` | 工作区分支对应的尽力而为 GitHub PR / GitLab MR，含打开的 PR 的检查、评审结论和合并状态 | [branch-review/SPEC.md](src/branch-review/SPEC.md) |
| `pr` | `pr.open` / `workspace.push`：push 工作区分支 + 打开/更新其 GitHub PR，正文来自计划、提交或模板；ready / merge / reopen 它并删除其远端分支 | [pr/SPEC.md](src/pr/SPEC.md) |
| `fs` | 在 worktree 内读取目录/文件（路径受限）+ 唯一的字节分类（text/mime/sha-256） | [fs/SPEC.md](src/fs/SPEC.md) |
| `changes` | 宿主侧回滚 worktree 中的某个 hunk 或某文件的整个改动，带回执 | [changes/SPEC.md](src/changes/SPEC.md) |
| `turns` | 按 agent 运行的 worktree 快照 → 每次运行产生的改动集（`workspace.turns`、`turn.changed`） | [turns/SPEC.md](src/turns/SPEC.md) |
| `trash` | 通过内置辅助程序把路径移入操作系统回收站（绝不用 `unlink`） | [trash/SPEC.md](src/trash/SPEC.md) |
| `spec` | worktree 的规格图谱快照（`spec.graph`）+ 项目级 `projectHasSpecs`，经由 `pi-spec-graph/core` | [spec/SPEC.md](src/spec/SPEC.md) |
| `todos` | 某个会话的按会话 TODO 计划读写（`todo.*`），经由 `pi-todos/core` | [todos/SPEC.md](src/todos/SPEC.md) |
| `reviews` | 文件/diff 上的草稿评审评论：存储 + 锚定 + 上下文包渲染 | [reviews/SPEC.md](src/reviews/SPEC.md) |
| `watch` | 按 worktree 的 fs 监视器 → 去抖的 `workspace.fsChanged` 失效推送 | [watch/SPEC.md](src/watch/SPEC.md) |
| `terminal` | 工作区作用域的 `bun-pty` 终端 | [terminal/SPEC.md](src/terminal/SPEC.md) |
| `agent` | 进程内 pi 会话 + 当前/保留的运行时世代 + 一次性补全 | [agent/SPEC.md](src/agent/SPEC.md) |
| `extensions` | ThinkRail 扩展的服务端注册表（`@thinkrail/ext-*/server`）：有序描述符 + 公开说明符、内联展平、按所有者作用域的 skill 解析 | [extensions/SPEC.md](src/extensions/SPEC.md) |
| `auth` | provider 状态/登录，加上原生 JetBrains Central 生命周期与配额编排 | [auth/SPEC.md](src/auth/SPEC.md) |
| `assist` | 在廉价模型上跑临时的单次任务（计划摘要、PR 描述），尽力而为 | [assist/SPEC.md](src/assist/SPEC.md) |
| `analytics` | 始终开启的基础事件 + 已脱敏的错误报告，以及受偏好控制的可选洞察 → PostHog 汇聚（隐私契约见其规格） | [analytics/SPEC.md](src/analytics/SPEC.md) |
| `feedback` | 宿主级的使用计数 + 需回应的产品访谈邀请生命周期 | [feedback/SPEC.md](src/feedback/SPEC.md) |
| `dialog` | 宿主原生文件夹选择器 | [dialog/SPEC.md](src/dialog/SPEC.md) |
| `editors` | 检测已安装的编辑器/IDE，在某个 worktree 启动一个，在文件管理器中显示某个 worktree | [editors/SPEC.md](src/editors/SPEC.md) |
| `history` | 基于 pi 会话文件的提示召回 + 会话搜索 | [history/SPEC.md](src/history/SPEC.md) |
| `templates` | 对 pi 的提示模板目录做文件增删改查（全局 + 项目作用域） | [templates/SPEC.md](src/templates/SPEC.md) |

`src/index.ts` 重新导出 `host` 加 `agent` barrel 的 `registerBundledRuntime` 接缝；显式的包子路径
暴露构建支持和获准的历史夹具，而不拓宽运行时 barrel。`src/dev.ts` 通过 `bootHost` 从环境变量
引导宿主，用于开发/e2e。

## 内部依赖图（Internal dependency graph）

`host` 是**唯一的组合根**——它把每个特性的处理器接进 WS 注册表。

- `host` → `projects`、`workspaces`、`git`、`github`、`branch-review`、`pr`、`fs`、`spec`、`todos`、
  `reviews`、`changes`、`turns`、`watch`、`terminal`、`dialog`、`editors`、`agent`、`auth`、`assist`、
  `settings`、`environment`、`locales`、`themes`、`history`、`templates`、`analytics`、`feedback`、`log`、
  `persistence`（`dataDir`，用于崩溃报告）
- `workspaces` → `projects`、`git`、`persistence`
- `branch-review` → `git`、`subprocess`
- `pr` → `workspaces`、`git`、`todos`、`branch-review`（provider 检测 + gh 输出解析 + 共享 CLI 运行器）、
  `github`（`ghSetupProblem`——命名的 compare 回退原因）
- `projects` → `git`（共享运行器）、`persistence`
- `git` → `subprocess`（每个与网络或其他 CLI 对话的子进程）、`fs`（`resourceMeta`/`decodeText`——
  diff 一侧的内容分类与 `fs.readFile` 报告的是同一个）
- `changes` → `git`（scope→range 解析器 + 原始侧在其解析后 oid 处的 blob）、`fs`（路径受限 + 字节身份）、
  `persistence`（工作区查找）、`trash`（删除文件式的回滚）
- `turns` → `git`（`snapshotWorktree` + `turn` 作用域的 `gitStatus`）、`persistence`（`turns.json`）、`log`；
  它从不 import `agent`——`host` 把会话事件流喂给它，并把会话解析到其工作区，
  与运行分析观察者用的是同一组合
- `github` → `subprocess`（两个 `gh auth status` 探针与 `git`/`branch-review` 运行在同一个有界运行器下）
- `git`、`fs`、`spec`、`watch`、`terminal`、`settings`、`locales`、`themes`、`analytics`、`feedback` →
  `persistence`（`locales` 还 → `@thinkrail/locales` 根，属外部式使用：目录校验与文件命名）
  （`spec` 还 → `pi-spec-graph/core`，外部；`analytics` 还 → pi-ai 内置的 provider/model 目录 +
  `posthog-node`，外部——身份分桶词汇表与投递 SDK）
- `log` → `persistence`（`dataDir`）——而且**任何特性模块（+ `host`）都可以 → `log`**：
  它是唯一的横切边，和 `persistence` 一样不受"彼此绝不互相 import"规则的约束
  （当前：`host`、`agent`、`workspaces`、`watch`、`git`、`todos`、`reviews`、`analytics`）。
  `persistence` 从不 import `log`（会成环）；`initLogging` 只从 `host` 的 `bootHost` 调用
- `todos` → `workspaces`（worktree 路径查找）+ `pi-todos/core`（外部、值导入、不含 pi）
- `reviews` → `workspaces`（worktree 路径查找）、`persistence`（数据目录）、`git`（解析评审的 baseSha，
  加上 base 侧锚点背后的 diff range + blob 读取）、`fs`（共享字节分类 + 作为锚点 `contentHash` 的 sha-256）。
  `review.send*` 流程**在 `host` 的处理器中组合**（reviews 构建上下文包，`agent` 跑会话——
  没有 `reviews`→`agent` 边；`host` 通过 `reviewLock` 按工作区串行化发送*和*评审变更，
  并通过 `agent.ensureSessionAttached` 重新挂接评审已持久化的聊天），
  agent 侧的 `resolve_comment` 工具通过一个由 `host` 安装的接缝回调
  （`agent.setReviewCommentHandler` → `reviews.resolveCommentFromAgent`）
- `environment` → `persistence`（`environment.json`）、`log`（+ `@thinkrail/shared/shellEnv` 用于启动键
  和 pi 的 `getAgentDir`，外部）；绝不 → `settings`
- `assist` → `agent`（一次性补全原语）
- `auth` → `agent`（当前运行时/认证门面加候选准备/激活；单向，`agent` 从不 import `auth`）
- `agent` → `log`、`persistence`（用于委派的 `dataDir` 加会话生命周期/回执读写操作）、
  `trash`（删除聊天时对转录的可恢复移动）、`extensions`（注册表展平后的父级与子级内联扩展，
  以及非打包开发时的 skill 根）——除此之外只有 pi 运行时；auth 通过其公开的世代接缝
  传入期望的不透明 Central 路径
- `extensions` → `@thinkrail/ext-*/server` + `@thinkrail/extension-api/server`（外部）；`buildSupport.ts`
  （仅构建）也读取它来暂存扩展的 skill 根
- `persistence`、`dialog`、`history`、`templates`、`subprocess`、`trash` →（叶子）

规则：特性模块从不 import `host`，除上述边外也从不互相 import。该图无环。
`agent` 的 WS 表面（`session.*` + `pi.event` 转发）挂到 `host`。自行推送的特性模块同样从不 import `host`：
它们暴露一个**发布者注入接缝**（`setTerminalPublisher`、
`setSessionPublisher` + `setSessionCreatedPublisher` + `setSessionDeletedPublisher` +
`setSessionStatePublisher`、`setLoginPublisher`、`projects` 的全快照 `project.updated` 生命周期所用的
`setProjectPublisher`、`workspaces` 的 `workspace.created`/`updated`/`removed` 生命周期三件套所用的
`setWorkspacePublisher`、`settings` 的 `settings.changed`（完整合并配置加上成功的已应用更新，
用于宿主侧的显式字段效果）所用的 `setSettingsPublisher`、`feedback` 的需回应邀请发布者，
以及 auth 的 Central 动作分析 + `provider.changed` 失效发布者），由 `host` 在
`createServer` 时安装——因此 channel/分析接线只存在于 `host`。当前布局没有宿主模块、持久化、方法或发布者。

`settings` 校验它拥有的有界、无资源自定义布局预设目录。当前/默认预设、
分组限制、frame、工作区资源放置、选中与几何从不进入宿主。创建工作区与引导恢复路径把 `workspaces`
与 `terminal` 组合：先保留确定性的、无进程的默认终端，然后在持久化目录成功之后才清除工作区的待定标记。
在这套握手上，没有兄弟模块 import 另一个。
`history` 保持不依赖注册表（从不 import `projects`/`workspaces`）；`host` 在处理器层
（`history.search` 处理器）从注册表注入作用域过滤 + 标签。`templates` 同样保持不依赖注册表——
它接受普通 `cwd`，从不接受 `workspaceId`；`template.*` 处理器在调用 `templates` 之前通过
`workspaces` 把 `workspaceId` 解析为 `cwd`。

分析同样由宿主中介：**每个采集调用点都位于 `host` 中**。宿主把既有的引导、会话、设置、任务、
评审和 PR 观测翻译成封闭的事件词汇表，并从显式包含 `analyticsEnabled` 的已应用更新同步
附加数据投递门，无关的设置广播保留当前偏好。关联信息保持在宿主本地，并在关闭共享时清除。
`analytics` 没有 `settings` 边，也没有任何特性模块知道分析的存在。

Chat Resources 集成新增一条 `agent` → `pi-background-commands` 的外部包边，与它已有的委派包并列。
`host` 继续通过 `agent` barrel 组合 wire；没有新的资源管理器兄弟模块，也没有
`agent` → `terminal` / `subprocess` / `settings` 边。命令生命周期与有界输出属于可移植包，
而管理器拥有它与实际父会话的关联。

子 agent 可用性同样由宿主中介：`settings` 拥有全局默认值，`workspaces` 拥有可选的本地覆盖，
`host` 把它们生效后的取值注入 `agent`，并在任一处变更后请求对活动会话重新求值。
没有特性模块 import 兄弟模块来推导该策略。

扩展的开/关同样由宿主中介：`settings` 拥有 `disabledExtensions`，`extensions` 拥有注册表及其目录派生，
`host` 通过 `setDisabledExtensionsResolver` 把该集合注入 `agent`，通过把注册表目录与设置联结来响应
`extension.list`，并在包含 `disabledExtensions` 的已应用更新之后调用 `refreshExtensionTools()`。
没有 `agent` → `settings` 或 `extensions` → `settings` 边。

## 必须做对（Get right）

- **没有进程隔离**——致命的 agent/provider 故障会拖垮整个宿主（已接受的权衡）。
- **没有通用的跨进程状态协调**——`bootHost` 允许多个宿主使用同一个数据目录。安装身份、它的一次性
  事件抢占以及浏览器归因尝试使用窄范围的排他文件系统操作；其他共享持久化没有通用的锁、事务或
  收敛保证，因此并发写可能是最后写入者胜出。
- **每个宿主只有一次优雅关闭**——启动器 await `RunningServer.shutdown()`；重复调用共享同一个 promise，
  而突然死亡依赖操作系统的进程清理。
- **WS 命令直接返回值**；只有事件 + 扩展 UI 使用推送通道。
- 通过宿主选项绑定到 localhost 之外（Tailscale 接缝）。

## 后续（Later）

把持久化放到数据层之后，`owner` 贯穿。
