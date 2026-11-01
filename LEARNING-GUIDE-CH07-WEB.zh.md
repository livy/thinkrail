# 第 7 章深度专题：Web 客户端——从启动到渲染的空间结构

> 本专题是 [LEARNING-GUIDE.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE.zh.md) 第 7 章的展开版。
> 主指南讲的是“Web 客户端有哪些层”，这里讲的是**每一层为什么存在、状态写在哪里、旧代次如何被挡住**。
>
> 阅读前提：已读主指南第 3 章（三环架构）、第 5 章（生命周期）与第 6 章（服务端）。
> 本章与第 5 章专题互补：第 5 章从点击追踪到渲染，本章从**进程/内存结构**解释同一条链路为什么不会乱。

---

## 7.0 先建立四个“Web 坐标系”

### 7.0.1 Web 客户端是什么、不是什么

| 是 | 不是 |
| --- | --- |
| 一个**静态 bundle**，拨号到任意同源/远端宿主 | 一个自带后端的应用（它不连数据库、不跑 pi） |
| 宿主的**唯一 UI 投影**：把 `pi.event` 渲染成 chat 为中心的 IDE | 第二个 agent 运行时 |
| **领域状态的消费者**：项目/工作区/会话/终端都由宿主报告 | 领域状态的权威（除了浏览器本地偏好） |
| **组合根** `Shell` + 单一 Zustand store + 单一 transport | 一堆组件各自订阅 socket、各自维护缓存 |
| 可独立发布的产物：老 UI 可以连新宿主，新 UI 也可以连老宿主 | 与宿主版本绑死的同构应用 |

一句话：**宿主拥有事实，客户端拥有呈现；客户端唯一“拥有”的领域状态是它自己的本地布局与偏好。**

### 7.0.2 五个“代次/修订版”，先分清

这是 Web 端最容易读错的一组数字。它们都不是“业务数据”，而是**围栏**：

| 名字 | 谁推进 | 作用 |
| --- | --- | --- |
| `connectionGeneration` | `setStatus("connected")` | 每次重连推进；所有异步水合必须校验它 |
| `welcomeGeneration` | 完整 `server.welcome` 原子落地 | 冷启动导航/能力读取的就绪边；**连接状态不是就绪信号** |
| `providerVersion` | `provider.changed` | 模型目录/登录状态的失效代数；旧应答不能复活旧目录 |
| `eventRevision` | 每个 Pi 事件 | 权威转录读取的比较并装入围栏；被 UI 忽略的事件也推进 |
| `layoutProjectionEpoch` | 本地布局投影失效时 | 只用于让不受控的指针/尺寸草稿失效 |
| `settlementTick` | 每次 `agent_settled` | 批次里 start+settle 仍暴露完成边 |
| `statsRefreshTick` | 用量/上下文边界事件 | 让已挂载 chat 重新读权威统计 |

规则：**旧代次的迟到应答必须被丢弃，而不是“尽量合并”。** 合并会让重启后的旧快照覆盖新状态。

### 7.0.3 三层结构

```
静态启动层（main.tsx）
  ├── 主题/字体/偏好先于 React
  ├── transport 单例 + 客户端错误上报
  ├── navigation 驱动器
  └── 扩展注册
        │
领域投影层（store）
  ├── 项目/工作区/会话状态（宿主事实）
  ├── 每个活动会话的 SessionRuntime
  ├── 本地工作台帧/视图/注意
  └── 资源快照/MCP/评审/轮次信号
        │
呈现层（shell / panels / chat / resource-renderers）
  ├── Shell 组装布局
  ├── 面板只填容器，不知道摆放
  └── chat 只把 runtime 投影成行
```

### 7.0.4 术语表

| 术语 | 含义 | 易错点 |
| --- | --- | --- |
| 规范轮次（canonical turn） | pi 的消息序列 | 不是渲染行；行是派生的 |
| 行（row） | `deriveRows` 的输出 | 折叠可跨 assistant 消息边界 |
| placement（放置） | 某个资源在本地工作台里的位置 | 不是宿主状态，不能发给宿主 |
| attention（注意） | 每个工作区最后聚焦的组/标签 | 与“未读/需要输入”不是一回事 |
| tombstone（墓碑） | 页面寿命内拒绝过期到达的标记 | 防止已删除 chat 被在途读取重建 |
| 代次围栏（generation fence） | 用单调数字拒绝旧应答 | 不是节流，也不是缓存 |
| 失效信号（invalidation） | 只表示“重新读”，不带数据 | `locales.changed` 等就是这种 |

---

## 7.1 启动：`main.tsx` 的 12 步

### 7.1.1 逐行顺序

[main.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/main.tsx#L25-L38)：

```ts
initializeThemes();
applyThemePreference(readThemeHint());
initializeFonts();
initTransport();
installClientErrorReporting();
setErrorBoundaryReporter((error, reportId) =>
	reportClientError(clientErrorReport("render", error, reportId)),
);
initChatPreferencesPersistence();
initModelVisibilityPersistence();
initProjectExpansionPersistence();
initFilesTreePersistence();
initNavigation();
registerWebExtensions();
```

每一步为什么在这个位置：

| 步骤 | 为什么必须早/晚 |
| --- | --- |
| `initializeThemes()` | 主题目录是同步构建的；必须在 React 首帧前可用 |
| `applyThemePreference(readThemeHint())` | 带版本的首帧提示含系统配色，避免先闪默认主题 |
| `initializeFonts()` | 自托管字体要在首帧前注册，否则文字会换字重/换行 |
| `initTransport()` | store 的任何水合都需要 socket；它同时注册全部订阅 |
| `installClientErrorReporting()` | 要覆盖 transport 创建之后的错误，且早于任何 React 渲染 |
| `setErrorBoundaryReporter(...)` | 让 ErrorBoundary 的崩溃走同一条上报路径 |
| 四个 `init*Persistence()` | 本地偏好要在组件读取前水合 |
| `initNavigation()` | 恢复路由会驱动 store 激活；必须在水合之后 |
| `registerWebExtensions()` | 工具渲染器注册表要在读第一份会话记录之前完整 |

**关键**：这些副作用全部在 `createRoot(...).render(...)` 之前执行。不是“越早越好”的玄学，
而是因为后面的组件把“注册表已完整”“主题已应用”“transport 已连接”当成不变量。

### 7.1.2 `App` 渲染树

```tsx
<ErrorBoundary label={...} reportId="app">
  <TooltipProvider delayDuration={250} skipDelayDuration={400}>
    <AppResourceHost>
      <Shell />
    </AppResourceHost>
  </TooltipProvider>
</ErrorBoundary>
```

四层的职责：

- `ErrorBoundary`：最后兜底，渲染重载屏而不是空白根；
- `TooltipProvider`：全应用唯一实例；
- `AppResourceHost`：把内置资源渲染器注册进注册表，并提供模块级 `ResourceHost`；
- `Shell`：唯一组装根。

### 7.1.3 React Compiler：不是“可选项”，而是代码形状约束

`apps/web` 对所有 bundle 开启 React Compiler。它带来一组**必须遵守的写法**，否则函数会整体 bailout：

| 规则 | 原因 |
| --- | --- |
| 最新值 ref 在 `useInsertionEffect` 写，绝不在 render 写 | 插入副作用早于布局副作用；子组件的布局回调要看到当前值 |
| render 从不读 `ref.current` | 编译器无法证明它稳定 |
| 传给 prop 的 ref 命名 `*Ref` | 让处理器明确它是可变的 |
| 默认参数不读成员表达式 | `caret = text.length` 在参数位置读的是 render 值 |
| 闭包不 `++/--` 捕获变量 | 改写为 `attempts += 1` |
| 组件/hook 不在 `try` 内用 `try/finally`/`throw` | 移进本地 async 闭包并 `.catch().finally()` |
| `useMemo/useCallback` 列全依赖 | 否则编译器无法保留 |
| 回调不读的 `useMemo` 依赖要删掉 | 否则 memo 会因无关 key 重置 |
| render 不读编译器看不出变化的值（`matchMedia`/storage/`Date.now`） | 用 `useSyncExternalStore` 或状态 |

已知 bailout 有清单（`useVirtualRows` 是热路径）。**新的 bailout 视为回归**，除非加入清单。
验证方式：`bun run check:compiler` 对比 `scripts/compiler-census.baseline.txt`。

### 7.1.4 加载词汇：只有两层

整份 Web UI 的异步间隙只允许两种形态（[components/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/components/SPEC.zh.md)）：

1. **`SkeletonRows` / `LoadingRegion`** —— 用于**内容区域**；尺寸贴合将要到达的内容，300ms 宽限后淡入。
2. **`Loader2` spinner + 短标签** —— 用于**在途动作/瞬态状态**；钉在按钮、菜单项或图标上。

禁止：裸 `Loading…` 文字、居中的大 spinner 替代内容骨架、在加载指示旁边同时显示 Retry。

---

## 7.2 transport：唯一 WS 客户端

### 7.2.1 `WsTransport` 的字段

[transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L51-L200)：

| 字段 | 作用 |
| --- | --- |
| `ws` | 当前 socket；旧 socket 的回调会自我作废 |
| `seq` | 生成 `trpi_<n>` 请求 id |
| `pending` | id → `{ frame, resolve, reject, timer }` |
| `subscribers` | channel → handler 集合 |
| `latest` | 可重放 channel 的最后值（供迟到订阅者） |
| `ackQueue` / `ackScheduled` | microtask 批量 ack |
| `backoff` | 500ms → 10s 指数退避 |

### 7.2.2 `request` 的真实语义

```ts
request(method, params, { sessionId?, timeoutMs = 60_000 })
```

- 生成 `id = trpi_<seq>`；
- frame 立刻发送；如果 socket 没开，`sendFrame` 静默不发送，但 `pending` 保留 frame；
- 60 秒超时：删除 pending、**如果 socket 还开着就 queueAck**（让宿主释放重放条目），然后 reject；
- 重连时 `onopen` 先发 `{ resume: [...pending.keys()] }`，再把每个 pending frame 原样重发。

这解释了为什么“同一个请求 id 不会执行两次”：宿主按 `(clientKey, requestId)` 去重；
客户端负责在同一 id 下重放。

### 7.2.3 连接与重连

```ts
ws.onopen = () => {
	backoff = 500;
	onStatus("connected");
	ackQueue = [];
	send({ resume: [...pending.keys()] });
	for (const entry of pending.values()) send(entry.frame);
};
ws.onclose = () => {
	onStatus("disconnected");
	setTimeout(connect, backoff);
	backoff = Math.min(backoff * 2, 10_000);
};
```

`?client=` 是页面身份（`randomId("client")`，纯 http 也可生成，不用 `crypto.randomUUID`）；
`?protocol=` 是当前 wire 版本。两者都在 `withClientId` 里追加。

### 7.2.4 ack 与 latest：两类完全不同的“读”

| 机制 | 作用 | 语义 |
| --- | --- | --- |
| `queueAck(id)` | 每次**响应**读取后批量确认 | 尽力而为、不重传；`resume` 负责一次性修复 |
| `subscribe(channel, cb)` | 通道订阅 | 可重放通道会**立即**把 `latest` 交给新订阅者 |

`NON_REPLAYABLE_CHANNELS`（[transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L21-L30)）：

| 通道 | 为什么不重放 |
| --- | --- |
| `terminal.data` / `terminal.exit` / `terminal.detached` | 仅追加的终端流；重放会重复输出 |
| `session.deleted` | 一次性生命周期事实；错过了靠 `session.list` 对账 |
| `session.resourcesChanged` | 限定范围的失效；迟到订阅者应重新读 |
| `mcp.status` | 按会话快照；错过由下一次 `mcp.list` 修复 |
| `provider.changed` | 失效信号；重连会自然刷新 |
| `feedback.interview` | 被寻址的一次性邀请；welcome 后会重新投递 |

其余通道（如 `project.updated`、`workspace.*`、`settings.changed`、`session.state`）会保留最后值；
新订阅者先拿到最后值，再等后续推送。

### 7.2.5 `piEventBatcher`：32ms / 128 条

[piEventBatcher.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/piEventBatcher.ts#L4-L63)：

```ts
const DEFAULT_DELAY_MS = 32;
const DEFAULT_MAX_BATCH_SIZE = 128;
```

规则：

- `enqueue` 推入队列；
- 达到 128 条立刻 `flush`；
- 否则最多等 32ms；
- `flush` 把整批交给 `handlePiEvents`（一次原子 store 写入）；
- `dispose` 丢弃队列并取消计时器。

**顺序**由队列保证；**不丢事件**由“先拷贝再清空”保证。

### 7.2.6 投递屏障：为什么批量不能越过其他消息

`wireTransport` 给 transport 注入 `beforeDispatch`：

```ts
beforeDispatch: (message) => {
	if (shouldFlushPiEventsBefore(message)) piEvents.flush();
}
```

而

```ts
export function shouldFlushPiEventsBefore(message) {
	return !("channel" in message) || message.channel !== WS_CHANNELS.piEvent;
}
```

含义：**任何非 `pi.event` 消息到达前，先冲刷已排队的 Pi 事件。**

原因：`session.state` 等推送会引用“刚刚发生的事件”；如果 Pi 批次还压在队列里，
store 会先看到状态、后看到转录，选择器就可能按旧转录渲染。

`onStatus` 变化也会 `piEvents.flush()`，保证断连/重连边之前旧事件已经落地。

### 7.2.7 `wireTransport` 的路由表

[wireTransport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/wireTransport.ts#L245-L442) 在 `initTransport` 里一次性订阅：

| 通道/来源 | 去向 |
| --- | --- |
| `server.welcome` | `installWelcomeSnapshot` + 刷新项目列表/语言/主题/字体 + 水合会话状态 |
| `host.updateAvailable` | `applyHostUpdate` |
| `project.updated` | `applyProjectUpdated` |
| `pi.event` | `piEvents.enqueue` |
| `pi.extensionUi` | `applyExtUi` |
| `session.created` | `noteClosedChats`（只进历史，不自动放置） |
| `session.deleted` | `deleteChat` 墓碑折叠 |
| `session.state` | 水合缓冲或 `applySessionStateRecord` |
| `provider.login` | `applyLoginFrame`（墓碑工作区则取消） |
| `mcp.status` | `applyMcpStatus`（代次守卫） |
| `provider.changed` | `noteProviderChanged` + 重读 `model.list` |
| `locales.changed` / `themes.changed` / `fonts.changed` | 重新读对应目录 |
| `packages.changed` | `bumpPackagesVersion` |
| `feedback.interview` | 仅 v97+ 显示邀请 |
| `workspace.created` | `addWorkspace` + 重新水合会话状态 |
| `workspace.updated` | `updateWorkspace`（整条替换） |
| `workspace.removed` | `applyWorkspaceRemoved` + 取消定向登录 |
| `review.changed` | `applyReviewChanged` |
| `workspace.fsChanged` | `noteFsChanged` |
| `turn.changed` | `applyTurnChanged` |
| `settings.changed` | `applyConfig` |

**所有订阅都在 init 时完成，不在组件 effect 里反复订阅。** 这让事件顺序与 store 提交边界可控。

### 7.2.8 welcome 的水合链

welcome 不是“设置一个对象”，而是一串有序动作：

```
hideSurveyInvitation()
installWelcomeSnapshot(protocol, projects, recents, config, hostPlatform, hostUpdate, appVersion)
refreshLoadedWorkspaceLists(connectionGeneration)   // includeDiffStats:false
refreshLocalLocales / refreshUserThemes / refreshUserFonts
hydrateSessionStates(connectionGeneration) 或 installSessionStateSnapshot([])
loadInAppSurvey()
```

原因：

- 项目列表先落地，后面的工作区/会话水合才有作用域；
- `includeDiffStats: false` 避免在每个项目上同步扇出昂贵的 diff 统计；
- 会话状态用**缓冲 + 一次性安装**，避免 UI 看到半个映射；
- 旧协议宿主直接安装空映射，而不是让它一直“加载中”。

### 7.2.9 错误归一化与请求错误码

| 工具 | 作用 |
| --- | --- |
| `errorText(err, fallback?)` | 把拒绝/超时/非 Error 变成一行可展示文字 |
| `RequestError` | 携带宿主 `errorCode` 的拒绝 |
| `wsErrorCode(err)` | 取出 code，用于分支 |

规则：**带 code 的响应以 `RequestError` 拒绝，其他一切以普通 `Error` 拒绝。**
“有 code”正是调用方区分“这个具体失败”与“读取失败了”的依据。

### 7.2.10 `mcp.ts`：请求队列不是第二个状态所有者

`requestMcpList(method, params)` 按工作区与连接代次串行化：

- `mcp.list` 与每个返回列表的变更都排队；
- 下一个请求派发前先 `installMcpList`；
- 只有队尾的普通读取可以共享；变更从不共享；
- 排队请求在派发前重新检查 MCP 可用性、工作区移除、连接代次；
- 过期的排队工作拒绝而不发送。

目的：**旧列表不能覆盖新配置，实时推送不能被旧应答截断。**

### 7.2.11 客户端错误上报

`installClientErrorReporting()` 监听 `error` / `unhandledrejection`；
`clientErrorReport(kind, error, boundary?)` 生成上报：

- 只发给支持 `CLIENT_ERROR_REPORT_PROTOCOL_VERSION`（v90）的宿主；
- welcome 之前的上报最多排 10 条；每页最多 50 条；
- 从栈里切掉 message（V8 栈头会重复 message，看起来像 frame）；
- 带 PostHog chunk id（≤20）。

浏览器**不擦洗**错误；宿主是唯一的清洗者。这与第 6 章的分析隐私契约对齐。

### 7.2.12 协议能力门禁：为什么导出这么多 `supports*`

`transport` 导出一组能力判断，调用点据此降级：

| 函数 | 门禁的能力 |
| --- | --- |
| `supportsPlanReview` | 计划评审 UI（老宿主没有 `todo.startReview/reviewAll`） |
| `supportsPlanSummaryGeneration` | 自动计划摘要（`todo.generateSummary`） |
| `supportsChangeMutations` | `change.*`（回滚/撤销） |
| `supportsRichAnchors` | `region`/`structural` 评审锚点 |
| `supportsTurnChanges` | `workspace.turns` / 轮次回执 |
| `supportsUserThemes` | v92 用户主题目录 |
| `supportsToolResponses` | v93 `session.respondToTool`（否则退回 `session.answerQuestion`） |
| `supportsHostUpdateRun` | v70 宿主更新动作 |

原则：**旧宿主上要隐藏能力，而不是让请求失败后才显示错误。**

### 7.2.13 `httpBase()` 与 `/files`、`/blob`

`httpBase()` 把 WS URL 的协议改成 http/https 并保留 origin。
资源面板用 `resourceBytesUrl(...)` 在这上面拼 `/files/<workspaceId>/<path>` 或 `/blob/<workspaceId>/<oid>/<path>`。

为什么必须在 transport 上推导：Vite dev server 会代理 `/ws`、`/files`、`/blob`；
如果客户端硬编码一个 URL，开发环境会拿到 SPA 的 `index.html`，图片/PDF“成功但空白”。

---

## 7.3 store：一个 Zustand，六类状态

[store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md)。
`appStore.ts` 约 4660 行，但可以按“谁拥有”拆成六类：

| 类别 | 例子 | 权威 |
| --- | --- | --- |
| 连接/能力 | `status`、`connectionGeneration`、`welcomeGeneration`、`protocolVersion` | 客户端连接 + welcome |
| 项目/工作区 | `projects`、`recentProjects`、`workspaces`、`activeWorkspaceId` | 宿主快照/推送 |
| 会话运行时 | `sessions: Record<sessionId, SessionRuntime>` | pi 事件 + 权威转录读取 |
| 规范化会话状态 | `sessionStateByWorkspace` | 宿主 `session.state*` |
| 本地工作台 | `workbenchFrame`、`workspaceViewsByWorkspace`、attention | 浏览器本地 |
| 资源/信号 | Chat Resources、MCP、reviews、turns、fs tick、specs | 宿主快照 + 失效 |

### 7.3.1 `installWelcomeSnapshot`：一次写入的原子性

`installWelcomeSnapshot(protocol, projects, recents, config, hostPlatform, hostUpdate, appVersion)`：

- 一次 Zustand 写入；
- 装入协议 + 两个排好序的项目视图 + 可选 config/平台/更新快照；
- 最后推进 `welcomeGeneration`；
- 路由校验与能力读取**绝不会观察到半个 welcome**。

`installProjectSnapshot` 是仅项目的原语，给聚焦调用方用。

### 7.3.2 连接状态的两条边

| 边 | 何时推进 | 语义 |
| --- | --- | --- |
| `setStatus("connected")` | 每次 socket 连接成功 | 重连水合推进 `connectionGeneration` |
| 完整 welcome 落地 | 收到 `server.welcome` | 推进 `welcomeGeneration`，冷启动导航就绪 |

每次连接状态转换都会清空 `protocolVersion`，因此 UI 能力在 socket 之间、以及 welcome 装入之前始终不可用。

### 7.3.3 工作区生命周期：替换，不是合并

`workspace.updated` 的 fold 是 `updateWorkspace(ws)`：

- 在 `workspaces[projectId]` 中按 id **替换**该记录；
- 只沿用列表算出的 `diffStats`（推送快照本身没有它）。

为什么必须替换：合并无法清掉**宿主丢掉的某个可选字段**——

- `diffBase` 重新指向创建基线；
- 最后一条 `skillOverrides` 被移除；
- `subagentsOverride` 回到继承。

如果合并，客户端会继续按宿主已不再持有的值贴标签、做读取键。

未知项目或 id 不在列表里 → **no-op**；下一次 `workspace.list` 对账。

`addWorkspace` 同样按 id upsert；项目尚未列出时也是 no-op，避免播种一个只有一行的局部列表。

### 7.3.4 `applyWorkspaceRemoved`：一次完整的移除反应

这不是“删一行”，而是一组原子动作：

```
removeWorkspace(projectId, id)         // 丢掉该行
clearWorkspaceState(...)               // 丢掉本地视图/注意/终端映射/chat runtime
丢掉 recentProjects 里的死 id
若它是本客户端活动工作区 → 激活最近选中的、项目仍打开的已加载工作区
若无剩余 → selectProject(projectId) 回到 Project Home
两条兜底都给同一个中性 toast
```

**后台移除绝不移动焦点。** 只有“正好是我当前所在的工作区”才触发兜底——
否则一个远端客户端删除工作区会让你的编辑器突然跳走。

### 7.3.5 会话运行时 `SessionRuntime` 的字段

`sessions: Record<sessionId, SessionRuntime>`：

| 字段 | 作用 |
| --- | --- |
| `turns` | pi 规范轮次 |
| `toolResults` / `toolResponses` | 工具结果 / 交互式工具的迟到响应（按 toolCallId） |
| `currentAssistantId` / `attemptAssistantId` | 当前/本次尝试的 assistant 消息 |
| `isStreaming` | 是否在流式 |
| `settlementTick` | 每次 `agent_settled` 单调递增 |
| `model` / `thinkingLevel` | 会话创建时的快照 |
| `eventRevision` | 每收到一个 Pi 事件递增 |
| `syncedConnectionGeneration` | 转录最后读取自哪个连接代次 |
| `hostState` | 宿主撰写的会话状态 |
| `stats` / `statsRefreshTick` | 权威统计与其失效信号 |
| `commands` | 斜杠命令 |
| `draft` | 输入框草稿 |
| `queue` | steer/followUp 的排队文本 |
| `pendingExtUi` / `extUiQueue` | 扩展 UI 请求与 FIFO |
| `toolResponses` | 工具响应 |

**多个 chat 可并发流式**，正因为状态是按 `sessionId` 分片的，而不是全局一份。

### 7.3.6 `reduceSessionEvent`：纯函数折叠

[appStore.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/appStore.ts#L660)：

```ts
export function reduceSessionEvent(rt: SessionRuntime, event: PiEvent): SessionRuntime
```

它不碰 transport、不碰 React、不碰选择器。输入一个 runtime 和一个事件，输出新 runtime。

它承担的语义：

| 事件 | 效果 |
| --- | --- |
| `text_delta` / `thinking_delta` | **追加**到当前 assistant 内容 |
| `tool_execution_update.partialResult` | **替换**该工具的部分结果 |
| `message_start/end` | 建立/收口消息，处理乐观回显去重 |
| `agent_start` | 标记运行开始，消费 `try-again` 标记 |
| `agent_end` | 尝试结束；**不清空** `isStreaming` |
| `agent_settled` | 真正结束；清空 `isStreaming`、推进 `settlementTick` |
| `compaction_start/end` | 一等轮次：running → done/cancelled/failed/resuming |
| `auto_retry_start` | 丢弃被重试取代的失败 assistant 轮次 |
| `queue_update` | 折叠 steer/followUp 队列 |
| `session_info_changed` | 走统一标题 fold |
| `entry_appended`（usage/compaction/branch_summary/context_edit） | 推进 `statsRefreshTick` |

### 7.3.7 `handlePiEvents`：一次原子批次

`handlePiEvents(payloads)` 在一个 store 写入里：

- 按顺序折叠整批；
- 为每个受影响 runtime **按事件各递增一次 `eventRevision`**；
- 即使 UI 忽略该事件也递增——被忽略仍意味着它越过了快照定序边界。

`handlePiEvent(event, sessionId)` 委托给同一路径，因此测试与非线上调用方不会漂移。

### 7.3.8 乐观发送：只有空闲 prompt 才进转录

| 动作 | 是否乐观追加 | 为什么 |
| --- | --- | --- |
| 空闲 `prompt` | 是：`appendUserMessage` | 输入后立即看到自己的消息 |
| `steer` / `followUp` | **否**：只进 `queue` | 文本要等 pi 规范 `message_start` 才落地，实时与水合才一致 |

流式发送如果乐观追加，末轮回显去重会在“追加与回显之间出现 assistant 内容”时失手，
表现为重复且错位的排队气泡。这个 bug 在真实环境里复现过，所以规则是硬的。

### 7.3.9 错误轮次：被拒绝的发送 vs 流式失败

| 场景 | 表现 |
| --- | --- |
| `session.prompt`/`steer`/`followUp`/`create` 被拒绝 | `appendErrorTurn(sessionId, text)` 追加 `error` 轮次 |
| 流式期间 `stopReason: "error"` | `agent_settled` 时用宿主最终态元数据收口为 `error` 轮次 |
| `stopReason: "length"` | 变成可操作的截断错误，不能显示 “✓ Done” |
| 扩展 `notify` 的 error 级别 | 同样变成 `error` 轮次 |

被拒绝的发送**不清空** `isStreaming` 或当前 assistant id：一次被拒绝的 steer 无法结算宿主仍在跑的工作。

只有“由结算派生的当前失败”携带 web 本地的 `recovery: "try-again"`；历史失败不会重新获得可操作性。

### 7.3.10 压缩是一等轮次

- `compaction_start` 追加 `compaction` 轮次（`running`）；
- `compaction_end` 就地结算：
  - 成功 → `done` + tokens before/after（过守卫，线上数据不可信）；
  - `aborted` → `cancelled`；
  - `errorMessage` → `failed` 并携带消息；
- 没有正在运行的轮次时（压缩中途重连）追加已结算轮次；
- `willRetry: true` 的成功结束标 `resuming`，结算会清除它；
- 成功且没有持久 `summary` 的实时轮次，是 chat 集成去读 pi 规范转录的信号——**reducer 自己绝不猜切分边界**。

### 7.3.11 Ext-UI 缓冲：为什么要 64 槽环

宿主在回答“安装运行时”的 RPC **之前**就发布了扩展在 `session_start` 做的事。
如果客户端没缓冲，会静默丢掉扩展启动时设置的每个状态、标题、widget 与 notify。

因此：

- 发射即忘的帧（`notify`/`setStatus`/`setWidget`/`setTitle`）在没有可落地运行时时进入 `extUiOrphans`（有界 64）；
- 运行时一安装（`openChatSession` 或 `hydrateSession`）就排空；
- **对话框**（`select`/`confirm`/`input`/`editor`）绝不缓冲：它是一次往返的一半，另一半可能已被另一个客户端回答或因重启丢失。

已知缺口：64 槽是全局列表，而宿主向每个客户端广播；一批会话同时附加时可能驱逐仍需要的帧。

### 7.3.12 Chat Resources：快照 + 失效修订版

Resources 视图把**宿主快照**与 Pi 对话运行时分开保存，按 chat 身份与连接权威做 key。

规则：

- 装入/失效/清空一个快照是一个原子的 store 动作；
- 单调失效修订版 + 连接代数栅栏住每次读取装入与失败；
- 墓碑与被清空的条目拒绝迟到应答；
- 连接状态变化把保留快照标记为过期；不支持的 welcome 清空它们；
- **不存在“凭空造一个空目录”**：失败读取保留看起来已过期的数据。

分组/活动计数由 `selectors.ts` 的 `selectChatResourceGroups` 统一推导，
组件不各自独立推导。

### 7.3.13 MCP 分片：`mcpByWorkspace` + `mcpRevision`

两个原子动作写入：

| 动作 | 规则 |
| --- | --- |
| `applyMcpStatus(snapshot)` | 只在已连接、MCP 可用协议、作用域存活时装入；按代次守卫 |
| `installMcpList(read, result)` | 由请求前捕获的 `selectMcpRead` token 栅栏 |

`deriveMcpServerRows(projection, now)` 是唯一聚合：配置条目 → 运行时状态 → 注意行优先级。
`starting` 超过一分钟读作 unknown（stalled）。

### 7.3.14 本地状态与领域状态的界线

| 本地状态 | 存储位置 | 是否上线 |
| --- | --- | --- |
| 工作台帧/视图/注意 | store（`layoutState` 持久化） | 否 |
| `chatMessageOrder` / `streamingResponseMovement` | 浏览器 localStorage / 原生适配器 | 否 |
| `hiddenModels` | 浏览器偏好 | 否 |
| 展开的项目、Files 树状态、settled shelf 展开 | 本地持久化模块 | 否 |
| 主题的**生效外观** | `themes` 模块 + DOM | 否（`AppConfig.theme` 只是不透明选择） |
| 项目/工作区/会话 | store 投影 | 是（宿主权威） |

原则：**浏览器本地状态与领域状态分开。** 布局、折叠、未读提示等不应进入 `AppConfig`，
也不应发给宿主。

### 7.3.15 选择器：派生规则的唯一家

仓库规则：**派生状态在 selector，不在组件。**

典型：

- `selectWorkspacePartition(state, projectId, now)` —— live/settled 的唯一来源；
- `selectActiveWorkspace` / `selectWorkspaceById`；
- `selectChatResourceGroups` / `selectChatResourceOldest`；
- `selectSkillsStale(state, workspaceId, sessionId)`；
- `selectDiffScope` / `selectDiffBaseRef` / `selectDiffTabTargetRef`；
- `selectPickerModels`（隐藏模型过滤的唯一实现）。

`now` 作为参数传入，因此调用方决定 tick，selector 保持纯。

---

## 7.4 chat：从规范轮次到渲染行

[chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md)。

### 7.4.1 分层

```
ChatView（应用集成：store + transport）
   │
   ├── 选择器：当前 runtime、工具结果、流式标志、turns、hostState
   ├── deriveRows(turns, toolResults, isStreaming, isSpec, agentTurns, workspaceRoot)
   │      └── 纯函数，把内容块走成行
   └── ChatTranscript
          ├── UserTurn / SystemTurn / RetryTurn
          ├── MarkdownTurn（BlockMarkdown 逐块增量）
          ├── CompactionNotice
          ├── ErrorTurn（永不折叠）
          └── ToolTurn → ToolRendererBody → getToolRenderer(toolName)
```

`ChatView` 在 [ChatView.tsx](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/ChatView.tsx#L89) 导出，
在约 796 行渲染 `ChatTranscript`。

**为什么渲染行而不是轮次**：折叠可以跨 assistant 消息边界（pi 每个工具轮次发一条 assistant 消息），
按轮次的条目模型无法分组。

### 7.4.2 行种类

| 行 | 来源 | 行为 |
| --- | --- | --- |
| `user` | 用户消息 | 超过 500 字符且 agent 已回复 → `line-clamp` + `Show more` |
| `system` | 本地提示 | 1:1 |
| `retry` | `auto_retry_*` / `summarization_retry_*` | 按来源区分指示器 |
| `markdown` | assistant 文本块 | 逐块渲染，流式只重解析尾部 |
| `compaction` | 压缩生命周期/持久摘要 | 运行/完成/失败/取消 |
| `activity` | 连续常规工作 | 外层折叠区，内部按思考分组 |
| `tool` | 主工具调用 | `ToolCard` 或 bare 渲染器 |
| `backgroundCommandCompletion` / `subagentCompletion` | 后台结果 | 默认折叠的一行 |
| `reviewFix` | 计划评审修复 | 紧凑卡片 |
| `divider` | 回合结束 | 耗时/工具数/文件徽标块 |

`ErrorTurn` **永不折叠**：失败的轮次不能看起来像什么都没发生。

### 7.4.3 `deriveRows` 与 `projectRows`

[rows.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/rows.ts#L149) 的 `deriveRows`
保持规范与时间顺序；[projectRows](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/rows.ts#L88)
在用户行处划分组：

```
1,2,3 | 4,5,6  →（最新优先）→  6,5,4 | 3,2,1
```

行 id 逐字保留，因此折叠、闪烁、工具状态、历史锚点不会因顺序投影而分叉。
投影**绝不深入单行内部**：Markdown 段落/代码、工具卡正文、Activity 展开层级保持语义顺序。

### 7.4.4 `foldState`：手动切换永远胜出

所有折叠表面共享 `foldState` 缓存，键是稳定的行/步骤 id：

- Activity 外层分组；
- 嵌套 Thinking 分组；
- 工具行/`ToolCard`；
- 分隔线的多产物徽标块（用 `useSelection` 保存“选中的键”，而不是每侧布尔）。

行 id 在流式快照之间稳定（pi 只追加，不重排），所以折叠状态能跨重新派生与虚拟化存活。

### 7.4.5 增量 Markdown：为什么按块

`BlockMarkdown` 用 `marked` 的块级 lexer 切分，每个块一次记忆化的 react-markdown 渲染。
流式增量只重新解析尾部块，而不是整条消息。

不变量：

- 增量切分 == 全新切分；
- 块渲染 == 整文档渲染（除已知缺口）；
- 链接/脚注定义会让消息在此后整体渲染；
- 回退情形（未闭合 HTML、惰性行、缩进代码）**只合并块，绝不切块**。

已知缺口（`marked` 与 micromark 结构读法不同）在 SPEC 中列明；它们被接受，而不是“待修”。

### 7.4.6 工具注册表：能力与呈现的解耦

`toolRegistry.tsx` 是唯一扩展点。一个工具有两个侧面，由**工具名**连接：

| 侧面 | 注册位置 |
| --- | --- |
| 能力 | 服务端 pi 会话（自定义工具 / pi 扩展 / skill） |
| 呈现 | `toolRegistry` |

一次注册包含：

- `renderer`：专用卡片正文；
- `summary`：折叠头部的一行摘要（纯函数）；
- `chrome`：`"card"` 或 `"bare"`；
- `prominence`：`"routine"` 或 `"primary"`；
- `defaultExpanded`。

解析顺序：**精确名 → 最长应用本地前缀（当前只有 `mcp__`）→ 默认渲染器**。
`"bare"` chrome 无条件隐含 `primary`；错误注册不能静默破坏折叠。

### 7.4.7 扩展 UI：在 chat 内提问，不在模态框

每个 pi 的 `select`/`confirm`/`input`/`editor` 都渲染为 chat 内联提示（`ExtUiPrompt`），
位于 `AttentionLine` 之下：

- select：每个选项一个按钮，无额外 Cancel；
- confirm：Cancel / OK；
- input / editor：一个字段 + Cancel / Submit。

边界：

- 停靠提示上限 chat 高度的 60%；
- 带 `toolCallId` 的请求**就在那个工具上**提问（`ToolPromptSlot`）；
- 带 `toolInput` 的请求把参数列在提示框内，不截断、不藏进 Show more；
- 新请求通过 polite 活区播报；首次挂载自动揭示，之后不重复。

### 7.4.8 `ChatActions`：渲染器回话的唯一接缝

渲染器不 import store/transport。它们通过 `ChatTranscript` 提供的 React 上下文：

| 动作 | 作用 |
| --- | --- |
| `respondToTool(toolCallId, response)` | 回答交互工具；宿主拒绝时 reject |
| `focusComposer()` | 自行了结的渲染器把焦点还给输入框 |
| `openSubagentTranscript(childSessionId)` | 打开只读子会话转录 |
| `markResultSeen(target)` | 后台结果被打开后标记已见 |
| `readMcpOutput(toolCallId)` | 读取本 chat 的 MCP 输出 |
| `openMcpSettings()` / `openProviderSettings()` | 深链 Settings |
| `findModel(ref)` | 在完整目录（含隐藏模型）解析运行模型 |

没有 provider 时（计划页、独立渲染）这些动作隐藏，而不是报错。

### 7.4.9 轮次回执优先于工具参数

`deriveRows` 接收**本 chat** 已记录的 agent 运行（`turnsByWorkspace` 按 `sessionId` 过滤）。

`matchTurnReceipt` 按开始时间把回执与回合连接，从回合开场点到下一个开场点。
有回执时，文件徽标块列出**宿主快照里的路径**与每文件 `+/−`，读作 `N files changed · +a −r`。

为什么优先于工具参数：工具参数会漏掉 shell 写入、格式化器与重命名；
快照才是 agent 实际留下的东西。

### 7.4.10 吸顶面包屑与滚动

当会话记录顶部可见内容仍在展开的 Activity → Thinking → 工具层级内、而原始头部已滚出视口时，
一条紧凑面包屑覆盖在滚动容器上，显示根到叶路径。

它：

- 不重排会话记录内容；
- 不制造并行的导航/折叠状态；
- 不扰动 Virtuoso 的最新边缘、阅读带与跳转到消息行为。

---

## 7.5 shell 与布局：组装根

[shell/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/shell/SPEC.zh.md)。

### 7.5.1 `Shell` 拥有什么

`Shell` 是唯一组装根：

- topbar（身份、连接状态、Settings、位置条、配额）；
- 活动项目/工作区路由；
- 主题副作用（唯一修改 `[data-theme]` 的地方）；
- 全局快捷键；
- 单次挂载的 Settings / 同意 / 问卷 / 通知前言 / Toaster；
- 工作台与 store、持久化、面板、transport、错误边界之间的集成。

### 7.5.2 子模块与依赖

| 子模块 | 职责 | 关键约束 |
| --- | --- | --- |
| `layout/` | 纯帧/视图变换、投影、渲染引擎 | 不 import 功能面板/store/transport/持久化 |
| `layoutState/` | 本地水合、校验、持久化、原子装入 | 拥有“布局文档”，不是 store |
| `layoutIntents/` | 一次性排布意图路由 | 处理即消费，重挂载不能重放 |
| `chatReconciliation/` | 宿主会话/本地放置/缓存/历史收敛 | 校验精确 chat 深链 |
| `terminalReconciliation/` | 宿主终端目录/本地放置收敛 | 不拥有 PTY 生命周期 |
| `legacySelection/` | 把工作台注意映射到旧编辑器/终端镜像 | 临时适配器 |

依赖方向：`layoutState → layout`；`chatReconciliation → layout + layoutState`；
`terminalReconciliation → layout`；`layoutIntents → layout + chatReconciliation + terminalReconciliation`。

### 7.5.3 帧、视图、注意

| 概念 | 范围 | 例子 |
| --- | --- | --- |
| `WorkbenchFrame` | 整个前端表面的拓扑/几何 | 侧边组、底部组、对齐 |
| `WorkspaceViewState` | 每个工作区的资源放置/顺序 | 中心标签、侧边工具 |
| attention | 每个工作区的选择/焦点/导航时钟 | 最后聚焦的中心组 |

切换工作区**只改变资源内容与注意**，绝不改变帧拓扑、Projects/Specs/Files/Review 的放置、
侧/底几何、折叠、可见性、对齐。

### 7.5.4 不重挂载的切换

两条 React 不变量：

1. `WorkspaceWorkbench` 在切换期间保持挂载，靠 `workspaceId` prop 重新指向，**绝不以工作区为 key**；
2. 资源体既按资源 id、也按工作区做 key，因为某些布局 id 是确定性的（初始终端是 `terminal:thinkrail-initial`）。

因此切换工作区不会为 chrome 绘制淡出/骨架/空帧；chat 转录、编辑器、终端保留 DOM 与状态。

### 7.5.5 形状变化也不重挂载

改变形状的帧命令（折叠/展开、侧/底可见性、对齐、组增删、预置应用/重置）以命令式方式强制已提交尺寸，
绝不重挂载 `ResizablePanelGroup` 或资源体。

唯一例外：在“外层列”与“对齐行”之间迁移的侧堆栈会重挂载，因为它换了父级。

### 7.5.6 Project Home 与引导

没有活动工作区时，`Shell` 挂载 `ProjectHome`：

- Welcome 与项目导航器并排；
- 使用独立的本地几何，绝不用工作台帧；
- 与帧的唯一联系是原始默认值（左侧宽度与 Balanced 预置一致）。

引导流程活跃时（`selectOnboardingActive`），工具栏与 Projects pane 完全不渲染，
Welcome 面板就地保持挂载，因此完成向导后交接对话框仍在。

### 7.5.7 全局快捷键

| 快捷键 | 作用 |
| --- | --- |
| `Ctrl+R` | 为本地选中的 chat（否则工作区最近 chat）打开历史 |
| `Mod+B` | 切换左侧，恢复本地注意或符合条件的单例工具 |
| `Mod+J` | 切换右侧 |
| `Mod+Shift+J` | 切换底部，恢复底部注意或终端创建表面 |
| `Mod+N` / `Mod+Alt+N` | 打开 New workspace 对话框（上下文项目） |
| `Mod+K` | 打开 `SessionSwitcher` 模态面板 |

规则：

- 字母快捷键匹配物理 `KeyboardEvent.code`，不用依赖布局的 `key`（原生 shell 的关闭/退出是例外）；
- 布局与创建类快捷键在 xterm 内仍归应用所有，模态打开时抑制；
- 没有活动工作区时，右/底快捷键不起作用也不吞浏览器快捷键；
- 终端内的 `Ctrl+R` 仍归 xterm。

### 7.5.8 错误韧性

- 每个独立挂载的工作台资源体都有自己的 keyed 区域边界；
- 一个出错的懒加载面板不能让工作台 chrome、同级组或 shell 变空；
- 切换工作区或资源会重置卡住的区域错误；
- 失败的动态 chunk → 页面重载，而不是重试同一个过期模块；
- `main.tsx` 保留围绕 `Shell` 的最后兜底边界。

**边界捕获范围**：React 边界只捕获渲染 + lazy 导入抛出的错误；事件处理器、effect、
被拒绝 promise 的错误走 `transport.errorText()` → 错误轮次/提示。

---

## 7.6 panels：布局无关的功能视图

[panels/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/panels/SPEC.zh.md)。

### 7.6.1 面板的契约

- 面板填满容器；
- **从不感知自己的排布方式**（因此移动端 shell 是叠加层，而不是重写）；
- 只接收资源身份、可见性与容器边界；
- 不读取组顺序、不画自己的停靠条。

### 7.6.2 Projects 树与 Settled 置物架

`ProjectTree` 的规则：

| 元素 | 规则 |
| --- | --- |
| 顶层项目行 | 始终可见 chevron + 名称 + 折叠时工作区计数 + 固定右缘 Create workspace `+` |
| 项目上下文菜单 | Create workspace / Open existing worktree / Rename（协议门禁）/ Close project |
| 工作区行 | 两行：名称 + 分支（仅不同时显示）；kebab 菜单 |
| Default 工作区 | 固定最前、House 图标、无 Rename/Remove |
| 注意点 | 静态绿/强调色，二元（needs-input 或未读结果） |
| Working | 共享 `RunningIcon` 替代身份图标，不添加第二个标记 |

Settled 置物架：

- 只在宿主支持 `WORKSPACE_SETTLE_PROTOCOL_VERSION` 时渲染；
- 活跃行按最新创建优先，**绝不按活动时间重排**（agent 工作时行不移动）；
- 已结算行默认折叠、紧凑单行、带原因 chip（merged / closed / idle / by you）；
- 第一次自动移动在每个浏览器只通告一次。

### 7.6.3 Files / Changes / Review

| 面板 | 关键行为 |
| --- | --- |
| Files | 两种模式（All / Changed）；树状态按工作区持久化；`revealInFiles` 展开祖先并记录已见 |
| Changes | 一个 scope 覆盖所有变更文件；section 自己读 `git.diffFile`；`viewed`/`activePath`/折叠覆写/一次性 reveal |
| Review | 一个屏幕、按文件手风琴；草稿→进行中→已解决；发送后不删除、不回滚、不重开 |

`DiffTab` 的 scope 是**标签页身份的一部分**：`${workspaceId}:diff:${scopeKey}:${path}`；
`ChangesTab` 的身份是 `tupleKey("changes", workspaceId, scopeKey)`。
目标 ref 移动时标签页重读，但**目标 ref 不属于标签页 id**（分支名钉不住任何东西，只有 commit sha 不可变）。

### 7.6.4 长操作反馈

启动 agent 会话需要数秒，因此绝不静默：

- `beginChatStart` / `endChatStart`（按工作区计数器，允许重叠）；
- `beginWorktreeCreation` / `endWorktreeCreation`（按项目）；
- 消费方在结果将出现的位置显示行内 pending，而不是全局 spinner；
- 工作区移除会丢弃该计数器。

### 7.6.5 面板与资源渲染器的分派

- 文件/diff 栏**只分派，不分类格式**；
- 每个栏描述宿主元信息、按意图与手机类别解析注册表、惰性挂载候选；
- 标签页保留 `rendererId` + 不透明 `viewState`；
- 两个以上候选变成有序切换，id 是测试钩子。

---

## 7.7 资源渲染器与 Markdown

### 7.7.1 一个注册表服务所有展示文件的地方

[packages/resource-renderers/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/resource-renderers/SPEC.zh.md)：

- Files/Diff 窗格、文件对比、扩展工具卡都用同一个注册表；
- 11 个渲染器：code、markdown、binary、LFS、image、SVG、CSV、JSON、notebook、PDF、HTML；
- 宿主通过 `ResourceHostProvider` 提供 markdown、code、`bytesUrl`、`openFile`、`readFile`、
  `subscribeFileChanges`、`FileReview`、`ReviewSend`。

`ResourcePreview` 可单独预览一个文件，也可给 `before` 做前/后 diff。
渲染器实现动态导入；`GuardedRendererDiff` 负责大 diff 提示与虚拟化。

### 7.7.2 Markdown 的安全链接

assistant 散文中的链接：

- 工作区相对链接 → 在 ThinkRail 内打开目标；
- 绝对路径仅在规范化后位于活动工作区内时接受；
- URL scheme、协议相对 URL、片段、外部/不安全路径 → 普通安全新标签页锚点；
- 百分号编码路径先解码一次，再包含性检查（编码分隔符/目录穿越不能绕过）；
- 仅 assistant 的 remark 变换处理 Windows 盘符与反斜杠路径。

文件查看器中的仓库 Markdown 保持 CommonMark 语义（与 GitHub 一致）。

### 7.7.3 Mermaid

反引号围栏的 ```mermaid 块通过 `@thinkrail/ext-visualize/web` 的 `MermaidView` 渲染：

- 全屏平移缩放；
- 出错回退源码；
- 在每个 Markdown 表面一致（chat、文件/规格预览）；
- 挂载前作为高亮源码渲染，因此静态上下文（`renderToStaticMarkup`）会正常降级。

### 7.7.4 资源字节 URL 的唯一函数

所有宿主字节路由都经 `panels/resourcePane.resourceBytesUrl` 在 transport 的 HTTP origin 上组合。
这是为什么 Vite 必须代理 `/files` 与 `/blob`：未代理的路由会返回 SPA 的 `index.html`，
`<img>` 什么都不显示，PDF 抓取以 HTML “成功”。

---

## 7.8 样式、主题、i18n、字体

### 7.8.1 样式：token 工具类，不是任意值

- Tailwind v4 工具类映射到生成的语义 CSS 变量；
- 禁止内联 `style` 对象（渲染器实测几何是唯一例外）、raw hex、内部调色板名、未知 token 工具类；
- 颜色命名**语义角色**（`bg-container-elevated-bg`），不是调色板条目；
- 着色是四步 alpha 尺度上的 token，不是 `/40` 修饰符；
- 间距是规范数值尺度 `0/2/4/6/8/12/16/24/32/40/64`，步长名就是像素值；
- 圆角 token 就是 Tailwind 命名空间：`rounded-xs/sm/md/lg`。

守卫：`styles/colorUsage.test.ts`、`spacingUsage.test.ts`、`buttonUsage.test.ts`。

### 7.8.2 排版：唯一来源 + 生成类

- `styles/typography.json` 是唯一来源；
- `styles/generated/typography.css` 发出 `--tr-*` 与语义类（`tr-text-ui`、`tr-title-dialog`、`tr-code-text`、`tr-prose-chat`）；
- **不得引入手写排版工具类**；要改排版就改 JSON；
- 只有第三方集成（Monaco/Pierre/xterm/mermaid）可以命名原语字体家族；
- `<pre>` / `<code>` 即使容器已有类也必须自带一个排版类。

### 7.8.3 图标：Line 默认，Fill 表示活动

- `@remixicon/react`，按名导入，不用 `<Icon>` 包装，不用 `size=` prop；
- 三档尺寸：`size-12`（chat 内容）、`size-14`（紧凑 shell/导航）、`size-16`（所有展开/折叠箭头）；
- 按钮内部图标由 `Button`/`IconButton` 按按钮尺寸定尺寸；
- 图标↔文本间距统一 `gap-4`；
- 选中/活动项切换 `Ri…Fill`，其他保持 `Ri…Line`。

Remix 缺少的字形放在 `public/custom-icons/`，经 `CustomIcon`（`mask-image`）渲染。

### 7.8.4 主题：manifest 是数据，不是代码

`src/themes`：

- 内置主题是严格完整的 `*.theme.json`；
- 用户在 `~/.thinkrail/themes` 放同一份 manifest；
- 清单自包含（无继承），只含规范颜色数据；
- 不能改变布局/排版/动效，不能注入 CSS/代码；
- 应用是原子的：写入完整变量集、`color-scheme`、对比度元数据，**最后**才 `data-theme`；
- 系统模式只有一个 `prefers-color-scheme` 监听器；
- 用户 id 在 `user.` 命名空间，永不遮蔽内置主题。

新增一个内置主题 = 新增一个文件 + 重建；不改 contracts、CSS、测试。

### 7.8.5 i18n：唯一活动语言

- `AppConfig.locale` + 浏览器语言 + 内置目录 + 宿主本地文件 → 共享 i18next；
- 组件只调 `useT()`；
- 英文在首帧前同步安装，因此连接期间不会渲染出 key；
- 选中的本地 tag 消失时渲染英文，偏好保持不变；
- 日期/数字用 `currentLocale()`，不用硬编码 `"en-US"`。

### 7.8.6 字体：自托管，无 CDN

- `styles/generated/fonts.css` 导入自托管可变字面（Geist Variable + JetBrains Mono Variable）；
- 哪些包属于哪个家族在 `typography.json` 的 `selfHosted` 声明；
- Vite 输出 woff2 到 `dist/assets`，CLI 内嵌；
- 没有字体 CDN：宿主常常离线，外部链接还会把首绘推到第三方之后。

---

## 7.9 Web 扩展：注册顺序与工具呈现

### 7.9.1 `registerWebExtensions()` 必须在读第一份转录之前

`extensions/registry.ts` 把有序描述符喂给 chat 的工具注册表。
计划与评审面板从注册表的 `interactive` 注册派生交互状态，因此注册表必须在第一份会话记录之前完整，
而不只是在 chat 视图挂载之后。

`main.tsx` 在 `createRoot` 之前调用它，就是为了这个不变量。

### 7.9.2 模块边界：为什么 panels/chat 没有 barrel

`apps/web/SPEC.zh.md` 明确：`panels`/`chat` **按设计逐文件导入**。

原因：

- 给它们加 barrel 会把懒加载的 Monaco/shiki/xterm chunk 拉进急切 bundle；
- 会破坏 shadcn 的按原语惯例。

多数模块（`navigation`/`transport`/`store`/`updates`/`prompt`/`lib`/`extensions`/`themes`/`i18n`/`auth` 等）
都有 `index.ts` barrel；只有 `panels`/`chat` 按设计逐文件导入。

### 7.9.3 依赖规则

- `apps/web` 只依赖 `contracts`、`locales`、`ui`、`extension-api/web`、`resource-renderers`、`thinkrail-extensions/*/web`；
- **绝不值导入 `pi`；绝不导入 `server`/`shared`**；
- 面板不横向导入另一个面板；
- 没有任何东西导入 `shell`（它是组合根）。

---

## 7.10 通知、更新与登录

### 7.10.1 通知：派生消费者，不引入第二套真相

[notifications/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/notifications/SPEC.zh.md)：

- 触发信号只有一个：**绿色 attention 点亮起**（`needsInput !== null || completionUnread` 从 false → true）；
- 宿主已把 `agent_end` 与 `agent_settled` 折叠成 `completion`，本模块**永不监听原始 agent 事件**；
- 流水线：观察者 → 1 秒收集窗口 → 消费者；
- 消费者门禁：总开关、权限、窗口未聚焦、点仍亮、且不是本窗口自己的“标记未读”；
- 一个存活会话 → 打开它的 chat；多个 → 打开 `SessionSwitcher`。

**抑制规则只有一条：窗口聚焦 ⇒ 永不通知。**

### 7.10.2 更新：原生权威与宿主生命周期

- `__THINKRAIL_NATIVE_UPDATES__` 存在时总是胜过宿主状态；
- 原生快照留在 shell 本地，永不进 Zustand；
- 宿主快照经带版本 wire 到达，在 store 同步；
- 宿主只暴露 available/running/succeeded/failed + **Run Update**；
- 任何命令文本/诊断都不跨 wire。

### 7.10.3 登录：纯呈现 + store 归约

- `auth` 只拥有 `LoginState` 类型与 props 驱动的 `LoginDialog`；
- 唯一的 `activeLogin` 与 `foldLoginFrame` reducer 在 `store`；
- 登录是**扁平、无会话**的：它在 Welcome 界面上运行，那时还没有 chat；
- `provider.login` 帧经 `wireTransport → applyLoginFrame`；
- 一次新登录取代同 provider 未完成登录；
- 粘贴兜底只在活动 URL 是 `http:` 回环地址时显示。

---

## 7.11 前端必须做对：反模式表

| 反模式 | 为什么错 | 正确做法 |
| --- | --- | --- |
| 组件自己 `new WebSocket()` | 第二套 id/ack/重连语义 | 用 `getTransport()` |
| 组件自己推导 live/settled | 三个表面会不一致 | 用 `selectWorkspacePartition` |
| 把布局/折叠写进 `AppConfig` | 跨客户端污染 | 浏览器本地持久化模块 |
| 对 `workspace.updated` 做 merge | 无法清掉宿主丢掉的字段 | 整条替换 |
| 流式发送乐观追加轮次 | 末轮回显去重会失手 | 只进 `queue`，等 `message_start` |
| 用 `agent_end` 清 `isStreaming` | 重试/压缩/续跑还没完 | 用 `agent_settled` |
| 逐条 `pi.event` setState | 流式高频导致大量渲染 | `piEventBatcher` 32ms/128 |
| 在组件 effect 里反复订阅 | 事件顺序不可控 | 全部在 `initTransport` 一次订阅 |
| 用 `message` 拿错误原因 | 宿主已归一化 | `errorText()` / `RequestError` + `wsErrorCode` |
| 老宿主上直接发新方法 | 请求失败才显示错误 | `supports*` 门禁 |
| 把 `data-theme` 设成空 | 消费者看到半个调色板 | 原子应用，最后发布 id |
| 手写排版工具类 | 与字号系统漂移 | 改 `typography.json` |
| 用裸 `Loading…` 文字 | 无形状、无宽限、看起来像失败 | `SkeletonRows` / `Loader2` |
| 在 spinner 旁显示 Retry | 两个矛盾信号 | 只在真正失败后显示 |
| 把扩展对话框做成模态 | “奇怪弹窗”，且会话可能不存在 | chat 内联 `ExtUiPrompt` |
| 让 chat 渲染器 import store | 破坏 props 驱动与复用 | 用 `ChatActions` 上下文 |
| 加 barrel 到 panels/chat | 拉入 Monaco/shiki/xterm 急切 chunk | 按设计逐文件导入 |
| 在浏览器里擦洗错误 | 宿主才是唯一清洗者 | 原样 `client.reportError` |

---

## 7.12 追踪练习

### 练习 1：为什么 `welcomeGeneration` 与 `connectionGeneration` 不能合并？

<details><summary>参考答案</summary>

`connectionGeneration` 回答“这是第几次 socket 连接”，用于所有重连水合；
`welcomeGeneration` 回答“我们是否已经拿到一份完整 welcome”。
socket 连接成功但 welcome 还没到时，协议版本、项目列表、能力都不可用；
把两个边合并会让导航/能力读取提前放行，看到半个 welcome。

见 [store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md)。
</details>

### 练习 2：客户端断线 5 秒后重连，一个在途的 `session.prompt` 会怎样？

<details><summary>参考答案</summary>

`request` 在 socket 关闭时保留 `pending` 里的 frame 与 id。
`onopen` 先发 `{ resume: [...pending.keys()] }`，再把 frame 原样重发。
宿主按 `(clientKey, requestId)` 去重：如果原请求已结算且结果仍在重放预算内，直接返回缓存结果；
如果结果超过字节预算，返回 `RequestReplayUnretainedError`，**不会重新执行**。

见 [transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L84-L131)。
</details>

### 练习 3：为什么 `subscribe` 对某些通道立即回调，对另一些不？

<details><summary>参考答案</summary>

`WsTransport.subscribe` 会检查 `latest`。可重放通道（`project.updated`、`workspace.*`、`settings.changed` 等）
保存最后值，因此迟到订阅者立即拿到当前快照。
`NON_REPLAYABLE_CHANNELS` 里的通道（终端流、`session.deleted`、`mcp.status` 等）不保存，
因为它们只能追加、是一次性事实、或应由下一次权威读取修复。

见 [transport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/transport.ts#L21-L30)。
</details>

### 练习 4：`pi.event` 已经排队 20 条，此时来了一个 `session.state`，会发生什么？

<details><summary>参考答案</summary>

`beforeDispatch` 调用 `shouldFlushPiEventsBefore(message)`；它不是 `pi.event`，所以返回 true，
先 `piEvents.flush()`，把 20 条事件作为一次原子 store 写入。
之后 `session.state` 才被派发，因此运行时转录/渲染状态先于引用它的宿主状态。
这保住了跨消息顺序与 store 的转录修订围栏。

见 [wireTransport.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/transport/wireTransport.ts#L245-L263)。
</details>

### 练习 5：为什么 `steer` / `followUp` 不在发送时立刻追加用户轮次？

<details><summary>参考答案</summary>

因为它们的文本可能被 pi 排队/合并/拒绝，规范位置由 pi 的 `message_start` 决定。
如果乐观追加，末轮回显去重会在“追加与回显之间出现 assistant 内容”时失手，
产生重复且错位的排队气泡。
所以文本只进 `queue`，轮次只在规范 `message_start` 落地；实时与水合因此一致。

见 [store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md)。
</details>

### 练习 6：为什么 `workspace.updated` 必须整条替换，而不能合并？

<details><summary>参考答案</summary>

宿主可能**丢掉**可选字段：`diffBase` 回到创建基线、最后一条 `skillOverrides` 被移除、
`subagentsOverride` 回到继承。
合并无法清除这些字段，客户端会继续按宿主已不再持有的值贴标签、做读取键。
替换才是权威快照的正确折叠；列表算出的 `diffStats` 是唯一需要沿用的聚合。

见 [store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md)。
</details>

### 练习 7：切换工作区为什么不会让 chat 转录重新挂载？

<details><summary>参考答案</summary>

`WorkspaceWorkbench` 在切换期间保持挂载，靠 `workspaceId` prop 重新指向，绝不以工作区为 key。
帧拓扑、面板放置、几何与注意都不变；只有资源内容与注意改变。
资源体按 `(resource id, workspaceId)` key，避免确定性 id（初始终端）跨工作区串台。

见 [shell/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/shell/SPEC.zh.md)。
</details>

### 练习 8：一个扩展在 `session_start` 里发了 `setTitle` 和 `notify`，但客户端此时还没有该会话的运行时会怎样？

<details><summary>参考答案</summary>

发射即忘的帧进入 `extUiOrphans`（有界 64）。运行时一安装（`openChatSession` 或 `hydrateSession`）
就通过 `replayExtUiOrphans` 排空。
但**对话框**（`select`/`confirm`/`input`/`editor`）不缓冲：它是一次往返的一半，重放会让 chat 卡在
一个没人在等的问题上。

见 [store/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/store/SPEC.zh.md)。
</details>

### 练习 9：为什么 `ChatView` 要把 `ChatTranscript` 拆出来？

<details><summary>参考答案</summary>

隔离高频变动。`ChatTranscript` 用 `useShallow` 订阅渲染所需的流式字段
（turns、toolResults、currentAssistantId、toolResponses、isStreaming、settlementTick 等），
并拥有滚动容器与行；`ChatView` 的 chrome 外壳只读少量粒度化、很少变化的字段。
因此文本/思考/工具增量不再重渲染头部/输入框/队列/对话框。

见 [chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md)。
</details>

### 练习 10：为什么主题应用必须先写变量，最后才写 `data-theme`？

<details><summary>参考答案</summary>

`data-theme` 是给通用消费者（Monaco/xterm/mermaid）的重建信号。
如果先发布 id，消费者会在变量尚未写全时重建，看到半个调色板。
因此顺序是：完整变量集 → `color-scheme` → 对比度元数据 → 最后 `data-theme`。

见 [themes/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/themes/SPEC.zh.md)。
</details>

---

## 7.13 一页速查

```
启动（main.tsx，全部在 render 之前）
  themes → theme hint → fonts → transport → error reporting → boundary reporter
  → chat/model/project/filesTree persistence → navigation → web extensions
  App = ErrorBoundary > TooltipProvider > AppResourceHost > Shell

transport
  request: id trpi_<seq>, 60s timeout, pending frame replayed on reconnect
  ack: microtask batch; resume: full unresolved list before replay
  subscribe: replayable channels get latest immediately
  non-replayable: terminal.* / session.deleted / session.resourcesChanged / mcp.status
                   / provider.changed / feedback.interview
  reconnect backoff 500ms → 10s
  pi.event → batcher 32ms / 128 → handlePiEvents (one atomic write)
  beforeDispatch: flush Pi batch before any non-pi.event message
  httpBase(): /files + /blob

store
  connectionGeneration vs welcomeGeneration
  installWelcomeSnapshot: one write, then advance welcomeGeneration
  updateWorkspace: replace, never merge
  applyWorkspaceRemoved: row + state + recents + fallback focus
  reduceSessionEvent: pure; text/thinking append; partialResult replace
  agent_settled ends streaming; agent_end does not
  idle prompt optimistic; steer/followUp only queue
  error turn: rejected send vs settled failure
  ext-ui orphans 64; dialogs never buffered
  selectors own derived rules

chat
  turns → deriveRows → rows → ChatTranscript
  projectRows: newest-first groups, stable ids
  foldState: manual toggle wins
  BlockMarkdown: incremental tail parsing
  tool registry: exact → prefix (mcp__) → default
  ExtUiPrompt inline; ChatActions is the reply seam
  turn receipts beat tool args

shell/layout
  Shell = composition root
  frame / workspace view / attention
  workspace switch = no remount
  shape change = no remount (except side stack reparent)
  error boundary per resource body

styles
  token utilities only; semantic color names; spacing steps = px
  typography from typography.json; no handwritten type utilities
  icons Line default / Fill active; 12/14/16
  themes: atomic variables then data-theme; user namespace user.
  i18n: one i18next; English synchronous; useT everywhere
  fonts self-hosted, no CDN
```

---

### 与主指南的对应

| 主指南小节 | 本专题的展开 |
| --- | --- |
| 7.1 启动与初始化顺序 | 7.1、7.2.7、7.8 |
| 7.2 transport | 7.2 |
| 7.3 store | 7.3 |
| 7.4 chat | 7.4 |
| 7.5 shell 与布局 | 7.5 |
| 7.6 样式、图标、i18n、主题 | 7.8 |
| 7.7 资源渲染器 | 7.7 |
| 7.8 前端必须做对 | 7.11 |

> 建议的下一步：把本章与 [第 5 章专题](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH05-LIFECYCLE.zh.md)、
> [第 6 章专题](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE-CH06-SERVER.zh.md) 并排读：
> 第 5 章给你时间轴，第 6 章给你宿主空间，本章给你客户端空间。三者对上之后，
> “为什么这条消息会这样渲染”就只剩一个答案。
