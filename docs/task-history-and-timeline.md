# 任务历史与时间线 API

本文档说明 Shannon 的任务历史 API、持久化事件存储以及从 Temporal 历史派生的确定性时间线。

## 概述

- 实时事件：用于进行中任务的 Server-Sent Events（SSE）。
- 持久化事件：存储在 Postgres（`event_logs`）中的人类可读应用事件。
- 确定性时间线：从 Temporal 的规范工作流历史构建的简洁、面向人类的摘要，可选择持久化到 `event_logs`。

## 数据存储

- Temporal 历史：由 Temporal 存储在其自身数据库中（不归 Shannon 所有）。
- Redis Streams：用于 SSE 的实时流缓冲区（约 24 小时 TTL）。
- Postgres `event_logs`：客户端和 API 使用的长期审计/事件存储。

## HTTP 端点（Gateway）

- `GET /api/v1/tasks` — 列出任务
  - 查询参数：`limit`、`offset`、`status`（QUEUED|RUNNING|COMPLETED|FAILED|CANCELLED|TIMEOUT）、`session_id`
  - 响应：`{ tasks: TaskSummary[], total_count }`
- `GET /api/v1/tasks/{id}` — 任务状态
  - 包含：`query`、`session_id`、`mode` 用于重放/续接
  - 现在也包含由工作流填充并持久化到数据库的使用量元数据：
    - `model_used`（字符串）、`provider`（字符串）
    - `usage`（对象）：`{ total_tokens, input_tokens?, output_tokens?, estimated_cost? }`

响应示例（结构）：

```json
{
  "task_id": "task-...",
  "status": "TASK_STATUS_COMPLETED",
  "result": "...",
  "model_used": "gpt-5-mini-2025-08-07",
  "provider": "openai",
  "usage": {
    "total_tokens": 300,
    "input_tokens": 200,
    "output_tokens": 100,
    "estimated_cost": 0.006
  }
}
```
- `GET /api/v1/tasks/{id}/events` — 持久化事件历史（来自 `event_logs`）
  - 查询参数：`limit`、`offset`
  - 响应：`{ events: [...], count }`
- `GET /api/v1/tasks/{id}/timeline` — 确定性重放（Temporal → 时间线）
  - 查询参数：
    - `run_id` — 可选的运行 ID
    - `mode` — `summary`（默认）或 `full`
    - `include_payloads` — 默认 `false`
    - `persist` — 默认 `true`；为 true 时异步持久化并返回 202
  - 响应：
    - 202 Accepted：`{ status: "accepted", workflow_id, count }`（异步持久化中）
    - 200 OK：`{ workflow_id, events, stats }`（仅预览）

除非设置 `GATEWAY_SKIP_AUTH=1`，否则所有端点都需要身份验证。

## SSE vs 持久化事件 vs 时间线

- SSE（实时）：
  - `GET /api/v1/stream/sse?workflow_id=...`
  - 临时性，通过 `Last-Event-ID` 支持恢复，超出 Redis TTL 后不存储。

- 持久化事件（`event_logs`）：
  - 应用级别可读事件（例如 TOOL_INVOKED、AGENT_THINKING）在执行期间异步写入。
  - 通过 `GET /api/v1/tasks/{id}/events` 检索。

- 确定性时间线：
  - 从 Temporal 历史派生，确保即使在流中断时也能完整。
  - 按需构建；当 `persist=true` 时异步持久化。
  - 事件类型前缀用于追溯来源：`WF_`、`ACT_`、`CHILD_`、`SIG_`、`TIMER_`、`ATTR_`、`MARKER_`。

## 示例

```bash
# 列出最近的任务
curl -H "X-API-Key: $API_KEY" \
  "http://localhost:8080/api/v1/tasks?limit=20&offset=0&status=COMPLETED"

# 获取任务的持久化事件
curl -H "X-API-Key: $API_KEY" \
  "http://localhost:8080/api/v1/tasks/$TASK/events?limit=200" | jq

# 构建并持久化时间线（异步），然后读取事件
curl -H "X-API-Key: $API_KEY" \
  "http://localhost:8080/api/v1/tasks/$TASK/timeline?mode=summary&persist=true"

# 预览时间线而不持久化
curl -H "X-API-Key: $API_KEY" \
  "http://localhost:8080/api/v1/tasks/$TASK/timeline?mode=full&include_payloads=false&persist=false" | jq
```

## 事件语义

- 摘要模式将活动已调度/已开始/已完成折叠为单行，包含持续时间，并删除大型负载。
- 完整模式包含原始标记和更细粒度的步骤。
- UI 中的徽章表示来源：
  - `stream` — 应用发出的保存到 `event_logs` 的 SSE 事件。
  - `timeline` — 从 Temporal 历史派生。

## 性能与幂等性

- 对 `event_logs` 的写入是异步且批处理的，以避免影响工作流延迟。
- 时间线构建器分页获取 Temporal 历史，可按需触发。
- 可选的未来增强：添加物化标记，以避免重复构建时重新插入现有时间线段。

## 模式

`event_logs`（由 `migrations/postgres/004_event_logs.sql` 创建）：

```
(id UUID PK, workflow_id TEXT, type TEXT, agent_id TEXT NULL,
 message TEXT, timestamp TIMESTAMPTZ, seq BIGINT NULL, stream_id TEXT NULL,
 created_at TIMESTAMPTZ)
```

## 安全

- 所有端点通过网关认证尊重租户/用户范围。
- 时间线默认：`mode=summary`、`include_payloads=false` 以避免敏感数据泄露。

## OpenAPI

- 网关在 `GET /openapi.json` 发布 OpenAPI，包含这些端点。
