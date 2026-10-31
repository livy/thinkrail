# settings —— 服务器同步的应用配置

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

服务器同步的应用配置：不透明的固定主题选择、固定/系统模式与可选明暗配对、附加分析偏好与首次运行
对话框完成标记、宿主拥有的新聊天模型与力度默认值、终端回放预算与 Windows shell 偏好、聊天撰写框增长
预设、聊天/文件视觉行宽加上独立的窗格界限、有界的自定义布局预设目录、JetBrains 配额显示/节奏、宿主
范围的子 agent 默认值、计划评审策略、用户关掉的宿主注册表扩展，以及 UI 语言偏好（`locale`）。
`defaultModel` / `defaultEffort` 选择每个新聊天收到的默认值（未设模型意味着首选可用模型 —— pi 的
provider 默认，否则第一个；未设力度意味着 medium）；`reviewModel` / `reviewEffort` 选择评审人
runtime（未设意味着同样的新聊天默认值）；`reviewAutoFix: false` 记录一个 `request_changes` 结论并
等待，而不是自动发送修复。**`favoriteModels`** 是选择器的星标列表，按展示顺序 —— 一次整列表的客户端
写入，校验为模型形状的条目并按 `sameModel` 去重；**`recentModels`** 由宿主拥有：
`noteRecentModel(model)`（由 `host` 在一次显式 `session.create` 模型与每次 `session.setModel` 时调用）
把该模型放到最前、去重、夹到 `RECENT_MODELS_LIMIT`、持久化并发布，而 `settings.update` 中客户端提供的
`recentModels` 在校验之前就被丢弃，因此没有客户端能改写宿主关于「选了什么」的记录。本模块读取、
规范化、持久化、缓存并广播那些刻意跨前端跟随所有者的值。

当前工作台框架、工作区资源放置、当前/默认预设选择、侧/底组界限、选择与焦点明确不在此列。它们是
[[submodule-web-shell-layout-state]] 之下的前端表面本地视图状态。内建布局预设仍由 web 拥有。

一个数值设置在某个领域拥有安全上限时由该消费者限界 —— 例如 `terminal` 夹住 `terminalReplayKb`，因此
手工编辑的配置无法耗尽内存。settings 校验共享的 40–240 行宽契约与自定义布局预设，因为它拥有它们的
跨前端存储契约。它还校验全局 `jbcentralQuotaEnabled` 布尔值与整个 `jbcentralQuotaRefreshSeconds`
间隔（`1–3600`，默认 30），因为这些值在每个前端上支配宿主进程的节奏。

## 边界（Boundary）

- `settleIdleDays` —— Settled 置物架的空闲窗口：一个正的整数天数（夹到 1…365），或 `null` 表示永不；
  默认 `7`。像其他每个已知字段一样校验；手工编辑出的非数字会回退到默认值，而不是禁用结算。
- **拥有：** 缓存的当前 `AppConfig`；`getConfig()`；`updateConfig(partial)`（拒绝非对象或未知键 → 合并
  → 校验已知字段 → 持久化 → 发布合并后的 `AppConfig` 与成功应用的 `AppConfigUpdate`）；行宽与无资源
  自定义预设的校验/规范化；自定义预设的安全上限；`setSettingsPublisher`；以及供测试用的
  `resetConfigCache`。
- **公开接口（barrel）：** `getConfig`、`updateConfig`、`noteRecentModel`、`setSettingsPublisher`、
  `SettingsPublisher`、`resetConfigCache`，以及在持久化加载之后由宿主启动所用的纯自定义预设规范化。
- **允许的依赖：** `persistence`（`loadConfig`/`saveConfig`）；`contracts`（`AppConfig`、
  `LayoutPreset`、`isTerminalWindowsShell`）。
- **禁止：** host 或另一个特性兄弟；当前布局文档/快照类型；工作区 id/资源；当前框架校验；拥有 WS
  通道；或导入 web 的预设定义。

## 必须做对（Get right）

- **在广播上收敛，无客户端乐观更新。** `updateConfig` 在替换在线缓存或发布之前先持久化；写入失败既
  不改动 runtime 读取，也不改前端。发布者收到完整合并配置用于广播，以及被应用的部分更新，好让宿主
  消费者能区分一次显式字段写入与一次无关的合并。每个前端，包括发起者，都采纳 `settings.changed`。
  `server.welcome` 播种同一个缓存值。
- `terminalWindowsShell` 默认 `"auto"`；已持久化的值回退经由 persistence 共享的
  `isTerminalWindowsShell` 守卫，而封闭 shell 联合之外的一次线上变更会在缓存、持久化或广播变化之前
  拒绝整个更新。settings 拥有校验与同步；`terminal` 拥有可执行文件解析与 spawn。
- `chatLineWidth` / `fileLineWidth` 各自默认 120，只接受 40 到 240 的有限整数；它们各自的 `Bounded`
  开关独立地默认 `true`。一个畸形的已存字段会回退，而不丢弃有效的兄弟字段；任何提供的无效字段会在
  缓存、持久化或广播变化之前拒绝整个变更。
- `subagentsEnabled` 在缺失时默认 `true`，因此旧配置保留当前行为；一个提供的非布尔更新会在缓存、
  持久化或广播变化之前被拒绝。settings 只拥有那个全局默认值；工作区覆盖与有效值解析留在本模块之外。
- `onboardingDismissed` 在磁盘上缺失或畸形时默认 `false`；一个提供的非布尔更新像其他布尔字段一样被
  拒绝。只有 web 的入驻向导读它。
- `subagentMaxConcurrent`（每聊天并发子 agent 数）在磁盘上缺失或畸形时默认 4；一次线上更新落在
  `isSubagentMaxConcurrent`（整个 1–16）之外会拒绝整个变更 —— 永不夹取。与 `subagentsEnabled` 一样，
  settings 只拥有全局值。
- `disabledExtensions` 列出用户关掉的宿主注册表扩展 `name`；缺失意味着 `[]`（每个扩展都开）。一次线上
  更新必须是非空白字符串的列表，否则整个变更被拒绝；重复被去重，而**未知名字被保留** —— 是注册表，而
  不是 settings，决定一个名字的含义，因此一个针对已移除扩展的陈旧条目永不会破坏启动或后续更新。一个
  畸形的已存列表保留其有效名字。settings 只拥有该列表；`host` 把它变成 agent 的解析器与在线工具刷新
  （[[module-server]] § 宿主中介的策略）。
- `locale` 是 `"system"`（默认）或一个规范的 BCP-47 标签（`isLocalePreference`：`pt-BR`，不是
  `pt-br`）。settings 从不检查某种语言是否可用 —— 打包的目录属于 web 构建，本地文件属于
  [[submodule-server-locales]] —— 因此一个其文件被删除的标签仍持久化，并在文件回来时回来。一个畸形的
  已存值回退到 `"system"`；一个畸形的更新拒绝整个变更。
- JetBrains 配额显示默认开启，其间隔在两个已存字段任一缺失/无效时默认 30 秒。线上更新会原子地拒绝一个
  非布尔标志或一个非整数/超范围的间隔；它们永不像夹取调用方的值那样把它变成一个不同的已持久化选择。
- 主题可用性/标签/调色板、操作系统外观以及有效主题不是服务器关心的事。`theme` 仍是不透明的固定选择；
  `themeMode` 默认 `"fixed"`，而 `systemThemePair` 在首次使用之前保持缺失。已持久化的配对只在两个槽
  都是字符串时保留；畸形的配对被丢弃，而没有保留配对的系统模式会规范化为 fixed，却不替换有效的 fixed
  id。缺失/无效的模式也规范化为 fixed，而一个独立有效的休眠配对可以存活。进入系统模式需要一份完整且
  形状有效的既有或新来配对；一次配对变更原子地替换两个槽。未知 id 仍被持久化，供每个独立交付的前端
  按所需外观解析。
- 一次携带 `theme` 而无显式 `themeMode` 的 `settings.update` 是一个旧版兼容的固定主题动作，并把模式设为
  `"fixed"`。因此一个连接到系统配置宿主上的旧客户端永不可能看起来只改变它自己：它的刻意主题选择会经
  通常的「先持久化后广播」路径退出系统模式。
- 已退役的宿主布局与聊天消息顺序字段，以及宿主拥有的 `recentModels`，会在一次更新中被静默丢弃，而不
  被持久化或广播，因此仍发送它们的旧客户端继续可用。布局实例化与转录顺序是前端本地偏好。
- **更新键是封闭集合；已存键不是。** `updateConfig` 在缓存、持久化或广播变化之前拒绝非对象载荷以及
  `AppConfigUpdate` 加已退役键之外的任何键。被接受的集合对着 `AppConfigUpdate` 编译期检查，因此一个
  增删键的契约变更会让构建失败，直到集合与之匹配。磁盘上已有的未知键仍会加载并在后续有效更新中存活，
  因此较旧的宿主保留较新宿主的设置。较新的客户端必须按宿主协议版本为一个新键设门禁，就像
  `THEME_SYSTEM_PROTOCOL_VERSION` 为外观设置设门禁那样；较旧的宿主会拒绝整个更新，而不是持久化一个
  未知键。
- **一个被接受的键在进入合并之前会做值校验；只有力度层级是被信任的。** `terminalReplayKb` 必须是有限
  数字（其范围留在 `terminal`，它读取时夹取）、`composerGrowthLimit` 必须是封闭增长预设之一，而
  `defaultModel` / `reviewModel` 必须是 `null` 或像 `favoriteModels` 条目那样的模型形状，与其他字段
  已有的类型化检查并列。一个畸形值会在缓存、持久化或广播变化之前拒绝整个更新，完全像未知键一样。
  `defaultEffort` / `reviewEffort` 携带 pi 的 `ThinkingLevel`，`contracts` 对它没有运行时列表，因此
  它们仅凭线上类型通过，而新聊天解析器经 pi 夹取一个未知层级；给 `contracts` 加一个层级列表才是封住
  这一点的办法，而不是 settings 本地的副本。
- 自定义布局预设是一次完整的顶层目录替换，不是逐条目的嵌套补丁。每个值都有界、无资源、唯一标识、只
  使用当前预设 schema，且不含任何工作区/标签页/会话/终端身份。一个畸形的已持久化成员在配置校验期间被
  隔离；一次携带任何畸形成员的线上变更被整体拒绝。不读取也不升级任何替代配置键或旧预设 schema。
- 删除或编辑一个自定义预设只改变共享定义。它不能改任何前端已实例化的框架或其本地默认选择。
- 分析偏好与首次运行对话框完成是受校验的布尔值。对话框可以只持久化默认开启的偏好；一次仅偏好的写入
  永不推断完成。无偏好的确认被拒绝，而改变一个已确认的决策会同时写入两个字段。投递门禁属于
  [[submodule-server-analytics]]；对话框生命周期属于 [[submodule-web-panels]]。
- `null` 清除可选的 `defaultModel`/`defaultEffort` 与 `reviewModel`/`reviewEffort` 覆盖；它是仅线上的
  哨兵，永不持久化。
- 已存的 `favoriteModels` / `recentModels` 若不是数组，在加载时回退到 `[]` 而不丢弃有效的兄弟字段；
  一次携带任何非模型成员的线上 `favoriteModels` 会拒绝整个更新。
