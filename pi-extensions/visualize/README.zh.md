# @thinkrail/pi-visualize

> 中文译文，原文见 [`README.md`](./README.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

一个 [pi](https://pi.dev) 扩展，为 agent 提供 `visualize` 工具，用于**图表**（Mermaid）、
**选项对比**和**文件预览**（例如 HTML mockup，可选带改动前后对比），因此当它解释架构、流程、权衡或
UI 改动时，会改用一张图，而不是 ASCII art 或 markdown 表格。

```bash
pi install npm:@thinkrail/pi-visualize
```

## 你会得到什么（What you get）

- **`visualize` 工具**——`type: "diagram"` 配原始 Mermaid 源码，或 `type: "comparison"` 配若干选项
  （名称、描述、优点、缺点、`recommended`、可选的内联 Mermaid），或 `type: "file"` 配一个工作区
  `path`（以及一个可选的 `before` 路径指向另一个文件以做改动前后视图），另可带一个可选的 `title`。
  该工具的 prompt 准则要求 agent 在提出任何 UI 改动时编写自包含的 HTML mockup 并用这种方式展示。宿主
  会渲染文件的当前内容，因此 agent 持续编辑的 mockup 会在所有展示过它的位置更新。
- **终端渲染。** 流程图、状态图、时序图、类图与 ER 图、饼图、思维导图、时间线以及 git 图会以 Unicode
  制表符直接画在 pi TUI 中，使用你 pi 主题的颜色（通过
  [`lovely-mermaid`](https://github.com/xl0/lovely-mermaid)，即 pi 在聊天消息中渲染 Mermaid 所用的同
  一族引擎）。当图表宽于你的终端、只能画出其中一部分，或属于渲染器不认识的族（gantt、xychart…）时，
  该工具改为在代码围栏中显示 Mermaid 源码——它从不把图表折行。展开工具结果可看到图下方的源码。对比会
  渲染为 markdown。文件预览会打印路径——在浏览器中打开它们；有文件渲染器的宿主（ThinkRail）会内联渲染
  该文件。
- **处处可读。** 该工具的文本结果是 markdown——一个 ```mermaid 围栏或分段式对比——因此在 `pi -p`、
  JSON/RPC 模式，或任何没有渲染器的宿主中也同样可读。

## 校验 —— 工具检查什么（Validation — what the tool checks）

该工具会拒绝空源码与形状错误（图表缺少 `mermaid`、对比缺少 `options`），并用一条告诉模型如何修正调用
的消息说明原因。

Mermaid 语法由渲染器自身的解析器做**尽力而为**的检查：对于它能绘制的族，该工具会拒绝未知的流程图方向
（`flowchart XX`）、什么都解析不出来的源码，以及解析器不得不丢弃的任何片段（悬空的 `A -->`、未闭合的
`[label`、无法读取的行），并指出丢弃了什么。对于终端渲染器来说过大的图表（超过 128 个节点）不会被拒绝
——它们会作为源码渲染。它**不会**运行真正的 Mermaid 解析器：该语法比较宽松，因此某些畸形输入仍能通过，
而 `accTitle`/`accDescr` 行虽然在 Mermaid 中被接受，却会被拒绝。其他族（gantt、xychart…）不经检查直接
通过并作为源码渲染。在 TUI 中源码永远只差一次按键，所以没有任何东西被隐藏。

自己渲染 Mermaid 的宿主可以接入一个严格校验器：

```ts
import { createVisualizeExtension } from "@thinkrail/pi-visualize";

export default createVisualizeExtension({
  // Throw to reject; the message is returned to the model with the field location.
  validateMermaid: async (source) => { await mermaid.parse(source); },
});
```

注入的校验器会**替换**默认检查。宿主还可以拒绝它无法展示的文件预览路径，并指定 prompt 准则把 mockup
送入的暂存目录：

```ts
export default createVisualizeExtension({
  checkFile: async (path, cwd) => { await access(resolve(cwd, path)); },
  scratchDir: ".scratch/mockups/",
});
```

没有 `checkFile` 时不检查路径（该包保持不依赖 `node:`）。

## 要求（Requirements）

pi ≥ 本包 `package.json` 中 `devDependencies` 所列的版本（pi SDK 包是 peer 依赖，由你安装的 pi 提供）。
在 Node（原生 pi）与 Bun 下均可工作。

## 许可（License）

Apache-2.0 —— 属于 [ThinkRail](https://thinkrail.ai) 的一部分。
