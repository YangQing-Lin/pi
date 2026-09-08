# Python 复刻实验工程

## 目标与边界

这个实验工程用于证明你已经理解 pi 的协议和运行时行为，不用于逐行翻译 TypeScript，也不在第一阶段追求通用 Agent 框架。

完成后应得到两类资产：

1. 一个网络隔离、确定性、严格类型的 Python 最小 Agent，用于复现 `pi-ai` 和 `pi-agent-core` 的关键不变量。
2. 一个健壮的 Python RPC client，用于验证“Python control plane + pi Node worker”这条默认集成路线。

实验工程应放在 pi 仓库之外的独立 Git 仓库，避免修改 pi 的依赖、测试配置和工作树。业务案例必须来自真实或脱敏真实数据；确定性 faux provider 只替代模型传输与输出轨迹，不生成虚构业务事实。

## 一、运行时与依赖

### Python 基线

- 默认 Python 3.12；如果公司生产基线不同，使用公司版本并在 ADR 中记录差异。
- 使用 `asyncio.TaskGroup`、`asyncio.timeout()` 和有界 `asyncio.Queue`；不依赖隐式后台任务。
- 所有外部输入在边界用 Pydantic v2 校验；领域内部使用不可变 model、dataclass 或严格类型。

### 建议初始化

从 pi 仓库根目录执行以下命令会在同级目录创建独立实验仓库。运行前先确认目标目录不存在；依赖下载需要公司允许的 Python package index。

```bash
cd "$(dirname "$(git rev-parse --show-toplevel)")"
uv init --lib --name agent-lab --python 3.12 --vcs git python-agent-lab
cd python-agent-lab
uv add "pydantic>=2,<3"
uv add --dev --bounds exact pytest pytest-asyncio pytest-timeout pyright ruff
```

初始化当天把 Pydantic 也改为解析出的精确 `==` 版本，并提交 `pyproject.toml`、`.python-version` 和 `uv.lock`。以后使用 `uv sync --frozen` 重建环境；升级依赖必须单独评审并重跑全部协议测试。长期文档不写死当前 PyPI patch 版本，安装真值由实验仓库的精确约束和 lockfile 共同保存。

### 最小质量配置

`pyproject.toml` 至少包含以下等价配置；可以合并到 `uv init` 生成的现有区段，不要重复创建同名表。

```toml
[tool.pyright]
typeCheckingMode = "strict"
pythonVersion = "3.12"
include = ["src", "tests"]

[tool.pytest.ini_options]
asyncio_mode = "strict"
testpaths = ["tests"]
timeout = 30

[tool.ruff]
target-version = "py312"
line-length = 120

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "ASYNC", "RUF"]
```

若公司基线不是 Python 3.12，同步修改 `.python-version`、`requires-python`、Pyright 和 Ruff 四处配置，避免本地与 CI 使用不同语义。

## 二、目录与模块边界

```text
python-agent-lab/
├── pyproject.toml
├── uv.lock
├── package.json
├── package-lock.json
├── tsconfig.rpc.json
├── README.md
├── src/agent_lab/
│   ├── messages.py
│   ├── events.py
│   ├── provider.py
│   ├── scripted_provider.py
│   ├── tools.py
│   ├── budgets.py
│   ├── loop.py
│   ├── rpc_models.py
│   ├── rpc_client.py
│   ├── session_v3.py
│   └── telemetry.py
└── tests/
    ├── contract/
    ├── integration/
    ├── fixtures/
    │   ├── cases/
    │   ├── jsonl_peer.py
    │   └── pi_rpc_faux.ts
    └── conftest.py
```

| 模块 | 当前唯一职责 | 明确不做什么 |
| --- | --- | --- |
| `messages.py` | 严格消息、content block、tool call/result 类型 | Provider 原生 payload |
| `events.py` | 事件判别联合、事件归约和唯一终态 | 发网络请求 |
| `provider.py` | `Provider` protocol 与 provider-neutral context | 重试整个 Agent loop |
| `scripted_provider.py` | 按测试脚本产生确定性事件 | 生成业务样本或连接真实 Provider |
| `tools.py` | 工具 protocol、Schema 校验、执行结果 | 身份授权和 sandbox 的假实现 |
| `budgets.py` | turn/tool-step/token/cost/deadline admission | Provider 账单真值 |
| `loop.py` | 单 run 的模型—工具—队列状态机 | session 持久化和控制平面 API |
| `rpc_models.py` | pi RPC command/response/event 的严格 Schema | 宽松透传未知字段 |
| `rpc_client.py` | 子进程、JSONL framing、请求关联和退出结算 | 拥有第二套 Agent loop |
| `session_v3.py` | 读取 v3 JSONL、重建树和 context projection | 手工改写 v3 文件，或混淆 legacy compatibility 与 R11 migration |
| `telemetry.py` | 结构化事件 sink 和关联 ID | 默认记录完整敏感原文 |

只有 `Provider` 和 `Tool` 从开始就需要 protocol：Provider 是隔离模型 I/O 与 loop 的稳定边界，P1 只实现 `ScriptedProvider`，路线 B 才在 ADR 后增加真实 adapter；Tool 则立即有多个业务实现。不要为了证明抽象合理而制造第二个假 Provider。storage、telemetry sink 等在出现第二个真实实现前使用具体类，避免为了框架外观制造抽象。

## 三、类型和边界规则

### Pydantic

- 每个外部边界 model 使用 `ConfigDict(strict=True, extra="forbid")`。
- 消息以 `role`、事件和 content block 以 `type` 构造判别联合。
- 使用 `Literal` 表达所有已知 tag；未知 tag 必须拒绝并产生带业务语义的协议错误。
- JSONL 先按 bytes/LF 分帧，再对单条 UTF-8 JSON record 做 `TypeAdapter.validate_json()`；不能先把整个 stdout 当普通文本切行。
- 模型输出的工具参数在执行前校验；middleware 变换后再次校验新值。
- 内部无法预知的 JSON 值使用递归 `JsonValue` 或 `object`，不以 `Any` 绕过类型检查。

### 错误

定义少量有业务语义的错误类别，例如 protocol、provider establishment、provider stream、tool validation、tool execution、budget、cancelled 和 child exit。边界捕获底层异常后保留 `cause` 给受控 trace，向上返回稳定错误码和去敏信息；禁止 `except Exception: pass`。

### 取消

- 捕获 `asyncio.CancelledError` 后只做有界清理并立即重新抛出。
- Provider、工具、stderr drain、listener 和 RPC pending request 都必须归属某个 `TaskGroup` 或明确 owner。
- 工具若调用子进程，先发送合作式终止；超过 grace period 后清理整个进程组。
- 取消只表示“不再继续推进”，不能把已经发生的外部副作用标成未执行。

## 四、确定性 Scripted Provider

`ScriptedProvider` 是协议测试器，输入是一组预先声明的 assistant turns，每个 turn 是合法事件序列或显式故障。它至少支持：

- text、thinking、tool call 多 content block。
- 可配置 delta 粒度与 `contentIndex`。
- 请求建立前失败、start 后失败、正常 done、aborted 和 length stop。
- 记录每次收到的 provider-neutral context、tools 和预算，供测试断言。
- 在脚本耗尽、模型 ID 不匹配或收到意外 context 时立即失败。

Scripted provider 不读取生产凭据，不注册真实 adapter。自动测试进程还应运行在无 egress 环境，并断言实际 provider/model ID 为 scripted/faux，形成第二道保护。

真实或脱敏业务案例放在 `tests/fixtures/cases/`，每个案例保存来源类别、脱敏说明、允许读取的 context、预期工具行为和禁止行为。不得为了让测试通过生成虚构工单、虚构用户或占位业务记录。

## 五、异步运行时不变量

### 结构化并发

- 一个 Agent 实例同一时间只有一个 active run。
- 每个 run 创建一个顶层 `TaskGroup`；Provider consumer、工具批和 event sink 子任务都在该作用域内结束。
- 工具并发使用固定上限，不能为模型返回的每个 call 无界创建 task。
- 同一资源写操作通过 canonical resource key 串行；不同资源也受全局 semaphore 限制。

### 背压

- 事件与 RPC 输出队列必须设置 `maxsize`。
- producer 在队列满时 await、超时或按明确策略终止，不能静默丢失 terminal/tool 事件。
- delta 可以在 UI 层按已记录策略合并；`message_end`、tool result、错误和结算事件不得丢弃。
- stderr 单独持续排空，不与 stdout JSONL parser 共用队列。

### 终态

- stream、turn、run 和 RPC request 分别最多结算一次。
- 所有异常、取消、timeout 和 child exit 都必须结算 pending Future。
- 事件 consumer 即使抛错，也不能让其他调用方永久等待；错误必须进入明确定义的 settlement 路径。
- 测试结束时除 pytest/事件循环自身任务外，不得残留未完成 task、子进程或未读取队列项。

## 六、实现顺序与验收

### P1：协议层

实现 `messages.py`、`events.py`、`provider.py` 和 `scripted_provider.py`。

验收：

- 判别联合拒绝未知 tag、额外字段和隐式类型转换。
- 多 block 能按 `contentIndex` 归约。
- 建连失败、流内失败和成功各只有一个最终结果。
- consumer 只依赖 provider-neutral 事件，不出现 Provider 专有分支；P1 用不同成功/失败脚本证明该契约。若阶段 2 的 ADR 选择路线 B，再增加一个真实 adapter，并只在独立人工 smoke job 使用。

### P2：最小 loop

实现 `tools.py`、`budgets.py` 和 `loop.py`，逐项完成[分阶段学习计划](./分阶段学习计划.md)的 LOOP-01–LOOP-13。

除 LOOP-01–LOOP-13 外增加：

| 编号 | 场景 | 必须断言 |
| --- | --- | --- |
| LOOP-14 | 事件队列达到上限 | producer 受背压且 terminal event 不丢失 |
| LOOP-15 | consumer/listener 失败 | run 进入可解释终态，调用方不悬挂 |
| LOOP-16 | 取消并发工具批 | 所有子 task 被回收，已完成结果与未完成结果可区分 |
| LOOP-17 | 总 deadline 与工具 timeout 竞争 | 只产生一个稳定终态，清理不超过 grace period |
| LOOP-18 | middleware 改写参数 | 新参数二次校验失败时工具从未执行 |

### P3：RPC client

实现 `rpc_models.py` 和 `rpc_client.py`，按[源码阅读与实验手册](./源码阅读与实验手册.md)的 RPC 检查表验收。

至少覆盖：response/event 交错、并发 ID 关联、LF/CRLF、超时、`clear_queue → abort`、stderr 大输出、child 正常/异常退出、进程组清理、session replacement 和 pending Future 全量结算。这里的 pi 集成子进程不是普通 `pi --mode rpc`：普通 CLI 没有选择 faux provider 的参数，必须使用下一节的公开 SDK fixture 注入 faux。

测试分为三层，不能让一个 happy-path fixture 承担全部故障覆盖：

1. Pydantic Schema、bytes/LF framing、CRLF 和 pending-map 结算使用纯单元测试。
2. `tests/fixtures/jsonl_peer.py` 是确定性本地传输 peer，按参数制造 response/event 交错、重复或未知 ID、半帧、stderr 洪水、非零退出和 hang；它只验证客户端传输与清理，不代表 pi 的业务行为，也不生成业务事实。
3. 真实 `runRpcMode()` + faux fixture 验证公开 pi API、事件语义和正常生命周期兼容。该层至少完成下述握手；升级 pi 后必须重跑。

三层自动测试都在无凭据、无 egress 环境执行。故障 peer 负责可控地覆盖 framing 与进程失败，pi fixture 负责证明没有把自制 peer 的行为误当成 pi 真值。

#### 可执行的 pi RPC + faux fixture

在独立 Python 仓库根目录初始化一个仅服务于 RPC fixture 的私有 ESM package，安装与本计划基线一致的精确 Node 依赖，并提交 `package.json` 和 `package-lock.json`。不要依赖 npm 偶然提升的传递依赖：

```bash
npm init --yes
npm pkg set type=module
npm pkg set private=true --json
npm install --ignore-scripts --save-exact \
  @earendil-works/pi-coding-agent@0.85.1 \
  @earendil-works/pi-ai@0.85.1
npm install --ignore-scripts --save-dev --save-exact \
  @types/node@22.19.19 \
  typescript@5.9.3
```

`tsconfig.rpc.json` 只检查这一个 fixture，并约束其语法可由 Node strip-only 模式直接删除：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noEmit": true,
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["tests/fixtures/pi_rpc_faux.ts"]
}
```

`skipLibCheck` 只跳过已发布依赖自身 `.d.ts` 的内部检查；fixture 仍按 strict mode 完整检查。当前 `0.85.1` 依赖声明在 NodeNext 下存在 JSON import attribute 和可选 MCP 类型解析问题，不加该项会在尚未检查 fixture 前被依赖错误阻断。

将以下文件保存为 `tests/fixtures/pi_rpc_faux.ts`。它使用公开 SDK 创建内存 runtime，显式注册 faux，确认唯一可用模型为 `faux/faux-model`，最后启动真实的 `runRpcMode()`：

```typescript
import { mkdtempSync, rmSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import {
	fauxAssistantMessage,
	fauxProvider,
	InMemoryCredentialStore,
	InMemoryModelsStore,
} from "@earendil-works/pi-ai";
import {
	createAgentSessionFromServices,
	createAgentSessionRuntime,
	createExtensionRuntime,
	ModelRuntime,
	runRpcMode,
	SessionManager,
	SettingsManager,
	type AgentSessionServices,
	type CreateAgentSessionRuntimeFactory,
	type ResourceLoader,
} from "@earendil-works/pi-coding-agent";

process.env.PI_OFFLINE = "1";

function createEmptyResourceLoader(): ResourceLoader {
	const extensionsResult = {
		extensions: [],
		errors: [],
		runtime: createExtensionRuntime(),
	};

	return {
		getExtensions: () => extensionsResult,
		getSkills: () => ({ skills: [], diagnostics: [] }),
		getPrompts: () => ({ prompts: [], diagnostics: [] }),
		getThemes: () => ({ themes: [], diagnostics: [] }),
		getAgentsFiles: () => ({ agentsFiles: [] }),
		getSystemPrompt: () => "You are an isolated faux RPC test agent.",
		getSystemPromptSource: () => undefined,
		getAppendSystemPrompt: () => [],
		getAppendSystemPromptSources: () => [],
		extendResources: () => {},
		reload: async () => {},
	};
}

const isolatedRoot = mkdtempSync(join(tmpdir(), "pi-rpc-faux-"));
process.on("exit", () => {
	rmSync(isolatedRoot, { recursive: true, force: true });
});

const faux = fauxProvider({
	api: "faux-rpc",
	provider: "faux",
	models: [{ id: "faux-model", name: "Faux Model", reasoning: false }],
});
faux.setResponses([fauxAssistantMessage("pong")]);

const modelRuntime = await ModelRuntime.create({
	credentials: new InMemoryCredentialStore(),
	modelsPath: null,
	modelsStore: new InMemoryModelsStore(),
	allowModelNetwork: false,
	refreshOnCreate: false,
});

modelRuntime.registerNativeProvider(faux.provider);
await modelRuntime.refresh({ allowNetwork: false });

const available = modelRuntime.getAvailableSnapshot();
if (available.length !== 1 || available[0]?.provider !== "faux" || available[0]?.id !== "faux-model") {
	throw new Error(
		`Expected only faux/faux-model, got: ${available
			.map((candidate) => `${candidate.provider}/${candidate.id}`)
			.join(", ")}`,
	);
}

const registeredProviderIds = modelRuntime.getRegisteredProviderIds();
if (registeredProviderIds.length !== 1 || registeredProviderIds[0] !== "faux") {
	throw new Error(`Unexpected explicitly registered providers: ${registeredProviderIds.join(", ")}`);
}

const model = faux.getModel();

const createRuntime: CreateAgentSessionRuntimeFactory = async ({
	cwd,
	agentDir,
	sessionManager,
	sessionStartEvent,
}) => {
	const diagnostics: AgentSessionServices["diagnostics"] = [];
	const services: AgentSessionServices = {
		cwd,
		agentDir,
		modelRuntime,
		settingsManager: SettingsManager.inMemory(),
		resourceLoader: createEmptyResourceLoader(),
		diagnostics,
	};

	const result = await createAgentSessionFromServices({
		services,
		sessionManager,
		sessionStartEvent,
		model,
		thinkingLevel: "off",
		noTools: "all",
	});

	return { ...result, services, diagnostics };
};

const runtimeHost = await createAgentSessionRuntime(createRuntime, {
	cwd: isolatedRoot,
	agentDir: isolatedRoot,
	sessionManager: SessionManager.inMemory(isolatedRoot),
});

await runRpcMode(runtimeHost);
```

Python 测试通过 `asyncio.create_subprocess_exec()` 启动以下 argv；fixture 所在仓库使用 pi 当前要求的 Node `>=22.19.0`，并先用上面的 lockfile 恢复依赖：

```python
from __future__ import annotations

import asyncio
import os
from pathlib import Path


async def start_faux_rpc(
    fixture_path: Path,
    isolated_home: Path,
    isolated_cwd: Path,
) -> asyncio.subprocess.Process:
    # 在改变 cwd 前固定 fixture 的绝对路径，并创建全新的 HOME/cwd，避免加载宿主凭据与资源。
    resolved_fixture = fixture_path.resolve(strict=True)
    isolated_home.mkdir(parents=True, exist_ok=False)
    isolated_cwd.mkdir(parents=True, exist_ok=False)
    resolved_home = isolated_home.resolve(strict=True)
    resolved_cwd = isolated_cwd.resolve(strict=True)

    return await asyncio.create_subprocess_exec(
        "node",
        "--experimental-strip-types",
        str(resolved_fixture),
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE,
        cwd=resolved_cwd,
        env={
            "HOME": str(resolved_home),
            "PATH": os.environ["PATH"],
            "PI_OFFLINE": "1",
        },
        start_new_session=True,
    )
```

验收时先发送 `get_state`，断言 model 严格等于 `faux/faux-model`；再发送 prompt `ping`，依次等待 prompt command response、`agent_settled`，最后用 `get_last_assistant_text` 断言 `pong`。关闭 stdin 后仍保持 stdout reader 与 stderr drain 运行到 EOF，再等待子进程在 10 秒内以 0 退出；stderr 应为空，pending map 应为空。

Python 3.12 的标准退出监管方案是：从进程启动起并发运行唯一的 stdout JSONL parser 和独立 stderr drain；关闭 stdin 后用 `asyncio.wait_for(process.wait(), 10)` 等待退出。`Process.wait()` 不会阻塞 event loop，但如果没有持续排空管道，子进程可能因 pipe buffer 填满而无法退出。超时后先向独立进程组发送 TERM，并再次有界等待；仍未退出再发送 KILL 并有界等待。进程退出后逐项 await 并检查 wait、parser、drain task 的结果，parser 的 `finally` 必须结算全部 pending Future，禁止吞掉异常。已有 stdout parser 时不得再调用 `stdout.read()` 竞争同一管道。

不要把 50ms heartbeat 或 `PidfdChildWatcher` 作为 Python 3.12 基线。只有目标 Python、内核和事件循环策略的压力测试能稳定复现“子进程已退出但 wait 通知延迟”，并已排除 pipe 背压和 task 泄漏后，才增加有界 heartbeat 或 pidfd 兼容层；记录最小复现、平台与版本，并在升级 Python 后删除或重新验证。`asyncio` child-watcher API 在 Python 3.12 已 deprecated，不能成为新的长期架构依赖。

这里有两个不能隐去的安全边界：

1. `ModelRuntime.create()` 仍会安装内建 Provider definition；公开 API 不能让 `getProviders()` 只含 faux。因此 fixture 检查的是“显式注册项只有 faux、净化环境后可用模型只有 faux、当前模型固定为 faux”，不是声称 registry 中不存在其他 definition。
2. `noTools: "all"` 会移除 Agent 工具，但 `runRpcMode()` 仍暴露直接的 RPC `bash` command。应用级测试不得发送 `bash`，并使用白名单环境、临时 HOME/cwd、内存凭据与 session；CI 还必须使用无网络 namespace、容器 egress deny 或等价 OS 隔离。`PI_OFFLINE=1` 与 `allowModelNetwork: false` 不是通用网络沙箱。

该 fixture 已在本计划的当前工作区依赖与 Node `v22.20.0` 上实跑通过；发布包的全新冷安装仍由学习者按上述精确依赖与 lockfile 单独验收。仓库或包升级后，先重跑这一握手测试，再接受新的 RPC Schema 或生命周期行为。

#### Linux 网络隔离验收

Linux 开发机或 CI 先验证非特权 user/network namespace 可用，再在新 network namespace 中运行 RPC 集成测试。依赖必须在进入 namespace 前通过 lockfile 安装完成：

```bash
unshare --user --map-root-user --net true
unshare --user --map-root-user --net -- \
  uv run --offline pytest -q \
  tests/integration/test_no_egress.py \
  tests/integration/test_pi_rpc_faux.py
```

`tests/integration/test_no_egress.py` 同时检查 namespace 只有 loopback，并在此前提下证明公共地址连接失败：

```python
from __future__ import annotations

import socket
from pathlib import Path


def test_network_namespace_has_no_egress() -> None:
    # 读取当前 network namespace 的 procfs；socket.if_nameindex() 本身可能先被 seccomp 拒绝。
    interfaces = {
        line.split(":", maxsplit=1)[0].strip()
        for line in Path("/proc/net/dev").read_text(encoding="utf-8").splitlines()
        if ":" in line
    }
    assert interfaces == {"lo"}

    result: int | None = None
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client:
            client.settimeout(1.0)
            result = client.connect_ex(("1.1.1.1", 443))
    except OSError:
        # seccomp 或 namespace policy 可以在 socket/connect syscall 边界直接拒绝。
        return
    assert result is not None and result != 0
```

先断言只有 loopback，避免隔离命令被误删时测试反而发出真实请求。若第一条 `unshare` 预检失败，必须改用 CI 容器 `--network none`、Kubernetes default-deny egress 或公司等价策略，并保留同类负向断言；不能把该测试静默 skip 后仍声称“网络隔离”。这只验证网络边界，文件、进程和 syscall 隔离仍需阶段 3B/L1 sandbox 测试。

### P4：v3 session 与生产差距

实现只读 `session_v3.py`，解析一份真实但去敏的 JSONL，重建 tree、active branch 和 compaction context。显式测试 blank/malformed/torn final line：记录诊断并区分“严格拒绝”与 pi 当前“跳过 malformed 行”的兼容策略，任何被跳过内容都不得算作可恢复 transcript。它是学习工具，不承担生产持久化。

验收：branch 不删除历史、fork 产生独立 session、最新 summary + kept tail 投影正确、tool call/result 不被切点拆散，并能重现“toolCall 已落盘但 effect 结果未知”的经典恢复缺口。

## 七、测试设计

### 分层

- `tests/contract/`：消息、事件、Provider、工具、budget 和 loop；完全进程内且无网络。
- `tests/integration/` 的传输故障层：用 `jsonl_peer.py` 覆盖 framing、stderr、乱序、异常退出、hang 和 pending 结算，不把它当作 pi 行为真值。
- `tests/integration/` 的 pi 兼容层：由公开 SDK fixture 启动真实 `runRpcMode()` 子进程，验证临时 session、正常退出和进程组行为；仍只用 faux provider，不把普通 CLI 误写成可直接选择 faux。
- `tests/fixtures/cases/`：真实或脱敏真实业务案例及其来源元数据。
- 真实 Provider smoke/eval：独立人工 job，不进入默认 `pytest`。

### 禁止使用睡眠猜时序

并发测试使用 barrier、`asyncio.Event`、受控 queue 或虚拟 clock 协调，不用“sleep 100ms 应该已经完成”作为正确性证据。只有测试 timeout 本身可以使用短 wall-clock 上限防止永久挂起。

### 任务泄漏断言

为每个异步测试建立 task/subprocess 基线；测试逻辑结束后让事件循环再推进一次，检查新建任务均已 done，RPC 子进程及其进程组已经退出，pending request map 为空。发现泄漏时测试失败并打印 task 名称和创建位置，不能在 teardown 中静默取消后当作通过。

## 八、统一质量命令

每次实现变更运行：

```bash
npm ci --ignore-scripts
./node_modules/.bin/tsc --project tsconfig.rpc.json
uv sync --frozen
uv run ruff format --check .
uv run ruff check .
uv run pyright
uv run pytest -q
```

CI 的依赖恢复使用单独、受审计的阶段，严格按 lockfile 从批准镜像或预热缓存安装；随后关闭 egress 并运行 typecheck 与测试。测试阶段还必须满足：

- 无 Provider 凭据、无网络 egress。
- `src/` 中没有显式 `Any`，第三方无类型边界在单独 adapter 中用 `object` 校验收窄。
- 每个测试设置总 timeout，测试进程退出后没有残留子进程。
- 失败时保存去敏后的事件 trace 和随机种子；不保存凭据或未经授权的完整业务原文。

如果修改测试文件，先运行对应单文件，再运行以上完整命令。随机化或 property-based 测试不是 LOOP-01–LOOP-18 的替代品；应先有可读的确定性回归，再补充状态空间探索。

## 九、与结业项目的关系

阶段 2 结束时必须在 ADR 中二选一：

- 路线 A：Python control plane + pi Node worker。这个实验工程的 Python loop 到 P2 为止，作为行为规格；结业项目使用 `rpc_client.py`，pi 唯一拥有生产 Agent loop。
- 路线 B：Python-native runtime。继续把 P2 扩展为生产候选；必须自己补 Provider adapter、session、compaction、durability 和全部运维能力，pi 只作为对照规格。

默认选择路线 A，因为它能直接复用正在学习的 pi 实现并缩小首个生产切片。路线 B 不是更“纯粹”的毕业方式，而是一项维护成本更高的架构决策；只有明确组织约束和长期所有者时再选。

无论选择哪条路线，12 周课程终点都是 L1 内部只读原型与生产评审包。真实 shadow/canary、写副作用、HA 和灾备必须使用[生产化能力清单](./生产化能力清单.md)继续推进，不能把实验工程的网络隔离回归通过结果当成生产就绪证明。
