# SalesCopilot-Distill（经验蒸馏服务）— 系统概述

---

## 0. 为什么需要一个独立的蒸馏服务

经验库的积累不能靠人工：
- 销售对话每天产生大量，人工审核跟不上速度
- 经验的结构化提取（情境 / 判断 / 方法 / 效果 / 证据）需要高度一致的格式
- 提取出的经验必须自动写入知识图谱，才能被检索侧使用

> **SalesCopilot-Distill 是经验库积累的离线基础设施。**
> 它不出现在用户界面里，但支撑着 GraphRAG 经验检索的运转。

---

## 1. 产品定义

### 一句话

将销售对话批量提炼为结构化经验，自动写入 Neo4j 知识图谱。

### 完整定义

输入：一批（counselor_id, user_id, 日期范围）的销售对话；
输出：Neo4j 中新增的 Approach 节点、Experience 节点及其关联边，每条经验有完整的情境、判断、方法、效果、证据标注。

**关键词拆解：**
- **批量**：支持 CSV 批量上传，也支持 JSON 接口单批请求
- **结构化**：LLM 提取的经验严格符合预定义 Schema，不合格的自动重试或跳过
- **自动写入图谱**：Approach 归一化 + 去重后直接 MERGE 进 Neo4j，对检索侧透明
- **异步执行**：长任务后台运行，前端轮询进度，支持暂停 / 恢复 / 取消

### 核心设计原则

| 原则 | 说明 |
|------|------|
| **幂等优先** | 每个阶段都可以安全重跑，相同数据不重复写入 |
| **断点续跑** | 任务暂停或失败后，已完成阶段不重跑，从断点继续 |
| **质量门控** | 每条经验必须通过结构校验 + Approach 归一化，才能写入图谱 |
| **标签一致性** | 全局 Tag Catalog 保证不同任务的 Factor / Topic 标签命名统一 |
| **逐条容错** | 单条经验失败不影响整批，记录错误原因后继续处理 |

---

## 2. 整体架构

### 2.1 系统组件

```
                    ┌──────────────────┐
   CSV / JSON  ───► │  FastAPI Server  │ 8000 端口
                    └────────┬─────────┘
                             │ 创建任务，启动后台线程
                             ↓
                    ┌──────────────────┐
                    │  Pipeline        │
                    │  Orchestrator    │ 四阶段编排
                    └────────┬─────────┘
                             │
          ┌──────────────────┼───────────────────┐
          ▼                  ▼                   ▼
   ┌─────────────┐   ┌──────────────┐   ┌──────────────┐
   │  LLM 网关   │   │    MySQL     │   │    Neo4j     │
   │ (OpenAI 兼  │   │  中间结果     │   │  知识图谱      │
   │  容接口)    │    │  + 审计日志   │   │  (最终输出)   │
   └─────────────┘   └──────────────┘   └──────────────┘
```

### 2.2 四阶段流水线

```
Stage 1: FETCH
  拉取对话 → 格式化 → SHA256 去重 → 存 MySQL

Stage 2: KEY MOMENTS
  LLM 判断经验密度（high/medium/low）→ 阈值过滤

Stage 3: DISTILL
  注入全局 Tag Catalog → LLM 提取结构化经验 → 标签校验

Stage 4: INGEST
  经验校验 → Approach 归一化 → 语义去重 → 写入 Neo4j
```

详见 [01-pipeline.md](01-pipeline.md)

### 2.3 技术栈

| 层 | 技术 | 用途 |
|----|------|------|
| Web 框架 | FastAPI + asyncio | HTTP 接口 + lifespan 钩子 |
| 任务执行 | Python threading | 后台长任务，不阻塞主线程 |
| 持久化（中间） | MySQL / PyMySQL | 任务状态、中间结果、审计日志 |
| 持久化（最终） | Neo4j / neo4j-driver | 知识图谱存储与查询 |
| LLM 调用 | OpenAI-compatible API | 三个阶段各用独立配置 |
| 可视化 | PyVis + vis-network | 图谱交互式 HTML 渲染 |
| 前端 | 单页 HTML+JS（无构建） | 任务管理 UI |

---

## 3. 数据流全景

### 输入

```yaml
# JSON 接口（/distill）
{
  "samples": [
    {
      "user_id": 123,
      "counselor_id": 456,
      "start_date": "2025-01-01",   # 可选
      "end_date": "2025-01-31"      # 可选
    }
  ],
  "key_moments_threshold": "medium"  # high | medium | low
}

# CSV 接口（/upload）
user_id,counselor_id,start_date,end_date
123,456,2025-01-01,2025-01-31
```

### 输出（Neo4j 新增内容）

```
新节点:
  IntentStage(S0~S4)     — 意向阶段（已存在则 MERGE）
  Factor(name, polarity) — 正/负向因素
  Approach(id, name)     — 归一化后的销售打法
  Experience(id, ...)    — 完整经验记录
  Topic(name)            — 对话话题

新关系:
  TRANSITION   IntentStage → IntentStage   阶段转移路径
  BLOCKS       Factor → IntentStage        负向因素阻塞
  DRIVES       Factor → IntentStage        正向因素驱动
  RESOLVES     Approach → Factor           打法化解阻塞
  LEVERAGES    Approach → Factor           打法利用驱动
  EVIDENCE_OF  Experience → Approach       经验支撑打法
  INVOLVES     Experience → Topic          经验涉及话题
```

---

## 4. 非功能特性

### 4.1 并发控制

- 每个任务独占一个后台线程
- LLM 调用支持多线程并发（`threads` 配置项）
- 数据 API 调用受 QPS 限流（令牌桶，配置项 `qps`）
- TaskStore 每次操作独立建 MySQL 连接，线程安全

### 4.2 可观测性

| 层级 | 机制 |
|------|------|
| 任务状态 | MySQL `distill_tasks.status` + `current_stage` |
| 阶段进度 | `progress_detail` JSON 字段（stage / processed / total / message） |
| 经验级别 | `distill_ingestions.status`（ok / failed / skipped）+ `error_message` |
| 服务日志 | 结构化日志，携带 `task_id` 上下文 |
| 图谱可视化 | `/tasks/{id}/graph/visual` 返回 PyVis HTML |

### 4.3 错误恢复

```
LLM 调用失败       → 该条记录写 _call_error，跳过，其余继续
经验结构校验失败    → status=failed，记录原因，继续下一条
Approach 归一化失败 → 降级为 new，不阻断主流程
Neo4j 写入失败     → status=failed，已写部分不回滚（幂等 MERGE 设计保证可重入）
```

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-pipeline.md](01-pipeline.md) | 四阶段蒸馏流水线详解（Fetch / Key Moments / Distill / Ingest） |
| [02-experience-model.md](02-experience-model.md) | 经验数据模型（五种类型 / 字段结构 / 验证规则） |
| [03-graph-schema.md](03-graph-schema.md) | 知识图谱结构（Neo4j 节点 / 边 / 查询模式） |
| [04-tag-normalization.md](04-tag-normalization.md) | 标签目录与打法归一化（Tag Catalog / Approach 归一化） |
| [05-task-management.md](05-task-management.md) | 任务管理与数据库设计（MySQL Schema / 断点续跑 / 幂等） |
| [06-api-deployment.md](06-api-deployment.md) | API 接口与部署（端点 / config.ini / 前端 UI） |
