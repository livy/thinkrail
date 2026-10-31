# extensions —— ThinkRail 扩展的 server 注册表

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

宿主列出它所组合的 ThinkRail 扩展的唯一场所（[[module-thinkrail-extensions]] § 组装）。`registry.ts`
静态导入每个 `@thinkrail/ext-*/server`，并导出有序条目 `{ specifier, extension }` ——
`defineServerExtension` 描述符**以及**它作为字符串的公开 server specifier。静态导入经普通 server 图把
工厂带进编译后的 CLI 二进制与桌面 server-runtime 包，因此打包器无需为注册表扩展生成工厂清单。

## 边界（Boundary）

- **拥有：** `serverExtensions`（有序条目）、`registryInlineExtensions(kind, isDisabled?, entries?,
  session?)`（把 `extensions` / `childExtensions` 摊平成以扩展命名的 pi `InlineExtension` —— 一个
  工厂保留裸名，多个则得到 `name#n` —— 因此 pi 诊断显示 `<inline:visualize>` 而不是一个位置；每个
  工厂在 **pi 加载它时**询问 `isDisabled(name)`，因此重载会重新决定成员关系，无需重建 loader。给定
  `session` 的父级摊平会在静态工厂之后、以同样命名追加 `sessionExtensions(session)`；它每次摊平调用
  一次，因此每个父会话的工厂闭包住它自己的服务。没有 session 时，以及在子集合中，会话工厂缺席）、
  `omitDisabledRegistryExtensions(result, isDisabled)`（从一个 pi `LoadExtensionsResult` 中丢弃一个被
  禁用扩展的 `<inline:…>` 条目，用于父 loader 的 `extensionsOverride`，因此被禁用的扩展是缺席而不是
  在场且空）、
  `preferRegistryTools(result)`（也组装进那个 override，位于被禁用过滤之后）。它把一个注册表扩展直接
  移到第一个注册了它某个工具名的非注册表扩展之前，因此用户自行安装的同名包永不会接管一个其宿主 UI
  与生命周期以名字为键的工具 —— `ask_user_question` 就是原因。另一个包的其他工具、命令与钩子仍被
  加载。pi 自己的冲突诊断像以前一样留在 loader 结果里，不被报告。它所依赖的 pi 事实登记在
  [[pi-compatibility]]）、
  `registryExtensionName(path)`（把一个 `<inline:name>` / `<inline:name#n>` 路径 —— 一个扩展或其工具的
  `sourceInfo.path` —— 按名字映射回拥有它的注册表扩展，否则 `undefined`；按名字而非重算名字匹配，
  使它独立于一个会话得到了多少个逐会话工厂）、
  `listExtensionCatalog(disabled)`（Settings 目录：每条目 `name`、`title` —— 回退到 `name` —— 可选
  `description`、`hasChildExtensions`、`enabled = !disabled.has(name)`，以及它的**父**工厂对着一个
  记录桩 `ExtensionAPI`（其其他每个成员都是惰性的）所注册的工具名。为此用惰性 session 构建逐会话
  工厂，其 `awaitToolResponse` 会拒绝。工具每进程只计算一次，永不自在线会话计算，而一个对着桩抛错的
  工厂会保留它在抛错之前记录的工具）、以及 `resolveExtensionSkillRoots(entry)`（按所有者作用域的技能
  解析器：先由 `specifier` 得到扩展的入口文件，然后经 `createRequire(extensionEntry)` 得到每个
  `skillPackages` 清单，返回其 `pi.skills` 目录；一个被点名但没有技能的包会响亮失败）、
  `STAGED_HOST_SKILLS_ROUTE`（`host`）与 `stagedExtensionSkillRoute(name)`（`extensions/<name>`）——
  宿主与扩展的根在一个 stage 后的技能根下所暂存的文件夹，由构建支持与 runtime 共享，因此一个被禁用
  扩展的文件夹可以留在 pi 的技能输入之外，以及 `extensionSkillRoots(entry, stagedSkillsDir?)`（拥有
  一个扩展技能的那些根：无 staged 目录时是解析出的源根，有它时是那个所有者文件夹，对没有
  `skillPackages` 的扩展则什么也没有）。specifier 被保留为字符串，正是因为描述符持有函数，而函数
  不产生模块身份。
- **公开接口（barrel）：** `serverExtensions`、`ServerExtensionEntry`、`registryInlineExtensions`、
  `RegistryExtensionKind`、`ExtensionDisabled`、`omitDisabledRegistryExtensions`、
  `preferRegistryTools`、`registryExtensionName`、
  `listExtensionCatalog`、
  `resolveExtensionSkillRoots`、`STAGED_HOST_SKILLS_ROUTE`、`stagedExtensionSkillRoute`、
  `extensionSkillRoots`、`interactiveTools`、`InteractiveTools`、`ExtensionSessionServices`、
  `InteractiveTool`、`ToolResponse`。
- **会话接缝类型：** `ExtensionSessionServices`、`InteractiveTool` 与 `ToolResponse` 从
  `extension-api/server` 再导出，好让 `agent` 能通过本 barrel 为它构建的服务做类型标注。
- **交互式工具（`interactive.ts`）：** `interactiveTools(isDisabled)` 把已启用的注册表扩展的
  `interactiveTools` 声明折叠成一个 名字 → `InteractiveTool` 映射（同名的第一个声明胜出）。它是
  `agent` 为重启修复、由转录派生的 needs-input 与迟到响应措辞所持有的唯一工具知识；
  `packages/server` 里没有任何东西点名一个扩展的工具，或在 `registry.ts` 之外导入一个扩展。宿主知道
  的唯一工具名是 `web_search`，位于 `agent` 的无头搜索策略中：它为任何一份 `pi-web-access` 补偿宿主的
  UI，因此它不是该扩展该拥有的。
- **允许的依赖：** `@thinkrail/extension-api/server`（类型）、每个 `@thinkrail/ext-*/server`、
  `@earendil-works/pi-coding-agent`（仅类型）、`@thinkrail/contracts`（`ExtensionCatalogEntry` 类型）、
  Node 的 `module`/`fs`/`path`。
- **禁止：** `host`；`settings`（被禁用集合作为参数到达）；兄弟特性；从宿主自己的 `createRequire`
  解析扩展拥有的包（宿主不依赖一个扩展所组合的 pi 包，因此那种解析在隔离布局下会失效）。

## 消费方（Consumers）

`host` 用 `listExtensionCatalog` 对着 `disabledExtensions` 设置回答 `extension.list`。
`agent/extensions.ts` 把摊平后的父集合组合进每个资源 loader，把子集合组合进精心挑选的隐藏子扩展，
在 dev 与打包模式下同样如此，并把一个被禁用扩展的 `extensionSkillRoots` 留在 pi 的技能输入之外；
`buildSupport.ts` 把每个扩展的技能根连同它们的 `stagedExtensionSkillRoute` 交给两个打包器；未打包的
dev 在 runtime 解析同样的根，而打包后的 runtime 只使用启动器 stage 的那个根。
