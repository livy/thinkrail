# terminal —— 工作区 PTY

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

以工作树的 cwd 为根的、工作区作用域的 `bun-pty` 终端，以及逐工作区的终端身份目录。一个标签页的
shell 比每个观察它的客户端活得更久；每个前端各自从本地的 workspace-view 摆放中引用它的 `tabKey`，
而本模块从不接收那个摆放。

## 边界（Boundary）

- **拥有：** 持久化的逐工作区终端目录（稳定的存在/元数据顺序，而非可视 workbench 摆放）以及每个
  标签页背后的 PTY，以 `(workspaceId, tabKey)` 为键；`terminal.data` 上的批次输出加上
  `terminal.exit` / `terminal.detached`（寻址）与 `terminal.tabs`（广播），经注入的发布者发出；
  在附加时重放的有界逐终端输出记录器。
- **公开接口（barrel）：** `reserveTerminal`、`attachTerminal`、`listTerminals`、`writeTerminal`、
  `resizeTerminal`、`closeTerminalTab`、`resumeClientTerminals`、`closeWorkspaceTerminals`、
  `persistTerminalSessions`、`reviveTerminalSessions`、`closeAllTerminals`、`clearTerminalClient`、
  `resetTerminalState`（测试接缝）、`setTerminalPublisher`、`setTerminalTabsPublisher`；一次被接受
  的 `writeTerminal` 返回其所属工作区 id（被顶替的调用方返回 `null`），因此宿主只把真正到达某个 PTY
  的输入计为工作区活动；与宿主发布者适配器共享的 `TerminalDeliveryResult` 类型。
- **允许的依赖：** `persistence`、`contracts`（`WS_CHANNELS`、`TerminalWindowsShell`）、`bun-pty`、
  `Bun.which`、`Bun.spawnSync`（`id -un`）、`process.env`。
- **禁止：** `host`；兄弟特性。没有 WebSocket 类型跨越这个边界——客户端是不透明的 key。

## 决策（Decisions）

- **shell 选择是终端本地的**，而 Windows 的选择还额外可由用户配置
  （`AppConfig.terminalWindowsShell`、`apps/web/src/panels/TerminalSettings.tsx` 的仅 Windows 选择
  器；[[task-windows-default-terminal-shell]]）。优先级从高到低：显式的 `SHELL` 环境变量总是胜出
  （未变——宿主不在 Windows 上发明一个全局 `SHELL`，因为那是 Unix 登录 shell 的约定，改它会影响
  进程内 agent 和每一个无关的子进程，而只是为了配置本模块的 PTY 可执行文件）；然后，仅在 Windows 上，
  `terminalWindowsShell`：
  - `"auto"`（默认）：如果 `Bun.which` 能在 PATH 上解析到 `pwsh.exe`（PowerShell 7+）就用它，否则
    用 `powershell.exe`。不再进一步回退到 `cmd.exe`——`powershell.exe` 随每个受支持的 Windows 版本
    发布，所以它已经是有保证的地板。`defaultWhich` 传入显式的 `{ PATH: process.env.PATH }`，而不是
    裸调用 `Bun.which`，这与 `editors` 的 `defaultWhich` 所防的是同一个陷阱：单独调用 `Bun.which`
    读的是进程启动时快照的 PATH，而不是 `resolveShellEnv()` 后来修好的实时
    `process.env.PATH`。
  - `"pwsh"` / `"powershell"` / `"cmd"`：字面钉到 `pwsh.exe` / `powershell.exe` / `cmd.exe`——不做
    存在性检查，不做静默替换。一个结果缺失的钉子会以与过期的自定义 `SHELL` 已相同的方式生成失败，而
    唯一带优雅替换的模式是 `auto`。同步的 PTY 生成就是权威的可用性检查：它不透明的原生失败会被包装
    成基于所选 shell 来源的稳定指引。客户端收不到任何原生错误文本、可执行文件路径或原始 `SHELL`
    值。
  - 非 Windows 平台不受影响：无论 `terminalWindowsShell` 如何都是 `/bin/bash`。
  `cmd.exe` 曾是无条件的先前默认；`"auto"` 的基线是 `powershell.exe`，一次平实的回退替换（随每个
  受支持 Windows 版本发布，因此没有新依赖），因为它比 `cmd.exe` 功能更强（行编辑、tab 补全、VS Code
  自己的默认），而对默认安装没有代价。**Windows 默认刻意从不查询 `ComSpec`/`COMSPEC`**：那个变量
  几乎总是操作系统自己的 `cmd.exe` 路径——它命名的是 `cmd /c`/批处理文件调用的解释器，而不是一个有
  文档的“选我的交互 shell”旋钮——因此把它排在 `powershell.exe` 之前会让几乎每个安装的默认保持
  不变；一个刻意换掉命令解释器的用户已经有 `SHELL` 覆盖，而第二个更弱的覆盖只会是死代码。VS Code
  自己的默认 Windows 终端配置文件检测也做了同样的选择。
- **已知且已接受的限制：PowerShell 削弱忙碌关闭检测。** `closeTerminalTab` 的忙碌检查
  （`shellBusy.ts`）统计 shell 的操作系统子进程。许多 PowerShell cmdlet 在进程内完成工作而不是派生
  外部程序——`sleep` 是 `Start-Sleep` 的内建别名，它在 `powershell.exe` 自身内部阻塞并派生零个子
  进程，不像 `cmd.exe` 的外部 `sleep.exe`——因此一个真正忙于某个原生 cmdlet（`Start-Sleep`、
  `Invoke-WebRequest`、`Copy-Item`、一个 dot-source 的脚本）的 PowerShell 终端可能被读成空闲，并
  在无警告的情况下被强制关闭。日常开发工具的调用（`git`、`npm`、`bun`、`python`、`docker`、
  `node`、……）仍是外部可执行文件，仍被正确检测——这是一个真实但狭窄的缺口，与
  `shellBusy.test.ts` 的“无法作答的平台报告不忙”案例同属已接受限制的类别。遇到它的用户可以选
  **Command Prompt (cmd)**，拿回完全可检测的先前行为。
- **不构建 `listAvailableShells` 式的宿主探针**来在 Settings 选择器里把未安装的选项置灰，与
  `editors` 的 `listAvailableEditors` 不同：`auto` 已经拿到大多数用户想要的优雅替换，而一个探针
  无法证明 ConPTY 的可启动性，还会重复真实的生成检查。一次失败的启动反而保持可通过客户端的
  Settings 和 Retry 动作恢复。
- **macOS 的 PTY 以登录模式（`-l`）启动用户的 shell**，以匹配 Terminal.app 与平台的终端约定；其他
  平台保持一个普通的交互式 shell。PTY 自身提供交互性，因此不需要显式 `-i`。
- **PTY 环境是用户的环境，不是启动器的**（`ptyEnv.ts`）。宿主继承启动它的任何东西——一个终端模拟器、
  一个 IDE、`turbo`、Claude Code——而一份逐字副本会把那套上下文交给每个 shell：
  `TERM_PROGRAM`/`TERMINAL_EMULATOR`/`ITERM_*`/`VSCODE_*`/`TMUX`/`CLAUDECODE` 让工具去适配一个并不
  存在的终端，而 runner 设置的 `FORCE_COLOR`/`CLICOLOR_FORCE`/`DEBUG_COLORS` 会覆盖工具自己的颜色
  检测。这些、一个正在启动的 pi 的标记（`PI_CODING_AGENT`、`AI_AGENT`，以及 bash 工具的
  `PI_SESSION_ID`/`PI_SESSION_FILE`/`PI_PROVIDER`/`PI_MODEL`/`PI_REASONING_LEVEL`）、宿主自己的变量
  （`THINKRAIL_*`、`ELECTROBUN_*`、`JITI_*`、`BUN_PTY_LIB`）、逐 shell 的记账（`PWD`、`OLDPWD`、
  `SHLVL`、`_`）以及父终端的 `WINDOW`/`TERMCAP`/`COLUMNS`/`LINES`（bun-pty 只会从它自己的
  `process.env` 默认值里剥离这些，而一份被复制的映射绕过了那一步）都被丢弃。VS Code 的 askpass 覆盖
  随其支撑变量一起走：一个 `VSCODE_GIT_ASKPASS_*` 变量会丢弃 `GIT_ASKPASS`，一个
  `VSCODE_SSH_ASKPASS_*` 变量会丢弃 `SSH_ASKPASS`/`SSH_ASKPASS_REQUIRE`——否则成了孤儿的
  `askpass.sh` 会以空输出退出 0，而 Git 会接受空白凭据而不是提示输入。一个没有 VS Code 支撑变量的
  辅助程序是用户自己的，会被保留；终端把自己标识为 `TERM=xterm-256color`、`COLORTERM=truecolor`、
  `TERM_PROGRAM=ThinkRail`。`NO_COLOR` 被保留：它是用户的长期偏好，不是某个 runner 的覆盖。其他
  `PI_*` 名字是用户的 pi 配置，会被保留，绝不按前缀过滤：启动时或经 Settings → Environment 设置的
  `PI_CODING_AGENT_DIR` 必须能到达终端中运行的一次 `pi`，否则它会悄悄切到 `~/.pi/agent`（auth、
  models、settings、sessions），而宿主使用的是配置的文件夹（[[submodule-server-environment]]）。
  在 Unix 上丢弃是安全的，因为 shell 会重新读取用户自己的启动文件。过滤后的映射是子进程的*整个*
  环境：bun-pty >= 0.5 不会把它合并到宿主的原生环境之上（0.4 会，因此宿主启动时带着的某个变量无论
  过滤器怎么说都会到达 shell）。在 Unix 上，`USER` 与 `LOGNAME` 从 `id -un` 盖章，每个进程从固定的
  系统路径（`/usr/bin:/bin:/run/current-system/sw/bin`，绝不是启动器的 `PATH`）在短超时之下解析一次：
  PTY 不注册 utmpx 会话，因此一个没有它们的 shell 会去问 `getlogin()`，而那会读到一个被复用的 pty
  设备的陈旧记录，并可能报告另一个用户（#427）。Bun 的 `os.userInfo().username` 不是替代品——它读
  `$USER`，并在无法解析 uid 时回退到 `unknown`。当 `id` 失败时，继承来的值被保留，而不是被一个猜测
  替换。
- **一个 shell 以 `(workspaceId, tabKey)` 为键**，绝不是以 socket、客户端或组件为键。`tabKey` 是持久
  且由客户端提供的；PTY id 是逐次运行的，**永不持久化**（附加到一个比其进程活得更久的 id 正是 Theia
  的 `Couldn't attach - can't find terminal with id`）。
- **目录预留与 PTY 附加是分开的。** `reserveTerminal` 幂等地记录一个客户端铸造的标签页身份而不生成
  进程。一次新预留是事务性的：校验并插入、持久化完整目录，然后发布成员关系；持久化失败会移除内存中
  的插入且什么都不发布。`attachTerminal` 仍是幂等的 get-or-create，也是 PTY 诞生的唯一方式。这不是
  一个活性分割：客户端仍绝不持有指向一个运行中 shell 的唯一指针。
- **默认终端是一个宿主组装的创建握手，而不是布局播种。** `host` 看到工作区持久的待定标记，用其确定
  性 key 调用 `reserveTerminal`，并只在目录持久化成功之后才要求 `workspaces` 清除该标记。重试是幂等
  的；稍后关闭那个终端无法重建它，因为标记已经清除。本模块从不导入工作区注册表，也不选择前端摆放。
- **持久终端身份是有界的。** 一个工作区目录最多容纳 256 个标签页；key 非空且最多 500 个字符，标题
  非空且最多 1000 个字符。预留与新附加共享这些检查。复活会截断超大目录、丢弃非法 key，并在暴露
  宿主权威的成员关系之前修复非法标题。
- **在任何 PTY 或目录工作之前校验终端输入与网格尺寸。** 终端输入必须是字符串，因为 PTY 的 `write`
  会把一个数字变成裸 `TypeError`、把数字数组变成裸字节。附加（当它提供尺寸时）与 resize 要求
  `cols` 与 `rows` 是 1 到 32,767 的整数。xterm 的 fit 至少发送 2x1，而 32,767 是 Windows ConPTY
  有符号 16 位尺寸字段能容纳的最大值，因此真实屏幕永不会碰到任一端。这不是一个安全边界：一个能
  resize 的客户端也能输入命令。
- **被跟踪网格的更新是仅变更的。** 每个活跃条目跟踪在生成时或由最后一次成功 resize 所应用的网格。
  附加与显式 resize 只在网格变化时推进它，失败的调用不推进。一次重新附加仍可能执行一次*瞬时*重绘
  轻推（见下文），它不触碰被跟踪的网格。即便是一次同网格 resize 也能通过 `SIGWINCH` 唤醒 shell；
  而在附加快照之后发出的重绘会覆盖刚重放的行。
- **附加到一个未变网格仍会轻推前台应用**（`nudgePtyRedraw`、`ptyGrid.ts`）：对相同尺寸的
  `TIOCSWINSZ` 是内核空操作——没有 `SIGWINCH`，因此一个全屏 alt-screen 应用（vim、htop、一个交互式
  CLI）在一个被切走又切回的标签页后面继续运行时收不到重绘信号，而记录器从未捕获其 alt-screen 内容
  来替它重放（见下文）——该标签页否则会显示一个冻结的 alt-screen 之前缓冲。`attachTerminal` 的
  重新附加分支只在上面那次真实 resize 是空操作时才 resize 到 `cols - 1`，强制一个真正的
  `SIGWINCH`；把尺寸恢复回 `cols` 的那次 resize 被**延迟**（`NUDGE_RESTORE_DELAY_MS`，
  `setTimeout`），而不是与第一次背靠背触发。在同一个 tick 里的两次 `pty.resize()` 调用，从子进程
  的角度看是一次可观察的迁移——它自己的事件循环在两者之间从未被调度——因此一个 ncurses 风格的应用
  最终看到“与之前相同的尺寸”，并做它正常的*增量*刷新（对着它上次绘制的内容做 diff）而不是一次完整
  的清除并重绘；那会把新内容画在客户端当前显示的陈旧/空白缓冲之上，而不是替换它——肉眼可见地错乱，
  不只是陈旧，而且比什么都不做更糟。延迟的恢复给子进程一个真实的调度机会，去观察——并在其上完整
  重绘——那个中间尺寸，然后第二次 resize 才落地。恢复在触发前检查活性
  （`terminals.get(id) === entry`），因为标签页可能在这段间隔中关闭或重生，它还会检查被跟踪的网格
  仍是它捕获的那一个：否则一个在延迟期间落地的真实 resize 会被恢复撤销，让 PTY 停在旧尺寸而被跟踪
  的网格却说不是——而仅变更规则随后会让重试新尺寸成为空操作。这与上面那条要点已接受的“附加快照
  之后的重绘”是同一类竞态，只是刻意在每次重新附加时触发，而不是仅在偶然情况下。
- **属主是宿主的属主，而不是浏览器页面。** 任何客户端都可附加；这与 `history`、`todos` 和
  `templates` 一致，它们都假定单属主宿主。后果是：shell 在一次重载、一个关闭的浏览器和一个不同的
  浏览器之后仍然存活。
- **附加是独占的，并带接管。** 一个 PTY 只有一个尺寸，因此一次新附加成为接收者，而前一个客户端收到
  `terminal.detached`。若将来需要镜像，那是加法式的。
- **只有已附加的客户端可以驱动一个终端。** `writeTerminal`/`resizeTerminal` 接收调用方，否则就是
  空操作：一个被顶替的客户端仍持有一个有效的 PTY id，而一次重连会重放它排队的帧，那些帧会落到现在
  持有该标签页的人那里。收回是一个显式手势，如同 `tmux attach -d` 中那样。这样的调用方会被**再次
  告知它已被分离**——最初的告知是 fire-and-forget 的，可能丢失（一个在接管期间正在重连的客户端会
  重放它的附加并拿回缓存的成功响应），所以在第一次按键时得知，正是阻止一个标签页看起来还活着而什么
  都没发生的东西。客户端还用附加世代守卫反向顺序，因此一次过期的附加响应永远无法清除一次更新的
  分离。
- **输出保持寻址**，绝不广播——一帧只会到达一个已附加的客户端。标签页*目录*是例外：哪些终端存在是
  共享的领域状态（架构 #12），因此每次预留或移除都在 `terminal.tabs` 上作为一个幂等的逐工作区快照
  扇出。
- **一个 shell 恰好因五种原因死亡：** 标签页关闭、工作区归档、自然退出、宿主停止、附加时的孤儿清扫。
  卸载一个视图什么都不杀。
- **没有空闲剔除。** 终端“活动”只能指最后一次 PTY I/O，因此一个安静的长时命令会在中途被剔除
  （Jupyter 的 `cull_inactive_timeout` 正是这么做的）。**也没有被遗弃客户端的回收。**
- **跨宿主重启是复活，而不是重连。** shell 无法在其中存活。**成员关系在每次变更时持久化**（打开 /
  关闭 / 归档），而不只在 `stop()` 时——宿主没有崩溃隔离，因此非优雅退出是普通路径，而一个只在关闭
  时写的文件会复活一个已关闭的标签页并为它生成 shell。`stop()` 还会在 `closeAllTerminals()` 之前
  捕获一整组录制；`reviveTerminalSessions()` 恢复那些首次成功附加就生成一个新 shell 并显示旧画面的
  标签页。一次失败的生成不消耗待定的录制，因此修正 shell 并重试同一个标签页仍能恢复它。录制是尽力
  而为的，因此一次不干净的退出会交回正确的标签页，但屏幕是空白的。
- **不是 tmux。** 那会以我们无法在 Windows 上假定的依赖、一个竞争性的标签页模型、环境传播破坏以及
  `capture-pane` 轮询为代价，换取重启存活。我们本来就接受没有崩溃隔离。

## 限制（Restrictions）

- **`attachTerminal` 及其处理器必须保持同步。** 在同一个 tick 内查找并插入，正是让附加在 Bun 的单一
  事件循环上成为原子的原因；两者之间一个 `await` 会重新引入双重生成进程。
- **`closeTerminalTab` 在同一个同步趟中检查 `busy` 并杀死。** 一个被分开询问的问题会让一个在此期间
  启动的进程在无人告知的情况下死去。
- 记录器规则：原始字节（不是序列化网格）；永不重放 resize 事件；重新发出观察到的私有模式，**鼠标
  跟踪（1000/1002/1003/1006）除外**——那些属于一个活跃的前台程序，而一个退出时没有自己发 `DECRST`
  的 TUI 会让最后观察到的值卡在 `on`，因此在一个裸提示符下把它重新发出到一个新的 xterm，会把每次
  鼠标移动变成 shell 回显成垃圾的 SGR 报告；**永不记录 alt 屏幕**，而是把它作为一条*流*跟踪，因为
  一次切换可能被 PTY 读取拆开，且两个屏幕可能出现在同一次读取中；**永不记录模式序列本身**（重放
  `?1049h` 会把新终端翻到 alt 屏幕）；在绑定时一次写入中应用。**`restore()` 解析它拿到的东西，而
  不是复制它**——持久化的字符串是一个 `snapshot()`，因此一次逐字复制会把它的模式序言搬进正文，那里
  它不再是可重新推导的摘要，而变成每个后续快照都会重放并重新持久化的字面字节：一个坏模式于是比
  观察到它的那次运行活得更久，跨越重启。
- 附加交回录制，然后**丢弃**持有的 batcher 输出——重放已经包含它。

## 验证（Validation）

- `outputRecorder.test.ts`——边界、行/转义安全的裁剪、alt 屏幕排除（包括一次被拆到多次读取中的切换
  与一次读取中的进入+退出）、模式恢复、鼠标跟踪永不恢复，以及 `restore()` 把模式序列挡在正文之外
  （包括一份由仍会重放它们的宿主持久化的录制）。
- `outputBatcher.test.ts`——批次、背压、截断、`reset`。
- `shellBusy.test.ts`——子进程检测，包括一个无法作答的平台报告*不*忙。
- `shellArgs.test.ts`——跨 Unix 与 Windows 的 shell 可执行文件优先级，加上平台特定参数；`auto` 经
  注入的 `WhichFn` 做 pwsh-然后-powershell 解析；每个显式偏好都是字面钉子，无论 `which` 报告什么；
  `ComSpec`/`COMSPEC` 不再影响结果。
- `ptyEnv.test.ts`——启动器身份（包括一个正在启动的 pi 的标记）、颜色强制与宿主内部变量被丢弃，而
  用户自己的变量（包括 `NO_COLOR`、`PI_CODING_AGENT_DIR` 与其他 pi 配置）存活；父终端的
  `WINDOW`/`TERMCAP`/`COLUMNS`/`LINES` 被丢弃；VS Code 的 askpass 覆盖连同其支撑变量成对离开，而
  独立辅助程序留下；`USER`/`LOGNAME` 只在 Unix 上来自账户，无法解析账户时被保留；解析出的名字映射
  回宿主 uid；一个由*以泄漏变量且无 `USER` 启动的*宿主生成的真实 PTY 只看到过滤后的环境，其账户
  经 `id -u` 而非实现自己的查找检查。
- `terminalManager.test.ts`——事务性的持久预留而不生成进程、目录边界、拒绝畸形写入数据以及 resize
  与附加上的越界或非整数网格尺寸、附加幂等（包括并发）、接管、被顶替客户端拒绝、标签页列表广播、
  关闭/忙碌、复活，以及一个真实 shell——其 `USER`/`LOGNAME` 在泄漏了 `LOGNAME=root` 的情况下仍映射
  回宿主 uid，并且它仍看到宿主的 `PI_CODING_AGENT_DIR`。重放持久化与自然退出用例使用有界的、
  发布者可观察的数据/退出条件作为就绪边，绝不以经过时间为准；它们期望的输出标记从不在命令输入中
  连续出现，因此终端回显无法冒充命令执行。“任何输出”不是一个重放就绪边：一个 shell 的第一帧可能
  只有模式（bash 的 `ESC[?2004h`），而记录器从不存储它，因此一个复活测试只在其打印的标记到达之后
  才持久化，并断言重放包含它。一次期待“不忙”的非强制关闭也先等待一个打印的标记：在启动完成之前，
  rc 文件会运行短的子进程（`$(groups)`、`lesspipe`、`dircolors`），它们会正确地让该 shell 读作忙碌。
- `e2e/terminals.spec.ts`——快速重入回归、重载存活、第二客户端接管、跨客户端标签页收敛、寻址输出
  隔离。它的输出标记同样从不在被输入的文本中连续出现：按键回显先到达屏幕，并被拆到多个输出批次中。
