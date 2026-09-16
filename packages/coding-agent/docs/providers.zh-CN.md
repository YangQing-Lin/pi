# 提供商

Pi 支持通过 OAuth 使用订阅型提供商，也支持通过环境变量或身份验证文件使用 API key 提供商。内置目录随 Pi 一同提供；已配置的提供商可以刷新较新的目录，并将其缓存在 `~/.pi/agent/models-store.json` 中供离线使用。

## 目录

- [订阅](#订阅)
- [API key](#api-key)
- [身份验证文件](#身份验证文件)
- [云提供商](#云提供商)
- [llama.cpp](#llamacpp)
- [自定义提供商](#自定义提供商)
- [解析顺序](#解析顺序)

## 订阅

在交互模式中使用 `/login`，然后选择提供商：

- ChatGPT Plus/Pro（Codex）
- Claude Pro/Max
- GitHub Copilot
- xAI（Grok/X 订阅）
- OpenRouter（通过 OAuth 生成 API key，从 OpenRouter 额度计费）
- Radius

使用 `/logout` 清除凭据。token 存储在 `~/.pi/agent/auth.json` 中，并在过期时自动刷新。OpenRouter 则会生成由用户控制、不会自动过期的 API key。

### OpenAI Codex

- 需要 ChatGPT Plus 或 Pro 订阅
- 获得 OpenAI 官方认可：[Codex for OSS](https://developers.openai.com/community/codex-for-oss)

### Claude Pro/Max

Claude Pro/Max 账户可以使用 Anthropic 订阅身份验证。第三方执行框架的用量会计入[额外用量](https://claude.ai/settings/usage)，按 token 计费，不占用 Claude 套餐限额。

### GitHub Copilot

- 对 github.com 直接按 Enter，或者输入你的 GitHub Enterprise Server 域名
- 如果出现“model not supported”，请在 VS Code 中启用：Copilot Chat → 模型选择器 → 选择模型 → “Enable”

### xAI（Grok/X 订阅）

- 运行 `/login xai`，然后选择 **Use a subscription**
- 仍可通过 **Use an API key** 使用 `XAI_API_KEY`

### OpenRouter

- 运行 `/login openrouter`，然后选择 **Sign in with OpenRouter**，打开 OpenRouter PKCE 授权流程
- 授权会创建由用户控制的 OpenRouter API key，并从你的 OpenRouter 额度计费
- 在远程/无头计算机上（例如通过 SSH），浏览器无法访问回环回调；请改为将最终重定向 URL（或授权代码）粘贴到登录提示中
- 仍可通过 **Use an API key** 使用 `OPENROUTER_API_KEY`

### Radius

Radius 是动态 `pi-messages` gateway。`/login radius` 将 OAuth token 存入 `auth.json`；gateway 目录会独立刷新并缓存在 `models-store.json` 中。可以在 `models.json` 中使用 `"oauth": "radius"` 和 gateway `baseUrl` 声明自定义 Radius gateway。

## API key

### 环境变量或身份验证文件

在交互模式中使用 `/login` 并选择提供商，可将 API key 存入 `auth.json`；也可以通过环境变量设置凭据：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

| 提供商 | 环境变量 | `auth.json` key |
|----------|----------------------|------------------|
| Anthropic | `ANTHROPIC_API_KEY` | `anthropic` |
| Ant Ling | `ANT_LING_API_KEY` | `ant-ling` |
| Azure OpenAI Responses | `AZURE_OPENAI_API_KEY` | `azure-openai-responses` |
| OpenAI | `OPENAI_API_KEY` | `openai` |
| DeepSeek | `DEEPSEEK_API_KEY` | `deepseek` |
| NVIDIA NIM | `NVIDIA_API_KEY` | `nvidia` |
| Google Gemini | `GEMINI_API_KEY` | `google` |
| Amazon Bedrock | `AWS_BEARER_TOKEN_BEDROCK` | `amazon-bedrock` |
| Mistral | `MISTRAL_API_KEY` | `mistral` |
| Groq | `GROQ_API_KEY` | `groq` |
| Cerebras | `CEREBRAS_API_KEY` | `cerebras` |
| Cloudflare AI Gateway | `CLOUDFLARE_API_KEY`（以及 `CLOUDFLARE_ACCOUNT_ID`、`CLOUDFLARE_GATEWAY_ID`） | `cloudflare-ai-gateway` |
| Cloudflare Workers AI | `CLOUDFLARE_API_KEY`（以及 `CLOUDFLARE_ACCOUNT_ID`） | `cloudflare-workers-ai` |
| xAI | `XAI_API_KEY` | `xai` |
| OpenRouter | `OPENROUTER_API_KEY` | `openrouter` |
| Vercel AI Gateway | `AI_GATEWAY_API_KEY` | `vercel-ai-gateway` |
| ZAI Coding Plan（全球） | `ZAI_API_KEY` | `zai` |
| ZAI Coding Plan（中国） | `ZAI_CODING_CN_API_KEY` | `zai-coding-cn` |
| OpenCode Zen | `OPENCODE_API_KEY` | `opencode` |
| OpenCode Go | `OPENCODE_API_KEY` | `opencode-go` |
| Radius | `RADIUS_API_KEY` | `radius` |
| Hugging Face | `HF_TOKEN` | `huggingface` |
| Fireworks | `FIREWORKS_API_KEY` | `fireworks` |
| Together AI | `TOGETHER_API_KEY` | `together` |
| Baseten | `BASETEN_API_KEY` | `baseten` |
| Kimi For Coding | `KIMI_API_KEY` | `kimi-coding` |
| MiniMax | `MINIMAX_API_KEY` | `minimax` |
| MiniMax（中国） | `MINIMAX_CN_API_KEY` | `minimax-cn` |
| Qwen Token Plan（现有目录） | `QWEN_TOKEN_PLAN_API_KEY` | `qwen-token-plan` |
| Qwen Token Plan（个人版） | `QWEN_TOKEN_PLAN_API_KEY` | `qwen-token-plan-individual` |
| Qwen Token Plan（中国） | `QWEN_TOKEN_PLAN_CN_API_KEY` | `qwen-token-plan-cn` |
| Xiaomi MiMo | `XIAOMI_API_KEY` | `xiaomi` |
| Xiaomi MiMo Token Plan（中国） | `XIAOMI_TOKEN_PLAN_CN_API_KEY` | `xiaomi-token-plan-cn` |
| Xiaomi MiMo Token Plan（阿姆斯特丹） | `XIAOMI_TOKEN_PLAN_AMS_API_KEY` | `xiaomi-token-plan-ams` |
| Xiaomi MiMo Token Plan（新加坡） | `XIAOMI_TOKEN_PLAN_SGP_API_KEY` | `xiaomi-token-plan-sgp` |

环境变量和 `auth.json` key 的参考实现是 [`packages/ai/src/env-api-keys.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/env-api-keys.ts) 中的 [`const envMap`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/env-api-keys.ts)。

#### 身份验证文件

在 `~/.pi/agent/auth.json` 中存储凭据：

```json
{
  "anthropic": { "type": "api_key", "key": "sk-ant-..." },
  "ant-ling": { "type": "api_key", "key": "..." },
  "openai": { "type": "api_key", "key": "sk-..." },
  "deepseek": { "type": "api_key", "key": "sk-..." },
  "nvidia": { "type": "api_key", "key": "nvapi-..." },
  "google": { "type": "api_key", "key": "..." },
  "opencode": { "type": "api_key", "key": "..." },
  "opencode-go": { "type": "api_key", "key": "..." },
  "together": { "type": "api_key", "key": "..." },
  "qwen-token-plan":  { "type": "api_key", "key": "sk-sp-..." },
  "qwen-token-plan-individual": { "type": "api_key", "key": "sk-sp-..." },
  "qwen-token-plan-cn": { "type": "api_key", "key": "sk-sp-..." },
  "xiaomi": { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-cn":  { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-ams": { "type": "api_key", "key": "..." },
  "xiaomi-token-plan-sgp": { "type": "api_key", "key": "..." }
}
```

`qwen-token-plan-individual` 使用与 `qwen-token-plan` 相同的国际端点和 `QWEN_TOKEN_PLAN_API_KEY`，但选择器仅显示个人订阅文档中列出的模型。现有提供商为保持向后兼容而保留更广泛的目录。使用 `auth.json` 时，请将凭据存储在你选择的提供商下；两个国际提供商共享同一个环境变量。

文件创建时使用 `0600` 权限（仅用户可读写）。身份验证文件中的凭据优先于环境变量。

API key 凭据还可以包含提供商范围的环境值。解析凭据 key、提供商/模型 header，以及 Cloudflare 账户 ID、Azure OpenAI 设置、Vertex 项目/位置、Bedrock 设置、`PI_CACHE_RETENTION` 和 `HTTP_PROXY`/`HTTPS_PROXY` 等提供商配置时，这些值会优先于进程环境变量使用。

```json
{
  "cloudflare-ai-gateway": {
    "type": "api_key",
    "key": "$CLOUDFLARE_API_KEY",
    "env": {
      "CLOUDFLARE_API_KEY": "...",
      "CLOUDFLARE_ACCOUNT_ID": "account-id",
      "CLOUDFLARE_GATEWAY_ID": "gateway-id"
    }
  }
}
```

当 Pi 应使用不同于项目 shell 环境的提供商设置时，请使用此功能。

### Key 解析

`key` 字段支持执行命令、环境变量插值和字面值：

- **Shell 命令：** 值以 `"!command"` 开头时，会将整个值作为命令执行并使用 stdout（在进程生命周期内缓存）
  ```json
  { "type": "api_key", "key": "!security find-generic-password -ws 'anthropic'" }
  { "type": "api_key", "key": "!op read 'op://vault/item/credential'" }
  ```
- **环境变量插值：** `"$ENV_VAR"` 或 `"${ENV_VAR}"` 使用指定变量的值。插值也可用于较长的字面值内部。
  ```json
  { "type": "api_key", "key": "$MY_ANTHROPIC_KEY" }
  { "type": "api_key", "key": "${KEY_PREFIX}_${KEY_SUFFIX}" }
  ```
  `$FOO_BAR` 表示变量 `FOO_BAR`；当 `BAR` 是字面文本时，请使用 `${FOO}_BAR`。缺少环境变量会使该值无法解析。
- **转义：** `"$$"` 生成字面量 `"$"`；`"$!"` 生成字面量 `"!"`，且不会触发命令执行。
  ```json
  { "type": "api_key", "key": "$$literal-dollar-prefix" }
  { "type": "api_key", "key": "$!literal-bang-prefix" }
  ```
- **字面值：** 直接使用。`MY_API_KEY` 等纯大写字符串是字面值；环境变量请使用 `$MY_API_KEY`。
  ```json
  { "type": "api_key", "key": "sk-ant-..." }
  { "type": "api_key", "key": "public" }
  ```

通过 `/login` 获得的 OAuth 凭据也存储在这里，并自动管理。

## 云提供商

### Azure OpenAI

```bash
export AZURE_OPENAI_API_KEY=...
export AZURE_OPENAI_BASE_URL=https://your-resource.ai.azure.com
# 同样支持：https://your-resource.cognitiveservices.azure.com
# 同样支持：https://your-resource.openai.azure.com
# 根端点会自动规范化为 /openai/v1
# 也可以使用资源名称代替基础 URL
export AZURE_OPENAI_RESOURCE_NAME=your-resource

# 可选
export AZURE_OPENAI_API_VERSION=2024-02-01
export AZURE_OPENAI_DEPLOYMENT_NAME_MAP=gpt-4=my-gpt4,gpt-4o=my-gpt4o
```

### Amazon Bedrock

使用 `/login amazon-bedrock` 存储 Bedrock API key，或配置以下任一 AWS 环境凭据来源：

```bash
# 方案 1：AWS Profile
export AWS_PROFILE=your-profile

# 方案 2：IAM Key
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...

# 方案 3：Bearer Token
export AWS_BEARER_TOKEN_BEDROCK=...

# 可选区域（默认为 us-east-1）
export AWS_REGION=us-west-2
```

还支持 ECS task role（`AWS_CONTAINER_CREDENTIALS_*`）和 IRSA（`AWS_WEB_IDENTITY_TOKEN_FILE`）。

```bash
pi --provider amazon-bedrock --model us.anthropic.claude-sonnet-4-20250514-v1:0
```

对于 ID 中包含可识别模型名称的 Claude 模型（基础模型和系统定义的推理 profile），会自动启用提示缓存。对于 ARN 中不含模型名称的应用推理 profile，请设置 `AWS_BEDROCK_FORCE_CACHE=1` 启用缓存点：

```bash
export AWS_BEDROCK_FORCE_CACHE=1
pi --provider amazon-bedrock --model arn:aws:bedrock:us-east-1:123456789012:application-inference-profile/abc123
```

如果连接到 Bedrock API 代理，可以使用以下环境变量：

```bash
# 设置 Bedrock 代理 URL（标准 AWS SDK 环境变量）
export AWS_ENDPOINT_URL_BEDROCK_RUNTIME=https://my.corp.proxy/bedrock

# 代理不需要身份验证时设置
export AWS_BEDROCK_SKIP_AUTH=1

# 代理仅支持 HTTP/1.1 时设置
export AWS_BEDROCK_FORCE_HTTP1=1
```

### Cloudflare AI Gateway

可以通过 `/login` 设置 `CLOUDFLARE_API_KEY`。账户 ID 和 gateway slug 可以作为环境变量设置，也可以放在 `auth.json` 中 API key 凭据的 `env` 对象内。

```bash
export CLOUDFLARE_API_KEY=...           # 或使用 /login
export CLOUDFLARE_ACCOUNT_ID=...
export CLOUDFLARE_GATEWAY_ID=...        # 在 dash.cloudflare.com → AI → AI Gateway 中创建
pi --provider cloudflare-ai-gateway --model "claude-sonnet-4-5"
```

通过 Cloudflare AI Gateway 将请求路由到 OpenAI、Anthropic 和 Workers AI。Workers AI 使用 Unified API（`/compat`）和带前缀的模型 ID（`workers-ai/@cf/...`）。OpenAI 使用 OpenAI 透传路由（`/openai`）和 `gpt-5.1` 等原生 OpenAI 模型 ID。Anthropic 使用 Anthropic 透传路由（`/anthropic`）和 `claude-sonnet-4-5` 等原生 Anthropic 模型 ID。

AI Gateway 身份验证使用 `CLOUDFLARE_API_KEY` 作为 `cf-aig-authorization`。上游身份验证可以是以下方式之一：

| 模式 | 请求身份验证 | 上游身份验证 |
|------|--------------|---------------|
| Workers AI | 仅 Cloudflare token | Cloudflare 原生 |
| 统一计费 | 仅 Cloudflare token | Cloudflare 处理上游身份验证并扣除额度 |
| 存储式 BYOK | 仅 Cloudflare token | Cloudflare 注入存储在 AI Gateway 控制面板中的提供商 key |
| 内联 BYOK | Cloudflare token 加上游 `Authorization` header | 请求提供上游提供商 key |

对于常规 Pi 用法，优先使用统一计费或存储式 BYOK。内联 BYOK 需要为 Cloudflare AI Gateway 提供商配置额外的上游 `Authorization` header，例如通过 `models.json` 提供商/模型覆盖设置。

### Cloudflare Workers AI

可以通过 `/login` 设置 `CLOUDFLARE_API_KEY`。`CLOUDFLARE_ACCOUNT_ID` 可以设为环境变量，也可以放在 `auth.json` 中 API key 凭据的 `env` 对象内。

```bash
export CLOUDFLARE_API_KEY=...           # 或使用 /login
export CLOUDFLARE_ACCOUNT_ID=...
pi --provider cloudflare-workers-ai --model "@cf/moonshotai/kimi-k2.6"
```

Pi 会自动设置 `x-session-affinity`，以获得[前缀缓存](https://developers.cloudflare.com/workers-ai/features/prompt-caching/)折扣。

### Google Vertex AI

使用 Application Default Credentials：

```bash
gcloud auth application-default login
export GOOGLE_CLOUD_PROJECT=your-project
export GOOGLE_CLOUD_LOCATION=us-central1
```

也可以将 `GOOGLE_APPLICATION_CREDENTIALS` 设为 service account key 文件。

## llama.cpp

Pi 支持 llama.cpp 路由服务器。使用 `/login llama.cpp` 配置，通过 `/llama` 管理已加载模型，并使用 `/model` 选择已加载模型。

有关服务器配置、模型目录布局、环境变量和命令用法，请参阅 [llama.cpp](llama-cpp.zh-CN.md)。

## 自定义提供商

**通过 models.json：** 添加 Ollama、LM Studio、vLLM 或任何使用受支持 API（OpenAI Completions、OpenAI Responses、Anthropic Messages、Google Generative AI）的提供商。参阅 [models.md](models.zh-CN.md)。

**通过扩展：** 对于需要自定义 API 实现或 OAuth 流程的提供商，请创建扩展。参阅 [custom-provider.md](custom-provider.zh-CN.md) 和 [examples/extensions/custom-provider-gitlab-duo](../examples/extensions/custom-provider-gitlab-duo/)。

## 解析顺序

解析提供商凭据时，按以下顺序：

1. CLI `--api-key` flag
2. `auth.json` 条目（API key 或 OAuth token）
3. 环境变量
4. `models.json` 中的自定义提供商 key
