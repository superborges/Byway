# Phase 3.14 上传攻略附件吸收设计

## 目标

Phase 3.7 已经让 `/api/uploads` 支持本地文本/base64/URL metadata 落盘，但 `/api/chat` 仍没有读取 message attachments 中的 `fileId`。这意味着用户上传 Markdown/文本攻略后，Guide ingestion 仍只看聊天文字，无法真正吸收附件内容。

Phase 3.14 补齐这条链路：

- 用户先调用 `POST /api/uploads` 保存攻略文本或 Markdown。
- 聊天消息带上 `attachments: [{ fileId }]` 并触发 guide ingestion。
- Proxy 从本地 upload store 读取 text-like 文件内容。
- 文件正文与用户聊天文本一起交给 `ingest_user_guide_source`。
- assistant 明确说明已读取上传附件；如果附件不可读，则降级说明。

## 支持范围

首版只处理可安全转文本的本地上传：

- `text/*`
- `text/markdown`
- `application/json`
- `text/uri-list`

读取总长度限制在 12,000 字符以内，避免把大文件直接塞进上下文。

## 边界

- 不做 OCR，不解析图片/PDF。
- 不读取未通过 `/api/uploads` 保存的任意本地路径。
- 不因为附件读取失败阻断聊天；失败时继续使用用户文本。

## 验收

- 上传包含“五四广场住宿、八大关、崂山偏累”的 Markdown 后，聊天附件触发 GuideSummaryArtifact。
- assistant 文案包含“已读取上传附件”。
- GuideSummaryArtifact 的 changeSummary 能体现附件中的住宿/地点信息。
