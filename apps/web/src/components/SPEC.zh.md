# components —— 共享 UI 原语

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

应用中依赖很轻的共享 React 原语：让某个失败区域不会把根卸载的错误边界、项目自定义图标、功能面板
（当前是 Projects 侧栏）使用的二元 attention 点与“正在工作”图标形态、静默滚动框架，以及共享的加载
骨架原语。共享的 shadcn 原语位于 [[module-ui]]（`@thinkrail/ui/*`），不在本模块。

## 边界（Boundary）

- **拥有：** `ErrorBoundary.tsx` —— 一个类边界（`getDerivedStateFromError`），它渲染一个带主题、
  自足的兜底界面，而不是把抛出的错误传播到根。它：
  - 当任何 `resetKeys` 值变化时重置已捕获的错误（接到子树的身份 —— 工作区／标签 id —— 上，因此
    导航离开即可自动恢复）；
  - 通过纯函数、有单元测试的 **`isChunkLoadError`** 对失败的动态 `import()` 分类（旧的 Vite chunk／
    504／Safari 的 "module script failed"），并把它们导向页面**重新加载**（重新拉取该 chunk），而不是
    原地重试；
  - 把崩溃记录到 console（`componentDidCatch`）—— UI 本身已优雅降级 —— 并把它连同该边界未翻译的
    `reportId`（绝不用已显示、已翻译的 `label`）交给经 `setErrorBoundaryReporter` 安装的唯一上报器
    （由 `main.tsx` 接到 transport 的错误上报；上报器自身失败会被吞掉）。这个接缝让本模块不带
    store／transport。
- **`CustomIcon.tsx`** —— 通过一个 CSS `mask-image` 的 span（`index.css` 中的 `.custom-icon*` 类）
  把 `public/custom-icons/` 里的 SVG 渲染成可主题化的 `currentColor` 字形，因此自定义字形与 Remix
  图标完全一样地用 `size-*` 定尺寸、用 `text-*` 上色。名称是一个带类型的联合（`CustomIconName`）；
  目前有：`file-diff-line`／`file-diff-fill`（Changes 评审标签的字形）。
- **`QuietScrollArea.tsx`** —— 无 store 的溢出观察器与两个呈现表面：`QuietScrollArea` 拥有一个普通的
  原生滚动视口（可选的 `viewportRef` 回调把该元素交给需要保存／恢复其滚动的调用方，例如
  `panels/RememberedScrollArea`），而 `QuietScrollFrame` 观察第三方后代滚动控件，既不接管其内容也不
  接管其输入，并能接收该库的权威边缘状态。`styles/global.css` 拥有每个原生滚动容器的唯一滚动条定义：
  6px 的槽位、透明轨道、静止时透明的滑块、滚动元素被悬停或携带 `data-quiet-scroll-intent` 时为
  `--border-default`、直接悬停／激活滑块时为 `--text-muted`。这一显露经由元素级自定义属性
  （`--scrollbar-thumb`）传递，滑块伪元素与 Firefox 的 `scrollbar-color` 都读取它，因为 Chromium 会
  针对滚动条部件而非元素来求值 `::-webkit-scrollbar-*` 选择器内的 `:hover`；每个元素都会重置该属性，
  因此被悬停的祖先永不显露其后代。随附的高对比主题保留 2px 的静止细线；操作系统强制色保留系统色滑块
  并移除幕帘；减少动态效果则移除过渡。因此 `QuietScrollArea` 与 `QuietScrollFrame` 不拥有任何外观：
  它们提供超出悬停的意图（focus-within、带短暂宽限期的进行中滚动、离开框架的指针拖拽、触摸）、溢出
  测量，以及在被裁切方向上的指针透明 16px 边缘幕帘。`QuietScrollFrame` 还把 xterm 的非原生滑块重新
  换肤为同一个滑块（底层滑块透明、6px 的视觉滑块、不透明度按意图与实测垂直溢出门控）。表面颜色是
  显式的语义 prop（`sidebar` 或 `terminal`），绝不从排布推断。
- **`AttentionDot.tsx`** —— 无 store 的静态绿／强调色点，其唯一的无障碍标签是 “Needs attention.”。
  它不携带原因、计数、tooltip、动效或清除行为；调用方只决定它是否存在。它是需要输入与未读结果两类
  attention 的唯一标记。
- **`RunningIcon.tsx`** —— 无 store 的 **“ThinkRail is working” 徽标**：位于圆角方形框架内的品牌
  字母组合，以 `currentColor` 绘制，因此像 Remix 图标一样用 `size-*` 定尺寸、用 `text-*` 上色，
  无障碍标签为 “Agent working”。在允许动效时，框架（左半边留出间隙、右半边相接的两段）与字母组合的
  六笔画在一个 3.263s 的周期里绘制又擦除（`index.css` 中的 `loader-*` 关键帧）；只有框架可见的一拍
  也是设计的一部分。每一笔画都是被裁切到 `BRAND_MARK_SEGMENTS` 对应区域的宽线，而 `BRAND_MARK_PATH`
  正由同一批分段拼成，因此完整绘制出的徽标就是 shell 的标志，而不是可能发生偏移的第二份副本。本地
  动画几何只有笔画中心线、框架，以及 R 的竖干与顶横之间的小间隙（竖干笔画提前终止）。减少动态效果
  时每一笔画都停在完整绘制的虚线上：静态、完整的徽标。这个加载器取代了带追赶列车与脉冲字母组合的
  圆环；在那之前是行自身的身份字形在呼吸，因其在 14px 下变暗的文件夹读作“已禁用”而被弃用。调用方
  只传尺寸／颜色类，并从归一化的宿主状态判断顶层会话是否在运行；该图标不渲染点、spinner、计数、
  tooltip 或空闲状态。
- **同样治理（实现位于 `@thinkrail/ui/skeleton`，[[module-ui]]，因此应用与
  [[module-resource-renderers]] 共享它）：** `SkeletonRows`，即每个加载表面使用的唯一脉冲行占位符，
  以及 `LoadingRegion`，即包裹它的定尺寸包装形状 —— 大多数调用点真正想要的（一个用于该区域自身
  内边距／尺寸的 `className`、一个可选 `label`，它被接到 `SkeletonRows` 的 `role="status"` 区域而
  不是再开第二个区、以及一个可选 `testId`）。完整的加载词汇及其规则见下文。
- **同样拥有：** `preloadedLazy.tsx` —— 一个分块组件加 `preload()`。普通 `lazy` 即使在 chunk 已缓存
  时也会在首次渲染挂起；在一次同步 store 更新之下，React 随后会让兜底保持约 300ms。一次挂载若发现
  chunk 已加载则直接渲染它；挂载永不切换组件类型。在拥有它的界面挂载时调用 `preload()`，而不是在
  启动时，这样其他界面永不拉取该 chunk。加载失败会像普通 `lazy` 一样到达 `ErrorBoundary`；渲染永不
  重试它，只有下一次 `preload()` 会。已知限制：在慢速链路上，或当揭示的数据在挂载时已存在于 store 中，
  首次挂载仍走 `lazy` 路径。
- **公开接口：** `ErrorBoundary`、`isChunkLoadError` —— 经 `@/components/ErrorBoundary` 直接导入
  （没有 barrel；占位符来自 `@thinkrail/ui/skeleton`）；`AttentionDot` 经
  `@/components/AttentionDot`；`RunningIcon` 经 `@/components/RunningIcon`；`CustomIcon`、
  `CustomIconName` 经 `@/components/CustomIcon`；`QuietScrollArea`、`QuietScrollFrame` 与
  `QuietScrollEdges` 类型经 `@/components/QuietScrollArea`；`preloadedLazy` 经
  `@/components/preloadedLazy`；`useNow()` 经 `@/components/useNow`（把挂钟时间作为
  `useSyncExternalStore` 的值架在共享的 30 s 计数器上，因此渲染永不自己调用 `Date.now`）。
- **允许的依赖：** React、`@remixicon/react`、`@thinkrail/ui/utils`（`cn`）、`@thinkrail/ui/button`
  （错误兜底的 Reload / Try again 是共享按钮）、`@thinkrail/locales`（UI 字符串）、`lib`
  （`shallowEqualArrays` —— 重置 key 的比较，共享而不是重述）、`constants`（`BRAND_MARK_SEGMENTS`
  —— `RunningIcon` 裁切到的字母组合区域，与 shell 标志共享而不是复制）。刻意保持依赖轻量，且两者
  都是叶子，因此*任何*区域（shell、panels、`main.tsx`）都仍能把它包起来而不会造成循环。
- **禁止：** `store`／`transport`／`panels`／`shell`／`chat`／`contracts`；`server`／`shared`／`pi`；
  内联 `style` 对象或原始十六进制色值（兜底只用 token 工具类上主题）。

## 加载词汇（Loading vocabulary）

应用的加载词汇恰好是**两层** —— 每个异步间隙都渲染这两者之一，绝不用第三种东西，绝不用裸文字，
绝不什么都不显示：

1. **`SkeletonRows`（通常经 `LoadingRegion`）—— 用于*内容区域***：任何将填充大量内容的区域，无论该
   区域如何被框定（工具面板、对话框的列表、菜单的列表区段、标签主体、恢复 chat／plan／编辑器的
   `Suspense` 兜底）。**表面尺寸决定层级，而不是容器类型** —— 一个居中的 spinner 放在大片空区域里，
   读起来比隔壁面板针对同一件“数据正在路上”的事所用的贴合内容的骨架更沉重、更令人紧张，因此无论
   区域身处何种部件（对话框、下拉、workbench 标签），它都绝不只有 spinner。参考位置：
   `panels/FileTree.tsx`、`panels/SpecsPanel.tsx`、`panels/ChangesPanel.tsx`、`panels/ReviewPanel.tsx`、
   `panels/PlanPane.tsx`、`panels/ProjectTree.tsx`（worktree 列表）、
   `panels/ExistingWorktreeDialog.tsx`（worktree 候选列表）、`panels/ChangesScopeMenu.tsx`（提交列表）、
   `panels/WelcomePanel.tsx` 的 `CardSkeleton`（一个手工塑形、尺寸贴合它所替代卡片的变体，因为通用行
   列表会误述卡片栅格的占地），以及 `shell/WorkspaceWorkbench.tsx` 的 `MissingResource`（每个
   chat／plan／编辑器标签主体与 lazy chunk 的 `Suspense` 兜底）。
2. **`Loader2` spinner（通常配一段短标签）—— 用于*在途动作或瞬态状态*，且钉在其控件上**：单个按钮、
   单个菜单项、单个图标 —— 绝不是整个区域。参考位置："New chat"/"Creating…" 按钮状态、
   `shell/WorkspaceChatHistory.tsx` 的历史图标、`chat/ActivityGroup.tsx`／`chat/ToolCard.tsx` 的步骤
   图标、`panels/ProjectTree.tsx` 的 "Creating worktree…" 待定行。在共享按钮上该层级是 `loading`
   （在图标位置放图标尺寸的 spinner）。该层级下更窄的具名子惯用法：**`RefreshCw` 原地旋转**（图标
   不变，只是旋转；在共享按钮上写作 `loading spinIcon`），用于某个控件上的手动 “Refresh” 动作，而
   该控件周围的内容在刷新运行期间保持可见且有效（`chat/ModelEffortPicker.tsx`、
   `chat/ModelSelector.tsx`、`panels/GithubSettings.tsx`、`panels/ProvidersSettings.tsx`、
   `panels/BranchPicker.tsx`）—— 在那里换成通用 spinner 会丢掉原地旋转免费给出的“这仍然可用，只是
   在刷新”信号。在减少动态效果下旋转无法承载信号，因此 `spinIcon` 改为显示静态的 spinner 字形。

除了选对层级之外：

- **骨架在宽限期后淡入，绝不闪现。** `SkeletonRows` 携带 `animate-loading-in`（`index.css`：300ms
  不可见，然后 150ms 的不透明度淡入），因此一次在宽限期内完成的读取会从空白但已定尺寸的区域直接走到
  揭示出的内容，而不是先闪一两帧骨架（打开 Changes 标签曾是可见的案例）。它全程保持挂载，因此其
  `role="status"` 区域与 test id 从首帧起就在场；只有它的不透明度会等待。
- **没有布局跳动。** 加载态占据到达内容将占据的同一片占地 —— 尺寸／形状贴合其区域的骨架、尺寸贴合其
  后继卡片的 `CardSkeleton` —— 因此替换它的数据永不移动周围任何东西。
- **重试／错误能力永不与活动中的加载指示并存。** 在 spinner 或骨架旁边显示 "Retry" 会同时发出两个
  互相矛盾的信号（“这是正常的，等等”和“这已经失败了，动手”），而绝大多数情况下它仍是正常加载。两
  种正当形状：(a) 重试按钮**只在真正的失败之后**出现 —— 被拒绝的请求落进 `catch`，例如
  `panels/ChangesPanel.tsx` 的 `changes-retry`、`panels/ExistingWorktreeDialog.tsx` 的
  `existing-worktree-retry`；或 (b) 对于一个已有*自动*后台进程正在竞相恢复的资源（经
  `shell/chatReconciliation` 的 chat 会话水合），只显示加载指示，只有当那次自动尝试在短暂宽限期之后
  仍然停滞时才露出重试能力 —— `shell/WorkspaceWorkbench.tsx` 的 `ChatResourceBody`／
  `CHAT_RETRY_DELAY_MS` 是参考实现；在发明新形状之前先找同一形状。
- **条件性缺失的内容不是加载态。** 当“仍在加载”与“这里确实什么都没有”渲染得一模一样（都是空白）
  *并且*空白是正确的稳态时 —— 而不只是未处理的间隙：在未解析时什么都不显示是正确的，不是缺陷。
  `panels/PlanRailPane.tsx` 因其 `plan.data === null` 分支而露出小小的 "Loading…" 一行，因为侧栏工具
  可能在首次计划读取落地之前打开，而“空”是合法的稳态；那里没有像
  `panels/WelcomePanel.tsx` 那条最终总会被填充的卡片行那样承诺内容的空区域。只有当区域保证会被填充
  时才求助骨架；只有当它可能永远合法地不被填充、且调用方已把缺失当作缺失而不是待定状态时，才求助
  什么都不显示。
- 既无骨架也无 spinner 的裸 "Loading…"/"Restoring…" 文字是缺陷，句号。
- **已解析的内容淡入而不是弹出。** 骨架的数据到达之后，替换它的元素携带 `motion-safe:animate-reveal`
  （`index.css` 中共享的 `reveal` 关键帧 —— 150ms 不透明度 + translateY，与问卷卡片
  （[[module-ext-ask-user-question]]）已经用于卡片落位的同一原语；用 `motion-safe:` 前缀，使
  `prefers-reduced-motion` 得到瞬时切换）。它加在已解析分支自己的根上 —— 绝不在骨架上，也绝不加在
  同时包住骨架分支的包装器上（`Suspense` 边界的*子元素*，而不是 `Suspense` 元素本身）—— 因为该动画
  在该元素挂载时只播放一次，而包住两个分支的共享包装器会在首帧触发，那时数据（或已解析内容本身）还
  不存在可供淡入。它永不在普通重渲染时重复触发（更长的 todo 列表、一条新 chat 消息），因为 React 在
  这些情况下复用同一个持久 DOM 节点 —— 只有分支切换才是真正的挂载。已注册的
  `RenderedDiff`／`MarkdownPreview` 根携带它 —— 但明确**不加**在 `MonacoEditor` 上：编辑器与其视图在
  同一次提交中出现，内部没有加载态，150ms 的淡入只会推迟可读文字。终端主体
  （`TerminalWorkbenchBody`）也跳过淡入，因为终端的 PTY 附加受其自身可见性挂载门控，不值得为一个
  装饰性淡入冒风险；Radix 菜单的项同样跳过，因为把裸 `DropdownMenuItem` 包进 div 会破坏该原语基于
  直接子级的键盘／typeahead 遍历（`panels/ChangesScopeMenu.tsx` 的提交列表因此保持先骨架后弹出）。
  `chat/ChatView.tsx` 跳过的理由更锋利：祖先上非 `none` 的 `transform` —— 包括动画进行中、关键帧尚未
  定在 `none` 时 —— 会建立新的包含块，这破坏了嵌套其中的 `chat/activityBreadcrumbs.tsx` 面包屑栏的
  粘性定位（`e2e/activity-breadcrumbs.spec.ts` 在评审中抓住了这一点）。任何带有 `position: sticky`／
  `fixed` 后代的元素都被径直判定不适用这种基于 transform 的淡入，不需要例外清单 —— 添加之前先检查
  是否存在这样的后代。

## 必须做对（Get right）

- **保护范围：** React 边界只捕获**渲染 + lazy 导入**抛出的错误 —— **不**捕获事件处理器、effect 或
  被拒绝 promise（例如 `transport.request`）中的错误。那些经 `transport` 的 `errorText()` 以错误
  轮次／提示的形式出现，不在这里。shell 的“面板不能让应用变空白”保证针对的是渲染／懒加载；异步
  失败是另一条路径。
- 错误边界挂载在哪里（每个区域 + 最后兜底的根包裹）由 `shell/SPEC.md` 与 `apps/web/SPEC.md` 中的
  父依赖图拥有，不在此处重复。
- 静默滚动的意图局限于它所渲染的表面，而不是 workbench 记忆的活动组。一次滚轮／触控板手势不必移动
  DOM 焦点，而记忆的组焦点会让一条侧栏无限期可见。因此第三方框架在自己宿主上观察焦点与指针意图，
  只添加视觉／测量适配器；它永不读取 shell placement。
