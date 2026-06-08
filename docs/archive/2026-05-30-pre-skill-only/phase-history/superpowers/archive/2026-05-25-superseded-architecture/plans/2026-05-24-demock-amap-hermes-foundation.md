# Phase 3.0 去 Mock 基础 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在不破坏现有 mock 闭环的前提下，为 Byway 接入真实高德 Place/Route Web Service，并把 Proxy 的 Hermes 编排抽到可替换边界后面。

**Architecture:** MCP Server 新增 `PlaceGeoProvider` 抽象，默认 `auto`：有 `AMAP_API_KEY` 使用真实高德，没有 key 继续使用稳定 mock。Proxy API 新增 `HermesRuntimeAdapter` 边界，本阶段仍由 mock orchestrator 承担执行，后续接本地 Hermes 时只替换 adapter。

**Tech Stack:** TypeScript、Zod canonical schema、Vitest、Express SSE、pnpm workspace、Next.js PWA。

---

## 文件结构

- Create: `services/byway-mcp-server/src/providers/placeGeoProvider.ts`
  - 定义 `PlaceGeoProvider` 接口、mock provider、provider factory、`AmapProviderError`。
- Create: `services/byway-mcp-server/src/providers/amapProvider.ts`
  - 封装高德 `/v3/place/text` 和 `/v3/direction/driving`，把响应映射到 `PlaceFact` 和 route estimate。
- Create: `services/byway-mcp-server/src/providers/amapProvider.test.ts`
  - 用 fake fetch 覆盖 POI 映射、驾车路线映射、高德错误降级。
- Modify: `services/byway-mcp-server/src/index.ts`
  - `BywayMcpServer.create(options)` 支持注入 provider。
  - `resolve_places_batch` 和 `estimate_route_matrix` 使用 provider。
  - detail/taxi source 按 provider id 区分 mock 与 web service。
- Create: `services/proxy-api/src/hermesRuntimeAdapter.ts`
  - 定义 `HermesRuntimeAdapter`、`MockHermesRuntimeAdapter`、factory。
- Modify: `services/proxy-api/src/index.ts`
  - `createProxyApp` 支持注入 Hermes adapter，并把 `/api/chat` 当前编排声明为 mock adapter 路径。
- Create: `.env.example`
  - 记录 `AMAP_API_KEY`、`BYWAY_PLACE_PROVIDER`、`HERMES_RUNTIME` 等本地配置。
- Modify: `README.md`
  - 增加 Phase 3 本地真实 Provider 配置说明。

## Task 1: Amap Provider 红灯测试

**Files:**
- Create: `services/byway-mcp-server/src/providers/amapProvider.test.ts`
- Test: `pnpm vitest run services/byway-mcp-server/src/providers/amapProvider.test.ts`

- [ ] **Step 1: 写失败测试**

测试应先引用尚不存在的 `AmapPlaceGeoProvider`，覆盖：

```ts
const provider = new AmapPlaceGeoProvider({
  apiKey: "fake-key",
  baseUrl: "https://amap.test",
  fetchFn: async () => ({
    ok: true,
    json: async () => ({
      status: "1",
      pois: [{
        id: "B0FFFAKE",
        name: "八大关风景区",
        location: "120.355,36.061",
        address: "山东省青岛市市南区武胜关支路",
        cityname: "青岛市",
        adname: "市南区",
        adcode: "370202",
        biz_ext: { rating: "4.6", open_time: "全天开放" },
        photos: [{ url: "https://img.test/badaguan.jpg" }]
      }]
    })
  })
});
```

期望：

```ts
expect(place?.providerPlaceId).toBe("B0FFFAKE");
expect(place?.coordinate).toEqual({ lng: 120.355, lat: 36.061, system: "GCJ02" });
expect(place?.source).toBe("amap_search");
```

- [ ] **Step 2: 跑红灯**

Run: `pnpm vitest run services/byway-mcp-server/src/providers/amapProvider.test.ts`

Expected: FAIL，原因是 `amapProvider` 模块尚不存在。

## Task 2: 最小实现 Amap Provider

**Files:**
- Create: `services/byway-mcp-server/src/providers/amapProvider.ts`
- Create: `services/byway-mcp-server/src/providers/placeGeoProvider.ts`

- [ ] **Step 1: 实现 provider 接口和高德映射**

`AmapPlaceGeoProvider.resolvePlace`：

- 请求 `/v3/place/text`
- 参数：`keywords`、`city`、`extensions=all`、`output=JSON`、`key`
- `status !== "1"` 抛 `AmapProviderError("AMAP_UNAVAILABLE")`
- 第一条 POI 映射为 `PlaceFact`

`AmapPlaceGeoProvider.estimateRoute`：

- 请求 `/v3/direction/driving`
- 参数：`origin=lng,lat`、`destination=lng,lat`、`extensions=base`、`output=JSON`、`key`
- 把 `duration` 秒转成向上取整分钟，把 `distance` 转成整数米。

- [ ] **Step 2: 跑绿灯**

Run: `pnpm vitest run services/byway-mcp-server/src/providers/amapProvider.test.ts`

Expected: PASS。

## Task 3: MCP Server 接入 Provider

**Files:**
- Modify: `services/byway-mcp-server/src/index.ts`
- Modify: `services/byway-mcp-server/src/bywayMcpServer.test.ts`

- [ ] **Step 1: 写失败测试**

在 MCP 测试里新增：

```ts
const mcp = await createBywayMcpServer({
  placeGeoProvider: new AmapPlaceGeoProvider({ apiKey: "fake", fetchFn })
});
```

期望 `resolve_places_batch` 返回 fake 高德 POI，`estimate_route_matrix` 返回 fake 高德路线分钟数。

- [ ] **Step 2: 跑红灯**

Run: `pnpm vitest run services/byway-mcp-server/src/bywayMcpServer.test.ts`

Expected: FAIL，原因是 `createBywayMcpServer` 还不接受 options。

- [ ] **Step 3: 实现注入和 fallback**

`BywayMcpServer` 构造函数接收 `PlaceGeoProvider`。默认用 `createPlaceGeoProviderFromEnv()`：

- `BYWAY_PLACE_PROVIDER=mock`：mock
- `BYWAY_PLACE_PROVIDER=amap`：必须配置 key，否则仍返回 recoverable error
- `BYWAY_PLACE_PROVIDER=auto` 或未配置：有 key 用高德，否则 mock

- [ ] **Step 4: 跑绿灯**

Run: `pnpm vitest run services/byway-mcp-server/src/bywayMcpServer.test.ts`

Expected: PASS。

## Task 4: Hermes Runtime Adapter 边界

**Files:**
- Create: `services/proxy-api/src/hermesRuntimeAdapter.ts`
- Modify: `services/proxy-api/src/index.ts`
- Test: `services/proxy-api/src/proxyApi.test.ts`

- [ ] **Step 1: 写失败测试**

在 Proxy 测试里验证 `createProxyApp({ hermesRuntime: customAdapter })` 能注入 adapter metadata，并且 `/health` 返回当前 runtime mode。

- [ ] **Step 2: 实现最小 adapter**

```ts
export interface HermesRuntimeAdapter {
  readonly runtime: "mock" | "local";
}
```

本阶段不把 `/api/chat` 的编排整体移动出去，只让 Proxy 明确知道当前 runtime；下一阶段再按真实 Hermes 协议拆执行器。

- [ ] **Step 3: 跑绿灯**

Run: `pnpm vitest run services/proxy-api/src/proxyApi.test.ts`

Expected: PASS。

## Task 5: 配置文档和全量验证

**Files:**
- Create: `.env.example`
- Modify: `README.md`

- [ ] **Step 1: 写配置文档**

`.env.example` 必须包含：

```dotenv
BYWAY_PLACE_PROVIDER=auto
AMAP_API_KEY=
AMAP_BASE_URL=https://restapi.amap.com

HERMES_RUNTIME=mock
HERMES_BASE_URL=
HERMES_API_KEY=
```

README 必须说明：

- 不填 `AMAP_API_KEY` 时还是 mock。
- 填 key 后 `BYWAY_PLACE_PROVIDER=auto` 会启用高德。
- Hermes 当前默认 `mock`，本地 Hermes 协议确认后再接 `local`。

- [ ] **Step 2: 全量验证**

Run:

```bash
pnpm typecheck
pnpm test
pnpm test:e2e
pnpm build
```

Expected: 全部 exit 0。构建后如开发服务被停止，重新启动 `pnpm dev:all` 供浏览器验收。
