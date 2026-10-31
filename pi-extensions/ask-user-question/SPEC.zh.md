# @thinkrail/pi-ask-user-question —— 面向原生 pi 的 ask_user_question 工具

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

结构化提问能力中已发布、可移植的那一半：一个工具 `ask_user_question`，它暂停运行直到有人回答
选择题，并返回一个面向模型的外壳。它对 ThinkRail 一无所知。它以三种方式之一提问：
终端问卷（pi 的交互模式）、每个问题一个对话框（VS Code 或 Zed 这类 RPC 宿主），
或嵌入者注入的询问器。ThinkRail 通过 [[module-ext-ask-user-question]] 组合它，
注入一个由宿主答案邮箱支撑的询问器。

## 公开接口（Public surface）

- **默认导出**：带内置询问器的工厂，`pi install` 通过 `pi` 清单加载它。
- **`createAskUserQuestionExtension({ ask?, announce?, guidance? })`**：
  - `ask: AskUserQuestionAsker` 取代内置提问。此时嵌入者就是 UI，因此没有 `hasUI` 门禁。
  - `announce(toolCallId)` 从 `message_end` 钩子中、针对该批次所保留的那个 ask 调用被调用，
    条件是助手的停止理由允许它执行（`executableAskCallIds`）。一个在 `execute()` 之外跟踪调用的
    嵌入者从这里开始跟踪，因此只有本扩展所运行的调用被跟踪。
  - `guidance` 取代描述、`promptSnippet` 与 `promptGuidelines`，用于 UI 行为不同的宿主，
    例如 ThinkRail 的 composer 取代。
- **`./core`**：无运行时导入。导出工具名、限制、保留标签、结果类型与文本，以及
  `validateQuestionnaire`、`buildQuestionnaireResponse`、`outcomeResult`、`isAckDetails`、
  `isAskUserQuestionArgs`、`isAskUserQuestionResult`、`isolateAskUserQuestionBatch` 与
  `executableAskCallIds`。这些正是嵌入者在 `execute()` 之外修复或继续一个问题所需要的东西。
- **工具契约**，渲染器与嵌入者都以它为键：
  - 名称；
  - 参数形态：`questions[]`，元素为 `{ question, header, options[]{ label, description, preview?,
    recommendedReason? }, multiSelect? }`；
  - 每种结果下的结果：

    | Outcome | `content` | `details` | `terminate` |
    | --- | --- | --- | --- |
    | `answered`（含部分回答与 Skip） | `buildQuestionnaireResponse` | 该构建器的 details | 否 |
    | `deferred` | 规范的 ack，或 “Chat about this” 的聊天文本 | `{kind: "ack"}` | 是 |
    | `abandoned` | 已停止的文本 | cancelled | 是 |
    | 询问器拒绝 | 抛出，于是 pi 记录一个工具错误 | — | — |

  参数与结果类型是问卷形态的唯一定义：ThinkRail 的卡片从 `./core` 类型导入它们，而延期 ack
  `{kind: "ack"}` 兼作宿主的通用 “response pending” 标记（[[module-extension-api]]）。
  改动其中任何一个都会破坏 ThinkRail 的卡片及其重启修复。

## 决策（Decisions）

- **询问器拥有 Stop/Submit 竞争。** `execute()` 从不检查 signal。内置询问器保留 Stop 之前给出的回答，
  否则解决为 `abandoned`：带 `terminate` 的已停止文本，于是 pi 不会再对已中止的运行发出任何模型请求
  （抛出错误会让 pi 发出一次以 aborted 失败的请求）。嵌入者应用自己的顺序；
  ThinkRail 的邮箱让一个已被接受的 Submit 胜过更晚的 Stop。
- **延期结束本次运行。** 带 `terminate: true` 时，pi 跳过该工具批次本会触发的模型请求，
  但仍会排空已排队的 steering。一个排入了用户自己消息的宿主 —— 例如 ThinkRail 的 composer 取代 ——
  因此会带着那条消息继续。已针对 pi 的 agent 循环并在 `src/session.test.ts` 中验证。
- **按模式的内置提问：**
  - `ctx.mode === "tui"`：经由 `ctx.ui.custom` 的终端问卷。若 `custom()` 解析出 `undefined`，
    则回退到对话框。
  - 其他任何有 UI 的模式：对话框遍历器。
  - 无 UI：不提问，直接报 “not available”；工作流 skill 随后在已陈述的假设上继续。
- **终端问卷与 ThinkRail 的卡片一致。**
  - 相同的答案模型：`src/questionnaire.ts` 刻意移植了卡片的 `deriveAnswer`，因为卡片不得导入 pi 包。
  - 单选：↑↓ 移动并选中。多选：Space 切换。
  - Other 打开一个 pi-tui `Editor`；在其行上输入即启动它。`n` 添加一条注记。
  - “Recommended” 徽标与 `Why:` 行。
  - 所选选项的 markdown 预览：100 列以上放在列表旁边，否则放在其下方，上限 24 行。
  - 一个评审页，列出未回答的问题，Esc 跳过，以及一行 “Chat about this”。
  - 已知限制：问卷高于终端时没有视口。
- **对话框遍历器以丰富度换可达性。**
  - 每个问题一个 `select`，带 “Other…” 与 “Chat about this”。
  - 多选是一个 `input`，形如 “1,3; own text”。
  - 预览进入标题（600 字符）。没有注记，也没有评审页。
  - 关闭任何对话框即拒绝整个问卷。
- **一个批次一个 ask。** `message_end` 钩子把第一个 ask 保留为唯一的工具调用，因为 pi 的顺序中止漏洞
  会让未执行的同级调用没有结果。模型在得到回答后重新发出它们。该工具是 `sequential` 且
  `model-only`，因此 codemode 脚本无法调用它。
- **问题数量不设上限；长度限制仅是建议性的。** 限制只存在于字段描述中：schema 的 `maxLength`
  会拒绝整个调用并把每个参数回显回来。保留标签（“Other”、“Type something.”、“Chat about this”、
  “Next →”）被拒绝，因为 UI 会自己添加它们。

## 边界（Boundary）

- **允许的依赖：** 对等依赖 `@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui` 与 `typebox`；
  仅测试用的 `@earendil-works/pi-ai`（faux provider）。
- **禁止：** 任何来自 ThinkRail 的东西（`@thinkrail/*`、`apps/*`、`packages/server`）以及已交付代码中的
  Bun API。

## 结构（Structure）

- `index.ts`：导出与默认导出。
- `src/core.ts`：契约。
- `src/schema.ts`。
- `src/extension.ts`：工厂与默认 guidance。
- `src/dialogs.ts`。
- `src/questionnaire.ts`：答案模型。
- `src/tui.ts`：问卷与消息记录渲染器。

## 验证（Verification）

- 每个模块的单元测试。
- `src/session.test.ts` 在 pi-ai 的 faux provider 上于真实 `AgentSession` 中运行该工具：
  - 一个回答会继续运行；
  - “Chat about this” 在模型调用一次之后结束它；
  - 同级工具调用被丢弃。
- 一致性门禁在 Node 下用脚本化 UI 对打包后的 tarball 驱动终端与对话框路径。
- 首次发布之前：在真实终端（Ghostty 或 iTerm2，以及 tmux 内）做一次手工检查。
