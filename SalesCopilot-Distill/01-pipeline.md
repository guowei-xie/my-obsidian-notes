# 四阶段蒸馏流水线

---

## 0. 流水线全景

```
原始输入（user_id + counselor_id + 日期范围）
        │
        ▼
┌───────────────────────────────────────────────────────┐
│  Stage 1: FETCH                                       │
│  调用对话 API → 格式化对话文本 → SHA256 去重 → MySQL      │
└───────────────────────────┬───────────────────────────┘
                            │ 保留未见过的对话
                            ▼
┌───────────────────────────────────────────────────────┐
│  Stage 2: KEY MOMENTS                                 │
│  LLM 判断经验密度（high/medium/low）→ 阈值过滤            │
└───────────────────────────┬───────────────────────────┘
                            │ 保留 retained=1 的对话
                            ▼
┌───────────────────────────────────────────────────────┐
│  Stage 3: DISTILL                                     │
│  注入 Tag Catalog → LLM 提取经验 → 标签校验重试           │
└───────────────────────────┬───────────────────────────┘
                            │ 结构化经验 JSON
                            ▼
┌───────────────────────────────────────────────────────┐
│  Stage 4: INGEST                                      │
│  结构校验 → Approach 归一化 → 语义去重 → Neo4j 写入       │
└───────────────────────────────────────────────────────┘
```

每个阶段结果写入 MySQL，完成状态可追溯。任务暂停或中断后，从当前阶段的 MySQL 状态恢复，**已完成的阶段不重跑**。

---

## 1. Stage 1: FETCH — 对话拉取与去重

### 1.1 做什么

从数据 API 拉取指定销售顾问与用户之间的对话记录，格式化为纯文本，计算去重哈希，过滤重复对话后存入 MySQL。

### 1.2 执行流程

```
对每个 (user_id, counselor_id, 日期范围) 样本：
  │
  ├─ 调用数据 API（带 QPS 限流）
  │    接口返回: [ {msg_from: "老师", text: "..."}, ... ]
  │
  ├─ 格式化为纯文本:
  │    "老师: ...\n用户: ...\n老师: ..."
  │
  ├─ 计算 SHA256(dialogue_text) = dialogue_hash
  │
  ├─ 去重检查 1：任务内去重
  │    本次任务已出现相同 hash → 跳过
  │
  ├─ 去重检查 2：历史去重
  │    曾经有任何任务成功处理过该 hash → 跳过
  │
  └─ 写入 distill_conversations（task_id, sample_id, dialogue_text, hash, msg_count）
```

### 1.3 关键设计

- **QPS 限流：** `[hetao_data] qps=5`，令牌桶，防止接口过载
- **SHA256 去重：** 不依赖 (user_id, counselor_id, 日期) 组合，哈希更可靠
- **两层去重：** 任务内 + 历史（防止同一对话经验多次写入图谱）

---

## 2. Stage 2: KEY MOMENTS — 经验密度判断

### 2.1 做什么

用 LLM 判断每段对话的「经验密度」：这段对话中有多少值得提炼的销售判断时刻？密度低的对话在进入昂贵的经验提取阶段之前被提前过滤掉。

### 2.2 LLM 输出结构

```json
{
  "experience_density": "high",     // high | medium | low
  "worth_extracting": true,
  "key_moments": [
    "顾问在用户表达价格顾虑时，主动切换到价值锚定策略",
    "用户问了具体开课时间，顾问把握住了需求明确信号"
  ],
  "reasoning": "..."
}
```

### 2.3 阈值过滤

| 配置阈值 | 保留的密度等级 |
|---------|--------------|
| `high`   | 仅 high |
| `medium`（默认） | high + medium |
| `low`   | high + medium + low（全保留） |

`retained` 字段标记是否进入下一阶段，过滤逻辑与具体密度值解耦，便于业务调整。

### 2.4 幂等保证

`distill_key_moments` 有 `UNIQUE KEY (task_id, conversation_id)`。任务恢复时，跳过已有记录，仅处理新对话。

---

## 3. Stage 3: DISTILL — 经验提取

### 3.1 做什么

对每段「已保留」的对话，调用 LLM 提取完整结构化经验列表。LLM 必须遵循全局 Tag Catalog 中的 Factor / Topic 标签命名，不得自造标签（除非申请新增）。

### 3.2 Tag Catalog 注入

```
阶段启动时：
  ├─ 从 Neo4j 读取所有 Factor 节点（positive_factors + negative_factors 分列）
  ├─ 从 Neo4j 读取所有 Topic 节点
  ├─ 合并内置默认 Factor 列表
  └─ 将目录文本注入 Prompt 模板（{positive_factor_catalog}, {negative_factor_catalog}, {topic_catalog}）
```

**为什么重要：** 不同任务的 LLM 调用共享同一套标签，保证经验库中「价格敏感」和「价格顾虑」不会变成两个孤立节点。

详见 [04-tag-normalization.md](04-tag-normalization.md)

完整字段定义与示例见 [02-experience-model.md](02-experience-model.md)

### 3.4 标签校验与重试

LLM 使用了未在目录中的 Factor / Topic 时，系统自动触发一次重试，在 Prompt 中加入「以下标签不在目录中，请修正或提供充分理由」的提示。重试后仍然包含非目录标签，但有充分证据说明的，允许作为新标签写入。

---

## 4. Stage 4: INGEST — 归一化与图谱写入

### 4.1 做什么

将 Stage 3 输出的经验 JSON 逐条写入 Neo4j。每条经验在写入前经过三道门控：

```
经验 JSON
  │
  ├─ 门控 1: 结构校验（validate_experience）
  │    experience_type 合法 / from_stage ≤ to_stage /
  │    evidence 三段完整 / execution.approach 非空
  │    → 不通过：status=failed，记录原因，跳过
  │
  ├─ 门控 2: Approach 归一化
  │    从 Neo4j 检索候选 Approach → LLM 判断 matched/revise/new
  │    → 失败：降级为 new，不中断
  │
  ├─ 门控 3: 语义去重
  │    同 Approach + 同路径(from→to) + Factor 重叠度 ≥ 阈值
  │    → 命中：status=skipped，不写图谱
  │
  └─ 写入 Neo4j
       ├─ MERGE 节点：IntentStage / Factor / Approach / Topic
       ├─ CREATE 节点：Experience
       └─ MERGE 关系：TRANSITION / BLOCKS|DRIVES / RESOLVES|LEVERAGES / EVIDENCE_OF / INVOLVES
```

### 4.2 Approach 归一化三种结果

| 结果 | 含义 | 图谱操作 |
|------|------|---------|
| `matched` | 经验与已有 Approach 高度一致 | 复用 approach_id，新增 EVIDENCE_OF 边 |
| `revise` | 已有 Approach 名称/描述需更新 | MERGE 更新节点属性 |
| `new` | 全新打法，无可匹配 | Counter 节点自增，CREATE 新 Approach |

详见 [04-tag-normalization.md](04-tag-normalization.md)

### 4.3 语义去重

**触发条件（三者同时满足）：**
1. approach_id 相同（同一 Approach 节点）
2. from_stage 和 to_stage 均相同（同一转移路径）
3. factors_addressed 的 Jaccard 相似度 ≥ 配置阈值（默认 0.5）

**每个 Approach + 路径组合最多保留 N 条经验**（配置项 `max_similar_experiences`，默认 3）。

这一机制防止图谱被大量高度相似的「废话」经验撑大，稀释统计信号。

### 4.4 Neo4j 建模适配

BLOCKS / DRIVES 语义上「指向 TRANSITION 边」，但 Neo4j 不支持边指向边。解决方案：将路径信息冗余到边属性：

```cypher
(Factor {name: '价格敏感'})-[:BLOCKS {from_stage: 'S2', to_stage: 'S3'}]->(IntentStage {name: 'S3'})
(Approach {id: 'AP-001'})-[:RESOLVES {for_from_stage: 'S2', for_to_stage: 'S3'}]->(Factor {name: '价格敏感'})
```

---

## 5. 断点续跑机制

### 5.1 实现原理

每个阶段写入 MySQL 前检查「是否已存在」：

```
Stage 1 恢复：跳过 distill_conversations 中已有相同 hash 的记录
Stage 2 恢复：跳过 distill_key_moments 中已有 (task_id, conversation_id) 的记录
Stage 3 恢复：跳过 distill_distillations 中已有 (task_id, conversation_id) 的记录
Stage 4 恢复：跳过 distill_ingestions 中已有 (task_id, experience_id) 的记录
```

所有唯一键均在 DDL 中声明，INSERT 使用 `ON DUPLICATE KEY UPDATE` 或先查后跳过。

### 5.3 暂停 / 恢复 / 取消

```
暂停（pause）：  写 status='paused'，下一个控制检查点停止迭代
恢复（resume）： 原子地将 status 从 paused 改为 running，启动新后台线程
取消（cancel）： 写 status='cancelled'，中间结果保留，不可再恢复
```

控制检查点位于每个阶段「处理下一条对话/经验之前」，粒度为单条记录。

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [00-overview.md](00-overview.md) | 系统概述与整体架构 |
| [02-experience-model.md](02-experience-model.md) | 经验数据模型（五种类型 / 字段结构 / 验证规则） |
| [03-graph-schema.md](03-graph-schema.md) | 知识图谱结构（节点 / 边 / 查询） |
| [04-tag-normalization.md](04-tag-normalization.md) | 标签目录与打法归一化 |
| [05-task-management.md](05-task-management.md) | 任务管理与 MySQL Schema |
| [06-api-deployment.md](06-api-deployment.md) | API 接口与部署 |
