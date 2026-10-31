# Web 访问扩展

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

对第三方 pi 包 **`pi-web-access`**（`web_search`、`fetch_content`、`get_search_content`，
及其 `librarian` skill）的 ThinkRail 包装。该包被原样组合，它宣告的一切都会加载，如同 `pi install`
所为。作为一个注册表扩展，正是这一点让 Settings → Extensions 能把它作为一个整体关掉：
父会话与 subagent 子会话的工具，以及那个 skill。

## 边界（Boundary）

一个入口点 `@thinkrail/ext-web-access/server`，默认导出
`defineServerExtension({ name: "web-access", title, description, extensions: [piWebAccess], childExtensions: [piWebAccess], skillPackages: ["pi-web-access"] })`。
它可以导入 `extension-api/server`、`pi-web-access` 与 pi 类型；永不导入宿主内部。目前没有 `./web`：
三个工具的聊天卡片仍留在应用本地（[[submodule-web-chat-tools-web]]），按工具名加入，
因此以后迁移它们不改变线上契约的任何东西。

- **不在这里：无头搜索策略。** 把 `web_search` 的 `workflow` 默认为 `"none"`（否则该包会打开它的浏览器
  curator，而宿主机器可能根本显示不了）补偿的是宿主的 UI，不是本包，因此宿主拥有它并无条件组合它
  （[[submodule-server-agent]]）。它必须在本扩展关闭期间继续生效：那正是用户自己安装的
  `pi-web-access` 运行的时候。
- **`piWebAccess.js` + `piWebAccess.d.ts`** —— 该包交付的原始 `.ts` 通不过仓库严格的 `tsc`；
  配对的声明把该 `.js` 标注为 `ExtensionFactory`，因此 `tsc` 永不进入该包。`.js` 惰性加载该包：
  在 Bun 下（开发、测试、编译后的二进制、桌面 bundle）经一个字面量
  `import("pi-web-access/index.ts")`，两个打包器都会包含它；在 Node 下 —— 只有 workflow 测试装置，
  它在那边加载 server barrel（[[module-workflow-tests]]）—— 经 jiti，置于一个非字面量 specifier 之后，
  使任何打包器都不会把 jiti 拉进来。Node 拒绝在 `node_modules` 下剥离类型，因此一个普通导入会破坏那个
  测试装置。一旦该包通过严格检查或交付类型，就换成普通导入。
- 固定的版本是本包的依赖；宿主不再依赖 `pi-web-access`。

## 不变量（Invariants）

- 关闭后，会话不加载其中任何东西：没有工厂（新加载）、它的工具被扣留（活跃会话），
  且它的 skill 文件夹被排除在 pi 的 skill 输入之外，于是 `librarian` 从会话、斜杠命令列表与
  Settings → Skills 中消失 —— 而用户自己的、本会被它遮蔽的 `librarian` 则改为被加载。
  一个已加载该 skill 的活跃空闲聊天会重载一次以丢弃它（[[submodule-server-agent]]）。
- 用户自己安装的 `pi-web-access` 继续可用：扩展开启时，注册表副本赢得工具名；关闭时，用户的副本运行。
  它自己的 `librarian` 属于用户，不被本开关拥有。

## 验证（Verification）

`server/index.test.ts` 固定描述符，以及被包装的工厂确实注册了该包的工具。组合、skill 所有权与实时开关
在 `packages/server/src/extensions` 与 `packages/server/src/agent` 的测试中覆盖。
