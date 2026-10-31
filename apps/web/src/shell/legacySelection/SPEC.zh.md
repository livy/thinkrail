# shell/legacySelection —— 临时的选择兼容适配器

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

容纳那条临时桥接：从 workbench 局部的 attention 通往迁移期由功能渲染缓存所使用的活动编辑器、预览和
终端镜像。

## 边界（Boundary）

- **拥有：** 把选中的中心 placement 解析为匹配的编辑器缓存或终端目录 key；原子地清除不兼容的遗留
  镜像；当某个缓存身份在一个稳定 placement 之下被修复时重新运行；以及当活动标签变为一个被评审的
  文件时显示 Review 工具 —— 在 `doubleClickPendingMs()` 表明某个浏览行的双击仍可能到达时延后，这样
  该行能在 Review 取代它之前保留自己的预览。当 Review 已经显示（由 workbench 提供的 reader）时，
  没有东西能取代该行，因此显示立即执行：迟到的显示其焦点请求会关闭用户在此期间打开的任何东西。
  同理，若自调度以来正在显示的辅助工具（reader 的 `showingKey`）发生变化，延后的显示会放弃：
  用户在该窗口内选中的工具，例如 Specs，会保留。在触发前被取消的延后显示即被遗忘，因此用相同选择
  重跑会重新调度它。
- **公开接口（`index.ts`）：** 挂载的兼容 hook 以及 workbench 回调所使用的命令式 attention 变更
  适配器。
- **外部依赖：** web 局部的中心资源类型；store 选择器、唯一的遗留镜像动作，以及显示工具的 layout
  intent；shell 中立的 `lib`（`doubleClickPendingMs`）；React。
- **禁止：** 改变 frame 或工作区 placement、递增 navigation、transport 调用、panel／session／终端
  生命周期、server/shared/pi 导入，或成为第二个 attention 权威。

所有对 `activeTabByWorkspace`／`activeTerminalByWorkspace` 兼容状态的 shell 写入都经过本模块。
移除这些镜像仍是一次独立的迁移，不属于本次重构。
