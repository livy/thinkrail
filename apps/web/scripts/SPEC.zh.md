# apps/web 构建期脚本 —— 排版、颜色与间距流水线

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

# `apps/web/scripts`

`apps/web` 的构建期工具。**这里的东西都不发布**：这些模块在开发者机器或 CI 上以 Bun 运行，绝不进入
浏览器 bundle。它们从 `src/` 读取文件，把生成的文件写回 `src/`，并以一个状态码退出。

该目录以同样的方式构建四条流水线 —— 排版、颜色、间距与 provider 字形 —— 外加 React Compiler 普查：
一个以棘轮门禁为基线的诊断。它们之所以在这里而不在 `src/`，是因为它们是*生成器*：它们使用 `node:fs`
与 `node:path`，而这些绝不能到达被浏览器打包的代码。

## 拥有内容（What it owns）

| 文件 | 职责 |
|---|---|
| `typography.ts` | 库：加载 → 校验 → 渲染 CSS。CSS 自定义属性名、语义类名与散文根类名被推导的**唯一**位置。 |
| `generate-typography.ts` | CLI。写入 `src/styles/generated/typography.css` 与 `src/styles/generated/fonts.css`（自托管字面所用 `@import`）；`--check` 在某个已提交文件过期时失败而不写入。 |
| `validate-typography.ts` | CLI。校验 `src/styles/typography.json` 并打印摘要。真正强制的门禁。 |
| `colors.ts` | 库：加载 → 校验 → 渲染。颜色推导（一个调色板别名或一个 alpha 步）被写入的**唯一**位置。 |
| `generate-colors.ts` | CLI。写入 `src/styles/generated/colors.css` —— 角色、外观级效果与 Tailwind 映射；`--check` 在它过期时失败。 |
| `spacing.ts` | 库：加载 → 校验 → 渲染。它强制步长同一性，并且是生成的 spacing CSS 形状与 `--spacing: 1px`（数字即 px）基准被推导的唯一位置；步长长度只存在于 `spacing.json` 中。 |
| `spacing.test.ts` | 依据 schema 钉住 spacing 源校验器的封闭根／元数据结构与值类型。 |
| `generate-spacing.ts` | CLI。写入 `src/styles/generated/spacing.css` —— `--space-<n>` token + Tailwind 基准；`--check` 在它过期时失败。 |
| `providerGlyphs.ts` | 库：**唯一**的 pi-provider-id → 厂商标记映射，外加一个读取器 —— 它只在某个 vendored SVG 是 24×24、仅 path、`currentColor` 字形时才接受它（分层标记保留其逐 path 的不透明度），以及一个为每个标记共享一个常量的渲染器。来源集合：仅开发依赖的 `@lobehub/icons-static-svg`（MIT；这些标记仍是其各自所有者的商标）。 |
| `generate-provider-glyphs.ts` | CLI。写入 `src/chat/generated/providerGlyphs.ts`；`--check` 在它过期时失败。生成的模块与 `styles/generated` 一样被 Biome 忽略。 |
| `providerGlyphs.test.ts` | 钉住每个被映射的标记都存在于 vendored 集合中，且带颜色、尺寸怪异或非 path 的 SVG 被拒绝而不是被拍平。 |
| `generatedFiles.ts` | 每个 generate CLI 对渲染结果做的事：`--check` 报告漂移，否则写入。“过期”的**唯一**定义，因此三条流水线与测试不可能出现分歧。 |
| `generatedFiles.test.ts` | 钉住该定义 —— 内容漂移与文件缺失算过期，CRLF 的工作树不算。 |
| `compiler-census.ts` | CLI。以 logger 在应用、共享 UI 与扩展 web 源码（排除测试、story 与生成数据）上运行 `babel-plugin-react-compiler`，以 `file:line: [fn] category: reason` 打印编译器跳过的每个函数，并打印合计；`--json` 打印完整普查。运行：`bun run --cwd apps/web compiler:census [--json]`。`--write` 把被跳过的函数以 `file fnName` 形式渲染到 `compiler-census.baseline.txt`，不含行号，因此它们上方编辑不会造成抖动。`fnName` 是词法的，绝不用位置：函数被赋值到的绑定（穿过 `memo(…)` 之类的包装调用），否则是它自己的名字，并由其具名外层函数加以限定（`factory.Row`）。同一文件中的两个编译器单元若身份相同会让普查失败，因为行号与序数都无法在它们上方插入内容后存活。`--check`（`bun run check:compiler`，CI）在任何漂移时失败，并指名每个新被跳过或现在能编译的函数。新的跳过要么修好，要么通过重新生成接受（`bun run compiler:census:generate`）。 |
| `compiler-census.baseline.txt` | 已提交的普查基线。 |
| `compiler-census.test.ts` | 钉住某个函数的基线身份能在其之前插入一个兄弟函数后存续、嵌套函数由其所有者限定，以及无法区分的编译器单元被拒绝。 |

`designSources.ts` 拥有共享的采纳守卫与编译器普查源码扫描：应用源码、`packages/ui`，以及每一棵
`thinkrail-extensions/*/web` 源码树，排除测试、生成产物与已安装依赖。同一批 SDK／扩展根在应用样式表中
是显式的 Tailwind `@source` 指令。扩展源码要写出递归的文件 glob：只写一个以 `*/web` 结尾的通配目录并
不能发现嵌套文件，会静默丢掉仅属于扩展的类，例如全屏缩放。浏览器缩放探针检查渲染尺寸，而不只是缩放
标签，以钉住 CSS 真的被生成。每个 token 的 `*:check` 命令既检查生成产物，也检查其对应的采纳守卫，因此
把一个原语移出应用无法隐藏 token 误用，也无法让一个活跃 token 看起来未被使用。

公开接口：`typography.ts`、`colors.ts`、`spacing.ts`、`providerGlyphs.ts`、`generatedFiles.ts` 与
`designSources.ts` 的导出。没有 `index.ts` barrel —— CLI 是由 `package.json` 按名称调用的入口点，而本
目录之外唯一的导入方（`src/styles/*.test.ts`）直接导入该库，这使测试与生成器在同一批函数上可证明地
一致。

## 边界（Boundary）

- **允许的依赖：** `node:fs`、`node:path`、`node:module`（解析 vendored 字形集合）、三个 JSON 源 +
  它们的 schema、以*数据*方式读取的仅开发依赖 `@lobehub/icons-static-svg` 文件，以及另外两处同样作为
  *数据*、绝不作为导入的读取：`src/themes/schema.ts`（供 `colors.ts`，使角色与 `THEME_COLOR_KEYS` 无法
  漂移）与 `apps/web/package.json`（供 `typography.ts`，使某个 `selfHosted` 条目与已安装的字体包无法
  漂移）。
- **禁止：** 对 React、Tailwind、`src/` 或任何 `@thinkrail/*` 包的运行时导入，任何运行时（非开发）依赖，
  以及任何网络或 shell 访问。采纳守卫把源码当数据读；生成器只读它们声明的输入。一个需要这些东西的
  生成器形状就是错的。
- **普查例外：** `compiler-census.ts` 还使用 `@babel/core` 与 `babel-plugin-react-compiler`，并以*文本*
  方式从共享源码扫描中读取手写的 TypeScript，因此把一个组件提取到 SDK 或扩展中不会把它从报告中静默
  移除。它必须把与 `vite.config.ts` 中 `reactCompilerPreset()` 相同的选项传给编译器，并镜像该预设的
  文件过滤，使其计数与真实构建一致。构建期编译器校验与 lint 规则不同（例如 `StaticComponents` 仅属
  lint），因此其计数与 oxlint 的 React 扫描按设计不同。
- **被谁导入：** `apps/web/package.json` 的脚本（`typography:generate` / `:validate` / `:check`、
  `colors:generate` / `:check`、`spacing:generate` / `:check`、`provider-glyphs:generate` / `:check`、
  `compiler:census:generate`，从根 `package.json` 重新导出，外加作为根 `check:compiler` 的
  `compiler:census:check`；`compiler:census` 本身仅限 web），以及 `src/styles/typography.test.ts` +
  `src/styles/typographyUsage.test.ts` + `src/styles/colorUsage.test.ts` +
  `src/styles/spacingUsage.test.ts`。发布的应用中任何东西都不得从这里导入 —— 生成的 CSS 才是接口。
- **写入：** `src/styles/generated/` 与 `src/chat/generated/`，外加普查的 `compiler-census.baseline.txt`。
  两个目录都被提交（因此每次排版、颜色、间距或字形改动都可作为 diff 评审）并在 `biome.json` 中被排除。

## 不变量（Invariants）

- **内置调色板的重置先于 `@theme inline` 块。** `colors.ts` 在我们自己的条目*之前*发出
  `--color-*: initial;`，使 Tailwind 的库存调色板（`bg-red-500`、`text-white`）根本不再是工具类 ——
  否则库存工具类会愉快地编译出来：硬编码、不可主题化，且在评审中不可见。顺序是承重的：在更晚的块里
  发出的重置会把我们的条目一并抹掉。
- **纯别名角色永不重声明自己的调色板变量。** 一个名字*就是*其调色板键的角色（`primary` → `--primary`）
  直接发布该调色板变量，不发 `:root` 行：`--primary: var(--primary)` 是一次计算值时的自引用（无效），
  今天之所以被掩盖，只是因为运行时把调色板写成内联样式，压过了 `:root`。
- **JSON 是唯一来源。** `typography.ts` 机械地推导每个发出的名称，因此新增一个 token、样式或散文系统
  无需在此改动。如果新增一个样式需要编辑本目录，那说明命名规则错了。
- **生成是确定性且幂等的。** 相同输入 → 逐字节相同的输出；没有时钟、没有随机、没有环境读取。这才使
  `--check` 能在 pre-commit、`apps/web build` 与 CI 中作为可用的漂移门禁。行尾是唯一的例外，因为它们
  属于检出而不是内容：`.gitattributes` 把工作树钉到 LF，而每次与已提交文件的比较都经过 `normalizeEol`，
  因此一个 CRLF 克隆不会报告并不存在的漂移。
- **`validate()` 才是门禁，不是 JSON Schema。** schema 是面向编辑器的契约（每个源里的 `$schema` 提供
  补全 + 内联错误）；工具链没有 JSON Schema 校验器。每个校验器在施加 Schema 无法表达的策略之前，先镜像
  其 schema 的封闭根与嵌套对象形状。引用完整性、mono 仅用于代码、`title.card` == `title.dialog`、散文
  元素集匹配、文档标题阶梯，以及 spacing 步长名／值同一性都活在 `validate()` 里。改动任一方契约都意味
  着要检查另一方。
- **生成的文件永不被手工编辑。** 发出的 CSS 头部如此声明，而每条流水线的 `*:check` 命令强制执行它。

各系统本身的设计与理由：[../src/styles/TYPOGRAPHY.md](../src/styles/TYPOGRAPHY.md)（`web-typography`）、
[../src/styles/COLOR.md](../src/styles/COLOR.md)（`web-color`），以及
[../src/styles/SPACING.md](../src/styles/SPACING.md)（`web-spacing`）。
