# Shannon 脚本

此目录包含用于 Shannon 平台运维、开发和测试的实用脚本。

## 核心脚本

### 用户界面
- **`submit_task.sh`** - 通过 gRPC 向 Shannon 提交任务。用于测试的主要用户界面。
  ```bash
  ./scripts/submit_task.sh "你的查询内容"
  ```

### 测试与验证
- **`smoke_e2e.sh`** - 端到端冒烟测试，用于验证整个系统流程。
  通过 `make smoke` 使用。

- **`stream_smoke.sh`** - 测试 SSE/WebSocket 流式传输能力。
  通过 `make stream` 使用。

### 设置与初始化
- **`bootstrap_qdrant.sh`** - 可选的辅助脚本，在重新启用向量搜索后，从宿主机创建 Qdrant 集合。
- **`init_qdrant.sh`** - 可选的容器端包装脚本，用于调用共享的 Qdrant 迁移脚本。
- **`seed_postgres.sh`** - 使用初始模式和数据填充 PostgreSQL。通过 `make dev` 使用。
- **`install_buf.sh`** - 在缺少时安装 `buf` CLI 工具用于 protobuf 管理。

### 开发工具
- **`replay_workflow.sh`** - 从导出的历史记录中重放 Temporal 工作流，用于确定性调试。
- **`signal_team.sh`** - 在手动测试期间向运行中的工作流发送 Temporal 信号（招募/退休）。

## 测试脚本

测试脚本已移至 `tests/scripts/` 以获得更好的组织结构：
- `test_budget_controls.sh` - 测试令牌预算强制执行
- `test_ci_local.sh` - 在本地运行 CI 流水线
- `test_grpc_reflection.sh` - 验证 gRPC 反射 API
- `test_token_aggregation.sh` - 测试令牌使用量聚合

## 在 Makefile 中的使用

这些脚本已集成到 Makefile 目标中：
- `make seed` → 使用 `seed_postgres.sh`
- 首先重新启用向量搜索，然后手动运行 `bootstrap_qdrant.sh`（如果你想在本地创建 Qdrant 集合）
- `make smoke` → 运行 `smoke_e2e.sh`
- `make stream` → 运行 `stream_smoke.sh`
- `make proto` → 可能调用 `install_buf.sh`

## 最佳实践

1. 所有脚本应有正确的 shebang（`#!/bin/bash` 或 `#!/usr/bin/env python3`）
2. 使用 `set -e` 在出错时退出
3. 在顶部包含描述性注释
4. 尽可能使脚本具有幂等性
5. 使用有意义的退出码

## 贡献

添加新脚本时：
1. 遵循 snake_case 命名约定
2. 为 bash 脚本添加 `.sh` 扩展名
3. 使脚本可执行：`chmod +x script_name.sh`
4. 使用描述更新此 README
5. 考虑是否应将其集成到 Makefile 中
