# Phase 1.5 AI 原生旅行工作区重构设计

## 状态

2026-05-24 产品讨论后确认方向。本文替换上一版英文设计，以中文记录 Phase 1.5 的目标、交互模型、状态模型和验收标准。

## 背景

Phase 1 已经用 Mock Provider 跑通 Byway 的核心闭环：

```text
Dream -> Plan -> PlaceFact -> Travel Replan -> Review/Memory
```

当前 Web UI 更像调试台：一侧聊天，一侧把所有 artifact 按类型排序并持续堆叠。这个形态能验证工具链，但不符合用户对 AI-native 产品的预期：

- 旧 artifact 一直占据空间，当前要处理什么不够清楚。
- 聊天、artifact、确认动作的归属混在一起。
- 用户进入旅行后，想看到当前权威计划，却需要在历史 artifact 或聊天上下文里找。
- 旅行计划、今日状态这类“当前旅行状态”和目的地候选、重排方案这类“临时工作对象”没有区分。

讨论后形成新的核心原则：

> Artifact 可以是历史，Trip Workspace 必须是当前。

也就是说，Byway 不是纯 ChatGPT Canvas / Claude Artifacts 式的“聊天 + 临时生成物”产品。Byway 的右侧工作区应该常驻展示当前旅行的权威状态，同时允许从聊天里打开具体 artifact 详情。

## 产品参考

- ChatGPT Canvas：当内容超出普通聊天时，打开并排工作区，用户和 AI 在同一对象上协作。参考：https://openai.com/index/introducing-canvas/
- Claude Artifacts：较大的独立输出进入右侧专用窗口，便于查看、引用和迭代。参考：https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them
- OpenUI Artifacts：聊天中展示 compact preview，点击后在 side panel 展开，同一时间只有一个活动详情。参考：https://www.openui.com/docs/chat/artifacts
- Microsoft Copilot Pages：从聊天回答中提取可持续编辑的页面，形成 side-by-side workspace。参考：https://techcommunity.microsoft.com/blog/microsoft365copilotblog/announcing-copilot-pages-for-multiplayer-collaboration/4242701
- Agent UI Patterns：Agent 产品需要清晰展示进度、状态、确认和人类接管点。参考：https://brainy.ink/paper/ai-agent-ui-design-patterns

这些参考能借鉴“聊天 + 工作对象”的模式，但 Byway 需要额外保留“当前旅行驾驶舱”。

## 目标

1. 左侧保持主聊天：用户始终通过自然语言推进旅行。
2. 聊天消息内显示 compact artifact preview：候选目的地、计划已生成、今日重排方案等。
3. 右侧变成常驻 `Trip Workspace`，不是 artifact 历史流。
4. `Trip Workspace` 默认展示当前旅行权威状态，例如当前计划或今日状态。
5. 点击聊天里的 preview 后，右侧进入 `artifact_detail` 详情模式。
6. 高影响动作的确认控件出现在相关 workspace 上下文内。
7. 移动端保持聊天优先，用底部入口或 bottom sheet 打开旅行工作区。
8. 不接真实 Hermes / 高德 / LLM；Phase 1.5 只重构 Web shell 和前端状态模型。

## 非目标

- 不做真实 Provider 集成。
- 不做账号体系。
- 不做完整旅行管理后台。
- 不做拖拽式日程编辑器。
- 不做跨会话 artifact 历史浏览器。
- 不改 MCP 工具契约，除非实现中发现当前前端无法表达必要状态。

## 信息架构

### 桌面端

```text
┌───────────────────────────────────────────────────────────────┐
│ Trip Header：Byway / 当前旅行 / 当前阶段 / 快速状态             │
├───────────────────────────────┬───────────────────────────────┤
│ Chat Main                     │ Trip Workspace                │
│                               │                               │
│ Assistant / User messages     │ [全程计划] [今天] [待确认] [详情]│
│ Inline artifact previews      │                               │
│ Tool progress summary         │ Current Plan / Today / Detail │
│ Composer                      │ Workspace actions             │
└───────────────────────────────┴───────────────────────────────┘
```

左侧是主栏，右侧是旅行工作区。右侧不再无限堆所有 artifact。

### 移动端

```text
┌──────────────────────────┐
│ Trip Header              │
├──────────────────────────┤
│ Chat Main                │
│ Inline artifact previews │
│ Composer                 │
├──────────────────────────┤
│ 底部固定入口：计划 / 今天   │
│ 点开后为 bottom sheet      │
└──────────────────────────┘
```

移动端不做常驻双栏。默认聊天优先，旅行工作区通过底部入口打开。

## 右侧 Trip Workspace 模式

`Trip Workspace` 有四种模式：

```ts
type WorkspaceMode =
  | "current_plan"
  | "today"
  | "pending"
  | "artifact_detail";
```

### `current_plan`

默认展示当前权威计划，即 `Trip.currentPlanId` 对应的最新 `PlanArtifact`。

用户进入已有旅行，或者 Plan Mode 生成计划后，右侧应优先展示这个视图。

内容：

- 旅行标题。
- 天数全图。
- 每天主题、关键节点、meal / hotel anchor。
- 假设和缺失信息，例如第一天下午到、第四天下午走。
- 是否为当前版本。

### `today`

Travel Mode 默认视图。展示当前日状态。

内容：

- 当前节点。
- 下一个节点。
- 餐食锚点。
- 酒店锚点。
- 快捷反馈入口，例如“老人累了”“晚了 40 分钟”“想回酒店”。

### `pending`

展示当前待确认动作。

内容：

- 待确认类型，例如 `apply_replan`、`accept_memory`。
- 影响范围。
- 当前选择。
- 确认 / 拒绝操作。

如果存在待确认动作，workspace header 需要有明显状态提示。

### `artifact_detail`

点击聊天里的 preview 后进入详情。

适用于：

- DestinationShortlistArtifact
- PlaceFactReviewArtifact
- ReplanOptionsArtifact
- DayLogArtifact
- TripLogArtifact
- MemoryCandidateArtifact
- 历史版本 PlanArtifact

关闭详情后回到最近的常驻模式：

- 已有旅行计划时回到 `current_plan`。
- 旅行中回到 `today`。
- 没有计划时回到空状态或 Trip Brief。

## Artifact 分类

### 常驻旅行状态

这些 artifact 代表当前旅行权威状态，应该常驻或一键可见：

| 状态 | 来源 | 默认位置 |
| --- | --- | --- |
| 当前计划 | 最新 PlanArtifact | `current_plan` |
| 今日状态 | TodayArtifact | `today` |
| 待确认动作 | PendingConfirmation | `pending` 或相关 detail 内 |
| 旅行需求摘要 | TripBrief | 无计划时的 workspace 空状态 |

### 聊天工作对象

这些 artifact 从聊天里出现 preview，点击后打开右侧详情：

| Artifact | 聊天 Preview | Workspace Detail |
| --- | --- | --- |
| DestinationShortlistArtifact | 目的地候选摘要、Top 3 名称 | 候选对比、风险、选择目的地 |
| PlaceFactReviewArtifact | 需要确认地点数量 | 低置信地点选择 |
| ReplanOptionsArtifact | 3 个重排方案摘要 | 选择取消 / 顺延 / 重排，并应用 |
| DayLogArtifact | 今日复盘摘要 | 完整 DayLog |
| TripLogArtifact | 全程总结摘要 | 完整 TripLog |
| MemoryCandidateArtifact | 记忆候选数量 | 接受 / 拒绝家庭记忆 |
| 历史 PlanArtifact | 计划版本摘要 | 历史版本详情，标明非当前 |

## 状态模型

前端不再使用 `artifacts[]` 直接排序渲染。建议状态如下：

```ts
type WorkspaceArtifact = {
  artifactType: string;
  artifactId: string;
  version: number;
  content: Record<string, unknown>;
  createdFromMessageId: string;
};

type ChatItem = {
  id: string;
  role: "assistant" | "user";
  text: string;
  artifactIds: string[];
};

type WorkspaceMode =
  | "current_plan"
  | "today"
  | "pending"
  | "artifact_detail";

type WorkspaceState = {
  tripId?: string;
  messages: ChatItem[];
  artifactsById: Record<string, WorkspaceArtifact>;
  activeArtifactId?: string;
  workspaceMode: WorkspaceMode;
  currentPlanArtifactId?: string;
  todayArtifactId?: string;
  activeConfirmation?: ConfirmationEvent;
  selectedReplanOptionId?: string;
  isStreaming: boolean;
};
```

状态规则：

- `artifact_updated` 存入 `artifactsById`。
- `artifact_updated` 同时把 artifact id 挂到当前 assistant message 的 `artifactIds`。
- 新 PlanArtifact 到达时，更新 `currentPlanArtifactId`，并切到 `current_plan`。
- 新 TodayArtifact 到达时，更新 `todayArtifactId`。
- 新 ReplanOptionsArtifact 到达时，切到 `artifact_detail`，因为用户需要马上选择方案。
- `confirmation_required` 设置 `activeConfirmation`。
- 如果 confirmation 关联 `apply_replan`，workspace 保持在对应 ReplanOptions 详情。
- 用户点聊天 preview 时，设置 `activeArtifactId` 并切到 `artifact_detail`。
- 用户关掉详情时，根据当前阶段回到 `current_plan` 或 `today`。

## 关键交互

### Dream Mode

1. 用户说“不知道去哪”。
2. 聊天中出现 DestinationShortlist preview。
3. 右侧打开候选详情。
4. 用户在右侧选择青岛。
5. Chat 继续推进 Plan Mode。
6. 计划生成后，右侧切到 `current_plan`。

### Plan Mode

1. 用户直接要求规划青岛 4 天。
2. 聊天中出现 Plan preview。
3. 右侧默认展示当前计划全图。
4. 用户可继续聊天调整计划。

### Travel Mode

1. 用户说“老人累了，我们晚了 40 分钟”。
2. TodayArtifact 更新。
3. ReplanOptions preview 出现在聊天中。
4. 右侧打开 ReplanOptions 详情。
5. 用户选择 `opt_postpone`。
6. 待确认区显示“选中方案：opt_postpone”。
7. 用户应用后，当前计划版本更新。
8. 用户可以一键回到“全程计划”查看更新后的整体影响。

### Review Mode

1. 用户要求复盘。
2. 聊天中出现 DayLog / TripLog / Memory preview。
3. 右侧可打开 Memory detail。
4. 用户接受 Family Memory。
5. 右侧回到当前计划或旅行总结状态。

## 组件设计

建议拆分当前 `apps/web/src/app/page.tsx`：

- `apps/web/src/app/page.tsx`：顶层编排、API 调用、SSE 处理。
- `apps/web/src/app/workspaceState.ts`：状态类型和纯 helper。
- `apps/web/src/app/components/ChatThread.tsx`：聊天消息、preview、composer 区域。
- `apps/web/src/app/components/ArtifactPreviewCard.tsx`：聊天内 compact preview。
- `apps/web/src/app/components/TripWorkspace.tsx`：右侧常驻旅行工作区。
- `apps/web/src/app/components/workspaceSurfaces.tsx`：CurrentPlan / Today / Pending / Detail surface。
- `apps/web/src/app/components/artifactRenderers.tsx`：具体 artifact detail 渲染器。

## 验收标准

1. Dream Mode 产生聊天内 DestinationShortlist preview。
2. 点击 DestinationShortlist preview 后，右侧进入目的地详情。
3. 选择青岛后进入 Plan Mode。
4. PlanArtifact 生成后，右侧默认展示当前计划全图。
5. 旧 artifact 不再作为永久列表堆在右侧。
6. Travel Mode 中 ReplanOptions 可在右侧详情选择 `opt_postpone`。
7. 应用重排时，POST 的是当前选中方案，而不是永远使用推荐方案。
8. 用户可以从重排详情一键回到“全程计划”。
9. Review Mode 的 DayLog / TripLog / Memory 仍可从聊天 preview 打开。
10. 移动端不出现永久双栏，使用底部入口打开旅行工作区。
11. `pnpm test`、`pnpm test:api`、`pnpm test:e2e`、`pnpm typecheck`、`pnpm build` 通过。

## 风险与取舍

- 常驻计划会占据右侧注意力，但这是旅行产品的合理中心对象。
- 聊天 preview 太弱会让用户不知道有详情可打开，因此 preview 需要有明显标题、摘要和状态。
- 右侧模式过多会像后台管理页，因此 Phase 1.5 只保留四个 workspace mode。
- 移动端底部 sheet 需要小心遮挡 composer；实现时要保证关闭和返回清晰。

## Phase 1.5 之外

- 后续可以加入跨会话恢复，让用户刷新后仍能看到当前计划。
- 后续可以增加“版本历史”入口，但不在本阶段实现。
- 后续接真实高德 / Hermes 后，Trip Workspace 的 Current Plan 和 Today 仍是稳定承载面。
