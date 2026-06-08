# Phase 3.7 本地上传存储 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `/api/uploads` 保存本地文件和 metadata，`/api/files/:fileId` 可读取内容。

**Architecture:** Proxy 内部增加轻量 `LocalUploadStore`，只依赖 Node fs/path/crypto。JSON 上传写入 `.byway/uploads`；multipart 暂时保存 metadata shell 以保持现有验收不破。

**Tech Stack:** TypeScript、Express、Node fs/path/crypto、Vitest/Supertest。

---

## Task 1: 红灯测试

**Files:**
- Modify: `services/proxy-api/src/proxyApi.test.ts`

- [ ] 用临时 upload dir 创建 app。
- [ ] POST JSON 上传 Markdown 文本。
- [ ] GET `/api/files/:fileId` 返回同一内容和 content-type。

## Task 2: LocalUploadStore

**Files:**
- Modify: `services/proxy-api/src/index.ts`
- Modify: `services/proxy-api/src/server.ts`

- [ ] 增加 upload options。
- [ ] 实现 `saveJsonUpload`、`readFile`。
- [ ] `POST /api/uploads` 根据 JSON body 写入本地文件。
- [ ] `GET /api/files/:fileId` 返回文件；不存在时 404。
- [ ] `server.ts` 从 `BYWAY_UPLOADS_DIR` 读取目录，默认 `.byway/uploads`。

## Task 3: 配置与文档

**Files:**
- Modify: `.env.example`
- Modify: `README.md`

- [ ] 增加 `BYWAY_UPLOADS_DIR=.byway/uploads`。
- [ ] README 说明 JSON 上传和本地文件边界。

## 验收

- `pnpm test:api`
- `pnpm typecheck`
- `pnpm test`
- `pnpm build`
