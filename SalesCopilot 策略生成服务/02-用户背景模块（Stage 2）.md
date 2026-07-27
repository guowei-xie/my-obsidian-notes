---
source: /Users/xgw/workspace/Python/SalesCopilot-Strategy @4aaa067
type: repo
distilled: 2026-07-27
summary: Stage 2 从 PRD 的空占位演进成完整用户背景模块——当前单 + lag 上一单、问卷、通话对话补全、过往订单 LLM 摘要，mysql/hive 双后端，逐层容错。
tags: [distill, user-profile, data-pipeline, LLM, mysql, hive]
---

# 用户背景模块（Stage 2）

> 子笔记 · 回链 [[00-总览]]。姊妹篇：[[01-五阶段在线管线与容错]] · [[03-图谱检索·数据契约·Prompt]] · [[04-方法论·可迁移模式]]
> 以代码 `@4aaa067` 为准。**本模块是 PRD 中"v2 才做"的部分，代码已实现**（PRD §10 说 v1 固定返回 `{}`）。

## 它解决什么

给 Stage 3/5 的 prompt 提供 `{user_profile}`——一个用户的**订单 + 问卷 + 历史对话**背景。价值在 PRD §10 说得很清楚：**重复体验用户的策略不该每次从 S1 破冰，而应从上次断点续接**（"上次意向到 S3、流失原因=时间冲突" → 这次直接承接卖点 + 优先化解时间冲突）。

产出结构（slim 后注入 prompt）：`{"orders": [当前单, 过往单?]}`，每单含 `order_type`、`term_name`、`sku`、`questionnaire_json`、`call_records_text_json`（仅对话正文），过往单额外带 `summary`（LLM 话题/因素摘要）。

## 取数逻辑：当前单 + lag 上一单

入口 `user_profile/fetch.py::fetch_user_profile` → `user_profile/user_profile.py::fetch_user_order_history`：

1. **当前单**：按 `user_id + counselor_id` 在订单宽表里取 `pay_time` 最新一行（`_fetch_current_order_row`）。无本单 → 返回 `[]` → `user_profile` 最终为 `{}`。
2. **过往单**：仅当当前行的 `lag_order_no` 非空时，用 `user_id + lag_order_no` 再查一行（`_fetch_order_row_by_order_no`）。查不到就不加第二单（**无回退逻辑**，明确不猜）。
3. 返回顺序 `[当前, 过往?]`，每条打 `order_type` = `当前订单` / `过往订单`。

去重防呆：若 `lag_order_no` 归一后与当前单 `order_no` 相同，跳过重复过往单。

**`_normalize_order_no` 是个值得一看的防御性函数**：order_no 可能以 int/Decimal/float/str 各种形态从库里出来，直接 `str(float)` 会产生科学计数法或大整数丢精度，导致上一单匹配不到。它逐类型规范成"可与库表比较的整数字符串"，非整型 float 直接弃用并告警。这类"看似简单实则全是坑"的 ID 归一，是数据管线里典型的隐性 bug 源。

## 对话补全：把 call_records_json 换成真对话

订单宽表里的 `call_records_json` 只是通话元数据（含 `counselor_id`）。`_enrich_call_records_with_dialogue` 解析它，对每个 `counselor_id` 调 `HetaoClient.fetch_dialogue`（`until=现在`，不按 pay_time 截断）拉真实对话，写回 `call_records_text_json`。

- **按 counselor_id 去重请求**：`dialogue_cache` 共享，同一顾问只拉一次对话（批量时省大量重复请求）。
- **逐层容错**：hetao 初始化失败 → 每条标 `fetch_error` 但结构完整；单次 fetch 失败 → 该 cid 缓存为 `("", 0, err)`，不中断其余。

## 过往订单 LLM 摘要（第三处 LLM）

`_past_order_dialogue_summary`：过往单额外拉**全量历史对话**（`until=现在`），若同时提供了 `catalog` + `history_summary_llm`，就调 `run_history_dialogue_summary`（`llm/history_dialogue_summary.py`）产出结构化摘要：

```json
{"topics": [{"topic": "...", "positive_factors": [...], "negative_factors": [...]}], "overview": "1-3句概述"}
```

- **同一份受控词表约束**：topic/因素必须逐字来自 Neo4j catalog 目录，程序端 `_normalize_summary` 再过滤词表外标签——与 Stage 3 意向分析、蒸馏服务同一套纪律（见 [[04-方法论·可迁移模式]]）。用轻量模型 `[llm.history_summary]`（qwen-plus），对话超 12000 字符尾部保留。
- **降级链**：缺 counselor_id → 不摘要；对话为空 → `{"topics":[], "overview":"无可用对话记录"}`；缺 catalog/llm → 只记"对话存在但未摘要"；LLM 异常 → None。**任何一步失败都不影响主流程**——摘要是锦上添花。

这处 LLM 是 PRD 完全没有的第三处调用，把"上次聊过什么、家长什么态度"压缩成结构化摘要喂给当前策略，正是"从断点续接"落地的关键。

## 双后端：mysql / hive

`resolve_results_data_backend`（`utils/app_runtime.py`）决定读哪：`--database` CLI 参数写进程内全局，`mysql` 走 PyMySQL 读 `[mysql] mysql_order_table`，`hive` 走 PyHive 读 `[hive] hive_order_table`，默认表名 `mid_sales_copilot_current_info_hdf`。表名用正则 `^[A-Za-z0-9_]+$` 白名单校验防注入。

Hive 特殊处理：`SELECT *` 常返回 `表名.列名`，`_strip_hive_table_prefix` 去前缀对齐业务字段名。

## slim：只留 prompt 需要的

`user_profile/slim.py::slim_one_order` 把订单行压成注入 prompt 的最小结构——`order_type`/`term_name`/`sku` 在前，问卷尽量 JSON 解析（失败保留原串），`call_records_text_json` 只抽 `counselor_id + dialogue_text`（丢掉通话元数据噪声），空值字段不带。`json_safe` 递归把 datetime/Decimal/bytes 转成可 `json.dumps` 的类型。**目的**：喂 LLM 的背景要精简、干净、无冗余元数据——省 token 也降噪。

## 整体容错姿态

`fetch_user_profile` 最外层 `try/except` 兜住一切：无订单 → `{}`，任何异常 → 记 exception 日志 + `{}`。**Stage 2 永不把异常抛给编排器**（对比 Stage 3/4/5 会抛 StageError）——因为背景是增强信息，缺了整个策略仍能生成。这与 [[01-五阶段在线管线与容错]] 的"按依赖性质分层容错"一致。

## 待确认

- 订单宽表 `mid_sales_copilot_current_info_hdf` 的真实列（`lag_order_no`/`call_records_json`/`questionnaire_json`/`term_name`/`sku`/`pay_time`/`order_no`）以库表为准，本次未连库核对，仅从代码读取字段名。
- `questionnaire_json` 的实际 schema 未知——代码只做"尽量解析 JSON，失败保留原串"，不理解其语义。

## 来源与可追溯

- 入口与组装：`src/user_profile/fetch.py`
- 取数 + 对话补全 + 摘要编排：`src/user_profile/user_profile.py`
- slim / json_safe：`src/user_profile/slim.py`
- 历史对话摘要 LLM：`src/llm/history_dialogue_summary.py` + `prompts/history_dialogue_summary_v1/`
- 后端选择：`src/utils/app_runtime.py`；Hive 连接：`src/storage/hive_connection.py`
