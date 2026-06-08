# Byway / 另辟蹊径 — 数据模型文档

---

## 1. 数据建模原则

Byway 是 Agent 产品，但必须有结构化状态。不要只存聊天记录。

必须区分：

```text
AgentMessage：对话过程
Trip / TripBrief：旅行权威摘要
GuideSource：攻略来源
CanonicalGuide：聚合攻略底稿
PlaceFact：地点事实
RouteFact：路线事实
PlanArtifact：旅行计划
TravelEvent：旅行中真实事件
DayLog / TripLog：旅行日志
FamilyMemory：长期家庭旅行规则
```

---

## 2. 枚举定义

```ts
type TripPhase =
  | "intake"
  | "dream"
  | "destination_shortlist"
  | "planning"
  | "guide_research"
  | "place_fact_resolution"
  | "plan_confirmation"
  | "ready_to_travel"
  | "traveling"
  | "day_review"
  | "trip_review"
  | "memory_extraction"
  | "completed";

type Pace = "relaxed" | "normal" | "packed";

type PlaceFactConfidence = "verified" | "high" | "medium" | "low" | "unresolved";

type PlanNodeType =
  | "activity"
  | "meal"
  | "hotel"
  | "transport"
  | "rest"
  | "buffer"
  | "optional";

type TravelEventType =
  | "late"
  | "current_overrun"
  | "elder_tired"
  | "child_tired"
  | "hungry"
  | "bad_weather"
  | "want_return_hotel"
  | "transport_problem"
  | "place_closed"
  | "queue_too_long"
  | "place_disappointing"
  | "place_exceeded_expectation"
  | "custom";

type ReplanOptionType = "cancel" | "postpone" | "replan" | "no_change";
```

---

## 3. UserProfile

```ts
type UserProfile = {
  id: string;
  displayName: string;
  homeCity?: string;
  defaultOrigin?: string;
  createdAt: string;
  updatedAt: string;
};
```

说明：

- 用户可以只有一个家庭账号。
- 不需要做复杂社交账号体系。

---

## 4. FamilyMember

```ts
type FamilyMember = {
  id: string;
  userId: string;
  name?: string;
  role: "self" | "spouse" | "elder" | "child" | "other";
  ageGroup: "adult" | "elder" | "child_12" | "child_3" | "unknown";
  staminaLevel?: "high" | "medium" | "low";
  foodPreferences?: string[];
  constraints?: string[];
  notes?: string;
  createdAt: string;
  updatedAt: string;
};
```

说明：

- 这些是静态画像。
- 动态经验应进入 FamilyMemory。

---

## 5. Trip

```ts
type Trip = {
  id: string;
  userId: string;
  title?: string;
  phase: TripPhase;
  status: "active" | "archived";
  currentPlanId?: string;
  currentDayIndex?: number;
  createdAt: string;
  updatedAt: string;
};
```

---

## 6. TripBrief

```ts
type TripBrief = {
  tripId: string;
  origin?: string;
  destination?: string;
  destinationSelectedFromCandidateId?: string;
  dateRange?: {
    startDate?: string;
    endDate?: string;
  };
  days?: number;
  monthOrHoliday?: string;
  travelers: {
    familyMemberId?: string;
    role: string;
    ageGroup: string;
  }[];
  pace?: Pace;
  budgetLevel?: "low" | "medium" | "high" | "unknown";
  domesticOrOverseas?: "domestic" | "overseas" | "either" | "domestic_preferred";
  themes?: string[];
  arrivalInfo?: string;
  departureInfo?: string;
  hotelInfo?: string;
  transportationInfo?: string;
  assumptions: string[];
  missingCriticalFields: string[];
  updatedAt: string;
};
```

说明：

- TripBrief 是 Agent 做 Dream / Plan 的核心输入。
- 不要求一开始填全。缺失字段可由 Agent 假设或追问。

---

## 7. AgentMessage

```ts
type AgentMessage = {
  id: string;
  tripId?: string;
  role: "user" | "assistant" | "tool" | "system";
  content: string;
  attachments?: Attachment[];
  toolCallId?: string;
  artifactRefs?: string[];
  createdAt: string;
};

type Attachment = {
  id: string;
  type: "image" | "file" | "url" | "markdown" | "text";
  url?: string;
  filename?: string;
  textContent?: string;
  mimeType?: string;
};
```

说明：

- 对话记录不是权威计划。
- 计划必须存在 PlanArtifact。

---

## 8. GuideSource

```ts
type GuideSource = {
  id: string;
  tripId: string;
  sourceType:
    | "model_generated"
    | "user_markdown"
    | "user_text"
    | "user_url"
    | "xiaohongshu_link"
    | "screenshot"
    | "manual_note";
  title?: string;
  content?: string;
  url?: string;
  attachments?: Attachment[];
  provider?: string;
  model?: string;
  promptRole?: string;
  userNote?: string;
  extractionStatus: "pending" | "extracted" | "failed";
  createdAt: string;
};
```

说明：

- Synthetic Guide 与用户导入攻略使用同一模型。
- 小红书链接不保证可抓取正文。

---

## 9. GuideExtraction

```ts
type GuideExtraction = {
  id: string;
  tripId: string;
  guideSourceId: string;
  lodgingAreas: ExtractedArea[];
  mealAreas: ExtractedArea[];
  places: ExtractedPlace[];
  routeIdeas: ExtractedRouteIdea[];
  warnings: string[];
  assumptions: string[];
  createdAt: string;
};

type ExtractedArea = {
  name: string;
  reason?: string;
  fitForFamily?: "high" | "medium" | "low";
  risks?: string[];
  confidence?: "explicit" | "inferred" | "defaulted";
};

type ExtractedPlace = {
  name: string;
  typeHint?: string;
  suggestedPriority?: 1 | 2 | 3 | 4 | 5;
  recommendedDurationMinutes?: number;
  recommendedTimeOfDay?: "morning" | "afternoon" | "evening" | "any";
  familyFit?: "high" | "medium" | "low";
  elderlyRisk?: "high" | "medium" | "low";
  childRisk?: "high" | "medium" | "low";
  reason?: string;
  warnings?: string[];
  fieldSource?: {
    duration?: "explicit" | "inferred" | "defaulted";
    priority?: "explicit" | "inferred" | "defaulted";
  };
};

type ExtractedRouteIdea = {
  title: string;
  dayRole?: "arrival" | "full_day" | "departure" | "any";
  suggestedDayIndex?: number;
  sequence: string[];
  intensity?: "low" | "medium" | "high";
  reason?: string;
};
```

---

## 10. CanonicalGuide

```ts
type CanonicalGuide = {
  id: string;
  tripId: string;
  version: number;
  sourceGuideIds: string[];
  lodgingAreas: CanonicalArea[];
  mealAreas: CanonicalArea[];
  places: CanonicalPlace[];
  routeIdeas: CanonicalRouteIdea[];
  conflicts: GuideConflict[];
  warnings: string[];
  revisionSummary?: string;
  createdAt: string;
};

type CanonicalArea = {
  id: string;
  name: string;
  areaType: "lodging" | "meal";
  mentionCount: number;
  sourceGuideIds: string[];
  recommendationLevel: "high" | "medium" | "low";
  reasons: string[];
  risks: string[];
  userExcluded?: boolean;
  userLocked?: boolean;
};

type CanonicalPlace = {
  id: string;
  name: string;
  aliases: string[];
  typeHint?: string;
  mentionCount: number;
  sourceGuideIds: string[];
  suggestedPriority: 1 | 2 | 3 | 4 | 5;
  userPriorityOverride?: 1 | 2 | 3 | 4 | 5;
  finalPriority: 1 | 2 | 3 | 4 | 5;
  reasons: string[];
  warnings: string[];
  placeFactId?: string;
  userExcluded?: boolean;
  userLocked?: boolean;
};

type CanonicalRouteIdea = {
  id: string;
  title: string;
  routeFamily?: string;
  dayRole?: "arrival" | "full_day" | "departure" | "any";
  suggestedDayIndex?: number;
  placeNames: string[];
  placeIds?: string[];
  intensity?: "low" | "medium" | "high";
  sourceGuideIds: string[];
  confidence: number;
};

type GuideConflict = {
  id: string;
  conflictType: "lodging_area" | "place_priority" | "route" | "warning";
  description: string;
  sources: string[];
  suggestedResolution?: string;
};
```

说明：

- CanonicalGuide 是计划生成的攻略底稿。
- 它不等于旅行计划。

---

## 11. PlaceFact

```ts
type PlaceFact = {
  id: string;
  tripId?: string;
  inputName: string;
  standardName?: string;
  aliases: string[];

  provider: "amap" | "google" | "manual" | "unknown";
  providerPlaceId?: string;

  city?: string;
  district?: string;
  adcode?: string;

  coordinate?: {
    lng: number;
    lat: number;
    system: "GCJ02" | "WGS84";
  };

  address?: string;
  displayAddress?: string;
  taxiAddress?: string;
  entranceCoordinate?: {
    lng: number;
    lat: number;
    system: "GCJ02" | "WGS84";
  };
  navigationPoiId?: string;

  phone?: string | null;
  openingHoursToday?: string | null;
  openingHoursWeek?: string | null;
  rating?: number | null;
  photos?: string[];
  businessArea?: string | null;

  confidence: PlaceFactConfidence;
  needsReview: boolean;
  reviewReason?: string;
  source: "amap_search" | "amap_id" | "geocode" | "manual" | "unknown";
  lastResolvedAt?: string;
};
```

硬规则：

- 坐标、电话、营业时间、评分等不能由 LLM 编造。
- 字段缺失时保持 null。
- 低置信度核心地点必须触发用户确认。

---

## 12. RouteFact

```ts
type RouteFact = {
  id: string;
  tripId: string;
  fromPlaceFactId: string;
  toPlaceFactId: string;
  mode: "walking" | "driving" | "transit" | "unknown";
  provider: "amap" | "manual" | "estimated";
  distanceMeters?: number;
  durationMinutes?: number;
  confidence: "high" | "medium" | "low";
  warning?: string;
  lastEstimatedAt: string;
};
```

说明：

- RouteFact 是路线估算，不是导航级实时路线。

---

## 13. PlanArtifact

```ts
type PlanArtifact = {
  id: string;
  tripId: string;
  version: number;
  status: "draft" | "confirmed" | "active" | "completed";
  title: string;
  days: PlanDay[];
  lodgingRecommendation?: AreaRecommendation;
  mealRecommendations?: AreaRecommendation[];
  assumptions: string[];
  warnings: PlanWarning[];
  sourceCanonicalGuideId?: string;
  createdAt: string;
  updatedAt: string;
};

type PlanDay = {
  id: string;
  dayIndex: number;
  title: string;
  date?: string;
  dayRole: "arrival" | "full_day" | "departure" | "unknown";
  summary: string;
  nodes: PlanNode[];
  warnings: PlanWarning[];
};

type PlanNode = {
  id: string;
  type: PlanNodeType;
  title: string;
  description?: string;
  placeFactId?: string;
  canonicalPlaceId?: string;
  plannedStart?: string;
  plannedEnd?: string;
  priority?: 1 | 2 | 3 | 4 | 5;
  status: "pending" | "current" | "completed" | "skipped" | "postponed";
  anchor: boolean;
  optional: boolean;
  reason?: string;
  warnings?: string[];
};

type AreaRecommendation = {
  areaName: string;
  reason: string;
  risks: string[];
  confidence: "high" | "medium" | "low";
};

type PlanWarning = {
  severity: "low" | "medium" | "high";
  message: string;
  affectedNodeIds?: string[];
  suggestedFix?: string;
};
```

规则：

- meal / hotel / rest 是 anchor，不应被普通 activity 挤掉。
- PlanArtifact 必须可版本化。

---

## 14. DestinationShortlistArtifact

```ts
type DestinationShortlistArtifact = {
  id: string;
  tripId: string;
  candidates: DestinationCandidate[];
  summary: string;
  assumptions: string[];
  createdAt: string;
};

type DestinationCandidate = {
  id: string;
  destination: string;
  score: number;
  recommendedDays: {
    minimumComfortable: number;
    ideal: number;
  };
  familyFit: "high" | "medium" | "low";
  timeFit: "high" | "medium" | "low";
  transportFit: "high" | "medium" | "low";
  lodgingConvenience?: "high" | "medium" | "low";
  mealConvenience?: "high" | "medium" | "low";
  mainRisks: string[];
  whyRecommended: string;
  whyNot?: string;
  planPreview?: string[];
};
```

---

## 15. TodayArtifact

```ts
type TodayArtifact = {
  id: string;
  tripId: string;
  planId: string;
  dayIndex: number;
  currentNode?: PlanNode;
  nextNode?: PlanNode;
  remainingNodes: PlanNode[];
  mealAnchors: PlanNode[];
  hotelAnchor?: PlanNode;
  events: TravelEvent[];
  updatedAt: string;
};
```

---

## 16. TravelEvent

```ts
type TravelEvent = {
  id: string;
  tripId: string;
  dayIndex: number;
  relatedNodeId?: string;
  eventType: TravelEventType;
  severity: 1 | 2 | 3 | 4 | 5;
  userText: string;
  context: {
    delayMinutes?: number;
    weather?: string;
    mealStatus?: string;
    stepCount?: number;
    affectedTravelers?: string[];
  };
  createdAt: string;
};
```

---

## 17. ReplanOptionsArtifact

```ts
type ReplanOptionsArtifact = {
  id: string;
  tripId: string;
  dayIndex: number;
  eventId: string;
  recommendedOptionId: string;
  options: ReplanOption[];
  createdAt: string;
};

type ReplanOption = {
  id: string;
  type: ReplanOptionType;
  summary: string;
  cancelledNodeIds: string[];
  postponedNodeIds: string[];
  updatedTimes?: {
    nodeId: string;
    oldStart?: string;
    newStart?: string;
    oldEnd?: string;
    newEnd?: string;
  }[];
  preservedNodeIds: string[];
  warnings: string[];
  requiresConfirmation: boolean;
  reason: string;
};
```

---

## 18. DayLog

```ts
type DayLog = {
  id: string;
  tripId: string;
  dayIndex: number;
  summary: string;
  plannedSummary: string;
  actualSummary: string;
  completedNodeIds: string[];
  skippedNodeIds: string[];
  travelEventIds: string[];
  learnings: string[];
  createdAt: string;
};
```

---

## 19. TripLog

```ts
type TripLog = {
  id: string;
  tripId: string;
  summary: string;
  highlights: string[];
  failures: string[];
  recommendationsForNextTrip: string[];
  dayLogIds: string[];
  createdAt: string;
};
```

---

## 20. FamilyMemory

```ts
type FamilyMemory = {
  id: string;
  userId: string;
  status: "candidate" | "accepted" | "rejected" | "archived";
  category:
    | "stamina"
    | "meal"
    | "transport"
    | "lodging"
    | "pacing"
    | "interest"
    | "child"
    | "elder"
    | "weather"
    | "risk"
    | "other";
  rule: string;
  condition?: string;
  effect?: string;
  confidence: number;
  evidenceTripIds: string[];
  evidenceEventIds: string[];
  lastVerifiedAt?: string;
  createdAt: string;
  updatedAt: string;
};
```

规则：

- Candidate 不参与规划。
- Accepted 才能进入 Dream / Plan / Travel 推理。

---

## 21. PendingConfirmation

```ts
type PendingConfirmation = {
  id: string;
  tripId: string;
  type:
    | "select_destination"
    | "confirm_plan"
    | "apply_replan"
    | "accept_memory"
    | "confirm_place_fact"
    | "major_plan_change";
  summary: string;
  payload: Record<string, unknown>;
  status: "pending" | "accepted" | "rejected" | "expired";
  createdAt: string;
  expiresAt?: string;
};
```

---

## 22. ArtifactSnapshot

```ts
type ArtifactSnapshot = {
  id: string;
  tripId: string;
  artifactType:
    | "DestinationShortlistArtifact"
    | "GuideSummaryArtifact"
    | "PlanArtifact"
    | "TodayArtifact"
    | "ReplanOptionsArtifact"
    | "DayLogArtifact"
    | "TripLogArtifact"
    | "MemoryCandidateArtifact";
  artifactId: string;
  version: number;
  content: Record<string, unknown>;
  createdAt: string;
};
```

说明：

- 前端可通过 artifact snapshot 恢复页面状态。
- 每次重要更新都应该产生新 snapshot。

---

## 23. 数据一致性要求

1. `Trip.currentPlanId` 必须指向当前权威计划。
2. `PlanArtifact.version` 每次应用重排或重大修改递增。
3. `TravelEvent` 不得删除，只能标记更正。
4. `PlaceFact` 字段不得由 LLM 伪造。
5. `FamilyMemory.status = accepted` 才能参与后续规划。
6. `PendingConfirmation` 未处理前不得应用对应高影响变更。
