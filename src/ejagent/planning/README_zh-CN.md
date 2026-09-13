# 任务规划与能力绑定

[English](README.md) · [模块索引](../README_zh-CN.md)

`ModelTaskPlanner` 将 query 转为 `TaskDefinition`，包含任务文本、目标、验收计划、
初始执行步骤和未知项。应用定义可用的验证能力，模型选择能力并生成任务描述与步骤，
不会生成新的验证函数实现。

## 必需配置与默认值

| 构造参数 | 默认值 | 接入责任 |
| --- | --- | --- |
| `model` | 必填 | `ModelPort`；会作为 Harness 托管资源暴露 |
| `capabilities` | 必填 | 非空 `VerificationCapability` 序列，能力 ID 唯一 |
| `environment` | `None`（空对象） | 应用提供的 JSON 环境描述 |
| `limits` | `PlannerLimits()` | 单次准备的限制，包含纠错 |
| `response_format` | `{"type": "json_object"}` | Provider 参数；设为 `None` 只关闭该参数，不关闭本地 JSON 校验 |

能力包含 `capability_id` 和 `EvaluationCriterion`，后者的 `method`、`evidence_keys`
必须能在评估器中解析。`required=False` 允许模型不选择该能力，true 要求必须选择。
`constraint=False` 将所选验收项绑定为 requirements，true 绑定为 constraints。
最终验收计划仍需至少一个 requirement。

```python
from ejagent.evaluation import EvaluationCriterion
from ejagent.planning import ModelTaskPlanner, VerificationCapability


def create_planner(model):
    return ModelTaskPlanner(
        model,
        capabilities=(
            VerificationCapability(
                capability_id="artifact_shape",
                criterion=EvaluationCriterion(
                    "shape",
                    "Artifact contains answer",
                    "shape",
                    ("artifact",),
                ),
                required=True,
            ),
        ),
        environment={"output_path": "result.json"},
    )
```

按[评估模块](../evaluation/README_zh-CN.md) 注册 `artifact` 和 `shape`，随后向 Harness
传入 `planner=create_planner(model)` 与 `trajectory=monitor`。
`environment` 是应用提供的数据，不是自动刷新的文件系统视图。

## 预算与输出纠错

| `PlannerLimits` 字段 | 默认值 | 范围 |
| --- | --- | --- |
| `timeout_seconds` | `60.0` | 整次准备，包含重试 |
| `max_tokens` | `16_384` | 准备期间所有请求上报的总 token |
| `max_output_tokens` | `4096` | 单次请求，同时受剩余 token 预算限制 |
| `max_prompt_bytes` | `65_536` | 每次组装的提示词，包含 schema 和纠错上下文 |
| `max_response_bytes` | `32_768` | 单次输出 |
| `max_format_retries` | `1` | 额外格式或 schema 纠错次数；0 禁止重试 |

Planner 在提示词中收到 query、会话、环境、能力目录和工具定义；其模型请求不暴露可调用工具。
输出必须符合内部定义的 Pydantic schema。非法 JSON 或 schema 会收到有限纠错上下文；
合法的 Markdown 包裹 JSON 会被接受，并为后续请求保留格式提醒。
同一实例跨任务准备只保留通用格式提醒，不保留上次错误响应。

usage 缺失、预算耗尽、不支持的目标、未知能力、遗漏必选能力、初始计划引用错误，
均通过 `PlanningError` 停止准备。业务绑定错误不属于格式重试。
Planner 成本与 Actor Run usage 分开记录；已发出的请求可能超出预算。

## 固定验收与可修改步骤

仅当配置了 Planner 且未显式传入 `evaluation_plan` 时，Harness 才自动准备新任务。
`continue_run()` 不调用 Planner。绑定后的验收项保留应用配置的 method、证据 keys、
语义标记和原始描述。初始步骤必须为 pending，并覆盖每个 requirement。

对于已准备任务，Harness 向 Actor 注入 `update_plan` 工具，其参数为 `expected_version`、
`based_on_checkpoint`、`reason`、`steps`。每个步骤包含 `id`、`description`、
`requirement_ids`、`status`（`pending`、`in_progress`、`completed`、`blocked`）。
版本和检查点必须是当前值，步骤只能引用合法验收 ID、覆盖所有要求，并最多有一个执行中步骤。
失败返回工具错误并保留原计划；成功原子更新执行步骤，不改变任务目标和验收计划。
启用规划时不能另行注册同名 `update_plan` 工具。

替换规划实现时，实现 `TaskPlanner.plan(request, cancellation=...) -> PlanningResult`；
Harness 会校验返回的任务和计划。详见[规划设计](../../../docs/task-planning.md)和
[结构化输出](../../../docs/structured-output.md)。
