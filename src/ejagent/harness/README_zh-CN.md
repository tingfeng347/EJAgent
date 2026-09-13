# Harness 配置

[English](README.md) · [模块索引](../README_zh-CN.md)

[AgentHarness](core.py) 管理一个逻辑智能体的资源、会话版本、串行 Run 和提交决策。
接入应用负责提供具体组件。

## 构造参数

| 参数 | 默认值 | 作用 |
| --- | --- | --- |
| `agent_id`、`model`、`tools` | 必填 | 稳定会话标识、`ModelPort`、`ToolExecutor` |
| `context` | `None` | 默认直接投影；需要额外上下文时提供 pipeline |
| `planner` | `None` | 不自动准备任务；配置后必须同时提供 `trajectory` |
| `trajectory` | `None` | 不进行检查点评估；通常使用 `EvaluationMonitor` |
| `completion_policy` | `CompletionPolicy()` | 默认观察模式 |
| `initial_messages` | `()` | 初始会话，通常包含 `SystemMessage` 指令 |
| `store` | `None` | 仅保留进程内状态；持久化需提供 `SessionStore` |
| `observers`、`resources` | `()`、`()` | 审计观察器及额外生命周期资源 |
| `limits` | `RunLimits()` | 每个 Run 的默认 Actor 预算 |
| `configuration_revision` | `"default"` | 随 Run 记录的应用配置标签 |
| `run_id_factory`、`clock` | `None`、`None` | 默认 UUID 与 UTC 时钟；可为集成或测试注入 |
| `steering_capacity`、`follow_up_capacity` | `16`、`16` | 队列容量，必须为正整数 |

`async with harness:` 启动资源并恢复存储快照，退出时释放托管资源。
恢复出的会话优先于初始消息。拥有资源的集成可实现 `ManagedResource.start/shutdown`。
`RunObserver.observe(audit)` 接收已结束 Run 的审计数据。

## 完成策略与执行限制

`CompletionPolicy(mode=CompletionMode.OBSERVE, max_retries=2)` 默认只提供建议性评估。
`ENFORCE` 要求监控器和验收计划，仅在监控器明确允许时接受完成声明。
未知或冲突证据不能批准完成。`max_retries` 表示同一个 Run 内额外的完成尝试次数；
耗尽后 Run 失败，会话版本不推进。

`RunLimits` 默认 `max_turns=20`、`max_tokens=None`、`max_repeated_tool_calls=3`。
重复保护按工具名称和参数统计连续相同的工具调用，包括同一批次内的调用。token 限制依赖 Provider 上报的 usage，
在执行边界检查，不是 Provider 计费的硬上限。Planner 和 Judge 使用独立预算。

## 装配与单次 Run 覆盖

以下工厂接收已有模型和执行器，自身不需要凭据：

```python
from ejagent import AgentHarness
from ejagent.contracts import RunLimits


def create_harness(model, tools):
    return AgentHarness(
        agent_id="assistant",
        model=model,
        tools=tools,
        limits=RunLimits(max_turns=12, max_tokens=20_000),
    )
```

`run(task, limits=..., metadata=..., evaluation_plan=...)` 可替换该 Run 的默认限制。
显式传入验收计划会跳过自动规划，否则已配置的 Planner 会在 Actor 执行前准备新任务。
`continue_run()` 不追加用户消息，也不调用 Planner；强制验收的续跑需提供计划。
计划不会跨 Run 或随会话恢复自动继承。

运行时，`steer(text)` 将临时指令排入队列，在安全边界消费；
`follow_up(task, evaluation_plan=...)` 排队启动后续 Run 并返回 handle。
检查控制回执，处理队列满或当前没有活动 Run 等拒绝情况。`cancel(reason)` 请求协作式取消。
仅成功 Run 提交会话增量；失败、拒绝和取消保留之前的会话版本。

监控器与上下文装配见[评估](../evaluation/README_zh-CN.md)，保留工具 `update_plan`
见[规划](../planning/README_zh-CN.md)。
