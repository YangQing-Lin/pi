# 自定义提供商

扩展可以通过 `pi.registerProvider()` 注册自定义模型提供商。这支持：

- **代理** - 通过企业代理或 API gateway 路由请求
- **自定义端点** - 使用自托管或私有模型部署
- **OAuth/SSO** - 为企业提供商添加身份验证流程
- **自定义 API** - 为非标准 LLM API 实现流式传输

## 示例扩展

请参阅以下完整的提供商示例：

- [`examples/extensions/custom-provider-anthropic/`](../examples/extensions/custom-provider-anthropic/)
- [`examples/extensions/custom-provider-gitlab-duo/`](../examples/extensions/custom-provider-gitlab-duo/)

## 目录

- [示例扩展](#示例扩展)
- [快速参考](#快速参考)
- [覆盖现有提供商](#覆盖现有提供商)
- [注册新提供商](#注册新提供商)
- [注销提供商](#注销提供商)
- [OAuth 支持](#oauth-支持)
- [自定义流式 API](#自定义流式-api)
- [上下文溢出错误](#上下文溢出错误)
- [测试你的实现](#测试你的实现)
- [配置参考](#配置参考)
- [模型定义参考](#模型定义参考)

## 快速参考

扩展既可以注册完整的 pi-ai `Provider`，也可以使用旧版 provider-config 形式。当需要自定义身份验证、过滤、刷新或流式传输行为时，优先使用完整的提供商。Pi 会将 `models.json` 覆盖配置组合到已注册的原生提供商之上。

```typescript
import { createProvider, openAICompletionsApi } from "@earendil-works/pi-ai";
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.registerProvider(createProvider({
    id: "native-local",
    name: "Native Local",
    baseUrl: "http://localhost:8080/v1",
    auth: {
      apiKey: {
        name: "Local server API key",
        async login(interaction) {
          return {
            type: "api_key",
            key: await interaction.prompt({ type: "secret", message: "API key" })
          };
        },
        async resolve({ credential }) {
          return credential?.key
            ? { auth: { apiKey: credential.key }, source: "stored API key" }
            : undefined;
        }
      }
    },
    models: [],
    api: openAICompletionsApi()
  }));

  // 旧版 provider-config 形式：
  // 覆盖现有提供商的 baseUrl
  pi.registerProvider("anthropic", {
    baseUrl: "https://proxy.example.com"
  });

  // 注册带模型的新提供商
  pi.registerProvider("my-provider", {
    name: "My Provider",
    baseUrl: "https://api.example.com",
    apiKey: "$MY_API_KEY",
    api: "openai-completions",
    models: [
      {
        id: "my-model",
        name: "My Model",
        reasoning: false,
        input: ["text", "image"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 4096
      }
    ]
  });
}
```

扩展工厂也可以是 `async`。若要动态发现模型，请在工厂中获取并注册模型，而不是在 `session_start` 中进行。pi 会等待工厂完成后再继续启动，因此该提供商在交互式启动期间以及对 `pi --list-models` 都可用。

## 覆盖现有提供商

最简单的用例：通过代理重定向现有提供商。

```typescript
// 现在，所有 Anthropic 请求都会通过你的代理
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com"
});

// 向 OpenAI 请求添加自定义 header
pi.registerProvider("openai", {
  headers: {
    "X-Custom-Header": "value"
  }
});

// 同时设置 baseUrl 和 headers
pi.registerProvider("google", {
  baseUrl: "https://ai-gateway.corp.com/google",
  headers: {
    "X-Corp-Auth": "$CORP_AUTH_TOKEN"  // 环境变量或字面值
  }
});
```

当只提供 `baseUrl` 和/或 `headers`（没有 `models`）时，该提供商的所有现有模型都会保留，但改用新端点。

## 注册新提供商

要添加全新的提供商，请在必需配置之外指定 `models`。

如果模型列表来自远程端点，请使用异步扩展工厂：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default async function (pi: ExtensionAPI) {
  const response = await fetch("http://localhost:1234/v1/models");
  const payload = (await response.json()) as {
    data: Array<{
      id: string;
      name?: string;
      context_window?: number;
      max_tokens?: number;
    }>;
  };

  pi.registerProvider("local-openai", {
    baseUrl: "http://localhost:1234/v1",
    apiKey: "$LOCAL_OPENAI_API_KEY",
    api: "openai-completions",
    models: payload.data.map((model) => ({
      id: model.id,
      name: model.name ?? model.id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: model.context_window ?? 128000,
      maxTokens: model.max_tokens ?? 4096,
    })),
  });
}
```

这会在启动完成前注册获取到的模型。

```typescript
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com/v1",
  apiKey: "$MY_LLM_API_KEY",  // 环境变量引用
  api: "openai-completions",  // 要使用的流式 API
  models: [
    {
      id: "my-llm-large",
      name: "My LLM Large",
      reasoning: true,        // 支持扩展思考
      input: ["text", "image"],
      cost: {
        input: 3.0,           // 美元/百万 token
        output: 15.0,
        cacheRead: 0.3,
        cacheWrite: 3.75
      },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});
```

提供 `models` 时，它会**替换**该提供商的所有现有模型。

`apiKey` 和自定义 header 值使用与 `models.json` 相同的配置值语法：开头的 `!command` 会执行命令并将其结果作为整个值，`$ENV_VAR` 和 `${ENV_VAR}` 会插值环境变量，`$$` 会生成字面量 `$`，而 `$!` 会生成字面量 `!`。

## 注销提供商

使用 `pi.unregisterProvider(name)` 移除先前通过 `pi.registerProvider(name, ...)` 注册的提供商：

```typescript
// 注册
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com/v1",
  apiKey: "$MY_LLM_API_KEY",
  api: "openai-completions",
  models: [
    {
      id: "my-llm-large",
      name: "My LLM Large",
      reasoning: true,
      input: ["text", "image"],
      cost: { input: 3.0, output: 15.0, cacheRead: 0.3, cacheWrite: 3.75 },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});

// 稍后将其移除
pi.unregisterProvider("my-llm");
```

注销会移除该提供商的动态模型、API key 回退、OAuth 提供商注册以及自定义流处理程序注册。所有被覆盖的内置模型或提供商行为都会恢复。

在初始扩展加载阶段之后进行的调用会立即生效，因此不需要 `/reload`。

### API 类型

`api` 字段决定使用哪种流式实现：

| API | 用途 |
|-----|---------|
| `anthropic-messages` | Anthropic Claude API 及其兼容 API |
| `openai-completions` | OpenAI Chat Completions API 及其兼容 API |
| `openai-responses` | OpenAI Responses API |
| `azure-openai-responses` | Azure OpenAI Responses API |
| `openai-codex-responses` | OpenAI Codex Responses API |
| `mistral-conversations` | 原生 Mistral Chat Completions 流式传输 |
| `google-generative-ai` | Google Generative AI API |
| `google-vertex` | Google Vertex AI API |
| `bedrock-converse-stream` | Amazon Bedrock Converse API |

大多数 OpenAI 兼容提供商可使用 `openai-completions`。模型特定的思考级别使用模型级 `thinkingLevelMap`，提供商的特殊行为使用 `compat`。`xhigh` 和 `max` 级别为选择启用，要求对应的映射条目非 null，并且中间可以由不受支持的空缺级别分隔：

```typescript
models: [{
  id: "custom-model",
  // ...
  reasoning: true,
  thinkingLevelMap: {              // 将 pi 级别映射到提供商值；null 会隐藏不受支持的级别
    minimal: null,
    low: null,
    medium: null,
    high: "default",
    xhigh: null,
    max: "max"
  },
  compat: {
    supportsDeveloperRole: false,   // 使用 "system" 而不是 "developer"
    supportsReasoningEffort: true,
    maxTokensField: "max_tokens",   // 而不是 "max_completion_tokens"
    requiresToolResultName: true,   // 工具结果需要 name 字段
    thinkingFormat: "qwen",        // 顶层 enable_thinking: true
    cacheControlFormat: "anthropic" // Anthropic 风格的 cache_control 标记
  }
}]
```

对 OpenRouter 风格的 `reasoning: { effort }` 控制使用 `openrouter`。对 Together 风格的 `reasoning: { enabled }` 控制使用 `together`；搭配 `supportsReasoningEffort` 时，还会发送 `reasoning_effort`。对于读取 `chat_template_kwargs.enable_thinking` 且需要 `preserve_thinking` 的本地 Qwen 兼容服务器，请使用 `qwen-chat-template`。
对于通过 system prompt、最后一个工具定义以及最后一段用户、助手或工具结果文本内容上的 `cache_control` 来提供 Anthropic 风格 prompt 缓存的 OpenAI 兼容提供商，请使用 `cacheControlFormat: "anthropic"`。

对于使用 `api: "anthropic-messages"` 的 Anthropic 兼容提供商，如果其上游模型要求自适应思考（`thinking.type: "adaptive"` 加 `output_config.effort`），请在模型或提供商上设置 `compat.forceAdaptiveThinking: true`。内置的自适应 Claude 模型会自动设置此项。仅对会生成空思考签名并期望重放时使用 `signature: ""` 的提供商设置 `compat.allowEmptySignature: true`。

> 迁移说明：Mistral 已从 `openai-completions` 迁移到 `mistral-conversations`。
> 原生 Mistral 模型请使用 `mistral-conversations`。
> 如果你有意通过 `openai-completions` 路由 Mistral 兼容/自定义端点，请根据需要显式设置 `compat` 标志。

### 身份验证 Header

如果你的提供商需要 `Authorization: Bearer <key>` 但并未使用标准 API，请设置 `authHeader: true`：

```typescript
pi.registerProvider("custom-api", {
  baseUrl: "https://api.example.com",
  apiKey: "$MY_API_KEY",
  authHeader: true,  // 添加 Authorization: Bearer header
  api: "openai-completions",
  models: [...]
});
```

每次请求都会解析 key。显式的请求 `Authorization` header 优先于生成的值。

## OAuth 支持

添加与 `/login` 集成的 OAuth/SSO 身份验证：

```typescript
import type { OAuthCredentials, OAuthLoginCallbacks } from "@earendil-works/pi-ai";

pi.registerProvider("corporate-ai", {
  baseUrl: "https://ai.corp.com/v1",
  api: "openai-responses",
  models: [...],
  oauth: {
    name: "Corporate AI (SSO)",

    async login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials> {
      const method = await callbacks.onSelect({
        message: "Select login method:",
        options: [
          { id: "browser", label: "Browser OAuth" },
          { id: "device", label: "Device code" }
        ]
      });
      if (!method) throw new Error("Login cancelled");

      let code: string;
      if (method === "device") {
        callbacks.onDeviceCode({
          userCode: "ABCD-1234",
          verificationUri: "https://sso.corp.com/device",
          intervalSeconds: 5,
          expiresInSeconds: 900
        });
        code = await pollDeviceCodeUntilComplete();
      } else {
        callbacks.onAuth({ url: "https://sso.corp.com/authorize?..." });
        code = await callbacks.onPrompt({ message: "Enter SSO code:" });
      }

      // 交换 token（由你实现）
      const tokens = await exchangeCodeForTokens(code);

      return {
        refresh: tokens.refreshToken,
        access: tokens.accessToken,
        expires: Date.now() + tokens.expiresIn * 1000
      };
    },

    async refreshToken(credentials: OAuthCredentials, signal: AbortSignal): Promise<OAuthCredentials> {
      const tokens = await refreshAccessToken(credentials.refresh, signal);
      return {
        refresh: tokens.refreshToken ?? credentials.refresh,
        access: tokens.accessToken,
        expires: Date.now() + tokens.expiresIn * 1000
      };
    },

    getApiKey(credentials: OAuthCredentials): string {
      return credentials.access;
    }
  }
});
```

注册后，用户可以通过 `/login corporate-ai` 进行身份验证。

### OAuthLoginCallbacks

`callbacks` 对象为提供商自己的流程提供与 UI 无关的交互：

```typescript
interface OAuthLoginCallbacks {
  // 在浏览器中打开 URL（用于 OAuth 重定向）
  onAuth(params: { url: string }): void;

  // 显示设备码（用于设备授权流程）
  onDeviceCode(params: {
    userCode: string;
    verificationUri: string;
    intervalSeconds?: number;
    expiresInSeconds?: number;
  }): void;

  // 显示临时进度
  onProgress?(message: string): void;

  // 提示用户输入（用于手动输入 token）
  onPrompt(params: { message: string }): Promise<string>;

  // 显示交互式选择器，例如选择浏览器 OAuth 或设备码
  onSelect(params: {
    message: string;
    options: { id: string; label: string }[];
  }): Promise<string | undefined>;
}
```

### OAuthCredentials

凭据持久化到 `~/.pi/agent/auth.json`：

```typescript
interface OAuthCredentials {
  refresh: string;   // 刷新 token（供 refreshToken() 使用）
  access: string;    // 访问 token（由 getApiKey() 返回）
  expires: number;   // 过期时间戳，以毫秒为单位
}
```

## 自定义流式 API

对于使用非标准 API 的提供商，请实现 `streamSimple`。编写自己的实现之前，请先研究现有 API 实现：

**参考实现：**
- [anthropic-messages.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/anthropic-messages.ts) - Anthropic Messages API
- [mistral-conversations.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/mistral-conversations.ts) - Mistral Conversations API
- [openai-completions.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-completions.ts) - OpenAI Chat Completions
- [openai-responses.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-responses.ts) - OpenAI Responses API
- [google-generative-ai.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/google-generative-ai.ts) - Google Generative AI
- [bedrock-converse-stream.ts](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/bedrock-converse-stream.ts) - AWS Bedrock

### 流模式

所有提供商都遵循相同的模式：

```typescript
import {
  type AssistantMessage,
  type AssistantMessageEventStream,
  type Context,
  type Model,
  type SimpleStreamOptions,
  calculateCost,
  createAssistantMessageEventStream,
} from "@earendil-works/pi-ai";

function streamMyProvider(
  model: Model<any>,
  context: Context,
  options?: SimpleStreamOptions
): AssistantMessageEventStream {
  const stream = createAssistantMessageEventStream();

  (async () => {
    // 初始化输出消息
    const output: AssistantMessage = {
      role: "assistant",
      content: [],
      api: model.api,
      provider: model.provider,
      model: model.id,
      usage: {
        input: 0,
        output: 0,
        cacheRead: 0,
        cacheWrite: 0,
        totalTokens: 0,
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0, total: 0 },
      },
      stopReason: "pending",
      timestamp: Date.now(),
    };

    try {
      // 推送 start 事件
      stream.push({ type: "start", partial: output });

      // 发出 API 请求并处理响应……
      // 在内容事件到达时推送它们，并根据终止事件设置 stopReason。
      if (output.stopReason === "pending") {
        throw new Error("Provider stream ended without a stop reason");
      }
      if (output.stopReason === "error" || output.stopReason === "aborted") {
        throw new Error(output.errorMessage || "An unknown error occurred");
      }

      // 推送 done 事件
      stream.push({
        type: "done",
        reason: output.stopReason,
        message: output
      });
      stream.end();
    } catch (error) {
      output.stopReason = options?.signal?.aborted ? "aborted" : "error";
      output.errorMessage = error instanceof Error ? error.message : String(error);
      stream.push({ type: "error", reason: output.stopReason, error: output });
      stream.end();
    }
  })();

  return stream;
}
```

### 事件类型

按照以下顺序通过 `stream.push()` 推送事件：

1. `{ type: "start", partial: output }` - 流已启动

2. 内容事件（可重复，每个 block 都要跟踪 `contentIndex`）：
   - `{ type: "text_start", contentIndex, partial }` - 文本 block 已开始
   - `{ type: "text_delta", contentIndex, delta, partial }` - 文本片段
   - `{ type: "text_end", contentIndex, content, partial }` - 文本 block 已结束
   - `{ type: "thinking_start", contentIndex, partial }` - 思考已开始
   - `{ type: "thinking_delta", contentIndex, delta, partial }` - 思考片段
   - `{ type: "thinking_end", contentIndex, content, partial }` - 思考已结束
   - `{ type: "toolcall_start", contentIndex, partial }` - 工具调用已开始
   - `{ type: "toolcall_delta", contentIndex, delta, partial }` - 工具调用 JSON 片段
   - `{ type: "toolcall_end", contentIndex, toolCall, partial }` - 工具调用已结束

3. `{ type: "done", reason, message }` 或 `{ type: "error", reason, error }` - 流已结束

每个事件中的 `partial` 字段都包含当前的 `AssistantMessage` 状态。接收数据时更新 `output.content`，然后将 `output` 作为 `partial` 包含在事件中。

### 内容 Block

内容 block 到达时，将其添加到 `output.content`：

```typescript
// 文本 block
output.content.push({ type: "text", text: "" });
stream.push({ type: "text_start", contentIndex: output.content.length - 1, partial: output });

// 文本到达时
const block = output.content[contentIndex];
if (block.type === "text") {
  block.text += delta;
  stream.push({ type: "text_delta", contentIndex, delta, partial: output });
}

// block 完成时
stream.push({ type: "text_end", contentIndex, content: block.text, partial: output });
```

### 工具调用

工具调用需要累积并解析 JSON：

```typescript
// 开始工具调用
output.content.push({
  type: "toolCall",
  id: toolCallId,
  name: toolName,
  arguments: {}
});
stream.push({ type: "toolcall_start", contentIndex: output.content.length - 1, partial: output });

// 累积 JSON
let partialJson = "";
partialJson += jsonDelta;
try {
  block.arguments = JSON.parse(partialJson);
} catch {}
stream.push({ type: "toolcall_delta", contentIndex, delta: jsonDelta, partial: output });

// 完成
stream.push({
  type: "toolcall_end",
  contentIndex,
  toolCall: { type: "toolCall", id, name, arguments: block.arguments },
  partial: output
});
```

### 用量和费用

根据 API 响应更新用量并计算费用：

```typescript
output.usage.input = response.usage.input_tokens;
output.usage.output = response.usage.output_tokens;
output.usage.cacheRead = response.usage.cache_read_tokens ?? 0;
output.usage.cacheWrite = response.usage.cache_write_tokens ?? 0;
output.usage.totalTokens = output.usage.input + output.usage.output +
                           output.usage.cacheRead + output.usage.cacheWrite;
calculateCost(model, output.usage);
```

### 上下文溢出错误

当请求超出模型的上下文窗口时，pi 可以通过压缩对话并重试来自动恢复。只有当 pi 将故障识别为溢出时，才会触发此恢复机制。

检测针对最终确定的助手消息运行：

- `stopReason === "error"`
- `errorMessage` 与 pi 已知的某种溢出模式匹配（参见 [`packages/ai/src/utils/overflow.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/utils/overflow.ts)）

如果你的提供商返回的溢出错误消息无法被 pi 识别，请在注册该提供商的同一个扩展中规范化错误。使用 `message_end` 处理程序重写助手消息，让其 `errorMessage` 以 pi 能识别的短语开头。通用回退值 `context_length_exceeded` 是最安全的选择。

```typescript
const MY_PROVIDER_OVERFLOW_PATTERN = /your provider's overflow phrase/i;

export default function (pi: ExtensionAPI) {
  pi.registerProvider("my-provider", { /* ... */ });

  pi.on("message_end", (event, ctx) => {
    const message = event.message;
    if (message.role !== "assistant") return;
    if (message.stopReason !== "error") return;
    if (
      message.provider !== "my-provider" &&
      ctx.model?.provider !== "my-provider"
    )
      return;

    const errorMessage = message.errorMessage ?? "";
    if (errorMessage.includes("context_length_exceeded")) return;
    if (!MY_PROVIDER_OVERFLOW_PATTERN.test(errorMessage)) return;

    return {
      message: {
        ...message,
        errorMessage: `context_length_exceeded: ${errorMessage}`,
      },
    };
  });
}
```

`message_end` 会在 pi 为自动压缩跟踪助手消息之前运行，因此 pi 检查的是重写后的 `errorMessage`。采用此设置后，pi 将：

1. 从 `errorMessage` 检测到溢出。
2. 从实时上下文中丢弃失败的助手消息。
3. 运行压缩。
4. 重试请求一次。

请谨慎设置重写守卫：

- 将其作用域限制在你的提供商（`message.provider` 和 `ctx.model?.provider`），从而不触碰来自其他提供商的无关错误。
- 匹配提供商特定的模式，而不是 pi 的通用溢出模式。重写速率限制或限流错误（`rate limit`、`too many requests`）会错误触发压缩，而不是 pi 正常的指数退避重试路径。
- 当 `errorMessage` 已包含 `context_length_exceeded` 时跳过，使处理程序具有幂等性。

### 注册

注册你的流函数：

```typescript
pi.registerProvider("my-provider", {
  baseUrl: "https://api.example.com",
  apiKey: "$MY_API_KEY",
  api: "my-custom-api",
  models: [...],
  streamSimple: streamMyProvider
});
```

## 测试你的实现

使用内置提供商所用的相同测试套件测试你的提供商。从 [packages/ai/test/](https://github.com/earendil-works/pi/tree/main/packages/ai/test) 复制并调整以下测试文件：

| 测试 | 用途 |
|------|---------|
| `stream.test.ts` | 基本流式传输、文本输出 |
| `tokens.test.ts` | Token 计数和用量 |
| `abort.test.ts` | AbortSignal 处理 |
| `empty.test.ts` | 空响应/最小响应 |
| `context-overflow.test.ts` | 上下文窗口限制 |
| `image-limits.test.ts` | 图像输入处理 |
| `unicode-surrogate.test.ts` | Unicode 边界情况 |
| `tool-call-without-result.test.ts` | 工具调用边界情况 |
| `image-tool-result.test.ts` | 工具结果中的图像 |
| `total-tokens.test.ts` | 总 token 计算 |
| `cross-provider-handoff.test.ts` | 提供商之间的上下文交接 |

使用你的提供商/模型组合运行测试以验证兼容性。

## 配置参考

```typescript
interface ProviderConfig {
  /** 提供商在 UI（例如 /login）中显示的名称。 */
  name?: string;

  /** API 端点 URL。定义模型时必需。 */
  baseUrl?: string;

  /** API key 字面值、环境变量插值（$ENV_VAR 或 ${ENV_VAR}）或 !command。定义模型时必需（使用 oauth 时除外）。 */
  apiKey?: string;

  /** 流式传输的 API 类型。在提供商或模型级别定义模型时必需。 */
  api?: Api;

  /** 非标准 API 的自定义流式传输实现。 */
  streamSimple?: (
    model: Model<Api>,
    context: Context,
    options?: SimpleStreamOptions
  ) => AssistantMessageEventStream;

  /** 请求中包含的自定义 header。值使用与 apiKey 相同的解析语法。 */
  headers?: Record<string, string>;

  /** 若为 true，则添加包含已解析 API key 的 Authorization: Bearer header。 */
  authHeader?: boolean;

  /** 要注册的模型。如果提供，则替换该提供商的所有现有模型。 */
  models?: ProviderModelConfig[];

  /** 为 /login 提供支持的 OAuth 提供商。 */
  oauth?: {
    name: string;
    login(callbacks: OAuthLoginCallbacks): Promise<OAuthCredentials>;
    refreshToken(credentials: OAuthCredentials, signal: AbortSignal): Promise<OAuthCredentials>;
    getApiKey(credentials: OAuthCredentials): string;
  };
}
```

## 模型定义参考

```typescript
interface ProviderModelConfig {
  /** 模型 ID（例如 "claude-sonnet-4-20250514"）。 */
  id: string;

  /** 显示名称（例如 "Claude 4 Sonnet"）。 */
  name: string;

  /** 此特定模型的 API 类型覆盖。 */
  api?: Api;

  /** 此特定模型的 API 端点 URL 覆盖。 */
  baseUrl?: string;

  /** 模型是否支持扩展思考。 */
  reasoning: boolean;

  /** 将 pi 思考级别映射到提供商/模型特定的值；null 表示不支持该级别。 */
  thinkingLevelMap?: Partial<Record<"off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max", string | null>>;

  /** 支持的输入类型。 */
  input: ("text" | "image")[];

  /** 每百万 token 的费用（用于用量跟踪）。 */
  cost: {
    input: number;
    output: number;
    cacheRead: number;
    cacheWrite: number;
  };

  /** 最大上下文窗口大小，以 token 为单位。 */
  contextWindow: number;

  /** 最大输出 token 数。 */
  maxTokens: number;

  /** 此特定模型的自定义 header。 */
  headers?: Record<string, string>;

  /** 所选 API 的兼容性设置。 */
  compat?: {
    // openai-completions
    supportsStore?: boolean;
    supportsDeveloperRole?: boolean;
    supportsReasoningEffort?: boolean;
    supportsUsageInStreaming?: boolean;
    supportsFinishReason?: boolean;
    supportsStrictMode?: boolean;
    supportsOpenAIGrammarTools?: boolean; // openai-completions/openai-responses；false 时回退到普通 function 工具
    maxTokensField?: "max_completion_tokens" | "max_tokens";
    requiresToolResultName?: boolean;
    requiresAssistantAfterToolResult?: boolean;
    requiresThinkingAsText?: boolean;
    requiresReasoningContentOnAssistantMessages?: boolean;
    thinkingFormat?: "openai" | "openrouter" | "deepseek" | "together" | "baseten" | "zai" | "qwen" | "chat-template" | "qwen-chat-template" | "string-thinking" | "ant-ling";
    chatTemplateKwargs?: Record<string, string | number | boolean | null | { "$var": "thinking.enabled" | "thinking.effort" | "thinking.budget"; omitWhenOff?: boolean }>;
    chatTemplateArgs?: Record<string, string | number | boolean | null | { "$var": "thinking.enabled" | "thinking.effort" | "thinking.budget"; omitWhenOff?: boolean }>;
    thinkingTokenBudgetField?: "thinking_token_budget" | "thinking_budget" | "thinking_budget_tokens";
    supportsThinkingTokenBudget?: boolean;
    cacheControlFormat?: "anthropic";
    sessionAffinityFormat?: "openai" | "openai-nosession" | "openrouter";
    sendSessionAffinityHeaders?: boolean;

    // anthropic-messages
    supportsEagerToolInputStreaming?: boolean;
    supportsLongCacheRetention?: boolean;
    sendSessionAffinityHeaders?: boolean;
    supportsCacheControlOnTools?: boolean;
    forceAdaptiveThinking?: boolean;
    allowEmptySignature?: boolean;
    supportsStrictTools?: boolean;
  };
}
```

`openrouter` 会发送 `reasoning: { effort }`。`deepseek` 会发送 `thinking: { type: "enabled" | "disabled" }`，并在启用时发送 `reasoning_effort`。`together` 会发送 `reasoning: { enabled }`，并在启用 `supportsReasoningEffort` 时也发送 `reasoning_effort`。`qwen` 用于 DashScope 风格的顶层 `enable_thinking`。对于读取 `chat_template_kwargs.enable_thinking` 且需要 `preserve_thinking` 的本地 Qwen 兼容服务器，请使用 `qwen-chat-template`。对于可配置的 `chat_template_kwargs`，请使用 `chat-template`，例如通过 vLLM 运行的 DeepSeek V3.x 可设置 `chatTemplateKwargs: { "thinking": { "$var": "thinking.enabled" } }`。如果提供商期望在 `chat_template_args` 下提供开关值，并且可选择支持顶层 `reasoning_effort`，请将 `thinkingFormat: "baseten"` 与 `chatTemplateArgs` 一起使用。
`thinkingTokenBudgetField` 会将按级别限制范围后的思考预算作为顶层请求字段发送（vLLM 上为 `thinking_token_budget`，Qwen/SGLang 上为 `thinking_budget`，llama.cpp 上为 `thinking_budget_tokens`）。`supportsThinkingTokenBudget: true` 是 vLLM 字段名称的别名。不要在 DashScope Qwen 模型上将其与 `reasoning_effort` 结合使用。
`cacheControlFormat: "anthropic"` 会将 Anthropic 风格的 `cache_control` 标记应用到 system prompt、最后一个工具定义，以及最后一段用户、助手或工具结果文本内容。
