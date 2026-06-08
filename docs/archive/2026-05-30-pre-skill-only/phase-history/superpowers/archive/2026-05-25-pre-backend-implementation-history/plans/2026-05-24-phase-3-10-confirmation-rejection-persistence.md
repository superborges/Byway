# Phase 3.10 Confirmation 拒绝状态持久化 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** confirmation 被拒绝后，MCP 持久化状态不再保留 pending。

**Architecture:** MCP 增加 `respond_pending_confirmation`，Proxy rejection 分支根据 confirmation kind 调用相应 MCP 工具更新状态。

**Tech Stack:** TypeScript、Vitest/Supertest。

---

## Task 1: 红灯测试

**Files:**
- Modify: `services/proxy-api/src/proxyApi.test.ts`

- [ ] 生成 replan pending confirmation。
- [ ] POST decision `reject`。
- [ ] GET trip state，断言该 confirmation 不再是 pending。

## Task 2: MCP 工具

**Files:**
- Modify: `services/byway-mcp-server/src/index.ts`
- Modify: `services/byway-mcp-server/src/mcpProtocol.ts`

- [ ] 实现 `respond_pending_confirmation`。
- [ ] 加入 MCP tools list。

## Task 3: Proxy Rejection

**Files:**
- Modify: `services/proxy-api/src/index.ts`

- [ ] apply_replan rejection 调用 `respond_pending_confirmation`。
- [ ] accept_memory rejection 调用 `accept_family_memory(accepted=false)`。

## 验收

- `pnpm test:api`
- `pnpm --filter @byway/mcp-server test`
- `pnpm typecheck`
