# Phase 3.13 Accepted Family Memory 注入 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:test-driven-development. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** accepted Family Memory 参与后续 Dream / Plan，不再只是可查看的历史数据。

**Architecture:** Proxy 增加 memory loading helper，通过 SSE tool call 暴露 `get_accepted_family_memory`；MCP 在候选和计划生成中根据 accepted memory ids 做稳定调整。

**Tech Stack:** TypeScript、Express SSE、Byway MCP mock server、Vitest/Supertest。

---

## Task 1: 红灯测试

**Files:**
- Modify: `services/proxy-api/src/proxyApi.test.ts`
- Modify: `services/byway-mcp-server/src/bywayMcpServer.test.ts`

- [x] Proxy 测试：接受 memory 后发起新 Dream，断言调用 `get_accepted_family_memory`。
- [x] Proxy 测试：Dream 工具调用参数包含 accepted memory id。
- [x] Proxy 测试：新 Plan 的 `generate_plan` 参数包含 accepted memory id，PlanArtifact 含家庭记忆提示。
- [x] MCP 测试：只有 accepted memory 会影响候选和计划。

## Task 2: Proxy Memory 注入

**Files:**
- Modify: `services/proxy-api/src/index.ts`

- [x] 增加 `loadAcceptedFamilyMemory` helper。
- [x] Dream / Dream refine 调用 helper 并传入 candidate tools。
- [x] Destination select / Plan 调用 helper 并传入 `generate_plan`。
- [x] 有 memory 时输出简短 assistant 提示。

## Task 3: MCP Memory 应用

**Files:**
- Modify: `services/byway-mcp-server/src/index.ts`

- [x] 增加 accepted stamina memory 判断 helper。
- [x] destination candidates 增加强度风险提示。
- [x] `generate_plan` 在 assumptions / warnings 中体现 memory。

## Task 4: 验证

- [x] `pnpm test:api`
- [x] `pnpm --filter @byway/mcp-server test`
- [x] `pnpm typecheck`
- [x] `pnpm test`
