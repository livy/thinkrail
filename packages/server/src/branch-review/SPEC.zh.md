# branch-review —— 打开的 PR/MR 元数据

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

尽力查找与一个工作区分支关联的代码评审 —— 打开的、已合并的或已关闭的 —— GitHub.com 的 PR 经本地
`gh` CLI，GitLab.com 的 MR 经 `glab`。该查找**先问打开的评审**（`gh pr list --state open --limit 1`、
`glab mr list --per-page 1`；一次调用，与已结算状态存在之前相同），**只当没有时**才分页取最新的已合并/
已关闭行（`--state all` / `--all`，五行，最近合并或关闭的胜出）。两次有界调用而不是一个合并页，因为
合并页按创建时间排序：一个带有更旧的仍打开评审和若干更新的已结算评审的分支，会只呈现已结算的那些，
而一个信任该答案的宿主会把一个 PR 仍打开的工作区搁置。答案在 `OpenBranchReview` 上携带 `state` 与
`changedAt`（合并或关闭时间）：一次查找就服务 `workspace.branchStatus` / `openReview`、顶栏徽标
（打开 = success，已合并 = info，已关闭 = neutral）以及 Settled 置物架的「PR 已合并/已关闭」规则，
而不是两次轮询互相矛盾。已合并/已关闭行只有在带上其匹配的合并/关闭时间戳时才可靠；没有那个排序事实，
宿主无法证明之后的工作是否应保持在线，因此它拒绝该答案，而不是持久化不完整的置物架权威。一个只理解
打开评审的 pre-v78 消费者会把已合并/已关闭行读作在线 PR，这就是为什么 contracts 规格把 `state` 钉到
`WORKSPACE_SETTLE_PROTOCOL_VERSION`，而宿主对这类客户端的 `workspace.openReview` 为已合并或已关闭的
评审回答 `null`。

## 边界（Boundary）

- **拥有：** 远端宿主探测与有界、异步的 CLI 查找，返回一个 `OpenBranchReview` 或 `null`，加上对成功
  查找答案的短期记忆。
- **远端宿主探测也是异步的。** 它的 `git config` / `git remote` 读取经 `gitAsync` 在
  `LOOKUP_TIMEOUT_MS` 下运行，永不同步 spawn：宿主的已结算生命周期趟每五分钟为每个候选行运行这次
  查找，而同步读取会把整个宿主（每个聊天、agent 与终端）每行阻塞约 45 ms。
- **该查找向 provider 一并索取评审的 `url` 与其编号**（`gh … --json number,url`；GitLab 的行携带
  `web_url`）并把它放到 `OpenBranchReview` 上。它不是装饰：客户端的 `PR #N` 徽标只有在 url 已知时才
  是链接，而在此之前 url 只存在于刚刚运行过 `pr.open` 的那个会话中 —— 每次重载、第二个窗口与重连
  都把编号渲染成死文本。url 缺失或非 `https:` 的行产生一个无 url 的评审，而不是一个坏链接。
- **一个打开 GitHub PR 的评审事实搭同一次打开行调用**（`isDraft, reviewDecision, mergeStateStatus,
  statusCheckRollup, latestReviews` 一起进入 `--json` 列表 —— 每次读取不做第二次调用）并成为
  `OpenBranchReview.details` —— 只在 PR 的 head 仓库（`headRepositoryOwner` / `headRepository`，也一并
  请求）是 origin 自己的时候：`--head` 仅按分支名匹配，因此一个 fork 的、来自同名分支的 PR 也会被找到，
  而它永不能看起来像是本工作区可以动手的东西。已结算页也索取 head 仓库，并在挑选之前丢弃已知的外来行，
  因此一个 fork 的已合并或已关闭同名 PR 永不会搁置本工作区，也不会提供 Reopen。检查按每次检查的最新
  尝试各计一次（一次重跑在 rollup 里按尝试出现）：SUCCESS / NEUTRAL / SKIPPED 通过；FAILURE /
  TIMED_OUT / CANCELLED / ACTION_REQUIRED / STARTUP_FAILURE 失败；任何未完成的（以及 PENDING /
  EXPECTED 状态上下文）是待定；STALE 什么也不算，因为一个永不完成的检查会让客户端的 CI 轮询永远活着。
  只有携带详情字段的行才得到详情，因此 GitLab 与已结算行保持它们的形状。仓库的合并方法与
  `deleteBranchOnMerge` 来自 `gh repo view`，只在存在这样一个 PR 时才问，并按 origin URL 缓存 10 分钟
  （仅成功）；一次失败的读取会让它们缺席，而不让查找失败。
- **公开接口：** `findBranchReviewOutcome(cwd, branch, { fresh? })` → `{ value, reliable }` ——
  `reliable` 是被暴露出的可缓存位，因此一个*持久化*该答案的宿主能区分「provider 说没有评审」与「
  provider 没回答」，并在一次 `gh` 故障期间保住它最后已知的快照；`forgetOpenBranchReview(cwd)`；
  加上 `pr` 动作模块复用的读取原语 —— `providerFromRemoteUrl`、`githubSlug`（GitHub 远端 URL 的
  owner/repo，与 `pr` 共享）、`reviewNumber`，以及 `runProviderCommand`（有界、禁用提示的 CLI runner；
  它把 stderr 作为 `err` 返回，好让一次被拒的变更说明原因）。
- **成功答案按 `(worktree, branch)` 缓存，从结算起 60 秒，且查找是单飞的。** 一个语法上有效的空
  provider 响应是一次成功的 `null` 并被缓存 —— 「没有 PR」是常见情形，也是重新推导代价高的那种。
  一次 provider-CLI 失败、失败的强制本地远端检查、抛错的 runner 或畸形响应仍对调用方降级为 `null`，
  但不被保留，因此修好 CLI 认证或一次瞬时故障可以在下次读取时恢复。过期条目在缓存活动时惰性清理。
- 普通工作区激活可以复用一次已结算的答案；`{ fresh: true }` 绕过一次但仍加入一个已在运行的查找。web
  在窗口聚焦时使用新鲜路径，从而把聚焦保留为在 ThinkRail 之外打开、关闭或合并的评审的显式重新校验点。
- `forgetOpenBranchReview(cwd)` 使该工作树的每个分支代次失效。一次在途时被取代的查找必须经当前代次
  结算，而不是返回或重新缓存它陈旧的答案；这正是让失效对并发读取安全的东西。
- **允许的依赖：** `contracts` 用于结果类型；server 的 `git` barrel 用于本地远端检查与
  `nonInteractiveGitEnv()`（子进程运行环境的唯一定义 —— `process.env` 加 `GIT_TERMINAL_PROMPT=0`，
  本模块在其上叠加自己的 `GH_PROMPT_DISABLED`/`GLAB_PROMPT_DISABLED`）；`subprocess` barrel，它在本
  模块的 `LOOKUP_TIMEOUT_MS` 下运行查找。
- **禁止：** `host`、`workspaces`、浏览器代码、持久化，或超出本次读取的任何 PR/评审动作（动作住在
  `pr` 里）。
- 缺失 CLI/认证、不支持的远端、超时、畸形输出以及没有打开的评审都降级为 `null`。
- **界限必须是*这次调用的*，不是子进程的。** 杀掉 `gh`/`glab` 然后 await 它的 stdout 到 EOF，在一个
  孙进程继承了那条管道时永不返回，而 `review.get` 会 await 这次查找 —— 因此上面承诺的降级为 `null`
  只能通过客户端自己无因的请求超时到达。`subprocess` 的 `runBounded` 才是让 `LOOKUP_TIMEOUT_MS` 真实
  的东西；本模块永不得长出第二个自己的 spawn。
