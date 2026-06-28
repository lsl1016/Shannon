# HTTP 任务提交 API

本文档记录了通过 HTTP 网关提交任务的所有参数。

## 端点

- POST `/api/v1/tasks` — 提交任务
- POST `/api/v1/tasks/stream` — 提交并接收流式 URL（201）

响应头包含 `X-Workflow-ID` 和 `X-Session-ID`。

## 顶级请求体

- `query`（字符串，必需）— 用户查询或命令
- `session_id`（字符串，可选）— 会话连续性（UUID 或自定义）
- `context`（对象，可选）— 执行上下文（见下文）
- `mode`（字符串，可选）— `simple` | `standard` | `complex` | `supervisor` — 工作流路由
  - `simple`：直接到 SimpleTaskWorkflow（无分解）
  - `standard`：带标准复杂度提示的路由器
  - `complex`：带高复杂度提示的路由器
  - `supervisor`：直接到 SupervisorWorkflow（多代理）
  - 默认：基于查询复杂度自动检测
- `model_tier`（字符串，可选）— `small` | `medium` | `large`
  - 注入到 `context.model_tier` 中，由服务使用
- `model_override`（字符串，可选）— 特定模型名称（例如 `gpt-5-2025-08-07`、`MiniMax-M3`、`claude-sonnet-4-5-20250929`）
  - 顶级替代 `context.model_override`
- `provider_override`（字符串，可选）— 强制指定提供商（例如 `openai`、`anthropic`、`minimax`、`kimi`）
  - 顶级替代 `context.provider_override`

### Swarm 模型覆盖（通过 context）

对于 swarm 工作流，Lead 和 Worker 代理有单独的覆盖：

- `context.lead_model_override`（字符串）— Lead 代理的显式模型（例如 `kimi-k2.5`）
- `context.lead_provider_override`（字符串）— Lead 的显式提供商（例如 `kimi`）
- `context.agent_model_override`（字符串）— 所有 Worker 代理的显式模型（例如 `MiniMax-M3`）
- `context.agent_provider_override`（字符串）— Worker 的显式提供商（例如 `minimax`）

```bash
# 使用 MiniMax worker 的 Swarm
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"query": "比较 React 与 Vue", "context": {"force_swarm": true, "agent_model_override": "MiniMax-M3", "agent_provider_override": "minimax"}}'

# 使用 Kimi Lead + MiniMax worker 的 Swarm
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"query": "比较 React 与 Vue", "context": {"force_swarm": true, "lead_model_override": "kimi-k2.5", "lead_provider_override": "kimi", "agent_model_override": "MiniMax-M3", "agent_provider_override": "minimax"}}'
```

### 文件附件（通过 context）

附件在 `context.attachments` 中以 base64 编码对象形式发送：

```json
{
  "query": "分析这些数据",
  "session_id": "...",
  "context": {
    "attachments": [
      {
        "media_type": "image/png",
        "data": "<base64 编码内容>",
        "filename": "chart.png"
      },
      {
        "media_type": "text/csv",
        "data": "<base64 编码内容>",
        "filename": "sales.csv"
      }
    ]
  }
}
```

| 字段 | 类型 | 必需 | 描述 |
|-------|------|----------|-------------|
| `media_type` | 字符串 | 是 | MIME 类型：`image/png`、`application/pdf`、`text/csv`、`application/json` 等 |
| `data` | 字符串 | 是 | Base64 编码的文件内容（无 `data:` 前缀） |
| `filename` | 字符串 | 推荐 | 文件的显示名称 |

**支持的类型：** 图片（png/jpeg/gif/webp）、PDF、文本文件（txt/md/csv/html/json/xml/yaml、源代码）。
**限制：** HTTP 体 30MB，解码后附件总大小 20MB，按请求计。

### 研究策略控制（映射到 context）

这些可选字段由 Gateway 验证，然后添加到工作流 `context` 中：

- `research_strategy` — `quick | standard | deep | academic`
- `max_concurrent_agents` — 整数（1..20）
- `enable_verification` — 布尔值（当存在引用时启用声明验证）

#### 模型分层架构（成本优化）

研究策略使用分层模型架构以实现成本效率：

| 活动类型 | 模型层级 | 理由 |
|--------------|------------|-----------|
| 工具活动（覆盖率评估、事实提取、子查询生成） | small | 结构化输出任务 |
| 代理执行（quick） | small | 快速、廉价的研究 |
| 代理执行（standard） | medium | 质量与成本的平衡 |
| 代理执行（deep） | medium | 迭代细化弥补 |
| 代理执行（academic） | medium | 迭代细化弥补 |
| 最终综合 | large | 面向用户的质量至关重要 |

这种分层方法可将成本降低 50-70%，同时保持输出质量。参见 `config/research_strategies.yaml` 获取配置。

**注意**：`max_iterations` 参数为向后兼容而被网关接受，但当前工作流不使用。请使用 `context.react_max_iterations` 控制 ReAct 循环深度。

### 深度研究 2.0 控制

当设置 `context.force_research: true` 时，**默认启用**深度研究 2.0：

- `context.iterative_research_enabled` — 布尔值（默认：`true`）— 启用/禁用迭代覆盖率循环
- `context.iterative_max_iterations` — 整数（1-5，默认：`3`）— 覆盖率改进的最大迭代次数
- `context.enable_fact_extraction` — 布尔值（默认：`false`）— 将结构化事实提取到元数据中

```bash
# 基础深度研究 2.0（默认设置）
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"query": "2025 年 AI 趋势", "context": {"force_research": true}}'

# 自定义迭代次数
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"query": "比较 LLM", "context": {"force_research": true, "iterative_max_iterations": 2}}'

# 禁用 2.0（使用旧版）
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"query": "解释 ML", "context": {"force_research": true, "iterative_research_enabled": false}}'
```

### HITL 研究审查（人机回环）

在执行前启用研究计划的人工审查。启用后，工作流在生成研究计划后暂停，等待用户批准。

- `context.review_plan` — 字符串（`"auto"` | `"manual"`）— 审查模式
  - `"auto"`（默认）：研究在计划生成后立即执行
  - `"manual"`：工作流暂停进行用户审查；用户必须通过审查 API 批准
- `context.require_review` — 布尔值 — `review_plan: "manual"` 的替代（API 友好）
- `context.review_timeout` — 整数（秒，默认：`900`）— 用户批准的超时时间（最大 15 分钟）

**要求：** HITL 仅在同时设置 `context.force_research: true` 时适用。

```bash
# 启用 HITL 审查（桌面风格）
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "query": "研究量子计算趋势",
    "context": {
      "force_research": true,
      "review_plan": "manual"
    }
  }'

# 启用 HITL 审查（API 风格）
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "query": "分析 AI 市场动态",
    "context": {
      "force_research": true,
      "require_review": true,
      "review_timeout": 600
    }
  }'
```

**HITL 工作流：**
1. 以 `review_plan: "manual"` 或 `require_review: true` 提交任务
2. 工作流生成初始研究计划 → 发出 `RESEARCH_PLAN_READY` 事件
3. 用户通过审查 API 审查计划（见下文）
4. 用户可以提供反馈（最多 10 轮）或批准
5. 批准后 → 发出 `RESEARCH_PLAN_APPROVED` → 研究执行开始
6. 超时后 → 工作流以错误失败

反馈/批准端点请参见[审查 API](#review-api)。

### 研究策略预设

为研究工作流应用预设配置：

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "每周研究简报",
    "research_strategy": "deep",
    "max_concurrent_agents": 7,
    "enable_verification": true,
    "context": {
      "react_max_iterations": 4
    }
  }'
```

### 引用（可选，映射到 context）

- `context.enable_citations` — 布尔值开关，用于在非研究工作流中收集/集成引用
  - ReactWorkflow：选择加入。为 true 时，从工具输出（例如 web_search、web_fetch）收集引用，并在最终报告末尾附加来源部分。
  - DAGWorkflow：选择退出。默认启用；设为 false 以跳过引用收集和来源部分。
  - ResearchWorkflow：不变；始终在内部管理引用。

示例（在 React 中启用引用）：

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "解释 XKCD 风格的加密最佳实践",
    "mode": "standard",
    "context": {"enable_citations": true}
  }'
```

### 完整上下文示例

结合会话、模式、模型层级和自定义参数：

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "总结我们的 Q3 成果",
    "session_id": "sales-2025-q3",
    "mode": "supervisor",
    "model_tier": "large",
    "context": {
      "role": "analysis",
      "prompt_params": {"profile_id": "49598h6e", "current_date": "2025-10-25"}
    }
  }'
```

## 已识别的 `context.*` 键

- `role` — 角色预设（例如 `analysis`、`research`、`writer`）。当存在时，编排器绕过 `/agent/decompose` 并创建单子任务计划，让特定角色的代理处理任何内部多步骤/工具逻辑。
- `system_prompt` — 覆盖角色提示词；支持 `prompt_params` 中的 `${var}`
- `prompt_params` — 用于提示词/工具/适配器的任意参数
- `model_tier` — 当顶级未提供时的回退
- `model_override` — 特定模型（例如 `gpt-5-2025-08-07`、`gpt-5-nano-2025-08-07`、`gpt-5-pro-2025-10-06`）
  - 可在顶级或 context 中指定
  - 如果模型在主提供商上不可用，回退到下一个提供商
- `provider_override` — 强制特定提供商（例如 `openai`、`anthropic`、`google`）
  - 可在顶级或 context 中指定
  - 短路提供商选择逻辑
  - 如果提供商失败，回退到基于层级的选择
- `template` — 模板名称
- `template_version` — 模板版本（字符串）
- `template_name` — `template` 的别名（已接受）
- `disable_ai` — true 表示仅需要模板（无 AI 回退）
  - ⚠️ 不能与 `model_tier`、`model_override` 或 `provider_override` 组合（返回 400）
- 高级上下文窗口控制：
  - `history_window_size`、`use_case_preset`、`primers_count`、`recents_count`
  - `compression_trigger_ratio`、`compression_target_ratio`
- 深度研究 2.0 控制（当 `force_research: true` 时）：
  - `iterative_research_enabled` — 启用/禁用迭代循环（默认：`true`）
  - `iterative_max_iterations` — 最大迭代次数 1-5（默认：`3`）
  - `enable_fact_extraction` — 提取结构化事实（默认：`false`）

## 常见场景

### 全功能 AI 执行

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "分析八月网站流量趋势",
    "session_id": "analytics-session-123",
    "mode": "supervisor",
    "model_tier": "large",
    "context": {
      "role": "analysis",
      "system_prompt": "你是一位专注于网站分析的分析师。",
      "prompt_params": {
        "profile_id": "49598h6e",
        "aid": "fcb1cd29-9104-47b1-b914-31db6ba30c1a",
        "current_date": "2025-10-31"
      },
      "history_window_size": 75,
      "primers_count": 3,
      "recents_count": 20,
      "compression_trigger_ratio": 0.75,
      "compression_target_ratio": 0.375
    }
  }'
```

**参数注释：**
- `query`（必需）— 要执行的任务
- `session_id`（可选）— 多轮对话的会话 ID（省略则自动生成）
- `mode`（可选）— 执行模式："simple" 或 "supervisor"（默认：基于复杂度）
- `model_tier`（可选）— 模型大小："small"、"medium" 或 "large"（默认："small"）
- `context.role`（可选）— 角色预设名称（例如 "analysis"、"research"、"writer"）
- `context.system_prompt`（可选）— 自定义系统提示词（覆盖角色预设）
- `context.prompt_params`（可选）— 传递给工具/适配器的任意键值对
  - `profile_id`、`aid`、`current_date` 是**示例** — 使用你需要的任何键
- `context.history_window_size`（可选）— 最大对话历史消息数（默认：50）
- `context.primers_count`（可选）— 保留的早期消息数（默认：5）
- `context.recents_count`（可选）— 保留的最近消息数（默认：15）
- `context.compression_trigger_ratio`（可选）— 触发压缩的窗口百分比（默认：0.8）
- `context.compression_target_ratio`（可选）— 压缩到的窗口百分比（默认：0.5）

### 强制模型层级

顶级 `model_tier` 优先于 `context.model_tier`：

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{"query": "复杂分析", "model_tier": "large"}'
```

### 强制特定模型

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{"query": "编写计划", "context": {"model_override": "gpt-5-2025-08-07"}}'
```

### 仅模板执行（无 AI）

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "每周研究简报",
    "context": {
      "template": "research_summary",
      "template_version": "1.0.0",
      "disable_ai": true,
      "prompt_params": {"week": "2025-W44"}
    }
  }'
```

### Supervisor 模式

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{"query": "评估系统可靠性", "mode": "supervisor"}'
```

### 覆盖模型（顶级）

```bash
# 使用顶级覆盖强制特定模型
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{"query": "编写计划", "model_override": "gpt-5-2025-08-07"}'

# 使用特定 GPT‑5 模型的示例
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{"query": "分析数据", "model_override": "gpt-5-2025-08-07"}'
```

### 覆盖提供商

```bash
# 强制 OpenAI 提供商
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{"query": "数到 5", "provider_override": "openai"}'

# 通过 context 强制 Anthropic 提供商
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{"query": "解释量子计算", "context": {"provider_override": "anthropic"}}'
```

### 复杂模式与覆盖

```bash
# 组合模式、模型和提供商覆盖
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "综合市场分析",
    "mode": "complex",
    "model_override": "gpt-5-pro-2025-10-06",
    "provider_override": "openai"
  }'
```

## 验证与优先级规则

- `model_tier`：仅 `small|medium|large`（无效则 400）
- `mode`：仅 `simple|standard|complex|supervisor`（无效则 400）
- 顶级 `model_tier` 覆盖 `context.model_tier`
- 顶级 `model_override` 覆盖 `context.model_override`
- 顶级 `provider_override` 覆盖 `context.provider_override`
- `template_name` 作为 `template` 的别名被接受
- **冲突验证**：`disable_ai: true` 不能与以下组合：
  - `model_tier`（顶级或 context）
  - `model_override`（顶级或 context）
  - `provider_override`（顶级或 context）
  - 检测到冲突时 Gateway 返回 400 及错误消息

## 任务状态响应示例

提交任务后，轮询 `GET /api/v1/tasks/{id}`。完成时的典型响应结构：

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
  },
  "metadata": {
    "model_breakdown": [
      {
        "model": "gpt-5-mini-2025-08-07",
        "provider": "openai",
        "executions": 5,
        "tokens": 300,
        "cost_usd": 0.006,
        "percentage": 100
      }
    ]
  }
}
```

### 模型分解（多模型任务）

对于使用多个模型的复杂任务（例如带有代理执行 + 综合的深度研究），`metadata.model_breakdown` 提供详细的每个模型使用情况：

```json
{
  "model_used": "claude-haiku-4-5-20251001",
  "metadata": {
    "model_breakdown": [
      {
        "model": "claude-haiku-4-5-20251001",
        "provider": "anthropic",
        "executions": 52,
        "tokens": 254324,
        "cost_usd": 0.060671,
        "percentage": 54
      },
      {
        "model": "gpt-5.1",
        "provider": "openai",
        "executions": 1,
        "tokens": 11628,
        "cost_usd": 0.052037,
        "percentage": 46
      }
    ]
  }
}
```

**注意**：`model_used` 显示最常用的模型，而 `model_breakdown` 提供任务执行期间使用的所有模型的完整可见性。这对于理解研究工作流中的成本分布特别有用，其中综合使用的层级（更大）与代理执行不同。为向后兼容，旧字段如 `agent_usages` 可能仍出现在元数据中，但它们不完整；客户应优先使用 `model_breakdown` 进行任何使用量或成本分析。

## ⚠️ 避免的常见错误

**1. 不要同时使用 `template` 和 `template_name`**
```json
// ❌ 错误：冗余别名
{"context": {"template": "research_summary", "template_name": "research_summary"}}

// ✅ 正确：仅使用 template
{"context": {"template": "research_summary"}}
```

**2. 不要将 `disable_ai: true` 与模型参数组合（Gateway 返回 400）**
```json
// ❌ 错误：冲突 - gateway 以 HTTP 400 拒绝
{"context": {"disable_ai": true, "model_tier": "large"}}
{"model_tier": "large", "context": {"disable_ai": true}}
{"model_override": "gpt-5-2025-08-07", "context": {"disable_ai": true}}
{"context": {"disable_ai": true, "provider_override": "openai"}}

// ✅ 正确：仅模板执行（无模型参数）
{"context": {"template": "summary", "disable_ai": true}}

// ✅ 正确：带模型控制的 AI 执行（无 disable_ai）
{"model_tier": "large", "model_override": "gpt-5-2025-08-07"}
{"context": {"provider_override": "openai"}}
```

**3. 不要重复 `model_tier`（顶级优先）**
```json
// ❌ 错误：重复造成混淆（context.model_tier 被忽略）
{"model_tier": "large", "context": {"model_tier": "small"}}

// ✅ 正确：仅在顶级指定一次
{"model_tier": "large"}

// ✅ 正确：或仅在 context 中指定（作为回退）
{"context": {"model_tier": "large"}}
```

## 注意

### 模型选择行为
- `model_override` 按名称选择特定模型
- `provider_override` 强制提供商选择（短路基于层级的路由）
- 当两者都指定时：使用指定提供商的指定模型
- 如果主提供商失败，回退到下一个提供商

### 模型命名
仅使用规范模型名称（无旧别名）。如果所选模型在所选提供商上不可用，则应用基于层级的回退。

### 回退行为
- 如果指定模型在请求的提供商上不可用 → 尝试下一个提供商
- 如果指定提供商失败 → 回退到基于层级的选择
- 系统优先保证任务完成，而非严格遵循参数

### 附加功能
- 使用头部 `Idempotency-Key` 安全重试提交；gateway 缓存 2xx 响应 24 小时。
- SSE 流式端点由 `POST /api/v1/tasks/stream` 返回。
- 所有上下文参数均为可选；未指定时应用默认值。

### 响应格式
- **Gateway API 响应**：`/api/v1/tasks/{task_id}` 返回包含原始 LLM 响应的 `result` 字段
  - `result` 字段包含纯文本或 JSON 字符串响应
  - 如果结果是有效 JSON，可选的 `response` 字段包含解析后的 JSON（用于向后兼容）

---

## 审查 API

审查 API 支持人机回环（HITL）与研究计划生成的交互。这些端点仅在任务以 `review_plan: "manual"` 或 `require_review: true` 提交时可用。

### 获取审查状态

检索当前审查对话状态。

```
GET /api/v1/tasks/{workflowID}/review
```

**头部：**
- `Authorization: Bearer <token>`（生产环境必需）

**响应：**
```json
{
  "status": "reviewing",
  "round": 2,
  "version": 3,
  "current_plan": "研究计划内容...",
  "query": "原始查询",
  "rounds": [
    {"role": "assistant", "message": "初始计划...", "timestamp": "2025-01-29T10:00:00Z"},
    {"role": "user", "message": "你能更多关注 X 吗？", "timestamp": "2025-01-29T10:01:00Z"},
    {"role": "assistant", "message": "更新后的计划...", "timestamp": "2025-01-29T10:01:30Z"}
  ]
}
```

**响应头部：**
- `ETag`：用于乐观并发的版本号

### 提交反馈或批准

提交用户反馈以细化计划，或批准以开始研究执行。

```
POST /api/v1/tasks/{workflowID}/review
```

**头部：**
- `Authorization: Bearer <token>`（生产环境必需）
- `If-Match: <version>`（可选，用于乐观并发）
- `Content-Type: application/json`

**请求体：**
```json
{
  "action": "feedback",
  "message": "请更多关注 2025 年的近期发展"
}
```

| 字段 | 类型 | 必需 | 描述 |
|-------|------|----------|-------------|
| `action` | 字符串 | 是 | `"feedback"` 或 `"approve"` |
| `message` | 字符串 | 对于 feedback | 用户的反馈消息 |

**反馈响应：**
```json
{
  "status": "reviewing",
  "plan": {
    "message": "基于您的反馈更新后的研究计划...",
    "round": 3,
    "version": 4,
    "intent": "ready"
  }
}
```

**批准响应：**
```json
{
  "status": "approved",
  "message": "研究已开始"
}
```

### Intent 值

反馈响应中的 `intent` 字段表示 LLM 的评估：

| Intent | 描述 |
|--------|-------------|
| `feedback` | LLM 正在询问澄清问题；计划未就绪 |
| `ready` | LLM 提出具体计划；用户可以批准或细化 |
| `execute` | 用户的消息明确表示批准（少见） |

### 错误响应

| 状态 | 描述 |
|--------|-------------|
| `404` | 审查会话未找到或已过期 |
| `403` | 不是任务所有者 |
| `409` | 版本冲突（过时的 `If-Match`）或已达到最大轮次（10） |
| `502` | LLM 服务不可用 |

### 示例：完整 HITL 流程

```bash
# 1. 启用 HITL 提交任务
RESPONSE=$(curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{"query": "研究 AI 安全趋势", "context": {"force_research": true, "review_plan": "manual"}}')
WORKFLOW_ID=$(echo $RESPONSE | jq -r '.workflow_id')

# 2. 通过 SSE 等待 RESEARCH_PLAN_READY 事件
curl -N "http://localhost:8081/stream/sse?workflow_id=$WORKFLOW_ID&types=RESEARCH_PLAN_READY"

# 3. 获取当前审查状态
curl -sS "http://localhost:8080/api/v1/tasks/$WORKFLOW_ID/review"

# 4. 提供反馈
curl -sS -X POST "http://localhost:8080/api/v1/tasks/$WORKFLOW_ID/review" \
  -H "Content-Type: application/json" \
  -d '{"action": "feedback", "message": "更多关注对齐研究"}'

# 5. 批准计划
curl -sS -X POST "http://localhost:8080/api/v1/tasks/$WORKFLOW_ID/review" \
  -H "Content-Type: application/json" \
  -d '{"action": "approve"}'

# 6. 研究执行开始，通过 SSE 监控
curl -N "http://localhost:8081/stream/sse?workflow_id=$WORKFLOW_ID"
```
