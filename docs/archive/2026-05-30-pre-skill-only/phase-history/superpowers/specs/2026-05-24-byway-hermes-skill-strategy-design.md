# Byway Skill 设计策略

## 背景

Byway 的核心不是把旅行流程做成一组死板规则，而是让 Backend Agent 成为真正的旅行 Agent：能理解用户自然语言、维护 trip 上下文、主动选择工具、解释取舍，并在关键动作前请求确认。

Phase 4.0 后，Skill 分为两类：

- `skills/byway-family-travel-agent/SKILL.md`：Hermes 家庭入口 Skill，负责识别旅行意图、携带家庭上下文、把任务转交 Backend HTTP 或 Backend MCP Adapter。
- `skills/byway-runtime/*.md`：Backend Agent 的 Runtime Skills，负责 Dream / Plan / Plan Edit / Travel / Review / Memory 的策略、工具剧本和边界。

这两类 Skill 都不能变成规则引擎。它们应该给 Agent 智能空间，同时用 ToolGateway、PlaceFact、RouteFact、Confirmation 和 Trace 把事实、状态和执行边界固定住。

一句话定义：

> Runtime Skills 是 Backend Agent 的行为宪法；Hermes Skill 是家庭入口协议；ToolGateway 是执行层的法律和账本；Trace 是审计记录。

## Skill 的定位

Byway Skill 体系同时承担三类职责。

第一，它是意图理解手册。Runtime Skills 告诉 Backend Agent 如何根据用户语言和当前 trip context 判断用户是在 Dream、Plan、Plan Edit、Place QA、Travel、Review、Memory 还是 Boundary 场景中。Hermes Skill 只判断家庭聊天里的旅行需求是否应该 handoff 给 Byway。

第二，它是工具协作策略。Runtime Skills 告诉 Backend Agent 在不同场景下应该怎样和 ToolGateway 协作，例如什么时候先读 `get_trip_state`，什么时候调用 `resolve_places_batch`，什么时候更新 `PlanArtifact`，什么时候必须创建或消费 `PendingConfirmation`。

第三，它是事实与安全边界。它明确哪些内容可以推理，哪些内容必须来自工具。地点名称、经纬度、地址、电话、营业时间、路线耗时、打车点和评分不能编造；确认计划、应用重排、接受 Memory 等高影响动作不能绕过确认。

Skill 不应该承担这些职责：

- 不持久化状态。
- 不替代 MCP tool contract。
- 不保存用户隐私、API key、session token 或生产配置。
- 不维护城市词典、关键词表或大量硬编码映射。
- 不直接规定每个具体城市、景点、餐厅的答案。

## 七层设计

### 1. 产品人格与优先级

Byway 不是普通攻略机器人，而是家庭旅行 Agent。Skill 应该把产品优先级稳定写清楚：

```text
安全和隐私 > 事实可靠 > 用户确认 > 老人孩子和吃饭 > 行程可执行 > 景点数量
```

这个优先级用于压住模型常见倾向：为了显得丰富而塞满景点、为了显得确定而编造事实、为了显得主动而越过确认。

回复风格也应服务这个定位：清楚、可执行、懂家庭旅行，有假设就明说，有不确定性就标出，不输出大段泛泛攻略。

### 2. 模式识别

Skill 应该定义稳定的工作模式，而不是用关键词分支替代理解。

核心模式包括：

- Dream：用户不知道去哪，先生成候选目的地，不直接生成完整计划。
- Plan：用户已知目的地，研究攻略、解析地点、估算路线、生成计划。
- Plan Edit：用户基于已有计划做自然语言修改。
- Place QA：用户问地点事实、营业时间、电话、路线、打车点。
- Travel：用户在旅行中报告突发事件或现场反馈。
- Review：用户要求生成 DayLog、TripLog 或旅行总结。
- Memory：用户确认或查询家庭旅行记忆。
- Boundary：用户要求预订付款、无授权抓取、暴露敏感信息等。

其中 Plan Edit 是 Agent 产品感最强的能力。用户说“第三天可以改成灵隐寺吗”时，Backend Agent 不应该重新套模板，也不应该猜城市，而应该读取当前 trip 和 plan，在上下文里完成修改。

### 3. 状态纪律

Runtime Skills 必须让 Backend Agent 形成一个固定习惯：只要用户在继续某个旅行，就先读权威状态，而不是靠聊天历史猜。

关键规则：

- 如果用户没有明确说新目的地，不要覆盖当前 `destination`。
- 如果用户说“第三天”“第四天”“今天下午”，必须先读当前 `PlanArtifact` 或 `TodayArtifact`。
- 如果用户说“应用推荐方案”，必须查 pending confirmation。
- 如果用户在 Plan Edit 中新增地点，先解析 PlaceFact，再更新计划。
- 如果用户的表达模糊但不阻塞，可以先给假设；如果会显著影响计划，最多一次问 3 个问题。

这层规则直接服务一个核心验收目标：用户选杭州后，后续自然语言修改不能聊着聊着变成青岛。

### 4. 工具剧本

Runtime Skills 应该提供工具协作 playbook，但不要把 playbook 写成不可变脚本。Backend Agent 可以根据上下文调整顺序，但必须遵守工具边界。

Dream 常见路径：

```text
update_trip_brief
get_accepted_family_memory
generate_destination_candidates
research_destination_candidate
score_destination_candidates
```

Plan 常见路径：

```text
update_trip_brief
generate_synthetic_guides 或 ingest_user_guide_source
merge_guides
resolve_places_batch
estimate_route_matrix
generate_plan
verify_plan_geo
```

Plan Edit 常见路径：

```text
get_trip_state
get_current_plan
resolve_places_batch
estimate_route_matrix
update_plan_artifact
verify_plan_geo
```

Travel 常见路径：

```text
get_today_status
record_travel_event
replan_today
等待用户确认
apply_replan
```

Review / Memory 常见路径：

```text
generate_day_log
generate_trip_log
extract_family_memory_candidates
等待用户确认
accept_family_memory
```

这些剧本是策略，不是正则规则。Backend Agent 的价值在于能根据自然语言、当前计划、家庭偏好和工具结果做判断。

### 5. 事实边界

Skill 要非常硬地约束事实来源。

这些字段不能由 Agent 编造：

- 标准 POI 名称。
- 经纬度。
- 地址。
- 打车地址。
- 入口坐标。
- 电话。
- 营业时间。
- 评分。
- 路线耗时。
- 是否闭馆。
- 门票预约状态。

需要这些字段时，必须使用 PlaceFact / RouteFact 相关工具。如果工具没有返回，应该说“这个字段我目前没有可靠来源确认”，而不是猜。

这层边界让 Byway 既像 Agent，又不像一个会编细节的攻略生成器。

### 6. 确认边界

Skill 应该区分“可以直接草拟”和“必须用户确认”。

可以直接做：

- 生成候选目的地。
- 生成旅行草案。
- 查询地点事实。
- 解释计划风险。
- 生成未生效的重排选项。

必须确认：

- `confirm_plan`
- `apply_replan`
- `accept_family_memory`

需要重点提示并请求确认：

- 取消高优先级景点。
- 改变酒店、交通、餐点 anchor。
- 影响当天现实行程。
- 修改已确认计划。
- 写入长期 Family Memory。

这能防止 Agent “太主动”，也能让用户始终掌握高影响动作。

### 7. Trace 友好

Skill 不是只给模型看的，也要服务开发和验收。

每轮 Byway chat turn 的 Trace 应能回答：

- Backend Agent 加载了哪些 Runtime Skills？如果来自家庭入口，Hermes 加载了哪个 entry Skill？
- Skill 的 version 和 hash 是什么？
- Backend Agent 判断用户处于哪个模式？
- 为什么调用这些 ToolGateway tools？
- 每个 tool 的输入输出摘要是什么？
- 错误发生在 Agent、ToolGateway、MCP Adapter、Provider 还是前端 refresh？

因此 Skill 中应要求 Agent 行为可追踪：不要假装工具成功，不要把工具失败包装成完成，不要在缺少事实来源时输出确定答案。

## 反模式

最需要避免的反模式是把 Proxy 里的规则搬进 Skill。

不要写成：

```text
如果出现“灵隐寺”，就认为目的地是杭州。
如果出现“老人累了”，就固定选择 cancel。
如果出现“下雨”，就固定推荐博物馆。
```

这种写法会让 Skill 变成脆弱的关键词系统。正确做法是写判断原则：

```text
如果用户提到新增地点，先读取当前 trip destination 和 plan，再解析新增地点，最后判断路线和体力影响。
如果用户报告疲劳，先读取 TodayArtifact，记录 TravelEvent，再生成多个重排选项并请求确认。
```

另一个反模式是 Skill 过长。Skill 应保持稳定、紧凑、可记忆。城市知识、具体景点知识、工具参数细节不应该堆在 Skill 里。工具契约属于 ToolGateway / shared schema，事实属于 PlaceFact / RouteFact，长期偏好属于 FamilyMemory。

## 推荐结构

`skills/byway-family-travel-agent/SKILL.md` 建议控制在 150-250 行左右，包含：

- 身份和最高优先级。
- 工作模式。
- 状态纪律。
- 工具剧本。
- 事实边界。
- 确认边界。
- 缺失信息处理。
- Travel Mode 重排策略。
- Review / Memory 策略。
- Trace 与失败处理。
- 回复风格。

如果未来内容继续增长，应拆成 Runtime Skills 或引用型文档，而不是让主 Skill 无限膨胀。

## 与 Phase 4.0 Hybrid 架构的关系

本文最初服务于 Hermes-first 方案。经过后续讨论，Phase 4.0 已调整为一体化 Backend Hybrid Agent 架构：

```text
Byway Web -> Backend HTTP -> Backend Agent -> ToolGateway -> Domain / DB / Providers
Hermes 家庭入口 -> Hermes Byway Skill -> Backend HTTP / Backend MCP Adapter -> ToolGateway -> 同一套 Byway 状态
```

其中：

- Backend HTTP 负责网关、鉴权、SSE、Trace、Artifact refresh。
- Backend Agent 负责 Web 产品主链路的理解、编排、checkpoint、人机确认和工具选择。
- Hermes 负责家庭 Agent 入口、日常对话、家庭记忆和 Skill 生态复用。
- Hermes Byway Skill 负责识别家庭聊天中的旅行意图，并把任务转交 Backend，不再承担完整旅行规划。
- Byway Runtime Skills 负责 Backend Agent 内部的 Dream / Plan / Plan Edit / Travel / Review 策略。
- ToolGateway / Byway Tools 负责确定性执行和权威状态。
- Backend MCP Adapter 只是协议适配层，让 Hermes 或外部 Agent 调同一套 ToolGateway。
- PlaceFact / RouteFact 负责地点和路线事实。
- Trace 负责审计和验收。

实现时，如果发现 HTTP routes 又开始大量判断自然语言意图，应回到本文和 `2026-05-24-phase-4-hybrid-agent-runtime-design.md` 检查边界：这类逻辑通常应该进入 Backend Agent + Runtime Skills，而不是留在 HTTP routes；如果发生在 Hermes 家庭入口，则 Hermes Skill 只负责 handoff 和家庭上下文传递。

## 验收判断

一个好的 Byway Skill 体系应该让这些场景成立：

- 用户说“不知道去哪”，Backend Agent 生成候选目的地，不直接生成完整计划。
- 用户选杭州后说“第三天可以改成灵隐寺吗”，Backend Agent 不丢失杭州上下文。
- 用户问“几点关门”，Backend Agent 使用 PlaceFact，不编造营业时间。
- 用户说“老人累了，我们晚了 40 分钟”，Backend Agent 生成取消、顺延、重排选项，不直接应用。
- 用户说“应用推荐方案”，Backend Agent 查 pending confirmation 后调用 `apply_replan`。
- 用户说“这次旅行结束了”，Backend Agent 生成 TripLog 和 Memory candidates，但不自动接受 Memory。
- Hermes 家庭入口提出旅行需求时，Hermes Skill 把任务转交 Backend HTTP 或 Backend MCP Adapter，而不是在 Hermes 内完整生成计划。
- Trace 能看到 Skill version/hash、Agent 意图判断、ToolGateway tools、Provider 结果和 SSE 输出。
