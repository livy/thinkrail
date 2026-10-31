# ThinkRail——顶层架构

> 中文译文，原文见 [`architecture.md`](./architecture.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 驱动因素（Drivers）

产品围绕 `pi` agent 构建，**在进程内**运行（`createAgentSession`）。两个增量式启动器共享同一个
宿主库：保留下来的 CLI 启动引擎宿主并打开浏览器，Electrobun 则把该宿主打包进原生系统 webview
shell。桌面配置仅限本地；后续的共享客户端配置可以连接已有的宿主。UI 独立于宿主发布，并通过
网络连接宿主；手机通过 Tailscale 访问选定的宿主。

## 拓扑——三个环（Topology — three rings）

- **引擎宿主**（`packages/server` + `packages/shared`，在本地宿主模式下由 `apps/cli` 或
  `apps/desktop` 启动）：拥有 `pi`、会话状态、持久化，并提供传输层端点。它把 pi 扩展
  （`pi-spec-graph`、`pi-thinkrail-workflow`、`pi-todos`）以及来自其服务端 registry 的
  ThinkRail 扩展（`visualize`、`ask-user-question`、`web-access`）打包进每个会话。
- **传输线**（`packages/contracts`）：带类型、带版本的协议——客户端与宿主之间唯一的耦合。
- **UI 客户端**（`apps/web`）：移动优先的 React 客户端，由传输层驱动、端点可配置，可作为
  静态资源独立于宿主发布。

```
apps/cli        browser host launcher: boot server + open browser ── depends on ─▶ packages/server
apps/web        UI client (mobile-first)                           ── depends on ─▶ packages/contracts, packages/locales, packages/ui, packages/extension-api/web, packages/resource-renderers, thinkrail-extensions/*/web
apps/desktop    Electrobun local-host launcher                     ── depends on ─▶ packages/server, packages/contracts, packages/shared, packages/locales
apps/website    public landing + blog + /vibecoding (Cloudflare Pages) ── depends on ─▶ packages/website-analytics
packages/website-analytics  dependency-free browser analytics policy for the public website
packages/server createServer(): Bun.serve(HTTP+WS) + AgentSessionManager (in-process pi) ── depends on ─▶ packages/contracts, packages/locales, packages/shared, packages/pi-background-commands, packages/pi-delegation, packages/pi-subagents, packages/extension-api/server, thinkrail-extensions/*/server
packages/contracts  the wire (types plus tiny pure runtime)
packages/locales    UI string catalogs (bundled en.json) + pure catalog rules; ./react holds the one i18next
                    instance ── depends on ─▶ nothing in the workspace (i18next, react-i18next)
packages/shared     shellEnv (server-side only)
packages/spec-graph portable pi extension: spec_* tools + skill (bundled into every session by packages/server;
                    its pi-free core/ read model also backs the host's spec.graph read method)
packages/pi-delegation         portable pure-pi package: the delegation core — controlled child sessions
                    for live session parents or independent resource owners
packages/pi-dag               portable durable backend DAGs over pi-delegation; host-owned resources;
                    not bundled into ThinkRail, no workflow UI, not a second pi runtime
packages/pi-background-commands portable explicit session-owned background commands over Pi's executor
                    (bundled into every ThinkRail parent session by packages/server)
packages/pi-subagents          portable pure-pi extension: Agent + get_subagent_result tools over
                    pi-delegation (bundled into every ThinkRail parent session by packages/server)
packages/pi-thinkrail-workflow pi extension: the workflow skill system + its always-on routing rule
                    (bundled into every session; workspace-internal, not portable)
pi-extensions/*     portable pi packages published to npm as @thinkrail/pi-<name>; work in vanilla pi
                    (decided, Decision 20; the pi-* packages above move here per publish wave)
thinkrail-extensions/*  ThinkRail extensions: a pi capability + ./server and ./web halves, composed by one
                    registry file per side (Decision 21; visualize and ask-user-question shipped) ── depends on ─▶ pi-extensions/*,
                    packages/extension-api, packages/ui, packages/locales (web half)
packages/extension-api  types + define* helpers for extension halves ── depends on ─▶ packages/contracts, packages/locales (types)
packages/ui         owned shadcn/Radix primitives + cn + onThemeSwap ── may depend on ─▶ packages/contracts, packages/locales
packages/resource-renderers  file/diff renderers, registry + review surfaces behind a host seam ── may depend on ─▶
                    packages/contracts, packages/locales, packages/ui
```

产物验证是一个独立的 source-only 工作区，[[module-artifact-tests]]。它依赖 CLI 构建元数据、
服务端测试夹具与共享的清理逻辑；根工具与浏览器 E2E 消费它。没有任何产品包导入该测试工作区，
它也没有应用构建步骤或 Electrobun SDK 依赖。这使测试进程驱动位于两个启动器与服务端库之外。

## 决策（Decisions）

1. **客户端/宿主分离。** 引擎宿主拥有 `pi` 与状态；UI 是可移植客户端；传输线是唯一的耦合。
   **规则：`apps/web` 只依赖 `contracts`、`locales`、`ui`、`extension-api/web`、
   `resource-renderers` 与 `thinkrail-extensions/*/web`**——绝不依赖 `server` 或 `shared`。
   所有新增的边都是浏览器安全的展示类包；UI 在没有宿主时依然可发布。
2. **启动器是薄的；宿主是库。** `apps/cli` 与 `apps/desktop` 都在进程内嵌入共享的启动路径。
   CLI 打开浏览器；桌面在全新的单源 loopback 宿主上打开原生系统 webview。二者都不拥有引擎
   逻辑，也不互相拉起。CLI 始终是完整、独立的产物与回滚手段。后续的桌面共享客户端配置可以
   省略本地宿主；每种配置都使用同一条传输线与同一个 web 产物。

   **各部署共用一条功能路径。** 一个普通的产品功能改动的是它的契约、所属服务端功能模块、
   共享 web 客户端以及它们的测试——绝不是每个启动器。启动器与未来的部署只拥有组装、生命周期、
   端点选择、原生呈现与产物打包。若某个真实的第二环境无法提供既有的宿主操作，则在拥有该行为
   的功能模块中开一个窄端口；不要预先用全局平台适配器把宿主抽象掉。物理运行时需求通过服务端
   拥有的 build-support 清单声明一次，再由各打包器转换。相同的行为与产物套件在每个启动器上
   运行，因此复用是由边界与一致性保证的，而不是靠并行实现。
3. **传输线带版本。** `contracts` 是类型加一个小型无依赖运行时：传输常量、版本、配置默认值
   与守卫。任何其他运行时都要在其规格中列为具名例外，目前只有退出确认规则；新增一个即需更新
   本决策。`server.welcome` 携带协议版本，使独立发布的 UI 能发现宿主版本漂移。
4. **传输层端点是参数。** 默认为同源（`location.host`）；远程浏览器、桌面或移动客户端把它
   指向所选宿主的 Tailscale MagicDNS 名称。原生恢复状态按 backend profile 建键，因此来自
   一个宿主的 id 绝不会被按另一个宿主解释。
5. **UI = 面板 + shell。** 与布局无关、由 store 驱动的面板（项目→工作区导航、文件树、代码
   渲染器、变更/diff、工作区局部评审、终端、聊天、输入框）从不知道自己如何排布。每个桌面前端
   窗口拥有一个本地持久化、无资源的 workbench frame：递归分割的中心区，加上位于纵向左右栈中
   的辅助组和水平分组排列的底部区域。当该窗口切换工作区时，frame 的拓扑、单例工具位置、
   可见性、折叠、几何与对齐保持不变；工作区范围的资源与注意力从各自独立的本地视图投射进去。
   终端可以占据中心区或辅助组，新建工作区默认把一个终端放在底部。另一个窗口绝不重排这个窗口。
   未来的移动 shell 可以用不同方式投射相同的面板；桌面停靠不定义那种投射。细节：
   [[submodule-web-shell-layout]]。
6. **工作区是 git worktree。** project（git 仓库）→ workspace（位于 `~/.thinkrail/worktrees`
   下、拥有自己的分支/cwd 的 `git worktree`）→ {聊天、文件、终端}。**两个刻意的例外，二者在
   传输线上都带 `kind` 标记，且都*归用户所有*——绝不被 ThinkRail 重命名或回收：** 每个项目
   恰好携带一个内置的 **Default 工作区**（`kind: "default"`），其 cwd 就是项目文件夹本身
   （git 的*主工作树*）——不可移除、不可重命名，只能从项目的 Welcome 分叉入口
   （“Work in project folder”）显式进入，绝不自动进入——它是困在 worktree 模型里的用户的
   “就在自己的项目文件夹里干活”锚点。唯一的例外是一个刚被 ThinkRail 创建为空、且开启了规格
   起草开关的项目：它为那次初始化聊天进入 Default，因为全新的仓库还没有可隔离的东西
   （[[submodule-web-panels]]）；以及用户显式就地附加的**已有 worktree**
   （`kind: "external"`），ThinkRail 可以遗忘它，但绝不修改它（见
   [[submodule-server-workspaces]]）。shell 先建，`pi` 最后接。**Open PR** 是通过用户自己的
   `gh` CLI 做的确定性宿主侧推送与打开/更新分支的 GitHub PR（不存 token，不走 provider REST
   API），正文由经核实的计划渲染，`gh`/GitHub 不可用时回退到 compare URL（见
   [[submodule-server-pr]]）。
7. **认证是外部的。** Tailscale ACL 与设备身份就是认证；应用携带 `owner` 字段，而不是登录 UI。
8. **先水合、再流式（每个客户端都从宿主重建领域状态）。** 客户端绝不依赖*亲眼见证*过事件来
   得知领域状态——连接时它**读取**当前状态，然后**订阅**实时增量。宿主在 `pi.event` 之外
   暴露 `project.list` / `workspace.list` / **`session.list`** / **`session.getMessages`**。
   因此刷新、第二个标签页、手机或**宿主重启**都能重建同样的项目、工作区、会话与记录。
   `session.list` 把内存中的会话与 pi 磁盘上的会话并集在一起；基于这次权威读取，一个界面会
   水合它本地放置的聊天，然后被动自动打开数量有界的最新未放置会话——条件是它们仍然存活或带有
   未完成 todo（当没有任何会话符合条件、且尚未放置任何东西时，以单个最近的会话作为回退打开，
   因此只要存在聊天历史，工作区就不会空着落地），其余一切则列入历史供显式重开。这次自动打开
   尝试对每个界面-工作区连接只触发一次，而不是每次回读目录都触发；解析精确聊天路由目标的界面
   会完全把这件事让给该目标。另一个前端启动会话时，`session.created` 提供那条仅属于历史的
   实时增量；重连通过 `session.list` 修复错过的增量。客户端是**领域状态的无状态投射**，绝不是
   第二个领域真相来源；它另外独立拥有前端局部导航与 workbench 视图状态。自动 agent 运行在
   重试、压缩与排队续跑期间保持活动：pi 的 `agent_end` 只是尝试边界，之后可能还有更多工作；
   `agent_settled` 才是转入空闲的权威转换。

   **归一化的会话状态。** 宿主把执行、具体的输入阻塞、Pi 拥有的队列计数与最近一次完成投射为
   正交的事实。需要输入是水平触发的，且不能通过查看被清除；完成只由 `agent_settled` 或重启
   重建创建。成功、失败/超长与中断在所有者范围内保持未读，直到看到确切结果为止：在聚焦窗口里
   聚焦且未被遮挡的聊天中渲染，且记录处于最新边缘。显式 Stop 是静默的，空闲聊天的完成要等其
   后台命令与 subagent 都结束后才变为未读；在此期间宿主报告该后台工作，因此聊天读起来像是在
   工作。仅有的显式读取手势按工作区进行——*Mark as unread* 重新点亮其最近的结果，
   *Mark as read* 清除每一个未读结果——它们写入同样的宿主回执，因此“看到”仍是默认路径。
   每个客户端都从宿主水合相同的状态与精确的完成回执，而焦点、在场与滚动位置始终是前端局部的
   “看到”证据。

   **聊天标题契约。** 工作区显示名、其 Git 分支/cwd，以及每个聊天标题都是彼此独立的身份；
   它们之间不存在重命名级联。聊天标题是 pi 持久的会话名（`session_info`），绝不是浏览器视图
   状态或宿主 sidecar。**主 agent 通过 `set_title` 工具在任务明确后为其聊天与工作区命名**：
   它拥有读取关联 PR/issue/ticket（`Review #567 <title>`）的上下文与工具，且适用于任何
   tracker。没有辅助模型、首词启发式或宿主侧链接解析来命名任何东西：这些做法都产生了用户想要
   修正的名字。每次写入都以目标仍未被命名为前提，因此任何手动命名始终胜出，且第一个名字即最终
   名字——不会有任何自动重命名。当目标未被命名时，每一轮的系统提示都会说明这一点；仅靠指南曾
   被可测量地跳过。当一个受管工作区未被命名时，每次调用都必须携带其名称，并且要求 agent 在
   一次调用中传入所有缺失的名称，以便聊天与工作区一致，但它们仍是各自独立写入的身份——不是
   级联。如果 agent 从未调用该工具，目标保持未命名（已接受）；未命名的聊天*显示*为其首个
   prompt 的开头，宿主把它投射在标题旁（`firstPrompt`），且绝不写入任何地方——这是只读标签，
   不是名字：重命名字段从真实的（空）标题开始，而 agent 或用户的标题会在所有界面同时替换该
   标签。客户端水合 `SessionSummary.title`，在 `session_info_changed` 上实时收敛，并继续按
   会话 id 路由，因此重复的人类标题是合法的。
9. **领域状态、前端局部 frame 与工作区局部视图。** *领域*状态——项目、工作区、**会话及其
   记录**、终端目录/PTY 与 git——由后端拥有、共享且持久；每个客户端都从宿主水合它。当前
   workbench 状态是视图状态，绝不跨过传输线。每个浏览器标签页或原生窗口恰好拥有一个无资源的
   `WorkbenchFrame`，用于中心区与左/右/底部拓扑、单例工具位置、可见性、折叠、归一化几何、
   底部对齐与恢复目标。它另外为每个工作区拥有一个 `WorkspaceViewState`，用于打开的文件/
   diff/聊天/文档/终端放置、标签顺序与预览，外加一个以该 frame 为键的每工作区
   `LayoutAttention` 覆盖层。挂载的 workbench 是这些局部值的投射，而不是另一个权威。

   frame 的变更局限于一个前端窗口，并通过其 shell 拥有的本地存储适配器持久化。切换工作区
   只改变被投射的工作区视图。空分组会一直保留，直到某条显式 frame 命令移除或合并它们；这样的
   命令会在每个本地保留的工作区视图中原子地重新安置受影响的资源。应用预设也是如此。另一个
   浏览器、设备或窗口既不会收到也不会采纳这些变更。内置预设与显式本地 frame 重置所用的默认值
   仍归客户端所有；只有数量有界、无资源的自定义预设定义由宿主持久化并作为设置广播。传输线上
   不存在任何当前布局快照、修订、变更、读写方法或推送通道。

   这始终只是放置，而绝不是资源生命周期。关闭文件/聊天放置是局部的，会话仍然存在；终端关闭
   保留其显式的宿主领域 PTY 语义。活动客户端位置同样是局部的：一条相对于后端的路由命名
   main / Project Home / workspace / 精确聊天；web 把它存在带版本的 fragment 中，而原生
   shell 按 backend profile 与窗口持久化它。传入的 id 会针对已水合的宿主状态校验，且不存在
   后端拥有的“当前屏幕”或当前布局，让一个客户端去移动另一个。没有有效本地文档的前端界面直接
   从 Balanced frame 开始。先前持久化的宿主布局快照与旧的浏览器注意力条目永远不会被读取、
   迁移或删除；已退役的配置、预设与终端标记形态被忽略，而不是升级。细节：
   [[submodule-web-shell-layout]] 与 [[submodule-web-shell-layout-state]]。
10. **依赖锁定精确版本。** 每个清单里的每个依赖都锁定**精确**版本——不用范围（`^` `~` `>`
    `<` `.x` `*`）。理由：`pi` 每天发布破坏性版本，因此浮动范围就是一条带电的电线；更普遍地
    说，静默的 minor/patch 升级是经典的可复现构建陷阱。精确锁定使 lockfile 成为依赖版本的
    唯一来源，并让每次升级都成为显式、可审查的 diff。横切依赖（pi、TypeScript、typebox、
    bun types）只在根 `workspaces.catalog` 中锁定**一次**，并通过 `catalog:` 引用，因此它们
    的版本只存在于一个地方**且只在那里：规格绝不重述它。** 依赖 pi 行为的规格只命名它验证了
    *什么*（dist 文件、函数、观察到的规则），并说明“pi 升级时重新验证”；它不携带“锁定于
    vX”，那是 catalog 的第二份副本，每次升级都会失效，而且不增加任何信息。历史理由
    （“pi 0.86 让 loader 的选择变成运行时相关”）则不同：它解释*为什么*存在某个决策，且
    永远不需要更新。
    **被强制**，而不仅是被记录：`scripts/check-catalog.ts`（`bun run check:deps`，位于
    pre-commit + CI）拒绝任何范围、任何 catalog 漂移，以及把 `react` 或 `react-dom` 解析到
    其唯一 catalog 锁定之外的 lockfile 图（React 锁定的理由属于 [[module-web]]）。每次 pi
    升级还要遵循 [[pi-compatibility]]——由架构拥有的宿主适配登记表及其移除门禁；详细理由留在
    各所属模块规格中。任何其他根 `overrides` 条目都是精确的安全锁定，其理由与移除门禁随其
    所有者。豁免：`peerDependencies`（扩展包刻意声明 `"*"`——依赖由宿主提供）与本地协议
    （`workspace:` / `link:` / `file:`）。精确的 SemVer 预发布/构建后缀仍然是精确锁定
    （`19.3.0-canary-a1124489-20260826`）；检查器接受完整的标识符语法，包括连字符，但不接受
    范围。

    根 `packageManager` 字段还为开发、CI 与 CLI 编译锁定 Bun；Bun 类型放在 catalog 里。
    Bun `1.4.4` 使这些路径与桌面运行时对齐，桌面配置通过 Electrobun 的 `build.bun.version`
    选择该运行时（见 [[module-desktop]]）；一个桌面测试保持这两个锁定相等。CI 读取根锁定，
    而不是在 workflow YAML 中维护第二个版本。

11. **终端 = DOM 渲染器上的 xterm.js。** 浏览器终端是 `@xterm/xterm`，由
    `apps/web/src/panels/TerminalInstance.tsx` 驱动，连接到 `packages/server/src/terminal`
    中的真实 PTY（`bun-pty`）。它保持这个选择，因为它是唯一可用于生产的浏览器终端：可信的
    替代品都是编译到 WebAssembly 的 Ghostty VT 引擎（`ghostty-web`、`restty`、`wterm`），
    而其中最成熟的一个只发布了一个 tag，既不支持鼠标上报也不支持 OSC 8 链接——vim/htop/
    lazygit 都会退化。**渲染器刻意是默认的 DOM 版**，而不是 `addon-webgl`：xterm 自己的
    维护者称 DOM 渲染器是触摸支持的前提，而 WebGL 带有我们会继承的缺陷
    （`WebglAddon.dispose()` 会泄漏其 WebGL2 上下文——对我们这种按 worktree 频繁开关终端的
    场景是致命的——还有 iOS 的上下文数量上限崩溃）。加载 `addon-webgl` 会是退步而非升级；
    连字与 `rescaleOverlappingGlyphs` 是被接受的代价。耦合被刻意保持得很薄（约十几个 xterm
    API 成员；没有 parser 钩子、装饰或序列化），因此更换仍是对单个文件的受控重写。**当以下
    两者同时成立时重新评估**：(a) 上游为 `libghostty-vt` 打 tag 并提供官方 WASM/npm 分发，
    以及 (b) `ghostty-web` 发布 0.4.0 之后且鼠标上报与 OSC 8 可用。

12. **shell 属于标签页，映射由宿主拥有。** 终端以 `(workspaceId, tabKey)` 为键；
    `terminal.reserve` 可以在没有进程的情况下持久地建立目录标签页，而一个幂等的
    `terminal.attach` 仍是其 PTY 诞生的唯一方式。预留会在发布成员关系之前持久化，如果持久化
    失败则回滚其内存中的插入。这种分离让一个同步的隐藏默认放置能在不启动 shell 的情况下跨刷新
    与另一个客户端存活。客户端自己不留 tab→shell 指针。shell 是**按所有者划分作用域**的，
    与 `history`/`todos`/`templates` 一致，因此它们能跨刷新、关闭的浏览器与不同的浏览器
    存活——attach 是独占的，接管一个标签页会通知被顶替的客户端。生命周期由引用（没有标签页
    → 没有 shell）加上宿主进程来界定，**而不是**由定时器：不做空闲清理，不做废弃客户端收割。
    宿主重启无法保留 shell（进程内 `pi`，PTY 挂断），因此标签页会以新 shell 复活并显示已记录
    的输出。**tmux 被拒绝**作为持久化层：在 Windows 上是无法假定的依赖、一个竞争的标签页
    模型、环境变量传播的破坏，以及基于轮询的捕获——为了重启存活，我们已经决定不保留。细节：
    [[submodule-server-terminal]]。
13. **Central 的跨模块生命周期只有一个架构所有者。** 它的适配器、运行时代际、传输线状态/
    配额、同步偏好、provider 卡片与顶栏读数保留在各自有界的模块中；这些界面与其存活性义务
    之间的对应关系属于 [[central-integration]]。这使功能专属的机制留在各自的叶子规格里，
    同时让一个非终止的组装在架构层可见。

14. **公共网站是一个源与一个生产部署。** `apps/website` 在一次静态 Astro 构建中拥有 `/`、
    `/blog/` 与 `/vibecoding/`，并与同项目的 Cloudflare Pages Functions 一起部署。由 D1
    支撑的 [[submodule-website-attribution]] 在该部署下提供短时浏览器声明；它传递有界的
    活动/推荐来源触点数据，而非页面身份，并且不是第二个产品宿主或身份所有者。React 与
    Tailwind 只允许在 [[submodule-website-vibecoding]] 内使用；无关路由保留其原生的 vanilla
    运行时与手写样式表。浏览器分析与同意在精确的 `thinkrail.ai` 源上初始化一次。已退役的
    `vibecoding.thinkrail.ai` 主机名是一个边缘重定向，保留路径与查询，绝不是到第二个站点的
    代理。

15. **桌面打包保留宿主/运行时边界。** Electrobun `2.0.5` 显式选择锁定为 `1.4.4` 的 Bun
    运行时，并在该进程中嵌入宿主，而不是实验性的 Cottontail 运行时；它绝不包裹或拉起 CLI。
    它精确的 npm 引导锁定选择 Hutch 构建工具链与生成的 SDK；Bun 仍是工作区包管理器。原生
    窗口从宿主实际的 loopback 端口加载打包后的 web 构建，因此 UI、传输线、文件与 SPA 回退
    保持同一个源。需要路径的原生资源保持未打包状态。shell 在导入 server 之前设置暂存的
    `bun-pty` 库，并从单独打包的 `.ts` 运行时加载 PI，使外部 TypeScript 扩展获得 PI 打包的
    虚拟模块，而不是不存在的 built-Node 别名。CLI 与桌面共享宿主启动与优雅关闭，但启动器
    不强制任何进程级单实例或规范数据目录所有权策略。每个宿主绑定自己的 loopback 端口；当
    多个宿主指向同一个可变数据目录时，跨进程一致性刻意不做保证。桌面产物是增量式的；
    Ubuntu 24.04+/glibc 2.38 上的原生 WebKitGTK 是受支持的 Linux 下限。标准框架 CLI/配置
    拥有打包与安装器；一个已记录的 pre-build 钩子准备 ThinkRail 的物理资源与 PI 运行时。
    `JetBrains/thinkrail-signing` 中的发布 workflow 消费公开的构建配方，并为某个显式公开
    源码提交协调“构建 → JetBrains 服务签名 → 发布”。产品代码与普通 CI 保持公开；凭据与
    发布保持私有。macOS 服务签名消费框架展开后的 app 归档，并通过已记录的 JetBrains SRE
    流程完成 DMG，没有本地 Apple 凭据流程，也不修改 Electrobun 的压缩包装。确切的交接与
    发布验证契约属于 [[module-ci-release]]。细节：[[module-desktop]]、
    [[module-ci-release]]。

16. **委托是可移植的；ThinkRail 只是嵌入者之一。** `packages/pi-delegation` 拥有会话织体：
    一个带有正交轴的创建原语、一个运行所有权句柄、血统、registry 与生命周期事件。
    `packages/pi-subagents` 消费它以暴露 `Agent` 工具。二者都在原版 pi 下工作，SDK 作为
    `peerDependency`（peer 依赖豁免精确锁定规则，见决策 #10），创建进程内的隐藏 pi 会话，
    并保持其宿主绑定可选。ThinkRail 在 `packages/server` 中组装它们：每个工作区一个服务、
    宿主数据目录下的子记录、一套精选的子扩展集合，以及每个父会话保留的精确 `ModelRuntime`，
    使子会话在 Central 变化时仍处于该父会话的 provider 代际。传输线只镜像面向 UI 的运行
    细节并暴露记录读取；两个可移植包都不依赖 ThinkRail。同一个核心支持拥有显式执行上下文的
    独立资源所有者，以及面向持久编排器的不可变捕获历史分叉；它绝不伪造父聊天，也不获取
    调度/存储策略。契约、语义与完整决策日志：[[module-pi-delegation]]、
    [[module-pi-subagents]] 与 [[submodule-server-agent]]。

17. **持久 DAG 是宿主拥有的资源，不是父聊天。** [[module-pi-dag]] 是
    [[module-pi-delegation]] 的一个独立作用域的可移植消费者，不是 subagent 或 workflow
    skill 系统的依赖。委托拥有规范的历史捕获/分叉、保留的资源执行上下文、子会话组装/重开与
    现有的运行循环；DAG 拥有持久化、调度、门禁与恢复。ThinkRail 选定的未来组装是工作区
    拥有的，并带有显式的宿主运行时，因此不需要领头会话。嵌入者可以通过可信的生命周期控制
    应用不同的生命周期策略，而无需另一个调度器或绕过人工门禁。恢复保持暂停；一个进程控制
    每个 DAG。ThinkRail 既不打包它，也不通过传输线或 UI 暴露它。

18. **Chat Resources 投射能力所有者；它不是通用资源运行时。**
    [[module-pi-background-commands]] 在 Pi 的公开执行器之上提供显式的、会话拥有的仅记录
    命令；普通 Bash 与工作区 PTY 保持不变。`packages/server` 通过 `agent` 嵌入该可移植包，
    并把它与直接的 [[module-pi-delegation]] 子会话一起投射到当前聊天的 Resources 视图中。
    父会话条目拥有注入的命令服务与跨扩展重载保留的 subagent 完成投递；作用域读取、控制与
    失效经 agent barrel 组装。详细集成属于 [[submodule-server-agent]]，命令生命周期/保留
    属于 [[module-pi-background-commands]]。

19. **评审界面与引擎无关；渲染器是注册的，不是硬连线的。** 一个资源（一个文件，或 diff 的
    一侧配对）由 `@thinkrail/resource-renderers` registry 依据当前意图与视口类别，按匹配、
    排序与能力选出的渲染器展示；评审评论通过 `contracts` 的 `ReviewSelector` 集合锚定
    （文本用 `lineRange`/`textQuote`，`json-pointer`/`table-cell`/`ipynb-cell` 等
    `structural` 方案，以及 `region` 几何），每个渲染器把它映射到自己的几何并回报为已放置
    或未放置——宿主的锚点是唯一权威。`@pierre/diffs` 渲染每个源码 diff 与每个手机类代码
    界面；富格式通过其自身渲染器的 diff 渲染。Monaco 在桌面上渲染文件，且绝不在手机上加载。
    写入路径由宿主推导，并以比较并交换（compare-and-swap）保护
    （`change.revert`/`change.undo`，[[submodule-server-changes]]）；客户端绝不发送要写入的
    字节。ThinkRail 不承载 VS Code 扩展：语言智能（TextMate 语法、Shiki）作为库导入。活动
    内容（HTML、SVG、notebook 输出）只在沙箱化、拒绝网络的 frame 内渲染。渲染器、registry
    与评审界面是一个包（[[module-resource-renderers]]），位于一个宿主接缝之后（markdown、
    代码高亮、字节 URL、链接导航），因此每个展示文件的界面都使用同一套栈。细节：
    [[module-resource-renderers]]、[[submodule-web-panels]]、
    [[submodule-server-reviews]]。

20. **可移植 pi 包被发布、划分作用域，并遵守原版对等（vanilla-parity）标准。** agent 在
    任何地方都能使用的能力位于 `pi-extensions/*` 下，并以 `@thinkrail/pi-<name>` 发布到
    npm（npm 组织是 `thinkrail`，它也保留了私有、永不发布的 `@thinkrail/*` 工作区包的
    作用域；原生 TypeScript、`pi` 清单、`pi-package` 关键字；无作用域的 `pi-*` 名称会与
    第三方冲突）。
    “在原版 pi 中可用”是被定义的，而不是被假定的：从打包的 tarball 安装到隔离夹具，在
    **Node**（原版 pi 的运行时——ThinkRail 在 Bun 下运行同样的代码，因此发布的代码是双运行时）
    下通过 pi 自己的 loader 加载，注册并执行工具，在真实终端中渲染。发布经 Changesets 与带
    provenance 的 npm Trusted Publishing；宿主通过 `workspace:*` 继续消费这些包。可移植的
    *库*（delegation）在不带清单的情况下发布，且绝不赋予伪造的 factory。细节与门禁：
    [[module-pi-extensions]]。

21. **ThinkRail 扩展是位于小型宿主 UI SDK 之上的独立包，由 registry 文件组装。**
    `thinkrail-extensions/<name>` 把一个 pi 能力与一个 `./server` 部分（宿主打包的内容，
    作为具名内联 factory；skill 包按 specifier 命名，并从扩展自身而非宿主解析）和一个
    `./web` 部分（按工具名键控的工具渲染器，加上宿主直接使用的具名导出）组装起来。两半
    绝不互相导入，也不导入宿主内部；宿主拥有的作用域状态只通过显式接缝到达扩展——这是现在
    唯一固定的特性，以便后续扩展（传输线方法、面板）扩展 `define*` 对象而不是替换它们。
    组装是静态的：一个服务端 registry
    （`packages/server/src/extensions/registry.ts`），其静态导入把 factory 带入每个
    启动器，而无需生成的 factory 列表（打包器生成的列表现在只覆盖尚未包装为扩展的 pi 包），
    以及一个 web registry（`apps/web/src/extensions/registry.ts`）。skill 包从扩展自身的
    依赖图解析，绝不从宿主解析。SDK 是 `packages/extension-api`（类型 + `define*`）与
    `packages/ui`（自有原语、`cn`、`onThemeSwap`）；高亮的 `CodeBlock` 在出现第二个消费者
    之前保持应用局部。决策 1 的浏览器专用依赖规则通过 source-half 与公开子路径检查来强制，
    包括 SDK 分离的 `./web` 与 `./server` 契约。**交付规则：** 每个扩展以三个可独立发布的
    PR 到达——新 pi 包放在旧包旁边、web 半边迁移、server 半边 + 接线并删除旧包——其中没有
    一个会改变用户可观察到的任何东西。**启用/禁用已发布：** Settings → Extensions 列出
    服务端 registry（`extension.list`），每个扩展一个开关；全局
    `AppConfig.disabledExtensions` 通过宿主注入的 resolver 到达 `agent`（不存在
    `agent` → settings 的边），它在每次 loader（重新）加载时过滤 registry factory **及其
    skill**，并通过 pi 的活动工具 API 从实时会话中扣留被禁用扩展的工具——只有当该扩展的
    skill 在该会话中加载时才重载空闲会话——而 web 渲染器保持注册，以便过往记录仍可渲染。
    安装 UX、市场、每扩展设置、每工作区启停覆盖与运行时加载的第三方扩展仍是显式的推迟项。
    被拒绝：现在采用运行时加载的 bundle（先解决 React 单例、带版本的 UI API、安全叙事）、
    把逻辑扩展放进 apps 内部（三个物理住所，没有边界）、从描述符派生的生成式 factory 列表
    （函数不产出导入 specifier）、在 inline 组装旁保留基于路径的 dev 加载（一种组装模式；
    `{ name, factory }` 保留诊断能力）、从宿主解析 `skillPackages`（宿主不再依赖扩展拥有的
    pi 包，因此在隔离布局中会出现 `MODULE_NOT_FOUND`）。试点，分三个 PR 交付：visualize
    ——可移植包中用 `lovely-mermaid` 做 TUI 渲染与尽力验证，ThinkRail 的 server 半边通过
    `createVisualizeExtension({ validateMermaid })` 注入严格的 `mermaid`+`linkedom` 验证；
    旧的 `packages/pi-visualize` 被删除。第二个：ask-user-question——可移植包通过终端问卷或
    逐问题对话框提问；ThinkRail 通过 registry 的 `sessionExtensions(session)` 接缝按父会话
    组装它，asker 位于宿主的**交互工具邮箱**（`awaitToolResponse`）之上——这是扩展收到的
    第一个按会话的宿主服务。该邮箱、重启修复、迟到响应、输入框顶替、Stop/Submit、需要输入
    以及 web 的 `interaction` props 都是**与工具无关**的：以 `toolCallId` 为键，由描述符的
    `interactiveTools` 与渲染器的 `interactive` 标志告知哪些工具名在等人，且在 `contracts`、
    `agent` 或 `apps/web` 中绝不指名任何工具。`ask_user_question` 是该契约的第一个客户端，
    而不是特例；该扩展可以删除而不触动核心（原版包随后回退到 pi 的 `ctx.ui` 对话框，宿主
    已经渲染它们）。产品规则：内置（registry）扩展在其启用期间赢得与用户包的同名工具冲突；
    关掉后，用户包的工具运行（[[pi-compatibility]] 持有 pi 层的机制）。第三个：web-access
    ——**第三方** pi 包（`pi-web-access`）的第一个包装，因此没有我们自己的包要发布，交付
    收缩为 server 半边；其渲染器保持应用局部，直到它们使用的组件加入 SDK。它加载该包声明的
    一切，如同 `pi install` 所为——工具及其 `librarian` skill——并且是第一个带 skill 的
    扩展，因此开关现在会连同工具一起丢弃扩展的 skill（启动器把它们暂存在所有者文件夹下，
    使所有权在扁平化后依然存在）。细节：[[module-thinkrail-extensions]]。

22. **路径的拼写跟随其类型。** 目录身份（`Project.path`、`Workspace.worktreePath`）是宿主
    的**已解析**拼写（`path.resolve`），正是 pi 记录为记录 cwd 的拼写（pi 会解析交给它的
    cwd），因此把它与 pi 报告的 cwd 比较就是普通的 `===`。以其他拼写进入的路径在进入处解析：
    git 在 Windows 上打印 `C:/…`，`THINKRAIL_DATA_DIR` 可能是相对路径，而在此规则之前写入的
    记录会在加载时被修复（[[submodule-server-persistence]]）。传输线上或 store 中的
    **工作区相对**路径（文件树、变更、评审锚点与阅读顺序、fs 事件）是 POSIX 的 `/` 且包含在
    worktree 内；服务端在进入处转换原生 `path.relative` 输出与模型写入的路径，git 对象名
    （`<ref>:<path>`）使用 `/`。pi 报告的绝对路径（工具参数、记录 cwd）仍属于 pi，可能使用
    任一种分隔符；浏览器通过 `projectRelativePath`（[[submodule-web-lib]]）读取它们，而在
    agent 书写的行文或路径中，反斜杠是分隔符，绝不是 Markdown 转义。原因：同一个 Windows
    目录的两种拼写破坏了每一处裸比较（#766），导致 Default 与外部工作区中的聊天无法被列出、
    重开、重命名、删除或在历史中限定范围。被拒绝：在每个比较点归一化（每个未来的比较都必须
    记得；第一个补丁在十二处中漏掉了九处）；把 `realpath` 作为规范形式（在存储路径跨过符号
    链接或 8.3 名称的地方，会与磁盘上已有的头信息分歧）；折叠大小写（pi 自身的 cwd 身份区分
    大小写，git 与 `resolve` 都保留磁盘上的大小写）。被接受的代价：共享 `~/.thinkrail` 的
    较旧 ThinkRail 对一个已修复的项目会在从 Recent 重开时失败（`ALREADY_OPEN`），直到它
    升级。

23. **UI 文本是键控目录；本地语言文件是运行时数据，不是代码。** `apps/web`、`packages/ui`
    与扩展 web 半边中每个面向用户的字符串都是 [[module-locales]] 的 `en.json` 中的一个具名
    扁平键（`settings.language.title`），那是唯一的英文真相来源，通过一个共享的 i18next 实例
    （`@thinkrail/locales/react`）渲染。**lint 门禁会让硬编码的面向用户文本失败**（这些树中
    的 JSX 文本与带文本的属性，如 `aria-label`、`title`、`placeholder`、`alt`、`label`），
    因此新字符串不带上 key 就无法发布。ThinkRail 随附打包目录（英文加上由测试对齐到
    `en.json` 的译文）；用户通过把扁平 i18next-JSON 文件放入**全局**数据目录
    （`~/.thinkrail/locales/<tag>.json`）来新增或覆盖语言——绝不是工作区，因此克隆的仓库
    无法改写 UI。宿主校验并监听这些文件（[[submodule-server-locales]]）并通过传输线提供
    它们；web 把它们合并到打包目录之上（[[submodule-web-i18n]]），逐字符串回退顺序为
    *本地精确 → 打包精确 → 打包基础 → 英文*。选择（`AppConfig.locale`，默认 `"system"`）
    像主题一样跟随用户。agent、工具、skill、prompt 输出与服务端错误文本绝不翻译。存储与
    传输线上的数据绝不携带译文：ThinkRail 写入的默认值（未命名聊天、默认终端、宿主命名的
    默认工作区、聊天轮次通知）在渲染时措辞，因此切换语言会重新措辞它们全部，而由用户或 agent
    设置的名字逐字显示。主题清单是另一种运行时加载的用户资源，模型相同
    （`~/.thinkrail/themes/*.theme.json`，由 [[submodule-server-themes]] 读取、由
    [[submodule-web-themes]] 校验）：清单只是颜色数据，写为 CSS 自定义属性值，绝不是 CSS
    或代码。代码扩展保持编译进去（决策 21），因为字符串映射或调色板不携带可执行面，而扩展
    携带。被拒绝：以英文文本作键（一次措辞修正会孤立每一份译文）；由宿主提供打包目录
    （打包字符串属于渲染它们的 web 构建）；自研运行时（选择 i18next 是因为其标准格式与
    工具）；把目录放进 `contracts`（数据与翻译逻辑不是传输线类型）。推迟：RTL 布局、
    一个 agent 翻译 skill。

## 不变量（Invariants）

- 绝不要在浏览器打包的代码中对 `pi` 做**值**导入；只导入类型，且从 `pi-ai` /
  `pi-agent-core` 包根导入（仅类型导入在构建时被擦除，使 bundle 保持无 provider）。
  `@earendil-works/pi-coding-agent` 仅限服务端——它绝不进入 `contracts`/`web`。
- 单一 id 模型：UI 标签页 id 与 `session.sessionId`（`AgentSession` 的 id）。没有单独的
  pi UUID。
- agent 在进程内运行，**没有崩溃隔离**——包裹会话调用并转发错误；致命故障会让整个宿主倒下
  （相对于子进程 RPC 模式接受的权衡）。
- `pi` 拥有状态并发出真相；宿主是薄桥——它通过读取方法**暴露** `pi` 的状态（不重新计算），
  并把 `pi` 的事件作为增量转发。客户端**从读取中水合，然后流式接收增量**——它们只持有自己
  的视图状态。
- 由宿主或 CLI 启动的后台控制台子进程设置 **`windowsHide: true`**。有界的 Windows 子进程
  还保持**非脱离（non-detached）**，使普通辅助进程继承一个非可视控制台，而不是自己打开一个；
  平台策略与原生回归属于 [[submodule-server-subprocess]]。这包括窗口聚焦时的 PR 重新校验；
  终端忙碌检查属于关闭动作，Central 配额在可见性变化时续读，而不是在普通聚焦时。同步与
  发后即忘的场景使用 `@thinkrail/shared/spawn`；定制的有界 runner 直接设置该选项。豁免：
  继承已有终端 stdio 的 spawn（CLI 的 `update`/`uninstall` 子命令、构建脚本）以及在 win32
  上是空操作的 shell 探测（`shellEnv`）。
