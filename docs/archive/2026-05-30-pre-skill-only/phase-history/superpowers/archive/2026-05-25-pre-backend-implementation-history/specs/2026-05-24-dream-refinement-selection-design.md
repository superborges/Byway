# Phase 2.1 Dream 偏好重排与文字选择设计

## 背景

Dream Mode 已经能在用户“不知道去哪”时生成候选目的地，也能通过卡片点击选择目的地。但验收场景 A2/A3 还需要两种自然对话能力：

- 用户补充偏好后，不进入 Plan Mode，而是重排候选。
- 用户用文字说“那就青岛吧”时，也能完成目的地选择并进入规划。

## 范围

### 本阶段实现

1. Proxy API 增加 `dream_refine` intent：
   - 识别“更想去海边 / 吃饭方便 / 最好...”等偏好补充。
   - 更新 TripBrief themes。
   - 重新调用 destination candidate / score 工具。
   - 返回更新后的 DestinationShortlistArtifact 和变化原因。

2. Proxy API 增加 `destination_select` intent：
   - 识别“那就青岛吧 / 选青岛 / 决定去青岛”等文字选择。
   - 调用 `select_destination`。
   - 进入 Plan Mode，并用现有 mock plan pipeline 生成可调整计划。
   - 明确抵离时间仍是 assumptions，不要求长表单。

3. Web 无新增 UI，只复用聊天输入、artifact preview、workspace。

### 本阶段不实现

- 不做复杂自然语言排序模型。
- 不做多轮 candidate memory。
- 不接真实目的地研究服务。

## 验收标准

1. Dream 后输入“更想去海边，最好吃饭方便。”：
   - 触发 `update_trip_brief`。
   - 触发 `generate_destination_candidates`。
   - 触发 `score_destination_candidates`。
   - 更新 DestinationShortlistArtifact。
   - assistant message 说明提高了海边和吃饭方便权重。

2. Dream 后输入“那就青岛吧。”：
   - 触发 `select_destination`。
   - TripBrief.destination 设为青岛。
   - 生成 PlanArtifact。
   - assistant message 说明先按抵离时间假设生成计划。
