# spec —— 工作树规格图谱读取

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

为只读的 Specs 查看器提供服务：活动工作树规格图谱的全图快照（`spec.graph`），映射为线上契约 DTO。
按需读取——每次调用都经由派生索引重新读取（读时重校验），因此被 agent、编辑器或 git 改动过的规格
在下一次获取时就是最新的。（同级的 `watch` 模块另外提示客户端重新获取——那是失效推送，不是数据
推送；本模块的读取路径保持纯拉取。）它还回答项目级的 **`projectHasSpecs(root)`**——某个仓库是否
带有*任何一个持久*规格（任何 `type` 不是临时性 `task-spec` 的节点）——`host` 通过**惰性的
`project.hasSpecs`** 方法暴露它（这是一次全树遍历，因此只对 Welcome 界面渲染的那一个项目请求，
绝不为每个项目提前请求）。

读取是**同步**的（core 的遍历是同步文件系统，每次调用 O(工作树目录数)）。这可以接受——获取是按需的
（工作区激活 / `workspace.fsChanged` 提示 / 错误状态下的 Retry，无轮询），且逐文件解析缓存会跳过
重复读取。若遍历真的成为瓶颈，升级手段是 core 的 **watcher-as-dirty-flag**（见 `pi-spec-graph`
的 core/SPEC.md），而不是异步包装——那仍会整体阻塞一次事件循环。

## 边界（Boundary）

- **拥有：** `specGraph(workspaceId) → SpecGraphSnapshot` —— 通过按工作区复用的 `SpecIndex`
  （跨调用复用，让解析缓存有回报——与 agent 工具同一模式）读取该工作区的工作树，并把 core 的
  `SpecNode` 映射为 `contracts` 的 DTO（字段集在那里；`title` 回退到 `id`，使线上契约永不携带
  无标题节点）。**只做映射**——没有遍历逻辑；树由客户端构建。`evictSpecIndex(workspaceId)` ——
  丢弃缓存的索引；`host` 在 `workspace.remove` 时调用它，使已归档工作区的解析缓存不会比它活得
  更久（之后的读取只会重建）。**`projectHasSpecs(root) → boolean`** —— 某个仓库**根**（不是工作
  树）是否带有任何**持久**规格（任何 `type` 不是 `task-spec` 的节点——临时草稿 task-spec，例如
  `.thinkrail/context/` 下的，绝不能发出“已设置”的信号），经由按根复用的 `SpecIndex`；这是
  Welcome 界面“Set up project”建议背后的项目级信号。在 glob/解析失败时降级为 `false`，因此绝不会
  破坏 `project.open` / `project.list`。
- **公开接口（barrel）：** `specGraph`、`evictSpecIndex`、`projectHasSpecs`。
- **允许的依赖：** `persistence`（工作区查找）；`contracts`（DTO）；**`pi-spec-graph/core`**
  （不含 pi 的读取模型——宿主侧对该扩展包唯一的值导入，已由 `module-spec-graph` 批准）。
- **禁止：** `host`；同级特性；`pi-spec-graph` 的扩展入口或 `tools/`（与 pi 耦合）；任何 pi 包。
