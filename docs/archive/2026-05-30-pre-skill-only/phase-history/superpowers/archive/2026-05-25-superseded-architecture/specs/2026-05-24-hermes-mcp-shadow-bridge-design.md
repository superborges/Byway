# Phase 3.3 Hermes MCP Shadow Bridge 设计

## 背景

Byway Proxy 已能通过 stdio 连接 `hermes mcp serve`，并确认当前 Hermes MCP 工具面是 messaging bridge：

- `conversations_list`
- `conversation_get`
- `messages_read`
- `attachments_fetch`
- `events_poll`
- `events_wait`
- `messages_send`
- `channels_list`
- `permissions_list_open`
- `permissions_respond`

这个工具面不是 Byway 专用旅行 Agent API，也不是可直接替换 `/api/chat` 的 planner runtime。它适合先做“桥接和旁路观察”：Byway 可以知道 Hermes 连接了哪些消息频道、读取会话、显式发送消息；等消息会话映射稳定后，再让 Hermes 接管部分编排。

## 目标

1. Proxy 提供 Hermes MCP 工具桥 API，方便本地调试和后续 UI/Agent 映射。
2. `/api/chat` 支持可关闭的 Shadow Mode，把用户消息镜像到一个显式配置的 Hermes 目标频道。
3. 默认保持 `off`，不自动发送到微信/其他平台，不替换当前 Byway mock orchestrator。

## API 设计

新增 Proxy API：

- `GET /api/hermes/mcp/channels`
  - 调用 `channels_list`
  - 返回可用 target，例如 `weixin:<chat_id>`
- `GET /api/hermes/mcp/conversations`
  - 调用 `conversations_list`
- `GET /api/hermes/mcp/conversations/:sessionKey/messages`
  - 调用 `messages_read`
- `POST /api/hermes/mcp/messages`
  - body: `{ "target": "...", "message": "..." }`
  - 显式调用 `messages_send`
  - 缺 target/message 时返回 400

保留：

- `GET /api/hermes/mcp/probe`

## Shadow Mode

新增环境变量：

```dotenv
HERMES_MCP_SHADOW_MODE=off
HERMES_MCP_TARGET=
```

取值：

- `off`：默认，不镜像任何聊天消息。
- `mirror_user_messages`：在 `/api/chat` 收到用户消息后，调用 `messages_send` 镜像到 `HERMES_MCP_TARGET`。

安全规则：

1. 没有 `HERMES_MCP_TARGET` 时，即使 mode 打开也不发送。
2. Shadow 失败不影响 Byway 主流程。
3. Shadow 发送通过 SSE tool event 可见，工具名为 `hermes_mcp.messages_send`。
4. 仍由 Byway mock orchestrator 生成当前产品 Artifact；Hermes 只作为旁路消息目标。

## 非目标

- 不自动选择第一个 Hermes channel。
- 不把 Byway `/api/chat` 直接切给 Hermes。
- 不让 Hermes MCP messaging bridge 直接调用 Byway MCP Tools。
- 不在前端暴露“发送到微信”的 UI，避免误触。

## 下一阶段

下一阶段可以在此基础上做：

1. 选择一个专用 Hermes 会话作为 Byway Agent 会话。
2. 将 Hermes 回复事件通过 `events_wait` 映射成 Byway SSE。
3. 将 Hermes tool call 显式限制到 Byway MCP Tools。
4. 最后再把 `/api/chat` 从 mock orchestrator 切到 Hermes runtime。
