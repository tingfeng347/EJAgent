# 数据协议与扩展接口

[English](README.md) · [模块索引](../README_zh-CN.md)

本模块以与 Provider 无关的 dataclass 和 Protocol 定义 EJAgent 的公共边界。
应用提供符合协议的行为实现，无需重新定义协议本身。多数接口已有可直接使用的内置适配器。

## 应用提供的行为

| 接口 | 约定 | 何时自行实现 |
| --- | --- | --- |
| `ModelPort` | `stream(request, cancellation=...)` 产出模型事件 | 接入其他模型后端；否则使用内置 Provider |
| `ToolExecutor` | `definitions`、异步 `execute(...) -> ToolExecutionResult` | 自定义调度；否则注册 `FunctionTool` 或 MCP |
| `TaskPlanner` | 异步 `plan(...) -> PlanningResult` | 自定义规划；否则配置 `ModelTaskPlanner` |
| `ContextPipeline` | 异步 `build(...) -> ContextView` | 自定义投影；已有直接投影、Skills 和压缩实现 |
| `ContextCompactor` | 异步 `compact(...) -> ContextCompactionOutput` | 使用 `DerivedCompactionPipeline` 时必须提供；没有内置摘要器 |
| `SessionStore` | 异步 `load(agent_id)`、`commit(SessionCommit)` | 自定义持久化后端；已有 JSONL |
| `AuditReader` | 异步 `load_audit(agent_id)` | 可选审计查询接口；JSONL 已实现 |
| `RunObserver` | 异步 `observe(RunAudit)` | 可选的执行后报告 |
| `ManagedResource` | 异步 `start()`、`shutdown()` | 集成拥有连接等资源时 |
| `RunControlSource` | `drain_steering()` | 直接接入 Kernel 时；Harness 通常已提供 |

`EvidenceSource` 与异步函数类型别名 `Verifier` 位于[评估模块](../evaluation/README_zh-CN.md)。
`TrajectoryMonitor` 和结构化返回约定 `TrajectoryCaptureResult` 位于
[内核](../kernel/README_zh-CN.md)。内置评估无法满足需求时才需要自定义这些实现。

## 配置与运行数据的区别

`RunLimits`、`CompletionPolicy`、`EvaluationCriterion`、`EvaluationPlan` 是配置输入；
默认值与组合条件见 [Harness](../harness/README_zh-CN.md) 和
[评估](../evaluation/README_zh-CN.md)。`ModelRequest` 可携带 `tools=()`、
`max_output_tokens=None`、`response_format=None`，具体功能由调用方和 Provider 决定。

`ToolControl` 是应用请求的执行决策，见[工具](../tools/README_zh-CN.md)。
`StepStatus` 表示计划状态；`RunStatus`、`StopReason`、`FailureCode`、控制回执状态描述结果，
不属于功能开关。

消息与快照不可变。会话包含用户、系统、助手消息及工具结果；临时上下文还可包含摘要和临时指令。
当前消息内容是文本，没有图片输入协议。适配时需保持工具调用与结果顺序，并使用 JSON 兼容数据。

## 校验与失败责任

公共 dataclass 校验自身不变量。Planner/Judge 的模型 JSON 使用独立的内部 Pydantic 模型校验，
随后检查能力绑定和引用，详见[结构化输出](../../../docs/structured-output.md)。

适配器必须响应取消并返回约定类型。预期操作失败使用对应异常类型，协议违规属于集成错误。
异常的 `retryable` 字段只是元数据，不代表通用自动重试；生命周期和重试行为由消费模块决定。
