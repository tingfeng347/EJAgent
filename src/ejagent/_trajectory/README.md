# Internal trajectory analysis and projection

[中文](README_zh-CN.md) · [Module index](../README.md)

This private package implements checkpoint analysis and disposable trajectory
context. Its classes are internal integration surfaces, not stable public tuning
APIs. Applications normally use [EvaluationMonitor](../evaluation/README.md).

## Internal configuration surfaces

| Component | Inputs and defaults |
| --- | --- |
| `ShadowTrajectoryAnalyzer` | `max_period=3`, a positive maximum cycle period |
| `ShadowTrajectoryObserver` | Required `checkpoint_source`, `report_sink`; `analyzer=None` uses the default analyzer |
| `OnlineTrajectoryMonitor` | Required `CheckpointEvaluator`; `analyzer=None`, `update_sink=None`, `run_close_sink=None` |
| `TrajectoryContextPipeline` | Required synchronous `source`; `base=None` uses identity, `projector=None` uses `TrajectoryContextProjector` |

`CheckpointEvaluator.evaluate(request, cancellation=...)` returns
`CheckpointEvaluation` with checkpoint facts and acceptance observations.
`checkpoint_source(run_id)` returns an iterable of checkpoints; the shadow
observer's `report_sink(report)` is async. Online `update_sink(update)` and
`run_close_sink(run_id, checkpoints)` are synchronous. This shadow report sink
differs from the synchronous evaluation report sink.
Context `source(request)` returns a matching
`TrajectoryContextFrame` or `None`. Update, report, close, and checkpoint callbacks
are supplied by the integrating code; they are not additional model tools.
`EvaluationMonitor` already assembles the evaluator adapter, online monitor, and
frame buffer. Its public constructor does not expose `max_period` or a custom
projector; changing those requires internal integration rather than a Harness flag.

## State visibility and feedback

Use `monitor.context_pipeline(base=...)` as Harness `context` and the same monitor
as `trajectory`. The v2 projector exposes checkpoint state independently of alerts.
Fresh ordinary observations therefore remain visible even when `feedback` is null.
Missing frames or incomplete evaluation produce an unavailable projection instead
of replaying an old observation as current or inventing a completion score.

Actionable feedback is added for `cycle_confirmed`, `constraint_violated`,
`external_state_changed`, `completion_audit_failed`, and `evaluation_unavailable`.
`facts_updated`, `progress_evaluated`, and `cycle_suspected` do not by themselves
add that intervention feedback. These event rules are fixed implementation policy,
not a configurable event allowlist.

Frames must match the request's Run and turn. Fact references and validity are
checked before projection; invalid frames are context protocol errors. The pipeline
appends a transient instruction after base context and exposes visibility metadata.
Run closure clears buffered frames and analysis state.

## Numeric analysis and control boundaries

Requirement coverage is computed from true requirement observations divided by
the number of requirements; unknown values do not count as satisfied. Progress
and cycle analysis also consider evidence novelty, changes, and causal actions.
There is no public coverage threshold that independently approves completion.
The evaluator checks requirements and constraints, and the Kernel consumes the
monitor's explicit `completion_allowed` under the configured completion policy.
Cycle feedback does not itself deny an Actor action or force a plan update.

See [projection design](../../../docs/trajectory-context-projection.md),
[shadow design](../../../docs/trajectory-shadow-design.md), and
[online integration](../../../docs/trajectory-runtime-readiness.md).
