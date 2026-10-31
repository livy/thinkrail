# 按钮系统迁移 —— 旧 → 新

> 中文译文，原文见 [`buttons-migration.md`](./buttons-migration.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

由本次改动中按区域划分的迁移报告生成。映射规则：先取外观上最接近的变体，再取最接近的高度
（旧 `default` 32px → `l`，`sm` 28px → `m`，`icon` → `IconButton m`；裸按钮 ≈18–22px → `s`，
≈24px → `m`，≈28–32px → `l`；同高时取较小尺寸）。例外情况单独列出。

已移除的旧名称：变体 `default` → `primary`、`soft` → `soft-primary`；尺寸 `sm` / `icon` / `default` /
`lg` → `s` / `m` / `l` + `IconButton`。

## shell（41 处）

| 位置 | 之前 | 之后 | 说明 |
| --- | --- | --- | --- |
| `auth/LoginDialog.tsx:130` | `<a> buttonVariants({ className: 'flex-1' })（旧 default 填充）+ icon size-16` | `<a> buttonVariants primary l (+ flex-1)` | 不在 packet 的 rawButtons 列表中（buttonVariantsCalls: 1，审计 SHELL-058）；移除了 icon size 类 |
| `auth/LoginDialog.tsx:140` | `Button outline default`，Check/Copy icon 元素 size-16 | `Button outline l` | 通过 `iconLeft={copied ? Check : Copy}` 保留图标切换 |
| `auth/LoginDialog.tsx:250` | `Button default default` | `Button primary l` |  |
| `auth/LoginDialog.tsx:281` | `Button outline default` | `Button outline l` |  |
| `auth/LoginDialog.tsx:285` | `Button outline default` | `Button outline l` |  |
| `shell/Shell.tsx:178` | IconTooltip 中的裸 `size-28` 透明 muted 图标按钮 | `IconButton ghost m` | label 'Settings'（aria），tooltip 保留快捷键文本 |
| `shell/WorkspaceChatHistory.tsx:36` | `PopoverTrigger` `w-panel-header-row` `border-l` muted 图标触发器，chatStarting 期间手动 Loader2 spinner，`IconTooltip` wrapTrigger | `PopoverTrigger asChild > IconButton ghost m`（wrapTrigger，chatStarting 期间图标切换为旋转的 `StartingIcon`，self-center） | 不在 packet 的 rawButtons 列表中（审计 SHELL-038，Radix trigger）。与之前一样，聊天启动过程中触发器仍可点击（无 `loading`）。移除了 border-l 分隔线。 |
| `shell/WorkspaceChatHistory.tsx:190` | 裸 `cn(menuItemClass,'shrink-0 px-4 text-text-muted')` 图标按钮（约 26px） | `IconButton ghost m` | aria 名称 'Rename <title>'，tooltip 'Rename chat' |
| `shell/locationBar/BranchSegment.tsx:153` | 裸 `rowActionClass`（`size-24` muted ghost 图标按钮） | `IconButton ghost m` | `rowActionClass` 常量已删除 |
| `shell/locationBar/BranchSegment.tsx:217` | `<a> scope-review-open` 配 `rowActionClass` | `<a> buttonVariants ghost m shape icon (+ shrink-0)` | 不在 packet 列表中；与已删除的 `rowActionClass` 共用，保留在其 `IconTooltip` 内 |
| `shell/layout/workbenchCenter.tsx:100` | 裸 `w-panel-header-row`（28 方形）muted 图标按钮，禁用外观 | `IconButton ghost m`（self-center） | tooltip 保留不可用的原因；无 wrapTrigger（旧代码本来就没有） |
| `shell/layout/workbenchCenter.tsx:115` | 裸 `w-panel-header-row` muted 图标按钮 | `IconButton ghost m`（self-center） |  |
| `shell/layout/AuxiliaryPaneHeader.tsx:71` | 裸 `w-panel-header-row` muted 图标按钮（Hide pane） | `IconButton ghost m`（self-center） | `aria-controls` / `aria-expanded` 保留 |
| `shell/layout/workbenchTabs.tsx:674` | 裸 `p-2` + 14px 图标的（约 18px）标签页关闭按钮，hover 显示 | `IconButton ghost s` | `tabIndex -1` 与 `opacity-0 group-hover:opacity-100 focus:opacity-100` 的显示逻辑保留；现在根据 label 额外获得 tooltip 'Close <name>' |
| `shell/layout/workbenchTabs.tsx:287` | `PopoverTrigger` `w-panel-header-row` `border-l` 的 'Search open tabs'，在 `IconTooltip` wrapTrigger 中 | `PopoverTrigger asChild > IconButton ghost m`（wrapTrigger，self-center） | 不在 packet 列表中（审计 SHELL-101）；移除了 border-l 分隔线 |
| `shell/layout/workbenchBottom.tsx:64` | `DropdownMenuTrigger` `w-panel-header-row` `border-l` 图标触发器，带原生 title | `DropdownMenuTrigger asChild > IconButton ghost m`（wrapTrigger，self-center） | 不在 packet 列表中（审计 SHELL-111）；原生 title 变为 IconButton 的 tooltip；添加了 wrapTrigger，因为 Dropdown 触发器与新的 tooltip 触发器共用同一个元素（与代码库中其他「触发器+tooltip」位置相同的模式） |
| `shell/layout/workbenchBottom.tsx:207` | 裸 `border-border-default bg-container-elevated-bg px-12 py-4 tr-text-ui`（约 26px） | `Button outline m` |  |
| `shell/layout/workbenchSide.tsx:184` | `DropdownMenuTrigger` `w-panel-header-row` `border-l` 的 '+' 触发器，在 `IconTooltip` wrapTrigger 中 | `DropdownMenuTrigger asChild > IconButton ghost m`（wrapTrigger，self-center） | 不在 packet 列表中（审计 SHELL-112）；移除了 border-l 分隔线 |
| `shell/layout/workbenchSide.tsx:258` | 裸 `w-panel-header-row` muted 图标按钮，禁用外观 | `IconButton ghost m`（self-center） | tooltip 保留不可用的原因 |
| `shell/LayoutSettings.tsx:226` | 裸 `p-2` + 14 图标（约 18px）muted，仅文字 hover | `IconButton ghost s` | 长篇解释性文本作为 tooltip 传入；aria label 'About the default layout' |
| `shell/LayoutSettings.tsx:268` | 裸 `p-4` + 14 图标（约 22px）透明 text-primary（保存名称） | `IconButton soft-primary s` | 绿色 ghost；22px 同高 20/24 → s |
| `shell/LayoutSettings.tsx:277` | 裸 `p-4` muted 图标（取消重命名） | `IconButton ghost s` |  |
| `shell/LayoutSettings.tsx:287` | 裸 `p-4` muted 图标（重命名） | `IconButton ghost s` |  |
| `shell/LayoutSettings.tsx:322` | `Button outline sm` | `Button outline m` |  |
| `shell/LayoutSettings.tsx:344` | `Button default sm` | `Button primary m` |  |
| `shell/LayoutSettings.tsx:418` | `Button ghost sm` | `Button ghost m` |  |
| `shell/LayoutSettings.tsx:426` | `Button default sm`（type=submit） | `Button primary m` | type submit + aria-label 保留 |
| `shell/LayoutSettings.tsx:446` | `Button outline sm` | `Button outline m` |  |
| `shell/WorkspaceWorkbench.tsx:219` | 裸 `border-border-default transparent px-8 py-4 tr-text-ui`（Retry） | `Button outline m` |  |
| `shell/WorkspaceWorkbench.tsx:707` | 裸 bordered elevated `px-12 py-4`，chatStarting 期间手动切换 Loader2 | `Button outline m, iconLeft MessageSquarePlus, loading={chatStarting}` | label 切换 'Starting chat…'/'New chat'、disabled、`data-starting` 保留；`mt-4` 保留 |
| `shell/WorkspaceWorkbench.tsx:740` | 裸 `w-panel-header-row` `border-l` muted 图标按钮 | `IconButton ghost m`（self-center） | 移除了 border-l 分隔线 |
| `updates/UpdateSettings.tsx:253` | `Button outline default` + Refresh size-14 | `Button outline l` | 'Checking…' label + disabled 保留（原本没有 spinner，所以不用 loading） |
| `updates/UpdateSettings.tsx:264` | `Button outline default` | `Button outline l` |  |
| `updates/UpdateSettings.tsx:270` | `Button default default` | `Button primary l` |  |
| `updates/UpdateSettings.tsx:276` | `Button default default` | `Button primary l` |  |
| `updates/UpdateSettings.tsx:284` | `Button default/outline`（spread 的条件变体）default | `Button primary\|outline l` | `variant={hostFailed ? 'outline' : 'primary'}`，iconLeft 条件传入 |
| `updates/UpdateReadyButton.tsx:85` | `Button ghost sm` + `className text-primary` | `Button soft-primary m` | 绿色 ghost；移除了 className 覆盖；label span 保留 `hidden md:inline` |
| `components/ErrorBoundary.tsx:92` | 裸 bordered elevated `px-12 py-4`（Reload page） | `Button outline m` |  |
| `components/ErrorBoundary.tsx:101` | 裸 bordered elevated `px-12 py-4`（Try again） | `Button outline m` |  |
| `notifications/NotificationPermissionPrompt.tsx:58` | `Button outline default` | `Button outline l` |  |
| `notifications/NotificationPermissionPrompt.tsx:61` | `Button default default` | `Button primary l` |  |

## projects（31 处）

| 位置 | 之前 | 之后 | 说明 |
| --- | --- | --- | --- |
| `panels/ConfirmDialog.tsx:62` | `Button outline`（默认 32） | `Button outline l` | 源码未变；本就有效 |
| `panels/ConfirmDialog.tsx:65` | `Button destructive\|default`（32） | `Button destructive\|primary l` | 'default' -> 'primary' |
| `panels/ConfirmPopover.tsx:59` | `Button outline sm` | `Button outline m` |  |
| `panels/ConfirmPopover.tsx:62` | `Button destructive\|default sm` | `Button destructive\|primary m` |  |
| `panels/NoticeDialog.tsx:43` | `Button default`（32） | `Button primary l` | 依赖组件默认值；源码未变 |
| `panels/OpenProjectPathDialog.tsx:77` | `Button outline`（32） | `Button outline l` | 源码未变 |
| `panels/OpenProjectPathDialog.tsx:80` | `Button default submit`（32） | `Button primary l` | 源码未变；type=submit 保留 |
| `panels/Toaster.tsx:63` | `ToastAction asChild` 内的 `Button outline sm` | `Button outline m` |  |
| `panels/NewWorkspaceDialog.tsx:554` | `Button outline sm`（Trust project） | `Button outline m` |  |
| `panels/NewWorkspaceDialog.tsx:674` | 裸 `<button>` `h-32 bg-control-primary-bg px-12 tr-text-action`，creating 期间手动 Loader2 spinner | `Button primary l loading={creating}` | 移除手动 spinner；保留 'Creating…'/'Starting…' 的 label 切换（按组件行为，label 在居中的 spinner 下仍然不可见）；`↵` kbd 提示 span 作为内容保留；`disabled={!submitEnabled}` 保留；className shrink-0 |
| `panels/ProviderWarningBanner.tsx:35` | `Button outline sm shrink-0` | `Button outline m` |  |
| `panels/ProjectTrustNotice.tsx:43` | `Button ghost sm`（Stop trusting） | `Button ghost m` | 中性 ghost，不是红字 |
| `panels/ProjectTrustNotice.tsx:112` | `Button outline sm` | `Button outline m` |  |
| `panels/BranchPicker.tsx:86` | 裸图标 `<button>` `size-24` muted ghost，title+aria-label，refreshing 期间 RefreshCw 旋转 | `IconButton ghost m label='Refresh branches' loading={refreshing}` | 旋转图标 -> loading spinner；refreshing 期间该按钮现在会阻止重复激活（旧实现允许重复点击） |
| `panels/ExistingWorktreeDialog.tsx:123` | `Button outline sm` + 内联 RefreshCw | `Button outline m iconLeft=RefreshCw` |  |
| `panels/ExistingWorktreeDialog.tsx:199` | `Button outline`（32） | `Button outline l` | 源码未变 |
| `panels/PrComposeDialog.tsx:69` | `Button default`（32），busy 期间图标换成 Loader2 | `Button primary l iconLeft=GitPullRequestArrow loading={busy}` | 移除手动 spinner |
| `panels/PrSetupDialog.tsx:41` | `Button ghost icon`（28），Copy/Check（成功色）切换，aria-label | `IconButton ghost m label='Copy command'` | `icon={copied ? CopiedIcon : Copy}`；`CopiedIcon` 是模块级包装，在勾选字形上保留 `text-feedback-success`；现在也会获得共享 tooltip |
| `panels/PrSetupDialog.tsx:49` | `Button outline sm` + Play，title tooltip | `Button outline m iconLeft=Play tooltip='Run this command in a workspace terminal — answer its prompts there'` | 原生 title → `tooltip`（Button/IconButton 都不接受 `title`） |
| `panels/PrSetupDialog.tsx:205` | `<a> buttonVariants({variant:'outline'})` + svg size-14 | `<a> buttonVariants({variant:'outline', size:'l'})` | 移除 svg size 类（由组件决定 svg 尺寸），添加 aria-hidden |
| `panels/PrSetupDialog.tsx:211` | `Button default`（32）+ RefreshCw 子元素 | `Button primary l iconLeft=RefreshCw` |  |
| `panels/ProjectTree.tsx:274` | `AddProjectMenu` `DropdownMenuTrigger asChild` 内的 `Button ghost icon size-panel-row`（22），带 aria-label | `IconButton ghost s label='Add project'` | 保留 ref；获得共享 tooltip；菜单打开 -> 通过 data-state=open 呈选中态 |
| `panels/ProjectTree.tsx:594` | `IconTooltip` + `Button ghost icon size-panel-row shrink-0` | `IconButton ghost s label=CREATE_WORKSPACE_LABEL className=shrink-0` |  |
| `panels/ProjectTree.tsx:810` | `IconTooltip` 'Keep active' + 裸 `<button>` `HOVER_CONTROL_CLASS`（size-20） | `IconButton ghost s label='Keep {name} active' tooltip='Keep active'` | `HOVER_CONTROL_CLASS` 缩减为仅布局 + hover 显示 |
| `panels/ProjectTree.tsx:821` | `IconTooltip` + 裸 `<button>` `HOVER_CONTROL_CLASS`，带 `disabled:opacity-0 disabled:pointer-events-none` | `IconButton ghost s label='Settle {name}' tooltip={blocked ? 'Finish or read the result first' : 'Settle'} wrapTrigger` | 去掉 `disabled:opacity-0`（决策 2）：被阻塞的 Settle 现在保持可见但呈禁用外观；添加 wrapTrigger，使 'Finish or read the result first' 提示在禁用按钮上也可见 |
| `panels/ProjectTree.tsx:899` | 带 `HOVER_CONTROL_CLASS` 的 `DropdownMenuTrigger`（渲染 `<button>`），kebab | `DropdownMenuTrigger asChild > IconButton ghost s label='Actions for {name}'` | 在 packet 中未列为裸 `<button>`，但与 `HOVER_CONTROL_CLASS` 共用（审计 PROJ-096）；已迁移，从而使该共享类只保留布局；获得 tooltip；打开 -> 选中；e2e 的 hover 门控透明度检查仍满足 |
| `panels/InterviewPromptDialog.tsx:65` | `Button ghost`（32） | `Button ghost l` | 源码未变 |
| `panels/InterviewPromptDialog.tsx:74` | `Button outline`（32），带 ref | `Button outline l` | 源码未变 |
| `panels/InterviewPromptDialog.tsx:90` | `<a> buttonVariants()` + pending 时的 'pointer-events-none opacity-50' | `<a> buttonVariants({variant:'primary', size:'l'})` + pending 时的 data-disabled | 调用点的禁用外观替换为组件的 data-disabled 外观（同样含 pointer-events-none）；aria-disabled 保留；移除 cn 导入 |
| `panels/AnalyticsConsentDialog.tsx:51` | `Button default`（32） | `Button primary l` | 源码未变；保留 'Saving…' label 逻辑 |
| `panels/HeaderIconButton.tsx:37` | 裸图标 `<button>` `size-24`，active = `bg-container-elevated-bg`，`IconTooltip` | `IconButton ghost m, selected={active}, aria-pressed={active}`，data-active 保留 | API 保留（testid/label/active/disabled/onClick/children-as-icon-element），因为所有调用点（ChangesReviewPane、DiffPane、ChangesFileSection）都在本 packet 之外；children 通过一个稳定的、由 context 支撑的图标组件渲染（span display:contents、aria-hidden），因此不会重新挂载；新增可选的 `icon?: ButtonIcon` prop，供调用点日后迁移 |

## settings（44 处）

| 位置 | 之前 | 之后 | 说明 |
| --- | --- | --- | --- |
| `panels/ProvidersSettings.tsx:141` | `Button ghost sm`，refreshing 期间 RefreshCw 图标旋转，`disabled={refreshing}` | `Button ghost m iconLeft=RefreshCw loading={refreshing}` | animate-spin 的图标切换由 loading 取代；aria-label 保留；移除未使用的 cn 导入 |
| `panels/ProvidersSettings.tsx:214` | `Button outline sm`（Show N more） | `Button outline m` |  |
| `panels/ProvidersSettings.tsx:297` | `Button soft sm` + LogOut（Sign out） | `Button soft-primary m iconLeft=LogOut` | 范例 |
| `panels/ProvidersSettings.tsx:342` | `Button soft sm` + LogIn（Sign in） | `Button soft-primary m iconLeft=LogIn` | 范例 |
| `panels/ProvidersSettings.tsx:355` | `Button outline sm` + KeyRound（API key） | `Button outline m iconLeft=KeyRound` | 范例 |
| `panels/ModelsSettings.tsx:117` | `Button ghost sm`（Retry） | `Button ghost m` |  |
| `panels/GithubSettings.tsx:37` | `Button outline sm`，refreshing 期间 RefreshCw 旋转，`disabled={refreshing}` | `Button outline m iconLeft=RefreshCw loading={refreshing}` | 移除未使用的 cn 导入 |
| `panels/FeedbackSettings.tsx:17` | `<a> buttonVariants()`（旧 default/default）+ size-14 图标 | `<a> buttonVariants primary l, self-start` | 移除图标 size 类（由组件决定 svg 尺寸） |
| `panels/FeedbackSettings.tsx` `feedback-join-discord` | `<a> buttonVariants({ variant: 'outline' })` + size-14 图标（在 main 上新增，#945） | `<a> buttonVariants outline l, self-start` | 移除图标 size 类，添加 aria-hidden，与它上面的面试链接一致 |
| `panels/CustomThemesSettings.tsx` Copy / `custom-themes-open-folder` | `Button outline sm` + Copy / FolderOpen size-14 子元素（在 main 上新增，#937） | `Button outline m iconLeft=Copy / FolderOpen` | 与 LanguageSettings 的文件夹动作是同一对 |
| `panels/NotificationsSettings.tsx:109` | `Button outline sm`（Allow in browser） | `Button outline m` |  |
| `panels/ModelContextSettings.tsx:279` | `Button ghost sm`（Retry） | `Button ghost m` | `disabled={pending}` 保留（原本没有 spinner） |
| `panels/AppearanceSettings.tsx:99` | `Button outline default` + `className h-32 justify-between`，ChevronDown size-16 子元素（`DropdownMenuTrigger asChild`） | `Button outline l iconRight=ChevronDown` | 类似 select 的触发器；移除 h-32/justify-between（在 SettingsRow 的控件槽中按内容尺寸布局，所以 justify-between 本无效果）；truncate 的 label span 保留 |
| `panels/McpSettings.tsx:303` | `Button ghost sm`（次级行内动作） | `Button ghost m` |  |
| `panels/McpSettings.tsx:315` | `Button (review\|sign-in ? default : outline) sm` | `Button (review\|sign-in ? primary : outline) m` |  |
| `panels/McpSettings.tsx:335` | `IconTooltip > DropdownMenuTrigger asChild > Button ghost icon` + aria-label（More for …） | `DropdownMenuTrigger asChild > IconButton ghost m icon=MoreVertical label='More for …'` |  |
| `panels/McpSettings.tsx:439` | `IconTooltip > Button ghost icon className size-20`，Close size-12（撤销 always-allowed 工具） | `IconButton ghost s icon=Close label='Ask again before … in this project'` | `disabled={inert}` 保留 |
| `panels/McpSettings.tsx:618` | `Button outline sm` + Plus（空状态预设 Add） | `Button outline m iconLeft=Plus` |  |
| `panels/McpSettings.tsx:632` | `Button default sm` + Plus（Add a server） | `Button primary m iconLeft=Plus` |  |
| `panels/McpSettings.tsx:642` | `Button outline sm`（Import mcpServers JSON） | `Button outline m` |  |
| `panels/McpSettings.tsx:905` | `Button default sm` + Plus（Add server） | `Button primary m iconLeft=Plus` |  |
| `panels/McpSettings.tsx:945` | `Button outline sm`（Retry） | `Button outline m` |  |
| `panels/TemplatesSettings.tsx:198` | `Button soft sm` + Plus（New，项目级） | `Button soft-primary m iconLeft=Plus` |  |
| `panels/TemplatesSettings.tsx:281` | `Button soft sm` + Plus（New，全局级） | `Button soft-primary m iconLeft=Plus` |  |
| `panels/TemplatesSettings.tsx:298` | `Button outline sm` + MagicWand（Add starter templates） | `Button outline m iconLeft=MagicWand` | `disabled={adding}` 保留 |
| `panels/TemplatesSettings.tsx:392` | `IconTooltip >` 裸 button `size-24` muted ghost（Open as file） | `IconButton ghost m icon=FileText label='Open as file'` |  |
| `panels/TemplatesSettings.tsx:399` | `IconTooltip >` 裸 button `size-24` muted ghost（Edit） | `IconButton ghost m icon=Pencil label='Edit'` |  |
| `panels/JetBrainsAiCard.tsx:238` | `Button default sm disabled` + 旋转 Loader2 + '{action}…'（jetbrains-configuring） | `Button primary m iconLeft=Loader2 loading` | 必须传 iconLeft，这样 spinner 占据图标槽位且 '{action}…' label 保持可见（没有图标的 loading 会隐藏 label）；渲染 loading 外观而不是禁用外观 |
| `panels/JetBrainsAiCard.tsx:251` | `Button default sm` + RefreshCw（Retry） | `Button primary m iconLeft=RefreshCw` |  |
| `panels/JetBrainsAiCard.tsx:267` | `Button ghost sm` + RefreshCw（Recheck，头部） | `Button ghost m iconLeft=RefreshCw` |  |
| `panels/JetBrainsAiCard.tsx:325` | `ActionButton: Button (disconnect ? outline : default) sm` + 条件 LogOut/Wrench/Play 图标 | `Button (disconnect ? outline : primary) m`，条件 iconLeft | 仅在存在图标时才传 iconLeft（exactOptionalPropertyTypes） |
| `panels/JetBrainsAiCard.tsx:356` | `Button ghost sm self-start` + RefreshCw（Recheck，安装主体） | `Button ghost m iconLeft=RefreshCw self-start` |  |
| `panels/JetBrainsAiCard.tsx:444` | `SignInButton: Button soft sm self-start`，signingIn 期间 ExternalLink 换成旋转 Loader2，`disabled={signingIn}` | `Button soft-primary m iconLeft=ExternalLink loading={signingIn} self-start` |  |
| `panels/JetBrainsAiCard.tsx:564` | 裸 button `size-24` muted ghost，aria-label 'Copy: cmd'，title 'Copy'，Copy/Check（text-feedback-success）切换 | `IconButton ghost m icon={copied ? CopiedCheck : Copy} label='Copy: cmd' tooltip='Copy' shrink-0` | 原生 title 变为 tooltip；小的本地 `CopiedCheck` 组件保留绿色成功对勾颜色 |
| `panels/McpServerDialogs.tsx:246` | `Button ghost icon` aria-label Remove + Trash2 | `IconButton ghost m icon=Trash2 label='Remove'` | 现在也会显示 'Remove' tooltip（IconButton 的 label 就是 tooltip） |
| `panels/McpServerDialogs.tsx:255` | `Button ghost sm self-start` + Plus（Add header/variable） | `Button ghost m iconLeft=Plus self-start` |  |
| `panels/McpServerDialogs.tsx:487` | `Button outline sm` + Plus（预设 Add/Added） | `Button outline m iconLeft=Plus` |  |
| `panels/McpServerDialogs.tsx:525` | `Button outline sm`（Use recipe） | `Button outline m` |  |
| `panels/McpServerDialogs.tsx:760,763,773,777,781,920,923,933,936,1062,1065,1156,1224` | 对话框页脚默认尺寸（32）的 `Button default/outline` | `Button primary l / outline l`（通过组件默认值） | 无需改源码：旧的默认变体/尺寸与新默认的 primary/l 一一对应；`disabled={saving…}` 保留（原本没有 spinner） |
| `panels/SubagentSettings.tsx SourceTag 'Custom ↺'` | 裸 accent-text 的 link-action 标签 | `Button soft-primary s iconRight=RiArrowGoBackLine` | 绿色 → soft-primary |
| `panels/EnvironmentSettings.tsx:196` | `Button ghost icon`，Eye/EyeOff 切换，aria-label Show/Hide value | `IconButton ghost m icon={shown ? EyeOff : Eye} label=Show/Hide value` | 在本次改动分支之后添加到 main（#912）；现在也会把 label 作为 tooltip 显示 |
| `panels/EnvironmentSettings.tsx:207` | `Button ghost icon` aria-label Remove + Trash2 | `IconButton ghost m icon=Trash2 label='Remove'` | 在 main 上添加（#912） |
| `panels/EnvironmentSettings.tsx:232` | `Button ghost sm self-start` + Plus（Add variable） | `Button ghost m iconLeft=Plus self-start` | 在 main 上添加（#912） |

## chat（29 处）

| 位置 | 之前 | 之后 | 说明 |
| --- | --- | --- | --- |
| `chat/QueueStrip.tsx:47` | 裸图标按钮 `size-20 rounded-xs` 中性 hover（Pencil size-12） | `IconButton ghost s tooltip='Edit'` | aria-label -> label（完整的排队文本）；tooltip 用简短的 'Edit'，因此绝不重复整条消息；data-testid 保留；shrink-0 保留 |
| `chat/QueueStrip.tsx:56` | 裸图标按钮 `size-20 rounded-xs` 中性 hover（X size-12） | `IconButton ghost s tooltip='Remove'` | 同上（edit） |
| `chat/CopyButton.tsx:20` | 裸图标按钮 `h-lh w-24` muted，hover 底色，hover 显示，原生 title | `IconButton ghost s` | `h-lh` ≈ 20px → s（m 会溢出消息行）；图标通过 icon prop 在 Copy/Check 之间切换；title -> tooltip（label）；data-copied 保留；hover 显示的 `opacity-0 group-hover/focus-visible/data-[copied]:opacity-100` 保留；调用方的布局类（`absolute z-10` 定位）仍会传入 |
| `chat/SkillsButton.tsx:18` | 裸文本按钮 `px-8 py-2 tr-text-metadata` muted，中性 hover（约 19px），BookOpen size-14 | `Button ghost s iconLeft=BookOpen` | 保留 `IconTooltip` 包装（带 tooltip 的文本按钮）、data-testid/data-stale、stale 徽标点子元素；className prop 仍会合并（NewWorkspaceDialog 传入 ml-auto） |
| `chat/ChatTranscript.tsx:698` | 裸文本按钮 `border-border-default bg-container-elevated-bg px-8 py-4 tr-text-metadata`（约 25px）+ shadow-md，浮动 | `Button outline m iconLeft=ArrowUp\|ArrowDown` | 定位保留（`-translate-x-1/2 absolute bottom-12 left-1/2`）；`shadow-[var(--shadow-md)]` 作为非布局属性被去掉——该浮动按钮不再有浮起阴影 |
| `chat/SkillsDialog.tsx:252` | `Button outline sm`（Reload，RefreshCw 子元素） | `Button outline m iconLeft=RefreshCw` | 保留 `IconTooltip` wrapTrigger（流式输出期间禁用） |
| `chat/SkillsDialog.tsx:293` | `Button outline sm`（Trust project） | `Button outline m` |  |
| `chat/SkillsDialog.tsx:410` | `Button default sm`（Enable，ShieldCheck 子元素） | `Button primary m iconLeft=ShieldCheck` |  |
| `chat/ToolResultImages.tsx:30` | 裸图标按钮 `border-border-default bg-container-elevated-bg p-4`，图标 14（约 24px），原生 title 'Full screen' | `IconButton outline m` | label = 旧 aria-label，tooltip = 旧 title；仍是 `DialogTrigger asChild` 的子元素；`absolute top-4 right-4` 保留 |
| `chat/ModelEffortPicker.tsx:428` | 裸图标开关 `size-16`，置顶时 `text-feedback-warning`，否则 subtle + hover 显示 | `IconButton ghost s selected={favorite}` | aria-pressed 保留 + selected；Star 的填充/线条通过 icon prop 切换；`mt-2 shrink-0` + hover 显示（仅在未置顶时）保留；置顶时通过 `tone="warning"` 保持黄色，外加选中外观（在 cmdk 行内上浮）与实心星；尺寸从 16 变为 20px |
| `chat/ModelEffortPicker.tsx:658` | 裸 outline 外观按钮 `control-border/control-bg px-8 py-4 tr-text-metadata`（约 25px），refreshing 期间 RefreshCw animate-spin | `Button outline m iconLeft=RefreshCw loading={refreshing}` | 手动旋转由 loading 取代；保留 'Updating…' label 切换、`disabled={refreshing}`、data-refreshing |
| `chat/promptImages.tsx:206` | 裸图标按钮（移除图片），无内边距，X size-12，muted->default | `IconButton ghost s` | 现在有 tooltip 'Remove image'；FileChip 尾部槽内的 20px 方块使图片 chip 高约 30px 而非约 25px（见决策） |
| `chat/TemplateEditorDialog.tsx:219` | `Button outline` 默认尺寸 | `Button outline l` | 无需改代码（默认尺寸就是 l） |
| `chat/TemplateEditorDialog.tsx:226` | `Button default` 默认尺寸 | `Button primary l` | 无需改代码（默认值） |
| `chat/ModelSelector.tsx:130` | 裸图标开关 `size-16`，置顶时 `text-feedback-warning`，否则 subtle + hover 显示 | `IconButton ghost s selected={isPinned}` | 与 ModelEffortPicker 的收藏开关相同（置顶时 `tone="warning"`） |
| `chat/ModelSelector.tsx:248` | 裸 outline 外观按钮 `px-8 py-4 tr-text-metadata` + shadow-sm，RefreshCw animate-spin | `Button outline m iconLeft=RefreshCw loading={refreshing}` | pointer-events-auto 保留（在 pointer-events-none 的渐变页脚内是必需的）；shadow-sm 去掉 |
| `chat/ChatView.tsx:855` | `Button ghost` 默认尺寸 | `Button ghost l` | 无需改代码 |
| `chat/ChatView.tsx:858` | `Button default` 默认尺寸 | `Button primary l` | 无需改代码 |
| `chat/ExtUiPrompt.tsx:157` | `Button outline sm` + `OPTION_BUTTON`（`h-auto min-h-28 justify-start whitespace-normal break-words py-4 text-left`） | `Button outline m className=max-w-full` | `OPTION_BUTTON` 缩减为仅布局的 'max-w-full' |
| `chat/ExtUiPrompt.tsx:174` | `Button outline sm` + `OPTION_BUTTON`（Cancel） | `Button outline m className=max-w-full` |  |
| `chat/ExtUiPrompt.tsx:183` | `Button default sm` + `OPTION_BUTTON`（OK/Submit） | `Button primary m className=max-w-full` |  |
| `chat/turns.tsx:584` | `Button outline sm`（Try again，RotateCw 子元素） | `Button outline m iconLeft=RotateCw` |  |
| `chat/Composer.tsx:661` | `IconTooltip` 'Search history' 下的 `Button ghost icon rounded-full` | `IconButton ghost m label='Search history'` | `IconTooltip` 包装并入 IconButton；aria-label 同文本；圆形 -> 24px 方形 |
| `chat/Composer.tsx:673` | 裸 `STOP_PILL`：`h-28 rounded-full` ghost 胶囊，`tr-text-action` muted，hover 底色，StopFill size-12；`@max-md` 下为 28 方形图标 | `Button ghost m iconLeft=StopFill tooltip='Stop' className=shrink-0 @max-md:w-24 @max-md:px-0` | 移除 `STOP_PILL` 常量；保留 aria-label='Stop'；`WIDE_ONLY` label span 作为子元素保留；紧凑（`@max-md`）模式是历史记录 IconButton 旁 24px 的纯图标方块（守护规则带原因地覆盖） |
| `chat/HistoryOverlay.tsx:168` | 裸图标按钮 `p-4` 图标 14（约 22px）muted，hover elevated-bg，hover 显示，`IconTooltip` | `IconButton ghost s` | label='Save as template'，tooltip='Save as template (shortcut)'；透明度显示 + isSelected 的 opacity-100 保留 |
| `chat/HistoryOverlay.tsx:194` | 裸图标按钮 `p-4` 图标 14（约 22px）muted，hover 显示，`IconTooltip` '⇧⏎ go to chat' | `IconButton ghost s` | label='Go to chat'，tooltip='⇧⏎ go to chat' |
| `chat/TodoList.tsx:124` | 裸图标按钮 `size-24` muted 中性 hover，`IconTooltip` | `IconButton ghost m` | label='Open the plan page'，tooltip 保留；去掉无效的 `focus-visible:opacity-100`（该按钮从未被隐藏） |
| `chat/TodoList.tsx:427` | 裸图标按钮 `p-2` 图标 14（约 18px）muted，hover control-bg-selected，hover 显示，`IconTooltip` | `IconButton ghost s` | `opacity-0 group-hover:opacity-100 focus:opacity-100` 保留；移除未使用的 IconTooltip 导入 |
| `chat/ExtUiPrompt.tsx options` | `Button outline sm`（+ 换行覆盖） | `Button outline m`；select 选项保留一个有文档记录的换行覆盖 | 守护规则 OVERRIDES 条目 |

## tools（11 处）

| 位置 | 之前 | 之后 | 说明 |
| --- | --- | --- | --- |
| `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:667` | 裸 `<button>` ask-skip：透明、text-text-muted、hover 时 text-default、`px-4 tr-text-ui`（约 18px） | `Button ghost l` | 通过 sizeException 取尺寸 l（与 Next/Submit 组成页脚按钮对）；`disabled={!canRespond}` 保留；className shrink-0 |
| `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:678` | 裸 `<button>` ask-continue：`bg-control-primary-bg px-12 py-8 tr-text-action`（约 34px） | `Button primary l` | label 'Next →' 保留；className shrink-0 |
| `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:688` | 裸 `<button>` ask-submit：`bg-control-primary-bg px-12 py-8` + 禁用外观（约 34px） | `Button primary l` | `disabled={!canSubmit}` 保留；去掉 `disabled:cursor-not-allowed` 外观（禁用由组件负责） |
| `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:1012` | 裸 `<button>` ask-note-toggle：透明 text-text-muted `tr-text-metadata`，hover 时 text-default，Pencil size-12（约 15px） | `Button ghost s iconLeft=Pencil` | aria-label 保留；审计归为 link-action 家族，但无下划线、不在散文中——看起来就是带图标的 ghost 文本按钮 |
| `chat/resources/ResourcesButton.tsx:35` | `Button ghost sm` + `h-panel-header-control`（24px）`gap-4 tr-text-metadata aria-expanded:bg-control-bg-selected`，图标子元素 size-14 且活跃时带 `motion-safe:animate-working` | `Button ghost m iconLeft={Stack 图标} selected={open}` | `aria-expanded:bg-control-bg-selected` 由 `selected={open}` 取代；呼吸动画通过模块级图标包装（`breathing(RiStackFill/RiStackLine)`）保留，因为 iconLeft 接收的是组件；`IconTooltip` wrapTrigger、aria-*、data-* 保留 |
| `chat/resources/CommandLogView.tsx:23` | `Button ghost sm`（28） | `Button ghost m` |  |
| `chat/resources/ResourcesInspector.tsx:298` | 裸 `<button>` resources-inspector-close：`size-28` 类 ghost，RiCloseLine size-16，aria-label 'Close resources'，无 tooltip | `IconButton ghost m icon=RiCloseLine label='Close resources'` | 现在获得 'Close resources' tooltip（IconButton 的 label = tooltip） |
| `chat/resources/ResourcesInspector.tsx:333` | `Button outline sm`（28）resources-retry | `Button outline m` |  |
| `chat/resources/ResultRow.tsx:163` | 裸 `<button>` background-result-open：透明 text-primary `px-4 py-2 tr-text-metadata`（约 19px），hover 填充，RiFileList3Line size-12 | `Button soft-primary s iconLeft=RiFileList3Line` | 绿色 ghost；className shrink-0 |
| `chat/resources/mcpRow.tsx:119` | 裸 `<button>` resource-mcp-<id>：透明 text-primary `px-4`，hover:bg-control-bg，继承 `tr-text-metadata`（约 15px） | `Button soft-primary s` | 绿色 ghost；禁用条件与 aria-describedby 保留；去掉 `disabled:hover:bg-transparent` |
| `chat/resources/mcpRow.tsx:164` | 裸 `<button>` resources-mcp-settings：透明 text-primary `px-4 tr-text-metadata`，hover:bg-control-bg（约 15px） | `Button soft-primary s` | 绿色 ghost |

## workbench（17 处）

| 位置 | 之前 | 之后 | 说明 |
| --- | --- | --- | --- |
| `panels/HeaderIconButton.tsx` | 裸图标按钮（children 作图标），active 状态 | `IconButton ghost m`，active → selected + aria-pressed | 在全部 10 个调用点（DiffPane、ChangesReviewPane、ChangesFileSection）用 icon prop 替换 children API |
| `panels/DiffPane.tsx diff-copy` | 带 Check/Copy 子元素的 `HeaderIconButton` | `HeaderIconButton icon={copied ? CopiedCheck : Copy}` | 保留成功色的对勾 |
| `panels/ChangesReviewGuide.tsx` | 裸 prev/next/apply 按钮，next 为 bg-primary，prev/apply 为 outline（24px） | `IconButton outline m`（prev）· `Button primary m iconRight`（next）· `Button outline m iconLeft`（Apply fixes）· `Button primary s`（空状态动作） | Apply fixes 上恢复了 title |
| `panels/ChangesReviewPane.tsx` | 裸 retry（quiet 文本），Viewed 开关开启时带 border-primary，以及 HeaderIconButtons | `Button ghost s`（retry）· `Button outline m iconLeft=Check selected={viewed}` · `HeaderIconButton icon=` | viewed = 持久选中状态，不是按下态 |
| `panels/TerminalWorkbench.tsx terminal-add` | 裸图标按钮 | `IconButton outline s label='New terminal'` | 仅保留定位类 |
| `panels/TerminalInstance.tsx` | 裸的类 outline 24px 按钮 | `Button outline m` |  |
| `panels/PlanPane.tsx plan-next-action-go ×5` | `NEXT_ACTION_BUTTON_CLASS`（primary 填充，28px） | `Button primary m` | 28 → 24（同高规则不适用：最近的是 24/32 → 24）；移除本地类常量 |
| `panels/PlanPane.tsx plan-start-review` | 裸 primary 24px，hover 显示 | `Button primary m` | 显示用的透明度类保留（允许） |
| `panels/PlanPane.tsx plan-add-task` | 裸 quiet 24px，带 Plus | `Button ghost m iconLeft=Plus` |  |
| `panels/PlanPane.tsx plan-review-comments` | 裸 quiet 32px | `Button ghost l iconLeft=MessageSquare` |  |
| `panels/PlanPane.tsx plan-pr-chip (<a>)` | `<a>` quiet 32px | `<a> buttonVariants ghost l` | 仍是链接 |
| `panels/PlanPane.tsx plan-open-pr` | 裸 32px，按状态为 primary 或 quiet，Loader2 切换 | `Button primary\|ghost l iconLeft=GitPullRequestArrow loading={prBusy}` | 手动 spinner 切换由 loading 取代 |
| `panels/SendReviewButton.tsx` | 裸 primary `h-panel-header-control`（24px），busy → disabled | `Button primary m iconLeft=Send loading={busy}` |  |
| `panels/ChangesPanel.tsx changes-retry` | 裸 quiet 文本 | `Button ghost s` |  |
| `panels/DiffSurface.tsx diff-open-file` | `Button ghost sm` | `Button ghost m` |  |
| `panels/SpecsPanel.tsx specs-retry` | `Button ghost sm` + RefreshCw 子元素 | `Button ghost m iconLeft=RefreshCw` |  |
| `panels/ReviewPanel.tsx review-comment-send / -file / -chat` | 裸的纯图标按钮 + IconTooltip | `IconButton ghost s`（label = tooltip） |  |

## renderers（31 处）

| 位置 | 之前 | 之后 | 说明 |
| --- | --- | --- | --- |
| `packages/resource-renderers/src/support/ReviewComposer.tsx:106` | 裸 `<button>` `.review-composer-btn tr-text-action`（Save draft；elevated 背景 + border-default，`py-2 px-8` ≈ 21.6px） | `Button outline s` | 禁用条件、data-testid、onClick 保留 |
| `packages/resource-renderers/src/support/ReviewComposer.tsx:115` | 裸 `<button>` `.review-composer-btn-primary`（Send now；bg-primary + text-on-primary，≈21.6px） | `Button primary s` | 悬停现在使用 `control-primary-bg-hovered` 与 `control-primary-text`，因此 label 保持可读（旧的 `.review-composer-btn:hover` 把背景重绘为 control-bg-hovered，却保留 text-on-primary） |
| `packages/resource-renderers/src/support/ReviewComposer.tsx:124` | 裸 `<button>` `.review-composer-btn-quiet`（Cancel；透明、text-subtle，≈21.6px） | `Button ghost s` |  |
| `packages/resource-renderers/src/support/ReviewThreadCard.tsx:102` | `IconTooltip` wrapTrigger + 裸 `<button>` `.review-thread-action`（20px，透明，text-subtle）send | `IconButton ghost s`（wrapTrigger） | 去掉 `disabled:pointer-events-none`（禁用外观由组件负责） |
| `packages/resource-renderers/src/support/ReviewThreadCard.tsx:118` | `IconTooltip` wrapTrigger + 裸 `<button>` `.review-thread-action` delete | `IconButton ghost s`（wrapTrigger） |  |
| `packages/resource-renderers/src/renderers/pdf/PdfToolbar.tsx:24` | `Button ghost icon`（28）Previous page | `IconButton ghost m` | aria-label → label（现在也是 tooltip） |
| `packages/resource-renderers/src/renderers/pdf/PdfToolbar.tsx:33` | `Button ghost icon` Next page | `IconButton ghost m` |  |
| `packages/resource-renderers/src/renderers/pdf/PdfToolbar.tsx:39` | `Button ghost icon` Zoom out | `IconButton ghost m` | 图标 14 → 16（由组件决定） |
| `packages/resource-renderers/src/renderers/pdf/PdfToolbar.tsx:43` | `Button ghost icon` Zoom in | `IconButton ghost m` | 图标 14 → 16 |
| `packages/resource-renderers/src/renderers/image/View.tsx:85` | `Button ghost icon` Fit image | `IconButton ghost m` | data-testid image-fit 保留 |
| `packages/resource-renderers/src/renderers/image/View.tsx:91` | `Button ghost icon`，文本 span '1:1'（28 方形） | `Button ghost m`（文本 '1:1'，aria-label 保留） | 文本内容不能作为 IconButton 的图标；现在是 24px 的文本按钮（px-8）而不是 28 方形 |
| `packages/resource-renderers/src/renderers/image/View.tsx:100` | `Button ghost icon` Zoom out | `IconButton ghost m` |  |
| `packages/resource-renderers/src/renderers/image/View.tsx:104` | `Button ghost icon` Zoom in | `IconButton ghost m` |  |
| `packages/resource-renderers/src/renderers/json/JsonTree.tsx:73` | `Button ghost icon className size-24 shrink-0`（展开/收起节点） | `IconButton ghost m className shrink-0` | aria-expanded 保留；图标通过 icon prop 在 ChevronDown/ChevronRight 之间切换；现在有 tooltip（见 openIssues: mcpCards.test.ts） |
| `packages/resource-renderers/src/renderers/code/PierreFile.tsx:166` | `IconTooltip` + 裸 `<button>` `size-24 bg-primary text-on-primary`（review-add-icon 装订槽） | `IconButton primary m` | onPointerDown/onClick 保留 |
| `packages/resource-renderers/src/renderers/code/PierreDiff.tsx:156` | `IconTooltip` + 裸开关 `<button>` `h-24 px-8 tr-text-metadata`，kept → text-feedback-success（hunk-keep） | `Button ghost m iconLeft=Check selected={kept}` | aria-pressed 保留 + `selected={kept}`；绿色的 'kept' 文字色由共享的选中外观取代；Button 外层仍保留 `IconTooltip` |
| `packages/resource-renderers/src/renderers/code/PierreDiff.tsx:170` | `IconTooltip` + 裸 `<button>` `size-24` ghost（hunk-revert） | `IconButton ghost m` | `disabled={reverting}` 保留（旧代码没有 spinner，所以不用 loading） |
| `packages/resource-renderers/src/renderers/code/PierreDiff.tsx:177` | `IconTooltip` + 裸 `<button>` `size-24` ghost（hunk-ask-agent） | `IconButton ghost m` |  |
| `packages/resource-renderers/src/renderers/code/PierreDiff.tsx:428` | `IconTooltip` + 裸 `<button>` `size-24 bg-primary`（review-add-icon 装订槽） | `IconButton primary m` |  |
| `packages/resource-renderers/src/renderers/RegionReviewSurface.tsx:218` | `IconTooltip` + 裸 `<button>` `absolute size-24 bg-primary`（区域上的 review-add-icon） | `IconButton primary m className 'pointer-events-auto absolute -right-12 -bottom-12'` |  |
| `packages/resource-renderers/src/renderers/notebook/Diff.tsx:76` | `IconTooltip` + 裸 `<button>` `size-24` ghost（notebook-comment-cell-${side}） | `IconButton ghost m` |  |
| `packages/resource-renderers/src/renderers/notebook/View.tsx:108` | `IconTooltip` + 裸 `<button>` `size-24` ghost（notebook-comment-cell） | `IconButton ghost m` |  |
| `packages/resource-renderers/src/support/UnplacedReviewStrip.tsx:92` | 裸 `<button>` `ml-12 px-8 py-4 tr-text-action text-primary hover:bg-control-bg-hovered`（≈23.6px） | `Button soft-primary m className ml-12` | 绿色 ghost |
| `packages/resource-renderers/src/support/ExpensiveDiffNotice.tsx:18` | `Button outline sm` | `Button outline m` |  |
| `packages/resource-renderers/src/support/PreviewCommenting.tsx:197` | portal 裸 `<button>` `.review-add-icon .review-add-icon-float`（22px、border-default、elevated 背景、primary 图标、投影、原生 title） | `IconButton outline s className review-add-icon-float` | title → IconButton 的 tooltip（与 label 同文本）；ref 保留；`.closest('.review-add-icon')` 的 pointerdown 检查现在用 `iconRef.current.contains(target)`；`review-add-icon-float`（固定定位 + 命令式的 data-visible/data-dragging 可见性）作为布局类保留。外观与旧实现的差异：图标现在是 text-default 而非 primary；投影以 `shadow-[var(--shadow-sm)]` 恢复 |
| `thinkrail-extensions/visualize/web/MermaidView.tsx:67` | 裸 `<button>` `absolute p-4 border-default elevated bg text-muted`（≈24px），title 'Full screen' | `IconButton outline m className 'absolute top-4 right-4', tooltip 'Full screen'` | aria-label 'View diagram full screen' → label |
| `thinkrail-extensions/visualize/web/PanZoomView.tsx:121` | 裸 `<button className={btn}> p-4` ghost，图标 16（24px）Zoom out | `IconButton ghost m` | 移除本地 `btn` 常量；现在有 tooltip |
| `thinkrail-extensions/visualize/web/PanZoomView.tsx:133` | 裸 `btn` Zoom in（24px） | `IconButton ghost m` |  |
| `thinkrail-extensions/visualize/web/PanZoomView.tsx:139` | 裸 `btn` Reset zoom，图标 14（22px） | `IconButton ghost m` | 见 sizeExceptions |
| `thinkrail-extensions/visualize/web/FileCard.tsx:44` | `CardAction: IconTooltip` + 裸 `<button>` `size-24` ghost（children 为图标元素） | `IconButton ghost m` | `CardAction` 现在接收 `icon: ButtonIcon` 而不是 children；两个调用点（tool-visualize-open-tab、tool-visualize-fullscreen）均已更新 |
| `packages/resource-renderers/src/support/reviewWidgets.ts review-add-icon / thread actions (plain DOM)` | `.review-add-icon / .review-thread-action` CSS 外观 | `buttonVariants({variant:'outline', size:'s', shape:'icon'})` 类字符串 + `shadow-[var(--shadow-sm)]` + `buttonDataAttributes` | 移除 index.css 规则；holder/float 定位保留 |

## 绿色（accent-text）ghost 动作 → `soft-primary`

- `shell/LayoutSettings.tsx:268 — Save <preset> name（对勾图标）`
- `updates/UpdateReadyButton.tsx:85 — Update available / Update ready / …（顶栏更新入口）`
- `chat/resources/ResultRow.tsx:163 — Open log / Transcript`
- `chat/resources/mcpRow.tsx:119 — MCP 行内动作（Disable in this chat / Enable in this chat / Reconnect / Sign in / Review / Edit）`
- `chat/resources/mcpRow.tsx:164 — Settings（MCP servers 章节标题处）`
- `packages/resource-renderers/src/support/UnplacedReviewStrip.tsx:92 — Show in {renderer}`
- `panels/SubagentSettings.tsx — SourceTag "Custom ↺"（accent 文字 link-action）`

## 尺寸例外（Size exceptions）

- `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:667` → `l`：Skip 与 Next/Submit（primary l，32px）
  组成卡片页脚的按钮对；作为 ghost 它现在有 hover 填充，因此 20px 的 Skip 挨着 32px 的 primary 会让按钮对
  错位。仅按最近高度规则会得到 s（约 18px）。
- `thinkrail-extensions/visualize/web/PanZoomView.tsx:139` → `m`：原本 22px（同高 → s），但它与 24px 的
  zoom-out/zoom-in 按钮同处一个缩放工具栏；m 让这一行保持统一

## 剩余例外 —— 保持原样，需要变体决策（Remaining exceptions (kept as-is, need a variant decision)）

- `shell/WorkspaceChatHistory.tsx:208`：红字动作（无变体）——closed-chat-delete 在聚焦时变红（`focus:text-feedback-error`）
- `shell/LayoutSettings.tsx:297`：红字动作（无变体）——删除预设仅在悬停时变红（`hover:bg-feedback-error-subtle hover:text-feedback-error`）
- `panels/TemplatesSettings.tsx:402`：红字动作（无变体）——模板 Delete 图标按钮，muted 且 `hover:text-feedback-error`（`PopoverTrigger asChild` 位于 `IconTooltip` wrapTrigger 内）；保持未改
- `chat/HistoryOverlay.tsx:88`：红字动作（无变体）——history-delete-chat 在悬停时变为 `text-feedback-error`
- `chat/promptImages.tsx:184`：红字动作（无变体）——`tone=error` 的 FileChip 内的关闭按钮继承 `text-feedback-error`
- `chat/resources/resourceRow.tsx:267`：红字动作（无变体）——RowActionButton（resource-stop）是 ghost，仅在悬停时变红（danger）；仍保持为裸 `<button>`
- 已由 `tone` 解决：`chat/resources/ResourcesInspector.tsx` 的 resource-detail-stop（`Button outline m tone="danger"`）、resources-stop-all（`IconButton ghost m tone="danger"`）与 `chat/resources/ResourcesButton.tsx` 的 ResourcesAttention（`IconButton ghost m tone="warning"`）——它们的调用点颜色覆盖已从守护规则中移除。
- `panels/ReviewPanel.tsx review-clear, review-comment-delete`：红字动作（无变体）——悬停时变红
- `panels/ReviewPanel.tsx review-file-done, review-comment-resolve`：成功色悬停的图标动作（无变体）

## 已退役的规格覆盖（Spec overrides retired）

- `shell/layout/workbenchCenter.tsx:100,115; workbenchSide.tsx:184,258; workbenchBottom.tsx:64; workbenchTabs.tsx:287; AuxiliaryPaneHeader.tsx:71; WorkspaceWorkbench.tsx:740; WorkspaceChatHistory.tsx:36`：整高 28px（`w-panel-header-row`）的条带/头部动作，其中若干带 border-l 细线分隔；现在是 24px 的居中方块、无分隔（apps/web/src/shell/layout/SPEC.md（Layout grammar 中的 'Full-height strip actions share that width.'））
- `components/ErrorBoundary.tsx:92,101`：components 模块现在导入 `@thinkrail/ui/button`，而它不在其 'Allowed deps' 列表中（React、@remixicon/react、@thinkrail/ui/utils、lib、constants）（apps/web/src/components/SPEC.md（Allowed deps））
- `chat/Composer.tsx:661`：`rounded-full` 28px ghost 圆形（History：`IconTooltip` 下的 `Button variant="ghost" size="icon"`）（apps/web/src/chat/SPEC.md（Composer trailing controls））
- `chat/Composer.tsx:673`：Stop 是 `rounded-full h-28` ghost 胶囊；紧凑模式收缩为 28px 图标圆（`@max-md:w-28 px-0`）——现在是 ghost m 按钮，收缩为 24px 方块（`@max-md:w-24 px-0`）（apps/web/src/chat/SPEC.md（Composer trailing controls / Compact mode））
- `chat/ExtUiPrompt.tsx:15`：`OPTION_BUTTON` 的 `h-auto min-h-28 whitespace-normal break-words justify-start text-left py-4`——'Option buttons wrap long labels'（apps/web/src/chat/SPEC.md（Extension dialogs ask inside the chat））
- `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:667,678,688,1012`：Skip/Next/Submit/Add note 上的 `focus-visible:ring-2` ring-primary 聚焦环——规格说卡片内每个聚焦指示都是应用范围的 `focus-visible:ring-2` + `ring-primary` 环；共享 Button 改用偏移的 `outline-primary` 聚焦轮廓（仍是 accent）（apps/web/src/chat/tools/SPEC.md（"Every focus indicator in the card is the app-wide accent focus ring"））
- `chat/resources/mcpRow.tsx:164`：规格把章节标题处的 Settings 称作 "link"（`resources-mcp-settings`）；它现在是 soft-primary s 按钮（仅措辞差异）（apps/web/src/chat/resources/SPEC.md（MCP servers paragraph））

## 未转换的专用按钮（Specialized buttons not converted）

| 区域 | 位置 | 原因 |
| --- | --- | --- |
| shell | `auth/LoginDialog.tsx:200` | login-option：整行可选的全宽选项行（审计 list-row），类似 radio 选项 |
| shell | `auth/LoginDialog.tsx:215` | login-paste-toggle：带下划线的类文本链接动作（审计 link-action） |
| shell | `shell/WorkspaceChatHistory.tsx:165` | closed-chat-item：菜单行（`menuItemClass`，整行点击目标） |
| shell | `shell/locationBar/BranchSegment.tsx:116` | scope-remote-unpushed：chip（`chipClass`） |
| shell | `shell/locationBar/BranchSegment.tsx:127` | scope-remote-behind：chip（`chipClass`） |
| shell | `shell/NativeWindowControls.tsx:33` | 原生窗口控件（最小化），SPEC 中记录的 46x40 标题栏按钮 |
| shell | `shell/NativeWindowControls.tsx:42` | 原生窗口控件（最大化/还原） |
| shell | `shell/NativeWindowControls.tsx:52` | 原生窗口控件（关闭） |
| shell | `shell/JbcentralQuotaIndicator.tsx:41` | 'Quota unavailable · Retry'：顶栏的纯文本状态读数，充当类链接的重试（审计 link-action）；shell 规格把它描述为读数 |
| shell | `shell/JbcentralQuotaIndicator.tsx:87` | 过期配额读数：与非按钮的「可用」读数（span）是同一元素、同样的纯文本样式；改用 Button 会让读数在可用/过期之间改变几何尺寸——按类链接的内联动作处理（需要决策，审计称其为 'button'） |
| shell | `shell/layout/AuxiliaryRail.tsx:51` | 边缘轨道的工具选择器：横过来的标签页语法（roving 工具栏、aria-pressed、由框架拥有的选中填充 + accent 指示，见 shell/layout SPEC） |
| shell | `shell/layout/workbenchTabs.tsx:554` | WorkbenchTab 标签页按钮（role=tab）/ 它的轨道模式选择器——标签条 |
| shell | `shell/layout/workbenchRails.tsx:101` | 轨道组选择器 + 放置目标（类标签的轨道条目，见 AuxiliaryRail） |
| shell | `shell/LayoutSettings.tsx:365` | layout-add-new：大型可点击卡片（PopoverTrigger） |
| shell | `shell/LayoutSettings.tsx:459` | layout-advanced-toggle：展开头 |
| projects | `panels/WelcomePanel.tsx:205` | 大型可点击 Welcome 卡片 |
| projects | `panels/ExistingWorktreeDialog.tsx:153` | 大型可点击候选卡片 / 列表行（PROJ-055） |
| projects | `panels/ProjectTree.tsx:434` | settled-shelf-toggle：展开头（aria-expanded） |
| projects | `panels/ProjectTree.tsx:459` | settled-shelf-more：整宽列表行 / 类链接动作（PROJ-089） |
| projects | `panels/ProjectTree.tsx:549` | project-expand：树展开箭头 |
| projects | `panels/ProjectTree.tsx:574` | project-name：树行点击目标 |
| projects | `panels/ProjectTree.tsx:870` | 工作区标识：树行点击目标 |
| settings | `panels/SettingsDialog.tsx:71` | 设置章节导航条目——标签条（SET-004） |
| settings | `panels/SettingsDialog.tsx:93` | Feedback 导航卡片：位于轨道标签列表中的带选中状态的章节导航条目，在 192px 轨道中放换行的多行推广文本；固定高度、不换行的 Button 会溢出（SET-005）——如果它应成为 Button 则需要决策 |
| settings | `panels/McpSettings.tsx:475` | ExposureMenu 触发器，样式为状态 chip（`CHIP_CLASS`），select 家族（SET-081） |
| settings | `panels/VisibleModelsSettings.tsx:37` | 行展开头（SET-037） |
| settings | `panels/ModelContextSettings.tsx:315` | 行展开头 'By each model'（SET-037） |
| settings | `panels/AppearanceSettings.tsx:148` | 类 radio 的主题列表选项，带 aria-pressed（SET-054） |
| settings | `panels/AppearanceSettings.tsx:194` | 展开头（SET-055） |
| settings | `panels/NotificationsSettings.tsx:21` | 带 aria-expanded 的 'How to enable' 展开开关（SET-036） |
| settings | `panels/SettingsSwitch.tsx:17` | 开关（SET-006） |
| chat | `chat/ReviewPackageComments.tsx:30` | 展开行开关（CHAT-029） |
| chat | `chat/assistantLinks.tsx:59` | 散文中的带下划线内联文本链接（CHAT-041） |
| chat | `chat/ModelEffortPicker.tsx:269` | 分段式 effort 滑块选项（CHAT-111） |
| chat | `chat/ModelEffortPicker.tsx:630` | 审计认为 `FOOTER_LINK` 是 link-action（CHAT-112）——见决策：看起来是 ghost s 按钮 |
| chat | `chat/ModelEffortPicker.tsx:647` | 审计认为 `FOOTER_LINK` 是 link-action（CHAT-112）——见决策 |
| chat | `chat/TemplateEditorDialog.tsx:264` | 分段式范围选项（CHAT-157） |
| chat | `chat/ThinkingSelector.tsx:40` | 菜单项行（CHAT-121） |
| chat | `chat/activityBreadcrumbs.tsx:115` | 复合面包屑片段内的展开箭头（可旋转），其包装元素拥有 hover 填充；展开控件（CHAT-021）——见决策 |
| chat | `chat/activityBreadcrumbs.tsx:126` | 面包屑跳转 link-action（CHAT-022） |
| chat | `chat/ActivityGroup.tsx:208` | 展开头（CHAT-060） |
| chat | `chat/ActivityGroup.tsx:422` | 展开头（CHAT-061） |
| chat | `chat/turns.tsx:371` | Show more/less 展开（CHAT-025） |
| chat | `chat/turns.tsx:408` | 展开卡片头（CHAT-027） |
| chat | `chat/turns.tsx:548` | 压缩展开分隔条（CHAT-048） |
| chat | `chat/turns.tsx:710` | 轮次分隔条中的 ArtifactChip chip（CHAT-053） |
| chat | `chat/turns.tsx:759` | artifact 列表行（CHAT-054） |
| chat | `chat/turns.tsx:888` | 轮次分隔条的 Review chip（CHAT-055）——绿色 ghost 外观，见决策 |
| chat | `chat/turns.tsx:905` | 轮次分隔条的 'N still running' rounded-full chip（CHAT-056） |
| chat | `chat/Composer.tsx:497` | mention 菜单项（CHAT-082） |
| chat | `chat/Composer.tsx:521` | slash-templates-empty 菜单行（CHAT-085） |
| chat | `chat/Composer.tsx:689` | composer 发送胶囊的主段 chat-send（有文档记录的例外） |
| chat | `chat/Composer.tsx:711` | composer 发送胶囊的箭头段 send-menu（有文档记录的例外） |
| chat | `chat/Composer.tsx:729` | 发送模式菜单行（CHAT-094） |
| chat | `chat/FileChip.tsx:46` | chip（FileChip 的 button 分支） |
| chat | `chat/HistoryOverlay.tsx:142` | 整行点击目标（PromptRow 选择） |
| chat | `chat/HistoryOverlay.tsx:247` | 整行点击目标（MessageRow 选择） |
| chat | `chat/HistoryOverlay.tsx:589` | 整宽 border-t 页脚条提示（history-expand-hint），整行目标——见决策 |
| chat | `chat/ToolCard.tsx:59` | 展开头（CHAT-059） |
| chat | `chat/TodoList.tsx:282` | DoneGroup 展开（CHAT-168） |
| chat | `chat/TodoList.tsx:324` | 文本 link-action（CHAT-169） |
| chat | `chat/TodoList.tsx:342` | 文本 link-action 路径（CHAT-170） |
| chat | `prompt/SlashCommandCompletion.tsx:152` | 模板 slot-hint chip（CHAT-086） |
| chat | `prompt/SlashCommandCompletion.tsx:188` | slash 命令菜单项（CHAT-084） |
| tools | `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:779` | TabChip（ask-tab）——标签条 |
| tools | `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:1093` | OptionRow（ask-option）——radio/checkbox 选项卡片 |
| tools | `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:1174` | OtherOptionRow ask-custom-toggle——checkbox/radio 指示器 |
| tools | `thinkrail-extensions/ask-user-question/web/AskUserQuestionCard.tsx:1299` | ask-unanswered——警告色的带下划线文本链接 |
| tools | `chat/tools/mcp/McpActions.tsx:33` | CliActionButton（`ACTION_CLASS`）——accent 文本链接，hover:underline |
| tools | `chat/tools/mcp/McpActions.tsx:86` | mcp-full-output-retry（`ACTION_CLASS`）——错误段落内的带下划线文本链接 |
| tools | `chat/tools/mcp/McpActions.tsx:163` | mcp-full-output 触发器（`ACTION_CLASS`）——accent 文本链接，hover:underline |
| tools | `chat/tools/mcp/McpJsonTree.tsx:100` | JSON 'Show all N'——树内的带下划线文本链接 |
| tools | `chat/tools/mcp/McpJsonTree.tsx:130` | JsonBranch——树展开行 |
| tools | `chat/tools/mcp/McpJsonTree.tsx:193` | mcp-structured-toggle——展开头 |
| tools | `chat/tools/mcp/McpResourceCards.tsx:98` | 'Show all N resources'——带下划线文本链接 |
| tools | `chat/tools/Collapsible.tsx:34` | collapsible-toggle 'Show all N lines'——带下划线文本链接 / 展开 |
| tools | `chat/tools/subagent/AgentCard.tsx:75` | agent-report-toggle——样式为带下划线链接的展开（箭头切换） |
| tools | `chat/tools/subagent/AgentCard.tsx:98` | agent-open-transcript——accent 带下划线文本链接 |
| tools | `chat/tools/ToolFileLink.tsx:39` | ToolFileLink——文件文本链接（截断，hover:underline） |
| tools | `chat/resources/resourceRow.tsx:431` | resource-inspect——整行点击目标（名称 + 活动） |
| tools | `chat/resources/ResultRow.tsx:146` | background-result-toggle——展开头 / 整行点击目标 |
| tools | `chat/resources/ResourcesDock.tsx:70` | resources-dock-toggle——整宽展开摘要行 |
| workbench | `panels/planFileRow.tsx, TreeRow.tsx, ChangesPanel.tsx change-item, ReviewPanel.tsx review-file-row / review-comment-open, SpecsPanel.tsx spec-node` | 列表/树行——整行点击目标 |
| workbench | `panels/PlanPane.tsx plan-change-set-toggle, plan-item-toggle; ChangesFileSection.tsx changes-section-toggle; SpecsPanel.tsx spec-toggle` | 展开控件 |
| workbench | `panels/PlanPane.tsx plan-commit-chip, revision chip` | 状态 chip |
| workbench | `panels/PlanPane.tsx plan-item-changes-requested, plan-now-status ×2; ChangesFileSection.tsx expand/retry links` | 内联文本 link-actions |
| workbench | `panels/PlanPane.tsx plan-now-idle` | 整宽的整行入口 |
| workbench | `panels/ChangesFileSection.tsx changes-section-viewed` | 复选框（role/aria-checked） |
| renderers | `packages/resource-renderers/src/renderers/VisualDiff.tsx:256` | 滑动分隔条 / 分割器（control 的 resize-handle 类） |
| renderers | `packages/resource-renderers/src/renderers/code/PierreDiff.tsx:473` | 阻塞选择提示内的 'Dismiss' hover 下划线文本链接动作 |
| renderers | `packages/resource-renderers/src/renderers/json/JsonTree.tsx:83` | 树行选择点击目标（节点标签） |
| renderers | `packages/resource-renderers/src/renderers/json/JsonTree.tsx:97` | 带编号的评论标记钉（rounded-full 20px 徽标），不是动作按钮 |
| renderers | `packages/resource-renderers/src/renderers/csv/Diff.tsx:216` | CSV 单元格选择点击目标 |
| renderers | `packages/resource-renderers/src/renderers/csv/Diff.tsx:240` | 带编号的评论标记钉（徽标） |
| renderers | `packages/resource-renderers/src/renderers/csv/View.tsx:110` | CSV 单元格选择点击目标 |
| renderers | `packages/resource-renderers/src/renderers/csv/View.tsx:120` | 带编号的评论标记钉（徽标） |
| renderers | `packages/resource-renderers/src/renderers/RegionReviewSurface.tsx:199` | 带编号的区域评论标记钉（区域角上的 rounded-full 24px 徽标） |
| renderers | `packages/resource-renderers/src/support/ReviewThreadCard.tsx:83` | 线程锚点标签 link-action（eyebrow 文本，激活锚点） |
| renderers | `packages/resource-renderers/src/support/markdownLinks.tsx:142` | 文档文件链接，样式为带下划线的链接文本（SPEC：'a button styled as document-link text'） |
| renderers | `packages/resource-renderers/src/support/RenderedDiff.tsx:186` | 折叠隐藏区段的整宽展开（rendered-diff-collapsed） |
| renderers | `thinkrail-extensions/visualize/web/FileCard.tsx:22` | FileLink：内联路径文本链接（hover 下划线） |

## 第二轮 —— 类按钮控件并入系统（Second pass — button-like controls moved onto the system）

| 位置 | 之前 | 之后 |
| --- | --- | --- |
| `chat/turns.tsx` `turn-divider-<kind>`（artifact chip） | 透明 accent 文字 chip，`px-4`，hover 底色，展开 = 选中填充 | `Button soft-primary s`（`iconLeft` 为种类图标，列出多项时 `iconRight` 为箭头） |
| `chat/turns.tsx` `turn-divider-review` | 透明 accent 文字 chip | `Button soft-primary s` `iconLeft={Eye}` |
| `chat/turns.tsx` `turn-divider-running` | `rounded-full` 的 accent 浅色胶囊 | `Button soft-primary s`，图标上保留工作动画 |
| `panels/ChangesFileSection.tsx` `changes-section-toggle` | 手写的 20px ghost 箭头 | `IconButton ghost s`（保留 `aria-expanded`） |
| `panels/ChangesFileSection.tsx` `changes-section-viewed` | 手写的 24px ghost + 自绘复选框方块 | `Button ghost m` `selected` + `aria-pressed`，Remix 复选框图标 |
| `panels/ChangesFileSection.tsx` `changes-section-menu` | 手写的 24px ghost 菜单触发器 | `DropdownMenuTrigger asChild` → `IconButton ghost m` |
| `panels/ChangeRowActions.tsx` `change-row-menu` | 手写的 20px hover 显示触发器 | `IconButton ghost s`（保留显示逻辑） |
| `panels/PlanPane.tsx` `plan-menu` | 手写的 32px ghost 触发器 | `IconButton ghost l` |
| `panels/ChangesScopeMenu.tsx` `changes-scope-trigger` | 手写的 24px ghost 文本触发器 | `Button ghost m`（`iconLeft` GitCompare，`iconRight` 箭头） |
| `panels/PlanCommitsMenu.tsx` `plan-commits-trigger` | 手写的 24px ghost 文本触发器 | `Button ghost m` |
| `chat/ModelEffortPicker.tsx` `model-set-default`, `thinking-reset` | `FOOTER_LINK` ghost 文本按钮 | `Button ghost s`（该常量已删除） |
| `prompt/SlashCommandCompletion.tsx` `slot-hint` | bordered elevated 提示按钮 | `Button outline s` + elevation 阴影 token |

按设计未迁移：model / thinking 选择器触发器（select 家族，`data-[open]` 边框）、history scope chip，以及下面这些
裸的红 / 琥珀 / 悬停变绿动作（`tone` 会在所有状态下给 ghost 或 outline 的 label 上色；仅悬停时变色的动作没有
对应的变体）。

## 评审后续 —— rebase 到 main 之后（Review follow-ups (after the rebase onto main)）

ask-user-question 卡片在 main 上被移入 `thinkrail-extensions/ask-user-question`（#869）；下表各行使用新的路径，
它的迁移也随之一起过来了。

以下是处理 PR 评审期间所做的系统级改动；上面按位置划分的行凡有变化都已同步更新。

| 改动 | 位置 | 原因 |
| --- | --- | --- |
| `Button` / `IconButton` 在 `disabled` 时把 tooltip 触发器移到包装 span 上；`Button` 增加 `tooltip`；两者都不接受 `title` | `packages/ui/button.tsx`；9 处原来的 `title` 位置（PlanPane ×4、ChangesReviewGuide apply、ChangesScopeMenu、PlanCommitsMenu、PrSetupDialog run、SubagentSettings） | 禁用按钮的 `pointer-events: none` 会同时挡住原生 `title` 和未包装的 tooltip，因此不可用的原因从不显示 |
| `data-row` 内的 ghost 在 hover / pressed / focus / selected 时使用 `container-elevated-bg` | 项目与工作区行、改动行、todo 行、history-overlay 行、cmdk `CommandItem` | 行填充与 ghost 填充是同一个 `--hover` 角色，因此行内动作与已打开行的菜单都不可见 |
| `soft-primary` 的选中态 = 它的 hover 外观 | 轮次分隔条的 artifact chip（`selected={many && expanded}`） | 展开的 "N files" chip 看起来像折叠状态 |
| ghost 与 outline 上的 `tone`（`warning` / `danger`） | ResourcesAttention、Stop all、resource-detail Stop、model-picker 的置顶星 | 调用点的颜色会被 ghost 自身的聚焦 / 按下文字色覆盖 |
| `button-disabled` 变体（`[data-disabled="true"]` 或 `:disabled:not([data-loading])`） | 每处禁用外观 | `<fieldset disabled>`（保存中的 Environment 设置）会让按钮看起来仍然可用 |
| `spinIcon` 在减少动效下显示静态 spinner | 刷新按钮 | 减少动效时，加载中的刷新看起来像空闲 |
| 纯图标 ghost 的聚焦环画在方块内侧 | 头部条、轨道 | 28px 且 `overflow: hidden` 的条会裁掉外侧聚焦环 |
| tooltip 内容位于 `z-150` | `packages/ui/tooltip.tsx` | toast 关闭按钮的 tooltip 被画在 toast 之下（`z-[100]`） |
| `buttonDataAttributes`、`withIconClass` helper | 轨道、Monaco 评论 widget；六个图标包装 | 一份 data 属性契约和一个着色 / 动画图标 helper，取代多处复制 |
| `BranchPicker` 触发器使用 `Button ghost m`，除非传入 pill 类 | Changes 目标选择器、位置栏的 Compare to | Changes 工具栏的作用域与目标触发器重新读作一对 |
| 轨道放置提示增加内嵌描边；徽标与 join droppable 位于 32px 框架上 | `workbenchRails.tsx`、`workbenchTabs.tsx` | 选中填充盖住了提示；徽标盖住了更小的图标 |

## 剩余的裸 `<button>` 元素 —— 当前代码（Remaining raw `<button>` elements (current code)）

从工作副本生成（`packages/ui` 之外的裸 `<button>`；`web/` = apps/web/src，`rr/` =
packages/resource-renderers/src，`ext/` = thinkrail-extensions）：123 个元素，按它们不是 `Button` /
`IconButton` 的原因分组。

### 面板工具轨道 —— 在按钮系统之上（通过 `buttonVariants` 的 ghost M 外观），保留裸元素（3）

_原因：_ roving 工具栏、拖放目标与装饰都挂在元素本身；它穿着共享外观，并用 `data-selected` 表示打开的面板。

- `web/shell/layout/AuxiliaryRail.tsx:76` `tool-rail-${entry.tool`
- `web/shell/layout/workbenchRails.tsx:127`
- `web/shell/layout/workbenchTabs.tsx:589`

### 红 / 琥珀 / 绿字动作 —— 需要变体决策（10）

_原因：_ 没有任何变体能在透明表面上承载带颜色的 label。

- `web/chat/HistoryOverlay.tsx:102` `history-delete-chat`
- `web/chat/promptImages.tsx:187`
- `web/chat/resources/resourceRow.tsx:272` `testId`
- `web/panels/ReviewPanel.tsx:144` `review-clear`
- `web/panels/ReviewPanel.tsx:203` `review-file-done`
- `web/panels/ReviewPanel.tsx:463` `review-comment-delete`
- `web/panels/ReviewPanel.tsx:488` `review-comment-resolve`
- `web/panels/TemplatesSettings.tsx:430` `template-delete`
- `web/shell/LayoutSettings.tsx:303`
- `web/shell/WorkspaceChatHistory.tsx:216` `closed-chat-delete`

### 分离式动作胶囊 —— 有文档记录的例外（3）

_原因：_ 两段式胶囊各自带色调：composer 的发送胶囊（chat/SPEC.md，Composer trailing controls）与顶栏的
ship 分离式动作（primary / warning 色调 + 菜单半边），没有任何单一 Button 变体能表达。

- `web/chat/Composer.tsx:686` `chat-send`
- `web/chat/Composer.tsx:710` `send-menu`
- `web/shell/locationBar/ShipSegments.tsx:90` `${testid`

### 标签页与导航（3）

_原因：_ 标签条 / 导航列表的选中语法。

- `ext/ask-user-question/web/AskUserQuestionCard.tsx:783` `ask-tab`
- `web/panels/SettingsDialog.tsx:74` `settings-nav-${id`
- `web/panels/SettingsDialog.tsx:96` `settings-nav-feedback`

### 选项、分段控件、复选框、开关（9）

_原因：_ radio / checkbox / switch 语义。

- `web/chat/ModelEffortPicker.tsx:273` `thinking-option`
- `web/chat/TemplateEditorDialog.tsx:272` `template-scope-${id`
- `web/chat/ThinkingSelector.tsx:43` `thinking-option`
- `ext/ask-user-question/web/AskUserQuestionCard.tsx:1104` `ask-option`
- `ext/ask-user-question/web/AskUserQuestionCard.tsx:1186` `ask-custom-toggle`
- `web/panels/AppearanceSettings.tsx:169` `theme-option-${id`
- `web/panels/MergePrDialog.tsx:69`
- `web/panels/PrComposeDialog.tsx:81`
- `web/panels/SettingsSwitch.tsx:17` `testId`

### 展开控件（26）

_原因：_ 整宽或内联的展开/收起头部以及 "show more" 开关。

- `web/chat/ActivityGroup.tsx:212` `${testId`
- `web/chat/ActivityGroup.tsx:428` `activity-step-toggle`
- `web/chat/HistoryOverlay.tsx:608` `history-expand-hint`
- `web/chat/ReviewPackageComments.tsx:30` `review-package-item-toggle`
- `web/chat/TodoList.tsx:293` `todo-group-done`
- `web/chat/ToolCard.tsx:61` `tool-card-toggle`
- `web/chat/activityBreadcrumbs.tsx:117`
- `web/chat/resources/ResourcesDock.tsx:72` `resources-dock-toggle`
- `web/chat/resources/ResultRow.tsx:150` `background-result-toggle`
- `web/chat/tools/Collapsible.tsx:36` `collapsible-toggle`
- `web/chat/tools/mcp/McpJsonTree.tsx:196` `mcp-structured-toggle`
- `web/chat/tools/subagent/AgentCard.tsx:76` `agent-report-toggle`
- `web/chat/turns.tsx:381` `user-message-toggle`
- `web/chat/turns.tsx:419` `skill-invocation-toggle`
- `web/chat/turns.tsx:575`
- `web/panels/AppearanceSettings.tsx:215` `testId`
- `web/panels/ModelContextSettings.tsx:340` `model-context-customize`
- `web/panels/NotificationsSettings.tsx:23`
- `web/panels/PlanPane.tsx:109` `plan-change-set-toggle`
- `web/panels/PlanPane.tsx:335` `plan-item-toggle`
- `web/panels/ProjectTree.tsx:437` `settled-shelf-toggle`
- `web/panels/ProjectTree.tsx:563` `project-expand`
- `web/panels/SpecsPanel.tsx:156` `spec-toggle`
- `web/panels/VisibleModelsSettings.tsx:39` `visible-models-toggle`
- `web/shell/LayoutSettings.tsx:467` `layout-advanced-toggle`
- `rr/support/RenderedDiff.tsx:188` `rendered-diff-collapsed`

### 行、树项、卡片、菜单行（28）

_原因：_ 整行点击目标与可选卡片，而不是独立动作。

- `web/auth/LoginDialog.tsx:210` `login-option`
- `web/chat/Composer.tsx:501` `mention-item`
- `web/chat/Composer.tsx:525` `slash-templates-empty`
- `web/chat/Composer.tsx:728` `mode.testid`
- `web/chat/HistoryOverlay.tsx:157`
- `web/chat/HistoryOverlay.tsx:255`
- `web/chat/resources/resourceRow.tsx:437` `resource-inspect`
- `web/chat/turns.tsx:803` `${testid`
- `web/panels/ChangesPanel.tsx:242` `change-item`
- `web/panels/ChangesReviewGuide.tsx:173` `changes-review-guide-step`
- `web/panels/ChangesReviewGuide.tsx:227`
- `web/panels/ExistingWorktreeDialog.tsx:152` `existing-worktree-candidate`
- `web/panels/PlanPane.tsx:807` `plan-now-idle`
- `web/panels/ProjectTree.tsx:466` `settled-shelf-more`
- `web/panels/ProjectTree.tsx:592` `project-name`
- `web/panels/ProjectTree.tsx:892`
- `web/panels/ReviewPanel.tsx:176` `review-file-row`
- `web/panels/ReviewPanel.tsx:388` `review-comment-open`
- `web/panels/SpecsPanel.tsx:173` `spec-node`
- `web/panels/TreeRow.tsx:49` `testid`
- `web/panels/WelcomePanel.tsx:205` `cta ? "welcome-cta" : "welcome-action"`
- `web/panels/planFileRow.tsx:17` `plan-file-row`
- `web/prompt/SlashCommandCompletion.tsx:190` `slash-command`
- `web/shell/LayoutSettings.tsx:371` `layout-add-new`
- `web/shell/WorkspaceChatHistory.tsx:174` `closed-chat-item`
- `rr/renderers/csv/Diff.tsx:215`
- `rr/renderers/csv/View.tsx:112`
- `rr/renderers/json/JsonTree.tsx:85`

### Chips —— 状态 chip（7）

_原因：_ 状态 / 元数据 chip，有自己的形状。

- `web/chat/FileChip.tsx:46`
- `web/chat/TodoList.tsx:335` `todo-changes-chip`
- `web/panels/McpSettings.tsx:488` `mcp-server-exposure`
- `web/panels/PlanPane.tsx:125` `plan-commit-chip`
- `web/panels/PlanPane.tsx:223`
- `web/shell/locationBar/BranchSegment.tsx:163` `scope-remote-unpushed`
- `web/shell/locationBar/BranchSegment.tsx:174` `scope-remote-behind`

### 文本链接（26）

_原因：_ 散文或元数据中带下划线或链接色的内联文本。

- `web/auth/LoginDialog.tsx:225` `login-paste-toggle`
- `web/chat/TodoList.tsx:353` `todo-change-path`
- `web/chat/activityBreadcrumbs.tsx:132`
- `web/chat/assistantLinks.tsx:108` `chat-file-link`
- `ext/ask-user-question/web/AskUserQuestionCard.tsx:1315` `ask-unanswered`
- `web/chat/tools/ToolFileLink.tsx:39` `tool-file-link`
- `web/chat/tools/mcp/McpActions.tsx:35` `mcp-cli-action`
- `web/chat/tools/mcp/McpActions.tsx:89` `mcp-full-output-retry`
- `web/chat/tools/mcp/McpActions.tsx:165` `mcp-full-output`
- `web/chat/tools/mcp/McpJsonTree.tsx:102`
- `web/chat/tools/mcp/McpJsonTree.tsx:132`
- `web/chat/tools/mcp/McpResourceCards.tsx:119`
- `web/chat/tools/subagent/AgentCard.tsx:99` `agent-open-transcript`
- `web/panels/ChangesFileSection.tsx:360` `changes-section-expand`
- `web/panels/ChangesFileSection.tsx:368`
- `web/panels/ChangesFileSection.tsx:383` `changes-section-retry`
- `web/panels/ChangesReviewPane.tsx:595`
- `web/panels/PlanPane.tsx:365` `plan-item-changes-requested`
- `web/panels/PlanPane.tsx:752` `plan-now-status`
- `web/panels/PlanPane.tsx:764` `plan-now-status`
- `web/shell/JbcentralQuotaIndicator.tsx:43` `jbcentral-quota`
- `web/shell/JbcentralQuotaIndicator.tsx:90` `jbcentral-quota`
- `rr/renderers/code/PierreDiff.tsx:474` `review-selection-blocked-close`
- `rr/support/ReviewThreadCard.tsx:85` `review-thread-anchor`
- `rr/support/markdownLinks.tsx:142` `markdown-file-link`
- `ext/visualize/web/FileCard.tsx:23` `tool-visualize-file-link`

### 原生窗口控件（3）

_原因：_ 平台标题栏按钮（46×40，shell/SPEC.md）。

- `web/shell/NativeWindowControls.tsx:35` `window-minimize`
- `web/shell/NativeWindowControls.tsx:44` `window-maximize`
- `web/shell/NativeWindowControls.tsx:56` `window-close`

### 评论标记钉与手柄（5）

_原因：_ 内容上的带编号圆形徽标，以及滑动对比分隔条。

- `rr/renderers/RegionReviewSurface.tsx:202` `region-comment-marker`
- `rr/renderers/VisualDiff.tsx:280`
- `rr/renderers/csv/Diff.tsx:239`
- `rr/renderers/csv/View.tsx:122`
- `rr/renderers/json/JsonTree.tsx:99`
