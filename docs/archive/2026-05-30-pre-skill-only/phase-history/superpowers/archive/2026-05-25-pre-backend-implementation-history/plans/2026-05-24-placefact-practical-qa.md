# PlaceFact 实用问答与打车地址 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 PlaceFact 营业时间问答和打车地址建议，补齐验收场景 C3/C4。

**Architecture:** MCP mock server 提供 `get_place_detail` 和 `suggest_taxi_address`。Proxy API 识别地点事实问题，先解析地点，再调用对应工具，把结构化结果转成 assistant message。前端复用现有聊天流和工具进度。

**Tech Stack:** TypeScript、Express Proxy API、Byway MCP mock server、Vitest、Playwright。

---

## 文件结构

- 修改：`services/byway-mcp-server/src/index.ts`
  增加 PlaceFact detail 和 taxi address 工具。

- 修改：`services/byway-mcp-server/src/bywayMcpServer.test.ts`
  增加 MCP 工具单测。

- 修改：`services/proxy-api/src/index.ts`
  增加 place detail / taxi intent routing。

- 修改：`services/proxy-api/src/proxyApi.test.ts`
  增加 API SSE 测试。

- 修改：`apps/web/e2e/byway.spec.ts`
  增加 C3/C4 端到端测试。

## 任务 1：红测

- [ ] **步骤 1：MCP 单测**

验证：

```ts
const detail = await mcp.get_place_detail({ placeFactId, fields: ["openingHoursToday"] });
expect(detail.data.openingHoursToday).toContain("全天开放");

const taxi = await mcp.suggest_taxi_address({ placeFactId, context: { travelers: ["elder", "child"], preferredMode: "taxi" } });
expect(taxi.data.taxiAddress).toContain("大河东客服中心入口");
```

- [ ] **步骤 2：Proxy API 测试**

验证 SSE 文本包含：

- 八大关：`来源：amap_mock`、`置信度：medium`
- 崂山：`大河东客服中心入口`、`高德导航 POI`

- [ ] **步骤 3：E2E 测试**

新增两个 Playwright 用例：

```ts
test("PlaceFact answers opening hours with source and confidence", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "八大关几点关门？");
  await expect(page.getByText("全天开放")).toBeVisible();
  await expect(page.getByText("来源：amap_mock")).toBeVisible();
  await expect(page.getByText("置信度：medium")).toBeVisible();
});
```

```ts
test("PlaceFact suggests taxi address for Laoshan", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "我现在要打车去崂山，打到哪里？");
  await expect(page.getByText("大河东客服中心入口")).toBeVisible();
  await expect(page.getByText("入口较多")).toBeVisible();
  await expect(page.getByText("高德导航 POI")).toBeVisible();
});
```

## 任务 2：MCP 工具实现

- [ ] **步骤 1：扩展 mockPlace**

为八大关和崂山补充：

- `openingHoursToday`
- `rating`
- `businessArea`
- `taxiAddress`
- `entranceCoordinate`
- `navigationPoiId`

- [ ] **步骤 2：实现 `get_place_detail`**

从 `placeFacts` 读取，字段缺失返回 `null`，并返回 `fieldConfidence`。

- [ ] **步骤 3：实现 `suggest_taxi_address`**

返回打车地址、入口坐标、导航 POI 和 warning。

## 任务 3：Proxy 路由实现

- [ ] **步骤 1：新增 `detectMode` 分支**

在 plan/travel 之前识别：

- `place_detail`
- `taxi_address`

- [ ] **步骤 2：新增地点名提取 helper**

支持八大关、崂山、小麦岛、栈桥。

- [ ] **步骤 3：在 `/api/chat` 增加分支**

调用 `resolve_places_batch` 后调用 detail/taxi 工具，并返回 assistant message。

## 任务 4：验证

- [ ] `pnpm typecheck`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] `pnpm build`

## 自检

- C3/C4 均覆盖。
- 不接真实外部服务。
- 字段缺失路径明确。
- 前端不需要新增复杂 UI。
