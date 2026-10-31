# ThinkRail——产品目标与范围

> 中文译文，原文见 [`goal-and-requirements.md`](./goal-and-requirements.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 目标（Goal）

ThinkRail 是越用越好的 agentic IDE。它是一个薄宿主，运行 `pi` 并将其桥接到富交互 UI，把工作
隔离到独立的工作区，并通过活的规格与可复用的 skill 赋予 agent 持久的项目知识——于是 agent
工作无需终端即可上手，并保持隔离、可评审、以规格为依据：用 agent 构建，而不失去控制。

本文档描述产品当前的样子及其存在的原因，并随产品一起变化：落地一项能力就加在这里，改变一个
决策就重写在这里。它不包含版本或路线图。

## 问题（Problem）

`pi` 是一个刻意保持最小、可扩展的 harness。这种灵活性要付出时间代价：新手需要花很长时间学会
驾驭它，而有经验的用户仍然觉得终端对于 agent 周边的工作很笨拙。

- **并行 agent 在一个工作树里相互碰撞**——彼此碰撞，也与用户自己的编辑碰撞。
- **agent 的改动难以从 CLI 评审**：阅读 diff、在精确的行上评论，并把这份反馈交回给 agent。
- **意图会蒸发。** 聊天中做出的决策随会话结束而结束；下一个会话重新推导它们，或与之矛盾。

## 面向谁（Who it's for）

在真实 git 仓库上用 `pi` 构建的人——想要可视化入口的新手、想在 agent 周围有一个 IDE 又不想
放弃 pi 深度的有经验用户，以及想快速推进又不失去对 agent 所作所为控制权的 agent 优先
构建者。

## 价值（Value）

- **默认隔离。** ThinkRail 创建的工作区是一个拥有自己分支与 cwd 的 git worktree：agent
  并行运行，主分支在用户合并前保持干净。直接在项目文件夹（Default 工作区）中工作是显式选择，
  绝不是默认——除了 ThinkRail 刚创建为空的项目的那次初始化聊天，因为此时还没有可隔离的东西。
- **agent 周围的真正 IDE。** 编辑器、diff、终端与评审在一个 workbench 中，因此用户看到并
  操控 agent 的工作，而不是从回滚缓冲区里把它重建出来。
- **规格即真相。** 项目的意图存在于代码旁的规格图谱中，agent 会读取、搜索并维护它，因此
  意图能跨越会话存活。
- **pi 的全部，无所隐藏。** 模型、skill、扩展与会话状态仍是 pi 的；ThinkRail 增加的是一个
  界面，而不是第二个 agent。

## 原则（Principles）

每个功能都遵循的持久决策；强制它们的结构由 [[architecture]] 承载。

- **pi 是唯一的引擎**，在进程内运行——没有第二个运行时（不用 `claude-agent-sdk`）。`pi`
  拥有模型 registry、系统提示、skill/扩展、压缩、成本与会话状态。
- **通过喂料施加影响，绝不代为组装。** 功能只通过喂给 `pi` 的东西——prompt 上下文、文件、
  pi 自己的 skill/扩展——以及会话启动时携带的标志来塑造 agent。
- **暴露，不重算。** 宿主展示 `pi` 报告的内容，而不是推导自己的副本。
- **用户自己的工具与凭据。** Git、GitHub 与模型访问走用户的 `git`、`gh`、`central` CLI
  以及 pi 的 provider 认证；ThinkRail 自己不留任何账户或 token。
- **信任是显式的。** 被克隆的仓库可能注入 agent 的东西——它提交的 skill 别名、pi 自己的
  项目资源（`.pi/` 设置、扩展、skill、prompt、subagent 定义），以及它定义的 MCP 服务器
  ——只有在按项目授予信任之后才加载；由仓库定义的 MCP 服务器还需要按条目的批准，且该批准在
  条目变化时失效。
- **规格优先。** ThinkRail 以规格优先的方式构建，并帮助它打开的项目也这样做。

## 能力（Capabilities）

ThinkRail 在产品层面做了什么；细节由链接的规格拥有。

- **两个启动器，一个应用**——原生桌面应用与在浏览器中打开同一 UI 的 `thinkrail` CLI，
  二者都嵌入同一个宿主；状态存放在 `~/.thinkrail` 下（[[module-desktop]]、[[module-cli]]）。
- **项目 → 工作区**——git 仓库即项目；工作区是一个 git worktree，外加内置的 Default 工作区
  （项目文件夹本身）与就地附加的已有 worktree（[[submodule-server-workspaces]]）。可以把
  Git 仓库 URL 克隆到选定的新宿主文件夹并作为项目打开，支持可选的浅历史与取消
  （[[submodule-server-projects]]）。
- **Workbench**——文件、diff、文档、聊天与终端的可分割布局，侧边与底部工具面板可移动，且
  保持在每个窗口本地（[[submodule-web-shell-layout]]）。
- **聊天**——每个工作区并发的 `pi` 会话，各自拥有模型、pi 报告的 token/成本、运行中的操控
  与后续指令、skill 与 prompt 模板补全，以及历史搜索（[[submodule-web-chat]]、
  [[submodule-server-agent]]）。
- **复用已有 skill**——为其他编程 agent 保留的可移植 Agent Skills 被就地读取；`pi` 仍是
  解析器与运行时（[[submodule-server-agent]]）。
- **带逐步评审的计划**——一份 agent 执行、用户可编辑的共享 TODO 计划；当某个步骤的改动可
  证明只属于它自己时，该步骤会独立提交，并可由独立评审者评审（[[module-pi-todos]]、
  [[submodule-server-todos]]、[[submodule-server-host-plan-review]]）。
- **Subagent**——委托给隔离的子会话，可前台、并行或后台执行（[[module-pi-subagents]]）。
- **评审**——在文件与 diff 上的锚定评论，无需启动 agent 即可收集，并作为结构化上下文发送给
  能解决它们的聊天；属于本地评审，不是 forge 集成（[[submodule-server-reviews]]）。
- **Open PR**——推送工作区分支并通过用户的 `gh` 打开或更新其 GitHub PR，回退到预填的
  compare URL（GitLab：预填的新 MR 页面），入口可以是计划，或直接从顶栏；顶栏的 Remote 与
  Pull request 片段始终显示该分支的下一步——发布、推送、同步、按计划描述的 PR、提交、模板
  或 agent 草稿——对于已打开的 GitHub PR，还显示其检查与评审决定，并支持标记就绪、合并、
  修复失败的检查，或用 agent 整合已分叉的分支、重开、删除远程分支
  （[[submodule-server-pr]]、[[submodule-web-shell-location-bar]]）。
- **规格**——agent 通过 `spec_*` 工具搜索、导航并维护规格图谱；只读的 Specs 工具把它渲染
  为树（[[module-spec-graph]]、[[submodule-server-spec]]）。
- **Workflow**——随附的 workflow skill，用于项目初始化、设计头脑风暴与交付 PR
  （[[module-thinkrail-workflow]]）。
- **Provider**——通过 pi 的认证在应用内登录与配置 API key，以及通过用户的 `central` CLI
  使用 JetBrains AI（[[submodule-server-auth]]、[[central-integration]]）。
- **Pi 配置**——用户自己的 pi 配置文件夹，或另一个（个人与工作两套配置），加上额外的
  环境变量，在 Settings 中选择，并在下次启动时应用到整个宿主；每个宿主一套配置，而不是每个
  项目一套（[[submodule-server-environment]]）。
- **Agent 工具**——web 研究、agent 在聊天中提出的结构化多选题（而不是猜测），以及聊天中的
  内联图表、对比与文件预览；提议的 UI 改动通过渲染出的前后对比 mockup 展示，使用与 diff
  评审相同的渲染器与评审评论（[[submodule-web-chat-tools-web]]、
  [[module-ext-ask-user-question]]、[[module-ext-visualize]]）。
- **MCP 服务器**——pi 内置的 MCP，从用户的 `mcp.json` 读取，可在 Settings 中管理，并在
  每个聊天中与后台命令、subagent 并列可见；对于可能改动数据的调用，在聊天中、在调用处先
  确认，用户信任的工具可按项目设置 “Always allow”（[[submodule-server-agent]]、
  [[submodule-web-panels]]）。
- **工作区周边**——在已安装的编辑器或 IDE 中打开 worktree，并保持应用自身更新
  （[[submodule-server-editors]]、[[submodule-web-updates]]）。
- **界面语言**——UI 跟随系统语言或用户的选择；英文与一组译文随附内置（未翻译的字符串以
  英文显示），用户可用 `~/.thinkrail/locales` 中的扁平 JSON 文件添加或覆盖一种语言，保存
  即生效。agent 与工具输出绝不翻译（[[module-locales]]、[[submodule-web-i18n]]）。
- **主题**——浅色、深色、极简与高对比主题随附内置，跟随系统外观或固定选择；用户在
  `~/.thinkrail/themes` 中保存一份内置主题文件的副本并换上新配色即可添加主题，保存即生效
  （[[submodule-web-themes]]）。
- **品牌**——ThinkRail 绿色强调色（深色主题上更亮，浅色主题上加深，以便两者都达到 AA），
  Darcula 系列深色背景，品牌展示角色用 Orbitron，UI 与代码用 Geist 与 JetBrains Mono
  （[[submodule-web-themes]]、[[web-typography]]）。

## 非目标（Non-goals）

只有这些是被决策排除的。既未在上面列出、也未在此排除的东西都是开放的，而不是被禁止的。

- **第二个 agent 运行时**，或任何宿主侧的 agent 循环或 prompt 组装（见原则）。
- **ThinkRail 账户或登录 UI。** 谁能访问宿主由网络决定——Tailscale ACL 与设备身份
  （[[architecture]]）。
