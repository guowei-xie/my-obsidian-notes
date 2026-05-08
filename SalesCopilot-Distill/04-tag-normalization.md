# 标签目录与打法归一化

---

## 0. 两个核心问题

- **标签一致性（Tag Catalog）：** 「价格顾虑」和「价格敏感」不能成为两个孤立节点
- **打法去碎片化（Approach 归一化）：** 相同打法的不同经验必须指向同一 Approach 节点，才能积累统计信号

---

## 1. Tag Catalog — 全局标签目录

### 1.1 什么是 Tag Catalog

Tag Catalog 是一个**任务级快照**，在每次蒸馏任务 Stage 3（DISTILL）开始时，从 Neo4j 读取当前图谱中所有 Factor 和 Topic 节点，合并内置默认 Factor 列表，形成本次任务的标签目录，注入到 LLM Prompt 中。

```
Stage 3 启动时:
  ├─ 读 Neo4j: SELECT * FROM Factor NODES → {正向因素列表, 负向因素列表}
  ├─ 读 Neo4j: SELECT * FROM Topic NODES → {话题列表}
  ├─ 合并内置默认 Factor（保底因素，防止冷启动时目录为空）
  └─ 注入 Prompt:
       {positive_factor_catalog} = "学习兴趣高、主动询价、社交背书..."
       {negative_factor_catalog} = "价格敏感、时间冲突、信任缺失..."
       {topic_catalog} = "价格异议、课程匹配、竞赛路径..."
```

### 1.2 LLM 的使用约束

LLM 在提取经验时**必须**：
- 从 `positive_factor_catalog` 中选择正向因素，不得自造
- 从 `negative_factor_catalog` 中选择负向因素，不得自造
- 从 `topic_catalog` 中选择话题，不得自造

如需使用目录中没有的标签，LLM 必须：
1. 将新标签声明在 `new_labels_requested` 字段中
2. 提供充分理由（为什么现有标签无法描述此概念）

### 1.3 标签校验与重试

```
Stage 3 LLM 输出后:
  ├─ 检查 positive_factors / negative_factors / topics 中是否有目录外标签
  │
  ├─ 如有未申报的目录外标签（LLM 擅自创造）:
  │    → 触发重试（在 Prompt 末尾追加：「以下标签不在目录中，请修正或申报：{非法标签列表}」）
  │    → 重试后仍有未申报的 → 该经验跳过
  │
  └─ 已申报的新标签:
       → 进入 Stage 4 时，随经验写入 Neo4j，成为下一个任务的目录成员
```


---

## 2. Approach 归一化

### 2.1 候选召回策略

LLM 做判断前，先从 Neo4j 检索候选 Approach 目录，优先级如下：

```
优先级 1（最高）: 同路径 + 同类型 + 因素重叠
  条件: from_stage=S2, to_stage=S3, experience_type=intent_advance
        AND factors_addressed 有交集
  
优先级 2: 同路径 + 同类型
  条件: from_stage=S2, to_stage=S3, experience_type=intent_advance

优先级 3（兜底）: 邻近路径 + 同类型
  条件: from_stage 相邻（S1 或 S3）+ experience_type=intent_advance
```

召回后格式化为目录文本，注入 Prompt：

```
现有打法目录（相关路径）:
AP-003: 先挖需求再推方案 — 在用户对课程尚无明确认知时...
AP-007: 先强化价值再处理价格异议 — 在用户提出价格顾虑...
AP-012: 从孩子学习情况切入 — 开口先了解孩子情况，建立...
```

### 2.3 三种归一化结果

#### matched — 复用已有打法

```json
{
  "match_result": "matched",
  "matched_approach_id": "AP-012",
  "reason": "「从孩子情况切入」与现有AP-012的核心逻辑完全一致"
}
```

**图谱操作：** 新增 `(经验)-[:EVIDENCE_OF]->(AP-012)`，AP-012 节点不修改。

#### revise — 修订已有打法

```json
{
  "match_result": "revise",
  "matched_approach_id": "AP-012",
  "revised_name": "先了解孩子情况，再建立需求认知",
  "revised_description": "不急于介绍课程，以孩子学习情况为切入点...",
  "reason": "AP-012 的描述不够精确，本经验揭示了更清晰的执行逻辑"
}
```

**图谱操作：** MERGE 更新 AP-012 的 `name` 和 `description`，新增 EVIDENCE_OF 边。

#### new — 创建新打法

```json
{
  "match_result": "new",
  "new_name": "承接直播间卖点，直接切入竞赛路径",
  "new_description": "当用户来自信奥专场直播时，直接引用直播中的竞赛内容...",
  "reason": "现有目录中没有涉及「承接渠道上下文」的打法，是全新策略"
}
```

**图谱操作：**
1. 读 Counter 节点，原子地将 `value` +1，获取新 ID（如 43 → `AP-043`）
2. CREATE 新 Approach 节点
3. 新增 EVIDENCE_OF 边

### 2.4 归一化失败的兜底

LLM 调用失败或返回格式异常时，**不中断流程**，直接降级为 `new`：

```python
try:
    data, cost_ms = self._call_llm(exp, catalog_text)
except Exception as e:
    logger.warning(f"Approach 归一化 LLM 失败，降级 new: {e}")
    return self._create_new(exp, writer, "", "")
```

目录为空（冷启动第一批任务）时同样直接走 `new` 分支。

---

## 3. 语义去重

Approach 归一化保证了经验指向正确的 Approach 节点，但同一 Approach 上积累大量高度相似的经验也没有价值。语义去重在 INGEST 阶段进行：

### 3.1 触发条件（三者同时满足）

```
① approach_id 相同（同一 Approach 节点）
② from_stage = 相同 AND to_stage = 相同（同一转移路径）
③ Jaccard(新经验.factors_addressed, 已有经验.factors_addressed) ≥ 阈值
   （配置项: factors_overlap_threshold = 0.5）
```

### 3.2 上限控制

每个 (approach_id, from_stage, to_stage) 组合最多保留 `max_similar_experiences`（默认 3）条经验，超出的写入 `status='skipped'`。

---

## 4. 配置项汇总

```ini
[dedup]
max_similar_experiences = 3       ; 每个(打法+路径)最多保留多少条相似经验
factors_overlap_threshold = 0.5   ; Jaccard 相似度阈值，超过则视为重复
```

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [00-overview.md](00-overview.md) | 系统概述与整体架构 |
| [01-pipeline.md](01-pipeline.md) | 四阶段蒸馏流水线（Stage 3/4 详解） |
| [02-experience-model.md](02-experience-model.md) | 经验数据模型（factors / topics 字段） |
| [03-graph-schema.md](03-graph-schema.md) | 知识图谱结构（Factor / Approach / RESOLVES 等） |
| [05-task-management.md](05-task-management.md) | 任务管理与数据库设计 |
| [06-api-deployment.md](06-api-deployment.md) | API 接口与部署 |
