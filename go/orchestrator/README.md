# Shannon Orchestrator (Go)

orchestrator 是 Shannon 的中心协调服务，通过 Temporal 使用基于模式的认知架构管理 AI agent 工作流、执行策略并处理会话状态。

## 🎯 核心职责

- **基于模式的编排** - 将任务路由到认知模式（CoT、ToT、ReAct、Debate、Reflection）
- **工作流管理** - 通过 Temporal workflows 协调多 agent 执行
- **预算管理** - 跟踪并执行 agents 之间的 token 使用限制
- **策略执行** - 集成 OPA 以处理安全和合规规则
- **会话管理** - 在交互之间维护对话状态
- **服务协调** - 在 Rust agent core 和 Python LLM services 之间路由

## 🏗️ 架构

```
用户请求 → gRPC Server (:50052)
    ↓
OrchestratorRouter（模式选择）
    ↓
模式分析 → 认知模式选择
    ├── Chain of Thought (CoT) - 顺序推理
    ├── Tree of Thoughts (ToT) - 带回溯的探索
    ├── ReAct - 推理 + 行动循环
    ├── Debate - 多 agent 论证
    └── Reflection - 自我改进迭代
    ↓
模式执行 → Agent 协调
    ↓
结果 → 综合 → 会话更新 → 响应
```

## 📁 项目结构

```
go/orchestrator/
├── main.go                      # 服务入口点
├── Dockerfile                   # 容器构建
├── internal/
│   ├── activities/              # Temporal activity 实现
│   │   ├── agent.go            # Agent 执行 activities
│   │   ├── budget.go           # 预算跟踪 activities
│   │   ├── decompose.go        # 任务分解
│   │   ├── synthesis.go        # 结果综合
│   │   └── metrics.go          # 模式指标跟踪
│   ├── workflows/               # Temporal workflow 定义
│   │   ├── orchestrator_router.go  # 主模式路由器
│   │   ├── supervisor_workflow.go  # 重试和监督
│   │   ├── simple_workflow.go      # 简单任务执行
│   │   ├── patterns/               # 认知模式
│   │   │   ├── chain_of_thought.go
│   │   │   ├── tree_of_thoughts.go
│   │   │   ├── react.go
│   │   │   ├── debate.go
│   │   │   └── reflection.go
│   │   ├── strategies/            # 旧版 workflow 策略
│   │   │   ├── dag.go
│   │   │   ├── exploratory.go
│   │   │   ├── research.go
│   │   │   └── scientific.go
│   │   └── execution/             # 执行模式
│   │       ├── parallel.go
│   │       ├── sequential.go
│   │       └── hybrid.go
│   ├── server/                  # gRPC service 实现
│   ├── policy/                  # OPA policy engine 集成
│   ├── budget/                  # Token 预算管理
│   ├── auth/                    # 身份验证和授权
│   ├── db/                      # PostgreSQL 操作
│   ├── health/                  # 健康检查和降级
│   ├── config/                  # 配置管理
│   ├── streaming/               # SSE/WebSocket streaming
│   └── circuitbreaker/          # 故障保护
├── histories/                   # Workflow replay 测试文件
├── tests/                       # 集成测试
│   └── replay/                  # 确定性测试
└── tools/replay/                # Temporal replay 工具
```

## 🚀 快速开始

### 前置条件
- Go 1.21+
- Docker 和 Docker Compose
- PostgreSQL、Redis、Temporal 正在运行

### 开发

```bash
# 安装依赖
go mod download

# 运行测试
go test -race ./...

# 构建二进制文件
go build -o orchestrator .

# 本地运行（需要服务）
./orchestrator
```

### Docker 部署

```bash
# 构建镜像
docker build -t shannon-orchestrator .

# 使用 compose 运行（推荐）
make dev  # 从仓库根目录
```

## ⚙️ 配置

配置从 `/app/config/shannon.yaml` 加载（在 Docker 中挂载）：

```yaml
# 关键配置部分
service:
  port: 50052           # gRPC 端口
  health_port: 8081     # 健康检查 HTTP 端口

policy:
  enabled: true         # OPA policy 执行
  mode: "dry-run"      # off | dry-run | enforce
  path: "/app/config/opa/policies"

temporal:
  host_port: "temporal:7233"
  namespace: "default"
  task_queue: "shannon-task-queue"

patterns:
  chain_of_thought:
    max_iterations: 10
    timeout: "5m"
  tree_of_thoughts:
    max_depth: 5
    branching_factor: 3
  react:
    max_steps: 15
    timeout: "10m"

budget:
  max_tokens_per_request: 10000
  max_cost_per_request: 1.0
```

### 环境变量

- `PRIORITY_QUEUES`（默认值：空）
  - 设置为 `on`/`true`/`1` 时，orchestrator 会为每个优先级队列启动一个 Temporal worker：
    - `shannon-tasks-critical`、`shannon-tasks-high`、`shannon-tasks`（normal）、`shannon-tasks-low`
  - 每个队列的并发量在 `main.go` 中调优

- `ENABLE_TOOL_SELECTION`（默认值：`1`）
  - 启用后，orchestrator 调用 LLM service `/tools/select` 来自动填充 `context.tool_calls`
  - 当 `TOOL_PARALLELISM > 1` 时，这会在 Agent Core 中启用并行工具执行

- 优先级 worker 并发（可选覆盖）：
  - `WORKER_ACT_CRITICAL` / `WORKER_WF_CRITICAL`（默认值：`12` / `12`）
  - `WORKER_ACT_HIGH` / `WORKER_WF_HIGH`（默认值：`10` / `10`）
  - `WORKER_ACT_NORMAL` / `WORKER_WF_NORMAL`（默认值：`8` / `8`）
  - `WORKER_ACT_LOW` / `WORKER_WF_LOW`（默认值：`4` / `4`）

- 单队列模式并发（当 `PRIORITY_QUEUES` 关闭时）：
  - `WORKER_ACT` / `WORKER_WF`（默认值：`10` / `10`）

### 按优先级提交

通过 `SubmitTaskRequest` 中的 `metadata.labels["priority"]` 设置优先级。

有效值：`critical`、`high`、`normal`、`low`（大小写不敏感）。无效值会回退到默认队列。

示例：
```go
req := &pb.SubmitTaskRequest{
    Metadata: &common.TaskMetadata{
        UserId: "user-123",
        Labels: map[string]string{"priority": "critical"},
    },
    Query: "Plan and execute task",
}
resp, err := client.SubmitTask(ctx, req)
```

## 🔧 关键功能

### 基于模式的工作流

**认知模式：**
- `ChainOfThought` - 逐步逻辑推理
- `TreeOfThoughts` - 通过回溯探索多条解决路径
- `ReAct` - 为交互式任务结合推理和行动
- `Debate` - 面向复杂决策的多 agent 论证
- `Reflection` - 迭代式自我改进

**核心工作流：**
- `OrchestratorRouter` - 选择模式的主入口点
- `SupervisorWorkflow` - 处理重试和监督
- `SimpleWorkflow` - 简单任务的直接执行

**关键 Activities：**
- `DecomposeTask` - 分析复杂度并创建子任务
- `ExecuteAgent` - 运行单个 agent 任务
- `SynthesizeResults` - 合并 agent 输出
- `UpdateSessionResult` - 持久化会话状态
- `RecordPatternMetrics` - 跟踪模式性能

### 预算管理

Token 使用量在多个层级跟踪：
- 带背压的按请求预算
- 带 circuit breakers 的按用户配额
- 执行前成本估算
- 实时使用量监控

### 策略执行

OPA policies 控制：
- 任务执行权限
- Agent 访问控制
- 资源使用限制
- 数据访问边界

### 健康和降级

负载下自动降级：
- Complex → Standard mode 回退
- 外部服务的 circuit breakers
- 优雅的超时处理
- 位于 `:8081/health` 的健康端点

## 📊 可观测性

### 指标（Prometheus 格式）
- **Endpoint**：`:2112/metrics`
- Workflow 执行时间
- 模式选择分布
- 每种模式的 token 使用量
- 按 workflow 类型统计的错误率

### 日志
- 使用 zap 的结构化 JSON 日志
- 用于请求跟踪的 correlation IDs
- 可通过 `LOG_LEVEL=debug` 启用 debug mode

### Streaming
- SSE：`GET /stream/sse?workflow_id=<id>`
- WebSocket：`GET /stream/ws?workflow_id=<id>`
- 说明：
  - 所有 child workflow 和 agent events 都统一到父级 `workflow_id` 下，形成单一 stream。
  - SSE/WS events 会缓冲在 Redis Streams（约 24h TTL）中，并在配置后持久化到 Postgres。

#### Synthesis Events
- `LLM_OUTPUT`（AgentID：`synthesis`）：最终综合内容（截断为 10k chars）。在 LLM 成功以及所有 fallback/simple 路径上发出。
- `DATA_PROCESSING` summary：轻量级 token 使用消息（例如 `"Used 1.5k tokens"`），基于 synthesis 报告的 model/tokens。
- 顺序：`LLM_OUTPUT` → summary（`DATA_PROCESSING` "Used … tokens"）→ `DATA_PROCESSING` "Answer ready" → `WORKFLOW_COMPLETED`。
- Bypass 行为：当 synthesis 被 bypass（单个合适结果）时，不会发出额外的 synthesis events；agent 自身的 `LLM_OUTPUT` 作为最终结果。

## 🧪 测试

### 单元测试
```bash
go test ./internal/...
```

### 集成测试
```bash
# 需要正在运行的服务
go test ./tests/integration/...
```

### Replay 测试
```bash
# 导出 workflow history
make replay-export WORKFLOW_ID=task-xxx OUT=histories/test.json

# 测试确定性
make replay HISTORY=histories/test.json

# 运行所有 replay 测试
go test ./tests/replay
```

## 🚨 常见问题

### Workflow 非确定性
- 确保 activities 中没有 `time.Sleep()`
- 在 workflows 中使用 `workflow.Sleep()`
- 使用一致名称注册所有 activities

### 预算超出
- 检查 config 中的 token 限制
- 通过 metrics 监控使用量
- 调整 `max_tokens_per_request`

### 模式选择
- 查看分解结果
- 检查模式置信度分数
- 监控模式指标

## 📚 更多文档

- [Pattern Usage Guide](../../docs/pattern-usage-guide.md)
- [Multi-Agent Architecture](../../docs/multi-agent-workflow-architecture.md)
- [Testing Guide](../../docs/testing.md)
- [Main README](../../README.md)
