# watch —— 工作树变更通知器

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

UI 实时刷新背后的文件系统变更通知器：每个被监视的工作区工作树一个递归 watcher（Bun 原生的
`fs.watch(root, { recursive: true })` —— 不依赖 watcher 库），把事件合并为去抖的
**`workspace.fsChanged`** 发布（`WorkspaceFsChangedPayload`：
`{ workspaceId, paths, truncated, skillChange }`）。该帧是一个**失效信号，不是数据** —— 客户端经既有
读取方法（`fs.*` / `git.*` / `spec.graph`）重新读取，因此读取保持唯一权威来源，重复/重放的帧是无害的
（多一次重新获取，永不会是错误状态）。选择它而非逐路径的客户端侧树补丁（那会让客户端成为第二个权威
来源）也非可见面板轮询（迟滞，在 Tailscale 上浪费）。

## 边界（Boundary）

- **拥有：** watcher 注册表 + 其生命周期：`ensureWatch(workspaceId)`（幂等且**自愈**；返回在线 watcher
  的共享就绪 promise；由 `host` 在一次工作区读取或 `workspace.watchReady` 预检落定时惰性启动 ——
  那次读取就是「有客户端在看」的信号；就绪报告 watcher 是否已被知晓为就绪）、
  `stopWatch(workspaceId)`（在 `workspace.remove` 的快速路径中调用）、`stopAllWatches()`
  （在 `server.stop()` 中调用）；忽略过滤器（任何路径段为 `.git` 或 `node_modules`，以及
  `.DS_Store`）；逐工作区合并（去重的相对路径，在 300ms 静默 / 1s 最大等待后冲刷，上限 100 个路径 →
  `truncated: true` = 一个通用通配 —— ≤ ~1 帧/秒 的界由 `live-refresh.spec.ts` 中的 **e2e 抖动金丝雀**
  钉住：~3s 内 ~200 次写入必须以 ≤ 8 帧到达客户端，同时一场风暴中途的 `/health` 往返仍保持快速）。
  技能相关性在**该上限之前且独立于它**地累积为
  `skillChange: "none" | "detected" | "unknown"`：`host` 注入 `agent` 的项目技能路径谓词；每个具体事件
  都贡献 `detected`/`none`，即使它的路径无法被保留，而 null 文件名贡献 `unknown`（`detected` 胜出）。
  一个已在保留集合中的重复项不会让该批次变成 truncated。因此一次 100+ 文件的构建无法冒充技能编辑，
  而一个在上限之后到来的技能路径仍被检测到。**启动信号** —— 一个新 watcher 在平台流完成注册窗口
  （~750ms）之后发布一次合成的**truncated / 技能未知通配** —— 因为落进该窗口的写入可能永久丢失其事件。
  它的就绪 promise 只在该发布之后才结算。web 的资源加载流程（会话创建、资源重载；转录读取发送预检
  但不等待它）在捕获其加载起点新鲜度 tick **之前** await 类型化的 `workspace.watchReady` 宿主预检，
  因此启动期的不确定性被吸收进新会话的基线，而不是被虚假地标记为陈旧；就绪之后的真实编辑仍比该基线
  更新。除非 watcher 已被知晓为就绪，结果会告诉 web 在其 socket 随广播一同死掉而请求响应存活了重放时
  —— 或在启动在发布之前失败时 —— 折入一个重复的 `skillChange: "unknown"` 回退。普通工作区读取不
  await 就绪，也永不因此获得启动延迟。停止一个 watcher 会结算其挂起的就绪状态并取消该信号，因此调用方
  不会挂住，被拆卸的 watcher 也无法发布。一条失效信号仍然是幂等的，因此回退可能的重复只是一次廉价的
  空操作重取。
- **预热层（全局有界）：** `ensureWatch(workspaceId, { prewarm: true })` —— 经
  `workspace.watchReady` 的可选 `prewarm` 标志到达，即 web 的预选预热 —— 启动同一个 watcher，但把它的
  条目标记为**仅预热**。仅预热的条目住在一个全局池里，上限为 **8**：创建超出上限的一个会经
  `stopWatch` 逐出最近最少预热的仅预热条目（含其保守的就绪结算），因此在多个项目间点击会复用一个有界
  池，而不是在一个宿主生命期内累积 watcher。任何**真实**调用都会把该条目永久提升出该层 —— 一个被真实
  工作区读取或技能加载预检激活的 watcher 永不被逐出 —— 而一次对在线仅预热条目的预热命中只刷新其逐出
  新近度，且一次预热永不能把真实 watcher 降级（包括跨越一次 inode 变化的重建）。逐出下的正确性依附于
  既有恢复路径：下一次真实调用重建 watcher，其新鲜保守的启动信号覆盖盲窗。
- **仓库元数据信号（第二个接缝）：** 一次 git 元数据写入是*元数据，不是内容*，因此它永不会成为一个
  `fsChanged` 路径（`.git` 黑障成立 —— plumbing 风暴不得变成帧）。它改为触发一个单独去抖（300ms）、
  **无路径**的 `setRepoMetaPublisher(workspaceId)` 信号。对于一次让工作树字节完全不变的变更，这是唯一
  的信号：`git switch -c <new-branch>` 在 git 目录之外什么都不写，而一次 `git commit` 移动 `HEAD` 却
  不触碰任何工作树文件。它刻意**不**按具体路径匹配（`.git/HEAD`、`.git/logs/HEAD`、…）：平台流会合并
  并每轮突发只报告*一个*代表性路径，因此浮现的是哪个 git 内部路径并不可靠。一个通配事件（null 文件名）
  也会触发它。两个来源喂给这一个信号：
  - 由递归**根** watcher 看到的、以 `.git` 为前缀的事件 —— 覆盖**仓库根**工作区，其 `.git` 目录位于
    被监视树内；
  - 当工作树的 git 目录**位于根之外**时，对该目录的第二个**非递归** watcher（仓库根内树的 `.git` 目录
    已被上面覆盖，因此它永不被监视两次）：对一个*链接工作树*（本应用创建的每个工作区）而言，`.git`
    是一个*文件*（`gitdir: <path>`），指向 `<repo>/.git/worktrees/<name>`，即在**被监视根之外**，因此
    在该工作树终端里做的一次提交否则根本产生不了信号。用普通 fs 解析（stat + 解析 gitfile 那行），永不
    shell 出去 —— 本模块没有 `git` 兄弟边。非递归是因为只有该目录顶层持有会移动的 ref（`HEAD`、
    `index`、`ORIG_HEAD`），而 `objects/`/`logs/` 是纯风暴；缺失/不可读的 git 目录（非 git 文件夹）或
    启动失败会静默降级。

  `host` 把该信号扇出到两个收敛点：`refreshUserOwnedWorkspace`（用户拥有工作区的文件夹真相分支标签）
  **以及**一个无路径、技能中立的 `fsChanged` 帧（`paths: []`、`truncated: false`、
  `skillChange: "none"`），好让客户端那些由 git 派生的读取重新读取 —— `git.status` 与一个打开的
  `uncommitted` 作用域 diff 标签页都相对 `HEAD`，否则会一直把一次已提交的变更报告为未提交，直到下一次
  文件编辑。
- **定时器驱动的投递是被围住的：** 合并器冲刷、启动信号与仓库元数据信号都跑在本模块自己的定时器上，
  因此订阅方抛错只被 warn 记录并吞掉，而不会变成一个致命的 `uncaughtException` 把宿主带下去（一次在
  信号中途删除 `workspaces.json` 的 e2e 重置曾正是这样做的）。后续事件继续流动。watch 已启动信号是
  例外：它运行在下文描述的受守卫注册区域内。
- **组装接缝：** 从不导入 `host` 或 `agent` —— `host` 在接线时注入发布回调与 `agent` 的纯项目技能路径
  分类器（发布者-tee 模式）。缺失分类器会把具体事件降级为 `skillChange: "unknown"`，永不是假干净。
- **每次读取自愈（带外工作树抖动是常态 —— e2e 重置、终端里 `rm -rf`）：** 每次 `ensureWatch` 都重新
  stat 根，并在 **inode 变化时重建 watcher**（一个删除+重建的路径会让旧流静默跟随一个死 inode）、
  **回收其工作区记录已不存在的僵尸 watcher**（一个复活的基于路径的流会为一个被遗忘的 id 继续发布），
  并**在下一次读取时重试一次失败的启动**（没有粘性失败标记）。一个中途出错（ENOSPC、根被删除）的
  watcher 被 warn 记录并丢弃 —— 面板回退到按需读取，直到稍后的读取重建它。没有空闲停止（受实际访问
  过的工作区加上有上限的预热层限界）。
- **watch 已启动信号（第三个接缝）：** 在 watcher 条目被创建的那一刻（真实或预热），在任何事件能够
  到达仓库元数据信号之前，`setWatchStartedPublisher(workspaceId)` 触发一次 —— 宿主用它取得工作树的
  HEAD 基线，因此一个 watcher 的第一个 git 元数据事件就是对照一个已知 sha 判断的，而不是被花在播种上。
  一个被重建的 watcher（inode 变化、预热重新准入）会再次触发它；消费者与自己保留的基线比较，因此在
  未被监视期间做的一次 HEAD 移动仍记录为活动，而未变化的 HEAD 是空操作。它运行在注册的受守卫区域内，
  那里的抛出 —— 来自它，或来自 `entries.set` 之后的任何东西 —— 会拆卸这个新条目（`stopWatch`），而
  不是留下一个就绪永不结算的已注册 watcher。
- **公开接口（barrel）：** `ensureWatch`、`stopWatch`、`stopAllWatches`、`setWatchPublisher`、
  `setRepoMetaPublisher`、`setWatchStartedPublisher`、`setSkillPathClassifier`。
- **允许的依赖：** `persistence`（工作区查找）、`log`；`contracts`（载荷类型）；Bun/Node。
- **禁止：** `host`；兄弟特性；任何 pi 包。
