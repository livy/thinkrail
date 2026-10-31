# resourceHistory —— 聊天 Resources 背后的持久归档

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

保留某个聊天运行过的每一条后台命令与子代理——其摘要，以及命令被保留的日志尾部——使宿主重启后
Resources 仍能显示该聊天的完整历史。活跃的持有者
（[[module-pi-background-commands]]、[[module-pi-delegation]]）留在内存中且对此一无所知；`agent`
把自己的生命周期事件喂给它，并向它索取持有者不再保留的内容。它只记录持有者报告过的内容，
从不授予控制权：这里没有任何东西是活跃权威。

## 边界（Boundary）

- **拥有：** `<dataDir>/resource-history/<workspaceId>/<sessionId>/` —— `index.jsonl` 加上
  `logs/<commandId>.json`（`{ text, truncated }`）—— 以及已加载索引的有界内存缓存。
- **公开接口（barrel）：** `recordResource`、`finishedResources`、`pageResources`、
  `archivedCommandOutput`、`removeWorkspaceResourceHistory`、`resourceHistoryRoot`，以及
  `ResourceHistoryScope` / `ArchivedOutput` 类型。
- **允许的依赖：** `contracts`（资源记录词汇与排序）、`persistence`（`dataDir`）、
  `log`；Node 的 `fs`/`path`/`crypto`。
- **禁止：** `agent` 或任何活跃持有者、pi、会话或转录；自行判定存活性（调用方传入其持有者仍
  报告的那些 id）；停止或读取活跃工作。

## 格式与不变量（Format and invariants）

- 索引是追加式的 JSON 行 `{ kind, summary }`；每个 `kind:id` 以最后一行为准，完全相同的摘要不会
  被追加两次，因此一条记录对每个上报过的状态只花一行。无法解析或未通过形状校验的行被跳过——
  崩溃留下的截断末行就是其中之一，而下一次追加从新的一行开始，因此不会把它吞掉。
- 写入是同步的：优雅关闭的终止边沿恰好在进程退出前到达，而一次被 await 的写入可能丢失。已完成
  命令的日志在其索引行之前被原子写入，因此两者之间的崩溃留下的是无人认领的日志，而不是被认领却
  缺失的日志。写入失败会被记录日志，绝不抛入持有者的生命周期；追加失败之后，下一次追加从新的
  一行开始，以防失败留下了半行。
- **Interrupted 是推导出来的，读取方从不存储它：** 一条仍在进行中的归档记录，若没有任何活跃持有者
  报告其 id，就说明它被切断了——宿主崩溃或聊天被拆除——它按 `interruptedResource` 定义的线上契约
  形式读出（终止状态加 `interrupted: true`；没有完成时间，按其开始时间排序）。因此被崩溃中断的
  命令没有日志；被优雅拆除切断的命令已由 `agent` 连同日志一起记录，只有 `agent` 决定一次拆除何时
  切断了一次运行。
- 存储 id 在构建任何路径之前即被校验；不安全的工作区、会话或命令 id 被忽略，绝不会逃出目录。
- 按决策（用户已定，2026-10），历史按聊天不设上限：每条记录与每个日志尾部都随工作区一同存续。
  删除聊天会把它留在原处——从操作系统废纸篓恢复的聊天会拿回它的历史，正如其子代理转录一样——
  而归档工作区会移除该工作区的目录。
- 多个宿主可能共享一个数据目录，但不会共享一个活跃聊天（pi 的转录已作出此假设）；对同一聊天的
  并发写入者采用最后写入者胜出。
- 历史从引入该归档的宿主开始：此前的运行不会从转录重建，因为其状态、角色与时间无法被忠实还原
  （用户已定，2026-10）。

## 验证义务（Verification obligations）

Upsert/去重、跨全新模块实例（即重启）后仍存续、针对活跃集合的派生中断、日志往返与崩溃情形下
日志的缺失、截断/外来行的容忍以及追加失败后的恢复、基于游标的最新优先分页、不安全 id 的拒绝，
以及磁盘与缓存中的工作区移除。
