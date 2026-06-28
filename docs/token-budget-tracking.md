# 令牌预算与成本追踪

## 概述

Shannon 实现了双路径令牌追踪系统，确保在所有工作流模式下准确报告成本，同时在启用预算时防止重复记录。

**关键原则**：每次代理执行**恰好一次**记录令牌使用量——通过预算活动（启用预算时）或工作流模式（禁用预算时）。

## 架构

### 记录路径

```
┌─────────────────────────────────────────────────────────────────┐
│                     工作流执行                                   │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ├── 预算启用？ ──────┐
                 │                          │
        ┌────────▼─────────┐     ┌─────────▼──────────┐
        │   是（预算）      │     │  否（非预算）       │
        └────────┬─────────┘     └──────────┬─────────┘
                 │                           │
                 ▼                           ▼
    ┌────────────────────────┐   ┌──────────────────────────┐
    │ ExecuteAgentWithBudget │   │    ExecuteAgent          │
    │    （活动）             │   │    （活动）              │
    └────────┬───────────────┘   └──────────┬───────────────┘
             │                               │
             ▼                               │
    ┌────────────────────────┐              │
    │ 在活动中记录令牌使用     │              │
    │ （budget.go）          │              │
    └────────────────────────┘              │
                                             ▼
                                  ┌──────────────────────────┐
                                  │ 模式记录令牌              │
                                  │ （parallel/react/等）    │
                                  └──────────────────────────┘
                 │                           │
                 └───────────┬───────────────┘
                             ▼
                ┌────────────────────────────┐
                │    token_usage 表           │
                │  （每个代理一条记录）        │
                └────────────────────────────┘
```

## 核心组件

### 1. 预算执行活动

**文件**：`go/orchestrator/internal/activities/budget.go`

**函数**：`ExecuteAgentWithBudget`

**职责**：
- 对每个代理强制执行令牌预算限制
- 在令牌约束下调用 LLM 服务
- **立即记录令牌使用量**（第 346-356 行）
- 向工作流返回执行结果

**记录逻辑**：
```go
// 第 346-356 行：在活动内部记录使用量
err = b.budgetManager.RecordUsage(ctx, &budget.BudgetTokenUsage{
    UserID:         input.UserID,
    SessionID:      input.AgentInput.SessionID,
    TaskID:         input.TaskID,
    AgentID:        input.AgentInput.AgentID,
    Model:          actualModel,
    Provider:       actualProvider,
    InputTokens:    inputTokens,
    OutputTokens:   outputTokens,
    IdempotencyKey: idempotencyKey,
})
```

**使用场景**：
- `budgetPerAgent > 0`（Parallel、Sequential、Hybrid 模式）
- `opts.BudgetAgentMax > 0`（React、Chain-of-Thought、Debate、Tree-of-Thoughts）
- 带预算约束的 SupervisorWorkflow

---

### 2. 非预算执行活动

**文件**：`go/orchestrator/internal/activities/agent.go`

**函数**：`ExecuteAgent`

**职责**：
- 在无预算约束下执行代理
- 调用 LLM 服务
- **不记录令牌使用量**（由模式记录）
- 向工作流返回执行结果

**使用场景**：
- SimpleTaskWorkflow
- 无预算约束的工作流
- Decompose/Refine/Synthesis 阶段（始终非预算）

---

### 3. 工作流模式记录

所有工作流模式实现条件性令牌记录以避免重复：

#### 守卫模式

```go
// 预算路径：活动记录，模式跳过
if budgetPerAgent <= 0 {  // 或 opts.BudgetAgentMax <= 0
    _ = workflow.ExecuteActivity(ctx, constants.RecordTokenUsageActivity, ...)
}
```

#### 带记录的模式

| 模式 | 文件 | 守卫位置 | 代理 ID |
|---------|------|----------------|-----------|
| **Parallel** | `patterns/execution/parallel.go` | 第 267 行 | 结果 AgentID |
| **Sequential** | `patterns/execution/sequential.go` | 第 289 行 | 结果 AgentID |
| **React** | `patterns/react.go` | 第 154、367、493 行 | reasoner、action、synthesizer |
| **Chain-of-Thought** | `patterns/chain_of_thought.go` | 第 145、236 行 | cot-reasoner、cot-clarifier |
| **Debate** | `patterns/debate.go` | 第 201、354 行 | debater AgentIDs |
| **Tree-of-Thoughts** | `patterns/tree_of_thoughts.go` | 第 272-350 行 | tot-generator-{id} |
| **Reflection** | `patterns/wrappers.go`、`patterns/reflection.go` | 第 212-245 行（初始）、第 123-159 行（综合） | reflection-initial、reflection-synth |
| **ScheduledTask** | `workflows/scheduled/scheduled_task_workflow.go` | 版本门控 `scheduled_quota_record_v1` | 不适用（记录租户配额，非每代理） |

#### 无记录的模式（委托）

| 模式 | 文件 | 委托目标 |
|---------|------|-------------------|
| **Hybrid** | `patterns/execution/hybrid.go` | 委托给 Parallel（第 275 行） |

---

### 4. 工作流级别记录（始终记录）

某些工作流阶段始终记录使用量（无预算约束）：

**OrchestratorWorkflow - Decompose**
- 文件：`internal/workflows/orchestrator_router.go:218`
- AgentID：`"decompose"`
- 元数据：`{"phase": "decompose"}`

**SupervisorWorkflow - Decompose**
- 文件：`internal/workflows/supervisor_workflow.go:337`
- AgentID：`"decompose"`
- 元数据：`{"phase": "decompose"}`

**ResearchWorkflow - 阶段**
- Refine：`strategies/research.go:204` → AgentID：`"research-refiner"`
- Decompose：`strategies/research.go:265` → AgentID：`"decompose"`
- Synthesis：`strategies/research.go:826` → AgentID：`"synthesis"`

**SimpleTaskWorkflow**
- 文件：`internal/workflows/simple_workflow.go:487`
- AgentID：`"simple-agent"`
- 始终记录（单代理，无预算拆分）

---

## 令牌使用量记录

### 数据库模式

**表**：`token_usage`

```sql
CREATE TABLE token_usage (
    id                uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id           uuid,
    provider          varchar(50)  NOT NULL,
    model             varchar(255) NOT NULL,
    prompt_tokens     integer      NOT NULL,
    completion_tokens integer      NOT NULL,
    total_tokens      integer      NOT NULL,
    cost_usd          numeric(10,6) NOT NULL,
    created_at        timestamp with time zone DEFAULT CURRENT_TIMESTAMP,
    task_id           uuid REFERENCES task_executions(id) ON DELETE CASCADE
);
```

**索引**：
- `id` 主键
- `task_id` 外键 → `task_executions(id)`
- `user_id` 索引，用于用户级别成本查询
- `provider`、`model` 索引，用于提供商/模型分析
- `created_at` 索引，用于时间序列查询

---

### 记录流程

```
┌──────────────────────────────────────────────────────────────────┐
│ 1. LLM 提供商返回响应                                            │
│    - model_used: "gpt-5-2025-08-07"                             │
│    - provider: "openai"                                         │
│    - input_tokens: 732                                          │
│    - output_tokens: 1464                                        │
└────────────────────┬─────────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ 2. 活动捕获令牌计数                                              │
│    - AgentExecutionResult.InputTokens = 732                     │
│    - AgentExecutionResult.OutputTokens = 1464                   │
│    - AgentExecutionResult.ModelUsed = "gpt-5-2025-08-07"        │
│    - AgentExecutionResult.Provider = "openai"                   │
└────────────────────┬─────────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ 3. 记录决策（预算 vs 非预算）                                     │
│                                                                  │
│  如果 budgetPerAgent > 0：                                       │
│    → ExecuteAgentWithBudget 在活动中记录（budget.go:346）        │
│    → 工作流模式跳过记录（守卫防止重复）                           │
│                                                                  │
│  否则（budgetPerAgent <= 0）：                                   │
│    → ExecuteAgent 不记录                                         │
│    → 工作流模式通过 RecordTokenUsageActivity 记录                │
└────────────────────┬─────────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ 4. RecordTokenUsageActivity（activities/session.go）             │
│    - 从 workflow_id 解析 task_id                                 │
│    - 计算成本：pricing.CostForSplit(model, in, out)              │
│    - 插入 token_usage 表                                         │
└────────────────────┬─────────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ 5. 数据库持久化（PostgreSQL）                                     │
│    INSERT INTO token_usage (                                     │
│      user_id, provider, model,                                  │
│      prompt_tokens, completion_tokens, total_tokens,            │
│      cost_usd, task_id                                          │
│    ) VALUES (...)                                               │
└────────────────────┬─────────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ 6. 工作流聚合（metadata/aggregate.go）                           │
│    - 对任务的 token_usage 记录求和                                │
│    - 计算总成本                                                   │
│    - 填充 TaskResult.Metadata：                                  │
│      {                                                           │
│        "model_used": "gpt-5-2025-08-07",                        │
│        "provider": "openai",                                    │
│        "input_tokens": 4362,                                    │
│        "output_tokens": 8727,                                   │
│        "total_tokens": 13089,                                   │
│        "cost_usd": 0.183258                                     │
│      }                                                           │
└────────────────────┬─────────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────────┐
│ 7. API 响应（GET /api/v1/tasks/{id}）                            │
│    {                                                             │
│      "task_id": "task-...",                                     │
│      "status": "TASK_STATUS_COMPLETED",                         │
│      "usage": {                                                 │
│        "total_tokens": 13089,                                   │
│        "prompt_tokens": 4362,                                   │
│        "completion_tokens": 8727,                               │
│        "cost_usd": 0.183258                                     │
│      },                                                          │
│      "metadata": {                                              │
│        "model_used": "gpt-5-2025-08-07",                        │
│        "provider": "openai"                                     │
│      }                                                           │
│    }                                                             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 零令牌执行

### 仅工具路径

当未发生 LLM 推理时，某些代理执行按设计返回 **0 个 LLM 令牌**：

**常见场景**：
- **强制工具调用**：代理在没有 LLM 推理的情况下使用工具（`forced_tools` 模式）
- **缓存命中**：完全从缓存提供响应（未来功能）
- **仅工具工作流**：仅执行工具而不生成文本的工作流

**示例**（来自 ResearchWorkflow）：
```go
// 仅工具的网络搜索执行
result := executeWebSearch(ctx, query)
// 返回：InputTokens=0，OutputTokens=0，ToolExecutions=[...]
```

参见[深度研究概述](deep-research-overview.md#差距填补与收敛)了解差距填补工具执行详情。

---

### 记录行为

**默认**：零令牌执行**不记录**到 `token_usage` 表

**理由**：
- 无 LLM 成本可追踪
- 无可计费的令牌消耗
- 减少对非 LLM 操作的数据库写入

**实现**：
```go
// 模式记录守卫（parallel.go、sequential.go）
if result.TokensUsed == 0 && result.InputTokens == 0 {
    if recordZeroToken, ok := context["record_zero_token"].(bool); !ok || !recordZeroToken {
        logger.Warn("跳过零令牌记录（未发生 LLM 推理）")
        return nil  // 跳过记录
    }
}
```

**位置**：
- `go/orchestrator/internal/workflows/patterns/execution/sequential.go:316-380`
- `go/orchestrator/internal/workflows/patterns/execution/parallel.go:295-320`

---

### 启用零令牌记录

为**审计追踪目的**记录零令牌执行：

**通过 API 上下文**：
```bash
curl -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "仅工具执行的任务",
    "context": {
      "record_zero_token": true
    }
  }'
```

**通过工作流上下文**：
```go
baseContext := map[string]interface{}{
    "record_zero_token": true,  // 启用零令牌记录
}
```

**启用时**：
- 零令牌记录插入 `token_usage` 表
- `cost_usd` = 0.0
- 元数据可区分仅工具执行与 LLM 执行

**用例**：
- 合规/审计要求（追踪所有代理执行）
- 调试工具执行工作流
- 仅工具路径与 LLM 路径的分析

---

### 区分零令牌类型

审查具有 0 令牌的 `token_usage` 记录时：

```sql
-- 查找零令牌执行
SELECT
    tu.model,
    tu.provider,
    tu.total_tokens,
    tu.cost_usd,
    te.metadata->>'tool_executions' as tools_used
FROM token_usage tu
JOIN task_executions te ON tu.task_id = te.id
WHERE tu.total_tokens = 0
ORDER BY tu.created_at DESC;
```

**解释**：
- `tools_used IS NOT NULL` → 强制工具调用（预期为零）
- `tools_used IS NULL` → 潜在问题（LLM 本应返回令牌）

---

## 防止重复记录

### 问题（修复前）

**问题**：预算执行记录了两次使用量：
1. **活动记录**：`budget.go:346`（在 ExecuteAgentWithBudget 内部）
2. **模式记录**：`parallel.go:289`（工作流级别）

**影响**：2× 令牌计数、2× 成本计算、重复的数据库行

**示例**：
```
实际使用量：2,196 令牌（$0.030744）
记录量：     4,392 令牌（$0.061488）  ← 2 倍高估
```

---

### 解决方案（方案 B - 条件守卫）

**策略**：在启用预算时阻止模式级别记录

**实现**：用预算检查守卫所有模式级别记录

```go
// 之前（重复）
_ = workflow.ExecuteActivity(ctx, constants.RecordTokenUsageActivity, ...)

// 之后（无重复）
if budgetPerAgent <= 0 {  // 仅当非预算时记录
    _ = workflow.ExecuteActivity(ctx, constants.RecordTokenUsageActivity, ...)
}
```

**守卫变体**：
- Parallel/Sequential/Hybrid：`if budgetPerAgent <= 0`
- React/CoT/Debate：`if opts.BudgetAgentMax <= 0`
- Tree-of-Thoughts：`if tokenBudget > 0 { 预算 } else { 记录 }`

---

### 验证

**测试查询**：
```sql
-- 检查重复记录（应返回 0 行）
SELECT
  prompt_tokens,
  completion_tokens,
  model,
  COUNT(*) as record_count
FROM token_usage tu
JOIN task_executions te ON tu.task_id = te.id
WHERE te.workflow_id = 'task-...'
GROUP BY prompt_tokens, completion_tokens, model
HAVING COUNT(*) > 1;
```

**预期结果**：0 行（无重复）

---

## 预算配置

### 设置预算

**通过 API**：
```bash
curl -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "复杂研究任务",
    "context": {
      "budget_agent_max": 50000,  # 每个代理最大令牌数
      "budget_total_max": 200000  # 整个工作流最大令牌数
    }
  }'
```

**通过模板**：
```yaml
name: research_workflow
defaults:
  budget_agent_max: 30000
  budget_total_max: 150000
```

**通过环境**：
```bash
# .env 中的默认预算
BUDGET_AGENT_MAX=40000
BUDGET_TOTAL_MAX=200000
```

---

### 预算执行

**SupervisorWorkflow 示例**：
```go
// 确定每个代理的预算
agentMaxTokens := 0
if v, ok := baseContext["budget_agent_max"].(int); ok {
    agentMaxTokens = v
}

// 计算每个代理的预算
budgetPerAgent := 0
if agentMaxTokens > 0 && len(tasks) > 0 {
    budgetPerAgent = agentMaxTokens / len(tasks)
}

// 带预算执行
if budgetPerAgent > 0 {
    // 预算路径：活动记录
    err := workflow.ExecuteActivity(ctx,
        constants.ExecuteAgentWithBudgetActivity,
        activities.BudgetedAgentInput{
            AgentInput: agentInput,
            MaxTokens:  budgetPerAgent,
            UserID:     input.UserID,
            TaskID:     workflowID,
            ModelTier:  modelTier,
        }).Get(ctx, &result)
} else {
    // 非预算路径：模式记录
    err := workflow.ExecuteActivity(ctx,
        activities.ExecuteAgent,
        agentInput).Get(ctx, &result)
}
```

---

## 成本计算

### 定价配置

**文件**：`config/models.yaml`

```yaml
pricing:
  defaults:
    combined_per_1k: 0.005
  models:
    openai:
      gpt-5-2025-08-07:
        input_per_1k: 0.0060   # $0.006 每 1K 输入令牌
        output_per_1k: 0.0180  # $0.018 每 1K 输出令牌
```

参见 [centralized-pricing.md](centralized-pricing.md) 了解详情。

---

### 计算逻辑

**文件**：`go/orchestrator/internal/pricing/pricing.go`

**函数**：`CostForSplit`

```go
func CostForSplit(model string, inputTokens, outputTokens int) float64 {
    provider := DetectProvider(model)

    // 查找定价配置
    inputPrice := getInputPricePerToken(provider, model)
    outputPrice := getOutputPricePerToken(provider, model)

    // 计算成本
    inputCost := float64(inputTokens) * inputPrice
    outputCost := float64(outputTokens) * outputPrice

    return inputCost + outputCost
}
```

**示例**：
```
模型：gpt-5-2025-08-07
输入：732 令牌 × $0.000006 = $0.004392
输出：1464 令牌 × $0.000018 = $0.026352
总计：$0.030744
```

---

## 元数据聚合

### 聚合逻辑

**文件**：`go/orchestrator/internal/metadata/aggregate.go`

**函数**：`AggregateAgentMetadata`（旧版辅助函数）

此辅助函数从内存中的代理执行结果构建轻量级摘要（model_used、provider、input_tokens、output_tokens、total_tokens），并可能为调试目的添加尽力而为的 `agent_usages` 数组。

> ⚠️ `agent_usages` 已弃用，不适用于外部消费者。如需完整的用量和成本数据（包括综合、分解、工具和失败尝试），请依赖 `model_breakdown`，它从 `token_usage` 表构建。

工作流仍然调用 `AggregateAgentMetadata` 来填充任务级别元数据，但计费、报告和公共 API 应将 `model_breakdown` 视为真实来源。

---

## 监控与调试

### 数据库查询

**检查任务的令牌使用量**：
```sql
SELECT
    tu.model,
    tu.provider,
    tu.prompt_tokens,
    tu.completion_tokens,
    tu.total_tokens,
    tu.cost_usd,
    tu.created_at
FROM token_usage tu
JOIN task_executions te ON tu.task_id = te.id
WHERE te.workflow_id = 'task-...'
ORDER BY tu.created_at;
```

**按任务聚合成本**：
```sql
SELECT
    te.workflow_id,
    COUNT(tu.id) as agent_count,
    SUM(tu.prompt_tokens) as total_input,
    SUM(tu.completion_tokens) as total_output,
    SUM(tu.total_tokens) as total_tokens,
    SUM(tu.cost_usd) as total_cost
FROM task_executions te
LEFT JOIN token_usage tu ON tu.task_id = te.id
WHERE te.user_id = '...'
GROUP BY te.workflow_id
ORDER BY total_cost DESC
LIMIT 10;
```

**检测重复记录**：
```sql
SELECT
    prompt_tokens,
    completion_tokens,
    model,
    COUNT(*) as duplicates
FROM token_usage
WHERE task_id = (
    SELECT id FROM task_executions WHERE workflow_id = 'task-...'
)
GROUP BY prompt_tokens, completion_tokens, model
HAVING COUNT(*) > 1;
```

---

### Orchestrator 日志

**预算跟踪**：
```bash
docker compose -f deploy/compose/docker-compose.yml logs orchestrator | \
  grep "Budget used"
```

**令牌记录**：
```bash
docker compose -f deploy/compose/docker-compose.yml logs orchestrator | \
  grep "RecordTokenUsage"
```

**成本计算**：
```bash
docker compose -f deploy/compose/docker-compose.yml logs orchestrator | \
  grep "cost_usd"
```

---

### Temporal 工作流历史

**查看工作流事件**：
```bash
docker compose exec temporal temporal workflow describe \
  --workflow-id task-... \
  --address temporal:7233
```

**检查活动执行**：
```bash
docker compose exec temporal temporal workflow show \
  --workflow-id task-... \
  --address temporal:7233 | \
  grep -A 5 "RecordTokenUsageActivity"
```

---

## 最佳实践

### 1. 始终使用拆分令牌计数

**推荐**：
```go
cost := pricing.CostForSplit(model, inputTokens, outputTokens)
```

**避免**：
```go
cost := pricing.CostForTokens(model, totalTokens)  // 不够准确
```

**原因**：输入和输出令牌价格不同

---

### 2. 传播模型信息

**AgentExecutionResult 中必需**：
- `ModelUsed`（例如 "gpt-5-2025-08-07"）
- `Provider`（例如 "openai"）
- `InputTokens`（提示词令牌）
- `OutputTokens`（补全令牌）

**示例**：
```go
result := activities.AgentExecutionResult{
    Response:      llmResponse.Content,
    TokensUsed:    llmResponse.Usage.TotalTokens,
    InputTokens:   llmResponse.Usage.PromptTokens,
    OutputTokens:  llmResponse.Usage.CompletionTokens,
    ModelUsed:     llmResponse.Model,
    Provider:      "openai",
    AgentID:       agentID,
}
```

---

### 3. 守卫模式级别记录

**添加新模式时**，始终守卫记录：

```go
// 检查预算执行是否激活
if budgetPerAgent <= 0 {  // 或 opts.BudgetAgentMax <= 0
    // 记录令牌使用量
    _ = workflow.ExecuteActivity(ctx,
        constants.RecordTokenUsageActivity,
        activities.TokenUsageInput{
            UserID:       userID,
            SessionID:    sessionID,
            TaskID:       workflowID,
            AgentID:      agentID,
            Model:        result.ModelUsed,
            Provider:     result.Provider,
            InputTokens:  result.InputTokens,
            OutputTokens: result.OutputTokens,
            Metadata:     map[string]interface{}{"phase": "pattern_name"},
        }).Get(ctx, nil)
}
```

---

### 4. 包含阶段元数据

**目的**：在分析中区分记录来源

**示例**：
- `{"phase": "parallel"}`
- `{"phase": "react-reasoner"}`
- `{"phase": "decompose"}`
- `{"phase": "synthesis"}`

**用法**：
```sql
-- 按阶段分析令牌使用量
SELECT
    metadata->>'phase' as phase,
    COUNT(*) as executions,
    AVG(total_tokens) as avg_tokens,
    SUM(cost_usd) as total_cost
FROM token_usage
GROUP BY metadata->>'phase'
ORDER BY total_cost DESC;
```

---

### 5. 更改后验证无重复

**修改令牌记录逻辑后**：

1. 运行启用预算的测试工作流
2. 查询重复项：
```sql
SELECT COUNT(*) FROM (
    SELECT prompt_tokens, completion_tokens, model, COUNT(*)
    FROM token_usage
    WHERE task_id IN (
        SELECT id FROM task_executions
        WHERE workflow_id = 'test-task-...'
    )
    GROUP BY prompt_tokens, completion_tokens, model
    HAVING COUNT(*) > 1
) duplicates;
```
3. 预期结果：`0`

---

## 相关文档

- [集中式定价配置](centralized-pricing.md) - 定价配置与成本计算
- [速率感知预算](rate-aware-budgeting.md) - 速率限制管理与控制
- [多代理工作流架构](multi-agent-workflow-architecture.md) - 工作流模式与执行
- [模式使用指南](pattern-usage-guide.md) - 何时使用每种工作流模式
- [API 参考](api-reference.md) - 任务提交与状态的 REST API 端点

---

## 故障排查

### 缺失令牌计数

**症状**：API 响应中 `input_tokens` 和 `output_tokens` 为 0

**原因**：
1. LLM 提供商未返回令牌计数
2. 代理活动未捕获令牌计数
3. 回退到 `TokensUsed`（仅总计）

**解决方案**：
```go
// 确保 LLM 响应包含令牌计数
if result.InputTokens == 0 && result.OutputTokens == 0 {
    if result.TokensUsed > 0 {
        // 近似拆分（60/40）
        result.InputTokens = result.TokensUsed * 6 / 10
        result.OutputTokens = result.TokensUsed - result.InputTokens
    }
}
```

---

### 重复记录

**症状**：令牌计数是预期值的 2 倍

**原因**：
1. 模式级别记录未守卫
2. 预算检查缺失或不正确
3. 活动和模式均记录

**解决方案**：
1. 为模式记录添加预算守卫
2. 验证守卫条件：`if budgetPerAgent <= 0`
3. 运行重复检测查询
4. 检查 Temporal 工作流历史中的重复活动

---

### 成本不正确

**症状**：成本与预期值不匹配

**原因**：
1. `models.yaml` 中定价配置错误
2. 模型名称不匹配（例如 "gpt-5" vs "gpt-5-2025-08-07"）
3. 提供商检测失败
4. 使用组合定价而非拆分定价

**解决方案**：
1. 验证 `config/models.yaml` 中的定价
2. 检查模型名称是否完全匹配：`SELECT DISTINCT model FROM token_usage`
3. 强制提供商：`result.Provider = "openai"`
4. 使用 `CostForSplit` 而非 `CostForTokens`

---

### 预算未执行

**症状**：代理超出预算但无错误

**原因**：
1. 预算未传递给活动
2. 预算检查已禁用
3. 使用 `ExecuteAgent` 而非 `ExecuteAgentWithBudget`

**解决方案**：
1. 验证上下文中的预算：`budget_agent_max`
2. 检查活动调用：
```go
// 应使用 ExecuteAgentWithBudget
err := workflow.ExecuteActivity(ctx,
    constants.ExecuteAgentWithBudgetActivity,
    activities.BudgetedAgentInput{
        AgentInput: agentInput,
        MaxTokens:  budgetPerAgent,  // ← 确保 > 0
        ...
    })
```
3. 审查预算管理器日志：`grep "Budget exceeded"`

---

## 缓存感知令牌会计（迁移 121）

Shannon 跟踪**两个并行的总令牌字段**，同时在 `token_usage`（每次 LLM 调用）和 `task_executions`（每个工作流）中持久化：

| 字段 | 公式 | 用途 |
|-------|---------|---------|
| `total_tokens` | `prompt_tokens + completion_tokens` | OpenAI 兼容 API 响应；响应形状元数据 |
| `cache_aware_total_tokens` | `prompt + completion + cache_read + cache_creation` | 配额跟踪、真实成本、企业计费 |

两者有意保持不同：`total_tokens` 保持 OpenAI 语义，使现有 SDK 客户端（以及 `/v1/chat/completions`/`/v1/completions` 响应中的 `usage.total_tokens`）保持不变。`cache_aware_total_tokens` 是企业配额计数器应读取的字段。

### 不变式

对于两个表中的每一行：

```
cache_aware_total_tokens
  == prompt_tokens + completion_tokens + cache_read_tokens + cache_creation_tokens
```

运行 `scripts/verify_cache_quota_invariant.sh` 进行确认。空输出（每行不匹配 = 0，每任务不匹配 = 0）表示不变式成立。

### 写入路径

`token_usage` 有**四条**写入路径。所有路径都必须填充 `cache_aware_total_tokens`：

| 路径 | 文件 | 说明 |
|------|------|------|
| BudgetManager（编排工作流） | `internal/budget/manager.go` `RecordUsage` / `storeUsage` | 在 `TotalTokens` 之后立即计算 `CacheAwareTotalTokens` |
| Gateway completions 代理 | `cmd/gateway/internal/openai/handler.go` `recordCompletionUsage` | `/v1/completions` 绕过编排器 — 其 INSERT 内联计算该字段 |
| Gateway 工具代理 | `cmd/gateway/internal/handlers/tools.go` `recordToolUsage` | `/api/v1/tools/.../execute` 直接 INSERT — 将 `tokens` 存储在 `completion_tokens` 下，使不变式成立而无需伪造缓存字段 |
| TaskExecution 写入器（汇总，单独表） | `internal/db/task_writer.go` | 将每个工作流汇总写入 `task_executions`；在每次 `INSERT` 前防御性重新计算，使调用者无法偏离 |

`internal/activities/budget.go`（`shouldRecordUsage`）中的零令牌守卫允许纯缓存命中：`input + output == 0` 但 `cache_read > 0` 或 `cache_creation > 0` 的调用会被记录，以便提示缓存成本仍然计费。

### 读取/配额路径已更新

| 路径 | 文件 | 行为 |
|------|------|----------|
| 工作流汇总 SUM | `internal/server/service.go`（`getTaskMetadataFromDB`） | `SUM(cache_aware_total_tokens)`，带基于部分的回退 |
| 每个代理明细 | `internal/server/service.go`（`agent_usages`） | 在 `total_tokens` 旁边发出 `cache_aware_total_tokens` |
| 使用量报告 | `internal/budget/manager.go` `GetUsageReport` | 填充 `UsageReport.CacheAwareTotalTokens`；每个模型的 `ModelUsage.CacheAwareTokens` |
| 会话配额更新 | `internal/workflows/strategies/research.go` | `session.TokensUsed` 读取 `report.CacheAwareTotalTokens`（迁移中回退到 `TotalTokens`） |
| 会话 API（`tokens_used`） | `cmd/gateway/internal/handlers/session.go` | 单会话和列表端点均 SUM `cache_aware_total_tokens` |
| 每个任务的内存元数据 | `internal/metadata/aggregate.go` | 除 `total_tokens` 外还发出 `meta["cache_aware_total_tokens"]` |

### gRPC 表面

`RecordTokenUsageRequest`（proto 标签 7-9）携带 `cache_read_tokens`、`cache_creation_tokens`、`cache_creation_1h_tokens`。`OrchestratorService.RecordUsage` 钩子接受带有预计算 `CacheAwareTotalTokens` 的 `RecordUsageDetails` 结构，用于企业覆盖。

### shannon-cloud Cherry-Pick 检查清单

将 `feat/cache-aware-quota-rollup` 提交 cherry-pick 到 `shannon-cloud` 后：

1. 确认 `migrations/postgres/121_cache_aware_total_tokens.sql` 已在生产环境应用。
2. 在 shannon-cloud 对 `OrchestratorService.RecordUsage` 的覆盖中，将配额计数器从任何 `tokensUsed` 风格的参数切换为 `details.CacheAwareTotalTokens`。
3. **审计任何读取 `SUM(total_tokens)` / `tokens_used` 或写入自身 `token_usage` 行的云端 SQL**。每次出现都是缓存泄漏的候选。OSS 审计发现了四条写入路径和六条读取路径（上表）——假设云端差异具有相同形态。
4. 对生产数据库运行 `scripts/verify_cache_quota_invariant.sh` — 预期两个不匹配计数均为 0。
5. 观察配额仪表板：每个租户的令牌使用量将上升（这是泄漏最终被记录 — 不是回归）。
