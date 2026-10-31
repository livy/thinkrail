# auth —— 应用内的 provider 登录 UI

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

应用内 provider 登录的客户端一半：累积式的客户端侧**登录状态类型**，以及渲染它们的**纯呈现性**
OAuth 对话框。该对话框是 `chat/ExtUiPrompt` 的无会话类似物 —— pi 的流程，经 wire 桥接 —— 但渲染为
modal，因为登录在 Welcome 界面上运行，那时还不存在任何会话或 chat 可供提问。（与 `chat`／ExtUi 的
拆分相镜像，**frame 归约器位于 `store`**，不在此处 —— `auth` 保持纯呈现性 + 类型，因此这里没有任何
东西导入 `store`。）

## 边界（Boundary）

- **拥有：**
  - `loginState` —— **类型** `LoginState`（+ `LoginInput`／`LoginInputSelect`／`LoginInputPrompt`）：
    一次登录的客户端累积状态。它的文档钉住 store 归约器所实现的累积契约 —— frame 是**叠加**而非替换
    （`url` 可以与一个粘贴的 `prompt` 同时存在，即浏览器与粘贴的竞态），`status` 走
    `active → success/error`。可选的 `target` 指明某次 MCP 登录或连接测试的 MCP 服务器
    （`LoginPush.target`）。**仅类型**（与 `chat/types` 一样）。
  - `LoginDialog` —— **props 驱动、无 store／transport**（像 chat 渲染器那样）：渲染一个 `LoginState`
    （一个 **Open sign-in page** 链接 —— 真正的锚点，因此原生复制／中键点击可用 —— + **Copy link**；
    绝不显示原始 URL 文本，它可能长达数百个 scope 参数 —— 一次失败的剪贴板写入改为露出一个单行、
    已选中的只读字段；一个设备码，带一个**可点击的验证链接**，并在到达时尽力在新标签中**自动打开**；
    一个 `select`／`prompt` 输入 —— `select` 渲染为应用的选项卡片，其 pi 终端措辞由 `loginOptions`
    整理（消息去掉尾随冒号；选项标签尾部的 `(…)` 变成卡片的弱化详情行 —— `(default)` 读作
    "Recommended"，而 pi 的终端行话 `(headless)` 用平实的话解释；prompt 的终端提示变成对话框文案 ——
    尾部的 `(blank for X)` 变成一句平白的 "Leave empty for X."（GitHub Copilot 的
    Enterprise-domain 步骤），而 "press Enter" 读作 "choose Continue"（Bedrock 的凭证链步骤））；
    带 `secret` 标记的 prompt 渲染为**掩码**（API key，issue #97）；进度／工作 spinner —— 当一个登录
    URL 仍存活且没有必需的 prompt 待定，spinner 读作宿主文案（"Waiting for you to finish signing
    in…"）而不是 pi 面向终端的进度（回环端口、回调 URL）；终端的成功／错误）并回调
    `onReply(value)`／`onCancel()`／`onClose()`。文案说 **"Connect"**，而不是 "Sign in" —— 同一个
    对话框同时服务 OAuth 与 API-key 录入。只用 token 工具类上主题；`@remixicon/react` 图标；
    shadcn `Dialog`／`Button`。
  - `loginOptions` —— `loginSelectMessage`／`loginOptionParts`／`loginPromptMessage`：上面那套纯
    呈现性拆分；`loginOptionParts` 对它改写的注记取用调用方的翻译器，而 pi 自己的标签与消息以 pi 的
    语言原样通过。
  - `uiOrigin` —— `isLoopbackHost`／`uiServedFromLoopback`：UI 是否经回环宿主打开，即对话框对
    “浏览器跑在 ThinkRail 宿主机上”的替代判断；`redirectsToLoopback`：某个 authorize URL 的
    `redirect_uri`／`callback_url` 是否是 `http:` 回环地址。
- **公开接口（barrel `index.ts`）：** `LoginDialog`；`LoginState`／`LoginInput*`（类型）。
- **允许的依赖：** `@thinkrail/ui/dialog`、`@thinkrail/ui/button` 与 `@thinkrail/ui/utils`（`cn`）；
  `lib`（`copyText`）；`@thinkrail/locales/react`（`useT`／`<T>`，用于对话框自身的文案）。（状态类型
  不需要任何导入。）
- **禁止：** 导入 `store`／`transport`（对话框保持纯呈现性 —— **面板**才是集成部件，正如
  `chat/ChatView` 之于 chat 渲染器）；任何 `pi`／`server` 导入。

## 集成（由调用方拥有，不在此处）

- **store** 持有唯一的 `activeLogin: LoginState | null`（扁平、无会话），在 Providers 与 MCP 任一方
  发出请求之前同步预留登录启动，并拥有 frame 归约器（`foldLoginFrame`）—— 绝不在某个会话运行时之下
  （那条路径会在会话之前丢弃 frame）。一次失败的启动会释放预留；一个 frame 仍可能在启动响应之前创建
  匹配的对话框。迟到的响应永不替换另一个活动登录。
- **`provider.login` 通道**在 `transport/wireTransport` 中被路由到 `store.applyLoginFrame`。
- `panels/ProvidersSettings`（Settings 对话框的 Providers 区段）是集成部件：它启动登录
  （`provider.loginStart`，`type` 为 `"oauth"` 或 `"api_key"` —— 两条认证路由走同一通道，issue #97
  —— → `store.beginLogin`），从 `store.activeLogin` 挂载 `LoginDialog`，把 `onReply`／`onCancel`
  接到 `provider.loginReply`／`loginCancel`，并在一次登录（或登出）结算后重新拉取 `provider.status`；
  它忽略携带 `target` 的登录。那套接线只存在于 `panels/useProviderAuth` 一处，与 onboarding 的连接
  界面（`ConnectProviderDialog`、向导的 Connect 步骤）共享；同一时刻只有一个挂载该对话框 —— Settings
  在它打开期间。
- `panels/McpSettings` 是第二个集成部件：`mcp.login`／`mcp.testConnection` 返回一个登录 id，其
  `provider.login` frame 只到达发起连接；它把服务器作为 `target` 启动登录，只为有目标的登录挂载同一个
  `LoginDialog`，并通过同样的 `provider.loginReply`／`loginCancel` 应答。登录页面仍只从对话框的按钮
  打开（桌面 shell 把 `window.open` 路由到系统浏览器）；粘贴字段接受完整的重定向后 URL。

## 必须做对（Get right）

- **frame 是累积的** —— 当一个粘贴式 `prompt` 到达时，归约器不得覆盖 `url`，反之亦然。
- 对话框以 `key={state.loginId}` 挂载，因此一次新登录获得全新的本地（文本／忙碌）状态。
- **粘贴兜底，保守分类。** pi 以同样的方式发送竞态的回环兜底与一个*必需的*代码 prompt（例如
  Anthropic 的 copy-code）（`manual_code` → `prompt`），因此对话框去推断它：只有当活动 URL 的
  `redirect_uri`／`callback_url` 是一个 `http:` 回环地址时，`prompt` 才是兜底。其他任何情形 —— 一个
  https 的 copy-code 重定向、没有重定向参数、畸形 URL —— 都把该 prompt 渲染为带 pi 消息的普通可见
  输入。这是一个呈现层启发式，并不证明有监听器在运行（Codex 在 1455 端口被占用时保留其回环 URL，
  随后就需要粘贴），因此兜底永远只差一次点击，且其文案覆盖两种情况："Sign-in didn't finish? Paste
  the final URL"。远程浏览器只能通过粘贴完成（回环重定向在那里失败），因此该兜底在回环 UI 源
  （桌面、本地开发）上**折叠**，在任何其他源上**展开**；SSH 端口转发看起来是本地的却仍然需要它
  （已接受）。它的单行提示是宿主文案，并替换该 prompt 的消息。
- 设备码**自动打开**是尽力而为（弹窗拦截器可能拦住异步的 `window.open`）；可点击的验证链接是
  有保证的兜底，因此流程永不会因弹窗被拦而走死。
