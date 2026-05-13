---
name: multi-agent-meeting-skill
description: 多agent会议控制Skill v2.0。支持主持人主动点名、议题状态追踪、共识提案流程、跨Agent消息桥接，彻底实现多Agent完整自动会议。
---

# Multi System Meeting Skill

## 1. 目标

在 OpenClaw 中基于插件 `multi-agent-meeting` 编排一次完整的多 Agent 自动会议，实现：

- **主持人主动主导**：主 Agent（小会）全程负责会议生命周期与议程控制
- **议题级闭环**：每个议题独立状态追踪、讨论轮次控制、共识验证
- **Agent 间充分互动**：支持主持人点名提问、定向答疑、异议收集
- **结果可交付**：最终产出 summary、action items、导出文件路径、决议记录

## 2. 执行主体约定（强制）

- **主 agent（Host/小会）**：唯一流程控制者，负责会议生命周期管理、议程推进、发言协调、共识收敛
- **其他 agent**：仅参与讨论和任务执行，不直接推进会议状态
- **人类用户**：负责确认关键输入、关键决策、关键结果

### 2.1 命令执行边界（必须遵守）

- 终端 CLI（人类在终端执行）：`openclaw ...`
- 插件工具（主 agent 通过工具调用执行）：所有 `meeting_*`、`agenda_*`、`speaking_*`、`voting_*`、`recording_*`、`output_*`、`consensus_*`、`communication_*`
- 禁止：主 agent 在对话中假装已执行 CLI 并返回结果

## 3. 存储口径（必须一致）

- 主数据（会议状态、议程、投票、任务索引）：PostgreSQL
- 连接来源：`pgDsn`（插件配置）或 `PG_DSN`（环境变量）
- 导出目录：`storageDir`（summary/actions/transcript 等文件产物）
- 禁止把导出目录文件当作会议主状态来源；主状态以工具查询结果为准

## 4. 触发条件

命中以下任一意图，立即触发本 Skill：

- 帮我开会 / 组织一次多 Agent 会议
- 做头脑风暴 / 需求评审 / 技术评审 / 项目启动会
- 让多个 Agent 协同讨论并产出结论/任务
- 让 Agent 之间可以互相提问、讨论、达成共识

## 5. MUST-FIRST（最高优先级）

1. 立即进入会议编排模式，不先走普通闲聊
2. 首轮交互先尝试问题卡片；仅当 channel 不支持卡片时改文本
3. 未收集完必备输入，禁止 `meeting_create`
4. 未执行并成功 `agenda_confirm`，禁止 `meeting_start`
5. 未通过 `meeting_start_readiness(can_start=true)`，禁止 `meeting_start`
6. 每个阶段成功后都必须发送一次进展通知
7. **v2 新增**：每个议题必须走完「pending → discussing → consensus_checking → resolved」完整闭环

## 6. 主流程 Quick Start（v2.0 强制参考）

```pseudo
1) set interaction_mode = card|text
2) openclaw agents list（用户终端执行） -> 用户勾选 participants >= 2
3) 一次性收集必备输入: theme/purpose/type/expected_duration/participants
4) meeting_create -> 获取 meeting_id
5) 生成议程草案（4~7项） -> 逐条 agenda_add_item（自动初始化议题状态为 pending）
6) 向用户展示并修改议程（可 update/remove/reorder）
7) 用户最终确认后调用 agenda_confirm -> 校验 agenda_confirmed=true
8) meeting_start_readiness -> 仅当 can_start=true 才 meeting_start
9) 逐议题完整闭环循环：
   a) 进入议题 → agenda_get_discussion_progress
   b) 主持人主动点名 → speaking_invite 邀请相关 Agent 先发言
   c) 定向提问 → speaking_raise_question 抛出具体问题给特定 Agent
   d) 多轮互动 → 可选 communication_relay_message 中转 Agent 间关键问题
   e) 共识提案 → consensus_propose 生成正式提案
   f) 收集反馈 → consensus_collect_feedback 逐个 Agent 收集同意/反对
   g) 完成验证 → 全部同意则 consensus_finalize；异议太大则 consensus_reopen
   h) 轮次耗尽 → 仍未达成共识则 agenda_escalate_to_arbitration
   i) 记录笔记 → recording_take_note
   j) 推进下一议题 → agenda_next_item
10) 如需投票决策: voting_create -> voting_cast -> voting_get_result -> voting_end
11) 任务闭环: meeting_assign_task -> meeting_update_task_status -> meeting_record_task_result
12) 会后收口: output_generate_summary -> output_generate_action_items -> output_export -> meeting_end
```

## 7. 交互模式决策

- `if channel_supports_card == true`：卡片收集与确认
- `else`：纯文本收集与确认

主 agent 必须在上下文记录：

- `interaction_mode`: `card` or `text`
- `interaction_reason`: 使用原因

首轮固定话术：

- 卡片模式：`已进入会议编排模式(v2.0)，我先用问题卡片收集会议输入。`
- 文本模式：`已进入会议编排模式(v2.0)；当前频道不支持问题卡片，改用纯文本收集输入。`

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

## 9. v2 新增核心工具说明

### 9.1 主持人主动调度工具（direct-speaking-tools）

| 工具名 | 用途 | 典型场景 |
|--------|------|----------|
| `speaking_invite` | 主持人主动点名邀请特定 Agent 发言 | 议题开始时请产品经理先讲需求，或有异议时请架构师答疑 |
| `speaking_raise_question` | 主持人向特定 Agent 定向抛出具体问题 | 对方案有疑问，直接点名后端 Agent 解答 |
| `speaking_pause_topic` | 临时暂停当前议题 | 当前遇到分歧暂时无法达成一致，先跳其他议题 |

### 9.2 共识流程工具（consensus-tools）

| 工具名 | 用途 | 典型场景 |
|--------|------|----------|
| `consensus_propose` | 主持人基于讨论结果，给出正式共识提案 | 几轮讨论后，整理核心观点生成一份结构化提案 |
| `consensus_collect_feedback` | 逐个收集 Agent 对提案的同意/反对反馈 | 提案生成后，逐个 Agent 确认态度 |
| `consensus_finalize` | 标记议题已达成最终共识 | 全部 Agent 同意，正式 mark 该议题为 resolved |
| `consensus_reopen` | 发现异议太大时，重开讨论 | 反对过多，主持人需要重新引导讨论 |
| `agenda_get_discussion_progress` | 获取当前议题完整讨论进度 | 查看已用轮次、已发言列表、待处理问题 |
| `agenda_escalate_to_arbitration` | 整理分歧，升级等待用户仲裁 | 讨论轮次耗尽仍未达成一致，交人类决策者拍板 |

### 9.3 跨 Agent 消息桥接工具（communication-tools）

| 工具名 | 用途 | 典型场景 |
|--------|------|----------|
| `communication_relay_message` | 安全中转 Agent 间关键问题与答疑 | 前端 Agent 的问题，主持人定向转发给后端 Agent |
| `communication_list_unresolved_questions` | 列出当前悬而未决的问题 | 议题结束前，检查是否还有遗漏未答的问题 |
| `communication_generate_context_snapshot` | 生成议题上下文快照 | 新 Agent 加入或被邀请发言时，快速获取背景信息 |

## 10. v2 议题级状态机（新增）

每个议程项独立追踪以下状态：

`pending` → `discussing` → `consensus_checking` → `resolved`（成功）或 `blocked`（升级仲裁）

### 10.1 状态流转规则

- `pending`：新创建议题的初始状态
- `discussing`：进入该议题，开始讨论时切换
- `consensus_checking`：生成了共识提案，正在收集反馈时
- `resolved`：全员达成一致，正式完成议题
- `blocked`：讨论轮次耗尽或分歧太大，升级给用户仲裁

## 11. 生命周期与门禁

概念状态机：

`DRAFT -> CREATED -> AGENDA_CONFIRMED -> IN_PROGRESS -> WRAP_UP -> ENDED`

说明：

- 上述是概念流程态；实际字段以 `meeting_get` 返回为准
- 任何阶段推进都以插件工具返回值为准，不靠推测

### 11.1 强门禁（v2 扩展）

- 未完成必备输入：禁止 `meeting_create`
- 未 `agenda_confirm`：禁止 `meeting_start`
- `meeting_start_readiness.can_start != true`：禁止 `meeting_start`
- **v2 新增**：当前议题未达成 consensus_finalize，禁止 agenda_next_item
- `status=ended` 后禁止继续会议写入动作（仅允许查询/导出）

### 11.2 confirm 失效规则（必须重确认）

若执行过以下任一工具，确认立即失效，必须再次 `agenda_confirm`：

- `agenda_add_item`
- `agenda_update_item`
- `agenda_remove_item`
- `agenda_reorder_items`

## 12. 阶段进展通知规范（强制）

每个阶段成功都必须通知，至少覆盖：

1. Agent 列表获取完成
2. 必备输入收集完成
3. `meeting_create` 成功（附 `meeting_id`）
4. 议程草案完成（附议题数）
5. `agenda_confirm` 成功
6. readiness 通过
7. `meeting_start` 成功
8. **v2 新增**：每个议题 consensus_finalize 完成
9. **v2 新增**：共识提案反馈收集结果
10. 每次投票完成
11. 任务分配完成
12. 会后产出完成
13. `meeting_end` 成功

标准模板：

`[会议进展] <阶段> 已完成 | meeting_id=<ID> | status=<status> | next=<next_step>`

失败模板：

`[会议进展] <阶段> 失败 | error_code=<code> | 建议动作=<required_action>`

## 13. 工具编排主流程（v2.0 详细）

### 13.1 会前选人

- 人类在终端执行：`openclaw agents list`
- 主 agent 向用户展示可选 Agent，要求勾选 >=2
- 无法获取列表时，要求用户手动提供参会 Agent

### 13.2 创建会议

调用 `meeting_create`，参数必须完整：

- `theme`
- `purpose`
- `type`
- `expected_duration`
- `participants`

记录 `meeting_id`。

### 13.3 议程生成与确认

- 生成 4~7 个议程项，逐条 `agenda_add_item`（会自动初始化议题状态为 pending）
- 展示给用户确认，可 `agenda_update_item / agenda_remove_item / agenda_reorder_items`
- 用户明确确认开始后，调用 `agenda_confirm`
- 校验返回 `agenda_confirmed=true`

### 13.4 启动会议

- 调用 `meeting_start_readiness`
- 仅当 `can_start=true` 调用 `meeting_start`

### 13.5 议题完整闭环流程（v2 重点）

每个议题完整按以下步骤执行：

#### 步骤 A：进入议题讨论
1. 调用 `meeting_get` 读取当前议题
2. 调用 `agenda_get_discussion_progress` 获取初始状态
3. 更新议题状态为 `discussing`（如果还是 pending）

#### 步骤 B：主动点名与定向提问
1. 根据议题性质，用 `speaking_invite` 邀请最相关的 Agent 先发言（如产品先讲需求，后端讲技术方案）
2. 根据 Agent 发言内容，用 `speaking_raise_question` 抛出具体问题给其他 Agent
3. 可选：用 `communication_relay_message` 中转关键问题与答疑

#### 步骤 C：共识提案与反馈收集
1. 多轮讨论后（一般默认3轮），调用 `consensus_propose` 生成正式共识提案
2. 逐个 Agent 调用 `consensus_collect_feedback` 收集同意/反对反馈
3. 调用 `agenda_get_discussion_progress` 查看同意率

#### 步骤 D：决议确认或重开
1. **全员同意** → 调用 `consensus_finalize` 标记议题 `resolved`
2. **异议太大** → 调用 `consensus_reopen` 回到 `discussing`，重新引导讨论
3. **轮次耗尽仍未达成** → 调用 `agenda_escalate_to_arbitration` 整理分歧升级给用户

#### 步骤 E：收尾记录
1. `recording_take_note` 记录核心结论
2. `agenda_next_item` 推进下一议题

### 13.6 投票子流程（按需）

- `voting_create`
- 全员 `voting_cast`
- `voting_get_result`
- `voting_end`
- 若平票/无共识：
  - 最多再讨论+重投 1 次
  - 仍无共识则请求用户裁决，必要时 `voting_override`

### 13.7 任务闭环

- `meeting_assign_task`
- `meeting_update_task_status`
- `meeting_record_task_result`
- 会末 `meeting_list_tasks` 输出完成率

### 13.8 会后收口

- `output_generate_summary`
- `output_generate_action_items`
- `output_export`（建议 markdown）
- `meeting_end`

## 14. 四类会议场景策略

> 使用说明：主 agent 在议程生成前，应将对应场景的议程建议作为 constraints 输入。

### 14.1 brainstorm

- 议程建议：问题定义 → 发散创意 → 聚类 → 共识提案 → 落地责任
- 共识策略：`consensus_propose` 后快速收集反馈，有创意亮点即可
- 产出重点：Top 点子、试点方案、负责人

### 14.2 requirement_review

- 议程建议：背景目标 → 逐条评审 → 风险边界 → 范围确认 → 里程碑
- 共识策略：每条评审点单独走共识流程，争议点单独升级
- 产出重点：通过/退回条目、变更清单、验收口径

### 14.3 tech_review

- 议程建议：方案陈述 → 对比评估 → 争议点深入 → 共识提案 → 实施计划
- 共识策略：架构师 Agent 先邀请发言，技术争议点定向提问
- 产出重点：最终方案、技术债、执行计划

### 14.4 project_kickoff

- 议程建议：目标范围 → 角色协作 → 里程碑依赖 → 风险预案 → 启动确认
- 共识策略：每个里程碑单独走共识
- 产出重点：RACI、里程碑、风险台账、首周任务

## 15. 典型错误与回退分支

### 15.1 工具不可见/未加载

典型信号：`TOOL_NOT_AVAILABLE` / 工具列表缺失。

处理：

1. 停止推进会议状态
2. 提示检查：`openclaw plugins inspect multi-agent-meeting`
3. 检查 `pgDsn/PG_DSN` 与数据库连通性
4. 重启 Gateway 并重开会话

### 15.2 议程未确认

典型信号：`AGENDA_NOT_CONFIRMED` 或 message 包含 `Agenda must be confirmed`。

处理：

1. 回到议程确认步骤
2. 若中间改过议程，必须重新 `agenda_confirm`
3. 再次执行 readiness，再尝试 start

### 15.3 数据库不可用

典型信号：连接失败/鉴权失败/超时。

处理：

1. 明确提示主存储不可用，暂停流程推进
2. 检查 `pgDsn/PG_DSN`
3. 检查 PostgreSQL 实例可用性和网络连通性
4. 恢复后从 `meeting_list` + `meeting_get` 续跑

### 15.4 LLM JSON 输出失败

处理：

1. 同节点重试 1 次，明确只输出 JSON
2. 再失败则进入最小安全动作：
   - 不执行关键写操作
   - 向用户索取最小必要信息
   - 等用户确认后再继续

### 15.5 v2 新增：议题讨论阻塞

典型信号：连续多轮 `consensus_reopen` 仍无法达成一致。

处理：

1. 调用 `agenda_get_discussion_progress` 检查待处理问题数和轮次使用情况
2. 调用 `speaking_pause_topic` 临时暂停，切换到下一个议题
3. 全部议题完成后，再回来尝试解决阻塞议题
4. 仍无进展则调用 `agenda_escalate_to_arbitration` 升级给用户

## 16. 会后最终回复结构（v2 扩展，必须包含）

- 会议信息：`meeting_id`、主题、类型、时长
- 议题完成度：已完成/总议题、共识达成率
- 核心结论：决策点（带共识记录）
- 行动项：负责人、状态
- 导出路径：`output_export` 返回路径
- 未决事项与下一步建议（如有升级仲裁的问题）

## 17. 执行约束

- 严禁使用不存在的工具名或字段名
- 工具参数必须与插件 schema 一致
- 信息未确认时先提问，不得臆造输入/投票/结果
- 关键决策（平票、无共识、重大取舍）必须用户确认
- **v2 新增**：每个议题必须走完共识闭环，不得跳步
