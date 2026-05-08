# API 接口与部署

---

## 1. API 端点清单

所有端点基于 FastAPI，服务地址由 `config.ini [server]` 配置（默认 `http://0.0.0.0:8000`）。

| 方法 | 路径 | 功能 | 返回状态码 |
|------|------|------|-----------|
| `POST` | `/distill` | 从 JSON 创建蒸馏任务 | 202 |
| `POST` | `/upload` | 从 CSV 文件创建蒸馏任务 | 202 |
| `GET` | `/tasks/{task_id}` | 查询任务状态与进度 | 200 / 404 |
| `GET` | `/tasks` | 列出最近任务（倒序，最多 200 条） | 200 |
| `POST` | `/tasks/{task_id}/pause` | 暂停运行中的任务 | 200 / 409 |
| `POST` | `/tasks/{task_id}/resume` | 恢复暂停的任务 | 200 / 409 |
| `POST` | `/tasks/{task_id}/cancel` | 取消任务（不可恢复） | 200 / 409 |
| `GET` | `/tasks/{task_id}/graph` | 获取任务图谱 delta（JSON） | 200 / 404 |
| `GET` | `/tasks/{task_id}/graph/visual` | PyVis 图谱可视化页面（HTML） | 200 / 404 |
| `GET` | `/` | 前端 UI 主页 | 200 |

---

## 2. 任务创建

### 2.1 POST /distill — JSON 接口

**请求体：**

```json
{
  "samples": [
    {
      "user_id": 123,
      "counselor_id": 456,
      "start_date": "2025-01-01",
      "end_date": "2025-01-31"
    },
    {
      "user_id": 789,
      "counselor_id": 456
    }
  ],
  "key_moments_threshold": "medium"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `samples` | array | ✓ | 样本列表，每项含 user_id + counselor_id |
| `samples[].user_id` | integer | ✓ | 用户 ID |
| `samples[].counselor_id` | integer | ✓ | 销售顾问 ID |
| `samples[].start_date` | string | — | 格式 YYYY-MM-DD，不填则不限起始日期 |
| `samples[].end_date` | string | — | 格式 YYYY-MM-DD |
| `key_moments_threshold` | enum | — | `high` / `medium`（默认）/ `low` |

**响应体（202）：**

```json
{
  "task_id": "a1b2c3d4-...",
  "status": "pending",
  "created_at": "2025-01-15T10:30:00"
}
```

### 2.2 POST /upload — CSV 接口

**请求格式：** `multipart/form-data`

| 字段 | 类型 | 说明 |
|------|------|------|
| `file` | File | CSV 文件（支持 UTF-8 BOM / GBK 编码） |
| `key_moments_threshold` | Form | `high` / `medium`（默认）/ `low` |

**CSV 格式：**

```csv
user_id,counselor_id,start_date,end_date
123,456,2025-01-01,2025-01-31
789,456,,
```

`start_date` 和 `end_date` 列可以为空，日期格式支持 `YYYY-MM-DD` 或 `YYYY/M/D`。

响应体与 `/distill` 相同（202）。

---

## 3. 任务状态查询

### GET /tasks/{task_id}

**响应字段：** `task_id` / `status` / `created_at` / `finished_at` / `current_stage` / `progress`（含 stage, processed, total, message）/ `error_message` / `summary`（完成后附带，字段见第 5 节）

### GET /tasks

查询参数：`limit`（默认 20，最大 200）

返回按创建时间倒序的任务列表，`done` 状态的任务附带 `summary`。

---

## 4. 任务控制

### POST /tasks/{task_id}/pause

暂停 `running` 或 `pending` 状态的任务。任务在下一个控制检查点（处理下一条记录前）停止。

| 响应码 | 含义 |
|--------|------|
| 200 | 暂停成功，返回更新后的 TaskResponse |
| 404 | task_id 不存在 |
| 409 | 任务不在可暂停状态（已完成/已取消） |

### POST /tasks/{task_id}/resume

恢复 `paused` 状态的任务，原子地将状态切换为 `running` 并启动新后台线程。已处理的记录自动跳过，从断点继续。

| 响应码 | 含义 |
|--------|------|
| 200 | 恢复成功 |
| 409 | 任务不在 paused 状态（或已被其他请求恢复） |

### POST /tasks/{task_id}/cancel

不可逆取消。中间结果保留在 MySQL 中，返回当前已产出的 `summary`。

---

## 5. 图谱可视化

### GET /tasks/{task_id}/graph

返回本次任务新增的图谱 delta（JSON）：

```json
{
  "nodes": [
    {"node_key": "AP-007", "label": "Approach", "display_name": "先强化价值再处理价格异议"},
    {"node_key": "EXP-123_456-0", "label": "Experience", "display_name": "..."}
  ],
  "edges": [
    {"edge_type": "EVIDENCE_OF", "src_key": "EXP-123_456-0", "dst_key": "AP-007"}
  ]
}
```

查询参数：`limit`（默认 25，最大 500）——限制返回节点数量，防止大任务数据量过载。

### GET /tasks/{task_id}/graph/visual

返回 PyVis + vis-network 渲染的交互式 HTML 页面，前端主页通过 iframe 嵌入。

---

## 6. 配置文件（config.ini）

```ini
[hetao_data]
api_key = your_api_key
api_url = https://your-data-api.example.com
qps = 5                    ; 每秒最多 5 个数据 API 请求

[llm.default]
base_url = https://your-llm-gateway.example.com
model = deepseek-r1
api_key = your_llm_key
threads = 5                ; 并发 LLM 调用线程数

; 以下三个区块可选，不填时继承 [llm.default]
[llm.key_moments]          ; Stage 2 使用的 LLM 配置
[llm.distill]              ; Stage 3 使用的 LLM 配置
[llm.approach_normalization]  ; Stage 4 Approach 归一化使用的 LLM 配置

[neo4j]
uri = bolt://localhost:7687
user = neo4j
password = your_password
database = neo4j

[mysql]
host = localhost
port = 3306
user = root
password = your_password
database = distill

[server]
host = 0.0.0.0
port = 8000

[dedup]
max_similar_experiences = 3      ; 每个(打法+路径)最多保留多少条相似经验
factors_overlap_threshold = 0.5  ; Jaccard 相似度阈值
```

---

## 7. 前端 UI

单页 HTML+JS 应用（无需构建），位于 `src/static/index.html`，通过 `GET /` 访问。

**功能：**

| 功能 | 说明 |
|------|------|
| CSV 上传 | 拖拽或点击上传 CSV 文件，选择经验密度阈值 |
| 任务列表 | 实时轮询显示所有任务的状态、阶段进度 |
| 进度条 | 每个阶段独立进度条，显示已处理 / 总数 / 当前消息 |
| 任务控制 | 暂停 / 恢复 / 取消按钮（根据当前状态动态显示） |
| 图谱预览 | 任务完成后嵌入 `/graph/visual` iframe，展示本次新增图谱 |
| 结果统计 | 任务完成后展示对话数、经验数、成功 / 失败 / 跳过数量 |

---

## 8. 部署检查清单

```
□ MySQL 建表（服务启动时自动执行 init_db，幂等）
□ Neo4j 唯一约束初始化（服务启动时通过 neo4j_writer 执行）
□ config.ini 完整填写（参考 config.ini.example）
□ 启动服务：uvicorn src.main:app --host 0.0.0.0 --port 8000
□ 验证：访问 http://localhost:8000 确认前端 UI 可用
□ 验证：POST /distill 提交测试样本，GET /tasks/{id} 确认任务推进
```

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [00-overview.md](00-overview.md) | 系统概述与整体架构 |
| [01-pipeline.md](01-pipeline.md) | 四阶段蒸馏流水线 |
| [02-experience-model.md](02-experience-model.md) | 经验数据模型 |
| [03-graph-schema.md](03-graph-schema.md) | 知识图谱结构 |
| [04-tag-normalization.md](04-tag-normalization.md) | 标签目录与打法归一化 |
| [05-task-management.md](05-task-management.md) | 任务管理与数据库设计（MySQL Schema） |
