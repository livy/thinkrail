# 间距系统

> 中文译文，原文见 [`SPACING.md`](./SPACING.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

间距是**一套规范的数值词汇**。当设计说「使用 spacing 8」时，实现写 `p-8` / `gap-8` / `py-8`，
它恰好解析为 8px —— 一个值、一个名字，没有本地重新声明。这集中了编写时的节奏值；它并不从源码、
生成的 CSS 或编译后的 bundle 中移除字面量。

```
styles/spacing.json            the authored SOURCE — the only authored canonical rhythm lengths
styles/spacing.schema.json     the editor-facing contract for it
scripts/spacing.ts             load / validate / render — the only place a spacing derivation is written
scripts/generate-spacing.ts    writes the output; `--check` fails when it is stale
styles/generated/spacing.css   GENERATED: the `--space-<n>` tokens + the `--spacing: 1px` base
styles/spacingUsage.test.ts    the adoption guard
```

## 刻度（The scale）

`spacing.json` 声明步进 **`0 · 2 · 4 · 6 · 8 · 12 · 16 · 24 · 32 · 40 · 64`**。**步进名就是其像素
值** —— `"8": "8px"` —— 因此一条命名了数字的指令恰好映射到一个 token 与一族工具类。`0` 是恒等
步进（无间距）。

刻度是**一个刻意定义的原始集合，而不是当前用量的清单**：一个步进可以在任何消费方出现之前就存在。
`32`/`40`/`64` 是保留的原始值，没有节奏调用点 —— 它们是规范的，不是孤儿，也不会因为无人使用而被
移除。反之，新增一个步进是刻意的设计决策，而不是机械操作。`6` 是为一个有明确决定的消费方而加的 ——
L 按钮的图标↔文本间距（`packages/ui` Button system：S 4 · M 4 · L 6，低于按钮的侧内边距，只有 S
例外，在那里它等于侧内边距）。更早审计中的四处不相关的 6px 用法仍然被归并进已有的角色；新的 6px
节奏需要它自己的决策，而不是这个步进的存在。

`spacing.json` 是编写源；没有其它编写源重述某个规范节奏长度。生成的输出必然把这些值物化为
`--space-<n>` 自定义属性。它们被少数无法使用工具类的手写 CSS 表面直接读取（`index.css` 中的
Monaco 评审部件）以及严格的数值型第三方选项（Monaco 的文件查看器顶部内缩）读取；组件调用点使用
`p-<n>` / `m-<n>` / `gap-<n>` 工具类。编辑生成文件会使 `bun run spacing:check` 失败，该检查在
pre-commit 与 `apps/web` 的构建中运行。

## 数字 = px（以及为何尺寸也共享它）（Number = px (and why sizing shares it)）

工具类通过生成 `@theme inline` 块中设置的 **`--spacing: 1px`** 解析。Tailwind v4 把裸数字间距
工具类计算为 `calc(var(--spacing) * n)`，因此 1px 基底使 `p-4` = 4px、`gap-12` = 12px，且形式
统一。这**替换**了 Tailwind 内建的 `0.25rem` 基底，因此任何长度都不会回退到 Tailwind 自己的数值
刻度 —— 正是这个要求让 t-shirt 名字（`p-xs`、`gap-sm`）值得退役。

Tailwind v4 从**同一个 `--spacing` 基底**驱动**间距（`p`/`m`/`gap`）与尺寸（`w`/`h`/`size`/
inset/translate）** —— 它们无法按主题分开。因此 1px 基底也让尺寸变成「数字 = px」：`w-16` = 16px、
`size-14` = 14px、`h-32` = 32px。这是刻意的，并且是 1:1 迁移的（每个旧的 `w-4`/`size-3.5`/`h-8`
都变成 `w-16`/`size-14`/`h-32`），UI 完全保持原样。**但一个盒子是哪个 px 属于布局约束，不是节奏**
—— 尺寸有意*不*受间距门禁管辖，尺寸值是自由形式的 px，而不是规范步进集。

Tailwind 的**具名容器预设保持为独立刻度**：`max-w-lg`（以及其余 `w`/`min-w`/`max-w`/`basis` 的
t-shirt 名字）通过 `--container-*`（32rem）解析，独立于 `--spacing`。它们**不是**间距别名，绝不可
迁移为数值工具类 —— 机械的 `lg`→`16` 重写会把 32rem 的列塌缩成 16px。`spacingUsage.test.ts`
钉住这一点（门禁从不管辖宽度/尺寸前缀，我们依赖的容器预设保持具名）。

间距独立于字体、颜色与圆角：字体刻度或主题的变化从不移动布局。`tokens.css`（结构）不持有间距刻度；
`spacing.json` 是它唯一的编写源。

## 布局角色（Layout roles）

一个多个表面必须**同时**改变的值是一个具名布局角色，而不是重复的字面量，因此密度变化一次移动所有
消费方，不可能被半途应用（曾经漏掉一个字面量，结果每个兄弟行都移到了 24px，而 Specs 行仍停在
28px）。角色在 `tokens.css` 的 `:root` 中声明，并通过 `index.css` 的 `@theme inline` 映射进
Tailwind —— 与 `--panel-header-row-height` → `h-panel-header-row` 相同的形状：

| 角色 | 值 | 工具类 | 保持相等的东西 |
|---|---|---|---|
| `--panel-header-row-height` | `28px` | `h-` / `w-panel-header-row` | 每个头部条的高度（标签条、面板头、聊天工具栏、底部工具轨）及其全高方形动作的宽度 |
| `--panel-header-control-height` | `calc(var(--panel-header-row-height) - var(--space-4))` | `h-panel-header-control` | 每个浮动在头部条内的控件，因此每当头部高度变化，它都始终避开条带的 1px 边框 |
| `--panel-row-height` | `22px` | `h-` / `min-h-` / `size-panel-row` | 每个侧栏行（文件、变更、规格、项目、工作区）以及其中行高的图标按钮 |
| `--panel-inset` | `var(--space-8)` | `p-panel-inset` | 每个工具面板的内容内缩（Projects、Specs、Files、Review） |
| `--tree-indent` | `var(--space-8)` | `pl-tree-indent` | Files、Changes 与 Specs 树的逐层缩进 |
| `--chat-gutter` | `var(--space-8)` | `px-` / `p-chat-gutter` | 聊天行边距、流状态与活动面包屑条、composer 内缩，以及转录宽度与无界用户气泡计算中的边距项 |

- 一个**节奏角色别名一个规范步进**（`var(--space-<n>)`），绝不用原始长度，因此 `spacing.json`
  仍是节奏长度的唯一来源，而 `tokens.css` 依然不持有自己的刻度。尺寸角色（`--panel-row-height`）
  属于几何，和所有尺寸一样保持自由形式的 px。
- 只为有多个消费方且必须保持相等的值铸造角色；一次性长度仍是数值工具类。角色不是 t-shirt 别名，
  也从不替代刻度。
- 角色名是 kebab-case，且**两个或更多单词**（`panel-inset`）。`cn`（`@thinkrail/ui/utils`）教会
  tailwind-merge 把这种形状当作间距/尺寸值，因此角色会覆盖同一工具类的步进（`size-28` +
  `size-panel-row` → `size-panel-row`），而不是两个 class 都存活、由样式表顺序静默选一个；门禁会
  对单词角色名失败。

## 门禁（The gate）

`styles/spacingUsage.test.ts` 在 `p`/`m`/`gap` 调用点（以及手写 CSS 的节奏属性）上执行这套词汇，
从 `spacing.json` 读取允许的步进，因此两者不会漂移。共享的源码扫描器覆盖应用源码、`packages/ui`，
以及每个 `thinkrail-extensions/*/web` 树：

- 间距工具类命名一个**规范步进** —— `p-8`、`gap-4` —— 或一个**已声明的节奏角色** ——
  `p-panel-inset`：即 `index.css` 中 `--spacing-<role>: var(--<role>)` 而其在 `tokens.css` 的值为
  `var(--space-<step>)`；退役的 t-shirt 别名（`p-xs`）、未知的字母后缀（`p-bananas`）、被当作节奏
  使用的尺寸角色（`p-panel-row`）以及任何脱离刻度的数字（`p-10`、`py-1`、`gap-0.5`）都被拒绝；
  只有与前缀匹配的 Tailwind 关键字，如 `ml-auto`、`gap-px` 与 `space-x-reverse`，仍然有效；
- 在调用点上一个长度**绝不是原始像素值**（`py-[3px]`），一个步进也**绝不能通过任意值重新拼写**
  （`p-[8px]`、`p-[var(--space-8)]`）—— 数值工具类是唯一的做法；
- 不是节奏的关键字后缀依然可以（`ml-auto`、`gap-px` 发丝线）；
- **方括号逃生口是关闭的**，除了有记录的、非节奏的测量/几何表达式：`pr-[2rem]`（关闭按钮的预留）、
  `pl-[1.6em]`（相对于 em 的列表缩进），以及与图标对齐的
  `pl-[calc(0.875rem+var(--space-8))]` / `pl-[calc(1.125rem+var(--space-8))]` 缩进。任何其它任意
  间距工具类都会使守卫失败；添加一个是记录在这里的设计决策，而不是对刻度的静默逃逸；
- 刻度是已定义的原始值集合，因此一个步进**不**要求有消费方 —— 门禁没有可能拒绝保留原始值
  （`32`/`40`/`64`）的孤儿/可达性检查；
- 这些源码根目录下的手写 CSS 声明都在覆盖范围内，包括多行声明与 TypeScript 中的 CSS 字符串字面量：
  `gap` / `padding` / `margin`（及其长写属性）必须使用已声明的 `--space-*` token（或 `0` /
  `auto`），绝不用原始长度或不相关/未知的自定义属性。尺寸、坐标与 box-shadow/border 偏移属于几何，
  不是节奏，不被扫描；一个有记录的非常规光学校正偏移可以通过守卫的 `CSS_RHYTHM_EXEMPT` 允许清单
  保持原始值（`.review-region` 的轨道对 —— `padding-left: 10px` 由 `margin-left: -10px` 抵消，
  零布局偏移）。

与颜色和字体守卫一样，这个守卫存在是因为漂移是**不可见的**：不同于未知的颜色工具类（Tailwind 会
丢弃它，什么都不渲染），脱离刻度的长度总会渲染，因此它在评审中看起来正确，并通过其它每一个门禁。

## 新增或修改步进（Adding or changing a step）

编辑 `spacing.json`，运行 `bun run spacing:generate`，提交重新生成的
`styles/generated/spacing.css`。这就是全部改动 —— 工具类（`p-<n>`、`gap-<n>`、…）与原始的
`--space-<n>` token 都从该来源导出。步进名是不带前导零的规范非负整数，且必须等于其像素值
（`validate()` 强制执行 `"<n>": "<n>px"`）。新步进无需消费方即有效 —— 保留的原始值可以在使用之前
就添加。然而，移除一个调用点仍在耗费的步进会破坏那些工具类（它们变成脱离刻度的）。
