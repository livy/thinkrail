# updates —— 可选的应用更新能力

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

在本地原生更新权威或宿主拥有的 CLI 更新生命周期之上，拥有唯一一种应用更新呈现。本模块把这些能力归一
化为一个 shell hook、一个软件更新设置界面，以及一个持久的状态提示能力，而不让 UI 知道是哪个启动器在
运行。

## 边界（Boundary）

- **公开接口（`index.ts`）：** `useUpdates`、`UpdateSettings`、`UpdateReadyButton` 与
  `UpdateInstallSurfaces`。
- **外部依赖：** React、Remix Icon、`@thinkrail/contracts` 类型、兄弟 store barrel（宿主通知、toast）、
  `@thinkrail/ui` 的 `button`／`dialog`／`utils`，以及用于每条用户可见字符串的
  `@thinkrail/locales/react`。
- **禁止：** Electron/Electrobun 导入；server/shared/transport 导入；启动器名称检查；由渲染器选择的
  feed URL；更新检查／下载／安装逻辑；浏览器提供的 shell 命令、路径、通道、版本或 URL；草稿保存、
  输入保护、渲染器准备，或任何第二个原生重启同意步骤。

可选且冻结的 `__THINKRAIL_NATIVE_UPDATES__` 桥接仍是原生权威。它的适配器在首次读取之前就订阅，在一个
入口点接受单调修订，并用失败的阶段包装被拒绝的请求。原生快照保持 shell 本地，永不进入 Zustand。检查
为可用发布暴露 **Download**；传输进度会先成为一个独立的 **Preparing update** 阶段，之后就绪状态才暴露
**Install & Restart**。关闭 Settings 是推迟动作。禁用／来源的原生状态不暴露任何区段。UI 把打包的
`canary` 身份称作 **nightly**，而不改变 feed 选择。

可选宿主快照经带版本的 wire 到达，并在 store 中同步。它在一次临时断开期间保持可见，下一次 welcome 会
替换或清除它。当前宿主暴露 available/running/succeeded/failed 状态与一个 **Run Update** 动作；该动作
发送一个空的、带版本门控的请求，由宿主独自选择并运行其更新命令。成功之后仍要求手工重启宿主。较旧的
宿主保留固定的 `thinkrail update` 指引。没有任何命令文本或任意诊断信息跨过 wire。

存在原生桥接时它总是胜过宿主状态，包括在其能力正在解析或禁用期间。两者都不可用时，Updates 设置区段
与提示能力缺席。在整个可行动的原生与宿主阶段中，顶栏都保持为回到 Settings 的路径，而不只在包就绪之后。

## 原生安装旅程（Native install journey）

`UpdateInstallSurfaces` 由 Shell 挂载一次，是唯一推送 toast 或调用 `dismissInstallResult` 的部分。
这些对话框是 props 驱动的，且位于主 bundle 中。这一点很重要，因为关机期间宿主 socket 会关闭，而 RPC
桥接仍在工作，因此它们所用的任何东西都不能是惰性 chunk。所有措辞归 web 所有，位于
`packages/locales/en.json`（`updates.install`、`.log`、`.result`、`.recovery`、`.cause`）；desktop
只发送数据。
- **处于 `installing` 时：** 一个无法关闭的对话框。它把四个步骤（prepared、stopping、handoff、
  restarting）显示为确定性进度条，事前说明 ThinkRail 会关闭并重新打开，并提供一个带 **Copy log** 的
  日志控制台。控制台会翻译每个结构化日志事件与来源标签；只有持有原始 SDK 或助手错误的 `diagnostic`
  事件原样出现。它没有 Cancel：Install & Restart 就是同意，退出保持协调。一旦状态离开 `installing`
  它就关闭，因此错误永不把用户困住。
- **`lastInstall: installed`：** 一个带 **Details** 的成功 toast，随后立即消失。Details 从 web 自己的
  快照打开，显示已保存的日志与 **What's new**（`releases/tag/v<version>`）。
- **`failed`／`unreported`：** 一个失败对话框、Settings 里的提示与顶栏 chip "Update didn't install"
  会保留到被关闭或重试。
  - 原因来自纯函数 `installFailureCopy`。它是平台中立的，因为本模块不检测平台，而原始安装器文本只出现
    在控制台中。
  - 恢复跟随实时状态，而不是假定包在回滚之后仍存活：`ready` 时 **Try again**，`available` 时
    **Download again**，其他情况 **Check again**，工作运行期间显示禁用的进度标签。**Later** 关闭。
