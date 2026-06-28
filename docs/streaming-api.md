# Shannon 流式 API

本文档描述编排器暴露的最小化、确定性流式接口。涵盖 gRPC、Server-Sent Events (SSE) 和 WebSocket (WS) 端点，包括过滤器和用于重新加入会话的恢复语义。

## 事件持久化策略

Shannon 使用**两层持久化模型**，针对性能进行了优化：

### Redis（所有事件）
- **目的**：实时 SSE/WebSocket 交付
- **保留期**：每个工作流最近 256 个事件，24 小时 TTL
- **事件**：所有事件，包括 `LLM_PARTIAL`（流式令牌）
- **存储**：每个 500 令牌响应约 30-50KB

### PostgreSQL（仅重要事件）
- **目的**：历史审计追踪和重放
- **保留期**：永久（或按保留策略）
- **事件**：仅持久化关键事件：
  - ✅ `WORKFLOW_COMPLETED`、`WORKFLOW_FAILED`
  - ✅ `AGENT_COMPLETED`、`AGENT_FAILED`
  - ✅ `TOOL_INVOKED`、`TOOL_OBSERVATION`、`TOOL_ERROR`
  - ✅ `ERROR_OCCURRED`、`LLM_OUTPUT`、`STREAM_END`
  - ❌ `LLM_PARTIAL`（thread.message.delta）- **不持久化**
  - ❌ `HEARTBEAT`、`PING` - **不持久化**
- **减少量**：数据库写入减少约 95%（500 个增量 → 5-10 个重要事件）

**理由**：流式增量是临时性的，仅用于实时交付。Redis 为调试提供足够的保留期（24 小时），而 PostgreSQL 存储永久审计追踪。

---

## 事件模型

- 字段：`workflow_id`、`type`、`agent_id?`、`message?`、`timestamp`、`seq`。
- 最小事件类型（在 `streaming_v1` 门控之后）：
  - `WORKFLOW_STARTED`、`AGENT_STARTED`、`AGENT_COMPLETED`、`ERROR_OCCURRED`。
  - P2P v1 添加：`MESSAGE_SENT`、`MESSAGE_RECEIVED`、`WORKSPACE_UPDATED`。
- 确定性：事件从工作流作为活动发出，记录在 Temporal 历史中，并发布到本地流管理器。

### 事件中的附件引用

当任务包含文件附件时，`WORKFLOW_STARTED` 在 `payload.task_context.attachments` 中携带附件元数据：

```json
{
  "type": "WORKFLOW_STARTED",
  "payload": {
    "task_context": {
      "attachments": [
        {"id": "abc123", "media_type": "image/png", "filename": "chart.png", "size_bytes": 14797},
        {"id": "def456", "media_type": "application/pdf", "filename": "report.pdf", "size_bytes": 1422}
      ]
    }
  }
}
```

附件 `id` 引用存储在 Redis 中的数据（TTL 30 分钟）。实际的二进制内容**不包含**在 SSE 事件中——仅包含轻量级元数据。

### 增强事件类型

**LLM 响应事件：**
- `LLM_PARTIAL`：在代理执行期间发出，包含流式文本增量（逐令牌或小批量）。
  - 流式文本：`message` 包含文本增量。
  - SSE 映射：以 `event: thread.message.delta` 发送，数据为：
    ```json
    {
      "delta": "部分文本...",
      "workflow_id": "task-...",
      "agent_id": "simple-agent",
      "seq": 9,
      "stream_id": "1700000000000-0"
    }
    ```
- `LLM_OUTPUT`：在 LLM 调用完成时发出，包含最终响应和使用量元数据。
  - 最终文本：`message` 包含最终响应文本（为安全起见截断）。
  - 使用量元数据：可用时在 `payload` 字段中的 JSON 结构：
    ```json
    {
      "tokens_used": 174,
      "input_tokens": 104,
      "output_tokens": 70,
      "cost_usd": 0.0012,
      "model_used": "gpt-5-nano-2025-08-07",
      "provider": "openai"
    }
    ```
  - SSE 映射：以 `event: thread.message.completed` 发送，数据为：
    ```json
    {
      "response": "5 + 5 等于 10。",
      "workflow_id": "task-...",
      "agent_id": "simple-agent",
      "seq": 10,
      "stream_id": "1700000000001-0",
      "metadata": {
        "tokens_used": 174,
        "input_tokens": 104,
        "output_tokens": 70,
        "cost_usd": 0.0012,
        "model_used": "gpt-5-nano-2025-08-07",
        "provider": "openai"
      }
    }
    ```

**工具执行事件：**
- `TOOL_INVOKED`：当 agent-core 执行工具时发出（web_search、calculator、file_read 等）
  - `message` 是人类可读的描述（例如 `"Calling web_search with query: 'latest news'"`）
  - `payload` 包含结构化数据：
    ```json
    {
      "tool": "web_search",
      "params": {
        "query": "latest news"
      }
    }
    ```
- `TOOL_OBSERVATION`：当工具执行完成并返回结果时发出
  - `message` 包含截断的工具输出或简短的错误描述（UTF-8 安全，最多约 2000 字符）
  - `payload` 包含元数据，例如：
    ```json
    {
      "tool": "web_search",
      "success": true,
      "duration_ms": 1234
    }
    ```

**注意**：仅 Python 工具（供应商适配器、自定义集成）使用内部函数调用，按设计不发出 `TOOL_INVOKED`/`TOOL_OBSERVATION` 事件。结果嵌入在 LLM 响应文本中。

## gRPC：StreamingService

- RPC：`StreamingService.StreamTaskExecution(StreamRequest) returns (stream TaskUpdate)`
- 请求字段：
  - `workflow_id`（必需）
  - `types[]`（可选）— 按事件类型过滤
  - `last_event_id`（可选）— 恢复点；接受 Redis `stream_id`（首选）或数字 `seq`。当使用数字值时，重放包含 `seq > last_event_id` 的事件。
- 响应：`TaskUpdate` 镜像事件模型。

示例（伪 Go）：

```go
client := pb.NewStreamingServiceClient(conn)
stream, _ := client.StreamTaskExecution(ctx, &pb.StreamRequest{
    WorkflowId: wfID,
    Types:      []string{"AGENT_STARTED", "AGENT_COMPLETED"},
    LastEventId: 42,
})
for {
    upd, err := stream.Recv()
    if err != nil { break }
    fmt.Println(upd.Type, upd.AgentId, upd.Seq)
}
```

## SSE：HTTP `/stream/sse`

- 方法：`GET /stream/sse?workflow_id=<id>&types=<csv>&last_event_id=<id-or-seq>`
- 头部：支持 `Last-Event-ID` 用于浏览器自动恢复。
- CORS：`Access-Control-Allow-Origin: *`（开发友好；生产环境中前端网关应强制执行认证）。

示例（curl）：

```bash
# 观看代理生命周期事件
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF&types=AGENT_STARTED,AGENT_COMPLETED"

# 观看 LLM 输出和使用量元数据
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF&types=LLM_OUTPUT"

# 观看工具执行
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF&types=TOOL_INVOKED,TOOL_OBSERVATION"

# 观看所有事件
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF"
```

注意：
- 服务器在可用时发出 Redis `stream_id` 作为 `id`（首选），否则回退到数字 `seq`。您可以使用 `Last-Event-ID` 头部或 `last_event_id` 查询参数以任一形式重新连接。
- 心跳大约每 10 秒作为 SSE 注释发送，以保持中间代理存活。
- `LLM_PARTIAL` 事件映射为 `thread.message.delta` SSE 事件，带有 `delta` 字段用于流式文本。
- `LLM_OUTPUT` 事件包含 `message` 中的最终响应文本和 `payload` 中的使用量元数据；SSE 处理器将其映射为 `thread.message.completed` 事件，包含 `response`（文本）和可选的 `metadata`。

## WebSocket：HTTP `/stream/ws`

- 方法：`GET /stream/ws?workflow_id=<id>&types=<csv>&last_event_id=<id-or-seq>`
- 消息：JSON 对象，匹配事件模型。
- 心跳：服务器大约每 20 秒 ping 一次；客户端应回复 pong。

示例（JS）：

```js
const ws = new WebSocket(`ws://localhost:8081/stream/ws?workflow_id=${wf}`);
ws.onmessage = (e) => {
  const evt = JSON.parse(e.data); // {workflow_id,type,agent_id,message,timestamp,seq}
};
```

## 无效工作流检测

gRPC 和 SSE 流式端点均自动验证工作流是否存在，以便对无效工作流 ID 快速失败：

### 行为

- **验证超时**：从连接开始 30 秒
- **验证方法**：使用 Temporal `DescribeWorkflowExecution` API
- **首个事件定时器**：如果 30 秒内未收到事件则触发

### 按传输方式的响应

**gRPC（`StreamingService.StreamTaskExecution`）**
- 返回 `NotFound` gRPC 错误码
- 错误信息：`"workflow not found"` 或 `"workflow not found or unavailable"`

**SSE（`/stream/sse`）**
- 在关闭前发出 `ERROR_OCCURRED` 事件：
  ```
  event: ERROR_OCCURRED
  data: {"workflow_id":"xxx","type":"ERROR_OCCURRED","message":"Workflow not found"}
  ```
- 在等待期间每 10 秒包含心跳 ping（`: ping`）

**WebSocket（`/stream/ws`）**
- 与 SSE 行为相同，发送 JSON 错误事件后关闭连接

### 有效工作流边缘情况

- **工作流存在但 30 秒内无事件产生**：流保持打开，定时器重置
- **验证期间 Temporal 不可用**：立即返回错误
- **有效工作流**：首个事件到达后禁用定时器

### 使用示例

```bash
# 无效工作流 - 约 30 秒后返回错误
shannon stream "invalid-workflow-123"
# 30 秒后输出：
# ERROR_OCCURRED: Workflow not found

# 有效工作流 - 正常流式传输
shannon stream "task-user-1234567890"
# 输出：立即流式传输事件
```

### 注意

- 这防止了流式传输不存在的工作流时无限挂起
- 30 秒超时在响应性和允许工作流缓慢启动之间取得平衡
- 心跳在验证期间通过代理保持连接存活

### Gateway 行为

- HTTP gateway 将流式过滤器转发给编排器，并接受任何事件类型。未知类型不会产生事件。
- `last_event_id` 接受数字序列和 Redis 流 ID（例如 `1700000000000-0`）。

## 动态团队（信号）+ 团队事件

当 `SupervisorWorkflow` 中启用 `dynamic_team_v1` 时，工作流接受信号：

- 招募：信号名称 `recruit_v1`，JSON `{ "Description": string, "Role"?: string }`。
- 退休：信号名称 `retire_v1`，JSON `{ "AgentID": string }`。

授权操作发出流式事件：

- `TEAM_RECRUITED`，`agent_id` 为角色（用于最小 v1），`message` 为描述。
- `TEAM_RETIRED`，`agent_id` 为退休的代理。

在 docker compose 内通过 Temporal CLI 发送信号的辅助脚本：

```bash
# 为子任务招募新 worker
./scripts/signal_team.sh recruit <WORKFLOW_ID> "总结第 3 节" writer

# 退休 worker
./scripts/signal_team.sh retire <WORKFLOW_ID> agent-xyz
```

提示：使用 SSE/WS 过滤器仅观看团队事件：

```bash
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF&types=TEAM_RECRUITED,TEAM_RETIRED"
```

## Swarm 工作流事件

当 `force_swarm: true` 触发 SwarmWorkflow 时，为实时任务板 UI 发出额外事件类型：

| 事件类型 | 代理 ID | 负载 | 时机 |
|-----------|----------|---------|------|
| `AGENT_STARTED` | `{agent-name}` | `{role: "researcher"}` | 代理以角色生成 |
| `AGENT_COMPLETED` | `{agent-name}` | — | 代理完成 |
| `TASKLIST_UPDATED` | `tasklist` | `{tasks: SwarmTask[]}` | 任务列表变更 |
| `LEAD_DECISION` | `swarm-lead` | `{event_type, actions_count}` | Lead 协调 |

### TASKLIST_UPDATED 负载

`TASKLIST_UPDATED` 事件在其负载中携带完整任务列表，使前端能够渲染实时任务板：

```json
{
  "type": "TASKLIST_UPDATED",
  "agent_id": "tasklist",
  "message": "task=T1 status=in_progress",
  "payload": {
    "tasks": [
      {
        "id": "T1",
        "description": "研究美国 AI 芯片市场",
        "status": "in_progress",
        "owner": "takao",
        "created_by": "decompose",
        "depends_on": [],
        "created_at": "2026-02-26T10:00:00Z"
      },
      {
        "id": "T2",
        "description": "分析比较数据",
        "status": "pending",
        "owner": "",
        "depends_on": ["T1"],
        "created_at": "2026-02-26T10:00:00Z"
      }
    ]
  }
}
```

### 带角色的 AGENT_STARTED

Swarm `AGENT_STARTED` 事件在负载中包含代理的角色，这在非 swarm 工作流中不存在：

```json
{
  "type": "AGENT_STARTED",
  "agent_id": "takao",
  "message": "Agent takao started",
  "payload": {
    "role": "researcher"
  }
}
```

### 流式传输 Swarm 事件

```bash
# 观看 swarm 任务板和代理生命周期
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF&types=TASKLIST_UPDATED,AGENT_STARTED,AGENT_COMPLETED,LEAD_DECISION"

# 观看所有事件，包括工具执行
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF"
```

### HITL：执行中的人工输入

用户可以向运行中的 swarm 发送消息，这些消息以 `human_input` 事件到达 Lead 的决策循环：

```bash
POST /api/v1/swarm/{workflowID}/message
Content-Type: application/json

{"message": "更多关注三星的代工策略"}
```

Lead 在下一个决策周期中纳入反馈。

## HITL 研究审查事件

当设置 `review_plan: "manual"` 或 `require_review: true` 时，工作流为研究计划审查发出 HITL 特定事件。

### 事件类型

| 事件 | 描述 | 由谁发出 |
|-------|-------------|------------|
| `RESEARCH_PLAN_READY` | 初始计划已生成，等待审查 | Orchestrator |
| `REVIEW_USER_FEEDBACK` | 用户提交了反馈 | Gateway（审查 API） |
| `RESEARCH_PLAN_UPDATED` | 根据反馈细化计划 | Gateway（审查 API） |
| `RESEARCH_PLAN_APPROVED` | 用户批准，开始执行 | Orchestrator |

### 流式传输 HITL 事件

```bash
# 观看所有 HITL 审查事件
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF&types=RESEARCH_PLAN_READY,REVIEW_USER_FEEDBACK,RESEARCH_PLAN_UPDATED,RESEARCH_PLAN_APPROVED"

# 最小化：仅计划就绪和批准（用于进度跟踪）
curl -N "http://localhost:8081/stream/sse?workflow_id=$WF&types=RESEARCH_PLAN_READY,RESEARCH_PLAN_APPROVED"
```

### HITL 事件负载

HITL 事件包含 `payload` 字段，带有审查元数据：

```json
{
  "type": "RESEARCH_PLAN_UPDATED",
  "agent_id": "research-planner",
  "message": "更新后的计划聚焦于安全性...",
  "payload": {
    "round": 2,
    "version": 3,
    "intent": "ready"
  }
}
```

**负载字段：**
- `round`：当前审查轮次（1-10，最多 10）
- `version`：乐观并发的状态版本（与 `If-Match` 头部一起使用）
- `intent`：LLM 的评估 — `"feedback"`（提出问题）、`"ready"`（计划就绪）、`"execute"`（用户已批准）

### 前端集成示例

```jsx
function HITLReviewStream({ workflowId }) {
  const [reviewState, setReviewState] = useState({ status: 'waiting', plan: null });

  useEffect(() => {
    const types = 'RESEARCH_PLAN_READY,REVIEW_USER_FEEDBACK,RESEARCH_PLAN_UPDATED,RESEARCH_PLAN_APPROVED';
    const es = new EventSource(`/stream/sse?workflow_id=${workflowId}&types=${types}`);

    es.onmessage = (e) => {
      const event = JSON.parse(e.data);

      switch (event.type) {
        case 'RESEARCH_PLAN_READY':
          setReviewState({ status: 'reviewing', plan: event.message, ...event.payload });
          break;
        case 'RESEARCH_PLAN_UPDATED':
          setReviewState(prev => ({ ...prev, plan: event.message, ...event.payload }));
          break;
        case 'RESEARCH_PLAN_APPROVED':
          setReviewState(prev => ({ ...prev, status: 'approved' }));
          break;
      }
    };

    return () => es.close();
  }, [workflowId]);

  return (
    <div>
      <p>状态：{reviewState.status}</p>
      {reviewState.plan && <pre>{reviewState.plan}</pre>}
      {reviewState.intent === 'ready' && (
        <button onClick={() => approveReview(workflowId)}>批准计划</button>
      )}
    </div>
  );
}
```

### HITL 工作流时间线

```
时间    事件                     描述
─────────────────────────────────────────────────────────
0s      WORKFLOW_STARTED          任务已提交
1s      DATA_PROCESSING           准备上下文
3s      RESEARCH_PLAN_READY       计划已生成，UI 显示审查
        ─── 工作流暂停，等待用户 ───
45s     REVIEW_USER_FEEDBACK      用户："关注 X"
47s     RESEARCH_PLAN_UPDATED     细化后的计划（intent: ready）
60s     RESEARCH_PLAN_APPROVED    用户点击批准
        ─── 工作流恢复 ───
62s     DELEGATION                启动研究代理
...     [正常研究事件]
300s    WORKFLOW_COMPLETED        研究完成
```

## 快速开始

### 开发测试
```bash
# 启动 Shannon 服务
make dev

# 测试特定工作流的流式传输
make smoke-stream WF_ID=<workflow_id>

# 可选：自定义端点
make smoke-stream WF_ID=workflow-123 ADMIN=http://localhost:8081 GRPC=localhost:50052
```

<!-- 浏览器演示部分已移除：文件不再包含 -->

## 容量与重放行为

- 实时流式传输使用 Redis Streams，每个工作流有界长度（近似 maxlen ~256），用于确定性重放。
- 容量可通过环境变量或代码调整：
  - `STREAMING_RING_CAPACITY`（整数）— 编排器读取并在启动时调用 `streaming.Configure(n)`
  - 编程方式：在初始化期间调用 `streaming.Configure(n)`

## 运维注意事项

- 重放安全：事件发出经过版本门控并通过活动路由，保持 Temporal 确定性。
- 背压：向慢订阅者丢弃事件（非阻塞通道）；客户端应视需要使用 `last_event_id` 重新连接。
- 安全：在生产环境中，在管理 HTTP 端口前放置经过认证的代理；gRPC 在外部暴露时应要求 TLS。

### 反模式与负载考虑
- 避免每个客户端无界缓冲区。进程内管理器使用有界通道和固定环来防止内存增长。
- 不要依赖每个事件都交付给慢客户端。相反，使用 `last_event_id` 重新连接以确定性方式追赶。
- 对于简单的仪表板和日志，优先使用 SSE；仅在需要双向控制消息时使用 WebSocket。
- 对于高扇出，在外部放置事件网关（例如 NGINX 或轻量级 Go 扇出）；进程内管理器不是消息代理。

## 架构

### 事件流
```
工作流 → EmitTaskUpdate（活动）→ 流管理器 → 环缓冲区 + 实时订阅者
                                                           ↓
                        SSE ← HTTP 网关 ← 事件分发 → gRPC 流
                         ↓                                       ↓
                    WebSocket ←────────────────────────────── 客户端 SDK
```

### 关键组件
- **流管理器**：内存中的发布/订阅，每个工作流有环缓冲区
- **环缓冲区**：可配置容量（默认：256 个事件），用于重放支持
- **多协议**：gRPC（企业级）、SSE（浏览器原生）、WebSocket（交互式）
- **确定性事件**：所有事件通过 Temporal 活动路由，保证重放安全

### 服务端口
- **管理 HTTP**：8081（SSE `/stream/sse`、WebSocket `/stream/ws`、健康检查、审批）
- **gRPC**：50052（StreamingService、OrchestratorService、SessionService）

## 集成示例

### Python SDK（伪代码）
```python
import grpc
from shannon.pb import orchestrator_pb2, orchestrator_pb2_grpc

# gRPC 流式传输
channel = grpc.insecure_channel('localhost:50052')
client = orchestrator_pb2_grpc.StreamingServiceStub(channel)
request = orchestrator_pb2.StreamRequest(
    workflow_id='workflow-123',
    types=['AGENT_STARTED', 'AGENT_COMPLETED'],
    last_event_id=0
)

for update in client.StreamTaskExecution(request):
    print(f"Agent {update.agent_id}: {update.type} (seq: {update.seq})")
```

### React 组件
```jsx
import React, { useEffect, useState } from 'react';

function WorkflowStream({ workflowId }) {
  const [events, setEvents] = useState([]);
  const [llmOutput, setLlmOutput] = useState('');
  const [usage, setUsage] = useState(null);

  useEffect(() => {
    const eventSource = new EventSource(
      `/stream/sse?workflow_id=${workflowId}&types=LLM_OUTPUT,AGENT_COMPLETED,TOOL_OBSERVATION`
    );

    eventSource.onmessage = (e) => {
      const event = JSON.parse(e.data);

      if (event.type === 'LLM_OUTPUT') {
        // 检查 message 是使用量元数据（JSON）还是文本块（字符串）
        try {
          const parsed = JSON.parse(event.message);
          if (parsed.usage) {
            // 使用量元数据
            setUsage(parsed);
          }
        } catch {
          // 文本块
          setLlmOutput(prev => prev + event.message);
        }
      }

      setEvents(prev => [...prev, event]);
    };

    return () => eventSource.close();
  }, [workflowId]);

  return (
    <div>
      <div className="llm-output">
        <pre>{llmOutput}</pre>
        {usage && (
          <div className="usage-stats">
            模型：{usage.model}（{usage.provider}）<br/>
            令牌：{usage.usage.total_tokens}
            （入：{usage.usage.input_tokens}，出：{usage.usage.output_tokens}）
          </div>
        )}
      </div>

      <div className="events">
        {events.map(event => (
          <div key={event.seq}>
            {event.type}：{event.agent_id || event.message?.substring(0, 50)}
          </div>
        ))}
      </div>
    </div>
  );
}
```

## 故障排查

### 常见问题

**"未收到事件"**
- 验证 workflow_id 存在且正在运行
- 检查工作流中是否启用了 `streaming_v1` 版本门控
- 确保管理 HTTP 端口（8081）可访问
- 如果使用 `types`，请确保工作流确实发出这些事件类型

**"重新连接后事件丢失"**
- 使用 `last_event_id` 参数或 `Last-Event-ID` 头部
- 重放从有界的 Redis Stream 读取；非常旧的事件可能在流（约 256 项）淘汰它们后被修剪

**Python async：在 async-for 内部 await 时出现 RuntimeError**
- 不要在流的 `async for` 内部 await 其他客户端调用。先跳出循环，然后再 `await`：

```python
async with AsyncShannonClient() as client:
    h = await client.submit_task("复杂分析")
    async for e in client.stream(h.workflow_id, types=["LLM_OUTPUT","WORKFLOW_COMPLETED"]):
        if e.type == "WORKFLOW_COMPLETED":
            break
    final = await client.wait(h.task_id)
    print(final.result)
```

**"内存使用量高"**
- 减少配置中的环缓冲区容量
- 实施客户端过滤以减少事件量
- 对多个并发流使用连接池

### 调试命令
```bash
# 检查流式端点
curl -s http://localhost:8081/health
curl -N "http://localhost:8081/stream/sse?workflow_id=test" | head -10

# 测试 gRPC 连接
grpcurl -plaintext localhost:50052 list shannon.orchestrator.StreamingService

# 监控流式日志
docker compose logs orchestrator | grep "stream"
```

## 提供商兼容性

### 使用量元数据流式传输支持

Shannon 的 LLM 提供商在 `LLM_OUTPUT` 流式事件中发出使用量元数据，支持程度不同：

| 提供商 | 流式传输 | 使用量元数据 | 实现 |
|----------|-----------|----------------|----------------|
| **OpenAI** | ✅ | ✅ | 使用 `stream_options: {"include_usage": true}` |
| **Anthropic** | ✅ | ✅ | 流式传输后使用 `await stream.get_final_message()` |
| **Groq** | ✅ | ✅ | 使用量在最终流式块中可用 |
| **Google** | ✅ | ✅ | 使用量通过最终块中的 `usage_metadata` 可用 |
| **xAI** | ✅ | ⚠️ API 限制 | REST API 在流式模式下不发出使用量 |
| **OpenAI 兼容** | ✅ | ⚠️ 视情况而定 | 取决于端点；部分不支持 `stream_options` |

### 已知限制

**xAI 流式传输**
- xAI 的 REST API（`api.x.ai/v1` 的 OpenAI 兼容端点）在流式传输期间**不**发出使用量元数据
- 使用量仅在非流式响应中可用
- 这是 xAI API 的限制，而非 Shannon 实现的问题
- 替代方案：如果使用量元数据是必需的，对 xAI 使用非流式模式
- 原生 xAI gRPC SDK（`xai-sdk` 包）行为不同，但需要完全重写提供商

**OpenAI GPT-5 模型**
- GPT-5-nano 和 GPT-5-mini 模型不流式传输文本增量（OpenAI API 错误）
- 仅流式传输使用量元数据；无内容块
- 影响模型：`gpt-5-nano-2025-08-07`、`gpt-5-mini-2025-08-07`
- 变通方案：使用 GPT-4o 模型获取流式文本 + 使用量

**OpenAI 兼容端点**
- 部分 OpenAI 兼容端点（DeepSeek、Qwen、本地模型）可能不支持 `stream_options` 参数
- Shannon 优雅地处理此情况 - 流式传输将在没有使用量元数据的情况下工作
- 使用量元数据仍可在最终任务完成响应中获取

**仅 Python 工具**
- 供应商特定工具（GA4、自定义适配器）通过内部函数调用执行
- 按架构设计，不发出 `TOOL_INVOKED`/`TOOL_OBSERVATION` 事件
- 结果嵌入在 LLM 响应文本中
- 这避免了 Python ↔ Go ↔ Rust 之间复杂的跨语言参数映射

### 实现细节

**回退流式传输**
- 当主要提供商失败时，Shannon 自动回退到备用提供商
- 使用量元数据块现在通过回退路径保留
- 修复应用：`manager.py:737-738` 接受 `(str, dict)` 而非仅 `str`

**UTF-8 安全**
- 工具观察消息使用 UTF-8 安全截断截断至 2000 字符
- 使用基于 rune 的截断（Go 的 `truncateQuery()`）防止分割多字节字符
- 修复应用：`agent.go:1346-1347`

**使用量元数据结构**
所有支持使用量元数据的提供商发出一致的结构：
```json
{
  "usage": {
    "total_tokens": 174,
    "input_tokens": 104,
    "output_tokens": 70
  },
  "model": "gpt-5-nano-2025-08-07",
  "provider": "openai"
}
```

此元数据也被聚合，并可在以下位置获取：
- 最终任务完成 API 响应（`GET /api/v1/tasks/{id}`）
- 任务执行数据库记录
- Temporal 工作流元数据

## 路线图

### 阶段 1（已完成）
- ✅ 最小事件类型：WORKFLOW_STARTED、AGENT_STARTED、AGENT_COMPLETED、ERROR_OCCURRED
- ✅ 扩展事件类型：LLM_OUTPUT、TOOL_INVOKED、TOOL_OBSERVATION
- ✅ 为 OpenAI、Anthropic、Groq、Google 提供商提供使用量元数据流式传输
- ✅ 三种协议：gRPC、SSE、WebSocket
- ✅ 使用有界 Redis Streams 的重放支持
- ✅ UTF-8 安全消息截断
- ✅ 回退流式传输保留使用量元数据
- ✅ 选择性 PostgreSQL 持久化（过滤临时事件）

### 阶段 2A：多代理协调（已完成）
利用现有基础设施的快速胜利：
- ✅ **ROLE_ASSIGNED**：当基于角色的代理被激活时发出（例如 `analysis`、`browser_use`）
  - 负载：角色名称、可用工具、工具数量
  - 位置：`orchestrator_router.go:205-217`
- ✅ **DELEGATION**：当编排器将子任务委托给代理时发出
  - 已在 `orchestrator_router.go` 中实现
- ✅ **BUDGET_THRESHOLD**：当令牌预算超过警告阈值时发出
  - 负载：usage_percent、tokens_used、budget_type（task/session）
  - 位置：`budget/manager.go:258-274`

**数据库持久化**：所有阶段 2A 事件持久化到 PostgreSQL（通过 `shouldPersistEvent` 过滤器）

### 阶段 2B：高级多代理（已规划）
需要新系统实现：
- ⏳ **AGENT_MESSAGE_SENT**：代理间通信（需要 `mailbox_v1` 系统）
- ⏳ **POLICY_EVALUATED**：策略框架事件（OPA 集成存在，需要流式传输）
- ⏳ **WASI_SANDBOX_EVENT**：Python 代码执行事件（需要 Rust→Go 事件桥接）

### 阶段 3（未来）
- 单个连接中多个工作流的 WebSocket 多路复用
- Python/TypeScript 的 SDK 辅助工具，便于使用
- 实时仪表板组件和可视化工具
