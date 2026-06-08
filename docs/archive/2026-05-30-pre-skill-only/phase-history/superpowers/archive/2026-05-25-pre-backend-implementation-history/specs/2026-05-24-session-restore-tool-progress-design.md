# Phase 1.6 会话恢复与工具进度设计

## 背景

Phase 1.5 已经把 Byway Web 调整为“左侧主聊天 + 右侧常驻 Trip Workspace”。现在最大的体验断点有两个：

1. 页面刷新后，当前 Trip、Plan、Today、PendingConfirmation 和聊天上下文会丢失。
2. Proxy API 已经通过 SSE 发出 `tool_call_started` / `tool_call_completed`，但前端没有显示工具执行过程，用户只看到等待后的结果。

Phase 1.6 的目标是补上 Agent 产品的连续性和可观察性，让 Byway 从一次性 demo 更接近可持续使用的旅行工作台。

## 产品原则

> 刷新不应该让旅行消失；等待不应该像黑箱。

Byway 的右侧 Trip Workspace 是当前旅行的权威承载面。刷新页面后，它应恢复到当前 Trip 的最新 Plan / Today / Pending 状态。聊天流不是权威数据库，但应保留最近对话和 artifact preview 的上下文，让用户不觉得“重新打开了一个新机器人”。

工具进度不应该变成技术日志。前端只展示面向用户的轻量进度，例如：

- 正在创建旅行
- 正在更新旅行摘要
- 正在定位关键地点
- 正在生成计划
- 正在记录现场变化
- 正在生成重排方案
- 已完成地点定位

## 范围

### 本阶段实现

1. 前端保存本地会话快照：
   - `tripId`
   - 最近聊天消息
   - artifact preview 绑定关系
   - 当前 workspace mode
   - 当前 active artifact

2. 页面加载时恢复：
   - 从 `localStorage` 找到最近 `tripId`。
   - 调用 `GET /api/trips/:tripId` 和 `GET /api/trips/:tripId/artifacts`。
   - 用后端 artifacts 重建 `artifactsById`、`currentPlanArtifactId`、`todayArtifactId`、`activeArtifactId`。
   - 用 pending confirmations 恢复右侧确认区。
   - 如果后端 Trip 不存在或请求失败，保留欢迎态并清理失效 `tripId`。

3. 工具进度显示：
   - 收到 `tool_call_started` 时，在当前 assistant 消息下追加进度 item。
   - 收到 `tool_call_completed` 时，把对应 item 标记为完成。
   - 进度 item 显示用户友好的工具名，不暴露原始 args。
   - 工具进度不生成 artifact，也不进入右侧详情。

4. 刷新后的行为：
   - 如果已有 PlanArtifact，右侧默认回到“全程计划”。
   - 如果有 TodayArtifact 但没有 Plan，右侧默认回到“今天”。
   - 如果有 pending confirmation，右侧显示确认区，并允许继续确认。

### 本阶段不实现

- 不做服务端 AgentMessage 持久化。
- 不做多 Trip 管理界面。
- 不做真实用户登录和云端同步。
- 不把工具调用参数完整暴露给用户。
- 不解决 dev server 重启后的数据库持久化问题；当前 mock SQLite 生命周期仍随 MCP server 生命周期走。

## 状态设计

### 本地快照

新增 `byway.session.v1`：

```ts
type StoredSessionSnapshot = {
  version: 1;
  tripId?: string;
  messages: ChatItem[];
  activeArtifactId?: string;
  workspaceMode: WorkspaceMode;
  currentPlanArtifactId?: string;
  todayArtifactId?: string;
  updatedAt: string;
};
```

只保存前端展示上下文，不保存权威 artifact content。artifact content 由后端 `/api/trips/:tripId/artifacts` 恢复。

### 工具进度

扩展聊天消息：

```ts
type ToolProgressItem = {
  callId: string;
  toolName: string;
  label: string;
  status: "running" | "completed" | "failed";
};

type ChatItem = {
  id: string;
  role: "assistant" | "user";
  text: string;
  artifactIds: string[];
  toolProgress?: ToolProgressItem[];
};
```

`callId` 同时兼容 Proxy 当前发出的 `callId` 和文档里的 `toolCallId`。

## 数据流

### 发送消息时

```text
用户输入
-> append user message
-> append assistant message
-> SSE assistant_message_delta 更新文本
-> SSE tool_call_started 追加工具进度
-> SSE tool_call_completed 标记完成
-> SSE artifact_updated 保存 artifact 并挂到当前 assistant message
-> SSE confirmation_required 设置 activeConfirmation
-> SSE assistant_message_completed 保存 tripId
-> 写入 localStorage session snapshot
```

### 页面刷新时

```text
读取 localStorage snapshot
-> 先恢复聊天骨架
-> 如果有 tripId：
   -> GET /api/trips/:tripId
   -> GET /api/trips/:tripId/artifacts
   -> 重建 artifactsById
   -> 重建 currentPlan / today / activeArtifact
   -> 恢复 pending confirmation
-> 成功后写回 snapshot
```

## UI 设计

工具进度显示在 assistant bubble 下方，位于 artifact preview 之前：

```text
我先把这次旅行的关键上下文收好。

✓ 创建旅行
✓ 更新旅行摘要
✓ 生成目的地候选

[目的地候选 preview]
```

进行中的工具使用 spinner 图标；完成后使用 check 图标。工具进度 item 高度稳定，避免消息跳动。

刷新恢复时不弹 toast，不打断用户。Trip Header 的 trip id 和 artifact type 会自然恢复。

## 错误处理

- `localStorage` 解析失败：清理快照，进入欢迎态。
- `GET /api/trips/:tripId` 404 或失败：清理快照，进入欢迎态，并在聊天里显示一条轻量 assistant 提示。
- artifact content 缺失：保留消息文本，但不渲染对应 preview。
- pending confirmation 无法恢复为可提交动作时：仍展示确认摘要，但提交失败后显示错误消息。

## 验收标准

1. Dream Mode 中发送首条消息时，聊天流显示工具进度。
2. Plan Mode 中至少显示定位地点和生成计划相关工具进度。
3. 生成青岛计划后刷新页面，仍能看到：
   - `#<tripId>`。
   - Plan preview。
   - 右侧“全程计划”。
   - 青岛 4 天家庭游。
4. Travel Mode 生成 pending confirmation 后刷新页面，仍能看到：
   - 今日重排方案 artifact。
   - “是否应用推荐调整方案？”。
   - 当前选中方案。
5. 全量 `pnpm typecheck`、`pnpm test`、`pnpm test:e2e`、`pnpm build` 通过。
