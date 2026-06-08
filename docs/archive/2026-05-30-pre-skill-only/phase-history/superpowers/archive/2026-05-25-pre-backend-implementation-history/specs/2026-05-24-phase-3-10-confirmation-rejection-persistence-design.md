# Phase 3.10 Confirmation 拒绝状态持久化设计

## 目标

用户拒绝一个 pending confirmation 后，该确认必须在 MCP 状态中变为 `rejected`，不能刷新后再次出现。

## 行为

- `POST /api/confirmations/:id/respond` 收到 `decision !== "accept"`：
  - `apply_replan`：只把对应 PendingConfirmation 标记为 `rejected`，不改计划。
  - `accept_memory`：调用 `accept_family_memory(accepted=false)`，把 memory candidate 和 PendingConfirmation 都标记为 rejected。
- 新增 MCP 内部工具 `respond_pending_confirmation`，用于通用更新 pending confirmation 状态。

## 边界

- 不允许拒绝后执行高影响动作。
- 已 accepted/rejected 的 confirmation 不重复执行。
