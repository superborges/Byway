# AI 原生旅行工作区重构实现计划

> **给 agentic worker 的要求：**执行本计划时必须使用 `superpowers:subagent-driven-development`（推荐）或 `superpowers:executing-plans`。任务使用 checkbox（`- [ ]`）追踪。

**目标：**把 Byway Web 从“聊天 + 永久 artifact 列表”的调试台，重构为“主聊天 + 常驻 Trip Workspace + 聊天内 artifact preview”的 AI 原生旅行工作区。

**架构：**不改 MCP 核心工具，不接真实 Provider。Web 前端引入显式 workspace 状态：聊天消息持有 artifact preview，右侧 Trip Workspace 常驻展示当前计划 / 今日状态 / 待确认 / artifact 详情。高影响操作的确认控件归属到右侧相关工作区。

**技术栈：**Next.js App Router、React 19 client components、TypeScript、Playwright E2E、Vitest、现有 `@byway/shared-schemas` 类型。

---

## 文件结构

- 修改：`apps/web/src/app/page.tsx`
  负责顶层状态、SSE 处理、API 调用、把状态传给子组件。

- 新建：`apps/web/src/app/workspaceState.ts`
  放置 workspace 类型、artifact key、状态更新 helper。

- 新建：`apps/web/src/app/components/ChatThread.tsx`
  渲染聊天消息、artifact preview、quick prompts、composer。

- 新建：`apps/web/src/app/components/ArtifactPreviewCard.tsx`
  渲染聊天内 compact artifact preview。

- 新建：`apps/web/src/app/components/TripWorkspace.tsx`
  渲染右侧常驻旅行工作区，包括模式切换和确认区域。

- 新建：`apps/web/src/app/components/workspaceSurfaces.tsx`
  渲染 Current Plan、Today、Pending、Artifact Detail 四类 surface。

- 新建：`apps/web/src/app/components/artifactRenderers.tsx`
  渲染具体 artifact 详情。

- 修改：`apps/web/src/app/globals.css`
  调整布局为左侧主聊天、右侧 Trip Workspace；移动端改为 bottom sheet。

- 修改：`apps/web/e2e/byway.spec.ts`
  更新验收测试，覆盖常驻计划、preview 打开详情、重排后回到全程计划。

---

## 任务 1：先写失败的 E2E，锁定新交互

**文件：**

- 修改：`apps/web/e2e/byway.spec.ts`

- [ ] **步骤 1：替换 Dream Mode 测试**

把原本直接检查 `artifact-DestinationShortlistArtifact` 的测试，改成检查聊天 preview 和右侧 Trip Workspace：

```ts
test("Dream mode renders destination preview and opens detail in trip workspace", async ({ page }) => {
  await page.goto("/");

  await expect(page.getByRole("heading", { name: "Byway" })).toBeVisible();
  await sendMessage(page, "我想暑假从北京出发，带父母和两个孩子玩 4 天，不知道去哪，别太累。");

  const preview = page.getByTestId("artifact-preview-DestinationShortlistArtifact");
  await expect(preview).toContainText("目的地候选");
  await expect(preview).toContainText("青岛");

  await expect(page.getByTestId("trip-workspace")).toContainText("DestinationShortlistArtifact");
  await expect(page.getByTestId("trip-workspace")).toContainText("青岛");
});
```

- [ ] **步骤 2：新增“选择目的地后右侧常驻当前计划”的测试**

追加：

```ts
test("selecting a destination turns the workspace into the current plan cockpit", async ({ page }) => {
  await page.goto("/");

  await sendMessage(page, "我想暑假从北京出发，带父母和两个孩子玩 4 天，不知道去哪，别太累。");
  await page.getByRole("button", { name: /选择 青岛/ }).click();

  await expect(page.getByTestId("artifact-preview-PlanArtifact")).toContainText("青岛 4 天家庭游");
  await expect(page.getByTestId("trip-workspace")).toContainText("全程计划");
  await expect(page.getByTestId("trip-workspace")).toContainText("青岛 4 天家庭游");
  await expect(page.getByTestId("trip-workspace")).toContainText("第一天下午到");
});
```

- [ ] **步骤 3：扩展 Travel Mode 测试**

在现有 Travel Mode 测试里，重排后增加：

```ts
await expect(page.getByTestId("trip-workspace")).toContainText("今日重排方案");

const postponeOption = page.getByRole("button", { name: /选择方案 整体顺延 40 分钟/ });
await expect(postponeOption).toBeEnabled();
await postponeOption.click();
await expect(postponeOption).toHaveAttribute("aria-pressed", "true");
await expect(page.getByTestId("trip-workspace")).toContainText("选中方案：opt_postpone");

await page.getByRole("button", { name: "全程计划" }).click();
await expect(page.getByTestId("trip-workspace")).toContainText("青岛 4 天家庭游");
```

- [ ] **步骤 4：运行红测**

运行：

```bash
pnpm exec playwright test --grep "Dream mode renders|selecting a destination|Plan mode and Travel mode"
```

预期：

```text
3 failed
```

失败原因应是缺少 `artifact-preview-*`、`trip-workspace` 或 workspace mode 控件。

---

## 任务 2：引入 workspace 状态模型

**文件：**

- 新建：`apps/web/src/app/workspaceState.ts`
- 修改：`apps/web/src/app/page.tsx`

- [ ] **步骤 1：创建状态类型和 helper**

创建 `apps/web/src/app/workspaceState.ts`：

```ts
import type { PendingConfirmation } from "@byway/shared-schemas";

export type ChatRole = "assistant" | "user";

export type WorkspaceMode =
  | "current_plan"
  | "today"
  | "pending"
  | "artifact_detail";

export type ArtifactUpdate = {
  artifactType: string;
  artifactId: string;
  version: number;
  content: Record<string, unknown>;
};

export type WorkspaceArtifact = ArtifactUpdate & {
  createdFromMessageId: string;
};

export type ChatItem = {
  id: string;
  role: ChatRole;
  text: string;
  artifactIds: string[];
};

export type ConfirmationEvent = {
  confirmationId: string;
  confirmationType: PendingConfirmation["type"];
  summary: string;
  payload?: Record<string, unknown>;
  recommendedOptionId?: string;
};

export type WorkspaceState = {
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

export const artifactKey = (artifact: Pick<ArtifactUpdate, "artifactType" | "artifactId">) =>
  `${artifact.artifactType}:${artifact.artifactId}`;

export const attachArtifactToMessage = (
  messages: ChatItem[],
  messageId: string,
  artifactId: string
) => messages.map((message) => {
  if (message.id !== messageId || message.artifactIds.includes(artifactId)) return message;
  return { ...message, artifactIds: [...message.artifactIds, artifactId] };
});

export const storeArtifact = (
  artifactsById: Record<string, WorkspaceArtifact>,
  artifact: ArtifactUpdate,
  messageId: string
) => {
  const key = artifactKey(artifact);
  return {
    ...artifactsById,
    [key]: {
      ...artifact,
      createdFromMessageId: messageId
    }
  };
};

export const defaultWorkspaceModeForTrip = (
  currentPlanArtifactId?: string,
  todayArtifactId?: string
): WorkspaceMode => {
  if (todayArtifactId) return "today";
  if (currentPlanArtifactId) return "current_plan";
  return "artifact_detail";
};
```

- [ ] **步骤 2：替换 `page.tsx` 中的本地类型**

删除 `ChatRole`、`ChatItem`、`ArtifactUpdate`、`ConfirmationEvent` 的本地定义，改为：

```ts
import {
  artifactKey,
  attachArtifactToMessage,
  defaultWorkspaceModeForTrip,
  storeArtifact,
  type ArtifactUpdate,
  type ChatItem,
  type ConfirmationEvent,
  type WorkspaceArtifact,
  type WorkspaceMode
} from "./workspaceState";
```

- [ ] **步骤 3：替换 artifact 数组状态**

把：

```ts
const [artifacts, setArtifacts] = useState<ArtifactUpdate[]>([]);
```

改成：

```ts
const [artifactsById, setArtifactsById] = useState<Record<string, WorkspaceArtifact>>({});
const [activeArtifactId, setActiveArtifactId] = useState<string>();
const [workspaceMode, setWorkspaceMode] = useState<WorkspaceMode>("artifact_detail");
const [currentPlanArtifactId, setCurrentPlanArtifactId] = useState<string>();
const [todayArtifactId, setTodayArtifactId] = useState<string>();
```

- [ ] **步骤 4：让消息支持 `artifactIds`**

初始消息改成：

```ts
const [messages, setMessages] = useState<ChatItem[]>([
  {
    id: "welcome",
    role: "assistant",
    text: "我们从一句话开始。你可以先说不知道去哪，也可以直接给我一个目的地。",
    artifactIds: []
  }
]);
```

`sendMessage` 内新建消息改成：

```ts
const userMessage: ChatItem = {
  id: `user_${Date.now()}`,
  role: "user",
  text: clean,
  artifactIds: []
};

setMessages((current) => [
  ...current,
  userMessage,
  { id: assistantId, role: "assistant", text: "", artifactIds: [] }
]);
```

- [ ] **步骤 5：替换 `upsertArtifact`**

把原来的 `upsertArtifact` 改成：

```ts
const upsertArtifact = (artifact: ArtifactUpdate, assistantId: string) => {
  const key = artifactKey(artifact);
  setArtifactsById((current) => storeArtifact(current, artifact, assistantId));
  setMessages((current) => attachArtifactToMessage(current, assistantId, key));
  setActiveArtifactId(key);

  if (artifact.artifactType === "PlanArtifact") {
    setCurrentPlanArtifactId(key);
    setWorkspaceMode("current_plan");
    return;
  }

  if (artifact.artifactType === "TodayArtifact") {
    setTodayArtifactId(key);
    setWorkspaceMode("today");
    return;
  }

  setWorkspaceMode("artifact_detail");
};
```

- [ ] **步骤 6：更新 SSE 处理**

把：

```ts
upsertArtifact(streamEvent.data as ArtifactUpdate);
```

改成：

```ts
upsertArtifact(streamEvent.data as ArtifactUpdate, assistantId);
```

- [ ] **步骤 7：运行类型检查**

运行：

```bash
pnpm typecheck
```

预期：当前可能因为 JSX 仍引用旧变量而失败。失败只允许来自旧 JSX 引用，不允许来自新类型文件语法错误。

---

## 任务 3：拆出聊天主栏和 artifact preview

**文件：**

- 新建：`apps/web/src/app/components/ArtifactPreviewCard.tsx`
- 新建：`apps/web/src/app/components/ChatThread.tsx`
- 修改：`apps/web/src/app/page.tsx`

- [ ] **步骤 1：创建 `ArtifactPreviewCard.tsx`**

```tsx
import { CalendarDays, MapPinned, RotateCcw, Sparkles } from "lucide-react";
import type {
  DestinationShortlistArtifact,
  PlanArtifact,
  ReplanOptionsArtifact
} from "@byway/shared-schemas";
import type { WorkspaceArtifact } from "../workspaceState";

type Props = {
  artifact: WorkspaceArtifact;
  isActive: boolean;
  onOpen: (artifactId: string) => void;
};

export function ArtifactPreviewCard({ artifact, isActive, onOpen }: Props) {
  const id = `${artifact.artifactType}:${artifact.artifactId}`;

  return (
    <button
      type="button"
      className={isActive ? "artifact-preview active" : "artifact-preview"}
      data-testid={`artifact-preview-${artifact.artifactType}`}
      onClick={() => onOpen(id)}
    >
      <span className="artifact-preview-icon">{previewIcon(artifact.artifactType)}</span>
      <span>
        <strong>{previewTitle(artifact)}</strong>
        <small>{previewSummary(artifact)}</small>
      </span>
    </button>
  );
}

function previewIcon(artifactType: string) {
  if (artifactType.includes("Destination")) return <Sparkles size={17} aria-hidden />;
  if (artifactType.includes("Plan")) return <CalendarDays size={17} aria-hidden />;
  if (artifactType.includes("Replan")) return <RotateCcw size={17} aria-hidden />;
  return <MapPinned size={17} aria-hidden />;
}

function previewTitle(artifact: WorkspaceArtifact) {
  if (artifact.artifactType === "DestinationShortlistArtifact") return "目的地候选";
  if (artifact.artifactType === "PlanArtifact") return (artifact.content as unknown as PlanArtifact).title;
  if (artifact.artifactType === "TodayArtifact") return "今日状态";
  if (artifact.artifactType === "ReplanOptionsArtifact") return "今日重排方案";
  if (artifact.artifactType === "DayLogArtifact") return "今日复盘";
  if (artifact.artifactType === "TripLogArtifact") return "旅行总结";
  if (artifact.artifactType === "MemoryCandidateArtifact") return "家庭记忆候选";
  return artifact.artifactType;
}

function previewSummary(artifact: WorkspaceArtifact) {
  if (artifact.artifactType === "DestinationShortlistArtifact") {
    const content = artifact.content as unknown as DestinationShortlistArtifact;
    return content.candidates.map((candidate) => candidate.destination).join(" / ");
  }
  if (artifact.artifactType === "PlanArtifact") {
    const content = artifact.content as unknown as PlanArtifact;
    return `${content.days.length} 天 · ${content.assumptions.join(" / ")}`;
  }
  if (artifact.artifactType === "ReplanOptionsArtifact") {
    const content = artifact.content as unknown as ReplanOptionsArtifact;
    return `${content.options.length} 个方案 · 推荐 ${content.recommendedOptionId}`;
  }
  return `v${artifact.version}`;
}
```

- [ ] **步骤 2：创建 `ChatThread.tsx`**

```tsx
import type { ChatItem, WorkspaceArtifact } from "../workspaceState";
import { ArtifactPreviewCard } from "./ArtifactPreviewCard";

type Props = {
  messages: ChatItem[];
  artifactsById: Record<string, WorkspaceArtifact>;
  activeArtifactId?: string;
  isStreaming: boolean;
  onOpenArtifact: (artifactId: string) => void;
};

export function ChatThread({
  messages,
  artifactsById,
  activeArtifactId,
  isStreaming,
  onOpenArtifact
}: Props) {
  return (
    <div className="thread">
      {messages.map((message) => (
        <article key={message.id} className={`bubble ${message.role}`}>
          <span>{message.text || (message.role === "assistant" && isStreaming ? "..." : "")}</span>
          {message.artifactIds.length > 0 ? (
            <div className="message-artifacts">
              {message.artifactIds.map((artifactId) => {
                const artifact = artifactsById[artifactId];
                if (!artifact) return null;
                return (
                  <ArtifactPreviewCard
                    key={artifactId}
                    artifact={artifact}
                    isActive={activeArtifactId === artifactId}
                    onOpen={onOpenArtifact}
                  />
                );
              })}
            </div>
          ) : null}
        </article>
      ))}
    </div>
  );
}
```

- [ ] **步骤 3：`page.tsx` 使用 `ChatThread`**

导入：

```ts
import { ChatThread } from "./components/ChatThread";
```

把原来的 `.thread` JSX 替换为：

```tsx
<ChatThread
  messages={messages}
  artifactsById={artifactsById}
  activeArtifactId={activeArtifactId}
  isStreaming={isStreaming}
  onOpenArtifact={(artifactId) => {
    setActiveArtifactId(artifactId);
    setWorkspaceMode("artifact_detail");
  }}
/>
```

- [ ] **步骤 4：运行 focused E2E**

运行：

```bash
pnpm exec playwright test --grep "Dream mode renders"
```

预期：preview 相关断言通过，右侧 workspace 断言仍可能失败。

---

## 任务 4：实现 Trip Workspace 和常驻计划视图

**文件：**

- 新建：`apps/web/src/app/components/TripWorkspace.tsx`
- 新建：`apps/web/src/app/components/workspaceSurfaces.tsx`
- 新建：`apps/web/src/app/components/artifactRenderers.tsx`
- 修改：`apps/web/src/app/page.tsx`

- [ ] **步骤 1：创建 `artifactRenderers.tsx`**

```tsx
import type {
  DestinationShortlistArtifact,
  PlanArtifact,
  ReplanOptionsArtifact,
  TodayArtifact
} from "@byway/shared-schemas";
import type { WorkspaceArtifact } from "../workspaceState";

type RendererProps = {
  artifact: WorkspaceArtifact;
  isStreaming: boolean;
  selectedReplanOptionId?: string;
  onSelectDestination: (destination: string, days: number) => void;
  onSelectReplanOption: (optionId: string) => void;
};

export function ArtifactDetail(props: RendererProps) {
  const { artifact } = props;
  if (artifact.artifactType === "DestinationShortlistArtifact") return <DestinationDetail {...props} />;
  if (artifact.artifactType === "PlanArtifact") return <PlanDetail artifact={artifact} />;
  if (artifact.artifactType === "TodayArtifact") return <TodayDetail artifact={artifact} />;
  if (artifact.artifactType === "ReplanOptionsArtifact") return <ReplanDetail {...props} />;
  if (artifact.artifactType === "MemoryCandidateArtifact") return <MemoryDetail artifact={artifact} />;
  return <pre className="artifact-json">{JSON.stringify(artifact.content, null, 2)}</pre>;
}

export function PlanDetail({ artifact }: { artifact: WorkspaceArtifact }) {
  const content = artifact.content as unknown as PlanArtifact;
  return (
    <div className="artifact-body">
      <h3>{content.title}</h3>
      <div className="assumptions">
        {content.assumptions.map((assumption) => <span key={assumption}>{assumption}</span>)}
      </div>
      <div className="day-list">
        {content.days.map((day) => (
          <section key={day.id} className="day-row">
            <h4>Day {day.dayIndex} · {day.title}</h4>
            <p>{day.summary}</p>
            <ul>
              {day.nodes.map((node) => <li key={node.id}>{node.title}</li>)}
            </ul>
          </section>
        ))}
      </div>
    </div>
  );
}

function DestinationDetail({ artifact, isStreaming, onSelectDestination }: RendererProps) {
  const content = artifact.content as unknown as DestinationShortlistArtifact;
  return (
    <div className="artifact-body">
      <p>{content.summary}</p>
      <div className="candidate-list">
        {content.candidates.map((candidate) => (
          <button
            key={candidate.id}
            type="button"
            className="candidate-row"
            aria-label={`选择 ${candidate.destination}`}
            disabled={isStreaming}
            onClick={() => onSelectDestination(candidate.destination, candidate.recommendedDays.ideal)}
          >
            <div>
              <h3>{candidate.destination}</h3>
              <p>{candidate.whyRecommended}</p>
            </div>
            <strong>{candidate.score}</strong>
          </button>
        ))}
      </div>
    </div>
  );
}

function TodayDetail({ artifact }: { artifact: WorkspaceArtifact }) {
  const content = artifact.content as unknown as TodayArtifact;
  return (
    <div className="artifact-body today-grid">
      <section><span>当前</span><strong>{content.currentNode?.title ?? "待开始"}</strong></section>
      <section><span>下一个</span><strong>{content.nextNode?.title ?? "休息"}</strong></section>
      <section><span>晚餐</span><strong>{content.mealAnchors[0]?.title ?? "就近解决"}</strong></section>
    </div>
  );
}

function ReplanDetail({
  artifact,
  isStreaming,
  selectedReplanOptionId,
  onSelectReplanOption
}: RendererProps) {
  const content = artifact.content as unknown as ReplanOptionsArtifact;
  return (
    <div className="artifact-body option-list">
      {content.options.map((option) => {
        const isRecommended = option.id === content.recommendedOptionId;
        const isSelected = option.id === (selectedReplanOptionId ?? content.recommendedOptionId);
        return (
          <button
            key={option.id}
            type="button"
            className={`${isRecommended ? "recommended " : ""}${isSelected ? "selected " : ""}option-row`}
            aria-label={`选择方案 ${option.summary}`}
            aria-pressed={isSelected}
            disabled={isStreaming}
            onClick={() => onSelectReplanOption(option.id)}
          >
            <div>
              <h3>{option.summary}</h3>
              <p>{option.reason}</p>
            </div>
            <span>{option.type}</span>
          </button>
        );
      })}
    </div>
  );
}

function MemoryDetail({ artifact }: { artifact: WorkspaceArtifact }) {
  const content = artifact.content as { candidates?: { id: string; rule: string; effect: string }[] };
  return (
    <div className="artifact-body">
      {(content.candidates ?? []).map((candidate) => (
        <section key={candidate.id} className="day-row">
          <h3>{candidate.rule}</h3>
          <p>{candidate.effect}</p>
        </section>
      ))}
    </div>
  );
}
```

- [ ] **步骤 2：创建 `workspaceSurfaces.tsx`**

```tsx
import type { ConfirmationEvent, WorkspaceArtifact } from "../workspaceState";
import { ArtifactDetail, PlanDetail } from "./artifactRenderers";

type Props = {
  mode: "current_plan" | "today" | "pending" | "artifact_detail";
  activeArtifact?: WorkspaceArtifact;
  currentPlan?: WorkspaceArtifact;
  today?: WorkspaceArtifact;
  confirmation?: ConfirmationEvent;
  isStreaming: boolean;
  selectedReplanOptionId?: string;
  onSelectDestination: (destination: string, days: number) => void;
  onSelectReplanOption: (optionId: string) => void;
};

export function WorkspaceSurface(props: Props) {
  if (props.mode === "current_plan") {
    if (!props.currentPlan) return <EmptyWorkspace />;
    return <PlanDetail artifact={props.currentPlan} />;
  }

  if (props.mode === "today") {
    if (!props.today) return <EmptyWorkspace />;
    return (
      <ArtifactDetail
        artifact={props.today}
        isStreaming={props.isStreaming}
        selectedReplanOptionId={props.selectedReplanOptionId}
        onSelectDestination={props.onSelectDestination}
        onSelectReplanOption={props.onSelectReplanOption}
      />
    );
  }

  if (props.mode === "artifact_detail" && props.activeArtifact) {
    return (
      <ArtifactDetail
        artifact={props.activeArtifact}
        isStreaming={props.isStreaming}
        selectedReplanOptionId={props.selectedReplanOptionId}
        onSelectDestination={props.onSelectDestination}
        onSelectReplanOption={props.onSelectReplanOption}
      />
    );
  }

  return <EmptyWorkspace />;
}

function EmptyWorkspace() {
  return (
    <div className="empty-artifact">
      <img src="/byway-map.svg" alt="" />
      <p>当前计划、今日状态和打开的详情会在这里展示。</p>
    </div>
  );
}
```

- [ ] **步骤 3：创建 `TripWorkspace.tsx`**

```tsx
import { Check } from "lucide-react";
import type { ConfirmationEvent, WorkspaceArtifact, WorkspaceMode } from "../workspaceState";
import { WorkspaceSurface } from "./workspaceSurfaces";

type Props = {
  mode: WorkspaceMode;
  activeArtifact?: WorkspaceArtifact;
  currentPlan?: WorkspaceArtifact;
  today?: WorkspaceArtifact;
  confirmation?: ConfirmationEvent;
  isStreaming: boolean;
  selectedReplanOptionId?: string;
  onModeChange: (mode: WorkspaceMode) => void;
  onApplyConfirmation: () => void;
  onSelectDestination: (destination: string, days: number) => void;
  onSelectReplanOption: (optionId: string) => void;
};

export function TripWorkspace({
  mode,
  activeArtifact,
  currentPlan,
  today,
  confirmation,
  isStreaming,
  selectedReplanOptionId,
  onModeChange,
  onApplyConfirmation,
  onSelectDestination,
  onSelectReplanOption
}: Props) {
  return (
    <aside className="trip-workspace" data-testid="trip-workspace" aria-live="polite">
      <div className="workspace-tabs" role="tablist" aria-label="旅行工作区">
        <button type="button" className={mode === "current_plan" ? "active" : ""} onClick={() => onModeChange("current_plan")}>
          全程计划
        </button>
        <button type="button" className={mode === "today" ? "active" : ""} onClick={() => onModeChange("today")}>
          今天
        </button>
        <button type="button" className={mode === "pending" ? "active" : ""} onClick={() => onModeChange("pending")}>
          待确认
        </button>
        <button type="button" className={mode === "artifact_detail" ? "active" : ""} onClick={() => onModeChange("artifact_detail")}>
          详情
        </button>
      </div>

      <WorkspaceSurface
        mode={mode}
        activeArtifact={activeArtifact}
        currentPlan={currentPlan}
        today={today}
        confirmation={confirmation}
        isStreaming={isStreaming}
        selectedReplanOptionId={selectedReplanOptionId}
        onSelectDestination={onSelectDestination}
        onSelectReplanOption={onSelectReplanOption}
      />

      {confirmation ? (
        <div className="panel-confirmation" data-testid="confirmation-bar">
          <div>
            <p>{confirmation.summary}</p>
            <span>
              {confirmation.confirmationType === "apply_replan"
                ? `选中方案：${selectedReplanOptionId ?? confirmation.recommendedOptionId ?? "opt_cancel"}`
                : "等待确认"}
            </span>
          </div>
          <button type="button" className="icon-command primary" onClick={onApplyConfirmation}>
            <Check size={17} aria-hidden />
            <span>应用</span>
          </button>
        </div>
      ) : null}
    </aside>
  );
}
```

- [ ] **步骤 4：`page.tsx` 接入 `TripWorkspace`**

导入：

```ts
import { TripWorkspace } from "./components/TripWorkspace";
```

计算当前对象：

```ts
const activeArtifact = activeArtifactId ? artifactsById[activeArtifactId] : undefined;
const currentPlanArtifact = currentPlanArtifactId ? artifactsById[currentPlanArtifactId] : undefined;
const todayArtifact = todayArtifactId ? artifactsById[todayArtifactId] : undefined;
```

把原有 `<aside className="artifact-panel">...</aside>` 替换为：

```tsx
<TripWorkspace
  mode={workspaceMode}
  activeArtifact={activeArtifact}
  currentPlan={currentPlanArtifact}
  today={todayArtifact}
  confirmation={confirmation}
  isStreaming={isStreaming}
  selectedReplanOptionId={selectedReplanOptionId}
  onModeChange={setWorkspaceMode}
  onApplyConfirmation={acceptConfirmation}
  onSelectDestination={selectDestination}
  onSelectReplanOption={selectReplanOption}
/>
```

- [ ] **步骤 5：删除 `page.tsx` 里的旧 artifact 渲染函数**

删除：

```ts
ArtifactCard
ArtifactIcon
ArtifactContent
```

这些逻辑已经移动到新组件。

- [ ] **步骤 6：运行 focused E2E**

运行：

```bash
pnpm exec playwright test --grep "Dream mode renders|selecting a destination"
```

预期：

```text
2 passed
```

---

## 任务 5：把确认动作归属到 Trip Workspace

**文件：**

- 修改：`apps/web/src/app/page.tsx`
- 修改：`apps/web/e2e/byway.spec.ts`

- [ ] **步骤 1：删除聊天区的确认条**

从 `page.tsx` 删除旧的：

```tsx
{confirmation ? (
  <div className="confirmation-bar" data-testid="confirmation-bar">
    ...
  </div>
) : null}
```

确认条只在 `TripWorkspace` 内渲染。

- [ ] **步骤 2：`confirmation_required` 切到待确认上下文**

在 `handleStreamEvent` 的 `confirmation_required` 分支里：

```ts
if (streamEvent.event === "confirmation_required") {
  const nextConfirmation = streamEvent.data as ConfirmationEvent;
  setConfirmation(nextConfirmation);
  setSelectedReplanOptionId(
    nextConfirmation.confirmationType === "apply_replan" ? nextConfirmation.recommendedOptionId : undefined
  );
  if (nextConfirmation.confirmationType !== "apply_replan") {
    setWorkspaceMode("pending");
  }
}
```

说明：`apply_replan` 的详情在 ReplanOptionsArtifact 到达时已经打开，因此不强制切走。

- [ ] **步骤 3：应用确认后回到当前计划**

在 `acceptConfirmation` 成功后：

```ts
setConfirmation(undefined);
setSelectedReplanOptionId(undefined);
setWorkspaceMode(defaultWorkspaceModeForTrip(currentPlanArtifactId, todayArtifactId));
```

- [ ] **步骤 4：运行 Travel 测试**

运行：

```bash
pnpm exec playwright test --grep "Plan mode and Travel mode"
```

预期：

```text
1 passed
```

---

## 任务 6：改造桌面和移动布局 CSS

**文件：**

- 修改：`apps/web/src/app/globals.css`

- [ ] **步骤 1：把 workspace 改为左聊天右工作区**

替换相关布局：

```css
.workspace {
  flex: 1;
  display: grid;
  grid-template-columns: minmax(440px, 0.95fr) minmax(420px, 1.05fr);
  min-height: 0;
  overflow: hidden;
}

.chat-panel {
  min-width: 0;
  min-height: 0;
  display: grid;
  grid-template-rows: 1fr auto auto;
  background: var(--surface);
  border-right: 1px solid var(--line);
}

.trip-workspace {
  min-width: 0;
  min-height: 0;
  overflow-y: auto;
  background: var(--paper);
  padding: 18px;
}
```

- [ ] **步骤 2：添加 preview 样式**

```css
.message-artifacts {
  display: grid;
  gap: 8px;
  margin-top: 10px;
}

.artifact-preview {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 10px;
  border: 1px solid var(--line);
  border-radius: 8px;
  background: var(--surface);
  color: var(--ink);
  padding: 10px;
  text-align: left;
}

.artifact-preview.active,
.artifact-preview:hover,
.artifact-preview:focus-visible {
  border-color: rgba(21, 122, 110, 0.55);
  box-shadow: 0 8px 22px rgba(21, 122, 110, 0.1);
  outline: none;
}

.artifact-preview-icon {
  color: var(--coral);
  flex: 0 0 auto;
}

.artifact-preview strong,
.artifact-preview small {
  display: block;
}

.artifact-preview small {
  color: var(--muted);
  margin-top: 3px;
  line-height: 1.35;
}
```

- [ ] **步骤 3：添加 Trip Workspace tab 样式**

```css
.workspace-tabs {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 8px;
  margin-bottom: 12px;
}

.workspace-tabs button {
  min-height: 36px;
  border: 1px solid var(--line);
  border-radius: 8px;
  background: var(--surface);
  color: var(--ink);
  font-weight: 800;
}

.workspace-tabs button.active {
  border-color: rgba(21, 122, 110, 0.55);
  background: var(--teal-soft);
  color: var(--teal);
}
```

- [ ] **步骤 4：添加 workspace 内确认样式**

```css
.panel-confirmation {
  position: sticky;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  border: 1px solid #eadba9;
  border-radius: 8px;
  padding: 12px;
  margin-top: 14px;
  background: var(--amber-soft);
}
```

- [ ] **步骤 5：移动端改为 bottom sheet**

```css
@media (max-width: 860px) {
  .workspace {
    grid-template-columns: 1fr;
    grid-template-rows: minmax(0, 1fr);
  }

  .chat-panel {
    border-right: 0;
  }

  .trip-workspace {
    position: fixed;
    left: 0;
    right: 0;
    bottom: 0;
    z-index: 8;
    max-height: 72vh;
    border-top: 1px solid var(--line);
    box-shadow: 0 -18px 48px rgba(31, 40, 37, 0.16);
  }
}
```

- [ ] **步骤 6：运行全量 E2E**

运行：

```bash
pnpm test:e2e
```

预期：

```text
4 passed
```

---

## 任务 7：Review / Memory 保持可用

**文件：**

- 修改：`apps/web/e2e/byway.spec.ts`
- 修改：`apps/web/src/app/components/ArtifactPreviewCard.tsx`
- 修改：`apps/web/src/app/components/artifactRenderers.tsx`

- [ ] **步骤 1：更新 Review 测试**

把 Review 测试改成 preview + workspace detail：

```ts
test("Review mode surfaces logs, memory candidates, and memory confirmation", async ({ page }) => {
  await page.goto("/");

  await sendMessage(page, "帮我规划青岛 4 天，带父母和两个孩子，节奏轻松一点。");
  await expect(page.getByTestId("trip-workspace")).toContainText("青岛 4 天家庭游");

  await sendMessage(page, "今天结束了，帮我生成复盘和家庭记忆。");
  await expect(page.getByTestId("artifact-preview-DayLogArtifact")).toContainText("今日复盘");
  await expect(page.getByTestId("artifact-preview-TripLogArtifact")).toContainText("旅行总结");
  await expect(page.getByTestId("artifact-preview-MemoryCandidateArtifact")).toContainText("家庭记忆候选");

  await page.getByTestId("artifact-preview-MemoryCandidateArtifact").click();
  await expect(page.getByTestId("trip-workspace")).toContainText("下午 3 点后");
  await expect(page.getByTestId("confirmation-bar")).toContainText("接受 Family Memory");
});
```

- [ ] **步骤 2：确保 Memory detail 可读**

确认 `artifactRenderers.tsx` 中 `MemoryDetail` 渲染 `rule` 和 `effect`，不要只显示 JSON。

- [ ] **步骤 3：运行 Review 测试**

运行：

```bash
pnpm exec playwright test --grep "Review mode"
```

预期：

```text
1 passed
```

---

## 任务 8：完整验证

**文件：**

- 不新增源码文件，除非验证失败需要修复。

- [ ] **步骤 1：运行单元测试**

```bash
pnpm test
```

预期：

```text
14 passed
```

- [ ] **步骤 2：运行 API 集成测试**

```bash
pnpm test:api
```

预期：

```text
5 passed
```

- [ ] **步骤 3：运行 E2E**

```bash
pnpm test:e2e
```

预期：

```text
4 passed
```

- [ ] **步骤 4：运行类型检查**

```bash
pnpm typecheck
```

预期：

```text
exit code 0
```

- [ ] **步骤 5：运行生产构建**

```bash
pnpm build
```

预期：

```text
apps/web build: Done
services/proxy-api build: Done
```

- [ ] **步骤 6：手动浏览器 smoke**

启动：

```bash
pnpm dev:all
```

打开：

```text
http://127.0.0.1:3000/
```

手测路径：

```text
Dream prompt
-> 聊天中出现目的地 preview
-> 右侧打开目的地详情
-> 选择青岛
-> 右侧默认展示全程计划
-> 输入老人累了 / 晚了 40 分钟
-> 右侧打开重排详情
-> 选择 opt_postpone
-> 应用
-> 点击全程计划，能看到当前旅行计划仍常驻可达
```

预期：

```text
右侧不再永久堆所有 artifact。
当前计划是常驻旅行状态。
聊天可以继续输入。
重排选择和确认在 Trip Workspace 内完成。
移动端没有永久双栏。
```

## 自查

- 设计覆盖：常驻 Current Plan、Today、Pending、Artifact Detail、移动端 bottom sheet 都有任务覆盖。
- 占位检查：本文不包含延后填充步骤，所有任务都有明确文件、代码片段、命令和预期结果。
- 类型一致性：`WorkspaceMode`、`WorkspaceArtifact`、`ChatItem`、`ConfirmationEvent` 统一从 `workspaceState.ts` 导出。
