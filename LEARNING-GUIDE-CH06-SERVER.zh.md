# 第 6 章深度专题：服务端（引擎宿主）详解

> 本专题是 [LEARNING-GUIDE.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE.zh.md) 第 6 章的展开版。
> 主指南讲的是“服务端有哪些东西”，这里讲的是**每个东西为什么存在、边界在哪里、出错时谁负责**。
>
> 阅读前提：已读主指南第 3 章（三环架构）与第 4 章（仓库地图）。
> 与 [第 5 章专题](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH05-LIFECYCLE.zh.md) 的分工：
> 第 5 章追踪**一条消息的时间轴**；本章追踪**一个宿主进程的空间结构**——模块、请求、资源、生命周期。

---

## 6.0 先建立五个“服务端坐标系”

读服务端代码最常见的迷路方式，不是文件太多，而是**同一个词在不同层指不同东西**。
先把坐标系钉死，后面每一节都只是往这五个坐标上挂细节。

### 6.0.1 服务端是什么、不是什么

| 是 | 不是 |
| --- | --- |
| 一个**可嵌入的 Bun 库**：`createServer(options) → Promise<RunningServer>` | 一个独立进程 / 命令行程序（`src/dev.ts` 只是开发与 e2e 的引导） |
| `pi` 的**进程内宿主**：`createAgentSession` 在同一个进程里跑 | `pi` 的子进程包装器，或第二个 agent 运行时 |
| HTTP + WebSocket + 静态 SPA 的**唯一服务器** | 一组按平台分叉的产品实现 |
| 磁盘领域状态的**唯一拥有者**（`~/.thinkrail`） | pi 会话内部状态的拥有者（那是 pi 的） |
| 所有推送通道与分析的**唯一组合根**（`host`） | 一个“大 service 类”，什么都自己实现 |

一句话：**服务端是薄宿主，pi 是厚引擎。** 服务端负责“把 pi 接到浏览器”，不负责“替 pi 思考”。

### 6.0.2 五个身份

| 身份 | 谁生成 | 作用域 | 生命周期 | 典型用途 |
| --- | --- | --- | --- | --- |
| `piRuntime` 世代 | `agent` 的 runtime 管理 | 一个宿主进程内可多次更换 | 换 provider / 换 Central 代理时整体替换 | 每个会话保留自己创建时的世代 |
| `session.sessionId` | pi 的 `SessionManager` | 全局 | 聊天存在期间 | WS 方法、事件、UI tab 共用同一个 id |
| `workspace.id` | `workspaces` | 一个项目下 | 工作区存在期间 | git worktree、终端、diff、评审的作用域 |
| `project.id` | `projects` | 全局注册表 | 项目记录存在期间 | 项目信任、技能、MCP 覆盖的归属 |
| `dataDir()` | `persistence` | 一个宿主机 | 进程/安装期间 | 所有 JSON 状态与日志的根 |

仓库规则里的 “one id model” 说的就是：**UI 的 tab id 与 `session.sessionId` 是同一个会话的两层视图**，
没有隐藏的 pi UUID。工作区与项目则是另外两个真实存在的领域身份，不要混用。

### 6.0.3 三层装配，不是一层

服务端内部其实有三次“装配”，每次只做一件事：

1. **进程装配**：`bootHost(...)` —— 日志、崩溃日志、宿主环境、Central、端口、信号。
2. **宿主装配**：`createServer(...)` —— `Bun.serve`、路由、WS 注册表、发布者接缝、后台计时器。
3. **请求装配**：`handleRequest(method, params, ctx)` —— 把方法名收窄到 `WsMethodName`，再派发给 `handlers`。

越往上，代码越“基础设施”；越往下，代码越“领域”。**只有 `host` 同时看见所有领域。**

### 6.0.4 术语表（读代码前先对齐）

| 术语 | 含义 | 容易混淆的点 |
| --- | --- | --- |
| 组合根（composition root） | 唯一允许把多个特性拼起来的地方，这里是 `host` | 不是“最大的模块”，而是“唯一有装配权的地方” |
| 接缝（seam） | 让下层不 import 上层、由上层注入的函数/发布器 | 不是 mock，而是产品结构 |
| 发布者（publisher） | 领域模块只调用 `setXxxPublisher(...)` 注册的回调 | 领域模块不知道 WS channel 的名字 |
| 收窄（narrow） | 把宽类型变成线上允许的类型 | 例如 `Model → WireModel`，不是简单序列化 |
| 投影（projection） | 把 pi 事件变成稳定的线上事件 | 丢掉内部字段，保留渲染需要的事实 |
| 证据 / 事实层 | 从 git / 文件系统读出来的真实结果 | 不缓存权威，不推导第二套状态 |

---

## 6.1 目录、模块与边界

### 6.1.1 服务端目录的全貌

`packages/server/src/` 下每个目录都是一个**有界子模块**，各有一个 `SPEC.md` + 一个 `index.ts` barrel。
当前实际目录共 35 个（以目录为准，而不是以某张会变的表为准）：

```
agent  analytics  assist  auth  branch-review  changes  dialog  editors
environment  extensions  feedback  fonts  fs  git  github  history  host
locales  log  persistence  pr  projects  resourceHistory  reviews  settings
spec  subprocess  templates  terminal  themes  todos  trash  turns  watch
workspaces
```

按“它在系统里扮演什么角色”归类，比按字母排序有用得多：

| 角色 | 模块 | 共同特征 |
| --- | --- | --- |
| **组合根** | `host` | 唯一能 import 所有特性、唯一知道 WS channel 的地方 |
| **状态拥有者** | `persistence`、`settings`、`environment`、`projects`、`workspaces`、`todos`、`reviews`、`feedback`、`resourceHistory` | 拥有磁盘/内存中的领域状态，自己不做传输 |
| **事实层（git / 文件）** | `git`、`fs`、`changes`、`turns`、`spec`、`watch` | 从仓库与工作树读事实，返回快照/差异 |
| **进程边界** | `subprocess`、`trash`、`dialog`、`editors`、`terminal` | 启动/结束外部进程或操作系统服务 |
| **远端协作** | `github`、`branch-review`、`pr` | 与 GitHub/GitLab 对话，但只做产品需要的窄动作 |
| **会话引擎桥** | `agent`、`auth`、`assist`、`extensions` | 直接围绕 pi 的会话/模型/登录/扩展 |
| **横切** | `log`、`analytics`、`locales`、`themes`、`fonts`、`history`、`templates` | 被多方使用，自身不拥有业务状态 |
| **叶子** | `persistence`、`dialog`、`history`、`templates`、`subprocess`、`trash` | 不 import 兄弟特性（除明确列出的边） |

> 注意：`fonts` 只有英文 `SPEC.md`，正文仍是权威；不要因为缺中文译文就跳过它。

### 6.1.2 barrel 规则：为什么它重要

规则只有一句：**兄弟模块只经 `index.ts` 公开面 import，绝不 import 内部文件。**

这不是风格问题，而是可验证的架构约束：

- `bun run check:boundaries` 与 `bun run check:deps` 会抓越界；
- 公开面被 `bun run check:spec-surface` 校验；
- 子模块的依赖边写在**父模块**的 `SPEC.md`，不散落在每个叶子。

一个模块的 `index.ts` 通常只做两件事：

1. 重新导出真正公开的函数/类型；
2. 导出发布者注入接缝（`setXxxPublisher`）。

例如服务端包的根 barrel 极小：

[packages/server/src/index.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/index.ts)：

```ts
export {
	type BundledExtensionFactory,
	type BundledExtensions,
	registerBundledRuntime,
} from "./agent";
export * from "./host";
export { dataDir } from "./persistence";
```

根 barrel 只暴露三样东西：**宿主工厂、编译后运行时的注入接缝、数据目录**。
其他所有能力都通过 `host` 的 WS 方法暴露，而不是通过扩展根 barrel 暴露——这是刻意的：
**包的公开面越小，启动器与测试越不可能绕过宿主装配。**

### 6.1.3 依赖图：逐边理解

完整依赖图在 [packages/server/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/SPEC.zh.md)。
把它压成“为什么需要这条边”，比背箭头有效：

| 边 | 为什么存在 | 反向为什么不行 |
| --- | --- | --- |
| `host → 所有特性` | 组合根必须能把每个处理器接进注册表 | 特性 import `host` 会形成环，并让领域代码知道传输细节 |
| `workspaces → projects/git/persistence` | 工作区要落在项目下，要用 git worktree，要存记录 | `projects` 不该知道工作区布局 |
| `git → subprocess/fs` | 每个联网 git 走有界 runner；diff 侧字节分类与 `fs.readFile` 同源 | `subprocess` 不该知道 git 语义 |
| `changes → git/fs/persistence/trash` | 用唯一 scope 解析、路径受限、字节身份、工作区查找、删除进废纸篓 | `git` 不该知道“回滚”这个产品动作 |
| `turns → git/persistence/log` | 用 tree 快照定义“一次运行改了什么”，持久化 `turns.json` | **`turns` 从不 import `agent`**：`host` 把事件流喂给它 |
| `reviews → workspaces/persistence/git/fs` | 评审需要 worktree 路径、存储、baseSha 与 blob、内容哈希 | **`reviews` 没有 → `agent` 边**：发送流程在 `host` 组装 |
| `todos → workspaces/git/log` | 计划文件在 worktree 内，完成步骤要提交增量 | `todos` 不 import `host` 或其他兄弟 |
| `projects → git/persistence` | 打开路径要靠 git 判断仓库根，注册表要落盘 | `projects` 不 import `workspaces`（工作区依赖它，不是反过来） |
| `pr → workspaces/git/todos/branch-review/github` | 推送需要 workspace/branch，正文可来自计划，需探测已有评审与 `gh` 环境 | `pr` 不 import `host`，也不持有 provider token |
| `branch-review → git/subprocess` | 只做“这个分支有没有 PR/MR”的远端探测 | 不 import `workspaces`（调用方传 cwd），不持久化 |
| `agent → log/persistence/trash/extensions` + pi | 会话生命周期、转录回收、扩展展平、诊断 | `agent` 不 import `settings`：设置由 `host` 传参/构建 |
| `auth → agent` | 登录与 Central 要驱动当前 runtime 世代 | `agent` 不 import `auth`，方向必须单向 |
| `assist → agent` | 一次性补全复用 agent 的原语 | `agent` 不 import `assist` |
| `environment → persistence/log` | 存宿主环境、记诊断 | **`environment` 绝不 → `settings`**：两者都改 `AppConfig` 附近，但归属不同 |
| `log → persistence` | 日志目录来自 `dataDir()` | `persistence` **从不** import `log`（否则成环） |
| `persistence/dialog/history/templates/subprocess/trash` | 叶子 | 叶子不 import 特性兄弟 |

`log` 是唯一的横切边：任何特性模块（以及 `host`）都可以 → `log`；`persistence` 是另一个例外，
因为几乎所有状态拥有者都要它，但 **`persistence` 自己绝不 import 特性模块**。

### 6.1.4 发布者注入接缝：领域模块如何“推送”却不认识 WS

服务端有一条很容易被忽略的原则：**能自行推送的领域模块，不知道自己推到了哪个 channel。**

它们只暴露 `setXxxPublisher(...)`，由 `host` 在 `createServer` 时安装。典型接缝：

| 接缝 | 谁调用 | `host` 接到哪里 |
| --- | --- | --- |
| `setSessionPublisher` | `agent` | `pi.event` 扇出 |
| `setSessionCreatedPublisher` / `setSessionDeletedPublisher` | `agent` | `session.created` / `session.deleted` |
| `setSessionStatePublisher` | `agent` | `session.state` |
| `setTerminalPublisher` / `setTerminalTabsPublisher` | `terminal` | `terminal.data|exit|detached` / `terminal.tabs` |
| `setProjectPublisher` | `projects` | `project.updated` 全快照 |
| `setWorkspacePublisher` | `workspaces` | `workspace.created|updated|removed` |
| `setSettingsPublisher` | `settings` | `settings.changed` |
| `setReviewPublisher` / `setReviewFailedPublisher` | `reviews` | `review.changed` / `review.failed` |
| `setTurnPublisher` | `turns` | `turn.changed` |
| `setWatchPublisher` | `watch` | `workspace.fsChanged` |
| `setRepoMetaPublisher` | `watch` | 仓库元数据刷新 + 空路径失效 |
| `setLoginPublisher` / `setLogoutPublisher` | `auth` | `provider.login` / `provider.changed` |
| `setFsNudgePublisher` | `git` 的 prefetch 结果 | 基点移动后的失效 |

这个设计带来两个直接好处：

1. **可测试**：领域模块的单元测试只需注入一个记录调用的小函数；
2. **可演进**：channel 改名、加协议门禁、加分析采集，只改 `host`，不改领域模块。

### 6.1.5 新增一个服务端模块的检查单

如果你要给服务端加一个模块，按这个顺序做，能避免 90% 的返工：

- [ ] 目录有 `SPEC.md`（职责 / 边界 / 公开面 / 允许依赖 / 禁止依赖 / 必须做对）
- [ ] 目录有 `index.ts` barrel，只导出公开面
- [ ] 需要在父 `SPEC.zh.md` 的模块表与依赖图里加边
- [ ] 如果它会主动推送，定义 `setXxxPublisher`，**不要 import `host`**
- [ ] 如果它需要传输类型，从 `contracts` 导入；不要自己定义线上形状
- [ ] 如果它要写磁盘，用 `persistence` 的原子写与 `dataDir()`，不要自己拼路径
- [ ] 如果它有错误值得区分，抛 `CodedError`，不要靠 message
- [ ] 在 `host/handlers.ts` 里接方法，并在 `WsHandlers` 类型里补齐（缺一个方法 typecheck 就失败）

---

## 6.2 启动与关停：宿主进程的生命周期

### 6.2.1 `bootHost` 的精确顺序

[boot.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/boot.ts#L62-L88) 的顺序不是随意的，
每一步都依赖前一步的副作用：

```ts
export async function bootHost(options: BootHostOptions): Promise<BootedHost> {
	const shellEnv = options.resolveShellEnv ? resolveShellEnv() : undefined;
	await initLogging({ ... });
	installCrashLog(options.appVersion);
	await shellEnv;
	applyHostEnvironment();
	await initializeJbcentralRuntime();

	const requested = options.port;
	const port = options.portMode === "free" ? await findFreePort(requested, options.host) : requested;
	const running = await createServer({ port, host: options.host, ... });
	const server = attachProcessSignals(running);
	log.info(`listening on port ${server.port}`);
	return { server, port: server.port, requested };
}
```

逐行解释：

1. **先启动 `resolveShellEnv()` 的 promise，但不 await**：登录 shell 解析可能慢；把它与日志初始化重叠。
2. **`initLogging` 必须最早 await**：后续任何 `logger(scope)` 都要有目的地。
3. **`installCrashLog` 紧随其后**：从这一刻起，未捕获异常有落盘与上报路径。
4. **`await shellEnv` 后才 `applyHostEnvironment()`**：已存宿主环境要覆盖/修补登录 shell 导入的变量，
   顺序反了就会被 shell 环境盖回去。
5. **`initializeJbcentralRuntime()` 在选端口前**：Central 失败会回退到普通 runtime，但必须在绑定端口前完成，
   这样每个嵌入者拿到同一个“已初始化”的宿主。
6. **`portMode: "free"` 时先找空闲端口**：桌面端常用；端口确定后才 `createServer`。
7. **`attachProcessSignals`**：SIGINT/SIGTERM 只触发一次 `shutdown()`，然后 `exit(0)`。
8. **最后才记录 `listening on`**：日志只在真正可服务时出现。

> 细节：`createServer` 内部还会再调一次 `initializeJbcentralRuntime()` 与 `getConfig()`。
> 这不是重复劳动，而是让**直接调用 `createServer` 的嵌入者**（测试、无头夹具）也能拿到同样的引导不变量。

### 6.2.2 `createServer` 的装配顺序

[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L271-L1048) 很长，但结构清楚：

| 阶段 | 做什么 | 为什么在这里 |
| --- | --- | --- |
| 预热 | `initializeJbcentralRuntime()`、`getConfig()`、`setPiTrustSeed(...)` | 后续装配要读配置与运行时 |
| 会话状态 | `setSessionProjectResolver`、`initializeSessionStates(loadWorkspaces())` | WS 打开前，历史会话状态要能列出 |
| 内存结构 | `sockets`、`reapTimers`、`RequestReplayCache`、`terminalBackpressured` | 服务器回调需要这些闭包 |
| 后台计时器 | 5 分钟评审刷新、1 秒背压对账 | 与请求无关的周期性工作 |
| `Bun.serve` | HTTP 路由 + `websocket` 回调 | 服务器本体 |
| 发布者接缝 | `setProjectPublisher` … `setSettingsPublisher` … | 领域模块从此可以推送 |
| 观察者 | `watchLocalLocales` / `watchThemes` / `watchFonts` / MCP 配置 | 文件变化变成失效提示 |
| 恢复 | `reviveTerminalSessions()`、`provisionInitialTerminal()`、`openProject(projectPath)` | 重启后恢复成员关系与初始项目 |
| 分析 | `setupObservation`、`observeLoggedErrors`、survey | 只在宿主里接 |
| 返回 | `{ port, startAttributionClaim, stop, shutdown }` | 公开接口 |

关键点：**`Bun.serve` 创建之前**，所有“需要在第一个请求到来前就绪”的东西都已就绪；
**`Bun.serve` 创建之后**，只做接线与恢复，不再改变引导不变量。

### 6.2.3 HTTP 路由表

`Bun.serve.fetch` 的路由是线性的，顺序就是优先级：

| 路径 | 处理 | 作用 |
| --- | --- | --- |
| `/ws` | upgrade + `clientKey` / `protocol` 解析 + 刷新最近项目文件夹 | 唯一的实时通道 |
| `/health` | `new Response("ok")` | 启动探针 |
| `/files/<workspaceId>/<relpath>` | `serveWorktreeFile` | 工作树当前字节，文本/图片都可用 |
| `/blob/<workspaceId>/<oid>/<relpath>` | `serveBlob` | 历史提交中不可变字节 |
| `/fonts/<file>` | `serveUserFonts` | 用户字体 GET/PUT |
| 其他 + `staticDir` | `serveStatic` | SPA 静态资源与 index 回退 |
| 其他 | 404 | — |

注意：**`/files` 与 `/blob` 不是“文件浏览 API”，而是“给渲染器取原始字节”的两条窄通道**。
元数据、目录列表、diff 文本都走 WS；只有“我需要这棵树 / 这个提交的这串字节”才走 HTTP。

### 6.2.4 `stop()` 与 `shutdown()`：两个不同的东西

公开接口里同时有 `stop` 与 `shutdown`，它们不是别名：

| | `stop()` | `shutdown()` |
| --- | --- | --- |
| 语义 | 同步释放资源，面向低层测试 | 幂等、有界的生产生命周期 |
| 是否 await 会话收敛 | 否 | 是：`settleSessionsForShutdown()` |
| 是否排空 turn 回执 | 否 | 是：`turnTracker.drain()` |
| 是否停分析 | 发起 `void shutdownAnalytics()` | await `shutdownAnalytics()` |
| 重复调用 | `stopping` 卫兵，第二次直接返回 | `shutdownPromise ??=` 记忆化 |
| 调用者 | 测试、`bootHost` 的信号包装 | **每个启动器都必须 await** |

`stop()` 的实际清理顺序（[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L983-L1019)）：

```
标记 stopping
→ 清评审计时器 / 清三个观察者 / 解除日志错误观察者
→ void shutdownAnalytics()
→ 停 host update 检查
→ cancelAllLogins / cancelAllMcpProbes / stopJbcentralRuntime
→ stopAllWatches / 用户配置/语言/主题/字体 watcher
→ disposeAllSessions
→ 清 reapTimers / sockets / 背压对账器 / 重放缓存
→ persistTerminalSessions() → closeAllTerminals()
→ 清各发布者接缝
→ server.stop(true)
```

`shutdown()` 只是在其前面加了三件事：

```ts
shutdownPromise ??= (async () => {
	stopHostUpdateChecks();
	await Promise.allSettled([settleSessionsForShutdown(), shutdownAnalytics()]);
	await turnTracker.drain();
	stop();
})();
```

> 规则：**每个宿主进程只优雅关闭一次。** 不要把 `shutdown()` 当成“可反复调用的 reset”。

### 6.2.5 崩溃日志：只救第一次

[crashLog.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/crashLog.ts) 的策略很简单：

- `uncaughtException` / `unhandledRejection` → 追加到 `<dataDir>/logs/crash.log` + stderr；
- **只有第一个故障**会 await（≤1 秒）`reportFatalError`，然后 `exit(1)`；
- `NODE_ENV=test` 不安装。

为什么“只救第一次”：第一个异常之后的进程状态已经不可信；继续上报只会制造噪声，
甚至可能在错误路径里再抛错。**崩溃可诊断，但不试图恢复。**

### 6.2.6 多宿主共用 `dataDir` 的现实

仓库明确承认：**多个宿主可以共享一个数据目录，但没有跨进程的状态协调。**

这意味着：

- JSON 文件写入是“原子替换”，但并发写入是**最后写入者胜出**；
- 同一个聊天不会被两个宿主同时活跃（pi 转录已经假设了这一点）；
- 终端成员关系、`turns.json`、评审文件都可能被另一个宿主覆盖；
- 不要试图在服务端加跨进程锁——那不在当前产品模型里。

---

## 6.3 HTTP 表面：两条字节通道

### 6.3.1 `/files`：工作树当前字节

[fileRoutes.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/fileRoutes.ts#L52-L69)：

1. 解析 `/files/<workspaceId>/<relpath>`，两段都要 URL 解码；
2. `resolveWorktreeFile(workspaceId, relPath)` —— 路径必须落在工作树内；
3. `statSync` 必须是文件；
4. 读前 `CONTENT_SNIFF_BYTES`（8 KiB）做字节分类；
5. 返回 `Bun.file(abs)` 流，头由 `contentHeaders(...)` 生成。

响应头：

| 头 | 值 | 原因 |
| --- | --- | --- |
| `Content-Type` | 字节分类优先，文件名回退，最后 `application/octet-stream` | **扩展名永不胜过字节** |
| `Cache-Control` | `no-store` | 工作树会变，不能缓存 |
| `Content-Length` | `stat.size` | 已知长度 |
| `X-Content-Type-Options` | `nosniff` | 防止浏览器猜类型 |
| `Content-Security-Policy` | 仅活跃类型：`sandbox; default-src 'none'` | HTML/SVG 同源打开时不给执行能力 |

坏 id、路径逃逸、目录、缺失、读失败——全部统一 404，且带 `no-store`。
**不区分“没权限”与“不存在”**，避免把路径结构泄露给调用方。

### 6.3.2 `/blob`：历史提交中的不可变字节

[fileRoutes.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/fileRoutes.ts#L71-L97)：

1. `/blob/<workspaceId>/<oid>/<relpath>`，oid 必须是 40 或 64 位 hex；
2. 用 `loadWorkspaces()` 找 `worktreePath`，找不到 404；
3. **仍然调用 `resolveWorktreeFile`**——即使读的是历史，relpath 也不能逃逸；
4. `readBlobSizeAtAsync` 先查大小；超过 `BLOB_SIZE_LIMIT`（64 MiB）→ 413；
5. `readBlobStreamAtAsync` 只 await 8 KiB 探测头，其余流式；
6. 头：`Cache-Control: public, max-age=31536000, immutable`，无 `Content-Length`。

为什么 `/blob` 可以 `immutable`：**oid 就是内容哈希**。同一个 URL 永远对应同一串字节，
这是全仓库少有的“可以永久缓存”的资源。

为什么还要 8 KiB 探测头：`Content-Type` 必须来自字节分类；把整个 blob 读进内存再做分类，
会让一个 64 MiB 的 PDF 占满宿主内存。

### 6.3.3 `/fonts/`：GET 与 PUT 不是同一套权限

[fontRoutes.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/fontRoutes.ts)：

- `GET /fonts/<name>`：`fontFileKind(name) === "font"` 才服务；`Cache-Control: no-cache` + `nosniff`；
- `PUT /fonts/<name>`：必须带 `USER_FONT_UPLOAD_HEADER`，否则 403；集合字体（`.ttc/.otc`）→ 422；
  `Content-Length > MAX_FONT_FILE_BYTES` → 413；然后 `saveUserFont`；
- 其他方法 → 405 + `Allow: GET, PUT`。

**上传头是必要的**：同源 HTTP 端点如果没有额外头，任何能发简单请求的页面都可以往数据目录写文件。
自定义头让浏览器必须预检，而宿主不返回 CORS 头，于是跨源写被挡在预检阶段。

### 6.3.4 静态 SPA 的路径处理

[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L1050-L1058)：

```ts
async function serveStatic(pathname: string, staticDir: string): Promise<Response> {
	const safe = normalize(pathname).replace(/^(\.\.(\/|\\|$))+/, "");
	const requested = safe === "/" || safe === "" ? "index.html" : safe;
	const file = Bun.file(join(staticDir, requested));
	if (await file.exists()) return new Response(file);
	const index = Bun.file(join(staticDir, "index.html"));
	if (await index.exists()) return new Response(index);
	return new Response("not found", { status: 404 });
}
```

两个行为值得记住：

- 去掉开头的 `../` 片段：静态目录之外不可读；
- 找不到具体文件就回退 `index.html`：这是 SPA 客户端路由的前提。

---

## 6.4 WebSocket 请求生命周期

### 6.4.1 一帧请求的形状

宿主接受的 frame 只有三种：

| 形状 | 处理 |
| --- | --- |
| `{ ack: string[] }` | `requestReplays.acknowledge(clientKey, ids)`，报告 abandoned |
| `{ resume: string[] }` | `requestReplays.retain(clientKey, unresolvedIds)`，其余释放 |
| `{ id, method, params?, sessionId? }` | 计算 fingerprint，进入重放缓存并执行 |

解析失败、非对象、缺 `id`/`method`、类型不对——**静默丢弃**，不回错误。
理由：这些帧要么来自损坏的客户端，要么来自探测；回错误只会放大问题。

### 6.4.2 fingerprint：同一个 id 必须对应同一个请求

[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L480-L482)：

```ts
const fingerprint = createHash("sha256")
	.update(JSON.stringify([method, params, sessionId ?? null]))
	.digest("hex");
```

重放缓存以 `(clientKey, requestId)` 为键。若同一个 id 第二次出现但 fingerprint 不同：

```
RequestReplayConflictError: request id "..." was reused with a different payload
```

这是客户端 bug 的硬信号，**不能静默地“以最后一次为准”**，否则重试语义会变得不可推理。

### 6.4.3 重放缓存：保留、放弃、溢出

[requestReplayCache.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/requestReplayCache.ts) 的模型：

| 概念 | 含义 |
| --- | --- |
| settled | handler 已经 resolve/reject |
| pending | handler 仍在跑 |
| abandoned | 页面在 handler 完成前就 ack 掉了它 |
| weight | 字符串结果按长度计，非字符串计 1 |

三个上限（构造函数默认值）：

| 上限 | 默认 | 触发行为 |
| --- | --- | --- |
| `maxRequestsPerClient` | 512 个 live 请求 | 新请求抛 `REQUEST_REPLAY_OVERFLOW` |
| `maxWeightPerClient` | 16 MiB 结果 | 已结算结果**保留条目但丢弃值**，重试抛 `RequestReplayUnretainedError` |
| `maxTrackedPerClient` | 2048 个条目（含 abandoned） | 同样视为命名空间满 |

两个关键语义：

1. **缓存内容永不主动驱逐**：只有 `ack` / `resume` / `clearClient` 会让条目消失。
   这保证“已发出的请求结果”不会因为宿主内部策略而消失。
2. **`clearClient` 可能拒绝回收**：只要有任意未结算条目，返回 `false`，宿主重新计时 60 秒再试。
   这避免一个还在跑的 handler 在 socket 关闭后失去归属。

溢出时的处理（[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L513-L527)）：

- 若是 `RequestReplayOverflowError` 且带 `load`，日志记录“哪些方法占满了命名空间”；
- 分析上报按**最重的方法**归类；
- 仍然给客户端回 `{ id, ok: false, error, errorCode? }`。

### 6.4.4 `ackSend`：为什么长轮次请求也要先回 ack

[ackSend.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/ackSend.ts)：

```ts
export const SEND_ACK_MS = 10_000;

export async function ackSend(run: Promise<void>, windowMs: number = SEND_ACK_MS): Promise<void> {
	let acked = false;
	const guarded = run.catch((err) => {
		if (!acked) throw err;
	});
	await Promise.race([guarded, new Promise<void>((resolve) => setTimeout(resolve, windowMs))]);
	acked = true;
}
```

它解决的是两个看似矛盾的需求：

- 客户端希望 `session.prompt` 很快返回，不要等整个 agent 轮次结束（可能几分钟）；
- 但 **真正的失败**（例如 pi 拒绝启动）必须传回客户端。

做法：最多等 10 秒。10 秒内 handler 抛错 → 错误沿方法结果返回；10 秒后 handler 仍在跑 → 先返回 `{ ok: true }`，
后续失败通过事件流表达。`acked` 标记保证**不会出现“已经 ack 了又抛错”**。

调用点（[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L335-L353)）：

```ts
async function sendUserMessage(mode, sessionId, text, clientKey, operation) {
	const control = isControlMessage(text);
	const provider = control ? undefined : sessionProviderAnalytics(sessionId);
	await ackSend(runObservation.send(sessionId, control ? "internal" : "user", operation));
	...
	return { ok: true };
}
```

`session.prompt` / `session.steer` / `session.followUp` / `session.respondToTool` 都走这条路径。

### 6.4.5 类型化注册表：漏一个方法就编译失败

[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L318-L323)：

```ts
type WsHandler<M extends WsMethodName> = (
	params: WsParams<M>,
	ctx: RequestContext,
) => WsResult<M> | Promise<WsResult<M>>;

type WsHandlers = { [M in WsMethodName]: WsHandler<M> };
```

`WsHandlers` 是**映射类型**，`WsMethodName` 每多一个方法，对象字面量就必须多一个实现；
参数类型、返回类型也由方法名自动推导。于是：

- 缺方法 → typecheck 失败；
- 多方法 → typecheck 失败；
- 参数/返回形状错 → typecheck 失败。

入口只有一个：

```ts
export async function handleRequest(method, params, ctx) {
	if (!isWsMethod(method)) throw new Error(`Unknown method: ${method}`);
	return dispatch(method, params as WsParams<typeof method>, ctx);
}
```

注意：它**只收窄 `unknown`，不做运行时形状校验**。参数形状是编译期契约 + 客户端同仓库生成类型的共同假设；
如果你在服务端加运行时校验，会得到一份与 `contracts` 重复的第二套规则。

### 6.4.6 协议版本：订阅在 `open` 时就分叉

`PROTOCOL_VERSION = 102`（[wsProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/wsProtocol.ts#L138)）。
但客户端可用能力不是一个整数，而是一组特性常量。

在 WS `open` 时，宿主按客户端版本决定**订阅哪些 channel**：

| 版本门禁 | 只有达到版本才订阅的 channel |
| --- | --- |
| `MCP_PROTOCOL_VERSION` | `mcp.status` |
| `LOCALES_PROTOCOL_VERSION` | `locales.changed` |
| `USER_THEMES_PROTOCOL_VERSION` | `themes.changed` |
| `PI_PACKAGES_PROTOCOL_VERSION` | `packages.changed` |
| `FONTS_PROTOCOL_VERSION` | `fonts.changed` |

此外，`hostUpdateAvailable` 只在服务端确实配置了 host update 时才订阅。

**为什么订阅也要门禁**：老客户端收到不认识的数据形状会崩，而 not-subscribed 比“收到再丢弃”更省带宽，
也更符合“客户端按能力降级”的契约。

请求层也有版本分支，例如：

- `workspace.openReview` / `workspace.branchStatus`：对早于 `WORKSPACE_SETTLE_PROTOCOL_VERSION` 的客户端，
  把已合并/已关闭的评审答为 `null`（[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L1520-L1535)）。

### 6.4.7 通道推送：广播，不是请求

`host` 用 `server.publish(channel, JSON.stringify({ channel, data }))` 扇出。要点：

- 推送**没有请求 id**，**不参与重放**；重放只覆盖请求应答；
- 订阅是 per-socket 的，但 channel 内容是全局的（除终端数据按 `clientKey` 归属）；
- `server.welcome` 是 push 形态的**快照**，客户端必须能据此重建初始状态。

常见失效提示型推送（payload 为空对象）——`locales.changed`、`themes.changed`、`packages.changed`、`fonts.changed`：
它们只说“重新读”，不带数据。这是有意的：**失效信号不携带数据，避免第二套缓存。**

### 6.4.8 终端背压：一个闩锁 + 两套解除

终端输出是唯一可能“瞬间灌满 socket”的通道。宿主的处理：

| 机制 | 位置 | 作用 |
| --- | --- | --- |
| `terminalBackpressured` 集合 | `server.ts` | 某个 `clientKey` 一旦发送返回非 OK，就加锁 |
| `drain` 回调 | `server.ts` | 快路径：缓冲区清空就解锁并恢复终端 |
| 1 秒对账器 | `server.ts` | 慢路径：`getBufferedAmount() === 0` 也解锁（drain 可能丢） |
| 发送状态判断 | `terminalSend.ts` | `delivered` / `backpressured` / `unavailable` |

chunk 级发送失败时，宿主不是“丢数据”，而是**暂停该客户端的终端恢复**（`resumeClientTerminals` 不再推进），
等到可写再继续。这保证 PTY 的字节顺序不因背压而乱序。

> 交叉参考：客户端收到 `pi.event` 后的 32 ms / 128 条批量在 **web 侧**的
> [piEventBatcher.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/piEventBatcher.ts)，
> 不在服务端。服务端只负责投影与发送。

---

## 6.5 `agent`：会话与事件

### 6.5.1 三层对象

| 层 | 对象 | 拥有者 | 说明 |
| --- | --- | --- | --- |
| 运行时 | `piRuntime`（`ModelRuntime` 世代） | `agent` | 模型目录、认证、Central 代理；可整体替换 |
| 会话 | pi `AgentSession` | `agentSessionManager` | 一个聊天；`session.sessionId` 为键 |
| 宿主条目 | `SessionEntry` | `agentSessionManager` | 包装 pi 会话 + 订阅 + 发布器 + 生命周期 |

关键纪律：**每个会话保留自己创建时的 runtime 世代。** 全局 runtime 换代时，已有会话不被静默迁移；
新会话才用新世代。否则一个正在跑的长任务会在中途换模型目录/认证，行为不可复现。

### 6.5.2 会话创建的真实顺序

[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts)
里的 `createSession`（约 1365 行）→ `prepareSessionEntry`（约 1121 行）→ `registerSession`（约 1317 行）：

```
resolve 模型/思考级别
→ 构建 settings（buildSessionSettings(cwd, projectTrusted)）
→ createAgentSession(...)              // pi 进程内会话
→ 订阅 pi 事件流
→ 绑定扩展 / 资源加载器 / 会话设置
→ registerSession                      // 登记 + 发布 session.created
→ publishEntryState                    // 首次状态
```

`createParentSessionInternal`（约 1416 行）是父子会话路径；`createParentSession`（约 1401 行）是它的公开包装。

失败时：**方法错误返回 + 清理半初始化资源**。不要留下一个“注册了但没有订阅”的条目。

### 6.5.3 四个会话动作的语义

| 方法 | 何时用 | 关键行为 |
| --- | --- | --- |
| `prompt` | 空闲时开始一个新轮次 | 流式中由 `promptSession` 检测并转 `steerEntry`（兜底，不是推荐） |
| `steer` | 运行中改变方向 | 立即插入当前轮次 |
| `followUp` | 运行中追加，等当前轮结束再跑 | 排队，不打断 |
| `abort` | 停止当前运行 | 可恢复排队项，视参数而定 |

`promptSession`（约 2284 行）、`steerEntry`（约 2271 行）、`followUpEntry`（约 2330 行）、
`abortSession`（约 2471 行）是具体入口。

> 注意：pi 原始 API 在流式期间调 `prompt()` 会抛错；ThinkRail 的包装器会自动转 steer。
> 但客户端仍应表达真实意图（steer vs followUp），不要把语义交给宿主猜。

### 6.5.4 事件投影：什么上线、什么不上线

`sessionEventProjection.ts` 把 pi 事件变成 `PiEvent`。原则：

| pi 内部字段 | 是否上线 | 原因 |
| --- | --- | --- |
| 文本/思考增量 | 是 | 渲染必需 |
| 工具调用 id / 名称 / 参数 | 是 | 工具卡渲染必需 |
| `tool_execution_update.partialResult` | 是（替换语义） | 增量输出 |
| `structuredContent` | **否** | pi 内部结构，体积大且随版本变 |
| 模型原始对象 | **否** | 用 `toWireModel` 允许列表收窄 |
| 成本/统计的内部推导 | **否** | pi 报什么就发什么 |

流式语义必须记牢：**`text_delta` / `thinking_delta` 追加；`tool_execution_update.partialResult` 替换。**

### 6.5.5 结算：`agent_end` 不是结束

- `agent_end` 是**一次尝试**结束；后面可能跟重试、压缩、恢复、排队续跑。
- `agent_settled` 才是**自动工作真的结束**，`turns` 在这里做第二次快照。

`settledLifecycle.ts` 与 `agent_end` 的终止元数据会被附加到 `agent_settled`，这样客户端既能知道
“这一轮怎么结束的”，也能知道“整个自动工作何时停”。

### 6.5.6 模型上线：`toWireModel` 是允许列表

`Model` 是 pi/运行时对象，字段多且不稳定。**绝不原样上线**：

- `toWireModel` 只暴露客户端需要的标识、能力、上下文窗口等；
- 入站的模型引用用 `resolveWireModel` 重新解析，不信任客户端塞进来的对象；
- `settledAvailableModels` 是唯一读取可用模型的入口，**绝不用 `getAvailable()`**。

`modelAvailability` 还会按认证来源收窄：例如 ChatGPT OAuth 只留 `openai-codex`；
Central 只留代理 `/models` 公布的 id。

### 6.5.7 重开磁盘会话：先修悬空工具调用

`repairDanglingToolCalls` 在重开前运行。原因：一个崩溃/强杀可能留下“工具调用已发出但没有结果”的转录；
pi 的后续推断会因此卡住或产生错误上下文。修复后再 `parseSessionEntries → migrateSessionEntries → buildSessionContext`。

### 6.5.8 删除与墓碑

`session.delete` 不是“立刻抹掉”：

- 转录通过 `trash` 做**可恢复移动**；
- 会话条目从注册表移除；
- 发布 `session.deleted`。

这解释了 `agent → trash` 的依赖边：**唯一的用户文件销毁路径必须是操作系统废纸篓，不能 `unlink`。**

---

## 6.6 `persistence`：数据目录的规则

### 6.6.1 一个目录，所有领域状态

[persistence.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/persistence/persistence.ts#L48-L49)：

```ts
export function dataDir(): string {
	return resolve(process.env.THINKRAIL_DATA_DIR ?? join(homedir(), ".thinkrail"));
}
```

永远是绝对路径：相对路径也 `resolve` 成绝对。这样所有模块拼出来的文件路径不会因为 cwd 变化而漂移。

### 6.6.2 文件清单

| 文件 | 拥有什么 | 谁读 |
| --- | --- | --- |
| 项目/工作区记录 | 注册表、成员关系、字段真值 | `projects`、`workspaces` |
| 配置 | `AppConfig` | `settings`、`host` |
| `session-lifecycle.json` | 最近已结算 completion + 显式取消的 run id | `agent` |
| `session-receipts.json` | 首次安装基线 + 会话→已处理 completion id | `agent` |
| `turns.json` | 每个工作区的运行改动集 | `turns` |
| `installation.json` | 安装 UUID | `analytics` |
| `attribution.json` | 浏览器归因 claim | `host` / 归因流程 |
| `environment.json` | 宿主环境（0600） | `environment` |

**安装 UUID 只存在于服务端，绝不上线。**

### 6.6.3 原子替换与损坏恢复

- 写入：先写临时文件，再 `renameSync` 覆盖目标（[persistence.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/persistence/persistence.ts#L76)）；
- 读取失败：尽力把文件重命名为 `<file>.corrupt-<epoch-ms>`，然后**按缺失处理**
  （[persistence.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/persistence/persistence.ts#L107-L109)）；
- 损坏**绝不阻塞启动**。

### 6.6.4 加载从不写入 / 保存不做归一化

两条容易违反的规则：

1. **加载路径只读**。读时可以做路径拼写自愈（例如修正历史拼写），但不回写。这样“打开应用”不会产生磁盘副作用。
2. **保存路径不做归一化**。保存就是把调用方给的值写下去；规范化是领域模块的责任。
   如果保存时偷偷修正，调用方永远不知道自己给了错值。

### 6.6.5 `session-lifecycle` 与 `session-receipts` 的分工

这两个文件很容易混：

| | `session-lifecycle.json` | `session-receipts.json` |
| --- | --- | --- |
| 回答的问题 | 最近哪个 completion 已结算？哪个 run 被显式取消？ | 安装基线是哪次？这个会话处理过哪些 completion？ |
| 用途 | 重启后不重复处理同一结算 | 防止历史 completion 被当成“新工作”再跑一遍 |
| 何时写 | 结算/取消时 | 首次安装 + 每次处理完成 |

### 6.6.6 配置校验：只校验它拥有的部分

`persistence` 逐字段用 `DEFAULT_CONFIG` 校验，但**保留未知顶层字段**。这是为了前向兼容：
老宿主读到新宿主写的配置时，不认识的键不应被丢掉。

### 6.6.7 并发模型：最后写入者胜出

没有跨进程锁。多个宿主共用一个 `dataDir` 时，最后写入者覆盖前者。
这是被接受的限制，不是待修的 bug——因为产品模型假设同一聊天只有一个活跃宿主。

---

## 6.7 `settings` 与 `environment`：两个不同的“设置”

### 6.7.1 `settings` 是同步的宿主配置

[settings](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/settings/index.ts) 的公开面：

| 函数 | 作用 |
| --- | --- |
| `getConfig()` | 返回当前完整配置 |
| `updateConfig(partial)` | 合并 → 校验 → 持久化 → **发布完整合并配置 + 已应用部分更新** |
| `noteRecentModel(model)` | 记录最近使用模型（宿主拥有） |
| `setSettingsPublisher(fn)` | 注入推送接缝 |
| `resetConfigCache()` | 测试/重载用 |

`updateConfig` 的关键顺序：

```
拒绝非对象 / 未知键
→ 合并
→ 校验（夹取有界字段、拒绝非法值）
→ 持久化
→ setSettingsPublisher(完整配置, 已应用更新)
```

**先持久化再广播**：客户端收到 `settings.changed` 时，磁盘已经一致。

### 6.7.2 哪些字段归宿主、哪些归客户端

| 字段 | 归属 | 规则 |
| --- | --- | --- |
| `recentModels` | 宿主 | 客户端提供会被丢弃 |
| `favoriteModels` | 客户端整列表写入 | 宿主只存 |
| `disabledExtensions` | 宿主 | 未知名字保留，便于重新启用 |
| `theme` / `themeMode` | 客户端解析 | 宿主不校验主题内容 |
| `locale` | 客户端解析 | 默认 `"system"` |
| `chatLineWidth` / `fileLineWidth` | 宿主校验范围 40–240 | 有界 |
| `subagentMaxConcurrent` | 宿主校验 1–16，不夹取 | 超范围直接拒绝 |
| JetBrains 配额间隔 | 宿主校验 1–3600 秒 | 默认 30 秒 |
| `defaultEffort` / `reviewEffort` | 仅凭线上类型通过 | 不额外校验 |

**退役字段静默丢弃**：更新键是封闭集合，已存键不封闭。这样客户端永远不能通过 `update` 写入一个已退役字段。

### 6.7.3 `environment` 是启动时的一次性动作

[environment](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/environment/index.ts)：

- 存储：可选的 pi 配置文件夹（`PI_CODING_AGENT_DIR`）+ 有序 `NAME=value` 变量；
- 应用：`applyHostEnvironment()` **只由 `bootHost` 调用一次**，在 `resolveShellEnv()` 之后、
  `initializeJbcentralRuntime()` / `createServer()` 之前；
- 优先级：**启动环境 > 已存记录 > 登录 shell 导入 / locale 修补**；
- 变更 = 重启（`restartRequired`），因为环境变量已经进入 pi 与子进程；
- `environment.json` 原子写、0600；
- 只由 `environment.get` 获取，**永不广播**——环境变量可能含密钥材料。

> 为什么 `environment` 绝不依赖 `settings`：两者都“像配置”，但一个在启动时应用一次并需要重启，
> 另一个在运行中同步更新。把它们合并会让 `settings.update` 有机会修改需要重启的字段，语义就乱了。

---

## 6.8 `projects` 与 `workspaces`

### 6.8.1 项目注册表

`projects` 拥有“已知仓库”的稳定身份：

| 行为 | 规则 |
| --- | --- |
| 打开路径 | 先对宿主文件系统解析；绝对路径通过、`~` 展开、相对路径拒绝 |
| 判定仓库 | `git rev-parse --show-toplevel`；非仓库 → `NOT_GIT` |
| 去重 | 按仓库根去重；稳定唯一 slug |
| 重复打开 | 若根已作为某工作区的 `worktreePath` 持有 → `ALREADY_OPEN` |
| 关闭 | 只改视图，不删记录（可无损重开） |
| 重命名 | 只改 `name` |
| 初始化 | `git init` + `git add -A` + 允许空的首次提交 |
| 克隆 | 拒绝远端辅助程序、内嵌密码、userinfo、查询、片段、反斜杠；独占 mkdir；30 分钟期限 |

项目级还拥有：技能准入、`piResourceTrust`、`mcpApprovals`、`mcpOverrides`、`mcpToolGrants`。

### 6.8.2 工作区是什么

**工作区 = 自己分支上的 git worktree。** 两种 `kind`：

| kind | 含义 | 限制 |
| --- | --- | --- |
| `default` | 每个项目恰好一个，`worktreePath` = 项目文件夹本身 | 不可移除、不可重命名 |
| `external` | 原地记录一个已有 worktree | 从不创建、从不改动 |

`name`（Title Case 显示名）与 `branch`（kebab slug）**解耦**：
用户可以给工作区起一个漂亮名字，而分支名保持 git 友好。

### 6.8.3 `baseBranch` 与 `diffBase`：两个不同的基准

这是本章最值得记住的一组字段：

| 字段 | 含义 | 用途 |
| --- | --- | --- |
| `baseBranch` | 工作区**创建时**的来源分支 | 记录血统、推送基点保护、默认 diff 参考 |
| `diffBase` | 评审/差异**对照**的目标 | 用户可以把它改成 `main`、某个提交或另一个分支 |

`diffBaseRef(ws) = diffBase ?? baseBranch` 是唯一的塌缩点（[git](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/git/index.ts)）。
不要把两者合并成一个字段——一个回答“我从哪来”，一个回答“我在跟谁比”。

### 6.8.4 创建工作区的真实流程

`createWorkspace` 是异步的，核心步骤：

```
确定 base ref（用户输入经 assertSafeRef）
→ worktree add -b <branch> --no-track <full-refs/remotes/<base>>
→ 路径 dataDir/worktrees/<project-slug>/<branch>
→ 写 scratch 目录 + 自忽略 .gitignore
→ 持久化记录（baseBranch = 创建来源）
→ 发 workspace.created
```

细节：

- **守卫看 ref 是否存在，不看退出码**：git 的退出码不足以区分“ref 不存在”与“其他失败”；
- 探针与 `worktree add` 都用完整 `refs/remotes/<base>`，避免短名歧义；
- `--no-track`：新分支不自动跟踪远程，否则推送语义会变得意外；
- 远程 ref 新鲜度在关键路径外 `prefetch`；必要时回退 `git fetch`（走 `gitAsync`）；
- 用户提供名字时，名字即显示名，并置 `renamed: true`。

### 6.8.5 重命名与移除

`renameWorkspace`：

- 带 `opts.branch` → `git branch -m`（改分支名）；
- **工作树目录永不移动**（路径身份比显示名稳定）；
- 不带 `opts.branch` → 只改显示标签。

移除分三个原语：

| 原语 | 速度 | 做什么 |
| --- | --- | --- |
| `forgetWorkspace` | 快 | 丢弃注册记录，立即返回 |
| `reclaimWorktree` | 慢 | `git worktree remove --force`；失败回退 rm + prune |
| `removeWorkspace` | 同步组合 | 先 forget，再后台 reclaim |

`host` 的 `workspace.remove`（[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L717-L730))：

```
forgetWorkspace
→ evictSpecIndex / removeWorkspaceReviews / forgetWorkspaceChanges / forgetWorkspaceTurns
→ stopWatch / closeWorkspaceTerminals
→ void archiveTeardown(ws)
```

先同步释放所有“指向这个工作区”的内存结构，再把慢的磁盘回收放到后台——**请求不必等 `git worktree remove`。**

`reclaimWorktree` 拒绝两种用户自有类型（default/external）以及解析到项目文件夹的路径。

### 6.8.6 列表：异步读取后的重读纪律

`listWorkspaces` 做了一件很值得学的事：

```
确保 Default 工作区存在
→ 对账文件夹真值
→ 并行 await gitAsync 取 diff stats
→ await 之后重新读一次成员关系
```

为什么 await 后要重读：diff stats 是慢的异步 git 调用；期间工作区可能被移除/重命名。
用 await 前的成员关系去拼结果，会把已删除工作区的统计挂回列表。

Diff stats 缓存键是 `worktreePath/baseBranch/diffBase`（`workspaceDiffKey`）——三者任一变化都失效。

### 6.8.7 Default 工作区的 CQS 例外

`listWorkspaces` 名义上是查询，但它会**确保 Default 工作区存在**：必要时执行一次注册表写入 + 生命周期发出。
这是刻意的 CQS 例外，因为“每个项目恰好一个 default”是产品不变量，不能靠调用方记得先创建。

同样，字段的 folder-truth 在列表时刷新并 `updated`。自愈带外变化（用户手动删了文件夹/改了分支）
是 Default 工作区的职责，不是 UI 的。

### 6.8.8 scratch 目录与初始终端

- `ensureWorkspaceScratchDir`：工作区根缺失时抛出；拒绝符号链接组件（`lstat`）；`.gitignore` 用 `wx` 仅缺失时创建。
- `initialTerminalPending: true` → `host` 保留一个确定性终端（无进程）→ 首个 attach 时 `completeInitialTerminalReservation`。
  这样 UI 打开工作区时标签页立即存在，但 PTY 只在实际使用时创建。

---

## 6.9 `git`：事实层的纪律

### 6.9.1 同步 runner 与异步 runner

| runner | 用在什么路径 | 原因 |
| --- | --- | --- |
| `git(cwd, args)` / `gitBytes` | 写入者、微管道叶子 | 简单、快，不占事件循环太久 |
| `gitAsync` / `gitAsyncBytes` | 请求路径的读取 | 请求不能阻塞主线程 |
| `gitAsyncStream` | 大 blob | 流式，不在内存里攒整份 |

**请求路径读取都走异步**：`resolveDiffRange`、`gitStatus`、`gitDiffFile`、`listCommits`、`listBranches`、diff stats。
同步 runner 保留给：`currentBranch`、`tryCurrentBranch`、`resolveDefaultBranch`、`resolveCommitOid`、
`readBlobAt`、`gitHeadSha`，以及 `gitUncommittedPaths` 的生命周期例外。

### 6.9.2 网络 git 的 55 秒预算

- 网络 git 默认 `timeoutMs = 55s`，贴近客户端 60s 通用超时；
- 到期杀子进程，错误形如 `timed out after <actual>s — the remote never answered`；
- `nonInteractiveGitEnv()` 设 `GIT_TERMINAL_PROMPT=0`，否则 git 可能停下来等密码。

**为什么是 55 而不是 60**：让带 git stderr 的具体错误先到客户端，而不是被客户端的通用超时吞掉。

### 6.9.3 `resolveDiffRange`：唯一的 scope 定义

| scope | 范围 | 特殊处理 |
| --- | --- | --- |
| `branch` | merge-base(base, HEAD)..HEAD + untracked | 评审默认 |
| `uncommitted` | HEAD..工作树 + untracked | 只看未提交 |
| `commit` | `sha^..sha` | 根提交降级为 `git show --format=` |
| `pinned` | 不可变 oid..工作树 + untracked | 锁定到某提交 |
| `turn` | baseTree..headTree | 两个 oid 都校验 `^{tree}` |

**只有“修改侧是工作树”的 scope 可变**；`commit` 是只读的，回滚它会得到 `SCOPE_IMMUTABLE`。

`changedFileArgs` 在前后都界定 rev：`--end-of-options` + `--`。这防止以 `-` 开头的路径被当成选项。

### 6.9.4 ref 安全与 commit 校验

- `isSafeRef` / `assertSafeRef` 在进程内复现 `git check-ref-format` 的规则；
- 用户输入的 ref 先过 `assertSafeRef` 才到 git；
- `commit` sha 做双重校验：hex 形状 + `rev-parse --verify`；
- 消失的提交抛 `CodedError("UNKNOWN_COMMIT")`，不靠 message 判断。

### 6.9.5 快照与提交

`snapshotWorktree`：

- 用临时 `GIT_INDEX_FILE`，不污染真实索引；
- `git add --ignore-errors`；
- 只有 `fatal:` / 超时 / 启动失败才返回 `null`；普通路径错误不应让整次快照作废；
- 返回 tree oid。

`gitCommitPaths`：

- 只提交指定 paths，`--literal-pathspecs`；
- `--no-verify`（不触发用户钩子，避免 hook 改变产品语义）；
- 快照并恢复真实 index 文件；
- 返回 `{ sha } | null`。

### 6.9.6 blob 流与 diff 文本

- `readBlobStreamAtAsync`：只 await 8 KiB 探测头；取消时杀 git；
- `gitDiffFile`：两侧按字节读取 + `ResourceMeta`；只有文本才解码；
  纯字节侧的 `content` 是 `""`，由 `/files` / `/blob` 提供真实字节；
- `gitStatus` 用 `--numstat` 取行数；未跟踪文本才计新增；路径用 NUL 安全解析；
- `listCommits` 用 NUL 分隔固定元数，并剥掉控制字符与不可见欺骗字符。

### 6.9.7 `gitCommitPaths` 与 `todos` 的协作

`todos` 完成一个步骤时提交“这个步骤的增量”，而不是整个工作树。它依赖：

- `gitUncommittedPaths` 记录 `in_progress` 打开窗口的基线；
- `gitCommitPaths` 只提交窗口内的路径；
- 四道门禁：已记录基线、无残留外来脏改动、窗口未共享、非空增量。

---

## 6.10 `fs` / `changes` / `turns` / `watch` / `spec`

### 6.10.1 `fs`：路径受限 + 唯一字节分类

公开面：

| 函数 | 作用 |
| --- | --- |
| `readDir(workspaceId, path)` | 目录列表；`.git` 隐藏；目录在前；线上 POSIX 相对路径 |
| `readFile(workspaceId, path)` | `{ content, meta }`；仅文本解码 |
| `resolveWorktreeFile(workspaceId, path)` | 被包含的绝对路径；拒词法逃逸、`.git`、逃逸符号链接 |
| `workspaceRelativePath(root, path)` | 唯一转线上相对路径的地方 |
| `classifyBytes` / `hashBytes` / `decodeText` / `mimeFromPath` / `resourceMeta` | 字节身份 |

`classifyBytes` 的 8 KiB 头能识别：png/jpeg/gif/webp/avif/bmp/ico/pdf/zip/gzip/woff/woff2、
SVG 文本、git-lfs 指针；`text = 无魔数 + 无 NUL + 严格 UTF-8`。

**两条必须做对的规则**：

1. 一处分类、一个哈希：`git` 的 diff 侧与 `fs.readFile` 必须报告同一种分类；
2. 扩展名永不胜过字节。

### 6.10.2 `changes`：CAS + 回执

`revertChange` / `undoChange` 的顺序值得背下来：

```
await 加载（scope 解析 + 有界历史字节读取）
→ 同步 CAS 校验（originalHash / modifiedHash）
→ 写入趟
```

为什么“先 await 再同步”：CAS 校验与写入必须原子，中间不能有 await，否则另一个客户端可能在你读完之后改了文件。

- 两侧哈希不匹配 → `STALE_VIEW`；
- 符号链接 / 仅模式变化 → `UNSUPPORTED_CHANGE`；
- 整文件回退按状态分：两侧在 → 写原始字节；原始缺失 → 重命名到隐藏占用名 + 移入废纸篓；
  工作树缺失 → 恢复原始字节。

回执（`ChangeReceipt`）就是撤销令牌：

- 每个工作区 ≤20 条、≤64 MiB，保留最新总和；
- `RECEIPT_UNKNOWN` 表示令牌已不在环里；
- 回执**只存宿主内存**，不落盘。

### 6.10.3 `turns`：运行边界的唯一答案

`TurnTracker` 的模型：

```
agent_start  → snapshotWorktree（base tree）
agent_settled → snapshotWorktree（head tree）
两 tree 不同 → 记录 TurnChangeSet（两 oid + 墙钟 + turn 作用域 gitStatus）
→ 持久化 turns.json
→ 推 turn.changed
```

关键点：

- 运行边界是 `agent_start → agent_settled`，**不是 `turn_*`**；
- 每个工作区上限 30 条；
- 零变更不记录；
- 第一个 `agent_start` 胜出（一个工作区同时只应有一个运行边界）；
- `drain()` 有 5 秒上限，关闭时不能无限等快照；
- `turns` 从不 import `agent`：`host` 把事件流喂给它，并把会话解析到工作区。

### 6.10.4 `watch`：失效信号，不是数据

`workspace.fsChanged` 的 payload：`{ workspaceId, paths, truncated, skillChange }`。它是**失效信号**，
客户端收到后重新读，不把它当数据源。

合并规则：

- 忽略 `.git` / `node_modules` / `.DS_Store`；
- 300 ms 静默合并 / 1 s 最大延迟 / 100 路径上限；
- `skillChange: none | detected | unknown`。

启动信号：新 watcher 注册窗口（约 750 ms）后发布一次合成的 truncated/技能未知通配，
保证“启动前已存在的文件”也能让客户端刷新。`workspace.watchReady` 是预检。

预热层全局上限 8（`ensureWatch(workspaceId, { prewarm: true })`）；真实调用永久提升。

仓库元数据信号：`.git` 事件或工作树 git 目录变化 → 无路径的 `setRepoMetaPublisher(workspaceId)`；
`host` 接到后做 `recordWorkspaceHead` + `refreshUserOwnedWorkspace` + 空路径 `fsChanged`。

`watch` 每次 `ensureWatch` 重新 stat 根；inode 变化重建；回收僵尸 watcher；失败启动下次重试。

### 6.10.5 `spec`：同步读取的规格图

`specGraph(workspaceId) → SpecGraphSnapshot` 是同步读取，按工作区复用 `SpecIndex`；
`evictSpecIndex` 在工作区移除时调用；`projectHasSpecs(root)` 只认持久规格（`task-spec` 不算）。

同步是可以的，因为规格文件数量有限，且结果按工作区缓存；但工作区一变就必须 evict，
否则客户端会看到上一个分支的图。

---

## 6.11 `terminal` / `subprocess` / `trash` / `dialog` / `editors`

### 6.11.1 `terminal`：PTY 是工作区资源

| 概念 | 规则 |
| --- | --- |
| 键 | `(workspaceId, tabKey)` |
| PTY id | 逐次运行生成，**永不持久化** |
| `reserveTerminal` | 幂等记录目录，不生成进程 |
| `attachTerminal` | 幂等 get-or-create，同步 |
| 附着 | 独占带接管；前客户端收 `terminal.detached` |
| 写入/resize | 只有已附加客户端可以 |
| 目录上限 | 256 标签页；key ≤500；标题 ≤1000 |
| 输入 | 字符串；cols/rows 1–32767 |

shell 选择：

- `SHELL` 环境变量优先；
- Windows 用 `terminalWindowsShell`（`auto`：pwsh → powershell；`pwsh`/`powershell`/`cmd` 字面钉子）；
- 非 Windows 用 `/bin/bash`；
- macOS 加 `-l`。

PTY 环境会过滤启动器变量、pi 标记、`THINKRAIL_*`、`PICK` 等；保留 `NO_COLOR` 与用户 `PI_*`；
Unix 的 `USER` / `LOGNAME` 从 `id -un` 盖章。

跨宿主重启是**复活**：`persistTerminalSessions()` → 重启后 `reviveTerminalSessions()`；
成员关系每次变更持久化；记录原始字节、重放模式（除鼠标跟踪）、不记 alt screen。

### 6.11.2 `subprocess`：完成 = 退出，不是 EOF

`runBounded(argv, { timeoutMs, cwd?, env?, stdout?, signal? })` 返回：

```
{ ok, out, err, timedOut, launchFailed, waitedMs, aborted? }
```

最容易错的一点：**完成是子进程退出，不是管道 EOF。** 一个后台孙进程可能持有 stdout 管道，
如果等 EOF，会永远等下去。

- POSIX：分离进程组；到期组杀；
- Windows：`taskkill /PID <pid> /T /F`（5 秒）；
- 超时也排空输出（保留 git stderr 诊断）；
- `DRAIN_GRACE_MS` 250 ms 在所有路径；
- `timeoutMs` 夹到 `[0, 2^31-1]`；
- env 默认活 `process.env`；stdin 为 `ignore`；计时器 unref；两路流不设上限。

### 6.11.3 `trash`：唯一的销毁路径

`trashFile(path, implementation?)` 是**唯一**允许销毁用户文件的入口：

- 启动器通过 `setBundledTrashHelpers` 注入平台辅助程序；
- 测试通过 `setTrashImplementationForTests`；
- **禁止 `unlink` 回退**——删不掉就报错，不要偷偷永久删除。

消费者只有两个：`agent`（删聊天转录）与 `changes`（整文件回退）。

### 6.11.4 `dialog` 与 `editors`

`dialog.selectDirectory()`：macOS `osascript`、Linux `zenity`/`kdialog`、Windows PowerShell `FolderBrowserDialog`；
取消 → `null`；退出码分类；无图形会话提前失败；`THINKRAIL_PICK_DIR` 可覆盖。

`editors`：

- `listAvailableEditors()` 用 `Bun.which` 探测固定候选清单；
- **必须显式传 `{ PATH: process.env.PATH }`**：`Bun.which` 默认读进程启动时的 PATH 快照；
- `openEditor(id, worktreePath)` 重新检查 PATH 后分离启动；
- `revealInFileManager(worktreePath)` 发后不管（`explorer` 成功也可能非零退出）；
- Vim 是 `kind: "terminal"`，由客户端在内嵌终端处理。

---

## 6.12 `reviews` 与 `todos`：两个带状态机的领域

### 6.12.1 评审：快照 + 锚点 + 锁

每个工作区一个打开 `Review`，惰性单飞创建。核心字段：

| 字段 | 含义 |
| --- | --- |
| `baseSha` | 分支范围分叉点的完整提交 oid |
| `fileSessions` | key → sessionId；整体备注 key 为 `""` |
| `doneFiles` | 已标记完成的文件 |
| 评论 `status` | `draft → sent → resolved / dismissed` |

锚点 `ReviewAnchor` = path + side + contentHash + selectors（lineRange/textQuote/structural/region/diffHunk）。
状态：

| 状态 | 何时 |
| --- | --- |
| `anchored` | 仍精确命中 |
| `moved` | 内容还在但位置变了；**粘性** |
| `outdated` | 纯字节 / 有位置但找不到 |

发送流程**在 `host` 组装**（`requestReview.ts` + `handlers.ts`）：

```
withReviewLock(workspaceId)
→ 重新锚定
→ 渲染上下文包（动态属性实体转义）
→ agent.createSession / followUp
→ markSent
→ 分离 prompt
```

`rollbackSend` 在失败时回滚；`resolve_comment` 工具通过 `setReviewCommentHandler` 接缝回调，
并绑定 `sessionId`。

**`review.close` = Clear**：归档到 `reviews/archive/<workspaceId>/<reviewId>.json`，不删除。

### 6.12.2 计划：读-改-写 + 提交门禁

`todos` 的计划文件是 `.thinkrail/context/todos/<sessionId>.json`，经 `pi-todos/core` 的 `TodoStore` 读-改-写。
评审 sidecar 是 `<sessionId>.reviews.json`。

变更工件：

```
in_progress 打开窗口：
  gitUncommittedPaths + head → .baselines.json

done 时：
  gitCommitPaths 提交增量 + 收编窗口内提交
```

四道提交门禁：

1. 已记录基线；
2. 无残留外来脏改动；
3. 窗口未共享；
4. 非空增量。

评审状态：

- `reviewedShas` 水位线；
- `pending` 标记；
- `approveTodoReview` / `requestTodoFix` / `rollbackTodoFix`；
- `commitReviewTransition` 一次快照写入；
- **1 次自动周期上限**，防止评审-修复无限循环；
- `clearAllPendingReviews` 启动清扫。

`listTodos` 有三个装饰：`commit.files`、`unattributed`、`adoptedCommits`；读屏障 `settleChangeArtifacts`。

---

## 6.13 `pr` / `branch-review` / `github`

### 6.13.1 三个模块的分工

| 模块 | 回答的问题 | 不做什么 |
| --- | --- | --- |
| `github` | 本机 `gh` 安装/登录状态如何？ | 不碰 provider HTTP API，不存 token |
| `branch-review` | 这个分支有没有 PR/MR？状态/检查如何？ | 不持久化，不 import `workspaces` |
| `pr` | 推送、打开/更新 PR、ready/merge/reopen、删远端分支 | 不 import `host`，不持有 token |

### 6.13.2 `github`：只读探测

- `githubAuthStatus()` / `githubRefresh()` / `ghSetupProblem()`；
- `gh auth status` 用 8 秒 `runBounded`；
- `THINKRAIL_GH_OFFLINE=1` 强制未连接；
- 缺失/未认证/超时都降级，不抛给 UI。

### 6.13.3 `branch-review`：尽力而为 + 可靠性标记

`findBranchReviewOutcome(cwd, branch, { fresh? }) → { value, reliable }`：

- 先查打开评审；没有才分页取已合并/已关闭（五行）；
- 打开 PR 评审事实搭同一次 `--json`；
- 只接受 head 仓库为 origin 自己的行；
- 检查按最新尝试计；
- 缓存 `(worktree, branch)` 60 秒、单飞；`forgetOpenBranchReview` 按分支代次失效。

`reliable: false` 表示远端没答或输出不可信——**调用方必须据此降级**，不能把“没查到”当“没有 PR”。

### 6.13.4 `pr`：推送与打开的真实守卫

`pushBranch`：

- `assertSafeRef(ws.branch)`；
- 前后 `refreshUserOwnedWorkspace` 重校验；
- `--set-upstream origin <branch>`；
- `PUSH_AUTH_FAILED` 是命名错误；
- 分支就是自己基点（`ws.branch === baseRef(ws.baseBranch)`）时需要 `confirmBase`。

`openPr`：

- 拒绝非 origin 基点；
- 拒绝 `ws.branch === baseRef(ws.baseBranch)`；
- 刷新 LIVE 分支；
- `gh --repo <push slug>`；
- `--head` + `--base`，**`--base` 总显式**；
- 变更型 `gh` 60 秒预算；
- `create` 失败后复查（可能已经创建成功但输出丢了）；
- compare 回退 + `ghProblem`。

`dirtyFiles` 永不阻塞：用户有未提交文件也可以开 PR（PR 只看已推送提交）。

### 6.13.5 删除远端分支：为什么不用 `gh pr merge --delete-branch`

- `gh pr view <number>` 校验 head 仓库；
- `--match-head-commit <HEAD>` 防止删错；
- 远端分支用 git 删除：
  `--force-with-lease=refs/heads/<branch>:<tip> origin --delete refs/heads/<branch>`；
- 删除前检查没有打开的 PR。

`--force-with-lease=<ref>:<tip>` 是“只有远端还在我以为的那个 tip 时才删”的 CAS；
比 `--delete-branch` 更可控，也避免 gh 替你决定删除时机。

---

## 6.14 其余领域：一张表带过，但每条都可追

| 模块 | 它拥有的最重要一条规则 |
| --- | --- |
| `extensions` | 静态 import 每个 `@thinkrail/ext-*/server`；注册表有序；父级摊平；`disabledExtensions` 由 `host` 回答 |
| `auth` | `startLogin` 同步返回 `{ loginId }`，登录脱离运行；新登录取代同 provider 未完成登录；Central 用封闭 `jbcentral` 状态 |
| `assist` | 一次性补全始终时间盒 + `maxTokens`；降级 `null` 绝不抛 |
| `history` | 自己异步枚举/读取 pi JSONL，绝不调 `SessionManager.listAll()`；只索引 assistant 消息；工具结果/思考不索引 |
| `templates` | 全局 + 项目两个提示模板目录；每次重读无缓存；不跟随符号链接；frontmatter 只读前 8 KiB |
| `locales` | 每文件 2 MB、共 64 个；每秒签名轮询而非 `fs.watch`；启动时创建文件夹一次 |
| `themes` | 每文件 256 KB、共 64 个；`MAX_THEME_NESTING` 32；病态嵌套在解析前就报 `invalid-json` |
| `fonts` | 每文件 50 MiB、共 128 个；自研 sfnt 读取；`/fonts/` GET+PUT；PUT 必须带上传头 |
| `feedback` | 10 条消息初始/重试间隔；`feedback.json` 记录邀请状态；survey 从 PostHog 拉取 |
| `resourceHistory` | `<dataDir>/resource-history/...` 的追加式 JSONL；Interrupted 是读取时推导，不存储；同步写 |
| `analytics` | 只上报**分类器**：type、code、source/scope/method、ThinkRail 栈帧；绝不 `message` 文本 |
| `log` | `logger(scope)` 唯一入口；`<dataDir>/logs/thinkrail.jsonl`；pino-roll 按日 + 10 MB、14 个轮转 |
| `shellEnv`（shared） | 登录 shell 环境解析；`resolveShellEnv()` 是 PATH 修复的唯一来源 |

### 6.14.1 分析隐私：允许列表是产品契约

`analytics` 的错误允许列表（[analytics/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/analytics/SPEC.zh.md)）：

- ≤5 个异常；
- type 标识符；
- 分类器 `value`；
- `mechanism`；
- 栈帧仅：platform / function / filename / lineno / colno / in_app / chunk_id。

**绝不发送**：上下文行、vars、abs_path、module、message 文本。

根异常的 `value` 由允许字段构建，例如 `request pr.open PUSH_AUTH_FAILED`。
`code` 只允许 `^[A-Z][A-Z0-9_]+$` 通过。

### 6.14.2 日志：持久文件只放安全内容

`logger(scope)` 写 Pino JSONL 到 `<dataDir>/logs/thinkrail.jsonl`；pino-pretty 只渲染 stderr。

持久消息里只允许插值：

- 封闭方法名；
- 计数；
- 生成的 id；
- 有界元数据。

**不要**把错误 message、聊天内容、文件内容、路径、凭据写进持久日志。

`observeLoggedErrors(observer | null)` 是唯一接缝：`host` 接到它，把 `log.warn/error(msg, err)`
转成 `host_logged` 分析事件。`log` 自己不 import `analytics`。

### 6.14.3 语言/主题/字体：为什么是轮询不是 `fs.watch`

三个模块共享同一种变更检测：**每秒比较文件夹签名**（名字、inode、大小、mtime、ctime）。

原因写在 locales 的 SPEC 里：Linux 上的 Bun 在整个进程内对每个路径共享一个 inotify watch；
文件夹或数据目录被删除后重建时，重新 watch 可能拿到一个死 watcher。每秒十几次 `stat` 没有这种失效模式。

ctime 能捕捉 `chmod` 修复或“重写但恢复自身 mtime”的保存；一个 tick 内的突发（写入 + 重命名）只算一次提示。

---

## 6.15 请求 → 模块地图

WS 方法按前缀归到模块。看这张表，你应该能预测“这个请求最终会碰哪些磁盘/进程资源”。

| 前缀 | 例子 | 主要模块 | 副作用 |
| --- | --- | --- | --- |
| `session.*` | `create` / `prompt` / `steer` / `followUp` / `abort` / `compact` / `rename` | `agent`、`settings` | 启动 pi 轮次、写转录、发事件 |
| `session.stateList` / `session.list` | — | `agent`、`todos` | **查询**（`session.list` 会读计划统计） |
| `session.resources` / `resourceHistory` | — | `agent`、`resourceHistory` | 读归档 |
| `workspace.*` | `create` / `remove` / `rename` / `push` / `diffStats` / `openReview` | `workspaces`、`git`、`pr`、`reviews`、`terminal` | 写 worktree、重启 watcher、关终端 |
| `project.*` | `open` / `clone` / `init` / `setTrust` / `skills` | `projects`、`git` | 注册表写入、克隆 |
| `git.*` | `status` / `diffFile` / `listCommits` / `prefetch` | `git` | 读仓库；prefetch 可能移动远端 ref |
| `fs.*` | `readDir` / `readFile` | `fs`、`watch` | 读工作树；`readDir` 会 `ensureWatch` |
| `change.*` | `revert` / `undo` | `changes`、`git`、`trash` | 写工作树 + 废纸篓 |
| `workspace.turns` | — | `turns` | 读 `turns.json`（推送 channel 是 `turn.changed`） |
| `review.*` | `commentAdd` / `sendBatch` / `sendComment` / `close` | `reviews`、`agent`、`git`、`fs` | 写评审 JSON；send 会启动会话 |
| `todo.*` | `add` / `update` / `review` / `startReview` / `reviewAll` | `todos`、`git` | 写计划 JSON；done 会提交 |
| `template.*` | `list` / `save` / `delete` | `templates` | 读写提示模板目录 |
| `terminal.*` | `attach` / `write` / `resize` / `close` | `terminal` | 生成/写 PTY |
| `mcp.*` | `add` / `approve` / `login` / `testConnection` | `agent`（MCP 部分）、`host` | 写 MCP 配置、探测 |
| `model.*` | `list` / `refresh` / `default` / `clampThinking` | `agent`、`settings` | 刷新目录、写偏好 |
| `provider.*` | `status` / `loginStart` / `jbcentral*` | `auth`、`agent` | 登录、Central 代理 |
| `pr.*` | `preview` / `open` / `merge` / `ready` | `pr`、`branch-review`、`github` | 网络 gh / git；可能开 PR |
| `settings.update` | — | `settings` | 写配置 + 广播 |
| `environment.*` | `get` / `update` | `environment` | 写 0600 文件；**需要重启** |
| `locales.*` / `themes.*` / `fonts.*` | `list` / `reveal` | 对应模块 + `editors` | 扫描目录、reveal |
| `history.search` | — | `history` | 读 pi JSONL；不写 |
| `spec.graph` | — | `spec` | 同步读规格 |
| `extension.list` / `skill.*` | — | `extensions`、`agent` | 读注册表/技能 |
| `feedback.*` | `respond` / `survey` / `survey.submit` | `feedback`、`analytics` | 写邀请状态；提交 survey |
| `client.reportError` | — | `analytics` | 上报客户端错误（仍走分类器） |
| `host.update` | — | `appliers` | 启动宿主更新 |

> 这不是完整的 190+ 方法清单，而是“看到前缀就能定位”的目录。完整清单在
> [handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts) 的 `handlers` 对象。

---

## 6.16 追踪练习

> 用法：先自己回答，再展开 `<details>`。答案引用的是真实代码位置，不是伪代码。

### 练习 1：`workspace.remove` 为什么先返回、后删目录？

<details><summary>参考答案</summary>

因为 `git worktree remove --force` 可能很慢（大仓库、文件锁、Windows 杀毒）。
`workspace.remove` 先 `forgetWorkspace` 丢弃注册记录，同步清掉所有指向它的内存结构
（spec index、reviews、changes、turns、watch、terminals），然后 `void archiveTeardown(ws)` 在后台做
`removeWorkspaceSessions` → `settleChangeArtifacts` → `reclaimWorktree`。
客户端立即看到工作区消失；磁盘回收不阻塞 UI。

见 [handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L717-L733)。
</details>

### 练习 2：为什么 `/blob` 可以 `immutable`，而 `/files` 只能 `no-store`？

<details><summary>参考答案</summary>

`/blob` 的 URL 含 oid；oid 是内容哈希，所以同一 URL 的字节永远不变，可以永久缓存。
`/files` 指向工作树当前文件，用户/agent 随时会改，缓存会让 UI 显示旧内容。
两者都做字节分类与安全头，但缓存策略由“内容是否不可变”决定。

见 [fileRoutes.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/fileRoutes.ts#L8-L30)。
</details>

### 练习 3：一个 `session.prompt` 请求，什么时候回 `ok: true`？

<details><summary>参考答案</summary>

`sendUserMessage` → `ackSend(operation)`：最多等 10 秒。
10 秒内 pi 的 prompt 启动失败 → 方法返回 `ok: false`；
10 秒后仍在跑 → 先回 `{ ok: true }`，后续失败通过 `pi.event` / `session.state` 表达。
这避免客户端等几分钟的轮次，同时不吞掉“启动失败”。

见 [ackSend.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/ackSend.ts) 与
[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L335-L353)。
</details>

### 练习 4：客户端断线 10 秒后重连，重发同一个请求 id，会发生什么？

<details><summary>参考答案</summary>

分三种情况：

1. 原 handler 已结算且结果仍在保留预算内 → 直接返回缓存结果，不重新执行；
2. 原 handler 已结算但结果超过字节预算 → `RequestReplayUnretainedError`，**不重新执行**；
3. 原 handler 仍未结算 → 返回同一个 promise，继续等它。

如果 fingerprint 变了 → `RequestReplayConflictError`。
60 秒无人重连且全部已结算 → `clearClient` 回收；否则重新计时。

见 [requestReplayCache.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/requestReplayCache.ts#L104-L215)。
</details>

### 练习 5：为什么 `turns` 不 import `agent`，而 `reviews` 也不 import `agent`？

<details><summary>参考答案</summary>

两者都需要“agent 运行了什么”，但都不应该知道 agent 的对象模型：

- `turns` 只需要事件流里的 `agent_start` / `agent_settled` 与工作区 id；`host` 把事件喂给它；
- `reviews` 需要“发一条带评审上下文的 prompt”，但会话创建与发送由 `host` 组装；
  它只通过 `setReviewCommentHandler` 接缝处理 agent 侧的 `resolve_comment` 工具。

这样 `agent` 换代、增加父子会话、改事件形状时，这两个领域模块不需要跟着改。

见 [packages/server/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/SPEC.zh.md) 的依赖图。
</details>

### 练习 6：`baseBranch` 与 `diffBase` 各在什么时候被写？

<details><summary>参考答案</summary>

- `baseBranch`：`createWorkspace` 时记录“创建来源”，之后不再变（除非重命名相关操作）；
- `diffBase`：用户通过 `workspace.setDiffBase` 修改，决定评审/差异跟谁比。

读取时统一走 `diffBaseRef(ws) = diffBase ?? baseBranch`。
不要用 `diffBase` 做推送基点保护，也不要用 `baseBranch` 决定评审范围。

见 [workspaces/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/workspaces/SPEC.zh.md)。
</details>

### 练习 7：为什么 `updateConfig` 要先持久化再广播？

<details><summary>参考答案</summary>

因为 `settings.changed` 携带的是**完整合并配置**。如果先广播后写盘，客户端会看到“新配置”，
但下一个请求读 `getConfig()` 可能读到旧值（或写盘失败后永久不一致）。
先写盘再广播，保证“收到广播 ⇒ 磁盘已一致”。

见 [settings/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/settings/SPEC.zh.md)。
</details>

### 练习 8：一个 64 MiB 的 PDF 在 `/blob` 上会发生什么？

<details><summary>参考答案</summary>

不会进内存读完整份：

1. `readBlobSizeAtAsync` 先取大小；
2. 超过 `BLOB_SIZE_LIMIT`（64 MiB）→ 413；
3. 未超 → `readBlobStreamAtAsync` 只 await 8 KiB 探测头；
4. 头用于 `classifyBytes` → `application/pdf`；
5. 其余流式转发，5 分钟转发时限。

见 [fileRoutes.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/fileRoutes.ts#L71-L97) 与
[git/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/git/SPEC.zh.md)。
</details>

### 练习 9：分析上报里能看到用户的消息吗？

<details><summary>参考答案</summary>

不能。错误上报只允许分类器：异常 type、`code`、source/scope/request method、ThinkRail 栈帧
（platform/function/filename/lineno/colno/in_app/chunk_id）。
根异常的 `value` 由允许字段构建，例如 `request pr.open PUSH_AUTH_FAILED`。
`message` 文本、聊天内容、文件内容、路径、凭据都不在允许列表里。

见 [analytics/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/analytics/SPEC.zh.md)。
</details>

### 练习 10：为什么 `subprocess` 不能等 stdout EOF？

<details><summary>参考答案</summary>

因为子进程可能 fork 出后台孙进程并继承 stdout。子进程退出后，孙进程仍持有管道，EOF 永远不来。
`runBounded` 以**子进程退出**为完成信号；超时则按进程组杀（POSIX）或 `taskkill /T /F`（Windows），
并保留 250 ms 排空宽限以拿到诊断输出。

见 [subprocess/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/subprocess/SPEC.zh.md)。
</details>

---

## 6.17 必须做对：一张“反模式 → 正确做法”表

| 反模式 | 为什么错 | 正确做法 |
| --- | --- | --- |
| 特性模块 import `host` 发推送 | 形成环，领域代码知道传输 | 暴露 `setXxxPublisher`，由 `host` 安装 |
| 在 `handlers` 里写领域逻辑 | 组合根膨胀，无法单测 | 逻辑进模块，handler 只组合 |
| 在 `persistence` 推导会话状态 | 第二套状态机 | 只存/取，推导留给拥有者 |
| 把 `Model` 原样发给客户端 | 泄漏 pi 内部、随版本破 | `toWireModel` 允许列表 + `resolveWireModel` 回解析 |
| 用 `agent_end` 判断“结束” | 后面还可能有重试/压缩/续跑 | 用 `agent_settled` |
| 用扩展名判断文件类型 | 用户可改扩展名，字节不会骗人 | `classifyBytes` 优先 |
| 在请求路径用同步 `git()` | 阻塞事件循环 | 读取走 `gitAsync`，写入才用同步 |
| 用 `unlink` 删用户文件 | 不可恢复 | `trashFile`，失败就报错 |
| 广播 `environment.get` 的内容 | 可能含密钥 | 只允许显式获取，永不广播 |
| 错误上报带 `message` | 泄漏用户内容 | 只发分类器 + 允许字段 |
| 重放缓存主动驱逐 | 已发出的结果消失，客户端无法重建 | 只靠 ack/resume/clearClient |
| 在服务端加跨进程锁 | 不在产品模型内 | 接受最后写入者胜出 |
| 用 `fs.watch` 盯数据目录 | Linux 共享 inotify 的死 watcher | 每秒签名轮询 |
| `Bun.which` 不传 PATH | 读的是启动快照 | 显式 `{ PATH: process.env.PATH }` |
| 保存时偷偷归一化 | 调用方永远不知道错 | 保存原样写，校验在入口 |
| 把 close 当 delete | 用户失去可恢复性 | 评审 close = 归档；会话 delete 进废纸篓 |

---

## 6.18 一页速查

```
启动
  bootHost: resolveShellEnv(并行) → initLogging → installCrashLog → await shellEnv
            → applyHostEnvironment → initializeJbcentralRuntime → findFreePort
            → createServer → attachProcessSignals → "listening on"
  createServer: 预热 → 会话状态 → 内存结构 → 5min/1s 计时器 → Bun.serve
                → 发布者接缝 → watcher → 恢复终端/项目 → 观察者 → 返回接口

HTTP
  /ws        upgrade + clientKey + protocol
  /health    ok
  /files     no-store + 字节分类 + sniff
  /blob      immutable + oid + 64MiB + 8KiB 探测头
  /fonts     GET/PUT + 上传头 + 50MiB/128
  其他       静态 SPA + index 回退

WS
  frame: {ack} | {resume} | {id,method,params,sessionId}
  fingerprint = sha256([method, params, sessionId ?? null])
  重放: 512 live / 16MiB / 2048 tracked; 永不主动驱逐
  overflow: REQUEST_REPLAY_OVERFLOW; 超预算: RequestReplayUnretainedError
  ackSend: 10s 窗口, acked 后不再抛错
  handlers: { [M in WsMethodName]: WsHandler<M> }  // 漏一个就 typecheck 失败

模块要点
  agent       runtime 世代 / 会话保留世代 / toWireModel / agent_settled
  persistence dataDir 绝对 / 原子 rename / .corrupt-<ms> / 加载不写 / 保存不归一
  settings    updateConfig: 合并→校验→持久化→广播（完整配置 + applied）
  environment applyHostEnvironment 只调用一次；变更=重启；永不广播
  projects    路径解析 / NOT_GIT / ALREADY_OPEN / 克隆拒绝表
  workspaces  kind default|external / name≠branch / baseBranch≠diffBase
  git         同步写、异步读 / 55s 网络预算 / 5 种 scope / assertSafeRef
  fs          resolveWorktreeFile / classifyBytes / 扩展名不胜字节
  changes     await 加载 → 同步 CAS → 写入 / 回执 20 条 64MiB
  turns       agent_start→agent_settled / tree 快照 / 30 条 / 零变更不记
  watch       300ms/1s/100 路径 / 失效信号 / .git 触发 repoMeta / 预热上限 8
  terminal    (workspaceId,tabKey) / 接管 / 复活 / 背压闩锁
  subprocess  完成=退出 / 组杀 / 250ms 排空 / timeout 夹取
  reviews     baseSha 钉住 / 锚点 moved 粘性 / send 在 host / close=归档
  todos       窗口基线 / 4 道提交门禁 / reviews sidecar / 1 次自动周期
  pr          assertSafeRef / 基点守卫 / gh --repo / 60s / lease 删除
  analytics   只分类器 / 无 message / 允许列表
  log         logger(scope) / JSONL / 14 轮转 / observeLoggedErrors
```

---

### 与主指南的对应

| 主指南小节 | 本专题的展开 |
| --- | --- |
| 6.1 `host/` | 6.2、6.3、6.4 |
| 6.2 `agent/` | 6.5 |
| 6.3 持久化与设置 | 6.6、6.7 |
| 6.4 项目、工作区与 git | 6.8、6.9 |
| 6.5 `shared/` | 6.1.1 的进程边界、6.11 |
| 6.6 分析与日志 | 6.14.1、6.14.2 |
| 6.7 必须做对 | 6.17 |

> 建议的下一步：读完本章后，打开 [第 5 章专题](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH05-LIFECYCLE.zh.md)，
> 用本章的模块地图去重走一遍“从点击到渲染”的时间轴——空间结构与时间轴对上之后，服务端就不再是黑盒。
