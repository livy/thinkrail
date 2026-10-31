# projects —— git 仓库作为项目

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

维护一份已知 git 仓库的稳定注册表，暴露它的打开与最近视图，并在不破坏其工作区身份的前提下打开/关闭
一个项目。对一个还不是仓库的文件夹，对它分类并引导其成为仓库，从而可被打开。

## 边界（Boundary）

- **拥有：** 在检查或变更任何外部项目文件夹输入之前，先把它对**宿主文件系统**解析：宿主绝对路径通过，
  精确的 `~` 与 `~/…`（Windows 上还有 `~\…`）针对宿主账号展开，其他所有相对路径都被拒绝，而不是
  针对宿主进程的 cwd 解释。同一个解析后的路径喂给 `openProject`、`inspectProjectPath`、`initProject`
  与 `createProject`，因此它们的分类与动作不会互相矛盾。然后它校验路径是仓库（`git rev-parse
  --show-toplevel`；非仓库以点名的 `NOT_GIT` 代码拒绝），按宿主解析后的书写形式记录根（git 在
  Windows 上打印 `C:/…`；architecture Decision 22），按根去重，并分配一个稳定、唯一、可读的 `slug`；
  `getProjects` 返回所有已知记录（含 slug 回填），`listProjects` 按 `lastOpened` 返回打开的记录，
  `listRecentProjects` 按 `lastOpened` 返回打开 + 已关闭的记录，仅同步丢弃那些「最后一次完成的检查
  发现其文件夹确实已消失」的*已关闭*记录（缺失，或已不再是目录）。打开的记录永不被检查，因此一个
  临时卸载的驱动器不会丢失任何东西；其他文件系统错误不证明删除，而尚未被检查的记录仍被列出。
  **`refreshRecentProjectFolders(budgetMs)`** 异步重新 stat 已关闭记录的文件夹，并在它们完成或预算
  用尽时结算；仍挂起的检查继续运行，后续刷新加入它而不是在一个挂死的挂载点上堆上第二次 stat。该过滤
  仅用于展示：已存记录不受影响。一个持久化的可选 **`Project.closed: true`** 就是全部成员状态：缺失
  即打开，因此既有记录迁移为打开。**`openProject`** 即使已关闭也能找到已知根，清除 `closed`，递增
  `lastOpened`，保留其 id，持久化并发布完整快照；**`closeProject`** 把同一条记录标记为关闭并发布它，
  而不删除项目、仓库、工作区记录或在线 runtime。**`renameProject`** 只替换显示用 `name`（空白折叠、
  去首尾空白、1–100 字符）并发布该记录；由文件夹派生的 `slug`、`path` 与 id 永不改变，因此工作区
  路径与身份不受影响，重新打开也保留所选名字。**一个 cwd，一个 ThinkRail 身份：** `openProject`
  拒绝（点名 `ALREADY_OPEN`）一个已作为某工作区 `worktreePath` 持有的根 —— pi 按*目录*键控聊天转录，
  因此在一个已被拥有的文件夹上再开第二个身份，会把该工作区的聊天当作自己的来服务，并在任一侧被归档
  时把它们清除。比较是**规范化**的（受管工作树存储的路径是组合出来的，`--show-toplevel` 回答的是
  符号链接解析后的路径），且只在上述重新打开**之后**（那次重新打开自己的 Default 工作区合法地持有
  该项目文件夹）。同一扇门的工作区侧一半是 `openExistingWorktree`
  （[[submodule-server-workspaces]]）；为此读取工作区记录仍留在 `persistence` 依赖之内 —— 本模块
  依旧从不导入它的兄弟模块。`setProjectPublisher` 是宿主注入的推送接缝；本模块从不导入 `host`。
  它还拥有 **`inspectProjectPath`**（对一个路径分类 —— `repo` / `initable` / `missing` /
  `notDirectory` —— 好让 UI 在打开、提供 init 或报错之间选择）与 **`initProject`**（引导一个普通
  目录：`git init` + `git add -A` + 一个 **allow-empty** 的初始提交 —— 提交该文件夹的内容，或在它
  为空时提交一个空提交，从而仓库获得 HEAD 且 `git worktree add` 可用；已是仓库的路径短路到
  `openProject`；缺失/非目录路径抛错）。**`createProject(parentPath, name)`** 是从零开始的路径：
  `name` 必须是一个普通文件夹名（无分隔符，不是 `.`/`..`），解析后的父目录在缺失时被创建，目标文件夹
  被创建 —— 既有**空**文件夹被复用，非空文件夹或文件则拒绝 —— 然后获得与 `initProject` 相同的引导，
  且即使它位于某个外层仓库内也仍在该文件夹中运行（`~` 里的 dotfiles 仓库不得吞掉新项目）；引导或
  打开失败时只删除本次调用创建的文件夹（仅当它仍为空；期间写入其中的任何东西都留存），并且总是删除
  它添加的 `.git`。一个可选的显示名 —— 用户键入的、slug 化之前的内容 —— 被传给 `openProject` 并在
  它发布的那一条记录里成为项目名（空白折叠、去首尾空白），因此创建只发出一个项目事件，而不是先以
  文件夹命名、再重命名。提交只为 git 未配置字段的情况提供**回退的 `user.name`/`user.email`**，因此
  真实的全局身份永不被覆盖。（「项目有规格吗？」**不**在此计算 —— `host` 经 `spec.projectHasSpecs`
  回答惰性的 `project.hasSpecs` 查询，使本模块不含任何规格依赖。）项目记录还拥有持久化的项目级技能
  准入状态：信任被显式且可撤销地授予；授予时把当前发现的项目别名快照为已确认，因此后来的新增仍为
  待确认；被禁用的 skill/组集合构成项目基线，位于任何工作区覆盖之下。宿主把发现与这些变更操作组合
  起来；本模块不存任何技能目录，也不导入任何 agent 代码。项目记录还携带 pi 级信任状态
  `piResourceTrust`（对项目原生 `.pi/` 资源的信任 —— 设置、扩展、技能、prompt、主题、`SYSTEM.md`、
  子 agent 定义 —— 与别名技能的授予不同）。一个新加入的项目起始为 `untrusted`，或当 pi 自身已信任
  其路径时为 `granted`；该答案经由宿主注入的 `setPiTrustSeed` 到达（本模块不导入任何 pi 代码）。
  **不存在祖父条款**：早于该字段的记录按存储原样保留并读作不受信，因此一个在 pi 级信任存在之前加入的
  项目，会通过信任通知询问一次，而不是保留没人同意过的可执行代码资源（用户决策，PR #690 评审）。
  pi 级信任本身从不启用 MCP：记录还携带 `mcpApprovals`（仓库定义的 MCP 服务器名 → 整个原始项目文件
  条目的指纹）、`mcpOverrides`（用户级服务器的逐项目启用/暴露，保持在受跟踪的 `.pi/mcp.json` 之外）
  与 `mcpToolGrants`（逐服务器，为「Always allow」工具的列表以及它们被授予时条目的 `target` ——
  放在仓库之外，因此它永不可能被预授予地交付）。`grantProjectMcpTool(id, name, target, tool)` 追加
  一个工具，`target` 变化时重新开始该列表；`revokeProjectMcpTool` 丢弃一个，并在为空时先移除服务器、
  再移除该字段。两者都把服务器名存为自有属性，因此名为 `__proto__` 的服务器只是一个名字。
  `setProjectMcpOverride` 只把提供的字段同步合并进最新的已存覆盖，因此重叠的启用与暴露补丁不会互相
  丢失。空补丁是空操作；`null` 清除覆盖。它在持久化之前拒绝非布尔的启用/暴露值，以及 `deferred`、
  `direct`、`hidden` 之外的暴露值。`setProjectTrust(id, trusted, acknowledged?, { resources? })` 保持
  两种授予相互分离，因为它们同意的是不同东西：pi 级资源包含代码（扩展、设置包），因此只有点名它们的
  界面才去请求它们。`trusted: true` 设置别名授予，且只在 `resources: true` 时设置
  `piResourceTrust: "granted"`，否则保持不动；`trusted: false` 撤销两者。一次信任变更会持久化并发布
  该记录，因此即使做出它的宿主请求随后失败（一个打开的聊天无法重载），每个客户端也都能显示已持久化
  的信任状态。
## 仓库克隆（Repository cloning）

`inspectCloneDestination` 返回经校验的绝对目标，供 UI 预览，但不预留它；克隆在占用文件夹之前重复该
检查。`cloneProject` 接受一个仓库 URL、宿主父路径、一个文件夹名以及可选的正整数历史深度。接受
HTTP(S)、SSH、Git、SCP 风格 SSH 与本地 `file:` URL；拒绝远端辅助程序、内嵌密码、HTTP userinfo、查询
串、片段与反斜杠。反斜杠在 web URL 解析器与 Git 之间有歧义，因此校验永不让一种书写形式上通过而 Git
执行另一种。Git 使用宿主既有的凭据辅助程序与 SSH 配置；ThinkRail 不收集任何凭据。父目录必须已经是
目录。任何已存在的目标（包括空目录或符号链接）都拒绝。一个独占的 mkdir 在 Git 启动之前预留目标。
失败与取消只清理那份预留，且在删除前检查它的文件系统身份；替代性条目永不被移除。克隆是异步的，有
30 分钟期限，取消会在清理之前等待进程拆除。克隆成功后跟一次 `openProject`；打开失败会保留检出以便
恢复。不启动任何 agent，也不隐式授予任何信任。

`startProjectClone` / `cancelProjectClone` 只在请求活跃期间把客户端拥有的操作 id 与一个
AbortController 配对。取消按所有者限定且幂等，原始克隆响应只在清理之后报告取消。socket 重连使用宿主
请求重放，而不是新操作。Git 诊断在截断之前先脱敏：用户配置的 URL 重写可能引入凭据，而先截断会把
秘密后缀与识别它所需的 URL 前缀分开。

- **公开接口（barrel）：** `cloneProject`、`inspectCloneDestination`、`startProjectClone`、
  `cancelProjectClone`、`openProject`、`listProjects`、`listRecentProjects`、
  `refreshRecentProjectFolders`、`closeProject`、`renameProject`、
  `getProjects`、`setProjectPublisher`、`inspectProjectPath`、`initProject`、`createProject`、
  `setProjectTrust`、`setProjectSkillEnabled`、`setProjectGroupEnabled`、`acknowledgeProjectSkills`、
  `approveProjectMcpServer`、`setProjectMcpOverride`、`grantProjectMcpTool`、`revokeProjectMcpTool`、
  `setPiTrustSeed`。
- **允许的依赖：** `persistence`；`git` 子模块（共享的 `git()` runner，它现在拥有其子进程派生所用的
  环境 —— 本模块不传任何东西）；`contracts`（`Project`、`ProjectPathStatus`）；`shared/codedError`
  （两个点名的打开失败）；Node/Bun。
- **禁止：** `host`；`git` 之外的兄弟特性（`workspaces` 依赖 `projects`，永不反向）。
