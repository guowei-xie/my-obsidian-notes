# 数据基建与标签体系

---

## 标签体系

### 设计原则

标签分为两大类：**事实标签**（可观测、确定性的）和**推断标签**（AI 推理、概率性的）。

推断标签内部有严格的层次关系：**意向 = 话题 + 因素** — 意向不是独立判定的，而是从话题和因素中推导出来的。三层推导结构（话题 → 因素 → 意向）的完整定义与示例见 [01-scenarios.md](01-scenarios.md) 场景 1。

```
事实标签（确定性）              推断标签（概率性）
┌─────────────────┐           ┌─────────────────────────────────┐
│  T1: 属性标签    │           │  T3: 推断标签（三层结构）           │
│  用户画像、渠道   │           │                                  │
│  来源于：数仓     │           │  ┌─ 话题层（在聊什么）             │
├─────────────────┤           │  │                               │
│  T2: 行为标签    │           │  ├─ 因素层（信号是正是负）          │
│  埋点、履约数据   │           │  │                              │
│  来源于：埋点     │           │  └─ 意向层（综合判定）             │
└─────────────────┘           │     由话题×因素推导 → S0-S4        │
                              │                                 │
                              │  来源于：LLM，可阶段性更新          │
                              └─────────────────────────────────┘
```

### T1: 属性标签（用户画像）

| 标签名         | 示例值            |
| ----------- | -------------- |
| 城市等级        | 一线/新一线/二线/三线+  |
| 年级          | 1-9 年级         |
| 渠道来源        | 直播             |
| 设备类型        | iPad/PC/Mac    |
| Leads 评分    | 0-100 连续值      |
| 是否复购        | 是/否            |
| 来源直播间       | 直播间 ID + 主题    |
| **直播间内容摘要** | **直播主题、卖点关键词** |
| **客服问卷结果**  | **问卷填写内容**     |
| **客服沟通摘要**  | **初始诉求、关键信息**  |
| **往期记录摘要**      | **LLM总结的往期用户摘要**   |

### T2: 行为标签（埋点 + 履约）

#### 底层：行为事件表

行为标签的计算基础。每条记录 = 一个用户行为事件。完整表结构见 [表 1: 事实行为表](#表-1-事实行为表-sales_copilot_fact_behavior)。

**行为标签相关的事件名称枚举：**

| event_name | 含义 | 来源 |
|------------|------|------|
| `class_attend` | 到课 | 履约数据 |
| `class_complete` | 完课 | 履约数据 |
| `homework_submit` | 作业提交 | 埋点 |
| `creation_save` | 作品保存 | 埋点 |
| `creation_publish` | 作品发布 | 埋点 |
| `parent_app_visit` | 家长端访问 | 埋点 |
| `im_reply` | 家长回复消息 | IM 记录 |
| `class_interact` | 课堂互动（举手/连麦/答题） | 课堂数据 |

#### 聚合层：行为标签

基于行为事件表统计产出，每个标签 = 一条聚合规则。

| 标签名    | 来源事件                            |
| ------ | ------------------------------- |
| 体验课完课率 | class_attend, class_complete    |
| 作品提交次数 | homework_submit                 |
| 创作活跃度  | creation_save, creation_publish |
| 家长端活跃度 | parent_app_visit                |
| 回复响应速度 | im_reply                        |
| 课堂互动频率 | class_interact, class_attend    |

### T3: 推断标签（AI 生成）

话题、因素、意向的完整定义与推导逻辑见 [01-scenarios.md](01-scenarios.md) 场景 1「标签提取的三层结构」与「意向分类体系」。本节仅关注数据建模层面。

**存储要点：**
- 每次推理产出一条完整快照，包含话题×因素明细（证据层）和意向判定（结论层）
- 积极/消极因素从 `topics_factors` 中冗余汇总，便于查询和聚合
- 保留模型版本与源对话 ID，支持溯源和版本对比

具体字段设计见 [表 2: 推断标签表](#表-2-推断标签表-sales_copilot_inferred_tags)。

---


## 建表结构

### 表 1: 事实行为表 `sales_copilot_fact_behavior`

记录用户行为事实（每条记录 = 一个行为事件）。

| 字段名 | 类型 | 含义 | 示例 |
|--------|------|------|------|
| `id` | BIGINT, PK | 自增主键 | |
| `user_id` | VARCHAR(32) | 用户 ID | |
| `class_id` | VARCHAR(32) | 班级 ID | |
| `counselor_id` | VARCHAR(32) | 课程顾问 ID | |
| `term_id` | VARCHAR(16) | 学期 ID | "2026spring" |
| `perform_stage` | ENUM | 履约阶段 | 见下方枚举 |
| `event_name` | VARCHAR(64) | 行为事件名称 | "call_made", "wechat_sent" |
| `event_time` | DATETIME | 事件发生时间 | |
| `event_source` | VARCHAR(32) | 事件来源 | "sensors", "im", "crm" |
| `event_metadata` | JSON | 事件附加信息 | `{"duration_sec": 180}` |
| `created_at` | DATETIME | 记录创建时间 | |

**`perform_stage` 枚举值：**
- `pre_first_class`: 进班至首课开始前
- `first_to_renewal`: 首课开始至续报开始前
- `renewal_period`: 续报开始后

**索引：** `(user_id, perform_stage, event_time)`, `(counselor_id, perform_stage)`

### 表 2: 推断标签表 `sales_copilot_inferred_tags`

AI 推理的用户标签（每条记录 = 一次推理结果的完整快照）。

| 字段名                   | 类型          | 含义                 | 示例                   |
| --------------------- | ----------- | ------------------ | -------------------- |
| `id`                  | BIGINT, PK  | 自增主键               |                      |
| `user_id`             | VARCHAR(32) | 用户 ID              |                      |
| `class_id`            | VARCHAR(32) | 班级 ID              |                      |
| `counselor_id`        | VARCHAR(32) | 课程顾问 ID            |                      |
| `term_id`             | VARCHAR(16) | 学期 ID              |                      |
| `perform_stage`       | ENUM        | 履约阶段               | 同上                   |
| `topics_factors`      | JSON        | 话题×因素明细（证据层）       | 见下方示例                |
| `positive_factors`    | JSON        | 积极因素汇总             | 见下方示例                |
| `negative_factors`    | JSON        | 消极因素汇总             | 见下方示例                |
| `customer_intention`  | ENUM        | 意向分类（结论层，由话题×因素推导） | "S0"-"S4"            |
| `intention_reason`    | TEXT        | 意向推导理由             |                      |
| `model_version`       | VARCHAR(32) | 模型版本               | "v1.0-qwen"          |
| `source_dialogue_ids` | JSON        | 源对话 ID             | `["d_001", "d_002"]` |
| `inferred_at`         | DATETIME    | 推理时间               |                      |
| `created_at`          | DATETIME    | 记录创建时间             |                      |

**`topics_factors` JSON 示例（证据层：话题×因素的完整推导链）：**
```json
[
  {
    "topic": "学习情况",
    "factors": [
      {"polarity": "positive", "tag": "学习兴趣", "proof": "孩子数学不错，很喜欢编程"}
    ]
  },
  {
    "topic": "价格费用",
    "factors": [
      {"polarity": "positive", "tag": "主动询价", "proof": "这个课程多少钱？"},
      {"polarity": "negative", "tag": "价格敏感", "proof": "感觉有点贵"}
    ]
  },
  {
    "topic": "时间安排",
    "factors": [
      {"polarity": "negative", "tag": "时间冲突", "proof": "周末排得满满的"}
    ]
  }
]
```

**`positive_factors` / `negative_factors` JSON 示例（从 topics_factors 中汇总提取）：**
```json
// positive_factors — 所有 polarity=positive 的因素汇总
[
  {"tag": "学习兴趣", "proof": "孩子数学不错，很喜欢编程"},
  {"tag": "主动询价", "proof": "这个课程多少钱？"}
]

// negative_factors — 所有 polarity=negative 的因素汇总
[
  {"tag": "价格敏感", "proof": "感觉有点贵"},
  {"tag": "时间冲突", "proof": "周末排得满满的"}
]

// intention_reason — 综合推导
// "用户在学习情况话题下表现出学习兴趣（积极），主动询问价格（强购买信号），
//  但同时存在价格敏感和时间冲突。综合判断：积极因素主导，有明确需求但存在可化解阻塞。"
// → customer_intention: S3（需求明确）
```

**索引：** `(user_id, perform_stage, inferred_at)`, `(counselor_id, perform_stage)`


### 表 3: 策略推荐表 `sales_copilot_strategies`

AI 生成的策略记录（用于追踪策略效果）。

| 字段名 | 类型 | 含义 | 示例 |
|--------|------|------|------|
| `id` | BIGINT, PK | 自增主键 | |
| `user_id` | VARCHAR(32) | 用户 ID | |
| `counselor_id` | VARCHAR(32) | 课程顾问 ID | |
| `term_id` | VARCHAR(16) | 学期 ID | |
| `scenario` | ENUM | 场景 | "pre_first_class", "pre_renewal" |
| `intent_stage` | ENUM | 当前意向阶段 | "S2" |
| `target_stage` | ENUM | 目标意向阶段 | "S3" |
| `strategy_content` | JSON | 策略内容 | 完整策略卡片 JSON |
| `model_version` | VARCHAR(32) | 模型版本 | |
| `generated_at` | DATETIME | 生成时间 | |
| `was_viewed` | BOOLEAN | 是否被查看 | true |
| `was_applied` | BOOLEAN | 是否被采用（销售反馈） | |
| `outcome` | ENUM | 结果 | "converted", "not_converted", "pending" |
| `counselor_feedback` | TEXT | 销售反馈 | "建议有用" |
| `created_at` | DATETIME | 记录创建时间 | |

**索引：** `(counselor_id, scenario, generated_at)`, `(user_id, scenario)`
