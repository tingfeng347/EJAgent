# Session storage and audit persistence

[中文](README_zh-CN.md) · [Module index](../README.md)

`JsonlSessionStore` implements `SessionStore` and `AuditReader`. Harness without a
store retains conversation only in its own process; persistence is an explicit
application choice.

## Configuration

| Parameter | Default | Meaning |
| --- | --- | --- |
| `root` | Required | Directory for agent session journals; `~` is expanded |
| `lock_timeout` | `10.0` seconds | Nonnegative lock-acquisition budget; `0` tries without waiting, `None` waits without a timeout |

```python
from ejagent import AgentHarness
from ejagent.storage import JsonlSessionStore


def persistent_harness(model, tools, root):
    return AgentHarness(
        agent_id="assistant",
        model=model,
        tools=tools,
        store=JsonlSessionStore(root, lock_timeout=10.0),
    )
```

Reuse `agent_id` and root to restore a conversation at Harness startup. The store
uses process-local locks plus POSIX advisory file locks, so this implementation
requires POSIX for locked operations. Lock contention raises
`SessionStoreLockTimeoutError`; invalid serialization uses
`SessionStoreSerializationError`. The internal polling interval is not a public
`JsonlSessionStore` parameter.

## Commit and recovery guarantees

A journal commit stores the Run outcome and audit. Only `COMPLETED` advances the
conversation revision and appends its delta. Failed, rejected, and cancelled Runs
can still be recorded in audit while preserving conversation. A `commit()` call
therefore does not necessarily mean that conversation changed.

Writes compare the complete base conversation and revision under the lock. Stale
bases and conflicting reuse of a Run ID raise `SessionConflictError`; replaying an
identical commit is idempotent. Incomplete final journal tails can be recovered;
malformed complete records and unsupported schema versions are rejected. Do not
edit journals to change configuration or treat them as executable instructions.

`load_audit(agent_id)` returns immutable Run audits. Full evaluation reports use
`JsonlEvaluationJournal(path)` from [evaluation](../evaluation/README.md), a separate
sink; SessionStore audit references do not replace that report archive. Restoring
conversation does not restore a live Run, pending controls, or its acceptance plan.

## Implement another backend

Implement async `load(agent_id) -> SessionSnapshot | None` and
`commit(SessionCommit) -> SessionSnapshot`. Preserve atomic compare-and-commit,
idempotency, conversation ordering, audit, and success-only revision advancement.
Harness validates returned snapshots. Add async `load_audit(agent_id)` only if
offering the separate `AuditReader` capability. Database connections may implement
`ManagedResource.start/shutdown`; the built-in file store needs no such lifecycle.

The application chooses storage/report locations, access, and retention; journals
contain application messages and observed data. See [contracts](../contracts/README.md)
and [usage guide](../../../docs/usage-guide.md).
