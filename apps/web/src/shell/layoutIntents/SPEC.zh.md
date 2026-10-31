# shell/layoutIntents —— 排布意图的编排

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

为一个已挂载的工作区消费与排布无关的 store 意图，并把每个意图翻译成一次纯本地 frame／workspace-view
事务，外加相应的 attention／焦点转换。

## 边界（Boundary）

- **拥有：** 过期的 frame／view／attention 身份守卫；仅消费一次的处理；目标与 navigation 仲裁；
  open／select／close／tool／terminal／辅助开关的派发；attention／焦点计算；以及为某个结果至多发出
  一次原子本地转换。
- **公开接口（`index.ts`）：** 工作区意图处理 hook 与范围很窄的回调类型。
- **外部依赖：** store 的意图、attention 与 navigation API；仅用于领域请求的 transport 错误归一化；
  React。
- **禁止：** 当前布局的 WS 调用；本地持久化归属；会话或终端目录／生命周期；面板渲染；server/shared/pi
  导入；或在纯 `layout` 兄弟模块之外做可变拓扑逻辑。

只有确认意图捕获的 frame、工作区 view 与 attention 仍然是最新的之后，该意图才会被消费。被延后的
chat／history 工作保留其请求时的 navigation 戳，因此迟到的完成无法偷走焦点。每个发出的焦点请求都
携带其工作区 id；已挂载的 workbench 会拒绝来自被取代工作区的请求。全局终端 placement 解析为该
工作区最后存活的底部焦点，并选择一个已有的兼容槽位；创建新的 frame 槽位是一条显式的 frame 命令，
绝不是领域归约的隐藏后果。上下文的组 id 仍然优先。

底部显示／切换使用与左／右相同的仅消费一次转换，包括在提供无进程的终端创建界面之前进行合格的
单例恢复，以及在隐藏时做非底部焦点恢复。隐藏某一侧或底部只是恢复：只有在焦点确实丢失（原本位于
被隐藏区域内，或位于不再渲染的元素上）时，其焦点请求才把 DOM 焦点移到中心组选中的标签上，而
attention 照旧更新。焦点在别处时，例如聊天草稿或某个问卷选项，保持不动；一次被抢的焦点曾取消某份
问卷的 attention 声明并让 `e2e/ask-user-question.spec.ts` 变得不稳定。显示某个区域绝不调用终端创建；
只有显式的 New terminal 动作才会。一个全部折叠的区域在首次切换时恢复一个合适的最近聚焦组。产生的
文档与 attention 一起提交，而不是通过各自可观测的写入。单例工具动作修改那一个 frame；在已有组之间
的资源打开／移动只修改被指定的那个工作区 view。
