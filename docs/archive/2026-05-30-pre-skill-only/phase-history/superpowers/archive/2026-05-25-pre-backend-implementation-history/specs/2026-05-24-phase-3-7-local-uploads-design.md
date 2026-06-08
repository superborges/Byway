# Phase 3.7 本地上传存储设计

## 目标

把 `/api/uploads` 从 mock fileId 改成本地可读取的文件存储，满足自用阶段保存攻略文本、Markdown、URL 摘要或后续截图附件的需求。

## 输入

第一阶段支持 JSON 上传：

```json
{
  "tripId": "trip_123",
  "filename": "guide.md",
  "mimeType": "text/markdown",
  "textContent": "# 青岛攻略..."
}
```

也支持：

- `contentBase64`：用于二进制或图片内容。
- `url`：只保存 URL metadata，不抓取正文。
- multipart 请求暂时仍返回本地 metadata shell，后续 UI 接入文件选择器时再引入 multipart parser。

## 输出

```json
{
  "ok": true,
  "fileId": "file_...",
  "storage": "local",
  "url": "/api/files/file_..."
}
```

`GET /api/files/:fileId` 返回原始内容，并使用保存的 `mimeType`。

## 存储

- 默认目录：`.byway/uploads`。
- 每个文件写入 `<fileId>.bin`。
- metadata 写入 `<fileId>.json`。
- 目录被 `.gitignore` 忽略。

## 边界

- 不抓取小红书评论区或不可访问内容。
- 不做云同步。
- 不引入复杂媒体处理。
