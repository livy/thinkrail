# @thinkrail/pi-visualize —— 面向原生 pi 的 visualize 工具

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

可视化能力中已发布、可移植的那一半：一个工具 `visualize`，它校验图示、对比或文件预览请求，
返回一个 markdown 回退作为面向模型的结果，并在 pi 的终端 UI 中绘制结果。它对 ThinkRail 一无所知。
ThinkRail 通过 [[module-ext-visualize]] 的 server 半边（已被删除的 `packages/pi-visualize` 的后继）
组合它，并注入其严格校验器。

## 公开接口（Public surface）

- **默认导出** —— 带默认配置的 pi `ExtensionFactory`；`pi install` 通过 `pi` 清单加载的东西。
- **`createVisualizeExtension({ validateMermaid?, checkFile?, scratchDir? })`** —— 返回一个
  `ExtensionFactory`，可注入 `MermaidValidator: (source) => void | Promise<void>`。该校验器**取代**
  默认检查（绝不补充它，因此更严格的引擎可以接受探针会拒绝的语法）。本包保留形态检查、空源拒绝、
  源枚举、await 回调，以及把任何抛出包进字段专属诊断（`` `mermaid` `` / `` `options[N].mermaid` ``），
  提示模型更正并重试。`checkFile: (path, cwd) => void | Promise<void>` 是文件预览的同一接缝
  （每个 `path`/`before` 都被包进一个 `` cannot show `path` `` 诊断，提示模型先写文件）；
  没有它时路径不被检查，因为本包保持不依赖 `node:`。`scratchDir` 指明 prompt 指南把 mockup 送到哪里。
- **工具契约**，渲染器以它为键：名称 `"visualize"`、其参数形态
  （`type: "diagram" | "comparison" | "file"`、`title?`、`mermaid?`、`options?[]`、`path?`、`before?`），
  以及 `details` = 校验后的 params。改动这个形态对每个渲染器都是破坏性变更，包括 ThinkRail 的。

## 决策（Decisions）

- **文件预览引用工作区文件；它们从不携带内容。** `type: "file"` 取一个工作区路径
  （外加一个用于前/后对比的可选 `before` 路径，当它指向同一个文件时被拒绝），而不是内联 HTML，
  因此宿主用它既有的文件渲染器渲染它，评审意见也锚定到一个真实文件，随后由 agent 就地编辑。
  另一面是：被渲染的文件永远是它的当前内容：在一个 mockup 上迭代会更新所有更早的卡片，
  因此指南告诉 agent 复制一个它希望持续可见的提案，并把该副本作为 `before` 传入。该工具唯一的那条
  `promptGuidelines` 要求 agent 写一个自包含的 HTML mockup（内联 CSS、`data:` 图片、
  用样式化元素画出的控件，因为宿主可能剥掉脚本、表单控件与外部资源），并在它提议或询问 UI 变更时
  以这种方式展示它。面向模型的结果与终端都是一行，指明该路径；终端用户用浏览器打开它们。
- **对原生用户而言依赖极轻：`lovely-mermaid`。** 本包只依赖 `lovely-mermaid`
  （零依赖、约 1 MB、同步、Apache-2.0）。它是 `grok-mermaid` 的维护中的后继，而 `grok-mermaid` 正是
  pi 自己在聊天消息中绘制 ```` ```mermaid ```` 围栏所用的引擎 —— pi 从不把它施加到工具结果上，
  因此有了本包自己的渲染器 —— 于是一个 `visualize` 图示看起来就像模型内联写出的那个。
  `mermaid`（连同其依赖树约 83 MB）以及它在 Node/Bun 下需要的 `linkedom` DOM 垫片在此被否决：
  它们换来严格解析却没有终端渲染，而一个用 mermaid 渲染的宿主可以通过该接缝注入那个解析器。
- **复盘：`beautiful-mermaid` 被替换了。** 它的 ASCII 引擎（`@vercel` 与 `@ktrysmt` 两个 fork 共用）
  把一对节点之间的每条边都路由到单条路径上，因此一个往返或平行的对 —— 常见的请求/响应或
  `Idle ⇄ Running` 形状 —— 会叠印标签、丢掉一个，或丢失每一条的走向，而折叠后的 TUI 把那张图
  毫无提示地显示出来。在一次对 OAuth 流程的原生 pi 实机检查中发现；它连同 `elkjs` 还要花掉约 10 MB。
- **默认校验是渲染器自己的解析，尽力而为。** 对于 `lovely-mermaid` 绘制的那些种类
  （flowchart/graph、state、sequence、class、ER、pie、mindmap、timeline、gitGraph —— 其 `diagramKind`，
  会跳过 frontmatter、指令与注释），探针会拒绝未知的 flowchart 方向（`lovely-mermaid` 静默接受
  `flowchart XX`；mermaid 不会）、一个什么都没解析出来的源（`render` → `null`），
  以及解析**丢弃**的任何片段（`warnings`：悬空链接、未闭合标签、无法读取的语句）。
  尺寸上限警告（`diagram truncated`）是渲染器限制，不是源错误，永不构成拒绝。该库建议不要用 warnings
  门禁*渲染*，因为流式传输中的源在编辑中途会告警；而一次工具调用是完整的，因此那里被丢弃的片段是
  值得重试的真实缺陷。被接受的代价：语法宽松（未闭合的 class 体也能通过），且有罕见的误报
  （`accTitle`/`accDescr`）；拥有严格解析器的宿主通过该接缝替换探针。其他族（gantt、xychart、……）
  **原样通过、不做校验**，而不是冒险拒绝一个渲染器只是不认识的图示类型。探针与 TUI 共用一次渲染调用，
  该调用先剥掉前导空行与 `%%` 行：`lovely-mermaid` 0.3.3 对注释或 `%%{init}%%` 指令之后的 mindmap
  返回 `null`（其他种类不受影响）。
- **终端渲染分级。** `renderCall` 是一行摘要（`visualize <title | diagram | comparison — N options>`）。
  `renderResult` 返回一个宽度感知的组件：在 `render(width)` 中，只有当图**完整**（无 warnings）
  且每一行都符合宽度时，才用 pi-tui 的 `visibleWidth` 绘制图示（它与渲染器的字素正确宽度一致，
  因此 CJK 与 emoji 标签也能绘制）；否则，以及对于渲染器不绘制的种类，显示源的 pi-tui `Markdown` 围栏。
  Span 角色完全按 pi 原生渲染器的方式映射到 pi 的主题（`border` → `borderMuted`、`text` → `text`、
  `edge` → `accent`、`edgeLabel` → `muted`）。图示永不做词换行。折叠 = 仅图；展开 = 图 + 源围栏。
  图示从防御性检查过的 `result.details` 渲染；对比、错误（`context.isError`）与缺失的 details
  从 `result.content` 渲染；部分结果显示一个占位符。图片级渲染（Kitty/iTerm2）被推迟：
  它需要不依赖浏览器的 SVG 光栅化，且在 tmux/SSH 下会退化。
- **双运行时。** 已交付代码使用不依赖 `node:` 的标准 ESM 与仅由 pi 宿主提供的包，因此它既能在
  Node（原生 pi、jiti）下运行，也能在 Bun（ThinkRail）下运行。对 `@earendil-works/pi-coding-agent`
  的运行时导入限于 `getMarkdownTheme`；`@earendil-works/pi-tui` 提供 `Text`、`Markdown`、`visibleWidth`。

## 边界（Boundary）

- **允许的依赖：** `lovely-mermaid`（精确固定版本）；对等依赖 `@earendil-works/pi-ai`（`StringEnum`）、
  `@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui`、`typebox`。
- **禁止：** 任何来自 ThinkRail 的东西（`@thinkrail/*`、`apps/*`、`packages/server`）、`mermaid`、
  `linkedom`、Bun 专属 API。

## 结构（Structure）

`index.ts`（导出 + 默认导出）、`src/extension.ts`（工厂、工具注册、校验器包装）、`src/schema.ts`、
`src/validate.ts`（形态 + 源枚举）、`src/markdown.ts`（第 1 级回退）、
`src/probe.ts`（共享渲染调用 + 默认校验器）、`src/tui.ts`（渲染器 + `DiagramComponent`）。
`bun test` 覆盖每一项；[[module-pi-extensions]] 中的原生一致性门禁演练打包后的产物。
