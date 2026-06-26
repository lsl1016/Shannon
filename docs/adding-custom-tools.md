# 向 Shannon 添加自定义工具

**使用自定义工具扩展 Shannon**

## 目录

* [概览](#概览)
* [快速开始：添加 MCP 工具](#快速开始添加-mcp-工具)
* [添加 OpenAPI 工具](#添加-openapi-工具)
* [添加内置 Python 工具](#添加内置-python-工具)
* [配置参考](#配置参考)
* [测试与验证](#测试与验证)
* [故障排查](#故障排查)
* [安全最佳实践](#安全最佳实践)

## 概览

Shannon 支持三种方式添加自定义工具：

| 方法                 | 最适合                          | 代码变更        | 是否需要重启 |
| ------------------ | ---------------------------- | ----------- | ------ |
| **MCP Tools**      | 外部 HTTP APIs、快速原型            | 无           | ✅ 仅服务  |
| **OpenAPI Tools**  | 带有 OpenAPI specs 的 REST APIs | 无           | ✅ 仅服务  |
| **Built-in Tools** | 复杂逻辑、数据库访问、性能                | Python code | ✅ 仅服务  |

**核心特性：**

* ✅ 无需 Proto/Rust/Go 变更（通用容器）
* ✅ 通过 API 或 YAML config 动态注册
* ✅ 内置速率限制和熔断器
* ✅ 用于安全的域名白名单
* ✅ 成本跟踪和预算强制控制

## 快速开始：添加 MCP 工具

MCP（Model Context Protocol）工具可以在零代码变更的情况下，将任意 HTTP endpoint 集成为 Shannon 工具。

### 步骤 1：添加工具定义

编辑 `config/shannon.yaml` 中的 `mcp_tools`：

```yaml
mcp_tools:
  weather_forecast:
    enabled: true
    url: "https://api.weather.com/v1/forecast"
    func_name: "get_weather"
    description: "Get weather forecast for a location"
    category: "data"
    cost_per_use: 0.001
    parameters:
      - name: "location"
        type: "string"
        required: true
        description: "City name or coordinates"
      - name: "units"
        type: "string"
        required: false
        description: "Temperature units (celsius/fahrenheit)"
        enum: ["celsius", "fahrenheit"]
    headers:
      X-API-Key: "your_api_key_here"  # For MCP, header values are literal (no env expansion)
      # Tip: Prefer the runtime registration API below and inject secrets from your env at call time
```

**必填字段：**

* `enabled`：设置为 `true` 以激活
* `url`：HTTP endpoint（必须是 POST，接收 JSON）
* `func_name`：内部函数名
* `description`：LLM 可见描述
* `category`：工具类别（例如 `search`、`data`、`analytics`、`code`）
* `cost_per_use`：预估成本，单位 USD
* `parameters`：参数定义数组

**可选字段：**

* `headers`：用于认证的 HTTP headers

### 步骤 2：配置域名访问

**开发环境（宽松）：**
添加到 `.env`：

```bash
MCP_ALLOWED_DOMAINS=*  # Wildcard - allows all domains
```

**生产环境（推荐）：**

```bash
MCP_ALLOWED_DOMAINS=localhost,127.0.0.1,api.weather.com,api.example.com
```

或者在 `deploy/compose/docker-compose.yml` 中设置：

```yaml
services:
  llm-service:
    environment:
      - MCP_ALLOWED_DOMAINS=api.weather.com,api.stocks.com
```

### 步骤 3：添加 API Keys

将你的 API key 添加到 `.env`：

```bash
# MCP Tool API Keys
WEATHER_API_KEY=your_api_key_here
STOCK_API_KEY=your_stock_key_here
```

### 步骤 4：重启服务

**重要：** 你必须**重新创建**服务：

```bash
docker compose -f deploy/compose/docker-compose.yml up -d --force-recreate llm-service
```

等待健康检查：

```bash
docker inspect shannon-llm-service-1 --format='{{.State.Health.Status}}'
```

### 步骤 5：验证注册

检查日志：

```bash
docker compose logs llm-service | grep "Loaded MCP tool"
```

通过 API 列出工具：

```bash
curl http://localhost:8000/tools/list | jq .
```

获取工具 schema：

```bash
curl http://localhost:8000/tools/weather_forecast/schema | jq .
```

### 步骤 6：测试你的工具

**直接执行：**

```bash
curl -X POST http://localhost:8000/tools/execute \
-H "Content-Type: application/json" \
-d '{
  "tool_name": "weather_forecast",
  "parameters": {"location": "Tokyo", "units": "celsius"}
}'
```

**通过 workflow：**

```bash
SESSION_ID="test-$(date +%s)" ./scripts/submit_task.sh "What's the weather forecast for Tokyo?"
```

### 替代方案：运行时 API 注册

仅用于开发/测试（工具会在重启后丢失）：

```bash
# Set admin token in .env
MCP_REGISTER_TOKEN=your_secret_token

# Register tool
curl -X POST http://localhost:8000/tools/mcp/register \
-H "Authorization: Bearer your_secret_token" \
-H "Content-Type: application/json" \
-d '{
  "name": "weather_forecast",
  "url": "https://api.weather.com/v1/forecast",
  "func_name": "get_weather",
  "description": "Get weather forecast",
  "category": "data",
  "parameters": [
    {"name": "location", "type": "string", "required": true},
    {"name": "units", "type": "string", "enum": ["celsius", "fahrenheit"]}
  ]
}'
```

### MCP 请求约定

Shannon 发送 POST 请求：

```json
{
  "function": "get_weather",
  "args": {
    "location": "Tokyo",
    "units": "celsius"
  }
}
```

你的 endpoint 应该返回 JSON：

```json
{
  "temperature": 18,
  "condition": "Cloudy",
  "humidity": 65
}
```

## 添加 OpenAPI 工具

对于带有 OpenAPI 3.x 规范的 REST APIs，Shannon 会自动生成工具。

> 💡 v0.8 新增：对于需要自定义转换的特定领域 APIs，请查看 [Vendor Adapter Pattern](#vendor-adapter-pattern) 或完整的 [Vendor Adapters Guide](vendor-adapters.md)。

### 特性

**已支持：**

* ✅ OpenAPI 3.0 和 3.1 specs
* ✅ 基于 URL 或内联 spec 加载
* ✅ JSON 请求/响应体
* ✅ Path 和 query 参数
* ✅ Bearer、API Key（header/query）、Basic auth
* ✅ 按 `operationId` 或 tags 过滤操作
* ✅ 熔断器（5 次失败 → 60s 冷却）
* ✅ 带指数退避的重试逻辑（默认 2 次重试；可通过 `OPENAPI_RETRIES` 覆盖）
* ✅ 可配置的速率限制和超时
* ✅ 相对 server URLs（基于 spec URL 解析）
* ✅ 基础 `$ref` 解析（指向 `#/components/schemas/*` 的本地引用）

**限制（MVP）：**
Shannon OpenAPI 集成对约 70% 的 REST APIs（基于 JSON 且简单认证）已可用于生产。以下特性**尚不支持：**

* **❌ 文件上传 APIs（multipart/form-data）**

  * 无法上传文件或二进制数据
  * 变通方案：在 JSON body 中使用 base64 编码文件
  * 受影响 APIs：图像生成、文件处理、文档上传 APIs
* **❌ OAuth 保护的 APIs**

  * 无 OAuth 2.0 flows（Authorization Code、Client Credentials）
  * 只能使用 Bearer tokens（手动获取）
  * 受影响 APIs：Google APIs、GitHub、Slack、Twitter 等
  * 变通方案：手动获取 OAuth token，并使用 `bearer` `auth_type`
* **❌ 复杂参数编码**

  * 不支持 `style`、`explode` 或 `deepObject` 序列化
  * 仅支持基础 path/query 参数替换
  * 受影响 APIs：带有复杂数组/对象 query 参数的 APIs
* **❌ 多文件 OpenAPI Specs**

  * 不支持远程 `$ref` 解析（例如 `https://example.com/schemas/Pet.json`）
  * 仅支持本地 refs（`#/components/...`）
  * 变通方案：将外部 schemas 合并到单个 spec 文件中
* **❌ 高级 Schema 组合器**

  * 不支持 `allOf`、`oneOf`、`anyOf`
  * 仅基础类型映射
  * 受影响 APIs：带有多态类型或复杂校验的 APIs
* **❌ 表单编码请求**

  * 不支持 `application/x-www-form-urlencoded` content type
  * 仅支持 JSON 请求体

**运行良好的场景：**

* ✅ 简单 REST APIs，JSON 请求/响应
* ✅ 带有 Bearer/API Key/Basic 认证的 APIs
* ✅ 读多写少操作（GET 请求）
* ✅ 结构良好且带有本地 `$ref` references 的 specs
* ✅ Path 和 query 参数（基础类型）

**重要：** 对于带有相对 server URLs（例如 `/api/v3`）的 specs，你必须通过 `spec_url`（而不是 `spec_inline`）提供 spec，这样 Shannon 才能解析完整 base URL。示例：PetStore spec 包含 `servers: [{url: "/api/v3"}]`，当从 `https://petstore3.swagger.io/api/v3/openapi.json` 加载时，会解析为 `https://petstore3.swagger.io/api/v3`。

### 步骤 1：添加工具定义

编辑 `config/shannon.yaml` 中的 `openapi_tools`：

```yaml
openapi_tools:
  petstore:
    enabled: true
    spec_url: "https://petstore3.swagger.io/api/v3/openapi.json"
    # OR use inline spec:
    # spec_inline: |
    #   <paste OpenAPI JSON/YAML here>
    auth_type: "api_key"  # none|api_key|bearer|basic
    auth_config:
      api_key_name: "X-API-Key"           # Header name or query param name
      api_key_location: "header"          # header|query
      api_key_value: "$PETSTORE_API_KEY"  # Use $ prefix for env vars
    category: "data"
    base_cost_per_use: 0.001
    rate_limit: 30                        # Requests per minute
    timeout_seconds: 30                   # Request timeout
    max_response_bytes: 10485760          # Max response size (10MB)
    # Optional: Filter to specific operations
    operations:
      - "getPetById"
      - "findPetsByStatus"
    # Optional: Filter by tags
    # tags:
    #   - "pet"
    # Optional: Override base URL from spec
    # base_url: "https://custom-petstore.example.com"
```

### 认证示例

**Bearer Token（GitHub API）：**

```yaml
github:
  enabled: true
  spec_url: "https://raw.githubusercontent.com/github/rest-api-description/main/descriptions/api.github.com/api.github.com.json"
  auth_type: "bearer"
  auth_config:
    token: "$GITHUB_TOKEN"
  operations:
    - "repos/get"
    - "repos/list-for-user"
```

**Query 中的 API Key（OpenWeather）：**

```yaml
weather:
  enabled: true
  spec_url: "https://api.openweathermap.org/data/3.0/openapi.json"
  auth_type: "api_key"
  auth_config:
    api_key_name: "appid"
    api_key_location: "query"
    api_key_value: "$OPENWEATHER_API_KEY"
```

**Basic Auth：**

```yaml
custom_api:
  enabled: true
  spec_url: "https://api.example.com/openapi.json"
  auth_type: "basic"
  auth_config:
    username: "$API_USERNAME"
    password: "$API_PASSWORD"
```

### 步骤 2：配置环境

添加到 `.env`：

```bash
# OpenAPI Security
OPENAPI_ALLOWED_DOMAINS=*                # Comma-separated or * for dev
OPENAPI_MAX_SPEC_SIZE=5242880            # 5MB default
OPENAPI_FETCH_TIMEOUT=30                 # Seconds

# API Keys
PETSTORE_API_KEY=your_key_here
GITHUB_TOKEN=ghp_xxxxxxxxxxxxx
OPENWEATHER_API_KEY=your_key
API_USERNAME=username
API_PASSWORD=password

# Same registration token as MCP
MCP_REGISTER_TOKEN=your_admin_token
```

### 步骤 3：重启服务

```bash
docker compose -f deploy/compose/docker-compose.yml up -d --force-recreate llm-service
```

### 步骤 4：验证和测试

**先验证 spec：**

```bash
curl -X POST http://localhost:8000/tools/openapi/validate \
-H "Content-Type: application/json" \
-d '{"spec_url": "https://petstore3.swagger.io/api/v3/openapi.json"}' | jq .
```

**响应：**

```json
{
  "valid": true,
  "operations_count": 19,
  "operations": [
    {"operation_id": "getPetById", "method": "GET", "path": "/pet/{petId}"},
    {"operation_id": "addPet", "method": "POST", "path": "/pet"}
  ],
  "base_url": "https://petstore3.swagger.io/api/v3"
}
```

**列出已注册工具：**

```bash
curl http://localhost:8000/tools/list | grep Pet
```

**执行工具：**

```bash
curl -X POST http://localhost:8000/tools/execute \
-H "Content-Type: application/json" \
-d '{
  "tool_name": "getPetById",
  "parameters": {"petId": 1}
}' | jq .
```

### 替代方案：运行时注册

```bash
curl -X POST http://localhost:8000/tools/openapi/register \
-H "Content-Type: application/json" \
-H "Authorization: Bearer your_admin_token" \
-d '{
  "name": "petstore",
  "spec_url": "https://petstore3.swagger.io/api/v3/openapi.json",
  "auth_type": "none",
  "category": "data",
  "operations": ["getPetById", "findPetsByStatus"],
  "rate_limit": 15,
  "timeout_seconds": 10
}' | jq .
```

**响应包含验证信息：**

```json
{
  "success": true,
  "collection_name": "petstore",
  "operations_registered": ["getPetById", "findPetsByStatus"],
  "rate_limit": 15,
  "timeout_seconds": 10,
  "max_response_bytes": 10485760
}
```

## 添加内置 Python 工具

适用于复杂逻辑、数据库访问或性能关键操作。

### 何时使用内置工具

**当出现以下情况时使用内置工具：**

* 需要直接访问数据库/Redis
* 需要复杂 Python libraries（pandas、numpy）
* 性能关键操作（避免 HTTP 往返）
* 需要会话状态管理
* 实现安全敏感操作

**当出现以下情况时使用 MCP/OpenAPI：**

* 集成外部 APIs
* 偏好无代码部署
* 快速原型
* 第三方服务集成

### 步骤 1：创建工具类

在 `python/llm-service/llm_service/tools/builtin/my_custom_tool.py` 中创建文件：

```python
from typing import Any, Dict, List, Optional
from ..base import Tool, ToolMetadata, ToolParameter, ToolParameterType, ToolResult

class MyCustomTool(Tool):
    """
    Brief description of what this tool does.
    """

    def _get_metadata(self) -> ToolMetadata:
        """Define tool metadata."""
        return ToolMetadata(
            name="my_custom_tool",
            version="1.0.0",
            description="Clear description for LLM to understand when/how to use this tool",
            category="custom",  # search, data, analytics, code, file, custom
            author="Your Name",
            requires_auth=False,
            timeout_seconds=30,
            memory_limit_mb=128,
            sandboxed=False,
            session_aware=False,  # Set True if tool needs session state
            dangerous=False,      # Set True for file writes, code execution
            cost_per_use=0.001,   # USD per invocation
            rate_limit=60,        # Requests per minute (enforced by base class)
        )

    def _get_parameters(self) -> List[ToolParameter]:
        """Define tool parameters with validation."""
        return [
            ToolParameter(
                name="required_param",
                type=ToolParameterType.STRING,
                description="Description shown to LLM",
                required=True,
            ),
            ToolParameter(
                name="optional_number",
                type=ToolParameterType.INTEGER,
                description="An optional number parameter",
                required=False,
                default=10,
                min_value=1,
                max_value=100,
            ),
            ToolParameter(
                name="choice_param",
                type=ToolParameterType.STRING,
                description="Parameter with predefined choices",
                required=False,
                enum=["option1", "option2", "option3"],
            ),
        ]

    async def _execute_impl(
        self,
        session_context: Optional[Dict] = None,
        **kwargs
    ) -> ToolResult:
        """
        Execute the tool logic.

        Args:
            session_context: Session data if session_aware=True
            **kwargs: Tool parameters (validated automatically)

        Returns:
            ToolResult with success/error status
        """
        try:
            # Extract parameters (already validated by base class)
            required_param = kwargs.get("required_param")
            optional_number = kwargs.get("optional_number", 10)
            choice_param = kwargs.get("choice_param")

            # Your tool logic here
            result = self._do_work(required_param, optional_number, choice_param)

            return ToolResult(
                success=True,
                output=result,
                metadata={"processed": True},
                execution_time_ms=50,
            )

        except Exception as e:
            return ToolResult(
                success=False,
                output=None,
                error=f"Tool execution failed: {str(e)}"
            )

    def _do_work(self, param1, param2, param3):
        """Your actual implementation."""
        # Example: Database query, API call, computation
        return {"result": "success", "data": [1, 2, 3]}
```

### 步骤 2：注册工具

编辑 `python/llm-service/llm_service/api/tools.py` 并更新 `startup_event()` 注册列表：

```python
# Add import at top
from ..tools.builtin.my_custom_tool import MyCustomTool

# Add to registration list in startup_event()
@router.on_event("startup")
async def startup_event():
    registry = get_registry()

    tools_to_register = [
        WebSearchTool,
        CalculatorTool,
        FileReadTool,
        FileWriteTool,
        PythonWasiExecutorTool,
        MyCustomTool,  # Add your tool here
    ]

    for tool_class in tools_to_register:
        try:
            registry.register(tool_class)
            logger.info(f"Registered tool: {tool_class.__name__}")
        except Exception as e:
            logger.error(f"Failed to register {tool_class.__name__}: {e}")
```

### 步骤 3：重启服务

```bash
docker compose -f deploy/compose/docker-compose.yml up -d --force-recreate llm-service
```

### 步骤 4：测试工具

```bash
# Verify registration
curl http://localhost:8000/tools/list | grep my_custom_tool

# Get schema
curl http://localhost:8000/tools/my_custom_tool/schema | jq .

# Execute
curl -X POST http://localhost:8000/tools/execute \
-H "Content-Type: application/json" \
-d '{
  "tool_name": "my_custom_tool",
  "parameters": {
    "required_param": "test",
    "optional_number": 42,
    "choice_param": "option1"
  }
}' | jq .
```

### 进阶：会话感知工具

对于需要在多次执行之间维护状态的工具：

```python
class SessionAwareTool(Tool):
    def _get_metadata(self) -> ToolMetadata:
        return ToolMetadata(
            name="session_tool",
            session_aware=True,  # Enable session context
            ...
        )

    async def _execute_impl(
        self,
        session_context: Optional[Dict] = None,
        **kwargs
    ) -> ToolResult:
        # Access session data
        session_id = session_context.get("session_id") if session_context else None
        user_id = session_context.get("user_id") if session_context else None

        # Store/retrieve session-specific data
        # Example: Redis, database, in-memory cache

        return ToolResult(success=True, output={"session": session_id})
```

## 配置参考

### MCP 工具配置

```yaml
mcp_tools:
  tool_name:
    enabled: true                    # Required: Enable/disable tool
    url: "https://api.example.com"   # Required: HTTP endpoint
    func_name: "function_name"       # Required: Remote function name
    description: "Tool description"  # Required: LLM-visible description
    category: "data"                 # Required: Tool category
    cost_per_use: 0.001              # Required: Cost in USD
    parameters:                      # Required: Parameter definitions
      - name: "param1"
        type: "string"               # string|integer|float|boolean|array|object
        required: true
        description: "Param description"
        enum: ["val1", "val2"]       # Optional: Allowed values
        default: "val1"              # Optional: Default value
    headers:                         # Optional: HTTP headers
      X-API-Key: "your_api_key"      # Note: MCP does not expand env vars in headers
      # Prefer dynamic registration and pass secrets at runtime
```

### OpenAPI 工具配置

```yaml
openapi_tools:
  collection_name:
    enabled: true
    spec_url: "https://api.example.com/openapi.json"  # OR spec_inline
    auth_type: "none"                # none|api_key|bearer|basic
    auth_config:                     # Required if auth_type != "none"
      # For api_key:
      api_key_name: "X-API-Key"
      api_key_location: "header"     # header|query
      api_key_value: "$API_KEY"
      # For bearer:
      token: "$BEARER_TOKEN"
      # For basic:
      username: "$USERNAME"
      password: "$PASSWORD"
    category: "api"
    base_cost_per_use: 0.001
    rate_limit: 30                   # Requests per minute
    timeout_seconds: 30              # Request timeout
    max_response_bytes: 10485760     # Max response size (bytes)
    operations:                      # Optional: Filter operations
      - "operationId1"
      - "operationId2"
    tags:                            # Optional: Filter by tags
      - "tag1"
    base_url: "https://override.com" # Optional: Override spec base URL
```

### 环境变量

**MCP 配置：**

```bash
# Domain Security
MCP_ALLOWED_DOMAINS=localhost,127.0.0.1,api.example.com  # Or * for dev

# Circuit Breaker
MCP_CB_FAILURES=5                    # Failures before circuit opens
MCP_CB_RECOVERY_SECONDS=60           # Circuit open duration

# Request Limits
MCP_MAX_RESPONSE_BYTES=10485760      # 10MB default
MCP_RETRIES=3                        # Retry attempts
MCP_TIMEOUT_SECONDS=10               # Request timeout

# Registration Security
MCP_REGISTER_TOKEN=your_secret       # API registration protection
```

**OpenAPI 配置：**

```bash
# Domain Security
OPENAPI_ALLOWED_DOMAINS=*            # Comma-separated or * for dev
OPENAPI_MAX_SPEC_SIZE=5242880        # 5MB spec size limit
OPENAPI_FETCH_TIMEOUT=30             # Spec fetch timeout

# Request Behavior
OPENAPI_RETRIES=2                    # Retry attempts (default: 2). Set higher if needed
```

**工具专用 API Keys：**

```bash
# Add your tool API keys here
WEATHER_API_KEY=your_key
STOCK_API_KEY=your_key
GITHUB_TOKEN=ghp_xxxxx
PETSTORE_API_KEY=your_key

# ... add more as needed
```

## 测试与验证

### 健康检查

```bash
# Check service health
curl http://localhost:8081/health | jq .

# Check LLM service status
docker inspect shannon-llm-service-1 --format='{{.State.Health.Status}}'
```

### 列出工具

```bash
# All tools
curl http://localhost:8000/tools/list | jq .

# By category
curl "http://localhost:8000/tools/list?category=data" | jq .

# Exclude dangerous
curl "http://localhost:8000/tools/list?exclude_dangerous=true" | jq .

# List categories
curl http://localhost:8000/tools/categories | jq .
```

### 获取工具 Schema

```bash
# Single tool schema
curl http://localhost:8000/tools/my_tool/schema | jq .

# All schemas
curl http://localhost:8000/tools/schemas | jq .

# Tool metadata
curl http://localhost:8000/tools/my_tool/metadata | jq .
```

### 执行工具

**直接执行：**

```bash
curl -X POST http://localhost:8000/tools/execute \
-H "Content-Type: application/json" \
-d '{
  "tool_name": "calculator",
  "parameters": {"expression": "sqrt(144) + 2^3"}
}' | jq .
```

**批量执行：**

```bash
curl -X POST http://localhost:8000/tools/batch-execute \
-H "Content-Type: application/json" \
-d '[
  {"tool_name": "calculator", "parameters": {"expression": "2+2"}},
  {"tool_name": "calculator", "parameters": {"expression": "10*5"}}
]' | jq .
```

**通过 workflow：**

```bash
SESSION_ID="test-$(date +%s)" ./scripts/submit_task.sh "Calculate 2+2 and then multiply by 5"
```

### 监控日志

```bash
# Registration logs
docker compose logs llm-service | grep -i "registered tool"
docker compose logs llm-service | grep -i "loaded.*tools"

# Execution logs
docker compose logs -f llm-service orchestrator agent-core

# Tool-specific logs
docker compose logs llm-service | grep "my_tool"
```

### E2E 测试

```bash
# Run all tests
make smoke

# Run tool-specific tests
./tests/e2e/01_basic_calculator_test.sh
./tests/e2e/06_openapi_petstore_test.sh

# Run with MCP tools
./tests/e2e/run.sh
```

## 故障排查

### 工具未注册

**现象：** 工具没有出现在 `/tools/list` 中
**调试步骤：**

```bash
# 1. Check YAML syntax
yamllint config/shannon.yaml

# 2. Check logs for errors
docker compose logs llm-service | grep -i error

# 3. Verify enabled flag
grep -A 10 "my_tool" config/shannon.yaml | grep enabled

# 4. Force recreate service
docker compose -f deploy/compose/docker-compose.yml up -d --force-recreate llm-service

# 5. Wait for health
sleep 10
docker inspect shannon-llm-service-1 --format='{{.State.Health.Status}}'
```

### 域名校验错误

**现象：** `URL host 'example.com' not in allowed domains`
**解决方案：**

* **开发环境：** 使用通配符

  ```
  # .env
  MCP_ALLOWED_DOMAINS=*
  OPENAPI_ALLOWED_DOMAINS=*
  ```
* **生产环境：** 添加特定域名

  ```
  # .env
  MCP_ALLOWED_DOMAINS=localhost,127.0.0.1,api.example.com
  OPENAPI_ALLOWED_DOMAINS=api.example.com,api.github.com
  ```
* **Docker Compose：** 在 environment 中设置

  ```yaml
  services:
    llm-service:
      environment:
        - MCP_ALLOWED_DOMAINS=api.example.com
  ```

### 工具执行失败

**现象：** `ToolResult { success: false, error: "..." }`
**调试：**

```bash
# 1. Test tool directly
curl -X POST http://localhost:8000/tools/execute \
-H "Content-Type: application/json" \
-d '{"tool_name":"my_tool","parameters":{...}}' | jq .

# 2. Check parameter types
curl http://localhost:8000/tools/my_tool/schema | jq '.parameters'

# 3. Validate parameters match schema
# Required params must be provided
# Types must match (string vs integer)
# Enum values must be in allowed list

# 4. Check agent core logs
docker logs shannon-agent-core-1 | grep "Tool execution error"

# 5. Check LLM service logs
docker logs shannon-llm-service-1 | grep my_tool
```

### LLM 未选择工具

**现象：** LLM 没有在相关 query 中使用你的工具
**解决方案：**

* **改进 description：** 让它具体，并使用 LLM 友好的关键词

  ```
  # Bad
  description: "Weather tool"

  # Good
  description: "Get real-time weather forecast data for any city including temperature, humidity, and conditions. Use for queries about current or future weather."
  ```
* **添加到 decomposition context：**
  当前限制：Orchestrator 会将空 tools list 传递给 decomposition。Tools 默认不会出现在 system prompt 中。
  **快速修复：** Agent service 有自动加载 fallback（`agent.py` 中的 598-609 行）
  **正确修复：** 更新 orchestrator workflows，调用 `fetchAvailableTools()` 并传入 `AvailableTools` 字段：

  * `go/orchestrator/internal/workflows/orchestrator_router.go:80`
  * `go/orchestrator/internal/workflows/supervisor_workflow.go:273`
* **测试工具选择：**

  ```bash
  curl -X POST http://localhost:8000/tools/select \
  -H "Content-Type: application/json" \
  -d '{
    "task": "Get weather forecast for Tokyo",
    "max_tools": 3
  }' | jq .
  ```
* **显式提及：**

  ```bash
  ./scripts/submit_task.sh "Use the weather_forecast tool to get weather for Tokyo"
  ```

### 熔断器触发

**现象：** `Circuit breaker open for <url> (too many failures)`
**调试：**

```bash
# Check recent errors
docker logs shannon-llm-service-1 --tail 100 | grep -i "circuit\|failure"

# Wait for recovery (default 60s)
sleep 60

# Or restart to reset
docker compose restart llm-service
```

**预防：**

* 增加失败阈值：`MCP_CB_FAILURES=10`
* 增加恢复时间：`MCP_CB_RECOVERY_SECONDS=120`
* 修复底层 API 问题

### 超出速率限制

**现象：** `Rate limit exceeded for tool <name>`
**解决方案：**

```yaml
# Increase rate limit in config
# config/shannon.yaml
rate_limit: 120  # Increase from default 60

# Or in tool metadata (built-in tools)
ToolMetadata(
  rate_limit=120,  # Requests per minute
  ...
)
```

## 安全最佳实践

### 域名白名单

**开发环境：**

```bash
# Permissive for testing
MCP_ALLOWED_DOMAINS=*
OPENAPI_ALLOWED_DOMAINS=*
```

**预发环境：**

```bash
# Specific domains + localhost
MCP_ALLOWED_DOMAINS=localhost,127.0.0.1,staging-api.example.com
```

**生产环境：**

```bash
# Explicit allowlist only
MCP_ALLOWED_DOMAINS=api.example.com,api.partner.com
OPENAPI_ALLOWED_DOMAINS=api.github.com,api.openweathermap.org
```

**子域名匹配：**

* `api.example.com` 允许 `api.example.com` 和 `v1.api.example.com`
* 通配符 `*` 会绕过所有校验（请谨慎使用！）

### API Key 管理

**❌ 永远不要硬编码：**

```yaml
# BAD - Don't do this!
headers:
  X-API-Key: "sk-1234567890abcdef"
```

**✅ 使用环境变量：**

```yaml
# GOOD - Reference env vars
headers:
  X-API-Key: "${WEATHER_API_KEY}"
```

**存储在 `.env` 中（不要提交到 git）：**

```bash
# .env
WEATHER_API_KEY=sk-real-key-here
STOCK_API_KEY=your-stock-key
```

**生产环境：** 使用 secrets management（Docker、Kubernetes、HashiCorp Vault、AWS Secrets Manager）

### 速率限制

**每工具限制：**

```yaml
# MCP tools
mcp_tools:
  expensive_api:
    rate_limit: 10  # Low limit for expensive calls

# OpenAPI tools
openapi_tools:
  github:
    rate_limit: 60  # GitHub's actual limit
```

**全局限制：**

```bash
# .env
MCP_RATE_LIMIT_DEFAULT=60    # Default for all MCP tools
```

### 熔断器

**配置：**

```bash
# MCP circuit breaker
MCP_CB_FAILURES=5                 # Open after 5 failures
MCP_CB_RECOVERY_SECONDS=60        # Stay open for 60s

# Built-in per-base_url breakers
# - Prevents cascading failures
# - Isolates misbehaving services
# - Auto-recovery after timeout
```

### 认证

**MCP Registration API：**

```bash
# Require token for dynamic registration
MCP_REGISTER_TOKEN=your-secure-random-token

# Use in requests
curl -H "Authorization: Bearer your-secure-random-token" ...
# OR
curl -H "X-Admin-Token: your-secure-random-token" ...
```

**生成安全 tokens：**

```bash
openssl rand -hex 32
```

### 危险工具

标记会修改状态或访问敏感资源的工具：

```python
ToolMetadata(
    name="file_write",
    dangerous=True,        # Triggers OPA policy checks
    requires_auth=True,    # Requires user authentication
    ...
)
```

**OPA policies 随后可以限制访问：**

```rego
# config/opa/policies/tools.rego
package tools

deny[msg] {
  input.tool == "file_write"
  not is_admin(input.user)
  msg := "file_write requires admin role"
}
```

### 响应大小限制

**防止大响应造成 DoS：**

```yaml
# OpenAPI tools
max_response_bytes: 10485760  # 10MB default

# MCP tools (env var)
MCP_MAX_RESPONSE_BYTES=10485760
```

### 超时配置

**防止请求挂起：**

```yaml
# OpenAPI tools
timeout_seconds: 30  # Per-request timeout

# MCP tools (env var)
MCP_TIMEOUT_SECONDS=10
```

### HTTPS 强制

**对于非 localhost URLs，Shannon 强制使用 HTTPS：**

* ✅ `https://api.example.com` - 允许
* ✅ `http://localhost:8080` - 允许
* ❌ `http://api.example.com` - 生产环境拒绝

## 后续步骤

**探索进阶主题：**

* [Tools Implementation Guide](tools-implementation-guide.md) - 架构深度解析
* [MCP Integration](mcp-integration.md) - 完整 MCP 规范
* [Python WASI Execution](python-code-execution.md) - 沙箱化代码执行

**示例工具：**

* 内置工具：`python/llm-service/llm_service/tools/builtin/`
* MCP 示例：`config/shannon.yaml`（注释示例）
* OpenAPI 示例：`tests/e2e/06_openapi_petstore_test.sh`

**社区：**

* 报告问题：[https://github.com/anthropics/shannon/issues](https://github.com/anthropics/shannon/issues)
* 贡献工具：[https://github.com/anthropics/shannon/pulls](https://github.com/anthropics/shannon/pulls)

## 总结

**三种添加工具的方式：**

| 方法           | 命令                                                  | 配置文件                  | 代码变更     |
| ------------ | --------------------------------------------------- | --------------------- | -------- |
| **MCP**      | `docker compose up -d --force-recreate llm-service` | `config/shannon.yaml` | 无        |
| **OpenAPI**  | `docker compose up -d --force-recreate llm-service` | `config/shannon.yaml` | 无        |
| **Built-in** | `docker compose up -d --force-recreate llm-service` | `api/tools.py` + 新文件  | 仅 Python |

**核心要点：**

* ✅ 零 proto/Rust/Go 变更（通用 `google.protobuf.Struct` 容器）
* ✅ 内置安全能力（域名白名单、速率限制、熔断器）
* ✅ 自动成本跟踪（在 metadata 中设置 `cost_per_use`）
* ✅ Schema 驱动（OpenAI 兼容 JSON schemas）

**快速参考：**

```bash
# List tools
curl http://localhost:8000/tools/list

# Get schema
curl http://localhost:8000/tools/{name}/schema

# Execute
curl -X POST http://localhost:8000/tools/execute \
-d '{"tool_name":"my_tool","parameters":{...}}'

# Via workflow
./scripts/submit_task.sh "Your query here"
```

祝你构建工具顺利！🛠️

## Vendor Adapter Pattern

**用于特定领域 APIs 和自定义 agents**

使用 vendor adapters 将 vendor 逻辑从 Shannon core 中分离出来，适用于需要特定领域转换的专有或内部 APIs。

### 何时使用

当你的 API 集成需要以下能力时，使用 vendor adapters：

* 自定义字段名别名（例如 `users` → `my:unique_users`）
* 请求/响应转换
* 从 session context 动态注入参数
* 特定领域校验或标准化
* 带有自定义 system prompts 的专用 agent 角色

### 快速示例

**1. 创建 vendor adapter：**
`python/llm-service/llm_service/tools/vendor_adapters/myvendor.py`：

```python
class MyVendorAdapter:
    def transform_body(self, body, operation_id, prompt_params):
        # Transform field names
        if isinstance(body.get("metrics"), list):
            body["metrics"] = [m.replace("users", "my:users") for m in body["metrics"]]

        # Inject session params
        if prompt_params and "account_id" in prompt_params:
            body["account_id"] = prompt_params["account_id"]

        return body
```

**2. 注册 adapter：**
`python/llm-service/llm_service/tools/vendor_adapters/__init__.py`：

```python
def get_vendor_adapter(name: str):
    if name.lower() == "myvendor":
        from .myvendor import MyVendorAdapter
        return MyVendorAdapter()
    return None
```

**3. 使用 vendor 标记配置：**
`config/overlays/shannon.myvendor.yaml`：

```yaml
openapi_tools:
  myvendor_api:
    enabled: true
    spec_path: config/openapi_specs/myvendor_api.yaml
    auth_type: bearer
    auth_config:
      vendor: myvendor  # Triggers adapter loading
      token: "${MYVENDOR_API_TOKEN}"
    category: custom
```

**4. 使用环境：**

```bash
SHANNON_CONFIG_PATH=config/overlays/shannon.myvendor.yaml
MYVENDOR_API_TOKEN=your_token_here
```

### 优势

* ✅ **清晰隔离：** Vendor code 与 Shannon core 隔离
* ✅ **无需 core 变更：** Shannon infrastructure 保持通用
* ✅ **条件加载：** 如果 vendor module 不可用，会优雅降级
* ✅ **易于测试：** adapters 可独立进行单元测试
* ✅ **Secrets 管理：** 所有 tokens 通过环境变量传入

### 完整指南

获取包含以下内容的完整指南：

* 面向专用领域的自定义 agent roles
* Session context 注入模式
* 测试策略
* 最佳实践和故障排查

请参见：**[Vendor Adapters Guide](vendor-adapters.md)**
