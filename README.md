# Byway / 另辟蹊径

Byway 当前主线是 **Hermes 中的家庭旅行 Skill**。

目标：

> 用户不用管理旅行计划，只要在 Hermes 里自然表达旅行需求和现场变化；Skill 帮 Hermes 维护一份简洁活计划，并给出下一步建议。

## 当前有效内容

1. `AGENTS.md`：项目最高约束和克制原则。
2. `skills/byway-family-travel-assistant/SKILL.md`：当前唯一产品主线。
3. `skills/byway-family-travel-assistant/references/sops.md`：Skill SOP。
4. `docs/BYWAY_BOUNDARIES_AND_QUALITY_BAR.md`：功能边界与好坏标准。
5. `docs/BYWAY_SKILL_EVAL.md`：Skill 评测方法和通过线。
6. `docs/evals/byway_skill_golden_cases.md`：固定回归评测用例。
7. `docs/README.md`：当前文档索引。

## 已清理内容

旧的 TypeScript App / Backend / MCP / schema 工程代码已经移除。

旧产品和工程文档归档在：

```text
docs/archive/2026-05-30-pre-skill-only/
```

归档内容只作为历史参考，不再作为当前实现依据。

## 当前保留的产品原语

- `旅行上下文`：目的地、天数、同行人、节奏、硬约束、当前状态。
- `活的旅行计划`：一份可被自然语言持续修改的简洁计划。
- `事实校验边界`：地点、路线、营业时间等事实不能由 LLM 编造。

## 下一阶段

先继续打磨 Skill，不写复杂系统：

1. 用 Golden Cases 压测 Skill。
2. 调整 SOP 和回复策略。
3. 明确最小上下文格式。
4. 再决定是否需要极少量工具支持。
