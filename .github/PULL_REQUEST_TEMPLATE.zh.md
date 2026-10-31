# 拉取请求模板

> 中文译文，原文见 [`PULL_REQUEST_TEMPLATE.md`](./PULL_REQUEST_TEMPLATE.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 问题（Problem）

本 PR 解决什么问题？描述对用户或开发者的影响，以及为什么需要这项改动。

## 方案（Approach）

该方案如何解决这个问题？在评审者可理解的层面解释重要决策与权衡；避免逐文件的实现流水账。

## 改动（Changes）

列出评审者应当检查的具体行为、契约或模块改动。

## 截图（Screenshots）

前端改动必须提供：同一场景、同一视口下的改动前后截图。否则写
`Not applicable — no frontend changes.`

## 相关 issue（Related issues）

Closes #...

## 检查清单（Checklist）

- [ ] 快速门禁通过：`bun run lint`、`bun run typecheck`、`bun run test`
- [ ] 影响应用的改动通过 E2E 套件（`bun run e2e`；涉及 agent 行为时用 `bun run e2e:full`）
- [ ] 前端改动附有改动前后截图，或标记为不适用
- [ ] 相关的 `SPEC.md` / 顶层规格已更新，以反映任何边界、契约或行为改动
- [ ] 我已阅读[贡献指南](../CONTRIBUTING.md)并同意遵守[行为准则](../CODE_OF_CONDUCT.md)
