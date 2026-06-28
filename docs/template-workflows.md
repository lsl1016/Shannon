# 模板工作流文档

## 概述

Shannon 的模板工作流系统通过 YAML 定义的工作流，为常见模式实现零令牌路由。这种双系统架构将确定性模板执行（系统 1）与智能 AI 驱动的分解（系统 2）相结合，在重复任务上实现 85-95% 的令牌节省。

## 架构

### 双系统设计

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

### 系统 1：基于模板的执行
- **零令牌路由**：无需 LLM 调用的直接模板匹配
- **确定性执行**：可预测、可重复的工作流
- **版本门控**：Temporal 工作流版本控制确保向后兼容
- **预算感知**：每个节点的令牌限制，带自动降级

### 系统 2：学习路由器
- **模式识别**：学习成功的分解模式
- **Epsilon-Greedy 策略**：90% 利用已知模式，10% 探索新方法
- **监督器内存**：存储和检索已验证的策略
- **持续改进**：根据结果更新策略有效性

## 模板结构

### 基本模板格式

```yaml
name: template_name           # 唯一标识符
description: "它的作用"        # 人类可读的描述
version: "1.0.0"              # 语义化版本
defaults:
  budget_agent_max: 10000     # 默认每个节点的代理预算
  require_approval: false     # 可选：是否需要人工签批

nodes:
  - id: node_1
    type: simple              # 节点类型（simple/cognitive/dag/supervisor）
    strategy: react           # 覆盖该节点的策略
    budget_max: 2000          # 节点特定预算
    tools_allowlist:          # 该节点允许的工具
      - web_search
      - calculator
    depends_on: []            # 节点依赖（用于 DAG 排序）

  - id: node_2
    type: cognitive
    strategy: tree_of_thoughts
    budget_max: 5000
    depends_on: ["node_1"]    # 通过 depends_on 引用先前输出

edges:
  - from: node_1
    to: node_2
```

### 节点类型

#### Simple 节点
- 单任务执行
- 直接工具调用
- 所需上下文最少

```yaml
- id: fetch_data
  type: simple
  strategy: react
  tools_allowlist: [weather_api]
  budget_max: 500
  metadata:
    query: "获取当前天气数据"
```

#### Cognitive 节点
- 复杂推理任务
- 多步骤分析
- 预算感知的自动降级（无需额外字段）

```yaml
- id: analyze
  type: cognitive
  strategy: chain_of_thought
  budget_max: 3000
  metadata:
    query: "分析趋势并提供见解"
  # 注意：预算导致的降级自动发生；无需 pattern_degrade 标志。
```

#### DAG 节点
- 并行执行分支
- 复杂依赖图
- 针对并发操作优化

```yaml
- id: parallel_research
  type: dag
  depends_on: [previous_node]
  budget_max: 4000
  metadata:
    tasks:
      - id: search_1
        query: "研究主题 A"
        tools: [web_search]
      - id: search_2
        query: "研究主题 B"
        depends_on: [search_1]  # DAG 内的依赖
        tools: [web_search, web_fetch]
```

**注意**：DAG 子任务定义在 `metadata.tasks` 中，而非顶层的 `sub_nodes` 字段。每个任务可以有自己的 `depends_on`，以在并行执行中创建复杂的依赖图。

#### Supervisor 节点
- 层次化任务分解
- 子代理协调
- 自适应执行规划

```yaml
- id: complex_project
  type: supervisor
  strategy: reflection
  budget_max: 8000
  metadata:
    query: "协调多阶段项目"
    sub_agents: ["research_agent", "analysis_agent", "synthesis_agent"]
```

### 模板继承
- 复用已建立的模板，仅覆盖变更部分
- 通过 `extends` 指定父模板（合并时顺序重要）
- 从父模板继承默认值、节点、边和元数据

```yaml
name: research_summary_executive
extends:
  - research_summary

defaults:
  model_tier: large
  require_approval: true

nodes:
  - id: explore
    budget_max: 4000
    metadata:
      exploration_depth: deep
  - id: finalize
    metadata:
      summary_style: executive
```

允许多个父模板：

```yaml
extends:
  - research_summary
  - research_summary_data_appendix
```

父模板按列出的顺序应用，派生模板最后合并。除非派生模板提供自己的边列表，否则父模板的边保留。

### 策略类型

#### ReAct（推理 + 行动）
- **令牌使用**：每周期约 500-1500
- **最佳场景**：基于工具的简单任务
- **降级**：最终回退选项

```yaml
strategy: react
```

#### Chain of Thought（CoT）
- **令牌使用**：每次分析约 1500-3000
- **最佳场景**：逐步推理
- **降级**：预算受限时从 ToT 降级

```yaml
strategy: chain_of_thought
```

#### Tree of Thoughts（ToT）
- **令牌使用**：每次探索约 3000-8000
- **最佳场景**：复杂问题求解
- **降级**：到 CoT，再到 ReAct

```yaml
strategy: tree_of_thoughts
metadata:
  branches: 3
  depth: 2
```

## 模式降级

### 自动预算管理

系统在检测到预算约束时自动降级执行模式：

```
Tree of Thoughts（8000 令牌）
       ↓ （预算 < 5000）
Chain of Thought（3000 令牌）
       ↓ （预算 < 2000）
ReAct（1000 令牌）
```

### on_fail（已验证；运行时使用有限）

YAML 支持带有 `degrade_to`、`retry` 和 `escalate_to` 的 `on_fail`。这些字段当前已验证，但基于预算的降级是当前生效的机制：

```yaml
nodes:
  - id: analyze
    type: cognitive
    strategy: tree_of_thoughts
    budget_max: 5000
    on_fail:
      degrade_to: chain_of_thought  # 回退策略
      retry: 1                       # 重试次数
```

## 学习路由器

### 策略推荐

学习路由器维护成功模式的历史记录：

```go
type StrategyRecommendation struct {
    Pattern     string        // 查询模式签名
    Strategy    StrategyType  // 推荐的策略
    Confidence  float64       // 成功率（0-1）
    TokenSaved  int          // 平均节省的令牌数
}
```

### Epsilon-Greedy 选择

```yaml
learning_router:
  enabled: true
  epsilon: 0.1               # 10% 探索率
  min_confidence: 0.7        # 使用学习模式的最低置信度
  history_size: 1000         # 记住的模式数量
```

## 注册表管理

### 模板加载

模板在启动时从配置的目录加载：

```bash
# 默认位置
/app/config/workflows/
/app/config/workflows/examples/

# 通过环境自定义
export TEMPLATES_PATH="/custom/templates:/shared/templates"
```

### 热重载（未来）

```yaml
registry:
  hot_reload: true           # 启用文件监视
  reload_interval: 30s       # 检查间隔
  validation: strict         # 验证级别
```

## 验证

### 模板验证规则

1. **结构验证**：必需字段、有效 YAML
2. **DAG 验证**：无循环、有效依赖
3. **预算验证**：节点预算 ≤ 总预算
4. **工具验证**：工具存在且可用
5. **引用验证**：变量引用可解析

### 验证错误

```go
type ValidationIssue struct {
    Code    string
    Message string
}

type ValidationError struct {
    Issues []ValidationIssue
}
```

## 示例

### 研究摘要模板

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

### 复杂 DAG 工作流

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
          tools: [web_search]
        - id: competitor_2
          query: "分析竞争对手 B"
          tools: [web_search]
        - id: competitor_3
          query: "分析竞争对手 C"
          tools: [web_search]

  - id: market_trends
    type: cognitive
    strategy: chain_of_thought
    budget_max: 3000

  - id: synthesis
    type: supervisor
    budget_max: 6000
    depends_on: ["competitors", "market_trends"]
```

### 派生模板：高管研究

```yaml
name: research_summary_executive
description: "输出简短的高管版本"
extends:
  - research_summary

defaults:
  model_tier: large
  budget_agent_max: 18000
  require_approval: true

nodes:
  - id: explore
    budget_max: 4000
    metadata:
      exploration_depth: deep
  - id: finalize
    metadata:
      summary_style: executive_brief
      include_risks: true
```

### 带组合的 Playbook 工作流

```yaml
name: market_analysis_playbook
description: "带合规性和模板化报告的市场分析"
extends:
  - market_analysis

defaults:
  require_approval: true

nodes:
  - id: compliance_review
    type: simple
    strategy: react
    depends_on: [synthesize]
    tools_allowlist: [policy_checker]
    budget_max: 1200
  - id: report
    metadata:
      report_template: playbook_v1

edges:
  - from: synthesize
    to: compliance_review
  - from: compliance_review
    to: report
```

## 最佳实践

### 模板设计

1. **从简单开始**：从基础模板开始，逐步增加复杂度
2. **设置现实的预算**：监控实际使用情况并调整
3. **合理使用依赖**：最小化顺序依赖以获得更好的并行性
4. **启用降级**：允许模式降级以获得预算灵活性
5. **谨慎版本管理**：对破坏性变更使用语义化版本

### 性能优化

1. **并行执行**：对独立任务使用 DAG 节点
2. **工具白名单**：仅限制为必要的工具
3. **最小化引用**：仅传递节点之间所需的数据
4. **预算分配**：为复杂推理节点分配更多预算

### 监控

1. **令牌使用**：跟踪实际使用量与预算使用量
2. **降级频率**：监控模式降级率
3. **成功率**：衡量模板完成率
4. **学习效果**：跟踪路由器预测准确性

## API 参考

### 模板注册

```go
// 从目录加载模板
registry, err := workflows.InitTemplateRegistry(logger, "/path/to/templates")

// 获取特定模板
entry, found := registry.Get("research_summary@1.0.0")

// 列出所有模板
summaries := registry.List()
```

### 执行

使用 OrchestratorService SubmitTask API，在请求上下文中指定模板。路由器将确定性地执行 TemplateWorkflow：

```
grpcurl -plaintext -d '{
  "query": "总结量子计算",
  "context": { "template": "research_summary", "template_version": "1.0.0" }
}' localhost:50052 shannon.orchestrator.OrchestratorService/SubmitTask
```

### 监控

操作指标（执行计数、令牌、成功率）通过 Prometheus 兼容端点导出。参见代码库中的指标命名（例如 internal/metrics）。

## 迁移指南

### 转换现有工作流

1. **识别模式**：查找重复的任务结构
2. **提取模板**：创建 YAML 定义
3. **本地测试**：部署前验证模板
4. **监控性能**：比较前后的令牌使用量
5. **迭代**：根据实际使用情况优化

### 回滚策略

模板在 Temporal 工作流中具有版本门控：

```go
version := workflow.GetVersion(ctx, "template_router_v1",
    workflow.DefaultVersion, 1)
if version >= 1 {
    // 使用模板系统
} else {
    // 使用旧版路由
}
```

## 故障排查

### 常见问题

#### 模板未加载
- 检查 `TEMPLATES_PATH` 环境变量
- 使用验证器验证 YAML 语法
- 检查容器日志中的验证错误

#### 预算超限
- 审查节点预算分配
- 启用模式降级
- 检查 DAG 中是否存在无限循环

#### 学习路由器性能不佳
- 临时增加探索率
- 从内存中清除过时模式
- 检查置信度阈值

### 调试日志

```bash
# 启用调试日志
export DEBUG_TEMPLATES=true
export LOG_LEVEL=debug

# 查看模板加载
docker compose logs orchestrator | grep -i template

# 监控执行
docker compose exec orchestrator cat /var/log/templates.log
```

## 未来增强

### 计划功能

1. **热重载**：无需重启的动态模板更新
2. **模板市场**：跨组织共享模板
3. **可视化编辑器**：基于 Web 的模板设计器
4. **A/B 测试**：自动性能比较
5. **成本优化**：基于 ML 的预算分配

### 实验功能

1. **自适应预算**：基于复杂度的动态预算调整
2. **模板组合**：将模板组合成更大的工作流
3. **条件分支**：复杂的 if/then/else 逻辑
4. **外部触发器**：事件驱动的模板执行
