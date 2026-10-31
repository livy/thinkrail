# Vibecoding 落地页体验

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

在 `https://thinkrail.ai/vibecoding/` 直接提供、并作为同一模块上的更多路由在
`https://thinkrail.ai/agentic-development/` 与 `https://thinkrail.ai/agentic-ide/` 提供的面向特定
受众的体验：其页头导航、hero 与快速开始、桌面下载控件、脚本化聊天、原则与能力、编排动画、工作流
说明、行动号召、元数据，以及路由本地化的呈现。它保持已批准的视觉设计，而父级网站拥有路由组合、
分析、SEO 聚合、部署与跨路由校验。

`index.ts` 是唯一的公开接口。每个路由先获得可抓取的服务端渲染内容，然后水化一个 React island 以
支持有状态交互。

两个 prop 按路由参数化 `Landing`。`heroTitle` 默认为 vibecoding 文案（“Vibe code without losing
control.”）；`/agentic-development/` 传入“Agentic development without losing control.”。
`positioning`（`"control" | "compounding"`，默认 `control`）选择营销文案变体：`/agentic-ide/` 传入
`compounding`，hero 标题为“The agentic IDE that gets better every time you use it.”——这是一次定位
测试（2026-02），把 ThinkRail 框定为越用越好的 IDE（活的规格、可复用的 skill、即将到来的扩展），
而不是“掌控”框架。在 `compounding` 下，hero 副标题、聊天演示脚本（多一段“how do you get better
over time”对话；worktree 那段保留）、原则、能力（另加一张即将推出的“Extensions”卡片）、spec-first
标题/副标题与 CTA 都会替换；其他每个区块共享，文案变体放在各自组件旁边，按 `Positioning` 为键。
分析仅通过父级路径派生的 `content_key`（`agentic-ide` 对比 `vibecoding` / `agentic-development`）
区分这些单元格；该变体不添加事件属性、分析导入或 CTA 位置。完整的模块副本被否决（2026-02）：约 25
个会逐渐漂移的重复文件，与仓库的禁止重复规则冲突。

agentic-development 与 agentic-ide 路由整体复用 vibecoding 的身份：相同的 `siteMetadata` 标题/
描述，相同的 `/vibecoding/` favicon、OG 图片与 wordmark 资源 URL，以及指向
`https://thinkrail.ai/vibecoding/` 的 canonical 与 `og:url`（决策，2026-02：内容近乎相同，因此原始
页面保持为搜索主页面；额外 URL 服务于营销活动/测试/直链，并被排除在 sitemap 之外）。
`validateBuild.ts` 按路由固定这一切，包括每个路由的 hero 标题与每个变体的定位标记——一旦某个 prop
被丢掉，就会静默渲染默认值。

## 边界（Boundary）

- 可以依赖 React 与 Lucide，加上自己的 Tailwind 入口样式表和 `public/vibecoding/` 下签入的资源，
  以及父级不依赖框架的 `src/installLinkShare.ts` 辅助函数（它唯一可导入的父级文件；这条边记录在
  [[module-website]]）。它没有工作区依赖。
- 不得导入父级的 IDE shell 组件或手写样式表、blog 模块、分析、Astro 页面组合、包级构建校验、
  部署配置、另一个工作区，或复制的 UI 套件。
- Tailwind 使用 `source(none)`，只扫描本模块。它的 preflight、字体、token、工具类与动画只输出到
  `/vibecoding/` 引用的样式表中；父级落地页与 blog 绝不能引用该样式表或 React 渲染器/组件 chunk。
- 该路由只呈现桌面应用。一个带类型的下载模型驱动紧凑的 macOS / Windows / Linux 标签页，含一行
  兼容性与克制的操作；macOS 暴露 Apple Silicon，Windows 暴露 x64，Linux 暴露 x64 加 ARM64，不
  猜测浏览器架构。命令行产品、启动器命令、安装命令复制控件、展开说明、Windows shell 状态或分析
  导入都不会进入 vibecoding/agentic 体验。父级网站文档级的事件委托观察其静态下载链接，并从所在
  区块推导 `quick_start` 或 `final_cta`；它还会观察 GitHub 链接（`#top` 处为 `hero`，以及
  `header`）。在检测到手机或平板时，快速开始选择器在操作系统标签页上方增加一个“Send link to my
  computer”操作（水化之后；静态 HTML 保持桌面优先）。它使用父级辅助函数并带 `quick-start` 锚点，
  在剪贴板回退后显示“Link copied”，并且只携带父级委托读取的 `data-share-install-link` 标记。最终
  CTA 不重复选择器：它为单个紧凑下载操作检测受支持的桌面操作系统，并把 Linux、未知/移动端或其他
  平台的访客链接回 Quick start。每次下载都使用 GitHub 无版本号的 `releases/latest/download` 别名。
- 初始 HTML 包含完整的营销信息。JavaScript 增强控件与动画，而不是让内容变得可被发现。
- 每个控件都可用键盘操作。动态效果较重的特效在 `prefers-reduced-motion` 下收敛到稳定的最终状态；
  当请求减少动态效果或文档被隐藏时，canvas 动画暂停。
- 组件使用映射到本模块语义 token 的本地 Tailwind 工具类。它们不携带原始颜色字面量或内联样式对象；
  运行时几何使用 DOM/SVG 属性或样式表拥有的自定义属性。
