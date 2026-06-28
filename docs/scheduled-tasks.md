# 计划任务

Shannon 支持使用 Temporal 的原生 Schedule API 执行周期性任务。用户可以创建基于 cron 的计划，按指定间隔自动执行任务。

## 功能特性

- **基于 Cron 的计划**，支持时区
- **资源限制**，防止滥用
- **预算控制**，每次执行均可设置
- **执行历史**，附带成本追踪
- **暂停/恢复/删除**操作
- **多租户隔离**，包含用户/租户所有权

## 架构

```
用户 → Gateway → Orchestrator gRPC → Schedule Manager → Temporal Schedule API
                                                             ↓
                                           ScheduledTaskWorkflow (包装器)
                                                             ↓
                                           OrchestratorWorkflow (现有)
```

### 组件

- **Schedule Manager** (`internal/schedules/manager.go`)：业务逻辑、Temporal API 集成、资源限制执行
- **ScheduledTaskWorkflow** (`internal/workflows/scheduled/`)：包装器工作流，用于跟踪执行、强制执行租户配额，并委托给现有工作流
- **Schedule Activities** (`internal/activities/schedule_activities.go`)：Temporal 活动，包括用于配额执行的 `PauseScheduleForQuota`
- **数据库表**：
  - `scheduled_tasks`：计划配置（cron、查询、预算等）
  - `scheduled_task_executions`：执行历史，包含时间戳、状态、成本

## 配置

### 环境变量

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `SCHEDULE_MAX_PER_USER` | `50` | 每个用户的最大计划数 |
| `SCHEDULE_MIN_INTERVAL_MINS` | `60` | 运行之间的最小间隔（分钟） |
| `SCHEDULE_MAX_BUDGET_USD` | `10.0` | 每次执行的最大预算（美元） |

### 示例

```bash
# 允许每个用户 100 个计划，每次运行预算 20 美元
SCHEDULE_MAX_PER_USER=100
SCHEDULE_MAX_BUDGET_USD=20.0
SCHEDULE_MIN_INTERVAL_MINS=30
```

## API 端点

所有端点都需要身份验证，并强制执行用户/租户所有权。

### 创建计划

```bash
POST /api/v1/schedules
Content-Type: application/json
Authorization: Bearer <token>

{
  "name": "每日摘要",
  "description": "生成每日活动摘要",
  "cron_expression": "0 9 * * *",
  "timezone": "America/New_York",
  "task_query": "总结昨天的活动",
  "task_context": {
    "report_format": "markdown"
  },
  "max_budget_per_run_usd": 5.0,
  "timeout_seconds": 600
}
```

**响应：**
```json
{
  "schedule_id": "550e8400-e29b-41d4-a716-446655440000",
  "message": "计划创建成功",
  "next_run_at": "2025-12-16T09:00:00-05:00"
}
```

### 列出计划

```bash
GET /api/v1/schedules?page=1&page_size=50&status=ACTIVE
Authorization: Bearer <token>
```

**响应：**
```json
{
  "schedules": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "每日摘要",
      "cron_expression": "0 9 * * *",
      "timezone": "America/New_York",
      "status": "ACTIVE",
      "next_run_at": "2025-12-16T09:00:00-05:00",
      "total_runs": 45,
      "successful_runs": 43,
      "failed_runs": 2
    }
  ],
  "total": 1,
  "page": 1,
  "page_size": 50
}
```

### 获取计划

```bash
GET /api/v1/schedules/{id}
Authorization: Bearer <token>
```

### 更新计划

```bash
PUT /api/v1/schedules/{id}
Content-Type: application/json
Authorization: Bearer <token>

{
  "cron_expression": "0 10 * * *",
  "max_budget_per_run_usd": 8.0
}
```

### 暂停计划

```bash
POST /api/v1/schedules/{id}/pause
Content-Type: application/json
Authorization: Bearer <token>

{
  "reason": "临时维护"
}
```

### 恢复计划

```bash
POST /api/v1/schedules/{id}/resume
Content-Type: application/json
Authorization: Bearer <token>

{
  "reason": "维护完成"
}
```

### 删除计划

```bash
DELETE /api/v1/schedules/{id}
Authorization: Bearer <token>
```

## Cron 表达式格式

使用标准 cron 语法（5 个字段）：

```
┌───────────── 分钟 (0 - 59)
│ ┌───────────── 小时 (0 - 23)
│ │ ┌───────────── 日期 (1 - 31)
│ │ │ ┌───────────── 月份 (1 - 12)
│ │ │ │ ┌───────────── 星期 (0 - 6) (周日至周六)
│ │ │ │ │
* * * * *
```

### 示例

| 表达式 | 描述 |
|------------|-------------|
| `0 9 * * *` | 每天上午 9:00 |
| `0 */4 * * *` | 每 4 小时 |
| `0 0 * * 1` | 每周一午夜 |
| `30 8 1 * *` | 每月第一天上午 8:30 |
| `0 12 * * 1-5` | 工作日中午 12:00 |

## 数据库模式

### scheduled_tasks

| 列 | 类型 | 描述 |
|--------|------|-------------|
| `id` | UUID | 主键 |
| `user_id` | UUID | 所有者用户 ID |
| `tenant_id` | UUID | 租户 ID（可为空） |
| `name` | VARCHAR(255) | 计划名称 |
| `description` | TEXT | 可选描述 |
| `cron_expression` | VARCHAR(100) | Cron 计划 |
| `timezone` | VARCHAR(50) | IANA 时区（例如 "America/New_York"） |
| `task_query` | TEXT | 要执行的任务查询 |
| `task_context` | JSONB | 额外的上下文参数 |
| `max_budget_per_run_usd` | DECIMAL(10,2) | 每次执行的预算限制 |
| `timeout_seconds` | INTEGER | 工作流超时时间 |
| `temporal_schedule_id` | VARCHAR(255) | Temporal 计划 ID（唯一） |
| `status` | VARCHAR(20) | ACTIVE、PAUSED 或 DELETED |
| `next_run_at` | TIMESTAMP | 下一次计划执行时间 |
| `last_run_at` | TIMESTAMP | 上次执行时间 |
| `total_runs` | INTEGER | 总执行次数 |
| `successful_runs` | INTEGER | 成功执行次数 |
| `failed_runs` | INTEGER | 失败执行次数 |

### scheduled_task_executions

| 列 | 类型 | 描述 |
|--------|------|-------------|
| `id` | UUID | 主键 |
| `schedule_id` | UUID | 外键，关联 scheduled_tasks |
| `task_id` | VARCHAR(255) | Temporal 工作流 ID |
| `status` | VARCHAR(20) | RUNNING、COMPLETED、FAILED、CANCELLED |
| `total_cost_usd` | DECIMAL(10,4) | 执行成本 |
| `error_message` | TEXT | 错误详情（如果失败） |
| `started_at` | TIMESTAMP | 执行开始时间 |
| `completed_at` | TIMESTAMP | 执行完成时间 |

## 实现细节

### 验证

1. **Cron 表达式**：在创建 Temporal 计划之前使用 `robfig/cron/v3` 解析器进行验证
2. **资源限制**：在创建计划之前进行检查：
   - 用户的活动计划数必须 < `SCHEDULE_MAX_PER_USER`
   - 预算必须 ≤ `SCHEDULE_MAX_BUDGET_USD`
3. **所有权**：所有操作都会验证 user_id 和 tenant_id 是否与已认证的上下文匹配

### 执行流程

1. Temporal 在计划时间触发 `ScheduledTaskWorkflow`
2. 工作流在 `scheduled_task_executions` 和 `task_executions` 中记录执行开始
3. **配额预检查**（`scheduled_quota_check_v1`）：通过 `CheckTenantQuota` 活动检查租户每日/每月令牌配额。如果超出 → 记录 FAILED 状态，发出 `QUOTA_EXCEEDED` 事件，返回 nil（无 Temporal 重试）
4. 工作流将 `OrchestratorWorkflow` 作为子工作流执行，附带任务查询
5. 捕获子工作流结果（成功/失败、成本、令牌）
6. **配额用量记录**（`scheduled_quota_record_v1`）：通过 `RecordTenantQuotaUsage` 活动记录已消耗的令牌（适用于 COMPLETED 和 FAILED 运行）。当元数据缺少 `total_tokens` 时，回退使用 `result.TokensUsed`
7. 工作流记录执行完成，包含状态、成本和元数据
8. 更新计划统计信息（`total_runs`、`successful_runs`、`failed_runs`）

### 错误处理

- **计划创建失败**：如果数据库插入失败，回滚 Temporal 计划
- **执行失败**：记录在执行历史中，计划保持活动状态
- **预算超限**：子工作流因预算错误而失败
- **配额超限**：计划运行被拒绝，状态为 FAILED，发出 `QUOTA_EXCEEDED` 事件用于 webhook/LINE 通知。计划保持活动状态（下次运行将重新检查）
- **配额检查失败**：故障开放 —— 如果配额检查活动出错，执行继续（与网关行为一致）
- **Temporal 不可用**：计划操作返回服务不可用

## 部署

### 数据库迁移

```bash
PGPASSWORD=shannon psql -h localhost -U shannon -d shannon \
  -f migrations/postgres/009_scheduled_tasks.sql
```

### 服务重启

计划功能需要重启 orchestrator 服务以加载 schedule manager：

```bash
docker compose -f deploy/compose/docker-compose.yml restart orchestrator
```

## 监控

### 指标

- 通过 Temporal UI 查看计划执行历史
- 查询 `scheduled_task_executions` 进行成本分析
- 监控 `scheduled_tasks` 表中的计划统计信息

### Temporal UI

导航到 `http://localhost:8088` → Schedules 以查看：
- 计划状态（运行中/已暂停）
- 最近的执行记录
- 下一次计划时间
- 执行积压

### 数据库查询

```sql
-- 按用户统计活动计划
SELECT user_id, COUNT(*)
FROM scheduled_tasks
WHERE status = 'ACTIVE'
GROUP BY user_id;

-- 每个计划的总成本
SELECT s.name, SUM(e.total_cost_usd) as total_cost
FROM scheduled_tasks s
JOIN scheduled_task_executions e ON s.id = e.schedule_id
WHERE e.status = 'COMPLETED'
GROUP BY s.id, s.name
ORDER BY total_cost DESC;

-- 失败率
SELECT
  name,
  total_runs,
  failed_runs,
  ROUND(100.0 * failed_runs / NULLIF(total_runs, 0), 2) as failure_rate
FROM scheduled_tasks
WHERE total_runs > 0
ORDER BY failure_rate DESC;
```

## 限制

- 最小间隔：60 分钟（可配置）
- 每个用户最大计划数：50（可配置）
- 每次执行最大预算：10 美元（可配置）
- 时区支持：IANA 时区数据库
- Cron 精度：分钟级（不支持秒）

## 未来增强

- 计划模板（每日/每周/每月预设）
- 执行结果通知（邮件、webhook）
- 基于历史成本的动态预算调整
- 计划依赖（串联多个计划）
- 失败执行的重试策略
