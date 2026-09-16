# 自定义模型

通过 `~/.pi/agent/models.json` 添加自定义提供商和模型（Ollama、vLLM、LM Studio、代理）。

## 目录

- [最小示例](#最小示例)
- [完整示例](#完整示例)
- [支持的 API](#支持的-api)
- [提供商配置](#提供商配置)
- [模型配置](#模型配置)
- [覆盖内置提供商](#覆盖内置提供商)
- [按模型覆盖](#按模型覆盖)
- [Anthropic Messages 兼容性](#anthropic-messages-兼容性)
- [OpenAI 兼容性](#openai-兼容性)

## 最小示例

对于本地模型（Ollama、LM Studio、vLLM），每个模型只需指定 `id`：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        { "id": "llama3.1:8b" },
        { "id": "qwen2.5-coder:7b" }
      ]
    }
  }
}
```

`apiKey` 的值是占位符，因为 Ollama 会忽略它。pi 仍会将模型视为需要身份验证，只有通过验证后模型才会出现在 `/model` 中。因此，无密钥的本地服务器应保留一个虚拟值、使用 `/login` 为该提供商保存密钥，或在选择模型时传入 `--api-key`。

部分兼容 OpenAI 的服务器无法识别具备推理能力的模型所使用的 `developer` 角色。对于这些提供商，请将 `compat.supportsDeveloperRole` 设为 `false`，这样 pi 会改用 `system` 消息发送系统提示词。如果服务器也不支持 `reasoning_effort`，还应将 `compat.supportsReasoningEffort` 设为 `false`。

可以在提供商级别设置 `compat`，使其应用于所有模型；也可以在模型级别设置，以覆盖特定模型的配置。这通常适用于 Ollama、vLLM、SGLang 以及类似的 OpenAI 兼容服务器。

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "compat": {
        "supportsDeveloperRole": false,
        "supportsReasoningEffort": false
      },
      "models": [
        {
          "id": "gpt-oss:20b",
          "reasoning": true
        }
      ]
    }
  }
}
```

## 完整示例

需要特定值时，可覆盖默认值：

```json
{
  "providers": {
    "ollama": {
      "baseUrl": "http://localhost:11434/v1",
      "api": "openai-completions",
      "apiKey": "ollama",
      "models": [
        {
          "id": "llama3.1:8b",
          "name": "Llama 3.1 8B (Local)",
          "reasoning": false,
          "input": ["text"],
          "contextWindow": 128000,
          "maxTokens": 32000,
          "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 }
        }
      ]
    }
  }
}
```

每次打开 `/model` 时都会重新加载该文件。可以在会话期间编辑，无需重启。

## Google AI Studio 示例

使用带有 `baseUrl` 的 `google-generative-ai`，可添加 Google AI Studio 中的模型，包括自定义 Gemma 4 条目：

```json
{
  "providers": {
    "my-google": {
      "baseUrl": "https://generativelanguage.googleapis.com/v1beta",
      "api": "google-generative-ai",
      "apiKey": "$GEMINI_API_KEY",
      "models": [
        {
          "id": "gemma-4-31b-it",
          "name": "Gemma 4 31B",
          "input": ["text", "image"],
          "contextWindow": 262144,
          "reasoning": true
        }
      ]
    }
  }
}
```

向 `google-generative-ai` API 类型添加自定义模型时，必须指定 `baseUrl`。

## 支持的 API

| API | 说明 |
|-----|-------------|
| `openai-completions` | OpenAI Chat Completions（兼容性最广） |
| `openai-responses` | OpenAI Responses API |
| `anthropic-messages` | Anthropic Messages API |
| `google-generative-ai` | Google Generative AI |

在提供商级别设置 `api`（作为所有模型的默认值），或在模型级别设置（按模型覆盖）。

## 提供商配置

| 字段 | 说明 |
|-------|-------------|
| `baseUrl` | API 端点 URL |
| `api` | API 类型（见上文） |
| `apiKey` | 可选的 API 密钥配置（参见下文的值解析）。通过 `/login`/`auth.json` 或 CLI `--api-key` 提供身份验证时可省略。 |
| `oauth` | 动态 OAuth 提供商类型。目前支持 `"radius"`；需要网关的 `baseUrl`。 |
| `headers` | 自定义请求头（参见下文的值解析） |
| `authHeader` | 设为 `true` 可自动添加 `Authorization: Bearer <apiKey>` |
| `models` | 模型配置数组 |
| `modelOverrides` | 对此提供商的内置模型或扩展注册模型进行按模型覆盖 |

对于包含 `models` 的提供商，非内置提供商配置需要提供 `baseUrl`，并在提供商或模型级别设置 `api` 值。加载文件并不要求 `apiKey`：通过 `/login`/`auth.json`、CLI `--api-key` 或提供商的 `apiKey` 配置身份验证后，模型即可使用。如果未配置身份验证，模型仍会加载，但在 `/model` 和 `--list-models` 中保持不可用状态。

### 值解析

`apiKey` 和 `headers` 字段支持执行命令、环境变量插值和字面量：

- **Shell 命令：** 值以 `"!command"` 开头时，会将整个值作为命令执行并使用其 stdout
  ```json
  "apiKey": "!security find-generic-password -ws 'anthropic'"
  "apiKey": "!op read 'op://vault/item/credential'"
  ```
- **环境变量插值：** `"$ENV_VAR"` 或 `"${ENV_VAR}"` 使用指定变量的值。插值也适用于较长的字面量。
  ```json
  "apiKey": "$MY_API_KEY"
  "apiKey": "${KEY_PREFIX}_${KEY_SUFFIX}"
  ```
  `$FOO_BAR` 表示变量 `FOO_BAR`；当 `BAR` 是字面文本时，请使用 `${FOO}_BAR`。缺失的环境变量会使该值无法解析。
- **转义：** `"$$"` 会生成字面量 `"$"`；`"$!"` 会生成字面量 `"!"`，且不会触发命令执行。
  ```json
  "apiKey": "$$literal-dollar-prefix"
  "apiKey": "$!literal-bang-prefix"
  ```
- **字面量值：** 直接使用。`MY_API_KEY` 等纯大写字符串属于字面量；环境变量应使用 `$MY_API_KEY`。
  ```json
  "apiKey": "sk-..."
  ```

对于 `models.json`，Shell 命令会在请求时解析。pi 有意不对任意命令应用内置 TTL、陈旧值复用或恢复逻辑。不同命令需要不同的缓存和失败处理策略，而 pi 无法推断正确的策略。

如果命令速度慢、开销大、受速率限制，或者在暂时失败时应继续使用之前的值，请用自己的脚本或命令进行包装，并在其中实现所需的缓存或 TTL 行为。

`/model` 可用性检查只判断是否存在已配置的身份验证，不会执行 Shell 命令。

### 自定义请求头

```json
{
  "providers": {
    "custom-proxy": {
      "baseUrl": "https://proxy.example.com/v1",
      "apiKey": "$MY_API_KEY",
      "api": "anthropic-messages",
      "headers": {
        "x-portkey-api-key": "$PORTKEY_API_KEY",
        "x-secret": "!op read 'op://vault/item/secret'"
      },
      "models": [...]
    }
  }
}
```

## 模型配置

| 字段 | 必填 | 默认值 | 说明 |
|-------|----------|---------|-------------|
| `id` | 是 | — | 模型标识符（传递给 API） |
| `name` | 否 | `id` | 人类可读的模型标签。用于匹配（`--model` 模式），并显示为模型的次要详情文本。 |
| `api` | 否 | 提供商的 `api` | 为此模型覆盖提供商的 API |
| `reasoning` | 否 | `false` | 是否支持扩展思考 |
| `thinkingLevelMap` | 否 | 省略 | 将 pi 的思考级别映射到提供商的值，并标记不支持的级别（见下文） |
| `input` | 否 | `["text"]` | 输入类型：`["text"]` 或 `["text", "image"]` |
| `contextWindow` | 否 | `128000` | 以 token 为单位的上下文窗口大小 |
| `maxTokens` | 否 | `16384` | 最大输出 token 数 |
| `samplingParams` | 否 | 省略 | 原样合并到每个请求体中的采样参数（见下文） |
| `cost` | 否 | 全部为零 | 每百万 token 的费率，可选按请求整体输入量分级定价 |
| `compat` | 否 | 提供商的 `compat` | 提供商兼容性覆盖。同时设置两者时，与提供商级别的 `compat` 合并。 |

一个费用层级会提供一整套替代费率。当输入总用量（`input + cacheRead + cacheWrite`）超过 `inputTokensAbove` 时，该费率应用于整个请求。若多个层级均匹配，则以阈值最高者为准。

```json
{
  "cost": {
    "input": 5,
    "output": 30,
    "cacheRead": 0.5,
    "cacheWrite": 6.25,
    "tiers": [
      {
        "inputTokensAbove": 272000,
        "input": 10,
        "output": 45,
        "cacheRead": 1,
        "cacheWrite": 12.5
      }
    ]
  }
}
```

当前行为：
- `/model`、`--list-models` 和交互式页脚按模型 `id` 显示条目。
- 已配置的 `name` 用于模型匹配和次要模型详情文本。它不会取代页脚/状态栏中的模型 ID。

### 采样参数

`samplingParams` 是一个自由格式对象。在 pi 自身设置的字段之后，它会原样合并到该模型的每个请求体中，因此其中的键优先。可用它发送 pi 未建模的采样参数，包括 llama.cpp 的 `min_p` 或 vLLM 的 `top_k` 等服务器特有参数：

```json
{
  "id": "deepseek-v4-flash",
  "samplingParams": {
    "temperature": 1.0,
    "top_p": 0.95,
    "top_k": 0,
    "min_p": 0.0
  }
}
```

只有 OpenAI 兼容 API 会应用它（`openai-completions`、`openai-responses`、`azure-openai-responses`）；其他 API 会忽略它。其中的键会覆盖 pi 的命名请求字段（例如，这里的 `temperature` 键优先于请求级 temperature），因此最好将它作为模型采样配置的唯一事实来源。在 `modelOverrides` 中，`samplingParams` 会逐键与基础模型的值合并。

也可以在此设置固定的思考 token 上限，但它不会遵循 `thinkingBudgets`，也不会为答案预留空间。更推荐使用 `compat.thinkingTokenBudgetField`（或 `supportsThinkingTokenBudget` 别名）。

### 思考级别映射

在模型上使用 `thinkingLevelMap` 描述该模型专用的思考控制。键是 pi 的思考级别：`off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max`。映射中可以存在空缺；例如，模型可以公开 `high` 和 `max`，但不公开 `xhigh`。

值有三种状态：

| 值 | 含义 |
|-------|---------|
| 省略 | 从标准级别到 `high` 均使用提供商的默认映射；扩展的 `xhigh` 和 `max` 级别不受支持 |
| string | 支持该级别，并将此值发送给提供商 |
| `null` | 不支持该级别，并将其隐藏、跳过或钳制到其他级别 |

仅支持关闭、高和最高推理级别的模型示例：

```json
{
  "id": "deepseek-v4-pro",
  "reasoning": true,
  "thinkingLevelMap": {
    "minimal": null,
    "low": null,
    "medium": null,
    "high": "high",
    "xhigh": null,
    "max": "max"
  }
}
```

无法禁用思考的模型示例：

```json
{
  "id": "always-thinking-model",
  "reasoning": true,
  "thinkingLevelMap": {
    "off": null
  }
}
```

迁移：使用旧版 `compat.reasoningEffortMap` 的配置应将该映射移至模型级别的 `thinkingLevelMap`。对于不应出现在 UI 中的级别，请使用 `null`。

## 覆盖内置提供商

通过代理路由内置提供商，无需重新定义模型：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1"
    }
  }
}
```

所有内置 Anthropic 模型仍然可用。现有 OAuth 或 API 密钥身份验证可继续使用。

要将自定义模型合并到内置提供商中，请包含 `models` 数组：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://my-proxy.example.com/v1",
      "apiKey": "$ANTHROPIC_API_KEY",
      "api": "anthropic-messages",
      "models": [...]
    }
  }
}
```

合并语义：
- 保留内置模型。
- 在提供商内部按 `id` 对自定义模型执行 upsert。
- 如果自定义模型的 `id` 与内置模型的 `id` 匹配，自定义模型会替换该内置模型。
- 如果自定义模型的 `id` 是新的，则会将其添加到内置模型旁。

## 按模型覆盖

使用 `modelOverrides` 可自定义内置模型和匹配的扩展注册模型，而无需替换提供商的完整模型列表。

```json
{
  "providers": {
    "openrouter": {
      "modelOverrides": {
        "anthropic/claude-sonnet-4": {
          "name": "Claude Sonnet 4 (Bedrock Route)",
          "compat": {
            "openRouterRouting": {
              "only": ["amazon-bedrock"]
            }
          }
        }
      }
    }
  }
}
```

`modelOverrides` 支持为每个模型设置以下字段：`name`、`reasoning`、`thinkingLevelMap`、`input`、`cost`（部分字段）、`contextWindow`、`maxTokens`、`samplingParams`（逐键合并）、`headers`、`compat`。

直接使用 OpenAI GPT-5.6 Sol、Terra 和 Luna 时，默认上下文窗口为 `272000`，以确保请求处于 OpenAI 的短上下文定价层级内。要启用 OpenAI 的 1.05M 上下文窗口，请为使用的每个模型增大该值：

```json
{
  "providers": {
    "openai": {
      "modelOverrides": {
        "gpt-5.6-sol": {
          "contextWindow": 1050000
        }
      }
    }
  }
}
```

该覆盖会保留内置定价元数据。总输入 token 超过 272K 的请求会对整个请求使用 GPT-5.6 的长上下文费率。需要时，对 `gpt-5.6-terra` 或 `gpt-5.6-luna` 应用相同的覆盖。

行为说明：
- `modelOverrides` 应用于内置提供商模型和匹配的扩展注册提供商模型。
- 未知模型 ID 会被忽略。
- 可以将提供商级别的 `baseUrl`/`headers` 与 `modelOverrides` 结合使用。
- 覆盖 `name` 只会改变模型匹配和次要详情文本；页脚和主要模型列表仍会显示模型 `id`。
- 如果还为提供商定义了 `models`，则会在内置覆盖之后合并自定义模型。具有相同 `id` 的自定义模型会替换覆盖后的内置模型条目。

## Anthropic Messages 兼容性

对于使用 `api: "anthropic-messages"` 的提供商或代理，请使用 `compat` 控制 Anthropic 特有的请求兼容性。

默认情况下，pi 会为每个工具发送 `eager_input_streaming: true`。如果代理或 Anthropic 兼容后端拒绝该字段，请将 `supportsEagerToolInputStreaming` 设为 `false`。此时 Pi 会省略 `tools[].eager_input_streaming`，并改为在启用工具的请求中发送旧版 `fine-grained-tool-streaming-2025-05-14` beta 请求头。

部分 Anthropic 模型要求使用自适应思考（`thinking.type: "adaptive"` 加 `output_config.effort`），而不是旧版基于预算的思考载荷。内置模型会自动设置此项。对于路由到这些模型的自定义提供商或别名，请将 `forceAdaptiveThinking` 设为 `true`。

支持按轮次设置 effort 的 Claude 模型使用 `supportsMidConvoEffort`。随后，Pi 会持久保存每个响应的提供商 effort，在后续请求中重建仅含 effort 的系统消息，并发送带有 `prefix_mismatch_behavior: "drop_block"` 的思考绑定控制，以免陈旧的已签名思考前缀持续引发 400 响应。只应为通过忠实 Anthropic Messages 传输层访问的确切受支持 Claude 模型设置此项；不要为仅模仿 Messages 格式的 API 启用它。

部分 Anthropic 兼容提供商会生成签名为空的思考块，并仍要求在重放时包含这些块。仅对这些提供商将 `allowEmptySignature` 设为 `true`；真正的 Anthropic 会拒绝空思考签名。

内置 Anthropic 模型在其模型元数据中启用了 `supportsStrictTools`。当自定义 Anthropic 兼容模型的端点接受严格 JSON-schema 工具定义时，必须将其设为 `true`。

```json
{
  "providers": {
    "anthropic-proxy": {
      "baseUrl": "https://proxy.example.com",
      "api": "anthropic-messages",
      "apiKey": "$ANTHROPIC_PROXY_KEY",
      "compat": {
        "supportsEagerToolInputStreaming": false,
        "supportsLongCacheRetention": true,
        "forceAdaptiveThinking": true,
        "allowEmptySignature": true
      },
      "models": [
        {
          "id": "claude-opus-4-7",
          "reasoning": true,
          "input": ["text", "image"]
        }
      ]
    }
  }
}
```

| 字段 | 说明 |
|-------|-------------|
| `supportsEagerToolInputStreaming` | 提供商是否接受按工具设置的 `eager_input_streaming`。默认值：`true`。设为 `false` 可省略该字段，并在启用工具的请求中使用旧版细粒度工具流式传输 beta 请求头。 |
| `supportsLongCacheRetention` | 当缓存保留策略为 `long` 时，提供商是否接受 Anthropic 长期缓存保留（`cache_control.ttl: "1h"`）。默认值：`true`。 |
| `sendSessionAffinityHeaders` | 启用缓存时，是否根据 session ID 发送 `x-session-affinity`。默认值：对已知提供商自动检测。 |
| `supportsCacheControlOnTools` | 提供商是否接受工具定义中的 Anthropic 风格 `cache_control` 标记。默认值：`true`。 |
| `forceAdaptiveThinking` | 是否为此模型发送自适应思考（`thinking.type: "adaptive"` 加 `output_config.effort`）。内置自适应模型会自动设置此项。默认值：`false`。 |
| `supportsMidConvoEffort` | 该确切 Claude 模型传输层是否支持按轮次的 effort 系统消息和思考绑定控制。启用后，Pi 会持久保存原生 effort 级别，并始终发送 `drop_block`。默认值：`false`。 |
| `allowEmptySignature` | 是否以 `signature: ""` 的形式重放空思考签名，而不是将思考转换为文本。默认值：`false`。 |
| `supportsStrictTools` | 提供商是否接受严格的 JSON-schema 工具定义。默认值：`false`；内置 Anthropic 模型已在生成的元数据中启用它。 |

## OpenAI 兼容性

对于仅部分兼容 OpenAI 的提供商，请使用 `compat` 字段。

- 提供商级别的 `compat` 会将默认值应用于该提供商下的所有模型。
- 模型级别的 `compat` 会为该模型覆盖提供商级别的值。

```json
{
  "providers": {
    "local-llm": {
      "baseUrl": "http://localhost:8080/v1",
      "api": "openai-completions",
      "compat": {
        "supportsUsageInStreaming": false,
        "maxTokensField": "max_tokens"
      },
      "models": [...]
    }
  }
}
```

| 字段 | 说明 |
|-------|-------------|
| `supportsStore` | 提供商是否支持 `store` 字段 |
| `supportsDeveloperRole` | 使用 `developer` 还是 `system` 角色 |
| `supportsReasoningEffort` | 是否支持 `reasoning_effort` 参数 |
| `supportsUsageInStreaming` | 是否支持 `stream_options: { include_usage: true }`（默认值：`true`） |
| `supportsFinishReason` | 流式响应是否包含 `finish_reason`。为 `false` 时，pi 会在流结束时推断 `stop` 或 `toolUse`。默认值：`true`。 |
| `maxTokensField` | 使用 `max_completion_tokens` 还是 `max_tokens` |
| `requiresToolResultName` | 在工具结果消息中包含 `name` |
| `requiresAssistantAfterToolResult` | 在工具结果后的用户消息之前插入一条助手消息 |
| `requiresThinkingAsText` | 将思考块转换为纯文本 |
| `requiresReasoningContentOnAssistantMessages` | 启用推理时，在所有重放的助手消息中包含空的 `reasoning_content` |
| `thinkingFormat` | 使用 `reasoning_effort`、`openrouter`、`deepseek`、`together`、`baseten`、`zai`、`qwen`、`chat-template` 或 `qwen-chat-template` 思考参数 |
| `chatTemplateKwargs` | `thinkingFormat: "chat-template"` 的 `chat_template_kwargs` 值；使用 `{ "$var": "thinking.enabled" }`、`{ "$var": "thinking.effort" }` 或 `{ "$var": "thinking.budget" }` 表示由 pi 控制的思考值 |
| `chatTemplateArgs` | `thinkingFormat: "baseten"` 的 `chat_template_args` 值；使用 `{ "$var": "thinking.enabled" }`、`{ "$var": "thinking.effort" }` 或 `{ "$var": "thinking.budget" }` 表示由 pi 控制的思考值 |
| `thinkingTokenBudgetField` | 用于根据 `thinkingBudgets` 限制推理 token 的顶层请求字段，该值会被钳制，以便为答案至少保留 1024 个 token。`"thinking_token_budget"`（vLLM）、`"thinking_budget"`（Qwen/DashScope/SGLang）、`"thinking_budget_tokens"`（llama.cpp）。默认关闭；生成的目录中不会设置。 |
| `supportsThinkingTokenBudget` | `thinkingTokenBudgetField: "thinking_token_budget"`（vLLM）的别名。推荐使用 `thinkingTokenBudgetField`。默认值：`false`。 |
| `cacheControlFormat` | 在系统提示词、最后一个工具定义，以及最后一条用户、助手或工具结果文本内容上使用 Anthropic 风格的 `cache_control` 标记。目前仅支持 `anthropic`。 |
| `sendSessionAffinityHeaders` | 对于 `openai-completions`，启用缓存时根据 session ID 发送会话亲和性请求头。默认值：`false`。 |
| `sessionAffinityFormat` | 对于 `openai-completions` 和 `openai-responses`，会话亲和性请求头格式：`openai` 发送 `session_id`/`x-client-request-id`（completions 还发送 `x-session-affinity`），`openai-nosession` 省略包含下划线的 `session_id` 请求头，`openrouter` 发送 `x-session-id`。不影响请求体参数 `prompt_cache_key`。默认值：自动检测。 |
| `supportsStrictMode` | 提供商是否接受严格的 JSON-schema 函数工具定义。默认值取决于 API；内置 OpenAI 模型带有明确的能力元数据。 |
| `supportsOpenAIGrammarTools` | OpenAI 兼容 API 是否生成自定义 Lark/regex 语法工具。为 `false` 时，受语法约束的工具会回退为普通函数工具。默认值：`false`；内置模型目录为 OpenAI、OpenAI Codex、Azure OpenAI、GitHub Copilot、opencode 和 Cloudflare AI Gateway 上的 GPT-5+ 模型启用了它。 |
| `deferredToolsMode` | 使用提供商特有的延迟工具序列化。目前仅支持 `"kimi"`，用于 Kimi 的 OpenAI 兼容 Chat Completions 格式。 |
| `supportsLongCacheRetention` | 当缓存保留策略为 `long` 时，提供商是否接受长期缓存保留：GPT-5.6+ Responses 模型使用 `prompt_cache_options.ttl: "30m"`，较早的 OpenAI 模型使用 `prompt_cache_retention: "24h"`，当 `cacheControlFormat` 为 `anthropic` 时使用 `cache_control.ttl: "1h"`。默认值：`true`。 |
| `openRouterRouting` | OpenRouter 提供商路由偏好。该对象会原样作为 [OpenRouter API 请求](https://openrouter.ai/docs/guides/routing/provider-selection)中的 `provider` 字段发送。 |
| `vercelGatewayRouting` | 用于选择提供商的 Vercel AI Gateway 路由配置（`only`、`order`） |

`openrouter` 使用 `reasoning: { effort }`。`together` 使用 `reasoning: { enabled }`，并在启用 `supportsReasoningEffort` 时同时使用 `reasoning_effort`。`qwen` 使用顶层 `enable_thinking`。对于要求 `chat_template_kwargs.enable_thinking` 和 `preserve_thinking` 的本地 Qwen 兼容服务器，请使用 `qwen-chat-template`。对于需要可配置 `chat_template_kwargs` 的 vLLM/Hugging Face 聊天模板，请使用 `chat-template`；例如，DeepSeek V3.x 模板可使用 `chatTemplateKwargs: { "thinking": { "$var": "thinking.enabled" } }`。对于通过 `chat_template_args` 提供开关控制，并可选择支持顶层 `reasoning_effort` 的提供商，请使用 `thinkingFormat: "baseten"` 和 `chatTemplateArgs`。

`thinkingTokenBudgetField` 独立于 `thinkingFormat`。不要在生成的 Qwen 目录中启用它：这些模型已经发送 `reasoning_effort`，而 DashScope 会拒绝同时包含 `thinking_budget` 和 `reasoning_effort` 的请求。

`cacheControlFormat: "anthropic"` 用于通过文本内容和工具定义上的 `cache_control` 标记公开 Anthropic 风格提示词缓存的 OpenAI 兼容提供商。

示例：

```json
{
  "providers": {
    "openrouter": {
      "baseUrl": "https://openrouter.ai/api/v1",
      "apiKey": "$OPENROUTER_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "openrouter/anthropic/claude-3.5-sonnet",
          "name": "OpenRouter Claude 3.5 Sonnet",
          "compat": {
            "openRouterRouting": {
              "allow_fallbacks": true,
              "require_parameters": false,
              "data_collection": "deny",
              "zdr": true,
              "enforce_distillable_text": false,
              "order": ["anthropic", "amazon-bedrock", "google-vertex"],
              "only": ["anthropic", "amazon-bedrock"],
              "ignore": ["gmicloud", "friendli"],
              "quantizations": ["fp16", "bf16"],
              "sort": {
                "by": "price",
                "partition": "model"
              },
              "max_price": {
                "prompt": 10,
                "completion": 20
              },
              "preferred_min_throughput": {
                "p50": 100,
                "p90": 50
              },
              "preferred_max_latency": {
                "p50": 1,
                "p90": 3,
                "p99": 5
              }
            }
          }
        }
      ]
    }
  }
}
```

Vercel AI Gateway 示例：

```json
{
  "providers": {
    "vercel-ai-gateway": {
      "baseUrl": "https://ai-gateway.vercel.sh/v1",
      "apiKey": "$AI_GATEWAY_API_KEY",
      "api": "openai-completions",
      "models": [
        {
          "id": "moonshotai/kimi-k2.5",
          "name": "Kimi K2.5 (Fireworks via Vercel)",
          "reasoning": true,
          "input": ["text", "image"],
          "cost": { "input": 0.6, "output": 3, "cacheRead": 0, "cacheWrite": 0 },
          "contextWindow": 262144,
          "maxTokens": 262144,
          "compat": {
            "vercelGatewayRouting": {
              "only": ["fireworks", "novita"],
              "order": ["fireworks", "novita"]
            }
          }
        }
      ]
    }
  }
}
```
