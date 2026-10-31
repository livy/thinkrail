# feedback —— 宿主范围内的问卷邀请

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

拥有应用内问卷邀请及其所提供问卷的宿主范围生命周期。统计已接受的用户撰写消息，持久化推迟、忽略以及
已完成的问卷 id 加迭代，加载已发布的 PostHog 问卷，并在该问卷到期时给出一次定向邀请。Settings 入口
读取同一份问卷，且从不推迟。

## 边界（Boundary）

- **拥有：** 经校验的 `feedback.json`（`acceptedMessages`、`nextInvitationAt`、`dismissed`、
  `submittedSurveyId`、`submittedIteration`、`lastInvitedAt`）；固定的 10 条消息初始/重试间隔；
  一个仅存于内存的认领客户端；`recordAcceptedMessage`；`respondToInvitation`；`completeSurvey`；
  定向发布器；`survey.ts` 中的公开问卷加载器、提交解析器与 `survey sent` 载荷构建器。
- **公开接口（barrel）：** 生命周期操作、问卷来源配置、选择与刷新、提交解析、响应构建器、发布器
  安装器，以及测试重置。
- **允许的依赖：** `persistence`（仅 `dataDir`）；`contracts` 的问卷与邀请类型；Node 的文件系统/
  路径 API；`fetch`，且仅在 `survey.ts` 中用于 `GET /api/surveys/`。
- **禁止：** 导入 `host` 或分析。宿主传入分析所暴露的问卷端点，并捕获本模块构建的载荷。本模块
  不在 PostHog 中定向人群，也不打开日历。

## 必须做对（Get right）

- 缺失或格式错误的文件从 `{ acceptedMessages: 0, nextInvitationAt: 10, dismissed: false,
  submittedSurveyId: null, submittedIteration: null, lastInvitedAt: null }` 开始。缺少完成或邀请键的
  旧文件会把那些键加载为 null。每次更新都写入同目录的临时文件，并以原子重命名覆盖 `feedback.json`。
- `dismissed` 之后计数继续。`postpone` 设置 `nextInvitationAt = acceptedMessages + 10`；
  “Take the survey”也会推迟，因此关闭问卷窗口而不发送，会在再十条消息后重新邀请。`never` 永久设置
  `dismissed`：新的问卷或迭代永远不会清除它，由已退役的面谈邀请记录的忽略（其 Never 或 Book）也会
  延续。Settings 入口保持可用。完成时先在内存中存储问卷 id 与 `current_iteration` 并清除认领，然后
  写文件；写入失败会让内存中的完成继续生效，并由下一条被接受的消息持久化它；相同的 id 与迭代保持
  隐藏，而下一个迭代或新的 id 可以再次显示。
- 只有在以下条件全部满足时才投递邀请：已加载的问卷已缓存、未过期且未完成（当刷新到期时，邀请要
  等它），PostHog 的等待期已过，计数已达到 `nextInvitationAt`，邀请未被忽略，且没有客户端被认领。
  分析设置与 `--no-analytics` 不影响它。等待期是问卷的 `conditions.seenSurveyWaitPeriodInDays`
  （上限十年），从 `lastInvitedAt` 起算，即某次认领最后一次成功首次投递的问卷时钟时间（重连再
  投递不会移动它）；它为消息计数加上以天计的下限，因此被推迟的邀请既需要再多十条消息，也需要满足
  等待。PostHog 的 `schedule: "always"` 及其 web 触发器（URL、选择器、事件、设备）不被使用。未
  配置问卷来源时，不安排任何刷新。
- `survey.ts` 选择名为 `in-app-feedback` 的最新已发布问卷，最多每 15 分钟重新读取一次列表，并在
  最后一次成功读取之后一小时把其缓存副本视为消失（失败的读取与重试节流从不延长它），并向客户端
  发送一个窄化的题目 DTO（开放文本、单选与多选并含可选的 Other、评分）。当某题接受自定义答案
  （`hasOpenChoice`）时，PostHog 把该开放选项存为 `choices` 的最后一项，无论其标签是什么；它总是
  从列表中移除，只作为可选的 Other 保留，客户端把它画成一行（应用本地化的“Other”）并带其文本框。
  其他题型、PostHog 分支（#1029）与显示条件不会被画出。
- 随问卷一起解析的 PostHog 翻译留存在其基础 DTO 旁边（问卷级 `thankYouMessageHeader`；每题
  `question`、`description`、`choices`、评分的边界标签），以小写、类 BCP 47 的 tag 为键。
  `localizeSurvey` 为所请求的语言挑选一个键——精确 tag，否则其基础语言，再否则 PostHog 自己的
  匹配——逐部分应用它（无该键的部分回退到其基础语言，再回退到原文），并用该键标记 `language`；
  没有匹配时返回原问卷。当题目有开放选项时，翻译后的 `choices` 列表会像原文一样丢掉自己的最后
  一项，之后只有在其长度与原文一致且无重复时才计入，因此 `surveyResponse` 用展示的文本校验所选项，
  并把每一项按同一索引作为原语言的选项发送，外加 `$survey_language`。键入的 **Other** 文本单独
  到达（`others`），按键入内容原样发送，即使它与某个选项相同。缺少 `others` 的提交来自 v98 之前
  的客户端：其 Other 文本是内联的，不是展示选项的条目会被读作 Other 文本（每题至多一个）。
  `$survey_questions` 保留原题文本。既无端点也无成功获取时，问卷保持隐藏。
- 提交会指明用户所回答的问卷 id 与迭代，并对照一次全新读取校验（`confirmInAppSurvey` 绕过 15
  分钟缓存，在无法读取列表时拒绝），绝不对照展示副本。`surveyResponse` 要求每个非可选答案都存在，
  并逐项对照其题目校验，因此变化过或不完整的问卷永远不会被贴错标签。
