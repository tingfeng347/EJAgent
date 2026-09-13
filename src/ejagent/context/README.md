# Context projection and compaction

[中文](README_zh-CN.md) · [Module index](../README.md)

Context pipelines build a disposable `ContextView` before each Actor model request.
They do not rewrite committed conversation. Input separates committed messages,
pending Run messages, and transient instructions.

## Options and required implementations

| Pipeline | Configuration | Behavior |
| --- | --- | --- |
| `IdentityContextPipeline()` | No options; Kernel default | Committed + pending + transient messages |
| `DerivedCompactionPipeline(compactor, minimum_messages=20)` | Required `ContextCompactor`; positive threshold | Summarize committed history once its eligible message count reaches the threshold |
| `SkillsContextPipeline(skills_root, base=None)` | Required local root; identity base by default | Add skill index and explicit selection, then invoke base |
| `monitor.context_pipeline(base=None)` | Existing `EvaluationMonitor`; identity base by default | Project trajectory state/feedback into Actor context |

Compaction counts committed messages after leading system messages; it is not a
token threshold and does not compact pending messages. Leading system messages,
pending tool protocol, and transient instructions remain. The summary is derived
again on eligible builds; there is no built-in summary cache or automatic model
summarizer. Summary generation cost is not automatically added to Actor usage.

## Supply a compactor

Implement async `compact(request, cancellation=...) -> ContextCompactionOutput`.
The output needs nonempty `content` and `compactor_id`. The application chooses
the summarization function, model, budget, and preservation policy. This adapter
accepts an application-owned async summarization function:

```python
from collections.abc import Awaitable, Callable
from ejagent.context import DerivedCompactionPipeline
from ejagent.contracts import (
    CancellationToken,
    ContextCompactionOutput,
    ContextCompactionRequest,
)


class ApplicationCompactor:
    def __init__(self, summarize: Callable[[ContextCompactionRequest], Awaitable[str]]):
        self._summarize = summarize

    async def compact(
        self, request: ContextCompactionRequest, *, cancellation: CancellationToken
    ) -> ContextCompactionOutput:
        content = await cancellation.run(self._summarize(request))
        return ContextCompactionOutput(content, "application-summary-v1")


def compacting_context(summarize):
    return DerivedCompactionPipeline(
        ApplicationCompactor(summarize), minimum_messages=20
    )
```

Raise `ContextCompactorError` for expected summarization failures; the pipeline
maps this to `COMPACTION_FAILED`. Undeclared exceptions or wrong return types are
protocol errors. Failure does not silently fall back to dropping history.

## Compose context for an evaluated task

This factory requires an existing monitor, skill directory, and compactor:

```python
from ejagent.context import DerivedCompactionPipeline, SkillsContextPipeline


def evaluated_context(monitor, skills_root, compactor):
    base = SkillsContextPipeline(
        skills_root,
        base=DerivedCompactionPipeline(compactor, minimum_messages=20),
    )
    return monitor.context_pipeline(base=base)
```

Pass the returned pipeline as Harness `context` and the same monitor as
`trajectory`. Wrappers preserve base context and transient instructions. They
forward lifecycle to managed children; Harness normally starts the outer wrapper.

To replace projection, implement `ContextPipeline.build(request, cancellation=...)`
and preserve `run_id`, `source_revision`, and `turn` in the returned view. Context
metadata is inspectable data, not automatically a model message. Planner and Judge
construct their own contexts; this pipeline configures Actor requests.

See [skills](../skills/README.md), [evaluation](../evaluation/README.md), and
[trajectory projection](../_trajectory/README.md) for injection rules and cleanup.
