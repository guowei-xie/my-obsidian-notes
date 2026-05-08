# 任务管理与数据库设计

---

## 0. 设计原则

> **MySQL 记录什么，就能从哪里恢复。** 每阶段输入输出均持久化，兼顾审计日志与断点恢复。

---

## 1. 任务生命周期

### 1.1 状态机

```
                    ┌─────────┐
             创建   │ pending │
  ─────────────────►│         │
                    └────┬────┘
                         │ 后台线程启动
                         ▼
                    ┌─────────┐
                    │ running │◄──────────────────┐
                    └────┬────┘                   │
               ┌─────────┼──────────┐             │ resume
               │         │          │             │
          暂停 ▼    完成  ▼     失败  ▼      ┌─────┴────┐
         ┌────────┐ ┌──────┐ ┌────────┐    │  paused  │
         │ paused │ │ done │ │ failed │    └──────────┘
         └────┬───┘ └──────┘ └────────┘
              │
         取消  ▼
         ┌───────────┐
         │ cancelled │
         └───────────┘
```

### 1.2 状态说明

| 状态 | 含义 | 可转换至 |
|------|------|---------|
| `pending` | 已创建，等待后台线程启动 | running |
| `running` | 正在执行 | paused / done / failed |
| `paused` | 用户主动暂停，可恢复 | running（resume） / cancelled |
| `done` | 所有阶段完成 | —（终态） |
| `failed` | 发生未捕获异常 | —（终态） |
| `cancelled` | 用户取消，不可恢复 | —（终态） |

**暂停粒度：** 控制检查点位于每个阶段「处理下一条记录之前」。暂停命令写入 DB 后，正在运行的当前条处理完成后才停止，不会在中间状态截断。

---

## 2. MySQL Schema

| 表 | 阶段 | 关键字段 | 幂等键 |
|----|------|---------|-------|
| `distill_tasks` | — | task_id, status, request_params, current_stage, progress_detail | PRIMARY KEY |
| `distill_conversations` | Stage 1 | sample_id, dialogue_text, dialogue_hash, msg_count | idx_dialogue_hash（跨任务去重） |
| `distill_key_moments` | Stage 2 | conversation_id, density, key_moments_json, retained | UNIQUE(task_id, conversation_id) |
| `distill_distillations` | Stage 3 | conversation_id, experience_json, experience_count | UNIQUE(task_id, conversation_id) |
| `distill_ingestions` | Stage 4 | experience_id, approach_id, match_type, status, error_message | UNIQUE(task_id, experience_id) |
| `distill_graph_nodes` | Stage 4 | node_key, label, display_name, properties_json | UNIQUE(task_id, node_key) |
| `distill_graph_edges` | Stage 4 | edge_type, src_key, dst_key, properties_json | UNIQUE(task_id, edge_type, src, dst) |

**关键设计注意：**
- `experience_json` 存完整 LLM 原始输出（Neo4j 只存检索侧属性）；LLM 失败时写入 `{"_call_error": "原因"}`，Stage 4 识别后跳过
- `distill_ingestions.status`：`ok` / `failed`（结构校验或写入失败）/ `skipped`（语义去重命中）
- `distill_graph_nodes/edges` 记录本任务 delta，供 `/tasks/{id}/graph` 可视化，不是 Neo4j 全量镜像

---

## 3. TaskStore 模式

所有 MySQL 操作通过 `TaskStore` 类统一访问，每次操作独立建连（不使用连接池），防止长任务期间连接泄漏或被 `wait_timeout` 关闭。主要方法：

`create_task` / `get_task` / `list_tasks` / `pause_task` / `cancel_task` / `resume_task_atomic`（paused→running 原子切换）/ `get_task_summary` / `get_task_graph`

---

## 4. 幂等设计汇总

| 表 | 幂等机制 | 覆盖场景 |
|----|---------|---------|
| `distill_conversations` | `dialogue_hash` 索引去重 | 相同对话不重复拉取 |
| `distill_key_moments` | UNIQUE(task_id, conversation_id) | 重跑 Stage 2 不重复判断 |
| `distill_distillations` | UNIQUE(task_id, conversation_id) | 重跑 Stage 3 不重复提取 |
| `distill_ingestions` | UNIQUE(task_id, experience_id) | 重跑 Stage 4 不重复写图谱 |
| `distill_graph_nodes` | UNIQUE(task_id, node_key) | 图谱节点缓存去重 |
| `distill_graph_edges` | UNIQUE(task_id, edge_type, src, dst) | 图谱边缓存去重 |
| Neo4j 节点 | MERGE on 唯一约束（approach_id、experience_id 等） | 多次写入不创建重复节点 |

---

## 5. 任务汇总统计

任务完成后（`status=done` 或 `cancelled`），`GET /tasks/{id}` 返回 `summary`：

```json
{
  "conversations_fetched": 45,
  "conversations_retained": 28,
  "experiences_extracted": 87,
  "experiences_ok": 72,
  "experiences_failed": 8,
  "experiences_skipped": 7,
  "approaches_new": 5,
  "approaches_matched": 52,
  "approaches_revised": 15
}
```

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [00-overview.md](00-overview.md) | 系统概述与整体架构 |
| [01-pipeline.md](01-pipeline.md) | 四阶段蒸馏流水线（断点续跑机制） |
| [02-experience-model.md](02-experience-model.md) | 经验数据模型 |
| [03-graph-schema.md](03-graph-schema.md) | 知识图谱结构 |
| [04-tag-normalization.md](04-tag-normalization.md) | 标签目录与打法归一化 |
| [06-api-deployment.md](06-api-deployment.md) | API 接口（暂停 / 恢复 / 取消端点） |
