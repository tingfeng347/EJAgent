# Evidence, verification, and completion evaluation

[中文](README_zh-CN.md) · [Module index](../README.md)

`GoalEvaluator` collects environment evidence and applies registered checks.
`EvaluationMonitor` connects it to Harness checkpoints. The application registers
capabilities before execution; evidence values are collected during execution.

## Register sources and rules

Both constructor mappings are required, although either can be empty when the
plan does not need it. Names must resolve before Actor execution:

```text
criterion.evidence_keys -> sources[name]   -> EvidenceSnapshot
criterion.method        -> verifiers[name] -> CheckResult (non-semantic)
criterion.guard_method  -> verifiers[name] -> optional guard (semantic)
```

```python
from ejagent.evaluation import (
    EvaluationCriterion,
    EvaluationMonitor,
    EvaluationPlan,
    FileEvidenceSource,
    GoalEvaluator,
    json_fields,
)

evaluator = GoalEvaluator(
    sources={"artifact": FileEvidenceSource("result.json")},
    verifiers={"shape": json_fields("answer")},
)
plan = EvaluationPlan(
    goal="Write an answer artifact",
    version="v1",
    requirements=(
        EvaluationCriterion(
            "answer",
            "JSON contains answer",
            "shape",
            ("artifact",),
        ),
    ),
)
monitor = EvaluationMonitor(evaluator)
evaluator.validate_plan(plan)
```

Supply `trajectory=monitor` and `context=monitor.context_pipeline(base=...)` to
`AgentHarness`, then pass `evaluation_plan=plan` to `run()`. Monitoring and context
injection are separate connections. Add `CompletionPolicy(CompletionMode.ENFORCE)`
to require approval; the default only observes. See [Harness](../harness/README.md).

Call `validate_plan()` yourself for an explicitly supplied plan, as above. Harness
performs this binding check during automatic Planner preparation; it does not
automatically preflight every explicit plan. Unconfigured dependencies encountered
during evaluation can instead produce unknown results.

## Custom evidence and verifier contracts

`EvidenceSource` is a Protocol with async `revision(signal, cancellation=...)`,
async `read(...) -> EvidenceSnapshot`, and synchronous `close_run(run_id)`.
`EvidenceSnapshot` carries `revision`, JSON `value`, `location`, and an aware
`observed_at` timestamp (UTC now by default). Reads are observational. Change the
revision on relevant mutations, including restoration of old content; content
identity is computed separately for semantic comparison.

This small source receives actual application state through `publish()`; the
initial false state is not a fabricated successful observation:

```python
from ejagent.contracts import CancellationToken
from ejagent.evaluation import EvidenceSnapshot
from ejagent.kernel.trajectory import CheckpointSignal


class ReadySource:
    def __init__(self) -> None:
        self._revision = 0
        self._ready = False

    def publish(self, ready: bool) -> None:
        self._revision += 1
        self._ready = ready

    async def revision(
        self, signal: CheckpointSignal, *, cancellation: CancellationToken
    ) -> str:
        cancellation.raise_if_cancelled()
        return str(self._revision)

    async def read(
        self, signal: CheckpointSignal, *, cancellation: CancellationToken
    ) -> EvidenceSnapshot:
        cancellation.raise_if_cancelled()
        return EvidenceSnapshot(
            str(self._revision),
            {"ready": self._ready},
            "application:ready",
        )

    def close_run(self, run_id: str) -> None:
        pass  # Application state persists; no Run-local cache to clear.
```

`Verifier` is an async callable type alias, not a Protocol class:
`(VerificationRequest, CancellationToken) -> Awaitable[CheckResult]`. The request
contains the criterion, checkpoint signal, declared evidence snapshots, and prior
report. A custom rule can be registered exactly like `json_fields()`:

```python
from collections.abc import Mapping
from ejagent.contracts import CancellationToken
from ejagent.evaluation import CheckResult, EvaluationStatus, VerificationRequest


async def verify_ready(
    request: VerificationRequest, cancellation: CancellationToken
) -> CheckResult:
    cancellation.raise_if_cancelled()
    values = [s.value for s in request.evidence.values()]
    if not values or any(
        not isinstance(v, Mapping) or type(v.get("ready")) is not bool for v in values
    ):
        return CheckResult(
            EvaluationStatus.UNKNOWN,
            "Missing ready state",
            missing_evidence=request.criterion.evidence_keys,
        )
    passed = all(v["ready"] for v in values)
    return CheckResult(
        EvaluationStatus.PASS if passed else EvaluationStatus.FAIL,
        "Application readiness checked",
        request.criterion.evidence_keys,
    )


ready_source = ReadySource()
ready_evaluator = GoalEvaluator(
    sources={"state": ready_source},
    verifiers={"ready": verify_ready},
)
```

Declare all external dependencies through sources; reading hidden state in a
verifier can make cached results incorrect. Known results cite declared evidence
keys. `pass/fail` map to true/false; `unknown/conflict` remain undecided.
`EvidenceUnavailable`, I/O failures, and timeouts yield unknown. Invalid integration
results raise `EvaluationProtocolError`. Revision barriers surround collection and
verification; changed dependencies invalidate results. This does not lock the environment.

## Criterion and evaluator options

| Option | Default | Effect |
| --- | --- | --- |
| `semantic` | `False` | Ordinary rules use `method`; true requires `semantic_judge` |
| `guard_method` | `None` | Semantic-only deterministic guard; Judge runs only after guard pass |
| `completion_only` | `False` | When true, remain unknown until a completion candidate |
| `EvaluationPlan.constraints`, `artifact_refs` | `()`, `()` | Additional conditions and artifact references; requirements must be nonempty |
| `timeout_seconds` | `2.0` | Timeout per bounded source/revision/verifier operation |
| `max_concurrency` | `4` | Concurrent bounded evaluator operations |
| `max_evidence_bytes` | `1_048_576` | Serialized value bound per registered-source snapshot |
| `semantic_judge` | `None` | No LLM evaluation unless supplied and selected by a criterion |
| `report_sink` | `None` | Optional synchronous `EvaluationReport` callback |
| `EvaluationMonitor.update_sink` | `None` | Optional synchronous trajectory update callback |

Every criterion needs a unique `criterion_id`, `description`, `method`, and nonempty unique
`evidence_keys`. For semantic criteria `method` labels the check; it does not select
a deterministic verifier. `$completion` is reserved for bounded final text and
must not be registered as a source. Its separate fixed bound is 65,536 characters;
absent or truncated completion text is unavailable. No plan means `not_evaluated`
in observe mode.

## Built-in sources and rules

| Source | Application supplies | Defaults and observed value |
| --- | --- | --- |
| `FileEvidenceSource` | UTF-8 file `path` | `max_bytes=262_144`; `exists`, `text` |
| `WorkspaceEvidenceSource` | `root` and 1–128 unique relative `paths` | `max_bytes=262_144` total; `files` mapping; no automatic repository discovery |
| `CommandEvidenceSource` | `workspace` source and fixed `command` argument tuple | `timeout_seconds=30.0`, `max_output_bytes=65_536` per output stream; exit code, output, truncation flag, dependency revision |
| `ProbeEvidenceSource` | Optional pair of probe names; application calls `record()` | `names=("probe_a", "probe_b")`, `max_records=1024` per Run; completion flags and `overlapped` |

`record(run_id, tool_name, started_at=..., finished_at=..., cancelled=False)` records
completed intervals; cancelled records are ignored. Overflow makes evidence unavailable.
`file_exists` checks presence; `json_fields(*fields)` checks JSON and top-level key
presence, not field types; `boolean_field(field)` preserves conflict between sources;
`command_succeeded` checks exit status only. Register these functions explicitly.

Commands run without a shell in the workspace root. Configure observational
commands and include every relevant file dependency; call `invalidate()` for
external dependency changes. Unchanged revisions reuse Run-local command results.
Raise the evaluator's operation timeout when a command needs more than its default
2 seconds; the command's own 30-second timeout does not override that outer limit.

## Model Judge budgets and output

`ModelJudge(model, limits=JudgeLimits(), response_format={"type": "json_object"})`
does not execute tools. Limits are:

| Field | Default | Scope |
| --- | --- | --- |
| `max_requests`, `max_tokens` | `8`, `16_384` | Per Run, including correction requests |
| `max_output_tokens` | `1024` | Per model request |
| `max_prompt_bytes`, `max_response_bytes` | `32_768`, `16_384` | Each prompt/response |
| `timeout_seconds` | `30.0` | One judge evaluation including corrections |
| `max_concurrency` | `2` | Across Runs; requests within a Run serialize |
| `max_format_retries` | `1` | Additional corrections per evaluation |

Malformed JSON, schema failures, and invalid criterion/evidence references receive
bounded correction context. Valid fenced JSON is accepted and sets a subsequent
format reminder. Valid negative verdicts are not retried as formatting failures.
Budget exhaustion or missing usage produces unknown; missing usage also blocks
further requests in that Run. An in-flight request can exceed the remaining token
budget. Set `response_format=None` for providers without this option, including
the native Anthropic adapter. See [structured output](../../../docs/structured-output.md).

## Reports and cleanup

`report_sink=JsonlEvaluationJournal(path)` persists full reports separately from
`SessionStore`; otherwise `latest_report(run_id)` exposes the latest active report.
Harness closes Run-local evaluator state. Standalone users must call
`evaluator.close_run(run_id)` in `finally`. Sources owning connections need explicit
application lifecycle management, for example Harness `resources`; the evaluator
exposes the Judge model as a managed dependency. See [evaluation design](../../../docs/evaluation.md).
