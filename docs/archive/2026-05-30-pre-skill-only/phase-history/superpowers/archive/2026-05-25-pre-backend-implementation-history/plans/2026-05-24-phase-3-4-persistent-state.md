# Phase 3.4 持久化与恢复 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 Byway 的 Trip、Artifact、TravelEvent、PendingConfirmation、FamilyMemory 在服务重启后仍可恢复。

**Architecture:** 继续沿用当前 `sql.js` 单表 JSON store，不引入新的数据库运行时。`BywayMcpServer.create({ storagePath })` 从文件加载数据库，并在每次写入后导出到本地 SQLite 文件；Proxy 的 `/api/trips` 改为调用 MCP `list_trips`，不再依赖仅存在内存里的 `knownTripIds`。

**Tech Stack:** TypeScript、sql.js、Node fs/path、Vitest、Supertest。

---

## Task 1: MCP 文件型 SQLite Store

**Files:**
- Modify: `services/byway-mcp-server/src/index.ts`
- Test: `services/byway-mcp-server/src/bywayMcpServer.test.ts`

- [ ] 写失败测试：用临时 `storagePath` 创建 trip 和 plan，重新创建 server 后能读取同一 trip、artifact、pending confirmation。
- [ ] 实现 `SqliteJsonStore.create({ storagePath })`：存在文件则 `readFileSync` 加载，不存在则创建新库。
- [ ] 在 `set` 后调用 `db.export()` 并 `writeFileSync` 到 `storagePath`。
- [ ] 初始化后扫描已有 record id，把全局 counter 提升到已存最大后缀，避免重启后 `trip_1` 覆盖旧数据。

## Task 2: MCP Trip 列表工具

**Files:**
- Modify: `services/byway-mcp-server/src/index.ts`
- Test: `services/byway-mcp-server/src/bywayMcpServer.test.ts`

- [ ] 写失败测试：重启后新建第二个 trip，`list_trips({ userId })` 返回两个 active trip，且 id 不重复。
- [ ] 实现 `list_trips({ userId, includeArchived })`，返回按 `updatedAt` 倒序排列的 trip summary。

## Task 3: Proxy 使用持久化 Trip 列表

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Modify: `services/proxy-api/src/server.ts`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] 写失败测试：用同一 `storagePath` 预先创建 trip，再创建新的 Proxy app，`GET /api/trips` 能列出已存在 trip。
- [ ] `/api/trips` 调用 `byway.list_trips({ userId: local_family })`，不再只看 `knownTripIds`。
- [ ] `server.ts` 使用 `BYWAY_SQLITE_PATH`，默认 `.byway/byway.sqlite`。

## Task 4: 配置与文档

**Files:**
- Modify: `.env.example`
- Modify: `.gitignore`
- Modify: `README.md`

- [ ] 增加 `BYWAY_SQLITE_PATH=.byway/byway.sqlite` 示例。
- [ ] 忽略 `.byway/`。
- [ ] README 说明 dev server 会持久化本地旅行数据，测试仍可使用内存 server。

## 验收

- `pnpm --filter @byway/mcp-server test`
- `pnpm test:api`
- `pnpm typecheck`
- `pnpm test`
- `pnpm build`
