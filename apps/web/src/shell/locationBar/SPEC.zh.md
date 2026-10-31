# shell/locationBar —— 顶栏中带标题说明的位置分段

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

ThinkRail 标识之后那条常驻的“我在哪里／处于什么状态”条带：一行带标题说明的分段 ——
**Project · Workspace · Branch · from base · Remote · Pull request** —— 其值就是控件，用于切换项目或工作区、
对工作区执行动作、读取或改指分支，以及执行分支的**下一步交付动作**（publish、push、打开 PR、sync、
修复检查、标记 ready、merge、settle）。它渲染 store 与 `workspace.branchStatus`（较旧的宿主为
`workspace.openReview`）已经汇报的内容，并通过 panels 所属的 ship runner 执行动作；它不拥有任何宿主状态。

## 边界（Boundary）

- **拥有：** 分段语法（标题说明压在 20px 值行之上，分段之间仅以空白分隔）、每个值都使用的
  pill／chip／split-button 类、项目切换菜单、工作区菜单（动作 + 同级切换器 + New workspace）、
  分支卡片（复制、based-on、compare-to 选择器、remote 与 review 行）、Remote 与 Pull request
  分段，以及响应式的丢弃顺序。
- **公开接口（`index.ts`）：**
  - `LocationBar`
- **允许的依赖：** `store`（projects、workspaces、会话状态、diff base 选择器、toasts）、`transport`
  （`workspace.setDiffBase`）、`lib`（`copyText`、`platformShortcutLabel`）、`components/RunningIcon`、
  `@thinkrail/ui/*`、`@remixicon/react`、`@thinkrail/locales`（UI 字符串）、contracts 类型，以及以下
  `panels` 文件（逐文件）：`workspaceActions`（open-in／rename／reveal／remove／`editor.list`／
  `workspace.list` 的唯一实现，与 `ProjectTree` 共享）、`RemoveWorkspaceDialog`、`BranchPicker` +
  `branches`、`useOpenProject`、`useBranchStatus`（`openReviewLabel`、`resolveBranchReview`）、
  `shipActions`（推导逻辑），以及 `useShipRunner`（带其对话框的动作执行器，与 Plan 面板共享）。
  以工作区 id 为 key 的 `ShipStrip` 拥有该执行器：一个 compose 草稿能在断开连接、分段卸载之后继续存在，
  而切到另一个工作区会连同草稿、它的重试以及任何待处理的预览一起丢弃，因此一次提交永远不可能把某个
  工作区的草稿发给另一个工作区（与 rename、Remove 相同的绑定规则）。
- **禁止：** server/shared/pi；layout engine、intents 或 frame 状态；被 `Shell` 以外的任何东西导入；
  对 `panels/workspaceActions` 已提供的工作区动作、或 `panels/useShipRunner` 已提供的 ship 动作
  （push、open PR、sync）再写第二份实现。

## 语法（Grammar）

- 这条带和每个分段都拉伸到头部的内容盒（39px：40px 行减去其底部边框），绝不使用固定的
  `h-topbar-row` —— 39px 盒中放一个 40px 子元素会在半像素上居中，在 1x 屏幕上使文字发虚并压住边框。
  分段把一行 14px 的标题说明（12px 句首大写形式的 `tr-text-metadata`，颜色 `text-subtle`，用它自己的
  15px 行距：`leading-none` 会让字形框溢出那行截断行）居中压在 20px 值行之上，标题说明行上方与值悬停
  填充下方各留 2.5px。分段不带细分隔线：`pl-8` 加上 pill 自身的内边距（文字到文字约 42px，
  悬停填充之间 8px）就是分隔，因为通高竖线读起来像表格，且永远无法与下方标签栏的分隔线对齐。
  标题说明片段（`· from <base>`、`· settled`）使用标题说明自身的字形 —— 绝不在标题说明内部再引入
  第二种字体。
- 值是 pill（`pillClass`：20px、`window-no-drag`，悬停 `control-bg-hovered`，打开时
  `control-bg-selected`；chevron 静止时隐藏，在悬停、键盘聚焦或打开期间出现）或状态 chip
  （`chipClass`：20px、rounded-full、`tr-text-emphasis`、反馈色的 `-subtle` 填充配实心反馈文字；
  可交互 chip 悬停到 `-muted` 档），或分段自己的**分体按钮**（20px、rounded-sm、带边框：下一个动作
  用 `primary-subtle` —— Sync 用 `feedback-warning-subtle` —— 外加该分段其他动作的 ▾ 菜单；没有
  下一个动作时只显示 ▾）。每个 pill 值都是同一种样式、同一种静止颜色，`tr-title-section` 配
  `text-muted` —— 即 Projects 树中工作区名称的颜色 —— 仅在悬停或打开期间提亮为 `text-default`：
  混杂的静止灰度读起来像意外，所以条带中唯一的静止色调层级是标题说明（`text-subtle`）对值。
  项目首字是中性方块（`control-bg-selected`／`text-muted`），这样最不重要的字形不是唯一带颜色的那个。
  等宽字体只留在分支卡片内部，那里 ref 才是内容，不出现在条带中。
- 标题说明仍属于头部拖拽区域；每个 pill、chip、链接以及内联重命名输入框都用 `window-no-drag`
  退出该区域（有单元测试），这就是对 shell 的“按钮只放在 `topbar-actions`”规则仅针对本条件带的
  刻意放宽。
- 随窗口变窄的丢弃顺序：项目分段在 `sm` 以下、分支分段在 `md` 以下、**无动作**的 Remote 分段在
  `md` 以下、既无 review 又无下一个动作的 Pull request 分段在 `sm` 以下丢弃。工作区分段、带 review
  或带下一个动作的 Pull request 分段，以及携带动作的 Remote 分段永不隐藏；在 `md` 以下它们的分体
  按钮折叠为图标。工作区保留 64px 最小值，因此被原生内边距挤压的条带会裁掉尾部分段，而不会裁掉
  工作区名（e2e `topbar-chrome` 在带内边距的手机宽度下保持 `scope-name` 可见）。行内分支先收缩
  （`shrink-[4]`），然后是项目（`shrink-[3]`），再是工作区，各自受 `max-w` 限制；Remote 与
  Pull request 分段不收缩。面对没有 `workspace.branchStatus` 的宿主，旧规则成立：Remote 与 review
  chip 只在存在 review 时渲染，并在 `md`／`sm` 以下隐藏。

## 行为（Behaviour）

- **Project** 列出每个已打开的项目（当前项目带勾选）、*Project home*（身在其中时禁用）以及
  *Add project…*（共享的 `useOpenProject` 选择器 + 其对话框）。选择某个项目会用 `reveal` 选中它并
  加载其工作区列表，即与 Projects 树相同的转换。
- **Workspace** 在 *Switch to* 之上显示活动工作区的动作（Open in、Rename、Copy path、Reveal、
  侧栏的**标记为已读／未读**开关、**Settle** 或 **Keep active** —— 视哪种适用，经共享的
  `workspaceActions` 执行 —— 以及经共享确认对话框执行的 Remove；Default 不可移除且永不 settle）。
  *Switch to* 列出该项目的**活跃**同级工作区，最近活动者在前 —— 与侧栏稳定的“最新创建”顺序不同，
  因为切换器服务的是“回到我刚才所在的” —— 有 agent 工作时显示 `RunningIcon`；随后是一个
  **`Settled · N ›`** 子菜单，列出搁架行及其原因 chip —— 与侧栏渲染相同的
  `selectWorkspacePartition` 和共享 `useNow` 时钟，因此当空闲边界越过时，菜单、活动的 `· settled`
  标题说明和侧栏永不出现分歧；在 settled 协议之下它改为保留宿主原始的遗留顺序，并省略一切搁架
  相关能力。首次打开时若树尚未加载，列表会加载）以及带 `Mod+N` 标签的 *New workspace*，它打开由
  shell 拥有的对话框。当活动工作区处于 settled 状态时，标题说明读作 **`Workspace · settled`**
  （即 `Branch · from base` 已在使用的标题说明片段语法，采用警告色调），因此你打开的一个被搁置的
  工作区无需横幅也能自我表明。Rename 用别处已用的无 chrome 内联输入框替换 pill，并运行在树的共享
  `useInlineRename` 控制器上：Enter/blur 提交，Escape 取消，未改动或留空永不发起请求，而在 socket 的
  rename 能力未知时做出的提交会停留在编辑器里待定，直到一个具备该能力的 welcome 将其恢复。派发
  绑定到编辑开始时所在的工作区，且由于本分段是单一持久实例（树的各行按工作区分别 key），在编辑器
  打开时激活另一个工作区会放弃该次编辑 —— 包括待定的离线提交 —— 而不是让稍后的 welcome 去重命名
  新激活的工作区。Remove 确认遵循同一规则：它依据选择 Remove 时快照的工作区渲染，并在其下方活动
  工作区发生变化时（浏览器 Back、远端移除回退）自行关闭，因此确认永远不会删除它所指向之外的
  工作区。在 Project home 时 pill 读作 "Project home"，只提供切换器和创建入口。
- **Branch · from base** 打开 git 卡片，把焦点移到卡片自身（而非它的第一个按钮，那会弹出复制
  tooltip），这样一次 Tab 就能到达各动作：分支（复制 → 成功 toast）、*Based on*（对用户自有的
  工作区隐藏，其标题说明只是 "Branch"）、*Compare to*（Changes 面板的 `BranchPicker`，绑定到
  `workspace.setDiffBase`，因此两个控件永不出现分歧）、*Remote*（`n commits to push ·
  n commits behind origin`、"In sync with origin"、"Not published" 或 "No origin remote" —— 在
  v87 之前的宿主上只在与 review 并存时出现），以及 —— 存在 review 时 —— 带链接的 PR/MR 行。
- **Remote**（v87 的 `workspace.branchStatus`）为每个工作区渲染，包括其 base 上的 Default。其值读作
  *No remote*（没有 `origin`）、*Unreachable*（分离 origin 的 push 仓库无法读取 —— 不提供动作）、
  *Not published*（origin 缺少该分支）、*In sync*，或 ↑N 警告／↓N 信息 chip。它的下一个动作是
  `shipActions` 的 remote 分支：**Publish**（未发布且领先 base 有提交）、**Push N**（领先且未落后）
  或 **Sync…**（落后：带 Plan 面板的 shell 惰性 `git pull --rebase origin <branch>` 规则的分叉
  对话框，绝不用 force 命令；当 `pushesElsewhere` 时完全没有 pull 命令，因为拉取 origin 会集成那个
  fetch 仓库，而计数描述的是 push 仓库 —— `shipActions` 的 `syncCommand` 是唯一规则）。Publish/Push
  用条带显示的分支调用 `workspace.push`（若检出此后已移动，宿主什么都不 push）；当该分支**就是**它
  的 base（`main` 上的 Default）时，runner 先确认并指明共享分支，然后才发送 `confirmBase`。顶栏 push
  永不改写 PR 描述 —— Plan 面板的 *Push updates* 仍会从计划刷新它。Publish/Push 动作替换该值；
  Sync 位于 chip 旁边。
- **Pull request**（v87）为每个工作区渲染。没有 review 时，若分支就是其 base，其值为空，否则为
  *No PR*／*No MR*，存在主动作时让位给主动作；它的分体按钮携带 `shipActions` 的 review 分支：
  **Open PR**（领先 base 有提交、origin 为 GitHub —— 共享的 compose 对话框；没有可用的 `gh` 时走
  宿主的 compare／gh-setup 结果，与 Plan 面板一致）、**Create MR**（GitLab：同一个 compose 对话框；
  宿主 push 并打开 GitLab 预填的新 MR 页面）、▾ 中的 **Open draft PR**，以及非活动的
  **Merge into <base> locally** 占位项（"coming soon"，属另一项任务）。已合并或已关闭的 review 通过
  共享的 `workspaceActions` 提供 **Settle**（▾ Keep active／Remove）；在 v88 宿主上 ▾ 还含
  **Reopen PR on GitHub**（已关闭）和 **Delete remote branch…**（仍已发布；确认中指明
  `origin/<branch>` 并承诺本地分支保留）。
- **一个打开的 GitHub PR**（v88 的 `details`）在草稿期间以中性色调读作 `Draft #N`，其 chip 的尾部
  字形改为检查结论（✕ 失败、spinner 进行中、✓ 通过 —— tooltip 与无障碍名称携带计数和失败项名称），
  而非链接图标。它的动作分支依次为：**Fix with agent**（检查失败）、**Ready for review**（草稿）、
  **Merge PR…**（`mergeState` 为 clean／unstable／has_hooks，且分支已知与 origin 一致）—— 第一个是
  主动作，其余放在 ▾ 中。当分支领先或落后 origin 时，该 PR 没有主动作：Remote 分段的 Push／Sync
  才是下一步，此时 merging 会跳过尚未推送的提交。一个就是自身 base 的分支不获得任何 PR 动作（也不
  获得 Reopen）：它的 review 是某个同名分支的 PR，例如 fork 的，绝不是它自己的。是否可 merge 由
  merge 状态决定，绝不由批准决定，因此没有必需评审的仓库只要 CI 允许即可 merge。分支卡片增加一行
  **Checks** 和一行 **Review**（已批准／谁请求了修改，或等待评审）—— 评审决定是信息，绝不是动作门禁。
- **CI 轮询**属于顶栏的 `useBranchStatus`（Shell 的 hook 选择启用；Plan 面板中同一 key 的实例不启用，
  因此不会有重复轮询）：当打开的 review 的检查处于待定且窗口可见时，每 20 s 做一次跳过 origin fetch
  的全新 review 读取。ship 动作的刷新（push、Open PR、Ready、…）也无论在哪个界面执行，都欠该 key
  三次**发现**轮询，落在共享的 `branchStatusState` 中：GitHub 在 push 返回之后才为新 head 创建检查套件，
  所以动作之后的首次读取通常还看不到待定检查。
- 在 v87 之前的宿主上，仅 review 规则成立：**Remote** 只在 `unpushedCommits`（警告 chip）或
  `behindCommits`（信息 chip）非零时渲染，并打开同一张卡片。两个计数都随 `OpenBranchReview` 到达，
  所以 Remote（以及卡片的 *Remote* 行）只能与 review 并存 —— 没有 review 的分支不显示远端状态，
  而不是显示 "in sync"。**Pull request** 是 `openReviewLabel` chip —— review 打开时为 success，
  一旦 `OpenBranchReview.state` 如此声明便为读作 *Merged #N* 的 **info** 和读作 *Closed #N* 的
  **neutral**（v78 之前的宿主省略 `state`，chip 保持打开形态）—— provider 报告了 URL 时是一个链接。
  Remote chip 只为打开的 review 渲染；对已合并分支的分叉信息是噪声。在 v78 上，宿主保留的
  `workspace.review` 快照对 review 的存在／身份／状态具有权威性；匹配的新鲜 `workspace.openReview`
  应答只覆盖 URL 与分叉细节。因此一次已合并／已清除的 push 不会被过期的 hook 缓存掩盖，而 `gh` 的
  抖动也永不会清空搁架仍在据以行动的 chip。没有匹配的实时细节时，PR 行仍在，但 Remote 摘要隐藏 ——
  缺失的仅查询计数意味着未知，绝不是 "In sync with origin."。v78 之前的宿主只用查询结果，忽略因回滚
  而陈旧的快照字段。在 v87 上，同样的解析运行于 `workspace.branchStatus.review` 之上，而远端计数来自
  `BranchStatus` 本身，因此不再依赖 review。
