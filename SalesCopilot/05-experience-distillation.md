# 经验萃取与检索设计

---

## 1. 经验定义

### 1.1 经验 ≠ 话术

话术是"说什么"，经验是"在什么情境下，做出什么判断，为什么有效"。

一条经验 = 一个**关键时刻**的结构化记录：

> 在特定情境下（用户处于什么状态 + 存在什么因素），销售做出了什么判断（而非其他选择），产生了什么效果（用户状态如何变化），背后的原理是什么（为什么这个判断在这个情境下是对的）。

### 1.2 经验的粒度：关键时刻

| 粒度 | 优点 | 缺点 | 是否采用 |
|------|------|------|---------|
| 整通对话 | 完整上下文 | 太粗，难以精准检索 | ✗ |
| 单轮对话 | 精确 | 太细，缺少前后文 | ✗ |
| **关键时刻** | 有情境、有判断、可复用 | 需要 LLM 识别 | **✓** |

**关键时刻的识别标准：**
- 对话中出现了可观察的"状态变化"（用户态度转变、新话题引入、异议被化解）
- 该变化与销售的某个**具体判断/动作**有关联
- 销售的判断是"非默认的"（不是按 SOP 发模板，而是基于情境做出的选择）
- 存在效果反馈（用户后续反应可观察）

### 1.3 经验的五种类型

| 类型 | 标签 | 说明 | 示例 |
|------|------|------|------|
| 意向推进 | `intent_advance` | 推动用户意向阶段向前流转 | S2→S3: 从"随便看看"引导到主动问方案 |
| 阻塞化解 | `blocker_resolve` | 有效化解用户的顾虑或异议 | 化解"太贵了"→ 重新定义价值参照系 |
| 信号捕捉 | `signal_capture` | 捕捉并放大用户的积极信号 | 用户提到"朋友在学" → 强化社交背书 |
| 节奏把控 | `pacing` | 在正确的时机做正确的事（或选择不做） | S2 阶段不急于报价，先建立需求认知 |
| 信息承接 | `context_leverage` | 利用用户已有背景信息（直播间行为、历史咨询、leads 数据等）增强沟通针对性 | 承接直播间信奥卖点，直接切入竞赛路径；利用用户历史试听记录针对性推荐 |

> **关于节奏把控的识别说明：** 节奏把控常表现为"选择不做某事"，在对话记录中缺乏直接文本证据，LLM 自动识别率较低。可自动识别的场景有限，主要是销售在对话中明确拉回话题（如用户追问价格时主动转移话题先谈需求）。建议初期以人工标注为主，LLM 辅助。

### 1.4 什么不是经验

- **SOP 动作**：按流程发的固定消息、模板问候
- **纯信息传递**：报价、课程时间、排课信息等事实性回答
- **无法复用的个例**：高度依赖特殊条件的一次性操作，其他销售无法复制。例如：用户是内部员工亲戚。
- **无效果反馈**：销售做了某件事，但用户无可观察的反应变化。"可观察"指对话文本中有证据的态度转变，如：从"我再想想"变为主动询问试听时间、回复变长变详细、开始主动分享孩子情况等。仅凭销售主观感觉"用户态度变好了"不算

---

## 2. 经验萃取 Prompt 设计

### 2.1 输入规范

采用对比萃取方案：每次输入一对配对案例（成功 + 失败），通过控制用户条件相似性来隔离销售行为差异的影响。

```yaml
输入字段:
  success_case:                   # 成功案例（成交）
    user_background:              # 用户背景摘要（L1+L2 标签汇总）
      grade: "五年级"
      city: "杭州"
      channel: "抖音-XX老师直播间"
      leads_score: 72
      completion_rate: "100%"
      # ...
    tag_extraction_result:        # 意向/话题/信号萃取结果（L3 标签）
      topics_factors: [...]
      positive_factors: [...]
      negative_factors: [...]
      intention: "S3"
      intention_reason: "..."
    dialogue:                     # 对话记录（完整，含角色标注和时间戳）
      - {role: "counselor", content: "...", time: "..."}
      - {role: "parent", content: "...", time: "..."}
    outcome:                      # 转化结果
      converted: true
      final_stage: "S4"
      initial_stage: "S2"

  contrast_case:                  # 对照案例（未成交，用户条件与 success_case 相似）
    user_background: {...}
    tag_extraction_result: {...}
    dialogue: [...]
    outcome:
      converted: false
      final_stage: "S2"
      initial_stage: "S2"

  matching_criteria:              # 配对依据，让 LLM 理解"为什么这两个可比"
    shared_features: ["同渠道", "同年级段", "相近leads评分"]
```

**配对规则：**

| 匹配维度     | 匹配规则          | 数据源      |
| -------- | ------------- | -------- |
| 渠道来源     | 相同            | 订单表      |
| 年级段      | 相同区间（低/中/高年级） | 用户画像     |
| Leads 评分 | 差值 < 10       | Leads 模型 |
| 时间窗口     | 同一学期          | 订单表      |

不需要完美配对，粗匹配已经比无对比显著更好。冷启动期用规则匹配，数据充足后可升级为 PSM（倾向得分匹配）。

**样本优先级 — "成功"不只看是否购买：**

经验萃取要找的是**销售判断真正起了作用**的对话，而非用户本来就会买的"自然成交"。

```
样本优先级矩阵:

                        最终购买    未购买
                       ┌─────────┬─────────┐
  leads 评分高          │ 自然成交 │  少见    │  ← 萃取优先级低（用户本来就会买）
  (初始条件好)           │         │         │
                       ├─────────┼─────────┤
  leads 评分中低        │ 逆袭成交 │  正常    │  ← 萃取优先级最高（销售贡献大）
  (初始条件一般)         │         │         │
                       └─────────┴─────────┘

实操: leads_score < P60 且最终购买 → "逆袭型"成交 → 优先作为 success_case
```

### 2.2 输出规范

完整的输出 JSON Schema（含 `comparability_check`、对照案例证据等）见 [06-prompts.md](06-prompts.md) Prompt 2-2 的输出格式定义。

**核心结构概览：**

| 字段 | 说明 |
|------|------|
| `comparability_check` | 配对可比性检查（条件差异标注） |
| `experiences[]` | 结构化经验记录（情境/判断/方法/效果/证据/检索标签） |
| `contrast_insights[]` | 两个案例的分歧节点分析 |
| `conversation_summary` | 整体对话模式概述 |

### 2.3 Prompt

完整的经验对比萃取 Prompt（含分析方法、经验类型定义、萃取原则、输出格式）见 [06-prompts.md](06-prompts.md) Prompt 2-2。

### 2.4 经验萃取的执行策略

**样本选择与配对：**

```
Step 1: 选取成功案例（success_case）
  优先级从高到低:
  1. "逆袭型"成交（leads_score < P60 且购买）— 销售贡献最大
  2. 有显著意向推进的对话（S1→S3、S2→S4 等跨阶段跃迁）— 关键时刻最明显
  3. 有明确异议化解的对话 — 阻塞化解经验最丰富
  4. 高转化销售（Top 20%）的成交对话 — 密度最高

  排除:
  - 用户条件极好、几乎无需销售努力的"自然成交"（leads_score > P80）
  - 对话极短、信息密度极低的沟通

Step 2: 为每个成功案例匹配对照案例（contrast_case）
  按配对规则（见 2.1）找到用户条件相似但未成交的对话
  若无法找到合理配对，该成功案例暂不萃取，等待后续配对
```

**执行节奏：**

```
初期（冷启动）:
  - 人工选取 50-100 对配对案例
  - 跑萃取 Prompt → 人工审核 → 修正 Prompt → 迭代
  - 目标: 打磨 Prompt 质量 + 积累种子经验库（~200 条经验）

稳定期:
  - 每周/每月批量处理新增成交对话，自动配对
  - 自动萃取 → 抽样审核（10-20%）
  - 持续积累经验库
```

---

## 3. 经验检索策略

### 3.1 Phase 1: 向量检索（MVP）

#### 整体流程

```
经验入库:
  经验记录 → 构建检索文本 → Embedding → Milvus

策略生成时检索:
  用户当前状态 → 构建查询文本 → Embedding → Top-K 检索 → 注入策略生成 Prompt
```

#### 检索方案：结构化过滤 + 向量排序

纯向量检索的问题：语义相似 ≠ 情境匹配。"用户很有兴趣但嫌贵"和"用户没兴趣但不在意价格"在文本语义上可能接近，但适用的经验完全不同。

**推荐方案：先用结构化字段缩小范围，再用向量排序。**

```
Step 1: 结构化过滤（缩小候选集）
  WHERE from_stage = 用户当前意向阶段
  AND experience_type IN (根据情况选择需要的经验类型)
  AND factors_addressed 与用户当前因素有交集

Step 2: 向量相似度排序（精排）
  query_text = 用户背景摘要 + 当前话题/因素描述
  experience_text = 经验的 context + judgment 部分
  ORDER BY cosine_similarity(query_embedding, experience_embedding)
  LIMIT K（建议 K=5-8）
```

#### 向量化文本构建

```
# 经验侧（入库时构建）
experience_embedding_text = f"""
意向阶段: {from_stage}→{to_stage}
用户特征: {applicable_segments}
涉及话题: {topics}
涉及因素: {factors_addressed}
情境描述: {context 描述}
判断概述: {judgment.what}
"""

# 查询侧（检索时构建）
query_embedding_text = f"""
意向阶段: {current_stage}→{target_stage}
用户特征: {user_profile 关键特征}
当前话题: {active_topics}
当前因素: 积极[{positive_factors}] 消极[{negative_factors}]
"""
```

#### Phase 1 的局限

| 能做到 | 做不到 |
|--------|--------|
| 找到情境相似的经验 | 推理"从 S1 到 S4 的最优路径" |
| 基于语义匹配检索 | 聚合多条经验的统计洞察 |
| 冷启动快、实现简单 | 区分"S2→S3 有效"和"S3→S4 有效"的经验 |
| | 回答"这个阻塞因素最有效的化解方式是什么" |

这些局限正是 GraphRAG 要解决的。

### 3.2 Phase 2: GraphRAG

#### 3.2.1 为什么需要 GraphRAG

向量检索回答的是"谁和我像"，GraphRAG 回答的是 **"我该往哪走、怎么走"**。

Sales Copilot 的核心问题本质上是一个**状态转移问题**：
- 当前状态 = (意向阶段, 因素组合, 用户画像)
- 目标状态 = 下一个意向阶段
- 策略 = 从当前状态到目标状态的"路径"

图谱天然适合建模这种结构：
- **节点** = 状态（意向阶段、因素、方法）
- **边** = 转移（从一个状态到另一个状态的路径）
- **边上的权重** = 经验证据数量（支撑该路径的经验条数，系统上线后可逐步补充效果追踪数据）

#### 3.2.2 图谱结构

```
节点类型:
  ┌─────────────────────────────────────────────────┐
  │  IntentStage     意向阶段节点                     │
  │  {name: "S0"|"S1"|"S2"|"S3"|"S4"}               │
  │                                                 │
  │  Factor          因素节点                        │
  │  {name: "价格敏感", polarity: "negative"}        │
  │  {name: "学习兴趣", polarity: "positive"}        │
  │                                                 │
  │  Approach         方法节点（从经验中抽象出的策略）   │
  │  {name: "先强化价值再谈价格",                      │
  │   description: "不急于让价，而是改变'贵'的参照系"}   │
  │                                                 │
  │  Experience       经验节点（具体的经验记录）        │
  │  {id, context, judgment, evidence, ...}         │
  └─────────────────────────────────────────────────┘

边类型:
  ┌─────────────────────────────────────────────────┐
  │  TRANSITION: IntentStage → IntentStage          │
  │  "从一个意向阶段推进到另一个"                       │
  │  properties: {total_count, success_count,       │
  │               success_rate}                     │
  │                                                 │
  │  BLOCKS: Factor → TRANSITION                    │
  │  "某个消极因素阻碍了某条转移路径"                    │
  │  properties: {frequency, block_strength}        │
  │                                                 │
  │  DRIVES: Factor → TRANSITION                    │
  │  "某个积极因素驱动了某条转移路径"                    │
  │  properties: {frequency, drive_strength}        │
  │                                                 │
  │  RESOLVES: Approach → (Factor, TRANSITION)      │
  │  "某种方法在某条路径上有效化解了某个阻塞因素"          │
  │  properties: {success_rate, example_count}      │
  │                                                 │
  │  LEVERAGES: Approach → (Factor, TRANSITION)     │
  │  "某种方法在某条路径上有效利用了某个驱动因素"          │
  │  properties: {success_rate, example_count}      │
  │                                                 │
  │  EVIDENCE_OF: Experience → Approach             │
  │  "某条具体经验是某种方法的证据/实例"                 │
  └─────────────────────────────────────────────────┘
```

**图谱可视化示例：**

```
          ┌──── [价格敏感] ──BLOCKS───┐
          │                          ↓
  [S2] ──TRANSITION──→ [S3] ──TRANSITION──→ [S4]
          ↑                          ↑
          │                          │
   [学习兴趣] ──DRIVES──┘    [主动询价] ──DRIVES──┘
          ↑
   [先挖需求再推方案] ──LEVERAGES──┘
          ↑
   [经验#042] ──EVIDENCE_OF──┘

   [先强化价值再谈价格] ──RESOLVES──→ [价格敏感]（在 S2→S3 路径上）
          ↑
   [经验#017] ──EVIDENCE_OF──┘
   [经验#089] ──EVIDENCE_OF──┘
```

#### 3.2.3 图谱构建（从经验到图谱）

每条萃取出的经验自动填充图谱：

```
For each experience:

  1. TRANSITION 边
     MERGE (from:Stage {name: exp.from_stage})-[t:TRANSITION]->(to:Stage {name: exp.to_stage})
     t.total_count += 1
     if exp.outcome == success: t.success_count += 1
     t.success_rate = t.success_count / t.total_count

  2. BLOCKS / DRIVES 边
     For each factor in exp.factors_addressed:
       if factor.polarity == "negative":
         MERGE (f:Factor {name: factor})-[:BLOCKS]->(t)
       if factor.polarity == "positive":
         MERGE (f:Factor {name: factor})-[:DRIVES]->(t)

  3. APPROACH 节点
     MERGE (a:Approach {name: exp.execution.approach})
     // approach 名称需要归一化，相似的方法应合并为同一节点
     // 初期可人工审核，后期可用 LLM 自动归并

  4. RESOLVES / LEVERAGES 边
     if exp.type == "blocker_resolve":
       MERGE (a)-[:RESOLVES {for_transition: t}]->(blocked_factor)
     if exp.type == "signal_capture":
       MERGE (a)-[:LEVERAGES {for_transition: t}]->(driven_factor)

  5. EVIDENCE_OF 边
     CREATE (exp_node:Experience {...})-[:EVIDENCE_OF]->(a)
```

**Approach 归一化（关键问题）：**

不同经验中的方法描述可能措辞不同但本质相同。需要归一化为同一个 Approach 节点，
否则图谱中会出现大量碎片节点，稀释统计信号。

```
示例 — 以下三条经验的 approach 是同一件事:
  经验A: "先聊孩子的学习情况，再自然过渡到课程推荐"
  经验B: "不急于介绍课程，先了解孩子学习背景和兴趣"
  经验C: "从孩子情况切入，建立需求认知后再推方案"
  → 归一化为: "先挖需求再推方案"
  → 图谱中: 1 个 Approach 节点 + 3 条 EVIDENCE_OF 边（统计更有力）

如果不做归一化:
  → 3 个孤立的 Approach 节点，每个只挂 1 条经验（统计无意义）
```

```
方案 1（初期，推荐）: 人工审核 + 合并
  - 经验量 < 300 条时完全可控
  - 好处: 同时建立 Approach 的命名规范

方案 2（成熟期）: LLM 自动归并
  输入: 新经验的 approach 描述 + 已有 Approach 节点列表
  Prompt: "以下新经验的方法描述，是否与已有某个 Approach 等价？
           如果是，返回匹配的节点；如果不是，建议创建新节点的规范名称。"
  输出: 匹配已有节点 或 创建新节点
```

#### 3.2.4 查询模式

**查询 1：直接路径 — "从当前阶段推进到下一阶段，怎么做？"**

```cypher
-- 输入: 用户在 S2，消极因素=[价格敏感]，积极因素=[学习兴趣]
-- 目标: 推进到 S3

-- 找到 S2→S3 的转移路径
MATCH (s2:Stage {name: 'S2'})-[t:TRANSITION]->(s3:Stage {name: 'S3'})

-- 找到阻塞该路径的因素中，与用户当前消极因素匹配的
MATCH (blocker:Factor {name: '价格敏感'})-[:BLOCKS]->(t)

-- 找到能化解该阻塞的方法
MATCH (resolve_approach:Approach)-[r:RESOLVES]->(blocker)
WHERE r.for_transition = t

-- 找到能利用用户积极因素的方法
MATCH (driver:Factor {name: '学习兴趣'})-[:DRIVES]->(t)
MATCH (leverage_approach:Approach)-[l:LEVERAGES]->(driver)

-- 找到相关经验作为证据
MATCH (exp:Experience)-[:EVIDENCE_OF]->(resolve_approach)

RETURN resolve_approach, leverage_approach, exp
ORDER BY r.evidence_count DESC
```

**查询 2：多步路径 — "从 S1 到 S4 的最优路径是什么？"**

```cypher
-- 找到 S1 到 S4 的所有路径（最多 3 步）
MATCH path = (s1:Stage {name: 'S1'})-[:TRANSITION*1..3]->(s4:Stage {name: 'S4'})

-- 按经验证据数量评估路径可靠性
WITH path,
     reduce(cnt = 0, t IN relationships(path) | cnt + t.evidence_count) AS path_evidence

RETURN path, path_evidence
ORDER BY path_evidence DESC
LIMIT 5

-- 示例结果:
-- Path 1: S1 → S2 → S3 → S4  (evidence: 87)  ← 最常见的渐进路径，证据最充分
-- Path 2: S1 → S3 → S4        (evidence: 23)  ← 跳跃式，证据较少但存在
-- Path 3: S1 → S2 → S4        (evidence: 15)  ← 中间路径，证据有限
```

**查询 3：阻塞因素专项 — "这个阻塞因素怎么化解？"**

```cypher
-- 输入: 用户在 S3，被"决策权模糊"阻塞
MATCH (factor:Factor {name: '决策权模糊'})-[:BLOCKS]->(t:TRANSITION)
WHERE startNode(t).name = 'S3'

MATCH (approach:Approach)-[r:RESOLVES]->(factor)
MATCH (exp:Experience)-[:EVIDENCE_OF]->(approach)

RETURN approach.name, approach.description, r.evidence_count, 
       collect(exp.evidence) AS examples
ORDER BY r.evidence_count DESC
```

**查询 4：用户因素组合匹配 — "和我情况类似的成功案例走的什么路？"**

```cypher
-- 输入: 用户在 S2，因素=[价格敏感, 学习兴趣, 时间冲突]
-- 找到涉及相同因素组合的经验
MATCH (exp:Experience)
WHERE exp.context.intent_stage = 'S2'
  AND ANY(f IN exp.retrieval_tags.factors_addressed 
          WHERE f IN ['价格敏感', '学习兴趣', '时间冲突'])

-- 按因素匹配度排序
WITH exp, 
     size([f IN exp.retrieval_tags.factors_addressed 
           WHERE f IN ['价格敏感', '学习兴趣', '时间冲突']]) AS match_count
ORDER BY match_count DESC
LIMIT 10

-- 溯源到这些经验关联的 Approach
MATCH (exp)-[:EVIDENCE_OF]->(approach:Approach)
RETURN approach, collect(exp) AS supporting_experiences
```

#### 3.2.5 GraphRAG 与策略生成的整合

GraphRAG 的完整策略生成流程（含图谱查询步骤、Context 组装、降级策略）见 [03-architecture.md](03-architecture.md) Agent 3 章节。

---

## 4. 端到端流程总览

```
═══════════════════════════════════════════════════════════════
                      离线：经验积累
═══════════════════════════════════════════════════════════════

配对样本（成功案例 + 条件相似的失败案例）
  │
  ├─ 分别跑标签提取 Agent（已有）→ topics_factors + intention
  │
  └─ 对比萃取 Agent（本文档）→ 结构化经验记录 + contrast_insights
       │
       ├─ Phase 1: → Embedding → Milvus（向量索引）
       │
       └─ Phase 2: → 图谱构建 → Neo4j/等（图索引）
                      ├─ TRANSITION 边（阶段转移统计）
                      ├─ BLOCKS/DRIVES 边（因素关系）
                      ├─ RESOLVES/LEVERAGES 边（方法效果）
                      └─ EVIDENCE_OF 边（经验证据）

═══════════════════════════════════════════════════════════════
                      实时：策略生成
═══════════════════════════════════════════════════════════════

销售请求策略（某个用户的策略卡片）
  │
  ├─ 获取用户标签（Agent 2 实时/缓存）
  │     → 当前意向阶段 + 话题 + 因素
  │
  ├─ 经验检索
  │     Phase 1: 结构化过滤 + 向量排序 → Top-K 经验
  │     Phase 2: 图谱查询 → 最优路径 + 推荐方法 + 证据经验
  │
  └─ 策略生成 Agent（Agent 3）
        输入: 用户标签 + 检索结果
        输出: 策略卡片
```

---

## 5. 已确认 / 待讨论

### 已确认

| # | 问题 | 结论 |
|---|------|------|
| 1 | **对比萃取作为默认方案** | ✅ 直接采用对比萃取方案（非增强版可选项）。每次萃取输入一对配对案例（成功+失败）。"成功"定义采用"逆袭型成交"（leads < P60 且购买），配对用规则粗匹配（同渠道+同年级段+leads差<10+同学期）|
| 2 | **经验人工审核** | ✅ 初期安排人工审核质量 |
| 3 | **Approach 归一化** | ✅ 初期人工合并（经验量 < 300 可控），后期 LLM 自动归并 |
| 4 | **图数据库选型** | ✅ 自建 Neo4j，可先本地跑通再上服务器 |
| 5 | **样本量** | ✅ 近千条已购买用户，样本充足。种子经验 ≥ 200 条后启动图谱构建 |

### 待讨论

| # | 问题 | 影响 | 建议 |
|---|------|------|------|
| 6 | **经验的时效性**：课程产品更新后旧经验是否失效？ | 经验库的生命周期管理 | 经验记录加入时间戳，产品大版本更新时标记旧经验 |
