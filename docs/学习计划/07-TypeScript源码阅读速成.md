# TypeScript 源码阅读速成：面向 Python 开发者的五个必修概念

## 这份文档解决什么问题

这不是一门完整的 TypeScript 或前端开发课程。它面向熟悉 Python、但 TypeScript 基础较弱，希望尽快阅读 Pi Agent 源码的开发者。

目标是在 **2–3 小时**内建立足够的理解框架，补齐最容易阻塞源码阅读的五个主题：

1. 判别联合与 narrowing（类型收窄）
2. 泛型
3. `AsyncIterable`
4. ESM 顶层 import
5. `AbortSignal`

每个主题都会映射到熟悉的 Python 概念：

| TypeScript | Python 对照 | 需要特别注意的差异 |
| --- | --- | --- |
| 判别联合与 narrowing | Pydantic 判别联合、`match`、静态类型收窄 | TypeScript narrowing 主要发生在静态检查阶段；Pydantic 负责运行时校验 |
| 泛型、结构化类型 | `TypeVar`、`Generic`、`Protocol` | TypeScript 默认采用结构化类型；Python 只有 `Protocol` 明确表达这种关系 |
| `AsyncIterable<T>` | `AsyncIterator[T]`、async generator、`async for` | 两者都表示异步序列，但具体取消、关闭和背压语义取决于实现 |
| ESM 顶层 import | Python module import | ESM 区分值 import 和 type-only import，并受 Node.js 模块解析规则约束 |
| `AbortSignal` | `asyncio` task cancellation、AnyIO cancel scope | `AbortSignal` 是协作式通知，不会自动终止任意任务或回滚副作用 |

学完后，你应该能读懂下面这条主线：

```text
ESM import 连接模块
  ↓
泛型约束事件流和工具的输入/输出类型
  ↓
AsyncIterable 持续产出流式事件
  ↓
判别联合 + narrowing 识别每种事件
  ↓
AbortSignal 将取消意图沿调用链传递
```

## 先建立一个大的理解框架

### 1. JavaScript 是运行时，TypeScript 是静态检查层

TypeScript 在 JavaScript 语法之上增加类型系统。多数类型信息在运行前会被擦除：

```typescript
function add(left: number, right: number): number {
  return left + right;
}
```

运行时看到的核心逻辑接近：

```javascript
function add(left, right) {
  return left + right;
}
```

因此需要始终区分两个问题：

- **静态阶段**：类型检查器能否证明代码安全？
- **运行阶段**：真实输入是否经过校验，异步任务和副作用实际如何执行？

这与 Python 类型注解很像：`mypy` 或 Pyright 能检查类型，但外部 JSON 并不会因为写了注解就自动变成可信对象。Python 通常使用 Pydantic 做运行时校验；TypeScript 项目同样需要 TypeBox、Zod 或手写校验处理不可信输入。

### 2. Pi 使用严格、面向 Node.js 的 TypeScript

当前仓库的关键配置包括：

- 根 `package.json` 使用 `"type": "module"`，表示采用 ESM。
- 根 `tsconfig.json` 使用 `module: "NodeNext"` 和 `moduleResolution: "NodeNext"`。
- `tsconfig.base.json` 启用 `strict: true` 和 `erasableSyntaxOnly: true`。
- 项目允许源码 import 使用 `.ts` 扩展名，并在输出时改写相对扩展名。

阅读时可以把它理解为：

```text
严格静态类型
+ Node.js ESM 模块
+ 尽量只使用可直接擦除的 TypeScript 语法
```

这也是项目禁止内联动态 import、要求 import 位于文件顶部的背景。

### 3. TypeScript 默认是结构化类型系统

Python 日常代码经常通过“对象属于哪个类”理解类型。TypeScript 更关注“对象是否具有所需结构”：

```typescript
interface Named {
  name: string;
}

const user = { name: "Luke", age: 30 };
const named: Named = user;
```

`user` 没有显式声明 `implements Named`，但它包含 `name: string`，因此可以赋给 `Named`。

Python 中最接近的是 `Protocol`：

```python
from typing import Protocol


class Named(Protocol):
    name: str


def display(value: Named) -> str:
    return value.name
```

先理解结构化类型，后面的接口、泛型工具和事件类型会容易很多。

## 阅读源码前的最小语法包

不需要先学完整语法，但以下符号必须能一眼识别：

| 语法 | 含义 | Python 近似写法 |
| --- | --- | --- |
| `const value = ...` | 变量不能重新赋值；对象内部仍可能可变 | 没有完全等价语法；接近“约定不重新绑定” |
| `let value = ...` | 变量可以重新赋值 | 普通变量 |
| `name: string` | 类型注解 | `name: str` |
| `value?: string` | 可选属性，可能不存在 | `value: str | None` 只近似；Python 字段通常仍然存在 |
| `string | null` | 值可以是字符串或 `null` | `str | None` |
| `T[]`、`Array<T>` | `T` 的数组 | `list[T]` |
| `Promise<T>` | 稍后得到一个 `T` | `Awaitable[T]` / coroutine |
| `AsyncIterable<T>` | 随时间异步产生多个 `T` | `AsyncIterable[T]` |
| `type X = A | B` | 类型别名或联合类型 | `type X = A | B`（Python 3.12） |
| `interface X { ... }` | 描述对象必须具备的结构 | `Protocol`、`TypedDict` 或抽象基类，取决于用途 |
| `value as X` | 告诉检查器“把它当作 X” | `cast(X, value)` |
| `unknown` | 类型未知，使用前必须检查 | 接近 `object`，比 `Any` 更安全 |
| `any` | 关闭这部分类型检查 | `Any` |

三个特别容易混淆的概念：

- `undefined`：没有值、属性不存在或函数没有返回值。
- `null`：显式的空值。
- 可选属性 `x?: T`：对象可能根本没有 `x`；这不完全等于对象一定有 `x`、但值是 `undefined`。

阅读源码时优先跟踪数据结构和控制流，不要在第一遍停下来研究装饰器、前端 DOM、CSS、React 或复杂类型体操。

## 主题一：判别联合与 narrowing

### 问题：同一个变量可能表示多种事件

Agent 系统会产生多种事件。每种事件有共同的 `type` 字段，但携带的数据不同：

```typescript
type AgentEvent =
  | { type: "text_delta"; delta: string }
  | { type: "tool_end"; toolName: string; isError: boolean };
```

这里的 `type` 是**判别字段**，字符串字面量 `"text_delta"` 和 `"tool_end"` 是不同的 tag。整个类型称为判别联合（discriminated union，也常称 tagged union）。

如果直接访问 `event.delta`，TypeScript 会报错，因为 `tool_end` 事件没有该字段。必须先判断事件种类：

```typescript
function describe(event: AgentEvent): string {
  if (event.type === "text_delta") {
    return event.delta;
  }

  return `${event.toolName}: ${event.isError ? "failed" : "ok"}`;
}
```

在 `if` 分支内，TypeScript 根据 `event.type` 将类型从 `AgentEvent` 缩小为 `{ type: "text_delta"; delta: string }`。这个过程就是 narrowing。

### `switch` 与穷尽性检查

事件较多时通常使用 `switch`：

```typescript
function assertNever(value: never): never {
  throw new Error(`Unhandled event: ${JSON.stringify(value)}`);
}

function describe(event: AgentEvent): string {
  switch (event.type) {
    case "text_delta":
      return event.delta;
    case "tool_end":
      return `${event.toolName}: ${event.isError ? "failed" : "ok"}`;
    default:
      return assertNever(event);
  }
}
```

`never` 表示“不可能出现的值”。当联合类型的所有成员都已处理，`default` 中的 `event` 会被收窄为 `never`。以后新增事件但忘记增加 `case` 时，类型检查会在 `assertNever(event)` 处失败。

### 常见 narrowing 手段

| 写法 | 适用对象 |
| --- | --- |
| `value.type === "..."` | 判别联合，Pi 源码中最常见 |
| `typeof value === "string"` | JavaScript 原始类型 |
| `"field" in value` | 按属性是否存在区分对象 |
| `value instanceof Error` | 按运行时构造函数区分实例 |
| `value !== null` | 排除 `null` |
| 自定义 `value is SomeType` 函数 | 复用复杂类型判断 |

不要无脑使用 truthiness narrowing：

```typescript
if (value) {
  // value 不仅排除了 null/undefined，也排除了 ""、0、false。
}
```

如果空字符串或零是合法业务值，应显式检查 `value !== undefined` 或 `value !== null`。

### 映射到 Pydantic 判别联合

Python/Pydantic v2 的运行时模型可以写成：

```python
from typing import Annotated, Literal

from pydantic import BaseModel, Field, TypeAdapter


class TextDelta(BaseModel):
    type: Literal["text_delta"]
    delta: str


class ToolEnd(BaseModel):
    type: Literal["tool_end"]
    tool_name: str
    is_error: bool


AgentEvent = Annotated[
    TextDelta | ToolEnd,
    Field(discriminator="type"),
]

adapter = TypeAdapter(AgentEvent)
event = adapter.validate_python({"type": "text_delta", "delta": "hello"})

match event:
    case TextDelta(delta=delta):
        print(delta)
    case ToolEnd(tool_name=name, is_error=is_error):
        print(name, is_error)
```

对应关系如下：

```text
TS 字符串字面量 type: "text_delta"
  ↔ Python Literal["text_delta"]

TS 联合 A | B
  ↔ Python A | B

TS 检查 event.type 后静态 narrowing
  ↔ Pydantic 先按 discriminator 做运行时解析，类型检查器再结合 isinstance/match 收窄
```

关键差异：TypeScript 类型声明本身不会验证网络 JSON；Pydantic 的 `validate_python()` 会在运行时拒绝错误输入。

### 在 Pi 源码中的落点

- `packages/ai/src/types.ts`：`AssistantMessageEvent` 是流式消息事件联合。
- `packages/agent/src/types.ts`：`AgentEvent` 是 agent 生命周期、消息和工具事件联合。
- `packages/agent/src/agent-loop.ts`：`switch (event.type)` 消费流式事件。
- `packages/coding-agent/src/core/agent-session.ts`：大量 `event.type === ...` 分支将事件投影到产品层。

第一遍阅读时，只做一件事：找到联合类型定义，然后为每个 `case` 标出该分支新增了哪些可访问字段。

## 主题二：泛型

### 问题：希望复用逻辑，又不想丢失具体类型

如果一个函数接收任意类型并返回同一类型，可以用泛型表达输入与输出的关系：

```typescript
function identity<T>(value: T): T {
  return value;
}

const name = identity("pi"); // 推断为 string
const count = identity(3);   // 推断为 number
```

`T` 不是某个具体类型，而是一个类型参数。调用函数时，TypeScript 通常可以从参数推断它。

没有泛型时只能写 `unknown` 或 `any`：

- `unknown` 安全，但丢失了“返回类型与输入类型相同”的关系。
- `any` 不仅丢失关系，还会关闭类型检查。

### 泛型约束与默认值

```typescript
interface Identified {
  id: string;
}

function getId<T extends Identified>(value: T): string {
  return value.id;
}
```

`T extends Identified` 的意思不是面向对象继承，而是“`T` 至少满足 `Identified` 的结构”。

接口也可以使用泛型和默认值：

```typescript
interface ToolResult<TDetails = unknown> {
  content: string;
  details: TDetails;
}

const result: ToolResult<{ durationMs: number }> = {
  content: "done",
  details: { durationMs: 12 },
};
```

`TDetails = unknown` 表示调用方没有指定类型参数时使用 `unknown`。

### 映射到 `TypeVar` 与 `Protocol`

Python 泛型函数使用 `TypeVar` 表达相同关系：

```python
from collections.abc import Sequence
from typing import TypeVar


T = TypeVar("T")


def first(items: Sequence[T]) -> T:
    return items[0]
```

TypeScript interface 的结构化能力可映射到 Python `Protocol`：

```python
from typing import Protocol, TypeVar


T = TypeVar("T")
T_co = TypeVar("T_co", covariant=True)


class Loader(Protocol[T_co]):
    def load(self) -> T_co: ...


def load_value(loader: Loader[T]) -> T:
    return loader.load()
```

记忆方式：

```text
TypeVar：在多个位置之间保存“这是同一个类型”的关系
Protocol：声明“对象只要具有这些成员，就满足接口”
```

### 阅读复杂泛型的固定顺序

遇到类似下面的定义时，不要从左到右一次读完：

```typescript
interface AgentTool<
  TParameters extends TSchema = TSchema,
  TDetails = unknown,
> {
  execute(params: Static<TParameters>): Promise<ToolResult<TDetails>>;
}
```

按四步拆解：

1. `AgentTool<...>`：这是可配置类型的工具接口。
2. `TParameters extends TSchema`：参数类型必须是某种 schema。
3. `= TSchema`：省略类型参数时的默认值。
4. `TDetails` 从 `execute()` 的结果中传出，保持具体详情类型。

上面的教学示例让 `TDetails` 默认取 `unknown`，便于保留类型安全。当前源码中的 `AgentTool` 默认使用 `TDetails = any`，这会放宽相应位置的检查。需要注意：`strict: true` 会启用一组严格检查，但不会禁止开发者显式写出 `any`；阅读时仍要区分 `unknown` 与 `any`。

第一遍不需要掌握条件类型、映射类型、协变/逆变或高阶类型。只要能回答“这个类型参数从哪里进入、从哪里出来、受什么约束”即可。

### 在 Pi 源码中的落点

- `packages/ai/src/types.ts`：`Tool<TParameters>`、`Model<TApi>`。
- `packages/agent/src/types.ts`：`AgentTool<TParameters, TDetails>`、`AgentToolResult<T>`。
- `packages/ai/src/utils/event-stream.ts`：`EventStream<T, R = T>` 区分流中事件类型 `T` 与最终结果 `R`。
- `packages/coding-agent/src/core/session-manager.ts`：`CompactionEntry<T = unknown>` 等泛型条目保存实现特定详情。

## 主题三：`AsyncIterable`

### 先区分“一次结果”和“多个异步结果”

| 类型 | 产生结果的方式 | Python 对照 |
| --- | --- | --- |
| `T` | 现在有一个结果 | 普通值 |
| `Promise<T>` | 将来有一个结果 | awaitable / coroutine |
| `Iterable<T>` | 同步产生多个结果 | iterable / generator |
| `AsyncIterable<T>` | 随时间异步产生多个结果 | async iterable / async generator |

LLM 流式响应不是“等待结束后得到一个大字符串”，而是连续产生 `start`、`text_delta`、`toolcall_delta`、`done` 等事件，因此适合 `AsyncIterable<Event>`。

### 消费异步序列

```typescript
type StreamEvent =
  | { type: "text_delta"; delta: string }
  | { type: "done" };

async function consume(events: AsyncIterable<StreamEvent>): Promise<string> {
  let text = "";

  for await (const event of events) {
    if (event.type === "text_delta") {
      text += event.delta;
    }
  }

  return text;
}
```

`for await...of` 每次等待一个新元素，直到迭代器返回 `done: true` 或抛出异常。

### 使用 async generator 生产异步序列

```typescript
async function* generateEvents(): AsyncGenerator<StreamEvent> {
  yield { type: "text_delta", delta: "hel" };
  await Promise.resolve();
  yield { type: "text_delta", delta: "lo" };
  yield { type: "done" };
}
```

Python 对应写法：

```python
import asyncio
from collections.abc import AsyncIterator
from typing import Literal, TypedDict


class TextDelta(TypedDict):
    type: Literal["text_delta"]
    delta: str


class Done(TypedDict):
    type: Literal["done"]


StreamEvent = TextDelta | Done


async def generate_events() -> AsyncIterator[StreamEvent]:
    yield {"type": "text_delta", "delta": "hel"}
    await asyncio.sleep(0)
    yield {"type": "text_delta", "delta": "lo"}
    yield {"type": "done"}


async def consume() -> str:
    text = ""
    async for event in generate_events():
        if event["type"] == "text_delta":
            text += event["delta"]
    return text
```

### `AsyncIterable` 不等于“后台线程”

它只定义异步迭代协议：对象能返回 `AsyncIterator`，后者的 `next()` 会返回 `Promise<IteratorResult<T>>`。它不保证：

- 生产者运行在其他线程；
- 队列有界；
- 自动提供背压；
- 消费者退出时一定取消上游；
- 异常或资源一定自动清理。

这些行为由具体实现决定。

Pi 的 `EventStream<T, R>` 使用队列连接推送型生产者和拉取型异步消费者：

- `push(event)` 将事件交给等待中的消费者，或放入队列。
- `[Symbol.asyncIterator]()` 让对象可被 `for await...of` 消费。
- `result()` 返回最终聚合结果。

### 在 Pi 源码中的落点

- `packages/ai/src/utils/event-stream.ts`：`EventStream<T, R>` 的完整实现。
- `packages/agent/src/agent-loop.ts`：`for await (const event of response)` 消费模型流。
- `packages/ai/src/api/*`：不同提供商将上游 SSE/SDK 流转换为统一事件。

阅读时画出两个方向：

```text
Provider → push(event) → queue → for await 消费者
Provider 完成 → 最终事件 → result() Promise
```

## 主题四：ESM 顶层 import

### 什么是 ESM

ESM（ECMAScript Modules）是 JavaScript 标准模块系统，核心语法是 `import` 和 `export`：

```typescript
import type { AgentEvent } from "./types.ts";
import { runAgentLoop } from "./agent-loop.ts";

export { runAgentLoop };
export type { AgentEvent };
```

“顶层 import”是指 import 声明放在模块顶层，而不是函数或条件分支内部。静态 import 让运行时、类型检查器和工具可以在执行前分析依赖图。

### 值 import 与 type-only import

```typescript
import { runAgentLoop } from "./agent-loop.ts";
import type { AgentEvent } from "./types.ts";
```

- `runAgentLoop` 是运行时值，生成的 JavaScript 仍需要加载它。
- `AgentEvent` 只在类型位置使用，`import type` 会在类型擦除后消失。

这与 Python 的普通 import 不完全相同。Python 类型通常仍是运行时对象；如果只供类型检查使用，经常借助 `TYPE_CHECKING`：

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from app.types import AgentEvent
```

### 为什么路径中常出现 `.ts`

当前 Pi 仓库使用 Node 风格 ESM，并启用了：

- `allowImportingTsExtensions`
- `rewriteRelativeImportExtensions`
- `moduleResolution: "NodeNext"`

因此源码中可以写：

```typescript
import { runAgentLoop } from "./agent-loop.ts";
```

不要机械地把其他项目的 import 规则套到这里。是否写 `.ts`、`.js` 或省略扩展名，取决于运行时、构建工具和 `tsconfig`。

### 与 Python module import 的共同点和差异

共同点：

- 模块首次加载时会执行顶层代码。
- 模块通常会缓存。
- 循环依赖可能产生部分初始化状态或难以理解的行为。
- 顶层 import 能让依赖关系更清晰。

主要差异：

- ESM 的静态 import 在语法层面只能位于模块顶层。
- 动态加载使用 `import()`，但本项目规则禁止内联动态 import，应使用顶层 import。
- `import type` 没有运行时依赖。
- ESM 的路径和包导出受 Node.js `package.json`、`exports` 与模块解析策略影响。

### 现在不需要学习什么

为了阅读 Pi 后端源码，暂时不需要先学：

- Webpack、Vite、Rollup 等前端 bundler
- React/Vue 组件系统
- DOM 和浏览器事件
- CommonJS 与所有历史互操作边角

先能回答“这个 symbol 从哪个模块导入、它是值还是类型、运行时是否真的加载”即可。

### 在 Pi 源码中的落点

- `packages/agent/src/agent.ts`：同一文件同时使用值 import 和 `import type`。
- 根 `package.json`：声明 `"type": "module"`。
- `tsconfig.json`、`tsconfig.base.json`：定义 Node 模块解析和可擦除语法约束。

## 主题五：`AbortSignal`

### 问题：如何把“请停止”沿异步调用链传下去

`AbortController` 负责发出取消请求，`AbortSignal` 负责只读地传播取消状态：

```typescript
const controller = new AbortController();
const signal = controller.signal;

controller.abort(new Error("user cancelled"));

console.log(signal.aborted); // true
```

通常由上层创建 controller，只把 signal 传给下层：

```typescript
async function run(signal: AbortSignal): Promise<void> {
  signal.throwIfAborted();
  // 将同一个 signal 继续传给 fetch、模型流或工具执行。
}
```

下层不能通过 signal 调用 `abort()`，从而保持取消所有权清晰。

### 协作式取消，而不是强制终止

调用 `controller.abort()` 不会自动：

- 停止任意 JavaScript 计算；
- 杀死没有接入 signal 的子进程；
- 回滚已经写入的文件；
- 撤销已经发出的网络请求或外部副作用。

操作必须主动检查 `signal.aborted`、调用 `signal.throwIfAborted()`，或监听 `abort` 事件，并把 signal 传给支持它的下游 API。

一个可运行且正确清理监听器的延迟函数：

```typescript
function delay(milliseconds: number, signal: AbortSignal): Promise<void> {
  return new Promise((resolve, reject) => {
    signal.throwIfAborted();

    const onAbort = (): void => {
      clearTimeout(timeout);
      signal.removeEventListener("abort", onAbort);
      reject(signal.reason);
    };

    const timeout = setTimeout(() => {
      signal.removeEventListener("abort", onAbort);
      resolve();
    }, milliseconds);

    signal.addEventListener("abort", onAbort, { once: true });

    // 处理 throwIfAborted() 与监听器注册之间发生取消的竞态。
    if (signal.aborted) onAbort();
  });
}
```

### 映射到 `asyncio` cancellation

`asyncio` 通常取消具体的 `Task`：

```python
import asyncio


async def worker() -> None:
    try:
        while True:
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        # 做必要清理，然后继续向上传播取消。
        raise


async def main() -> None:
    task = asyncio.create_task(worker())
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        pass
```

区别是：

- `AbortSignal` 是普通的取消状态和事件通道，函数是否响应取决于实现。
- `Task.cancel()` 会安排在 coroutine 的下一次执行机会抛出 `CancelledError`。
- 两者都无法可靠撤销已经完成的外部副作用。

不要吞掉 `CancelledError`。如果确实捕获它做清理，通常应再次 `raise`。

### 映射到 AnyIO cancellation

AnyIO 通过 cancel scope 表达结构化取消：

```python
import anyio


async def worker() -> None:
    while True:
        await anyio.sleep(1)


async def main() -> None:
    async with anyio.create_task_group() as task_group:
        task_group.start_soon(worker)
        task_group.cancel_scope.cancel()
```

AnyIO 的 scope 可以嵌套，并会影响其内部任务。它采用 level cancellation：处于已取消 scope 中的任务在后续 yield point 仍会继续收到取消异常；这与 `asyncio` 通常所说的 edge cancellation 不完全相同。

概念映射可以这样记：

```text
AbortController
  ↔ 持有取消权的上层代码

AbortSignal
  ↔ 沿调用链传递的取消上下文

AbortSignal.aborted / throwIfAborted()
  ↔ 在 await/checkpoint 附近观察取消

AnyIO cancel scope
  ↔ 有词法范围和父子传播关系的取消边界
```

### 副作用边界比“抛出取消错误”更重要

假设文件写入分为三步：创建目录、写文件、返回成功。取消可能发生在任意两个步骤之间。即使最终抛出“已取消”，文件也可能已经写入。

Pi 的 `packages/coding-agent/src/core/tools/write.ts` 没有在 abort 监听器中立即释放文件 mutation queue，而是在每个 `await` 后检查取消。原因是：底层文件操作可能仍会完成，过早释放队列可能让后续写入与未结束操作并发，破坏顺序保证。

这是阅读取消代码时最重要的问题：

```text
取消发生时，哪些动作尚未开始？
哪些动作正在进行且无法回滚？
锁、队列和资源何时才能安全释放？
调用方看到“取消”时，外部世界可能处于什么状态？
```

### 在 Pi 源码中的落点

- `packages/agent/src/types.ts`：工具 `execute()` 接收可选 `AbortSignal`。
- `packages/agent/src/agent.ts`：signal 沿上下文转换、工具 hook 和下一轮准备逻辑传递。
- `packages/coding-agent/src/core/tools/bash.ts`：监听取消并终止子进程。
- `packages/coding-agent/src/core/tools/write.ts`：在文件 mutation queue 内按异步边界检查取消。
- `packages/ai/src/utils/abort.ts`：封装 promise 与 signal 的竞速和组合逻辑。

## 五个主题如何在真实 Agent 流程中连起来

阅读 `packages/agent/src/agent-loop.ts` 时，可以使用下面的解释框架：

```text
1. 文件顶部通过 ESM import 引入模型、消息、工具和流函数。
2. AgentTool<TParameters, TDetails> 用泛型保持 schema、实参和结果详情之间的类型关系。
3. 模型调用返回可异步迭代的流。
4. for await...of 逐个读取 AssistantMessageEvent。
5. switch (event.type) 对判别联合做 narrowing。
6. signal 被传给模型和工具；上层 abort 时，下游协作停止。
```

如果这六句都能在源码中找到对应位置，就已经具备阅读主循环的最低 TypeScript 能力。

## 2–3 小时学习安排

### 0:00–0:20：建立语言与运行时地图

完成：

- 阅读“大的理解框架”和“最小语法包”。
- 打开 `package.json`、`tsconfig.json`、`tsconfig.base.json`。
- 确认 ESM、`strict`、NodeNext 和类型擦除的含义。

通过标准：能解释 TypeScript 类型为什么不能替代运行时输入校验。

### 0:20–0:50：判别联合与 narrowing

完成：

- 手写一个包含三种事件的联合。
- 用 `switch` 处理，并加入 `assertNever()`。
- 再增加第四种事件，观察遗漏分支时的类型错误。
- 写出对应的 Pydantic v2 判别联合。

通过标准：看到 `event.type` 时，能立即想到“这个分支中类型会变窄”。

### 0:50–1:15：泛型与结构化类型

完成：

- 解释 `EventStream<T, R = T>` 中 `T` 与 `R` 的职责。
- 解释 `AgentTool<TParameters, TDetails>` 的类型参数如何流动。
- 写一个 `TypeVar` 泛型函数和一个 Python `Protocol`。

通过标准：能沿函数参数和返回值追踪同一个类型参数，而不是把 `<T>` 当作噪声。

### 1:15–1:45：`AsyncIterable`

完成：

- 写一个产生三条事件的 async generator。
- 用 `for await...of` 消费。
- 对照 Python async generator 和 `async for`。
- 阅读 `EventStream<T, R>` 的队列、等待者和完成逻辑。

通过标准：能区分 `Promise<T>` 与 `AsyncIterable<T>`，并解释流何时结束。

### 1:45–2:05：ESM 顶层 import

完成：

- 在 `packages/agent/src/agent.ts` 中区分值 import 与 `import type`。
- 找到一个相对路径 import 和一个 workspace package import。
- 说明为什么本项目使用 `.ts` 扩展名。

通过标准：能判断某个 import 是否产生运行时依赖。

### 2:05–2:35：`AbortSignal`

完成：

- 写出 controller/signal 的所有权关系。
- 找到 `bash.ts` 的 abort listener 和 `write.ts` 的边界检查。
- 对照 `asyncio.Task.cancel()` 和 AnyIO cancel scope。
- 写出一个“取消后外部副作用结果未知”的例子。

通过标准：不再把 `abort()` 理解为强制终止或事务回滚。

### 2:35–3:00：沿 Pi 主循环做一次闭环

按顺序阅读：

1. `packages/ai/src/types.ts` 中的 `AssistantMessageEvent`
2. `packages/ai/src/utils/event-stream.ts`
3. `packages/agent/src/types.ts` 中的 `AgentTool` 与 `AgentEvent`
4. `packages/agent/src/agent-loop.ts` 中消费模型流的循环
5. `packages/coding-agent/src/core/tools/write.ts` 的取消处理

产出一页笔记，必须回答：

- 流中每种事件由哪个判别字段区分？
- 哪些类型参数把 schema 与实际参数连接起来？
- 事件如何从 producer 到达 `for await`？
- signal 从哪里创建、传到哪里、由谁观察？
- 取消发生在写入后，调用方能否假设文件未改变？

如果只有 2 小时，保留前五段学习，把最后 25 分钟的源码闭环移到第二天；不要删除取消语义。

## 源码阅读速查命令

以下命令只读取文件，不会修改仓库：

```bash
# 判别联合及 narrowing
rg -n 'export type (AssistantMessageEvent|AgentEvent)|switch \(event\.type\)' \
  packages/ai/src packages/agent/src

# 泛型工具和事件流
rg -n 'interface AgentTool<|class EventStream<' \
  packages/agent/src packages/ai/src

# 异步迭代
rg -n 'AsyncIterable|for await \(' \
  packages/ai/src packages/agent/src

# 取消传播
rg -n 'AbortSignal|throwIfAborted|addEventListener\("abort"' \
  packages/ai/src packages/agent/src packages/coding-agent/src/core/tools

# ESM 与 type-only import
rg -n '^import type|^import ' packages/agent/src/agent.ts
```

## 常见误区

### 误区 1：TypeScript 类型已经验证了外部 JSON

错误。类型通常在运行时被擦除。外部输入仍需 schema 校验。

### 误区 2：`as SomeType` 会转换或验证数据

错误。类型断言主要是在说服编译器，不会自动增加字段或检查 JSON。

### 误区 3：泛型只是为了减少重复代码

不完整。泛型更重要的用途是保留多个位置之间的类型关系，例如“输入 schema 决定 execute 参数类型”。

### 误区 4：`AsyncIterable` 会自动并发或提供无限队列

错误。它只定义异步迭代协议。并发、缓冲、背压和资源清理由实现决定。

### 误区 5：`AbortSignal` 等同于 Python `Task.cancel()`

不等同。两者都用于协作取消，但通知方式、传播边界和异常模型不同。

### 误区 6：收到取消就代表没有产生副作用

错误。文件、进程、网络和工具调用可能已经部分执行。必须单独设计幂等、队列、事务或恢复策略。

### 误区 7：必须先学完整前端 TypeScript 才能读 Pi

错误。Pi 的核心阅读路径是 Node.js、事件流、类型建模和异步控制，不依赖完整前端知识体系。

## 自测题

不看答案先口述：

1. `Promise<Event>` 和 `AsyncIterable<Event>` 有什么区别？
2. 为什么检查 `event.type === "text_delta"` 后可以访问 `event.delta`？
3. `T extends TSchema = TSchema` 中的 `extends` 和 `=` 分别表示什么？
4. `import type` 会不会成为运行时依赖？
5. `controller.abort()` 是否能保证正在写入的文件保持不变？
6. Pydantic 判别联合与 TypeScript 判别联合最重要的差异是什么？
7. 为什么捕获 Python `CancelledError` 做清理后通常还要重新抛出？

参考答案：

1. 前者异步得到一个值；后者随时间异步产生多个值。
2. `type` 是联合的判别字段，控制流分析将对象收窄到对应成员。
3. `extends` 是类型约束，`=` 是默认类型参数。
4. 不会；它会随类型擦除而消失。
5. 不能；取消是协作式的，副作用可能已经开始或完成。
6. TypeScript 主要提供静态检查和控制流收窄；Pydantic 对真实输入执行运行时校验与解析。
7. 取消是控制流信号；吞掉它可能让上层误认为任务正常完成，并破坏结构化并发。

## 官方与优质资料

以下链接已于 2026-09-16 核验可访问。按“先读什么”排序，无需一次读完。

### TypeScript 与 JavaScript

- [TypeScript Handbook：Narrowing](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)：判别联合、控制流分析和类型守卫，主题一的主资料。
- [TypeScript Handbook：Generics](https://www.typescriptlang.org/docs/handbook/2/generics.html)：泛型函数、约束与泛型接口，主题二的主资料。
- [TypeScript Handbook：Modules Reference](https://www.typescriptlang.org/docs/handbook/modules/reference.html)：模块语法、Node.js 模块解析和类型声明导入。
- [MDN：`for await...of`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for-await...of)：异步迭代协议和消费语法。
- [MDN：`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal)：`aborted`、`reason`、`throwIfAborted()` 和组合方法。

### Python 对照资料

- [Pydantic：Unions / Discriminated Unions](https://docs.pydantic.dev/latest/concepts/unions/#discriminated-unions)：`Literal`、`Field(discriminator=...)` 和运行时错误行为。
- [Python `typing` 官方文档](https://docs.python.org/3/library/typing.html)：`TypeVar`、`Generic`、`Protocol` 和类型系统术语。
- [Python 语言参考：Asynchronous generator functions](https://docs.python.org/3/reference/expressions.html#asynchronous-generator-functions)：async generator 的协议定义。
- [Python `asyncio`：Task Cancellation](https://docs.python.org/3/library/asyncio-task.html#task-cancellation)：`Task.cancel()`、`CancelledError` 和清理原则。
- [AnyIO：Cancellation and timeouts](https://anyio.readthedocs.io/en/stable/cancellation.html)：cancel scope、level cancellation、shielding 和超时。

## 学完之后再学什么

只有在源码实际阻塞时，再按需补充：

1. `unknown`、类型守卫和外部 schema 校验
2. utility types：`Pick`、`Omit`、`Partial`、`Record`
3. 条件类型和 `infer`
4. Node.js stream、SSE 和 WebSocket
5. TypeBox schema 与静态类型推导

仍然不必先扩展到完整前端体系。下一步应回到[源码阅读与实验手册](./03-源码阅读与实验手册.md)，用真实调用链巩固这些概念。
