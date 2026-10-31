# subagent 渲染器（Agent / get_subagent_result）与运行详情辅助函数

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

**`pi-subagents`** 能力的展示型渲染器，按工具名接入：`AgentCard`（`Agent` 工具 —— 同时也注
册给 `get_subagent_result`，后者的结果携带同样的 `DelegationRunDetails`），以及纯函数模块
`runDetails`（防御式的 `DelegationRunDetails` 读取器、用量/token/成本/时长格式化器、折叠态
头部摘要行，以及完成报告辅助函数）。一个分离式运行注入的 `subagent-completion` 自定义消息
**不在**这里渲染：它是由父级 chat 拥有的后台结果行（`BackgroundResult.tsx` 叠在
`resources` 的 `ResultRow` 之上，见 [[submodule-web-chat]]），消费 `usageCounters`、
`completionReport` 与 `reportTeaser`。运行的*活跃性*有意不在这里推导：转录对话框从每个
`subagent.getTranscript` 响应上读宿主的注册 `status`（理由：父级 chat SPEC 的转录视图接缝）。

## 渲染约定 —— 用户已定案，2026-08，有调研支撑（The rendering convention (user-settled, 2026-08, research-backed)）

调研过的对象：Claude Code（折叠成一行，该行实时跳动 ——
`Running… 45s · 12 tool uses · 30k tokens` →`Done (…)`；详情按需展开）、Codex（常驻的实时
运行时面板 + 简短完成通知）、OpenCode（紧凑可点击的盒子 →打开子会话；单行的后台完成通知）。
采纳的方案：

- **`Agent` 是一等公民的、开箱即用的 `ToolCard` —— 默认折叠，但它的头部摘要*就是*那条实时
  行**：`role · N turns · X tok · $c · elapsed · current-step`，从每个
  `partialResult.details` 的 REPLACE 快照重新推导。展开后：任务、实时活动行、运行所用的模
  型标签（`RunModelLabel`：目录名 + effort，tooltip 显示原始 `provider/id`），与用量计数器
  并列、打开转录的动作，以及进入终态后折叠在 fold 后面的最终报告。错误会自动展开（沿用既有
  chrome）。**失败的运行保留转录动作**：错误结局抛出的工具结果仍携带该运行的最终 details
  —— `pi-subagents` 通过它的 `tool_result` 覆写把详情重新注入（其 SPEC，PR #304 评审发现）
  —— 因此 `details.childSessionId` 这道门禁在实时路径与水合路径上都会亮起（水合时钉住）。
- **后台运行的工具卡片在其 ack 处冻结**（工具 promise 结算后 pi 忽略 `onUpdate`）—— 这是有
  意的：卡片说"started in the background"，实时阶段由 Resources dock 承载，父级 chat 的
  **结果行**才是终态信号（OpenCode 的通知模式）。它的状态沿用 Resources 的词汇表 —— 一个被中
  止（例如撞上轮次上限）的运行读作 Stopped，绝不是 success（PR #304 评审发现）。
- `completionReport` 剥掉 pi-subagents 的 `Subagent "<role>" (<id>) <status>:` 头部（按这
  一确切的内容格式做拼接是有意的；不匹配的文本整体透传）；`reportTeaser` 是报告的第一条可
  读行，已去掉 Markdown 块标记与强调符号。
- `get_subagent_result` 保持**常规**（折叠进活动流），正文 + 摘要相同，唯一区别是父级 chat
  会把对已完成运行的一次拉取渲染为该运行的后台结果行（[[submodule-web-chat]]，仅首次回执）。

## 边界（Boundary）

- **拥有：** `AgentCard`、`runDetails`（纯函数、有单元测试），以及它们的注册
  （`register.ts`，由父级 `tools/register` 以副作用方式导入）。
- **公开接口：** 副作用 `register`；`runDetails` 的纯辅助函数（父级 chat 的
  `BackgroundResult` 读取用量与报告辅助函数）。
- **允许的依赖：** 父级 chat 原语（`toolRegistry`、`ChatActions`、`Markdown`、`foldState`、
  `ModelEffortLabel`）；同级 `Collapsible`；`@thinkrail/extension-api/web`（渲染契约 +
  纯辅助函数）；`contracts`（仅类型 + `SUBAGENT_COMPLETION_CUSTOM_TYPE` 守卫家族）；
  `@remixicon/react`；`lib`。
- **禁止：** 值导入任何 `pi` 包、`pi-subagents` 或 `pi-delegation`；`store`/`transport`
  （渲染器保持展示型 —— 转录对话框作为集成文件放在 `chat/` 里，只能通过
  `ChatActions.openSubagentTranscript` 触达）。

## 必须做对（Get right）

- **防御式渲染：** `DelegationRunDetails` 以无类型的 `result.details` 到达 —— 通过
  `readRunDetails` 收窄（childSessionId/status/task 齐备），绝不信任其形状；在第一次更新落
  地之前回退到 `args`（`subagent_type`、`task`）。
- `partialResult` 是 **REPLACE** —— 每个快照都是完整的；渲染器侧不累积任何东西。
- 工具名 `Agent` / `get_subagent_result` 必须与 `pi-subagents` 能力完全一致 {D} 名字就是连
  接键。
- 仅使用 token 工具的样式（不用裸十六进制 / 内联 `style`）。
