# Phase 1.7 Travel Mode 多场景重排增强设计

## 背景

Phase 1.5 和 1.6 已经让 Byway 有了稳定的聊天工作区、Trip Workspace、工具进度和刷新恢复。现在 Travel Mode 仍然只有“老人累了 + 晚 40 分钟”的通用重排 mock，无法覆盖 PRD 中 E2-E5 的现场救场场景。

Phase 1.7 的目标是把 Travel Mode 从单一 demo 扩展为多场景可预测 mock：

- 现在大家饿了。
- 孩子不想坐车了。
- 下雨了，下午怎么安排？
- 不，我还是想去小鱼山，取消别的。

## 产品原则

> 旅行中重排不是重新规划整趟旅行，而是保护当前家庭状态下最重要的锚点。

Byway 在 Travel Mode 的优先级应是：

1. 先保护人：老人疲劳、孩子疲劳、饥饿、天气风险。
2. 再保护锚点：用餐、酒店、返程、已明确表达的强偏好。
3. 最后才保护景点数量。

所有高影响调整仍必须经过 PendingConfirmation。用户确认前，不调用 `apply_replan`。

## 范围

### 本阶段实现

1. Proxy API 增强现场反馈识别：
   - `hungry`：饿了、吃饭、午饭、晚饭。
   - `bad_weather`：下雨、天气不好、暴雨。
   - `transport_problem`：不想坐车、坐车太久、少坐车。
   - `child_tired`：孩子累、孩子困。
   - `late` / `elder_tired` 保持现有兼容。
   - `custom`：拒绝推荐方案、明确保留某景点。

2. MCP mock `replan_today` 根据 TravelEvent 生成不同方案：
   - 饥饿：优先吃饭，取消/顺延低优先级活动。
   - 下雨：改室内或低风险活动，取消室外低优先级景点。
   - 孩子不想坐车：减少跨区移动，转酒店附近轻活动。
   - 拒绝推荐方案：保留用户点名景点，取消其他备选。

3. 仍输出三类方案：
   - `cancel`
   - `postpone`
   - `replan`

4. 前端沿用现有 ReplanOptionsArtifact 渲染，不新增复杂 UI。

### 本阶段不实现

- 不接真实天气 API。
- 不接真实高德实时路况。
- 不接真实 LLM 做自由文本规划。
- 不实现多日跨天联动。
- 不把拒绝方案设计成完整交互式约束编辑器。

## 数据与行为设计

### Proxy 事件识别

新增 `inferTravelEvent(text)`：

```ts
type InferredTravelEvent = {
  eventType: TravelEventType;
  context: {
    delayMinutes?: number;
    mealStatus?: string;
    affectedTravelers?: string[];
    weather?: string | null;
  };
};
```

示例：

| 输入 | eventType | context |
| --- | --- | --- |
| 现在大家都饿了。 | `hungry` | `{ mealStatus: "hungry" }` |
| 下雨了，下午怎么安排？ | `bad_weather` | `{ weather: "rain" }` |
| 孩子不想坐车了。 | `transport_problem` | `{ affectedTravelers: ["child"] }` |
| 不，我还是想去小鱼山，取消别的。 | `custom` | `{ affectedTravelers: [] }` |

### MCP 重排策略

`replan_today` 通过 `eventId` 读取已保存的 `TravelEvent`，根据 `eventType` 和 `userText` 选择 mock strategy。

#### hungry

推荐方案：`opt_meal_first`

```text
先去吃饭，取消下一个低优先级景点
```

理由：大家饿了时优先保障 meal anchor，不建议继续赶景点。

#### bad_weather

推荐方案：`opt_indoor`

```text
改为室内/低风险活动，取消室外低优先级景点
```

理由：下雨时优先降低户外风险。

#### transport_problem / child_tired

推荐方案：`opt_nearby`

```text
减少跨区移动，改酒店附近轻活动
```

理由：孩子抗拒坐车时减少远距离移动。

#### custom with 小鱼山

推荐方案：`opt_keep_xiaoyushan`

```text
保留小鱼山，取消五四广场附近备选活动
```

理由：尊重用户明确偏好，重新选择取消对象。

## 验收标准

1. 输入“现在大家都饿了。”后，Trip Workspace 显示：
   - “先去吃饭”
   - “优先保障 meal anchor”
   - 仍出现 pending confirmation。

2. 输入“下雨了，下午怎么安排？”后，Trip Workspace 显示：
   - “室内/低风险活动”
   - “取消室外低优先级景点”

3. 输入“孩子不想坐车了。”后，Trip Workspace 显示：
   - “减少跨区移动”
   - “酒店附近轻活动”

4. 先触发老人累了重排，再输入“不，我还是想去小鱼山，取消别的。”后，Trip Workspace 显示：
   - “保留小鱼山”
   - “取消五四广场附近备选活动”
   - 推荐方案不再是原来的 `opt_cancel`。

5. 全量验证通过：
   - `pnpm typecheck`
   - `pnpm test`
   - `pnpm test:e2e`
   - `pnpm build`
