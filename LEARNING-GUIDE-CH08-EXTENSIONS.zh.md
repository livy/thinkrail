# 第 8 章深度专题：扩展体系——能力、组合、开关与渲染

> 本专题是 [LEARNING-GUIDE.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE.zh.md) 第 8 章的展开版。
> 主指南讲的是“有两种扩展”；这里讲的是**四种容易混淆的扩展形态、扩展如何被组装、开关到底关掉什么、
> 交互式工具如何跨重启存活**。
>
> 阅读前提：第 3 章（三环）、第 6 章（服务端）、第 7 章（Web 客户端）。
> 本章的每个结论都能在 owning `SPEC.zh.md` 与具体源文件里找到落点。

---

## 8.0 先分清“扩展”的四种形态

### 8.0.1 四种形态，不要混用同一个词

| 形态 | 位置 / 身份 | 谁拥有 | 能在原生 pi 跑吗 |
| --- | --- | --- | --- |
| **可移植 pi 包** | `pi-extensions/<name>`（`@thinkrail/pi-<name>`，发布到 npm）<br>`packages/pi-*`（工作区内部/尚未迁移） | 包自身 | **能** |
| **ThinkRail 扩展** | `thinkrail-extensions/<name>`（`@thinkrail/ext-<name>`，私有） | 扩展的 `./server` + `./web` | 不能（需要宿主半边） |
| **应用本地工具渲染器** | `apps/web/src/chat/tools/...` | `apps/web` | 不适用 |
| **MCP server** | `.pi/mcp.json` / Settings › MCP | pi 的 MCP 引擎 + `agent` | 能（pi 原生能力） |

另外还有一类“用户自己安装的 pi 包”（`packages.*` / `pi install`），它既不是注册表扩展，也不是可移植包；
它由 pi 的资源加载器按配置加载。

一句话：**能力可以可移植，呈现是宿主专属，组合是注册表的事。**

### 8.0.2 五个身份

| 身份 | 由谁定义 | 作用域 | 用途 |
| --- | --- | --- | --- |
| `name` | 扩展描述符 | 宿主注册表 | Settings 开关、`disabledExtensions`、诊断 |
| `specifier` | 注册表条目字符串 | 构建/解析 | `createRequire` 解析入口与 `skillPackages` |
| 工具名 | pi 工具 | 一个会话 | **能力与渲染的唯一连接键** |
| `toolCallId` | pi | 一次工具调用 | 交互式工具的邮箱、迟到响应 |
| `sessionId` | pi | 一个聊天 | 会话级工厂（`sessionExtensions`） |

**`name` 不是模块身份，`specifier` 才是。** 描述符里装的是函数，函数不产生模块身份；
所以注册表必须同时保存 `specifier` 字符串，技能解析才能找到入口文件。

### 8.0.3 一张总图

```
可移植能力（pi 包）
  @thinkrail/pi-visualize        @thinkrail/pi-ask-user-question     第三方 pi-web-access
        ▲                                  ▲                              ▲
        │ 组合                              │ 组合                          │ 原样组合
  @thinkrail/ext-visualize/server   @thinkrail/ext-ask-user-question/server   @thinkrail/ext-web-access/server
        │                                  │                              │
        └──────────────┬───────────────────┴──────────────────────────────┘
                       ▼
        packages/server/src/extensions/registry.ts   （静态 import，有序条目）
                       ▼
        agent.buildResourceLoader → pi DefaultResourceLoader
                       ▼
        InlineExtension `<inline:name>` / `<inline:name#n>`（每个工厂）
                       │
                       │  注册工具名 "visualize" / "ask_user_question" / "web_search"
                       ▼
        apps/web/src/extensions/registry.ts  （静态 import，有序 web 扩展）
                       ▼
        chat/toolRegistry.registerToolRenderer(toolName, renderer)
                       ▼
        工具卡 / 问卷卡 / 图表卡
```

两个注册表是**两条独立的边**，只通过工具名对齐。服务端不需要知道渲染器，渲染器也不需要知道服务端。

### 8.0.4 术语表

| 术语 | 含义 | 易错点 |
| --- | --- | --- |
| 描述符（descriptor） | `defineServerExtension` / `defineWebExtension` 的返回值 | 不是注册；`define*` 是恒等函数 |
| 工厂（factory） | pi `ExtensionFactory` | 可有多个，展平时得到 `name#1`、`name#2` |
| 内联扩展 | pi 的 `InlineExtension` | 诊断里显示 `<inline:visualize>` |
| 父会话 / 子会话 | 主 chat / subagent | `childExtensions` 只给子会话 |
| 邮箱（mailbox） | 宿主对交互式工具响应的跟踪 | 不是 pi 的 steer 队列 |
| 暂存技能根 | 打包后的 `skills/extensions/<name>/` | 归属必须保留，否则开关无法精确丢弃 |

---

## 8.1 可移植 pi 包：在原生 pi 里也能工作

工作区根规格：[pi-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-extensions/SPEC.zh.md)。

### 8.1.1 身份与分发

- npm 名是 `@thinkrail/pi-<name>`，目录是 `pi-extensions/<name>/`；
- 私有工作区包共享 scope 但 `private: true`（防止名字被占）；
- 最初发布在过渡组织 `thinkrail.ai` 下的包已弃用并指向这里；
- scope 是刻意的：无 scope 的 `pi-*` 会冲突（例如第三方的 `pi-subagents`），
  而 scope 会在 `pi.dev/packages` 画廊中标注归属。

**两种形态**：

| 形态 | 标志 | 例子 |
| --- | --- | --- |
| 可安装的 pi 扩展 | `pi` 清单（`extensions`、可选的 `skills`、`image`）+ `pi-package` 关键字 | `@thinkrail/pi-visualize`、`@thinkrail/pi-ask-user-question` |
| 可移植库 | 只有命名导出；没有清单、没有关键字、绝不是空操作工厂 | `pi-delegation` |

`pi-extensions/visualize` 的清单就是：

```json
{ "pi": { "extensions": ["./index.ts"] } }
```

### 8.1.2 原始 TypeScript，无构建步骤

pi 通过 jiti 加载入口点，所以包直接交付 `.ts`。`files` 是包级白名单，覆盖入口点与 `exports` 传递性 import
的一切（测试除外）；`exports` 保留既有消费方使用的每个子路径（例如 `./core`）。

pi SDK 包是 `peerDependencies: "*"`；有 TUI 渲染时 `@earendil-works/pi-tui` 也是 `"*"` peer。
被测试的 pi 版本是根目录版本，绝不在这里重复声明。

### 8.1.3 原生一致性门槛（Vanilla-parity bar）

这就是“在原生 pi 中可工作”的定义：

- 从**打包 tarball** 安装，依赖由 npm 解析，通过 pi 自己的解析器/加载器加载且**零诊断**，
  并注册其工具与 skills；
- **双运行时**：原生 pi 是 Node ≥ 22.19 + jiti 的 npm CLI；ThinkRail 在 Bun 下跑同一份代码。
  发布的代码只使用 `node:` 内建与标准 ESM——不得出现 `Bun.*`、`bun:*` 或 Bun 专属的 `import.meta`；
- **TUI**：在纯文本回退读起来很糟的地方提供 `renderCall` / `renderResult`；每个 UI 调用都在 `ctx.hasUI` 之后；
  并在真实终端（Ghostty/iTerm2，以及 tmux 内部）做一次手工检查；
- **无 ThinkRail 假设**：没有 `~/.thinkrail` 路径、没有宿主 wire、除兄弟可移植包外没有 `@thinkrail/*` import。

门禁 `bun run check:pi-packages` 做了什么：

```
打包包 + 每个待发的可移植工作区依赖
→ 在 tarball 中 grep 仅 Bun 可用的 API
→ 用 pi 自己的 npm 策略（--legacy-peer-deps）把 tarball 安装到仓库之外的临时 agent 目录
→ 配置 packages: ["npm:@thinkrail/pi-<name>"]
→ 把 parity runner 复制到夹具中并在 Node 下运行
→ DefaultResourceLoader.reload() 加载，断言零错误 + 预期工具/skills
→ 执行用例表（工具调用、文本/错误、renderResult 宽度、terminate 标志）
```

**为什么把 runner 复制到夹具**：Node 忽略 `import.meta.resolve` 的父 URL；
从脚本自身位置解析会静默地测试仓库里的 pi，而不是安装的 tarball。

### 8.1.4 发布：人工版本化 + Trusted Publishing

- `@changesets/cli` 锁定；`privatePackages: { version: false, tag: false }`，只有工作区根被版本化/打标签；
- 改动已发布包的 PR 要加 changeset；
- **版本化是人工步骤**：维护者运行 `bun run release:version` 并开普通 PR
  （JetBrains 组织策略禁止 Actions 创建 PR，普通 PR 还能拿到正常 CI）；
- 合并到 `main` 后，`release-pi-packages.yml` 在 `npm-release` 环境以 `id-token: write` 运行：
  重新跑一致性门禁 → `bun run release:publish`；
- 发布顺序是**依赖先于依赖方**；`bun pm pack` 把 `workspace:*` / `catalog:` 改写成精确版本；
- `npm publish <tgz> --provenance --access public`（Bun 没有 OIDC；npm ≥ 11.5.1）；
- 只有 `readyToTag=true`（没有待发 changeset、不是 dry run、每个公开版本都已上 npm）才 `changeset tag` 并推送。
  因此提前退出绝不会给占位版本 `0.0.0` 打标签。

**新包必须先手工引导发布**：Trusted Publishing 按“已存在的包”配置；引导时用 `--provenance=false`，
且受信发布者配置要在包创建 48 小时内的首次 OIDC 发布之后才验证——所以它在下一个版本 PR 合并前才创建。

### 8.1.5 成员与依赖

| 包 | 形态 | 提供 | 备注 |
| --- | --- | --- | --- |
| `pi-extensions/visualize` | 扩展 | `visualize` 工具 | 可用 `createVisualizeExtension({ validateMermaid })` 注入严格校验器 |
| `pi-extensions/ask-user-question` | 扩展 | `ask_user_question` 工具 + `./core` | `createAskUserQuestionExtension({ ask, announce, guidance })` |
| `pi-delegation` | 库 | `createChild`、运行所有权、fork/capture | 子会话织层；没有清单 |
| `pi-subagents` | 扩展 | `Agent` / `get_subagent_result` | 定义发现/优先级/信任；delivery 一次 |
| `pi-background-commands` | 扩展 | `background_command` 四动作 | 会话绑定；有界输出；完成投递 |
| `pi-spec-graph`（目录 `packages/spec-graph`） | 扩展 | 七个 `spec_*` 工具 + skill + 规则 | `core` 供宿主只读消费 |
| `pi-todos`（目录 `packages/pi-todos`） | 扩展 | 六个 `todo_*` 工具 + skill | `core` 供宿主计划查看器 |
| `pi-thinkrail-workflow` | 扩展 | workflow skill 家族 + 路由规则 | 不发布（工作区内部） |
| `pi-dag` | 扩展 + 服务 | 持久 DAG、证据、门禁 | 不捆绑进 ThinkRail，直到有消费方 |

兄弟依赖边只有：`subagents → delegation`、`dag → delegation`。

### 8.1.6 设计上的三个“不可移植”反例

| 反例 | 为什么禁止 |
| --- | --- |
| 用 `Bun.serve` / `Bun.spawn` | 原生 pi 是 Node + jiti |
| 假设 `~/.thinkrail` 存在 | 原生用户没有这个目录 |
| import `@thinkrail/contracts` 的运行时值 | 会把宿主 wire 拖进可移植包 |
| 注册工具却不提供 `renderCall`/`renderResult` | TUI 下退化成不可读文本 |
| 用函数推断 `skillPackages` | 构建步骤需要模块身份，函数不产生 |

---

## 8.2 ThinkRail 扩展：两个入口点，没有根 barrel

总规格：[thinkrail-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/thinkrail-extensions/SPEC.zh.md)。

### 8.2.1 形态

```ts
// ./server
export default defineServerExtension({
  name,                 // 身份：Settings 开关与 disabledExtensions 的键
  title?,               // Settings 显示名，回退到 name
  description?,         // Settings 描述
  extensions,           // 组合进每个父会话的 pi ExtensionFactory[]
  sessionExtensions?,   // (session) => ExtensionFactory[]，按会话
  interactiveTools?,    // 调用需要等待人的工具
  childExtensions?,     // 子会话使用的工厂
  skillPackages?,       // 需要暂存的 skills 包说明符
});

// ./web
export default defineWebExtension({
  name,
  toolRenderers,        // 工具名 → { renderer, options? }
});
```

`./web` 还可以有**命名导出**供宿主直接渲染（`visualize` 导出 `MermaidView`）。
`./server` 没有宿主消费的命名导出——宿主需要的一切都声明在描述符上（`interactiveTools`），
因此核心永远只在两个注册表文件里 import 扩展，删除一个扩展无需触碰核心。

### 8.2.2 每个字段的真实语义

| 字段 | 语义 | 边界 |
| --- | --- | --- |
| `name` | 注册表身份；`disabledExtensions` 的键 | 不是模块说明符 |
| `title` / `description` | Settings 目录文案 | 可选 |
| `extensions` | 每个父会话都组合的工厂 | 会以 `<inline:name>` 进入资源加载器 |
| `sessionExtensions(session)` | 用宿主持有的按会话服务构建更多父工厂 | 每个父会话调用一次，闭包住自己的服务 |
| `interactiveTools` | 工具名 → `{ describeResponse? }` | 宿主重启后迟到响应的措辞 |
| `childExtensions` | 只给精选子会话的工厂 | 子会话不加载 MCP；工具集是精选的 |
| `skillPackages` | 包说明符，宿主暂存其 `pi.skills` 目录 | **构建步骤的显式模块身份，绝不从函数推断** |

`sessionExtensions` 是扩展触达宿主状态的唯一方式：
宿主传入 `ExtensionSessionServices`（`expectToolResponse` / `awaitToolResponse`），
扩展不创建、不缓存、也不跨会话复用这些服务。

### 8.2.3 Host UI SDK：`./web` 能 import 什么

| 包 | 能用的部分 |
| --- | --- |
| `@thinkrail/extension-api` | 只有类型与 `define*` 辅助函数；没有 React 组件 |
| `@thinkrail/ui` | 自有 shadcn/Radix 原语、`cn`、`onThemeSwap`；按文件 import 以保留代码分割 |
| `@thinkrail/resource-renderers` | `ResourcePreview`、`useResourceScope`、`useOptionalResourceHost` |
| `@thinkrail/locales/react` | `useT` / `<T>` / `t`；字符串是 `en.json` 中扩展前缀下的键 |

**暂时不在 SDK 中**：高亮的 `CodeBlock` 与 shiki 高亮器（依赖未闭包）。第二个扩展需要高亮时再迁移。

### 8.2.4 边界规则（由 `check-module-boundaries` 强制）

- 每个扩展内部，`web/**` 与 `server/**` **绝不互相 import**（包括相对路径）；
- `apps/web` 只能 import `@thinkrail/ext-*/web`；`packages/server` 只能 `@thinkrail/ext-*/server`；
- 边：
  - `extension-api → contracts`
  - `ui → contracts`
  - `resource-renderers → contracts, ui`
  - `ext-*/web → ui, extension-api, resource-renderers, contracts` + 对自身 pi 包的**仅类型** import
  - `ext-*/server → extension-api, its pi package`
  - `apps/web → contracts, ui, extension-api, resource-renderers, ext-*/web`
  - `packages/server → extension-api, ext-*/server`（**不再直接依赖扩展拥有的 pi 包**）
- 负向测试存在于 `scripts/check-module-boundaries.ts`，不是口头约定。

### 8.2.5 交付模板：三个 PR

1. **新建 pi 包**并发布；旧包与宿主接线不动。
2. **web 半边迁移**到 `thinkrail-extensions/<name>/web`，由 web 注册表注册；
   服务端仍跑旧包——两者注册同一个工具名，因此渲染不变。
3. **服务端半边 + 接线**：`./server` 组合新包，服务端注册表接收它，旧包删除。

PR 1 与 PR 2 独立；PR 3 依赖两者。没有 web 半边的**新**扩展在第 2 步之前通过默认渲染器渲染。

**包装第三方 pi 包**没有第 1 步；服务端半边单独落地，原样组合该包 + 任何 ThinkRail 策略，
应用本地渲染器继续按同一工具名渲染。

> 关键点：**工具名是迁移期间稳定的**。三步之间没有任何线上契约变化。

### 8.2.6 成员

| 扩展 | pi 能力 | 半边 | 关键设计 |
| --- | --- | --- | --- |
| `visualize` | `@thinkrail/pi-visualize` | server + web | server 注入严格 mermaid 校验器（mermaid + linkedom）；web 承载图表/对比/文件卡并导出 `MermaidView` |
| `ask-user-question` | `@thinkrail/pi-ask-user-question` | server + web | 第一个交互式工具；`sessionExtensions` 组合可移植工具 + 宿主邮箱；web 注册 `interactive` 问卷卡 |
| `web-access` | 第三方 `pi-web-access` | 仅 server | 原样组合父/子工具 + `librarian` skill；**无头搜索策略归宿主**，不归扩展 |

`web-access` 的 `piWebAccess.js` + `.d.ts` 是一个值得记的接缝：

- 该包交付的原始 `.ts` 通不过仓库严格 `tsc`；
- 配对的声明把 `.js` 标注为 `ExtensionFactory`，因此 `tsc` 不进入该包；
- `.js` 在 Bun 下用字面量 `import("pi-web-access/index.ts")`（两个打包器都会包含）；
- 在 Node 下（只有 workflow 测试装置）经 jiti，放在一个非字面量 specifier 之后，
  使打包器不会把 jiti 拉进来。

---

## 8.3 注册表与组装：从描述符到 pi 内联扩展

### 8.3.1 服务端注册表只有三个条目

[registry.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/registry.ts#L1-L15)：

```ts
export const serverExtensions: readonly ServerExtensionEntry[] = [
	{ specifier: "@thinkrail/ext-visualize/server", extension: visualize },
	{ specifier: "@thinkrail/ext-ask-user-question/server", extension: askUserQuestion },
	{ specifier: "@thinkrail/ext-web-access/server", extension: webAccess },
];
```

两个字段：

- `extension`：描述符（工厂、交互声明、技能包）；
- `specifier`：**公开的服务端说明符字符串**，用于 `createRequire` 解析入口文件与技能包。

**静态 import 的意义**：工厂经普通服务端图进入编译后的 CLI 二进制与桌面打包产物，
因此打包器**不需要生成工厂清单**。生成的清单只留给 `pi-spec-graph`、`pi-thinkrail-workflow`、
`pi-todos`（它们还没被包装成注册表扩展）。

### 8.3.2 Web 注册表只有两行

[apps/web/src/extensions/registry.ts](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/extensions/registry.ts)：

```ts
export const webExtensions: readonly WebExtension[] = [visualize, askUserQuestion];

export function registerWebExtensions(): void {
	for (const extension of webExtensions) {
		for (const [name, { renderer, options }] of Object.entries(extension.toolRenderers)) {
			registerToolRenderer(name, renderer, options);
		}
	}
}
```

规则：

- 注册忽略宿主的扩展开/关设置：**服务端关闭后，web 渲染器仍注册**，
  因此历史转录里已有的工具调用继续渲染；
- 注册必须早于读第一份转录（`main.tsx` 在 `createRoot` 之前调用）；
- Map 归 `chat` 所有，本模块不创建第二份状态。

### 8.3.3 展平：`name` 与 `name#n`

[inline.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/inline.ts#L14-L43)：

```ts
const factories = factoriesOf(entry, kind, session);
return factories.map((factory, index) => ({
	name: factories.length === 1 ? name : `${name}#${index + 1}`,
	factory: async (pi) => {
		if (isDisabled(name)) return;
		await factory(pi);
	},
}));
```

要点：

- 一个工厂保留裸名；多个得到 `name#1`、`name#2`；
- `isDisabled(name)` 在 **pi 加载工厂时**询问，因此重载会重新决定成员关系，无需重建 loader；
- 诊断显示 `<inline:visualize>` 而不是一个位置路径。

`factoriesOf` 的分支：

| kind | 组成 |
| --- | --- |
| `extensions` | `extension.extensions` + 有 session 时的 `sessionExtensions(session)` |
| `childExtensions` | 只取 `extension.childExtensions`（**不含** `sessionExtensions`） |

### 8.3.4 两个 override：过滤与优先

[inline.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/inline.ts#L54-L87)：

| 函数 | 作用 | 为什么 |
| --- | --- | --- |
| `omitDisabledRegistryExtensions(result, isDisabled)` | 丢弃被禁用扩展的 `<inline:…>` 条目 | 让被禁用扩展**缺席**，而不是“在场但空” |
| `preferRegistryTools(result)` | 把注册表扩展移到同名工具的非注册表扩展之前 | 用户自己安装的同名包不能接管宿主 UI/生命周期以名字为键的工具 |

`preferRegistryTools` 的动机就是 `ask_user_question`：
如果用户自己安装了同名包，而宿主 UI 与重启修复都按名字绑定，那个包不能抢走它。
另一个包的其他工具、命令与钩子仍会被加载；pi 自己的冲突诊断也照常留在 loader 结果里。

### 8.3.5 `registryExtensionName`：按名字匹配，不重算

```ts
const match = /^<inline:([^#>]+)(?:#\d+)?>$/.exec(path);
return entries.find((entry) => entry.extension.name === match[1])?.extension.name;
```

它用于：

- `preferRegistryTools` 判断“这是不是我们的扩展”；
- `omitDisabledRegistryExtensions` 判断要丢弃谁；
- `applyExtensionTools` 判断某个工具的 `sourceInfo.path` 属于哪个扩展、
  从而在禁用时只扣留该扩展的工具。

**按名字匹配而不是重算工厂数量**，使它独立于一个会话得到了多少个逐会话工厂。

### 8.3.6 目录：对着桩记录工具名

[Catalog.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/catalog.ts)：

- `recordToolNames(factory)` 用一个 Proxy 记录 `registerTool` 的调用；
- `describeEntry` 立即调用工厂（不进入真实 pi API）；
- 工具名**每进程只计算一次**，永不对活跃会话计算；
- 一个对着桩抛错的工厂会保留它在抛错之前记录的工具；
- 逐会话工厂用一个惰性 session 构建，其 `awaitToolResponse` 会拒绝；
- 结果：`{ name, title, description?, tools, hasChildExtensions, enabled }`。

`host` 用 `listExtensionCatalog(new Set(getConfig().disabledExtensions))` 回答 `extension.list`。

### 8.3.7 交互式工具映射：第一个声明胜出

[interactive.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/interactive.ts#L7-L18)：

```ts
export function interactiveTools(isDisabled, entries = serverExtensions) {
	const tools = new Map();
	for (const { extension } of entries) {
		if (isDisabled(extension.name)) continue;
		for (const [name, tool] of Object.entries(extension.interactiveTools ?? {})) {
			if (!tools.has(name)) tools.set(name, tool);
		}
	}
	return tools;
}
```

它是 `agent` 持有的**唯一工具知识**，用于：

- 重启修复（哪些调用可能等待人）；
- 从转录推导 needs-input；
- 迟到响应的措辞（`describeResponse`）。

`packages/server` 里没有任何东西点名一个扩展的工具；宿主唯一知道的工具名是
`web_search`（在 `agent` 的无头搜索策略中，它补偿宿主的 UI）。

### 8.3.8 技能根：按所有者解析与暂存

[skills.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/skills.ts)：

```ts
export function resolveExtensionSkillRoots(entry) {
	const skillPackages = entry.extension.skillPackages ?? [];
	const extensionEntry = createRequire(import.meta.url).resolve(entry.specifier);
	const requireFromExtension = createRequire(extensionEntry);
	return skillPackages.flatMap((specifier) => {
		const manifestPath = requireFromExtension.resolve(`${specifier}/package.json`);
		const skills = JSON.parse(readFileSync(manifestPath)).pi?.skills ?? [];
		if (skills.length === 0) throw new Error(...);
		return skills.map((dir) => resolve(dirname(manifestPath), dir));
	});
}
```

两个关键点：

1. **从扩展自己的入口解析**，而不是从宿主的 `createRequire`。
   宿主不依赖扩展所组合的 pi 包，因此从宿主解析在隔离布局下会失败。
2. **一个被点名但没有技能的包会响亮失败**，不静默跳过。

暂存路由：

| 常量/函数 | 值 | 用途 |
| --- | --- | --- |
| `STAGED_HOST_SKILLS_ROUTE` | `"host"` | 宿主技能根 |
| `stagedExtensionSkillRoute(name)` | `"extensions/<name>"` | 扩展技能根 |

`extensionSkillRoots(entry, stagedSkillsDir?)`：

- 无暂存目录：解析源根；
- 有暂存目录：返回该所有者文件夹；
- 没有 `skillPackages`：返回 `[]`。

**为什么归属必须保留**：pi 会递归发现 skills；如果所有扩展技能混在一个根里，
开关就无法精确丢弃某个扩展的 skills。

### 8.3.9 dev 与 bundled 的差异

[agent/extensions.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/extensions.ts)：

| 模式 | 扩展路径 | 技能路径 |
| --- | --- | --- |
| dev | `require.resolve("pi-spec-graph/index.ts")` 等放入 `additionalExtensionPaths` | 包的 `skills/` + `../../skills` 放入 `additionalSkillPaths` |
| bundled | 工厂经 `registerBundledRuntime` 注入 | 启动器 stage 的 `skills/host` + `skills/extensions/<name>` |

`extensionSkillRootsByName()` 在 bundled 时用 `bundled.skillsDir` 映射到暂存文件夹；
dev 时缓存解析出的源根。

`buildResourceLoader` 组装：

```
registryInlineExtensions("extensions", isDisabledAtReload, serverExtensions, session)
+ headlessSearchPolicy
+ reviewToolExtension
+ requestReviewExtension
+ setTitleExtension
+ oversizedImageGuard
+ skillCarryover
+ extraFactories
```

`extensionsOverride` 依次做：

```
recordProviderOwners(result)
recordLoadedExtensions(result)
preferRegistryTools(omitDisabledRegistryExtensions(result, isDisabledAtReload))
```

---

## 8.4 开关：`disabledExtensions` 到底关掉什么

### 8.4.1 一条设置，四处效果

`AppConfig.disabledExtensions` 是宿主级设置，按描述符 `name` 为键：

```
settings.update({ disabledExtensions })
→ settings.changed 广播
→ host 发现 appliedUpdate.disabledExtensions
→ refreshExtensionTools()
```

[server.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/host/server.ts#L827-L833)。

`refreshExtensionTools()` 是 `agentSessionManager` 的入口；它只在一个禁用集合**实际变化**时
递增策略世代。

### 8.4.2 立即生效的部分

[applyExtensionTools](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L748-L782)：

1. 遍历 `session.getAllTools()`，用 `registryExtensionName(tool.sourceInfo.path)` 找所有者；
2. `isOff(tool)` = 所有者被禁用；
3. 把当前活动集中的禁用工具加入 `withheldExtensionTools`；
4. 重新启用时，**只恢复被扣留且默认活动的工具**，绝不恢复“默认就不活动”的；
5. `session.setActiveToolsByName([...kept, ...restored])`；
6. 若会话正在流式，推迟到 `agent_settled`；空闲则同步。

**`setActiveToolsByName` 会重建系统 prompt**，因此扣留工具同时也丢弃它的指导
（`promptGuidelines`）。这正是“关掉一个扩展”不仅仅是隐藏 UI 的原因。

### 8.4.3 延后生效的部分：钩子

pi **无法卸载一个已加载扩展**。因此：

- 已禁用扩展的钩子（`before_agent_start`、`session_start` 等）会继续在打开的会话里运行，
  直到下一次资源重载；
- 每次禁用都重载会击穿 prompt 缓存，所以只在两种情形重载：
  1. **会话从未加载过的已启用扩展**（在它关闭时创建的）且它并未加载失败；
  2. **会话已加载其 skills** 的已禁用扩展（`loadsDisabledExtensionSkills`）。

第二种是必须的：skills 活在系统 prompt 中，只有重载才能重建；
留着一个 skill 会把 agent 引向被扣留的工具。

无 skill 的扩展保持“不重载的禁用”。这是刻意的取舍：

> 工具立即消失；钩子留到下一次重载；skills 触发重载。

### 8.4.4 策略世代：避免无意义重载

`refreshExtensionTools` 只在禁用集合实际变化时递增世代；每次重载记录它启动时的世代，
完成时以“仅当世代移动才允许重载”的方式协调，并判定**新的**加载器。

效果：

- 一次未变更的重发不会带来额外开销；
- 重载期间一次无 skill 的开关不会导致第二次重载；
- 一次“禁用重载期间又启用”会重载，否则扩展会保持未加载，而 Settings 显示它是开的。

### 8.4.5 子会话与重启

| 场景 | 行为 |
| --- | --- |
| 新父会话 | loader 在每次重载时读禁用集合，被禁用的工厂直接跳过；`extensionsOverride` 省略条目 |
| 隐藏子会话 | `childExtensionFactories()` 读同一集合；`childExtensions` 只组合已启用扩展 |
| 宿主重启 | Settings 是事实来源；Pi ≥ 1.3 会恢复聊天记录的工具配置 |
| 重启后工具复活问题 | 挂接先重新激活“注册即激活”会激活的已启用注册表工具，然后扣留已禁用的 |
| 用户自己安装的同名包 | 扩展开时注册表副本赢；关时用户的副本运行 |

### 8.4.6 Web 侧：开关不影响渲染

`registerWebExtensions()` 忽略宿主设置。原因：

- 已存在转录里的工具调用必须继续渲染；
- 开关只决定宿主接下来给 agent 什么，不决定历史如何显示。

### 8.4.7 关闭一个扩展的五种资源

以 `web-access` 为例：

| 资源 | 关闭后 |
| --- | --- |
| 父会话工厂 | 新加载不组合 |
| 子会话工厂 | 新子会话不组合 |
| 活跃会话工具 | 从活动集扣留 |
| `librarian` skill | 从 pi 技能输入排除；从斜杠命令与 Settings › Skills 消失 |
| 用户自己的同名包 | 继续可用；它自己的 `librarian` 不被本开关拥有 |

一个已经加载该 skill 的空闲聊天会重载一次以丢弃它。

---

## 8.5 交互式工具：跨重启的人机回路

### 8.5.1 定义的三个条件

一个工具是**交互式**的，当且仅当：

1. 它的 `execute` 会等待人的回应；
2. 服务端描述符在 `interactiveTools` 里点名它；
3. web 渲染器注册时带 `interactive: true`。

宿主只从这两处声明得知工具名；它不按名字硬编码任何扩展工具。

### 8.5.2 邮箱的阶段

[toolResponses.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/toolResponses.ts)：

| 阶段 | 含义 |
| --- | --- |
| `expected` | 扩展在 `message_end` 里宣告了该调用 |
| `waiting` | 工具的 `execute` 已开始等待 |
| `response-accepted-uncommitted` | 用户已提交，等待工具结果持久化 |
| `superseded` | 用户发了自由消息，替换了空闲问题 |
| `stopped` | 运行被停止 |

三个 outcome：

| outcome | 含义 |
| --- | --- |
| `{ kind: "response", value }` | 人回答了 |
| `{ kind: "deferred" }` | 问题被推迟；工具返回 ack + `terminate: true` |
| `{ kind: "abandoned" }` | 会话被处置/停止 |

### 8.5.3 完整时序

```
pi 发出 message_end
→ 扩展 announce(toolCallId)
→ 宿主 expectToolResponse(toolCallId)
→ pi 运行工具 execute
→ 扩展 awaitToolResponse(toolCallId, signal)   // 挂起
→ 宿主状态推送 needsInput
→ web 卡片拿到 ToolRenderProps.interaction
→ 用户点击 Submit / 输入文本
→ session.respondToTool(toolCallId, response)
→ waiters.respond：accepted
→ 工具返回结果（details 必须深等于 accepted response）
→ persistTurn：resolvePersisted
→ 工具结果写入 pi 转录
```

**为什么结果 details 必须深等于 accepted response**：
`persistTurn` 用 `isDeepStrictEqual` 校验；不相等会 `rejectPersisted`，因此“已接受但未持久化”不会被误判成功。

### 8.5.4 重启修复：三种悬空调用

[sessionRepair.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/sessionRepair.ts#L17-L48)：

| 情形 | 写入的 toolResult |
| --- | --- |
| 可执行 + 交互式 | pending ack 文本，`details: { kind: "ack" }`，`isError: false` |
| 可执行 + 非交互式 | 通用 “Operation aborted (the host restarted…)” 错误 |
| 不可执行（被截断/中止） | “arguments may be truncated” 错误 |

待处理标记是两侧共读的唯一形状：`contracts` 的 `isToolResponsePending`。
工具不得把它用于真实结果。

**为什么不是错误**：一个交互式调用在重启后仍然可以回答；把它标成错误会让问题永久消失。
**为什么不是空结果**：空结果会让模型以为工具已经执行完。

### 8.5.5 迟到响应：`describeResponse`

宿主重启后，工具自己的 `execute` 已经消失。
响应通过一条 `tool-response` 自定义消息送达（旧转录是 `ask-user-answers`）。

`InteractiveTool.describeResponse?(args, response)` 让扩展把响应渲染成**面向模型的文本**：

- 返回 `undefined` 或没有钩子 → 宿主写一段通用前言 + JSON 响应；
- 实时响应永不经过它：工具自己格式化其结果。

`ask_user_question` 的实现：当 args/response 都有本包形状时，用 `buildQuestionnaireResponse`
渲染迟到回答，因此模型读到的措辞与实时回答一致。

### 8.5.6 可响应性判定

`assessRespondability(messages, toolCallId, interactive)` 返回：

| 结果 | 条件 |
| --- | --- |
| `unknown_call` | 转录里找不到该交互式调用 |
| `not_awaiting` | 调用已死（stopReason 不可执行）或已有真实结果 |
| `already_responded` | 已有 `tool-response` 自定义消息 |
| `superseded` | 调用之后出现了 user 消息 |
| `ok` | 以上都不成立 |

`awaitingToolCallId` 从尾部反向扫描，遇到 user 消息就停；它用于宿主状态里的 needs-input。

### 8.5.7 Web 侧：`accepting` 与不可提交

`ToolRenderProps.interaction.state.accepting` 只在：

- 宿主的 `needsInput` 点名了这次调用；**或**
- 尚无任何宿主状态到达（兼容窗口）

时为 true。因此：

- 另一个包的同名工具不会让本卡片可提交；
- 对已关闭工具的调用渲染但不可回应；
- 宿主未在等待的调用无法提交响应。

### 8.5.8 为什么交互 UI 在 chat 内，而不是模态框

登录对话框是例外（登录发生在 Welcome 界面，没有 chat）。
除了它，所有 pi 的 `select`/`confirm`/`input`/`editor` 都渲染为 chat 内联提示：

- 问题与它批准的内容在同一个框里；
- 发送一条 chat 消息就是逃生口；
- 不会出现“奇怪弹窗”，也不会把焦点从编辑器里拽走。

---

## 8.6 工具名：唯一连接键

### 8.6.1 服务端与 Web 的契约

```
pi 工具 "visualize"
   ↕ 同一个字符串
registerToolRenderer("visualize", VisualizeCard)
```

没有共享类型、没有共享对象、没有隐式约定；**只有工具名**。

`mcp__` 前缀是唯一的应用本地前缀注册：

- 解析顺序：**精确名 → 最长前缀 → 默认渲染器**；
- 精确名永远胜出；
- 空前缀永不匹配；
- 扩展描述符只按精确名匹配。

### 8.6.2 注册项包含什么

| 字段 | 作用 |
| --- | --- |
| `renderer` | 卡片正文（`ToolRenderProps`） |
| `summary` | 折叠头部的一行摘要，纯函数，经 `t` 翻译 |
| `chrome` | `"card"` 或 `"bare"` |
| `prominence` | `"routine"` 或 `"primary"` |
| `defaultExpanded` | 主卡片完成后是否展开 |
| `interactive` | 请求宿主填充 `ToolRenderProps.interaction` |

`"bare"` chrome **无条件**隐含 `primary`：自持框架的渲染器无法活在折叠的步骤行里，
因此错误注册不能静默破坏折叠。

### 8.6.3 为什么工具名必须稳定

- 服务端能力与 UI 渲染器靠它对齐；
- 历史转录靠它继续渲染；
- 服务端禁用后 web 仍注册；
- 迁移期间可以“旧服务端 + 新 web”共存。

**改名一个工具 = 破坏以下所有东西**：历史卡、扩展注册、子会话允许列表、用户安装的同名包优先级。
如果必须改，先加新名，双注册一段时间，再删旧名。

### 8.6.4 渲染器仍然是 props 驱动

扩展渲染器：

- 不 import store / transport；
- 只从 `ToolRenderProps` 拿 `toolCallId` / `args` / `result` / `status` / `workspaceRoot` / `streaming`；
- 通过 `ChatActions` 回话（`respondToTool`、`openSubagentTranscript`、`markResultSeen` 等）；
- 文件预览通过 `ResourceScopeProvider` + `ResourcePreview`，而不是自己读文件。

---

## 8.7 随会话加载的包：各包到底提供什么

### 8.7.1 `pi-spec-graph`（`packages/spec-graph`）

- **七个工具**：`spec_grep`、`spec_get`、`spec_graph`、`spec_create`、`spec_update`、`spec_delete`、`spec_validate`；
- 一个 skill + 一条常开的 `before_agent_start` 规则；
- `core/` 不含 pi 依赖，宿主可值导入以提供只读规格查看器；
- 文件系统是真相；索引是派生的、内存的、只读的、按 `(mtimeMs, size)` 重新校验；
- 工具**从不编辑正文**；`spec_update` 只改 frontmatter，保留字段顺序/注释。

对本书主题相关的部分：

| 事实 | 意义 |
| --- | --- |
| 规则在 `systemPromptOptions.sections` 里就地修改 | 只补一块可 diff 的 section，不强制整段提示 |
| 返回 `systemPrompt` 的第三方扩展会丢弃所有结构化 section | 这是 pi 文档化的语义，接受它 |
| 每个工具带 `description` + `promptSnippet` | 约束进工具，不进 prompt 体积 |
| `core` 是唯一供宿主消费的子路径 | 服务端不 import 扩展的 pi 运行时 |

### 8.7.2 `pi-todos`（`packages/pi-todos`）

- 六个 `todo_*` 工具 + 一个 skill + 一条简短规则；
- `core/` 的 `TodoStore` 读-改-写 `.thinkrail/context/todos/<sessionId>.json`；
- **分组 = 任务**；分组状态是派生的，从不存储；
- 恰好一个 `in_progress`，设置新的会自动降级旧的；
- `todo_write` 是**保留身份的核对**，不是替换；松散条目通道不可被它触碰；
- 磁盘文件 `version: 6`。

它是宿主 `todos` 模块（第 6 章）的引擎：宿主只通过 `pi-todos/core` 读计划，
通过工具名触达代理写入。

### 8.7.3 `pi-thinkrail-workflow`

- 只贡献两样：一个 `before_agent_start` 路由规则 + 一组 workflow skills；
- 没有自定义工具；只有 skill 内容；
- 与 `pi-spec-graph` 相同：就地修改 `systemPromptOptions.sections`，不返回整段提示；
- 已路由的工作恢复其活动工作流；
- 不假设 ThinkRail 专有工具：`spec_*` 与 `ask_user_question` 都只按名字触达；
- 不发布（工作区内部），只由 ThinkRail 暂存。

### 8.7.4 `pi-subagents` + `pi-delegation`

| 包 | 角色 |
| --- | --- |
| `pi-delegation` | 纯 pi 的委托核心：`createChild`、运行循环、用量聚合、血缘存储、运行 registry |
| `pi-subagents` | 委托核心的第一个消费者：`Agent` / `get_subagent_result`、定义发现、spawn 映射、完成投递 |

`pi-subagents` 的关键纪律：

- **零私有子会话装配代码**：一切经 `SessionOptions`；
- 定义优先级：`builtins → personal → project`（项目定义只在信任时加载）；
- 内置项是 TS 常量（`scout` / `planner` / `worker` / `reviewer`），不是 `.md`；
- 只有 `worker` 能改文件，且它的提示禁止历史/索引 git 操作；
- 后台结果**绝不随父轮次中止信号**；
- 结果在父级下一步**恰好一次**到达，经推送或拉取先到者；
- `delivery` 标记：`wake` / `run` / `appended`。

### 8.7.5 `pi-background-commands`

- `background_command` 四动作：`start` / `list` / `output` / `stop`；
- 命令是会话绑定的；id 不是 PID、路径或授权令牌；
- 状态：`running` / `stopping` / `completed` / `error` / `stopped`；
- 一个会话最多 8 个活动命令，保留最新 20 条终态；
- 输出只保留最新 2000 行或 50 KiB；
- 完成投递与 subagents 同一套规则；
- **不含 PTY、终端标签、stdin、重启/重跑、守护进程发现**。

### 8.7.6 `pi-dag`

- 持久 DAG 服务 + pi 扩展，建立在 `pi-delegation` 之上；
- 独立于 `pi-subagents` 和 ThinkRail；
- 拥有图调度、持久证据、门禁、命令与恢复；
- 不捆绑进 ThinkRail，也不暴露 UI 或 wire。

### 8.7.7 这些包如何被加载

| 加载方式 | 用于 |
| --- | --- |
| `additionalExtensionPaths` | dev 下的 `pi-spec-graph` / `pi-thinkrail-workflow` / `pi-todos` |
| `additionalSkillPaths` | 同一批包的 `skills/` |
| `registerBundledRuntime({ factories, skillsDir })` | 编译二进制/桌面打包 |
| 注册表扩展的 `skillPackages` | 扩展拥有的技能，按所有者暂存 |

**为什么有两种加载方式**：打包后的二进制没有 `node_modules`，不能按路径加载；
静态 import 的注册表扩展可以进入服务端图，而 `pi-spec-graph` 等仍按路径加载（尚未包装为扩展）。

---

## 8.8 新增一个能力：决策树与改动点

### 8.8.1 五个问题

1. **这个能力在原生 pi 中有意义吗？**
   有 → 先做 `pi-extensions/<name>` 可移植包。
2. **它需要 ThinkRail 专属的宿主服务或 UI 吗？**
   需要 → 做 `thinkrail-extensions/<name>`：`./server` 组合工厂，`./web` 注册渲染器。
3. **它需要新的线上方法/通道吗？**
   需要 → 改 `packages/contracts`，提升协议版本，在 `host/handlers.ts` 注册，在 `apps/web/transport` 路由。
4. **它只是展示层变化吗？**
   可能只需要 `registerToolRenderer`，不需要新协议。
5. **它需要宿主拥有的按会话状态吗？**
   需要 → 通过 `sessionExtensions(session)` 的 `ExtensionSessionServices`；
   不要在扩展里创建第二份状态所有者。

### 8.8.2 改动点对照表

| 你要加的东西 | 应该改哪里 |
| --- | --- |
| 一个新的可移植工具 | `pi-extensions/<name>/index.ts` + `pi` 清单 + 用例表 |
| 新的 ThinkRail 组合 | `thinkrail-extensions/<name>/server/index.ts` + `serverExtensions` |
| 新的工具卡 | `thinkrail-extensions/<name>/web/index.ts` + `webExtensions` |
| 新的交互式工具 | server `interactiveTools` + web `interactive: true` + 邮箱接入 |
| 新的技能包 | 描述符 `skillPackages` + `buildSupport` 暂存 |
| 新的宿主方法 | `contracts` + 协议版本 + `host/handlers.ts` + web 路由 |
| 新的 SDK 依赖 | 先迁移到 `extension-api` / `resource-renderers` / `ui`，再让扩展 import |
| 新的开关粒度 | 需要设计变更，不是加一个描述符字段 |

### 8.8.3 新增扩展的最小检查单

- [ ] `SPEC.md` 先写：职责、两个入口点、允许依赖、禁止依赖、必须做对
- [ ] `./server` 默认导出 `defineServerExtension`，无根 barrel
- [ ] `./web` 默认导出 `defineWebExtension`，`toolRenderers` 以工具名为键
- [ ] `web/**` 与 `server/**` 不互相 import
- [ ] `./web` 不 import pi 值；只 import 类型
- [ ] 服务端注册表加条目（含 `specifier` 字符串）
- [ ] Web 注册表加条目
- [ ] 交互式工具在两侧都声明
- [ ] 技能包在 `skillPackages` 声明，并验证暂存路由
- [ ] 工具名与可移植包一致
- [ ] 关闭扩展后：工具被扣留、技能被排除、历史仍渲染
- [ ] 跑 `check:boundaries`、`check:seams`、`lint`、`typecheck`
- [ ] 可移植包另跑 `check:pi-packages`

### 8.8.4 为什么扩展不能“注册一个面板”或“注册一个设置页”

当前描述符只有 `name/title/description/extensions/sessionExtensions/interactiveTools/childExtensions/skillPackages`。
面板、设置页、传输线方法都是**显式延后项**，不是缺口：

- 面板贡献需要布局无关的注册机制；
- 设置页需要 schema 与存储归属；
- 传输线方法需要协议版本与客户端能力门禁。

在出现真实消费方之前，不要为假设需求加字段——那会让所有扩展被迫跟随。

---

## 8.9 必须做对：反模式表

| 反模式 | 为什么错 | 正确做法 |
| --- | --- | --- |
| 扩展 `web` import `server` | 把 pi 运行时拖进浏览器 | 两个入口点，绝不互 import |
| 扩展 `web` 值导入 pi 包 | 浏览器包出现 `node:fs` / provider SDK | 只 import `./core` 类型 |
| `packages/server` 直接 import 扩展的 pi 包 | 宿主与扩展拥有物耦合，隔离布局下解析失败 | 只 import `@thinkrail/ext-*/server` |
| 用函数推断 `skillPackages` | 构建步骤没有模块身份 | 显式包说明符 |
| 用 `pi install <path>` 做一致性测试 | 它只记录路径，不安装任何东西 | 打包 tarball + npm 解析 |
| 把 runner 留在仓库里跑 | Node 解析的是仓库里的 pi，不是 tarball | 复制到夹具中运行 |
| 认为禁用会立即卸载钩子 | pi 无法卸载，每次禁用都重载会击穿缓存 | 立即扣留工具；必要时才重载 |
| 禁用后删除历史卡 | 转录仍引用该工具 | web 注册忽略开关 |
| 省掉 `specifier` | 技能解析拿不到入口文件 | 注册表条目同时存描述符与说明符 |
| 让用户同名包接管 `ask_user_question` | 宿主 UI/重启修复按名字绑定 | `preferRegistryTools` |
| 把交互式工具做成模态框 | 会话可能不存在，焦点会被拽走 | chat 内联提示；登录是例外 |
| 用真实结果详情伪造 pending ack | `isToolResponsePending` 的约定被破坏 | `{ kind: "ack" }` 只用于待处理 |
| 重启修复把交互式调用标成错误 | 问题永久消失 | 写 pending ack |
| 扩展自己读文件/存评论 | 宿主才是文件与评审的权威 | `ResourcePreview` + provider |
| 注册了工具却不给 TUI 渲染 | 原生 pi 下退化成不可读文本 | `renderCall` / `renderResult` |
| 假设扩展能在运行时热替换 | 没有运行时加载器 | 静态注册表 + 开关 |
| 加第二个 web 注册表 | 两份真相会漂移 | 一个 `webExtensions` |
| 在扩展里创建按会话状态 | 生命周期归宿主 | `sessionExtensions(session)` 显式接缝 |

---

## 8.10 追踪练习

### 练习 1：一个扩展同时有 `extensions` 和两个 `sessionExtensions` 工厂，展平后叫什么？

<details><summary>参考答案</summary>

`factoriesOf` 把 `extension.extensions` 与 `sessionExtensions(session)` 拼接。
当总工厂数 > 1 时，命名为 `name#1`、`name#2`、`name#3`；只有一个时保留裸名 `name`。
`registryExtensionName` 能匹配 `name#n`，并按名字回查，因此不依赖工厂数量。

见 [inline.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/inline.ts#L14-L43)。
</details>

### 练习 2：为什么服务端注册表必须保存 `specifier` 字符串？

<details><summary>参考答案</summary>

描述符里装的是函数；函数不产生模块身份。
技能解析需要 `createRequire(import.meta.url).resolve(specifier)` 找到扩展入口，
再从扩展入口 `createRequire(extensionEntry)` 解析每个 `skillPackages` 的 `package.json`。
没有 `specifier` 就无法按所有者解析技能根。

见 [skills.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/skills.ts#L10-L26)。
</details>

### 练习 3：关闭 `web-access` 后，哪些东西立即消失，哪些要等重载？

<details><summary>参考答案</summary>

立即：

- 活跃会话里属于该扩展的工具从活动集扣留；
- 新父会话/子会话不再组合它的工厂；
- 新加载的技能输入排除它的技能根。

等重载：

- 已加载扩展的钩子继续运行（pi 无法卸载）；
- 如果会话已经加载了它的 `librarian` skill，则触发一次资源重载以从系统 prompt 中移除。

见 [agentSessionManager.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/agentSessionManager.ts#L748-L782) 与
[agent/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/SPEC.zh.md) 的“注册表扩展”段。
</details>

### 练习 4：宿主重启时，一个悬空的 `ask_user_question` 会变成什么？

<details><summary>参考答案</summary>

`repairDanglingToolCalls` 判断：

- 调用可执行且工具在 `interactiveTools` 里 → 写入 pending ack 文本，
  `details: { kind: "ack" }`，`isError: false`；
- 调用可执行但非交互 → 通用 “Operation aborted (the host restarted…)” 错误；
- 调用不可执行（截断/中止）→ “arguments may be truncated” 错误。

之后用户的回答通过 `tool-response` 自定义消息送达，并用 `describeResponse` 渲染给模型。

见 [sessionRepair.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/sessionRepair.ts#L17-L48)。
</details>

### 练习 5：用户在卡片上提交回答后，什么情况下这次回答算“成功”？

<details><summary>参考答案</summary>

`waiters.respond` 先把阶段设为 `response-accepted-uncommitted` 并解析工具等待；
只有工具结果持久化时 `persistTurn` 用 `isDeepStrictEqual(result.details, acceptedResponse)` 校验通过，
才 `resolvePersisted`。不相等或 `isError: true` 会 `rejectPersisted`。
因此“已接受但未持久化”不会被误判为成功。

见 [toolResponses.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/agent/toolResponses.ts#L270-L312)。
</details>

### 练习 6：为什么 `preferRegistryTools` 存在？

<details><summary>参考答案</summary>

因为用户可能自己安装了同名包（例如 `pi-web-access`，或第三方 `ask_user_question`）。
宿主的 UI、生命周期与重启修复都按工具名绑定；如果用户的副本抢先注册，宿主的卡片会与一个
不受宿主控制的工具对接。`preferRegistryTools` 把注册表扩展移到同名工具的非注册表扩展之前。
关闭扩展后，用户的副本自然接管。

见 [inline.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/extensions/inline.ts#L66-L87) 与
[web-access/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/thinkrail-extensions/web-access/SPEC.zh.md)。
</details>

### 练习 7：为什么 `visualize` 的服务端要注入严格 mermaid 校验器，而可移植包还保留自己的探针？

<details><summary>参考答案</summary>

可移植包不能依赖浏览器/Node 的 `mermaid`（体积大且需要 DOM），因此它用 `lovely-mermaid` 做尽力而为的探针。
ThinkRail 的 web 渲染器用固定版本的 `mermaid`，所以 server 半边注入与该渲染器完全一致的严格解析器，
保证“工具接受 = 渲染器能解析”。渲染器仍保留防御性回退：保证的是语法解析，不是 SVG/布局成功。

见 [visualize/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/thinkrail-extensions/visualize/SPEC.zh.md)。
</details>

### 练习 8：一个扩展想贡献一个新的设置页，应该加什么字段？

<details><summary>参考答案</summary>

**当前不应该加。** 面板、设置页、传输线方法是显式的延后项：
它们需要布局无关的注册、schema 与存储归属、协议版本与能力门禁。
在出现真实消费方之前，描述符不增加字段，避免所有扩展被迫跟随。

见 [thinkrail-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/thinkrail-extensions/SPEC.zh.md) 的“形态”段。
</details>

### 练习 9：为什么 `pi-extensions` 的包可以直接交付 `.ts`，而 `thinkrail-extensions` 不行？

<details><summary>参考答案</summary>

pi 通过 jiti 加载入口点，因此可移植包可以直接交付原始 TypeScript（无构建步骤）。
`thinkrail-extensions` 是工作区包，由宿主/Bun 打包器消费；它们有自己的 `tsconfig.json`，
走仓库正常的 TypeScript 构建/类型检查。两条路径不同，不能混用。

见 [pi-extensions/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-extensions/SPEC.zh.md) 的“身份与分发”段。
</details>

### 练习 10：一个工具改了名字，需要同时改哪些地方？

<details><summary>参考答案</summary>

至少：

- pi 包里的工具注册名；
- `thinkrail-extensions/<name>/web` 的 `toolRenderers` 键；
- 任何 `interactiveTools` 声明与 `describeResponse`；
- 子会话允许列表（如果有）；
- 用户的同名包优先级假设（`preferRegistryTools` 按名字）；
- 历史转录的可读性（旧名不再有渲染器）。

因此正确做法是**先加新名，双注册一段时间，再删旧名**，而不是直接重命名。

见 [chat/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/apps/web/src/chat/SPEC.zh.md) 的工具注册表段。
</details>

---

## 8.11 一页速查

```
四种形态
  pi 包（可移植，@thinkrail/pi-*）        → 原生 pi 也能跑
  ThinkRail 扩展（@thinkrail/ext-*）      → ./server + ./web，宿主专属
  应用本地渲染器（apps/web/chat/tools）    → 只呈现
  MCP server                              → pi 的 MCP 引擎

两个注册表
  packages/server/src/extensions/registry.ts   { specifier, extension }
  apps/web/src/extensions/registry.ts          webExtensions → registerToolRenderer

展平
  name（单工厂）/ name#n（多工厂）
  isDisabled(name) 在 pi 加载工厂时询问
  extensions = extensions + sessionExtensions(session)
  childExtensions 只取 childExtensions

override
  omitDisabledRegistryExtensions  丢弃 <inline:...>
  preferRegistryTools            注册表扩展优先于同名用户包

开关
  disabledExtensions（宿主设置，按 name）
  → refreshExtensionTools
  → 活跃工具扣留（流式推迟到 agent_settled）
  → 需要时 requestSessionReload（缺扩展 / 已加载其 skills）
  钩子留到重载；web 注册忽略开关

交互式工具
  server interactiveTools + web interactive: true
  message_end → announce → expect → await
  respond → accepted-uncommitted → persistTurn 深比较 → persisted
  重启：可执行+交互 → pending ack；可执行非交互 → aborted；不可执行 → truncated
  迟到：describeResponse(args, response)
  pending 标记：details { kind: "ack" } + isToolResponsePending

技能
  skillPackages 说明符 → createRequire(extensionEntry) 解析
  dev：源根；bundled：skills/extensions/<name>
  关闭：从 pi 技能输入排除

新增能力
  原生 pi 有意义？ → pi-extensions 包
  需要宿主服务/UI？ → thinkrail-extensions
  需要新 wire？ → contracts + 版本 + handler + transport
  只是展示？ → registerToolRenderer
  需要按会话状态？ → sessionExtensions(session)
```

---

### 与主指南的对应

| 主指南小节 | 本专题的展开 |
| --- | --- |
| 8.1 两种扩展 | 8.0、8.1、8.2 |
| 8.2 注册表组装 | 8.3 |
| 8.3 工具名是连接键 | 8.6 |
| 8.4 随会话加载的包一览 | 8.7 |
| 8.5 想新增一个能力 | 8.8 |

> 建议的下一步：拿 `ask-user-question` 做一次完整追踪——从 `./server` 的
> `sessionExtensions` → `expectToolResponse` → 卡片 `respond` → `persistTurn`，
> 再故意关闭它，观察哪些东西立即消失、哪些留到重载。这条路径把本章的四个核心机制
> （组合、邮箱、开关、渲染）一次串完。
