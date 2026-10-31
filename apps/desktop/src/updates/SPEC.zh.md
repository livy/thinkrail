# 原生桌面更新

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

以一个控制器拥有原生桌面更新器：仅向前的资格判定、后台检查、显式整包下载、有界的检查重试/调度、
状态快照、显式的安装/重启交接，以及重启之后对那次交接如何结束的持久化报告。该控制器是 Electrobun
状态回调的唯一拥有者。它把回调投射为 available、transfer、preparation、ready 与 installation 阶段，
外加安装步骤与重启后的结果。Electrobun 对每个 SDK 操作单飞，而控制器只保留防止重叠所需的跨操作
时序。

## 边界（Boundary）

- **公开接口：** Electrobun 的更新控制器与退出协调器构造，以及供启动闪屏使用的
  `readPendingInstallVersion`，只通过 `index.ts` 导出。纯控制器、启用策略、日志文件存储、依赖形状
  与生命周期实现保持内部。
- **允许的依赖：** 来自 `@thinkrail/contracts` 的仅类型原生更新契约、SDK 适配器中的
  `electrobun/main`，以及纯控制器中注入的时钟/随机源/生命周期回调。
- **禁止：** web 或 server 的导入；由渲染器选择的 feed URL；宿主线上方法；版本的发布、签名或鉴权；
  草稿持久化；关闭流程的所有权；第二个 SDK 订阅。

只有打包的、受支持的 stable/canary 构建，且其发布元数据提供非空 HTTPS 更新器基址时，才会启用生产
适配器。打包元数据是 feed 的唯一权威；渲染器输入和重复的应用 feed 常量都被禁止。开发与普通产物
接缝保持禁用。测试向控制器注入更新器依赖，而不是改动生产 feed 选择。

手动操作立即确认，而工作通过状态修订继续。自动检查在桌面就绪后开始，以带抖动的六小时节奏重复，
并在有界范围内重试瞬时的检查失败；它们从不启动或重试下载。一个更新且尚未准备就绪的提议变为
`available`，**Download** 拥有传输与准备，且只有匹配的更新 `updateReady` 结果才变为 `ready`。
传输到 100% 不是就绪：Electrobun 的解压/准备阶段仍单独可见。Electrobun 2.0.5 没有提供取消进行中
的下载或丢弃已准备包的操作，因此在应用继续运行期间，传输与准备会继续进行到完成或出错；关闭
Settings 只是推迟安装。UI 不得在原生工作仍在继续时宣称已取消。错误会保留失败的阶段，使重试重复
原本意图的操作，而已准备的包能经受不相关的检查失败。**Install & Restart** 应用该包而不重新获取
feed；Electrobun 的原生辅助程序在交接期间负责最终载荷校验。

## 安装尝试与结果（Install attempt and result）

一次安装尝试横跨整个协调交接，而不只是第一次 SDK 调用。退出协调器的首次 `applyUpdate()` 总是被
否决，以便宿主可以关停。它解析为关闭是否已经开始，而这决定该尝试是否继续。只有进程退出或终态错误
才会结束一次仍在进行的尝试。

在尝试期间，`installStep` 跟随 SDK 状态：接受后为 `stopping`，`applying` 时为 `handoff`，
`launching-new-version` 时为 `restarting`。有界的 `log` 记录结构化事件，从不记录措辞：各步骤，以及
宿主关闭耗时。唯一的自由文本是原样携带 SDK 或辅助程序错误的 `diagnostic` 事件。任何 `error` 状态
——包括第二次交接的辅助程序启动失败——都会以 `install` 失败结束该尝试。从未启动关闭的交接会以 SDK
的错误失败，否则以“couldn't start the installer”失败。最后两次转换在退出之前的一个同步 SDK 块中
发生，因此它们的推送是尽力而为的。

Electrobun 的原生辅助程序没有 UI，只写入一个最终结果，SDK 在重启后对它协调一次并立即标记为已观测。
因此 ThinkRail 在 userData 中保留一个经校验、有界的 `update-journal.json`，含两种状态：
- **pending。** 起始/目标版本、开始时间与日志。在 `launching-new-version` 同步写入，即辅助程序武装
  之后的检查点。写入失败留在本地，处于 SDK 的清理路径之外。
- **resolved。** `lastInstall` 结果。它在每次启动时被重新水化，直到被忽略。

重启之后，协调结果成为 `lastInstall`：
- `complete` → `installed`。
- “failed during <phase>” → `failed`，并带阶段与原始原因。这里解析的是 SDK 2.0.5 的消息，并有原始
  回退。
- 处于 pending 的日志文件贡献起始版本与日志。从没有该日志文件的构建升级上来，会报告为 null 起始
  版本与空日志。
- 处于 pending 的日志文件若在 5 s 内没有协调结果：当运行中的版本就是其目标时解析为 `installed`，
  否则为 `unreported`。

接受新的尝试会清除先前的结果，`dismissInstallResult` 按请求清除它。只有当被恢复的构建带有该日志
文件时才能报告回滚；更旧的构建会静默重启。
