# Model providers and request configuration

[中文](README_zh-CN.md) · [Module index](../README.md)

Use `OpenAIModelPort` or `AnthropicModelPort` to satisfy `ModelPort`. Applications
provide credentials and choose each Actor/Planner/Judge model; those roles can
use different instances. Harness manages adapter lifecycle when assembled normally.

## OpenAI-compatible configuration

`ModelConfig.from_env()` loads `.env`; explicit `ModelConfig(...)` construction
uses the supplied values without calling this loader.

| Field | Default | Environment variable |
| --- | --- | --- |
| `model` | Required | `CHAT_MODEL` |
| `api_key` | Required | `MODEL_API_KEY` |
| `base_url` | Required | `MODEL_URL` |
| `timeout` | `60` seconds | `LLM_TIMEOUT` |
| `temperature` | `0.7` | `LLM_TEMPERATURE` |
| `include_usage` | `True` | `LLM_INCLUDE_USAGE` (`true`/`false`) |

```python
from ejagent.providers import ModelConfig, OpenAIModelPort


def configured_model():
    return OpenAIModelPort(ModelConfig.from_env())
```

`include_usage=False` omits the streaming usage request for compatible endpoints
that reject it. If usage is then absent, finite Actor token budgets cannot admit
further requests, Planner preparation fails, and Judge returns unknown and blocks
further requests in that Run. It does not disable cost safeguards.

## Anthropic configuration

Install `ejagent-core[anthropic]`. `AnthropicConfig.from_env()` uses:

| Field | Default | Environment variable |
| --- | --- | --- |
| `model`, `api_key` | Required | `ANTHROPIC_MODEL`, `ANTHROPIC_API_KEY` |
| `base_url` | `None` (SDK default) | `ANTHROPIC_BASE_URL` |
| `max_tokens` | `4096` | `ANTHROPIC_MAX_TOKENS` |
| `timeout` | `60.0` seconds | `ANTHROPIC_TIMEOUT` |
| `temperature` | `0.7` (valid range 0–1) | `ANTHROPIC_TEMPERATURE` |

Construct `AnthropicModelPort(AnthropicConfig.from_env())` after supplying these
variables. Both adapters accept an optional keyword `client=None` for SDK client
injection. Use the adapter lifecycle even when injecting a client; the application
remains responsible for closing injected clients. Internally created SDK clients
disable SDK retries (`max_retries=0`); an injected client's settings are its owner's responsibility.

## Per-request options and compatibility

| `ModelRequest` field | Default | OpenAI-compatible | Native Anthropic |
| --- | --- | --- | --- |
| `tools` | `()` | Function definitions | Translated tool definitions |
| `max_output_tokens` | `None` | Sent as `max_completion_tokens` when set | Minimum of this value and config `max_tokens`; config value when absent |
| `response_format` | `None` | Forwarded unchanged when set | Non-None value rejected |

Planner/Judge default `response_format` to `{"type":"json_object"}`. Configure both
with `response_format=None` for native Anthropic; local strict JSON/Pydantic
validation and correction still apply. Custom JSON-schema options are passed
through for compatible providers; backend support is the application's choice.
The Actor's ordinary Kernel request does not automatically use Planner/Judge JSON
settings. Provider output caps and aggregate Run budgets have different scopes.

## Custom backend contract

Implement `stream(request, cancellation=...)` as an async iterator producing
`ModelTextDelta`/`ModelThinkingDelta` followed by exactly one
`ModelResponseCompleted` with the normalized assistant message and usage.
Do not emit events after completion. Normalize tool arguments/results and report
expected backend failures as `ModelCallError`; malformed streams are protocol errors.
Current message contracts support text, not image inputs.

See [contracts](../contracts/README.md) and
[structured output](../../../docs/structured-output.md). Keep credentials in local
environment configuration, outside tracked documentation and session artifacts.
