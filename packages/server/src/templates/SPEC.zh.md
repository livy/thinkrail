# templates —— pi 提示词模板目录上的文件 CRUD

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

对 pi 两个受认可的提示词模板目录（全局 + 项目作用域）做文件 CRUD：列出 / 获取 / 保存 / 删除 `.md`
文件，并把 pi 自己的 frontmatter（`description`、`argument-hint`）作为元数据呈现。由 `template.*`
宿主处理器消费；本模块自己不拥有任何 WS 接口面——一个可选的 `cwd` 由调用方在解析出一个活跃工作区
或一个会话前项目的当前 checkout 之后传入，绝不在这里查找。

## pi 事实（pi facts (`@earendil-works/pi-coding-agent`, the catalog-pinned version)）

通过阅读已安装包中的 `dist/core/prompt-templates.js`（`loadTemplateFromFile` /
`loadTemplatesFromDir` / `loadPromptTemplates`）、`dist/core/resource-loader.js`（`dedupePrompts`、
`updatePromptsFromPaths`），以及 `dist/utils/frontmatter.js` + `dist/utils/text.js` 验证——事实来源
优先于假设；pi 升级时需重新验证。

- **目录：** 全局 = `join(agentDir, "prompts")`；项目 = `resolve(cwd, CONFIG_DIR_NAME, "prompts")`
  （`CONFIG_DIR_NAME = ".pi"`，pi 的根导出）——正是 `templateDirs` 的两个字段。
- **Frontmatter 键：** `description`（字符串）与 `argument-hint`（kebab-case；pi 自己的加载器通过
  `...(frontmatter["argument-hint"] && { argumentHint: frontmatter["argument-hint"] })` 把它映射为
  camelCase 的 `argumentHint` 字段）。pi 的加载器不读取任何其他 frontmatter 键。
- **名字推导：** `basename(filePath).replace(/\.md$/, "")`——文件名去掉一个结尾的 `.md` 后缀，只去
  一次（一个带内嵌点的名字如 `foo.bar.md` 推导出 `foo.bar`）。
- **pi 的加载器完全没有名字校验。** `loadTemplatesFromDir` 接受任何满足
  `entry.isFile()`（或一个解析到文件的符号链接）`&& entry.name.endsWith(".md")` 的目录条目——没有
  正则、没有穿越守卫，也没有点开头过滤（pi 会列出手工放置的 `.hidden.md`）。**我们的**按名路径
  （`saveTemplate` / `getTemplate` / `deleteTemplate`）的路径穿越安全性完全由本模块自己的
  `isValidTemplateName` 门禁负责；pi 在那里没有给我们任何可依靠的东西。
- **`loadTemplatesFromDir` 的错误处理分两层、在两个不同粒度上。** 整个 `readdirSync` 调用及其循环
  位于一个 `try { … } catch { return { templates, diagnostics }; }` 之内——一个不可读的目录返回已经
  收集到的内容，而不是把异常抛出函数。独立地，每个文件走 `loadTemplateFromFile`，它现在有*两个*
  分开的内层 `try/catch`——一个包住读取，一个包住 `parseFrontmatter`——任一处失败都返回
  `{ template: null, diagnostics: [{ type: "warning", message, path }] }`，而不是静默返回 `null`
  （上游 PR #9830：一个畸形文件过去会毫无痕迹地消失；自 0.87 起它产生一个资源 diagnostic，pi 自己的
  调用方可以把它呈现出来）。本模块的 `listDir` 只镜像两层*结构*——它自己的逐文件/逐扫描 `catch {}`
  仍然静默吞掉，因为本模块从不调用 `loadPromptTemplates`/`loadTemplatesFromDir`，也没有可填充的
  diagnostics 通道；见下面的“设计”。整扫描包装器正是保护 `listTemplates` 不让一次 EACCES（或类似
  错误）清空每个作用域结果的东西，而不只是保护出错的那个。
- **`parseFrontmatter` 的真实签名**（`dist/utils/frontmatter.d.ts`）：`parseFrontmatter<T extends
  Record<string, unknown> = Record<string, unknown>>(content: string): { frontmatter: T; body: string
  }`——一个普通的同步函数，对 frontmatter 形状泛型（运行时不做校验；这个泛型就是调用方自己的断言，
  正如 pi 自己的加载器信任它那样）。我们使用默认的 `Record<string, unknown>`，并在把某个值当作
  `string` 信任之前用 `typeof` 收窄。
- **无 frontmatter 的文件解析为 `{ frontmatter: {}, body: <normalized content> }`。** 阅读
  `frontmatter.js` 本身：在围栏检测之前，一个前导 UTF-8 BOM 被剥离，换行被归一化（`\r\n`/`\r` →
  `\n`）；如果剩余内容不以 `"---"` 开头，或一个 `"---"` 开头符从未找到闭合的 `"\n---"`，那段归一化
  内容就成为未 trim 的 `body`。只有*成功闭合*的 frontmatter 块才产出被 `.trim()` 过的 `body`。本
  模块根本不读 pi 的 `body`（见下面的“content”），因此那些 BOM/trim/归一化的行为没有一样泄漏进
  `Template.content`——它们只对元数据提取路径有意义。
- **pi 的加载器会合成一个回退 `description`**，来自正文的第一条非空行（截断到 60 字符 + `"…"`），
  当 frontmatter 没有时——用于 pi 自己的 `/` 菜单简介。我们刻意**不**复制这一点：
  `TemplateInfo.description` 在线上是可选的（`contracts/domain.ts`），因此一个没有 `description`
  frontmatter 键的模板呈现出 `description: undefined`（该键被省略），绝不是 pi 从未写到磁盘上的
  人造值。
- **优先级不是在 `loadPromptTemplates` 内部决定的。** 它的 `includeDefaults` 分支只是把
  `[...global, ...project]` 拼接起来，**在该层完全不去重**。真正的优先级稍后在
  `resource-loader.js` 的 `dedupePrompts` 中解析——一个按名字为键的 `Map`，其中为某个名字注册的
  **第一个** prompt 胜出（后续同名条目变成一个 `collision` diagnostic，而不是覆盖）——作用于一个由
  项目配置合并顺序构建的 `promptPaths` 列表，我们没有进一步追踪它（这超出本模块有界调查的范围，而且
  无关：`loadPromptTemplates` / `dedupePrompts` 都没有根导出——密封的 exports 映射只给我们
  `parseFrontmatter` / `stripFrontmatter` / `getAgentDir` / `CONFIG_DIR_NAME`，所以即便我们想，也
  无法调用 pi 的版本）。**这里的“项目遮蔽全局”是我们自己的产品决策**（设计规格 §2.2，“作用域省略
  → pi 优先级：项目胜全局”——与需求书一致），在 `listTemplates`/`getTemplate` 中独立实现：一个按
  名字为键的 `Map`，先插入全局、再插入项目（对已有 key 的 `Map.set` 会覆盖值）——这是 pi 自己
  first-wins 的 `dedupePrompts` 的镜像反面，之所以刻意这样选，是因为“更具体的作用域胜出”正是产品
  规格所要求的行为，而不是因为 pi 的内部实现替我们那样解析。
- **`stripFrontmatter`（也是根导出）本模块不使用。** `Template.content` 是“完整文件文本：
  frontmatter + 正文”（`contracts/domain.ts`），而设计规格 §2.4 端到端确认它：“客户端组装 markdown
  （frontmatter + 正文）；服务端逐字写入它。”因此 `content` 始终是 `readFileSync` 的原始结果——
  绝不是 pi 解析/剥离后的 `body`——而 `stripFrontmatter` 没有字段可喂。（后续某个任务的槽位解析器，
  在 `apps/web` 那边，若想要 pi 的 `body` 形状，将需要它自己的小型非 pi frontmatter 剥离——它根本
  无法导入 pi 的，包括浏览器打包代码在内，按根架构不变量。）

## 设计（Design）

- `templateDirs(cwd?, agentDir = getAgentDir())` → `{ globalDir, projectDir? }`——纯路径算术，无
  文件系统访问。`agentDir` 的默认值是一个**默认参数表达式**，在调用时求值，而不是模块级常量——
  `getAgentDir()` 实时读取 `PI_CODING_AGENT_DIR`（history/SPEC.md 为 pi 的 `SessionManager` 路径
  钉住同一事实），因此在导入时缓存它，会在某个测试（或 e2e seeder）在本模块已加载之后设置该环境
  变量时立刻过期。
- `listTemplates` **每次调用都重读**两个目录——本模块任何地方都没有缓存。这是“/ 菜单新鲜度”规则
  （设计规格 §2.2）：pi 自己的 `session.promptTemplates` getter 经 `ResourceLoader.getPrompts()`
  读取，那份数据在会话加载时填充一次，只有显式的资源重载才刷新——一个**创建时快照**，一旦模板在
  会话中途被保存就过期（通过阅读 `agent-session.js` 的 `get promptTemplates()` /
  `resource-loader.js` 的 `updatePromptsFromPaths` 确认）。一个通过 `template.save` 保存的模板必须
  在任何客户端下次询问时出现（`template.list`、Web 的 `/` 菜单、Templates 设置面板），因此本模块
  从不做记忆化；代价是每次调用几个小 `readdir`，足够便宜。
- `isValidTemplateName` 是**穿越门禁**：在每次调用方提供的 `name` 被 `join()` 进路径之前应用，
  在 `getTemplate`、`saveTemplate` 与 `deleteTemplate` 中一视同仁。它的职责是路径穿越*安全性*，而
  不是命名风格，因此它只拒绝作为单一路径段不安全的形状——空、前导 `.`（一条规则同时覆盖 `.`、
  `..` 和 `.hidden` 式点文件）、名字中任意位置的路径分隔符（`/` 或 `\`）、内嵌 NUL 字节，或冒号
  （在每个操作系统上都拒绝，具体是 NTFS 备用数据流的情况：NTFS 把 `a:b.md` 读作名为 `a` 的文件的
  备用数据流，因此一个冒号命名的模板无法在 Windows checkout 中存活）——并接受其他一切，包括内嵌点
  （`foo.bar`）、大写、空格和 unicode。因此一个在 Linux/macOS 上手工放置的冒号命名文件会从
  `template.list` 中消失，并被 get/save/delete 拒绝；这是刻意的，因为该名字无法在 Windows checkout
  中存活，而 list/get 的一致性成立，因为 `listTemplates` 经同一门禁过滤。这一点重要，因为 pi 自己
  的加载器在从文件名推导名字时**完全不做任何清洗**（见上面的“pi 事实”），所以一个更窄的规则会破坏
  **list/get 一致性**：一个 `listTemplates` 能显示、但 `getTemplate`/`saveTemplate`/
  `deleteTemplate` 拒绝的名字是用户可见的 bug，而不是安全改进。（这个门禁最初使用一条
  `/^[a-z0-9][a-z0-9-_]*$/i` 允许清单正则——安全，但过度限制：它拒绝任何带内嵌点的名字，因此一个
  完全普通、pi 合法的名为 `foo.bar` 的模板会列出却在 get/save/delete 上 404。被抓住后修复了一次——
  允许清单形状在解决错误的问题，即美学而不是穿越。）`listTemplates`（经 `listDir`）**确实**按这条
  确切门禁过滤——不是为了镜像 pi（pi 自己的扫描器没有这样的过滤器，会乐于列出手工放置的
  `.hidden.md`），而是为了让 list/get 一致性在*结构上*成立：复用守卫 get/save/delete 的同一个谓词，
  而不是一个手写的点检查（后者日后可能悄悄漂移），意味着门禁拒绝的任何名字对 `listTemplates` 也是
  不可见的，而不只是无法经它获取。（这个过滤本身也是一次修复：更早的版本无条件列出每个 `.md` 文件，
  于是手工放置的 `.hidden.md` 会出现在 `template.list` 中，然后在 `template.get` 上 404。）
- **不跟随门禁（符号链接包含）：** 上面的穿越门禁约束的是*名字*；这一道约束的是该名字可以*解析到*
  什么。每个按名操作都对目标做 `lstat`（绝不跟随），并把任何不是常规文件的东西——首先是符号链接
  ——视为**不是模板**：`getTemplate` 报告它不存在，`saveTemplate` 拒绝透过它写入（响亮错误，磁盘上
  什么都不动），`deleteTemplate` 报告未找到；`listDir` 在结构上已经跳过符号链接（一个符号链接
  dirent 的 `isFile()` 为 false，而且不会补一次 `stat`）。这是**对 pi 自己扫描器的刻意偏离**，后者
  明确跟随文件符号链接（`loadTemplatesFromDir` 对它 stat 并加载文件目标——见上面“pi 事实”）：pi 的
  加载器是一个只读的本地便利，而本模块是一个*写能力、跨线上的 CRUD 接口面*——跟随
  `.pi/prompts/linked.md → ~/somewhere` 会让 `template.get` 披露目标、让 `template.save` 覆盖它，
  于是一个被检出的仓库可以植入一个链接，把一次例行的模板编辑变成工作树之外的文件写入。代价：一个
  合法符号链接的单个模板，pi 自己的 `/` 菜单会提供它，但它在 ThinkRail 的菜单中不出现——可接受，
  而且在这个门禁存在之前列表行为就已如此。同一规则**向上适用一级**：一个符号链接的 `<cwd>/.pi` 或
  `<cwd>/.pi/prompts` *目录*（仓库控制那些路径组件）会让项目目录对**每一个**项目作用域操作都不可
  穿越——`listTemplates`/`getTemplate` 把它视为没有模板（list 与 get 共享一个谓词
  `readableProjectDir`，因此它们永不互相矛盾），而 `saveTemplate`/`deleteTemplate` 响亮拒绝（写入
  必须显式失败，绝不静默空操作）。没有读取那一半，`template.list`/`template.get` 仍会跨线上披露该
  链接目标的 `.md` 文件——即文件级门禁所关闭的同一个逃逸。**全局**目录刻意豁免：`~/.pi/agent/prompts`
  是用户自有的（能在那里写入的攻击者已经赢了），而点文件管理器经常对它做符号链接。`lstat` 与写入
  之间的空隙是一个只有并发本地进程才能利用的 TOCTOU 竞态——对属主作用域的宿主而言超出范围（这样的
  进程可以直接写目标）。由 `templates.test.ts` 中的符号链接用例钉住。
- **有界列出 + 尺寸上限：** `listTemplates` 是**仅元数据**的（`TemplateInfo`，无 `content`），并且
  无论某个 checkout 往目录里丢了什么，都按文件做有界工作：`readTemplateMeta` 复用不跟随的 `lstat`
  作为尺寸门（超过 **`MAX_TEMPLATE_BYTES`**，1 MiB 的文件被静默跳过——list/get 一致性：两者都不会
  有用地呈现它），并且只读取头部的前 `FRONTMATTER_SCAN_BYTES`（8 KiB）来取 frontmatter——pi 的
  `extractFrontmatter` 把没有闭合围栏的截断块当作普通内容，因此一个病态的 >8 KiB frontmatter 降级
  为“无元数据”，绝不报错也不读整个文件（按名路径仍会看到这样的元数据——这是有界列出的一处有文档的
  不对称，由测试钉住）。**完整文本**只在按名的 `template.get`/`template.save` 路径
  （`Template`）上传输，那里一个超大文件/负载**响亮**失败（读取：在 `readFileSync` 之前；保存：在
  任何东西触碰磁盘之前）——一次直接指名的操作配得上一个响亮的回答，与格式错误的 frontmatter 是
  同一组响亮-对-跳过的非对称。这正是使一个巨大的被检入的 `.pi/prompts/*.md` 无法在每次 `/` 菜单
  打开时阻塞共享的进程内宿主、也无法让单个 WS 响应因每个文件的正文而膨胀的东西。
- `listDir`（`listTemplates` 调用两次的目录扫描）内部有两层失败包含，各自在不同的粒度上镜像一种不同
  的 pi 行为：**(1)** 一个逐文件的读取/解析失败（不可读文件、畸形 YAML frontmatter）被捕获并跳过
  那一个文件——镜像 pi 自己 `loadTemplateFromFile` 的*形状*（它逐文件的 catch 在失败时仍返回 null
  模板；自 0.87 起它们还额外推一个 warning diagnostic，但本模块没有 diagnostics 通道，只是像以前
  一样丢弃该文件）。**(2)** *目录扫描本身*——`readdirSync` 调用及其循环——也被包装，因此一个不可读
  的目录（EACCES，或者确定性地，一个根本被证明不是目录的路径）返回已经收集到的内容，而不是把异常
  整个抛出 `listTemplates`——镜像 pi 自己的 `loadTemplatesFromDir`，它的 try/catch 包住*整个*扫描，
  而不只是每个文件。（层 (2) 本身也是一次修复：更早的版本只有层 (1)，因此一个不可读的目录会直接
  穿透 `listTemplates`，并可能把*另一个*作用域的结果也清空，而不只是出错的那个。）`getTemplate` 对
  一个*被直接指名*的文件不做这种宽容：那里的解析失败会传播，因为调用方明确要求那一个文件，理应知道
  它出了问题（由专门的测试覆盖——与 `listTemplates` 的吞掉相对，这是一处有文档的非对称）。
- `saveTemplate` 在写入任何东西*之前*解析传入 `content` 的 frontmatter。构建返回值本来会在写入之后
  经 `readTemplateFile` 调用 `parseFrontmatter`——这意味着一个语法非法的 `---` 围栏块会先把文件写到
  磁盘、*然后*才抛出，留下一个对 `listTemplates` 不可见（上面的层 (1) 吞掉它）且无法 `get`（那里会
  传播，按设计）的孤儿文件——在磁盘上存在，却无法经本模块到达。先校验把一个被拒绝的保存变成文件
  系统上的空操作：什么都不写，`mkdirSync`/`writeFileSync` 从不运行。成功写入之后的读回原则上仍可能
  失败，但那时只会是文件系统竞态，而不是内容问题——这是该函数不去尝试消除的可接受残留。
- `listTemplates` 的结果按 `name` 排序——`readdir` 的顺序跨平台没有保证，而模板*列表*的每个消费者
  （`/` 菜单、Templates 设置面板）想要的都是稳定顺序，而不是文件系统的到达顺序。

## 边界（Boundary）

- **拥有：** 恰好那两个受认可目录（`TemplateDirs.globalDir` / `.projectDir`）中的文件 CRUD；作为
  穿越门禁的名字校验；把 frontmatter 提取为 `description` / `argumentHint` 元数据。绝不触碰任何其他
  路径——没有对调用方输入的 `resolvePath` / 符号链接跟随，只有在 `name` 通过 `isValidTemplateName`
  之后才 `join(dir, \`${name}.md\`)`。
- **公开接口（barrel）：** `templateDirs`、`TemplateDirs`、`listTemplates`、`getTemplate`、
  `saveTemplate`、`deleteTemplate`、`isValidTemplateName`。
- **允许的依赖：** `@earendil-works/pi-coding-agent`（`getAgentDir`、`CONFIG_DIR_NAME`、
  `parseFrontmatter`——仅根导出）、`@thinkrail/contracts`（`TemplateInfo`、`TemplateScope`）、
  `node:fs`、`node:path`。
- **禁止：** 导入 `workspaces` / `projects`——像 `history` 一样保持不依赖注册表；`template.*` 处理器
  把读取请求的那一个可选 `workspaceId` / `projectId` 位置解析为 `cwd` 并自己传入 `templateDirs`。
  两个 id 同时给出会被拒绝；没有位置信息时保持全局唯一。跨调用缓存列出结果。在 `globalDir` /
  `projectDir` 之外读写任何东西。

## 必须做对（Get right）

- **`content` 是完整文件文本，不是 pi 解析后的 `body`。** `TemplateInfo.content` 在
  `saveTemplate` → 磁盘 → `getTemplate` / `listTemplates` 之间逐字节往返，不受
  `parseFrontmatter` 的 trim/换行归一化怪癖影响——那些只影响元数据提取，绝不影响 `content` 字段。
- **每次调用都是新读取。** 本模块任何地方都没有内存缓存（见上面的“设计”）——这是刻意的，不是疏忽；
  不要在不重新阅读它要满足的新鲜度理由的情况下添加一个。
- **list/get 一致性，双向——是结构性的，不是约定。** `isValidTemplateName` 只排除作为文件名单段
  不安全的形状（空、前导 `.`、分隔符、NUL）加上 `:`（上面的 NTFS 备用数据流案例），绝不排除像
  `foo.bar` 这样普通的 pi 合法名字——所以不要收紧它而不检查它仍接纳 pi 自己的加载器会列出的每一样
  东西（一个更窄的“干净 slug”正则在评审中看起来安全，却会悄悄破坏这一点；它曾经如此，见上面的
  “设计”）。反方向也按构造成立：`listDir` 用这*同一个*门禁函数过滤每个目录条目，而不是一个部分或
  手写的复检，因此一个点开头的文件如手工放置的 `.hidden.md` 被扫描彻底忽略，从一开始就不在
  `listTemplates` 中浮现。门禁与扫描不会漂移，因为它们共享一个谓词——不要为任一方向重新引入第二个
  并行检查。
- **`deleteTemplate` 在目标已消失时抛出**，而不是把一个缺失的文件当作成功的空操作——这是处理器契约
  Task B3 应当建在其上的东西。一个删除请求之所以到达本模块，只是因为某个调用方的 UI 指名了一个
  就其所知仍然存在的模板；如果该文件已经消失，那个视图就是过期的，而 `template.*` 处理器应当把它
  呈现为错误（例如“已删除，请刷新”），而不是为一件没有发生的事报告成功。对过期 UI 而言，响亮胜过
  静默。
