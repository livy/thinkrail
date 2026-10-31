# Web 扩展组合注册表

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

web 扩展的静态组合点。`registry.ts` 拥有它们的有序列表；`ChatView` 在模块加载时、于应用本地工具注册
之后调用注册。每个描述符的工具条目按顺序喂给既有的 chat 注册表，保留其后注册胜出的行为与兜底渲染器。
重新注册同一列表会替换条目而不是累积它们。

注册忽略宿主的扩展开／关设置：一个 web 扩展的渲染器在其服务器侧被禁用时仍保持注册，因此已经含有其
工具调用的转录会继续渲染它们。该开关只决定宿主接下来给 agent 什么。

## 边界（Boundary）

`index.ts` 只导出 `webExtensions` 与 `registerWebExtensions`。导入只触及扩展的公开 `./web` 入口、
web SDK 契约，以及 chat 的注册函数；绝不触及 `ChatView`、store、transport、扩展内部或服务器侧。注册表
Map 仍归 chat 所有，因此本模块不创建平行的状态所有者，也不创建运行时扩展加载器。`registerWebExtensions()`
在应用引导时（`main.tsx`）运行一次：交互状态与 plan／review 概览都派生自注册表中的 `interactive` 注册，
因此它必须在任何惰性视图读取转录之前完成。直接的组件消费者独立于工具注册使用扩展的具名公开导出：
Markdown 渲染 `MermaidView`。计划页面经注册表而非按名称到达问卷卡片。
