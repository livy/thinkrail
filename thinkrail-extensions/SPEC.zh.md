# ThinkRail 扩展（thinkrail-extensions/）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

一个 **ThinkRail 扩展**是工作区包 `thinkrail-extensions/<name>/`（`@thinkrail/ext-<name>`，私有），
它把一项 pi 能力——通常是一个 [[module-pi-extensions]] 包，也可能是第三方 pi 包——与其 ThinkRail 专属
半边组合起来：即宿主打包进会话的内容，以及 web 客户端如何呈现该能力的工具。宿主**每侧一个注册表文件**
地组合扩展。用户在“设置 → 扩展”中看到服务端注册表，并可以逐个关闭扩展（一个以描述符 `name` 为键的
宿主级设置；扩展无需为此声明任何东西）；用户如何安装或发现新扩展是刻意未设计的。

第一个扩展（`visualize`，[[module-ext-visualize]]）交付两个半边；宿主通过下面两个注册表文件组合它们
（[[submodule-server-extensions]]、[[submodule-web-extensions]]）。安装体验、市场、按扩展的设置、
按工作区的开关覆盖、扩展贡献的传输线方法、面板与 store 贡献都是显式的延后项，而不是缺口。
开关覆盖扩展贡献的一切——它的工厂、它在活动会话中的工具，以及它 `skillPackages` 的 skills——
就像在 pi 中移除一个包会同时丢弃它的扩展与 skills。

## 形态——两个入口点，无根 barrel（Shape — two entrypoints, no root barrel）

- `./server`——默认导出 `defineServerExtension({ name, title?, description?, extensions, sessionExtensions?, interactiveTools?, childExtensions?, skillPackages? })`
  （`title` / `description` 是设置页的显示文案）。
  `extensions` / `childExtensions` 是宿主组合进每个父会话 / 精选子集会话的 pi `ExtensionFactory`，
  它们注册为具名 `InlineExtension`，以便 pi 诊断保留身份。
  `sessionExtensions(session)` 返回基于某个会话的宿主服务构建的更多父工厂；
  `interactiveTools` 列出其调用需要等待人的工具，并可选地说明迟到的应答如何呈现给模型
  （[[module-extension-api]]）。
  `skillPackages` 是**包说明符**，宿主暂存其 `pi.skills` 目录——这是构建步骤的显式模块身份，
  绝不从函数推断。仅服务端；可以以值方式 import pi。
- `./web`——默认导出 `defineWebExtension({ name, toolRenderers })`，以工具名为键，**外加命名导出**
  供宿主直接使用（visualize 为聊天 markdown 中的围栏 mermaid 导出 `MermaidView`）。
  仅浏览器；绝不以值方式 import pi；重型渲染器保持惰性加载。

`define*` 对象与 `./web` 的命名导出就是全部契约：`./web` 的命名导出是宿主直接渲染的组件。
`./server` 没有宿主消费的命名导出——宿主为某项能力生命周期所需的一切都声明在描述符上
（`interactiveTools`），因此核心永远只在其两个注册表文件中 import 扩展，并且删除一个扩展无需触碰核心。

未来的能力（传输线方法、面板、设置）在出现真实消费方时添加到这些对象上。

**现在固定下来、使后续扩展永不被迫重写的性质——运行时/生命周期归属：** 分离的服务端与浏览器入口点；
不 import 宿主内部构件；宿主拥有的作用域状态（按会话所有者，例如 subagents 与 background commands；
按工作区服务）通过显式接缝提供，绝不由扩展创建。因此按会话所有者留在宿主的会话管理器中。
扩展只能通过 `sessionExtensions(session)` 这个显式的按会话接缝触达它们的服务。描述符本身保持静态。

## 宿主 UI SDK（Host UI SDK）

`./web` 半边可以 import 什么：

- `@thinkrail/extension-api`（[[module-extension-api]]）——只有类型与 `define*` 辅助函数，没有 React 组件。
  `./web`：`ToolRenderProps`、`ToolRenderer`、`ToolRegistrationOptions`、`ToolStatus`、`WebExtension`、
  `defineWebExtension`，以及纯渲染器辅助函数（`toolHelpers`、`toolResultContent`）。
  `./server`：`ServerExtension`、`defineServerExtension`。它自成一个包，因为服务端 pi 类型会污染
  `contracts` 中的传输线边界，而 `ui` 中的服务端契约会让宿主获得一个呈现层依赖。
- `@thinkrail/ui`（[[module-ui]]）——整体搬来的自有 shadcn/Radix 原语、`cn`，以及仅 `onThemeSwap` 函数
  （不是主题目录/偏好运行时）。为代码分割保留按文件 import（`@thinkrail/ui/dialog`）。
  设计 token 留在 `apps/web/src/styles`；SDK 与扩展 web 半边只使用 token 工具类，
  而应用的 CSS `@source` 与颜色/间距/排版检查覆盖 `packages/ui` 与 `thinkrail-extensions/*/web`。
- `@thinkrail/resource-renderers`（[[module-resource-renderers]]）——宿主的文件与 diff 渲染器：
  `ResourcePreview` 用与 diff 审查相同的渲染器和审查评论展示一个工作区文件（或两者的前后对比）；
  `useResourceScope` 提供聊天的工作区与会话，`useOptionalResourceHost` 提供宿主的标签页操作。
  扩展自己从不加载文件或保存评论；宿主通过 provider 提供两者。
- `@thinkrail/locales/react`（[[module-locales]]）——基于唯一共享 i18next 实例的 `useT` / `<T>` / `t`；
  扩展的 UI 文本是 `packages/locales/en.json` 中其自有前缀（`visualize.*`）下的键，与 app 一样由
  `check:i18n` 把关。
- **（暂时）不在 SDK 中：** 高亮的 `CodeBlock` 与 shiki 高亮器——依赖未闭包（高亮器 → 主题 barrel）。
  当第二个扩展需要高亮时，它们会与通用的 CSS 变量 shiki 映射一起迁移。

## 组合——注册表文件（Composition — the registry files）

- **服务端：** `packages/server/src/extensions/registry.ts` 静态 import 每个 `@thinkrail/ext-*/server`
  并导出条目 `{ specifier, extension }`——即描述符**及其公开服务端说明符字符串**。静态 import 让工厂
  经由正常的服务端图进入编译后的 CLI 二进制与桌面打包产物，因此打包器不为它们生成工厂列表
  （生成的列表只留给 `pi-spec-graph`、`pi-thinkrail-workflow` 和 `pi-todos`，它们尚未被包装为扩展）。
  资源加载器在开发与打包模式下同样把 `extensions` / `childExtensions` 展平为具名内联扩展——
  一种组合模式，没有基于路径的开发加载。**skill 解析按所有者限定：** 一个共享解析器从 `specifier`
  解析扩展的入口，然后通过 `createRequire(extensionEntry)` 解析每个 `skillPackages` 清单——
  非打包的开发在运行时解析；构建支持把扩展的根暂存到 `skills/extensions/<name>/`，
  与宿主暂存到 `skills/host/` 下的根并列（pi 会递归发现 skills），而打包后的运行时把扩展映射到
  注入的暂存根的那个文件夹，因此**归属在展平后依然存在**，开关可以精确丢弃该扩展的 skills。
  启动器基础设施（桌面端的打包服务端运行时、`startDesktopHost`、`registerBundledRuntime` 的
  OAuth/Bedrock/回收站注册）不因此改变。
- **Web：** `apps/web/src/extensions/registry.ts` 导出有序的 web 扩展；
  `registerWebExtensions()` 把它们的 `toolRenderers` 送入聊天工具注册表。应用本地的工具注册保留其
  副作用 import；扩展描述符不需要任何副作用 import。

## 边界规则（由 `scripts/check-module-boundaries.ts` 强制，含负向测试）

- 发现 `pi-extensions/*` 与 `thinkrail-extensions/*` 作为工作区根。
- **源码半边：** 在每个扩展内部，`web/**` 与 `server/**` 绝不互相 import，包括通过相对路径。
- **公开子路径：** `apps/web` 只能 import `@thinkrail/ext-*/web`；`packages/server` 只能
  `@thinkrail/ext-*/server`。
- 边：`extension-api → contracts`；`ui → contracts`；`resource-renderers → contracts, ui`；
  `ext-*/web → ui, extension-api, resource-renderers, contracts`，外加对其自身 pi 包的**仅类型** import
  （参数/结果形状属于该包；既有的浏览器规则仍拒绝任何 pi 包的值 import）；
  `ext-*/server → extension-api, its pi package`；
  `apps/web → contracts, ui, extension-api, resource-renderers, ext-*/web`；
  `packages/server → extension-api, ext-*/server`（并且不再直接依赖扩展拥有的 pi 包）。

## 交付模板（我们每个扩展、每个 PR 对用户不可见）（Delivery template）

1. **在旧包旁边新建 pi 包**，并发布；旧包与宿主接线不动。
2. **web 半边迁移**到 `thinkrail-extensions/<name>/web`，由 web 注册表注册；服务端仍运行旧包——
   两者注册同一个工具名，因此渲染不变。
3. **服务端半边 + 接线**：`./server` 组合新包，服务端注册表接收它，旧的包被删除。

PR 1 与 PR 2 相互独立；PR 3 依赖两者。一个没有 web 半边的*新*扩展在第 2 步之前通过默认工具渲染器渲染
——绝不是 visualize，它的渲染器在前后都存在。

**包装第三方 pi 包**没有第 1 步（我们没有东西要发布）。它的服务端半边单独落地，原样组合该包外加任何
ThinkRail 策略，同时应用本地渲染器继续渲染相同的工具名；当渲染器的依赖进入 SDK 后，web 半边再跟上。
一个无法通过仓库严格 `tsc` 的原始 `.ts` 包，通过一个配对的声明式 JavaScript re-export 来 import
（`piWebAccess.js` + `.d.ts`）：打包器与 Bun 跟随 `.js`，`tsc` 止于声明文件。

## 成员（Members）

| 扩展 | pi 能力 | 备注 |
| --- | --- | --- |
| `visualize` | `@thinkrail/pi-visualize` | 已交付。`./server` 注入**严格**的 mermaid 校验器（`mermaid` 使用 web 渲染器目录中的版本，通过 `linkedom` 初始化——这是必需的：DOMPurify 需要真实的 `document` 才能处理 flowchart/class/state/gantt/mindmap，已在 Bun 下验证）。`./web` 承载图表/对比卡片并导出 `MermaidView`。保证的是语法解析，而不是 SVG/布局成功；浏览器渲染器保留其回退。 |
| `ask-user-question` | `@thinkrail/pi-ask-user-question` | 已交付；第一个交互式工具。`./server` 仅限会话：`sessionExtensions` 把可移植工具与 ThinkRail 的措辞、以及一个基于宿主交互式工具邮箱（`awaitToolResponse`）的 asker 组合起来；`interactiveTools` 声明 `ask_user_question`，并用 `describeResponse` 以该包自己的信封措辞呈现迟到的回答。`./web` 注册问卷卡片 `interactive`，完全由宿主通过 `ToolRenderProps.interaction` 供给；计划页像聊天一样通过工具注册表渲染它。[[module-ext-ask-user-question]]。 |
| `web-access` | 第三方 `pi-web-access` | 已交付（服务端半边）。`./server` 为父级与子级原样组合该包（宿主保留其无头搜索策略，该策略必须比开关活得更久）并加载其 `librarian` skill（`skillPackages`）；第一个带 skills 的扩展。渲染器保持应用本地（`apps/web/src/chat/tools/web`），直到 `Markdown`、`Collapsible` 和 `ToolFileLink` 加入 SDK。[[module-ext-web-access]]。 |
