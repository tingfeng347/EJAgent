# 上下文投影与压缩

[English](README.md) · [模块索引](../README_zh-CN.md)

上下文 pipeline 在每次 Actor 模型请求前生成临时 `ContextView`，不改写已提交会话。
输入区分已提交消息、当前 Run 的待提交消息和临时指令。

## 配置与必需实现

| Pipeline | 配置 | 行为 |
| --- | --- | --- |
| `IdentityContextPipeline()` | 无参数；Kernel 默认 | 已提交 + 待提交 + 临时消息 |
| `DerivedCompactionPipeline(compactor, minimum_messages=20)` | 必须提供 `ContextCompactor`，阈值为正数 | 可压缩的已提交消息数量达到阈值后生成摘要 |
| `SkillsContextPipeline(skills_root, base=None)` | 必须提供本地目录；base 默认直接投影 | 添加 Skills 索引与显式选择内容，再调用 base |
| `monitor.context_pipeline(base=None)` | 已有 `EvaluationMonitor`；base 默认直接投影 | 将轨迹状态和反馈投影到 Actor 上下文 |

压缩统计开头系统消息之后的已提交消息，不是 token 阈值，也不压缩待提交消息。
开头系统消息、待提交工具协议和临时指令均保留。每次满足条件的 build 都重新生成派生摘要，
没有内置摘要缓存或自动摘要模型。摘要生成成本不会自动加入 Actor usage。

## 提供 compactor

实现异步 `compact(request, cancellation=...) -> ContextCompactionOutput`，
输出必须包含非空 `content` 和 `compactor_id`。应用选择摘要函数、模型、预算和信息保留策略。
下面的适配器接收由应用提供的异步摘要函数：

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

预期摘要失败抛出 `ContextCompactorError`，pipeline 将其映射为 `COMPACTION_FAILED`。
未声明异常或错误返回类型属于协议错误。失败不会静默降级为丢弃历史。

## 为验收任务组合上下文

下面的工厂需要已有监控器、Skills 目录和 compactor：

```python
from ejagent.context import DerivedCompactionPipeline, SkillsContextPipeline


def evaluated_context(monitor, skills_root, compactor):
    base = SkillsContextPipeline(
        skills_root,
        base=DerivedCompactionPipeline(compactor, minimum_messages=20),
    )
    return monitor.context_pipeline(base=base)
```

将返回值传为 Harness `context`，并将同一个监控器传为 `trajectory`。
包装层保留 base 上下文与临时指令，并向托管子组件传递生命周期；正常由 Harness 启动最外层。

替换投影实现时，提供 `ContextPipeline.build(request, cancellation=...)`，
返回 view 需保持 `run_id`、`source_revision`、`turn`。
上下文 metadata 是可检查数据，不会自动成为模型消息。Planner 和 Judge 独立构造各自上下文，
这里的 pipeline 配置 Actor 请求。

注入规则与清理见 [Skills](../skills/README_zh-CN.md)、[评估](../evaluation/README_zh-CN.md)
和[轨迹投影](../_trajectory/README_zh-CN.md)。
