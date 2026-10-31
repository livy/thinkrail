# notifications —— 应用外 attention 通知

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

当某个 worktree 需要用户、而 ThinkRail 未处于聚焦状态时，发出应用外通知 —— 一个 agent 提问、完成或
失败。本模块是会话状态转换的**派生消费者**：它不引入新的真相来源，只提供 store 无法知道的客户端侧
要素 —— 窗口焦点、收集窗口、浏览器权限、格式，以及总开关。

触发信号只有一个：**绿色 attention 点亮起**（`needsInput !== null || completionUnread` 从 false
上升到 true，与 [[submodule-web-store]] 使用的同一个谓词）。每种原因都被统一覆盖 ——
`ask_user_question`、ext-UI 对话框，以及一次未读完成（`succeeded`／`failed`／`interrupted`）。一次
被用户取消的运行在宿主上永不置 `completionUnread`，因此它永不点亮该点，也永不通知。同理，当一个空闲
chat 仍在运行后台命令或子 agent 时，宿主把 `completionUnread` 保持为 false，因此该边沿只在那项工作
结束时触发（在此期间该会话显示为运行中）。宿主已经把 `agent_end` 与 `agent_settled` 折叠成
`completion`，因此本模块永不监听原始 agent 事件。

## 流水线（Pipeline）

生产者 → 队列 → 消费者，因此“是否通知”的决定被推迟到它真正有意义的时刻：

- **观察者**（`useAttentionNotifications`，由 [[submodule-web-shell]] 挂载一次）：订阅 store，并把
  每个上升边沿喂给引擎。store 在每次变更时都会通知，因此该 hook 在重建候选之前先以 `sessionStateClock`
  门控 —— 流式增量永不触碰 attention。上升边沿的决定本身是纯函数 `attentionObserver`（基于
  `detectAttention`）：它在首个已安装快照上以及每次重连（`connectionGeneration` 变化）时静默预热，
  因此一次全新的 `installSessionStateSnapshot` 永不引发风暴。
- **引擎**（`attentionNotificationEngine`）：一个无 DOM 的队列，带一个全局收集窗口
  （`ATTENTION_WINDOW_MS`，1s）。累积的边沿作为一批冲刷；每个会话的最新事件胜出。
- **消费者**（冲刷）：只有当每道门禁都通过时才发出 —— 总开关打开、通道权限已授予、窗口**未**处于
  聚焦状态、且该会话的点**仍然亮着**（已应答的边沿被丢弃），且其原因不是本窗口自己的*标记为未读*
  （`selectCompletionMarkedUnread`，精确的完成 id；当请求在途时，只算那些在它开始时仍可重新点亮的
  完成）—— 手工重新点亮一个结果在这里永不通知，虽然同一所有者的另一个未聚焦窗口仍可能为它通知。
  一个存活的会话会打开它的 chat（`activateWorkspaceFromRoute`，落在最新消息）；多个则打开跨工作区的
  `SessionSwitcher` 面板（[[submodule-web-store]] 上的 `openSessionSwitcher`），让用户挑出要处理的
  那个，而不是落进一个可能不显示正确 chat 的窗口。`formatNotification` 把标题保留为加粗的
  **ThinkRail** 标识，把 worktree 作为 `subtitle`，把原因作为 `body`（多个时为
  `N worktrees need your attention`）；按会话的 `tag` 让浏览器替换而不是堆叠。

## 通道（Channels）

消费者通过一个 `NotificationChannel`（permission / requestPermission / show）发出，由
`selectNotificationChannel()` 按能力挑选 —— 抑制、格式、队列与设置在其之上保持与通道无关：

- **Web**（`webNotifications`）：页面 Notifications API。它没有 subtitle 字段，因此把
  `subtitle · body` 折成一行，并使用浏览器权限流程。它还设置仅符号的 ThinkRail 图标
  （`/favicon.svg`，即浏览器标签／shell 标志的艺术字），被 Chrome/Edge/Firefox 采纳；Safari 忽略
  页面通知的图标并显示自己的。在移动端构造函数可能抛出（Android Chrome 需要 Service Worker；
  iOS Safari 需要已安装的 PWA），并被吞掉当作无操作。
- **Desktop**（`desktopNotifications`）：经冻结的 `__THINKRAIL_NATIVE_NOTIFICATIONS__` 桥接
  （[[module-desktop]]）使用原生 OS 通道，把 `subtitle` 作为独立字段使用。OS 拥有权限，因此
  `permission()` 总是 `granted`，应用内前言永不显示。Electrobun 的点击回调尚未落地，因此原生通知
  不携带点击动作。

选择只是能力探测（桥接是否存在），绝不是一个 `desktop` 分支。两个通道都没有 Service Worker：
完全关闭的应用没有存活客户端，按设计什么都不发出。

**依赖 OS（desktop）。** 原生渲染是固定的 Electrobun 2.0.5 后端，它接受 `title` + `body` +
`subtitle` + `silent` 并按平台渲染：macOS 用 `UNUserNotificationCenter`（原生
title/subtitle/body，首次显示时由 OS 请求授权），Windows 用 `Shell_NotifyIconW` 托盘**气球**
（title + body；按 SDK 的说法 `subtitle` 折为额外一行 —— 不是 Windows Toast），Linux 用
libnotify/D-Bus（依赖桌面环境，有 Wayland 注意事项，subtitle 同样折入）。macOS 的 bundle id 来自
`apps/desktop/electrobun.config.ts`（`identifier: ai.thinkrail.app`，名称 `ThinkRail`），且 macOS
自己提供应用图标（原生调用没有图标字段）。Windows 气泡显示 Electrobun 的通用应用图标。它自身不携带
App User Model ID，因此 Windows 显示的发送方跟随进程的 ID（[[module-desktop]]）；该归属未经验证。
**限制：** 桥接没有暴露任何读取 OS 权限状态的方式，因此 desktop 通道无条件报告 `granted`；如果用户
在 OS 层面拒绝了通知，调用会被接受并静默地不显示。查询／处理 OS 权限（以及点击回调）被推迟。

## 抑制（Suppression）

一条规则：**窗口聚焦 ⇒ 永不通知。** 如果窗口持有焦点，用户就在 ThinkRail 里，无论打开了哪个
工作区／chat 都能看到那个点，因此无需按 chat 的可见性检查。

焦点信号是**通道感知**的：`NotificationChannel.isWindowFocused()` 让每个通道使用可用的最佳 API。
desktop 通道从原生桥接获得 OS 层面的窗口焦点
（`__THINKRAIL_NATIVE_NOTIFICATIONS__.isWindowFocused()`），当 DevTools 或其他窗口内 chrome 持有
渲染器焦点时它能正确报告 `true` —— 原生窗口在 OS 层面仍然是聚焦的。浏览器通道回退到
`document.hasFocus()`，在该场景下它返回 `false`；这是浏览器 API 的已知限制，不影响最终用户（只影响
打开了 DevTools 的开发者）。

**平台注意事项（Windows）：** 在 Windows 上，DevTools 作为独立的原生窗口打开；聚焦它会为主
`BrowserWindow` 触发 `blur`，因此 `isWindowFocused()` 返回 `false` —— 与浏览器通道相同的限制。
在 macOS 上，DevTools 停靠在同一个原生窗口内，主窗口焦点得以保留。Electrobun 2.0.5 只暴露按窗口的
`focus`／`blur` 事件，没有应用级激活信号，因此跨窗口的方案需要未来的 Electrobun API。

## 权限（Permission）

权限在最清晰的用户手势处请求。主要、可见的路径是 Settings 面板
（`panels/NotificationsSettings`）：打开开关，或在权限仍为 `default` 时点击 **Allow in browser**，
会直接调用 `Notification.requestPermission()` —— 面向在场用户的获取，不依赖于一个失焦事件（该面板
还显示 blocked/granted/unsupported 状态）。对于已经启用的客户端，应用内前言
（`NotificationPermissionPrompt`）是离开路径上的兜底 —— 只会在真正要通知的第一批（已通过抑制与
开关）之前、系统请求之前显示，从而降低硬拒绝的风险。"Enable" 调用
`Notification.requestPermission()`；"Not now" 小睡 7 天（`notificationPrompt`，浏览器本地存于
`localStorage`）；`denied` 状态永不再次自动提示，并给出一个手工重新启用的提示。权限是按浏览器的，
因此小睡是客户端本地的，而总开关是宿主同步的应用配置且默认为开（通知仍需要浏览器授予）。

## 边界（Boundary）

- **拥有：** attention 通知流水线、`NotificationChannel` 接缝，以及两个通道实现（浏览器 Notifications
  API + 原生 desktop 桥接包装）。
- **公开接口（`index.ts` barrel）：** `useAttentionNotifications`、`NotificationPermissionPrompt`、
  `selectNotificationChannel`、`NotificationChannel`、`NotificationPermissionState`、
  `ATTENTION_WINDOW_MS`。
- **允许的依赖：** [[submodule-web-store]]（会话状态读取、prompt 状态、`activateWorkspaceFromRoute`
  与 `openSessionSwitcher` 动作）、`ui`（前言对话框）、[[module-contracts]]（`SessionState`／
  `SessionCompletion`／`NativeNotificationBridge` 类型）、`@thinkrail/locales`（通知与前言字符串，
  在构建每条通知时解析）、React、Remix Icon。
- **禁止：** 第二个 attention 真相来源；监听原始 agent 事件；任何 host／`pi` 导入；按 `desktop`
  分支（通道选择是对原生桥接的能力探测）。

总开关（`notificationsEnabled`）位于应用配置中；其 Settings 面板是 `panels/NotificationsSettings`，
不在本模块。多客户端是被接受的：焦点是客户端本地的，因此一个 agent 事件可以在多个已连接客户端中各自
独立地通知。
