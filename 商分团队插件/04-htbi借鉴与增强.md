# 04 - HTBI 借鉴与增强清单

> 阅读了同机器上的 `openclaw-htbi-suite` 后，识别出 9 项与本项目架构兼容且确有增益的设计，整理为本文件作为对 [[01-架构与权限]] / [[02-内容与迭代]] 的增强提案。
> 两个项目**架构完全不同**（htbi 是 OpenClaw 插件 + 飞书交付 + 单租户），不做模仿，只取精华。

---

## 一、采纳清单（按优先级）

| # | 借鉴点 | htbi 原型 | 在本项目中的落位 | 优先级 |
|---|--------|----------|------------------|--------|
| 1 | `references/{tables,metrics,methods}.md` 三件套 | ops-analyst 的引用资料 | 角色包内容资产的**标准目录结构** | P0 |
| 2 | "分析师 + 策展人"职责分离 | ops-analyst + curator 双 agent | data-collector 升级为**结构化候选变更生成器** | P0 |
| 3 | 候选变更 → diff → 逐条确认 → change-log | curator 沉淀流程 | 反馈池接收的不是模糊文本，而是**结构化候选包** | P0 |
| 4 | 报告交付前校验工具 | `htbi_report_validate` + 模板契约 | MCP 网关新增 `validate_report` 工具 | P1 |
| 5 | PreToolUse 第二层 hook 保护 | `plugin-write-guard` | 本地 PreToolUse hook 保护 `preferences.md` 与缓存目录 | P1 |
| 6 | SKILL 模板：你负责 / 你不负责 / 强制工作流 / 红线 | ops-analyst SKILL.md | 角色包内所有 skill 的**统一写作模板** | P1 |
| 7 | 用户侧 change-log（角色包变更日志） | curator 追加式 change-log | SessionStart 拉到新版时摘要变更项给用户 | P2 |
| 8 | onboarding-wizard 作为可被命中的 SKILL | htbi-onboarding-wizard | 首次安装/换 token/排错引导，由 Claude 命中 | P2 |
| 9 | 配置 schema（JSON Schema 校验） | plugin.json `configSchema` | plugin.json 加 `configSchema` 校验 gateway URL/token 路径 | P2 |

下文逐条展开。

---

## 二、详细设计

### 1. references 三件套（P0）

**htbi 做法**：ops-analyst skill 强制先读 `references/tables.md` → `metrics.md` → `methods.md`，查不到不取数，转交 curator。

**本项目落位**：角色包内容资产固化为三个 markdown 文件，由后台 `metric_defs / tables_meta / methods`（新增）拼装生成，下发到会话级 `.claude/skills/<role>/references/`。

```
<project>/.claude/skills/<role>/
├── SKILL.md
└── references/
    ├── tables.md      # 来自 tables_meta + role_pack.table_acl，含字段中文名、口径、敏感度
    ├── metrics.md     # 来自 metric_defs，含口径定义、单位、版本号、可下钻维度
    └── methods.md     # 来自 method_defs（新增表），含时间窗、显著性、异常阈值
```

**SKILL.md 强制工作流**新增首条：
> 任何取数前，必须先读 `references/metrics.md` 找口径 → 找不到时**停手**，提示用户走反馈流程

**新增 DDL（追加到 01 §5）**：
```sql
CREATE TABLE method_defs (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  code VARCHAR(64), name VARCHAR(128),
  description TEXT NOT NULL,        -- 时间窗 / 显著性 / 阈值 / 归因方法
  version VARCHAR(32),
  status ENUM('draft','active','archived'),
  UNIQUE KEY (code, version)
);
```
`role_assets.asset_type` 枚举追加 `'method'`。

---

### 2 + 3. data-collector 升级为「候选变更生成器」（P0）

**htbi 做法**：curator agent 收到用户反馈 → 提炼候选变更（分类 + 来源 + 影响文件 + diff + 风险）→ 用户逐条确认 → 追加 change-log。所有结构化。

**本项目落位**：原 data-collector 只输出 Q/E/B 三类自由文本反馈。升级后输出**结构化候选变更包**，反馈池直接收到 diff。

升级前：
```yaml
type: Q
content: "用户问按地域看留存，但没找到模板"
```

升级后（与 htbi curator 候选变更对齐）：
```yaml
batch_id: 2026-05-12T14:33:00+08:00
source_session: <transcript_hash>
candidate_changes:
  - id: C1
    category: sql_template            # metric | sql_template | method | table_doc | skill | preference
    target_role_pack: growth_analyst
    target_file: skills/channel-quality/sql-templates/retention-by-region.sql
    summary: "新增按地域分组的 7 日留存模板"
    diff_or_new: |
      + WITH base AS (
      +   SELECT user_id, region, DATE(register_time) AS d
      +   FROM dwd_user_register WHERE dt BETWEEN :start AND :end
      + ) ...
    risk: "region 字段非分区列，大表全表扫描需评估"
    user_confirmed: false             # 用户在 AskUserQuestion 中勾选后置 true
```

**用户交互**（替代 02 §2.3 的范式）：
```
data-collector：
我整理了 2 条候选变更，是否上报到反馈池？逐条确认：

[C1] 新增模板「按地域的 7 日留存」
     文件：skills/channel-quality/sql-templates/retention-by-region.sql
     风险：region 非分区字段
     [ ] 上报

[C2] 修正指标「新用户」口径
     文件：references/metrics.md
     diff:  - 注册当天首单完成
            + 注册当天且首单完成且未退款
     [ ] 上报
```

**后台反馈池的演进**：每条 feedback 直接带 diff，运营点"采纳"→ 自动生成内容仓库 PR 草案，**审核环节就是 review diff**，效率几倍提升。

**`feedback.content_json` 格式**统一为上述 `candidate_changes[]` 结构。

---

### 4. 报告校验工具 `validate_report`（P1）

**htbi 做法**：`htbi_report_validate` 工具读 `references/metrics.md` 与 `tables.md` 构建 catalog，与报告包做交叉校验（章节齐全？必填字段？指标/表/字段都登记过？），输出 `passed / passed_with_caveats / failed`。SKILL 强制：交付前必须出现 `validation: status: passed`。

**本项目落位**：MCP 网关新增工具：

```
validate_report(role_code, report_package) → ReportValidationResult
```

报告模板（`report_templates`）新增字段 `contract_json`，定义：
```json
{
  "required_sections": ["背景", "核心指标", "异常归因", "建议"],
  "required_fields": ["period", "channel", "yoy_pct"],
  "required_methods": ["同环比", "结构归因"]
}
```

校验逻辑：把 `report_package.metrics/tables/fields` 与角色包 references 求差集，未登记的进 `unknown_*`；未在 `caveats` 中说明的 unknown 触发 failed。

**SKILL 模板**强制要求：引用任何 `report_template` 的角色，交付前必须调用 `validate_report`，结果 `passed` 或 `passed_with_caveats` 才能输出最终报告，否则补齐 / 写 caveats 后重试。

---

### 5. PreToolUse hook 第二层保护（P1）

**htbi 做法**：`plugin-write-guard` 在 `before_tool_call` 拦截非 curator 对插件目录的 Edit/Write/apply_patch 与写入型 shell 命令。

**本项目落位**：本地 `hooks/pre-tool-use.sh` 拦截以下场景：

| 触发条件 | 动作 |
|---------|------|
| 任何 agent（除 data-collector）尝试 Edit/Write `~/.<插件名>/preferences.md` | block |
| 任何 agent 尝试写 `~/.<插件名>/cache/`、`~/.<插件名>/token`、`~/.<插件名>/active_role` | block |
| 任何 shell 命令在上述路径执行 `>` / `tee` / `rm` / `sed -i` | block |

**为什么需要**：MCP 网关只能管远端，本地文件的"完整性"需要客户端侧第二层保护。尤其 `preferences.md` 是用户私有偏好，绝不允许角色包下发的 agent 静默改写。

只有 **`data-collector`** agent（带固定 agent_id）可以 append-only 写 preferences.md。

---

### 6. SKILL 写作模板（P1）

**htbi 做法**：ops-analyst SKILL.md 顶部就有 "你负责 / 你不负责 / 强制工作流（编号 1-9）/ 安全红线"。

**本项目落位**：角色包所有 skill 必须遵循统一模板（运营在后台编辑 skill 时模板预填）：

```markdown
---
name: <skill-name>
description: <场景与触发条件>
---

# <Skill 名>

## 你负责
- ...

## 你不负责（红线）
- 不查 references/tables.md 未登记的表
- 不输出明文敏感字段（手机号/邮箱/订单号/金额）
- 不把 SQL/host/凭证暴露给业务用户
- 不修改 ~/.<插件名>/preferences.md 之外的本地文件

## 强制工作流
1. 读 references/metrics.md 找口径，找不到则停手
2. 必要时 mysql_describe 验证字段
3. 调 run_template_sql 或 run_query 取数
4. 分析与检验（按 references/methods.md）
5. 交付前若引用 report_template，必须 validate_report
6. 发现新口径/缺模板时，data-collector 会自动生成候选变更

## 常见澄清
- ...

## 输出格式（YAML 结构化）
```yaml
period: { start: ..., end: ... }
findings: [...]
caveats: [...]
```
```

校验：后台保存 skill 时检测必备章节是否齐全（lint），缺则提示。

---

### 7. 用户侧 change-log（P2）

**htbi 做法**：curator 每次改完 references 追加 change-log.md。

**本项目落位**：每次 SessionStart 拉到新版角色包时，client 比对 manifest 的 `change_summary` 字段（后台发布时填写），以系统提示方式告知用户：

```
角色包「用户增长商分」已更新到 v2026.05.12-3
本次变更：
- 修正指标「新用户」口径（v3→v4）
- 新增模板「按地域 7 日留存」
- 修正表 dwd_user_register 的分区字段说明
```

实现：`role_pack_versions` 表新增 `change_summary TEXT` 字段。

---

### 8. onboarding-wizard SKILL（P2）

**htbi 做法**：把"首次安装/配置/绑定/验证"做成可被 Claude 命中的 SKILL，而非孤立 README。

**本项目落位**：插件壳内置一个 `onboarding-wizard` skill（不依赖 role pack，全员可见），description 写：
> 首次使用本插件、token 失效、角色切换异常、网关连不通时的引导

涵盖：
- `/<插件名> token set` 怎么用
- `/<插件名> role list/use/current` 怎么用
- token 过期/吊销错误码识别
- `/<插件名> doctor` 自检报告解读
- 常见错误处理表

---

### 9. configSchema（P2）

**htbi 做法**：plugin.json 内嵌 JSON Schema 校验客户端配置。

**本项目落位**：plugin.json 加：
```json
{
  "configSchema": {
    "type": "object",
    "properties": {
      "gatewayUrl":      { "type": "string", "format": "uri" },
      "tokenFile":       { "type": "string", "default": "~/.<插件名>/token" },
      "preferencesFile": { "type": "string", "default": "~/.<插件名>/preferences.md" },
      "requestTimeoutMs":{ "type": "number", "default": 30000 }
    },
    "required": ["gatewayUrl"]
  }
}
```

避免 SessionStart hook 在错配置下静默失败。

---

## 三、不采纳的部分（与本项目架构冲突或重叠）

| htbi 元素 | 不采纳的原因 |
|----------|-------------|
| 双外壳 agent（ops-monitor / htbi-curator） | 本项目用户直接在 Claude Code 主会话用，无需外壳层 |
| pyproject + bi-stats / bi-charts CLI | 数据处理由 Claude 自身完成，不引入额外 Python 工具链 |
| 飞书交付（htbi-deliver-feishu） | 本项目不在范围 |
| `passwordSecretRef` 凭证引用 | 本项目客户端不持 DB 凭证，凭证只在网关侧 |
| `htbi-curator` 本地写知识文件 | 本项目知识沉淀**必须**走后台审核，不在本地完成 |

---

## 四、落地路径

P0 项（references 三件套 + 候选变更生成器）建议在 **M1 阶段** 落地，与后台首版同步设计 schema。
P1 项（报告校验 + PreToolUse hook + SKILL 模板）建议在 **M1 末/M2 初** 引入，作为质量门。
P2 项（change-log + onboarding skill + configSchema）可与 M2 一起做，提升体验。

---

## 五、待用户拍板

| # | 待确认 | 影响 |
|---|--------|------|
| A | 是否同意将 data-collector 升级为「候选变更生成器」（输出结构化 diff，而非自由文本） | 决定反馈池数据结构与后台审核 UI |
| B | 是否引入 `validate_report` 强制门（不通过不许交付） | 影响分析师工作流硬约束程度 |
| C | references 三件套是否包含 methods.md（即是否新增 method_defs 表） | 数据模型多一张表，但能沉淀方法学 |
| D | 用户侧 change-log 用"系统消息"提示是否过于打扰，是否改为 `/<插件名> changelog` 命令按需查看 | UX 取舍 |
