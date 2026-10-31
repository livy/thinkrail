# pi-background-commands —— 显式的、会话拥有的后台命令

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

为 Pi 提供一项显式的后台命令能力：启动一个非交互命令而不占用 agent 轮次、查看它最近的输出、
停止那个确切的命令。普通 `bash` 不变。用户批准的范围是仅日志型命令；没有 PTY、终端标签页、
自动交接、stdin、重启/重跑、守护进程发现或任意进程管理。

## 边界（Boundary）

- **拥有：** 一个会话绑定的命令服务、它不透明的命令标识、准入、运行时快照、有界输出、
  取消与完成投递的簿记；`background_command` Pi 工具和一个默认的原版-Pi 扩展入口。
- **公开接口：** `BACKGROUND_COMMAND_COMPLETION_MESSAGE`, `BackgroundCommandInput`,
  `BackgroundCommandsExtensionOptions`, `CompletionDelivery`, `createBackgroundCommandsExtension`,
  `createBackgroundCommands`, `default`, `BackgroundCommandCompletion`,
  `BackgroundCommandCompletionBinding`, `BackgroundCommandContext`, `BackgroundCommandFlush`,
  `BackgroundCommandHandle`, `BackgroundCommandOutput`, `BackgroundCommandSnapshot`,
  `BackgroundCommandStart`, `BackgroundCommandStatus`, `BackgroundCommands`,
  `BackgroundCommandsBinding`, `BackgroundCommandsOptions`.
- **允许的依赖：** 公共包根 Pi SDK API 与 `typebox` 作为 peer；Node 标准库。
  执行委托给 Pi 导出的 `createLocalBashOperations`，而不是复制一份 spawn 运行器。
- **禁止：** ThinkRail 包、web/TUI 部件所有权、Pi 私有 import、委托所有权、工作区 PTY，
  以及通用资源 provider/插件机制。wire DTO 在 contracts 中镜像，
  绝不从本包导入或经本包重新导出。

## 会话绑定与执行（Session binding and execution）

根 `index.ts` 是唯一的 import 面；`src/` 是私有实现。服务暴露
start/list/find/change-subscription/dispose，外加完成绑定与重放；命令句柄暴露其快照、
输出、幂等停止与 `claimCompletion()`。控制器与子进程细节保持私有。

嵌入方把一个服务绑定到一个不可变的会话身份，并提供当前会话上下文与生效的 shell 设置。
独立扩展从 Pi 的公开 context/settings API 提供等价绑定，并尊重项目信任。cwd、shell 路径/命令前缀
以及 `PI_SESSION_*`/模型/推理环境在命令启动时遵循 Pi 当前会话的值；用户提供的环境覆盖
和宿主选定的 cwd 不是额外的工具输入。

`createBackgroundCommands({ sessionId, getContext, canDeliverCompletion? })` 捕获这个不可变身份；
同步的 `getContext()` 对每次被接受的启动都求值一次。它的投影携带 `cwd`、可选的 `sessionFile`、
`model`（provider/id）、`thinkingLevel`、`shellPath`、`commandPrefix` 以及
`exposeSessionEnvironment`（默认 true）。嵌入方投影其生效的内存设置；独立 Pi 在启动时
以最新工具上下文的项目信任决定读取 `SettingsManager.create`。Pi 不在 ExtensionContext 上
暴露其实时 SettingsManager。环境保留进程值、移除过期的 Pi 会话/模型标记，
并只注入本次启动的当前值。由于 Pi 的 `getShellEnv` 不是公开的，managed-bin PATH 一致性
改用公开的 `getAgentDir()` 加 `bin`，同时保留已有条目和环境里 PATH 键的大小写。
测试拿完整环境与公开的原生 bash 定义作比较；生产代码不使用该定义，
也不使用它无上限的临时输出累加器。可选的第二个工厂参数 `createOperations` 是供外部边界测试
使用的执行器构造接缝，默认使用 Pi 的公开本地执行器。

shell 命令本身仍在前台；后台化意味着工具不 await 它的结果。每个命令拥有一个独立于父轮次的
AbortController。输出回调、退出状态和进程树终止由 Pi 提供。超时是可选项，
含义是终止，绝不切换执行模式。对有意长期运行的命令不施加默认超时。

`background_command` 有四个动作：`start`（命令、可选的显示名与超时）、`list`、
`output`（命令 id）和 `stop`（命令 id）。start 用不透明 id 和当前快照确认受理。
id 字段在 output/stop 上是 `id`；动作专属的额外字段会被拒绝，包括 path、cwd、env、PID
与 stdin 控制。其他每个动作都限定在绑定的会话内；命令 id 不是 PID、路径或授权令牌。
重复 stop 是安全的，且 stop 永不创建命令。工具提示词指引要求受管命令避免 `&`/`nohup`
之类的 shell 脱离；这不是 shell 沙箱，也不承诺回收脱离监督的后代进程。

## 运行时与保留（Runtime and retention）

命令状态是 `running`、`stopping`、`completed`、`error` 和 `stopped`。只有执行器结算才能确立
终态：退出码为零是 completed，非零退出或无信号的空退出是 error，
而请求的取消在结算之后是 stopped。仅一个停止请求并不证明进程已退出。错误文本有界至 4 KiB，
启动失败仍可检视。超时是执行器错误（自然失败通知），不是用户停止请求。

一个会话最多准入八个活动命令，并保留其最新的二十条终态记录；超出的 start 显式失败而不是排队。
活动/停止中的工作从不被驱逐。显示名有界至 200 个字符，命令有界至 64 KiB。每条输出记录只保留
最新的 2,000 行或 50 KiB，先到者为准，采用 UTF-8 安全的截断并给出明确的截断指示。
读取是非消耗性的，输出快照**替换**更早的快照。行计数包含尾部换行之后的空末行。
增量 UTF-8 解码保留被执行器分块切开的字符；非法/不完整的终态字节使用正常的 UTF-8
替换解码。这是有意提供近期日志而非归档：没有无上限的临时文件，也没有第二份持久命令索引。
驱逐会释放输出，即使调用方仍持有旧句柄。

只要所属宿主/会话仍然存活，记录/输出就能在浏览器重载、客户端断开、聊天位置关闭
与父轮次 Stop 之后继续保持。它们在宿主重启时丢失；历史上的 Pi 确认从不重建活动记录。
Pi 会话记录中已完成命令的通知只保留有界诊断摘录，而不是完整日志。缺失/已驱逐的输出
报告为不可用。

实际的会话处置会关闭准入、禁用完成唤醒、在 await 任何一个之前先给每个活动命令发信号，
并参与嵌入方的有界关闭。工作区归档与聊天删除使用同一条生命周期路径。没有空闲计时器会杀掉
安静的工作。宿主的突然死亡与逃逸的守护进程后代无法被承诺干净终止或可恢复；
绝不从持久化的 PID 重建控制权威。

`list()` 返回快照副本；`find(id)` 返回本服务的句柄或 undefined。`stop()` 是同步的，
返回请求后的快照；它不 await 也不制造结算。`output` 是一个非消耗性的
`{ text, truncated }` 快照，驱逐后为 undefined。`onChange(listener)` 返回一个取消订阅函数，
并报告准入、停止中的变化与终态/驱逐变化，绝不报告字节更新。观察者错误无法中断命令清理。
`dispose({ timeoutMs? })` 立即关闭准入，并在并发调用方之间共享同一个 promise。
它的默认等待预算是 5 秒；宿主可以选择更短的非负预算。到期结束的是等待，不是执行权威：
未结算的命令保持 `stopping`，而任何最终的执行器结算仍然确立 `stopped`。
所有活动命令都在 await 任何一个之前先被发信号。

注入的服务属于嵌入方，而不是某个扩展实例：资源重载会重新绑定完成投递而不丢弃它的任务。
独立扩展在会话关闭时处置自己的服务。完成认领与保留的记录放在一起，因此重新绑定既不会
重复投递同一结果，也不会在重载间隙悄悄丢失已保留的完成。独立资源重载是一个 Pi 会话关闭事件，
会处置独立拥有的工作；只有注入的服务在该扩展生命周期之外拥有所有者。

`bindCompletion({ deliver, canDeliverCompletion? })` 替换当前的同步投递绑定，
立即重放待处理的已保留结果，并返回一个身份安全的解绑函数。`flushCompletions(flush?)`
针对当前绑定重试待处理结果而无需轮询；可选的 `{ atTurnEnd }` 标志会透传给绑定。
一次 flush 认领所有待处理结果并作为单批交给绑定 —— `deliver(batch, flush)` 返回它实际
交给 Pi 的那些完成，其余每一个重新变为待处理 —— 因此绑定对整个批次的顺序与唤醒选择
拥有一次性的决定权。服务绑定的与投递绑定的可选谓词都必须允许投递。记录是唯一的持有处：
一条记录在被实际发送之前一直是待处理，而 `claimCompletion()`（工具对已完成命令的 `output`
读取，绝不是 UI 读取）会清除它，因此 agent 已经读过的结果绝不会再被推送。宿主在临时删除墓碑
期间关闭其谓词，回滚之后再调用 `flushCompletions()`；确认删除则调用 `dispose()`。
资源重载间隙没有投递绑定，会在下一次绑定时重放。认领发生在同步发送之前，以防止可重入的重复；
未被接受或抛出的发送会让该记录保持待处理。发送成功意味着被 Pi 的公开即发即弃 API 接受，
而不是确认了最终的 provider 响应。驱逐也会移除待处理的认领：不存在无上限的第二通知队列。

## 完成与控制（Completion and controls）

已完成的命令在 agent 的下一步、只到达 agent 一次，投递规则与分离式 subagent 相同
（[[module-pi-subagents]] Result delivery）：扩展只允许服务在合格的 `turn_end`、
在 `agent_before_settle`、向空闲父级作为单批（最多启动一次运行）释放待处理记录，
或通过嵌入方在其 `turn_end` 监听器中调用 `flushCompletions({ atTurnEnd: true })` 释放；
它还会在 `agent_settled` 之后、以及压缩或树事件之后安排一次空闲 flush。每条消息都是
一条显示出来的 Pi 自定义消息，带有有界的结果摘录和一个 `delivery` 标记
（`wake` | `run` | `appended`）。导出的 `BACKGROUND_COMMAND_COMPLETION_MESSAGE`
是 `background-command-completion`；消息 details 省略完整命令，只保留 40 行/4 KiB 的
诊断输出摘录以及有界的快照元数据。工具指引告诉 agent 报告会自行到达，
因此它结束自己的轮次来等待，而不是轮询；已完成命令的 `output` 以其状态、退出码和错误开头，
因为认领了完成的读取不能隐藏推送本会说出的话。

被停止的命令从不唤醒父级；在父级运行被中止时已经完成的命令也一样 ——
按服务记忆，直到该服务的下一次用户提示词，因此一个服务多个会话的工厂无法清掉另一个会话的 Stop。
在空闲批次中，轮次由最后一个唤醒型完成触发，且它被最后发送：如果该发送失败，
该完成保持待处理，并在重试时仍携带该唤醒。显式的用户/工具 stop 记录取消而不启动空闲父级；
它不会中止已在运行的父级。实际的会话关闭会抑制向垂死会话的投递。

工具与宿主 UI 调用同样的句柄。服务变更订阅报告目录或生命周期变化，而不是输出的每个字节；
输出在被检视时另行读取。一旦 start 确认解析完成，原生 `tool_execution_update`
就不再是投递路径。

## 验证义务（Verification obligations）

要演练准入与驱逐、有界多字节输出、自然/非零/空退出、启动错误、逐作业停止隔离、
父级中止的独立性、重复停止、通知/重载竞态与有界处置，以及投递：单次运行内的下一步投递、
每个空闲批次一次运行、已读取的完成输出不再推送而宿主读取从不认领、以及父级停止后不再唤醒。
集成检查使用真实本地命令与 Pi 的公开执行器；由 provider 驱动的覆盖通过真实 Pi 会话验证
工具使用与完成。Windows CI 原生运行本包套件，包括关闭/进程树覆盖所用的后代标记；
可见控制台行为仍需要原生视觉探测，而不是只基于 Unix 的断言。本包套件使用 Bun 测试、
受控的执行器结算、真实本地 shell 命令，以及由 SDK 的 faux provider 驱动的 Pi AgentSession。
真实 provider 的宿主 E2E 仍是嵌入方的验收检查，与这些包内测试不同。
