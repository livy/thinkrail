# ThinkRail

> 中文译文，原文见 [`README.md`](./README.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

[![JetBrains incubator project](https://jb.gg/badges/incubator-plastic.svg)](https://confluence.jetbrains.com/display/ALL/JetBrains+on+GitHub)
[![GitHub stars](https://img.shields.io/github/stars/JetBrains/thinkrail?style=social)](https://star-history.com/#JetBrains/thinkrail&Date)
[![Discord](https://img.shields.io/badge/Discord-Join%20the%20chat-5865F2?logo=discord&logoColor=white)](https://thinkrail.ai/discord)

ThinkRail 是一个 agentic IDE，你用得越多它越好 —— 今天通过活规格了解你的项目，明天构建它自己的
能力。

我们围绕
[`pi`](https://www.npmjs.com/package/@earendil-works/pi-coding-agent) 编程 agent 构建了丰富的
界面，把工作隔离到独立工作区，并通过活规格与可复用的 skill 赋予 agent 持久的项目知识。接下来，
我们正在添加完整的扩展支持，让 ThinkRail 能够根据你实际的工作方式，创建它所需要的工具。

**网站：** [thinkrail.ai](https://thinkrail.ai/) · **博客：** [thinkrail.ai/blog](https://thinkrail.ai/blog/) ·
**Discord：** [thinkrail.ai/discord](https://thinkrail.ai/discord)

⭐ 如果你觉得 ThinkRail 有用，请给它一颗 star —— 这能帮助其他人发现它。

<p align="center">
 <a href="https://www.star-history.com/jetbrains/thinkrail">
  <picture><source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/badge?repo=JetBrains/thinkrail&type=rank&theme=dark" /><source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/badge?repo=JetBrains/thinkrail&type=rank" /><img alt="Star History Rank" src="https://api.star-history.com/badge?repo=JetBrains/thinkrail&type=rank" /></picture> <picture><source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/badge?repo=JetBrains/thinkrail&type=trending&theme=dark" /><source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/badge?repo=JetBrains/thinkrail&type=trending" /><img alt="GitHub Trending Repository of the Day" src="https://api.star-history.com/badge?repo=JetBrains/thinkrail&type=trending" /></picture>
 </a>
</p>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/readme-assets/workbench-dark.png">
  <img src=".github/readme-assets/workbench-light.png" alt="ThinkRail workbench: the Projects pane beside the left tool rail listing worktree workspaces and their Settled shelf, a finished pi conversation with its tool steps in the center, the resulting diff beside it, and the Specs tree above the Changes list on the right with their tools on the edge rail" width="900">
</picture>

*一次 `pi` 会话以及它所做的改动，范围限定在一个隔离的 git-worktree 工作区里。*

## 你能得到什么（What you get）

- **隔离的工作区。** 把任意 git 仓库作为项目打开，并从中切出 `git worktree` 作为工作区，每个都有
  自己的分支与工作目录。agent 可以并行工作而不触碰你的检出目录；合并好的分支，删掉其余的。你也
  可以附加已有 worktree，或明确地在项目文件夹本身中工作。已完成与休眠的工作区会自行结算到按项目
  划分的 shelf 上 —— 当它们的 pull request 合并或关闭时，或经过安静的一周之后 —— 并在你于其中
  工作时立刻回来。顶栏显示活跃分支相对于 `origin` 与其 pull request 的处境，下一步只需一次点击：
  发布、推送，或打开一个由计划、提交、你的 PR 模板或 agent 撰写的草稿所描述的 PR。一旦 GitHub
  PR 打开，它就显示 CI 检查与评审决定，并让你把它标记为就绪、把失败的检查或已分叉的分支交给
  agent、用仓库上次使用的方法合并、重新打开，或删除远程分支。
- **agent 周围的真正 IDE。** 一个可分割的 workbench：Monaco 编辑器标签、带实时 git 变更标记的
  Files 树与 Changed 视图、终端与文档 —— 全部限定在活跃工作区内。在文件和 diff 上留下锚定的评审
  评论并把它们作为结构化上下文发给聊天，或在已安装的 IDE 中打开该 worktree。
- **像引擎思考一样流式输出的聊天。** 每个工作区可有多个并发 `pi` 会话，各自有自己的模型与实时
  token、成本读数。引导正在运行的轮次或排队一个后续，委托给子代理，并推进一个 agent 更新、你来
  编辑的共享计划 —— 每个完成的步骤都可以单独提交与评审。
- **作为事实来源的规格。** 一个由 `SPEC.md` 文件构成、位于代码旁的可连通图谱；agent 通过专用的
  `spec_*` 工具读取、搜索并维护它，因此意图能活过会话。只读的 Specs 工具在 workbench 中把该图谱
  渲染为一棵树。
- **Skill 与工作流。** 你已为其他编程 agent 保存的可移植 Agent Skills 会被就地读取，内置工作流
  引导项目搭建、设计头脑风暴与交付 pull request —— 推送工作区分支，并通过你自己的 `gh` 打开或
  更新其 GitHub PR。
- **shell 之外的 agent 工具。** Web 研究、agent 在聊天中向你提出的结构化多选题（而不是猜测），
  以及直接渲染在聊天中的内联图表、选项对比与文件预览 —— 当 agent 提出一处 UI 改动时，它会展示一个
  前后对比 mockup，你可以像对 diff 一样评论。Settings → Extensions 列出 agent 获得的 ThinkRail
  扩展，并让你逐个关闭。提问与可视化工具也各自作为普通 `pi` 包发布 ——
  `pi install npm:@thinkrail/pi-ask-user-question` 用于提问
  （[`@thinkrail/pi-ask-user-question`](https://www.npmjs.com/package/@thinkrail/pi-ask-user-question)），
  `pi install npm:@thinkrail/pi-visualize` 用于图表与预览
  （[`@thinkrail/pi-visualize`](https://www.npmjs.com/package/@thinkrail/pi-visualize)）—— 因此
  终端中的原生 `pi` 也获得同样的工具。
- **MCP 服务器。** 连接你的 `pi` `mcp.json` 中的 MCP 服务器（或从 Settings 添加它们以及一些预设）：
  对服务器未标记为只读的调用，ThinkRail 会在聊天中先询问（或始终允许你在某个项目中信任的工具），
  仓库定义的服务器在你批准之前保持关闭，你可从 Settings 登录 OAuth 服务器，并显示每个聊天的服务器
  与其后台命令、子代理并列。
- **你的 provider，你的凭据。** 用 [JetBrains AI](https://www.jetbrains.com/ai/) 登录，或通过
  `pi` 自己的认证连接任意 provider。ThinkRail 没有自己的账号。
- **你的 pi 配置。** ThinkRail 读取你已有的 `pi` 配置（`~/.pi/agent`）。要使用另一个 —— 例如个人
  的 `~/.pi/agent-personal` —— 或向 agent、终端与 MCP 服务器传递额外环境变量，就在
  Settings → Environment 中设置；它们在下次启动时生效。启动时设置的变量
  （`PI_CODING_AGENT_DIR=… thinkrail`）仍然优先。
- **桌面应用或浏览器。** 一个原生桌面应用，或在你浏览器中打开同一应用的 `thinkrail` CLI。两者都
  嵌入同一个引擎宿主；应用状态位于 `~/.thinkrail` 下。
- **你的语言。** Settings → Language 跟随你的系统语言或你选择的语言；内置英语、日语、简体中文与
  西班牙语。把一份翻译后的 `en.json` 放到 `~/.thinkrail/locales`，即可添加 ThinkRail 尚未发布的
  语言 —— 保存时即生效。agent 的回复与工具输出保持 agent 写入时的样子。
- **你的主题。** Settings → Appearance 内置 dark、light、minimal 与高对比度主题。复制一个内置的
  `*.theme.json`，改动其 id 与颜色，保存到 `~/.thinkrail/themes`，即可制作你自己的主题。

<picture>
  <source media="(prefers-color-scheme: dark)" srcset=".github/readme-assets/specs-dark.png">
  <img src=".github/readme-assets/specs-light.png" alt="The Specs tool, opened from the right edge rail, listing the project's spec tree while the product goal spec is rendered in the editor" width="900">
</picture>

*项目的规格图谱以树呈现，规格本身在旁边一同渲染。*

## 安装（Install）

ThinkRail 以两种可叠加的形式发布：原生桌面安装包与自包含的 `thinkrail` CLI。两者都在
[releases 页面](https://github.com/JetBrains/thinkrail/releases)随 `SHA256SUMS` 一同发布。

### 桌面应用（Desktop）

下载适合你平台的 `thinkrail-desktop-*` 资产：

- **macOS 14+（Apple Silicon 或 Intel）** —— 打开对应架构的 DMG：
  `thinkrail-desktop-darwin-arm64.dmg` 或 `thinkrail-desktop-darwin-x64.dmg`。
- **Windows x64** —— 解压完整的安装 ZIP，然后就地运行其安装可执行文件，与它的载荷放在一起。
- **Linux x64 / ARM64** —— 解压安装 tarball 并运行 `installer`。桌面构建需要 glibc 2.38+ 以及
  GTK 3、WebKitGTK 4.1、Ayatana AppIndicator 3 与 librsvg 2 —— Ubuntu 24.04 或更新的版本：

  ```bash
  sudo apt install libgtk-3-0 libwebkit2gtk-4.1-0 libayatana-appindicator3-1 librsvg2-2
  ```

打包的 stable 与 nightly 桌面构建会在后台检查更新，并在 **Settings → Updates** 中提供：
**Download** → **Preparing update** → **Install & Restart**。在你选择 **Install & Restart** 之前
不会应用任何东西。随后 ThinkRail 显示安装的进度与日志，关闭，并在新版本上重新打开。重启后它会
告诉你更新是否已安装。如果没有，它会说明原因并提供重试。仅当安装包携带有效的 HTTPS feed 身份时
更新器才会出现；早于该身份的安装需要先手动安装一次。

### 命令行（Command line）

安装器会下载正确的二进制、校验其 SHA-256 校验和，并把 `thinkrail` 放到你的 PATH 上。

**macOS / Linux**（Windows 上在 Git Bash 中同样适用）：

```bash
curl -fsSL https://raw.githubusercontent.com/JetBrains/thinkrail/main/install.sh | bash
```

**Windows** —— 同样的命令在 cmd 与 PowerShell 中都能用：

```powershell
powershell -c "irm https://raw.githubusercontent.com/JetBrains/thinkrail/main/install.ps1 | iex"
```

然后运行 `thinkrail`，可选地带上一个作为项目打开的 git 仓库：

```bash
thinkrail ~/code/my-repo
```

`thinkrail` 需要 PATH 上有 `git`，并需要一个模型 provider —— 从应用中用 JetBrains AI 登录，或使用
你自己的 provider 凭据。首次启动会引导你连接 provider 并选择一个项目；
**Start a new project** 会创建一个空的 git 仓库（默认在 `~/thinkrail` 下），因此你不必事先准备
一个。**Add project → Clone repository…** 把一个 Git URL 克隆到运行 ThinkRail 的计算机上的新
文件夹，并作为项目打开。选择父文件夹与文件夹名，可选地限制历史深度，或取消进行中的克隆。GitHub
与其它 Git 宿主使用宿主已有的 Git 凭据与 SSH 密钥；已有的目标文件夹永远不会被覆盖。
`thinkrail --help` 列出所有参数；`thinkrail --version` 打印构建版本。

<details>
<summary><strong>Nightly 构建、固定版本、更新与卸载</strong></summary>

安装器接受 `--channel stable|nightly` 与 `--version`；固定的版本必须属于所选的 channel：

```bash
curl -fsSL https://raw.githubusercontent.com/JetBrains/thinkrail/main/install.sh | bash -s -- --channel nightly
curl -fsSL https://raw.githubusercontent.com/JetBrains/thinkrail/main/install.sh | bash -s -- --channel stable --version 0.1.2
curl -fsSL https://raw.githubusercontent.com/JetBrains/thinkrail/main/install.sh | bash -s -- --channel nightly --version 0.2.0-nightly.10
```

在 Windows 上，这些选项是环境变量（`THINKRAIL_CHANNEL`、`THINKRAIL_VERSION`、
`THINKRAIL_PREFIX`、`THINKRAIL_NO_MODIFY_PATH`）：

```powershell
$env:THINKRAIL_CHANNEL='nightly'; irm https://raw.githubusercontent.com/JetBrains/thinkrail/main/install.ps1 | iex   # PowerShell
set "THINKRAIL_CHANNEL=stable" && set "THINKRAIL_VERSION=0.1.2" && powershell -c "irm https://raw.githubusercontent.com/JetBrains/thinkrail/main/install.ps1 | iex"   # cmd
```

已安装的 stable 与 nightly CLI 宿主会定期在 **Settings → Updates** 中提供 **Run Update**。它在
后台运行宿主机的 `thinkrail update` —— 当 UI 在另一台机器的浏览器中打开时也是如此 —— 成功的更新
会要求你手动重启宿主。你可以在任意平台上直接运行 `thinkrail update`：它为已安装的 channel 重新
运行安装器并替换 `<prefix>/bin/thinkrail[.exe]`；只有需要显式覆盖时才传 `--channel` 或
`--version`。

`thinkrail uninstall` 移除可执行文件、安装器添加的 PATH 条目与安装元数据，并询问是否删除你的
`~/.thinkrail` 应用状态（默认保留 —— `--remove-data` 会删除它，`-y` 跳过这些询问）。

更喜欢手动安装？从 releases 页面下载一个二进制与 `SHA256SUMS`，并校验校验和。为了让自更新生效，
把它重命名为 `thinkrail`（Windows 上是 `thinkrail.exe`）并放到 `<prefix>/bin/thinkrail[.exe]`；
保留其发布文件名或放在其它位置的二进制必须手动替换，或用脚本重新安装。

</details>

### 平台说明（Platform notes）

| 平台 | 桌面 | CLI |
| --- | --- | --- |
| macOS Apple Silicon / Intel | DMG | 已签名二进制 |
| Windows x64 | 已签名的安装可执行文件与应用二进制 | 已签名的 `.exe` |
| Linux x64 / ARM64 | 安装 tarball，未签名 | 未签名 |

JetBrains 对 Windows CLI、桌面安装可执行文件，以及桌面应用安装的每个可执行文件与 DLL 签名。
macOS CLI 已签名，但尚未公证。已签名并公证的桌面 DMG 需要协调的 JetBrains 服务流水线；更早发布
的 DMG 与本地构建的包可能仍未签名，并会被 Gatekeeper 阻止。Linux 产物未签名。

## 分析与隐私（Analytics & privacy）

ThinkRail 应用会收集基础使用事件 —— 启动、创建聊天、发送消息、连接 provider —— 并绑定到一个随机
安装 ID。使用分析从不包含提示词、代码、文件、凭据或账号身份。错误报告也总是发送，绑定同一个安装
ID，以便我们修复崩溃：错误类型与代码，以及它发生在 ThinkRail 自己代码的哪个位置。它们从不包含
错误消息、文件路径、名称、URL，或来自你的项目或扩展的任何内容。额外的产品使用情况以及你是如何
找到 ThinkRail 的，仅在 **Share additional usage data** 打开时共享；该开关作为首次运行设置的一个
步骤提供（或在你跳过它之后立即提供），并位于 **Settings → Privacy**。从命令行启动时，
`thinkrail --no-analytics`（或 `THINKRAIL_NO_ANALYTICS=1`）会在该次运行中关闭额外共享。开启额外
共享时，打包构建可能会在你的浏览器中打开一次 ThinkRail 博客，把你的安装与你最初的网站访问关联
起来 —— 仅在你于网站上接受了营销 cookie 或无需同意的地方 —— 该关联在 30 天后过期。

反馈问卷与使用分析相互独立。ThinkRail 可能邀请你填写一份简短问卷，你也可以从
**Settings → Feedback** 打开它。你的回答，包括你输入的任何内容，只有在你按下 **Send** 时才会发给
ThinkRail 团队，无论你的分析设置如何，并且绑定同一个随机安装 ID —— 而不是你的姓名或账号。请不要
包含密钥、代码或个人数据；要删除你的回答，请发邮件至 info@thinkrail.ai。

ThinkRail 应用与 ThinkRail 网站（[thinkrail.ai](https://thinkrail.ai)）使用 PostHog Inc. 的服务
进行分析。网站还使用 PostHog Surveys 在桌面下载后收集反馈，并且仅在你选择填写时收集你的邮箱地址，
该地址用于进一步征求反馈并发送产品更新。PostHog 的隐私政策见
[posthog.com/privacy](https://posthog.com/privacy)。

## 技术实现（Under the hood）

ThinkRail 是一个薄宿主，在进程内运行 `pi` 并将其桥接到丰富的 Web UI。`pi` 拥有模型、skill、压缩、
成本与会话状态；应用拥有工作区、编辑器与线上契约 —— 它绝不会在你背后组装提示词。三个环：

- **引擎宿主** —— `packages/server`（+ `packages/shared`）：一个 `Bun.serve` HTTP+WS 宿主，每个
  聊天一个进程内 `pi` 会话，由 `apps/cli` 或 `apps/desktop` 启动。
- **线上契约** —— `packages/contracts`：宿主与 UI 之间带类型、带版本的协议。
- **UI 客户端** —— `apps/web`：React 19 + Zustand + Tailwind v4，独立发布；它通过线上契约拨通
  宿主，并只依赖 contracts。

[`goal-and-requirements.md`](goal-and-requirements.md) 与 [`architecture.md`](architecture.md) 是
规范的产品与设计规格。ThinkRail 采用规格先行的开发方式：每个模块旁都有 `SPEC.md`，而一处改动若
移动了边界、契约或决策，就要在同一次改动中更新规格。

## 开发 ThinkRail（Developing ThinkRail）

你需要 **Bun** 1.4.4（固定的包管理器与运行时）、**Node.js** ≥ 22.19（进程内 `pi` 引擎所需），
以及一个已认证的 `pi` provider 用于 agent 工作。

```bash
git clone https://github.com/JetBrains/thinkrail.git
cd thinkrail
bun install
bun run dev                          # host + web client together; Ctrl+C stops both
```

```bash
bun run --filter @thinkrail/cli dev  # browser launcher
bun run build:binary                 # standalone CLI artifact
bun run desktop:dev                  # package and open the Electrobun app
bun run desktop:build                # package without opening it
```

[`CONTRIBUTING.md`](CONTRIBUTING.md) 涵盖检查、端到端套件、桌面打包与仓库布局。

## 贡献（Contributing）

欢迎贡献 —— 见 [`CONTRIBUTING.md`](CONTRIBUTING.md)。本项目与社区受
[Code of Conduct](CODE_OF_CONDUCT.md) 约束。

## 社区（Community）

在 [Discord](https://thinkrail.ai/discord) 上加入我们，提问、分享你在构建的东西，并与团队交流。
Bug 与功能请求请提交到 [GitHub issues](https://github.com/JetBrains/thinkrail/issues)。

## 许可（License）

基于 [Apache License 2.0](LICENSE) 许可。
