# 2026-05-25 Pre-Backend Implementation History Archive

本目录归档 Phase 4.0 一体化 Backend 之前的阶段性 specs 和 plans。

这些文档记录了 Byway 从 mock 闭环、AI 原生工作区、Travel Mode、PlaceFact、持久化、上传、确认、Memory 等方向一路演进的实现历史。它们仍可作为迁移业务语义和验收背景的参考，但不能作为当前文件路径、服务拆分或开发计划的依据。

当前开发依据是：

- `docs/superpowers/specs/2026-05-24-phase-4-hybrid-agent-runtime-design.md`
- `docs/superpowers/plans/2026-05-24-phase-4-hybrid-agent-runtime.md`
- `docs/superpowers/specs/2026-05-24-byway-hermes-skill-strategy-design.md`

## 为什么归档

这些历史文档多数仍使用旧的实现口径：

- `services/proxy-api`
- `services/byway-mcp-server`
- Proxy 规则 orchestrator
- MCP mock server
- Phase 1-3 的临时 mock / 去 mock 步骤

Phase 4.0 后，目标架构已经变成：

```text
Web -> Backend HTTP -> Backend Agent -> ToolGateway -> Domain / DB / Providers
Hermes -> Backend HTTP / Backend MCP Adapter -> ToolGateway -> 同一套 Byway 状态
```

因此，开发新功能时不要直接照搬本目录里的文件路径、服务边界或执行顺序。

## 使用方式

可以读取这些文档来理解：

- 某个验收场景为什么存在。
- 某个 artifact 或工具行为最初想解决什么问题。
- 旧代码迁移到 `services/backend` 时需要保留哪些用户语义。

不应该读取这些文档来决定：

- 新文件应该放在哪个服务。
- `/api/chat` 应该由谁编排。
- Hermes 是否作为 Web 主链路 runtime。
- MCP 是否应该作为长期独立服务运行。
