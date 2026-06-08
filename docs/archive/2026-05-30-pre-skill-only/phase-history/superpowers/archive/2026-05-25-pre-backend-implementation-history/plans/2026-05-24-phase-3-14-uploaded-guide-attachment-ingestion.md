# Phase 3.14 上传攻略附件吸收 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:test-driven-development. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 本地上传的攻略文本/Markdown 可以通过聊天 attachments 进入 Guide ingestion pipeline。

**Architecture:** Proxy 复用已有 `LocalUploadStore`，新增 attachment reader。Guide branch 将附件正文、公开 URL 正文、用户文本合并为 source content，并在 assistant 文案中说明读取状态。

**Tech Stack:** TypeScript、Express、Supertest、Vitest。

---

## Task 1: 红灯测试

**Files:**
- Modify: `services/proxy-api/src/proxyApi.test.ts`

- [x] 通过 `POST /api/uploads` 上传 Markdown 攻略。
- [x] 聊天 message 带 `attachments: [{ fileId }]`。
- [x] 断言调用 `ingest_user_guide_source` 和生成 GuideSummaryArtifact。
- [x] 断言 assistant 文案包含“已读取上传附件”。
- [x] 断言 GuideSummaryArtifact 含附件中的住宿/地点信息。

## Task 2: Attachment Reader

**Files:**
- Modify: `services/proxy-api/src/index.ts`

- [x] 从 attachments 中安全提取 `fileId`。
- [x] 只读取 text-like MIME。
- [x] 限制合并正文长度。
- [x] 返回读取成功/跳过数量，供 assistant 文案使用。

## Task 3: Guide Branch 接入

**Files:**
- Modify: `services/proxy-api/src/index.ts`

- [x] guide_ingest / guide_url 合并附件正文。
- [x] `ingest_user_guide_source` content 包含附件正文。
- [x] assistant 文案体现附件读取结果。

## Task 4: 验证

- [x] `pnpm test:api`
- [x] `pnpm typecheck`
- [x] `pnpm test`
