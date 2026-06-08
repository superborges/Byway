# Phase 1.9 攻略研究与用户攻略吸收设计

## 背景

当前 Byway 已经能从 Dream 走到 Plan，并能用 Mock PlaceFact 生成可执行计划。但 Plan Mode 仍然少了文档中定义的攻略层：`generate_synthetic_guides`、`ingest_user_guide_source`、`merge_guides`。这会让计划看起来像直接从用户输入生成，而不是先形成可追踪的 Canonical Guide。

Phase 1.9 的目标是补齐验收场景 B1、B3、B4 的最小闭环。仍然使用 Mock Provider，不抓取真实小红书，不接真实 LLM。

## 产品原则

攻略层只负责“判断与建议”，事实层仍然交给 PlaceFact / RouteFact。Byway 可以说“用户攻略里提到了五四广场住宿”，但不能把链接中没有读到的内容说成已经确认。

## 范围

### 本阶段实现

1. Shared Schemas 增加攻略相关 canonical schema：
   - `GuideSource`
   - `GuideExtraction`
   - `GuideSummaryArtifact`

2. MCP mock server 增加 Guide 工具：
   - `generate_synthetic_guides`
   - `ingest_user_guide_source`
   - `merge_guides`

3. Proxy API 增加三条 orchestration：
   - Plan Mode 生成计划前调用 synthetic guides 和 merge guides。
   - 用户粘贴 markdown 攻略时，保存、抽取、合并，并返回攻略变化摘要。
   - 用户输入小红书链接时，保存链接和用户备注；如果无法读取正文，明确说明没有完整阅读。

4. Web 复用现有聊天流、工具进度、artifact preview / workspace：
   - 展示 `GuideSummaryArtifact` 的新增地点、强化区域、冲突、warning。

### 本阶段不实现

- 不抓取真实网页正文。
- 不做 OCR。
- 不做多模型真实研究。
- 不自动应用用户攻略导致的重大计划变更，只询问是否基于新攻略更新计划。

## 数据模型

### GuideSource

保存攻略来源，包括 synthetic guide、markdown、文本、URL、小红书链接、截图备注。

关键字段：

- `sourceType`
- `title`
- `content`
- `url`
- `provider`
- `model`
- `promptRole`
- `userNote`
- `extractionStatus`

### GuideExtraction

保存从单个来源中抽出的结构化信息：

- `lodgingAreas`
- `mealAreas`
- `places`
- `routeIdeas`
- `warnings`
- `assumptions`

### GuideSummaryArtifact

作为前端可展示 artifact，描述一次合并后的 Canonical Guide 摘要：

- `canonicalGuideId`
- `sourceIds`
- `lodgingAreas`
- `mealAreas`
- `places`
- `routeIdeas`
- `conflicts`
- `warnings`
- `revisionSummary`
- `changeSummary`

## 数据流

### B1：直接 Plan Mode

```text
用户：帮我规划青岛 4 天...
-> create_trip
-> update_trip_brief
-> generate_synthetic_guides
-> merge_guides
-> resolve_places_batch
-> estimate_route_matrix
-> generate_plan
-> verify_plan_geo
-> artifact_updated: GuideSummaryArtifact
-> artifact_updated: PlanArtifact
```

### B3：粘贴 markdown 攻略

```text
用户：# 青岛四天三晚亲子攻略...
-> ingest_user_guide_source(sourceType=user_markdown)
-> merge_guides(preserveUserOverrides=true)
-> artifact_updated: GuideSummaryArtifact
-> assistant_message_delta:
   我吸收了这份攻略：新增 2 个地点，强化 1 个住宿区域，发现 1 个冲突。
   如果你愿意，我可以基于新攻略更新当前计划。
```

### B4：小红书链接

```text
用户：这个小红书链接是讲五四广场住宿的：https://...
-> ingest_user_guide_source(sourceType=xiaohongshu_link)
-> merge_guides
-> assistant_message_delta:
   我已保存这个链接；当前 mock 阶段无法读取完整笔记正文。
   我只基于你的备注强化“五四广场住宿”判断。
```

## Mock 抽取规则

1. 文本包含“八大关 / 小麦岛 / 崂山 / 栈桥”时抽为 places。
2. 文本包含“五四广场住宿 / 奥帆中心住宿”时抽为 lodgingAreas。
3. 文本包含“台东 / 老城 / 酒店附近吃饭”时抽为 mealAreas。
4. 文本包含“老人累 / 崂山累 / 暑期人多”时抽为 warnings。
5. 小红书链接默认不读取正文；如果用户备注包含“五四广场住宿”，只把该备注抽为 lodgingAreas，并增加 warning。

## 验收标准

1. Plan Mode 的 SSE 工具顺序中包含 `generate_synthetic_guides`、`merge_guides`、`verify_plan_geo`。
2. Plan Mode 返回 `GuideSummaryArtifact` 和 `PlanArtifact`。
3. 粘贴 markdown 攻略后，回答包含“新增”“强化”“冲突”，并询问是否基于新攻略更新计划。
4. 小红书链接处理后，回答包含“已保存链接”“无法读取完整笔记”“五四广场住宿”。
5. 所有新增行为有 MCP、Proxy、E2E 测试覆盖。
