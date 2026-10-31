# pi-todos tools（pi 包装器）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

把待办清单暴露给 agent 的七个 `pi` 自定义工具 —— `todo_list`、`todo_add`、`todo_update`、
`todo_promote`、`todo_remove`、`todo_write`、`todo_plan_summary`。每一个都是 `core/` 之上的
**薄包装**：一个 TypeBox `parameters` schema、一个针对 `ctx.cwd` 调用某个 `TodoStore` 方法的
`execute`，以及 `textResult`/`errorResult` 返回。有限词表的 `status` 参数经 `StringEnum` 从 `core/`
的元组推导其枚举，因此 schema 与模型一起移动（由 `tools.test.ts` 固定）。

**面向 agent 的约束住在本层**（core 保持宽容 —— 用户通道与宿主线上契约都使用松散条目，
且 agent 也可以撰写松散条目）：

- **松散撰写是便签板，不是计划表面：** `todo_write` 的 schema 只提供 `groups`，因此重新规划永不铸造
  或编辑松散条目。`todo_add` 接受 `group`、`after`，或**两者都不给** —— 两者都省略会把一个原始松散条目
  丢进用户与 agent 共享的预备队列。一个锚定到松散条目的 `after` 被**拒绝**：松散条目是被提升进一个组，
  而不是在其上长出同级。`TodoStore.add` 保持宽容，因为宿主通过它写入用户自己的松散通道。
- **提升是一个原子工具，不是一套舞步：** `todo_promote` 包装 `TodoStore.promote` —— 一次调用消费一个
  松散条目并浮现出一个它自己的组（同 id、同标题除非被覆盖），等待 `todo_add` 追加第一个步骤。
  这是用户来源的松散请求唯一受支持的提升路径；工具描述脚本化后续动作（`todo_add` 第一个步骤 →
  翻为 `in_progress`），而 todos skill 承载“不要退回 `todo_add group:` + `todo_remove` 舞步”的规则。
  `todo_promote` 之后立刻出现的空组是一个合法的“任务已接、计划成形中”状态
  （见 [[submodule-pi-todos-core]] 中关于空组的内容）—— 没有合成的占位步骤。
- **`todo_write` 进行调和，永不破坏性替换：** 该工具把其 `groups` 转发给 `TodoStore.replaceAll`，
  后者按组 + 步骤标题把写入的步骤匹配到既有步骤并保留其进度（见 [[submodule-pi-todos-core]]）。
  因此任务中段的重新规划是安全且无损的 —— **没有运行时的劝告**来劝阻它；todos skill 承载的是
  （仅为效率的）在单一变更上偏好 `todo_add`/`todo_update` 的倾向。
- **带内提示**（状态纪律反馈）：每个变更/列表结果在存在未完成条目但没有任何条目处于 `in_progress` 时
  追加 `consistencyNudge`；一个 `todo_update` → `done` 则改为点名该组的下一个未完成步骤
  （仅建议，永不自动启动）；被自动降级的条目被报告为 `(paused: …)`。一个使**任何地方都不再有未完成条目**
  的 `done`，还会额外提示 `todo_plan_summary` —— 总体完成摘要恰好在它应当出现的时刻被要求；
  当一次更早完成留下的摘要仍存活在计划上时（新增工作路径永不丢弃它），该提示会**回显那段文本并要求
  agent 扩展它**，而不是从头重写，于是注记对已完成的一切保持累积，而不是收窄到最后一步。
- **完成摘要（评审轨迹）：** `todo_update` 接受可选的 `summary`（做了什么/为什么 —— diff 无法体现的
  决策，加上任何范围漂移；**结构化 Markdown** —— 引导句 + 项目符号）、**`commitSubject`**
  （该步骤增量的面向 git 的主题行，以宿主仓库自己的提交风格书写 —— “读 `git log` 并匹配它”的指令
  住在 schema 描述里，因为工具是 agent 接触这个字段的唯一途径）以及 **`verification`**
  （所运行的精确检查 + 结果，或“未验证” —— 一个独立字段，以便 UI 用一个状态字形为它打徽标；
  **Markdown**，于是多项检查渲染为项目符号列表，而不是挤在一行里的 `;`，而含混或缺失的行一眼可见）——
  三者与 `status: done` 一起设置（对改代码的步骤由 skill 强制，绝不是工具门禁：
  工具无法知道该步骤是否改了代码 —— git 住在宿主侧）。`todo_plan_summary` 设置计划级交接注记
  （`TodoStore.setSummary`）—— 对整份计划中已完成的一切（每个任务、每次完成）的**累积**回顾，
  当更早的注记存活时被扩展而不是被重写；它接受提前调用，但会指出还有多少条目未完成
  （UI 只在一切都完成后才显示该注记）。计划级摘要使用 `## Summary`、`## Changes` 与 `## Testing`
  章节，因为宿主把它们逐字插入 PR 正文。
- **组优先的输出：** `formatPlan` 把每个组渲染在 `formatGroupHeader` 之下 —— `▸ <title>
  [<derived status> <done>/<total>]` —— 其步骤缩进，**然后是最后面的松散通道（共享的原始输入队列）**，
  位于 `Loose queue:` 标题之下。松散通道刻意放在最后：一个在任务中途添加的条目排在 agent 当前工作
  *之后*，因此自上而下阅读会先恢复/完成进行中的任务。与用户看到的（`TodoList`）以及 `flat()`/`list()`
  返回的是同一个两级顺序。

`shared.ts` 持有 `storeFor(ctx)`（为活跃的 `(ctx.cwd, sessionId)` 新建一个 `TodoStore` —— store 是无状态的，
因此没有缓存）、结果辅助函数、`formatTodo`/`formatGroupHeader`/`formatPlan`（工具输出中使用的渲染），
以及 `consistencyNudge`/`withNudges`。

## 公开接口（Public surface）

`index.ts` barrel：`registerTodoTools(pi)`，唯一入口点，由扩展入口（`../index.ts`）调用。

## 边界（Boundary）

- **允许的依赖：** `@earendil-works/pi-coding-agent`（类型与公开截断辅助函数）、
  `@earendil-works/pi-ai/compat`（`StringEnum`）、`typebox`，以及 `../core`（经其 barrel）。
  Node 内置模块可以创建仅用于响应的私有 OS 临时文件；项目 TODO 的持久化仍专属 `TodoStore`。
- **禁止：** 触及 `core/` 内部（经 barrel 导入）、任何 `@thinkrail/*` 包，以及 `TodoStore` 之外的项目
  文件系统访问。

## 有界输出（Bounded output）

面向模型的文本（含通知）保持在 Pi 的 2,000 行 / 50 KiB 限制内。溢出被捕获进一个私有临时文件，
该文件可能过期；捕获失败是显式的，且永不让一个已完成的变更失败。工具错误设置 `isError: true`，
并即使捕获失败也保留一个可见的错误标记。结构化 details 仍是展示数据，位于模型上下文之外。

只要整份渲染后的计划能放进一个在 Pi 限制之下为通知预留的载荷预算（1,990 行 / 48 KiB），
`todo_list` 就返回它的全部（组优先；松散队列与提示最后）。只有更大的计划才分页，按渲染行偏移并使用
同一个状态过滤器；偏移寻址的是当前计划而不是快照，因此计划编辑之后要重新开始。
刻意没有页大小参数：一个短页只承载长计划中已完成的开头，会隐藏进行中的任务、松散队列与提示；
`status` 才是读得更少的方式。
