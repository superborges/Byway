# Phase 4.0 实现状态记录

日期：2026-05-25

## 当前结论

Phase 4.0 已经从设计进入可运行实现：

```text
Web -> Backend HTTP/SSE -> Agent Runtime -> ToolGateway -> Byway Tool Runtime
Hermes -> Backend HTTP / Backend MCP Adapter -> 同一个 ToolGateway
```

旧 `services/proxy-api` 已从工作区移除；独立 `services/byway-mcp-server` 不再是新功能主线。`services/byway-mcp-server` 目前作为 in-process legacy tool runtime 被 backend ToolGateway 调用，后续逐步迁入 backend 内部 tools/domain。

## 已完成能力

- `services/backend` 包、健康检查、HTTP/SSE。
- `ToolGateway` 统一工具入口。
- `MCP Adapter`，外部 Agent 可通过同一工具入口调用 Byway。
- `Agent Context Package`，保留 trip destination、phase、artifacts、memory。
- Hybrid Agent turn plan：
  - 有 LLM provider 时生成结构化 turn plan。
  - 生产 `/api/chat` 必须配置 DeepSeek LLM；无 LLM 或 LLM 失败时不再静默回退确定性判断。
  - 确定性判断只保留在 `BYWAY_TEST_MODE=1` / `NODE_ENV=test` 的测试路径。
  - 若当前 trip 已有 destination，而用户没有明确换城市，强制沿用当前 destination，避免“杭州聊成青岛”。
- Hybrid Dream：
  - 生产 Dream Mode 的目的地候选由 LLM 根据当前用户需求、TripBrief 和 Memory 生成。
  - LLM turn plan 可直接携带 `dreamShortlist`，减少 Dream Mode 的额外 LLM 往返；若该可选字段格式不合格，会自动降级为专用 Dream 候选生成器。
  - 生产路径不再调用固定目的地候选池；固定候选只保留在显式测试 fixture 中。
  - `score_destination_candidates` 只负责保存 LLM 候选 artifact 和应用已接受家庭记忆，不承担静态推荐。
- Hybrid Guide / Plan drafting：
  - 有 LLM provider 时，Plan Mode 会先生成多视角攻略底稿，并通过 `ingest_user_guide_source` 入库。
  - 生产路径中没有 LLM 攻略底稿且没有用户攻略源时，会明确失败，不再调用 mock `generate_synthetic_guides`。
  - 生成基础 PlanArtifact 后，LLM 会基于 GuideSummary 和基础计划生成家庭可执行计划补丁。
  - 计划补丁仍通过 `update_plan_artifact` 写入，随后继续走 `verify_plan_geo`。
- DeepSeek provider：
  - 使用 OpenAI-compatible `/chat/completions`。
  - 默认模型 `deepseek-v4-pro`，可通过 `DEEPSEEK_MODEL` 覆盖。
  - 支持请求超时和瞬时失败重试，默认不把 API key 写入错误信息。
  - 只从环境变量读取 key，不落库、不写文档。
- Guide ingestion：
  - Plan Mode 在 LLM 可用时生成 LLM 多视角攻略底稿。
  - 测试路径可显式使用 synthetic guide fixture；产品路径不使用 mock 攻略。
  - 聊天附件中的文本 / Markdown 攻略会进入 `ingest_user_guide_source`。
  - 聊天中粘贴公开 HTTP/HTTPS URL 时，会抽取公开页面正文并作为 `user_url` GuideSource；本地 / 内网 URL 和不可访问页面不会被当作已读取攻略。
  - 多源攻略会通过 `merge_guides` 合并为 `GuideSummaryArtifact`，再进入后续 PlaceFact 和 Plan 生成。
- Plan / Plan Edit：
  - LLM turn plan 中出现的新目的地会先写入 TripBrief，再进入攻略、PlaceFact 和 Plan 生成。
  - 非杭州 / 青岛等内置目的地会使用 generic family plan profile，并优先用已解析 PlaceFact 填充活动，避免未知目的地退回青岛模板。
  - Plan Edit 支持 add / replace / remove / soften 语义，仍保留 meal / hotel / rest anchors。
  - Plan Edit 更新计划后会重新读取完整 `PlanArtifact` 并通过 SSE 发送完整 content，避免前端工作区拿到空 content 后崩溃。
- PlaceFact 安全边界：
  - `resolve_places_batch` 对低置信或未解析地点生成 `PlaceFactReviewArtifact`。
  - 同时创建 `confirm_place_fact` PendingConfirmation，并由 Agent Runtime 透传为 `confirmation_required`。
  - 高德 Provider 对相同请求做运行期缓存，避免同一轮或同一服务实例内重复请求相同 POI / 路线。
- Plan quality guard：
  - `verify_plan_geo` 不只返回跨区地理风险，也返回 `qualityWarnings` 和 `overallQualityFit`。
  - 当前校验覆盖满日缺少午晚餐 anchor、缺少休息/缓冲 anchor、家庭日程活动密度过高、抵达/返程日活动过多等风险。
- Local Auth：
  - `BYWAY_AUTH_PIN` 启用后，除 `/health` 和 `/api/auth/*` 外的 API 都需要 Bearer session token。
  - session token 使用本地 HMAC 签名，不把 PIN、Hermes、高德或 LLM key 发给前端。
- Trace：
  - JSONL trace store。
  - `/api/traces` 和 `/api/traces/:traceId`。
  - 记录 `agent.turn_plan.created`、`agent.mode.decided`、`agent.skills.loaded`、LLM started/completed/failed、tool started/completed。
  - `agent.skills.loaded` 会写入 Runtime Skill 文件、version 和内容 hash，方便验收 Agent 本轮策略来源。
- Web 兼容：
  - 兼容旧 `message.text` 请求体。
  - 兼容旧 SSE `assistant_message_delta.delta` 和 `assistant_message_completed.tripId`。
  - `/api/chat` 在长 Agent turn 开始前会立即输出轻量进度 delta，避免 DeepSeek 或高德请求期间前端看起来卡死。
  - 右侧 Trip Workspace 显示当前 Trace 链接。
  - Composer 支持选择文本 / Markdown 攻略文件，先上传到 `/api/uploads`，发送时带 `fileId`。
- Data portability：
  - `GET /api/data/export` 导出本地 JSON snapshot。
  - `POST /api/data/import` 可把 snapshot 导入新的本地库，用于备份恢复或换机迁移。
  - Playwright E2E 通过 `scripts/run-e2e-dev.sh` 启动隔离 backend/web，并显式打开 `BYWAY_TEST_MODE=1` 使用 fixture；产品运行不接受 mock Provider。

## 已覆盖验收链路

- Dream：不知道去哪，只生成 `DestinationShortlistArtifact`，不直接生成计划。
- Plan：直接规划杭州，生成杭州 `PlanArtifact`，不泄漏青岛计划。
- Plan：直接规划成都等非内置目的地时，生成 generic 目的地计划，不泄漏青岛计划。
- Dream -> Plan：同一个 trip 中选择杭州后进入杭州计划。
- Plan Edit：用户自然语言修改第 N 天，读取当前 trip 上下文，解析地点，更新计划版本。
- Plan Edit：添加低置信地点时，前端会收到 `confirm_place_fact` confirmation。
- LLM Plan Edit：LLM 可以补全隐式地点/日期，但 destination 会被当前 trip 上下文约束。
- LLM Guide/Plan：LLM 可生成攻略底稿和计划草案补丁，但地点事实、计划写入和地理核验仍通过工具链。
- Travel：老人累了、晚点等事件生成 `ReplanOptionsArtifact` 和 `apply_replan` confirmation。
- Confirmation：确认后调用 `apply_replan`，不是只把 confirmation 标为 accepted。
- Review/Memory：生成 DayLog、TripLog、Memory candidate；用户确认后才 accepted。
- Place QA：问营业时间/打车点时通过 `resolve_places_batch`、`get_place_detail`、`suggest_taxi_address`，不编造地点事实。

## 本地验收方式

推荐：

```bash
pnpm dev
```

或临时分开启动：

```bash
PORT=4100 BYWAY_BACKEND_DB_PATH=.byway/backend.sqlite BYWAY_TRACE_DIR=.byway/traces pnpm --filter @byway/backend dev
NEXT_PUBLIC_BYWAY_BACKEND_URL=http://127.0.0.1:4100 pnpm --filter @byway/web exec next dev --hostname 127.0.0.1 --port 3001
```

检查 Agent 状态：

```bash
curl http://127.0.0.1:4100/api/agent/status
```

配置 LLM：

```dotenv
BYWAY_LLM_PROVIDER=deepseek
DEEPSEEK_API_KEY=本地私有 key
DEEPSEEK_MODEL=deepseek-v4-pro
DEEPSEEK_REASONING_EFFORT=medium
```

## 仍然保留的边界

- LLM 负责意图理解、攻略判断层底稿和计划草案补丁。
- 所有业务写入仍通过 ToolGateway。
- PlaceFact / RouteFact 仍通过工具和 provider。
- 高影响动作必须通过 PendingConfirmation。
- Trace 要脱敏，不记录真实 key。
