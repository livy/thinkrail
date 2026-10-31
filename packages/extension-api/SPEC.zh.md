# 宿主扩展契约与纯渲染器辅助函数

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

仅源码的宿主扩展契约，独立于呈现组件和宿主状态。浏览器与服务器描述符在不导入任一应用内部实现的
前提下组合能力。宿主保留注册、生命周期与顺序；`define*` 辅助函数原样保留所提供的描述符。

## 边界（Boundary）

- `./web` 暴露 `ToolStatus`、`ToolRenderProps`、`ToolRenderer`、`ToolSummary`、`ToolChrome`、
  `ToolProminence`、`ToolRegistrationOptions`、`ToolInteraction`、`ToolInteractionState`、
  `ToolRevealOptions`、`WebExtension`、`defineWebExtension`，以及来自 `toolHelpers` 和
  `toolResultContent` 的纯参数/结果读取器。
- **交互式工具。** 其 `execute` 会等待人的回应的工具是*交互式*的。该契约在两侧都与工具无关，并以
  `toolCallId` 为键；宿主只从下面两处声明中得知某工具的名字：
  - server：`ServerExtension.interactiveTools?: Record<toolName, InteractiveTool>` 指名哪些工具的
    调用会等待，使宿主能修复因重启而悬空的调用（它写入待处理标记，而不是错误），并从它并未实时
    连接的转录中推导出需要输入。`InteractiveTool.describeResponse?(args, response)` 返回**宿主重启
    之后**投递的回应所对应的、面向模型的文本（这是工具自身 `execute` 已消失的唯一路径）；
    `undefined` 或没有该钩子 ⇒ 宿主写入一段通用前言加上 JSON 形式的回应。实时回应永不经过它：工具
    自己格式化其结果。
  - web：`ToolRegistrationOptions.interactive: true` 请求宿主为该渲染器填充
    `ToolRenderProps.interaction`。
  - **待处理标记。** 交互式调用上 `details` 为 `{ kind: "ack" }` 的结果意味着“回应仍待处理”：宿主
    在重启修复时写入它，而工具在 `awaitToolResponse` 解析为 `deferred` 时返回它（连同
    `terminate: true`）。它是两侧共读的唯一形状（`contracts`：`isToolResponsePending`）；工具不得把
    它用于真实结果。
- `ToolRenderProps.interaction` 是可选的。宿主为每个注册为 `interactive` 的渲染器填充它。它携带：
  - `state`：一个 `response`（`unknown`——宿主重启后在工具结果之外投递的回应，或当最终结果的
    `details` 不是待处理标记时该 `details`），外加 `superseded`、`ended` 和 `accepting`——即宿主此刻
    是否在等待该调用的回应（宿主未在等待的调用，例如来自另一个包的同名工具，会渲染但无法回应）；
  - `respond(response)`——该值原样送往工具；由工具校验其形状；
  - 该聊天的 `focusScope`，使卡片每次聊天挂载时只领取一次注意力焦点；
  - `prepareLayoutChange`，在卡片尺寸变化时保持读者的位置；
  - `revealElement(element, ToolRevealOptions)` 和 `cancelAutomaticReveal()`；
  - `focusComposer()`。

  没有它时，渲染器从 `result` 只读地显示该调用，如同在子代理转录中。渲染器保持被 props 驱动：它
  永不触碰聊天的上下文、store 或传输层。`state` 和 `respond` 是稳定契约；焦点/布局/揭示成员是聊天
  自己的服务借用给你的，可能随聊天而变化（渲染器必须容忍它们缺失）。其 DOM 类型的成员正是本包
  `lib` 包含 `DOM` 的原因；本包自身仍然从不触碰 DOM。
- `WebExtension` 是 `{ name, toolRenderers }`，每个工具名条目在 `options` 下携带一个 `renderer` 和
  可选的 `ToolRegistrationOptions`。`defineWebExtension` 是一个类型化恒等辅助函数：它保留描述符及
  其推断出的工具键，不做注册，也不注入默认值。
- `./server` 暴露 `ServerExtension` 和 `defineServerExtension`，仅带 pi 的 `ExtensionFactory` 类型。
  描述符携带 `name`（身份——Settings 的开关和 `disabledExtensions` 的键都基于它）、供 Settings 目录
  使用的可选显示文案 `title` / `description`、`extensions`、可选的 `sessionExtensions`、可选的
  `interactiveTools`、可选的 `childExtensions`，以及可选的 `skillPackages` 说明符。它的恒等辅助函数
  保留工厂、顺序以及可选字段的缺失状态；加载与技能解析仍属宿主职责。
- `sessionExtensions(session: ExtensionSessionServices) => ExtensionFactory[]` 是扩展使用宿主持有的
  逐会话状态的方式：宿主为每个父会话调用一次，传入该会话的服务。这些服务就是交互式工具邮箱：
  - `expectToolResponse(toolCallId)` 从扩展的 `message_end` 钩子、在 pi 运行它之前，宣告一个该扩展
    将要执行并等待的调用。宿主只跟踪已宣告的调用，因此被关掉的扩展或来自另一个包的同名工具永不
    成为宿主跟踪的交互。
  - `awaitToolResponse(toolCallId, signal)` 解析出一个 `ToolResponse`：`{kind: "response", value}`、
    `{kind: "deferred"}` 或 `{kind: "abandoned"}`。
- 根入口是浏览器安全的，只暴露 web 契约；它永不再次导出 server 那一半。
- 工作区依赖是 `contracts`，以及仅类型的 `@thinkrail/locales/react` 的 `Translator`。React 和 pi 是
  仅类型的契约依赖；不存在 React 组件、注册表、DOM 访问、宿主导入或 pi 运行时值。
- `ToolSummary` 是 `(props, t) => string`：宿主传入来自 `useT()` 的渲染期 `Translator`
  （`getToolSummary(t, name, props)`），因此折叠卡片或常规行在语言变更时会重新计算其摘要；摘要只通过
  该 `t` 翻译，永不用全局翻译器（`check:i18n` 会标记读取全局翻译器的 `summary`
  ——[[module-locales]]）。

规范的工具有结果解析仍由渲染器文本提取和宿主的通用图像层共享：只接受格式良好且受支持的图像块，
保持文本块顺序，并对非规范结果回退到可读值。被 props 驱动的渲染器从宿主获得可选的文件导航，永不
从 store 或传输层获取它。扩展组合规则见 [[module-thinkrail-extensions]]。

## 验证（Verification）

包内测试通过其公开入口检验描述符、纯参数/结果辅助函数，以及对根入口与 `./web` 的浏览器打包。浏览器
打包在运行时只允许 contracts，并拒绝 React、pi、宿主和 server 导入；`./server` 单独验证其 pi 依赖
被擦除。应用的注册表与通用图像层测试保留对宿主拥有的渲染行为的覆盖。
