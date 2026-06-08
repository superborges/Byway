# 会话恢复与工具进度实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：**让 Byway Web 在刷新后恢复当前旅行工作区，并在聊天流中展示 MCP 工具调用进度。

**架构：**前端新增轻量 session persistence：`localStorage` 保存聊天骨架和最近 Trip ID，刷新后通过 Proxy API 拉取 Trip state 和 artifact snapshots 作为权威内容。SSE 的 `tool_call_started` / `tool_call_completed` 转为 assistant message 下的 tool progress item，不进入 Trip Workspace。

**技术栈：**Next.js App Router、React client component、TypeScript、Playwright、Vitest、现有 Proxy API 与 MCP mock server。

---

## 文件结构

- 修改：`apps/web/e2e/byway.spec.ts`
  新增工具进度和刷新恢复的 E2E。

- 修改：`apps/web/src/app/workspaceState.ts`
  扩展 `ChatItem`，加入 tool progress 类型；新增 session snapshot、artifact snapshot normalize helper。

- 新建：`apps/web/src/app/sessionStorage.ts`
  封装 `localStorage` 读写、版本校验、失效清理。

- 修改：`apps/web/src/app/page.tsx`
  在加载时恢复 session；处理 tool call SSE；在关键状态变化后保存 snapshot。

- 修改：`apps/web/src/app/components/ChatThread.tsx`
  渲染 tool progress list。

- 修改：`apps/web/src/app/globals.css`
  增加 tool progress 样式。

- 修改：`services/proxy-api/src/index.ts`
  让 `GET /api/trips/:tripId` 在返回 pending confirmations 时包含足够的前端字段；必要时从 pending payload 重建确认提交索引。

## 任务 1：E2E 红测

**文件：**

- 修改：`apps/web/e2e/byway.spec.ts`

- [ ] **步骤 1：添加工具进度断言**

在 Dream Mode 测试中，发送消息后断言：

```ts
await expect(page.getByTestId("tool-progress")).toContainText("创建旅行");
await expect(page.getByTestId("tool-progress")).toContainText("生成目的地候选");
```

- [ ] **步骤 2：添加刷新恢复计划测试**

新增：

```ts
test("refresh restores the current trip plan workspace", async ({ page }) => {
  await page.goto("/");

  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");

  await page.reload();

  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");
  await expect(page.getByTestId("trip-workspace")).toContainText("全程计划");
  await expect(page.getByTestId("trip-workspace")).toContainText("第一天下午到");
});
```

- [ ] **步骤 3：添加刷新恢复重排确认测试**

新增：

```ts
test("refresh restores a pending replan confirmation", async ({ page }) => {
  await page.goto("/");

  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");

  await sendMessage(page, "老人累了，我们晚了 40 分钟。");
  await expect(page.getByTestId("trip-workspace")).toContainText("是否应用推荐调整方案");

  await page.reload();

  await expect(page.getByTestId("trip-workspace")).toContainText("今日重排方案");
  await expect(page.getByTestId("trip-workspace")).toContainText("是否应用推荐调整方案");
  await expect(page.getByTestId("trip-workspace")).toContainText("选中方案：opt_cancel");
});
```

- [ ] **步骤 4：运行红测**

```bash
pnpm exec playwright test --grep "tool progress|refresh restores"
```

预期：新增用例失败，原因是没有 `tool-progress` 或刷新后没有恢复 artifact/workspace。

## 任务 2：状态模型和本地存储

**文件：**

- 修改：`apps/web/src/app/workspaceState.ts`
- 新建：`apps/web/src/app/sessionStorage.ts`

- [ ] **步骤 1：扩展状态类型**

加入 `ToolProgressItem`、`StoredSessionSnapshot`、`ArtifactSnapshotPayload`、`normalizeArtifactSnapshot`、`workspaceIdsFromArtifacts`。

- [ ] **步骤 2：封装 localStorage**

`sessionStorage.ts` 提供：

```ts
export const loadStoredSession = (): StoredSessionSnapshot | undefined;
export const saveStoredSession = (snapshot: StoredSessionSnapshot) => void;
export const clearStoredSession = () => void;
```

服务端渲染阶段或 `localStorage` 不可用时安全返回。

- [ ] **步骤 3：运行类型检查**

```bash
pnpm typecheck
```

预期：通过。

## 任务 3：前端恢复与工具进度

**文件：**

- 修改：`apps/web/src/app/page.tsx`
- 修改：`apps/web/src/app/components/ChatThread.tsx`
- 修改：`apps/web/src/app/globals.css`

- [ ] **步骤 1：页面加载恢复 snapshot**

`useEffect` 读取本地 snapshot，先恢复消息和 `tripId`。

- [ ] **步骤 2：从 Proxy 拉权威 artifacts**

如果 snapshot 有 `tripId`，调用：

```ts
GET /api/trips/:tripId
GET /api/trips/:tripId/artifacts
```

用 artifacts 重建 workspace，并恢复 pending confirmation。

- [ ] **步骤 3：状态变化后保存 snapshot**

当 `tripId`、`messages`、`activeArtifactId`、`workspaceMode`、`currentPlanArtifactId`、`todayArtifactId` 变化时写入 `localStorage`。

- [ ] **步骤 4：处理 tool call SSE**

`tool_call_started` 追加 `running` item；`tool_call_completed` 更新为 `completed`。

- [ ] **步骤 5：渲染 tool progress**

`ChatThread` 在 bubble 下展示 tool progress list，使用 `data-testid="tool-progress"`。

- [ ] **步骤 6：运行 targeted E2E**

```bash
pnpm exec playwright test --grep "tool progress|refresh restores"
```

预期：通过。

## 任务 4：确认恢复的后端兜底

**文件：**

- 修改：`services/proxy-api/src/index.ts`
- 修改：`services/proxy-api/src/proxyApi.test.ts`

- [ ] **步骤 1：补 REST 测试**

验证 Travel Mode 后调用 `GET /api/trips/:tripId` 能返回 pending `apply_replan` confirmation。

- [ ] **步骤 2：确认索引兜底**

当 `/api/confirmations/:id/respond` 找不到内存索引时，从 MCP pending confirmation 的 payload 尝试恢复 `apply_replan` 或 `accept_memory`。

- [ ] **步骤 3：运行 API 测试**

```bash
pnpm test:api
```

预期：通过。

## 任务 5：完整验证

- [ ] **步骤 1：类型检查**

```bash
pnpm typecheck
```

- [ ] **步骤 2：单元测试**

```bash
pnpm test
```

- [ ] **步骤 3：E2E**

```bash
pnpm test:e2e
```

- [ ] **步骤 4：构建**

```bash
pnpm build
```

- [ ] **步骤 5：浏览器验收**

打开 `http://127.0.0.1:3000/`，手动验证：

```text
Plan -> 刷新 -> 全程计划仍在
Travel -> 刷新 -> 重排确认仍在
聊天里能看到工具进度
```

## 自检

- 设计覆盖：刷新恢复、artifact 重建、pending confirmation、工具进度均有任务覆盖。
- 范围控制：不做登录、多 Trip 管理、服务端消息持久化。
- 类型一致：所有新增状态从 `workspaceState.ts` 统一导出。
