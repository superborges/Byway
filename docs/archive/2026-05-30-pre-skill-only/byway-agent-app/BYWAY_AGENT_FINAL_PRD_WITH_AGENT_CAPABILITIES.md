# 另辟蹊径 / Byway Agent — 最终产品规划 PRD

> **产品定位一句话**：Byway 是一个面向家庭旅行的全 Agent 智能助手。它可以在用户不知道去哪时推荐合适目的地；在目的地确定后自动研究攻略、生成住宿/用餐/路线计划；在旅行中根据突发状况实时调整；在旅行后沉淀家庭旅行 Memory，让下一次推荐和计划更懂这个家庭。

---

## 0. 文档目标

这份 PRD 描述 Byway 的最终完整产品目标，不拆分 MVP / V1 / V2。它关注产品体验、Agent 交互、后台能力、数据对象和核心边界。

本版相对于之前的 planner / app 方案有一个根本变化：

- 不再做传统 App。
- 不再让用户维护地点、节点、星级、计划字段。
- 不再以表单、列表、逐项审核作为主交互。
- 改为移动端 HTML / PWA + Hermes Agent 后台 + Byway MCP 工具层。
- 用户只表达意图、补充资料、反馈现场状况。
- Agent 负责研究、追问、规划、校验、调整、解释和记忆沉淀。

核心原则：

> **用户不维护旅行计划，用户只表达目的和反馈；Agent 维护计划，并在关键变化前请求确认。**

---

## 1. 产品背景

目标用户是一类高频家庭旅行者：

- 每年多次旅行。
- 有国内和海外旅行经验。
- 家庭组合复杂，常见 3 人 / 4 人 / 6 人出行。
- 需要同时考虑夫妻、老人、12 岁左右孩子、3 岁左右幼童。
- 真正困难的不是“有没有攻略”，而是：
  - 去哪里合适；
  - 这些攻略是否可信；
  - 住宿住在哪片区域；
  - 吃饭如何嵌入路线；
  - 路线是否真实可执行；
  - 老人孩子体力如何影响计划；
  - 旅行中计划被打乱后怎么调整；
  - 这次经验如何沉淀到下次。

过去的方案偏“AI 加持的旅行计划管理工具”。新版 Byway 应该是：

> **能对话、会追问、能研究攻略、会定位地点、能维护计划、能在旅行中实时调整的家庭旅行 Agent。**

---

## 2. 产品名称

中文名：**另辟蹊径**
英文名：**Byway**

命名含义：

- “另辟蹊径”强调不是复制大众攻略，而是找到适合自己家庭的路线。
- “Byway”意为小路、旁路、非主干道路，表达“找到属于我们家的旅行路径”。
- 产品不是导航 App，也不是 OTA 平台，而是一个懂家庭约束的旅行 Agent。

---

## 3. 产品愿景

Byway 最终希望解决的是：

> **从“不知道去哪”到“旅行结束复盘”的完整家庭旅行决策和执行闭环。**

完整闭环：

```text
Dream：不知道去哪，Agent 推荐合适目的地
↓
Plan：目的地确定后，Agent 自动研究攻略并生成计划
↓
Travel：旅行中，Agent 根据突发事件实时调整
↓
Review：每天和整趟旅行结束后，Agent 生成日志
↓
Memory：沉淀家庭旅行偏好和约束，反哺下一次 Dream / Plan
```

---

## 4. 产品非目标

Byway 不做以下事情：

```text
不做 OTA / 旅行交易平台
不做机票 / 高铁 / 酒店 / 餐厅预订
不做大众点评 / 携程 / 飞猪替代品
不做旅行社区
不做全网小红书爬虫
不做真实评论抓取和评论排序
不做复杂地图导航
不做实时路况承诺
不做门票购买 / 抢票 / 预约
不做完整预算系统
不做多人社交协作平台
不做用户手工维护节点的复杂计划工具
```

Byway 可以提供：

```text
目的地推荐
攻略研究和合并
住宿区域建议
用餐区域建议
地点事实定位
路线合理性判断
旅行计划生成
旅行中事件调整
日志和家庭 Memory
跳转高德地图 / 其他地图进行导航或打车
```

---

## 5. 核心设计原则

### 5.1 Agent-first，而不是 UI-first

产品主入口是对话。用户不需要理解底层对象、节点、POI、路线模板、计划版本。

用户说：

```text
我想暑假带老人孩子从北京出发玩 4 天，不知道去哪。
```

Agent 应该理解这是 Dream Mode。

用户说：

```text
我想去青岛 4 天，轻松一点。
```

Agent 应该理解这是 Plan Mode。

用户说：

```text
我们晚了 40 分钟，老人累了。
```

Agent 应该理解这是 Travel Mode。

---

### 5.2 前端极简，后台智能

前端只做：

```text
对话输入
流式回复展示
计划卡片展示
今日卡片展示
资料上传/粘贴
确认按钮
```

前端不做：

```text
复杂节点编辑
表格型审核
地点管理页
路线手工编排页
debug 面板
```

---

### 5.3 Agent 主动追问，但一次最多问关键问题

Agent 不应该一次性抛出复杂表单。它最多追问 3 个阻塞信息。

例如：

```text
我可以先帮你按“北京出发、4 天含往返、6 人全家、轻松节奏”做一版。
不过这 3 个信息会明显影响计划：
1. 大概几月出行？
2. 这次更想海边、历史文化、美食，还是亲子放松？
3. 第一天下午到、最后一天下午走这个假设可以吗？
```

如果用户暂时无法回答，Agent 应该用明确假设继续推进。

---

### 5.4 LLM 负责研究和解释，地图工具负责事实

LLM 可以判断：

```text
这个目的地适不适合老人孩子
攻略共识是什么
住宿区域怎么选
路线为什么这么排
发生突发事件后如何取舍
```

LLM 不应该凭空生成：

```text
标准 POI 名称
精确坐标
电话
营业时间
评分
打车地址
入口坐标
真实路线耗时
```

这些必须通过高德 / Google / 官方地图工具查询，并带置信度。

---

### 5.5 关键变更必须确认

Agent 可以自动记录：

```text
迟到
超时
老人累
孩子累
大家饿了
下雨
想回酒店
```

Agent 可以自动建议：

```text
取消低优先级节点
顺延后续计划
调整晚餐区域
改成酒店附近休息
```

但以下动作必须用户确认：

```text
取消 4/5 星核心景点
大幅重排当天计划
改变明天计划
删除用户明确保留的事项
改变酒店 / 交通 / 用餐 anchor
```

---

### 5.6 Memory 需要确认后生效

旅行后提取出的家庭 Memory 不应自动无限写入。流程应为：

```text
Travel Events
↓
Day Log
↓
Trip Log
↓
Family Memory Candidate
↓
用户确认
↓
Accepted Family Memory
↓
影响下一次 Dream / Plan / Travel
```

---

## 6. 目标用户与家庭画像

默认服务对象是高频家庭旅行者。

典型家庭：

```text
夫妻：40 岁左右，互联网公司中层，旅行经验丰富
父母：65 岁左右，体力和步行耐受有限
男孩：12 岁左右，能接受城市探索、博物馆、轻户外
女孩：3 岁左右，需要考虑午睡、推车、餐食、情绪和体力
```

常见旅行组合：

```text
3 人：夫妻 + 12 岁男孩
4 人：夫妻 + 两个孩子，复杂度较高
6 人：夫妻 + 两个孩子 + 双方/一方父母
```

Agent 必须理解：

```text
3 人出行可以更灵活、更充实
6 人出行必须更稳、更松、更重视吃饭和休息
4 人出行因 3 岁孩子存在不确定性
海外旅行通常更适合 3 人
国内旅行更适合 6 人
```

---

## 7. 产品整体体验

### 7.1 主界面

移动网页 / PWA 的核心界面由三部分组成：

```text
顶部：当前旅行状态
中间：Agent 对话流
底部：输入框 + 快捷输入
浮动/折叠卡片：当前计划 Artifact
```

顶部状态示例：

```text
Byway / 另辟蹊径
当前：青岛 4 天家庭游
状态：准备中 / 旅行中 Day 2 / 已完成
```

输入方式：

```text
文字
语音转文字
粘贴 markdown 攻略
粘贴小红书链接
上传截图
输入旅行中事件
```

---

### 7.2 Artifact 卡片

对话不是唯一输出。Agent 的结构化结果必须形成 Artifact。

主要 Artifact：

```text
Destination Recommendation Card
Guide Research Summary
Place Fact Review Card
Travel Plan Card
Today Card
Replan Options Card
Day Log Card
Trip Log Card
Family Memory Card
```

用户不需要编辑底层节点，但需要清楚看到当前计划。

---

## 8. Agent 模式

Byway Agent 有四个核心模式。

```text
Dream Mode：不知道去哪
Plan Mode：已经知道目的地
Travel Mode：旅行中
Review Mode：日志与记忆
```

### 8.1 Dream Mode：目的地推荐

触发方式：

```text
不知道去哪
帮我推荐一个地方
北京出发 4 天带老人孩子去哪
暑假适合全家去哪
国庆不知道去哪玩
```

Agent 需要收集或假设：

```text
出发地
出行月份 / 日期
天数
同行人
预算倾向
国内/海外偏好
旅行主题
节奏偏好
历史旅行 Memory
```

Agent 输出：

```text
3-5 个候选目的地
每个目的地的适合度
推荐天数
交通适配
家庭适配
住宿/用餐便利度
主要风险
为什么推荐
为什么不推荐某些热门目的地
```

示例输出：

```text
如果这次是北京出发、4 天、6 人全家、轻松节奏，我最推荐青岛和杭州。

1. 青岛
适合度：高
推荐天数：4 天
原因：海边 + 城市成熟 + 晚餐和住宿选择稳定，对老人孩子友好。
风险：暑期海边人多，热门区域酒店价格偏高。

2. 杭州
适合度：高
推荐天数：4 天
原因：城市成熟、吃住方便、路线可松可紧。
风险：暑期湿热，西湖周边人多。

3. 西安
适合度：中高
推荐天数：4-5 天
原因：文化内容强，12 岁孩子会有收获。
风险：兵马俑占用半天到一天，6 人全家 4 天会略紧。
```

用户选择目的地后，自动进入 Plan Mode。

---

### 8.2 Plan Mode：目的地已知后的旅行计划

触发方式：

```text
我想去青岛 4 天
帮我规划西安 3 天 2 晚
这是我去杭州的攻略，帮我整理计划
```

Agent 需要：

```text
自动研究目的地攻略
支持用户补充攻略
理解同行人和节奏
询问抵达/离开时间
建议住宿区域
建议用餐区域
定位关键地点
生成 Day-by-day 计划
检查路线是否合理
```

Agent 输出：

```text
住宿区域建议
用餐区域建议
核心景点与可选景点
每日计划
路线理由
风险提示
需要用户确认的问题
```

计划不是静态文本，而是 Plan Artifact。

---

### 8.3 Travel Mode：旅行中实时调整

触发方式：

```text
我们晚了 40 分钟
老人累了
孩子不坐车
这个景点超时了
现在大家饿了
下雨了
想回酒店
```

Agent 必须读取：

```text
当前 Day
当前节点
下一个节点
剩余节点
酒店/住宿区域
用餐 anchor
地点星级/优先级
Family Memory
当前事件
```

Agent 输出：

```text
调整建议
取消方案
顺延方案
重排方案
推荐方案
影响范围
是否需要用户确认
```

示例：

```text
你们现在比计划晚 40 分钟，而且老人累了。

我建议：
保留：晚餐和酒店附近安排
取消：小鱼山，2 星，可选
顺延：信号山改为明天上午可选
缩短：八大关停留时间从 90 分钟缩短到 50 分钟

原因：继续完整执行会导致晚餐过晚，且后续步行压力较高。
是否应用这个调整？
```

---

### 8.4 Review Mode：日志与家庭记忆

触发方式：

```text
今天结束了
帮我总结今天
这次旅行结束了
```

也可以由 Agent 在当天晚间主动提示。

每天结束生成 Day Log：

```text
计划完成情况
取消/顺延/重排记录
突发事件
体验问题
当天亮点
潜在家庭规律
```

旅行结束生成 Trip Log：

```text
整趟旅行总结
计划 vs 实际
最成功安排
最失败安排
体力与用餐问题
住宿区域评价
下次建议
```

提取 Family Memory Candidate：

```text
父母同行时，下午 3 点后不适合继续安排高步行强度景点。
3 岁孩子同行时，午餐最好不要晚于 12:40。
6 人全家出行时，晚餐应优先安排在酒店 20 分钟交通圈内。
上午高强度博物馆后，下午更适合安排低强度开放空间或酒店休息。
```

用户确认后写入 Accepted Family Memory。

---

## 9. 核心功能模块

### 9.1 Trip Intake：旅行意图理解

Agent 需要从自然语言中抽取：

```text
是否知道目的地
目的地
出发地
天数
日期/月份
同行人
节奏偏好
预算倾向
主题偏好
是否已有交通
是否已有酒店
是否已有攻略资料
```

缺失信息处理：

```text
阻塞信息：需要追问
非阻塞信息：使用明确假设推进
```

---

### 9.2 Destination Recommendation：目的地推荐

目的地推荐不是简单热门城市列表，而是基于家庭约束的适配判断。

考虑维度：

```text
从出发地可达性
门到酒店复杂度
本次天数是否够
老人孩子友好度
住宿便利性
用餐便利性
景点分布集中度
城市移动压力
攻略确定性
季节风险
预算压力
历史 Memory 适配
```

输出结构：

```text
候选目的地名称
一句话推荐结论
适合度
推荐天数
适合旅行组合
交通判断
住宿和用餐判断
核心风险
不推荐原因
Plan Preview
```

---

### 9.3 Guide Research：自动攻略研究

Byway 不要求用户先去问 ChatGPT / 豆包 / 千问再粘贴 markdown。Agent 可以自动生成多份 Synthetic Guide。

攻略研究视角：

```text
经典首次游攻略
老人孩子友好攻略
住宿区域攻略
用餐区域攻略
避坑攻略
雨天/疲劳备选攻略
低强度路线攻略
```

每份攻略输出统一结构：

```text
住宿区域候选
用餐区域候选
核心地点
可选地点
路线建议
推荐停留时间
家庭适配
老人风险
孩子风险
注意事项
```

用户仍可补充：

```text
AI 生成的 markdown
小红书链接
小红书截图
朋友建议
手写备注
```

小红书处理原则：

```text
能读到文字/截图就抽取
只有链接则保存为引用和备注
不承诺抓取完整笔记、评论、视频内容
不做小红书爬虫
```

---

### 9.4 Guide Synthesis：多源攻略合并

Agent 需要合并：

```text
Synthetic Guide Sources
User Guide Sources
Family Memory
Place Facts
```

合并目标：

```text
发现共识
发现分歧
去重地点
合并别名
形成核心/可选地点池
形成住宿区域建议
形成用餐区域建议
形成初步路线思路
生成风险提示
```

输出示例：

```text
我综合了 5 个攻略视角：

共识很强：
- 八大关、栈桥、五四广场、小麦岛都被多次推荐。

存在分歧：
- 崂山：体验价值高，但对 6 人全家强度偏高。

住宿区域：
- 首选五四广场 / 奥帆中心附近。
- 备选老城区。
- 不建议住崂山附近作为全程 base。
```

---

### 9.5 PlaceFact：地点事实层

这是 Byway 可靠性的地基。

Agent 不得凭空确认地点事实。所有关键地点需要通过地图工具解析。

PlaceFact 字段：

```text
inputName：用户/攻略中的原始名称
standardName：地图标准名称
aliases：别名
provider：amap / google / manual
providerPlaceId：地图 POI ID
city / district / adcode
coordinate：经纬度
address：地址
displayAddress：展示地址
taxiAddress：建议打车地址
entranceCoordinate：入口坐标 / 到达点
navigationPoiId：导航 POI ID
phone：电话
openingHoursToday：今日营业时间
openingHoursWeek：周营业时间
rating：评分
photos：照片
businessArea：商圈
confidence：verified / high / medium / low / unresolved
needsReview：是否需要用户确认
lastResolvedAt：最后解析时间
```

关键规则：

```text
5 星/核心地点必须有 high 或 verified 置信度；否则 Agent 需要询问用户确认。
大型景区必须关注入口/打车点，不只用景区中心点。
营业时间、评分、电话为可选字段，缺失时不应阻塞计划。
评论正文不作为地图事实字段，不做真实评论抓取。
```

国内默认使用高德。高德开放平台支持 POI 搜索、地理/逆地理编码、路径规划和 URI 调用能力；POI 搜索返回的字段可包含名称、地址、经纬度、电话、入口经纬度、导航 POI ID、别名、商圈、评分和照片等，具体字段按接口和权限可用性处理。高德也提供 MCP Server 方向的集成能力，适合 Agent 工具层接入。

---

### 9.6 RouteFact：路线事实层

Byway 不做导航，但必须知道路线是否大体合理。

RouteFact 需要支持：

```text
两点路线估算
多点路线矩阵
步行/驾车/公交模式
距离
预计时间
路线来源
置信度
缓存时间
```

用途：

```text
判断是否明显绕路
判断是否走回头路
判断景点是否太分散
判断午餐/晚餐是否顺路
判断住宿区域是否适合作为 base
判断当天老人孩子是否会过载
```

Plan 输出时，Agent 不需要展示复杂路线，但需要解释：

```text
Day 2 我把老城和八大关放在同一天，因为它们相对顺路；没有把崂山塞进这一天，因为距离和强度都偏高。
```

---

### 9.7 Plan Generation：旅行计划生成

Plan 是结构化 Artifact。

Plan 需要包含：

```text
目的地
天数
同行人
节奏
住宿区域建议
用餐区域建议
每日计划
每个节点的优先级
每个节点的类型
交通/用餐/休息 anchor
风险提示
Plan Assumptions
```

PlanNode 类型：

```text
activity：景点/活动
meal：用餐区域/餐厅
hotel：酒店/住宿
transport：抵达/离开/长距离交通
rest：休息/午睡/回酒店缓冲
optional：可选填充项
```

优先级定义：

```text
5 星：本次核心目标，尽量不能取消
4 星：强烈希望去，除非当天明显崩盘
3 星：正常安排，可顺延/替换
2 星：可选项目，冲突时优先取消
1 星：顺路看看，随时可取消
```

Meal / Hotel / Transport / Rest 是 anchor，不应被普通活动挤掉。

---

### 9.8 Lodging Area：住宿区域建议

Byway 不替用户订酒店，但需要建议住宿区域。

Agent 需要输出：

```text
首选住宿区域
备选住宿区域
不建议区域
每个区域适合什么组合
到核心景点是否方便
晚餐是否方便
老人孩子是否友好
机场/高铁站可达性
```

用户可以自行去携程 / 飞猪 / Booking 等平台订酒店。订好后告诉 Agent：

```text
我订了 XX 酒店。
```

Agent 需要：

```text
定位酒店 PlaceFact
更新路线
重排计划
检查晚餐和返程是否顺路
```

---

### 9.9 Meal Area：用餐区域建议

Byway 不替代大众点评，不做餐厅排行榜。

Byway 负责：

```text
这顿饭适合在哪个区域解决
这顿饭应靠近哪个景点/酒店/返程方向
是否需要提前吃
是否应避免排队网红店
老人孩子是否需要稳定用餐
```

输出示例：

```text
Day 2 午餐建议在小寨 / 大雁塔附近解决，不建议返回钟楼吃饭，否则下午会走回头路。
Day 2 晚餐建议放在酒店附近 20 分钟交通圈内，因为当天上午博物馆 + 下午步行强度较高。
```

用户可以自行去大众点评找具体餐厅，再告诉 Agent 餐厅名称。Agent 负责定位和更新计划。

---

### 9.10 Active Travel Runtime：旅行中状态

旅行中，Agent 必须维护：

```text
当前 Day
当前节点
下一个节点
剩余节点
已完成节点
跳过节点
延迟情况
旅行事件
酒店位置
用餐 anchor
```

Today Card 显示：

```text
当前：正在参观 A
下一个：午餐 / B 景点
后续：C / 晚餐 / 酒店
当前风险：晚了 35 分钟 / 老人可能疲劳 / 午餐时间接近
```

---

### 9.11 Travel Event：旅行事件

事件类型：

```text
late：迟到
current_overrun：当前节点超时
child_tired：孩子累了
elder_tired：老人累了
hungry：大家饿了
bad_weather：天气变差
want_return_hotel：想回酒店
transport_problem：交通不顺
place_closed：地点关闭
queue_too_long：排队太久
place_disappointing：地点体验低于预期
place_exceeded_expectation：地点体验超预期
custom：自定义事件
```

事件记录需要包含：

```text
时间
地点
相关节点
影响人群
严重度
用户描述
Agent 建议
用户最终选择
计划变更结果
```

---

### 9.12 Active Replan：旅行中重排

每次事件发生后，Agent 输出三类方案：

```text
方案 A：取消
方案 B：顺延
方案 C：重排
```

每个方案需要包含 diff：

```text
取消哪些节点
保留哪些节点
顺延哪些节点
调整哪些时间
是否影响晚餐/酒店/明天
原因
```

推荐方案必须明确。

示例：

```text
方案 A：取消下一个 2 星景点
- 取消：小鱼山
- 保留：八大关、晚餐
- 优点：最稳，不影响晚餐

方案 B：整体顺延
- 所有后续节点延后 40 分钟
- 风险：晚餐会偏晚

方案 C：重排剩余计划
- 八大关缩短到 50 分钟
- 小鱼山取消
- 晚餐提前到酒店附近
- 推荐：方案 C
```

用户确认后应用。

---

### 9.13 Logs & Family Memory

#### Day Log

每天结束生成：

```text
当天实际路线
完成/取消/顺延节点
突发事件
体验亮点
失败点
体力/用餐/交通问题
明天计划建议
```

#### Trip Log

旅行结束生成：

```text
整趟旅行总结
目的地适配评价
住宿区域评价
用餐节奏评价
老人孩子体力表现
哪些安排成功
哪些安排失败
下次建议
```

#### Family Memory

Memory 分类：

```text
stamina：体力
meal：用餐
transport：交通
lodging：住宿
pace：节奏
interest：兴趣
child：孩子约束
elder：老人约束
weather：天气影响
risk：失败模式
```

Memory 结构：

```text
规则
适用场景
置信度
证据旅行
证据事件
最近验证时间
是否已被用户确认
```

示例：

```text
规则：父母同行时，下午 3 点后不适合继续安排高步行强度景点。
适用场景：6 人全家 / 城市步行 / 夏季或高强度上午之后
证据：青岛 Day 2、杭州 Day 3
置信度：0.78
状态：待用户确认
```

---

## 10. 技术产品架构

最终目标架构：

```text
Mobile HTML / PWA
        ↓
Byway Proxy API
        ↓
Hermes Agent API Server
        ↓
Byway Hermes Skill
        ↓
Byway MCP Server
        ├── Trip State Tools
        ├── Dream / Destination Tools
        ├── Guide Research Tools
        ├── PlaceFact / Geo Tools
        ├── Plan Artifact Tools
        ├── Travel Runtime / Replan Tools
        ├── Log / Memory Tools
        └── External Provider Tools
                ├── LLM Providers
                ├── 高德 Web Service / 高德 MCP
                ├── Google Maps / Places / Routes for overseas
                ├── Web Search
                └── Database / Cache
```

核心比喻：

> **Hermes 是大脑，高德是眼睛和地图，Byway MCP 是手脚，Byway DB 是记账本。**

---

## 11. Hermes Agent 角色

Hermes 用于：

```text
自然语言理解
多轮对话
主动追问
调用工具
多模型接入
技能加载
长期记忆辅助
流式输出
```

Hermes 不直接承担：

```text
唯一业务数据库
唯一计划状态源
地点事实判断
路线事实判断
所有计划变更的自动执行
```

Hermes Profile：

```text
profile: byway
purpose: family travel planning and travel-time adjustment agent
```

Byway Skill 定义：

```text
身份
工作流
模式切换
追问策略
事实校验原则
计划生成原则
旅行中事件处理原则
Memory 写入原则
安全边界
```

---

## 12. LLM Agent 能力边界与智能要求

本章节用于明确：在 Byway 中，Hermes Agent 到底需要承担哪些智能能力，哪些能力必须依赖工具，哪些事情不得由 LLM 凭空完成。

核心判断：

> **Byway 的智能体验来自 Agent 的规划、追问、综合、取舍和解释；Byway 的可靠性来自 MCP 工具、结构化状态、地点事实和用户确认。**

也就是说，Hermes Agent 不是一个“会聊天的 UI 层”，而是 Byway 的智能编排层；但它也不是唯一事实源、唯一数据库或无约束执行器。

---

### 12.1 Hermes Agent 在 Byway 中的职责定位

Hermes Agent 应承担以下职责：

```text
理解用户自然语言意图
判断当前属于 Dream / Plan / Travel / Review 哪个模式
识别缺失信息和关键风险
主动追问少量关键问题
在信息不足时基于明确假设继续推进
调用 Byway MCP 工具读取和更新结构化状态
调用多个 LLM / 多种 prompt 视角进行攻略研究
合成多源攻略共识与分歧
基于家庭 Memory 做目的地和路线取舍
解释为什么推荐、为什么不推荐、为什么调整
在旅行中根据突发事件生成可执行调整方案
在旅行后总结日志并提取 Memory 候选
```

Hermes Agent 不应承担以下职责：

```text
不作为唯一业务数据库
不直接持久化旅行计划，必须通过工具写入
不凭空生成地点坐标、电话、营业时间、评分、打车地址
不凭空假设高铁/航班/酒店价格/门票状态
不自动执行高影响计划变更
不绕过用户确认取消核心景点、改动酒店、写入长期 Memory
不替代高德 / Google Maps / 官方信息等事实源
不替代用户最终判断
```

一句话边界：

> **Hermes Agent 可以思考、研究、建议和解释；凡是事实、状态、持久化、执行和高影响变更，都必须通过 Byway MCP 工具完成。**

---

### 12.2 Agent 工作循环

Byway 的每一次复杂响应都应该遵循以下工作循环：

```text
Understand：理解用户意图和上下文
↓
Classify：判断当前模式与任务类型
↓
Check State：读取 Trip / Plan / Memory / PlaceFact 等状态
↓
Ask or Assume：必要时追问；不阻塞时说明假设
↓
Use Tools：调用 MCP 工具获取事实、生成 artifact、记录事件
↓
Reason：基于工具结果进行取舍和解释
↓
Propose：给出目的地/计划/调整建议
↓
Confirm：关键变化前请求用户确认
↓
Persist：通过工具写入状态、计划、日志或 Memory 候选
```

Agent 不应该在没有读取状态的情况下直接回答旅行中调整问题。例如用户说：

```text
我们晚了 40 分钟，老人累了。
```

Agent 必须先获得：

```text
当前 Trip
当前 Day
当前节点
下一个节点
当天剩余计划
酒店/用餐 anchor
地点优先级
家庭 Memory
已发生事件
```

然后才能生成调整方案。

---

### 12.3 Agent 能力等级

Byway 中的 Agent 能力可以分为 5 个等级。

```text
L0：普通回答
- 解释产品能力
- 回答非状态型问题
- 不产生 artifact，不更新状态

L1：对话收集
- 识别目的地、天数、同行人、偏好
- 追问缺失信息
- 更新 TripBrief

L2：工具增强生成
- 调用攻略研究、地点事实、路线估算工具
- 生成目的地候选、计划草稿、住宿/用餐建议

L3：多步骤 Agent 工作流
- 多模型攻略研究
- 多源共识合并
- 地点事实校验
- 计划 artifact 生成
- 风险诊断和解释

L4：旅行中实时应对
- 理解突发事件
- 读取当前计划状态
- 生成取消 / 顺延 / 重排多方案
- 保护 meal / hotel / rest / transport anchor
- 请求确认并应用计划变更
```

Byway 的核心功能至少需要达到：

```text
目的地推荐：L3
自动攻略研究：L3
旅行计划生成：L3
旅行中突发事件应对：L4
旅行后 Memory 提取：L3
```

---

### 12.4 Intent Routing：意图识别与模式切换能力

Agent 必须能从自然语言中判断用户当前意图。

典型模式：

```text
Dream Mode：用户不知道去哪
Plan Mode：用户已经知道目的地，需要生成计划
Travel Mode：用户正在旅行中，报告突发状况或进展
Review Mode：用户结束一天或结束整趟旅行
General Mode：解释、闲聊、设置类问题
```

示例：

```text
“暑假带父母孩子从北京出发 4 天，不知道去哪”
→ Dream Mode

“我想去青岛 4 天，轻松一点”
→ Plan Mode

“我们晚了 40 分钟，老人累了”
→ Travel Mode

“今天结束了，帮我总结一下”
→ Review Mode
```

Agent 需要支持用户在同一段对话中自然切换模式。例如：

```text
用户先说不知道去哪
Agent 推荐青岛
用户说“那就青岛”
Agent 应自动从 Dream Mode 切换到 Plan Mode
```

模式切换必须通过 `set_trip_phase` 或相应 Trip State Tool 持久化。

---

### 12.5 缺失信息处理能力

Agent 不应该把旅行准备变成表单。它必须区分：

```text
阻塞信息：没有它无法生成合理结果
重要信息：有它更准，但可先假设
可选信息：没有也不影响当前阶段
```

Dream Mode 的阻塞信息通常是：

```text
出发地
大致天数
同行人结构
大致时间/季节
```

Plan Mode 的阻塞信息通常是：

```text
目的地
旅行天数
同行人
大致抵达/离开时间，若涉及第一天/最后一天安排
```

Travel Mode 的阻塞信息通常是：

```text
当前 Trip
当前 Day
当前节点或大致位置
突发事件类型
严重程度，若影响取舍
```

Agent 追问规则：

```text
一次最多问 3 个问题
优先问会显著改变结果的问题
如果用户没有回答，允许用明确假设继续
假设必须写进 artifact.assumptions
后续用户补充信息时，Agent 应主动重算受影响部分
```

示例：

```text
我先按“北京出发、4 天含往返、6 人全家、轻松节奏”推荐一版。
但这 2 个信息会显著影响结果：
1. 具体是暑假还是国庆？
2. 这次更想海边放松还是历史文化？
```

---

### 12.6 多模型攻略研究能力

Byway 不应要求用户先去 ChatGPT / 豆包 / 千问分别问攻略再粘贴。Agent 应具备自动生成多视角攻略底稿的能力。

`generate_synthetic_guides` 应至少支持以下研究视角：

```text
经典首次游路线
老人孩子友好路线
低强度慢节奏路线
住宿区域分析
用餐区域和用餐节奏分析
避坑和不推荐项
雨天/疲劳备选方案
```

每个 synthetic guide 应保留：

```text
来源 provider / model
promptRole
生成时间
主要假设
结构化抽取结果
适用场景
置信度
```

Agent 不能简单把多个模型结果拼接。它必须形成：

```text
共识：多个来源都支持的结论
分歧：来源之间明显冲突的结论
少数观点：只有一个来源提到，但可能有价值
风险：来源提醒的坑、排队、过度商业化、体力压力
不确定：需要地图/用户/实时信息确认的部分
```

多模型研究的结果只能作为“攻略判断层”，不能作为“事实层”。例如：

```text
多个模型都说“八大关适合下午去”
→ 可作为计划偏好

多个模型都写了“八大关地址”
→ 仍必须调用 PlaceFact 工具确认标准 POI 和坐标
```

---

### 12.7 目的地推荐智能要求

Destination Recommendation 不是简单列热门城市。Agent 必须根据家庭约束做取舍。

输入：

```text
出发地
出行时间/季节
可用天数
同行人结构
预算倾向
国内/海外倾向
旅行主题偏好
历史 Family Memory
```

Agent 必须考虑：

```text
从出发地的可达性
天数是否足够
老人孩子友好度
城市移动压力
酒店区域是否容易选择
用餐是否方便
景点内容密度
攻略确定性
季节/天气/节假日风险
预算压力
与历史旅行 Memory 的匹配度
```

输出要求：

```text
推荐 3-5 个候选目的地
每个目的地给出适合度和主要理由
明确说明适合哪种家庭组合
明确说明本次天数是否够
明确说明主要风险
明确说明为什么某些热门目的地这次不优先推荐
用户选择后能无缝进入 Plan Mode
```

目的地推荐必须调用工具生成结构化 artifact：

```text
generate_destination_candidates
research_destination_candidate
score_destination_candidates
```

如果涉及当前交通、天气、签证、政策、价格等强时效信息，Agent 必须说明是否已调用实时工具；没有工具时不得假装已确认。

好的目的地推荐示例：

```text
这次如果是北京出发、4 天、6 人全家、节奏轻松，我最推荐青岛和杭州。

青岛适合度最高：交通和城市成熟度好，海边和城市结合，老人孩子压力较低。
西安内容更强，但 4 天对 6 人全家略紧，兵马俑会占用一整天。
重庆这次不优先推荐，因为山城上下坡、打车和热门餐饮排队对老人孩子压力更大。
```

---

### 12.8 旅行计划生成智能要求

Plan Generation 的目标不是生成“看起来丰富”的攻略，而是生成“真实可执行”的家庭计划。

Agent 必须基于以下输入生成计划：

```text
TripBrief
CanonicalGuide
FamilyMemory
PlaceFact
RouteFact
抵达/离开约束
酒店/住宿区域约束
用餐策略
```

计划必须包含：

```text
住宿区域建议
用餐区域策略
每日主题
每日核心节点
meal / hotel / rest / transport anchor
可选节点
风险提示
关键假设
需要用户确认的问题
```

计划生成原则：

```text
每天不要过度塞景点
老人/幼童同行时优先保留休息和用餐
第一天和最后一天要尊重抵达/离开时间
大型景区和全日活动不能被拆得过碎
高优先级景点需要 PlaceFact 高置信度或明确提醒
路线不追求全局最优，但不能明显来回穿城
晚餐尽量靠近酒店或当天最后活动区域
不确定的营业时间只能作为提示，不能伪装成已确认
```

Agent 不得直接把自由文本当成最终计划。最终计划必须由 `generate_plan` 或 `update_plan_artifact` 生成结构化 `PlanArtifact`。

如果用户补充新信息，例如酒店地址、交通时间、小红书链接，Agent 应判断影响范围：

```text
只影响第一天/最后一天
只影响住宿区域建议
影响整条路线
影响部分地点优先级
不影响当前计划，只作为备注
```

然后调用相应工具修订计划，而不是完全重来。

---

### 12.9 旅行中突发事件应对智能要求

Travel Mode 是 Byway 最体现智能价值的场景。Agent 必须能处理旅行中的不确定性。

典型事件：

```text
迟到
当前景点游玩超时
老人累了
孩子累了
大家饿了
天气不好
排队太久
景点临时关闭
交通不顺
想回酒店
临时想加一个地点
```

Agent 处理步骤：

```text
1. 调用 get_today_status 读取当前状态
2. 调用 record_travel_event 记录事件
3. 判断事件严重度和影响范围
4. 保护 meal / hotel / rest / transport anchor
5. 根据节点优先级、家庭 Memory、剩余时间生成调整方案
6. 调用 replan_today 生成方案
7. 用自然语言解释方案差异
8. 对高影响变更请求用户确认
9. 用户确认后调用 apply_replan
```

`replan_today` 必须至少返回：

```text
取消方案
顺延方案
重排方案
推荐方案
每个方案的影响范围
需要保留的 anchor
被取消/顺延/调整的节点
为什么推荐这个方案
```

Agent 不应只说：

```text
建议你们休息一下。
```

而应给出可执行调整：

```text
我建议应用“轻量重排”方案：
- 取消下一个 2 星的小鱼山
- 保留 18:00 酒店附近晚餐
- 把 4 星的八大关缩短为 40 分钟，只保留核心段
- 信号山改为明天上午可选

原因：你们已经晚了 40 分钟，且老人反馈疲劳；继续完整走原计划会导致晚餐过晚。
是否应用？
```

高影响变更必须确认：

```text
取消 4/5 星核心景点
改动明天计划
取消酒店/交通/用餐 anchor
将旅行状态标记为完成
写入长期 Memory
```

低影响动作可以自动记录：

```text
记录“迟到 20 分钟”事件
记录“当前节点超时”事件
生成一条未应用的调整建议
```

---

### 12.10 Review 与 Memory 提取智能要求

Agent 应把旅行中的执行情况变成未来可用的家庭经验，而不是只生成流水账。

Day Log 应总结：

```text
原计划 vs 实际执行
完成了哪些节点
取消/顺延了哪些节点
发生了哪些事件
哪些安排有效
哪些安排失败
当天对后续规划有什么启发
```

Trip Log 应总结：

```text
这次旅行整体节奏
住宿区域是否合适
用餐策略是否有效
老人孩子体力情况
哪些目的地/景点超预期
哪些安排不值得重复
计划生成阶段有哪些假设被验证或推翻
```

Family Memory 候选必须包含：

```text
规则内容
适用条件
证据事件
证据旅行
置信度
可能的反例
是否建议写入长期 Memory
```

Memory 不得自动长期生效。必须先作为 candidate，由用户确认后变为 accepted。

示例：

```text
候选 Memory：
父母同行时，如果上午安排中高强度景点，下午 15:00 后不宜继续安排高步行活动。

证据：
青岛 Day 2：上午老城区 + 八大关后，老人 15:30 反馈明显疲劳，取消小鱼山。

置信度：0.72
是否写入家庭旅行 Memory？
```

---

### 12.11 工具使用边界

Agent 可以直接用 LLM 完成：

```text
解释推荐理由
总结用户已提供内容
生成追问问题
组织自然语言表达
比较多个已知方案的优缺点
```

Agent 必须调用工具完成：

```text
创建/更新 Trip
生成或更新目的地推荐 artifact
生成 synthetic guide
合并攻略
解析地点标准名、坐标、地址、电话、营业时间、评分
估算路线距离和耗时
生成或修改 PlanArtifact
读取今日状态
记录 TravelEvent
生成 ReplanOption
应用计划调整
生成 DayLog / TripLog
写入 FamilyMemory candidate / accepted 状态
```

Agent 不得完成：

```text
伪造地点事实
伪造真实评论
假装已查实时交通/价格/天气/门票
绕过用户确认执行高影响计划变更
直接访问不在 allowlist 中的工具
把敏感家庭数据发送给无关工具
```

---

### 12.12 Agent 自主性与确认策略

Byway 需要智能主动，但不能失控。

Agent 可以自动执行：

```text
识别模式
整理用户输入
生成初版攻略研究
生成计划草案
记录低影响事件
生成未应用的调整建议
生成日志草稿
生成 Memory candidate
```

Agent 需要用户确认：

```text
最终选择目的地
进入旅行中状态
将计划标记为 confirmed / active
应用 replan 方案
取消或移动 4/5 星核心节点
改变第二天及之后的计划
写入 accepted FamilyMemory
删除旅行、计划或 Memory
```

Agent 不允许执行：

```text
预订或付款
自动购买门票
自动取消外部订单
自动向第三方平台发布内容
无授权分享家庭行程或酒店地址
```

---

### 12.13 多模型使用边界

Hermes 可以接入多个 LLM Provider，但 Byway 不应该让模型选择变成随机行为。

多模型使用原则：

```text
按任务角色选择模型，而不是每轮随意切换
高推理任务使用强模型
低风险摘要/分类可使用低成本模型
风险审查可以使用独立模型或独立 prompt role
多模型输出必须结构化保存为 GuideSource 或 ReviewResult
不同模型结论冲突时，不直接投票决定事实，而是标记 conflict
```

建议任务角色：

```text
destination_research_model：目的地候选研究
guide_classic_model：经典路线攻略
guide_family_model：老人孩子友好攻略
guide_lodging_model：住宿区域分析
guide_food_model：用餐区域分析
guide_risk_model：避坑和风险审查
planner_model：计划生成和解释
replan_model：旅行中重排建议
memory_model：日志总结和 Memory 候选提取
```

多模型共识只能提高“攻略判断置信度”，不能替代 PlaceFact / RouteFact / 实时工具。

---

### 12.14 Hermes Skill 要求

Byway Hermes Skill 必须明确写入以下内容：

```text
Byway 的身份和语气
Dream / Plan / Travel / Review 模式定义
模式切换规则
缺失信息追问规则
工具优先级
高德/PlaceFact 事实边界
多模型攻略研究方法
目的地推荐标准
计划生成原则
旅行中事件处理原则
确认策略
日志与 Memory 写入原则
安全与隐私边界
典型好/坏回答示例
```

Skill 中引用工具时，应使用 Byway MCP 暴露的工具名，而不是让 Agent 自行猜测实现方式。

---

### 12.15 Agent 质量评估标准

Agent 能力是否合格，不看回答是否“像攻略”，而看是否能可靠完成任务。

目的地推荐评估：

```text
是否考虑出发地、天数、同行人、季节和 Memory
是否能说明为什么推荐和为什么不推荐
是否避免只列热门城市
是否能在用户选择后进入 Plan Mode
```

计划生成评估：

```text
是否生成结构化 PlanArtifact
是否保留 meal / hotel / rest / transport anchor
是否避免明显走回头路
是否对低置信度地点提出确认
是否避免老人孩子同行时过度紧凑
```

旅行中调整评估：

```text
是否读取了当前状态
是否记录了 TravelEvent
是否给出取消/顺延/重排多方案
是否明确方案差异
是否保护吃饭和休息
是否请求确认后再应用高影响变更
```

Memory 评估：

```text
是否基于证据事件提取
是否有适用条件和置信度
是否避免过度泛化
是否经过用户确认才生效
```

Agent 失败时的要求：

```text
工具失败要说明影响，并给出降级方案
信息不确定要承认，不可编造
状态冲突要先澄清或读取最新状态
无法完成完整任务时，返回当前最有用的部分结果
```

---

### 12.16 典型 Agent 任务链路

#### 目的地推荐链路

```text
用户输入“不知道去哪”
→ classify_intent: Dream Mode
→ update_trip_brief
→ get_accepted_family_memory
→ generate_destination_candidates
→ research_destination_candidate
→ score_destination_candidates
→ 输出 3-5 个候选目的地
→ 用户选择目的地
→ select_destination
→ set_trip_phase: planning
```

#### 自动计划生成链路

```text
用户输入目的地和天数
→ update_trip_brief
→ generate_synthetic_guides
→ merge_guides
→ resolve_places_batch
→ estimate_route_matrix
→ generate_plan
→ verify_plan_geo
→ 输出计划卡片和需要确认的问题
```

#### 旅行中重排链路

```text
用户输入“老人累了，我们晚了 40 分钟”
→ classify_intent: Travel Mode
→ get_today_status
→ record_travel_event
→ replan_today
→ 输出 cancel / postpone / replan 三方案
→ 用户确认
→ apply_replan
→ 更新 Today Card
```

#### 旅行后 Memory 链路

```text
用户输入“今天结束”
→ generate_day_log
→ 提取当天 learnings
→ 旅行结束后 generate_trip_log
→ extract_family_memory_candidates
→ 用户确认
→ accept_family_memory
```

这些链路是 Byway Agent 的核心行为标准。Codex 实现时，应优先保证这些链路稳定，而不是增加更多 UI 或更多功能。

---

## 13. Byway MCP 工具清单

本节是最终产品能力清单。实际实现名、输入输出和副作用以 `docs/BYWAY_MCP_TOOL_CONTRACTS.md` 为准；若某个工具尚未在契约文档中定义，必须先补契约再实现。

### 13.1 Trip State Tools

```text
create_trip
update_trip_brief
get_trip_state
set_trip_phase
list_user_trips
```

### 13.2 Dream Tools

```text
generate_destination_candidates
research_destination_candidate
score_destination_candidates
select_destination
```

### 13.3 Guide Tools

```text
generate_synthetic_guides
ingest_user_guide_source
extract_guide_source
merge_guides
summarize_guide_consensus
summarize_guide_conflicts
```

### 13.4 Place / Geo Tools

```text
resolve_place
resolve_places_batch
get_place_detail
suggest_taxi_address
estimate_route
estimate_route_matrix
verify_plan_geo
create_amap_navigation_link
```

### 13.5 Plan Tools

```text
generate_plan
get_current_plan
update_plan_assumptions
update_plan_artifact
save_plan_version
```

### 13.6 Travel Runtime Tools

```text
start_travel_day
get_today_status
record_travel_event
replan_today
apply_replan
mark_node_completed
mark_node_skipped
```

### 13.7 Review / Memory Tools

```text
generate_day_log
generate_trip_log
extract_family_memory_candidates
accept_family_memory
get_accepted_family_memory
```

---

## 14. 数据模型

### 14.1 User / Family

```ts
type FamilyProfile = {
  id: string
  name: string
  homeCity: string
  travelers: Traveler[]
  acceptedMemoryIds: string[]
}
```

```ts
type Traveler = {
  id: string
  name: string
  role: "adult" | "elder" | "child" | "toddler"
  age?: number
  stamina?: "high" | "medium" | "low"
  constraints?: string[]
  interests?: string[]
}
```

---

### 14.2 Trip

```ts
type Trip = {
  id: string
  userId: string
  title: string
  phase: "intake" | "dream" | "destination_shortlist" | "planning" | "guide_research" | "place_fact_resolution" | "plan_confirmation" | "ready_to_travel" | "traveling" | "day_review" | "trip_review" | "memory_extraction" | "completed"
  destination?: string
  origin?: string
  dateRange?: DateRange
  days?: number
  travelers: Traveler[]
  pace?: "relaxed" | "normal" | "packed"
  arrivalInfo?: string
  departureInfo?: string
  hotelInfo?: string
  currentPlanId?: string
  createdAt: string
  updatedAt: string
}
```

---

### 14.3 Conversation Message

```ts
type ConversationMessage = {
  id: string
  tripId?: string
  role: "user" | "assistant" | "tool"
  content: string
  attachments?: Attachment[]
  artifactRefs?: string[]
  createdAt: string
}
```

---

### 14.4 Guide Source

```ts
type GuideSource = {
  id: string
  tripId: string
  type: "synthetic" | "markdown" | "xiaohongshu_link" | "screenshot" | "manual_note"
  provider?: string
  promptRole?: string
  title?: string
  rawContent?: string
  url?: string
  extractedData?: GuideExtraction
  createdAt: string
}
```

---

### 14.5 Canonical Guide

```ts
type CanonicalGuide = {
  id: string
  tripId: string
  lodgingAreaCandidates: LodgingAreaCandidate[]
  mealAreaCandidates: MealAreaCandidate[]
  places: CanonicalPlace[]
  routeIdeas: RouteIdea[]
  consensus: string[]
  conflicts: string[]
  risks: string[]
  assumptions: string[]
  sourceIds: string[]
  updatedAt: string
}
```

---

### 14.6 PlaceFact

```ts
type PlaceFact = {
  id: string
  inputName: string
  standardName: string
  aliases: string[]
  provider: "amap" | "google" | "manual"
  providerPlaceId?: string
  city?: string
  district?: string
  adcode?: string
  coordinate?: Coordinate
  address?: string
  displayAddress?: string
  taxiAddress?: string
  entranceCoordinate?: Coordinate
  navigationPoiId?: string
  phone?: string
  openingHoursToday?: string
  openingHoursWeek?: string
  rating?: number
  photos?: string[]
  businessArea?: string
  confidence: "verified" | "high" | "medium" | "low" | "unresolved"
  needsReview: boolean
  source: "amap_search" | "amap_id" | "geocode" | "manual"
  lastResolvedAt: string
}
```

---

### 14.7 Plan Artifact

```ts
type PlanArtifact = {
  id: string
  tripId: string
  version: number
  title: string
  status: "draft" | "confirmed" | "active" | "completed"
  assumptions: string[]
  lodgingRecommendation: string
  mealStrategy: string
  days: PlanDay[]
  risks: string[]
  placeFactIds: string[]
  routeFactIds: string[]
  generatedAt: string
}
```

```ts
type PlanDay = {
  id: string
  dayIndex: number
  date?: string
  title: string
  theme?: string
  nodes: PlanNode[]
  dayRisk?: string[]
}
```

```ts
type PlanNode = {
  id: string
  type: "activity" | "meal" | "hotel" | "transport" | "rest" | "buffer" | "optional"
  title: string
  placeFactId?: string
  priority?: 1 | 2 | 3 | 4 | 5
  plannedStart?: string
  plannedEnd?: string
  status: "pending" | "current" | "completed" | "skipped" | "postponed"
  notes?: string
  reason?: string
}
```

---

### 14.8 Travel Event

```ts
type TravelEvent = {
  id: string
  tripId: string
  dayIndex: number
  relatedNodeId?: string
  type: "late" | "current_overrun" | "child_tired" | "elder_tired" | "hungry" | "bad_weather" | "want_return_hotel" | "transport_problem" | "place_closed" | "queue_too_long" | "place_disappointing" | "place_exceeded_expectation" | "custom"
  severity: 1 | 2 | 3 | 4 | 5
  description: string
  timestamp: string
  replanOptionIds?: string[]
  appliedReplanOptionId?: string
}
```

---

### 14.9 Replan Option

```ts
type ReplanOption = {
  id: string
  tripId: string
  eventId: string
  type: "cancel" | "postpone" | "replan" | "no_change"
  title: string
  cancelledNodeIds: string[]
  postponedNodeIds: string[]
  updatedTimes: NodeTimeChange[]
  preservedAnchorIds: string[]
  explanation: string
  recommended: boolean
  requiresConfirmation: boolean
}
```

---

### 14.10 Logs and Memory

```ts
type DayLog = {
  id: string
  tripId: string
  dayIndex: number
  summary: string
  completedNodeIds: string[]
  skippedNodeIds: string[]
  eventIds: string[]
  highlights: string[]
  failures: string[]
  memoryCandidateIds: string[]
  createdAt: string
}
```

```ts
type FamilyMemory = {
  id: string
  userId: string
  category: "stamina" | "meal" | "transport" | "lodging" | "pace" | "interest" | "child" | "elder" | "weather" | "risk"
  rule: string
  condition?: string
  effect?: string
  confidence: number
  evidenceTripIds: string[]
  evidenceEventIds: string[]
  status: "candidate" | "accepted" | "rejected" | "archived"
  lastUpdated: string
}
```

---

## 15. 前端设计

### 15.1 页面结构

```text
/             当前对话和旅行
/trips        历史旅行列表
/settings     家庭信息、模型、地图配置
```

主页面结构：

```text
顶部状态栏
聊天消息流
Artifact 卡片区
底部输入区
```

---

### 15.2 输入区

支持：

```text
自然语言
快捷按钮
粘贴攻略
上传截图
发送链接
旅行中事件快速输入
```

快捷按钮：

```text
不知道去哪
我知道目的地
补充攻略
今天开始旅行
我们晚了
老人累了
孩子累了
大家饿了
想回酒店
今天结束
```

---

### 15.3 计划卡片

Plan Card 展示：

```text
住宿建议
用餐策略
Day 1 / Day 2 / Day 3
每天核心节点
风险提示
需要确认的问题
```

Today Card 展示：

```text
当前节点
下一个节点
后续节点
当前风险
调整入口
```

Replan Card 展示：

```text
方案 A / B / C
推荐方案
影响范围
应用按钮
```

---

## 16. 安全与隐私

### 16.1 Hermes 不应直接暴露公网

推荐：

```text
Mobile Web
↓
Byway Proxy API
↓
Hermes localhost / private network
```

Byway Proxy 负责：

```text
认证
限流
隐藏 Hermes API key
只暴露 Byway 所需接口
屏蔽不必要工具
记录审计日志
```

---

### 16.2 工具权限最小化

Byway Hermes Profile 应只暴露旅行所需工具。

不应在面向移动前端的生产环境中暴露：

```text
任意 terminal
任意文件系统读写
无约束浏览器自动化
无约束数据库访问
```

---

### 16.3 位置与家庭数据

敏感数据：

```text
家庭成员年龄
旅行时间
酒店位置
实时旅行状态
家庭偏好 Memory
```

要求：

```text
默认私有
不用于公开分享
可删除历史旅行
可删除 Memory
可导出数据
```

---

## 17. Agent 行为规则

### 17.1 必须使用工具的场景

Agent 在以下场景必须调用工具，而不是凭空回答：

```text
创建或更新旅行状态
生成目的地候选 artifact
生成计划 artifact
解析地点标准名称和坐标
查询地点详情
估算路线距离和时间
记录旅行事件
应用计划调整
生成日志
写入 Family Memory
```

---

### 17.2 必须说明假设

如果缺少关键信息，Agent 可以继续，但必须说清楚假设：

```text
我先假设第一天下午到、第四天下午走，按轻松节奏生成草案。
等你补充车次/航班后，我会重排第一天和最后一天。
```

---

### 17.3 不确定必须承认

例如：

```text
我没有确认这个景点的官方闭馆日，所以只把营业时间作为提示。
如果你决定把它作为核心景点，建议出行前再确认预约和开放状态。
```

---

### 17.4 解释要面向家庭，而不是算法

错误示范：

```text
根据路线矩阵和权重计算，Day 2 节点分配更优。
```

正确示范：

```text
我把八大关和小麦岛分开安排，是因为它们虽然都靠海，但中间移动会打断午餐和休息；6 人全家更适合把一天控制在一个主区域内。
```

---

## 18. 质量标准

### 18.1 好的目的地推荐

必须满足：

```text
候选不超过 5 个
每个候选有明确适合/不适合理由
能解释为什么不推荐热门但不适合的地方
考虑出发地、天数、老人孩子和季节
能被用户快速决策
```

---

### 18.2 好的旅行计划

必须满足：

```text
每天不明显过载
有 meal anchor
有 rest/buffer 意识
抵达日和离开日不硬塞景点
路线不明显走回头路
住宿区域和用餐区域合理
高优先级地点有 PlaceFact 支撑
风险说清楚
```

---

### 18.3 好的旅行中调整

必须满足：

```text
读取当前计划状态
保护 meal / hotel / transport / rest anchor
优先取消低优先级活动
核心活动变动前要确认
输出方案 diff
能解释取舍
能更新计划和日志
```

---

### 18.4 好的 Memory

必须满足：

```text
来自真实事件和日志
不是一句泛泛偏好
有适用场景
有证据
有置信度
用户确认后才生效
能影响下一次推荐和计划
```

---

## 19. 成功指标

产品最终成功不以功能数量衡量，而以旅行决策和执行体验衡量。

核心指标：

```text
用户从一句话到得到可用旅行草案的时间
用户手工编辑计划的次数
旅行中突发事件后得到可用调整建议的速度
每天用餐延误/崩盘次数减少
老人孩子疲劳事件减少
用户对目的地推荐的信任度
旅行后 Memory 被用户确认的比例
下一次计划是否明显更贴合家庭偏好
```

---

## 20. 最终产品定义

Byway 不是：

```text
攻略搜索 App
旅行计划表格工具
地图导航工具
OTA 平台
小红书替代品
```

Byway 是：

> **一个以 Agent 为核心的家庭旅行操作系统。它把目的地选择、攻略研究、地点事实、计划生成、旅行中调整、旅行后记忆沉淀连接成一个对话式闭环。**

最终体验应该是：

```text
用户：我想暑假从北京带父母和两个孩子玩 4 天，不知道去哪。

Byway：我推荐青岛、杭州、大连，并解释为什么西安这次略紧。

用户：那就青岛。

Byway：我已经研究了青岛 4 天家庭游，建议住五四广场/奥帆中心附近。Day 1 低强度抵达，Day 2 老城和八大关，Day 3 崂山轻量版或市区轻松版，Day 4 海边收尾返程。我还需要你确认第一天和第四天的交通时间。

旅行中用户：我们晚了 40 分钟，老人累了。

Byway：建议取消下一个 2 星景点，保留晚餐，把 4 星景点改为明天可选。是否应用？

旅行后 Byway：这次旅行说明父母同行时下午 3 点后不适合继续安排高步行强度景点，是否写入家庭旅行 Memory？
```

这就是“另辟蹊径 / Byway”的最终目标。

---

## 21. 参考依据

- Hermes Agent API Server 支持 OpenAI-compatible HTTP endpoint、Responses API、SSE 流式事件和工具进度展示。
- Hermes Skills 是按需加载的知识文档，适合承载 Byway 的旅行工作流说明。
- Hermes MCP 能连接外部工具服务器，适合接入 Byway MCP Server、高德地图工具、数据库和内部 API。
- Hermes 支持多种 AI Provider 和 custom endpoint，适合接入多模型。
- 高德开放平台提供 POI 搜索、地理/逆地理编码、路径规划、URI 调用以及 MCP Server 方向的集成能力。
- 高德 POI 搜索返回字段可包含名称、地址、经纬度、电话、入口经纬度、导航 POI ID、别名、商圈、评分、照片等；这些字段应按可选和置信度处理。
