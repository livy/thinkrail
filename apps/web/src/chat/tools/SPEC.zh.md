# tools —— 内置工具渲染器

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

**随包发布的工具在应用本地的呈现半边**，按工具名与各自服务端的能力相连（扩展模型见父规格）。
`register.ts` 通过 `registerToolRenderer`（且只对 `mcp__*` 用 `registerToolRendererPrefix`）接上这些
渲染器——渲染器 + 折叠头部摘要 + chrome/显著度——作为 `ChatView` 的一个副作用导入。抽取出的扩展渲染器
由宿主的 `extensions/registry.ts` 注册。未注册工具回退到 `DefaultToolRenderer`，并被当作**常规**
工具（它们会折进活动分组）。

## 本模块内容（What's here）

- **核心 pi 工具** —— `BashCard`（终端块）、`ReadCard`/`WriteCard`（工作区相对路径 + 高亮文本文件）、
  `EditCard`（路径 + 删除/新增行 diff）。全部**常规**。一个已结算的 `read`（无论成功或出错）以及一次
  成功的 `write`/`edit`，在路径属于活跃工作区时使用共享的结构化文件链接基础组件，打开 shell 的预览槽；
  运行中/失败的变更或外部路径保持为文本。`read` 返回的图片内容由父 chat 模块的通用结果内容层预览，
  与其他每个产出图片的工具走同一路径；`ReadCard` 不对图片载荷做特例处理。
- **`ResolveCommentCard`** —— 宿主拥有的 `resolve_comment` 评审工具的紧凑回执（能力：服务端的 `agent`
  模块 + `reviews` 接缝；见 [[submodule-server-reviews]]）：一个 ✓ + 被解决评论的 id/备注。**常规**
  ——评审侧栏才是解决状态所在之处；该卡片只是会话记录内的痕迹。
- **`RequestReviewCard`** —— 宿主拥有的 `request_review` 工具的判定卡片（能力：服务端的 `agent`/`host`
  request-review 接缝，它会派生一个只读评审 subagent）。它从工具结果的 `details` 读取结构化
  `PlanReviewResult`（防御性地，经 `isPlanReviewResult`），显示一个 批准/请求修改 指示器 + 评审者的一段
  摘要，并把发现项渲染为共享的展开式 `ReviewPackageComments`（评论→条目映射与行基础组件都在
  `reviewPackage.ts` + `ReviewPackageComments.tsx` 中，被 `reviewFix`/评审包渲染器复用）。注册为
  `primary` + `defaultExpanded`；`data-verdict` + `data-testid="request-review-card"` 是测试钩子。
- **Ask user question** 由 [[module-ext-ask-user-question]] 拥有，在本模块之外。其卡片通过公开的 web
  入口注册为 bare 且 `interactive`；计划页通过注册表渲染它。
- **Visualize** 由 [[module-ext-visualize]] 拥有，在本模块之外。其 primary/default-expanded 注册与具名
  `MermaidView` 通过公开的 web 入口到达 chat。
- **规格图谱工具** —— 一个防御性的 `SpecToolCard`，为全部七个 `spec_*` 名字注册。每个都保留该能力
  可读的文本、折叠长输出、用有意义的活动摘要替代冗余的参数 JSON，并且只把从已知成功结果 `details` 形状
  取得的、确切的独立工作区路径 token 变成预览动作；较短的已知路径绝不会去链接另一个路径内部的某段后缀。
  `spec_delete` 绝不链接其成功删除的路径。常规。
- **`web/`** —— `pi-web-access` 的搜索/抓取/已存内容渲染器；有自己的子规格
  （[web/SPEC.md](web/SPEC.md)）。常规。
- **`mcp/`** —— pi 的内置 MCP 工具：`mcp__*` 卡片（服务器 / 工具、key/value 参数、有序文本、宿主
  结构化内容摘要的折叠 JSON 树、Full output、登录动作）、资源列举/读取卡片以及 `tool_search`；有自己的
  子规格（[mcp/SPEC.md](mcp/SPEC.md)）。常规。
- **五个 `todo_*` 工具**刻意保留常规回退回执：它们相连的产品表面是 chat 计划，而不是本模块里的第二套
  渲染器体系。
- **`subagent/`** —— `pi-subagents` 的委派渲染器：`AgentCard`（`Agent` 工具——**主**；也为
  `get_subagent_result` 注册，常规）以及纯的 `runDetails` 读取/格式化器（父 chat 的后台结果行复用它们）；
  有自己的子规格（[subagent/SPEC.md](subagent/SPEC.md)）。
- **共享件** —— `CodeBlock`（shiki）、`Collapsible`（长输出的 "Show all N lines" 折叠；调用方提供稳定
  的、由工具调用派生的 id，使状态与几何锚定能跨虚拟化存活）、`ToolFileLink` + 由父 chat 模块共享的
  `workspaceFileTarget` 与 `lib` 的 `projectRelativePath` 支撑的精确引用链接文本。纯工具函数
  `toolHelpers`（参数读取器、`resultText`、`languageFromPath`）与规范 `toolResultContent` 解析位于
  `@thinkrail/extension-api/web`，因此扩展的文本抽取与 chat 的通用图片层在什么算有效内容上不会产生分歧。

## 边界（Boundary）

- **公开接口：** 副作用式 `register` 导入 + 供兄弟渲染器使用的共享 `CodeBlock`/`Collapsible`。无 barrel
  （chat 会拉入 shiki——逐文件导入，与父模块相同）。
- **允许的依赖：** 父 chat 的基础组件（`toolRegistry`、`Markdown`、`ChatActions`、`interactionState`、
  `fileTargets`、`foldState`）；`contracts`（类型与纯守护）；`@thinkrail/ui/*`（包括 `attention-line`
  ——与扩展提示共享的 "Your input is needed" 行）；`@thinkrail/extension-api/web`；`lib`；
  `@remixicon/react`。
- **禁止：** 值导入任何 `pi` 包；`store`/`transport`（渲染器保持展示型——可抽取进未来的
  `packages/chat-ui`）。

## 必须做对（Get right）

- **防御性地渲染：** 工具的 `details` 结果形状不是稳定 API——尽力读取，并回退到其文本内容
  （`resultText`）。缺失的结构化引用只会去掉链接，绝不影响散文。
- 文件动作只来自显式的参数/细节，且只通过 `ToolRenderProps.onOpenFile`（打开）或 chat 的资源作用域与
  宿主（渲染一个 `ResourcePreview`、在标签页中打开）；绝不从任意结果文本推断，也绝不把应用集成导入进
  渲染器。
- 工具名必须与能力完全一致——名字就是连接键。唯一被认可的例子是 `mcp__` 前缀：pi 把 MCP 工具命名为
  `mcp__<server>__<tool>`，因此注册表先解析精确名，其次解析最长的已注册前缀，且由渲染器、摘要、chrome
  与显著度共享**同一个**解析器。前缀渲染器绝不从服务器加载代码；它们渲染 pi 的结果
  （[mcp/SPEC.md](mcp/SPEC.md)）。三个资源工具与 `tool_search` 按精确名注册。
- 只用 token 工具类样式（无原始 hex / 内联 `style`）。
