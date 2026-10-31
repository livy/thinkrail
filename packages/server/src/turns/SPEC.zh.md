# turns —— 每次 agent 运行产生的变更集

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

在不信任工具参数的前提下回答「这次 agent 运行改了什么？」。在 `agent_start` 时跟踪器把工作树快照进
一个 git **tree 对象**（`snapshotWorktree`：真实索引被复制到一个临时 `GIT_INDEX_FILE`，`add -A` 对着
副本运行，`write-tree` 返回 oid —— 已跟踪的编辑与未跟踪文件都包含，忽略文件与用户真实索引不受触动）；
在 `agent_settled` 时它再次快照，当两个 tree 不同时就记录一个 `TurnChangeSet` —— 两个 tree oid、该次
运行的墙钟时间，以及二者之间 `turn` 作用域上的 `gitStatus` —— 持久化它，并推送 `turn.changed`。运行
边界是 `agent_start`→`agent_settled`，即 pi 的 round（一个 prompt 及其引发的一切），而不是 pi 的
`turn_*`（一条 assistant 消息）：每条消息一份回执会把一个答案碎成许多卡片。

## 边界（Boundary）

- **拥有：** `TurnTracker`（逐会话的挂起运行 → 结算，加上用于在途记录的 `drain()`）、数据目录下的
  `turns.json`（`persistence.loadTurns` / `saveTurns`，带版本、原子、每工作区上限 30 次运行、随工作区
  一起丢弃）、`listTurns(workspaceId)`、`turnScope(turn)`、`turn.changed` 发布接缝。
- **公开接口（barrel）：** `TurnTracker`、`listTurns`、`forgetWorkspaceTurns`、`setTurnPublisher`、
  `turnScope`。
- **消费方：** `host` 把跟踪器接进会话发布者，紧邻运行分析，并服务 `workspace.turns`；`git` 把 `turn`
  作用域解析为 `diff <baseTree> <headTree>`，两侧都不可变（无未跟踪趟，`modifiedRef` 已设置 —— 因此
  `changes` 拒绝变更它，完全像一个 commit 作用域）。

## 决策（Decisions）

- **快照，不是工具参数。** 聊天的轮次分隔符已经列出 `write`/`edit` 参数中的路径；那会漏掉 shell 写入、
  格式化器与生成器，也对重命名或行数一无所知。tree 快照才是 agent 实际留下的东西。
- **tree，不是 commit。** 一次快照不得触碰 HEAD、索引或 reflog；悬空的 tree 只耗费对象存储，`git gc`
  会在其宽限期之后修剪它们。代价是一个非常旧的轮次可能停止解析 —— 此时该作用域以 `UNKNOWN_COMMIT`
  失败：Changes 侧边栏重置到 *All changes*，与一个被改写的提交完全一样，而该轮次的一个打开评审标签页
  会说明这点并提供 *All changes*。
- **一份回执是该工作区的区间，不是该次运行的著作权。** 两次快照都覆盖整个工作树，因此在该运行挂起
  期间落地的第二个聊天、一次用户编辑或一次回退，也会出现在它的回执里。排他归属需要宿主并不拥有的
  逐写入出处；工具参数恰恰是本模块所替换掉的那个不可靠来源。
- **零变更的运行不记录任何东西。** 一份写着「0 个文件」的回执是噪音，而 *Last turn* 意味着最后一次
  改动了东西的运行。
- **一次运行的第一个 `agent_start` 胜出。** pi 在重试时会重新发出 `agent_start`；一个挂起运行被保留
  到它结算，因此基快照是 agent 首次写入之前的状态。
- **两次快照都从各自的边界开始。** 基快照从 `agent_start` 开始、head 快照从 `agent_settled` 开始，
  各自同步地在事件处理器内；随后结算会 await 两者。若为等一个慢的基快照而不开始 head，下一次运行的
  首次写入就会漏进本次运行的回执。剩下的是被接受的竞速：快照是异步的 git 工作，而一次在它运行期间
  落地的 agent 写入可能落在任一侧。模型延迟使这在第一个工具调用处实际上不可能；而阻塞式快照会延迟
  大仓库上每一次运行的开始。
- **回执属于被捕获的工作区，而不是在线会话。** 在 `agent_start` 时捕获工作区 id 与工作树路径；一个在
  结算期间被分离的会话不得丢失它的回执。在记录/发布之前独立地立即检查工作区注册表，因此工作区移除仍
  无法在 `turns.json` 中重建它的条目（`forgetWorkspaceTurns` 只删除既有记录）。`workspace.turns` 同样
  拒绝未知工作区。
- **优雅关停在结算会话之后、销毁之前排干记录。** `drain()` await 在 `agent_settled` 时开始的任务；
  它永不为一挂起的 start 发明一次结算。它像之前的会话结算一样有界（5 s）：一次卡在慢文件系统上的 git
  只耗费该次运行的回执，永不会变成一次挂几分钟的退出。同步的 `stop()` 仍是应急路径，不是记录屏障。
- **启动顺序就是顺序。** 轮次按 `startedAt` 存储与限额，而不是按结算时间，因此一次更晚开始却先结算的
  运行不会成为「Last turn」—— 客户端对在线 `turn.changed` 应用同样的规则，因此重载永不会与推送矛盾。
- 宿主在连接打开时把每个 socket 都订阅到 `turn.changed`，与其他工作区通道一样；一份只能经重载时的
  `workspace.turns` 才到达客户端的回执，正是这行所防范的缺陷。
