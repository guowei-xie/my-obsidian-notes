# 经验落库处理

---

## 1. 落库在管线中的位置

```
Prompt 2-2/2-2-lite 输出（结构化经验 JSON）
  ↓
Prompt 2-3 质量评审 → reject 直接丢弃
  ↓
Prompt 2-4 Approach 归一化 → 确定 approach_id
  ↓
▶ 落库处理（本文档）
  ├─ 字段提取与校验
  ├─ 节点写入（Neo4j）
  ├─ 边写入（Neo4j）
  ├─ 统计指标更新
  └─ 向量写入（Milvus）
```

落库的输入是**经过质量评审（approve）且完成 Approach 归一化**的经验记录。每条经验独立入库，幂等处理。

---

## 2. 输入数据结构

落库接收的 JSON 是 Prompt 2-2-lite 输出 + Prompt 2-4 归一化结果的合并体：

```json
{
  "experience_id": "自动生成的唯一 ID",
  "source": {
    "dialogue_id": "原始对话 ID",
    "contrast_dialogue_id": "对照对话 ID（lite 版为 null）",
    "extraction_method": "comparison | single",
    "extracted_at": "2026-04-27T10:00:00Z",
    "quality_score": 4.2
  },

  "experience_type": "intent_advance | blocker_resolve | signal_capture | pacing | context_leverage",

  "context": {
    "description": "适用情境描述",
    "intent_stage": "S2",
    "intent_stage_reason": "判定依据",
    "positive_factors": ["学习兴趣"],
    "negative_factors": ["价格敏感"]
  },

  "judgment": {
    "what": "销售做了什么",
    "why_effective": "为什么有效",
    "alternative_avoided": "对照案例做法（仅 comparison 版有）"
  },

  "execution": {
    "approach": "方法一句话概括",
    "approach_id": "APR-017（来自 Prompt 2-4 归一化结果）",
    "approach_match_type": "matched | revise | new",
    "how": "具体做法"
  },

  "effect": {
    "user_response_change": "用户反应变化"
  },

  "evidence": {
    "before": "...", "action": "...", "after": "...",
    "contrast_action": "...", "contrast_after": "..."
  },

  "retrieval_tags": {
    "from_stage": "S2",
    "to_stage": "S3",
    "factors_addressed": ["价格敏感"],
    "topics": ["课程体验", "价格费用"],
    "applicable_segments": ["低年级家长"]
  }
}
```

> `source` 和 `execution.approach_id` 是管线中间环节附加的元数据，不来自 LLM 输出。

---

## 3. 字段提取与校验

落库前的预处理，确保数据完整性和一致性。

### 3.1 必填字段校验

| 字段路径 | 校验规则 | 失败处理 |
|---------|---------|---------|
| `experience_type` | 必须是 5 种类型之一 | 拒绝入库 |
| `retrieval_tags.from_stage` | S0-S4 | 拒绝入库 |
| `retrieval_tags.to_stage` | S0-S4，且 >= from_stage | 拒绝入库 |
| `execution.approach_id` | 非空 | 拒绝入库（说明归一化未完成） |
| `evidence.before/action/after` | 非空字符串 | 拒绝入库 |
| `context.positive_factors` + `negative_factors` | 至少一个非空 | 警告，允许入库 |

### 3.2 因素标签标准化

LLM 输出的因素标签可能有措辞偏差，需映射到标准标签体系：

```python
STANDARD_FACTORS = {
    "positive": ["学习兴趣", "目标明确", "主动询价", "主动约课",
                 "社交背书", "体验认可", "竞赛需求", "时间窗口明确"],
    "negative": ["价格敏感", "决策权模糊", "时间冲突", "效果怀疑",
                 "兴趣不足", "竞品倾向", "年龄顾虑"]
}

# 模糊匹配映射表（人工维护，随 LLM 输出偏差积累迭代）
FACTOR_ALIASES = {
    "对价格敏感": "价格敏感",
    "嫌贵": "价格敏感",
    "决策权不明确": "决策权模糊",
    "担心效果": "效果怀疑",
    # ...
}
```

处理逻辑：
1. 精确匹配标准标签 → 直接使用
2. 命中别名映射 → 替换为标准标签
3. 均未命中 → 标记为 `_unmapped`，记录到待审核队列，人工决定归入已有标签或新增标签

### 3.3 去重检查

同一 `dialogue_id` + 相同 `evidence.action`（文本相似度 > 0.95）→ 视为重复，跳过。

当 `extraction_method: "single"` 的经验被 `comparison` 版重新萃取时，以 comparison 版为准，更新已有记录（详见第 4.5 节）。

---

## 4. 经验 → 图数据转换（核心）

本节详细展开一条经验 JSON 如何被拆解、转换、写入图谱和向量库。

### 4.1 转换总览

```
经验 JSON（校验通过后）
  │
  ├─ Step A: 因素极性分类
  │    factors_addressed × (positive_factors ∪ negative_factors) → 分组
  │
  ├─ Step B: 边类型规划
  │    experience_type + 因素分组 → 确定需要创建哪些边
  │
  ├─ Step C: 节点写入（Neo4j）
  │    Factor(MERGE) → Approach(MERGE) → Experience(CREATE)
  │
  ├─ Step D: 边写入（Neo4j）
  │    TRANSITION → BLOCKS/DRIVES → RESOLVES/LEVERAGES → EVIDENCE_OF
  │
  ├─ Step E: 向量写入（Milvus）
  │    拼接语义文本 → embedding → upsert
  │
  └─ Step F: 派生指标更新
       evidence_count → success_rate / block_strength / drive_strength
```

### 4.2 Step A: 因素极性分类

`retrieval_tags.factors_addressed` 记录了该经验涉及的因素，但不标注极性。需要与 `context` 中的因素列表交叉确认，将其分为 positive 和 negative 两组。

**分类逻辑：**

```python
def classify_factors(experience):
    pos_set = set(experience["context"]["positive_factors"])
    neg_set = set(experience["context"]["negative_factors"])

    result = {"positive": [], "negative": []}
    for factor in experience["retrieval_tags"]["factors_addressed"]:
        if factor in neg_set:
            result["negative"].append(factor)
        elif factor in pos_set:
            result["positive"].append(factor)
        else:
            # factors_addressed 中出现但 context 中未列出
            # 根据标准标签体系的默认极性归类
            result[default_polarity(factor)].append(factor)
    return result
```

**示例：**

```
输入:
  context.positive_factors: ["学习兴趣"]
  context.negative_factors: ["价格敏感"]
  retrieval_tags.factors_addressed: ["价格敏感", "学习兴趣"]

输出:
  positive: ["学习兴趣"]
  negative: ["价格敏感"]
```

### 4.3 Step B: 边类型规划

根据 `experience_type` + Step A 的因素分组，确定需要创建哪些边。

**决策矩阵：**

| 经验类型 | TRANSITION | BLOCKS | DRIVES | RESOLVES | LEVERAGES | EVIDENCE_OF |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| `intent_advance` | 必建 | — | 有 pos 因素则建 | — | 有 pos 因素则建 | 必建 |
| `blocker_resolve` | 必建 | 有 neg 因素则建 | — | 有 neg 因素则建 | — | 必建 |
| `signal_capture` | 必建 | — | 有 pos 因素则建 | — | 有 pos 因素则建 | 必建 |
| `pacing` | from≠to 时建 | — | — | — | — | 必建 |
| `context_leverage` | 必建 | 有 neg 因素则建 | 有 pos 因素则建 | — | 有 pos 因素则建 | 必建 |

**规划逻辑（伪代码）：**

```python
def plan_edges(experience_type, factors, from_stage, to_stage):
    plan = {
        "TRANSITION": experience_type != "pacing" or from_stage != to_stage,
        "EVIDENCE_OF": True,  # 所有类型必建
        "BLOCKS": [],         # 需要建 BLOCKS 的消极因素列表
        "DRIVES": [],         # 需要建 DRIVES 的积极因素列表
        "RESOLVES": [],       # 需要建 RESOLVES 的消极因素列表
        "LEVERAGES": [],      # 需要建 LEVERAGES 的积极因素列表
    }

    if experience_type in ("blocker_resolve", "context_leverage"):
        plan["BLOCKS"] = factors["negative"]

    if experience_type == "blocker_resolve":
        plan["RESOLVES"] = factors["negative"]

    if experience_type in ("intent_advance", "signal_capture", "context_leverage"):
        plan["DRIVES"] = factors["positive"]
        plan["LEVERAGES"] = factors["positive"]

    return plan
```

**示例（blocker_resolve 类型）：**

```
输入:
  experience_type: "blocker_resolve"
  factors: {positive: ["学习兴趣"], negative: ["价格敏感"]}
  from_stage: "S2", to_stage: "S3"

边规划结果:
  TRANSITION:  ✓ (S2 → S3)
  BLOCKS:      ["价格敏感"]     → (价格敏感)-[BLOCKS]->(S3)
  DRIVES:      []               → 不建
  RESOLVES:    ["价格敏感"]     → (APR-017)-[RESOLVES]->(价格敏感)
  LEVERAGES:   []               → 不建
  EVIDENCE_OF: ✓               → (EXP-xxx)-[EVIDENCE_OF]->(APR-017)
```

### 4.4 Step C: 节点写入（Neo4j）

节点采用 **MERGE**（存在则复用，不存在则创建），避免重复。

**写入顺序：IntentStage → Factor → Approach → Experience**（后者依赖前者存在）

#### (1) IntentStage 节点

系统初始化时预建，共 5 个（S0-S4），落库时不操作。

```cypher
// 初始化脚本（仅运行一次）
UNWIND ["S0","S1","S2","S3","S4"] AS stage
MERGE (:IntentStage {name: stage})
```

#### (2) Factor 节点

**来源字段解析：**

经验 JSON 中有 3 处涉及因素，作用不同：

| 字段 | 作用 | 是否触发创建 Factor 节点 |
|------|------|:---:|
| `context.positive_factors` | 描述该时刻存在的积极因素 | 是（MERGE） |
| `context.negative_factors` | 描述该时刻存在的消极因素 | 是（MERGE） |
| `retrieval_tags.factors_addressed` | 该经验实际化解/利用了哪些因素 | 否（只引用，不新建） |

> `factors_addressed` 是 `positive_factors ∪ negative_factors` 的子集 —— 经验不一定处理了所有存在的因素。

```cypher
// 写入所有 context 中提到的因素
MERGE (f:Factor {name: $factor_name, polarity: $polarity})
```

#### (3) Approach 节点

根据 Prompt 2-4 归一化结果的 `approach_match_type` 分三种情况：

```cypher
// matched: 复用已有节点，增加计数
MATCH (a:Approach {approach_id: $approach_id})
SET a.example_count = a.example_count + 1,
    a.last_updated = $timestamp

// revise: 复用已有节点，增加计数，同时更新名称/描述
MATCH (a:Approach {approach_id: $approach_id})
SET a.name = $revised_name,
    a.description = $revised_description,
    a.example_count = a.example_count + 1,
    a.last_updated = $timestamp

// new: 创建新节点
CREATE (a:Approach {
  approach_id: $approach_id,
  name: $approach_name,
  description: $approach_desc,
  example_count: 1,
  created_at: $timestamp
})
```

**字段来源映射：**

| Approach 节点属性 | matched | revise | new |
|------------------|---------|--------|-----|
| `approach_id` | 2-4 输出 `matched_approach_id` | 2-4 输出 `revise_approach_id` | 自动生成 `APR-{seq}` |
| `name` | 不变 | 2-4 输出 `revised_name` | 2-4 输出 `suggested_name` |
| `description` | 不变 | 2-4 输出 `revised_description` | 2-4 输出 `suggested_description` |
| `example_count` | +1 | +1 | 1 |

> `revise` 的 confidence 为 medium 时建议人工复核后再执行更新，避免少数新经验的措辞偏差覆盖了已有的好命名。

#### (4) Experience 节点

每条经验创建一个新节点（CREATE，非 MERGE），存储完整经验内容（扁平化）。

```cypher
CREATE (e:Experience {
  experience_id: $exp_id,
  experience_type: $type,

  // context 字段
  description: $context_description,
  intent_stage: $context_intent_stage,
  intent_stage_reason: $context_intent_stage_reason,
  positive_factors: $context_positive_factors,    // 字符串数组
  negative_factors: $context_negative_factors,

  // judgment 字段
  judgment_what: $judgment_what,
  judgment_why: $judgment_why_effective,
  alternative_avoided: $judgment_alternative_avoided,  // lite 版为 null

  // execution 字段（approach 内容冗余存储，查询时不需要 JOIN）
  approach_id: $approach_id,
  approach_name: $approach_name,
  execution_how: $execution_how,

  // effect 字段
  effect: $effect_user_response_change,

  // evidence 字段
  evidence_before: $evidence_before,
  evidence_action: $evidence_action,
  evidence_after: $evidence_after,
  evidence_contrast_action: $evidence_contrast_action,  // lite 版为 null
  evidence_contrast_after: $evidence_contrast_after,

  // retrieval_tags（冗余存储，便于图内过滤）
  from_stage: $from_stage,
  to_stage: $to_stage,
  topics: $topics,
  factors_addressed: $factors_addressed,
  applicable_segments: $applicable_segments,

  // 元数据
  source_dialogue_id: $dialogue_id,
  contrast_dialogue_id: $contrast_dialogue_id,
  extraction_method: $extraction_method,
  quality_score: $quality_score,
  created_at: $timestamp
})
```

> Experience 节点存储完整经验内容（扁平化），查询时直接返回，不需要再查外部存储。冗余是有意为之——图谱查询的核心场景是"沿路径找到相关经验后立即展示"，减少跨库 JOIN。

### 4.5 Step D: 边写入（Neo4j）

按 Step B 的规划结果，依次创建各类边。

**写入顺序：TRANSITION → BLOCKS/DRIVES → RESOLVES/LEVERAGES → EVIDENCE_OF**

#### (1) TRANSITION（IntentStage → IntentStage）

```cypher
MERGE (s1:IntentStage {name: $from_stage})-[t:TRANSITION]->(s2:IntentStage {name: $to_stage})
ON CREATE SET t.evidence_count = 1, t.created_at = $timestamp
ON MATCH SET t.evidence_count = t.evidence_count + 1
SET t.last_updated = $timestamp
```

> TRANSITION 是**聚合边**，多条经验共享同一条边，通过 `evidence_count` 积累统计信号。例如 S2→S3 可能有 45 条经验支撑。

#### (2) BLOCKS（Factor → IntentStage）

对 Step B 中 `plan["BLOCKS"]` 的每个消极因素：

```cypher
MERGE (f:Factor {name: $neg_factor, polarity: "negative"})
      -[b:BLOCKS {from_stage: $from_stage}]->
      (s:IntentStage {name: $to_stage})
ON CREATE SET b.evidence_count = 1
ON MATCH SET b.evidence_count = b.evidence_count + 1
```

> `from_stage` 作为边属性是 Neo4j 的建模适配——语义上 BLOCKS 指向的是 `from_stage → to_stage` 这条 TRANSITION，但 Neo4j 不支持"边指向边"，所以将路径信息冗余到 BLOCKS 边属性中。

**含义：** "价格敏感"在 S2→S3 的推进过程中构成阻碍。

#### (3) DRIVES（Factor → IntentStage）

对 Step B 中 `plan["DRIVES"]` 的每个积极因素：

```cypher
MERGE (f:Factor {name: $pos_factor, polarity: "positive"})
      -[d:DRIVES {from_stage: $from_stage}]->
      (s:IntentStage {name: $to_stage})
ON CREATE SET d.evidence_count = 1
ON MATCH SET d.evidence_count = d.evidence_count + 1
```

**含义：** "学习兴趣"在 S2→S3 的推进过程中起驱动作用。

#### (4) RESOLVES（Approach → Factor）

仅当 `experience_type == "blocker_resolve"` 时，对 `plan["RESOLVES"]` 的每个消极因素：

```cypher
MERGE (a:Approach {approach_id: $approach_id})
      -[r:RESOLVES {for_from_stage: $from_stage, for_to_stage: $to_stage}]->
      (f:Factor {name: $neg_factor, polarity: "negative"})
ON CREATE SET r.evidence_count = 1
ON MATCH SET r.evidence_count = r.evidence_count + 1
```

**含义：** 方法"先强化价值再谈价格"能在 S2→S3 路径上化解"价格敏感"。

#### (5) LEVERAGES（Approach → Factor）

对 `plan["LEVERAGES"]` 的每个积极因素：

```cypher
MERGE (a:Approach {approach_id: $approach_id})
      -[l:LEVERAGES {for_from_stage: $from_stage, for_to_stage: $to_stage}]->
      (f:Factor {name: $pos_factor, polarity: "positive"})
ON CREATE SET l.evidence_count = 1
ON MATCH SET l.evidence_count = l.evidence_count + 1
```

**含义：** 方法"用孩子兴趣撬动家长认同"能在 S2→S3 路径上利用"学习兴趣"。

#### (6) EVIDENCE_OF（Experience → Approach）

```cypher
MATCH (e:Experience {experience_id: $exp_id})
MATCH (a:Approach {approach_id: $approach_id})
CREATE (e)-[:EVIDENCE_OF]->(a)
```

> 一对一，每条经验只关联一个 Approach。反向遍历时，一个 Approach 可能有多条 EVIDENCE_OF 指向它，形成证据集合。

### 4.6 Step E: 向量写入（Milvus）

#### Embedding 文本构造

拼接检索时最需要语义匹配的字段，**不含 evidence（对话原文）**——原文是特定用户的措辞，会引入噪声：

```python
def build_embedding_text(exp):
    parts = [
        f"情境：{exp['context']['description']}",
        f"判断：{exp['judgment']['what']}",
        f"原因：{exp['judgment']['why_effective']}",
        f"方法：{exp['execution']['approach']}",
    ]
    return "\n".join(parts)
```

#### 写入字段

| Milvus 字段 | 类型 | 来源 | 用途 |
|-------------|------|------|------|
| `experience_id` | VARCHAR | 自动生成 | 主键，关联 Neo4j |
| `embedding` | FLOAT_VECTOR(dim) | build_embedding_text → encode | 语义检索 |
| `from_stage` | VARCHAR | retrieval_tags.from_stage | 结构化过滤 |
| `to_stage` | VARCHAR | retrieval_tags.to_stage | 结构化过滤 |
| `experience_type` | VARCHAR | experience_type | 结构化过滤 |
| `factors_addressed` | VARCHAR (JSON) | retrieval_tags.factors_addressed | 结构化过滤 |
| `topics` | VARCHAR (JSON) | retrieval_tags.topics | 结构化过滤 |
| `quality_score` | FLOAT | source.quality_score | 排序加权 |
| `extraction_method` | VARCHAR | source.extraction_method | 区分 single/comparison |

#### 检索配合方式

```
Step 1: 结构化过滤（缩小候选集）
  WHERE from_stage = 当前意向阶段
    AND experience_type IN (需要的类型)
    AND factors_addressed 与当前因素有交集

Step 2: 向量排序（精排）
  query_embedding = embed(用户背景摘要 + 当前话题/因素描述)
  ORDER BY cosine_similarity(embedding, query_embedding) DESC
  LIMIT K
```

### 4.7 Step F: 派生指标更新

经验入库后更新图谱上的聚合统计指标，供策略生成时使用。

#### TRANSITION.success_rate

```
success_rate = 该路径上的经验数 / 该 from_stage 出发的总经验数
```

```cypher
MATCH (s1:IntentStage {name: $from_stage})-[t:TRANSITION]->(s2)
WITH s1, sum(t.evidence_count) AS total
MATCH (s1)-[t2:TRANSITION]->(target:IntentStage {name: $to_stage})
SET t2.success_rate = toFloat(t2.evidence_count) / total
```

> 不是真正的"成功率"，而是"经验分布比例"。萃取的经验本身来自成功案例，该比例反映成功路径的经验密度。

#### BLOCKS.block_strength / DRIVES.drive_strength

```
block_strength = 该因素在该路径上的阻碍经验数 / 该路径总经验数
drive_strength = 该因素在该路径上的驱动经验数 / 该路径总经验数
```

> 越高表示该因素在该路径上越频繁出现，越值得策略关注。

#### 更新策略

- **实时更新**：每条经验入库后立即更新涉及边的 `evidence_count`（Step D 中已完成）
- **批量更新**：派生比例指标（`success_rate`, `block_strength`, `drive_strength`）在每批入库完成后统一重算

### 4.8 完整示例：一条经验的全流程转换

**输入（经过校验和标准化后）：**

```json
{
  "experience_type": "blocker_resolve",
  "source": {
    "dialogue_id": "DLG-2026-0412",
    "extraction_method": "single",
    "quality_score": 4.2
  },
  "context": {
    "description": "家长对价格犹豫，但孩子有明确学习兴趣",
    "intent_stage": "S2",
    "intent_stage_reason": "家长有话题交集但未形成明确方向",
    "positive_factors": ["学习兴趣"],
    "negative_factors": ["价格敏感"]
  },
  "judgment": {
    "what": "不直接回应价格，先聊孩子体验课表现",
    "why_effective": "将对话焦点从价格转移到孩子的成长价值，让家长自己感知投入的合理性"
  },
  "execution": {
    "approach": "先强化价值再谈价格",
    "approach_id": "APR-017",
    "approach_match_type": "matched",
    "how": "先提及孩子在体验课中的具体表现（作品/进步），引导家长关注学习效果，等家长主动认可后再自然过渡到课程方案"
  },
  "effect": {
    "user_response_change": "从'有点贵'变为'那我了解一下具体怎么上课'"
  },
  "evidence": {
    "before": "家长: 感觉价格有点贵啊",
    "action": "销售: 对了张妈妈，上次体验课老师特别提到小张的逻辑思维很好...",
    "after": "家长: 是吗！那我了解一下具体怎么上课？每周几次？"
  },
  "retrieval_tags": {
    "from_stage": "S2",
    "to_stage": "S3",
    "factors_addressed": ["价格敏感", "学习兴趣"],
    "topics": ["课程体验", "价格费用"],
    "applicable_segments": ["低年级家长"]
  }
}
```

**Step A → 因素分类：**

```
positive: ["学习兴趣"]
negative: ["价格敏感"]
```

**Step B → 边规划（blocker_resolve 类型）：**

```
TRANSITION:  ✓  (S2 → S3)
BLOCKS:      ["价格敏感"]
DRIVES:      []
RESOLVES:    ["价格敏感"]
LEVERAGES:   []
EVIDENCE_OF: ✓
```

**Step C → 节点写入：**

```
1. MERGE (:Factor {name:"价格敏感", polarity:"negative"})
2. MERGE (:Factor {name:"学习兴趣", polarity:"positive"})
3. MATCH (:Approach {approach_id:"APR-017"})  SET example_count += 1
4. CREATE (:Experience {experience_id:"EXP-00142", ...全部字段扁平化写入...})
```

**Step D → 边写入：**

```
5. MERGE (S2)-[:TRANSITION]->(S3)  ON MATCH SET evidence_count += 1
6. MERGE (价格敏感)-[:BLOCKS {from_stage:"S2"}]->(S3)  ON MATCH SET evidence_count += 1
7. MERGE (APR-017)-[:RESOLVES {for_from_stage:"S2", for_to_stage:"S3"}]->(价格敏感)  ON MATCH SET evidence_count += 1
8. CREATE (EXP-00142)-[:EVIDENCE_OF]->(APR-017)
```

> 注意：虽然 context 中有"学习兴趣"，但 blocker_resolve 类型不建 DRIVES/LEVERAGES 边——该经验的核心价值是化解阻塞，不是利用驱动因素。

**Step E → 向量写入：**

```
embedding_text = """情境：家长对价格犹豫，但孩子有明确学习兴趣
判断：不直接回应价格，先聊孩子体验课表现
原因：将对话焦点从价格转移到孩子的成长价值，让家长自己感知投入的合理性
方法：先强化价值再谈价格"""

→ encode → Milvus upsert (experience_id="EXP-00142", from_stage="S2", ...)
```

**Step F → 派生指标（批量结束后统一重算）：**

```
S2→S3 TRANSITION: evidence_count=46, success_rate 重算
(价格敏感)-[BLOCKS]->(S3): evidence_count=12, block_strength 重算
```

**最终图谱局部视图：**

```
                  ┌── [价格敏感] ──BLOCKS{from:"S2", count:12}──┐
                  │                                            ↓
  [S2] ──TRANSITION{count:46, rate:0.62}──→ [S3]
                                               ↑
  [APR-017: 先强化价值再谈价格] ──RESOLVES{count:8}──→ [价格敏感]
                  ↑
  [EXP-00142] ──EVIDENCE_OF──┘
```

---

## 5. 向量写入（Milvus）

### 5.1 Embedding 文本构造

经验不是直接对全 JSON 做 embedding，而是拼接检索时最需要匹配的语义字段：

```python
def build_embedding_text(exp):
    parts = [
        f"情境：{exp['context']['description']}",
        f"判断：{exp['judgment']['what']}",
        f"原因：{exp['judgment']['why_effective']}",
        f"方法：{exp['execution']['approach']}",
    ]
    return "\n".join(parts)
```

> 不包含 evidence（对话原文）—— 原文是特定用户的措辞，会引入噪声，降低语义匹配的泛化性。

### 5.2 写入 Milvus 的字段

| 字段 | 类型 | 用途 |
|------|------|------|
| `experience_id` | VARCHAR | 主键，关联 Neo4j |
| `embedding` | FLOAT_VECTOR(dim) | 语义检索 |
| `from_stage` | VARCHAR | 结构化过滤 |
| `to_stage` | VARCHAR | 结构化过滤 |
| `experience_type` | VARCHAR | 结构化过滤 |
| `factors_addressed` | VARCHAR (JSON array) | 结构化过滤（交集匹配） |
| `topics` | VARCHAR (JSON array) | 结构化过滤 |
| `quality_score` | FLOAT | 排序加权 |
| `extraction_method` | VARCHAR | 区分 single/comparison |

### 5.3 检索时的使用方式

```
Step 1: 结构化过滤（缩小候选集）
  WHERE from_stage = 当前意向阶段
    AND experience_type IN (需要的类型)
    AND factors_addressed 与当前因素有交集

Step 2: 向量排序（精排）
  query_embedding = embed(用户背景摘要 + 当前话题/因素描述)
  ORDER BY cosine_similarity(embedding, query_embedding) DESC
  LIMIT K
```

---

## 6. 派生指标更新

经验入库后，需要更新图谱上的聚合统计指标，供策略生成时使用。

### 6.1 TRANSITION.success_rate

```
success_rate = 该路径上的经验数 / 该 from_stage 的总经验数

更新时机: 每次写入 TRANSITION 边后
计算方式:
  MATCH (s1:IntentStage {name: from_stage})-[t:TRANSITION]->(s2)
  WITH s1, sum(t.evidence_count) AS total
  MATCH (s1)-[t2:TRANSITION]->(target:IntentStage {name: to_stage})
  SET t2.success_rate = toFloat(t2.evidence_count) / total
```

> 这不是真正的"成功率"，而是"经验分布比例"。命名保留 `success_rate` 是因为萃取的经验本身来自成功案例，该比例反映了成功路径的经验密度。

### 6.2 BLOCKS.block_strength / DRIVES.drive_strength

```
block_strength = 该因素在该路径上的阻碍经验数 / 该路径总经验数
drive_strength = 该因素在该路径上的驱动经验数 / 该路径总经验数

含义: 越高表示该因素在该路径上越频繁出现，越值得关注。
```

### 6.3 更新策略

- **实时更新**：每条经验入库后立即更新涉及的边的 `evidence_count`
- **批量更新**：派生比例指标（`success_rate`, `block_strength`, `drive_strength`）在每批入库完成后统一计算，避免频繁重算

---

## 5. Demo 版 vs 正式版的差异

Demo 版直接使用 Neo4j 构建 GraphRAG，与正式版共享同一套图谱结构和落库流程。

| 环节 | Demo 版（single） | 正式版（comparison） |
|------|-------------------|---------------------|
| 输入来源 | Prompt 2-2-lite | Prompt 2-2 |
| `alternative_avoided` | 无 | 有（来自真实对照案例） |
| `contrast_insights` | 无 | 有 |
| `evidence.contrast_*` | 无 | 有 |
| Approach 归一化 | 全人工（< 200 条不启用 2-4） | 前 50 条人工 + 后续 Prompt 2-4 |
| 图谱写入 | 写 Neo4j | 写 Neo4j |
| 向量写入 | 写 Milvus | 写 Milvus |
| `extraction_method` 标记 | `"single"` | `"comparison"` |

> Demo 版虽然经验量较少（~150-200 条），图谱统计信号较弱，但 Neo4j 的结构化查询能力（路径查询、因素关联）在小数据量下已经比纯向量检索更有价值。策略生成时 `confidence` 如实标注为 low/medium 即可。

**Demo → 正式的过渡处理：**

当同一对话被正式版重新萃取时：
1. 以 `source_dialogue_id` 查找已有 single 版 Experience 节点
2. **替换 Experience 节点内容**（更新属性），`extraction_method` 改为 `"comparison"`
3. 补入 contrast 相关字段（`alternative_avoided`, `evidence_contrast_*`）
4. **边不需要删除重建**——MERGE 语义保证 TRANSITION/BLOCKS/DRIVES 等聚合边天然幂等
5. Milvus 中以 `experience_id` 做 upsert，embedding 用新内容重新生成

---

## 6. 入库流程伪代码

```python
def ingest_experience(exp_json):
    """单条经验入库主流程（Demo 版和正式版共用）"""

    # ── Step 0: 校验与预处理 ──
    validate_required_fields(exp_json)
    standardize_factor_labels(exp_json)

    if is_duplicate(exp_json):
        if should_upgrade(exp_json):  # single → comparison 升级
            upgrade_existing(exp_json)
            return "upgraded"
        else:
            return "skipped: duplicate"

    # ── Step A: 因素极性分类 ──
    exp_id = generate_experience_id()
    exp_json["experience_id"] = exp_id
    factors = classify_factors(exp_json)

    # ── Step B: 边类型规划 ──
    tags = exp_json["retrieval_tags"]
    edge_plan = plan_edges(
        exp_json["experience_type"], factors,
        tags["from_stage"], tags["to_stage"]
    )

    # ── Step C + D: Neo4j 写入（节点 + 边，在同一事务内） ──
    approach_id = exp_json["execution"]["approach_id"]
    with neo4j_session() as tx:
        # C: 节点
        upsert_factor_nodes(tx, exp_json["context"])
        upsert_approach_node(tx, exp_json["execution"])  # 处理 matched/revise/new 三种情况
        create_experience_node(tx, exp_json)

        # D: 边
        if edge_plan["TRANSITION"]:
            upsert_transition(tx, tags["from_stage"], tags["to_stage"])

        for neg_f in edge_plan["BLOCKS"]:
            upsert_blocks(tx, neg_f, tags["from_stage"], tags["to_stage"])
        for pos_f in edge_plan["DRIVES"]:
            upsert_drives(tx, pos_f, tags["from_stage"], tags["to_stage"])
        for neg_f in edge_plan["RESOLVES"]:
            upsert_resolves(tx, approach_id, neg_f, tags["from_stage"], tags["to_stage"])
        for pos_f in edge_plan["LEVERAGES"]:
            upsert_leverages(tx, approach_id, pos_f, tags["from_stage"], tags["to_stage"])

        create_evidence_edge(tx, exp_id, approach_id)

    # ── Step E: Milvus 写入 ──
    embedding_text = build_embedding_text(exp_json)
    embedding = embed_model.encode(embedding_text)
    milvus_upsert(
        experience_id=exp_id,
        embedding=embedding,
        from_stage=tags["from_stage"],
        to_stage=tags["to_stage"],
        experience_type=exp_json["experience_type"],
        factors_addressed=tags["factors_addressed"],
        topics=tags["topics"],
        quality_score=exp_json["source"]["quality_score"],
        extraction_method=exp_json["source"]["extraction_method"],
    )

    return f"ingested: {exp_id}"


def upgrade_existing(new_exp):
    """single → comparison 升级：更新已有记录"""
    old_id = find_by_dialogue(new_exp["source"]["dialogue_id"])

    # Neo4j: 更新 Experience 节点属性（边不需要重建，MERGE 幂等）
    with neo4j_session() as tx:
        tx.run("""
            MATCH (e:Experience {experience_id: $old_id})
            SET e.extraction_method = "comparison",
                e.alternative_avoided = $alt,
                e.evidence_contrast_action = $ca,
                e.evidence_contrast_after = $caf,
                e.quality_score = $score,
                e.updated_at = $ts
        """, ...)

    # Milvus: 用新内容重新生成 embedding 并 upsert
    embedding = embed_model.encode(build_embedding_text(new_exp))
    milvus_upsert(experience_id=old_id, embedding=embedding, ...)


def ingest_batch(experiences):
    """批量入库"""
    results = [ingest_experience(exp) for exp in experiences]

    # ── Step F: 批量完成后统一重算派生指标 ──
    recalculate_all_derived_metrics()

    return results
```

---

## 7. 数据一致性保障

| 风险 | 应对 |
|------|------|
| Neo4j 写入成功但 Milvus 失败 | 入库操作记录状态（neo4j_done / milvus_done），失败时可重试。两侧均以 experience_id 为主键，重试幂等 |
| MERGE 节点并发冲突 | Factor/Approach 的 MERGE 天然幂等；Experience 用 CREATE + unique ID，不会冲突 |
| 统计指标不一致 | 派生指标允许最终一致，批量完成后统一重算 |
| Approach 归一化后续调整 | 人工合并两个 Approach 节点时，需迁移所有 EVIDENCE_OF 边 + 更新 Milvus 中的关联（低频操作，手动脚本） |
