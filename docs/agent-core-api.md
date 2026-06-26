# Shannon Agent Core - API 文档

## 目录

1. [gRPC API](#grpc-api)
2. [Tool Registry API](#tool-registry-api)
3. [Python Integration API](#python-integration-api)
4. [错误码](#错误码)
5. [示例](#示例)

## gRPC API

agent core 在 50051 端口暴露 gRPC 服务。

### ExecuteTask

通过 Agent Core gateway 执行任务。Modes 是建议性的，并会路由到 Python；Rust 会统一强制执行请求级策略（超时、速率限制、熔断器）。

**请求：**

```protobuf
message ExecuteTaskRequest {
  TaskMetadata metadata = 1;
  string query = 2;
  google.protobuf.Struct context = 3;
  ExecutionMode mode = 4;
  repeated string available_tools = 5;
  AgentConfig config = 6;
  SessionContext session_context = 7;
}
```

**响应：**

```protobuf
message ExecuteTaskResponse {
  string task_id = 1;
  StatusCode status = 2;
  string result = 3;
  repeated ToolCall tool_calls = 4;
  repeated ToolResult tool_results = 5;
  ExecutionMetrics metrics = 6;
  string error_message = 7;
  AgentState final_state = 8;
}
```

**示例：**

```rust
let request = ExecuteTaskRequest {
    query: "Calculate the sum of 42 and 58".to_string(),
    mode: ExecutionMode::Simple as i32,
    ..Default::default()
};

let response = client.execute_task(request).await?;
println!("Result: {}", response.result);
```

### StreamExecuteTask

执行任务并返回流式更新。

**请求：** 与 ExecuteTask 相同

**响应流：**

```protobuf
message TaskUpdate {
  string task_id = 1;
  AgentState state = 2;
  string message = 3;
  ToolCall tool_call = 4;
  ToolResult tool_result = 5;
  double progress = 6;
}
```

### DiscoverTools

基于 query 和 filters 发现可用工具。

**请求：**

```protobuf
message DiscoverToolsRequest {
  string query = 1;
  repeated string categories = 2;
  repeated string tags = 3;
  bool exclude_dangerous = 4;
  int32 max_results = 5;
}
```

**响应：**

```protobuf
message DiscoverToolsResponse {
  repeated ToolCapability tools = 1;
}

message ToolCapability {
  string id = 1;
  string name = 2;
  string description = 3;
  string category = 4;
  google.protobuf.Struct input_schema = 5;
  google.protobuf.Struct output_schema = 6;
  repeated string required_permissions = 7;
  int64 estimated_duration_ms = 8;
  bool is_dangerous = 9;
  string version = 10;
  string author = 11;
  repeated string tags = 12;
  repeated ToolExample examples = 13;
  RateLimit rate_limit = 14;
  int64 cache_ttl_ms = 15;
}
```

**示例：**

```rust
let request = DiscoverToolsRequest {
    query: "search".to_string(),
    exclude_dangerous: true,
    max_results: 5,
    ..Default::default()
};

let response = client.discover_tools(request).await?;
for tool in response.tools {
    println!("Found tool: {} - {}", tool.name, tool.description);
}
```

### GetToolCapability

获取特定工具的详细信息。

**请求：**

```protobuf
message GetToolCapabilityRequest {
  string tool_id = 1;
}
```

**响应：**

```protobuf
message GetToolCapabilityResponse {
  ToolCapability tool = 1;
}
```

### GetCapabilities

获取 agent 能力和配置。

**响应：**

```protobuf
message GetCapabilitiesResponse {
  repeated string supported_tools = 1;
  repeated ExecutionMode supported_modes = 2;
  int64 max_memory_mb = 3;
  int32 max_concurrent_tasks = 4;
  string version = 5;
}
```

### HealthCheck

检查 agent 健康状态。

**响应：**

```protobuf
message HealthCheckResponse {
  bool healthy = 1;
  string message = 2;
  int64 uptime_seconds = 3;
  int32 active_tasks = 4;
  double memory_usage_percent = 5;
}
```

## Tool Registry API

### Rust API

```rust
use shannon_agent_core::tool_registry::{ToolRegistry, ToolCapability, ToolDiscoveryRequest};

// Create registry
let registry = ToolRegistry::new();

// Register a tool
let tool = ToolCapability {
    id: "my_tool".to_string(),
    name: "My Tool".to_string(),
    description: "A custom tool".to_string(),
    category: "custom".to_string(),
    // ... other fields
};
registry.register_tool(tool);

// Discover tools
let request = ToolDiscoveryRequest {
    query: Some("search".to_string()),
    categories: None,
    tags: None,
    exclude_dangerous: Some(true),
    max_results: Some(10),
};
let tools = registry.discover_tools(request);

// Get specific tool
let tool = registry.get_tool("calculator");
```

## Python Integration API

Rust agent 通过 HTTP REST API 与 Python LLM service 通信。

### Tool Selection

**Endpoint：** `POST /tools/select`

**请求：**

```json
{
  "task": "Search for information about Rust programming",
  "context": {
    "session_id": "abc123",
    "previous_tools": ["web_search"]
  },
  "exclude_dangerous": true,
  "max_tools": 3
}
```

**响应：**

```json
{
  "calls": [
    {
      "tool_name": "web_search",
      "parameters": {
        "query": "Rust programming",
        "max_results": 5
      }
    }
  ],
  "provider_used": "openai"
}
```

### Tool Execution

**Endpoint：** `POST /tools/execute`

**请求：**

```json
{
  "tool_name": "calculator",
  "parameters": {
    "expression": "42 + 58"
  }
}
```

**响应：**

```json
{
  "success": true,
  "output": 100,
  "error": null
}
```

### Task Analysis

**Endpoint：** `POST /analyze_task`

**请求：**

```json
{
  "query": "Build a web application with user authentication",
  "context": {
    "session_id": "abc123"
  }
}
```

**响应：**

```json
{
  "execution_mode": "complex",
  "subtasks": [
    "Design database schema",
    "Implement authentication",
    "Create API endpoints",
    "Build frontend"
  ],
  "estimated_tokens": 5000,
  "confidence": 0.85
}
```

### Tool List

**Endpoint：** `GET /tools/list`

**Query 参数：**

* `exclude_dangerous`（boolean）：过滤掉危险工具

**响应：**

```json
[
  "calculator",
  "web_search",
  "database_query",
  "code_executor"
]
```

## 错误码

### gRPC 状态码

| Code | Name               | 描述        |
| ---- | ------------------ | --------- |
| 0    | OK                 | 成功        |
| 3    | INVALID_ARGUMENT   | 无效请求参数    |
| 5    | NOT_FOUND          | 工具或资源未找到  |
| 8    | RESOURCE_EXHAUSTED | 内存或速率限制超出 |
| 13   | INTERNAL           | 内部服务器错误   |
| 14   | UNAVAILABLE        | 服务暂时不可用   |

### Agent 错误类型

```rust
pub enum AgentError {
    // Tool errors
    ToolNotFound { name: String },
    ToolExecutionFailed { tool: String, reason: String },
    ToolTimeout { tool: String, timeout_ms: u64 },
    
    // Resource errors
    MemoryExhausted { requested: usize, available: usize },
    TokenLimitExceeded { used: u32, limit: u32 },
    
    // Sandbox errors
    SandboxViolation { operation: String },
    WasmExecutionError(String),
    
    // Network errors
    NetworkError(String),
    ServiceUnavailable { service: String },
    
    // Configuration errors
    ConfigurationError(String),
    InvalidParameter { param: String, reason: String },
}
```

## 示例

### 完整任务执行示例

```rust
use shannon_agent_core::grpc_server::proto::agent::*;
use tonic::Request;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Connect to agent
    let mut client = agent_service_client::AgentServiceClient::connect(
        "http://localhost:50051"
    ).await?;
    
    // Prepare request
    let request = Request::new(ExecuteTaskRequest {
        query: "Search for the latest Rust programming news and summarize it".to_string(),
        mode: ExecutionMode::Standard as i32,
        available_tools: vec![
            "web_search".to_string(),
            "calculator".to_string(),
        ],
        config: Some(AgentConfig {
            max_iterations: 5,
            timeout_seconds: 30,
            enable_sandbox: true,
            memory_limit_mb: 256,
            enable_learning: false,
        }),
        session_context: Some(SessionContext {
            session_id: "user-123".to_string(),
            history: vec![
                "User: What is Rust?".to_string(),
                "Agent: Rust is a systems programming language...".to_string(),
            ],
            total_tokens_used: 1500,
            total_cost_usd: 0.003,
            ..Default::default()
        }),
        ..Default::default()
    });
    
    // Execute task
    let response = client.execute_task(request).await?;
    let response = response.into_inner();
    
    // Process response
    println!("Task ID: {}", response.task_id);
    println!("Status: {:?}", response.status);
    println!("Result: {}", response.result);
    
    // Check metrics
    if let Some(metrics) = response.metrics {
        println!("Tokens used: {}", metrics.token_usage.as_ref().unwrap().total_tokens);
        println!("Execution time: {}ms", metrics.latency_ms);
    }
    
    Ok(())
}
```

### 流式执行示例

```rust
use tokio_stream::StreamExt;

let mut stream = client.stream_execute_task(request).await?.into_inner();

while let Some(update) = stream.next().await {
    match update {
        Ok(task_update) => {
            println!("Progress: {}%", (task_update.progress * 100.0) as u32);
            println!("State: {:?}", task_update.state);
            println!("Message: {}", task_update.message);
            
            if let Some(tool_call) = task_update.tool_call {
                println!("Calling tool: {}", tool_call.tool_name);
            }
        }
        Err(e) => {
            eprintln!("Stream error: {}", e);
            break;
        }
    }
}
```

### 工具发现示例

```rust
// Discover calculation tools
let request = DiscoverToolsRequest {
    categories: vec!["calculation".to_string()],
    exclude_dangerous: true,
    max_results: 10,
    ..Default::default()
};

let response = client.discover_tools(Request::new(request)).await?;
let tools = response.into_inner().tools;

for tool in tools {
    println!("Tool: {} ({})", tool.name, tool.category);
    println!("  Description: {}", tool.description);
    println!("  Duration: ~{}ms", tool.estimated_duration_ms);
    println!("  Dangerous: {}", tool.is_dangerous);
    
    // Check rate limits
    if let Some(rate_limit) = tool.rate_limit {
        println!("  Rate limit: {}/min", rate_limit.requests_per_minute);
    }
}
```

### 错误处理示例

```rust
match client.execute_task(request).await {
    Ok(response) => {
        let response = response.into_inner();
        if response.status == StatusCode::Ok as i32 {
            println!("Success: {}", response.result);
        } else {
            eprintln!("Task failed: {}", response.error_message);
        }
    }
    Err(status) => {
        match status.code() {
            tonic::Code::InvalidArgument => {
                eprintln!("Invalid request: {}", status.message());
            }
            tonic::Code::ResourceExhausted => {
                eprintln!("Resource limit exceeded: {}", status.message());
            }
            tonic::Code::Unavailable => {
                eprintln!("Service unavailable, retry later");
            }
            _ => {
                eprintln!("Unexpected error: {}", status);
            }
        }
    }
}
```

## 速率限制

agent core 支持在多个层级强制执行速率限制：

1. **工具级别**：每个工具都可以指定速率限制
2. **请求级别**：每个请求的超时和预算限制
3. **全局级别**：整体系统速率限制

**注意：** HTTP 速率限制 headers（`X-RateLimit-*`）由 Gateway service 返回，而不是由 agent-core gRPC endpoints 直接返回。请查看 `go/orchestrator/cmd/gateway` 了解 HTTP 层面的速率限制。

## 指标

Prometheus metrics 可通过 `http://localhost:2113/metrics` 获取：

**任务指标：**

* `agent_core_tasks_total{mode, status}`：已处理任务总数
* `agent_core_task_duration_seconds{mode}`：任务执行时长
* `agent_core_task_tokens{mode, model}`：每个任务使用的 tokens

**工具指标：**

* `agent_core_tool_executions_total{tool_name, status}`：工具执行总次数
* `agent_core_tool_duration_seconds{tool_name}`：工具执行延迟
* `agent_core_tool_selection_duration_seconds{status}`：工具选择延迟

**内存指标：**

* `agent_core_memory_pool_used_bytes`：当前内存池使用量
* `agent_core_memory_pool_total_bytes`：总内存池大小

**gRPC 指标：**

* `agent_core_grpc_requests_total{method, status}`：gRPC 请求总数
* `agent_core_grpc_request_duration_seconds{method}`：gRPC 请求时长

**强制执行指标：**

* `agent_core_enforcement_drops_total{reason}`：被强制执行层丢弃的请求
* `agent_core_enforcement_allowed_total{outcome}`：被强制执行层允许的请求

**FSM 指标：**

* `agent_core_fsm_transitions_total{from_state, to_state}`：FSM 状态转换
* `agent_core_fsm_current_state`：当前 FSM 状态（编码为数字）

## 版本控制

API 遵循语义化版本控制：

* **Major**：破坏性变更
* **Minor**：新特性，向后兼容
* **Patch**：Bug 修复


通过 capabilities 检查版本：

```rust
let capabilities = client.get_capabilities(Request::new(GetCapabilitiesRequest {})).await?;
println!("Agent version: {}", capabilities.into_inner().version);
```
