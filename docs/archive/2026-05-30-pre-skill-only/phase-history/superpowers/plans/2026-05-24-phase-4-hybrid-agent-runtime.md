# Phase 4.0 一体化 Backend Hybrid Agent Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 Byway 从 `proxy-api + byway-mcp-server + 规划中的 agent-runtime` 收敛为一个一体化 `services/backend`，同时保留 Agent、Tools、MCP Adapter、Domain、Provider、Persistence、Trace 的逻辑边界。

**Architecture:** Web 调 Backend HTTP API；Backend 内部 Agent Module 负责自然语言理解、状态图、checkpoint 和人机确认；Agent 通过 in-process ToolGateway 调 Byway Tool Runtime；Hermes 通过 Backend MCP Adapter 或 HTTP API 复用同一套工具和状态。MCP 不再必须作为独立服务运行，而是 backend 内的协议适配层。

**Tech Stack:** TypeScript monorepo、Express、SSE、Zod、SQLite、Vitest、Playwright、MCP JSON-RPC adapter、LangGraph / DeepAgents-style state graph、JSONL Trace、Amap provider。

---

## 文件结构

- Create: `services/backend/package.json`
  - 新 backend 包 `@byway/backend`。
- Create: `services/backend/tsconfig.json`
  - 继承根 TypeScript 配置。
- Create: `services/backend/src/index.ts`
  - 后端启动入口。
- Create: `services/backend/src/app.ts`
  - Express app，挂载 HTTP routes、SSE、health。
- Create: `services/backend/src/http/routes.ts`
  - `/api/chat`、trips、artifacts、confirmations、uploads、traces。
- Create: `services/backend/src/http/sse.ts`
  - SSE 写入 helper。
- Create: `services/backend/src/agent/contracts.ts`
  - `AgentTurnRequest`、`AgentEvent`、`AgentRunStatus`。
- Create: `services/backend/src/agent/contextPackage.ts`
  - 构造 Agent Context Package。
- Create: `services/backend/src/agent/graph/state.ts`
  - Agent graph state 类型。
- Create: `services/backend/src/agent/graph/bywayGraph.ts`
  - 状态图编排。
- Create: `services/backend/src/agent/graph/nodes/loadContext.ts`
- Create: `services/backend/src/agent/graph/nodes/decideMode.ts`
- Create: `services/backend/src/agent/graph/nodes/planToolSteps.ts`
- Create: `services/backend/src/agent/graph/nodes/executeTools.ts`
- Create: `services/backend/src/agent/graph/nodes/confirmationGate.ts`
- Create: `services/backend/src/agent/graph/nodes/composeResponse.ts`
- Create: `services/backend/src/agent/graph/nodes/refreshArtifacts.ts`
- Create: `services/backend/src/tools/toolCatalog.ts`
  - `BYWAY_TOOLS`。
- Create: `services/backend/src/tools/toolGateway.ts`
  - Agent 和 MCP Adapter 共用的工具调用入口。
- Create: `services/backend/src/tools/toolSchemas.ts`
  - 工具输入 schema。
- Create: `services/backend/src/tools/handlers/*.ts`
  - Trip、Dream、Plan、Place、Travel、Review、Memory handlers。
- Create: `services/backend/src/mcp/mcpAdapter.ts`
  - MCP protocol 到 ToolGateway 的适配。
- Create: `services/backend/src/domain/*`
  - 纯业务模型和服务。
- Create: `services/backend/src/providers/*`
  - Amap、uploads、mock provider。
- Create: `services/backend/src/persistence/sqliteStore.ts`
  - SQLite store。
- Create: `services/backend/src/persistence/checkpointStore.ts`
  - Agent checkpoint。
- Create: `services/backend/src/observability/traceTypes.ts`
- Create: `services/backend/src/observability/redaction.ts`
- Create: `services/backend/src/observability/traceStore.ts`
- Create: `services/backend/src/observability/logger.ts`
- Create: `skills/byway-runtime/dream.md`
- Create: `skills/byway-runtime/plan.md`
- Create: `skills/byway-runtime/plan-edit.md`
- Create: `skills/byway-runtime/travel.md`
- Create: `skills/byway-runtime/review-memory.md`
- Create: `skills/byway-runtime/fact-and-confirmation-boundaries.md`
- Modify: `skills/byway-family-travel-agent/SKILL.md`
  - 改为 Hermes 家庭入口 Skill。
- Modify: `apps/web/src/app/page.tsx`
  - 接入 backend 新 SSE 事件和 trace/runId。
- Modify: `apps/web/src/app/components/TripWorkspace.tsx`
  - 展示 trace 入口。
- Modify: `.env.example`
  - 更新 backend、Agent mode、Trace、Hermes、Amap 配置。
- Modify: `package.json`
  - `dev:all` 改成 backend + web。
- Modify: `README.md`
  - 记录一体化 backend 架构和验收方式。
- Later remove or deprecate: `services/proxy-api`
- Later remove or deprecate: `services/byway-mcp-server`
  - 第一轮迁移可以保留旧目录不删除，避免大爆炸；所有新开发以 `services/backend` 为准。

---

## Task 1: 建立 backend 包和 Agent 契约

**Files:**
- Create: `services/backend/package.json`
- Create: `services/backend/tsconfig.json`
- Create: `services/backend/src/agent/contracts.ts`
- Test: `services/backend/src/agent/contracts.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/backend/src/agent/contracts.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { agentEventSchema, agentTurnRequestSchema } from "./contracts";

describe("backend agent contracts", () => {
  it("validates a chat turn request", () => {
    const parsed = agentTurnRequestSchema.parse({
      traceId: "trace_1",
      requestId: "req_1",
      turnId: "turn_1",
      userId: "local_family",
      tripId: "trip_1",
      text: "第三天可以改成灵隐寺吗？",
      attachments: [],
      clientContext: { timezone: "Asia/Shanghai" }
    });

    expect(parsed.tripId).toBe("trip_1");
  });

  it("validates streamable agent events", () => {
    expect(agentEventSchema.parse({
      type: "mode_decided",
      mode: "plan_edit",
      reasonSummary: "用户引用第三天并要求修改当前计划。"
    })).toMatchObject({ type: "mode_decided", mode: "plan_edit" });

    expect(agentEventSchema.parse({
      type: "confirmation_required",
      confirmationId: "conf_1",
      summary: "应用重排方案前需要确认。"
    }).type).toBe("confirmation_required");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/agent/contracts.test.ts
```

Expected: FAIL，提示找不到 `services/backend` 或 `./contracts`。

- [ ] **Step 3: 创建 backend package**

创建 `services/backend/package.json`：

```json
{
  "name": "@byway/backend",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "dev": "tsx watch src/index.ts",
    "test": "vitest run src/**/*.test.ts",
    "typecheck": "tsc -p tsconfig.json --noEmit"
  },
  "dependencies": {
    "@byway/shared-schemas": "workspace:*",
    "better-sqlite3": "^11.7.0",
    "cors": "^2.8.5",
    "express": "^5.0.1",
    "zod": "^3.24.1"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.12",
    "@types/cors": "^2.8.17",
    "@types/express": "^5.0.0",
    "@types/node": "^22.10.2",
    "@types/supertest": "^6.0.2",
    "supertest": "^7.0.0",
    "tsx": "^4.19.2",
    "typescript": "^5.7.2",
    "vitest": "^2.1.8"
  }
}
```

创建 `services/backend/tsconfig.json`：

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src/**/*.ts"]
}
```

- [ ] **Step 4: 实现 contracts**

创建 `services/backend/src/agent/contracts.ts`：

```ts
import { z } from "zod";

export const agentTurnRequestSchema = z.object({
  traceId: z.string().min(1),
  requestId: z.string().min(1),
  turnId: z.string().min(1),
  userId: z.string().min(1),
  tripId: z.string().optional(),
  text: z.string().min(1),
  attachments: z.array(z.object({
    fileId: z.string().optional(),
    filename: z.string().optional(),
    mimeType: z.string().optional(),
    textPreview: z.string().optional()
  })).default([]),
  clientContext: z.record(z.unknown()).default({})
});

export type AgentTurnRequest = z.infer<typeof agentTurnRequestSchema>;

export const agentEventSchema = z.discriminatedUnion("type", [
  z.object({ type: z.literal("run_started"), runId: z.string(), threadId: z.string(), traceId: z.string() }),
  z.object({ type: z.literal("mode_decided"), mode: z.string(), reasonSummary: z.string() }),
  z.object({ type: z.literal("subtask_started"), name: z.string(), label: z.string() }),
  z.object({ type: z.literal("subtask_completed"), name: z.string(), label: z.string() }),
  z.object({ type: z.literal("tool_call_started"), toolName: z.string(), callId: z.string(), inputSummary: z.unknown().optional() }),
  z.object({ type: z.literal("tool_call_completed"), toolName: z.string(), callId: z.string(), outputSummary: z.unknown().optional() }),
  z.object({ type: z.literal("artifact_updated"), artifactType: z.string(), artifactId: z.string(), version: z.number().int().positive() }),
  z.object({ type: z.literal("confirmation_required"), confirmationId: z.string(), summary: z.string() }),
  z.object({ type: z.literal("assistant_message_delta"), text: z.string() }),
  z.object({ type: z.literal("run_completed"), runId: z.string() }),
  z.object({ type: z.literal("run_failed"), code: z.string(), message: z.string(), recoverable: z.boolean() })
]);

export type AgentEvent = z.infer<typeof agentEventSchema>;
```

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/agent/contracts.test.ts
```

Expected: PASS。

---

## Task 2: 建立 ToolGateway 和工具目录

**Files:**
- Create: `services/backend/src/tools/toolCatalog.ts`
- Create: `services/backend/src/tools/toolGateway.ts`
- Test: `services/backend/src/tools/toolGateway.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/backend/src/tools/toolGateway.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { createToolGateway } from "./toolGateway";

describe("ToolGateway", () => {
  it("exposes canonical Byway tools through one in-process gateway", async () => {
    const gateway = createToolGateway();
    const names = gateway.listTools().map((tool) => tool.name);

    expect(names).toEqual(expect.arrayContaining([
      "get_trip_state",
      "update_trip_brief",
      "resolve_places_batch",
      "estimate_route_matrix",
      "generate_plan",
      "update_plan_artifact",
      "record_travel_event",
      "replan_today",
      "apply_replan",
      "accept_family_memory"
    ]));
  });

  it("requires context for tool calls", async () => {
    const gateway = createToolGateway();
    await expect(gateway.callTool("get_trip_state", { tripId: "trip_1" }, undefined as never))
      .rejects.toThrow("ToolCallContext is required");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/tools/toolGateway.test.ts
```

Expected: FAIL，提示找不到 `toolGateway`。

- [ ] **Step 3: 实现工具目录**

创建 `services/backend/src/tools/toolCatalog.ts`：

```ts
export type ToolCatalogItem = {
  name: string;
  description: string;
  highImpact: boolean;
  confirmationRequired: boolean;
};

export const BYWAY_TOOLS: ToolCatalogItem[] = [
  { name: "create_trip", description: "Create a trip.", highImpact: false, confirmationRequired: false },
  { name: "update_trip_brief", description: "Update trip brief.", highImpact: false, confirmationRequired: false },
  { name: "get_trip_state", description: "Read authoritative trip state.", highImpact: false, confirmationRequired: false },
  { name: "set_trip_phase", description: "Set trip phase.", highImpact: true, confirmationRequired: true },
  { name: "generate_destination_candidates", description: "Generate destination candidates.", highImpact: false, confirmationRequired: false },
  { name: "research_destination_candidate", description: "Research one candidate.", highImpact: false, confirmationRequired: false },
  { name: "score_destination_candidates", description: "Score destination candidates.", highImpact: false, confirmationRequired: false },
  { name: "select_destination", description: "Select destination.", highImpact: true, confirmationRequired: true },
  { name: "resolve_places_batch", description: "Resolve places into PlaceFact.", highImpact: false, confirmationRequired: false },
  { name: "estimate_route_matrix", description: "Estimate RouteFact matrix.", highImpact: false, confirmationRequired: false },
  { name: "verify_plan_geo", description: "Verify plan geography.", highImpact: false, confirmationRequired: false },
  { name: "generate_plan", description: "Generate PlanArtifact.", highImpact: false, confirmationRequired: false },
  { name: "get_current_plan", description: "Read current PlanArtifact.", highImpact: false, confirmationRequired: false },
  { name: "update_plan_artifact", description: "Update PlanArtifact draft.", highImpact: true, confirmationRequired: true },
  { name: "confirm_plan", description: "Confirm current plan.", highImpact: true, confirmationRequired: true },
  { name: "get_today_status", description: "Read TodayArtifact.", highImpact: false, confirmationRequired: false },
  { name: "record_travel_event", description: "Record travel event.", highImpact: false, confirmationRequired: false },
  { name: "replan_today", description: "Generate replan options.", highImpact: false, confirmationRequired: false },
  { name: "apply_replan", description: "Apply selected replan option.", highImpact: true, confirmationRequired: true },
  { name: "generate_day_log", description: "Generate day log.", highImpact: false, confirmationRequired: false },
  { name: "generate_trip_log", description: "Generate trip log.", highImpact: false, confirmationRequired: false },
  { name: "extract_family_memory_candidates", description: "Extract memory candidates.", highImpact: false, confirmationRequired: false },
  { name: "accept_family_memory", description: "Accept family memory.", highImpact: true, confirmationRequired: true },
  { name: "get_accepted_family_memory", description: "Read accepted memory.", highImpact: false, confirmationRequired: false }
];
```

- [ ] **Step 4: 实现 ToolGateway 骨架**

创建 `services/backend/src/tools/toolGateway.ts`：

```ts
import { BYWAY_TOOLS, type ToolCatalogItem } from "./toolCatalog";

export type ToolCallContext = {
  traceId: string;
  turnId?: string;
  runId?: string;
  userId: string;
  tripId?: string;
  source: "agent" | "mcp" | "test";
};

export type ToolResult<T> = {
  ok: boolean;
  data: T | null;
  warnings: string[];
  errors: Array<{ code: string; message: string; recoverable: boolean }>;
  artifactUpdates: Array<{ artifactType: string; artifactId: string; version: number; content: Record<string, unknown> }>;
  nextSuggestedActions: string[];
};

export const createToolGateway = () => ({
  listTools(): ToolCatalogItem[] {
    return BYWAY_TOOLS;
  },

  async callTool(name: string, input: unknown, context: ToolCallContext): Promise<ToolResult<unknown>> {
    if (!context) throw new Error("ToolCallContext is required");
    const tool = BYWAY_TOOLS.find((item) => item.name === name);
    if (!tool) throw new Error(`Unknown Byway tool: ${name}`);
    return {
      ok: true,
      data: { toolName: name, input, source: context.source },
      warnings: [],
      errors: [],
      artifactUpdates: [],
      nextSuggestedActions: []
    };
  }
});
```

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/tools/toolGateway.test.ts
```

Expected: PASS。

---

## Task 3: HTTP App 和 `/api/chat` SSE 外壳

**Files:**
- Create: `services/backend/src/app.ts`
- Create: `services/backend/src/index.ts`
- Create: `services/backend/src/http/sse.ts`
- Create: `services/backend/src/http/routes.ts`
- Test: `services/backend/src/app.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/backend/src/app.test.ts`：

```ts
import request from "supertest";
import { describe, expect, it } from "vitest";
import { createBackendApp } from "./app";

describe("backend app", () => {
  it("reports integrated backend health", async () => {
    const app = createBackendApp();
    const response = await request(app).get("/health").expect(200);

    expect(response.body).toMatchObject({
      ok: true,
      service: "backend",
      modules: {
        http: { ok: true },
        agent: { ok: true },
        tools: { ok: true },
        mcp: { ok: true },
        trace: { ok: true }
      }
    });
  });

  it("streams chat events from the in-process agent", async () => {
    const app = createBackendApp();
    const response = await request(app)
      .post("/api/chat")
      .send({
        userId: "local_family",
        text: "杭州 3 天",
        attachments: [],
        clientContext: {}
      })
      .expect(200);

    expect(response.text).toContain("event: trace_started");
    expect(response.text).toContain("event: assistant_message_delta");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/app.test.ts
```

Expected: FAIL，提示找不到 app。

- [ ] **Step 3: 实现 SSE helper**

创建 `services/backend/src/http/sse.ts`：

```ts
import type { Response } from "express";

export const writeSse = (res: Response, event: string, data: unknown) => {
  res.write(`event: ${event}\n`);
  res.write(`data: ${JSON.stringify(data)}\n\n`);
};
```

- [ ] **Step 4: 实现 app 和 routes**

创建 `services/backend/src/app.ts`：

```ts
import cors from "cors";
import express from "express";
import { registerRoutes } from "./http/routes";

export const createBackendApp = () => {
  const app = express();
  app.use(cors());
  app.use(express.json({ limit: "2mb" }));
  registerRoutes(app);
  return app;
};
```

创建 `services/backend/src/http/routes.ts`：

```ts
import { randomUUID } from "node:crypto";
import type { Express } from "express";
import { writeSse } from "./sse";

export const registerRoutes = (app: Express) => {
  app.get("/health", (_req, res) => {
    res.json({
      ok: true,
      service: "backend",
      modules: {
        http: { ok: true },
        agent: { ok: true },
        tools: { ok: true },
        mcp: { ok: true },
        trace: { ok: true }
      }
    });
  });

  app.post("/api/chat", async (req, res) => {
    const traceId = `trace_${randomUUID()}`;
    const runId = `run_${randomUUID()}`;
    const threadId = req.body.tripId ? `thread_${req.body.tripId}` : `thread_${randomUUID()}`;

    res.writeHead(200, {
      "Content-Type": "text/event-stream; charset=utf-8",
      "Cache-Control": "no-cache, no-transform",
      Connection: "keep-alive"
    });

    writeSse(res, "trace_started", { traceId, runId, threadId });
    writeSse(res, "assistant_message_delta", { text: "我先读取这次旅行的上下文。" });
    writeSse(res, "tool_call_started", { toolName: "get_trip_state", callId: "call_get_trip_state" });
    writeSse(res, "tool_call_completed", { toolName: "get_trip_state", callId: "call_get_trip_state" });
    res.end();
  });
};
```

创建 `services/backend/src/index.ts`：

```ts
import { createBackendApp } from "./app";

const port = Number(process.env.PORT ?? 4000);
const app = createBackendApp();

app.listen(port, () => {
  console.log(`Byway backend listening on http://localhost:${port}`);
});
```

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/app.test.ts
```

Expected: PASS。

---

## Task 4: Trace 基础设施迁入 backend

**Files:**
- Create: `services/backend/src/observability/traceTypes.ts`
- Create: `services/backend/src/observability/redaction.ts`
- Create: `services/backend/src/observability/traceStore.ts`
- Create: `services/backend/src/observability/logger.ts`
- Test: `services/backend/src/observability/redaction.test.ts`
- Test: `services/backend/src/observability/traceStore.test.ts`

- [ ] **Step 1: 写脱敏测试**

创建 `services/backend/src/observability/redaction.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { redactForTrace } from "./redaction";

describe("redactForTrace", () => {
  it("redacts secrets and phone numbers", () => {
    expect(redactForTrace({
      AMAP_API_KEY: "abc123456789xyz",
      Authorization: "Bearer token",
      phone: "13812345678",
      text: "hello"
    })).toEqual({
      AMAP_API_KEY: "abc...xyz",
      Authorization: "[redacted]",
      phone: "138****5678",
      text: "hello"
    });
  });
});
```

- [ ] **Step 2: 写 TraceStore 测试**

创建 `services/backend/src/observability/traceStore.test.ts`：

```ts
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { describe, expect, it } from "vitest";
import { JsonlTraceStore } from "./traceStore";

describe("JsonlTraceStore", () => {
  it("writes and reads trace events", async () => {
    const store = new JsonlTraceStore(mkdtempSync(join(tmpdir(), "byway-trace-")));
    await store.append({
      timestamp: "2026-05-24T10:00:00.000Z",
      level: "info",
      service: "backend",
      traceId: "trace_1",
      spanId: "span_1",
      event: "chat.request.received",
      input: { text: "杭州 3 天" }
    });

    const events = await store.readTrace("trace_1");
    expect(events).toHaveLength(1);
    expect(events[0].event).toBe("chat.request.received");
  });
});
```

- [ ] **Step 3: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/observability/redaction.test.ts services/backend/src/observability/traceStore.test.ts
```

Expected: FAIL，observability 文件不存在。

- [ ] **Step 4: 实现 observability**

实现要求：

- `TraceService` 包含 `"backend" | "agent" | "tools" | "mcp" | "provider"`。
- Trace 默认写 `.byway/traces/YYYY-MM-DD/<traceId>.jsonl`。
- 日志和 Trace 输入输出都通过 `redactForTrace`。
- `/api/traces` 和 `/api/traces/:traceId` 在后续 task 挂载到 routes。

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/observability/redaction.test.ts services/backend/src/observability/traceStore.test.ts
```

Expected: PASS。

---

## Task 5: 持久化和现有 MCP 业务迁移

**Files:**
- Create: `services/backend/src/persistence/sqliteStore.ts`
- Create: `services/backend/src/domain/bywayDomain.ts`
- Create: `services/backend/src/tools/handlers/tripTools.ts`
- Create: `services/backend/src/tools/handlers/planTools.ts`
- Create: `services/backend/src/tools/handlers/placeTools.ts`
- Create: `services/backend/src/tools/handlers/travelTools.ts`
- Create: `services/backend/src/tools/handlers/memoryTools.ts`
- Test: `services/backend/src/tools/bywayToolsWorkflow.test.ts`
- Reference: `services/byway-mcp-server/src/index.ts`

- [ ] **Step 1: 写迁移验收测试**

创建 `services/backend/src/tools/bywayToolsWorkflow.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { createToolGateway } from "./toolGateway";

describe("backend Byway tools workflow", () => {
  it("creates a Hangzhou trip and generates a Hangzhou plan without Qingdao leakage", async () => {
    const gateway = createToolGateway();
    const context = { traceId: "trace_1", userId: "local_family", source: "test" as const };

    const created = await gateway.callTool("create_trip", {
      userId: "local_family",
      initialMessage: "帮我规划杭州 3 天，节奏轻松一点。",
      source: "chat"
    }, context);
    const tripId = (created.data as { tripId: string }).tripId;

    await gateway.callTool("update_trip_brief", {
      tripId,
      patch: { destination: "杭州", days: 3, pace: "relaxed" },
      sourceMessageId: "msg_1"
    }, { ...context, tripId });

    const plan = await gateway.callTool("generate_plan", {
      tripId,
      canonicalGuideId: "cg",
      tripBrief: {},
      placeFactIds: [],
      routeFactIds: [],
      familyMemoryIds: [],
      planningAssumptions: []
    }, { ...context, tripId });

    const text = JSON.stringify(plan.data);
    expect(text).toContain("杭州");
    expect(text).not.toContain("青岛");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/tools/bywayToolsWorkflow.test.ts
```

Expected: FAIL，ToolGateway 尚未接入真实业务 handlers。

- [ ] **Step 3: 迁移现有 MCP server 业务逻辑**

从 `services/byway-mcp-server/src/index.ts` 迁移业务逻辑到 backend：

- store 迁到 `persistence/sqliteStore.ts`。
- 工具 handler 按领域拆到 `tools/handlers/*`。
- 与 HTTP/MCP 无关的业务 helper 下沉到 `domain/*`。
- Provider 迁到 `providers/*`。

迁移时保持现有工具输入输出结构，避免前端和测试大面积变化。

- [ ] **Step 4: ToolGateway 接入 handlers**

`ToolGateway.callTool()` 根据工具名分发到 handler，并统一：

- schema 校验。
- trace span。
- highImpact confirmation 检查。
- error normalization。

- [ ] **Step 5: 运行迁移测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/tools/bywayToolsWorkflow.test.ts
```

Expected: PASS。

---

## Task 6: Agent Context Package 和 mode decision

**Files:**
- Create: `services/backend/src/agent/contextPackage.ts`
- Create: `services/backend/src/agent/graph/nodes/decideMode.ts`
- Test: `services/backend/src/agent/contextPackage.test.ts`
- Test: `services/backend/src/agent/graph/nodes/decideMode.test.ts`

- [ ] **Step 1: 写上下文测试**

创建 `services/backend/src/agent/contextPackage.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { buildAgentContextPackage } from "./contextPackage";

describe("buildAgentContextPackage", () => {
  it("preserves current destination in plan edit", async () => {
    const context = await buildAgentContextPackage({
      request: {
        traceId: "trace_1",
        requestId: "req_1",
        turnId: "turn_1",
        userId: "local_family",
        tripId: "trip_hz",
        text: "第三天可以改成灵隐寺吗？",
        attachments: [],
        clientContext: {}
      },
      tools: {
        callTool: async (name: string) => name === "get_trip_state"
          ? {
              ok: true,
              data: {
                trip: { id: "trip_hz", phase: "planning" },
                tripBrief: { destination: "杭州", days: 3 },
                currentArtifacts: [{ artifactType: "PlanArtifact", artifactId: "plan_1", content: { title: "杭州 3 天家庭游" } }],
                pendingConfirmations: []
              },
              warnings: [],
              errors: [],
              artifactUpdates: [],
              nextSuggestedActions: []
            }
          : {
              ok: true,
              data: { memories: [] },
              warnings: [],
              errors: [],
              artifactUpdates: [],
              nextSuggestedActions: []
            }
      } as never
    });

    expect(context.trip.destination).toBe("杭州");
    expect(JSON.stringify(context)).not.toContain("青岛");
  });
});
```

- [ ] **Step 2: 写 mode decision 测试**

创建 `services/backend/src/agent/graph/nodes/decideMode.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { decideMode } from "./decideMode";

describe("decideMode", () => {
  it("detects plan edit from current plan context", async () => {
    const result = await decideMode({
      requestText: "第三天可以改成灵隐寺吗？",
      trip: { destination: "杭州", artifacts: [{ artifactType: "PlanArtifact" }] }
    });

    expect(result).toMatchObject({
      mode: "plan_edit",
      reasonSummary: expect.stringContaining("当前计划")
    });
  });
});
```

- [ ] **Step 3: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/agent/contextPackage.test.ts services/backend/src/agent/graph/nodes/decideMode.test.ts
```

Expected: FAIL。

- [ ] **Step 4: 实现 Context Package 和 deterministic mode fallback**

实现要求：

- context 必须通过 ToolGateway 的 `get_trip_state` 读取权威状态。
- 如果用户没有明确新目的地，不覆盖当前 `destination`。
- `decideMode` 第一版是 deterministic fallback，后续接 LLM decision。
- mode decision 必须输出 reasonSummary 供 Trace 和验收使用。

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/agent/contextPackage.test.ts services/backend/src/agent/graph/nodes/decideMode.test.ts
```

Expected: PASS。

---

## Task 7: Plan Edit 首个 Agent 真闭环

**Files:**
- Create: `services/backend/src/agent/graph/bywayGraph.ts`
- Create: `services/backend/src/agent/graph/nodes/planToolSteps.ts`
- Create: `services/backend/src/agent/graph/nodes/executeTools.ts`
- Test: `services/backend/src/agent/planEditFlow.test.ts`

- [ ] **Step 1: 写 Plan Edit 流程测试**

创建 `services/backend/src/agent/planEditFlow.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { runAgentTurn } from "./graph/bywayGraph";

describe("plan edit flow", () => {
  it("keeps Hangzhou context when user adds Lingyin Temple to day three", async () => {
    const events = await runAgentTurn({
      traceId: "trace_1",
      requestId: "req_1",
      turnId: "turn_1",
      userId: "local_family",
      tripId: "trip_hz",
      text: "第三天可以改成灵隐寺吗？",
      attachments: [],
      clientContext: {}
    });

    const text = JSON.stringify(events);
    expect(text).toContain("杭州");
    expect(text).toContain("灵隐寺");
    expect(text).not.toContain("青岛");
    expect(events.some((event) => event.type === "tool_call_started" && event.toolName === "get_trip_state")).toBe(true);
    expect(events.some((event) => event.type === "tool_call_started" && event.toolName === "resolve_places_batch")).toBe(true);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/agent/planEditFlow.test.ts
```

Expected: FAIL，`runAgentTurn` 不存在或没有真实工具闭环。

- [ ] **Step 3: 实现 Plan Edit 工具计划**

Plan Edit 第一版工具序列：

```text
get_trip_state
get_current_plan
resolve_places_batch
estimate_route_matrix
update_plan_artifact
verify_plan_geo
```

实现要求：

- city 必须来自 `tripBrief.destination`。
- 用户没有明确新目的地时，不允许覆盖 `destination`。
- 每个工具调用都 yield `tool_call_started` / `tool_call_completed`。
- 更新计划后 yield `artifact_updated`。
- 高影响计划更新如果涉及已确认计划，必须进入 confirmation gate。

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/agent/planEditFlow.test.ts
```

Expected: PASS。

---

## Task 8: Confirmation interrupt 和 checkpoint

**Files:**
- Create: `services/backend/src/persistence/checkpointStore.ts`
- Create: `services/backend/src/agent/graph/nodes/confirmationGate.ts`
- Test: `services/backend/src/persistence/checkpointStore.test.ts`
- Test: `services/backend/src/agent/confirmationFlow.test.ts`

- [ ] **Step 1: 写 checkpoint 测试**

创建 `services/backend/src/persistence/checkpointStore.test.ts`：

```ts
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { describe, expect, it } from "vitest";
import { SqliteCheckpointStore } from "./checkpointStore";

describe("SqliteCheckpointStore", () => {
  it("stores and restores a paused run", () => {
    const store = new SqliteCheckpointStore(join(mkdtempSync(join(tmpdir(), "byway-checkpoint-")), "checkpoint.sqlite"));
    store.save({
      runId: "run_1",
      threadId: "thread_trip_1",
      traceId: "trace_1",
      tripId: "trip_1",
      status: "waiting_confirmation",
      checkpoint: { confirmationId: "conf_1" }
    });

    expect(store.get("run_1")).toMatchObject({
      status: "waiting_confirmation",
      checkpoint: { confirmationId: "conf_1" }
    });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/persistence/checkpointStore.test.ts
```

Expected: FAIL。

- [ ] **Step 3: 实现 SQLite checkpoint**

表结构：

```sql
CREATE TABLE IF NOT EXISTS agent_checkpoints (
  run_id TEXT PRIMARY KEY,
  thread_id TEXT NOT NULL,
  trace_id TEXT NOT NULL,
  trip_id TEXT,
  status TEXT NOT NULL,
  checkpoint_json TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

- [ ] **Step 4: 实现 confirmation gate**

规则：

- `confirm_plan`、`apply_replan`、`accept_family_memory` 必须等待用户确认。
- 如果 ToolGateway 返回 PendingConfirmation，Agent 进入 `waiting_confirmation`。
- Agent 输出 `confirmation_required` 事件。
- checkpoint 保存 run 状态。

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/persistence/checkpointStore.test.ts services/backend/src/agent/confirmationFlow.test.ts
```

Expected: PASS。

---

## Task 9: Backend MCP Adapter

**Files:**
- Create: `services/backend/src/mcp/mcpAdapter.ts`
- Test: `services/backend/src/mcp/mcpAdapter.test.ts`

- [ ] **Step 1: 写 MCP Adapter 测试**

创建 `services/backend/src/mcp/mcpAdapter.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { createMcpAdapter } from "./mcpAdapter";
import { createToolGateway } from "../tools/toolGateway";

describe("Backend MCP Adapter", () => {
  it("exposes the same tool catalog as ToolGateway", () => {
    const gateway = createToolGateway();
    const adapter = createMcpAdapter({ gateway });

    expect(adapter.listTools().map((tool) => tool.name)).toEqual(gateway.listTools().map((tool) => tool.name));
  });

  it("forwards MCP tool calls to ToolGateway with mcp source", async () => {
    const adapter = createMcpAdapter({ gateway: createToolGateway() });
    const result = await adapter.callTool("get_trip_state", { tripId: "trip_1" }, {
      traceId: "trace_1",
      userId: "local_family"
    });

    expect(result.ok).toBe(true);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/backend/src/mcp/mcpAdapter.test.ts
```

Expected: FAIL。

- [ ] **Step 3: 实现 MCP Adapter**

实现要求：

- `listTools()` 直接返回 `gateway.listTools()`。
- `callTool()` 注入 `source: "mcp"`。
- 从 MCP `_meta.byway.trace` 或环境变量中读取 `traceId`。
- 不包含业务逻辑。

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/backend/src/mcp/mcpAdapter.test.ts
```

Expected: PASS。

---

## Task 10: Hermes 家庭入口 Skill 改造

**Files:**
- Modify: `skills/byway-family-travel-agent/SKILL.md`
- Create: `skills/byway-family-travel-agent/skillContract.test.ts`
- Create: `scripts/sync-hermes-skill.sh`

- [ ] **Step 1: 写 Skill contract 测试**

创建 `skills/byway-family-travel-agent/skillContract.test.ts`：

```ts
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { describe, expect, it } from "vitest";

const skill = readFileSync(join(process.cwd(), "skills/byway-family-travel-agent/SKILL.md"), "utf8");

describe("Hermes Byway Skill", () => {
  it("is positioned as a family entry skill, not the Web orchestration runtime", () => {
    expect(skill).toContain("家庭 Agent 入口");
    expect(skill).toContain("Backend Agent");
    expect(skill).toContain("不要在 Hermes Skill 内完整生成旅行计划");
    expect(skill).toContain("调用 Backend HTTP 或 Backend MCP Adapter");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run skills/byway-family-travel-agent/skillContract.test.ts
```

Expected: FAIL，当前 Skill 仍是旧口径。

- [ ] **Step 3: 更新 Skill**

Skill 必须表达：

- Hermes 是家庭入口。
- Backend Agent 才是 Web 产品主编排。
- Hermes Skill 负责识别旅行需求、携带家庭上下文、调用 Backend HTTP 或 MCP Adapter。
- 不在 Hermes Skill 内完整生成计划，不伪造 PlaceFact / RouteFact。

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run skills/byway-family-travel-agent/skillContract.test.ts
```

Expected: PASS。

---

## Task 11: Byway Runtime Skills

**Files:**
- Create: `skills/byway-runtime/dream.md`
- Create: `skills/byway-runtime/plan.md`
- Create: `skills/byway-runtime/plan-edit.md`
- Create: `skills/byway-runtime/travel.md`
- Create: `skills/byway-runtime/review-memory.md`
- Create: `skills/byway-runtime/fact-and-confirmation-boundaries.md`
- Test: `skills/byway-runtime/runtimeSkills.test.ts`

- [ ] **Step 1: 写 Runtime Skills 测试**

创建 `skills/byway-runtime/runtimeSkills.test.ts`：

```ts
import { existsSync, readFileSync } from "node:fs";
import { join } from "node:path";
import { describe, expect, it } from "vitest";

const skillDir = join(process.cwd(), "skills/byway-runtime");

describe("Byway runtime skills", () => {
  it("defines mode-specific strategy documents", () => {
    for (const file of ["dream.md", "plan.md", "plan-edit.md", "travel.md", "review-memory.md", "fact-and-confirmation-boundaries.md"]) {
      expect(existsSync(join(skillDir, file))).toBe(true);
    }
  });

  it("keeps plan edit context discipline explicit", () => {
    const text = readFileSync(join(skillDir, "plan-edit.md"), "utf8");
    expect(text).toContain("先读取当前 Trip 和 PlanArtifact");
    expect(text).toContain("如果用户没有明确说新目的地，不要覆盖当前 destination");
    expect(text).toContain("新增地点必须先解析 PlaceFact");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run skills/byway-runtime/runtimeSkills.test.ts
```

Expected: FAIL。

- [ ] **Step 3: 创建 Runtime Skills 文档**

每个文档控制在 80-160 行，写策略和边界，不写城市词典。

`plan-edit.md` 必须包含：

```md
# Plan Edit Skill

Plan Edit 处理用户基于已有旅行计划提出的自然语言修改。

核心纪律：

- 先读取当前 Trip 和 PlanArtifact。
- 如果用户没有明确说新目的地，不要覆盖当前 destination。
- 如果用户引用“第三天”“今天下午”“晚餐后”，必须定位到当前计划中的对应 day / time block。
- 新增地点必须先解析 PlaceFact。
- 路线影响必须通过 RouteFact 或 route matrix 判断。
- 修改已确认计划属于高影响动作，必须进入确认边界。
```

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run skills/byway-runtime/runtimeSkills.test.ts
```

Expected: PASS。

---

## Task 12: Web 接入 backend SSE 和 Trace

**Files:**
- Modify: `apps/web/src/app/page.tsx`
- Modify: `apps/web/src/app/components/TripWorkspace.tsx`
- Test: `tests/e2e/*.spec.ts`

- [ ] **Step 1: 写 E2E 验收**

新增或更新 Playwright 用例：

```ts
test("keeps Hangzhou context through natural language plan edit", async ({ page }) => {
  await page.goto("http://127.0.0.1:3000/");
  await page.getByPlaceholder("说一句你的旅行需求...").fill("帮我规划杭州 3 天，节奏轻松一点。");
  await page.getByRole("button", { name: "发送" }).click();
  await expect(page.getByText("杭州")).toBeVisible();

  await page.getByPlaceholder("说一句你的旅行需求...").fill("第三天可以改成灵隐寺吗？");
  await page.getByRole("button", { name: "发送" }).click();

  await expect(page.getByText("灵隐寺")).toBeVisible();
  await expect(page.getByText("青岛")).not.toBeVisible();
  await expect(page.getByText(/trace_/)).toBeVisible();
});
```

- [ ] **Step 2: 运行 E2E 确认失败**

Run:

```bash
pnpm test:e2e
```

Expected: FAIL，当前前端仍指向旧 proxy 链路或旧 SSE 状态。

- [ ] **Step 3: 更新前端 SSE 处理**

前端处理：

- `trace_started`：保存 `traceId`、`runId`、`threadId`。
- `assistant_message_delta`：追加聊天消息。
- `tool_call_started` / `tool_call_completed`：显示工具进度。
- `artifact_updated`：刷新工作区或标记需要 refresh。
- `confirmation_required`：显示确认卡。
- `error`：终止 running 状态。

- [ ] **Step 4: 运行 E2E 确认通过**

Run:

```bash
pnpm test:e2e
```

Expected: PASS。

---

## Task 13: 环境变量、运行脚本和旧服务退场

**Files:**
- Modify: `.env.example`
- Modify: `package.json`
- Modify: `README.md`
- Modify: `pnpm-workspace.yaml`

- [ ] **Step 1: 更新 `.env.example`**

增加或更新：

```dotenv
PORT=4000
BYWAY_BACKEND_MODE=native
BYWAY_AGENT_MODE=native
BYWAY_TRACE_DIR=.byway/traces
BYWAY_CHECKPOINT_PATH=.byway/agent-checkpoints.sqlite
BYWAY_SQLITE_PATH=.byway/byway.sqlite
BYWAY_LOG_LEVEL=info
BYWAY_LOG_FORMAT=pretty
HERMES_COMMAND=hermes
HERMES_BYWAY_SKILL=byway-family-travel-agent
AMAP_API_KEY=
```

- [ ] **Step 2: 更新根 `package.json` scripts**

```json
{
  "scripts": {
    "dev": "pnpm dev:all",
    "dev:all": "concurrently -k -n backend,web -c cyan,magenta \"pnpm --filter @byway/backend dev\" \"pnpm --filter @byway/web dev\"",
    "dev:backend": "pnpm --filter @byway/backend dev",
    "test:backend": "vitest run services/backend/src/**/*.test.ts",
    "test:e2e": "BYWAY_PLACE_PROVIDER=mock BYWAY_AGENT_MODE=native playwright test"
  }
}
```

- [ ] **Step 3: README 记录一体化 backend**

README 必须说明：

- 本地只需要跑 backend + web。
- MCP 仍存在，但作为 backend 内 adapter。
- Hermes 通过 Backend HTTP/MCP 进入 Byway。
- Trace 查看方式：`/api/traces/:traceId`。
- 旧 `services/proxy-api` 和 `services/byway-mcp-server` 暂时保留为迁移参考，后续删除。

- [ ] **Step 4: 运行基础验证**

Run:

```bash
pnpm test:backend
pnpm typecheck
pnpm build
```

Expected: PASS。

---

## Task 14: 最终验收矩阵

**Files:**
- Modify: `docs/BYWAY_ACCEPTANCE_SCENARIOS.md`
- Create: `docs/superpowers/specs/2026-05-24-phase-4-integrated-backend-acceptance.md`

- [ ] **Step 1: 写验收文档**

创建 `docs/superpowers/specs/2026-05-24-phase-4-integrated-backend-acceptance.md`：

```md
# Phase 4.0 一体化 Backend 验收矩阵

## A. Web 主链路

- 输入：帮我规划杭州 3 天，节奏轻松一点。
- 期望：生成杭州 PlanArtifact。
- Trace 必须包含：chat.request.received、agent.run.started、agent.context.loaded、agent.mode.decided、tool.started、artifact.refreshed。

## B. Plan Edit

- 输入：第三天可以改成灵隐寺吗？
- 期望：仍然是杭州 trip，新增或替换灵隐寺相关节点，不出现青岛。

## C. Travel Replan

- 输入：老人累了，我们晚了 40 分钟。
- 期望：生成多个 ReplanOptions，未确认前不应用。

## D. Hermes 家庭入口

- 输入：通过 Hermes 家庭对话发起旅行需求。
- 期望：Hermes Skill 调 Backend HTTP 或 MCP Adapter，返回 Byway trip 摘要或链接。

## E. MCP Adapter

- 输入：Hermes 调 Byway MCP `get_trip_state`。
- 期望：MCP Adapter 转发到同一个 ToolGateway，Trace 中能看到 source=mcp。
```

- [ ] **Step 2: 运行全量验证**

Run:

```bash
pnpm test
pnpm test:backend
pnpm typecheck
pnpm build
pnpm test:e2e
```

Expected: 全部 PASS。

---

## 执行顺序建议

1. Task 1-3 先搭出 backend 外壳和 `/api/chat` SSE。
2. Task 4 做 Trace，避免后续调试盲飞。
3. Task 5 迁移现有 Byway MCP 业务到 ToolGateway。
4. Task 6-8 做 Agent 上下文、Plan Edit、Confirmation。
5. Task 9-11 做 MCP Adapter 和 Skills。
6. Task 12-14 做前端、运行脚本和验收矩阵。

## 关键工程原则

- 物理上合并为一个 backend，逻辑上保留 Agent、Tools、MCP、Domain 边界。
- HTTP routes 不写自然语言规则。
- Agent 不直接写数据库。
- 所有业务写入只走 ToolGateway。
- MCP Adapter 不写业务逻辑，只做协议转换。
- 所有高影响动作必须产生 PendingConfirmation。
- 每个验收 bug 都必须能从 Trace 定位到 Context、Mode、Tool Plan、ToolGateway、Provider、Artifact Refresh 的某一层。
