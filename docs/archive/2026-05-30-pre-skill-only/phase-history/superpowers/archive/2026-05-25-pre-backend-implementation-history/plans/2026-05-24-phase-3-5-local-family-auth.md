# Phase 3.5 本地家庭 PIN / Session Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 配置 `BYWAY_AUTH_PIN` 后，Byway Proxy API 需要家庭 session token 才能访问，前端可完成 PIN 登录。

**Architecture:** Proxy 增加轻量 auth helper 和 middleware；token 使用 HMAC 生成并通过 Bearer header 校验。Web 用 localStorage 保存 token，所有 fetch 统一带 Authorization；auth 未启用时 UI 不出现。

**Tech Stack:** TypeScript、Express middleware、Node crypto、React localStorage、Vitest/Supertest。

---

## Task 1: Proxy Auth API

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] 写失败测试：配置 PIN 后，未带 token 的 `GET /api/trips` 返回 401。
- [ ] 写失败测试：`POST /api/auth/session` 错误 PIN 返回 401，正确 PIN 返回 token。
- [ ] 写失败测试：带正确 Bearer token 的 `GET /api/trips` 返回 200。
- [ ] 实现 `createAuthConfig`、`issueAuthToken`、`isAuthorizedRequest` 和 middleware。

## Task 2: Frontend PIN Session

**Files:**
- Modify: `apps/web/src/app/sessionStorage.ts`
- Modify: `apps/web/src/app/page.tsx`

- [ ] 增加 auth token localStorage helper。
- [ ] 页面加载时请求 `/api/auth/status`。
- [ ] auth enabled 且未认证时展示 PIN 表单。
- [ ] 成功登录后保存 token，并恢复正常工作台。
- [ ] 所有 API fetch 带上 Authorization header。

## Task 3: 配置与文档

**Files:**
- Modify: `.env.example`
- Modify: `README.md`

- [ ] 增加 `BYWAY_AUTH_PIN` 和 `BYWAY_AUTH_SECRET` 示例。
- [ ] README 说明未配置 PIN 时 auth disabled，配置后 API 受保护。

## 验收

- `pnpm test:api`
- `pnpm typecheck`
- `pnpm test`
- `pnpm build`
