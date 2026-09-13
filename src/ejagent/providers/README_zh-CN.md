# 模型 Provider 与请求配置

[English](README.md) · [模块索引](../README_zh-CN.md)

可使用 `OpenAIModelPort` 或 `AnthropicModelPort` 满足 `ModelPort`。
应用提供凭据并为 Actor、Planner、Judge 选择模型，三个角色可使用不同实例。
正常装配时由 Harness 管理适配器生命周期。

## OpenAI 兼容配置

`ModelConfig.from_env()` 会加载 `.env`；直接构造 `ModelConfig(...)` 使用传入值，
不会调用该环境加载方法。

| 字段 | 默认值 | 环境变量 |
| --- | --- | --- |
| `model` | 必填 | `CHAT_MODEL` |
| `api_key` | 必填 | `MODEL_API_KEY` |
| `base_url` | 必填 | `MODEL_URL` |
| `timeout` | `60` 秒 | `LLM_TIMEOUT` |
| `temperature` | `0.7` | `LLM_TEMPERATURE` |
| `include_usage` | `True` | `LLM_INCLUDE_USAGE`（`true`/`false`） |

```python
from ejagent.providers import ModelConfig, OpenAIModelPort


def configured_model():
    return OpenAIModelPort(ModelConfig.from_env())
```

`include_usage=False` 会省略流式 usage 请求，适用于拒绝此参数的兼容端点。
若因此缺失 usage，有限 Actor token 预算不能继续放行请求，Planner 准备失败，
Judge 返回 unknown 并阻止该 Run 的后续请求。关闭上报参数不会关闭成本保护。

## Anthropic 配置

安装 `ejagent-core[anthropic]`。`AnthropicConfig.from_env()` 使用：

| 字段 | 默认值 | 环境变量 |
| --- | --- | --- |
| `model`、`api_key` | 必填 | `ANTHROPIC_MODEL`、`ANTHROPIC_API_KEY` |
| `base_url` | `None`（SDK 默认） | `ANTHROPIC_BASE_URL` |
| `max_tokens` | `4096` | `ANTHROPIC_MAX_TOKENS` |
| `timeout` | `60.0` 秒 | `ANTHROPIC_TIMEOUT` |
| `temperature` | `0.7`（有效范围 0–1） | `ANTHROPIC_TEMPERATURE` |

提供上述环境变量后构造 `AnthropicModelPort(AnthropicConfig.from_env())`。
两个适配器均支持可选关键字参数 `client=None`，用于注入 SDK 客户端；注入后仍应遵循适配器生命周期，
但由应用负责关闭注入的客户端。内部创建的 SDK 客户端关闭 SDK 重试（`max_retries=0`）；
注入客户端的配置由其所有者负责。

## 单次请求与兼容性

| `ModelRequest` 字段 | 默认值 | OpenAI 兼容 | 原生 Anthropic |
| --- | --- | --- | --- |
| `tools` | `()` | 函数定义 | 转换后的工具定义 |
| `max_output_tokens` | `None` | 非空时映射为 `max_completion_tokens` | 与配置 `max_tokens` 取较小值；未传时用配置值 |
| `response_format` | `None` | 非空时原样透传 | 拒绝非 None 值 |

Planner/Judge 默认 `response_format={"type":"json_object"}`。接原生 Anthropic 时两者均需
设为 `response_format=None`，本地严格 JSON/Pydantic 校验及纠错仍然生效。
兼容 Provider 可透传自定义 JSON-schema 选项，应用负责选择支持它的后端。
Actor 的普通 Kernel 请求不会自动使用 Planner/Judge 的 JSON 配置。
Provider 输出上限与 Run 总预算作用范围不同。

## 自定义后端协议

将 `stream(request, cancellation=...)` 实现为异步迭代器，产出
`ModelTextDelta`/`ModelThinkingDelta`，最后恰好一个 `ModelResponseCompleted`，
携带标准化助手消息与 usage。完成后不得继续产出事件。标准化工具参数和结果；
预期后端失败使用 `ModelCallError`，非法事件流属于协议错误。当前消息协议支持文本，不支持图片输入。

另见[协议](../contracts/README_zh-CN.md)和[结构化输出](../../../docs/structured-output.md)。
凭据应保存在本地环境配置中，不写入受版本控制的文档或会话产物。
