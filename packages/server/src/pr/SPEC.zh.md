# pr —— 推送工作区分支、打开/更新其 GitHub PR，并在其上执行动作

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

「计划完成 → PR 打开」（`pr.open`）的确定性宿主侧：把工作区分支推送到 `origin`，并打开 —— 或更新
—— 该分支的 GitHub PR，PR 正文由会话的 TODO 计划渲染（经校验计划的叙述：摘要、带 sha + 验证的步骤、
评审轨迹）。正文还为 `TodoPlan.adoptedCommits`（不属于任何步骤的分支提交 —— 见
[[submodule-server-todos]]）携带一个 **`## Committed outside the plan`** 章节，而评审轨迹横跨计划步骤
**与**被采纳的提交，因此一个工作从未进入计划的分支仍能交出一份非空、诚实的描述，而不是空白正文。
该按钮永不经过 agent（一个由 prompt 支撑的 PR 按钮很脆弱 —— 见 task-open-pr）；未来的 pi 扩展把同一
库暴露给 agent。

## 行为（Behavior）

`previewPr({ workspaceId, sessionId?, title?, source?, template? })` → `{ title, body, templates,
planAvailable, source }`（`pr.preview`）：`openPr` 将会使用的确切标题/正文，好让客户端的撰写对话框在
任何东西被推送之前把它们提供出来供编辑。`source` 选择正文：**`plan`**（对该会话的线上计划运行
`renderPrBody`，含被采纳的提交 —— 只要该计划有条目就是默认值；`planAvailable` 报告它到底是否渲染出
任何内容）、**`commits`**（`<base>..HEAD` 的主题行，最旧在前，作为项目符号列表 —— 其他情况下的默认
值，因此一个无计划或空计划的 PR 仍得到诚实正文；`host` 在来自 pre-v87 客户端的、以会话为后端的预览
上显式要求 `plan`，因为那种客户端只见过计划正文）、**`template`**（所选发现模板的原样内容；`template`
缺失时取第一个），或 **`empty`**。**模板发现**读的是工作树，不是 forge：GitHub 的
`pull_request_template.md`，位于 `.github/`、根目录或 `docs/`（任意大小写），以及每个
`.github/PULL_REQUEST_TEMPLATE/*.md`，再加上 GitLab 的 `.gitlab/merge_request_templates/*.md`；路径
是工作树相对的、有大小上限且被包含校验（每个文件的真实路径，解析符号链接之后，必须落在真实工作树
之内），并按该顺序列出。

`prDraftInput({ workspaceId, sessionId?, template? })` → `assist.suggestPrDescription` 据以起草的有界
输入（`pr.generate`，在 `host` 中组装）：分支 + 基点、提交主题、对基点的 `git diff --stat` 并夹到一个
固定行数预算、会话计划渲染出的正文（若存在），以及所选模板的文本。

`pushBranch({ workspaceId, branch, confirmBase? })` → `{ dirtyFiles }`（`workspace.push`）：安全 ref
守卫与 `refreshUserOwnedWorkspace` 重新校验，分别在脏文件读取之前与之后 —— 每次 live 分支都必须仍是
请求点名的那个 `branch`（用户看到并确认的那一个），因此中途的一次终端切换什么也不推送，而不是推送
另一个分支 —— 然后是同样的 `--set-upstream origin <branch>` 推送、环境变量，以及下文第 2 步的
`PUSH_AUTH_FAILED` 分类 —— 只有一份实现，`openPr` 也调用它。一个**就是**自己基点的分支被拒绝，除非
设置了 `confirmBase`（客户端先确认，点名那条共享分支）；它从不读取、编辑或创建 PR，且像每次推送一样
丢弃工作树的评审缓存。

`openPr({ workspaceId, sessionId?, title?, body?, draft? })`（`sessionId` 只在 `body` 缺失时才重要：
此时一个会话的 PR 会从其计划正文刷新 —— 显式指定 source `plan`，永不用预览的默认值 —— 因此 *Push
updates* 让一个只由被采纳提交组成的计划保持其结构化形式），一次调用 = 一次幂等尝试（`body` —— 用户
在该对话框中编辑过的描述 —— 存在时原样替换渲染出的计划正文；标题/正文默认值只来自**一处** —— openPr
消费 `previewPr`，因此预览的草稿与被推送的 PR 永不背离）。该调用以 **`assertSafeRef(ws.branch)`**
开头：被采纳/外部工作树的分支是按 git 原样存储的（只有创建出来的工作区才走 `toBranch`），因此来自不
可信仓库的、形如选项的分支（`--repo=x` 可通过 `git update-ref` 创建）必须在进入任何 git/gh argv 之前
被拒绝。然后它在脏文件读取或推送**之前**拒绝**非 origin 的远端基点**：它只针对 `origin` 推送与打开；
悄悄把 `upstream/main` 当作该仓库中的分支，会先发布工作区分支，随后才发现无法确定其 PR 目标。一个
活得比其远端配置更久的远端跟踪 ref 也会在同一侧被拒绝。错误会引导用户手动打开，或从一个本地/
`origin` 基点创建；跨仓库的 fork PR 创建属于未来范围，不从字符串前缀猜测。下一个守卫在该分支**就是
它的基点**时拒绝（`ws.branch === baseRef(ws.baseBranch)`，仍在 origin 检查或任何推送之前）：Default
工作区的 cwd 就是项目自己的检出，因此只要用户在仓库的默认分支上，`branch`/`baseBranch` 通常重合
（`workspaces` 的 `folderTruth` 从实时的 `git symbolic-ref` 跟踪二者），而一个外部（被采纳的）工作区
同样可以停在它的基点上 —— 没有这个守卫，Open PR 会 `git push origin <default-branch>`，直接推到未
受保护远端上的共享分支，在任何 PR 能够提供其评审边界之前就发布了提交。客户端镜像了通常的本地/
`origin` 同名分支情形；宿主对不支持的远端基点保持权威。**`refreshUserOwnedWorkspace(workspaceId)` 为
Default/外部工作区解析 LIVE 分支**（`workspaces` 中与 fs-watcher tee 相同的那次同步）并持久化它 ——
对创建出来的工作区是空操作，因为它们的分支只有 ThinkRail 会移动。Open PR 在其异步脏文件读取之前
校验一次（保住安全基分支的快速失败），然后在该 await 之后立即刷新、重载、重新校验。两者缺一，
在 Git 读取进行中或按下 Open PR 前一刻于终端切换的分支，就会在异步 watcher 下次追上之前，按**陈旧的**
已持久化分支去推送/打开/比较：分支敏感的 `ws` 是新的，因此基点守卫、推送、`gh` 查找/创建以及比较
URL 全都源自那一个刷新后的值。当请求点名了 `branch`（客户端据以撰写草稿的那个分支）时，两次刷新都
必须仍看到它是 live 的，完全与 `pushBranch` 一样：在撰写与提交之间被移动的检出什么也不推送，而不是
用这一分支的标题和正文去为另一个分支打开 PR。然后：
1. 没有 `origin` 远端 → 抛错（客户端弹出提示）。forge（GitHub slug / GitLab 项目）读自 origin 的
   **已配置推送 URL**（`remote.origin.pushurl`，否则 `remote.origin.url`，经由 `git` 的 `originUrls`），
   而不是 fetch URL，因此一个推送目标与拉取目标不同的远端，会在分支实际落地的仓库上打开评审；
   `insteadOf` 重写不被展开，因为它们映射的是同一宿主的传输层。推送失败 → 带 git stderr 抛错；
   若 stderr 匹配非交互式认证特征（`Permission denied (publickey)`、
   `could not read Username/Password`、`Authentication failed`、`terminal prompts disabled`、
   `Host key verification failed` —— `isPushAuthFailure`），则改为抛出
   `CodedError("PUSH_AUTH_FAILED")`：宿主在没有终端的情况下推送，因此 git 的凭据提示永不可能出现，
   客户端用设置指引作答，而不是一份原始 stderr 提示。
2. 总是先推送 `--set-upstream origin <branch>` —— 分支由工作区拥有，因此一次普通推送就是对的；重复
   调用把后续提交推到**同一**分支（永不是第二个 PR）。推送在**非交互环境**（`pushGitEnv`）下运行：
   `GIT_TERMINAL_PROMPT=0`；**`LC_MESSAGES=C`，并把 `LC_ALL` 降级为 `LC_CTYPE`**（丢弃，其值在原本
   未设置时保留为 `LC_CTYPE`），好让 git 的 stderr 在任何宿主 locale 下都保持英文以匹配
   PUSH_AUTH_FAILED 模式 —— 同时推送所派生的钩子与辅助程序保留它们的字符编码（`LC_ALL=C` 会把基于
   Python 的 pre-push 钩子翻到 ASCII 模式；完全丢弃一个只设 LC_ALL 的 locale 也会因 POSIX 的
   `LC_CTYPE` 产生同样效果）；以及 `GIT_SSH_COMMAND=ssh -oBatchMode=yes` —— 但**只在**用户自己没有
   ssh 命令时（`GIT_SSH_COMMAND`/`GIT_SSH` 环境变量或 **`core.sshCommand` git 配置** —— 环境变量
   覆盖配置，因此盲目设置会破坏一个可用的自定义密钥/1Password 设置，并将其误报为认证失败）。当宿主
   有控制终端时，ssh 否则会在 `/dev/tty` 上提示输入密钥口令，把整个调用挂到客户端超时 —— 批处理模式
   把这次挂起变成即时的 `Permission denied`，正是分类所针对的。
3. GitLab.com origin → `action: "compare"`，带 GitLab 预填的新建 MR 页面（`gitlabNewMrUrl`：source、
   target、title、description，按 GitHub compare 正文同样方式限长）—— 本迭代不做任何 `glab` 变更。
   任何其他 origin（含本地/bare 测试远端）→ `action: "pushed"` —— 诚实的下限。
4. GitHub origin：每次 `gh` 调用都携带 `--repo <slug>`，即推送 URL 的仓库 —— gh 否则会从远端的 fetch
   URL 解析仓库，因此一个分裂的 origin 会去查找、编辑或创建那个从未收到该分支的仓库上的 PR。已存在
   的打开 PR（`gh pr list --head <branch> --base <input.base>`，由 `branch-review` 的 `reviewNumber`
   解析）→ `gh pr edit <NUMBER> --title --body` 刷新 → `action: "updated"` + **`bodyRefreshed`**（编辑
   失败会被报告，永不被声称为成功）。编辑选择器是列表步骤刚刚返回的 NUMBER，永不是分支位置参数 ——
   gh 会把数字分支名（"1234"）解析成 PR 编号，那会改写陌生人的 PR —— 而 `--title` **只在客户端标记
   它被编辑过**（`titleEdited`）时才随行：撰写预填是从计划重新生成的，不是从在线 PR 读取的，因此一个
   未被触碰的标题永不能覆盖在 github.com 上做的重命名。列表步骤按 **`--head` 与 `--base` 两者**过滤
   —— 裸 `--head` 会匹配该分支上任何目标的打开 PR（同一个 head 可以有两个指向不同基点的打开 PR），
   从而编辑/报告一个与本工作区完全无关的 PR；没有 → `gh pr create --base <ws.baseBranch minus
   origin/>` → `action: "created"` + 从 gh 输出解析出的 `{ review, url }`。**`--base` 总是显式的**
   （`baseRef`，与 compare URL 相同的规范化）：没有它，gh 会指向分支配置或仓库默认分支，因此一个从
   `release/x` 创建的工作区会对着 `main` 打开它的 PR。**变更型 gh 调用以 60s 超时运行**（共享 runner
   的 8s 默认只适合只读查找 —— 中途杀掉一次变更正是重复 PR 产生的方式），而一次失败/含糊的 `create`
   之后会跟一次**复查**（再次 `gh pr list`）：若尽管报错 PR 已在服务端存在，就把它作为 `created`
   返回 —— 永不用 compare 回退，那会招来一个重复。
5. `gh` 缺失/未认证/失败，或 `THINKRAIL_GH_OFFLINE=1`（与 `github` 模块共享的 e2e 接缝）→
   `action: "compare"` + 一个预填的 `quick_pull` compare URL（标题 + 正文，正文为 URL 长度安全而限长
   到 4k）—— 客户端在浏览器里打开的零认证回退。当 `github` 的 `ghSetupProblem()` 能点名原因
   （`missing` / `unauthenticated`）时，结果以 **`ghProblem`** 携带它，好让客户端显示设置指引，而不是
   一个光秃秃的 compare 交接。该探测**只在 gh 流程确实失败之后**运行（永不在顺途上 ——
   `gh auth status` 是一次网络往返，而一个*能工作*但报告古怪的 gh，例如一个陈旧的第二宿主 token，
   永不能让它的单击路径被跳过），且它是**带 8s kill 的异步**调用 —— 进程内宿主共享一个事件循环，
   同步探测会冻住每个会话。发现 gh 已安装 + 已认证（一次瞬时流程失败）的探测以及离线接缝仍不被
   点名 —— 静默的 compare 回退保持。
6. 每个结果都携带 `dirtyFiles`（未提交变更数），好让客户端警告这些不会进入 PR。脏状态永不阻塞。
   服务端的 `openPr` 结果还携带用于 push/gh/compare 的确切刷新后 `branch`；宿主据此对评审快照做 CAS
   并在线上响应之前把它剥掉。在脏读取 await 之前捕获分支，对用户拥有的检出是不健全的，因为 `openPr`
   刻意在该 await 之后刷新。

**评审动作**（`reviewActions.ts`，v88）：`readyPr`（`pr.ready`）、`reopenPr`（`pr.reopen`）、
`mergePr({ …, method, deleteBranch })`（`pr.merge`）与 `deleteRemoteBranch`
（`workspace.deleteRemoteBranch`）。每个 PR 变更都绑定到用户所看到的：刷新后的 live 分支必须仍是请求
的 `branch`（终端切换什么都不改），且不能是它自己的基点，而 `gh pr view <number>` 必须仍显示该编号
从 **origin 自己的仓库**指向这个分支（`headRepositoryOwner` / `headRepository` 对照 origin 推送 URL
的 slug），且处于该动作所需的状态（ready/merge 为打开，reopen 为已关闭）。`gh pr list --head` 仅按
分支名匹配，因此一个 fork 的、来自同名分支（`contributor:main`、`patch-1`）的 PR 否则无法区分 —— 而
基分支拒绝使停在 `main` 的 Default 工作区永不会对其动手。合并传入 **`--match-head-commit <HEAD>`**，
让 GitHub 恰好合并用户所看的那个 head（客户端只在分支与 origin 一致时提供 Merge），然后重读 PR：
`gh pr merge` 在合并队列或自动合并只是**入队**时也会退出 0，因此 `MergePrResult.merged` 报告 GitHub
显示的状态，而分支只在真正合并之后才删除 —— 删除一个已入队 PR 的 head 分支会关闭它。变更经注入的
runner 在 60 s 超时下运行，被拒时暴露 gh 的 stderr，且不论结果如何都丢弃工作树的评审缓存。与
`openPr` 的 gh 流程一样，每个评审动作的 `gh` 调用都携带 **`--repo <push slug>`**：分支是从推送
仓库删除的，因此被检查的 PR 也必须住在那里 —— 只靠 gh 会解析 fetch 仓库，在一个分裂的 origin 上那是
另一个仓库的 PR。**远端分支用 git 删除，永不用 `gh pr merge --delete-branch`**：gh 还会删除本地分支
并切换检出 —— 对工作树和对用户自己的检出都是致命的。两条路径都通过 `pushOrigin`（`pushToOrigin` 也
用的那一个推送辅助函数，含 `pushGitEnv` + `PUSH_AUTH_FAILED`）运行
`git push --force-with-lease=refs/heads/<branch>:<tip> origin --delete refs/heads/<branch>`（完整
ref，因此同名标签永不是被删的那个）。lease 点明删除所据以校验的那个 tip —— 合并之后是已合并的 HEAD，
独立删除则是远端跟踪的 `origin/<branch>`，后者还必须包含在 HEAD 中 —— 因此其间被某人推送的提交永不
成为远端唯一副本而被删除：lease 被拒，什么也不删。一次被拒的 lease 若其 ref 已消失（对推送 URL 做
`ls-remote`）则计为已删除 —— GitHub 自己的自动删除可能已胜出。合并成功之后，删除失败会报告在
`MergePrResult.deleteError` 中，而不是抛出。在任一次删除之前，
`gh pr list --repo <push slug> --head <branch> --state open` 必须回答该处没有以本分支为 head 的
打开 PR（来自该仓库；一个 fork 的同名 PR 不算）：删除一个 PR 的 head 分支会关闭它，而未得到回答的
检查什么也不删。非 GitHub 的 origin 改用新鲜的 `branch-review` 查找，并遵循同样的「无回答即拒绝」
规则。独立删除也拒绝基分支。

打开 PR 的*读*侧仍是 `workspace.branchStatus` / `workspace.openReview`（`branch-review`）—— 本模块
不持久化任何东西；客户端从那次查找加上本调用的结果重新推导按钮/徽标状态。那次查找是缓存的，因此
每次成功推送都会丢弃工作树先前的答案：即使远端不是 GitHub，或动作回退到 compare，推送也可能改变
分支的远端配置。一次 GitHub 变更尝试在结算之后再丢弃它一次，因此并发的「无 PR」读取无法在一次更新、
一次创建或一次含糊的创建结果之后存活。

## 边界（Boundary）

- **拥有：** 推送（带或不带 PR）+ PR 动作、GitHub 评审变更（ready、merge、reopen）与远端分支删除、
  从线上计划渲染 PR 正文、提交与模板正文来源、PR/MR 模板发现、生成输入、GitHub slug/compare-URL
  推导。
- **公开接口（barrel）：** `openPr`、`previewPr`、`pushBranch`、`prDraftInput`、`onBaseBranch`、
  `readyPr`、`mergePr`、`reopenPr`、`deleteRemoteBranch`。
- `onBaseBranch` 是「分支就是它自己的基点」那唯一一处检查，与 `host` 的 `workspace.branchStatus`
  共享。
- **允许的依赖：** `workspaces`（工作区记录、`refreshUserOwnedWorkspace`）、`git`（exec + status）、
  `todos`（正文用的 `listTodos`）、`branch-review`（provider 探测、`githubSlug`、既有 PR 查找、远端
  删除前的新鲜 open-review 检查、共享命令 runner、`forgetOpenBranchReview`）、`github`
  （`ghSetupProblem` —— 被点名的 compare 回退原因）；`contracts` 类型；`shared/codedError`
  （`PUSH_AUTH_FAILED`）。
- **禁止：** `host`；provider 的 HTTP API 或已存 token（认证保持外部 —— 只有用户的 `gh`）；持久化
  PR 状态（那会重复 `workspace.openReview` 的推导）。
- 所有 `gh` 流量都经过一个注入的 `PrCommandRunner`，因此单元测试永不 spawn 真实 CLI。
