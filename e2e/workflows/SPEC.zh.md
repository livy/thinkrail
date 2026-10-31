# e2e/workflows —— 无头工作流测试装置 + 场景

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

无头工作流测试套件：让一个**真实的进程内 pi agent** 跑过各工作流 skill（[[module-thinkrail-workflow]]），
**没有浏览器也没有 HTTP 宿主**，并用严格的裁决模型判定通过/失败。按需通过
`bun run test:workflows` 运行（自带 Playwright 配置 `playwright.workflows.config.ts`——没有
`webServer`；有意保留本地 PI 鉴权设置，而不是浏览器 agent 套件的 real-Central 模式）。其隔离的
`PI_CODING_AGENT_DIR` + 固定模型由 `env.ts` **按 worker 进程**克隆（带 pid 后缀，鉴权 + 设置被复制
进去）：worker 绝不能共享一个 agent 目录——一个将死的 worker 的会话释放与下一个 worker 的新生会话
在共享的 `sessions/` 树中重叠，曾把一次失败放大成之后每个活跃会话测试的 ENOENT 级联。`env.ts` 还
设置隔离的 `HOME`/vendor-home 集合，使可移植的 skill 发现从不读取开发者个人的库。需要 pi 鉴权并
消耗真实 provider token——绝不属于 `bun run test`、pre-commit、CI 门禁，或浏览器 `e2e`/`e2e:agent`
脚本（主配置对此目录做了 `testIgnore`）。

**裁决模型（这里一切所服务的不变量）：LLM 永不判定*通过*。**
- *有约束力的：* 对捕获的事件日志与工作区做确定性检查 + 信号——skill 的加载就是对它 `SKILL.md` 的
  `read`；产物在磁盘上被断言；被禁止的工具/路径不存在。
- *仅输入/诊断：* persona 应答器与用户模拟器（廉价 LLM）生成用户侧输入；watchdog 的“在轨”评估只做
  带诊断的中止（随后测试因未被触发的停止信号而失败）；judge 以**顾问性质**给每个场景的评分表打分
  （告警 + 运行日志，绝不构成测试失败）。

## 测试装置（Harness）（`harness/`，barrel `index.ts` —— 规格文件唯一的导入面）

新增一个工作流测试 = 一次 `defineScenario` 调用：`{ name, skill, workspace, preset?, entry?, user?,
dialog?, stopWhen?, forbid?, watchdog?, expect, judge?, record? }`。

- `session` —— 基于生产用的 `@thinkrail/server/agent` barrel 的生命周期（`createSession` 等）；
  在 pi 运行时初始化之前设置 `PI_CODING_AGENT_DIR`、`HOME` 以及受支持的 vendor-home 覆盖
  （`env.ts`，最先导入）；来自停止信号的中止是预期结果——但仅限*被请求的*中止（`promptTurn` 只在
  调用方确认某个信号/预算要求它时，才吞掉形如中止的错误；未被请求的“aborted”是 provider/网络崩溃，
  会重新抛出）。绝不用 `setModel`。
- `events` —— `EventLog`：一个进程范围的 `setSessionPublisher` 订阅者无条件缓冲每个会话的事件
  （捕获不会漏掉早期事件）；查询 + `waitFor` + 紧凑的转录渲染；delta 追加，`partialResult` 替换。
- `workspace` —— e2e 数据目录下的一次性 git cwd（`empty` / `code-only` / `specced` / 自定义种子）。
- `dialog` —— 通过**生产用的 `respondToTool` 桥**回答 `ask_user_question` 轮次，走一条阶梯：脚本
  匹配器 → persona（经校验；格式错误则走下一级）→ 确定性回退（`skip` / `pickRecommended`）。未编写
  脚本的面谈永远不会挂住一次运行，*会抛错的*脚本（场景作者代码）也一样：它直接降级到确定性回退，
  并把错误记录在该轮次上——绝无未处理的 rejection。投递失败也算轮次：`respondToTool` 可能在事后
  拒绝（慢的 persona 答案被用户模拟器的下一条消息取代），这被记录在该轮次上，而不是抛出——必须赢下
  这场竞争的场景会为它们的轮次编写脚本（脚本这一级在 `tool_execution_start` 同步作答）。一个实时
  答案会解除阻塞的工具并继续其当前运行；一个经重启修复的 ack 答案会触发新的轮次。dialog 跟踪每次
  投递并暴露 `settle()`——当所有由此产生的工作都结束时 resolve（包括级联轮次，失控的轮次由预算
  绊线约束）——场景在裁决之前 await 它：检查绝不与某个答案所启动的工作竞争。
- `presets` —— 流程中入场：产物预设（工作流原生——task-spec 是工作流的脊梁）与转录夹具（经服务端
  的 `setSessionManagerFactory` 接缝调用 `SessionManager.open`；一个夹具 = `session.jsonl` +
  `workspace/` 快照，其中记录的 cwd 在临时副本中被改写）。夹具由录制模式诞生：
  `THINKRAIL_WORKFLOW_RECORD=1` + 场景的 `record` 名 → `fixtures/<name>/`。**夹具中的 markdown 在
  落盘时被掩码**——快照中的每个 `*.md` 都存为 `*.md.test`（录制时掩码，回放到一次性 cwd 时解除
  掩码），使夹具规格（`id: acme-root`、…）永不出现在宿主仓库自己的规格图谱中：规格发现会 glob
  `*.md`，只有一份硬编码的目录忽略清单，因此未掩码的夹具会浮现到产品的 Specs 栏以及每一次
  `spec_grep` 中。*被否决的替代方案：* 扩展 `pi-spec-graph` 的忽略清单——忽略 `e2e/` 会吞掉这份
  真实的规格，基于名字的“fixtures”忽略会为一个仓库本地的测试关切而改变一个可移植包的语义，而掩码
  还能挡住其他每一个 `*.md` 扫描器，不只是规格发现。
- `signals` —— 日志上的谓词；`stopWhen`（通过 → 中止，主要的成本控制）与 `forbid`（失败 → 立即
  中止）。
- `watchdog` —— 确定性的预算绊线（轮次 / 工具调用 / 墙钟时间，每个场景默认开启）+ 可选的轮次间
  “在轨”评估。
- `userSim` —— 模拟的人（与 `dialog` 共享场景的 persona 简报）：组织开场与后续聊天消息，直到某个
  信号、预算或 `DONE`。
- `checks` / `judge` / `runlog` —— 有约束力的裁决；顾问性的评分表打分器；每次运行追加一行 JSON 到
  **被 git 忽略**的 `e2e/.workflow-runs.jsonl`（在被清空的 e2e 数据目录之外——按决策作为本地证据；
  第 14 条规则的中止理由见 [[submodule-workflow-skills]]）。在给出裁决之前就抛错的运行会记录
  `crashed` + 一个失败的确定性裁决——日志绝不会声称一次其检查从未运行的通过
  （`THINKRAIL_WORKFLOW_RUNLOG` 重定向该文件，使单元测试可以断言记录而不污染本地证据）。
- `scenario` —— 编排（seed → presets → start → attach → 对话循环 → dialog settle → 裁决 → 运行
  记录），在 `finally` 中做清理——也覆盖最早的步骤：转录夹具工厂替换与 `startSession` 位于受保护
  区域内，因此在那里抛错仍会恢复进程范围的接缝；`workflowTest` 把场景注册为 Playwright 测试
  （标记 `@agent`）。

依赖方向是单向的：`scenario` → 一切；`dialog`/`signals`/`userSim`/`watchdog` → `events`/`session`；
`checks`/`judge` 读取冻结的日志；只有 `session`/`presets`/`dialog`/`judge`/`userSim`/`watchdog`
触碰服务端 barrel（会话、答案桥、`completeOnce`）。

## 边界（Boundary）

- **允许的依赖：** `@thinkrail/server/agent`（为本测试装置新增的子路径导出——记录在
  [[module-server]]；barrel 的行为见 [[submodule-server-agent]]）、
  `@earendil-works/pi-coding-agent`（`SessionManager`）、`@thinkrail/contracts`（类型）、
  `@playwright/test`、`../fixtures/paths`、Node。
- **禁止：** `@thinkrail/server` 的根导出（会求值仅 Bun 可用的 `host`）、`apps/*`、任何浏览器/页面
  夹具、模型选择（`setModel` 会在运行中途把一个默认值持久化进共享的隔离 agent 目录）。
- **可被 Node 加载的服务端 barrel。** 测试装置运行在 Playwright 的 Node 进程中，因此
  `@thinkrail/server/agent` 背后的模块图必须在没有 Bun 专有解析的情况下加载：不能有目录或无
  扩展名的深层导入，不能对 `node_modules` 下的 TypeScript 做裸 `require`。违反这一点会让每个活跃
  场景在创建会话时、在任何 skill 运行之前就失败——该套件曾以这种方式静默地无法运行，直到 2026-10。

## 套件（Suites）

- `harness.unit.spec.ts` —— 纯逻辑部分（解析器、谓词、检查、预算绊线）针对合成日志；不需要活跃
  agent，也不需要鉴权。
- `routing.live.spec.ts` —— 路由套件（第 2 片）：两个路由器分类表的每条路由一个场景——符合工作流
  条件的根路由经由常开规则到达；dispatcher 通过应用精确的 `/skill:` 种子强制加载（该种子会注入
  skill 内容，因此不期望读取它自己的 SKILL.md）。brainstorming 路由从其默认入口的两侧各钉一个：
  一个小的、明确指定的行为变更，以及一个 bug 报告（它必须在任何源码编辑之前到达 bug 分支）。通过 =
  被路由的 worker 的 skill 加载信号；direct-work 行绕过 worker skill 并产生有依据的结果。直接编辑的
  断言钉住最终的工作树，而不是报告里偶然出现的词汇或 agent 选了哪个合法的变更工具。
- `smoke.live.spec.ts` —— 证明基础设施的活跃场景：一次流程中的 `brainstorming` 往返，必须在不重新
  进入根路由器的情况下恢复（产物预设 + persona 答案 + 用户模拟器 + 磁盘上的决策），以及转录续接
  （重新打开的夹具能回忆流程中的状态）。
- `importing.live.spec.ts` —— 首个第 3 片 worker 覆盖：在文档丰富的夹具上端到端跑
  `importing-a-codebase`——文档采纳提议（候选被提议并**就地**采纳，已完成的计划文件从不被提议也不
  被触碰，已填充的架构槽位不被重写）——另加一个无候选的回归变体（只有 README/CHANGELOG/TODO 存在
  → 提议必须不触发，普通导入路径不变）。夹具文档从场景代码内联播种（落盘时没有 `*.md` 夹具文件
  ——上述掩码顾虑不会出现）。
- 后续（仅场景定义）：其余 worker 流程的端到端（第 3 片：`starting-a-new-project` /
  `brainstorming` 完整运行）——跟踪于 [[module-thinkrail-workflow]] 的 § Testing。
