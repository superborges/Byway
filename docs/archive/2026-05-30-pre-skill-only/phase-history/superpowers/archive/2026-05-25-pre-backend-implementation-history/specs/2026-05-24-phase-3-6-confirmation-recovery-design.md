# Phase 3.6 Pending Confirmation 恢复设计

## 目标

Phase 3.4 已经让 PendingConfirmation 持久化，但 Proxy 仍用内存 `confirmations` map 保存“确认按钮该调用哪个工具”的索引。服务重启后前端能从 Trip state 恢复 pending confirmation，却无法提交确认。

Phase 3.6 的目标是：即使 Proxy 重启，`POST /api/confirmations/:confirmationId/respond` 也能从 MCP 持久化状态重建确认上下文。

## 方案

- 保留现有内存 map，用于当前会话快速响应。
- 当 map miss 时，Proxy 调用 MCP `list_trips` 和 `get_trip_state(includeArtifacts=true)` 搜索该 confirmation。
- 根据 `PendingConfirmation.type` 和 payload 重建内部 `ConfirmationIndexEntry`：
  - `apply_replan`：从 payload 读取 `replanOptionId`、`artifactId`，从对应 `ReplanOptionsArtifact` snapshot 读取 `dayIndex`。
  - `accept_memory`：从 payload 读取 `candidateIds`。
- `replan_today` 新写入的 confirmation payload 增加 `dayIndex`，减少未来恢复时对 artifact snapshot 的依赖。

## 边界

- 不恢复已 accepted/rejected 的确认；这些应返回 404 或不可操作。
- 不允许仅凭前端传入的 tripId/optionId 执行高影响动作，仍以 MCP pending confirmation 为准。
