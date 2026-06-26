# 浏览器自动化 API 指南

本指南说明如何通过 API 使用 Shannon 的浏览器自动化工具。

## 快速开始

在上下文中指定 `role: "browser_use"` 来提交任务：

```bash
curl -X POST https://api.shannon.run/api/v1/tasks \
  -H "Content-Type: application/json" \
  -H "X-API-Key: YOUR_API_KEY" \
  -d '{
    "query": "导航到 https://example.com 并提取主标题",
    "session_id": "my-session-123",
    "context": {
      "role": "browser_use"
    }
  }'
```

## 工作原理

当指定 `role: "browser_use"` 时：

1. **React 工作流**：任务被路由到多轮 React 工作流（非单次调用）
2. **迭代执行**：代理可以按顺序进行多次工具调用
3. **会话持久化**：浏览器会话在同一任务内的多次工具调用之间保持
4. **自动清理**：任务完成后会话会被清理（5 分钟 TTL）

## API 参考

### 提交任务

```
POST /api/v1/tasks
```

**请求体：**

```json
{
  "query": "你的浏览器自动化指令",
  "session_id": "unique-session-id",
  "context": {
    "role": "browser_use"
  }
}
```

**响应：**

```json
{
  "task_id": "task-xxx-1234567890",
  "status": "STATUS_CODE_OK",
  "message": "任务提交成功"
}
```

### 获取任务状态

```
GET /api/v1/tasks/{task_id}
```

**响应（已完成）：**

```json
{
  "task_id": "task-xxx-1234567890",
  "status": "TASK_STATUS_COMPLETED",
  "result": "提取的内容或摘要...",
  "metadata": {
    "iterations": 3,
    "actions": 4,
    "observations": 3
  }
}
```

### 流式获取任务事件（SSE）

```
GET /api/v1/tasks/{task_id}/events
```

事件包括：
- `WORKFLOW_STARTED` - 任务处理开始
- `ROLE_ASSIGNED` - browser_use 角色已激活
- `AGENT_STARTED` - 代理迭代开始
- `TOOL_STARTED` / `TOOL_COMPLETED` - 工具执行
- `AGENT_COMPLETED` - 代理迭代完成
- `WORKFLOW_COMPLETED` - 任务完成

## 浏览器工具

一个带有 `action` 参数的统一 `browser` 工具，处理所有浏览器自动化操作。

### 操作

| 操作 | 描述 | 必需参数 | 可选参数 |
|--------|-------------|----------------|-----------------|
| `navigate` | 导航到 URL | `url` | `wait_until`、`timeout_ms` |
| `click` | 点击元素 | `selector` | `button`、`click_count`、`timeout_ms` |
| `type` | 在输入框中输入文本 | `selector`、`text` | `timeout_ms` |
| `screenshot` | 截取页面图片 | — | `full_page` |
| `extract` | 获取页面/元素内容 | — | `selector`、`extract_type`、`attribute` |
| `scroll` | 滚动页面或元素 | — | `selector`、`x`、`y` |
| `wait` | 等待元素出现或持续指定时间 | — | `selector`、`timeout_ms` |
| `close` | 结束浏览器会话 | — | — |

注意：代码中存在 `evaluate` 操作（执行 JavaScript），但已完全禁用——不会在工具模式中公布，且会被会话上下文清理器阻止。如需启用，请在 `api/tools.py` 和 `api/agent.py` 中将 `allow_browser_evaluate` 添加到 safe_keys 中。

## 使用示例

### 1. 读取并总结网页

```json
{
  "query": "阅读 https://example.com/article 并总结要点",
  "session_id": "reader-001",
  "context": {
    "role": "browser_use"
  }
}
```

### 2. 截取屏幕截图

```json
{
  "query": "截取 https://example.com 的完整页面截图",
  "session_id": "screenshot-001",
  "context": {
    "role": "browser_use"
  }
}
```

### 3. 填写并提交表单

```json
{
  "query": "前往 https://example.com/contact，将姓名填写为 'John Doe'，邮箱填写为 'john@example.com'，然后提交表单",
  "session_id": "form-001",
  "context": {
    "role": "browser_use"
  }
}
```

### 4. 提取特定数据

```json
{
  "query": "导航到 https://example.com/products 并提取所有产品名称和价格",
  "session_id": "scrape-001",
  "context": {
    "role": "browser_use"
  }
}
```

### 5. 多步骤交互

```json
{
  "query": "前往 https://example.com，点击 '了解更多' 按钮，等待页面加载，然后提取主文章的内容",
  "session_id": "multistep-001",
  "context": {
    "role": "browser_use"
  }
}
```

## 前端集成示例

### React/TypeScript

```typescript
interface BrowserTaskRequest {
  query: string;
  session_id: string;
  context: {
    role: 'browser_use';
  };
}

interface TaskResponse {
  task_id: string;
  status: string;
  result?: string;
  metadata?: {
    iterations: number;
    actions: number;
  };
}

async function submitBrowserTask(query: string): Promise<string> {
  const response = await fetch('/api/v1/tasks', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': API_KEY,
    },
    body: JSON.stringify({
      query,
      session_id: `browser-${Date.now()}`,
      context: { role: 'browser_use' },
    }),
  });

  const data = await response.json();
  return data.task_id;
}

async function pollTaskStatus(taskId: string): Promise<TaskResponse> {
  const response = await fetch(`/api/v1/tasks/${taskId}`, {
    headers: { 'X-API-Key': API_KEY },
  });
  return response.json();
}

// 使用 SSE 实现实时更新
function streamTaskEvents(taskId: string, onEvent: (event: any) => void) {
  const eventSource = new EventSource(
    `/api/v1/tasks/${taskId}/events?api_key=${API_KEY}`
  );

  eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    onEvent(data);

    if (data.type === 'WORKFLOW_COMPLETED' || data.type === 'STREAM_END') {
      eventSource.close();
    }
  };

  return eventSource;
}
```

## 最佳实践

1. **使用描述性查询**：明确说明你希望浏览器做什么
   - 好的示例："导航到 https://example.com，点击登录按钮，并提取表单字段"
   - 不好的示例："检查网站"

2. **处理超时**：浏览器操作可能需要时间。设置适当的超时时间（建议 30-60 秒）

3. **会话 ID**：为独立的浏览器会话使用唯一的会话 ID

4. **错误处理**：检查任务状态以发现失败并妥善处理

5. **使用 SSE 实现实时更新**：在 UI 中使用 SSE 流式传输获取实时进度更新

## 响应元数据

已完成任务中的 `metadata` 字段包括：

| 字段 | 描述 |
|-------|-------------|
| `iterations` | React 循环迭代次数 |
| `actions` | 工具调用总次数 |
| `observations` | 观察到的工具结果 |
| `thoughts` | 推理步骤 |
| `model` | 所使用的 LLM 模型 |
| `cost_usd` | 预估 API 成本 |

## 限制

- **会话 TTL**：浏览器会话在 5 分钟无活动后过期
- **最大会话数**：每个服务实例最多 50 个并发会话
- **页面渲染**：JavaScript 密集型页面可能需要使用 `browser(action="wait")` 来处理动态内容
- **身份验证**：对于需要登录的网站，请在查询中包含完整的登录流程
