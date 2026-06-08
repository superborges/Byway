# 2026-05-25 Superseded Architecture Archive

本目录归档已经被 Phase 4.0 一体化 Backend 架构取代的中间讨论和实施计划。

这些文档保留为决策历史，不能作为当前开发依据。当前执行依据是：

- `docs/superpowers/specs/2026-05-24-phase-4-hybrid-agent-runtime-design.md`
- `docs/superpowers/plans/2026-05-24-phase-4-hybrid-agent-runtime.md`
- `docs/superpowers/specs/2026-05-24-byway-hermes-skill-strategy-design.md`

## 归档原因

归档文档主要属于以下旧路线：

- Hermes-first `/api/chat` 主编排。
- Hermes MCP shadow bridge。
- 独立 `services/byway-mcp-server` stdio server 作为长期接入形态。
- Hermes 本机注册独立 Byway MCP server。
- `services/proxy-api` + `services/byway-mcp-server` 的旧去 mock roadmap。

当前方向是单个 `services/backend`：

```text
Web -> Backend HTTP -> Backend Agent -> ToolGateway -> Domain / DB / Providers
Hermes -> Backend HTTP / Backend MCP Adapter -> ToolGateway -> 同一套 Byway 状态
```

## 文件清单

### Specs

- `specs/2026-05-24-phase-4-agent-runtime-orchestration-design.md`
- `specs/2026-05-24-hermes-mcp-shadow-bridge-design.md`
- `specs/2026-05-24-phase-3-8-byway-mcp-stdio-design.md`
- `specs/2026-05-24-phase-3-11-hermes-byway-mcp-registration-design.md`
- `specs/2026-05-24-production-readiness-roadmap-design.md`
- `specs/2026-05-24-demock-amap-hermes-foundation-design.md`

### Plans

- `plans/2026-05-24-phase-4-agent-runtime-orchestration.md`
- `plans/2026-05-24-hermes-mcp-shadow-bridge.md`
- `plans/2026-05-24-phase-3-8-byway-mcp-stdio.md`
- `plans/2026-05-24-phase-3-11-hermes-byway-mcp-registration.md`
- `plans/2026-05-24-demock-amap-hermes-foundation.md`
