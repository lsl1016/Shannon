# Shannon Agent Core - 架构文档

## 概览

Shannon Agent Core 是 agent 执行层的高性能 Rust 实现，提供安全沙箱、高效内存管理和智能工具编排。本文档描述了遵循 2025 年最佳实践的现代化架构。

## 架构原则

1. **关注点分离**：智能（Python）与执行（Rust）
2. **零拷贝操作**：尽量减少字符串克隆和内存分配
3. **现代并发**：使用 `OnceLock` 和 `std::sync::Once`，而不是 `lazy_static`
4. **全面错误处理**：通过 `thiserror` 使用带结构化错误的 `Result<T>` 类型
5. **可观测系统**：OpenTelemetry tracing 和 Prometheus metrics
6. **安全优先**：用于不可信代码执行的 WASI 沙箱

## 组件架构

```
┌─────────────────────────────────────────────────────────────┐
│                     gRPC Server (port 50051)                │
├─────────────────────────────────────────────────────────────┤
│                    Enforcement Gateway                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │  Timeouts    │  │  Rate Limits │  │ Circuit Breakers │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│                    Tool Execution Layer                     │
│ 工具执行层 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │   Tool   │  │   Tool   │  │   Tool   │  │   WASI   │  │
│  │ Registry │  │   Cache  │  │ Executor │  │  Sandbox │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    Infrastructure Layer                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Memory  │  │  Config  │  │ Tracing  │  │ Metrics  │  │
│  │   Pool   │  │  Manager │  │  (OTEL)  │  │  (Prom)  │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## 核心组件

### 1. Enforcement Gateway（`src/enforcement.rs`）

对每条代码路径进行统一的逐请求策略强制执行：

* 超时：每个请求的硬性墙钟时间限制
* Token 上限：拒绝估算 tokens 过高的请求
* 速率限制：简单的按 key token bucket
* 熔断器：按 key 的滚动错误窗口

  * 可选分布式限流器：设置 `ENFORCE_RATE_REDIS_URL` 以启用跨实例共享的 Redis-backed token bucket。

配置位于 `config/agent.yaml` 中的 `enforcement` 下，并支持环境变量覆盖（`ENFORCE_*`）。

### 2. 工具系统

#### Tool Registry（`src/tool_registry.rs`）

* 集中式工具能力管理
* 带过滤和相关性评分的发现 API
* 包含 schemas、permissions 和 TTL 的 metadata

#### Tool Cache（`src/tool_cache.rs`）

* 带可配置 TTL 的 LRU 缓存
* 确定性 cache key 生成
* 自动过期和清理
* 全面的统计信息跟踪

#### Tool Executor（`src/tools.rs`）

* 统一的工具执行接口
* 与 Python LLM service 集成
* 用于代码执行的 WASI 沙箱路由
* 自动结果缓存

### 3. WASI Sandbox（`src/wasi_sandbox.rs`）

安全的 WebAssembly 执行环境，具备：

* 文件系统隔离（只读 `/tmp` 访问）
* 内存限制（可配置，默认 256MB）
* 执行超时（默认 30s）
* 用于 CPU 使用控制的 fuel metering

### 4. 内存管理（`src/memory.rs`）

高效内存池，具备：

* 预分配内存块
* 自动垃圾回收
* 基于压力的拒绝
* 线程安全的分配/释放

### 5. 配置（`src/config.rs`）

集中式配置管理：

* 基于 YAML 的配置文件
* 环境变量覆盖（包括 enforcement：`ENFORCE_*`）
* 热重载支持（未来）
* 结构化配置类型

### 6. 可观测性

#### Tracing（`src/tracing.rs`）

* OpenTelemetry 集成
* W3C trace context 传播
* 活跃 span context 注入
* 跨服务 tracing 支持

#### Metrics（`src/metrics.rs`）

* Prometheus metrics 导出
* 工具执行 metrics
* 内存使用跟踪
* 缓存性能统计
* Enforcement metrics：按原因统计 drops、允许结果统计

## API 契约

### gRPC API

agent 暴露以下 gRPC 服务：

```protobuf
service AgentService {
  rpc ExecuteTask(ExecuteTaskRequest) returns (ExecuteTaskResponse);
  rpc StreamExecuteTask(ExecuteTaskRequest) returns (stream TaskUpdate);
  rpc GetCapabilities(GetCapabilitiesRequest) returns (GetCapabilitiesResponse);
  rpc HealthCheck(HealthCheckRequest) returns (HealthCheckResponse);
  rpc DiscoverTools(DiscoverToolsRequest) returns (DiscoverToolsResponse);
  rpc GetToolCapability(GetToolCapabilityRequest) returns (GetToolCapabilityResponse);
}
```

### Python-Rust 契约

Rust agent 通过 HTTP 与 Python LLM service 通信：

#### 工具选择

```
POST /tools/select
{
  "task": "string",
  "context": {},
  "exclude_dangerous": boolean,
  "max_tools": number
}
```

#### 工具执行

```
POST /tools/execute
{
  "tool_name": "string",
  "parameters": {}
}
```

#### 任务分析

```
POST /analyze_task
{
  "query": "string",
  "context": {}
}
```

## 错误处理

使用 `thiserror` 的全面错误分类：

```rust
pub enum AgentError {
    ToolNotFound { name: String },
    ToolExecutionFailed { tool: String, reason: String },
    MemoryExhausted { requested: usize, available: usize },
    SandboxViolation { operation: String },
    ConfigurationError(String),
    NetworkError(String),
    // ... 20+ error variants
}
```

## 性能优化

### 1. 零拷贝字符串

使用 `Cow<str>` 进行字符串处理，以避免不必要的分配：

```rust
pub fn process_text<'a>(input: &'a str) -> Cow<'a, str>
```

### 2. 惰性初始化

用于 metrics 的现代 `OnceLock` 模式：

```rust
static METRICS: OnceLock<HashMap<String, Counter>> = OnceLock::new();
```

### 3. 并行工具执行

使用 tokio 进行并发工具执行：

```rust
let futures = tools.iter().map(|tool| executor.execute_tool(tool));
let results = futures::future::join_all(futures).await;
```

### 4. 缓存优先架构

* 带可配置 TTL 的工具结果缓存
* 简单 query 的 LLM 响应缓存
* 发现结果缓存

## 安全模型

### WASI Sandbox 隔离

* 无网络访问
* 有限文件系统访问（只读 `/tmp`）
* 强制内存限制
* 通过 fuel metering 控制 CPU 使用

### 工具权限系统

```rust
pub struct ToolCapability {
    pub required_permissions: Vec<String>,
    pub is_dangerous: bool,
    pub requires_confirmation: bool,
}
```

### 输入校验

* 参数 schema 校验
* 输入大小限制
* 超时保护

## 测试策略

### 单元测试

* 组件级测试
* Mock dependencies
* 针对复杂逻辑的基于性质的测试

### 集成测试

* Python-Rust 契约验证
* 端到端工具执行
* 缓存行为验证
* 错误处理场景

### 性能测试

* 关键路径 benchmark
* 内存使用 profiling
* 并发执行压力测试

## 部署

### Docker Container

```dockerfile
FROM rust:latest as builder
WORKDIR /usr/src/app

# Copy manifests and sources
COPY rust/agent-core/Cargo.toml ./
COPY rust/agent-core/build.rs ./
COPY protos /protos
COPY rust/agent-core/src ./src

# Build
RUN apt-get update && apt-get install -y protobuf-compiler \
 && cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates netcat-openbsd \
 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /usr/src/app/target/release/shannon-agent-core /usr/local/bin/shannon-agent-core
EXPOSE 50051
CMD ["shannon-agent-core"]
```

### 配置

环境变量：

* `RUST_LOG`：日志级别
* `OTEL_EXPORTER_OTLP_ENDPOINT`：Tracing endpoint
* `MEMORY_POOL_SIZE_MB`：内存池大小
* `WASI_MEMORY_LIMIT_MB`：WASI 沙箱内存限制
* `LLM_SERVICE_URL`：Python LLM service base URL

注意：工具结果 cache TTL 通过 agent config（YAML）中的 `tools.cache_ttl_secs` 配置。不存在 `TOOL_CACHE_TTL_SECONDS` 环境变量覆盖。

### 健康检查

* 端口 `50051` 上的 gRPC HealthCheck RPC
* Metrics endpoint：`:2113/metrics`


## 未来增强

1. **WebAssembly Component Model**：支持 WASI Preview 2
2. **分布式缓存**：用于缓存共享的 Redis 集成
3. **GPU 加速**：用于 ML 操作的 CUDA/ROCm 支持
4. **多区域支持**：地理分布式 agent 部署
5. **高级监控**：自定义 metrics 和 tracing spans

## 贡献

请遵循以下准则：

1. 提交前使用 `cargo fmt` 和 `cargo clippy`
2. 为新功能添加测试
3. 针对 API 变更更新文档
4. 遵循错误处理最佳实践
5. 尽量减少不必要的分配

