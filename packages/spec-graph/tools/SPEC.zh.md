# 规格图谱工具（pi 包装器）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

把规格模型暴露给 agent 的七个 `pi` 自定义工具 —— `spec_grep`、`spec_get`、`spec_graph`、
`spec_create`、`spec_update`、`spec_delete`、`spec_validate`。每一个都是 `core/` 之上的**薄包装**：
解析 typebox 参数、调用一个 `core/` 函数、格式化一个文本结果 + 结构化 `details`。没有工具编辑散文。

## 边界（Boundary）

- **拥有：** 工具注册、参数 schema 与结果格式化；按根的 `SpecIndex` 缓存与 `spec_create` 的脚手架标题
  （`shared.ts`）。
- **公开接口：** `index.ts` **barrel**，导出 `registerSpecTools(pi)`；扩展入口（`../index.ts`）
  是唯一的调用方。
- **允许的依赖：** `core/`（经其 barrel）、`@earendil-works/pi-coding-agent`（类型与公开的截断辅助函数）、
  `@earendil-works/pi-ai`（`StringEnum`，从包根导入）、`typebox`、Node 内置模块。
  项目写入仍限于写工具；任何工具都可以把超大响应捕获进私有的 OS 临时文件，绝不写进项目。
- **禁止：** 触及 `core/` 的叶子文件（只导入 barrel）；任何 `@thinkrail/*` 包。

## 叶子（Leaves）

每个工具一个叶子（`grep.ts` → `spec_grep`，以此类推），各自包装对应的 `core/` 函数并依赖
`shared.ts`（索引缓存 + 结果/脚手架辅助函数），后者依赖 `core/`。叶子彼此不依赖；`index.ts` 组合它们。
`spec_create`/`spec_update`/`spec_delete` 写入文件；其余四个只读。

## 不变量（Invariants）

- 面向模型的文本（含通知）保持在 Pi 的 2,000 行 / 50 KiB 限制内。溢出被捕获进一个私有临时文件，
  该文件可能过期；捕获失败是显式的，且永不让一个已完成的变更失败。工具错误设置 `isError: true`，
  并即使捕获失败也保留一个可见的错误标记。结构化 `details` 仍是展示数据，位于模型上下文之外。
- `spec_grep` 按从零开始的匹配偏移分页，同时受条数与格式化字节上限约束。用同样的过滤条件跟随
  `nextOffset`；偏移读取的是当前已排序的源，而不是快照，因此文件变化时重新开始。
  一个超大的匹配被落到文件，而不是被跳过或切分。
- 工具永不编辑散文；`spec_update` 只动 frontmatter，且永不移除文件的规格身份 —— 它不会删除或清空
  `id`/`type`，也不会重命名 `id`。它设置/移除标量字段，并通过 `addList`/`removeList` 在每个列表字段
  （`depends-on`/`references`/`implements` + `covers`/`tags`）上增加/移除条目；`set` 拒绝列表字段
  （否则会把一个标量按列表强转成一条错误条目）。编辑由 `core` 的 `updateFrontmatterText` 就地施加，
  因此注释、嵌套/未知字段、字段顺序与文件的换行符都被保留。
- 工具读写的 frontmatter 字段键来自 `core` 的 `FIELDS` 注册表，而有限词表上的参数使用由 `core/` 元组
  播种的 `StringEnum`（`spec_create.type` ← `SPEC_TYPES`、`spec_create.status` ← `SPEC_STATUSES`、
  `spec_graph.direction` ← `SLICE_DIRECTIONS`、`spec_graph.edge` ← `LINK_KINDS`）—— 绝无重新手打的字面量，
  因此一次 `core` 重命名会无改动地流到这里（由 `tools/tools.test.ts` 固定）。
- `spec_create` 只写索引之后能读回来的东西，并在三处检查这一点：目标路径经过 `core` 的
  `resolveSpecPath`（它同时产出该工具报告的规范相对路径，因此其 `Created <path>` 永不会与 `spec_get`
  将要说的话不一致）、组装后的字节在任何写入之前必须能解析回一个规格、而写入本身是排他的
  （`flag: "wx"`），因此存在性检查无法被竞争，也无法被一个链接满足。
- 规格根是 `ctx.cwd`；每个根复用一个 `SpecIndex`（新鲜度在 `core/` 中处理 —— 见 `module-spec-graph`）。
  `spec_update` 通过 `recordForId` 读取，以复用扫描的缓存读取；写工具只负责写，
  下一次读取会把变更拾起。
