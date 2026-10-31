# locales —— 数据目录中的用户语言文件

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

读取、校验并监视 `<dataDir>/locales/` 中用户的本地语言文件（仅全局数据目录，绝不是工作区——
[[architecture]] 决策 23），并把它们报告给客户端。它不拥有任何翻译逻辑：校验与文件命名来自
[[module-locales]]。

## 边界（Boundary）

- **拥有：** 启动时创建该文件夹；扫描 `*.json`（非 JSON 与点号前缀的名字被忽略）；逐文件状态；
  文件夹轮询器；大小上限（每文件 2 MB、共 64 个文件——其余报告为错误）。
- **公开接口（barrel）：** `localesDir()`、`readLocalLocales(dir?)`（每次请求都读取——就几个小
  文件，没有需要失效的缓存）、`watchLocalLocales({ dir?, onChange, intervalMs? })` → 停止函数，
  以及上限 `MAX_LOCALE_FILE_BYTES` / `MAX_LOCALE_FILES`。
- **线上契约**（由 `LOCALES_PROTOCOL_VERSION` 把关）：`locales.list` → `{ dir, files }`，每个
  文件为 `{ tag, fileName, ok: true, strings } | { tag: string | null, fileName, ok: false, error,
  detail? }`，其中 `error` 是一个由客户端翻译的代码（`invalid-name`、`duplicate`、`too-large`、
  `too-many`、`unreadable`、`invalid-json`、`not-object`），`detail` 是未翻译的解析器/操作系统
  文本；`locales.changed` 是**失效提示，不是数据**——客户端重新读取 `locales.list`，与
  `workspace.fsChanged` 同一模式；`locales.reveal` → `Ack`（`host` 用 `editors` 的文件管理器
  显示功能组合出 `localesDir()`）。
- **允许的依赖：** `persistence`（`dataDir`）、`log`；外部的 `@thinkrail/locales` 根入口。
- **禁止：** `settings`（所选语言是 `AppConfig` 字段；哪个 tag 生效是客户端的决策）、`agent`、
  任何工作区路径。

## 必须做对（Get right）

- 坏文件从不阻塞其他文件——包括校验本身抛错的文件：非法 JSON（解析器给出时附行/列）、非对象根、
  非法文件名或超大文件都变成 `ok: false` 条目。
- 变更检测靠轮询，不使用 `fs.watch`：每秒（`intervalMs`）比较一次该文件夹语言文件的签名（名字、
  inode、大小、mtime、ctime——ctime 能捕捉 `chmod` 修复或恢复自身 mtime 的重写），不同就发出
  提示。一个 tick 内的突发（编辑器保存 = 写入 + 重命名）只算一次提示；空文件夹出现或消失则不算。
  Linux 上的 Bun 在整个进程内对每个路径共享一个 inotify watch，因此当文件夹或整个数据目录被删除
  后重建时，只要还有其他宿主 watcher 持着该路径，重新 watch 该路径就可能返回一个死的 watcher——
  CI 的 e2e 环境会在宿主启动后重建数据目录，其上挂有 jbcentral 的产物 watcher。每秒十几次 `stat`
  没有这种失效模式，也不需要重新附着逻辑。
- 宿主在启动时创建该文件夹一次，之后再也不创建——自己重建被删除的文件夹，会让之后执行的
  `mv backup locales` 把替换物嵌套进它里面。
