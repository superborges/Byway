# 正式可用 Roadmap 设计

## 目标

在不继续美化 UI 的前提下，把 Byway 从可演示 mock 闭环推进到可正式使用的家庭旅行 Agent。正式可用的定义不是“完全没有 mock”，而是核心事实、状态、确认和失败边界可依赖，且所有 mock 都有明确开关和替换边界。

## 当前状态判断

已经完成：

- Dream / Plan / Travel / Review / Memory 的端到端闭环。
- 共享 schema、MCP 工具契约、Proxy SSE、PWA 工作区。
- 高德 Place/Route Provider 可通过环境变量启用。
- 本地 Hermes MCP messaging bridge 可探测和调用，shadow mirror 默认关闭。

尚未达到正式可用的关键缺口：

- Trip、Artifact、Confirmation、Memory 仍随进程生命周期丢失。
- `/api/trips` 依赖 Proxy 内存集合，重启后无法列出已有旅行。
- Hermes 目前只是 messaging bridge，不是 Byway 旅行编排大脑。
- 上传仍是 mock storage。
- 本地家庭 PIN/session 还未落地。
- Guide ingestion 仍不能读取真实公开网页，只能吸收用户提供内容。

## 阶段拆分

### Phase 3.4：持久化与恢复

把 Byway MCP 的 sql.js 内存库变成文件型 SQLite，Proxy Trip 列表从 MCP 读取。解决刷新、重启、继续旅行的基础问题。

### Phase 3.5：本地家庭 Session / PIN

增加轻量本地会话，保护本机家庭数据。第一阶段只做本地 PIN 和 session token，不做复杂账号体系。

### Phase 3.6：上传落地

把 `/api/uploads` 从 mock fileId 改成本地文件存储，保存 metadata，并接入 GuideSource。

### Phase 3.7：Hermes 事件回流

用 Hermes MCP `events_wait/events_poll` 建立事件桥，把真实消息、审批、回复映射到 Byway SSE 的事件结构。

### Phase 3.8：Hermes 编排适配

在不破坏现有 mock orchestrator 的情况下，引入 Hermes orchestration adapter。Hermes 可以调用 Byway MCP Tools，但高影响动作仍必须通过 PendingConfirmation。

### Phase 3.9：真实攻略来源最小化

支持用户粘贴 URL 时做可访问公开页面的正文抽取；不可访问或平台禁止抓取时明确降级为“只保存链接和用户备注”。

## 边界

- UI 只做必要的功能入口和状态展示，不做视觉美化。
- 不自动把消息发到真实微信/消息平台，必须显式配置 target。
- 不自动预订、支付、绕过平台权限或抓取不可访问评论区。
- 高德失败时返回 recoverable error，不编造 PlaceFact。
