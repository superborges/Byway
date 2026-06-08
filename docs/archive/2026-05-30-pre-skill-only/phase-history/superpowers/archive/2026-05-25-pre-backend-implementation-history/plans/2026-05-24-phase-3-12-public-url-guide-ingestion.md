# Phase 3.12 公开 URL 攻略吸收 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 普通公开攻略 URL 可被 Byway 抽取正文片段并进入 Guide ingestion pipeline。

**Architecture:** Proxy 增加可注入的 `urlFetcher` 和轻量 HTML 文本抽取函数。Intent detection 将普通 URL 分到 `guide_url`，小红书仍保持 `guide_link` 安全降级。

**Tech Stack:** TypeScript、Express、native fetch、Vitest/Supertest。

---

## Task 1: 红灯测试

**Files:**
- Modify: `services/proxy-api/src/proxyApi.test.ts`

- [x] 注入 fake `urlFetcher` 返回 HTML。
- [x] 发送普通公开 URL。
- [x] 断言调用 guide ingestion、生成 GuideSummaryArtifact。
- [x] 断言 assistant 文案包含“已读取公开网页正文片段”。

## Task 2: Proxy URL Fetcher

**Files:**
- Modify: `services/proxy-api/src/index.ts`

- [x] `ProxyAppOptions` 增加 `urlFetcher`。
- [x] 增加 `extractReadableTextFromHtml`。
- [x] 增加 `fetchPublicUrlText`，限制正文长度。
- [x] `detectMode` 区分 `guide_url` 与 `guide_link`。
- [x] guide branch 对普通 URL 使用 `user_url` + extracted content。

## Task 3: 验证

- [x] `pnpm test:api`
- [x] `pnpm typecheck`
- [x] `pnpm test`
