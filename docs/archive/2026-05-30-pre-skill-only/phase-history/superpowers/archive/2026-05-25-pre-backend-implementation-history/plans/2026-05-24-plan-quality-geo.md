# Plan Artifact 质量与地理核验 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 mock PlanArtifact 满足家庭旅行 anchor-first 质量要求，并让 `verify_plan_geo` 能识别明显跨区往返。

**Architecture:** 在 MCP mock planner 中集中增强 `makePlan` 与 `verify_plan_geo`。Proxy 保持现有 Plan orchestration，只把 geo warning 转成用户可见说明。Web 在现有 Plan renderer 内展示 warnings，不新增复杂 UI。

**Tech Stack:** TypeScript、Vitest、Playwright、Next.js。

---

## 文件结构

- 修改：`services/byway-mcp-server/src/index.ts`
  增强 `makePlan`、新增 meal anchor 校验 helper、增强 `verify_plan_geo`。

- 修改：`services/byway-mcp-server/src/bywayMcpServer.test.ts`
  增加 D1-D4 MCP 单测。

- 修改：`services/proxy-api/src/index.ts`
  将 geo warning 追加到 Plan Mode assistant message。

- 修改：`services/proxy-api/src/proxyApi.test.ts`
  增加 Plan Mode 质量断言。

- 修改：`apps/web/src/app/components/artifactRenderers.tsx`
  在 PlanArtifact 详情中展示 warnings。

- 修改：`apps/web/e2e/byway.spec.ts`
  增加计划质量和 warning 展示验收。

## 任务 1：MCP 红测

- [ ] **步骤 1：写失败测试**

在 `services/byway-mcp-server/src/bywayMcpServer.test.ts` 增加：

```ts
it("keeps meal anchors on every full travel day and lightens arrival/departure days", async () => {
  const mcp = await createBywayMcpServer();
  const { data: created } = await mcp.create_trip({ userId: "user_1", initialMessage: "青岛 4 天", source: "chat" });
  const tripId = created.tripId;
  await mcp.update_trip_brief({
    tripId,
    patch: { destination: "青岛", days: 4, arrivalInfo: "第一天下午 3 点到", departureInfo: "第四天下午 4 点返程", travelers: [], pace: "relaxed" },
    sourceMessageId: "msg"
  });

  const { data: plan } = await mcp.generate_plan({ tripId, canonicalGuideId: "cg", tripBrief: {}, placeFactIds: [], routeFactIds: [], familyMemoryIds: [], planningAssumptions: [] });
  const fullDays = plan.days.filter((day) => day.dayRole === "full_day");
  for (const day of fullDays) {
    expect(day.nodes.filter((node) => node.type === "meal" && node.anchor).map((node) => node.title).join(" ")).toContain("午餐");
    expect(day.nodes.filter((node) => node.type === "meal" && node.anchor).map((node) => node.title).join(" ")).toContain("晚餐");
  }
  expect(plan.days[0].nodes.map((node) => node.title).join(" ")).toContain("放行李");
  expect(plan.days[0].nodes.map((node) => node.title).join(" ")).not.toContain("崂山");
  expect(plan.days[3].nodes.map((node) => node.title).join(" ")).toContain("返程缓冲");
  expect(plan.days[3].nodes.map((node) => node.title).join(" ")).toContain("交通方向午餐");
});
```

再增加 `verify_plan_geo` 测试：把同一天 nodes 改成包含“崂山 + 栈桥 + 五四广场”，断言返回“跨区往返”和建议。

- [ ] **步骤 2：运行红测**

Run: `pnpm test -- services/byway-mcp-server/src/bywayMcpServer.test.ts`

Expected: FAIL，Day 3 缺 meal anchor，`verify_plan_geo` 无 warning。

- [ ] **步骤 3：实现 MCP**

在 `makePlan`：

- Day 3 增加 `day3_lunch`、`day3_dinner`。
- Day 1 节点标题包含“放行李”。
- Day 4 增加 `day4_buffer`、`day4_lunch`。
- 生成后调用 `withMealAnchorWarnings(plan)`，缺 meal anchor 时写 plan warning。

在 `verify_plan_geo`：

- 读取 plan。
- 对每一天检查 `崂山` + 市区点混排。
- 返回 `{ geoWarnings, overallGeoFit, suggestedFixes }`。

- [ ] **步骤 4：运行 green**

Run: `pnpm test -- services/byway-mcp-server/src/bywayMcpServer.test.ts`

Expected: PASS。

## 任务 2：Proxy 与 Web

- [ ] **步骤 1：写失败测试**

在 `services/proxy-api/src/proxyApi.test.ts` 增加断言：

```ts
expect(planEvent?.data.content.days[2].nodes.some((node) => node.type === "meal" && node.title.includes("午餐"))).toBe(true);
expect(planEvent?.data.content.days[2].nodes.some((node) => node.type === "meal" && node.title.includes("晚餐"))).toBe(true);
```

在 `apps/web/e2e/byway.spec.ts` 增加：

```ts
await expect(page.getByTestId("trip-workspace")).toContainText("崂山附近午餐");
await expect(page.getByTestId("trip-workspace")).toContainText("酒店附近晚餐");
```

- [ ] **步骤 2：运行红测**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts && pnpm exec playwright test --grep "Plan quality"`

Expected: FAIL。

- [ ] **步骤 3：实现 Proxy/Web**

Proxy Plan Mode 中读取 `verify_plan_geo` 返回值，如果有 warning，assistant message 增加“我也做了地理合理性核验...”。

Plan renderer 中展示：

- plan warnings
- day warnings

- [ ] **步骤 4：运行 green**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts && pnpm exec playwright test --grep "Plan quality"`

Expected: PASS。

## 任务 3：全量验证

- [ ] `pnpm typecheck`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] 停掉 3000/4000 dev server 后运行 `pnpm build`
- [ ] 重启 `pnpm dev:all`

## 自检

- D1-D4 都有测试覆盖。
- meal anchor 是 plan node，不是纯文案。
- geo warning 不承诺导航级最优。
- 前端展示 warnings 但不引入复杂地图 UI。
