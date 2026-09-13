# Tool registration and execution control

[中文](README_zh-CN.md) · [Module index](../README.md)

Applications define the tools available to the Actor. EJAgent supplies dispatchers
and an MCP adapter; it does not automatically grant filesystem, shell, or web tools.

## Function tools and parameter validation

`FunctionToolExecutor(tools=())` accepts `FunctionTool(definition, function)` values
with unique names. `ToolDefinition` requires `name` (1–64 letters, digits, `_`, `-`);
`description=None` and `input_schema={}` are defaults. `ToolFunction` is an async
callable `(ToolCall, CancellationToken) -> ToolExecutionResult`.

The schema is sent to the model. The executor validates routing and return types,
but does not automatically validate argument values against JSON Schema or a
Pydantic model. Validate arguments inside your tool before performing its operation:

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

## Return values and Run control

`ToolExecutionResult` requires a JSON-compatible `result`; `control=CONTINUE`,
`output=None`, and `error=None` are defaults. A nonempty `error` marks the result
as a tool error visible to the Actor. `output` supplies terminal output when used.

| `ToolControl` | Effect |
| --- | --- |
| `CONTINUE` | Return the result to the model and continue execution |
| `COMPLETE` | Propose successful completion; enforced acceptance still applies |
| `REJECT` | End as rejected; do not commit conversation |
| `CANCEL` | End as cancelled; do not commit conversation |

Raise `ToolExecutionError` for expected infrastructure failure; this fails the Run
instead of returning a normal tool error for Actor correction. Wrong return types
raise `ToolProtocolError`. Adapters must cooperate with cancellation.

## Compose executors and connect MCP

`CompositeToolExecutor(executors)` requires a nonempty iterable, combines definitions,
rejects duplicate names, and manages child resources. A custom `ToolExecutor` must
provide `definitions` and async `execute(call, cancellation=...)`.

`McpToolExecutor(config_path=None, manager=None)` requires exactly one argument.
For a config file, install `ejagent-core[mcp]` and provide an `mcpServers` mapping:

```json
{
  "mcpServers": {
    "docs": {"url": "http://localhost:8000/mcp"}
  }
}
```

Start the configured service separately. The adapter discovers tools at startup,
namespaces them as `service__tool`, and clears definitions at shutdown. The built-in
manager logs individual service connection failures and continues with available
services; verify expected tools after startup. Alternatively inject `McpManager`
with async `startup/shutdown/call_tool` and synchronous `get_openai_tools`.

Harness owns normal startup/shutdown. Planner-enabled Harness reserves `update_plan`;
see [planning](../planning/README.md). Planner and Judge themselves do not execute
these Actor tools. Skills inject instructions and do not register tools.
