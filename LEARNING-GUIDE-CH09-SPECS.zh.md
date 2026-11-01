# 第 9 章深度专题：规格驱动开发——从 frontmatter 到可执行门禁

> 本专题是 [LEARNING-GUIDE.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/LEARNING-GUIDE.zh.md) 第 9 章的展开版。
> 主指南讲的是“为什么先改 spec”；这里讲的是**spec 本身的数据模型、它如何被索引、哪些声明会被机器校验、
> 以及一次变更如何从 owning spec 走到可合并的 PR**。
>
> 阅读前提：第 3 章（三环）、第 6 章（服务端模块）、第 8 章（扩展边界）。
> 本章把“规格”当作一种**有 schema、有索引、有门禁**的工程产物来拆解。

---

## 9.0 先建立“规格”的坐标系

### 9.0.1 规格是什么、不是什么

| 是 | 不是 |
| --- | --- |
| **与代码同仓、可评审的决策记录** | 聊天记录、个人记忆、口头约定 |
| 模块的**边界、公开面、允许/禁止依赖** | API 文档的替代品（代码才是实现） |
| **持久意图**：为什么这样做、放弃了什么、代价是什么 | 逐行解释代码的注释 |
| 可被 `pi-spec-graph` 索引的**派生图谱节点** | 第二套运行时状态 |
| 变更流程的**第一站**（先改 spec，再改代码） | 事后补写的 README |

一句话：**spec 记录“边界与理由”，代码记录“实现”，测试记录“证据”。**

### 9.0.2 四种文档层级

| 层级 | 例子 | 回答的问题 |
| --- | --- | --- |
| 仓库级 | `goal-and-requirements.md`、`architecture.md`、`AGENTS.md`、`pi-compatibility.md` | 产品目标、跨模块决策、工作规则、pi 适配清单 |
| 包级 | `packages/*/SPEC.md`、`apps/*/SPEC.md` | 这个包拥有什么、公开面是什么、依赖谁 |
| 子模块级 | `packages/server/src/<sub>/SPEC.md`、`apps/web/src/<sub>/SPEC.md` | 这个目录的职责、边界、叶子 |
| 任务级 | `*.task-spec` / 临时设计 | 一次具体工作的范围（**不进入持久图谱**） |

### 9.0.3 一个 spec 的五个身份

| 身份 | 字段 | 作用 |
| --- | --- | --- |
| 节点 id | `id` | 图谱主键；跨文件引用 |
| 类型 | `type` | 层级/角色；`spec_create` 的撰写词表 |
| 状态 | `status` | 生命周期：draft/active/stale/done/deprecated |
| 父节点 | `parent` | 父树；模块归属 |
| 链接 | `depends-on` / `references` / `implements` | 跨模块边 |

还有两个列表字段：

- `covers`：这份 spec 覆盖哪些范围/条目；
- `tags`：自由标签；`public-surface-checked` 是**门禁纳入标记**。

### 9.0.4 术语表

| 术语 | 含义 | 易错点 |
| --- | --- | --- |
| owning spec | 拥有某个行为的模块规格 | 不是“最相关的那份” |
| 父树 / DAG | `parent` 构成树；`depends-on` 等构成有向无环图 | 两者不是同一张图 |
| 派生索引 | `SpecIndex` 从磁盘构建的只读缓存 | 不是权威状态 |
| 纳入（enrollment） | 用 tag 把一份 spec 交给 `check:spec-surface` | 不是自动全量 |
| 裸标识符列表 | 公开面声明的机器可读形式 | 散文接口不算纳入 |
| 移除门禁（removal gate） | pi 升级后可以删除某适配的条件 | 不是“等上游修好”的口号 |
| 任务规格 | `type: task-spec` | 不算持久规格 |

---

## 9.1 frontmatter：spec 的数据模型

### 9.1.1 字段注册表

[parse.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/spec-graph/core/parse.ts#L7-L18)：

```ts
export const FIELDS = {
	id: "id",
	type: "type",
	status: "status",
	title: "title",
	parent: "parent",
	dependsOn: "depends-on",
	references: "references",
	implements: "implements",
	covers: "covers",
	tags: "tags",
} as const;

export const REQUIRED_FIELDS = [FIELDS.id, FIELDS.type, FIELDS.title] as const;
export const IDENTITY_FIELDS = [FIELDS.id, FIELDS.type] as const;
```

要点：

- 字段名是**单一来源**的 `as const`，没有第二份字面量列表；
- `dependsOn` 的 YAML 键是 `depends-on`（kebab-case）；
- `id` + `type` 是身份，`title` 也必填；
- `status` 不是身份，但参与生命周期。

### 9.1.2 类型与状态词表

```ts
export const SPEC_TYPES = [
	"goal-and-requirements",
	"architecture-design",
	"module-design",
	"submodule-design",
	"task-spec",
] as const;

export const SPEC_STATUSES = ["draft", "active", "stale", "done", "deprecated"] as const;
```

**词表只约束撰写接口**（`spec_create`），不约束图谱：

> `SpecNode.type` 保持 `string`：读取模型索引磁盘上的任何东西，因此它容忍任何 `type`。
> 词表只约束 `spec_create` 的撰写接口，永不约束图谱。

这条很重要：**读取宽容，写入严格。** 一个历史 spec 用了新类型，索引不会拒绝它；
但新写入必须落在已知词表内。

### 9.1.3 链接种类

```ts
export const SINGLE_LINK_FIELDS = [FIELDS.parent] as const;
export const LIST_LINK_FIELDS = [FIELDS.dependsOn, FIELDS.references, FIELDS.implements] as const;
export const LIST_FIELDS = [...LIST_LINK_FIELDS, FIELDS.covers, FIELDS.tags] as const;
```

| 字段 | 基数 | 语义 |
| --- | --- | --- |
| `parent` | 单值 | 父节点；构成父树 |
| `depends-on` | 列表 | 运行时/构建依赖；构成 DAG |
| `references` | 列表 | 解释性引用；不一定是依赖 |
| `implements` | 列表 | 实现某条上游决策/规格 |
| `covers` | 列表 | 覆盖范围，不是链接 |
| `tags` | 列表 | 自由标签/门禁标记 |

**`references` 与 `depends-on` 不是同义词**：前者是“读它有帮助”，后者是“没有它就不成立”。

### 9.1.4 is-a-spec 的唯一规则

```ts
export function isSpec(fm: Frontmatter | null): fm is Frontmatter {
	return fm !== null && IDENTITY_FIELDS.every((field) => scalar(fm, field) !== undefined);
}
```

只要 frontmatter 有 `id` 与 `type`，它就是一份规格。没有别的注册表、没有目录白名单。

### 9.1.5 读取有损，写入无损

| 路径 | 行为 |
| --- | --- |
| 读取 | 把 frontmatter 强制成标量/字符串数组方言；嵌套映射与注释被丢弃 |
| 写入 `updateFrontmatterText` | 就地变更活的 `yaml` Document：未触碰字段保持顺序，注释/嵌套值存活 |
| 正文 | **逐字节拼回**；前导 BOM 放回；换行符读自 frontmatter 自己的第一个换行 |
| 字段顺序 | 保留，永不重排；`FIELD_ORDER` 只用于 `spec_create` 构建**新** frontmatter |

这保证 `spec_update` **永不编辑正文**——在字节层面成立，而不只是字段层面。

### 9.1.6 CRLF / BOM 的两个方向

- 写：施加到重写块的换行符读自 frontmatter 的第一个换行（LF 或 CRLF）；
- 读：`grepSpecs` 按 `\n` 切分，匹配前丢掉行尾 `\r` 与前导 BOM，
  因此 `^` 模式在 LF、CRLF、BOM 前缀的规格上表现一致。

这是“同一个语义在多种文件编码下一致”的典型处理：不靠规范化整个文件，只在读取边界吸收差异。

---

## 9.2 图谱：父树 + DAG + 反向边

### 9.2.1 两层关系

[core/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/spec-graph/core/SPEC.zh.md)：

- `parent` 构成**父树**；
- `depends-on` / `references` / `implements` 构成 **DAG + 反向边**；
- `graph.ts` 还跟踪重复 `id`；
- `validate.ts` 检查悬空链接、重复 id、父环。

### 9.2.2 派生索引：文件系统是真相

`SpecIndex`：

- 按需 glob；
- 逐文件解析缓存；
- 记忆化图谱；
- 按 `(mtimeMs, size)` 重新校验；
- 只有规格集真的变化才重建派生图谱。

**索引不持有权威状态**：它绝不提供过期图谱；文件系统变了，索引就重建。

### 9.2.3 全序：为什么排序不是小事

glob 先把每个目录过滤到遍历候选（未忽略目录 + `.md` 文件），再规范化并排序。
顺序是**全序**：按 NFC 规范化名称比较，以原名码元决胜。

原因：**重复 `id` 的胜者必须跨文件系统稳定**。

- 不能是 `readdir` 碰巧返回的顺序；
- 不能在 NFD 文件系统上得到另一个答案；
- 不能让预组合 `é` 与 `e\u0301` 再次采用 `readdir` 的顺序——它们共享同一个 NFC 键。

目录与 `.md` 文件留在**同一个**候选列表里，因此子目录在同级文件之间的位置被下钻，
而不是排在它们全部之前或之后。

### 9.2.4 `resolveSpecPath`：索引是否可能看到这条路径

它是这个问题的**唯一答案**，并用**规范相对路径**作答。要求：

- 根相对、在根之内；
- `.md`；
- 在被忽略目录之外（写路径按折叠后的形式匹配 `IGNORED_DIRS`）；
- 根存在；
- 根之下任何组件处都没有符号链接（逐组件 `lstat`，悬空链接也算）。

Windows 额外：

- 含冒号的路径在规范化前就拒绝（关掉驱动器相对路径与 NTFS 备用数据流）；
- 规范化后仍存活的 `..` 段拒绝；
- 绝对结果通过最后一次 `relative(root, target)` 包含性检查。

**写路径过度拒绝，读路径保持精确**：

- 解析器把 `NODE_MODULES/SPEC.md` 的任意拼写都拒绝；
- glob 仍按逐字节精确的 `readdir` 名称匹配。

因为“拒绝过多”给调用方一个可行动的原因；“跳过过多”会把用户真的命名为 `Build/` 的目录从索引里丢掉。

### 9.2.5 目录顺序与重复 id

出现重复 `id` 时，该全序中的**第一个文件**赢得节点槽位；重复集合记录给 `validate`。
这正是“排序必须稳定”的产品后果。

---

## 9.3 仓库的文档层级与权威

### 9.3.1 根级文档分工

| 文档 | 拥有 | 不该放什么 |
| --- | --- | --- |
| [goal-and-requirements.md](file:///Users/luzhengli/Documents/github/thinkrail/goal-and-requirements.md) | 产品目标、原则、能力、非目标 | 实现细节 |
| [architecture.md](file:///Users/luzhengli/Documents/github/thinkrail/architecture.md) | 拓扑、跨模块决策、全仓不变量 | 某个模块的局部规则 |
| [AGENTS.md](file:///Users/luzhengli/Documents/github/thinkrail/AGENTS.md) | 工作规则、验证门禁、交接卫生 | 产品行为 |
| [pi-compatibility.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-compatibility.md) | pi 升级检查清单与移除门禁 | 每个模块的完整理由 |

`architecture.md` 当前有 **23 条决策**，每条都回答“为什么这样划分、代价是什么”。
例如：

- 决策 1：客户端/宿主分离，`apps/web` 绝不依赖 `server`/`shared`；
- 决策 3：wire 有版本，`server.welcome` 携带协议版本；
- 决策 8：hydrate-then-stream，每个客户端都能从宿主重建领域状态；
- 决策 20：可移植 pi 包发布、scoped、受 vanilla-parity 约束；
- 决策 23：UI 字符串是键控目录，本地语言文件是运行时数据。

**决策与 spec 的分工**：决策是跨模块的“宪法”；spec 是某个模块如何遵守它。

### 9.3.2 中文译本的权威规则

每个 `.zh.md` 都在开头声明：

> 中文译文，原文见 `SPEC.md`。英文原文是唯一权威来源，二者不一致时以原文为准。

这不是客套：译文可以滞后、可以有措辞差异，但**门禁与代码以英文原文为准**。
因此改行为时至少要改英文 `SPEC.md`；中文译本可随后同步。

### 9.3.3 任务规格不进入持久图谱

`projectHasSpecs(root)` 只认持久规格；`task-spec` 不算。

原因：任务规格描述一次工作，不是模块的长期边界。把它算进“这个项目有没有规格”，
会让一个临时任务把项目误判为“规格化项目”。

---

## 9.4 边界如何在 spec 里表达

### 9.4.1 每份子模块 spec 的固定骨架

一个合格的子模块 `SPEC.md` 至少有：

| 段 | 必须回答 |
| --- | --- |
| Responsibility | 这个模块拥有什么（一句话） |
| Boundary / Owns | 具体职责清单 |
| Public surface | 公开面（barrel 导出的名字） |
| Allowed dependencies | 允许 import 谁 |
| Forbidden | 绝不能 import 谁/做什么 |
| Get right / Invariants | 必须做对的规则与代价 |
| Verification（可选） | 哪些测试钉住它 |

### 9.4.2 依赖边写在父模块，不写在叶子上

规则：**兄弟模块之间的依赖边写在父模块的 `SPEC.md`，不写在每个叶子上。**

原因：

- 叶子的职责是“我做什么”，不是“整个图长什么样”；
- 一张图只有一个所有者，避免 N 份局部图互相漂移；
- `check:boundaries` 读的是代码，父 spec 读的是意图；两者要能对上。

例：`packages/server/SPEC.zh.md` 拥有 35 个模块的依赖图；
`packages/server/src/git/SPEC.zh.md` 只写自己 → `subprocess`/`fs`，不写“谁依赖我”。

### 9.4.3 barrel 规则与例外

一般规则：子模块只经 `index.ts` barrel 暴露公开面；兄弟从 barrel import。

例外必须**被命名**：

| 例外 | 理由 |
| --- | --- |
| `apps/web/src/panels` | 加 barrel 会把懒加载的 Monaco/shiki/xterm chunk 拉进急切 bundle |
| `packages/ui` | shadcn 的按原语 import 惯例 |
| `apps/web/src/chat` | 同样的代码分割理由（工具/渲染器按文件） |

**例外不是漏洞**：它们由父 spec 与 `check:boundaries` 共同维持。

### 9.4.4 公开面声明的两种写法

| 写法 | 是否可纳入 `check:spec-surface` |
| --- | --- |
| `**Public surface (barrel):** \`a\`, \`b\`, \`c\`` | 是（裸标识符列表） |
| `公开接口：一个描述性段落` | 否（只报告为 skipped） |

机器只认**裸反引号标识符列表**：去掉反引号后只剩标识符与分隔符。
`type Foo` 会取 `Foo`；散文、括号解释、路径都不行。

### 9.4.5 边界变化的正确顺序

```
改 owning spec（说明新边界/新公开面/新依赖边）
→ 改父 spec 的依赖图（如果边变了）
→ 改代码（barrel 与 spec 一致）
→ 跑 check:boundaries + check:spec-surface
```

如果反过来，代码会先形成一个 spec 说不清的形状，review 只能靠猜。

---

## 9.5 `check:spec-surface`：把公开面变成可执行契约

### 9.5.1 纳入是显式的

[specSurface.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/specSurface.ts#L556-L591)：

```ts
const enrolled = list(entry.frontmatter, FIELDS.tags).includes(PUBLIC_SURFACE_TAG);
if (!enrolled) {
	report.skipped.push({ path: entry.path, reason: skippedReason(block, barrel) });
	continue;
}
```

`PUBLIC_SURFACE_TAG = "public-surface-checked"`。

**为什么纳入而不是全量**：

- 全量会迫使每个描述性接口都改成机器格式，制造噪音；
- 纳入是显式的，新增文字不能静默关闭强制；
- 未纳入的 spec 仍会被 `--list-skipped` 报告，不会被遗忘。

当前仓库有 **24 份** spec 带这个 tag（以扫描时为准）。

### 9.5.2 纳入的四个条件

被纳入的 spec 必须同时满足：

1. 声明了一个公开面块；
2. 该块是**裸标识符列表**；
3. 能找到 TypeScript barrel（同目录 `index.ts` 或 `src/index.ts`）；
4. barrel 能被 TypeScript 解析。

否则直接 violation，而不是“跳过”。

### 9.5.3 公开面块的识别

`readSurfaceBlock` 支持：

- 段落/列表项的标签以 `Public surface` 或 `owns / public surface` 开头（大小写不敏感）；
- 标题形式：`## Public surface`，取到下一个同级/任意标题之前；
- 非标题形式：取该段落或列表项的范围。

代码块与 HTML 行会被隐藏，避免把示例当成声明。

### 9.5.4 barrel 发现

```ts
for (const candidate of [
	join(dirname(specFile), "index.ts"),
	join(dirname(specFile), "src", "index.ts"),
]) {
	if (fileExists(candidate)) return resolve(candidate);
}
```

两种布局都支持：

- 模块目录直接 `index.ts`（`packages/server/src/git/index.ts`）；
- 包目录 `src/index.ts`（`packages/server/src/index.ts`）。

### 9.5.5 TypeScript 解析：两个方向都必须匹配

`checkCompilerGroup`：

1. 用 barrel 向上找 `tsconfig.json`；
2. 建 `ts.createProgram`（把 barrel 加入 rootNames）；
3. 用 checker 取模块符号；
4. 先查 `exportGraphIssues`（无效/歧义 re-export）；
5. 取有效导出名；
6. `diffSurface(declared, exported)`。

两个方向：

| 差异 | 报错 |
| --- | --- |
| `promised`（声明了但 barrel 不再导出） | “names its public surface promises but the barrel no longer exports” |
| `undeclared`（barrel 导出了但声明没列） | “the barrel exports names its public surface does not list” |

**只查一个方向不够**：少导出是破坏契约；多导出是悄悄扩大公开面。

### 9.5.6 有效导出名的细节

- 包含 type-only 与默认导出；
- 包含传递 re-export；
- CommonJS `export =` 是模块唯一的 `default` 接口，而不是被赋值值的合成成员；
- 无法解析的 re-export 是违规，而不是“接口不完整但通过”；
- 歧义 star export 也是违规。

### 9.5.7 为什么它比“人工 review”可靠

- 人可以漏掉一个 `export *` 新增的名字；
- 编译器不会；
- spec 的声明与 barrel 的导出是两个独立来源，交叉校验才叫契约。

---

## 9.6 其他门禁：规格之外的可执行契约

规格本身是文字，但仓库用一组脚本把假设变成失败。根 `package.json` 的映射：

| 命令 | 实现 | 校验什么 |
| --- | --- | --- |
| `check:deps` | `scripts/check-catalog.ts` | catalog 精确 pin、`catalog:` 引用、`react`/`react-dom` 单版本 |
| `check:boundaries` | 检查器测试 + `check-module-boundaries.ts` | 清单边 + 静态源码 import（含 type-only/dynamic/re-export/CJS/子路径） |
| `check:seams` | `check-binary-seams.ts` | pi dist 中打包器不透明 `import()` 是否被接缝处理 |
| `check:spec-surface` | `check-spec-surface.ts` + `specSurface.ts` | 纳入 spec 的公开面与 barrel 双向一致 |
| `check:i18n` | 检查器测试 + `check-i18n.ts` | 硬编码 UI 文本、翻译键、渲染期全局翻译器 |
| `check:compiler` | `@thinkrail/web compiler:census:check` | React Compiler bailout 是否偏离基线 |
| `check:pi-packages` | `check-pi-packages.ts` | 可移植 pi 包的 vanilla parity |

### 9.6.1 `check:deps`：一条依赖只有一种 pin

[check-catalog.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/check-catalog.ts)：

- catalog 的每个条目必须是精确版本（无范围）；
- 根 `overrides` 的 `react`/`react-dom` 必须是 `"catalog:"`；
- 工作区清单里，catalog 管理的包必须写 `"catalog:"`，不能再 pin；
- 未在 catalog 的普通依赖必须精确 pin（URL/git 等含 `:` 的除外）；
- `bun.lock` 中 `react`/`react-dom` 只能有一个解析版本，且等于 catalog pin。

**为什么这么严**：两个 React 版本会让 hooks/compiler 行为分叉；
范围版本会让 CI 与本地解析出不同树。

### 9.6.2 `check:boundaries`：清单边与源码边都查

[check-module-boundaries.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/check-module-boundaries.ts#L20-L61) 的 `MODULE_RULES` 是穷举的：

```
packages/artifact-tests → apps/cli, packages/server, packages/shared
packages/contracts      → 无
packages/shared         → contracts
pi-delegation           → 无
pi-background-commands  → 无
pi-subagents            → pi-delegation
pi-dag                  → pi-delegation
packages/server         → contracts, locales, shared, spec-graph,
                          pi-delegation, pi-background-commands, pi-subagents,
                          pi-thinkrail-workflow, pi-todos, extension-api
apps/web                → contracts, locales, ui, extension-api, resource-renderers
apps/cli                → server, shared
apps/desktop            → server, shared, contracts, locales
```

然后动态补充：

- `packages/ui` / `packages/extension-api` → contracts、locales；
- `packages/locales` → 无；
- `packages/resource-renderers` → contracts、locales、ui；
- 每个 `pi-extensions/*` → 无；
- 每个 `thinkrail-extensions/*` → contracts、locales、ui、extension-api、resource-renderers、
  对应的 `pi-extensions/<name>`。

源码扫描：

- type-only、动态 import、re-export、CommonJS、`import type`、包子路径、相对跨模块都算；
- 排除 `.git`、`.hutch`、`.cottontail-tmp`、`.stage`、`artifacts`、`build`、`dist`、`node_modules`；
- 浏览器源码（web、UI、extension-api 的 web 半边、扩展 web 半边）**绝不可 import extension-api 的 server 半边**；
- 浏览器源码拒绝 pi 值 import，包括未注册为工作区根的外部 pi 包。

**“type-only 不能绕过”**：`import type` 仍受同样的模块边界约束。

### 9.6.3 `check:seams`：pi 升级后的打包接缝

[check-binary-seams.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/check-binary-seams.ts)：

- 从 `packages/server` 解析 pi 包根；
- 递归遍历 `@earendil-works/*` 依赖；
- 扫描 `dist` 下的 `.js`；
- 找 `import(<非字符串字面量>)`（打包器无法静态解析）；
- 与 allowlist 对照：
  - 新增的不透明 import → 失败，要求注册静态接缝或证明只接收 `node:` 内建；
  - allowlist 中已不存在的 import → 失败（stale），要求重新验证；
- `SKIPPED_DIST_DIRS` 只跳过明确不进入进程内库路径的目录。

它防的是：pi 升级后新增一个动态 `import()`，编译二进制在运行时解析不到，直到用户触发才炸。

### 9.6.4 `check:i18n`：四种字面量 + 渲染期读取

[check-i18n.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/check-i18n.ts)：

扫描根：`apps/web/src`、`packages/ui`、`packages/resource-renderers/src`、`thinkrail-extensions`；
跳过 `node_modules`、`generated`、`dist`、`server`、`scripts`。

拒绝四种字面量 UI 文本：

1. JSX 文本；
2. 承载文本的 JSX 属性（`aria-label`、`title`、`placeholder`、`alt`、`label`、`description`、`hint`、
   以及 `*Label`/`*Title`/`*Description` 等后缀），包括经条件表达式、`||`/`??`、`+`、模板片段抵达的字面量；
3. `{…}` 内的字面子节点；
4. 文本命名对象属性（`label`、`title`、`description`、`detail`、`hint`、`message`、`reason` 等）中的散文，
   以及字面量 `toast.*()` 消息或 `errorText()` 回退值。

“文本”定义：去掉符号后连续两个或更多字母；`⌘K`、`·`、计数通过；裸品牌 `ThinkRail` 允许。

还检查 `packages/locales/en.json`：

- 键已排序；
- 值非空；
- 每个键（`_one`/`_other` 用复数基础键）都被某个 UI/桌面/locales 源码以字符串字面量引用；
- 移除 UI 不会留下孤立译文。

最后构建 web 的 TypeScript program，对**渲染期读取全局翻译器**失败：

- `t`、`i18n`、`currentLocale`、`currentTranslator` 在组件/hook 函数体、类 `render()`、
  `useMemo`/`useState` 初始化器、`useAppStore` 选择器、`.map` 回调、JSX 函数子节点、`render*` 属性中求值；
- 同步抵达这些位置的调用（直接、命名空间 import、作为引用传递），包括默认参数，且传递性；
- 模块加载期调用；
- 读取它们的 getter 或已注册工具的 `summary`。

事件处理器、effect、延迟回调属于事件期，可以通过。

**没有抑制机制**：误报在检查器里用测试修复。

### 9.6.5 `check:compiler` 与 `check:pi-packages`

- `check:compiler` 对比 `scripts/compiler-census.baseline.txt`；新的 bailout 视为回归；
- `check:pi-packages` 用 tarball + Node 运行时验证原生一致性（详见第 8 章）。

这两个门禁的共同点：**它们校验的是“产物行为”，不是源码风格。**

---

## 9.7 `pi-compatibility`：升级的移除门禁清单

### 9.7.1 它解决什么问题

宿主为了兼容 pi 的缺失 API/缺陷/打包限制，会写适配代码。适配会腐烂：

- pi 修了，但没人删除；
- pi 改了，但适配还在按旧行为工作；
- 新适配没有记录，下一个人不知道它为什么存在。

`pi-compatibility.md` 是**唯一清单**：每个适配都有 owner、理由、移除门禁、验证命令。

### 9.7.2 升级契约

> 每次 pi 升级都对照**已安装包的 exports 与实现**检查每个条目，而不只是 release notes 或上游 issue 状态。
> 升级的验证摘要必须点名：移除了哪些适配、保留了哪些（附证据）、改变了哪些，以及跑了哪些测试。

规则：

- 新 workaround → 在这里加条目，并在 owning spec 写理由；
- 删除 workaround → 同时删除条目；
- parity 失败是**漂移的证据**，不是“把测试改弱”的许可；
- 生产代码 import 私有 dist 路径不算替代；这类 import 只允许出现在 parity 测试里。

### 9.7.3 移除门禁的写法

一个好的 gate 必须可判定：

| 差 | 好 |
| --- | --- |
| “等 pi 支持” | “公开的 per-session 状态读取订阅 + 可取消的管理/认证 API + 结构化日志” |
| “上游修好后” | “`AgentSession.dispose()` 会发出 extension shutdown” |
| “不再需要” | “公开 loader/validator/writer 保留 entry provenance，并允许在项目覆盖合并前准入” |

### 9.7.4 一个条目的结构

```
### <适配名>
**Owner:** [[owning-spec]] + 具体文件
<为什么需要 + 现在怎么做>
**Removal gate:** <可判定的条件>
<升级时要验证什么 + 跑哪些测试>
```

`Last re-check` 段记录最近一次升级的结论：
“No removal gate is satisfied” 也是合法结论，但要逐条说明为什么。

---

## 9.8 一次变更的工作流

### 9.8.1 九步

```
1. 定位 owning spec
2. 读 spec 与相邻 spec（边界、公开面、禁止依赖、版本门禁）
3. 如果是行为/边界变化：先改 spec
4. 改代码，保持 barrel 与 spec 一致
5. 跑最小相关测试
6. 跑快速门禁：check:deps / check:boundaries / check:seams / lint / typecheck
7. 涉及发布行为：合并实现后跑一次完整无 agent E2E（bun run e2e）
8. 涉及真实 agent 行为：跑聚焦 live spec（例如 bun run e2e:full e2e/agent-naming.live.spec.ts）
9. 交接前：审查任务范围 diff，做一次“减法检查”
```

### 9.8.2 第 3 步为什么不是形式主义

先改 spec 会迫使你回答：

- 这属于哪个模块？
- 公开面变了吗？
- 需要升协议版本吗？
- 旧客户端怎么办？
- 有没有新增跨模块边？

如果这些答案只在代码里，review 很难发现边界被悄悄模糊。
**spec 是 review 的“预期形状”**；代码是“实际形状”。两者不一致时，门禁或人应该失败。

### 9.8.3 减法检查

交接前问：

- 有没有可以删除的抽象？
- 有没有为了“以后可能需要”而加的可配置性？
- 有没有兼容层/回退在真实场景中永不触发？
- 有没有第二个状态所有者？
- 有没有重复的非平凡推导？

仓库规则：**移除可避免的抽象、状态所有者、依赖、兼容层与回退；报告实质性移除并解释保留的层。**

### 9.8.4 什么时候该写决策，而不是 spec

| 情况 | 写哪里 |
| --- | --- |
| 跨模块拓扑/不变量 | `architecture.md` 的新决策 |
| 一个模块的职责/边界 | 该模块 `SPEC.md` |
| pi 兼容适配 | `pi-compatibility.md` + owning spec |
| 一次工作的范围 | task spec（不进入持久图谱） |
| 用户可见能力变化 | `README.md` 也要检查 |
| 新增/修改门禁 | `scripts/SPEC.md` + 根 `package.json` |

---

## 9.9 规格写作规范

### 9.9.1 简洁，不重述代码

- 不复述函数签名/实现；
- 不复制另一个 spec 的整段；
- 只写“为什么、边界、代价、必须做对”。

### 9.9.2 注释近乎零

仓库规则：

> 注释近乎为零：lint/type 指令与罕见的一行危险提示可以接受；理由与叙事属于 owning spec。

因此：

- 复杂分支的理由写进 spec，而不是逐行注释；
- 一行 `// hazard: ...` 只在“下一个人会踩”的地方出现；
- 不用注释解释显而易见的代码。

### 9.9.3 单一来源

- 有限词表、字段名、依赖边、预设列表都应有唯一来源；
- 不在两个 spec 里复制同一张表；
- 不在代码与 spec 里各写一份枚举。

### 9.9.4 状态的生命周期

| status | 含义 |
| --- | --- |
| `draft` | 正在形成，可能变化 |
| `active` | 当前权威 |
| `stale` | 仍相关但已过时，需要更新 |
| `done` | 一次性工作完成 |
| `deprecated` | 不再适用，保留历史 |

`status` 不是装饰：它告诉读者“这份文字该被信任到什么程度”。

### 9.9.5 中文译本的维护

- 英文原文是唯一权威；
- 译文与原文不一致时，以原文为准；
- 改行为/边界时，英文 spec 必须先改；
- 译文可以稍后同步，但不能成为“事实上的唯一版本”。

---

## 9.10 常见失败模式

| 反模式 | 后果 | 正确做法 |
| --- | --- | --- |
| 先写代码，再补 spec | spec 变成事后描述，边界已漂移 | 行为/边界变化先改 spec |
| 把 spec 写成实现注释 | 与代码重复，必然漂移 | 只写意图/边界/代价 |
| 依赖边写进叶子 | N 份局部图互相矛盾 | 写在父模块 |
| 公开面写成散文 | 无法纳入 `check:spec-surface` | 写成裸标识符列表 |
| 只声明公开面，不查 barrel | 多导出会悄悄扩大接口 | 双向校验 |
| 用 `export *` 后不更新 spec | undeclared 违规 | 显式列出或更新声明 |
| 用 type-only import 绕过边界 | 依赖仍然存在 | `check:boundaries` 也查 type-only |
| 加 `biome-ignore`/`@ts-expect-error` 让门禁过 | 把设计问题藏起来 | 先当作设计信号；确需抑制要用户批准 |
| 新 workaround 不登记 | 下一个人不知道它为何存在 | 加 `pi-compatibility` 条目 |
| 移除 workaround 不删条目 | 清单腐烂 | 同时删除两者 |
| 把 parity 测试改弱 | 漂移被掩盖 | parity 失败是证据，不是许可 |
| 在 spec 里复制另一份表 | 两份真相漂移 | 链接到 owning spec |
| 中文译文当权威 | 与英文原文分叉 | 以英文原文为准，先改英文 |
| 任务规格算持久规格 | 项目被误判为“有规格” | `projectHasSpecs` 排除 `task-spec` |
| 用注释解释设计 | 下一个人不会去读 | 写进 owning spec |

---

## 9.11 追踪练习

### 练习 1：一份 spec 有 `id` 与 `type`，但没有 `title`，它是 spec 吗？

<details><summary>参考答案</summary>

`isSpec` 只要求 `IDENTITY_FIELDS`（`id`、`type`）存在，因此**进入图谱**。
但 `REQUIRED_FIELDS` 还包含 `title`，所以 `spec_create` 的撰写校验会拒绝一份缺 `title` 的新规格。
读取宽容、写入严格。

见 [parse.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/spec-graph/core/parse.ts#L7-L31)。
</details>

### 练习 2：为什么重复 `id` 的胜者必须依赖稳定排序？

<details><summary>参考答案</summary>

图谱按 id 建节点槽位；重复 id 时第一个文件胜出。如果顺序来自 `readdir`，
同一个仓库在 ext4、APFS、NFD 文件系统上可能得到不同胜者，
于是 `spec_get` 返回的 frontmatter/路径会随机器变化。
因此 glob 用 NFC 规范化的全序，并以原名码元决胜。

见 [core/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/spec-graph/core/SPEC.zh.md) 的“全序”段。
</details>

### 练习 3：为什么 `check:spec-surface` 要双向比较？

<details><summary>参考答案</summary>

单向只能发现“声明了但没导出”。如果 barrel 新增一个导出而 spec 没列，
公开面实际上扩大了，但调用方仍以为接口没变。
双向比较把“少导出”和“多导出”都当作 drift。

见 [specSurface.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/specSurface.ts#L533-L545)。
</details>

### 练习 4：一份 spec 的公开面写成散文，会发生什么？

<details><summary>参考答案</summary>

如果它带了 `public-surface-checked` tag，会直接 violation：
“tagged ... but its public surface is not a bare identifier list”。
如果没带 tag，它只是 `skipped`，原因记录为“not enrolled; surface written as prose”。
纳入是显式的，散文不会静默开启强制，也不会静默通过强制。

见 [specSurface.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/specSurface.ts#L122-L132)。
</details>

### 练习 5：为什么 `check:boundaries` 连 type-only import 也查？

<details><summary>参考答案</summary>

因为 type-only import 仍表示模块知道另一个模块的内部形状，边界没有真正隔离。
而且它会让类型层与运行时层分叉：编译通过，但架构上已经越界。
规则明确：type-only 不能绕过被禁止的模块或源码半边边。

见 [scripts/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/scripts/SPEC.zh.md) 的边界段。
</details>

### 练习 6：`check:seams` 的 allowlist 为什么会“过期”失败？

<details><summary>参考答案</summary>

因为 pi 升级可能移动/删除一个动态 import。
如果 allowlist 还保留旧路径，检查会报 stale：它要求你重新验证接缝是否仍覆盖替换物，
然后更新 allowlist。
这防止“allowlist 变成永久豁免”，让每次 pi 升级都必须重新看。

见 [check-binary-seams.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/check-binary-seams.ts#L122-L158)。
</details>

### 练习 7：为什么 `check:i18n` 不禁止事件处理器里的字面量？

<details><summary>参考答案</summary>

事件处理器不是渲染期。它可能在用户操作后执行，且不参与首帧/翻译器读取顺序。
规则只针对“渲染期读取全局翻译器”和“会被渲染的字面量 UI 文本”；
事件期、effect、延迟回调可以通过。

见 [scripts/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/scripts/SPEC.zh.md) 的 `check:i18n` 段。
</details>

### 练习 8：一个新 workaround 应该写进哪里？

<details><summary>参考答案</summary>

两处：

1. `pi-compatibility.md` 加条目：owner、为什么、怎么做、移除门禁、验证命令；
2. owning `SPEC.md` 写详细理由与边界。

删除时两处一起删。没有条目的 workaround 不允许存在。

见 [pi-compatibility.md](file:///Users/luzhengli/Documents/github/thinkrail/pi-compatibility.md)。
</details>

### 练习 9：为什么 `check:deps` 不允许范围版本？

<details><summary>参考答案</summary>

范围版本会让不同时间/不同机器的 `bun install` 解析出不同树，
CI 与本地可能跑不同版本。仓库要求精确 pin，并把跨包依赖集中到 catalog，
由一个 `catalog:` 引用保证全仓同一版本。
`react`/`react-dom` 还额外检查 `bun.lock` 只有一个解析版本。

见 [check-catalog.ts](file:///Users/luzhengli/Documents/github/thinkrail/scripts/check-catalog.ts)。
</details>

### 练习 10：为什么任务规格不算 `projectHasSpecs`？

<details><summary>参考答案</summary>

任务规格描述一次工作的范围，不是模块的长期边界。
如果把它算进去，一个临时任务会让项目被误判为“规格化项目”，
从而影响引导/规格查看器的行为。

见 [packages/server/src/spec/SPEC.zh.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/spec/SPEC.zh.md)。
</details>

---

## 9.12 一页速查

```
spec frontmatter
  id / type / status / title / parent / depends-on / references / implements / covers / tags
  required: id, type, title
  identity: id, type
  types: goal-and-requirements, architecture-design, module-design, submodule-design, task-spec
  statuses: draft, active, stale, done, deprecated
  is-a-spec = id && type
  read: lossy; write: lossless; body: byte-preserved

graph
  parent tree + depends-on/references/implements DAG + reverse edges
  duplicate id: first in NFC-total-order wins
  validate: dangling links / duplicate ids / parent cycles
  resolveSpecPath: root-relative, .md, not ignored, no symlink, no colon/.. escape

hierarchy
  goal-and-requirements / architecture (23 decisions) / AGENTS / pi-compatibility
  packages/*/SPEC / apps/*/SPEC
  submodule SPEC
  .zh.md = translation; English authoritative

boundaries
  Responsibility / Owns / Public surface / Allowed / Forbidden / Get right
  sibling edges live in parent spec
  barrel; exceptions: web panels/chat, ui

check:spec-surface
  enrollment = tag public-surface-checked
  block = bare identifier list (heading or bullet/paragraph label)
  barrel = dir/index.ts or dir/src/index.ts
  TS program; both directions; type-only/default/transitive; export= -> default
  unresolved/ambiguous re-export = violation
  24 enrolled (at scan time)

other gates
  check:deps       exact pins + catalog + one react
  check:boundaries manifest + static imports incl type-only, extension halves
  check:seams      opaque import() in pi dist; allowlist stale fails
  check:i18n       4 literal forms + en.json + render-time translator
  check:compiler   bailout census baseline
  check:pi-packages tarball vanilla parity

change workflow
  locate owner → read neighbors → spec first → code → tests → gates → e2e → subtraction → handoff

pi-compatibility
  each adaptation: owner + why + removal gate + verification
  upgrade checks installed exports/implementation, not release notes
  parity failure = drift evidence, not permission to weaken
```

---

### 与主指南的对应

| 主指南小节 | 本专题的展开 |
| --- | --- |
| 9.1 为什么 | 9.0、9.8.2 |
| 9.2 层级 | 9.0.2、9.3 |
| 9.3 边界规则 | 9.4、9.6.2 |
| 9.4 检查脚本 | 9.5、9.6 |
| 9.5 一次安全的修改流程 | 9.8 |

> 建议的下一步：打开一份已纳入的 spec（例如 [git/SPEC.md](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/git/SPEC.md)），
> 对照它的公开面块与 [git/index.ts](file:///Users/luzhengli/Documents/github/thinkrail/packages/server/src/git/index.ts) 的导出，
> 然后故意在 barrel 里加一个未声明的导出，跑一次 `bun run check:spec-surface`，
> 看它如何把“多导出”也判为 drift。一次亲手制造的失败，比读十遍规则更有用。
