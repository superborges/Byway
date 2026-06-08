# Byway / 另辟蹊径 — 前后端 API 与流式事件文档

---

## 1. 文档目标

定义移动 Web 前端如何与 Byway Proxy API 通讯。

前端不直接调用 Hermes、不直接调用高德、不直接调用 LLM Provider。前端只访问 Byway Proxy API。

---

## 2. API 总览

```text
POST   /api/chat
GET    /api/trips
POST   /api/trips
GET    /api/trips/:tripId
GET    /api/trips/:tripId/artifacts
POST   /api/confirmations/:confirmationId/respond
POST   /api/uploads
GET    /api/files/:fileId
GET    /api/health
```

---

## 3. 鉴权

自用阶段可以使用简单家庭 PIN 或 session token。

请求头：

```http
Authorization: Bearer <byway-session-token>
```

前端不得持有：

- Hermes API Key。
- 高德 API Key。
- LLM Provider Key。

---

## 4. `POST /api/chat`

### 目的

发送用户消息给 Agent，并接收流式事件。

### 请求

```json
{
  "tripId": "trip_123",
  "message": {
    "text": "老人累了，我们晚了 40 分钟",
    "attachments": []
  },
  "clientContext": {
    "timezone": "Asia/Shanghai",
    "locale": "zh-CN",
    "currentPage": "travel",
    "currentArtifactId": "today_123"
  }
}
```

### 响应

使用 SSE 或 fetch streaming。

```http
Content-Type: text/event-stream
```

事件类型见第 10 节。

---

## 5. `GET /api/trips`

### 目的

获取旅行列表。

### 响应

```json
{
  "trips": [
    {
      "id": "trip_123",
      "title": "青岛 4 天家庭游",
      "phase": "traveling",
      "destination": "青岛",
      "updatedAt": "2026-05-23T10:00:00Z"
    }
  ]
}
```

---

## 6. `POST /api/trips`

### 目的

创建空旅行会话。通常也可以由 `/api/chat` 自动触发。

### 请求

```json
{
  "initialText": "我想国庆带父母和孩子从北京出发玩 4 天，不知道去哪"
}
```

### 响应

```json
{
  "tripId": "trip_123",
  "phase": "intake"
}
```

---

## 7. `GET /api/trips/:tripId`

### 目的

获取当前旅行状态。

### 响应

```json
{
  "tripId": "trip_123",
  "phase": "traveling",
  "tripBrief": {},
  "currentPlanId": "plan_123",
  "currentDayIndex": 2,
  "pendingConfirmations": []
}
```

---

## 8. `GET /api/trips/:tripId/artifacts`

### 目的

获取当前旅行可展示 Artifact。

### 查询参数

```text
type=plan|today|destination_shortlist|replan|day_log|trip_log|memory
```

### 响应

```json
{
  "artifacts": [
    {
      "artifactId": "plan_123",
      "artifactType": "PlanArtifact",
      "version": 3,
      "content": {}
    }
  ]
}
```

---

## 9. `POST /api/confirmations/:confirmationId/respond`

### 目的

用户对 pending confirmation 做确认或拒绝。

### 请求

```json
{
  "decision": "accept",
  "selectedOptionId": "opt_cancel",
  "userNote": "就按推荐方案执行"
}
```

### 响应

```json
{
  "ok": true,
  "result": {
    "updatedArtifacts": ["today_124"],
    "message": "已应用推荐方案"
  }
}
```

---

## 10. `POST /api/uploads`

### 目的

上传截图、markdown 文件、图片等。

### 请求

`multipart/form-data`

字段：

```text
file
tripId
userNote
```

### 响应

```json
{
  "fileId": "file_123",
  "type": "image",
  "url": "/api/files/file_123",
  "textPreview": null
}
```

---

## 11. 流式事件协议

`/api/chat` 返回的事件类型。

---

### 11.1 `assistant_message_delta`

Agent 文本增量。

```json
{
  "type": "assistant_message_delta",
  "messageId": "msg_123",
  "delta": "我建议先取消低优先级景点，"
}
```

---

### 11.2 `assistant_message_completed`

Agent 回复完成。

```json
{
  "type": "assistant_message_completed",
  "messageId": "msg_123"
}
```

---

### 11.3 `tool_call_started`

工具调用开始。

```json
{
  "type": "tool_call_started",
  "toolCallId": "tc_123",
  "toolName": "resolve_places_batch",
  "displayText": "正在定位关键地点..."
}
```

---

### 11.4 `tool_call_completed`

工具调用完成。

```json
{
  "type": "tool_call_completed",
  "toolCallId": "tc_123",
  "toolName": "resolve_places_batch",
  "summary": "已定位 12 个地点，其中 2 个需要确认"
}
```

---

### 11.5 `artifact_updated`

Artifact 更新。

```json
{
  "type": "artifact_updated",
  "artifactType": "PlanArtifact",
  "artifactId": "plan_123",
  "version": 2,
  "content": {}
}
```

前端收到后更新对应卡片。

---

### 11.6 `confirmation_required`

需要用户确认。

```json
{
  "type": "confirmation_required",
  "confirmationId": "confirm_123",
  "confirmationType": "apply_replan",
  "summary": "是否应用推荐调整方案？",
  "options": [
    { "id": "accept", "label": "应用推荐方案" },
    { "id": "reject", "label": "不应用" }
  ]
}
```

---

### 11.7 `error`

错误事件。

```json
{
  "type": "error",
  "code": "PLACE_FACT_FAILED",
  "message": "部分地点定位失败，可以继续使用草案，但需要确认核心地点",
  "recoverable": true
}
```

---

## 12. Artifact 渲染规则

前端必须识别：

```text
DestinationShortlistArtifact
GuideSummaryArtifact
PlaceFactReviewArtifact
PlanArtifact
TodayArtifact
ReplanOptionsArtifact
DayLogArtifact
TripLogArtifact
MemoryCandidateArtifact
```

未知 artifact 类型：

- 显示通用 JSON/文本摘要。
- 不阻断聊天。

---

## 13. 前端状态恢复

页面刷新时：

```text
GET /api/trips/:tripId
GET /api/trips/:tripId/artifacts
```

恢复：

- 当前 phase。
- 当前计划。
- 今日状态。
- pending confirmations。
- 最近消息。

---

## 14. 错误处理

### 可恢复错误

例如：

- 部分地点未定位。
- LLM 研究失败。
- 高德 API 超时。

前端应展示 Agent 的解释和下一步建议。

### 不可恢复错误

例如：

- 鉴权失败。
- Trip 不存在。
- 数据库错误。

前端应展示明确错误，并允许重试。

---

## 15. 安全要求

- 前端不存外部 API Key。
- 上传文件需要大小限制。
- URL 抽取需要防 SSRF。
- Proxy 需要限流。
- 用户数据默认私有。
- Hermes 只通过受控 Proxy 调用。

---

## 16. API 验收标准

- 聊天流式输出可用。
- 工具调用进度可见。
- Artifact 能实时更新。
- 用户确认能触发后端状态变更。
- 页面刷新能恢复当前状态。
- 高德/LLM 失败时有可恢复提示。
