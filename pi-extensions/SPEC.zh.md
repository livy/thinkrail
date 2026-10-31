# 可移植 pi 包（pi-extensions/）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

ThinkRail **发布到 npm** 且必须在**原生 `pi`** 中、没有 ThinkRail 存在时也能工作的 pi 包的工作区根。
这里的每个包都是 agent 可以在任何地方使用的一项能力（工具、skills）；ThinkRail 只是众多嵌入者之一。
某项能力的 ThinkRail 专属半边位于 [[module-thinkrail-extensions]]，绝不在这里。

在一个包的接线 PR 落地之前，它在 `packages/pi-*` 下的前身保持不动并由宿主加载——两份副本是刻意共存的，
接线 PR 会删除旧的那份。新包从 `version: 0.0.0` 开始，并从它的第一个 changeset 得到第一个真实版本。

## 身份与分发（Identity and distribution）

- npm 名为 `@thinkrail/pi-<name>`（npm 组织为 `thinkrail`；组织名 = scope），目录为
  `pi-extensions/<name>/`。私有工作区包共享该 scope 但保持 `private: true`；拥有该组织可以防止它们的名字
  在 npm 上被他人占用。最初发布在过渡性 `thinkrail.ai` 组织下的包（只有 `@thinkrail.ai/pi-visualize@0.1.0`）
  在那里被标记为弃用并指向这里。该 scope 是刻意的：无 scope 的 `pi-*` 名字会冲突（`pi-subagents` 是
  第三方包），而 scope 在 `pi.dev/packages` 画廊中标注归属，该画廊是带有 `pi-package` 关键字的 npm 包的
  索引——没有需要提交的 registry。
- **两种形态。** *可安装 pi 扩展*携带 `pi` 清单（`extensions: ["./index.ts"]`，存在时还有 `skills`，
  以及 `image`）和 `pi-package` 关键字。*可移植库*（例如 delegation）只有命名导出——没有清单、没有关键字、
  绝不是一个空操作工厂——并通过其打包后的消费方验证。
- **原始 TypeScript，无构建步骤**（pi 通过 jiti 加载入口点）。`files` 是包级白名单，
  涵盖入口点与 `exports` 传递性 import 的一切，测试除外；`exports` 保留既有消费方使用的每个子路径
  （例如 `./core`）。Pi SDK 包是 `peerDependencies: "*"`；存在 TUI 渲染时 `@earendil-works/pi-tui`
  是 `"*"` peer。被测试的 pi 版本是根目录的版本，并且绝不在这里重复声明（architecture 决策 10）。
- ThinkRail 宿主通过 `workspace:*` 消费这些包；发布产物由一致性门禁验证，而不是由宿主验证。

## 原生一致性门槛（Vanilla-parity bar）

在一个包首次发布之前必须满足，也是“在原生 pi 中可工作”的定义：

- 从它的**打包 tarball** 安装，依赖由 npm 解析，通过 pi 自己的解析器/加载器加载且零诊断，
  并注册其工具与 skills。（本地路径的 `pi install` 不会安装任何东西——它只记录路径——因此门禁从不使用它。）
- **双运行时。** 原生 pi 是一个在 Node ≥ 22.19 下用 jiti 的 npm CLI；ThinkRail 在 Bun 下运行同一份代码。
  发布的代码只使用 `node:` 内建模块和标准 ESM——不得出现 `Bun.*`、`bun:*` 或 Bun 专属的 `import.meta`。
- TUI：在纯文本回退读起来很糟的地方提供 `renderCall` 和 `renderResult`；每个 UI 调用都在 `ctx.hasUI` 之后。
  在真实终端（Ghostty/iTerm2，以及 tmux 内部）做一次**手工检查**是该门槛的一部分。
- 无 ThinkRail 假设：没有 `~/.thinkrail` 路径、没有宿主传输线、除兄弟可移植包外没有 `@thinkrail/*` import。
- 门禁：`bun run check:pi-packages`（`scripts/check-pi-packages.ts`，CI job `pi-packages`，不依赖 provider）。
  它打包该包以及每个待发的可移植工作区依赖，在 tarball 中 grep 仅 Bun 可用的 API，用 pi 自己的 npm 策略
  （`--legacy-peer-deps`）把 tarball 加上目录版本的 pi 安装到**仓库之外**的临时 agent 目录，
  配置 `packages: ["npm:@thinkrail/pi-<name>"]`，然后把 `scripts/pi-package-parity.node.mjs`
  **复制到夹具中**并在 Node 下运行它，这样 pi 从夹具解析（Node 忽略 `import.meta.resolve` 的父 URL——
  从脚本自身位置解析会静默地测试仓库里的 pi）。runner 通过 `DefaultResourceLoader.reload()` 加载，
  断言零错误以及预期的工具/skills，并执行每个包的用例表（工具调用、预期文本/错误、
  `renderResult(...).render(width)` 检查以及预期的 `terminate` 标志）。一个用例可以脚本化工具看到的 UI：
  `ui.mode`（`tui` 或 `rpc`），`ui.replies` 按顺序应答 `select`（一个数字选中该选项）和 `input`，
  其中 `null` 表示关闭，`ui.keys` 作为原始终端输入喂给 `ui.custom` 工厂返回的组件；
  没有 `ui` 的用例得到空上下文。*会话绑定*的工具（在 `session_start` 中注册，例如 subagents、
  background-commands）将需要一种在 pi 的 faux provider 上绑定隔离 `AgentSession` 的模式——
  与首个交付此类工具的波次一起添加。需要 token 的冒烟检查保持按需。

## 发布（Release）

`@changesets/cli`（目录锁定；`privatePackages: { version: false, tag: false }`，因此只有这个根会被
版本化或打标签）。改动已发布包的 PR 要添加一个 changeset（`bun changeset`）。
**版本化是人工步骤**：维护者在分支上运行 `bun run release:version`（`changeset version` + `bun install`）
并开一个普通 PR——JetBrains 的组织策略禁止 Actions 创建 PR，而且普通 PR 还能获得正常 CI，
机器人开的 PR 就不会。当该 PR 位于 `main` 上时，`.github/workflows/release-pi-packages.yml`
在 `npm-release` 环境中以 `id-token: write` 运行：重新运行一致性门禁，然后运行
`bun run release:publish`（`scripts/publish-pi-packages.ts`），该脚本在任何 changeset 仍然待发时
（版本尚未提升）提前退出，否则以**依赖先于依赖方**的顺序发布每个版本尚未上 npm 的包：`bun pm pack`
（把 `workspace:*`/`catalog:` 改写为精确版本，满足决策 10）→ 在 npm Trusted Publishing 下运行
`npm publish <tgz> --provenance --access public`（Bun 没有 OIDC 认证；npm ≥ 11.5.1）→
仅当脚本报告 `readyToTag=true` 时才 `changeset tag` 并推送：没有待发 changeset、不是 dry run，
且每个当前公开版本都已上 npm（现在或更早发布，例如通过手工引导）。因此提前退出绝不会给占位版本
`0.0.0` 打标签，而 `changeset tag` 会跳过已存在的标签，这让标签推送失败后的重试是安全的。

Trusted Publishing 按*已存在*的包配置，因此每个新包先要做一次手工引导发布：
`bun run release:pack <dir>` 打包它并打印给 npm 组织所有者使用的
`npm publish … --provenance=false` 命令（否则清单的 `publishConfig.provenance` 会让本地发布失败；
只有 CI 可以做证明）。受信发布者配置本身（`npm trust github …`）必须允许 `npm publish`
（npm 2026-09 之后的默认值是仅暂存），并且只由创建后 48 小时内的首次 OIDC 发布验证——
手工引导不算——因此它在包的下一个版本 PR 合并之前才创建，而不是在引导时创建。运维细节：
`pi-extensions/README.md`。

顺序：visualize → ask-user-question → delegation + subagents → background-commands → spec-graph →
todos。不发布：`pi-thinkrail-workflow`（工作区内部）、`pi-dag`（直到它有消费方）。

## 边界（Boundary）

- **允许的依赖：** pi SDK peer、兄弟 `pi-extensions/*`、按精确版本声明的普通 npm 运行时依赖。
- **禁止：** `@thinkrail/*` 宿主包、`packages/server`、`apps/*`、`thinkrail-extensions/*`。
- 兄弟之间的依赖边（例如 subagents → delegation）在存在时列在这里：
  `subagents → delegation`、`dag → delegation`。

## 成员（Members）

| 包 | 形态 | 备注 |
| --- | --- | --- |
| `visualize` | 扩展 | `visualize` 工具；`lovely-mermaid` 用于 TUI 框线绘制和尽力而为的原生校验；导出 `createVisualizeExtension({ validateMermaid })`，让嵌入者可注入严格校验器——ThinkRail 从 [[module-ext-visualize]] 注入。细节：[[module-pi-visualize]]。 |
| `ask-user-question` | 扩展 | `ask_user_question` 工具：在 pi 的交互模式下是终端问卷，在 RPC 宿主中每个问题一个对话框；导出 `createAskUserQuestionExtension({ ask, guidance })`，让嵌入者可以通过自己的 UI 作答，还导出一个不含运行时 import 的 `./core`。ThinkRail 从 [[module-ext-ask-user-question]] 组合它，并使用其应答邮箱之上的 asker。细节：[[module-pi-ask-user-question]]。 |
