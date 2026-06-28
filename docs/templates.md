# 模板

用于常见模式的确定性、零令牌工作流。本指南结合了入门步骤与更深层的架构和最佳实践。

---

## 快速入门

1) 在 `config/workflows/examples/`（或你的文件夹）下创建模板 YAML：

```yaml
name: simple_analysis
version: "1.0.0"
defaults:
  model_tier: medium
  budget_agent_max: 5000
  require_approval: false

nodes:
  - id: analyze
    type: simple
    strategy: react
    tools_allowlist: ["web_search"]
    budget_max: 500
    depends_on: []
```

提示：
- `type`: `simple | cognitive | dag | supervisor`
- `strategy`: `react | chain_of_thought | reflection | debate | tree_of_thoughts`
- 设置每个节点的 `budget_max` 和 `tools_allowlist` 以限制执行。

2) 启动时加载模板

通过 `InitTemplateRegistry` 从一个或多个目录加载：参见 `go/orchestrator/internal/workflows/template_catalog.go`。

3) 列出可用模板

```
grpcurl -plaintext -d '{}' localhost:50052 \
  shannon.orchestrator.OrchestratorService/ListTemplates
```

4) 执行模板

按名称/版本请求模板执行，并可选择禁用 AI：

```
grpcurl -plaintext -d '{
  "query": "总结本周的科技新闻",
  "context": {
    "template": "simple_analysis",
    "template_version": "1.0.0",
    "disable_ai": true
  }
}' localhost:50052 shannon.orchestrator.OrchestratorService/SubmitTask
```

注意：
- 当 `disable_ai` 为 true 且模板缺失时，请求快速失败。
- 当 `workflows.templates.fallback_to_ai`（或 `TEMPLATE_FALLBACK_ENABLED=1`）启用时，失败的模板运行可以回退到 AI 分解。

HTTP 用法：

```bash
curl -sS -X POST http://localhost:8080/api/v1/tasks \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "每周研究简报",
    "context": {
      "template": "simple_analysis",
      "template_version": "1.0.0",
      "disable_ai": true
    }
  }'
```

别名支持：调用 HTTP API 时，`context.template_name` 被接受为 `context.template` 的别名。

5) 最佳实践

- 保持节点小而确定性；倾向于更多节点而非大型单体。
- 在每个节点显式限制工具。
- 当需要人工签批时设置 `defaults.require_approval`。
- 使用 `extends` 共享默认值；注册表通过 `Finalize()` 验证。

---

## 概述与架构

模板提供零令牌路由（系统 1）以及智能 AI 分解（系统 2）：

```
┌─────────────────────────────────────────────────────────────┐
│                     用户查询                                  │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│              编排器路由器                                      │
│  1. 检查模板注册表是否有精确匹配                                  │
│  2. 咨询学习路由器以获取学习到的模式                              │
│  3. 如果需要，回退到 AI 分解                                   │
└────────────────┬────────────────────────────────────────────┘
                 │
        ┌────────┴────────┬─────────────┐
        ▼                 ▼             ▼
┌───────────────┐ ┌──────────────┐ ┌──────────────┐
│  模板         │ │  学习路由器   │ │   AI         │
│  工作流        │ │  (0 令牌)    │ │  分解        │
│  (0 令牌)     │ └──────────────┘ │  (完整成本)  │
└───────────────┘                  └──────────────┘
```

系统 1（模板）：确定性、版本门控的工作流；每个节点的预算与自动降级。
系统 2（学习路由器）：为重复模式推荐策略；可配置的探索。

---

## 模板结构

### 基本格式

```yaml
name: template_name
description: "它的作用"
version: "1.0.0"
defaults:
  budget_agent_max: 10000
  require_approval: false

nodes:
  - id: node_1
    type: simple                 # simple/cognitive/dag/supervisor
    strategy: react              # react/chain_of_thought/tree_of_thoughts/debate/reflection
    budget_max: 2000
    tools_allowlist:
      - web_search
      - calculator
    depends_on: []               # DAG 排序

  - id: node_2
    type: cognitive
    strategy: tree_of_thoughts
    budget_max: 5000
    depends_on: ["node_1"]

edges:
  - from: node_1
    to: node_2
```

在节点上使用 `tools_allowlist`。对于 `metadata.tasks` 内的混合 DAG 任务，`tools` 或 `tools_allowlist` 均可接受。

### 节点类型

Simple
- 单任务执行、直接工具调用、最小上下文。

Cognitive
- 复杂推理任务；多种模式（ReAct/CoT/ToT/等）。
- 自动基于预算的降级（见下文）——无需额外标志。

DAG
- 在一个节点中的并行执行分支和依赖关系。

Supervisor
- 聚合分支结果；协调最终综合。

---

## 模式降级

### 自动预算管理（当前）

模式通过 `budget_max` 使用阈值自动降级。示例：

```
Tree of Thoughts → Chain of Thought → ReAct
   (预算 ↓)           (预算 ↓)
```

运行时在节点元数据中标记降级（例如 `degraded_from`、`degraded_to`）。

### on_fail（已验证，使用有限）

YAML 支持带有 `degrade_to`、`retry` 和 `escalate_to` 的 `on_fail`。这些字段当前已验证，但运行时尚未完全强制执行。基于预算的降级是当前生效的机制。

```yaml
nodes:
  - id: analyze
    type: cognitive
    strategy: tree_of_thoughts
    budget_max: 5000
    on_fail:
      degrade_to: chain_of_thought  # 已验证；运行时强制执行待定
      retry: 1
```

---

## 学习路由器

当启用时（`continuous_learning.enabled: true` 或 `CONTINUOUS_LEARNING_ENABLED=1`），路由器将在回退到分解之前咨询策略推荐器。未知策略会被安全忽略。

---

## 注册表管理

模板加载

```bash
# 默认位置
/app/config/workflows/
/app/config/workflows/examples/

# 通过环境自定义
export TEMPLATES_PATH="/custom/templates:/shared/templates"
```

热重载（未来）

```yaml
registry:
  hot_reload: true
  reload_interval: 30s
  validation: strict
```

---

## 验证

规则
1. 结构：必需字段、有效 YAML
2. DAG：无循环、有效依赖
3. 预算：节点预算 ≤ 默认值
4. 工具：必须存在于注册表中
5. 引用：变量可解析

错误模型

```go
type ValidationIssue struct { Code, Message string }
type ValidationError struct { Issues []ValidationIssue }
```

---

## 示例

研究摘要

```yaml
name: research_summary
version: "1.0.0"
description: "研究一个主题并提供结构化摘要"

defaults:
  budget_agent_max: 10000
  require_approval: false

nodes:
  - id: discover
    type: simple
    strategy: react
    tools_allowlist: [web_search]
    budget_max: 1500

  - id: expand
    type: cognitive
    strategy: chain_of_thought
    tools_allowlist: [web_search, web_fetch]
    budget_max: 3000
    depends_on: ["discover"]

  - id: synthesize
    type: cognitive
    strategy: tree_of_thoughts
    budget_max: 5000
    depends_on: ["expand"]

edges:
  - from: discover
    to: expand
  - from: expand
    to: synthesize
```

复杂 DAG

```yaml
name: market_analysis
version: "1.0.0"
description: "并行市场研究与分析"

defaults:
  budget_agent_max: 15000
  require_approval: false

nodes:
  - id: competitors
    type: dag
    budget_max: 6000
    metadata:
      tasks:
        - id: competitor_1
          query: "分析竞争对手 A"
          tools_allowlist: [web_search]
        - id: competitor_2
          query: "分析竞争对手 B"
          tools_allowlist: [web_search]
        - id: competitor_3
          query: "分析竞争对手 C"
          tools_allowlist: [web_search]

  - id: market_trends
    type: cognitive
    strategy: chain_of_thought
    budget_max: 3000

  - id: synthesis
    type: supervisor
    budget_max: 6000
    depends_on: ["competitors", "market_trends"]
```

---

## API 参考

注册与列出

```go
// 从目录加载模板
registry, err := workflows.InitTemplateRegistry(logger, "/path/to/templates")

// 获取特定模板
entry, found := registry.Get("research_summary@1.0.0")

// 列出所有模板
summaries := registry.List()
```

通过 OrchestratorService 执行

```
grpcurl -plaintext -d '{
  "query": "总结量子计算",
  "context": { "template": "research_summary", "template_version": "1.0.0" }
}' localhost:50052 shannon.orchestrator.OrchestratorService/SubmitTask
```

---

## 故障排查

模板未加载
- 检查 `TEMPLATES_PATH` 环境变量
- 验证 YAML
- 检查编排器日志中的验证错误

预算超限
- 重新平衡节点预算
- 依赖自动模式降级
- 检查不必要的顺序依赖

学习路由器质量
- 临时增加探索率
- 如有需要重置过时模式
- 检查置信度阈值

启用调试日志

```bash
export DEBUG_TEMPLATES=true
export LOG_LEVEL=debug
docker compose logs orchestrator | rg -i template
```

---

## 未来

- 模板热重载
- 可视化编辑器/市场
- 模板变体的 A/B 测试
- 基于 ML 的预算分配
