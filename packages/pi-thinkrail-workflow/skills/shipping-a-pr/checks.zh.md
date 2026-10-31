# checks.md —— 观测，然后要么等待，要么报告

> 中文译文，原文见 [`checks.md`](./checks.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

入口：一个已打开的 PR，外加调用方选定的完成模式。不保存任何内容。本文档以下文两种终态之一结束
该工作流。

## 快照模式 —— 默认的交付与报告（Snapshot mode — default delivery and reporting）

在普通的 PR 创建、同步、截图、标题/正文维护、评审意见、任何常规的 head 更新，或一次性的
检查/合并状态请求之后使用。一次推送并不意味着监视；只要用户没有明确请求远端完成，就用这个模式。

1. 运行一次 `gh pr checks <n>`。空的汇总只意味着“当前没有报告任何检查”；CI 可能还没有为新的 head
   注册，因此把它的存在报告为不确定。把 pending 或失败的检查按观测到的原样报告。
2. 一次性一起查询 `mergeStateStatus`、`isDraft` 和 `reviewDecision`。每个值都按观测到的原样报告；
   不要轮询、同步、重跑或修复它。
3. 给用户 PR 链接、已完成的任何元数据动作，以及当前的检查 + 合并状态快照。

**快照终态：** 被请求的 PR 动作已完成，且当前 PR 状态已报告。这不是在声称 PR 已全绿或已合并就绪。

## 等待模式 —— 明确的远端完成（Wait mode — explicit remote completion）

### 观察（Watch）

- `gh pr checks <n> --watch`（或针对单次运行用 `gh run watch <run-id> --exit-status`）。当观察不可行时，
  带间隔地轮询 `gh pr checks <n>`。
- 如果没有报告任何检查，短暂等待后重试，直到检查注册，或独立地确认该仓库对这个 PR 没有 CI。
  绝不从一次空的汇总推断“没有配置检查”；只有在确认之后才把它写进终态摘要。
- 失败时，查看 `gh run view --job <job-id> --log-failed`；当日志不足以下结论时，在本地复现。

### 反应（React）

- 修复由该分支造成的失败，提交、推送，并重启这个循环。看起来像是偶然波动的失败要去排查，
  而不是重跑到它通过为止；`gh run rerun --failed` 只在该失败可证明与分支无关时使用一次。
- 如果全绿需要用户决策或请求之外的工作，报告这个阻塞项并停下，而不是扩大范围。

### 验证合并就绪（Verify merge readiness）

检查解决之后，一次性查询完整状态：

`gh pr view <n> --json mergeStateStatus,isDraft,reviewDecision`

显式处理每一个状态：

- `isDraft: true` —— 如果用户要求把 PR 标记为就绪，运行 `gh pr ready` 并重新查询。否则报告它仍是
  草稿；用户请求创建的草稿可以在这个预期状态下结束，但绝不称作合并就绪。
- `UNKNOWN` —— 短暂等待并重新查询，直到 GitHub 计算出该状态。
- `BEHIND` 或 `DIRTY` —— 阅读并遵循 `syncing.md`，然后在等待模式下回到这里。
- `UNSTABLE` —— 提交状态并非完全通过。重新读取当前检查并处理由分支导致的失败；否则报告这个
  范围之外的阻塞项。绝不把它描述成全绿。
- `BLOCKED` —— 报告 `reviewDecision`，不要宣告成功。当处理评审反馈在范围内时，`CHANGES_REQUESTED`
  路由到 `review-comments.md`；`REVIEW_REQUIRED` 需要人工评审；其他任何值意味着还有另一条分支保护
  规则仍在阻塞该 PR。
- `CLEAN` —— 肯定：可合并且提交状态通过。
- `HAS_HOOKS` —— 肯定，但带一条明确的保留意见：检查通过，GitHub 也认为该 PR 可合并，但尝试合并时
  pre-receive hooks 仍会运行。

**等待终态：** 当用户明确请求了远端完成时，PR 存在、非草稿、与其基线一致、每项已配置的检查都绿
（或明确没有 CI），且 `mergeStateStatus` 为 `CLEAN`，或为 `HAS_HOOKS` 并已报告其保留意见。
用户会得到链接，以及交付了什么、验证了什么，以及任何有意排除的内容。用户请求的草稿，或需要人工/
范围之外决策的阻塞项，是明确的替代终态，绝不是合并就绪的成功。
