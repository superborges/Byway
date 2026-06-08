# 核心地点解析与歧义确认 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 补齐 C1/C2，让 Plan Mode 解析核心地点并显式询问歧义地点。

**Architecture:** 复用现有 `resolve_places_batch` 和 PlaceFactReviewArtifact。只增强 Proxy 的 plan orchestration 和 mock PlanArtifact 的 placeFactId 引用。

**Tech Stack:** TypeScript、Vitest、Playwright。

---

## 文件结构

- 修改：`services/byway-mcp-server/src/index.ts`
  给 Day 3 崂山节点补 `placeFactId`。

- 修改：`services/proxy-api/src/index.ts`
  扩展 Plan Mode 的核心地点列表，并在 `needsReview` 时输出自然语言确认问题。

- 修改：`services/proxy-api/src/proxyApi.test.ts`
  增加 C1/C2 SSE 测试。

- 修改：`apps/web/e2e/byway.spec.ts`
  增加地点核验确认问题验收。

## 任务 1：红测

- [ ] **步骤 1：写 Proxy 失败测试**

新增测试：

```ts
it("resolves core places and asks user to confirm ambiguous place facts", async () => {
  const app = createProxyApp({ mcp: await createBywayMcpServer() });
  const response = await request(app).post("/api/chat").send({ message: { text: "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。", attachments: [] } }).expect(200);
  const events = parseSse(response.text);
  const placeResult = events.find((item) => item.event === "tool_call_completed" && item.data.toolName === "resolve_places_batch")?.data.result;
  expect(placeResult.data.resolved.map((place) => place.inputName)).toEqual(expect.arrayContaining(["八大关", "小麦岛", "崂山", "栈桥"]));
  expect(events.some((item) => item.event === "artifact_updated" && item.data.artifactType === "PlaceFactReviewArtifact")).toBe(true);
  expect(events.filter((item) => item.event === "assistant_message_delta").map((item) => item.data.delta).join("")).toContain("我找到了几个可能的“老街”");
});
```

- [ ] **步骤 2：写 E2E 失败测试**

新增测试：

```ts
test("Plan mode asks for ambiguous place confirmation", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await expect(page.locator("body")).toContainText("我找到了几个可能的“老街”");
  await expect(page.getByTestId("trip-workspace")).toContainText("地点核验");
});
```

- [ ] **步骤 3：运行红测**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts && pnpm exec playwright test --grep "ambiguous place"`

Expected: FAIL。

## 任务 2：实现

- [ ] **步骤 1：扩展地点列表**

Plan Mode 中 `resolve_places_batch` places 改为：

```ts
[
  { name: "八大关", typeHint: "activity", priority: 5 },
  { name: "小麦岛", typeHint: "activity", priority: 4 },
  { name: "崂山", typeHint: "activity", priority: 4 },
  { name: "栈桥", typeHint: "activity", priority: 4 },
  { name: "五四广场", typeHint: "meal_area", priority: 4 },
  { name: "老街", typeHint: "activity", priority: 2 }
]
```

- [ ] **步骤 2：输出确认问题**

`places.data.needsReview.length > 0` 时写 assistant delta：

```ts
我找到了几个可能的“老街”，你指的是哪一个？确认前我不会把它当作高置信度核心地点。
```

- [ ] **步骤 3：补 PlanArtifact placeFactId**

Day 3 崂山节点增加 `placeFactId: "pf_崂山"`。

- [ ] **步骤 4：运行 green**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts && pnpm exec playwright test --grep "ambiguous place"`

Expected: PASS。

## 任务 3：全量验证

- [ ] `pnpm typecheck`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] 停掉 3000/4000 dev server 后运行 `pnpm build`
- [ ] 重启 `pnpm dev:all`
