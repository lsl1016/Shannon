# 工作流控制信号

本文档介绍 Shannon 工作流的暂停/恢复/取消控制信号系统。

## 概述

Shannon 支持用于工作流管理的三种控制信号：
- **暂停**：在下一次检查点临时停止工作流执行
- **恢复**：从检查点继续已暂停的工作流
- **取消**：永久停止工作流并执行清理

## 信号语义

### 暂停信号（`pause_v1`）

**暂停何时生效：**
- 暂停信号会立即排队，但仅在**检查点**处生效
- 检查点是工作流执行中暂停安全的位置
- OrchestratorWorkflow 的检查点：`pre_routing`、`pre_simple_workflow`、`pre_supervisor_workflow`、`pre_dag_workflow`
- Strategy 工作流的检查点：`pre_simple_strategy`、`pre_research_workflow` 等
- 子工作流有自己的检查点：`pre_execution`、`pre_completion`、`post_agent`

**“检查点”的含义：**
- 检查点是工作流开发者有意插入的暂停点
- 在检查点处，工作流查询其控制状态，如果已暂停则阻塞
- 检查点确保工作流在一致、可恢复的状态下暂停
- 正在进行的工作（LLM 调用、活动）会在暂停前完成
- 检查点使用 `workflow.Await()` 实现高效阻塞（无定时器轮询）

**行为：**
1. 信号发送到父工作流
2. 父工作流将信号传播到所有已注册的子工作流
3. 每个工作流在下一个检查点处阻塞，直到恢复或取消

### 恢复信号（`resume_v1`）

**恢复何时生效：**
- 立即解除在检查点处阻塞的工作流
- 传播到所有子工作流

**行为：**
1. 清除暂停状态
2. 发出 `WORKFLOW_RESUMED` SSE 事件
3. 工作流从检查点继续执行

### 取消信号（`cancel_v1`）

**取消何时生效：**
- 立即将工作流标记为已取消
- 下一个检查点返回错误，终止工作流

**行为：**
1. 设置带原因的状态为已取消
2. 传播到所有子工作流
3. 发出 `WORKFLOW_CANCELLING` 然后 `WORKFLOW_CANCELLED` 事件
4. 工作流以取消错误终止

## API 端点

### gRPC

```protobuf
// 暂停运行中的工作流
rpc PauseTask(PauseTaskRequest) returns (PauseTaskResponse);

// 恢复已暂停的工作流
rpc ResumeTask(ResumeTaskRequest) returns (ResumeTaskResponse);

// 取消工作流
rpc CancelTask(CancelTaskRequest) returns (CancelTaskResponse);

// 获取当前控制状态
rpc GetControlState(GetControlStateRequest) returns (GetControlStateResponse);
```

### 查询处理器

工作流注册一个查询处理器 `control_state_v1`，返回：
```go
type WorkflowControlState struct {
    IsPaused      bool
    PausedAt      time.Time
    PauseReason   string
    PausedBy      string
    IsCancelled   bool
    CancelReason  string
    CancelledBy   string
}
```

## SSE 事件

控制信号发出 SSE 事件供前端集成：

| 内部类型 | SSE 事件名称 | 负载 |
|--------------|----------------|---------|
| WORKFLOW_PAUSING | workflow.pausing | workflow_id、agent_id、message |
| WORKFLOW_PAUSED | workflow.paused | workflow_id、agent_id、checkpoint、message |
| WORKFLOW_RESUMED | workflow.resumed | workflow_id、agent_id、message |
| WORKFLOW_CANCELLING | workflow.cancelling | workflow_id、agent_id、message |
| WORKFLOW_CANCELLED | workflow.cancelled | workflow_id、agent_id、message |

## 集成指南

### 向工作流添加检查点

```go
func MyWorkflow(ctx workflow.Context, input TaskInput) (TaskResult, error) {
    // 初始化控制处理器
    controlHandler := &control.SignalHandler{
        WorkflowID: workflow.GetInfo(ctx).WorkflowExecution.ID,
        AgentID:    "my-agent",
        Logger:     workflow.GetLogger(ctx),
        EmitCtx:    emitCtx,
    }
    controlHandler.Setup(ctx)

    // 执行前检查点
    if err := controlHandler.CheckPausePoint(ctx, "pre_execution"); err != nil {
        return TaskResult{Success: false, ErrorMessage: err.Error()}, err
    }

    // ... 执行工作 ...

    // 完成前检查点
    if err := controlHandler.CheckPausePoint(ctx, "pre_completion"); err != nil {
        return TaskResult{Success: false, ErrorMessage: err.Error()}, err
    }

    return TaskResult{Success: true}, nil
}
```

### 子工作流注册

对于生成子工作流的父工作流：

```go
// 启动子工作流之前
childFuture := workflow.ExecuteChildWorkflow(ctx, ChildWorkflow, input)
var childExec workflow.Execution
childFuture.GetChildWorkflowExecution().Get(ctx, &childExec)
controlHandler.RegisterChildWorkflow(childExec.ID)

// 子工作流完成后
err := childFuture.Get(ctx, &result)
controlHandler.UnregisterChildWorkflow(childExec.ID)
```

## 确定性与重放

控制信号使用版本门控确保重放安全：

```go
version := workflow.GetVersion(ctx, "pause_resume_v1", workflow.DefaultVersion, 1)
if version < 1 {
    // 旧行为（无控制信号）
    return
}
// 控制信号处理
```

这确保：
- 现有工作流重放不受影响
- 新工作流获得控制信号支持
- 运行中工作流平滑迁移

## 测试

控制信号的单元测试：
```bash
go test -v ./internal/workflows/control/...
```

测试覆盖：
- 信号处理器设置
- 暂停信号接收与阻塞
- 恢复信号解除阻塞
- 取消信号终止
- 重放确定性
- 子工作流注册
- 多次暂停/恢复循环

## 故障排查

### 发送了暂停信号但工作流继续执行
- **原因**：工作流尚未到达检查点
- **解决方案**：在工作流的关键位置添加检查点

### 子工作流未暂停
- **原因**：父工作流未注册子工作流 ID
- **解决方案**：在启动子工作流之前使用 `RegisterChildWorkflow()`

### 控制状态显示已暂停但状态为 RUNNING
- **原因**：数据库状态更新发生在终端状态
- **解决方案**：查询 `GetControlState` 获取实时暂停状态
