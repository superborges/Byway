# Phase 3.13 Accepted Family Memory 注入设计

## 目标

F3 已经能生成 Memory candidate，并通过 PendingConfirmation 接受或拒绝。但 accepted memory 还没有稳定参与后续 Dream / Plan 编排，导致“这家人的旅行偏好会越来越懂”只停留在存储层。

Phase 3.13 让 accepted Family Memory 成为后续推荐与计划的真实输入：

- Dream Mode 生成/重排目的地前读取 accepted memory。
- Plan Mode 生成计划前读取 accepted memory。
- MCP 工具接收到 `familyMemoryIds` 后，对候选和计划做可见、保守的调整。
- 只使用 `status=accepted` 的 memory，candidate / rejected 不参与。

## 交互与编排

Proxy 在 Dream、Dream refine、Destination select 生成计划、直接 Plan 四条路径中调用：

```text
get_accepted_family_memory(userId=local_family, categories=["stamina"])
```

然后把返回的 memory ids 传给：

- `generate_destination_candidates`
- `score_destination_candidates`
- `generate_plan`

assistant 文案只在确实有 accepted memory 时提示“已参考家庭记忆”，避免空洞提示。

## MCP 行为

当前最小支持 `stamina` 类记忆：

- 目的地候选：对强度更高的候选增加风险说明，必要时下调分数。
- 计划生成：在 assumptions / warnings 中标注家庭记忆影响，并保持下午休息或低步行安排。

## 边界

- 不使用未确认 candidate memory。
- 不把 memory 当作事实源；它是家庭偏好/约束。
- 不因为 memory 阻断计划，只把它作为优先级和风险调整依据。

## 验收

- 接受 memory 后，下一次 Dream 会调用 `get_accepted_family_memory`。
- `generate_destination_candidates` 和 `score_destination_candidates` 的输入包含 accepted memory id。
- 下一次 Plan 的 `generate_plan` 输入包含 accepted memory id。
- PlanArtifact 中可以看到家庭记忆相关 assumption 或 warning。
