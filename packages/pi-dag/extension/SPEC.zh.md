# 可移植的 pi DAG 适配器

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 边界（Boundary）

拥有 pi 的工具/命令注册、受信任的会话来源、瞬时的上下文/历史投影、被动通知投递与独立组合。
公开 barrel 暴露 `createDagExtension`、`DagExtensionOptions` 与 `default`（零配置的扩展工厂）。
外部依赖是公开的 pi API、typebox、pi-delegation 的 barrel 与 Node API。同级边在父规格中声明。
禁止触达：运行时/域/delegation 内部、ThinkRail/UI/server 包、pi 私有字段、复制凭据与独立的 worker 循环。

## 适配器（Adapter）

被注入的服务始终归宿主所有，且永不在此关闭。五个控制器工具使用域 schema，并直接映射到 execute
或四个具名读取。调用方的 command id 与版本不变。结果在 details 与 JSON 文本中保留整个 DagResult；
已准入的 DTO 已经捕获了批量正文，因此任何展示层的上限都不得隐藏拓扑或门禁身份。公开的
`tool_result` 覆盖会标记结构化失败，无需逐调用暂存。

每个 session_start 轮换一个生命周期与一个仅通知的绑定。命令绑定实际的 session id，把调用取消与
生命周期撤销合并，并在 finally 中撤销其短期绑定。执行投影是同步且当前的，永不被服务保留为回调；
undefined 被透传，以便后端回执重放先于上下文要求。读取不投影执行上下文。控制器历史使用真实的管理器
与实际的 execute toolCallId 的工具调用前切点，而不是 commandId。

`/dag` 解析并校验 RequestSchema，然后要求 hasUI 与对完整未改动请求的显式确认。支持 TUI 与 RPC 确认；
非交互模式什么也不做。人类来源是 `operatorId: pi:<actual session id>`，配以那个 conversation id。
确认之后，在投影上下文与当前历史的入口处切点之前，重新检查所捕获的生命周期。没有工具接受 actor
或提升为人类。操作者的结果是自定义的被动消息。

通知汇在提交一条带 triggerTurn:false 的自定义 `dag-notice` 消息之前，检查所捕获的身份、当前生命周期
与 isIdle。agent_settled 唤醒就绪监听器；agent_end 不会。撤销会移除监听器与所有被保留的上下文闭包。
后端绑定通过只读扫描恢复通知；附加不会声明执行所有权。运行时的附件身份把一个控制器会话与一个人类的
显式匹配 conversation id 映射到一个 conversation key，同时保持它们的调用方对象 —— 因而命令权限 ——
彼此区分。没有 conversation 的人类与所有者保留各自独立的附件身份；
不存在第二个观察者或隐藏的附加路径。

## 独立所有者（Standalone owners）

一个惰性的、带版本的 globalThis Symbol.for 注册表 —— 而不是模块缓存持久化 —— 拥有各作用域的服务。
键包含规范的 agent 根与 cwd；作用域是 cwd 的 SHA-256。存储位于规范 agent 根的 dags 与 delegation
目录下。规范化会为最初不存在的目录解析最近的既有祖先，并传播除 ENOENT 之外的错误。
悬空符号链接失败，而不是创建一个非规范的 owner 键。获取与插入是同步的，因此关闭无法漏掉一个在途的
owner 获取。一旦关闭开始，获取就失败，直到所有关闭完成且注册表被移除。
关闭失败会保留被阻塞的注册表并向上传播；不兼容的注册表版本/形态响亮失败。

New/resume/fork 只撤销 conversation 绑定。Quit/reload 在清理注册表之前 await 每个被保留所有者的关闭，
包括未显示的作用域。替换代码获取全新的服务，并看到已暂停的持久资源。注册表不保留任何 pi/上下文闭包。
Delegation 拥有 provider 重放与资源执行；借来的 provider 回调仍是提供方宿主的生命周期契约，
而不是适配器的一条笼统限制。

## 验证（Verification）

公开适配器测试使用真实的 pi 会话与隔离的本地 faux 模型/凭据存储。类型化的 service-client 替身把转发、
来源、确认与通知投递同后端策略隔离开。真实服务用例固定调用历史、完整读取与独立生命周期。
测试覆盖直接映射、身份/历史切点、不替换执行上下文的重放、结构化错误、被动的空闲通知、
切换存活、进程注册表的模块替换，以及全作用域的静默 reload/quit。读取不得调用 provider。
没有付费/网络 provider、宿主打包或重复的后端策略测试。
