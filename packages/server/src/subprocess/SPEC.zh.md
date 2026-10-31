# subprocess —— 有界子进程

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

在墙钟预算内跑完一个子进程，并返回它写下的内容。“这个宿主如何 spawn 一个可能永不返回的东西”的
唯一答案——只负责生存期机制，绝不负责某个子进程输出意味着什么。

## 边界（Boundary）

- **拥有：** `runBounded(argv, { timeoutMs, cwd?, env?, stdout?, signal? })` →
  `{ ok, out, err, timedOut, launchFailed, waitedMs, aborted? }`：捕获两路流并在子进程**退出**时
  完成；stdout 默认为解码文本，而 `stdout: "bytes"` 把它保留为 `Uint8Array`，供读取不透明内容的
  调用方使用（stderr 仍是文本诊断）。POSIX 子进程是分离的进程组首领，而 Windows 子进程保留一个非可见
  控制台。到期或取消会杀掉 POSIX 进程组；Windows 异步运行 `taskkill /PID <pid> /T /F`（五秒预算），
  等它退出，taskkill 失败时回退为杀直接子进程。启动失败是一个结果（`ok: false`、`launchFailed: true`、
  启动错误作为 `err`），绝不是抛出；调用方能区分基础设施失败与一个确实运行过但非零退出的子进程。
  预先中止的 `signal` 什么都不启动。在飞中止与退出和到期竞争。中止与到期都会停掉整棵树并在返回之前
  等待直接子进程退出，因此调用方可以安全清理部分 clone 的目录。只有取消携带 `aborted: true`
  （`ok: false`，既非超时也非启动失败），并以同样的排空宽限保留已捕获的 stdout/stderr。
  一旦退出或到期胜出，之后的中止就无效；监听器在排空之前被移除。
- **也拥有：** `streamBounded(argv, { timeoutMs, cwd?, env? })` → `{ stdout, exited }`——同样的
  spawn、时限、组杀与 stderr 捕获，供那个必须在子进程仍在产出时转发其 stdout 的调用方使用
  （宿主的 `/blob` 路由）。完成仍是子进程退出，并带与 `runBounded` 相同的“孙进程持有管道”规则：
  子进程退出之后仍处于 pending 超过 `DRAIN_GRACE_MS` 的读取会被取消，因为那时仍未到达的东西不是我们
  子进程的输出；在退出*之后*发出的读取从发出那一刻起获得同样的宽限，因此排空管道余量的慢消费者永不
  被提前切断——背压保留到退出为止，只有挂起被限定。`stdout` 流**只在**零退出之后**关闭**，
  并在非零退出、到期或启动失败之后**报错**，因此消费者永不能把截断的转发误认为完整的转发；
  取消该流会杀掉子进程。`exited` resolve 为 `runBounded` 返回的同一形状，减去 `out`。
  流式不接受中止信号；消费者改为取消它的 stdout 流。
- **公开接口：** `runBounded`、`streamBounded`、`BoundedRun`、`BoundedBytesRun`、`BoundedRunOptions`、
  `BoundedBytesRunOptions`、`BoundedStream`、`BoundedStreamOptions`。
- **允许的依赖：** Bun/Node 的进程 API。别无其他——它不认识任何特性、任何线上类型、任何持久化。
- **禁止：** 消息措辞、重试、截断，或任何针对特定程序的策略。`git` 的停滞/stderr 文本与
  `branch-review` 的降级为 `null` 都是调用方的事。
- **已知的非消费者。** 有两个调用方会 spawn 可能比人的耐心更久的东西，并且仍然自己动手，各自都是
  刻意的，也各自是一笔常驻债务而不是定案的设计：
  - `@thinkrail/shared/jbcentral`——已经以时限为先，且契约不同（有界字节读取、闭合的结果词汇表、
    可注入的 `run` 接缝、Central 的保密规则）。把它折进来会为另一个包里的一个调用方扩大本接口。
  - `dialog` 的 `selectDirectory`（`dialog.ts`）——仍是本模块取代的那种以 EOF 为门的形状，围绕
    `zenity`/`kdialog`/`osascript`，其 fork 出的助手继承 stdio。未迁移是因为选择器真实的预算就是
    一个人坐在对话框前：把它的生存期接到 runner 的取消接缝上仍被延后。

  `github` 的 `githubAuthStatus`（`github.ts`）曾是第三个——在单一事件循环上一次 `spawnSync` 的
  HTTPS 调用——直到它搬到 `runBounded` 上，这也是 `ghSetupProblem` 不再手搓自己的
  TERM+KILL 升级的原因：两个 `gh auth status` 探测现在共享本模块的超时机制，而不是复制它们。

  上面的清点是关于*无界等待*的，而不是仓库里的每次 spawn——其余要么已经有界，要么不能等待任何东西：
  `git` 的同步 `git()` 与 `terminal/shellBusy` 走 `@thinkrail/shared/spawn` 的
  `spawnSyncCaptured`（`shared/shellEnv` 保留它自己带计时的 `Bun.spawn`——win32 上是空操作），
  而 `editors`、`cli/bootstrap` 与 `cli/powershell` 是 fire-and-forget 的
  `@thinkrail/shared/spawn` `spawnDetached`，没有人读它们的输出。`trash` 的助手 `execFile` 是唯一
  真正的落伍者：无界，但一个本地 trash 助手既无网络也无提示，因此它违背的是本模块的字面而非其目的。

## 必须做对（Get right）

- **完成是子进程退出，绝不是它的管道读到 EOF。** 一个继承子进程 `stdout`/`stderr` 并比它活得更久的
  孙进程会保持写端打开，因此以 EOF 为门的读取永不返回——而因为该读取*就是*完成信号，一个毫秒内成功
  结束的子进程被报告为满预算超时（`git` + 一个后台化的 `ssh`，issue #209 的第二副面孔）。
  `Bun.spawn` 自己的 `timeout` 选项以同样方式门禁，不是替代方案。
- **时限与子进程的*退出*赛跑，而排空宽限在那场赛跑判定之后才运行。** 宽限不是预算的一部分：把时限与
  *宽限之后*的结算赛跑，反而把每个预算最后的 `DRAIN_GRACE_MS` 变成假超时窗口，因为一个持有管道的
  孙进程每次都让宽限跑满。一次在 55s 预算的第 54.9s 作答的 `git fetch` 会以停滞措辞与 `ok: false`
  返回，而其健康的进程组在退出途中被 SIGKILL。由一次测试钉住：在 250ms 预算下，子进程紧跟一个持有
  管道的孙进程立即退出。
- **排空宽限（250ms）在退出、到期与取消时都运行。** 它无法截断子进程自己的输出：管道至多持有其缓冲，
  而写入超过缓冲的子进程会*阻塞*直到读者排空它——因此在退出时剩下的东西都已缓冲，在微秒内排空；
  宽限只覆盖调度延迟。之后到达的任何东西只能来自不是我们子进程的进程——而仍会被当作子进程的输出捕获，
  这是已知的瑕疵，不是保证。该常量从另一侧也被钉住：一个测试断言每条路径在孙进程让管道永不 EOF 时
  *等完它*（且不再多等），而那是唯一能观测到它的情形。
- **超时路径也会排空——当场取消读者会丢掉诊断。** 组杀才是关闭写端的东西，因此到期时管道中仍在的字节
  正是调用方消息的原料：git 的 `Permission denied (publickey)`、代理的拒绝、证明大传输仍在进行的一行
  `remote:` 进度。先杀然后在同一同步步骤里取消，会让一个 pending 的 `read()` 与那次取消赛跑并可能丢掉
  最后一块——一个承诺“kill 之前 git 写过的任何内容”的调用方却丢掉了 git 的最后一句话。kill 之后读取
  几乎立刻命中 EOF，因此宽限在正常情况下零成本，并给病态情况设上限（一个通过自己的 `setsid` 逃出进程组
  的持有者，测试正是用它让该等待可观测）。
- **`timeoutMs` 在到达 `setTimeout` 之前被夹取。** 类型说是 `number`，而 `setTimeout` 会静默把
  `Infinity`、`NaN`、负数和任何 ≥ 2^31 的值塌成约 1ms——因此 `Infinity`（“无预算”的自然写法）买到了
  立刻超时、对一个健康子进程立即组 `SIGKILL`，以及宿主 stderr 上的一条 `TimeoutOverflowWarning`。
  非有限值现在意味着 `setTimeout` 能持有的最长预算（约 24.8 天）——这是“无预算”唯一不说谎的解读。
  `NaN` **不**享该解读：“不是数字”是坏掉的调用方，而坏预算的安全失败是立刻到期，不是等 24 天。
  有限值夹取进 `[0, 2^31-1]`，而负预算立即到期，因为它本就已经到了。
- **POSIX 到期时杀进程组——释放描述符的是这一点，不是计时器。** `detached: true`（`setsid`）让子进程
  成为组首领，因此 `process.kill(-pid, "SIGKILL")` 会收割仍持有我们管道的孙进程；读取随后自行命中
  EOF。改为放弃读取才是让 Bun 事件循环被孤儿钉住整个生存期的原因——实测对着一个 unref 的 400ms 计时器
  占据了 20s，且每次停滞约泄漏 1.4 个描述符。收割进程组则什么也不留下：没有存活的孙进程、没有描述符
  增长，宿主可以自由退出。
- **退出路径永不杀进程组。** 那里的孙进程是健康且刻意的——一个 `ssh` `ControlMaster`、一个凭据缓存
  守护——杀掉它们会让用户为一次*成功*的调用失去其多路复用连接或缓存凭据。改为取消读者。
- **`setsid` 丢掉控制终端——这是刻意的权衡，不是白得的胜利。** 子进程不能再打开 `/dev/tty` 来提示，
  这正是把 issue #209 从 55 秒等待变成大多数用户立即得到 `Permission denied (publickey)` 的原因；
  `SSH_ASKPASS` 不受影响（无 tty 正是它的触发条件），这也是调用方的预算仍须按一个人坐在对话框前来
  定尺寸的原因。**它的代价是真实的：** CLI 入口是一个从终端启动、仍然占据前台的 `thinkrail` bin，
  坐在那儿的用户此前能*看到* `ssh` 的 `Enter passphrase for key …` 打在 `/dev/tty` 上
  （ssh 直接读 tty，因此我们管道化的 stdio 从未遮住它）并把它打进去。他们不再能这样做。接受它是因为
  UI 是浏览器客户端，而一个隐藏的终端提示是它永远无法展示、等待或报告的流程——快而可读的失败胜过
  只有启动 shell 才看得见的提示。它还让子进程落在宿主终端的信号组之外，因此结束一次停滞调用的是预算，
  永远不是宿主上的 `Ctrl-C`。把 passphrase 请求呈现在 UI 中（#209 的首要诉求，相对于它的“至少”条款）
  仍未建；runner 的取消接缝本身并不提供提示 UI。
- **Windows 子进程是隐藏的，不是分离的。** `windowsHide: true` 配 `detached: false` 为普通后代保留一个
  非可见控制台以便继承。分离会移除该控制台：直接子进程仍隐藏，但它启动的助手可能分配一个可见控制台。
  这就是为什么一个聚焦触发的 `gh` 查询尽管隐藏了直接子进程，仍可能闪现 `tzutil`、Git 与 SSH 窗口。
  Windows 不支持 POSIX 的组杀路径；到期与流取消使用与中止相同的异步 taskkill 树终止。直接子进程回退
  无法保证终止后代。非分离子进程参与 libuv 的正常作业生存期；本 runner 用于有界的宿主自有工作，
  而不是独立存活的守护进程。原生 Windows 回归使用无控制台驱动、真实 runner 与一个普通控制台孙进程，
  并带一个分离的阳性对照。它检查原生控制台所有权/窗口状态，而不是选项的存在、输出成功或浏览器 DOM。
- **env 默认是活的 `process.env`，不是启动时的快照**——`boot` 的 `resolveShellEnv()` 在启动*之后*
  通过变更 `process.env` 修复 `PATH`/`LANG`，而一个从快照 spawn 的子进程会静默错过该修复。
  两半分别被钉住——调用方的 `env`/`cwd` 到达子进程，以及启动后的变更对一个未传 `env` 的 spawn 可见
  ——因为只断言默认值的测试无法区分二者。
- `stdin` 永远是 `ignore`，两路输出流永远被管道化并**从 spawn 那一刻起被读取**；文本增量解码，
  字节 stdout 不解码直接累积——一条未排空的管道正是健谈子进程死锁的方式。读者在结果判定*且*上述宽限
  跑完之后才被取消，绝不提前。每个计时器都被 `unref`，因此有界等待永不把关停拖住，而 `waitedMs`
  来自 `performance.now()`，因此时钟调整无法渲染出负的等待时间。
- **两路流都不设上限。** `runBounded` 在预算允许的时长内累积子进程写的一切，因此调用方的字节上限
  （`git` 的 2000 字符 `boundedStderr`）约束的是到达客户端的内容，绝不是本模块在内存中持有的内容。
  对 `git`/`gh`/`glab` 足够，那里的最坏情况是数百 KB 量级的进度日志；一个可能无界喷吐的调用方需要
  这里的 `maxBytes` 选项，而不是事后截断。
