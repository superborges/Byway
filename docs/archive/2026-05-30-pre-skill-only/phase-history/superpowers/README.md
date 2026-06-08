# Byway Superpowers 文档索引

## 当前执行依据

后续开发应优先引用以下文档：

- `docs/superpowers/specs/2026-05-24-phase-4-hybrid-agent-runtime-design.md`
  - 当前 Phase 4.0 一体化 Backend Hybrid Agent 架构设计。
- `docs/superpowers/plans/2026-05-24-phase-4-hybrid-agent-runtime.md`
  - 当前 Phase 4.0 一体化 Backend 实施计划。
- `docs/superpowers/specs/2026-05-24-byway-hermes-skill-strategy-design.md`
  - Hermes Byway Skill 与 Backend Agent / Runtime Skills 的分工策略。
- `docs/superpowers/specs/2026-05-25-phase-4-implementation-status.md`
  - Phase 4.0 当前实现状态、验收方式和已完成链路。

产品需求、契约和数据模型仍以 `docs/BYWAY_*.md` 为准。

## 归档规则

`docs/superpowers/archive/` 下的文档只保留为决策历史，不能作为当前开发依据。

如果归档文档与当前执行依据冲突，以当前执行依据为准。尤其是涉及这些关键词的归档内容需要谨慎：

- Hermes-first 主编排。
- `services/proxy-api` 作为长期主 backend。
- `services/byway-mcp-server` 作为长期独立 MCP 服务。
- `services/agent-runtime` 作为独立 Agent Runtime 服务。
- `BYWAY_AGENT_RUNTIME=hermes_agent`。
- Hermes MCP shadow bridge。

当前方向是：

```text
Web -> Backend HTTP -> Backend Agent -> ToolGateway -> Domain / DB / Providers
Hermes -> Backend HTTP / Backend MCP Adapter -> ToolGateway -> 同一套 Byway 状态
```
