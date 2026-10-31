# i18n —— web 客户端中的活动语言与本地目录

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

把同步来的 `AppConfig.locale` 偏好、浏览器的语言、随附的目录，以及宿主的本地文件
（[[submodule-server-locales]]）汇聚成共享 i18next 实例（[[module-locales]] `./react`）的唯一活动
语言。组件永不自己做这件事：它们调用 `useT()`。

## 边界（Boundary）

- **水合：** `transport` 在每次来自 v89+ 宿主的 welcome 之后、以及每次 `locales.changed` 提醒时读取
  `locales.list`，并用 `installLocalLocales` 安装该快照（较旧的宿主安装 `null`，因此只有随附的语言
  存在）。本模块只从 store 推导。
- **拥有：** `languageOptions` 推导（System + 每个可用 tag 的自身语言名称，经 `Intl.DisplayNames`，
  以及它的来源：`builtin`、`local`、`builtin+local`，或一个损坏／缺失的条目）、生效 tag 的推导
  （`resolveLocale` 遍历浏览器的 `navigator.languages`，在 `languagechange` 时重读），以及唯一的应用
  effect —— 安装覆盖层、切换语言、设置 `<html lang>`、经可选的 `__THINKRAIL_NATIVE_MENU__` 桥接
  推送桌面菜单与对话框标签（`pushNativeMenuLabels`）。
- **公开接口（barrel）：** `useLocaleSync()`（由 `main.tsx` 的 `App` 挂载一次）、
  `useSystemLanguages()`、`localeSignature`，以及纯函数 `languageOptions`、`effectiveLocale`、
  `availableLocales`、`localOverlays`。
- **允许的依赖：** `store`、`@thinkrail/locales`、`@thinkrail/contracts`（类型）。
- **禁止：** 渲染 UI（Language 页面位于 `panels`）、读取文件、第二个 i18next 实例。

## 必须做对（Get right）

- **只在真正变化时应用。** 该 effect 在 `localeSignature` —— 生效 tag 加上恰好其回退链的字符串 ——
  变化时才应用；一个无关文件被编辑或一次重复的 `locales.list` 都不会有动作。应用会发布一个新的翻译器，
  因此每个经 `useT()` 渲染的组件都会重渲染并重算（[[module-locales]] § Render 把翻译器当作值来读取）。

- 英文在首帧之前同步安装，因此在宿主连接期间不会有任何东西渲染出 key；本地字符串稍后到达并就地重渲染。
- 一个被选中的本地 tag 消失时（文件被删除、如今损坏，或宿主没有 `LOCALES_PROTOCOL_VERSION`），界面
  渲染英文而偏好保持不变，等文件回来时再恢复。
- 日期与数字使用 `currentLocale()`，绝不用硬编码的 `"en-US"` 或裸浏览器默认值。
