---
source: /Users/xgw/workspace/Python/SalesCopilot-Strategy @4aaa067
type: repo
distilled: 2026-07-27
summary: 服务与三个外部世界的接口——hetao 对话 API 解析、Neo4j catalog/advance/resolve 检索、以及三处 LLM 的分阶段配置、prompt 契约与防御式 JSON 解析。
tags: [distill, neo4j, cypher, RAG, prompt-engineering, LLM, data-contract]
---

# 图谱检索 · 数据契约 · Prompt

> 子笔记 · 回链 [[00-总览]]。姊妹篇：[[01-五阶段在线管线与容错]] · [[02-用户背景模块（Stage 2）]] · [[04-方法论·可迁移模式]]
> 以代码 `@4aaa067` 为准。本篇讲服务与三个外部世界的接口：hetao 对话、Neo4j 图谱、LLM。

## 一、hetao 对话 API（Stage 1 数据契约）

`hetao/client.py::HetaoClient.fetch_dialogue(user_id, counselor_id, until)` → `(fulltext, msg_count)`。

- **请求**：`POST` 到 `[hetao_data].api_url`，header `Authorization: APPCODE {api_key}`，body `{user_id, counselor_id, data_type:["call","wechat"]}`。`HETAO_API_KEY`/`HETAO_API_URL` 环境变量可覆盖配置。
- **响应解析**（`_extract_messages`）：每条消息按 `(data_type, msg_type)` 分派：
  - `wechat/text` → 直接文本。
  - `wechat/voice` → 文本，或 `[{"content":...}]` JSON 数组取首元素 content。
  - `wechat/meeting_voice_call` 与 `call` → `[{userType, content, startTime}]` 数组，逐条展开。
- **嵌套时间戳**：数组内每条消息时间 = `父消息 msg_time + startTime 毫秒偏移`（`_parse_timestamped_items`）。
- **时区归一**：hetao 的 `msg_time` 是 naive 串（`YYYY-MM-DD HH:MM:SS`），按业务约定视为 `Asia/Shanghai`；`until` 入参也 `astimezone(Asia/Shanghai)`，同区比较后**保留 `timestamp <= until` 的消息**——这是 `moment` 截断的落点。
- **角色映射**：`teacher→老师`，`parent|student→家长`，其它→未知。
- **输出格式**：`[1] 老师: ...\n[2] 家长: ...`（按 timestamp 排序后编号）。空对话返回 `("", 0)`。

> 本客户端注明改造自 `salescopilot-ct-skill/scripts/fetch_dialogue.py`——与蒸馏服务共用同一份 hetao 取数逻辑血统。

## 二、Neo4j 图谱检索

`graph/neo4j_client.py::Neo4jReadClient`——**只读**客户端，driver 由 lifespan 建一次复用，session 每查一次性创建。三类查询：

### catalog 查询（词表）
```cypher
MATCH (f:Factor) WHERE f.name IS NOT NULL RETURN f.name, f.polarity ORDER BY f.polarity, f.name
MATCH (t:Topic)  WHERE t.name IS NOT NULL RETURN t.name ORDER BY name
```
→ `{factors:[{name,polarity}], topics:[name]}`。经 `CatalogCache` 缓存（下节）。

### advance 查询（推进意向经验）
匹配 `(e:Experience)-[:EVIDENCE_OF]->(a:Approach)`，`WHERE e.from_stage = $from_stage AND e.experience_type IN ['intent_advance','pacing','context_leverage']`。若有因素，用 `CASE WHEN any(f IN e.factors_addressed WHERE f CONTAINS '<factor>') ... THEN 0 ELSE 1 END AS factor_priority`，排序 `ORDER BY factor_priority ASC, a.example_count DESC`——**因素重叠的排前，其次按证据量**。无因素则退化为纯 `example_count` 倒序。
- 传入的 `factors` = 正因素 + 负因素（`stage4_graph.py` 里 `advance_factors = positive + negative`）。

### resolve 查询（化解阻塞经验）
`WHERE e.experience_type = 'blocker_resolve' AND (any(f IN e.factors_addressed WHERE f CONTAINS '<neg_factor>') OR ...)`，按 `example_count` 倒序。仅用负因素。无负因素关键词返回 `[]`。

### 两个关键设计
- **CONTAINS 模糊匹配而非等值**：容忍 LLM 偶发产出图谱里没有的 factor 字面量（多/少标点、近义词）。
- **`_sanitize` 防注入**：factor 字面量被拼进 Cypher 字符串，先过正则 `^[一-鿿\w]+$` 白名单（中英文/数字/下划线），非法 token 直接丢弃——因为这些值来自 LLM 输出，不可信。

### 合并与 Top-K（`stage4_graph.py::_merge_and_topk`）
advance + resolve 并行结果合并：按 `approach_id` 去重（**同 ID 时 advance 先加故优先保留**），转 `RetrievedExperience` 模型，按 `example_count` 倒序，截 `top_k`（默认 8）。`example_count` = 该方法被多少条经验佐证 → **证据量即排序权重/置信度**。

## 三、catalog 缓存

`graph/catalog_cache.py::CatalogCache`：TTL 600s，lifespan 启动同步加载一次，过期时同步刷新。`threading.Lock` 防并发重复刷新（拿到锁后再判一次是否过期，双检）。`version` = `{loaded_at}-{Nf}-{Mt}`（因素数/话题数），随响应 `debug_meta.catalog_version` 回传，方便判断某次结果用的是哪版词表。`GET /catalog?refresh=true` 强制刷新。

**为什么缓存**：词表变化慢（图谱不常改），但每次 Stage 3/Stage 2 摘要都要注入——缓存避免每请求打 Neo4j。这与"进程内共享、workers=1"是同一套进程内单例思路。

## 四、三处 LLM：分阶段异构配置

`llm/openai_chat.py`，OpenAI 兼容 Chat Completions 封装。三处调用各有独立配置，缺字段回退 `[llm.default]`：

| 阶段 | section | 默认模型 | 温度 | 诉求 |
|---|---|---|---|---|
| 意向分析 | `[llm.intent]` | qwen-plus | 0.0 | 延迟敏感、要确定性 |
| 历史摘要 | `[llm.history_summary]` | qwen-plus | 0.1 | 轻量、批量 |
| 策略生成 | `[llm.strategy]` | deepseek-r1 | 0.1 | 质量敏感，用重量推理模型 |

`load_llm_config` 的回退细节：阶段 section 里**空字符串视为未设置**，继续 fall through 到 default（commit `277926a` 专门修的）。`base_url` 指向内网网关 `data.corp.hetao101.com/dc-route-api/route/v1/`。client 按 `(api_key, base_url)` 用 `lru_cache` 复用连接池。

### 防御式 JSON 解析
`chat_completion_json_dict` 强制 `response_format={"type":"json_object"}`，再多层兜底解析 `_parse`：
1. 严格 `json.loads`（要求 root 是 dict）；
2. 失败则 `_strip_fences` 去掉 ` ```json ... ``` ` 代码围栏重试；
3. 再失败用 `JSONDecoder().raw_decode` 从第一个 `{` 开始解析，**容忍尾部多余文本**（如推理模型在 JSON 后带解释），尾部非空只告警不报错。
全失败才抛异常，由调用方决定降级。—— 这是喂 LLM 结构化输出的标准防御，尤其 deepseek-r1 这类推理模型容易在 JSON 前后带思考文本。

## 五、Prompt 契约

三处 prompt 都是 `prompts/<name>_v1/{system,user}.txt`，`load_prompt_files` 读取，user 模板用 `str.format` 注入变量。

### intent_analysis_v1（Stage 3）
- **步骤链**：读对话 → 识别话题 → 每话题下提正/负因素附原文证据 → 综合推导 S0-S4。
- **词表纪律**（system 反复强调）：topic/factor 必须逐字用 user 消息提供的清单内字面量，不自造，对不上宁可不提取。
- **输出**：`topics_factors[]`（嵌套 topic→factors）、`positive_factors[]`、`negative_factors[]`、`intention`(S0-S4)、`intention_reason`，以及**改版后**的 `proof` 字段（每个因素的 proof 从单串变成字符串数组，新增顶层 `proof` 数组——commit「改进意向推断提示词」）。
- **user 注入**：`{topics_catalog}` / `{positive_factors_catalog}` / `{negative_factors_catalog}`（catalog 快照，逗号分隔）+ `{user_profile}` + `{dialogue_text}`。

程序端 `stage3_intent.py` 收口：`_normalize_topics_factors` 把嵌套结构**展平**成 `TopicFactor(topic, factor, polarity)` 并过滤词表外标签；`_extract_factor_tags` 从 `[{tag,proof}]` 抽 tag 去重过滤；非法 `intention` 回退 S2。**注意 response 模型 `TopicFactor` 是扁平三字段**，与 PRD §3.1 展示的嵌套 JSON 不同——扁平化在解析层完成。

### strategy_generation_v1（Stage 5）
- **策略原则**（system）：先化解阻塞再利用驱动、具体到"下一步做什么"、参考经验但不照搬、动作 2-4 个按优先级。
- **反复申明"顾问不是话术机"**：给思路方向不给逐字脚本；每条动作要有经验依据（可写 approach_id）；经验为空时 confidence=low 给 1-2 条通用建议；语言给一线销售看要直白。
- **输出**：`strategy_card`（含 `markdown` 字段——LLM 同步产出可直接前端渲染的整卡 markdown）+ `evidence_references[]`。
- **user 注入**：`{user_profile}` + 意向分析全量 + `{dialogue_excerpt}`（尾部 6000 字符）+ `{reference_experiences_block}`。

`reference_experiences_block` 由 Python 端 `render_reference_experiences_block` 渲染——把 Top-K 经验拼成 markdown（标题带 `[APR-xxxx] 名称（已验证 N 次）`、来源类型、适用阶段、化解/利用因素、方法描述、执行要点、为什么有效、三段式话术证据 before/action/after）。经验为空时注入一句"暂无参考经验，请给保守建议 confidence=low"。**RAG 检索结果的"上下文渲染"在程序端做，LLM 只消费**。

### history_dialogue_summary_v1（Stage 2 过往单）
见 [[02-用户背景模块（Stage 2）]]。同样是"词表约束 + 程序端过滤"的结构化摘要。

## 来源与可追溯

- hetao 客户端：`src/hetao/client.py`
- Neo4j 只读客户端 + Cypher：`src/graph/neo4j_client.py`
- catalog 缓存：`src/graph/catalog_cache.py`
- Stage 4 合并/Top-K：`src/pipeline/stage4_graph.py`
- LLM 封装 + JSON 防御：`src/llm/openai_chat.py`
- Stage 3 解析收口：`src/pipeline/stage3_intent.py`
- Stage 5 渲染/兜底：`src/pipeline/stage5_strategy.py`
- Prompt：`prompts/{intent_analysis_v1,strategy_generation_v1,history_dialogue_summary_v1}/`
- 响应模型（TopicFactor 扁平等）：`src/models/response.py`
