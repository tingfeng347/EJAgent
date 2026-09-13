# 工具注册与执行控制

[English](README.md) · [模块索引](../README_zh-CN.md)

应用定义 Actor 可调用的工具。EJAgent 提供调度器与 MCP 适配器，不会自动授予文件、shell 或网页工具。

## 函数工具与参数校验

`FunctionToolExecutor(tools=())` 接收名称唯一的 `FunctionTool(definition, function)`。
`ToolDefinition` 必填 `name`（1–64 个字母、数字、`_`、`-`），
默认 `description=None`、`input_schema={}`。
`ToolFunction` 是异步函数：`(ToolCall, CancellationToken) -> ToolExecutionResult`。

schema 会发给模型。执行器校验路由和返回类型，但不会自动用 JSON Schema 或 Pydantic
校验参数值。应在工具执行具体操作前自行校验参数：

```python
from ejagent.contracts import (
    CancellationToken,
    ToolCall,
    ToolDefinition,
    ToolExecutionResult,
)
from ejagent.tools import FunctionTool, FunctionToolExecutor


async def echo(call: ToolCall, cancellation: CancellationToken) -> ToolExecutionResult:
    cancellation.raise_if_cancelled()
    if set(call.arguments) != {"text"} or not isinstance(call.arguments["text"], str):
        return ToolExecutionResult(None, error="Expected exactly one string: text")
    return ToolExecutionResult({"text": call.arguments["text"]})


tools = FunctionToolExecutor(
    (
        FunctionTool(
            ToolDefinition(
                "echo",
                "Return the supplied text",
                {
                    "type": "object",
                    "properties": {"text": {"type": "string"}},
                    "required": ["text"],
                    "additionalProperties": False,
                },
            ),
            echo,
        ),
    )
)
```

## 返回值与 Run 控制

`ToolExecutionResult` 必填 JSON 兼容的 `result`；默认 `control=CONTINUE`、
`output=None`、`error=None`。非空 `error` 将结果标记为 Actor 可见的工具错误。
`output` 可用于提供终止输出。

| `ToolControl` | 作用 |
| --- | --- |
| `CONTINUE` | 将结果返回模型，继续执行 |
| `COMPLETE` | 提出成功完成请求；仍受强制验收约束 |
| `REJECT` | 以拒绝状态结束，不提交会话 |
| `CANCEL` | 以取消状态结束，不提交会话 |

预期基础设施失败可抛出 `ToolExecutionError`，这会使 Run 失败，而非返回可供 Actor 纠正的普通
工具错误。错误返回类型会触发 `ToolProtocolError`。适配器必须配合取消。

## 组合执行器与 MCP

`CompositeToolExecutor(executors)` 要求非空 iterable，合并定义、拒绝重复名称，并管理子资源。
自定义 `ToolExecutor` 需提供 `definitions` 和异步 `execute(call, cancellation=...)`。

`McpToolExecutor(config_path=None, manager=None)` 必须且只能提供其中一项。
使用配置文件时安装 `ejagent-core[mcp]`，并提供 `mcpServers` 映射：

```json
{
  "mcpServers": {
    "docs": {"url": "http://localhost:8000/mcp"}
  }
}
```

另行启动配置中的服务。适配器启动时发现工具，以 `service__tool` 命名，关闭时清空定义。
内置 manager 会记录单个服务连接失败，并继续使用其他可用服务；启动后需检查预期工具是否存在。
也可注入自定义 `McpManager`，实现异步 `startup/shutdown/call_tool` 和同步 `get_openai_tools`。

正常接入由 Harness 管理启动和关闭。启用 Planner 的 Harness 保留 `update_plan` 名称，见
[规划](../planning/README_zh-CN.md)。Planner 和 Judge 本身不会执行这些 Actor 工具。
Skills 注入指令，不会注册工具。
