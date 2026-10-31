# workspaces —— git worktree

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

工作区是数据目录下位于自己分支上的一个 `git worktree`——文件/git/终端/聊天的锚点。它的**显示 `name`
与它的 git `branch` 解耦**：`name` 是人类可读的标签（Title Case、含空格），`branch` 是从它派生出的
kebab slug——它们曾被要求相等，对自动的 `workspace-N` 占位名也仍然重合，但一个具名工作区两者各自
不同地携带。

两类是**用户自有**的——ThinkRail 使用它们的 cwd，但从不重命名或回收它们。每个项目携带**恰好一个
内建的默认工作区**（`kind: "default"`），其 `worktreePath` 就是项目文件夹本身（git 的*主工作树*）
——“就在我的项目文件夹里干活”的锚点，惰性确保，**不可移除也不可重命名**。用户显式附加的任何
**既有工作树**都原地记录为 `kind: "external"`——位于数据目录之外，在此从不创建或改动。

## 边界（Boundary）

- **拥有：** `listExistingWorktrees(projectId)`（**异步**——解析经 `gitAsync` 读取的
  `git worktree list --porcelain -z`，与 `git/SPEC.md` 为 `gitStatus`/`listCommits` 陈述的同一条
  请求路径读取规则：这是 Existing-Worktree 对话框的填充调用，因此一个慢的/庞大的注册表绝不能冻结
  宿主的事件循环；列出的每条路径都被解析，因为 git 在 Windows 上打印 `C:/…`，而该行的路径会成为
  存储的 `worktreePath`（架构决策 22）；丢弃项目文件夹、可修剪的注册条目，以及每个已在 ThinkRail
  中表示的路径；有分支支撑的行是 `available`，detached-HEAD 的是 `detached`）、
  `openExistingWorktree(projectId, path)`（**异步**，因同一次工作树列表读取；在变更入口对 Git
  注册表重新校验，比较规范化后的路径；同项目重试是幂等的，跨项目复用 cwd 会被拒绝；持久化 + 发出
  `created`，带 `kind: "external"`、一个目录 basename 作显示名、`renamed: true`，以及仓库默认分支
  作为其初始评审目标——**不做任何 Git 或 checkout 改动**）、`createWorkspace`（**异步**；给定
  `baseRef` 时从中切出分支——用 `worktree add -b` 建分支，绝不是 detached 的远程 checkout，并且
  **`--no-track`**：否则一个远程跟踪的基点会变成新分支的上游（git 的 `autoSetupMerge` 默认），把
  工作区终端的 `git push`/`git pull` 指向*基点*分支——工作区分支的上游应由用户在首次推送时设置，
  绝不是我们；否则从仓库 `HEAD` 切出；**远程 ref 的新鲜度在关键路径之外预取**——New-Workspace
  对话框在后台 `git.prefetch` 基点，因此 create 只在本地远程跟踪 ref 完全缺失时才作为回退执行
  `git fetch`——该回退经 `gitAsync` 运行（网络绝不能阻塞事件循环；限制等待时长并截断 stderr 是
  runner 的职责，不是这个调用点的）并在 `--` 之后传入分支，而且**一次未落地该 ref 的回退 fetch 会
  让 create 失败**，报出该 ref 名并携带 git 自己的 stderr——包括远程干脆没应答时，用 runner 自己的
  `timed out after <n>s …` 顶替它。配置的 remote 是所选 ref 的最长匹配前缀，因为 Git 允许像
  `team/upstream` 这样的名字。守卫是 **ref**，而不是退出码：一个收窄的 refspec（`remote add -t`、
  `--single-branch`，或编辑过的 `remote.<name>.fetch`）可以让 `git fetch <name> -- <branch>` 以
  0 退出却不创建期望的 `refs/remotes/<base>`，而以 `!fetched.ok` 为键恰好让这种情况落到这条要点
  声称已移除的派生错误 `fatal: invalid reference`。**探针完整命名 `refs/remotes/<base>`**，完全像
  `git` 的 `prefetchBranch` 那样——像 `origin/<b>` 这样的简写还可能解析到一个字面名为 `origin/main`
  的本地分支；这场竞态的两侧必须询问同一个 ref。**`worktree add` 拿到的也是同一个完整名**，而不是
  简写：检查一个 ref 却 checkout 另一个，会把歧义留给 Git 去选择或拒绝错误的对象。`baseBranch` 仍
  记录用户可见的 `<remote>/<branch>`；只有交给 Git 的修订被限定。**消息按同一区分分流**：*失败*的
  fetch 携带 git 的 stderr，而*成功*但未落地该 ref 的 fetch 改为点名 refspec——它的 stderr 是一份
  进度日志（`* branch release -> FETCH_HEAD`），所以把它贴在 “Could not fetch” 之后，会让用户看到
  一次成功的 fetch 成为 fetch 失败的原因。它是知道基点*为什么*缺失的那一步，而丢弃它的结果会让
  `worktree add` 去报派生出来的 `fatal: invalid reference: <remote>/<branch>`——或者，当 fetch 根本
  没有返回时，什么都不报，直到客户端请求超时（issue #209）。抛出前 ref 会**用 `rev-parse` 重新
  检查**：一次并发的 `git.prefetch` 可能在 fetch 在途时已把它落地，而一次本会成功的 create 不该输掉
  这场竞态；`Workspace.baseBranch` 记录**创建来源**——工作树是从哪个 ref 切出的，它此后永不变动
  （diff 所*对照*的是另一个可重新指向的 `diffBase`；见 `setWorkspaceDiffBase`）；**分支名对 ref
  *和*工作树目录唯一**——归档会把分支留在原处，而重命名会释放一个分支名、其工作树目录却仍被占用，
  因此候选名跳过两者；路径 `dataDir/worktrees/<project-slug>/<branch>`；**为新工作区播种临时的逐
  工作区 scratch 目录**（`WORKSPACE_CONTEXT_DIR`，带一个自我忽略的 `*` `.gitignore`——零 git 足迹），
  它是临时文档（任务规格 / 工作文件）的家，这些文档不进 git 但仍可被规格工具扫描（路径约定住在
  `@thinkrail/shared/paths`；见 [[submodule-workflow-skills]] 的工件规则）；**用户提供的名字就是
  显示名**（经 `toDisplayName` 保留大小写 + 标点；分支由它派生）并在创建时**设置
  `renamed: true`**——用户已经选定，所以自动命名永不触碰它；自动的 `workspace-N` 则让它未设置，
  此时 `name === branch`；**在等待过的回退 fetch 之后重读注册表**再追加——等待前的快照那时已过期，
  保存它会覆盖一次并发 list 的 Default 确保（与 `renameWorkspace` 在其 git 子进程之后重新加载同一
  纪律））、`renameWorkspace`（**同步**；设置经清洗的、保留大小写与文字的显示 `name`，并总是设置
  `renamed: true`，因此自动命名再也不碰它。**`opts.branch`** 是 agent 从 `set_title` 得来的英文
  slug：存在时，分支变成该 slug 的 ASCII kebab 形式，钳制在 60，对 refs + 工作树目录去重，并用
  `git branch -m` 移动，而**工作树目录永不移动**（pi 以那个确切的 cwd 为会话和终端/标签页的键）；
  一个清洗后为空的 slug 保持当前分支。移动分支的路径会重新指向那些 `baseBranch` 或 `diffBase` 曾
  指向旧分支的兄弟记录，在 Git 子进程之后重新加载注册表，使一次并发移除不会被复活，保存一次，并为
  每条变更记录发出 `updated`。没有 `opts.branch` 时——即手动线上方法——只有显示标签变化：`branch`、
  `worktreePath`、`baseBranch` 与 `diffBase` 保持不变，不运行任何 Git 变更，并发出一次全快照
  `updated`。**`name` 与 `branch` 在此类重命名后刻意不同**——name 只用于显示且可以重复；其下显示的
  分支负责消歧（见 [[submodule-web-panels]]）。未知/默认/外部 id 和非法名称仍会抛出；由调用方决定
  呈现）、`listWorkspaces(projectId, { includeDiffStats? })`（**异步**；在 Default 确保 + 用户自有
  文件夹真值对账之后的完整权威成员/顺序——那个同步序言的 load→mutate→save 在第一个 await 之前
  完成；diff-stat 徽章随后**经 `gitAsync` 并行**解析，在事件循环之外，并且**在 await 之后重读成员
  关系**（与写入者相同的过期快照纪律）：一个在徽章解析期间创建的工作区必须出现在这次响应中，因为它
  的 `created` 推送可能先于这次回复到达某个客户端并为其播种列表，而推送到未列出项目是刻意不折叠的
  ——一行在途记录会不带统计信息地发出，直到下一次 list。缓存的统计只在其**定义 diff 的字段
  （`worktreePath`、`baseBranch`、`diffBase`）仍与计算它所依据的快照匹配**时才附到新行上——一次在
  徽章解析期间落地的 `setDiffBase` 同样会丢掉该行的徽章，绝不把新 ref 与旧 ref 的合计配对。
  `workspaceDiffKey` 是这个身份的单一公开定义，与 `reviews` 共享，因此跨越同一竞态创建的评审会对
  新目标重试，而不是钉住旧的。为兼容性，diff 统计默认**开启**，而 `includeDiffStats: false` 跳过
  逐工作区扇出以支持冷导航——共享宿主上的自动重新加载不该对每个工作树做 diff）、
  `listWorkspaceRecords`（原始注册表记录，不做 Default 确保、文件夹真值对账或逐工作区 git
  diffStats——用于历史作用域映射这类不得阻塞在 git 派生上的内部只读路径）及其不含项目的兄弟
  `listAllWorkspaceRecords`（每条记录，用于必须跨越工作区而不必知道哪些项目打开的宿主读取——即全
  工作区的 `session.stateList` 读取）、`workspaceDiffStats`、**`setWorkspaceSubagentsOverride(id,
  override)`**——持久化 `"on"` / `"off"`，或对 `null`（继承）删除 `Workspace.subagentsOverride`，
  然后发出权威的全量 `workspace.updated` 快照。它校验这个封闭取值，但从不读取全局默认值或解析生效
  的 agent 策略；宿主跨兄弟边界组合那件事；**`setWorkspaceSubagentMaxConcurrent(id, value | null)`**
  ——对数值 `Workspace.subagentMaxConcurrentOverride` 的同一契约（一个
  `SUBAGENT_MAX_CONCURRENT` 整数，`null` 删除）；**`setWorkspaceDiffBase(id, ref | null)`**——重新
  指向该工作区 diff 所对照的 ref（`Workspace.diffBase`），`null`（或创建基点本身，那会是一次冗余
  覆盖）清除它；持久化 + **广播更新后的记录**，让每个客户端在推送上收敛，绝不乐观（完全仿照
  `setWorkspaceSkillOverride`）。两道 **ref 门**——这一道和 `createWorkspace` 的 `baseRef`——在 ref
  能到达 git 参数之前，先经 `git` 模块的 `assertSafeRef` 校验其*形状*（一个选项形状的分支可从任何
  不可信仓库到达；见 [[submodule-server-git]]）。`createWorkspace` **无条件地**校验**解析后的**基点，
  而不仅是客户端提供的那个：没选基点时它来自 `rev-parse --abbrev-ref HEAD`，即来自仓库，而一个
  不可信的 checkout 可能检出选项形状的分支（`git branch` 拒绝这样的名字，`symbolic-ref` 不拒绝）
  ——同一道门的两半，由一次检查关闭。**两个基点含义刻意是两个字段：** `baseBranch` = 分支来自哪里
  （对一个用户自有工作区，其来源不是我们能声称的：仓库默认分支作为其*初始*评审目标，永不标注
  `from`），`diffBase` = 其评审所对照的对象；把它们合并会让一个重新指向的目标对来源说谎
  （`branch · from baseBranch` 回执）。每一次对“基点”的读取都经 `git` 模块解析，绝不内联——
  `diffStats` 组合 git 模块的**分支作用域范围**（`resolveDiffRange` +
  `changedFileArgs(…, "--shortstat")`），因此工作区聚合恰好度量 Changes 面板所示的内容（包含
  merge-base 语义——基点上的上游提交永不使其膨胀），并像本应用传递的每个其他修订一样，被
  `--end-of-options` … `--` 括起来到达 git。当 git 无法作答时，`diffStats` 产出**完全没有统计**
  （记录日志），而不是伪造一个 `+0 −0`——一次失败的读取绝不能把脏工作树画成干净；
  `Workspace.diffStats` 字段直接缺失，而 `workspaceDiffStats` 拒绝、`getWorkspace`（按 id 查找，
  对未知抛出——锚定聊天会话的 cwd），以及**归档**原语，切分使快速的记录删除与缓慢的 git 回收可
  分离（宿主在请求关键路径之外执行归档）：`forgetWorkspace(id)`（丢弃持久化记录，返回被移除的记录
  或 `null`——立即从 `listWorkspaces` 消失）、`reclaimWorktree(ws)`（慢的那一半——
  `git worktree remove --force`，保留分支；加固：git 失败时用 rm + `prune`；**同时拒绝两种用户自有
  类型，并作为纵深防御拒绝任何 `worktreePath` 解析到项目文件夹的记录**——rm 回退绝不能看到用户的
  仓库或一个附加的 checkout，无论一条损坏/手改的记录是怎么进去的）、以及 `removeWorkspace(id)`
  （两者的同步组合，留给想要一次调用完成整个归档的调用方/测试）。
- **默认工作区（`kind: "default"`）：** 每个项目恰好一个。`listWorkspaces` **确保**它——按
  `projectId`+`kind` 查找或创建（id 是普通的 `randomUUID`；`kind` 字段才是标记，绝不是 id 约定），
  在带外状态变动曾产出两个时**防御性地合并重复**，并把它**钉在首位**返回。列表内的确保是一个
  **刻意的 CQS 例外**：查询执行一次注册表写入 + 生命周期发出，从而*任何*调用方都能自愈带外状态
  变动（回填在该特性之前打开的项目；e2e 重置会在运行中重写 `workspaces.json`）——这次写入针对的是
  数据目录下的应用注册表，绝不写进用户的仓库。没有 `git worktree add`（该文件夹本来就是主工作树），
  并且在确保时**不播种 scratch 目录**——仅仅列出项目绝不能写进用户的仓库（见
  `ensureWorkspaceScratchDir`）。字段是文件夹真值，漂移时在列表时刷新——**发出 `updated`**，让每个
  客户端的侧栏收敛，而不是在一次终端里的 `git checkout` 之后让某个标签页保持过期（
  `renameWorkspace` 为相同字段使用同一通道；store 的合并不触发重新列表，因此没有反馈环）：
  `branch` = 该文件夹当前的 HEAD（`symbolic-ref --short`，未出生安全；detached → 字面量 `HEAD`），
  `baseBranch` = 经 `git` 的 `resolveDefaultBranch` 得到的仓库默认分支（未出生安全——它最后的回退是
  `currentBranch`，因此字面量 `"HEAD"` 永不被持久化）——于是 Default 的 Changes 像任何工作区一样
  度量，当该文件夹就坐在默认分支上时退化为未提交的工作。漂移**不只是**列表时的发现：
  `refreshUserOwnedWorkspace(workspaceId)` 是**不带** diff-stat 列表的同一重新同步（外部工作区只
  重新同步它的 `branch`，而一个不可读的 checkout 绝不被持久化为伪造的 detached `HEAD`；未知 id /
  托管工作区 / 无漂移 → 不保存、不发出），宿主把它接线到 `watch` 的**仓库元数据轻推**
  （由宿主中介，`watch` 没有 `workspaces` 边——见 [[submodule-server-watch]]）。因此 Default 工作区
  终端里的一次 `git switch` 会让侧栏、顶栏和空回执实时收敛，而不是把它们留在旧分支上直到手动重载
  项目——包括一次让工作树字节不变（`git switch -c`）的切换，它在 `.git` 之外什么都没写
  （`gitStatus` 出于同一原因实时读取其头部所在分支——见 [[submodule-server-git]]）。首次实体化发出
  `created`（对 store 幂等）；无漂移的 list 什么都不发出。每一次确保发出——`created`、`updated`、
  合并的 `removed`——都发生在保存**之后**，与该模块在其他所有地方 persist-then-publish 的顺序一致。
  **不可移除 + 不可重命名，在此强制执行，而不只是藏在 UI 里：** `forgetWorkspace` 与
  `renameWorkspace` 对 `kind: "default"` **抛出**——forget 会把项目文件夹交给归档拆除的 `rm -rf`
  回退，rename 会对用户真实的分支执行 `git branch -m`；该记录携带 `renamed: true`，因此
  `set_title` 会像背带裤一样保持距离。
- **已确定的生命周期事实（宿主自有，绝不是分区本身）。** 记录携带 `lastActiveAt`、
  `settledOverride` / `settledAt`，以及 [[module-contracts]] 描述的 `review` 快照；哪些行是活跃的、
  哪些放到架子上，是*客户端*基于这些事实加会话状态做的推导，因此本模块从不存储一个可能与规则相
  冲突的 “settled” 布尔值。写入者：`recordWorkspaceActivity(id, at = now)`——盖章 `lastActiveAt`，
  **清除任何 `settledOverride`**（一次显式停放或钉住是关于一个安静工作区的陈述；其中真实的工作会
  取代两者），并**合并**写入到每个工作区每秒最多一次，使流式 agent 不会逐 token 重写
  `workspaces.json`——只有当被跳过的那次盖章不可能改变分区时才允许合并：覆盖的清除永不被合并掉，
  一条当前被其 merged/closed 评审所结算的行（它先前的盖章不比该评审的 `changedAt` 更新）上的盖章
  也不会。每次观察还会推进一个内存高水位标记：如果工作在存储的评审仍打开时落地，而稍后一次刷新
  首次揭示它其实已经合并，那么评审写入者会在同一次保存中冲刷那次合并的盖章，因此得知合并不可能把
  更新的工作放到架子上；`backfillWorkspaceActivity(id, at)`——仅当记录没有盖章时设置它，且不触碰
  任何覆盖（为早于该字段的记录准备的升级路径）；`recordWorkspaceHead(id)`——读一次记录（两次读取
  之间的一次带外 `workspaces.json` 重写绝不能把一个已移除的工作区变成抛出）和该工作树的 `HEAD`
  sha，并在它与本宿主生命周期内最后观察到的 sha 不同时调用 `recordWorkspaceActivity`（一次 HEAD
  移动是 commit/pull/checkout 的代理；裸 git-dir watcher 也会在索引刷新时触发，这就是信号是 sha
  而不是事件的原因），而 `seedWorkspaceHead(id)` **每当工作树 watcher 启动时**观察 HEAD：宿主生命
  周期内的第一次观察负责播种，而 watcher 重建时则与保留的基线比较，并计入 watcher 缺席期间所做的
  工作。`settleWorkspace(id)` / `unsettleWorkspace(id)`——设置 `"settled"` / `"active"` 与
  `settledAt`，**对 `kind: "default"` 抛出**（项目文件夹永不被搁置）；`setWorkspaceReview(id,
  review | null, branch)`——持久化该快照，且只在它确实变化时发出 `updated`，因此在什么都没动时
  周期性的 provider 刷新是静默的，并且**仅当记录仍在那次查找所针对的分支上时**——一个 provider 的
  回答是关于一个分支的事实，而外部工作树可能在一次查找所需的几秒内被切换；出于同一原因，每一次
  持久化的分支迁移都会丢弃 `review` 快照——既包括用户自有的文件夹真值刷新，也包括托管工作区那
  一次性的分支重命名——因为一个比其分支活得更久的已合并快照会搁置新分支，而且由于它已合并，永不
  会被被动刷新。每个写入者都发出全量快照 `updated`。`createWorkspace` 与 `openExistingWorktree`
  在创建时盖章 `lastActiveAt`，因此一个全新的行按构造就是活跃的。**迁移：** 宿主为没有盖章的记录
  回填，来源依次为最新聊天的文件时间，否则该工作树 `.git` gitfile 的 mtime（≈ `worktree add`
  时间），否则当前时间——因此在升级之后，一个休眠的积压会自行结算，而任何有近期聊天或一个打开 PR
  的东西保持活跃。
- **初始终端供给是一次持久的宿主握手。** 每个由 `createWorkspace`、`openExistingWorktree` 或
  Default 确保首次持久化的工作区记录都携带可选的布尔字面量 `initialTerminalPending: true`。
  `host` 幂等地保留那个确定性的、无进程的终端标签页，然后调用
  `completeInitialTerminalReservation(id)`，它清除标记、持久化并发布更新后的工作区。保留失败则让它
  保持待定，等待宿主的启动/创建恢复趟；成功则在任何前端摆放之前清除。只有由当前代码创建的记录会
  收到这个标记；没有它的记录被视为已完成且永不回填，因此一次升级无法复活用户先前关闭的默认终端。
  不涉及任何布局修订。
- **`ensureWorkspaceScratchDir(ws)`**——幂等地播种被 gitignore 的 `WORKSPACE_CONTEXT_DIR` scratch
  目录（mkdir + 自我忽略的 `*` `.gitignore`）；宿主在**每个工作区的会话创建**时调用它，因此 Default
  工作区只在聊天真正在那里启动时才写入用户的仓库（工作树创建仍在创建时急切播种；这同时会自愈一个
  scratch 目录已被删除的工作树）。加固，因为在 Default 工作区中它是针对**仓库受控内容**运行的：它
  **在工作区根缺失时抛出**（一个被外部删除的工作树必须让会话响亮失败，而不是被复活成一个空的
  非 git 目录）；它**拒绝符号链接形式的路径组件**（`lstat`，绝不跟随——一个恶意 checkout 无法把
  播种重定向到工作区之外）；而 `.gitignore` 写入是**排他创建（`wx`）——仅在文件缺失时**：在
  Default 工作区里那条路径位于用户自己的仓库之内，一个已存在（可能被跟踪、可能被定制）的文件是
  他们要保留的。叶子节点先做 `lstat` 检查，因此任何已存在的条目——包括一个（可能悬空的）符号链接
  ——都算作存在：POSIX 的 `O_EXCL` 拒绝跟随符号链接，但 Windows 的排他创建（`CREATE_NEW`）会跟随
  重解析点，并可能写到 checkout 之外。播种只填补空缺，从不覆盖（由回归测试钉住，符号链接那一个
  在宿主可以创建符号链接的地方都会跑）。
- **生命周期事件：** 每一次成员变更——`createWorkspace`（`created`）、`renameWorkspace`
  （`updated`，agent 的 `set_title` 也算，因为它走这条路径）、`forgetWorkspace`（`removed`）——都
  通过一个**注入的发布者**（`setWorkspacePublisher`，与 `terminal`/`agent`/`auth` 相同的反转；
  单元测试 / e2e 重置中为 `null` → 静默空操作）发出一个 `WorkspaceLifecycleEvent`。本模块对 WS
  通道保持无知：它发出一个领域事件（`created`/`updated` 携带记录，`removed` 携带
  `{ projectId, id }`），由宿主把 `kind` 映射到 `workspace.*` 通道。这使本模块成为**工作区生命
  周期推送的唯一来源**（命名处理器从不推送——重命名自行发布），因此注册表成员关系在每个客户端
  之间保持为共享的领域状态（架构 #9）。
- **公开接口（barrel）：** `createWorkspace`、`listExistingWorktrees`、`openExistingWorktree`、
  `listWorkspaces`、`listWorkspaceRecords`、`listAllWorkspaceRecords`、`forgetWorkspace`、
  `reclaimWorktree`、`removeWorkspace`、`workspaceDiffStats`、`workspaceDiffKey`、`getWorkspace`、
  `renameWorkspace`、`refreshUserOwnedWorkspace`、`completeInitialTerminalReservation`、
  `ensureWorkspaceScratchDir`、`setWorkspacePublisher`、`WorkspaceLifecycleEvent`、
  `setWorkspaceDiffBase`、`setWorkspaceSkillOverride`、`setWorkspaceSubagentsOverride`、
  `setWorkspaceSubagentMaxConcurrent`、`recordWorkspaceActivity`、`backfillWorkspaceActivity`、
  `recordWorkspaceHead`、`seedWorkspaceHead`、`settleWorkspace`、`unsettleWorkspace`、
  `setWorkspaceReview`。
- **允许的依赖：** `projects`（仓库查找）、`git`（runner）、`persistence`、`log`；`contracts`；
  `@thinkrail/shared/paths`（scratch 目录路径约定）；Node。
- **禁止：** `host`；伸进另一个特性的内部（用它的 barrel）。
