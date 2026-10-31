# design-audit —— 按钮系统目录（预览工具，不随应用发布）

> 中文译文，原文见 [`README.md`](./README.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

`packages/ui/button.tsx` 中共享按钮系统的预览工具（[[module-ui]] "Button system"）。`apps/`、
`packages/` 或 `thinkrail-extensions/` 中没有任何代码导入它，生产环境的 Tailwind 也从不扫描它。

| 路径 | 说明 |
| --- | --- |
| `buttons.html` | **单文件自包含的按钮目录**——直接打开即可，无需服务器或网络。它用生产样式表渲染生产环境的 `Button` / `IconButton`，并带主题切换器，包含：60 个样本的矩阵（5 个变体 × 3 个尺寸 × 4 种内容配置）；按变体与尺寸标注的状态（默认、悬停、按下、focus-visible、禁用、加载中，以及存在该状态的选中态）；一个「按下 vs 选中」章节，含应用中的持久选中项与真实的面板工具轨（条目为模拟数据）；用于真实悬停、按下与键盘聚焦的实时按钮；用于真实 `ToastClose` 聚焦的实时生产 toast；每个样本下测量出的尺寸、内边距、间隙、图标与对比度；以及范例的改动前后裁剪图（`data/buttons-before/`）。 |
| `buttons/` | 目录源码。外壳只使用 `bk-*` 类，从不针对样本标记。 |
| `catalog/pseudo.ts` | 强制状态夹具：每条带 `:hover`、`:active` 或 `:focus-visible` 的生产规则都被复制到一个标记类上，因此无需指针也能渲染出这些状态。使用它的样本标注为 "forced"。 |
| `buttons-migration.md` | 迁移记录：每个调用点的旧 → 新变体与尺寸、被移到 `soft-primary` 的 accent-text 动作、尺寸例外，以及每个仍然保留的裸 `<button>` 及其不能成为共享按钮的原因。 |
| `tooling/` | `build-buttons.ts` 构建该目录（`--serve` 提供实时预览）；`verify-buttons.ts` 在全部六个主题中加载它并检查几何尺寸、聚焦与标签对比度 → `data/verification/buttons/summary.json`。 |

```sh
bun run build:web                                  # production apps/web/dist — the catalog uses its CSS verbatim
bun design-audit/tooling/build-buttons.ts [--serve] # buttons.html, or a live preview at http://localhost:24390/
bun design-audit/tooling/verify-buttons.ts         # all 6 themes (Node ≥ 22 on PATH for Playwright)
```
