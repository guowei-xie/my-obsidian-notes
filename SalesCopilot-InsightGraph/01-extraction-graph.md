# 萃取模型与图谱 Schema

---

## 0. 萃取目标

InsightGraph 的萃取目标不是生成销售经验，而是把一个用户-班主任在一个学期内的完整沟通记录变成可统计、可对比、可追溯的过程事实。

需要回答：

- 用户在这个学期内经历过哪些意向阶段，分别发生在哪个课程阶段？
- 用户表达了哪些话题、诉求、异议、阻塞信号和机会信号？
- 班主任采取了哪些动作？
- 动作是否缓解了异议、放大了机会、推进了意向？
- 局部处理结果和 CSV 中的最终转化结果是否一致？

---

## 1. 信息分层

| 层级    | 信息                                            | 来源                     |
| ----- | --------------------------------------------- | ---------------------- |
| 主体层   | 用户、班主任、学期沟通对象                                 | CSV                    |
| 课程阶段层 | `首课前`、`续报开始前`、`续报开始后` 等分段文本                   | 接口按 `perform_stage` 返回 |
| 用户特征层 | 年级、渠道、城市、features_tag 等                       | CSV 配置字段               |
| 最终结果层 | renewed（续报，即转化） 等                             | CSV 配置字段               |
| 状态层   | 意向阶段、阶段变化、阶段判断依据                              | LLM 从对话推断              |
| 信号层   | 话题、异议、阻塞、机会、风险                                | LLM 从对话抽取              |
| 行为层   | 班主任动作、跟进节奏、承接方式                               | LLM 从对话抽取              |
| 局部结果层 | 异议是否缓解、机会是否利用、态度是否改善                          | LLM 根据后续对话推断           |
| 证据层   | before/action/after 原文、时间戳、所属 `perform_stage` | 对话原文 + LLM 标注          |

要求：

- 一个学期 fulltext 中允许出现多个意向阶段、多个话题、多个异议、多个机会信号和多次局部处理结果。
- 每条 `IntentObservation / Objection / Opportunity / SalesAction / LocalOutcome` 都应尽量标注 `perform_stage`。
- LLM 不需要也不允许输出 confidence。
- 最终转化结果只来自 CSV，不由 LLM 推断。

---

## 2. 核心萃取对象

### 2.1 IntentObservation

阶段枚举：

| 阶段 | 含义 | 典型信号 |
|------|------|----------|
| `S0` | 无有效互动 | 不回复、拒绝沟通、仅系统消息 |
| `S1` | 基础互动，无明确意向 | 回复寒暄、接受信息但无主动问题 |
| `S2` | 初步兴趣，需求模糊 | 询问课程、孩子情况、是否适合 |
| `S3` | 需求明确，主动比较或决策 | 问价格、排课、报名、名额、优惠 |
| `S4` | 强转化信号或已转化 | 明确报名、确认上课 |

示例：

```json
{
  "observation_id": "IO-{window_key}-0",
  "perform_stage": "首课前",
  "stage": "S2",
  "stage_reason": "用户主动询问课程是否适合四年级孩子",
  "observed_at": "2026-03-05T19:20:00",
  "evidence": [
    {
      "speaker": "user",
      "text": "这个课适合四年级吗？孩子之前没学过编程。",
      "timestamp": "2026-03-05T19:20:00"
    }
  ]
}
```

阶段变化：

```json
{
  "from_stage": "S2",
  "to_stage": "S3",
  "from_perform_stage": "首课前",
  "to_perform_stage": "续报开始前",
  "transition_reason": "用户从询问课程适配转为询问续报安排"
}
```

### 2.2 Topic、Factor、Objection、Opportunity

Topic 是对话主题，例如课程价格、课程效果、上课时间、课程难度、孩子兴趣、竞赛路径、试听体验、优惠政策。

Factor 是聚合分析用因素：

- 负向：价格敏感、时间冲突、决策权模糊、效果怀疑、信任不足、兴趣不足、竞品倾向、基础不匹配。
- 正向：主动询价、明确学习目标、孩子兴趣高、家长教育投入意愿强、社交背书、试听认可、竞赛需求、时间窗口明确。

Objection 示例：

```json
{
  "objection_id": "OBJ-{window_key}-0",
  "perform_stage": "续报开始前",
  "name": "价格异议",
  "normalized_name": "价格敏感",
  "severity": "medium",
  "explicitness": "explicit",
  "first_seen_at": "2026-03-07T20:10:00",
  "evidence_text": "感觉还是有点贵，我再考虑一下。"
}
```

Opportunity 示例：

```json
{
  "opportunity_id": "OPP-{window_key}-0",
  "perform_stage": "续报开始前",
  "name": "主动询价",
  "strength": "high",
  "evidence_text": "那这个课程现在报名是多少钱？",
  "observed_at": "2026-03-08T21:03:00"
}
```

### 2.3 SalesAction

老师动作必须能从原文观察。常见类型：

| 动作类型 | 说明 |
|----------|------|
| `value_reframe` | 重建价值认知，弱化价格锚点 |
| `need_discovery` | 继续挖掘孩子情况和家长期待 |
| `case_social_proof` | 使用案例、同龄人或结果背书 |
| `urgency_create` | 提醒名额、时间、优惠窗口 |
| `risk_reversal` | 用试听、退费、服务承诺降低风险 |
| `schedule_coordination` | 处理时间安排和排课冲突 |
| `decision_facilitation` | 帮助用户推动家庭内部决策 |
| `direct_close` | 直接推动报名或确认 |
| `follow_up` | 后续跟进、提醒、二次触达 |
| `information_answer` | 纯信息回复，如价格、时间、规则说明 |

示例：

```json
{
  "action_id": "ACT-{window_key}-0",
  "perform_stage": "续报开始前",
  "action_type": "value_reframe",
  "action_summary": "先回顾孩子试听中的积极表现，再解释课程长期价值",
  "target_type": "objection",
  "target_id": "OBJ-{window_key}-0",
  "action_text": "刚才孩子在解题时其实反应很快...",
  "acted_at": "2026-03-07T20:13:00"
}
```

### 2.4 LocalOutcome

`LocalOutcome` 由 LLM 根据动作后的用户反应推断，只表示过程效果，不替代 CSV 中的最终业务结果。

异议处理结果：

| 结果 | 含义 |
|------|------|
| `resolved` | 异议明显缓解，用户进入下一步具体讨论 |
| `partially_resolved` | 异议有所缓解，但仍保留顾虑 |
| `unresolved` | 异议未被缓解 |
| `amplified` | 处理不当导致异议更强 |
| `unknown` | 后续证据不足，无法判断 |

机会利用结果：

| 结果 | 含义 |
|------|------|
| `leveraged` | 抓住机会并推动用户进入更具体决策 |
| `partially_leveraged` | 有承接但推进不足 |
| `missed` | 用户释放机会信号但未有效承接 |
| `misused` | 承接方式不当，削弱机会 |
| `unknown` | 证据不足 |

示例：

```json
{
  "local_outcome_id": "LO-{window_key}-0",
  "target_type": "objection",
  "target_id": "OBJ-{window_key}-0",
  "action_id": "ACT-{window_key}-0",
  "result": "partially_resolved",
  "result_reason": "用户没有继续强调价格，但仍表示需要再考虑",
  "before": {
    "perform_stage": "续报开始前",
    "text": "感觉还是有点贵，我再考虑一下。"
  },
  "action": {
    "perform_stage": "续报开始前",
    "text": "刚才孩子在解题时其实反应很快..."
  },
  "after": {
    "perform_stage": "续报开始后",
    "text": "嗯我理解，那我回去和孩子爸爸说一下。"
  },
  "observed_after_at": "2026-03-07T20:18:00"
}
```

---

## 3. 图谱节点

### 3.1 主体节点

```cypher
(:User {user_id: '10001', grade: '四年级', channel: '直播间', leads_score: 72, features_tag1: '价格敏感'})
(:Counselor {counselor_id: '8001'})
(:Term {name: '2026春02期'})
```

`User` 只写 CSV 配置允许的属性。

### 3.2 ConversationWindow

```cypher
(:ConversationWindow {
  window_key: 'CW-9f2a...',
  user_id: '10001',
  counselor_id: '8001',
  start_date: '2026-03-01',
  end_date: '2026-06-30',
  dialogue_hash: 'sha256...',
  message_count: 138,
  effective_interaction: true
})
```

一个 `ConversationWindow` 表示完整学期沟通对象，允许连接多个课程阶段片段、多个意向观察、多个异议、多个机会和多个动作结果。

### 3.3 StageSegment

```cypher
(:StageSegment {
  segment_id: 'SEG-CW001-首课前',
  perform_stage: '首课前',
  message_count: 42,
  segment_hash: 'sha256...',
  fetch_status: 'ok'
})
```

`StageSegment` 用于支持按课程阶段分析。

### 3.4 分析事实节点

```cypher
(:IntentObservation {observation_id: 'IO-CW001-0', perform_stage: '首课前', stage: 'S2'})
(:Topic {name: '课程价格'})
(:Factor {name: '价格敏感', polarity: 'negative'})
(:Objection {objection_id: 'OBJ-CW001-0', perform_stage: '续报开始前', normalized_name: '价格敏感'})
(:Opportunity {opportunity_id: 'OPP-CW001-0', perform_stage: '续报开始前', name: '主动询价'})
(:SignalObservation {signal_id: 'SIG-CW001-0', perform_stage: '续报开始后', name: '长时间未回复'})
(:SalesAction {action_id: 'ACT-CW001-0', perform_stage: '续报开始前', action_type: 'value_reframe'})
(:LocalOutcome {local_outcome_id: 'LO-CW001-0', result: 'partially_resolved'})
(:BusinessOutcome {outcome_id: 'BO-CW001', converted: true, converted_at: '2026-06-20T12:30:00', source: 'input_csv'})
(:ExtractionRun {run_id: 'RUN-20260507-001', prompt_version: 'IG-EXTRACT-V1-1.0'})
```

`BusinessOutcome` 只保存 CSV 配置允许的最终业务结果字段，不由 LLM 推断。

---

## 4. 图谱关系

主体关系：

```cypher
(:User)-[:HAS_WINDOW]->(:ConversationWindow)
(:Counselor)-[:HANDLED]->(:ConversationWindow)
(:ConversationWindow)-[:IN_TERM]->(:Term)
(:ExtractionRun)-[:PROCESSED]->(:ConversationWindow)
(:ConversationWindow)-[:HAS_STAGE_SEGMENT]->(:StageSegment)
```

状态与信号关系：

```cypher
(:ConversationWindow)-[:HAS_INTENT]->(:IntentObservation)
(:ConversationWindow)-[:MENTIONS]->(:Topic)
(:ConversationWindow)-[:HAS_FACTOR]->(:Factor)
(:ConversationWindow)-[:HAS_OBJECTION]->(:Objection)
(:ConversationWindow)-[:HAS_OPPORTUNITY]->(:Opportunity)
(:ConversationWindow)-[:HAS_SIGNAL]->(:SignalObservation)

(:StageSegment)-[:HAS_INTENT]->(:IntentObservation)
(:StageSegment)-[:HAS_OBJECTION]->(:Objection)
(:StageSegment)-[:HAS_OPPORTUNITY]->(:Opportunity)
(:StageSegment)-[:HAS_SIGNAL]->(:SignalObservation)
(:StageSegment)-[:HAS_ACTION]->(:SalesAction)
```

归一化关系：

```cypher
(:Objection)-[:NORMALIZED_TO]->(:Factor)
(:Opportunity)-[:NORMALIZED_TO]->(:Factor)
(:SignalObservation)-[:NORMALIZED_TO]->(:Factor)
(:Objection)-[:ABOUT_TOPIC]->(:Topic)
(:Opportunity)-[:ABOUT_TOPIC]->(:Topic)
```

动作与结果关系：

```cypher
(:ConversationWindow)-[:HAS_ACTION]->(:SalesAction)
(:SalesAction)-[:ADDRESSES]->(:Objection)
(:SalesAction)-[:LEVERAGES]->(:Opportunity)
(:SalesAction)-[:PRODUCED_LOCAL_OUTCOME]->(:LocalOutcome)
(:LocalOutcome)-[:OUTCOME_OF]->(:Objection)
(:LocalOutcome)-[:OUTCOME_OF]->(:Opportunity)
(:ConversationWindow)-[:HAS_BUSINESS_OUTCOME]->(:BusinessOutcome)
```

---

## 5. 约束与索引

```cypher
CREATE CONSTRAINT user_id_unique IF NOT EXISTS
  FOR (u:User) REQUIRE u.user_id IS UNIQUE;

CREATE CONSTRAINT counselor_id_unique IF NOT EXISTS
  FOR (c:Counselor) REQUIRE c.counselor_id IS UNIQUE;

CREATE CONSTRAINT term_name_unique IF NOT EXISTS
  FOR (t:Term) REQUIRE t.name IS UNIQUE;

CREATE CONSTRAINT window_key_unique IF NOT EXISTS
  FOR (w:ConversationWindow) REQUIRE w.window_key IS UNIQUE;

CREATE CONSTRAINT segment_id_unique IF NOT EXISTS
  FOR (s:StageSegment) REQUIRE s.segment_id IS UNIQUE;

CREATE CONSTRAINT intent_observation_unique IF NOT EXISTS
  FOR (i:IntentObservation) REQUIRE i.observation_id IS UNIQUE;

CREATE CONSTRAINT objection_id_unique IF NOT EXISTS
  FOR (o:Objection) REQUIRE o.objection_id IS UNIQUE;

CREATE CONSTRAINT opportunity_id_unique IF NOT EXISTS
  FOR (o:Opportunity) REQUIRE o.opportunity_id IS UNIQUE;

CREATE CONSTRAINT action_id_unique IF NOT EXISTS
  FOR (a:SalesAction) REQUIRE a.action_id IS UNIQUE;

CREATE CONSTRAINT local_outcome_id_unique IF NOT EXISTS
  FOR (l:LocalOutcome) REQUIRE l.local_outcome_id IS UNIQUE;

CREATE CONSTRAINT business_outcome_id_unique IF NOT EXISTS
  FOR (b:BusinessOutcome) REQUIRE b.outcome_id IS UNIQUE;

CREATE CONSTRAINT topic_name_unique IF NOT EXISTS
  FOR (t:Topic) REQUIRE t.name IS UNIQUE;

CREATE CONSTRAINT factor_unique IF NOT EXISTS
  FOR (f:Factor) REQUIRE (f.name, f.polarity) IS UNIQUE;
```

推荐索引：

```cypher
CREATE INDEX stage_segment_perform_stage IF NOT EXISTS
  FOR (s:StageSegment) ON (s.perform_stage);

CREATE INDEX intent_stage IF NOT EXISTS
  FOR (i:IntentObservation) ON (i.stage);

CREATE INDEX action_type IF NOT EXISTS
  FOR (a:SalesAction) ON (a.action_type);

CREATE INDEX local_outcome_result IF NOT EXISTS
  FOR (l:LocalOutcome) ON (l.result);

CREATE INDEX business_converted IF NOT EXISTS
  FOR (b:BusinessOutcome) ON (b.converted);
```

---

## 6. 质量规则

必须拒绝：

- 没有原文证据的 `Objection / Opportunity / SalesAction / LocalOutcome`。
- `LocalOutcome` 没有 after evidence 但结果不是 `unknown`。
- LLM 推断最终转化结果。
- 使用目录外标签但没有申报新增标签。
- 输出任何 `confidence` 字段。
- `SalesAction` 不是老师动作。
- 观察结果缺少可定位的 `perform_stage`，且没有说明原因。

允许但需 warning：

- 对话过短，无法稳定判断意向阶段。
- 用户长时间沉默，局部结果只能为 `unknown`。
- 老师连续发多条消息，动作边界不清晰。
- 用户语义模糊，需要在 `warnings` 中说明不确定点。
- 多个异议同时出现，动作作用目标不唯一。

---

## 7. 类别与标签归属机制

凡涉及类别判断、名词命名或标签归属，都必须经过至少两层候选匹配，再决定是否创建新类别。

### 7.1 候选层级

| 层级 | 来源 | 说明 |
|------|------|------|
| L1 预设目录 | 系统内置标准目录 | 冷启动可用，包含稳定的 Topic、Factor、Objection、Opportunity、SalesAction、LocalOutcome 枚举 |
| L2 图谱已有目录 | Neo4j 中已沉淀的标准标签和别名 | 反映历史任务审核通过的标签，任务开始时读取成目录快照 |
| L3 新建申请 | LLM 申报 + 审核队列 | 只有 L1/L2 均无法表达时才允许进入 `new_labels_requested` |

独立维护分析型目录：

- Topic Catalog
- Factor Catalog
- Objection Catalog
- Opportunity Catalog
- SalesAction Catalog
- LocalOutcome Result 枚举

### 7.2 匹配规则

LLM 输出时必须按顺序处理：

1. 先从 L1 预设目录中选择最贴近类别或标签。
2. L1 不匹配时，从 L2 图谱已有目录和别名中选择。
3. L1/L2 都无法表达时，才允许写入 `new_labels_requested`。
4. 新建申请必须包含 `label_type`、`name`、`reason`、`evidence_text`。
5. 新标签在审核通过前不得并入标准聚合指标。

适用范围：

- 话题命名
- 异议类型判断
- 机会信号命名
- 阻塞/驱动因素归属
- 老师动作类型判断
- 局部结果枚举选择
- 任何后续新增的分析标签
