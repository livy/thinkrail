# github —— 本地 gh 认证状态

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

为 New-Workspace 对话框的“Connected”药丸 + Refresh（以及 Settings 的“Local GitHub”区块）提供只读的
本地 GitHub CLI（`gh`）认证状态。仅在服务端 shell-out，使用宿主解析出的登录 PATH。

## 边界（Boundary）

- **拥有：** `githubAuthStatus()` → `Promise<{ connected, login?, scopes? }>`，通过 shell 执行
  `gh auth status`（从其报告中解析账号 + token 范围）；`githubRefresh()`（重新 shell 同一检查）。
  优雅降级——缺失 / 未认证 / 无响应的 `gh` 都 resolve 为 `{ connected: false }`，因此该对话框完全可
  离线工作。`THINKRAIL_GH_OFFLINE=1` 不 shell 直接强制为未连接结果（e2e 驱动离线路径）。
  还有 `ghSetupProblem()` → `Promise<GhSetupProblem | null>`——gh 直连路径不可用的*具名*原因
  （`Bun.which("gh")` 找不到二进制或探测从未启动时为 `missing`，探测以非零退出时为
  `unauthenticated`，gh 可用或离线接缝开启时为 `null`）。两个探测都在 `subprocess` 的 `runBounded`
  下以 8s 预算（`GH_PROBE_TIMEOUT_MS`）运行同一 `gh auth status`：`gh auth status` 会做一次网络往返，
  而宿主只有一个事件循环——这里一个 `spawnSync` 会在探测期间冻结每个会话，这正是 `runBounded`
  存在的目的（见 `subprocess/SPEC.md`）。**超时**的探测报告 `null`/`{ connected: false }`
  （瞬时网络停顿不是“已登出”——是静默比较回退，绝不是登录对话框）。`pr` 消费 `ghSetupProblem`
  （只在 gh 流程失败之后），使客户端能展示设置指引，而不是静默降级到比较页面。
- **公开接口（barrel）：** `githubAuthStatus`、`githubRefresh`、`ghSetupProblem`。
- **允许的依赖：** `contracts`（`GithubAuthStatus`）；`subprocess`（`runBounded`，两个探测都在其下
  spawn 的有界子进程）；Bun（`which`）。不触及 `git`/`projects`——它是纯粹的 `gh` 探测。
- **禁止：** `host`；兄弟特性模块；被打包进浏览器（`gh` 是 shell 执行的，永不打包）。
