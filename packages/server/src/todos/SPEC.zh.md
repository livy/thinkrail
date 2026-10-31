# todos —— 聊天会话级 TODO 计划（读写）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

为聊天会话提供聊天内 TODO 计划，并映射到线上 DTO。该列表**以 `sessionId` 为作用域**（每个会话一个
JSON 文件，位于工作区的工作树之内、临时上下文 scratch 目录
`.thinkrail/context/todos/<sessionId>.json`），而不是以工作树为作用域。按需读-改-写：每次调用都
经由 `pi-todos` 的无 pi 依赖 `TodoStore` 重新读取，因此 agent 在会话内的 `todo_*` 写入与用户的 UI
编辑收敛到同一个文件，没有过期窗口。`listTodos` 在输出时还会**为每个组装饰其推导出的 `status`**
（`pi-todos` 的 `groupStatus`）：这条规则属于拥有计划语义的包，把结果发出来则可避免 `apps/web`
——它只能导入 `contracts`——携带它的第二份副本。

与 agent 自己的工具（它们拥有状态）不同，宿主的写接口面是**用户**的编辑杠杆：`todo.add` 给新条目
打上 `origin: "user"` 标签，因此 agent 的 `todo_write` 重新规划永远不会丢弃它们。`todo.update` 存在
于线上（接受 status/title/note），但当前没有任何 UI 路径调用它——状态仍归 agent 所有（见
[[module-pi-todos]]）。`todo.remove` 是用户的队列取消操作，受 `removeTodo` 的 **done 守卫**约束：
pending / in_progress 条目可以被移除（UI 在它们上面显示悬停的 “Remove” 按钮）；`done` 条目会被
拒绝，因为已完成的步骤是计划的审计轨迹。宿主还叠加了一层进行中评审守卫
（`isItemUnderActiveReview`），使评审中途的取消不会让评审记账悬空。agent 自己的 `todo_remove` 工具
直接走 `TodoStore.remove`，不经过这道门禁。一次 UI 移除还会丢弃该条目的基线、评审记录和自动评审
周期计数。`updateTodo` 会解包 store 的 `TodoUpdateResult`（`{ todo, paused }`——`paused` = 为保持只
有一个 `in_progress` 而被自动降级的条目）；线上响应仍是一个裸 `TodoItem`——UI 在变更时重读整个
计划，因此降级随下一次 `todo.list` 到达。

本模块**不做**推送：用户编辑不会被广播到其他客户端。执行编辑的客户端乐观更新；第二个查看者在下次
由 `pi.event` 驱动的重新拉取时对齐。对单属主宿主而言这没问题（它服务的聊天计划 UX：
[[submodule-web-chat]] 的 “Chat TODO plan”）。

**变更工件（`artifacts.ts`）——以提交为基础的评审映射。** 状态仍归 agent 所有，但宿主*观察*状态
迁移以附上条目的代码变更，于是计划变成一份持久的评审映射。`host/server.ts` 从会话事件流上分接
`isTodoToolEnd`，并在发布路径之外触发 `maybeAttachChangeArtifacts(workspaceId, sessionId)`
（`void`——它会执行 git 写入）。对账**按工作区串行化**（一个 promise 链），因此两次快速的
`todo_*` 结束不会在提交过程中竞争索引。每次 `maybeAttachChangeArtifacts` 调用都先检查最新计划，
并在**入队之前**同步捕获任何新出现的 `in_progress` 窗口；当更早的一次对账已占据队列、或某个
done 条目先于活跃条目时，这一点依然成立。入队的那一趟在生产中永不打开缺失的窗口——缺失意味着
捕获失败，因此后续完成会降级到无基线的路径清单回退，而不是把工作排除在一个迟到的基线之外。整条
路径是尽力而为，永不向事件流抛出异常。同一条链串行化**每一个宿主计划/sidecar 写入者**——UI 的
`todo.add` / `todo.update` 与 `session.delete` 的窗口移除都排在任意进行中的对账之后
（`enqueueTodoMutation`）——一次对账会读取计划 + 基线、等待 git，并以整张映射的基线写入收尾；
落在这个窗口内的未入队写会被那一趟过期处理复活。agent 侧的 `todo_*` 计划写入无法入队（它们发生
在 pi 内部），因此对账会为自己读取的计划与持久化基线 sidecar 计算指纹，并在每次对应写入前重新校验
权威指纹。漂移会在过期的那一趟能够打开一个过晚的基线、提交一个被重新打开的条目、擦除一个外部
捕获/共享的窗口或覆盖更新的工件之前中止它；agent 工具已入队的后续对账则作用于新计划。每个成功
对账的条目都会在循环为下一个条目前去等待 Git 之前，先为其基线的添加/删除打检查点，因此后来的
漂移无法回滚更早条目已完成的窗口。

在 `in_progress` 时它**打开该条目的工作窗口**：一份工作树**未提交**变更路径集合的基线 + 当前
`HEAD` sha，在工具结束发布者返回之前经由 git 模块刻意保持同步的 `gitUncommittedPaths` 叶子节点
捕获，然后**持久化**在 todos JSON 旁边的宿主自有 sidecar 中
（`.thinkrail/context/todos/<sessionId>.baselines.json`，像 store 一样读-改-写）——因此条目中途
宿主重启不会改变任何东西；`head` 是**窗口内提交收编**（见下文）的范围基准。当**另一个聊天**已有
窗口时打开窗口会记录 `shared: true`，并把那个窗口也标记为共享
（`markOtherSessionWindowsShared`）——该标志是**粘性的**，因为“这个窗口在其整个生命周期内是否
独占？”正是门禁所需的，而一旦另一个窗口关闭就无法再推导。（*同一个*计划的两个条目不可能重叠：
`pi-todos` 恰好保持一个条目为 `in_progress`，被降级条目的窗口会被丢弃——由测试钉住，因为门禁
依赖这一点。）**窗口永不比其属主活得更久**：条目已从计划中消失的基线会在每次对账的开头被清除，
而 `session.delete` 会移除聊天的整个 sidecar（`removeSessionTodoWindows`）——一个孤儿会被读成一个
永久打开的外部窗口，并迫使每个兄弟聊天永远走回退路径。在 `done` 时：

- **提交该条目的增量。** `git.gitCommitPaths` 提交**恰好是增量路径**——条目自己的工作，绝不是
  “当前所有脏文件”——使用 `--no-verify`（该提交不得运行/失败用户的钩子；作者/提交者仍是用户
  自己的配置——这是他们的分支）。它在任何失败情况下都保留用户的索引（见
  [[submodule-server-git]]）。该条目得到**一个 `commit` 工件**（sha，`label` = 条目标题）**且别无
  其他**：该提交是自足的——它的文件清单是*推导*出来的，从不反规范化进 JSON（见下面的
  `listTodos` 装饰）。
- **收编窗口内的提交。** 宿主是条目窗口期间预期的唯一提交者（worker 子 agent 的 prompt 禁止它
  提交——[[module-pi-subagents]]），但那是指引而非强制，用户也可能手工提交。因此在增量提交之前，
  任何在条目打开期间落入 `base.head..HEAD`（`git.listCommitsSince`）且不被任何计划条目拥有的
  提交，都会作为 `commit` 工件**附到该条目上**——最老的在前，排在增量提交之前，于是该步骤的修订
  历史按时间顺序阅读。没有这一点，一个自行提交其工作的子 agent 会在 `done` 时把增量清空——该
  步骤将不显示任何变更集，而工作会作为孤儿泄漏到 `adoptedCommits`（见下文）。收编沿用与增量提交
  **相同的独占窗口门禁**（门禁 3：绝不 `shared`，没有其他聊天正在工作），需要有已记录的
  `base.head`，**并且要求该条目的窗口是本趟中唯一的同会话窗口**。唯一窗口规则是承重的：范围是
  `base.head..HEAD`，所以在一条线性分支上，*更早*条目的范围是*更晚*条目窗口的**超集**——如果一个
  done 条目在另一个同会话窗口打开时被对账（它自己的 done 趟被按工作区队列推迟，而下一个条目已
  开始且其子 agent 已提交），那么朴素的 `base.head..HEAD` 会让更早的条目贪婪地认领更晚条目的提交
  （`owned` 去重只阻止*同一个* sha 被认领两次，而不阻止这种跨窗口的越界）。因此当存在第二个同会话
  窗口时，该条目**什么都不收编**，那些提交降级到安全的 `adoptedCommits` 回退——即既有行为，绝不是
  错误归属。收编**不需要**门禁 2（外来的*未提交*脏改动），后者只管辖增量提交。整趟的 `owned` sha
  集合还会额外阻止一个已被任何条目拥有的提交（例如某次重做的先前提交）被再次收编。
- **提交消息是用户可以直接推送而无需改写的单行主题。** 这些提交落在用户自己的分支上，与他们手写
  的提交位于同一历史中，而该分支会直接送到 PR（[[submodule-server-pr]] 会推送它）——因此任何用户
  推送前不得不再措辞的内容都是缺陷。主题是条目由 agent 撰写的 **`commitSubject`**（见
  [[module-pi-todos]]：在 `done` 时按*宿主仓库的*提交风格撰写，由 agent 从 `git log` 读出），缺失时
  回退到条目 `title`。别无其他：**没有 `todo:` 前缀，也没有正文/trailer。** 早期版本写的是
  `todo: <title>` 加一个 `ThinkRail-Todo: <sessionId>/<todoId>` trailer；两者都已移除。前缀 + 一个
  计划步骤标题在 Conventional-Commits 历史中读起来像外来工件（本仓库自己的
  `todo: Get design sign-off, promote decisions into apps/website/SPEC.md` 就是标本），而那个 trailer
  是**只写**的——条目↔提交的归属靠 `commit` 工件的 `sha`，从没有任何东西把 trailer 解析回来，所以
  它只换来一个泄漏进公开历史的内部会话 UUID。`CommitWindow` 接缝相应地是 `{ subject, paths }`：选择
  主题是对账的职责，git 叶子节点只负责提交。
- **提交门禁（用户分支上的安全）。** 一次提交只能包含该条目可以被*证明*拥有的工作，因此以下四条
  必须全部成立——否则**不提交**，由实时 diff 的 `change` 路径清单工件顶替（分支作用域；`change`
  **只**作为这一回退存活）：
  1. **已记录的基线。** 没有基线 = 没有观察到的窗口（条目直接翻到 `done`、计划早于 sidecar）。
     对账没有任何诚实的依据去声称该步骤做了什么，因此 **`deltaPaths` 为空**：不提交、**没有
     `change` 工件**、什么都没有。脏路径改为出现在 `unattributed` 中，那正是“对账无法归属的工作”
     的正确位置——认领每一条脏路径会把兄弟步骤的工作和用户 WIP 抓进一个被捆绑完成的步骤。
  2. **没有残留的外来脏改动**——基线时脏的每一条路径到 `done` 时又重新干净。这正是让持有用户
     WIP 的默认工作区悄悄禁用自动提交的原因，也是预期的守卫。
  3. **窗口从未共享**（`shared` 未设置）且当前没有其他聊天在工作——并发窗口共享同一个工作树，它们
     的脏改动无法在彼此之间切分。
  4. **非空增量。**

  每个已提交的条目都会离开未提交集合，因此记忆化的变更路径读取在**每次提交后都被丢弃**——否则
  同一趟中对账的第二个条目会把第一个已提交的路径继承为自己的增量。
- **合并 + 重做时追加。** agent 的 `file`/`spec` 工件始终保留。一个已带变更集且**没有新基线**的
  `done` 条目是稳态空操作（幂等）；一个被重新打开、重新加工的条目（存在新基线）会把它的新
  `commit` **追加**到现有提交之后——工件清单就是该条目的**修订历史**（1 个 TODO = N 个提交是一等
  公民；每个修复周期多一个提交，下面的评审水位线对着这个清单做 diff）——而旧的 `change` 路径清单
  会被替换（实时增量没有历史可保留）。一次重做若其新附件包含**任何**路径清单 `change`，会**丢弃该
  条目的评审记录**（→ `unreviewed`）：实时路径增量无法用 sha 打水位线，而 `reviewInfo` 只从提交
  推导 `unreviewedShas`，所以一个与被收编提交并行的 `change` 对评审状态会不可见（一个被覆盖的 sha
  集合可能读作完全已评审，而路径增量却未被看到）——因此“只评审新增量”会诚实地降级为重新评审整个
  变更集。记录**仅当新附件完全由 SHA 支撑**时才保留（纯粹是窗口内提交收编、没有残留路径清单）：
  被收编的 sha 可打水位线，因此 `unreviewedShas` 增量会标记它们而不丢弃先前的裁决。**自动周期计数
  在这次丢弃中幸存**——它保存在一张独立的持久映射中（`reviews.ts` 的 `autoCycles`，以条目 id 为
  键，与 `items`/`pending` 同级），而非嵌入评审记录，因此为上述 sha 水位线重置而丢弃记录，永远
  不会悄悄重新授予一个已用尽的自动修复周期（曾有一个 bug 被修复：回退的 `dropReviewRecord` 过去会
  连同裁决一起抹掉 `autoCycles`，导致后来的手动评审把 `spent` 读作 0，并在上限之外又授予一次
  自动周期）。`approveTodoReview` 会清空它（一轮结算会为下一轮重置计数器）；
  `recordAgentChangesRequested` 独立于它同时写入的记录来写它；`todoReviewAutoCycles` 是读取侧。

宿主自己的磁盘状态（`WORKSPACE_INTERNAL_DIR` = `.thinkrail/…` 下的任何内容，例如 `context/todos/`
下的 todos JSON）会从每个变更集中被过滤掉——写入一个 TODO 会出现在 `git status` 中，但绝不是该步骤
*产生*的变更。无 pi 依赖的 `TodoStore` 从不接触 git；`commit`/`change` 是宿主专属，而 agent 通过
`todo_*` 工具自行附上 `file`/`spec`（见 [[module-pi-todos]]）。已知限制（已接受）：一个在条目中途
*自行*提交的 agent，由上面的窗口内提交收编处理（它的提交附到该步骤）**仅限独占的、唯一的窗口且
已记录 `base.head`**——共享窗口、缺失基线、未出生 HEAD 的基线，或同一趟中打开着第二个同会话窗口，
仍会把这些提交留作孤儿 `adoptedCommits`；而此机制看不见的写入者——用户在窗口中途通过终端或外部
编辑器编辑，或一个完全没有计划的聊天——在 `git status` 中与 agent 的工作无法区分，因此其未提交
编辑可能落入条目的增量提交，**其提交也可能被收编进该步骤**（应用自己的编辑器是只读的，而窗口
打开时已经脏的任何内容会被门禁 2 拦住）。

**`listTodos` 装饰——展开提交。** 线上 DTO 的 `commit` 工件携带一个推导出的 **`files`** 清单——
完整的 `GitFileChange[]` 行（路径 + 状态 + `+/−` 行数），在 **`commit:{sha}` 作用域**下通过
`git.gitStatus` 读取（正是 Changes 面板在那里渲染的相同行，一次推导）——在内存中**按工作区 + sha**
记忆化（可解析性是仓库本地的：两个克隆可以共用一个 sha，而只有一个仍持有该对象，所以一个工作区的
命中绝不能满另一个工作区的解析检查）——不可变，因此缓存永不做过期检查；只有成功的解析才被缓存，
一次瞬时的 git 失败（或 `UNKNOWN_COMMIT`）会在下一次 list 时重试。一个**无法解析的 sha**（在一次
历史重写后已被 GC——reflog 让被重写的提交存活约 90 天，远长于聊天计划的短暂生命；我们刻意不钉
任何东西）产出**没有 `files`**——这一缺失是客户端静默降级该能力（不显示 chip，绝不显示坏掉的
diff 标签页）的信号。同一个装饰趟也正是 `groupStatus` 已经在发送的地方，因此这个模式只有一个家。

**`listTodos` 装饰——未归属的剩余部分。** 同一趟还发送 **`TodoPlan.unattributed`**（仅非空时
出现）：工作树中**不属于本计划任何条目**的未提交 `gitStatus` 行——即未提交集合减去应用状态路径、
减去每个条目的 `change` 工件路径，并且在工作窗口打开时再减去打开条目的在途增量（其基线*之外*的
路径被推定该条目的工作；基线自身的路径恰是上面归属机制永远无法认领的既有脏改动）。这是计划的诚实
区：没有它，对账无法归属的工作——在第一个窗口打开之前、在最后一个条目结算之后所做的编辑，或在一个
从未做过计划的聊天中所做的工作——就从评审映射中完全消失，而那读起来像“没有其他变化”。每次读取时
推导（`unattributedChanges`，纯函数——与归属规则同一个家），从不存储；git 失败降级为省略该字段。
并发聊天在同一工作树中的在途工作也会出现在这里——它确实在本计划之外——这是已接受的噪声，与上面的
共享窗口限制同属一族。

**`listTodos` 装饰——被收编的提交（已提交的剩余部分）。** `unattributed` 只覆盖*未提交*行；工作
一旦被提交就离开未提交集合，并会从评审映射中完全消失（一个空计划聊天的 agent 通过 bash 提交、用户
的手工提交、宿主为某个后来被清空的计划所做的 todo 提交）。因此同一趟还发送
**`TodoPlan.adoptedCommits`**（仅非空时出现）：每个 `base..HEAD` 提交（`git.listCommits`）中 sha
不被**任何**条目的 `commit` 工件拥有的那些，以**仅线上存在的 `done` `TodoItem`** 呈现——
`id: "commit:<sha>"`、`origin: "adopted"`、`title` = 提交主题、一个用同样的按 sha `files` 清单装饰
的 `commit` 工件，以及从 sidecar 以该 id 为键读取的 `review` 装饰。这些条目**从不写入 store**——
agent 的计划 JSON 保持是 agent 的计划（那个不变量），而 `"adopted"` 永不进入 `pi-todos`；它们每次
读取时重新计算。生命周期是免费的：`commit:<sha>` 稳定，被 rebase 掉的提交的 id 直接消失（它的
sidecar 记录变得惰性，像任何孤儿一样），而一个后来被真实条目认领的松散提交会自动退出（现在已被
拥有）。一个被*另一个*会话的计划条目拥有的提交也会出现在这里——已接受的噪声，与共享窗口限制同属
一族。尽力推导；git 失败则省略该字段。

**被收编的提交在没有 store 条目的情况下也可评审。** 评审操作通过 `reviewableItem` 解析其目标，它
先读 `TodoStore`，未命中时为被收编的 `commit:<sha>` id 重建一个合成的 `StoredItem`——但**只有在校验
了推导所产出的确切集合之后**：`git.resolveListedCommit` 把该 sha 解析为其**规范 OID** 并确认它属于
**`listCommits` 所产出的同一份有上限的 `base..HEAD` 清单**（共享 `COMMIT_LIST_MAX`——因此一个超出
最新 200 条、永远不会产出收编条目的提交同样不可评审），请求的 id 必须等于
`commit:<canonical-oid>`（正是 `listTodos` 产出的形式——缩写或非规范 id 会被拒绝，因此评审状态
永远不会写在一个任何收编条目都不会携带的 id 下），**并且**该规范 sha 不能被任何计划条目拥有。
一个已掉出该集合的 id——被 rebase 进基线、被 GC、被缩写、超出上限，或此后被某个步骤认领——会被
拒绝（`No TODO with id`），因此一个过期的 Plan 操作永远不会对某个下次重新加载就消失的提交
启动/批准/修复评审状态。因此每个操作（`startTodoReview`、`approveTodoReview`、`cancelTodoReview`、
`requestTodoFix`、`recordAgentChangesRequested`、`renderReviewPackage`）都能对一个被收编的提交驱动
Start-review / Review All / 裁决，且**零 store 写入**；评审状态持久化在既有 sidecar 中，以
`commit:<sha>` 为键，而每个 finding 都携带其来源计划会话 + 条目 id（`{ todoId, sessionId }`）——
评审人是该计划会话的隐藏委派子级，所以路由不需要评审人会话身份，无论是否存在已存储的条目都同样
有效。修复包的“重新打开这个确切条目”指令对一个被收编的提交没有可重新打开的 todo，因此它读作
“修订提交 `<sha>` 中的变更”；worker 的后续提交以一个新的收编条目（或一次修订，一旦被追加）出现。

**被收编提交的限制（已接受）。** (1) `base..HEAD` 的枚举受 `git` 的 `COMMIT_LIST_MAX`（200）限制
——提交更多的分支会从 `adoptedCommits` 中丢掉最老的，这与 Changes/commits 菜单所受的上限相同。
(2) 提交不可变，所以自动修复周期无法自我修复一个提交：agent 对一个被收编提交给出
`changes_requested` 裁决会让它永久处于被标记状态，而 worker 的修复会作为*新的*收编条目落地，而不是
对原提交的一次修订（并且 `maybeAutoReReview` 有意排除 `adoptedCommits`——新 sha 就是新 id，被标记
的那个上没有可重新评审的新增量）。(3) 被*另一个*会话的计划条目拥有的提交在这里显示为被收编
（`owned` 集合只是本会话的计划），而每个会话可以为同一 sha 持有自己的评审记录——已接受的噪声，与
共享窗口限制同属一族。(4) 一个在被收编提交中提交、随后又被编辑为未提交的路径会同时出现在
`adoptedCommits`（提交的文件）和 `unattributed`（更新的未提交行）中——它们确实是两个不同的工件。

**评审工作流（`reviews.ts` + `todos.ts` 中的操作）。** 一个带有宿主变更集的已完成条目是**可评审
的**——门禁就是该工件的存在，因此研究/验证步骤永不要求评审，也不涉及任何 LLM 归属。用户的决定住在
第二个宿主自有 sidecar 中，`.thinkrail/context/todos/<sessionId>.reviews.json`（读-改-写、原子、
与基线相同的生命周期：`session.delete` 移除该文件；孤儿记录是惰性的）——刻意**不是** agent 可写的
todos JSON：agent 的重新规划绝不能翻转一个评审决定。一条记录是 `reviewed` 或
`changes_requested` 加上 **`reviewedShas`，即水位线**：评审人实际作用过的 sha 集合。对一个人工
操作，那就是该时刻条目的提交 sha；对 agent 流程，待定标记携带真相——**`startTodoReview` 会把条目
起始时刻的 sha 清单盖章进在途的 `pending` 条目**（`{ at, shas }`，持久化在 sidecar 中），而
`approveTodoReview` / `recordAgentChangesRequested` 优先采用它们而非当前 sha，因此 worker 在评审人
轮次流式传输期间落地的提交仍是一个未评审增量，而不会被悄悄打上已读水位线。`unreviewed` 是记录的
缺失。同一个 `listTodos` 装饰趟会发送 `TodoItem.review`（状态、`revision` = 提交数、
`unreviewedShas` = 水位线之后追加的提交——即 UI 重新评审而非重看原始 diff 的“自评审以来已变化”
增量——以及 `feedback` 回显）和 `TodoPlan.summary`（计划级完成说明，由 agent 经
`todo_plan_summary` 撰写；条目 `summary` 按存储内容随条目 DTO 一起传递）。

- **`generateTodoSummary`（`todo.generateSummary`）** 是宿主对该说明的尽力回退：当一个计划**完全
  done 但不带 `summary`** 时，宿主起草一份。它原样返回已有的 agent 说明，在计划未完成或起草失败时
  返回 `null`，否则返回新起草的说明。慢模型调用（`assist.suggestPlanSummary`，基于已完成步骤的
  title/summary/verification）在写锁**之外**运行；最终的重新检查 + `setSummary` 在
  `enqueueTodoMutation` 内运行，且永不覆盖期间落地的说明或一个被重新打开的计划，每个会话同时只有
  一次在途生成——并发调用者会等待并共享那一代的结果，而不是拿到 `null`，因此没有界面被留在无摘要
  状态。它永不覆盖 agent 自己的 `todo_plan_summary`。**`previewPr`（因而 `openPr`）在
  `renderPrBody` 读取 `plan.summary` 之前等待 `generateTodoSummary`**，因此一个 UI 锚点被拆除的
  聊天（聊天被关闭到历史、Plan 页面被关闭）不会与 `pr.preview` / `pr.open` 竞争出一个空摘要。
  在途去重让这次额外调用保持幂等：如果 UI 的 `useAutoPlanSummary` 已触发一次起草，`previewPr`
  等待它的 promise，而不是开始第二次生成。

- **`approveTodoReview`** 记录 `reviewed` + 水位线——当 agent 评审在途时用待定标记的起始时刻 sha，
  否则用当前 sha（对未知或不可评审的 id 抛出 → 线上 `{ ok:false }`）。
- **`requestTodoFix`** 记录 `changes_requested` + 反馈 + 水位线，并渲染**修复包**
  （`renderFixPackage`，纯函数）：原始步骤（title/note）、其完成摘要、变更集*引用*（短 sha / 路径
  ——绝不是完整 diff；agent 用自己的工具读取内容）、逐字反馈，以及重新打开**这个确切条目**的指令
  ——修订必须附到它所修订的步骤上（todos skill 从 agent 一侧镜像这一点）。**发送在 `host` 中组装**
  （本模块从不导入 `agent`）：宿主把它作为结构化的 `todo-review-fix` 自定义消息
  （`sendReviewFixToSession`）投递进该条目**自己的聊天**（逐会话计划/窗口强制如此）——渲染出的包
  就是 agent 读到的消息 `content`，而 `ReviewFixDetails`（`buildReviewFixDetails`，从 `reviews/`
  导出）作为聊天气泡卡的 `details` 随行；以评审发送模式分离触发——轮次前拒绝会调用
  **`rollbackTodoFix`**（恢复该请求替换掉的记录）并出现在聊天中，因此一次未送达的修复请求永远不会
  搁浅为 `changes_requested`。手动请求携带一个不透明的 `requestId`；补偿只在该确切请求仍是当前
  请求时才恢复之前记录，因此一次延迟的发送失败无法抹掉更新的裁决或重试。

**agent 评审人（[[submodule-server-reviews]] 是 finding 的家）。** `todo.startReview` 和 worker
自己的 `request_review` 工具把一个可评审条目摆在一个**隐藏的、短暂的评审子 agent**面前——它是计划
会话的一个委派子级，携带宿主的评审人角色（[[submodule-server-host]] 拥有 prompt + 输出契约）。没有
评审人聊天，也没有被钉住的评审人会话：这里不保存 `reviewerSessionId`，评审人也不持有本模块的任何
工具。本模块拥有状态 + 包：`startTodoReview`（标记条目的在途 `pending` 标记——即 DTO 的
`reviewing`——并渲染 `renderReviewPackage`：一份变更集**引用**加上 worker 的摘要/验证声明供其核实，
一次只命名未评审增量的重新评审；它不命名任何工具，因为评审人的角色和输出契约属于宿主，不属于这个
包）、`cancelTodoReview`（评审失败或返回无可解析裁决）、`approveTodoReview(…, "agent")`（标注
`reviewedBy`）、以及 `recordAgentChangesRequested`（裁决说明作为反馈 + `autoCycles`）。后两者一次
性变更三样东西——条目的记录、它的 `autoCycles` 计数、以及它的 `pending` 清除——因此它们把这三者
作为**一次快照写入**持久化（`commitReviewTransition`），绝不是一串写入：否则两次写入之间的部分失败
可能让一个条目搁浅在“已用尽周期但 pending 标记仍设置”的 `changes_requested` 状态，而没有任何取消
能撤销它。自动周期机制——宿主的 **1 次自动周期上限**：周期 0 的裁决自动把评审人的 finding 发给
worker（autoCycles 1），修复后的修订自动再次评审一次（触发条件是 autoCycles === 1 + 一个新增量——
一个在水位线之后追加的 sha，或者状态读作 `unreviewed`（因为路径清单回退把它重置了，那本身就是
路径清单条目没有 sha 可携带的增量信号）——见上面回退的 autoCycles 持久性），而那次裁决记录
autoCycles 2——终结，由人来决定；当 `reviewAutoFix` 设置为关闭时该上限被**短路**——
`host/requestReview` 那时把裁决终结性地记录（autoCycles 2）且不发送，因此 finding 只是等待人来
处理）。评审人的 finding 是 reviews 模块中**由 agent 撰写的评审评论**（`author: "agent"`），绝不是
一份并行 store；编排/发送住在 `host/requestReview.ts` 中。**宿主重启安全：** 在途记账只在内存中，
因此宿主上次停止时仍在运行的评审所留下的 `pending` 标记否则永远不会清除——
`clearAllPendingReviews(root)` 清扫一个工作区下每个会话的 sidecar 并丢弃每一个 `pending` 条目，
无条件；`host/todoReview` 的 `reconcilePendingReviewsOnBoot` 在启动时对每个工作区调用它一次，在
任何客户端能观察到过期 spinner 之前（见 host/SPEC.md）。只清空 spinner——底层评审记录（若有）不受
影响。**Review All**（`todo.reviewAll`）是在同一流程之上的纯宿主编排：它在计划的串行链上启动每个
未结算的可评审条目（`host/planReviewQueue.ts`），因此它在这里不增加状态（见 host/SPEC.md）。

**读屏障。** `listTodos` 先等待工作区在途的对账（`settleChangeArtifacts`——同一个按工作区链）。
客户端唯一的刷新信号是 `todo_*` 工具结束所发布的 `pi.event`，而对账是*与该次发布同步*入队的，但
稍后才结算（它会提交）——因此没有屏障时，一次慢于客户端重新拉取防抖的提交会交回一个没有变更集的
`done` 条目，让一个打开的 plan 页面承诺一个它并不显示的入口，直到某个无关事件到来。等待使读取
**因果地晚于**触发它的写入；没有在途任务时它立即解析，且从不拒绝。屏障会跟随等待期间安装的更新
的工作区队列尾部，因此被计划漂移中止的过期趟无法在已入队的替代趟对账完当前计划之前释放读者。

## 边界（Boundary）

- **拥有 / 公开接口（barrel）：** `listTodos({workspaceId, sessionId}) → Promise<TodoPlan>`（异步
  仅因上面的读屏障）、`countOpenTodos({workspaceId, sessionId}) → number` 及其纯规则
  `openTodoCount(plan)`（未完成 = 除 `done` 之外的任何状态，松散 + 分组——宿主 `session.list`
  处理器为客户端历史/状态呈现附上的 `SessionSummary.openTodos` 装饰；没有 todo 文件的会话计数为
  0）、`addTodo(...) → TodoItem`（校验非空标题；打 `origin: "user"` 标签）、
  `updateTodo(...) → TodoItem`（对未知 id 抛出 → 一个 `{ ok:false }` 的 WS 响应）、
  `approveTodoReview(...)` / `requestTodoFix(...) → { pkg, previous }` / `rollbackTodoFix(...)`
  + 纯 `renderFixPackage`（评审操作；发送本身是 `host` 的组装），以及 `TodoReviewRecord` 类型。
  **仅映射**——没有计划逻辑；`TodoStore` 拥有磁盘。
- **允许的依赖：** `workspaces`（经 `getWorkspace` 查工作树路径，对未知抛出）；`git`
  （`gitStatus`——未提交的变更路径集合 + 提交作用域 DTO 装饰；`gitCommitPaths`——每个 done 条目的
  增量提交；`gitHeadSha`——基线的 head；`listCommits`——`adoptedCommits` 背后的 `base..HEAD` 枚举
  + 评审解析器的合成条目重建）；`contracts`（DTO + `isTodoToolEnd` 用的 `PiEvent`）；
  `@thinkrail/shared/paths`（`WORKSPACE_INTERNAL_DIR`——从变更集中过滤掉的应用状态前缀）；
  **`pi-todos/core`**（无 pi 依赖的读/写模型——对扩展包的一种受认可的宿主侧值导入，与 `spec` →
  `pi-spec-graph/core` 相同的模式）；`log`。
- **禁止：** `host`；除 `workspaces` + `git` + `log` 之外的兄弟特性；`pi-todos` 的扩展入口或
  `tools/`（与 pi 耦合）；任何 pi 包。
