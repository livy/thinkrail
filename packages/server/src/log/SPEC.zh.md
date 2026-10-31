# log —— 分级诊断到 stderr + 轮转的支持文件

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

宿主的诊断记录：每个模块都通过 `logger(scope)` 而不是 `console.*` 记录日志。运维人员看到可读的
stderr，同时同样的应用记录以结构化 JSONL 落到 `<dataDir>/logs/` 之下，用户可以把它们发给 ThinkRail，
用于由 agent 主导的复现与调查。只有经由本模块显式 logger 的调用才进入持久文件；pi / 扩展任意的
`console.*` 参数只留在终端，因为它们可能包含 prompt、工具数据或凭据。

## 目的地与轮转（Destinations & rotation）

- Pino 负责记录序列化。每行 JSONL 都是一条独立的、带 schema 版本的记录，包含 ISO 时间戳、
  数字 + 文本严重级别、scope、message，以及提供时的结构化错误。追加式 JSONL 可流式处理，且被
  截断的最后一次写入不会使更早的记录失效。
- `pino-pretty` 只把应用日志渲染到 stderr。持久文件保持 Pino 的 NDJSON，而不经过人类可读文本
  格式化器。
- `pino-roll` 负责完整的文件生命周期：按天加 10 MB 大小轮转、生成的序号、大小核算、重启续写与
  清理。基名是 `thinkrail.jsonl`；生成的名字遵循所用传输层自己的约定。保留十四个轮转文件加上
  活跃文件，这把常规诊断存储限制在 150 MB 左右。适配器在打开之后调用一次所固定传输层自己的清理
  例程，因为 `pino-roll@4.0.0` 否则只在进程内发生轮转后才运行它；这在不为它的保留算法重复实现
  的前提下，为频繁重启、低量的宿主维持该上限。轮转遵循宿主本地的日历边界；记录时间戳仍为 UTC
  ISO 字符串。
- 轮转目的地同步写入，使已接受的记录不会滞留在工作线程队列中。`crash.log` 仍是由
  `host/crashLog.ts` 拥有的独立同步纯文本致命报告，不在 pino-roll 的文件名集合内。

## 级别与初始化（Levels & initialization）

- `debug | info | warn | error`；默认阈值 **info**。该阈值同时把关 stderr 与文件。解析顺序：
  显式的 `initLogging({ level })`（CLI 的 `--verbose` → debug）> `THINKRAIL_LOG_LEVEL`（本模块是
  该变量的唯一读取者；无效值会告警但不回显环境值，并回退到 info）> info。
- 在 `initLogging` 之前，`logger(...)` 调用只回显到 stderr，不写文件，因此单元测试与库嵌入方
  永远不会隐式长出支持文件。
- `initLogging` 异步打开 pino-roll，并首先由 `host` 的 `bootHost` await（绝不由 `createServer`
  await；进程级日志属于进程所有者）。重复调用会重新解析级别，并发调用 await 同一个一次性打开
  动作。直接的进程内流有意避开 `pino.transport()` 的工作线程及其额外文件的二进制打包契约。

## 控制台边界（Console boundary）

`console.debug/log/info/warn/error` 绝不被补丁或复制进持久文件。pi、扩展与第三方包可能把敏感项目
数据放进任意 console 参数；没有哪个结构化脱敏器能让它们的自由文本变安全。属于支持文件的宿主诊断
必须改为经过一次评审过的 `logger(scope)` 调用，消息有意保持有界。

## 失败与隐私边界（Failure & privacy boundary）

- 日志绝不能把宿主搞垮。初始化与日志调用在失败时降级为直接写 stderr；每个目的地的错误通道都被
  消费掉，不再绕回 logger。
- 支持文件绝不有意包含凭据、授权头、cookie、环境变量值、prompt/消息内容、文件内容、工具参数/
  结果或 WS/协议载荷。已知的结构化密钥字段通过 Pino 脱敏移除，作为纵深防御；调用点仍必须遵守
  封闭诊断规则，因为嵌入自由文本消息中的密钥无法被结构化脱敏。持久消息只可插值封闭的方法名、
  计数、生成的 id 以及等价的有界元数据——绝不可用原始错误文本或请求派生的路径。显式批准了结构化
  错误的调用点只得到 type/message/stack，绝不到任意可枚举字段。

## 已记录错误观察者（Logged-error observer）

`observeLoggedErrors(observer | null)` 是唯一的接缝：携带错误对象的 `warn`/`error` 调用也经由它
交给一个观察者（scope + error），与级别阈值无关；观察者的失败被吞掉。宿主在服务器启动时把它接到
分析错误报告上，并在停止时清除，因此 `log` 不带任何分析相关的边。

## 边界（Boundary）

- **拥有：** 进程级 Pino 适配器、级别/环境解析、带作用域的外观层、JSONL schema 与隐私策略、
  pino-roll 配置与启动时的清理调用、美化 stderr，以及独立崩溃报告使用的共享 `describeError`。
- **公开接口（barrel）：** `logger`、`Logger`、`LogLevel`、`initLogging`、`InitLoggingOptions`、
  `setLogLevel`、`logsDir`、`describeError`、`observeLoggedErrors`。
- **允许的依赖：** `persistence`（`dataDir`）；`pino`、`pino-pretty`、`pino-roll`；Node 的
  path/streams。
- **禁止：** 导入任何其他同级模块或 `host`；被 `persistence` 导入（它是位于其下的唯一模块——
  `persistence → log` 这条边会形成环）。
