# environment —— 已存的宿主环境（pi 配置文件夹 + 变量）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

用户已存的宿主环境：一个可选的 pi 配置文件夹（`PI_CODING_AGENT_DIR`）以及一个有序的 `NAME=value`
变量列表，在 Settings → Environment 中编辑，并在启动时每个宿主进程向 `process.env` 应用一次。它存在的原因是
一次 Dock/Finder 启动无法传递 `PI_*`（登录 shell 导入从不导入它们），而 shell 别名从不被导出。于是进程内
的一切 —— pi、扩展、bash 工具、终端、stdio MCP 服务器 —— 都看到一份一致的环境，因此在 ThinkRail 终端里
跑的 `pi` 使用同一个配置文件夹。

## 边界（Boundary）

- **拥有：** 该记录的结构解析 + 语义校验、`applyHostEnvironment()`（只由 `host` 的 `bootHost` 调用）、
  已应用快照与启动键快照，以及 Settings 分区所读取的状态。
- **公开接口（barrel）：** `applyHostEnvironment`、`hostEnvironmentStatus`、`updateHostEnvironment`。
- **允许的依赖：** `persistence`（`loadHostEnvironment` / `saveHostEnvironment`）、`log`、`contracts`
  （类型）、`@thinkrail/shared/shellEnv`（`launchEnvKeys`）、`@earendil-works/pi-coding-agent`
  （`getAgentDir`，用于在用的文件夹）。
- **禁止：** `settings` / `AppConfig`（值可能是秘密，而 `AppConfig` 会被广播到每个前端并在
  `server.welcome` 中播种）；在任何通道上发布；记录或报告变量名或值。

## 必须做对（Get right）

- **应用点。** `bootHost` 在 `await resolveShellEnv()` 之后、在 `initializeJbcentralRuntime()` /
  `createServer()` 之前调用 `applyHostEnvironment()` —— 即在第一个 `getAgentDir()` 读取者之前。稍后再
  应用会把进程劈开：模型 runtime、MCP 用户配置 watcher、history 根与 Central 持有它们最初看到的 agent
  目录。
- **优先级：启动环境 > 已存记录 > 登录 shell 导入 / locale 修补。** 在 shell 修补之前就存在的名字
  （`launchEnvKeys()`）永不被覆盖，因此 `FOO=1 thinkrail` 以及经 `PI_CODING_AGENT_DIR` 的 e2e/smoke
  隔离继续有效；状态把这些名字列为 `launchOverridden`。键集合在应用时被复制，因此一个回退快照（未运行
  shell 修补）永不包含该记录自己的写入。
- **变更 = 重启。** `environment.update` 只持久化；`restartRequired` 就是「已存 ≠ 已应用」。在线重新
  应用因上述劈开进程的理由被否决。一个「Restart now」动作被推迟：不存在通用的宿主重启。
- **校验**（更新时全有或全无，逐字段/逐行报告 issue）：agent 目录在 `~` 展开之后必须是绝对路径且是
  已存在的目录；名字匹配 `[A-Za-z_][A-Za-z0-9_]*`，不得重复；保留名（大小写不敏感）是 `PATH`
  （shellEnv 拥有它）、`HOME` / `USERPROFILE`（Bun 缓存 `homedir()`）、`PI_CODING_AGENT_DIR`（它自己
  的字段）与 `THINKRAIL_*`（数据目录、端口与宿主在该记录之前或之外被读取）。空白行被丢弃；一个结构上
  畸形的记录是一次抛出的请求错误。
- **名字遵循平台的字母大小写规则。** 在 Windows 上环境变量名是大小写不敏感的（Bun 的 `process.env`
  也是），因此那里的启动键成员关系、重复检测与覆盖报告比较的是大写后的键，其他地方比较精确名字；展示
  保留用户的书写。`applyHostEnvironment` 接受一个可注入的 `platform`，因此两个分支在任何 OS 上都能被
  测试钉住。
- **启动容忍漂移。** 磁盘上无效的条目（一个被删除的文件夹、一个手工编辑出的保留名）被跳过，并附一条
  只点名字段与代码的 `log.warn`；其余部分照常应用。被跳过的 agent 目录回退到 pi 的默认值，状态中的
  `effectiveAgentDir` 使这一点可见。
- **存储：** `<dataDir>/environment.json`，以 `0600` 模式原子替换；只由 `environment.get` 获取，永不
  广播。
- **每宿主一次设置，而不是每项目一次。** 逐项目方案被否决：pi 在进程内，`process.env` 是进程全局的，
  因此 provider key 查找与扩展代码看不到逐项目的值，而第三方扩展无论有无逐会话 `agentDir` 都读取全局的
  `getAgentDir()`。仅当 pi 获得一个带作用域的 agent 目录时才重新考虑。
- **用户已接受的后果：** pi 把转录保存在 `<agentDir>/sessions` 下，因此切换该文件夹会隐藏另一个文件夹下
  做过的聊天，直到切换回来，与 pi 的 CLI 中一样。
