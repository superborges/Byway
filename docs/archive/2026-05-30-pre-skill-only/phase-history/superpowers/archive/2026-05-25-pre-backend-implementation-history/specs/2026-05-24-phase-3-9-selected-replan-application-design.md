# Phase 3.9 选中重排方案真实应用设计

## 目标

Travel Mode 已能生成取消、顺延、重排等多个方案，前端也能选择不同 option。但当前后端 `apply_replan` 没有真正按 `replanOptionId` 应用差异，导致交互不可信。

Phase 3.9 要让用户选中的方案真实影响 PlanArtifact：

- `cancelledNodeIds` → 对应节点状态改为 `skipped`。
- `postponedNodeIds` → 对应节点状态改为 `postponed`。
- `preservedNodeIds` → 保持原状态，不被其他默认逻辑覆盖。
- 返回的 `changeSummary` 使用所选 option 的 summary/reason。

## 数据来源

`apply_replan` 根据 `tripId`、`dayIndex`、`replanOptionId` 在已保存的 `ReplanOptionsArtifact` 中查找 option。找不到则返回 recoverable error，不猜测。

## 边界

- 本阶段不做精确时间重排；`updatedTimes` 存在时先保留在 option 中。
- 不自动应用未确认方案。
- 不改动非目标 day 的节点状态。
