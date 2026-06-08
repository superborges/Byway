# Phase 3.0 去 Mock 基础：高德 Provider 与 Hermes Adapter 设计

## 背景

当前 Byway 的前端、Proxy API、MCP 工具和 Artifact 闭环已经跑通，但事实数据和 Agent 编排仍以 mock 为主。下一步要进入“可逐步去 mock”的阶段：先把外部能力包成稳定边界，再逐项替换 Provider。

本阶段先做两件事：

1. 高德 Place/Route Provider：用真实高德 Web 服务替换地点解析和路线估算，未配置 Key 时回退 mock。
2. Hermes Runtime Adapter 契约：把 Proxy 中的 mock orchestrator 包到明确接口后面，为接本地 Hermes 环境做准备。

## 设计原则

1. 外部服务不可用时，Byway 不能崩溃，必须返回可恢复错误或回退 mock。
2. 前端 API 契约不变，仍然只消费 Byway Proxy SSE。
3. 高德结果只进入 PlaceFact / RouteFact，LLM/Hermes 不得编造坐标。
4. 去 mock 是渐进式：`AMAP_API_KEY` 打开真实高德；没有配置时现有 E2E 仍可跑。

## 高德 Provider

### 环境变量

- `AMAP_API_KEY`：高德 Web 服务 Key。
- `BYWAY_PLACE_PROVIDER=amap|mock|auto`：默认 `auto`。`auto` 在有 Key 时使用高德，否则 mock。
- `AMAP_BASE_URL`：默认 `https://restapi.amap.com`，测试可替换。

### 真实接口

依据高德官方 Web 服务文档：

- POI 关键词搜索：`GET /v3/place/text`
  - `keywords`
  - `city`
  - `extensions=all`
  - `output=JSON`
  - `key`

- 驾车路径规划：`GET /v3/direction/driving`
  - `origin=lng,lat`
  - `destination=lng,lat`
  - `extensions=base`
  - `output=JSON`
  - `key`

### 降级

- 高德返回 `status !== "1"`：返回 recoverable ToolResult error。
- 没有 Key：继续使用当前 mock provider。
- 缺坐标：路线估算回退现有 mock matrix。

## Hermes Adapter

### 本阶段只做边界

Proxy 里保留现有 mock orchestrator，但把它视为 `HermesRuntimeAdapter` 的一个实现。后续接本地 Hermes 时新增 `LocalHermesRuntimeAdapter`。

### 预留环境变量

- `HERMES_RUNTIME=mock|local`
- `HERMES_BASE_URL`
- `HERMES_API_KEY`

由于本地 Hermes 的 endpoint 形态可能是 OpenAI-compatible、Responses API 或自定义 SSE，本阶段不猜协议，只先固定 Byway 内部 adapter 输入/输出。

## 验收标准

1. 不配置 `AMAP_API_KEY` 时，现有 17 条 E2E 仍通过。
2. 注入 fake 高德 fetch 时，`resolve_places_batch` 会把 POI 响应映射为真实 PlaceFact。
3. 注入 fake 高德 fetch 时，`estimate_route_matrix` 会把路径规划响应映射为 RouteFact-like matrix。
4. 高德错误响应返回 `AMAP_UNAVAILABLE` recoverable error。
5. README 或 `.env.example` 说明如何配置 Key。

## 参考

- 高德 Web服务 API 概述：Web 服务 API 通过 HTTP 提供 JSON/XML 数据，调用前需要申请 Web 服务 Key。
- 高德路径规划 API：路径规划以 HTTP 形式提供步行、公交、驾车、距离测量等接口，`key`、`origin`、`destination` 为核心参数。
