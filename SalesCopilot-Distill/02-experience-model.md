# 经验数据模型

---

## 0. 经验的定义

一条经验 = 特定情境下的**情境-判断-方法-效果**结构化记录，有对话原文为证，可复用于相似情境的检索。

**不是经验：** SOP 模板消息、纯信息传递、evidence 三段缺失、approach 为空、from_stage > to_stage。

---

## 1. 五种经验类型

| 类型 | 标签 | 核心特征 | 示例 |
|------|------|---------|------|
| **意向推进** | `intent_advance` | 推动用户意向阶段向前跃迁 | S2→S3: 从「随便看看」引导到主动询问方案 |
| **阻塞化解** | `blocker_resolve` | 化解用户的顾虑或明确异议 | 化解「太贵了」→ 重建价值参照系后用户主动问报名方式 |
| **信号捕捉** | `signal_capture` | 捕捉并放大用户的积极信号 | 用户提到「朋友孩子在学」→ 强化社交背书 + 需求认知 |
| **节奏把控** | `pacing` | 在正确时机做正确的事，或选择不做 | S2 阶段不急于报价，先建立需求认知（「不做」同样是判断） |
| **信息承接** | `context_leverage` | 利用用户已有背景信息增强针对性 | 承接直播间信奥卖点，直接切入竞赛路径 |

> **节奏把控：** 常表现为「不做某事」，LLM 识别率较低，evidence.action 通常体现为有意引导话题转移。

---

## 2. 完整字段结构

### 2.2 context — 情境

```json
{
  "description": "用户（小学4年级，数学基础一般）主动提到价格贵，顾问判断为表面异议",
  "intent_stage": "S2",
  "intent_stage_reason": "用户有初步兴趣但尚未明确需求，价格顾虑是当前主要障碍",
  "positive_factors": ["学习兴趣高", "家长重视教育"],
  "negative_factors": ["价格敏感", "时间冲突顾虑"]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `description` | string | 情境自然语言描述（对话背景摘要） |
| `intent_stage` | enum S0-S4 | 经验发生时用户所处的意向阶段 |
| `intent_stage_reason` | string | 判断该阶段的理由 |
| `positive_factors` | string[] | 当时的积极驱动因素（必须来自 Tag Catalog） |
| `negative_factors` | string[] | 当时的消极阻塞因素（必须来自 Tag Catalog） |

### 2.3 judgment — 判断

```json
{
  "what": "顾问判断用户对价格的顾虑是表面异议而非真实障碍",
  "why_effective": "用户已经对学习效果表达过期待，价值认同已建立，价格只是决策的最后一道门",
  "alternative_avoided": "没有直接解释性价比（那样会强化价格锚定，让用户更关注数字）"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `what` | string | 顾问做出了什么判断（核心决策） |
| `why_effective` | string | 这个判断为什么在此情境下有效 |
| `alternative_avoided` | string? | 顾问刻意回避了什么（揭示判断的「非默认性」） |

### 2.4 execution — 执行

```json
{
  "approach": "先强化价值再处理价格异议",
  "how": "先回顾孩子课堂上表现出的学习兴趣，再类比孩子未来发展投入，最后才回应价格话题",
  "approach_id": "AP-007",
  "approach_match_type": "matched"
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `approach` | string | 打法名称（LLM 原始输出，后经归一化） |
| `how` | string | 具体执行方式（how，非 what） |
| `approach_id` | string? | 归一化后的 Neo4j Approach 节点 ID |
| `approach_match_type` | enum? | matched / revise / new（Stage 4 写入） |

### 2.5 effect — 效果

字符串，描述**用户状态的可观察变化**（必须在 evidence.after 有原文支撑，不接受主观感受）。

### 2.6 evidence — 证据

| 字段 | 类型 | 说明 |
|------|------|------|
| `before` | string | 顾问行动前的对话片段 |
| `action` | string | 顾问的具体行动原文 |
| `after` | string | 用户的反应变化原文 |
| `contrast_action` | string? | 对比案例动作（对比萃取时使用） |
| `contrast_after` | string? | 对比案例用户反应（对比萃取时使用） |

**三段（before/action/after）缺一不可，** `validate_experience` 会拒绝任何一段为空的经验。

### 2.7 retrieval_tags — 检索标签

```json
{
  "from_stage": "S2",
  "to_stage": "S3",
  "factors_addressed": ["价格敏感"],
  "topics": ["价格异议", "价值认知"],
  "applicable_segments": ["小学中年级", "首次体验用户"]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `from_stage` | enum S0-S4 | 经验起始意向阶段 |
| `to_stage` | enum S0-S4 | 经验目标意向阶段（≥ from_stage） |
| `factors_addressed` | string[] | 此经验处理的因素（阻塞或驱动） |
| `topics` | string[] | 涉及的对话话题（必须来自 Tag Catalog） |
| `applicable_segments` | string[] | 适用的用户细分（年级段等，可选） |

---

## 3. 验证规则

`validate_experience()` 的完整校验逻辑：

| 规则 | 检查内容 | 不通过时 |
|------|---------|---------|
| 类型合法 | `experience_type` ∈ {intent_advance, blocker_resolve, signal_capture, pacing, context_leverage} | status=failed |
| 阶段合法 | `from_stage` 和 `to_stage` ∈ {S0, S1, S2, S3, S4} | status=failed |
| 阶段方向 | `int(to_stage[1]) >= int(from_stage[1])` | status=failed |
| 证据完整 | `before` / `action` / `after` 均非空 | status=failed |
| 方法非空 | `execution.approach` 非空字符串 | status=failed |

通过全部规则后，经验才进入 Approach 归一化阶段。

---

## 4. experience_id 格式

```
EXP-{sample_id}-{idx}

sample_id = "{user_id}_{counselor_id}"
idx       = 该对话中第几条经验（从 0 开始）

示例: EXP-123_456-0
      EXP-123_456-1
```

同一对话中多条经验共享 sample_id，idx 区分顺序。Neo4j 中 experience_id 设置唯一约束。

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [00-overview.md](00-overview.md) | 系统概述与整体架构 |
| [01-pipeline.md](01-pipeline.md) | 四阶段蒸馏流水线 |
| [03-graph-schema.md](03-graph-schema.md) | 知识图谱结构（经验如何落入图谱） |
| [04-tag-normalization.md](04-tag-normalization.md) | 标签目录与打法归一化 |
| [05-task-management.md](05-task-management.md) | 任务管理与数据库设计 |
| [06-api-deployment.md](06-api-deployment.md) | API 接口与部署 |
