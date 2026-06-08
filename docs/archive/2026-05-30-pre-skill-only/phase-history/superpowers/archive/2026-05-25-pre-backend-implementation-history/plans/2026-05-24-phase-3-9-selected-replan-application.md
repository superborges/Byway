# Phase 3.9 选中重排方案真实应用 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `apply_replan` 按用户实际选择的 replan option 更新 PlanArtifact。

**Architecture:** MCP server 从保存的 ReplanOptionsArtifact 查找 option，然后对当前 PlanArtifact 的目标 day 节点做最小状态变更，生成新版本 PlanArtifact 和 TodayArtifact。

**Tech Stack:** TypeScript、Vitest、现有 shared schema。

---

## Task 1: 红灯测试

**Files:**
- Modify: `services/byway-mcp-server/src/bywayMcpServer.test.ts`

- [ ] 生成 replan options。
- [ ] 应用 `opt_postpone`。
- [ ] 断言 `day2_activity_low` 状态为 `postponed`，而不是 `skipped`。
- [ ] 断言 `changeSummary` 包含所选 option 的 summary。

## Task 2: MCP 实现

**Files:**
- Modify: `services/byway-mcp-server/src/index.ts`

- [ ] 增加 `findReplanOption` helper。
- [ ] 增加 `applyOptionToPlan` helper。
- [ ] `apply_replan` 找不到 option 时返回 `REPLAN_OPTION_NOT_FOUND`。
- [ ] 生成新 PlanArtifact 版本并更新 `currentPlanId`。

## 验收

- `pnpm --filter @byway/mcp-server test`
- `pnpm test:api`
- `pnpm typecheck`
