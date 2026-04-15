---
name: "multi-system-meeting-orchestrator"
description: "使用 multi-agent-meeting-plugin 组织多Agent会议。用户要求开会/评审/头脑风暴/项目启动且需全流程闭环时触发。"
---

# 多场景会议总控 Skill（OpenClaw）

## 目标

在 OpenClaw 中基于插件 `multi-agent-meeting-plugin` 完整编排一次多 Agent 会议，确保：

- 覆盖会议全生命周期（创建 -> 启动 -> 议程推进 -> 讨论/投票 -> 产出 -> 结束）。
- 支持四类核心场景（头脑风暴、需求评审、技术评审、项目启动）。
- 有明确门禁、异常分支、重试与收敛条件，避免会议“挂起”。

## 触发条件

当用户出现以下意图时立即触发本 Skill：

- “帮我开会 / 组织一次多 Agent 会议”
- “做头脑风暴 / 需求评审 / 技术评审 / 项目启动会”
- “让多个 Agent 协同讨论并产出结论/任务”

## 必备输入（缺一则先提问补齐）

1. 会议主题（`theme`）
2. 会议目的（`purpose`）
3. 会议类型（`type`）：
   - `brainstorm`
   - `requirement_review`
   - `tech_review`
   - `project_kickoff`
4. 参会 Agent 列表（至少 2 个，结构：`agent_id + role`）
5. 预计时长（`expected_duration`，分钟）

## 生命周期状态机（强约束）

`DRAFT -> CREATED -> AGENDA_DRAFTED -> AGENDA_CONFIRMED -> STARTED/IN_PROGRESS -> AGENDA_LOOP -> WRAP_UP -> ENDED`

### 状态门禁

- `CREATED` 阶段先生成议程草案，再进入用户确认。
- 只有 `AGENDA_CONFIRMED` 才能 `meeting_start`。
- `started`/`in_progress` 才能分配任务、推进议程、投票。
- `ended` 后禁止再写入讨论动作（仅允许查询与导出既有结果）。

## 工具编排主流程

### 0) 会前准备（允许同主题多次开会）

- 不做“同主题防重复开会”拦截。
- 允许同主题会议多次创建，通过 `meeting_id` 天然区分不同会议实例。
- 如需更强可读性，可在主题后追加序号（例如：`支付重构评审-第2次`）。

### 1) 创建会议

调用 `meeting_create`，必须传：

- `theme`
- `purpose`
- `type`
- `expected_duration`
- `participants`

记录返回 `meeting_id`。

### 2) 初始化议程

根据会议类型生成 4\~7 个议程项，逐条调用 `agenda_add_item`：

- 必填：`meeting_id`、`title`、`expected_duration`
- 建议填：`description`、`time_limit`、`materials`

### 2.1) 用户确认议程（新增强制环节）

- 主 agent 必须把议程草案展示给用户确认。
- 用户可执行以下修改指令：
  - 调整议题顺序
  - 增删议题
  - 修改单个议题时长与描述
- 每次修改后，主 agent 重新展示最新议程，直到用户明确回复“确认开始”。

确认后调用 `meeting_start` 启动会议。

### 3) 议程循环（每个议题都执行）

1. 调用 `meeting_get` 获取 `current_agenda_index` 与议程状态。
2. 在当前议题做“发言编排”：
   - 参会 Agent 先 `speaking_request`
   - 主持按优先级 `speaking_grant`
   - 发言结束 `speaking_release`
3. 每次发言后立刻 `recording_take_note`（必须写入 `agenda_item_id`）。
4. 需要决策时进入投票子流程（见下一节）。
5. 议题收束后调用 `agenda_next_item` 推进下一项。
6. 若 `agenda_next_item` 返回“已是最后一项”，退出议程循环。

### 4) 投票子流程（按需）

1. `voting_create`
   - 必填：`meeting_id`、`topic`、`options`、`type`、`window_type`
   - 可选：`agenda_item_id`、`description`
2. 全员投票：逐个调用 `voting_cast`（`option_id` 必须来自创建返回的选项 ID）。
3. 读结果：`voting_get_result`
4. 关投票：`voting_end`
5. 若平票/无共识：
   - 先追加一轮短讨论 + 再投一次（最多 1 次）
   - 仍无共识则请求用户裁决，必要时调用 `voting_override`

### 5) 任务闭环（必须执行）

至少创建一批可执行任务：

1. `meeting_assign_task`：把任务分配给非主持 Agent。
2. 执行中可用 `meeting_update_task_status` 标记 `in_progress`。
3. 收到结果后 `meeting_record_task_result`。
4. 会末调用 `meeting_list_tasks` 统计完成率。

### 6) 会后产出与结束

按顺序调用：

1. `output_generate_summary`
2. `output_generate_action_items`
3. `output_export`（建议 `format: "markdown"`，`content: ["summary","transcript","actions"]`）
4. `meeting_end`

## 四类场景模板（核心差异）

## A. 头脑风暴（`brainstorm`）

- 议程建议：
  - 问题定义
  - 发散创意
  - 创意聚类
  - 优先级投票
  - 行动落地
- 投票策略：`type: "simple"` + `window_type: "simple"`
- 产出重点：Top 想法、试点方案、责任分工

## B. 需求评审（`requirement_review`）

- 议程建议：
  - 背景与目标
  - 需求逐条评审
  - 风险与边界
  - 范围确认投票
  - 里程碑与负责人
- 投票策略：`type: "yes_no_abstain"` + `window_type: "moderate"`
- 产出重点：通过/退回条目、变更清单、验收口径

## C. 技术评审（`tech_review`）

- 议程建议：
  - 候选方案陈述
  - 成本/性能/风险对比
  - 关键争议点讨论
  - 方案决策投票
  - 实施与回滚计划
- 投票策略：`type: "ranked"` + `window_type: "complex"`
- 产出重点：最终方案、技术债、实施分工与时间窗

## D. 项目启动（`project_kickoff`）

- 议程建议：
  - 目标与范围
  - 角色与协作机制
  - 里程碑与依赖
  - 风险预案
  - 启动确认投票
- 投票策略：`type: "yes_no_abstain"` + `window_type: "simple"`
- 产出重点：RACI、里程碑、风险台账、首周任务

## 异常分支与恢复策略

### 1) Agent 无响应

- 单个 Agent 超时：记录笔记并继续下一位，不阻塞全局。
- 连续两轮无响应：标记该 Agent 仅观察模式，后续仍可被重新拉回。

### 2) 发言拥塞

- 使用 `speaking_status` 查看队列；同议题每位 Agent 最多连续 1 次发言。

### 3) 投票失效

- 若 `voting_cast` 出现非法选项，立即重读 `voting_get_result` 并提示合法 `option_id` 重投。

### 4) 进程中断恢复

恢复步骤固定：

1. `meeting_list` 查 `in_progress`
2. `meeting_get` 读取当前议程索引
3. `recording_get_transcript` 拉取最近记录
4. 从当前议题继续，不重建会议

## 结束判定（收敛条件）

只有满足全部条件才可结束会议：

1. 议程已全部处理（或用户明确提前结束）。
2. 至少生成一次总结（`output_generate_summary` 成功）。
3. 已提取行动项（`output_generate_action_items` 成功）。
4. 任务列表已统计（`meeting_list_tasks` 已调用）。

## 输出给用户的最终结构

会后回复必须包含：

1. 会议基本信息（`meeting_id`、主题、类型、时长）
2. 核心结论（决策点）
3. 行动项（负责人、状态）
4. 导出文件路径（来自 `output_export`）
5. 未决事项与建议下一步

## 执行约束

- 严禁使用不存在的工具名或字段名。
- 工具参数必须与插件实际 schema 一致。
- 未确认关键信息时先提问，不得臆造会议输入或投票结果。

## LLM 介入策略（已对齐）

### 介入范围

- 标准介入：议程生成、发言结构化、决策建议、任务拆解、会后总结。
- Prompt 组织：分节点模板。
- 输出格式：强约束 JSON。
- 决策边界：LLM 仅给建议，关键决策必须用户确认。

### 关键确认门

- 门1（会前）：会议合同确认（目标/范围/参与者/时长/产出）。
- 门1.5（会前）：议程草案确认与可修改（顺序/增删/时长/描述）。
- 门2（会中）：平票、无共识、重大取舍时请求用户裁决。
- 门3（会后）：最终产出确认后再发布执行。

### 节点1：议程生成模板

```text
[系统角色]
你是会议编排助手。请基于输入生成“会议议程草案”，仅输出 JSON。

[输入]
{
  "meeting": {
    "theme": "...",
    "purpose": "...",
    "type": "brainstorm|requirement_review|tech_review|project_kickoff",
    "expected_duration": 60,
    "participants": [{"agent_id":"a1","role":"participant"}]
  },
  "user_instruction": "...",
  "constraints": ["议程项4-7个","每项时长>=1分钟","总时长尽量不超 expected_duration"]
}

[输出JSON Schema]
{
  "agenda_items":[
    {
      "title":"string",
      "description":"string",
      "expected_duration": number,
      "time_limit": number|null,
      "materials":["string"],
      "rationale":"string"
    }
  ],
  "risks":["string"],
  "need_user_confirmation": boolean,
  "confirmation_questions":["string"]
}

[硬约束]
- 只输出合法 JSON。
- expected_duration 必须为正整数。
- 信息不足时 need_user_confirmation=true 并给出问题。
```

### 节点2：发言结构化模板

```text
[系统角色]
你是会议记录结构化助手。将发言归类并提炼关键信息，仅输出 JSON。

[输入]
{
  "meeting_id":"...",
  "agenda_item_id":"...",
  "messages":[{"agent_id":"a1","raw_content":"...","timestamp":"..."}],
  "current_topic":"..."
}

[输出JSON Schema]
{
  "notes":[
    {
      "agent_id":"string",
      "raw_content":"string",
      "message_type":"statement|question|vote|insight|action",
      "confidence": number,
      "insight_tags":["risk|opportunity|decision|action"]
    }
  ],
  "summary_points":["string"],
  "open_questions":["string"]
}

[硬约束]
- 只输出 JSON。
- confidence 范围 0~1。
- 不得补充未出现事实。
```

### 节点3：决策建议与投票设计模板

```text
[系统角色]
你是会议决策顾问。根据讨论上下文给出决策路径建议，仅输出 JSON。

[输入]
{
  "agenda_context": "...",
  "discussion_summary":["..."],
  "candidate_options":["..."],
  "participants":["a1","a2","a3"]
}

[输出JSON Schema]
{
  "decision_required": boolean,
  "recommended_path":"continue_discussion|start_voting|escalate_user",
  "voting_plan":{
    "topic":"string",
    "type":"simple|ranked|yes_no_abstain",
    "window_type":"simple|moderate|complex",
    "options":["string"]
  },
  "reasoning":["string"],
  "need_user_confirmation": boolean,
  "confirmation_message":"string"
}

[硬约束]
- 关键决策默认 need_user_confirmation=true。
- recommended_path=start_voting 时 options 不能为空。
```

### 节点4：任务拆解模板

```text
[系统角色]
你是任务编排助手。基于议题结论生成任务建议，仅输出 JSON。

[输入]
{
  "meeting_id":"...",
  "agenda_item_id":"...",
  "decisions":["..."],
  "participants":[{"agent_id":"a1","role":"participant"}],
  "constraints":{"max_tasks":8}
}

[输出JSON Schema]
{
  "task_suggestions":[
    {
      "assignee_agent_id":"string",
      "title":"string",
      "description":"string",
      "output_format":"markdown|json|text|structured",
      "priority": number,
      "timeout_seconds": number|null,
      "reason":"string"
    }
  ],
  "coverage_check":{
    "uncovered_decisions":["string"],
    "duplications":["string"]
  },
  "need_user_confirmation": boolean
}

[硬约束]
- assignee_agent_id 必须来自 participants。
- priority 范围 1~10。
```

### 节点5：会后总结模板

```text
[系统角色]
你是会后交付助手。将会议结果整理成可提交版本，仅输出 JSON。

[输入]
{
  "meeting_info":{"meeting_id":"...","theme":"...","type":"..."},
  "agenda_summaries":["..."],
  "voting_results":["..."],
  "tasks":["..."]
}

[输出JSON Schema]
{
  "final_summary":"string",
  "key_decisions":["string"],
  "action_items":[
    {"item":"string","owner":"string|null","status":"pending|in_progress|completed|failed"}
  ],
  "risks":["string"],
  "next_steps":["string"],
  "need_user_confirmation": true,
  "confirmation_message":"string"
}

[硬约束]
- 关键交付前 need_user_confirmation 必须为 true。
- 不得编造未发生的投票与任务结果。
```

### 通用回退策略

- JSON 解析失败：同模板重试 1 次（低温、强调“只输出 JSON”）。
- 再失败：进入最小安全动作，不执行关键写操作。
- 最小安全动作：记录失败原因 -> 向用户索取最小必要信息 -> 等待确认。
