# Phase 3.11 Hermes 注册 Byway MCP 设计

## 目标

让本机 Hermes 能直接连接 Byway MCP Tools。Phase 3.8 已经提供了 Byway stdio MCP server，但直接用 `pnpm --filter @byway/mcp-server mcp` 注册给 Hermes 会有两个问题：

- `pnpm` 默认脚本输出会污染 stdout，MCP stdio 只能输出 JSON-RPC。
- standalone MCP server 需要加载仓库 `.env.local`，否则高德、SQLite 路径等配置不会进入进程。

Phase 3.11 增加一个稳定启动脚本，并把 Hermes 本机 MCP server 注册到该脚本。

## 方案

- 新增 `scripts/run-byway-mcp.sh`：
  - 定位仓库根目录。
  - 读取 `.env.local` 中的本地配置。
  - 设置默认 `BYWAY_SQLITE_PATH=.byway/byway.sqlite`。
  - 用 `pnpm --silent --filter @byway/mcp-server mcp` 启动，保证 stdout 只输出 MCP JSON-RPC。
- `stdioServer.ts` 处理 stdout EPIPE，避免上游客户端断开时抛未捕获异常。
- 使用 `hermes mcp add byway --command bash --args <script>` 注册。
- 使用 `hermes mcp test byway` 验证工具列表。

## 边界

- 这一步只让 Hermes 能看到和调用 Byway tools，不把 `/api/chat` 主流程切到 Hermes。
- 不把高德 key 写入 Hermes 配置；脚本从仓库本地 `.env.local` 读取。
- 如果用户未来迁移仓库路径，需要重新注册 Hermes MCP server。
