# screenshots.md —— 给评审者的前后对比证据

> 中文译文，原文见 [`screenshots.md`](./screenshots.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

入口：一个 PR 已存在（或刚刚创建），对应一处 UI 可见的改动。把 PNG 保存到
`.thinkrail/context/pr-shots/` 下，并重写 PR 正文以引用它们。仅当调用方明确请求了远端完成时，
才保留等待模式；否则使用快照模式，包括在 PR 创建之后。

## 捕获（Capture）

- **After** = 分支构建上的该场景；**Before** = 基线分支上的*同一个*场景（分离检出、重新构建、
  捕获、返回）。同样的视口，同样的场景 —— 这一对必须能视觉上 diff 出差别。
- 用这个项目里任何能驱动应用的方式来捕获：带一份一次性捕获规格的 e2e 测试装置，或针对运行中的
  开发宿主开一个浏览器自动化会话。捕获规格是一次性的 —— 它从不交付，而它的清理归*本文档*
  负责（见下文“下一步”）：creating.md 的干净分支门禁在这里的每条进入路径上都已经跑过了，
  不会抓到它。
- 把挑中的截图暂存为 `.thinkrail/context/pr-shots/<name>.png`，命名要让评审者看得懂
  （`loader-before-gap.png`、`loader-after-working.png`）。

## 附加 —— Attach — review-only assets ref (the default)

截图绝不进入 PR 的文件树。把它们作为一枚孤儿 ref 推送，并引用钉住 SHA 的 URL：

```bash
TREE_INPUT=""
for f in .thinkrail/context/pr-shots/*.png; do
  sha=$(git hash-object -w "$f")
  TREE_INPUT+=$(printf '100644 blob %s\t%s' "$sha" "$(basename "$f")")$'\n'
done
TREE=$(printf '%s' "$TREE_INPUT" | git mktree)
COMMIT=$(printf 'PR screenshots (review-only ref)\n' | git commit-tree "$TREE")
git push -f origin "$COMMIT:refs/heads/assets/<topic>"
```

然后更新正文 —— **绝不重建它**：先取当前正文
（`gh pr view <n> --json body -q .body > .thinkrail/context/pr-body.md`），在其中添加或更新一节
`## Screenshots`，把每张图引用为
`https://github.com/<owner>/<repo>/blob/<COMMIT>/<name>.png?raw=true`（钉住所推送的提交 SHA，
这样后续的分支推送绝不会弄坏图片），然后执行
`gh pr edit <n> --body-file .thinkrail/context/pr-body.md` 并删除该文件。在这里从零构建的正文
会抹掉 PR 的摘要、测试注记与 issue 链接。

回退方案（用户有时更喜欢它）：把 PNG 留在暂存处并交出它们的路径 —— 用户自己把它们拖进 GitHub。
只有当用户已经暗示了这个选择时才提供该选项；assets ref 是默认值。

## 下一步（Next）

正文编辑落地后，清理本阶段的一次性产物：删除 `.thinkrail/context/pr-shots/`（除非用户正在手工
上传 —— 那就留着直到他们确认）以及每一处捕获脚手架（规格、一次性构建）。如果有脚手架曾被提交，
删除它的那次提交也要推送 —— 留在工作树里的删除什么也交付不了。那次推送不改变完成模式。
按选定的模式阅读并遵循 `checks.md`。
