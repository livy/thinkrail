# @thinkrail/pi-ask-user-question

> 中文译文，原文见 [`README.md`](./README.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

一个 [pi](https://pi.dev) 扩展，为 agent 提供 `ask_user_question` 工具。agent 不再靠猜，而是向你提出
一个或多个结构化的多选题，并等待你的回答。

```bash
pi install npm:@thinkrail/pi-ask-user-question
```

## 你会得到什么（What you get）

- **`ask_user_question` 工具。** 每个问题有一个简短的 `header`、两到四个选项（`label`、
  `description`、可选的 markdown `preview`，以及被标记为 `(Recommended)` 的那个选项的可选
  `recommendedReason`），多个答案都有效时可用 `multiSelect`。一次调用会带上 agent 现在能问的所有问题。
  运行会暂停，直到你回答。
- **终端中的问卷。**
  - 每个问题一页，然后是 **Review & submit**。
  - ↑↓ 移动（单选时这也等于选中），Space 在多选时切换，Enter 确认。
  - **Other** 用于你自己的答案：在该行上输入。
  - `n` 给某个选项附加一条备注。
  - 推荐选项会显示一行 `Why:`。
  - 选项的 markdown 预览在宽终端中显示在列表旁边，在窄终端中显示在列表下方。
  - Esc 跳过问卷。**Chat about this** 结束本轮次，以便你用自己的话回答。
- **RPC 宿主中的对话框。** 在通过 RPC 驱动 pi 的编辑器（VS Code、Zed 等）中，问题一次一个地以原生
  对话框形式到达：一个带 "Other…" 和 "Chat about this" 的选项列表。多选题接收你所选选项的编号
  （`1,3`），其后可选地跟 `; your own answer`。
- **可读的答案。** 工具结果会准确告诉模型你选了什么、你自己的答案、你的备注，以及哪些问题你未作回答。
  会话记录中显示一份紧凑摘要。

在没有交互式 UI 时（`pi -p`、JSON 模式），该工具会报告它无法提问，因此 agent 会改为陈述它的假设。

## 嵌入（Embedding）

拥有自己提问 UI 的宿主可以保留该工具并替换它的提问方式：

```ts
import { createAskUserQuestionExtension } from "@thinkrail/pi-ask-user-question";

export default createAskUserQuestionExtension({
  // Show request.args in your UI and resolve with the person's answer:
  // { kind: "answered", result }, { kind: "deferred" } (the answer arrives as a chat message)
  // or { kind: "abandoned" }. Reject to report the run as stopped.
  ask: async (request) => myHost.ask(request.toolCallId, request.args, request.signal),
  // Optional: learn about each executable call before pi runs it.
  announce: (toolCallId) => myHost.expect(toolCallId),
  // Optional: describe your UI to the model.
  guidance: { description: "…", promptGuidelines: ["…"] },
});
```

`@thinkrail/pi-ask-user-question/core` 导出了这类宿主在工具之外需要的与 schema 无关的部分：结果类型、
面向模型的信封构建器、确认文本以及校验逻辑。
