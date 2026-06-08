# Phase 2.2 核心地点解析与歧义确认设计

## 背景

当前 Plan Mode 会解析部分地点，并在“老街”歧义时生成 PlaceFactReviewArtifact。但验收场景 C1/C2 要求更明确：

- 计划前应解析 CanonicalGuide 中 4/5 星核心地点：八大关、小麦岛、崂山、栈桥。
- 对“老街”这类歧义地点，应自然语言问用户确认，而不是只展示卡片。
- 确认前不能把歧义地点当作高置信度核心节点。

## 范围

### 本阶段实现

1. Proxy Plan Mode 的 `resolve_places_batch` 输入扩展为：
   - 八大关
   - 小麦岛
   - 崂山
   - 栈桥
   - 五四广场
   - 老街

2. 如果 `resolve_places_batch` 产生 `needsReview`：
   - assistant message 输出：`我找到了几个可能的“老街”，你指的是哪一个？`
   - 保留 PlaceFactReviewArtifact。

3. PlanArtifact 中核心节点引用 PlaceFact：
   - Day 2 八大关引用 `pf_八大关`
   - Day 3 崂山引用 `pf_崂山`

### 本阶段不实现

- 不做用户确认后的 PlaceFact 写回。
- 不做多候选地点选择 UI。
- 不接真实高德多候选。

## 验收标准

1. Plan Mode 的 `resolve_places_batch` 返回 resolved 中包含八大关、小麦岛、崂山、栈桥。
2. Plan Mode 返回 PlaceFactReviewArtifact。
3. assistant message 包含“我找到了几个可能的‘老街’，你指的是哪一个？”
4. PlanArtifact 中至少八大关、崂山节点有 placeFactId。
