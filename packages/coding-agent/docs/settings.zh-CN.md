# 设置

Pi 使用 JSON 设置文件，项目设置会覆盖全局设置。

| 位置 | 作用域 |
|----------|-------|
| `~/.pi/agent/settings.json` | 全局（所有项目） |
| `.pi/settings.json` | 项目（当前目录） |

可以直接编辑，也可以使用 `/settings` 配置常用选项。要以交互方式保存启动模型默认值，请使用 `/model` 并在所需模型上按 Ctrl+S。要保存启动思考级别，请使用 `/thinking` 并按 Ctrl+S。

## 项目信任

交互启动时，如果项目文件夹包含项目本地设置、资源或项目 `.agents/skills`，且 `~/.pi/agent/trust.json` 中没有该文件夹或父文件夹的已保存决定，Pi 会先询问是否信任。信任项目后，Pi 可以加载 `.pi/settings.json` 和 `.pi` 资源、安装缺失的项目包并执行项目扩展。

非交互模式（`-p`、`--mode json` 和 `--mode rpc`）不会显示信任提示。没有适用的已保存决定时，它们使用全局设置中的 `defaultProjectTrust`：`ask`（默认）和 `never` 会忽略这些项目资源，而 `always` 会信任它们。传入 `--approve`/`-a` 或 `--no-approve`/`-na` 可覆盖单次运行的项目信任设置。

如果扩展和已保存决定均不适用，`defaultProjectTrust` 控制回退行为。在 `~/.pi/agent/settings.json` 中将其设为 `"ask"`、`"always"` 或 `"never"`，也可以通过 `/settings` 更改。

`pi config` 和包命令使用相同的项目信任流程，但 `pi update` 绝不会提示。传入 `--approve` 可在单次命令中信任项目本地设置，传入 `--no-approve` 则忽略它们。

在交互模式中使用 `/trust` 可保存项目信任决定，供未来会话使用，也可以信任直接父文件夹。它只写入 `~/.pi/agent/trust.json`；当前会话不会重新加载，因此需重启 Pi 才能应用更改。

## 所有设置

### 模型与思考

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `defaultProvider` | string | - | 启动提供商（例如 `"anthropic"`、`"openai"`；可在 `/model` 中按 Ctrl+S 保存，或手动编辑） |
| `defaultModel` | string | - | 启动模型 ID（可在 `/model` 中按 Ctrl+S 保存，或手动编辑） |
| `defaultThinkingLevel` | string | - | 启动思考级别（可在 `/thinking` 中按 Ctrl+S 保存，或手动编辑）：`"off"`、`"minimal"`、`"low"`、`"medium"`、`"high"`、`"xhigh"`、`"max"` |
| `modelThinkingLevels` | object | - | 按 `"provider/modelId"` 设置每个模型的启动思考级别；可通过 `/settings` → Default thinking level per model 配置，或手动编辑 |
| `hideThinkingBlock` | boolean | `false` | 在输出中隐藏思考块 |
| `showCacheMissNotices` | boolean | `false` | 显示明显的提示缓存未命中、上下文压缩或分支摘要用量，以及丢弃 Anthropic 思考块等提供商恢复诊断通知 |
| `thinkingBudgets` | object | - | 每个思考级别的自定义 token 预算。Anthropic、Google 和 Bedrock 原生使用；OpenAI 兼容模型在设置 `compat.thinkingTokenBudgetField`（或 `supportsThinkingTokenBudget`）时使用。 |

#### thinkingBudgets

```json
{
  "thinkingBudgets": {
    "minimal": 1024,
    "low": 4096,
    "medium": 10240,
    "high": 32768
  }
}
```

### UI 与显示

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `theme` | string | `"dark"` | 主题名称（`"dark"`、`"light"` 或自定义主题） |
| `externalEditor` | string | `$VISUAL`，然后 `$EDITOR`，再回退到 Windows 上的 Notepad 或其他平台上的 `nano` | Ctrl+G 外部编辑器命令；优先于环境变量 |
| `quietStartup` | boolean | `false` | 隐藏启动标题区 |
| `defaultProjectTrust` | string | `"ask"` | 项目信任回退行为：`"ask"`、`"always"` 或 `"never"`。仅限全局设置 |
| `collapseChangelog` | boolean | `false` | 更新后显示精简的 changelog |
| `enableInstallTelemetry` | boolean | `true` | 发送匿名安装/更新 ping 和所选提供商归因 header。不控制更新检查 |
| `enableAnalytics` | boolean | `false` | 选择加入分析数据分享。目前只在实验性首次配置（`PI_EXPERIMENTAL=1`）时询问 |
| `trackingId` | string | - | 分析跟踪标识符，启用 `enableAnalytics` 时生成 |
| `doubleEscapeAction` | string | `"tree"` | 连按两次 Escape 的操作：`"tree"`、`"fork"` 或 `"none"` |
| `treeFilterMode` | string | `"default"` | `/tree` 默认筛选器：`"default"`、`"no-tools"`、`"user-only"`、`"labeled-only"`、`"all"` |
| `editorPaddingX` | number | `0` | 输入编辑器水平内边距（0–3） |
| `outputPad` | number | `1` | 用户消息、assistant 消息和思考内容的水平内边距（0 或 1） |
| `autocompleteMaxVisible` | number | `5` | 自动补全下拉菜单中最多显示的项目数（3–20） |
| `showHardwareCursor` | boolean | `false` | TUI 为支持 IME 定位光标时显示终端光标 |
| `tuiMode` | string | `"regular"` | 交互 TUI 模式：`"regular"` 或实验性的 `"fullscreen"`。在 `/settings` 中更改会立即应用；启动时 `--tui-mode` 覆盖此设置 |
| `fullscreenExitOutput` | string | `"transcript"` | 全屏退出输出：`"transcript"` 打印最终记录和恢复提示；`"resume-hint"` 恢复先前屏幕并只打印恢复提示。对常规 TUI 模式无效 |
| `fullscreenScrollbar` | string | `"auto"` | 全屏记录滚动条：`"auto"` 在滚动时或指针位于最右侧轨道时临时显示；`"always"` 保留该列并始终显示；`"hidden"` 隐藏。对常规 TUI 模式无效 |
| `fullscreenCopyOnSelect` | boolean | `true` | 全屏模式下自动复制所选文本。禁用时选择保持高亮，使用 `Ctrl+X` 复制活动选择 |

对于 VS Code，请包含 `--wait`，让 Pi 在编辑器退出后恢复：

```json
{
  "externalEditor": "code --wait"
}
```

### 遥测和更新检查

`enableInstallTelemetry` 控制发往 `https://pi.dev/api/report-install` 的匿名安装/更新 ping，以及 OpenRouter、NVIDIA NIM 和 Cloudflare 提供商请求中的 Pi 归因 header。选择退出会同时禁用两者，但不会禁用更新检查；Pi 仍可获取 `https://pi.dev/api/latest-version` 查找最新版本。

设置 `PI_SKIP_VERSION_CHECK=1` 可禁用 Pi 版本更新检查。使用 `--offline` 或 `PI_OFFLINE=1` 可禁用这里描述的所有启动网络操作，包括更新检查、包更新检查和安装/更新遥测。

### 网络

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `httpProxy` | string | - | 作为 `HTTP_PROXY` 和 `HTTPS_PROXY` 应用的 HTTP 代理 URL。仅限全局设置。 |

```json
{
  "httpProxy": "http://127.0.0.1:7890"
}
```

### 警告

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `warnings.anthropicExtraUsage` | boolean | `true` | Anthropic 订阅身份验证可能产生付费额外用量时显示警告 |

```json
{
  "warnings": {
    "anthropicExtraUsage": false
  }
}
```

### 上下文压缩

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `compaction.enabled` | boolean | `true` | 启用自动压缩 |
| `compaction.reserveTokens` | number | `16384` | 为 LLM 回复预留的 token |
| `compaction.keepRecentTokens` | number | `20000` | 保留的近期 token（不总结） |

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

### 分支摘要

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `branchSummary.reserveTokens` | number | `16384` | 选择分支历史时预留的 token；输出最多 4096 token |
| `branchSummary.skipPrompt` | boolean | `false` | `/tree` 导航时跳过“Summarize branch?”提示（默认为不生成摘要） |

### 重试

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `retry.enabled` | boolean | `true` | 对瞬态错误启用 agent 级自动重试 |
| `retry.maxRetries` | number | `3` | agent 级最大重试次数 |
| `retry.baseDelayMs` | number | `2000` | agent 级指数退避基础延迟（2s、4s、8s） |
| `retry.provider.timeoutMs` | number | SDK 默认值 | 提供商/SDK 请求超时（毫秒） |
| `retry.provider.maxRetries` | number | `0` | 提供商/SDK 重试次数 |
| `retry.provider.maxRetryDelayMs` | number | `60000` | 失败前允许服务器请求的最大延迟（60s） |

提供商请求的重试延迟超过 `retry.provider.maxRetryDelayMs` 时，请求会立即失败并提供明确错误，而不是静默等待。设为 `0` 可禁用限制。

除非明确需要提供商级重试，否则请将 `retry.provider.maxRetries` 保持为 `0`。设为大于 `0` 可能让 SDK/提供商在 Pi 收到超出用量限制错误之前进行重试，某些情况下会阻塞 agent，直到提供商配额重置。

```json
{
  "retry": {
    "enabled": true,
    "maxRetries": 3,
    "baseDelayMs": 2000,
    "provider": {
      "timeoutMs": 3600000,
      "maxRetries": 0,
      "maxRetryDelayMs": 60000
    }
  }
}
```

### 消息投递

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `steeringMode` | string | `"one-at-a-time"` | 引导消息发送方式：`"all"` 或 `"one-at-a-time"` |
| `followUpMode` | string | `"one-at-a-time"` | 后续消息发送方式：`"all"` 或 `"one-at-a-time"` |
| `transport` | string | `"auto"` | 支持多种传输方式的提供商所用的首选传输：`"sse"`、`"websocket"`、`"websocket-cached"` 或 `"auto"` |
| `httpIdleTimeoutMs` | number | `300000` | HTTP header/body 空闲超时（毫秒），也用于具有显式流空闲超时的提供商。设为 `0` 可禁用。 |
| `websocketConnectTimeoutMs` | number | `15000` | 支持 WebSocket 传输的提供商所用的 WebSocket 连接/打开握手超时（毫秒）。设为 `0` 可禁用。 |

### 终端与图像

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `terminal.showImages` | boolean | `true` | 在终端中显示图像（如果支持） |
| `terminal.imageWidthCells` | number | `60` | 内联图像的首选终端单元格宽度 |
| `terminal.clearOnShrink` | boolean | `false` | 内容缩小时清除空行（可能闪烁） |
| `terminal.hyperlinks` | boolean 或 `"auto"` | `"auto"` | 覆盖 OSC 8 超链接支持（高级，仅限 JSON） |
| `terminal.images` | string 或 boolean | `"auto"` | 使用 `"kitty"`、`"iterm2"`、`false` 或 `"auto"` 覆盖图像协议支持（高级，仅限 JSON） |
| `terminal.trueColor` | boolean 或 `"auto"` | `"auto"` | 覆盖真彩色支持（高级，仅限 JSON） |
| `images.autoResize` | boolean | `true` | 将图像最大调整为 2000×2000。适用于 `@file` 附件、`read` 和工具返回的图像 |
| `images.blockImages` | boolean | `false` | 阻止将所有图像发送给 LLM |

### Shell

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `shellPath` | string | - | 自定义 shell 路径（例如 Windows 上的 Cygwin）；支持以 `~` 表示主目录 |
| `shellCommandPrefix` | string | - | 每条 bash 命令的前缀（例如 `"shopt -s expand_aliases"`） |
| `npmCommand` | string[] | - | 用于 npm 包查找/安装操作的命令 argv（例如 `["mise", "exec", "node@20", "--", "npm"]`） |

JSON 中的 Windows 路径必须使用正斜杠或转义的反斜杠：

```json
{
  "shellPath": "C:/Program Files/Git/bin/bash.exe"
}
```

```json
{
  "shellPath": "C:\\Program Files\\Git\\bin\\bash.exe"
}
```

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

`npmCommand` 用于所有 npm 包管理器操作，包括安装、卸载，以及 git 包内的依赖安装。用户范围的 npm 包安装到 `~/.pi/agent/npm/`；项目范围的 npm 包安装到 `.pi/npm/`。请按照进程实际启动方式填写 argv 样式条目。配置 `npmCommand` 后，git 包依赖安装使用普通 `install`，避免向包装器或替代包管理器传递 npm 特定 flag。

### 工具

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `defaultTools` | string[] | - | 初始启用的内置工具。省略时，Pi 使用标准默认值 |

`defaultTools` 选择启动时启用的内置工具。扩展和 SDK 自定义工具仍会启用。可用内置工具包括 `read`、`bash`、`powershell`、`edit`、`write`、`grep`、`find` 和 `ls`：

```json
{
  "defaultTools": ["bash", "edit", "write"]
}
```

在 Windows 上，请选择 `powershell` 代替 `bash`，也可以同时包含两者：

```json
{
  "defaultTools": ["read", "powershell", "edit", "write"]
}
```

空数组会在不启用内置工具的情况下启动，但保留扩展和 SDK 自定义工具。`--tools` 以严格 allowlist 替换该行为，`--no-tools` 禁用所有工具，`--no-builtin-tools` 禁用内置默认工具。`--exclude-tools` 对最终列表进行筛选。项目 `defaultTools` 数组会替换全局数组。

### 会话

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `sessionDir` | string | - | 存储会话文件的目录。接受绝对路径、相对路径和 `~`。 |

```json
{ "sessionDir": ".pi/sessions" }
```

多个来源指定会话目录时，优先级依次为 `--session-dir`、`PI_CODING_AGENT_SESSION_DIR`、settings.json 中的 `sessionDir`。

### 模型循环切换

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `enabledModels` | string[] | - | 用于 Ctrl+P 循环切换的模型模式（格式与 `--models` CLI flag 相同） |

```json
{
  "enabledModels": ["claude-*", "gpt-4o", "gemini-2*"]
}
```

### Markdown

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `markdown.codeBlockIndent` | string | `"  "` | 代码块缩进 |
| `markdown.mermaid` | string | `"streaming"` | Mermaid 渲染模式：`"off"`、`"final"` 或 `"streaming"` |

### 资源

这些设置定义从何处加载扩展、技能、提示和主题。

`~/.pi/agent/settings.json` 中的路径相对于 `~/.pi/agent` 解析；`.pi/settings.json` 中的路径相对于 `.pi` 解析。支持绝对路径和 `~`。

| 设置 | 类型 | 默认值 | 说明 |
|---------|------|---------|-------------|
| `packages` | array | `[]` | 用于加载资源的 npm/git 包 |
| `extensions` | string[] | `[]` | 本地扩展文件路径或目录 |
| `skills` | string[] | `[]` | 本地技能文件路径或目录 |
| `prompts` | string[] | `[]` | 本地提示模板路径或目录 |
| `themes` | string[] | `[]` | 本地主题文件路径或目录 |
| `enableSkillCommands` | boolean | `true` | 将技能注册为 `/skill:name` 命令 |

数组支持 glob 模式和排除项。使用 `!pattern` 排除，使用 `+path` 强制包含精确路径，使用 `-path` 强制排除精确路径。

#### packages

字符串形式会加载包中的所有资源：

```json
{
  "packages": ["pi-skills", "@org/my-extension"]
}
```

对象形式可筛选要加载的资源：

```json
{
  "packages": [
    {
      "source": "pi-skills",
      "skills": ["brave-search", "transcribe"],
      "extensions": []
    }
  ]
}
```

包管理详情请参阅 [packages.md](packages.zh-CN.md)。

## 示例

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4-20250514",
  "defaultThinkingLevel": "medium",
  "modelThinkingLevels": {
    "anthropic/claude-sonnet-4-20250514": "high"
  },
  "theme": "dark",
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  },
  "retry": {
    "enabled": true,
    "maxRetries": 3
  },
  "enabledModels": ["claude-*", "gpt-4o"],
  "warnings": {
    "anthropicExtraUsage": true
  },
  "packages": ["pi-skills"]
}
```

## 项目覆盖设置

项目设置（`.pi/settings.json`）会覆盖全局设置。嵌套对象会合并：

```json
// ~/.pi/agent/settings.json（全局）
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 16384 }
}

// .pi/settings.json（项目）
{
  "compaction": { "reserveTokens": 8192 }
}

// 结果
{
  "theme": "dark",
  "compaction": { "enabled": true, "reserveTokens": 8192 }
}
```
