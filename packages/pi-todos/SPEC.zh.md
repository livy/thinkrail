# pi-todos 扩展 —— 聊天 TODO 列表

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

`pi-todos` 是一个可移植的 pi-package，为 `pi` agent 提供**以聊天为作用域的 TODO 列表** ——
它对该会话而言是工作计划，用户也可以向其中添加条目。它是聊天 TODO 计划 UX 背后的*引擎*
（[[submodule-web-chat]] 的 "Chat TODO plan"），参照 [[module-spec-graph]] 建模：一个 skill、
六个 `todo_*` 自定义工具，以及一条 `before_agent_start` 规则。

- **`index.ts`** —— 一个 `ExtensionFactory`，注册六个工具和一条始终启用的
  `before_agent_start` 规则。该规则刻意保持**简短且字节稳定** —— 只声明存在共享列表 +
  `todo_*` 工具，并给出加载 todos skill 的阈值：用户显式请求，或至少三个实质性执行步骤，
  且对任务的理解已足以做计划。该阈值管辖的是*创建*计划，而不是遵守计划 ——
  一个松散条目（双方都可以创建，见下文）在 agent 将其纳入工作时被提升为正式分组，
  而不是原地推进。杠杆是*理解*，而非提示词体量：
  **如何使用这个列表写在 skill 里；每个工具的不变量写在它自己的 description 里。**
  （曾尝试把实时列表注入每个提示词，随后又撤回了 —— 改由工具 + skill 承载。）
  该规则作为 pi 的 `systemPromptOptions.sections` 中的 `pi-todos` 条目生效，就地修改，
  绝不作为强制的 `systemPrompt` 返回 —— 机制与理由同 [[module-spec-graph]]。
- **`core/`** —— 不含 pi 依赖的模型（[[submodule-pi-todos-core]]）：`Todo` 类型与按会话的
  `TodoStore`（读-改-写 `.thinkrail/context/todos/<sessionId>.json`）。它不 import 任何
  `@earendil-works/*`，因此宿主可以值导入 `pi-todos/core` 来驱动计划查看器 ——
  读取计划，并写入用户自己的编辑（`spec/` → `spec.graph` 模式）。
- **`tools/`** —— 六个 `todo_*` 自定义工具（[[submodule-pi-todos-tools]]），是对 `core/` 的薄包装。
- **`skills/todos/SKILL.md`** —— 内置 skill：聊天计划纪律 —— 分组 = 任务（一次用户请求，
  以结果命名标题；通常 3–7 个实质性、可验证的步骤），严格按顺序处理任务，同一时刻只有一个步骤
  `in_progress`（被阻塞的任务 = 说明原因、告知用户、继续下一个），并在选择下一个条目之前、
  用户输入之后、以及完成之前，核对用户的实时编辑。被纳入工作的松散条目会被提升（新建分组，
  配以恰当的结果标题 + 分解步骤，并移除原始松散条目）；针对较小任务的"不做计划"规则
  只适用于从未进入队列的普通聊天请求。

## 工具（The tools）

| 工具 | 用途 |
| --- | --- |
| `todo_list` | 读取当前计划，以**分组优先**方式渲染（每个分组位于派生状态 + 已完成/总数表头之下），可选按状态过滤。 |
| `todo_add` | 添加一个条目 —— 加入某个 `group`、`after` 某个已有条目，或作为**松散条目**（两者都省略）—— 这是 agent 与用户共同写入的共享原始输入队列。对松散条目使用 `after` 会被拒绝（松散条目会被提升为分组，而不是长出兄弟条目）。 |
| `todo_update` | 修改条目的状态 / 标题 / 备注 / 制品 —— agent 就是这样翻转 `pending → in_progress → done`。会报告被自动降级（`paused`）的条目；把某条翻成 `done` 时会提示该分组的下一开放步骤。 |
| `todo_remove` | 删除一个条目。 |
| `todo_write` | 从全新的**仅分组**内容**核对**agent 的计划 —— 每个任务一个分组，步骤在其内部，在对任务的理解足以做计划之后写入。保留身份，而非破坏性替换：见下文。 |
| `todo_plan_summary` | 设置/清除计划级完成摘要（`TodoFile.summary`）—— 整个计划完成时写入的总体交接说明。 |

**分组 = 任务。** 计划的模型是两层的：一个分组就是一次用户请求（标题 = 结果），其条目是步骤。
分组自身的状态是**派生的、从不存储**（`core/` 中的 `groupStatus`：全部完成 → `done`，
任一 in_progress → `active`，否则 `pending`），因此不会与步骤产生偏差。宿主通过该辅助函数读取它，
并在 wire DTO（`TodoGroupItem.status`）上传递，因此没有客户端会重新派生它 ——
一张真值表，一个归属地。
两个不变量由结构保证，而不是靠模型记忆：**整个计划中恰好一个 `in_progress`**
（设置它会把前一个自动降级回 `pending` —— 在结果中报告为 "paused"），以及
**`todo_write` 永不触碰松散条目通道**（`WritePlan` 没有 `todos` 字段，因此重新计划
无法生成、编辑或删除松散条目）。松散条目本身对双方开放 —— 用户从 UI 添加，agent 通过
不带 `group`/`after` 的 `todo_add` 添加 —— 因此松散条目就是共享的原始输入队列；skill 承载
"纳入工作时即提升"的纪律（agent 把松散条目纳入工作时，将其提升为一个带恰当标题 + 分解步骤的
分组，并移除原始松散条目，而不是原地推进原始文本）。状态纪律会带内反馈：当存在开放条目
但没有任何 `in_progress` 时，工具结果会追加一条提示；把某条翻成 `done` 时会点名该任务的
下一开放步骤 —— 仅建议，绝不自动启动（那会在 agent 停下时伪造"处理中"）。

工具从 `ctx.sessionManager.getSessionId()` 解析它的列表，因此它读写的永远是
自身所运行会话的列表。

## 范围与持久化 —— 每个聊天一个列表（Scope & persistence — one list per chat）

该列表**以聊天会话为作用域**，而不是 worktree：每个会话一个 JSON 文件，
位于 worktree 根下的 `.thinkrail/context/todos/<sessionId>.json` —— 就在宿主预置、
被 git 忽略的临时 `context/` 暂存目录中，因此这些计划与其他按会话的工作文件放在一起。
它是 agent 对该会话的工作计划；用户可以（从 UI）向其中添加条目，agent 在其下一轮次
（`todo_list`）拾取它们，并通过 `todo_update` 推进这些确切的条目。文件是唯一事实来源 ——
`TodoStore` 在每次操作时重新读取它 —— 因此 agent 的会话内写入与用户的 UI 编辑会收敛，
没有陈旧窗口；缺失或损坏的文件读作空列表。按聊天临时存在（被 gitignore），
不随仓库提交。

## 状态归属与来源（Status ownership & provenance）

状态**归 agent 所有**：agent 在推进其计划时通过 `todo_update` 翻转
`pending → in_progress → done`（store 在每次设置新的 `in_progress` 时自动降级前一个）。
当前 UI 从不切换状态 —— 它的编辑面只有添加 / 删除。（`todo.update` wire 方法存在并接受
状态，但今天没有 UI 路径调用它；它是预留的，不是用户的杠杆。）

每个条目携带 **`origin`**（`agent` | `user`）—— UI 添加的是 `user`，agent 的工具写入 `agent`。
`todo_write` 是**保留身份的核对，不是替换** —— 写入的步骤按 `(group title, step title)`
与已有步骤匹配，并保留其 id/status/summary/verification/commitSubject/artifacts
（只刷新 `note`；匹配项上写入的 status 会被忽略 —— status 通过 `todo_update` 推进）。
未匹配的写入步骤会被创建；分组内被省略的 **agent 开放**步骤会被删除；
**松散条目与任何 `done` 条目始终保留** —— 因此重新计划绝不会丢掉共享原始输入队列或
已完成历史，重跑它是无损的。**松散通道是共享原始输入**：UI 标出 `user` 条目，
让人仍能看到哪些是自己的；`WritePlan` 没有 `todos` 字段，所以 `todo_write` 无论来源如何
都无法生成或编辑松散条目。完整的核对契约见 [[submodule-pi-todos-core]]
（按标题匹配是被接受的限制 —— 重命名会读作新步骤）。

## 制品（Artifacts）

条目可以通过 **`artifacts`** 关联它产出的东西 —— `kind: "file" | "change" | "spec" | "commit"`、
可选的 `label`，以及按 kind 区分的 worktree 相对 `path`（`file`/`change`/`spec`，
对 `spec` 还加一个持久的 `specId`）或一个 `sha`（`commit`）。归属按 kind 划分：
**agent** 通过工具附加 `file`/`spec`（`spec` 来自 `spec_create` 的 `{path,id}`）；
**`change` 与 `commit` 归宿主所有** —— 当 agent 把条目标为 `done` 时，宿主提交该条目的工作
并只记录 `sha`（一个 `commit` 制品 —— 文件列表在读取时从 git 派生，从不反范式化）；
`change` 的路径列表只是宿主的**不提交回退**，见 [[submodule-server-todos]]。
不含 pi 依赖的 `core`/`tools` 从不接触 git —— 它们只存储交给它们的制品。除制品之外，
已完成条目可以携带 **`summary`** + **`verification`**（复核轨迹）和 **`commitSubject`**
（宿主提交时使用的面向 git 的标题）；条目离开 `done` 时这三者都清空，
见 [[submodule-pi-todos-core]]。磁盘文件的 `version` 是 `6`（`3` 加入制品，`4` 加入
`commit` kind，`5` 加入 `summary` 字段，`6` 加入 `commitSubject`）；
较旧的文件在下一次写入时升级。

## 边界（Boundary）

- **允许的依赖：** `@earendil-works/pi-coding-agent` + `@earendil-works/pi-ai/compat`
  （`StringEnum`）—— 类型/公开截断辅助函数/compat，作为 peer 依赖 —— 以及 `typebox`
  和 Node 内置模块。`core/` 只用 Node 内置模块。工具层仅用于响应的临时文件
  不改变规范 TODO store 的归属（见 [[submodule-pi-todos-tools]]）。
- **禁止：** 任何 `@thinkrail/*` 包、`apps/web`、`packages/server` 内部 ——
  只通过工具*名称*触达，绝不通过 import。宿主只以一种方式触达本包：`pi-todos/core`
  （值导入，不含 pi 依赖）用于查看器，加上通过 `additionalExtensionPaths` 的扩展入口。
- **可移植。** 与 `pi-thinkrail-workflow` 不同，本包不假设任何 thinkrail 专有的宿主工具；
  它在原版 pi（`pi install`）和 thinkrail 中同样运行。

## thinkrail 集成（thinkrail integration）

`packages/server/src/agent/extensions.ts` 以与 `pi-spec-graph` 相同的方式添加本包：
`require.resolve("pi-todos/index.ts")` 放入 `additionalExtensionPaths`，其 `skills/` 目录
放入 `additionalSkillPaths`；`packages/server/package.json` 携带
`"pi-todos": "workspace:*"`；编译二进制生成器（`apps/cli/scripts/build-binary.ts`）
把它作为值导入的工厂打包以保持一致。

## 测试（Testing）

`core/core.test.ts` 针对真实临时目录钉住 store 的契约（add/update/list/remove/replaceAll、
按会话隔离，以及损坏文件与非法条目的降级）。`tools/tools.test.ts` 通过一个伪
`ExtensionAPI`（带桩的 `sessionManager.getSessionId`）在临时 cwd 下驱动每个工具的
`execute` —— 参数传递、未知 id 的错误路径，以及有限词汇的参数从 core 元组派生枚举。
