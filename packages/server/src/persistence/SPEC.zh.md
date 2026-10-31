# persistence —— JSON 应用状态

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

持久化的宿主状态——项目、工作区、跨前端的应用配置、终端目录、会话完成回执、安装身份，以及仅服务端
使用的获客状态——以数据目录下的 JSON 形式保存。当前工作台框架与工作区摆放位置属于前端本地状态，
没有宿主持久化。

## 边界（Boundary）

- **拥有：** `dataDir()`（开发/e2e 隔离用 `THINKRAIL_DATA_DIR`，否则 `~/.thinkrail`，始终解析为
  绝对路径——否则相对覆盖值会把受管工作树落到两处：`mkdirSync` 相对宿主 cwd 解析它，
  `git -C <project>` 相对项目解析它）；项目/工作区/配置的加载-保存操作，以及对 `DEFAULT_CONFIG`
  的逐字段校验，同时保留未知的顶层扩展字段；带版本的原子 `session-lifecycle.json`（最近观测到的
  已结算完成 + 显式取消的 run id）、`session-receipts.json`（首次安装基线的完成态 + 会话→已处理
  完成 id）与 `turns.json`（按工作区的 agent 运行变更集，由 `turns` 拥有）；仅服务端使用的
  `installation.json`（`{ id, appInstalled?: true }`）与仅营销活动使用的 `attribution.json`；
  `environment.json` 的结构化加载/保存（`loadHostEnvironment` / `saveHostEnvironment`，原子，
  模式 `0600`，语义由 [[submodule-server-environment]] 拥有）。不轮换的安装 UUID 绝不上线传输。
- **公开接口（barrel）：** `dataDir`，项目/工作区/配置与终端目录的加载-保存操作，会话回执的
  加载-保存操作，只返回 `{ id }` 的 `ensureInstallation()`，范围很窄的安装/浏览器认领操作，以及
  严格的获客读取/保存。初始 ID 创建通过原子且不覆盖的硬链接发布一个完整的同级临时文件，并重读
  并发的胜出者；已存在但格式错误的记录被保留。一个独占的安装认领标记在跨进程范围内选出一个打包
  宿主。完整的安装 JSON 被原子替换；被捕获的替换失败会移除标记与临时文件以便重试，而认领之后的
  崩溃可能消耗了事件却没有发出。遗留的已安装记录保持已认领状态。
- **浏览器尝试：** `attribution.json` 用 `wx` 独占创建其首个 `{ browserClaimAttempted: true }`
  标记。兑现（redeem）用严格的首次/末次触点字段（有界的 UTM 值、referrer 类别、时间戳与策略版本）
  原子替换它；网络或替换工作失败会让该尝试处于终态。保留期为 `last_touch` 之后 30 天：过期或无效
  的记录变为终态。持久化 schema 拒绝 claim/verifier/challenge/URL、journey/bridge ID、IP 与
  user agent。其他状态没有跨进程锁。
- **允许的依赖：** `contracts`（`Project`、`Workspace`、`AppConfig`、`HostEnvironment`、
  `LayoutPreset`、`DEFAULT_CONFIG`、`isTerminalWindowsShell`）；Node 的 `fs`/`os`/`path`。
- **禁止：** 导入同级特性或 `host`；推导会话状态；存储转录内容、缓存的运行/输入状态或本地激活；
  把生命周期元数据与已读回执合并；持久化当前框架/视图、选择/焦点或前端界面身份；读取备用配置键
  或旧 schema；或读取、重写、删除旧的宿主布局快照。

项目与工作区的加载以宿主解析后的拼写返回 `Project.path` / `Workspace.worktreePath`（架构决策
22）。以另一种拼写存储的记录——git 的 `C:/…`、未解析的路径段、旧宿主写入的文件——在读取时自愈，
并由该文件的下一次保存持久化；加载从不写入，在本宿主上非绝对的值保持存储原样。保存不做归一化：
生产者在自己的入口完成解析，因此创建调用返回的对象永远与其存储的内容一致。

会话元数据的写入是完整复制的临时文件替换。生命周期只持久化转录无法证明的事实——`agent_settled`
结果与显式 Stop——并以活跃的 run/completion id 为键；陈旧记录绝不作用于更新的轮次。加载会把每个
文件报告为已加载、缺失或被搁置。除非两个文件都加载成功，持有者会执行一次服务前的既有完成 id 基线，
以免历史凭空亮起；未解析的输入从不进入基线。不可读的文件（I/O、解析或 schema 失败）绝不阻塞宿主
启动：它被重命名为 `<file>.corrupt-<epoch-ms>` 以便诊断（尽力而为）并按缺失处理，于是重新基线会把
既有的完成标记为已读。用可能未被看见的关注度换取一个可启动的宿主是有意为之：曾有 v0.2.0-nightly.16
宿主因一个坏的 `session-receipts.json` 而拒绝启动，直到用户删除它。写入失败保持保守：它们可能让
关注度重新出现或被保留，但绝不把未看见的工作记录为已读。

分析配置独立地保留已保存的布尔偏好和一个有效的 `analyticsConsentConfirmed` 布尔值；缺失/格式错误
的值默认为 false。仅写偏好永不意味着已完成同意。初始偏好预置以及“偏好加确认”的写入由 Settings
拥有。

配置校验把封闭的主题模式加完整的不透明 system 配对、封闭的 composer 增长偏好、封闭的 Windows
终端 shell 偏好（无效/缺失 → `DEFAULT_CONFIG.terminalWindowsShell`）、聊天/文件行宽及其面板绑定
开关，以及 JetBrains 配额布尔值 + 整数 `1–3600` 秒节奏，归一到各自的默认值之上；它只接受当前
有界的 `customLayoutPresets` 目录作为同步布局数据。当前/默认预设 id、分组上限与聊天消息顺序不是
配置字段；退役的配置形态被剥离，而不是升级或作为扩展保留。历史 `layouts/` 文件保持不动且不生效。
