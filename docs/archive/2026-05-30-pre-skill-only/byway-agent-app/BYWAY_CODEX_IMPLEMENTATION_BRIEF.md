# Byway / 另辟蹊径 — Codex 实现 Brief

> 本文档面向 Codex，用工程语言说明如何实现 Byway。它不是产品 PRD，而是开发时的结构、边界、模块和质量要求。

---

## 1. 实现目标

实现一个移动端网页家庭旅行 Agent：

- 用户通过手机网页与 Agent 对话。
- Agent 可以在用户不知道去哪时推荐目的地。
- Agent 可以在目的地确定后自动研究攻略并生成计划。
- Agent 可以调用高德 / PlaceFact 工具精确定位地点、地址、坐标和路线。
- Agent 可以在旅行中根据突发事件调整今日计划。
- Agent 可以在旅行后生成日志和家庭旅行 Memory。

最终交互形态：

```text
聊天窗口 + 动态旅行 Artifact
```

不是：

```text
表单 + 节点编辑器 + 复杂后台管理系统
```

---

## 2. 推荐技术架构

```text
Frontend:
- Next.js / React PWA 或任意轻量移动 Web 技术
- 聊天 UI + Artifact 卡片

Proxy API:
- 隐藏 Hermes API Key
- 处理鉴权、限流、流式事件转发
- 管理前端可见 API

Agent Runtime:
- Hermes Agent API Server
- 使用 Byway Hermes Skill 约束行为
- 通过 MCP 调用 Byway 工具

Tool Layer:
- Byway MCP Server
- 提供 Trip、Dream、Guide、Place、Plan、Travel、Review、Memory 工具

Storage:
- SQLite / Postgres
- Redis / 内存缓存可选

External Services:
- 高德 Web Service API / 高德 MCP
- 多 LLM Provider
- Web Search / 网页抽取工具
```

---

## 3. 推荐目录结构

```text
byway/
  AGENTS.md
  README.md

  docs/
    BYWAY_CODEX_IMPLEMENTATION_BRIEF.md
    BYWAY_ARCHITECTURE.md
    BYWAY_AGENT_STATE_MACHINE.md
    BYWAY_MCP_TOOL_CONTRACTS.md
    BYWAY_DATA_MODEL.md
    BYWAY_UI_WIREFRAMES.md
    BYWAY_FRONTEND_BACKEND_API.md
    BYWAY_ACCEPTANCE_SCENARIOS.md

  skills/
    byway-family-travel-agent/
      SKILL.md

  apps/
    web/
      src/
        app/
        components/
        lib/
        api/

  services/
    proxy-api/
      src/

    byway-mcp-server/
      src/
        tools/
          trip/
          dream/
          guide/
          place/
          plan/
          travel/
          review/
          memory/
        providers/
          amap/
          llm/
          web/
        persistence/
        schemas/
        tests/

  packages/
    shared-schemas/
      src/
```

如果项目希望更简单，可以合并 `proxy-api` 与 `byway-mcp-server`，但逻辑边界仍需保留。

---

## 4. 服务边界

### 4.1 Web 前端

负责：

- 展示聊天消息。
- 展示旅行 Artifact。
- 提交用户输入、文件、链接、确认动作。
- 接收流式事件。

不负责：

- 不直接调用 LLM。
- 不直接调用高德。
- 不保存权威旅行状态。
- 不实现行程规划算法。

---

### 4.2 Byway Proxy API

负责：

- 用户鉴权。
- 隐藏 Hermes API Key。
- 将前端消息转发给 Hermes。
- 将 Hermes / MCP 的事件转换为前端可消费的流式事件。
- 提供 artifact 查询、确认动作提交、上传文件等 API。

不负责：

- 不做复杂 Agent 推理。
- 不直接替代 MCP 工具层。

---

### 4.3 Hermes Agent

负责：

- 识别用户意图。
- 在 Dream / Plan / Travel / Review 模式间切换。
- 主动追问关键缺失信息。
- 调用 Byway MCP 工具。
- 解释工具结果。
- 在关键变更前请求确认。

不负责：

- 不保存权威业务状态。
- 不编造地点事实。
- 不绕过 MCP 直接修改计划。

---

### 4.4 Byway MCP Server

负责：

- 所有确定性业务操作。
- Trip / Plan / PlaceFact / Event / Memory 的权威读写。
- 高德 / LLM / Web Search 的封装。
- 工具输入输出 schema 校验。
- 错误、置信度、降级策略。

---

## 5. 核心模块清单

### 5.1 Trip State 模块

- 创建旅行。
- 更新旅行摘要。
- 管理旅行阶段。
- 返回当前 Trip State。

### 5.2 Dream 模块

- 处理“不知道去哪”。
- 生成候选目的地。
- 对候选目的地做轻量研究。
- 结合 Family Memory 打分。
- 用户选择目的地后进入 Plan Mode。

### 5.3 Guide 模块

- 自动生成 synthetic guide。
- 吸收用户补充攻略：markdown、文字、链接、截图。
- 合并攻略为 Canonical Guide。
- 识别共识、冲突、风险。

### 5.4 Place / Geo 模块

- 调用高德解析标准 POI。
- 获取坐标、地址、入口坐标、电话、营业时间、评分等可用字段。
- 生成打车地址 / 高德导航链接。
- 估算路线和距离。
- 校验计划是否绕路。

### 5.5 Plan 模块

- 生成旅行计划 Artifact。
- 生成住宿区域、用餐区域、每日路线。
- 保留计划版本。
- 支持确认计划。

### 5.6 Travel / Replan 模块

- 获取今日状态。
- 记录突发事件。
- 生成取消 / 顺延 / 重排三类方案。
- 用户确认后应用调整。

### 5.7 Review / Memory 模块

- 生成每日日志。
- 生成旅行总日志。
- 提取 Family Memory Candidate。
- 用户确认后写入 Accepted Memory。

---

## 6. Agent 运行原则

### 6.1 缺失信息处理

Agent 不应该一开始要求用户填写长表单。它应遵循：

```text
能假设就先假设；
会显著影响计划的，最多一次追问 3 个问题；
假设必须显式说明；
后续用户补充信息后可以重新规划。
```

### 6.2 事实处理

- 地点事实必须来自工具。
- 低置信度必须提示。
- 字段缺失时说“未确认”，不能补写。

### 6.3 计划变更

- 低影响变更可以作为建议。
- 高影响变更必须确认。
- 应用调整必须通过 `apply_replan` 或计划工具完成。

---

## 7. Artifact 要求

前端展示的核心 artifact：

```text
DestinationShortlistArtifact
GuideSummaryArtifact
PlanArtifact
TodayArtifact
ReplanOptionsArtifact
DayLogArtifact
TripLogArtifact
MemoryCandidateArtifact
```

Artifact 必须是结构化 JSON，可被前端稳定渲染。

不要只返回自然语言计划。

---

## 8. 外部 API 要求

### 8.1 高德

用于：

- POI 搜索。
- 地理编码 / 逆地理编码。
- 路径规划 / 距离估算。
- 导航 URI。

所有高德结果必须带：

```text
provider
providerPlaceId
confidence
needsReview
lastResolvedAt
```

### 8.2 多 LLM Provider

用于：

- synthetic guide 生成。
- 攻略摘要与合并。
- 目的地候选研究。
- 日志与 memory 候选生成。

但 LLM 输出必须经过 schema 校验，不得直接作为权威事实。

---

## 9. 质量标准

一个合格实现必须满足：

1. 用户能用一句话开始：
   - “不知道去哪，帮我推荐。”
   - “帮我规划青岛 4 天。”
   - “老人累了，我们晚了 40 分钟。”

2. Agent 能自动识别模式：
   - Dream / Plan / Travel / Review。

3. 关键地点必须被 PlaceFact 工具解析。

4. 旅行中事件必须形成结构化 TravelEvent。

5. 重排必须输出三方案，不得只输出一段聊天建议。

6. 旅行 Memory 必须先作为候选，用户确认后才生效。

---

## 10. 禁止的实现捷径

不要为了快速完成而这样做：

```text
用户输入 -> LLM 直接返回整段攻略文本
```

正确方式是：

```text
用户输入
↓
Hermes 判断意图
↓
调用 MCP 工具
↓
MCP 更新结构化状态 / 调用事实工具
↓
返回 Artifact
↓
Hermes 解释 Artifact
↓
前端渲染 Artifact
```

---

## 11. Codex 实现时的优先检查点

每完成一个模块，检查：

- 是否符合 `AGENTS.md`？
- 是否没有引入传统节点管理 UI？
- 是否所有工具有 schema？
- 是否有 artifact 输出？
- 是否能通过 `BYWAY_ACCEPTANCE_SCENARIOS.md` 中的场景？
- 是否没有让 LLM 编造事实？
