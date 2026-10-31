# pi-todos core（不含 pi 的模型）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

不含 pi 的 TODO 模型：`Todo` 类型（状态词汇表）与 `TodoStore` —— 一份**按会话**的清单，
以单个文件 `.thinkrail/context/todos/<sessionId>.json` 存储（位于临时上下文暂存目录下），
按读-改-写方式读写。该文件是事实来源；store 不持有可变状态，因此每个方法都重新读取，
过期的读取不可能发生（agent 在会话内的写入与 UI 的编辑汇聚到同一个文件）。构造上即健壮：
缺失或损坏的文件读作空清单，未知/非法字段在读取时被丢弃（`sanitize`），
因此一个手工编辑过的文件永不致会话崩溃。

**产物。** 一个条目可以携带 `artifacts` —— 指向工作产出的链接：`kind: "file" | "change" |
"spec" | "commit"`、一个可选的 `label`，并按 kind 携带一个相对 worktree 的 `path`
（`file`/`change`/`spec`，且 `spec` 还带一个持久图谱的 `specId`）或一个 `sha`（`commit`）。
模型只存储它们；它不解析路径、不计算 diff、不碰 git。`file`/`spec` 由 agent 附加
（`spec` 自然来自 `spec_create` 的 `{path,id}`）；`change` **与** `commit` 由一个条目到达 `done`
时由宿主附加 —— 宿主提交该条目的工作并只记录 sha，或当它无法提交时回退为 `change` 的路径列表
（见 `server/src/todos` —— store 保持不依赖 git）。`sanitize` 会丢弃缺少其键的条目
（`commit` 缺 `sha`，其他任何 kind 缺 `path`）。磁盘上的 `version` 是 `6`（`3` 加入 `artifacts`，
`4` 加入 `commit` kind，`5` 加入 `summary` 字段，`6` 加入 `commitSubject`）；
更旧的文件可干净读取，并在下一次写入时升级。

**摘要（评审轨迹）。** 一个条目可以携带 `summary` —— agent 的完成注记（做了什么/为什么、
diff 无法体现的决策，以结构化 Markdown 书写）—— 以及 **`verification`**，一个独立字段，
用于记录所运行的精确检查 + 结果（或诚实的“未验证”），与散文分开保存，以便 UI 用一个状态字形
为它打徽标；二者都是 **Markdown**（多项检查读作一个项目符号列表，而不是挤成一行），
缺失的验证行一眼可见；二者在条目翻转为 `done` 时经 `TodoPatch` 设置；计划本身也可携带一个
计划级 `summary`（`TodoFile.summary`，由 `TodoStore.setSummary` 写入）—— 整个计划完成时 agent
写下的总体交接注记；它是**累积的**（涵盖整个计划中已完成的一切，当先前的注记在一次重新完成中
存活下来时被扩展而非重写 —— 工具层会回显存活下来的文本，以便 agent 在其上继续构建），
绝不是只对最后一步的复述。二者在后续编辑中都被逐字保存，只有一条失效规则：一旦某个条目的
`status` 离开 `done`，`update` 就清除该条目自己的 `summary`/`verification` —— 除非同一个 patch
同时提供了新值，此时以新值为准。**计划级 `summary` 在重新打开后存活**（它不会随条目字段一起被丢弃）：
它保持陈旧，以便读者在计划被重做期间把它显示为 “Updating…”，而下一次完成会扩展它而不是从空白开始
（累积契约）。把陈旧的注记挡在某个显示或导出之外，是读者的职责。

**`commitSubject`（面向 git 的标题）。** 第三个完成时字段，同族且同一失效规则：宿主在提交该条目
的增量时逐字使用的主题行（见 [[submodule-server-todos]]）。它存在的原因是 `title` 与提交主题是
**写给不同读者的不同文本** —— `title` 是用户状态面板里的一个计划步骤（“Newest-first chat order”），
而主题落在仓库的永久历史里、与人类提交并列，必须匹配那段历史自己的约定
（“feat(web): add newest-first chat order”）。从一方推导另一方是不可能的（变更的类型/范围只能从
变更本身得知），因此 agent 撰写两者，store 只承载它们。模型保持不依赖 git：它永不校验风格，
永不读 `git log` —— “匹配本仓库历史”的指令由工具描述与 todos skill 承载，字段缺失时宿主回退到
`title`。读者以“一切都完成”为条件门禁计划级摘要（UI 实时显示它，在重新打开期间标记为陈旧；
不带门禁的消费者，例如 PR 正文/工作报告，只在每个条目都是 `done` 时渲染它），因此一段陈旧的
“全部完成”叙事即使留在磁盘上也不会外泄。`replaceAll` 刻意**不**继承计划摘要 —— 新计划就是新工作。
评审*状态*永不存于此处：它归用户所有，住在宿主的 sidecar 中（见 `server/src/todos`），
因此 agent 的重新规划无法翻转一个评审决策。

**组 = 任务。** 一个组建模一个用户请求；其条目就是步骤。组的生命周期是**推导而来、永不存储**的：
`groupStatus(group)` —— 全部 done → `done`，任一 in_progress → `active`，否则 `pending` ——
因此它不可能与步骤漂移。它只有**一个家**：宿主通过这个辅助函数读取它，并把结果放到线上 DTO 上
（`TodoGroupItem.status`，见 [[submodule-server-todos]]），因此 `apps/web` —— 它只能导入 `contracts` ——
渲染它，而不是保留第二份真值表。

**`replaceAll` 是一次保持同一性的调和，而不是替换。** `todo_write` 发送一份*期望的*计划；
模型用它来说“让计划看起来像这样，但保留进度”，因此 `replaceAll(plan)` 进行调和，
而不是从头重建：

- 每个写入的分组条目按键 `(组标题, 条目标题)` **匹配**到一个既有的 **agent** 条目 —— 两侧都比较
  *解码后并扁平化为一行*（见下）；第一个未被消费的匹配胜出（重复标题按位置调和）。
  一个**匹配会复用既有条目**：其 `id`、`createdAt`、`status`、`summary`、`verification`、
  `commitSubject` 与 `artifacts` 都保留；只有 `note` 从写入中更新。写入中的 `status`
  **对匹配条目被忽略** —— 状态只通过 `update` 前进，因此一个被重新列出的 `in_progress`/`done`
  步骤永不会被敲回 `pending`。一个**未匹配**的写入条目被新建。
- **剩余项**（没有任何写入条目匹配到的既有条目）：松散通道完整存活 —— `origin: "user"` 的条目
  *以及*未 `done` 的 `origin: "agent"` 条目都会带过来（松散通道是共享的原始输入便签板，见下）；
  组内的 `done` 条目被保留（重新加入同标题的新组，否则在其原组之下带过来，追加到新组之后）；
  **组内 agent + 未完成条目**被**丢弃** —— 那正是合法的“删掉了一个步骤”。
  **提升所生的空组**（`todos[]` 为空的组 —— 见下文的一等公民规则）在新计划没有重新发出同一标题时，
  连同其原 id 一起向前带；丢弃它们会静默丢失一个被搁置的用户请求，而提升契约承诺要保留它。
  松散通道里一个剩余的 **agent `done` 条目**（超出契约 —— 松散条目是 agent 本应提升或移除、
  而非就地完成的预备注记）会被带进一个 `"Completed"` 组，而不是留在松散通道，因此一个 `done`
  的松散条目永不会被静默丢失。后果：重写同一个计划对进度是**空操作**（状态不重置、无重复、无孤立）。

**松散通道 = 共享的原始输入便签板。** `TodoPlan.todos`（UI 渲染为 “Other” 的通道）承载来自
**两种**来源的未处理条目：人类从 UI 输入的 `user` 条目，以及 agent 通过不带 `group`/`after` 的
`todo_add` 丢下的 `agent` 条目。松散通道刻意是*工作之前*的 —— 原始注记、种子、半成形的请求。
当 agent 把一个松散条目纳入工作时，它用 `TodoStore.promote(id, titleOverride?)` 在单个原子步骤中
**提升**它：该松散条目从通道中被剪出，一个具有**相同 id**（以及该条目的标题，除非被覆盖）的组
出现在组列表中，等待 `add` 追加分解。一次写入，没有重复窗口 —— 一个在提升后立刻死掉的会话仍留下
一致的计划（用户的请求显示为其自己的组）。该组被有意创建为空（空组状态是一等公民，见下）。

`promote` 返回一个**可判别结果**（`TodoPromoteResult`）：成功时 `{ ok: true, group }`，
否则 `{ ok: false, reason }`，reason 是四个 `TodoPromoteFailure` 成员之一。
结果形态之所以存在，正是因为 agent 的恢复方式随失败而异 —— `title-collision` 可用一个消歧的
`title:` 重试，`grouped-step` 根本无法提升，`unknown-id` 指向一个陈旧的 id，
`empty-title` 需要一个非空的覆盖值。在每种失败情形下，松散条目都被留在原处，使重试能够成功，
且 `todo_promote` 工具包装器按每个 reason 发出一个定制的错误，使 agent 永不退回已弃用的
`todo_add group:` + `todo_remove` 那套舞步。四个 reason：

1. `unknown-id` —— 该 id 在计划中根本不存在；
2. `grouped-step` —— 该 id 是既有组内的一个步骤（提升只适用于松散通道条目）；
3. `empty-title` —— 规范化后的标题为空（在 `decodeUnicode` + `flattenTitle` 之后 ——
   空组没有可寻址的任务供 `todo_add group:` 定位）；
4. `title-collision` —— 规范化后的标题与既有组冲突；把松散条目静默合并进那个事先存在的组
   不会增加任何步骤，因此第二个完全相同的请求会直接消失；改为拒绝，让调用方提供一个唯一的覆盖值。

仍然成立的结构性守卫：`WritePlan` 没有 `todos` 字段（`todo_write` 只调和 `groups`，
因此重新规划永不铸造或编辑松散条目）；一个在重新规划中到达 `done` 的松散条目会被带出松散通道
进入 `"Completed"`，使该通道永不保留已完成条目。

**空组是一等公民 —— 为提升而存在，而不是作为 `remove` 之后的落脚点。** `sanitizeGroup`
与 `write` 保留 `todos[]` 为空的组 —— 刚被提升的请求作为一个空组存在，直到 agent 分解它，
而在读取或写入时丢弃它会静默撤销 `promote`。`groupStatus` 对空组返回 `pending`；
skill 告诉 agent 立即添加第一个实质性步骤并把它翻为 `in_progress`，因此空组永远只是 `promote`
与首次 `add` 之间的一个瞬态。宿主 UI 把这个瞬态渲染为组标题下一条可见的 “planning…” 行
（`apps/web/src/panels/PlanPane.tsx` 与 `apps/web/src/chat/TodoList.tsx` 中的 `plan-group-planning` /
`todo-group-planning`），而不是一个空标题，因此读者永不会把该组看成半渲染的破损行。

`remove` 在不留下 `remove` 所生的幽灵的前提下维持这条仅限提升的不变量：它在删除之前对每组条目计数
做快照，并当且仅当某组的计数在本次调用中从 `>0` 变成 `0` 时剪掉该组。一个在 `remove` 之前
就已经为空的提升所生组被保留（搁置用户请求的不变量）；一个在 `remove` 中失去最后一个条目的组被剪掉，
因此用户永不会被困在一个永久的 “planning…” 行上、且没有工具可删除它。

**标题是单行的，在写入边界处规范化。** 条目标题与组标题既是 markdown 导出的条目项目符号与
`##` 组标题，而那份导出是一件已交付产物（计划页的导出与 PR 正文）。一个换行 —— 来自计划的多行添加
输入或某次 agent 转义 —— 会破坏那个列表，因此 `flattenTitle` 在每条写路径上把任何连续空白折叠成
单个空格（宽容处理，而非拒绝）：`add`、`update`、`replaceAll`（包括其调和键）以及加载时的
`sanitize`。调用方与读者可以假定标题永不含换行；没有逐次渲染的净化。

**匹配基于标题，这是设计当前的增量。** 由于键是 `(组标题, 条目标题)`，**组重命名**或**步骤重命名**
不被跟随：旧条目已完成的工作被保留（在旧名下带过来），其未完成的工作被丢弃，重命名后的条目以全新
面貌出现。这是标题匹配被接受的代价；`todo_write` schema 中可选的按条目 `id` 是计划中的下一个增量，
它能让匹配对重命名免疫。

**线性不变量**（结构性保持）：`update` 设置 `in_progress` 时会在同一次写入中把其他每个
`in_progress` 条目自动降回 `pending` 并返回它们（`TodoUpdateResult.paused`），使这次变化保持可见；
`replaceAll` 在其**合并后**的结果（新计划 + 被保留的 user/done 条目）之上重新建立它，按显示顺序 ——
只规范化新鲜的那一半，会留下一个 `in_progress` 的被保留用户条目与一个新鲜的 `in_progress` 步骤并存，
即同时有两个。`add` 接受 `after`（一个既有条目 id）以紧接该条目之后插入，并**继承其通道** ——
这是精准的计划中段插入（`after` 优先于 `group`；未知 id 抛错）。

## 公开接口（Public surface）

`index.ts` barrel：
- `TodoStore`（按 `(root, sessionId)` 构造，含 `setSummary` —— 计划级完成摘要）、`STORE_DIR` / `storeRel`，
  以及 `countItems(plan)` + `flatItems(plan)`（按显示顺序的每个条目：组在前，用户的松散通道最后 ——
  读取/更新/渲染复用的唯一一次扁平化）+ `groupStatus(group)` 辅助函数。
- 模型类型：`Todo`、`TodoGroup`、`TodoPlan`、`TodoFile`、`TodoInput`、`TodoPatch`、
  `TodoUpdateResult`、`WriteItem`、`WritePlan`、`TodoArtifact`，以及 `TodoStatus` / `TodoOrigin` /
  `TodoGroupStatus` / `TodoArtifactKind` 别名。
- `TODO_STATUSES`（`pending | in_progress | done`）、`TODO_ORIGINS`（`agent | user`）与
  `TODO_GROUP_STATUSES`（`pending | active | done`，仅推导）元组 —— 工具的 param 枚举的唯一来源。
  （**没有**优先级概念；优先级已被移除。）

写入是原子的（临时文件 + `rename`）；session id 在成为文件名之前被校验为安全的路径段。

**转义解码是一次狭窄的写入期补救，按字段类别划分，并且（在 Markdown 字段内部）按语法划分。**
`\uXXXX` 是无歧义的（没有 Windows 路径、正则或源码片段会字面写出 `\u`），因此它在每个 agent 撰写的
写入中、跨每个文本字段都被解码；一个 `(?<!\\)` 后行断言让 `\\u` 不受影响。`\n` 是**有歧义的** ——
LLM 可能把它作为 Markdown 换行替代物发出（`summary = "a\\n\\nb"`、`verification = "- a\\n- b"`），
也可能是一个合法的反斜杠前缀标识符（Windows 路径 `C:\\new file`、正则模式 `/\\n/g`、代码片段）。
即使在 Markdown 渲染字段内部，我们也无法在不变造路径的前提下补救每一个 `\n`，因此补救是*结构性的*：
只有当 `\n` 后跟一个 Markdown 分隔符时才被解码 —— 另一个 `\n`（段落分隔）、一个项目符号 / 有序列表 /
标题 / 引用 / 围栏块 / 主题分隔标记。任何其他 `\n`（短散文、路径、模式）逐字保留。`\r` 与 `\t`
不被补救（作为有意为之的 Markdown 分隔符太罕见，离 Windows 行尾与硬制表缩进太近）。
补救**只**适用于计划级 `summary`、按条目的 `summary` 与按条目的 `verification`（UI 与
`renderPrBody` 渲染为 Markdown 的三个字段）；其他每个 agent 撰写的字段（标题、组名、注记、提交主题、
产物标签）只运行 `decodeUnicode`。所有解码都以来源为门禁：用户撰写的文本永不被补救。
`\\n`（转义的反斜杠 + n）经后行断言被保持原样。读取是直通的：存储的文本逐字浮现，
因此磁盘上的旧数据在重新加载时永不会被静默改动。

## 边界（Boundary）

- **允许的依赖：** 仅 Node 内置模块（`node:fs`、`node:path`、`node:crypto`）。
- **禁止：** 任何 `@earendil-works/*` **以及任何 `@thinkrail/*`** 导入 —— 这是不含 pi、不含 thinkrail 的
  一层，宿主可以值导入它而不把 pi 拉进自己的 bundle，且它仍可在原生 `pi` 下安装。后果：`STORE_DIR`
  （`.thinkrail/context/todos`）携带 `@thinkrail/shared` 的 `WORKSPACE_CONTEXT_DIR` 的一个**本地镜像**，
  而不是导入它 —— 共享常量是宿主侧的事实来源；保持二者同步。`tools/` 通过 barrel 导入此处；
  此处不导入 `tools/`。
