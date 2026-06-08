# Byway / 另辟蹊径 — MCP 工具契约文档

---

## 1. 文档目标

Byway 的核心不是“让 LLM 直接生成文本”，而是让 Hermes Agent 通过 Byway MCP 工具读写状态、调用事实源、生成 Artifact。

本文定义 MCP 工具的：

- 目的。
- 输入。
- 输出。
- 副作用。
- 调用时机。
- 禁止事项。
- 错误行为。

所有工具必须返回结构化 JSON，并进行输入输出 schema 校验。

---

## 1.1 契约命名规则

实现时按 `docs/BYWAY_CONTRACT_INDEX.md` 和 `docs/BYWAY_DATA_MODEL.md` 统一命名：

- MCP 工具名使用 `snake_case`。
- 枚举值使用 `snake_case`。
- Artifact 类型名使用 `PascalCase`。
- 计划 Artifact 统一命名为 `PlanArtifact`，不使用 `TravelPlan` 作为实现类型名。
- Memory 接受工具统一命名为 `accept_family_memory`，不使用 `confirm_family_memory`。
- 计划更新工具统一命名为 `update_plan_artifact`，不使用 `revise_plan_with_new_info`。

---

## 2. 通用返回格式

所有工具统一返回：

```json
{
  "ok": true,
  "data": {},
  "warnings": [],
  "errors": [],
  "artifactUpdates": [],
  "nextSuggestedActions": []
}
```

失败时：

```json
{
  "ok": false,
  "data": null,
  "warnings": [],
  "errors": [
    {
      "code": "PLACE_NOT_FOUND",
      "message": "未能定位该地点",
      "recoverable": true,
      "suggestedUserQuestion": "我找到了多个同名地点，你指的是哪一个？"
    }
  ],
  "artifactUpdates": [],
  "nextSuggestedActions": []
}
```

---

## 3. Trip State Tools

## 3.1 `create_trip`

### 目的

创建一趟新的旅行会话。

### 输入

```json
{
  "userId": "user_123",
  "initialMessage": "我想国庆带父母和孩子从北京出发玩 4 天，不知道去哪",
  "source": "chat"
}
```

### 输出

```json
{
  "tripId": "trip_123",
  "phase": "intake",
  "tripBrief": {
    "origin": "北京",
    "destination": null,
    "days": 4,
    "travelers": [],
    "pace": null
  }
}
```

### 副作用

- 创建 Trip。
- 创建初始 AgentMessage。

### 调用时机

- 用户开始新的旅行会话。
- 当前没有可用 tripId。

---

## 3.2 `update_trip_brief`

### 目的

更新旅行摘要。

### 输入

```json
{
  "tripId": "trip_123",
  "patch": {
    "origin": "北京",
    "destination": "青岛",
    "days": 4,
    "dateRange": null,
    "travelers": [
      { "role": "self", "ageGroup": "adult" },
      { "role": "elder", "ageGroup": "elder" },
      { "role": "child", "ageGroup": "child_3" }
    ],
    "pace": "relaxed",
    "arrivalInfo": "第一天下午到",
    "departureInfo": "第四天下午走",
    "hotelInfo": null
  },
  "sourceMessageId": "msg_123"
}
```

### 输出

```json
{
  "tripId": "trip_123",
  "tripBrief": {},
  "changedFields": ["destination", "days", "pace"],
  "missingCriticalFields": ["arrivalInfo", "departureInfo"]
}
```

### 副作用

- 更新 TripBrief。
- 保存变更记录。

### 调用时机

- 用户补充目的地、天数、同行人、交通、酒店等信息。

---

## 3.3 `get_trip_state`

### 目的

返回当前旅行权威状态。

### 输入

```json
{
  "tripId": "trip_123",
  "includeArtifacts": true
}
```

### 输出

```json
{
  "tripId": "trip_123",
  "phase": "planning",
  "tripBrief": {},
  "currentArtifacts": {
    "destinationShortlist": null,
    "plan": {},
    "today": null
  },
  "pendingConfirmations": []
}
```

### 调用时机

- 每次用户消息前后。
- 页面刷新恢复。
- Agent 需要判断当前状态。

---

## 3.4 `set_trip_phase`

### 目的

切换 Trip 阶段。

### 输入

```json
{
  "tripId": "trip_123",
  "phase": "traveling",
  "reason": "用户确认计划并开始旅行"
}
```

### 输出

```json
{
  "tripId": "trip_123",
  "phase": "traveling"
}
```

### 注意

只有在满足状态机转换条件时才能切换。

---

## 4. Dream / Destination Tools

## 4.1 `generate_destination_candidates`

### 目的

根据用户约束生成候选目的地。

### 输入

```json
{
  "tripId": "trip_123",
  "constraints": {
    "origin": "北京",
    "days": 4,
    "monthOrHoliday": "暑假",
    "travelers": ["adult", "elder", "child_12", "child_3"],
    "pace": "relaxed",
    "domesticOrOverseas": "domestic_preferred",
    "budgetLevel": "medium",
    "themes": ["海边", "亲子", "轻松"]
  },
  "familyMemoryIds": ["mem_1", "mem_2"],
  "maxCandidates": 8
}
```

### 输出

```json
{
  "candidates": [
    {
      "destination": "青岛",
      "reason": "北京出发相对方便，海边和城市结合，适合老人孩子轻松节奏",
      "initialFit": "high",
      "risks": ["暑期热门海边区域人多", "酒店价格可能较高"]
    }
  ]
}
```

### 注意

此工具生成候选，不做最终推荐。

---

## 4.2 `research_destination_candidate`

### 目的

对单个候选目的地做轻量研究。

### 输入

```json
{
  "tripId": "trip_123",
  "destination": "青岛",
  "constraints": {},
  "researchDepth": "standard"
}
```

### 输出

```json
{
  "destination": "青岛",
  "recommendedDays": {
    "minimumComfortable": 3,
    "ideal": 4
  },
  "familyFit": {
    "threePerson": "high",
    "fourPerson": "medium_high",
    "sixPerson": "high",
    "reasons": ["城市成熟", "海边和城区结合", "吃饭住宿选择多"]
  },
  "transportFit": {
    "fromOrigin": "北京",
    "railFeasibility": "medium_high",
    "flightFeasibility": "high",
    "doorToHotelComplexity": "medium"
  },
  "lodgingAreasPreview": ["五四广场/奥帆中心", "老城区"],
  "mealAreasPreview": ["五四广场", "台东", "老城区"],
  "risks": ["崂山对老人孩子可能偏累"],
  "planPreview": ["Day 1 抵达 + 酒店附近", "Day 2 老城 + 八大关"],
  "recommendation": "strong"
}
```

### 注意

LLM 可参与研究，但事实结论必须标注不确定性。

---

## 4.3 `score_destination_candidates`

### 目的

对候选目的地做解释型排序。

### 输入

```json
{
  "tripId": "trip_123",
  "researchResults": [],
  "familyMemoryIds": []
}
```

### 输出

```json
{
  "artifactType": "DestinationShortlistArtifact",
  "candidates": [
    {
      "destination": "青岛",
      "score": 9,
      "familyFit": "high",
      "timeFit": "high",
      "transportFit": "medium_high",
      "mainRisks": ["暑期人多"],
      "whyRecommended": "4 天轻松节奏可执行",
      "whyNot": "如果想纯历史文化，青岛不是最强"
    }
  ]
}
```

### 副作用

- 保存 DestinationShortlistArtifact。

---

## 4.4 `select_destination`

### 目的

用户选择目的地后，写入 TripBrief 并进入 Plan Mode。

### 输入

```json
{
  "tripId": "trip_123",
  "destination": "青岛",
  "selectedCandidateId": "candidate_1"
}
```

### 输出

```json
{
  "tripId": "trip_123",
  "destination": "青岛",
  "phase": "planning"
}
```

---

## 5. Guide Tools

## 5.1 `generate_synthetic_guides`

### 目的

由后端 Agent 自动生成多视角攻略草稿。

### 输入

```json
{
  "tripId": "trip_123",
  "destination": "青岛",
  "tripBrief": {},
  "guideRoles": [
    "经典首次游",
    "老人孩子友好",
    "住宿区域",
    "用餐区域",
    "避坑风险",
    "雨天备选"
  ],
  "llmProviders": ["openai", "anthropic", "qwen"]
}
```

### 输出

```json
{
  "sources": [
    {
      "sourceId": "guide_1",
      "sourceType": "model_generated",
      "role": "老人孩子友好",
      "provider": "openai",
      "content": "...",
      "structuredDraft": {}
    }
  ]
}
```

### 注意

- 每份 synthetic guide 必须保留 provider、model、prompt role、生成时间。
- 后续合并时要区分“攻略判断”和“事实字段”。

---

## 5.2 `ingest_user_guide_source`

### 目的

吸收用户补充资料。

### 输入

```json
{
  "tripId": "trip_123",
  "sourceType": "markdown",
  "title": "ChatGPT 青岛 4 天攻略",
  "content": "# 青岛四天三晚...",
  "url": null,
  "attachments": [],
  "userNote": "这个主要参考住宿"
}
```

### 输出

```json
{
  "sourceId": "guide_user_1",
  "extraction": {
    "lodgingAreas": [],
    "mealAreas": [],
    "places": [],
    "routeIdeas": [],
    "warnings": []
  }
}
```

### 注意

- 小红书链接如果不可读，只保存链接和用户备注。
- 截图可通过视觉/OCR工具提取，但不保证完整。

---

## 5.3 `merge_guides`

### 目的

将 synthetic guide 和用户 guide 合并成 Canonical Guide。

### 输入

```json
{
  "tripId": "trip_123",
  "sourceIds": ["guide_1", "guide_2", "guide_user_1"],
  "preserveUserOverrides": true
}
```

### 输出

```json
{
  "canonicalGuideId": "cg_123",
  "artifactType": "GuideSummaryArtifact",
  "lodgingAreas": [],
  "mealAreas": [],
  "places": [],
  "routeIdeas": [],
  "conflicts": [],
  "warnings": [],
  "revisionSummary": "新增 3 个地点，强化 2 个住宿区域，发现 1 个争议点"
}
```

### 规则

- 多源提及提高优先级。
- 冲突要保留，不要强行消除。
- 用户手动覆盖不能被自动覆盖。

---

## 6. Place / Geo Tools

## 6.1 `resolve_place`

### 目的

将一个地点名称解析为标准 PlaceFact。

### 输入

```json
{
  "tripId": "trip_123",
  "query": "八大关",
  "city": "青岛",
  "context": {
    "placeTypeHint": "attraction",
    "nearbyArea": "市南区",
    "sourceText": "上午八大关，下午小麦岛"
  }
}
```

### 输出

```json
{
  "placeFact": {
    "id": "pf_123",
    "inputName": "八大关",
    "standardName": "八大关风景区",
    "provider": "amap",
    "providerPlaceId": "B0xxxx",
    "city": "青岛市",
    "district": "市南区",
    "coordinate": { "lng": 120.354, "lat": 36.058, "system": "GCJ02" },
    "address": "山东省青岛市市南区...",
    "entranceCoordinate": null,
    "navigationPoiId": null,
    "phone": null,
    "openingHoursToday": null,
    "rating": null,
    "confidence": "high",
    "needsReview": false,
    "lastResolvedAt": "2026-05-23T10:00:00Z"
  },
  "candidates": []
}
```

### 低置信度输出

```json
{
  "placeFact": null,
  "candidates": [
    { "name": "某某老街", "district": "市南区", "confidence": "medium" },
    { "name": "某某老街", "district": "市北区", "confidence": "low" }
  ],
  "needsUserSelection": true
}
```

### 规则

- 不允许 LLM 替代此工具生成坐标。
- 同名地点必须返回候选并请求用户确认。

---

## 6.2 `resolve_places_batch`

### 目的

批量解析 Canonical Guide 或 Plan 中的关键地点。

### 输入

```json
{
  "tripId": "trip_123",
  "city": "青岛",
  "places": [
    { "name": "八大关", "typeHint": "attraction", "priority": 5 },
    { "name": "小麦岛", "typeHint": "attraction", "priority": 4 }
  ],
  "resolveLevel": "key_places_first"
}
```

### 输出

```json
{
  "resolved": [],
  "unresolved": [],
  "needsReview": [],
  "summary": "共解析 12 个地点，其中 10 个高置信度，2 个需要确认"
}
```

### 规则

- 4/5 星地点必须优先解析。
- 低优先级地点可以延迟解析。

---

## 6.3 `get_place_detail`

### 目的

获取 POI 详情。

### 输入

```json
{
  "placeFactId": "pf_123",
  "fields": ["phone", "openingHours", "rating", "photos", "businessArea", "entrance"]
}
```

### 输出

```json
{
  "placeFactId": "pf_123",
  "phone": "0532-xxxx",
  "openingHoursToday": "09:00-17:00",
  "openingHoursWeek": "...",
  "rating": 4.6,
  "photos": [],
  "businessArea": "五四广场",
  "fieldConfidence": {
    "openingHoursToday": "medium",
    "rating": "medium"
  }
}
```

### 规则

- 字段缺失时返回 null，不允许补写。
- 营业时间仅作为提示，除非用户确认，不作为强约束。

---

## 6.4 `suggest_taxi_address`

### 目的

为地点生成适合打车/下车的地址建议。

### 输入

```json
{
  "placeFactId": "pf_123",
  "context": {
    "travelers": ["elder", "child_3"],
    "arrivingFrom": "hotel",
    "preferredMode": "taxi"
  }
}
```

### 输出

```json
{
  "taxiAddress": "八大关风景区某入口",
  "entranceCoordinate": { "lng": 120.35, "lat": 36.05, "system": "GCJ02" },
  "navigationPoiId": "...",
  "warning": "该景区较大，入口可能影响步行距离，建议出发前确认具体入口"
}
```

---

## 6.5 `estimate_route`

### 目的

估算两个地点之间的距离和耗时。

### 输入

```json
{
  "tripId": "trip_123",
  "fromPlaceFactId": "pf_hotel",
  "toPlaceFactId": "pf_bada",
  "mode": "driving"
}
```

### 输出

```json
{
  "routeFact": {
    "id": "rf_123",
    "fromPlaceFactId": "pf_hotel",
    "toPlaceFactId": "pf_bada",
    "mode": "driving",
    "distanceMeters": 6200,
    "durationMinutes": 22,
    "provider": "amap",
    "confidence": "high",
    "lastEstimatedAt": "2026-05-23T10:00:00Z"
  }
}
```

### 规则

- Byway 不做导航级实时路况。
- 耗时是估算，不应声称绝对准确。

---

## 6.6 `estimate_route_matrix`

### 目的

批量估算多个地点之间的距离，用于计划排序和地理校验。

### 输入

```json
{
  "tripId": "trip_123",
  "placeFactIds": ["pf_1", "pf_2", "pf_3"],
  "mode": "driving"
}
```

### 输出

```json
{
  "matrix": [
    {
      "from": "pf_1",
      "to": "pf_2",
      "durationMinutes": 18,
      "distanceMeters": 5400,
      "confidence": "high"
    }
  ],
  "missingPairs": []
}
```

---

## 6.7 `verify_plan_geo`

### 目的

校验计划是否存在明显地理问题。

### 输入

```json
{
  "tripId": "trip_123",
  "planId": "plan_123"
}
```

### 输出

```json
{
  "geoWarnings": [
    {
      "severity": "medium",
      "dayIndex": 2,
      "message": "上午和下午地点相距较远，老人孩子同行可能偏累",
      "affectedNodeIds": ["node_1", "node_2"],
      "suggestedFix": "将下午活动换成酒店附近轻活动"
    }
  ],
  "overallGeoFit": "medium_high"
}
```

---

## 6.8 `create_amap_navigation_link`

### 目的

生成可在手机上打开高德 App 的导航链接。

### 输入

```json
{
  "placeFactId": "pf_123",
  "mode": "driving"
}
```

### 输出

```json
{
  "url": "amapuri://route/plan?...",
  "fallbackUrl": "https://uri.amap.com/navigation?..."
}
```

---

## 7. Plan Tools

## 7.1 `generate_plan`

### 目的

生成旅行计划 Artifact。

### 输入

```json
{
  "tripId": "trip_123",
  "canonicalGuideId": "cg_123",
  "tripBrief": {},
  "placeFactIds": [],
  "routeFactIds": [],
  "familyMemoryIds": [],
  "planningAssumptions": ["第一天下午到", "第四天下午走"]
}
```

### 输出

```json
{
  "planId": "plan_123",
  "artifactType": "PlanArtifact",
  "status": "draft",
  "days": [
    {
      "dayIndex": 1,
      "title": "抵达 + 酒店附近轻活动",
      "nodes": []
    }
  ],
  "warnings": [],
  "assumptions": []
}
```

### 规则

- 必须包含住宿区域建议和用餐区域建议。
- 不应把低置信度核心地点当作确定地点。
- 计划是 Artifact，不是单纯自然语言。

---

## 7.2 `get_current_plan`

### 目的

获取当前权威计划。

### 输入

```json
{ "tripId": "trip_123" }
```

### 输出

```json
{
  "planId": "plan_123",
  "status": "confirmed",
  "days": []
}
```

---

## 7.3 `update_plan_artifact`

### 目的

基于用户要求或工具结果更新计划草稿。

### 输入

```json
{
  "tripId": "trip_123",
  "planId": "plan_123",
  "patch": {},
  "reason": "用户补充酒店位置"
}
```

### 输出

```json
{
  "planId": "plan_124",
  "previousPlanId": "plan_123",
  "changeSummary": "根据酒店位置调整 Day 1 晚餐区域"
}
```

### 规则

- 必须生成新版本或可追踪变更。

---

## 7.4 `confirm_plan`

### 目的

用户确认计划，进入 ReadyToTravel。

### 输入

```json
{
  "tripId": "trip_123",
  "planId": "plan_123",
  "userConfirmed": true
}
```

### 输出

```json
{
  "tripId": "trip_123",
  "planId": "plan_123",
  "planStatus": "confirmed",
  "phase": "ready_to_travel"
}
```

---

## 8. Travel / Replan Tools

## 8.1 `get_today_status`

### 目的

获取旅行中今日状态。

### 输入

```json
{
  "tripId": "trip_123",
  "dayIndex": 2
}
```

### 输出

```json
{
  "today": {
    "dayIndex": 2,
    "currentNode": {},
    "nextNode": {},
    "remainingNodes": [],
    "mealAnchors": [],
    "hotelAnchor": {},
    "events": []
  }
}
```

---

## 8.2 `record_travel_event`

### 目的

记录旅行中突发事件。

### 输入

```json
{
  "tripId": "trip_123",
  "dayIndex": 2,
  "eventType": "elder_tired",
  "severity": 4,
  "userText": "老人累了，我们晚了 40 分钟",
  "relatedNodeId": "node_current",
  "context": {
    "delayMinutes": 40,
    "weather": null,
    "mealStatus": "unknown"
  }
}
```

### 输出

```json
{
  "eventId": "evt_123",
  "eventType": "elder_tired",
  "recordedAt": "2026-05-23T10:00:00Z"
}
```

---

## 8.3 `replan_today`

### 目的

基于 TravelEvent 和今日计划生成调整方案。

### 输入

```json
{
  "tripId": "trip_123",
  "dayIndex": 2,
  "eventId": "evt_123"
}
```

### 输出

```json
{
  "artifactType": "ReplanOptionsArtifact",
  "eventId": "evt_123",
  "recommendedOptionId": "opt_cancel",
  "options": [
    {
      "id": "opt_cancel",
      "type": "cancel",
      "summary": "取消下一个低优先级景点，保留晚餐",
      "cancelledNodeIds": ["node_low"],
      "postponedNodeIds": [],
      "updatedTimes": [],
      "preservedNodeIds": ["node_dinner", "node_hotel"],
      "warnings": [],
      "requiresConfirmation": true
    },
    {
      "id": "opt_postpone",
      "type": "postpone",
      "summary": "整体顺延后续计划，但晚餐可能偏晚",
      "cancelledNodeIds": [],
      "postponedNodeIds": ["node_next"],
      "warnings": ["晚餐可能延迟"],
      "requiresConfirmation": true
    },
    {
      "id": "opt_replan",
      "type": "replan",
      "summary": "重排今天剩余计划，优先回酒店休息，再酒店附近晚餐",
      "cancelledNodeIds": ["node_low"],
      "postponedNodeIds": ["node_mid"],
      "warnings": [],
      "requiresConfirmation": true
    }
  ]
}
```

### 硬规则

- 必须输出取消 / 顺延 / 重排三类方案。
- 不自动应用。
- meal / hotel / rest anchor 优先保留。
- 1/2 星活动优先取消。
- 4/5 星取消必须强调需要确认。

---

## 8.4 `apply_replan`

### 目的

用户确认后应用某个重排方案。

### 输入

```json
{
  "tripId": "trip_123",
  "dayIndex": 2,
  "replanOptionId": "opt_cancel",
  "userConfirmed": true
}
```

### 输出

```json
{
  "planId": "plan_125",
  "changeSummary": "已取消小鱼山，保留晚餐，更新今日计划",
  "todayArtifact": {}
}
```

---

## 9. Review / Memory Tools

## 9.1 `generate_day_log`

### 目的

生成每日日志。

### 输入

```json
{
  "tripId": "trip_123",
  "dayIndex": 2
}
```

### 输出

```json
{
  "dayLogId": "daylog_123",
  "summary": "今天原计划完成 3 个地点，取消 1 个地点...",
  "completedNodes": [],
  "skippedNodes": [],
  "events": [],
  "learnings": []
}
```

---

## 9.2 `generate_trip_log`

### 目的

生成整趟旅行总结。

### 输入

```json
{ "tripId": "trip_123" }
```

### 输出

```json
{
  "tripLogId": "triplog_123",
  "summary": "这次青岛 4 天整体节奏适合家庭...",
  "highlights": [],
  "failures": [],
  "recommendationsForNextTrip": []
}
```

---

## 9.3 `extract_family_memory_candidates`

### 目的

从日志和事件中提取家庭旅行 Memory 候选。

### 输入

```json
{
  "tripId": "trip_123",
  "dayLogIds": [],
  "tripLogId": "triplog_123"
}
```

### 输出

```json
{
  "candidates": [
    {
      "id": "mem_candidate_1",
      "category": "stamina",
      "rule": "父母同行时，下午 3 点后不适合安排高步行强度景点",
      "confidence": 0.72,
      "evidence": ["Day 2 老人疲劳事件", "Day 3 下午取消活动"]
    }
  ]
}
```

### 规则

- 只生成候选，不自动写入 accepted memory。

---

## 9.4 `accept_family_memory`

### 目的

用户确认 Memory 候选后写入长期记忆。

### 输入

```json
{
  "tripId": "trip_123",
  "candidateIds": ["mem_candidate_1"],
  "accepted": true
}
```

### 输出

```json
{
  "acceptedMemoryIds": ["mem_123"]
}
```

---

## 9.5 `get_accepted_family_memory`

### 目的

返回用户已确认的长期家庭旅行 Memory，用于 Dream / Plan / Travel 阶段的推荐、计划和重排判断。

### 输入

```json
{
  "userId": "user_123",
  "categories": ["stamina", "meal", "lodging"],
  "tripId": "trip_123"
}
```

### 输出

```json
{
  "memories": [
    {
      "id": "mem_123",
      "category": "stamina",
      "rule": "父母同行时，下午 3 点后不适合安排高步行强度景点",
      "condition": "6 人全家 / 城市步行 / 上午已有中高强度活动",
      "effect": "下午优先安排休息、酒店附近或低步行活动",
      "confidence": 0.78,
      "evidenceTripIds": ["trip_123"],
      "evidenceEventIds": ["evt_123"]
    }
  ]
}
```

### 规则

- 只返回 `status = "accepted"` 的 FamilyMemory。
- `candidate` / `rejected` / `archived` Memory 不得参与推荐和计划。
- 如果没有已确认 Memory，返回空数组，不阻塞当前任务。

---

## 10. 工具调用安全规则

1. Agent 需要保存状态时，必须调用工具。
2. Agent 需要事实字段时，必须调用 Place / Geo 工具。
3. Agent 不能直接说“我已更新计划”，除非工具返回成功。
4. 工具失败时，Agent 应向用户解释并给出恢复路径。
5. 对用户高影响操作，工具应返回 `requiresConfirmation: true`。
6. 所有工具调用结果都要可被前端转成 Artifact 更新。
