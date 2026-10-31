# 分析 —— 基础事件与受偏好控制的产品洞察

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责与边界（Responsibility and boundary）

仅宿主使用的产品分析与错误报告，由 CLI/源码/桌面共享，以无个人身份（personless）方式投递到 PostHog
EU。本模块拥有封闭的事件词汇表、目录分桶、安装身份的使用、异常清洗、投递、同意门禁以及有界关闭。
只有宿主观察特性结果并捕获事件；特性模块保持不含分析。兄弟依赖边属于 [[module-server]]。

- **公开接口：** 初始化、基础捕获、受同意范围约束的额外捕获、survey 端点与响应捕获
  （`surveyEndpoint`、`SurveyEndpoint`、`captureSurveyResponse`）、额外数据启用、错误报告
  （`reportError`、`reportFatalError`、`reportClientError`、`ErrorReport`、`ErrorSource`）、
  关闭/测试重置、事件类型、分桶辅助函数以及 `BuildKind`。
- **允许的依赖：** persistence、log、contracts 类型、pi-ai 的内建目录、Node，以及仅在 sink 内部的
  `posthog-node`（`errors.ts` 不依赖 SDK；sink 把它适配进 `before_send`）。
- **禁止：** 导入宿主/特性兄弟；在宿主之外被导入；在线上暴露安装 UUID；把丰富的特性负载复制进遥测，
  下文记录的 survey 响应除外；浏览器自动捕获或仅原生 sink。

## 事件（Events）

在带 key 的人类运行中，基础事件始终开启：`app_installed`、`app_started`、
`chat_started { provider, model, auth_method }`、`message_sent { mode, provider, auth_method }`，
以及 `provider_login { provider, method, auth_method }`。`app_installed` 只携带标准环境属性，并在
一个非 CI/非测试的二进制或桌面初始化认领共享安装标记时，在 `app_started` 之前发出一次。初始化在
认领之前构建基础 sink；一旦认领，状态在入队之前就被安装。因此 sink 构建失败会让该标记保持可用，
而稍后的投递失败既不清除它也不重试它。已有的 `{ id }` 记录在其首个符合条件的打包启动时发出。启动
指宿主启动，而不是 UI 就绪。聊天可以是空的；一个宿主在没有任何可用模型的情况下不得不创建的聊天，
把 `provider` 与 `model` 报告为 `none`、`auth_method: unknown`；这种情况由宿主自己的模型解析决定，
而不是 pi 的占位模型（否则它会被分桶为 `custom`），而一个已解析的聊天会报告所创建会话实际持有的
模型（一个启动扩展可能切换它）。“创建了聊天却没有什么可用来发送”因此仍可与一个已配置的自定义
provider 区分。发送在 `ackSend` 之后计数，排除 TODO 控制轻推，并且不证明执行成功。登录要求相关联的
成功，或既有的、已应用的 Central 连接动作。`auth_method` 是一个封闭的
`api_key | subscription | oauth | central | other | unknown` 类别，绝不是凭据或账户/套餐身份。它
描述观察到的认证路径，而不是计费权益。宿主在分发前捕获发送元数据，使用每个会话/登录保留的运行时，
并把模糊模式留作 other/unknown。Central 来源使用加载器注册元数据，而不检查不透明的 auth。

额外事件遵循 `analyticsEnabled` 偏好。它们确切属性并集与负载测试就是 schema。产品结果字段使用固定
枚举或有界分桶；下面的获客触点字段是有界的浏览器派生字符串，绝不是资源身份或任意产品负载。

| 事件 | 信号 |
| --- | --- |
| `setup_state_observed` | Provider/模型/项目就绪状态（`yes/no/unknown`）；首次当前观察及变化，而非每次轮询。 |
| `setup_action_finished` | 显式设置操作、结果与固定失败类别；自动的 Default 供给被排除。`directory_pick` 是原生文件夹选择器（成功 / 取消 / 失败，绝不是路径）；`project_create` 是应用内的新建项目引导；`project_open` 报告 projects 模块的编码错误中带类型的 `not_git` / `already_open` 原因，而每一个未类型化的失败都保持 `unknown`——原因绝不从错误文本推导。 |
| `agent_run_started` | 工作周期来源、工作区类型以及按目录分桶的 provider/模型。 |
| `agent_run_settled` | 最终结果、耗时/重试/压缩分桶；只有 `agent_settled`，绝不是尝试级的 `agent_end`。 |
| `task_completed` | 工件对账之后的非空任务组完成迁移、变更证据以及是否记录了验证。 |
| `plan_opened` | 计划向用户浮现，按 `surface`（`page`/`rail`）；一次刻意的打开动作，而非每次计划重新拉取。 |
| `plan_item_added` | 一个用户添加的计划条目（`origin:"user"`），按 `surface`（`chat`/`page`/`rail`）；用户策展计划，而非 agent 重新规划。 |
| `review_decided` | 真实的用户/agent 批准或要求改动决定，而非中止评审的清理。 |
| `review_comment_added` | 在文件/diff/文档上创建的一条评审评论，按 `author`（`user`/`agent`）与 `kind`；人类草稿流与评审人 agent 的 finding，被区分开——不是草稿编辑。 |
| `review_comment_sent` | 投递给 agent 的**每条评论**一个事件（价值时刻），带 `outdated`（`yes`/`no`）；逐评论使 added→sent→resolved 成为可计数的漏斗，而 `outdated` 是重新锚定的质量信号。是投递，不证明 agent 做了有用的动作。 |
| `review_comment_resolved` | 一条评论的终态，按 `actor`（`user`/`agent`）与 `outcome`（`resolved`/`dismissed`）；人类更新与 agent 的 `resolve_comment` 工具，而非草稿删除或 Clear。agent 的 `resolved` 是 agent 的声称，绝不证明问题已修好。 |
| `pr_action_finished` | 结果/类别；创建的 PR 与更新、推送和 compare 页交接仍相互区分；当由计划页的 PR 阶段驱动时 `source` = `plan_page`，否则 `other`。 |
| `acquisition_linked` | 一次成功的浏览器认领兑换，携带临时的旅程/桥接 id 与归一化的首/末次获客字段。 |

`survey sent` 留在两个并集之外：它是用户的显式提交，而不是使用遥测。`captureSurveyResponse` 通过
**基础**客户端对每份完成的 survey 发送一次，因此它无视额外偏好或授予，并在 `--no-analytics` 之下
发出，但从不从无 key、CI 或 `NODE_ENV=test` 运行中发出（不存在客户端）。它无个人身份
（`$process_person_profile: false`），以安装 ID 为键，携带标准环境属性加上 `$survey_id`、
`$survey_iteration`、`$survey_submission_id`、`$survey_completed`、`$survey_language`（所展示的
PostHog 翻译，当有翻译时）、`$survey_questions`（`id` 与原始问题文本）以及
`$survey_response_<question id>`——答案文本、评分或选项。它是唯一被允许包含用户所写内容的负载；
survey 窗口在第一个问题之前就说明了这一点。它返回客户端是否接受了该事件（已入队，而非已投递），
因此宿主只在那时才记录完成。不发送 `survey shown` 或 `survey dismissed`。`surveyEndpoint()` 暴露
survey 列表必须使用的 key、host（`THINKRAIL_POSTHOG_HOST` 优先）与 fetch，或在没有客户端时返回
null；宿主是两者的唯一调用方。

评审评论漏斗是宿主从既有线上/工具动作上观察的：`review_comment_added` 来自 `review.commentAdd`
处理器（人类，`author:user`）以及宿主写入 Review 标签页的每条计划评审 finding（`author:agent`，
带持久化的 `kind`，因此一次锚点回退读作 `review`）。`review_comment_sent` 对每次向 agent 的投递
逐评论触发一次：`review.sendComment` / `review.sendBatch`、投递给 worker 的计划评审修复（按钮路径
在其确认之后，工具路径在其加锁的文件+标记+记录事务提交时），以及用户的 Request-fix。逐评论使
added→sent→resolved 在一个无个人身份的人群中仍可计数（逐动作的计数分桶会让该比率不可恢复）；
`outdated` 报告已重新锚定的发送集合中的锚点漂移。它在**接受**时触发，而不是在分离的 prompt 启动时：
一次轮次前拒绝（模型不对/key 过期）会把评论回滚为草稿，因此过早捕获会夸大转化率。
`review_comment_resolved` 只在真正迁移进 `resolved`/`dismissed` 时触发——`review.commentUpdate`
（用户）与 agent 的 `resolve_comment` 接缝（`agent`/`resolved`）——因此一个来自过期客户端的幂等
重复永不计数两次。评论正文、路径、锚点文本或行号从不被复制——只有封闭的
`author`/`kind`/`actor`/`outcome`/`outdated` 枚举。

计划/TODO 漏斗搭载既有的线上动作，绝不是浏览器自动捕获：宿主从一次刻意的 `todo.list { opened }`
调用捕获 `plan_opened`（计划页打开或 TODO 侧栏打开——不是自动重新拉取），从一次成功的 `todo.add`
捕获 `plan_item_added`（`surface` 命名 TODO 侧栏的添加行、聊天的添加路径，或计划页的添加行），并从
`pr.open` 调用方为 `pr_action_finished` 盖上 `source` 判别值（当由计划页的 PR 按钮驱动时为
`plan_page`）。计划页的 Review 阶段运行经 pi-subagents 委派（[[submodule-server-host]] 的计划评审），
其异步裁决与触发界面解耦，因此 `review_decided` 今天不携带 source。`surface`/`source` 是封闭枚举；
缺省默认为 `chat`/`other`，因此一个被省略的参数永不捏造计划页归属。

获客触点 schema 是 [[submodule-website-attribution]] 的严格服务端镜像：有界归一化的 UTM
source/medium/campaign/content 字符串、封闭的 referrer 类别、时间戳与策略版本。网站与服务端两份
副本一起变更；产品包从不导入网站代码。落地页不被传输；PostHog 通过 `journey_id` 把网站事件与
`acquisition_linked` 连接起来。当额外授予有效时，持久化的仅活动首/末次字段会丰富后续的基础与额外
事件，`app_installed` 除外；被丰富的基础事件只使用当前授予的可撤销 sink，而未丰富的基础事件留在
永久的 base sink 上。因此撤销会连同额外事件一起丢弃排队/重试中的已丰富基础事件，而不停止普通基础
事件；只有 `acquisition_linked` 携带旅程/桥接 id。获客在 `last_touch` 之后 30 天过期：启动会把过期
或无效状态终结化，而每次捕获在丰富之前先检查，因此一个跨越过期时刻的进程会清空内存并原子地用终结
尝试标记替换该文件。当链接发生在该次启动期间时，首个 `app_started` 保持未丰富。

关联（correlation）是短暂的，且作用域限于一个启用偏好的时期。不重放历史，也不重建在共享启用之前
开始的工作；来自禁用时期的异步结果在重新启用之后仍被丢弃。内部/未知的工作永不让用户激活膨胀。一次
正常停止或 agent 声明的任务/验证状态不是价值、正确性或一个通过的测试的证明。到达顺序不是执行顺序。

## 错误报告（Error reports）

PostHog Error Tracking 的 `$exception` 事件属于基础层：在与基础事件相同的门禁之下始终开启（带 key
的构建，非 CI/测试），不受 `analyticsEnabled`、`--no-analytics` 或 `THINKRAIL_NO_ANALYTICS` 影响，
且永不被获客丰富。封闭的 `source`：`host_crash`（crashLog 的未捕获/未处理，级别 `fatal`，`scope` =
崩溃类型与根本机制 `onuncaughtexception`/`onunhandledrejection`，用 `captureExceptionImmediate` 发送，
并在退出前最多等待 1 秒——一个可达的 PostHog 在其中的几分之一秒内就会应答，而每多一秒都在让一个已
出故障的宿主继续运行）、`host_logged`（一个携带错误的主机 `log.warn/error`，带其 `scope`）、
`request`（每个失败的 WS 请求，CodedErrors 也算——可重访；PostHog 自身排除预期的 4xx 结果——只有
对已注册方法才有 `request_method`）、`ui`（`client.reportError`；`scope` = kind[:boundary]，
render = handled）以及 `desktop`（已处理的桌面主进程错误，带其 scope）。错误 `code` 只在匹配
`^[A-Z][A-Z0-9_]+$` 时通过。

隐私保证是允许清单，而不是拒绝清单。捕获使用 posthog-node 5.46.1 的 `captureException`，经由一个
`PostHog` 子类，其被覆写的（protected）`createErrorPropertiesBuilder` 保留 SDK 的强制转换器与
node 解析器，但**不含任何帧修饰器**：默认的那些会从磁盘读取每帧上下 ±7 行源码、把路径变为相对于
`cwd`（仍然是 `../../Users/<name>/…`）并推导模块名。`@posthog/core` 被钉到 posthog-node 自己的
core，仅为该 builder 导入。sink 的（无文档的）类型化 `before_send` 随后从零**重建**每个
`$exception`：≤ 5 个异常，`type` 仅标识符、一个分类器 `value`、
`mechanism.{handled,synthetic,type}`（根的 `handled` 来自报告，因为 SDK 的捕获 API 不接受提示、
总是默认它为 true；cause 保持 handled），以及只携带 `platform`、仅标识符的 `function`、归一化的
`filename`、`lineno`、`colno`、`in_app` 与一个 UUID 形状的 `chunk_id` 的帧。上下文行、`vars`、
`abs_path`、`module` 或任何后续 SDK 新增的字段都永远无法通过。应用根（源码时为仓库根，打包时为
bundle 目录，桌面上经 `appRoot` 为 `Resources/app`）内的绝对文件名变为根相对；loopback/`views:`
形式的 Web URL 变为 `web/…`/`views/…`；`node:` 与 `native` 保持原样；其余一切变成 `<external>`，
函数为 `?`——用户扩展与项目文件不留下任何名字。一个相对文件名只可能来自 CLI 二进制内嵌的 source
map（Bun 会相对其编译目录重映射帧），并且只经启动器的 `remapBase` 解析为仓库相对的 JS/TS 路径，
绝不通过 `cwd`。

Source map 让上报的帧可读，而不运送超出 CLI 自己那份 map 的任何东西（[[module-ci-release]] 拥有
上传）。PostHog 只按 `chunk_id` 解析 Node 帧、按 `chunk_id` 解析浏览器帧（或按我们没有的公开 URL）：
SDK 的 core builder 从注入 bundle（桌面运行时）写入的 `_posthogChunkIds` 注册表复制 chunk id，而 UI
报告携带它们自己的 `chunkIds`（URL → id），宿主把它作为内部 `$thinkrail_chunk_ids` 属性传入，清洗器
消费它且从不转发。CLI 二进制内嵌其 map，因此它的帧到达时已经解析好。

**永不发送消息文本。** 错误消息经常携带第三方文字——git stderr（一个 pre-push 钩子的输出会到达
`pr.open` 抛出的消息）、带私有主机名的 DNS/TLS 诊断、provider 与扩展错误——而没有任何脱敏能让任意
文本不含个人数据。因此根异常的 `value` 是一个只由通过校验的允许清单字段构建的分类器——先是
`source`，然后是 `request_method` 或 `scope`，然后是 `code`（例如
`request pr.open PUSH_AUTH_FAILED`、`ui render:chat`）——而 cause 携带空 `value`；PostHog 按类型与
解析后的帧分组，因此分类器只是给问题命名。消息还藏在每个 V8/Bun 栈头里，而一条多行消息（git
stderr、某个测试 runner 的 traceback）可能包含形状像帧的行，SDK 的解析器会把这些行变成“in-app”
帧；因此 SDK 只会看到 `reportableError(error)`——类加上每个异常与 cause 的栈，其中的消息被切掉
（无法定位消息的栈被丢弃，非 `Error` 值不携带栈）——而浏览器在 `client.reportError` 之前从自己的
栈中切掉消息，该调用完全不携带消息字段。一个线上级测试钉住 SDK 实际请求体的确切属性键集合，包括
SDK 在 `before_send` 之后添加的 `$lib`/`$lib_version`/`$is_server`/`$geoip_disable`，因此一个新增
字段的 SDK 升级会让 CI 失败，而不是泄漏。

`enableExceptionAutocapture` 保持关闭：它自己的 `uncaughtException` 监听器把退出推迟到我们的监听器
上，因此它的异步捕获会丢失。SDK 的限流器只作用于自动捕获，因此服务对相同
（source、scope、method、type、code、抛出点——栈顶帧）的报告去重 60 秒，并限制每次启动 100 条，其中
任何单一来源最多 50 条，使一个嘈杂来源无法饿死其余来源；致命崩溃报告绕过这两者。浏览器栈以原始形式
到达，并从 WebKit/Gecko 的 `fn@url:line:col` 重写成 node 解析器可读的 V8 行；浏览器从不持有 vendor
客户端。宿主分析初始化之前的错误（桌面/CLI 预启动）不上报；桌面主进程 bundle 不携带 source map。

## 偏好与投递（Preference and delivery）

`analyticsEnabled` 是额外数据偏好与宿主投递门禁；`analyticsConsentConfirmed` 记录首次运行对话框的
完成情况，而不是一个持续的投递门禁。宿主初始化会让未确认的配置保持关闭，直到该对话框挂载。对话框
默认开启的挂载播种会在未确认的情况下持久化 `analyticsEnabled: true`，并在应用的设置更新成功时立即
启用投递，早于用户按下 Done。切换开关会立即应用其偏好。Done 带着确认持久化当前偏好；该确认只控制
提示的生命周期。初始化之后，只有显式包含 `analyticsEnabled` 的已应用设置更新才改变授予；无关的
全配置广播保留它。设置变更把偏好与确认一起写入。窗口行为属于 [[submodule-web-panels]]。已确认的
后续启动既不播种也不重新打开。

只有带项目 key 标记的构建会上报（见数据边界），外加一个用 `THINKRAIL_DEV_POSTHOG=1` 与
`THINKRAIL_POSTHOG_PROJECT_KEY` 选择加入的源码开发宿主。其他无 key 的源码/开发构建、CI 与
`NODE_ENV=test` 不创建 vendor 客户端，也不认领安装标记。`--no-analytics` /
`THINKRAIL_NO_ANALYTICS` 只抑制额外事件而不改变同意；它们不抑制一次显式发送的 survey。宿主侧分析
是跨各启动器的唯一环境策略读取者。浏览器归因还被进一步限制为：二进制/桌面的人类运行、偏好与确认
均为真、注入了启动器 opener，且没有先前的尝试。源码、CI/测试、逐次运行抑制、显式关闭以及 CLI
`--no-open` 都不消耗该尝试。对话框未确认的播种开启可以启用普通额外事件，但无法启动归因；最终确认
的更新可以。已保存的确认开启只在 CLI 打开其正常本地 UI 之后，或桌面窗口首次 `dom-ready` 之后启动；
服务端启动与经过的时间并不意味着启动器就绪。

宿主生成一个随机的 32 字节 verifier，发送其 SHA-256 challenge，要求严格的协议响应，恰好一次调用
返回的同源相对 claim URL opener 而不等待它，并以 10 秒间隔最多轮询 54 次状态，随后执行一次 redeem。
每个请求都有中止超时，整个认领有九分钟截止时间，低于网站认领的有效期。当前同意世代拥有一个
AbortController；撤销与关闭会中止 fetch 和正文读取，并在后续捕获之前移除丰富。失败与已完成的尝试
都是终结性的，永不自动重试；重新启用只恢复一条仍有效的已存储活动记录。一次通过校验的兑换会在其世代
仍然活跃时激活内存并发出 `acquisition_linked`，即使尽力而为的活动持久化失败也是如此。

额外授权的撤销会在传输层边界丢弃排队/重试中的请求，而不停止基础事件；一个已发出的请求无法撤回。
被撤销的队列永不复活。捕获/启动永不阻塞产品流程，也不向调用方抛出；优雅关闭会等待一次幂等的两秒
SDK 排空。

## 数据边界（Data boundary）

稳定的安装 UUID 与可选的 `appInstalled: true` 字段只留在服务端；
[[submodule-server-persistence]] 拥有独占的 ID 创建与至多一次的跨进程安装认领。认领之后的崩溃可能
丢失该事件，但绝不重复它。独立的 `attribution.json` 存储要么一个终结性的浏览器尝试标记，要么通过
校验的首/末次活动上下文，绝不存 claim ID、verifier、challenge、claim URL、旅程/桥接 ID、IP 或
user agent。过期或无效的上下文变为终结；已存储的活动数据在偏好关/开之间存活，而活跃的丰富不存活。
其他状态没有单实例协调。计数描述的是安装，不是人。每个事件携带 `app_version`、`channel`、`os`、
`arch`、`build` 加上其封闭属性。只有内建的 provider/模型名按原样通过；自定义值变成 `custom`，并
保留既有显式的 `jbcentral` 登录名。不收集任何聊天/文件内容、路径/名字、资源 ID、凭据、错误消息、
token/成本计数或录屏；错误报告只携带上面允许清单中的形状。

sink 使用启动器提供的公钥、EU 端点、禁用的 GeoIP 丰富与 `$process_person_profile: false`；
key/端点/fetch 注入支持测试与自托管。仓库不携带 key：发布 CI 把它盖章进 [[module-shared]] 的版本
接缝，因此源码、开发与本地构建的工件（`channel=dev`、`0.0.0-*`）不上报。2026-10-06 的一次突增——
在没有 `CI=1` 的未盖章开发桌面/二进制测试装置运行中产生 275 个 `app_installed`——促成了这一点；
仅靠 `CI`/测试静音依赖每个测试装置记得设置它们。源码开发宿主是唯一的例外：
`THINKRAIL_DEV_POSTHOG=1` 与 `THINKRAIL_POSTHOG_PROJECT_KEY` 一起把该 key 传入 `bootHost` 并创建
一个真实客户端，它会发送基础事件与 survey 答案。一次普通的开发运行保持开关未设置并维持无 key。CLI
与桌面源码运行仍使用空的盖章 key。`phc_` key 仍是可从已发布二进制中提取的公开摄取 key，因此这阻止
的是意外污染，而不是刻意滥用。无个人身份处理并不移除 UUID 关联；vendor 的 IP 丢弃/保留策略是另一
回事。自动化/schema、同意撤销、迁移、宿主触发与打包 loopback 投递测试钉住这些边界。
