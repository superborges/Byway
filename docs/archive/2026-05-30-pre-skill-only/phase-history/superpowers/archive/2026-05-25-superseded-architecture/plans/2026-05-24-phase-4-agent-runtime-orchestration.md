# Phase 4.0 最终 Agent 架构 Implementation Plan

> 状态更新：本文是 Hermes-first 版本的实施计划，现保留为决策历史。后续开发以 Hybrid 版本为准：`docs/superpowers/plans/2026-05-24-phase-4-hybrid-agent-runtime.md`。

> **给 agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 Byway `/api/chat` 主链路切到最终 Agent 架构：Hermes 加载 Byway Skill 后作为 MCP client 调用 Byway MCP，Proxy 负责网关、SSE、Artifact refresh 和生产级 Trace。

**Architecture:** 新增 `services/proxy-api/src/agentRuntime/*` 和 `services/proxy-api/src/observability/*`。Proxy 构造 Agent Context Package，加载 `skills/byway-family-travel-agent/SKILL.md` 的 manifest，然后启动 Hermes Agent turn；Hermes 通过 Byway Skill 理解用户意图并直接调用 Byway MCP tools；Byway MCP protocol 层记录每个工具调用输入输出；Proxy 在 turn 结束后只读刷新 artifacts 并向前端发送更新事件。

**Tech Stack:** TypeScript、Express、SSE、Vitest、Playwright、Hermes CLI/Gateway、MCP JSON-RPC、SQLite、JSONL Trace、Zod/结构化校验。

---

## 文件结构

- Create: `services/proxy-api/src/observability/traceTypes.ts`
  - 定义 `TraceContext`、`TraceEvent`、`TraceSummary`。
- Create: `services/proxy-api/src/observability/redaction.ts`
  - 对日志和 trace 的输入输出做递归脱敏。
- Create: `services/proxy-api/src/observability/traceStore.ts`
  - 写入和读取 `.byway/traces/YYYY-MM-DD/<traceId>.jsonl`。
- Create: `services/proxy-api/src/observability/logger.ts`
  - 结构化 JSON logger，所有日志带 traceId。
- Create: `services/proxy-api/src/agentRuntime/contextPackage.ts`
  - 从 request、trip state、memory、tool catalog 构造 Hermes Agent Context Package。
- Create: `services/proxy-api/src/agentRuntime/skillManifest.ts`
  - 读取 Byway Skill 的 name、version、hash、Hermes CLI 参数和同步目标。
- Create: `services/proxy-api/src/agentRuntime/agentEvents.ts`
  - 定义 Hermes 事件归一化类型和 SSE 映射。
- Create: `services/proxy-api/src/agentRuntime/hermesAgentRuntime.ts`
  - 启动 Hermes Agent turn，解析结构化事件，处理超时和失败。
- Create: `services/proxy-api/src/agentRuntime/mockAgentRuntime.ts`
  - 显式测试 fallback，使用 fixture 而不是扩展自然语言规则。
- Create: `services/proxy-api/src/agentRuntime/index.ts`
  - runtime factory，根据 env 选择 `hermes_agent` 或 `mock`。
- Modify: `services/proxy-api/src/index.ts`
  - `/api/chat` 改为最终 Agent 主链路。
  - 新增 `/api/traces`、`/api/traces/:traceId`。
  - `/health` 增加 Hermes Agent、Byway MCP、Trace 状态。
- Modify: `services/proxy-api/src/hermesRuntimeAdapter.ts`
  - 从“状态探测”扩展为 Hermes Agent 能力探测，不再把 chat orchestration 标记为 mock。
- Create: `services/byway-mcp-server/src/toolCatalog.ts`
  - 导出 `BYWAY_MCP_TOOLS`，供 MCP protocol 和 Proxy context package 共同使用。
- Modify: `services/byway-mcp-server/src/index.ts`
  - 重新导出 `BYWAY_MCP_TOOLS`。
- Modify: `skills/byway-family-travel-agent/SKILL.md`
  - 补充 Plan Edit、Place QA、Trace、MCP 工具边界和版本说明。
- Create: `skills/byway-family-travel-agent/skillContract.test.ts`
  - 校验 Skill frontmatter、关键行为边界和工具名引用。
- Create: `scripts/sync-hermes-skill.sh`
  - 把仓库内 Byway Skill 同步到 `~/.hermes/skills/travel/byway-family-travel-agent/SKILL.md`。
- Modify: `services/byway-mcp-server/src/mcpProtocol.ts`
  - 从 `toolCatalog.ts` 读取工具目录。
  - 提取 `_meta.byway.trace`，记录 MCP tool span。
- Modify: `services/byway-mcp-server/src/providers/amapProvider.ts`
  - Provider 调用记录 trace span，错误进入 trace。
- Modify: `apps/web/src/app/page.tsx`
  - 处理 `trace_started` SSE 事件，保存当前 traceId。
- Modify: `apps/web/src/app/components/TripWorkspace.tsx`
  - 显示当前 traceId 和 trace 查看入口。
- Modify: `.env.example`
  - 增加 Agent runtime、trace、日志、超时配置。
- Modify: `README.md`
  - 增加最终架构运行方式和 trace 验收方式。
- Modify: `vitest.config.ts`
  - 增加 `skills/**/*.test.ts`，让 Skill contract 进入 `pnpm test`。

---

## Task 1: 定义 Trace 类型和脱敏规则

**Files:**
- Create: `services/proxy-api/src/observability/traceTypes.ts`
- Create: `services/proxy-api/src/observability/redaction.ts`
- Test: `services/proxy-api/src/observability/redaction.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/proxy-api/src/observability/redaction.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { redactForTrace } from "./redaction";

describe("redactForTrace", () => {
  it("redacts secrets while preserving debug shape", () => {
    const redacted = redactForTrace({
      AMAP_API_KEY: "abc123456789xyz",
      headers: { Authorization: "Bearer session-token-123" },
      phone: "13812345678",
      nested: [{ HERMES_API_KEY: "hermes-secret" }],
      text: "hello"
    });

    expect(redacted).toEqual({
      AMAP_API_KEY: "abc...xyz",
      headers: { Authorization: "[redacted]" },
      phone: "138****5678",
      nested: [{ HERMES_API_KEY: "[redacted]" }],
      text: "hello"
    });
  });

  it("truncates very long text fields", () => {
    const redacted = redactForTrace({ text: "x".repeat(2000) }, { maxStringLength: 20 });
    expect(redacted).toEqual({ text: "xxxxxxxxxxxxxxxxxxxx...[truncated 1980 chars]" });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/observability/redaction.test.ts
```

Expected: FAIL，提示找不到 `./redaction`。

- [ ] **Step 3: 实现 Trace 类型**

创建 `services/proxy-api/src/observability/traceTypes.ts`：

```ts
export type TraceService = "web" | "proxy-api" | "hermes" | "byway-mcp-server" | "provider";
export type TraceLevel = "debug" | "info" | "warn" | "error";

export type TraceContext = {
  traceId: string;
  turnId: string;
  requestId: string;
  tripId?: string;
  userId?: string;
};

export type TraceError = {
  code: string;
  message: string;
  recoverable: boolean;
};

export type TraceEvent = {
  timestamp: string;
  level: TraceLevel;
  service: TraceService;
  traceId: string;
  turnId?: string;
  requestId?: string;
  tripId?: string;
  spanId: string;
  parentSpanId?: string;
  event: string;
  name?: string;
  durationMs?: number;
  input?: unknown;
  output?: unknown;
  error?: TraceError;
};

export type TraceSummary = {
  traceId: string;
  tripId?: string;
  startedAt: string;
  completedAt?: string;
  eventCount: number;
  hasError: boolean;
  lastEvent: string;
};
```

- [ ] **Step 4: 实现脱敏**

创建 `services/proxy-api/src/observability/redaction.ts`：

```ts
type RedactionOptions = {
  maxStringLength?: number;
};

const secretKeys = new Set([
  "authorization",
  "cookie",
  "set-cookie",
  "pin",
  "token",
  "session",
  "secret",
  "hermes_api_key"
]);

const partlyRedactedKeys = new Set(["amap_api_key"]);

const redactString = (value: string, maxStringLength: number) => {
  const phoneRedacted = value.replace(/(\d{3})\d{4}(\d{4})/g, "$1****$2");
  if (phoneRedacted.length <= maxStringLength) return phoneRedacted;
  return `${phoneRedacted.slice(0, maxStringLength)}...[truncated ${phoneRedacted.length - maxStringLength} chars]`;
};

const partialSecret = (value: string) => {
  if (value.length <= 6) return "[redacted]";
  return `${value.slice(0, 3)}...${value.slice(-3)}`;
};

export const redactForTrace = (
  value: unknown,
  options: RedactionOptions = {}
): unknown => {
  const maxStringLength = options.maxStringLength ?? 1200;

  const visit = (item: unknown, keyHint?: string): unknown => {
    const normalizedKey = keyHint?.toLowerCase();
    if (typeof item === "string") {
      if (normalizedKey && secretKeys.has(normalizedKey)) return "[redacted]";
      if (normalizedKey && partlyRedactedKeys.has(normalizedKey)) return partialSecret(item);
      return redactString(item, maxStringLength);
    }
    if (Array.isArray(item)) return item.map((entry) => visit(entry));
    if (!item || typeof item !== "object") return item;

    const output: Record<string, unknown> = {};
    for (const [key, child] of Object.entries(item)) {
      output[key] = visit(child, key);
    }
    return output;
  };

  return visit(value);
};
```

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/observability/redaction.test.ts
```

Expected: PASS。

---

## Task 2: 实现 JSONL TraceStore

**Files:**
- Create: `services/proxy-api/src/observability/traceStore.ts`
- Test: `services/proxy-api/src/observability/traceStore.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/proxy-api/src/observability/traceStore.test.ts`：

```ts
import { mkdtempSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { describe, expect, it } from "vitest";
import { JsonlTraceStore } from "./traceStore";

describe("JsonlTraceStore", () => {
  it("writes events and reads a complete trace", async () => {
    const dir = mkdtempSync(join(tmpdir(), "byway-traces-"));
    const store = new JsonlTraceStore(dir);

    await store.append({
      timestamp: "2026-05-24T10:00:00.000Z",
      level: "info",
      service: "proxy-api",
      traceId: "trace_1",
      turnId: "turn_1",
      requestId: "req_1",
      tripId: "trip_1",
      spanId: "span_1",
      event: "chat.request.received",
      input: { text: "杭州 3 天" }
    });

    const events = await store.readTrace("trace_1");
    expect(events).toHaveLength(1);
    expect(events[0].event).toBe("chat.request.received");

    const summaries = await store.listSummaries({ limit: 5 });
    expect(summaries[0]).toMatchObject({
      traceId: "trace_1",
      tripId: "trip_1",
      eventCount: 1,
      hasError: false
    });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/observability/traceStore.test.ts
```

Expected: FAIL，提示找不到 `./traceStore`。

- [ ] **Step 3: 实现 TraceStore**

创建 `services/proxy-api/src/observability/traceStore.ts`：

```ts
import { appendFileSync, existsSync, mkdirSync, readdirSync, readFileSync, statSync } from "node:fs";
import { dirname, join } from "node:path";
import type { TraceEvent, TraceSummary } from "./traceTypes";

const dayFromTimestamp = (timestamp: string) => timestamp.slice(0, 10);

export class JsonlTraceStore {
  constructor(private readonly rootDir: string) {}

  async append(event: TraceEvent) {
    const filepath = this.pathFor(event.traceId, dayFromTimestamp(event.timestamp));
    mkdirSync(dirname(filepath), { recursive: true });
    appendFileSync(filepath, `${JSON.stringify(event)}\n`, "utf8");
  }

  async readTrace(traceId: string): Promise<TraceEvent[]> {
    for (const dayDir of this.dayDirs()) {
      const filepath = join(dayDir, `${traceId}.jsonl`);
      if (!existsSync(filepath)) continue;
      return readFileSync(filepath, "utf8")
        .split("\n")
        .filter(Boolean)
        .map((line) => JSON.parse(line) as TraceEvent);
    }
    return [];
  }

  async listSummaries(options: { tripId?: string; limit?: number } = {}): Promise<TraceSummary[]> {
    const summaries: TraceSummary[] = [];
    for (const dayDir of this.dayDirs()) {
      for (const name of readdirSync(dayDir).filter((item) => item.endsWith(".jsonl"))) {
        const events = readFileSync(join(dayDir, name), "utf8")
          .split("\n")
          .filter(Boolean)
          .map((line) => JSON.parse(line) as TraceEvent);
        if (events.length === 0) continue;
        if (options.tripId && events.every((event) => event.tripId !== options.tripId)) continue;
        const first = events[0];
        const last = events[events.length - 1];
        summaries.push({
          traceId: first.traceId,
          tripId: events.find((event) => event.tripId)?.tripId,
          startedAt: first.timestamp,
          completedAt: last.timestamp,
          eventCount: events.length,
          hasError: events.some((event) => event.level === "error" || Boolean(event.error)),
          lastEvent: last.event
        });
      }
    }
    return summaries
      .sort((a, b) => b.startedAt.localeCompare(a.startedAt))
      .slice(0, options.limit ?? 20);
  }

  private pathFor(traceId: string, day: string) {
    return join(this.rootDir, day, `${traceId}.jsonl`);
  }

  private dayDirs() {
    if (!existsSync(this.rootDir)) return [];
    return readdirSync(this.rootDir)
      .map((name) => join(this.rootDir, name))
      .filter((path) => existsSync(path) && statSync(path).isDirectory())
      .sort()
      .reverse();
  }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/observability/traceStore.test.ts
```

Expected: PASS。

---

## Task 3: 实现结构化 Logger 和 Trace helper

**Files:**
- Create: `services/proxy-api/src/observability/logger.ts`
- Test: `services/proxy-api/src/observability/logger.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/proxy-api/src/observability/logger.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { createTraceLogger } from "./logger";

describe("createTraceLogger", () => {
  it("writes redacted structured events to trace store and sink", async () => {
    const events: unknown[] = [];
    const traceEvents: unknown[] = [];
    const logger = createTraceLogger({
      service: "proxy-api",
      level: "debug",
      sink: (entry) => events.push(entry),
      traceStore: { append: async (event) => { traceEvents.push(event); } }
    });

    await logger.info({
      traceId: "trace_1",
      turnId: "turn_1",
      requestId: "req_1",
      spanId: "span_1",
      event: "agent.turn.started",
      input: { Authorization: "Bearer abc" }
    });

    expect(events).toHaveLength(1);
    expect(traceEvents).toHaveLength(1);
    expect(traceEvents[0]).toMatchObject({
      service: "proxy-api",
      traceId: "trace_1",
      input: { Authorization: "[redacted]" }
    });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/observability/logger.test.ts
```

Expected: FAIL，提示找不到 `./logger`。

- [ ] **Step 3: 实现 logger**

创建 `services/proxy-api/src/observability/logger.ts`：

```ts
import { randomUUID } from "node:crypto";
import { redactForTrace } from "./redaction";
import type { TraceEvent, TraceLevel, TraceService } from "./traceTypes";

type TraceStoreLike = {
  append(event: TraceEvent): Promise<void>;
};

type LoggerOptions = {
  service: TraceService;
  level?: TraceLevel;
  sink?: (entry: TraceEvent) => void;
  traceStore?: TraceStoreLike;
};

const rank: Record<TraceLevel, number> = { debug: 10, info: 20, warn: 30, error: 40 };

export const createTraceLogger = (options: LoggerOptions) => {
  const minimum = options.level ?? "info";

  const write = async (level: TraceLevel, event: Omit<TraceEvent, "timestamp" | "level" | "service">) => {
    if (rank[level] < rank[minimum]) return;
    const entry: TraceEvent = {
      ...event,
      timestamp: new Date().toISOString(),
      level,
      service: options.service,
      spanId: event.spanId || `span_${randomUUID()}`
    };
    const redacted: TraceEvent = {
      ...entry,
      input: redactForTrace(entry.input),
      output: redactForTrace(entry.output)
    };
    options.sink?.(redacted);
    await options.traceStore?.append(redacted);
  };

  return {
    debug: (event: Omit<TraceEvent, "timestamp" | "level" | "service">) => write("debug", event),
    info: (event: Omit<TraceEvent, "timestamp" | "level" | "service">) => write("info", event),
    warn: (event: Omit<TraceEvent, "timestamp" | "level" | "service">) => write("warn", event),
    error: (event: Omit<TraceEvent, "timestamp" | "level" | "service">) => write("error", event)
  };
};
```

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/observability/logger.test.ts
```

Expected: PASS。

---

## Task 4: 定义 Agent event contract

**Files:**
- Create: `services/proxy-api/src/agentRuntime/agentEvents.ts`
- Test: `services/proxy-api/src/agentRuntime/agentEvents.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/proxy-api/src/agentRuntime/agentEvents.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { normalizeHermesEvent, toSseEvent } from "./agentEvents";

describe("agentEvents", () => {
  it("normalizes Hermes assistant delta and maps it to SSE", () => {
    const normalized = normalizeHermesEvent({
      type: "assistant_delta",
      delta: "我会先读取当前计划。"
    });

    expect(normalized).toEqual({ type: "assistant_delta", delta: "我会先读取当前计划。" });
    expect(toSseEvent(normalized)).toEqual({
      event: "assistant_message_delta",
      data: { text: "我会先读取当前计划。" }
    });
  });

  it("normalizes tool events", () => {
    expect(toSseEvent(normalizeHermesEvent({
      type: "tool_started",
      toolName: "get_trip_state",
      callId: "call_1",
      input: { tripId: "trip_1" }
    }))).toMatchObject({
      event: "tool_call_started",
      data: { toolName: "get_trip_state", callId: "call_1" }
    });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/agentEvents.test.ts
```

Expected: FAIL，提示找不到 `./agentEvents`。

- [ ] **Step 3: 实现 Agent event 类型和映射**

创建 `services/proxy-api/src/agentRuntime/agentEvents.ts`：

```ts
export type AgentEvent =
  | { type: "assistant_delta"; delta: string }
  | { type: "assistant_completed"; text?: string }
  | { type: "tool_started"; toolName: string; callId: string; input?: unknown }
  | { type: "tool_completed"; toolName: string; callId: string; output?: unknown }
  | { type: "tool_failed"; toolName: string; callId: string; error: { code: string; message: string; recoverable: boolean } }
  | { type: "confirmation_required"; confirmation: unknown }
  | { type: "turn_completed"; summary?: string };

export type ProxySseEvent = {
  event:
    | "trace_started"
    | "assistant_message_delta"
    | "assistant_message_completed"
    | "tool_call_started"
    | "tool_call_completed"
    | "confirmation_required"
    | "error";
  data: Record<string, unknown>;
};

export const normalizeHermesEvent = (value: unknown): AgentEvent => {
  if (!value || typeof value !== "object") throw new Error("Invalid Hermes event");
  const event = value as Record<string, unknown>;
  if (event.type === "assistant_delta" && typeof event.delta === "string") {
    return { type: "assistant_delta", delta: event.delta };
  }
  if (event.type === "assistant_completed") {
    return { type: "assistant_completed", text: typeof event.text === "string" ? event.text : undefined };
  }
  if (event.type === "tool_started" && typeof event.toolName === "string" && typeof event.callId === "string") {
    return { type: "tool_started", toolName: event.toolName, callId: event.callId, input: event.input };
  }
  if (event.type === "tool_completed" && typeof event.toolName === "string" && typeof event.callId === "string") {
    return { type: "tool_completed", toolName: event.toolName, callId: event.callId, output: event.output };
  }
  if (event.type === "tool_failed" && typeof event.toolName === "string" && typeof event.callId === "string") {
    return {
      type: "tool_failed",
      toolName: event.toolName,
      callId: event.callId,
      error: {
        code: typeof (event.error as Record<string, unknown> | undefined)?.code === "string"
          ? String((event.error as Record<string, unknown>).code)
          : "AGENT_TOOL_FAILED",
        message: typeof (event.error as Record<string, unknown> | undefined)?.message === "string"
          ? String((event.error as Record<string, unknown>).message)
          : "Agent tool failed",
        recoverable: Boolean((event.error as Record<string, unknown> | undefined)?.recoverable ?? true)
      }
    };
  }
  if (event.type === "confirmation_required") {
    return { type: "confirmation_required", confirmation: event.confirmation };
  }
  if (event.type === "turn_completed") {
    return { type: "turn_completed", summary: typeof event.summary === "string" ? event.summary : undefined };
  }
  throw new Error("Invalid Hermes event");
};

export const toSseEvent = (event: AgentEvent): ProxySseEvent => {
  if (event.type === "assistant_delta") {
    return { event: "assistant_message_delta", data: { text: event.delta } };
  }
  if (event.type === "assistant_completed") {
    return { event: "assistant_message_completed", data: { text: event.text ?? "" } };
  }
  if (event.type === "tool_started") {
    return { event: "tool_call_started", data: { toolName: event.toolName, callId: event.callId } };
  }
  if (event.type === "tool_completed") {
    return { event: "tool_call_completed", data: { toolName: event.toolName, callId: event.callId } };
  }
  if (event.type === "tool_failed") {
    return { event: "error", data: { code: event.error.code, message: event.error.message, recoverable: event.error.recoverable } };
  }
  if (event.type === "confirmation_required") {
    return { event: "confirmation_required", data: { confirmation: event.confirmation } };
  }
  return { event: "assistant_message_completed", data: { text: event.summary ?? "" } };
};
```

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/agentEvents.test.ts
```

Expected: PASS。

---

## Task 5: 工程化 Byway Hermes Skill

**Files:**
- Modify: `skills/byway-family-travel-agent/SKILL.md`
- Reference: `docs/superpowers/specs/2026-05-24-byway-hermes-skill-strategy-design.md`
- Create: `skills/byway-family-travel-agent/skillContract.test.ts`
- Create: `services/proxy-api/src/agentRuntime/skillManifest.ts`
- Test: `services/proxy-api/src/agentRuntime/skillManifest.test.ts`
- Create: `scripts/sync-hermes-skill.sh`
- Modify: `vitest.config.ts`

- [ ] **Step 1: 写失败测试**

创建 `skills/byway-family-travel-agent/skillContract.test.ts`：

```ts
import { readFileSync } from "node:fs";
import { dirname, join } from "node:path";
import { fileURLToPath } from "node:url";
import { describe, expect, it } from "vitest";
import { BYWAY_MCP_TOOLS } from "@byway/mcp-server";

const here = dirname(fileURLToPath(import.meta.url));
const skillText = readFileSync(join(here, "SKILL.md"), "utf8");

describe("byway-family-travel-agent Skill", () => {
  it("has Hermes-loadable metadata and core Byway boundaries", () => {
    expect(skillText).toContain("name: byway-family-travel-agent");
    expect(skillText).toContain("version:");
    expect(skillText).toContain("Hermes 是智能层");
    expect(skillText).toContain("Byway MCP Tools 是确定性业务层");
    expect(skillText).toContain("Plan Edit Mode");
    expect(skillText).toContain("Place QA Mode");
    expect(skillText).toContain("Trace");
  });

  it("references only registered Byway MCP tool names", () => {
    const allowed = new Set(BYWAY_MCP_TOOLS.map((tool) => tool.name));
    const referenced = [...skillText.matchAll(/`([a-z][a-z0-9_]+)`/g)]
      .map((match) => match[1])
      .filter((name) => name.includes("_"))
      .filter((name) => !name.startsWith("BYWAY_"));
    const unknown = [...new Set(referenced)].filter((name) => !allowed.has(name));
    expect(unknown).toEqual([]);
  });
});
```

创建 `services/proxy-api/src/agentRuntime/skillManifest.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { loadBywaySkillManifest } from "./skillManifest";

describe("loadBywaySkillManifest", () => {
  it("reads the repository Byway Skill manifest for Hermes", () => {
    const manifest = loadBywaySkillManifest({ repoRoot: process.cwd() });

    expect(manifest).toMatchObject({
      name: "byway-family-travel-agent",
      version: "0.1.0",
      repoRelativePath: "skills/byway-family-travel-agent/SKILL.md",
      hermesArgs: ["--skills", "byway-family-travel-agent"]
    });
    expect(manifest.hash).toMatch(/^[a-f0-9]{64}$/);
    expect(manifest.hermesInstallPath).toContain(".hermes/skills/travel/byway-family-travel-agent/SKILL.md");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run skills/byway-family-travel-agent/skillContract.test.ts services/proxy-api/src/agentRuntime/skillManifest.test.ts
```

Expected: FAIL。`skillManifest.ts` 不存在，`vitest.config.ts` 还未 include `skills/**/*.test.ts`，Skill 里也还缺 Plan Edit / Place QA / Trace 明确章节。

- [ ] **Step 3: 补强 Byway Skill 内容**

先阅读 `docs/superpowers/specs/2026-05-24-byway-hermes-skill-strategy-design.md`。实现时遵守该文档的七层设计：产品人格与优先级、模式识别、状态纪律、工具剧本、事实边界、确认边界、Trace 友好。不要把城市词典、关键词表或具体景点映射写进 Skill。

在 `skills/byway-family-travel-agent/SKILL.md` 中增加这些章节。章节编号可以按文件现状顺延，但标题必须保留：

````md
### Plan Edit Mode：自然语言修改已有计划

触发示例：

```text
第三天可以改成灵隐寺吗？
第四天从西湖边散步改成大运河。
第三天下午加一点室内活动，别太累。
把崂山放到第二天上午，下午回酒店休息。
```

目标：

- 先读取当前 trip state，不因为用户没重复目的地就丢失上下文。
- 判断用户是在改计划草案、改已确认计划，还是旅行中临时重排。
- 必要时解析新增地点。
- 更新 PlanArtifact 或请求用户确认。

必须优先调用：

- `get_trip_state`
- `get_current_plan`

按需要调用：

- `resolve_places_batch`
- `estimate_route_matrix`
- `update_plan_artifact`
- `verify_plan_geo`

如果计划已经确认且修改影响当日行程、酒店、交通、餐点 anchor 或高星活动，必须先请求确认，不能直接生效。

### Place QA Mode：地点实用问答

触发示例：

```text
灵隐寺几点关门？
这个地方打车到哪里？
五四广场电话多少？
从酒店到这里要多久？
```

目标：

- 用 PlaceFact / RouteFact 回答地点事实。
- 不编造电话、营业时间、地址、经纬度和路线耗时。

必须调用或读取：

- `resolve_places_batch`
- `get_place_detail`
- `suggest_taxi_address`
- `estimate_route_matrix`

### Trace 与可观测性

每次 Byway chat turn 都会有 traceId。你在回复中不需要暴露技术细节，但你的工具调用必须允许 Proxy 和 MCP 记录：

- 当前加载的 Skill name、version、hash。
- 本轮意图判断。
- 每次 MCP tool 的输入输出摘要。
- Provider 错误和 recoverable 状态。

当工具失败时，不要假装已经完成。应说明当前可恢复状态，并让用户知道可以稍后重试或补充信息。
````

- [ ] **Step 4: 实现 Skill manifest loader**

创建 `services/proxy-api/src/agentRuntime/skillManifest.ts`：

```ts
import { createHash } from "node:crypto";
import { readFileSync } from "node:fs";
import { join } from "node:path";

export type BywaySkillManifest = {
  name: string;
  version: string;
  hash: string;
  repoRelativePath: string;
  absolutePath: string;
  hermesInstallPath: string;
  hermesArgs: string[];
};

type LoadOptions = {
  repoRoot: string;
  homeDir?: string;
};

const repoRelativePath = "skills/byway-family-travel-agent/SKILL.md";

const frontmatterValue = (frontmatter: string, key: string) => {
  const match = frontmatter.match(new RegExp(`^${key}:\\s*(.+)$`, "m"));
  return match?.[1]?.trim().replace(/^["']|["']$/g, "");
};

export const loadBywaySkillManifest = (options: LoadOptions): BywaySkillManifest => {
  const absolutePath = join(options.repoRoot, repoRelativePath);
  const text = readFileSync(absolutePath, "utf8");
  const frontmatter = text.match(/^---\n([\s\S]*?)\n---/)?.[1] ?? "";
  const name = frontmatterValue(frontmatter, "name");
  const version = frontmatterValue(frontmatter, "version");
  if (!name || !version) throw new Error("Byway Skill frontmatter must include name and version.");

  const homeDir = options.homeDir ?? process.env.HOME ?? "";
  return {
    name,
    version,
    hash: createHash("sha256").update(text).digest("hex"),
    repoRelativePath,
    absolutePath,
    hermesInstallPath: join(homeDir, ".hermes/skills/travel/byway-family-travel-agent/SKILL.md"),
    hermesArgs: ["--skills", name]
  };
};
```

- [ ] **Step 5: 实现 Skill 同步脚本**

创建 `scripts/sync-hermes-skill.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
REPO_ROOT="$(cd -- "${SCRIPT_DIR}/.." && pwd)"
SOURCE="${REPO_ROOT}/skills/byway-family-travel-agent/SKILL.md"
TARGET="${HOME}/.hermes/skills/travel/byway-family-travel-agent/SKILL.md"

mkdir -p "$(dirname "${TARGET}")"
cp "${SOURCE}" "${TARGET}"
printf 'Synced Byway Hermes Skill to %s\n' "${TARGET}"
```

Run:

```bash
chmod +x scripts/sync-hermes-skill.sh
```

- [ ] **Step 6: 让 Skill contract 进入测试集**

修改 `vitest.config.ts` 的 `include`：

```ts
include: [
  "packages/**/*.test.ts",
  "services/**/*.test.ts",
  "apps/**/*.test.ts",
  "skills/**/*.test.ts"
]
```

- [ ] **Step 7: 运行测试确认通过**

Run:

```bash
pnpm vitest run skills/byway-family-travel-agent/skillContract.test.ts services/proxy-api/src/agentRuntime/skillManifest.test.ts
```

Expected: PASS。

---

## Task 6: 构造 Hermes Agent Context Package

**Files:**
- Create: `services/proxy-api/src/agentRuntime/contextPackage.ts`
- Test: `services/proxy-api/src/agentRuntime/contextPackage.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/proxy-api/src/agentRuntime/contextPackage.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { buildAgentContextPackage } from "./contextPackage";

describe("buildAgentContextPackage", () => {
  it("keeps trip context and tool boundaries for Hermes", () => {
    const context = buildAgentContextPackage({
      trace: { traceId: "trace_1", turnId: "turn_1", requestId: "req_1", tripId: "trip_1", userId: "local_family" },
      request: { text: "第三天可以改成灵隐寺吗？", attachments: [], clientContext: {} },
      tripState: {
        trip: { tripId: "trip_1", phase: "planning" },
        tripBrief: { destination: "杭州", days: 3, travelers: ["父母", "孩子"] },
        currentArtifacts: [{ artifactType: "PlanArtifact", title: "杭州 3 天家庭游" }],
        pendingConfirmations: []
      },
      familyMemory: [{ category: "stamina", content: "下午 3 点后不适合高强度景点。" }],
      tools: [{ name: "get_trip_state", description: "Read trip state.", inputSchema: {} }],
      skill: {
        name: "byway-family-travel-agent",
        version: "0.1.0",
        hash: "abc123"
      }
    });

    expect(context.skill).toMatchObject({ name: "byway-family-travel-agent", version: "0.1.0" });
    expect(context.trip.brief).toMatchObject({ destination: "杭州" });
    expect(context.request.text).toBe("第三天可以改成灵隐寺吗？");
    expect(context.tools[0]).toMatchObject({ name: "get_trip_state" });
    expect(context.safety.confirmationRules).toContain("confirm_plan 和 apply_replan 必须在用户确认后生效");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/contextPackage.test.ts
```

Expected: FAIL，提示找不到 `./contextPackage`。

- [ ] **Step 3: 实现 context package**

创建 `services/proxy-api/src/agentRuntime/contextPackage.ts`：

```ts
import type { TraceContext } from "../observability/traceTypes";

type ToolDescriptor = {
  name: string;
  description?: string;
  inputSchema?: unknown;
};

type BuildInput = {
  trace: TraceContext;
  request: {
    text: string;
    attachments?: unknown[];
    clientContext?: Record<string, unknown>;
  };
  tripState?: {
    trip?: Record<string, unknown>;
    tripBrief?: Record<string, unknown>;
    currentArtifacts?: Record<string, unknown>[];
    pendingConfirmations?: Record<string, unknown>[];
  };
  familyMemory?: Record<string, unknown>[];
  tools: ToolDescriptor[];
  skill: {
    name: string;
    version: string;
    hash: string;
  };
};

const summarizeArtifact = (artifact: Record<string, unknown>) => ({
  artifactId: artifact.artifactId,
  artifactType: artifact.artifactType,
  version: artifact.version,
  title: artifact.title,
  summary: artifact.summary
});

export const buildAgentContextPackage = (input: BuildInput) => ({
  trace: input.trace,
  skill: input.skill,
  request: {
    userId: input.trace.userId ?? "local_family",
    text: input.request.text,
    attachments: (input.request.attachments ?? []).slice(0, 5),
    clientContext: input.request.clientContext ?? {}
  },
  trip: {
    tripId: input.trace.tripId,
    phase: input.tripState?.trip?.phase,
    brief: input.tripState?.tripBrief ?? {},
    recentArtifacts: (input.tripState?.currentArtifacts ?? []).slice(-8).map(summarizeArtifact),
    pendingConfirmations: input.tripState?.pendingConfirmations ?? []
  },
  memory: {
    acceptedFamilyMemory: input.familyMemory ?? []
  },
  tools: input.tools.map((tool) => ({
    name: tool.name,
    description: tool.description ?? "",
    highImpact: ["confirm_plan", "apply_replan", "accept_family_memory"].includes(tool.name),
    confirmationRequired: ["confirm_plan", "apply_replan", "accept_family_memory"].includes(tool.name)
  })),
  safety: {
    forbiddenIntents: ["booking_payment", "unauthorized_scraping", "credential_exposure"],
    factBoundaries: ["PlaceFact owns location facts", "RouteFact owns route duration and distance"],
    confirmationRules: ["confirm_plan 和 apply_replan 必须在用户确认后生效"]
  }
});
```

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/contextPackage.test.ts
```

Expected: PASS。

---

## Task 7: 抽出 Byway MCP 工具目录并记录工具 Trace

**Files:**
- Create: `services/byway-mcp-server/src/toolCatalog.ts`
- Modify: `services/byway-mcp-server/src/index.ts`
- Modify: `services/byway-mcp-server/src/mcpProtocol.ts`
- Test: `services/byway-mcp-server/src/mcpProtocol.test.ts`

- [ ] **Step 1: 写失败测试**

在 `services/byway-mcp-server/src/mcpProtocol.test.ts` 增加：

```ts
it("emits trace events for MCP tool calls when trace metadata is present", async () => {
  const traceEvents: unknown[] = [];
  const server = {
    get_trip_state: async () => ({ ok: true, data: { tripId: "trip_1" } })
  };
  const handle = createBywayMcpJsonRpcHandler(server as never, {
    traceSink: (event) => traceEvents.push(event)
  });

  const response = await handle({
    jsonrpc: "2.0",
    id: 1,
    method: "tools/call",
    params: {
      name: "get_trip_state",
      arguments: {
        tripId: "trip_1",
        _meta: { byway: { traceId: "trace_1", turnId: "turn_1", requestId: "req_1" } }
      }
    }
  });

  expect(response?.result).toBeDefined();
  expect(traceEvents.map((event) => (event as { event: string }).event)).toEqual([
    "mcp.tool.started",
    "mcp.tool.completed"
  ]);
  expect(traceEvents[0]).toMatchObject({
    service: "byway-mcp-server",
    traceId: "trace_1",
    name: "get_trip_state"
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/byway-mcp-server/src/mcpProtocol.test.ts -t "emits trace events"
```

Expected: FAIL，`createBywayMcpJsonRpcHandler` 不接受第二个参数。

- [ ] **Step 3: 抽出工具目录**

创建 `services/byway-mcp-server/src/toolCatalog.ts`：

```ts
const looseInputSchema = {
  type: "object",
  additionalProperties: true
};

export const BYWAY_MCP_TOOLS = [
  { name: "create_trip", description: "Create a Byway trip and initial trip brief." },
  { name: "update_trip_brief", description: "Update the canonical trip brief." },
  { name: "get_trip_state", description: "Read trip state, artifacts, and pending confirmations." },
  { name: "list_trips", description: "List trips for a local family user." },
  { name: "set_trip_phase", description: "Move a trip through the Byway phase state machine." },
  { name: "generate_destination_candidates", description: "Generate destination shortlist candidates." },
  { name: "research_destination_candidate", description: "Research one destination candidate." },
  { name: "score_destination_candidates", description: "Score destination candidates and emit shortlist artifact." },
  { name: "select_destination", description: "Select a destination from the shortlist." },
  { name: "generate_synthetic_guides", description: "Generate model-style guide sources for planning." },
  { name: "ingest_user_guide_source", description: "Ingest user supplied guide text, URL, note, or markdown." },
  { name: "merge_guides", description: "Merge guide extractions into a canonical guide summary artifact." },
  { name: "resolve_places_batch", description: "Resolve places through PlaceFact provider." },
  { name: "get_place_detail", description: "Read PlaceFact details such as opening hours." },
  { name: "suggest_taxi_address", description: "Suggest a taxi/navigation address from PlaceFact data." },
  { name: "estimate_route_matrix", description: "Estimate route duration and distance between PlaceFacts." },
  { name: "verify_plan_geo", description: "Check plan geography and surface route warnings." },
  { name: "generate_plan", description: "Generate a PlanArtifact." },
  { name: "get_current_plan", description: "Read the current PlanArtifact." },
  { name: "update_plan_artifact", description: "Create a revised PlanArtifact version." },
  { name: "confirm_plan", description: "Confirm a plan after user approval." },
  { name: "get_today_status", description: "Generate a TodayArtifact from the current plan." },
  { name: "record_travel_event", description: "Record an in-trip event or disruption." },
  { name: "replan_today", description: "Generate replan options for the current day." },
  { name: "apply_replan", description: "Apply a replan option after user confirmation." },
  { name: "generate_day_log", description: "Generate a day review log." },
  { name: "generate_trip_log", description: "Generate a whole-trip review log." },
  { name: "extract_family_memory_candidates", description: "Extract candidate family travel memories." },
  { name: "accept_family_memory", description: "Accept or reject family memory candidates." },
  { name: "respond_pending_confirmation", description: "Mark a pending confirmation as accepted, rejected, or expired." },
  { name: "get_accepted_family_memory", description: "Read accepted family travel memories." }
].map((tool) => ({ ...tool, inputSchema: looseInputSchema }));
```

在 `services/byway-mcp-server/src/index.ts` 顶部或底部增加：

```ts
export { BYWAY_MCP_TOOLS } from "./toolCatalog";
```

在 `services/byway-mcp-server/src/mcpProtocol.ts` 删除本地 `looseInputSchema` 和 `BYWAY_MCP_TOOLS` 定义，改为：

```ts
import { BYWAY_MCP_TOOLS } from "./toolCatalog";
```

- [ ] **Step 4: 修改 MCP protocol handler**

在 `services/byway-mcp-server/src/mcpProtocol.ts` 中新增 options、meta 提取和 trace sink：

```ts
type McpTraceSink = (event: {
  timestamp: string;
  level: "info" | "error";
  service: "byway-mcp-server";
  traceId: string;
  turnId?: string;
  requestId?: string;
  spanId: string;
  event: "mcp.tool.started" | "mcp.tool.completed" | "mcp.tool.failed";
  name: string;
  durationMs?: number;
  input?: unknown;
  output?: unknown;
  error?: { code: string; message: string; recoverable: boolean };
}) => void;

type McpProtocolOptions = {
  traceSink?: McpTraceSink;
};

const extractTrace = (args: Record<string, unknown>) => {
  const meta = args._meta as Record<string, unknown> | undefined;
  const byway = meta?.byway as Record<string, unknown> | undefined;
  return {
    traceId: typeof byway?.traceId === "string" ? byway.traceId : undefined,
    turnId: typeof byway?.turnId === "string" ? byway.turnId : undefined,
    requestId: typeof byway?.requestId === "string" ? byway.requestId : undefined
  };
};

const stripMeta = (args: Record<string, unknown>) => {
  const { _meta, ...rest } = args;
  return rest;
};
```

把 handler 签名改成：

```ts
export const createBywayMcpJsonRpcHandler = (
  server: BywayMcpServer,
  options: McpProtocolOptions = {}
) => async (
  request: JsonRpcRequest
): Promise<JsonRpcResponse | undefined> => {
```

在 `tools/call` 分支中，调用业务方法前后写 trace：

```ts
const trace = extractTrace(args);
const startedAt = Date.now();
const spanId = `mcp_${id ?? Date.now()}_${toolName}`;
if (trace.traceId) {
  options.traceSink?.({
    timestamp: new Date().toISOString(),
    level: "info",
    service: "byway-mcp-server",
    traceId: trace.traceId,
    turnId: trace.turnId,
    requestId: trace.requestId,
    spanId,
    event: "mcp.tool.started",
    name: toolName,
    input: stripMeta(args)
  });
}

try {
  const result = await callable.call(server, stripMeta(args)) as unknown;
  if (trace.traceId) {
    options.traceSink?.({
      timestamp: new Date().toISOString(),
      level: "info",
      service: "byway-mcp-server",
      traceId: trace.traceId,
      turnId: trace.turnId,
      requestId: trace.requestId,
      spanId,
      event: "mcp.tool.completed",
      name: toolName,
      durationMs: Date.now() - startedAt,
      output: result
    });
  }
  return okResponse(id, { content: [{ type: "text", text: JSON.stringify(result) }] });
} catch (error) {
  if (trace.traceId) {
    options.traceSink?.({
      timestamp: new Date().toISOString(),
      level: "error",
      service: "byway-mcp-server",
      traceId: trace.traceId,
      turnId: trace.turnId,
      requestId: trace.requestId,
      spanId,
      event: "mcp.tool.failed",
      name: toolName,
      durationMs: Date.now() - startedAt,
      error: {
        code: "MCP_TOOL_FAILED",
        message: error instanceof Error ? error.message : "Byway MCP tool failed",
        recoverable: true
      }
    });
  }
  return errorResponse(id, -32000, error instanceof Error ? error.message : "Byway MCP tool failed");
}
```

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/byway-mcp-server/src/mcpProtocol.test.ts -t "emits trace events"
```

Expected: PASS。

---

## Task 8: 实现 Hermes Agent Runtime

**Files:**
- Create: `services/proxy-api/src/agentRuntime/hermesAgentRuntime.ts`
- Test: `services/proxy-api/src/agentRuntime/hermesAgentRuntime.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/proxy-api/src/agentRuntime/hermesAgentRuntime.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { HermesAgentRuntime } from "./hermesAgentRuntime";

describe("HermesAgentRuntime", () => {
  it("streams normalized events from a JSONL Hermes runner", async () => {
    const runs: unknown[] = [];
    const runtime = new HermesAgentRuntime({
      command: "fake-hermes",
      args: ["chat", "--json-events"],
      skill: {
        name: "byway-family-travel-agent",
        version: "0.1.0",
        hash: "abc123",
        hermesArgs: ["--skills", "byway-family-travel-agent"]
      },
      timeoutMs: 1000,
      runCommand: async function* (input) {
        runs.push(input);
        yield JSON.stringify({ type: "assistant_delta", delta: "我先读取当前计划。" });
        yield JSON.stringify({ type: "tool_started", toolName: "get_trip_state", callId: "call_1", input: { tripId: "trip_1" } });
        yield JSON.stringify({ type: "tool_completed", toolName: "get_trip_state", callId: "call_1", output: { ok: true } });
        yield JSON.stringify({ type: "turn_completed", summary: "完成" });
      }
    });

    const events = [];
    for await (const event of runtime.runTurn({
      trace: { traceId: "trace_1", turnId: "turn_1", requestId: "req_1", tripId: "trip_1", userId: "local_family" },
      contextPackage: { request: { text: "第三天可以改成灵隐寺吗？" } }
    })) {
      events.push(event);
    }

    expect(events.map((event) => event.type)).toEqual([
      "assistant_delta",
      "tool_started",
      "tool_completed",
      "turn_completed"
    ]);
    expect(runs[0]).toMatchObject({
      args: ["chat", "--json-events", "--skills", "byway-family-travel-agent"],
      env: { BYWAY_SKILL_NAME: "byway-family-travel-agent", BYWAY_SKILL_VERSION: "0.1.0" }
    });
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/hermesAgentRuntime.test.ts
```

Expected: FAIL，提示找不到 `./hermesAgentRuntime`。

- [ ] **Step 3: 实现 HermesAgentRuntime**

创建 `services/proxy-api/src/agentRuntime/hermesAgentRuntime.ts`：

```ts
import { spawn } from "node:child_process";
import readline from "node:readline";
import { normalizeHermesEvent, type AgentEvent } from "./agentEvents";
import type { TraceContext } from "../observability/traceTypes";

type RunnerInput = {
  command: string;
  args: string[];
  env: NodeJS.ProcessEnv;
  stdin: string;
  timeoutMs: number;
};

type RunTurnInput = {
  trace: TraceContext;
  contextPackage: unknown;
};

type HermesAgentRuntimeOptions = {
  command: string;
  args: string[];
  skill: {
    name: string;
    version: string;
    hash: string;
    hermesArgs: string[];
  };
  timeoutMs: number;
  runCommand?: (input: RunnerInput) => AsyncIterable<string>;
};

const defaultRunCommand = async function* (input: RunnerInput): AsyncIterable<string> {
  const child = spawn(input.command, input.args, {
    env: input.env,
    stdio: ["pipe", "pipe", "pipe"]
  });
  const stderr: string[] = [];
  child.stderr.on("data", (chunk) => {
    stderr.push(String(chunk));
    if (stderr.length > 8) stderr.shift();
  });
  child.stdin.write(input.stdin);
  child.stdin.end();

  const timeout = setTimeout(() => {
    if (!child.killed) child.kill();
  }, input.timeoutMs);

  try {
    const lines = readline.createInterface({ input: child.stdout, crlfDelay: Infinity });
    for await (const line of lines) {
      if (line.trim()) yield line;
    }
  } finally {
    clearTimeout(timeout);
    if (!child.killed) child.kill();
  }
};

export class HermesAgentRuntime {
  constructor(private readonly options: HermesAgentRuntimeOptions) {}

  async *runTurn(input: RunTurnInput): AsyncIterable<AgentEvent> {
    const stdin = JSON.stringify({
      instruction: "You are Byway travel agent. The byway-family-travel-agent Skill is the durable behavior contract. Use the registered Byway MCP tools for all trip state, place facts, route facts, confirmations, and memory. Emit JSONL events only.",
      trace: input.trace,
      skill: {
        name: this.options.skill.name,
        version: this.options.skill.version,
        hash: this.options.skill.hash
      },
      context: input.contextPackage
    });

    const runner = this.options.runCommand ?? defaultRunCommand;
    for await (const line of runner({
      command: this.options.command,
      args: [...this.options.args, ...this.options.skill.hermesArgs],
      env: {
        ...process.env,
        BYWAY_TRACE_ID: input.trace.traceId,
        BYWAY_TURN_ID: input.trace.turnId,
        BYWAY_REQUEST_ID: input.trace.requestId,
        BYWAY_SKILL_NAME: this.options.skill.name,
        BYWAY_SKILL_VERSION: this.options.skill.version,
        BYWAY_SKILL_HASH: this.options.skill.hash
      },
      stdin,
      timeoutMs: this.options.timeoutMs
    })) {
      yield normalizeHermesEvent(JSON.parse(line));
    }
  }
}
```

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/hermesAgentRuntime.test.ts
```

Expected: PASS。

---

## Task 9: 新增 runtime factory 和显式 mock fallback

**Files:**
- Create: `services/proxy-api/src/agentRuntime/mockAgentRuntime.ts`
- Create: `services/proxy-api/src/agentRuntime/index.ts`
- Test: `services/proxy-api/src/agentRuntime/index.test.ts`

- [ ] **Step 1: 写失败测试**

创建 `services/proxy-api/src/agentRuntime/index.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { createAgentRuntimeFromEnv } from "./index";

describe("createAgentRuntimeFromEnv", () => {
  it("selects Hermes agent runtime by default for local production-like config", () => {
    const runtime = createAgentRuntimeFromEnv({
      BYWAY_AGENT_RUNTIME: "hermes_agent",
      HERMES_COMMAND: "hermes",
      HERMES_AGENT_ARGS: "chat,--json-events",
      HERMES_AGENT_SKILLS: "byway-family-travel-agent",
      HERMES_AGENT_TIMEOUT_MS: "90000"
    });
    expect(runtime.kind).toBe("hermes_agent");
  });

  it("requires explicit mock runtime for fixture mode", () => {
    const runtime = createAgentRuntimeFromEnv({ BYWAY_AGENT_RUNTIME: "mock" });
    expect(runtime.kind).toBe("mock");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/index.test.ts
```

Expected: FAIL，提示找不到 `./index`。

- [ ] **Step 3: 实现 mock runtime**

创建 `services/proxy-api/src/agentRuntime/mockAgentRuntime.ts`：

```ts
import type { AgentEvent } from "./agentEvents";

export class MockAgentRuntime {
  readonly kind = "mock" as const;

  async *runTurn(): AsyncIterable<AgentEvent> {
    yield { type: "assistant_delta", delta: "我现在使用本地 mock Agent。这个模式只用于测试和离线演示。" };
    yield { type: "assistant_completed", text: "我现在使用本地 mock Agent。这个模式只用于测试和离线演示。" };
    yield { type: "turn_completed", summary: "mock turn completed" };
  }
}
```

- [ ] **Step 4: 实现 runtime factory**

创建 `services/proxy-api/src/agentRuntime/index.ts`：

```ts
import { HermesAgentRuntime } from "./hermesAgentRuntime";
import { loadBywaySkillManifest } from "./skillManifest";
import { MockAgentRuntime } from "./mockAgentRuntime";

type EnvLike = Record<string, string | undefined>;

const splitArgs = (value: string | undefined) => (
  value?.split(",").map((item) => item.trim()).filter(Boolean) ?? ["chat", "--json-events"]
);

export const createAgentRuntimeFromEnv = (env: EnvLike = process.env) => {
  if (env.BYWAY_AGENT_RUNTIME === "mock") return new MockAgentRuntime();
  const skill = loadBywaySkillManifest({ repoRoot: process.cwd() });
  const requestedSkills = env.HERMES_AGENT_SKILLS?.trim();
  const hermesArgs = requestedSkills ? ["--skills", requestedSkills] : skill.hermesArgs;

  return Object.assign(new HermesAgentRuntime({
    command: env.HERMES_COMMAND?.trim() || "hermes",
    args: splitArgs(env.HERMES_AGENT_ARGS),
    skill: { ...skill, hermesArgs },
    timeoutMs: Number(env.HERMES_AGENT_TIMEOUT_MS ?? 90000)
  }), { kind: "hermes_agent" as const });
};
```

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/agentRuntime/index.test.ts
```

Expected: PASS。

---

## Task 10: 改造 `/api/chat` 为 Hermes Agent 主链路

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] **Step 1: 写失败测试**

在 `services/proxy-api/src/proxyApi.test.ts` 增加：

```ts
it("streams chat through Agent runtime and returns trace id", async () => {
  const agentRuntime = {
    kind: "mock",
    async *runTurn() {
      yield { type: "assistant_delta", delta: "我先读取当前杭州计划。" };
      yield { type: "tool_started", toolName: "get_trip_state", callId: "call_1" };
      yield { type: "tool_completed", toolName: "get_trip_state", callId: "call_1" };
      yield { type: "turn_completed", summary: "完成" };
    }
  };

  const app = createProxyApp({
    mcp: await createBywayMcpServer(),
    agentRuntime: agentRuntime as never
  });

  const response = await request(app)
    .post("/api/chat")
    .send({ message: { text: "第三天可以改成灵隐寺吗？" }, clientContext: {} })
    .expect(200);

  const events = parseSse(response.text);
  expect(events[0].event).toBe("trace_started");
  expect(events.some((event) => event.event === "assistant_message_delta")).toBe(true);
  expect(events.some((event) => event.event === "tool_call_started")).toBe(true);
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/proxyApi.test.ts -t "streams chat through Agent runtime"
```

Expected: FAIL，`createProxyApp` 还不接受 `agentRuntime`，SSE 也没有 `trace_started`。

- [ ] **Step 3: 修改 Proxy 类型和 SSE event**

在 `services/proxy-api/src/index.ts`：

- `ProxyAppOptions` 增加 `agentRuntime?: { kind: string; runTurn(input: unknown): AsyncIterable<unknown> }`。
- `SseEvent` 增加 `"trace_started"`。
- 初始化 `traceStore`、`traceLogger`、`agentRuntime`、`skillManifest`。
- `/api/chat` 开头生成 `traceId`、`turnId`、`requestId`。

核心片段：

```ts
const trace = {
  traceId: `trace_${randomUUID()}`,
  turnId: `turn_${randomUUID()}`,
  requestId: `req_${randomUUID()}`,
  tripId: body.tripId,
  userId: USER_ID
};

sendEvent(res, "trace_started", {
  traceId: trace.traceId,
  turnId: trace.turnId,
  requestId: trace.requestId
});
```

- [ ] **Step 4: 调用 Agent runtime 并映射事件**

在 `/api/chat` 中，用 Agent runtime 替代规则分支：

```ts
await traceLogger.info({
  traceId: trace.traceId,
  turnId: trace.turnId,
  requestId: trace.requestId,
  tripId: trace.tripId,
  spanId: `skill_${trace.turnId}`,
  event: "agent.skill.loaded",
  name: skillManifest.name,
  output: {
    version: skillManifest.version,
    hash: skillManifest.hash,
    path: skillManifest.hermesInstallPath
  }
});

const contextPackage = buildAgentContextPackage({
  trace,
  skill: {
    name: skillManifest.name,
    version: skillManifest.version,
    hash: skillManifest.hash
  },
  request: {
    text,
    attachments: body.message?.attachments ?? [],
    clientContext: body.clientContext ?? {}
  },
  tripState: currentState,
  familyMemory: acceptedMemory,
  tools: BYWAY_MCP_TOOLS
});

for await (const agentEvent of agentRuntime.runTurn({ trace, contextPackage })) {
  const normalized = normalizeHermesEvent(agentEvent);
  const sse = toSseEvent(normalized);
  sendEvent(res, sse.event, sse.data);
}
```

保留 turn 结束后的只读 artifact refresh：

```ts
if (trace.tripId) {
  const state = await server.get_trip_state({ tripId: trace.tripId, includeArtifacts: true });
  for (const artifact of state.data?.currentArtifacts ?? []) {
    sendEvent(res, "artifact_updated", { artifact });
  }
}
```

- [ ] **Step 5: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/proxyApi.test.ts -t "streams chat through Agent runtime"
```

Expected: PASS。

---

## Task 11: 新增 Trace API

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] **Step 1: 写失败测试**

在 `services/proxy-api/src/proxyApi.test.ts` 增加：

```ts
it("exposes trace summaries and trace events", async () => {
  const app = createProxyApp({
    mcp: await createBywayMcpServer(),
    traceStore: {
      listSummaries: async () => [{
        traceId: "trace_1",
        tripId: "trip_1",
        startedAt: "2026-05-24T10:00:00.000Z",
        completedAt: "2026-05-24T10:00:01.000Z",
        eventCount: 2,
        hasError: false,
        lastEvent: "chat.request.completed"
      }],
      readTrace: async () => [{
        timestamp: "2026-05-24T10:00:00.000Z",
        level: "info",
        service: "proxy-api",
        traceId: "trace_1",
        spanId: "span_1",
        event: "chat.request.received"
      }],
      append: async () => undefined
    } as never
  });

  const list = await request(app).get("/api/traces").expect(200);
  expect(list.body.traces[0].traceId).toBe("trace_1");

  const detail = await request(app).get("/api/traces/trace_1").expect(200);
  expect(detail.body.events[0].event).toBe("chat.request.received");
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/proxyApi.test.ts -t "exposes trace summaries"
```

Expected: FAIL，trace API 未实现。

- [ ] **Step 3: 实现 Trace API**

在 `services/proxy-api/src/index.ts` 增加：

```ts
app.get("/api/traces", async (req, res) => {
  const tripId = typeof req.query.tripId === "string" ? req.query.tripId : undefined;
  const limit = typeof req.query.limit === "string" ? Number(req.query.limit) : 20;
  const traces = await traceStore.listSummaries({ tripId, limit });
  res.json({ ok: true, traces });
});

app.get("/api/traces/:traceId", async (req, res) => {
  const events = await traceStore.readTrace(req.params.traceId);
  if (events.length === 0) {
    res.status(404).json({ ok: false, error: { code: "TRACE_NOT_FOUND", message: "Trace not found" } });
    return;
  }
  res.json({ ok: true, traceId: req.params.traceId, events });
});
```

- [ ] **Step 4: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/proxyApi.test.ts -t "exposes trace summaries"
```

Expected: PASS。

---

## Task 12: 前端展示 traceId 和处理 trace_started

**Files:**
- Modify: `apps/web/src/app/workspaceState.ts`
- Modify: `apps/web/src/app/page.tsx`
- Modify: `apps/web/src/app/components/TripWorkspace.tsx`
- Test: `apps/web/src/app/workspaceState.test.ts`

- [ ] **Step 1: 写失败测试**

在 `apps/web/src/app/workspaceState.test.ts` 增加：

```ts
it("stores the active trace id when a trace starts", () => {
  const state = reduceWorkspaceState(initialWorkspaceState, {
    type: "trace_started",
    traceId: "trace_1"
  });

  expect(state.activeTraceId).toBe("trace_1");
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run apps/web/src/app/workspaceState.test.ts -t "stores the active trace id"
```

Expected: FAIL，state 不支持 `activeTraceId`。

- [ ] **Step 3: 更新 workspace state**

在 `apps/web/src/app/workspaceState.ts`：

- state 增加 `activeTraceId?: string`。
- action 增加 `{ type: "trace_started"; traceId: string }`。
- reducer 保存 `activeTraceId`。

核心片段：

```ts
if (action.type === "trace_started") {
  return { ...state, activeTraceId: action.traceId };
}
```

- [ ] **Step 4: 更新 SSE 处理**

在 `apps/web/src/app/page.tsx` 的 SSE event switch 中增加：

```ts
if (event.event === "trace_started") {
  dispatch({ type: "trace_started", traceId: String(event.data.traceId) });
  return;
}
```

- [ ] **Step 5: 在 TripWorkspace 显示 Trace**

在 `apps/web/src/app/components/TripWorkspace.tsx` 的 props 中增加：

```ts
activeTraceId?: string;
traceBaseUrl: string;
```

在 `apps/web/src/app/page.tsx` 渲染 `TripWorkspace` 时传入：

```tsx
activeTraceId={state.activeTraceId}
traceBaseUrl={API_BASE}
```

在 `TripWorkspace` 的标题区域显示：

```tsx
{activeTraceId ? (
  <a className="trace-link" href={`${traceBaseUrl}/api/traces/${activeTraceId}`} target="_blank" rel="noreferrer">
    Trace {activeTraceId.slice(0, 12)}
  </a>
) : null}
```

- [ ] **Step 6: 运行测试确认通过**

Run:

```bash
pnpm vitest run apps/web/src/app/workspaceState.test.ts -t "stores the active trace id"
```

Expected: PASS。

---

## Task 13: 更新 health、env 和 README

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Modify: `.env.example`
- Modify: `README.md`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] **Step 1: 写失败测试**

在 `services/proxy-api/src/proxyApi.test.ts` 增加：

```ts
it("reports production readiness dependencies in health", async () => {
  const app = createProxyApp({ mcp: await createBywayMcpServer() });
  const response = await request(app).get("/health").expect(200);

  expect(response.body).toHaveProperty("agentRuntime");
  expect(response.body).toHaveProperty("bywaySkill");
  expect(response.body).toHaveProperty("trace");
  expect(response.body).toHaveProperty("mcp");
});
```

- [ ] **Step 2: 运行测试确认失败**

Run:

```bash
pnpm vitest run services/proxy-api/src/proxyApi.test.ts -t "reports production readiness"
```

Expected: FAIL，health 缺少字段。

- [ ] **Step 3: 更新 health**

在 `/health` 返回体中增加：

```ts
agentRuntime: {
  kind: agentRuntime.kind,
  productionReady: agentRuntime.kind === "hermes_agent",
  eventStream: process.env.HERMES_AGENT_ARGS?.includes("--json-events") ?? false
},
bywaySkill: {
  name: skillManifest.name,
  version: skillManifest.version,
  hash: skillManifest.hash,
  hermesInstallPath: skillManifest.hermesInstallPath
},
mcp: {
  bywayTools: BYWAY_MCP_TOOLS.length,
  stdioServer: "available"
},
trace: {
  enabled: process.env.BYWAY_TRACE_ENABLED !== "false",
  dir: process.env.BYWAY_TRACE_DIR || ".byway/traces"
}
```

- [ ] **Step 4: 更新 `.env.example`**

追加：

```dotenv
# Agent runtime
BYWAY_AGENT_RUNTIME=hermes_agent
HERMES_COMMAND=hermes
HERMES_AGENT_ARGS=chat,--json-events
HERMES_AGENT_SKILLS=byway-family-travel-agent
HERMES_AGENT_TIMEOUT_MS=90000

# Observability
BYWAY_TRACE_ENABLED=true
BYWAY_TRACE_DIR=.byway/traces
BYWAY_LOG_LEVEL=info
BYWAY_LOG_FORMAT=json
```

把原来“`/api/chat` 仍走 mock orchestrator”的描述改为“`mock` 只用于测试和离线演示”。

- [ ] **Step 5: 更新 README**

在 `README.md` 增加：

```md
## Phase 4.0 Agent runtime

本地正式链路使用 `BYWAY_AGENT_RUNTIME=hermes_agent`。Proxy 会把 trip context 交给 Hermes，Hermes 作为 MCP client 调用 Byway MCP tools。Proxy 不再用 regex 编排旅行流程。

Hermes 必须加载 Byway Skill：

`HERMES_AGENT_SKILLS=byway-family-travel-agent`

仓库源文件位于 `skills/byway-family-travel-agent/SKILL.md`。同步到本地 Hermes：

`scripts/sync-hermes-skill.sh`

验收时，每次聊天都会返回 `trace_started` SSE 事件。前端会显示 traceId，也可以直接打开：

`GET http://127.0.0.1:8787/api/traces/<traceId>`

Trace 中能看到 Byway Skill name/version/hash、Agent turn、MCP tool、Provider、SSE 的输入输出摘要。日志和 trace 默认会脱敏密钥、token、PIN、手机号和长文本。
```

- [ ] **Step 6: 运行测试确认通过**

Run:

```bash
pnpm vitest run services/proxy-api/src/proxyApi.test.ts -t "reports production readiness"
```

Expected: PASS。

---

## Task 14: 端到端验收和回归

**Files:**
- Modify: `playwright.config.ts` 如需增加 trace 验收 env。
- Modify: existing e2e specs only when current helpers cannot inspect trace.

- [ ] **Step 1: 跑单元测试**

Run:

```bash
pnpm test
```

Expected: PASS，包含 observability、agentRuntime、byway Skill contract、mcpProtocol、proxyApi 和 workspaceState 测试。

- [ ] **Step 2: 跑类型检查**

Run:

```bash
pnpm typecheck
```

Expected: PASS。

- [ ] **Step 3: 跑构建**

Run:

```bash
pnpm build
```

Expected: PASS。

- [ ] **Step 4: 跑 E2E**

Run:

```bash
pnpm test:e2e
```

Expected: PASS。

- [ ] **Step 5: 手动验收 Hermes 主链路**

启动：

```bash
scripts/sync-hermes-skill.sh
BYWAY_AGENT_RUNTIME=hermes_agent HERMES_AGENT_SKILLS=byway-family-travel-agent BYWAY_PLACE_PROVIDER=auto pnpm dev:all
```

在浏览器执行：

1. 输入“不知道去哪，带父母和两个孩子玩 4 天，别太累。”
2. 选择杭州。
3. 输入“第三天可以改成灵隐寺吗？”
4. 打开前端显示的 trace 链接。

Expected:

- 前端仍显示杭州旅行。
- trace 中有 `agent.skill.loaded`，并包含 `byway-family-travel-agent` 的 version 和 hash。
- trace 中有 `agent.turn.started`。
- trace 中有至少一个 Byway MCP `mcp.tool.started`。
- MCP tool input 或 Agent context 中的 destination 是杭州。
- 没有出现青岛，除非用户文本明确提到青岛。

---

## 完成标准

- `/api/chat` 主链路由 Hermes Agent Runtime 驱动。
- Hermes 每轮 Byway chat 都加载 `byway-family-travel-agent` Skill。
- Hermes 作为 MCP client 调 Byway MCP tools。
- Proxy 不再用 regex 分支编排旅行业务。
- Trace API 可查看每一轮 Agent、MCP、Provider、SSE 的输入输出摘要。
- 前端能显示当前 traceId。
- 生产配置不会静默使用 mock runtime。
- `pnpm test`、`pnpm test:api`、`pnpm typecheck`、`pnpm build`、`pnpm test:e2e` 全部通过。
