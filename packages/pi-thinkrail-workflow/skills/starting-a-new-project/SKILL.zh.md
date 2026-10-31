# 启动一个新项目

> 中文译文，原文见 [`SKILL.md`](./SKILL.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

工作区是空的：没有代码，没有决策。把用户的想法变成一个清晰、活的产品文档 ——
`goal-and-requirements.md`。后续的特性、行为变更与缺陷修复走 `brainstorming`。

**守住 writing-specs 的质量线。** 在保存任何内容之前先读那个概念 skill —— 它承载着
短 / 诚实 / 不脱轨的规则，以及你保存的每一节都必须符合的目标文档形态。

## 方法（Method）

1. **在已经说过的内容之上构建。** 绝不重复提问请求已经告诉你的东西。
2. **先推断，再确认** —— 提出一个具体草稿让用户纠正；一个建议胜过一个开放式问题。按
   **asking-user-questions** 概念 skill 组织 `ask_user_question` 轮次（在首轮之前读它 ——
   它承载着轮次、选项、确认与降级的规范）。
3. **最小可用的首次构建。** 它比用户预期的要小；其中的每一项能力都必须能自证其必要性。
   被从中砍掉的想法交还给用户，不予保存（见 writing-specs）。
4. **增量保存。** 第一节点确定下来就创建文件，然后按模板顺序加入每一个被确认的章节。不要攒批；
   不要虚构未确认的内容。
5. 被跳过的问题不是阻塞项 —— 按当前模型继续，并把真实的缺口就地注记。

## 工作模型（Working model —— infer from the request; never ask these directly）

```
audience:   personal | public | both        domain: what space this is in
tech:       stack mentioned, or null         scope:  small | large
depth:      light | standard | full          creator_is_user: does the maker use it?
```

`depth` 缩放文档：`light` = 一句话的想法 → 几行；`full` = 点名了竞争对手 / 多种用户类型 →
一份完整的 PRD。它只能在对话过程中增长，绝不缩小。

## 快速通道 —— Fast path — pre-filled brief

如果请求本身读起来已经像一份规格（若干标题，或一份多章节的简述），解析它，把那些章节视为
**已确认**，立即保存它们，只追查真正缺失且 `depth` 要求的部分。不要请用户确认他们自己已经写下的
东西。按目标文档的形态保存，而不是按简述的形态：版本或路线图式的切分（`MVP` / `v1` / `v2` / later）
对范围内的一切变成 Capabilities，只有简述明确以决策排出的内容才进 Non-Goals；其余交还给用户、
不予保存（writing-specs）。唯一始终提供的附加项是替代方案调研（见下文）。

## 流程（Flow）

1. **定位** —— 一句话：“我们把目标和范围敲定，然后我把它保存为 `goal-and-requirements.md`。”
2. **概述** —— 推断它（按 `depth` 定尺寸：一句话 → 一段点名它取代了什么的文字）并确认。
3. **问题** —— 一个参照具体领域的定制问题（绝不用通用问题）；把答案变成一段陈述
   （谁 / 他们今天在做什么 / 具体的痛点）并确认。如果概述已经隐含了它，就跳过。
4. **按模型路由** —— 除非确实含糊，否则不问“这是给谁用的”：personal / 第一人称的痛点 /
   `depth=light` → **Personal spec**；public / 点名了用户 / `depth=full` → **PRD**。
5. **引出该分支的章节**（见下文），逐一推断并确认，边做边保存。
6. **调研替代方案**（始终提供，绝不强推）：用 `web_search` + `fetch_content` 找最接近的开源项目 /
   产品，然后提出增加一节 **Alternatives Considered**（名称、一句话差距、URL）。对一份预填的简述，
   先请求许可。
7. **评审**完整的纯 markdown 草稿并确认，然后定稿。

### 个人规格（Personal spec (sections)）

`# Title` + 一句话标语 · **Overview** · **Problem** · **Capabilities**（只留没有它这个工具就没用的
部分）· **Tech Notes**（技术栈，确定之后）。

### PRD（sections）

`# Title` + 标语 · **Overview** · **Problem Statement** · **Target Users**（角色，而非人群统计）·
**Jobs to Be Done**（“When [situation], I want [motivation], so I can [outcome]”）·
**Key User Story**（一个具体场景）· **Goals**（动词开头、可度量）· **Non-Goals** ·
**Success Metrics** / **Success Conditions** · **Capabilities**（每一项都要对着某个 Goal 或成功条件
自证）· **Non-Functional Requirements**（只有确实存在时才写）· **Technology**（Aspect | Choice |
Rationale）。

跳过模型已经回答、或 `depth` 不足以支撑的任何章节（`light` → 跳过 Goals/NFR，用二元的
Success Conditions 代替指标）。就地把含糊的目标打回：“‘更好的 UX’不是目标 ——‘首个结果在 30 秒内’
才是。”

## 保存（Saving）

- `spec_create` 一次，`path: "goal-and-requirements.md"`，一个 slug 形式的 `id`，
  `type: "goal-and-requirements"`，`title`，`status: "draft"`；用选定的模板加上目前为止已确定的
  章节替换脚手架。
- 用 `edit` 按模板顺序加入每一个被确认的章节。
- 用户批准评审过的草稿之后，用 `spec_update` 把 `status: draft → active`。

## 下一步（Next）

明确说明规格已保存，并会随项目一起演进。提出自然的下一步 —— 勾画 `architecture.md`，然后通过
`brainstorming` 构建每个特性。没有看板/工单交接 —— 说完就停：**本工作流到此结束**。
