# 会话存储与审计持久化

[English](README.md) · [模块索引](../README_zh-CN.md)

`JsonlSessionStore` 实现 `SessionStore` 和 `AuditReader`。
Harness 未配置 store 时，仅在当前进程中保留会话；持久化需由应用显式选择。

## 配置

| 参数 | 默认值 | 作用 |
| --- | --- | --- |
| `root` | 必填 | 各智能体会话日志的目录，展开 `~` |
| `lock_timeout` | `10.0` 秒 | 非负锁等待预算；`0` 不等待，`None` 不设等待超时 |

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

复用相同 `agent_id` 和 root，可在 Harness 启动时恢复会话。
存储使用进程内锁与 POSIX 文件咨询锁，因此受锁保护的操作要求 POSIX。
锁竞争超时抛出 `SessionStoreLockTimeoutError`，非法序列化使用
`SessionStoreSerializationError`。内部轮询间隔不是 `JsonlSessionStore` 的公共参数。

## 提交与恢复保证

日志提交保存 Run 结果及审计，仅 `COMPLETED` 推进会话版本并追加增量。
失败、拒绝和取消 Run 仍可写入审计，同时保留原会话。因此，调用 `commit()` 不一定代表会话改变。

写入在锁内比较完整基础会话和版本。过期基础状态、Run ID 被用于不同提交时抛出
`SessionConflictError`；重放相同提交具有幂等性。可恢复末尾不完整日志，
但会拒绝损坏的完整记录及不支持的 schema 版本。不要通过手改日志改变配置，也不要将日志当作可执行指令。

`load_audit(agent_id)` 返回不可变 Run 审计。完整评估报告使用
[评估模块](../evaluation/README_zh-CN.md)的 `JsonlEvaluationJournal(path)` 单独保存；
SessionStore 审计引用不能替代报告归档。恢复会话不会恢复正在执行的 Run、待消费控制或验收计划。

## 实现其他后端

实现异步 `load(agent_id) -> SessionSnapshot | None` 和
`commit(SessionCommit) -> SessionSnapshot`，保持原子比较提交、幂等性、会话顺序、审计，
以及仅成功时推进版本的语义。Harness 会校验返回快照。
需要独立 `AuditReader` 能力时再提供异步 `load_audit(agent_id)`。
数据库连接可实现 `ManagedResource.start/shutdown`；内置文件存储不需要此生命周期。

应用负责存储和报告的位置、访问权限与保留周期；日志包含应用消息和观测数据。
另见[协议](../contracts/README_zh-CN.md)和[使用指南](../../../docs/usage-guide.md)。
