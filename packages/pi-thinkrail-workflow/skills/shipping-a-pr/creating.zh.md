# creating.md —— 先过门禁，再建 PR

> 中文译文，原文见 [`creating.md`](./creating.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

入口：分支上已完成的工作，尚无 PR。在 `.thinkrail/context/pr-body.md` 保存正文草稿。创建默认走
快照模式；明确的远端完成请求会选择等待模式。控制权随后带着选定的模式交给 `screenshots.md`
（UI 可见的改动）或 `checks.md`。

## 门禁 —— Gates — all five pass before `gh pr create`, in this order

1. **已提交、工作树干净。** 所有要交付的内容都已提交 —— `git status --porcelain` 返回空。
   git 未持有的工作（未提交的编辑、未跟踪的文件）不会到达 PR，而下一个门禁里的 rebase 反正也需要
   一棵干净的树。
2. **新鲜的基线。** `git fetch origin`，然后 rebase 到基线分支（默认 `origin/main`）。冲突现在就
   解决，而不是等评审开始之后。
3. **干净的分支。** 移除一次性产物 —— 复现测试、捕获规格、临时文件、测试输出目录。像评审者那样读
   `git log --oneline <base>..HEAD` 与 `git status --short`：diff 里的每个文件都必须能一行解释清楚。
4. **验证证据。** 应用主干的最终树规则：收集开发期间已经跑过的确切检查，复用覆盖输入未变的结果，
   只运行缺失或已失效的项目门禁。不要仅仅因为 PR 创建开始了就重跑一条已通过的命令。仅提交 ID 变化
   不会使证据失效；结果树中相关的改动才会。把确切的命令与结果报告在正文的 Testing 一节；
   没有门禁的项目靠手工验证，并如实报告这一点，绝不默默当作已验证。
5. **自审。** 以评审者的身份重读完整 diff（`git diff <base>...HEAD` 加上工作树），守住项目的
   交接卫生标准：没有默默的 lint/类型抑制、没有注释蔓延、没有半迁移的模式、没有残留物。
   发现什么就修掉；不要给它加注记。

红旗 —— 停下，某个门禁正在被合理化掉：

- “创建 PR 就意味着什么都要重跑” —— 有效的最终变更证据是可以复用的。
- “那个早先的结果大概仍然适用” —— 后来的改动可能已经让它失效。
- “rebase 可以等到评审开始再说。”
- “那个文件大概没问题” —— 你已经无法向评审者一行解释它。

## PR 本身（The PR）

- **标题**：`scope: imperative summary` —— 例如 `feat(web): …`、`fix(website): …`、`ci: …`。
- **正文** → `.thinkrail/context/pr-body.md`。在起草的那一刻读取项目的
  `.github/PULL_REQUEST_TEMPLATE.md`，保留它的章节、顺序与每一个检查清单条目；把提示语替换成
  具体内容，只勾选真正完成的检查。只有当这个 PR 不关闭任何东西时，才删掉相关 issue 一节。
  如果项目没有模板，就用三段式的 PR 正文形态 —— `## Summary`（做了什么以及为什么，issue 驱动时
  加 `Closes #NNN`）、`## Changes`（重要时包括排除项或迁移步骤），以及 `## Testing`（确切的命令与
  结果，绝不用一句光秃秃的 "tests pass"）。**当一份聊天 TODO 计划驱动了这项工作，不要凭记忆写
  正文 —— 从计划里取素材：** `plan.summary` 已经以同样的三段形态承载了跨领域的意图与测试
  （见 `todos` skill —— 计划摘要），每一步各自带有 `summary` + `verification` + `commitSubject`，
  而应用的 PR 预览正文已经渲染出整个计划。有仓库模板时，把计划素材重新组织进模板的章节，
  而不是复述它；没有模板时，计划摘要*就是*正文形态 —— 原样采用它，让自动生成的 `## Plan` 一节
  承载逐步的细节。
- **先稳定树，再推送，然后创建。** 门禁 3–5 可能编辑门禁 1 检查过的那棵树。当它们这样做时，
  提交所有内容，然后针对新的 head 重复门禁 3–5；只有某一趟不再产生进一步编辑、且
  `git status --porcelain` *此刻*为空之后才推送（主干的行动点规则）。然后推送这枚已评审、
  已验证最终变更的 head —— `gh pr create --head` **不会**替你推送：`git push -u origin <branch>`
  （远端分支已存在且曾被 rebase 时用 `--force-with-lease`）—— 然后
  `gh pr create --base <base> --head <branch> --title "…" --body-file .thinkrail/context/pr-body.md`
  —— 远端不明确时加 `--repo`；只有用户要求草稿时才加 `--draft`。绝不内联传正文：长的内联/heredoc
  正文曾发生截断而失败；正文文件*就是*配方。PR 存在之后删除正文文件。

## 下一步（Next）

- 改动是 UI 可见的 → 主动提出截图（不要等着被要求），然后阅读并遵循 `screenshots.md`，
  带上选定的完成模式。
- 否则 → 按选定的完成模式阅读并遵循 `checks.md`。
