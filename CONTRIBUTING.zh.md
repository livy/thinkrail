# 为 ThinkRail 做贡献

> 中文译文，原文见 [`CONTRIBUTING.md`](./CONTRIBUTING.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

感谢你对 ThinkRail 的关注！我们欢迎各种形式的贡献——缺陷报告、修复、新功能、文档和反馈。

参与贡献即表示你同意你的贡献以 [Apache License 2.0](LICENSE) 授权，并且同意遵守我们的
[行为准则](CODE_OF_CONDUCT.md)。

## 报告缺陷与请求新功能（Reporting bugs and requesting features）

请开一个 GitHub issue。提交前先搜索已有 issue 以避免重复。缺陷报告请包含：

- 你做了什么、期望什么，以及实际发生了什么
- 你的操作系统、Bun 与 Node 版本
- 相关日志或最小复现步骤

## 开发环境搭建（Development setup）

**前置条件：**[Bun](https://bun.sh) 1.4.4（仓库固定的包管理器与运行时）、Node.js ≥ 22.19
（进程内 `pi` 引擎的要求），以及一个已完成认证的 `pi` provider 用于 agent 相关工作。

```bash
git clone <repo-url>
cd thinkrail
bun install
bun run dev
```

`bun run dev` 会同时启动宿主与 web 客户端，并在 `Ctrl+C` 时清理。源码/开发运行与本地未打戳的构建
不暴露应用更新器：更新检出请用 Git 并重新构建。在源码目录里运行 `thinkrail update` 会安装一个已发布
的二进制，它不会拉取代码、安装依赖或重新构建检出。磁盘上的应用状态（项目、工作区、worktree）位于
`~/.thinkrail` 下。

运行各个启动器：

```bash
bun run --filter @thinkrail/cli dev  # browser launcher
bun run build:binary                 # standalone CLI artifact
bun run desktop:dev                  # package and open the Electrobun app
bun run desktop:build                # package without opening it
```

桌面端命令使用标准的 Electrobun CLI/配置。它的预构建钩子会构建共享 UI 并暂存 ThinkRail 的
PI/原生资源；Electrobun 负责 preload 打包与安装包创建。用 `bun run desktop:package:stable` 或
`bun run desktop:package:canary` 创建宿主原生安装包；`canary` 是 Electrobun 对公开 **nightly**
通道的内部称呼。原生/安装包冒烟与共享的 CLI/桌面端探针位于 `packages/artifact-tests`，即在应用包之外。
开发构建之后运行 `bun run smoke:desktop`；安装包冒烟通过
`bun run smoke:desktop:installer <path> <stable|canary>` 接收产物路径与通道。本地安装包冒烟不等同于
公证验证：协同的 macOS 发布流水线会为 Electrobun 解包后的应用归档签名并完成 DMG，而该中间归档不是
公开下载项。

## 仓库布局（Repository layout）

[`architecture.md`](architecture.md) 列出了每个 app 与 package 及其职责和允许的依赖；每个模块都带有
自己的 `SPEC.md`。

## 测试与 lint（Testing and linting）

在开 PR 之前运行这些快速门禁（它们同时也是 husky pre-commit 钩子）：

```bash
bun run lint        # biome
bun run typecheck   # tsc across all packages
bun run test        # unit tests (root tooling + each package)
```

任何触及应用的改动都通过端到端套件针对真实 UI 验证。no-agent 门禁只构建一次，并自动使用隔离的并行
分片（可用 CPU 的一半，上限为八）：

```bash
bunx playwright install chromium                    # one-time
bun run e2e                                         # complete no-agent gate
bun run e2e -- e2e/changes.spec.ts                  # focused iteration
bun run e2e -- --last-failed                        # repair loop
bun run e2e:serial                                  # one-host debugging fallback
bun run e2e -- --shards=12                          # explicit 1–16 override
bun run e2e:binary                                  # packaged CLI host (build first)
bun run e2e:desktop                                 # packaged desktop host (build first)
bun run e2e:full                                    # everything; needs pi auth
bun run e2e:agent                                   # only @agent; remains serial
```

迭代过程中使用聚焦运行或 last-failed 运行，交接前完整运行一次 `bun run e2e`。在 macOS 上，每个公开的
浏览器 E2E 命令都会在 runner 生命周期内阻止系统闲置睡眠，同时允许显示器正常休眠。由 agent 驱动的规格
带有 `@agent` 标记，并针对真实 provider 在**隔离的** `pi` agent 目录中运行——绝不是你的真实
`~/.pi/agent`。

## 模块边界（Module boundaries）

ThinkRail 由一组边界清晰的分形模块构成：每个 package 是一个模块，package 内部的目录也是模块。子模块
通过 `index.ts` barrel 暴露其公开接口；同级模块通过该 barrel 导入，绝不直接导入其内部实现。每个模块都
有一个 `SPEC.md`，说明它拥有什么、暴露什么，以及绝不能伸入什么。改动请保持在这些边界之内，并在可行的
情况下用测试覆盖模块的公开接口。

## 规格驱动的开发（Specification-driven development）

ThinkRail 以规格优先的方式开发：分层的、相互关联的规格与代码一同放在仓库中——顶层规格位于仓库根目录
（[`goal-and-requirements.md`](goal-and-requirements.md)、[`architecture.md`](architecture.md)），
每个模块都有一个同目录的 `SPEC.md`。**规格先于代码：** 任何移动或模糊边界、契约或决策的改动，都先更新
相关的 `SPEC.md`，再改代码和锁定它的测试。规格工作流见 [`AGENTS.md`](AGENTS.md)。

## 翻译（Translations）

ThinkRail 的界面字符串集中在一个文件里：[`packages/locales/en.json`](packages/locales/en.json)，它是
一个由具名键组成的扁平 JSON 对象（`"settings.language.title": "Language"`）。一份翻译就是同一个文件，
只是值被翻译过。

1. 把 `en.json` 复制到 `~/.thinkrail/locales/`，以你语言的代码命名——`de.json`、`pt-BR.json`、
   `zh-Hant.json`。保留键；翻译值。你没有提供的字符串保持英文。
2. 每个 `{{placeholder}}` 都要照原样保留（占位符与英文不一致的字符串会被忽略），`<0>…</0>` 标记也要
   包住同样的词。用户必须逐字输入的字面量也要保留——例如 OpenSSH 的 `yes` 这类协议应答、JSON 键、
   标志或命令——即使周围的散文已被翻译。
3. 计数器以复数形式出现：英文有 `key_one` 和 `key_other`；请按你的语言实际使用的形式从 `_zero`、
   `_one`、`_two`、`_few`、`_many`、`_other` 中书写
   （[CLDR 复数规则](https://www.unicode.org/cldr/charts/latest/supplemental/language_plural_rules.html)）。
   例如俄语使用 `_one`（1、21、31…）、`_few`（2–4、22–24…）、`_many`（5–20…）和 `_other`：
   `"tools.read.firstLines_one": "первая {{count}} строка"`、
   `"tools.read.firstLines_few": "первые {{count}} строки"`、
   `"tools.read.firstLines_many": "первые {{count}} строк"`。每种形式都可以显示 `{{count}}` 或
   省略它；你跳过的形式会回退到你的 `_other`，因此计数器永远不会切回英文。
4. 打开 **Settings → Language** 并选择你的语言。每次保存时 ThinkRail 都会重新加载该文件；如果文件
   无法读取，页面会显示该文件的问题。

要让一份翻译内置给所有人，请开一个 pull request：把它作为 `packages/locales/<code>.json` 添加，在
`packages/locales/src/index.ts` 的 `BUNDLED` 中注册它，并把该语言加入 `README.md` 的内置语言列表
——不需要改其他东西。它可以是部分的；`bun run test` 会对照 `en.json` 检查它（键、占位符、标记、
复数形式）。agent 回复、工具输出、skills 与 prompts 不属于界面，不会被翻译。

## 提交改动（Submitting changes）

1. [Fork 该仓库](https://docs.github.com/articles/fork-a-repo)并从 `main` 创建一个分支。
2. 进行你的改动，保持聚焦。与周围的代码风格以及 `AGENTS.md` 中的约定保持一致。
3. 添加或更新测试与规格，并运行上面的检查。
4. 针对 `main` 开一个 [pull request](https://docs.github.com/articles/creating-a-pull-request)，
   清楚说明*改了什么*以及*为什么*。

提交信息与 PR 描述要解释改动背后的理由，而不只是操作步骤。维护者会评审你的 PR，并可能在合并前提出
调整建议。
