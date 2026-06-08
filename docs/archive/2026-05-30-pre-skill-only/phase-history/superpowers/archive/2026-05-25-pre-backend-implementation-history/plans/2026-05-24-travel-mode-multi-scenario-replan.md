# Travel Mode 多场景重排增强 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 扩展 Byway Travel Mode mock，使饥饿、下雨、孩子不想坐车、拒绝推荐方案四类现场反馈都能生成对应的三类重排方案并等待确认。

**Architecture:** Proxy API 负责把用户输入分类为 `TravelEventType` 和 context；MCP mock server 根据已记录的 `TravelEvent` 生成场景化 `ReplanOptionsArtifact`。前端复用现有 Trip Workspace 和 ReplanOptions 渲染，不新增专用 UI。

**Tech Stack:** TypeScript、Express Proxy API、Byway MCP mock server、Zod shared schema、Vitest、Playwright。

---

## 文件结构

- 修改：`apps/web/e2e/byway.spec.ts`
  新增 E2-E5 端到端场景。

- 修改：`services/proxy-api/src/index.ts`
  新增 `inferTravelEvent`，替换当前简单三元分类。

- 修改：`services/proxy-api/src/proxyApi.test.ts`
  验证 Proxy 对饥饿、下雨、孩子不想坐车的分类和 artifact 输出。

- 修改：`services/byway-mcp-server/src/index.ts`
  `replan_today` 根据 TravelEvent 生成不同 mock options。

- 修改：`services/byway-mcp-server/src/bywayMcpServer.test.ts`
  验证 MCP 对各 TravelEventType 的重排策略。

## 任务 1：写红测

**文件：**

- 修改：`apps/web/e2e/byway.spec.ts`
- 修改：`services/byway-mcp-server/src/bywayMcpServer.test.ts`
- 修改：`services/proxy-api/src/proxyApi.test.ts`

- [ ] **步骤 1：新增 E2E 饥饿场景**

```ts
test("Travel mode prioritizes meal anchors when everyone is hungry", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");

  await sendMessage(page, "现在大家都饿了。");

  await expect(page.getByTestId("trip-workspace")).toContainText("先去吃饭");
  await expect(page.getByTestId("trip-workspace")).toContainText("优先保障 meal anchor");
  await expect(page.getByTestId("trip-workspace")).toContainText("是否应用推荐调整方案");
});
```

- [ ] **步骤 2：新增 E2E 天气和交通场景**

```ts
test("Travel mode handles rain and child transport fatigue", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");

  await sendMessage(page, "下雨了，下午怎么安排？");
  await expect(page.getByTestId("trip-workspace")).toContainText("室内/低风险活动");
  await expect(page.getByTestId("trip-workspace")).toContainText("取消室外低优先级景点");

  await sendMessage(page, "孩子不想坐车了。");
  await expect(page.getByTestId("trip-workspace")).toContainText("减少跨区移动");
  await expect(page.getByTestId("trip-workspace")).toContainText("酒店附近轻活动");
});
```

- [ ] **步骤 3：新增 E2E 拒绝推荐方案**

```ts
test("Travel mode regenerates options when user rejects the recommended replan", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");

  await sendMessage(page, "老人累了，我们晚了 40 分钟。");
  await expect(page.getByTestId("trip-workspace")).toContainText("取消下一个低优先级景点");

  await sendMessage(page, "不，我还是想去小鱼山，取消别的。");
  await expect(page.getByTestId("trip-workspace")).toContainText("保留小鱼山");
  await expect(page.getByTestId("trip-workspace")).toContainText("取消五四广场附近备选活动");
  await expect(page.getByTestId("trip-workspace")).toContainText("选中方案：opt_keep_xiaoyushan");
});
```

- [ ] **步骤 4：运行红测**

```bash
pnpm exec playwright test --grep "hungry|rain|rejects the recommended"
```

预期：失败，因为当前 mock 只生成通用 E1 方案。

## 任务 2：MCP 重排策略

**文件：**

- 修改：`services/byway-mcp-server/src/index.ts`
- 修改：`services/byway-mcp-server/src/bywayMcpServer.test.ts`

- [ ] **步骤 1：添加 MCP 单测**

新增测试覆盖 `hungry`、`bad_weather`、`transport_problem`、`custom + 小鱼山` 四类事件。

- [ ] **步骤 2：在 MCP 中读取 TravelEvent**

`replan_today` 使用：

```ts
const event = this.store.get<TravelEvent>("travelEvents", input.eventId);
```

找不到事件时回退到当前默认 E1 策略。

- [ ] **步骤 3：新增 `replanOptionsForEvent` helper**

根据 `event.eventType` 和 `event.userText` 返回 `{ recommendedOptionId, options }`。

- [ ] **步骤 4：运行 MCP 测试**

```bash
pnpm test -- services/byway-mcp-server/src/bywayMcpServer.test.ts
```

预期：通过。

## 任务 3：Proxy 事件分类

**文件：**

- 修改：`services/proxy-api/src/index.ts`
- 修改：`services/proxy-api/src/proxyApi.test.ts`

- [ ] **步骤 1：新增 `inferTravelEvent`**

分类规则：

```ts
if (/(饿|吃饭|午饭|晚饭)/.test(text)) return hungry;
if (/(下雨|暴雨|天气)/.test(text)) return bad_weather;
if (/(不想坐车|坐车太久|少坐车)/.test(text)) return transport_problem;
if (/(孩子累|孩子困)/.test(text)) return child_tired;
if (/(老人累|父母累)/.test(text)) return elder_tired;
if (/(晚了|迟到)/.test(text)) return late;
return custom;
```

- [ ] **步骤 2：替换 Travel branch 里的 eventType 三元表达式**

用 `inferTravelEvent(text)` 的结果传给 `record_travel_event`。

- [ ] **步骤 3：运行 API 测试**

```bash
pnpm test:api
```

预期：通过。

## 任务 4：E2E 转绿与完整验证

- [ ] **步骤 1：运行 targeted E2E**

```bash
pnpm exec playwright test --grep "hungry|rain|rejects the recommended"
```

- [ ] **步骤 2：类型检查**

```bash
pnpm typecheck
```

- [ ] **步骤 3：单元测试**

```bash
pnpm test
```

- [ ] **步骤 4：全量 E2E**

```bash
pnpm test:e2e
```

- [ ] **步骤 5：构建**

```bash
pnpm build
```

## 自检

- 覆盖 E2-E5：饥饿、下雨、孩子不想坐车、拒绝推荐方案均有 E2E。
- 范围控制：不接真实外部服务，不改前端布局。
- 确认规则：所有 replan 仍生成 pending confirmation，用户确认前不应用。
