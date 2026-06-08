# Phase 2.3 Family Memory 确认交互设计

## 背景

Review/Memory 后端已经能生成 MemoryCandidateArtifact，并通过 confirmation 接受家庭记忆。但前端当前复用 Travel Replan 的确认文案：点击后会说“已应用推荐调整”，这不符合 F3“确认后 accepted”的用户语义。

## 范围

### 本阶段实现

1. ConfirmationBlock 根据 confirmation type 显示按钮：
   - `apply_replan`：应用
   - `accept_memory`：接受

2. 点击 `accept_memory` 成功后，聊天中追加：
   - `已接受家庭记忆，下次推荐和计划会参考这条偏好。`

3. 保持现有 API，不新增后端接口。

### 本阶段不实现

- 不做 memory 管理列表。
- 不做 rejected memory UI。
- 不做跨 Trip 的真实用户账号体系。

## 验收标准

1. Review Mode 生成 MemoryCandidateArtifact 后，confirmation 按钮显示“接受”。
2. 点击后页面出现“已接受家庭记忆”。
3. 原 Travel Replan confirmation 仍显示“应用”。
