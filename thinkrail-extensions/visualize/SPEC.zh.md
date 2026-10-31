# 可视化扩展

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

`visualize` 工具在 ThinkRail 这一侧的两半。`./server` 用与浏览器渲染器达成一致的**严格** mermaid
校验器组合那个可移植能力（[[module-pi-visualize]]）；`./web` 是由 props 驱动的展示层。
工具名把二者连在一起，无需线上契约改动。

## 边界（Boundary）

两个入口点，没有根 barrel。`@thinkrail/ext-visualize/server` 默认导出
`defineServerExtension({ name: "visualize", title, description, extensions: [createVisualizeExtension({ validateMermaid, checkFile, scratchDir })] })`
—— 无子节点、无 skill 包 —— 且可以导入 `extension-api/server`、`@thinkrail/pi-visualize`、`mermaid`、
`linkedom`，以及 `node:fs`/`node:path`。`@thinkrail/ext-visualize/web` 默认导出一个
`defineWebExtension` 描述符，外加具名 `MermaidView`，用于聊天 markdown 中的围栏 mermaid，
并消费 `extension-api/web`、逐文件 UI 原语、`@thinkrail/resource-renderers`（`ResourcePreview` 与宿主接缝）、
contracts、`@thinkrail/locales/react`（用于其字符串，`visualize.*` 键）、React 与 Remix Icon。
两半都不导入对方、应用/宿主内部，或（web 侧）任何 pi 包；浏览器在渲染器内部惰性加载 `mermaid`，
绝不在急切的入口块中加载。

## 严格校验（Strict validation (server)）

该校验器是通过 pi 包的 `validateMermaid` 接缝注入的那一个，它**取代**本包尽力而为的 `lovely-mermaid`
探针；本包仍拥有形态检查、空源拒绝、源枚举与字段专属诊断。其保证是**由 web 渲染器所用的那个确切
`mermaid` 版本（在 catalog 中固定）进行语法解析**，而不是 SVG/布局成功 —— 渲染器保留其防御性回退。

- `mermaid` 在其解析器初始化期间需要浏览器全局对象，而 DOMPurify 对 flowchart/class/state/gantt/mindmap
  需要一个真实的 `document`（在 Bun 下已验证），因此 `linkedom` 是必需的，而非可选：
  一个轻量 DOM 只在惰性 `import("mermaid")` 期间存在，而先前的 `window`/`document` 描述符在 `finally`
  中被恢复。
- `mermaid.parse` 调用通过一个存储在 mermaid 实例上的 promise 尾部串行化（因此本模块的每个已加载副本
  共享它），因为 mermaid 会为每次解析重置共享配置。
- 非法语法表现为本包的 `isError` 工具结果，要求模型更正并重试；扩展不注入任何 prompt，也不强制重试。

文件预览得到 `workspaceFileChecker`：路径必须相对、留在会话 cwd 之内（词法上以及在解析链接之后，
使用宿主读取守卫的围堵规则，因此 `..draft.html` 是一个名称而不是父目录）、指向一个普通文件、
并避开 `.git`，否则工具结果要求模型先写入它 —— 与宿主 `fs.readFile` 随后所服务的正是同一批路径，
因此卡片永不会把工具已接受的文件报告为不可读。`scratchDir` 是 `.thinkrail/context/mockups/`，
位于宿主被 gitignore 的暂存目录之内，因此 mockup 永不会变成项目变更。

宿主从 server 注册表组合这个描述符（[[submodule-server-extensions]]）；通过该组合的一次严格校验调用
在 `packages/server`（开发）与产物探针（CLI 二进制与桌面端）中被固定，因此原生探针永不可能成为
在 ThinkRail 中实际运行的东西。

## 渲染契约（Rendering contract (web)）

图示、对比选项与推荐选项保持既有的聊天呈现。一个已完成的 `type: "file"` 调用渲染 `FileCard`：
标题、`before → path` 链接（经由 `onOpenFile`），以及 [[module-resource-renderers]] 的 `ResourcePreview`
用于 path 与 `before` —— 与 diff 评审相同的渲染器、diff 与评审意见。卡片外框是一个固定的 480px 盒子
（Monaco 与有界 diff 需要高度），其工具栏增加 **Open in tab**（保留时用 `host.openFile`，
前/后对比时用 `host.openComparison`）与 **Full screen**（一个用 Split/Inline 渲染同一预览的对话框；
渲染器选择与视图状态，例如 HTML 预览宽度，与卡片共享）。卡片永远显示文件的**当前**内容，
绝不是工具调用时的快照：编辑一个 mockup 会更新显示它的每一张卡片（评审意见跟随文件）。
一个在已挂载卡片之下被删除的 mockup，会让它停在最后加载的版本上，并带一条 “Couldn't refresh” 提示；
之后才挂载的卡片显示加载错误。这就是把评审锚定到真实文件的代价；工具指南要求 agent，
当一个更早的提案应当保持可见时，把它保留为一份副本并作为 `before` 传入。工作区与聊天来自
`useResourceScope`，聊天消息记录把它提供给自己的行，因此评审发送落在显示该卡片的那个聊天里；
没有宿主或作用域时（一个 subagent 消息记录），卡片只显示链接。该注册是主要的且默认展开；
其标题或图示/对比/文件摘要无需渲染即可获得。非法或流式传输中的参数保持防御性，
工具错误保留可读的结果文本。

`MermaidView` 从宿主的 CSS 变量推导 mermaid 严格模式的主题，并在一次完成的 `data-theme` 切换之后重新渲染。
它固定 `look: "classic"` 与 `layout: "dagre"`：mermaid 12 的默认值会添加节点辉光阴影与正交边，
与应用的扁平表面冲突，并会重排既有图示。它支持调用方提供的加载回退、一个全屏 Dialog，
以及平移/缩放控件。全屏画布从它本地的姿态起始基线接管修饰键滚轮缩放与 WebKit 手势捏合，
使桌面页面的缩放不会抢走该手势。渲染器失败时显示错误与一个与应用的 CodeBlock 无高亮分支一致的
纯源块；它永不导入应用的 shiki 高亮器。`args` 与平移/缩放运算保持聚焦的单元覆盖。
