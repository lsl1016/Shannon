# Shannon 测试指南

全面指南，涵盖从单元测试到端到端场景的 Shannon 平台测试。

## 测试类型

### 单元测试
- **Go**：与代码放在一起，作为 `_test.go` 文件
- **Rust**：在 crate 内使用内联 `#[cfg(test)]` 模块
- **Python**：位于 `python/llm-service/tests/`

### 集成测试
- **Temporal 工作流**：使用 Temporal 测试套件进行内存执行，并带有存根活动
- **冒烟测试**：基础跨服务连接、持久化、指标验证
- **端到端（E2E）**：位于 `tests/` 下使用 Docker Compose 的多服务场景

## 快速开始

```bash
# 1. 运行所有单元测试
make test

# 2. 启动完整服务栈
make dev

# 3. 运行冒烟测试（健康检查、gRPC、持久化、指标）
make smoke

# 4. 运行 E2E 场景
tests/e2e/run.sh
```

## 详细测试命令

### 按语言的单元测试

```bash
# Go 测试（带竞态检测）
cd go/orchestrator && go test -race ./...

# Rust 测试（带输出）
cd rust/agent-core && cargo test -- --nocapture

# Python 测试（带覆盖率）
cd python/llm-service && python3 -m pytest --cov

# WASI 沙箱测试
wat2wasm docs/assets/hello-wasi.wat -o /tmp/hello-wasi.wasm
cd rust/agent-core && cargo run --example wasi_hello -- /tmp/hello-wasi.wasm
```

### Temporal 工作流测试

```bash
# 导出工作流历史
make replay-export WORKFLOW_ID=task-dev-XXX OUT=history.json

# 测试确定性
make replay HISTORY=history.json

# 运行所有重放测试
make ci-replay
```

### 冒烟测试详情

冒烟测试（`make smoke`）验证：
- Temporal UI 可达性（http://localhost:8088）
- Agent-Core gRPC 健康检查和 ExecuteTask
- Orchestrator SubmitTask + GetTaskStatus 流程
- LLM 服务健康检查/live/ready 端点
- PostgreSQL 连接和迁移
- Prometheus 指标端点（:2112、:2113）

在此仓库副本中，向量搜索默认禁用，因此冒烟测试不需要本地 Qdrant 实例。

### 手动服务验证

```bash
# Agent-Core 健康检查
grpcurl -plaintext localhost:50051 shannon.agent.AgentService/HealthCheck

# Orchestrator 提交任务
grpcurl -plaintext \
  -d '{"metadata":{"user_id":"dev","session_id":"s1"},"query":"Say hello"}' \
  localhost:50052 shannon.orchestrator.OrchestratorService/SubmitTask

# LLM 服务健康检查
curl -fsS http://localhost:8000/health
curl -fsS http://localhost:8000/health/ready

# 指标端点
curl http://localhost:2112/metrics | head  # Orchestrator
curl http://localhost:2113/metrics | head  # Agent Core
```

## 测试文件位置

### Go 测试
- `go/orchestrator/internal/workflows/*_test.go` - 工作流测试
- `go/orchestrator/internal/activities/*_test.go` - 活动测试
- `go/orchestrator/internal/session/manager_test.go` - 会话管理
- `go/orchestrator/tests/replay/workflow_replay_test.go` - 重放验证

### Rust 测试
- `rust/agent-core/src/memory.rs` - TTL、限制、LRU 淘汰
- `rust/agent-core/src/wasi_sandbox.rs` - 路径验证、沙箱文件系统
- `rust/agent-core/src/tools.rs` - 工具执行器测试

### Python 测试
- `python/llm-service/tests/test_manager.py` - 提供商路由、缓存、回退
- `python/llm-service/tests/test_ratelimiter.py` - 速率限制
- `python/llm-service/tests/test_tools.py` - 工具执行

## CI 流水线

GitHub Actions 运行：
1. Rust 测试（带 clippy 检查）
2. Go 构建和测试（带竞态检测）
3. Python 检查（ruff）和 pytest
4. Temporal 工作流重放测试
5. 覆盖率报告（仅供参考）

## 覆盖率要求与设置

### 覆盖率阈值
- **Go**：最低 50% 覆盖率（当前：约 57%）
- **Python**：基线 20% 覆盖率（当前：约 15%，目标：70%）
- **Rust**：仅供参考（尚无最低要求）

### 运行覆盖率测试

```bash
# 运行所有覆盖率门禁
make coverage-gate

# 各语言覆盖率
make coverage-go       # Go 覆盖率（带阈值检查）
make coverage-python   # Python 覆盖率（带 venv 设置）

# 对于 Rust 覆盖率，直接使用 cargo tarpaulin：
cd rust/agent-core && cargo tarpaulin --out Html

# 生成详细报告
cd go/orchestrator && go test -coverprofile=coverage.out -covermode=atomic ./...
cd python/llm-service && pytest --cov=. --cov-report=html
```

### 覆盖率较好的模块
- **Go Budget Manager**：81%+ 覆盖率
- **Go Circuit Breaker**：61%+ 覆盖率
- **Python Rate Limiter**：良好的时序测试覆盖率
- **Rust Memory Manager**：LRU 淘汰和 TTL 测试

### 覆盖率目标

当前关注领域：
- Gateway 执行：超时、速率限制、断路器
- Budget Manager：边缘情况、数据库操作、令牌跟踪
- LLM 路由：提供商回退、速率限制、缓存
- WASI 沙箱：路径遍历防护、资源限制
- 模式工作流：CoT、ToT、ReAct、Debate、Reflection

## 故障排查

### 常见问题

**Orchestrator 无法连接到 Temporal**
- 验证环境中的 `TEMPORAL_HOST=temporal:7233`
- 检查 Temporal worker 是否已启动：`docker compose logs temporal`

**PostgreSQL 迁移失败**
```bash
docker compose down -v  # 清理卷
make dev                # 全新启动
```

**LLM 服务未就绪**
- 提供商 API 密钥在开发模式下是可选的
- 没有密钥时服务可能显示 ready=false，但仍可工作

**端口冲突**
确保以下端口空闲：
- 50051（Agent-Core gRPC）
- 50052（Orchestrator gRPC）
- 8000（LLM Service HTTP）
- 8088（Temporal UI）
- 5432（PostgreSQL）
- 6379（Redis）

### 调试命令

```bash
# 查看所有日志
make logs

# 查看特定服务日志
docker compose logs -f orchestrator
docker compose logs -f agent-core
docker compose logs -f llm-service

# 检查数据库状态
docker compose exec postgres psql -U shannon -d shannon \
  -c "SELECT workflow_id, status FROM task_executions ORDER BY created_at DESC LIMIT 5;"

# 检查 Redis 会话
docker compose exec redis redis-cli KEYS "session:*"

# 列出 Temporal 工作流
docker compose exec temporal temporal workflow list --address temporal:7233
```

## 性能测试

```bash
# 使用并发请求进行负载测试
for i in {1..10}; do
  ./scripts/submit_task.sh "测试查询 $i" &
done
wait

# 在负载期间监控指标
watch -n 1 'curl -s http://localhost:2112/metrics | grep shannon_'
```

## 清理

```bash
# 停止服务
make down

# 清理所有数据（包括卷）
make clean
```
