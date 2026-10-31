# git —— runner + worktree 状态/diff

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

Git 管道层：底层的 `git` runner（同步 + 异步），加上一个 worktree 在**diff 范围**内的变更文件与 diff、
该范围的唯一定义（范围解析器）、供分支选择器使用的项目仓库分支列表、工作区分支自己的提交列表，
以及一个把远程基准 ref 预热出工作区创建关键路径的后台预取。

## 边界（Boundary）

- **拥有：** `git(cwd, args)`（*同步* spawn git，捕获裁剪过的 stdout/stderr + ok；`opts.raw` 保留
  未裁剪的解码 stdout）及其模块内部孪生 `gitBytes(cwd, args, opts?)`（同步、保字节，留给 `reviews`
  的同步快照轮）。`gitAsync(cwd, args, opts?)` 与 **`gitAsyncBytes(cwd, args, opts?)`** 是基于
  `subprocess.runBounded` 的有界异步孪生；后者把 stdout 保留为 `Uint8Array`，而 stderr 仍是解码的
  诊断。它们把绑定网络的操作如 `fetch` **以及请求路径上的历史读取**挡在事件循环之外：该模块只拥有
  git 形状的 55 秒预算与停滞/stderr 措辞，绝不拥有子进程生存期机制。`opts.env` 让调用方用自己的环境
  无提示地运行，并把 blob 诊断钉为英文；捕获型 runner 的可选 `opts.redactStderr` 在任何截断或超时
  措辞之前转换诊断（clone 用它处理携带凭据的 URL 重写）。`opts.network` 仅标记 fetch/push，
  以便一次无输出的超时能点名远程。**超时保留 git 在被杀之前写下的任何内容**：runner 持续排空，
  因此到期时它的 `err` 已持有真实诊断——publickey 拒绝、代理的拒绝、证明大传输仍在进行的 `remote:`
  进度。当 git 什么也没写时，显式联网的操作得到有条件的 ssh-key 提示；本地 `diff`/`log`/ref 读取
  只得到观测到的事实：git 没有退出——绝不编造远程原因。两个异步 runner 都为执行失败携带
  `failure: "timeout" | "launch" | "aborted"`；正常的非零 Git 退出不携带执行失败，因此语义探测能
  区分“ref 不存在”与“Git 从未作答”。两者都接受 `opts.signal` 并把取消委派给 `runBounded`：
  预先取消不启动任何东西，而在飞取消只在直接子进程退出后才返回。被取消的结果保留已捕获的 stdout
  （按通常的 text/raw 或 bytes 处理），给有界 stderr 加上显式的 `git operation cancelled` 诊断前缀，
  且永不报告超时或 SSH 提示。`gitAsyncStream` 不接受该选项；其消费者取消 stdout 流。这些读取沿用同样的
  55 秒默认值（`opts.timeoutMs` 覆盖它）：一个本地读取若还需要被*有界*，那它就是卡住的 git，而它们在
  搬出事件循环之前全都是无界的；**`remoteTrackingRef(ref, remotes)`** → 当 ref 以仓库某个已配置远程名
  开头时为 `refs/remotes/<ref>`（`remoteNameOf`，由 `listRemotes` 供给），否则 `null`——**构造该写法的
  唯一地方**，因此下面的探测与 `workspaces` 的 `worktree add` 不会漂移。匹配选择最长的远程前缀，
  因为 Git 允许远程名里含 `/`（`team/upstream`）；按第一段解析会抓错远程。远程性按实际远程列表判定，
  绝不按字符串形状：`upstream/main` 与本地 `feature/main` 形状相同。因此 `listBranches` 枚举整个
  `refs/remotes`（每个符号别名，包括各远程的 `HEAD`，都被跳过）并在同一异步扇出中读取 `git remote`，
  以可选的 `BranchList.remoteGroups` 附加权威所有权。所有权读取失败会 reject 该目录，而不是让浏览器
  去猜；没有已配置属主的跟踪 ref 保留在 `null` 组下。`prefetchBranch` 从 ref 所点名的任何已配置远程
  抓取，而 `workspaces` 抓取并检出同一个完全限定的跟踪 ref。
  **`resolveDefaultBranch` 保留其同步 spawn 预算**——每个 Default 工作区的 `folderTruth` 都会调用它，
  因此多一次 spawn 就是每次工作区列表都要付的共享事件循环时间。origin 的 HEAD 仍是单独的
  `symbolic-ref`（唯一能在它悬空时看到它的读取），随后是对完整 `refs/remotes/…` 名字的一次
  `for-each-ref`。只有全名以 `/HEAD` 结尾的 ref 才能提供另一个远程的默认值；任意符号别名被忽略，
  而对 `refs/remotes/origin/main` 的确切全名检查不会被一个字面名为 `origin/main` 的本地分支混淆。
  一个悬空的非 origin HEAD 仍对 `for-each-ref` 不可见，并降级为既有猜测。它的触及范围**仅是创建**，
  而 `resolveDiffRange` 是那个具名幸存者：`diffBaseRef` 把记录在 `baseBranch` 中的 `origin/<b>`
  简写交给 git，因此在创建现在防范的那种情形——字面名为 `origin/main` 的本地分支——中，worktree 从
  `refs/remotes/origin/main` 切出，而 diff 却对着那个诱饵测量。之所以不改，是因为 `diffBase` 可单独由
  用户设置并可命名任何 ref，因此限定它属于改变*被钉住*基准的含义，而不是拼写修正；
  **`remoteRefOid(repoPath, ref)`** → `refs/remotes/<ref>` 解析到的 oid，或 `null`——**询问
  远程跟踪 ref 是否存在的唯一方式**，由 `prefetchBranch`（比较 oid 以报告 `moved`）与 `workspaces`
  的创建回退共享。它以 `isSafeRef` 而非远程列表为门禁，因为该跟踪 ref *是否解析*就是它回答的全部，
  而这样的 ref 会比远程的配置活得更久。它在 `--end-of-options` 之后探测**完整的** `refs/remotes/`
  路径：`origin/<b>` 简写会走 git 的消歧，可能解析到同名标签或本地分支，那样各调用点就会在不同对象上
  竞速。“唯一方式”是字面意思——没有调用方手工重拼 `rev-parse`；
  **`nonInteractiveGitEnv()`**——两个 runner 默认 spawn 的环境（`git` 唯一的选项是 `raw`；
  只有 `gitAsync` 接受 `opts.env` 覆盖，例如 `pr` 的非交互 push，它会叠加自己的 SSH batch-mode 设置）：
  `process.env` 加上 `GIT_TERMINAL_PROMPT=0`，默认**别无其他**。它不读任何配置，也不自行重写用户的
  ssh 设置；**请求路径的读取都走 `gitAsync`**——`resolveDiffRange`、`gitStatus`、`gitDiffFile`、
  `listCommits`、`listBranches` 以及工作区徽标扇出都是异步的，因此一次多 spawn 的读取永不冻结宿主的
  单个协作事件循环（迁移前实测每次 `workspace.list` 冻结循环 119–246ms）。同步 runner 保留给
  **写入者**（它们的加载→变更→保存原子性依赖不交错——`gitCommitPaths` 的索引快照/恢复、
  `workspaces` 的写入者）以及**微管道叶子助手**（与那些写入者共享）
  （`currentBranch`/`tryCurrentBranch`/`resolveDefaultBranch`/`resolveCommitOid`/`readBlobAt`/
  `gitHeadSha`——单次快速的本地 ref 读取），外加 `gitUncommittedPaths`，那个刻意保留的生命周期例外：
  它在 agent 能越过其 `in_progress` 工具结束继续之前快照一个 TODO 工作窗口；
  **`snapshotWorktree(worktreePath)`** → git 此刻看待该 worktree 的树 oid（已跟踪编辑与未跟踪文件，
  排除被忽略文件），通过 `GIT_INDEX_FILE` 下的索引临时副本写入，因此用户的真实索引、`HEAD` 与 reflog
  永不变动——这是 `turns` 模块围绕一次 agent 运行的前后栅栏；它返回 `null` 而不是抛出，因为错过一次
  快照只损失一个回执。git 读不到的文件被跳过而非致命：`add --ignore-errors` 在写完全部其他内容后仍以
  1 退出，因此只有 `fatal:` 或超时/启动失败才作废它；**范围→区间解析器**——
  `resolveDiffRange(ws, scope?)` → `Promise<DiffRange>`（异步——且刻意保持为*单一*实现：它的
  `reviews` 消费者随它一起转异步，而不是留一个易漂移的同步孪生）——**`GitDiffScope` 含义的唯一定义**
  （`branch`：`git diff <merge-base(base, HEAD)>` + 未跟踪，两侧 = **分叉点** ↔ worktree——即工作区
  *自分叉以来*改了什么，因此在它下方前进的基准（一次抓取移动 `origin/main`、上游工作落地）永不浮现
  为幽灵变更；当基准尚未分叉时 merge-base *就是*它的顶端，而正常的非零 `merge-base`（缺基准、
  无共同历史、未出生 `HEAD`）回退到原始 ref，保留旧有的错误面——并让文件列表与
  `listCommits` 的 `base..HEAD` 在祖先关系上保持一致；超时或启动失败会抛出而不是冒充该语义结果。
  同一区分让一次超时的提交探测不会变成 `UNKNOWN_COMMIT`，也让一次超时的父提交探测不会变成根提交；
  `uncommitted`：`git diff HEAD` + 未跟踪，两侧 = `HEAD` ↔ worktree；`commit`：
  `git diff <sha>^ <sha>`，无未跟踪，两侧都来自历史——**根**提交降级为带空原始侧的
  `git show --format=`，与缺失路径已有的添加式降级相同；`pinned`：`git diff <oid>` + 未跟踪，
  两侧 = 给定的不可变提交 ↔ worktree——review 侧栏的基准侧导航，校验方式与 `commit` 的 sha 完全相同，
  同样的 `UNKNOWN_COMMIT` 拒绝；`turn`：`git diff <baseTree> <headTree>`，无未跟踪，两侧都是
  `turns` 模块围绕一次 agent 运行所取的**树**快照——每个 oid 都做形状校验并验证为 `^{tree}`，
  被剪枝的快照以同样的 `UNKNOWN_COMMIT` 拒绝，使客户端以同样方式重置范围）。两种读取都通过
  `changedFileArgs(range, mode)` 从它构建 argv，因此文件列表与某文件的两侧永不就该区间产生分歧——
  而那个 argv 在**两侧**都界定它的 rev：前面是 **`--end-of-options`**（没有 ref 能被重新解析为
  git 选项），后面是结尾的 **`--`**（一个同时命名磁盘路径的 rev——一个叫 `docs` 的分支——会被当作
  rev 读取，而不是让命令因“歧义参数”失败）。**失败**的 `git diff`、未跟踪 `ls-files` 或 `git show`
  都会**抛出**；只有 Git 显式的路径缺失诊断才产出空文件侧，因此坏读取永不被报告为无变更或添加/删除。
  `commit` 范围的 `sha` 被校验**两次**——形状（十六进制 oid 正则，因此构造的值永不能以选项或路径形式
  到达 git 参数）然后存在性（`rev-parse --verify`，其完整 oid 才是随后使用的值）——而消失的提交抛出
  **`CodedError("UNKNOWN_COMMIT")`**（`@thinkrail/shared/codedError`），宿主把它作为
  `WsResponse.errorCode` 上线，客户端把它转成“重置该范围，并弹 toast”——*只*对该具名失败，
  绝不对超时或未具名的宿主失败；
  **`isSafeRef(ref)` / `assertSafeRef(ref)`**——每个**用户/仓库提供的 ref** 在其变更入口通过的形状
  检查（`workspaces` 的 `createWorkspace` 基准——**解析后**的那个，包括从仓库自身 `HEAD` 读出的值
  ——+ `setWorkspaceDiffBase` 目标）。规则集就是 `git check-ref-format` 的规则，在进程内复现
  （校验路径上不 spawn）：非空、无前导 `-`、无空白/控制字符、无 `..`、无修订元字符
  （`~ ^ : ? * [ \`）、无 `@{` 且无裸 `@`、无空路径组件、无以 `.` 开头的组件、无 `.lock` 后缀、
  无结尾 `.` 或 `/`。git 自己拒绝的名字绝不是我们会接受的——并且对称地，**不设长度上限**：
  `check-ref-format` 没有，因此仓库真的有（且 `for-each-ref` 真的列出）的长分层分支仍可选；长度不是
  安全属性，而真正的限制（文件系统组件上限、argv 大小）会作为读取错误大声失败，而不是“格式错误”。
  威胁来自**不可信仓库**，而不是恶意客户端：`git update-ref` 接受像 `refs/heads/--output=x` 这样的
  名字（只有 `git branch` porcelain 拒绝它），`listBranches` 用 `for-each-ref` 读 ref，因此选项形状
  的分支会进入用户打开的任何仓库的选择器——而浏览别人的仓库正是产品的职责；
  **`diffBaseRef(ws)`**——`diffBase ?? baseBranch`，工作区两种基准含义（创建来源 vs 评审目标）的
  唯一塌缩点，由解析器与 `listCommits` 消费（`workspaces` 模块的 `diffStats` 是*经由*解析器到达它
  ——见“必须做对的点”）；**`resolveCommitOid(worktreePath, ref)`**——某 ref 此刻点名的完整提交 oid，
  或 `null`。符号 ref 被**冻结**的唯一地方，而每个此后仍须表示同一含义的调用方都走它：review 的
  `baseSha`、基准侧评论的 `baseRef`。范围的 `originalRef` 并非已经不可变（`uncommitted` 是字面
  `HEAD`；`branch` 范围在 `merge-base` 失败时降级为原始基准 ref），因此逐字存储它会让内容在存储者
  脚下移动。`DiffRange` 把该冻结形式作为 **`resolvedOriginalOid`** 携带（通过 `resolveCommitOid`
  冻结，原始侧没有提交时为 `null`——根提交、未出生 `HEAD`、不再解析的基准 ref）：必须读取*同一*
  原始侧两次的调用方——`changes` 模块的加载→校验→写入、宿主的不可变 `/blob` URL——按 oid 而非按
  某个含义可能被提交在请求中途改变的 ref 来寻址；
  `gitStatus(workspaceId, scope?)`——区间内的已变更文件加未跟踪文件（仅当区间结束于 worktree 时），
  各自携带按文件的 `added`/`removed` 行数（`git diff --numstat`，其改名畸形的路径经 `numstatPath`
  解析以匹配 `--name-status`；二进制行被丢弃；未跟踪文件仅当 `fs.classifyBytes` 认可它是文本时才把
  整份内容算作新增，因此非法 UTF-8、含 NUL 与魔法类型的未跟踪文件与其 `ResourceMeta` 一样省略计数）
  供 Changes 树的 `+/−` 徽标使用；
  `gitDiffFile(workspaceId, path, scope?)` →
  `{ original, modified, originalOid, meta: { original, modified } }`——`originalOid` 是该区间解析出的
  不可变起点（或 `null`），而两者是单个文件在中央 diff 标签页的两侧（`original` = 区间起始 ref 处的
  文件，原始、那里不存在时为空——未跟踪/新增、改名文件的新路径、或根提交——降级为添加式 diff；
  `modified` = 结束于那里的区间取 worktree 文件（被删除时为空），否则取提交自己的树；路径经
  `fs.resolveWorktreeFile` 校验词法逃逸、`.git` 与逃逸的符号链接；历史侧使用有界异步
  `git cat-file blob` 并带确定性英文诊断，因此只有 Git 显式的路径缺失结果才成为空侧；树、提交、
  gitlink、普通非零失败、超时与启动失败全都抛出）。**两侧都按字节读取并携带 `ResourceMeta`**
  （`fs.resourceMeta`——sha-256、字节长度、文本性、嗅探 mime）：一侧只有是文本时才解码，因此纯字节侧
  以 `""` 加上其元数据传递，客户端改经宿主的 `/files`/`/blob` 路由抓取字节，而不是渲染替换字符。
  提交范围的两侧通过 `gitAsyncBytes` 并发读取，既保持字节身份又不阻塞事件循环、也不 spawn 第二次
  解码读取；**`gitUncommittedPaths(workspaceId)`** → 保留给上述 TODO 基线边界的同步已跟踪 + 未跟踪
  路径集，用 NUL 分隔符与原始输出读取，因此文件名中合法的前导/尾随空白（或换行）就是身份，永不裁剪；
  `gitStatus` 对其已跟踪/未跟踪行使用同样的 NUL 安全路径处理；
  **`listCommits(workspaceId)`** → `{ commits: GitCommit[] }`——`git log <diff base>..HEAD`，
  最新在前且有上限，每个提交一行 `--format`，其字段以 **NUL 字节**分隔并按**固定元数**读取
  （前四个按位置，其后所有内容重新拼接为主体）。NUL 是仓库可控文本无法偷带的唯一字节：作者身份既不
  携带 NUL 也不携带换行，因此任何构造过的 `%an` 都无法让 `%cI` 位移或截断自身，而携带一个的 `%s`
  无论如何都会落在尾部。（`%an` 是*结构化字段与主体之间*的自由文本，这也是“先结构化字段”永不足够的
  原因——一个名为 `a<sep>2020-01-01T00:00:00Z` 的作者会让主体位移一个字段。）自由文本字段随后在
  上线之前被剥掉控制字符**以及不可见欺骗**——双向覆盖/隔离符、零宽与格式字符——而普通国际文本与
  emoji 存活；一个让 Git 正常退出的不可读区间（被删除的基准、未出生 HEAD）降级为空列表，使范围菜单
  仍能提供它的其他范围；超时或启动失败会抛出，而不是抹掉此前有效的列表；
  **`countPushDivergence(worktreePath, branch, {fetch?})`**（异步——同步孪生会在每次窗口聚焦重抓时
  阻塞共享事件循环）→ `{ ahead, behind }`，经一次
  `rev-list --left-right --count origin/<branch>...HEAD`：**`ahead`** = origin 没有的本地提交
  （未推送），**`behind`** = 分支没有的 origin 提交。`behind > 0` 意味着分支已**分叉**——纯 push 会
  非快进，历史必须调和（plan 页面把它当作同步冲突，而不是强推提示）。只在远程 ref 不存在（从未推送）
  时为 `null`；超时/启动失败抛出，而 ref 仍存在的正常非零也会抛出，而不是冒充不存在。带 **`fetch`**
  时它先尽力刷新 `origin/<branch>`，使 `behind` 反映*真实*远程（离线时回退到最近的已知 ref，
  而不是让整个查询失败）；宿主的 `workspace.openReview` **只在全新查询**时传 `fetch`
  （聚焦/显式刷新，而不是缓存激活）并把 `ahead`→`unpushedCommits`、`behind`→`behindCommits`
  组合到一个打开的 review 上（与 gh 查询并行，而不是之后），使 plan 页面能标记 PR 尚无的提交与
  已分叉的同步冲突状态；**`originUrls(worktreePath)`**（异步）→ origin 已配置的 `{ fetch, push }`
  URL（`push` = `remote.origin.pushurl`，否则抓取 URL；不展开，无 origin 时为 `null`）；
  **`countPushTargetDivergence(worktreePath, branch, pushUrl)`**（异步）→ `{ published: false }` 或
  `{ published: true, ahead?, behind? }`，对推送 URL 与抓取 URL 不同的 origin 而言是相对**推送**仓库
  测量的：在那里远程跟踪的 `origin/<branch>` 镜像的是抓取仓库（一次抓取会覆盖推送写下的内容；git 自己
  的 `@{push}` 解析到同一个 ref），因此它用 `ls-remote` 读推送目标的顶端，在对象缺失时抓取该顶端的对象
  （`--no-write-fetch-head`，不写任何 ref——在每次读取时都如此，包括缓存的读取，因为 `ls-remote`
  已经走过网络，而抓取侧的 ref 绝不能替代它），然后与 HEAD 比较；始终不可用的顶端让计数保持未知，
  只有失败的 `ls-remote` 才抛出；**`commitsAheadOf(worktreePath, baseRef)`**（异步）→
  `rev-list --count <base>..HEAD`，对不安全或不可解析的基准返回 `0`，永不抛出——这里的 `origin/`
  是全远程清扫的第二个刻意幸存者，因为它问的是*这个*工作区自己的分支被推到了哪里，
  而不是某个基准从哪个远程分出；`listBranches(projectId)` → `{ local, remote, remoteGroups?,
  defaultBranch }`（本地 `refs/heads`；规范 `remote` = `refs/remotes` 下每个直接完整 ref 并省略
  符号别名；附加的 `remoteGroups` = 宿主拥有的远程/分支元数据供展示；默认 =
  origin 的 `HEAD`→另一个远程的 `HEAD`→`origin/main`→仓库 `HEAD`；任何 ref 列表或所有权列表失败
  都抛出，绝不留成功的不完整目录），**`resolveDefaultBranch(repoPath)`**——把该默认分支解析抽出
  （命名一次），由 `listBranches` 与 `workspaces` 模块的 Default 工作区 ensure（其 `baseBranch`）共享；
  它最后回退到 `currentBranch`，因此未出生 `HEAD` 解析为它将成为的分支名，而不是字面 `"HEAD"`
  （那会被持久化进用户可见的 `baseBranch`）；**`currentBranch(repoPath)`**——checkout 当前拿出的
  分支（`symbolic-ref --short HEAD`，未出生安全；分离 → 字面 `HEAD`），由 `workspaces` 模块用于
  用户自有工作区的 folder-truth `branch`，而 **`tryCurrentBranch`** 是其可失败形式（路径不是可读的
  worktree 根时为 `null`，因此刷新永不把 I/O 失败持久化为 detach）；
  **`canonicalPath(path)`**——任何与 git 输出比较的路径必须采用的符号链接解析形式（git 解析符号链接，
  调用方的路径不会），与 `workspaces` 的 worktree 身份检查共享；`prefetchBranch(projectId, ref)`——
  一次对已配置远程 ref 的尽力后台 `git fetch`（经 `gitAsync`），使后续 `createWorkspace` 从新鲜顶端
  分出而关键路径上没有网络往返（本地/未知 ref 或离线 → 空操作）。它的 ref **由线上提供，并通过
  `createWorkspace` 使用的同一 `isSafeRef` 门禁**之后才到达 git：`--` 分隔的是*选项*而不是 refspec，
  因此仅在 `fetch` 之后加 `--` 只挡住了 `-` 前缀的名字，别的什么也没挡——
  `origin/+main:refs/heads/victim` 本会作为 **refspec** 到达，让任何客户端在每个打开的项目里强制覆盖
  或创建任意本地分支（不安全的 ref → 同样的 `{ ok: false, moved: false }` 空操作，永不抛出，
  因为该路径是 fire-and-forget）。它的结果还说明抓取是否**`moved`**了本地远程跟踪 ref
  （首次出现也算；即使在 fetch 报告失败之后也探测抓取后的 oid，因为 Git 可能在后来的失败/超时之前
  更新该 ref；在完全限定的 `refs/remotes/…` 上比较——即 fetch 实际更新的那个 ref——因此字面名为
  `origin/<b>` 的本地分支无法经 git 的 DWIM 顺序遮蔽该检查）：被移动的 ref *可能*改变兄弟工作区
  分支范围 diff 的含义（其 merge-base 可能移动），而它对 `watch` 模块不可见（写入落在项目仓库共享的
  `.git`，在每个被监听位置之外）——因此 `git.prefetch` 处理器用 `moved` 扇出宿主的无路径
  `fsChanged` 轻推（`host` 的 fsNudge 接缝；未受影响的重读是幂等空操作）。`moved` 是宿主内部的；
  线上响应仍是 `{ ok }`；
  **`readBlobAt(worktreePath, ref, path)`** → 文件在某 ref 处 UTF-8 **解码**后的内容，读取没有产出时
  为 `null`（diff 侧把它降级为 `""`；`reviews` 模块用它渲染基准侧锚点的文本片段）；
  **`readBlobBytesAt(worktreePath, ref, path)`** → 同一次读取但保持**字节精确**（`Uint8Array`）且
  同步，仅供 `reviews` 的快照轮使用；**`readBlobBytesAtAsync`** 是 `changes` 使用的有界请求路径原语，
  它的读取必须是完整的才能被哈希。两者都运行 `git cat-file blob`，只对 Git 显式的路径缺失诊断返回
  `null`，而对非 blob 对象或其他任何失败抛出；因此超时/启动失败永不能伪装成缺失。
  **`readBlobStreamAtAsync`** 是供宿主 `/blob` 路由使用的同一次读取，该路由不得把 blob 留在内存中：
  它只 await 前 `CONTENT_SNIFF_BYTES`（路由分类的 `head`），其余部分作为 `body` 流交出，该流重发头部
  然后随到达转发 `git cat-file` 的 stdout——缺失与失败由第一次读取观测到的退出决定，契约同为
  `null`/抛出，而*在*头部已交出之后的失败会让该流报错而不是关闭它，因此消费者看到的是一次中止的传输，
  而不是静默截断的 blob。可选的 `signal`（HTTP 请求的那个）可在任意时刻取消该读取——包括头部仍在
  await 时，在任何可取消的响应体存在之前——而取消会杀掉 `git` 子进程，而不是让它等完转发的时限。
  **`readBlobSizeAtAsync`** 在宿主准入一次不可变 blob 响应之前，通过 `git cat-file -s` 执行同样的
  有界、严格缺失即空转的读取；**`readPathModeAtAsync`** 用有界 `git ls-tree` 读取某路径的树模式，
  使 `changes` 能拒绝符号链接并恢复 Git 的可执行位，而无需自己推导树元数据。这些读取组成的每个
  `<ref>:<path>` 对象名都使用 `/`。Git 会把它那里的 Windows `\` 读作名字的一部分并报告文件缺失，
  因此在 win32 上路径的 `\` 变成 `/`；在 POSIX 上反斜杠是合法文件名字符并被保留（架构 Decision 22）；
  **`gitCommitPaths(workspaceId, message, paths)`** → `{ sha } | null`——为 TODO 变更集特性把
  **恰好 `paths`** 提交为一个提交（见 [[submodule-server-todos]]）：先暂存它们
  （`git add -A -- <paths>`，因此删除也作为一项暂存），然后
  `git commit --no-verify -- <paths>`（宿主的提交不得运行/失败用户的钩子；author/committer 保持用户
  的 git 配置——这是他们的分支），并返回新 sha。**只有被命名的路径**——绝不是“现在脏的一切”：
  调用方传入它已证明属于该条目的集合（且因为是它的过滤后 delta，它从不包含 `.thinkrail/`），
  因此调用方的 `gitStatus` 与本次调用之间出现的脏内容无法被扫入，用户其他已暂存的工作保持暂存而不会
  搭车。路径是**字面文件名，绝不是 pathspec**：每个消费路径的命令都运行 `--literal-pathspecs`，
  因此某个*名字*为 pathspec 魔法或 glob（`:(top)*`）的已跟踪文件无法扩展到已证明 delta 之外、
  也无法破坏精确路径保证或 `.thinkrail/` 排除。**索引跨失败被保留**：checkout 真实的索引**文件**
  （`rev-parse --git-path index`——在链接 worktree 中按 worktree 区分）在暂存之前逐字节快照，
  并在每条失败路径上写回，因此被跳过的提交让用户的暂存区完全如旧——*包括树往返会丢掉的仅索引状态*
  （`git add -N` 的 intent-to-add 条目没有树表示，因此 `write-tree`/`read-tree` 快照会静默取消暂存它）。
  暂存会成功但提交可能失败（未设置身份、不可用的签名 key），而一个尽力而为的特性不得让用户的下一次
  提交带上他们从未暂存的文件。带未合并条目的索引（在飞的冲突合并）原样退出不触碰；半合并的 worktree
  本来也没什么可自动提交的。对空路径集、当那些路径无可提交时
  （`git diff --cached --quiet -- <paths>`）或任何 git 失败都返回 `null`——调用方（`todos/artifacts`）
  把它当作“回退到路径列表制品”且永不令其抛出。它是唯一**写入**用户分支的 git 原语；调用方按工作区
  串行化它。**`gitHeadSha(workspaceId)`** → `string | null`——`rev-parse HEAD`（未出生 HEAD 时为
  `null`），在 `in_progress` 时记录进 todos 基线 sidecar。**`headShaAt(repoPath)`** 是给已持有
  工作区记录的调用方的同一次读取，因此它永不去重新解析一个可能已被其他写入者移除的 id。
  **`readCommitSubject(workspaceId, sha)`** → `string | null`——某提交的主体行（sha 畸形或不可解析
  时为 `null`），即 todos 模块已采纳提交解析器背后的同步读取（`base..HEAD` 中不属于任何 plan 条目的
  提交）。**`listCommitsSince(workspaceId, sinceSha)`** → `sinceSha..HEAD` 的
  `{ sha, subject }[]`，**最旧在前**（`git log --reverse`，上限 `COMMIT_LIST_MAX`），供 todos 模块的
  工作窗口提交采纳使用（[[submodule-server-todos]]）：某条目在 `in_progress` 期间由子 agent/用户落地的
  提交，因此 `done` 的条目会认领它们，而不是把它们泄漏到 `adoptedCommits`。`sinceSha` 是该条目自己的
  基线 head（`in_progress` 时的 `gitHeadSha`）；null/未出生 head 或非十六进制值产出 `[]`，
  而 sha 无论如何都被 `--end-of-options` 括住。超时/启动失败抛出；正常退出的不可读区间降级为 `[]`。
  **`resolveListedCommit(workspaceId, sha)`** → `string | null`——当且仅当 `sha` 在
  **`listCommits` 输出的同一有上限 `base..HEAD` 集合**中（同样的 `COMMIT_LIST_MAX` + 区间）时返回其
  **规范 OID**，否则 `null`。返回规范 sha 让已采纳提交的解析器能拒绝缩写/非规范 id，而共享上限保证它
  永不接受超出最新 `COMMIT_LIST_MAX` 的提交——超出者永远不会被输出为已采纳条目。与 `listCommits`
  保持同步。**`ignoredPaths(worktreePath, paths)`** → git 忽略的子集
  （`git -c core.quotePath=false check-ignore -- <paths>`，参数按 24k 字符分块以适配 Windows 命令行
  上限；已跟踪文件永不计数，而被忽略文件夹内的路径则计）。宿主用它把 `fs.readDir` 节点标为
  `gitignored`，使 Files 树能把它们变暗。它永不抛出：非仓库、失败或超时都报告空。必须由 git 做 C 引号
  的名字（`"`、`\` 或控制字符）永不报告，因为 `check-ignore -z` 需要 stdin，而有界 runner 不提供它。
- **公开接口（barrel）：** `git`、`gitAsync`、`gitAsyncBytes`、`gitAsyncStream`、
  `nonInteractiveGitEnv`、`remoteRefOid`、`remoteTrackingRef`、`gitStatus`、`gitUncommittedPaths`、
  `gitDiffFile`、`readBlobAt`、`readBlobBytesAt`、`readBlobBytesAtAsync`、`readBlobStreamAtAsync`、
  `readBlobSizeAtAsync`、`readPathModeAtAsync`、`readCommitSubject`、`commitsAheadOf`、`originUrls`、
  `countPushTargetDivergence`、`PushTargetStatus`、`gitCommitPaths`、`gitHeadSha`、`headShaAt`、
  `listCommits`、`listCommitsSince`、`resolveDiffRange`、`changedFileArgs`、`diffBaseRef`、
  `resolveCommitOid`、`DiffRange`、`isSafeRef`、`assertSafeRef`、`listBranches`、
  `resolveDefaultBranch`、`tryCurrentBranch`、`currentBranch`、`canonicalPath`、
  `resolveListedCommit`、`prefetchBranch`、`countPushDivergence`、`listRemotes`、`remoteNameOf`、
  `snapshotWorktree`、`ignoredPaths`。
- **允许的依赖：** `persistence`（工作区 + 项目查询）、`log`；`fs`（`resourceMeta`/`decodeText`——
  唯一的内容分类，因此 diff 侧的元数据无法与 `fs.readFile` 的相矛盾）；`contracts`
  （`Git*`/`BranchList`/`ResourceMeta` 类型）；`subprocess`（`runBounded`，两个异步 runner 背后的
  有界子进程）；`@thinkrail/shared/codedError`（为线上命名失败）；`@thinkrail/shared/spawn`
  （`spawnSyncCaptured` / `spawnSyncCapturedBytes`，带 `windowsHide` 的同步 runner）。
- **禁止：** `host`；兄弟特性模块。

## 必须做对（Get right）

- **范围只定义一次。** 任何必须知道“那个 diff”是什么的新读取都走 `resolveDiffRange`——永不写自己的
  `git diff <base>` 行——而任何对基准 ref 的读取都走 `diffBaseRef`，使 `diffBase ?? baseBranch`
  在整个代码库中恰好只存在一处。
- **提交范围校验该提交*存在*，而不是仍可从分支到达。** rebase 或 reset 可能把历史从一个选择脚下重写；
  对象仍在，展示它的 diff 比静默把用户重置到“All changes”*更有用*。哪些提交被*提供*是范围菜单
  （`listCommits`）的职责，而不是读取的——因此没有读取要为一次 `merge-base --is-ancestor` 付出代价。
- **失败的读取是错误，绝不是“无变更”。** `gitStatus`（及其 `--numstat` 轮）尊重退出码：跑不起来的
  diff 会抛出，因此面板保留它最后一份好列表并说明刷新失败，而不是渲染空的变更集。`workspaces` 模块的
  `diffStats` 从另一端遵循同一规则——它返回*无*统计（并记录原因），而不是编造 `+0 −0`。
  一个把脏 worktree 称作干净的评审界面是这个产品可能有的最糟失败。
- `gitStatus` 为用户自有（`kind: "default" | "external"`）工作区报告**实时**当前分支
  （它的分支会在带外移动——一次终端 `git checkout`——而持久化快照只在列表时自愈；Changes 头部不得滞后）。
- **网络 git 调用由我们限时，而不是靠劝它别等。** 一个远程要求 passphrase 的 `fetch` 会永远阻塞在
  `/dev/tty` 上，而唯一能结束那次等待的是*客户端*的 60 秒请求超时（`transport.ts` 的
  `DEFAULT_TIMEOUT_MS`，未对 `workspace.create` 覆盖）——它按构造不携带原因，因此用户得到的是
  `request "workspace.create" timed out`（issue #209）。因此 `gitAsync` 让子进程与 **55 秒预算**赛跑
  （`opts.timeoutMs` 覆盖；测试以 500ms 驱动它）：到期时它杀掉子进程并以
  `timed out after <actual>s — the remote never answered` resolve，`workspaces` 把它带进创建失败。
  **预算被刻意设到客户端 60 秒允许的最高值。** 任何低于客户端的预算都会把一次*慢但健康*、本来会完成的
  操作变成硬失败——大仓库在慢链路上的首次 fetch 就是真实案例。那个窗口**并未关闭，只是收窄**到约
  55–58 秒，且无法关闭：等于或高于客户端预算会把竞速交回本条正要去掉的、无因的 `request … timed out`。
  余下的约 5 秒是抛出要回传并仍胜过客户端所需的；若阻塞的循环超时过多，客户端的通用超时就会触发——
  即修复前的行为，属降级而非新失败。顺序本身**没有任何东西钉住**：`transport.ts` 的 60 秒与本处的
  55 秒是两个独立发布产物中手工维护的数字，靠约定而非构造正确。从彼此推导其中一个属于 `contracts`，
  且刻意超出本次变更的范围。**消息永不点名我们未观测到的原因。** 到期时我们所知的只是远程没有作答；
  未加载的 SSH key 被作为*一个*可能原因提供，且以远程使用 SSH 为条件，因为直接断言它把每次 `https://`
  停滞与每次慢链路 fetch 都误诊为认证问题。经过秒数下限为 1：`Math.round` 会把不足 500ms 的等待渲染为
  `0s`，读起来像 bug。**它是下界，不是上限。** `setTimeout` 在循环阻塞时无法触发，而*同步* runner 按
  设计阻塞它（`spawnSyncCaptured`），因此真实等待是预算加上兄弟 `git()` 调用占住循环的时间——在故意的
  风暴下实测为预算的 6 倍。因此消息报告**实际经过**的时间而不是配置的预算：用户读到的数字绝不是虚构。
  关闭该缺口意味着把 git 完全移出循环，而本工单不买这个。计时器被 `unref`：它存在的意义是*限定*一次
  等待，因此它自己绝不能成为宿主继续存活的原因——否则一次在飞的 fire-and-forget `prefetchBranch` 会把
  关停最多拖满整个预算。**`boundedStderr` 切中间，不切尾部。** git 自己的话总在最后，而切尾会把它们藏掉：
  `git.ts` 的 `exists on disk, but not in` 探测就坐在路径*之后*，丢掉它会把一次普通未命中变成
  `console.warn` 风暴。这是残留的、未修复的、且是任何定长裁剪固有的——一条长到足以把那句话推进被省略
  中段的消息（约 2000 字符的路径*加上*约 800 字符的 ref）仍会丢掉它，而移动头/尾分割点只是移动窗口。
  **等待在 git 退出时结束，而不是在 git 的管道结束时。** 那个区分是 `subprocess`（`runBounded`）的
  职责，而把它弄错就曾把这个特性反转：一个孙进程 `ssh`——`ControlMaster`、凭据缓存守护、企业包装器
  ——继承了 `stderr` 并比 git 活得更久，因此读到 EOF 永不返回，*即使 fetch 已经成功*。把该读取当作完成
  信号时，一次毫秒内完成的 fetch 被报告为满预算的认证失败。`gitExec.test.ts` 从两个方向验证了这一点：
  一个把持有 stderr 的子进程放到后台并退出 0 的 `core.sshCommand`，在远不到一秒内产出 git 自己的
  `Could not read from remote repository`，而修复前会产出满预算之后的停滞消息。
- **我们永不碰用户的 ssh 客户端——那件事曾被尝试并撤回。** 给其有效 ssh 命令（`GIT_SSH_COMMAND`，
  否则 `core.sshCommand`，否则 `ssh`）追加 `-o BatchMode=yes` 对常见配置能修好 #209，且无法做到安全。
  它的两个失败在原则上无法修复，而不仅仅未修复：`BatchMode` 会压制 `SSH_ASKPASS`，因此每个桌面
  passphrase 对话框*本来能用*的用户都失去它——对话框**就是**对人的等待，因此禁止等待就禁止了对话框；
  以及通过 `GIT_SSH_COMMAND`（一个*进程*范围的变量，优先级高于 `core.sshCommand`）投递*按仓库*的选项，
  会在该环境下触及的每个其他仓库中反转优先级——一个带自己 deploy key 的子模块会拿到父仓库的客户端。
  另外四个缺陷来自不得不重写一个我们必须先按 git 方式解析（`split_cmdline`）的命令——带引号的 plink
  路径绕过非 OpenSSH 的退出分支、企业包装器拒绝该 `-o`、`GIT_SSH` 以未修复挂起的方式退出，以及每次
  `git()` 一次 `spawnSync` 配置读取。上面的预算一个都不需要：它对哪个 ssh 客户端、哪个变体、哪个包装器、
  以及是否存在 askpass 全都无感。
- **env 叠加在 `process.env` 之上，永不替换它。** `boot` 的 `resolveShellEnv()` 通过*变更*
  `process.env` 修复 `PATH`/`LANG`；一个裸 env 对象会丢掉该修复以及 `SSH_AUTH_SOCK`，而一个活的 agent
  socket 正是让确实加载了 key 的用户完全无提示地抓取的原因。
