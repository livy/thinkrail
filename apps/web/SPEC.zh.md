# Web UI 客户端

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

移动优先的 React UI。以静态资源发布，通过 wire 拨号到一个引擎宿主。把 `pi` 的事件流渲染为
以 chat 为中心、多会话的 IDE 外壳。

## 边界（Boundary）

- **拥有：** 浏览器 UI —— 客户端本地导航与工作台状态、传输层客户端、store、面板、响应式
  外壳、品牌 token。
- **公开接口：** 构建出的静态 bundle（`dist/`）—— 一个可部署、拨号到宿主的产物。
- **允许的依赖：** `@thinkrail/contracts`（类型加上它那小的纯运行时：WS 常量、版本、守卫、
  `createQuitConfirmation`；见 [[module-contracts]]）、`@thinkrail/locales`（UI 字符串 +
  共享 i18next 实例；见 [[module-locales]]）、`@thinkrail/ui/*`、`@thinkrail/extension-api/web`
  与 `@thinkrail/ext-*/web`；React / Zustand / Vite 等。
- **部署义务：** 一个构建出的客户端服务每个启动器与未来的部署。端点选择属于传输层引导；
  面板、store 与功能流程绝不在 `cli`、`desktop` 或某个部署名上分支。
- **禁止：** 导入 `server` / `shared` / 任何 `pi` 包（值或类型）。靠仅类型导入 +
  `verbatimModuleSyntax` 保持干净（一次 `dist/` 构建中不出现 provider SDK / `node:fs`）。

## 内部模块（Internal modules）

每个都是有界的子模块；`navigation`/`transport`/`store`/`updates`/`prompt`/`lib`/`extensions`
暴露一个 `index.ts` **barrel**（它们唯一的公开接口）。`panels`/`chat` **按设计逐文件导入** ——
给它们加 barrel 会把懒加载的 Monaco/shiki/xterm chunk 拉进急切 bundle，并破坏 shadcn 的
按原语惯例；它们的边界由惯例 + 规格维持。同级边在这里，不在叶子里。

| 模块 | 拥有 | barrel | 规格 |
| --- | --- | --- | --- |
| `navigation` | 相对后端的定位模型 + 片段驱动/经过校验的恢复 | 是 | [navigation/SPEC.md](src/navigation/SPEC.md) |
| `transport` | WS 客户端 + 其单例/store 接线 | 是 | [transport/SPEC.md](src/transport/SPEC.md) |
| `store` | Zustand：领域投影、一个本地工作台帧、按工作区的视图/注意、chat 运行时 | 是 | [store/SPEC.md](src/store/SPEC.md) |
| `panels` | 与布局无关、由 store 驱动的功能视图 | 否 | [panels/SPEC.md](src/panels/SPEC.md) |
| `chat` | pi 会话 UI 原语：内容块渲染器 + 工具渲染器注册表 | 否 | [chat/SPEC.md](src/chat/SPEC.md) |
| `prompt` | 与生命周期中立的斜杠补全 + prompt 模板槽位编辑 | 是 | [prompt/SPEC.md](src/prompt/SPEC.md) |
| `auth` | 应用内 provider 登录：展示性 OAuth 对话框 + 其客户端状态 reducer | 是 | [auth/SPEC.md](src/auth/SPEC.md) |
| `shell` | 响应式组合 + 前端本地的工作台所有权（有界的 `layout/` 与 `layoutState/` 子模块） | 否 | [shell/SPEC.md](src/shell/SPEC.md) |
| `updates` | 可选的原生/宿主更新 shell 钩子与由 props 驱动的控件 | 是 | [updates/SPEC.md](src/updates/SPEC.md) |
| `notifications` | 应用外注意通知：绿点观察者、队列/窗口引擎、浏览器 Notifications 包装、权限前言 | 是 | [notifications/SPEC.md](src/notifications/SPEC.md) |
| `components` | 依赖轻量的 app 原语：错误隔离、状态图标、自定义图标、安静滚动框架、预加载的拆分 chunk | 否 | [components/SPEC.md](src/components/SPEC.md) |
| `extensions` | 把 web 扩展描述符按顺序组合进 chat 注册表 | 是 | [extensions/SPEC.md](src/extensions/SPEC.md) |
| `themes` | 经过校验的单文件清单、内置目录 + 原子 token 应用 | 是 | [themes/SPEC.md](src/themes/SPEC.md) |
| `i18n` | 当前语言：同步的偏好 + 浏览器语言 + 内置/本地目录 → 共享 i18next 实例、`<html lang>`、桌面菜单标签 | 是 | [i18n/SPEC.md](src/i18n/SPEC.md) |
| `lib` | 共享路径/数组原语 + 高亮 | 是 | [lib/SPEC.md](src/lib/SPEC.md) |

没有自己规格的叶子工具：`constants/`（品牌 —— 产品名、存储/事件前缀，以及唯一的
`BRAND_MARK_SEGMENTS` 字母组合区域，拼接成 shell logo 填充、并被 components 的工作徽标裁剪的
`BRAND_MARK_PATH`）、`clientPreferences.ts`（对可选原生稳定字符串适配器的功能中立访问），
以及 `styles/` —— 它持有三个设计系统**来源**（`typography.json`、`colors.json`、
`spacing.json`）、其生成的 CSS 与结构 token 契约；按主题的调色板属于 `themes`。每个系统都在
其来源旁有规格：[TYPOGRAPHY.md](src/styles/TYPOGRAPHY.md)、[COLOR.md](src/styles/COLOR.md)
与 [SPACING.md](src/styles/SPACING.md)；建立在其上的应用内通知与 toast 外观见
[NOTIFICATIONS.md](src/styles/NOTIFICATIONS.md)。在 `src/` 之外，**[`scripts/`](scripts/SPEC.md)**
是构建期生成器模块 —— 它在 Bun 下运行，从不发布，把那三个 JSON 来源变成
`styles/generated/`（并把随附的 provider 标记变成 `chat/generated/`）。`index.html` 命名
产品，并链接从与 shell logo 相同的 ThinkRail 图稿派生的本地、纯符号 SVG favicon（足够紧凑
以适配浏览器标签尺寸与明暗浏览器 chrome）。`main.tsx` 是入口/组合根 —— 它同步构建内置主题
目录、在 React 之前解析并应用带版本的首绘主题偏好提示（含当前系统配色方案）、初始化
transport 及其客户端错误上报（window 监听器 + error-boundary 上报接缝）、客户端本地偏好
接缝（chat 呈现、隐藏模型、项目展开）与导航，然后把 `<Shell />` 包进
`components/ErrorBoundary` 作为最后兜底边界（逃出每个区域的崩溃显示重载屏，而不是空白根）
以及应用唯一的 `TooltipProvider`。

**React 是在带有上游修复 #34803 的发行版上、一个精确且互相匹配的 `react` + `react-dom` pin。**
React 19.2 的开发版 Performance Tracks 保留了每条 `performance.measure()` 记录；Pi 速率的
chat 渲染让一个 Vite 标签页增长到超过 10 GB，而 JS 堆保持平稳，随后 Chrome 杀掉渲染器，
Vite 把已死的 socket 报成 `EPIPE` / `ECONNRESET`（React #34770）。ThinkRail 曾运行一个带
React 自有 `clearMeasures` 修复的 19.3 canary，直到稳定版包含它，并刻意不携带第二套测量
清理策略。根包管理器的 overrides 通过 catalog 条目解析两个运行时包，而 `check:deps` 读取
`bun.lock` 并拒绝任何第二个 `react` 或 `react-dom` 版本，因此每次 React 迁移都同时升级
两者。每次 React 迁移还要重做挂载 chat 的测量探针：开发构建
（`NODE_ENV=development vite build --mode development`）把重放的文本增量
（`e2e/perf/chatReplay`）流进已挂载的 chat，且必须保持
`performance.getEntriesByType("measure")` 平稳。在 19.2.0 上它每条 12 KB 消息增长约 1,500 条
记录；在修复后的发行版上它保持为零。

**发布 source map 仅构建期存在，永不发布。** 设置了 `THINKRAIL_SOURCEMAPS_RELEASE`（带凭据的
发布构建，[[module-ci-release]]）时，`vite.config.ts` 输出隐藏的 map，而
`sourceMapUpload.ts` 插件对 `dist` 运行 `posthog-cli sourcemap process --delete-after`，若残留
任何 `.map` 就失败；未设置时构建不变。`posthogCli.ts` 是仓库中运行 posthog-cli 的唯一方式
（根 `posthog-cli` 脚本把它暴露给桌面的 `preBuild`）：它在运行前按钉住的一组 SHA-256 值校验
该二进制（[[module-ci-release]]）。注入的 chunk-id 注册表就是 `transport` 读取用于 UI 错误
报告的东西。

**React Compiler 对所有 `apps/web` bundle 开启**（`vite.config.ts`：`@rolldown/plugin-babel`
在 dev 与 build 中运行 `@vitejs/plugin-react` 的 `reactCompilerPreset()`）。三个
devDependencies 只存在于本清单中，而 `@babel/core` 7.x 的 pin 是关键载荷：
`babel-plugin-react-compiler` 1.0.0 在 Babel 8 下错误降级解构默认值，并静默把这些函数从
编译中丢弃。bailout 从不使构建失败 —— 编译器无法证明安全的函数会被整体跳过，而不是错编译
—— 因此要检查它确实运行了：生产构建的 `dist/assets/*.js` 中带有成百上千处
`react.memo_cache_sentinel`（React 自身运行时占三处），而 CI 的 `bun run check:compiler` 在
某个函数的编译状态偏离 `scripts/compiler-census.baseline.txt` 时失败。`bun test` 不经 Vite
转译，因此只有浏览器 E2E 套件会执行编译输出。

外壳、工作台分组视图、`ChatView`、`Composer` 与 `useChatScroll` 之所以能编译，是因为它们
遵守这些约定：

- 最新值 ref（`xRef.current = value`）在 `useInsertionEffect` 中写入，绝不在 render 中写，
  且 render 从不读 `ref.current`。插入副作用在任何布局副作用之前为整棵树运行，因此子组件
  的布局副作用回调到父组件（`react-resizable-panels` 的 `onLayout`、`useChatScroll` 中
  Virtuoso 的几何读取）时看到的是当前值。该副作用只赋值 ref：无状态、无 DOM。
- 一个 hook 把 ref 与 render 值一起以元组形式返回（`useElementSize`），或由调用方解构
  （`useCollapsibleRegion`、dnd-kit 结果）：从携带 ref 的对象上读取 render 值算作 ref 读取。
- 作为 prop 传递的 ref 命名为 `*Ref`（`selectionEpochRef`），以便处理器可变它。
- 读取 ref 的闭包只能通过 hook（`useReadingBandController`）到达 `useState` 初始化器，且
  render 从不把 ref 交给普通辅助函数；该辅助函数变成一个拥有它的 hook（`useSideResizeBinder`）。
- 默认参数从不读取成员表达式（`caret = text.length`）；在函数体里解析它。
- 闭包从不向它捕获的变量做 `++`/`--`；写成 `attempts += 1`。
- 组件或 hook 从不在 `try` 内使用 `try/finally` 或 `throw`。把函数体移进一个本地 async
  闭包，以 `run().catch(onError).finally(cleanup)` 链接；同步守卫移到模块级辅助函数
  （`setLayoutFlagged`）。`try/catch` 内的条件、逻辑或可选链值（`a ? b : c`、`a ?? b`、
  `a?.b`）移到 `try` 之上，或在其分支 await 时移进该闭包。
- `useMemo`/`useCallback` 列出它读取的每个依赖，否则编译器无法保留它。
- 回调不读取的 `useMemo` 依赖会被去掉，因此 memo 不能因它忽略的 key 而重置
  （`useMemo(() => new Map(), [key])` 每次挂载构建一个 map）。改为通过以该值作 key 的状态
  来重置（`ChatView` 的行高估算缓存）。
- Render 从不读取编译器看不出变化的值 —— `matchMedia`、storage、`Date.now`、模块单例。
  它通过 `useSyncExternalStore` 或状态到达：`AppearanceSettings` 用
  `onSystemAppearanceChange` 订阅，相对时间标签从 `components/useNow` 取 `now`。

已知 bailout 包含一条热路径：`useVirtualRows` 在每次 `ChatView` render（每个流式增量）时
运行。共享 hook 与资源界面也有 bailout；chat/shell 热路径的编译并不意味着覆盖了每个
文件/diff 渲染器。

- render 中的 ref 访问：`useVirtualRows`（在 render 期间调整状态时读取可见锚点 ref；用状态
  会在每次滚动时多付一次 render）、`useWorkspaceRead`、`useBranchStatus`、`useBranchList`、
  `useTemplateCommandPicker`、`usePendingSelection`、`MonacoEditor`、`AskUserQuestionCard`、
  `useScrollViewState`、Pierre diff/file 的 `useThreadAnnotations`、`PierreDiffSurface`、
  `PierreFileSurface`、图片 diff 的 `ImageContent`、`ImageView`、`PdfView`、`usePdfDocument`。
- 手工 memo 依赖无法保留：`CsvDiff`、`JsonDiff`、`NotebookDiffSurface`、`PdfDiff`。
- 其他：`useLiveTabContent`（`??=`）、`HistoryOverlay` 的 `Highlight`（可变闭包计数器）。

新的 bailout 即为回归，除非它加入此列表。

**渲染成本在单独的 profiling 构建上测量，绝不在发布构建上。**
`bun run --cwd apps/web build:profile`（`vite build --mode profile`）输出 `dist-profile/`：
相同编译后的生产代码，但 `react-dom/client` 别名到 `react-dom/profiling` 且关闭压缩，因此
组件名得以保留以便归因。源码不携带 `<Profiler>` 包装、标志或收集器：按组件的成本由
harness 安装的 React DevTools hook 读取（`e2e/perf`，见 [[module-browser-e2e]]）。包装需要
在产品代码里加构建标志，且只给出区域总计；hook 能看到每个组件而无需任何源码改动。因此
`dist/` 中没有 profiler 计时器（`dist/assets` 中不存在 `actualStartTime`）。

### 依赖图（Dependency graph）

- `navigation` → `store`、`transport`、`contracts`（仅类型）；两个依赖都不导入它，集成由
  `main.tsx` 初始化
- `shell` → 子模块 `shell/layout` + `shell/layoutState`、`@thinkrail/resource-renderers`
  （`AppResourceHost` 注册内置渲染器并提供一个模块级 `ResourceHost` —— `chat/Markdown`、
  基于 `lib/highlighter` + `themes` 的代码宿主、`panels/resourcePane` 的字节 URL、
  `panels/openTabs` 导航（`openFileInTab` / `openFileComparisonInTab`）、`fs.readFile` +
  fs 变化订阅，以及会话感知的 `useFileReview`（作为 `FileReview` render-prop）+
  `SendReviewButton` —— 外加经 `ResourcePreferencesProvider` 的行宽偏好与经
  `ResourceEnvironmentProvider` 的手机类别；`main.tsx` 在其内包裹 `Shell`）、`updates`
  （一个可选能力 hook + 由 props 驱动的 Settings 内容与持久状态提示）、`notifications`
  （一次性挂载注意观察者 + 权限前言）、`panels`、`chat`（仅 app 集成的 render/水合）、
  `store`、`transport`（领域水合 + 端点身份）、`contracts`（类型 + 共享的
  `createQuitConfirmation` 规则）、`ui`、`components`（每个已挂载区域外裹
  `ErrorBoundary` + shell 自有工具体外裹 `QuietScrollArea` + 为 `ChatView` chunk 使用
  `preloadedLazy`）、`constants`、`lib`（平台快捷键语义）、`themes`（目录/媒体解析与原子
  主题应用的唯一所有者，由已水合的 store 偏好或水合前提示驱动）
- `shell/layout` → `contracts`（仅 `LayoutPreset` + `GitDiffScope` 类型）、`lib`
  （注意/id 原语）、`ui`（含 `@thinkrail/ui/resizable`）以及 React / `@dnd-kit/core`；
  `shell/layoutState` → `shell/layout`、`store`、`transport`（浏览器端点身份 + 错误规范化）、
  `clientPreferences`（原生稳定持久化）、`contracts`（仅 `LayoutPreset` 类型）、`lib` 以及
  React。父组件注入 store 状态与功能渲染器，因此纯布局子模块没有功能模块运行时边
- `updates` → `contracts`（原生桥 + 宿主通知类型）、`store`（宿主通知、toast）、`ui`、
  React 与 Remix Icon；原生快照仍留在 shell 本地
- `notifications` → `store`（会话状态读取、prompt + `activateWorkspaceFromRoute` /
  `openSessionSwitcher` 动作）、`ui`（权限前言对话框）、`contracts`（仅类型 ——
  `SessionState`/`SessionCompletion`/`NativeNotificationBridge`）、React 与 Remix Icon；
  主开关 Settings 面板位于 `panels`，由 `shell` 挂载观察者/前言。它通过能力选择的通道发出：
  页面 Notifications API，或在可选 `__THINKRAIL_NATIVE_NOTIFICATIONS__` 桥存在时的原生桌面
  通道（仅能力检测，无 `desktop` 分支）。没有 Service Worker：关闭的 app 没有活客户端可通知
- `panels` → `@thinkrail/resource-renderers`（注册表、review 契约、`ReviewThreadCard`、
  草稿上下文、`createAskAgentRequest`）、`store`、`transport`、`ui`、`components`
  （功能体外裹 `ErrorBoundary` + 面板自有列表/xterm 的安静滚动界面）、`lib`、`contracts`、
  `constants`（`WelcomePanel` 的字标）、`prompt`（`NewWorkspaceDialog` 消费共享的
  斜杠/模板行为）、`chat`（`NewWorkspaceDialog` 急切复用
  `chat/ModelEffortPicker`+`useModelCatalog`+`useModelPreferences`，`ModelsSettings` 复用
  较旧的 `ModelSelector`+`ThinkingSelector`，`VisibleModelsSettings` 复用
  `chat/modelPicker` 的 `formatContext` + `chat/ProviderGlyph`，`ProvidersSettings` 复用
  `chat/modelPicker` 的连接类型术语 —— 全部不依赖 shiki，因此急切导入保持拆分安全；
  `TemplatesSettings` 复用 `chat/TemplateEditorDialog` 实现其 New/Edit 流程 —— 见
  `panels/SPEC.md` 的 `TemplatesSettings` 段落；`ProjectTrustNotice` 与
  `NewWorkspaceDialog` 与 Skills 对话框的信任横幅共享 `chat/projectTrust` +
  `chat/useProjectTrustSummary`）、`auth`（`useProviderAuth` —— 由 `ProvidersSettings`、
  `ConnectProviderDialog` 与 onboarding 向导共享 —— 以及 `McpSettings` 挂载
  `auth/LoginDialog`）、`notifications`（`NotificationsSettings` 在启用时经 barrel 请求
  浏览器权限）、`themes`（`AppearanceSettings` 消费活动目录；代码界面消费通用主题
  变量/语法映射）、`i18n`（`LanguageSettings` 渲染其语言派生与 `useSystemLanguages`），
  以及 `react-virtuoso`（懒加载的 Changes review 列表）；渲染器引擎是
  [[module-resource-renderers]] 自有的依赖
- `chat` → `contracts`（pi 消息类型，**仅类型**）、`ui`、`extension-api/web`、
  `@thinkrail/resource-renderers`（`ChatTranscript` 向它的行提供 chat 的 `ResourceScope`）、
  `ext-visualize/web`、`extensions`（仅注册）、`components`（`useNow`，以及流页脚的
  `RunningIcon` 徽标）、`prompt`（共享的斜杠/模板行为）、`lib`、`clientPreferences`；
  `store` + `transport`（**仅 app 集成文件** —— 渲染器保持无 store；当前集合见
  `chat/SPEC.md`）
- `prompt` → `contracts`（仅斜杠/模板类型）、`@thinkrail/ui/utils`（`cn`）、
  `@thinkrail/ui/button`（模板槽位提示是一个共享 `Button`）以及 React；它没有生命周期
  集成依赖
- `extensions` → `ext-*/web`、`extension-api/web`、`chat/toolRegistry`（仅注册；不导入
  ChatView）
- `auth` → `ui`、`lib`（`copyText`；对话框无 store/transport —— 面板集成它；状态类型无需
  导入）
- `store` → `transport`（**仅类型** —— `ConnectionStatus`）、`chat`（**仅类型** ——
  `ChatTurn`/`ToolResultState`）、`auth`（**仅类型** —— `LoginState`；`foldLoginFrame`
  reducer 位于 `store`，与 `reduceExtUi` 一样）、`contracts`（领域 + 自定义预设类型，
  绝不是当前布局的 DTO）、`lib`（共享路径/数组原语 —— 一个叶子，因此无环），以及
  `shell/layout`（对 web 本地帧/视图状态**仅类型**）
- `transport` → `contracts`、`store`（欢迎路由；`store → transport` 的反向边仅类型，因此
  运行时图无环）、`lib`（纯 HTTP 安全的随机页面身份）
- `components` → `lib`（`ErrorBoundary` 的 `shallowEqualArrays` 用于其重置 key）与
  `constants`（`RunningIcon` 绘制共享品牌字母组合）—— 两者都是叶子，因此任何区域仍可包裹
  它；展示性控件消费 `ui`
- `lib` → `themes`（懒加载高亮器使用唯一的通用 CSS 变量 Shiki 注册）、React（仅手机视口
  hook）与 `contracts`（仅类型，例如 MCP 状态词汇的 `McpServerState`）
- `themes` → `constants`（品牌存储前缀限定首绘提示与用户主题缓存的适用范围）、
  `clientPreferences`（原生稳定提示存储）；`transport` 水合 store 的用户主题快照，
  `shell` 的主题副作用安装它
- `i18n` → `store`、`contracts`（类型）、`@thinkrail/locales`；`transport` 水合 store 的
  本地语言快照，`main.tsx` 的 `App` 一次性挂载 `useLocaleSync`，`panels` 从其纯派生渲染
  Language 页
- **任何模块都可 → `@thinkrail/locales/react`**（`useT`、`<T>`、`currentLocale`）以获取其
  UI 文本 —— 除 `lib` 之外唯一的横切边；面向用户的文本从不是字面量（[[architecture]]
  Decision 23）
- 叶子（`clientPreferences`、`constants`、`utils`、`styles`）→ 无内部依赖

规则：面板从不横向导入另一个面板；没有任何东西导入 `shell`（它是组合根）。

渲染器引擎（`@pierre/diffs`、`jsondiffpatch`、`pdfjs-dist`、`@shikijs/monaco`、
`monaco-editor`）是 [[module-resource-renderers]] 的依赖；应用为自己的面板保留
`react-virtuoso`（由 catalog 钉住，与该包共享）。分包契约对 app 构建仍然有效。
`pdfjs-dist` 及其 `pdf.worker.min.mjs` worker URL 只从懒加载 PDF 渲染器 chunk 触达；两者
都不属于入口图。Pierre、其 Shiki 语言/主题图与 worker 入口，以及 Monaco、其精选 Shiki
语法与适配器，都保留在 resource-loader 动态导入之后。只有懒加载的 Pierre 渲染器模块挂载
provider，且每个已挂载界面都会获取 Pierre 的模块单例 worker 池；因此普通工作区既不加载
Pierre 也不初始化其池。渲染器元数据与加载器是唯一的急切边。生产构建必须保留彼此不同的
Pierre diff、Pierre file、worker 池与 Monaco chunk，且它们的实现代码都不在入口 chunk 中。
Monaco 留在入口图之外，但并非只在需要时加载：在桌面上，`shell` 在工作台挂载后按页空闲
预加载代码视图一次，条件是页面可见且在线，而 Files 行在聚焦时以及悬停 150 ms 后预热它。
只有以 `capabilities.preload` 选择性加入的渲染器（代码视图）才提前加载，因此悬停 PDF 或
notebook 行不加载任何东西。手机类别视口与 `saveData` 连接从不预加载。Vite 把每个 worker
输出为 ES 模块（`worker.format: "es"`），因此 worker 的 `import()` 调用会被拆成按需 chunk
而不是内联；每个目标 webview（Chromium、WebKit/Safari 17+、WebView2）都运行模块 worker。
Safari/WebKit 17 是浏览器下限：发布的 webview（macOS 12+ 系统 WebKit、Ubuntu 24.04
WebKitGTK、WebView2）都满足它，因此 web 代码可以使用 baseline-17 API，如 `URL.canParse`。

模块集合：`transport` / `store` / 有品牌的 `shell` 及其无头的 `shell/layout` 子模块；
与布局无关的 Project/File/Specs/Changes/Review 渲染器；注册表分派的资源体与懒加载 xterm
终端体；共享的 `prompt` 行为模块；以及 `chat` 模块（`ChatView`、内容块渲染器、工具注册表
与完整 Composer）。工作台在这些体周围拥有中部以及左/右/底部的辅助条带/组，而不是拥有
面板本身。

## 样式与主题（Styling & theming）

- **Tailwind v4 工具类，映射到设计 token**（`src/index.css` 的 `@theme inline`）。组件用
  工具类表达颜色、间距、边框与布局（`bg-container-header-bg`、`text-primary`、
  `border-border-default`、`px-12`），用**生成的语义排版类**表达字体（`tr-text-ui`、
  `tr-title-dialog`、`tr-code-text` 等）—— **绝不使用内联 `style` 对象，除非是渲染器实测
  的几何，且绝不使用原始十六进制值。** 有界的几何例外覆盖诸如内禀媒体边界、规范化覆盖层、
  缩放、滑动位置、portal 放置以及虚拟列表几何（加载区段的预留高度、实测的尾部占位）等值；
  颜色、间距与控件皮肤仍是 token 工具类。响应式（`md:` …）与状态（`hover:` /
  `focus-visible:`）来自 Tailwind（内联样式无法表达它们，而响应式外壳需要它们）。
- **外壳几何位于 `index.css`，宿主几何以 CSS 自定义属性到达。** 在生成的颜色/间距层旁边，
  `index.css` 映射 shell 的结构行（`--spacing-panel-header-row`、来自 `tokens.css` 的
  `--spacing-topbar-row`）与两个宿主发布的窗口外壳内边距
  （`--spacing-window-chrome-inset-left|right: var(--window-chrome-inset-*, 0px)`），而宿主
  还发布 `--window-chrome-drag-region`。它声明仅有的两个非排版手写工具类 `window-drag` /
  `window-no-drag`（`-webkit-app-region` + `app-region`）；`window-drag` 除非宿主选择加入
  否则无效。移除了自有标题按钮的宿主还会安装来自 `contracts` 的仅类型
  `NativeWindowControlsBridge`，shell 把它变成 HTML 窗口控件。原生宿主可以在 `<html>` 上
  设置这些属性；应用从不检测宿主，它读取属性及其 `0px` 回退（[[submodule-web-shell]] 拥有
  消费者，[[module-desktop]] 拥有发布者）。
- **颜色与排版系统属于本应用，不属于 monorepo。** `apps/website` 有意保留自己硬编码的
  样式表 —— 一个无主题化的静态页面用不上 token 层，而跨 app 触达只会无谓地耦合它们。
- **颜色工具类命名语义角色，绝不命名调色板条目** —— `bg-container-elevated-bg`，而非
  `bg-[var(--elevated)]`；而着色是四步 alpha 尺度上的 token，不是 `/40` 修饰符。
  `src/styles/COLOR.md` 是系统；`src/styles/colorUsage.test.ts` 是采纳守卫（Tailwind 会
  静默丢弃未知工具类，因此未发布的 token 渲染出来什么都没有）。
- **常规圆角与节奏间距值来自其尺度，绝不是原始像素长度** —— `rounded-md` 与 `p-8` /
  `gap-12`，而非 `rounded-[7px]` 或 `py-[3px]`。圆角是项目 T 恤家族
  （`--radius-xs/sm/md/lg` —— 一个上限 8px 的小型原语几何：`sm`（4px）是默认圆角，`md`
  （6px）是嵌套 4px 子元素表面的外圆角，`lg`（8px）是大型独立抬升表面（对话框、用户消息
  气泡）的例外）。间距是**一个规范数值尺度** ——
  `0 / 2 / 4 / 6 / 8 / 12 / 16 / 24 / 32 / 40 / 64` —— 其中步长名*就是*其像素值，因此
  `p-8` / `gap-12` / `py-4` 解析为正好那么多
  像素；它由单一 JSON 来源生成（`src/styles/spacing.json` →
  `src/styles/generated/spacing.css`），因此每个规范长度在 JSON 中只写一次，而不在调用点
  重复声明。多个界面必须一起移动的值是一个命名**布局角色**，为某个步长起别名
  （`p-panel-inset`、`h-panel-row`），绝不是重复的字面量。`src/styles/SPACING.md`
  （`web-spacing`）是权威系统；`src/styles/spacingUsage.test.ts` 是该采纳守卫，它的存在
  是因为这类漂移是**不可见的**：不像颜色工具类，任意长度总能渲染，因此偏离尺度的值在评审
  中看起来正确，并通过其他每个门禁。根本不是尺度步长的长度 —— `max-w-[78ch]`、
  `w-[320px]`、`max-h-[40vh]`、实测的 `pl-[calc(…)]` 缩进 —— 仍然允许；它们是布局约束，
  不是节奏。
- **图标是 `@remixicon/react`（Remix Icon）字形，按 UI *上下文*而非位置在 Tailwind
  `size-*` 尺度上定尺寸** —— 按名导入，无 `<Icon>` 包装，无 `size=` prop，也绝不为了调整
  字形尺寸而添加容器。Remix 字形是 `fill="currentColor"`，因此语义 `text-*` 工具类为它们
  上色。**样式由状态驱动：`Line`（描边）变体是默认；`Fill`（实心）变体标记*活动/选中*
  项** —— 选中的项目与活动工作区行（`ProjectTree`）、活动/主规格节点（`SpecsPanel`）、
  活动的文件/目录行（`TreeRow`）与活动编辑器标签（`Workbench`）切换为 `Ri…Fill`；其他
  一切（按钮、折叠箭头、状态、composer、chat、菜单）保持 `Ri…Line`。没有 Line/Fill 对的
  图标（`RiParagraph`/`RiDraggable`/`RiLinkM`/`RiListCheck3`）在两种状态下都渲染其单一的
  样式。**Remix 缺少的项目自定义字形**作为 SVG 位于 `public/custom-icons/`，并通过
  `CustomIcon` 原语（`components/CustomIcon`）渲染 —— 一个 `currentColor` 的 CSS
  `mask-image` span，因此它们像 Remix 字形一样随主题变化并按状态切换 Line/Fill。
  **Changes** 工具使用自定义 `file-diff` 字形；**Review** 使用
  `RiDiscussLine`/`RiDiscussFill`。三档：`size-12`（12px）用于 **chat 内容**指示器
  （工具活动、计划/待办状态、展开/折叠 chat 详情 —— 从属于 chat 文本）；`size-14`
  （14px）用于**紧凑界面/导航外壳**（左/右面板、面板与工具栏头部、标签、移动切换栏、
  菜单项）；`size-16`（16px）用于**每个展开/折叠箭头**（`ChevronDown/Right/Left/Up`），
  无论哪个档次在所有上下文中都是 `size-16` —— 按钮内部的箭头除外。**按钮内部的图标不按
  上下文分档**：`Button` / `IconButton`（`@thinkrail/ui/button`）按按钮尺寸（S 14、M 16、
  L 20）为其图标（含箭头）定尺寸，调用点传入图标组件，绝不传 `size-*` 类。**图标↔文本
  间距在所有图标+标签行（导航行、菜单/命令项、标签）都是 `gap-4`（4px）**。**两行**行
  （例如分支与名称不同的工作区）把图标**顶部对齐**到第一行（`items-start` + `mt-2`），
  使字形挂在其标题上，而单行行保持垂直居中（`items-center`，不做微调）。菜单项图标集中
  在 `packages/ui/menu-styles.ts` 一次（`menuItemClass` 的 `[&_svg]:size-14` + `gap-4`）。
- **`src/themes` 是主题契约与目录；`src/styles/tokens.css` 是结构性的。** 内置主题是一份
  严格完整的 `*.theme.json` 清单：外观/对比度元数据 + 语义 UI 颜色 + 全部 16 个 ANSI 颜色
  + 语义语法调色板。选中文本前景覆盖是唯一可为空的颜色槽（`null` 保留消费者默认）。构建期
  glob 在引导时校验该集合（我们的文件 —— 坏文件会大声失败），因此新增主题只改那一个文件
  —— 绝不改 contracts、标签映射、CSS 选择器、编辑器导入、测试或规格 —— 并在重建后出现。
  用户在运行时以同样方式添加主题：同一份清单放在 `~/.thinkrail/themes`，由
  [[submodule-server-themes]] 提供并在此校验，不可用的文件被报告而非致命。清单是自包含的
  （无继承），只含规范颜色数据，且不能改变布局/排版/动效或注入 CSS/代码。引擎派生重复的
  着色/效果，并在改变 `[data-theme]` 之前原子地写入映射后的自定义属性；`@theme inline` 让
  每个工具类都指向活变量，因此组件保持不变。`tokens.css` 保留间距基准、圆角、动效与通用
  派生公式 —— **完全没有排版**（既无值，也无指向某值的别名；`--font` / `--font-mono` /
  `--font-accent` / `--font-mono-size` / `--line-height` 别名已去掉，因为同一个值的第二个
  名字正是漂移的来源），也没有命名的主题块。
- 主题偏好是**服务端同步**的：`AppConfig.theme` 仍是宿主拥有的不透明固定选择，而封闭的
  `themeMode` 与可选的不透明 `systemThemePair` 选择加入按客户端的系统匹配。每个客户端对
  自身的 `prefers-color-scheme` 解析该对；操作系统变化是本地 DOM 效果，绝非配置写入，因此
  两个客户端可能同时渲染不同外观。固定 Dark 仍是新配置与旧配置的默认值。`settings.update`
  经 `settings.changed` 收敛；一次仅主题的旧式变更会退出系统模式。不可用的系统槽在所需
  外观内确定性地回退，而不破坏性地重写所求值。带版本的 React 前提示缓存的是偏好，而不是
  最后生效的 id，在存在注入的原生稳定偏好适配器时使用它，否则使用 localStorage。主题随
  应用发布：只能经源码 PR 添加，运行时注册/扩展加载被刻意设计为不支持。
- **每个代码界面都与目录无关。** xterm 与 Monaco 在原子 `[data-theme]` 信号之后从通用
  变量重建，包括可选的选中文本前景。Monaco 从清单的外观/对比度元数据选择 `vs`/`vs-dark`
  或相应的高对比度基底，绝不按主题 id。Shiki 使用一份代码自有的语义 CSS 变量 TextMate
  映射：chat 消费其实时引用，而 Monaco 把同一映射解析为十六进制供 `@shikijs/monaco` 使用；
  Pierre 独立的 `thinkrail` 注册也发出实时变量引用。都不需要按主题的导入或选择器，只有
  严格模式下的 Monaco 在切换后重新解析。Pierre 自有的继承 diff 变量把语义反馈颜色与 canvas
  颜色带入其 Shadow DOM。Mermaid 从同一批变量重新派生。严格消费者的读取仍经
  `lib.cssColorToHex`。数据驱动测试对每个被发现清单强制既有的对比度下限（body/muted ≥
  4.5:1 与 hint ≥ 3:1，在主声明的表面上）。
- 圆角 token **就是** Tailwind 的圆角命名空间：`tokens.css` 在一个先重置 Tailwind 默认
  尺度（`--radius-*: initial`）的 `@theme static` 块中声明 `--radius-xs/sm/md/lg`。于是
  `rounded-<step>` 及其方向变体解析为活 token，未声明的步长如 `rounded-xl` 永不生成，而
  `static` 保持每个变量都为手写 CSS 发出。这取代了早先把冲突 token 写成任意值
  （`rounded-[var(--radius-md)]`）的规则：冲突在源头被移除，而不是在约 300 个调用点绕过，
  且守卫拒绝任何任意圆角值。
- **排版来自 `styles/typography.json`，经生成的语义类 —— 别无其他。**
  `styles/generated/typography.css`（已提交，由 `bun run typography:generate` 重新生成，
  由 `typography:check` 做漂移门禁）发出原语 `--tr-*` 自定义属性、`<body>` 基础与每个语义
  样式一个类：`tr-brand-*`、`tr-heading-*`、`tr-title-*`、`tr-text-*`、`tr-code-*`，外加
  每个 markdown 表面一个 `tr-prose-<surface>` 系统（`tr-prose-chat`、`tr-prose-doc` ——
  相同元素集、不同尺度，因为 chat 气泡与渲染文档需要不同的标题阶梯）。调用点只命名一个类
  并加上自己的**颜色**；`italic` 与 `leading-*` 是唯一可以覆盖语义样式的 Tailwind 工具类
  （这些类在 `@layer components` 中发出，因此工具类赢得它命名的那单个属性，而
  `@layer base` 中的 `<body>` 基础会输给它们全部）。
- **不得引入手写排版工具类。** 任何新 `@utility` 规则不得设置 `font-family`、
  `font-size`、`font-weight`、`line-height`、`letter-spacing` 或 `text-transform`，任何
  组件不得组合它们 —— 而是向 `typography.json` 添加语义样式。此前的 `text-mono` /
  `text-base-mono` / `text-brand` / `text-eyebrow` 工具类与 `--text-xs|sm|base|md|lg` 映射
  全部去掉；生成的类取代每一个。`index.css` 只映射两个排版名 —— `--font-sans` /
  `--font-mono`，因此 Tailwind 针对 `html` 与 `code`/`pre` 的 *preflight* 默认值也来自
  JSON；由此启用的 `font-sans` / `font-mono` 工具类在调用点被禁用。
  `styles/typographyUsage.test.ts` 强制执行这一切，包括组件命名的 `tr-` 类确实是生成器
  实际发出的类（未知类被 Tailwind 静默丢弃，于是元素无样式渲染而类列表却声称不然）。
- **只有在已文档化的第三方集成中才能命名原语字体家族。** Monaco
  （`resource-renderers/support/monacoSetup.ts`）读取代码家族、`s11` 与默认行高；Pierre
  通过 `index.css` 中的 `.pierre-code-surface` 自定义属性桥接收相同原语；xterm
  （`panels/TerminalInstance.tsx`）读取代码家族 + `s13` 并拥有其行高；mermaid
  （`thinkrail-extensions/visualize/web/mermaid.ts`）读取代码家族。这些第三方集成是
  `styles/typographyUsage.test.ts` 中的穷尽 allowlist。其他任何地方都要求类，且 `<pre>` /
  `<code>` 即使在一个已有类的容器内也必须携带一个：preflight 直接作用于这些元素，而直接
  匹配的规则胜过继承的家族。注意裸任意值 `font-[var(--font-mono)]` 有歧义 —— Tailwind 把
  它编译成无效的 `font-weight`，因此它静默无效果；可用的形式是
  `font-(family-name:--font-mono)`（`styles/fontClasses.test.ts` 在裸形式下失败）。
- **字体随产物发布。** `styles/generated/fonts.css` 导入自托管的可变字面（fontsource；
  Geist Variable + JetBrains Mono Variable，都带真正的斜体 —— 与 `apps/website` 共享，
  因此其版本位于根 `workspaces.catalog`）。哪些包属于哪个家族在 `styles/typography.json`
  中以 `selfHosted` 声明，因此字体栈与打包字面不会漂移；导入语句由它生成。Vite 把 woff2
  文件输出到 `dist/assets`，而 `apps/cli` 内嵌它们。没有字体 CDN —— 宿主是本地的且常常
  离线，而且外部 `<link>` 还会把首绘推到第三方之后，并在尽管选择退出 analytics 的情况下
  于每次加载时联系它。`e2e/fonts.spec.ts` 钉住两半（无 CDN 请求；真实字面存在）。
- **排版系统 —— 以 `typography.json` 为唯一真源、由它生成原语与语义样式、`<body>` 基础、
  370/400/500/600/800 字重策略、仅代码用等宽、两套 prose 系统，以及如何添加或修改样式 ——
  规格在 [src/styles/TYPOGRAPHY.md](src/styles/TYPOGRAPHY.md)**（`web-typography`）；把改动
  对照它检查。把它变成 CSS 的生成器是 [scripts/SPEC.md](scripts/SPEC.md)。
- **图标：`@remixicon/react`（默认 Line，活动/选中时 Fill）。组件：shadcn/ui**（Radix
  原语），由 [[module-ui]] 拥有并用我们的 token 工具类主题化（`@thinkrail/ui/utils` 中的
  `cn()`）—— 绝不是 shadcn 的默认 oklch 调色板。可访问菜单/对话框/工具提示使用这些。
  **每个普通动作按钮都是 `Button` 或 `IconButton`** —— 五种变体 × 三种尺寸，状态由组件
  拥有（见 [[module-ui]] 的 "Button system"）；调用点只添加放置，而
  `styles/buttonUsage.test.ts` 拒绝旧变体名、调用点皮肤与裸 `<button>` 皮肤。仅图标控件
  从 `IconButton` 必需的 `label` 获得其名称与共享 `IconTooltip`；两个组件都不接受原生
  `title` —— 解释，包括按钮为何被禁用，就是它的 `tooltip`。

## 必须做对（Get right）

- **`apps/web` 只依赖 `contracts`、`ui`、`extension-api/web` 与
  `thinkrail-extensions/*/web`。** 绝不值导入 `pi`；绝不导入 `server`/`shared`。
- 流式不变量：`text_delta` / `thinking_delta` **追加**；
  `tool_execution_update.partialResult` **替换**。尝试级的 `agent_end` 从不意味着空闲；
  自动工作只在 `agent_settled` 结束。
- 面板保持与布局无关，使移动 shell 是附加层，而不是重写。
- **应用组合的每个宿主 HTTP 路由都由 Vite dev server 代理。** 应用经 `/ws` 以及两条字节
  路由 `/files` 与 `/blob`（图片、PDF、notebook、markdown 相对图片）触达宿主，永远通过
  一个函数（`panels/resourcePane.resourceBytesUrl`）在 transport 的 HTTP origin 上组合。
  在 `bun run dev` 下该 origin 是 Vite，而 Vite 未代理的路由会以 200 状态返回 SPA 的
  `index.html` —— 一个 `<img>` 什么都不显示，一次 PDF 抓取以 HTML “成功”，而宿主服务的
  构建与 e2e 套件仍全绿。因此 `vite.config.ts` 在 `/ws` 旁边代理 `/files` 与 `/blob`，
  而 `devProxy.test.ts` 把该列表钉到 composer 的常量上。

## 后续（Later）

移动端单视图 shell 与 PWA 打包（可安装、离线 shell）在此拆分之上推进，而不触碰面板或
store。
