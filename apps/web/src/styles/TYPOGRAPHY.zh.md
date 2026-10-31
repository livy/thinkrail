# 字体排版系统

> 中文译文，原文见 [`TYPOGRAPHY.md`](./TYPOGRAPHY.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

**`styles/typography.json` 是字体排版的唯一事实来源。** 每个字体家族、字号、字重、行高、字距、
变换与语义文本样式都存在于其中；应用发布的 CSS 是由它*生成*的。任何组件都不得声明字体排版值，
任何手写工具类都不得新增一个，任何其它文件都不得持有指向它的别名 —— 通过编辑 JSON 来改变任何东西
的字体。

```
styles/typography.json            the source (see typography.schema.json for the shape)
styles/typography.schema.json     the editor-facing contract: primitives + fully-resolved styles
scripts/typography.ts             load / validate / render — the only place CSS names are derived
scripts/generate-typography.ts    writes the CSS; `--check` fails when it is stale
scripts/validate-typography.ts    the enforced gate: shape + referential + policy validation
styles/generated/typography.css   GENERATED, committed, never edited by hand
```

| 命令 | 作用 |
|---|---|
| `bun run typography:generate` | 重新生成 `styles/generated/typography.css` |
| `bun run typography:validate` | 校验 JSON（形状、引用、策略） |
| `bun run typography:check` | 若提交的 CSS 过期，或采纳情况违反 token 契约则失败 —— pre-commit + `apps/web build` |

`bun test` 也会运行守卫：`styles/typography.test.ts`（来源 + 生成输出）与
`styles/typographyUsage.test.ts`（使用共享源码扫描器，跨应用源码、`packages/ui` 与每个扩展 web
两半的采纳情况）。`e2e/typography.spec.ts` 断言真实表面上的*计算后*样式（hero、对话框/卡片标题、
实体行、分支元数据、eyebrow、Monaco、xterm、两个 markdown 表面、`<body>` 基底）。

> **关于 schema。** `typography.schema.json` 是编辑器通过 `$schema` 读取的契约 —— 它在你编辑时
> 提供补全与内联错误。它**不是**门禁：工具链中没有 JSON Schema 校验器，形状检查以及下面的每条
> 策略都实现在 `scripts/typography.ts` → `validate()` 中。改一个，就要检查另一个。

## 原始值与语义样式（Primitives vs semantic styles）

**原始值（Primitives）** 是原始词汇 —— 扁平的 id → 值映射。它们是系统中唯一的数字：

| 分组 | id |
|---|---|
| `fontFamilies` | `interface`（Geist Variable，全部比例 UI + 阅读文本）· `code`（JetBrains Mono Variable，仅代码）· `brand` → **Orbitron Variable**（自托管 `@fontsource-variable/orbitron`，为品牌角色 —— `wordmark` + `hero` —— 提供的独立展示字体；它原先是指向 `interface` 的 `$ref`）。Orbitron **仅限拉丁字符**，而 `brand.hero` 渲染项目自己的名字，因此字体栈在任何系统字体之前就回退到 *interface* 字体 —— 非拉丁名字落在 Geist（西里尔/拉丁扩展/越南语，均已内置）上，而不是落在操作系统恰好提供的字体上 |
| `fontWeights` | `light` 370 · `regular` 400 · `medium` 500 · `semibold` 600 · `brand` 400 · `button` 470 |
| `fontSizes` | `s10` `s11` `s12` `s13` `s14` `s16` `s18` `s20` `s24` `s44`（px） |
| `lineHeights` | `compact` 1.25 · `metadata` 1.25（12px→15px）· `ui` 1.3（14px→18.2px）· `code` 1.4 · `relaxed` 1.4（13px→18.2px）· `default` 1.3（14px→18.2px）· `reading` 1.45（14px→20.3px） |
| `letterSpacings` | `normal` · `loose` 0.02em · `wide` 0.05em · `widest` 0.1em · `brand` 0.5px |

前导刻度刻意**密集**（1.25–1.4）：Geist 在 14px 时的 x-height 为 7.4px（大写高度 9.9px），因此
原先的 1.5–1.6 会在比字母本身还高的行与行之间留下一条空白带 —— 它读起来近似双倍行距。阅读散文 ——
`body.reading`：聊天、用户消息、文档 —— 是刻意的例外，用 `reading` 1.45：在密集值下长回复读起来
像一堵压缩的墙，因此阅读文本比 UI 装饰保留更多前导。Monaco 与 Pierre diff 表面读取
`--tr-line-height-default` 并保持密集；两者都使用整像素的代码行（Pierre 的虚拟化器需要精确行高）。
`typography.json` 中的 `codeRow` 命名其字号与行高，生成器输出预先取整的 `--tr-code-row-height`，
Pierre 的 CSS 与其虚拟化器都读取它，因为 Safari 15.4 之前没有 CSS `round()`。这些值刻意写成简短
的精确小数：生产压缩器会把自定义属性舍入到五位小数，因此像 18/14 这样的循环比值永远无法落到整像素
（它曾以 `17.9999px` 发布）。

**语义样式（Semantic styles）** 是组件使用的东西。每个都只命名七个原始引用，别的什么都没有，因此
它确定性地解析 —— 没有继承、没有按用法分支、没有回退：

```json
"title": { "dialog": {
  "fontFamily": "interface", "fontSize": "s14", "fontWeight": "semibold",
  "lineHeight": "compact", "letterSpacing": "normal",
  "textTransform": "none", "fontStyle": "normal"
} }
```

一个样式也可以是一个**别名** —— `{ "$ref": "title.dialog" }` —— 意味着*与该样式完全相同*。规则
全部由 `typography:validate` 强制执行：

- 一个**规范定义**携带完整的七个值；
- 一个**别名**只携带 `$ref`，别的什么都没有；
- `$ref` 必须命名一个**规范定义** —— 绝不能是另一个别名，因此**禁止链式**，解析是一次直接查找；
- 缺失目标，或 `$ref` 指向自身，都会被拒绝；
- **两个规范定义不得持有相同的值** —— 那是一个缺失的 `$ref`；
- 别名仍然输出自己的语义 CSS class，因此 `.tr-title-card` 与 `.tr-title-dialog` 是由一个定义支撑
  的独立 class。

`textStyles` 分组：**brand**（`hero` —— Welcome 文本的身份；shell logo 由矢量素材提供）·
**heading**（`xl`、`lg`、`md`、`sm` —— 共享的文档标题刻度）· **title**（`dialog`、`card`→dialog、
`section`、`compact`、`entity`→body.reading）· **ui**（`default`、`metadata`、`eyebrow`、
`labelPill`→eyebrow、`caption` —— 10px/compact 大写徽标与磁贴首字母样式（行状态徽标、顶栏项目
首字母），在 `eyebrow` 的 18px 行放不下的地方，`action`→title.compact、`button` —— 共享按钮标签，
比 `ui.default` 重一个字重步进、`emphasis`→title.compact）· **body**（`reading`）· **code**
（`text` —— 基础 13px 代码样式，`document`、`otp`、`textSmall` —— 用于表格单元格内联代码的 11px
代码样式）。`proseSystems` 为每个 markdown 表面保存一个条目，几乎全是指向上述样式的别名。死别名
不予保留：散文直接指向它使用的语义样式。

**21 个规范定义 + 30 个别名 = 51 个样式。**

有一条散文规则刻意*不是*语义样式：`<strong>` / `<b>` 只获得**字重**（`--tr-font-weight-medium`），
由生成器输出到每个散文系统中。那里若用完整样式，会覆盖粗体文本所在元素的字号与行高。

JSON **不**持有 CSS 选择器、class 字符串、组件路径、用法清单、理由或审计数据。理由是放在本文件里。

## `<body>` 基底（The `<body>` base）

`rootStyle` 是不携带任何语义 class 的文本的字体排版 —— 而且它**是一个 `$ref`，绝不是值**（今天
是 `{ "$ref": "ui.default" }`，即 14px/370 interface，行高 20px）。生成器把它输出为 `@layer base`
中的 `body { … }` 规则，这把它放在层叠中每个语义 class *之下*：它是地板，绝不是组件所依赖的东西。

为什么它是引用很重要：当基底持有自己的数字时，无 class 的文本渲染成没有任何语义样式命名的字号，
因此一个静默停止解析的 class 名看起来合理，而不是明显错误。因此 `styles/global.css` 只声明**颜色
与渲染** —— 没有家族、字号、字重或行高。

Tailwind 的 preflight 默认值（`index.css` `@theme inline` 中的 `--font-sans` / `--font-mono`）指向
同一批生成的 token，因此即便是系统尚未触达的元素 —— 一个裸 `<pre>` —— 也渲染在系统选择的字体中。
该映射同时启用的 `font-sans` / `font-mono` 工具类在调用点被禁止。

## 组件如何使用它（How components consume it）

生成器机械地为每个语义样式推导出一个 class：

| 来源 id | 生成的 class |
|---|---|
| `brand.hero` | `.tr-brand-hero` |
| `heading.xl` · `heading.lg` · `heading.md` · `heading.sm` | `.tr-heading-xl` · `.tr-heading-lg` · `.tr-heading-md` · `.tr-heading-sm` |
| `title.dialog` · `title.card` · `title.section` · `title.compact` · `title.entity` | `.tr-title-dialog` · `.tr-title-card` · `.tr-title-section` · `.tr-title-compact` · `.tr-title-entity` |
| `ui.default` · `ui.metadata` | `.tr-text-ui` · `.tr-text-metadata` |
| `ui.eyebrow` · `ui.labelPill` · `ui.caption` · `ui.action` · `ui.button` · `ui.emphasis` | `.tr-text-eyebrow` · `.tr-text-label-pill` · `.tr-text-caption` · `.tr-text-action` · `.tr-text-button` · `.tr-text-emphasis` |
| `body.reading` | `.tr-text-reading` |
| `code.text` · `code.document` · `code.otp` · `code.textSmall` | `.tr-code-text` · `.tr-code-document` · `.tr-code-otp` · `.tr-code-text-small` |
| `proseSystems.<id>.*` | `.tr-prose-<id>` + 各一个元素选择器 |

原始 token 也被输出为自定义属性 —— `--tr-font-family-code`、`--tr-font-size-s11`、
`--tr-line-height-default`、… —— 供无法使用 class 的表面使用。

**层叠层（Cascade layers）。** 三个层级，全部是刻意的：token 块不加层（自定义属性没有竞争者），
`<body>` 基底位于 `@layer base`，语义 class 位于 `@layer components`。Tailwind v4 把层排序为
`theme, base, components, utilities`，因此语义 class 胜过基底，而调用点的 Tailwind 工具类仍会覆盖
它命名的那个单一属性 —— 这正是让 `italic` 与 `leading-tight` / `leading-snug` 能与语义 class 共存
的原因。把 class 不加层地输出会压过每个工具类并静默胜出。

调用点的规则：

- **字体排版 = 恰好一个语义 class。** 绝不要组合 `font-*`、`text-<size>`、`tracking-*` 或
  `uppercase`。例外是 `italic` 与 `leading-*`：调用点可以添加一个来覆盖那个单一属性（见*层叠层*），
  这就是少数行与空状态携带它们的原因。
- **颜色留在调用点**，作为语义颜色 token（`text-text-muted`、`text-text-subtle`、条件性的
  actives）—— 见 [COLOR.md](./COLOR.md)。活跃/选中状态是**颜色**变化 —— 绝不是字重变化。
- **`<pre>` 与 `<code>` 总是命名一个样式**，即便在已经有一个样式的容器内。Preflight 直接针对这些
  元素，而直接匹配的规则胜过继承的家族，因此裸 `<pre>` 会渲染成与其 `tr-code-text` 父元素*不同*的
  mono 字体。由 `typographyUsage.test.ts` 钉住。
- 间距、截断、布局、hover 与状态 class 不受影响。

```tsx
<span className="tr-text-eyebrow text-text-muted">Projects</span>
<h2 className="tr-title-dialog text-text-default">{title}</h2>
<code className="tr-code-text text-text-subtle">{command}</code>
```

**当元素不属于你时。** 一个渲染自己内部 DOM 的第三方组件（cmdk 的分组标题）无法通过任意变体获得
class —— `[&_[x]]:tr-text-eyebrow` 什么也不产出，因为 Tailwind 会静默丢弃它不认识的 class 名。
改为传入一个节点（`heading={<span className="tr-text-eyebrow">Remote</span>}`），或把该元素留给
`<body>` 基底。`typographyUsage.test.ts` 会对生成器未输出的任何 `tr-` class 失败，这就是整类失败
模式现在被捕获的方式。

## 散文系统（The prose systems）

Markdown 通过 `proseSystems` 渲染：每个表面一个条目，各自拥有**同一个固定元素集**（校验会拒绝缺少
某个元素的系统，或添加了没有选择器的元素的系统），各自命名自己的字体。现存两个：

| | `chat`（`.tr-prose-chat`） | `doc`（`.tr-prose-doc`） |
|---|---|---|
| 挂载者 | `chat/Markdown.tsx` | `resource-renderers/support/MarkdownPreview.tsx` |
| 正文、块引用、列表 | 14 / 370 / 1.45 | 14 / 370 / 1.45 |
| h1 | 18 / 600 | **24 / 600** |
| h2 | 14 / 600 | **20 / 600** |
| h3 | 12 / 600 | **18 / 600** |
| h4 | 12 / 500 | **16 / 600** |
| h5 | 12 / 500 | 14 / 600 |
| h6 | 10 / 500 uppercase | 12 / 600 uppercase |
| 内联代码 | 13 mono | 13 mono |
| 围栏代码 | 13 mono / 1.54 | 13 mono / 1.5 |
| 表体 / 表头 | 12 / 370 · 12 / 600 | 14 / 370 · 14 / 600 |
| 表格单元格内联代码 | **11 mono / 1.5**（`code.textSmall`） | 13 mono（`code.text`） |
| `strong` / `b` | 仅字重 500 | 仅字重 500 |

**为什么是两个。** 聊天气泡是一串短消息：那里标题是分隔符，而气泡里 24px 的 h1 会大声喧哗。渲染后
的文件是文档：它的标题必须在视觉上比段落更大，否则结构就消失了。一个共享刻度无法同时胜任两者 ——
正文为 14px 且刻度紧凑时，六个标题层级中有四个落在正文尺寸或更低，这正是 `doc` 系统存在的原因。
校验强制执行：`doc.h1`–`doc.h4` 每一个都必须**大于 `doc.body`**，且 h1→h6 的阶梯绝不倒置。`h5`
与正文同尺寸，`h6` 在其下，两者都是 semibold，且 `h6` 大写 —— 每个 markdown 渲染器最终都落定的
惯例，因为文档基本从不嵌套到那么深。

两个系统共享它们的规范定义：`chat.h1` 与 `doc.h3` 都是 `{ "$ref": "heading.md" }`，各处正文都是
`body.reading`。28 个散文条目中只有 3 个是它们自己的规范定义。

在两个系统中 `strong` / `b` 都只获得**字重 500** —— 家族、字号、行高、字距、变换与颜色都继承自
外围元素，因此标题中的粗体保持标题的字号，单元格中的粗体保持表格的字号。

调用点的「散文皮肤」只承载间距、宽度度量与装饰（`chat/Markdown.tsx` = 气泡节奏，
`resource-renderers/support/MarkdownPreview.tsx` = 文档节奏、标题规则、表格装饰）。绝不要给皮肤
添加字体排版 class 或值；改 `proseSystems` 而不是。添加第三个表面意味着添加一个 JSON 条目 ——
生成器、命名与守卫都会跟随。

## 字体（Fonts）

自托管的可变字体。这些包在 JSON 中按家族以 `selfHosted` 声明，并由它们输出
`styles/generated/fonts.css`（`@fontsource-variable/geist`、`@fontsource-variable/jetbrains-mono`），
指纹后进入 `dist/assets` 并嵌入二进制 —— **没有字体 CDN**，因此离线宿主渲染真实系统。两种字体
都是可变的，所以 800 与斜体是真实的，不是合成的。由 `e2e/fonts.spec.ts` 钉住。

**替换一个家族是一处文件改动**：编辑家族的 `stack` + `selfHosted`，`bun add` 新包，
`typography:generate`。字体名不写在任何其它地方 —— 测试从生成输出读回它
（`e2e/fixtures/typography.ts`），而 `validate` 既拒绝不是依赖的 `selfHosted` 包，也拒绝没有家族
认领的字体依赖。字体栈的另一份副本在 `apps/website`（一个无法导入我们的独立叶子）；当它与本文件
漂移时 `apps/website/src/fonts.test.ts` 失败。

`tokens.css` 持有**完全没有字体排版** —— 没有值，也没有别名。它曾经携带 `--font`、`--font-mono`、
`--font-accent`、`--font-mono-size` 与 `--line-height` 作为指向生成 token 的别名；它们全部消失，
因为给一个值起第二个名字正是会漂移的东西。无法使用 class 的消费方直接读取 `--tr-*` token。
Cabinet Grotesk 已在全仓库退役，不得回归。

## Mono 策略（Mono policy）

`code` 仅用于**代码与技术内容**：编辑器与终端文本、代码块、内联代码、diff 与技术文件路径、shell
命令、斜杠命令语法、JSON/代码编辑表面、键帽、原始工具输出，以及 OTP 例外。它**绝不**用于项目或
工作区名称、分支或基线引用、模型名或 id、skill 名、标签、tag、普通元数据或状态 —— 那些是比例
字体。校验强制执行：非代码样式上的等宽家族会使 `typography:validate` 失败，比例家族上的代码样式
同样失败，等宽的 `rootStyle` 亦失败。

## 字重策略（Weight policy）

- **370**（`light`）—— 普通 UI、正文、实体、元数据与状态文本（interface 家族）。
- **400**（`regular` / `brand`）—— 等宽/代码文本（`code.*`）与品牌展示文本（`brand.*`）。
- **470**（`button`）—— 共享的 `Button` / `IconButton` 标签（`ui.button`，由 `packages/ui` 的
  `buttonVariants` 拥有）：14px interface 字体比普通 UI 文本重一个字重步进（+100），使动作标签
  读起来像动作，同时仍轻于标题。调用点从不设置它；它随按钮一起走。
- **500** —— 非按钮的动作文本（`ui.action`）、页内区块标题、紧凑标题、内联强调（`ui.emphasis`）、
  大写标签（`ui.eyebrow`/`labelPill`）、聊天散文 h4–h6 与 `strong`。
- **600** —— 对话框标题、卡片标题、警告标题、每个 `heading.*`、聊天散文 h1–h3、文档散文 h1–h6 与
  表头。

禁用控件的颜色来自 `control-disabled-bg` / `control-disabled-text`；禁用的非控件文本使用
`text-disabled`。字体排版不编码禁用状态，控件也不使用透明度来模拟它。

## 许可的例外（Permitted exceptions）

允许清单刻意极小，每一条都在 `styles/typographyUsage.test.ts` 中按名称强制执行：

| 表面 | 为何无法使用语义 class |
|---|---|
| `resource-renderers/support/monacoSetup.ts` | Monaco 以 JS 选项接收 `fontFamily` / `fontSize` / `lineHeight` —— 它读取 `--tr-font-family-code`、`--tr-font-size-s11`、`--tr-line-height-default`，因此无法与代码块漂移 |
| `resource-renderers/renderers/code/pierreVirtualizer.ts` | Pierre 的虚拟化器以 JS 度量接收行高 —— 它读取生成的 `--tr-code-row-height`，即 `.pierre-code-surface` 给 `--diffs-line-height` 的同一个值 |
| `panels/TerminalInstance.tsx` | xterm，同样原因 —— 它读取 `--tr-font-family-code` + `--tr-font-size-s13`（`code.text` 背后的原始值），并通过自己的 `lineHeight` 选项而不是 CSS 行高来拥有行高 |
| `thinkrail-extensions/visualize/web/mermaid.ts` | mermaid 的主题配置接收家族字符串（`--tr-font-family-code`） |
| `index.css`、`styles/tokens.css`、`styles/global.css` | 映射层本身；`.pierre-code-surface` 把代码家族、`s11` 与默认行高（取整到整像素）传入 Pierre 的 Shadow DOM |

OTP 代码**不再**是例外：它是具名的 `code.otp` 样式（`.tr-code-otp`）。

## 新增或修改样式（Adding or changing a style）

1. 编辑 `styles/typography.json` —— 一个新的原始值，或 `textStyles` / `proseSystems` 下的一个新
   条目。复用原始值；只有当没有现成值适用时才新增。**如果你需要的样式已经存在，就写
   `{ "$ref": "<that.style>" }`**，而不是重复它的值 —— 校验会拒绝值相同的第二个规范定义，也会拒绝
   指向另一个别名的别名。只有当两者真的产生分歧时，才把别名拆成它自己的定义。
2. 运行 `bun run typography:validate`，然后 `bun run typography:generate`，并**提交生成的 CSS**。
3. 在调用点使用生成的 class（颜色保持独立）。
4. `bun test`（来源 + 采纳守卫）与 `bun run e2e -- e2e/typography.spec.ts` 验证计算后样式。

**不要**：向 `index.css` 添加字体排版工具类或 `--text-*` 映射，向 `tokens.css` 添加字体别名，
在组件中声明字体属性，或编辑 `styles/generated/typography.css`。pre-commit 钩子与 `apps/web build`
运行 `typography:check`，因此过期的生成 CSS 无法落地。

这套「原始值 + 语义 token + schema + 生成器」的形状，正是**颜色** JSON 应当镜像的模式；这里没有
任何东西是 Tailwind 特有的，Tailwind 只是消费生成的 CSS。
