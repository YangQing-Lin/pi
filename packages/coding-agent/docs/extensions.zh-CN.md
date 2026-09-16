> pi 可以创建扩展。你可以让它针对自己的用例构建一个扩展。

# 扩展

扩展是用于扩展 pi 行为的 TypeScript 模块。它们可以订阅生命周期事件、注册供 LLM 调用的自定义工具、添加命令等。

> **支持 `/reload` 的放置位置：** 将扩展放入 `~/.pi/agent/extensions/`（全局）或 `.pi/extensions/`（项目本地），以便自动发现。`pi -e ./path.ts` 仅用于快速测试。自动发现位置中的扩展可使用 `/reload` 热重载。

**关键能力：**
- **自定义工具** - 通过 `pi.registerTool()` 注册供 LLM 调用的工具
- **事件拦截** - 阻止或修改工具调用、注入上下文、自定义压缩
- **用户交互** - 通过 `ctx.ui` 提示用户（选择、确认、输入、通知）
- **自定义 UI 组件** - 通过 `ctx.ui.custom()` 创建支持键盘输入的完整 TUI 组件，以实现复杂交互
- **自定义命令** - 通过 `pi.registerCommand()` 注册 `/mycommand` 之类的命令
- **会话持久化** - 通过 `pi.appendEntry()` 存储重启后仍保留的状态
- **自定义渲染** - 控制工具调用、工具结果和消息在 TUI 中的显示方式

**用例示例：**
- 权限门禁（执行 `rm -rf`、`sudo` 等命令前请求确认）
- Git 检查点（每轮暂存一次，切换分支时恢复）
- 路径保护（阻止写入 `.env`、`node_modules/`）
- 自定义压缩（按你的方式总结对话）
- 对话摘要（参见 `summarize.ts` 示例）
- 交互式工具（问题、向导、自定义对话框）
- 有状态工具（待办事项列表、连接池）
- 外部集成（文件监视器、Webhook、CI 触发器）
- 等待期间运行的游戏（参见 `snake.ts` 示例）

请参阅 [examples/extensions/](../examples/extensions/) 了解有效的实现。

## 目录

- [快速入门](#快速入门)
- [扩展位置](#扩展位置)
- [可用导入](#可用导入)
- [编写扩展](#编写扩展)
  - [扩展样式](#扩展样式)
- [事件](#事件)
  - [生命周期概述](#生命周期概述)
  - [资源事件](#资源事件)
  - [会话事件](#会话事件)
  - [代理事件](#代理事件)
  - [模型事件](#模型事件)
  - [工具事件](#工具事件)
- [ExtensionContext](#extensioncontext)
- [ExtensionCommandContext](#extensioncommandcontext)
- [ExtensionAPI 方法](#extensionapi-方法)
- [状态管理](#状态管理)
- [自定义工具](#自定义工具)
  - [动态工具加载](#动态工具加载)
- [自定义 UI](#自定义-ui)
- [错误处理](#错误处理)
- [模式行为](#模式行为)
- [示例参考](#示例参考)

## 快速入门

创建`~/.pi/agent/extensions/my-extension.ts`：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  // React to events
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });

  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf")) {
      const ok = await ctx.ui.confirm("Dangerous!", "Allow rm -rf?");
      if (!ok) return { block: true, reason: "Blocked by user" };
    }
  });

  // Register a custom tool
  pi.registerTool({
    name: "greet",
    label: "Greet",
    description: "Greet someone by name",
    parameters: Type.Object({
      name: Type.String({ description: "Name to greet" }),
    }),
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Hello, ${params.name}!` }],
        details: {},
      };
    },
  });

  // Register a command
  pi.registerCommand("hello", {
    description: "Say hello",
    handler: async (args, ctx) => {
      ctx.ui.notify(`Hello ${args || "world"}!`, "info");
    },
  });
}
```

使用 `--extension`（或 `-e`）标志测试：

```bash
pi -e ./my-extension.ts
```

## 扩展位置

> **安全：** 扩展以你的完整系统权限运行，并且可以执行任意代码。请仅安装来自可信来源的扩展。

扩展是从受信任的位置自动发现的。项目本地`.pi/extensions`条目仅在项目受信任后加载。

|地点 |范围 |
|----------|-------|
| `~/.pi/agent/extensions/*.ts` |全球（所有项目）|
| `~/.pi/agent/extensions/*/index.ts` |全局（子目录）|
| `.pi/extensions/*.ts` |项目本地 |
| `.pi/extensions/*/index.ts` |项目本地（子目录）|

通过 `settings.json` 的其他路径：

```json
{
  "packages": [
    "npm:@foo/bar@1.0.0",
    "git:github.com/user/repo@v1"
  ],
  "extensions": [
    "/path/to/local/extension.ts",
    "/path/to/local/extension/dir"
  ]
}
```

要通过 npm 或 git 将扩展共享为 pi 包，请参阅 [packages.md](packages.zh-CN.md)。

## 可用导入

| 包 | 用途 |
|---------|---------|
| `@earendil-works/pi-coding-agent` |扩展类型（`ExtensionAPI`、`ExtensionContext`、事件）|
| `typebox` | 工具参数的 Schema 定义 |
| `@earendil-works/pi-ai` | AI 实用程序（`StringEnum` 适用于 Google 兼容枚举）|
| `@earendil-works/pi-tui` | TUI 用于自定义渲染的组件 |

npm 依赖项也可以工作。在扩展旁边（或父目录中）添加 `package.json`，运行 `npm install`，然后从 `node_modules/` 导入将自动解析。

对于使用 `pi install`（npm 或 git）安装的分布式 pi 包，运行时依赖必须放在 `dependencies` 中。包安装默认采用生产安装（`npm install --omit=dev`），因此 `devDependencies` 在运行时不可用；配置 `npmCommand` 时，为兼容包装脚本，git 包使用普通的 `install`。

Node.js 内置模块（`node:fs`、`node:path` 等）也可使用。

## 编写扩展

扩展导出一个接收 `ExtensionAPI` 的默认工厂函数。工厂可以是同步的或异步的：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  // Subscribe to events
  pi.on("event_name", async (event, ctx) => {
    // ctx.ui for user interaction
    const ok = await ctx.ui.confirm("Title", "Are you sure?");
    ctx.ui.notify("Done!", "info");
    ctx.ui.setStatus("my-ext", "Processing...");  // Footer status
    ctx.ui.setWidget("my-ext", ["Line 1", "Line 2"]);  // Widget above editor (default)
  });

  // Register tools, commands, shortcuts, flags
  pi.registerTool({ ... });
  pi.registerCommand("name", { ... });
  pi.registerShortcut("ctrl+x", { ... });
  pi.registerFlag("my-flag", { ... });
}
```

扩展通过 [j​​iti](https://github.com/unjs/jiti) 加载，因此TypeScript无需编译即可工作。

如果工厂返回`Promise`，pi 会在继续启动之前等待它。这意味着异步初始化在`session_start`之前、`resources_discover`之前以及通过`pi.registerProvider()`排队的提供者注册被刷新之前完成。

### 异步工厂函数

使用异步工厂进行一次性启动工作，例如获取远程配置或动态发现可用模型。

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

此模式使获取的模型在正常启动期间可用并达到`pi --list-models`。

### 长期资源和关闭

扩展工厂可能在从不启动会话的调用中运行。不要从工厂启动后台资源，例如进程、套接字、文件观察程序或计时器。

将后台资源的启动推迟到 `session_start`，或推迟到需要该资源的命令、工具或事件中。注册幂等的 `session_shutdown` 处理器，以关闭你启动的所有会话级资源。

### 扩展样式

**单个文件** - 最简单，适用于小型扩展：

```
~/.pi/agent/extensions/
└── my-extension.ts
```

**带有 index.ts** 的目录 - 用于多文件扩展名：

```
~/.pi/agent/extensions/
└── my-extension/
    ├── index.ts        # Entry point (exports default function)
    ├── tools.ts        # Helper module
    └── utils.ts        # Helper module
```

**具有依赖项的包** - 对于需要 npm 包的扩展：

```
~/.pi/agent/extensions/
└── my-extension/
    ├── package.json    # Declares dependencies and entry points
    ├── package-lock.json
    ├── node_modules/   # After npm install
    └── src/
        └── index.ts
```

```json
// package.json
{
  "name": "my-extension",
  "dependencies": {
    "zod": "^3.0.0",
    "chalk": "^5.0.0"
  },
  "pi": {
    "extensions": ["./src/index.ts"]
  }
}
```

在扩展目录中运行`npm install`，然后从`node_modules/`自动导入。

## 事件

### 生命周期概述

```
pi starts
  │
  ├─► project_trust (user/global and CLI extensions only, before project resources load)
  ├─► session_start { reason: "startup" }
  └─► resources_discover { reason: "startup" }
      │
      ▼
user sends prompt ─────────────────────────────────────────┐
  │                                                        │
  ├─► (extension commands checked first, bypass if found)  │
  ├─► input (can intercept, transform, or handle)          │
  ├─► (skill/template expansion if not handled)            │
  ├─► before_agent_start (can inject message, modify system prompt)
  ├─► agent_start                                          │
  ├─► message_start / message_update / message_end         │
  │                                                        │
  │   ┌─── turn (repeats while LLM calls tools) ───┐       │
  │   │                                            │       │
  │   ├─► turn_start                               │       │
  │   ├─► context (can modify messages)            │       │
  │   ├─► before_provider_headers (can mutate headers)     |
  │   ├─► before_provider_request (can inspect or replace payload)
  │   ├─► after_provider_response (status + headers, before stream consume)
  │   │                                            │       │
  │   │   LLM responds, may call tools:            │       │
  │   │     ├─► tool_execution_start               │       │
  │   │     ├─► tool_call (can block)              │       │
  │   │     ├─► tool_execution_update              │       │
  │   │     ├─► tool_result (can modify)           │       │
  │   │     └─► tool_execution_end                 │       │
  │   │                                            │       │
  │   └─► turn_end                                 │       │
  │                                                        │
  ├─► agent_end                                            │
  └─► agent_settled (no retry/compaction/follow-up left)   │
                                                           │
user sends another prompt ◄────────────────────────────────┘

/new (new session) or /resume (switch session)
  ├─► session_before_switch (can cancel)
  ├─► session_shutdown
  ├─► session_start { reason: "new" | "resume", previousSessionFile? }
  └─► resources_discover { reason: "startup" }

/fork or /clone
  ├─► session_before_fork (can cancel)
  ├─► session_shutdown
  ├─► session_start { reason: "fork", previousSessionFile }
  └─► resources_discover { reason: "startup" }

/name or pi.setSessionName()
  └─► session_info_changed

/compact or auto-compaction
  ├─► session_before_compact (can cancel or customize)
  ├─► session_compact (success)
  └─► session_compact_failed (failure or abort)

/tree navigation
  ├─► session_before_tree (can cancel or customize)
  └─► session_tree

/model or Ctrl+P (model selection/cycling)
  ├─► thinking_level_select (if model change changes/clamps thinking level)
  └─► model_select

thinking level changes (settings, keybinding, pi.setThinkingLevel())
  └─► thinking_level_select

exit (Ctrl+C, Ctrl+D, SIGHUP, SIGTERM)
  └─► session_shutdown
```

### 启动事件

#### project_trust

在 pi 决定是否信任具有动态配置的项目（`.pi`或`.agents/skills`）之前触发。它在启动期间以及当会话替换（例如`/resume`）进入当前进程中其信任尚未解决的cwd时运行。仅用户/全局扩展和CLI`-e`扩展参与；直到信任解决后才会加载项目本地扩展。

```typescript
pi.on("project_trust", async (event, ctx) => {
  // event.cwd - current working directory
  // ctx has a limited trust context: cwd, mode, hasUI, and select/confirm/input/notify UI helpers
  if (await ctx.ui.confirm("Trust project?", event.cwd)) {
    return { trusted: "yes", remember: true };
  }
  return { trusted: "undecided" };
});
```

`project_trust`处理器必须返回`{ trusted: "yes" | "no" | "undecided" }`。返回 `"yes"` 或 `"no"` 的用户/全局或 CLI 扩展拥有决策权；第一个是/否决定获胜并抑制内置信任提示。使用`remember: true`来坚持是/否决定；否则它仅适用于当前进程。返回`"undecided"`，让后续处理器或内置信任流程决定。在提示之前检查`ctx.hasUI`。如果没有处理器返回是/否，则继续正常的信任解析：首先应用保存的`trust.json`决策，然后`defaultProjectTrust`控制pi默认询问、信任还是拒绝。

### 资源事件

#### resources_discover

在`session_start`之后触发，因此扩展可以贡献额外的技能、提示和主题路径。
启动路径使用`reason: "startup"`。重新加载使用`reason: "reload"`。

```typescript
pi.on("resources_discover", async (event, _ctx) => {
  // event.cwd - current working directory
  // event.reason - "startup" | "reload"
  return {
    skillPaths: ["/path/to/skills"],
    promptPaths: ["/path/to/prompts"],
    themePaths: ["/path/to/themes"],
  };
});
```

### 会话事件

有关会话存储内部结构和SessionManagerAPI，请参阅[会话格式](session-format.zh-CN.md)。

#### session_start

当会话启动、加载或重新加载时触发。

```typescript
pi.on("session_start", async (event, ctx) => {
  // event.reason - "startup" | "reload" | "new" | "resume" | "fork"
  // event.previousSessionFile - present for "new", "resume", and "fork"
  ctx.ui.notify(`Session: ${ctx.sessionManager.getSessionFile() ?? "ephemeral"}`, "info");
});
```

#### session_info_changed

当通过`/name`、RPC或`pi.setSessionName()`设置当前会话显示名称时触发。

```typescript
pi.on("session_info_changed", async (event, ctx) => {
  // event.name - current normalized name, or undefined if cleared
  ctx.ui.notify(`Session renamed: ${event.name ?? "(none)"}`, "info");
});
```

#### session_before_switch

在开始新会话 (`/new`) 或切换会话 (`/resume`) 之前触发。

```typescript
pi.on("session_before_switch", async (event, ctx) => {
  // event.reason - "new" or "resume"
  // event.targetSessionFile - session we're switching to (only for "resume")

  if (event.reason === "new") {
    const ok = await ctx.ui.confirm("Clear?", "Delete all messages?");
    if (!ok) return { cancel: true };
  }
});
```

成功切换或新会话操作后，pi 为旧扩展实例发出 `session_shutdown`，为新会话重新加载并重新绑定扩展，然后发出 `session_start` 以及 `reason: "new" | "resume"` 和 `previousSessionFile`。
在`session_shutdown`中进行清理工作，然后在`session_start`中重新建立内存中的状态。

#### session_before_fork

通过 `/fork` 分叉或通过 `/clone` 克隆时触发。

```typescript
pi.on("session_before_fork", async (event, ctx) => {
  // event.entryId - ID of the selected entry
  // event.position - "before" for /fork, "at" for /clone
  return { cancel: true }; // Cancel fork/clone
  // OR
  return { skipConversationRestore: true }; // Reserved for future conversation restore control
});
```

成功分叉或克隆后，pi 会为旧扩展实例发出 `session_shutdown`，为新会话重新加载并重新绑定扩展，然后发出 `session_start` 以及 `reason: "fork"` 和 `previousSessionFile`。
在`session_shutdown`中进行清理工作，然后在`session_start`中重新建立内存中的状态。

#### session_before_compact / session_compact / session_compact_failed

压缩时触发。详情请参阅[压缩](compaction.zh-CN.md)。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, reason, willRetry, signal } = event;

  // reason - "manual" (/compact), "threshold", or "overflow"
  // willRetry - whether the aborted turn is retried after compaction (overflow recovery)

  // Cancel:
  return { cancel: true };

  // Custom summary:
  return {
    compaction: {
      summary: "...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      // usage: summaryResponse.usage, // Optional; included in session totals
    }
  };
});

pi.on("session_compact", async (event, ctx) => {
  // event.compactionEntry - the saved compaction
  // event.fromExtension - whether extension provided it
  // event.reason - "manual" (/compact), "threshold", or "overflow"
  // event.willRetry - whether the aborted turn is retried after compaction (overflow recovery)
});

pi.on("session_compact_failed", async (event, ctx) => {
  // event.reason - "manual" (/compact), "threshold", or "overflow"
  // event.errorMessage - present for non-abort failures
  // event.aborted - true for cancelled/aborted compactions
  // event.willRetry - whether the aborted turn would have retried after compaction
  // event.fromExtension - whether extension-provided compaction content was being used
});
```

#### session_before_tree / session_tree

在`/tree`导航上触发。有关树导航概念，请参阅[会话](sessions.zh-CN.md)。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;
  return { cancel: true };
  // OR provide custom summary:
  return {
    summary: {
      summary: "...",
      // usage: summaryResponse.usage, // Optional; included in session totals
      details: {},
    },
  };
});

pi.on("session_tree", async (event, ctx) => {
  // event.newLeafId, oldLeafId, summaryEntry, fromExtension
});
```

#### session_shutdown

在启动的会话运行时被拆除之前触发。使用它来清理从 `session_start` 或其他会话范围的挂钩打开的资源。

```typescript
pi.on("session_shutdown", async (event, ctx) => {
  // event.reason - "quit" | "reload" | "new" | "resume" | "fork"
  // event.targetSessionFile - destination session for session replacement flows
  // Cleanup, save state, etc.
});
```

### 代理事件

#### before_agent_start

在用户提交提示后、代理循环之前触发。可以注入消息和/或修改系统提示。

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  // event.prompt - user's prompt text
  // event.images - attached images (if any)
  // event.systemPrompt - current chained system prompt for this handler
  //   (includes changes from earlier before_agent_start handlers)
  // event.systemPromptOptions - structured options used to build the system prompt
  //   .customPrompt - any custom system prompt (from --system-prompt, SYSTEM.md, or custom templates)
  //   .selectedTools - tools currently active in the prompt
  //   .toolSnippets - one-line descriptions for each tool
  //   .promptGuidelines - custom guideline bullets
  //   .appendSystemPrompt - text from --append-system-prompt flags
  //   .cwd - working directory
  //   .contextFiles - AGENTS.md files and other loaded context files
  //   .skills - loaded skills

  return {
    // Inject a persistent message (stored in session, sent to LLM)
    message: {
      customType: "my-extension",
      content: "Additional context for the LLM",
      display: true,
    },
    // Replace the system prompt for this turn (chained across extensions)
    systemPrompt: event.systemPrompt + "\n\nExtra instructions for this turn...",
  };
});
```

`systemPromptOptions` 字段让扩展能够访问 pi 构建系统提示时使用的同一组结构化数据。这样无需重新发现资源或重新解析标志，就能检查 pi 已加载的自定义提示、准则、工具摘要、上下文文件和技能。扩展需要在尊重用户配置的前提下深入调整系统提示时，可使用此字段。

内部`before_agent_start`、`event.systemPrompt`和`ctx.getSystemPrompt()`都反映了当前处理器的链接系统提示符。稍后`before_agent_start`处理器仍然可以再次修改它。

#### agent_start / agent_end / agent_settled

底层代理运行开始时会触发 `agent_start`，结束时会触发 `agent_end`；但随后 pi 仍可能自动重试、自动压缩后重试，或继续处理排队的后续消息。状态集成若需确认 pi 不会再自动运行，应使用 `agent_settled`。

```typescript
pi.on("agent_start", async (_event, ctx) => {});

pi.on("agent_end", async (event, ctx) => {
  // event.messages - messages from this low-level run
});

pi.on("agent_settled", async (_event, ctx) => {
  // ctx.isIdle() is true here unless another extension started a new run.
});
```

#### ui_prompt_start / ui_prompt_end

仅通知生命周期事件，用于阻止面向用户的扩展 UI 提示。它们在 `ctx.ui.select()`、`ctx.ui.confirm()`、`ctx.ui.input()`、`ctx.ui.editor()` 和 `ctx.ui.custom()` 周围触发，因此主机/状态集成可以报告“正在等待用户”而不仅仅是“正在运行”。

嵌套或重叠的提示被合并到一个外部等待区间中。处理器会尽力调用，并且在显示或关闭提示之前不会等待。

```typescript
pi.on("ui_prompt_start", async (event, ctx) => {
  // event.reason === "ui_prompt"
  // event.kind: "select" | "confirm" | "input" | "editor" | "custom"
  // event.title: prompt title when available
});

pi.on("ui_prompt_end", async (event, ctx) => {
  // Pi is no longer waiting on that UI prompt span.
});
```

#### turn_start / turn_end

每回合触发（一个 LLM 响应 + 工具调用）。

```typescript
pi.on("turn_start", async (event, ctx) => {
  // event.turnIndex, event.timestamp
});

pi.on("turn_end", async (event, ctx) => {
  // event.turnIndex, event.message, event.toolResults
});
```

#### message_start / message_update / message_end

因消息生命周期更新而触发。

- `message_start`和`message_end`触发用户、助手和toolResult消息。
- `message_update` 在助手消息流式更新时触发。
- `message_end` 处理器可以返回 `{ message }` 来替换最终确定的消息。替换者必须保持相同的`role`。

```typescript
pi.on("message_start", async (event, ctx) => {
  // event.message
});

pi.on("message_update", async (event, ctx) => {
  // event.message
  // event.assistantMessageEvent (token-by-token stream event)
});

pi.on("message_end", async (event, ctx) => {
  if (event.message.role !== "assistant") return;

  return {
    message: {
      ...event.message,
      usage: {
        ...event.message.usage,
        cost: {
          ...event.message.usage.cost,
          total: 0.123,
        },
      },
    },
  };
});
```

#### tool_execution_start / tool_execution_update / tool_execution_end

因工具执行生命周期更新而触发。

在并行工具模式下：
- `tool_execution_start` 在预检阶段按照辅助源顺序发出
- `tool_execution_update` 事件可能在多个工具之间交错
- `tool_execution_end` 在每个工具完成后按工具完成顺序发出
- final `toolResult` 消息事件仍会按助手源顺序稍后发出

```typescript
pi.on("tool_execution_start", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args
});

pi.on("tool_execution_update", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.args, event.partialResult
});

pi.on("tool_execution_end", async (event, ctx) => {
  // event.toolCallId, event.toolName, event.result, event.isError
});
```

#### context

在每次 LLM 调用之前触发。非破坏性地修改消息。消息类型请参见[会话格式](session-format.zh-CN.md)。

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages - deep copy, safe to modify
  const filtered = event.messages.filter(m => !shouldPrune(m));
  return { messages: filtered };
});
```

#### before_provider_headers

在组装传出的 HTTP 标头后触发。使用它来添加、覆盖或删除请求标头。

处理器就地变异`event.headers`。将键设置为字符串以添加或覆盖它，或设置为`null`以删除它。

```typescript
pi.on("before_provider_headers", (event, ctx) => {
  // Add or override — e.g. a session id for gateway tracing/attribution
  event.headers["x-session-id"] = ctx.sessionManager.getSessionId();

  // Drop a tracking header pi adds for this call
  event.headers["X-OpenRouter-Title"] = null;
});
```

每个提供者请求运行一次；重试重用相同的标头而不是重新触发钩子。

#### before_provider_request

在构建特定于提供者的负载之后、发送请求之前触发。处理器按扩展加载顺序运行。返回 `undefined` 保持有效负载不变。返回任何其他值都会替换后续处理器和实际请求的有效负载。

该钩子可以重写提供者级别的系统指令或完全删除它们。这些有效负载级别的更改不会通过 `ctx.getSystemPrompt()` 反映出来，它报告 Pi 的系统提示字符串而不是最终的序列化提供者有效负载。

```typescript
pi.on("before_provider_request", (event, ctx) => {
  console.log(JSON.stringify(event.payload, null, 2));

  // Optional: replace payload
  // return { ...event.payload, temperature: 0 };
});
```

这主要用于调试提供者序列化和缓存行为。

#### after_provider_response

在收到 HTTP 响应之后且在其流主体被消耗之前触发。处理器按扩展加载顺序运行。

```typescript
pi.on("after_provider_response", (event, ctx) => {
  // event.status - HTTP status code
  // event.headers - normalized response headers
  if (event.status === 429) {
    console.log("rate limited", event.headers["retry-after"]);
  }
});
```

响应头是否可用取决于提供者和传输方式。对 HTTP 响应进行了抽象的提供者可能不会公开响应头。

### 模型事件

#### model_select

当模型通过 `/model` 命令、模型循环 (`Ctrl+P`) 或会话恢复更改时触发。

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model - newly selected model
  // event.previousModel - previous model (undefined if first selection)
  // event.source - "set" | "cycle" | "restore"

  const prev = event.previousModel
    ? `${event.previousModel.provider}/${event.previousModel.id}`
    : "none";
  const next = `${event.model.provider}/${event.model.id}`;

  ctx.ui.notify(`Model changed (${event.source}): ${prev} -> ${next}`, "info");
});
```

使用它来更新 UI 元素（状态栏、页脚）或在活动模型更改时执行特定于模型的初始化。

#### thinking_level_select

思考级别发生变化时触发。此事件仅用于通知；处理器的返回值会被忽略。

```typescript
pi.on("thinking_level_select", async (event, ctx) => {
  // event.level - newly selected thinking level
  // event.previousLevel - previous thinking level

  ctx.ui.setStatus("thinking", `thinking: ${event.level}`);
});
```

当`pi.setThinkingLevel()`、模型更改或内置思维级别控件改变活跃思维级别时，使用此更新扩展UI。

### 工具事件

#### tool_call

在 `tool_execution_start` 之后、工具执行之前触发。 **可以阻止。** 使用 `isToolCallEventType` 缩小范围并获取键入的输入。

在 `tool_call` 运行之前，pi 等待先前发出的 Agent 事件以完成对 `AgentSession` 的排空。这意味着`ctx.sessionManager`通过当前的辅助工具调用消息是最新的。

在默认的并行工具执行模式下，来自同一助手消息的同级工具调用将按顺序进行预检，然后并发执行。 `tool_call` 不保证能够从 `ctx.sessionManager` 中的同一助手消息中看到同级工具结果。

`event.input` 是可变的。在执行之前对其进行适当修改以修补工具参数。

行为保证：
- `event.input`的突变会影响实际的工具执行
- 后来`tool_call`处理器看到早期处理器所做的突变
- 突变后不会进行重新验证
- 从`tool_call`返回值通过`{ block: true, reason?: string, terminate?: boolean }`控制阻塞
- `terminate`仅适用于被阻止的呼叫；仅当批次中的每个最终结果都终止时，代理才会提前停止

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  // event.toolName - "bash", "read", "write", "edit", etc.
  // event.toolCallId
  // event.input - tool parameters (mutable)

  // Built-in tools: no type params needed
  if (isToolCallEventType("bash", event)) {
    // event.input is { command: string; timeout?: number }
    event.input.command = `source ~/.profile\n${event.input.command}`;

    if (event.input.command.includes("rm -rf")) {
      return { block: true, reason: "Dangerous command", terminate: true };
    }
  }

  if (isToolCallEventType("read", event)) {
    // event.input is { path: string; offset?: number; limit?: number }
    console.log(`Reading: ${event.input.path}`);
  }
});
```

#### 自定义工具输入类型

自定义工具应导出自己的输入类型：

```typescript
// my-extension.ts
export type MyToolInput = Static<typeof myToolSchema>;
```

将 `isToolCallEventType` 与显式类型参数一起使用：

```typescript
import { isToolCallEventType } from "@earendil-works/pi-coding-agent";
import type { MyToolInput } from "my-extension";

pi.on("tool_call", (event) => {
  if (isToolCallEventType<"my_tool", MyToolInput>("my_tool", event)) {
    event.input.action;  // typed
  }
});
```

#### tool_result

在工具执行完成后且在发出 `tool_execution_end` 加上最终工具结果消息事件之前触发。 **可以修改结果。**

在并行工具模式下，`tool_result`和`tool_execution_end`可能会按照工具完成顺序交错，而最终的`toolResult`消息事件仍会按照辅助源顺序稍后发出。

`tool_result` 处理器链如中间件：
- 处理器按扩展加载顺序运行
- 每个处理器都会看到前一个处理器更改后的最新结果
- 处理器可以返回部分补丁（`content`、`details`、`isError`或`usage`）；省略的字段保留其当前值

使用 `ctx.signal` 进行处理器内的嵌套异步工作。这允许 Esc 取消模型调用、`fetch()`以及由扩展启动的其他中止感知操作。

```typescript
import { isBashToolResult } from "@earendil-works/pi-coding-agent";

pi.on("tool_result", async (event, ctx) => {
  // event.toolName, event.toolCallId, event.input
  // event.content, event.details, event.isError, event.usage

  if (isBashToolResult(event)) {
    // event.details is typed as BashToolDetails
  }

  const response = await fetch("https://example.com/summarize", {
    method: "POST",
    body: JSON.stringify({ content: event.content }),
    signal: ctx.signal,
  });

  // Modify result:
  return { content: [...], details: {...}, isError: false, usage: nestedModelUsage };
});
```

### 用户 Bash 事件

#### user_bash

当用户执行 `!` 或 `!!` 命令时触发。 **可以拦截。**

```typescript
import { createLocalBashOperations } from "@earendil-works/pi-coding-agent";

pi.on("user_bash", (event, ctx) => {
  // event.command - the bash command
  // event.excludeFromContext - true if !! prefix
  // event.cwd - working directory

  // Option 1: Provide custom operations (e.g., SSH)
  return { operations: remoteBashOps };

  // Option 2: Wrap pi's built-in local bash backend
  const local = createLocalBashOperations();
  return {
    operations: {
      exec(command, cwd, options) {
        return local.exec(`source ~/.profile\n${command}`, cwd, options);
      }
    }
  };

  // Option 3: Full replacement - return result directly
  return { result: { output: "...", exitCode: 0, cancelled: false, truncated: false } };
});
```

### 输入事件

#### input

收到用户输入时触发：此时已检查扩展命令，但尚未展开技能和模板。该事件看到的是原始输入文本，因此 `/skill:foo` 和 `/template` 尚未展开。

**加工订单：**
1. 首先检查扩展命令（`/cmd`）——如果找到，则运行处理器并跳过 `input` 事件
2. `input` 事件触发 - 可以拦截、转换或处理
3. 如果不处理：技能命令（`/skill:name`）扩展为技能内容
4. 如果不处理：提示模板（`/template`）扩展到模板内容
5. 代理处理开始（`before_agent_start`等）

```typescript
pi.on("input", async (event, ctx) => {
  // event.text - raw input (before skill/template expansion)
  // event.images - attached images, if any
  // event.source - "interactive" (typed), "rpc" (API), or "extension" (via sendUserMessage)
  // event.streamingBehavior - "steer" | "followUp" | undefined
  //   undefined when idle, "steer" for mid-stream interrupts,
  //   "followUp" for messages queued until the agent finishes

  // Transform: rewrite input before expansion
  if (event.text.startsWith("?quick "))
    return { action: "transform", text: `Respond briefly: ${event.text.slice(7)}` };

  // Handle: respond without LLM (extension shows its own feedback)
  if (event.text === "ping") {
    ctx.ui.notify("pong", "info");
    return { action: "handled" };
  }

  // Route by source: skip processing for extension-injected messages
  if (event.source === "extension") return { action: "continue" };

  // Intercept skill commands before expansion
  if (event.text.startsWith("/skill:")) {
    // Could transform, block, or let pass through
  }

  return { action: "continue" };  // Default: pass through to expansion
});
```

**结果：**
- `continue` - 原样传递（处理器未返回内容时的默认行为）
- `transform` - 修改文本或图像，然后继续展开
- `handled` - 完全跳过代理（以第一个返回此操作的处理器为准）

多个处理器的转换会依次串联。有关感知 `streamingBehavior` 的路由，请参阅 [input-transform.ts](../examples/extensions/input-transform.ts) 和 [input-transform-streaming.ts](../examples/extensions/input-transform-streaming.ts)。

## ExtensionContext

所有处理器都会收到 `ctx: ExtensionContext`。

### ctx.ui

用于与用户交互的 UI 方法。完整说明请参阅[自定义 UI](#自定义-ui)。

### ctx.mode

当前运行模式：`"tui"`、`"rpc"`、`"json"` 或 `"print"`。对于 `custom()`、组件工厂、终端输入和直接 TUI 渲染等仅限终端的功能，请使用 `ctx.mode === "tui"` 进行保护。

### ctx.hasUI

在 TUI 和 RPC 模式下为 `true`，在打印模式（`-p`）和 JSON 模式下为 `false`。使用它保护对话框方法（`select`、`confirm`、`input`、`editor`）以及即发即弃方法（`notify`、`setStatus`、`setWidget`、`setTitle`、`setEditorText`）；这些方法在 TUI 和 RPC 模式下均可使用。在 RPC 模式下，部分 TUI 专用方法不会执行任何操作或会返回默认值（参阅 [rpc.md](rpc.zh-CN.md#扩展-ui-协议)）。

### ctx.cwd

当前工作目录。

构建项目本地配置路径时，使用 `CONFIG_DIR_NAME` 而不是硬编码 `.pi`。重新命名的发行版可以使用不同的配置目录名称。

```typescript
import { CONFIG_DIR_NAME, type ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { join } from "node:path";

export default function (pi: ExtensionAPI) {
  pi.on("session_start", (_event, ctx) => {
    const projectConfigPath = join(ctx.cwd, CONFIG_DIR_NAME, "my-extension.json");
    // ...
  });
}
```

### ctx.isProjectTrusted()

返回当前会话上下文是否启用了项目本地信任。这包括临时信任决策和 CLI 信任覆盖，而不仅限于全局信任存储中保存的决策。

读取仅应由受信任项目采用的项目本地扩展配置前，请先调用此方法。

### ctx.sessionManager

对会话状态的只读访问。有关完整的 SessionManager API 和条目类型，请参阅[会话格式](session-format.zh-CN.md)。

对于 `tool_call`，处理器运行前，此状态会同步到当前助手消息。在并行工具执行模式下，仍不保证其中包含同一助手消息内其他工具调用的结果。

```typescript
ctx.sessionManager.getEntries()             // All entries
ctx.sessionManager.getBranch()              // Current branch
ctx.sessionManager.buildContextEntries()    // Active branch entries with compaction applied
ctx.sessionManager.getLeafId()              // Current leaf entry ID
```

### ctx.modelRegistry / ctx.model / ctx.thinkingLevel / ctx.scopedModels

访问模型、提供者和解析后的身份验证信息。`ctx.modelRegistry.getProvider(id)` 返回当前生效的 pi-ai 提供者；`getProviderAuth(id)` 无需加载模型即可解析其当前 API 密钥、请求头、基础 URL 和提供者作用域环境。`ctx.model` 是当前模型，`ctx.thinkingLevel` 是当前生效的思考级别。

`ctx.scopedModels` 是当前会话作用域内模型的只读列表，与 `/scoped-models` 命令显示的集合相同。它在会话开始时根据 `--models` CLI 标志和 `enabledModels` 设置解析：使用 minimatch 将模式与可用目录中的 `provider/modelId` 或不含提供者的 `modelId` 匹配。未配置作用域时，此列表为空，表示所有可用模型均可使用。每个条目都是 `{ model, thinkingLevel? }`；只有模式明确指定了思考级别（例如 `anthropic/*:high`）时，才会设置 `thinkingLevel`。需要构建与内置选择器一致的模型选择器时，请使用此列表，而不是通过 `ctx.modelRegistry.getAvailable()` 枚举整个目录。

#### 流式模型调用

对于 `reasoning` 等与提供者无关的选项，请使用 `ctx.modelRegistry.streamSimple(model, context, options)`；对于 API 专用选项，请使用 `stream()`。两者都会使用已配置的提供者并解析身份验证，也支持通过 `pi.registerProvider()` 注册的提供者。请使用这些方法，而不是 `pi-ai/compat` 的流式函数，因为后者无法识别扩展注册的提供者。

两者都返回 `AssistantMessageEventStream`。迭代该对象可读取响应事件，等待 `.result()` 可取得最终消息。初始化失败会产生错误事件和错误结果。

### ctx.signal

当前代理的中止信号；没有正在进行的代理轮次时为 `undefined`。

将此用于由扩展处理器启动的中止感知嵌套工作，例如：
- `fetch(..., { signal: ctx.signal })`
- 接受`signal`的模型调用
- 接受`AbortSignal`的文件或进程助手

`ctx.signal`通常在活动回合事件期间定义，例如`tool_call`、`tool_result`、`message_update`和`turn_end`。
在空闲或非轮流上下文中，例如会话事件、扩展命令和 pi 空闲时触发的快捷方式，它通常为`undefined`。

```typescript
pi.on("tool_result", async (event, ctx) => {
  const response = await fetch("https://example.com/api", {
    method: "POST",
    body: JSON.stringify(event),
    signal: ctx.signal,
  });

  const data = await response.json();
  return { details: data };
});
```

### ctx.isIdle() / ctx.abort() / ctx.hasPendingMessages()

这些是控制流程的辅助方法。当 pi 正在处理代理运行、自动重试、自动压缩重试或排队中的继续操作时，`ctx.isIdle()` 为 `false`。

### ctx.shutdown()

请求正常关闭 pi。

- **交互模式：** 延迟到代理空闲时执行（即所有排队的引导消息和后续消息均已处理完毕）。
- **RPC 模式：** 延迟到下一个空闲状态执行（完成当前命令响应并等待下一条命令时）。
- **打印模式：** 不执行任何操作。处理完所有提示后，进程会自动退出。

退出前会向所有扩展发出 `session_shutdown` 事件。此方法可在所有上下文中使用（事件处理器、工具、命令、快捷键）。

```typescript
pi.on("tool_call", (event, ctx) => {
  if (isFatal(event.input)) {
    ctx.shutdown();
  }
});
```

### ctx.getContextUsage()

返回当前模型的上下文用量。若最后一条助手消息包含用量信息，则优先使用；否则估算后续消息的 token 数。

```typescript
const usage = ctx.getContextUsage();
if (usage && usage.tokens > 100_000) {
  // ...
}
```

### ctx.compact()

触发压缩而不等待完成。使用`onComplete`和`onError`进行后续操作。

```typescript
ctx.compact({
  customInstructions: "Focus on recent changes",
  onComplete: (result) => {
    ctx.ui.notify("Compaction completed", "info");
  },
  onError: (error) => {
    ctx.ui.notify(`Compaction failed: ${error.message}`, "error");
  },
});
```

### ctx.getSystemPrompt()

返回 Pi 当前的系统提示字符串。

- 在`before_agent_start`期间，这反映了当前回合迄今为止所做的连锁系统提示更改。
- 不包括后来的`context`消息突变。
- 它不包括`before_provider_request`有效负载重写。
- 如果还有在当前扩展之后运行的扩展，它们仍可修改最终发送的内容。

```typescript
pi.on("before_agent_start", (event, ctx) => {
  const prompt = ctx.getSystemPrompt();
  console.log(`System prompt length: ${prompt.length}`);
});
```

## ExtensionCommandContext

命令处理器接收 `ExtensionCommandContext`，它在 `ExtensionContext` 的基础上增加了会话控制方法。这些方法仅在命令中可用，因为从事件处理器调用它们可能导致死锁。

### ctx.getSystemPromptOptions()

返回 Pi 当前用于构建系统提示的基本输入。

```typescript
const options = ctx.getSystemPromptOptions();
const contextPaths = options.contextFiles?.map((file) => file.path) ?? [];
```

它与 `before_agent_start` 的 `event.systemPromptOptions` 具有相同的结构和可变性：包括自定义提示、当前工具、工具摘要、提示准则、追加的系统提示文本、cwd、已加载的上下文文件和技能。它可能包含上下文文件的完整内容，因此应将其视为扩展内部的敏感数据，避免通过命令列表、日志或自动补全元数据泄露。

这会报告当前的基本提示输入。它不包括每轮`before_agent_start`链式系统提示更改、后来的`context`事件消息突变或`before_provider_request`有效负载重写。

### ctx.waitForIdle()

等待代理彻底结束运行，包括自动重试、自动压缩后的重试以及排队的继续操作：

```typescript
pi.registerCommand("my-cmd", {
  handler: async (args, ctx) => {
    await ctx.waitForIdle();
    // Agent is now idle, safe to modify session
  },
});
```

### ctx.newSession(options?)

创建一个新会话：

```typescript
const parentSession = ctx.sessionManager.getSessionFile();
const kickoff = "Continue in the replacement session";

const result = await ctx.newSession({
  parentSession,
  setup: async (sm) => {
    sm.appendMessage({
      role: "user",
      content: [{ type: "text", text: "Context from previous session..." }],
      timestamp: Date.now(),
    });
  },
  withSession: async (ctx) => {
    // Use only the replacement-session ctx here.
    await ctx.sendUserMessage(kickoff);
  },
});

if (result.cancelled) {
  // An extension cancelled the new session
}
```

选项：
- `parentSession`：要记录在新会话标头中的父会话文件
- `setup`：在`withSession`运行之前改变新会话的`SessionManager`
- `withSession`：使用新的替换会话上下文执行切换后的工作。不要使用捕获的旧 `pi` 或旧命令 `ctx`；请参阅[会话替换生命周期及注意事项](#会话替换生命周期及注意事项)。

### ctx.fork(entryId, options?)

从特定条目分叉，创建一个新的会话文件：

```typescript
const result = await ctx.fork("entry-id-123", {
  withSession: async (ctx) => {
    // Use only the replacement-session ctx here.
    ctx.ui.notify("Now in the forked session", "info");
  },
});
if (result.cancelled) {
  // An extension cancelled the fork
}

const cloneResult = await ctx.fork("entry-id-456", { position: "at" });
if (cloneResult.cancelled) {
  // An extension cancelled the clone
}
```

选项：
- `position`：`"before"`（默认）在选定的用户消息之前分叉，将该提示恢复到编辑器中
- `position`：`"at"` 复制截至所选条目的当前路径，不恢复编辑器文本
- `withSession`：使用新的替换会话上下文执行切换后的工作。不要使用捕获的旧 `pi` 或旧命令 `ctx`；请参阅[会话替换生命周期及注意事项](#会话替换生命周期及注意事项)。

### ctx.navigateTree(targetId, options?)

导航到会话树中的另一个位置。当代理响应、手动或自动压缩或其他树导航正在进行时，即使设置了 `summarize: false`，此方法也会拒绝执行。这些冲突不会改变当前分支，并会拒绝 Promise，而不是返回 `{ cancelled: true }`。请等待当前操作结束（例如在命令处理器中执行 `await ctx.waitForIdle()`），然后重试：

```typescript
const result = await ctx.navigateTree("entry-id-456", {
  summarize: true,
  customInstructions: "Focus on error handling changes",
  replaceInstructions: false, // true = replace default prompt entirely
  label: "review-checkpoint",
});
```

选项：
- `summarize`：是否生成废弃分支的摘要
- `customInstructions`：摘要器的自定义指令
- `replaceInstructions`：如果为 true，`customInstructions` 将替换默认提示而不是附加
- `label`：附加到分支摘要条目的标签（如果不汇总，则附加到目标条目）

### ctx.switchSession(sessionPath, options?)

切换到不同的会话文件：

```typescript
const result = await ctx.switchSession("/path/to/session.jsonl", {
  withSession: async (ctx) => {
    await ctx.sendUserMessage("Resume work in the replacement session");
  },
});
if (result.cancelled) {
  // An extension cancelled the switch via session_before_switch
}
```

选项：
- `withSession`：使用新的替换会话上下文执行切换后的工作。不要使用捕获的旧 `pi` 或旧命令 `ctx`；请参阅[会话替换生命周期及注意事项](#会话替换生命周期及注意事项)。

要发现可用会话，请使用静态 `SessionManager.list()` 或 `SessionManager.listAll()` 方法：

```typescript
import { SessionManager } from "@earendil-works/pi-coding-agent";

pi.registerCommand("switch", {
  description: "Switch to another session",
  handler: async (args, ctx) => {
    const sessions = await SessionManager.list(ctx.cwd);
    if (sessions.length === 0) return;
    const choice = await ctx.ui.select(
      "Pick session:",
      sessions.map(s => s.file),
    );
    if (choice) {
      await ctx.switchSession(choice, {
        withSession: async (ctx) => {
          ctx.ui.notify("Switched session", "info");
        },
      });
    }
  },
});
```

### 会话替换生命周期及注意事项

`withSession` 接收一个全新的 `ReplacedSessionContext`。该类型扩展了 `ExtensionCommandContext`，并提供绑定到替换会话的异步 `sendMessage()` 和 `sendUserMessage()` 辅助方法。

生命周期及注意事项：
- `withSession` 只会在旧会话发出 `session_shutdown`、旧运行时被销毁、替换会话重新绑定且新扩展实例已收到 `session_start` 后运行。
- 回调仍在原始闭包中执行，而不是在新的扩展实例内执行。因此，`withSession` 开始前，旧扩展实例可能已经完成关闭清理。
- 捕获的旧 `pi` 或旧命令 `ctx` 中与会话绑定的对象在替换后已失效，继续使用会抛出异常。涉及会话的工作只能使用传给 `withSession` 的 `ctx`。
- 之前提取的原始对象仍由你负责。例如，若替换前保存了 `const sm = ctx.sessionManager`，`sm` 仍指向旧的 `SessionManager`，替换后不要再次使用。
- `withSession` 中的代码必须假定：所有被 `session_shutdown` 处理器清理的状态都已不存在。只能捕获能安全跨越关闭过程的纯数据，例如字符串、ID 和序列化配置。

安全模式：

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const kickoff = "Continue from the replacement session";
    await ctx.newSession({
      withSession: async (ctx) => {
        await ctx.sendUserMessage(kickoff);
      },
    });
  },
});
```

不安全模式：

```typescript
pi.registerCommand("handoff", {
  handler: async (_args, ctx) => {
    const oldSessionManager = ctx.sessionManager;
    await ctx.newSession({
      withSession: async (_ctx) => {
        // stale old objects: do not do this
        oldSessionManager.getSessionFile();
        pi.sendUserMessage("wrong");
      },
    });
  },
});
```

### ctx.reload()

运行与`/reload`相同的重新加载流程。

```typescript
pi.registerCommand("reload-runtime", {
  description: "Reload extensions, skills, prompts, themes, and context files",
  handler: async (_args, ctx) => {
    await ctx.reload();
    return;
  },
});
```

重要行为：
- `await ctx.reload()` 为当前扩展运行时发出 `session_shutdown`
- 然后重新加载资源并发出`session_start`和`reason: "reload"`和`resources_discover`以及原因`"reload"`
- 当前运行的命令处理器仍然在旧的调用栈帧中继续
- `await ctx.reload()`之后的代码仍然从预重载版本运行
- `await ctx.reload()`之后的代码不得假设旧的内存扩展状态仍然有效
- 处理器返回后，未来的命令/事件/工具调用将使用新的扩展版本

对于可预测的行为，请将重新加载视为该处理器的终端（`await ctx.reload(); return;`）。

工具以`ExtensionContext`运行，因此它们不能直接调用`ctx.reload()`。使用命令作为重新加载入口点，然后公开一个将该命令作为后续用户消息排队的工具。

LLM可以调用来触发重新加载的示例工具：

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

export default function (pi: ExtensionAPI) {
  pi.registerCommand("reload-runtime", {
    description: "Reload extensions, skills, prompts, themes, and context files",
    handler: async (_args, ctx) => {
      await ctx.reload();
      return;
    },
  });

  pi.registerTool({
    name: "reload_runtime",
    label: "Reload Runtime",
    description: "Reload extensions, skills, prompts, themes, and context files",
    parameters: Type.Object({}),
    async execute() {
      pi.sendUserMessage("/reload-runtime", { deliverAs: "followUp" });
      return {
        content: [{ type: "text", text: "Queued /reload-runtime as a follow-up command." }],
      };
    },
  });
}
```

## ExtensionAPI 方法

### pi.on(event, handler)

订阅事件。事件类型和返回值请参阅[事件](#事件)。

### pi.registerTool(definition)

注册供 LLM 调用的自定义工具。完整说明请参阅[自定义工具](#自定义工具)。

`pi.registerTool()` 在扩展加载期间和启动后均可使用。你可以在 `session_start`、命令处理器或其他事件处理器中调用它。新工具会立即在当前会话中刷新，因此无需 `/reload` 就会出现在 `pi.getAllTools()` 中，并可供 LLM 调用。

使用`pi.setActiveTools()`在运行时启用或禁用工具（包括动态添加的工具）。

使用 `promptSnippet` 可让自定义工具在默认系统提示的 `Available tools` 部分中显示一行摘要；工具处于启用状态时，`promptGuidelines` 会把工具专用的条目追加到默认的 `Guidelines` 部分。

**重要提示：** `promptGuidelines` 的条目会直接追加到 `Guidelines` 部分，不带工具名称前缀。每条准则都必须明确指出所涉及的工具。不要写“在……时使用此工具”，因为 LLM 无法判断“此工具”指什么；应写成“在……时使用 my_tool”。

有关完整示例，请参阅 [dynamic-tools.ts](../examples/extensions/dynamic-tools.ts)。

```typescript
import { Type } from "typebox";
import { StringEnum } from "@earendil-works/pi-ai";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "What this tool does",
  promptSnippet: "Summarize or transform text according to action",
  promptGuidelines: ["Use my_tool when the user asks to summarize previously generated text."],
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),
    text: Type.Optional(Type.String()),
  }),
  prepareArguments(args) {
    // Optional compatibility shim. Runs before schema validation.
    // Return the current schema shape, for example to fold legacy fields
    // into the modern parameter object.
    return args;
  },

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // Stream progress
    onUpdate?.({ content: [{ type: "text", text: "Working..." }] });

    return {
      content: [{ type: "text", text: "Done" }],
      details: { result: "..." },
    };
  },

  // Optional: Custom rendering
  renderCall(args, theme, context) { ... },
  renderResult(result, options, theme, context) { ... },
});
```

### pi.sendMessage(message, options?)

向会话中注入自定义消息。自定义消息会进入 LLM 上下文。对于不应发送给 LLM、但需要持久显示在 TUI 中的内容，请结合使用 [`pi.appendEntry()`](#piappendentrycustomtype-data) 和 [`pi.registerEntryRenderer()`](#piregisterentryrenderercustomtype-renderer)。

```typescript
pi.sendMessage({
  customType: "my-extension",
  content: "Message text",
  display: true,
  details: { ... },
}, {
  triggerTurn: true,
  deliverAs: "steer",
});
```

**选项：**
- `deliverAs` - 交付模式：
  - `"steer"`（默认）- 流式响应期间将消息排队；当前助手轮次执行完工具调用后、下一次 LLM 调用前交付。
  - `"followUp"` - 等待代理完成，仅在代理不再有工具调用时交付。
  - `"nextTurn"` - 排队到下一条用户提示；不会中断或触发任何操作。
- `triggerTurn: true` - 如果代理空闲，立即触发 LLM 响应。仅适用于 `"steer"` 和 `"followUp"` 模式（`"nextTurn"` 会忽略此项）。

### pi.sendUserMessage(content, options?)

向代理发送用户消息。`sendMessage()` 发送的是自定义消息，而此方法发送一条真正的用户消息，效果与用户手动输入相同，并且总会触发一个轮次。

```typescript
// Simple text message
pi.sendUserMessage("What is 2+2?");

// With content array (text + images)
pi.sendUserMessage([
  { type: "text", text: "Describe this image:" },
  { type: "image", source: { type: "base64", mediaType: "image/png", data: "..." } },
]);

// During streaming - must specify delivery mode
pi.sendUserMessage("Focus on error handling", { deliverAs: "steer" });
pi.sendUserMessage("And then summarize", { deliverAs: "followUp" });

// Opt in to extension command dispatch and skill/prompt template expansion
pi.sendUserMessage("/review src/index.ts", { expandPromptTemplates: true });
```

**选项：**
- `deliverAs` - 代理正在流式响应时必填：
  - `"steer"` - 在当前助手轮完成执行其工具调用后将消息排队等待传递
  - `"followUp"` - 等待代理完成所有工具
- `expandPromptTemplates` - 分派扩展命令，并展开技能命令和提示模板。默认为 `false`。

未在流式响应时，消息会立即发送并触发新轮次。在流式响应期间未指定 `deliverAs` 会抛出错误。

有关完整示例，请参阅 [send-user-message.ts](../examples/extensions/send-user-message.ts)。

### pi.appendEntry(customType, data?)

持久化扩展数据。自定义条目不进入 LLM 上下文。在交互模式下，与 `pi.registerEntryRenderer()` 配合后，它们也可以显示在聊天记录中。

```typescript
pi.appendEntry("my-state", { count: 42 });
pi.appendEntry("status-card", { title: "Indexed files", count: 17 });

// Restore on reload
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      // Reconstruct from entry.data
    }
  }
});
```

### pi.setSessionName(name)

设置会话显示名称（显示在会话选择器中而不是第一条消息中）。

```typescript
pi.setSessionName("Refactor auth module");
```

### pi.getSessionName()

获取当前会话名称（如果已设置）。

```typescript
const name = pi.getSessionName();
if (name) {
  console.log(`Session: ${name}`);
}
```

### pi.setLabel(entryId, label)

设置或清除条目上的标签。标签是用户定义的书签和导航标记（显示在`/tree`选择器中）。

```typescript
// Set a label
pi.setLabel(entryId, "checkpoint-before-refactor");

// Clear a label
pi.setLabel(entryId, undefined);

// Read labels via sessionManager
const label = ctx.sessionManager.getLabel(entryId);
```

标签在会话中保留并在重新启动后继续存在。使用它们来标记对话树中的重要点（回合、检查点）。

### pi.registerCommand(name, options)

注册命令。

如果多个扩展注册了相同的命令名称，pi 会将它们全部保留并按加载顺序分配数字调用后缀，例如 `/review:1` 和 `/review:2`。

```typescript
pi.registerCommand("stats", {
  description: "Show session statistics",
  handler: async (args, ctx) => {
    const count = ctx.sessionManager.getEntries().length;
    ctx.ui.notify(`${count} entries`, "info");
  }
});
```

可选：为 `/command ...` 添加参数自动补全：

```typescript
import type { AutocompleteItem } from "@earendil-works/pi-tui";

pi.registerCommand("deploy", {
  description: "Deploy to an environment",
  getArgumentCompletions: (prefix: string): AutocompleteItem[] | null => {
    const envs = ["dev", "staging", "prod"];
    const items = envs.map((e) => ({ value: e, label: e }));
    const filtered = items.filter((i) => i.value.startsWith(prefix));
    return filtered.length > 0 ? filtered : null;
  },
  handler: async (args, ctx) => {
    ctx.ui.notify(`Deploying: ${args}`, "info");
  },
});
```

### pi.getCommands()

获取当前会话中可通过 `prompt` 调用的斜杠命令，包括扩展命令、提示模板和技能命令。
该列表符合 RPC `get_commands` 顺序：首先是扩展，然后是模板，最后是技能。

```typescript
const commands = pi.getCommands();
const bySource = commands.filter((command) => command.source === "extension");
const userScoped = commands.filter((command) => command.sourceInfo.scope === "user");
```

每个条目的结构如下：

```typescript
{
  name: string; // Invokable command name without the leading slash. May be suffixed like "review:1"
  description?: string;
  source: "extension" | "prompt" | "skill";
  sourceInfo: {
    path: string;
    source: string;
    scope: "user" | "project" | "temporary";
    origin: "package" | "top-level";
    baseDir?: string;
  };
}
```

使用 `sourceInfo` 作为规范出处字段。不要从命令名称或临时路径解析推断所有权。

这里不包括内置的交互式命令（如`/model`和`/settings`）。它们仅在交互中处理
模式，如果通过`prompt`发送则不会执行。

### pi.registerMessageRenderer(customType, renderer)

为指定 `customType` 的自定义消息注册 TUI 渲染器。自定义消息通过 `pi.sendMessage()` 创建，并会进入 LLM 上下文。请参阅[自定义 UI](#自定义-ui)。

### pi.registerMarkdownTransformer(transformer)

为普通用户文本、助手文本和思考块中的 Markdown 注册转换器。转换器按扩展加载顺序运行，每个转换器都会收到上一个转换器返回的 Markdown。转换链结束后，pi 使用内置渲染器渲染转换后的内容。

转换器接收 Markdown 字符串和一个上下文对象，其中：

- `messageType` — `"user"`、`"assistant"`或`"assistant-thinking"`
- `isStreaming` — `true`用于部分助手更新； `false` 用于用户、最终确定的助手和恢复的消息
- `availableWidth` — 可用于转换后的 Markdown 内容的精确终端列

返回转换后的 Markdown：

```typescript
pi.registerMarkdownTransformer((markdown, { messageType, isStreaming }) => {
  if (isStreaming || messageType === "assistant-thinking") return markdown;
  return markdown.replaceAll("-->", "→");
});
```

如果转换器抛出异常，pi 会保留当前已经生成的 Markdown，并继续运行下一个转换器。此钩子只影响显示：会话和模型上下文中的原始消息不会改变。新用户消息、助手流式更新、恢复的会话消息以及终端宽度变化都会触发它，因此转换器应保持同步且开销低。

### pi.registerEntryRenderer(customType, renderer)

为指定 `customType` 的自定义条目注册 TUI 渲染器。自定义条目通过 `pi.appendEntry()` 创建，不进入 LLM 上下文。

```typescript
import { Box, Text } from "@earendil-works/pi-tui";

pi.registerEntryRenderer("status-card", (entry, { expanded }, theme) => {
  const data = entry.data as { title: string; count: number };
  const box = new Box(1, 1, (text) => theme.bg("customMessageBg", text));
  box.addChild(new Text(`${theme.bold(data.title)}: ${data.count}`));
  if (expanded) {
    box.addChild(new Text(theme.fg("dim", JSON.stringify(data, null, 2))));
  }
  return box;
});

pi.appendEntry("status-card", { title: "Indexed files", count: 17 });
```

### pi.registerShortcut(shortcut, options)

注册键盘快捷键。有关快捷方式格式和内置按键绑定，请参阅 [keybindings.md](keybindings.zh-CN.md)。

```typescript
pi.registerShortcut("ctrl+shift+p", {
  description: "Toggle plan mode",
  handler: async (ctx) => {
    ctx.ui.notify("Toggled!");
  },
});
```

### pi.registerFlag(name, options)

注册 CLI 标志。

```typescript
pi.registerFlag("plan", {
  description: "Start in plan mode",
  type: "boolean",
  default: false,
});

// Check value
if (pi.getFlag("plan")) {
  // Plan mode enabled
}
```

### pi.exec(command, args, options?)

执行 shell 命令。

```typescript
const result = await pi.exec("git", ["status"], { signal, timeout: 5000 });
// result.stdout, result.stderr, result.code, result.killed
```

### pi.getActiveTools() / pi.getAllTools() / pi.setActiveTools(names)

管理活动工具。这适用于内置工具和动态注册工具。 `pi.getActiveTools()`返回活动工具名称为`string[]`； `pi.getAllTools()` 返回所有已配置工具的元数据。

```typescript
const active = pi.getActiveTools(); // ["read", "bash", ...]
const all = pi.getAllTools();
// all = [{
//   name: "read",
//   description: "Read file contents...",
//   parameters: ...,
//   promptGuidelines: ["Use read to examine files instead of cat or sed."],
//   sourceInfo: { path: "<builtin:read>", source: "builtin", scope: "temporary", origin: "top-level" }
// }, ...]
const builtinTools = all.filter((t) => t.sourceInfo.source === "builtin");
const extensionTools = all.filter((t) => t.sourceInfo.source !== "builtin" && t.sourceInfo.source !== "sdk");
pi.setActiveTools([...new Set([...active, "my_custom_tool"])]); // Keep current tools and enable my_custom_tool
pi.setActiveTools(["read", "bash"]); // Switch to read-only
```

`pi.getAllTools()`返回`name`、`description`、`parameters`、`promptGuidelines`和`sourceInfo`。

典型的`sourceInfo.source`值：
- `builtin` 用于内置工具
- `sdk` 对于通过`createAgentSession({ customTools })`传递的工具
- 由扩展注册的工具的扩展源元数据

### pi.setModel(model)

设置当前会话的模型。更改会记录在会话历史记录中，并在会话恢复时恢复，但不会更改新会话使用的配置的`defaultProvider`或`defaultModel`。如果没有为模型提供者配置身份验证，则返回`false`。请参阅 [models.md](models.zh-CN.md) 配置自定义模型。

```typescript
const model = ctx.modelRegistry.find("anthropic", "claude-sonnet-4-5");
if (model) {
  const success = await pi.setModel(model);
  if (!success) {
    ctx.ui.notify("No API key for this model", "error");
  }
}
```

### pi.getThinkingLevel() / pi.setThinkingLevel(level)

获取当前思考级别。该级别受模型能力限制（非推理模型始终使用 `"off"`）。更改会触发 `thinking_level_select`。

`pi.setThinkingLevel()` 更改当前会话的思考级别。更改会记录在会话历史中，并在恢复该会话时还原，但不会改变新会话使用的配置默认值。

```typescript
const current = pi.getThinkingLevel();  // "off" | "minimal" | "low" | "medium" | "high" | "xhigh" | "max"
pi.setThinkingLevel("high");
```

### pi.events

用于扩展之间通信的共享事件总线：

```typescript
pi.events.on("my:event", (data) => { ... });
pi.events.emit("my:event", { ... });
```

### pi.registerProvider(name, config)

动态注册或覆盖模型提供者。对于代理、自定义端点或团队范围的模型配置很有用。

一旦运行程序初始化，扩展工厂函数期间进行的调用就会排队并应用。此后进行的调用（例如，从用户设置流程之后的命令处理器进行的调用）立即生效，无需 `/reload`。

动态提供者可以实现 `refreshModels`。pi 会在刷新模型时调用它，通过提供者同步发布返回的列表，并传入规范的凭据、已存储目录、网络和信号上下文。扩展可通过带代次检查的 `context.publish({ persist: entry })` 决定是否持久化目录元数据；llama.cpp 等实时服务器可以返回模型而不持久化这些模型。

`context.signal` 始终是具体的信号，提供者回调必须把它传给阻塞式 I/O。公开的 `ModelRuntime.refresh()` 和 `ModelRegistry.refresh()` 接受可选信号；省略时不设时限，扩展和应用需自行决定截止时间。即使提供者忽略该信号，取消操作仍会让调用方停止等待，但要终止底层工作，提供者仍必须配合处理信号。

需要使用提供者原生身份验证、筛选、刷新或流式行为的扩展，可以注册来自 `@earendil-works/pi-ai` 的完整 `Provider`。该提供者会成为组合基础，而 `models.json` 的覆盖配置仍会应用在其上。

```typescript
import { createProvider, openAICompletionsApi } from "@earendil-works/pi-ai";

const provider = createProvider({
  id: "local-server",
  name: "Local Server",
  baseUrl: "http://localhost:8080/v1",
  auth: {
    apiKey: {
      name: "Local server setup",
      async login(interaction) {
        return {
          type: "api_key",
          key: await interaction.prompt({ type: "secret", message: "API key" }),
        };
      },
      async resolve({ credential }) {
        return credential?.key
          ? { auth: { apiKey: credential.key }, source: "stored API key" }
          : undefined;
      },
    },
  },
  models: [],
  api: openAICompletionsApi(),
});

pi.registerProvider(provider);

// Register a new provider with custom models
pi.registerProvider("my-proxy", {
  name: "My Proxy",
  baseUrl: "https://proxy.example.com",
  apiKey: "$PROXY_API_KEY",  // env var reference
  api: "anthropic-messages",
  models: [
    {
      id: "claude-sonnet-4-20250514",
      name: "Claude 4 Sonnet (proxy)",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 200000,
      maxTokens: 16384
    }
  ]
});

// Register a live llama.cpp catalog without persisting discovered models
pi.registerProvider("llama.cpp", {
  baseUrl: "http://localhost:8080/v1",
  apiKey: "local",
  api: "openai-completions",
  async refreshModels({ signal }) {
    const response = await fetch("http://localhost:8080/v1/models", { signal });
    const { data } = await response.json();
    return data.map(({ id }) => ({
      id,
      name: id,
      reasoning: false,
      input: ["text"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 128000,
      maxTokens: 16384
    }));
  }
});

// Override baseUrl for an existing provider (keeps all models)
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com"
});

// Register provider with OAuth support for /login
pi.registerProvider("corporate-ai", {
  baseUrl: "https://ai.corp.com",
  api: "openai-responses",
  models: [...],
  oauth: {
    name: "Corporate AI (SSO)",
    async login(callbacks) {
      // Custom OAuth flow
      callbacks.onAuth({ url: "https://sso.corp.com/..." });
      const code = await callbacks.onPrompt({ message: "Enter code:" });
      return { refresh: code, access: code, expires: Date.now() + 3600000 };
    },
    async refreshToken(credentials, signal) {
      signal.throwIfAborted();
      // Refresh logic
      return credentials;
    },
    getApiKey(credentials) {
      return credentials.access;
    }
  }
});
```

对象形式接受完整的 pi-ai `Provider`，包括原生 `auth`、`getModels`、`refreshModels`、`filterModels`、`stream` 和 `streamSimple` 行为。

**旧配置选项：**
- `name` - UI中提供者的显示名称，例如`/login`。
- `baseUrl` - API端点URL。定义模型时需要。
- `apiKey` - API 密钥字面量、环境变量插值（`$ENV_VAR` 或 `${ENV_VAR}`），或者以 `!command` 开头的命令。定义模型时必填（除非提供 `oauth`）。`$$` 用于转义 `$`，`$!` 用于转义字面量 `!`，且不会触发命令执行。
- `api` - API类型：`"anthropic-messages"`、`"openai-completions"`、`"openai-responses"`等。
- `headers` - 要包含在请求中的自定义标头。
- `authHeader` - 如果为 true，则自动添加 `Authorization: Bearer` 标头。
- `models` - 模型定义数组。如果提供，则替换该提供者的所有现有模型。模型定义可以设置 `baseUrl` 来覆盖该模型的提供者端点。
- `refreshModels` - 异步动态发现回调。其返回模型会替换扩展提供的模型。`context.stored` 包含持久化的提供者快照；只有更新后的目录数据需要持久化时，才调用带代次检查的 `context.publish({ persist: entry })`。使用 `persist: null` 删除该快照。
- `oauth` - OAuth `/login` 支持的提供者配置。提供后，提供者会出现在登录菜单中。
- `streamSimple` - 非标准 API 的自定义流式实现。

有关自定义流式 API、OAuth 细节和模型定义参考等高级主题，请参阅 [custom-provider.md](custom-provider.zh-CN.md)。

### pi.unregisterProvider(name)

删除先前注册的提供者及其模型。被提供者覆盖的内置模型将被恢复。如果提供者未注册，则无效。

与`registerProvider`一样，这在初始加载阶段后调用时立即生效，因此不需要`/reload`。

```typescript
pi.registerCommand("my-setup-teardown", {
  description: "Remove the custom proxy provider",
  handler: async (_args, _ctx) => {
    pi.unregisterProvider("my-proxy");
  },
});
```

## 状态管理

具有状态的扩展应将其存储在工具结果`details`中以获得正确的分支支持：

```typescript
export default function (pi: ExtensionAPI) {
  let items: string[] = [];

  // Reconstruct state from session
  pi.on("session_start", async (_event, ctx) => {
    items = [];
    for (const entry of ctx.sessionManager.getBranch()) {
      if (entry.type === "message" && entry.message.role === "toolResult") {
        if (entry.message.toolName === "my_tool") {
          items = entry.message.details?.items ?? [];
        }
      }
    }
  });

  pi.registerTool({
    name: "my_tool",
    // ...
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      items.push("new item");
      return {
        content: [{ type: "text", text: "Added" }],
        details: { items: [...items] },  // Store for reconstruction
      };
    },
  });
}
```

## 自定义工具

通过 `pi.registerTool()` 注册供 LLM 调用的工具。工具会出现在系统提示中，并可提供自定义渲染。

使用 `promptSnippet` 可在默认系统提示的 `Available tools` 部分添加一行简短说明。省略时，该部分不会列出自定义工具。

使用 `promptGuidelines` 可向默认系统提示的 `Guidelines` 部分添加工具专用条目。这些条目只会在工具启用时出现（例如调用 `pi.setActiveTools([...])` 后）。

**重要提示：** `promptGuidelines` 的条目会直接追加到 `Guidelines` 部分，不带工具名称前缀，也不按工具分组。每条准则必须明确指出所涉及的工具。不要写“在……时使用此工具”，因为 LLM 无法判断“此工具”指什么；应写成“在……时使用 my_tool”。

注意：有些模型会错误地在工具路径参数中加入 `@` 前缀。内置工具会在解析路径前移除开头的 `@`；如果自定义工具接受路径，也应进行同样的规范化。

如果自定义工具会修改文件，请使用 `withFileMutationQueue()`，使它加入内置 `edit` 和 `write` 使用的同一逐文件队列。这一点很重要，因为工具调用默认并行执行。若不使用队列，两个工具可能读取同一份旧内容、分别计算更新，最后写入的一方就会覆盖另一方。

失败示例：自定义工具编辑 `foo.ts`，同时内置 `edit` 也在同一助手轮次中修改 `foo.ts`。如果自定义工具未加入队列，两者都可能读取原始 `foo.ts` 并分别应用修改，最终丢失其中一项修改。

传给 `withFileMutationQueue()` 的必须是真实目标路径，而不是原始用户参数。先相对于 `ctx.cwd` 或工具工作目录将其解析为绝对路径。对于现有文件，该辅助函数会通过 `realpath()` 规范化路径，因此指向同一文件的符号链接别名会共用一个队列。对于新文件，由于还无法调用 `realpath()`，它会回退到解析后的绝对路径。

将整个变异窗口排队到该目标路径上。这包括读取-修改-写入逻辑，而不仅仅是最终写入。

```typescript
import { withFileMutationQueue } from "@earendil-works/pi-coding-agent";
import { mkdir, readFile, writeFile } from "node:fs/promises";
import { dirname, resolve } from "node:path";

async execute(_toolCallId, params, _signal, _onUpdate, ctx) {
  const absolutePath = resolve(ctx.cwd, params.path);

  return withFileMutationQueue(absolutePath, async () => {
    await mkdir(dirname(absolutePath), { recursive: true });
    const current = await readFile(absolutePath, "utf8");
    const next = current.replace(params.oldText, params.newText);
    await writeFile(absolutePath, next, "utf8");

    return {
      content: [{ type: "text", text: `Updated ${params.path}` }],
      details: {},
    };
  });
}
```

### 工具定义

```typescript
import { Type } from "typebox";
import { StringEnum } from "@earendil-works/pi-ai";
import { Text } from "@earendil-works/pi-tui";

pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "What this tool does (shown to LLM)",
  promptSnippet: "List or add items in the project todo list",
  promptGuidelines: [
    "Use my_tool for todo planning instead of direct file edits when the user asks for a task list."
  ],
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),  // Use StringEnum for Google compatibility
    text: Type.Optional(Type.String()),
  }),
  prepareArguments(args) {
    if (!args || typeof args !== "object") return args;
    const input = args as { action?: string; oldAction?: string };
    if (typeof input.oldAction === "string" && input.action === undefined) {
      return { ...input, action: input.oldAction };
    }
    return args;
  },

  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // Check for cancellation
    if (signal?.aborted) {
      return { content: [{ type: "text", text: "Cancelled" }] };
    }

    // Stream progress updates
    onUpdate?.({
      content: [{ type: "text", text: "Working..." }],
      details: { progress: 50 },
    });

    // Run commands via pi.exec (captured from extension closure)
    const result = await pi.exec("some-command", [], { signal });

    // Return result
    return {
      content: [{ type: "text", text: "Done" }],  // Sent to LLM
      details: { data: result },                   // For rendering & state
      // usage: nestedModelResponse.usage,          // Optional nested LLM usage
      // Optional: stop after this tool batch when every finalized tool result
      // in the batch also returns terminate: true.
      terminate: true,
    };
  },

  // Optional: Custom rendering
  renderCall(args, theme, context) { ... },
  renderResult(result, options, theme, context) { ... },
});
```

**用量统计：** 如果工具内部又调用了 LLM，请将这些调用合计后的 `Usage` 作为 `usage` 返回。pi 会把它保存在工具结果上，并计入页脚、`/session` 和 RPC 会话总量。`tool_result` 处理器可以检查或替换此值。

**表示错误：** 若要把工具执行标记为失败（在结果上设置 `isError: true` 并报告给 LLM），请从 `execute` 抛出错误。无论返回对象包含什么属性，正常返回都不会设置错误标志。

**提前终止：** 从 `execute()` 返回 `terminate: true`，可提示 pi 在当前工具批次结束后跳过自动的后续 LLM 调用。只有该批次中每个最终工具结果都要求终止时才会生效。有关代理以最终结构化输出工具调用结束的最小示例，请参阅 [examples/extensions/structured-output.ts](../examples/extensions/structured-output.ts)。

```typescript
// Correct: throw to signal an error
async execute(toolCallId, params) {
  if (!isValid(params.input)) {
    throw new Error(`Invalid input: ${params.input}`);
  }
  return { content: [{ type: "text", text: "OK" }], details: {} };
}
```

**重要提示：** 使用 `@earendil-works/pi-ai` 中的 `StringEnum` 作为字符串枚举。 `Type.Union`/`Type.Literal` 不适用于 Google 的 API。

**参数预处理：** `prepareArguments(args)` 是可选的。定义后，它会在 Schema 校验和 `execute()` 之前运行。当 pi 恢复旧会话，而其中保存的工具调用参数已不符合当前 Schema 时，可以用它兼容此前接受的输入结构。返回值将依据 `parameters` 校验。公开 Schema 应保持严格；不要仅为了让旧会话可恢复，就把已弃用的兼容字段加入 `parameters`。

示例：旧会话可能包含带有顶级 `oldText` 和 `newText` 的 `edit` 工具调用，而当前架构仅接受 `edits: [{ oldText, newText }]`。

```typescript
pi.registerTool({
  name: "edit",
  label: "Edit",
  description: "Edit a single file using exact text replacement",
  parameters: Type.Object({
    path: Type.String(),
    edits: Type.Array(
      Type.Object({
        oldText: Type.String(),
        newText: Type.String(),
      }),
    ),
  }),
  prepareArguments(args) {
    if (!args || typeof args !== "object") return args;

    const input = args as {
      path?: string;
      edits?: Array<{ oldText: string; newText: string }>;
      oldText?: unknown;
      newText?: unknown;
    };

    if (typeof input.oldText !== "string" || typeof input.newText !== "string") {
      return args;
    }

    return {
      ...input,
      edits: [...(input.edits ?? []), { oldText: input.oldText, newText: input.newText }],
    };
  },
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    // params now matches the current schema
    return {
      content: [{ type: "text", text: `Applying ${params.edits.length} edit block(s)` }],
      details: {},
    };
  },
});
```

### 覆盖内置工具

扩展程序可以通过注册同名工具来覆盖内置工具（`read`、`bash`、`powershell`、`edit`、`write`、`grep`、`find`、`ls`）。发生这种情况时，交互模式会显示警告。

```bash
# Extension's read tool replaces built-in read
pi -e ./tool-override.ts
```

或者，使用 `--no-builtin-tools` 在不使用任何内置工具的情况下启动，同时保持扩展工具启用：
```bash
# No built-in tools, only extension tools
pi --no-builtin-tools -e ./my-extension.ts
```

有关使用日志记录和访问控制覆盖 `read` 的完整示例，请参阅 [examples/extensions/tool-override.ts](../examples/extensions/tool-override.ts)。

**渲染：** 内置渲染器的继承按槽位分别解析，执行覆盖与渲染覆盖互不影响。若覆盖未定义 `renderCall`，则使用内置 `renderCall`；未定义 `renderResult` 时，则使用内置 `renderResult`；两者均未定义时，会自动使用完整的内置渲染器（语法高亮、diff 等）。因此可以为日志记录或访问控制包装内置工具，而无需重新实现 UI。

**提示元数据：** `promptSnippet`和`promptGuidelines`不是从内置工具继承的。如果您的覆盖应保留这些提示说明，请在覆盖上明确定义它们。

**您的实现必须与确切的结果形状匹配**，包括 `details` 类型。 UI 和会话逻辑依赖于这些形状来进行渲染和状态跟踪。

内置工具实现：
- [read.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/read.ts) - `ReadToolDetails`
- [bash.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/bash.ts) - `BashToolDetails`
- [powershell.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/powershell.ts) - `PowerShellToolDetails`
- [edit.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/edit.ts)
- [write.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/write.ts)
- [grep.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/grep.ts) - `GrepToolDetails`
- [find.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/find.ts) - `FindToolDetails`
- [ls.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/tools/ls.ts) - `LsToolDetails`

### 远程执行

内置工具支持可插拔操作以委托给远程系统（SSH、容器等）：

```typescript
import { createReadTool, createBashTool, type ReadOperations } from "@earendil-works/pi-coding-agent";

// Create tool with custom operations
const remoteRead = createReadTool(cwd, {
  operations: {
    readFile: (path) => sshExec(remote, `cat ${path}`),
    access: (path) => sshExec(remote, `test -r ${path}`).then(() => {}),
  }
});

// Register, checking flag at execution time
pi.registerTool({
  ...remoteRead,
  async execute(id, params, signal, onUpdate, _ctx) {
    const ssh = getSshConfig();
    if (ssh) {
      const tool = createReadTool(cwd, { operations: createRemoteOps(ssh) });
      return tool.execute(id, params, signal, onUpdate);
    }
    return localRead.execute(id, params, signal, onUpdate);
  },
});
```

**操作接口：** `ReadOperations`、`WriteOperations`、`EditOperations`、`BashOperations`、`PowerShellOperations`、`LsOperations`、`GrepOperations`、`FindOperations`

对于`user_bash`，扩展可以通过`createLocalBashOperations()`重用pi的本地shell后端，而不是重新实现本地进程生成、shell解析和进程树终止。

`bash`和`powershell`工具还支持spawn hook来在执行前调整命令、cwd或env：

```typescript
import { createBashTool } from "@earendil-works/pi-coding-agent";

const bashTool = createBashTool(cwd, {
  spawnHook: ({ command, cwd, env }) => ({
    command: `source ~/.profile\n${command}`,
    cwd: `/mnt/sandbox${cwd}`,
    env: { ...env, CI: "1" },
  }),
});
```

`createBashTool()`和`createPowerShellTool()`通过`PI_SESSION_ID`、`PI_SESSION_FILE`、`PI_PROVIDER`、`PI_MODEL`和`PI_REASONING_LEVEL`将当前会话公开给命令。注入发生在`spawnHook`之前，因此钩子在`env`中接收这些值，并在如上所述传播现有环境时保留它们。设置`exposeSessionEnvironment: false`以禁用它们：

```typescript
const bashTool = createBashTool(cwd, {
  exposeSessionEnvironment: false,
});
```

有关变量语义，请参阅 [Shell 工具的会话环境](environment-variables.zh-CN.md#shell-工具的会话环境)。有关带 `--ssh` 标志的完整 SSH 示例，请参阅 [examples/extensions/ssh.ts](../examples/extensions/ssh.ts)。

### 输出截断

**工具必须截断输出**，以免占满 LLM 上下文。输出过大可能导致：
- 上下文溢出错误（提示太长）
- 压缩失败
- 模型性能下降

内置上限为 **50KB**（约 10k token）或 **2000 行**，以先达到的上限为准。请使用导出的截断工具函数：

```typescript
import {
  truncateHead,      // Keep first N lines/bytes (good for file reads, search results)
  truncateTail,      // Keep last N lines/bytes (good for logs, command output)
  truncateLine,      // Truncate a single line to maxBytes with ellipsis
  formatSize,        // Human-readable size (e.g., "50KB", "1.5MB")
  DEFAULT_MAX_BYTES, // 50KB
  DEFAULT_MAX_LINES, // 2000
} from "@earendil-works/pi-coding-agent";

async execute(toolCallId, params, signal, onUpdate, ctx) {
  const output = await runCommand();

  // Apply truncation
  const truncation = truncateHead(output, {
    maxLines: DEFAULT_MAX_LINES,
    maxBytes: DEFAULT_MAX_BYTES,
  });

  let result = truncation.content;

  if (truncation.truncated) {
    // Write full output to temp file
    const tempFile = writeTempFile(output);

    // Inform the LLM where to find complete output
    result += `\n\n[Output truncated: ${truncation.outputLines} of ${truncation.totalLines} lines`;
    result += ` (${formatSize(truncation.outputBytes)} of ${formatSize(truncation.totalBytes)}).`;
    result += ` Full output saved to: ${tempFile}]`;
  }

  return { content: [{ type: "text", text: result }] };
}
```

**要点：**
- 对于开头很重要的内容使用`truncateHead`（搜索结果、文件读取）
- 对结尾重要的内容使用`truncateTail`（日志、命令输出）
- 输出被截断时始终通知LLM以及在哪里可以找到完整版本
- 在工具描述中记录截断限制

请参阅 [examples/extensions/truncated-tool.ts](../examples/extensions/truncated-tool.ts) 以获取使用适当截断来包装 `rg` (ripgrep) 的完整示例。

### 多种工具

一个扩展可以注册多个具有共享状态的工具：

```typescript
export default function (pi: ExtensionAPI) {
  let connection = null;

  pi.registerTool({ name: "db_connect", ... });
  pi.registerTool({ name: "db_query", ... });
  pi.registerTool({ name: "db_close", ... });

  pi.on("session_shutdown", async () => {
    connection?.close();
  });
}
```

### 自定义渲染

工具可以提供`renderCall`和`renderResult`用于自定义TUI显示。请参阅 [tui.md](tui.zh-CN.md) 了解完整组件 API 和 [tool-execution.ts](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/modes/interactive/components/tool-execution.ts) 了解工具行的组成方式。

默认情况下，工具输出包含在处理填充和背景的 `Box` 中。定义的`renderCall`或`renderResult`必须返回`Component`。如果未定义槽渲染器，则`tool-execution.ts`对该槽使用回退渲染。

当工具应该渲染自己的 shell 而不是使用默认的 `Box` 时，设置 `renderShell: "self"`。这对于需要完全控制取景或背景行为的工具非常有用，例如工具稳定后必须保持视觉稳定的大型预览。

```typescript
pi.registerTool({
  name: "my_tool",
  label: "My Tool",
  description: "Custom shell example",
  parameters: Type.Object({}),
  renderShell: "self",
  async execute() {
    return { content: [{ type: "text", text: "ok" }], details: undefined };
  },
  renderCall(args, theme, context) {
    return new Text(theme.fg("accent", "my custom shell"), 0, 0);
  },
});
```

`renderCall`和`renderResult`各自接收一个`context`对象，其中：
- `args` - 当前工具调用参数
- `state` - 跨`renderCall`和`renderResult`共享行本地状态
- `lastComponent` - 该插槽之前返回的组件（如果有）
- `invalidate()` - 请求重新渲染此工具行
- `toolCallId`, `cwd`, `executionStarted`, `argsComplete`, `isPartial`, `expanded`, `showImages`, `isError`

使用 `context.state` 来表示跨槽共享状态。当您想要跨渲染重用和改变同一组件时，请在返回的组件实例上保留插槽本地缓存。

#### renderCall

呈现工具调用或标头：

```typescript
import { Text } from "@earendil-works/pi-tui";

renderCall(args, theme, context) {
  const text = (context.lastComponent as Text | undefined) ?? new Text("", 0, 0);
  let content = theme.fg("toolTitle", theme.bold("my_tool "));
  content += theme.fg("muted", args.action);
  if (args.text) {
    content += " " + theme.fg("dim", `"${args.text}"`);
  }
  text.setText(content);
  return text;
}
```

#### renderResult

呈现工具结果或输出：

```typescript
renderResult(result, { expanded, isPartial }, theme, context) {
  if (isPartial) {
    return new Text(theme.fg("warning", "Processing..."), 0, 0);
  }

  if (result.details?.error) {
    return new Text(theme.fg("error", `Error: ${result.details.error}`), 0, 0);
  }

  let text = theme.fg("success", "✓ Done");
  if (expanded && result.details?.items) {
    for (const item of result.details.items) {
      text += "\n  " + theme.fg("dim", item);
    }
  }
  return new Text(text, 0, 0);
}
```

如果某个槽故意没有可见内容，则返回空的 `Component`，例如空的 `Container`。

#### 按键绑定提示

使用 `keyHint()` 显示遵循活动键绑定配置的键绑定提示：

```typescript
import { keyHint } from "@earendil-works/pi-coding-agent";

renderResult(result, { expanded }, theme, context) {
  let text = theme.fg("success", "✓ Done");
  if (!expanded) {
    text += ` (${keyHint("app.tools.expand", "to expand")})`;
  }
  return new Text(text, 0, 0);
}
```

可用功能：
- `keyHint(keybinding, description)` - 格式化配置的键绑定 ID，例如 `"app.tools.expand"` 或 `"tui.select.confirm"`
- `keyText(keybinding)` - 返回按键绑定 id 的原始配置按键文本
- `rawKeyHint(key, description)` - 格式化原始密钥字符串

使用命名空间键绑定 ID：
- 编码代理 ID 使用`app.*`命名空间，例如`app.tools.expand`、`app.editor.external`、`app.session.rename`
- 共享TUI ids 使用`tui.*`命名空间，例如`tui.select.confirm`、`tui.select.cancel`、`tui.input.tab`

有关按键绑定 ID 和默认值的详尽列表，请参阅 [keybindings.md](keybindings.zh-CN.md)。 `keybindings.json` 使用相同的命名空间 ID。

自定义编辑器和 `ctx.ui.custom()` 组件接收 `keybindings: KeybindingsManager` 作为注入参数。他们应该直接使用注入的管理器，而不是调用`getKeybindings()`或`setKeybindings()`。

#### 最佳实践

- 使用`Text`和填充`(0, 0)`。默认的 Box 处理填充。
- 使用`\n`表示多行内容。
- 处理`isPartial`以获取流式传输进度。
- 支持`expanded`按需提供详细信息。
- 保持默认视图紧凑。
- 读取`renderResult`中的`context.args`，而不是将参数复制到`context.state`。
- 仅对必须在调用和结果槽之间共享的数据使用`context.state`。
- 当可以就地更新相同的组件实例时，重用 `context.lastComponent`。
- 仅当默认的Box 外壳 妨碍时才使用 `renderShell: "self"`。在自行渲染外壳模式下，该工具负责其自己的框架、填充和背景。

#### 回退

如果槽渲染器未定义或抛出：
- `renderCall`：显示工具名称
- `renderResult`：显示来自`content`的原始文本

### 动态工具加载

扩展可以注册大量工具，但初始只启用其中一小部分。工具可在执行期间调用 `pi.setActiveTools()`，进一步启用其他工具。pi 会识别纯新增的变化，在当前工具结果上记录新启用的工具名称，并在下一次模型请求前应用更新后的启用集合。

此机制适用于所有模型。原生支持延迟加载的模型可以保持提示前缀稳定，并在工具结果所在位置加载新定义；其他模型采用下文所述的回退行为。

生命周期是：

1. 用`pi.registerTool()`注册每个工具，使其出现在`pi.getAllTools()`中。
2. 保持加载工具（例如`search_tools`）处于活动状态，并使可搜索工具保持不活动状态。
3. 加载器执行期间，调用`pi.setActiveTools([...currentTools, ...matchingTools])`。更改必须是附加的：不要在同一调用中删除当前活动的工具。
4. Pi 记录加载器的工具结果上添加了哪些工具。
5. 在下一个模型响应之前，Pi 在支持时使用原生延迟加载公开添加的定义，否则使用正常的活动工具列表。

无需返回提供者专用的工具引用，也无需把加载器标记为特殊的搜索工具；启用工具集合的变化本身就是信号。传给 `pi.setActiveTools()` 的名称必须已注册，未知名称会被忽略。

#### 具有原生延迟加载的模型

- **Anthropic**
  - **模型：** Sonnet、Opus、Fable 4.5 或更高版本（不含 Haiku）
  - **原生表示：** 延迟定义使用`defer_loading`；加载点使用`tool_reference`内容。
- **OpenAI**
  - **模型：** `gpt-5.4` 及更新系列
  - **原生表示：** Pi 在加载点添加已完成的客户端 `tool_search_call` 和 `tool_search_output` 项。

对于经过验证的自定义模型或代理，可以使用 `anthropic-messages` 的 `compat.supportsToolReferences: true` 或`openai-responses` 和 `openai-codex-responses` 的 `compat.supportsToolSearch: true` 启用原生处理。除非端点和模型接受相应的原生协议，否则将它们保持禁用状态。

#### 回退行为

对于所有其他模型和提供者，动态激活仍然有效：Pi 通常在下一个请求时发送完整的当前活动工具列表。该模型可以调用新激活的工具，但添加它们的定义可能会使提供者的缓存提示前缀无效。

当活动集不是纯粹的累加时，例如用一组工具替换另一组工具时，Pi 也会使用这种安全回退。因此，工具删除可以工作，但它们不使用延迟加载。

为了获得最佳缓存行为，请在整个会话中保持加载程序工具处于活动状态并添加工具而不是替换活动集。另请注意，使用`promptSnippet`或`promptGuidelines`激活工具会重建系统提示符；即使提供者支持延迟模式，系统提示的更改也可能使前缀无效。延迟加载的工具通常应该依赖于它们的工具`description`并省略仅活动的提示元数据。

#### 搜索工具示例

以下扩展注册了两个可搜索工具，将它们从初始活动集中删除，并仅保留 `search_tools` 作为它们的加载程序。该示例使用简单的关键字匹配，但搜索实现可以使用 BM25、嵌入、远程目录或特定于项目的路由。

```typescript
import type { ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { Type } from "typebox";

const SEARCHABLE_TOOL_NAMES = new Set(["lookup_weather", "search_issues"]);

export default function (pi: ExtensionAPI) {
  pi.registerTool({
    name: "lookup_weather",
    label: "Lookup Weather",
    description: "Look up the current weather for a city",
    parameters: Type.Object({ city: Type.String() }),
    async execute(_toolCallId, params) {
      return {
        content: [{ type: "text", text: `Weather for ${params.city}: sunny` }],
        details: {},
      };
    },
  });

  pi.registerTool({
    name: "search_issues",
    label: "Search Issues",
    description: "Search project issues by keyword",
    parameters: Type.Object({ query: Type.String() }),
    async execute(_toolCallId, params) {
      return {
        content: [{ type: "text", text: `No open issues matching ${params.query}` }],
        details: {},
      };
    },
  });

  pi.registerTool({
    name: "search_tools",
    label: "Search Tools",
    description: "Search for and enable tools relevant to a task",
    promptSnippet: "Search for additional tools when the active tools cannot perform the task",
    promptGuidelines: [
      "Use search_tools when a task requires a capability that is not currently available.",
    ],
    parameters: Type.Object({
      query: Type.String({ description: "Capability or task to search for" }),
      limit: Type.Optional(Type.Integer({ minimum: 1, maximum: 10 })),
    }),
    async execute(_toolCallId, params) {
      const terms = params.query.toLowerCase().split(/[^a-z0-9]+/).filter(Boolean);
      const matches = pi.getAllTools()
        .filter((tool) => SEARCHABLE_TOOL_NAMES.has(tool.name))
        .map((tool) => ({
          tool,
          score: terms.reduce(
            (score, term) =>
              score + (`${tool.name} ${tool.description}`.toLowerCase().includes(term) ? 1 : 0),
            0,
          ),
        }))
        .filter((match) => match.score > 0)
        .sort((a, b) => b.score - a.score)
        .slice(0, params.limit ?? 3)
        .map((match) => match.tool.name);

      if (matches.length === 0) {
        return {
          content: [{ type: "text", text: `No tools found for: ${params.query}` }],
          details: { matches: [] },
        };
      }

      const active = pi.getActiveTools();
      const added = matches.filter((name) => !active.includes(name));
      pi.setActiveTools([...new Set([...active, ...added])]);

      return {
        content: [{
          type: "text",
          text: added.length > 0
            ? `Loaded tools: ${added.join(", ")}`
            : `Matching tools already active: ${matches.join(", ")}`,
        }],
        details: { matches, added },
      };
    },
  });

  pi.on("session_start", () => {
    // Keep searchable tools registered but initially inactive. Preserve built-ins
    // and tools owned by other extensions, and keep the loader itself active.
    const initialTools = pi.getActiveTools().filter(
      (name) => !SEARCHABLE_TOOL_NAMES.has(name),
    );
    pi.setActiveTools([...new Set([...initialTools, "search_tools"])]);
  });
}
```

当 `search_tools` 添加匹配工具后，模型会在紧接着的下一次请求中收到其定义。对于原生支持此功能的模型，定义会锚定在搜索结果之后，不改变初始工具 Schema 前缀；对于其他模型，它会在同一后续请求的常规工具列表中出现。

## 自定义 UI

扩展可以通过 `ctx.ui` 方法与用户交互，并自定义消息/工具的呈现方式。

**自定义组件请参阅 [tui.md](tui.zh-CN.md)**，其中提供了可直接复制的以下模式：
- 选择对话框（SelectList）
- 可取消的异步操作（BorderedLoader）
- 设置开关（SettingsList）
- 状态指示器（setStatus）
- 流式响应期间的工作消息、可见性和指示器（`setWorkingMessage`、`setWorkingVisible`、`setWorkingIndicator`）
- 编辑器上方或下方的小组件（setWidget）
- 叠加在内置斜杠命令和路径补全之上的自动补全提供者（addAutocompleteProvider）
- 自定义页脚（setFooter）

### 对话框

```typescript
// Select from options
const choice = await ctx.ui.select("Pick one:", ["A", "B", "C"]);

// Confirm dialog
const ok = await ctx.ui.confirm("Delete?", "This cannot be undone");

// Text input
const name = await ctx.ui.input("Name:", "placeholder");

// Multi-line editor
const text = await ctx.ui.editor("Edit:", "prefilled text");

// Notification (non-blocking)
ctx.ui.notify("Done!", "info");  // "info" | "warning" | "error"
```

#### 带倒计时的定时对话框

对话框支持 `timeout` 选项，该选项可通过实时倒计时显示自动关闭：

```typescript
// Dialog shows "Title (5s)" → "Title (4s)" → ... → auto-dismisses at 0
const confirmed = await ctx.ui.confirm(
  "Timed Confirmation",
  "This dialog will auto-cancel in 5 seconds. Confirm?",
  { timeout: 5000 }
);

if (confirmed) {
  // User confirmed
} else {
  // User cancelled or timed out
}
```

**超时返回值：**
- `select()`返回`undefined`
- `confirm()`返回`false`
- `input()`返回`undefined`

#### 使用 AbortSignal 手动关闭

如需更精细的控制（例如区分超时和用户取消），请使用 `AbortSignal`：

```typescript
const controller = new AbortController();
const timeoutId = setTimeout(() => controller.abort(), 5000);

const confirmed = await ctx.ui.confirm(
  "Timed Confirmation",
  "This dialog will auto-cancel in 5 seconds. Confirm?",
  { signal: controller.signal }
);

clearTimeout(timeoutId);

if (confirmed) {
  // User confirmed
} else if (controller.signal.aborted) {
  // Dialog timed out
} else {
  // User cancelled (pressed Escape or selected "No")
}
```

有关完整示例，请参阅 [examples/extensions/timed-confirm.ts](../examples/extensions/timed-confirm.ts)。

### 小组件、状态和页脚

```typescript
// Status in footer (persistent until cleared)
ctx.ui.setStatus("my-ext", "Processing...");
ctx.ui.setStatus("my-ext", undefined);  // Clear

// Working loader (shown during streaming)
ctx.ui.setWorkingMessage("Thinking deeply...");
ctx.ui.setWorkingMessage();  // Restore default
ctx.ui.setWorkingVisible(false);  // Hide the built-in working loader row entirely
ctx.ui.setWorkingVisible(true);   // Show the built-in working loader row

// Working indicator (shown during streaming)
ctx.ui.setWorkingIndicator({ frames: [ctx.ui.theme.fg("accent", "●")] });  // Static dot
ctx.ui.setWorkingIndicator({
  frames: [
    ctx.ui.theme.fg("dim", "·"),
    ctx.ui.theme.fg("muted", "•"),
    ctx.ui.theme.fg("accent", "●"),
    ctx.ui.theme.fg("muted", "•"),
  ],
  intervalMs: 120,
});
ctx.ui.setWorkingIndicator({ frames: [] });  // Hide indicator
ctx.ui.setWorkingIndicator();  // Restore default spinner

// Widget above editor (default)
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"]);
// Widget below editor
ctx.ui.setWidget("my-widget", ["Line 1", "Line 2"], { placement: "belowEditor" });
ctx.ui.setWidget("my-widget", (tui, theme) => new Text(theme.fg("accent", "Custom"), 0, 0));
ctx.ui.setWidget("my-widget", undefined);  // Clear

// Custom footer (replaces built-in footer entirely)
ctx.ui.setFooter((tui, theme) => ({
  render(width) { return [theme.fg("dim", "Custom footer")]; },
  invalidate() {},
}));
ctx.ui.setFooter(undefined);  // Restore built-in footer

// Terminal title
ctx.ui.setTitle("pi - my-project");

// Editor text
ctx.ui.setEditorText("Prefill text");
const current = ctx.ui.getEditorText();

// Paste into editor (triggers paste handling, including collapse for large content)
ctx.ui.pasteToEditor("pasted content");

// Stack custom autocomplete behavior on top of the built-in provider
ctx.ui.addAutocompleteProvider((current) => ({
  triggerCharacters: ["#"],
  async getSuggestions(lines, line, col, options) {
    const beforeCursor = (lines[line] ?? "").slice(0, col);
    const match = beforeCursor.match(/(?:^|[ \t])#([^\s#]*)$/);
    if (!match) {
      return current.getSuggestions(lines, line, col, options);
    }

    return {
      prefix: `#${match[1] ?? ""}`,
      items: [{ value: "#2983", label: "#2983", description: "Extension API for autocomplete" }],
    };
  },
  applyCompletion(lines, line, col, item, prefix) {
    return current.applyCompletion(lines, line, col, item, prefix);
  },
  shouldTriggerFileCompletion(lines, line, col) {
    return current.shouldTriggerFileCompletion?.(lines, line, col) ?? true;
  },
}));

// Tool output expansion
const wasExpanded = ctx.ui.getToolsExpanded();
ctx.ui.setToolsExpanded(true);
ctx.ui.setToolsExpanded(wasExpanded);

// Custom editor (vim mode, emacs mode, etc.)
ctx.ui.setEditorComponent((tui, theme, keybindings) => new VimEditor(tui, theme, keybindings));
const currentEditor = ctx.ui.getEditorComponent();
ctx.ui.setEditorComponent((tui, theme, keybindings) =>
  new WrappedEditor(tui, theme, keybindings, currentEditor?.(tui, theme, keybindings))
);
ctx.ui.setEditorComponent(undefined);  // Restore default editor

// Theme management (see themes.md for creating themes)
const themes = ctx.ui.getAllThemes();  // [{ name: "dark", path: "/..." | undefined }, ...]
const lightTheme = ctx.ui.getTheme("light");  // Load without switching
const result = ctx.ui.setTheme("light");  // Switch by name
if (!result.success) {
  ctx.ui.notify(`Failed: ${result.error}`, "error");
}
ctx.ui.setTheme(lightTheme!);  // Or switch by Theme object
ctx.ui.theme.fg("accent", "styled text");  // Access current theme
```

自定义工作指示器的每一帧都会原样渲染。如需颜色，请自行在帧字符串中添加，例如使用 `ctx.ui.theme.fg(...)`。

### 自动补全提供者

使用 `ctx.ui.addAutocompleteProvider()` 可在内置斜杠命令和路径补全提供者之上叠加自定义自动补全逻辑。通过 `triggerCharacters` 设置 `$` 等自定义自然触发字符。

典型模式：

- 检查光标之前的文本
- 匹配到扩展专用语法时返回自己的建议
- 否则委托给 `current.getSuggestions(...)`
- 除非需要自定义插入行为，否则将 `applyCompletion(...)` 委托给当前提供者

```typescript
pi.on("session_start", (_event, ctx) => {
  ctx.ui.addAutocompleteProvider((current) => ({
    triggerCharacters: ["#"],
    async getSuggestions(lines, cursorLine, cursorCol, options) {
      const line = lines[cursorLine] ?? "";
      const beforeCursor = line.slice(0, cursorCol);
      const match = beforeCursor.match(/(?:^|[ \t])#([^\s#]*)$/);
      if (!match) {
        return current.getSuggestions(lines, cursorLine, cursorCol, options);
      }

      return {
        prefix: `#${match[1] ?? ""}`,
        items: [
          { value: "#2983", label: "#2983", description: "Extension API for registering custom @ autocomplete providers" },
          { value: "#2753", label: "#2753", description: "Reload stale resource settings" },
        ],
      };
    },

    applyCompletion(lines, cursorLine, cursorCol, item, prefix) {
      return current.applyCompletion(lines, cursorLine, cursorCol, item, prefix);
    },

    shouldTriggerFileCompletion(lines, cursorLine, cursorCol) {
      return current.shouldTriggerFileCompletion?.(lines, cursorLine, cursorCol) ?? true;
    },
  }));
});
```

完整示例请参阅 [github-issue-autocomplete.ts](../examples/extensions/github-issue-autocomplete.ts)。该示例使用 `gh issue list` 预加载最近的开放 GitHub issue，并在本地筛选，以快速完成 `#...` 补全。它要求安装 GitHub CLI（`gh`），并且当前目录是 GitHub 仓库检出目录。

### 自定义组件

对于复杂 UI，请使用 `ctx.ui.custom()`。它会暂时用自定义组件替换编辑器，直到调用 `done()`：

```typescript
import { Text, Component } from "@earendil-works/pi-tui";

const result = await ctx.ui.custom<boolean>((tui, theme, keybindings, done) => {
  const text = new Text("Press Enter to confirm, Escape to cancel", 1, 1);

  text.onKey = (key) => {
    if (key === "return") done(true);
    if (key === "escape") done(false);
    return true;
  };

  return text;
});

if (result) {
  // User pressed Enter
}
```

回调收到：
- `tui` - TUI 实例（用于屏幕尺寸和焦点管理）
- `theme` - 当前样式主题
- `keybindings` - 应用程序键绑定管理器（用于检查快捷方式）
- `done(value)` - 调用关闭组件并返回值

有关完整组件 API，请参阅 [tui.md](tui.zh-CN.md)。

#### 叠加模式（实验）

传入 `{ overlay: true }` 会将组件渲染为现有内容之上的浮动模态框，而不清空屏幕：

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new MyOverlayComponent({ onClose: done }),
  { overlay: true }
);
```

对于高级定位（锚点、边距、百分比、响应式可见性），请传递`overlayOptions`。使用 `onHandle` 以编程方式控制焦点或可见性：

```typescript
const result = await ctx.ui.custom<string | null>(
  (tui, theme, keybindings, done) => new MyOverlayComponent({ onClose: done }),
  {
    overlay: true,
    overlayOptions: { anchor: "top-right", width: "50%", margin: 2 },
    onHandle: (handle) => {
      handle.focus(); // focus this overlay and bring it to the visual front
      // handle.unfocus({ target: editorComponent }); // release input to a specific component
      // handle.setHidden(true/false); // toggle visibility
      // handle.hide(); // permanently remove
    }
  }
);
```

临时的非叠加自定义 UI 关闭后，已聚焦且可见的叠加层可以重新接管输入。若要让其他组件在叠加层仍可见时继续接收输入，请调用 `handle.unfocus({ target })`。传入 `{ target: null }` 可让叠加层释放输入，同时不聚焦其他组件。

有关完整的 `OverlayOptions` 和 `OverlayHandle` API 以及 [overlay-qa-tests.ts](../examples/extensions/overlay-qa-tests.ts) 的示例，请参阅 [tui.md](tui.zh-CN.md)。

### 自定义编辑器

用自定义实现（vim 模式、emacs 模式等）替换主输入编辑器：

```typescript
import { CustomEditor, type ExtensionAPI } from "@earendil-works/pi-coding-agent";
import { matchesKey } from "@earendil-works/pi-tui";

class VimEditor extends CustomEditor {
  private mode: "normal" | "insert" = "insert";

  handleInput(data: string): void {
    if (matchesKey(data, "escape") && this.mode === "insert") {
      this.mode = "normal";
      return;
    }
    if (this.mode === "normal" && data === "i") {
      this.mode = "insert";
      return;
    }
    super.handleInput(data);  // App keybindings + text editing
  }
}

export default function (pi: ExtensionAPI) {
  pi.on("session_start", (_event, ctx) => {
    ctx.ui.setEditorComponent((tui, theme, keybindings) =>
      new VimEditor(tui, theme, keybindings)
    );
  });
}
```

**要点：**
- 扩展 `CustomEditor`（而不是基础 `Editor`），以获得应用级按键绑定支持（按 Escape 中止、Ctrl+D、切换模型等）
- 对未自行处理的按键调用 `super.handleInput(data)`
- 自定义编辑器默认保留独立的工作状态行。若要使用内置的编辑器边框旋转指示器，请将 `{ embedWorkingStatus: true }` 作为 `CustomEditor` 构造函数的第四个参数
- 工厂会从应用接收 `tui`、`theme` 和 `keybindings`
- 在 `setEditorComponent()` 之前调用 `ctx.ui.getEditorComponent()`，可以包装此前配置的自定义编辑器
- 传入 `undefined` 可恢复默认编辑器：`ctx.ui.setEditorComponent(undefined)`

要与已替换编辑器的另一个扩展进行组合，请在设置您的工厂之前捕获以前的工厂：

```typescript
const previous = ctx.ui.getEditorComponent();
ctx.ui.setEditorComponent((tui, theme, keybindings) =>
  new MyEditor(tui, theme, keybindings, { base: previous?.(tui, theme, keybindings) })
);
```

有关模式指示器的完整示例，请参阅 [tui.md](tui.zh-CN.md) 模式 7。

### 消息和条目渲染

为指定 `customType` 的消息注册自定义渲染器。需要进入 LLM 上下文的内容应使用消息渲染器：

```typescript
import { Text } from "@earendil-works/pi-tui";

pi.registerMessageRenderer("my-extension", (message, options, theme) => {
  const { expanded, outputPad } = options;
  let text = theme.fg("accent", `[${message.customType}] `);
  text += message.content;

  if (expanded && message.details) {
    text += "\n" + theme.fg("dim", JSON.stringify(message.details, null, 2));
  }

  return new Text(text, outputPad, 0);
});
```

消息通过`pi.sendMessage()`发送：

```typescript
pi.sendMessage({
  customType: "my-extension",  // Matches registerMessageRenderer
  content: "Status update",
  display: true,               // Show in TUI
  details: { ... },            // Available in renderer
});
```

对于不应发送到 LLM 的仅限 TUI 的内容，请改为渲染自定义条目：

```typescript
pi.registerEntryRenderer("my-card", (entry, options, theme) => {
  return new Text(theme.fg("accent", JSON.stringify(entry.data)));
});

pi.appendEntry("my-card", { status: "done" });
```

### 主题颜色

所有渲染函数都会接收一个 `theme` 对象。请参阅 [themes.md](themes.zh-CN.md) 创建自定义主题和完整调色板。

```typescript
// Foreground colors
theme.fg("toolTitle", text)   // Tool names
theme.fg("accent", text)      // Highlights
theme.fg("success", text)     // Success (green)
theme.fg("error", text)       // Errors (red)
theme.fg("warning", text)     // Warnings (yellow)
theme.fg("muted", text)       // Secondary text
theme.fg("dim", text)         // Tertiary text

// Text styles
theme.bold(text)
theme.italic(text)
theme.strikethrough(text)
```

对于自定义工具渲染器中的语法突出显示：

```typescript
import { highlightCode, getLanguageFromPath } from "@earendil-works/pi-coding-agent";

// Highlight code with explicit language
const highlighted = highlightCode("const x = 1;", "typescript", theme);

// Auto-detect language from file path
const lang = getLanguageFromPath("/path/to/file.rs");  // "rust"
const highlighted = highlightCode(code, lang, theme);
```

## 错误处理

- 扩展错误会被记录，代理继续运行
- `tool_call` 中的错误会阻止工具执行（故障安全）
- 工具 `execute` 的错误必须通过抛出异常表示；异常会被捕获，并以 `isError: true` 报告给 LLM，随后继续执行

## 模式行为

| 模式 | `ctx.mode` | `ctx.hasUI` | 说明 |
|------|------------|-------------|-------|
| 交互模式 | `"tui"` | `true` | 提供完整的 TUI 和终端渲染 |
| RPC（`--mode rpc`） | `"rpc"` | `true` | 通过 JSON 协议提供对话框和通知；`custom()` 返回 `undefined`。参阅 [rpc.md](rpc.zh-CN.md) |
| JSON（`--mode json`） | `"json"` | `false` | 将事件流输出到 stdout；UI 方法不执行任何操作 |
| 打印模式（`-p`） | `"print"` | `false` | 扩展会运行，但不能提示用户 |

使用 TUI 专用功能（`custom()`、组件工厂、终端输入）前，请检查 `ctx.mode === "tui"`。使用同时适用于 TUI 和 RPC 模式的对话框及通知方法前，请检查 `ctx.hasUI`。

## 示例参考

[examples/extensions/](../examples/extensions/) 中提供了全部示例。

| 示例 | 说明 | 关键 API |
|---------|-------------|----------|
| **工具** |||
| `hello.ts` | 最小化工具注册 | `registerTool` |
| `question.ts` | 带用户交互的工具 | `registerTool`、`ui.select` |
| `questionnaire.ts` | 多步骤向导工具 | `registerTool`、`ui.custom` |
| `todo.ts` | 支持持久化的有状态工具 | `registerTool`、`appendEntry`、`renderResult`、会话事件 |
| `dynamic-tools.ts` | 在启动后和命令执行期间注册工具 | `registerTool`、`session_start`、`registerCommand` |
| `structured-output.ts` | 带 `terminate: true` 的最终结构化输出工具 | `registerTool`、终止型工具结果 |
| `truncated-tool.ts` |输出截断示例 | `registerTool`、`truncateHead` |
| `tool-override.ts` |覆盖内置读取工具 | `registerTool`（与内置名称相同）|
| **命令** |||
| `pirate.ts` |每回合修改系统提示 | `registerCommand`、`before_agent_start` |
| `summarize.ts` |对话摘要命令 | `registerCommand`、`ui.custom` |
| `handoff.ts` |跨提供者模型切换 | `registerCommand`、`ui.editor`、`ui.custom` |
| `qna.ts` | 带自定义 UI 的问答 | `registerCommand`、`ui.custom`、`setEditorText` |
| `send-user-message.ts` |注入用户消息 | `registerCommand`、`sendUserMessage` |
| `reload-runtime.ts` | 重载命令及 LLM 工具衔接 | `registerCommand`、`ctx.reload()`、`sendUserMessage` |
| `shutdown-command.ts` | 优雅关闭命令 | `registerCommand`、`shutdown()` |
| **事件与门禁** |||
| `permission-gate.ts` |阻止危险命令 | `on("tool_call")`、`ui.confirm` |
| `project-trust.ts` | 由用户级、全局或 CLI 扩展决定或推迟项目信任 | `on("project_trust")`、信任 UI、必需的信任结果 |
| `protected-paths.ts` |阻止写入特定路径 | `on("tool_call")` |
| `confirm-destructive.ts` |确认会话更改 | `on("session_before_switch")`、`on("session_before_fork")` |
| `dirty-repo-guard.ts` | 在 git 工作区有未提交修改时发出警告 | `on("session_before_*")`、`exec` |
| `input-transform.ts` | 转换用户输入 | `on("input")` |
| `input-transform-streaming.ts` | 感知流式状态的输入转换 | `on("input")`、`streamingBehavior` |
| `model-status.ts` |对模型变化做出反应 | `on("model_select")`、`setStatus` |
| `provider-payload.ts` |检查有效负载和提供者响应标头 | `on("before_provider_request")`、`on("after_provider_response")` |
| `system-prompt-header.ts` |显示系统提示信息 | `on("agent_start")`、`getSystemPrompt` |
| `claude-rules.ts` |从文件加载规则 | `on("session_start")`、`on("before_agent_start")` |
| `prompt-customizer.ts` | 使用 `systemPromptOptions` 添加上下文感知的工具准则 | `on("before_agent_start")`、`BuildSystemPromptOptions` |
| `file-trigger.ts` |文件监视器触发消息 | `sendMessage` |
| **压缩和会话** |||
| `custom-compaction.ts` |自定义压缩总结| `on("session_before_compact")` |
| `trigger-compact.ts` |手动触发压缩| `compact()` |
| `git-checkpoint.ts` | 每轮创建 Git 暂存检查点 | `on("turn_start")`、`on("session_before_fork")`、`exec` |
| `git-merge-and-resolve.ts` |获取、合并和解决冲突 | `on("agent_end")`、`exec`、`sendUserMessage` |
| `auto-commit-on-exit.ts` |退出时自动提交 | `on("session_shutdown")`、`exec` |
| **UI 组件** |||
| `status-line.ts` |页脚状态指示器 | `setStatus`，会话事件 |
| `working-indicator.ts` | 自定义流式响应期间的工作指示器 | `setWorkingIndicator`、`registerCommand` |
| `github-issue-autocomplete.ts` | 通过 `gh issue list` 预加载最近的开放 issue，在内置补全之上添加 `#1234` issue 补全 | `addAutocompleteProvider`、`on("session_start")`、`exec` |
| `custom-footer.ts` |完全替换页脚 | `registerCommand`、`setFooter` |
| `custom-header.ts` |替换启动头 | `on("session_start")`、`setHeader` |
| `modal-editor.ts` | Vim 风格的模态编辑器 | `setEditorComponent`、`CustomEditor` |
| `rainbow-editor.ts` |自定义编辑器样式 | `setEditorComponent` |
| `widget-placement.ts` |编辑器上方/下方的小组件 | `setWidget` |
| `overlay-test.ts` | 叠加层组件 | 带叠加选项的 `ui.custom` |
| `overlay-qa-tests.ts` | 全面的叠加层测试 | `ui.custom`、所有叠加选项 |
| `notify.ts` |简单的通知 | `ui.notify` |
| `timed-confirm.ts` |超时对话框 | `ui.confirm` 有超时/信号 |
| `mac-system-theme.ts` |自动切换主题 | `setTheme`、`exec` |
| **复杂的扩展** |||
| `plan-mode/` | 完整的计划模式实现 | 所有事件类型、`registerCommand`、`registerShortcut`、`registerFlag`、`setStatus`、`setWidget`、`sendMessage`、`setActiveTools` |
| `preset.ts` |可保存的预设（模型、工具、思维）| `registerCommand`、`registerShortcut`、`registerFlag`、`setModel`、`setActiveTools`、`setThinkingLevel`、`appendEntry` |
| `tools.ts` | 用 UI 启用或禁用工具 | `registerCommand`、`setActiveTools`、`SettingsList`、会话事件 |
| **远程和沙箱** |||
| `ssh.ts` | SSH 远程执行 | `registerFlag`、`on("user_bash")`、`on("before_agent_start")`、工具操作 |
| `interactive-shell.ts` |持久 shell 会话 | `on("user_bash")` |
| `sandbox/` |沙盒工具执行 |工具操作|
| `gondolin/` |将内置工具和`!`命令路由到 Gondolin micro-VM |工具操作、内置工具覆盖、`on("user_bash")` |
| `subagent/` |生成子代理 | `registerTool`、`exec` |
| **游戏** |||
| `snake.ts` |贪吃蛇游戏 | `registerCommand`，`ui.custom`，键盘处理 |
| `space-invaders.ts` |太空侵略者游戏| `registerCommand`、`ui.custom` |
| `doom-overlay/` | 在叠加层中运行 Doom | 带叠加选项的 `ui.custom` |
| **提供者** |||
| `custom-provider-anthropic/` |自定义 Anthropic 代理 | `registerProvider` |
| `custom-provider-gitlab-duo/` | GitLab Duo 集成 | `registerProvider` 与 OAuth |
| **消息与通信** |||
| `message-renderer.ts` |自定义消息渲染 | `registerMessageRenderer`、`sendMessage` |
| `entry-renderer.ts` | 仅用于 TUI 的自定义条目渲染 | `registerEntryRenderer`、`appendEntry` |
| `event-bus.ts` |扩展间事件 | `pi.events` |
| **会话元数据** |||
| `session-name.ts` |为选择器命名会话 | `setSessionName`、`getSessionName` |
| `bookmark.ts` | 为 `/tree` 中的条目添加书签 | `setLabel` |
| **杂项** |||
| `inline-bash.ts` |工具调用中的内联 bash | `on("tool_call")` |
| `bash-spawn-hook.ts` | 执行前调整 bash 命令、cwd 和 env | `createBashTool`、`spawnHook` |
| `with-deps/` | 带 npm 依赖的扩展 | 含 `package.json` 的包结构 |
