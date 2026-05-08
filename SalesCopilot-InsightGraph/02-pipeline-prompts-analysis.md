# 流水线、Prompt 与分析查询

---

## 0. 流水线总览

```text
CSV 上传
  │
  ▼
Stage 1: VALIDATE_INPUT
  校验必填字段、字段配置、类型转换、行级错误
  │
  ▼
Stage 2: FETCH_DIALOGUE
  按 user_id + counselor_id + perform_stage 分阶段拉取对话
  将各阶段文本带阶段标题拼接为完整学期 fulltext
  │
  ▼
Stage 3: EXTRACT_SIGNALS
  注入 L1 预设目录 + L2 图谱已有目录，LLM 抽取阶段、话题、异议、机会、动作、局部结果
  │
  ▼
Stage 4: NORMALIZE_VALIDATE
  标签归一化、结构校验、证据校验、课程阶段归属校验
  │
  ▼
Stage 5: INGEST_GRAPH
  写入独立 Neo4j 分析图谱
  │
  ▼
Stage 6: SUMMARIZE
  输出任务质量、字段完整率、图谱写入统计
```

每个阶段结果写入中间存储，任务失败或暂停后可从断点恢复。

---

## 1. 关键处理规则

### 1.1 VALIDATE_INPUT

- 校验 `user_id / counselor_id` 必填。
- 校验字段入库配置和 PII 排除规则。
- 解析可选参考日期、Bool、数值等类型；金额相关字段即使存在也不进入分析配置。
- 生成 `window_key = sha256(user_id + counselor_id + term_name?)`。
- 读取 `perform_stages` 配置。

行级状态：

| 状态 | 含义 |
|------|------|
| `valid` | 可进入对话拉取 |
| `failed_required_field` | 必填字段为空 |
| `warning_date_parse` | 参考日期解析失败，但窗口仍可处理 |
| `failed_config` | 配置列不存在或被 PII 规则拒绝 |
| `warning_optional_parse` | 可选字段解析失败，但窗口仍可处理 |

缺失处理：

| 场景 | 处理 |
|------|------|
| 缺少必填列 | 任务创建失败 |
| 某行 `user_id/counselor_id` 为空 | 该行失败，不拉取对话 |
| `term_name` 缺失 | 仍处理，不创建 `Term` 关系 |
| `start_date/end_date` 缺失或解析失败 | 仍处理，仅记录字段错误 |
| 业务结果字段缺失 | 创建 `BusinessOutcome`，对应状态标记为 `unknown` |
| 配置列不存在 | 任务创建失败 |
| CSV 中出现未配置列 | 忽略，不写 Neo4j |

### 1.2 FETCH_DIALOGUE

单阶段请求：

```json
{
  "user_id": "10001",
  "counselor_id": "8001",
  "perform_stage": "首课前"
}
```

输出：

```json
{
  "window_key": "CW-...",
  "stage_segments": [
    {
      "perform_stage": "首课前",
      "dialogue_text": "老师: ...\n用户: ...",
      "segment_hash": "sha256...",
      "message_count": 42,
      "fetch_status": "ok"
    }
  ],
  "fulltext": "【perform_stage: 首课前】\n老师: ...\n用户: ...",
  "dialogue_hash": "sha256...",
  "message_count": 98
}
```

拼接规则：

- 按 `perform_stages` 配置顺序拼接。
- 每段文本前必须加入 `【perform_stage: xxx】` 标题。
- 空阶段保留标题和 `fetch_status=empty`。
- 单阶段文本仍保存在中间表，便于回溯。

### 1.3 EXTRACT_SIGNALS

LLM 输入包含：

- 带 `perform_stage` 标题的完整学期 fulltext
- CSV 字段摘要
- L1 预设目录和 L2 图谱已有目录组成的 Tag Catalog
- 输出 JSON Schema
- 局部结果推断规则

硬性要求：

- LLM 必须允许一个 fulltext 中出现多个意向阶段、多个异议、多个机会和多次局部结果。
- CSV 中的 `converted` 只作为外部业务结果，不要求 LLM 推断。
- LLM 不输出 confidence。
- 金额相关信息不抽取、不分析。

### 1.4 NORMALIZE_VALIDATE

| 检查 | 失败处理 |
|------|----------|
| JSON 可解析 | 该窗口抽取失败 |
| 顶层字段存在 | 缺关键字段则失败 |
| 阶段值属于 S0-S4 | 非法阶段失败 |
| 标签经过 L1/L2 匹配或已申报 | 未申报目录外标签失败 |
| 不包含 confidence 字段 | 出现 confidence 则失败 |
| 证据文本非空 | 对应观察失败 |
| LocalOutcome 有 after evidence | 否则结果必须为 `unknown` |
| 观察结果包含 `perform_stage` | 缺失且无原因则失败 |
| SalesAction speaker 是老师 | 否则该动作失败 |

观察级状态：

| 状态 | 含义 |
|------|------|
| `ok` | 校验通过，可入图 |
| `failed_schema` | 结构不合法 |
| `failed_evidence` | 缺证据 |
| `failed_label` | 标签非法 |
| `failed_forbidden_confidence` | 输出了 confidence 字段 |
| `failed_stage_missing` | 课程阶段归属缺失 |
| `pending_label_review` | 新标签待审核 |

### 1.5 INGEST_GRAPH

写入顺序：

```text
1. MERGE ExtractionRun
2. MERGE User / Counselor / Term
3. MERGE ConversationWindow
4. MERGE StageSegment
5. MERGE BusinessOutcome
6. MERGE IntentObservation / Topic / Factor
7. MERGE Objection / Opportunity / SignalObservation
8. MERGE SalesAction
9. MERGE LocalOutcome
10. MERGE relationships
```

写入失败：

- 单个窗口失败时记录 `graph_ingestion_status='failed'`。
- 保存错误消息和失败节点类型。
- 继续处理下一窗口。

任务最终状态：

- 所有窗口都失败：`failed`
- 部分窗口失败：`done_with_errors`
- 无失败：`done`

---

## 2. 中间表与任务状态

任务状态机：

```text
pending -> running -> done
                   -> done_with_errors
                   -> failed
                   -> paused -> running
                   -> cancelled
```

中间表建议：

| 表 | 作用 | 幂等键 |
|----|------|--------|
| `insight_runs` | 任务主表 | `run_id` |
| `insight_input_rows` | CSV 行解析结果 | `run_id + row_number` |
| `insight_windows` | 学期沟通对象与 fulltext 哈希 | `window_key` |
| `insight_stage_segments` | 分阶段取数结果与文本哈希 | `window_key + perform_stage` |
| `insight_llm_outputs` | LLM 原始输出 | `run_id + window_key` |
| `insight_observations` | 校验后的观察结果 | `observation_id` |
| `insight_graph_ingestions` | 图谱写入状态 | `run_id + window_key` |
| `insight_label_reviews` | 新标签审核队列 | `label_type + label_name` |

任务摘要：

```json
{
  "row_count": 10000,
  "valid_window_count": 9820,
  "dialogue_fetched_count": 9700,
  "stage_fetch_success_rate": 0.96,
  "no_dialogue_count": 120,
  "llm_success_count": 9400,
  "llm_failed_count": 300,
  "observations_ok": 68000,
  "observations_failed": 2100,
  "local_outcomes_ok": 18500,
  "business_outcomes_written": 9820,
  "new_labels_requested": 36,
  "pending_label_review": 19,
  "graph_write_failed_count": 25
}
```

---

## 3. Prompt 管理

项目 Prompt 统一沉淀在本文档中。代码只引用 `prompt_id` 和 `prompt_version`，不散落维护大段 Prompt。

Prompt 列表：

| Prompt ID | 用途 | 输入 | 输出 |
|-----------|------|------|------|
| `IG-EXTRACT-V1` | 从完整学期 fulltext 中抽取分析事实 | CSV 字段摘要、分阶段 fulltext、L1/L2 Tag Catalog | 结构化观察结果 JSON |
| `IG-REPAIR-V1` | 修复结构校验失败的 JSON | 原始输出、错误列表、L1/L2 Tag Catalog | 修复后的 JSON |
| `IG-LABEL-NORMALIZE-V1` | 对新标签进行归一化建议 | 新标签、L1 预设目录、L2 图谱已有目录、证据 | matched/new/reject 建议 |
| `IG-QUALITY-REVIEW-V1` | 对抽取结果做规则化审阅 | 抽取 JSON、校验规则 | accept/reject/warnings |

通用要求：

- 只输出 JSON。
- 不要输出 confidence，也不要输出任何表达置信度的字段。
- 不要推断最终是否转化。
- 不要抽取或分析金额相关信息。
- 一个学期内可以存在多个意向阶段、多个异议、多个机会信号、多个老师动作和多个局部处理结果。
- 每条观察、异议、机会、动作、局部结果都必须尽量标注 `perform_stage`。
- 所有异议处理和机会利用的局部结果都必须有 before/action/after 证据；没有 after 证据时，结果只能是 `unknown`。
- 类别、命名和标签归属必须先匹配 L1 预设目录，再匹配 L2 图谱已有目录；两层都无可匹配时，才允许写入 `new_labels_requested` 并说明原因。

### 3.1 IG-EXTRACT-V1

System Prompt：

```text
你是 SalesCopilot-InsightGraph 的沟通记录分析抽取器。

你的任务是从一个用户与一个班主任在一个学期内的完整沟通记录中，抽取可用于运营和管理分析的结构化事实。

你必须遵守：
1. 只输出 JSON，不输出解释性文字。
2. 不要输出 confidence，也不要输出任何表达置信度的字段。
3. 不要推断最终是否转化；最终转化结果只来自 CSV。
4. 一个学期内可以存在多个意向阶段、多个异议、多个机会信号、多个老师动作和多个局部处理结果。
5. 每条观察、异议、机会、动作、局部结果都必须尽量标注 perform_stage。
6. 所有异议处理和机会利用的局部结果都必须有 before/action/after 证据；没有 after 证据时，结果只能是 unknown。
7. 类别、命名和标签归属必须先匹配 L1 预设目录，再匹配 L2 图谱已有目录；两层都无可匹配时，才允许写入 new_labels_requested 并说明原因。
8. 不要抽取或分析金额相关信息。
```

User Prompt：

```text
请分析以下学期沟通记录，并按指定 JSON Schema 输出。

【CSV 上下文】
{csv_context}

【L1 预设目录】
{preset_catalog}

【L2 图谱已有目录】
{graph_catalog}

【LocalOutcome Schema】
{local_outcome_schema}

【分阶段完整沟通记录】
{stage_fulltext}

要求：
- 你需要识别完整学期内的多个阶段和多个信号，不要只取最后状态。
- 如果同一类异议多次出现，应按出现时机和处理动作拆成多条观察。
- 如果老师动作跨阶段产生结果，需要在 LocalOutcome 中分别标注 before/action/after 的 perform_stage。
- 如果某个阶段没有有效沟通，也要在 window_summary.stage_segments 中体现。
- 不要输出 confidence。
```

### 3.2 输出 Schema 摘要

```json
{
  "window_summary": {
    "summary": "string",
    "message_count": 0,
    "effective_interaction": true,
    "stage_segments": [
      {
        "perform_stage": "首课前",
        "message_count": 0,
        "segment_status": "ok | empty | low_context"
      }
    ],
    "warnings": []
  },
  "intent_observations": [],
  "stage_transitions": [],
  "topics": [],
  "factors": [],
  "objections": [],
  "opportunities": [],
  "sales_actions": [],
  "local_outcomes": [],
  "new_labels_requested": [],
  "quality": {
    "missing_context": [],
    "warnings": []
  }
}
```

### 3.3 辅助 Prompt 摘要

`IG-REPAIR-V1`：

- 根据校验错误修复 JSON。
- 不新增原文无证据支持的观察。
- 不输出 confidence。
- 缺少 after 证据时，将 `LocalOutcome.result` 改为 `unknown`。
- 标签不在 L1/L2 目录中时，替换为最接近目录标签；实在无可匹配时加入 `new_labels_requested`。

`IG-LABEL-NORMALIZE-V1`：

```json
{
  "decision": "matched | new | reject",
  "matched_label": "string|null",
  "normalized_name": "string|null",
  "reason": "string"
}
```

`IG-QUALITY-REVIEW-V1`：

```json
{
  "decision": "accept | reject",
  "errors": [],
  "warnings": []
}
```

版本规则：

- 输出字段或枚举变化：升级 major。
- 措辞、示例、少量规则变化：升级 minor。
- 每次任务写入 `ExtractionRun.prompt_version`。
- 同一批任务内不得混用多个抽取 Prompt 版本。

---

## 4. 分析口径

| 结果 | 来源 | 用途 |
|------|------|------|
| `LocalOutcome` | LLM 根据后续对话推断 | 分析异议处理、机会利用、动作当下效果 |
| `BusinessOutcome` | 输入 CSV | 分析最终转化、续费等业务结果 |

口径要求：

- 「价格异议处理成功率」默认指 `LocalOutcome.result IN ['resolved', 'partially_resolved']`。
- 「价格异议最终转化率」指有价格异议的窗口中 `BusinessOutcome.converted=true` 的比例。
- 「局部成功但最终未转化」用于识别动作当下有效但后续链路断裂的场景。
- 按课程阶段分析时，使用观察节点上的 `perform_stage`，或回退到所属 `StageSegment`。
- 本项目不推断、不存储、不使用 confidence。

---

## 5. 核心查询模式

### 5.1 分析班主任薄弱点

```cypher
MATCH (c:Counselor {counselor_id: $counselor_id})-[:HANDLED]->(w:ConversationWindow)
MATCH (w)-[:HAS_ACTION]->(a:SalesAction)-[:PRODUCED_LOCAL_OUTCOME]->(lo:LocalOutcome)
OPTIONAL MATCH (a)-[:ADDRESSES]->(o:Objection)
WITH coalesce(o.normalized_name, a.action_type) AS target,
     count(lo) AS total,
     sum(CASE WHEN lo.result IN ['resolved','partially_resolved','leveraged','partially_leveraged'] THEN 1 ELSE 0 END) AS positive
WHERE total >= 5
RETURN target, total, positive, toFloat(positive) / total AS local_success_rate
ORDER BY local_success_rate ASC, total DESC
LIMIT 10;
```

### 5.2 对比班主任 A/B 销售特点

```cypher
MATCH (c:Counselor)-[:HANDLED]->(w:ConversationWindow)-[:HAS_ACTION]->(a:SalesAction)
WHERE c.counselor_id IN [$a_id, $b_id]
WITH c.counselor_id AS counselor_id, a.action_type AS action_type, count(*) AS action_count
RETURN counselor_id, action_type, action_count
ORDER BY counselor_id, action_count DESC;
```

动作效果对比：

```cypher
MATCH (c:Counselor)-[:HANDLED]->(:ConversationWindow)-[:HAS_ACTION]->(a:SalesAction)-[:PRODUCED_LOCAL_OUTCOME]->(lo:LocalOutcome)
WHERE c.counselor_id IN [$a_id, $b_id]
WITH c.counselor_id AS counselor_id,
     a.action_type AS action_type,
     count(lo) AS total,
     sum(CASE WHEN lo.result IN ['resolved','partially_resolved','leveraged','partially_leveraged'] THEN 1 ELSE 0 END) AS positive
RETURN counselor_id, action_type, total, toFloat(positive) / total AS local_success_rate
ORDER BY action_type, local_success_rate DESC;
```

### 5.3 按学期和课程阶段统计意向阶段构成

```cypher
MATCH (t:Term)<-[:IN_TERM]-(w:ConversationWindow)-[:HAS_INTENT]->(io:IntentObservation)
WHERE t.name IN $term_names
WITH t.name AS term_name, io.perform_stage AS perform_stage, io.stage AS stage, count(DISTINCT w) AS window_count
RETURN term_name, perform_stage, stage, window_count
ORDER BY term_name, perform_stage, stage;
```

### 5.4 价格异议局部成功率趋势

```cypher
MATCH (t:Term)<-[:IN_TERM]-(w:ConversationWindow)-[:HAS_ACTION]->(a:SalesAction)-[:ADDRESSES]->(o:Objection)
MATCH (a)-[:PRODUCED_LOCAL_OUTCOME]->(lo:LocalOutcome)
WHERE o.normalized_name = '价格敏感'
WITH t.name AS term_name, a.perform_stage AS perform_stage,
     count(lo) AS total,
     sum(CASE WHEN lo.result IN ['resolved','partially_resolved'] THEN 1 ELSE 0 END) AS success
RETURN term_name, perform_stage, total, toFloat(success) / total AS price_objection_local_success_rate
ORDER BY term_name, perform_stage;
```

### 5.5 价格异议最终转化率趋势

```cypher
MATCH (t:Term)<-[:IN_TERM]-(w:ConversationWindow)-[:HAS_OBJECTION]->(o:Objection)
MATCH (w)-[:HAS_BUSINESS_OUTCOME]->(bo:BusinessOutcome)
WHERE o.normalized_name = '价格敏感' AND bo.converted IS NOT NULL
WITH t.name AS term_name,
     count(DISTINCT w) AS total,
     sum(CASE WHEN bo.converted = true THEN 1 ELSE 0 END) AS converted
RETURN term_name, total, toFloat(converted) / total AS final_conversion_rate
ORDER BY term_name;
```

### 5.6 近期增长的阻塞信号

```cypher
MATCH (t:Term)<-[:IN_TERM]-(w:ConversationWindow)-[:HAS_OBJECTION]->(o:Objection)
WHERE t.name IN [$previous_term, $current_term]
WITH t.name AS term_name, o.normalized_name AS objection_name, count(DISTINCT w) AS cnt
WITH objection_name,
     sum(CASE WHEN term_name = $previous_term THEN cnt ELSE 0 END) AS prev_cnt,
     sum(CASE WHEN term_name = $current_term THEN cnt ELSE 0 END) AS curr_cnt
WHERE curr_cnt >= 10
RETURN objection_name, prev_cnt, curr_cnt,
       CASE WHEN prev_cnt = 0 THEN null ELSE toFloat(curr_cnt - prev_cnt) / prev_cnt END AS growth_rate
ORDER BY growth_rate DESC, curr_cnt DESC
LIMIT 20;
```

---

## 6. 管理层问题清单

图谱需要能够稳定回答：

1. 分析班主任 xxx 的薄弱点。
2. 对比班主任 A 和 B 的销售特点以及优劣势。
3. 统计最近多个学期用户在课前的意向阶段构成。
4. 观测最近学期销售在解决价格异议上的局部成功率趋势。
5. 观测价格异议最终转化率趋势。
6. 最近哪些话题或阻塞信号在增长。
7. 最近哪些机会信号在增长。
8. 哪些动作高频但低效。
9. 哪些机会信号最容易被错过。
10. 哪些用户特征下最容易出现价格异议。
11. 哪些渠道用户最终转化率高但局部处理过程弱。
12. 哪些班主任在同类用户下表现优于团队均值。
13. 哪些局部处理成功但最终未转化。
14. 哪些局部处理失败但最终转化。
15. 哪些新增标签需要沉淀到标准目录或培训材料。
16. 不同课程阶段的主要异议、机会信号、老师动作有什么差异。
17. 哪个课程阶段最容易出现机会信号被错过。

---

## 7. 风险与应对

| 风险 | 应对 |
|------|------|
| CSV 字段命名混乱 | 强制字段入库配置，配置列不存在则任务失败 |
| 标签碎片化 | L1 预设目录 + L2 图谱已有目录 + 别名表 + 新标签审核 |
| LLM 过度推断局部结果 | 必须有 after evidence；无证据只能 unknown |
| LLM 输出 confidence | 结构校验直接拒绝 |
| 最终结果和局部结果混淆 | `LocalOutcome` 与 `BusinessOutcome` 分开建模 |
| 图谱体积过大 | 不建 Message 节点；证据存观察属性；原始对话留中间存储 |
| 重复上传 | `window_key`、`dialogue_hash`、观察 ID 幂等 |
| 单点失败拖垮整批 | 行级、窗口级、观察级独立失败记录 |
