# 仓库开发与一致性脚本

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

跨仓库的开发编排与一致性门禁，它们不属于任何一个产品包。这些脚本校验架构、规格与产物的假设；
它们从不实现产品行为。

## 边界（Boundary）

- **拥有：** 多进程开发启动器；按需的 provider 登录冒烟（`smoke:logins`）；精确版本/目录校验；
  PI 二进制接缝金丝雀；模块依赖/import 边界校验；UI 文本门禁（`check:i18n`）；
  声明式规格接口对 TypeScript barrel 的一致性；按需的 CI 抖动报告（`ci:flakes`）；
  以及把原本相互独立的产品实现组合起来的聚焦根契约测试。
- **公开接口：** 由开发者、Husky 和 CI 消费的根 `package.json` 命令。
- **允许的依赖：** Bun/Node、开发者已认证的 `gh` CLI（只读，仅 `ci:flakes`）、
  TypeScript 编译器 API、用于规范化规格/frontmatter 模型的 `pi-spec-graph/core`，
  以及用于 CommonMark 块结构的 `mdast-util-from-markdown`；对工作区清单与源码树的只读检查；
  各校验所验证的公开包元数据与打包产物；以及当根级 harness 必须在不制造产品依赖的前提下
  证明某个跨包契约时，对真实产品内部构件的仅测试 import。
- **禁止：** 产品运行时逻辑、作为包或特性行为的第二来源、在校验过程中编辑源码，
  以及聚焦根契约测试之外的应用内部 import。根契约测试可以组合实现，
  但不得成为协议来源或产品依赖。

`smoke:logins` 在临时的 HOME/data/pi-agent 沙箱中启动一个一次性宿主（无环境凭据、真实网络）：
它从沙箱中以 `--no-env-file` 和 `toolHomeEnv` 运行，因此仓库 `.env` 与真实 tool home 都不会泄漏进来，
宿主绑定端口 0 并上报该端口。对 `provider.status` 列出的每个 provider，它通过传输线启动该 provider
提供的每种登录方式，走遍 pi 的 select 提示的每个分支，只有当每个流程都抵达其第一个用户步骤
（登录页、设备码或提示）时才通过；任何其他终止帧（包括立即返回的 `success`）、15 秒停滞、
10 秒请求超时、宿主连接中断或零个流程都会让它失败。它在删除沙箱之前等待宿主退出。它会取消每个流程，
因此从不存储任何内容，也不需要账号——它捕获的是破坏登录启动方式的 pi 升级或 registry 变更，
而不是真实凭据是否被接受。它是按需的，绝不是提交或 CI 门禁，因为它依赖各 provider 的实时端点。

`check:boundaries` 在扫描真实工作区之前先运行检查器自身的聚焦测试。它同时强制清单边与静态源码
import，包括 type-only、动态、re-export、CommonJS、import-type、包子路径以及相对跨模块形式。
生成/构建目录被排除，其中包括 Hutch 投射的 `.hutch` SDK 树和瞬态的 `.cottontail-tmp` 加载器。
当前的产品环与启动器边是穷举的：contracts、`pi-delegation` 和 `pi-background-commands` → 无；
`pi-subagents` 与 `pi-dag` → `pi-delegation`；shared → contracts；
server → contracts/shared，加上它既有的内置扩展与 delegation 包；web → contracts、locales，
以及下面列出的浏览器 SDK/扩展条目；CLI → server/shared；desktop → server/shared/contracts/locales；
server、UI 与扩展的 web 半边也可以使用 locales，而 locales 自身没有工作区边；
artifact-tests → CLI/server/shared。产品包不得 import artifact-tests。
未来的组合根必须添加显式规则，而不是静默继承对每个工作区包的访问权。

扩展边界遵循 [[module-thinkrail-extensions]]，从发现的清单注册，而不是一份固定的扩展名列表：

- 在 `apps/*`、`packages/*`、`pi-extensions/*` 和 `thinkrail-extensions/*` 下发现工作区根。
  每个被发现的 pi 扩展都没有宿主工作区依赖。每个 ThinkRail 扩展都被扫描，
  包括试点之外的扩展。
- 仅在存在时注册 `packages/ui` 和 `packages/extension-api`，因此边界优先的交付不需要未来的
  SDK 清单。两者都可以依赖 contracts，但绝不可依赖宿主内部构件。UI 允许按文件使用公开 import；
  extension-api 的浏览器消费方必须恰好使用 `@thinkrail/extension-api/web`，
  而不是其根、server 入口、深层路径或相对路径绕过。服务端消费方使用其公开的 `./server` 入口。
- 扩展清单可以声明其两个半边依赖的并集。它的 `web/**` 源码可以使用 UI、
  extension-api 公开的 `./web` 和 contracts；它的 `server/**` 源码可以使用 extension-api 公开的
  `./server` 及其对应的 `pi-extensions/<name>` 包。任何一个半边都不得 import 另一个半边，
  即使在同一个包内、使用相对路径或使用 type-only 语法也不行。扩展代码不得 import 宿主内部构件。
- `apps/web` 只能消费被发现的扩展的精确公开 `@thinkrail/ext-*/web` 入口；
  `packages/server` 只能用 `@thinkrail/ext-*/server`。根、深层、对侧半边和相对源码 import 都被拒绝。
  它们的清单可以依赖该扩展包，因为清单无法表达子路径边。既有的 server pi 包边保持到每个扩展的
  server 接线交付为止。
- 浏览器源码（web 应用、UI、extension-api 的浏览器半边，以及扩展的 web 半边）绝不可 import
  extension-api 的 server 半边。整个 extension-api 都保持运行时浏览器安全：pi 类型可以描述服务端
  契约，但两个入口都不得以值方式 import pi。浏览器源码与 contracts 同样拒绝 pi 值 import，
  包括未注册为工作区根的外部 pi 包。显式的类型 import/re-export 仍受既有工作区边规则约束；
  type-only import 不能绕过被禁止的模块或源码半边边。

这些是静态字面量 import 检查，不是运行时模块解析器。负向文件系统夹具覆盖与真实工作区门禁相同的
发现与扫描入口点。

`ci:flakes`（`bun run ci:flakes [--days 14] [--repo …] [--workflow ci.yml]`）通过 `gh api` 读取
CI 工作流在一段时间窗口内的测试失败并打印 Markdown 报告：运行、每次尝试的 job，
以及仅失败测试 job 的日志。纯解析器读取以 `##[group]<file>:` 上下文为键的 bun `(fail)` 行
（末尾的 `N tests failed:` 摘要只是重复它们）以及 Playwright 的 `failed`/`flaky` 摘要块。
一次变更是某个 PR——它的 PR 运行与合并队列条目只计一次——或者在没有 PR 时是某个分支。
对每个测试，它统计失败数、Playwright 抖动通过数、不同变更数、合并队列失败数，
以及 SHA 后来通过了同一 job 或整轮运行的失败。测试在以下情形被判为抖动：重跑通过、被标记为 flaky、
3 个以上变更出现失败、或 2 个以上变更出现合并队列失败；其余的被单独列为可能是真实失败。
没有被任何已解析测试解释的失败步骤（build、install、download，或与测试失败并列失败的步骤）
按步骤列出，且一次尝试只有在其中每个失败都是被判为抖动的测试时才算仅抖动。
它把 2026-10 CI 抖动系列背后的手工分析自动化；它是诊断，绝不是门禁，也没有定时运行。
在其他方面绿色的 job 中的抖动通过看不到，因为只读取失败 job 的日志。

`check:spec-surface` 只纳入标记为 `public-surface-checked` 的有效规格。纳入是显式的，
因此新增文字不能静默关闭强制：被纳入的规格必须保留一个裸反引号标识符列表和一个可发现的
TypeScript barrel，否则校验失败。未被纳入的文字接口保持描述性，只由 `--list-skipped` 报告。

TypeScript 编译器解析该 barrel 的有效导出名，包括 type-only 与默认导出以及传递 re-export；
CommonJS 的 `export =` 赋值是模块唯一的 `default` 接口，而不是被赋值值的合成成员。
无法解析的 re-export 是违规，而不是静默不完整的接口。声明的名字集合与有效的名字集合必须在两个方向上
都匹配。文件系统级测试通过 CI 调用的同一 runner 检验纳入、解析与失败行为。

`check:i18n` 在 `apps/web/src`、`packages/ui` 和扩展的 `web/` 半边（测试与生成文件排除）
上强制 [[architecture]] 决策 23。它用 TypeScript AST 拒绝四种形态的字面量 UI 文本：JSX 文本；
承载文本的 JSX 属性（`aria-label`、`title`、`placeholder`、`alt`、`label`、`description`、`hint`
以及 `*Label`/`*Title`/`*Description`/… 属性），包括经由条件表达式、`||`/`??`、`+` 和模板片段
抵达的字面量；`{…}` 内的字面子节点；文本命名对象属性（`label`、`title`、`description`、`detail`、
`hint`、`message`、`reason` 等）中的散文；以及字面量 `toast.*()` 消息或 `errorText()` 回退值。
“文本”指去掉符号后连续两个或更多字母，因此 `⌘K`、`·` 和计数都通过；裸品牌 `ThinkRail`
是唯一允许的词。它是启发式下限，不是证明：从普通函数返回的字符串仍然由评审负责。
它还检查 `packages/locales/en.json`：键已排序、值非空，且每个键（`_one`/`_other` 用复数基础键）
都被某个 UI、桌面或 locales 源码以字符串字面量引用，因此被移除的 UI 不会留下孤立译文。
最后它构建 web 的 TypeScript program，并对任何**渲染期读取全局翻译器**的行为失败
（[[module-locales]] § 渲染时把翻译器当作值读取）：对来自 `@thinkrail/locales/react` 的 `t`、`i18n`、
`currentLocale` 或 `currentTranslator` 的引用，若在渲染根中求值——组件或 hook 函数体、类的 `render()`，
以及它们同步运行的部分：`useMemo`/`useState` 初始化器、`useAppStore` 选择器、`.map` 风格回调、
JSX 函数子节点和 `render*` 属性；对任何同步抵达它们的函数的调用（直接调用、经由命名空间 import、
或作为引用传给此类回调），包括默认参数，且是传递性的；模块加载期调用；以及任何读取它们的
getter 或已注册工具的 `summary`。事件处理器（`on*` 及其他非渲染属性）、effect 和延迟回调
属于事件期，可以通过。夹具测试钉住每种形态，既包括被标记的也包括被允许的。
没有抑制机制；误报在检查器中用测试修复。
