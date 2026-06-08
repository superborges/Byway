# Phase 2.0 Plan Artifact 质量与地理核验设计

## 背景

Byway 当前已经具备 Dream、Plan、PlaceFact、Travel、Review、Guide 的 mock 闭环。下一步需要把 PlanArtifact 从“能生成”推进到“更像可执行家庭计划”：完整游玩日不能缺午晚餐，抵达/离开日必须轻量，明显跨区往返要给出 geo warning。

本阶段覆盖验收场景 D1-D4。

## 产品原则

家庭旅行计划先保护体力和确定性，再追求景点数量。用餐、酒店、返程是 anchor；activity 可以取消或降级，不能挤掉 anchor。

## 范围

### 本阶段实现

1. `generate_plan` 的 mock planner 增强：
   - 每个 `full_day` 至少有 lunch 和 dinner 两个 meal anchor。
   - Day 1 根据 arrivalInfo / assumptions 保持抵达、入住、放行李、酒店附近晚餐。
   - Day 4 根据 departureInfo / assumptions 保留返程缓冲和靠近酒店/交通方向的餐食。
   - 如果完整游玩日缺 meal anchor，生成 PlanWarning。

2. `verify_plan_geo` 增强：
   - 检测同一天把“崂山”与“栈桥 / 八大关 / 五四广场 / 小麦岛”等市区点混排的明显跨区往返。
   - 返回 `geoWarnings`、`overallGeoFit`、`suggestedFixes`。
   - 明确不承诺导航级最优路线。

3. Proxy Plan Mode：
   - 已经调用 `verify_plan_geo`，本阶段把 geo warning 写回 assistant message，必要时同步到 PlanArtifact warnings。

4. Web 展示：
   - PlanArtifact detail 中展示 plan-level 和 day-level warnings。

### 本阶段不实现

- 不接真实高德路径规划。
- 不做自动地图排序优化。
- 不做复杂时间表排程。
- 不自动重写用户明确锁定的景点。

## 数据流

```text
generate_plan
-> makePlan 根据 TripBrief/assumptions 生成 anchor-first days
-> verifyPlanMealAnchors 补 PlanWarning
-> verify_plan_geo(planId)
-> 返回 geoWarnings / suggestedFixes
-> Proxy 追加简短说明
```

## 验收标准

1. Day 2 / Day 3 作为完整游玩日，都包含 lunch 和 dinner meal anchor。
2. Day 1 包含抵达、入住/放行李、酒店附近晚餐，不包含崂山等远距离重景点。
3. Day 4 包含返程缓冲和靠近酒店/交通方向的餐食，不包含崂山等远距离重景点。
4. 同一天混排崂山与市区核心点时，`verify_plan_geo` 返回跨区往返 warning 和调整建议。
5. 前端 Plan 详情能看到 plan/day warnings。
