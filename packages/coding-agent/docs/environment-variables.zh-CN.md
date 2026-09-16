# 环境变量

Pi 通过三种方式使用环境变量：

- `PI_OFFLINE` 等变量用于配置 Pi 进程。
- Pi 会设置进程标记，使子进程能够识别 Pi 是启动它们的 agent。
- 由 LLM 可调用的 shell 工具运行的命令会收到描述当前会话的 `PI_*` 变量。

提供商 API key 变量单独记录在[提供商](providers.zh-CN.md#环境变量或身份验证文件)文档中。

## 进程标记

CLI 和 RPC 入口点会设置两个进程标记：

- `AI_AGENT=pi` 是一个通用标记，让工具能够识别 Pi 是启动该进程的 agent。
- `PI_CODING_AGENT=true` 是 Pi 专用标记，让子进程能够检测到自己正在 Pi 内部运行。

子进程会继承这两个标记。它们并非会话专用；通过 SDK 嵌入 Pi 时也不会自动设置。

## Shell 工具的会话环境

由 `bash` 和 `powershell` 工具运行的命令会收到当前 Pi 会话状态：

| 变量 | 说明 |
|----------|-------------|
| `PI_SESSION_ID` | 当前会话 ID |
| `PI_SESSION_FILE` | 当前会话 JSONL 文件的绝对路径；临时会话中不设置 |
| `PI_PROVIDER` | 当前选择的模型提供商 |
| `PI_MODEL` | 当前选择的模型 ID |
| `PI_REASONING_LEVEL` | 当前有效的推理级别：`off`、`minimal`、`low`、`medium`、`high`、`xhigh` 或 `max` |

每条命令启动时都会解析这些值。因此，切换模型或更改推理级别后，无需重启 Pi，下一条 shell 命令就会收到更新后的值。`PI_PROVIDER` 和 `PI_MODEL` 标识所选的 Pi 模型，而不是路由器可能在内部选择的其他上游模型。

当被问及当前运行的模型或提供商时，请检查这些变量，而不要从系统提示中推断：

```bash
printf '%s/%s\n' "$PI_PROVIDER" "$PI_MODEL"
printf 'reasoning=%s session=%s\n' "$PI_REASONING_LEVEL" "$PI_SESSION_ID"
```

持久会话的会话文件可以直接检查：

```bash
if [ -n "$PI_SESSION_FILE" ]; then
  tail -n 1 "$PI_SESSION_FILE"
fi
```

这些变量会注入 LLM 可调用的 `bash` 和 `powershell` 工具，但不会注入用户输入的 `!` 或 `!!` 命令。

### 自定义 Shell 工具

使用 `createBashTool()` 或 `createPowerShellTool()` 创建的工具在注册到 Pi 时，默认会公开会话环境。注入发生在 `spawnHook` 之前，因此 hook 可以通过 `ctx.env` 接收这些变量：

```typescript
const bashTool = createBashTool(cwd, {
  spawnHook: (ctx) => ({
    ...ctx,
    env: { ...ctx.env, CI: "1" },
  }),
});
```

可以独立于 spawn hook 禁用会话元数据：

```typescript
const powershellTool = createPowerShellTool(cwd, {
  exposeSessionEnvironment: false,
  spawnHook: (ctx) => ctx,
});
```

禁用后，Pi 会移除这些变量中继承的值，以免嵌套 Pi 进程公开过时的父会话元数据。

## Pi 进程配置

以下变量由 Pi 自身读取：

| 变量 | 说明 |
|----------|-------------|
| `PI_CODING_AGENT_DIR` | 覆盖配置目录；默认为 `~/.pi/agent` |
| `PI_CODING_AGENT_SESSION_DIR` | 覆盖会话存储位置；会被 `--session-dir` 覆盖 |
| `PI_PACKAGE_DIR` | 覆盖包目录，适用于 Nix/Guix 存储路径 |
| `PI_OFFLINE` | 禁用启动时的网络操作，包括更新检查、包更新以及安装/更新遥测 |
| `PI_SKIP_VERSION_CHECK` | 禁用对 `pi.dev` 最新版本的请求 |
| `PI_TELEMETRY` | 覆盖安装/更新遥测和提供商归因 header：`1`/`true`/`yes` 或 `0`/`false`/`no` |
| `PI_CACHE_RETENTION` | 设为 `long`，可在受支持的提供商上延长提示缓存时间 |
| `PI_SHARE_VIEWER_URL` | 覆盖 `/share` 使用的基础 URL |
| `PI_HARDWARE_CURSOR` | 设为 `1` 以显示硬件光标；参阅[终端配置](terminal-setup.zh-CN.md) |
| `PI_HYPERLINKS` | 使用 `1`、`0` 或 `auto` 覆盖 OSC 8 超链接检测 |
| `PI_IMAGE_PROTOCOL` | 使用 `kitty`、`iterm2`、`none` 或 `auto` 覆盖内联图像检测 |
| `PI_TRUE_COLOR` | 使用 `1`、`0` 或 `auto` 覆盖真彩色检测 |
| `PI_TUI_ESC_TIMEOUT` | 收到单独的 ESC 后，将其判定为 Escape 前的等待时间（毫秒）；SSH 下默认为 `100`，其他情况默认为 `10`。如果 Alt 组合键被误判为 Escape，请增大该值 |
| `VISUAL`、`EDITOR` | 未设置 `externalEditor` 时使用的外部编辑器回退项 |
| `HTTP_PROXY`、`HTTPS_PROXY` | 代理出站 HTTP 请求 |

`ANTHROPIC_API_KEY`、`OPENAI_API_KEY` 等提供商凭据和云提供商配置列在[提供商](providers.zh-CN.md#环境变量或身份验证文件)文档中。

`PI_SERVER_DIR` 和 `PI_SERVER_ID` 仅适用于只在源码中提供的[实验性远程执行框架](development.zh-CN.md#实验性远程执行框架)，不适用于分发构建。
