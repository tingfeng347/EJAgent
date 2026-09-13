# Contracts and extension interfaces

[中文](README_zh-CN.md) · [Module index](../README.md)

These provider-neutral dataclasses and protocols define EJAgent's public
boundaries. Applications implement behavior, not replacement definitions of the
protocols. Most interfaces already have usable built-in adapters.

## Application-supplied behavior

| Interface | Contract | When to implement it yourself |
| --- | --- | --- |
| `ModelPort` | `stream(request, cancellation=...)` yields model events | Unsupported model backend; otherwise use a Provider |
| `ToolExecutor` | `definitions`, async `execute(...) -> ToolExecutionResult` | Custom dispatch; otherwise register `FunctionTool` or MCP |
| `TaskPlanner` | async `plan(...) -> PlanningResult` | Custom planning; otherwise configure `ModelTaskPlanner` |
| `ContextPipeline` | async `build(...) -> ContextView` | Custom projection; built-in identity/skills/compaction are available |
| `ContextCompactor` | async `compact(...) -> ContextCompactionOutput` | Required when using `DerivedCompactionPipeline`; no built-in summarizer |
| `SessionStore` | async `load(agent_id)`, `commit(SessionCommit)` | Custom durable backend; JSONL is built in |
| `AuditReader` | async `load_audit(agent_id)` | Optional audit query API; JSONL implements it |
| `RunObserver` | async `observe(RunAudit)` | Optional reporting after execution |
| `ManagedResource` | async `start()`, `shutdown()` | Integrations owning connections or other resources |
| `RunControlSource` | `drain_steering()` | Direct Kernel integrations; Harness normally supplies it |

`EvidenceSource` and the async callable alias `Verifier` live in
[evaluation](../evaluation/README.md). `TrajectoryMonitor` and the structural
`TrajectoryCaptureResult` return contract live in [kernel](../kernel/README.md).
They need custom implementations only when built-in evaluation is insufficient.

## Configuration versus runtime data

`RunLimits`, `CompletionPolicy`, `EvaluationCriterion`, and `EvaluationPlan` are
configuration inputs. Their defaults and combinations are described in
[harness](../harness/README.md) and [evaluation](../evaluation/README.md).
`ModelRequest` optionally carries `tools=()`, `max_output_tokens=None`, and
`response_format=None`; callers and providers determine which features apply.

`ToolControl` is an application-requested execution decision, documented in
[tools](../tools/README.md). `StepStatus` is plan state; `RunStatus`, `StopReason`,
`FailureCode`, and control receipt statuses describe results, not feature toggles.

Messages and snapshots are immutable. Conversation contains user/system/assistant
messages and tool results; disposable context may additionally contain summaries
and transient instructions. Current message content is text, without an image
input contract. Preserve tool call/result ordering and JSON-compatible data.

## Validation and failure responsibility

Public dataclasses validate their invariants. Planner/Judge model JSON is validated
using separate internal Pydantic models, followed by capability/reference checks;
see [structured output](../../../docs/structured-output.md).

Adapters must honor cancellation and return the declared types. Report expected
operational failures using the relevant error type; protocol violations are
integration defects. An error's `retryable` flag is metadata, not a generic retry
engine. Lifecycle and retry behavior depend on the consuming module.
