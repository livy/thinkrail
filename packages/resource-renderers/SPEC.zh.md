# @thinkrail/resource-renderers — 文件与 diff 渲染器、注册表与评审表面

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

把工作区资源变成可查看、可评审表面的一切：引擎中立的资源描述与渲染器注册表、十一个内置的
文件/diff 渲染器（code、markdown、binary、LFS、image、SVG、CSV、JSON、notebook、PDF、HTML）
及其评审组合器、讨论串卡片和区域表面，以及它们所运行的共享 Monaco、Pierre 和 markdown 文档机制。
它属于表现层：它不拥有内容加载、评审持久化、标签或导航——那些由宿主提供。

该包的存在是为了让一套渲染器栈服务每一处展示文件的地方：web 应用的文件与 diff 窗格、
文件对比 diff 标签，以及扩展工具卡片（visualize 的文件预览，[[module-ext-visualize]]），
因此一次渲染器修复会同时落到所有地方。

## 公开接口（Public surface）

一个 barrel，`@thinkrail/resource-renderers`（`src/index.ts`）：

- 注册表契约——类型、`registerResourceRenderer`、`resolveRenderers`、`describeResource`、
  `anchorLabel`、`isPlaceable`、`threadLabel`、`outdatedReason`、`rendererImplementationKey`，
  以及每个表面共享的内容规则：`resourceContent(text, meta, url)`（尚无元数据即暂定文本，
  缺失的哈希即缺失的一侧，没有字节 URL 的二进制一侧即缺失）、
  `diffResourceMeta(original, modified)`（diff 由其修改侧描述，除非该侧缺失；
  仅当两个存在侧都是文本时才是文本）和 `PENDING_TEXT_META`——外加
  `registerBundledResourceRenderers()`（幂等）。`diffRendererCapabilities` 仅凭一个路径
  （在内容加载之前）就返回它会解析到的 diff 渲染器的能力——从路径的 MIME 推断文本/二进制划分，
  使已知的图片和 PDF 路径解析到各自无布局的渲染器；MIME 推断本身保持模块内部。
  窗格用它来隐藏没有任何文件能用的作用域级布局与空白控件；
- 宿主接缝——`ResourceHostProvider`、`ResourcePreferencesProvider`、`ResourceEnvironmentProvider`、
  `ResourceScopeProvider`、`useOptionalResourceHost`、`useResourceScope`，以及 `ResourceHost`、
  `ResourcePreferences`、`ResourceScope`、`ResourceFileReviewProps`、`ResourceReviewSendProps`、
  `ResourceMarkdown`、`ResourceCodeHost`、`ResourceCodeLanguage`、`ResourceMarkdownProps`、
  `MarkdownRehypePlugins` 类型；
- `canonicalWorkspacePath`——应用工作区相对路径身份在本包中的副本（`lib` 的
  `projectRelativePath`：POSIX 分隔符、无 `.` 或空段、内部 `..` 被解析；
  `apps/web/src/panels/pathIdentity.test.ts` 保持两者相等）；`describeResource` 和
  `ResourcePreview` 使用它，因此带 `./` 前缀的工具路径会在宿主 fs 事件、评审存储和打开的标签
  所用的同一路径下读取、刷新并锚定评审评论；
- **`ResourcePreview`**——一个自加载的文件视图；或者在给定 `before`（另一个文件）时，是两个
  工作区文件的前/后 diff，带注册表的切换、未放置条带、评审评论与 Send review，以及调用方 slot
  （`leading`、`actions`、受控的 `rendererId` / `layout`，以及一个不透明的 `viewState`，
  后者会用写入该状态的渲染器实现给它打标记，因此渲染器切换绝不会恢复另一个渲染器的状态，
  调用方也无需重置）；
- 每个表面组合渲染器所用的部件，因此一个文件标签、一个 diff 标签、一个 Changes 区块和
  `ResourcePreview` 跑同一条序列：
  - `useRendererSelection(resource, intent, rendererId)`——针对环境手机档的候选、选中的渲染器、
    它的实现键（渲染器 id + 手机档）以及已放置讨论串状态；
  - **`usePreloadResourceView()`**——一个稳定的 `(resource) => void`，为环境的手机档解析视图渲染器、
    加载其实现并运行模块可选的 `warm`；它需要两个 provider 都在；
  - 惰性的 `RendererView` 挂载与 **`GuardedRendererDiff`**，唯一的 diff 挂载：昂贵 diff 提示
    （`diff-too-large` / **Show diff anyway**，按挂载，带调用方动作 slot）直到选择加入，
    然后是惰性 diff 渲染器，在超过调用方 `lineCount`（`surfaceLineCount`）的
    `VIRTUALIZED_DIFF_LINES` 时虚拟化；
  - **`RendererToggles`**——渲染器候选分段，加上当调用方传入 diff `layout`、渲染器支持布局
    且视口不是手机档时的 Split/Inline（一条规则，因此手机绝不会显示 diff 拒绝的布局，
    单文件视图也绝不会显示 diff 布局）；
  - `UnplacedReviewStrip`、`ReviewThreadCard`、`ReviewDraftContext`/`ReviewDraftPersistence`，
    以及 `createAskAgentRequest`；
- `useScrollViewState`——唯一的滚动视图状态 hook（一旦内容足够高就恢复已保存的偏移、让位于用户输入、
  在卸载时报告偏移，并在自身的滚动器仍处于恢复位置期间采纳更新的已保存偏移——即同一 slot 的兄弟实例
  在卸载时报告的那个，正如 Changes 评审的 Stacked 交接所做的），以及 `viewStateScrollTop`，
  后者是它用来校验已报告/已保存偏移的读取器；渲染器使用它们，web 应用的 Files 面板
  （`RememberedScrollArea`）也使用它们，因此应用与渲染器保持同一条恢复规则。

渲染器实现从不导出；它们留在每个注册项的 `loadView`/`loadDiff` 动态导入之后，而大 diff 估算、
提示和惰性 diff 挂载留在 `GuardedRendererDiff` 之后。barrel 的急切体积是注册元数据加上那些
窗格侧部件，应用的评审窗格和聊天的 visualize 卡片本来就会急切加载它们
（visualize 卡片给 web 入口图增加了约 20 KiB，以生产构建实测）。

## 宿主接缝（Host seam）

宿主用一个静态的 `ResourceHost` 取值把渲染器包在 `ResourceHostProvider` 中：

- `markdown`——渲染 markdown 的组件（`ResourceMarkdownProps`）；文档预览、渲染 diff 和 notebook
  单元格都通过它传入各自的插件与组件。
- `code`——代码高亮宿主（带语法加载器的 `languages`、`languageId` 规范化，以及从 CSS 变量解析的
  代码 `theme`）。Monaco 是页面全局的：第一个 preload 的 `warm` 或 `MonacoEditor` 用其宿主的
  `code` 初始化它一次（`ensureMonaco`，在 Monaco chunk 加载后同步），之后每个编辑器都复用该设置。
  `ensureMonaco` 拥有唯一一次页面生命周期的主题监视，因此即便在 preload 与首个编辑器之间发生换色
  也会重新着色，而 N 个打开的编辑器每次换色只付一次刷新。语法在其语言的第一个 model 上加载，
  或在该路径的 warm 中加载。
- `bytesUrl(workspaceId, path, oid?)`——图片、PDF 和相对 markdown 资源的宿主字节路由。
- `openFile(workspaceId, path, { keep? })` 与 `openComparison(workspaceId, before, path)`——
  相对文档链接（预览）和显式"在标签中打开"动作（keep）的导航。
- `readFile(workspaceId, path)` 与 `subscribeFileChanges(workspaceId, onChange)`——`ResourcePreview`
  的内容与实时刷新（`null` 路径表示"任何东西都可能变了"：一批被截断的变更或一次新连接，
  其错过的推送从不重放）。读取可能从早于资源元数据的宿主那里返回而没有 `meta`；
  预览把它显示为暂定文本，正如文件和 diff 窗格所做的那样。
- `FileReview`（一个 render-prop 组件：`{ workspaceId, path, kind, sessionId?, children(review) }`）
  与 `ReviewSend`——宿主的评审表面和 Send review 控件；可选的会话路由发送。
  该接缝是组件，而不是宿主对象上的 hook：把 hook 当作值传递违反 React 规则，并让 React Compiler
  放弃对预览的编译；前/后视图会嵌套两个 `FileReview`。
值只构建一次（应用在模块作用域构建）。它的函数是 memo 输入：`documentComponents`
每次调用都定义自己的链接和图片组件，因此新的 `bytesUrl`/`openFile` 身份会重挂载打开的预览和
notebook 单元格中的每个链接与图片。阅读器的行宽偏好（`fileLineWidth` / `fileLineWidthBounded`）
通过 `ResourcePreferencesProvider` 单独传入；只有 `MonacoEditor` 读取它，
因此宽度改变不会重渲染任何其他东西。手机档同样通过 `ResourceEnvironmentProvider`（注册表的
`ResourceEnvironment`，`{ mobile }`）传入；只有 `ResourcePreview` 读取它——窗格用各自的视口读数
解析——因此视口翻转只重渲染那些必须重新解析渲染器的预览。属于某个工作区和聊天的子树设置
`ResourceScopeProvider`（`{ workspaceId, sessionId? }`，用 `useResourceScope` 读取；
web 聊天包住它的转录列表）：当没有传入 `workspaceId` prop 时，`ResourcePreview` 回退到它的工作区，
并把评审发送路由到它的会话。作用域是它自己的 context 而非宿主字段，
因为按聊天展开的宿主会让每个预览的读取和订阅在聊天的 actions 变化时重启。
`ResourcePreview` 只依赖宿主稳定的 `readFile` / `subscribeFileChanges` / `bytesUrl`；
它按发出顺序应用读取（更新的变更事件绝不会丢弃进行中的读取，因此一串突发事件不会饿死首次加载），
当重读返回相同哈希时保留已加载文件——作为同一个状态对象，就像以同样方式失败的重读那样，
使 React 直接跳过，而不是调度一次会中断其下方未完成挂载（比如一个大型 diff 的挂载）的渲染——
并且在重读失败时保留最后的好内容，像文件标签那样，但标记为过期：失败会在其上方命名
（`resource-preview-stale`：`Couldn't refresh <path>: <reason>`），因此被移动或删除的文件绝不会
静默地冒充当前版本。渲染后的 markdown diff 用 `renderToStaticMarkup` 在 React 树之外渲染两侧，
因此它在自己的渲染中读取 markdown 组件和链接，并显式传给 `MarkdownDocument`——
在静态渲染内部读取 context 会抛错（在抽取期间由 Changes/渲染 diff 的 e2e 发现）。

`ResourceRenderer.loadView(environment)` 收到与解析时所用的同一个 `ResourceEnvironment`
（`{ mobile }`），因此代码渲染器在手机档视口选择 Pierre `File`，否则选择 Monaco，而不必读取
视口全局变量。已加载的模块（`ResourceViewModule`，从根导出）可以导出 `warm(host, resource)`；
preload 在每次请求之后调用它，晚于模块 resolve 一个任务，因此 chunk 的求值与 warm 是两个独立的
长任务。它必须廉价且幂等。Monaco 的 `warm` 运行 `ensureMonaco`，并在一个任务之后启动高亮器
（引擎和主题）以及该资源路径的语法（空路径不加载语法）。`usePreloadResourceView` 只预加载
`capabilities.preload` 置位的渲染器，而只有代码视图会置位：否则一个对每行悬停都做预热的宿主
会加载该行首选候选的 PDF、notebook 或 CSV chunk。一旦 chunk 加载完成，
Monaco 的挂载就是同步的：视图提交与编辑器之间没有库加载器或引导 promise。

**每个键只有一个已加载实现，且只选择一次。** `rendererMount` 维护一个组件映射和按实现键
（渲染器 id + 手机档）的进行中加载，供 preload 与挂载共享。`RendererView` 自己从 `renderer` 和
`mobile` props 派生该键，因此挂载和 preload 不可能对它产生分歧。React `lazy` 在首次渲染时
即使其 promise 已 resolve 也会 suspend，而显示的 fallback 至少持续 300 ms。因此已完成的加载
会把模块的组件存到其键下，除非某个挂载已经在那里存了一个 `lazy`；而第一个没有已存组件的
`RendererView` 会存一个复用进行中加载的 `lazy`。该选择只存一次，绝不切换：lazy 和直接组件是
不同的类型，互换会重挂载活的编辑器。在一次加载进行中创建的 lazy 会为之后所有挂载解析，
因此它不会再次 suspend。失败的 preload（在加载中或在 `warm` 中）记录一条警告，而不是 reject。
失败的加载会被遗忘，连同某个挂载在它进行中时为其存的 `lazy`，因此下一次挂载会重试，
真实的失败会抵达窗格的错误边界。

**评审草稿暂存按 slot 读取。** 一个在重挂载之间保留临时撰写内容的窗格提供
`ReviewDraftContext` 和一个 `ReviewDraftPersistence`（`rendererId`、`read(key)`、`subscribe`、
`update(key, change)`），后者在该区块生命周期内保持稳定；没有它，每个 slot 都是本地状态。
`useReviewDraftState` 通过 `useSyncExternalStore` 读取一个 slot，因此一次写入只重渲染该 slot 的
读取者，而不是其周围的渲染器表面。组合器的文本在它的选区旁边有自己的 slot：表面订阅选区，
只有 `ReviewComposer` 订阅文本，因此一次击键只重渲染组合器，而不是表面以及 Pierre 随之重渲染的
每个注解 slot（`e2e/perf/changesComposerIsolation.perf.ts` 钉住这一点）。文本写入只在拥有它的
选区仍然当前时落地；新的选区、一次关闭或一次戳记刷新都会让文本退场；重新打开同一选区则保留它。

## 决策（Decisions）

- **Markdown 是注入的，不是 import 的。** 宿主的 markdown（应用的 `chat/Markdown`）通过
  `@thinkrail/ext-visualize/web` 绘制 Mermaid；import 它会形成 包 → 扩展 → 包 的循环
  （一旦扩展渲染文件）。因此本包只拥有 props 契约；它的测试通过一个仅测试用的
  react-markdown + GFM 辅助器渲染。
- **宿主取值是普通且静态的。** 抽取时最初通过惰性加载器加 `use` 解析 markdown 和 code，
  以把聊天的 markdown 和 Shiki 目录挡在入口图之外。两者其实都已在其中——聊天静态 import 了
  它的 markdown，而那会拉入 `lib/highlighter` 和 `@shikijs/core`——所以加载器没有推迟任何东西
  （rolldown 报告 markdown import 无效），却要付出首次挂载时一次 suspend 渲染，
  以及一份宿主可能破坏的加载器身份契约。普通取值消除了两者，web 急切图缩减了约 1.5 kB。
  偏好拿到自己的 context，理由与取值静态相同：会变化的宿主取值会重渲染每个读取它的渲染器。
- **小辅助函数随其唯一消费者移动，或变成原语。** `stripFrontmatter`、`threadLabel`
  和 `outdatedReason` 留在这里；`ToggleSegment` 和唯一的加载占位
  （`SkeletonRows` / `LoadingRegion`）变成 `@thinkrail/ui/toggle-segment` 和
  `@thinkrail/ui/skeleton`，因为应用也使用它们。`describeResource` 和 `ResourcePreview` 通过
  本包自己的 `canonicalWorkspacePath` 规范化路径，因为本包无法 import 应用的 `lib`。
- **Monaco 运行它自带的 DOMPurify。** `monaco-editor` 用 `esm/vs/base/browser/dompurify` 下
  自己的副本净化 hover 和渲染后的 markdown，因此只有升级 Monaco 才会改变实际运行的净化器；
  它声明的 `dompurify` 从不被 import。该自带副本早于若干 DOMPurify 安全公告。
  这一暴露面被接受：这个只读阅读器没有语言服务、hover provider 或装饰 hover 消息，
  因此 hover markdown 是 Monaco 自己的内容，而且它调用 DOMPurify 时带 hooks 和
  `RETURN_DOM_FRAGMENT`，从不使用 `IN_PLACE` 或 `SAFE_FOR_TEMPLATES`。
  Monaco 0.57 自带 3.4.15。根 `overrides.dompurify` 的 pin 只是让声明的那份副本留在 Mermaid
  打过补丁的版本上（Mermaid 确实 import 它）；它不是对 Monaco 的修复。每次升级 Monaco 都要
  重新检查自带副本的 banner。

## 边界（Boundary）

- **允许的依赖：** `@thinkrail/contracts`（类型）、`@thinkrail/locales`（`MessageKey` 类型和
  用于其 UI 文案的 `./react`——`useT`、`<T>`、`localeOf`）、`@thinkrail/ui` 原语、React，
  以及渲染引擎（`monaco-editor`、`@shikijs/monaco`、`shiki`、`@pierre/diffs`、`pdfjs-dist`、
  `react-markdown`、`rehype-raw`、`rehype-sanitize`、`diff`、`jsondiffpatch`、`react-virtuoso`、
  `node-htmldiff`）。
- **禁止：** web 应用（store、transport、panels、chat、shell、lib、themes）、扩展、服务端，
  以及 pi 包。`scripts/check-module-boundaries.ts` 强制这一点；目前只有 `apps/web` 消费本包。
- UI 文案遵循 [[architecture]] 决策 23：渲染器外框通过 `useT()` 的翻译器（`viewer.*` 键）渲染，
  且 `check:i18n` 像扫描应用那样扫描 `packages/resource-renderers/src`。渲染器的 `labelKey`
  是目录键，绝不是已翻译的文本：注册在模块加载时只发生一次，因此视图切换、候选选择器和
  未放置条带在渲染时用各自的翻译器翻译它。
- 样式遵循应用的 token 规则：Tailwind 的 `@source`、颜色/间距/排版守卫以及 React Compiler 普查
  都扫描 `packages/resource-renderers/src`。

## 注册表与评审表面契约（Registry and review-surface contract）

拥有被文件和 diff 窗格所用的、引擎中立的资源描述、渲染器注册表和评审表面契约。选择由元数据驱动：
匹配的渲染器按意图和手机支持过滤、排序，随后是必需的文本或字节回退。当宿主 MIME 元数据缺失时，
MIME 匹配使用与资源描述相同的、由扩展名派生的回退。得到的有序列表既是派发策略，
也是窗格的视图切换语法。渲染器能力还声明所选表面是否支持复制其修改后源码、分栏/统一布局，
以及忽略空白的重算；窗格只暴露那些适用的控件。`preload` 标记值得在首次挂载前就加载的视图
（只有代码视图置位）。`boundedDiff: "always"` 标记自行管理视口的 diff 渲染器
（虚拟化表格、分页 PDF、沙箱化 HTML 框架），因此需要受限高度；`"large"` 标记自然布局
但在受限后虚拟化的渲染器，因此宿主在 diff 很大时给它设界；宿主只对大到需要虚拟化的 diff
传入 `virtualize`。其他所有 diff 按其自然高度布局，这正是连续的 Changes 评审能在一次滚动中
堆叠多个区块的原因。`HunkActions.triage` 是可选的：跟踪已保留 hunk 的表面提供已保留键、setter
和渲染器 hunk 内容键的汇聚点；没有它的渲染器不显示 Keep 控件。

`ResourceContent` 让文本、可取回字节和缺失的 diff 侧保持区分。尤其，缺失绝不由没有 URL 的
字节载荷来表示。`SurfaceReview` 携带权威锚点；渲染器只把选择器投影成几何。`isPlaceable`
回答是否至少有一个定位后的选择器具有渲染器为所请求的视图或 diff 意图声明的几何，
而 `anchorLabel` 是共享的紧凑引用。`ResourceDiffProps.originalResource` 在原始侧是另一个文件
（一次文件对比）时描述该侧；从文档路径解析相对资源或链接的渲染器——渲染 markdown diff——
为原始侧使用它，而可视化 diff 用它给该侧起标题；缺失时两侧共享 `resource`。
diff 渲染器可以通过 `ResourceViewProps.onPlacedThreadIds` 或
`ResourceDiffProps.onPlacedThreadIds` 报告它实际放置的讨论串 id；窗格把能力相符却被该集合
省略的 id 也视为未放置，覆盖渲染器本地几何，例如被移除的 JSON 指针、被删除的表格单元格，
或折叠的上下文。虚拟化的分页渲染器针对已知页数几何报告，而不是易变的画布驻留，
因此释放离屏页不会让它的讨论串变成未放置。对其意图没有锚点能力的渲染器可以忽略它的评审 prop；
窗格则把这些讨论串留在它的未放置条带中。

`HunkActions` 是 diff 引擎与其窗格之间的变更边界。渲染器只报告它计算出的、引擎中立的
原始/修改 `LineSpan` 对：`revert` 应用那一个块，`revertFile` 应用文件目标，`askAgent` 返回
`AnchorDraft` 加渲染器在修改侧放置的初始组合器文本。`agentWorking` 仅用于表现，绝不禁用
受 CAS 保护的动作。窗格拥有 wire 调用、作用域与期望哈希、撤销回执、过期视图刷新和错误 UI。
不可变作用域省略 `hunkActions`；渲染器此时必须省略每个变更控件，而不是模仿一个。

注册不拥有内容或标签状态。窗格拥有传输层 URL、评审集成、选中的渲染器，以及不透明的按标签
视图状态。`anchorLabel` 优先使用结构化定位符而非其强制性的文本回退：`table-cell` 是
`R<row>C<column>`，`json-pointer` 是指针本身，未知方案保留 `<scheme> <ref>`；`ipynb-cell`
对遗留的 `index:<zero-based>` 引用是 `cell <n>`，对稳定 id 是 `cell <id>`，并有一个可选的、
渲染器提供的 id→序号映射产生本地 `cell <ordinal>` 标签。分页区域是 `p<page> region`；
不分页的区域、行和文件保持其既有形式。注册只携带派发、几何、视口和具体的 diff 控件能力；
它没有未经强制的来源或执行策略元数据。沙箱化属于每个渲染器实现。

**宽重写先等在一个提示之后。** 源码 diff 在主线程上运行，其开销跟随改动行数而非文件大小：
一个 10 MB 全部新增的文件约 0.1 s 就 diff 完，而约 10k 改动行要花 20 s 以上。在挂载 diff 渲染器之前，
`GuardedRendererDiff` 估算改动行数（`largeDiff`：两侧行集的线性多重集差）。
超过 `LARGE_DIFF_CHANGED_LINES`（2000，约一秒的 diff 时间），或当顺序敏感的检查无法在
50 ms / 2000 次编辑内完成时，它显示 `diff-too-large` 加 **Show diff anyway**（按挂载）
以及调用方的动作。单侧 diff（新增或删除文件）从不被守卫：它不花 diff 成本。
有界检查封住多重集估算对乱序行的盲区。提示写着 "This diff is too expensive to show automatically"，
不带行数；守卫只回答自动渲染是否应当等待。每个 diff 表面都通过它挂载，因此应用的 diff 标签与
Changes 区块、对比标签和聊天卡片不会产生分歧。页面的第一个 diff 通过一次 Suspense 重试
（惰性渲染器）挂载，React 以可中断方式渲染它且永不过期：期间任何无关的更新（watcher 的一次轻推、
一次文件重读）都会连同其 `useMemo` 结果丢弃未完成的挂载，而每次重启都重算会让一个
Show-anyway diff 在工作区持续变化期间一直停在骨架上。因此 `PierreDiff` 从按内容键的模块缓存
（`diffCache`）取它的解析和变更块，重启的渲染可复用，并且只对有 hunk 动作的表面计算变更块，
因此只读预览只 diff 一次。它的两张表（解析、变更块；它们持有相同文本）都以条目保留的文本为界——
两侧合计 4 Mi 字符，约相当于提示阈值处（2000 行约 100 字符的改动行）二十次完整重写——
并以 64 个条目为界，多于一次 Changes 视口渲染所挂载的数量，因此一次性释放所有区块也不会让它
循环抖动；最新条目无论多大都永远保留，因此一个选择加入的巨型 diff 能在自身的重启中存活。
权衡：由于是模块级，它在相关表面卸载后仍让最多这么多文本（或那一个更大的最新 diff）加上
Pierre 对它的解析保持存活；而在一次渲染中合计超出预算的表面，每次重启都会重算。

## 内置渲染器（Bundled renderers）

内置候选由 `registerBundledResourceRenderers()` 注册一次：`thinkrail/code` 用 Pierre `FileDiff`
渲染每个源码 diff，并支持复制、布局和空白控件；`thinkrail/markdown` 提供 `RenderedDiff`；
`thinkrail/binary` 报告两侧的字节大小。丰富渲染器省略那些控件，只有 CSV、JSON 和 notebook 的
diff 支持复制修改后源码。`thinkrail/lfs` 在代码渲染器之上认领宿主的 `application/vnd.git-lfs`
文本（一个*就是* Git LFS 指针的文件，无论扩展名是什么），并在视图中显示一张卡片——
"Stored in Git LFS"、大小、对象 id、`git lfs pull` 提示——在 diff 中则每侧各一张卡片，
因为 Monaco 编辑器里三行指针协议完全解释不了一个 `.png` 为什么没有图片；Source 仍只隔一个切换，
而该渲染器不放置任何锚点，因此指针的讨论串留在未放置条带中。`RenderedDiff` 保留其 worker 隔离的
htmldiff 合并、加载/错误状态和 token 样式（文本变化时，上一次合并在屏幕上保留到下一次合并抵达，
因此实时刷新既不会闪过加载占位，也不会重挂载文档；已完成的合并存在模块级 LRU
（`renderedDiffMerges`）中，键为哈希和两侧渲染内容的长度，并像 `diffCache` 一样有界——
16 个条目、保留 4 Mi 字符——因此未变化 diff 的重挂载会同步渲染，既无占位也无重放的出现淡入，
而缓存命中会成为已挂载的文档，供下一次文本变化继续在屏幕上保留），但它不声明 diff 锚点：
两侧的讨论串都留在窗格的未放置条带中，**Show in Source** 选择代码渲染器，diff 撰写只在 Source
中可用。它由注册表匹配选中，而不是窗格里的路径分支。修改侧是历史版本的作用域不获得评审表面
或变更动作。

**渲染 diff 像 Pierre 那样聚焦于自己的改动。** 合并后的文档只解析一次，一个 prose-root 块在
它是或包含 `ins`、`del` 或 `[data-diff-node]` 元素时*算作改动*，**或**当其精确渲染的 HTML
与其对齐的 before 单元不同时（`changedUnits`）。对齐是位置性的：before 与合并后的单元序列
（块，然后每个列表的项）通过去属性化 HTML（`shapeKey`）上的 LCS 一一匹配，
因此文档别处的相同孪生体无法为某个块作证，在两个其他方面相同的 `<details>` 之间交换 `open`
会把两者都标记，而没有对齐对应物的单元保持可见。第二个子句之所以存在，是因为 htmldiff 只按
标签名给普通标签建键，并为相等的行输出 *after* 的 token：被勾选的任务复选框、
`<details>` → `<details open>`、列表的 `start`、或图片的 `alt` 永远不会获得标记，
因此没有它它们会作为"未变化"被折叠，而空提示会声称预览完全相同。
这样的块在其 after 状态保持可见（不标记，因为合并没有可高亮的东西）。连续未变化块按 git hunk
语义折叠（`renderedDiffFocus.focusSegments`）：`FOCUS_CONTEXT_BLOCKS`（2）个块在改动两侧保持可见，
开头或结尾的连续段只在触及一端的那一侧保留上下文，而单块连续段从不隐藏，
因为替换一个段落的展开器省不下什么却要花一次点击。同一条规则向下一层适用于一个发生改动的
`ul`/`ol` 的项，当至少一项改动时——一份 markdown 规格经常带着三十个条目、其中一条被编辑——
而仅自身属性不同的列表，以及表格、引用和嵌套列表整体渲染；有序项保留其编号
（`start` 与显式 `value` 按 HTML 的整数解析规则读取，因此 React 留在 DOM 中的无效值——
`start=""`、`value=""`——会像浏览器那样回退到 `1` / 当前计数；`value` 优先，其余继续计数），
因此隐藏项绝不会重排其余项。每段隐藏连续段是一个 `rendered-diff-collapsed` 按钮，命名数量，
对块连续段还命名它隐藏的最后一个标题（其下方可见内容所属的区块——相当于 Pierre 的行信息分隔符）。
点击会就地展开该连续段，单向；展开是组件本地且位置性的，因此实时刷新会保留起点位置未变的展开，
并重置其余的。没有任何东西一次性提供整个合并文档：Source 和文件预览已经做到了。
没有任何块变化的合并——front matter 在渲染前被剥离，空白或 HTML 注释不渲染——显示
`rendered-diff-empty` 提示指向 Source，并把文档折叠成单个展开器，而不是把一份未标记的完整文档
当作 diff 呈现。可见块从解析后的元素重建（标签、属性、`innerHTML`），绝不包裹，
因此 prose 样式所目标的 DOM 不变；净化器放行的布尔属性（`details[open]`、
独立复选框的 `checked`/`disabled`）被映射为 `true`，因为 React 会丢弃空字符串布尔值。

`thinkrail/image` 渲染宿主支撑的字节 URL，支持适应、原始尺寸、按钮/滚轮缩放、固有尺寸和字节大小，
始终置于 `.media-backdrop` 透明棋盘之上（视图的图片和两个 diff 框；difference 混合保持平坦画布，
使棋盘不会渗入相减运算），因为在纯深色画布上，透明 PNG 上的深色 logo 与加载失败的图片无法区分。
在渲染后的固有框上拖动会产生一个规范化的 `region` 选择器；已放置的讨论串把该选择器投影回
轮廓矩形和带编号标记，卡片在图片下方。它的 diff 默认 2-up，也提供滑动、洋葱皮和 difference
混合模式。2-up 直接锚定到所绘制的那一侧；叠加模式显式暴露活动的 old/new 锚点空间，
因此 base 与 worktree 坐标绝不互相翻译。缺失的一侧是一个带标签的空框。
图片缩放、渲染后的固有宽高边界、区域矩形、滑动裁剪/分隔线位置，以及洋葱皮不透明度，
属于量测得到的几何，因此是无内联样式规则的有界例外：只有那些值是内联的，
而每个颜色、边框、间距和控件外观仍然是 token 工具类。

`thinkrail/svg` 在两个维度上把矢量内容容入其窗格（宽 ≤ 960 px），在 layout effect 中量测，
因此一次拖动的首个几何就已经是最终值。它是排名更高的 SVG 候选，而 Source 仍可通过
`thinkrail/code` 使用。它的 `sandbox=""` iframe 收到一个最小 `srcdoc` 文档，其中唯一的图片是
百分号编码的 `data:image/svg+xml` URL；SVG 源码从不进入文档的 HTML 流。该框架没有脚本或同源能力，
CSP 为 `default-src 'none'; img-src data:; style-src 'unsafe-inline'`，因此 SVG 脚本和外部引用
保持惰性。文档只接收解析后的工作区/内容背景与前景语义 token 值，并通过 `@thinkrail/ui/theme` 的
`onThemeSwap` 重建。在框架上拖动撰写与栅格图片相同的规范化 `region` 选择器，
量测针对从根的 `width`/`height`/`viewBox` 解析出的固有尺寸（框架即按该宽高比定尺寸，
因此包含矩形就是框架）。它绝不退化成整文件草稿：早前的一个修订就是这样（`selectors: []`、
label 为 `file`、等待源码区间映射），而一个以"整个文件"抵达 agent 的拖动矩形读起来就像丢失了坐标。
SVG 元素的源码区间仍未映射，因此锚点是几何而非行范围；分数是相对 SVG 根视口的，
发送包会告诉 agent 反演 viewport-to-viewBox 变换（viewBox 原点加 `preserveAspectRatio`）
以找到其下的元素。它的 diff 使用与栅格图片相同的四种视觉模式和两个锚点空间；
每侧的叠加层通过它自己的 object-contain 矩形投影，只有量测到的宽高边界加
滑动/分隔线/不透明度叠加几何使用内联值。

`thinkrail/csv` 用 RFC-4180 状态机在本地解析 CSV/TSV，包括转义引号、CRLF，以及跨源码行的
带引号字段。分隔符**从首二十条非空记录的形状推断**（`sniffDelimiter`，忽略 BOM）：
候选（`,`、`;`、`\t`、`|`）中的某一项只有在把每条采样记录切成相同数量（至少两个）字段时才合格；
逗号只要合格就优先，否则取唯一的字段最多者，否则取逗号——因此恰好含分号的表头值绝不会把
逗号文件重新分类，而 `id;name;score` 的本地化导出是表格而非单列；`.tsv` 永远是制表符。
diff 先采样修改侧，再回退到原始侧，因此两侧用同一分隔符解析。视图固定表头并用
`react-virtuoso` 虚拟化数据行。选中的单元格或 shift 扩展矩形发出所选行的原始文件 `lineRange`
加 `structural { scheme: "table-cell", ref: "<row>:<column>" }`，表头行为零；
缺失的单元格，或某坐标所在行不再与宿主重锚后的行范围相交，会在实际放置时被省略而不是被猜测。
统一 diff 用 `diffArrays` 对齐完整的原始行键，把等长的删除/新增连续段配对为改动行，
并且只高亮不同的单元格；未匹配的行使用 success/error 的淡淡表面。
diff 选择使用修改后源码行，被删除的行除外，它们留在 base 锚点空间。

`thinkrail/json` 只匹配 `*.json` 和 `*.jsonc`。它的扫描器**刻意容忍 JSONC**——
`tsconfig.json`、`.vscode/*.json` 和大多数编辑器设置文件都带注释和尾逗号，
拒绝它们的树会恰好把这些文件赶到 Source——但它对此诚实：文档记录它所需的 `dialect`，
一个只能按 JSONC 解析的 `.json` 文件会在树上方显示 `json-dialect` 提示（`.jsonc` 文件不显示，
因为对它来说这正是要点）；连 JSONC 都不是的文件会在视图和 diff 中都渲染显式的 `json-invalid`
提示指向 Source，而不是留一个空窗格。扫描器一趟构建解析后的值和 RFC-6901 指针到源码行的索引；
重复的对象键解析为最后一次出现，与 JSON 值语义一致。选中非根节点会发出其强制性的 `lineRange`
加 `structural { scheme: "json-pointer", ref }`；编辑后不再存在的指针，
或现在指向宿主重锚后行范围之外节点的指针，被报告为未放置。可折叠的统一 diff 来自一个带移动检测
和稳定对象哈希的 `jsondiffpatch` 实例，给新增/删除/改动/移动节点着色，
并把原始值替换显示为 old → new。被删除的节点和注释留在 `review.base` 上并使用 base 扫描器行；
其他所有选择使用修改后文档和 `review.worktree`。无效源码不渲染树，使 Source 成为备选候选。

`thinkrail/notebook` 接受 nbformat-4 JSON，并复用宽容的位置式 JSON 扫描器把每个渲染的单元格
绑定到其原始单元格对象的行范围。Nbformat 4.5 的 id 是 `ipynb-cell` 的结构化 ref；更旧的 notebook
使用 `index:<zero-based>`。Markdown 单元格使用共享的 Markdown 解析器，不带原始 HTML。
它的 notebook 组件映射（`notebookMarkdownComponents(resource)`，按 notebook 构建，
因为图片解析相对于 notebook 自身路径，如 Jupyter 中那样）允许应用文档中的 base64
PNG/JPEG/GIF/WebP/AVIF 图片，把 SVG 数据图片路由到空沙箱的 `NotebookFrame`，
通过 markdown 预览所用的同一个注入式字节 URL 组合器从宿主 `/files` 路由加载**相对**图片
（向上越过 worktree 的遍历是惰性文本，绝不发出请求），并把链接和每个远程图片 URL 渲染为惰性文本——
notebook 的 `![plot](figures/x.png)` 是仓库内容，第三方 URL 不是。Notebook 语言元数据必须通过
共享的 Shiki 目录解析；缺失元数据默认为 Python，未知取值渲染为纯文本。流和错误输出中的 ANSI
被移除。MIME 包依次选择 image（PNG、JPEG、GIF、SVG）、HTML、JSON，然后纯文本。
SVG 输出保持为惰性框架文档内的数据图片。HTML 输出使用与 HTML 渲染器相同的已净化、
无能力文档。单元格评审同时发出原始行范围和结构化 ref，把卡片保留在已解析单元格下方，
只报告实际放置，并对打开的组合器做哈希戳记。diff 先对齐 id，其次对齐精确规范化源码。
剩余连续段按有界行 token 相似度按序对齐；不使用字符级全对 diff。改动的源码使用一个
provider 支撑的统一 Pierre 单元格 diff，图片输出复用视觉对比模式，文本输出使用 Pierre，
HTML/JSON 输出保持并排。原始单元格选择器仍留在 `review.base` 中；两侧都不做转换。

`thinkrail/pdf` 匹配字节 PDF，并惰性加载 pdf.js 及其 Vite 输出的 worker。在页数元数据已知之前
不报告放置。一个页区域讨论串恰在其从 1 开始的页存在时可放置，与那一页当前是否拥有画布无关。
被聚焦的页被强制进入渲染窗口，且焦点只在其区域卡片挂载后才被消费。渲染窗口是可见页加每侧两页。
离开该窗口的工作被取消；画布和位图被释放，diff 对象 URL 被撤销。组合器身份使用文档身份加页，
绝不用缩放。工具栏的当前页是首个可见页（在任一页报告可见性之前的导航目标），
因此它同时跟随滚动和导航；该页与缩放像其他恢复滚动的渲染器那样在视图卸载时写入标签视图状态。
PDF diff 使用一个同步的页序列：每对页从 2-up 开始，可在共享的滑动、洋葱皮和 difference 模式间
切换，缺失的一侧由显式的空页框表示。

`thinkrail/html` 先把源码解析成文档，然后在无能力的 `sandbox=""` iframe 中预览它。
净化器移除活动、导航、嵌入和表单元素；去掉事件处理器和非 `data:` 资源 URL；
把锚点变成惰性的带文本元素。样式保留。Notebook 的 HTML 输出使用同一个净化器。
注入的 CSP 恰好是 `default-src 'none'; img-src data: blob:; style-src 'unsafe-inline'; font-src data:; base-uri 'none'; form-action 'none'; object-src 'none'; frame-src 'none'; child-src 'none'`。
diff 是共享的视觉框架，只限于 2-up 沙箱化文档。它不声明锚点几何，
因此 HTML 的讨论串留在窗格拥有的未放置条带中，Source 是唯一的撰写候选。
视图和 diff 在提示栏共享一个**预览宽度**开关——Desktop 1280 / Tablet 768 / Phone 390 / Fill，
默认 Desktop，作为 `{ viewport }` 视图状态持久化，并在调用方视图状态变化时采纳
（`useHtmlViewport`，因此聊天卡片会跟随在其全屏中选定的宽度）：框架按设计宽度渲染，
并被 CSS 缩小到其盒子（绝不放大），保留自己的垂直滚动，因此窄窗格、2-up 的一半或聊天卡片
展示的是缩小后的真实布局，而不是意外的移动端重排；Fill 是旧的重排行为。
缩放是包裹层上的纯几何——iframe 保持 `sandbox=""`，没有任何东西量测文档（没有
`allow-same-origin`）；包裹层的盒子在 layout effect 中量测（本包唯一的 `useElementSize`，
与 SVG 视图和区域表面共享），因此首次绘制就已适配，而不是为一个框架先在窗格宽度下重排文档。
净化器也剥离表单控件，因此 mockup 用带样式的元素画按钮和输入框；HTML 上的点（区域）评论
暂不提供。

Pierre 用当前忽略空白取值解析两个完整文本侧（`absent` → `null`），使用 `thinkrail` CSS 变量
Shiki 主题、词级内联改动、带行信息 hunk 分隔符的折叠未变化区域，以及分栏/统一布局；
手机档视口强制统一布局并隐藏 Split/Inline 分段，因此没有控件承诺视口会拒绝的布局。
折叠阈值（`COLLAPSED_CONTEXT_THRESHOLD`，3 行）就是 git 自己的默认 hunk 上下文（`-U3`），
因此 Pierre 在改动周围保持可见的内容正是评审者从 `git diff` 已期待的，
而同一个数喂给 `renderedDiffLineNumbers`，后者决定讨论串的结束行是在屏幕上还是停在未放置条带中——
一个常量，因此两者绝不会分歧。聚焦揭示（`usePierreFocus`）只在一个聚焦请求待处理时才安装子树
`MutationObserver` 加 `ResizeObserver`，并在卡片揭示的那一刻断开两者；它不是常驻监听器，
因此聚焦落定后 Pierre 在每次注解变化时的重渲染都不花成本。两侧相同的 diff
（一个在带外提交后离开改动集的文件）在 Pierre 表面上方显示显式的 `diff-empty` 提示，而不是空窗格。
惰性的 Pierre 文件/diff 模块只在它们的表面渲染时才挂载 `PierreProvider`；它把模块单例
（`getOrCreateWorkerPoolSingleton`，四个来自 `@pierre/diffs/worker/worker.js` 的模块 worker）
交给 Pierre 的 `WorkerPoolContext`，并且**绝不终止它**：池在任何代码表面首次渲染时创建，
然后活到页面结束。Pierre 自己的 provider 在其最后一个实例卸载的那一刻就拆掉池，
而虚拟化评审列表在每次跳转或快速滚动到尚未加载的文件时都会到达那一刻；每次重建都重新生成
worker、重新拉取 WASM、主题和语法、丢掉每个已高亮的 AST，并在初始化完成前什么都不画——
堆叠视图的"永远加载中"。四个空闲 worker 就是绝不再付这份代价的价钱；列表一次挂载少量区块，
因此四个够用。池的高亮缓存以 diff 的 `cacheKey` 为键，而 Pierre 只在两侧都携带该键时才派生它；
`PierreDiff` 自己从存在侧的哈希为一侧（新增或删除）diff 建键，因此重挂载——
Stacked/One-file 切换，或某个区块滚回视口——首帧就是高亮的，而不是先纯文本再上色。
手机代码文件实现是带同一主题和评审语法的 Pierre `File`；只有桌面文件加载 Monaco。

**Pierre 表面是虚拟化的。** 未虚拟化的 Pierre 把每一行都放进 DOM（10 万行文件约 40 万节点），
而 WebKit——桌面外壳的引擎——无法为其上样式或布局，导致应用挂死。
`pierreVirtualizer.ts` 每个表面拥有一个 `PierreVirtualizer`（核心 `Virtualizer` 的子类），
通过 `VirtualizerContext` 提供，使 `FileDiff`/`File` 构建各自的虚拟化实例，
并与 `useScrollViewState` 共用同一个 ref 回调来建立，因此滚动恢复继续有效。
滚动器的第一个子元素是内容容器，必须从首帧就给 Pierre 元素非零高度
（在 `min-h-full` 列中的 `flex-1`）：Pierre 的滚动锚点会把零高度文件视为位于视口之上，
并在高度落地后跳到其末尾。滚动器退出浏览器的滚动锚定（`overflow-anchor: none`，
如 Pierre 自己的 `CodeView` 所做），因为 Pierre 自己修正滚动。注解子元素留在 light DOM 中，
但只在渲染窗口内布局，因此聚焦揭示先请求虚拟化器把讨论串的结束行居中（`revealLine`，
经由已连接实例的 `getLinePosition`），然后在卡片渲染后居中卡片。新打开的 diff 组合器
得到同样处理——Ask agent 把它锚定在 hunk 的结束行，而对一个高 hunk 来说该行在窗口之外——
并且 `ReviewComposer` 把初始聚焦推迟到其 textarea 有盒子之后，因为聚焦一个未渲染的元素
是静默的空操作。被恢复的组合器改为保留恢复后的滚动位置。
虚拟化器只对真实滚动器开窗：一个有自然高度的堆叠 Changes 区块与其内容一样高，
因此它的虚拟化器"看见"每一行并全部渲染（10 万行新增文件约 7.2 万行，一落地就冻结应用）。
因此代码渲染器声明 `boundedDiff: "large"`，超过 `VIRTUALIZED_DIFF_LINES`
（1000，按较大文本侧的行数计，包含未变化上下文）的堆叠区块获得与 `"always"` 渲染器相同的有界
`60vh` 视口并拥有自己的滚动器。按源码行计数使展开后仍保持虚拟化，
即使大文件只有很小的编辑，也能覆盖没有 git 行数统计的未跟踪文件。
同一阈值在每个宿主中门控虚拟化：`GuardedRendererDiff` 只在超过它时传入 `virtualize`，
低于它时 `PierreDiff` 不提供虚拟化器，因此普通 `FileDiff` 像虚拟化之前那样渲染每一行。
虚拟化小 diff 什么也没换来，却付出两次失败。虚拟化实例的水合在其容器仍为空时量测偏移
（强制布局）；在堆叠列表中，其末端位于最大滚动量处，以便最后一个页头能到达顶部，
那次塌缩的布局会钳制 `scrollTop`，而 Virtuoso 的尺寸修正会叠加这次跳动，
直到被评审的区块离开渲染窗口。并且 `@pierre/diffs` 1.5.1 的虚拟化就地列重渲染会丢掉实例
仍记录的注解行，使刚打开的组合器无处安放，直到某个东西强制一次 `rerender()`。
大型 diff 仍带着这个注解风险。

**虚拟化滚动按 chunk 成本调优。** Pierre 把每个代码列布成一个 subgrid、每行一个项，
因此每次渲染窗口移动都会重排整个已渲染网格，而一次移动的成本跟随插入的行数，而不是保留的行数。
分栏布局在新行携带注解（hunk 工具栏、讨论串）时付两次：Pierre 的 `ResizeManager`
把每个注解行跨两列配对并强制一次 `getBoundingClientRect`，然后同步该对的
`--diffs-annotation-min-height`，强制第二次布局。在 Pierre 的 50 行 chunk 下，
Chromium profiler 显示每次移动约 90 ms 的长任务，WebKit 中约 80–95 ms 的卡顿。
因此 `usePierreVirtualizer` 传入 `hunkLineCount: 5` 和 400 px 过滚动（Pierre 默认：50 行、1000 px），
在一个 6k 行、150 hunk 的 diff 上用滚轮甩动实测：WebKit 中 inline 和 added-file 滚动从 12–34 帧
掉帧降到 2–3 帧，最坏的分栏卡顿大致减半（约 45 ms）。在密集 hunk 工具栏上的分栏滚动在 WebKit 中
仍会掉帧；没有工具栏时则流畅，而 `overflow: "wrap"`（因两侧共享网格行而跳过配对）
能改善中等速度的滚动但不能改善甩动。虚拟化器的 `lineHeight` 度量是真实行高：
`.pierre-code-surface` 把 `--diffs-line-height` 取整到整像素（`s11` × `default` = 14 px，
与 Monaco 的取整一致），而 `codeLineHeight()` 从 token 派生出同一个数。
相对 Pierre 20 px 的默认估算，一个 20k 行文件的滚动高度起初高了约 40%，并在滚动时随行被量测
而收缩。实测后否决：把不可见占位镜像到对侧分栏列（减少了绘制面积但在两个引擎中都不减少掉帧）
和更大的过滚动（在 5 行 chunk 时掉帧约为两倍，且快速甩动时空帧并不更少）。

**`PierreScroller` 防止合成器显示未绘制的行。** WebKit（桌面外壳）在主线程之外滚动 overflow：
在 20k 行文件中拖动滚动条每步跳约 2,600 px，而每步都会在新窗口被主线程绘制之前显示一帧
完全空白。没有过滚动能覆盖这一点，而像行那样绘制 Pierre 的占位元素也没用：空白是未光栅化的空间，
而不是占位。因此滚动器保留原生滚动（滚动条、滚轮、按键、惯性、自动滚动），但持有一个量测为
内容高度的 sizer、一个被裁剪到滚动器高度的 `sticky` 视口，以及由 `-scrollTop` 在主线程上平移的内容。
线程外滚动只移动 sizer，因此被钉住的视口持续显示最后绘制的行，直到主线程在渲染窗口的同一帧里
移动内容——快速拖动会滞后，但绝不空白。Pierre 在其 rAF 过程中修正滚动位置，
而 `useScrollViewState` 从 ResizeObserver 恢复它，两者都在该帧的滚动事件之后，
因此滚动器自己的 `scrollTo` 和 `scrollTop` 同步地重新同步变换；否则 Pierre 会对着过期偏移量测行。
自然高度的堆叠区块跑同样的结构但不滚动：视口的高度在 sizer 设置之后从滚动器读回，
因此它绝不会反向喂给内容自身的高度。

- **渲染后的 markdown 可导航。** 在预览中，链接 + 图片相对文件自身路径解析（通过
  `markdownLinks`，作为 `a`/`img` 渲染器传入）：**相对链接**通过共享的 **`openFileInTab`**
  （与 `FileTree` 使用的同一流程）在**预览**标签中打开目标文件——跟随链接属于浏览，
  因此复用该 slot 而不是像 VS Code 那样把源文档提为当前——被接受的文件目标是样式化为文档链接文本的
  按钮，绝不是带原始相对 `href` 的锚点，因为该 URL 不是 ThinkRail 路由，原生导航会逃逸到 Main 页面；
  URL 路径名解码一次，两种路径分隔符形式在解析前都被规范化，而畸形编码和越过 worktree 根的遍历
  会产生惰性控件，而不是打开错误的文件；slot 就是 slot，无论打开来自何处——**文档内 `#` 链接**
  滚动预览（标题携带来自仓库内 `remarkHeadingIds` transform 的 slug id），**外部**链接打开新标签，
  **相对图片**重写为宿主 **`/files/…`** 路由（经由 `resourcePane.resourceBytesUrl`，
  注入到 `documentComponents`，因此 markdown 层绝不自己去摸传输层）。跨文件链接的 `#fragment`
  暂不跟随（只打开文件）。
- **文档 markdown 渲染已净化的原始 HTML；聊天 markdown 不。** README 会用
  `<p align="center"><img src width>` 放 logo 和横幅，用 `<img align="right">` 浮动图片，
  用 `<details>` 折叠区块，并用 `<picture><source media srcset>` 交付主题感知的 logo；
  若被当作字面标签显示，这些文档恰好会丢掉它们最显眼的内容。
  `markdownHtml.documentRehypePlugins` 先运行 `rehype-raw`，再在 GitHub 的 `defaultSchema` 上
  运行 `rehype-sanitize`，带三处差异：仓库内 remark transform 发出的 alert 元素
  （`mdalert` + `variant`）被允许，`<source>` 保留 `srcset`/`media`/`type`/`sizes`。
  GitHub 的 **`user-content-` 覆盖前缀在每个 `id`/`name` 上都保留**（文档绝不能凭空造出
  `window.MonacoEnvironment`，或通过用全局名命名元素来遮蔽全局），因此文档内 `#slug` 链接
  通过 `scrollToAnchor` 解析，后者先尝试带前缀的 id，再尝试裸 slug。一切活动或样式性的东西——
  `script`、`iframe`、`style`、事件处理器、内联 `style`、`javascript:` URL——都被丢弃，
  `data:` 图片也一样（schema 的 `src` 协议是 http/https/relative）。原始 `<img>`/`<source>`
  走与 markdown 图片相同的相对 URL 重写，因此 README 的 logo 从 worktree 加载；`srcset` 用
  HTML 候选语法解析（`srcsetCandidates`：URL 内的逗号、描述符括号），并且由于净化器和
  react-markdown 都不对 `srcset` 做协议检查，每个候选只按 worktree URL 或
  `http(s)`/协议相对 URL 重新准入——`javascript:`/`data:` 候选被丢弃，裸 `src` 中这类取值也一样；
  `width`/`height` 保留下来，`align="left|right"` 映射为浮动工具类而不是表现性属性。
  用于评论的源码行戳记在净化**之后**添加，因此净化器永远剥不掉它们。聊天的 `Markdown` 原语
  不受影响：模型输出不是用户撰写的文档。
- **桌面文件源码在同步的文件列处换行。** `MonacoEditor` 把阅读器偏好 `fileLineWidth` 用作
  `wordWrapColumn`（40–240，默认 120）。独立的 `fileLineWidthBounded` 默认映射到 Monaco
  `wordWrap: "bounded"`，在挂载的文件窗格处更早换行；关闭映射到 `"wordWrapColumn"`，
  在更窄窗格中保留所选列并横向滚动。广播的变化会更新已挂载的桌面文件编辑器。
  Pierre diff 和手机文件自己拥有横向溢出，不消费这个 Monaco 偏好。渲染后的 Markdown 和
  渲染后的 Markdown diff 保留各自约 78ch 的阅读行长。React Compiler 以宽度、bounded 和路径
  memo 化 `MonacoEditor` 的 options，编辑器只在那个对象变化时调用 `updateOptions`；
  options 读取的排版与间距 token 是静态生成的 CSS（主题只设置颜色）。
- **桌面 Monaco 表面是阅读器，不是语言工作站。** 它的仅文件选项保留配置的换行、生成的代码排版、
  移除 minimap/概览标尺、6px 无阴影滚动条，并增加只读消息、作为其可访问标签的路径、
  8px 上内边距、三字符行号槽、整行活动行绘制、平滑滚动、活动括号对 + 缩进参考线、
  单文件出现处高亮，以及歧义 Unicode 抑制。不透明的 `viewState` 只在其编辑器状态形状通过校验后
  才被恢复，并在卸载时保存，因此标签切换能往返滚动与折叠。随挂载一起抵达的评审聚焦会跳到其行
  （`revealLineNow`，即时滚动）：在编辑器首次布局之前启动的平滑滚动可能中途停下，
  而被处理的聚焦不会重试，因此卡片会停在屏幕之外。对已挂载编辑器的聚焦仍然平滑滚动。
  Monaco 从 `editor.api` import，只带阅读相关的 contribution（查找、折叠、括号匹配、
  上下文菜单/复制、链接、hover、词高亮、粘性滚动、跳转到行/命令面板、只读消息和 Unicode 高亮）；
  保留词高亮是为了保住 F7。每个 contribution 通过其类型化的
  `monaco-editor/features/<name>/register.js` 条目加载，因为 Monaco 的 `exports` 映射关闭了
  深层 `esm/vs/...` 路径。编辑器核心自己加载 `coreCommands`。`codicon-modifiers.css` 未被导出，
  因此 codicon 旋转动画（这里只有 hover 的加载图标）保持静止。TS/JS/JSON/CSS/HTML 语言服务
  contribution 及其 worker 都不存在，因此这个查看器在构造上不发出任何语言服务诊断。
  每个编辑器挂载拥有自己的 model：model id 是路径之前的、按挂载的 `useId` 作用域，
  因此同一文件在屏幕上出现两次（一张聊天卡片与其全屏、一张卡片与该文件的标签）
  绝不会共享一个会被某次卸载在另一个脚下释放的 model。
- **`MonacoEditor` 自己挂载 Monaco。** 一个 layout effect 在挂载视图的那次提交中就创建编辑器
  及其 model，没有加载状态，然后恢复视图状态并挂接评审评论（评审只在挂载时挂接）。
  它的布局清理先保存视图状态，再按此顺序释放编辑器和 model。在活编辑器上改变路径会就地换 model，
  语言跟随路径，因为 `RendererView` 以实现为键，而不是以路径为键。新文本只在它与上次应用的文本
  不同时才调用 `setValue`：编辑器是只读的，因此上次应用的文本就是 model 的文本，
  读回它（`getValue()`）只会白付一次完整拷贝。model 与 options 两个 effect 跳过首次运行，
  因为挂载刚刚正以这些输入创建了编辑器。编辑器活在 layout effect 中，因此视图 Suspense 边界内
  的再次 suspend 会释放它（以及任何打开着的评审组合器）；今天那里没有任何东西会再次 suspend，
  因为每个 `RendererView` 有自己的边界且其已加载组件被缓存。
  在挂载提交中的评审聚焦用 `ScrollType.Immediate` 揭示其行：在编辑器首帧之前发出的平滑揭示
  永不落地（由远处 worktree 评论的 e2e 发现）。之后对活编辑器的聚焦则平滑滚动。
- **代码表面从通用 token 重新着色，并具备弹性。** `monacoSetup` 拥有唯一的 `EDITOR_THEME`：
  `createFileEditor` 应用它，主题刷新定义它。主题拥有一个 TextMate scope 定义：
  聊天使用其活的 CSS 变量形式，而 Monaco 把同一定义加上它完整的编辑器/部件/菜单/输入颜色映射
  解析为 hex，并直接由它定义 Monaco 主题。Token 颜色来自 `shikiTokens.ts`，
  它是不依赖 Monaco 的拥有者，保管一个 Shiki 高亮器、其按语言的语法加载以及颜色→scope 映射。
  它精选的语言目录与聊天高亮器共享，并为桌面文件格式做了扩展；缺失的 Monaco 语言 id
  连同其文件关联一起注册，然后每个目录 id 获得一个 `registerTokensProviderFactory` 条目。
  换色时在另外两个名字之一下加载高亮器主题，并从它定义的同一批已转换 Monaco 规则重建 scope 映射，
  重定义 Monaco，并选中 `EDITOR_THEME`。普通/高对比基色来自 manifest 的 appearance/contrast
  元数据——绝不来自已知 id——并在显式映射之外的专门颜色上保持继承。读取经过
  `lib.cssColorToHex`；缺失或不可解析的 token 被省略，使选中的基色调色板胜出，而不是让面板崩溃。
  Pierre 唯一注册的 `thinkrail` CSS 变量 Shiki 主题发出变量引用而不是目录颜色；
  继承的 `--diffs-*` 属性把前景/语法映射到 `--code-*`，把画布映射到工作区/内容角色，
  把新增/删除着色映射到反馈角色，因此换主题无需重新高亮。`TerminalInstance` 同样从完整的
  16 槽 ANSI 变量集重建。Monaco 和 xterm 在提供时消费可空的编辑器选区前景色覆盖。
- **Monaco 每种语言加载一个语法，并在 Oniguruma WASM 上分词。** 预先编译全部 35 个目录语法
  曾是在第一行上色之前的一次 210-250 ms 主线程任务（4x CPU 下约 870 ms），而 JS 正则引擎分词
  比 Oniguruma 慢约 4.5 倍（一个 2,861 行的 TS 文件：约 1,715 ms 对 380 ms）。工厂的 `create()`
  在 Monaco 首次遇到该语言的 model 时加载语法；`languageForPath` 的探测 model 在编辑器挂载前
  就把它启动起来。并发的请求共享一次加载，未知 id 保持纯文本。Monaco 只向一种语言的工厂要一次
  provider，因此失败的加载回答纯文本并把该语言标记为缺失；高亮器或引擎启动的失败不被缓存，
  因此之后的请求会重试。该语言下一个加载成功的 model 直接用 `setTokensProvider` 安装 provider。
  `create()` 从不抛错。每个 provider 绑定它被创建时当前的语法对象，因为加载一个 embed 会重建
  列出它的每个语法，而 Monaco 保存的行状态只适配旧的。Embed 按 model 加载：对围栏信息串
  （包括位于引用块和列表项内的围栏）与 `lang=` 属性（允许 `=` 两侧有空格）做正则扫描，
  挑出文本点名的、在目录内的 `embeddedLangsLazy`（markdown 26、vue 8、否则 svelte 2），
  并传递性地跟随 embed 自己的惰性 embed（vue -> markdown -> shellscript）；
  它们在第一行上色之后加载，然后重新注册 provider，使 Monaco 重新分词。每次扫描都会报告它加载了
  而当前已安装 provider 缺少的 embed，因此同时扫描的两个 model 各自为它们完成的部分重新注册，
  而不只是第一个。开头的 `---` 会加上 `yaml`。行首的标签、闭合标签、注释，或常见的内联元素
  如 `<img …>`、`<br>` 或 `<kbd>` 会加上 `html`（front matter 和内联 HTML 会无围栏地嵌入它们）。
  围栏内的泛型如 `Array<T>` 或 `Map<K, V>` 不计入。markdown 语法在另一语言下嵌入的围栏名
  （`jsx` 和 `es6` 视为 `javascript`、`golang` 视为 `go`、`h` 视为 `c`）在宿主 `languageId`
  之前先经 `FENCE_EMBEDS` 映射。`html` 目录条目加载 `html-derivative`，
  即 markdown 为内联 HTML 包含的语法。扫描在内容和语言变化时重跑。没有任何东西在引擎或语法上
  suspend：Suspense fallback 至少保持 300 ms。JS 正则引擎只在 `WebAssembly` 不存在的地方运行
  （WebKit 锁定模式）。`.wasm` 抓取或编译失败会让那次高亮器启动失败：文件保持纯文本，
  下一个文件会重试，因此一次瞬时故障（dev server 重启、SPA 回退返回 HTML）既不会让页面钉在
  慢 4.5 倍的引擎上，也不会把损坏的资源藏在它后面。资源在这里被抓取并编译，
  Shiki 只实例化已编译的模块，因为 Shiki 的 `loadWasm` 会把被拒的启动保留整个页面。
  主题加载失败只损失颜色：高亮器保留，下一次换主题会重试。Shiki 的 `setTheme` 对它最近选中的
  名字是空操作，因此每次换色都在另外两个名字之一下加载主题以获得新的颜色映射。
  使用裸的 `shiki/onig.wasm` 资源，因为 `shiki/wasm` 把它内联为 base64 并在主线程上逐字节解码
  （1x CPU 下约 22 ms），且在第一行上色之前。每个交付的宿主都以 `application/wasm` 服务 `.wasm`：
  开发时是 Vite，桌面应用和暂存 CLI 二进制中是 `serveStatic` 里的 `Bun.file`。
  代价：新语言的首次打开要等它的语法 chunk（1x CPU 下约一帧），而引擎会抓取一个 466 kB 的
  `.wasm` 资源；Pierre 较新的 Shiki 自带自己的副本。
- **代码视图在首次打开之前预加载。** 在标签挂载之后才加载 Monaco chunk、初始化 Monaco 并构建引擎，
  且背后是一个至少保持 300 ms 的 fallback，会让首次打开在 1x CPU 下约 420 ms（4x 下 840 ms），
  而热打开是 65 ms。有了应用的空闲预加载和行悬停 warm（`usePreloadResourceView`），
  首次打开在 1x 下约 120 ms（4x 下 430 ms，其余部分是 Oniguruma 在首次分词时编译该语法的扫描器）。
  本包只提供机制；何时调用由宿主决定。代价：每个桌面会话都加载 Monaco chunk 和引擎
  （解码后约 3.9 MB，堆再多约 7 MB），并在引导后于 4x CPU 下付出一次 70 ms 的求值任务。
  预加载仍在进行时的一次点击照旧 suspend。网络断开时失败的预加载会赔掉本会话剩下的时间：
  Chromium 缓存失败的模块抓取，因此之后对该 chunk 的每次 import 都无需请求即被拒绝，
  而静态拆分的 chunk 没有 URL 可做 cache-bust。首次打开随后显示错误边界的 **Reload page**，
  它可以恢复；在 main 上，同样的中断只有当它一直持续到首次打开时才会失败。
  因此宿主把预加载保持到页面在线为止。`e2e/error-boundary.spec.ts` 钉住 Chromium 的行为：
  如果浏览器更新后重新抓取失败的模块，该测试就会变红，而这份代价也就消失了。
- **Monaco 不经 `@monaco-editor/react` 挂载。** 那个包装器会 await `loader.init()`
  （即使 Monaco 已打包也是一个 promise），并在 `editor.create` 之前多花两次 React 提交。
  自有挂载在视图自己的提交中创建编辑器。点击到可见文本，5 轮交错的中位数（Chromium、Mac Studio）：
  1x CPU 下文件打开从 65 降到 34 ms，标签切换从 49 降到 30 ms；4x 下从 145 降到 77 ms，
  从 88 降到 70 ms。点击到高亮不再更晚。代价：创建与首次渲染现在跑在一个任务里，
  因此点击后最长任务在 4x CPU 下变长（冷打开 120 到 145 ms，热的大型打开 0 到 55 ms）。
  `MonacoEditor` chunk 缩小 12.1 kB（3.10 MB 到 3.09 MB；gzip 后 3.5 kB）。
