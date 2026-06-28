# Shannon 技能系统

## 概述

技能是基于 Markdown 的工作流定义，为常见任务提供结构化提示词、工具配置和执行约束。它们兼容 Anthropic 的 Agent Skills 规范。

当任务使用技能时，技能的 Markdown 内容将作为系统提示词，引导 AI 代理执行结构化工作流。这实现了一致、可重复的任务执行模式。

## 目录结构

```
config/skills/
├── core/           # 内置技能（提交到仓库）
│   ├── code-review.md
│   ├── debugging.md
│   └── test-driven-dev.md
├── user/           # 用户自定义技能（gitignore）
└── vendor/         # 供应商特定技能（gitignore）
```

| 目录 | 用途 | Git 状态 |
|-----------|---------|------------|
| `core/` | Shannon 附带的内置技能 | 已提交 |
| `user/` | 个人/团队自定义技能 | gitignore |
| `vendor/` | 第三方或供应商特定技能 | gitignore |

## 技能文件格式

技能是带有 YAML frontmatter 后跟 Markdown 内容的 Markdown 文件：

```markdown
---
name: my-skill
version: 1.0.0
author: Your Name
category: development
description: 该技能功能的简要描述
requires_tools:
  - file_read
  - file_write
  - bash
requires_role: generalist
budget_max: 5000
dangerous: false
enabled: true
metadata:
  complexity: medium
  estimated_duration: 10min
---

# 技能标题

Markdown 格式的技能说明...

## 步骤 1：收集信息
- 使用 `file_list` 发现文件
- 使用 `file_read` 读取相关文件

## 步骤 2：执行分析
...

## 输出格式
按以下结构提供发现：
- 摘要
- 详细信息
- 建议
```

### Frontmatter 字段

| 字段 | 是否必需 | 类型 | 默认值 | 描述 |
|-------|----------|------|---------|-------------|
| `name` | 是 | string | - | 唯一标识符（仅限小写字母、连字符、下划线） |
| `version` | 否 | string | `1.0.0` | 语义化版本 |
| `author` | 否 | string | - | 技能作者 |
| `category` | 否 | string | - | 分组类别（例如 development、research） |
| `description` | 否 | string | - | 简要描述（若 `dangerous: true` 则必需） |
| `requires_tools` | 否 | list | `[]` | 该技能需要的工具列表 |
| `requires_role` | 否 | string | - | 要使用的角色预设（绕过任务分解） |
| `budget_max` | 否 | int | - | 执行的最大令牌预算 |
| `dangerous` | 否 | bool | `false` | 技能是否执行危险操作 |
| `enabled` | 否 | bool | `true` | 技能是否激活 |
| `metadata` | 否 | object | `{}` | 额外的键值元数据 |

### 名称验证

技能名称只能包含：
- 小写字母（a-z）
- 大写字母（A-Z）
- 数字（0-9）
- 连字符（-）
- 下划线（_）

## 通过 API 使用技能

### 使用技能执行任务

```bash
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "query": "审查认证模块的安全问题",
    "skill": "code-review",
    "session_id": "my-session-123"
  }'
```

当指定 `skill` 时：
1. 技能的 Markdown 内容成为系统提示词
2. 如果设置了 `requires_role`，则应用该角色（绕过分解）
3. 任务在技能指导下以单代理执行方式运行

### 使用版本化技能

使用 `name@version` 请求特定版本：

```bash
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "query": "调试登录失败问题",
    "skill": "debugging@1.0.0",
    "session_id": "my-session-123"
  }'
```

## API 端点

### 列出所有技能

```bash
GET /api/v1/skills
```

响应：
```json
{
  "skills": [
    {
      "name": "code-review",
      "version": "1.0.0",
      "category": "development",
      "description": "系统化代码审查工作流",
      "requires_tools": ["file_read", "file_list", "bash"],
      "dangerous": false,
      "enabled": true
    }
  ],
  "count": 3,
  "categories": ["development"]
}
```

### 按类别筛选

```bash
GET /api/v1/skills?category=development
```

### 获取技能详情

```bash
GET /api/v1/skills/{name}
```

响应：
```json
{
  "skill": {
    "name": "code-review",
    "version": "1.0.0",
    "author": "Shannon",
    "category": "development",
    "description": "系统化代码审查工作流",
    "requires_tools": ["file_read", "file_list", "bash"],
    "requires_role": "critic",
    "budget_max": 5000,
    "dangerous": false,
    "enabled": true,
    "content": "# 代码审查技能\n\n你正在执行..."
  },
  "metadata": {
    "source_path": "/app/config/skills/core/code-review.md",
    "content_hash": "abc123...",
    "loaded_at": "2026-01-26T10:00:00Z"
  }
}
```

### 列出技能版本

```bash
GET /api/v1/skills/{name}/versions
```

响应：
```json
{
  "name": "code-review",
  "versions": [
    {"name": "code-review", "version": "2.0.0", ...},
    {"name": "code-review", "version": "1.0.0", ...}
  ],
  "count": 2
}
```

## 创建自定义技能

### 步骤 1：创建技能文件

在 `config/skills/user/` 中创建 `.md` 文件：

```bash
mkdir -p config/skills/user
cat > config/skills/user/my-analysis.md << 'EOF'
---
name: my-analysis
version: 1.0.0
author: Your Name
category: analysis
description: 自定义分析工作流
requires_tools:
  - file_read
  - file_list
requires_role: generalist
budget_max: 3000
---

# 我的分析技能

AI 代理的说明...

## 步骤 1：收集数据
...

## 步骤 2：分析
...

## 输出格式
...
EOF
```

### 步骤 2：将目录添加到 SKILLS_PATH（可选）

如果使用自定义目录：

```bash
export SKILLS_PATH="config/skills/core:config/skills/user:/custom/skills"
```

### 步骤 3：重启 Gateway

技能在 gateway 启动时加载：

```bash
docker compose -f deploy/compose/docker-compose.yml restart gateway
```

### 步骤 4：验证加载

```bash
curl -sS http://localhost:8080/api/v1/skills | jq '.skills[].name'
```

## 环境变量

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `SKILLS_PATH` | `config/skills/core`（开发）或 `/app/config/skills/core`（容器） | 用冒号分隔的目录列表，用于扫描技能 |

示例：
```bash
# 多个目录
export SKILLS_PATH="/app/config/skills/core:/app/config/skills/user:/custom/vendor-skills"
```

## 安全注意事项

### 危险技能

`dangerous: true` 表示技能执行可能具有破坏性的操作：

```yaml
---
name: cleanup-files
dangerous: true
description: 删除临时文件（dangerous=true 时必需）
---
```

危险技能的要求：
- 必须有非空的 `description` 字段
- 应在用户明确确认后使用
- 考虑在生产环境中实施审批工作流

### 基于角色的访问控制

`requires_role` 字段映射到 Shannon 的角色预设：

| 角色 | 描述 | 典型工具 |
|------|-------------|---------------|
| `generalist` | 通用代理 | file_read、file_list、bash、web_search |
| `critic` | 代码审查与分析 | file_read、file_list、bash |
| `developer` | 开发任务 | file_read、file_write、file_list、bash、python_executor |

当设置了 `requires_role` 时，编排器会绕过任务分解，以指定角色作为单代理执行任务。

### 工具限制

`requires_tools` 字段声明技能期望的工具：

```yaml
requires_tools:
  - file_read
  - file_list
  - bash
```

这作为文档，并可用于：
- 预验证所需工具是否可用
- 访问控制策略
- 审计日志

## 最佳实践

1. **具体明确**：技能应提供清晰的分步指导
2. **结构化输出**：在技能中定义预期的输出格式
3. **工具要求**：仅列出技能实际使用的工具
4. **版本控制**：对技能更新使用语义化版本
5. **测试技能**：在部署前验证技能能产生预期结果
6. **记录上下文**：包含技能期望的输入内容
7. **错误处理**：指导代理处理边缘情况

## 示例：内置技能

### code-review

系统化代码审查工作流：
- 安全分析（注入、XSS、认证漏洞）
- 代码质量指标
- 性能审查
- 测试覆盖率分析

### debugging

结构化调试方法论：
- 理解问题
- 收集信息
- 提出并验证假设
- 根因分析
- 实施解决方案

### test-driven-dev

测试驱动开发工作流：
- 先编写失败的测试
- 实现最少代码以通过测试
- 放心地进行重构

## 故障排查

### 技能未加载

1. 检查目录是否存在：
   ```bash
   ls -la config/skills/core/
   ```

2. 验证 YAML frontmatter 语法：
   ```bash
   head -20 config/skills/core/my-skill.md
   ```

3. 检查 gateway 日志中的加载错误：
   ```bash
   docker compose -f deploy/compose/docker-compose.yml logs gateway | grep -i skill
   ```

### 技能未找到（404）

1. 验证技能是否启用（`enabled: true` 或省略）
2. 检查名称拼写是否完全匹配
3. 确保文件扩展名为 `.md`
4. 确认目录在 `SKILLS_PATH` 中

### 版本冲突

如果两个技能具有相同的 `name@version`，加载将失败。请使用唯一的版本号或将技能放在不同的目录中。
