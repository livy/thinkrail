# lib —— UI 助手

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

跨组件共享的极小型应用助手。类名合并（`cn`）属于 `@thinkrail/ui/utils`，不属于本模块；带主题感知的
高亮器仍留在应用本地。

## 边界（Boundary）

- **拥有：** `utils.ts` → `cssColorToHex()`（把 CSS 颜色规范化为十六进制 —— 压缩后的 CSS 会给出
  `#fff`／`gray` 这类等价写法，而 Monaco、xterm 等严格消费者会拒绝它们；无法解析时返回 `""`）。
  外加那些不止一个模块需要、且谁都不该重述的原语：**`normalizePath()`**／**`isAbsolutePath()`**／
  **`fileName()`**（来自 pi 工具调用或宿主的路径可能使用任一种分隔符，也可能是相对或绝对路径 ——
  应用中的每个路径谓词都从这些出发，因此 `chat` 的展示助手与 `store` 的 worktree 匹配器共享同一定义；
  **`ancestorPaths()`**（一条 `/` 路径的父文件夹，最外层在前 —— Files 树的文件夹标记与其自动显示展开
  同一集合）与 **`isRecord()`**（针对不可信 JSON 的非数组对象检查）；`@thinkrail/resource-renderers`
  为 `describeResource` 与 `ResourcePreview` 保留自己的 `canonicalWorkspacePath`，因为包无法导入应用的
  `lib`；`panels/pathIdentity.test.ts` 在工作区相对路径上把它钉到 `projectRelativePath`），
  **`viewport.ts`** → `isPhoneViewport()` 加上实时的 `usePhoneViewport()` 投影（粗指针或低于 768px
  的视口即手机类；两个媒体查询都被观察），以及 **`shallowEqualArrays()`**（逐元素 `Object.is` ——
  store 的快照身份守卫与 `ErrorBoundary` 的重置 key 背后的“这真的变了吗？”测试），**`userText()`**
  （一条用户消息的纯文本 —— 由 `chat` 的转录水合／渲染器与 `store` 的实时事件折叠共享，因此“同一条
  消息”在任何地方都是同一个意思；它之所以在此，是因为 `store` 到 `chat/` 的边是纯类型的），
  **`parseSkillInvocation()`** + **`matchesSkillInvocationCommand()`**（Pi 规范的已展开 `<skill>`
  用户消息语法的锚定浏览器侧镜像，由 `chat` 的紧凑渲染器与 `store` 的乐观回显归约共享；畸形／带引号
  的块按失败关闭处理），**`relativeTime(t, ms, now)`**（`just now` / `5m ago` / `2d ago`，针对显式
  `now` 计算，绝不用 `Date.now` —— 由 chat 历史、标签栏的已关闭 chat 与 Changes 范围菜单的提交行共享；
  它在此处是因为 `chat/` 不得从 `panels/` 导入，而这正是曾让它积累了三份私有副本的原因），
  **`platformShortcutLabel()`** + **`hasPlatformModifier()`** + **`platformFamily()`**（快捷方式外框与
  全局处理器所使用的唯一 Apple 与非 Apple 定义，外加用于按操作系统快捷键表的
  `apple`／`windows`／`linux`／`other` 族；全部默认取浏览器报告的平台，但接受显式的平台字符串，因此
  非浏览器调用方与测试永不意外继承宿主运行时的合成 `navigator`；标签可选渲染 Alt 变体 ——
  `⌥⌘N` / `Ctrl+Alt+N` —— 因此快捷方式外框永不自行拼装修饰键字形），以及 **`copyText()`**（写入
  剪贴板并报告是否成功 —— 这是*降级*的唯一位置：不安全上下文（纯 http 远程访问）或被拒绝的权限都没有
  剪贴板，而每个调用方的答案都相同 —— 不做任何显眼的动作，文字保持可见／可选中），**`randomId()`**
  （通过 `getRandomValues` 生成 16 个随机字节，纯 HTTP 远程客户端仍可访问它）与 **`randomUuid()`**
  （同一来源塑成 RFC 4122 v4 UUID，用于宿主按 UUID 校验的 wire id；`crypto.randomUUID` 在此类客户端
  上缺失），**`DOUBLE_CLICK_SETTLE_MS`**（点击→双击的窗口，只在单击会提交某件被双击否认的事情的地方
  使用：单击（或键盘激活）活动预览标签，使双击不被读作两次保留；Changes 评审标签预览；以及经
  `doubleClickPendingMs()` 的 Review 显示），**`whenIdle(task, { timeout, fallbackDelay })`**（在
  `requestIdleCallback` 上带超时地运行任务，或在 API 缺失（Safari 与 macOS WKWebView）时在
  `fallbackDelay` 之后运行，并返回一个取消函数），**`whenPageActive(schedule, task)`**（在页面可见
  且在线之后通过 `schedule` 调度 `task`，并在调度的运行触发时再次检查：期间转隐藏或离线的页面会等待
  `visibilitychange`／`online` 并重新调度；页面读取器可注入以便测试），**`firstClickOnly()`**（包装
  一个浏览行的预览打开，使其只在首次点击或键盘激活、`detail <= 1` 时运行，并跳过双击的第二下）配
  **`doubleClickPendingMs()`**（最后一个此类行上的双击仍可能到达多久；Review 显示会等它过去，以免先
  把该行卸载）。两者都接受可选时钟（默认为 `performance.now()`），因此测试可以钉住窗口而无需睡眠，
  也不会继承另一个测试的点击，**`mcpState.ts`**（MCP 服务器状态词汇 —— `MCP_STATE_KEY`／
  `mcpStateLabel(t, state)`、`mcpStateTone`（已连接服务器报告零工具即警告）、`mcpStatusLabel`／
  `mcpToolCountLabel`（这里的文字助手把调用方的翻译器放在第一个参数，因此渲染输出跟随语言变化），
  以及 `isMcpAttentionState`（需要登录、失败、待批准）／`isMcpActiveState`（connected、starting）
  谓词 —— 由 `store` 的选择器、Settings 与仅 props 的 chat Resources 行共享，后者可能不共同导入任何
  其他东西），以及由 store、shell 与无头 layout 子模块共享的 **`LayoutAttention`** 设备本地覆盖层，
  带有面向不可信元组键映射的自身属性安全 `readLayoutSelection()`／`readLayoutNavigationClock()` 访问器，
  以及与键序无关的 `sameLayoutAttention()` 相等判断，使一次无操作的 attention 写入能在到达 store 之前
  被丢弃。可选的终端召回是工作区本地的历史，不是第二个选中标签权威；布局所有者按当前成员关系校验它。
  另含共享的 Shiki 高亮器，**留在 barrel 之外**，使及早的 `@/lib` 导入保持 shiki-free：`highlighter.ts`
  拥有由 chat 与桌面 Monaco 渲染器共享的精选语法／id／文件关联目录（两者在某种语言首次使用时加载其
  语法），外加 `createChatHighlighter(engine)`：`themes` 的唯一通用 CSS 变量注册，以及每种 chat 语法
  在首次使用时加载。chat 代码在主线程之外高亮：`highlightCode.ts` 解析语言别名（未知与 `mermaid`
  回答 `null` 且不启动 worker），然后向每个页面一个的惰性 `highlighter.worker.ts` 投递，后者使用
  Oniguruma WASM 引擎。每个调用方 key 至多一个请求在途；较新的调用替换等待中的那个，后者解析为
  `null`。如果没有 `Worker`、构造函数抛出、worker 触发 `error`／`messageerror`（宿主的 SPA 兜底会为
  一个被更新删掉的 chunk 返回 HTML），或它无法构建其高亮器（WASM 加载失败），客户端就永久切换到使用
  JS 正则引擎的线程内高亮（在 WebKit、桌面 webview 上，首个 TS 围栏约 1 s）并在那里重跑待定请求；
  这些失败永不应答 `null`。一次失败的语法加载为该次调用应答 `null`（纯文本），并在下一次重试。结果
  存放在有界 LRU 中（256 条目／400 万字符的代码 + HTML；超过其四分之一的单个结果不缓存），它同步应答
  `cachedHighlight()`；一个 key 占一个条目，因此流式围栏不会把它填满。`useHighlightedCode()` 是唯一的
  组件 hook：它只在客户端读取缓存（静态渲染，例如已渲染的 diff，保持纯文本），按请求顺序应用应答，并在
  worker 追上之前把上一次高亮与更新的流式文本按纯文本显示。两个文件都留在 barrel 之外，逐文件导入
  （`@/lib/highlighter`、`@/lib/highlightCode`）。chat 的 markdown 静态导入 `highlightCode`，
  `shell/AppResourceHost` 把目录交给资源渲染器，因此 `@shikijs/core` 在入口图中，而每种语法与 worker
  仍首次使用时才加载；主题身份／调色板永不位于 `lib`。防冲突的浏览器身份组合也在此处：
  **`tupleKey()`** 给独立字符串加长度前缀，**`parseTupleKey()`** 只读取它被请求的命名空间，而
  **`layoutResourceIdentity()`** 为每个前端本地 placement／缓存别名给出一个语义资源 key，因此分隔符与
  稳定的非规范 placement id 无法拆分或混叠身份。
- **公开接口（barrel）：** `cssColorToHex`、`isPhoneViewport`、`usePhoneViewport`、`normalizePath`、
  `isAbsolutePath`、`fileName`、`projectRelativePath`（规范的 worktree 相对 POSIX 身份；折叠根内的
  `.`／`..` 别名，但保留一次试图越界的实例以供宿主拒绝；Windows 盘符根下的包含关系以大小写不敏感方式
  比较路径／根，同时保留候选者自身的大小写）、`shallowEqualArrays`、`userText`、`firstNonBlankLine`
  （第一条去空白后的非空行，文本为空白时为 `undefined` —— 由 chat 的思考标题提取与历史提示行共享）、
  `isShellInert`（一个只由在 POSIX、PowerShell 与 cmd 中都保持字面的字符构成的值，因此它可以被插值
  进一条由人复制并运行命令中）、`parseSkillInvocation`、`matchesSkillInvocationCommand`、
  `relativeTime`、`compactAge`，渲染时的默认名称 `chatName`／`terminalName`（外加语言中立的
  `defaultTerminalTitle`／`defaultTerminalIndex`）／`changesTabName`／`diffTabName`／`turnTimeLabel`／
  `planDocumentName`（存储的标签名不带翻译文本；这些按调用方的翻译器为其措辞），
  **`chatLabel(t, name, firstPrompt)`** → `{ text, source, tooltip? }`，及其投影
  **`chatDisplayName`**（文本）与 **`fallbackChatTitle`**（一个无名 chat 标签的唯一推导：去空白后的
  真实名称以 `source: "name"` 胜出；否则 contracts 的首提示词头部，在 48 个 UTF-16 单元处的词边界处切开
  并加 `…` —— 绝不在代理对内部 —— 以 `source: "prompt"` 给出，完整首提示词作为 `tooltip`；再否则为
  `chatName` 的通用标签，以 `source: "default"`；每个显示 chat 标题的表面都渲染同一个 `chatLabel`，
  因此去空白、`data-title-source` 标记与 tooltip 保持一致，agent 或用户的标题一次性替换所有地方的标签）、
  `platformShortcutLabel`、`hasPlatformModifier`、`platformFamily`、`copyText`、`randomId`、
  `randomUuid`、`MCP_STATE_KEY`、`mcpStateLabel`、`mcpStateTone`、`mcpStatusLabel`、`mcpToolCountLabel`、
  `isMcpAttentionState`、`isMcpActiveState`、`McpStateTone` 类型、`DOUBLE_CLICK_SETTLE_MS`、
  `firstClickOnly`、`doubleClickPendingMs`、`tupleKey`、`parseTupleKey`、`layoutResourceIdentity`、
  `readLayoutSelection`、`readLayoutNavigationClock`、`sameLayoutAttention`、`LayoutAttention` 类型，
  以及 shell 的组合键与面板的纯字母评审快捷键共用的键盘门禁：`MODAL_LAYER_SELECTOR` + `hasLayer`
  （打开的 modal／dialog／alertdialog 拥有键盘）与 `isTextEntryTarget`（input、textarea、select 与
  contenteditable 保留自己的字母）—— 对目标做鸭子类型判断，因此在无 DOM 的测试中也成立。
- **允许的依赖：** React（仅视口订阅 hook）；`@thinkrail/locales`（相对时间、紧凑时长与 MCP 状态标签，
  在调用时按活动语言环境解析）；`@thinkrail/contracts`（仅为规范消息、`McpServerState` 与默认标签名所
  读取的 `GitDiffScope` 提供类型；布局资源身份的输入是本地结构类型）；`shiki`／`@shikijs/*`（仅逐文件
  的 shiki 模块 —— 绝不经由 barrel 可达）。
- **禁止：** 任何应用内部模块 —— 这是一个叶子。
