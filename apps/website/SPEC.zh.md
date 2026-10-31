# 项目网站（thinkrail.ai）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

项目在 `thinkrail.ai` 上的公开网站：IDE 外壳（shell）风格的落地页、它的博客，以及 `/vibecoding/`
上面向特定受众的 vibecoding 体验（同一内容也以 `/agentic-development/` 和 `/agentic-ide/`
提供，带按路由定制的 hero 标题与文案变体，以及指回 `/vibecoding/` 的 canonical）。落地页与博客的
创意设定是**站点自身就是一个 IDE**：用 HTML/CSS 忠实复刻 ThinkRail 外壳（标题栏、项目栏、标签条、
文件栏、终端、状态栏），其中央"编辑器"就是本应滚动的页面内容。每个落地页区块都伪装成 `website`
工作区的一个文件（`README.md`、`why.md`、`features/*.md`、`INSTALL.md`、`CONTRIBUTING.md`）。
**文件树是页面导航结构的唯一事实来源**，其选中状态随滚动响应（scroll-spy），就像编辑器在切换文件。
顶部编辑器的**标签条由该文件树导航派生**——不是手工维护的列表——而且是可交互的：当前标签
**通过同一 scroll-spy 跟随当前可见区块**，**点击标签则滚动到对应区块**。标签条对每个不重复的
区块目标**恰好持有一个导航标签**：文件树可以保持层级并在多行重复同一目标，但重复目标会折叠成
单个标签，因此 scroll-spy 每个区块始终只激活一个标签。标签条**可滚动**（`overflow-x: auto`，
滚动条隐藏）——标签在约 1500px 以下就会溢出，每一个都必须保持可达。它的底部分隔线**按标签**绘制，
而不是画在标签条上：标签条边框位于滚动视口之外，当前标签无法在其上覆盖以融入内容区。状态栏是
普通的版权页脚，不是实时的行计数。

`/vibecoding/` 刻意采用另一套视觉语法：[[submodule-website-vibecoding]] 以路由本地的 React island
加路由本地 Tailwind 承载动画化的产品故事与安装体验。它与网站共享同一个源、制品、分析初始化器、
SEO 输出和部署——但不共享 IDE 外壳的表现层。

不属于产品的一部分：应用内没有任何东西依赖公开网站，网站也从不打进二进制。

## 边界（Boundary）

- **独立部署的叶子模块。** 它唯一的工作区依赖是 [[module-website-analytics]]；它绝不能
  import contracts、server、shared 或 web。它从不接入产品宿主的 wire；仅
  [[submodule-website-attribution]] 拥有一个独立的、有界的 HTTP 声明协议和 D1 状态。
- **一个静态 Astro 制品加同项目的 claim 函数，外带一处路由本地的框架例外。**
  落地页和博客保留原生 TypeScript 加手写 CSS：这些路由上不出现 React island，
  也不引入 Tailwind 样式表或运行时。仅 [[submodule-website-attribution]] 增加 Pages Functions
  和 D1；只有 `/vibecoding/`、`/agentic-development/` 和 `/agentic-ide/` 可以使用共享的 React
  island 和 Tailwind v4。Astro 的 React 集成和 Tailwind Vite 插件是构建级工具；
  构建校验要求每个被允许的路由输出中各有一个 island，并拒绝在固定的落地页、博客、
  文章和 claim 输出中出现 island。浏览器分析工作区模块被编译进静态输出。
  `@fontsource-variable/*` 包是构建期的资源来源，其 woff2 文件被输出到 `dist/`。
  它们与 `apps/web` 共享，来自根 `workspaces.catalog`——两个应用共用一个 pin，
  从而让网站的字体与应用保持完全一致。
  - *为何选 Astro（决策，2026-08）：* 博客加计划中的文档触发了"自造 SSG"的警戒线
    （RSS、OG、类型化 frontmatter、内容开发体验）。Astro 底层就是 Vite——落地页可原样迁移，
    `bun test` 套件不变——而 React 19 island 只在某个路由确实需要时才可用。已否决：
    Next.js 静态导出（把 React 运行时发给静态页面，非 Vite 文化）、
    VitePress（Vue）、Eleventy（文档特性要自己攒——又落回自造陷阱）、Hugo/Zola
    （非 TS 工具链，丢失 token/测试集成）。此前的自造流水线
    （`scripts/build-blog.ts` + HTML 字符串模板）已删除。
  - *Lint 注意事项：* Biome 只解析 `.astro` 的 frontmatter，因此 `biome.json`
    对 `*.astro` 禁用了 `noUnusedVariables`/`noUnusedImports`
    （模板中的使用对它不可见——任何报错都会是误报）。`astro check` 代替它覆盖模板。

父模块拥有路由组合的边；vibecoding 叶子模块没有兄弟依赖：

```text
src/pages/{vibecoding,agentic-development,agentic-ide}/index.astro ──▶ src/vibecoding (through index.ts)
src/pages/{vibecoding,agentic-development,agentic-ide}/index.astro ──▶ src/components/Analytics.astro
landing + blog shells                                              ──▶ src/components/Analytics.astro
src/main.ts, src/vibecoding                                        ──▶ src/installLinkShare.ts
```

`src/installLinkShare.ts` 是两个路由族共用的、唯一不依赖框架的手机"发送链接"辅助模块：
手机/平板检测（Android、iPhone/iPad/iPod，或触屏 Mac）、分享 URL，以及先走原生分享再退到剪贴板的投递。
它不 import 任何东西，因此 vibecoding island 可以复用它而不必触达 IDE 外壳。

- **字体自托管；站点不发起任何外部字体请求。** 包与字体栈从应用的 `typography.json` 复制而来，
  而不是 import——并且 `src/fonts.test.ts` 在测试时读取该 JSON 并在发生漂移时失败，这正是复制安全的原因。
  `--font-display` 对应应用的 `brand` 字体族——**Orbitron**（`@fontsource-variable/orbitron`，自托管），
  品牌元素（wordmark + hero 标题）使用这一独立展示字体，回退到 `--font-sans`；
  区块标题和标语使用界面字体（`--font-sans`），与应用一致。
- **品牌取值是复制的，不是 import 的。** 主题配色在撰写时从
  `apps/web/src/themes/bundled/*.theme.json`（dark = 默认、darcula、light、gruvbox）搬到站点自己的
  CSS 自定义属性、置于 `[data-theme]` 之下；站点在构建期从不触达 `apps/web`
  （应用的 token 假设主题引擎在运行时换色）。
- **IDE 外壳的页面外框（chrome）在组件中单一来源。** `src/components/BaseHead.astro` 是落地页和博客
  共用的唯一 head：charset/viewport、favicon、它们的全局样式表（内含字体），以及绘制前主题守卫。
  `src/components/Analytics.astro` 不含表现，被每个路由族各包含一次。
  `src/components/IdeShell.astro` 是落地页和博客渲染所经的唯一 IDE 外壳——图标 sprite、跳转链接、
  标题栏（工作区名参数化：落地页为 `website`，博客页为 `blog`，各自带匹配的
  `workspace/<name> · from main` 分支标签，以及 GitHub + Discord 图标链接）、左侧项目栏、
  右侧文件栏 + 终端、状态栏、下载后的感谢对话框（`DownloadThanks.astro`），
  以及 `main.ts` 脚本 import；中央列是它的默认 slot，右侧栏的 Files 行是它的 `filetree` slot。
  左栏以 `website` 工作区的子行承载站点真实导航——**Landing**（`/`）和 **Blog**（`/blog/`，
  在索引页和每篇文章上都被选中）作为链接，**Docs** 禁用并带 `Coming soon` 芯片
  （无 tooltip——标签本身就承载状态）——其余模拟行保持 `data-mock-hint` 处理；
  被选中的行沿用与右栏活动文件行相同的 accent 着色语法（一条分组 CSS 规则）。
  `src/components/Copyright.astro` 是唯一的版权行（外壳状态栏）。深/浅色主题模型位于
  `src/theme.ts`（`localStorage` 中的显式选择 → `prefers-color-scheme` 回退，实时跟随系统，
  遗留的 `darcula`/`gruvbox` 取值归一为 dark）；BaseHead 中的内联 FOUC 守卫是它声明式的孪生体——
  行为改动要同时更新两者。
- **有对比度下限的颜色使用 `:root` token，绝不使用区域继承的 `--accent`。**
  区域可以重指向 `--accent`（浅色主题的 kicker 药丸就是这么做的；hero 曾经也是），
  因此读取它的后代可能继承到为别的表面选定的值。`--link` 和 `--focus-ring`
  在 `:root` 上各声明一次为 `var(--accent)`，解析到*根* accent 上——
  按主题生效，但不受区域覆盖的影响。
- **主按钮（以及 hero 的品牌强调色）按主题而定，不是品牌常量**（该决策在 2026-08 评审时反转——
  浅色主题此前沿用了深色主题的亮绿）：`--control-primary-*` token 对应应用的语义映射
  （`bg = accent`、`hovered = accentHover`、`text = onAccent`，见
  `apps/web/src/styles/colors.json`），通过上面的根 token 机制在 `:root` 上声明，
  其中 `--accent-hover` 按主题从应用调色板复制。hero 标题与其他所有 accent 着色的元素一样跟随主题 accent；
  `palette.test.ts` 钉住悬停填充时标签的对比度。
- 所有营销文案都是静态 DOM 文本；JS 只做*增强*（scroll-spy、派生的编辑器标签条及其
  scroll-spy/点击导航、终端打字、聊天流式重放、主题切换器、复制按钮、star 计数、安装平台选择）。
  页面必须在禁用 JS 时也能完整阅读（编辑器标签是在本就可通过滚动和文件树到达的内容之上、
  由 JS 构建的导航辅助），并且在 `prefers-reduced-motion` 下跳过动画。
- **由 `main.ts` 拥有的增强行为**（代码本身不含论证——这里就是论证）：
  - *终端重放*：Install 区块的命令标签把启动器命令保留在静态标记中；该命令是终端的
    唯一事实来源，终端订阅它并在简短的安装转录之前把它打出来。当选中的命令改变时，
    一个世代计数器会使进行中的序列失效；点击*已完成*的终端（或它只有在完成时才出现的、
    可通过键盘到达的 `Replay logo` 按钮）会重放 ASCII logo + 一个 GitHub CTA——绝不重放安装过程。
    logo 横幅的字符从不改动；只有它的字号会适配栏宽（尺寸变化时重新适配）。
  - *Worktree 提示*（左栏）：在标记中初始可见（禁用 JS 时也能读到）；一个内联脚本在绘制前隐藏它，
    `main.ts` 在 5s 后显示它——每个会话一次（`sessionStorage`），在 Understood 之后不再显示
    （`localStorage`），如果 `main.ts` 加载失败则内联脚本有 10s 的兜底。
    存储访问被完全加固（获取存储对象本身就可能抛异常）。
  - *模拟 callout 的 tooltip*：`data-mock-hint` 区域（右栏标签、左栏剩余的模拟行）
    打开一个点击后常驻的 callout 加 GitHub CTA——点击外部/Escape 关闭，Escape 恢复焦点。
    无障碍契约：触发器变成带 `aria-label`（取自 `data-mock-label`）的 `role="button"`，
    打开时带 `aria-expanded` + `aria-describedby`——不用 `aria-haspopup`
    （`tooltip` 不是允许的 popup 值）。`.rail-tabs` 的 callout 锚定到面板的实时边缘；
    其余放在触发器旁边，被视口钳制，并在尺寸变化时重定位。
- hero 的安装控件是一行中的三个动作：**Download**、**View on GitHub**（已有的 star 计数填在标签旁）
  和 **Join our Discord**。Download 为检测到的桌面平台启动文件：macOS 上是 Apple Silicon DMG，
  Windows 上是 Windows x64 ZIP，Linux 上是一个归档。仅当浏览器提示明确说明 ARM 时，
  Linux 才使用 ARM64 归档；否则按钮下载 x64 归档。当桌面平台未知时（包括 JS 关闭时），
  Download 滚动到 `#install` 而不是选择文件。**Other platforms & CLI** 位于该按钮下方，
  同样滚动到 `#install`。它不打开菜单。JS 在检测到桌面平台时设置标签与文件 URL，
  并在 Install 区块中标记那一行。不带版本号的 GitHub `releases/latest/download` URL
  指向 [[module-ci-release]] 拥有的公开别名；网站不携带任何发布版本、更新器载荷名或私有制品知识。
  浏览器提示只决定该选哪个文件和初始命令标签；它们从不隐藏备选项，从不把 `MacIntel`
  当作 Intel 构建，也从不声称检测到模糊的移动平台。
- `INSTALL.md` 区块就是下载表面：一个紧凑的平台行矩阵（macOS Apple Silicon DMG、Windows x64 ZIP、
  Linux x64 与 ARM64 归档），然后是始终可见的 **Browser UI via command line**。
  这些命令使用与博客选择器相同的标签外框，共三个标签——
  **macOS / Linux / WSL**、**Windows CMD**、**Windows PowerShell**。PowerShell 运行
  `irm …/install.ps1 | iex`，Command Prompt 启动 `powershell -c "irm …/install.ps1 | iex"`，
  macOS、Linux 和 WSL 共享 `install.sh`。每条命令都留在静态 DOM 中；JS 把该回退内容变成标签。
  Nightly/版本示例放在发布文档里，而不是营销页面上。
- **手机访客会在 hero 动作上方获得一个 "Send link to my computer" 动作。** 它在静态 DOM 中但
  `hidden`，`main.ts` 仅在检测到手机或平板时显示它（下载链接仍在其下方可见；不隐藏任何平台）。
  激活时用当前页面 URL 打开原生分享表，其 query 被替换为
  `utm_source=send_to_computer&utm_medium=share`，fragment 被替换为 hero 锚点，
  这样之后的桌面访问会记录一次带标记的首次触点。若没有 Web Share，或分享因除访客主动取消之外的
  任何原因失败，则复制链接并短暂显示 "Link copied"。该控件携带 `data-share-install-link`
  供分析委派使用，别无其他。

## 分析与同意（Analytics and consent）

`src/analytics.ts` 是 [[module-website-analytics]] 之上的站点本地门面。它提供确切的生产主机名
`thinkrail.ai`、构建期的 `PUBLIC_POSTHOG_PROJECT_KEY`（从不提交；生产 `site.yml` 从 `website`
环境的 secret 传入它，缺失时在构建前就失败，而预览和本地构建没有它并保持静默），
并把 Cookiebot 能感知响应状态的 Marketing 状态以及 consent-ready/accept/decline 事件适配到
共享的当前值/订阅契约；共享模块拥有 PostHog 端点与 GTM 标识符、旅程持久化、类型化采集、
隐私配置和脚本加载器。未知或被拒绝的 Marketing 状态保持无 cookie 且不携带 journey ID。
Localhost、`astro dev`、每个 `pages.dev` 部署、`jetbrains.github.io` 地址以及兄弟子域都不发送任何内容。

`src/components/Analytics.astro` 初始化该门面，然后对每个文档初始化一次幂等的站点事件委派；
它仍是 IDE 外壳路由与 vibecoding 路由的分析组合点。
内容键由 `window.location.pathname` 派生：`/` → `landing`，`/blog/` → `blog/index`，
其他只含小写安全字符的路径变成去掉外侧斜杠的路径（因此 `/blog/<slug>/` → `blog/<slug>`）。
键只允许小写字母、数字、连字符和路径分隔符，长度上限为 105 个字符，
足以容纳 `blog/` 加上最长 100 字符的文章 slug；无效路径不发出任何 content 或 action 事件。
query 与 fragment 字符串不在 `pathname` 中；带点的 `index.html` 路径不是路由别名。
新页面和文章无需改动分析代码。只有用 Analytics 组合的页面才加载它：手写的 `src/pages/404.astro`
是一个独立的、无分析的页面，用于未知 URL 的响应，而独立的 `/attribution/claim/` 页面同样保持无分析。
每个有效文档发出一个显式的 `content_viewed`，携带其派生的键。构建校验递归检查每个输出的 `.html`
文档恰好有一个 PostHog 和一个 GTM 加载器，唯二的例外是显式无分析的 `404.html` 和
`/attribution/claim/` 输出，它们必须既不含加载器也不含浏览器分析。按路由的布局、制品和 SEO
检查仍然聚焦于各自所属的路由。

初始化时，仅当已存在同意的旅程时才记录当前页面触点；初始化之后出现的旅程不会重放页面 URL
或 referrer，旅程被移除则清空归因上下文。触点只携带有限的 UTM 取值、referrer 类别、时间戳和策略版本；
它们不传递页面键。网站事件保留其 `content_key` 和旅程身份，因此 PostHog 可以把这些事件与应用的
`acquisition_linked.journey_id` 关联，用于落地页/末页分析。文档级的 `click`、中键 `auxclick`
和 disclosure 的 `toggle` 委派在不改变静态安装控件导航或 no-JS 行为的前提下为其加埋点。
在每个初始化后的安装 CTA、下载或 CLI 事件之前，都会尝试当前导航触点，使页面上授予的同意只影响
后续动作；一个页面在同一个已同意的旅程中最多记录一次该触点，因此其后的动作绝不会丢掉它下载时
建立的桥接（见 [[submodule-website-attribution]]）。只有四个确切的稳定 GitHub 桌面别名算作下载。
它们的 CTA 位置取自所在的 hero、install 区块、quick start、final CTA 或博客文章；一次被识别的
激活发出 `install_cta_clicked`（`desktop`），随后是 `download_started`。当存在已同意的旅程上下文时，
激活会在 `download_started` 之前创建并保存一个规范的、按下载独立的 bridge ID，该事件携带同一 ID。
之后的桌面下载只替换已保存的最新 bridge ID；没有同意的下载仍然没有 bridge。
在这四个下载之一上的主按钮点击还会打开感谢对话框，并且不会取消文件。该对话框记录 PostHog 调研
`Thanks for downloading ThinkRail`：打开时记录 `survey shown`，在作答前关闭则记录
`survey dismissed`，每次 Continue 或 Finish 记录 `survey sent`。一个
`$survey_submission_id` 把这些发送串起来，每次发送重复截至当时已收集的答案，
`$survey_completed` 仅在 Finish 时为真，因此最远的事件就说明该回应走到了哪一步。
选项按标签发送，自定义答案按输入的文本发送；多选题答案是数组。欢迎页说明继续即接受隐私政策并给出链接。
该对话框只在具有有效内容键的页面上初始化，因此它的事件总是携带一个内容键。
调研事件不尝试归因触点：它们是反馈而非获客动作，而打开对话框的那次下载已经记录了页面的触点和 bridge。
关闭对话框会把它重置回欢迎页。下一次打开是一次新的展示，带新的 submission id 和新的选项顺序；
已发送的答案留在记录它们的那个 submission 上。完成调研会被记在
`localStorage` 的 `thinkrail_download_survey_completed` 下，该值只保存调研 id，别无其他；
它是对话框的 UX 状态，不是分析状态，因此不受同意门控。之后的下载仍会打开对话框，
但调研卡片会显示访客已回答，并提供 **Answer again**：只有他们真正作答时才发送 `survey shown`，
因此关闭这样的对话框不发送任何内容，而新的调研 id 会再次提供该调研。选项顺序按每个回应打乱，
自定义答案除外，它始终在最后。必答步骤在答案有效前保持 Continue 禁用。首尝试说明和
邮箱是可选填的。已填写的邮箱必须在浏览器邮箱字段校验通过后才能启用 Finish；空邮箱不需要。
当 Finish 被禁用时，按钮同一行上有一条灰调提示，内容为
"— please enter a valid email or leave it blank"。打开博客文章选择器的命令行 disclosure 只发出
`install_cta_clicked`（`cli`）；关闭它则不发出，落地页始终可见的命令标签也不发出任何事件。
那次打开属于浏览，不是安装意图：在 `[data-copy]` 控件上的主按钮点击，若其取值是已知的安装命令，
则发出 `install_command_copied`，携带其 `sh`/`powershell`/`cmd` shell 以及 hero、install 区块
或博客文章的位置（其他复制取值不发出事件；手动选中并复制文本不被观测）。
任何其他指向 `JetBrains/thinkrail` 仓库的链接（确切仓库、带 `#`/`?` 后缀，或子路径；
绝不是兄弟仓库或桌面别名）发出 `github_clicked`，携带 `repo`/`releases`/`other` 目标
和取自所在博客文章、final CTA、quick start、install 区块、hero（`#readme` 或 vibecoding 的
`#top`）、header、footer、terminal 或模拟 callout tooltip 的位置——若都不包含则为 `other`。
GitHub 点击是独立信号：GTM 报表只把 `download_started` 和 `install_command_copied`
计为安装意图，因为仓库访问把加星和浏览与安装混在一起。
在 hero 或 quick start 中 `[data-share-install-link]` 控件上的主按钮点击发出
`install_link_share_clicked`；它记录激活，不记录分享表是否完成。
这些事件中的每一个都像既有动作一样先尝试实时导航触点。任何页面或子模块都不携带分析 import、
供应商配置或另一个加载器；控件只暴露委派所读取的 `data-copy` / `data-share-install-link` 标记。
唯一的例外是 `src/downloadThanks.ts`：它拥有调研的步骤状态，因此它自己通过站点门面
`src/analytics.ts` 采集三个调研事件，且不添加任何供应商配置或加载器。

既有 GTM 容器仍是 Cookiebot 的控制面；按路由的下游标签使用 `thinkrail.ai` 主机名条件加
Page Path，绝不使用另一个 GTM 容器。共享同一个确切 apex 源意味着 Cookiebot 扫描和浏览器同意状态
适用于全部三个路由族。

站点测试钉住生产主机身份、禁用主机、Cookiebot 适配器、每个路由键与稳定桌面别名、
事件派生以及初始化器的幂等性。共享包测试 PostHog/GTM、旅程和采集契约。
共享契约刻意没有 `posthog-js` 依赖、粘贴式 bootstrap 或静态 GTM `noscript` iframe。

未被索引的 `/attribution/claim/` 路由属于 [[submodule-website-attribution]]；
其无分析页面只在当前 Cookiebot `CookieConsent` cookie 记录了 Marketing 授权或 `-1` 时，
在加载时静默尝试 claim 绑定，然后导航到 `/blog/`。该模块拥有 claim 协议与框架保护。

## 部署（Deploy）

Cloudflare Pages 项目 `thinkrail-website` 拥有静态制品和 claim Functions 的生产与预览。
固定版本的 Wrangler `4.124.0` 从 `apps/website` 部署，因此它的配置、
`functions/` 和 `dist` 制品形成同一次部署。顶层 `ATTRIBUTION_DB` 服务于生产和本地 Pages/D1 工作流；
生产在部署前应用已提交的归因迁移。构建会编译 Functions，本地 Pages/D1 冒烟测试从已提交的配置启动
Pages，应用真实迁移，并演练 create、bind 和 redeem。`.github/workflows/site.yml` 在触及本模块、
[[module-website-analytics]]、根包清单或 lockfile 的 push 上（外加手动触发）运行
`bun run --filter @thinkrail/website build`（`astro check`、Functions 的类型检查与构建、
`astro build` 以及制品校验）。它在成功之前校验供应商 URL。`thinkrail.ai` 是项目的自定义 apex 域；
供应商 URL 是部署探针，不是产品身份。

生产并发从不取消进行中的发布。Cloudflare 将成功的生产部署保留为回滚目标，
因此回滚选择前一个 `main` 部署，而不是再跑第二条托管流水线。

### PR 预览部署（PR preview deploys）

`.github/workflows/site-preview.yml` 运行同样的构建命令并上传到 `thinkrail-website` 中的
`pr-<number>` 分支。Wrangler 的 `env.preview` 覆盖让预览部署没有 D1 绑定；
生产源守卫独立地在访问 D1 之前继续对预览归因请求返回 not-found。确定性别名
`https://pr-<number>.thinkrail-website.pages.dev` 以一条 sticky PR 评论和一个
`Website preview` 提交状态的形式呈现，覆盖 `/`、`/blog/`、`/vibecoding/`、
`/agentic-development/` 和 `/agentic-ide/`。它会等所有这些路由都能响应后才发布该 URL。

只有同仓库的 PR 才能获得预览；fork PR 被跳过，因为 Cloudflare 凭据从不跨越仓库边界。
预览 URL 是公开的，且在 PR 打开期间不发送分析。一个独立的关闭工作流识别 sticky 预览标记，
删除该 PR 分支的所有部署，并把它的预览元数据标记为已退役；没有该标记的 PR 是空操作。
共享的并发组防止清理与进行中的发布竞态。更新的 push 只取消该 PR 被取代的那个预览。

一次性初始化创建 `thinkrail-website`，生产分支为 `main`，使用
`CLOUDFLARE_API_TOKEN`（Pages:Edit 加 D1:Edit，用于已提交的归因迁移）和
`CLOUDFLARE_ACCOUNT_ID` 仓库 secret，然后在验证供应商托管的 main 部署之后挂上 `thinkrail.ai`
自定义域。token 权限更新必须先于首个启用 claim 的生产部署落地。

### 托管与已退役主机名（Hosting and retired hostname）

Cloudflare 管理的 DNS 把 `thinkrail.ai` 路由到已验证的 `main` 部署。生产回滚通过 Pages
选择此前一个成功的 `main` 部署；遗留托管不属于稳态。

`vibecoding.thinkrail.ai` 是一条 Cloudflare Single Redirect：主机名匹配、动态目标
`concat("https://thinkrail.ai/vibecoding", http.request.uri.path)`、状态 301、启用 query 保留。
它那个无源站的代理记录 `192.0.2.1` 让该重定向在没有部署时也可解析。

## 博客（Blog）

`/blog` 子站是 `content/blog/` 中 Markdown 文章之上的类型化 Astro 内容集合
（每篇文章：一个含 `index.md` 加可选 `images/` 的文件夹），由 `src/pages/blog/` 通过
`src/layouts/BlogLayout.astro` 渲染。携带实时安装指引的已发布文章遵循当前桌面优先的层级，
而不是冻结过时的仅 CLI 说明。

- **Schema 就是门禁**（`src/content.config.ts`，zod）：必填 `title`/`slug`/`date`/`author`，
  可选 `excerpt`/`draft`/`tags`。格式错误或保留的 slug、缺失字段、未知的
  `author` 键，或两篇文章共用同一个 slug **都会让构建失败**——没有静默的绿色部署。
  草稿在 `astro dev` 中渲染（作者预览、热重载），并被排除在生产构建之外
  （`src/blogCollection.ts`，索引、文章页和 RSS 三者共享的唯一查询——最新优先、过滤草稿）。
- **URL 是目录风格的**（`/blog/<slug>/`），这是博客尚未发布时决定的，因此没有破坏任何东西；
  RSS 位于 `/blog/rss.xml`（`@astrojs/rss`）。文章页携带 meta description（即摘要）、
  canonical、OG/article 标签。
- **代码块**：Astro 内置的 Shiki，`github-light`/`github-dark` 双主题以
  `--shiki-light`/`--shiki-dark` CSS 变量输出（`defaultColor: false`），因此由站点的
  `[data-theme]` 开关——而不是媒体查询——来决定配色。样式在 `src/styles.css` 的 `.astro-code` 下。
- **YouTube 嵌入必须无 cookie**：作者写普通 iframe；Sätteri 的 HAST 插件
  `src/youTubeEmbeds.ts`（Astro 7 的原生 Markdown 处理器——unified/rehype 流水线是我们不携带的
  独立遗留包）把 `youtube.com/embed` 重写为 `youtube-nocookie.com`，并在缺失时补上
  `title` + `loading="lazy"`。这保持了无同意横幅的立场不变。
- **外框：博客页生活在 IDE 外壳内**，而不是独立的页面框架（决策，2026-08——
  此前独立的博客页头/页脚已删除）。`BlogLayout.astro` 通过 `IdeShell.astro` 渲染：
  内容与落地页区块坐在同一个可滚动编辑器窗格中，位于 `.blog-main` 容器内并与其共享精确几何
  （一条分组的 `.file-section, .blog-main` CSS 规则——刻意*不*使用 `.file-section` 类，
  因为它的排版级联会与博客内容样式打架）。打开一篇文章会在同一中央区域替换索引。
  **博客页没有标签条**（落地页保留其区块标签；页面级的 Landing/Blog 标签目前被明确否决）。
  右栏的 Files 列表显示已发布文章（当前文章为活动状态）；终端保留其静态安装转录——
  `main.ts` 的守卫让每个仅落地页的增强都失效（没有区块 → 没有 scroll-spy；
  只有落地页的命令标签驱动终端重放）。
- **每篇文章都以安装选择器结束。** `src/components/InstallPicker.astro` 是博客选择器的标记
  （OS 标签、桌面下载、带 Windows shell 标签的 **Browser UI via command line** disclosure）。
  `src/pages/blog/[slug].astro` 在文章正文之后的 `.blog-post-install` 区块中渲染它。
  它位于 `.blog-post` 之内，因此其下载、复制和 disclosure 打开都报告 `blog_post`，
  且位于 `.blog-post-content` 之外，因此正文级联永不重设它的样式。`main.ts` 驱动该选择器；
  文章保留自己的正文安装指引，`validateBuild.ts` 要求每篇文章都带该选择器。
  文章卡片是整卡链接，只在边框上提示悬停（无下划线、无位移），并与落地页特性卡片共享表面，
  这使 `--elevated` 标签芯片在它们上面保持可见。主题：BaseHead（主题守卫、字体、分析）+
  经外壳 `main.ts` 的 `src/theme.ts` 开关——与落地页行为相同，只有一份实现。
- **作者是共享实体，不是每篇文章的字符串。** 第二个集合（`authors`）通过 Astro 的
  `file()` 加载器载入 `content/authors.json`——一个按作者键索引的对象，schema 为
  必填 `name` + 可选 `url`（存在 ⇒ 署名行渲染为链接）。文章携带
  `author: reference("authors")`，**必填**，因此引用完整性由框架检查：
  拼错的键就是构建失败，作者改名或新增链接会一次性更新所有文章。
  解析集中为 `src/blogCollection.ts` 中的 `postAuthor()`；没有任何页面内联该查找。
  Avatar/角色字段被刻意省略——没有渲染器的 schema 字段是死表面；两者都可以以后叠加。
  - *署名位置（决策，2026-08）：* **仅文章页头**，作为
    `.blog-post-meta` 行的第一项（`author · date · reading time`），共享日期的提示色、字号，
    以及一条分组的 `·` 分隔符规则。索引卡片、RSS 条目、OG/`article:author` meta、
    文章末尾作者卡片，以及 `/blog/authors/<key>/` 页面都各自被考虑过，但目前都否决了——
    共享实体让它们每一个都成为低成本的后续工作。
- **作者指南**：`content/blog/BLOG.md` 记录 frontmatter schema、作者档案、
  Markdown 特性、嵌入，以及本地预览循环（`bun run dev` 会热重载文章）。
  它们。主题：BaseHead（主题守卫 + 字体）+ 经外壳 `main.ts` 的 `src/theme.ts` 开关
  ——与落地页行为相同，只有一份实现；Analytics 仍是独立的共享 head 初始化器。
- **作者指南**：`content/blog/BLOG.md` 记录 frontmatter schema、Markdown 特性、
  嵌入，以及本地预览循环（`bun run dev` 会热重载文章）。
- **部署**：通过同一个 `site.yml` 工作流与主站一同部署；对
  `apps/website/content/blog/**` 的改动会触发重建。

## 发现与资源（Discovery and assets）

一份 apex 的 `robots.txt` 允许抓取公开站点，并把爬虫指向 Astro 生成的 sitemap。sitemap 派生自
同一个静态路由构建，因此覆盖 `/`、博客索引与已发布文章，以及 `/vibecoding/`；
`/agentic-development/` 和 `/agentic-ide/` 被从 sitemap 中过滤掉，因为它们 canonical 到
`/vibecoding/`（见 [[submodule-website-vibecoding]]）。预览保持生产 canonical URL，
不会变成第二个搜索身份。

`public/_redirects` 把 `/discord` 映射到社区服务器邀请，因此 `thinkrail.ai/discord` 是站点、
博客文章、README 和应用共用的唯一 Discord 链接——轮换邀请只是一次一行的站点部署，
绝不是一次应用发布。`validateBuild` 会让丢掉它的构建失败。Discord 点击目前不发出分析事件。

`public/favicon.svg` 是 IDE 外壳落地页/博客的标签页图标：品牌主色绿上的圆角方块，用黑色承载
**页头 wordmark 的 TR 标记**（同一 `viewBox` + 路径，不是重画），居中并带均衡内边距，
以保证在标签尺寸下可读。它通过 `index.html` 中的 `<link rel="icon" type="image/svg+xml">` 接线；
没有 `.ico` 回退。

`public/og.png` 是站点自身 hero 在 IDE 外壳内的截图（深色主题、选中 Landing、终端安装转录已完成、
worktree 提示可见）。`features/agent-chat.md` 区块中的转录来自本仓库上工作时在应用内捕获的一次真实
`pi` 会话。重新捕获：对构建后的站点跑 `astro preview`，在 1200×630 视口、deviceScaleFactor 2
（深色主题）下查看，等终端重放 + 侧栏提示完成（约 7s），截取视口，再把 2400×1260 的截图
降采样到 1200×630。

`public/vibecoding/` 拥有该路由独有的 favicon、1200×630 Open Graph 图像和渐变 wordmark。
它们的公开 URL 仍在 `/vibecoding/` 下，而 Vite 生成的 chunk 留在站点级的 `/_astro/` 根下，
因为一个 Astro base 服务所有路由族。
