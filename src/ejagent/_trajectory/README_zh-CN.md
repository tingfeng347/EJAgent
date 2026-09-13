# 内部轨迹分析与投影

[English](README.md) · [模块索引](../README_zh-CN.md)

此私有包实现检查点分析和临时轨迹上下文。类属于内部集成接口，不是稳定的公共调参 API。
应用通常使用 [EvaluationMonitor](../evaluation/README_zh-CN.md)。

## 内部配置入口

| 组件 | 输入与默认值 |
| --- | --- |
| `ShadowTrajectoryAnalyzer` | `max_period=3`，正整数的最大循环周期 |
| `ShadowTrajectoryObserver` | 必填 `checkpoint_source`、`report_sink`；`analyzer=None` 使用默认分析器 |
| `OnlineTrajectoryMonitor` | 必填 `CheckpointEvaluator`；`analyzer=None`、`update_sink=None`、`run_close_sink=None` |
| `TrajectoryContextPipeline` | 必填同步 `source`；`base=None` 默认直接投影，`projector=None` 使用 `TrajectoryContextProjector` |

`CheckpointEvaluator.evaluate(request, cancellation=...)` 返回包含检查点事实与验收观察的
`CheckpointEvaluation`。`checkpoint_source(run_id)` 返回检查点 iterable；shadow observer
的 `report_sink(report)` 是异步回调。在线监控器的 `update_sink(update)` 和
`run_close_sink(run_id, checkpoints)` 是同步回调。此 shadow 报告回调与同步的评估报告回调不同。
上下文 `source(request)` 返回匹配的 `TrajectoryContextFrame` 或 `None`。
更新、报告、关闭和检查点回调由接入代码提供，不是额外模型工具。
`EvaluationMonitor` 已装配评估器适配层、在线监控器和 frame 缓冲。
其公共构造器不暴露 `max_period` 或自定义 projector；调整这些需要内部集成，不能直接给 Harness 加开关。

## 状态可见性与反馈

将 `monitor.context_pipeline(base=...)` 传为 Harness `context`，同一个监控器传为
`trajectory`。v2 projector 将检查点状态与告警分别处理：即使 `feedback` 为 null，
新的普通观察仍然可见。缺失 frame 或评估不完整时投影为 unavailable，
不会将旧观察冒充当前状态，也不会编造完成分数。

以下事件会加入可行动反馈：`cycle_confirmed`、`constraint_violated`、
`external_state_changed`、`completion_audit_failed`、`evaluation_unavailable`。
`facts_updated`、`progress_evaluated`、`cycle_suspected` 本身不会加入该干预反馈。
这些事件规则是固定实现策略，不是可配置的事件白名单。

frame 必须匹配请求的 Run 和 turn。投影前校验事实引用与有效性；非法 frame 属于上下文协议错误。
pipeline 在 base 上下文之后追加临时指令，并暴露可见性 metadata。
Run 关闭会清理缓冲 frame 和分析状态。

## 数值分析与控制边界

要求覆盖率由已观察为 true 的要求数除以要求总数计算，未知项不计为满足。
进度和循环分析还会考虑证据新颖性、变化及因果动作。
没有可独立批准完成的公共覆盖率阈值。评估器检查 requirements 和 constraints，
Kernel 按配置的完成策略消费监控器明确给出的 `completion_allowed`。
循环反馈本身不会拒绝 Actor 动作，也不会强制更新计划。

详见[投影设计](../../../docs/trajectory-context-projection.md)、
[轨迹分析设计](../../../docs/trajectory-shadow-design.md)及
[在线集成](../../../docs/trajectory-runtime-readiness.md)。
