# mcp 渲染器（mcp__* 调用、资源工具、tool_search）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

pi 内置 MCP 引擎的展示型渲染器，按工具名接入：`McpToolCard` 负责每一次
`mcp__<server>__<tool>` 调用（注册表里唯一一条前缀注册），`McpResourceListCard`
（`list_mcp_resources`、`list_mcp_resource_templates`）、`McpReadResourceCard`
（`read_mcp_resource`）、`ToolSearchCard`（`tool_search`），以及纯函数的 `mcpResult` 读取器。
`register.ts` 把它们装配为常规的卡片 chrome 工具，并由父级 `tools/register` 以副作用方式
导入。

## 卡片读取什么（What a card reads）

- 定稿的结果只保留 `content`、`details` 与 `isError`。卡片按顺序渲染 `content` 文本块，像
  pi 那样用换行拼接：pi 已把资源链接、内嵌资源、音频与已保存的 blob 都转成文本行。卡片绝不
  绘制图片块 —— 父级通用的结果内容层会在每个渲染器之后追加那一条图片条带。
- `details` 携带 pi 的 `McpToolDetails` 和宿主的 `details.thinkrail` `McpResultSummary`。
  每个字段都过一道守卫；任何缺失或畸形的字段都按旧版转录的样子渲染（标题取自名字，文本取自
  `content`）。摘要的 `blocks` 不绘制，因为 pi 的文本已承载它们；卡片读取
  `structuredContent`、`structuredContentTruncated` 与 `isError`。
- 标题取 `details.server` / `details.tool`（pi 未清洗的名字），否则取工具名在 `mcp__` 之后
  第一个 `__` 处切分的结果 —— 抛异常的调用携带的是 `details: {}`。被哈希缩短的名字按尽力而
  为的方式解析。
- 参数是一个键/值列表（字符串原样，其他值用缩进 JSON，即 pi 的展开形式）；折叠态摘要是 pi
  的 `key=JSON` 单行，超过 100 字符即截断。
- 结构化内容只有一个可折叠的 JSON 树；表格视图推迟。它折叠在单个 disclosure 之下。展开时，
  至多 40 个节点的树完全展开，更大的树只展开顶层，单个分支列出 100 个子节点后才出现
  "Show all"。每个 disclosure 都以 JSON-pointer id 记入共享的 fold 缓存，因此状态能在虚拟
  化下存活。`structuredContentTruncated` 读作 "Structured content too large to keep."。
- 失败的调用（`status: "error"`，或摘要的 `isError`）像其他工具错误那样为其输出着色。

## 动作（Actions）

两者都通过 `ChatActions` 到达（绝不经 store/transport）。`null` provider（subagent 转录、
独立渲染）或早于 MCP 的宿主会把它们隐藏。

- **完整输出** 在 `details.fullOutputPath` 被记录且 provider 提供 `readMcpOutput` 时出现；
  `ChatView` 把它实现为 `mcp.readOutput({ workspaceId, sessionId, toolCallId })`。卡片从不
  发送或链接该路径 ——宿主读取的正是结果所记录的那个文件。文本在一个按内容定尺寸（最高 85vh）
  的对话框中打开，而不是内联，因为一行里多达 1 MiB 会扭曲虚拟化转录的几何。状态有：
  loading、text（注明宿主截断）、*expired*（pi 的临时文件已消失）、*unavailable*，以及带
  Try again 的请求失败。加载到的答案在卡片保持挂载期间保留；失败则在下次打开时重读。与共享
  的图片预览一样，对话框属于卡片所在的那一行而非 `ChatView`（与 subagent 转录不同，它不需
  要集成状态），因此一次把该行滚出虚拟化范围的运行会把它关闭。
- **Sign in / Open MCP settings** 用一个按钮替换 pi 那些面向 CLI 的指令
  （`requires sign-in`、`needs sign-in`、`mcp login`、`Run /mcp …`），该按钮调用
  `openMcpSettings`，打开 Settings › MCP servers；当这个按钮被提供时，所显示的错误文本（调
  用输出与列表的逐 server 错误同理）会去掉 `Run /mcp …` / `Run /login <provider> …` /
  `pi mcp login …` 那句话（`stripMcpCliInstruction`，经由 `useMcpFailureText`），并保留
  server 自己的消息，而没有 `ChatActions` 的独立渲染则逐字保留 pi 的文本。provider 支撑的错
  误（`Run /login <provider>`）是另一回事：它们的动作打开 Settings › Providers，绝不是 MCP
  OAuth，其文本以非 CLI 指令的形式保留 provider 身份。没有该 provider 动作时，原始指令仍然
  可见。登录类短语优先于裸的 `/mcp` 指令。检测只在失败结果和列表的逐 server `errors` 上运
  行，那正是 pi 发出这些短语的地方；仅仅是提到登录的成功 server 输出仍然是文本。

## 资源与 tool_search 卡片（Resource and tool_search cards）

- **列表** 给出 server 名，list-all 调用则为 "All servers"（pi 用 `details.server: ""` 表
  示）。行来自宿主的结构化 payload（整份列表，至多 64 KiB），否则来自 JSON 文本。pi 会对超
  过 20 KB 的文本做中部截断，因此解析不出的文本回退为原始输出并附带 Full output。list-all
  会多出一列 server；显示 20 行后出现 "Show all"；`nextCursor` 表示还有更多；逐 server 的
  `errors` 渲染为携带登录动作的错误行。
- **Read** 显示 server 与 URI，然后是内容文本；图片走共享的条带。
- **`tool_search`** 显示查询（以及 limit），然后是 `details.loaded`，其中带 pi 的
  `- name: description` 行里的单行描述。无匹配或 details 未知时逐字显示 pi 的文本。

## 边界（Boundary）

- **拥有：** 这些渲染器、`mcpResult` 读取器，以及它们的注册。
- **公开接口：** 只有副作用 `register`（与父级一样按文件导入）。
- **允许的依赖：** 父级 chat 原语（`toolRegistry`、`ChatActions`、`foldState`）、同级
  `Collapsible`；`@thinkrail/extension-api/web`；`contracts`（仅类型 —— `McpToolDetails`、
  `McpResultSummary`、`McpContentBlockSummary`、`McpReadOutputResult`）；`@thinkrail/ui/*`；
  `@remixicon/react`；`@thinkrail/locales`（UI 字符串）。
- **禁止：** 值导入 pi 或任何 MCP 包；`store`/`transport`。列表的行是 pi 文本 JSON 的本地
  解析视图，不是线上契约。

## 必须做对（Get right）

- MCP 调用与其他工具一样保持**常规**：需要登录的失败折叠进活动运行，chat 资源的注意标记仍
  是那一眼可见的信号。这次调用的审批不是这些渲染器的职责：宿主把它锚定到调用上，由工具的
  shell 渲染，并在等待期间把该步骤从折叠的 fold 里弹出（chat SPEC › Extension dialogs）。
- `tool_search` 与三个资源工具都是精确注册；只有 `mcp__` 按前缀解析。
- 仅使用 token 工具的样式（不用裸十六进制 / 内联 `style`）。
