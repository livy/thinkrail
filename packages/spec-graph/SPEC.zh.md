# Spec-Graph pi 扩展

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

`pi-spec-graph` 是一个可移植的 pi-package，教会 `pi` agent 认识本项目的**规格图谱**，并让它能搜索、
导航与管理规格。它提供一个 **skill**、七个 **`spec_*` 自定义工具**，以及一条全项目生效的
**`before_agent_start` 规则**。它不依赖本 monorepo 中的任何东西，因此在原生 `pi` 下即可运行；
thinkrail 会把它打包进每个会话。

它管理仓库规格所用的 frontmatter 模式——只要文件的 frontmatter 带有 `id` 与 `type`，它就是一份规格；
模式本身记录在 skill 中。Frontmatter 用 `yaml` 库处理，链接/元数据列表为内联形式。`spec_create` 以
规范的字段顺序写入新 frontmatter；`spec_update` **就地**编辑既有文件——保留文件自身的字段顺序与任何
注释/嵌套字段，仅重写 frontmatter 块（沿用该块已有的换行符），且不改动正文的任何一个字节。

## 边界（Boundary）

- **拥有：** 规格模型（is-a-spec 规则、派生图谱、读索引、校验）、那七个工具，以及 skill +
  `before_agent_start` 规则。
- **公开接口：** 扩展入口 `index.ts`（默认导出 `ExtensionFactory`）与 `package.json` 中的 `pi`
  清单（`{ extensions: ["./index.ts"], skills: ["./skills"] }`）——即原生 `pi`（`pi install`）与
  thinkrail（`additionalExtensionPaths` / `additionalSkillPaths`）加载它的方式——外加
  **`pi-spec-graph/core` 导出**（不含 pi 的读模型），供宿主（thinkrail 的 server）在不经过 agent 的
  情况下消费。`tools/` 保持内部。
- **允许的依赖：** `@earendil-works/pi-coding-agent`、`@earendil-works/pi-ai`、`typebox`（peer）；
  `yaml`（frontmatter 解析/序列化）；Node 内置模块。
- **禁止：** 任何 `@thinkrail/*` 包——依赖边是单向的（thinkrail → 本包），这正是它保持可移植的原因。

## 子模块（Sub-modules）

`core/`（`submodule-spec-graph-core`）是不含 pi 的规格模型；`tools/`（`submodule-spec-graph-tools`）
是套在它之上的薄 pi 包装。这条边是单向的——`tools/` → `core/`，经由 core 的 barrel——而 `index.ts` 把
工具与 `before_agent_start` 规则接进扩展。每个子规格钉住自己的边界与叶子节点。

## 派生的读索引（Derived read index）

文件系统是真相来源；工具读到的模型是**派生的、内存中的、只读的，并按需重新校验**。每次读取都重跑一次
**glob**——一次按名字排序的遍历，忽略 `node_modules`/`.git`/`dist`/`build`，以及其他助手在项目里留下的
临时草稿目录（`.claude`、`.superpowers`、`.gsd`、`.mockups`、`.playwright-mcp`），它们的计划与生成的
skill 带有形似 spec 的 frontmatter，但不是本项目的规格——并按 `(mtimeMs, size)` 重新校验每个文件：
未变的文件复用缓存的解析结果，变更/新增的文件重新解析，消失的文件被剔除，且只有当规格集真的发生变化时
才重建派生图谱。因此从任何来源新增、删除或编辑的规格——包括 pi 常规的 `write`/`edit`——总是最新的，
而在什么都没有变动时冗余的重新解析/重建被跳过。每个根复用一个 `SpecIndex`（按 cwd 为键），使缓存能在
一个 agent 的多次调用之间获益。唯一理论上的漏判是编辑落在同一 mtime 刻度内*且*保持字节长度不变
（可忽略；内容哈希是获准的升级方案）。

## 工具（Tools）

读——`spec_grep`（内容搜索，可按元数据收窄）、`spec_get`（某个节点的 frontmatter、解析后的链接与路径
——不含正文）、`spec_graph`（有界的 subtree/ancestors/neighbors 切片）。管理——`spec_create`
（它拒绝索引永远看不到的任何路径——见 `submodule-spec-graph-core` 中的 `resolveSpecPath`——以及任何
会写出一出生就不是规格的文件的参数）、`spec_update`（仅 frontmatter）、`spec_delete`、
`spec_validate`。各工具的用法记录在 skill 与每个工具的 `description` 中；**它们都不编辑正文**——
正文用 pi 常规的 `read`/`write`/`edit` 来写/改。

## 知识投递（Knowledge delivery）

概念、模式与工作流位于 **skill** 中（通过 `pi.skills` 清单 / `additionalSkillPaths` 自动发现）。
常开规则让规格保持权威，同时让查阅变得有上下文：当工作受某份记录在案的边界、契约、不变量、行为或架构
决策约束、或可能改变它时，查阅相关规格；局部化的工作无需阅读无关规格。该规则在 `before_agent_start`
中以 `systemPromptOptions.sections` 的 `pi-spec-graph` 条目贡献——就地修改，处理器不返回任何东西——
因此它被渲染成自己带 XML 标签的块，位于 `cwd` 之后，即旧的自由文本追加所在的尾部位置。返回
`systemPrompt` 会强制整个提示：任何变动都是缓存未命中，而具名 section 只是一块可 diff 的补丁。它也不是
`promptGuidelines` 条目——那会把这条多行规则压平成 pi 共享的 `rules` 列表。标签 = 包名：全局唯一，
且可移植的包不得携带宿主品牌。返回 `systemPrompt` 的第三方扩展会强制整个提示并丢弃每个结构化 section，
包括我们的——这是 pi 文档化的强制语义；接受它，而不是把规则重新追加到被强制的提示上。
每个工具带一个 `description`（其约束）与一行 `promptSnippet`（它在系统提示 Available-tools 列表中的
条目，与随包的 `pi-web-access` / `visualize` 工具一致）。这是通过扩展施行的 pi 原生提示影响，
不是宿主的提示组装。

## thinkrail 集成（thinkrail integration）

`packages/server/src/agent/extensions.ts` 在源码模式下把本包分层加入每个会话的
`DefaultResourceLoader`：`require.resolve("pi-spec-graph/index.ts")` 放入
`additionalExtensionPaths`，包的 `skills` 目录放入 `additionalSkillPaths`。服务端只通过解析后的路径
引用它（无值导入），因此它不进入服务端的 typecheck 图。另外，`packages/server/src/spec/` 值导入
**`pi-spec-graph/core`**（不含 pi 的模型——该子树中没有任何 pi 包）以通过线上通道提供只读的规格查看器
——与 agent 工具使用同一个 is-a-spec 规则、解析器与读时重新校验的 `SpecIndex`。

## 不变量（Invariants）

- 依赖边是单向的：thinkrail 依赖本包；本包不依赖 monorepo 中的任何东西，且 `core/` 不导入任何
  `@earendil-works/*`。
- 索引是建立在文件系统（真相来源）之上的派生读缓存；它不持有权威状态——状态归 `pi` 所有——且绝不提供
  过期图谱。
- 工具绝不编辑正文；`spec_update` 仅改 frontmatter、无损（保留注释/嵌套字段），且绝不把文件变成非规格。

## 非目标（Non-goals）

本包内的 UI（thinkrail 的规格查看器从外部消费 `core/`）、语义/嵌入搜索、关联代码的 frontmatter 字段、
孤立目录检测，以及移动/重命名规格。
