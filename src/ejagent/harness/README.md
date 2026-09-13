# Harness configuration

[中文](README_zh-CN.md) · [Module index](../README.md)

[AgentHarness](core.py) owns one logical agent's resources, conversation revision,
serialized Runs, and commit decisions. The application supplies ready integrations.

## Constructor options

| Parameter | Default | Meaning |
| --- | --- | --- |
| `agent_id`, `model`, `tools` | Required | Stable session identity, `ModelPort`, `ToolExecutor` |
| `context` | `None` | Identity projection; supply a pipeline for extra context |
| `planner` | `None` | No automatic preparation; requires `trajectory` when configured |
| `trajectory` | `None` | No checkpoint evaluator; usually use `EvaluationMonitor` |
| `completion_policy` | `CompletionPolicy()` | Observe by default |
| `initial_messages` | `()` | Initial conversation, commonly `SystemMessage` instructions |
| `store` | `None` | In-process state only; supply `SessionStore` for durability |
| `observers`, `resources` | `()`, `()` | Audit observers and extra lifecycle-managed objects |
| `limits` | `RunLimits()` | Default Actor budget for each Run |
| `configuration_revision` | `"default"` | Application configuration label captured in Run state |
| `run_id_factory`, `clock` | `None`, `None` | UUID and UTC clock defaults; injectable for embedding/tests |
| `steering_capacity`, `follow_up_capacity` | `16`, `16` | Positive queue capacities |

`async with harness:` starts resources and restores a stored snapshot; shutdown
releases managed resources. Restored conversation takes precedence over initial
messages. Implement `ManagedResource.start/shutdown` only for integrations that
own resources. `RunObserver.observe(audit)` receives completed Run audit data.

## Completion and execution limits

`CompletionPolicy(mode=CompletionMode.OBSERVE, max_retries=2)` keeps evaluation
advisory. `ENFORCE` requires a monitor and an evaluation plan, and accepts completion
only when the monitor explicitly allows it. Unknown or conflicting evidence does
not approve completion. `max_retries` counts additional completion attempts in the
same Run; exhaustion fails the Run without advancing conversation revision.

`RunLimits` defaults are `max_turns=20`, `max_tokens=None`, and
`max_repeated_tool_calls=3`. The repeat guard counts consecutive identical tool
calls by name and arguments, including calls within a batch. Token limits depend
on reported usage and are checked at execution boundaries; they are not a hard provider billing cap.
Planner and Judge budgets are configured separately.

## Assembly and per-Run overrides

This factory accepts an existing model and executor; it needs no credentials itself:

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

`run(task, limits=..., metadata=..., evaluation_plan=...)` replaces the default
limits for that Run. An explicit plan bypasses automatic planning. A configured
Planner otherwise prepares new tasks before Actor execution. `continue_run()` adds
no user message and does not invoke the Planner; provide a plan for enforced
continuations. Plans are not inherited across Runs or restored sessions.

During a Run, `steer(text)` queues transient instructions for a safe point;
`follow_up(task, evaluation_plan=...)` queues a later Run and returns a handle.
Check control receipts for rejection, including full queues and inactive Runs.
`cancel(reason)` requests cooperative cancellation. Only successful Runs commit
conversation deltas; failed, rejected, or cancelled Runs preserve the prior revision.

See [evaluation](../evaluation/README.md) for monitor/context assembly and
[planning](../planning/README.md) for the reserved `update_plan` tool.
