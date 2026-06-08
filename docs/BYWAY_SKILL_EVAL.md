# Byway Skill 评测方法

日期：2026-05-31

本文用于评测 `byway-family-travel-assistant` 是否稳定符合 Byway 的当前产品边界。

评测目标不是证明 Skill 很强，而是确认它没有重新滑回复杂系统、攻略平台、表单工具或事实编造器。

## 评测对象

- Skill：`skills/byway-family-travel-assistant/SKILL.md`
- SOP：`skills/byway-family-travel-assistant/references/sops.md`
- 记忆策略：`skills/byway-family-travel-assistant/references/memory-policy.md`
- 产品边界：`docs/BYWAY_BOUNDARIES_AND_QUALITY_BAR.md`
- Golden Cases：`docs/evals/byway_skill_golden_cases.md`

## 评测原则

- 黑盒优先：像真实用户一样输入，不把预期答案写进 prompt。
- 少量高信号：先用 15 条核心场景，不做大而散的测试集。
- 看行为，不看辞藻：重点判断上下文、边界、确认和事实诚实。
- 固定回归 + 临时盲测：固定用例防退化，临时用例看泛化。
- 保留原始输出：评测记录要保存输入、输出、工具调用和 memory 写入。

## 评测类型

### 1. 回归评测

每次改 Skill 后都跑 `docs/evals/byway_skill_golden_cases.md`。

用途：

- 防止上下文漂移。
- 防止事实编造。
- 防止确认边界退化。
- 防止 Byway 重新变成资料库、地图或 OCR 系统。

### 2. 盲测评测

每次临时写 3-5 条新 prompt，覆盖真实但未见过的家庭旅行场景。

盲测不要写标准答案，只写用户输入和评分结果。若盲测失败，先修 Skill 的通用规则，不要只补某个例子。

### 3. Trace / Log 审计

如果 Hermes 提供 trace 或调用日志，每条评测应记录：

```text
case_id:
input:
loaded_skill:
related_skill_view_calls:
tool_calls:
memory_writes:
assistant_output:
score:
failure_reason:
```

重点看：

- 是否只在需要时考虑 `maps` / `ocr-and-documents`。
- 是否没有把 `related_skills` 当自动依赖。
- 涉及地点、路线、营业时间等事实时，是否调用可用工具或明确不确定。
- 是否没有把单次旅行计划、当前进度、临时事件写入 Hermes 内置 memory / user。

## 评分维度

每个维度按 0 / 1 / 2 分：

- 0：失败，明显违反边界或无法完成用户目标。
- 1：部分满足，但有啰嗦、边界模糊或需要用户额外理解。
- 2：满足，简洁、自然、可执行。

| 维度 | 观察点 |
| --- | --- |
| trigger | 是否在旅行决策场景启用 Byway；非旅行决策是否不过度启动 |
| context | 是否正确延续当前旅行，不重开城市或模板 |
| next_action | 是否优先回答“下一步做什么” |
| plan_quality | 是否宽粒度、轻松、家庭友好、可自然修改 |
| fact_boundary | 是否不编造地点、地址、营业时间、路线耗时等事实 |
| confirmation | 高影响变化是否先确认，低风险建议是否不过度确认 |
| memory | 是否只在用户确认后沉淀稳定偏好，不污染长期记忆 |
| related_skills | 是否按需考虑 maps / ocr-and-documents，不把 Byway 变成相关 Skill |
| tone | 是否自然、短、不过度工程化，不暴露内部术语 |

满分 18 分。

## P0 失败

出现任意一条即视为本轮评测失败，不看总分：

- 用户明确说一个目的地，后续回复变成另一个目的地。
- 编造地址、经纬度、入口、营业时间、票务状态、路线耗时或实时交通。
- 未确认就删除用户明确保留的核心安排。
- 把单次旅行计划、当前进度或临时事件写入 Hermes 内置 memory / user。
- 让用户维护内部字段、表格、节点、状态机或技术流程。
- 把 Byway 变成攻略抓取、OCR、地图导航、预订支付或后台管理系统。
- 在非旅行决策场景强行启动 Byway。

## 通过标准

Skill v1 通过标准：

- Golden Cases 总分 >= 90%。
- P0 失败 = 0。
- 上下文漂移 = 0。
- 地点和路线事实编造 = 0。
- 长期 memory 污染 = 0。
- 至少 3 条盲测自然通过。

## 人工评测流程

1. 打开 Hermes。
2. 确认 `byway-family-travel-assistant` 为当前可用 Skill。
3. 对照 Golden Cases 逐条输入，不附加评分提示。
4. 保存每轮原始回答和 trace / log。
5. 按评分维度打分。
6. 标出 P0 失败和边界退化。
7. 只根据重复出现的问题修改 Skill，不因为单个措辞问题堆规则。

## 修改 Skill 后的判断

一次修改算有效，必须同时满足：

- 修复目标问题。
- 没有增加用户可见复杂度。
- 没有扩大 Byway 当前产品边界。
- 没有让 Skill 主文件显著膨胀。
- Golden Cases 不退化。

如果为了修一个用例，需要加入大量规则，优先重新讨论产品边界，而不是继续堆 Skill 文本。
