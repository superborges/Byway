# Byway 项目交接

日期：2026-06-09

## 当前状态

Byway 已收敛为 Hermes 中的家庭旅行 Skill。当前仓库不再维护独立 Web App、Backend、MCP、schema 或 Artifact 工作台。

当前可用版本：

- Skill 源码：`skills/byway-family-travel-assistant/SKILL.md`
- Hermes 本地副本：`~/.hermes/skills/travel/byway-family-travel-assistant/`
- Skill 版本：`0.4.3`

当前最重要的产品判断：

- 继续把 Byway 当作 Hermes 的家庭旅行能力，而不是独立产品系统。
- 真实使用优先于继续开发；优先用端午青岛旅行这类真实场景验收。
- 后续只修真实对话里反复出现的问题，不因为单个措辞问题堆规则。

## 恢复工作顺序

1. 先读 `AGENTS.md` 和 `README.md`。
2. 再读 `skills/byway-family-travel-assistant/SKILL.md`。
3. 如果要改 Skill，再读 `skills/byway-family-travel-assistant/references/sops.md`。
4. 如果要评测，再读 `docs/BYWAY_SKILL_EVAL.md` 和 `docs/evals/byway_skill_golden_cases.md`。
5. 归档目录只查历史，不作为当前实现依据。

## 同步到 Hermes

修改仓库内 Skill 后，同步到 Hermes 本地目录：

```bash
rsync -a --delete \
  skills/byway-family-travel-assistant/ \
  ~/.hermes/skills/travel/byway-family-travel-assistant/
```

同步后可检查：

```bash
diff -qr \
  skills/byway-family-travel-assistant \
  ~/.hermes/skills/travel/byway-family-travel-assistant
```

## 快速验收

最小冒烟用例：

```bash
hermes chat -Q --source tool --max-turns 6 \
  -s byway-family-travel-assistant \
  -q "灵隐寺今天几点关门？从西湖过去大概多久？适合放在同一天吗？"
```

期望：

- 不触发 `Reached maximum iterations`。
- 地点和路线事实来自可用地图工具，且说明来源。
- 不因为地图已有结果仍继续网页搜索或官网核验。
- 对宽泛地点给可用估算，并说明实际耗时取决于具体出发点。

完整评测按 `docs/BYWAY_SKILL_EVAL.md` 执行。

## 本地文件

以下内容是本地运行/评测数据，已被 `.gitignore` 忽略：

- `.env.local`
- `.byway/`
- `.DS_Store`

不要把这些文件提交到仓库。
