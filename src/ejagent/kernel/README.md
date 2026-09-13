# Direct Kernel integration

[中文](README_zh-CN.md) · [Module index](../README.md)

`RuntimeKernel` executes one immutable `RunSpec` and returns `RunOutcome`. Most
applications should assemble [AgentHarness](../harness/README.md), which owns
resource lifecycle, preparation, controls, and conversation commits around it.

## Constructor and Run inputs

| Option | Default | Effect |
| --- | --- | --- |
| `model`, `tools` | Required | Ready `ModelPort` and `ToolExecutor` |
| `context` | `None` | `IdentityContextPipeline` |
| `trajectory` | `None` | No checkpoint monitor |
| `clock` | `None` | UTC timestamps |
| `monotonic_clock` | `None` | Monotonic elapsed-time measurement |
| `run(..., cancellation=...)` | `None` | Creates a fresh cancellation token |
| `run(..., controls=...)` | `None` | No steering input source |

`RunSpec` requires `run_id`, `base_revision`, `intent`, `task`, and `messages`.
`TASK` requires nonempty task text; `CONTINUE` requires `task=None`. Defaults are
`limits=RunLimits()`, `configuration_revision="default"`, `metadata={}`,
`evaluation_plan=None`, and `completion_policy=CompletionPolicy()`.
Enforcement needs both a plan and a monitor. Actor budgets and completion retries
are described in [Harness](../harness/README.md).

This function accepts already-started integrations and does not persist its result:

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

Direct callers must start/close resources, supply unique Run IDs and current base
state, and decide whether/how to commit. The Kernel does not invoke a Planner,
restore a store, or schedule follow-ups. Implement `RunControlSource.drain_steering()`
only if supplying your own transient control queue.

## Checkpoints and custom monitoring

`TrajectoryMonitor.capture(signal, cancellation=...)` is async and returns an
object satisfying `TrajectoryCaptureResult`: `checkpoint_id: str`, `verdict: str`,
and `completion_allowed: bool | None`. `close_run(run_id)` releases monitor state.
Normally use `EvaluationMonitor`, which implements this boundary already.

Kernel capture sites include baseline (turn zero), completed tool batches, and proposed
completion. Tool `COMPLETE` results also participate in planned completion checks.
`CheckpointTrigger` additionally defines `VERIFICATION_COMPLETED` and
`EXTERNAL_CHANGE`; enum membership alone does not install a background watcher or
schedule those captures. Signals carry declared plans, bounded completion text,
tool receipt references, and execution cost; they are not raw environment access.

Enforcement consumes `completion_allowed`, not the display verdict string or a
coverage threshold. Capture failure cannot approve completion. The Kernel closes
monitor Run state on exit. See [evaluation](../evaluation/README.md) for collection,
source registration, and failure handling.
