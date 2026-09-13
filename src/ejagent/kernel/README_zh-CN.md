# 直接接入执行内核

[English](README.md) · [模块索引](../README_zh-CN.md)

`RuntimeKernel` 执行一个不可变 `RunSpec` 并返回 `RunOutcome`。
多数应用应装配 [AgentHarness](../harness/README_zh-CN.md)，由它管理外围的资源生命周期、
任务准备、控制和会话提交。

## 构造与 Run 输入

| 配置 | 默认值 | 作用 |
| --- | --- | --- |
| `model`、`tools` | 必填 | 已就绪的 `ModelPort` 与 `ToolExecutor` |
| `context` | `None` | 使用 `IdentityContextPipeline` |
| `trajectory` | `None` | 不配置检查点监控器 |
| `clock` | `None` | UTC 时间戳 |
| `monotonic_clock` | `None` | 单调时钟测量耗时 |
| `run(..., cancellation=...)` | `None` | 创建新的取消 token |
| `run(..., controls=...)` | `None` | 不配置 steering 输入源 |

`RunSpec` 必填 `run_id`、`base_revision`、`intent`、`task`、`messages`。
`TASK` 要求非空任务文本，`CONTINUE` 要求 `task=None`。
默认 `limits=RunLimits()`、`configuration_revision="default"`、`metadata={}`、
`evaluation_plan=None`、`completion_policy=CompletionPolicy()`。
强制验收同时需要计划和监控器。Actor 预算与完成重试见 [Harness](../harness/README_zh-CN.md)。

以下函数接收已经启动的集成组件，不持久化结果：

```python
from ejagent.contracts import CancellationSource, RunIntent, RunSpec
from ejagent.kernel import RuntimeKernel


async def execute_once(model, tools):
    kernel = RuntimeKernel(model=model, tools=tools)
    spec = RunSpec(
        run_id="one-run",
        base_revision=0,
        intent=RunIntent.TASK,
        task="Explain the configured tools",
        messages=(),
    )
    return await kernel.run(spec, cancellation=CancellationSource().token)
```

直接调用者需自行启动和关闭资源、提供唯一 Run ID 与当前基础状态，并决定如何提交。
Kernel 不调用 Planner、不恢复存储、不调度 follow-up。
仅在自行提供临时控制队列时实现 `RunControlSource.drain_steering()`。

## 检查点与自定义监控

`TrajectoryMonitor.capture(signal, cancellation=...)` 是异步方法，返回满足
`TrajectoryCaptureResult` 的对象，包含 `checkpoint_id: str`、`verdict: str` 和
`completion_allowed: bool | None`；`close_run(run_id)` 释放监控状态。
通常直接使用已实现此接口的 `EvaluationMonitor`。

Kernel 的采集位置包括初始基线（turn 为 0）、工具批次完成和提出完成声明。
工具 `COMPLETE` 结果也参与已绑定计划的完成检查。
`CheckpointTrigger` 还定义了 `VERIFICATION_COMPLETED` 和 `EXTERNAL_CHANGE`，
但存在枚举值不代表会自动安装后台监听器或调度这些采集。
信号包含声明的计划、受限完成文本、工具回执引用和执行成本，不等于直接访问原始环境。

强制验收消费 `completion_allowed`，不是展示用 verdict 字符串或覆盖率阈值。
采集失败不能批准完成。Kernel 退出时清理监控器的 Run 状态。
证据采集、注册和失败处理见[评估](../evaluation/README_zh-CN.md)。
