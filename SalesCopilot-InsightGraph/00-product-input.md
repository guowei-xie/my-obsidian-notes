# SalesCopilot-InsightGraph — 产品与输入设计

---

## 0. 产品定位

SalesCopilot-InsightGraph 是面向运营和管理层的离线分析萃取服务。它在一个学期结束后，批量读取用户与班主任在该学期内的全量沟通记录，抽取意向阶段、异议、机会信号、老师动作、局部处理结果等结构化事实，并写入一套独立 Neo4j 分析图谱。

它回答的问题包括：

- 分析班主任 xxx 的薄弱点。
- 对比班主任 A 和 B 的销售特点、优势与短板。
- 统计最近多个学期、不同课程阶段的用户意向阶段构成。
- 观察价格异议处理的局部成功率和最终转化趋势。
- 发现最近增长的话题、阻塞信号和机会信号。

---

## 1. 与 SalesCopilot-Distill 的边界

本服务只参考 `SalesCopilot-Distill` 的批处理、幂等、标签归一化、质量门控、断点续跑等机制，不复用它的经验图谱。

| 维度 | SalesCopilot-Distill | SalesCopilot-InsightGraph |
|------|----------------------|----------------------------|
| 目标 | 沉淀可复用销售经验 | 抽取全量沟通过程事实 |
| 图谱 | 经验检索图谱 | 独立分析图谱 |
| 核心对象 | Approach、Experience | User、Counselor、ConversationWindow、StageSegment、Objection、Opportunity、SalesAction、LocalOutcome、BusinessOutcome |
| 用途 | 给一线销售推荐打法 | 给运营和管理层做统计、对比、诊断、趋势分析 |

明确不做：

- 不与 Distill 图谱共用节点和关系。
- 不实时服务一线销售对话。
- 不生成销售建议。
- 不推断或存储 confidence。
- 不处理或分析金额相关字段。

---

## 2. CSV 输入契约

用户上传单个 CSV。每行表示一个用户与一个班主任在一个学期内的完整沟通对象。

必填字段：

```csv
user_id,counselor_id
```

常见可选字段：

```csv
term_name,start_date,end_date,grade,channel,city,features_tag1,features_tag2,renewed
```

约束：

- `user_id + counselor_id` 只可能对应一个学期，不做多学期匹配。
- `start_date / end_date` 如提供，只作为整体参考日期，通常表示开始进班到学期结束，不用于接口取数切分。
- 用户信息、特征字段和最终业务结果都由 CSV 提供。
- 金额相关字段不纳入当前产品处理和分析。

---

## 3. 入库白名单原则

CSV 可以携带任意用户特征和业务结果字段，但 Neo4j 只写入配置白名单中的字段。具体字段名、类型转换和缺失处理属于流水线配置与校验职责，见 [02-pipeline-prompts-analysis.md](02-pipeline-prompts-analysis.md)。

白名单按落库位置分组：

| 分组 | 写入位置 | 示例 |
|------|----------|------|
| 用户属性 | `User` | 年级、渠道、城市、特征标签 |
| 业务结果 | `BusinessOutcome` | 是否转化、是否续费、转化时间 |
| 窗口属性 | `ConversationWindow` | 批次、活动、参考开始/结束日期 |
| 学期字段 | `Term` 和窗口关系 | `term_name` |

原则：

- 必填字段只包含 `user_id / counselor_id`。
- `perform_stage` 列表由任务配置提供，不要求出现在 CSV 中。
- 未在白名单中的 CSV 列不写入 Neo4j。
- PII 排除字段优先级最高，即使被误加入白名单也拒绝写入。

---

## 4. 分阶段取数与 Fulltext

接口取数不再使用 `start_date/end_date` 判断窗口，而是使用 `perform_stage` 参数。系统对每行 CSV 自动按配置阶段发起多次请求：

```json
[
  {"user_id": "10001", "counselor_id": "8001", "perform_stage": "首课前"},
  {"user_id": "10001", "counselor_id": "8001", "perform_stage": "续报开始前"},
  {"user_id": "10001", "counselor_id": "8001", "perform_stage": "续报开始后"}
]
```

拼接 fulltext 时必须保留阶段标题：

```text
【perform_stage: 首课前】
老师: ...
用户: ...

【perform_stage: 续报开始前】
老师: ...
用户: ...

【perform_stage: 续报开始后】
老师: ...
用户: ...
```

设计含义：

- 一个 fulltext 表示完整学期沟通记录。
- 一个 fulltext 内允许出现多个意向阶段、多个异议、多个机会信号、多个老师动作和多个局部结果。
- 每条萃取结果都应尽量标注所属 `perform_stage`。
- 跨阶段处理结果需要分别记录 before/action/after 所属阶段。

---

## 5. CSV 示例

```csv
user_id,counselor_id,start_date,end_date,term_name,grade,channel,features_tag1,features_tag2,renewed
10001,8001,2026-03-01,2026-06-30,2026春02期,四年级,直播间,价格敏感,竞赛兴趣,1
10002,8001,2026-03-01,2026-06-30,2026春02期,三年级,转介绍,时间冲突,家长高参与,0
10003,8002,2026-03-01,2026-06-30,,五年级,自然流量,决策权模糊,,unknown
```

---

## 6. 验收要求

- 明确 CSV 必填字段只有 `user_id / counselor_id`。
- 明确用户特征和最终业务结果都来自 CSV。
- 明确字段是否写入 Neo4j 由白名单配置控制。
- 明确 `user_id + counselor_id` 唯一对应一个学期。
- 明确接口按 `perform_stage` 分阶段取数，拼接 fulltext 时保留阶段。
- 明确不需要 confidence。
- 明确不处理金额相关字段。
