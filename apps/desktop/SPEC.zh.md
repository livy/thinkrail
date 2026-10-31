# 桌面启动器/客户端（Electrobun）

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

在现有 web UI 与 wire 之上的原生 Electrobun 启动器/客户端。本地宿主配置档把服务端嵌入
Electrobun 的 Bun 进程，在一个 loopback origin 上服务打包的 web 产物，并在原生系统 webview
中打开该 origin。`apps/cli` 仍是同级的浏览器启动器与发布回滚方案。后续的共享客户端
配置档可能拨号到已有宿主，而不引入另一套 UI、wire 或引擎架构。

## 边界（Boundary）

- **拥有：** Electrobun 配置与生命周期；原生窗口策略，包括按平台的标题栏处理以及它向
  web 客户端发布的窗口外壳几何；本地 `bootHost()` 启动；打包资源暂存；PI 兼容的服务端
  运行时 bundle；桌面路由的 preload/持久化；稳定 backend-profile/window 身份之上的有界
  通用客户端偏好适配器；以及可选的活窗口测试接缝。独立产物 harness 属于
  [[module-artifact-tests]]。
- **公开接口：** 打包的桌面应用及其安装器。应用包不导出任何测试辅助库。
- **允许的依赖：** `server` 用于嵌入宿主与 build-support 清单；`shared` 用于发布身份、
  其 Windows 文件身份以及早期的 `resolveShellEnv()` 启动；`contracts` 用于兼容性/原生桥
  类型以及共享的 `createQuitConfirmation` 规则；已完成的构建 web 产物；Electrobun
  `2.0.5` 及其生成的 SDK；仅构建期的 `resedit`，用于弥合 Electrobun 2.0.5 的 Windows
  可执行文件身份缺口（见下）；shared 的原生图标集作为构建输入；Bun/Node。
- **禁止：** spawn CLI 或第二个引擎进程；实现普通产品功能或 agent/领域逻辑；运行时导入
  web 源码；引入桌面专属的 wire 或 UI 状态模型；在后端存储单一活动位置；在没有新的
  验收失败证明其必要性的情况下打包 CEF。原生外壳、生命周期与打包关注点是唯一属于
  桌面特有的行为。

## 配置档与拓扑（Profile and topology）

只有本地宿主配置档会发布。一个 Electrobun Bun 进程在同一事件循环上拥有原生外壳与服务端；
已接受的进程内崩溃权衡保持不变。宿主绑定 loopback 端口 `0`，其实际端口构成窗口 origin。
因此打包的 `web/dist`、`/ws`、`/files` 与 SPA 回退仍同源，web 客户端没有桌面分支。动态
loopback 端口从不持久化。

Desktop、CLI 与源码宿主不按数据目录互相排斥。每个启动器绑定独立的服务端口并初始化自己的
进程内服务。若多个宿主使用同一可变状态，它们的持久化与事件流不做协调，并发改动可能互相
覆盖。

## 启动与打包运行时（Startup and packaged runtime）

1. 在导入任何 server 之前，解析 app 资源并把 `BUN_PTY_LIB` 指向暂存的当前目标 FFI 库。
   Electrobun 输出的是普通 JavaScript 入口，而不是 `bun build --compile` 可执行文件，
   因此它不会内嵌 `bun-pty` 的库。
2. 读取 Electrobun 的 bundle 元数据，然后离开 bundle。Electrobun 的启动器把主进程的 cwd
   设在可替换的 bundle 内（`app/bin`、`Contents/MacOS`），而 Electrobun 2.0.5 会针对 cwd
   解析 `../Resources/*.json` 并缓存结果：导入时的 `PATHS` 与 `BuildConfig`，以及首次使用
   时的 `Utils.paths` 版本信息与 `Updater.getLocalInfo()`。启动时读取 `Utils.paths.userData`
   并创建更新控制器，把它的首次 `getLocalInfo()` 与其状态订阅保留在同一个 continuation 中。
   之后才 `chdir` 到用户 home，在所有平台上、且在 server 导入之前都如此，这样后续每个
   子进程都在 bundle 之外启动。这覆盖宿主拥有的 spawn、进程内扩展、MCP 服务器与编辑器。
   失败的 `chdir` 会被记录，启动继续。此处的更新元数据失败先于宿主获取，直接退出。除非
   设置了中立的或隐藏的产物模式，启动随后打开一个小型、居中、无边框的 splash。
   - 它从保存的原生标签（`desktop.splash.*`，桌面填入 `{{version}}`）读取 “Starting
     ThinkRail…”，或在更新日志待处理时读取 “Finishing update to X…”。
   - 其内容是在创建时给出的 `data:` URL。SDK 会把 `html:` 内容推迟到 100 ms 定时器之后，
     而同步的 server 导入会阻塞它。启动在该导入之前 await splash 自身的 `dom-ready`，
     上限 500 ms。
   - 它在主窗口首次到达 `dom-ready` 时关闭，主窗口被激活。
   - 在启动错误对话框之前它被隐藏，而不是关闭：作为最后一个窗口，它会在对话框出现之前
     触发 `exitOnLastWindowClosed`。
   - 在主窗口存在之前关闭 splash 会退出 ThinkRail。在主窗口就绪之前关闭它也会关闭 splash，
     因此最后一个窗口仍然消失，退出经由协调器执行。用户以原生方式关闭的 splash 不再被
     关闭。
   - 主窗口、RPC、导航规则与首 `dom-ready` 就绪判定不变。提前在 splash 页面创建主窗口被
     否决：那会把依赖 origin 的导航、就绪判定与宿主绑定的 RPC 提前到宿主启动之前。
3. 在 `chdir` 之后立即启动 `resolveShellEnv()`，使它 spawn 的登录 shell 也在 bundle 之外
   启动，然后动态导入单独构建、未打包的 `server-runtime.ts` 资源，并在
   `startDesktopHost()` 之前 await shell env。Dock/Finder 启动要支付登录 shell 探测（典型
   配置档上约 100 ms，慢速时数秒）；把它与 splash 和 bundle 导入重叠，最多可隐藏这段
   时间。bundle 自有的 `shellEnv` 副本是独立的模块实例，因此这次运行缓存在一个
   `Symbol.for` 全局 key 上：它的 `bootHost()` 调用 await 同一次运行，而不重新探测。该
   bundle 以 pi 的 `PI_BUNDLED_NODE=true` define 构建，这使 PI 对外部扩展（如 Central）
   使用其内嵌模块的扩展加载器（静态 jiti，内置 Babel，外加虚拟模块）。没有该 define，
   单文件 bundle 会被当作普通 Node 运行时，PI 的惰性 Babel `require` 无法在其中解析
   （[[submodule-server-agent]] 拥有该接缝）。把 PI 展平进 Electrobun 的常规入口仍被禁止：
   那会在 `BUN_PTY_LIB` 设置之前加载 `bun-pty`。
4. 运行时值导入三个内置 pi-package 工厂（`pi-spec-graph`、`pi-thinkrail-workflow`、
   `pi-todos`），并以这些工厂、暂存的 skills（每个 `BuildRuntimeSources.skillRoots` 条目按
   其 `route` 复制进同一个目录 —— 宿主根放在 `host/` 下，注册表扩展的放在
   `extensions/<name>/` 下 —— 重复 route 被拒绝）以及 macOS/Windows trash 辅助文件调用
   `registerBundledRuntime()`。来自服务端注册表的 ThinkRail 扩展
   （[[submodule-server-extensions]]）不需要生成入口：它们的工厂通过 `@thinkrail/server`
   的静态导入图抵达 bundle。生成器的 key 映射必须满足服务端拥有的 `BundledExtensions`
   契约的每个 key，因此新增一个必需的启动器字段会让桌面 typecheck 失败，而不是产出一个
   仅打包时出现的 `undefined`。随后它在 loopback 端口 `0` 上调用
   `bootHost({ resolveShellEnv: true, … })`，带上暂存的 web 目录、烧录版本与仅发布用的
   analytics key、`desktop` analytics 来源，并且仅在打包时带上启动器提供的
   `Utils.openExternal` 回调，供宿主的一次性浏览器归因声明使用。该回调经
   `DesktopHostOptions` 与生成的运行时传递。首个原生窗口 `dom-ready` 显式调用代理的
   `host.server.startAttributionClaim()` 就绪方法；宿主启动与已流逝时间不会启动一次保存
   选择的声明。不新增 deep-link 或 RPC 接口。运行时还传入 `appRoot`（`runtimeDir/..`，即
   `Resources/app`），使错误报告帧以打包 app 为根，并把 `reportDesktopError` 作为
   `DesktopHost.reportError` 返回。
5. 为 `{ backendProfileId: "local", windowId: "main" }` 恢复有效的路由片段与有界客户端偏好
   映射。路由被附加到新 origin 上；偏好映射被序列化为数据并前置到 preload 源码，使 web
   客户端在端口变化的情况下仍能在 React 挂载前完成水合。用系统渲染器打开一个普通原生
   `BrowserWindow`。

运行时 bundle 用 `--outdir` + `--entry-naming` 构建（Bun 拒绝 `--outfile` 配外部 map）。
只有在带凭据的发布构建（`THINKRAIL_SOURCEMAPS_RELEASE`，[[module-ci-release]]）中，
`preBuild` 才输出外部 map、自行注入 PostHog chunk id、通过根 `posthog-cli` 脚本（web 包中
校验过校验和的 runner）用 `posthog-cli sourcemap upload` 上传 `.js` 拷贝 + map，然后删除
map：posthog-cli 自己的 `inject` 会跳过 `.ts` 文件，并会把 Bun 的 `// @bun` pragma 推到
第 1 行之外，于是注册语句变成第 2 行、map 偏移一行。chunk id 是一个由发布版、代码**与**
其 map 哈希出的 UUID：Bun 对不同的检出路径（生成入口的绝对导入只存在于
`sourcesContent` 中）或仅注释的源码改动会输出字节相同的 JS，而 posthog-cli 的符号集上传
在已有 chunk id 但内容不同时会失败。`preBuild` 还拒绝暂存 web 的 `.map` 文件。

携带错误的有处理的 native-shell 失败（键盘状态、标题栏双击、更新监听器/生命周期/日志、
原生标签写入、离开 bundle 工作目录、Windows 边框恢复、Windows 窗口图标、Windows 任务栏
身份、启动）都经由 `errorReporting.ts` 的 `desktopError(scope, message, error)`：
`[desktop] …` 控制台行保留，宿主启动后该错误作为已净化的 `desktop` 错误报告转发给
`DesktopHost.reportError`（[[submodule-server-analytics]]）；在此之前仅进控制台。未捕获的
主进程故障不需要接缝 —— 它们共享宿主的进程处理器。`updates` 直接抵达这个同级辅助函数。

Electrobun 入口 bundle 只包含原生外壳代码。其中静态导入 server 被禁止：它可能在
`BUN_PTY_LIB` 之前加载 `bun-pty`，并绕过已定义的 server-runtime 构建。启动失败经共享崩溃
路径记录，显示在原生错误对话框中，并在不留下隐藏宿主的情况下退出。

打包资源保持为物理的、未打包的：web 资源与 skills 通过文件系统路径读取，PTY 使用 FFI，
trash 辅助文件是可执行的 sidecar，preload 以源文本读取。ASAR 不属于本设计。

## 原生应用菜单（Native application menu）

外壳在创建主窗口之前安装 Electrobun 的原生应用菜单。标准编辑命令由原生 role 而不是
浏览器级按键处理器掌管，因此 Command/Ctrl-C、V、X、A、Z 与 Shift-Z 在普通输入框、
Monaco、xterm 以及未来的 webview 界面中经操作系统响应链传递，且不与其本地按键处理竞争。

macOS 得到惯例的 application、Edit 与 Window role 菜单。Windows 得到受支持的 Edit role
菜单。Linux 跳过注册，因为 Electrobun 2.0.5 在那里不支持应用菜单；WebKitGTK 保留其渲染器
原生的编辑行为。macOS 的 Command-H、Option-Command-H 与 Command-M 带有显式快捷键，因为
role 项没有默认快捷键。Settings…（Command-逗号，排在 About 之后）、Quit 与 Close 是自定义
动作项，不是 role；见已确认的退出与关闭。它们的标签，以及 Edit 与 Window 菜单标题，是
[[module-locales]] 的 `DESKTOP_MENU_KEYS`；原生错误对话框（启动失败、重启以更新失败）与
启动 splash 使用其 `DESKTOP_DIALOG_KEYS`。web 通过 preload 的
`__THINKRAIL_NATIVE_MENU__` 桥（`menuLabelsChanged`）推送两者的当前语言标签；主进程净化
每个标签（非空白、菜单项 ≤ 80 字符、对话框文本 ≤ 200 字符，否则回退英文），重装菜单，
并在中立 e2e 宿主之外把它们保存到 `<userData>/native-labels.json`。下一次启动用保存的
标签安装菜单，而启动失败（可能发生在 web 加载之前）从该文件读取其对话框文本（首次运行
或任何读取错误时用英文）；桌面自身从不解析语言。role 项保留操作系统提供的标签。该策略
是平台纯的，打包就绪接缝报告注册是否运行过，因此单元测试钉住菜单组合，而展开 app 的
smoke 钉住生产接线。

### 已确认的退出与关闭（Confirmed quit and close）

Command-Q（`quit-shortcut`）必须被确认：按住 1200 ms，然后松开 Q 或 Command；或在松开
第一次按下后 500 ms 内再按一次，然后松开。单次轻按会静默过期。按住和弦期间键盘退出从不
触发，因此按键重复无法把 Command-Q 泄漏到下一个应用。Electrobun 在每次按下和每次操作
系统按键重复时都触发菜单动作且没有重复标志，因此 `quitShortcut` 驱动共享 contracts 的
`createQuitConfirmation` 规则（该手势的唯一副本，web shell 在 Linux 上的 Ctrl+Q 也使用
它），并向它喂入来自 CoreGraphics 的按键状态（`keyState.ts`，Bun FFI，HID 系统状态，
无需额外权限），在接触期间轮询它。按住只统计动作触发时处于按下状态的非修饰键，因为菜单
按字符匹配 Q，而其物理键位随布局不同（AZERTY、Dvorak）；按键重复的按下保持该集合且从不
向其中添加，因此较晚按下的键从不计入按住。菜单点击以同一动作到达，但没有按键按下，且有
晚于最后一次 key-down 的左键 mouse-up，因此它直接退出。键盘或 VoiceOver 菜单选择
（Ctrl+F2、Enter；VO+Space）到达时 Command 已抬起且其最后一次 key-down 晚于最后一次修饰键
变化，因此也直接退出。迟到处理的轻按则会武装：它的 Command 抬起是晚于 Q key-down 的修饰
键变化。迟到处理的按下若 Command 已抬起但 Q 仍按下，可能被当作菜单选择通过；处理耗时
毫秒级，因此这一点被接受。按下时按键状态不可读则直接退出；武装期间不可读则取消，除非已
到达松开。没有可见提示时（窗口最小化，或宿主在窗口与 RPC 存在之前启动），按下会跳过确认
并在松开时退出。菜单监听器紧接菜单之后挂上，因此启动期间退出即可用；在窗口存在之前命令
被丢弃。每次已确认退出都经由退出协调器。Dock Quit、Apple Event 退出与更新重启绕过菜单并
保持直接。提示以 `quitHintChanged` 消息传递。退出保持在命令通道之外：主进程拥有它，它
需要按键状态，且必须在 webview 挂起时仍可用。

在 Linux 上没有应用菜单，因此 web shell 掌管 Ctrl+Q：它从 keydown/keyup 确认同一手势，并
调用 `NativeShortcutsBridge.quit()`，即 `quitApp` RPC 请求，它在不 await 关闭的情况下启动
同一退出协调器。Windows 没有绑定退出和弦；Alt+F4 仍是 OS 窗口关闭及其常规关闭流程。

其他菜单快捷键被转发为 `NativeCommand`：菜单动作就是命令 id（一个穷尽的
`Record<NativeCommand, true>` 把关），主进程发送 `nativeCommand`。共享的 `repeatGate` 使
它们变为边沿触发：在一次带按键按下的转发之后，后续动作被丢弃直到没有非修饰键按下，因此
按键重复无法把命令触发两次。没有按键按下的点击，或不可读的按键状态，总是转发。Command-W
（`close-item`）请求 web 客户端关闭其聚焦项或什么都不做；它永不关闭原生窗口。Command-逗号
（`open-settings`）请求它打开 Settings；由 web 决定模态是否阻止它。preload 在每个桌面平台
上把提示与命令暴露为冻结的不可枚举 `__THINKRAIL_NATIVE_SHORTCUTS__`
`NativeShortcutsBridge`（由 `shortcutsBridge.ts` 构建）；如今只有 macOS 菜单发送事件。
在 Windows 与 Linux 上，web shell 通过其命令表掌管 Ctrl+W / Ctrl+F4（关闭项）与 Ctrl+,
（Settings）。

preload 还在每个桌面平台上暴露冻结的不可枚举 `__THINKRAIL_NATIVE_NOTIFICATIONS__`
`NativeNotificationBridge`（[[module-contracts]]）。它有两个能力：
`show({ title, subtitle?, body, silent? })` 把 `showNotification` RPC 请求转发给主进程，
后者调用 Electrobun `Utils.showNotification`（原生 OS 通知中心）；投递是尽力而为的，因此
失败的请求被丢弃，而不会浮出未处理的 rejection。`isWindowFocused()` 返回原生窗口是否持有
OS 级焦点；主进程通过 `BrowserWindow` 的 `focus`/`blur` 事件跟踪它，并通过
`windowFocusChanged` RPC 消息把状态推给渲染器，preload 同步缓存它。当 DevTools 或窗口内
其他 chrome 持有渲染器焦点时，这能正确报告 `true`，不同于此情况下返回 `false` 的
`document.hasFocus()`。**平台注意事项：** 在 Windows 上，DevTools 是独立的原生窗口；聚焦
它会触发主 `BrowserWindow` 的 `blur`，因此 `isWindowFocused()` 退化为与浏览器通道相同的
局限。在 macOS 上 DevTools 是停靠的，主窗口焦点得以保留。Electrobun 2.0.5 没有应用级
激活信号；跨窗口方案需要未来的 API。web 客户端按能力检测该桥，并让其注意通知经由该桥
而不是页面 Notifications API —— web shell 不保留桌面分支。Electrobun 的点击回调尚未落地，
因此原生通知不带点击动作。

每个和弦恰好有一个所有者：原生菜单快捷键（转发为 `NativeCommand`）或 web keydown，绝不
同时两者，因为 webview 仍会收到快捷键和弦的 keydown。原生拥有 OS 或浏览器保留的和弦
（Command-Q/W/H/M）；web 拥有没有原生菜单认领的 Ctrl+Q/W/F4。

## 原生页面缩放（Native page zoom）

Command/Ctrl-`+`（`=`）、`-` 与 `0` 像浏览器默认动作一样缩放：桌面 preload 在页面处理器
之后监听，并把处理器已用 `preventDefault` 认领的任何和弦让出（Monaco 的 `Mod+K` 折叠和弦
以这些键结束），然后向主进程发送一条有类型的单向请求。只停止传播的处理器同样保留该和弦，
因此当这样的输入获得焦点时缩放是惰性的。主进程从 webview 当前的缩放步进到相邻的有界
浏览器式倍率，因此经其他途径（WebView2 的 Ctrl+滚轮）改变的缩放不会造成失步。

在 macOS 上，WebKit 的双指捏合生命周期持续驱动同一原生页面缩放。preload 让出被页面内容
认领的手势并转发经过校验的倍率；主进程在手势开始时捕获原生缩放，并以该基线应用每个绝对
倍率，限定在 50%–200%，因此更新既不叠加也不引入第二个缩放所有者。web 应用不拥有任何
重复的快捷键、手势或缩放状态。

## 原生窗口外壳（Native window chrome）

web 顶栏（[[submodule-web-shell]]）就是窗口的标题栏，只要框架允许原生控件在没有原生条带
的情况下存活。Electrobun 2.0.5 在任何平台上都不暴露标题栏颜色 API（macOS 与 Windows 只
跟随系统明暗设置），因此主题色的标题栏意味着移除原生条带并让 web 头部占据它。该策略是
平台纯的（`windowChrome.ts`，由单元测试钉住）并展开进主 `BrowserWindow`：

| 平台 | `titleBarStyle` | 原生控件 | 发布的内边距 | 头部拖动 |
|---|---|---|---|---|
| macOS | `hiddenInset` —— 透明条带、隐藏标题、全尺寸内容视图 | AppKit 交通灯按钮保留；`trafficLightOffset { x: 0, y: 4 }` 把 12px 按钮居中在 40px 顶栏中（在打包构建上实测：默认位置使它们中心在 16px，因此 +4 落到 20） | 左 `64px` —— 交通灯区域（按间距网格为 7 + 3×12 + 2×8 = 59px） | 是 |
| Windows | `hiddenInset` —— Electrobun 创建 `WS_CAPTION \| WS_THICKFRAME` 并在 `WM_NCCALCSIZE` 中去掉标题区：调整边框与 DWM 阴影保留，没有原生标题按钮 | 无原生；web 渲染最小化 / 最大化或还原 / 关闭（[[submodule-web-shell]]）并通过 RPC 驱动窗口 | 右 `138px` —— 三个 46px 标题按钮 | 是 |
| Linux | `default` | GTK/WM 装饰 | 无 | 否 |

**Windows。** Electrobun 2.0.5 已启用 WebView2 的非客户端区支持，因此头部的
`app-region: drag` 是真实的 `HTCAPTION` 命中：Aero Snap、从最大化拖动、双击最大化与
Alt+Space 都来自 OS，而不是 Electrobun 的原始输入 mover。Electrobun 在其 `hiddenInset`
样式中省略的是 `WS_SYSMENU | WS_MINIMIZEBOX | WS_MAXIMIZEBOX`（上游
blackboardsh/electrobun#558）：没有它们，最大化会盖住任务栏，双击/系统菜单行为也失效。
在该修复落地上游之前，`windowsFrame.ts` 在窗口创建后通过 `dlopen` 的 `user32.dll`
（`GetWindowLongPtrW` → `SetWindowLongPtrW` → `SetWindowPos(SWP_FRAMECHANGED)`）把这三个位
OR 到 HWND 上一次 —— 可注入 API 背后的纯样式算术，无 C 源码、无运行时编译；失败被记录，
从不致命。成功与否靠回读样式判定，而不是 `SetWindowLongPtrW` 的返回值：0 的前一值有歧义，
而其 `SetLastError`/`GetLastError` 消歧会跨越多个运行时可能覆写的 FFI 调用。HTML 控件
调用 `minimizeWindow` / `toggleMaximizeWindow` / `closeWindow` 请求；关闭经由
`requestClose()`，与原生 X 同一条路径，因此退出协调器仍拥有关闭。主进程在每次
`dom-ready` 以及变化时的 `resize` 上发布 `windowStateChanged { maximized, fullScreen }`，
preload 仅在注入的种子表明 `windowControls` 时暴露 `NativeWindowControlsBridge`
（`__THINKRAIL_NATIVE_WINDOW_CONTROLS__`，[[module-contracts]]）。已知缺口，接受而非用窗口
子类打补丁：上边缘不是调整大小命中区（Electrobun 把它并入客户区，#558 同源），Windows 11
Snap Layouts 不会在悬停 HTML 最大化按钮时出现（那需要 `HTMAXBUTTON`，没有 WebView2 的
预览版窗口控件覆盖层就无从提供）。PR #469 探索过通过运行时编译的 HWND 子类和 WebView2
区域遮罩保留 DWM 自有的按钮；因验证过于沉重而被搁置。

Linux 不在计划内：`hiddenInset` 在 GTK 下是空操作，而 `hidden` 会连同 WM 提供的调整手柄
一起去掉装饰。完全自定义的 macOS 外壳（`titleBarStyle: "hidden"` 并自绘交通灯）被否决，
因为平台提供了真实的按钮。

**几何契约。** 桌面在 `<html>` 上发布三个 CSS 自定义属性：两个内边距属性加上
`--window-chrome-drag-region`（`drag` | `no-drag`），后者告诉页面其头部是否为标题栏。
拖动标志只随初始 preload 全局量传递（它是按策略的常量）；几何更新以
`windowChromeChanged { insetLeft, insetRight }` 到达。web 以 `0px` 回退消费这些属性，因此
浏览器托管的客户端与中立 E2E 宿主窗口（`about:blank`，无 preload）不受影响，且 *web 客户端
仍然没有桌面分支*。preload 校验几何，在文档根缺席时只保留最新值，并在 `DOMContentLoaded`
时刷出，因此一次待处理的启动写入无法覆盖更新的原生状态。右内边距在 Windows 上是
`138px`、其他平台是 `0px`；种子还携带按策略的 `windowControls` 标志，preload 把它转成
可选的控件桥。

**全屏。** macOS 原生全屏会随菜单栏自动隐藏交通灯，Windows 全屏窗口是裸的 `WS_POPUP`，
因此全屏把两个内边距归零，而 web 在 `windowStateChanged` 报告 `fullScreen` 时隐藏其控件。
主进程在每次 webview `dom-ready` 时发布几何（全屏期间的重载不得继承窗口化内边距），并且
只在几何变化时于 `resize` 发布。中立 E2E 宿主窗口（`THINKRAIL_DESKTOP_E2E_HOST=1`）保留
默认原生外壳且不发布任何内容。

**拖动。** 桌面的唯一贡献是 `drag` 标志：web 头部的 `window-drag` 工具仅在
`--window-chrome-drag-region` 如此声明时解析为 `-webkit-app-region: drag`，而 Electrobun
自注入的 preload 会把同源样式表中的 app-region 声明重写成一个镜像的自定义属性，在
`mousedown` 时对它做命中测试。带装饰的窗口（Linux、中立窗口）发布 `no-drag`，因此绝不会
获得第二条局部拖动条带。

**标题栏双击。** webview 覆盖原生条带，因此 `NSWindow` 永远看不到头部双击，而
Electrobun 2.0.5 只实现 app-region 拖动。其原生移动使用穿透式事件监视器，因此页面仍会
收到 `dblclick`。桌面 preload 监听页面未 `preventDefault` 的主键 `dblclick` 事件，并在
目标按 Electrobun 自己的命中测试输入（其样式表重写产生的、计算后继承的
`--electrobun-app-region` 属性）属于拖动区域时，发送一条无载荷的 `titleBarDoubleClick`
消息。因此恰好是拖动窗口的区域也会缩放窗口；`window-no-drag` 动作簇两者皆无，而 web
客户端仍没有桌面分支。主进程仅在 macOS 策略的 `titleBarDoubleClick` 标志为真且不在原生
全屏时动作；Windows 关闭它，因为其 `HTCAPTION` 拖动区域已能原生最大化，再处理一次会切换
两次。它在每次双击时读取 `AppleActionOnDoubleClick`（异步 `defaults read -g`，因此改变
设置无需重启即生效），并像 Chromium 对自定义可拖动区域那样映射：未设置、`Maximize`（Zoom）
或 `Fill` 经 `maximize()`/`unmaximize()`（`[NSWindow zoom:]`）切换缩放；`Minimize` 最小化；
`None` 与未知值不做任何事。`Fill` 使用 zoom，因为 Apple 的 `_zoomFill:` 是私有的且无法经
Electrobun 触达；该窗口的标准缩放帧就是屏幕的可见帧，因此只有平铺边距不同。前一次双击
仍在解析时到达的双击被丢弃。该变通存在只因 Electrobun 缺少该行为，上游追踪于
[electrobun#561](https://github.com/blackboardsh/electrobun/issues/561)。升级 Electrobun
时必须重新检查镜像属性名与该 issue：一旦框架自行处理拖动区域双击，就移除 preload 监听器、
`titleBarDoubleClick` 消息与处理器，否则窗口会切换两次。

## 导航与窗口安全（Navigation and window security）

原生窗口只允许在其精确的 loopback origin 内导航。用户请求的外部 URL 经 OS 打开，而不是
替换应用界面。导航监听器使用 SDK emitter 的 webview 作用域通道 `will-navigate-<id>` 与
`new-window-open-<id>`；无作用域的载荷没有 webview id，而实例监听器的有类型事件列表省略
弹窗。载荷类型源自 SDK 事件工厂，而非复制到本地声明。Detail 可以是原始 URL、弹窗对象或
序列化的导航 JSON；有界解码只保留字符串 URL 与 HTTP/HTTPS/mailto allowlist。原生
`navigationRules` 强制限制：导航事件响应无法取消它。

桌面 preload 发送有类型的单向路由、本地偏好、页面缩放与标题栏双击消息，并且是上述窗口外壳
CSS 属性的唯一写入者。它在页面脚本之前包装 `history.replaceState` 与 `history.pushState`，
也报告初始/hash/pop 导航，因为 Electrobun 的原生导航事件不观察 History API 路由变化。
主进程只接受来自主窗口的消息。路由以有界片段字符串持久化在按 channel 作用域的版本化
文档中；不可读/无效状态回退到 `#/v1`。偏好持久化在另一个按 `{ backendProfileId, windowId }`
作用域的有界、版本化通用字符串映射中；每个值上限 256 Ki 字符，整份文档上限 1 MiB。原生侧
只校验大小/形状，从不了解功能含义。其冻结的 preload 适配器只暴露 `getItem`、`setItem` 与
`removeItem`，写入经有类型的单向通道回传。格式错误的消息被忽略；文件系统拒绝被记录，而
不改变内存文档或终止客户端。web 功能仍拥有每个值的校验与默认值。web 路由仍是路由文法的
校验者，preload 不暴露任何宿主/域能力。

webview 到主进程的 RPC 在每个平台都使用 Electrobun 的默认 loopback WebSocket。复盘：
Electrobun 2.0.1 在 Windows 上以地址复用绑定其固定默认端口 `50000`，因此第二个正在运行的
Electrobun 应用（包括另一个 ThinkRail channel）共享该端口并收到本 webview 的 socket 流量；
每个请求都超时，每条消息都丢失。启动器当时从 Windows preload 中剥离注入的 socket 端口
全局量以强制使用原生宿主桥。Electrobun 2.0.5 的核心在 Windows 上以 `SO_EXCLUSIVEADDRUSE`
绑定宿主传输层并移向下一个空闲端口，注入它实际绑定的端口，因此该变通已移除。

宿主读取暂存的 preload bundle，并把其 JavaScript **源文本**传给 `BrowserWindow.preload`。
`views://` preload URL 被禁止：Electrobun 1.18.1 在 macOS 上能解析它，但在 Linux 上会把
字面 URL 当代码注入。

## 生命周期（Lifecycle）

每条退出路径都调用一次共享的幂等异步服务端关闭，包括在获取宿主之后、但在原生窗口或退出
监听器就绪之前的启动失败。关闭所有权在宿主被获取的那一刻开始。它在其界限内结束/中止活动
agent 工作、排空 analytics、释放服务端资源与 PTY，并关闭 socket。Electrobun 的同步
`before-quit` 回调在该 promise 挂起期间取消退出，并在完成守卫下重试 `Utils.quit()`。启动
失败也在错误对话框终结器中直接经该协调器请求退出，因此即使退出拦截缺失或对话框失败，也
无法绕过关闭。突然死亡只依赖操作系统进程清理。

产物测试通过可选的环境/就绪/控制接缝驱动同一入口点：隔离的用户数据、供浏览器后端测试用的
隐藏中立窗口、DOM-ready 时的宿主/启动器 id 与 origin，以及正常退出。原生 UI smoke 可以
捕获一次外部打开结果，并请求一次固定的导航探针，而不是启动用户的浏览器。给定标题栏探针
文件时，两条控制命令在活 webview 中派发合成 `dblclick`：`title-bar-double-click` 打在
头部，而 `title-bar-double-click-no-drag` 先打在 no-drag 动作簇，然后在一个任务内打在
头部。启动器记录它收到并处理了多少条双击消息，以及最后处理到的偏好、动作与前后窗口状态。
仅头部的阶段证明头部被转发；同时记录接收与处理，使第二阶段能检出被单飞处理器丢弃的已转发
no-drag 点击。给定窗口控件探针文件时，`window-controls-maximize` /
`window-controls-restore` 在 HTML 标题按钮带上预期标签后点击它，启动器记录它处理的每个
窗口控件请求以及最后发布的 `NativeWindowState`。这些钩子需要活窗口；它们独立的驱动器与
断言位于测试包中，产品代码从不导入。

## 构建与发布（Build and release）

该包把 Electrobun `2.0.5` 的 npm bootstrap 钉为仅构建期依赖。该精确 pin 选择其配套的
Hutch `0.28.2` 工具链与 SDK；直接全局调用 Hutch 与浮动版本覆盖不属于构建路径。该 pin 是
发布 macOS x64 core 的最低发行版，因此 Intel Mac 与其他目标一样经同一条未经修改的
`electrobun build` 路径构建；两个 macOS 架构都以 macOS 14 或更高为目标，且只在原生
runner 上构建。应用显式选择 Bun 主进程（`build.mainProcess: "bun"`，Electrobun 自 2.0.4
起的默认值）。打包的 Bun 是 `1.4.4`，由 `build.bun.version` 钉住，尽管它等于 Electrobun
2.0.5 的默认值，这样框架升级无法静默移动运行时。Hutch 像解析其他工具链版本那样解析该
字段（`configuredToolchainVersion`），但 `ElectrobunConfig` 不为它定型，因此配置把 Bun 块
作为命名值传入。`buildConfig.test.ts` 断言它等于根 `packageManager` pin，这使打包运行时
与 [[architecture]] 的根工具链契约保持一致。Electrobun 自己的套件只在其默认 Bun 上运行；
当该默认值与 pin 分歧时，桌面 smoke 与 e2e 就是该运行时的门禁。

该包运行官方的 `electrobun build` / `dev` 命令。配置读取与启动器相同的共享版本模块，没有
环境版本桥。一个已文档化的 `preBuild` 钩子构建共享 web 产物并暂存应用特有的
PTY/trash/skill 资源与 PI 运行时。该钩子调用真实的 Bun CLI，既用于读取
`@thinkrail/server/build-support` 清单（它会值加载服务端扩展注册表以及随之而来的 pi 图 ——
Cottontail 的模块运行时无法求值该图；`typebox` 的 `String` 导出是第一个牺牲品），也用于
打包单独暂存的 `.ts` 服务端运行时，而不是改动 PI 的打包器。这种拆分源自 Hutch 在
Cottontail 下运行钩子的时代；Hutch 0.28.2 在 Bun 下运行它们，因此这些子 Bun 进程如今是
可移除的而非必需的。其暂态工厂入口即使在失败时也会被移除。暂存资源包括内置 skills 所
消费的工作流 SPEC。一个已文档化的 `postBuild` 钩子在该框架复制完暂存内容后移除暂存；
失败构建的暂存会在下一次 pre-build 时被替换。在 Windows 上该钩子还在 Hutch 嵌入图标之后、
发布压缩、包装与签名之前为打包的可执行文件盖印品牌（见下）。同一工作树中的构建保持串行。

Electrobun 的平台图标配置指向 `packages/shared/assets` 中的原生图标集：macOS 图标集、
Windows 多分辨率 ICO 与 Linux PNG。[[module-shared]] 拥有它们的图稿、生成器与帧尺寸，而
CLI 的 Windows 二进制内嵌同一 ICO（[[module-cli]]）。浏览器 favicon 与应用内 shell 标记是
web 自有的独立界面。

在 Windows 上，Hutch 把 ICO 嵌入 setup/extractor 可执行文件与 `bin/launcher.exe`，快捷方式
与已安装应用条目指向启动器。没有发布 action 替换为第二个安装器图标。然而正在运行的应用是
`bin/bun.exe`，Electrobun 2.0.5 / Hutch 0.28.2 在那里留下三个缺口，由本项目弥合：

- **图标组。** Hutch 向 `bun.exe` 加入组 1，但 Bun 自有的命名 `IDI_MYICON` 仍在，而 shell
  会先选它。卸载器完全没有图标。
- **文件属性。** Hutch 不写 VERSIONINFO。`bun.exe` 保留 Bun 的（"Bun"/"Oven"），因此任务
  管理器把该应用列为 Bun；启动器与卸载器则没有。

  `postBuild` 弥合这两个缺口。它重写 `bin/bun.exe`、`bin/launcher.exe` 与
  `Resources/uninstall` 的资源：每个图标条目被替换为一个含全部帧的组 1，VERSIONINFO 来自
  [[module-shared]] 的 `windowsFileIdentity`。重写会丢弃 `bun.exe` 现已无效的 Oven 证书
  条目；发布签名随后覆盖每个 PE（[[module-ci-release]]）。Hutch 的 Win32 资源更新已经丢弃
  签名字节但保留其目录条目，该条目随后指向文件末尾之外。因此品牌化在解析前清除该条目并
  截掉任何签名字节，而不是依赖 PE 库的已签名文件绕过（它仍会读到该偏移）。Windows PR CI
  会构建并 smoke 桌面应用，因此品牌缺陷会让 PR 失败（[[module-ci-release]]）。
- **窗口图标。** Electrobun 注册其窗口类时不带图标，从不发送 `WM_SETICON`，其 Windows
  `setWindowIcon` 是空桩。因此启动时通过 `user32` 加载 `Resources/app.ico`（Hutch 对 ICO 的
  拷贝）并把它发给 splash 与主窗口，按窗口 DPI 设置尺寸。失败会被报告且不致命。

移除门禁：Hutch 写出可执行文件身份并替换主进程的图标组；Electrobun 在 Windows 上设置
窗口图标。

**任务栏身份。** 钉住的快捷方式指向启动器，而窗口属于 `bun.exe`，且 Electrobun 不设置
AppUserModelID。没有它，Windows 会从各自的 exe 推导每一方的身份，并把它们显示为两个
任务栏按钮（blackboardsh/electrobun#514）。因此桌面为自己命名：

- **进程 ID。** 启动做的第一件事（在任何窗口之前）是设置一个由产品名推导的显式按 channel
  ID：`JetBrains.ThinkRail`、`JetBrains.ThinkRail.Nightly`、`JetBrains.ThinkRail.Dev`。
  stable 与 nightly 保持为独立应用。
- **快捷方式 ID。** 主窗口首次到达 `dom-ready` 且就绪被报告之后，桌面把同一 ID 写到每个
  目标为本安装 `bin\launcher.exe` 的快捷方式上：
  - Electrobun 记录在 `<channel root>\.electrobun-uninstall.json` 中的开始菜单快捷方式
    （schema 1；其他 schema 被忽略）；
  - `User Pinned\TaskBar` 与 `User Pinned\ImplicitAppShortcuts` 中的任务栏固定项，使此次
    变更之前做出的固定项也能归组。

  桌面快捷方式不动。归组不需要它，而重写桌面快捷方式此前曾移动过它们的桌面图标
  （Firefox bug 633728）。

  `WScript.Shell`（Electrobun 使用的）无法设置该 ID，而 Electrobun 会在安装与更新时重写其
  快捷方式。因此盖印是幂等的，并在每次启动时运行，在 UI 加载之后，在一个只持续到盖印结束
  的 COM apartment 内。它如何工作：
  - 它通过 Bun FFI 使用 `ShellLink` + `IPropertyStore` COM 调用，Windows PR 作业会针对
    真实快捷方式执行它。PowerShell 的 `Add-Type` 辅助被否决：它每次启动都要编译 C#。
  - 候选的目标先按文件名比较，因此外来固定项的目标（可能在网络共享上）永不在宿主的线程上
    被解析。
  - 每个候选都以只读方式打开；只有指向本启动器且需要该 ID 的快捷方式才为写入重新打开，
    因此外来的只读或 ACL 受限固定项从不构成失败。
  - 只有 `S_OK` 才算写入成功。
  - 失败的 HRESULT 携带 `HRESULT_XXXXXXXX` 代码用于错误跟踪。
- **限制。** 指向其他目标的快捷方式永不被触碰。这包括此次变更之前从运行窗口做出的固定项，
  它们指向 `bun.exe` 并保持独立。失败会被报告且不致命。不可读的清单也会被报告，而固定项
  仍会被盖印。

移除门禁：Electrobun 为其进程与快捷方式提供共享的 AppUserModelID。

Electrobun 的 `build.views` 拥有浏览器 preload bundle；`build.copy` 拥有物理资源包含。
没有自定义 SDK 解析器、SDK 元数据校验器或 Electrobun 命令运行器。App 本地 Hutch 配置选择
Bun 作为包管理器，保留工作区 catalog 与 `bun.lock`。

Hutch 拥有生成的 `.hutch/devkit` 投影与下载缓存。该投影与暂态的 `.cottontail-tmp` 加载器
被忽略，并被排除在仓库源码边界扫描之外，永不编辑或提交。框架构建会隐式准备它；typecheck
在 TypeScript 之前运行标准 `electrobun prepare` 命令。准备错误正常传播，全新机器需要网络
访问。普通安装、仅 web 的开发/构建与单元测试不准备原生 SDK。

桌面 typecheck 通过与编辑器工具相同的、无 baseUrl 的路径消费官方 SDK 的 `.ts` 源码；不存在
手写的 API 声明或影子 typecheck 配置。经明确批准，仅桌面设置
`exactOptionalPropertyTypes: false` 以匹配 Electrobun 的源码契约，同时保留 `strict: true`
与 `noUncheckedIndexedAccess: true`。同样经明确批准并且是临时的，桌面设置
`noImplicitReturns: false`，因为 Electrobun 的浏览器与原生 SDK 源码无法通过它；其他基线
lint 风格标志保持开启。该标志无法只作用于 SDK 文件，因此它也停止检查桌面自身的源码，而
后者如今是通过的。等某个 Electrobun 版本发布
[blackboardsh/electrobun#590](https://github.com/blackboardsh/electrobun/pull/590) 后移除该
覆盖。这些设置作用于桌面编译，包括被导入的工作区源码；其他每个包保留其各自不变的 strict
检查。可选属性的不兼容追踪于 Electrobun issue #516；`skipLibCheck` 无法排除被导入的实现
`.ts`。直接源码方式遵循 v2 迁移指南，并避免第二套声明生成流水线。规范的主进程导入使用
`electrobun/main`；preload 保留 `electrobun/view`，RPC schema 保留其 `bun`/`webview` key。

参考：[官方 v2 迁移](https://framework.blackboard.sh/electrobun/guides/migrating-to-v2/)、
[上游可选属性 issue](https://github.com/blackboardsh/electrobun/issues/516)。

桌面安装器与 CLI 产物并排发布，覆盖 macOS ARM64、macOS x64、Windows x64、Linux x64 与
Linux ARM64。nightly 映射到 Electrobun canary，stable 映射到 stable。标准格式是 macOS 上的
DMG、Windows 上的 setup-EXE 加负载 ZIP，以及 Linux 上的 setup tar.gz。私有发布流水线保留
现有的 `thinkrail-desktop-*` 下载别名；其收集器为 channel 与原生目标选择确切的框架产物。
macOS app 归档被私下转移给 SRE 定稿，从不作为更新器载荷发布。运行时更新在下方集成；feed
发布仍归发布所有并受目标策略门禁。

### 签名（Signing）

签名与公证通过私有发布协调器使用 JetBrains 提供的服务；公开构建既不含服务凭据，也不含
新的 Apple 登录/钥匙串流程。Windows 覆盖签名 setup 存根及其相邻负载中的每个 PE 文件，因此
启动器、`bun.exe`、框架 DLL 以及安装并应用更新的内置 `uninstall` 辅助程序都携带 JetBrains
签名。私有流水线围绕已签名字节重新打包负载（[[module-ci-release]]）。

macOS 原生构建还把 Electrobun 标准的展开 `.app.tar.zst` 暴露为私有签名输入。经批准的
JetBrains SRE 流程对展开的 app 签名，并围绕它定稿一个常规 DMG，然后对该容器签名/公证/
装订。它不修改 Electrobun 的压缩自解包载荷，也不使用错误的元数据前装订钩子。本地框架 DMG
仍为未签名的构建输出；只有私有流水线中经过验证的最终 DMG 才是已签名发布版。交接与验证
契约属于 [[module-ci-release]]。

Linux 使用原生 WebKitGTK 而不使用 CEF，并声明 Ubuntu 24.04+/glibc 2.38 加上 `libgtk-3-0`、
`libwebkit2gtk-4.1-0`、`libayatana-appindicator3-1` 与 `librsvg2-2`。Xvfb 软件渲染标志仅
用于 CI，永不作为用户配置发布。

## 自动更新策略（Auto-update policy）

桌面更新在窗口就绪后检查，之后每六小时（带抖动）检查，但停在有可用发行版处。下载与安装
各自需要明确的用户动作；关闭 Settings 或正常退出不会安装任何东西。同版本与降级清单不合格。

生产检查只在打包的受支持 stable/canary 应用中启用，且其盖印的发布元数据提供非空 HTTPS
更新器 base URL。该元数据是唯一的 feed 权威；开发与标准产物测试接缝不暴露更新器。安装
保持在各自的打包 channel 上，而 UI 把 Electrobun 的 canary channel 称为 **nightly**。发布
资格属于 [[module-ci-release]]。

Electrobun 调用与生命周期状态留在桌面 `updates` 模块之后；控件留在 web 客户端，优雅的宿主
关闭留在 server。冻结的可选 `__THINKRAIL_NATIVE_UPDATES__` preload 桥暴露有类型状态、
check/download/install 动作与订阅，而无需把桌面 SDK 导入 web 代码。**Install & Restart**
通过现有退出协调器应用已备好的发行版；活动 agent 与 PTY 可能终止，没有第二次确认，也没有
渲染器准备握手。该桥还携带本次尝试的步骤与日志，以及重启后持久化的结果，由 web 呈现。
[[submodule-desktop-updates]] 拥有该尝试及其日志。

退出协调保留其完成动作。Electrobun 2.0.5 的首次 `applyUpdate()` 在异步 `before-quit` 否决
处返回，尚未武装其替换辅助程序。更新意图等待同一幂等宿主关闭，等待那次 SDK 调用落定，
然后在完成守卫下恢复 `applyUpdate()`；常规完成仍只调用 `Utils.quit()`。第二次交接失败会
报告原生错误并退出，而不是留下一个无服务端的窗口。SDK 替换回滚不是应用健康或用户数据
回滚。没有更新代码/feed 身份的现有安装需要一次手动引导安装。

该替换是一次目录重命名，而 Windows 拒绝在任何活进程的当前目录位于其中时重命名目录。在
启动离开 bundle（启动步骤 2）之前，长寿命子进程继承了 `app\bin`：从 pi `session_start` 启动
的 JetBrains Central 代理守护进程、编辑器、MCP 服务器。它们中任何一个都会让辅助程序的
`app → app.previous` 重命名在其整个重试窗口内以 `AccessDenied` 失败，然后回滚并重启旧
构建。POSIX 重命名不受影响。由没有该规则的构建启动的子进程保留旧 cwd，因此从这类构建做
首次更新时仍可能需要先把它们结束一次。该步骤是 Electrobun 2.0.5 的变通，带有两个移除
门禁。先读后 `chdir` 的顺序在 Electrobun 从可执行文件路径解析资源后即可去掉。`chdir`
本身在其启动器让主进程在可替换 bundle 之外启动后即可去掉。

参考：[钉住的更新器交接](https://github.com/blackboardsh/electrobun/blob/v2.0.5/package/src/sdks/main/core/Updater.ts#L2251-L2364)。

## 验证（Verification）

[[module-artifact-tests]] 拥有展开 app/首次安装 smoke、共享宿主探针、原生导航/preload 验证
与安装器隔离。每个原生发布目标都必须通过两层 smoke；[[module-browser-e2e]] 另外覆盖针对
打包宿主的 wire 后端行为。任何平台的原生结果都不从另一平台的运行推断。已签名下载验收需要
[[module-ci-release]] 中描述的私有发布检查。

## 推迟项（Deferred）

共享/远程 backend profile、profile 选择、多窗口/deep-link 路由、CEF，以及在
blackboardsh/electrobun#558 发布后移除 `user32` 样式位垫片。上述更新 feed 发布仍归发布所有。
