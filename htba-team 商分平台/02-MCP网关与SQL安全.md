---
source: /Users/xgw/workspace/Python/htba-team @90bb93d
type: repo
distilled: 2026-07-27
summary: MCP 网关的 19 个工具、双 transport 共享 registry、SQL guard 八层 AST 防御、黑名单式表列 ACL、限流/脱敏/审计、失败自动起草 infra_bug，以及对外数据应用 /api/v1 通道。
tags: [distill, MCP, SQL安全, 权限治理, 网关]
---

# MCP 网关与 SQL 安全

> 子笔记，回链 [[00-总览]]。业务库访问的**唯一出口**，横切治理集中在此。

## 双 transport 共享同一 registry

- `GET /healthz` → `{"status":"ok"}`（`gateway/server.py:104-106`）
- `POST /mcp/v1` → MCP Streamable HTTP（JSON-RPC 2.0），转 `dispatch_jsonrpc`（`server.py:109-113`）
- `POST /mcp/v1/{tool}` → REST transport，按工具名查注册表调 handler（`server.py:116-188`）
- 两条 transport 都 `from mcp_handlers import TOOL_REGISTRY`，schema/描述不漂移。
- **鉴权+限流前置守卫** `_auth_dep`：`extract_bearer → authenticate → check_rate`（限流在鉴权成功后）（`server.py:86-101`）；User-Agent 命中 `htba-team-doctor/` / `htba-team-ask-probe/` 前缀时旁路限流（token 鉴权仍生效）。
- 错误分层：校验错误降 400、`HtbaError` 按 `http_status`、未预期异常兜 `InternalError()` 500。

**工具清单 SoT**：`_DEFS` 元组（`mcp_handlers.py:874-994`），`TOOL_REGISTRY = {td.name: td for td in _DEFS}`；`registry.py` 的 `ToolDef.input_schema()` 用 Pydantic `model_json_schema()` 自动反射。

## 19 个 MCP 工具

| 分类 | 工具 | 用途 |
|---|---|---|
| 取数 | `run_query` | 自由文本只读 SQL，经 AST 校验 + ACL + 自动 LIMIT |
| 取数 | `run_template_sql` | 角色包登记的 sql_template + 参数渲染执行 |
| 元数据 | `list_tables` | 列 ACL 内表元数据（不含字段明细） |
| 元数据 | `describe_table` | 指定表字段元数据（列已按 ACL 过滤） |
| 元数据 | `check_role_reachability` | 对 ACL 内所有表跑 `LIMIT 0` 探针，返回可达/不可达 |
| 角色包 | `list_role_packs` | 列当前 token 授权的角色包及版本 |
| 角色包 | `get_role_pack` | 拉完整 manifest；local_version 命中返 not_modified |
| 全局 Agent | `get_system_agent` | 拉 active 全局 system agent markdown（按 checksum 增量） |
| 反馈 | `submit_feedback` | 上报候选变更，batch_id 幂等 |
| 提案 | `list_pending_change_proposals` / `get_change_proposal` / `submit_change_proposal_revision` | 扫待处理提案 / 拉单条 / 推新合并稿（不自动 apply） |
| 素材 | `list_feedback_materials` / `get_feedback_material` / `propose_from_feedback_material` / `archive_feedback_material` | 扫素材池 / 拉单条 / 生成提案 / 归档 |
| 偏好 | `get_preference` / `put_preference` / `set_preference_sync` | 拉 / 乐观锁写（冲突返 PREF_CONFLICT）/ 切同步开关 |

（与系统内 `mcp__plugin_htba-team_htba-team__*` 列表一一对应。）

## run_guarded_select 执行链路（单一管道）

`query_service.run_guarded_select`（`query_service.py:40-146`），被 MCP / 对外 API / 后台试跑三条链路复用：

1. 读 `business_duckdb` 配置（`default_limit` / `row_limit`）
2. `validate_sql(raw_sql, acl, default_limit, max_rows)` 安全校验（见下）
3. `execute_select` 落业务库执行
4. **脱敏**：查 `TablesMeta`，列 `masked=true` 或表 `sensitivity=="sensitive"` 的列收为敏感列，按列下标 `masker.mask_row` 脱敏
5. **审计**：成功/失败都 `write_audit`
6. 返回含 `columns/rows/row_count/elapsed_ms` + LIMIT 透明度字段 `auto_limit_applied/auto_limit_truncated/effective_limit`
7. 失败且 `draft_infra_bug=True` 时用**独立 session** 调 `infra_bug_drafter.maybe_draft` 起草，再重新抛出异常

**DuckDB 只读打开**（`business_db.py`）：`duckdb.connect(cfg.database, read_only=True)`；每次查询独立开/关连接（DuckDB 每线程一连接）；同步驱动经 `asyncio.to_thread` + `wait_for(timeout)`，超时映射 `QueryTimeout`；DuckDB 异常翻译为 `BackendTableMissing / BackendColumnMissing / BackendSqlError / BusinessDbUnavailable`。

## SQL guard 八层 AST 防御

`sql_guard.validate_sql`（`sql_guard.py:264-369`），docstring 明列顺序——**顺序即安全边界**：

1. **格式/多语句**：**先于 parse** 查 `";" in sql`（纵深第一闸），再 `sqlglot.parse`；解析失败 `SqlParseFailed`，多语句 `ForbiddenStmt`
2. **语句类型黑名单**：根类型命中 `FORBIDDEN_ROOT_TYPES` 抛 `ForbiddenStmt`——DML/DDL（Insert/Update/Delete/Drop/Alter/Create/Merge…）、DuckDB 副作用（Attach/Detach/Copy/Install/Set/Pragma/Use…）、元数据（Show/Describe）、兜底节点 `exp.Command`（LOAD/VACUUM/EXPLAIN）。**黑名单根类型而非白名单形状**
3. **危险函数黑名单** `FORBIDDEN_FUNCS`：`READ_CSV/READ_PARQUET/PARQUET_SCAN/READ_JSON/READ_BLOB/READ_XLSX/GLOB/QUERY/SLEEP/BENCHMARK/LOAD_FILE` 等文件/外部资源/耗时函数 → `ForbiddenFunc`
4. **系统库拦截** `SYSTEM_SCHEMAS`（information_schema/pg_catalog/system/temp）→ `ForbiddenStmt`；**排在表 ACL 之前**，防 information_schema 绕过 ACL 泄露表结构
5. **表 ACL**：`role_acl.table_allowed` 不过 → `ForbiddenTable`
6. **列 ACL**：黑名单语义，仅拦 `!col` 禁列 → `ForbiddenColumn`（输出别名/多表 JOIN 无法归属/CTE 别名列跳过）
7. **行过滤注入**：仅作用单层 SELECT；`SetOperation`（UNION/INTERSECT/EXCEPT）**直接拒绝**而非静默旁路；命中 `row_filters` 的表把过滤 SQL parse 成条件 `outer_select.where(cond)` 注入
8. **自动 LIMIT** `_enforce_limit`：只改最外层 SELECT——无 LIMIT 注入 default（默认 1000，`auto_applied`）；有 LIMIT 但 > max_rows（默认 10000）向下截断（`auto_truncated`）；**LIMIT 非字面量时保守降为 default**；子查询/CTE 内部 LIMIT 不干预

表引用提取 `_collect_table_refs` 用 CTE 别名集排除 CTE 误判；方言默认 `duckdb`。

## 表/列 ACL（黑名单语义）

- `acl.py:26-40` `_rows_to_acl`：授权某表默认可查**全部列**；`allowed_columns` 为空/含普通列名/`"*"` 都不构成限制。**要禁列须写 `!列名`**（如 `["!id_card","!phone"]`）。
- 行过滤：`row_filter_sql` 非空写入 `filters[table_name]` → `RoleACL.row_filters`。
- 角色 ACL 从 `table_acl` 按 `role_pack_id` 加载；数据应用 ACL 从 `data_app_table_acl` 按 `app_id` 加载，语义完全一致，role_code 用 `app:<code>` 前缀。
- 黑名单解析集中在 `sql_guard.parse_denied_columns`（仅 `!col` 视为禁列）。

## 限流

`rate_limit.py`：类名叫 `TokenBucket`，但**实现是双粒度滑动窗口计数**（秒级 + 分钟级，两个 `deque[float]` 存时间戳），非真正令牌桶（命名与实现不符，以实现为准）。

- MCP 通道按 `token_hash` 限流，限额取 `config.rate_limit.per_second/per_minute`（README 记默认 5/s、50/min 或 100/min，见 §待确认）
- 对外通道独立桶按 `app:<app_id>`（**应用级共享**，多 key 共享一桶，保护后端 DuckDB），限额优先用应用覆盖值否则回退 `[external_api]` 默认
- 单进程内存实现，多副本需换跨进程计数

## 脱敏 / 审计 / infra_bug 自动起草

- **脱敏** `masking.py`：5 条正则（phone `1\d{10}`→`***`、email→`***@***`、order `(?:ORD|TXN)\d{6,}`→`***`、idcard→`***`、amount ≥1000 且有币种上下文→`<amt>`）+ 列级整列 `***`；敏感列判定在 query_service（`masked=true` 或表 `sensitivity=="sensitive"`）
- **审计** `audit.py` `write_audit` → `AuditLog`：`token_hash / role_pack_id / tool / payload_hash（SQL 只存 sha256）/ tables / ok / err_code / latency_ms / rows_returned / details / data_app_id`；成功与失败都写；`commit=False` 时与业务写入合到同一事务
- **infra_bug 自动起草** `infra_bug_drafter.py`：仅覆盖 `BACKEND_TABLE_MISSING` / `BACKEND_COLUMN_MISSING`；**客户端 SQL 误用过滤**（`%`通配/≥4 位纯数字/含 CJK/非 identifier 的 failing_column 判为「把字符串字面值当列名」跳过）；**进程内 LRU 去重** 600s 防 retry 风暴；用**独立 session**、异常被吞不影响主错误响应

## 对外数据应用通道 `/api/v1`

非 AI 客户端（BI/ETL）走此通道只读查数，与角色包解耦；仅当 `config.external_api.enabled` 才挂载路由。

- **鉴权** `external_api/auth.py`：明文 key `dak_<43 base64url>`，库存 sha256；`Authorization: Bearer dak_...` 或 `X-API-Key` 头等价。校验链：key 存在 → key active → 未过期 → app active。key 层失败统一 401 `ApiKeyInvalid`（不区分不存在/吊销/过期，防探测）；app 停用 403 `DataAppDisabled`。`last_used_at` 60s 节流更新。
- **6 个路由**（prefix `/api/v1`）：`GET /me`（自检）、`GET /tables`、`GET /tables/{name}`、`GET /templates`、`POST /templates/{code}/run`、`POST /query`（需 `allow_free_sql`，否则 `FreeSqlDisabled`）。
- 复用同一 `query_service` 执行管道（只读 + AST + 自动 LIMIT + 脱敏 + 审计），Actor 用 `token_hash=key_hash / data_app_id / role_code="app:<code>"`。
- 参数渲染是**字符串占位**：`:key` 替换为字面值，字符串自动加单引号并转义 `'`→`''`；**只能传值不能传结构**（列名/表名/排序不可注入）。

## 来源与可追溯

- 源根：`/Users/xgw/workspace/Python/htba-team` @ `90bb93d`，目录 `src/htba_team/gateway/` + `src/htba_team/common/`
- 双 transport / 守卫 → `gateway/server.py:59-68,86-101,104-188`；`gateway/mcp_jsonrpc.py`
- 工具 SoT → `gateway/mcp_handlers.py:874-997`
- 执行管道 → `gateway/query_service.py:40-146`；`gateway/business_db.py:38-109`
- SQL guard → `common/sql_guard.py:25-91,220-369`
- ACL → `gateway/acl.py:26-88`；`common/sql_guard.py:94-133`
- 限流 → `gateway/rate_limit.py:12-103`
- 脱敏 → `common/masking.py:7-60`；审计 → `gateway/audit.py:12-55`
- infra_bug → `gateway/infra_bug_drafter.py:33-235`
- 对外通道 → `gateway/external_api/auth.py`、`routes.py`；`README.md`「对外只读查数服务」

## 待确认与边界

- MCP 限流默认值 README 一处写「5/s 与 50/min」、§17 另处提「50/min」、平台说明提「100/min」；精确默认以 `common/config.py` + `deploy/config/*.ini` 为准。
- `rate_limit.py` 类名 `TokenBucket` 与滑动窗口实现不符，属命名遗留。
