# Phase 3.11 Hermes 注册 Byway MCP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 本机 Hermes 可以通过 MCP stdio 连接 Byway MCP Tools。

**Architecture:** 用一个仓库内 shell wrapper 隔离 pnpm stdout、加载本地 env，并注册给 Hermes MCP 配置。Byway MCP server 自身保持纯 JSON-RPC stdio。

**Tech Stack:** Bash、pnpm、Hermes CLI、TypeScript stdio server。

---

## Task 1: 启动脚本

**Files:**
- Create: `scripts/run-byway-mcp.sh`
- Modify: `services/byway-mcp-server/src/stdioServer.ts`

- [ ] 新增 wrapper script，加载 `.env.local` 并用 `pnpm --silent` 启动。
- [ ] `stdioServer.ts` 捕获 stdout EPIPE。
- [ ] 用 JSON-RPC initialize/tools/list 手动验证 stdout 无 banner。

## Task 2: Hermes 注册和测试

**Commands:**
- `hermes mcp add byway --command bash --args /Users/qinkan/Documents/codex/Byway/scripts/run-byway-mcp.sh`
- `hermes mcp test byway`

- [ ] 如果已存在 byway，先读取当前配置，不盲目覆盖。
- [ ] 测试返回 Byway 工具列表。

## Task 3: 文档

**Files:**
- Modify: `README.md`

- [ ] 写明 Hermes 注册命令。
- [ ] 写明 stdout 必须纯 JSON-RPC，不要直接用非 silent pnpm。

## 验收

- `hermes mcp list`
- `hermes mcp test byway`
- `pnpm --filter @byway/mcp-server test`
- `pnpm typecheck`
