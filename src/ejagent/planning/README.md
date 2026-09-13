# Task planning and capability binding

[中文](README_zh-CN.md) · [Module index](../README.md)

`ModelTaskPlanner` turns a query into a `TaskDefinition`: task text, goal,
acceptance plan, initial execution steps, and unknowns. The application defines
available verification capabilities; the model selects them and generates the
task-specific descriptions and steps. It cannot create new verifier implementations.

## Required setup and defaults

| Constructor option | Default | Responsibility |
| --- | --- | --- |
| `model` | Required | `ModelPort`; exposed as a Harness-managed resource |
| `capabilities` | Required | Nonempty sequence of `VerificationCapability`, unique capability IDs |
| `environment` | `None` (empty object) | Application-provided JSON environment description |
| `limits` | `PlannerLimits()` | Bounds for one preparation, including corrections |
| `response_format` | `{"type": "json_object"}` | Provider option; `None` disables this option, not local JSON validation |

A capability supplies `capability_id` and an `EvaluationCriterion` whose `method`
and `evidence_keys` resolve in the evaluator. `required=False` allows omission;
`required=True` requires selection. `constraint=False` binds selected criteria as
requirements; true binds them as constraints. The resulting acceptance plan still
needs at least one requirement.

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

Register `artifact` and `shape` as shown in [evaluation](../evaluation/README.md).
Then pass `planner=create_planner(model)` and `trajectory=monitor` to Harness.
`environment` is supplied data, not an automatically refreshed filesystem view.

## Limits and output recovery

| `PlannerLimits` field | Default | Scope |
| --- | --- | --- |
| `timeout_seconds` | `60.0` | Entire preparation, including retries |
| `max_tokens` | `16_384` | Total reported tokens across preparation requests |
| `max_output_tokens` | `4096` | Per request, capped by remaining token budget |
| `max_prompt_bytes` | `65_536` | Each assembled prompt, including schema and correction context |
| `max_response_bytes` | `32_768` | Each response |
| `max_format_retries` | `1` | Additional format/schema corrections; zero disables retries |

The Planner receives the query, conversation, environment, capability catalog,
and tool definitions as prompt data. Its model request exposes no callable tools.
Output must satisfy the internally defined Pydantic schema. Invalid JSON/schema
receives bounded correction context; valid fenced JSON is accepted with a reminder
for subsequent requests. Only the generic format reminder persists across
preparations on this instance, not the previous malformed response.

Missing usage, exhausted budgets, unsupported goals, unknown capabilities, omitted
required capabilities, or invalid initial plan references stop preparation with
`PlanningError`. Business binding failures are not format retries. Planner cost is
reported separately from Actor Run usage; an in-flight call can exceed its budget.

## Fixed acceptance and revisable steps

Harness prepares a new task only when a Planner is configured and no explicit
`evaluation_plan` was supplied. `continue_run()` does not call the Planner.
The bound acceptance criteria retain the application's method, evidence keys,
semantic flags, and original description. Initial steps must be pending and cover
every requirement.

For prepared tasks, Harness injects `update_plan` into the Actor's tools. Its
arguments are `expected_version`, `based_on_checkpoint`, `reason`, and `steps`.
Each step has `id`, `description`, `requirement_ids`, and `status`
(`pending`, `in_progress`, `completed`, `blocked`). Versions and checkpoint
references must be current; steps must reference valid acceptance IDs, cover
requirements, and have at most one in-progress step. Failure returns a tool error
and preserves the existing plan. Success changes execution steps atomically,
without changing the task goal or acceptance plan. Do not register another
`update_plan` tool when planning is enabled.

To replace planning, implement `TaskPlanner.plan(request, cancellation=...) ->
PlanningResult`; Harness validates the returned task and plan. See
[planning design](../../../docs/task-planning.md) and
[structured output](../../../docs/structured-output.md).
