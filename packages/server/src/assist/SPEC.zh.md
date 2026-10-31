# assist —— 临时性一次性任务

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

小而**尽力而为**的 agentic 辅助函数，执行单次廉价模型补全——即“临时性一次性任务服务”。每个任务
拥有自己的 prompt、输出解析/校验与优雅降级回退；它绝不阻塞或拖垮调用方。当前任务在 agent 未撰写
摘要时，为一个已完成计划起草摘要。为聊天与工作区命名**不是** assist 任务：主 agent 通过
`set_title` 为它们命名（见 [[submodule-server-agent]]），因为每一种辅助模型命名方案产生的名字
用户都想改。PR 描述是第二个任务。这些任务是**库接口**——没有线上方法；消费者是宿主侧流程。

## 边界（Boundary）

- **拥有：**
  - 任务目录 + 它们的 prompt、输出校验与回退。始终受时间盒限制（`AbortSignal.timeout`）且受
    上限约束（`maxTokens`）。
  - `setOneShotRunner(fn)` —— 一个测试接缝，用于替换一次性运行器（默认 = `agent.completeOnce`），
    使任务可以对着假实现做单元测试，无需 pi/鉴权/网络。
  - `suggestPlanSummary(steps)` → 为已 COMPLETED 的计划给出一段简短的 **Markdown** 交接说明，由其
    已完成的步骤草拟（`{ title, summary?, verification? }[]`，由调用方构建——assist 不读任何
    store），经由同一条无工具的廉价模型路径，或返回 `null`（尽力而为：无鉴权 / 超时 / 没有可用
    步骤 / 输出为空时返回 `null`，绝不抛错）。`toPlanSummary(raw)` 是纯输出校验器（剥掉代码围栏
    和开头的 `Summary:` 标签，截断长度，再重塑为 `## Summary` / `## Changes` / `## Testing`——
    缺失的章节得到占位符，缺失 Summary 时取第一个标题之前的文本；为空时返回 `null`）。只有在
    结果非 null 时宿主才会持久化它；它绝不覆盖 agent 撰写的 `plan.summary`。
  - `suggestPrDescription(input)` → 由调用方构建的输入（`{ branch, base, commits, diffStat, plan?,
    template? }`，已由 `pr` 施加边界）草拟的 Markdown PR/MR 描述，或按同样的尽力而为条款返回
    `null`。有 `template` 时草稿保留模板的标题与清单并填充其正文；`toPrDescription(raw)` 是输出
    校验器（剥掉代码围栏，截断长度，为空时返回 `null`）。
- **公开接口（barrel）：** `setOneShotRunner`、`suggestPlanSummary`、`toPlanSummary`、
  `suggestPrDescription`、`toPrDescription`、`OneShotRunner`、`PlanSummaryStep`、
  `PrDescriptionInput`。
- **允许的依赖：** `agent`（`completeOnce`/`OneShotRequest`/`OneShotResult` 原语，经其 barrel）；
  Node。
- **禁止：** `host`；**直接依赖 `@earendil-works/pi-ai` / `pi-coding-agent`**（模型访问 + 派发
  属于 `agent`）；深入另一个特性的内部。

## 必须做对（Get right）

- 每个任务都**降级为 `null`**——失败绝不能阻塞调用方，也不能作为错误浮出。包裹运行器调用；由
  调用方决定改写入什么（以及是否写入）。
- 绝不相信模型的格式：限制 `maxTokens`，然后自行归一化/截断文本。
- 这些是共享事件循环上无状态、无副作用的 `fetch`（无磁盘、无工具）——可安全并行运行；不需要
  管理器/注册表。
