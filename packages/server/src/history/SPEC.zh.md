# history —— 聊天历史搜索索引

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）
`history.search` 后端：对 pi 的会话 JSONL 文件建立**惰性内存索引**（prompt 召回 + 全对话匹配）。它
**自己**枚举并读取会话文件（异步 `readdir`/`stat`/`readFile`，复刻下文钉住的 pi 发现布局）—— 它在任何
搜索/刷新路径上**绝不调用 `SessionManager.listAll()`**：`listAll` 仅仅为了构建它的 `SessionInfo`
（`messageCount`/`allMessagesText`，本模块从不使用）就会把*每个会话文件整体*读取并 JSON 解析，因此
经由它的索引刷新即使什么都没变也要付出整个语料库的代价，而且是在每个在线会话共享的事件循环上。
**从不写入**会话文件。由于发现过程会遍历磁盘上每个 pi 会话，宿主处理器的 `all` 作用域
（`host/historyScope.ts` 的 `filter = () => true`）刻意也暴露在任何已注册 ThinkRail 工作区之外的
pi-CLI 会话 —— 比 `session.getMessages` 自身为单个会话暴露的略多，但与一个以所有者为作用域的宿主
一致（没有多租户隔离需要保留）。

## 设计（Design）
- `extract.ts` —— 纯的 JSONL→`ExtractedSession | null`（`extractSession`）：会话的身份（来自头部的
  `id`/`cwd`，镜像 pi 自己的拒绝规则 —— 第一个可解析条目必须是带字符串 `id` 的 `type: "session"`
  头部，否则 `null`）、它的 `title`（**最新** `session_info` 条目的名字，包括显式清除 —— 与 pi 的
  `buildSessionInfo` 同样的 latest-wins 规则），以及它的 `HistoryEntry[]` —— 全部来自**一次解析**，
  因此索引永不需要第二个来源（`listAll`）来获取元数据。pi 会话文件是被压缩重写的**树**（被弃置的
  分支），因此它按 pi 的方式在客户端渲染之前先解析文件 —— `parseSessionEntries` →
  `migrateSessionEntries` → `buildSessionContext`（跟随当前叶子、应用最新一次压缩、丢弃已摘要/已弃置
  的条目）—— 然后索引解析后的消息，并经 contracts 的 **`isTranscriptMessageRole`** 过滤，那正是
  `getSessionMessages` 所依赖的守卫（一份策略，不是它的本地副本）。因此 `messageIndex` 与客户端的
  `turnIdByMessageIndex` 精确匹配（没有原始文件顺序的漂移），而被弃置/被摘要的文本永不成为命中 ——
  压缩摘要会被发送（它渲染客户端的压缩标记），因此它占用一个索引槽却不可搜索。内部的
  `TODO_NUDGE_PREFIX` 控制消息（在 hydrate 时对转录隐藏）在其索引槽被占用之后跳过，因此对齐保持；
  一次**被取代的自动重试尝试**（来自 contracts 的共享 `isRetriedAttempt` —— 一个出错的 assistant
  紧跟另一个 assistant，即 pi 的 `_prepareRetry` 产生的相邻形态，hydration 把它渲染为无轮次/null
  锚点）以同样方式跳过，因此一个失败残片的文本永不成为只能解析到「找不到该消息」的命中。条目文本是
  **完整的，从不截断** —— 命中的 `text` 就是召回所插入的内容，也是浮层预览作为完整 prompt 呈现的
  内容，因此加一个上限会悄悄破坏长粘贴日志 prompt 的召回，并让截断点之后的词不可搜索（内存先例是
  pi 本身：`SessionInfo.allMessagesText` 把每个会话的完整文本都放在内存里）。工具结果/思考不被索引。
- `historyIndex.ts` —— `HistoryIndex`：首次搜索时冷构建（逐文件异步 IO 让出事件循环；在预算内阻塞
  搜索，然后带 `indexing: true` 返回部分结果）。发现是它自己的异步枚举（钉住的布局见下文「pi 文件
  格式」）：自定义 `sessionDir` 是扁平的、非递归的 `.jsonl` 文件目录；默认根是
  `<agentDir>/sessions` —— agent 目录**每次刷新**都经由 pi 导出的 `getAgentDir()` 解析，它实时读取
  `PI_CODING_AGENT_DIR` —— 其下有一层逐 cwd 目录，包括 Pi 视为候选目录的顶层符号链接。损坏的或
  非目录的符号链接降级为与 Pi 相同的「不可读目录」空结果。新鲜度 = `(mtime, size)` 重新校验，节流到
  ~2 s（pi 把在线消息追加到文件，因此文件**就是**实时 feed —— 不需要 agent 模块钩子；size 与 mtime
  一起比较，因此一次落在同一粗糙 mtime tick 内的追加仍会重载）；只有**新的或已变更**的文件才会被
  读取（一次 `readFile`）并重新解析 —— 未变更的语料库只花一次 `readdir` 遍历 + 每文件一次 `stat`，
  无读取、无解析。温重新校验在**后台**运行 —— 一次批量变更（重启后的首次刷新、一次 git checkout）
  仍可能意味着大量解析，因此搜索永不阻塞在它上面；结果最多陈旧一个周期，而后台刷新会吞掉自己的
  错误（一个未处理的 rejection 可能让进程内宿主崩溃）。只要有构建在进行（冷构建**或**一次后台重新
  校验），就报告 `indexing`，因此客户端的重试循环会轮询到刚写入的会话落地为止 —— 不阻塞搜索的
  read-your-writes。匹配：大小写不敏感的子串，且对空白分词的所有词都匹配（查询长度 + 结果 `limit`
  在处理器与 `search()` 两处都被夹到协议上限 `MAX_HISTORY_QUERY_LENGTH`/`MAX_HISTORY_LIMIT`）；
  严格的新近顺序；prompt 按规范化文本去重并保留最新；有上限与真实总数。消息区（以及
  `messageTotal`）被过滤为仅 `role === "assistant"` —— 一个 user 角色的命中总是其自身 prompt 条目的
  文本重复（在这次抽取中 user 文本**就是**一个 prompt），因此它不会增加任何文本，只增加一个位置；
  那个位置改为经下面两个字段挂到 prompt 命中上。
- 跳转锚点容忍漂移：命中携带 `anchorText`（消息文本前缀）供客户端校验。每个 prompt 命中现在也携带
  它保留的最新一次出现的 `messageIndex`/`anchorText` —— `MessageHit` 一直就有的同样两个字段 ——
  使 prompt 行本身可跳转。两个字段在 `PromptHit` 上都是可选的（仅当宿主早于此特性时缺失）。
- `testFixtures.ts` —— 仅测试用的会话文件构造器（由 A5 钉住）：`writeFixtureSession` 写出一个最小但
  真实、pi 形状的 JSONL，每个会话一个扁平文件，供 `historyIndex` 测试、e2e 夹具播种器及其自身的格式
  钉住测试（`testFixtures.test.ts`）驱动；`defaultSessionDirFor` 复刻 pi 私有的默认布局目录编码
  （不可导入 —— 见下文「pi 文件格式」），好让夹具能落在真实的无参 `listAll()` 实际会看的位置。

## 磁盘上的 JSONL 结构（观察自 pi 会话文件）
- **`message` 条目：** `{ type: "message", ..., message: { role: "system"|"user"|"assistant"|"toolResult", content: string|array, timestamp: ms-number } }`。
  System 消息携带 prompt 的 `sections` 和/或 `toolsAdded`/`toolsRemoved`；第一次请求持久化完整 system
  消息，之后的 prompt/工具变更持久化补丁。`isTranscriptMessageRole` 排除 system，因此它不渲染转录行
  也不占 `messageIndex` 槽。一个其工具有嵌套调用（`ctx.executeTool`）的 `toolResult` 携带一个有界的
  `nestedCalls` 记录；它本身不是一行转录。
- **`usage` 条目：** `{ type: "usage", ..., kind, provider, model, usage, note? }`；它们对对话树与
  上下文隐藏，但 pi 把它们的 token/成本数据计入会话总计。压缩条目可能随其摘要携带一份
  `systemMessage` 快照。
- **`custom_message` 条目：** `{ type: "custom_message", customType: string, content: string|array, timestamp: ISO-string, display: boolean, ... }`。
  它们没有 `message` 包装，成为角色为 `custom` 的上下文消息。

## pi 文件格式（`@earendil-works/pi-coding-agent`，目录钉住的版本）
已对安装的 `dist/core/session-manager.{js,d.ts}` 验证。pi 升版时重新验证这些事实；`testFixtures.ts`
与 `historyIndex.ts` 都依赖该发现布局。
- **头部：** pi 写 `{ type: "session", version: 3, id, timestamp, cwd, parentSession? }`；
  `CURRENT_SESSION_VERSION = 3`。`cwd` 是交给 `SessionManager` 的 cwd 的 `path.resolve`（在
  `testFixtures.test.ts` 中钉住）；已存目录靠它以 `===` 比较（architecture Decision 22）。读取方要求
  第一个*可解析*条目是带字符串 `id` 的会话头部（不一定是物理第 1 行）。`cwd` 从该头部恢复；缺失/
  无效的头部会拒绝该文件。
- **命名与自定义目录：** 发现只考虑名字以 `.jsonl` 结尾的直接子项；pi 写
  `<ISO timestamp with ':' and '.' replaced by '-'>_<sessionId>.jsonl`，但文件名除此之外没有含义。
  `list(cwd, sessionDir)` 与 `listAll(sessionDir)` 使用扁平的、非递归的自定义目录；自定义列表的过滤
  仍把头部 `cwd` 与 `cwd` 比较。放在子目录里是不可见的。
- **默认布局：** 无参 `listAll()` 扫描 `<agentDir>/sessions/` 一层深，考虑子目录与符号链接，然后是
  它们的直接 `.jsonl` 子项。不可读或符号链接目标损坏的项不贡献文件。每个 cwd 目录名为
  `--<resolved cwd with a leading slash removed and slash/backslash/colon replaced by `-`>--`；
  `getDefaultSessionDirPath` 是私有的，而 `getDefaultSessionDir` 未从包根导出。`defaultSessionDirFor`
  复刻它。
- **文件何时出现：** 新会话的文件在它持有 user 或 assistant 消息时才首次写出（仅设置的会话不留文件），
  因此一个首轮从未完成的 prompt 仍在磁盘上。
- **环境：** `getAgentDir()` 在被调用时读取 `PI_CODING_AGENT_DIR`，因此在一次 SessionManager 发现调用
  之前改变该变量会改变 pi 使用的根。

## 边界（Boundary）
- **公开接口（`index.ts`）：** `HistoryIndex`、`getHistoryIndex()`、`matchesTerms`、`makeSnippet`、
  `clampLimit`、`extractSession`、类型。**无测试辅助函数** —— `writeFixtureSession`/
  `defaultSessionDirFor` 是会写盘的仅测试构造器，不得进入 runtime 模块图；它们只能通过 server 包
  专用的 **`@thinkrail/server/history-test-fixtures`** 子路径导出（package.json）到达，那是被认可的
  测试边界 —— 与 `@thinkrail/server/agent` 同一模式。包内测试相对导入它们（`./testFixtures`）；e2e
  播种器导入该子路径。
- **允许的依赖：** `@earendil-works/pi-coding-agent`（`getAgentDir` —— 默认 sessions 根 —— 加上
  `extract.ts` 所用的导出会话树辅助函数
  `parseSessionEntries`/`migrateSessionEntries`/`buildSessionContext`；生产代码中**不用
  `SessionManager`** —— 见职责。测试可以用 `SessionManager` 对着真实实现*钉住*布局/格式事实）、
  `@thinkrail/contracts`、`node:fs`、`node:fs/promises`、`node:path`。
- **禁止：** 导入 `agent`/`workspaces`/`projects`（作用域映射由宿主处理器经传入 `search()` 的
  `filter`/`labels` 回调注入）；向磁盘写任何东西（`writeFixtureSession` 仅测试用，永不被生产代码路径
  调用）。
