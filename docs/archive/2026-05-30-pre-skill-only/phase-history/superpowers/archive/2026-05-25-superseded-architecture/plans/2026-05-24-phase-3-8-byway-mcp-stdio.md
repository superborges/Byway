# Phase 3.8 Byway MCP Stdio Server Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 提供可被 Hermes 连接的 Byway MCP stdio server。

**Architecture:** 新增纯 TypeScript MCP JSON-RPC handler，复用 `BywayMcpServer` class；stdio server 只负责按行读取 JSON-RPC、调用 handler、逐行输出 response。

**Tech Stack:** TypeScript、Node readline/stdin/stdout、Vitest。

---

## Task 1: 红灯测试

**Files:**
- Create: `services/byway-mcp-server/src/mcpProtocol.test.ts`

- [ ] 测试 `initialize` 返回 serverInfo 和 tools capability。
- [ ] 测试 `tools/list` 包含核心工具名。
- [ ] 测试 `tools/call create_trip` 返回 MCP text content，解析后为 ToolResult。
- [ ] 测试未知工具返回 JSON-RPC error。

## Task 2: MCP Protocol Handler

**Files:**
- Create: `services/byway-mcp-server/src/mcpProtocol.ts`

- [ ] 定义 `BYWAY_MCP_TOOLS`。
- [ ] 实现 `createBywayMcpJsonRpcHandler(server)`。
- [ ] `tools/call` 动态分发到 `BywayMcpServer` 实例方法。

## Task 3: Stdio Entrypoint

**Files:**
- Create: `services/byway-mcp-server/src/stdioServer.ts`
- Modify: `services/byway-mcp-server/package.json`

- [ ] 用 readline 按行读取 stdin。
- [ ] 对 notification 不输出 response。
- [ ] 增加 `mcp` script：`tsx src/stdioServer.ts`。

## Task 4: 文档

**Files:**
- Modify: `README.md`

- [ ] 说明 Hermes 可用 `pnpm --filter @byway/mcp-server mcp` 连接 Byway MCP tools。

## 验收

- `pnpm --filter @byway/mcp-server test`
- `pnpm typecheck`
- `pnpm test`
- `pnpm build`
