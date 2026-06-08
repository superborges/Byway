# Byway / 另辟蹊径 — Schema 与契约统一索引

---

## 1. 文档目标

本文用于解决 PRD、实现 Brief、数据模型、MCP 契约和 Hermes Skill 之间的命名差异。实现时如果文档之间出现冲突，按本索引指定的 canonical source 为准。

---

## 2. Canonical Source 顺序

1. `AGENTS.md`：产品边界和最高优先级规则。
2. `docs/BYWAY_CONTRACT_INDEX.md`：schema / tool / artifact / enum 命名统一入口。
3. `docs/BYWAY_DATA_MODEL.md`：业务对象、枚举、Artifact schema 的权威来源。
4. `docs/BYWAY_MCP_TOOL_CONTRACTS.md`：MCP 工具名、输入输出、副作用的权威来源。
5. `docs/BYWAY_FRONTEND_BACKEND_API.md`：前后端 HTTP API 与流式事件协议来源。
6. `skills/byway-family-travel-agent/SKILL.md`：Hermes Agent 行为和工具调用策略来源。
7. `docs/BYWAY_AGENT_FINAL_PRD_WITH_AGENT_CAPABILITIES.md`：产品叙事和最终能力范围来源；其中 schema 片段只作为解释，不作为实现源。

---

## 3. 命名规则

- JSON 字段和枚举值统一使用 `camelCase` 字段名 + `snake_case` 枚举值。
- TypeScript 类型名、Artifact 类型名统一使用 `PascalCase`。
- MCP 工具名统一使用 `snake_case`。
- 用户可见中文文案不受上述规则限制。

---

## 4. Canonical Artifact 名称

实现中使用以下 Artifact 名称：

```text
DestinationShortlistArtifact
GuideSummaryArtifact
PlaceFactReviewArtifact
PlanArtifact
TodayArtifact
ReplanOptionsArtifact
DayLogArtifact
TripLogArtifact
MemoryCandidateArtifact
```

统一约定：

- 使用 `PlanArtifact`，不使用 `TravelPlan` 作为 Artifact 类型名。
- `PlanArtifact.version` 是计划版本追踪字段。
- 所有重要 Artifact 更新必须产生 `ArtifactSnapshot`。

---

## 5. Canonical TripPhase

实现中使用 `docs/BYWAY_DATA_MODEL.md` 的细粒度阶段：

```text
intake
dream
destination_shortlist
planning
guide_research
place_fact_resolution
plan_confirmation
ready_to_travel
traveling
day_review
trip_review
memory_extraction
completed
```

前端可以把这些阶段映射成更简短的展示状态，例如“规划中”“待出发”“旅行中”“已完成”，但后端存储和 API 返回使用上述 canonical 值。

---

## 6. Canonical TravelEventType

实现中使用以下事件类型：

```text
late
current_overrun
elder_tired
child_tired
hungry
bad_weather
want_return_hotel
transport_problem
place_closed
queue_too_long
place_disappointing
place_exceeded_expectation
custom
```

旧文档中的 `overrun`、`childTired`、`elderTired`、`badWeather`、`wantReturnHotel`、`transportIssue`、`placeClosed`、`queueTooLong` 只作为叙事别名，不进入代码。

---

## 7. Canonical MCP 工具名

以 `docs/BYWAY_MCP_TOOL_CONTRACTS.md` 中已定义的工具为准。当前统一后的关键命名：

```text
update_plan_artifact
accept_family_memory
get_accepted_family_memory
create_amap_navigation_link
```

旧文档中的以下名称不再作为实现名：

```text
revise_plan_with_new_info
confirm_family_memory
open_amap_navigation_link
```

如果未来确实需要这些语义，应在 MCP 契约文档中新增正式工具，而不是在 PRD 或 Skill 中临时引用。

---

## 8. 确认边界

以下动作必须创建 `PendingConfirmation`，用户确认后才允许修改权威状态：

- 选择目的地。
- 确认计划。
- 应用重排方案。
- 接受 Family Memory。
- 确认低置信度核心 PlaceFact。
- 取消或移动 4/5 星核心节点。
- 改变明天及之后的计划。
- 改变 hotel / meal / rest / transport anchor。

---

## 9. 实现检查清单

每次新增 schema、工具或 API 时检查：

1. 是否在 `BYWAY_DATA_MODEL.md` 或 `BYWAY_MCP_TOOL_CONTRACTS.md` 中有 canonical 定义。
2. Artifact 类型名是否在本索引的列表中。
3. 枚举值是否使用 snake_case。
4. 高影响动作是否返回或创建 confirmation。
5. PlaceFact / RouteFact 字段是否来自工具，而不是 LLM 推断。

---

## 10. 已命名但尚未正式契约化的工具

PRD 中出现过以下最终能力工具，但当前 `BYWAY_MCP_TOOL_CONTRACTS.md` 尚未定义完整输入/输出/副作用。实现前必须先补充正式契约：

```text
list_user_trips
extract_guide_source
summarize_guide_consensus
summarize_guide_conflicts
update_plan_assumptions
save_plan_version
start_travel_day
mark_node_completed
mark_node_skipped
```

这些名称可以作为未来工具候选，不应在代码里直接调用，直到它们进入 MCP 工具契约文档。
