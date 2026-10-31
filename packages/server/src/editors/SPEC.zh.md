# editors —— 宿主编辑器/IDE 集成

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

检测宿主实际安装了哪些编辑器/IDE，在工作树上以分离方式启动其中一个，并在宿主的文件管理器中显示
某个工作树——即工作区行的“Open in”菜单。

## 边界（Boundary）

- **拥有：** `listAvailableEditors()` —— 通过 `Bun.which` 探测一份固定的候选清单（VS Code
  `code`、Emacs `emacs`、Vim `vim`，以及一份 JetBrains 的产品级启动器垫片优先级列表——`idea`、
  `webstorm`、`pycharm`、`goland`、`rider`、`clion`、`phpstorm`、`rubymine`——Toolbox 会为每个
  已安装的产品在 PATH 上创建其中一个；不存在单一的“jetbrains”二进制，因此找到的第一个就成为
  菜单里唯一的 `"jetbrains"` 条目，标签随它是哪个产品而定）。绝不使用固定的客户端清单：不在
  PATH 上的编辑器就不出现在结果里，所以菜单无法提供死条目。`openEditor(id, worktreePath)` 把 id
  解析回它的二进制（重新检查 PATH——对 `list` 与点击之间发生卸载的廉价保险）并以分离方式启动它
  （`@thinkrail/shared/spawn` 的 `spawnDetached`，与 `apps/cli` 的 `openBrowser` 一致）；当 id
  未知或已不再安装时抛错。`revealInFileManager(worktreePath)` 在那里打开操作系统自带的文件
  管理器：`open`（macOS）、`explorer`（Windows）、`xdg-open`（Linux 桌面默认）——发后不管，因此
  从不去检查 `explorer` 众所周知的“成功也非零”退出码。
- **Vim 是 `kind: "terminal"`，不在此处启动。** 它没有自己的 GUI 窗口——在没有附加 TTY 的情况下
  以分离方式启动它，只会打印“Warning: Output is not to a terminal”然后退出。**客户端**自行处理
  这种情况：打开/聚焦所点击工作区的内嵌终端并在其中运行 `vim .`（见
  `apps/web/src/panels/SPEC.md`），绝不为此调用本模块。
- **单独使用 `Bun.which(bin)` 读的是进程启动时快照的 PATH，而不是实时的 `process.env.PATH`**
  （`shared` 的 jbcentral 记录过同一个陷阱）——`resolveShellEnv()` 在宿主启动时重新解析 PATH，
  因此这里每次查找都显式传入 `{ PATH: process.env.PATH }`（`defaultWhich`），否则修好的 PATH
  永远不会被看到，一个已安装但不在快照 PATH 上的编辑器会被错误地读成不存在。
- **`WhichFn`/`SpawnFn` 是可注入的接缝**，而不是对 `Bun.which`/`Bun.spawn` 的 mock：
  `listAvailableEditors`/`openEditor` 接受可选的 `which`，`openEditor`/`revealInFileManager` 接受
  可选的 `spawn`（分别默认 `Bun.which` 与 `@thinkrail/shared/spawn` 的 `spawnDetached`）——这样
  单元测试可以伪造哪些二进制“存在”并断言将会启动什么，既不依赖跑测试的机器上装了什么，也不会
  真的启动进程。`revealInFileManager` 还接受可选的 `platform`（默认 `process.platform`），与
  `dialog` 的 `pickersFor` 使用同一种可测试参数模式，因此其 darwin/win32/linux 分支都能被直接
  断言，而不是只在跑测试的那个操作系统上被断言。
- **公开接口（barrel）：** `listAvailableEditors`、`openEditor`、`revealInFileManager`、`WhichFn`、
  `SpawnFn`。
- **允许的依赖：** `Bun.which`、`@thinkrail/shared/spawn`（`spawnDetached`）、
  `process.env`/`process.platform`、`contracts`（`EditorInfo` 线上类型）。
- **禁止：** `host`；同级特性（`workspaces` 在调用进来之前解析好 `worktreePath`——本模块自己
  从不查找工作区）。
