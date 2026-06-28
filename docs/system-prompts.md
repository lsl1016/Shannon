# Shannon 中的系统提示词

本指南基于当前代码，说明 Shannon 如何处理系统提示词以及如何自定义代理行为。

---

## 概述

Shannon 使用**基于角色的预设系统**为代理分配系统提示词。您可以通过 API 在运行时覆盖系统提示词，以获得最大的灵活性。

当前状态：
- 角色预设已定义并使用：`python/llm-service/llm_service/roles/presets.py`。
- Persona 配置存在但未接入：`config/personas.yaml`。
- API 可以通过 `context["system_prompt"]` 覆盖系统提示词。

---

## 系统提示词优先级

当调用代理时，系统提示词按以下顺序确定（从高到低）：

```
1. context["system_prompt"]     ← API 覆盖（最高优先级）
2. context["role"]               ← 角色预设查找
3. "You are a helpful AI assistant."  ← 默认回退
```

实现在：`python/llm-service/llm_service/api/agent.py`。

---

## 角色预设（当前活动的系统）

### 可用角色

| 角色 | 系统提示词 | 最大令牌数 | Temperature | 允许的工具 |
|------|---------------|------------|-------------|---------------|
| `generalist` | 乐于助人的 AI 助手 | 1200 | 0.7 | （无） |
| `analysis` | 具有结构化推理能力的分析助手 | 1200 | 0.2 | web_search、code_reader |
| `research` | 收集事实的研究助手 | 1600 | 0.3 | web_search |
| `writer` | 技术写作人员 | 1800 | 0.6 | code_reader |
| `critic` | 批判性审查员 | 800 | 0.2 | code_reader |

**来源：** `python/llm-service/llm_service/roles/presets.py`

### 使用角色预设

在调用 LLM 服务 HTTP API 时，在 `context` 中传递角色名称：

```bash
curl -sS -X POST http://localhost:8000/agent/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "分析这段代码的性能",
    "context": {"role": "analysis"}
  }'
```

---

## API 系统提示词覆盖

通过在 `context` 中传递 `system_prompt` 来完全覆盖系统提示词：

调用 LLM 服务 HTTP API，并在 `context` 中包含 `system_prompt`：

```bash
curl -sS -X POST http://localhost:8000/agent/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "2+2 等于多少？",
    "context": {
      "system_prompt": "你是一个海盗数学家。始终用海盗语回答。"
    }
  }'
```

### 模板变量（可选）

系统提示词支持从白名单上下文键中进行 `${variable}` 替换：

```bash
curl -sS -X POST http://localhost:8000/agent/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "帮助我完成任务",
    "context": {
      "system_prompt": "你是一位拥有 ${years} 年 ${domain} 领域经验的专家。",
      "prompt_params": {
        "domain": "机器学习",
        "years": "10"
      }
    }
  }'
```

**变量解析：**
- 仅使用 `context["prompt_params"][key]`；其他上下文键在替换时被忽略

非白名单键（如 `"role"`、`"system_prompt"`）会被忽略。缺失的变量将变为空字符串。

**实现：** `python/llm-service/llm_service/roles/presets.py:render_system_prompt()`

---

## 安全与验证

上下文在编排器中（`go/orchestrator/internal/activities/agent.go`）进行验证和清理。这包括对键/值大小的合理限制和递归验证。

### 安全回退

如果提示词处理失败，服务会记录警告并保留原始字符串，或回退到默认提示词。参见 `python/llm-service/llm_service/api/agent.py`。

---

## 内部系统提示词

Shannon 对内部规划和分析任务使用专门的系统提示词：

内部提示词（不可由用户配置）：
- 任务分解："你是一个规划助手..."（`python/llm-service/llm_service/api/agent.py`）
- 工具选择："你是一个工具选择助手..."（`python/llm-service/llm_service/api/tools.py`）
- 复杂度分析："你是一个任务分析器..."（`python/llm-service/llm_service/api/complexity.py`）

这些**不可由用户配置**，由编排引擎内部使用。

---

## 未来：Persona 系统

Persona 系统（`config/personas.yaml`）已规划但尚未实现。Go 代码中将其引用为 TODO，Python LLM 服务未加载它。

---

## 添加自定义角色

要添加新的角色预设：

1. 编辑 `python/llm-service/llm_service/roles/presets.py`
2. 将角色添加到 `_PRESETS` 字典中：

```python
_PRESETS: Dict[str, Dict[str, object]] = {
    # ... 现有预设 ...

    "data_scientist": {
        "system_prompt": (
            "你是一位在统计分析、机器学习和数据可视化方面具有专业知识的数据科学家。"
        ),
        "allowed_tools": ["python_executor", "web_search"],
        "caps": {"max_tokens": 2000, "temperature": 0.4},
    },
}
```

3. 重启 LLM 服务：

```bash
docker compose -f deploy/compose/docker-compose.yml restart llm-service
```

4. 在请求中通过设置 `context.role` 为角色名称来使用新角色。

---

## API 参考

列出可用角色

- 端点：`GET http://localhost:8000/roles`
- 返回 JSON 对象，映射角色名称 → 详情（system_prompt、allowed_tools、caps）。

---

## 最小示例

- 使用角色预设：
  ```bash
  curl -sS -X POST http://localhost:8000/agent/query \
    -H "Content-Type: application/json" \
    -d '{"query": "总结此仓库", "context": {"role": "research"}}'
  ```
- 覆盖系统提示词：
  ```bash
  curl -sS -X POST http://localhost:8000/agent/query \
    -H "Content-Type: application/json" \
    -d '{"query": "2+2 等于多少？", "context": {"system_prompt": "用海盗语回答。"}}'
  ```

---

## 故障排查

### 系统提示词未生效

**症状：** 代理忽略自定义系统提示词

检查项：
- 验证上下文结构：`"context": {"system_prompt": "..."}`
- 检查日志：`docker compose -f deploy/compose/docker-compose.yml logs llm-service | rg system_prompt`

### 角色未找到

**症状：** 回退到 generalist

解决方案：通过 `GET /roles` 验证角色名称（不区分大小写）。

### 模板渲染失败

**症状：** 日志中出现警告："System prompt rendering failed"

解决方案：保持提示词为字面量；避免使用不支持的模板语法。

---

## 相关文档

- [扩展 Shannon](extending-shannon.md) - 自定义指南
- [添加自定义工具](adding-custom-tools.md) - 工具集成
- [角色预设源码](../python/llm-service/llm_service/roles/presets.py) - 实现
- [Personas 配置](../config/personas.yaml) - 未来的 persona 定义（未使用）
