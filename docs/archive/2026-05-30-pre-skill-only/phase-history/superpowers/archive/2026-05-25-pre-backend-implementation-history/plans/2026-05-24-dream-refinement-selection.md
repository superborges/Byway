# Dream 偏好重排与文字选择 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 补齐 A2/A3，让 Dream Mode 支持偏好补充重排和文字选择目的地。

**Architecture:** 主要改 Proxy mock orchestrator 的 intent routing。MCP 既有 Dream/Plan 工具已足够，必要时在 Proxy 中对 mock candidates 做轻量重排。前端复用现有 artifact preview 和 workspace。

**Tech Stack:** TypeScript、Express、Vitest、Playwright。

---

## 文件结构

- 修改：`services/proxy-api/src/index.ts`
  增加 `dream_refine`、`destination_select` intent 和对应 orchestration。

- 修改：`services/proxy-api/src/proxyApi.test.ts`
  增加 A2/A3 SSE 测试。

- 修改：`apps/web/e2e/byway.spec.ts`
  增加 A2/A3 页面验收。

## 任务 1：Proxy 红测与实现

- [ ] **步骤 1：写失败测试**

在 `services/proxy-api/src/proxyApi.test.ts` 增加：

```ts
it("refines Dream candidates when user adds sea and meal preferences", async () => {
  const app = createProxyApp({ mcp: await createBywayMcpServer() });
  const dream = await request(app).post("/api/chat").send({ message: { text: "我想暑假从北京出发，带父母和两个孩子玩 4 天，不知道去哪，别太累。", attachments: [] } });
  const tripId = parseSse(dream.text).find((item) => item.event === "assistant_message_completed")?.data.tripId;

  const response = await request(app).post("/api/chat").send({ tripId, message: { text: "更想去海边，最好吃饭方便。", attachments: [] } }).expect(200);
  const events = parseSse(response.text);
  expect(events.some((item) => item.event === "tool_call_started" && item.data.toolName === "score_destination_candidates")).toBe(true);
  expect(events.some((item) => item.event === "artifact_updated" && item.data.artifactType === "DestinationShortlistArtifact")).toBe(true);
  expect(events.filter((item) => item.event === "assistant_message_delta").map((item) => item.data.delta).join("")).toContain("海边和吃饭方便");
});
```

再增加文字选择测试，断言 `select_destination` 和 `PlanArtifact`。

- [ ] **步骤 2：运行红测**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts`

Expected: FAIL，当前会误入 Plan Mode。

- [ ] **步骤 3：实现 Proxy**

在 `detectMode` 中加入：

- `dream_refine`
- `destination_select`

新增 helper：

- `inferDestinationChoice(text)`
- `refineCandidateScores(candidates, text)`

实现两个 branch。

- [ ] **步骤 4：运行 green**

Run: `pnpm test -- services/proxy-api/src/proxyApi.test.ts`

Expected: PASS。

## 任务 2：E2E

- [ ] **步骤 1：写失败 E2E**

在 `apps/web/e2e/byway.spec.ts` 增加：

```ts
test("Dream mode refines candidates and supports text destination selection", async ({ page }) => {
  await page.goto("/");
  await sendMessage(page, "我想暑假从北京出发，带父母和两个孩子玩 4 天，不知道去哪，别太累。");
  await sendMessage(page, "更想去海边，最好吃饭方便。");
  await expect(page.locator("body")).toContainText("海边和吃饭方便");

  await sendMessage(page, "那就青岛吧。");
  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");
});
```

- [ ] **步骤 2：运行红测**

Run: `pnpm exec playwright test --grep "Dream mode refines"`

Expected: FAIL。

- [ ] **步骤 3：运行 green**

Run: `pnpm exec playwright test --grep "Dream mode refines"`

Expected: PASS。

## 任务 3：全量验证

- [ ] `pnpm typecheck`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] 停掉 3000/4000 dev server 后运行 `pnpm build`
- [ ] 重启 `pnpm dev:all`
