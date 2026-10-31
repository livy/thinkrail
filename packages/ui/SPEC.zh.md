# 共享的宿主 UI 基础组件

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

由应用与扩展 web 两半共享的、自有的 shadcn/Radix 基础组件。仅限展示层：功能状态、主题选择与 token
生成仍留在应用中。

## 边界（Boundary）

基础组件以逐文件的 `@thinkrail/ui/<primitive>` 子路径暴露，不使用根组件 barrel；这保持了懒加载 chunk
的独立性并保留 shadcn 约定。`./utils` 暴露 `cn`；`./theme` 只暴露 `onThemeSwap`，即对完成态的
`data-theme` 信号的一次 DOM MutationObserver 订阅，带显式清理。它对主题目录或偏好一无所知。

工作区的导入只能触及 `contracts` 与 `@thinkrail/locales/react`（共享的 i18next 实例，用于基础组件
自有的字符串，如对话框的屏幕阅读器 Close 与 toast 的 Dismiss 标签）。外部依赖是 React、Radix、cmdk、
react-resizable-panels、Remix Icon、class-variance-authority、clsx 与 tailwind-merge。禁止宿主内部
实现、pi、应用 store、传输层与主题运行时。token 仍是 `apps/web/src/styles` 下生成的全局 CSS 变量；
工具类遵循 [[web-color]]、[[web-spacing]] 与 [[web-typography]]。应用的 Tailwind 源与采纳门禁包含本包
与扩展 web 两半。

## 按钮系统（Button system）

`button` 是操作按钮的唯一实现：`Button`（文本，可选 `iconLeft` / `iconRight`）、`IconButton`（仅图标
——`label` 必填，并同时成为 `aria-label` 与共享的 `IconTooltip`），以及 `buttonVariants`（用于必须
看起来像按钮的非按钮元素，如链接）——它返回组件渲染的同一份合并后 class 列表，compound 已解析——并配
`buttonDataAttributes`，让这类元素拿到组件渲染的同一份 `data-slot` / `data-variant` / `data-size` /
`data-icon-only` / `data-selected` 约定（面板工具轨栏、Monaco 评论 widget），以及
`withIconClass(Icon, classes)`（用于着色或动画字形，如成功对勾、进行中脉冲）——在模块作用域创建一次，
绝不在渲染期间创建，从而图标保持其身份。两个组件都不接受原生 `title`：说明用 `tooltip`
（在 `IconButton` 上它覆盖 label 的 tooltip 文本），经由同一个 `IconTooltip` 渲染，因此它在禁用按钮上
也会显示——那正是最需要给出不可用原因的地方。处于 `disabled` 时，按钮会自动把 tooltip 触发器移到
`wrapTrigger` span 上，使 tooltip 在禁用按钮的 `pointer-events: none` 下仍然存活；起决定作用的是值
而非 prop 是否存在，因为 `DropdownMenuTrigger asChild` 这样的 Radix 触发器会向每个子元素注入
`disabled={false}`，而包装器会夺走文本按钮的 `min-w-0` 截断。`disabled` 翻转时按钮元素会在 span 内
重新挂载；显式 `wrapTrigger` 会永久保留该 span（并隔离第二个 Radix 触发器），`wrapTrigger={false}`
则退出包装。应用中、渲染器包与扩展 web 两半里的每个普通操作都经由它渲染；带有按钮皮肤的裸 `<button>`
会被 `apps/web/src/styles/buttonUsage.test.ts` 拒绝，除非该测试记录在案的专用清单指名了它。

- **变体**（恰好五种）：`primary`（强调填充——某个界面的主要操作）、`soft-primary`（强调色淡彩 +
  强调色文本；也覆盖所有先前的透明“绿色 ghost”）、`outline`（中性描边）、`ghost`（无表面、弱化文本、
  中性 hover 填充——`IconButton` 的默认）、`destructive`（危险填充；确认不可逆操作）。调用点选择与
  操作既有外观相符的变体；没有 `soft-default`，调用点也不做颜色覆盖。`tone`（`warning` | `danger`）
  是唯一获准的标签颜色：在 `ghost` 与 `outline` 上它在所有状态（静息、hover、focus-visible、按下、
  选中——禁用仍然优先）都保持该反馈色，因为调用点设置的颜色会在 focus 与按下态输给 ghost 自身的状态
  文本，而那时控件恰好正在被使用。
- **尺寸**（恰好三种；border-box，对所有变体与状态一致，圆角 `rounded-sm`，字型 `tr-text-button`——
  即字号 14px、字重 470 的界面字形，比普通 UI 文本高一档）：`s` 高 20px，文本内边距 4px，图标 14px，
  图标↔文本间距 4px，纯图标方形 20px；`m` 24 / 8 / 16 / 4 / 24；`l` 32 / 12 / 20 / 6 / 32
  （`Button` 的默认；`IconButton` 默认 `m`）。在 `s` 上间距等于侧内边距：12px 图标看上去比它旁边的
  14px 标签更轻，于是图标被提到 14px、间距提到 4px，让这一对保持可读。图标字重只跟随图标尺寸——
  Remix Icon 只提供 Line/Fill 而非各档描边粗细，因此没有按尺寸对应的字重可匹配，也没有用 CSS 去模拟。
  图标绝不缩小；文本按钮按内容定尺寸，除非布局要求 `w-full`。
- **状态**存在于组件内，不在调用点：hover 用变体的 hover 角色（soft-primary：
  `control-soft-primary-bg-hovered` 加一条 `primary-muted` 描边，使其强调色标签保持 AA）；按下
  （`:active`，指针按下或按住空格）看起来与 hover 完全相同——同样的 `active:` class 镜像
  `hover:` 的那些，因此没有指针的键盘按下也能看到它——且绝不移动几何；focus-visible 在所有变体里都是
  2px `primary` 描边加 2px 偏移，因此它在 primary 或 destructive 填充之上、在每个表面之上都可见
  （`ghost` 本身没有表面，在聚焦时也会取得 hover 填充与默认文本色，使聚焦按钮读起来像一个控件，
  而不只是空空间周围的一圈环）——除了 `ghost` 纯图标按钮，它把环画在方块内侧 2px，因为那些按钮贴在
  28px 的头部条与轨栏里，其 `overflow: hidden` 会裁掉外环；disabled 通过应用样式表定义的
  `button-disabled` 变体，用 `control-disabled-*` / `control-primary-disabled-*` /
  `control-destructive-disabled-*` 角色让元素留在原位（绝不用透明度，绝不隐藏）——包括带
  `data-disabled` 的元素，或原生 `:disabled` 且未在加载中的元素，因此禁用 `<fieldset>` 内的按钮也会
  取得该皮肤；`loading` 会把图标换成该尺寸图标大小的 spinner（配合 `spinIcon` 时改为就地旋转按钮自己
  的图标——`apps/web/src/components` 的 *Loading vocabulary* 刷新惯用法——并在减少动效下改显静态
  spinner 字形，使该状态仍然可见）（或无图标时，覆盖它为其宽度和无障碍名称而保持透明的标签），设置
  `aria-busy`，并原生禁用该元素，使操作无法运行两次；`selected`——或打开的 Radix 触发器
  （`data-state="open"`）——是 `outline` 与 `ghost` 的持续切换/打开填充（`control-bg-selected`），
  与按下不同；`soft-primary` 把选中显示为它的 hover 外观（hover 淡彩上的 `primary-muted` 描边），
  这是唯一保持标签 AA 的强调色层级，因此展开的强调色 chip 仍读作已展开。切换器保留 `aria-pressed`；
  展开器保留 `aria-expanded`。
- **行**：绘制自身 hover 或选中填充的列表行使用与 ghost 填充相同的角色（`control-bg-hovered` /
  `control-bg-selected`），因此其上的 ghost 操作将毫无反馈。这类行带 `data-row`，其中的 `ghost` 按钮
  在 hover、按下、focus-visible 与选中时改用 `container-elevated-bg`——即该系统之前行操作的那种浮起
  外观——使 hover 中的行操作与打开的行菜单在已填充的行上仍然可见。带标记的行：项目行与工作区行、
  变更行、todo 行、每个 cmdk `CommandItem`，以及仅在选中时的历史浮层行——这些行位于浮层的浮起表面之上，
  只在选中时填充，因此未选中的行保留 ghost 自身的 hover 填充。
- **调用点只负责摆放**：外边距、self/flex/grid 摆放、`w-full`、定位、响应式可见性与整按钮 hover 显现。
  高度、内边距、间距、圆角、颜色、描边、排版与状态皮肤都会被采纳门禁拒绝。
  记录在案的例外连同理由一起放在该门禁里（`tone` 无法表达的 hover 变红裸操作；
  必须折行显示长标签的扩展下拉选项）。
- **非按钮**：tab、菜单项、开关、切换分段、单选/分段选项、树与列表行、展开器标题、chip、行内文本链接、
  编辑器的发送胶囊按钮、原生窗口控件与大型选择卡片都保留各自的组件或基础组件。
- 对话框关闭按钮与 `ToastClose` 是 `IconButton ghost m`。

## 基础组件不变量（Primitive invariants）

逐文件的接口是 `button`、`switch`、`dialog`、`dropdown-menu`、`context-menu`、`menu-styles`、
`popover`、`command`（cmdk 组合框主体）、`textarea`、`tooltip`、`resizable`、`toast`、
`toggle-segment`（细长的头部切换器：活动分段在 `control-bg-selected` + `text-default` 上，
`data-active` / `aria-pressed`）、`skeleton`（`SkeletonRows` / `LoadingRegion`，唯一的加载占位；
其使用规则见 web 组件规格）以及 `attention-line`（`AttentionLine`，即
“Your input is needed · Agent is waiting” 那一行；一旦始终挂载的 polite 区域承载了宣告，
`announced` 会把可见行从辅助技术中隐藏，`live={false}` 则去掉该区域），外加 `utils` 与 `theme`。
`menu-styles` 由菜单基础组件与自定义应用菜单行共享。当资源渲染器成为包时，`toggle-segment` 与
`skeleton` 从应用移到这里，因为应用与 [[module-resource-renderers]] 都要用它们；当
`ask_user_question` 卡片成为扩展 web 一半时，`attention-line` 出于同样的原因迁移，因为应用的扩展提示
也要用它。

- Switch 暴露原生禁用行为与 `role="switch"`/`aria-checked`，而非可见的 On/Off 文本。每个模态对话框的
  遮罩都带 `data-modal-layer`，`useModalLayerOpen()` 报告是否有打开的模态：它让调用方无需对话框注册表
  即可区分“被模态覆盖”与自身可见性，且与 `aria-hidden` 如何对待活动区域无关。Dialog 接受 `hideClose`
  以用于无边框表面。`DialogPanel` 就地渲染 Radix 内容而不使用 portal 或遮罩，用于调用方定位的非模态
  对话框；标题/角色语义、Escape 关闭与焦点归位仍由 Radix 持有，而周边 UI 保持可交互。
- React 19 的 ref 会穿过控件；dialog/popover 的 portal 保持无障碍焦点与滚动行为。Popover 接受宿主
  Dialog 容器以兼容滚动锁定。
- 纯图标控件使用带 `aria-label` 的 `IconTooltip`，而非原生 `title`；一个根 `TooltipProvider` 设定延迟。
  原生 `title` 只保留给非控件的截断回退，或 provider 之外的表面（应用的评审 widget）。`wrapTrigger`
  为禁用控件或另一个 Radix 触发器提供可交互的 flex span，隔离它们的 `data-state` 值；调用方不要手写
  该包装器，`Button`/`IconButton` 在禁用时自行应用它。应用、SDK 与扩展 web 的调用点测试强制此
  不手写 span 的约定。`IconTooltip` 只在 hover 与*键盘可见*的聚焦时打开：不匹配 `:focus-visible` 的
  聚焦（指针打开后对话框或 popover 的自动聚焦、焦点回到被点击的触发器）绝不打开它，因为打开的 tooltip
  是最顶层的可关闭层，会吞掉本应给对话框的 Escape。出于同样的原因，打开的 `IconTooltip` 会在 window
  捕获监听器中、Radix 的文档级层处理之前同步地自我关闭，于是一次 Escape 既关闭 tooltip *也*关闭其下的
  对话框、popover 或菜单。Tooltip 内容堆叠在 `z-150`，高于对话框与 toast 视口（`z-[100]`），因此
  toast 自身的关闭 tooltip 绝不会被画在 toast 之下。
- 上下文菜单与下拉菜单共享菜单几何与 token class：内容表面、圆角/阴影、条目/图标间距、分隔符、语义动作
  颜色、聚焦行，以及视口碰撞行为。上下文菜单额外增加指针位置的右键/触摸长按锚定；功能侧拥有启用的手势
  与动作。下拉菜单在垂直方向以 60vh 与 Radix 可用高度中较小者为界，可滚动且水平溢出隐藏，使过长的行
  截断而不是滚动。
- Toast 暴露 `ToastProvider`/`Toast`/`ToastViewport`/`ToastTitle`/`ToastDescription`/`ToastAction`/
  `ToastClose` 与 `toastVariants`（`error`/`success`/`warning`/`info`）。严重程度是中性
  `bg-container-elevated-bg` 背景上的统一语义描边（`border-feedback-*-muted`）。`ToastAction` 不带
  样式：调用方通过 `asChild` 提供控件。队列所有权、图标与通知生命周期留在应用中（`panels/Toaster`
  组合这些基础组件）。
- 可调整大小的把手在其前缘绘制 1 px 细线；指针命中区只延伸到后续面板（精细指针时外边距 1 px），
  因此前一面板的后缘滚动条仍可用。没有握把变体。
- 基础组件只使用宿主的 token 工具类，绝不使用 shadcn 默认的 oklch 调色板。

代码高亮刻意缺席：`CodeBlock` 与 shiki 依赖应用的主题注册，在第二个扩展需要该接缝之前仍留在应用内。
