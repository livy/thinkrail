# themes —— 数据目录中的用户主题文件

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

读取并监视 `<dataDir>/themes/*.theme.json` 中用户的主题文件（仅全局数据目录，绝不是工作区，因此
克隆下来的仓库无法改变应用外观），并把它们报告给客户端。它懂文件容器，不懂主题：清单 schema、
id 规则与目录属于 [[submodule-web-themes]]，由它校验收到的每个文件。

## 边界（Boundary）

- **拥有：** 启动时创建该文件夹；扫描 `*.theme.json`（点号前缀的名字被忽略）；逐文件读取状态；
  大小上限（每文件 256 KB、共 64 个文件——其余报告 `too-many`）；文件夹轮询器。
- **公开接口（barrel）：** `themesDir()`、`readUserThemes(dir?)`（每次请求都读取——就几个小文件，
  没有缓存）、`watchUserThemes({ dir?, onChange, intervalMs? })` → 停止函数，以及上限
  `MAX_THEME_FILE_BYTES` / `MAX_THEME_FILES` / `MAX_THEME_NESTING`。
- **线上契约**（由 `USER_THEMES_PROTOCOL_VERSION` 把关）：`themes.list` → `{ dir, files }`，每个
  文件为 `{ fileName, ok: true, manifest } | { fileName, ok: false, error, detail? }`，`error` 为
  `too-large`、`too-many`、`unreadable`、`invalid-json` 之一，`manifest` 是解析后未校验的 JSON；
  `themes.changed` 是失效提示，不是数据；`themes.reveal` → `Ack`（`host` 用 `editors` 的文件
  管理器显示功能组合出 `themesDir()`）。
- **允许的依赖：** `persistence`（`dataDir`）、`log`。
- **禁止：** 主题 schema 或任何颜色知识、`settings`（哪个主题处于激活状态是由客户端解析的
  `AppConfig` 字段）、`agent`、任何工作区路径。

## 必须做对（Get right）

- 坏文件从不阻塞其他文件——包括病态嵌套的文件：嵌套深度超过 `MAX_THEME_NESTING`（32；清单需要
  2 层）的 JSON 在解析之前就报告 `invalid-json`，因为这样的值能解析成功，但当宿主序列化整个
  `themes.list` 响应时会栈溢出。
- 变更检测与 [[submodule-server-locales]] 相同，是每秒一次的文件夹签名轮询（为何不使用
  `fs.watch` 见其规格），且文件夹在启动时创建一次，绝不重建。
