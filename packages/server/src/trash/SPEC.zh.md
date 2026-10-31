# trash —— 把路径移入操作系统废纸篓

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

把路径移入操作系统的废纸篓；当宿主以编译产物运行时，经由启动器分发的辅助可执行文件完成。这是宿主
中任何代码销毁用户文件的唯一途径。

## 边界（Boundary）

- **拥有：** `trashFile(path, implementation?)` —— 单个路径，禁用 globbing，**允许抛错**：无法
  恢复该文件的调用方必须得知移动失败了，而不是继承一次静默的永久删除。`setBundledTrashHelpers`
  是启动器接缝（分发的 `macos-trash` / `windows-trash.exe` 可执行文件，因为 `trash` 包自己的
  `new URL(…, import.meta.url)` 在编译后指向 `/$bunfs/` 内部）；源码模式与 Linux 委托给 `trash`
  本身，其 Linux 路径还需要静态安装的 `@stroncium/procfs` `processMountinfo` 解析器，而编译后的
  bundle 无法通过模板字面量的 `require` 触达它。`setTrashImplementationForTests` 为不得触碰真实
  回收站的测试替换整个实现。
- **公开接口（barrel）：** `trashFile`、`setBundledTrashHelpers`、`setTrashImplementationForTests`、
  `TrashImplementation`、`BundledTrashHelpers`。
- **允许的依赖：** `trash` + `@stroncium/procfs`（外部——其深层导入点名各自的 `.js` 文件，因为
  承载工作流测试装置的 Node ESM 拒绝目录与无扩展名的说明符）；Node 的 `child_process`/`util`。
- **禁止：** `host`；同级特性；任何永久 `unlink` 的回退。

## 消费者（Consumers）

两个特性都需要“可恢复地移除这个文件”，这就是为什么它是一个独立模块，而不是它们任一内部的辅助函数：

- `agent` —— 删除聊天会把 pi 的转录移入废纸篓，失败时回滚其墓碑（见 [[submodule-server-agent]]）。
- `changes` —— 对新增/未跟踪文件做整文件还原时，会把工作树文件移入废纸篓并记录位置，这样评审中
  最可能发生的破坏性误操作，即使宿主自己的撤销回执已不存在，也仍然可恢复（见
  [[submodule-server-changes]]）。
