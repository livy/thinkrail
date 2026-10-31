# 通过 Central CLI 使用 JetBrains AI——跨模块生命周期

> 中文译文，原文见 [`central-integration.md`](./central-integration.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 驱动因素（Drivers）

JetBrains AI 模型经由用户自己的 `central` CLI 到达 ThinkRail，该 CLI 会把一个 `pi` 扩展写到
一个全局路径，并报告账户的周期性 AI credit 配额。因此 Central 表现为一条跨模块链条，而不是
一个模块边界。它的端到端生命周期映射、存活性与配额读取组装只存在于这条链条中。

**边界：本节点只拥有组装，别的一概不拥有。** 每个模块的规格对其自身界面保持权威，且不在此
重述——适配器的 argv/版本/配额解析契约是 [[module-shared]]，状态/动作/配额编排是
[[submodule-server-auth]]，运行时代际是 [[submodule-server-agent]]，同步的配额偏好是
[[submodule-server-settings]]，传输线形状是 [[module-contracts]]，provider 控件是
[[submodule-web-panels]]，全局读数是 [[submodule-web-shell]]。只影响一个界面的改动更新那个
叶子规格，而不是本文件。

## 链条（The chain）

唯一的端到端映射。每一列由表头中的模块拥有；这张表是唯一陈述它们之间*对应关系*的地方。

| 磁盘 / CLI 事实 | `inspectJbcentral`（`shared`） | 传输线 `JbcentralStatus`（`contracts`） | 卡片（`panels`） |
| --- | --- | --- | --- |
| 无法解析到 `central` | `absent` | `absent` | 宿主操作系统安装命令 + Recheck |
| 版本低于 `MINIMUM_CENTRAL_VERSION` | `outdated` | `outdated` | 引导式 Update |
| `--version` 无法解析 | `malformed-version` | `malformed-version` | 重装指引，无原生动作 |
| 探针启动/超时/退出失败 | `probe-failed` | `probe-failed` | 重试检查指引 |
| 二进制存在，产物缺失 | `supported` | `supported` | Connect |
| 二进制与产物都存在 | `supported` + `configured` | `configured` | Connected / Disconnect |
| …且 `Auth` 行为 `not connected` | —（观察） | `signedOut` 标志 | Sign in *替换*主操作 |
| …且 `Proxy` 行为 stopped | —（观察） | `proxyStopped` 标志 | Start proxy *替换* Disconnect |
| 有待完成的重建或动作在进行中 | — | `configuring` | 转圈，不提供动作 |
| 候选运行时加载失败 | — | `load-failed` | Retry / Disconnect |

### 配额链条（Quota chain）

当配额显示被启用时，可见的 web shell 按配置的 `1–3600` 秒节奏（默认 30 秒）请求一个封闭的
配额快照。宿主先应用同步设置以及 provider 卡片标记为 Connected 时所用的同一个健康 Central
谓词，然后服务端 `auth` 通过一个仅内存的缓存/单飞（single-flight）对读取者去重，并要求
`shared` 以 `central quota --json` 运行该绝对路径的二进制。`shared` 只接受周期性的
`remaining` + `total` 数字；`contracts` 只携带 hidden/available/stale/unavailable 状态；
`shell` 把它们渲染在宿主连接状态左侧紧邻处。隐藏的前端不轮询，禁用/不健康状态不做配额读取，
任何配额值都不被持久化。一次生命周期边会清除最后一次成功的回退值，因此后来的账户不会继承
先前账户的陈旧数字。

生命周期表编码了两条没有任何单个模块陈述的事实：

- **`configuring` 与 `load-failed` 没有检查侧对应物。** 它们是宿主重建机制的性质，而不是
  宿主文件系统的性质，因此只跟随 `shared` 的读者无法推导出它们。
- **signed-out 与 proxy-stopped 标志是单向的。** 只有正面观察到否定结果才会设置它们；未知
  永不设置。卡片把每一个都变成对主操作的*替换*，而不是一条注解，因为它绝不能宣传一个无法
  成功的动作。

## 决策（Decisions）

1. **一个被审查的产物把检查与运行时加载连起来。** 适配器拥有它的位置、存在性判定与监听器；
   运行时只接收它的不透明路径。由于“已配置”的真相只依赖存在性，就地替换依赖监听器而不是
   存在性轮询。本地机制在 [[module-shared]] 与 [[submodule-server-agent]] 中保持权威。
2. **任何源自 Central 的文本都不到达客户端。** 进程适配器、扩展前的 provider 允许列表与
   封闭的传输线状态各自执行这一保证的一部分；它们的本地契约留在 [[module-shared]]、
   [[submodule-server-agent]]、[[submodule-server-auth]] 与 [[module-contracts]] 中。产物
   重新注册的内置项（`anthropic`、`openai`、`google-vertex`）是 Central 在卡片之外*唯一*
   可见的地方：它们仍是普通 provider 行，标记为 `kind: "central"`（“JetBrains AI”），按
   注册身份归属，因此用户能看到哪些 provider 通过 Central 到达，而 Central 的配置没有任何
   一部分跨过传输线。
   配额以结构化的数字允许列表扩展该规则：账户、计划、用量、充值、续充、诊断与原始输出保持
   宿主本地并被丢弃。
3. **配额是一次独立读取，不是 provider 状态，也不是宿主 ticker。** provider 生命周期与配额
   有不同的延迟/失败语义，而宿主拥有的定时器会在没有可见消费者的情况下运行。可见客户端拥有
   节奏；一个宿主缓存/单飞防止它们成倍放大 CLI 工作。

## 不变量（Invariants）

1. **监听器的失效代表被审查产物上可能的边，而不是目录活动。** 平台事件分类与替换恢复机制
   只属于 [[module-shared]]。
2. **进入 `configuring` 的每条路径都有有界的出口。** 动作由适配器超时界定；未完成的重建在
   排水（drain）落定时退出。传输线状态本身不携带截止时间。
3. **重建排水在有界时间内落定被请求的状态，且独立于入站事件的速率。** 防抖限制的是突发
   宽度，而不是流的长度，因此它不能提供这一保证。

**不变量 3 目前被违反，由 issue #287 跟踪。** 当 `requestedSequence` 在准备期间发生变化时，
`runRebuildDrain` 会丢弃已完成的候选，因此推进仍需要一个安静窗口，激活可能无限期饥饿。
按产物范围的失效移除了已知的无关事件来源，但没有改变该算法。

**复盘。** 当产物目录不存在时，监听器回退到 `pi` 的状态目录，把无关的状态写入变成了重建
请求。排水的安静窗口假设使 `configuring` 在反复准备运行时的同时保持活动。每份叶子契约在
局部都是一致的，但没有规格拥有它们在组合中的存活性。

## 范围之外（Out of scope）

适配器的确切 argv/schema/超时；provider 卡片与顶栏的交互/文案；设置校验；候选准备与代际
激活；传输线字段形状；启动顺序；e2e 夹具。每一项都留在自己的模块规格中——这里只列出以便
读者知道本文件不是改动它们的地方。

## 被谁消费（Consumed by）

[[module-shared]] · [[submodule-server-auth]] · [[submodule-server-agent]] ·
[[submodule-server-settings]] · [[module-contracts]] · [[submodule-web-store]] ·
[[submodule-web-panels]] · [[submodule-web-shell]] · [[module-cli]]（启动顺序） ·
[[module-browser-e2e]]（`00-jbcentral*`）
