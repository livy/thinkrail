# 共享的服务端工具集

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

引擎宿主及其各启动器所用的横切运行时工具集。仅限服务端——绝不打包进 `apps/web`。
通过显式的子路径导出暴露，不使用 barrel。

## 边界（Boundary）

- **拥有：** 既不属于引擎、也不属于传输层的宿主侧运行时辅助函数，以及两个启动器都要产出的原生品牌图标集
  （`assets/` 及其生成器 `scripts/generate-icons.ts`）。
- **公开接口：** `@thinkrail/shared/shellEnv` → `resolveShellEnv()`、`repairShellEnv()`、`mergePath()`、
  `loginShellImports()`、`parseLoginShellEnv()`、`LOGIN_ENV_MARKER`、`localeRepair()`、`launchEnvKeys()`；
  `@thinkrail/shared/freePort` → `findFreePort()`、`isPortFree()`；
  `@thinkrail/shared/startupMark` → 静态递归字标，加上每个启动器都用到的纯函数式响应式/ANSI 渲染器
  与交互式输出门禁；
  `@thinkrail/shared/version` → 由 CLI、桌面端与宿主溯源共同使用的内建 `{ version, channel, commit }`
  发布身份（源码中以提交值 `0.0.0-dev` 为默认）；
  `@thinkrail/shared/windowsIdentity` → `windowsFileIdentity()`，即两个启动器写到各自可执行文件上的
  Windows 文件属性；`windowsProductName()`，仅返回按渠道区分的产品名；以及 `windowsIconPath`，
  即它们嵌入的 ICO；
  `@thinkrail/shared/paths` → 相对于工作树的路径约定（`WORKSPACE_INTERNAL_DIR`、
  `WORKSPACE_CONTEXT_DIR`、`WORKSPACE_TODOS_DIR`）；
  `@thinkrail/shared/codedError` → `CodedError` + `errorCodeOf()`；
  `@thinkrail/shared/toolHomes` → `toolHomeEnv(home)`，即把每个 agent 工具（Claude Code、Codex、
  Gemini、Copilot、Kimi、Grok）的配置 home 都指向同一个沙箱 home、并清除会越出该 home 的覆盖项的那份
  环境——也就是每个封闭式宿主 spawner（e2e、workflow harness、artifact probes）都应用的那一份列表，
  从而新的检测来源不会泄漏开发者真实的登录态；
  `@thinkrail/shared/removeTree` → `removeTree()`，即子进程曾在其上运行过的目录树在每次拆除时所走的、
  带重试的递归删除；
  `@thinkrail/shared/spawn` → `spawnSyncCaptured()` + `spawnSyncCapturedBytes()` + `spawnDetached()`，
  即宿主或 CLI 启动的每个控制台子进程都要经过的隐藏子进程接缝；
  `@thinkrail/shared/jbcentral` → 原生 Central CLI 适配器：绝对可执行文件/版本/状态的探测；最低支持版本
  与全局不透明的 PI 扩展路径；单向的鉴权判定；产物位置观察器；`add pi` / `remove pi` / `login` /
  `update --install` 动作；以及按操作系统区分的官方安装方案。它绝不修改 PI 模型或凭据配置。
- **/spawn** — `spawnSyncCaptured()`（同步捕获 → `{ launched, exitCode, stdout, stderr }`）、
  `spawnSyncCapturedBytes()`（相同的启动与退出语义，但 **`stdout` 保持为字节**，供需要读取二进制子进程
  输出的调用方使用——UTF-8 解码器会替换这类输出；stderr 仍为解码后的文本，
  因为诊断信息始终只按文本读取），
  以及 `spawnDetached()`（fire-and-forget `unref`），都建立在 `Bun.spawnSync` / `Bun.spawn` 之上，
  且始终 `windowsHide: true`。Bun 1.4.0 会把该选项映射到 libuv 的 `UV_PROCESS_WINDOWS_HIDE`；
  这个接缝防止后台控制台 spawn 漏掉它。两个定制的有界 runner（`server/subprocess` 的 `runBounded`、
  `jbcentral` 的版本/配额 runner）则直接设置它。
- **允许的依赖：** Bun/Node 运行时（`@types/bun`）；`contracts` 的**类型**（`JbcentralInstall`，
  即 `jbcentralInstall` 返回的线上形状——保留在线上数据里，以便服务端原样送达卡片）；图标生成器仅在
  构建期使用的 `@resvg/resvg-js` 与 `resedit`。
- **禁止：** 导入 `server` / `web` / 任何 `pi` 包；被 `web` 导入（它带有不得进入浏览器 bundle 的
  Bun/Node 代码）。

## 内容（Contents）

- **/shellEnv** — `resolveShellEnv()`（每个进程只跑一次并缓存结果；`repairShellEnv()` 是未缓存的
  worker，测试时注入探针来驱动它，而 `loginShellProbe(timeout)` 用适合测试的时限构建默认探针）：
  让宿主的环境对它启动的 shell 是安全的，因为由 GUI 启动的宿主（Finder/Dock、launchd、systemd unit、
  容器）继承的是一份被裁剪过的环境。四项彼此独立的修复：
  - **`PATH`** — 始终采用用户的登录 PATH，让进程内 agent 的 bash/工具以及它们触发的 git hooks 能找到
    `git`/`node`/`bun` 等，同时保留启动终端额外加在上面的内容。
  - **login-shell 环境** — 启动时按 VS Code 的做法一次性导入用户登录 shell 环境的其余部分，因为 GUI
    启动的宿主否则看不到任何 provider API key 以及 `models.json` / `mcp.json` 里 `${VAR}` 引用
    （常从厂商文档复制而来）所依赖的变量。当 `TERM` 已设置时跳过（从终端启动的宿主已经带着 shell 的
    环境）。进程中已设置的变量优先；`PATH` 仍走上面的独立修复；`PWD`/`OLDPWD`/`SHLVL`/`_` 以及
    `PI_*` / `THINKRAIL_*` 绝不导入（用户想要的 pi 配置目录或 `PI_*` 变量来自已存储的宿主环境，
    [[submodule-server-environment]]）。对登录 shell 做一次带标记分隔的 `env -0` 探测（`-l -i`，
    再退到 `-l`）同时服务于这次导入与 PATH 修复；标记（`LOGIN_ENV_MARKER`）之前的 rc 文件杂音被忽略。
    已记录的权衡：此后宿主、它启动的每个进程（含 stdio MCP 服务器）以及每次 `!command` 解析都能看到
    shell 导出的密钥——那是用户自己的密钥，与终端中一样，并非隔离。
  - **locale** — 当完全没有配置 locale 时（`LC_ALL`、`LC_CTYPE` 与 `LANG` 均未设置）把 `LANG` 设为
    UTF-8 locale。没有它，bash/readline 面向**字节**而非字符，一次退格就会删掉多字节字符（西里尔字母、
    变音字母、CJK）的一半并令该行失去同步。只设置 `LANG`，绝不设置 `LC_ALL`，从而用户的按类设置
    （`LC_NUMERIC`、`LC_TIME` 等）得以保留。
  - **`SSH_AUTH_SOCK`**（仅 darwin）— 缺失时，通过 `launchctl getenv SSH_AUTH_SOCK` 找回 launchd 的
    ssh-agent socket（3s 超时，从不抛错，launchd 未报告时为空操作）。没有它，宿主自身的 git push
    （`pr`）与每个内嵌终端都会失去用户的 ssh-agent：在 ThinkRail 终端里执行 `ssh-add` 会回答
    “Could not open a connection to your authentication agent”，而在 Terminal.app 里加载的密钥
    永远到不了宿主的 push。

  所有修复都落在 `process.env` 上，这正是它们能触达宿主启动的*每一个* shell 的原因：PTY 终端会复制它
  （`server/src/terminal`），进程内 agent 自己的 bash 亦然。
- **/freePort** — `findFreePort(preferred, host?)`：返回等于或高于 `preferred` 的第一个空闲端口，
  使宿主可以挑一个空闲端口，而不是与已在运行的端口冲突。`isPortFree(port, host?)`：底层的单端口检查。
- **/startupMark** — 启动器统一的开机签名：一个提交进仓库的 42×20 `TR` 轮廓，由循环的 `THINKRAIL·`
  字符场构成，再与产品身份、调用方提供的真实状态及其解析出的端点组合。在 72 列及以上它是右侧联排签名；
  42–71 列时改为堆叠；低于 42 列时只用身份/状态，而不是把图形折行。
  渲染只是一次同步写入的文本量：允许 ANSI 时用终端调色板的绿色 + 稀疏高亮信号 + 暗淡分隔符，
  `NO_COLOR`/`TERM=dumb` 时输出纯 UTF-8，stdout 非交互时不输出任何标记。根 dev runner 在派生并发任务前
  调用它（`starting`）；源码/编译版 CLI 在 `bootHost` 解析出实际 URL 后调用它（`host ready`）。
  SVG/图片转换从来不是运行时关注点。
- **/version** — 唯一的永久发布标记接缝。源码运行报告 `0.0.0-dev`；发布 CI 在其一次性 checkout 里、
  构建两类产物之前覆盖该模块，使 CLI `--version`、桌面端包元数据、analytics 与
  `server.welcome.appVersion` 不会漂移。它包含身份数据以及仅发布版才有的公开 `posthogProjectKey`
  （源码中为空，故未标记的构建永不上报），且不依赖任何启动器。
- **/windowsIdentity** — `windowsFileIdentity({ version, channel })`：Windows 为 ThinkRail 可执行文件
  显示的信息。产品名同时充当任务管理器显示的文件描述：stable 为 `ThinkRail`，nightly 为
  `ThinkRail Nightly`，未标记构建为 `ThinkRail Dev`。发布者与版权为 JetBrains s.r.o.，与 `LICENSE`
  和签名证书一致。版本号采用 Windows 要求的四个数字段：`X.Y.Z` 加上预发布版本的尾部数字
  （`0.3.0-nightly.6` → `0.3.0.6`），否则加 `.0`。未知渠道或任一段超过 65535 会抛错，
  使发布构建失败而不是产出错误的属性。CLI 编译、桌面端 `postBuild` 以及两个 Windows 冒烟测试
  （CLI 与桌面端）都由它派生，因此不会互相矛盾。
- **/assets** — 原生图标集：手工编写的 `icon.svg` 以及由它生成的位图。绝不手工编辑位图；用
  `bun run --cwd packages/shared generate-icons` 重新生成（`@resvg/resvg-js` 直接从矢量图光栅化，
  `resedit` 打包 ICO）。产出物包括 macOS iconset、Linux 512 px PNG，以及 Windows ICO，
  其路径由 `windowsIconPath` 导出。
  - **图形。** 标记保持圆角方形轮廓：深色系品牌绿（`#8dff4f`）的 ThinkRail 字形，落在纯黑方块上。
    排除半透明玻璃、高光与装饰性描边，以便小尺寸任务栏帧保持清晰。
  - **ICO 帧。** 以 PNG 压缩，尺寸 256/128/80/72/64/56/48/40/36/32/28/24/20/16：微软文档化的最小值
    （16/24/32/48/256），加上 64/128，以及 Windows 在 125% 到 250% 缩放之间会请求的每个大小图标尺寸。
    `LoadImage` 会挑选不超过请求尺寸的最接近帧，因此缺失的尺寸会被放大而显得模糊。早先的 4 帧 BMP ICO
    缺少 Windows 11 任务栏的 24 px 帧，在那里看起来就是模糊的。
  - **帧顺序。** 帧按从大到小排序，因为 Bun 的编译器只保留映射 icon #1 的具名图标组
    （oven-sh/bun#32428，[[module-cli]]）。
- **/codedError** — `CodedError(code, message)` + `errorCodeOf(err)`：携带线上 `WsErrorCode` 的错误，
  使客户端必须*针对性*响应的失败以名字而非待模式匹配的字符串传播。它放在这里，因为接缝两端都需要它、
  而任何一端都不得导入另一端：知道该失败的模块抛出它（当前为 `server/src/git`，用于消失的提交作用域），
  宿主的请求处理器把它读到 `WsResponse.errorCode`。
- **/removeTree** — `removeTree(path, options?)`：删除子进程曾在其上运行过的目录树，只要失败属于短等待
  即可解决的类型（`EBUSY`、`EMFILE`、`ENFILE`、`ENOTEMPTY`、`EPERM`、`EACCES`）就继续重试——
  默认十次尝试、线性 100 ms 退避。它存在的原因是 Windows 在子进程退出后异步释放可执行句柄，
  因此对冒烟/e2e 临时目录做一次裸的递归删除会在所有断言都已通过*之后*抛出 `EBUSY`；也因为
  **`rmSync` 自带的 `maxRetries`/`retryDelay` 在 Bun 下无效**——nightly 运行 33166583594 在打印
  `installer smoke OK` 后 252 ms 抛出 `EBUSY`，而当时的策略要求等待数秒。`EACCES` 在 nightly 运行
  33949193053 之后被加入可重试集合：同一个尚未释放的句柄在 Windows 上以 `EACCES` 出现的频率与
  `EBUSY` 相当。重试是拆除韧性，不是错误掩盖：超过退避仍被锁住的目录树会抛错，而任何等待都无法修复的
  失败（`ENOTDIR`、……）在第一次尝试就抛错。`options.remove` 是单元测试驱动的注入式删除器接缝；
  `attempts`/`delayMs` 让调用方用耐心换速度。
- **/paths** — ThinkRail 拥有的、相对于工作树的路径约定，集中命名一次以便当前与未来的使用方一致
  （当前：`workspaces` *创建*该临时草稿目录，git *忽略*它）：`WORKSPACE_INTERNAL_DIR`（`.thinkrail`——
  仓库内由宿主管理的目录，当前存放临时草稿，也是未来宿主文件（如缓存的规格索引）的预期归宿；
  **不**对文件树隐藏）、`WORKSPACE_CONTEXT_DIR`（它的 `context/` 临时草稿目录，用于临时文档）以及
  `WORKSPACE_TODOS_DIR`（`context/todos/`——聊天 TODO 计划，每个会话一个 JSON，因此与其余临时草稿一样
  是临时的）。不含 pi 的 `pi-todos/core` 无法导入本包（它保持可在原生 `pi` 下安装），因此它保留一份
  todos 路径的本地镜像；本模块是宿主侧的真相来源。它与 *home* 状态目录 `~/.thinkrail`
  （服务端 `persistence`）不同。（宿主播撒到临时草稿目录的 `.gitignore` *正文*——单独一个 `*`——
  是该调用点内联的一次性内容，不是路径，因此写在那里，不在这里。）
- **/jbcentral** — **唯一的 Central 进程/文件系统边界**。它按绝对路径解析 Central，把有界的
  `central --version` 结果解析成兼容性判定，仅暴露全局不透明产物路径
  （`~/.pi/agent/extensions/jetbrains-central.ts`，以及 `~/.local/bin/` 安装器回退）及其存在性，
  并且只调用经过评审的 argv：`status`、`quota --json`、`add pi`、`remove pi`、`login`、
  `update --install` 与 `proxy start --ensure-updated`。支持策略是**仅设最低版本**
  （`MINIMUM_CENTRAL_VERSION`，`1.4.0`——第一个携带原生 PI 接口面的 Central 版本）：等于或高于它的都
  支持，更低的版本需要更新，格式错误的输出视为不支持。这里刻意**没有上界**——较新的 Central 被假定与
  本适配器调用的 argv 向前兼容，而用上界设限会让用户被困在每一个 Central 版本之后。

  **鉴权与代理健康状态是来自同一次状态探测的、彼此独立且单向的观察。** 在 Central 交付所请求的 JSON
  状态接口之前，`central status` 在这些状态下都退出 0 并打印带样式的展示行。因此 `Auth` 判定只信任它
  单一的否定标记：`not connected` 表示**未登录**，该行的其他任何呈现（账号、许可、托管服务器，或我们
  没见过的措辞）都表示**已连接**，而该行缺失或探测失败表示**未知**。`Proxy` 判定同样只信任确切的
  stopped 标记为**已停止**；运行中、缺失或无法识别的呈现都收敛为**未知/未停止**。只有这些被正面观察到
  的否定事实才可以驱动恢复要求。这是唯一读取展示输出的地方，而共享的观察结果从不返回原始文本、端口、
  PID、URL、账号、许可、公司、服务器细节或诊断信息。日后换成 JSON 仍属于该边界内部的事；其他所有命令
  的输出仍永不被解析。按 Central 的设计，这次探测代价高昂（一次代理健康检查加一次网络更新检查，约 1.3s，
  且每次调用产生一个 CLI analytics 事件），因此 `JBCENTRAL_STATUS_TTL_MS` 界定调用方在重新探测前可继续
  使用一次观察结果多久；这里没有任何轮询。

  **配额是结构化、仅含数值的读取。** `readJbcentralQuota()` 解析同一个绝对可执行文件，只运行
  `quota --json`，并对 stdout 设界、设截止时间。它在内部解析 JSON，只接受有限的
  `tariffQuota.available` / `tariffQuota.maximum` 值（数字字符串或数字）作为周期性的 `remaining` /
  `total`；其余字段全部忽略。非法 JSON/结构/金额、超时、输出过大、启动失败或非零退出都成为封闭的原因码。
  这里没有文本输出回退，也没有缓存或轮询。原始输出以及账号、套餐、已用、充值、补足与诊断数据都被丢弃，
  永不返回、记录或持久化。

  **派生出登录进程不等于登录已启动。** `central login` 从终端 UI 驱动浏览器交接，因此没有 TTY 时它会
  立刻退出而不会发生任何登录——尽管 spawn 本身成功。于是启动流程会等一段宽限期以捕捉早期退出，并对任何
  非零退出报告 `launch-failed`，让调用方可以改为提供在宿主上运行的命令，
  而不是承诺一个永不会打开的浏览器。立刻的*零*退出是 Central 的“已登录”短路，计为已启动；
  真正开始的流程在宽限期结束时仍在运行，因为它在等浏览器确认。依赖接缝只暴露子进程的退出状态
  ——绝不暴露它的输出。

  **超时界定的是等待，而不只是子进程。** 杀掉子进程并不会关闭它自己的孙进程仍持有的一端管道——而 Central
  会派生一个代理守护进程——因此截止时间直接让调用返回，而不是先发 kill 再等待读取。这一条是承重的，
  因为版本探测位于宿主的启动路径上：那里无界等待意味着没有端口、没有 UI，也没有错误可展示。版本 stdout
  在内存中有界；动作的 stdout/stderr 被忽略。任何子进程输出都不被记录或返回，只有退出成功加上安全的后置
  条件才映射为封闭的适配器结果。`watchJbcentralArtifact(onChange)` 只观察那个经过评审的位置，并上报新增、
  删除与替换的失效事件；它通过从最近的已存在父目录重新布防来处理尚不存在的扩展目录，而一次仅检查存在性的
  轮询会修补丢失的 add/remove 文件系统事件。每次新安装的 watch 会再检查一次：当 watch 被移动期间出现了
  产物、或某个更深目录现已存在，它不会为此发出事件，因此观察器立刻失效或向更深处重新布防，而不是留给轮询
  （一次迟到的轮询命中曾在首次 Connect 已经稳定后重跑了运行时重建）。它绝不打开或取指纹生成的文件。
  调用方对事件去抖，并通过常规的检查 API 重新核对存在性。

  **目录事件不是产物事件。** `fs.watch` 是按目录的，而上面的重新布防意味着被观察的目录经常是一个*祖先*——
  当产物目录尚未创建时，观察器可能落在 `~/.pi/agent`，即 pi 的整个状态目录。因此失效判定以产物本身为
  门禁：事件只有在指名产物所在目录内的产物条目时才被转发，来自其他任何地方（祖先的变动、同级扩展）的具名
  事件至多触发重新布防与存在性复查。指名条目——而不是对文件做 stat 取指纹——正是让“替换”可被观察、
  同时永不读取所生成产物的方式。

  **事件按发出它的那个观察器分类，绝不按当前观察器。** 被观察的目录是可变状态，重新布防会改变它，因此一个
  已由后来被关闭的观察器排入队列的回调，否则会按继任者的目录来判定——把祖先的变动读成产物替换，或把产物
  事件读成祖先变动。所以每次 `arm()` 都把它自己的目录与一个单调递增的代次绑定进它安装的回调里，关闭句柄
  会使该代次退役，从而被取代的回调被丢弃而不是被重新分类。

  **无名事件是有作用域的，既非一律信任也非一律丢弃。** `fs.watch` 可能省略文件名，观察器错误也以同样方式
  显现，因此 `null` 不携带关于*什么*改变的信息——只关于我们当时在何处观察。在产物自己的目录上，它被当作
  可能的替换：句柄被关闭（它可能就是错误背后的死观察器），调用方被置为失效，因为就地重写不改变存在性，
  存在性轮询无法恢复它。来自祖先时则回退为只看存在性。这种不对称是刻意的：信任来自祖先的 `null` 会在任何
  省略文件名的平台上恢复那种活锁，而在产物目录上丢弃它则会静默地让过期的 Central 运行时搁浅直到重启。

  跨模块的活性义务与复盘见 [[central-integration]]；观察器机制由
  `packages/shared/src/jbcentral.test.ts` 钉住。

  该适配器刻意没有针对此前未发布集成的迁移路径：它绝不读取或编辑 `models.json`、`auth.json`、备份，
  或任何无关的 PI 状态。

  **安装指引按 OS 区分且单一来源：** `jbcentralInstall(platform)` 返回被送到卡片上的官方宿主 OS 方案；
  远程浏览器绝不猜测自己的 OS。**服务端的 `auth` 模块是唯一调用方**，它把这些宿主本地动作与观察器失效
  同 `agent` 的运行时代次接缝组合起来。

## 必须做对（Get right (shellEnv)）

- 在**启动时、创建任何 `AgentSession` 之前运行一次**。
- **启动键在任何修复之前被快照。** 第一次 `resolveShellEnv()` 调用记录已在 `process.env` 中的名字
  （同一份 `Symbol.for` 状态，因此打包的副本共享它）；`launchEnvKeys()` 返回该集合，若从未运行过修复
  则返回当前名字。已存储的宿主环境（[[submodule-server-environment]]）用它，使显式启动变量胜过存储记录，
  而 login-shell 导入或 locale 修复不会。
- **异步、每进程一次、在首次 spawn 前被 await。** `resolveShellEnv()` 返回缓存在 `Symbol.for` 全局键上的
  单个 promise，因此每个调用方都在等同一次运行，即使来自该模块的第二份打包副本（桌面 shell 及其 server
  bundle）。GUI 启动要为登录 shell 付出代价（典型约 100 ms，装了 nvm/conda 时数秒），而同步探测会让整个
  启动卡住。异步探测让启动器把它与不派生进程的工作（bundle 导入、日志初始化）重叠，并在首次 spawn 前
  await 它。改为在每个 spawn 点上门禁被否决：这些点很多（git、PTY、编辑器、Central、pi 工具、扩展），
  漏掉一个就会静默拿到被裁剪的 PATH。login-shell 与 `launchctl` 探测并行运行；`launchctl` 的
  `SSH_AUTH_SOCK` 只在 login-env 导入未设置它时生效。
- **输出只计到它的结束标记为止。** 探针在 `env -0` 之后打印 `LOGIN_ENV_END`，没有它的输出即为失败探测，
  因此被截断的读取绝不会应用被截断的 `PATH` 或静默丢弃变量。标记之后的字节被排空并丢弃直到 shell 退出，
  否则在 EXIT trap 或注销文件写入超过管道缓冲区时会阻塞 shell 直到截止时间。
  一旦 shell 已退出且标记已读到，读取即被取消，因此 rc 文件留下的后台任务即使仍持有管道
  也无法把这次运行拖住。退出后 100ms 的上界只覆盖那种 shell 未打印标记就退出、
  而此类任务仍持有管道的情形。
- **截止时间杀掉整个探测。** shell 在自己的进程组里启动，到 5s 时该组收到 SIGKILL：交互式 zsh 与 bash
  会忽略默认的 SIGTERM，而挂住的 rc 命令会活过它的 shell。
- 在 win32 上不做任何事（GUI 进程已继承注册表 PATH）。其他平台上它**总是**探测：派生登录 shell
  `[$SHELL||/bin/zsh, -l, -i, -c, <print LOGIN_ENV_MARKER>; env -0 && <print LOGIN_ENV_END>]`
  （非零退出时去掉 `-i` 重试），并把 PATH 重置为操作系统的基础目录（使 rc 文件计算它们规范的 PATH，
  而不是叠加到启动器传入的任何内容之上），5s 截止，解析标记之间以 `\0` 分隔的条目，然后
  `mergePath(current, login)`——登录 PATH，前面加上它缺少的当前条目（已激活的 venv、`nix develop`、
  direnv），使从终端启动时其显式追加仍排在前面。从不抛错——任何失败都让 PATH 保持不动。
- 没有“PATH 看起来已经完整”的短路。曾经有一个，以用户目录标记为键，而 `/usr/local/bin` 就在其中；
  它位于 macOS `/etc/paths` 之首、也是每个 Linux 默认 PATH 的成员，于是 Dock 启动的宿主永不运行探测，
  每个会话都失去 Homebrew/bun（`command not found`、git hooks 失败、subagent 卡住）。
  每次应用启动一次的 login-shell 探测比猜测更便宜。

## 必须做对（Get right (jbcentral)）

- **按绝对路径检测并调用 Central（`resolveJbcentralBin`），绝不用裸命令。** 把实时的 `process.env.PATH`
  传给查找，并保留官方的 `~/.local/bin` 回退，因为长跑的宿主必须能看到刚安装好的可执行文件而不必重启。
  **回退名是该 OS 安装器实际写入的文件**——Windows 上是 `central.exe`，其他平台是 `central`
  （`install.ps1` 安装 `$HOME\.local\bin\central.exe`）——而 PATH 查找保持裸名，因为 `Bun.which`
  自己会尝试 Windows 的可执行扩展名，并在那里拒绝无扩展名的文件。Windows 上无扩展名的回退永远匹配不到
  真实安装，而那正是回退存在的场景：Central 从应用内指引安装完成，而宿主已带着过期的 PATH 在运行。
  它的单元测试钉住确切路径，绝不用对一切回答 true 的 `exists` 桩。
- **`~` 在 Windows 上是 `USERPROFILE`，其他平台是 `HOME`**——按平台从实时环境读取，只有 `homedir()`
  作为回退。这与 pi 为 `getAgentDir()` 解析的是*同一个* home，而这正是要点：适配器与 pi 必须指名同一个
  产物。两种被否决的写法都是真实的 bug：`env.HOME ?? homedir()`（最初版本）在 Windows 上静默失效，
  那里 pi 忽略 `HOME`，但 Git Bash 会把 `HOME` 设成一个 MSYS 路径——Central 会被安装并配置好，
  而 ThinkRail 报告 `absent`；而裸 `homedir()` 在 Bun 下无法在进程内被重定向，它**在启动时解析一次**
  而非每次调用（Node 会每次重读），于是那些重定向 home 的测试与 e2e/冒烟宿主会静默读到开发者真实的
  home。平台是可注入依赖（`platform`），因此两个分支都能在任意宿主 OS 上被单元测试钉住。
- **受机密源码启发，但只用公开接口。** 入库产物只包含经评审的 argv、兼容性上下界、不透明路径、类型化
  结果以及独立编写的假实现。绝不复制或改写 Central 的源码/输出，绝不读取、解析、取指纹、快照、复制、
  记录、上传、异地持久化或提供生成的扩展。
- **不依赖独立的 PI。** Central 写全局 PI 配置；ThinkRail 把它的路径载入内嵌 PI 运行时，包括
  `PI_CODING_AGENT_DIR` 指向别处且不存在 `pi` 命令的情形。观察器事件只是提示：每次重建都重新检查
  版本 + 存在性，应用内动作在其后置条件之后显式请求同样的重建，因此丢失/合并的文件系统事件不会丢掉一次
  状态迁移。
- **绝不返回或抛出原始的子进程/加载器数据。** 适配器错误是封闭原因码，不含自由格式的子进程文本。
  配额成功只携带周期性的 `remaining` / `total` 数字；调用方只能报告通用指引。
- **无遗留迁移。** 这是第一个受支持的原生集成；适配器绝不触碰先前的 provider 覆盖配置，
  也不维护备份/回滚机制。

## 必须做对（Get right (freePort)）

- 通过 **TCP 连接探测**而不是捕获 bind 错误来检测占用：`Bun.serve` 并非在每个平台上都为被占用的
  `localhost` 端口报告 `EADDRINUSE`（它可能通过 `SO_REUSEPORT` 共享端口），因此 bind-and-catch 检查
  不可靠。连接被拒即表示空闲。
- `findFreePort` 从 `preferred` 向上扫描（可预测：`24242 → 24243 → …`），若整个扫描区间都被占用，
  则回退到由 OS 分配的临时端口。
