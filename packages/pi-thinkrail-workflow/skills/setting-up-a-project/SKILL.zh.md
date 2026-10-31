# 搭建项目 —— 调度器

> 中文译文，原文见 [`SKILL.md`](./SKILL.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

把一个项目变成规格图谱的单一入口。先弄清自己处在哪种情形，然后遵循与之匹配的 skill —— 不要自己
临时发挥流程。

## 1. 探测（Detect）

查看工作区并使用规格工具（`spec_grep` / `spec_graph`）：

- 是否已经存在规格图谱（一份 `goal-and-requirements.md`，或任何 `SPEC.md`）？
- 是否存在真实的源码，还是仓库为空 / 近乎为空（只有一个 README 或脚手架）？

## 2. 路由（Route）

| 情形 | 遵循 |
|---|---|
| **已有规格** | 不要重做。简要提出评审/扩展图谱（补齐明显的缺口 —— 缺失的 `architecture.md`、尚未成规格的模块）。其他工作照常路由：特性、行为变更与缺陷修复走 `brainstorming`；不改变行为的工作直接进行。拒绝 → 停下。接受 → 见下文的 **图谱扩展**。 |
| **无规格 · 空 / 近乎为空的仓库** | **`starting-a-new-project`** skill —— 访谈用户，把想法变成 `goal-and-requirements.md`。 |
| **无规格 · 真实的源码** | **`importing-a-codebase`** skill —— 分析代码 + agent 文件并起草首张规格图谱，只询问代码无法揭示的意图。 |

阅读并遵循那个 skill 的步骤。

## 图谱扩展（Graph extension (no dedicated skill)）

没有专属 skill 覆盖扩展现有图谱。如果该提议被接受，用自己的判断向用户的请求对齐并适配，
守住 **writing-specs** 的质量线，且不做路由宣告。

## 交接（Handoff）

以恰好点名下列之一结尾：**starting-a-new-project**、**importing-a-codebase**，或 —— 当规格已经
存在时 —— 上文的评审/扩展提议（拒绝 → 停下；接受 → 依据 **writing-specs** 的质量线凭判断做图谱
扩展）。
