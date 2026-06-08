# 边界安全与失败降级 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 补齐 G1/G2/G3/G5 的边界安全行为。

**Architecture:** Proxy 增加安全边界 intent，MCP `resolve_places_batch` 增加 mock failure 分支。现有 Plan Mode 已支持 assumptions，本阶段只加测试保证不过度追问。

**Tech Stack:** TypeScript、Express、Vitest、Playwright。

---

## 文件结构

- 修改：`services/proxy-api/src/index.ts`
  增加 `booking_request`、`comment_scrape_request` intent。

- 修改：`services/proxy-api/src/proxyApi.test.ts`
  增加 G1/G2/G5 测试。

- 修改：`services/byway-mcp-server/src/index.ts`
  `resolve_places_batch` 增加 mock 高德失败。

- 修改：`services/byway-mcp-server/src/bywayMcpServer.test.ts`
  增加 G3 测试。

- 修改：`apps/web/e2e/byway.spec.ts`
  增加边界安全 E2E。

## 任务 1：红测

- [ ] Proxy 测试 G1/G2/G5。
- [ ] MCP 测试 G3。
- [ ] E2E 测试 G1/G2。

## 任务 2：实现

- [ ] `detectMode` 在 plan/travel 前识别 `booking_request` 和 `comment_scrape_request`。
- [ ] 两个 branch 只返回安全回复，不调用 booking/scraping。
- [ ] `resolve_places_batch` 遇到 `city.includes("高德失败")` 返回 `fail("AMAP_UNAVAILABLE", "...", true)`。

## 任务 3：验证

- [ ] `pnpm typecheck`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] `pnpm build`
- [ ] 重启 `pnpm dev:all`
