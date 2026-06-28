# Shannon 事件类型参考

本文档提供 Shannon 工作流发出的所有事件类型的全面参考。

## 概述

Shannon 在工作流执行的不同阶段发出事件，以提供任务处理、代理协调、LLM 交互和工具执行的实时可见性。事件遵循最小化、确定性模型，设计用于通过 SSE、WebSocket 或 gRPC 进行流式传输。

## 事件结构

所有事件共享一个通用结构：

```json
{
  "workflow_id": "task-00000000-0000-0000-0000-000000000002-1761545271",
  "type": "AGENT_STARTED",
  "agent_id": "simple-agent",
  "message": "Processing query",
  "timestamp": "2025-10-27T06:07:51.49277Z",
  "seq": 7,
  "stream_id": "1761545276767-0"
}
```

**字段：**
- `workflow_id`（字符串，必需）- 唯一工作流标识符
- `type`（字符串，必需）- 事件类型（参见以下分类）
- `agent_id`（字符串，可选）- 发出事件的代理
- `message`（字符串，可选）- 人类可读的描述
- `timestamp`（RFC3339，必需）- 事件时间戳
- `seq`（整数，必需）- 用于排序的单调递增序列号
- `stream_id`（字符串，可选）- 用于事件分组的流标识符

## 事件分类

### 核心工作流事件

标记主要工作流生命周期转换的事件。

#### `WORKFLOW_STARTED`
**描述：** 编排器开始任务处理
**代理 ID：** `orchestrator`
**示例消息：** `"Starting task"`
**典型序列：** 工作流中的首个事件

```json
{
  "type": "WORKFLOW_STARTED",
  "agent_id": "orchestrator",
  "message": "Starting up",
  "payload": {
    "task_context": {
      "attachments": [
        {"id": "abc123", "media_type": "image/png", "filename": "chart.png", "size_bytes": 14797}
      ]
    }
  },
  "seq": 1
}
```

**注意：** `payload.task_context.attachments` 仅在任务包含文件附件时存在。每个条目包含 Redis 引用 ID、MIME 类型、文件名和已解码的大小。

#### `WORKFLOW_COMPLETED`
**描述：** 工作流成功完成
**代理 ID：** 完成任务的代理
**示例消息：** `"All done"`
**典型序列：** 成功工作流的最终事件

```json
{
  "type": "WORKFLOW_COMPLETED",
  "agent_id": "simple-agent",
  "message": "All done",
  "seq": 12
}
```

#### `AGENT_STARTED`
**描述：** 代理开始处理任务
**代理 ID：** 代理标识符
**示例消息：** `"Processing query"`

```json
{
  "type": "AGENT_STARTED",
  "agent_id": "simple-agent",
  "message": "Processing query",
  "seq": 7
}
```

#### `AGENT_COMPLETED`
**描述：** 代理完成处理
**代理 ID：** 代理标识符
**示例消息：** `"Task complete"`

```json
{
  "type": "AGENT_COMPLETED",
  "agent_id": "simple-agent",
  "message": "Task complete",
  "seq": 11
}
```

#### `ERROR_OCCURRED`
**描述：** 工作流执行期间发生错误
**代理 ID：** 发生错误的代理（可选）
**示例消息：** 错误描述
**注意：** 可能在消息字段中包含错误详情

```json
{
  "type": "ERROR_OCCURRED",
  "agent_id": "data-agent",
  "message": "Failed to connect to database: timeout",
  "seq": 8
}
```

#### `AGENT_THINKING`
**描述：** 代理处于规划/推理阶段
**代理 ID：** 代理标识符
**示例消息：** `"Thinking: What is 5 + 5?"`
**注意：** 提供代理决策过程的可见性

```json
{
  "type": "AGENT_THINKING",
  "agent_id": "simple-agent",
  "message": "Thinking: What is 5 + 5?",
  "seq": 6
}
```

---

### LLM 事件

与大型语言模型交互相关的事件。

#### `LLM_PROMPT`
**描述：** 发送给 LLM 的已清理提示词
**代理 ID：** 进行 LLM 调用的代理
**示例消息：** 实际的提示词文本
**隐私：** 可能对 PII/敏感数据进行清理

```json
{
  "type": "LLM_PROMPT",
  "agent_id": "simple-agent",
  "message": "What is 5 + 5?",
  "seq": 8
}
```

#### `LLM_PARTIAL`
**描述：** 流式传输期间的增量 LLM 输出块
**代理 ID：** 接收流的代理
**示例消息：** 部分文本块
**注意：** 在 UI 中通常被过滤掉以获得更清晰的聊天历史
**过滤：** 被 `/api/v1/sessions/{sessionId}/events` 端点排除

```json
{
  "type": "LLM_PARTIAL",
  "agent_id": "simple-agent",
  "message": "5 + 5",
  "seq": 9
}
```

#### `LLM_OUTPUT`
**描述：** 最终完整的 LLM 响应
**代理 ID：** 进行 LLM 调用的代理
**示例消息：** 完整响应文本
**注意：** 这是最终答案，而非流式块

```json
{
  "type": "LLM_OUTPUT",
  "agent_id": "simple-agent",
  "message": "5 + 5 equals 10.",
  "seq": 10
}
```

#### `TOOL_OBSERVATION`
**描述：** 工具执行的结果/输出
**代理 ID：** 执行工具的代理
**示例消息：** 工具输出或观察结果

```json
{
  "type": "TOOL_OBSERVATION",
  "agent_id": "data-agent",
  "message": "Database query returned 42 rows",
  "seq": 15
}
```

---

### 多代理协调事件

用于多代理工作流和团队协作的事件。

#### `DELEGATION`
**描述：** 任务委托给另一个代理或工作流
**代理 ID：** 委托代理（可选）
**示例消息：** `"Processing as simple task"`、`"Coordinating multiple agents"`

```json
{
  "type": "DELEGATION",
  "message": "Processing as simple task",
  "seq": 4
}
```

#### `MESSAGE_SENT`
**描述：** 代理向另一个代理发送消息
**代理 ID：** 发送代理
**示例消息：** 消息内容
**功能门控：** 需要启用 `p2p_v1`

```json
{
  "type": "MESSAGE_SENT",
  "agent_id": "coordinator",
  "message": "Please analyze section 3",
  "seq": 12
}
```

#### `MESSAGE_RECEIVED`
**描述：** 代理收到消息
**代理 ID：** 接收代理
**示例消息：** 消息内容
**功能门控：** 需要启用 `p2p_v1`

```json
{
  "type": "MESSAGE_RECEIVED",
  "agent_id": "analyzer",
  "message": "Received task: analyze section 3",
  "seq": 13
}
```

#### `TEAM_RECRUITED`
**描述：** 新代理被招募到团队
**代理 ID：** 被招募代理的角色
**示例消息：** 招募的描述/原因
**功能门控：** 需要启用 `dynamic_team_v1`

```json
{
  "type": "TEAM_RECRUITED",
  "agent_id": "writer",
  "message": "Summarize section 3",
  "seq": 8
}
```

#### `TEAM_RETIRED`
**描述：** 代理从团队退休
**代理 ID：** 退休代理标识符
**示例消息：** 退休原因
**功能门控：** 需要启用 `dynamic_team_v1`

```json
{
  "type": "TEAM_RETIRED",
  "agent_id": "agent-xyz",
  "message": "Task completed",
  "seq": 20
}
```

#### `ROLE_ASSIGNED`
**描述：** 角色被分配给代理
**代理 ID：** 接收角色的代理
**示例消息：** 角色详情

```json
{
  "type": "ROLE_ASSIGNED",
  "agent_id": "agent-123",
  "message": "Assigned data_analyst role with 3 tools",
  "seq": 5
}
```

---

### 进度与状态事件

提供状态更新和进度信息的事件。

#### `PROGRESS`
**描述：** 步骤完成或进度更新
**代理 ID：** 报告进度的代理
**示例消息：** `"Created a plan with 1 step"`、`"Completed step 2 of 5"`
**用例：** 进度条、状态指示器

```json
{
  "type": "PROGRESS",
  "agent_id": "planner",
  "message": "Created a plan with 1 step",
  "seq": 3
}
```

#### `DATA_PROCESSING`
**描述：** 处理、分析或准备数据
**代理 ID：** 处理代理（可选）
**示例消息：** `"Preparing context"`、`"Analyzing results"`、`"Answer ready"`
**注意：** 用于各种处理阶段

```json
{
  "type": "DATA_PROCESSING",
  "message": "Preparing context",
  "seq": 1
}
```

#### `TEAM_STATUS`
**描述：** 多代理协调状态更新
**代理 ID：** 协调员或团队负责人
**示例消息：** 团队状态描述
**用例：** 多代理工作流可见性

```json
{
  "type": "TEAM_STATUS",
  "agent_id": "supervisor",
  "message": "3 agents active, 2 completed",
  "seq": 14
}
```

#### `WAITING`
**描述：** 等待资源、响应或批准
**代理 ID：** 等待代理
**示例消息：** 代理在等待什么

```json
{
  "type": "WAITING",
  "agent_id": "integration-agent",
  "message": "Waiting for API response",
  "seq": 9
}
```

#### `WORKSPACE_UPDATED`
**描述：** 工作区或上下文被修改
**代理 ID：** 更新工作区的代理
**示例消息：** 更新描述
**功能门控：** 需要启用 `p2p_v1`

```json
{
  "type": "WORKSPACE_UPDATED",
  "agent_id": "editor",
  "message": "Updated shared document",
  "seq": 16
}
```

---

### 工具执行事件

与工具和函数调用相关的事件。

#### `TOOL_INVOKED`
**描述：** 工具/函数执行开始
**代理 ID：** 调用工具的代理
**示例消息：** 工具名称和参数
**注意：** 可能包含已清理的参数

```json
{
  "type": "TOOL_INVOKED",
  "agent_id": "data-agent",
  "message": "Calling database_query with table=users",
  "seq": 10
}
```

---

### 人机交互事件

用于人机回环工作流的事件。

#### `APPROVAL_REQUESTED`
**描述：** 工作流请求人工批准
**代理 ID：** 请求批准的代理
**示例消息：** 批准请求详情
**用例：** 人机回环工作流

```json
{
  "type": "APPROVAL_REQUESTED",
  "agent_id": "approval-agent",
  "message": "Approve database modification: DELETE 100 records?",
  "seq": 12
}
```

#### `APPROVAL_DECISION`
**描述：** 收到人工批准决定
**代理 ID：** 请求批准的代理
**示例消息：** 决定（已批准/已拒绝）及可选评论

```json
{
  "type": "APPROVAL_DECISION",
  "agent_id": "approval-agent",
  "message": "Approved by user@example.com",
  "seq": 13
}
```

---

### HITL 研究审查事件

用于人机回环研究计划审查工作流的事件。当任务以 `review_plan: "manual"` 或 `require_review: true` 提交时，会发出这些事件。

#### `RESEARCH_PLAN_READY`
**描述：** 初始研究计划已生成，等待用户审查
**代理 ID：** `research-planner`
**示例消息：** 生成的研究计划文本
**用例：** 通知前端显示审查 UI

```json
{
  "type": "RESEARCH_PLAN_READY",
  "agent_id": "research-planner",
  "message": "I'll research quantum computing trends focusing on...",
  "seq": 5,
  "payload": {
    "round": 1,
    "version": 1,
    "intent": "ready"
  }
}
```

#### `REVIEW_USER_FEEDBACK`
**描述：** 用户提交了对研究计划的反馈
**代理 ID：** `user`
**示例消息：** 用户的反馈文本
**用例：** 在事件历史中持久化用户反馈

```json
{
  "type": "REVIEW_USER_FEEDBACK",
  "agent_id": "user",
  "message": "Can you focus more on safety implications?",
  "seq": 6,
  "payload": {
    "round": 2,
    "version": 2
  }
}
```

#### `RESEARCH_PLAN_UPDATED`
**描述：** 根据用户反馈更新了研究计划
**代理 ID：** `research-planner`
**示例消息：** 更新后的计划文本
**用例：** 在审查 UI 中显示细化后的计划

```json
{
  "type": "RESEARCH_PLAN_UPDATED",
  "agent_id": "research-planner",
  "message": "Updated plan with focus on AI safety...",
  "seq": 7,
  "payload": {
    "round": 2,
    "version": 3,
    "intent": "ready"
  }
}
```

**负载字段：**
- `round`（整数）：当前审查轮次（1-10）
- `version`（整数）：乐观并发的状态版本
- `intent`（字符串）：LLM 的评估 — `"feedback"` | `"ready"` | `"execute"`

#### `RESEARCH_PLAN_APPROVED`
**描述：** 用户批准了研究计划，开始执行
**代理 ID：** `orchestrator`
**示例消息：** 批准确认
**用例：** 表示从审查过渡到执行

```json
{
  "type": "RESEARCH_PLAN_APPROVED",
  "agent_id": "orchestrator",
  "message": "Research plan approved, starting execution",
  "seq": 8,
  "payload": {
    "approved_by": "user-uuid",
    "final_round": 2
  }
}
```

---

### 高级功能

#### `DEPENDENCY_SATISFIED`
**描述：** 工作流依赖已满足
**代理 ID：** 代理或工作流标识符
**示例消息：** 依赖描述

```json
{
  "type": "DEPENDENCY_SATISFIED",
  "agent_id": "workflow-2",
  "message": "Data preprocessing completed",
  "seq": 6
}
```

#### `ERROR_RECOVERY`
**描述：** 系统从错误中恢复
**代理 ID：** 执行恢复的代理
**示例消息：** 恢复操作描述

```json
{
  "type": "ERROR_RECOVERY",
  "agent_id": "resilient-agent",
  "message": "Retrying with exponential backoff (attempt 2/3)",
  "seq": 11
}
```

---

## 事件过滤

### 按类型过滤

使用 `types` 参数按类型过滤事件：

**SSE：**
```bash
curl -N "http://localhost:8081/stream/sse?workflow_id=task-123&types=AGENT_STARTED,AGENT_COMPLETED"
```

**gRPC：**
```go
request := &pb.StreamRequest{
    WorkflowId: "task-123",
    Types:      []string{"LLM_OUTPUT", "ERROR_OCCURRED"},
}
```

### 常用过滤模式

**聊天 UI（排除流式块）：**
```
types=WORKFLOW_STARTED,AGENT_THINKING,LLM_PROMPT,LLM_OUTPUT,AGENT_COMPLETED,ERROR_OCCURRED
```

**进度跟踪：**
```
types=WORKFLOW_STARTED,PROGRESS,AGENT_COMPLETED,WORKFLOW_COMPLETED
```

**团队协调：**
```
types=TEAM_RECRUITED,TEAM_RETIRED,MESSAGE_SENT,MESSAGE_RECEIVED
```

**HITL 研究审查：**
```
types=RESEARCH_PLAN_READY,REVIEW_USER_FEEDBACK,RESEARCH_PLAN_UPDATED,RESEARCH_PLAN_APPROVED
```

**错误监控：**
```
types=ERROR_OCCURRED,ERROR_RECOVERY
```

### 会话事件 API

会话事件端点（`GET /api/v1/sessions/{sessionId}/events`）自动排除 `LLM_PARTIAL` 事件，以获得更清晰的聊天历史：

```bash
curl "http://localhost:8080/api/v1/sessions/{sessionId}/events?limit=200" \
  -H "X-API-Key: your-key"
```

**为什么排除 LLM_PARTIAL：**
- 减少对话历史中的噪音
- 防止重复内容（部分 + 最终）
- 通过更少的事件提高 UI 性能
- 最终输出始终可通过 `LLM_OUTPUT` 获得

---

## 事件顺序

事件保证：
1. **单调递增** - `seq` 编号始终增加
2. **确定性** - 相同工作流重放产生相同事件
3. **按时间戳排序** - 较早的事件具有较早的时间戳

### 典型事件流

简单查询工作流：
```
1. WORKFLOW_STARTED（orchestrator）
2. DATA_PROCESSING（准备上下文）
3. PROGRESS（规划者创建计划）
4. DELEGATION（移交给代理）
5. AGENT_STARTED（simple-agent）
6. AGENT_THINKING（推理）
7. LLM_PROMPT（发送查询）
8. LLM_PARTIAL（流式传输...）[通常被过滤]
9. LLM_OUTPUT（最终答案）
10. AGENT_COMPLETED（simple-agent）
11. WORKFLOW_COMPLETED（simple-agent）
```

复杂的多代理工作流：
```
1. WORKFLOW_STARTED
2. TEAM_RECRUITED（分析者）
3. TEAM_RECRUITED（撰写者）
4. MESSAGE_SENT（协调员 → 分析者）
5. MESSAGE_RECEIVED（分析者）
6. AGENT_STARTED（分析者）
7. TOOL_INVOKED（database_query）
8. TOOL_OBSERVATION（结果）
9. LLM_OUTPUT（分析）
10. MESSAGE_SENT（分析者 → 撰写者）
11. AGENT_STARTED（撰写者）
12. LLM_OUTPUT（最终文档）
13. WORKFLOW_COMPLETED
```

HITL 研究审查工作流：
```
1. WORKFLOW_STARTED（orchestrator）
2. DATA_PROCESSING（准备上下文）
3. RESEARCH_PLAN_READY（research-planner）← 工作流在此暂停
   [用户在 UI 中审查计划]
4. REVIEW_USER_FEEDBACK（user）← 用户提供反馈
5. RESEARCH_PLAN_UPDATED（research-planner）← 细化后的计划
   [用户可以提供更多反馈或批准]
6. RESEARCH_PLAN_APPROVED（orchestrator）← 用户批准
7. DELEGATION（启动研究代理）
8. AGENT_STARTED（research-agent-1）
... [正常研究工作流继续]
N. WORKFLOW_COMPLETED
```

---

## 实现说明

### 事件发出

事件通过 Temporal 活动发出以确保确定性：

```go
EmitTaskUpdate(ctx, EmitTaskUpdateInput{
    WorkflowID: workflowID,
    EventType:  StreamEventAgentStarted,
    AgentID:    "simple-agent",
    Message:    "Processing query",
    Timestamp:  time.Now(),
})
```

### 版本门控

某些事件类型需要启用功能门控：

| 事件类型 | 所需功能门控 |
|------------|----------------------|
| `MESSAGE_SENT`、`MESSAGE_RECEIVED`、`WORKSPACE_UPDATED` | `p2p_v1` |
| `TEAM_RECRUITED`、`TEAM_RETIRED` | `dynamic_team_v1` |

### 存储与保留

- 事件存储在 PostgreSQL 的 `event_logs` 表中
- 实时流式传输使用内存环缓冲区（默认：256 个事件）
- 历史事件可通过 REST API 查询
- 实时流式传输使用有界 Redis Streams；容量（默认每个工作流约 256 项）可通过 `STREAMING_RING_CAPACITY` 或编程方式 `streaming.Configure(n)` 配置

---

## API 端点

### 流式传输实时事件

**SSE：**
```bash
GET /stream/sse?workflow_id={id}&types={csv}&last_event_id={id-or-seq}
```

**WebSocket：**
```bash
GET /stream/ws?workflow_id={id}&types={csv}&last_event_id={id-or-seq}
```

注意：`last_event_id` 接受 Redis 流 ID（例如 `1700000000000-0`）或数字序列。当使用数字值时，重放包含 `seq > last_event_id` 的事件。

**gRPC：**
```protobuf
rpc StreamTaskExecution(StreamRequest) returns (stream TaskUpdate);
```

### 查询历史事件

**按任务查询事件：**
```bash
GET /api/v1/tasks/{id}/events?limit=50&offset=0
```

**会话事件（排除 LLM_PARTIAL）：**
```bash
GET /api/v1/sessions/{sessionId}/events?limit=200&offset=0
```

注意：如果会话已被软删除，返回 404。

---

## 最佳实践

### 对于前端开发者

1. **适当过滤** - 如果只需要子集，不要订阅所有事件
2. **处理重新连接** - 使用 `last_event_id` 从断开连接处恢复
3. **聊天 UI 排除 LLM_PARTIAL** - 使用会话事件 API 或手动过滤
4. **显示进度事件** - 使用 `PROGRESS` 和 `DATA_PROCESSING` 进行状态更新
5. **按 workflow_id 分组** - 多个工作流可能并发发出事件

### 对于后端开发者

1. **始终通过活动发出** - 确保确定性和 Temporal 重放安全
2. **包含有意义的消息** - 帮助前端显示有用信息
3. **使用适当的事件类型** - 不要使通用类型过载
4. **考虑功能门控** - 检查高级事件类型是否已启用
5. **清理敏感数据** - 尤其是在 `LLM_PROMPT` 事件中

### 对于运维人员

1. **监控 ERROR_OCCURRED 事件** - 设置告警
2. **跟踪 ERROR_RECOVERY 模式** - 识别可靠性问题
3. **调整环缓冲区大小** - 基于工作流持续时间和事件量
4. **使用事件过滤** - 减少分析和存储的带宽

---

## 相关文档

- **流式 API**：`/docs/streaming-api.md` - SSE、WebSocket、gRPC 协议
- **会话 API**：OpenAPI 规范位于 `/openapi.json` - REST 端点
- **事件类型源码**：`go/orchestrator/internal/activities/stream_events.go`

---

## 快速参考

| 分类 | 事件类型 | 数量 |
|----------|-------------|-------|
| **核心工作流** | WORKFLOW_STARTED、WORKFLOW_COMPLETED、AGENT_STARTED、AGENT_COMPLETED、ERROR_OCCURRED、AGENT_THINKING | 6 |
| **LLM 事件** | LLM_PROMPT、LLM_PARTIAL、LLM_OUTPUT、TOOL_OBSERVATION | 4 |
| **多代理** | DELEGATION、MESSAGE_SENT、MESSAGE_RECEIVED、TEAM_RECRUITED、TEAM_RETIRED、ROLE_ASSIGNED | 6 |
| **进度/状态** | PROGRESS、DATA_PROCESSING、TEAM_STATUS、WAITING、WORKSPACE_UPDATED | 5 |
| **工具** | TOOL_INVOKED | 1 |
| **人机交互** | APPROVAL_REQUESTED、APPROVAL_DECISION | 2 |
| **HITL 研究审查** | RESEARCH_PLAN_READY、REVIEW_USER_FEEDBACK、RESEARCH_PLAN_UPDATED、RESEARCH_PLAN_APPROVED | 4 |
| **高级** | DEPENDENCY_SATISFIED、ERROR_RECOVERY | 2 |
| **总计** | | **30** |

---
