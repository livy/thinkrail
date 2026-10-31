# 发布 pi 包

> 中文译文，原文见 [`README.md`](./README.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

`pi-extensions/*` 下各包的操作指南，这些包以 `@thinkrail/pi-<name>` 发布到 npm，并可在没有 ThinkRail
的情况下用于原生 `pi`。决策与理由见 [SPEC.md](SPEC.md)；本文件是检查清单。

## 新增一个包（Add a new package）

1. 在 `pi-extensions/<name>/` 创建包，位置**紧邻**它的前身（如果 `packages/` 下已有一个）；绝不在同一个
   PR 中修改或删除旧包——那由之后的接线 PR 完成。
2. `package.json`：复制 `visualize/package.json` 并调整。不可妥协的项：
   - `name: "@thinkrail/pi-<name>"`、`version: "0.0.0"`（第一个 changeset 产生第一个真实版本）、
     `private` 不出现、`license: "Apache-2.0"`、`repository.directory`、
     `publishConfig: { access: "public", provenance: true }`。
   - `keywords` 包含 `"pi-package"`，并且 `pi` 清单（`extensions`、`skills` … ）**仅**用于可安装的
     扩展。可移植的库（不带 factory）两者都不需要。
   - `files`：入口点与 `exports` 传递依赖到的全部内容，测试除外。对齐门禁会从 tarball 安装，所以任何
     遗漏都会在那里失败。
   - Pi SDK 包（`@earendil-works/pi-ai`、`pi-coding-agent`、`pi-tui`、`pi-agent-core`、`typebox`）为
     `peerDependencies: "*"` 与 `devDependencies: "catalog:"`。其他运行时依赖：精确锁定版本。
3. 从仓库根目录复制 `LICENSE`。编写 `README.md`，其最前面几行是
   `pi install npm:@thinkrail/pi-<name>` 以及该包增加了什么。
4. 创建 `SPEC.md`（`parent: module-pi-extensions`），并把该包加入 [SPEC.md](SPEC.md) 的成员表。
5. 在 `scripts/check-module-boundaries.ts` 中注册该包
   （`{ root: "pi-extensions/<name>", allowed: [] }` 或它所依赖的同级包），并在
   `scripts/checkModuleBoundaries.test.ts` 的夹具中注册。
6. 把它的预期加入 `scripts/check-pi-packages.ts` 的 `EXPECTATIONS`：工具名、skill 名，以及若干可执行
   用例（合法调用、被拒调用、一次 TUI 渲染）。在 `session_start` 内部注册的工具需要 session 绑定模式
   ——随第一个这样的包一起添加。
7. 运行 `bun install`，然后运行下方的达标线。

## 原生 pi 对齐达标线（The vanilla-parity bar (before the first publish)）

- `bun run check:pi-packages` 通过——它会打包、安装到**仓库之外**的隔离 pi 夹具中，并在 **Node**
  （原生 pi 的运行时）下加载，因此仅 Bun 可用的 API 与缺失的 `files` 都会在这里失败。
- 发布的代码是双运行时：只用 `node:` 内置模块与标准 ESM；不用 `Bun.*`、`bun:*`、`import.meta.dir`。
- 每个纯文本结果读起来很糟的工具都有 `renderCall`/`renderResult`；每处 UI 调用都在 `ctx.hasUI` 之后。
- 在真实终端中手动检查，包括在 `tmux` 内：
  ```bash
  node_modules/.bin/pi -e pi-extensions/<name>/index.ts
  ```
- 不使用 `~/.thinkrail` 路径，不使用宿主的线上契约，除同级 `pi-extensions/*` 之外不导入 `@thinkrail/*`。

## 发布一个版本（Release a version）

1. **在改动已发布包的那个 PR 中**添加一个 changeset：
   ```bash
   bun changeset            # pick the package(s), bump type, one-paragraph summary
   ```
   把生成的 `.changeset/<name>.md` 随改动一起提交。依赖方（`subagents` → `delegation`）会在其依赖发布
   时自动提升版本。
2. **要切出版本**，维护者在一个新分支上运行
   ```bash
   GITHUB_TOKEN="$(gh auth token)" bun run release:version  # changeset version + bun install (lockfile)
   ```
   （GitHub changelog 生成器需要一个 token 来查询 PR/作者链接；它本身不读取 `gh` 的凭据）
   并开一个普通 PR `chore(release): version pi packages`。它携带版本提升与 `CHANGELOG.md` 条目，并
   消耗掉 changeset 文件。普通 CI 会在它上面运行。
3. **合并到 `main` 后**，`.github/workflows/release-pi-packages.yml` 会重新运行对齐门禁，发布每个版本
   尚未出现在 npm 上的包（按依赖优先），并在每个当前版本都上了 npm 之后推送 `name@version` 标签——
   包括手动发布过的版本（bootstrap）以及先前推送失败而漏掉的标签。该任务运行在带 OIDC 的
   `npm-release` GitHub environment 中——任何地方都没有 npm token。只要还有任何 changeset 处于待处理
   状态，该任务就以 "nothing to publish" 提前退出，并且不打任何标签。

`bun run release:publish -- --dry-run` 会在本地显示该工作流将会发布什么。

## 新包的首次发布（一次性 bootstrap）（First publish of a new package (one-time bootstrap)）

npm Trusted Publishing 只能在已存在的包上配置，所以第一个版本必须由拥有 `thinkrail` npm 组织的
人来发布：

```bash
bun run release:pack <dir>                      # e.g. visualize → dist/pi-packages/<name>-<version>.tgz
npm login                                       # org owner, 2FA
npm publish dist/pi-packages/<tarball> --access public --provenance=false
```

这次手动发布必须使用 `--provenance=false`：清单中的 `publishConfig` 要求 provenance，而只有 CI provider
才能生成它。自动化发布仍保留 provenance。

然后注册 Trusted Publisher——用 CLI（同一个登录）：

```bash
npm trust github @thinkrail/pi-<name> --file release-pi-packages.yml --repo JetBrains/thinkrail \
  --env npm-release --allow-publish --allow-stage-publish
npm trust list @thinkrail/pi-<name>             # expect permissions: publish, stage publish
```

或在 npmjs.com 上 → 该包 → **Settings → Trusted Publisher → GitHub Actions**，填入同样的值
（`JetBrains` / `thinkrail` / `release-pi-packages.yml` / `npm-release`）并勾选
**Allow `npm publish`**。

两条 npm 规则决定该配置能否真正生效：

- **必须显式允许 `npm publish`。** 在 2026-09-03 之后创建的配置默认只允许 `npm stage publish`；发布
  工作流是直接发布，因此仅允许 stage 的配置会以 `ENEEDAUTH`/403 失败。`npm trust list` 会显示权限。
- **未使用会在 48 小时后过期。** 配置由它的第一次成功 OIDC 发布来验证；创建后 48 小时内没有发布过的
  配置会在 npmjs.com 上显示 *Expired*，必须撤销（`npm trust revoke <package> --id=<id>`）并重新创建。
  手动 bootstrap 发布**不会**验证它（它用的是登录凭据，不是 OIDC），所以要在**该包下一个版本 PR 即将
  合并时**创建该配置，而不是在 bootstrap 时——合并时的发布会验证它。

首次成功 OIDC 发布之后，把该包的发布访问权限设为 *Require two-factor authentication and disallow
tokens*。

## 故障排查（Troubleshooting）

- **`check:pi-packages` 在夹具内部报 module not found** —— 某个运行时导入不在 `dependencies` 中，或
  某个文件不在 `files` 中。
- **"uses a Bun-only API"** —— 把该调用移进 `*.test.ts`，或换成 `node:` 的等价实现。
- **发布任务全部跳过** —— `main` 上的版本已经在 npm 上（无事可做），或 changeset 仍在待处理（先把
  版本 PR 切出来）。
- **发布以 `ENEEDAUTH`/403 失败** —— 该包尚未配置 Trusted Publisher（见 bootstrap）、配置未使用已过期
  （48 小时规则）、只允许 `npm stage publish`，或者工作流/environment 名称与 npm 配置不完全一致。
  `npm trust list <package>` 会显示 npm 上已有的配置。
