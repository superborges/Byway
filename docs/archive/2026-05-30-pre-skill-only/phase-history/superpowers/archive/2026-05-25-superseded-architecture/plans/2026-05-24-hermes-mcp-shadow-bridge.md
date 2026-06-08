# Phase 3.3 Hermes MCP Shadow Bridge Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把已连通的 Hermes MCP stdio bridge 升级为 Proxy 可调用工具桥，并提供默认关闭的 `/api/chat` shadow mirror。

**Architecture:** `HermesRuntimeAdapter` 增加 `callMcpTool`，Proxy 新增 Hermes MCP 工具 API。`/api/chat` 在明确配置 `HERMES_MCP_SHADOW_MODE=mirror_user_messages` 和 `HERMES_MCP_TARGET` 时旁路发送用户消息，不影响 Byway 主流程。

**Tech Stack:** TypeScript、Express、Vitest、MCP JSON-RPC stdio、SSE。

---

## Task 1: Runtime Adapter 支持 MCP Tool Call

**Files:**
- Modify: `services/proxy-api/src/hermesRuntimeAdapter.ts`
- Modify: `services/proxy-api/src/proxyApi.test.ts`

- [ ] `HermesRuntimeAdapter` 增加 `callMcpTool(name, args)`。
- [ ] Mock runtime 对 tool call 返回不可用错误。
- [ ] Local runtime 委托给 `HermesMcpClient.callTool`。

## Task 2: Proxy Hermes MCP Tool API

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] 增加 `GET /api/hermes/mcp/channels`。
- [ ] 增加 `GET /api/hermes/mcp/conversations`。
- [ ] 增加 `GET /api/hermes/mcp/conversations/:sessionKey/messages`。
- [ ] 增加 `POST /api/hermes/mcp/messages`。
- [ ] MCP text content 如果是 JSON 字符串，Proxy 解析成 `result`；原始 MCP result 保留在 `raw`。

## Task 3: `/api/chat` Shadow Mode

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] 新增 env/options：`HERMES_MCP_SHADOW_MODE`、`HERMES_MCP_TARGET`。
- [ ] mode 为 `mirror_user_messages` 且 target 存在时，调用 `messages_send`。
- [ ] 通过 SSE 输出 `tool_call_started` / `tool_call_completed`，工具名 `hermes_mcp.messages_send`。
- [ ] Shadow 失败只输出 recoverable `error` event，不中断主流程。

## Task 4: 配置与验证

**Files:**
- Modify: `.env.example`
- Modify: `.env.local`
- Modify: `README.md`

- [ ] 默认 `HERMES_MCP_SHADOW_MODE=off`。
- [ ] README 说明如何通过 `channels_list` 找 target。
- [ ] 运行：

```bash
pnpm typecheck
pnpm test
pnpm test:e2e
pnpm build
```

- [ ] 重启 `pnpm dev:all`，验证 `/api/hermes/mcp/channels` 和 `/api/hermes/mcp/probe`。
