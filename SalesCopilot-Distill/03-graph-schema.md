# 知识图谱结构

---

## 0. 为什么用图

经验检索的核心是**状态转移**问题，而非纯向量相似度：节点 = 意向阶段/因素/打法，边 = 转移关系，边权重 = 经验数量。

> **向量检索回答「谁和我像」，图谱回答「我该往哪走、怎么走」。**

---

## 1. 节点定义

### 1.1 IntentStage — 意向阶段

```cypher
(:IntentStage {name: 'S2', description: '需求模糊，有初步兴趣但尚未明确'})
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | string, **唯一** | S0 / S1 / S2 / S3 / S4 |
| `description` | string | 阶段描述 |

五个固定节点，系统初始化时写入，此后只读（MERGE 不会重建）。

**意向阶段定义：**

| 阶段 | 含义 |
|------|------|
| S0 | 完全沉默 / 未有效互动 |
| S1 | 有基础互动，但无明确意向信号 |
| S2 | 有初步兴趣，需求模糊 |
| S3 | 需求明确，主动了解细节 |
| S4 | 强购买信号 / 已成交 |

### 1.2 Factor — 影响因素

```cypher
(:Factor {name: '价格敏感', polarity: 'negative'})
(:Factor {name: '学习兴趣高', polarity: 'positive'})
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | string | 因素名称（来自全局 Tag Catalog） |
| `polarity` | enum | `positive`（驱动因素）/ `negative`（阻塞因素） |

唯一约束：`(name, polarity)` 组合唯一，防止极性不同但同名的因素被合并。

### 1.3 Approach — 销售打法

```cypher
(:Approach {
  approach_id: 'AP-007',
  name: '先强化价值再处理价格异议',
  description: '在用户提出价格顾虑时，先回溯已建立的价值认知，再回应价格话题'
})
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `approach_id` | string, **唯一** | 格式 `AP-{序号}`，通过 Counter 节点自增 |
| `name` | string | 打法名称（归一化后的标准名） |
| `description` | string | 打法描述 |

### 1.4 Experience — 经验记录

```cypher
(:Experience {
  experience_id: 'EXP-123_456-0',
  experience_type: 'blocker_resolve',
  from_stage: 'S2',
  to_stage: 'S3',
  judgment: '...',
  effect: '...',
  created_at: '2025-01-15T10:30:00'
})
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `experience_id` | string, **唯一** | 格式见 [02-experience-model.md](02-experience-model.md) |
| `experience_type` | enum | 五种类型之一 |
| `from_stage` | string | 起始意向阶段 |
| `to_stage` | string | 目标意向阶段 |
| `judgment` | string | 判断摘要（what + why_effective） |
| `effect` | string | 用户状态变化描述 |

完整 JSON（含 evidence / execution 等）存储在 MySQL `distill_distillations` 中，Neo4j 只存检索侧必要属性。

### 1.5 Topic — 对话话题

```cypher
(:Topic {name: '价格异议'})
```

| 属性 | 类型 | 说明 |
|------|------|------|
| `name` | string, **唯一** | 话题名称（来自全局 Tag Catalog） |

### 1.6 Counter — 序号生成器

```cypher
(:Counter {name: 'approach_id', value: 42})
```

专用于 Approach ID 的分布式自增。每次创建新 Approach 时，在同一事务中用 Cypher 原子地将 `value` +1 并读取新值。

---

## 2. 边定义

### 2.1 TRANSITION — 阶段转移

```cypher
(:IntentStage {name:'S2'})-[:TRANSITION {
  success_rate: 0.72,
  example_count: 35
}]->(:IntentStage {name:'S3'})
```

| 属性 | 说明 |
|------|------|
| `success_rate` | 历史转移成功率（有经验支撑的比例） |
| `example_count` | 支撑该路径的经验数量 |

### 2.2 BLOCKS / DRIVES — 因素与阶段转移的关系

```cypher
(:Factor {name:'价格敏感'})-[:BLOCKS {from_stage:'S2', to_stage:'S3', strength: 0.8}]->(:IntentStage {name:'S3'})
(:Factor {name:'学习兴趣高'})-[:DRIVES {from_stage:'S2', to_stage:'S3', strength: 0.6}]->(:IntentStage {name:'S3'})
```

**Neo4j 建模注意：** BLOCKS / DRIVES 语义上指向「S2→S3 的 TRANSITION 边」，但 Neo4j 不支持边指向边。采用**路径属性冗余**方案：将 `from_stage` 和 `to_stage` 作为 BLOCKS / DRIVES 边的属性，查询时用属性过滤定位正确路径。

| 属性 | 说明 |
|------|------|
| `from_stage` | 阻塞/驱动发生的起始阶段 |
| `to_stage` | 目标阶段 |
| `strength` | 信号强度（0-1，由经验数量推导） |

### 2.3 RESOLVES / LEVERAGES — 打法与因素的关系

```cypher
(:Approach {id:'AP-007'})-[:RESOLVES {
  for_from_stage: 'S2',
  for_to_stage: 'S3',
  example_count: 12
}]->(:Factor {name:'价格敏感'})

(:Approach {id:'AP-003'})-[:LEVERAGES {
  for_from_stage: 'S2',
  for_to_stage: 'S3'
}]->(:Factor {name:'学习兴趣高'})
```

| 边 | 含义 |
|----|------|
| `RESOLVES` | 打法化解了某个阻塞因素（对应 BLOCKS） |
| `LEVERAGES` | 打法利用了某个驱动因素（对应 DRIVES） |

`for_from_stage` + `for_to_stage` 属性说明该打法在哪条转移路径上生效，同一打法可能在不同路径上都有效。

### 2.4 EVIDENCE_OF — 经验支撑打法

```cypher
(:Experience {id:'EXP-123_456-0'})-[:EVIDENCE_OF]->(:Approach {id:'AP-007'})
```

无额外属性。一条 Experience 对应一个 Approach，一个 Approach 可有多条 Experience 支撑。

### 2.5 INVOLVES — 经验涉及话题

```cypher
(:Experience {id:'EXP-123_456-0'})-[:INVOLVES]->(:Topic {name:'价格异议'})
```

一条 Experience 可 INVOLVES 多个 Topic。

---

## 3. 图谱可视化示例

```
                    ┌──── [价格敏感] ──BLOCKS{S2→S3}───┐
                    │                                  ↓
            [S1] ──→ [S2] ──TRANSITION──────────────→ [S3] ──→ [S4]
                    ↑                                  ↑
                    │                                  │
             [学习兴趣高] ──DRIVES{S2→S3}──────────────┘
                    ↑
        [先挖需求再推方案] ──LEVERAGES{S2→S3}──┘
                    ↑
         [EXP-789-0] ──EVIDENCE_OF──┘

         [先强化价值再处理价格异议] ──RESOLVES{S2→S3}──→ [价格敏感]
                    ↑
         [EXP-123_456-0] ──EVIDENCE_OF──┘
```

---

## 4. 经验类型 → 边类型映射

| 经验类型 | 产生的边 |
|---------|---------|
| `intent_advance` | TRANSITION + DRIVES + LEVERAGES + EVIDENCE_OF |
| `blocker_resolve` | TRANSITION + BLOCKS + RESOLVES + EVIDENCE_OF |
| `signal_capture` | TRANSITION + DRIVES + LEVERAGES + EVIDENCE_OF |
| `pacing` | TRANSITION（如有跃迁）+ EVIDENCE_OF |
| `context_leverage` | TRANSITION + DRIVES/BLOCKS + LEVERAGES + EVIDENCE_OF |

所有类型都产生 EVIDENCE_OF；BLOCKS / RESOLVES 仅在存在明确阻塞因素时创建。

---

## 5. 典型 Cypher 查询模式

### 5.1 直接路径查询：「S2→S3，有哪些打法？」

```cypher
MATCH (s:IntentStage {name:'S2'})-[t:TRANSITION]->(e:IntentStage {name:'S3'})
MATCH (a:Approach)-[r:RESOLVES {for_from_stage:'S2', for_to_stage:'S3'}]->(f:Factor)
WHERE f.polarity = 'negative'
WITH a, collect(f.name) AS resolves_factors, count(r) AS strength
ORDER BY strength DESC
RETURN a.approach_id, a.name, resolves_factors
LIMIT 5
```

### 5.2 多步路径：「从 S1 到 S4 的最优路径？」

```cypher
MATCH p = (s1:IntentStage {name:'S1'})-[:TRANSITION*1..3]->(s4:IntentStage {name:'S4'})
WITH p, [r IN relationships(p) | r.example_count] AS counts
RETURN p, reduce(total=0, c IN counts | total + c) AS total_evidence
ORDER BY total_evidence DESC
LIMIT 3
```

---

## 6. Neo4j 约束与索引

```cypher
// 唯一约束（系统启动时初始化）
CREATE CONSTRAINT approach_id_unique IF NOT EXISTS
  FOR (a:Approach) REQUIRE a.approach_id IS UNIQUE;

CREATE CONSTRAINT experience_id_unique IF NOT EXISTS
  FOR (e:Experience) REQUIRE e.experience_id IS UNIQUE;

CREATE CONSTRAINT intent_stage_name_unique IF NOT EXISTS
  FOR (s:IntentStage) REQUIRE s.name IS UNIQUE;

CREATE CONSTRAINT topic_name_unique IF NOT EXISTS
  FOR (t:Topic) REQUIRE t.name IS UNIQUE;

// Factor 复合唯一（name + polarity）
CREATE CONSTRAINT factor_unique IF NOT EXISTS
  FOR (f:Factor) REQUIRE (f.name, f.polarity) IS UNIQUE;
```

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [00-overview.md](00-overview.md) | 系统概述与整体架构 |
| [01-pipeline.md](01-pipeline.md) | 四阶段蒸馏流水线 |
| [02-experience-model.md](02-experience-model.md) | 经验数据模型（字段详解） |
| [04-tag-normalization.md](04-tag-normalization.md) | 标签目录与打法归一化 |
| [05-task-management.md](05-task-management.md) | 任务管理与数据库设计 |
| [06-api-deployment.md](06-api-deployment.md) | API 接口与部署 |
