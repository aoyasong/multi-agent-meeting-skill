---
name: multi-agent-meeting-skill
description: 多agent会议控制Skill。命中开会/头脑风暴/评审/项目启动意图时立即触发，按强约束流程编排会议。主数据使用PostgreSQL，导出文件使用storageDir。
---

# Multi System Meeting

## 1. 目标

在 OpenClaw 中基于插件 `multi-agent-meeting-plugin` 编排一次多 Agent 会议，保证：

- 流程可执行：创建 -> 议程 -> 确认 -> 启动 -> 讨论/投票 -> 任务 -> 产出 -> 结束。
- 约束可落地：工具边界清晰、门禁明确、异常可恢复。
- 结果可交付：最终产出 summary、action items、导出文件路径。

## 2. 执行主体约定（强制）

- 主 agent：唯一流程控制者，负责调用插件工具和推进会议状态。
- 其他 agent：仅参与讨论和任务执行，不直接推进会议状态。
- 人类用户：负责确认关键输入、关键决策、关键结果。

### 2.1 命令执行边界（必须遵守）

- 终端 CLI（人类在终端执行）：`openclaw ...`
- 插件工具（主 agent 通过工具调用执行）：`meeting_*`、`agenda_*`、`speaking_*`、`voting_*`、`recording_*`、`output_*`
- 禁止：主 agent 在对话中假装已执行 CLI 并返回结果。

## 3. 存储口径（必须一致）

- 主数据（会议状态、议程、投票、任务索引）：PostgreSQL。
- 连接来源：`pgDsn`（插件配置）或 `PG_DSN`（环境变量）。
- 导出目录：`storageDir`（summary/actions/transcript 等文件产物）。
- 禁止把导出目录文件当作会议主状态来源；主状态以工具查询结果为准。

## 4. 触发条件

命中以下任一意图，立即触发本 Skill：

- 帮我开会 / 组织一次多 Agent 会议
- 做头脑风暴 / 需求评审 / 技术评审 / 项目启动会
- 让多个 Agent 协同讨论并产出结论/任务

## 5. MUST-FIRST（最高优先级）

1. 立即进入会议编排模式，不先走普通闲聊。
2. 首轮交互先尝试问题卡片；仅当 channel 不支持卡片时改文本。
3. 未收集完必备输入，禁止 `meeting_create`。
4. 未执行并成功 `agenda_confirm`，禁止 `meeting_start`。
5. 未通过 `meeting_start_readiness(can_start=true)`，禁止 `meeting_start`。
6. 每个阶段成功后都必须发送一次进展通知。

## 6. 主流程 Quick Start（强制参考）

```pseudo
1) set interaction_mode = card|text
2) openclaw agents list（用户终端执行） -> 用户勾选 participants >= 2
3) 一次性收集必备输入: theme/purpose/type/expected_duration/participants
4) meeting_create -> 获取 meeting_id
5) 生成议程草案（4~7项） -> 逐条 agenda_add_item
6) 向用户展示并修改议程（可 update/remove/reorder）
7) 用户最终确认后调用 agenda_confirm -> 校验 agenda_confirmed=true
8) meeting_start_readiness -> 仅当 can_start=true 才 meeting_start
9) 逐议题循环: speaking_request -> speaking_grant -> recording_take_note -> speaking_release
10) 如需决策: voting_create -> voting_cast -> voting_get_result -> voting_end
11) 任务闭环: meeting_assign_task -> meeting_update_task_status -> meeting_record_task_result
12) 会后收口: output_generate_summary -> output_generate_action_items -> output_export -> meeting_end
```

## 7. 交互模式决策

- `if channel_supports_card == true`：卡片收集与确认。
- `else`：纯文本收集与确认。

主 agent 必须在上下文记录：

- `interaction_mode`: `card` or `text`
- `interaction_reason`: 使用原因

首轮固定话术：

- 卡片模式：`已进入会议编排模式，我先用问题卡片收集会议输入。`
- 文本模式：`已进入会议编排模式；当前频道不支持问题卡片，改用纯文本收集输入。`

## 8. 必备输入（一次性收集）

- `theme`（会议主题）
- `purpose`（会议目的）
- `type`（`brainstorm|requirement_review|tech_review|project_kickoff`）
- `participants`（至少 2 个，`agent_id + role`）
- `expected_duration`（分钟）

### 8.1 问题卡片字段模板

- 会议主题（短文本，必填）
- 会议目的（多行文本，必填）
- 会议类型（单选，必填）
- 参会 Agent（多选，>=2，来源于 `openclaw agents list`）
- 预计时长（数字，分钟，默认 60）

### 8.2 纯文本问句模板

请按顺序确认：

1. 本次会议主题是什么？
2. 会议希望达成什么目标？
3. 会议类型是：头脑风暴/需求评审/技术评审/项目启动？
4. 参会 Agent 列表（至少2个，给出 `agent_id + role`）？
5. 预计时长多少分钟？

## 9. 生命周期与门禁

概念状态机：

`DRAFT -> CREATED -> AGENDA_CONFIRMED -> IN_PROGRESS -> WRAP_UP -> ENDED`

说明：

- 上述是概念流程态；实际字段以 `meeting_get` 返回为准（如 `created|in_progress|ended`）。
- 任何阶段推进都以插件工具返回值为准，不靠推测。

强门禁：

- 未完成必备输入：禁止 `meeting_create`
- 未 `agenda_confirm`：禁止 `meeting_start`
- `meeting_start_readiness.can_start != true`：禁止 `meeting_start`
- `status=ended` 后禁止继续会议写入动作（仅允许查询/导出）

### 9.1 confirm 失效规则（必须重确认）

若执行过以下任一工具，确认立即失效，必须再次 `agenda_confirm`：

- `agenda_add_item`
- `agenda_update_item`
- `agenda_remove_item`
- `agenda_reorder_items`

## 10. 阶段进展通知规范（强制）

每个阶段成功都必须通知，至少覆盖：

1. Agent 列表获取完成
2. 必备输入收集完成
3. `meeting_create` 成功（附 `meeting_id`）
4. 议程草案完成（附议题数）
5. `agenda_confirm` 成功
6. readiness 通过
7. `meeting_start` 成功
8. 每个议题完成
9. 每次投票完成
10. 任务分配完成
11. 会后产出完成
12. `meeting_end` 成功

标准模板：

`[会议进展] <阶段> 已完成 | meeting_id=<ID> | status=<status> | next=<next_step>`

失败模板：

`[会议进展] <阶段> 失败 | error_code=<code> | 建议动作=<required_action>`

## 11. 工具编排主流程（详细）

### 11.1 会前选人

- 人类在终端执行：`openclaw agents list`
- 主 agent 向用户展示可选 Agent，要求勾选 >=2。
- 无法获取列表时，要求用户手动提供参会 Agent。

### 11.2 创建会议

调用 `meeting_create`，参数必须完整：

- `theme`
- `purpose`
- `type`
- `expected_duration`
- `participants`

记录 `meeting_id`。

### 11.3 议程生成与确认

- 生成 4~7 个议程项，逐条 `agenda_add_item`
- 展示给用户确认，可 `agenda_update_item / agenda_remove_item / agenda_reorder_items`
- 用户明确确认开始后，调用 `agenda_confirm`
- 校验返回 `agenda_confirmed=true`

### 11.4 启动会议

- 调用 `meeting_start_readiness`
- 仅当 `can_start=true` 调用 `meeting_start`

### 11.5 议程循环

每个议题执行：

1. `meeting_get` 读取当前议题
2. `speaking_request`（申请）
3. `speaking_grant`（授予）
4. `recording_take_note`（记录）
5. `speaking_release`（释放）
6. `agenda_next_item`（推进）

### 11.6 投票子流程（按需）

- `voting_create`
- 全员 `voting_cast`
- `voting_get_result`
- `voting_end`
- 若平票/无共识：
  - 最多再讨论+重投 1 次
  - 仍无共识则请求用户裁决，必要时 `voting_override`

### 11.7 任务闭环

- `meeting_assign_task`
- `meeting_update_task_status`
- `meeting_record_task_result`
- 会末 `meeting_list_tasks` 输出完成率

### 11.8 会后收口

- `output_generate_summary`
- `output_generate_action_items`
- `output_export`（建议 markdown）
- `meeting_end`

## 12. 四类会议场景策略

> 使用说明：主 agent 在议程生成前，应将对应场景的议程建议作为 constraints 输入。

### 12.1 brainstorm

- 议程建议：问题定义 -> 发散创意 -> 聚类 -> 投票 -> 落地
- 投票建议：`simple + simple`
- 产出重点：Top 点子、试点方案、负责人

### 12.2 requirement_review

- 议程建议：背景目标 -> 逐条评审 -> 风险边界 -> 范围确认 -> 里程碑
- 投票建议：`yes_no_abstain + moderate`
- 产出重点：通过/退回条目、变更清单、验收口径

### 12.3 tech_review

- 议程建议：方案陈述 -> 对比评估 -> 争议点 -> 决策投票 -> 实施/回滚
- 投票建议：`ranked + complex`
- 产出重点：最终方案、技术债、执行计划

### 12.4 project_kickoff

- 议程建议：目标范围 -> 角色协作 -> 里程碑依赖 -> 风险预案 -> 启动确认
- 投票建议：`yes_no_abstain + simple`
- 产出重点：RACI、里程碑、风险台账、首周任务

## 13. 典型错误与回退分支

### 13.1 工具不可见/未加载

典型信号：`TOOL_NOT_AVAILABLE` / 工具列表缺失。

处理：

1. 停止推进会议状态。
2. 提示检查：`openclaw plugins inspect multi-agent-meeting-plugin`
3. 检查 `pgDsn/PG_DSN` 与数据库连通性。
4. 重启 Gateway 并重开会话。

### 13.2 议程未确认

典型信号：`AGENDA_NOT_CONFIRMED` 或 message 包含 `Agenda must be confirmed`。

处理：

1. 回到议程确认步骤。
2. 若中间改过议程，必须重新 `agenda_confirm`。
3. 再次执行 readiness，再尝试 start。

### 13.3 数据库不可用

典型信号：连接失败/鉴权失败/超时。

处理：

1. 明确提示主存储不可用，暂停流程推进。
2. 检查 `pgDsn/PG_DSN`。
3. 检查 PostgreSQL 实例可用性和网络连通性。
4. 恢复后从 `meeting_list` + `meeting_get` 续跑。

### 13.4 LLM JSON 输出失败

处理：

1. 同节点重试 1 次，明确只输出 JSON。
2. 再失败则进入最小安全动作：
   - 不执行关键写操作
   - 向用户索取最小必要信息
   - 等用户确认后再继续

## 14. 会后最终回复结构（必须包含）

- 会议信息：`meeting_id`、主题、类型、时长
- 核心结论：决策点
- 行动项：负责人、状态
- 导出路径：`output_export` 返回路径
- 未决事项与下一步建议

## 15. 执行约束

- 严禁使用不存在的工具名或字段名。
- 工具参数必须与插件 schema 一致。
- 信息未确认时先提问，不得臆造输入/投票/结果。
- 关键决策（平票、无共识、重大取舍）必须用户确认。
