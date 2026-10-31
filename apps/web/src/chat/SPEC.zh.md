# chat —— pi 会话 UI 基础组件

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

chat/agent 会话 UI：渲染 pi 的**规范消息/内容块模型**的**展示型 React 基础组件**、一个
**工具渲染器注册表**（即扩展点），以及 `ChatView`（应用集成层）。全部手写——pi 不提供 web UI，
官方的 `@earendil-works/pi-web-ui`（MIT、**Lit**、在浏览器内运行 agent / "Direct Mode"）是
事件→渲染的规范*参考*，是借鉴对象，但**不**采用（架构与框架同本仓库的宿主运行 pi / 类型化 WS /
React+shadcn 模型不匹配）。构建时即考虑让他人复用/贡献（可作为未来的 `packages/chat-ui` 抽取）。
内置工具渲染器位于子规格 [tools/SPEC.md](tools/SPEC.md)；抽取出来的扩展渲染器位于
[[module-thinkrail-extensions]]。

## 渲染模型 —— 行与渐进式展开（Rendering model — rows and progressive disclosure）

会话记录是 pi 规范的轮次（`types.ts` 中的 `ChatTurn`：user/assistant 是 pi 消息；`system`、
`error`、`retry` 是 web 本地提示；`compaction` 同时承载实时生命周期与 pi 在替换更早消息处留下的
持久化摘要记录），但列表渲染的是**派生行，而不是原始轮次**——折叠会跨越 assistant 消息边界
（pi 每个工具轮次发一条 assistant 消息），因此按轮次的条目模型无法分组。纯函数
**`deriveRows(turns, toolResults, isStreaming, isSpec?)`**（`rows.ts`）按顺序把内容块走成行；
`ChatTurnView` 按行种类分发：

- `user` / `system` / `retry` —— 1:1 渲染器。属于 pi 规范展开 skill 块
  （`<skill name="…" location="…">`）的用户消息渲染为一张**折叠的 skill 调用卡**，而不是暴露完整的
  `SKILL.md`：skill 名称始终可见，`/skill:<name>` 之后附带的任何请求仍作为普通用户文本留在其下方，
  展开后以 Markdown 显示持久化的确切指令。解析来自 `lib` 的锚定 Pi 格式镜像（浏览器代码无法值导入
  Pi）；该展开走共享折叠缓存，因此手动选择能跨虚拟化存活。属于评审上下文包的
  （review context package）用户消息（`reviewPackage.ts` 识别服务端 `packageRender` 发出的
  `<review …>` 头 + `<comment …>` 项——该解析器是这种格式的读取半边，并由单元测试对着渲染器的
  逐字输出钉住）渲染为一张**紧凑卡片**：一句摘要（"Sent 3 review comments on script.ts"）以及
  紧贴其下的 COMMENT 行——没有文件层级：一次发送对每个文件就是一条消息，或者当它点名其 chat 时
  所有文件合成一条消息（chat 卡片自己的 chat 或当前打开的 chat——见宿主的发送路由），而跨越多个
  文件的包则改为在每行前加其文件名（`packageFileLabels`：能区分该包各文件的最短尾部路径，
  完整路径作为其 tooltip），因此前后对比评审的行仍可归属，而不需要一行基本只放一个条目的文件行。
  每行一条评论（`▸ L2 · the remark…`，单行），展开后显示其全文加上原样引用的 `<fragment>`——或者
  当该位置没有源文本时显示评论的 `<locator>` 行——使用等宽字体，并限制高度。属性值与 locator 值
  通过渲染器自身的转义读取回来，因此携带 `" < > &` 或换行符的路径或节点引用可以精确往返。
  一切都从 MESSAGE 本身解析——绝不是从评审快照（下一次评审会替换它）——因此在任何客户端上，
  任何会话记录都能永远回答"发出去的是什么"；评论行的折叠走共享折叠缓存（键为
  `rowId:<content-key>`），跨虚拟化存活。一条**普通**用户气泡（不是 skill/评审卡片）文本超过
  **500 字符**时，一旦 agent 已开始回复它，就折叠为 `line-clamp` 预览 + **卡片内部**的
  `Show more`/`Show less` 切换（在其内边距之内、紧贴消息正文下方，因此 line-clamp 只截断正文自身
  的元素——绝不截断控件）：`UserTurn` 在 `useFold(`${id}:user-collapse`, !agentResponded)` 上折叠，
  所以回退语义是 agent 回复*前*保持展开、回复*后*折叠——共享缓存的"手动切换永远胜过一次回退翻转"
  恰好给出所需行为（发送后立即显示为展开；agent 一产出内容就自动折叠；此后手动 `Show more` 能在
  持续流式期间存活）。共享的 `deriveMessageActions` 分类器从时间顺序的行推导 `agentResponded`
  （该用户行之后存在更晚的 `markdown`/`tool`/`activity`/`divider` 行，**或者**在 `isStreaming`
  时它就是末尾的用户行），每个会话记录集成点都把该状态提供给渲染器。它始终只是客户端视图状态，
  对线上契约没有影响。低于 500 字符时气泡不变。展开状态下，较大的正文会限制高度并带内部滚动
  （`max-h-[60vh] overflow-y-auto`），因此超长粘贴绝不会把该行撑成数千 px 的 DOM 节点；
  并且 `estimateChatRowHeight` 按折叠尺寸（静止状态）估算大用户行，而不是按其完整换行高度——两者
  都可防止虚拟化器超额预留空间、在流式期间留下幽灵空白间隙。重试倒计时携带一个 `source`
  （`turn` = pi `auto_retry_*`；`summarization` = 压缩/分支摘要的 `summarization_retry_*`，
  pi ≥0.81.1）——各流程可在运行中途重叠，每个流程恰好保留一个指示器（重新调度会替换，每个来源
  的结束事件只清除自己那个），`RetryIndicator` 用标签把它们区分开（"Retrying" 与
  "Retrying summarization"）。**`ErrorTurn`** 是常驻的失败提示（provider/模型错误、未恢复的
  `length` 截断，或被拒绝的发送），用共享的 [[web-notices]] 样式绘制：中性填充、错误边框与图标，
  页脚行中的 **Try again**。它**永不折叠**，因此失败的轮次不会看起来像什么都没发生。匹配已知
  模型不可用签名（`model_not_found`、Anthropic 针对 `model:` 的 `not_found_error`、OpenRouter 的
  `No endpoints found`、Bedrock 的模型访问拒绝等）的 provider 错误，会在 provider 原文之上先给出
  一条本地化的"pick another model"提示，因为这些属于账户/套餐限制，重试无法修复。只有当前由结算
  派生的失败才携带 web 本地的 `try-again` 恢复动作；其 **Try again** 按钮通过 `ChatView` 已有的
  立即发送路径发送一条可见的普通 `Try again.` 用户消息。被拒绝的发送、扩展通知以及通用应用错误
  保持不可操作，因为缺失的 prompt 或需要修复的地方可能并不在 Pi 的上下文中。乐观的用户追加会立即
  消耗该动作，而任何后续 `agent_start` 会为其他客户端/自定义消息启动消耗它，因此历史错误永远不会
  重新获得该可用性。实时结算与会话记录水合共享同一个 assistant 失败分类器和动作派生，因此重新加载
  不能把最新的未解决失败变成成功，也不能丢掉其恢复能力；已恢复的历史 `length` 尝试后跟更晚的工作
  不会被重新标记为当前失败。
- `compaction` —— 一条 1:1、会中断折叠的行，有两个会收敛的来源。实时 `compaction_start` /
  `compaction_end` 事件产生 `CompactionNotice`（见 store 规格）：运行中 "Compacting context…"
  （spinner）、完成 **"Context compacted"**（+ 在 pi 的溢出重试继续该次运行期间显示 "— resuming…"，
  + 当结果携带时显示 before→after token 数）、失败时带可操作的错误文本、取消则为一条柔化提示。
  这些状态可通过 `data-testid="compaction-notice"` + `data-status="running|done|failed|cancelled"`
  断言。一次成功的实时结束会请求应用集成的会话记录同步器读取 Pi 规范的摘要加尾部；重连对来自更老
  连接世代的运行时做同样的事。**在宿主摘要仍在流式期间不安装任何快照**，无论该需求来自重连还是压缩：
  Pi 持久化的 `session.messages` 省略了在途的 assistant 部分消息，而修订栅栏无法保护一次在读取开始
  之前就已折叠的更新。同步器等待结算并重新读取，而不是删除该部分消息并清除其关联 id。待定的连接世代
  同步优先，即使同一次读取也同时满足了一个未解决的压缩需求。会话记录读取的瞬时失败按有界退避重试；
  只有耗尽才抛出刷新错误，而新的世代/压缩键会获得全新预算。store 的对账只替换宿主派生的对话状态，
  并保留浏览器本地状态。随后持久化的 `compactionSummary` 就地变成同一行（当仍然适用时，实时 id、
  估算后的计数和实时的 `resuming` 标志都保留），而 Pi 已摘要掉的消息立即消失，而不是只在重新加载后
  才消失。它的 `summary` 点击后展开（`data-testid="chat-compaction"`）。水合/重新打开直接从同一
  持久形式开始。两种形式共享 **"Context compacted"** 标题；只有重新加载后不可用的事实才会消失。
- `markdown` —— 一个非空的 assistant 文本块（react-markdown + remark-gfm + shiki）。assistant 散文
  中的安全工作区相对链接会通过 `ChatTurnView` 已有的工作区文件回调在 ThinkRail 中打开目标；绝对路径
  只有在规范化后位于活跃工作区内时才被接受，而 URL scheme、协议相对 URL、片段，以及不安全/外部路径
  仍保留普通的安全新标签页锚点行为。百分号编码的文件路径在验证前被解码一次，因此编码后的分隔符与
  目录穿越都无法绕过包含性检查。一个仅作用于 assistant 的窄 URL 变换会在验证前保留可识别的 Windows
  盘符锚点路径，包括 Markdown 的百分号编码反斜杠形式；盘符根包含性检查按大小写不敏感比较，同时保留
  被链接路径的大小写。在该变换之前，一个仅作用于 assistant 的 remark 变换器
  （`remarkWrittenPathDestinations`）会在链接或定义的目标包含反斜杠且没有 URL scheme 时，按书写原样
  恢复该目标。否则 CommonMark 会把 `\.` 或 `\_` 读成转义，把 `C:\repo\.thinkrail\x.md` 变成死链接；
  在 agent 书写的路径里反斜杠就是分隔符（架构决策 22）。它从解析自身的文件读取源文本，因为
  `BlockMarkdown` 逐块解析，位置是块内局部的。文件查看器中的仓库 Markdown 保持 CommonMark 语义，
  与 GitHub 的渲染一致。其他所有值都委托给 react-markdown 的默认清洗器，而被拒绝的 Windows 路径
  会在回退锚点渲染前重新清洗。通用的 `Markdown` 基础组件保持 props 驱动，只在 assistant 轮次的
  集成边界上以 `a` 组件覆盖的形式获得该行为；被接受的工作区目标渲染为按钮控件且不带原始浏览器
  `href`，因此替代的原生锚点激活方式无法逃逸进 SPA 回退。该覆盖在其工作区输入不变时保持稳定的组件
  标识：工作台焦点可能在 pointer-down 与 click 之间重渲染一行 chat，而在该区间内替换控件会取消激活。
  一个反引号围起来的 ```mermaid 块通过 `@thinkrail/ext-visualize/web` 的 `MermaidView` 渲染为带主题
  的图（全屏平移缩放，出错 → 回退源码）——在每个 `Markdown` 表面（chat、文件/规格预览）都一致；
  挂载之前它作为高亮源码渲染，因此静态上下文（`RenderedDiff` 的 `renderToStaticMarkup`）会像 shiki
  块那样降级为代码。assistant 文本按顶层块渲染（`BlockMarkdown`）：`markdownBlocks` 用 `marked` 的
  块级 lexer 切分，每个块是一次记忆化的 react-markdown 渲染，因此流式增量只重新解析尾部。各块是
  prose `div` 的直接子节点，用整文档渲染所发出的同一个 `"\n"` 分隔。切分器是逐实例且增量的：它从
  最后一个"其后还有两个块"的已闭合块开始重新 lex。当一个块以空行结尾，或以 ATX 标题、分隔线、
  CommonMark 同样会闭合的围栏结尾时，该块即闭合，除非原始 HTML（一个未闭合标签，或任意行上未结束的
  `<!--`、`<?`、`<!X`、`<![CDATA[`）或未闭合的围栏（也包括列表项或引用内部的）仍可能延续它。每个
  token 的空行检查只看前一个 token 的尾部空白，因此一个长块的开销是线性的。不变量：增量切分等于
  全新切分；块渲染等于整文档渲染，下文所列已知缺口除外；链接或脚注定义（也包括列表项内部的）在其
  所在行结束后，会让该消息在此后整个流式期间整体渲染（半流式的 `[Label]: x` 行会在下一个增量时
  重新检查）；回退情形（未闭合 HTML、惰性行、缩进代码、只有 `marked` 会闭合的围栏、非无损的 lexer
  输出）只会合并块，绝不切块。在 micromark 切分之上选 `marked`：它的块级 lex 比一次 micromark 解析
  快数倍，而切分运行在逐块 micromark 渲染之上；代价是急切打包体积里约 12 KB gzip。`BlockMarkdown`
  的记忆化需要稳定的 `components`，因此调用方要对它们做记忆化。已知缺口——`marked` 与 micromark
  对结构的读法不同、因而块可能与整文档渲染不同的情形：原始 HTML 行（注释）惰性延续一个列表；
  `marked` 结束块而 micromark 继续的惰性行；`marked` 让一个 HTML 块吞掉链接定义；一个引用块与
  缩进代码被 `marked` 折进列表，并切出一个更晚的非 1 起始有序列表，而 micromark 把它读成段落；
  一个带尾随空格的空列表项位于表格之前；列表项中未闭合的围栏后跟 2 个以上空行与一次减少缩进；
  某个没有已闭合块的构造，例如各项通过惰性行一直延续的列表，会在每个增量从起点重新 lex；当块合并
  或消息整体化时，被移动的块会重新挂载，因此代码块会从高亮缓存重新渲染（命中时立即高亮）、Mermaid
  图会重绘，且其中的焦点或选区会丢失；保留下来的块持有较旧流式文本的子串，在 11 万字符的消息上约
  为 1 MB，直到该行卸载。
- **可配置的会话记录行宽（Configurable transcript measure）** —— 由宿主同步的 `chatLineWidth`
  （40–240，默认 120）是一个近似的 CSS `ch` 文本度量，因为 chat 保留其比例阅读字体。`ChatView`
  （store 感知的集成边界）把它应用到一个居中的外层列，该列由每个会话记录行、流指示器和吸顶活动
  面包屑共享；普通行的间距槽位于该度量之外，而用户气泡在其内部仍保持 85% 的最大宽度。
  输入框/头部/队列/历史等外围 chrome 仍按面板宽度。开启 `chatLineWidthBounded`（默认 `true`）时，
  该列以挂载的 chat 面板为上限；关闭时该列保持其配置度量，会话记录视口从其左边缘起横向滚动——
  但右对齐的**用户侧**额外地被钳制到面板宽度（一个以 `data-line-width-bounded=false` 为键的容器
  查询上限），因此用户消息绝不会落到可见视口之外并在左侧被裁剪；只有 assistant/工具内容使用完整
  度量。代码块与表格保留自己的内部滚动。这仅改变呈现：规范轮次与 props 驱动的行渲染器永远不会获得
  设置状态。
- **消息复制（Message copy）** —— 普通用户气泡与**本轮收尾的 assistant 回答**携带一个悬停显现的
  （`group`/`opacity-0 group-hover:opacity-100`）**`CopyButton`**（`chat/CopyButton.tsx`，
  `data-testid="chat-copy"`），它复制完整的消息**源文本**——用户气泡是 `userText(message.content)`，
  assistant 行是 markdown `text`——绝不是折叠预览或任何 UI 外围。只有一轮中**最后一条** `markdown`
  行可复制，而不是 agent 在工具步骤之间发出的中间叙述：`deriveMessageActions` 把一个 markdown 行
  标记为最终行，条件是下一个回合开场之前没有更晚的 `markdown`/`tool`/`activity` 行（见 `divider`）
  （`finalAnswerRowIds`，每个会话记录集成点都以 `isFinalAnswer` 提供）；非最终的 markdown 行渲染为
  普通文本，不带该动作。两者都走**同一套共享布局** `MessageWithCopy`（在 `turns.tsx` 中）：一个
  `relative` 包装层，把动作**覆盖在角色底部角落、与内容同一行**——assistant 回答在左下
  （`left-0 bottom-0`），用户气泡在右下（`right-0 bottom-8`，对齐于气泡已有的垂直内边距之内）——
  绝不在其下方另起一行。用户气泡保持仅含内容：按钮定位在外层包装层上，而不是注入气泡标记里。
  消息内容在动作一侧预留一条水平 `size-24` 区域（最终 assistant Markdown 包装层用 `pl-24`，用户
  气泡用 `pr-24`），而不是一条垂直区域，因此文本与大消息切换不会落在点击命中区之下。最终回答的列表
  标记留在其内容盒内，而不会画回 assistant 的动作区域。assistant 包装层只移除其最后一个 Markdown 块
  的尾部外边距，以便动作与实际最后一行对齐；块间与起始外边距保持不变。`MessageWithCopy` 还携带
  跳转/闪烁与测试所依赖的 `data-testid="chat-message"`/`data-role` 钩子。`CopyButton` 是一个自包含
  的展示型基础组件，通过共享的 `copyText()`（`@/lib`）复制——那是唯一内建降级行为的剪贴板写入路径
  ——只有当它报告成功时才翻转为本地约 1.2s 的 `Copy`→`Check` 图标；它**不**像 `panels/PlanPane`
  那样去碰 store 的 toast，从而让消息渲染器保持 props 驱动。skill 调用卡片与评审包卡片保留自己的
  展开 UI，不携带复制入口。
- `tool` —— 一次**主**工具调用：可折叠的 `ToolCard` 框架（除非注册为 `defaultExpanded` 否则折叠；
  出错自动展开；手动切换胜出），或者一个自持框架的 `"bare"` 渲染器。对一条已死消息
  （`stopReason` 为 aborted/error——pi 从不执行这些调用）的 `"bare"` 调用渲染为出错态，而不是永远
  保持可交互。每次渲染器调用——已注册或回退、常规或主、卡片或 bare——都经过同一个通用的结果内容层。
  结果完成后，每个有效的规范图片块都会追加一个有界卡片内预览，并带明确的进入全屏对话框动作；多张
  图片保持块顺序。这是内容类型行为，不是 `read` 的特例，也不会提升或自动展开该工具。结果字节已在
  会话记录中，因此预览无需宿主抓取，对工作区之外的路径同样可用。可预览块要求非空数据，且媒体类型
  属于 `contracts` 共享的栅格类型（PNG/JPEG/GIF/WebP）；畸形结构与不受支持的媒体类型被忽略，而
  规范内容数组永远不会被序列化成 base64 JSON。
- `activity` —— 一段连续不断的常规工作保持为一个**折叠的外层展开区**，其头部汇总每个原子步骤
  （思考块 + 常规工具调用）。展开它时，首个思考之前的工具保留为直接行，然后每个非空思考块渲染为一个
  嵌套展开区。当该块的第一个非空行是一个完整独立的 Markdown 加粗片段（`**…**` 或 `__…__`）时，
  其折叠头部以模型书写的内部文本取代冗余可见的 `Thinking` 标签，使用该行普通继承的字重、同时保留其
  默认文本颜色，并置于尾部工具/字符元数据之前；摘要在该元数据之前截断，展开后消失，此时展开区保留
  通用标签并包含确切文本。没有该约定的块在折叠时保留通用标签。语义面包屑与辅助标签在任何状态下都
  保持 `Thinking`。其后每次常规工具调用都留在该思考之下，直到下一个思考块或活动边界。这些思考分组
  是外层运行**内部**的兄弟节点，即使跨越 assistant 消息边界也是如此；该层级纯属呈现，绝不臆造 pi
  条目的父子关系。单个原子步骤仍直接渲染。非空文本、主工具以及非 assistant 轮次会中断外层运行。
  只有其末尾实例携带实时动态指示。出错的常规工具**不获得特殊处理**（有意为之——agent 常常会自我
  恢复；`ErrorTurn` 与主工具的错误自动展开是安全网）。
- `backgroundCommandCompletion` / `subagentCompletion` —— 后台工作**在 agent 收到它那一刻**的**结果**：
  一条 `background-command-completion` 或 `subagent-completion` 自定义消息，或者——当 agent 自己拉取了
  一个已完成结果时——末尾的 `get_subagent_result` / `background_command output` 步骤
  （`pulledResults.ts`；每次运行只取第一份回执，且当会话记录中已有该次运行的推送时绝不使用，这样
  在投递修复之前写下的会话记录保留其单行，而不会多出第二行）。起点（一次常规工具调用 / 一张冻结的
  `Agent` 卡片）、实时阶段以及"已完成、尚未查看"的信号位于 Resources
  （[[submodule-web-chat-resources]]）。两者都按 Resources 的行法渲染为**一条折叠行**
  （`BackgroundResult.tsx` 包装 `resources` 的 `ResultRow`，[[submodule-web-chat-resources]]）：
  状态、运行时间、退出徽标或用量计数，以及一行摘要（命令最后一行输出、报告第一行）。
  **默认始终折叠，失败也不例外**——红色状态承载信号，agent 的反应紧随其后；旧版常开的 240px 日志
  读起来像一个没有解释的框。展开后显示有界摘录（转义后的纯文本、自动换行、自然高度、无内部滚动）
  或报告 Markdown，但不含 pi-subagents 的 `Subagent "x" (id) status:` 头。打开动作来自
  `ChatActions.resultOpener`：`ChatView` 在该行被客户端持有时于该行打开 Resources 检查器（快照窗口
  内，或已加载到更远的分页），它不持有的 subagent 回退到会话记录对话框，而命令没有回退（无动作）。
  细节携带 `delivery` 标记的行会先说明 agent 是否已收到它（"Agent received"），还是会在用户下一条
  消息时作出反应（`appended`）；展开或打开它会通过 `ChatActions.markResultSeen` 把该结果标记为
  已见。这些行从不推断当前资源权限。永不折进活动分组。
- `reviewFix` —— 一条 `todo-review-fix` 自定义消息（#363）：计划评审判定的修复请求，由宿主以
  **结构化 `ReviewFixDetails`** 投递给 worker chat（而不是合成的用户轮次）。渲染为紧凑的
  `ReviewFixCard`（`turns.tsx`，`data-testid="review-fix-card"`）——一行摘要
  （`Requested a fix on “<title>” · N findings`）、可选的反馈备注，以及通过共享的
  `ReviewPackageComments`（`ReviewPackageComments.tsx`，即展开行基础组件）呈现为可展开评论的发现项，
  路径/行号已在服务端预先解析。该共享列表与 `ReviewFixComment`→`ReviewPackageItem` 映射
  （`reviewPackage.ts`）被评审包卡片与 `request_review` 判定卡片复用（[[submodule-chat-tools]]）。
  与上文的文件 chat 评审评论卡片不同：那条路径仍是由 `reviewPackage.ts` 解析的 `<review …>`
  **用户**消息（自有 `review-package-*` testid）；只有 todo 修复路径是结构化的。永不折进活动分组。
- **回合开场（Round openers）。** 一个回合在用户 prompt 处开场，*或者*在某个自行唤醒 agent 的
  入站事件处开场——一个后台结果或一个 `reviewFix` 请求（`rows.ts` 中的 `opensRound`）。在此改动之前，
  已结算回合之后到达的结果会静默并入该回合：该回合的分隔线消失，随后又出现在反应之下，而整个后台
  等待时间被算进它的耗时。现在，更早的分隔线留在原处，反应获得自己的分隔线（以该事件的消息时间戳
  计时），而开场永不自行闭合一个回合（被用户停止且无反应的结果不会得到分隔线）。同一边界也驱动
  `deriveMessageActions` 与回执匹配。由持久化的 `delivery` 标记决定哪些结果是开场：`run`
  （投递进一个已在运行的运行中——常见情形，因为结果在 agent 的下一步到达）留在其回合内，且不会把
  它之前的叙述终结为最终回答；`wake` 以及没有标记的结果（较旧的会话记录）会开一个回合；
  `appended`（在不唤醒的情况下显示——被停止的结果，或用户停止父级时被扣住的结果）未获回应，
  因此回合收尾线排在其前面。
- `divider` —— 回合结束摘要（`TurnDivider` + 纯函数 `turnDivider` 派生器），在回合结束的瞬间锚定：
  耗时、工具调用计数，以及该回合写入的文件，按**两个徽标块、按所属工具切分**——"N specs" 与
  "N files changed"。该切分是一次**划分**（一个路径只落在恰好一侧，绝不重复计数），在派生器中由注入的
  `isSpec` 谓词计算——store 的 `specPathMatcher` 作用于工作区规格图谱，再加上 `spec_create` 的目标，
  后者按构造就是规格，即使图谱快照尚未跟上。为何重要：一个规格常常是**被 git 忽略**的草稿
  （`.thinkrail/context/`），因此把它算作"已变更文件"会把用户深链到一个结构上无法显示它的 Changes
  视图。现在每个徽标块都路由到拥有该产物的面板。**一个产物 → 该徽标块就是直接深链；多个 → 它是一个
  展开区**，就在这里、在会话记录中把那回合的集合展开成列表（`ArtifactChip` + `ArtifactList`），
  每行深链一个路径。该集合保留在 chat 中、而不是浮在面板之上，是有意为之：它属于*这个回合*，而面板
  显示的是*当下*——几天前的回合会点亮此后已经变化的行（或者，对 Changes 来说，根本已不在 diff 中）。
  这也让计数保持诚实：点击 "5 files changed" 绝不会只悄悄呈现第一个，而各处理函数只接受恰好一个
  路径，因此下游无需猜测用户指的是其中哪一个。
- 这两个徽标块是一个**开关，而不是两个独立折叠**：至多一个列表打开，选择另一侧会替换它，再次选择已
  打开的那一侧会清除选择。该不变量是*结构性*的——分隔线保存**选中的键**（`useSelection`，每条分隔线
  行一个条目），因此不存在两者同时展开的状态。展开还**揭示所属的单例侧工具**（`onReveal` → store
  的与排布无关的工具揭示意图），而不暴露任何路径，这正是让这一对读起来像在 Specs 与 Changes 之间
  切换的原因；关闭等于"算了"，并让工具停在用户最后放置的位置。
- **宿主的轮次回执优先于工具参数。** `deriveRows` 还接收**本 chat** 已记录的 agent 运行
  （`ChatTranscript` 中按 `sessionId` 过滤的 `store.turnsByWorkspace`——一个工作区的其他 chat 也会写
  文件）；`matchTurnReceipt` 按该运行的开始时间把它们与回合连接，从回合的开场点到（不含）*下一个*
  开场点。两个时间戳都来自宿主，而 pi 在开始该次运行之前给用户消息打时间戳，因此这里不该有任何时钟
  偏移容差：把区间向后移会在 prompt 相隔不到一秒时偷走回执。规范的用户回显会替换乐观消息，同时保留
  其行标识与附件标签，因此即使浏览器时钟不同，实时回执也使用宿主时钟。当某个回合有回执时，文件徽标块
  列出*那些*路径并带每文件 `+/−` 计数，读作 `N files changed · +a −r`——工具参数会漏掉 shell 写入、
  格式化器与重命名，而快照才是 agent 实际留下的东西（见服务端 `turns` 规格）。合并回执之前，工具路径
  会对照工作区规范化；`spec_create` 的分类能在图谱刷新待定期间存活，因此一个规格绝不会同时出现在两个
  徽标块里，也不会以绝对路径和相对路径重复出现。一个回合内的多次运行（一次 steer、一次重试）会合并成
  一份回执，覆盖从首次运行的基础树到最后一次运行的头部树，并使用自己的作用域 id（`first..last`），
  因此它绝不会只与最后一次运行共享评审标签页。文件徽标块旁会出现一个 **Review turn** 徽标块并调用
  `onReviewTurn(scope)`，`ChatTranscript` 把它路由到 `requestChangesView(workspaceId, null, scope)`：
  Changes 面板采用该回合作用域并为它打开评审标签页；该徽标块的 `+a −r` 是对它所计文件的求和（规格
  有自己的徽标块），而多运行回合的每文件统计是该路径在最后一次运行中的值——评审标签页显示真实的
  首基准 → 末头部 diff。没有回执时（较旧的宿主、零变更运行）分隔线的行为与从前完全一致；快照此后被
  清理掉的回执仍会渲染（客户端无从得知），而清理只在点击 *Review turn* 时才显现——作为评审标签页
  作用域感知的 `UNKNOWN_COMMIT` 提示。

行/步骤 id 在流式快照之间保持稳定（外层运行的第一个原子步骤 id、每个思考块的消息锚定索引，以及每个
工具自己的 id——pi 只追加，从不重排），因此折叠状态能跨重新派生与虚拟化存活：**每个折叠表面（外层
活动分组、嵌套思考分组、工具行、`ToolCard`、分隔线的多产物徽标块）都把手动切换记录在共享的
`foldState` 缓存中**（`foldState.ts`，以行/步骤 id 为键。该模块之上有两个 hook：**`useFold`** 用于
独立布尔值，**`useSelection`** 用于单选组——即分隔线的徽标块，它们在 `${rowId}:artifacts` 下保存
*选中的键*，而不是每侧一个布尔值，因此"只有一个列表打开"不可能被违反；`AskUserQuestionCard` 的
模式见 [[module-ext-ask-user-question]]；有意永不清除——增长由手动切换的数量界定）。手动切换永远
胜出——胜过自动展开默认值，*也*胜过虚拟化重新挂载。`FoldGeometryProvider` 是会话记录显式的前/后
变更接缝：每个折叠头部都通过 `useFold` 注册一个由行派生的稳定 id，包括 `Collapsible` 与回退驱动的
自动展开/折叠。它把分离锚点的稳定化路由到 `ReadingBandController` 的单一运动通道，并从变更行的实测
几何刷新。修正落在变更的第一帧（该运动的帧，行的 resize 观察作为兜底），绝不在 React 提交中：提交时
写 `scrollTop` 会在 Virtuoso 的 ResizeObserver 测量变更行之前就到达其同步滚动处理器，于是它按过期
尺寸开窗——一次视口上方展开清空了列表，并在一帧之内重新挂载了每一行，包括读者所在的锚点。在该第一帧
或兜底写入之前，修正始终处于待定状态：原生输入或一次更新的折叠变更若先到，只会应用一次（因此更新的
变更接受一次同步的、测量前的写入用于重叠部分）。此后不再有待定状态，因此被再次打断的恢复式稳定化
绝不会在读者移动之上重放，而该移动的滚动事件尚未被处理。完全在屏幕之外的展开区按其自身的行高差保留
可见的会话记录。若一个吸顶面包屑折叠了包含读者视口的展开区主体，该内容不再有存活的锚点，于是折叠后
的原始头部在视口边界处取而代之。读者输入或一次更新的揭示只会取代该稳定化的滚动腿，无法把在途的跑道
释放搁死。

**消息顺序投影（Message-order projection）。** `deriveRows` 保持规范与时间顺序。纯函数
`projectRows(rows, chatMessageOrder)` 在用户行处划分该序列（用户行之前的一段提示是其自己的组）；
最旧优先返回原输入，而最新优先反转组顺序**以及**每组的顶层行：`1,2,3 | 4,5,6 → 6,5,4 | 3,2,1`。
稳定的行 id 逐字保留，因此折叠、闪烁、工具状态与历史锚点不会分叉。投影绝不深入单行内部：Markdown
段落/代码、工具卡正文、评审包评论，以及 Activity 行自身的展开层级都保持其语义顺序。Virtuoso 与 DOM
遍历消费投影后的行；Pi 轮次、持久化、流状态，以及一切非呈现的派生都保持时间顺序。
`chatPreferences.ts` 拥有该封闭偏好及其最旧优先的默认值，并在 React 挂载之前水合 store。浏览器客户端
读/写一个带宿主限定的 localStorage 键，并在同源标签页之间同步该键；原生 shell 可在其稳定的
后端特征/窗口标识下注入相同的窄字符串存储适配器，使动态回环端口在重启时无法抹掉该偏好。它绝不进入
`AppConfig`，因此选择最新优先无法改变另一个浏览器、设备、宿主或原生窗口。同一个持久化接缝还拥有
**流式响应移动（Streaming response movement）**，这是单个 `{ settle, trigger }` 客户端本地偏好，
而不是第二条适配器/订阅路径（Trigger 是响应边缘触发一次步进的位置；Settle 是每一步把它放置到的位置）：
两个值都用 5 点步长，Settle 为 25–90，Trigger 为 35–100，间隙至少 10 点，默认为
`{ settle: 75, trigger: 100 }`。无效的存储会原子地回退到默认对。它同样绝不进入 `AppConfig`，也不跨过
线上契约。

**吸顶活动面包屑（Sticky activity breadcrumb）。** 当会话记录顶部可见内容仍处于已展开的
Activity → Thinking → 工具展开区之内、而其原始头部已滚到视口之上时，一条不透明紧凑行会覆盖在滚动容器
之上，显示该活跃的根到叶路径。各片段只有当自己的头部越过顶部后才加入，在兄弟/结束边界处或折叠时
离开，并包含活跃的叶子工具。片段标签会滚动并聚焦其位于吸顶行正下方的原始头部，而不改变折叠状态；
其独立的 chevron 通过既有折叠状态来源写入。轨迹始终是一行：元数据先于名称截断，在窄面板下则保留
最外层与活跃片段，并把中间祖先压缩为 `…`。它绝不重排会话记录内容，不制造并行的导航/折叠状态，也不
扰动 Virtuoso 的模式感知最新边缘、阅读带与跳转到消息行为。根到叶标签与 chevron 在一个带标签的导航
区域内是各自独立的键盘目标；视觉进入/退出遵循减弱动效设置。

## 扩展点 —— 工具注册表（Extension point — the tool registry）

`toolRegistry.tsx` 就是**那个扩展点**；一个工具有两个解耦的侧面，由**工具名**连接：
**能力**注册到服务端的 pi 会话（自定义工具或 pi 扩展/skill），**呈现**则注册在这里。注册表 `Map`
与回退渲染器保持应用本地；`extensions/registerWebExtensions()` 在应用启动时（`main.tsx`）、在任何惰性
视图加载之前，把来自 `extensions/registry.ts` 的有序描述符喂给它：计划与评审面板从注册表的
`interactive` 注册派生交互状态，因此注册表必须在读取第一份会话记录之前就完整，而不只是在 chat 视图
挂载之后。内置应用渲染器保留其本地副作用注册（`tools/register`，随 `ChatView` 加载）。渲染契约
（`ToolRenderProps`、`ToolStatus`、注册元数据）以及纯参数/结果辅助函数来自
`@thinkrail/extension-api/web`，而不是 chat 内部。`Markdown` 从 `@thinkrail/ext-visualize/web` 具名
导入 `MermaidView` 以支持围栏 mermaid；shiki 与高亮的 `CodeBlock` 仍属应用本地。一次注册是：

- 一个**渲染器**（专用的卡片正文；`ToolRenderProps` 携带 `toolCallId`/`args`/`result`/
  `status`/`workspaceRoot`/`streaming`，外加一个可选的、由 shell 注入的 `onOpenFile` 回调——足以保持
  props 驱动；通用调用接缝会一致地装饰其已完成的规范图片结果），外加可选的
- 一个 **`summary`** —— 供折叠头部与活动步骤行使用的纯函数一行摘要，
- 一个 **`chrome`** —— `"card"`（默认，`ToolCard` 框架）或 `"bare"`（自持框架；用于
  `ask_user_question` 这类交互/主工具），
- **显著度元数据** —— `prominence`：`"routine"`（默认，未注册工具也算——进入外层 Activity 运行，
  位于首个思考之前，或位于当前嵌套 Thinking 展开区之下）或 `"primary"`（逃出折叠；`"bare"` chrome
  **无条件**隐含它，即使显式写了 `prominence: "routine"`——自持框架的渲染器无法活在折叠的步骤行里，
  因此错误注册绝不能静默破坏折叠），以及 `defaultExpanded`（主卡片在完成后渲染为展开，例如
  `visualize`）。都通过唯一的 **`resolveProminence`** 接缝读取——日后可在此插入按用户的覆盖映射
  （设置）。

每次注册表读取（渲染器、摘要、chrome、显著度）都经过一个解析器：精确工具名，否则最长应用本地前缀
注册（`registerToolRendererPrefix`——今天只有 `mcp__`，因为 pi 按服务器命名 MCP 工具），否则默认。
精确名永远胜出，空前缀永不匹配，而扩展描述符只按精确名匹配。

未注册工具回退到 `DefaultToolRenderer`。需要在运行中途获取用户输入的工具走两条路之一：

- 扩展 UI 桥（`pi.extensionUi` → 下文 `ExtUiPrompt`）；
- 对富内联卡片，则从其 `toolCall` 参数渲染，并通过仅 props 的 **`ToolRenderProps.interaction`** 回复。
  `ToolRendererBody` 通过 `useToolInteraction`（`toolInteraction.ts`）填充它，计划页对其卡片做同样的
  事。该适配器读取 `ChatActions`、`interactionState` 与折叠几何上下文，并为每个渲染器注册为
  `interactive` 的工具（`isInteractiveTool`）填充 `interaction`——它不按名字识别任何工具。其
  `state.accepting` 只在宿主的 `needsInput` 点名了这次调用（`awaitingToolCallIdOf`，携带在交互上下文
  中；v93 之前宿主的 `kind: "question"` / `question:<id>` 在一个兼容窗口内以同样方式读取）或尚无任何
  宿主状态到达时为 true，因此宿主并未在等待的调用（另一个包的同名工具、对已关闭工具的调用）无法提交
  响应。`respond(value)` 通过传输层的 `respondToTool`（`session.respondToTool`，或低于
  `TOOL_RESPONSE_PROTOCOL_VERSION` 时已退役的 `session.answerQuestion`）把值原样转发；工具自行校验
  其形状。渲染器自己从不读这些上下文。

示例：`ask_user_question` 流程见 [tools/SPEC.md](tools/SPEC.md)。

**扩展对话框在 chat 内部提问，绝不在模态框中**（`ExtUiPrompt.tsx`）。每一个 pi 的
`select`/`confirm`/`input`/`editor` 请求都渲染为共享 `AttentionLine`
（`@thinkrail/ui/attention-line`："Your input is needed · Agent is waiting"，`ask_user_question` 也
使用它）之下的一条内联提示：select 是按扩展自身顺序每个选项一个按钮，且没有额外的 Cancel
（一条 chat 消息就是逃生口：宿主在接收用户消息时取消所有待定对话框，提示的页脚提示也这么说——
"Sending a chat message cancels this request."），confirm 是 Cancel/OK，input 与 editor 是一个字段
加 Cancel/Submit。标题第一行是问题，并点明该区域及其字段（`aria-labelledby`，提示作为
`aria-describedby`）；后续行是弱化的补充文本，因此宿主无需添加契约字段就能补充上下文。选项按钮
（`Button outline m`，按钮系统唯一有记录的换行例外）会对长标签换行，而在小于 28rem 的提示下
（`@container` / `@max-md`）每一行都堆成一列，因此不会有恰好并排的两个选项（MCP 顺序让 Deny 离
Always allow 最远）。它绝不从输入框抢走焦点。**到达：** `ChatTranscript` 在虚拟化之外保留一个始终
挂载的 polite 活区，并在每个新请求之后隔一帧填入（"Your input is needed: <question>"），每个请求一
次；锚定提示的首次挂载会自我揭示（`revealChatElement`、`automatic-attention`、`block: "nearest"`，
位于面包屑内缩之下；首次指针/按键/滚轮即让位），之后的重新挂载则不会（`PendingExtUi.claimReveal`）。
被虚拟化掉的行得不到揭示，与 `ask_user_question` 相同；活区与需输入标记仍会告知用户。**边界：** 停靠
的提示上限为 chat 高度的 60%（`max-h-[60cqh]`）；其主体（问题、字段、select 选项）滚动，而
Cancel/Submit 与提示保持不动，editor 上限为 40%。退出时它遵循 `ask_user_question` 的交接
（[tools/SPEC.md](tools/SPEC.md)）：作答会卸载被聚焦的控件，因此当提示仍持有 `:focus-visible` 焦点
时，它把焦点交回输入框（`ChatActions.focusComposer`），而不是把焦点丢在 `<body>` 上；点按/点击作答
不改变焦点。`ChatTranscript` 拥有会话的 `pendingExtUi`、其 `session.extUiReply` 与
`PendingExtUiContext`。带类型的 input/editor 草稿位于那个按请求的 `PendingExtUi`（`draft()` /
`rememberDraft`）中，而不在提示里：把分组展开或折叠、虚拟化与从停靠转为卡片都会重新挂载提示，草稿
必须能在这些情况下存活；一次回复或忽略会结束该请求及其草稿。草稿不会跨 chat 视图重新挂载持久化。
`toolCallId` 出现在行中（`rowsHaveToolCall`）的请求**就在那个工具上**提问：工具外壳——`ToolCard`、
常规步骤行，以及 bare 工具的包装层——在其头部下方渲染 `ToolPromptSlot`，与该工具自身的折叠状态无关，
因此没有任何渲染器需要提示代码（MCP 审批也包括在内）。携带 `toolInput` 的请求把它**列在提示框架
内部**、问题与选项之间：每个参数一行 `ToolArgList`（与 `McpArgs` 共享的 `key / value` 列表），自动
换行，上限 40vh 并带自己的滚动——绝不截断，绝不藏在 "Show more" 之后——因此被批准的内容就在用户作答
的框内，无论提示是锚定还是停靠。工具卡片保留读者的折叠状态。*已否决：*自动展开等待中的卡片——
参数会落到提示之外，而会话记录并不持有的调用（MCP `codemode` 的嵌套调用）根本没有卡片。折叠的
Activity 或思考分组只在其头部下方显示等待中的步骤（`AwaitingStep`），作答后折回；它从不写入折叠
状态，因此读者的手动折叠得以保留。其他任何请求（无锚点，或会话记录不持有该锚点）停靠在会话记录
之下、输入框之上，位于虚拟化列表之外，因此在两种消息顺序下都可见。*已否决：*对非 MCP 扩展保留
模态框——每个对话框都有同样的"奇怪弹窗"问题，而锚定提示恰好显示正在被批准的内容。

## 交互接缝（Interaction seams）

- **会话记录/外壳拆分** —— `ChatView`（对外导出的集成组件）拆出一个与高频变动隔离的
  `ChatTranscript` 子组件，由它拥有逐增量的会话记录渲染——（通过 `useShallow`）订阅它所据以渲染的
  流式运行时字段（turns、toolResults、currentAssistantId、toolResponses、turnIdByMessageIndex、
  isStreaming、settlementTick、hostState、syncedConnectionGeneration、eventRevision）——并拥有滚动
  容器与行——外加一个 chrome 外壳，只读取少量粒度化、很少变化的字段，因此文本/思考/工具增量不再重渲染
  头部/输入框/队列/对话框；完成确认、位置揭示与提问焦点作用域活在子组件中（属于内部重构；
  `ChatActions`/`InteractionStatesContext` 由 `ChatTranscript` 提供）。外壳仍读取
  `turns`/`toolResponses`/`toolResults`，但只通过浅比较的派生选择器（最近 prompt、计划速览），它们
  在流式增量间保持稳定，因此外壳不会因它们而重渲染。
- **结构化工具文件导航** —— `ChatView` 接受来自外壳的可选 `onOpenFile(path)`，并在每条
  `ToolRenderProps` 路径上（常规、主、卡片或 bare）传递它。共享的工具文件基础组件只为非空相对路径、
  或由 `workspaceRoot` 包含的绝对路径提供该动作；URL、逃逸的相对路径与外部绝对路径保持为可选文本。
  渲染器只从显式的工具参数/细节中取得候选——绝不用正则去猜 Bash 输出、代码或散文中的路径。独立渲染器
  没有回调，因此保持惰性。该回调的预览槽语义属于 [[submodule-web-panels]]。
- **工具文件预览** —— `ChatTranscript` 用 [[module-resource-renderers]] 的 `ResourceScopeProvider`
  包住它的行列表，值为 chat 的 `{ workspaceId, sessionId }`，并按这两个 id 记忆化（列表上方一个
  provider，与折叠几何相同）。随后工具卡片可对工作区路径渲染 [[module-resource-renderers]] 的
  `ResourcePreview`，其评审发送会路由到这个 chat——渲染器绝不通过 props 获知工作区或会话 id，chat
  也绝不导入面板。
- **`ChatActions`** —— 一个 React 上下文（由 `ChatTranscript` 提供，独立使用时为 `null`）：渲染器
  借以**回话**给 agent——或请求集成层打开某物——而无需导入 store/传输层。目前有：
  `respondToTool(toolCallId, response)`——宿主拒绝（未知/已回答/已被取代的调用）时它 reject，而调用方
  拥有失败 UX——外加 `focusComposer()`，供*自行*了结的渲染器使用：它会卸载用户原本所在的控件，否则
  焦点会落到 `<body>` 上并吞掉之后每次按键（与历史浮层关闭时重新聚焦所避免的丢失是同一情形）。只有
  卡片自己的回复路径调用它，且只在卡片仍持有焦点时调用。另有
  `openSubagentTranscript(childSessionId)`——subagent 卡片的会话记录链接（无 provider → 卡片隐藏
  该动作）——以及 `markResultSeen(target)`，由后台结果行上报用户已打开它。可选的 MCP 动作——
  `readMcpOutput(toolCallId)`（本 chat 的 `mcp.readOutput`）、`openMcpSettings()`（Settings › MCP
  servers）与 `openProviderSettings()`（Settings › Providers，用于由 provider 支持的身份验证）——由
  `ChatView`（经 `ChatTranscript` 的 provider）只在支持 MCP 的宿主上提供；没有它们时（较旧的宿主、
  计划页的 provider，或 `null`），MCP 卡片隐藏 Full output 与 Sign in
  （[tools/mcp/SPEC.md](tools/mcp/SPEC.md)）。可选的 `findModel(ref)` 在全模型目录（`ChatView`
  基于 `useModelCatalog`，包含隐藏模型——运行可能用到其中之一）中解析一次运行的 `provider/id`；
  没有它时，运行标签显示原始引用。
- **运行模型标签（Run model label）**（`ModelEffortLabel.tsx`）：`ModelEffortPill` 是模型选择器
  触发器的内容（glyph · 名称 · 力度条与等级词），抽取出来使 `RunModelLabel` 能像输入框药丸那样精确
  渲染一次委派运行的模型：经 `ChatActions.findModel` 得到目录名，否则显示原始 `provider/id` 及等级词
  但没有力度条（没有可放置它的模型刻度）；原始引用也是它的 tooltip。选择器自己的
  `thinking-selector` test id 仍归它自己。没有 `thinkingLevel` 的运行只显示模型；没有 `model` 的则
  什么都不显示。
- **Subagent 会话记录视图**（`SubagentTranscriptDialog.tsx`——一个集成文件，与 `SkillsDialog` 同类）：
  `SubagentTranscriptPane` 用同一套基础组件（`messagesToRuntime` → `deriveRows` → `ChatTurnView`）
  渲染一个隐藏子会话的**只读**会话记录，经 `subagent.getTranscript` 抓取，键为
  `(workspaceId, parentSessionId = 该 chat, childSessionId)`；同一文件中的对话框把该面板包成浮层，供
  subagent 卡片的 `ChatActions.openSubagentTranscript` 使用，而 Resources 检查器把该面板作为其
  subagent 细节内嵌。两者都在 `null` 的 `ChatActions` provider 下渲染，因此内部任何东西都无法回话
  （嵌套的会话记录链接也不可能存在）。活性来自**宿主**，随每次响应给出：`subagent.getTranscript`
  携带该次运行当前的注册表 `status`（一旦宿主不再知道该运行就缺失——重启、dispose）。挂载的面板在
  同一时刻只保留一次读取在途，仅在收到一个响应之后、且状态仍为 queued/running 时才调度下一次约 2.5s
  的轮询——绝不依据本 chat 自己的运行时，因为它冻结的后台确认无法区分一个存活的运行和一个因重启而
  丢失的运行。终态/缺失状态，或线上契约的永久 `SUBAGENT_TRANSCRIPT_NOT_FOUND`，都会停止；普通传输
  失败在面板仍挂载时以有上限的退避重试。轮询快照以子会话作用域的 id 水合（由每条持久消息的
  角色/时间戳/索引派生），因此仅追加的刷新能保留行标识与手动折叠，而不是重新挂载会话记录。在整个运行
  期间、完成之后，以及宿主重启之后都可用（会话记录持久化在磁盘上；只有内存中的注册表会丢失——而它的
  缺失恰恰是停止轮询的依据）。
- **`interactionState`** —— 交互工具的生命周期接缝：纯函数
  `deriveInteractionStates(turns, toolResponses, toolResults)` + `InteractionStatesContext`/
  `useInteractionState`（由 `ChatTranscript` 提供，也由计划页的 `PlanSessionLive` 提供，因此同一份
  已注册卡片可以在计划页作答——见 `panels/SPEC.md`；独立使用时为 `null`）。它覆盖每个渲染器注册为
  `interactive` 的工具调用；响应是不透明的（`unknown`）——卡片自己读自己的形状。一次实时阻塞调用通过
  其原生工具结果（其 `details`）了结；一个重启修复的待定标记稍后通过一条 `tool-response` 自定义消息
  （较旧会话记录中的 `ask-user-answers`）了结；用户直接打字越过的一次实时调用也以同样方式原生返回该
  标记，因此"已被取代"（待定标记 + 更晚的用户轮次）无论会话是实时还是恢复的，读起来都一致；
  停止/错误/length 结果都是终态，因为 Pi 从不执行来自被 length 截断的 assistant 响应的工具。因此
  "已作答 / 已被取代 / 已停止 / 等待中"只从三份会话记录投影派生一次，由卡片（作为 `interaction.state`）
  与计划速览消费，从而让其他一切地方都保持 props 驱动。同一条接缝还提供不透明的**按挂载
  `ChatView` 计的焦点作用域**：等待中的卡片在该作用域内只声明一次注意力（因此 Virtuoso 的重新挂载
  无法偷走焦点），而一次全新挂载会创建新作用域，并可再次聚焦仍待定的问题。"全新挂载"比关闭/重新打开
  chat 更宽：`CenterTabs` 只渲染活跃标签页的主体，因此**每次切回 chat 标签页**——从一个文件、一个
  diff、另一个 chat——都是一个新作用域，并为仍在等待的问题重新声明注意力。这才是预期的读法（你回到
  了需要你的那个 chat），而不只是副作用。它不携带任何 store 或传输层状态。
- **水合（Hydration）**（`hydrate.ts`）—— 纯函数
  `messagesToRuntime(TranscriptMessage[], lastSettlement?, { idScope? })` 转换器（事件 reducer 的读侧
  对应物；合成的失败轮次携带其 `failure` 种类与 provider 消息，在渲染时由 `errorTurnText` 措辞）：
  从持久化会话记录重建 `{ turns, toolResults, toolResponses, turnIdByMessageIndex }`（一个
  `HydratedRuntime`），使重连/第二客户端渲染与实时路径一致（相同的 `raw` 结果形状）。一次性消费者
  保留新铸造的 id；重复快照消费者提供其会话作用域，从而为持久轮次派生出稳定的 角色/时间戳/索引 id。
  提供时，实时摘要的 `lastSettlement` 是权威；否则只有末尾那段对话式 assistant 能合成 error/length
  轮次，并按与实时结算标记相同的 `insertRoundCloser` 规则放置（见下）。被压缩的历史 length 尝试后跟
  更晚消息时仍属历史，而不是过期的当前警告。两条路径共用一条重试呈现规则：pi 会持久化一次被取代的
  自动重试尝试（"keep in session for history"），而实时 reducer 在 `auto_retry_start` 时丢弃它，因此
  水合会隐藏紧接在另一条 assistant 消息之前的出错 assistant 消息——正是 `_prepareRetry` 产生的相邻
  形状（`isRetriedAttempt`）；终态失败——出错 assistant 后跟一条用户消息，或其后什么都没有——保持
  可见，其失败由末尾结算派生的错误轮次报告。它还返回 `turnIdByMessageIndex`（消息位置 → 铸出的轮次
  id）——即历史搜索"跳转到消息"深链（`chatLocationRequest`，见 `store/SPEC.md`）所据以解析的跳转
  锚点映射；对 `toolResult` 或非轮次的 `custom` 消息，条目为 `null`（一条 `subagent-completion`
  消息映射到它自己那张完成轮次的 id；无文本时它仍永远不是锚点匹配），对 `compactionSummary` 也为
  `null`（它自成一轮，但永远不是搜索命中——宿主的索引消费同一个槽位，因此两者保持对齐），而以
  `stopReason: "error"` 结束的消息映射到它自己那条 assistant 轮次的 id，绝不是合成错误轮次的 id。
  `custom` 消息：工具响应（`readToolResponseMessage`：`tool-response`，或已退役的
  `ask-user-answers`）索引进 `toolResponses`（永远不是轮次——交互工具的卡片就是它们的渲染）；
  `subagent-completion` **变成自己的 `subagentCompletion` 轮次**（结果行按会话记录位置定位，因此它
  也映射自己的消息索引）；`todo-review-fix` **变成自己的 `reviewFix` 轮次**（定位方式相同）；未知
  customType 被忽略。无 store/传输层/shiki。
- **跳转到消息（Jump-to-message）**（`chatLocationRequest`——在某个已映射的消息命中上按 Enter 时由
  `useHistorySearch.ts` 的 `openMessage` 设置；store 层的请求/清除契约见 `store/SPEC.md`，工作台
  shell 集成的打开/重开/水合那一半也见其中）—— `ChatTranscript` 是唯一消费者。一旦
  `rows.length > 0`，它通过 `runtime.turnIdByMessageIndex` 解析请求的 `messageIndex`（只存在于*已水合*
  的运行时——实时/已打开会话的运行时由事件 reducer 构建，绝不携带该映射），回退到扫描 `turns`，
  找自身文本包含 `anchorText` 前缀的最新一个——同一回退也覆盖已水合映射中某个其轮次已不再包含该锚点
  的条目（例如会话记录在其下方已发生变化）。解析出的轮次通过纯函数 **`rowIndexForTurn(rows,
  turnId)`**（`rows.ts`）映射为一行，调用时传入投影后的行——`user`/`system`/`error`/`retry` 用轮次
  自己的行，`assistant` 用其第一条 `:text:` 行（assistant 轮次会消解为 `markdown`/`tool`/`activity`
  行，自身永不成行）——随后一次可取消、无动画的物化从 Virtuoso 的实测尺寸快照推导目标（只对从未测量
  的行使用保守估算），再由控制器拥有居中像素修正。物化保持待定，直到该确切行在该对齐位置上跨稳定帧
  保持挂载并被测量；Virtuoso 估算修正期间的瞬时挂载不算成功。它从不启动 Virtuoso 内部会重试的
  `scrollToIndex`，因此读者接管可以取消每个在途写入。其生命周期以请求标识为键，而不是以流式行数组的
  高频变动的键，且其活的行索引解析器会跟随请求待定期间发生的投影变化。一个瞬时的 `flashRowId`
  （渲染为行包装层上的 `data-flash` + `bg-primary-subtle` 过渡，1600ms 后清除）只在行挂载之后吸引
  目光。解析到一行、读者显式取消，或耗尽有界物化等待（以 "couldn't locate the message" 提示）都会
  清除那个仍然当前的确切请求；较早的 effect 不得清除较新的跳转。取消是用户胜出的失败路径，且有意
  静默，而不是对着读者重启。effect 拆除会把自己的、带身份校验的清除推迟一个微任务：立即的 StrictMode
  或替换式挂载会先认领同一请求，而真正的卸载则终止它。`ChatTranscript` 是其唯一终态消费者，因此一个
  未解决的当前请求绝不能滞留。
- **按当前对齐目标打开（Open at the current alignment target）** —— `ChatMessageOrder` 选择物理的
  最新边缘：最旧优先为底部，最新优先为顶部。阅读带是唯一的挂载放置所有者。它在第一行挂载之后放置一次，
  依据那一刻的工作状态：空闲的最旧优先会话记录无动画跳到末行，并在测量帧期间保持住（运动种类
  `mount`）；空闲的最新优先会话记录停在原生零滚动原点；工作中的会话记录直接在 Settle 处重建，只留出
  其活跃响应所需的余地，并在同样这些测量帧期间保持 Settle，无动画地重新应用且绝不后退，因此行测量
  无法把实时边缘推出屏幕。挂载目标跳过跟随步幅上限，因此矮视口也能把边缘落在 Settle。Virtuoso 不会
  得到 `initialTopMostItemIndex`：它会把列表藏起来，直到滚动到该行时看不到尺寸变化，而流式的末行
  永远不允许这种情况，于是返回或刚打开的工作中 chat 会一直空白。原生输入待定期间、以及工作中的会话
  记录尚无流边缘时，放置保持待定。空闲保持期间开始的流会取消该保持并在 Settle 重新放置。读者接管、
  新轮次，或任何控制器揭示或锚点稳定化（跳转到消息、卡片自动揭示、折叠）都会取消待定的放置；随后由
  那次运动拥有位置。切换顺序会重新武装放置，因为在完全反转之间保持像素位置没有稳定含义。不存在中间
  的错误边缘绘制，也没有跨顺序动画。初始虚拟几何是**行感知**的：每个投影行都获得一个保守估算，由
  散文换行、块分隔与物理的围栏代码行派生，而不会切分一个规范 Markdown 块。这些估算镜像渲染出的行
  chrome（行与气泡内边距、段落间隙、prose 行高、高度下限）并随之变化：一次密度调整曾让它们过期，
  这使 Virtuoso 的测量修正从增长翻转为收缩，并把一次最新优先的回到最新位置搁成脱离状态
  （`chat-scroll.spec.ts`）。有界的像素与条目 overscan 让附近的离群项在粗粒度输入耗尽一个虚假范围之前
  替换估算。原生滚轮物理保持原样。`chat-history.spec.ts` 钉住默认最新边缘与高历史几何；
  `chat-order.spec.ts` 钉住两种投影。
- **对齐是显式的，不靠邻近推断** —— `useChatScroll` 拥有两个正交事实：实际工作（`agent_start` 到
  `agent_settled`）与对齐（`following` 或手动 `detached`）。工作中跟随时意味着配置的响应窗口；空闲时
  跟随时意味着物理最新边缘。工作中脱离时显示顺序感知的 **Follow response** 按钮；空闲时脱离显示
  **Latest**。自动聚焦、行测量、内容增长、程序化揭示，以及 Virtuoso 的便利边缘阈值，都永不创建或清除
  脱离状态，因此该按钮始终意味着是人接管了。
- **读者意图与精确边缘重新武装** —— 滚轮、触控板、触摸、滚动条与导航键输入，只有在可能造成或已造成真实
  视口移动时才脱离；对一个跟随中的读者来说，朝当前物理边缘向外推是空操作，而一个已经恰好在那个边缘上的
  脱离读者（保留读者的余地可能让已无可滚动的东西）会在同一次推动时重新武装。潜在的原生输入会暂停与之
  竞争的控制器运动，并阻断新的自动运动直到它了结，但不改变对齐（连续跟随几乎每帧都写入，因此落在读者
  手势与其滚动事件之间的一次写入会把它吞掉）。一次被中断的返回在等待滚轮或导航键默认动作期间仍逻辑上
  在移动；显式的指针按住则是静止的。若其后没有移动，对齐会在有界输入意图窗口之后恢复，而不是在下一帧
  恢复，因为内嵌 webview 可能在该帧之后才应用默认滚轮滚动。移入历史的移动只脱离一次；打断活跃对齐的
  原生移动也会脱离，即使方向朝向最新，除非该移动自身恰好到达那个边缘。显式的文本选择，以及用户发起的
  消息/历史、面包屑或工具页导航，也会脱离。指针来源在释放后存续足够长以覆盖原生滚动条轨道动画，键盘
  来源覆盖来自交互式会话记录控件的焦点引发滚动，两者都会在滚动结束或有界超时时过期，因此后来的几何
  无法继承它们。脚本经滚动容器 `scrollBy`/`scrollTo` 造成的移动（Virtuoso 的估算到测量修正、
  `scrollToIndex`）会从观察到的移动中扣除，且永不携带读者方向；只闭合这类移动的滚动结束也不结束任何
  意图：一次为上方的行挂载而给最旧优先读者的 1px 修正，曾被读成朝最新的触摸惯性，而它的滚动结束在到
  达边缘之前就结束了惯性。该 hook 在其生命周期内遮蔽滚动容器自身的 `scrollBy`/`scrollTo`，并在清理时
  恢复；这依赖 Virtuoso 只通过这两个方法移动其滚动容器（已针对 react-virtuoso 4.18.7 验证，升级时
  需重新验证），且只归因这些调用同步施加的位移。一次返回手势只有在共享的 1px 几何容差内到达物理最新
  边缘时才重新武装一次；方向随顺序反转，而触摸/触控板意图能穿过惯性存续。没有 50px 近边缘阈值，也
  没有任何单独的几何变化可以重新武装。展开或折叠 Activity、Thinking、工具或消息展开区属于几何变化而非
  导航：它保持对齐，跟随时重新指向当前的最新/响应目的地，并让脱离读者的可见锚点保持固定。自己的 Send
  故意重新附着，并把其用户行放在会话记录高度的 10%（钳制到 48–80px）；一次排队/后台的续作在其开始时
  保留脱离读者的位置。
- **流式响应移动只存在于工作期间，且以窗口步幅移动** —— 跟随时，当活跃响应填充其 prompt 下方的窗口
  时，视图静止不动。每次响应边缘越过 Trigger（默认 100%），唯一的运动所有者就做一次约 220 ms 的缓动
  步进，把边缘放到 Settle（默认 75%），随后再次静止，直到边缘再次越过 Trigger。阅读静止的文本正是
  要点：连续跟随（把边缘保持在 Settle 并逐帧平滑）曾被尝试并否决，因为正在读的那行从未停止爬行。
  步进目的地会在这一步的每一帧从实时几何重算，因此步进期间的增长被包含在内，步进也绝不越界。步进永不
  后退。合成余地是**派生的，绝不累积**：步进期间它恰好是 Settle 目的地超出自然滚动范围的那部分；步进
  之间它是保留读者的剩余量，在每次内容变化时于绘制前重算，因此响应增长会一对一消耗它，而可见内容与
  滚动范围都不移动。旧的步进实现晚一帧消耗余地，这让滚动范围反复抖动、每一步都摇晃；让人觉得跳跃的是
  这个缺陷，而不是步进模型。因此最旧优先至多需要较低的 `100% - Settle` 带；最新优先在可用时使用更旧的
  投影内容，并在最旧组之后保留任何合成剩余量。合成余地绝不切分一个被反转的请求/回答组。
  **Follow response** 与一次精确边缘返回会做一次到 Settle 的步进并重新武装该循环。读者接管的移动永不
  搬动读者：余地只收缩到读者当前位置允许的程度（文档可能恰好结束在其视口底部），而之后的内容增长或
  向上阅读会消耗其余部分。释放余地的揭示（工具注意力、跳转到消息、面包屑）会在其目标仍是最新行时暂停
  步进，因为它即将自行放置视口；其后的第一个新最新行（例如已解决问题之后的那个回答）结束该暂停。把
  暂停延长到整个响应剩余部分，会在每张问题卡片之后冻结跟随，然后在结算时飞出 2–2.6k px。在原生输入
  仍待定时落地的结算会推迟其返回直到该输入了结，若该输入已脱离则放弃返回。
- **高个子到达显示其开头** —— 在最旧优先下，控制器记住读者实际看到过的最后一个响应边缘。当跟随目的地会
  把该边缘带到轮次内缩之上——比阅读空间（Settle 线减去轮次内缩）更高的内容到达其下方，通常是立即渲染
  的图或卡片，且可能分几个快速的布局步骤——目的地会被封顶，使该点落在轮次内缩处，与自己 prompt 落位
  的地方相同。封顶生效后 300 ms 内的增长属于这次到达；其后每次增长都按自身高度的两倍释放封顶，直到
  达到 Settle，而视图仍只以窗口步幅移动：一旦封顶后的目的地至少领先一个窗口（`Trigger − Settle`）就
  发生一次步进，因此该块以两倍阅读速度推进，而不是飞掠而过（一张 mermaid 卡片实测在不到一秒内滑动
  1.5k px，而单独评判每个布局步骤会漏掉分两步落地的卡片）。第二次高个子到达绝不把活跃的封顶推得更远。
  封顶在跟随目的地所在的任何地方都被评估，因此它也会约束一个已在飞行中的步进。位置相对于最顶端可见行
  保存，而不是文档坐标，且每次评估都会把它们相对当前那一行重新表示，因此视口上方的高度变化
  （Virtuoso 替换估算、代码块高亮）既不触发也不误置封顶，行可以卸载，而一个增长到已经在跟随它的行
  上方的块（WebKit 在下一轮次的行已存在之后才渲染图）仍被计入。改为锚定到最后一行恰恰漏掉这种情况。
  宽度重排、读者自己的展开切换，以及一次全新挂载，会重取看过的边缘，而不是被算作到达；自动展开
  （卡片完成时打开）仍可算作到达。Markdown 的 mermaid 围栏就地替换自身源码，因此其渲染后的顶部比旧
  边缘高出源码高度，而封顶从那一点起显示该图。最新优先会前插其最新行，因此那里的边缘增长不是追加的
  内容，封顶不适用。**Follow response**、一次精确边缘返回、一个新轮次、读者接管，以及一次释放余地的
  揭示都会清除它。结算像任何其他跟随结算那样结束封顶：下方仍未看到的内容获得一次向前移动到末尾。
  改为在该处让读者脱离，会在没有任何人接管的情况下露出 **Latest**，并把增量成大批到达的纯文本回答遗落。
- **跟随时保持屏幕上的内容静止** —— 当跟随中的读者观看最旧优先的流时，最顶端可见行就是视图的锚点：
  只有控制器自己的写入与读者输入可以移动它。在每次改变条目列表的布局之后（绘制前），以及在任何不是
  该 hook 造成的滚动上，被移动的锚点都会在同一帧恢复，当滚动范围收缩时加入合成余地。恰好落在收缩后
  范围末端的滚动属于钳制，交给绘制前路径处理，因为后者运行在 Virtuoso 自己的尺寸补偿之后；从滚动事件
  恢复它会让 Virtuoso 把该钳制读成向上阅读，并把恢复补偿掉（Chromium 中一次 844 px 的跳跃）。有两个
  实测原因促成这一点，且两者在没有虚拟化时同样出现：一张已回答的问题卡片折叠 400–800 px（浏览器钳制
  范围，随后 Virtuoso 的尺寸补偿滚动其余部分），以及 WebKit 在一个工具行出现在重渲染的文本行旁时重置
  `scrollTop`（627 → 0），其间没有脚本写入、焦点或滚动锚定参与。守护在读者脱离时、原生输入待定期间、
  揭示/折叠锚点/结算运动期间，以及一次轮次锚点把放置交给 Virtuoso 之后 1.3 s 内（它的 `scrollToIndex`
  在尺寸持续变化时重试；更晚的重试指向同一轮次位置）保持空闲，因此它绝不与某人请求的放置相互冲突。
  合成余地始终挂载（不用时为零高度），以便守护能同步扩展它；被留下的余地保留读者，之后的增长会消耗它。
  会自我宣告的变更完全避开钳制：折叠接缝（展开区，以及问题卡片自己的提交——它把卡片换成一个小
  "Answer sent" 状态）在跟随读者观看流时，在 DOM 变化之前为变更行的整体高度预留余地，然后修剪到读者
  位置所需。该修剪会把短于视口的内容计入，因为那时真实自然滚动范围是负的；把它钳制为零会预留不足的
  缺口，并让一个短会话记录在一次回答之后向下滑 280 px。
- **结算绝不搬动已接管的读者，也绝不把内容往后移** —— 每个 `agent_settled`（绝不是 `agent_end`）结束
  响应移动。在最旧优先下，内容已结束在视口内的跟随读者保持原位：剩余的合成余地变成保留读者的余地，
  随读者向上滚动或下一轮次开始而消失，绝不靠滑动屏幕上的内容来消失。在结算时移除它，会把每个已完成
  回答向下滑最多 `100% − Settle` 带（实测约 100 px），在一次被吸收的卡片折叠之后则会滑数百 px。只有
  当下方仍有未看到的内容时，跟随读者才做一次平滑的向前移动到物理最新边缘；最新优先保持它自己那次
  平滑返回其顶部最新边缘。脱离读者的可见内容保持原位，只有可用性从 **Follow response** 变为
  **Latest**。在结算时把脱离读者拉回，是实测最大的拉扯（数万 px），且与"读者胜出"相矛盾。store 在
  `isStreaming` 之外还暴露一个按会话单调递增的结算 tick，因此被合并进一次 React 渲染的开始与结算不会
  把乐观的轮次内缩或跑道搁死。延迟的虚拟测量会重新指向同一次有界返回，而不是制造硬钉住的循环。若读者
  输入与结算相交，任一条空闲重附着路径都会把部分的"余地去零"腿继续推进，而不是泄漏隐藏的跑道。被拒绝
  的立即 prompt 同样取消其本地武装的轮次状态。
- **稳定的工作状态几何** —— 一个固定尺寸的槽位始终占据逻辑上的最新会话记录边缘：最旧优先时在行之后，
  最新优先时在行之前。工作活跃时它始终包含一个 polite 活阶段——**Working…**、**Thinking…**、
  **Running `<tool>`…**、**Writing…** 或 **Compacting context…**；重试/provider 间隙回退到 Working
  而不是卸载它。空闲时保留一个不可访问、视觉上为空但几何相同的槽位。可见阶段在窄面板中单行并裁剪在
  槽位内，而其完整的活区文本仍可访问。因此开始、变更或结束一个阶段既不移动会话记录对齐，也不移动
  输入框。
- **工具注意力保持对齐来源** —— 一个等待中的 `ask_user_question` 可以清除临时余地并执行其既有的有界
  起始揭示/聚焦，但该自动路径会原样保留跟随或脱离状态，且无法暴露该按钮。此前已被脱离的读者因那次
  先前动作而保留该可用性；其后由用户驱动的工具页导航可以脱离。所有注意力、历史、面包屑与行揭示都
  路由到同一个控制器。虚拟 DOM 之外的历史行会从 Virtuoso 的实测尺寸快照获得有界的、hook 拥有的物化；
  一旦挂载，其居中修正使用控制器，因此没有独立重试能超越读者取消或与结算竞争。尺寸感知的 `nearest`
  让高目标的有用前缘保持可见。
- **唯一的可取消、可重指向运动所有者** —— 渲染器与投影绝不自己滚动。新轮次放置、窗口步进、上下文按钮
  返回、结算与显式揭示共享一个不重叠的通道，其目的地可随 Virtuoso 测量、状态几何或跑道变化落地而重新
  指向。修正延续当前运动，而不是发起重叠缓动或交替硬写入。读者的首次真实移动会同步取消它，原生物理
  胜出。最新优先的头部增量保留脱离的历史锚点；视口尺寸变化会重新评估实时百分比，而不搬动一个位于
  Trigger 之下的响应。初始/顺序放置是直接的，而减弱动效让每个程序化目的地立即生效，同时保持完全相同的
  状态与最终几何。
- **没有隐藏的运动所有者** —— 除控制器与读者之外，任何东西都不得移动视口。会话记录滚动容器退出浏览器
  滚动锚定（`overflow-anchor: none`）：Virtuoso 排除其条目，但 Chromium 否则会锚定到头部/页脚，并在
  响应于其下方增长时反向滚动读者。最旧优先的顶部内缩是常量，而不是随合成余地切换；切换会在脱离时把
  读者内容移动 48–80px 的内缩量，而那里没有锚定来补偿（WebKit）。合成余地与条目列表尺寸变化在同一帧
  的绘制前对账，因此滚动范围与滚动条绝不在帧间抖动。Virtuoso 的 resize 处理运行时不带其动画帧延迟，
  因此视口上方的测量修正落在绘制之前。wheel 监听器保持非被动：被动的监听器会让 Chromium 在 wheel
  事件到达 hook 之前就应用滚动，于是移动在没有读者意图的情况下到达，脱离与精确边缘重新武装都不会触发。
  滚动保持响应，因为流式增量只重渲染内容变化了的行，让主线程保持空闲。
- **输入框与外围 chrome（Composer & chrome）** —— `Composer`（prompt 字段 + send/steer/followUp/abort、
  `@` 提及（`mentionLookup` 把输入的查询拆成 `fs.readDir` 列出的文件夹与名称前缀，把输入的反斜杠
  `\` 读作分隔符）、`/` 命令 + 模板**槽位会话**（Tab 逐个跳转的占位符——见下文 Template slots 条目）、
  图片粘贴/拖放——经 **`imageAttachment.ts`** 路由：`fileToAttachedImage` 在浏览器内解码，并对任何
  超过 **1568px 长边**的图像降采样（`fitWithin`；Claude 标准档的长边——历史中一张超大图会让此后每一
  轮都返回 400，一旦 provider 的 >20 张图 2000px 上限生效，而 pi 自己的缩放器在服务端是有意关闭的）。
  只有当图像在像素范围内**且**类型被 provider 接受（png/jpeg/gif/webp）**且**低于 provider 的
  **4.5MB base64 编码上限**（`IMAGE_MAX_BASE64_BYTES`，经 `contracts` 共享——pi 在 Anthropic 5MB API
  限制之下给自己留的余量；线上契约传输 base64，因此上限按 `data.length` 测量，`base64EncodedLength`
  用于在编码前估算原始 File）时，图像才逐字节原样通过；其他任何情况都经 canvas 重编码，在编码超出
  上限期间走一条 **JPEG 质量阶梯**（一张在范围内但数 MB 的 GIF 或一张小 BMP，会像超大图像那样让请求
  返回 400）。无法解码的文件**只有在其媒体类型被 provider 接受时**才回退为原始字节
  （`ACCEPTED_IMAGE_TYPES`，经 `contracts` 共享）；无法解码 + 不受支持（HEIC……）会被**拒绝**
  （`fileToAttachedImage` → `null`）——原始透传会让此后每一轮都返回 400——并在附件条以可关闭的错误
  徽标块（`composer-image-error` testid）呈现，发送时清除。一条消息的批次还受请求级
  **`REQUEST_IMAGE_BASE64_BUDGET`** 限制（24MB base64，Anthropic 每请求 32MB 上限之下的余量）：会把
  批次推过它的文件以同一错误徽标块界面被拒绝。服务端的 `imageGuard` 扩展是历史部分的第二道防线。
  文件仍在解码时，会渲染一个占位徽标块（`composer-image-pending` testid），且发送被扣住（`canSubmit`
  是唯一的判断——`submitText` 拒绝，发送按钮禁用）——否则解码中途的发送会不带该图发出，并把它遗落到
  下一条消息。被扣住的发送保留其文本：输入框自己的手势会把草稿留在原处，而 `insertAndSubmit`
  （浮层的 ⌘/Ctrl+Enter，其文本尚未进入草稿）把它停放在那里而不是丢弃。待定徽标块显示
  `filename · W×H`（所拾文件的名称；只有在没有名称存活的水合轮次回退中才出现 mime 文本）
  （`composer-image` testid + `data-width`/`data-height`/`data-mime`——即
  `e2e/composer-images.spec.ts` 的钩子；两种徽标块皮肤共用 `FileChip.tsx`）。**徽标块是有界的，而
  标签是唯一会让步的部分**：徽标块是 `max-w-full` 并截断其 `label`，而图标、`meta` 后缀与尾部动作
  不收缩——文件名由用户控制，一个无界徽标块会把它自己的 Remove 按钮挤出手机视口（并被会话记录滚动容器
  的 `overflow-x-hidden` 裁掉）。因此任何必须在任意宽度下保持可读的内容都放进 `meta`，而不是
  `label`：`· W×H` 尺寸，以及一次附件错误的原因（其文件名被截断——原因才是用户能据以行动的东西，而
  手机没有 tooltip 可回退）——以及其命令式句柄上的 `openHistory` → `onHistoryOpen`），外加共享
  `prompt` 模块的**斜杠补全基础组件**（过滤/菜单/光标 + Up/Down、Enter/Tab、Escape）；
  `HistoryOverlay`（`Composer` 打开的历史召回/搜索浮层——纯呈现，完全由 `useHistorySearch.ts` 的状态
  + 回调驱动，外加对已映射命中的 **Save as template** 与一键 **Trash chat** 动作（`ChatView` 拥有
  `session.delete` + 幂等的 store 删除折叠；成功则关闭浮层，失败则 toast；`session.deleted` 也会在
  每个已连接客户端中驱动该折叠），以及一个**放大阶段预览面板** + **作用域选择器**——见下一条），
  **`ModelEffortPicker`**（那枚**模型 · 力度药丸**，`NewWorkspaceDialog` 在预会话模式下也会挂载它；
  `ModelsSettings` 仍挂载较旧的 `ModelSelector` + `ThinkingSelector` 组合——它们是有名字的幸存者，直到
  迁移为止）。决策：一个 chat 的模型与力度是**一个事实的两个部分**，由一个无边框触发器呈现
  （`[vendor glyph] name ▂▄▆ level [connection glyph] ▾`——力度条在该模型的推理等级中点亮该级别的
  位次，`litBars`，并使用该级别的**色调**——关闭/极低/低为冷蓝，中/高为强调色，昂贵档位为警告琥珀，
  因此成本在读这个词之前就能读出来），并在同一个 popover 中选定，其中**力度控件位于模型列表下方并
  跟随所选模型**——因此可提供的等级始终是*那个模型的* `thinkingLevels`，而"力度被禁用"的状态不可能
  存在。流程是**点模型 → 滑动力度**：选择一个模型立即应用它并保持 popover 打开（悬停绝不改变任何东西
  ——早先的悬停预览让力度行随指针移动而跳动）；力度控件是一个**滑块**：一条厚轨道里藏着从冷到暖的
  渐变（`feedback-info` → `primary` → `feedback-warning`），手柄移动时揭开它，级别词与其 glyph 骑在
  手柄上，其下每个等级一个可点击标签（用户保存的 `defaultEffort` 用点标出——web 永不去猜宿主的回退
  等级），一行提示，以及在昂贵档位上的警告说明（`COSTLY_LEVELS`：xhigh、max）。一个原生
  `<input type="range">` 驱动它——拖动、点击吸附、触摸与 ←/→ 都免费获得——且每次变更立即应用而
  popover 保持打开；当等级不同于已保存的默认值时，页脚提供 "reset to ‹default›"。手柄位置是唯一的
  内联样式（一个 `--effort` 自定义属性，即规范化几何例外），颜色留在 token 中，`motion-reduce` 会
  压平弹簧动效。Escape、外部点击，或点击已经是当前模型的那一项都会关闭。`onSelect({model, level?})`
  是模型回调（只有当一次键入的 `opus high` 同时选定两者时才带 level），`onSelectLevel` 是仅等级的回调，
  而调用方——绝不是选择器——与宿主对话（`ChatView` 把 `session.setModel` → `session.setThinkingLevel`
  串在一个选择计数器之后，每次模型*或*力度选择都推进该计数器，因此串联的等级只在那一选择仍是最新时
  才被发送——失败时回滚——于是 A·high 然后 B·low 绝不会把 A 的等级落到 B 上，而 B·high 随后滑到
  medium 也绝不会让 B 的响应重新发送 high）。当一次目录刷新让某会话停在其模型不再提供的等级上时，
  `ChatView` 询问宿主的 `model.clampThinking` 并应用答案——即 `NewWorkspaceDialog` 已为其组合运行的
  那个对账，也是 pi 在请求时应用的同一个钳制——因此药丸与滑块显示 pi 实际将使用的等级，而不是轨道上
  没有的一档。
  行是**两行**：名称，然后是 `provider · [kind glyph] what it draws on · price · context`。列表读作
  **Default 行**（仅预会话调用方，`defaultOption`：宿主会选的那一个，当调用方跟随它时打勾）→
  **Pinned** → **Recent**（宿主列表减去已钉住的模型）→ provider 分组，在存在短名单时折在一个
  "All models" 行之后、并在搜索时展开。每个 provider 分组都可折叠：其头部本身是一行 cmdk
  （`model-group-toggle`、`aria-expanded`、Enter 切换），携带匹配计数，而选择器打开时除当前模型所在的
  分组外全部折叠（当该模型被钉住时则全不展开）；一次搜索会打开每个匹配的分组，各分组对该次搜索仍可
  折叠。头部读起来像段头（强调字重的名称、vendor glyph），第一个之后的分组由一条分隔线隔开，其行缩进
  在头部之下并去掉自己的 vendor glyph 与 provider 词（头部已携带两者）——扁平无内边距的列表会让头部
  和模型难以区分。已钉住的模型绝不在分组内重复。头部、计数与行由选择器自己的 cmdk `filter` 匹配
  （`chat/pickerGroups.ts`，cmdk 默认打分器的移植），因此一个头部在其任一模型匹配时匹配，其计数等于
  它揭示的行数。无查询且无模型时，列表读作 "No models to show. Connect a provider, or show hidden
  models in Settings → Models → Visible models."；"No models found." 只在有查询时出现。隐藏模型绝不
  到达任一选择器：调用方传入 `store` 的 `selectPickerModels`，它会保留当前模型。Settings 的
  `ModelSelector` 共享这套分组、过滤、Pinned 段、空状态与 "Updating…" 刷新，但没有 Recent，也没有
  头部的连接细节。查询末尾的一个词若点名了高亮模型支持的等级（`opus high`），会预先选中它——即 pi 的
  `model:level` 惯用写法用空格输入——而输入框中的 `/model [query]` 会以预填方式打开选择器，而不是
  发送文本。
  第二行**说明用户实际付的是什么**（`kindLabel` / `costLabel`）：连接 glyph——**key** = API key、
  **∞** = 订阅、**{ }** = 环境 key、**JCP** 标记 = JetBrains AI（Central 代理）——然后是套餐/可变/
  key 词，且只在 provider 按 token 计费时（`auth.kind` 为 api-key / env）才显示 `$in / $out per M`；
  分组头部补上该 provider 的连接细节。**vendor glyph**（`ProviderGlyph`）是来自
  `generated/providerGlyphs.ts` 的单色 `currentColor` 标记，而 `scripts/generate-provider-glyphs.ts`
  在构建时从仅开发用的 `@lobehub/icons-static-svg` 集合中抽取它（`provider-glyphs:check` 像颜色流水线
  那样守护漂移）；pi-provider-id → 标记的映射只存在于 `scripts/providerGlyphs.ts` 一处，通过多个 pi
  provider 到达的厂商共用一个标记，而未映射的 provider 渲染为首字母图形。标记绝不被品牌色着色——颜色
  系统拥有颜色。Pinned/recents/default 通过 **`useModelPreferences(models)`** 进入（两个调用方共享的
  唯一 store+传输层接缝：列表重新指向实时目录并丢弃消失的模型，`toggleFavorite` 作为整条列表的
  `settings.update`，`setDefault` 同时写模型 + 力度，全部以 `MODEL_PICKER_PROTOCOL_VERSION` 为门禁，
  因此较旧的宿主既不显示钉住切换也不显示这些段）。页脚有 **Set as default**（一旦该组合匹配就读作
  "Default for new chats"）与 **Refresh** 行（`force: true`，在被等待的刷新运行期间旋转并读作
  "Updating…"；打开时会发起一次不经强制的读取，由宿主快照供给）。web 仍**不枚举等级词表**：行就是
  `WireModel.thinkingLevels` 的逐字值，宿主负责钳制，而 `LEVEL_HINT` 是一份部分记录，某个等级完全可以
  没有对应的定性文案。行跟随**实时目录**——`ChatView` 在向下传递之前通过 `store` 的
  `selectCatalogModel` 解析会话的模型，因此一次改变了某模型支持范围的 `model.refresh` 会连带改变所
  提供的等级）、`SessionStatsBar`、`ChatHeader`（固定的单行**面板头部行**——
  `h-panel-header-row`（`--panel-header-row-height`，当前 28px），与工作台 Group Headers 和 Changes
  工具栏共享的结构几何，而不是在这里钉住的一个值；它从不滚动，而受限宽度会裁剪/截断 TODO + 状态/
  用量文本，同时保留尾部的 Skills 动作。其 **Skills** 按钮是展示型的 **`SkillsButton`** 基础组件
  ——一枚 `BookOpen` 药丸，当磁盘上的某个 skill 目录发生变化时带徽标——也与 `NewWorkspaceDialog`
  共享，使两个触发器不会漂移），以及 **`SkillsDialog`**（**Skills 管理器**：按来源分组的一份目录，
  带**吸顶段头**——第一方的 **ThinkRail** 与 **Pi** 组在最前（在 All-plugins 主开关之上，后者只管辖
  插件组），然后是 Personal / **每个已安装 Claude 插件一组** / 仓库的 Project skills 最后——每组都带
  其准入判定、项目信任（与 Welcome 提示相同的 `projectTrust` 派生与文案——`useProjectTrustSummary`
  加上本目录中未受信任的别名，因此仅有工作区别名的情形仍会显示——所以在 chat 中这里就是授予 pi 级信任
  的地方，其 `project.setTrust` 恰在点名了项目的 Pi 资源时携带 `resources: true`）、重新确认新项、
  每组一个轨道/滑块**开关** + 一个 **All-plugins** 主开关，以及逐 skill 开关。每个 skill 行读作一个
  名称（卡片标题）压在其描述之上、描述截断为两行（悬停显示全文），因此扫描一串相似的 "Use when…"
  行仍能区分各 skill。开关位置加语义色在不显示 On/Off 文本的情况下承载状态；只有开关目标是可变的，
  不可用的控件保留既有的信任/父级解释与确认行为。它通过可选的 `workspace` prop 以**两种模式**运行：
  chat（`skills.state`、按工作区的 skill 覆盖，+ 一个 **Reload**，通过 `session.reloadResources` 把
  更改应用到本 chat 的会话，流式期间禁用）或 project（`project.skills`、按项目基线的切换、无会话）
  ——后者被 `panels` 在预会话阶段复用。全部 props 驱动；行为细节在各组件的 jsdoc 中。
- **实时会话遥测跟随 Pi 的定稿边界。** `SessionStatsBar` 只渲染宿主权威的 `session.getStats` 快照；
  web 绝不从消息内容推导计费总额或上下文用量。总额包含委派 subagent 的花费，宿主会在运行中的
  subagent 每次响应之后以及其运行结束时，把它作为 pi `usage` 条目记入父级，因此工具栏在 subagent
  工作期间就会变化，而不是在其结束时才跳一下。挂载的 chat 在挂载/重连时刷新，在每条定稿消息、压缩、
  结算，以及每追加一条携带花费或上下文的条目（来自 subagent 运行的 `usage` 或缓存预热、`compaction`、
  `branch_summary`、`context_edit`）之后刷新，也在 Pi 接受一次模型变更之后刷新。一次重连读取可能与
  重启中的宿主竞速，此时会话记录同步尚未重新接上已持久化的会话，因此该同步边缘会再次触发读取。批进
  同一次 store 提交的连续 Pi 事件会合并成一次读取，而被更新的会话修订、宿主世代或卸载取代的响应会被
  忽略。文本增量不触发读取，因为 Pi 自身在消息边界之前无法定稿新的用量；瞬时读取失败保留最后的好快照，
  而不是用猜测的状态替换它。
- **自适应输入框几何（Adaptive composer geometry）**（`Composer`）——一份能放进一个视觉行的空闲草稿
  渲染为共享的两层外壳：一行全宽、单视觉行的消息行，位于稳定的动作页脚之上。模型 · 力度药丸是页脚左侧
  唯一的触发器；History 与 Send 明确留在右侧。一次换行、显式换行符，或使草稿超出一个视觉行的宽度变化，
  会增高消息行而不移动页脚；再次放得进一行时只收缩消息行。这是一个持久的 textarea，绝不是有条件的孪生
  兄弟——该过渡不会丢失焦点、光标/选区、召回、草稿或模板槽位会话。流式时刻意使用展开的消息行，即使
  草稿为空，因为 Stop + 发送选项会加入页脚。`ChatView` 传入由服务端同步的 `ComposerGrowthLimit` prop：
  `compact` 上限为 6 个视觉行，`roomy` 为 10，默认的 `half-chat` 把**编辑器外壳**（textarea + 页脚）
  限制在挂载 chat 面板的 50%，绝不是浏览器视口；溢出随后在 textarea 内部滚动。附件徽标块、补全菜单、
  槽位提示与 QueueStrip 保持各自原有的独立 chrome。槽位高亮背板必须跟随每一次动态 textarea 盒变化，
  并遵守下文 Template slots 中确切的盒模型与滚动同步不变量。
- **输入框尾部控件（Composer trailing controls）**（`Composer`）——页脚右侧的一组是**一个实心对象 +
  幽灵按钮**，没有边框：History 是 `IconButton ghost m`（其 `label` 提供 tooltip），Stop 是
  `Button ghost m`（`■ Stop`，弱化文本在悬停底色上抬起），而发送是一枚 `rounded-full` 强调色药丸
  （`chat-send-pill`、`data-armed`），其主段（`chat-send`）携带动词与一个 `↩` 键帽，其 chevron 段
  （`send-menu`，仅流式时）是第二个命中区并带自己的悬停——分隔它们的是间距而不是分隔线。早先的那一组
  是四个等大带边框的 32px 方块，让一个实用功能（历史）与主操作同等分量，读起来像表单时代的 chrome。
  **惰性发送**：无内容可发时药丸停在 `control-bg-selected` + `control-disabled-text` 上，而不是 60% 的
  主色对——一块深色强调色会把目光吸向一个无法动作的控件，而切换到强调色填充才是"就绪"信号（该例外
  记录在 `styles/COLOR.md` 中）。**紧凑模式**：低于 chat 列的 `@md` 容器宽度（手机、窄分屏）时标签与
  键帽隐藏，发送药丸塌成图标，从而让模型药丸保住自己的空间。标签使用 `ui.action`。
- **排队消息：待定条（Queued messages: the pending strip）**（`QueueStrip.tsx`，props 驱动：`queue`
  + `onEdit`/`onRemove`）——pi 交互模式待定消息区的 web 镜像。**流式发送绝不渲染乐观的会话记录气泡**
  （见 store 规格的回显契约）：`ChatView.onSubmit` 对 `steer`/`followUp` 跳过 `appendUserMessage`，
  而排队的文本渲染为会话记录与输入框之间的一行行弱化文本——每条消息一行截断的
  `Steering:`/`Follow-up:`（`queue-strip` / `queue-item` testid、`data-kind` + `data-index`；完整文本
  + 投递含义在行的 `title` 中），来源是运行时的 `queue`。**每行携带自己的编辑与移除动作**
  （`queue-item-edit` / `queue-item-remove`）——两者都调用 `session.removeQueued { kind, index }`
  （行按位置寻址，与线上契约操作一致）；编辑还会把被移除消息的文本与图片恢复到草稿并重新聚焦。逐行
  动作之所以存在，是因为原先的全有或全无出队（点击条 → `clearQueue` → 每条消息合并进一个草稿块）在
  实际使用中被证明难以发现且有损。**中止会原子地恢复完整队列**（`onAbort` →
  `session.abort { restoreQueue: true }`）：宿主把两个 Pi lane 都排空并把中止作为一个操作发出信号，
  等待空闲，然后返回每条排队消息的文本 + 图片内容；web 前插这些文本并重新挂上每张图片。因此 Stop 既
  不能让一个排队的续作继续运行，也不能静默丢弃一个附件。一次**被拒绝的**流式发送同样把其文本恢复到
  草稿，并伴随 `appendErrorTurn`。普通的 `queue_update` 投影仍只携带可显示文本加一个保守的
  `hasImages` 聚合——不含图片字节——因此排队中的图片在条中不显示徽标块；规范会话记录轮次稍后用水合
  的回退标签渲染其图片块。E2e：`queue.live.spec.ts`（@agent）。
- **流式发送模式：拆分发送 + 打断（Streaming send modes: split send + interrupt）**（`Composer`）——
  steer/queue 语义是 pi 的循环设计（steer = 在下一个轮次边界注入，即当前 assistant 消息及其工具调用
  之后；queue = 在 agent 结算后运行；只有 abort 会中止在途响应），而仅靠键名提示无法读清。因此流式
  期间输入框会自我说明：主药丸读作 **Steer ↩** 而不是 Send，占位符说明 steer *何时*落位
  （"Steer the agent at its next step…"），而一个**发送选项菜单**（`send-menu`——那枚药丸的 chevron
  段；行 `send-mode-steer` / `send-mode-queue` / `send-mode-interrupt`）用一行含义 + 快捷键点名每种
  模式。chevron 在草稿为空时仍保持可用，使快捷键仍然可发现；只有行会禁用。菜单行是**动作**（以该模式
  发送当前草稿），绝不是粘性模式开关——持久模式会让下一次普通 Enter 静默服从隐藏状态。`Composer` 在
  槽位、菜单、召回或发送处理之前，把每个 keydown 让给活跃的 IME。它使用 `KeyboardEvent.isComposing`
  加上遗留的 `keyCode` 229 哨兵，因为 `compositionend` 可能先于最后一个 keydown，使 `isComposing` 在
  IME 仍拥有该事件时为 false。该守护不取消事件，因此确认候选的 Enter 会提交；只有其后的普通 Enter 才
  触发发送语义。**打断**（`SubmitBehavior: "interrupt"`，流式期间 Cmd/Ctrl+Shift+Enter；空闲时即普通
  发送；单独 Shift+Enter 仍是换行）是 pi 缺少的"现在就把我的消息收下"手势：`ChatView` 等待
  `session.abort`（该 ack 意味着空闲）然后执行一次普通的空闲发送——部分回复会留在会话记录中并标记为
  已中止，而仍在排队的消息保留其 lane（它们会在打断启动的那次运行中投递）。拒绝时像其他流式发送那样
  恢复草稿。
- **历史浮层：放大预览面板 + 作用域选择器**（`HistoryOverlay.tsx`）—— `Tab` 把紧凑的单列浮层长成
  **双栏** `zoomed` 布局：现有的 Prompts/Messages 段列表留在左侧（约 55% 宽度，
  `data-testid="history-results"`——键盘导航、`scrollIntoView`、计数与另存为模板动作都保持不变），
  而扁列表中**键盘选中**项的预览渲染在右侧（约 45%，`data-testid="history-preview"`，通过与
  `Enter`/Cmd/Ctrl+S 已在使用的同一个 `resolveHistorySelection` 解析——预览与键盘动作在"选中项"上
  绝不可能不一致）。`compact` 阶段保持原样：在按 `Tab` 之前 DOM 中根本不存在预览面板（不只是隐藏），
  因此 `history-preview` 的存在本身兼作放大/紧凑的信号。**预览主体：** 命中的完整 `text`——绝不是行的
  截断首行（`PromptRow`）或片段（`MessageRow`）——正是让预览有价值的东西：一个长 prompt 在列表中被
  截掉的尾部，在这里读全。`whitespace-pre-wrap break-words`、可滚动（`overflow-y-auto`），查询词通过
  **逐字复用**的 `Highlight` 高亮（与各行使用的同一个辅助函数），因此高亮绝不会在行与其自己的预览之间
  漂移。**预览页脚**（弱化、小字）：对 prompt 命中，是 chat 标题（若已设置）/ 只要 `workspaceId` 存在
  就有的工作区徽标块（与 `PromptRow` 的徽标块不同，绝不受作用域门控——单个细节面板有余地，而密集的
  列表行没有）/ 相对时间，以 `·` 连接；对消息命中，是 `sessionTitle · role · relative time`。无选择
  （空结果集）渲染一个空面板——绝不崩溃。**窄宽度**（低于 `md` 断点）：预览塌到列表**下方**而不是旁边
  （源码顺序中列表在前，因此列式 flex 堆叠已把它放在那里），每个面板在自己的高度预算内独立滚动。
  **作用域徽标**（`data-testid="history-scope"`，未变的 `<scope> ⌃R` 标签 + `data-scope`）现在也是一个
  `@thinkrail/ui/dropdown-menu` 触发器：其内容按循环顺序列出全部四个作用域
  （`data-testid="history-scope-option"` + `data-scope`，标签比徽标本身更完整——"This chat" /
  "Workspace" / "Project" / "Everywhere"——当前项打勾）。选中一个会调用 `useHistorySearch.ts` 新增的
  `setScope(kind)`，它像 `cycleScope` 那样重置结果选择——即 `Ctrl+R` 键盘路径（见下文 chord 归属条目），
  因为两者都只是设置同一个底层作用域状态。Radix 关闭时的默认行为是把焦点还给触发器；`onCloseAutoFocus`
  被覆盖（`preventDefault` + 聚焦查询输入框），因此鼠标选择会把焦点交回查询输入框——打字立即恢复，无需
  额外点击。该菜单是一个**受控**的 Radix 菜单（`scopeMenuOpen`），原因只有一个：浮层的窗口级 `Escape`
  与外部按压监听器在它打开期间让位，因此 Escape 或一次按压会关闭最内层。该菜单绝不与浮层自己的
  `ArrowUp`/`ArrowDown`/`Enter` 处理相争：该处理器绑定在查询 `<input>` 元素本身上，而 Radix 通过
  portal 渲染的下拉内容是一个**兄弟**子树——绝不是输入框的后代——因此菜单持有焦点时的 keydown 按构造
  无法到达输入框的处理器，而不是靠逐例守护。
- **Chord 归属：`Ctrl+R` 与 `Escape` 不是元素本地的。** 两者过去都是单元素按键处理器——`Ctrl+R` 在
  输入框 textarea 上，`Escape` 在浮层的查询 `<input>` 上——而两者都因同一个原因不对：它们只在该元素
  持有焦点时才触发。在其之外，`Ctrl+R` 会到达浏览器并**重载应用**，而一个输入框已失去焦点的浮层
  （点击回输入框、点击某行的图标按钮）**完全没有**键盘关闭方式。现在：
  - **`Ctrl+R`** 由 `shell/useGlobalHotkeys` 拥有——一个窗口捕获阶段监听器，在全应用范围吞掉该 chord
    （`preventDefault` + `stopPropagation`，因此它恰好只有一个处理器），并通过
    `store.requestHistoryOpen(sessionId)` 把它路由到唯一挂载的 `ChatView`
    （`selectActiveChatSessionId`）。`ChatView` 把它翻译为：浮层关闭 → `composerRef.openHistory()`
    （与历史按钮完全相同的路径——菜单关闭、草稿作种子）；浮层打开 → `cycleScope()`。`Composer` 与
    `HistoryOverlay` 都不再带 `Ctrl+R` 分支。有意的排除：来自终端内部（`.xterm`）的 keydown 原样放行
    （反向增量搜索属于 PTY），而 `Ctrl+Shift+R` / `Cmd+R` 保持不动，因此键盘重载仍然可能。
  - **`Escape`** 由 `HistoryOverlay` 自己的窗口捕获阶段监听器拥有，只在其打开期间注册。捕获 +
    `stopPropagation` 编码了"最上层的浮动面板先关闭"（输入框槽位会话能在该次忽略中存活，而不是被同一
    次按键清除）。它在作用域选择器打开期间让位——见上文受控 `scopeMenuOpen` 的说明——因此 Radix 自己的
    Escape 只关闭那个菜单。
  - **该浮层是一个 fzf 风格的模态选择器**（shell 的 `Ctrl+R` / fzf 历史小部件）：打开期间它拥有输入，
    只能通过*接受*（插入、插入并发送、跳转、另存为模板）或*放弃*（Escape、历史按钮、外部按压）离开，
    而放弃会让草稿保持原样。按进输入框也算放弃——选择器和 prompt 绝不会被同时编辑。每次打开都是全新的
    选择器：`HistoryOverlay` 只在其打开期间挂载其 `HistoryPicker`，因此 `scopeMenuOpen` 之类的局部状态
    无法带入下一次打开。外部按压是一个窗口捕获阶段 `pointerdown` 监听器，它像 Escape 一样在作用域选择器
    打开期间让位，因此第一次按压只关闭那个菜单（Radix 的模态菜单会把其外部的每次按压——包括浮层自己的
    查询——重定向到 `<html>`）。对历史按钮的按压标记了专用的 `data-history-trigger` 属性而不是它的
    test id，因此不算外部：该按钮切换浮层本身。
  - **忽略会把焦点还给 prompt 字段**（`ChatView.onDismissHistory` → 输入框的 `refocus` 句柄）：打开
    把焦点移进浮层的查询输入框，而关闭会卸载它，因此没有这一步，Escape 之后的每次按键都会落到 `<body>`
    上并丢失。光标回到原处（`Composer` 在 click/keyup/change 时跟踪它），或当槽位会话活跃时回到当前
    槽位的标记上。一次外部按压（`onOutsidePress` → `ChatView.onHistoryOutsidePress`）立即关闭，但只在
    浏览器自己的指针焦点处理之后（零延迟 timeout）才恢复焦点，且只在焦点落到 `<body>` 上时：按压空白处
    把光标交回 prompt，而按压一个会取焦点的控件（按钮、字段、终端）则保留它。放弃是*唯一*会恢复焦点的
    关闭——插入、跳转与另存为模板各自拥有焦点下一步该去哪（输入框、另一个 chat、一个对话框）。
- **历史浮层：仅 assistant 消息 + 可跳转的 prompt**（`HistoryOverlay.tsx`、`useHistorySearch.ts`——
  R3）—— `MESSAGES` 现在只包含 assistant 角色的命中（服务端已过滤；见
  `packages/server/src/history/SPEC.md`）：用户角色的命中永远是其自己那条 `PromptHit` 条目的文本重复，
  因此它过去添加的位置被移到 prompt 行上。现在每个 prompt 行都渲染一个前往 chat 的图标
  （`data-testid="history-jump"`、`aria-label="Go to chat"`，一个读作 "⇧⏎ go to chat" 的
  `IconTooltip`，位于既有的另存为模板图标旁）**当且仅当它可跳转**——`workspaceId` 存在且
  `messageIndex != null`（对未映射 cwd 的命中，或对不填充 prompt 锚点字段的宿主，则缺失）。点击它，
  或在一个 prompt 行是键盘选择时按 **`Shift+Enter`**，走的是消息命中 `Enter`/点击早已使用的同一条
  `onOpenMessage` 路径——两者现在都经过共享的 **`jumpTarget(hit)`** 辅助函数
  （`useHistorySearch.ts`，导出的纯函数），它把任一种命中形状解析为一个 `ChatLocationRequest` 或
  `null`，因此图标的渲染门控、`Shift+Enter` 处理器与消息命中门控在"这是否可跳转"上绝不可能不一致。
  未映射/遗留的 prompt 行不显示图标，`Shift+Enter` 是空操作（浮层保持打开）——即消息命中路径早已有的
  同一套双保险门控。图标本身保持悬停显现（`group-hover`/`isSelected` 透明度，像它旁边的另存为模板
  图标一样），但其快捷键字形（`data-testid="history-jump-shortcut"`，字面 `⇧⏎`）是一个**仅选中时**
  （而非仅悬停时）持久的 `<span>`——与作用域徽标 `⌃R`（始终在其标签旁）是同一先例——因为纯键盘用户
  正是 `Shift+Enter` 的受众，永远不会触发 `:hover`。另存为模板图标自己的快捷键
  （`SAVE_SHORTCUT_LABEL`、`⌘S`/`Ctrl+S`）出于同样原因获得完全相同的仅选中字形
  （`data-testid="history-save-shortcut"`），与跳转图标对称。无标题的命中通过 lib 的 `fileName` 以
  其 cwd 的最后一段命名（头部的 cwd 是 pi 的原生拼写，因此 `C:\Users\me\repo` 读作 `repo`）。
- **模板槽位（Template slots）**（共享 `prompt` 模块的解析器/状态机 + `Composer` 的几何 +
  `ChatView` 的目录/模板读取接线——即输入框 Tab 逐个跳转的占位符流程，端到端）。**解析**（`prompt`，
  纯函数）：`parseTemplateSlots(body, argumentHint)` 把 pi 自己的占位符语法（`$1..$n`、
  `$@`/`$ARGUMENTS`、`${N:-default}`、`${@:N}`、`${@:N:L}`——pi 的语法，单一所有者；见
  `packages/server/src/templates/`）展开成可见文本加 `TemplateSlot` 区间；`stripUntouchedSlots`/
  `shiftSlots` 补全会话的其余部分（发送时剥离、编辑时重新跟踪）——**只解析**，该模块绝不求值该语法
  （一次键入的 `/name args` prompt 早已通过 pi 自己的 `PromptOptions.expandPromptTemplates` 展开，
  无论有没有这个解析器）。**已观察**（设计的"待验证" #3，由 `e2e/templates.live.spec.ts` 的键入穿透
  测试解决）：pi 自己的会话记录为一次键入穿透的发送记录的是**已经展开**的正文，绝不是原始的
  `/name args`——`AgentSession.prompt()` 在持久化 `role: "user"` 消息之前把参数替换进 `expandedText`，
  因此一次 `session.getMessages` 重新抓取（重载，或从历史重新打开）显示的是展开后的文本。唯一的细微
  之处：web 客户端自己的即时气泡是一次**乐观回显**（`ChatView.onSubmit` → `appendUserMessage`，仅
  store，在传输层调用 resolve *之前*追加；附加的图片作为内容块随行，因此气泡会显示它们——`UserTurn`
  把图片块渲染为文本上方的紧凑"附件文件"徽标块（无内联预览；点击在对话框中打开图片，即图全屏模式）。
  徽标块标签是所拾文件的名称，作为 `attachmentNames` 携带在回显轮次上（仅 UI 侧——pi 的
  `ImageContent` 没有文件名），与图片块按索引对齐；水合的轮次没有名称，回退到 mime 类型标签）——
  它精确显示所输入的内容（原始命令），直到一次重新抓取用 pi 真实的持久记录替换它。**`/` 菜单合并**
  （`ChatView`）：pi 的 `commands` 快照（`session.getCommands`，在会话创建时冻结）减去其
  `source === "prompt"` 条目，加上一次全新的 `template.list { workspaceId }` 抓取映射为
  `SlashCommandInfo` 行（`source: "prompt"`，`sourceInfo` 合成得与 pi 自己的 prompt 模板约定完全一致：
  `{ path: filePath, source: "local", scope: scope === "global" ? "user" : "project",
  origin: "top-level" }`）——合成一个列表。chat 会在最前面加上它唯一的**浏览器原生命令**
  `/compact [instructions]`，作为一行显示本地的 `builtin`，标签为 `Pi/built-in`；contracts 中镜像 Pi
  的命令来源保持不变。原生 `compact` 在精确名扩展/模板冲突时被保留（skill 命令仍带命名空间），而确切
  的 Pi 解析器只识别 `/compact` 或 `/compact ` 加修剪后的指令——每个近似写法都仍是普通 prompt。一次
  compact 提交绕过乐观用户回显与所有流式发送模式：已完成的草稿图片**或队列中宿主书写的 `hasImages`
  聚合**会就地拒绝它并给出可操作的输入框徽标块（草稿 + 队列保留；待定的草稿图片已扣住所有提交）。否则
  该命令会被清除，把 `session.clearQueue { requireTextOnly: true }` 按 steer 先、follow-up 后的顺序
  排回输入框，然后调用 `session.compact`；宿主在破坏性操作处复查图片前置条件，因此过期的客户端或跨
  客户端竞速仍无法丢掉排队的字节。当该会话已有一次手动压缩在途时，宿主原子地拒绝第二次；Pi 拥有中止、
  摘要、持久化与生命周期。该请求会快照既有的压缩轮次 id，而被拒绝的 clear/compact 只在没有新生命周期
  轮次出现时才请求 store 追加一条失败压缩行，因此 Pi 发出的失败与生命周期之前的线上契约失败共享同一个
  界面而不重复。既有的实时/水合压缩渲染保持不变。当 `template.list` 响应回来为**空**时，
  `SlashCommandMenu` 渲染一条 `footer` 提示（`data-testid="slash-templates-empty"`），经 `ChatView`
  的 `onManageTemplates` 深链到 Settings → Templates——这是入门模板提供（`panels/SPEC.md`）的可发现性
  那一半，因为全新安装的全局 prompts 目录是空的，而管理器否则深藏在对话框里两次点击之后。门控条件是
  "不存在任何模板"，绝不是"当前查询没匹配到"，因此一个只是没命中的查询不会触发它；`footer` 是可选的，
  因此 `NewWorkspaceDialog` 对同一菜单的复用不受影响。**该门控是 `ChatView` 显式的 `templatesEmpty`
  prop——一个已解析的空列表——绝不是 `commands` 里没有 `source === "prompt"` 的行。** 合并列表在首次
  抓取 resolve 之前与一次抓取失败之后同样为空（那个 `.catch` 有意静默——列表失败不得破坏菜单），因此
  从它读取空会在这个 chat 首次按 `/` 时闪现"你没有模板"，并在列表失败后永久搁死该说法——而那一行的点击
  还会清空用户的斜杠草稿。抓取在**每次**斜杠菜单打开转换时运行（**`onSlashActive`**，一个镜像
  `onMentionQuery` 查询信号的布尔 prop——用户键入查询期间它保持 `true`，因此不会逐键重发），且刻意
  **不缓存**：prompt 文件也会在应用之外变化（pi CLI、编辑器、git pull），没有任何应用内失效计数器能
  看见——早先这里的 `(workspaceId, templatesVersion)` 缓存正是让那些外部变化的文件在此后整个 chat 里
  保持过期，而服务端为这份新鲜度恰恰每次调用都重读其目录（其 SPEC 称这些 readdir 很便宜）——这正是让
  `packages/server/src/agent/SPEC.md` 的"输入框的 `/` 菜单路径始终通过 `template.list` 保持新鲜"
  这一说法成立的原因，与键入穿透的 `/name args` 路径冻结的创建时快照不同。**选择模板**（`ChatView` 的
  `onPickTemplate`，一个 `Composer` prop）：不做普通的 `/name ` 插入，而是抓取 `template.get`、在
  客户端切分 frontmatter（`prompt` 模块共享的 `stripFrontmatter`——pi 自己的 frontmatter 解析器只在
  服务端，绝不进入浏览器打包，但边界规则被钉住以与之完全一致；见下文 Save-as-template 条目）、运行
  `parseTemplateSlots(body, argumentHint)`，并把结果通过 **`ComposerHandle.insertTemplate`**（与既有的
  `insertText` 并列）交给 `Composer`——替换整个草稿（像 `pickSlash`，而不是 `pickMention`：一个斜杠
  命令占据整个输入），并且若解析产出任何槽位，则启动一个**槽位会话**并选中槽位 0；无槽位 → 普通插入，
  光标在末尾，无会话。异步响应只在这次选择仍**当前**时才应用——最新选择胜出且草稿与选择时刻逐字节相同
  ——因此慢响应绝不会覆盖用户在此期间键入的草稿（或他们选的第二个模板）；在这次所选模板读取待定期间，
  共享选择器禁用 `Composer` 提交，使一次快速的 Enter 无法发送原始斜杠 token。该规则位于 `prompt` 共享
  的模板选择控制器中，并针对延迟与乱序响应做了单元测试。**会话**（共享 `prompt` 状态，由 `Composer`
  本地持有，无 store/传输层）：`Tab`/`Shift+Tab` 步进到下一个/上一个槽位（回绕；`preventDefault`；
  在提及/斜杠菜单打开时为空操作——在 `onKeyDown` 顶部检查，早于菜单自己的按键处理，因此一次真正的
  Tab 选择菜单项不受影响，对称地，菜单也打开时按 `Escape` 会让菜单自己的关闭先胜出）。从*用户编辑过*
  的槽位（其文本确实被用户改过——不是未触碰的标记，尤其也不是未触碰的 `${N:-default}`）**步出**时，
  把其当前文本拼接进共享其 `group` 且文本不同的每个其他槽位（分组镜像——重复的 `$N`/`${...}` 出现
  在槽位退出时传播，而不是逐键传播），每次拼接都通过共享 prompt 槽位状态机重新跟踪。一个槽位携带两个
  独立的位：**`filled`**（解析期属性——有真实内容：`${N:-default}` 的默认值，或一个被键入内容的标记
  ——驱动发送时剥离 + 着色）与 **`edited`**（会话运行期状态——用户改过它——唯一的镜像*来源*门控）。
  若用户把一个未触碰标记的选区收缩到其末尾并在那里键入，可见标记加上键入的后缀会变为 filled 并在发送
  时保留：这是所见即所得，而剥离长出来的区间会把用户的文本连同标记一起删掉。两者刻意不同：
  `${1:-foo} … ${1:-bar}` 出生即 `filled` 但不是 `edited`，因此它两个不同的逐出现默认值保持独立，直到
  用户通过编辑其中之一来给出参数——与 pi 自己的展开一致，后者从不把 "foo … bar" 改写为 "foo … foo"。
  `Escape` 结束会话（`setSlots(null)`），文本保持原样。一次真正的文本编辑（textarea 自己的
  `onChange`——绝不是程序化调用 `onChange(text)`；那些反而会直接结束会话，因为
  `pickMention`/`pickSlash`/方向键召回/`insertText` 都不参与槽位跟踪）围绕编辑后的 `selectionStart`
  对旧/新值做差分（一次公共前缀/后缀扫描），得到 `(editStart, removedLen, insertedLen)`，通过
  `shiftSlots` 重新跟踪每个槽位，并把编辑落在的槽位标记为 `filled: true` **且** `edited: true`；
  一次吃掉**整个**先前值的编辑（全选后输入/删除）会直接结束会话，而不是重新跟踪一组此刻已无意义的收缩
  区间。发送时，`submit()` 对**每个用户编辑过的槽位**运行同一趟分组镜像，而不只是最近一次 Tab 步出的
  那个（`mirrorAllGroups`——一次直接的 Send 无需先经过 Tab 就能让其镜像生效），把每个传播进其同组兄弟；
  只有**那时**它才剥离仍是未触碰的标记（`stripUntouchedSlots`），并始终清除会话——已发送**或**已排队
  （steer/followUp），规则相同。切换标签页无需显式清理：工作台可见性门控只挂载一组中本地选择的主体，
  因此离开 chat 标签页会卸载 `Composer`（及其会话），而 store 的 `draft` 文本本身持久存在。**提示
  徽标块**：会话活跃期间（且菜单不活跃，因此两个绝对定位的浮层绝不共享同一锚点矩形），textarea 上方
  一枚小药丸——`slot {slotIdx+1}/{n} · ⇥ next · esc done`（`data-testid="slot-hint"`）——可点击，
  点按步进到下一个槽位（与 `Tab` 相同的镜像规则），即无需键盘的移动端路径。**高亮背板**：会话活跃
  期间，输入框的间隙在消息字段本身中被视觉着色——原生 `<textarea>` 无法为其内部的文本区间设样式，
  因此 `Composer` 渲染一个**高亮背板**（一个样式化镜像层，定位在如今为 `bg-transparent` 的 textarea
  之后；输入背景改为移到包装容器上，并裁剪到同一个 `rounded-md`，因此会话之外视觉上毫无变化）。该包装层
  拥有输入边框与填充：`bg-clip-padding` 让背板着色留在圆角边框之内，而
  `focus-within:border-control-border-active` 是输入框唯一的焦点指示器，而不是 textarea 上的第二个强调
  环。纯函数 `highlightSegments(value, slots, activeIdx)`（来自共享 `prompt` 模块）把 `value` 拆成有序
  的 plain/unfilled/filled/active 段——当一个槽位区间的 `slots` 索引等于 `activeIdx`（`Composer` 自己的
  `slotIdx`）时该区间为 `"active"`，否则按其自身的 `filled` 标志为 `"unfilled"`/`"filled"`；其他一切
  都是 `"plain"`——纯偏移/切片，零间隙相邻槽位不产生空段，且测试在每种情况下都钉住
  `segments.map(s => s.text).join("") === value`。背板的内部镜像 div **精确**匹配 textarea 的盒模型
  （`px-12 py-8`、相同的 `tr-text-ui` 排版类、同宽度的 `border border-transparent` 使内容盒对齐、
  `whitespace-pre-wrap break-words`——显式写出来，因为 `<div>` 不像 `<textarea>` 那样默认这样软换行），
  因此每个 `SlotSegment` 的着色 span（`data-testid="slot-highlight"` + `data-slot-state`，`rounded-xs`
  ——文本段圆角档——unfilled/active/filled 分别为 `bg-primary-soft`/`-muted`/`-subtle`，plain 不着色，
  每个 span 都 `text-transparent`，使只有其上方真正的 textarea 文本透出）恰好落在其自身字符之下。
  **滚动同步**：textarea 的 `onScroll` **命令式地**（一个 ref——无 state、无内联 `style`：程序化滚动
  偏移根本不需要样式，因此仓库"只用 token 工具类"的不变量零例外成立；早先的版本在 state 中跟踪偏移并
  应用 `translate(...)` 内联样式，既违反该不变量又让输入框在每个滚动帧重渲染）把其
  `scrollLeft`/`scrollTop` 复制到背板的外层 `overflow-hidden` 层上。背板的 **ref 回调**在挂载时播种
  这些偏移，因此在一个已滚动的输入框中开始的会话绝不可能画出哪怕一帧错位。
- **原生 `/name`** —— 浏览器原生命令目录在 `/compact` 旁新增 `/name <title>`，标签为 `Pi/built-in`，
  并在精确名扩展或 prompt 模板冲突时保留，以匹配 pi 自己的命令。解析器同时保留裸 `/name` 与
  `/name <title>`；有效参数会绕过用户消息回显与 agent 发送，调用 `session.rename`，然后清空输入框。
  空白/超限输入留在输入框中并给出可操作的校验错误；传输层拒绝则不改变持久标题，并作为 chat 内错误呈现。
  该命令对早于会话重命名功能常量的宿主隐藏。它是通往与 shell 的标签/历史控件同一域变更的键盘路径——
  绝不是独立的标题来源——而自动生成没有 ChatView spinner 或会话记录行。
- **另存为模板 + 模板管理（Save-as-template + template management）**（`TemplateEditorDialog.tsx`；
  `HistoryOverlay` 的保存动作；`panels/TemplatesSettings.tsx`）——一个共享的 prompt 模板文件创建/编辑
  界面，被两个彼此从不对话的入口复用：Settings → Templates 面板（列表 + New/Edit/Delete，见
  `panels/SPEC.md`）与下文历史浮层的另存为模板动作。**为什么这在 `chat/` 而不是 `panels/`**
  （一个刻意的边界例外，与上文的 `ChatView.tsx`/`useHistorySearch.ts` 并列）：`panels/` 允许从
  `chat/` 导入（已经在做，用于 `ModelEffortPicker`/`ModelSelector`/`ThinkingSelector`/`Markdown`）
  但反向绝不允许，而 `HistoryOverlay`——需要这同一个对话框——位于 `chat/`，因此唯一的共享实现必须放在
  双方都能到达的地方。所以 `TemplateEditorDialog` 被提升为**第三份**被认可的、触碰 store/传输层的集成
  件（见下文 Boundary），即使它不是 `ChatView` 本身。
  - `prompt` 模块的**模板文本 API** 是唯一共享的 frontmatter 切分器/装配器——`stripFrontmatter`
    （`ChatView.tsx` 的输入框选择路径 + 本对话框的正文字段）、`assembleTemplate`（本对话框的保存）。
    它**不做 YAML 值解析**：对话框的描述/参数提示字段由服务端解析的 `template.get` 响应填充
    （`Template`——pi 真正的 YAML 解析器作用于**整个文件**，完整的标量风格保真，钉在
    `packages/server/src/templates/templates.test.ts`），绝不用浏览器侧的重实现（这里早先的
    `splitTemplate` 只处理裸/双引号标量，因此一个 pi 原生的 `description: 'single-quoted'` 会带着字面
    引号载入表单，并在保存回去时损坏）。其边界规则镜像 pi 自己的 `extractFrontmatter`
    （`@earendil-works/pi-coding-agent` 在目录钉定的 pi 中的 `dist/utils/frontmatter.js` +
    `dist/utils/text.js`——即 `packages/server/src/templates/SPEC.md` 在服务端依赖的同样事实；pi 升级
    时需重新验证两者）：剥离一个前导 UTF-8 BOM，规范化换行，然后把 frontmatter 块结束在**第一个**其后
    的 `\n---` 行；正文是那道围栏之后的一切，经过 `.trim()`——不是单个可选的 `\n`。早先版本有两个各自
    手写正则切分器（每个文件一个），各自只消费闭合围栏之后一个*可选*的 `\n` 而不是修剪——每次选择、每次
    编辑重开都会有一行前导空行漏进正文，并**累积**为每个"编辑-保存"周期多一个 `\n`（漏出的行被保存回
    正文字段，并在下一次保存时再包一层）。`templateText.test.ts` 钉住这处修复所依赖的往返/稳定性属性。
  - **字段**：name（在客户端按与服务端 `isValidTemplateName` 完全相同的规则校验，
    `packages/server/src/templates/templates.ts`——重复实现而非共享，因为它是 4 行纯谓词而服务端模块
    只在服务端）、一个作用域单选（Global / This project；无活跃工作区时 "This project" 禁用）、
    description、argument-hint，以及一个正文 `Textarea`，带静态一行语法提示（
    `$1, $ARGUMENTS, ${1:-default} — pi prompt-template syntax`；真正的语法由 `prompt` 模块解析 /
    由 pi 展开——这一行只是文档文本，它本身不被解析）。
  - **装配**：`---\ndescription: …\nargument-hint: …\n---\n\n<body>`，当某字段为空时省略对应键，
    而当两者都为空**且正文不以 `---` 开头**时**完全不加 frontmatter 块**——若正文确实以之开头，则仍强制
    加上一个显式（可能为空，`---\n---\n\n`）的块：若裸存，pi 自己的加载器（以及本仓库的切分器）会去该
    正文内部寻找一个*更晚*的 `\n---` 行当作闭合围栏，一旦正文包含这样一行，就会把真实内容静默当作 YAML
    吞掉；强制包装使本仓库自己的围栏无条件成为最早可能的匹配，因此正文自己的 `---` 样式行永远不会被
    重新解释（见 `templateText.test.ts` 的歧义正文用例）。每个值都以 `JSON.stringify` 引号形式发出，
    而不是裸值——YAML 的双引号标量转义集是 JSON 的超集，因此这永远是合法 YAML，无需仅为序列化两个短
    字符串而引入 `yaml` 包（见种子夹具自己的 `argument-hint: "[file] [scope]"`，出于同样原因加引号：
    未加引号的值并不总是合法 YAML）。
  - **编辑既有模板会锁住其名称 + 作用域**（两个字段均禁用）：`template.save` 是以 `(scope, name)` 为键
    的创建或覆盖，没有重命名/移动原语，因此在编辑时改变任一者会静默把旧文件遗弃在磁盘上，而不是重命名
    它。新建（包括另存为模板）两者都完全可编辑。**编辑保存时逐字使用 `template.name`——绝不修剪或
    规范化**：带空白的名称在服务端是*有意*合法的（pi 逐字从文件名派生模板名，因此手工创建的
    `report .md` 会列为 `report `；`packages/server/src/templates/templates.ts` 的门控刻意接受每个 pi
    可列出的名称），而在保存时修剪会写一个**新的** `report.md`，同时让正在编辑的文件保持不变（评审者
    指出；`templates-manage.spec.ts` 钉住该往返）。Save 按钮的空值门控出于同样原因仅限新模式——纯空白
    的手工名是合法的编辑身份。只有**新**模板键入的名称在验证/保存前被修剪：刻意的表单规范化，使意外的
    尾随空格无法铸出一个在每个列表里都与修剪后的孪生兄弟渲染相同的文件（输入框的 `/` 菜单可以经点击
    *使用*这类手工名，但键入的 `/name` token 无法携带空格——UI 不应制造二等名称）。
  - **编辑打开会抓取完整模板**，通过 `template.get`，钉在该行的确切 `(scope, name)` 上——`template.list`
    按设计只含元数据（有界头部扫描 + 尺寸上限，见 `packages/server/src/templates/SPEC.md`），因此列表行
    无法提供正文，而且——这才是踩到的坑——元数据也不可信：一个 frontmatter 闭合围栏位于列表扫描窗口
    之外的文件会*合法地*在列表中显示为**没有** description/argument-hint。因此 `get` 响应**对每个字段
    都是权威**：正文经 `stripFrontmatter(content)`，description/argument-hint 来自其全文件解析，替换那些
    仅用于即时绘制的列表行种子值。（否则就是评审者指出的数据丢失：从降级的行播种并在 Save 时把这些
    字段写回，意味着一次仅改正文的编辑会静默删掉文件真实的 description——
    `templates-manage.spec.ts` 钉住该往返。）在抓取 resolve 之前，description/argument-hint/正文输入
    被禁用且 Save 被门控（`loading`）——过早保存会用降级的种子覆盖文件；抓取失败同样让 Save 保持门控
    （错误内联显示，重新打开以重试）。
  - **Save** 调用 `template.save` 然后调用 store 的 `bumpTemplatesVersion()`；被拒绝的保存通过
    `data-testid="template-error"` 就地渲染其消息（绝不是 toast——对话框保持打开，使错误可就地修正）。
    **Delete 完全不经手对话框**——`panels/TemplatesSettings.tsx` 的行直接调用 `template.delete` +
    `bumpTemplatesVersion()`，位于锚定到该行自己 Delete 按钮的 `ConfirmPopover` 之后。**被拒绝的
    delete** 是与 Save 之间唯一的刻意不对称：它作为错误 toast 呈现，而非内联（没有对话框可供内联渲染），
    该行保持原处——即 `panels/SPEC.md` 为 `ProjectTree` 自己的工作区移除行所记录的同一模式。
  - **另存为模板**（`HistoryOverlay` 的选中 prompt 行动作，`data-testid="history-save-template"`，
    在选中 prompt 行时键盘 **Cmd/Ctrl+S**——浮层的 `onKeyDown` 始终对该组合 `preventDefault`，因此无论
    选中什么，浏览器自己的保存对话框都不会打开，但只有在解析出的选择是 prompt 命中时才触发动作）打开
    对话框，正文以该 prompt 的文本预填——一个"新模板"情形（没有既有名称/作用域身份），与点击 New 相同。
    **输入框溢出处的入口作为 YAGNI 被砍掉**（计划自己的用词）——历史路径已覆盖"复用我已写过的内容"；
    为同一个"先打字、再决定保存"手势再加第二个入口只会是冗余表面，而不是一个不同的用例。
  - **作为文件编辑**——只有项目作用域的行获得 `Open as file` 动作（`panels/TemplatesSettings.tsx`），
    复用 `openTabs.ts` 的确切 `openFileInTab(workspaceId, ".pi/prompts/<name>.md", "keep")`（与文件树
    点击使用的同一动作）——刻意用 **`keep`** 意图，因为一次显式的"在编辑器中打开"绝不该落在预览槽里、
    被之后的一次浏览点击静默替换（见 `panels/SPEC.md` 的 Preview tabs 条目）——然后
    `store.closeSettings()`。**全局行只走对话框**——这是刻意的不对称，不是疏忽：文件布局引用是工作区
    作用域的，而全局模板位于宿主的 agent 目录下、任何工作区之外，因此没有可据以打开它的工作区相对路径。
- **纯 `↑` 召回 + 历史按钮** —— `Composer` 的 `recentPrompts` prop（`ChatView`：本 chat 自己的用户
  轮次文本，经 `turnAnchorText` 取得，最新在前，去重时**保留最新的那次出现**——与服务端历史索引相同的
  最近优先排序规则，即 atuin/fzf 惯例）支撑一个轻量召回会话（`recallIdxRef`），其门控确保它绝不吞掉
  草稿：`↑` 只在字段为**空**或召回已活跃时步入（更旧 → 更高索引），`↓` 步向更新（越过最新则恢复
  `""`），任何分歧编辑或一次提交都会退出会话，而召回的文本落地时光标在其末尾。会话索引是一个
  **ref，绝不是 state**：没有任何渲染从它出发，且步进是*在这里*写索引，而草稿经 `onChange` 去往
  **父级的** store，因此若作为 state，两者可能在各自不同的批次中提交。在那个窗口里，textarea 已显示
  召回文本，却仍带着上一次渲染的处理器与其过期的索引——第二次 `↑` 会重新召回同一条目而不是步进，
  而一次编辑也未能结束会话，于是下一次 `↑`/`↓` 会从实时索引步进并**覆盖用户刚键入的内容**（即
  `replaceDraft` 在插入路径上守护的那种丢失，此处改经键盘路径到来）。ref 读到的是其最后写入的值，
  因此提交顺序无法介入。处理器对**每个事件取一次快照**——ref 不可能在一个同步处理器内部改变，而单次
  读取可以收窄类型，重复读 `.current` 则不行。一个 `History` 图标幽灵按钮
  （`data-testid="history-open"`、`aria-label="Search history"`，同名 tooltip，始终渲染在发送旁）
  调用全局 `Ctrl+R` 所到达的同一个 `openHistory`——移动端是点按路径，桌面端是可发现性入口。
- **Chat TODO 计划** —— chat 的 `pi-todos` 列表，呈现在 TODO 侧栏与 Plan 页面（引擎：
  [[module-pi-todos]]；宿主读/写：[[submodule-server-todos]]）：`useChatTodos`（`todo.*` 数据 hook
  ——抓取 + 实时 `pi.event` 重抓 + 编辑 + 追加提示 + **评审快照重抓**（计划的评审装饰由宿主派生，因此
  对工作区评审评论的改动，例如删除一条使某步骤的 `changes_requested` 被清除的发现，会重读计划）
  + `openMarkdown` 快照动作；工具完成立即刷新，而 `agent_settled` 提供最终刷新；重叠的列表读取是最新
  胜出且带连接世代戳，被接受的追加按条目 id 折叠；`remove` 乐观地把行剔除，而宿主拒绝时重新加载权威并
  抛出错误 toast；外加 agent 评审操作 `startReview`（`todo.startReview`）与 `reviewAll`
  （`todo.reviewAll`）——两者都重读计划，因为评审装饰由宿主派生、绝不在本地打补丁；手动判定操作已随
  计划页的手动模式移除——`todo.review`/`todo.requestFix` 仍留在线上契约上，属宿主侧）、`planView`
  （对 DTO 的纯派生：`groupProgress`、`planSummary`、`planGlance`/`sessionGlance`（按其输入的身份
  记忆化，因此 store 选择器可以把它当原始值返回，而不会在无关更新上重新派生 ask 状态）、
  `planSections`、`shouldNudgeOnAdd`，以及评审轨迹集合——`itemRevisions`（提交历史，1 个 TODO = N 个
  提交）、`reviewableItems`/`reviewProgress`（由 `TodoItem.review` 的存在性在宿主侧门控——可评审规则
  只有**一个**归属，在服务端；该集合覆盖计划的条目**以及** `TodoPlan.adoptedCommits`，因此 Review
  阶段/Review All 覆盖在计划之外提交的工作，而 `planSummary` 的构建 `done/total` 只统计计划条目）、
  `reviewChangesRequested` + `itemOpenFindings`（changes_requested 警告标记：该标志与评审者未关闭评论的
  计数——在打戳时按发现项的 `origin` 来源（todoId + 可选 sessionId）匹配，只有在无来源的评论上才回退到
  变更集路径关联，因此两个步骤触碰同一文件不会互相计入对方的发现项；Review 标签页才是真相），以及
  `planCompletionSummary`（agent 的计划级备注，以"全部完成"为门控，因此重新打开的计划绝不会把过期的
  全完成备注泄漏进未门控的输出——它供给 markdown 导出；计划页还显示一段派生的单行回顾，见
  `panels/SPEC.md`）以及它仅用于计划页的伴生 `planStaleSummary`（同一条存储备注，在某条目于完成之后
  重新打开时被呈现并标记为过期，使回顾留在页面上而不是在 agent 改写之前消失）与 `planChangeTotals`
  （那段回顾背后的全计划去重文件计数）。`itemChangeSet` 的优先级：实时 `change` 路径优先（一次回退
  重做的最新增量），否则用**最新的**可解析提交。一个组的*状态*在这里**不**派生——宿主计算它并通过
  `TodoGroupItem.status` 下发，因此该规则只有一个归属；所以用户编辑会重读计划而不是在本地打补丁，见
  `useChatTodos`）、`TodoList`（**按状态排序、组优先**的渲染（`planSections`）——组即任务：**进行中**
  的任务（其整组）在最上且**没有段头**，然后是一个 **To do** 段（待定组，然后是用户的待定散条目），
  最后是一个非常底部的 **"Done" 标签**，在其下**每个已完成任务都是自己可折叠的行**（折叠——标题 +
  `N done`）加上已完成的散条目（不是把整个 Done 折成一处）。已完成的*步骤*仍内联在其（活跃/待定）组里；
  只有整个完成的任务移到 Done。每个组是一行头部（派生状态图标 + 标题 + done/total 徽标），`active` 组
  被强调；用户的散条目携带逐行的 `user` 徽标（没有单独的 "Your requests" 头——它们按状态放置）。
  **紧凑列表只有标题**（状态 glyph + 标题 + 变更集徽标块）——行的 `note`、已完成条目的 agent 书写
  `summary`，以及其 `verification` 在这里**不**显示，因此长计划能一眼读完而不过载；**完整计划页**
  （`PlanPane`）才是它们出现的地方：`summary` 作为 **Markdown**（一段弱化的结构化备注——引导句 +
  要点），`verification` 作为共享的 **`VerificationBadge`**（`planKit`；一个状态 glyph——点名检查打勾、
  诚实的"未验证"用警告，该切分由 `planView.verificationStatus` 派生，唯一归属——置于作为 **Markdown**
  渲染的验证内容旁，因此多项检查读作要点而不是一行连写；该徽标的标题标明它属于自述——绝不是宿主运行的
  门禁）。计划页没有页内评审列表——其头部 kebab 提供 **Review All**（宿主侧队列，`todo.reviewAll`）与
  一个把焦点移到右栏 Review 标签页的评论徽标块（见 `panels/SPEC.md`）。评审已**结算**的行
  （`planView.reviewSettled`——已批准且此后无新内容落地）把其完成勾升级为**圈版 Verified glyph**
  （`StatusIcon reviewed`，悬停 "Verified"，`data-reviewed`）——即一眼可见的"该步骤已评审"状态，侧栏
  与计划页一致；**changes_requested** 判定则把 glyph 翻成警告色 `CircleAlert`（`StatusIcon
  changesRequested`，悬停 "Changes requested"，`data-changes-requested`）——计划页再补上徽标块 + 反馈
  备注（见 `panels/SPEC.md`）。**条目携带宿主变更集的行会长出一枚安静的 "N files" 徽标块**
  （`planView` 中的 `itemChangeSet`——与下文 markdown 快照共享的那一个派生，因此两者绝不可能不一致）：
  **已提交**条目的徽标块经 `useChatTodos.openChanges` 在 Changes 面板以该提交的 `commit:{sha}` 作用域
  打开它（`setDiffScope` + 一个 shell 的 `reveal-tool` 意图——面板自己列出该提交的文件；N = DTO 中
  宿主派生的 `commit.files`）；**路径列表回退**直接深链单一路径的实时 diff（先把作用域钉回 `branch`，
  因此它无法继承上一次点击留下的提交作用域），或展开一份内联路径列表。sha 已无法解析的提交产物**不带
  `files`** → 没有徽标块，绝不是一个坏掉的 diff 标签页（降级契约）。另有追加行 + 一个 **"Open the plan
  page"** 按钮（`todo-open-plan`）——`useChatTodos.openPlan` 打开（或聚焦）该 chat 的**实时计划页**，
  一个由 `panels/PlanPane` 渲染的中央 `plan` 标签页（见 `panels/SPEC.md`）；其标题通过 store 的
  `selectChatTitle` 解析该 chat 的名字（唯一归属，与该面板共享）。**状态排序仅限 UI**——agent 的
  `formatPlan` 保持计划顺序，因此其"按顺序工作"的纪律不受影响）、`planMarkdown`（一个纯 `plan →
  markdown` 编译器，`## <group> — n/m` 段落——计划页的**导出**（复制 / 另存为 .md），绝不是交互面：
  已完成条目的变更集渲染为其短 sha + `N files · +A −R` 以及带状态字母的逐文件行，**纯文本、无链接**
  ——导出会离开应用，在那里链接方案是死的；交互式导航是计划页的职责。它还为
  `TodoPlan.adoptedCommits` 发出一个 **`## Committed outside the plan`** 段落（镜像 `unattributed`
  的 `## Outside the plan`），因此导出覆盖没有步骤拥有的分支提交；当任一段落存在时，`No items yet`
  占位符被抑制）。紧凑 TODO 列表是侧栏工具（`panels/PlanRailPane.tsx`，见
  [[submodule-web-panels]]），而 Plan 页面是通过 `openPlan` 打开的中央标签页；chat 视图自身不渲染任何
  计划表面。**自动摘要触发器**（一个全完成且没有 agent `summary` 的计划会触发一次尽力而为的
  `todo.generateSummary`，折入一段宿主起草的备注——在计划重新打开或其摘要被清除时重新武装，绝不覆盖
  既有备注）是仅副作用的 hook `useAutoPlanSummary(workspaceId, sessionId)`，由
  `shell/WorkspaceWorkbench` 通过每个唯一 sessionId 一个 `PlanSummaryAutoDraft` 锚点挂载（覆盖任何
  打开的、会话支撑的标签页：chat、实时 Plan 页面、markdown 计划 `doc` 导出），该唯一 sessionId 经
  `editorSessionId(tab)` 取得，因此无论选中哪个标签页、打开哪个计划表面，它都会触发。计划写入会向同一
  chat 的其他已挂载 `useChatTodos` 实例发信号去重抓。前端本地的工作区视图状态把 Plan 页面持久化为一个
  已注册的 `todo-plan` 引用（解析器种类 + 会话身份，绝不是计划内容）；另一个客户端可以从宿主拥有的
  TODO 计划显式重新打开同一个实时页面，而不继承其放置位置。（早先的设计把计划编译成一个静态 markdown
  `doc` 标签页，并带自定义的 `thinkrail-diff:` 链接方案——已替换：agent 一改状态快照就在说谎，而
  markdown 无法携带 Changes 面板的可用性；现在页面是实时的，markdown 被降为它的导出。）
  **速览状态**让计划诚实地充当用户的状态窗口：记忆化的 `sessionGlance(runtime)` 选择器（对会话的流式
  标志 + ask 状态运行 `planGlance`）由 TODO 侧栏读取，并通过 `hostSessionGlance` 叠加在宿主
  `SessionState` 之上，**绝不存储**，因此 agent 无法让它说谎——它把 `in_progress` 步骤渲染为工作中
  （点）、**等待你的回答**（`MessageCircleQuestion`——与 `ask_user_question` 卡片相同的 glyph，在某个
  实时工具阻塞或某重启修复的会话等待期间），或**已暂停**（`CirclePause`，其他任何停止：轮次结束、
  错误）。没有待定问题的停止绝不会声称用户欠一个答案。速览的工作/等待生命周期来自规范化的宿主
  `SessionState`——在 TODO 侧栏中，并通过 `sessionGlance` 在 Plan 与 Review 面板中——因此一个在其后台
  命令或 subagent 上空闲的 chat 读作工作中（`isSessionWorking`）；会话记录派生的速览只是较旧宿主的
  回退。交互状态仍只用于识别并渲染确切的那份问卷/回顾。**未读结果在被看到时清除**（没有逐 chat 的
  已读手势；工作区菜单的标记已读/未读是由 [[submodule-web-store]] 拥有的显式例外）：
  `ChatTranscript` 在窗口有焦点且可见、chat 被关注（`selectChatAttended`：在活跃工作区最近聚焦的中央组
  中被选中，且没有为其打开或请求历史浮层）、Resources 检查器与任何打开的模态对话框都未覆盖会话记录
  （`@thinkrail/ui/dialog` 的 `useModalLayerOpen`，它标记每个模态浮层）、且会话
  记录跟随最新边缘（`followState`）时，在那个确切结果行渲染之后（`selectReadyCompletionAck`）确认确切
  的未读完成。在用户跟随时落地的结果绝不会一直亮着；在用户阅读历史、待在另一个组，或离开时落地的结果，
  会在他们回到该 chat 的最新边缘时清除；打开一个落地在最新边缘的 chat 会立即清除它。被动可见的相邻
  面板或未聚焦的第二客户端绝不替另一个读者清除标记。瞬时确认失败以有界、有上限的退避重试；重新进入
  已见状态会重新武装那个确切的 id。`TodoList` 保持 props 驱动——它接收解析好的速览，绝不读取传输层。
  其段标签 + 待定/活跃/完成状态 glyph 位于 **`planKit.tsx`**——Review 面板（`panels/ReviewPanel`）
  复用的共享展示型原子件，使两个"按段落呈现工作项"的表面读起来完全一致。
  **追加提示尊重该等待状态。** 用户追加总是把条目存下来（散条目，放在末尾）。在协议 v73+ 上，
  `session.nudge` 让宿主权威地、原子地做阻塞/执行决策：needs-input 为空操作，运行中则排队，空闲则提示。
  独立发布的客户端只在较旧宿主上保留先前的基于速览的提示/跟进加回退水合；该兼容路径会跳过一个等待中的
  问题，而不是越过其阻塞者唤醒 agent。

## Chat 资源（Chat Resources）

[[submodule-web-chat-resources]] 拥有三个呈现表面——头部触发器、输入框停靠区、会话记录区域的检查器
——以及它们共享的行法（row grammar）。`ChatView` 用 `useChatResources` 集成 hook 组合其 barrel：
它把触发器挂载在头部，把停靠区挂在队列条与输入框之间（仅在检查器关闭时），并把检查器挂在会话记录区域
内部，使头部与输入框保持可用；它拥有打开/选中状态，并填充检查器的细节槽——命令用由 `useCommandLog`
供给的 `CommandLogView`，subagent 用 `SubagentTranscriptPane`——以及 subagent 细节头部的
`renderSubagentModel`（用 `RunModelLabel`）。不涉及任何新的 shell 面板、工作台资源种类或终端附着。
工具/命令完成渲染仍留在对话基础组件中，通过工具/自定义消息名而非导入能力包来连接；最新轮次分隔线携带
"still running" 徽标块，作为父级提供的深链进入检查器。resources 视图还有第三个列表，即会话的 MCP
服务器（`SessionResources.mcpServers`）：`useChatResources` 只在支持 MCP 的协议上传递它（较旧的宿主
省略该字段；两种情况都没有列表），并以 `mcp.setSessionOverride`（在本 chat 中 Disable / Enable，在
下一个空闲重载时应用）与 `mcp.reconnect` 扩展共享控件，键为 `mcp:<name>`，位于停止动作旁。
`ChatView` 把已连接/正在启动的服务器加到触发器的计数上而不让它呼吸，在某个服务器需要登录、失败或等待
批准时于触发器旁挂载 `ResourcesAttention`，并把该标记与各行的 Sign in / Review / Edit 深链到
`openSettings(SettingsSection.Mcp)`。当检查器打开在一个有服务器的 chat 上时，它运行
`watchMcpWorkspace`，在每个已连接世代重启，使重连获得一次全新的有界读取：宿主在服务器完成连接时不发布
任何边缘，而 `mcp.list` 重读正是让它刷新状态并推送 `session.resourcesChanged` 的东西。分隔线的
"still running" 徽标块与停靠区只持续统计工作。不显示 PID——pi 不暴露——也没有逐服务器的立即停止。

**新结果（New results）。** `useFreshResults` 在 agent 收到它之前、或不论其何时收到，都告知用户存在
一个后台结果（用户结算，2026-10："Resources announces, the chat marks receipt"）。一个已完成的命令或
后台（`detached`）subagent 在被用户打开之前都是**新的**——打开可以是它完成后对检查器的选中，或是它的
chat 结果行（`ChatActions.markResultSeen`）——或者直到发送下一条消息。`ChatView` 把它们的计数传给
停靠区的完成行与触发器，并把该行的 Open 落在最新的新结果上，否则落在最新的已完成结果上
（`newestResourceId`）。已见集合是按 chat 的、带宿主限定的 localStorage 键下的客户端 UI 状态：一个
客户端的首次访问会把所有已完成的都基线化为已见（在该首次权威快照之前的一次确认会让基线保持待定），
之后的访问会呈现离开期间完成的，而每次发送都会把集合重置为当前已完成列表，因此它仍以快照为界。只有
快照窗口内的近期已完成记录（`groups.recentFinished`）可以是新的：此后分页进来的历史早于任何本可看到
它的访问，因此绝不会亮起。

依赖边是 `ChatView`/`useChatResources` → `resources`、`store`、`transport`，`BackgroundResult` →
`resources`（`ResultRow`、作为其 `metaLead` 的运行标签）、`tools/subagent`（`runDetails` 辅助函数）
与 `ModelEffortLabel`，以及 `ChatView` → 会话记录面板。`resources` 子模块保持仅 props，且不导入任何
兄弟工具实现；它把共享的 30 s 时钟作为 `now` prop 接收。命令日志由集成 hook 抓取并传入其只读视图；
该模块从不加载 xterm。

该 hook 在挂载/当前 welcome 时水合，订阅 `session.resourcesChanged`，并把失效合并到一次在途读取之后。
一次读取期间的失效需要一次全新的遍历；[[submodule-web-store]] 拥有世代/修订栅栏的快照安装与失败处理。
检查器关闭期间元数据保持最新；头部计数只对权威快照是数字，否则显式未知。一个证明宿主早于该能力的
welcome 会关闭检查器及其 Stop-all 确认，而重连期间未知的协议只是让它们变为过期。命令输出与 subagent
会话记录只在该行是检查器当前选中项时才刷新。共享的 `detailPolling` 循环处理命令输出与 subagent 会话
记录读取：单飞替换快照，在终态/永久不可用的结果上停止，以及带可见可重试失败的有上限瞬时退避。资源控件
的待定/错误状态限定在其动作与当前连接之内；确认与细节关闭的焦点语义属于
[[submodule-web-chat-resources]]。更旧的历史是又一个这样的控件 `loadOlder(kind)`（键为
`history:<kind>`，只由处于 `RESOURCE_HISTORY_PROTOCOL_VERSION` 且其快照携带 `history` 的宿主提供）：
它向 `session.resourceHistory` 请求 store 所持有最旧已完成记录之前的那一页，并交给 store，store 只在
它仍延续那条链时才保留它。`ChatView` 拥有检查器按种类的展开状态，与其选中项并列，并在每次打开时重置
两者；它把当前的新结果作为检查器的 `newIds` 传入，并告诉命令日志其行是否被中断、宿主是否保留日志
（`retained`）。头部计数不需要任何逐 token 的 subagent 进度或工具结果改写。

`backgroundCommandCompletion` 是一条中断折叠的历史行，也是一个回合开场，由 contracts 的守护在实时归约
与水合两条路径上识别，并携带消息 `timestamp`。其行（`BackgroundCommandResult`）把终端摘要与有界输出
渲染为转义的纯文本，绝不是 Markdown 或实时权威。
未知自定义消息保留其既有行为。

## 边界（Boundary）

- **公开接口：** 注册表 API（`toolRegistry`）、共享的工作区文件目标规范化器（`fileTargets`），以及各
  渲染器（包括展示型 `Markdown`——GFM + shiki，无 store/传输层；渲染是固定的，但**散文皮肤**由调用方
  通过可选的 `className` 决定——chat 使用紧凑气泡皮肤（`tr-prose-chat`），
  [[module-resource-renderers]] 的文档预览使用文档皮肤（`tr-prose-doc`）——shell 把该组件作为渲染器的
  `ResourceHost.markdown` 注入，因此 chat 绝不导入该包。一个皮肤恰好点名一个生成的 `tr-prose-*` 体系，
  随后只携带间距/度量/chrome——没有尺寸、字重、行高或字距（见 `styles/TYPOGRAPHY.md`）；调用方还可以
  用可选的 `urlTransform`、额外的 `remarkPlugins` 与 `components` **扩展**该渲染，例如文件视图的 GitHub
  alert 提示块）、视图类型（`types.ts`，包括 `ToolResultState` + `ExtUiDialogRequest`），以及
  `ChatView`（一个拆分出的 chunk，由 shell 工作台在挂载时通过 `components/preloadedLazy` 预加载，并作为
  chat 资源主体挂载；它接上 `SkillsDialog` + 头部 Skills 触发器，从 store 解析所属 `projectId`，并从
  store 选择器 `selectSkillsStale(state, workspaceId, sessionId)` 读取重载徽标——按会话且由 store 派生，
  因此能跨标签页切换的重新挂载存活；一次成功的重载调用 `markSkillsSynced` 只清除本 chat）。
  **没有 `index.ts` barrel**——chat 会拉入 **shiki**，因此按代码拆分例外，导入保持**逐文件**；注册表可
  从 `chat/toolRegistry` 导入而**不**拉入 shiki。
- **允许的依赖：** `contracts`（pi 消息/内容块类型，**仅类型**）；生命周期中立的 `prompt` 模块；
  `store` + `transport`（**仅应用集成文件**——接收 props 的渲染器绝不能去碰其中之一。今天即
  `ChatView.tsx`、**`ChatTranscript.tsx`**（与高频变动隔离的会话记录子组件——Virtuoso 列表、滚动控制、
  会话记录同步、完成确认、扩展对话框回复，以及 `ChatActions`/`InteractionStatesContext`/
  `PendingExtUiContext` 各 provider）、`chatPreferences.ts`（客户端本地持久化适配器），外加它组合的
  各 hook 与对话框：`useChatTodos.ts`、`useHistorySearch.ts`、`useModelCatalog.ts`、
  **`useChatResources.ts`**（Resources 的水合/控制/日志读取接缝）、**`useFreshResults.ts`**（按 chat 的
  "新结果"已见集合）、**`useSessionStats.ts`**（世代/修订栅栏的权威遥测读取）、
  **`useTranscriptSync.ts`**（成功压缩 + 连接世代的规范会话记录对账）、`SkillsDialog.tsx`、
  **`useProjectTrustSummary.ts`**（Skills 对话框与 `panels/ProjectTrustNotice`、New Workspace 共享的
  信任摘要读取；其纯派生与文案位于无 store 的 `projectTrust.ts`）、`TemplateEditorDialog.tsx`、
  `SubagentTranscriptDialog.tsx`、**`useModelPreferences.ts`**（钉住 / 最近 / 默认组合：store 读取加上
  每次选择器挂载都共享的 `settings.update` 写入），以及 **`modelVisibility.ts`**（隐藏模型的持久化接缝，
  由 `main.tsx` 启动一次：水合 `store.hiddenModels` 并通过稳定的客户端偏好适配器或带宿主限定的
  localStorage 写回——store 自身不触碰任何存储）。`useModelCatalog` 是共享的模型目录接缝，
  `panels/NewWorkspaceDialog` 也逐文件导入它，因此两个选择器无法漂移；激活时它**同步丢弃目录权威**
  （更早消费者设置的标志并不说明当前这个继承到的列表）并只在共享列表**为空**时读取 `model.list`——
  每次激活都读取会把一次完整的宿主 `runtime.refresh()` 挂到每次 chat 标签页切换上，而选择器的 Refresh
  行才是更新路径。它报告 **`fresh`**——直接读自 store 的 `modelsFresh`，因为目录权威属于**共享列表**
  而非某个消费者：只对被等待的强制刷新**宿主报告 `complete`** 后的已安装结果才为 true（其等待有上限，
  因此未结算的一趟仍会作答——给出可渲染的列表，而不是一个判断），并会被来自*任何*消费者的下一次
  `model.list` 安装丢弃。`model.list` 从其触发的分离刷新*之前*作答，因此它绝不是判定某模型已消失的依据）；
  `react-markdown` / `remark-gfm` / `shiki`（经 `lib/highlighter` / `lib/highlightCode`；流式围栏显示其
  最后一次高亮加上更新的文本为纯文本，直到 worker 跟上）；`marked`（仅用于 `markdownBlocks` 切分的块级
  lexer，绝不用于渲染）；`@thinkrail/ext-visualize/web`（`Markdown` 消费 `MermaidView`；`mermaid` 在
  扩展内部保持惰性）；`@thinkrail/extension-api/web`；`react-virtuoso`；`@remixicon/react`；
  `@thinkrail/ui/*`；`components/useNow`；`lib`。
- **禁止：** 值导入任何 `pi` 包；**展示型**渲染器导入 `store`/`transport`（只有上文列举的应用集成文件
  可以——请保持渲染器可复用）。
- **`ChatView`** 是主要的应用集成文件：接上传输层调用与本会话面向 shell 的 store 读取（对
  `store.sessions[sessionId]` 的粒度化逐字段选择器，加上 `recentPrompts` 与经浅比较选择器派生的计划
  速览）——连同 **`useHistorySearch.ts`**（Ctrl+R 历史召回浮层的 store/传输层边缘）、
  **`useSessionStats.ts`**（保持遥测鲜活的受守护读取）与 **`TemplateEditorDialog.tsx`**（共享的模板保存
  表单），即其余 shell 集成点。其与高频变动隔离的子组件 **`ChatTranscript`** 是第二个集成文件：它拥有
  流式运行时订阅（`store.sessions[sessionId]`），提供 `ChatActions` + `InteractionStates` 上下文，构建
  分隔线的深链（`onOpenChange` → `requestChangesView`、`onOpenSpec` → `requestSpecView`；各自接收用户
  所选的单一 路径）加上其视图切换（`onReveal` → 工具揭示意图）、来自 store 的 `specsByWorkspace` 快照
  的 `isSpec` 分类器（订阅为存储的数组——一个稳定 ref——并记忆化为匹配器，绝不在选择器内部新建 Set），
  以及收敛既有运行时的受守护 **`useTranscriptSync.ts`** 权威读取。一次**被拒绝的**发送
  （`prompt`/`steer`/`followUp`）经 store 的 `appendErrorTurn` 落到 chat 中——绝不被静默吞掉，也绝不
  获得仅限结算的 Try again 可用性；*流式*故障反而作为 pi 事件到达。

## 流式模型（Streaming model）

`store` 把 pi 事件**按会话**折成 pi 规范轮次：在途的 assistant 轮次**就是**最新的
`assistantMessageEvent.partial` 快照（每次更新替换——不是手工累积）。一条消息真正的终结点是
**`message_end`**：reducer 采用最终消息（它携带 `stopReason`，渲染器据此辨认已死的工具调用），并在那里
清除该消息的 `streaming` 标志。`agent_end` 只是一次尝试的边界（对一条调用工具的消息，它在工具运行之后
才到，但仍可能先于自动压缩/重试）；只有 `agent_settled` 才闭合这次自动运行、清除会话加载器，并通过
`insertRoundCloser`（`roundCloser.ts`，与水合共享）添加一个成功/错误标记：该标记位于任何尾部未唤醒
任何人的结果（`delivery: "appended"`；对没有该标记的会话记录，则是被停止的命令或被用户中止的 subagent）
之前。这类结果可能因投递时机而在结算之前或之后被追加；两条路径同一条规则让一个回合的收尾线与分隔线排在
它之前，因此实时行与重载行一致。一次成功且 `willRetry: true` 的溢出 `compaction_end` 会移除被取代的
出错/截断尝试，与 Pi 重建后的上下文一致。工具结果按 `toolCallId` 索引在 `toolResults` 中；工具响应的
自定义消息索引进 `toolResponses`（绝不是轮次列表——交互工具的卡片就是它们的渲染）；`subagent-completion`
自定义消息追加一个携带消息 `timestamp` 的 `subagentCompletion` 轮次（共享的 contracts 守护在实时与读取
两条路径上都收窄两者）。视图每次渲染都重新派生行（`deriveRows` 是纯的；`ChatTranscript` 记忆化它，然后
通过保存在 state 中的 `stabilizeRows` 复用未变化的行对象）——稳定的行/步骤 id 让折叠状态跨快照存活。

**只有一个实时指示器，始终如此。** pi 把一次运行拆成多条 assistant 消息，因此 reducer 在新消息开始与
最终 `agent_settled` 时扫描逐消息的 `streaming` 标志（至多一个轮次被标记）。会话在尝试级 `agent_end`
事件之间保持存活。该加载器是**单个页脚**（`StreamIndicator`：共享的 `RunningIcon` 品牌徽标，此处纯装饰，
+ 来自纯函数 `streamStatus` 派生器的阶段标签——`working` → `thinking` → `running-tool` → `writing`，
当会话记录尾部轮次是一次运行中的压缩时加上 `compacting`）——而不是逐轮次光标——因此它不会重复，并填补
发送后的间隙。在流式窗口之外（一次手动 compact，或 pi 在 `prompt()` 内部于 `agent_start` 之前运行的
pre-prompt 压缩）页脚按设计缺席——正在运行的 `CompactionNotice` 行自己携带 spinner，因此该节拍绝不是
死寂。尾部 Activity 折叠的实时动态指示是一条*状态*行（spinner，像运行中卡片的头部），不是第二个加载器。
`data-testid="stream-indicator"` + `data-phase` 使该生命周期可断言。

## 必须做对（Get right）

- 渲染器**只通过 CSS 变量 token 工具类使用主题**（无原始 hex / 内联 `style`）——正是这一点让基础组件
  能穿上任何 token 主题，也是复用的关键。
- 保持展示型组件 **props 驱动**（不绑定 store）；只有 `ChatView`/`ChatTranscript` 接线应用。这是日后
  抽取独立 `packages/chat-ui` 的接缝。
- 本规格保持**意图 + 边界 + 不变量**；逐组件的行为属于组件的 jsdoc，逐工具的细节属于
  [tools/SPEC.md](tools/SPEC.md)。
