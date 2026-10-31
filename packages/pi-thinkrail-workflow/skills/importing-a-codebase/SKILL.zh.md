# 导入一个代码库

> 中文译文，原文见 [`SKILL.md`](./SKILL.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

工作区里有真实代码，但没有规格。**逆向工程出这个项目本应拥有的规格图谱。** 当仓库已经带有
真正类似规格的文档时，围绕它们构建图谱，而不是与之并行。尽可能自己从文件中完成；只在代码确实
无法告诉你、而答案又会改变某份规格时才去问用户。

**守住 writing-specs 的质量线。** 起草之前先读那个概念 skill —— 这个流程中的一切都是推断而非
确认，因此它的诚实规则（用户评审之前保持 draft、未确认内容就地标记）在这里约束最紧。

## 1. 先读，最后才问（Read first, ask last）

在提出第一个问题之前先做勘察。大致按这个顺序阅读：

- **Agent 文件（先挖这些 —— 它们直接陈述意图 + 约定）：** `AGENTS.md`、`CLAUDE.md`、
  `.cursor/rules/*`、`.cursorrules`、`.github/copilot-instructions.md`、`GEMINI.md`、`.windsurfrules`。
- **文档：** `README`、`docs/`、`CONTRIBUTING`、ADR。
- **清单与布局：** `package.json` / `pyproject.toml` / `go.mod` / `Cargo.toml`、工作区 glob、
  `tree` 式的结构、入口点、构建/测试脚本。
- **代码：** 入口点，以及每个候选模块的顶部 —— 足以看出职责与它们之间的依赖边。

阅读过程中，收集**采纳候选**：那些持久、声明性地陈述世界当前样貌的文档 —— 架构/设计文档、
ADR / 决策记录、领域词表、协议/契约文档。永远不是候选的（只作输入）：README、CONTRIBUTING、
变更日志、路线图、TODO、实现计划（已完成的或计划中的）、生成的 API 文档。

用规格工具（`spec_grep` / `spec_graph`）确认尚未存在图谱。如果规格已经存在，停下并交回
`setting-up-a-project` 调度器 —— 本流程是给尚无规格的仓库用的。

## 2. 建立工作模型（Build a working model）

根据你读到的东西，形成一个关于这个项目**是什么**、**形状如何**的工作模型 —— 它保存在对话中，
不写入文件（本流程不宣告任何工作文件）：

```
what:       one-sentence purpose (the job the codebase does)
domain:     the space it's in
stack:      languages / frameworks / runtime
modules:    the real boundaries + the dependency edges between them (who imports whom)
invariants: rules the code already enforces (layering, "X never imports Y", public surfaces)
decisions:  non-obvious choices visible in the code (and where the "why" is missing)
```

Agent 文件与 README 通常免费给你 `what`、`invariants` 和 `decisions` —— 优先用它们，而不是从代码
重新推导。

## 3. 只访谈缺口（Interview only the gaps）

**只**问那些文件无法回答、且会改变某份规格的事 —— 典型的是：主要工作 / 它给谁用、明确的非目标，
以及某个非显然决策背后的 *why*。按 **asking-user-questions** 概念 skill 分轮访谈它们；推断出一个
具体答案让用户纠正，而不是开放式提问。

只要文件留白，就有两类问题属于同一轮 —— 代码展示*什么*，很少展示*why*：

- **工作模型中每个非显然决策背后的 why** —— 那些文件没有解释的 ——“我看到 X 是这样做 —— 为什么？”
  —— 并把你推断出的理由作为选项。答案进入拥有该决策的规格。
- **已知限制** —— 把你看到的一个个缺口（未处理的情况、TODO、缺失的测试、慢路径）做成一个
  multiSelect 让用户确认；被确认的记入所属规格。

如果存在采纳候选，在同一轮里加一个问题：一个列出它们的 multiSelect（很多时分组 —— 一组 `adr/`
算一个选项）—— 哪些应当成为规格图谱的节点？到目前为止发现的、候选与代码之间的矛盾也进入这一轮
（确认这项更正）。被跳过或被拒绝 → 一概不采纳；候选仍然只作输入，在交接时说明。

如果文件已回答了一切实质内容 —— 包括 why 与限制 —— 就**跳过访谈**并说明；不要制造问题
（仅采纳候选本身仍构成一轮 —— 这个提议绝不因“没有缺口”而被丢掉）。被跳过或被拒绝的问题不是阻塞项：
把假设就地记录在规格里，并标记为未确认。

## 4. 自上而下起草图谱（Draft the graph, top-down）

边做边用规格工具保存（每个节点 `spec_create`，正文 `edit`）。顺序：

1. **`goal-and-requirements.md`**（`type: goal-and-requirements`）—— 产品是什么以及为什么，
   采用 `writing-specs` 定下的目标文档形态；能力就是代码已经在做的事。这是图谱的根；
   已确认的意图住在这里。
2. **`architecture.md`**（`type: architecture-design`，`parent: <goal id>`）—— 拓扑、模块边界、
   真实的依赖边（只有携带真实信息时才画一个小 DAG），以及代码所强制的不变量。
3. **每个真正的模块一份简短的 `SPEC.md`**（`type: module-design`；包内的目录级模块用
   `submodule-design`；`parent:` 指向包住它的模块或 `architecture`）。每份写明它的**职责**与
   它的**边界**（允许的依赖 / 禁止的触达）。

**被采纳的文档就地成为节点。** 首先，对每个被接受的候选：仔细读它，然后在文件所在处补上规格
frontmatter（`id`、`type`、`title`、`status: draft`、`parent`；`depends-on`/`references` 只在
确实存在时加）—— 内容不动。被采纳文档填上的槽位不再重复起草：一份被采纳的架构文档*就是*
`architecture-design` 节点，一份被采纳的模块设计文档*就是*那个模块的节点，一份 ADR 只在其决策
仍然有效时才赢得一个节点。围绕它们构建图谱的其余部分，用 id 链接。“内容不动”的一个例外：
当被采纳的文档含混或已与代码漂移时，作为采纳的一部分更正那段内容，并把这项更正说出来
（及时发现时放在访谈轮里，否则在交接时说明）。

用 `parent` 镜像代码层级，`depends-on` 只加在代码确实呈现的边上。把**你起草**的每个文件都保持在
**writing-specs** 的质量线上 —— 它的粒度与“只说一次”规则决定了什么算一个模块、共享边住在哪里。
如果某个边界确实不清楚，就问，或者让那份规格保持 `draft` 并加一行注记 —— 不要大费周章地猜测。

## 5. 验证并交接（Validate & hand off）

- 运行 `spec_validate`；修复悬空链接、重复 id、父节点环。
- 告诉用户规格已在这个工作区的分支上起草好 —— **在 Changes 中评审它们；在他们批准之前不会合并**
  —— 并总结哪些是你推断的、哪些是他们确认的，哪些文档被采纳、哪些仅作输入，以及所做的任何漂移
  更正。
- 后续的特性、行为变更与缺陷修复走 `brainstorming`；不改变行为的工作直接进行。**本工作流到此结束**。
