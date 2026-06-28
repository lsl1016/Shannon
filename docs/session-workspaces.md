# Shannon 会话工作区

## 概述

会话工作区为每个用户会话提供隔离的文件系统环境。所有文件操作都限定在会话的工作区目录内，确保不同会话无法访问彼此的文件。

这种隔离对于以下方面至关重要：
- **多租户安全**：防止用户之间的数据泄露
- **可重现性**：每个会话从干净的工作区开始
- **资源管理**：每个会话的磁盘配额防止无限制的使用

## 本地与 EKS 执行

Shannon 根据环境使用**不同的 Python 沙箱后端**：

| | 本地（Docker Compose） | EKS（生产环境） |
|---|---|---|
| **`python_executor` 后端** | WASI 沙箱（通过 wasmtime 运行 Python.wasm） | Firecracker 微型虚拟机 |
| **Python 包** | 仅标准库 | 完整数据科学栈（pandas、numpy、scipy、torch 等） |
| **写入 `/workspace/` 的文件** | 当存在 `session_id` 时支持（WASI 预打开目录） | 支持，通过 EFS 支持的 ext4 块设备 |
| **文件持久化** | `file_*` 工具 + `python_executor` 写入 Docker 卷上的会话目录 | `file_*` 工具 + `python_executor` 均写入 EFS（带双向同步） |
| **隔离级别** | WASI 基于能力的安全（wasmtime 预打开目录） | 硬件级虚拟机隔离 |

**本文档描述本地 Docker Compose 路径。**

## 文件工具的角色要求

**重要**：文件工具受角色门控。您必须在请求上下文中指定适当的角色才能访问文件操作。

| 角色 | 可用的文件工具 |
|------|---------------------|
| `developer` | `file_read`、`file_write`、`file_list`、`bash`、`python_executor` |
| `generalist`（默认） | `file_read`、`file_list`、`bash`（无 `file_write`） |
| `critic` | `file_read`、`file_list`、`bash` |

**要写入文件，必须使用 `role: "developer"`：**

```json
{
  "query": "创建名为 test.txt 的文件，内容为 'Hello'",
  "session_id": "my-session-123",
  "context": {
    "role": "developer"
  }
}
```

如果没有 `role: "developer"`，代理无法使用 `file_write`，请求将失败，或者 LLM 将无法使用该工具。

### 示例：跨请求的文件持久化

```bash
# 第一个请求 - 创建文件（需要 developer 角色）
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "query": "将 'test data' 写入 data.txt",
    "session_id": "persist-test",
    "context": {"role": "developer"}
  }'

# 第二个请求 - 读取同一文件（相同 session_id）
curl -X POST http://localhost:8080/api/v1/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "query": "读取 data.txt 并显示内容",
    "session_id": "persist-test",
    "context": {"role": "developer"}
  }'
```

文件在会话内持久化——使用相同 `session_id` 的后续请求可以访问先前创建的文件。

## 架构

### 目录结构

```
/tmp/shannon-sessions/              # SHANNON_SESSION_WORKSPACES_DIR（基础目录）
├── session-abc123/                 # 会话 A 的工作区
│   ├── code/
│   │   └── main.py
│   ├── data/
│   │   └── input.csv
│   └── output.txt
├── session-def456/                 # 会话 B 的工作区
│   └── results.json
└── session-xyz789/                 # 会话 C 的工作区
    └── notes.md
```

每个会话获得一个以其 `session_id` 命名的专用子目录。工作区在首次文件操作时自动创建。

### 组件流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           工具请求                                           │
│  {"tool": "file_write", "path": "data.txt", "content": "Hello"}             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Python llm-service（file_ops.py）                         │
│                                                                              │
│  检查：is_sandbox_enabled()？                                                │
│    ├── 是 → 通过 gRPC 代理到 Rust agent-core                                │
│    └── 否 → 在本地使用 Python 路径验证执行                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┴───────────────────────┐
            │                                               │
            ▼（WASI 模式）                                   ▼（Python 模式）
┌───────────────────────────────┐           ┌───────────────────────────────┐
│   Rust agent-core             │           │   Python 路径验证              │
│   （sandbox_service.rs）       │           │                                │
│                               │           │   - 解析规范路径               │
│   - WorkspaceManager          │           │   - 检查白名单                 │
│   - SafeCommand 执行          │           │   - 允许则执行                 │
│   - 基于能力的安全             │           │                                │
└───────────────────────────────┘           └───────────────────────────────┘
```

## 安全模型

Shannon 为文件操作实现两个安全层。根据配置，其中一个或两个可能处于活动状态。

### 阶段 2：Python 路径验证（回退）

当 `SHANNON_USE_WASI_SANDBOX=0` 时，文件操作直接在 Python 中通过路径验证处理：

1. **路径规范化**：所有路径使用 `Path.resolve()` 解析，以消除符号链接和 `..` 组件
2. **白名单强制执行**：解析后的路径必须位于允许的目录内：
   - 会话工作区：`{SHANNON_SESSION_WORKSPACES_DIR}/{session_id}/`
   - 可选的共享工作区：`SHANNON_WORKSPACE`
   - 仅开发模式：当前工作目录（如果 `SHANNON_DEV_ALLOW_CWD=1`）
3. **符号链接保护**：指向允许目录外部的符号链接被拒绝

**被阻止的路径遍历示例：**
```python
# 请求：{"path": "../../../etc/passwd"}
# 解析后：/etc/passwd
# 结果：拒绝（不在会话工作区内）
```

### 阶段 3：WASI 沙箱（默认，增强安全）

当 `SHANNON_USE_WASI_SANDBOX=1` 时（docker-compose 中默认），文件操作通过 gRPC 代理到 Rust agent-core 服务，提供额外的安全保障：

1. **基于能力的安全**：WASI 运行时仅授予对会话工作区目录的访问权限
2. **内存限制**：沙箱执行受内存限制约束
3. **原生命令实现**：Shell 命令（`ls`、`cat` 等）在纯 Rust 中实现，而非通过生成 shell 进程
4. **元字符阻止**：Shell 元字符在解析时被拒绝

**安全属性：**
- 无 shell 注入可能（命令被解析，而非通过 shell 执行）
- 沙箱无网络访问
- 内存受限执行
- 无法访问系统文件；仅暴露 `/workspace/`（会话目录）

## 配置

### 环境变量

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `SHANNON_USE_WASI_SANDBOX` | `0` | 启用 WASI 沙箱模式。设置为 `1`、`true` 或 `yes` 以启用。 |
| `SHANNON_SESSION_WORKSPACES_DIR` | `/tmp/shannon-sessions` | 所有会话工作区的基础目录 |
| `SHANNON_MAX_WORKSPACE_SIZE_MB` | `100` | 每个会话的磁盘配额（兆字节） |
| `SHANNON_WORKSPACE` | （未设置） | 可选的对所有会话可用的共享工作区 |
| `SHANNON_DEV_ALLOW_CWD` | `0` | 仅开发：允许访问当前工作目录 |
| `SHANNON_ALLOW_GLOBAL_TMP` | `0` | 允许访问全局 `/tmp`（为隔离默认禁用） |

### Docker Compose 配置

`agent-core` 和 `llm-service` 必须共享相同的会话工作区卷：

```yaml
volumes:
  shannon-sessions:

services:
  agent-core:
    volumes:
      - shannon-sessions:/tmp/shannon-sessions
    environment:
      - SHANNON_SESSION_WORKSPACES_DIR=/tmp/shannon-sessions

  llm-service:
    volumes:
      - shannon-sessions:/tmp/shannon-sessions
    environment:
      - SHANNON_USE_WASI_SANDBOX=${SHANNON_USE_WASI_SANDBOX:-0}
      - SHANNON_SESSION_WORKSPACES_DIR=/tmp/shannon-sessions
```

## 使用文件工具

### file_read

从会话工作区读取文件内容。

**参数：**
| 参数 | 类型 | 必需 | 默认值 | 描述 |
|-----------|------|----------|---------|-------------|
| `path` | 字符串 | 是 | - | 文件路径（相对于工作区） |
| `encoding` | 字符串 | 否 | `utf-8` | 文件编码（`utf-8`、`ascii`、`latin-1`） |
| `max_size_mb` | 整数 | 否 | `10` | 最大读取文件大小（1-100 MB） |

**示例：**
```json
{
  "tool": "file_read",
  "path": "data/config.json"
}
```

**响应：**
```json
{
  "success": true,
  "output": {"key": "value"},
  "metadata": {
    "path": "/tmp/shannon-sessions/my-session/data/config.json",
    "size_bytes": 42,
    "encoding": "utf-8",
    "file_type": ".json"
  }
}
```

JSON 和 YAML 文件会自动解析为结构化数据。

### file_write

将会话工作区写入文件。

**参数：**
| 参数 | 类型 | 必需 | 默认值 | 描述 |
|-----------|------|----------|---------|-------------|
| `path` | 字符串 | 是 | - | 写入文件的路径 |
| `content` | 字符串 | 是 | - | 要写入的内容 |
| `mode` | 字符串 | 否 | `overwrite` | `overwrite` 或 `append` |
| `encoding` | 字符串 | 否 | `utf-8` | 文件编码 |
| `create_dirs` | 布尔值 | 否 | `false` | 如果需要则创建父目录 |

**示例：**
```json
{
  "tool": "file_write",
  "path": "output/results.txt",
  "content": "分析完成。\n总条目数：42",
  "create_dirs": true
}
```

**响应：**
```json
{
  "success": true,
  "output": "/tmp/shannon-sessions/my-session/output/results.txt",
  "metadata": {
    "path": "/tmp/shannon-sessions/my-session/output/results.txt",
    "size_bytes": 35,
    "mode": "overwrite",
    "encoding": "utf-8",
    "created_dirs": true
  }
}
```

### file_list

列出会话工作区内目录中的文件。

**参数：**
| 参数 | 类型 | 必需 | 默认值 | 描述 |
|-----------|------|----------|---------|-------------|
| `path` | 字符串 | 是 | - | 要列出的目录路径 |
| `pattern` | 字符串 | 否 | `*` | Glob 模式（例如 `*.txt`、`*.py`） |
| `recursive` | 布尔值 | 否 | `false` | 包含子目录 |
| `include_hidden` | 布尔值 | 否 | `false` | 包含隐藏文件（`.xxx`） |

**示例：**
```json
{
  "tool": "file_list",
  "path": ".",
  "pattern": "*.py",
  "recursive": true
}
```

**响应：**
```json
{
  "success": true,
  "output": [
    {"name": "main.py", "path": "main.py", "is_file": true, "size_bytes": 1024},
    {"name": "utils.py", "path": "lib/utils.py", "is_file": true, "size_bytes": 512}
  ],
  "metadata": {
    "directory": "/tmp/shannon-sessions/my-session",
    "pattern": "*.py",
    "recursive": true,
    "file_count": 2,
    "dir_count": 0
  }
}
```

## 沙箱命令（WASI 模式）

当 `SHANNON_USE_WASI_SANDBOX=1` 时，这些安全命令可直接执行。命令在 Rust 中原生实现（而非通过 shell），消除 shell 注入风险。

### 读取操作

| 命令 | 描述 | 示例 |
|---------|-------------|---------|
| `ls` | 列出目录内容 | `ls -la data/` |
| `cat` | 打印文件内容 | `cat config.json` |
| `head` | 打印前 N 行 | `head -n 10 log.txt` |
| `tail` | 打印后 N 行 | `tail -n 20 log.txt` |
| `wc` | 统计行数/单词数/字节数 | `wc data.csv` |
| `pwd` | 打印工作目录 | `pwd` |

### 写入操作

| 命令 | 描述 | 示例 |
|---------|-------------|---------|
| `mkdir` | 创建目录 | `mkdir -p output/reports` |
| `rm` | 删除文件/目录 | `rm -r temp/` |
| `cp` | 复制文件 | `cp source.txt backup.txt` |
| `mv` | 移动/重命名文件 | `mv old.txt new.txt` |
| `touch` | 创建空文件 | `touch marker.txt` |

### 搜索操作

| 命令 | 描述 | 示例 |
|---------|-------------|---------|
| `find` | 按名称查找文件 | `find . -name "*.py"` |
| `grep` | 搜索模式 | `grep -i error log.txt` |

### 工具

| 命令 | 描述 | 示例 |
|---------|-------------|---------|
| `echo` | 打印文本 | `echo "Hello World"` |

### 命令标志

| 命令 | 支持的标志 |
|---------|-----------------|
| `ls` | `-a`（全部）、`-l`（长格式） |
| `head` | `-n NUM`（行数） |
| `tail` | `-n NUM`（行数） |
| `mkdir` | `-p`（创建父目录） |
| `rm` | `-r`（递归） |
| `grep` | `-i`（忽略大小写） |
| `find` | `-name PATTERN` |

## 被阻止的 Shell 元字符

以下 shell 元字符被**阻止**以防止命令注入：

| 字符 | 含义 | 为何阻止 |
|-----------|---------|-------------|
| `\|` | 管道 | 防止链接到其他命令 |
| `;` | 命令分隔符 | 防止运行额外命令 |
| `&&` | AND 运算符 | 防止条件执行 |
| `\|\|` | OR 运算符 | 防止条件执行 |
| `>` | 输出重定向 | 防止写入任意文件 |
| `<` | 输入重定向 | 防止读取任意文件 |
| `>>` | 追加重定向 | 防止追加到任意文件 |
| `$(` | 命令替换 | 防止嵌入命令执行 |
| `` ` `` | 反引号替换 | 防止嵌入命令执行 |
| `\n` | 换行 | 防止多行注入 |
| `\r` | 回车 | 防止通过 CR 注入 |

**被阻止的命令示例：**
```
cat file.txt | grep secret     # 拒绝：管道字符
ls; rm -rf /                   # 拒绝：分号
echo $(whoami)                 # 拒绝：命令替换
```

## 配额执行

每个会话工作区有可配置的磁盘配额（默认：100MB）。

### 配额工作原理

1. 每次写入操作前，计算当前工作区大小
2. 如果添加新内容会超过配额，操作失败
3. 无部分写入——要么整个写入成功，要么什么都不写入

### 检查工作区大小

`WorkspaceManager` 提供 `get_workspace_size()` 方法，递归计算会话工作区中所有文件的总大小。

### 错误响应

当配额超限时：
```json
{
  "success": false,
  "error": "工作区配额超限：102.5MB > 100MB 限制"
}
```

### 建议

- 在大规模写入前使用 `file_list` 审计工作区内容
- 不再需要时清理临时文件
- 如果合法需求超过默认值，通过配置请求增加配额

## 会话 ID 验证

会话 ID 经过验证以防止路径遍历攻击：

| 规则 | 有效示例 | 无效示例 |
|------|---------------|-----------------|
| 仅字母数字、连字符、下划线 | `session-123`、`user_abc` | `session/../etc` |
| 最大 128 字符 | （任意 <=128 字符字符串） | （>128 字符字符串） |
| 不能以 `.` 开头 | `session.1` | `.hidden-session` |
| 不能包含 `..` | `my-session` | `session..test` |

## 清理

### 自动清理

会话工作区基于会话 TTL（生存时间）自动清理。当会话过期时：

1. 会话记录被标记为删除
2. 工作区目录及其所有内容被移除
3. 清理作为后台进程定期运行

### 手动清理

管理员可以手动删除会话工作区：

```rust
let mgr = WorkspaceManager::from_env();
mgr.delete_workspace("session-abc123")?;
```

### 列出所有工作区

查看所有活动会话工作区：

```rust
let mgr = WorkspaceManager::from_env();
let sessions = mgr.list_workspaces()?;
// 返回：["session-abc123", "session-def456", ...]
```

## 故障排查

### 文件未找到

```
错误：文件未找到：data.txt
```
- 验证文件在会话工作区中存在
- 使用 `file_list` 查看可用文件
- 检查路径是否相对于工作区根目录

### 访问被拒绝

```
错误：不允许读取 /etc/passwd。请使用会话工作区。
```
- 工作区外的绝对路径被拒绝
- 路径遍历尝试（`../`）被阻止
- 使用相对于工作区根目录的路径

### 沙箱 Proto 不可用

```
错误：沙箱 proto 不可用
```
- WASI 沙箱已启用但 agent-core 未运行
- 检查 agent-core 容器是否健康
- 验证服务之间的 gRPC 连接

### 配额超限

```
错误：工作区配额超限
```
- 使用 `rm` 清理未使用的文件
- 如需增加 `SHANNON_MAX_WORKSPACE_SIZE_MB`
- 检查是否有意外的大文件

## 已知限制

### 会话 ID 需要 `role: developer`

文件工具（`file_read`、`file_write`、`file_list`）受角色门控。如果请求上下文中没有 `role: "developer"`，LLM 将无法使用 `file_write`，并可能改用 `python_executor`（如果存在 session_id，它可以写入 `/workspace/`）。

**解决方案**：在提交需要文件操作的任务时，始终包含 `"context": {"role": "developer"}`。

### Python 执行器与文件工具

`python_executor` 和 `file_*` 工具在存在 `session_id` 时均可读写会话工作区：

- **`python_executor`**：在沙箱内将工作区视为 `/workspace/`。在本地 WASI 中，会话目录通过 wasmtime 的预打开目录挂载，具有完全读写权限（`DirPerms::all()`）。在 EKS 上，Firecracker 虚拟机在 `/workspace/` 挂载 ext4。如果没有 `session_id`，则不存在 `/workspace` 挂载，写入失败。
- **`file_*` 工具**：通过 Rust SandboxService（gRPC）直接写入主机上的会话目录。当提供 `session_id` 时始终可写。

**在 EKS 上**，两种工具访问相同的 EFS 支持 ext4，会话目录和 ext4 镜像之间具有双向同步。

## 关键源文件

| 组件 | 文件 | 用途 |
|-----------|------|---------|
| Workspace Manager | `rust/agent-core/src/workspace.rs` | 会话目录管理 |
| Safe Commands | `rust/agent-core/src/safe_commands.rs` | 原生命令实现 |
| Sandbox Service | `rust/agent-core/src/sandbox_service.rs` | 沙箱操作的 gRPC 服务 |
| Python 文件工具 | `python/llm-service/llm_service/tools/builtin/file_ops.py` | 文件读写/列表工具 |
| Sandbox Client | `python/llm-service/llm_service/tools/builtin/sandbox_client.py` | 沙箱代理的 gRPC 客户端 |
