# Phase 3.8 Byway MCP Stdio Server 设计

## 目标

让 `services/byway-mcp-server` 不只是进程内 TypeScript class，而是可以被 Hermes 以 MCP stdio server 方式连接。这样 Hermes 后续可以直接调用 Byway 的 `create_trip`、`generate_plan`、`replan_today` 等工具，而不是只能通过 Proxy mock orchestrator。

## 协议范围

实现最小 MCP JSON-RPC 能力：

- `initialize`
- `notifications/initialized`
- `tools/list`
- `tools/call`

`tools/call` 返回 MCP content：

```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"ok\":true,...}"
    }
  ]
}
```

## 工具暴露

暴露当前已实现的 Byway MCP 工具，包括 Trip、Dream、Guide、Place/Geo、Plan、Travel、Review/Memory。input schema 第一阶段采用宽松 object schema，具体契约仍以 `docs/BYWAY_MCP_TOOL_CONTRACTS.md` 和 shared schemas 为准。

## 边界

- 不在本阶段把 `/api/chat` 切到 Hermes 编排。
- 不引入外部 MCP SDK，先用轻量 JSON-RPC stdio，降低依赖风险。
- 工具内部仍沿用现有确认和 PlaceFact 边界。
