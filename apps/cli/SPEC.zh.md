# CLI 宿主启动器

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

保留的浏览器启动器与独立的回滚产物：`thinkrail` bin。它在进程内启动引擎宿主，并在其 URL
上打开浏览器 UI。它是 `apps/desktop` 的轻量同级；所有引擎与进程启动逻辑都位于
`packages/server`。

## 流程（Flow）

1. 把 argv + env 解析为选项（`src/args.ts`，一个纯函数）；`--help` 打印用法并退出。
2. 解析静态目录（`THINKRAIL_STATIC_DIR`，否则使用随 bin 一起发布的已构建 web 应用），
   缺失时给出警告。
3. await `bootHost({ portMode: "free", resolveShellEnv: true, … })`。共享启动路径安装崩溃
   日志、修复登录 shell 环境、初始化当前的 PI runtime/Central watcher、在请求端口或之上
   解析出一个空闲服务端口，并把宿主嵌入本 Bun 进程。使用同一数据目录的另一个 CLI 或
   桌面宿主不会阻塞启动；每个进程各自服务自己的端点。
4. 在交互式 stdout 上渲染共享的递归 ThinkRail 启动标记，带有诚实的 `host ready` 状态
   + 已解析的端点，保留可稳定解析的 `thinkrail → <url>` 行，并在该地址打开浏览器
   （跨平台：`open` / `start` / `xdg-open`，尽力而为），除非指定 `--no-open`。仅退出的
   命令与重定向输出会省略该标记。
5. SIGINT / SIGTERM 在退出前 await 共享的幂等 `server.shutdown()`；它们不会重复执行
   agent、analytics 或资源清理。

## 接口（Interface）

`bin` = `./src/index.ts`（bun 直接运行 TS 源码）。开头的 `update` 或 `uninstall` 位置参数
是一个**子命令**（
`thinkrail update [--channel stable|nightly] [--version X.Y.Z|X.Y.Z-nightly.N|latest]`、
`thinkrail uninstall [--remove-data|--keep-data] [-y]`），在启动参数之前被拦截 —— 见下文
*自更新* / *卸载*。该集合位于 `args.ts`（`parseSubcommand`）中，因为编译后的入口也需要
它：子命令从不启动宿主，因此不该为暂存资源缓存付出代价（在 `uninstall` 的情况下还必须
不重新创建它）。其余启动参数：
`--port`（稳定默认值 24242，冲突时向上扫描到下一个空闲端口）、`--host`（默认
`localhost`）、`--no-open`、`--no-analytics`（**仅对本次运行抑制附加 analytics** ——
基础事件仍然开启；持久化的附加数据选择在 Settings → Privacy，见
[[submodule-server-analytics]]）、`--verbose`（debug 级日志 —— 以 `verbose: true` 传给
`bootHost`；`<dataDir>/logs` 下的日志文件及其环境开关 `THINKRAIL_LOG_LEVEL` 属于
`submodule-server-log`，该模块是此变量的唯一读取者 —— 与 `THINKRAIL_NO_ANALYTICS`
同一模式，因此 `dev.ts` 也遵守它）、`-v`/`--version`（打印烧录的版本并退出）、
`-h`/`--help`，以及一个位置参数 `project-dir`（启动时作为项目打开的 git 仓库，尽力而为）。
环境默认值：`THINKRAIL_PORT` / `THINKRAIL_HOST` / `THINKRAIL_STATIC_DIR`
（flag > env > default）。`THINKRAIL_NO_ANALYTICS` 在 `--help` 中有文档，但此处刻意
**不**解析 —— 宿主的 analytics 模块是它的唯一读取者（见下文）。

## 更新提示（Update advisory）

位于受支持的 `<prefix>/bin/thinkrail[.exe]` 布局中的已编译 `stable`/`nightly` 二进制，会向
`bootHost` 提供一个可选的周期性更新提示生产者；源码身份与 `dev` 身份，以及手动放置的
二进制，都不暴露该能力。宿主就绪后，它立即执行一次有界查询，之后每六小时在烧录的
channel 上按与安装器相同的 GitHub 规则查询：stable 读取 `releases/latest`，nightly 则从
`releases?per_page=20` 中选取第一个 `vX.Y.Z-nightly.N` tag。只有严格更新的版本才会产生
`{ currentVersion, availableVersion, channel }` 状态；发现失败与无更新都保持静默。另一个
更新的发行版可以替换 available 或 failed 状态，但成功执行会锁定直到宿主重启，因此仍在
运行的旧进程既不能重新发现也不能安装另一个发行版。

共享更新接口为已编译宿主提供 **Run Update**，并为较旧的客户端/宿主或运行失败保留固定的
`thinkrail update` 指引。该动作以异步方式调用同一个已安装二进制的无参 update 子命令；
浏览器无法选择命令、路径、channel、版本或 URL。有界的 running/succeeded/failed 状态
在每个客户端上收敛，成功之后需要用户手动重启这个无监督宿主，而不是中止正在进行的工作。
发现与执行都不出现在 `update`/`uninstall`/help/version 退出路径上，也不出现在所有
源码/dev 启动中。

## 自更新（`thinkrail update`）

`src/update.ts` 移植了旧仓库的 `thinkrail upgrade`（已更名）：它针对二进制所属的 channel
重新调用**已发布的安装器** —— macOS/Linux 上是 `install.sh`，Windows 上是 `install.ps1` ——
从而让安装器继续作为下载 → 校验和 → 替换 → PATH 逻辑的唯一来源。对于已安装的二进制，
正在运行的 `<prefix>/bin` 布局是权威来源；显式的 channel flag 优先，其次是同一安装的
元数据，再次是其烧录的 channel。没有运行中二进制布局的源码调用可以使用
`~/.config/thinkrail/install.json`，并回退到 `stable` / `~/.local`。

- **Unix：** `curl` 取脚本，喂给 `bash -s -- --channel … --prefix … [--version …]`。
- **Windows：** 取 `install.ps1`，写入临时 `.ps1`，然后通过第一个可用的 PowerShell 宿主
  （`powershell.exe`，否则 `pwsh.exe`）以
  `-NoProfile -NonInteractive -ExecutionPolicy Bypass -File <tmp> -Channel … -Version … -Prefix …`
  运行。用 `-File`（而非需要 shell 管道的 `irm | iex`，也非引号处理极易出错的
  `-Command`）把安装器自身的参数经 argv 传入。三个参数**始终**传入，包括
  `-Version latest`：install.ps1 的参数*默认取自 `THINKRAIL_*` 环境变量*，而子进程会继承
  这些变量，所以只有显式传参才能让更新对设置了这些变量的用户是确定性的。替换*正在
  运行*的 exe 之所以可行，是因为 install.ps1 的 `Install-ThinkRailBinary` 会把被占用的
  `thinkrail.exe` 重命名为 `thinkrail.exe.<rand>.old`（Windows 拒绝*删除*正在运行的镜像，
  但允许*重命名*它），再放入新文件；下一次安装会清扫残留。

任何 Windows 失败（抓取、无 PowerShell 宿主、安装器非零返回）都会回退到*打印*按 shell
区分的手动命令（`windowsManualUpdateMessage`），并在其下方给出 releases 页面：cmd 的
`set "X=v" &&` 与 PowerShell 的 `$env:X='v';` 不可互换。更新规划在信任
`<prefix>/bin/thinkrail[.exe]` 的前缀或 channel 之前，先把元数据绑定到正在运行的该文件；
陈旧/无关的元数据无法更新另一份拷贝。缺少元数据时回退到该运行中布局，包括规范化后的
旧式 Git-Bash `/c/...` 与 `/cygdrive/c/...` 前缀。前缀可表示性遵循 [[module-ci-release]]，
而非仅 ASCII 的路径策略：Unicode 与良性标点仍然有效。位于可表示布局之外的手动二进制
—— 包括直接运行的发行版资源如 `thinkrail-linux-x64` —— 会安全失败并给出手动指引，而不是
安装到 `.local` 之下；该预检
同时不授予宿主更新能力。版本/channel 组合在解析之后进行校验。规划函数保持纯函数并有
单元测试；只有安装器抓取与执行接触 IO。`THINKRAIL_INSTALL_SCRIPT_URL` /
`THINKRAIL_INSTALL_PS1_URL` 覆盖安装器 URL（用于测试/分支）。安装器本身见
`module-ci-release`。

## 卸载（`thinkrail uninstall`）

`src/uninstall.ts` 是安装器的反面，且仅是它们的反面：它移除**可执行文件、安装器所做的
PATH 修改、`install.json` 以及二进制的暂存缓存** —— 然后*询问*用户的 app 状态（数据
目录），后者**默认保留**，因为其中存有工作区的 git worktree 及其中任何未提交的工作。
pi 自身的状态（`~/.pi`）永不被触碰：那不属于我们。

- **先规划，再确认，再执行。** `resolveUninstallTargets` 是纯函数（平台 + home + 安装
  元数据 + `process.execPath` → 路径）；一次检查遍历把它收窄到实际存在的内容（以及哪些
  rc 文件确实带有安装器的块），使打印出的计划是*真实*的；然后是提示；然后是删除，
  每项报告为 `removed` / `kept` / `not found` / `failed`（任一 `failed` → 退出码 1）。
- **提示**（`node:readline/promises`）：先是数据目录问题（默认*保留*），然后是最终确认。
  `-y`/`--yes` 跳过两者，`--keep-data`/`--remove-data` 以非交互方式回答第一个问题；没有
  TTY 且没有 `--yes` 时，命令会拒绝而不是猜测。
- **哪些可执行文件：** 来自安装元数据的 `<prefix>/bin/thinkrail[.exe]`（否则用安装器的
  `~/.local` 默认值）**加上 `process.execPath`——当它本身就是 `thinkrail` 二进制时** ——
  这覆盖了 `install.json` 已丢失的自定义前缀。无论元数据怎么说，其他任何东西都不会被
  删除。
- **PATH 修改，按平台接缝：** 在 Unix 上，安装器的块是*自我标识*的
  （`# >>> thinkrail PATH >>>` … `# <<< thinkrail PATH <<<`），因此 `stripRcPathBlock` 会从
  每个带有该块的候选 rc 文件（bash/zsh/`$ZDOTDIR`/`.profile`，而当 fish `conf.d` 文件中
  除了该块别无他物时该文件被删除）中移除它 —— 并且拒绝处理块没有结束标记的文件，而
  不是截断它。在 Windows 上，注册表项是*不透明的*（没有任何标记表明它属于我们），因此
  install.ps1 会**记录它是否添加了该项**（`install.json` 的 `path_entry_added`，在同一
  前缀的重复安装间保持粘性 —— 一次更新之所以看到 `already`，正是因为此前我们的一次
  运行添加了它），卸载器**仅在该 flag 为真、且仅针对记录时所对应的前缀**移除它。被安装
  与添加过该项不是一回事：`-NoModifyPath`、原本已存在的项、失败的注册表写入，以及
  Git-Bash 的 `install.sh` 安装，都记录了一次安装而未触碰 Windows PATH，而旧式元数据早于
  该 flag —— 所有这些都*不属于我们*，因此 PATH 保持原样，并告知用户应检查哪个目录。
  两侧都把项中的 `/` 与 `\` 视为同一分隔符，因此用户自有的 `C:/…/bin` 项算作已存在：
  安装永不追加一个它随后会记录为自己所有的重复项，而卸载器届时会连同用户的项一起移除
  该重复项。修改本身以嵌入的 PowerShell 脚本运行 —— 它是 install.ps1 的
  `Add-ThinkRailToUserPath` 的精确反面（相同的 `HKCU\Environment` 处理，保留值的
  `REG_EXPAND_SZ` 类型，对项做原样*与* `%VAR%` 展开后的比较，然后广播
  `WM_SETTINGCHANGE`）—— 因为让原始 PATH 值经管道往返有在控制台代码页中损坏非 ASCII
  项的风险。
- **删除正在运行的程序：** Unix 可以愉快地 unlink 正在运行的二进制。Windows 不行，因此
  exe 会被重命名为 install.ps1 清理逻辑已识别的同名 `thinkrail.exe.<rand>.old`，并在我们
  退出后由一个分离的 PowerShell 重试删除数秒；报告说明实际发生了哪种情况。bin 目录中
  陈旧的 `.old`/`.new` 残留也会被清扫。
- `src/powershell.ts` 是两条 Windows 路径共用的接缝（找一个宿主、把脚本文本送进去运行、
  用 `psQuote` 把值转成单引号字面量）。`src/paths.ts` 拥有*已安装*布局 ——
  `install.json`（由 `update` + `uninstall` 读取）与暂存缓存根（由 `compiled-entry` 写入、
  由 `uninstall` 删除）—— 因此这三者从构造上就一致。

## 版本标记（发布接缝）（Version stamping (release seam)）

`@thinkrail/shared/version` 导出 `{ version, channel, commit, posthogProjectKey }`，带一个
永久的源码默认值（`0.0.0-dev`，空 key）。发布流水线在构建 CLI 与桌面之前，在一次性 CI
检出中覆盖这一个模块，因此两者报告相同身份，且只有发布构建携带 analytics key。
`bootstrap.ts` 为 `--version` 打印共享版本，把它作为 `server.welcome.appVersion` 传入
`bootHost`，并把该 key、channel、`build: "binary" | "source"` 以及按次运行的附加数据抑制
一并穿入 analytics。当正常打开被启用时，它还为打包二进制的一次性归因声明提供现有的
`openBrowser` 启动器回调，先打开本地 UI，然后显式发出 `server.startAttributionClaim()`
信号。`--no-open` 省略该能力且从不发出就绪信号，因此它不消耗一次尝试。源码运行仍然
不具备资格。

## 启动入口与构建来源（Launch entries + build provenance）

`src/bootstrap.ts` 拥有启动序列（argv → 子命令或宿主启动 → 打开浏览器），并导出
`launch(build: BuildKind)`，其中承载唯一的错误退出路径。两个入口*仅*在它们声明的来源上
不同，且各自从构造上便知道自己的来源，而不是靠检查运行时：

- `src/index.ts` —— `bin`，即**从源码**运行：`launch("source")`。
- `src/compiled-entry.ts` —— **已编译二进制**的入口（先做暂存 + pi 注册）：
  `launch("binary")`。

`build` 作为普通属性随 analytics 传递，因此 `channel = dev` 的运行仍能区分本地编译的二进制
与源码运行（见 `submodule-server-analytics`）。刻意*不*从 Bun 的 `/$bunfs/` 模块路径嗅探：
那是 Bun 升级可能改变的实现细节，且会静默地错标。`src/args.ts` 把 `--no-analytics` 解析为
`CliOptions.noAnalytics`，但**不**读取 `THINKRAIL_NO_ANALYTICS` —— 宿主的 analytics 模块是
该变量的唯一读取者，因此每个入口点对附加数据都遵守它（包括不解析任何 argv 的
`packages/server/src/dev.ts`）。两种写法都不抑制基础事件；两个帮助条目都披露该范围，而
不是承诺零 analytics 流量。CI/测试抑制仍归宿主所有，并覆盖两个层级。

## 单文件二进制（`build:binary`）

二进制内嵌其 source map（`--sourcemap`，在 bytecode 构建之上约 +17 MB；Bun 的 `--compile`
只要生成 map 就会内嵌它，因此没有任何上传路径能避开这笔体积），以便 Bun 在设备端解析
错误堆栈（含 bytecode）；构建会删除它同时写出的外部 `.map`，拒绝内嵌 web 的 `.map`
文件，并从 `BINARY_SOURCE_MAP_BASE`（`apps/cli`）编译，启动器在二进制构建中把它作为
analytics 的 `remapBase` 传入，因为重映射后的帧相对于该目录。

`bun run build:binary` 通过 `bun build --compile` 产出**独立的 `thinkrail` 可执行文件** ——
每个平台一个自包含文件。Bun 打包宿主，*并*透明地内嵌 `bun-pty` 原生库；额外的步骤是
**web UI**（宿主通常服务的目录）、**内置 pi 扩展**（开发模式下服务端从 `node_modules`
按路径加载它们 —— 在二进制内部不可能），以及 `trash` 的**原生辅助 sidecar**（macOS/Windows
必须从真实文件系统路径执行它们）：

- `scripts/build-binary.ts` 消费 `@thinkrail/server/build-support`，写出三个**暂态**生成
  模块，对 `src/compiled-entry.ts` 运行
  `bun build --compile --no-compile-autoload-bunfig --bytecode --format=esm --sourcemap --target=<host|--target>`，
  随后删除它们（使产物无法在 ThinkRail 启动前执行项目本地的 `bunfig.toml` preload，并让
  工作树与 `tsc` 保持干净）；每个生成模块都有一个已提交的 `.d.ts` 类型契约，供 `.ts`
  缺席时 `tsc` 解析：
  - `src/web-assets.generated.ts` —— 枚举 `apps/web/dist`：每个资源一个 Bun 文件属性
    导入 + 一个 `{ route, data }[]` 清单 + 一个内容哈希版本。
  - `src/bundled-extensions.generated.ts` —— **值导入三个内置 pi-package 入口**
    （`pi-spec-graph`、`pi-thinkrail-workflow`、`pi-todos`），从*服务端包*的模块上下文
    解析（绝对路径 —— 它们不是 `cli` 的依赖），因此 Bun 把原始 `.ts` 及其真实依赖
    （`yaml` 等）编译进二进制。服务端注册表中列出的 ThinkRail 扩展
    （[[submodule-server-extensions]]）不生成入口 —— `@thinkrail/server` 以静态导入
    引入它们，因此 `bun build --compile` 通过常规服务端图携带它们（及其依赖，如
    `linkedom`，或带 `unpdf` 的 `pi-web-access`）。外加每个 `BuildRuntimeSources.skillRoots`
    条目（`pi-spec-graph`/`pi-thinkrail-workflow`/`pi-todos` 的 `skills/`，以及
    `packages/server/skills` 下服务端自有的宿主 skill，位于 `host/` 路由下；每个注册表扩展的
    `skillPackages` 根位于 `extensions/<name>/` 下，因此被关闭的扩展目录可以不出现在 pi 的
    skill 输入中）像 web 资源一样内嵌（与 dev 通过 `additionalSkillPaths` 装配的内容一致 ——
    是等价，而非超集）。其 `.d.ts` 通过服务端导出的 `BundledExtensionFactory` 为这些工厂
    定型，因此 `cli` 仍然从不导入 `@earendil-works/pi-coding-agent`。
  - `src/runtime-assets.generated.ts` —— 把 `trash` 的 `macos-trash` 与 `windows-trash.exe`
    辅助二进制（从服务端包的依赖上下文解析）作为内容哈希清单内嵌。
- `src/compiled-entry.ts` 是二进制的入口：启动时它把内嵌的 web + skills + runtime 辅助文件
  暂存到按构建划分的缓存目录（`$XDG_CACHE_HOME`/`~/.cache`/temp；文件直接写入带版本的
  目录，然后**最后**写一个同级的 `<dir>.complete` 标记 —— 就绪以该标记为门禁，因此首次
  运行被杀会留下不完整缓存，下次启动时重新解包。**没有先暂存再重命名**：Bun 对全新的
  非空目录 `renameSync` 在 Windows 上会 `EPERM`，所以用标记取代目录重命名的发布），让
  macOS 辅助文件可执行，设置 `THINKRAIL_STATIC_DIR`，然后 **await** 服务端的
  **`registerBundledRuntime`** 接缝 —— 它注入工厂 + 暂存 skills 目录 + 真实 trash 辅助路径，
  **并**执行 pi 的仅二进制注册（静态打包的 OAuth 流程 + Bedrock provider 模块，取代 pi
  对二进制不友好的动态导入 —— 见服务端 agent 的 SPEC）—— 随后交给 `index.ts`。
  （`bun-pty` 自动自解包；**没有 photon wasm** —— agent 的 read 工具被设置为在服务端发送
  原始图片。skills 必须暂存到*真实*文件系统：pi 通过普通 fs 读取 `SKILL.md` 并把路径
  嵌入系统提示。）
- **Bytecode。** `--bytecode` 把 JSC bytecode 内嵌到 bundle 旁，因此启动时跳过对约 25MB
  宿主图的解析：在 darwin-arm64 上 `/health` 约快 0.3 秒就绪，在 linux-x64 上约快 0.7 秒。
  `--format=esm` 是必需的，因为 `compiled-entry.ts` 使用了顶层 `await`；缺了它
  `--bytecode` 会产出 CJS，而 CJS 会拒绝顶层 await。代价是体积：linux-x64 上约 +41MB，
  darwin-arm64 上约 +42MB。这依赖于根 Bun pin 为 1.4.1+：Bun 1.4.0 在 linux-x64 上内嵌约
  +109MB，且跨目标 bytecode 构建在启动时崩溃（oven-sh/bun#18416）。读取 bytecode 的运行时
  就是写入它的 Bun，因此版本不匹配无法静默禁用它。Bun 1.4.3 还会在构建时优化 bytecode；
  若构建在此处中止（oven-sh/bun#44926），加上 `--no-optimize-bytecode`。不应用压缩：
  它会重命名被堆栈帧与 `constructor.name` 检查读取的标识符。
- 用 `--target=bun-darwin-arm64|bun-linux-x64|bun-windows-x64|…` 交叉编译；每个都会打包该
  平台匹配的 `bun-pty` 库。二进制是平台特定的，首次运行会自解包数 MB。
- **Windows 身份。** 编译后的 exe 否则会从 `bun.exe` 继承一切：Bun 的图标及其 VERSIONINFO
  （"Bun"/"Oven"，Bun 的版本）。于是资源管理器、任务管理器、属性页以及防火墙提示都会
  显示为 Bun。因此针对 Windows 目标，构建会传入 Bun 的 `--windows-icon`、`-title`、
  `-description`、`-publisher`、`-copyright` 与 `-version` 参数。
  - 字符串来自 [[module-shared]] 的 `windowsFileIdentity`。任务管理器显示的是 description，
    因此它等于产品名。
  - 图标是 [[module-shared]] 的 `windowsIconPath`，来自两个启动器共同发布的原生图标集。
  - Bun 只在 Windows 宿主上应用这些参数。发布与 CI 的 Windows 构建在 Windows 上运行；
    从 macOS/Linux 交叉编译会带警告地跳过它们，并保留 Bun 的身份。
  - Bun 保留自己命名的 `IDI_MYICON` 组，shell 会在编号组之前选取它，而它只映射到图标 #1
    （oven-sh/bun#32428）。因此 ICO 必须把 256 帧列在最前；shell 会为更小尺寸降采样它。
  - 编译前会删除输出文件，因为向已有 exe 写入元数据可能失败（oven-sh/bun#24079）。
  - 事后编辑 exe 不是选项：`pe-library`/`resedit` 拒绝 Bun 放在 `.rsrc` 之后的 `.bun` 节。
- **通过启动产物来验证**（而不只是构建它）：扩展接线的回归只在运行时暴露 —— 例如在二进制
  构建首次落地之后新增的每个扩展，其路径加载都静默失效了。[[module-artifact-tests]] 的
  CLI smoke（根：`bun run smoke:binary`，在 `build:binary` 之后）针对一次性 data/agent/cache
  目录启动已构建的二进制。其 CLI 适配器运行共享测试工作区中桌面也在用的共享宿主断言；
  仅 CLI 的断言额外证明暂存缓存与命令行形态。它们共同断言：项目本地的 `bunfig.toml`
  preload **不会**执行、`/health` 有应答、`/` 服务已暂存的 UI、内置 skills 被暂存到缓存
  目录、**一个外部合成的 PI 扩展通过绝对路径加载**（经由编译产物的公开 PI 加载器，且
  `PATH` 上没有 `pi` 可执行文件，在默认与自定义 `PI_CODING_AGENT_DIR` 两种情况下都如此；
  仅使用测试自有的源，不提交、复制、读取或快照任何 Central 生成的产物）、**一次 OAuth
  登录能抵达其 auth URL**（针对 Codex provider 的一个 WS `provider.loginStart` 必须应答
  method 选择并推送一帧 `authUrl` —— 离线且无需凭据，因为 pi 的流程在通知 URL 之前只做
  PKCE + 本地回调服务器；这钉住了静态注册的 OAuth 流程，它们只可能在产物内部出问题 ——
  这些帧是手写的 JSON，从而把 `contracts` 挡在 cli 之外）、**把一个预置的 pi 会话记录
  通过 `session.delete` 移入操作系统回收站**（钉住静态 `processMountinfo` 解析器的包含，
  否则 `trash` 的 Linux 实现会通过一个对二进制不透明的 CommonJS require 触达它；夹具被
  预置在**宿主报告的 `worktreePath`** 处，而绝不是 smoke 自己的临时路径，因为宿主存储的是
  git 符号链接解析后的根 —— macOS 的 `/var` → `/private/var`，Windows 的 8.3 形式 `TEMP` ——
  所以写在未解析路径处的夹具会落到宿主从不扫描的编码会话目录里，删除随后会如实地空操作
  而文件原地不动；其 header cwd 是 `resolve(worktreePath)`，正如 pi 所写，因此若存储的
  目录拼写与 pi 的 header 不同（#766），探针会失败而不是匹配到自己）、创建一个离线会话
  以使每个内置工厂都被求值、验证 macOS/Windows 两个辅助文件都从产物暂存出来，并验证
  SIGTERM 以 0 退出。CI 在 **ubuntu 与 windows** 上构建 + smoke 该二进制（各自用其宿主
  目标）：windows 在每个触及该应用的 PR 上运行，ubuntu 在每个 merge group 以及变更可能
  影响产物的 PR 上运行 `e2e:binary`（见 `module-ci-release`）；macOS 二进制覆盖仍只在
  发布矩阵中。Windows 这条腿不是可选的打磨：宿主只有在其 Central 检查说*已安装且受支持*
  时才会触达该扩展，所以整个断言都是 Windows 可执行形态的，而仅 ubuntu 的 smoke 让 #255
  发布了一个连续两天失败却什么都没发布的发布矩阵（见 `module-ci-release`）。它不向
  provider 发起提示；真实轮次行为仍留在 `e2e:agent`。该 smoke 的**广网兄弟**是
  `bun run e2e:binary`（根 `playwright.binary.config.ts`）：整套无 agent 的 e2e 测试针对
  这个二进制执行 —— 在 CI 中同样在那些运行里。而 `bun run check:seams`
  （根 `scripts/check-binary-seams.ts`）是该接缝类别的构建期金丝雀：当 pi 升级引入一个
  服务端 `registerBundledRuntime` 未静态注册的、新的打包器不透明动态导入时，它会失败。
  扫描**排除 `pi-coding-agent/dist/bundle/`**（在脚本中记录为跳过的目录，像 allowlist 一样
  做陈旧检查）：pi ≥ 0.84.3 在那里为其自身 CLI 的 `bin` 与 `./rpc-entry` 导出提供一份打包
  运行时 —— 这是触达它的仅有的两种方式 —— 而进程内库导入（`.` → `dist/index.js`）仍走
  模块化运行时，因此该 bundle 的不透明导入（allowlist 中模块化接缝的内容哈希分块副本）
  在 ThinkRail 中是死代码，而按分块名把它们加入 allowlist 会在每次 pi 发版时抖动。
- **该 smoke 的夹具是宿主操作系统形态的，不是 POSIX 形态的。** 每一个都曾是 Linux 上全绿
  套件中的 Windows 失败：
  - **假 Central CLI 是被编译的**（`bun build --compile` 成 `central`/`central.exe`），而不是
    写成 `#!/bin/sh` 脚本：只有当 `inspectJbcentral` 解析*并 spawn* 一个报告受支持版本的
    `central` 时，宿主才会把合成扩展喂给 PI，而 Windows 既不能把无扩展名文件解析为可执行
    文件，也不能对 shell 脚本 `CreateProcess`。`.cmd` 垫片被否决 —— `Bun.spawn` 无法在没有
    `cmd.exe` 包装器的情况下启动批处理文件，而且它会按 shell 方言分裂夹具。其 argv 接口
    保持为已评审的那套：`--version` 打印合成的 `central <semver>`，其余一切以非零退出
    （这样后台的 `status` 探针保持为 `unknown` 观察）。
  - **无 pi 的 `PATH` 由实时 `PATH` 推导而来**：去掉含有 `pi` 可执行文件的条目，然后在
    前面加上假 bin 目录 —— 绝不是硬编码的 `/usr/bin:/bin` 骨架，那会在 Windows 上让宿主
    失去 `git.exe` 或 System32（`project.open` 会 shell out 到裸 `git`）。smoke 仍断言没有
    `pi` 可达，并额外断言 `git` 在过滤后存活。
  - **`HOME` *与* `USERPROFILE` 在每个被 spawn 的宿主中都指向 smoke 的临时 home**，因为
    `homedir()` —— pi 的 `getAgentDir()` 使用它 —— 在 Windows 上读取 `USERPROFILE` 并忽略
    `HOME`。否则默认 agent 目录探针会写入 runner 的（或 Windows 开发者的）真实
    `%USERPROFILE%\.pi\agent` 而非沙箱。
  - **每个被 spawn 的宿主的 env 都由 `hostEnv` 构建，它会丢弃它所覆盖 key 的继承大小写变体。**
    Windows 环境变量名不区分大小写，而 runner 中拼作 `Path`，所以熟悉的
    `{...process.env, PATH: x}` 会同时送出*两个* key，子进程读到的却是继承的那个 —— 宿主
    于是带着机器真实 PATH 运行，看不到假 `central`，报告 `absent`，从不加载该扩展，而
    夹具本身却明明白白是好的。
  - **两条腿都要求 `provider.status` 的 `jbcentral` 为 `configured`**，而不仅仅是模型存在：
    产物位于*默认* agent 目录内，因此只检查 `model.list` 的腿原则上可能绕过 pi 自己的
    agent 目录发现，走不到该门禁要钉住的 Central 提供的绝对路径。状态读取会等过一个
    短暂的 `configuring`（一次启动期 watcher 事件）。
  - **两个宿主一次只运行一个**：默认 agent 探针先启动、断言并退出，然后才 spawn 自定义
    agent 宿主。它们加载*同一份*磁盘上的扩展，并发的初次加载会在加载器的转译缓存上竞争
    —— 在 POSIX 上无害，在 Windows 上则是 EPERM 一类失败，输家会静默回退到没有该扩展的
    运行时（`prepareInitialRuntime` 的纯运行时路径）。失败的*重建*会保留已加载的代次，
    因此只有启动加载会以这种方式丢掉 provider —— 这就是为什么断言失败时会报告
    `provider.status` 的 `jbcentral` 状态：`load-failed` 指明该原因，`absent` 指明某个从未
    运行过的夹具。

## 边界（Boundary）

- **拥有：** `src/args.ts`（纯 `parseArgs(argv, env) → CliOptions` + `parseSubcommand` +
  `USAGE`）、`src/index.ts`（从源码运行的 `bootstrap()`：shell env → server → 打开浏览器 →
  信号处理器），以及二进制构建（`scripts/build-binary.ts`、
  `scripts/artifactName.ts` —— 以 `@thinkrail/cli/artifact` 暴露，是产物文件名规则的唯一
  所在，包括 Bun 为 Windows 目标追加的 `.exe`，因此构建的输出路径与 smoke 的默认输入不会
  像它们在 Windows 上那样对不上。它同时持有 Windows 目标规则。发布 action 在 bash 中重新
  推导同一名称，因为它同时是已发布资源的契约，见 `module-ci-release`）、
  `src/compiled-entry.ts`、`src/web-assets.generated.*`、`src/bundled-extensions.generated.*`、
  `src/runtime-assets.generated.*`）、
  `src/update.ts`（`update` 子命令，加上为宿主提示做的同 channel GitHub 发行版发现）、
  `src/uninstall.ts`（`uninstall` 子命令）、`src/paths.ts`（已安装布局：`install.json` +
  暂存缓存根），以及 `src/powershell.ts`（Windows PowerShell 接缝）。Central 集成仍是
  服务端/认证功能；启动器没有 Central 子命令或协议实现。
- **允许的依赖：** `@thinkrail/server`（`bootHost`、`registerBundledRuntime`、build-support、
  `dataDir` —— 卸载器必须命名 app 状态目录，且必须命名与宿主使用的*同一个*）、
  `@thinkrail/shared/startupMark`（共享启动签名渲染器）+
  `@thinkrail/shared/version`（共享发布身份）+ `@thinkrail/shared/windowsIdentity`
  （构建所盖印的 Windows 文件属性与 ICO）、Bun/Node；生成的构建模块可以值导入内置扩展包
  的入口（经服务端包解析 —— 仅构建期，编译后删除）。
- **禁止：** 产品功能/领域逻辑；深入服务端内部（只用其公开接口）；导入 web、desktop 或
  `contracts` UI 层；直接导入 `@earendil-works/pi-coding-agent`；导入产物测试工作区。其
  smoke 适配器与共享探针属于 [[module-artifact-tests]]。普通产品功能不应需要 CLI 实现。

## 必须做对（Get right）

- 对于你会反复运行的 CLI，稳定默认端口比 `port:0` 更友好，但你必须知道解析出的端口才能
  打开 URL —— 因此从请求端口向上扫描到第一个空闲端口，然后打开解析出的 origin。
  （对于被占用的端口，`Bun.serve` 不会给出 `EADDRINUSE`，所以空闲端口靠探测而非捕获绑定
  错误来找到 —— 见 `@thinkrail/shared/freePort`。）
- 浏览器是一等客户端，不是回退方案 —— 同一个 UI 可以指向远程宿主。
- agent 运行在本进程中 —— 一次致命故障会拖垮整个应用（这是已接受的进程内权衡）。
- `resolveShellEnv()` 在任何 `AgentSession` 之前运行一次。
- 启动标记是对已解析启动结果的呈现，绝不是第二个就绪信号：`bootHost` 必须先返回，且其
  下方可稳定解析的 `thinkrail → <url>` 行保持不变。

## 后续（Later）

无头的 `serve` 模式（用于远程/自动化的常驻宿主）。已发布的桌面同级把“打开浏览器”换成了
“打开原生 webview”，复用同一 `bootHost()` 生命周期。
