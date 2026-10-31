# 原生制品测试装置

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

面向编译后 CLI 与原生桌面制品的、仅源码的测试基础设施：真实进程适配器、共享的
HTTP/WS/资源/扩展探针、扩展后的桌面冒烟测试，以及首次安装冒烟测试。
它消费已完成的制品；它不构建应用，也不提供应用运行时代码。

## 边界（Boundary）

- **拥有：** 制品定位器、隔离环境、原生/安装器冒烟入口、共享宿主探针、它们的夹具和辅助单元测试，
  以及公开构建动作的桌面打包调用与制品收集器契约测试。
- **公开接口：** `locateDesktopLauncher`
- **允许的依赖：** CLI 的公开制品名辅助函数；服务端获准的历史夹具导出；shared 的发布身份、
  其 Windows 文件身份与图标，以及带重试的清理；`resedit`（读取 Windows 可执行文件资源）；
  以只读方式访问
  `.github/actions/build-binary/action.yml`，以在隔离的夹具目录中执行其打包调用和收集器；
  Bun/Node 和原生安装器工具。
- **禁止：** 应用或 SDK 源码内部、Electrobun import/依赖、伪造的宿主或 agent、生产包 import 本包，
  或在测试期间变更真实用户状态。

根冒烟命令直接执行 CLI、桌面和安装器入口。浏览器 e2e 编排留在 [[module-browser-e2e]]，
并通过本包的 barrel import 那个纯定位器。单元测试与严格类型检查使用常规的工作区 Turbo 任务；
这里没有包构建步骤、SDK 准备、声明生成或 Playwright 测试发现。产品单元测试留在拥有它们的模块中。

## 制品验证（Artifact verification）

共享探针会启动真实制品、在没有 pi 可执行文件的情况下加载一个合成的外部 PI 扩展
（该夹具值导入一个裸 `@earendil-works/pi-coding-agent` 说明符，并在其模型名中暴露导入的值，
从而让这次加载经由宿主强制的 transform 路径演练 pi 的虚拟模块映射——缺少 `PI_BUNDLED_NODE`
define 的打包会在这项探针上失败；[[submodule-server-agent]] 拥有该接缝），
演练内置工厂/skill，运行**两个脚本化的 provider 轮次**——该夹具还在自己的 `api` 名下注册一个
pi-ai 的 faux `streamSimple`，它用一个悬空的 flowchart 边调用 `visualize`，因此探针断言严格的
mermaid 解析错误以 `isError` 工具结果的形式抵达转录（打包后的宿主组合的是 ThinkRail 扩展注册表，
绝不是可移植包的尽力而为探针；[[module-ext-visualize]]），并在第二个提示中调用
`ask_user_question`，因此探针看到带 `needsInput.kind === "tool-response"` 的会话阻塞，
通过 `session.respondToTool` 作答，并把答案读回为该调用非错误的工具结果
（注册表的按会话组合以及制品内的宿主邮箱；[[module-ext-ask-user-question]]）——
在真实聊天中启动 pi 内置的 MCP 引擎（一个写出的 stdio 夹具必须报告 *connected · 1 tool*，
而一个用受保护资源元数据回 401 的探针内 HTTP 服务器必须报告 *needs sign-in*，
通过 `mcp.list` 读回，从而证明惰性导入的 MCP 运行时在制品内执行），在不进行 provider 轮次的情况下
抵达一个 OAuth URL，校验健康/UI 和转录回收，然后关闭。CLI 专用探针还会检查其仅退出（exit-only）
和嵌入式缓存行为。在 Windows 上，两个冒烟测试都先读取可执行文件的资源，而不运行它们：
- CLI 冒烟测试读取构建出的 exe；
- 桌面冒烟测试读取打包内的 `bin/bun.exe`、`bin/launcher.exe` 和 `Resources/uninstall`。

每个可执行文件都必须通过两项检查，任一项失败都会在发布前让该发布失败：
- **图标。** 外壳显示的组是资源顺序中的第一个组；在 Bun exe 中，Bun 命名的
  `IDI_MYICON` 会遮蔽任何编号组。该组必须包含该 ICO 的最大帧，且只持有该 ICO 的帧。
  参照 ICO 对 CLI 是 shared 的 `windowsIconPath`，对桌面是打包自己的 `Resources/app.ico`。
- **VERSIONINFO。** 它必须等于 [[module-shared]] 对检出目录发布身份的 `windowsFileIdentity`，
  因此冒烟测试要在构建该制品的同一个带戳检出目录中运行。

资源目录遍历是本地实现的，因为 `pe-library` 在 `.rsrc` 之后拒绝 Bun 的 `.bun` 段；
由 `resedit` 解析图标组和版本结构。原生桌面冒烟测试加载真实 UI，校验路由/preload 消息
以及生产的外部导航处理器。在 macOS 上它还会端到端驱动标题栏双击路径：无拖动区域的双击必须不起作用，
而页头双击记录的动作和最终窗口状态必须与机器 `AppleActionOnDoubleClick` 设置的映射一致
（未设置、`Maximize` 和 `Fill` 缩放），如 [[module-desktop]] 所述。Linux 跳过它，因为其带装饰窗口
没有 web 拖动区域；Windows 跳过它，因为其原生标题命中已在双击时最大化。在 Windows 上它改为在活的
webview 中点击 HTML 最大化按钮及其随后的 Restore 后继按钮；启动器必须每次点击记录恰好一个
`toggleMaximize` 请求，并发布匹配的 `maximized` 状态，从而证明真实的 bridge 请求链路和原生状态推送，
而不是伪造的 bridge。关闭不被探测：它与原生 X 共享 `requestClose()` 路径，
而随后的外部导航探针还需要该窗口。基于桌面的 Playwright 使用启动器选择加入的中性窗口接缝，
因此它是唯一水合的客户端。活窗口的 ready/控制接缝仍留在启动器中，绝不对本包做运行时 import。

常规制品子进程设置 `CI=1`，在不改变生产运行时模式的前提下静音所有分析层级；仅
`THINKRAIL_NO_ANALYTICS` 只抑制附加事件。`bun run smoke:desktop --analytics [launcher]`
则针对回环 PostHog 收集器运行受控的隐藏宿主探针，绝不使用供应商端点。它会移除继承的
CI/测试/可选静音设置以及面向人类模式启动的代理，检查首次打包初始化在 `app_started` 之前发出
`app_installed`，检查重启只发出 `app_started`，并钉住标准的桌面/发布/平台来源信息
外加贯穿两者的一个 UUID（包括附加事件退出）。某次启动碰巧发出的任何 `$exception` 错误报告
留在该启动序列之外，但必须共享该 UUID、保持无人物标识，并且只携带允许列表内的普通属性。
随后它分别证明 CI 与测试静音。不模拟任何 UI 动作或确认；投递在正常宿主关闭之后检查。
期望的发布身份是构建该制品所用的共享身份，因此探针需要一个带发布戳（带密钥）的制品；
未打戳的构建正确地不发出任何事件。

每个宿主拥有隔离的 home、data、agent 和 cache 目录；环境覆盖尊重 Windows 大小写不敏感的键。
原生 UI 启动会预先创建其 home 目录：在 macOS 上，当设置了 `XDG_CACHE_HOME` 而 `HOME` 指向一个
不存在的目录时，窗口永远到不了 DOM-ready。就绪轮询拥有有限期限并观测根的早期退出；由于启动器
非原子地写入其 ready、route 和导航探针文档，冒烟测试把缺失或写了一半的 JSON 文件视为"尚未就绪"
并继续轮询，而不是因撕裂读失败。当活着的已启动根或已校验 ready 文档中的 app/启动器 PID 需要失败清理时，
清理是有界且被 await 的，之后才对临时安装根做共享的带重试移除。setup 根可能在它的 app 交接就绪之前
就成功退出；该退出之后，装置不会从 setup 进程推断或发现后代进程。首次安装冒烟测试运行真实的
DMG 应用、Windows ZIP setup 或 Linux tarball 安装器，观测安装器自动启动应用，校验健康与正常的
控制文件关闭，并等待安装器/宿主/启动器退出。仅测试装置使用的安装器 UI 自动关闭标志关闭的是完成对话框，
而不是错误或断言。

Windows 安装器冒烟测试只允许在一次性 GitHub 托管的 Actions runner 上运行：v2 安装会在 HOME 隔离
之外写入真实的已知文件夹快捷方式和 HKCU 注册。该守卫在创建文件或启动安装器之前运行。
其他原生冒烟测试仍然同时隔离 HOME 和 USERPROFILE。

定位器解析有文档记载的应用/安装器路径和按通道的 setup 名，而不是从含混目录中取第一个匹配的制品。
Windows setup 后缀保留 stable/canary 区分。一次 macOS 运行不构成 Windows/Linux 原生执行或下载时
Gatekeeper 接受的证据。签名/公证验证属于 [[module-ci-release]] 的私有服务交接。
