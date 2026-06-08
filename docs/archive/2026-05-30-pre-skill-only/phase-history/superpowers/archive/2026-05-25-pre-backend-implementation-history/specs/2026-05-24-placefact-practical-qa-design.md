# Phase 1.8 PlaceFact 实用问答与打车地址设计

## 背景

Byway 的核心可信度来自 PlaceFact：地点名称、地址、坐标、入口、营业时间、打车地址等事实不应由 LLM 临场编造。当前系统已经能在生成计划前通过 `resolve_places_batch` 产生 PlaceFact，但用户在旅行中直接问“八大关几点关门？”或“打车去崂山到哪里？”时，Proxy 还没有把问题路由到地点事实工具。

Phase 1.8 补齐验收场景 C3/C4：

- `get_place_detail`
- `suggest_taxi_address`

本阶段仍使用 Mock Provider，不接真实高德。

## 产品原则

> 地点事实宁可说“不可靠”，也不要猜。

当字段可靠时，Byway 可以回答并说明来源和置信度。当字段缺失时，必须明确说“当前未获取到可靠营业时间 / 入口信息”，并建议出发前确认。

## 范围

### 本阶段实现

1. MCP mock server 增加：
   - `get_place_detail(placeFactId, fields)`
   - `suggest_taxi_address(placeFactId, context)`

2. Proxy API 识别：
   - 营业时间问题：`几点关门`、`几点开门`、`营业时间`。
   - 打车地址问题：`打车`、`下车`、`入口`、`导航到哪里`。

3. Proxy 在没有现成 PlaceFact 时先调用 `resolve_places_batch` 解析地点。

4. 前端无需新增 UI，直接在 assistant message 中展示简洁事实回答。

### 本阶段不实现

- 不接真实高德 detail API。
- 不做多候选地点选择交互。
- 不做地图跳转深链。
- 不把 PlaceFactDetail 做成新的 canonical artifact 类型。

## Mock 数据

### 八大关

- 标准名：八大关风景区
- 来源：`amap_mock`
- 营业时间：`全天开放（街区），具体场馆以现场公告为准`
- 字段置信度：`medium`

### 崂山

- 标准名：崂山风景区
- 打车地址：`崂山风景区大河东客服中心入口`
- 入口提示：景区入口较多，带老人孩子建议出发前确认入口。
- 导航 POI：`amap_laoshan_dahadong`

## 数据流

### 营业时间

```text
用户：八大关几点关门？
-> Proxy detect place_detail
-> resolve_places_batch("八大关")
-> get_place_detail(fields=["openingHoursToday", "rating", "businessArea"])
-> assistant_message_delta 返回：
   八大关风景区：全天开放（街区），具体场馆以现场公告为准。
   来源：amap_mock，置信度：medium。
```

### 打车地址

```text
用户：我现在要打车去崂山，打到哪里？
-> Proxy detect taxi_address
-> resolve_places_batch("崂山")
-> suggest_taxi_address(context={ travelers:["elder","child"], preferredMode:"taxi" })
-> assistant_message_delta 返回：
   建议打到：崂山风景区大河东客服中心入口。
   入口较多，出发前确认具体入口。
```

## 验收标准

1. 用户问“八大关几点关门？”：
   - 触发 `resolve_places_batch`。
   - 触发 `get_place_detail`。
   - 回答包含“全天开放”。
   - 回答包含“来源：amap_mock”。
   - 回答包含“置信度：medium”。

2. 用户问“我现在要打车去崂山，打到哪里？”：
   - 触发 `resolve_places_batch`。
   - 触发 `suggest_taxi_address`。
   - 回答包含“大河东客服中心入口”。
   - 回答包含“入口较多”。
   - 回答包含“高德导航 POI”。

3. 字段缺失时回答“当前未获取到可靠营业时间”，不编造。

4. 全量验证通过：
   - `pnpm typecheck`
   - `pnpm test`
   - `pnpm test:e2e`
   - `pnpm build`
