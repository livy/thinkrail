# CI 与发布流水线

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

公开的 PR 门禁、贡献模板，以及可复用的原生构建配方。发布编排、签名/公证凭据、来源授权、标签、
校验和与发布都归属于 `JetBrains/thinkrail-signing`。本模块拥有由该私有流水线消费的公开 action
输入/输出以及产物/版本契约。

## 边界（Boundary）

- **拥有：** 公开的 CI/站点工作流、贡献模板、原生构建配方、版本计算，以及由私有发布控制器消费的
  产物接口。
- **消费：** CLI 与桌面构建命令、[[module-artifact-tests]] 冒烟入口点、shared 版本标记、
  根一致性/单元/浏览器命令、git，以及原生平台工具。
- **禁止：** 产品运行时逻辑；另一个公开发布控制器；公开签名凭据；在必需签名与验证之前发布；
  对签名前字节计算校验和；或绕过正常包命令的仅发布用应用构建路径。

## 持续集成（CI）

工作流的 Bun 安装通过 `bun-version-file` 读取根 `package.json` 的 `packageManager` 锁定版本。
桌面端打包后的运行时仍归 Electrobun 所有（见 [[module-desktop]]）。

PR 与合并队列门禁覆盖依赖/边界/接缝/规格接口一致性、lint/typecheck、单元测试与无 agent 浏览器测试。
Linux 与 Windows 运行原生 CLI 产物冒烟；Linux 还对编译后的二进制运行浏览器测试。桌面覆盖构建 Linux
目标，在 Xvfb 下以仅测试用的软件渲染标志运行原生窗口与共享产物探针，并运行由桌面支撑的浏览器测试。
Windows 还构建桌面应用并运行其原生冒烟，但不运行浏览器测试（见下文）。Linux 二进制与桌面套件、
以及 Windows 桌面冒烟在每个合并组上都运行，但在 PR 上仅在变更可能影响产物时运行（见下文的变更分类器）。
真实 provider 测试保持显式授权且独立。

三个浏览器套件占据了大部分实际耗时，因此每个都作为独立 runner 上的 Playwright 分片矩阵运行
（分片方式归 [[module-browser-e2e]] 所有）。分片数量以 runner 分钟换取实际耗时，并依据观测到的
单 job 时长调优。

**必需门禁。** 规则集只要求一项检查，“CI passed”：这是一个聚合 job，仅当变更分类器、一致性/lint/
typecheck 与单元测试都成功、且其他每个 job 要么成功要么被分类器跳过时才成功。这样 CI 就可以新增、
重命名或跳过 job 而无需修改规则集；重命名“CI passed”本身会阻塞合并队列。每个 job 都带有超时，
约为其健康耗时的数倍，因此挂起的进程会在几分钟内让该 job 以及“CI passed”失败，而不是占住 runner
直到 GitHub 六小时的默认上限。

**变更分类器。** 一个 `changes` job 把 PR 或合并组与其基线做 diff，重命名同时计入旧路径与新路径。
当每个变更路径都位于应用之外——网站及其分析包、网站工作流、README 与 PR 截图、根 README、
CONTRIBUTING、CODE_OF_CONDUCT 与 LICENSE，以及 GitHub 的 issue/PR 模板、CODEOWNERS 与本规格——
分类器就跳过 pi-package、浏览器、二进制、Windows 与桌面 job；任何其他路径，或 diff 失败，
都会全部运行。排除列表是路径白名单，绝不是文件类型规则：Markdown 在这里是产品输入
（skills 随产物发布，规格喂给一致性门禁）。在合并队列中它对照目标分支 diff，而不是事件的 `base_sha`：
后者是前一个条目的组头，而在 HEADGREEN 分组下，一个绿色的头会合并它前面的每个条目，
因此排在应用 PR 之后的站点 PR 仍然必须运行完整集合。一致性、lint/typecheck 与单元测试始终运行；
它们便宜且也覆盖网站。第二个输出 `artifacts` 门控 Linux 二进制与桌面套件、以及 Windows 桌面冒烟；
Linux 套件对构建产物重放无 agent 浏览器套件，耗费超过一半的 CI runner 时间：在 PR 上，只有当某个路径
喂给构建或打包时它们才运行——CLI 与桌面应用、产物测试、shared 的原生图标集及其生成器、
发布身份与 Windows 文件身份、服务端的构建支持与内置运行时注册、产物 Playwright runner、
CI 工作流、action 与脚本、patch，以及根清单与锁文件——而在合并组中它们始终运行，
因此 main 永远不会落地它们未演练过的产物。因此无关变更中的产物专属缺陷会在入队时暴露，
而不是在 PR 上；在拆分前它是 56 次失败运行中的 1 次。这包括产物套件重放的共享无 agent 规格与夹具：
只有产物 runner 与配置是分类器路径，因此仅破坏二进制或桌面重放的 harness 变更也会在入队时被捕获。
Windows 的单元与二进制 job 保留在每个应用 PR 上：它主要是原生单元门禁，而不是产物重放。

浏览器 job 会有界地重试 Playwright 浏览器下载，这类网络故障否则会把条目踢出合并队列；
在 job 内部，Playwright 在 CI 上对失败的测试重试一次（源码与产物配置都一样），在该重试中通过的测试
被报告为 flaky 且不会让 job 失败，并且没有 job 会被自动重跑。它们安装 Chromium 时不用 `--with-deps`：
runner 镜像已带有其共享库，而额外的 apt 包（CJK/泰文/X 字体）不喂给任何断言，
其 Ubuntu 镜像下载却是 job 最慢的一步，常常要几分钟。缺失的库会让 Chromium 启动时大声失败。
浏览器本身不缓存：Playwright CDN 下载只需几秒，不慢于对同样字节的 Actions cache 恢复。

原生产物冒烟只在分片 1 运行：每个分片重复同一探针不增加覆盖，只会成倍增加 runner 环境故障的暴露。

Linux 桌面 job 通过 `.github/actions/desktop-apt` 安装其 GTK/WebKitGTK 运行时，来源是缓存在
`apt-get update` 之后 apt 计划安装的确切包版本哈希下的 `.deb` 文件。镜像更新或不同的 runner 镜像会改变
该计划，从而改变键：旧缓存按前缀恢复，只有变化了的包会被下载，而 `apt-get autoclean` 会丢弃归档不再
提供的版本，因此在镜像发布之间的间隔以及镜像滚动的过程中，缓慢的 Ubuntu 镜像代价很小。
在 PR 或合并组 ref 上保存的缓存对该 ref 私有，因此 `CI cache` 工作流在每次推送到 main 时播种它，
这是每个 PR 与合并组都能恢复的唯一范围。安装步骤有十分钟上限：一滴一滴给字节的镜像永远不会触发 apt
自身的超时，否则会把 job 占住数小时。这些 job 用自己的步骤启动 Xvfb，并且只有在 X 客户端连接到所导出的
`DISPLAY` 后才继续；始终不响应的服务器会有界重启，然后让 job 失败。除了套件里那一次 Playwright 重试，
只有虚拟显示会被重试：一旦它被证明可用，冒烟与浏览器测试就不再获得进一步的环境重试。

一个平台的原生行为只在该平台上被证明。Windows 仍然是 PR 门禁，因为它的可执行文件、环境、路径与
回收站行为与 POSIX 宿主有实质差异。它的子进程单元门禁还演练原生后代控制台行为，这是 Linux 单元运行
无法证明的。它的桌面窗口装饰单元门禁绑定无边框标题栏所依赖的 `user32.dll` 框架符号，让窗口图标通过
真实窗口往返，并把 AppUserModelID 盖到真实快捷方式上（[[module-desktop]]）。两个门禁都不需要
Electrobun devkit。它的二进制冒烟还检查编译后 exe 的图标与文件属性（[[module-artifact-tests]]）。
一个单独的 Windows 桌面 job 构建桌面应用并运行原生桌面冒烟，检查带品牌的 `bun.exe`、`launcher.exe`
与卸载程序。只有 Windows 构建会在 Hutch 的输出上运行桌面的 `postBuild` 品牌化处理
（[[module-desktop]]）。在这个 job 存在之前，品牌化缺陷最早在夜间发布构建中才失败。
它的路径门禁（`bun run test:windows-paths`）在唯一一个 git 打印 `C:/…`、node 打印 `C:\…`、
且 `\` 不是文件名字符的宿主上运行那些钉住 architecture 决策 22 的套件：已存储目录、
Default 与外部工作区中的聊天生命周期、POSIX 传输线路径、git 对象名、选择器根，以及 `install.ps1`
的 PATH 检查。套件只有在 Windows 上通过后才加入该门禁。前提是 NTFS 无法承载的测试，例如 ref 名中的
`:` 或无特权符号链接，在那里被跳过，并继续由 Linux 覆盖。该门禁以每测试 30 秒的预算运行，
而不是 Bun 默认的 5 秒：那里每次 spawn git 约需 40 ms（工作区套件约比 Linux 慢 14 倍，有一个通过的测试
达到 4.2 s），而 runner 停顿（例如对全新安装的按访问扫描）会把它变成超时。带有显式超时的测试保留它。
runner 的 `TEMP` 是 8.3 短名（`C:\Users\RUNNER~1\…`），而 git 报告长名，因此路径夹具以
`realpathSync.native` 为根，这是 git 与操作系统一致的拼写。安装器文件只贡献其 `install.ps1` 套件，
因为它的 `install.sh` 测试理应拒绝这种前缀携带的 `~`。macOS 与 Linux ARM64 的原生验收属于发布矩阵。
Windows job 安装 Bun 与它们的依赖时不使用 Actions cache：恢复它比全新安装更慢，而且 main 上没有 Windows
job 时每个 PR 与合并组都保留自己的副本，约占仓库缓存配额的一半。

## 原生构建契约（Native build contract）

私有原生矩阵检出一个显式的公开源码提交并调用 `.github/actions/build-binary`。该 action 接受 `version`、
`channel` 以及与宿主匹配的 `target`，外加可选的 `posthog-cli-api-key` / `posthog-cli-project-id`
用于 source map 上传（见下文）。它稳定的公开输出是：

- `artifact-name` / `artifact-path`——原生 CLI；
- `desktop-artifact-name` / `desktop-artifact-path`——首次安装桌面产物；
- `desktop-app-archive-path`——Electrobun 展开后的 macOS 应用归档，在非 macOS 目标上为空；
- `desktop-update-manifest-path` / `desktop-update-archive-path`——Electrobun 的更新器元数据与
  完整应用归档，在每个桌面目标上都非空。

该配方标记共享版本，构建并冒烟 CLI，调用正常的 Electrobun dev 与 channel 构建，运行展开应用与首次安装
冒烟，然后收集确切的目标/channel 输出。更新输出保留 Electrobun 生成的基名与压缩字节。清单是确切的
`<stable|canary>-<macos|win|linux>-<arm64|x64>-update.json`。其 `artifact.file` 是归档基名：
stable 为 `stable-<platform>-ThinkRail[.app].tar.zst`，nightly 为
`canary-<platform>-ThinkRail-canary[.app].tar.zst`，其中 `<platform>` 是清单中的
操作系统-架构对，`.app` 只出现在 macOS 上。收集过程会为所请求的 channel 与目标解析两个名字，
而不是接受第一个 glob 匹配。这些路径直接指向 Electrobun 的输出树；更新器载荷不会改变仅安装器使用的
dist 别名。在 macOS 上，更新归档可以与通过 `desktop-app-archive-path` 暴露的展开应用归档相同；
该配方不会复制或重新打包它。[[module-artifact-tests]] 拥有隔离的打包调用与收集器契约测试；
本模块拥有该 action 与交付契约。

支持的桌面资产：

| 目标 | 原生 runner | 发布的资产 |
| --- | --- | --- |
| `bun-darwin-arm64` | `macos-15` | `thinkrail-desktop-darwin-arm64.dmg` |
| `bun-darwin-x64` | `macos-15-intel` | `thinkrail-desktop-darwin-x64.dmg` |
| `bun-windows-x64` | `windows-2022` | `thinkrail-desktop-windows-x64.zip` |
| `bun-linux-x64` | `ubuntu-24.04` | `thinkrail-desktop-linux-x64.tar.gz` |
| `bun-linux-arm64` | `ubuntu-24.04-arm` | `thinkrail-desktop-linux-arm64.tar.gz` |

私有流水线拥有这些 runner 基线。**Intel runner 关注项：** `macos-15-intel` 只运行 x64 构建与
最终 DMG 的安装冒烟。两个架构的 stapling、DMG 创建与签名检查都留在 arm64 的 `macos-15` 上，
因为 `xcrun stapler` 在 Intel 镜像上失败
（[runner-images#14924](https://github.com/actions/runner-images/issues/14924)）。`macos-26-intel`
被避开，因为它的 `iconservicesagent` 崩溃循环会挂住应用启动测试
（[#14751](https://github.com/actions/runner-images/issues/14751)）；出于同样原因 Electrobun 把它自己的
Intel 桌面验证保持在 `macos-15-intel` 上。还要关注 Intel GPU 挂起
（[#14836](https://github.com/actions/runner-images/issues/14836)）与缓慢的 Intel runner
（[#13358](https://github.com/actions/runner-images/issues/13358)）。GitHub 称 `macos-15-intel` 是其最后一个
托管 x86_64 镜像，可使用到约 2027 年 8 月；在那之前，需要在自托管 Intel runner 与放弃 Intel 构建之间
做选择。每当 macOS runner 镜像变化时都要重新核查这些问题。

Linux 别名指向未改动的框架安装器。Windows ZIP 包含其安装程序可执行文件与相邻载荷；私有流水线对两者
签名并重新打包载荷（见签名）。Linux 的 tarball 包含安装器与 README。对 macOS，私有 SRE 流程用包含
已签名展开应用的常规 DMG 替换未签名的框架 DMG，并使用同一发布别名。应用归档是私有的签名中间产物，
绝不是公开的发布资产。

在 Windows 上，打包调用把 System32 放在最前，使 Hutch 的裸 `tar` 解析到 Windows bsdtar；
Git 的 GNU tar 会把盘符路径解释为远程 `host:path` 操作数。收集器解析确切的框架文件名。
Stable 安装器省略 `stable-` 前缀；nightly 使用 Electrobun 的 `canary` 前缀/后缀。完整更新器归档与元数据
以其原始框架名交给私有流水线；patch 生成保持禁用。Linux 要求 Ubuntu 24.04+/glibc 2.38 以及所声明的
GTK、WebKitGTK、AppIndicator 与 librsvg 依赖。CEF 与其他安装器格式不在本契约之内。

## 发布身份与信任（Release identity and trust）

发布控制器提供一个显式的公开 `source_sha`。公开配方只把 `{ version, channel, commit, posthogProjectKey }`
标记进 `packages/shared/src/version.ts`；CLI、桌面配置、分析以及 `server.welcome.appVersion` 消费同一个
身份。该 action 必需的 `posthog-project-key` 输入是应用分析键的唯一来源——仓库不携带任何分析键——
当它为空或 channel 不是 `nightly|stable` 时标记会失败关闭（fail closed）。它是公开的摄取键，
不是服务凭据，因此托管原生 job 可以接收它；私有工作流从它的密钥存储传入，标记步骤从不回显它。
私有工作流自身的提交绝不是产品身份。

版本本身来自 `.github/scripts/next-version.sh`，这是唯一的公开版本策略。它读取 `git tag -l`，
除非 `--tags` 提供了列表。发布控制器传入 git 标签加上其更新馈送仍在公布的版本，因此删除一个发布及其
标签绝不会释放那个编号。所以 `--tags` 是被消费的接口，而不是仅测试用的钩子。

签名凭据与对 `codesign.labs.jb.gg` 的访问留在受保护的私有流水线中。托管原生 job 不接收任何服务凭据，
只有一个所有者批准的例外：一个 PostHog 个人 API 键（错误追踪写、组织读）用于 source map 上传。
PostHog 的 `posthog-cli` 必须在打包前把 chunk id 标记进 JS 并为此需要该键，而 map 绝不能发布，
因此上传无法移到更晚的受保护 job。该键只抵达 web 构建、`desktop:build` 与桌面打包步骤
（绝不抵达冒烟；身份标记步骤只看到它是否已设置），并且不出现在 PR CI 中，后者构建保持不变。
受保护的内部 job 除此之外只发起服务调用，不构建产品源码也不组装归档。

**Source map。** 当设置了 `posthog-cli-api-key` 时，这些步骤会收到
`THINKRAIL_SOURCEMAPS_RELEASE=<version>` 与 CLI 凭据（EU 宿主）。web 构建产出隐藏的 map 并运行
`posthog-cli sourcemap process --release-mode symbol-set --delete-after`
（release 为 `thinkrail@<version>`）；桌面端的 `preBuild` 自己把 chunk id 注入运行时 bundle 并上传它
（[[module-desktop]]）。`@posthog/cli` 在根目录中锁定一次，而它的 npm wrapper 在首次使用时下载原生
二进制且没有任何完整性校验，因此每次调用都经过根 `posthog-cli` 脚本（`apps/web/posthogCli.ts`）：
该脚本拒绝除锁定版本之外的 wrapper 版本，以及 SHA-256 不在七个按平台锁定的值之中的二进制
（这些值取自摘要与 PostHog 的 GitHub 发布匹配的归档）——都在二进制看到键或改写已发布 JS 之前完成。
升级 `@posthog/cli` 意味着重新锁定。构建会失败，而不是打包任何 web `.map`，并且 turbo 会把发布变量纳入
哈希，因此缓存命中会恢复注入后的无 map 输出。CLI 二进制则内嵌自己的 map（[[module-cli]]）。

CodeSign 与校验和 action 是私有的，并与发布工作流一起版本化。

流水线失败关闭：它构建每个目标，签名并验证所需产物，对最终字节计算校验和，将源码提交对公开 main 授权，
创建幂等标签，然后才发布。中间产物绝不进入公开发布。一个显式的未签名允许列表只接纳 Linux 产物；
任何新平台或意外文件都会阻塞发布，直到其签名策略被确定。

## 签名与公证（Signing and notarization）

Windows 签名覆盖 CLI、安装 stub，以及 stub 相邻载荷中的每个 PE 文件：启动器、`bun.exe`、
框架 DLL 与工具、用于安装并应用更新的内置 `uninstall` 辅助程序，以及运行时库。未签名的框架二进制在
安装与更新期间招致 Defender ML 隔离。私有流水线提取该载荷，对其 PE 文件签名，在不触碰其他成员的情况下
重新打包，并把同样的这些字节作为 Windows 更新归档发布。载荷中框架的 `hash` 是身份元数据，
不是内容摘要，因此重新打包使其保持有效。发布要求最终 ZIP 在托管 Windows runner 上通过首次安装冒烟。
macOS CLI 对 `thinkrail-darwin-arm64` 与 `thinkrail-darwin-x64` 都保留其独立的 Developer ID 签名路径。

桌面 macOS 签名消费 Electrobun 展开的 `.app.tar.zst`，因为 Hutch 在 `postBuild` 之后写入最终版本元数据，
并在 `postWrap` 之前压缩内层应用。托管 macOS 准备原生传输物；受保护的内部 job 对每个 Mach-O 文件签名，
封装完成的应用 ZIP，并在一个独立操作中对其公证。托管 macOS 钉合应用票据并创建 DMG。
受保护的 job 另行为该 DMG 签名、公证并钉合。

发布要求对应用与容器做原生 macOS 验证：JetBrains 签名身份、强化运行时（hardened runtime）、
Bun 权限声明（entitlements）、严格签名检查、Gatekeeper 评估、公证票据，以及首次安装冒烟。
签名与公证是不同的服务操作，并保留显式的内容类型。权限声明提取在 plist 解析之前请求 XML。

## 桌面更新器资格验证与发布（Desktop updater qualification and publication）

自动更新范围包含上表中每个当前原生桌面目标。Canary 资格验证先于 stable；每个目标在启用前都需要一次
观测到的、从已安装版本 A 升级到版本 B 的过程。公开构建交接现在包含原始 Electrobun 清单与完整归档；
私有运行时馈送与发布自动化尚未上线。运行时策略属于 [[module-desktop]]。

私有流水线消费这些原始名字，通过既有签名流程最终化 macOS 应用与 Windows 载荷，并用这些最终字节创建它们
的更新归档。它为每个最终化归档分配一个不可变的、带版本限定的、协议兼容的基名，并在发布前只修改对应框架
清单中的 `artifact.file`。其他清单身份字段保持框架生成。载荷在上传后、其 channel 清单之前公开可取。
这个更新器交接不新增任何签名、公证、验证或发布门禁；既有的平台要求继续有效。

原生更新验收是**手工优先**的。它使用最终发布的字节在一次性安装中运行，记录两个版本以及每个目标的结果，
覆盖以下情形：检查停在“可用”、显式下载、传输/准备彼此不同、通过关闭设置延后、**安装并重启**、
channel 隔离、传输被中断/重试、更新后版本/宿主健康，以及本地状态的恢复。运行中的 agent 与终端命令
不预期在宿主重启后存活。Windows 资格验证使用一次性 VM/runner，因为安装会集成到 HOME 之外。
当 Electrobun、打包、签名或重启处理变化时，相关检查会重复。这限制了对一个不常变化的集成在宽容的新更新器
套件上的投入；它不取消既有的签名、产物冒烟、快速或完整的无 agent 浏览器门禁。新的自动化覆盖保持在应用
自有生命周期回归与既有公开产物输出契约测试附近；不需要专门的更新器 CI 矩阵。

## CLI 安装与其他自动化（CLI installation and other automation）

根 `install.sh` 与 `install.ps1` 保持仅 CLI 消费方：校验 channel/版本身份、解析所请求的发布、
在构造任何产物 URL 之前按所选 channel 的确切语法校验其标签、下载原生 CLI 加 `SHA256SUMS`、验证它，
并在目标目录中暂存替换物，然后做原子重命名。前缀检查基于表示而不是仅 ASCII：
Unicode 与良性的 `@`/`+` 组件保持有效，而每个平台都会拒绝会破坏元数据或启动 shell 语法的字符。
Git Bash 写入原生可读的 Windows 元数据；格式错误的 shell PATH 块被保留而不是重写。在 macOS 上，
当 shell 在 Rosetta 下运行（`sysctl.proc_translated` 为 1）时，`install.sh` 选择 arm64，
即使 `uname -m` 报告 `x86_64`，因此 Apple Silicon 总能得到原生 CLI，而经过翻译的 x64 安装会在下一次
自更新时迁移到 arm64。CLI 自更新把元数据绑定到正在运行的可执行文件，并调用这些安装器，
而不是复制下载/校验和逻辑。受控脚本测试拥有这些回归；最终化的 Windows PowerShell/Git-Bash 与自定义卷
行为仍属于发布资格验证。`install.ps1` 套件在计时测试之前，用预热钩子以独立预算启动一次 PowerShell：
在全新的托管 runner 上首次启动 `powershell.exe` 曾耗时长达 31 s（通过的运行中为 2–14 s），
这是一次性环境成本，不应由任何单个测试的预算吸收。

网站部署不含任何签名凭据。CODEOWNERS 与 main 分支规则保护公开配方；本地构建与公开 PR job 绝不发布
面向用户的发布产物。
