# 应用内通知

> 中文译文，原文见 [`NOTIFICATIONS.md`](./NOTIFICATIONS.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

应用在自己的窗口内显示的每一条瞬态或内联消息都使用同一种视觉语言：toast（由 `panels/Toaster`
组合的 `packages/ui/toast.tsx`）以及内联通知，例如 `ProjectTrustNotice`、`ProviderWarningBanner`、
`SkillsDialog` / `NewWorkspaceDialog` 中的信任通知，以及聊天的 `ErrorTurn` / `CompactionNotice`。
操作系统级通知是另一回事，由 `notifications` 模块拥有。

## 弹出 toast（Raising a toast）

使用 store 辅助函数：来自 `@/store` 的 `toast.info|success|warning|error(message, title?)`。当你
需要 `action`（`{ label, onClick }`）或自定义 `durationMs` 时使用 `pushToast`。错误 toast 会一直
保留直到被关闭。其它每个 toast 都会自动消失。

## 表面（Surface）

- 中性填充：每个严重级别都用 `bg-container-elevated-bg`。不使用带色调的 `bg-feedback-*-subtle`
  填充，也没有左侧强调色条。
- 严重级别只通过 1px 边框（`border-feedback-<severity>-muted`）与图标颜色
  （`text-feedback-<severity>`）体现。正文文本保持 `text-text-default`。
- 几何：`rounded-sm`。内联通知使用 `px-12 py-8`，或在密集的侧面板（如 Resources inspector）内
  使用 `px-8 py-4`。toast 使用 `p-12`。
- 文本：消息用 `tr-text-ui text-text-default`，次级行用 `tr-text-metadata text-text-muted`。

## 图标（Icons）

每个严重级别在任何地方都使用一个 Line 字形，用匹配的 `text-feedback-*` 着色，并标记
`aria-hidden`：

| 严重级别 | 字形 |
|---|---|
| info | `RiInformationLine` |
| success | `RiCheckboxCircleLine` |
| warning | `RiAlertLine`（三角形） |
| error | `RiErrorWarningLine`（带圈的 `!`） |

含义比其严重级别更具体的通知可以使用领域字形，例如受信项目通知上的 `RiShieldCheckLine`。

## 布局（Layout）

根据内容选择两种布局之一：

- **单行**（一条短消息，没有次级行）：`flex items-center gap-8`。图标、消息（`flex-1`）与动作
  位于一行。用 `-mt-2` 微调 14px 图标，使其在视觉上与文本对齐。
- **多行**（有次级行，或消息可能换行）：`flex flex-col gap-8`。把图标与文本放在内容行
  （`flex items-start gap-8`，图标用 `mt-2` 以匹配首行）。把动作放在页脚行
  （`flex justify-end gap-8`）。绝不要把动作留在会换行的文本右侧。

toast 总是使用多行布局。`ToastClose` 位于内容行，可选的动作放在 `mt-12` 的页脚中。

## 动作（Actions）

- 使用 `size="m"` 按钮。主要动作是 `variant="outline"`。紧随其后的次要动作是 `variant="ghost"`。
- 绝不在通知内使用填充式（`primary`、`destructive`）或带颜色的按钮。边框已经承载了严重级别，
  而填充按钮会与页面的主要动作竞争。
- toast 动作通过 `ToastAction asChild` 包裹一个 `Button` 来渲染。这样 Radix 仍会关闭 toast，并
  播报动作的 `altText`。

## 尚未迁移（Not yet migrated）

少数较旧的错误表面仍使用带色调的样式，例如 `ExistingWorktreeDialog` 与 `SpecsPanel` 的加载错误。
下次触碰它们时迁移掉，并且不要把它们的样子复制到新通知中。Markdown alert callout
（`resource-renderers/support/markdownAlerts`）不是通知：它们刻意镜像 GitHub 的 alert 样式。
