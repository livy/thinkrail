# ThinkRail 学习指南：从 0 到 1

> 这是一份面向新加入者的系统学习材料。目标不是让你背下所有文件，而是让你在读完并完成练习后，
> 能独立回答三个问题：**这个项目为什么存在？一次用户输入如何变成屏幕上的输出？我应该去哪里改代码？**
>
> 权威来源始终是仓库中的英文原文：[AGENTS.md](file:///Users/luzhengli/Documents/github/thinkrail/AGENTS.md)、
> [goal-and-requirements.md](file:///Users/luzhengli/Documents/github/thinkrail/goal-and-requirements.md)、
> [architecture.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.md) 以及各模块的 `SPEC.md`。
> 中文译本与原文不一致时以原文为准。本指南中的行号可能随代码演进而漂移，定位符号时请优先用搜索。

---

## 目录

- [0. 先读这一页](#0-先读这一页)
- [1. 项目是什么：问题、用户与原则](#1-项目是什么问题用户与原则)
- [2. 架构总览：三环、两个启动器、一条事件流](#2-架构总览三环两个启动器一条事件流)
- [3. 仓库地图](#3-仓库地图)
- [4. 核心概念与数据模型](#4-核心概念与数据模型)
- [5. 端到端生命周期：一次聊天是如何跑起来的](#5-端到端生命周期一次聊天是如何跑起来的)
- [6. 服务端详解](#6-服务端详解)
- [7. Web 客户端详解](#7-web-客户端详解)
- [8. 扩展体系](#8-扩展体系)
- [9. 规格驱动开发](#9-规格驱动开发)
- [10. 开发环境、命令与验证](#10-开发环境命令与验证)
- [11. 分阶段学习路径（0 → 1）](#11-分阶段学习路径0--1)
- [12. 关键文件索引](#12-关键文件索引)
- [13. 术语表](#13-术语表)
- [14. 常见误解与 FAQ](#14-常见误解与-faq)
- [15. 学完之后](#15-学完之后)

---

## 0. 先读这一页

如果你只有十分钟，请记住下面五句话：

1. **ThinkRail 是一个 IDE 外壳，不是另一个 AI agent。** 真正的 agent 引擎是 `pi`，
   它被**进程内**嵌入宿主，模型、skill、扩展、压缩、成本、会话状态都由 `pi` 拥有。
2. **三个环。** 引擎宿主（`packages/server` + `packages/shared`）↔ 类型化线上协议
   （`packages/contracts`）↔ 可独立发布的 UI（`apps/web`）。
3. **宿主只做两件事：启动 `pi`，并把 `pi` 报告的事实桥接给 UI。** 它不重算成本、不重算统计、
   不拼 prompt、不实现第二个 agent 循环。
4. **客户端先水合、再流式。** 每个客户端连接后先从宿主重建领域状态（projects/workspaces/sessions），
   然后才接收增量事件；它绝不依赖“我正好看见了某条事件”才能显示正确状态。
5. **规格是代码的一部分。** 每个包和重要的子目录都有 `SPEC.md`，边界、公开面、禁止依赖都写在里面；
   改行为之前先改规格。

**这份材料怎么读：**

- 第一遍：读第 1、2、3、5 章，建立地图和主线，不要陷进细节。
- 第二遍：读第 6、7 章，配合 IDE 打开对应源码。
- 第三遍：读第 4、8、9、10 章，理解模型、扩展与工程约束。
- 最后：按第 11 章的阶段逐项完成练习与自测。练习不要求改产品代码，重点是“定位 + 解释 + 验证”。

---

## 1. 项目是什么：问题、用户与原则

### 1.1 一句话

ThinkRail 是 JetBrains 的 incubator 项目，一个**越用越好的 agentic IDE**：薄宿主运行 `pi`，
把工作隔离到独立 git worktree，并通过活的规格与可复用 skill 给 agent 持久的项目知识。
目标是“用 agent 构建，而不失去控制”。

来源：[goal-and-requirements.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/goal-and-requirements.zh.md)、
[README.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/README.zh.md)。

### 1.2 它要解决什么问题

`pi` 本身刻意保持最小、可扩展。这种自由带来的代价是：

| 问题 | 具体表现 | ThinkRail 的回应 |
| --- | --- | --- |
| 并行 agent 相互碰撞 | 多个 agent 和用户编辑挤在同一个工作树里 | 每个工作区是独立 git worktree，有自己的分支与 cwd |
| agent 改动难以评审 | CLI 里读 diff、按行评论、再把反馈交回 agent 很笨拙 | 内置 diff、锚定评论、结构化评审上下文回送聊天 |
| 意图会蒸发 | 聊天里的决策随会话结束而消失 | 规格图谱（spec graph）+ 可复用 skill，跨会话保存意图 |

### 1.3 面向谁

- 想用可视化入口上手 `pi` 的新手。
- 想在 agent 周围有一个真正 IDE、但不想放弃 `pi` 深度的有经验用户。
- 想快速推进、又不愿失去对 agent 行为控制权的 agent-first 构建者。

### 1.4 四条价值

1. **默认隔离**：ThinkRail 创建的工作区是带独立分支与 cwd 的 git worktree；直接改项目主文件夹
   （Default 工作区）是显式选择，不是默认。
2. **agent 周围的真正 IDE**：编辑器、diff、终端、评审在同一个 workbench 中，而不是从终端回滚缓冲区
   里重建 agent 的工作。
3. **规格即真相**：项目意图存在于代码旁的规格图谱，agent 可以读取、搜索、维护它。
4. **pi 的全部，无所隐藏**：模型、skill、扩展、会话状态仍是 `pi` 的；ThinkRail 增加的是界面，
   不是第二个 agent。

### 1.5 六条原则（会反复出现在代码里）

| 原则 | 在代码中的体现 |
| --- | --- |
| **`pi` 是唯一引擎，进程内运行** | 服务端通过 `createAgentSession` 嵌入 `@earendil-works/pi-coding-agent`；没有子进程，没有 `claude-agent-sdk` |
| **通过“喂料”影响 agent，绝不代为组装** | 功能通过 prompt 上下文、文件、pi skill/扩展、会话启动标志来塑造 agent；宿主不拼系统提示 |
| **暴露，不重算** | 成本、token、统计、会话状态直接采用 pi 报告的值 |
| **用户自己的工具与凭据** | git、gh、central、provider 认证都走用户环境；ThinkRail 不保存 token |
| **信任是显式的** | 仓库提交的 `.pi/` 资源、skill 别名、MCP server 需要按项目授权；MCP 条目变更会使批准失效 |
| **规格优先** | 每个模块先有 `SPEC.md`；行为改变先改规格，再改代码 |

### 1.6 能力地图

| 能力 | 一句话 | 主要规格 |
| --- | --- | --- |
| 两个启动器，一个应用 | 桌面应用与 `thinkrail` CLI，都嵌入同一宿主 | [apps/desktop/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/desktop/SPEC.zh.md)、[apps/cli/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/cli/SPEC.zh.md) |
| 项目 → 工作区 | git 仓库是 Project；工作区是 worktree（Default 是例外） | [packages/server/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/SPEC.zh.md) |
| Workbench | 可分割的文件/diff/文档/聊天/终端布局 | [apps/web/src/shell/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/shell/SPEC.zh.md) |
| 聊天 | 每工作区并发的 pi 会话、模型、运行中操控、后续指令 | [apps/web/src/chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md) |
| 规格 | `spec_*` 工具搜索/导航/维护规格图谱 | [packages/spec-graph/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/spec-graph/SPEC.zh.md) |
| 计划与逐步评审 | 共享 TODO 计划，步骤可独立提交与评审 | [packages/pi-todos/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/pi-todos/SPEC.zh.md) |
| Subagent | 前台、并行或后台的隔离子会话 | [packages/pi-subagents/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/pi-subagents/SPEC.zh.md) |
| 评审 | 文件与 diff 上的锚定评论，本地评审而非 forge 集成 | [packages/resource-renderers/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/resource-renderers/SPEC.zh.md) |
| Open PR | 推送分支并用用户的 `gh` 打开/更新 PR | [packages/server/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/SPEC.zh.md) |
| Provider | 应用内登录、API key、JetBrains AI via `central` | [central-integration.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/central-integration.zh.md) |
| 主题与语言 | 内置主题 + 用户主题文件；UI 文本走 i18n key | [packages/locales/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/locales/SPEC.zh.md) |

### 1.7 非目标（不要试图“补全”它们）

- **第二个 agent 运行时**，或宿主侧的 agent 循环 / prompt 组装。
- **ThinkRail 账户或登录 UI**：访问控制由网络（Tailscale ACL、设备身份）决定。

> 自测：为什么“在宿主里重算成本”违反项目原则？如果产品想要“更省钱”，正确做法可能是什么？
>
> <details><summary>参考思路</summary>
> 因为 pi 拥有会话状态与成本数据，宿主重算会产生第二份真相，且可能与 pi 的分支、压缩、缓存语义不一致。
> 正确做法是消费 pi 报告的成本，把优化放在“喂给 pi 的东西”上（模型选择、上下文、skill、压缩策略），
> 而不是在宿主里另算一套。
> </details>

---

## 2. 架构总览：三环、两个启动器、一条事件流

### 2.1 三环

```
┌──────────────────────────────────────────────────────────────┐
│  UI 环：apps/web                                             │
│  React 19 + Zustand + Tailwind v4 + Vite                     │
│  只依赖 contracts / locales / ui / extension-api/web /       │
│  resource-renderers / thinkrail-extensions/*/web             │
└───────────────────────────┬──────────────────────────────────┘
                            │ 类型化 WS 线上协议
                            │ packages/contracts
                            │ 请求/响应、通道、PiEvent、版本常量
┌───────────────────────────▼──────────────────────────────────┐
│  引擎宿主环：packages/server + packages/shared               │
│  Bun + Node ≥ 22.19；进程内运行 pi                           │
│  拥有 projects / workspaces / sessions / git / fs / reviews  │
└───────────────────────────┬──────────────────────────────────┘
                            │ 进程内调用
┌───────────────────────────▼──────────────────────────────────┐
│  pi：@earendil-works/pi-coding-agent 1.4.0                   │
│  模型、系统提示、skill、扩展、压缩、成本、会话状态            │
└──────────────────────────────────────────────────────────────┘
```

关键点：

- `apps/web` **不能**依赖 `server`、`shared` 或任何 `pi` 包。浏览器代码里值导入 pi 是硬红线。
- `packages/contracts` 是唯一的耦合面：它包含类型、方法映射、通道名、版本常量与少量无依赖运行时。
- `packages/server` 负责“怎么启动、怎么持久化、怎么调 pi、怎么投影事件”；
  `packages/shared` 只放跨进程/跨启动器的通用工具（free port、启动标记、版本、CodedError、路径等）。

对应架构决策：Decision 1（客户端/宿主分离）、Decision 2（启动器薄、宿主是库）、
Decision 3（传输线带版本）。见 [architecture.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.zh.md)。

### 2.2 两个启动器

```
apps/cli  ──┐
            ├── bootHost() ── packages/server (host)
apps/desktop┘
```

- `apps/cli`：`thinkrail` bin，解析 argv，调用 `bootHost({ portMode: "free", ... })`，
  然后在浏览器打开 UI。入口见 [bootstrap.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/cli/src/bootstrap.ts#L64)。
- `apps/desktop`：Electrobun 原生外壳，在同一个 Bun 进程里嵌入服务端，绑定 `127.0.0.1:0`，
  把打包的 web 产物、`/ws`、`/files` 同源服务给原生 webview。
- 两者都**不拥有**引擎逻辑；所有启动逻辑都在 [boot.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/boot.ts#L62)。

### 2.3 一条事件流（先记住这条主线）

```
用户在 ChatView 输入并发送
        │
        ▼
transport.request("session.prompt", { sessionId, text, ... })
        │  WS frame: { id, method, params, sessionId? }
        ▼
server.ts  收到 frame → ack/resume 对账 → handleRequest
        │
        ▼
handlers.ts  "session.prompt" → promptSession()
        │
        ▼
agentSessionManager.promptSession → pi session.prompt()
        │
        ▼
pi 产生事件（message_update / tool_execution_* / agent_end / agent_settled ...）
        │
        ▼
projectSessionEvent 投影为线上允许的 PiEvent
        │
        ▼
server.ts 通过 channel "pi.event" 推送
        │
        ▼
WsTransport.handleMessage → piEventBatcher → handlePiEvents
        │
        ▼
reduceSessionEvent 折叠进 SessionRuntime.turns
        │
        ▼
ChatView 选择器 → deriveRows → ChatTranscript → ToolRendererBody
```

这条主线在第 5 章会逐段展开。

### 2.4 五个必须记住的不变量

1. **一个 id 模型**：UI tab id 与 `session.sessionId` 是不同层的东西；不存在额外的 pi UUID。
2. **流式语义**：`text_delta` / `thinking_delta` **追加**；
   `tool_execution_update.partialResult` **替换**。
3. **`prompt()` 在流式期间会抛错**；运行中要用 `steer()` / `followUp()`。
4. **自动工作以 `agent_settled` 结束**，不是 `agent_end`。`agent_end` 只是“某一次尝试结束”，
   后面可能还有重试、压缩、恢复或排队的续接。
5. **错误是数据**：通过事件流与抛出的方法结果转发，不要把用户可见的失败当成宿主崩溃。

这些不变量在 [AGENTS.md](file:///Users/luzhengli/Documents/github/thinkrail/AGENTS.md) 中有原文。

---

## 3. 仓库地图

### 3.1 顶层目录

| 路径 | 作用 |
| --- | --- |
| `apps/cli` | 浏览器启动器 `thinkrail` bin；源码入口 + 编译二进制入口 |
| `apps/desktop` | Electrobun 原生桌面外壳与打包 |
| `apps/web` | React 客户端（Vite 构建） |
| `apps/website` | Astro 静态官网 |
| `packages/server` | 宿主：启动、host、agent、持久化、git、项目/工作区、评审、扩展注册表 |
| `packages/shared` | 宿主与启动器共享的小工具 |
| `packages/contracts` | 线上契约与协议版本 |
| `packages/ui` | 自有的 shadcn/Radix 原语、按钮系统、`cn` |
| `packages/locales` | 扁平 i18n 字典与 React 绑定 |
| `packages/extension-api` | 扩展 SDK：`defineWebExtension` / `defineServerExtension` |
| `packages/resource-renderers` | 文件/diff 渲染器注册表与 11 个渲染器 |
| `packages/spec-graph` | `spec_*` 工具 + skill + 规则 |
| `packages/pi-todos` | `todo_*` 六工具 + skill |
| `packages/pi-thinkrail-workflow` | 工作流 skill 家族与路由规则 |
| `packages/pi-subagents` | `Agent` / `get_subagent_result` |
| `packages/pi-background-commands` | `background_command` |
| `packages/pi-delegation` | 委托核心（可移植库） |
| `packages/pi-dag` | 持久 DAG（不捆绑进 ThinkRail） |
| `pi-extensions/*` | 发布到 npm 的可移植 pi 包（`@thinkrail/pi-<name>`） |
| `thinkrail-extensions/*` | ThinkRail 扩展包（server 半边 + web 半边） |
| `e2e` | 浏览器 E2E harness 与夹具 |
| `scripts` | 开发脚本、门禁脚本、规格表面检查 |
| `design-audit` | 设计审计相关材料 |

### 3.2 依赖方向（简化）

```
apps/cli ──▶ server ──▶ shared
apps/desktop ──▶ server ──▶ shared
apps/web ──▶ contracts, locales, ui, extension-api/web, resource-renderers,
             thinkrail-extensions/*/web
server ──▶ contracts, shared, spec-graph, pi-todos, pi-thinkrail-workflow,
           pi-subagents, pi-background-commands, pi-delegation,
           thinkrail-extensions/*/server
thinkrail-extensions/*/server ──▶ pi-extensions/*（或等价的 pi 能力包）
```

禁止方向：

- `apps/web` ↛ `server` / `shared` / `pi`
- `contracts` ↛ 任何 React、DOM、server 实现
- 功能面板（`apps/web/src/panels`）↛ 布局引擎；面板必须与布局无关
- 扩展 `./web` ↛ pi 值导入；扩展 `./server` ↛ 宿主内部构件

### 3.3 从哪里开始读

**第一遍（主线）：**

1. [AGENTS.md](file:///Users/luzhengli/Documents/github/thinkrail/AGENTS.md)
2. [goal-and-requirements.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/goal-and-requirements.zh.md)
3. [architecture.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.zh.md) 的“决策”一节
4. [packages/contracts/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/SPEC.zh.md)
5. [packages/server/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/SPEC.zh.md)
6. [apps/web/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/SPEC.zh.md)

**第二遍（按需）：**

- 要做服务端会话/流式：`packages/server/src/agent/`
- 要做前端聊天/渲染：`apps/web/src/chat/`
- 要做布局：`apps/web/src/shell/`
- 要做文件预览/评审：`packages/resource-renderers/`
- 要做扩展：`packages/extension-api/` + `thinkrail-extensions/`
- 要做启动/打包：`apps/cli/` + `apps/desktop/` + `packages/server/src/host/`

---

## 4. 核心概念与数据模型

### 4.1 Project 与 Workspace

- **Project** = 一个 git 仓库。它有 `id`、`path`、`name`，以及打开/最近状态。
- **Workspace** = 一个工作上下文。最常见的是 git worktree，位于 `~/.thinkrail/worktrees` 下；
  每个工作区有自己的分支与 cwd。
- **Default 工作区**（`kind: "default"`）是项目主工作树本身，是显式选择，不是默认隔离。
- **External 工作区**（`kind: "external"`）是就地附加的已有 worktree。

关系：

```
Project (git repo)
   ├── Default workspace   → 项目文件夹本身
   ├── Managed workspace   → ~/.thinkrail/worktrees/...（独立分支 + cwd）
   └── External workspace  → 用户已有的 worktree
```

为什么重要：agent 的隔离、并行、diff、评审、PR 都建立在“工作区是一个独立 worktree”上。
见 [architecture.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.zh.md) Decision 6。

### 4.2 Session、Tab 与“一个 id 模型”

- **Session** 是 `pi` 的 `AgentSession`，身份是 `session.sessionId`。
- **Tab** 是 UI 的标签页，有自己的 tab id；tab 记录“这个工作区里打开了哪些会话/文件/终端”。
- 两者不是同一个东西，也**不存在**额外的 pi UUID。宿主把“UI 的 tab 指向哪个 session”作为 UI 状态处理，
  会话本身只由 `sessionId` 标识。

常见错误：把 `tabId` 当成 `sessionId` 发给服务端，或反过来在会话事件里期待 tab 信息。

### 4.3 SessionState 与 AgentSettlement

`SessionState` 是宿主对“这个会话现在处于什么状态”的投影（[piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L82)）：

```ts
interface SessionState {
  execution: ...;        // 是否在跑、是否等待输入等
  runId: ...;            // 当前运行的 id
  needsInput: boolean;
  completion: ...;       // 完成信息
  completionUnread: boolean;
  queuedCount: number;
  backgroundWork: ...;   // 后台工作
}
```

`AgentSettlement` 是“自动工作真正结束”的边界（[piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L63)）。
只有 `agent_settled` 会产生 completion；`agent_end` 不是结束信号。

为什么需要 settlement：pi 在 `agent_end` 之后还可能自动重试、压缩上下文、恢复、执行排队的续接。
如果 UI 在 `agent_end` 就显示“完成”，用户会看到状态反复跳变。

### 4.4 PiEvent 与 SessionEventPayload

`PiEvent` 是线上允许的会话事件联合类型（[piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L110)）。
`SessionEventPayload` 是“事件 + 它属于哪个会话”的信封（[piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L161)）。

注意：**pi 的原始事件不会原样上网**。服务端会做投影：
`packages/server/src/agent/sessionEventProjection.ts` 只保留客户端需要的字段，
例如：

- `agent_settled` 带上最终的 terminal 元数据；
- `compaction_end` 只保留白名单字段；
- `tool_execution_update` 去掉 `structuredContent`；
- `entry_appended` 只留 `entryType`。

见 [sessionEventProjection.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/sessionEventProjection.ts#L4)。

### 4.5 请求 / 响应 / ack / resume

线上请求：

```ts
{ id: string, method: string, params: unknown, sessionId?: string }
```

线上响应：

```ts
{ id: string, ok: true, result: unknown }
{ id: string, ok: false, error: string, errorCode?: string }
```

客户端在每次响应读取后发送 `{ ack: [id] }`（微任务批量），宿主在收到 ack 前必须假定应答可能随 socket
死亡并保持可重放。重连时客户端发送 `{ resume: [ids] }`，列出仍未解析的完整集合，宿主释放其他一切。

宿主用 `(clientKey, requestId)` 去重，避免“已接受的变更被重放成两次执行”。

见 [contracts/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/SPEC.zh.md) 与
[transport/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/SPEC.zh.md)。

### 4.6 协议版本

当前线上协议版本是 **102**：

- [wsProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/wsProtocol.ts#L138)
- 同一文件还定义了特性版本常量（例如 chat resources、MCP、review anchors、turn changes 等）。
- `server.welcome` 携带协议版本；客户端据此启用/禁用能力。
- 客户端在 socket URL 上追加 `?client=<pageId>&protocol=<version>`，让宿主知道对面是谁、能渲染什么。

版本化不是“可选项”：独立发布的 UI 可能比宿主新或旧，客户端必须按特性门禁降级，而不是假设宿主一定支持。

---

## 5. 端到端生命周期：一次聊天是如何跑起来的

这一章是整份材料的核心。建议边读边打开对应文件。

> 深度展开版：[第 5 章专题：一次聊天从点击到渲染的完整生命周期](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH05-LIFECYCLE.zh.md)。
> 包含逐事件折叠表、ack/resume 协议、水合代次缓冲、错误路径与 8 组追踪练习。

### 5.1 启动宿主

1. `apps/cli` 解析 argv/env（`args.ts`），决定端口、host、静态目录、是否打开浏览器。
2. 调用 `bootHost(...)`（[bootstrap.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/cli/src/bootstrap.ts#L64)）。
3. `bootHost` 在 [boot.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/boot.ts#L62)：
   - 初始化日志；
   - 安装崩溃日志；
   - 应用宿主环境（修复登录 shell 环境等）；
   - 初始化 Central runtime；
   - 找空闲端口；
   - `createServer(...)`；
   - 安装进程信号处理。
4. `createServer` 在 [server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L271)：
   - 创建 `Bun.serve`；
   - 处理 `/ws` upgrade；
   - 处理 HTTP 静态资源、`/files`、`/blob`；
   - 装配请求处理器与通道发布器。

桌面端走同一条 `bootHost`，只是固定 `127.0.0.1`、端口 `0`，并在同一个 Electrobun Bun 进程里服务打包 web 产物。

### 5.2 客户端连接与 welcome

1. `apps/web` 启动时执行初始化副作用（[main.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/main.tsx#L25)）：
   `initializeThemes()` → `applyThemePreference()` → 字体 → `initTransport()` → 错误上报 →
   偏好持久化 → 导航 → `registerWebExtensions()`。
2. `initTransport()` 创建 `WsTransport` 单例，连接到 `ws://<host>/ws?client=...&protocol=102`。
3. 连接成功后，宿主在 `open` 处理里订阅通道并发送 `server.welcome`
   （[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L379)）。
4. `server.welcome` 携带：
   - 协议版本；
   - 打开/最近项目；
   - 可选 config；
   - appVersion / hostPlatform / hostUpdate 等。
5. 客户端 `wireTransport.serverWelcome` 把它交给 `installWelcomeSnapshot(...)`，
   在**一次 Zustand 写入**里安装协议、项目、config 等，避免 UI 看到半个 welcome。
6. 随后客户端按需请求 `session.stateList` 等，把会话状态水合回来。

“先水合、再流式”的体现：客户端不会因为“我没看到创建事件”而缺少一个项目/会话；
welcome + 显式 list 请求是权威来源，增量事件只是后续修正。

### 5.3 打开项目与工作区

1. 用户在 Welcome 屏或 rail 菜单打开项目。
2. 客户端发送项目/工作区请求；宿主在 `projects` / `workspaces` 模块里创建或附加。
3. 宿主返回快照；客户端用 `applyProjectUpdated` / `updateWorkspace` 等原子动作更新 store。
4. 工作区激活时，UI 设置 `activeWorkspaceId`、布局视图状态、位置条等。

注意：其他客户端的打开操作不会抢走当前导航；后台关闭也不会移动当前视图。
这些规则在 [store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md) 里。

### 5.4 创建会话

1. 客户端发送 `session.create`。
2. 服务端 `handlers.ts` 路由到 `createSession`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1365)）。
3. `prepareSessionEntry`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1121)）准备：
   - 会话设置（模型、thinking level 等）；
   - 资源加载器；
   - 订阅会话事件；
   - 绑定扩展（`bindExtensions({ mode: "rpc", uiContext })`）。
4. `createParentSessionInternal`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1416)）
   最终调用 `createAgentSession(...)`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1473)）。
5. 宿主登记会话、发布 `session.created`，并把初始 `SessionState` 投影给客户端。

关键：**宿主不创建第二个 agent 循环**。它只创建 pi 的 `AgentSession`，并把 pi 报出来的东西转发。

### 5.5 发送一条消息

1. `ChatView` 的发送路径调用 `transport.request("session.prompt", ...)`。
2. `WsTransport.request`：
   - 生成 `trpi_<seq>` 请求 id；
   - 把 frame 写入 socket；
   - 在 `pending` 里保存 resolver；
   - 60 秒默认超时（人类交互类请求需要显式提高 `timeoutMs`）。
3. 服务端 `server.ts` 的 `message` 处理：
   - 先处理 `ack` / `resume`；
   - 经 `RequestReplayCache` 去重；
   - 调 `handleRequest`（[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L1537)）。
4. `handlers.ts` 的 `"session.prompt"` 分支调用 `promptSession`
   （[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2284)）。
5. `promptSession` 调 pi 的 `session.prompt(...)`。

运行中再次发送：

- `steer()`：改变当前运行方向；
- `followUp()`：在当前运行结束后追加。

对应 `steerSession` / `followUpSession`
（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2312)）。

pi 原始 API 在流式期间调 `prompt()` 会抛错；但 ThinkRail 的 `promptSession` 包装器检测到
`isStreaming` 后会自动转 `steerEntry`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2291-L2294)）。
自动转 steer 只是兜底，UI 仍应显式选择 steer / followUp 来表达意图。

### 5.6 pi 事件如何变成线上事件

1. pi 在会话过程中产生事件。
2. 宿主通过 `setSessionPublisher` 把事件送到发布器
   （[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L352)）。
3. `projectSessionEvent`（[sessionEventProjection.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/sessionEventProjection.ts#L4)）
   投影为线上 `PiEvent`：
   - 去掉客户端不需要的内部字段；
   - 对 `agent_settled` 附上终态；
   - 对 `tool_execution_update` 去掉 `structuredContent`；
   - 对 `entry_appended` 只保留类型。
4. `server.ts` 的发布器把事件打到 `pi.event` 通道
   （[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L898)）。

为什么要投影：线上契约必须稳定、可版本化、体积可控；直接把 pi 的内部事件透出去会把 UI 绑死在 pi 实现细节上。

### 5.7 前端接收、批量与折叠

1. `WsTransport.handleMessage` 识别 `pi.event`，交给 `subscribe` 的回调。
2. `wireTransport` 把连续事件交给 `piEventBatcher`：
   - 保持到达顺序；
   - 不丢事件；
   - 约 30 Hz 一次原子投递；
   - 128 条强制冲刷。
3. 批量到达后调用 store 的 `handlePiEvents(payloads)`，在**一次原子写入**里折叠。
4. `reduceSessionEvent` 把每个 `PiEvent` 折叠进对应会话的 `SessionRuntime`：
   - `text_delta` / `thinking_delta` 追加；
   - `tool_execution_update.partialResult` 替换；
   - `message_end`、`compaction_end`、`agent_settled` 等更新边界状态。
5. `ChatView` 通过选择器读取当前会话运行时，`deriveRows(...)` 把 turn 投影为行，
   `ChatTranscript` 渲染，工具行交给 `ToolRendererBody`。
6. `ToolRendererBody` 调 `getToolRenderer(toolName)`；未注册的工具走默认渲染器。

为什么要有 `deriveRows`：pi 每个工具轮次发一条 assistant 消息，折叠会跨越 assistant 消息边界；
按“原始 turn”渲染无法正确分组，所以要先派生成行。

### 5.8 结算、完成与未读

1. pi 发出 `agent_end`：只表示某次尝试结束，**不产生 completion**。
2. 若之后没有重试/压缩/恢复/续接，pi 发出 `agent_settled`。
3. 服务端投影 `agent_settled` 并携带 terminal 信息。
4. 前端 `reduceSessionEvent` 在 `agent_settled` 时结束运行、生成 completion / unread 状态。
5. 用户查看后发送 `session.acknowledgeCompletion` 清除未读。

对应 `SessionState` 的 `completion`、`completionUnread` 与 `isSessionWorking`。

### 5.9 重连、水合与“错过的事件”

- socket 断开后，客户端保留未解析请求，重连时用 `{ resume: [...] }` 对账。
- `pi.event`、`session.created`、`session.deleted` 等通道**不重放**（它们是一次性增量）。
- 重连后的修复靠显式水合：`session.stateList`、`session.list`、`session.getMessages`。
- 工作台水合会从下一次权威 `session.list` 修复断开期间错过的删除。

这就是“先水合、再流式”的工程含义：**增量可以不重放，但权威快照必须能重建状态**。

### 5.10 时序图（完整版）

```
CLI/Desktop          Host (server)                 pi                 Web
    │                     │                         │                  │
    │ bootHost()          │                         │                  │
    ├────────────────────▶│                         │                  │
    │                     │ createAgentSession      │                  │
    │                     ├────────────────────────▶│                  │
    │                     │                         │                  │
    │                     │◀──── pi events ─────────│                  │
    │                     │ projectSessionEvent     │                  │
    │                     │                         │                  │
    │                     │◀────────── WS connect ──────────────────────┤
    │                     │──── server.welcome ────────────────────────▶│
    │                     │◀──── session.create ───────────────────────┤
    │                     │──── session.created / state ──────────────▶│
    │                     │◀──── session.prompt ───────────────────────┤
    │                     ├──── prompt() ───────────▶│                  │
    │                     │◀──── message_update ─────│                  │
    │                     │──── pi.event (text_delta) ─────────────────▶│
    │                     │                         │                  │  batch + reduce
    │                     │◀──── agent_end ──────────│                  │
    │                     │◀──── agent_settled ──────│                  │
    │                     │──── pi.event (settled) ────────────────────▶│
    │                     │                         │                  │  completion
```

### 5.11 追踪练习

1. 打开 `ChatView.tsx`，找到发送函数，列出它调用的 transport 方法。
2. 打开 `handlers.ts`，找到对应 method 分支，跟到 `agentSessionManager`。
3. 打开 `sessionEventProjection.ts`，解释为什么 `tool_execution_update` 要去掉 `structuredContent`。
4. 打开 `wireTransport.ts`，找到 `pi.event` 路由，跟到 `handlePiEvents`。
5. 在 `appStore.ts` 里找到 `reduceSessionEvent`，观察 `text_delta` 与 `tool_execution_update` 的不同处理。
6. 打开 `ChatView.tsx` / `rows.ts`，解释一个 assistant 消息为什么可能被拆成多个行。

> 自测：如果用户在流式期间又发了一条消息，客户端应该调用哪个方法？为什么不应依赖 `prompt()`？
>
> <details><summary>参考思路</summary>
> 应调用 `steer()`（改变当前运行方向）或 `followUp()`（等当前运行结束后追加）。pi 原始 API 在流式期间
> 调 `prompt()` 会抛错，因为 pi 不允许并发启动两个 prompt 轮次；ThinkRail 的 `promptSession` 虽然会自动
> 转成 steer，但那只是兜底，不能表达“追加”的意图，也不应让客户端把语义交给宿主猜测。
> </details>

---

## 6. 服务端详解

服务端规格：[packages/server/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/SPEC.zh.md)。
记住它的职责：**启动、持久化、环境、项目/工作区、git、会话、扩展、认证、分析**，
但它不是第二个 agent。

> 深度展开版：[第 6 章专题：服务端模块、请求与资源详解](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH06-SERVER.zh.md)。
> 包含 35 个模块的职责分类与依赖边、启动/关停顺序、HTTP 字节通道、WS 重放缓存与背压、
> 各领域状态机（工作区/评审/计划/git scope），以及 10 组追踪练习。

### 6.1 `host/`：进程与协议

| 文件 | 作用 |
| --- | --- |
| `host/boot.ts` | 共享启动路径：日志、崩溃日志、环境、Central、找端口、`createServer`、信号 |
| `host/server.ts` | `Bun.serve`、`/ws` upgrade、message 分发、通道发布、静态与 `/files` |
| `host/handlers.ts` | 所有 WS 方法的路由与实现入口 |
| `host/requestReview.ts` | 评审请求相关处理 |
| `host/directorySpelling.ts` | 目录拼写/身份处理 |
| `host/historyScope.ts` | 历史作用域 |

请求处理的关键点：

- 每个请求有 `id`；响应回同一个 `id`。
- `ack` / `resume` 是客户端的一半契约；宿主必须保留可重放的应答。
- 会话级方法带 `sessionId`，但 `session.create` 等可以不带。
- 处理器返回 `ok: false` 时，`errorCode` 让客户端区分具体失败（`RequestError` / `wsErrorCode`）。

### 6.2 `agent/`：会话与事件

| 文件 | 作用 |
| --- | --- |
| `agent/agentSessionManager.ts` | 会话注册表、创建、提示/操控/续接、中止、发布器 |
| `agent/sessionEventProjection.ts` | pi 事件 → 线上 `PiEvent` 投影 |
| 其他 | 资源加载、设置、会话绑定、扩展绑定等 |

`agentSessionManager` 的关键结构：

- `sessions` Map：会话 id → 条目。
- `registerSession`：登记并发布 `session.created`。
- `createSession` / `createParentSessionInternal`：构造 pi `AgentSession`。
- `prepareSessionEntry`：准备设置、资源加载器、订阅、扩展绑定。
- `promptSession` / `steerSession` / `followUpSession` / `abortSession`：会话操作。
- `recordSettlement`：记录 `agent_settled` 并更新状态。
- `buildSessionSettings`：把宿主设置转成 pi 会话设置。

重要约束：

- 不要在宿主里推导“运行是否结束”；用 pi 的 settlement。
- 不要把 pi 的内部对象直接序列化给客户端。
- 会话创建失败要作为方法错误返回，同时清理半初始化资源。

### 6.3 持久化与设置

`persistence/`、`settings/`、`environment/` 负责：

- `~/.thinkrail` 下的数据；
- 项目/工作区/会话的磁盘记录；
- 用户设置（模型、主题、语言、分析同意、扩展开关等）；
- 登录 shell 环境修复（`resolveShellEnv`）；
- 每个宿主一套 pi 配置（不是每个项目一套）。

原则：**磁盘是宿主拥有的领域状态，pi 拥有会话内部状态**。不要把两者混成一个存储模型。

### 6.4 项目、工作区与 git

`projects/`、`workspaces/`、`git/` 负责：

- 打开/克隆/关闭项目；
- 创建/附加/删除工作区（git worktree）；
- 分支、diff、提交、PR 支撑；
- 网络 git 操作的超时预算（服务端把网络 git 限制在 55 秒，让带 git stderr 的错误先于客户端的 60 秒通用超时到达）。

阅读时重点看：

- 工作区的 `kind` 与路径规则；
- Default 工作区的特殊语义；
- 删除工作区时如何处理本地未提交工作；
- 目录身份的大小写/拼写规则（Windows 与 POSIX 不同）。

### 6.5 `shared/`：跨进程小工具

[packages/shared/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/shared/SPEC.zh.md)：

- 空闲端口；
- 启动标记；
- 版本/发布身份；
- Windows 身份；
- `CodedError`（给失败一个稳定 code）；
- 删除树、路径、jbcentral。

这些工具不属于任何领域，所以放在 shared；但**不能**把领域逻辑塞进来。

### 6.6 分析与日志

- 错误无条件上报 PostHog Error Tracking，但**只作为分类器**：类型、`code`、source/scope/request method、
  ThinkRail 自己的栈帧；绝不上报 `message` 文本。
- 宿主诊断走 `logger(scope)`，不要用 `console.*`。
- 允许列表在 `packages/server/src/analytics/SPEC.md`。

### 6.7 必须做对

- 启动失败要可诊断，且不留下半启动的宿主。
- 会话创建/提示/中止的错误要沿事件流与方法结果返回。
- 不要在主线程做长时间阻塞操作（尤其是网络 git、文件扫描）。
- 路径要规范化后再比较（大小写、分隔符、`..`）。
- 不要假设客户端一定是当前版本的 web。

> 自测：为什么服务端不把 `tool_execution_update` 的 `structuredContent` 发给客户端？
>
> <details><summary>参考思路</summary>
> 它是 pi 的内部结构，体积可能大、格式可能随 pi 版本变化，客户端也不需要它来渲染工具卡。
> 投影掉它既减小传输量，又把 UI 与 pi 内部实现解耦；需要的信息通过稳定的 `PiEvent` 字段表达。
> </details>

---

## 7. Web 客户端详解

总规格：[apps/web/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/SPEC.zh.md)。

> 深度展开版：[第 7 章专题：Web 客户端从启动到渲染的空间结构](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH07-WEB.zh.md)。
> 包含 `main.tsx` 启动顺序、transport 的 ack/resume/重放与投递屏障、store 的代次围栏与六类状态、
> chat 的行派生与工具注册表、shell 布局的不重挂载规则、资源渲染与主题/i18n，以及 10 组追踪练习。

### 7.1 启动与初始化顺序

[main.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/main.tsx#L25)：

```
initializeThemes()
applyThemePreference()
initializeFonts()
initTransport()
installClientErrorReporting()
setErrorBoundaryReporter(...)
安装偏好持久化
initNavigation()
registerWebExtensions()
```

顺序有意义：主题和字体要在 React 挂载前生效，transport 要在 store 需要数据前就位，
错误上报要在边界之前装好，扩展注册要在渲染工具卡之前完成。

`App` 渲染树：

```
ErrorBoundary > TooltipProvider > AppResourceHost > Shell
```

### 7.2 transport：唯一 WS 客户端

[transport/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/SPEC.zh.md)：

- `WsTransport`：
  - `request(method, params, opts)` 按 id 关联响应；
  - 默认 60 秒超时；
  - 重连退避 500ms → 10s；
  - `subscribe(channel, cb)`；
  - `inferUrl` 默认同源；
  - `httpBase()` 推导宿主 HTTP 源，用于 `/files` 等 URL。
- `piEventBatcher`：
  - 30Hz 批量；
  - 128 条强制冲刷；
  - 保持顺序、不丢事件。
- `wireTransport`：
  - 单例 `initTransport` / `getTransport`；
  - `server.welcome` → `installWelcomeSnapshot`；
  - `pi.event` → 批量 → `handlePiEvents`；
  - `pi.extensionUi` → `applyExtUi`；
  - `workspace.created/updated/removed` → 对应原子动作；
  - `session.created/deleted` → 历史/墓碑；
  - `provider.changed` → `noteProviderChanged` + 重读 model list；
  - `settings.changed` → `applyConfig`。
- `errorText` / `RequestError` / `wsErrorCode`：把宿主错误归一化。
- `mcp.ts`：MCP 列表请求串行化，防止旧应答覆盖新配置。

一个容易忽略的点：**所有订阅都在 init 时一次性完成，不在组件 effect 里反复订阅**。
这让事件顺序与 store 提交边界可控。

### 7.3 store：Zustand 单店

[store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md)。

store 拥有：

- 连接状态（`connectionGeneration`、`welcomeGeneration`）；
- projects / recentProjects；
- workspaces / 活动工作区 / 选择历史；
- 工作台 frame / 每工作区视图状态 / 注意；
- 终端目录；
- 每个活动会话的 `SessionRuntime`（因此多个 chat 可并发流式）；
- Chat Resources（宿主快照 + 失效修订版 + 连接代数）；
- MCP 状态。

关键规则：

1. **原子写入**：welcome、批量事件、workspace 生命周期都用一次 Zustand 写入完成，
   避免 UI 看到中间态。
2. **代次围栏**：连接代数、welcome 代数、资源修订版、MCP 代数；旧代次的迟到应答必须被丢弃。
3. **派生状态在 selector**：组件不各自推导同一规则（例如资源分组、工作区 live/settled）。
4. **浏览器本地状态与领域状态分开**：布局、折叠、未读提示等是 UI 状态；项目/会话是宿主领域状态。

`reduceSessionEvent` 是纯函数：输入 `PiEvent`，输出新 runtime。它不碰 transport、不碰 React。

### 7.4 chat：从 turn 到行

[chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md)。

核心分层：

```
ChatView（应用集成：store + transport）
   │
   ├── 选择器：当前会话 runtime、工具结果、流式标志
   ├── deriveRows(turns, toolResults, isStreaming, isSpec?)
   │      └── 纯函数，把内容块走成行
   └── ChatTranscript
          ├── UserTurn / SystemTurn / RetryTurn
          ├── MarkdownTurn（流式增量按块渲染）
          ├── CompactionNotice
          ├── ErrorTurn（永不折叠）
          └── ToolTurn → ToolRendererBody → getToolRenderer(toolName)
```

重要行为：

- 用户消息超过 500 字符，agent 开始回复后折叠为 `Show more`。
- skill 调用、评审上下文包、压缩通知都有专门卡片。
- `ErrorTurn` 永不折叠，失败不会被“看起来什么都没发生”。
- 工具渲染器通过 `registerToolRenderer` 按工具名注册；未注册走默认渲染器。
- MCP 工具名带 `mcp__` 前缀。

### 7.5 shell 与布局

[shell/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/shell/SPEC.zh.md)。

- `Shell` 是唯一组装根：topbar、活动项目/工作区路由、主题、全局快捷键、错误边界。
- 面板是**布局无关**的；shell 决定怎么摆。
- `WorkspaceWorkbench` 组合每个工作区的视图状态与布局。
- 布局状态（`layoutState/`）拥有本地水合、校验、持久化与租约；布局引擎（`layout/`）是纯变换。
- `layoutIntents/` 消费一次性意图；`chatReconciliation/` / `terminalReconciliation/` 做宿主与本地放置的收敛。

为什么要“面板布局无关”：未来移动 shell 可以用不同方式投影同一批面板；桌面停靠不能泄漏进功能面板。

### 7.6 样式、图标、i18n、主题

- **样式**：Tailwind v4 工具类映射到生成的语义 CSS 变量 token。禁止内联 style 对象、raw hex、
  内部调色板名、未知 token 工具类。颜色看 [COLOR.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/styles/COLOR.md)，
  间距看 [SPACING.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/styles/SPACING.md)。
- **图标**：`@remixicon/react`；默认 Line，激活态 Fill。
- **原语**：`@thinkrail/ui/<primitive>`；`cn()` 在 `@thinkrail/ui/utils`。
- **i18n**：所有用户可见文本是 [packages/locales/en.json](file:///Users/luzhengli/Documents/github/thinkrail/packages/locales/en.json) 中的 key，
  通过 `@thinkrail/locales/react` 的 `useT` / `<T>` 渲染；lint 拒绝硬编码 JSX 文本。
- **主题**：内置主题 + `~/.thinkrail/themes/*.theme.json`；运行时校验，原子应用 CSS 变量。

### 7.7 资源渲染器

[packages/resource-renderers/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/resource-renderers/SPEC.zh.md)。

- 一个注册表服务所有展示文件的地方：Files/Diff 窗格、文件对比、扩展工具卡。
- 11 个渲染器：code、markdown、binary、LFS、image、SVG、CSV、JSON、notebook、PDF、HTML。
- 宿主通过 `ResourceHostProvider` 提供 markdown、code、bytesUrl、openFile、readFile、
  subscribeFileChanges、FileReview、ReviewSend。
- `ResourcePreview` 可单独预览一个文件，也可给 `before` 做前/后 diff。
- 渲染器实现是动态导入，`GuardedRendererDiff` 负责大 diff 提示与虚拟化。

### 7.8 前端必须做对

- 不要绕开 store 直接改 React 本地状态来模拟领域状态。
- 不要新增第二个 transport 或第二个 i18n 实例。
- 不要把 `server` / `shared` / pi 值导入浏览器代码。
- 流式渲染要能承受高频事件：批量、选择器、虚拟化、按块 markdown。
- 任何“能力”都要按协议版本门禁，旧宿主上要降级而不是崩溃。

> 自测：为什么 `pi.event` 要在前端批量处理，而不是收到一条就 `setState` 一次？
>
> <details><summary>参考思路</summary>
> 流式期间事件频率很高（text_delta 可能每 token 一条）。逐条 setState 会造成大量 React 渲染与选择器重算，
> 甚至让输入卡顿。批量把同一帧内的多条事件合成一次原子 store 写入，既保持顺序，又把渲染次数压到约 30Hz；
> 128 条强制冲刷避免低帧率时延迟无限增长。
> </details>

---

## 8. 扩展体系

这是本项目最有特色、也最容易混淆的部分。先记住：**有两种“扩展”，不要混为一谈。**

> 深度展开版：[第 8 章专题：扩展体系——能力、组合、开关与渲染](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH08-EXTENSIONS.zh.md)。
> 包含四种扩展形态的区分、可移植 pi 包的原生一致性门槛、`./server` + `./web` 描述符的逐字段语义、
> 静态注册表与展平规则、`disabledExtensions` 的立即/延后效果、交互式工具跨重启的邮箱生命周期，
> 以及 10 组追踪练习。

### 8.1 两种扩展

| 类型 | 位置 | 身份 | 目的 |
| --- | --- | --- | --- |
| **可移植 pi 包** | `pi-extensions/<name>/`（以及尚未迁移的 `packages/pi-*`） | npm `@thinkrail/pi-<name>` | 在**原生 pi** 中也能工作；ThinkRail 只是嵌入者之一 |
| **ThinkRail 扩展** | `thinkrail-extensions/<name>/` | 私有 `@thinkrail/ext-<name>` | 把一项 pi 能力与 ThinkRail 专属的宿主/UI 半边组合起来 |

可移植包的要求（vanilla parity）：

- 从打包 tarball 安装，用 pi 自己的解析器加载，零诊断；
- Node ≥ 22.19 + jiti 与 Bun 双运行时都能跑；
- 只用 `node:` 内建与标准 ESM，不出现 `Bun.*` / `bun:*`；
- 不假设 `~/.thinkrail` 存在，不 import 宿主 transport；
- 有 TUI 渲染时提供 `renderCall` / `renderResult`。

ThinkRail 扩展的形态（[thinkrail-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/thinkrail-extensions/SPEC.zh.md)）：

```ts
// ./server
export default defineServerExtension({
  name,
  title?,
  description?,
  extensions,        // 组合进每个父会话的 pi ExtensionFactory
  sessionExtensions?,// 按会话构建更多父工厂
  interactiveTools?, // 调用需要等待人的工具
  childExtensions?,  // 子会话使用的工厂
  skillPackages?,    // 需要暂存的 skills 包
});

// ./web
export default defineWebExtension({
  name,
  toolRenderers,     // 工具名 → React 渲染器
});
```

### 8.2 注册表组装

- 服务端：[packages/server/src/extensions/registry.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/registry.ts)
  静态 import 每个 `@thinkrail/ext-*/server`，导出 `{ specifier, extension }`。
- Web：[apps/web/src/extensions/registry.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/extensions/registry.ts)
  静态 import 每个 `@thinkrail/ext-*/web`。
- 静态 import 让扩展工厂进入正常的服务端构建图，因此编译二进制/桌面打包不需要生成工厂列表。

用户可以在 Settings → Extensions 里逐个关闭扩展；关闭会同时丢弃它的工厂、活动会话中的工具、
以及它 `skillPackages` 的 skills。

### 8.3 工具名是连接键

服务端能力与 UI 渲染器**只通过工具名**连接：

```
pi 工具 "visualize"  ──registerToolRenderer("visualize", ...)──▶  工具卡
```

未注册的工具使用 `DefaultToolRenderer`。MCP 工具使用 `mcp__` 前缀。
这条规则在 [chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md) 与
[toolRegistry.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/toolRegistry.tsx) 中。

### 8.4 随会话加载的包一览

| 包 | 提供 | 关键点 |
| --- | --- | --- |
| `packages/spec-graph` | `spec_*` 七工具 + skill + 启动规则 | 让 agent 读取/搜索/维护规格图谱 |
| `packages/pi-todos` | 六个 `todo_*` 工具 + skill | 分组即任务；按会话 JSON；计划评审基础 |
| `packages/pi-thinkrail-workflow` | workflow skill 家族 + 路由规则 | 项目初始化、设计头脑风暴、交付 PR |
| `packages/pi-subagents` | `Agent` / `get_subagent_result` | 前台/并行/后台子会话；定义发现与信任 |
| `packages/pi-background-commands` | `background_command` 四动作 | 会话绑定；有界保留；完成投递 |
| `packages/pi-delegation` | 委托核心 | `createChild`、运行所有权、fork/捕获 |
| `packages/pi-dag` | 持久 DAG | 资源所有权、命令/证据/门禁；不捆绑进 ThinkRail |

`pi-extensions/` 目前包含 `ask-user-question` 与 `visualize`（可移植半边），
对应的 ThinkRail 半边在 `thinkrail-extensions/`；`web-access` 直接位于 `thinkrail-extensions/`。

### 8.5 想新增一个能力，应该动哪里？

先判断：

1. **这个能力在原生 pi 中有意义吗？** 有 → 先做 `pi-extensions/<name>` 可移植包。
2. **它需要 ThinkRail 专属的宿主服务或 UI 吗？** 需要 → 做 `thinkrail-extensions/<name>`，
   `./server` 组合 pi 工厂，`./web` 注册工具渲染器。
3. **它需要新的线上方法/通道吗？** 需要 → 改 `packages/contracts`，提升协议版本，
   在 `packages/server/host/handlers.ts` 注册，在 `apps/web/transport` 路由。
4. **它只是展示层变化吗？** 可能只需要 `registerToolRenderer`，不需要新协议。

每一步都要先读并更新 owning `SPEC.md`。

---

## 9. 规格驱动开发

> 深度展开版：[第 9 章专题：规格驱动开发——从 frontmatter 到可执行门禁](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH09-SPECS.zh.md)。
> 包含 spec 的 frontmatter 数据模型、父树/DAG 与全序索引、`resolveSpecPath` 的路径门禁、
> 文档层级与权威规则、`check:spec-surface` 的纳入与双向校验、其余 7 个门禁的真实检查方式、
> `pi-compatibility` 的移除门禁写法，以及 10 组追踪练习。

### 9.1 为什么

ThinkRail 的模块很多、消息很多、协议有版本、UI 可独立发布。如果决策只存在于聊天记录或个人脑中，
下一个人会重新推导并得出不同结论。规格是**可评审的、与代码同仓的决策记录**。

### 9.2 层级

- 根级：
  - [goal-and-requirements.md](file:///Users/luzhengli/Documents/github/thinkrail/goal-and-requirements.md)：产品目标与范围。
  - [architecture.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.md)：拓扑、跨模块决策、全仓不变量。
  - [AGENTS.md](file:///Users/luzhengli/Documents/github/thinkrail/AGENTS.md)：工作规则与验证门禁。
- 包级：每个 `packages/*/SPEC.md`、`apps/*/SPEC.md`。
- 子模块级：`apps/web/src/<sub>/SPEC.md`、`packages/server/src/<sub>/SPEC.md` 等。
- 中文译本：同名 `.zh.md`，均声明英文原文为唯一权威。

### 9.3 边界规则

- 子模块只通过 `index.ts` barrel 暴露公开面；兄弟模块从 barrel 导入，不碰内部文件。
- 例外：`apps/web/src/panels` 与 `packages/ui` 保留按文件导入，以避免代码分割或库约定问题。
- 兄弟模块之间的依赖边写在父模块的 `SPEC.md`，不写在每个叶子上。
- 改变边界先改 owning spec。

### 9.4 检查脚本

| 命令 | 检查什么 |
| --- | --- |
| `bun run check:deps` | 依赖是否精确 pin、catalog 是否一致 |
| `bun run check:boundaries` | 模块边界是否被越过 |
| `bun run check:seams` | 编译二进制/桌面的接缝 |
| `bun run check:spec-surface` | 规格声明的公开面与 barrel 是否一致 |
| `bun run check:i18n` | 硬编码 UI 文本、翻译键 |
| `bun run check:compiler` | React Compiler bailout 是否超出基线 |
| `bun run check:pi-packages` | 可移植 pi 包的原生一致性 |

### 9.5 一次安全的修改流程

1. **定位 owning spec**：这个行为由哪个模块拥有？
2. **读 spec 与相邻 spec**：确认边界、公开面、禁止依赖、版本门禁。
3. **先改 spec**（如果是行为/边界变化）。
4. **改代码**，保持 barrel 公开面与 spec 一致。
5. **跑最小相关测试**。
6. **跑快速门禁**：`check:deps`、`check:boundaries`、`check:seams`、`lint`、`typecheck`。
7. **涉及发布行为**：合并实现后跑一次完整无 agent E2E `bun run e2e`。
8. **涉及真实 agent 行为**：跑聚焦 live spec，例如 `bun run e2e:full e2e/agent-naming.live.spec.ts`。
9. **交接前**：审查任务范围内的 diff，做一次“减法检查”（删除不必要的抽象/回退/兼容层）。

> 自测：为什么“先改 spec 再改代码”不是形式主义？
>
> <details><summary>参考思路</summary>
> 因为 spec 是边界与契约的唯一可评审来源。先改 spec 会迫使你回答：这属于哪个模块？公开面变了吗？
> 需要升协议版本吗？旧客户端怎么办？如果这些答案只在代码里，review 很难发现边界被悄悄模糊。
> </details>

---

## 10. 开发环境、命令与验证

### 10.1 前置

- **Bun 1.4.4**（`packageManager` 固定）。
- **Node ≥ 22.19**（pi 原生运行要求）。
- macOS / Linux / Windows 均可；桌面打包有平台专属细节。

安装：

```bash
bun install
```

### 10.2 本地开发

```bash
bun run dev
```

`scripts/dev.ts` 会：

1. 找宿主端口与 web 端口；
2. 设置 `THINKRAIL_PORT` / `THINKRAIL_WEB_PORT`；
3. 用 Turbo 同时起 `@thinkrail/server` 与 `@thinkrail/web`；
4. 在 SIGINT/SIGTERM 时清理并打开浏览器。

Vite 代理 `/ws`、`/files`、`/blob` 到宿主，因此浏览器开发与生产同源行为一致。

### 10.3 常用命令

| 命令 | 作用 |
| --- | --- |
| `bun run dev` | 同时起宿主与 web |
| `bun run dev:server` / `dev:web` | 只起一侧 |
| `bun run build` | Turbo 构建所有包 |
| `bun run build:web` | 构建 web |
| `bun run build:binary` | 构建单文件 `thinkrail` 二进制 |
| `bun run typecheck` | Turbo + 根 tsconfig 类型检查 |
| `bun run lint` | Biome 检查 |
| `bun run format` | Biome 写入格式 |
| `bun run test` | 根脚本测试 + Turbo 单元测试 |
| `bun run e2e` | 完整无 agent 浏览器套件（发布行为门禁） |
| `bun run e2e:serial` | 串行调试回退 |
| `bun run e2e:full` | 两阶段：无 agent + 真实 agent（会花真实 token） |
| `bun run e2e:agent` | 真实 provider agent 套件 |
| `bun run test:workflows` | 按需工作流测试（真实 token，不是 commit 门禁） |
| `bun run check:*` | 各类快速门禁 |
| `bun run desktop:dev` / `desktop:build` | 桌面开发/打包 |

完整命令表见 [package.json](file:///Users/luzhengli/Documents/github/thinkrail/package.json#L51)。

### 10.4 测试矩阵

| 层级 | 入口 | 是否需要真实 provider | 何时跑 |
| --- | --- | --- | --- |
| 单元测试 | `bun run test` | 否 | 每次改动 |
| 快速门禁 | `bun run check:*`、`lint`、`typecheck` | 否 | 每次改动 |
| 无 agent E2E | `bun run e2e` | 否 | 发布行为/集成改动，合并实现后跑一次 |
| 聚焦 live E2E | `bun run e2e:full <spec>` | 是 | 触及真实 agent 行为 |
| 完整 live E2E | `bun run e2e:full` | 是 | 需要用户同意（花费 token 与时间） |
| 工作流 | `bun run test:workflows` | 是 | 按需 |
| 二进制/桌面 | `e2e:binary` / `e2e:desktop` + smoke | 视模式 | 改对应产物时 |

E2E 的隔离、分片、凭据剥离、进程树清理细节在 [e2e/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/e2e/SPEC.zh.md)。

### 10.5 依赖与版本

- 所有依赖 pin **精确版本**，不用范围。
- 跨切依赖在根 `workspaces.catalog` 统一 pin，包内用 `catalog:` 引用。
- 只有 peer dependencies 与本地协议（`workspace:*`）豁免。
- 理由见 architecture Decision 10。

### 10.6 调试提示

- 宿主日志：`~/.thinkrail/logs`（`THINKRAIL_LOG_LEVEL` 控制）。
- `--verbose` 让 CLI 以 debug 级启动。
- 前端错误会经 `client.reportError` 上报（只发给支持该协议的宿主）。
- 追踪会话问题：先看 `pi.event` 是否到达，再看 `reduceSessionEvent` 是否正确折叠。
- 追踪请求问题：看 `RequestError` 的 `errorCode`，不要只看 `error` 文本。

---

## 11. 分阶段学习路径（0 → 1）

下面每个阶段都有“目标 / 阅读 / 动手 / 完成标准”。不要跳阶段；后面的阶段假设前面已经建立地图。

### 阶段 0：跑起来

**目标**：能在本地启动应用，打开一个项目，发一条消息。

**阅读：**

- [README.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/README.zh.md) 的安装与开发部分
- [CONTRIBUTING.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/CONTRIBUTING.zh.md)

**动手：**

- [ ] `bun install`
- [ ] `bun run dev`
- [ ] 打开浏览器中的 UI
- [ ] 打开一个本地 git 仓库作为项目
- [ ] 创建一个工作区，发一条简单消息
- [ ] 观察终端里宿主与 Vite 的日志

**完成标准**：你知道 UI、宿主、pi 三者各自的进程/端口关系，并能指出 `dev` 脚本做了什么。

### 阶段 1：建立地图与术语

**目标**：能不查文档说出三环、Project/Workspace、Session/Tab、协议版本。

**阅读：**

- 第 1、2、3、4 章
- [architecture.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.zh.md) 的 23 条决策标题

**动手：**

- [ ] 画出三环与启动器关系图（纸上或文本）
- [ ] 写出 Default / Managed / External 工作区的区别
- [ ] 找到 `PROTOCOL_VERSION` 并说出客户端如何使用它

**完成标准**：能解释“为什么 apps/web 不能 import server”。

### 阶段 2：端到端主线

**目标**：完整追踪一条 `text_delta` 从 pi 到屏幕。

**阅读：**

- 第 5 章
- [transport/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/SPEC.zh.md)
- [store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md) 的 runtime 部分

**动手：**

- [ ] 从 `ChatView` 发送函数跟到 `handlers.ts`
- [ ] 从 `promptSession` 跟到 `createAgentSession`
- [ ] 从 `sessionEventProjection.ts` 跟到 `server.ts` 的发布器
- [ ] 从 `WsTransport.handleMessage` 跟到 `handlePiEvents`
- [ ] 在 `reduceSessionEvent` 里指出 `text_delta` 的追加逻辑
- [ ] 解释 `agent_end` 与 `agent_settled` 的区别

**完成标准**：能在一张纸上按时序写出六个关键文件/函数，并说明每个边界的数据形态。

### 阶段 3：服务端

**目标**：理解宿主如何管理会话、持久化、工作区与错误。

**阅读：**

- [packages/server/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/SPEC.zh.md)
- `packages/server/src/agent/agentSessionManager.ts` 的关键函数
- [packages/shared/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/shared/SPEC.zh.md)

**动手：**

- [ ] 说出 `prepareSessionEntry` 做了哪几件事
- [ ] 找到 `registerSession` 与 `createSession`，说明会话何时被发布
- [ ] 找到 `abortSession`，说明中止如何反映到状态
- [ ] 找到网络 git 的 55 秒预算，解释它为什么必须小于 60 秒
- [ ] 找出一个 `CodedError` 的使用点，说明为什么要给错误 code

**完成标准**：能解释“宿主暴露、不重算”在会话状态与成本上的具体含义。

### 阶段 4：Web 客户端

**目标**：理解 transport → store → chat → shell 的分层。

**阅读：**

- [apps/web/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/SPEC.zh.md)
- [chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md)
- [shell/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/shell/SPEC.zh.md)

**动手：**

- [ ] 从 `main.tsx` 列出初始化顺序，并解释每一步为什么在那个位置
- [ ] 找到 `deriveRows`，说明为什么不能直接渲染 turn
- [ ] 找到 `registerToolRenderer`，新增一个“假设的”工具名映射的阅读路径
- [ ] 找到布局持久化，说明它为什么不经过 transport
- [ ] 找到一个 i18n key，说明 lint 如何阻止硬编码文本

**完成标准**：能区分“领域状态 / UI 本地状态 / 布局状态”，并说明各自所有者。

### 阶段 5：扩展

**目标**：能判断一个新能力应该放在 `pi-extensions`、`thinkrail-extensions` 还是宿主核心。

**阅读：**

- [packages/extension-api/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/extension-api/SPEC.zh.md)
- [pi-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-extensions/SPEC.zh.md)
- [thinkrail-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/thinkrail-extensions/SPEC.zh.md)
- 任选一个随会话包：`spec-graph` 或 `pi-todos`

**动手：**

- [ ] 说出 `defineServerExtension` 每个字段的用途
- [ ] 说出 `./web` 允许 import 什么、禁止 import 什么
- [ ] 找到服务端与 web 的 registry 文件
- [ ] 解释“工具名是连接键”
- [ ] 判断：新增一个“图表预览工具”应该改哪些目录

**完成标准**：能画出一个扩展从 server 工厂到 web 工具卡的完整路径。

### 阶段 6：规格与贡献

**目标**：能按仓库规则安全地做一个小改动并通过门禁。

**阅读：**

- [AGENTS.md](file:///Users/luzhengli/Documents/github/thinkrail/AGENTS.md) 的 Handoff hygiene
- [CONTRIBUTING.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/CONTRIBUTING.zh.md)

**动手：**

- [ ] 选一个只涉及单个模块的小改动（例如错误文案、一个选择器、一个渲染分支）
- [ ] 找到 owning spec，写一句要改什么
- [ ] 改代码
- [ ] 跑 `bun run test`、`bun run lint`、`bun run typecheck`、相关 `check:*`
- [ ] 如果触及发布行为，跑 `bun run e2e`
- [ ] 审查 diff，删除不必要的抽象

**完成标准**：改动可解释、边界未模糊、门禁全绿、没有为了过门禁而加抑制。

---

## 12. 关键文件索引

### 根文档

| 文件 | 为什么读 |
| --- | --- |
| [AGENTS.md](file:///Users/luzhengli/Documents/github/thinkrail/AGENTS.md) | 工作规则、边界、不变量、验证 |
| [goal-and-requirements.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/goal-and-requirements.zh.md) | 产品目标、原则、能力、非目标 |
| [architecture.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.zh.md) | 拓扑与 23 条决策 |
| [README.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/README.zh.md) | 对外能力与安装 |
| [CONTRIBUTING.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/CONTRIBUTING.zh.md) | 开发与提交 |
| [pi-compatibility.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-compatibility.zh.md) | 升级 pi 时的兼容门禁 |
| [central-integration.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/central-integration.zh.md) | JetBrains AI via central |

### 契约

| 文件 | 内容 |
| --- | --- |
| [wsProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/wsProtocol.ts#L138) | 协议版本、特性版本、WS_METHODS、WS_CHANNELS |
| [piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L63) | AgentSettlement、SessionState、PiEvent、SessionSummary |
| [packages/contracts/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/SPEC.zh.md) | 契约责任与边界 |

### 服务端

| 文件/目录 | 内容 |
| --- | --- |
| [host/boot.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/boot.ts#L62) | `bootHost` |
| [host/server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L271) | `createServer`、`/ws`、消息分发、发布 |
| [host/handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L1537) | `handleRequest` 与方法路由 |
| [agent/agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1365) | 会话生命周期 |
| [agent/sessionEventProjection.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/sessionEventProjection.ts#L4) | 事件投影 |
| [extensions/registry.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/registry.ts) | 服务端扩展注册表 |

### Web

| 文件/目录 | 内容 |
| --- | --- |
| [main.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/main.tsx#L25) | 初始化顺序 |
| [transport/transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts) | `WsTransport` |
| [transport/wireTransport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/wireTransport.ts) | 通道路由与 welcome |
| [transport/piEventBatcher.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/piEventBatcher.ts) | 事件批量 |
| [store/appStore.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/appStore.ts) | Zustand store |
| [store/selectors.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/selectors.ts) | 派生选择器 |
| [chat/ChatView.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/ChatView.tsx) | 聊天集成层 |
| [chat/rows.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/rows.ts) | `deriveRows` |
| [chat/toolRegistry.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/toolRegistry.tsx) | 工具渲染器注册表 |
| [shell/WorkspaceWorkbench.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/shell/WorkspaceWorkbench.tsx) | 工作台编排 |
| [shell/layoutState/layoutState.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/shell/layoutState/layoutState.ts) | 布局持久化与租约 |
| [extensions/registry.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/extensions/registry.ts) | Web 扩展注册表 |
| [styles/COLOR.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/styles/COLOR.md) | 颜色规则 |
| [styles/SPACING.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/styles/SPACING.md) | 间距规则 |

### 扩展与包

| 文件/目录 | 内容 |
| --- | --- |
| [packages/extension-api/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/extension-api/SPEC.zh.md) | 扩展 SDK |
| [thinkrail-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/thinkrail-extensions/SPEC.zh.md) | ThinkRail 扩展形态与注册表 |
| [pi-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-extensions/SPEC.zh.md) | 可移植 pi 包与发布 |
| [packages/spec-graph/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/spec-graph/SPEC.zh.md) | 规格图谱 |
| [packages/pi-todos/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/pi-todos/SPEC.zh.md) | TODO 计划 |
| [packages/pi-subagents/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/pi-subagents/SPEC.zh.md) | 子会话 |
| [packages/pi-delegation/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/pi-delegation/SPEC.zh.md) | 委托核心 |
| [packages/resource-renderers/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/resource-renderers/SPEC.zh.md) | 文件/diff 渲染 |

---

## 13. 术语表

| 术语 | 含义 |
| --- | --- |
| **pi** | `@earendil-works/pi-coding-agent`，本项目嵌入的编程 agent 引擎 |
| **宿主（host）** | `packages/server` 中的进程，负责启动、持久化、调 pi、桥接 UI |
| **三环** | 引擎宿主 / 类型化线上协议 / 独立 UI |
| **Project** | 一个 git 仓库 |
| **Workspace** | 一个工作上下文；通常是隔离的 git worktree |
| **Default workspace** | 项目主工作树本身，`kind: "default"` |
| **External workspace** | 用户已有的 worktree，`kind: "external"` |
| **Session** | pi 的 `AgentSession`，由 `sessionId` 标识 |
| **Tab** | UI 标签页，与 session 不是同一个 id |
| **SessionState** | 宿主投影的会话状态（执行、运行 id、等待输入、完成、未读、排队、后台工作） |
| **AgentSettlement** | `agent_settled` 携带的终态；自动工作真正结束的边界 |
| **PiEvent** | 线上允许的会话事件 |
| **SessionEventPayload** | 事件 + 所属会话的信封 |
| **welcome** | 连接后的权威快照（协议、项目、config 等） |
| **hydrate-then-stream** | 先重建权威状态，再接收增量 |
| **ack / resume** | 客户端对响应的确认，以及重连时的未解析请求对账 |
| **RequestReplayCache** | 宿主侧的请求去重/重放缓存 |
| **compaction** | pi 的上下文压缩；有 start/end 与摘要记录 |
| **turn** | pi 的一条消息轮次；UI 会派生成“行”再渲染 |
| **row** | `deriveRows` 产出的渲染单元，可跨 assistant 消息边界折叠 |
| **tool renderer** | 按工具名注册的 React 渲染器；未注册走默认 |
| **extension** | 两种：可移植 pi 包，或 ThinkRail 扩展（server + web 半边） |
| **registry** | 静态扩展注册表；服务端一个、web 一个 |
| **spec graph** | 项目规格图谱，agent 通过 `spec_*` 工具维护 |
| **vanilla parity** | 可移植 pi 包在原生 pi 中也能工作的标准 |
| **React Compiler** | 构建期优化；bailout 有基线门禁 |
| **catalog** | 根 `package.json` 中统一 pin 的跨切依赖版本表 |

---

## 14. 常见误解与 FAQ

### Q1：UI 是不是直接调用 pi？

不是。UI 只跟宿主说话（WS）。宿主在进程内调用 pi。浏览器代码里值导入 pi 是硬红线。

### Q2：`agent_end` 到了，是不是就完成了？

不是。`agent_end` 只是某次尝试结束；自动工作以 `agent_settled` 结束。中间可能有重试、压缩、恢复、排队续接。

### Q3：流式期间能再发一条消息吗？

可以，但要用 `steer()` 或 `followUp()`。`prompt()` 在流式期间会抛错。

### Q4：为什么事件不能直接透传给前端？

因为会把 UI 绑死在 pi 内部实现上，且体积/字段不稳定。服务端投影成稳定的 `PiEvent`。

### Q5：为什么前端要批量事件？

降低 React 渲染次数与选择器重算；保持顺序；30Hz 批量 + 128 条强制冲刷。

### Q6：为什么不能把 `tabId` 当 `sessionId`？

因为 tab 是 UI 本地概念，session 是 pi 的领域身份。宿主只认 `sessionId`；tab 指向哪个 session 由 UI 状态管理。

### Q7：为什么不能直接在 JSX 写中文文案？

用户可见文本必须是 `packages/locales/en.json` 的 key，经 `@thinkrail/locales/react` 渲染；
lint 会拒绝硬编码 JSX 文本与文本属性。

### Q8：为什么不能 `import { X } from "@thinkrail/server"` 到 web？

破坏三环边界，把 Bun/Node/pi 依赖带进浏览器包，且让 UI 不再能独立发布。用 `contracts` 的类型与
transport 方法替代。

### Q9：为什么改一个错误提示也要看 spec？

因为错误可能涉及协议错误码、i18n key、分析分类器、客户端降级。spec 帮你确认这是不是跨边界变化。

### Q10：为什么“跑通测试”不等于“可以交接”？

因为门禁只证明已知约束；交接还要求审查 diff、删除不必要的抽象、确认没有为了过门禁而加抑制、
确认 README/spec 与行为一致。

---

## 15. 学完之后

如果你完成了第 11 章的六个阶段，你应该能够：

- 用三环语言解释任何一条数据流；
- 在十分钟内定位一个新功能应该属于哪个包、哪个模块、哪一侧；
- 安全地修改一个模块，并通过对应门禁；
- 读懂一个 `SPEC.md` 并判断它是否被代码违反；
- 在 review 中指出“这是第二个状态所有者”“这越过了边界”“这会把旧客户端弄崩”。

下一步建议：

1. 选一个 `good first issue` 或小的边界内改动，走完整流程。
2. 读一个你还没读过的模块 `SPEC.md`，画出它的公开面与依赖边。
3. 尝试在不改产品行为的前提下，补一个缺失的测试或一个更清晰的 selector。
4. 如果你要改 pi 相关行为，先读 [pi-compatibility.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-compatibility.zh.md)
   并逐条核对移除门禁。

---

*本材料依据仓库中的英文规格与源码整理；英文原文是唯一权威来源。*
