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

#### 3.2.3 图谱构建：从经验到图谱的详细过程

##### 总览：入库数据流

```
经验萃取 Prompt 2-2 输出（JSON）
  │
  ├─ Step 3: 质量评审（Prompt 2-3）→ 过滤掉 reject 的经验
  │
  ├─ Step 4: Approach 归一化（Prompt 2-4 / 人工）→ 确定 approach_id
  │
  └─ Step 5: 入库
       │
       ├─ 5a. 字段提取与转换（JSON → 图操作所需的结构化字段）
       │
       ├─ 5b. 节点写入（IntentStage / Factor / Approach / Experience）
       │
       ├─ 5c. 边写入（TRANSITION / BLOCKS / DRIVES / RESOLVES / LEVERAGES / EVIDENCE_OF）
       │
       ├─ 5d. 统计量更新（边上的 count / rate 等聚合属性）
       │
       └─ 5e. 入库校验（完整性 + 一致性检查）
```

##### Step 5a: 字段提取与转换

从经验 JSON 中提取图操作所需的字段。核心映射关系：

```
经验 JSON 字段                        →  图谱操作所需字段
─────────────────────────────────────────────────────────────────
retrieval_tags.from_stage             →  TRANSITION 边的起始节点
retrieval_tags.to_stage               →  TRANSITION 边的目标节点
context.negative_factors[]            →  Factor 节点（polarity=negative）→ BLOCKS 边
context.positive_factors[]            →  Factor 节点（polarity=positive）→ DRIVES 边
retrieval_tags.factors_addressed[]    →  RESOLVES / LEVERAGES 边关联的因素
execution.approach                    →  Approach 节点（经 Prompt 2-4 归一化）
experience_type                       →  决定 RESOLVES 还是 LEVERAGES 边
全部字段                              →  Experience 节点（完整存储）
```

**转换逻辑（伪代码）：**

```python
def transform_experience_for_graph(exp_json, normalization_result):
    """
    将经验 JSON + Approach 归一化结果，转换为图操作指令列表。
    
    参数:
      exp_json: Prompt 2-2 产出的单条经验（已通过 Prompt 2-3 审核）
      normalization_result: Prompt 2-4 的输出（matched/new + approach_id）
    """
    
    ops = []  # 图操作指令列表
    
    # --- 1. 确定 from/to stage ---
    from_stage = exp_json["retrieval_tags"]["from_stage"]  # e.g. "S2"
    to_stage = exp_json["retrieval_tags"]["to_stage"]      # e.g. "S3"
    
    # 边界情况: stage_transition 为 null（如 pacing 类型经验，可能没有阶段跃迁）
    has_transition = (from_stage is not None and to_stage is not None 
                      and from_stage != to_stage)
    
    # --- 2. 确定因素及其极性 ---
    negative_factors = exp_json["context"]["negative_factors"]  # ["价格敏感", "时间冲突"]
    positive_factors = exp_json["context"]["positive_factors"]  # ["学习兴趣"]
    factors_addressed = exp_json["retrieval_tags"]["factors_addressed"]  # ["价格敏感", "学习兴趣"]
    
    # 为每个 addressed factor 确定极性
    addressed_with_polarity = []
    for f in factors_addressed:
        if f in negative_factors:
            addressed_with_polarity.append({"name": f, "polarity": "negative"})
        elif f in positive_factors:
            addressed_with_polarity.append({"name": f, "polarity": "positive"})
        else:
            # 因素既不在 positive 也不在 negative 列表中
            # → 标记为 unknown，入库后需人工确认
            addressed_with_polarity.append({"name": f, "polarity": "unknown"})
    
    # --- 3. 确定 Approach ---
    if normalization_result["match_result"] == "matched":
        approach_id = normalization_result["matched_approach_id"]
        approach_name = None  # 使用已有节点，不更新 name
    else:
        approach_id = generate_new_id()  # e.g. "APR_021"
        approach_name = normalization_result["suggested_name"]
        approach_desc = normalization_result["suggested_description"]
    
    # --- 4. 确定边类型映射 ---
    exp_type = exp_json["experience_type"]
    # 经验类型 → 边类型的映射规则（见下文详述）
    
    return {
        "from_stage": from_stage,
        "to_stage": to_stage,
        "has_transition": has_transition,
        "addressed_factors": addressed_with_polarity,
        "approach_id": approach_id,
        "approach_name": approach_name,
        "experience_type": exp_type,
        "experience_data": exp_json,  # 完整 JSON，存入 Experience 节点
    }
```

##### Step 5b: 节点写入

**写入顺序很重要**：先写节点，再写边。节点间用 MERGE（幂等），Experience 用 CREATE（每条唯一）。

```
写入顺序:
  ① IntentStage 节点（最多 5 个，初始化时一次性建好）
  ② Factor 节点（MERGE，按名称去重）
  ③ Approach 节点（MERGE 或 CREATE，取决于归一化结果）
  ④ Experience 节点（CREATE，每条经验唯一）
  ⑤ 所有边（依赖上述节点已存在）
```

**① IntentStage 节点（一次性初始化）：**

```cypher
-- 系统初始化时执行一次，后续不再变动
MERGE (s:IntentStage {name: 'S0'}) SET s.label = '暂缓/流失'
MERGE (s:IntentStage {name: 'S1'}) SET s.label = '沉默用户'
MERGE (s:IntentStage {name: 'S2'}) SET s.label = '需求模糊'
MERGE (s:IntentStage {name: 'S3'}) SET s.label = '需求明确'
MERGE (s:IntentStage {name: 'S4'}) SET s.label = '强购买意愿'
```

**② Factor 节点：**

```cypher
-- 对经验中涉及的每个因素，MERGE 确保不重复
MERGE (f:Factor {name: $factor_name})
ON CREATE SET f.polarity = $polarity,
              f.created_at = datetime(),
              f.mention_count = 1
ON MATCH SET f.mention_count = f.mention_count + 1

-- 示例:
-- MERGE (f:Factor {name: '价格敏感'}) ON CREATE SET f.polarity = 'negative', ...
-- MERGE (f:Factor {name: '学习兴趣'}) ON CREATE SET f.polarity = 'positive', ...
```

> **因素归一化问题：** 与 Approach 类似，因素名称也存在同义表述的问题（如"嫌贵"vs"价格敏感"）。
> 但因素来自标签体系（Prompt 1-1 的因素标签清单），天然受约束，碎片化风险较低。
> 若后续标签体系扩展导致同义因素出现，需要建立因素同义词映射表。

**③ Approach 节点：**

```cypher
-- Case A: 归一化结果 = matched（匹配到已有节点）
-- 不创建新节点，仅更新经验计数
MATCH (a:Approach {id: $approach_id})
SET a.example_count = a.example_count + 1,
    a.updated_at = datetime()

-- Case B: 归一化结果 = new（创建新节点）
CREATE (a:Approach {
  id: $approach_id,
  name: $approach_name,
  description: $approach_description,
  example_count: 1,
  created_at: datetime(),
  updated_at: datetime()
})
```

**④ Experience 节点：**

```cypher
CREATE (e:Experience {
  id: $experience_id,                         -- 全局唯一 ID，如 "EXP_00042"
  experience_type: $exp_type,                  -- "blocker_resolve" 等
  extraction_method: $method,                  -- "comparative" 或 "single"
  source_case_id: $success_case_id,            -- 关联的成功案例 ID
  contrast_case_id: $contrast_case_id,         -- 关联的失败案例 ID（单对话萃取为 null）
  
  -- 情境（扁平化存储关键字段，便于查询过滤）
  context_description: $context.description,
  context_intent_stage: $context.intent_stage,
  context_topics: $context.active_topics,       -- 字符串数组
  context_positive_factors: $context.positive_factors,
  context_negative_factors: $context.negative_factors,
  context_user_segments: $context.user_profile_relevance,
  
  -- 判断
  judgment_what: $judgment.what,
  judgment_why: $judgment.why_effective,
  judgment_alternative: $judgment.alternative_avoided,
  
  -- 执行
  approach_raw: $execution.approach,           -- 原始 approach 描述（归一化前）
  approach_id: $approach_id,                   -- 归一化后的 Approach 节点 ID
  topic_transition: $execution.topic_transition,
  key_expression: $execution.key_expression_pattern,
  
  -- 效果
  effect_response_change: $effect.user_response_change,
  effect_stage_transition: $effect.stage_transition,
  effect_factor_change: $effect.factor_change,
  
  -- 证据（完整保留，用于策略生成时展示给销售）
  evidence_before: $evidence.before,
  evidence_action: $evidence.action,
  evidence_after: $evidence.after,
  evidence_contrast_action: $evidence.contrast_action,
  evidence_contrast_after: $evidence.contrast_after,
  
  -- 检索标签（冗余存储，加速结构化过滤）
  from_stage: $retrieval_tags.from_stage,
  to_stage: $retrieval_tags.to_stage,
  factors_addressed: $retrieval_tags.factors_addressed,
  topics: $retrieval_tags.topics,
  applicable_segments: $retrieval_tags.applicable_segments,
  
  -- 元数据
  created_at: datetime(),
  quality_score: $quality_score,               -- 来自 Prompt 2-3 的综合评分
  batch_id: $batch_id                          -- 本次蒸馏批次 ID，便于追溯
})
```

##### Step 5c: 边写入

**经验类型与边类型的映射规则：**

```
┌─────────────────────┬──────────────────────────────────────────────────────┐
│ 经验类型             │ 产生的边                                              │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ intent_advance      │ TRANSITION + DRIVES（利用积极因素推进）                  │
│                     │ + LEVERAGES（approach 利用了哪些驱动因素）               │
│                     │                                                      │
│ blocker_resolve     │ TRANSITION + BLOCKS（消极因素阻碍了该路径）              │
│                     │ + RESOLVES（approach 化解了哪些阻塞因素）               │
│                     │                                                      │
│ signal_capture      │ TRANSITION + DRIVES（捕捉并放大了积极信号）              │
│                     │ + LEVERAGES（approach 利用了哪些驱动因素）               │
│                     │                                                      │
│ pacing              │ TRANSITION（如有阶段变化）                              │
│                     │ 注意: pacing 可能无阶段跃迁（"选择不做"的经验）           │
│                     │ 此时不产生 TRANSITION 边，只产生 EVIDENCE_OF             │
│                     │                                                      │
│ context_leverage    │ TRANSITION + DRIVES / BLOCKS（取决于利用的信息极性）      │
│                     │ + LEVERAGES（approach 如何利用上下文信息）               │
└─────────────────────┴──────────────────────────────────────────────────────┘

所有类型都产生: EVIDENCE_OF（experience → approach）
```

**① TRANSITION 边：**

```cypher
-- 仅在 has_transition = true 时写入
-- MERGE 确保同一 from→to 只有一条 TRANSITION 边，计数累加
MATCH (from:IntentStage {name: $from_stage})
MATCH (to:IntentStage {name: $to_stage})
MERGE (from)-[t:TRANSITION]->(to)
ON CREATE SET t.total_count = 1,
              t.success_count = CASE WHEN $is_success THEN 1 ELSE 0 END,
              t.created_at = datetime()
ON MATCH SET t.total_count = t.total_count + 1,
             t.success_count = t.success_count + CASE WHEN $is_success THEN 1 ELSE 0 END
-- success_rate 作为派生字段，在统计更新阶段计算（Step 5d）
```

> **关于 `$is_success`：** 经验萃取的输入是成功案例，因此经验本身记录的是"成功路径"。
> `is_success = true` 表示该经验来自成功案例的行为。
> 后续如果需要统计"尝试过但失败"的路径（来自对照案例），
> 可扩展为从 `contrast_insights` 中提取失败路径信息，此时 `is_success = false`。

**② BLOCKS 边（消极因素 → TRANSITION）：**

```cypher
-- 对经验中的每个消极因素，关联到对应的 TRANSITION 边
-- 语义：该消极因素在此路径上构成了阻碍（但被成功化解了）
MATCH (f:Factor {name: $negative_factor_name})
MATCH (from:IntentStage {name: $from_stage})-[t:TRANSITION]->(to:IntentStage {name: $to_stage})
MERGE (f)-[b:BLOCKS]->(t)
ON CREATE SET b.frequency = 1,
              b.block_strength = 'medium',   -- 初始默认值，后续由统计更新
              b.created_at = datetime()
ON MATCH SET b.frequency = b.frequency + 1

-- 注意: BLOCKS 边连接 Factor 到 TRANSITION 边
-- Neo4j 不支持"边指向边"，需要建模调整（见下方"建模适配"说明）
```

**③ DRIVES 边（积极因素 → TRANSITION）：**

```cypher
-- 对经验中的每个积极因素，关联到对应的 TRANSITION 边
MATCH (f:Factor {name: $positive_factor_name})
MATCH (from:IntentStage {name: $from_stage})-[t:TRANSITION]->(to:IntentStage {name: $to_stage})
MERGE (f)-[d:DRIVES]->(t)
ON CREATE SET d.frequency = 1,
              d.drive_strength = 'medium',
              d.created_at = datetime()
ON MATCH SET d.frequency = d.frequency + 1
```

**④ RESOLVES 边（Approach → Factor，在特定 TRANSITION 上下文中）：**

```cypher
-- 仅 blocker_resolve 类型的经验产生此边
-- 语义：该 Approach 在 from→to 路径上成功化解了该阻塞因素
MATCH (a:Approach {id: $approach_id})
MATCH (f:Factor {name: $resolved_factor_name})
MERGE (a)-[r:RESOLVES]->(f)
ON CREATE SET r.for_from_stage = $from_stage,
              r.for_to_stage = $to_stage,
              r.success_count = 1,
              r.example_count = 1,
              r.created_at = datetime()
ON MATCH SET r.example_count = r.example_count + 1,
             r.success_count = r.success_count + 1
-- success_rate 在 Step 5d 计算

-- 确定 resolved_factor: 从 factors_addressed 中筛选 polarity=negative 的因素
-- 一条 blocker_resolve 经验可能化解了多个消极因素，每个都建一条 RESOLVES 边
```

**⑤ LEVERAGES 边（Approach → Factor，在特定 TRANSITION 上下文中）：**

```cypher
-- intent_advance / signal_capture / context_leverage 类型产生此边
-- 语义：该 Approach 在 from→to 路径上有效利用了该驱动因素
MATCH (a:Approach {id: $approach_id})
MATCH (f:Factor {name: $leveraged_factor_name})
MERGE (a)-[l:LEVERAGES]->(f)
ON CREATE SET l.for_from_stage = $from_stage,
              l.for_to_stage = $to_stage,
              l.success_count = 1,
              l.example_count = 1,
              l.created_at = datetime()
ON MATCH SET l.example_count = l.example_count + 1,
             l.success_count = l.success_count + 1

-- 确定 leveraged_factor: 从 factors_addressed 中筛选 polarity=positive 的因素
```

**⑥ EVIDENCE_OF 边（Experience → Approach）：**

```cypher
-- 所有类型的经验都产生此边
MATCH (e:Experience {id: $experience_id})
MATCH (a:Approach {id: $approach_id})
CREATE (e)-[:EVIDENCE_OF {
  created_at: datetime(),
  experience_type: $exp_type
}]->(a)
```

##### Neo4j 建模适配："边指向边"的处理

上述设计中 BLOCKS/DRIVES 语义上是 **Factor → TRANSITION 边**，但 Neo4j 不支持"边指向边"。需要适配：

```
方案 A（推荐）: 将 TRANSITION 路径信息冗余到 BLOCKS/DRIVES 边属性中
  (Factor)-[BLOCKS {from_stage, to_stage}]->(IntentStage)
  
  即: BLOCKS/DRIVES 边直接连接 Factor → 目标 IntentStage 节点
  用 from_stage 属性标注"在哪条路径上"阻碍/驱动
  
  优点: 简单，查询方便
  缺点: from_stage 信息冗余（但属性级冗余成本极低）

方案 B: 引入 Transition 中间节点
  将 TRANSITION 边拆为一个节点:
  (S2)-[:FROM]->(T_S2_S3:Transition)-[:TO]->(S3)
  (Factor)-[:BLOCKS]->(T_S2_S3)
  
  优点: 语义最干净
  缺点: 多一层节点，查询语句更复杂
```

**采用方案 A**。调整后的实际 Cypher：

```cypher
-- BLOCKS 边: Factor → 目标 IntentStage，标注路径上下文
MATCH (f:Factor {name: $factor_name})
MATCH (to:IntentStage {name: $to_stage})
MERGE (f)-[b:BLOCKS {from_stage: $from_stage}]->(to)
ON CREATE SET b.frequency = 1, b.created_at = datetime()
ON MATCH SET b.frequency = b.frequency + 1

-- DRIVES 边: Factor → 目标 IntentStage，标注路径上下文
MATCH (f:Factor {name: $factor_name})
MATCH (to:IntentStage {name: $to_stage})
MERGE (f)-[d:DRIVES {from_stage: $from_stage}]->(to)
ON CREATE SET d.frequency = 1, d.created_at = datetime()
ON MATCH SET d.frequency = d.frequency + 1

-- 查询时按路径过滤:
-- MATCH (f:Factor)-[b:BLOCKS {from_stage: 'S2'}]->(to:IntentStage {name: 'S3'})
```

同理，RESOLVES/LEVERAGES 边也用 `for_from_stage` + `for_to_stage` 属性记录路径上下文。

##### Step 5d: 统计量更新

每批经验入库后，更新派生统计字段：

```cypher
-- 更新 TRANSITION 边的 success_rate
MATCH ()-[t:TRANSITION]->()
SET t.success_rate = toFloat(t.success_count) / t.total_count

-- 更新 RESOLVES 边的 success_rate
MATCH ()-[r:RESOLVES]->()
WHERE r.example_count > 0
SET r.success_rate = toFloat(r.success_count) / r.example_count

-- 更新 LEVERAGES 边的 success_rate
MATCH ()-[l:LEVERAGES]->()
WHERE l.example_count > 0
SET l.success_rate = toFloat(l.success_count) / l.example_count

-- 更新 BLOCKS/DRIVES 的 strength（基于频率的相对强度）
-- block_strength = 该因素在该路径上出现的频率 / 该路径的总经验数
MATCH (f:Factor)-[b:BLOCKS {from_stage: from_s}]->(to:IntentStage)
MATCH (from2:IntentStage {name: from_s})-[t:TRANSITION]->(to)
SET b.block_strength = toFloat(b.frequency) / t.total_count
```

##### Step 5e: 入库校验

每条经验入库后，运行校验确保数据完整性：

```
校验清单:
  ✓ Experience 节点已创建（通过 id 查询确认）
  ✓ EVIDENCE_OF 边已创建（Experience → Approach 连通）
  ✓ 如果 has_transition=true:
    ✓ TRANSITION 边存在且 total_count 已递增
    ✓ 至少一条 BLOCKS 或 DRIVES 边已创建（经验必须涉及因素）
    ✓ 根据 experience_type，RESOLVES 或 LEVERAGES 边已创建
  ✓ Approach 节点的 example_count 与 EVIDENCE_OF 入边数一致
  ✓ 无 polarity=unknown 的因素（如有，标记待人工确认）
```

```cypher
-- 校验: 某条经验入库是否完整
MATCH (e:Experience {id: $experience_id})
OPTIONAL MATCH (e)-[:EVIDENCE_OF]->(a:Approach)
RETURN e.id, 
       a.id AS approach_id,
       EXISTS((e)-[:EVIDENCE_OF]->()) AS has_evidence_link
```

##### 完整入库流程（单条经验）

```
┌─────────────────────────────────────────────────────────────────┐
│  输入: 经验 JSON（已审核通过）+ Approach 归一化结果                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                     ┌─────▼─────┐
                     │ 5a 字段提取 │  → 确定 from/to stage, factors 极性,
                     │    与转换  │     approach_id, 边类型映射
                     └─────┬─────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
   ┌──────────┐     ┌──────────┐     ┌──────────────┐
   │ MERGE    │     │ MERGE/   │     │ CREATE       │
   │ Factor   │     │ CREATE   │     │ Experience   │
   │ 节点      │     │ Approach │     │ 节点          │
   └─────┬────┘     └────┬─────┘     └──────┬───────┘
         │               │                  │
         └───────────────┼──────────────────┘
                         │  （节点全部就绪）
                         ▼
         ┌───────────────────────────────┐
         │         5c 边写入              │
         │                               │
         │  ① TRANSITION（如有跃迁）       │
         │  ② BLOCKS / DRIVES            │
         │  ③ RESOLVES / LEVERAGES       │
         │  ④ EVIDENCE_OF                │
         └───────────────┬───────────────┘
                         │
                   ┌─────▼─────┐
                   │ 5d 统计更新 │  → success_rate, block_strength 等
                   └─────┬─────┘
                         │
                   ┌─────▼─────┐
                   │ 5e 入库校验 │  → 完整性检查
                   └─────┬─────┘
                         │
                   ┌─────▼─────┐
                   │  同步写入   │  → Milvus embedding（向量索引）
                   │  向量库    │
                   └───────────┘
```

##### 批量入库策略

```
单条入库 vs 批量入库:

  冷启动期（经验 < 300 条）:
    逐条入库 + 逐条校验
    原因: 数据量小，优先保证质量，便于排查问题

  生产期（定期批量蒸馏）:
    批量入库（一批 50-100 条经验作为一个事务）
    流程:
      1. 开启 Neo4j 事务
      2. 批量写入所有节点
      3. 批量写入所有边
      4. 批量统计更新
      5. 批量校验
      6. 校验通过 → 提交事务；校验失败 → 回滚 + 报警
    
    Milvus 侧:
      经验 embedding 批量 insert（与 Neo4j 入库保持同步）
      embedding 文本构建见 3.1 节"向量化文本构建"
```

##### Approach 归一化（关键问题）

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
