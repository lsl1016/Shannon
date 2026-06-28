# 模板快速入门

本简短指南展示如何创建、加载和运行基于模板的工作流（系统 1）。

## 1) 创建模板

在 `config/workflows/examples/`（或您自己的文件夹）下创建 YAML 文件。最小示例：

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

## 2) 加载模板

模板在编排器启动时通过 `InitTemplateRegistry` 加载，可来自一个或多个目录。参见 `go/orchestrator/internal/workflows/template_catalog.go`。

## 3) 列出可用模板

使用新的 gRPC API：

```
grpcurl -plaintext -d '{}' localhost:50052 \
  shannon.orchestrator.OrchestratorService/ListTemplates
```

响应包含 `name`、`version`、`key` 和 `content_hash`。

## 4) 执行模板

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

## 5) 最佳实践

- 保持节点小而确定性；倾向于更多节点而非大型单体。
- 在每个节点显式限制工具。
- 当需要人工签批时设置 `defaults.require_approval`。
- 使用 `extends` 共享默认值；通过 `registry.Finalize()` 验证。
