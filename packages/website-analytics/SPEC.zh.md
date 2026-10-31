# 共享的网站分析

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

由统一公开网站所消费的浏览器分析策略。它是 PostHog EU 端点、浏览器旅程状态、类型化事件捕获与加载器
的唯一来源，也是 Google Tag Manager 容器与加载器的唯一来源。消费方提供它确切的生产主机名、构建注入
的 PostHog 项目键，以及营销同意来源；本包永不判断哪个产品 URL、路由或同意管理器属于生产。

## 边界（Boundary）

- **公开接口：** `src/index.ts` 导出配置/事件/同意类型以及
  `createWebsiteAnalytics({ productionHostname, postHogProjectKey, marketingConsent })`。返回的门面暴露
  一个纯主机名配置函数、一个幂等的浏览器初始化器、当前/订阅式旅程访问，以及对 `content_viewed`、
  `install_cta_clicked`、`download_started`、`install_command_copied`、`github_clicked`、
  `install_link_share_clicked`、`survey shown`、`survey sent` 和 `survey dismissed` 的类型化捕获。
  CTA 位置是封闭的 `hero | install_section | quick_start | final_cta | blog_post` 集合；
  `github_clicked` 用 `header | footer | terminal | mock_hint | other` 拓宽它，并携带封闭的
  `repo | releases | other` 目标；`install_command_copied` 携带封闭的 `sh | powershell | cmd` shell，
  且只允许 hero/install-section/blog-post 位置；`install_link_share_clicked` 只允许
  hero/quick-start 位置。下载的平台、架构和工件值都是封闭联合。网站为已同意的桌面下载提供
  `bridge_id`；该属性保持可选，使无桥下载也合法。三个 PostHog 问卷事件遵循 PostHog 的自定义问卷
  契约：`survey shown` 和 `survey dismissed` 携带 `$survey_id`、`content_key` 以及可选的 CTA 位置；
  `survey sent` 额外携带 `$survey_submission_id`、`$survey_completed`、`$survey_questions`（`id` 与
  `question`），以及每个已答问题一个 `$survey_response_<question_id>`：字符串，或多选题的数组。网站
  何时发送什么，见 [[module-website]]。
- **无依赖的浏览器模块。** 它使用类型化 DOM API，不含任何运行时包或工作区依赖。它永不导入网站、应用
  分析汇、contracts、server 或 shared。
- **构建期边界。** [[module-website]] 把这份源码编译一次进其静态工件。因此本包的一次变更会触发该
  站点的生产与 PR 预览工作流。
- **不包含站点知识。** 页面结构、路由到内容键的映射、导航、部署 provider、Cookiebot 全局/事件、同意
  UI 以及生产主机名都留在消费方网站模块中。本包只消费一个当前值/订阅适配器。

## 运行时契约（Runtime contract）

工厂的纯配置函数只在观察到的主机名与提供的生产主机名完全相等、且提供的键非空时，返回完整的 PostHog
与 GTM 设置。localhost、IP、provider 预览、同级子域、其他任何主机，以及任何无键构建，都产出空配置。
初始化器在触碰 DOM 之前应用同一个门禁，并在此再次被调用时不追加重复的第三方脚本。

PostHog 是基于团队既有 EU Cloud 项目的渐进增强：

- `array.js` 与事件摄入使用 `p.thinkrail.ai` 上的第一方托管反向代理；`ui_host` 保持
  `eu.posthog.com`，使生成的链接指向真实的 PostHog 应用。
- PostHog 保持无 cookie：`cookieless_mode: "always"`、`respect_dnt: true`、
  `disable_session_recording: true` 和 `person_profiles: "identified_only"`。自动捕获与页面浏览保持
  启用。无 cookie 的服务器哈希模式必须在 PostHog 项目中保持启用；其每日盐使非旅程的跨天唯一访客
  计数是近似值。
- `disable_surveys: true`：网站渲染自己的问卷 UI 并自行捕获问卷事件，因此 posthog-js 永不加载其
  问卷 bundle、不轮询问卷，也不原生渲染项目问卷（例如某个被误标为 popover 的问卷）。网站自己的问卷
  捕获不受影响。
- 异常自动捕获（一项 PostHog 项目设置）只报告我们的脚本能据以行动的错误：`before_send` 仅当某个栈帧
  的 URL 是生产主机名上的 `.js` 文件时才保留 `$exception`，其他事件一律原样放行。*原因（2026-10）：*
  iOS Firefox/Brave 以及应用内或扩展脚本会向每个页面注入代码，并抛出无栈的 `Script error.` 或归因于
  页面 URL 本身的错误（例如 `window.ethereum`），在流量高峰时淹没了 Error Tracking。已接受的代价：
  站点内联小脚本中的错误，以及从另一主机提供的第一方代码中的错误，也会被丢弃。PostHog 项目还在摄入
  时抑制 `Error: Script error.`，与此过滤器无关。
- 已确认的营销授权会读取或创建一个经过校验的随机第一方旅程 ID，并为后续事件注册它。未知同意永不访问
  存储：授权之后它清除内存注册、通知旅程订阅者，并剥离已排队的 `journey_id`/`bridge_id`，同时保留
  存储的 ID 以供再次授权。已知拒绝会移除存储的 ID 和注册。当前/订阅式旅程访问在授权期间只暴露经过
  校验的内存值，永不读取存储，也永不回溯性地丰富早先的捕获。
- 类型化捕获受生产门禁约束，并在 PostHog 初始化之前只保留最近 100 个事件。预览/禁用主机永不排队、
  不持久化旅程状态，也不触碰第三方客户端；加载器或初始化失败会为该文档清空并禁用队列。
- 公开的 `phc_…` 项目键预期出现在生产静态 bundle 中，永不出现在仓库里；由消费方在构建时注入。宿主侧
  的应用分析汇保持独立，并直接发往 PostHog EU，因为它没有浏览器广告拦截方面的顾虑。
- 加载器使用类型化脚本注入，而不是 `posthog-js` 或粘贴进来的压缩引导代码。代理的 `array.js` 建立
  `window.posthog`；初始化在其 load 处理器中进行。

GTM 容器 `GTM-WDW2DZW4` 在同一个生产门禁之后与 PostHog 一同加载。不输出静态 `noscript` iframe，
因为它无法遵守该门禁。GTM 自身不存储任何东西，但容器中配置的任何写 cookie 的标签都会重新开启公开
站点的同意要求；容器所有者负责那次审查。按路由的下游标签以 GTM 主机名 + Page Path 条件表达，永不
通过初始化另一个容器实现。

## 验证（Verification）

包内测试钉住第三方标识符（使用注入的测试键）、URL、隐私选项、第一方异常过滤器、精确主机门禁，以及
禁用主机的行为。消费方站点单独测试它提供的生产主机名，使部署身份不会漂移，而第三方策略保持不变。
