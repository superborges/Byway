# Phase 3.6 Pending Confirmation 恢复 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Proxy 重启后，持久化的 pending confirmation 仍能被用户确认并执行。

**Architecture:** 在 Proxy 的 confirmation respond route 中增加 fallback resolver，从 MCP 持久化 Trip state 搜索 pending confirmation，并恢复内部执行上下文。MCP `replan_today` 的新 confirmation payload 补充 `dayIndex`。

**Tech Stack:** TypeScript、Vitest/Supertest、现有 MCP JSON store。

---

## Task 1: 红灯测试

**Files:**
- Modify: `services/proxy-api/src/proxyApi.test.ts`

- [ ] 用同一 `storagePath` 创建第一个 app。
- [ ] 通过 chat 生成 travel replan pending confirmation。
- [ ] 用同一 `storagePath` 创建第二个 app 模拟 Proxy 重启。
- [ ] 直接 POST 原 confirmation id，期望 200 且应用 replan。

## Task 2: MCP Payload 补强

**Files:**
- Modify: `services/byway-mcp-server/src/index.ts`
- Test: `services/byway-mcp-server/src/bywayMcpServer.test.ts`

- [ ] `replan_today` 创建 confirmation 时把 `dayIndex` 写入 payload。

## Task 3: Proxy Fallback Resolver

**Files:**
- Modify: `services/proxy-api/src/index.ts`

- [ ] 增加 `confirmationIndexFromPending`。
- [ ] 增加 `resolveConfirmationIndex`：先查内存 map，再查 MCP persisted state。
- [ ] confirmation route 改用 resolver。

## 验收

- `pnpm test:api`
- `pnpm --filter @byway/mcp-server test`
- `pnpm typecheck`
