# 第 5 章深度专题：一次聊天从点击到渲染的完整生命周期

> 本专题是 [LEARNING-GUIDE.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE.zh.md) 第 5 章的展开版。
> 主指南给的是骨架，这里给的是**代码级的因果链**：每一步由谁触发、状态写在哪里、失败如何表现、
> 哪些事件可以丢、哪些快照必须能重建。
>
> 阅读前提：已读完主指南第 3 章（三环架构）与第 4 章（仓库地图）。本专题会频繁跳转到
> `packages/contracts`、`packages/server`、`apps/web` 三处的具体行号，建议边读边打开。

---

## 5.0 先建立三个“坐标系”

跨层追踪最容易迷路的不是代码量，而是**同一个东西在不同层有不同的名字**。先把三组坐标钉死。

### 5.0.1 三层与三种“状态源”

| 层 | 权威状态 | 传输形态 | 你能在哪里看到 |
| --- | --- | --- | --- |
| pi（引擎） | 会话分支、消息、工具调用、流式增量 | 进程内事件 / 方法返回 | `AgentSession`、pi 事件联合 |
| 宿主（server） | 会话登记表、结算记录、投影后的 `SessionState` | WS 请求/响应 + `session.state` 推送 | [agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts) |
| Web（客户端） | `SessionRuntime.turns` 等渲染状态 | Zustand store | [appStore.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/appStore.ts#L471-L495) |

记住一句话：**宿主不复制 pi 的推理状态，只投影 pi 报出来的事实**。所以你在服务端看到的“状态”
是 `deriveSessionState(...)` 的产物，而不是另一套可变状态机。

### 5.0.2 四种 id

| id | 谁生成 | 作用域 | 用途 |
| --- | --- | --- | --- |
| `sessionId` | pi 的 `SessionManager` | 全局 | 会话唯一标识；UI 与宿主共用 |
| `clientKey` | 浏览器页面 | 单个 socket 连接 | 请求重放、终端归属、MCP 探测归属 |
| 请求 `id`（`trpi_<seq>`） | `WsTransport` | 单个页面进程 | 请求/响应关联、ack/resume 对账 |
| `toolCallId` | pi | 单次工具调用 | 工具行定位、工具响应回填 |

没有“pi UUID”这种东西。仓库规则里写的“one id model”就是这张表：**UI 的 tab 和
`session.sessionId` 是同一个会话的两层视图**，不要凭空造第二个标识。

### 5.0.3 三条边界，先分清再读代码

1. **尝试边界 vs 结算边界**
   `agent_end` 只表示“这一轮尝试结束了”，后面可能跟重试、压缩、恢复、排队续跑；
   `agent_settled` 才是“自动工作真的结束了”。见 [piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L63-L109)。
2. **请求响应 vs 推送事件**
   请求有 id、有 ack、有超时；推送没有 id、默认不重放。两者走同一条 socket，但语义完全不同。
3. **增量 vs 快照**
   `pi.event` 是一次性增量，**不重放**；`server.welcome`、`session.stateList`、`session.list`、
   `session.getMessages` 是快照，**必须能重建状态**。

---

## 5.1 装配：宿主是怎么起来的

### 5.1.1 启动器只做三件事

`apps/cli` 与 `apps/desktop` 都是薄启动器：

1. 解析参数/环境（端口、host、静态目录、是否开浏览器）；
2. 调 `bootHost(...)`（[bootstrap.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/cli/src/bootstrap.ts#L64)）；
3. 打开 UI（CLI 打开浏览器；Desktop 用 Electrobun 同进程承载 web 产物）。

`bootHost` 的职责在 [boot.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/boot.ts#L62)：
初始化日志与崩溃日志、修环境、初始化 Central runtime、选端口、`createServer(...)`、装信号处理。
桌面端固定 `127.0.0.1` 且端口 `0`（系统分配），其余完全复用这条链路。

**关键点**：没有“桌面专用后端”。CLI 和 Desktop 共享同一个宿主实现，差异只在端口、静态资源与
窗口壳。

### 5.1.2 `createServer` 的装配顺序

打开 [server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L271)。
它做的第一件“有语义”的事不是起 HTTP，而是**把宿主需要的依赖注入进来**：

- 初始化 Central runtime、读配置、种下 pi trust；
- 设置 session → project 解析器（`setSessionProjectResolver`）；
- `initializeSessionStates(loadWorkspaces()...)`：把磁盘上的工作区先登记成会话状态容器，
  这样客户端一连上就能通过 `session.stateList` 看到历史会话；
- 建立 `sockets`、`reapTimers`、`RequestReplayCache`、放弃请求上报预算；
- 最后 `Bun.serve(...)`（[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L348)）。

注意 `initializeSessionStates` 在 `Bun.serve` **之前**。这不是实现细节，而是水合契约的一部分：
**先有权威快照，再允许客户端连接**。

### 5.1.3 HTTP 面与 WS 面

`Bun.serve` 的 `fetch` 里按路径分派：

| 路径 | 作用 |
| --- | --- |
| `/ws` | WebSocket upgrade，唯一的长连接入口 |
| `/files` | 读取 worktree 文件 |
| `/blob` | 读取附件/二进制 blob |
| 用户字体路由 | 本地字体资源 |
| `staticDir` | 打包后的 web 产物 |

`/ws` 的处理只做两件事：解析 `client` / `protocol`，然后 `srv.upgrade(req, { data })`
（[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L352-L360)）。

```ts
const clientKey = url.searchParams.get("client") ?? `anon-${randomUUID()}`;
const protocolVersion = clientProtocolVersion(url.searchParams.get("protocol"));
```

`clientProtocolVersion` 对非法值返回 `0`，所以“老客户端不带版本”会被当成版本 0，走最保守的能力门禁。

---

## 5.2 握手：welcome 是第一个原子快照

### 5.2.1 服务端 `open`

连接建立后（[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L379-L447)）：

1. **同 clientKey 的旧 socket 被替换并关闭**。一个页面同时只保留一条活连接，避免双写。
2. 清掉该 clientKey 的回收定时器（说明它“回来了”）。
3. 订阅通道。
4. 构造 `ServerWelcome` 并发送。
5. 按发送结果处理：`unavailable` 直接关闭；`backpressured` 打上背压闩锁；最后恢复终端与反馈邀请。

订阅不是无脑全订，而是**按协议版本门禁**：

| 通道 | 门禁 |
| --- | --- |
| `mcpStatus` | `>= MCP_PROTOCOL_VERSION`（83） |
| `localesChanged` | `>= LOCALES_PROTOCOL_VERSION`（89） |
| `themesChanged` | `>= USER_THEMES_PROTOCOL_VERSION`（92） |
| `packagesChanged` | `>= PI_PACKAGES_PROTOCOL_VERSION`（100） |
| `fontsChanged` | `>= FONTS_PROTOCOL_VERSION`（102） |

其余通道（`piEvent`、`sessionState`、`sessionCreated`、`sessionDeleted`、工作区/项目/设置等）
无条件订阅。这体现“同一个宿主同时服务新旧 UI”的兼容策略：**新通道靠版本门禁，旧通道永远在**。

### 5.2.2 welcome 里到底有什么

```ts
const welcome: ServerWelcome = {
  protocolVersion: PROTOCOL_VERSION, // 宿主当前协议版本，102
  hostPlatform,
  projects: listProjects(),
  recentProjects: listRecentProjects(),
  config: getConfig(),
  ...(appVersion ? { appVersion } : {}),
  ...(hostUpdateNotice ? { hostUpdate: hostUpdateNotice } : {}),
};
```

注意 `protocolVersion` 是**宿主**的版本，不是客户端声明的版本。客户端声明的版本只用于宿主做
能力门禁；客户端拿到 welcome 后要再校验一次“我能不能跟这个宿主说话”。

### 5.2.3 客户端接收：一次 Zustand 写入

`wireTransport` 订阅 `serverWelcome`（[wireTransport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/wireTransport.ts#L265-L292)）：

1. 校验 `protocolVersion` 与 `projects` 形状；不合法就丢弃（避免半个 welcome 污染 store）。
2. `hideSurveyInvitation()`。
3. `installWelcomeSnapshot(...)`：协议、项目、最近项目、config、平台、更新信息、版本号
   **一次性写入**。
4. 刷新已加载工作区的列表、本地化、主题、字体。
5. 若协议 `>= SESSION_STATE_PROTOCOL_VERSION`（73），调 `hydrateSessionStates(connectionGeneration)`；
   否则 `installSessionStateSnapshot([])`。

为什么强调“一次写入”：如果拆成多次 `set`，React 会在中间渲染出一个“项目有了、协议还没到”的
帧，UI 可能据此禁用能力或显示错误空态。**快照安装必须是原子的**。

### 5.2.4 一个容易忽略的事实：welcome 会进 `latest`

`WsTransport` 的 `latest` 缓存只排除 [NON_REPLAYABLE_CHANNELS](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L21-L30)：

```ts
terminalData / terminalExit / terminalDetached
sessionDeleted / sessionResourcesChanged
mcpStatus / providerChanged / feedbackInterview
```

`serverWelcome` **不在**这个集合里，所以它会被记住，后订阅的处理器能立刻拿到最后一份 welcome。
重连时新的 welcome 覆盖旧的 `latest`。这保证了“晚挂载的 UI 模块”也能看到握手快照。

### 5.2.5 welcome 之后为什么还要 `stateList`

welcome 携带**项目**，不携带**每个会话的实时状态**。会话状态是另一条通道：

- 推送：`session.state`（`SessionStateRecord`）；
- 快照：`session.stateList`（对所有工作区做 `listSessionStates`）。

`session.stateList` 的处理在 [handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L1158-L1159)：

```ts
"session.stateList": () =>
  listSessionStates(listAllWorkspaceRecords().map(sessionStateWorkspace)),
```

它覆盖**所有**工作区，而不只是当前打开的那个。这是“后台会话也能显示运行/完成状态”的基础。

---

## 5.3 打开项目与工作区（为什么它不影响会话时序）

项目/工作区激活走的是另一条数据通道（`project.updated`、`workspace.*`），但它和会话生命周期
有两个交点：

1. **会话只能挂在已存在的工作区上**。`session.create` 第一步就是 `getWorkspace(workspaceId)`，
   并确保 scratch 目录存在（[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L1056-L1059)）。
2. **工作区生命周期会取消正在准备的会话**。`captureWorkspaceLifecycle` / `workspaceAcceptsSessions`
   用“全局代次 + 工作区代次 + closing 集合”三件套判断（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L171-L190)）。

所以“另开一个工作区”不是纯 UI 动作：它可能让一个正在 `createAgentSession` 的请求在注册阶段
被判定为失效，并触发 `abandonPreparedEntry`。

UI 侧还有一条独立规则：**别的客户端打开项目不会抢走你的导航；后台关闭也不会移动当前视图**。
这条规则在 [store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md)，
它解释的是“为什么 store 要区分‘状态变化’和‘导航意图’”。

---

## 5.4 创建会话：一条链，五个阶段

### 5.4.1 总览

```
session.create
  └─ handlers.ts
      └─ createSession                    ← 生命周期令牌 + 运行时代次 + 设置
          └─ createParentSession          ← sessionId 已知，准备期纳入工作区代次
              └─ createParentSessionInternal
                  ├─ toolResponseWaiters / mcpHost / subagents / backgroundCommands
                  ├─ createAgentSession   ← pi 真正建会话
                  └─ registerSession
                      ├─ prepareSessionEntry  ← 订阅事件 + bindExtensions
                      ├─ 生命周期复查
                      ├─ sessions.set
                      ├─ 应用工具集 / MCP 状态 / 完成队列
                      ├─ session.created
                      └─ publishEntryState  ← 初始 SessionState
```

### 5.4.2 `createSession`：先拿令牌，再拿运行时

[createSession](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1365-L1393) 做四件事：

1. `captureWorkspaceLifecycle(workspaceId)`：记下当前全局/工作区代次；
2. `getPiRuntimeGeneration()`：拿到共享的模型运行时与“被排除的扩展路径”等；
3. `buildSessionSettings(cwd, trusted)`：把项目信任状态翻译成 pi 的设置；
4. 可选模型解析：`input.modelOptional` 为真时，解析失败也不阻断创建。

这里没有“新开一个进程/新起一个 event loop”的动作。`createAgentSession` 是**同进程**的库调用。

### 5.4.3 `createParentSessionInternal`：把“能力”作为扩展注入

[createParentSessionInternal](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1416-L1517) 的顺序值得逐行看：

```ts
const toolResponseWaiters = createToolResponseWaiters();
const mcpHost = createMcpSessionHost({ policy, onCommandSettled, askForToolCall, grantTool });
const subagents = subagentsFor(workspaceId, ...);
const commands = createBackgroundCommands({ sessionId, getContext, canDeliverCompletion });

const result = await createAgentSession({
  ...options,
  modelRuntime: generation.runtime,
  resourceLoader: await buildResourceLoader(
    cwd, settingsManager, skillAdmissionResolver,
    generation.excludedSessionExtensionPaths,
    [
      subagents.extension,
      createBackgroundCommandsExtension({ service: commands }),
      ...mcpHost.extensions,
      extraUsageModelSelectExtension(sessionId),
    ],
    toolResponseSessionServices(toolResponseWaiters),
  ),
});
```

关键设计：**子代理、后台命令、MCP、工具响应等待都不是宿主的旁路逻辑，而是以 pi 扩展的形式
挂进同一个 `AgentSession`**。宿主不拦截工具调用，也不自己跑第二轮循环。

`canDeliverCompletion` 把“能不能把完成消息投递出去”收束成三个条件：会话资源可用、没有可恢复的
工具响应等待、没有正在 reload。这避免“会话正在换资源时突然弹一条完成提示”。

### 5.4.4 `prepareSessionEntry`：先订阅，后绑定

[prepareSessionEntry](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1121-L1295)：

1. 建 `Entry`（会话句柄 + 工作区 + 代次 + 队列 + 结算 + 各种 pending 集合）。
2. `session.subscribe(...)`：事件订阅先挂上。
3. `watchMcpStatus(sessionId)`。
4. `await session.bindExtensions({ mode: "rpc", uiContext, onError })`。
5. 绑定失败 → `abandonPreparedEntry(entry)`：取消扩展 UI、清理 MCP 状态、放弃工具响应等待、
   退订事件、关闭资源、清空队列、让 pi 会话走关闭流程。

**顺序的含义**：订阅先于 `bindExtensions`，所以扩展在绑定期间产生的 pi 事件不会丢；但发布被
`sessions.get(sessionId) === entry` 门禁住，所以注册完成前的事件不会提前发给客户端。
这是“订阅早、发布晚”的双重保护。

### 5.4.5 `registerSession`：登记 + 公告 + 初始状态

[registerSession](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1317-L1363)：

1. 等 `prepareSessionEntry` 完成；
2. **复查**工作区代次与会话拆除状态，失效就放弃；
3. `registered = true`、`sessions.set(...)`；
4. 应用子代理/评审/注册表/扩展工具；
5. `restartMcpStatus`、刷新完成队列、后台核对子代理用量；
6. `announceCreation` 为真时发 `session.created`（`summaryOf`）；
7. `publishEntryState`——客户端拿到的第一份权威状态；
8. 如果项目信任状态与 pi 资源信任不一致，`requestSessionReload`。

注意第 6 与第 7 的顺序：**先 created，再 state**。客户端收到 `session.created` 时可以有壳，
收到 state 时才有“运行/空闲/未读”。

---

## 5.5 发送一条消息：客户端三种行为，服务端三种路径

### 5.5.1 客户端：`send` / `steer` / `followUp`

发送逻辑在 [ChatView.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/ChatView.tsx#L580-L609)：

```ts
const queued = behavior !== "send";
if (!queued && (text || attachments.length > 0)) {
  transcriptRef.current?.armImmediateTurn();
  useAppStore.getState().appendUserMessage(sessionId, text, attachments);
}
const method =
  behavior === "steer" ? "session.steer"
  : behavior === "followUp" ? "session.followUp"
  : "session.prompt";
```

两条规则必须记牢：

1. **只有完全空闲的 `send` 才乐观进入转录**（`appendUserMessage`）。
2. **`steer` / `followUp` 绝不乐观追加**。它们的文本先进入服务端队列，客户端通过 `queue_update`
   看到队列，真正落地为 turn 要等 pi 发 `message_start`。

失败处理也按“是否乐观”分流：

```ts
useAppStore.getState().appendErrorTurn(sessionId, errorText(err));
if (queued) restoreTextToDraft(text);
else {
  const streaming = useAppStore.getState().sessions[sessionId]?.isStreaming ?? false;
  transcriptRef.current?.cancelImmediateTurn(streaming);
}
```

- 排队类失败：把文本还回草稿，队列本来就没显示它；
- 乐观类失败：追加错误 turn，并撤销乐观 turn（如果会话还在流式，保留“正在运行”的视觉）。

`interrupt` 行为（[ChatView.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/ChatView.tsx#L636-L643)）是
“先 `session.abort`，成功后再以 `send` 发新消息”；失败则显示错误并恢复草稿。

### 5.5.2 请求在线上是什么样

`WsTransport.request`（[transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L110-L132)）：

1. `id = trpi_<seq>`；
2. frame 里带 `method` / `params` / 可选 `sessionId`；
3. 存进 `pending`（保留 frame 以便重连重发）；
4. 默认 60 秒超时；超时后删除 pending、在 socket 还活着时补一个 ack、reject `RequestTimeoutError`。

60 秒不是给“agent 跑完”用的，而是给**请求被宿主受理**用的。真正长跑的是事件流。

### 5.5.3 服务端：`sendUserMessage` 是统一入口

[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L335-L353)：

```ts
await ackSend(runObservation.send(sessionId, control ? "internal" : "user", operation));
```

- 控制消息（内部注入）不按“用户发送”统计；
- `ackSend` 保证“请求被接受”与“运行记录”一致；
- 返回 `{ ok: true }` 只是“受理成功”，不代表 agent 已经回复。

### 5.5.4 服务端三种路径的真正分支

#### 路径 A：`promptSession`

[promptSession](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2284-L2301)：

```ts
const entry = mustGetEntry(sessionId);
admitRun(entry);
if (entry.session.isStreaming) {
  await steerEntry(entry, text, images);
  return;
}
const release = admitPreflight(entry);
try { await entry.session.prompt(text, answeringDialogs(entry, images)); }
finally { release(); }
```

**这里必须纠正一个常见误读**：pi SDK 的原始语义确实是“流式期间 `prompt()` 抛错”，但
ThinkRail 的 `session.prompt` **包装器**在流式时会自动转成 `steerEntry`。所以：

- 如果客户端在流式期间错误地调用了 `session.prompt`，宿主不会把 pi 的抛错原样甩回去，
  而是按 steer 处理；
- 但 UI 仍然应该主动选对方法，因为**自动转 steer 只是兜底，不是语义承诺**：
  用户意图“打断当前方向”应显式用 steer，“在当前轮之后追加”应显式用 followUp。

`admitPreflight` 记录一次“前置阶段”，`agent_start` 时统一释放。后者（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1207-L1210)）：

```ts
if (event.type === "agent_start") {
  entry.lastSettlement = null;
  releasePreflights(entry);
}
```

`answeringDialogs` 的 `preflightResult` 让“这条消息是否处理掉了一个扩展对话框”反馈给宿主：
非 `handled` 就取消该会话的扩展 UI。

#### 路径 B：`steerEntry`

[steerEntry](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2271-L2282)：

```ts
const disposition = await queueSessionMessage(entry, "steering", text, images, () =>
  entry.session.steer(text, images),
);
if (disposition !== "queued") return;
cancelExtUiForSession(entry.session.sessionId);
if (entry.toolResponseWaiters.supersede()) publishEntryState(entry);
```

- 本地先记一条 `TrackedQueuedMessage`，发送失败会移除并重新同步队列；
- 一旦确认“已排队”，当前扩展对话框失效，可恢复的工具响应等待被 supersede；
- 立即推一份新的 `SessionState`，让其他客户端看到队列长度/交互状态变化。

#### 路径 C：`followUpEntry`

[followUpEntry](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2330-L2353)：

```ts
if (entry.session.isStreaming) {
  const disposition = await queueSessionMessage(entry, "followUp", text, images, () =>
    entry.session.followUp(text, images),
  );
  ...
  return;
}
const release = admitPreflight(entry);
try { await entry.session.prompt(text, ...); }
finally { release(); }
```

**`followUp` 在空闲时就是一次普通 prompt**。这个分支很重要：客户端可能因为“刚看到流式结束”
而选择 followUp，但服务端到达时已经空闲；此时排队没有意义，直接开启新一轮。

### 5.5.5 队列的“显示层”与“真实层”

宿主内部有两层队列：

- `entry.queuedMessages`：宿主跟踪的文本/图片（用于编辑、删除、恢复草稿）；
- pi 自己维护的 steering/followUp 队列（权威）。

`queue_update` 投影时不是原样转发，而是经过 `displayedLane(...)`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1196-L1199)），
把“已经以 `message_start` 落地但 pi 还没从队列里移除”的条目隐藏掉。否则 UI 会同时显示
“队列里的这条”和“已经发出的同一条”。

配套还有 `deliveredStuckEmptyLane`：如果 pi 发来的 user 消息是空的、而宿主的显示层仍认为有队列项，
就递增 `stuckEmptyDeliveries` 并重新同步，避免队列永久卡住。

### 5.5.6 中止

[abortSession](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2471-L2495)：

1. 流式中先 `markCancelledRun(entry)`（用于后续状态里的 `cancelledRun` 语义）；
2. `abortEntry`：
   - 需要恢复队列时先 `clearEntryQueue`；
   - `toolResponseWaiters.prepareAbort()`；
   - 等待“已被接受的答案”一个宽限窗口（`ACCEPTED_ANSWER_STOP_GRACE_MS`）；
   - 再 `mergeQueueContent` 一次（防止宽限期间又有新项）；
   - 最后 `await entry.session.abort()`。

宽限窗口的意义：用户刚点了“接受”，工具响应可能已经在途；立刻 abort 会把一个本该成功的
工具调用变成半状态。先给它一点时间落地，再停。

---

## 5.6 服务端事件管线：从 pi 事件到线上 `PiEvent`

### 5.6.1 订阅回调里发生的事

[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1179-L1253)
的回调顺序是有意为之：

| 阶段 | 事件 | 动作 |
| --- | --- | --- |
| 1 | `turn_end` | 持久化工具响应、冲刷后台命令/子代理完成队列 |
| 2 | `message_start`（user） | 处理“卡住的空投递”，重同步队列并推 `queue_update` |
| 3 | `queue_update` | 用 `displayedLane` 同步宿主跟踪队列 |
| 4 | `compaction_start/end` | 置/清 `piCompactionInProgress`，end 时冲刷完成队列、重考虑 reload |
| 5 | `agent_start` | 清 `lastSettlement`、释放 preflight |
| 6 | `agent_end` | **从最后一条 assistant 消息捕获 `terminal`** |
| 7 | 每个事件 | `projectSessionEvent(event, terminal)`，`queue_update` 再投影一次 |
| 8 | `agent_settled` | 写 `lastSettlement`、持久化结算、刷新挂起的工具、MCP 复核 |
| 9 | 受门禁的发布 | `publish(pi.event)` + `publishEntryState` |
| 10 | `agent_settled` 之后 | `terminal = null`，为下一轮准备 |

第 6 步是整条链的关键设计：`terminal` **不是在 settled 时才去读 pi 状态**，而是在
`agent_end` 时从“最后一条 assistant 消息”里抓 `stopReason` / `errorMessage`，然后挂到
`agent_settled` 上发出去。为什么？“结算”本身不带错误信息，错误属于那一次尝试。

### 5.6.2 `projectSessionEvent` 只做四类改写

[sessionEventProjection.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/sessionEventProjection.ts#L4-L41)：

1. `agent_settled` → `{ type: "agent_settled", terminal }`；
2. `entry_appended` → 只保留 `entryType`（客户端只需要知道“追加了什么类型”，不需要整条 entry）；
3. `compaction_end` → 白名单 `reason` / `result.tokensBefore` / `estimatedTokensAfter` /
   `aborted` / `willRetry` / `errorMessage`；
4. `tool_execution_end` / `tool_execution_update` → 顶层去掉 `structuredContent`。

其余事件**原样透传**（`return event as PiEvent`）。

```ts
function withoutStructured(result: unknown): unknown {
  if (typeof result !== "object" || result === null || !("structuredContent" in result))
    return result;
  const { structuredContent: _structuredContent, ...rest } = result;
  return rest;
}
```

注意它是**浅层**剥离：只处理结果对象自身的 `structuredContent`，不递归。这既控制了体积，
又避免把工具结果的其余结构改坏。

为什么需要投影：线上契约要稳定、可版本化、体积可控。把 pi 的内部事件原样透出去，等于把 UI
绑死在 pi 实现细节上；投影层是**唯一允许知道 pi 事件形状的地方**。

### 5.6.3 `setSessionPublisher`：发布前的旁路观察

[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L898-L918)：

```ts
setSessionPublisher((payload) => {
  runObservation.observe(payload.sessionId, payload.event);
  void turnTracker.observe(payload.sessionId, payload.event);
  if (payload.event.type === "tool_execution_start") {
    const workspaceId = getSessionWorkspaceId(payload.sessionId);
    if (workspaceId) taskObservation.toolStarted(workspaceId, payload.sessionId, payload.event);
  }
  server.publish(WS_CHANNELS.piEvent, JSON.stringify({ channel: ..., data: payload }));
  ...
});
```

也就是说，事件先经过运行观察、turn 追踪、任务统计，然后**广播**。这些观察者都不允许改变事件
本身——它们是旁路，不是过滤器。

### 5.6.4 `session.state` 是另一条独立通道

`publishEntryState`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L458-L474)）：

```ts
const state = stateFromEntry(entry);
const firstPrompt = entryFirstPrompt(entry);
const serialized = JSON.stringify({ state, ...firstPrompt });
if (entry.lastPublishedState === serialized) return;
entry.lastPublishedState = serialized;
publishState({ sessionId, workspaceId, projectId, state, ...firstPrompt });
```

三个要点：

1. **按序列化结果去重**：内容没变就不发。这让“每个 pi 事件后都调一次”变得便宜。
2. **`firstPrompt` 懒扫描**：只在需要时扫新增 entry，避免每次全量遍历分支。
3. **`scheduleEntryStatePublish` 用 `setImmediate` 合并**：同一 tick 内多次请求只发一次。

`stateFromEntry`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L401-L418)）
把输入收束给 `deriveSessionState`：

```ts
deriveSessionState({
  entries: sessionManager.getBranch(),
  isStreaming: session.isStreaming,
  pendingMessageCount: effectivePendingCount(entry),
  backgroundWorkActive: backgroundWorkActive(entry),
  lastSettlement: entry.lastSettlement,
  lifecycleCompletion: entry.lastSettlement === undefined ? persistedCompletion(sessionId) : undefined,
  liveInteraction: entry.toolResponseWaiters.currentInteraction(),
  liveRegistry: true,
  interactiveTools: enabledInteractiveTools(),
  pendingDialog: pendingExtUiDialog(sessionId),
  handledCompletionId: receipts()?.handledCompletionBySession[sessionId],
  cancelledRunId: lifecycle().cancelledRunBySession[sessionId],
});
```

这就是“宿主不自己发明状态”的具体写法：**所有输入都是 pi 或宿主持久化记录里已有的事实**，
`deriveSessionState` 只做纯函数投影。

### 5.6.5 `SessionState` 字段速查

定义在 [piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L82-L109)：

| 字段 | 含义 | 常见误读 |
| --- | --- | --- |
| `execution` | `running` / `idle`，来自 pi 是否在跑 | 以为 `idle` 就代表“完全没事” |
| `runId` | 当前运行标识 | 以为它等于 sessionId |
| `needsInput` | 需要用户回答的会话输入（含 dialog） | 只当成“弹窗” |
| `completion` | 最近一次完成记录 | 以为 `agent_end` 就会产生 |
| `completionUnread` | 完成是否未读 | 忽略 `canMarkCompletionUnread` 的前置条件 |
| `queuedCount` | 排队消息数 | 与 `queue` 里的文本重复计算 |
| `backgroundWork` | 后台命令/子代理等仍在工作 | 以为 `execution: idle` 就不算 working |

`isSessionWorking` 的实现很简单，但语义很重要：

```ts
return state.execution === "running" || state.backgroundWork === true;
```

所以“agent 停了，但后台命令还在跑”仍然是 working；UI 不能只看 `execution`。

`canMarkCompletionUnread`（[piProtocol.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/contracts/src/piProtocol.ts#L96-L108)）
决定某个完成能否被当成“未读”提示，避免把用户正在看的会话反复标红。

---

## 5.7 客户端传输层：ack、resume 与批量

### 5.7.1 `handleMessage` 的分流

[transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L173-L199)：

```ts
this.beforeDispatch?.(msg);
if ("channel" in msg) {
  if (!NON_REPLAYABLE_CHANNELS.has(msg.channel)) this.latest.set(msg.channel, msg.data);
  const set = this.subscribers.get(msg.channel);
  if (set) for (const handler of set) handler(msg.data);
  return;
}
this.queueAck(msg.id);
const entry = this.pending.get(msg.id);
if (!entry) return;
...
```

三个细节：

1. **`beforeDispatch` 先执行**。`wireTransport` 在这里做 `piEvents.flush()`：
   如果下一条消息不是 `pi.event`（比如请求响应或 `session.state`），先把已排队的 pi 事件冲刷掉。
   否则“响应”可能先于它引发的状态变化到达，UI 会短暂看到旧状态。
2. **非重放通道不进 `latest`**。这些通道要么是流（终端），要么是一次性墓碑（删除），
   要么是会在重连后由快照修复的状态。
3. **响应先 ack 再 resolve**。ack 是微任务批量发送的，避免每个响应都单独写一帧。

### 5.7.2 `ack` 与 `resume` 的协议

客户端在 `open` 时（[transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L88-L98)）：

```ts
this.sendFrame(JSON.stringify({ resume: [...this.pending.keys()] }));
for (const entry of this.pending.values()) this.sendFrame(entry.frame);
```

- 先告诉服务端“这些请求我还在等”；
- 再把原始 frame 重发一遍。

服务端 `message` 里先处理 `ack` / `resume`（[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L457-L466)），
普通请求则走 `requestReplays.run(clientKey, requestId, fingerprint, handler)`
（[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L480-L512)）。

- `fingerprint = sha256([method, params, sessionId ?? null])`；
- 同一个 `(clientKey, requestId)` 若已有结算结果，直接重放，不重复执行；
- fingerprint 不同则说明“同一个 id 被用于不同请求”，属于客户端/代理异常。

这解决的是**至少一次传输**的经典问题：socket 断了，请求可能已经执行；不能盲目重发。
`resume` 声明“我还在等”，`ack` 声明“我收到了”，两者结合才能安全回收重放命名空间。

### 5.7.3 `piEventBatcher`：32ms 与 128 条

[piEventBatcher.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/piEventBatcher.ts#L4-L63)：

```ts
const DEFAULT_DELAY_MS = 32;
const DEFAULT_MAX_BATCH_SIZE = 128;
```

- 入队后若已达 128 条，立即冲刷；
- 否则安排在 32ms 后冲刷；
- `flush()` 是同步投递，保留入队顺序，不丢事件；
- `shouldFlushPiEventsBefore(message)`：只要下一条不是 `pi.event`，就先冲刷。

为什么是 32ms 而不是 16ms：文本增量很密，16ms 会让 React 每帧都重算一次大转录；
32ms 约等于两帧，能在“跟手”和“吞吐”之间取平衡。128 条是防止极端突发把一个批次撑得过大。

### 5.7.4 一次性原子折叠

[handlePiEvents](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/appStore.ts#L4259-L4281)：

```ts
set((s) => {
  let state = s;
  let sessions = s.sessions;
  for (const { event, sessionId } of payloads) {
    if (event.type === "session_info_changed") { ... }
    const runtime = sessions[sessionId];
    if (!runtime) continue;
    if (sessions === s.sessions) sessions = { ...sessions };
    const next = reduceSessionEvent(runtime, event);
    sessions[sessionId] = {
      ...next,
      eventRevision: runtime.eventRevision + 1,
      statsRefreshTick: runtime.statsRefreshTick + (invalidatesSessionStats(event) ? 1 : 0),
    };
  }
  if (sessions !== s.sessions) state = { ...state, sessions };
  return state;
});
```

- **一个 `set` 写完整批**：React 只提交一次；
- 每个受影响会话的 `eventRevision + 1`：这是水合/对账的栅栏；
- `statsRefreshTick` 只在“会让统计失效”的事件上递增，避免每个 delta 都触发统计重取。

---

## 5.8 逐事件折叠表（`reduceSessionEvent`）

实现从 [appStore.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/appStore.ts#L660-L883)。
下表是这一章最值得背下来的部分。

| 事件 | 写入内容 | 边界含义 |
| --- | --- | --- |
| `agent_start` | 消费失败恢复、`isStreaming: true`、`attemptAssistantId: null` | 新一轮开始；上一轮的“失败恢复”标记可以清掉 |
| `queue_update` | 替换 `queue {steering, followUp, hasImages?}` | 队列是服务端投影后的显示层，不是本地推测 |
| `message_start`（assistant） | 新 `currentAssistantId`、清 turn streaming、`attemptAssistantId: null` | 一条新的 assistant 消息开始 |
| `message_start`（user） | 控制消息忽略；上一条同文本/skill 调用则替换；否则追加 | 与服务端乐观 turn 对齐，避免重复 |
| `message_update` | 从 `partial` / `done.message` / `error` 取**快照**，按 id 替换或追加 | **替换而非追加** |
| `message_end` | 先处理工具响应/后台命令/子代理/评审修复自定义消息；assistant 则结算当前 turn | 自定义消息不是普通 assistant 消息 |
| `tool_execution_start` | `toolResults[id] = { status: "running", raw: undefined }` | 工具行出现 |
| `tool_execution_update` | `raw = event.partialResult` | **替换** |
| `tool_execution_end` | `status = isError ? "error" : "done"`、`raw = event.result` | 工具行终态 |
| `agent_end` | **no-op** | 尝试级边界，不能结束 UI 运行态 |
| `agent_settled` | 失败则 error turn（`recovery: "try-again"`），否则 system closer；清 streaming/retry/compaction resuming；`settlementTick + 1` | 唯一的自动工作结束点 |
| `compaction_start` | 追加 `status: "running"` 的 compaction turn | 压缩是一等轮次 |
| `compaction_end` | `settleCompactionTurn`；overflow + willRetry 时移除被取代的 assistant | 压缩失败可能触发重试 |
| `auto_retry_start` | 移除被取代的 assistant，追加 retry turn | 失败的一轮不应留在转录里 |
| `auto_retry_end` | 清除 retry turn | 重试结束 |
| `summarization_retry_*` | 独立的 summarization 重试轮次 | 与普通 turn 重试分开 |
| `thinking_level_changed` | 更新 `thinkingLevel` | 设置变化也会走事件流 |

### 5.8.1 为什么 `message_update` 是替换

pi 的 `assistantMessageEvent` 携带的是**累积快照**（`partial`），不是增量片段。若前端把每个
`partial` 都追加成一个新 turn，文本会指数级重复。因此 reducer 用
`currentAssistantId` 作为“当前这条消息”的句柄，按 id `map` 替换。

`text_delta` / `thinking_delta` 的“追加”语义发生在 **pi 内部**；到了线上，它已经体现在
`partial.content` 的完整内容里。仓库规则里的“append vs replace”说的是两个不同层：
**pi 内部是 append，ThinkRail 渲染层对快照是 replace**。

### 5.8.2 为什么 `agent_end` 是 no-op

`agent_end` 之后可能发生：

- `auto_retry_start`：自动重试；
- `compaction_start`：压缩后续跑；
- `agent_settled`：才真正结束；
- 队列里的 followUp 续跑。

如果在这里清 `isStreaming`，UI 会在重试期间闪回“空闲”，用户可能在这个窗口发新消息，
造成语义冲突。

### 5.8.3 `settlementTick` 与 `eventRevision` 不是一回事

- `settlementTick`：**业务计数**，每次结算 +1。它驱动“完成后滚动/未读/音效”等 UI 副作用。
- `eventRevision`：**同步栅栏**，每个事件 +1。它用于 `reconcileSession` 判断
  “我拿到水合结果时，实时事件是否已经推进过运行时”。

两者都叫“计数”，但一个面向用户，一个面向并发正确性。

---

## 5.9 从 turn 到行：为什么需要 `deriveRows`

### 5.9.1 渲染链

```
ChatView (default export, line 89)
  └─ selector 读 SessionRuntime
      └─ deriveRows(turns, toolResults, isStreaming, isSpec, agentTurns, workspaceRoot)
          └─ ChatTranscript
              └─ ToolRendererBody
                  └─ getToolRenderer(toolName) ?? 默认渲染器
```

- `deriveRows` 在 [rows.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/rows.ts#L149-L156)；
- `ToolRendererBody` 在 [ToolRendererBody.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/ToolRendererBody.tsx#L7)；
- 注册表在 [toolRegistry.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/toolRegistry.tsx#L50)。

### 5.9.2 为什么不能直接渲染 turn

一个 assistant turn 的 `content` 可能同时包含：

- thinking 块；
- 文本块；
- 多个工具调用块；
- 工具结果。

pi 每完成一次工具轮次就发一条新的 assistant 消息，所以**一次“活动”会跨越多个 assistant 消息**。
直接按 turn 渲染会出现“文本—工具—文本—工具”被切成互不相关的卡片。`deriveRows` 把它们合并成
`activity` 行，并保留 `live` 标记给“正在流式”的那一段。

这也解释了主指南里的那句话：**按原始 turn 渲染无法正确分组，所以要先派生成行**。

### 5.9.3 展示层的小规则

- 用户消息超过 500 字符，且 agent 已开始回复后，折叠成 `line-clamp` + `Show more`；
- `ErrorTurn` 永不折叠；
- 只有结算派生的失败携带 `recovery: "try-again"`，普通请求错误没有这个恢复入口；
- compaction 行有 `data-testid="compaction-notice"` 与 `data-status`，这是 E2E 的锚点；
- markdown 按顶层块渲染，流式只重解析尾部块。

这些规则在 [chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md)。

---

## 5.10 结算、完成与未读

### 5.10.1 服务端：结算被持久化

`agent_settled` 到达时（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1234-L1247)）：

1. `entry.lastSettlement = terminal`；
2. `recordSettlement(entry, terminal)`：写 `completionBySession`（失败只记 warn，不让持久化失败
   拖垮会话）；
3. 刷新挂起的子代理/评审/扩展工具；
4. `reconsiderPendingReload`、`scheduleMcpStatusRefresh`、`reconcileMcpSessions`。

为什么持久化：宿主可能在你离开后重启，`session.stateList` 仍要能告诉你“上次是失败还是成功”。

### 5.10.2 客户端：结算生成“收口行”

`agent_settled` 的 reducer 分支（[appStore.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/appStore.ts#L822-L846)）：

```ts
const failure = assistantFailure(event.terminal);
const closer = failure
  ? { kind: "error", ... , recovery: "try-again" }
  : { kind: "system", ..., endedAt: Date.now() };
```

- 失败：生成一个带 `recovery: "try-again"` 的 error turn；
- 成功：生成一个 system 收口行（用于“已完成”视觉与时间戳）；
- 无论哪种，都清 streaming、清重试、清压缩 resuming，并插入 round closer。

### 5.10.3 completion 与 unread 的归属

- `SessionState.completion` 由服务端投影，客户端 `hostState` 只是镜像；
- `completionUnread` 的标记要满足 `canMarkCompletionUnread`；
- 用户查看后发 `session.acknowledgeCompletion`（[handlers.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/handlers.ts#L1160-L1162)），
  服务端写 `handledCompletionBySession` 收据。

这里有一个重要分工：**UI 的 `isStreaming` 是“我现在看到在流”，服务端的 `SessionState` 是“权威状态”**。
两者通常一致，但后台会话、其他客户端触发的工作、重连窗口里可能短暂不一致。UI 不应把
`isStreaming: false` 当成“服务端说完成了”。

---

## 5.11 重连、水合与墓碑

### 5.11.1 重连后哪些东西会回来

| 数据 | 重连后 |
| --- | --- |
| 未解析请求 | `resume` + 重发 frame，服务端可重放已结算结果 |
| 响应 | 通过 id 关联；ack 回收命名空间 |
| `server.welcome` | 重新发送，覆盖 `latest` |
| `pi.event` | **不重放** |
| `session.created` / `session.deleted` | **不重放** |
| `session.state` | 不重放；靠 `stateList` 快照 |
| 终端流 | 服务端恢复（`resumeClientTerminals`） |

### 5.11.2 `session.state` 的代次缓冲

`hydrateSessionStates(connectionGeneration)`（[wireTransport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/wireTransport.ts#L131-L171)）：

1. 建一个 `hydration { generation, installed: false, buffered: [] }`；
2. 请求 `session.stateList`；
3. 响应到达时校验：
   - `sessionStateHydration` 还是不是我这次；
   - store 的 `connectionGeneration` 还是不是我这次；
4. 合并 `records` 与缓冲：**同一 sessionId 以缓冲里的更新值为准**；
5. 一次 `installSessionStateSnapshot(ordered)`；
6. 失败则 500ms 起指数退避，最多 8s 重试。

在此期间的 `session.state` 推送（[wireTransport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/wireTransport.ts#L331-L340)）：

```ts
const hydration = sessionStateHydration;
if (hydration && !hydration.installed) {
  hydration.buffered.push(record);
  if (buffered.length > 4096) { /* 丢弃最旧或强制重取 */ }
  return;
}
applySessionStateRecord(record);
```

“缓冲上限 4096”是防内存爆炸的硬闸；它意味着**水合不能无限延迟**，否则会丢更新。

### 5.11.3 `session.list` 与 `session.getMessages` 修复什么

- `session.list`：工作台水合。它能发现“断开期间被删除的会话”，并修掉本地幽灵条目。
- `session.getMessages`：打开某个会话时拉完整转录；用于“错过 `pi.event` 后重建 turns”。

两者都是**权威快照**，与增量的关系是“快照优先，增量修正”。

### 5.11.4 墓碑

`deleteChat` / `noteClosedChats` / `closedChatsByWorkspace` 维护“这个会话已经删了”的本地事实。
`reconcileSession`（[appStore.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/appStore.ts#L4163-L4206)）
在应用水合结果前会检查：

```ts
!current ||
current.eventRevision !== expectedEventRevision ||
s.removedWorkspaceIds[summary.workspaceId] ||
isSessionDeleted(s, summary.workspaceId, summary.sessionId) ||
!selectWorkspaceSessionIds(...).includes(summary.sessionId)
```

这就是**栅栏**：如果实时事件已经推进过（`eventRevision` 变了），或者会话/工作区已被删除，
就丢弃这份水合结果。没有这个检查，一个迟到的快照会把刚删掉的会话“复活”。

---

## 5.12 错误路径：每条都有明确归属

### 5.12.1 请求失败

- 宿主 catch：取 `errorCodeOf(err)`，响应 `{ ok: false, error, errorCode? }`
  （[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L497-L507)）；
- 客户端：有 `errorCode` 就抛 `RequestError`，否则普通 `Error`
  （[transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L193-L198)）；
- 60 秒超时：`RequestTimeoutError`，并 ack 掉该 id。

注意错误上报规则：宿主只上报**分类器**（类型、code、source/scope/method、ThinkRail 栈帧），
从不上报 `message` 文本。所以 `errorCode` 的设计目的不是给人看，而是**让可区分的失败可被统计**。

### 5.12.2 发送失败

三种表现（见 5.5.1）：

| 行为 | UI 表现 |
| --- | --- |
| `send` 失败 | `appendErrorTurn` + 撤销乐观 turn |
| `steer` / `followUp` 失败 | `appendErrorTurn` + 文本回到草稿 |
| `interrupt` 的 abort 失败 | error turn + 恢复草稿，不发送新消息 |

`appendErrorTurn` **不会清 `isStreaming`**。因为请求被拒并不等于 agent 停了。

### 5.12.3 pi 内部失败

- 流式错误：`message_update` 的 `assistantMessageEvent.type === "error"`，快照替换；
- 尝试级失败：`agent_end` 捕获 `errorMessage` → `agent_settled.terminal` 携带；
- 客户端：结算分支生成 `recovery: "try-again"` 的 error turn；
- 自动重试：`auto_retry_start` 先删掉被取代的失败 assistant，避免转录里同时出现“失败版本”
  和“重试成功版本”；
- 溢出压缩：`compaction_end` 的 `reason === "overflow" && willRetry` 同样会移除被取代的 assistant。

### 5.12.4 扩展绑定失败

`bindExtensions` 抛出 → `abandonPreparedEntry`。此时会话从未被登记，客户端最多看到一个
`session.created` 之前的失败响应，不会看到半初始化的会话。

---

## 5.13 完整时序（带编号）

```
CLI/Desktop        Host(server)                 pi                 Web(transport/store)
    │                   │                        │                        │
 1  │ bootHost          │                        │                        │
    ├──────────────────▶│                        │                        │
 2  │                   │ Bun.serve /ws          │                        │
 3  │                   │◀──────────────────────────── WS ?client&protocol ─┤
 4  │                   │ 订阅通道 + welcome      │                        │
    │                   ├───────────────────────────────────────────────────▶│ 原子安装 welcome
 5  │                   │◀──────────────────────────── session.stateList ──┤
    │                   ├──── stateList(所有工作区) ───────────────────────▶│ 代次校验 + 安装
 6  │                   │◀──────────────────────────── session.create ──────┤
 7  │                   │ createSession          │                        │
    │                   ├───────────────────────▶│ createAgentSession     │
 8  │                   │ prepareSessionEntry    │                        │
    │                   │  subscribe + bindExt   │                        │
 9  │                   │ registerSession        │                        │
    │                   ├──── session.created ────────────────────────────▶│
    │                   ├──── session.state ──────────────────────────────▶│
10  │                   │◀──────────────────────────── session.prompt ──────┤ 乐观 user turn
11  │                   │ promptSession          │                        │
    │                   ├───────────────────────▶│ prompt()               │
12  │                   │◀──── agent_start ──────│                        │
    │                   ├──── pi.event(agent_start) ──────────────────────▶│ isStreaming=true
13  │                   │◀──── message_start ────│                        │
    │                   ├──── pi.event(message_start) ────────────────────▶│ 新 assistant turn
14  │                   │◀──── message_update ───│ (累积 partial)          │
    │                   ├──── pi.event(message_update) ───────────────────▶│ 快照替换
15  │                   │◀──── tool_execution_* ─│                        │
    │                   ├──── pi.event(tool_*) ───────────────────────────▶│ 工具行 running/done
16  │                   │◀──── agent_end ────────│ 捕获 terminal          │
    │                   │  (no-op 在客户端)      │                        │
17  │                   │◀──── agent_settled ────│                        │
    │                   ├──── pi.event(settled{terminal}) ────────────────▶│ 收口 + completion
18  │                   │  publishEntryState ────│                        │
    │                   ├──── session.state ──────────────────────────────▶│ hostState 镜像
19  │                   │                        │                        │ 用户查看
    │                   │◀──────────────────────── session.acknowledgeCompletion ┤
```

注意 16 与 17 之间可能有 `auto_retry_start`、`compaction_start`、`queue_update`；图中省略了这些
“尝试之间”的事件，但它们在 UI 上是可见的。

---

## 5.14 追踪练习（含参考答案）

### 练习 1：找出“客户端选方法”的那一行

在 [ChatView.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/ChatView.tsx#L593-L598)
里，`method` 的三元表达式就是。它决定了 `session.steer` / `session.followUp` / `session.prompt`。

### 练习 2：为什么 `steer` 不乐观追加，而 `send` 可以？

因为 `steer` 的文本是否真的成为一条 user turn，取决于 pi 是否接受并注入；服务端还会做
`displayedLane` 过滤。乐观追加会让 UI 同时显示“队列里的”和“已落地的”同一条。`send` 在空闲时
必定开启一轮，所以可以先显示。

### 练习 3：`agent_end` 之后客户端做了什么？

**什么也没做**（reducer 返回原 runtime）。真正的结束在 `agent_settled`。

### 练习 4：`terminal` 从哪里来？

[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L1211-L1223)
在 `agent_end` 里从最后一条 assistant 消息取 `stopReason` / `errorMessage`，存进闭包变量
`terminal`，然后在 `agent_settled` 时交给 `projectSessionEvent`，最后重置为 `null`。

### 练习 5：`tool_execution_update` 为什么是替换？

pi 发的是累积的 `partialResult`；替换才能反映“当前完整输出”。同理，服务端投影只剥掉顶层
`structuredContent`，其余结构保留。

### 练习 6：重连后如何知道某个会话已经完成？

`session.stateList` 返回的 `SessionStateRecord.state.completion` / `completionUnread`。`pi.event`
不重放，所以不能靠事件补。

### 练习 7：为什么 `handlePiEvents` 要在一个 `set` 里跑完整批？

避免 React 在批次中间渲染；同时保证 `eventRevision` 的递增与 turns 的变化是同一次提交。

### 练习 8：`session.prompt` 在流式期间到底会不会失败？

- **pi 原始 API**：会抛错，这是 pi 的并发约束；
- **ThinkRail 的 `session.prompt` 方法**：不会，因为 `promptSession` 检测到 `isStreaming` 后
  自动转 `steerEntry`（[agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L2291-L2294)）。

UI 仍应显式选择 steer/followUp，因为自动转 steer 只是兜底。

---

## 5.15 常见陷阱清单

1. **把 `agent_end` 当结束**：会漏掉重试、压缩、续跑，导致 UI 提前变空闲。
2. **把 `message_update` 当 delta 追加**：文本会重复；正确做法是按 `currentAssistantId` 替换快照。
3. **认为 `session.prompt` 流式中一定抛错**：宿主包装器会自动转 steer；但不要依赖它来表达意图。
4. **给 `steer` / `followUp` 做乐观 turn**：会与 `queue_update` 和 `message_start` 双重显示。
5. **以为 `pi.event` 会重放**：不会；重连必须靠 `stateList` / `list` / `getMessages`。
6. **用错误 `message` 做分类**：错误上报只允许分类器；需要区分就加 `CodedError` code。
7. **把 `execution: "idle"` 当成“没工作”**：`backgroundWork` 仍可能为真；用 `isSessionWorking`。
8. **以为 `session.state` 只是 UI 镜像**：它是权威投影，能跨客户端、跨重连、跨重启存在。
9. **忽略 `eventRevision` 栅栏**：迟到的水合结果会复活已删除会话或覆盖更新的实时状态。
10. **在 `registerSession` 之前发布事件**：发布有 `sessions.get(...) === entry` 门禁；绕过它会把
    半初始化会话暴露给客户端。

---

## 5.16 一页速查表

### 5.16.1 请求

| 方法 | 作用 | 关键分支 |
| --- | --- | --- |
| `session.create` | 建会话 | 工作区生命周期令牌 + pi runtime generation |
| `session.prompt` | 空闲启动 / 流式自动 steer | `isStreaming` 判断 |
| `session.steer` | 改变当前方向 | `queueSessionMessage("steering")` |
| `session.followUp` | 流式排队 / 空闲直接 prompt | `isStreaming` 判断 |
| `session.abort` | 中止 | 取消标记 + 答案宽限 + 清队列 |
| `session.stateList` | 全工作区状态快照 | 水合 |
| `session.list` | 工作区会话列表 | 工作台水合 |
| `session.getMessages` | 完整转录 | 重建 turns |
| `session.acknowledgeCompletion` | 清未读 | 写收据 |
| `session.respondToTool` | 回填工具响应 | 工具等待器 |

### 5.16.2 推送

| 通道 | 是否重放 | 用途 |
| --- | --- | --- |
| `server.welcome` | 是（`latest`） | 握手快照 |
| `pi.event` | 否 | 增量事件 |
| `session.state` | 否 | 状态投影 |
| `session.created` / `session.deleted` | 否 | 会话生命周期 |
| `session.resourcesChanged` | 否 | 资源变更 |
| `mcpStatus` / `providerChanged` | 否 | 状态重取触发 |
| `terminalData/Exit/Detached` | 否 | 终端流 |

### 5.16.3 事件

| 事件 | 前端动作 |
| --- | --- |
| `agent_start` | 进入流式 |
| `message_start` | 建/替换 turn |
| `message_update` | 快照替换 |
| `message_end` | 收口当前 turn（或自定义消息） |
| `tool_execution_start/update/end` | 工具行 running/替换/终态 |
| `queue_update` | 替换队列 |
| `compaction_start/end` | 压缩轮次 |
| `auto_retry_start/end` | 重试轮次 |
| `agent_end` | no-op |
| `agent_settled` | 结束 + 收口 + completion |

---

## 5.17 读完这一章你应该能回答

1. 为什么 `server.welcome` 要在一次 Zustand 写入里安装？
2. `prepareSessionEntry` 为什么先订阅、后绑定扩展？
3. `registerSession` 的“登记前复查”防的是什么？
4. 客户端三种发送行为分别怎样影响乐观 UI？
5. 服务端为什么要把 `terminal` 从 `agent_end` 搬到 `agent_settled`？
6. `projectSessionEvent` 为什么只剥顶层 `structuredContent`？
7. 32ms / 128 条这两个数字各自解决什么问题？
8. `settlementTick` 与 `eventRevision` 的区别？
9. 重连后哪四类数据分别由谁修复？
10. `agent_end` 与 `agent_settled` 的边界如果搞反，会制造哪两类 bug？

如果这 10 题都能用自己的话答出来，你已经掌握了这个项目最核心的一条链路。
