# web 工具渲染器（搜索 / 抓取 / 存储内容）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

**`pi-web-access`** 暴露的全部三个工具的展示型渲染器 —— 这些工具由 `web-access` ThinkRail
扩展在服务端组合（[[module-ext-web-access]]），该扩展目前还没有 `./web` 这一半 —— 通过**工
具名**接到该能力上：`WebSearchCard`（`web_search`）、`WebFetchCard`（`fetch_content`）与
`WebStoredContentCard`（`get_search_content`）。`register.ts` 通过 `registerToolRenderer`
装配全部三者（外加折叠态头部摘要），并由父级 `tools/register` 以副作用方式导入。

## 边界（Boundary）

- **拥有：** `WebSearchCard`（查询 + provider + 合成答案/来源）、`WebFetchCard`（抓取的
  URL 或本地视频路径 + 抽取内容）、`WebStoredContentCard`（响应 id + 所选查询/URL + 恢复出
  的完整内容）、三者共享的折叠态 Markdown 结果体，以及注册。
- **公开接口：** 除副作用 `register` 之外没有别的（无 barrel —— chat 会拉入 shiki，因此与
  同级模块一样按文件导入）。
- **允许的依赖：** 同级/父级 chat 原语（`toolRegistry`、`ToolFileLink`、`Collapsible`、
  `Markdown`）；`@thinkrail/extension-api/web`（`ToolRenderProps`/`ToolStatus` + 纯辅助函
  数）；`@remixicon/react`。
- **禁止：** 值导入任何 `pi` 包或 `pi-web-access`；`store`/`transport`（渲染器保持展示型
  —— 随时可抽取到 `packages/chat-ui`）。

## 必须做对（Get right）

- **防御式渲染。** `pi-web-access` 的 `details` 结构并非稳定的公开 API，因此对 `result` 做
  尽力而为的读取（provider 名走可选链），否则渲染工具的**文本内容**（`resultText`）—— 绝不
  硬依赖某个 `details` 字段。头部与摘要由 args 驱动：搜索取首个查询；抓取取首个 URL/本地路
  径；存储内容取所选查询/URL，再取响应 id。
- 成功的文本内容会被折叠并以 Markdown 渲染，因此合成引用与抓取到的链接是真正安全的新标签页
  链接，而不是被高亮的 Markdown 源码。错误文本保持纯文本。
- `fetch_content` 只把 HTTP(S) 当作外部锚点。其他任何输入都是结构化的本地路径候选，走父级
  的 worktree 门禁；外部绝对路径保持惰性，而不会变成一个坏掉的同源链接。
- 仅使用 token 工具的样式（不用裸十六进制 / 内联 `style`）。
- 工具名 `web_search` / `fetch_content` / `get_search_content` 必须与扩展完全一致。
