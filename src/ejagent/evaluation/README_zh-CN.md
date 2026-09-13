# 证据、验证与完成验收

[English](README.md) · [模块索引](../README_zh-CN.md)

`GoalEvaluator` 从环境采集证据并运行已注册的检查，`EvaluationMonitor` 将其接到 Harness
检查点。应用在执行前注册能力；证据值在执行过程中采集。

## 注册证据源与规则

构造时必须提供两个映射；计划不需要其中某类能力时，对应映射可为空。
以下名称必须在 Actor 执行前完成绑定：

```text
criterion.evidence_keys -> sources[name]   -> EvidenceSnapshot
criterion.method        -> verifiers[name] -> CheckResult（非语义项）
criterion.guard_method  -> verifiers[name] -> 可选前置规则（语义项）
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

向 `AgentHarness` 传入 `trajectory=monitor` 和
`context=monitor.context_pipeline(base=...)`，向 `run()` 传入 `evaluation_plan=plan`。
监控与上下文注入是两个独立接入动作。需要强制通过验收时，另加
`CompletionPolicy(CompletionMode.ENFORCE)`；默认仅观察。见 [Harness](../harness/README_zh-CN.md)。

显式提供计划时，应像上面一样自行调用 `validate_plan()`。Harness 会在自动 Planner 准备过程中
执行此绑定校验，但不会自动预检每个显式计划。执行评估时才遇到未配置依赖，可能表现为 unknown。

## 自定义证据源与验证函数

`EvidenceSource` 是 Protocol，要求异步 `revision(signal, cancellation=...)`、
异步 `read(...) -> EvidenceSnapshot` 和同步 `close_run(run_id)`。
`EvidenceSnapshot` 包含 `revision`、JSON `value`、`location` 和带时区的 `observed_at`
（默认当前 UTC 时间）。读取应只观测环境；相关状态发生变化时需改变 revision，包括恢复成旧内容。
用于语义比较的内容 identity 单独计算。

下面的小型证据源通过 `publish()` 接收应用实际状态；初始 false 不代表伪造的成功证据：

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

`Verifier` 是异步函数类型别名，并非 Protocol 类：
`(VerificationRequest, CancellationToken) -> Awaitable[CheckResult]`。
请求包含验收项、检查点信号、声明的证据快照和之前的报告。自定义规则与 `json_fields()`
一样注册：

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

所有外部依赖都应通过证据源声明；验证函数偷偷读取其他状态会导致缓存结果失真。
已知结果必须引用声明的证据 key。`pass/fail` 对应真/假，`unknown/conflict` 保持未决。
`EvidenceUnavailable`、I/O 失败和超时产生 unknown，非法集成返回值抛出
`EvaluationProtocolError`。采集和验证前后的版本屏障会使依赖变化后的结果失效，但不会锁住环境。

## 验收项与评估器配置

| 配置 | 默认值 | 作用 |
| --- | --- | --- |
| `semantic` | `False` | 普通规则使用 `method`；true 要求 `semantic_judge` |
| `guard_method` | `None` | 仅用于语义项；确定性前置规则通过后才调用 Judge |
| `completion_only` | `False` | true 时，在出现完成候选之前保持 unknown |
| `EvaluationPlan.constraints`、`artifact_refs` | `()`、`()` | 额外约束和产物引用；requirements 必须非空 |
| `timeout_seconds` | `2.0` | 每次受限证据读取、版本读取或验证操作的超时 |
| `max_concurrency` | `4` | 受限评估操作的并发数 |
| `max_evidence_bytes` | `1_048_576` | 每份注册证据源快照的序列化 value 大小限制 |
| `semantic_judge` | `None` | 需提供 Judge 且验收项启用语义检查才调用 LLM |
| `report_sink` | `None` | 可选的同步 `EvaluationReport` 回调 |
| `EvaluationMonitor.update_sink` | `None` | 可选的同步轨迹更新回调 |

每个验收项需要唯一 `criterion_id`、`description`、`method` 和非空且不重复的 `evidence_keys`。
语义项的 `method` 标记检查方法，不用于选择确定性验证函数。
`$completion` 是受限最终文本的保留证据源，不能自行注册；它使用独立的固定上限 65,536 字符，
完成文本缺失或被截断时不可用。
观察模式下没有验收计划会得到 `not_evaluated`。

## 内置证据源与规则

| 证据源 | 应用提供 | 默认限制与证据内容 |
| --- | --- | --- |
| `FileEvidenceSource` | UTF-8 文件路径 `path` | `max_bytes=262_144`；`exists`、`text` |
| `WorkspaceEvidenceSource` | 根目录 `root`、1–128 个唯一相对路径 `paths` | `max_bytes=262_144` 为总量；`files` 映射；不会自动发现仓库文件 |
| `CommandEvidenceSource` | 工作区证据源 `workspace`、固定参数元组 `command` | `timeout_seconds=30.0`，每个输出流 `max_output_bytes=65_536`；退出码、输出、截断标记、依赖版本 |
| `ProbeEvidenceSource` | 可选的两个探针名称；应用调用 `record()` | `names=("probe_a", "probe_b")`，每 Run `max_records=1024`；完成标记和 `overlapped` |

`record(run_id, tool_name, started_at=..., finished_at=..., cancelled=False)` 记录已完成区间；
已取消记录会被忽略，记录超限使证据不可用。
`file_exists` 检查存在性；`json_fields(*fields)` 检查 JSON 和顶层 key 存在性，不校验字段类型；
`boolean_field(field)` 保留不同来源间的冲突；`command_succeeded` 仅检查退出码。
这些内置函数也需要显式注册。

命令在工作区根目录执行，不经过 shell。应用应配置观测性命令并列出全部相关文件依赖；
其他外部依赖变化时调用 `invalidate()`。版本未变时复用 Run 内命令结果。
命令需要超过默认 2 秒时，应同时提高评估器操作超时；命令自身的 30 秒限制不会覆盖外层限制。

## 模型 Judge 预算与输出

`ModelJudge(model, limits=JudgeLimits(), response_format={"type": "json_object"})`
不执行工具。限制如下：

| 字段 | 默认值 | 范围 |
| --- | --- | --- |
| `max_requests`、`max_tokens` | `8`、`16_384` | 每个 Run，包含纠错请求 |
| `max_output_tokens` | `1024` | 单次模型请求 |
| `max_prompt_bytes`、`max_response_bytes` | `32_768`、`16_384` | 每次输入和输出 |
| `timeout_seconds` | `30.0` | 一次 Judge 评估，包括纠错 |
| `max_concurrency` | `2` | 跨 Run；同一 Run 内请求串行 |
| `max_format_retries` | `1` | 每次评估允许的额外纠错次数 |

JSON 语法错误、schema 不匹配、验收项或证据引用错误会进入有限纠错上下文。
合法 Markdown 包裹 JSON 会被接受，并为后续请求设置格式提醒。
合法的负面结论不会被当作格式错误重试。预算耗尽或 usage 缺失产生 unknown；usage 缺失还会
阻止该 Run 的后续 Judge 请求。已发出的请求可能超出剩余 token 预算。
不支持此参数的 Provider（包括原生 Anthropic 适配器）需设 `response_format=None`。
详见[结构化输出](../../../docs/structured-output.md)。

## 报告与清理

`report_sink=JsonlEvaluationJournal(path)` 独立于 `SessionStore` 保存完整报告；未配置时，
`latest_report(run_id)` 仅提供活动 Run 的最新报告。Harness 会清理 Run 内评估状态；
独立使用时必须在 `finally` 调用 `evaluator.close_run(run_id)`。
拥有连接的证据源需由应用显式管理生命周期，例如加入 Harness `resources`；评估器会将 Judge
模型作为托管依赖暴露。详见[评估设计](../../../docs/evaluation.md)。
