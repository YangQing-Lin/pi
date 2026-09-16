# 上下文压缩与分支摘要

LLM 的上下文窗口有限。对话过长时，Pi 会使用上下文压缩总结较早的内容，同时保留近期工作。本页同时介绍自动压缩和分支摘要。

**源文件**（[Pi](https://github.com/earendil-works/pi)）：

- [`packages/coding-agent/src/core/compaction/compaction.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)——自动压缩逻辑
- [`packages/coding-agent/src/core/compaction/branch-summarization.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)——分支摘要
- [`packages/coding-agent/src/core/compaction/utils.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/utils.ts)——共享工具（文件跟踪、序列化）
- [`packages/coding-agent/src/core/session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)——条目类型（`CompactionEntry`、`BranchSummaryEntry`）
- [`packages/coding-agent/src/core/extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts)——扩展事件类型

要查看项目中的 TypeScript 定义，请检查 `node_modules/@earendil-works/pi-coding-agent/dist/`。

## 概述

Pi 有两种摘要机制：

| 机制 | 触发条件 | 用途 |
|-----------|---------|---------|
| 上下文压缩 | 上下文超过阈值，或使用 `/compact` | 总结旧消息以释放上下文 |
| 分支摘要 | `/tree` 导航 | 切换分支时保留上下文 |

两者使用相同的结构化摘要格式，并以累积方式跟踪文件操作。上下文压缩和分支摘要请求会使用新的路由会话 ID；如果提供商支持，还会禁用提示缓存写入，因为这些一次性提示不太可能复用。

## 上下文压缩

### 触发时机

满足以下条件时会触发自动压缩：

```
contextTokens > contextWindow - reserveTokens
```

`reserveTokens` 默认为 16384 token（可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置），用于为 LLM 回复留出空间。

在多轮 agent 运行期间，Pi 会在工具完成且结果已追加后、开始下一次 assistant 回复前检查该阈值。如果超过阈值，Pi 会在同一次 agent 运行中压缩，然后使用摘要和保留的消息恢复执行。当已完成的工具批次会终止运行，且没有排队消息需要下一次回复时，Pi 会跳过这次轮次间检查。Pi 还会在新的用户提示前，以及底层 agent 运行结束后检查阈值。

也可以使用 `/compact [instructions]` 手动触发；可选指令用于指定摘要重点。

### 工作原理

1. **查找切分点**：从最新消息向后遍历并累加 token 估算，直到达到 `keepRecentTokens`（默认 20k，可在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置）
2. **提取消息**：收集从上一个保留边界（或会话开始）到切分点之间的消息
3. **生成摘要**：调用 LLM，以结构化格式生成摘要；如果存在上一个摘要，则将其作为迭代上下文传入
4. **追加条目**：保存包含摘要和 `firstKeptEntryId` 的 `CompactionEntry`
5. **重建上下文**：会话使用摘要和从 `firstKeptEntryId` 开始的消息，为下一个请求重建上下文

```
压缩前：

  条目：  0     1     2     3      4     5     6      7      8     9
        ┌─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┘
                └────────┬───────┘ └──────────────┬──────────────┘
                    待总结消息                   保留消息
                                   ↑
                          firstKeptEntryId（条目 4）

压缩后（追加新条目）：

  条目：  0     1     2     3      4     5     6      7      8     9     10
        ┌─────┬─────┬─────┬──────┬─────┬─────┬──────┬──────┬─────┬─────┬─────┐
        │ hdr │ usr │ ass │ tool │ usr │ ass │ tool │ tool │ ass │ tool│ cmp │
        └─────┴─────┴─────┴──────┴─────┴─────┴──────┴──────┴─────┴─────┴─────┘
               └──────────┬──────┘ └──────────────────────┬───────────────────┘
                    不发送给 LLM                        发送给 LLM
                                                         ↑
                                              从 firstKeptEntryId 开始

LLM 看到的内容：

  ┌────────┬─────────┬─────┬─────┬──────┬──────┬─────┬──────┐
  │ system │ summary │ usr │ ass │ tool │ tool │ ass │ tool │
  └────────┴─────────┴─────┴─────┴──────┴──────┴─────┴──────┘
       ↑         ↑      └─────────────────┬────────────────┘
    系统提示   来自 cmp          从 firstKeptEntryId 开始的消息
```

重复压缩时，被总结的区间从上一次压缩的保留边界（`firstKeptEntryId`）开始，而不是从压缩条目本身开始；如果在路径中找不到该保留条目，则回退到上一次压缩之后的条目。这样会在下一轮摘要中再次包含上次压缩后幸存的消息，从而保留它们。Pi 在写入新的 `CompactionEntry` 前，还会根据重建后的会话上下文重新计算 `tokensBefore`，因此 token 数会反映实际被替换的压缩前上下文。

### 拆分轮次

一个“轮次”从用户消息开始，包括之后的所有 assistant 回复和工具调用，直到下一条用户消息。通常，压缩会在轮次边界切分。

当单个轮次超过 `keepRecentTokens` 时，切分点会落在轮次中间的一条 assistant 消息上。这称为“拆分轮次”：

```
拆分轮次（一个超大轮次超过预算）：

  条目：  0     1     2      3     4      5      6     7      8
        ┌─────┬─────┬─────┬──────┬─────┬──────┬──────┬─────┬──────┐
        │ hdr │ usr │ ass │ tool │ ass │ tool │ tool │ ass │ tool │
        └─────┴─────┴─────┴──────┴─────┴──────┴──────┴─────┴──────┘
                ↑                                     ↑
         turnStartIndex = 1                  firstKeptEntryId = 7
                │                                     │
                └──── turnPrefixMessages (1-6) ───────┘
                                                      └── 保留 (7-8)

  isSplitTurn = true
  messagesToSummarize = []  （之前没有完整轮次）
  turnPrefixMessages = [usr, ass, tool, ass, tool, tool]
```

对于拆分轮次，Pi 会生成两个摘要并将其合并：

1. **历史摘要**：之前的上下文（如果有）
2. **轮次前缀摘要**：被拆分轮次的前半部分

### 切分点规则

有效的切分点包括：

- 用户消息
- Assistant 消息
- BashExecution 消息
- 自定义消息（custom_message、branch_summary）

绝不在工具结果处切分（工具结果必须与其工具调用保持在一起）。

### CompactionEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface CompactionEntry<T = unknown> {
  type: "compaction";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  firstKeptEntryId: string;
  tokensBefore: number;
  usage?: Usage;       // 生成摘要的 LLM 用量
  fromHook?: boolean;  // 由扩展提供时为 true（旧字段名）
  details?: T;         // 实现特定数据
}

// 默认压缩在 details 中使用此结构（来自 compaction.ts）：
interface CompactionDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

扩展可以在 `details` 中存储任何可 JSON 序列化的数据。默认压缩会跟踪文件操作，但自定义扩展实现可以使用自己的结构。生成的摘要和扩展提供的摘要会在可用时存储 LLM `usage`，从而将摘要工作计入会话总用量。

实现请参阅 [`prepareCompaction()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts) 和 [`compact()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/compaction.ts)。对于直接以编程方式生成摘要，`generateSummary()` 返回摘要文本，`generateSummaryWithUsage()` 返回 `{ text, usage }`。

## 分支摘要

### 触发时机

使用 `/tree` 导航到另一个分支时，Pi 会询问是否总结正在离开的工作。这会把左侧分支的上下文注入新分支。

### 工作原理

1. **查找共同祖先**：旧位置和新位置共享的最深节点
2. **收集条目**：从旧叶节点向后遍历到共同祖先
3. **按预算准备**：在 token 预算内包含消息（较新的优先）
4. **生成摘要**：调用 LLM 生成结构化摘要
5. **追加条目**：在导航位置保存 `BranchSummaryEntry`

```
导航前的树：

         ┌─ B ─ C ─ D（旧叶节点，即将放弃）
    A ───┤
         └─ E ─ F（目标）

共同祖先：A
待总结条目：B、C、D

带摘要导航后：

         ┌─ B ─ C ─ D
    A ───┤
         └─ E ─ F ─ [B、C、D 的摘要]（新叶节点）
```

### 累积文件跟踪

上下文压缩和分支摘要都会以累积方式跟踪文件。生成摘要时，Pi 从以下内容提取文件操作：

- 被总结消息中的工具调用
- 以前压缩或分支摘要的 `details`（如果有）

这意味着文件跟踪会跨多次压缩或嵌套分支摘要累积，保留已读取和已修改文件的完整历史。

### BranchSummaryEntry 结构

定义于 [`session-manager.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/session-manager.ts)：

```typescript
interface BranchSummaryEntry<T = unknown> {
  type: "branch_summary";
  id: string;
  parentId: string;
  timestamp: number;
  summary: string;
  fromId: string;      // 导航前所在的条目
  usage?: Usage;       // 生成摘要的 LLM 用量
  fromHook?: boolean;  // 由扩展提供时为 true（旧字段名）
  details?: T;         // 实现特定数据
}

// 默认分支摘要在 details 中使用此结构（来自 branch-summarization.ts）：
interface BranchSummaryDetails {
  readFiles: string[];
  modifiedFiles: string[];
}
```

与上下文压缩相同，扩展可以在 `details` 中存储自定义数据。

实现请参阅 [`collectEntriesForBranchSummary()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)、[`prepareBranchEntries()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts) 和 [`generateBranchSummary()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/branch-summarization.ts)。

## 摘要格式

上下文压缩和分支摘要使用相同的结构化格式：

```markdown
## Goal
[What the user is trying to accomplish]

## Constraints & Preferences
- [Requirements mentioned by user]

## Progress
### Done
- [x] [Completed tasks]

### In Progress
- [ ] [Current work]

### Blocked
- [Issues, if any]

## Key Decisions
- **[Decision]**: [Rationale]

## Next Steps
1. [What should happen next]

## Critical Context
- [Data needed to continue]

<read-files>
path/to/file1.ts
path/to/file2.ts
</read-files>

<modified-files>
path/to/changed.ts
</modified-files>
```

### 消息序列化

生成摘要前，消息通过 [`serializeConversation()`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/compaction/utils.ts) 序列化为文本：

```
[User]: What they said
[Assistant thinking]: Internal reasoning
[Assistant]: Response text
[Assistant tool calls]: read(path="foo.ts"); edit(path="bar.ts", ...)
[Tool result]: Output from tool
```

这样可以防止模型将其视为应当继续的对话。

序列化时，工具结果会截断到 2000 个字符。超出限制的内容会替换为标记，说明截断了多少字符。这样可以让摘要请求保持在合理的 token 预算内，因为工具结果（尤其来自 `read` 和 `bash` 的结果）通常是上下文大小的主要来源。

## 通过扩展自定义摘要

扩展可以拦截和自定义上下文压缩与分支摘要。事件类型定义请参阅 [`extensions/types.ts`](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts)。

### session_before_compact

在自动压缩或 `/compact` 前触发。可以取消或提供自定义摘要。请在类型文件中参阅 `SessionBeforeCompactEvent` 和 `CompactionPreparation`。

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, reason, willRetry, signal } = event;

  // preparation.messagesToSummarize - 待总结消息
  // preparation.turnPrefixMessages - 拆分轮次前缀（如果 isSplitTurn）
  // preparation.previousSummary - 上一次压缩摘要
  // preparation.fileOps - 提取的文件操作
  // preparation.tokensBefore - 压缩前的上下文 token
  // preparation.firstKeptEntryId - 保留消息开始的位置
  // preparation.settings - 压缩设置

  // branchEntries - 当前分支上的所有条目（用于自定义状态）
  // reason - "manual"（/compact）、"threshold" 或 "overflow"
  // willRetry - 压缩后是否重试被中止的轮次（溢出恢复）
  // signal - AbortSignal（传递给 LLM 调用）

  // 取消：
  return { cancel: true };

  // 自定义摘要：
  return {
    compaction: {
      summary: "Your summary...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      // usage: summaryResponse.usage, // 可选；计入会话总用量
      details: { /* 自定义数据 */ },
    }
  };
});
```

#### 将消息转换为文本

要使用自己的模型生成摘要，请使用 `serializeConversation` 将消息转换为文本：

```typescript
import { convertToLlm, serializeConversation } from "@earendil-works/pi-coding-agent";

pi.on("session_before_compact", async (event, ctx) => {
  const { preparation } = event;

  // 将 AgentMessage[] 转换为 Message[]，然后序列化为文本
  const conversationText = serializeConversation(
    convertToLlm(preparation.messagesToSummarize)
  );
  // 返回：
  // [User]: message text
  // [Assistant thinking]: thinking content
  // [Assistant]: response text
  // [Assistant tool calls]: read(path="..."); bash(command="...")
  // [Tool result]: output text

  // 现在发送给你的模型生成摘要
  const { summary, usage } = await myModel.summarize(conversationText);

  return {
    compaction: {
      summary,
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
      usage,
    }
  };
});
```

使用其他模型的完整示例请参阅 [custom-compaction.ts](../examples/extensions/custom-compaction.ts)。

### session_compact_failed

手动或自动压缩失败或中止时触发。对于需要将 `session_before_compact` 尝试与最终结果配对的遥测扩展，此事件很有用。

```typescript
pi.on("session_compact_failed", async (event, ctx) => {
  const { reason, errorMessage, aborted, willRetry, fromExtension } = event;
  // reason - "manual"（/compact）、"threshold" 或 "overflow"
  // errorMessage - 非中止失败时存在
  // aborted - 取消/中止的压缩为 true
  // willRetry - 压缩后是否会重试被中止的轮次
  // fromExtension - 是否正在使用扩展提供的压缩内容
});
```

### session_before_tree

在 `/tree` 导航前触发。无论用户是否选择生成摘要都会触发。可以取消导航或提供自定义摘要。

```typescript
pi.on("session_before_tree", async (event, ctx) => {
  const { preparation, signal } = event;

  // preparation.targetId - 导航目标
  // preparation.oldLeafId - 当前位置（即将放弃）
  // preparation.commonAncestorId - 共同祖先
  // preparation.entriesToSummarize - 原本要总结的条目
  // preparation.userWantsSummary - 用户是否选择生成摘要

  // 完全取消导航：
  return { cancel: true };

  // 提供自定义摘要（仅在 userWantsSummary 为 true 时使用）：
  if (preparation.userWantsSummary) {
    return {
      summary: {
        summary: "Your summary...",
        // usage: summaryResponse.usage, // 可选；计入会话总用量
        details: { /* 自定义数据 */ },
      }
    };
  }
});
```

请在类型文件中参阅 `SessionBeforeTreeEvent` 和 `TreePreparation`。

## 设置

在 `~/.pi/agent/settings.json` 或 `<project-dir>/.pi/settings.json` 中配置上下文压缩：

```json
{
  "compaction": {
    "enabled": true,
    "reserveTokens": 16384,
    "keepRecentTokens": 20000
  }
}
```

| 设置 | 默认值 | 说明 |
|---------|---------|-------------|
| `enabled` | `true` | 启用自动压缩 |
| `reserveTokens` | `16384` | 为 LLM 回复预留的 token |
| `keepRecentTokens` | `20000` | 保留的近期 token（不总结） |

使用 `"enabled": false` 禁用自动压缩。仍可使用 `/compact` 手动压缩。
