# Byway / 另辟蹊径 — Codex 项目总指令

当前主线：**Byway 是 Hermes 中的家庭旅行 Skill**。

目标：用户自然表达旅行需求和现场变化；Hermes 在 Byway Skill 约束下维护简洁活计划，并给出下一步建议。

## 当前依据

1. `skills/byway-family-travel-assistant/SKILL.md`
2. `docs/BYWAY_BOUNDARIES_AND_QUALITY_BAR.md`
3. `docs/README.md`

归档资料在 `docs/archive/2026-05-30-pre-skill-only/`，只查历史，不作为当前实现依据。

## 产品边界

当前不做独立旅行管理 App、攻略平台、OTA、地图导航或复杂后台。

默认不要扩展：

- 独立 Web 工作台。
- 多 Artifact 面板。
- 显式 Dream / Plan / Travel / Review 模式。
- 用户可见 Trace、MCP、ToolGateway、schema、phase 等内部概念。
- PIN 登录、后台配置、复杂上传系统。
- 大型契约文档和阶段计划。

## 克制原则

新增能力必须同时满足：

- 让用户更少思考。
- 让对话更自然。
- 真实家庭旅行中马上会用到。

不满足就先不做。

## 保留原语

- `旅行上下文`：目的地、天数、同行人、节奏、硬约束、当前状态。
- `活的旅行计划`：可被自然语言持续修改的简洁计划。
- `事实校验边界`：地点、路线、营业时间等事实不能由 LLM 编造。

## Skill 底线

- 用自然语言接住用户。
- 优先回答“下一步做什么”。
- 修改计划时沿用当前旅行上下文，不重开默认城市或模板。
- 高影响变化先确认，再视为生效。
- 地点、地址、坐标、营业时间、路线耗时等事实不编造；没有可靠来源就明确说明。

设计文档和实现计划都用中文写。`AGENTS.md` 只放未来行动规则，不放复盘或阶段流水账。
