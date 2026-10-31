# locales——UI 字符串目录与共享的 i18next 实例

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

ThinkRail UI 字符串以及每个语言来源都必须遵守的规则的唯一所有者（[[architecture]] 决策 23）。它发布
随包目录——`en.json` 是英文的权威来源，也是译者起步的文件，旁边每个 `<tag>.json` 都是一份译文——以及
用于校验目录、命名文件语言、挑选渲染语言的纯函数。其 `./react` 入口持有每个浏览器界面（`apps/web`、
`packages/ui`、扩展的 web 半边）渲染时所用的**唯一** i18next 实例，因此它们不会漂移到各自的实例上。

## 公开接口（Public surface）

- **根 `@thinkrail/locales`**（纯函数，浏览器与服务器安全，无 React）：
  - `EN`——英文目录；`MessageKey`——其键联合（复数后缀 `_one`/`_other`/… 折叠进基础键）。
  - `BUNDLED_LOCALES`——本次构建随包发布的标签（`en` 在前，然后是 `BUNDLED` 注册表）以及
    `bundledCatalog(tag)`。文档和测试从它推导已发布列表而不是硬写，因此新增一个随包语言只需改动其
    目录、`BUNDLED` 条目以及 README 中的列表。
  - `SYSTEM_LOCALE`（`"system"`）、`canonicalLocale(tag)`、`languageName(tag)`（通过
    `Intl.DisplayNames` 得到该语言自称的名字，首字母大写）、`DESKTOP_MENU_KEYS` 和
    `DESKTOP_DIALOG_KEYS`——桌面端原生菜单、错误对话框和启动闪屏所渲染的键，桌面端与推送这些翻译的
    web 各有一份列表。
  - `localeTagFromFileName(name)`——`de.json` → `"de"`，经 `Intl.getCanonicalLocales` 规范化；
    非 `.json`、以点开头或非 BCP-47 的名字返回 `null`。
  - `validateCatalog(raw)`——`{ ok: true, strings } | { ok: false, error: "not-object" }`。该值必须
    是 JSON 对象。以下内容被静默丢弃：非字符串和空值（改为显示英文）；对某个已知键而言，其
    `{{placeholder}}` 集合或索引标记集合（`<0>`、`</0>`、`<1/>`）与英文不同的字符串（复数形式
    `key_<category>` 会与英文的 `key_other` 比较，而 `{{count}}` 在任何形式中都可以存在或缺席，因此
    译者可以显示或省略数字）；任何不是裸索引标记的标签（不允许属性、不允许具名标签——目录不得覆盖
    代码拥有的组件属性，例如链接的 `href`；标签用 react-i18next 自带的引号感知解析器模式查找，而
    `<` 后跟 `{{` 会被拒绝，使插值永远无法补全一个标签）。包含 `$t(` 嵌套的值会被丢弃。未知键作为
    数据保留（文件可能比应用更旧或更新）；键查找只使用自有属性，因此 `constructor` 或 `__proto__`
    都是普通键。随包的英文目录也通过同一套校验。
  - `completePlurals(tag, strings)`——对目录所翻译的每个英文复数族，补齐该 `tag` 需要
    （`Intl.PluralRules`）但缺失的每个 CLDR 类别，取自该族的 `_other`（否则取任一已存在的形式），
    使一个部分翻译的计数器保持在其语言中，而不是回退到英文。`applyLocale` 在每次叠加时运行它。
  - `resolveLocale(preference, available, systemLanguages)`——`"system"` 会遍历
    `systemLanguages`，取第一个精确可用的标签，否则取其基础语言，再否则取用同一书写系统写成的标签；
    显式标签只在可用时被采用；否则为 `"en"`。书写系统这一层之所以存在，是因为系统语言通常携带目录
    未命名的区域：`zh-Hans-CN` 能抵达 `zh-CN`（两者都最大化到 `zh-Hans`），而仅做基础语言比较会停在
    `zh` 并错过它。书写系统永不跨越：`zh-TW` 和 `zh-HK` 最大化到 `zh-Hant`，会落到 `en`，而不会
    借用简体目录。
  - `baseLanguage(tag)` 和 `fallbackChain(tag)`（`[tag, base, "en"]`，已去重）。
- **`@thinkrail/locales/react`**（浏览器）：初始化好的 i18next 实例（`i18n`）、`useT()`——渲染用的
  `Translator`，一个**在语言或其字符串变化时就会被替换**的值（`useSyncExternalStore` 订阅），因此
  任何由它计算出的东西（包括 React Compiler 缓存）都会重算；`localeOf(t)`——某个翻译器被签发时的
  语言，供渲染时的 `Intl` 格式化使用；`currentTranslator()` / `t` / `currentLocale()`——同样的东西，
  在调用时读取，仅供事件期代码使用（处理器、effect、store 动作、toast）；`<T>`（由 `useT()` 的
  翻译器驱动的 react-i18next `Trans`；它在解析标记之前转义插值，之后对文本节点反转义，因此像
  `feature/<b>x</b>` 这样的分支名会按字面渲染）；以及 `applyLocale(tag, overlays)`，它为每个链成员
  移除并重新添加 `bundled ∪ overlay`（因此被删除的叠加字符串会消失），切换语言并发布新的翻译器。
  i18next 以 `load: "currentOnly"` 和显式的 `fallbackChain` 运行，因此它恰好查询 `tag → base → en`
  （永不用未列出的书写系统回退）。

## 格式（Format）

扁平 i18next JSON v4：一层 `"dotted.named.key": "text"`；`{{name}}` 插值；`key_one`/`key_other`（以及
其他 CLDR 类别）复数经 `Intl.PluralRules` 解析；仅 `<T>` 字符串使用 `<0>…</0>` 索引标记。没有元数据
块。实例以 `keySeparator: false`、`nsSeparator: false` 和 `escapeValue: false` 运行（由 React 转义）；
实时编辑文件会触发重渲染，因为 `applyLocale` 发布了新的翻译器。它在**导入时以英文同步初始化**，因此
任何界面——包括没有任何 setup 的测试——都会渲染英文。

键以界面命名（`settings.language.title`、`chat.composer.placeholder`），永不用英文文本：一次措辞修正
不得使译文变成孤儿。`en.json` 保持排序；一份随包译文持有其键的已排序子集，加上其语言需要而英文缺少
的 CLDR 类别（西班牙语增加 `_many`），因此文件的 diff 干净。

## 边界（Boundary）

- **允许的依赖：** 工作区内无；外部为 `i18next`、`react-i18next`（仅 `./react`）、`react`（peer）。
- **消费方：** `apps/web`、`packages/ui`、`thinkrail-extensions/*/web`（两个入口）；`packages/server`
  和 `apps/desktop`（仅根入口）。
- **禁止：** 线上类型、传输层、store、文件系统访问（由宿主读取文件）、任何关于本地文件所在位置的
  知识。

## 必须做对（Get right）

- 英文输出是契约测试所钉住的对象：改动英文措辞的目录编辑是一次产品变更，要像产品变更那样接受评审。
  复数对携带真实的英文单数（`"1 group"`，绝不是 `"1 groups"`）。
- **随包译文按设计是不完整的，并由测试而非运行时守门。** 它缺少的字符串会经回退链渲染英文，因此它
  可以不完整地发布、之后补齐；新键先落到 `en.json`。`applyLocale` 按原样加载随包目录（`validateCatalog`
  和 `completePlurals` 只对本地文件运行），因此目录测试要求每份随包译文都能通过 `validateCatalog`
  且无任何丢弃，只持有已排序的 `en.json` 键——加上当该语言需要而英文缺少某个 CLDR 类别时的复数形式
  ——并携带其语言需要的每一种复数形式。因此，改变英文字符串的占位符或标记，或删除一个键，都会在同
  一次变更中失败，而那次变更要更新或删除对应译文（删除即回退到英文）。改变含义的重新措辞同样删除
  译文，而不是让它过期。
- **用户必须逐字输入的字面量保持原样。** 在译文正文中引用的协议回答、JSON 键、标志或命令保持其确切
  拼写：OpenSSH 对主机密钥只接受 `yes`，因此 `panels.prSetup.hostKeyHint` 在每个目录中都保留 `yes`
  ——翻译成 «sí» 会让 ssh 重新提示，推送依旧失败。翻译句子，永不翻译该记号。
- **渲染把翻译器当作值读取，永不用全局。** React Compiler 按其各自的输入缓存每个渲染表达式，因此
  读取全局 `t` / `currentLocale()` 的辅助函数（或这样做的 getter）在切换后仍保持旧语言。在渲染时求值
  的每个函数——组件与 hook 主体、它们的 `useMemo` / `useState` 初始化器、`.map` 风格的
  回调、store 选择器，以及它们触及的每个辅助函数——都把 `useT()` 的 `Translator` 作为参数
  （`layoutTabName(t, tab)`、`relativeTime(t, ms, now)`），而带目录标签的数据携带由消费方翻译的键
  （`ResourceRenderer.labelKey`、`MCP_STATE_KEY`）。`check:i18n` 用类型化可达性检查强制执行这一点。
  已否决：语言变更时重挂载应用——它会丢弃未发送的附件和计划输入，并重跑终端附加等生命周期 effect，
  而终端附加可能从另一个客户端夺走终端。
- 持久化或线上标识符永不携带译文。默认值以语言中立方式存储、在渲染时措辞：内置布局预设名；未命名
  聊天存储空名字；默认终端存储模式 `Terminal N`（其标题经宿主与其他客户端共享，因此编号会读回该
  模式）；Changes / diff 标签名由其作用域推导；计划文档存储原始聊天名；宿主命名的默认工作区保留
  `DEFAULT_WORKSPACE_NAME`；聊天轮次携带种类（轮次结束标记、失败的 `error` / `truncated` 种类）而不是
  英文文本，而 provider 或 agent 自己的消息按原样显示。保存时匹配的标记（MCP 的隐藏值占位符）保持
  字面。
- 占位符永不使用 i18next 选项名（`count` 除外）：`t()` 会把 `ordinal`、`context`、`lng` 等当作选项
  消费——`{{ordinal}}` 曾把一个终端分组标签切换到序数复数规则并渲染出它的原始键。目录测试会拒绝这类
  名字。

## 暂缓（Deferred）

按需加载随包译文（每份都会被导入每个消费方的 bundle）、语言感知的数字格式（token 计数、成本与上下文
百分比仍使用英文分隔符）、内嵌已翻译计数片段的复合短语（`chat.trust.untrustedNotice` 插值一个
“1 skill”列表）、RTL 布局（`dir`）、首屏语言提示（在宿主回答 `locales.list` 之前，本地语言会短暂
渲染英文），以及翻译服务端错误文本（错误将改为由客户端渲染的错误码）。
