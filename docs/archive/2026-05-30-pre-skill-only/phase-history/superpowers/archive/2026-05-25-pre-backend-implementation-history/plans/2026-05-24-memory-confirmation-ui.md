# Family Memory 确认交互 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 Family Memory confirmation 在前端使用正确按钮和成功文案。

**Architecture:** 只改 Web 层，复用现有 `/api/confirmations/:id/respond`。`ConfirmationBlock` 根据 confirmation type 选择按钮文本，`acceptConfirmation` 根据 type 选择成功消息。

**Tech Stack:** TypeScript、Next.js、Playwright。

---

## 文件结构

- 修改：`apps/web/src/app/page.tsx`
  根据 `activeConfirmation.confirmationType` 生成确认成功消息。

- 修改：`apps/web/src/app/components/workspaceSurfaces.tsx`
  ConfirmationBlock 的按钮文案区分“应用”和“接受”。

- 修改：`apps/web/e2e/byway.spec.ts`
  增加点击 Family Memory confirmation 的 E2E。

## 任务 1：红测

- [ ] **步骤 1：写失败 E2E**

在 Review test 中追加：

```ts
await page.getByRole("button", { name: "接受" }).click();
await expect(page.locator("body")).toContainText("已接受家庭记忆");
```

- [ ] **步骤 2：运行红测**

Run: `pnpm exec playwright test --grep "Review mode"`

Expected: FAIL，按钮仍是“应用”或成功文案错误。

## 任务 2：实现

- [ ] **步骤 1：按钮文案**

`ConfirmationBlock` 中：

```tsx
<span>{confirmation.confirmationType === "accept_memory" ? "接受" : "应用"}</span>
```

- [ ] **步骤 2：成功文案**

`acceptConfirmation` 中根据 confirmation type 生成 message：

```ts
const successText = activeConfirmation.confirmationType === "accept_memory"
  ? "已接受家庭记忆，下次推荐和计划会参考这条偏好。"
  : "已应用推荐调整，今天的计划会以保留晚餐和降低体力压力为优先。";
```

- [ ] **步骤 3：运行 green**

Run: `pnpm exec playwright test --grep "Review mode"`

Expected: PASS。

## 任务 3：全量验证

- [ ] `pnpm typecheck`
- [ ] `pnpm test`
- [ ] `pnpm test:e2e`
- [ ] 停掉 3000/4000 dev server 后运行 `pnpm build`
- [ ] 重启 `pnpm dev:all`
