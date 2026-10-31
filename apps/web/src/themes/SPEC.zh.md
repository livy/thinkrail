# themes —— 随附 manifest 目录与应用

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

浏览器侧的主题引擎：在引导时校验随附的声明式主题 manifest，并在宿主上报时校验用户的主题文件，拥有
由此得到的目录，并原子地解析／应用活动调色板。宿主只拥有不透明的固定 id、加上固定／系统模式与可选的
明暗配对；本模块拥有这些值在这个客户端上的含义，包括操作系统配色方案的观察与同外观兜底。
**新增一个主题 = 新增一个 `*.theme.json` 文件** —— 加到 `bundled/`（一个 PR + 重新构建），或对用户
而言加到 `~/.thinkrail/themes`（实时，[[submodule-server-themes]]）—— 无需改动代码、契约、CSS 或
测试。

随附六个 manifest：标准的 **Dark** 与 **Light**（配置的默认及其配对，未变）、**Minimal Dark** 与
**Minimal Light**（Minimal 界面方案那套安静的中性表面层级、克制的绿色强调色与中性 `bubbleAccent`，
共享标准配对的语法与终端调色板），以及两个带更严格无障碍契约的 High Contrast 调色板。Minimal 只是
用户在 Appearance 里选择的一个调色板，仅此而已：每个主题渲染相同的角色，因此组件绝不因活动主题或其
对比度元数据而分支去省掉某个填充或边框。调色板无法通过 manifest 键表达的东西，是所有主题共同的产品
决策，而不是某个主题的事。

## 边界（Boundary）

- **拥有：** 带版本的 `ThemeManifest` 契约 + JSON schema；随附的 manifest 集合；目录构建（随附 +
  用户）与固定／系统解析；用户主题校验、其逐文件问题列表及其客户端缓存；确定性的配对推导与同外观
  兜底；操作系统配色方案的读取／订阅；原子的 CSS 自定义属性应用；带版本的首帧偏好提示；语义语法变量
  契约；Shiki 的通用 CSS 变量 TextMate scope 映射；以及把同一定义适配到严格桌面编辑器 API 的十六进制
  解析器 + Monaco 颜色键映射。
- **公开接口：** 仅 `index.ts` —— `initializeThemes`（同步引导：随附目录加上缓存的用户主题）、
  `installUserThemes`／`clearUserThemes`／`getUserThemeIssues`／`onThemesChange`（实时的用户主题目录
  及其 `UserThemeIssue` 列表）、`applyTheme`／`resolveTheme`、偏好感知的 `applyThemePreference`／
  `resolveThemePreference`、`deriveSystemThemePair`、`readSystemAppearance`、
  `onSystemAppearanceChange`、`getThemes`、`readThemeHint`／`writeThemeHint`，以及
  manifest／descriptor／preference-result 类型，外加 Shiki 注册及其 `resolveThinkrailShikiTheme`
  严格消费者适配器。
- **允许的外部依赖：** `@thinkrail/contracts`（不透明 id、主题偏好形状与配置的默认）；浏览器
  DOM／media／storage API 与 Vite 的构建期 glob；仅为给通用注册定型而用的 Shiki 类型。
- **禁止：** server/shared/pi；store、transport、panels、shell 或组件状态；自己读取文件或发现主题
  （用户文件只经 `installUserThemes` 到达）；可执行的主题代码；由 manifest 提供的选择器／布局或
  任意 CSS。

## Manifest 契约（Manifest contract）

一个主题恰好是一个 `*.theme.json` 文件。Schema 版本 2 是严格且自足的：id、label/order、明或暗的外观、
普通或高对比的元数据、完整的语义 UI 调色板（只有下面那些派生键可以省略）、全部 16 个终端 ANSI 颜色，
以及完整的语义语法调色板。颜色值是规范的六／八位十六进制；两处选中文本的前景覆盖可以显式为 `null`，
以保留消费者的原生前景。没有继承，也没有部分覆盖。引擎拥有外观级效果、TextMate/Monaco scope 映射与
CSS token 映射，因此这些机制永不泄漏进 manifest。排版、间距、圆角、字体与动效仍是产品 token，而非
主题值。

**manifest 是用户会保留的文件格式，因此其 schema 以兼容方式演进。** 解析器接受所有曾为用户文件发布的
schema 版本（v2 是第一个），并在校验之前用代码把较旧的迁移到当前形状，因此用户保存过的主题永不因升级
而损坏。相应地，在较晚版本中新增的键必须能从每个更早版本已有的键推导出来 —— 正如 v2 的 `header` 等于
v1 的 `content` —— 否则就不能添加；一个调色板确实需要变化的值，以一个派生默认值起步。兼容只向后：
一个声明了比运行中的应用所知的更新的 `schemaVersion` 的文件，或携带它不认识的键的文件，会被报告为
`invalid-theme`，而不是被猜测。

**manifest 提供的是调色板，不是角色。** 它回答*哪种颜色*；每种颜色*用于什么*，是 `styles/colors.json`
中声明的语义层 —— `container-elevated-bg`、`feedback-warning`、`text-subtle` —— 这是组件唯一使用的
层，也是拥有 alpha 刻度的层。正是这种拆分让一个主题可以是一次调色板替换而不改任何组件，也正是本模块
的变量（`--elevated`、`--border-strong`、`--muted`）是内部的原因：从组件直接够到一个变量就绕过了它
所属的角色。见 [`styles/COLOR.md`](../styles/COLOR.md)。

**每个主题可独立变化的角色各占一个键。** Schema 版本 2 把 `header` 从 `content` 中拆出，此前它同时
写入 `--bg-dark` 与 `--surface-content`，从而在每个主题里都把应用头部钉在代码画布上。每个随附 manifest
都让 `header` 等于其 `content`，因此这次拆分没有改变任何一个像素 —— 它只是让一个旋钮存在了。任何未来
的分化也都需要同样的动作：只有当 manifest 有对应的键，一个角色才能在不同主题之间变化。

主控件有一个显式的不透明填充契约：`accentSolid` 是其静止填充，`accentHover` 是其悬停填充，`onAccent`
是二者上的标签。独立的 `accent` 仍是前景／环／边框与色调来源。这让一个调色板能改变不透明控件而不静默
改变每一处强调色调：Minimal 调色板让静止填充与其强调色一致，而标准 Light 保留其更亮的独立实心填充。
危险控件在危险色相上有同样的契约：`danger` 是其静止填充（也是反馈色），`dangerHover` 是其悬停填充，
`onDanger` 是二者上的标签 —— `schema.test.ts` 要求 `onDanger` 在两个填充上都达到 AA，并保持悬停步与
静止填充可区分（≥ 1.15）。`dangerHover` 与 `onDanger` 是在用户文件已经发布之后才加入 v2 的，因此按
上面的兼容规则两者都是可选的，并从每个 v2 文件都有的键推导：`dangerHover` 默认取 `danger` 向黑色加深
12%（每个随附主题都用的那一步），`onDanger` 取调色板自身的 `text` 或 `background`，取在两种填充上都
更易读的那个。随附 manifest 仍显式设置两者。

**manifest→变量名是推导的，不是映射的。** 一个键写入其 kebab-case 名称 —— `borderStrong` →
`--border-strong`，`editorSelection` → `--editor-selection`。`runtime.ts` 在把调色板写入文档根时应用
这条规则，而 `scripts/colors.ts` 在解析某角色的 `from` 时应用同一条规则；两者都不查表，因为那张表*就是*
漂移路径（它的名字 —— `info` 用 `--blue`、`borderStrong` 用 `--border2` —— 早已不再描述所盛之物）。
`colors.json` 中没有 `palette` 区段，也没有生成的 TypeScript：`generate-colors.ts` 恰好产出一个产物，
`styles/generated/colors.css`（角色、按外观的 `effects`，以及 Tailwind `@theme inline` 映射）。让这两份
列表保持诚实的是一致覆盖，而不是映射 —— 除非**每一个** `THEME_COLOR_KEYS` 条目都被至少一个角色占用，
否则生成器拒绝运行，因此一个被加进 manifest schema 却在 `colors.json` 中被遗忘的键无法进入构建，而一个
命名了不存在键的角色也过不了同一道门禁。

**manifest 还必须易读，而不仅是完整。** `schema.test.ts` 在每个静止表面（`background`、`content`、
`sidebar`、`header`、`elevated`、`input`）上强制 WCAG AA，并对瞬态的 `hover` 表面设一个更低的 3.0 下限
—— 后者是我们自己的线而非标准所定，以便一个从别处借来的主题保住其标志性颜色。`accent` 与 `success`
仅免于 `input` 一项：它们都不会作为文字渲染在控件上。另外，`onAccent` 必须在**两个**主填充
（`accentSolid` 与 `accentHover`）上都通过 AA，因此一个主题无法把悬停步压暗到吞掉主按钮自身标签的程度。
仅可读性已被证明不够：High Contrast Light 曾经发布过它的 `hover` 填充 —— 选中／悬停控件角色的调色板
来源 —— 相对其自身侧栏只有 1.05:1，一个不可见的选中项，而所有检查都是绿的，因此测试套件还钉住
hover 与表面的**可区分性**（相对每个静止表面 ≥ 1.15 —— 包括 `content`，因为 PlanPane 与 workbench
背景会在那里承载悬停控件，而 Light 原先 `hover == content` 的调色板让它们不可见）。一个读起来糟糕的
新 manifest 无法合并；推理见 [`styles/COLOR.md`](../styles/COLOR.md)。

**一个 `contrast: "high"` 的 manifest 被要求静止时达到 AAA（7.0）、即使悬停时也达到完整 AA（4.5）**，
`hint` 除外。高对比正是这两个主题存在的全部理由，因此普通 AA 下限对它们是错误的门禁：它会让其中一个
退化成仅仅够用的主题而所有检查仍然绿。这并非假设 —— 当调色板变绿时，High Contrast Light 的 `accent`
从 8.98 掉到 5.17，其 `success` 从 7.39 掉到 5.08，而没有任何东西失败。更严格的下限正是让“高对比”成为
测试套件强制的属性、而不是 manifest 里一个标签的原因。

随附文件由构建期 glob 发现，而不是在代码目录中具名，并在引导时做全有或全无的校验。这些文件是我们自己
的，因此任何无效或重复的 manifest、缺失的配置默认，或一个不含至少一个明与一个暗 manifest 的目录都会
**大声失败**（单元测试在合并前抓住它）。运行时与 JSON-schema 校验一致。

**用户主题文件被要求符合同一 schema，但失败是软性的。** 每个文件单独校验；一个不可用的文件变成一个
`UserThemeIssue`（宿主的读取码、`invalid-theme`、`id-too-long`，或两个用户文件之间的 `duplicate-id`），
其余仍然加载。上面的可读性与 AAA 门禁是我们文件的合并门禁，刻意不施加于用户文件 —— 它们的对比度是
用户的选择。用户的文件名不携带任何含义；其中的 id 才有。

**用户 id 活在自己的命名空间中。** 一个用户主题的目录 id 是 `user.<manifest id>`（因此 manifest id 至多
59 个字符），而随附 manifest 永不能使用 `user.` 前缀。id 持久化在 `AppConfig.theme` 与系统配对中，因此
该命名空间让双方都能自由增长：一个从 `dark.theme.json` 复制而来却未改名的文件只是 `user.dark`，既不
被随附 Dark 遮蔽，也不遮蔽它，而一个我们后来以任意名称随附的主题也永不会静默夺走用户保存的选择。
已否决的做法：按名称预留随附 id（一个新增的随附主题会把已有的用户主题变成错误，并把其保存的配置重新
指向我们的），以及要求作者自己写前缀（常见的复制再编辑路径会失败）。

## 运行时契约（Runtime contract）

`initializeThemes` 在 `main.tsx` 中先于 React 运行；之后只有 `installUserThemes` 会改变目录（随附集合
在重新构建之前是固定的）。shell 在重新应用偏好之前安装每个宿主快照；重新应用即便对未变的 id 也会重新
发布 `data-theme`，因此一个被编辑的活动主题会像换主题一样重建 Monaco/xterm。写入首帧提示时也会缓存该
偏好所指定的用户 manifest（固定 id 与两个配对槽位 —— 至多三个，绝不是整个文件夹），并由
`initializeThemes` 安装它们，因此用户主题在首帧就上色，而不会在宿主应答之前先闪一下默认；下一个快照会
替换它们，而一个低于用户主题协议的已 welcome 宿主会清除它们（`clearUserThemes`）。超过 64 K 字符的
缓存值会被丢弃而不是写入，以一次首帧闪烁换得远在桌面端单值上限之内。这个界限在桌面端很重要，因为该
缓存与所有其他偏好共享原生偏好存储的单值与整文档上限。删除活动主题的文件会像下面任何不可用的 id 一样
解析。固定模式解析 `AppConfig.theme`。系统模式读取 `matchMedia("(prefers-color-scheme: dark)")` 并解析
该外观配置的槽位；缺失、失败或不匹配的 media API 意味着亮色。该查询是普通浏览器与 Windows、Linux、
macOS 上原生系统 webview 的唯一跨平台路径 —— 没有原生外观分支，`prefers-contrast` 被刻意忽略。

一个配置的系统槽位只有在它的 manifest 存在且声明了所需外观时才可用。否则解析会保持不透明的宿主值不变，
并选择该外观中 order 最低的普通 manifest，再退到该外观中 order 最低的 manifest，同 order 时以
label/id 决胜。首次启用时，解析出的固定主题填入它自己的槽位；对侧槽位选择对比度匹配的最低 order 主题，
再走同样的普通／任意兜底。持久化的配对在之后的启用中复用。每个兜底与每个推导出的对侧槽位都只从
**随附** manifest 中挑选 —— 那些通过了对比度门禁的 —— 因此一个碰巧排序在前（一个 order 为 10、标着
"Arctic" 的 Dark 副本）的用户主题永不会成为用户并未选择的主题。

从消费者的角度看，应用是原子的：写入完整变量集、`color-scheme` 与语义对比元数据，最后才通过
`data-theme` 发布生效的 manifest id，因此通用消费者（Monaco/xterm/mermaid）能在该信号之后重建，而不会
观察到半个调色板。与目录无关的 `onThemeSwap` 订阅位于 `@thinkrail/ui/theme`；应用与扩展消费者使用它，
而不是自己手搓 MutationObserver。主题的选择与应用留在此处。选中文本前景在其 manifest 值为 `null` 时
被移除。在系统模式下有一个 media-query 监听器在 `change` 时本地重新应用；它永不写配置，也不影响另一个
客户端，且当偏好或组件生命周期变化时其所有者必须清理它。

首帧提示是固定 id、模式与可选配对的带版本快照 —— 而不是最后一个生效 id。遗留字符串解析为固定模式；
畸形数据变成固定的 Dark 默认。启动时拿当前 media query 同步解析该快照，因此应用关闭期间的日程变化会在
首帧体现。宿主同步的配置总会在首次 welcome 之后与它对齐；在那之前，shell 保留并观察该提示，而不是用
store 的默认覆盖它。存储在有原生 shell 已注入的稳定偏好适配器时用它，否则用 localStorage，后者在
Electrobun 不断变化的回环源下依然存活，又不给启动器任何主题语义。

Shiki 使用一个仅 web 的 TextMate 注册，其颜色是语义 CSS 变量（一个被显式支持的 Shiki 模式）。chat 直接
消费该注册，无需重新高亮。Monaco 让同一注册经过 `resolveThinkrailShikiTheme`，它把每个可用的 CSS 变量
规范化为十六进制，省略不可用的值以便 Monaco 继承的 base 仍可用，并附加 token 拥有的编辑器 UI 映射；
面板提供 `cssColorToHex`，因此 themes 不保留任何回到 `lib` 的依赖。Monaco 从外观／对比度元数据中选择
其 base，绝不依据已知的主题 id，并在一次原子主题切换之后以同一个固定名称替换已解析的注册。因此新增一个
主题永不新增 Shiki 导入、CSS 选择器或编辑器专用目录条目。单元覆盖钉住精确与错误外观的解析、两个兜底层
级、配对推导、无信号时的亮色行为、遗留／带版本提示、media 监听器清理，以及共享语法映射的完整十六进制
解析。

## 推迟事项（Deferred）

开放但未排除：从随附主题做部分覆盖或继承、导入外部格式（shadcn/tweakcn CSS、VS Code 主题）、由扩展
打包的主题（接缝是同一批 `themes.list` 文件上的来源标签，按既有扩展开关过滤）、应用内编辑器，以及面向
用户主题的对比度警告。在那之前，一个用户主题就是一份完整的 manifest。

## 非目标（Non-goals (deliberate)）

用户主题永不替换或遮蔽随附主题 —— 上面那个 `user.` 命名空间就是决策。跟随操作系统的对比度设置，以及
日程／时区 UI 均不在范围内。
