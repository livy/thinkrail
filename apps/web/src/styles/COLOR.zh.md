# 色彩系统

> 中文译文，原文见 [`COLOR.md`](./COLOR.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

颜色以**两层**到达，而组件只允许命名第二层。

```
themes/bundled/*.theme.json    the PALETTE — one file per theme, the only place a hex lives
styles/colors.json             the SEMANTIC layer — what each palette entry is FOR (one file)
styles/colors.schema.json      the editor-facing contract for it
scripts/colors.ts              load / validate / render — the only place a derivation is written
scripts/generate-colors.ts     writes the output; `--check` fails when it is stale
styles/generated/colors.css    GENERATED: the roles, the `@theme inline` map, the effect blocks
themes/runtime.ts              writes the active manifest onto the document root
styles/colorUsage.test.ts      the adoption guard
```

**没有颜色被写两次。** 一个角色的推导只存在于 `colors.json`，不存在于任何其他地方 —— 不在
`tokens.css`（只放结构），不在 `index.css`（完全不含 `--color-*`），也不在 `runtime.ts`（它现在
推导变量名，而不是把它们制表列出）。编辑生成文件会使 `bun run colors:check` 失败，该检查在
pre-commit 与 `apps/web` 的构建中运行。

**变量名是推导出来的，不是选出来的。** manifest 的键写成其 kebab-case 名称 ——
`borderStrong` → `--border-strong`，`editorSelection` → `--editor-selection`。`colors.json` 与
`runtime.ts` 各自独立应用同一规则，因此一个角色的 `from` 与它读取的变量不可能不一致。没有需要
同步维护的查找表，也不再有把 `info` 写成 `--blue` 这样的名字。

**调色板条目（palette entry）** 回答的是*哪一个颜色*（`--warning`、`--elevated`、`--hint`）。
**语义 token** 回答的是*用来做什么*（`feedback-warning`、`container-elevated-bg`、`text-muted`）。
组件命名角色；调色板是内部实现。

```tsx
<div className="bg-container-elevated-bg text-text-muted border-border-default" />  // yes
<div className="bg-[var(--elevated)] text-hint border-border2" />                   // no
```

## 为什么拆分（Why the split）

主题可以改变一个角色解析到*哪一个*颜色，而不触碰任何一个组件；一个角色可以在一个地方重新指向，
而不是在 160 个调用点。它也让失败模式变得可见：`bg-elevated` 不再是 Tailwind 工具类了，而
Tailwind **会静默丢弃未知的工具类** —— 元素渲染成无样式，但它的 class 列表却声称并非如此。
`colorUsage.test.ts` 的存在正是因为这个问题曾经被发布出去。

## token 清单（The tokens）

在 `colors.json` 中声明，由生成器发布为工具类。每一个都被使用；没有调用点的 token 会被删除，
一个永远不可能与其相邻项不同的别名也会被删除（等于 `border-default` 的 `border-strong` 不是第二
种字重，而只是第二个名字）。

| 家族 | token | 说明 |
| --- | --- | --- |
| Text | `text-default` · `text-muted` · `text-subtle` · `text-disabled` · `text-on-primary` | `text-subtle`（来自 `hint` 调色板键）是次级元数据层级（分支行、规格角色标签）；`text-disabled`（默认的 `text` 颜色 @ 60% —— 禁用元素继承其启用时的语义颜色，在 token 层被压暗）保留给真正禁用的 UI 文本 |
| Container | `container-workspace-bg` · `container-sidebar-bg` · `container-terminal-bg` · `container-header-bg` · `container-content-bg` · `container-elevated-bg` | **打开文档的画布是 `workspace`，不是 `content`**：Monaco 桌面文件编辑器、Pierre 手机端文件和 markdown/规格预览与聊天栏、标签条位于同一个表面，因此文档读起来是工作区的一部分。`content` 是**凹陷的 diff 画布** —— Pierre 源码 diff、渲染后的 diff、Shiki 代码块，以及它们背后的 Center Workbench 背景。`terminal` 是终端主体 + xterm 画布（目前取自与 `sidebar` 相同的调色板键）；`elevated` 是每一个抬升表面 |
| Control | `control-bg` · `control-bg-hovered` · `control-bg-selected` · `control-primary-bg` · `control-primary-bg-hovered` · `control-primary-text` · `control-border-default` · `control-border-active` · `control-disabled-bg` · `control-disabled-text` · `control-disabled-border` · `control-primary-disabled-bg` · `control-primary-disabled-text` · `control-destructive-bg` · `control-destructive-bg-hovered` · `control-destructive-text` · `control-destructive-disabled-bg` · `control-destructive-disabled-text` · `control-soft-primary-bg-hovered` | 三个 `control-primary-*` token 是主要按钮/控件，并且它们**和其它每个角色一样是按主题推导的** —— `control-primary-bg`（来自 `accentSolid`，即主题可以独立于 `accent` 提亮的实心填充）为填充色，`control-primary-bg-hovered`（来自 `accentHover`）为 hover 填充色，`control-primary-text`（来自 `onAccent`）为标签色。`accentHover` 是强调色的 **hover 步进**，是它自己的 manifest 键，因此主题拥有那一步：主要按钮 hover 时变为 `control-primary-bg-hovered`（一个颜色 token），绝不使用 `hover:opacity-*`。只有在主题把 `accentSolid` 设为其 `accent` 的地方，`bg-control-primary-bg` 才等于 `bg-primary`（Dark、Minimal 一对、High Contrast Dark）；Light 与 High Contrast Light 保留更亮的填充。因为焦点指示器是同一个 `primary` 颜色，按钮焦点是一个*偏移*轮廓（`packages/ui` Button system），绝不是紧贴填充的 ring。**Destructive** 用自己的 manifest 键镜像主要三件套：`control-destructive-bg`（来自 `danger`）、`control-destructive-bg-hovered`（来自 `dangerHover`，一个颜色步进，绝不使用 `hover:opacity-*`）与 `control-destructive-text`（来自 `onDanger`，按主题选择以在两种填充上都满足 AA —— `themes/schema.test.ts`），而 `control-destructive-disabled-bg|text` 是同样两个值 @ `strong`。**Soft-primary** 落在 `primary-subtle` 上，带 `primary-soft` 边框，hover 到 `control-soft-primary-bg-hovered`（accent @ `wash`）加 `primary-muted` 边框 —— 而不是 hover 到 `primary-soft`（在那里 Light 的强调色标签会掉到 AA 以下）；`themes/schema.test.ts` 要求强调色标签在按钮所处的每个表面上的两种色调上都满足 AA（凹陷的 `content` 画布被排除：那里纯强调色文本在 Light 下已经处在 AA 边缘）。`control-bg-hovered` 仅用于指针 hover；`control-bg-selected` 是持久的选中/打开/活跃/高亮填充（目前是同一个调色板来源）。`control-border-default` 是静息态表单控件边框；`control-border-active`（来自 `borderStrong`）是*活跃*控件的**更强的中性**边框 —— 打开的 select（`data-[open=true]`），以及获得焦点的文本输入框/文本域。它只是边框：强调色焦点指示器保持独立（因此强调色 = 焦点，中性强色 = 活跃）。共享按钮的按下态**不是**边框切换：它看起来与 hover 完全一致（`packages/ui` Button system）。绝不用于非活跃/默认控件，也不用于选中的导航行、标签或静态表面。**禁用是一等控件状态**，不是透明度工具类：**禁用元素继承其启用时使用的同一语义颜色，在 `strong`（60%）alpha 步进上解析** —— 在 token 层推导，因此背景、文本、图标与边框各自保持明确归属，嵌套内容永远不会被压暗。因此禁用角色就是启用角色 @ 60%：`control-disabled-bg`（来自 `input`，即 `control-bg`）、`control-disabled-text`（来自 `text`，即 `text-default`）、`control-disabled-border`（来自 `border`，即 `control-border-default`），以及主要一对 `control-primary-disabled-bg`（来自 `accent`）+ `control-primary-disabled-text`（来自 `onAccent`，即 `control-primary-text`）。**没有**专用的 `disabled` / `primaryDisabled` / `onPrimaryDisabled` 调色板键 —— 禁用态从不携带自己的颜色。纯文本/图标控件只取 `control-disabled-text`；非控件的禁用文本仍用 `text-disabled`。**不要**在组件上使用 `disabled:opacity-*`（它会压暗嵌套内容并绕过 token 归属）。**一处有记录的例外**：composer 的发送胶囊是一个主要控件，它在多数时候是*惰性*的（草稿为空），而那里一块 60% 的强调色会把视线拉向一个无法行动的控件 —— 因此它落在 `control-bg-selected` + `control-disabled-text` 上，切换到完整的强调色填充就是「就绪」信号（`chat/SPEC.md`，*Composer trailing controls*）。任何其它禁用的主要控件保留 60% 那一对 |
| Border | `border-default` · `border-muted` | |
| Primary | `primary` + `primary-subtle` · `-soft` · `-muted`、`on-primary-soft` | |
| Feedback | `feedback-{info,success,warning,error}` + 使用中的 `-subtle` / `-muted` 步进 | 实心边框就是实心颜色，因此没有 `-border` 层 |
| Chat bubble | `bubble-user-bg` · `bubble-user-border` | 取自 manifest 自己的 `bubbleAccent` 的着色。标准与 High Contrast 调色板发布的它**等于各自的 `accent`** —— 用户气泡穿着品牌色；Minimal 调色板把它重新指向中性灰，这样用户内容不会与操作/注意色竞争。那个独立的键就是旋钮（依据 `themes/SPEC.md`）：主题决定气泡的观感，而不触碰组件 |
| Effects | `overlay` · `sunken` | 由主题引擎按浅色/深色分别写入 |
| Strict-consumer editor roles | `editor-selection-bg` · `editor-selection-text` · `editor-selection-highlight-bg` · `editor-find-match-bg` · `widget-shadow` | `publish: false`；Monaco/xterm 直接读取它们，不输出任何 Tailwind 工具类 |
| Media | `media-checker-tile` | `publish: false`；`index.css` 的 `.media-backdrop` 在每个图片与矢量预览背后绘制的透明棋盘格瓦片，由 `text` 在 `soft`（20%）步进上推导，因此在深色主题中是深底浅格，在浅色主题中是常规的白底 `#ccc` —— 透明 PNG 上的深色 logo 绝不能读成「没有图片」 |

没有 `text-strong` 也没有 `text-link` 工具类：它们与其它 token 重复，而 `--text-link` 作为变量
存在，只服务于 `global.css` 的裸 anchor 回退。该回退位于 CSS base 层，因此链接形状控件上显式的
语义文本工具类总能赢得层叠。

## 透明度：只有一种形式（Transparency: one form only）

**着色就是一个 token，在 alpha 刻度上按 `in srgb` 混合。**

```
subtle 10%   ·   wash 12%   ·   soft 20%   ·   muted 40%   ·   strong 60%
```

`wash`（12%）是反馈表面填充步进（`feedback-*-subtle`），比 `subtle`（10%）高一档，这样
`primary-subtle` 与 `bubble-user-bg` 保持在 10%，而反馈背景读作 12%。

Tailwind 的 `/40` 透明度修饰符**不用在颜色工具类上**。它以 `in oklab` 混合，因此同样的名义百分比
在来自 class 与来自 token 时渲染不同 —— 而且数字发生了漂移（10、12、15、16、25、30、35 与 50
曾同时存活）。新的着色是刻度上的新 token，绝不是 class 名里的新数字。

## 非 CSS 消费方（Non-CSS consumers）

Monaco、Pierre、xterm、mermaid 与 Shiki 无法为其内部绘制的内容套用一个 class。Monaco、xterm 与
mermaid 通过 `getComputedStyle` 读取 token，并在 `[data-theme]` 切换后重建；Monaco 现在只有
`EDITOR_THEME`，并在 `container-workspace-bg` 上绘制桌面文件。Pierre 打开的 Shadow DOM 继承一个
`.pierre-code-surface` 自定义属性桥：它的 CSS 变量 Shiki 主题把 `--diffs-token-*` 映射到已有的
`--code-*` 语法角色，而 `--diffs-bg*`、数字前景色，以及新增/删除基底映射到工作区/内容画布、弱化
文本与反馈成功/错误角色。这些值保持为活的变量引用，因此 Pierre 既不需要按主题注册，也不需要在
切换后重新高亮。Monaco 与聊天共享同一个 `THINKRAIL_SHIKI_THEME` TextMate 定义；聊天保留其实时
CSS 变量形式，而 Monaco 把同样的变量解析为 hex 以用于其严格的主题 API。因此所有消费方都命名着与
其它一切相同的语义 token，一个值只有一个名字。这些消费方与组件*共享*的角色（`--container-*-bg`、
`--text-muted`）保持发布并映射在 `@theme inline` 中；**只有**它们读取的那些（`--editor-selection-*`、
`--editor-find-match-bg`、`--widget-shadow`，以及仅样式表使用的 `--media-checker-tile`）是
`publish: false` 且未映射的 —— 一个没人能用的工具类是死重。严格 JS 消费方需要的值经
`cssColorToHex`（`lib/utils.ts`）规范化成 hex 抵达它们，因为构建后的 CSS 被压缩，而 Monaco/xterm
只接受 hex。

**它们有意不覆盖什么。** 这些库每一个的绘制都远超我们交给它的内容，其余部分来自它自己的内建
调色板：

| 消费方 | 我们设置的 | 其余来自 |
| --- | --- | --- |
| Monaco | 桌面文件画布/前景色/装订线；活跃 + 非活跃行号；光标 + 两种选择色；当前行填充/边框；缩进参考线；括号匹配 + 六种括号对颜色；单词/选区/查找高亮；粘性滚动画布/阴影；折叠/空白绘制；编辑器 + hover 部件；菜单、quick-input 与输入表面/状态；焦点边框；部件/滚动条阴影 + 滑块状态；每条 TextMate 语法规则 | 通过 `inherit: true` 的 `vs` / `vs-dark` / `hc-black` / `hc-light` —— suggest/peek/debug、minimap 与其它未列出的专门表面 |
| Pierre | 文件/diff 画布与装订线、前景色/行号/选区、语法变量、成功新增、错误删除、分隔/注释表面、代码家族/字号 | 未桥接属性的 Pierre 核心布局/效果；不应用任何 `@pierre/theme` 预设 |
| xterm | 背景、前景、光标、两种选择色、全部 16 个 ANSI | `cursorAccent` 与 `selectionInactiveBackground` 的 xterm 默认值 |
| mermaid | `thinkrail-extensions/visualize/web/mermaid.ts` 中的 `themeVariables` 映射 | 该映射中缺失内容的 mermaid `base` 主题 |

这是一个有界、被接受的缺口，而不是疏忽。仅 Monaco 就暴露约 200 个颜色键；逐一枚举会是一个庞大
而脆弱的表面，而基底是根据 manifest 的外观与对比度元数据选择的，因此它从不会错得离谱。如果某个
主题在这些部件之一中看起来不对，修法是加上那个具体的键 —— 而不是采纳整个表面。

## 新增或修改颜色（Adding or changing a colour）

每种情况都是一次 JSON 编辑，随后运行 `bun run colors:generate`。

1. **某个主题应该看起来不同** → 编辑 `themes/bundled/<theme>.theme.json`。其它什么都不变，
   也不需要重新生成：manifest 在运行时被读取。
2. **某个角色应该指向别处** → 修改它在 `colors.json` 中的 `from`。
3. **新角色** → 把它加入 `roles`，带上 `from`、可选的 `alpha` 步进，以及 `publish`
   （`true` → 一个 Tailwind 工具类；`false` → 由 Monaco/xterm/mermaid/`global.css` 直接读取）。
4. **新着色** → 一个来自 `scale` 的 `alpha` 步进，绝不在调用点写 `/N`。如果该步进本身是新的，
   就把它加入 `scale` —— 那是一个设计决策，而且只做一次。
5. **一个角色需要让两个主题能够不同** → 它需要自己的 manifest 键。把它加入 `THEME_COLOR_KEYS`
   与 `theme.schema.json`，以及每个内置 manifest，然后让某个角色指向它。当归某个角色命名了不存在
   的键，或某个键没有任何角色读取时，生成器拒绝运行。

绝不要：组件中的原始 hex 或 `rgb()`、内联 `style` 对象、`bg-[var(--palette-entry)]` 逃生口，或给
已有名字的值再起第二个名字。

## 由测试钉住的内容（What is pinned by tests）

`styles/colorUsage.test.ts` 通过共享的构建期源码扫描器扫描应用源码、`packages/ui`，以及每个
`thinkrail-extensions/*/web` 树。当以下情况出现时它失败：

- 一个颜色工具类命名的 token 未被 `@theme inline` 发布（静默丢弃 bug）；
- 一个组件包含原始 hex、`rgb()` 或 `hsl()`；
- 一个组件通过 `bg-[var(--…)]` 触达调色板条目；
- 一个颜色工具类上出现 `/N` 透明度修饰符；
- 一个已声明的 token 没有调用点，或一个已发布的工具类没有 token；
- 提交的生成文件与 `colors.json` 渲染出的内容不匹配。

`themes/schema.test.ts` 额外钉住**对比度下限**，这些值得在这里说明，因为其中两行是我们的判断而非
标准的要求：

- 在每一个静息表面（`background`、`content`、`sidebar`、`header`、`elevated`、`input`）上文本完整
  满足 WCAG AA —— 正文与 `text-muted` 层为 4.5；
- 刻意安静的 `text-subtle` / `hint` 层在每个静息表面上保持在 **3.0** 且可见；
- 在瞬态 HOVER 表面上，下限是 **3.0**，不是 4.5；
- `hover` 填充 —— `control-bg-hovered` / `control-bg-selected` 的调色板来源 —— 保持**与每个静息
  表面可区分**：对着全部六个表面 ≥ **1.15**。没有画布被豁免：交互填充会抵达其中每一个（content
  画布承载 PlanPane 的行与 workbench 背景的控件；`input` 是 hover 控件切离的静息填充）。

WCAG 没有「瞬态」豁免，因此 hover 层是我们刻意划出的线。这些主题在 hover 时把行背景向文本颜色
抬升，若把这一点也压到 4.5，就会迫使主题的标志性强调色走向褪色的色调，只是为了在 hover 行上存活。
3.0 让 hover 文本舒适可见，同时让各主题保持可辨识。如果「每个状态都严格 AA」某天成为要求，请重新
审视它。

可区分性下限同样是我们的：WCAG 完全没有相邻填充的度量。它存在的原因是 High Contrast Light 发布
时 `hover` 对它自己的侧边栏只有 1.05:1 —— 选中的项目/工作区行与面板无法区分 —— 而所有可读性检查
都是绿的；随后对该修复的复审又发现 Light 直接把 `hover == content` 发布出去，于是 PlanPane 的
hover 行同样消失了。1.15 是区分可见填充与不可见填充的那条线；每个内置主题在全部六个表面上都通过
它（存活的最弱一对是 Light 的 hover-on-content，1.165）。

`themes/runtime.test.ts` 钉住应用行为；`themes/shiki.test.ts` 钉住语法变量映射。manifest 本身见
[`themes/SPEC.md`](../themes/SPEC.md)，并行的字体系统见 [TYPOGRAPHY.md](./TYPOGRAPHY.md)。

## 范围：应用及其 UI SDK/扩展（Scope: the app and its UI SDK/extensions）

`apps/website` —— 公开落地页 —— 有自己的样式表，带有自己硬编码的颜色与字体，与本系统不共享任何
东西。这是刻意的：它是 GitHub Pages 上的静态营销页，没有主题切换、没有运行时，也没有理由携带
token 层。不要通过从这里导入来「修好」它；如果它某天需要与应用一致，做法是抽出一个小的共享 token
包，而不是跨应用伸手。
