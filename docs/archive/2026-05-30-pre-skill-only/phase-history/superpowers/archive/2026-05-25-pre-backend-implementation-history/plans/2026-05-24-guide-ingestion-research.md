# 攻略研究与用户攻略吸收 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 补齐 Plan Mode 的攻略研究层，并支持用户粘贴 markdown 攻略和小红书链接作为 GuideSource。

**Architecture:** `packages/shared-schemas` 先定义 GuideSource、GuideExtraction、GuideSummaryArtifact。MCP mock server 负责保存来源、抽取结构化攻略、合并为 GuideSummaryArtifact。Proxy API 识别 plan / markdown / link 三种入口，并通过 SSE 输出工具进度、artifact 更新和可信说明。

**Tech Stack:** TypeScript、Zod、Express、Next.js、Vitest、Playwright。

---

## 文件结构

- 修改：`packages/shared-schemas/src/index.ts`
  增加攻略相关 Zod schema 和导出类型。

- 修改：`packages/shared-schemas/src/index.test.ts`
  增加 GuideSource / GuideSummaryArtifact schema 单测。

- 修改：`services/byway-mcp-server/src/index.ts`
  增加 `guideSources`、`guideExtractions`、`canonicalGuides` 存储集合，以及 Guide 工具实现。

- 修改：`services/byway-mcp-server/src/bywayMcpServer.test.ts`
  增加 synthetic guide、markdown ingest、link ingest、merge 的 MCP 单测。

- 修改：`services/proxy-api/src/index.ts`
  增加 guide intent detection 和 Plan Mode 的 guide orchestration。

- 修改：`services/proxy-api/src/proxyApi.test.ts`
  增加 B1/B3/B4 的 SSE 测试。

- 修改：`apps/web/src/app/page.tsx`
  增加新工具中文标签。

- 修改：`apps/web/src/app/components/artifactRenderers.tsx`
  增加 GuideSummaryArtifact 渲染。

- 修改：`apps/web/e2e/byway.spec.ts`
  增加 markdown 攻略和小红书链接验收。

## 任务 1：Shared Schema 红测与实现

- [ ] **步骤 1：写失败测试**

在 `packages/shared-schemas/src/index.test.ts` 增加：

```ts
it("accepts guide source and guide summary artifact contracts", () => {
  const source = guideSourceSchema.parse({
    id: "guide_user_1",
    tripId: "trip_1",
    sourceType: "user_markdown",
    title: "青岛攻略",
    content: "# 青岛四天三晚亲子攻略",
    extractionStatus: "extracted",
    createdAt: "2026-05-24T00:00:00.000Z"
  });
  expect(source.sourceType).toBe("user_markdown");

  const summary = guideSummaryArtifactSchema.parse({
    id: "guide_summary_1",
    tripId: "trip_1",
    canonicalGuideId: "cg_1",
    version: 1,
    sourceIds: ["guide_user_1"],
    lodgingAreas: [{ name: "五四广场 / 奥帆中心", recommendationLevel: "high", reasons: ["吃饭和打车方便"], risks: [] }],
    mealAreas: [{ name: "酒店附近 20 分钟交通圈", recommendationLevel: "high", reasons: ["保留用餐 anchor"], risks: [] }],
    places: [{ name: "八大关", finalPriority: 5, reasons: ["亲子友好"], warnings: [] }],
    routeIdeas: [{ title: "老城 + 八大关", placeNames: ["八大关"], intensity: "low", confidence: 0.8 }],
    conflicts: [{ description: "崂山是否安排存在分歧", sources: ["guide_user_1"], suggestedResolution: "保留市区备选" }],
    warnings: ["崂山对老人孩子可能偏累"],
    revisionSummary: "新增 1 个地点，强化 1 个住宿区域，发现 1 个冲突",
    changeSummary: { added: ["八大关"], strengthened: ["五四广场住宿"], conflicts: ["崂山强度"] },
    createdAt: "2026-05-24T00:00:00.000Z"
  });
  expect(summary.changeSummary.added).toContain("八大关");
});
```

- [ ] **步骤 2：运行红测**

Run: `pnpm test -- packages/shared-schemas/src/index.test.ts`

Expected: FAIL，提示 `guideSourceSchema` 或 `guideSummaryArtifactSchema` 未导出。

- [ ] **步骤 3：实现最小 schema**

在 `packages/shared-schemas/src/index.ts` 增加：

```ts
export const guideSourceSchema = z.object({
  id: z.string(),
  tripId: z.string(),
  sourceType: z.enum(["model_generated", "user_markdown", "user_text", "user_url", "xiaohongshu_link", "screenshot", "manual_note"]),
  title: z.string().optional(),
  content: z.string().optional(),
  url: z.string().optional(),
  attachments: z.array(z.unknown()).optional(),
  provider: z.string().optional(),
  model: z.string().optional(),
  promptRole: z.string().optional(),
  userNote: z.string().optional(),
  extractionStatus: z.enum(["pending", "extracted", "failed"]),
  createdAt: isoDateStringSchema
});
```

并定义 `guideExtractionSchema`、`guideSummaryArtifactSchema`，导出 `GuideSource`、`GuideExtraction`、`GuideSummaryArtifact`。

- [ ] **步骤 4：运行 green**

Run: `pnpm test -- packages/shared-schemas/src/index.test.ts`

Expected: PASS。

## 任务 2：MCP Guide 工具

- [ ] **步骤 1：写失败测试**

在 `services/byway-mcp-server/src/bywayMcpServer.test.ts` 增加测试：

```ts
it("generates, ingests, and merges guide sources into a guide summary artifact", async () => {
  const mcp = await createBywayMcpServer();
  const { data: created } = await mcp.create_trip({ userId: "user_1", initialMessage: "青岛 4 天", source: "chat" });
  const tripId = created.tripId;

  const synthetic = await mcp.generate_synthetic_guides({
    tripId,
    destination: "青岛",
    tripBrief: {},
    guideRoles: ["老人孩子友好", "住宿区域"],
    llmProviders: ["mock"]
  });
  expect(synthetic.data.sources).toHaveLength(2);

  const markdown = await mcp.ingest_user_guide_source({
    tripId,
    sourceType: "user_markdown",
    title: "青岛亲子攻略",
    content: "# 青岛四天三晚亲子攻略\n五四广场住宿方便，八大关适合轻松散步，崂山可能偏累。",
    url: null,
    attachments: [],
    userNote: "重点看住宿和老人孩子强度"
  });
  expect(markdown.data.extraction.places.map((place) => place.name)).toContain("八大关");

  const merged = await mcp.merge_guides({
    tripId,
    sourceIds: [...synthetic.data.sources.map((source) => source.sourceId), markdown.data.sourceId],
    preserveUserOverrides: true
  });
  expect(merged.data.artifactType).toBe("GuideSummaryArtifact");
  expect(merged.data.revisionSummary).toContain("新增");
  expect(merged.artifactUpdates[0]?.artifactType).toBe("GuideSummaryArtifact");
});
```

- [ ] **步骤 2：运行红测**

Run: `pnpm test -- services/byway-mcp-server/src/bywayMcpServer.test.ts`

Expected: FAIL，提示 guide 工具不存在。

- [ ] **步骤 3：实现 MCP 工具**

在 `services/byway-mcp-server/src/index.ts`：

- 扩展 `Collection`。
- 增加 mock extraction helper。
- 实现 `generate_synthetic_guides`。
- 实现 `ingest_user_guide_source`。
- 实现 `merge_guides` 并保存 `GuideSummaryArtifact` snapshot。

- [ ] **步骤 4：运行 green**

Run: `pnpm test -- services/byway-mcp-server/src/bywayMcpServer.test.ts`

Expected: PASS。

## 任务 3：Proxy Guide Orchestration

- [ ] **步骤 1：写失败测试**

在 `services/proxy-api/src/proxyApi.test.ts` 增加：

```ts
it("runs guide research before generating a plan", async () => {
  const app = createProxyApp({ mcp: await createBywayMcpServer() });
  const response = await request(app)
    .post("/api/chat")
    .send({ message: { text: "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。", attachments: [] }, clientContext: { timezone: "Asia/Shanghai" } })
    .expect(200);

  const events = parseSse(response.text);
  expect(events.some((item) => item.event === "tool_call_started" && item.data.toolName === "generate_synthetic_guides")).toBe(true);
  expect(events.some((item) => item.event === "tool_call_started" && item.data.toolName === "merge_guides")).toBe(true);
  expect(events.some((item) => item.event === "tool_call_started" && item.data.toolName === "verify_plan_geo")).toBe(true);
  expect(events.some((item) => item.event === "artifact_updated" && item.data.artifactType === "GuideSummaryArtifact")).toBe(true);
});
```

再增加 markdown 和 link 两个测试，断言 assistant 文本包含“新增 / 强化 / 冲突”和“已保存链接 / 无法读取完整笔记 / 五四广场住宿”。

- [ ] **步骤 2：运行红测**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts`

Expected: FAIL，提示新工具事件不存在。

- [ ] **步骤 3：实现 Proxy**

在 `services/proxy-api/src/index.ts`：

- `detectMode` 增加 `guide_ingest` 和 `guide_link`。
- Plan branch 中先调用 `generate_synthetic_guides` / `merge_guides`，再生成 plan，并在 plan 后调用 `verify_plan_geo`。
- 新增 guide ingest branch。
- 新增 xiaohongshu link branch。

- [ ] **步骤 4：运行 green**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts`

Expected: PASS。

## 任务 4：Web 展示与 E2E

- [ ] **步骤 1：写失败 E2E**

在 `apps/web/e2e/byway.spec.ts` 增加：

```ts
test("Guide ingest summarizes markdown changes", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await sendMessage(page, "# 青岛四天三晚亲子攻略\n五四广场住宿方便，八大关适合轻松散步，崂山可能偏累。");
  await expect(page.getByText("新增")).toBeVisible();
  await expect(page.getByText("强化")).toBeVisible();
  await expect(page.getByText("是否基于新攻略更新当前计划")).toBeVisible();
});
```

再增加小红书链接用例，断言页面出现“已保存链接”“无法读取完整笔记”“五四广场住宿”。

- [ ] **步骤 2：运行红测**

Run: `pnpm exec playwright test --grep "Guide"`

Expected: FAIL，新文本不可见。

- [ ] **步骤 3：实现 Web 展示**

在 `apps/web/src/app/page.tsx` 增加工具标签：

```ts
generate_synthetic_guides: "生成攻略研究",
ingest_user_guide_source: "吸收用户攻略",
merge_guides: "合并攻略底稿",
verify_plan_geo: "验证路线地理合理性"
```

在 `artifactRenderers.tsx` 增加 `GuideSummaryArtifact` 分支，展示 `revisionSummary`、`changeSummary`、`warnings`。

- [ ] **步骤 4：运行 green**

Run: `pnpm exec playwright test --grep "Guide"`

Expected: PASS。

## 任务 5：全量验证

- [ ] `pnpm typecheck`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] 停掉 3000/4000 dev server 后运行 `pnpm build`
- [ ] 重启 `pnpm dev:all`

## 自检

- B1 调用了攻略研究和地理验证。
- B3 能吸收 markdown 并返回变化摘要。
- B4 明确不声称读完小红书正文。
- 不引入真实外部网络依赖。
- 不自动应用重大计划变更。
