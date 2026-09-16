# 容器化

Pi 默认以完整权限运行，但在某些情况下，你会希望更严格地控制 Pi 可以写入哪些目录，以及它可以进行哪些访问。

通常有两种方案：

1. 在隔离环境中运行完整的 `pi` 进程；或者
2. 在宿主机上运行 `pi`，并将工具执行转发到隔离环境。

## 选择一种模式

| 模式 | 隔离对象 | 最适合 | 说明 |
| --- | --- | --- | --- |
| Gondolin 扩展 | 内置工具和 `!` 命令 | 在宿主机保留身份验证的同时使用本地微型 VM 隔离 | 参阅 [`examples/extensions/gondolin/`](../examples/extensions/gondolin/)。 |
| 原生 Docker | 本地容器中的完整 `pi` 进程 | 简单的本地隔离 | 提供商 API key 会进入容器。 |
| OpenShell | 受策略控制的沙箱中的完整 `pi` 进程 | 本地或远程托管沙箱 | 需要 OpenShell gateway |
| Docker Sandboxes | 托管沙箱中的完整 `pi` 进程 | 将提供商 key 保留在宿主机上的本地隔离 | 需要 Docker Sandboxes（`sbx`）。 |

扩展会在 `pi` 进程所在的位置运行。如果在宿主机上运行 `pi` 并使用工具转发扩展，其他自定义扩展工具仍会在宿主机上运行，除非它们也委派自己的操作。

## Gondolin

[Gondolin](https://github.com/earendil-works/gondolin) 是一个本地 Linux 微型 VM。当你希望在宿主机上运行 `pi`，但将所有内置工具转发到 VM 时，请使用[示例扩展](../examples/extensions/gondolin)。

配置：

```bash
cp -R packages/coding-agent/examples/extensions/gondolin ~/.pi/agent/extensions/gondolin
cd ~/.pi/agent/extensions/gondolin
npm install --ignore-scripts
```

在需要挂载的项目中运行：

```bash
cd /path/to/project
pi -e ~/.pi/agent/extensions/gondolin
```

该扩展会将宿主机当前工作目录挂载到 VM 的 `/workspace`，并覆盖 `read`、`write`、`edit`、`bash`、`grep`、`find` 和 `ls`。用户的 `!` 命令也会转发到 VM。对 `/workspace` 下文件的更改会写回宿主机。

要求：`@earendil-works/gondolin` 需要 Node.js >= 23.6.0，此外还需要 QEMU（必须通过包管理器安装）。

## 原生 Docker

需要最简单的本地容器边界时，请在 Docker 中运行完整的 `pi` 进程。

`Dockerfile.pi`：

```dockerfile
FROM node:24-bookworm-slim

RUN apt-get update \
  && apt-get install -y --no-install-recommends bash ca-certificates git ripgrep \
  && rm -rf /var/lib/apt/lists/*
RUN npm install -g --ignore-scripts @earendil-works/pi-coding-agent

WORKDIR /workspace
ENTRYPOINT ["pi"]
```

构建并运行：

```bash
docker build -t pi-sandbox -f Dockerfile.pi .

docker run --rm -it \
  -e ANTHROPIC_API_KEY \
  -v "$PWD:/workspace" \
  -v pi-agent-home:/root/.pi/agent \
  pi-sandbox
```

`-v "$PWD:/workspace"` 会将当前目录挂载到容器的 `/workspace`。与 Gondolin 示例相同，在 Docker 内读写 `/workspace` 会直接影响宿主机文件。

如果希望设置和会话只保存在容器中，请为 `/root/.pi/agent` 使用命名卷。挂载宿主机的 `~/.pi/agent` 会将宿主机身份验证和会话文件暴露给容器。

## OpenShell

需要具备文件系统、进程、网络、凭据和推理控制的策略化沙箱时，请使用 [NVIDIA OpenShell](https://docs.nvidia.com/openshell/about/overview)。OpenShell 可以通过由 Docker、Podman 或 VM 运行时支持的本地 gateway 运行沙箱，也可以通过远程 Kubernetes gateway 运行。

每个沙箱都需要一个活动 gateway。创建沙箱前，请注册并选择一个 gateway：

```bash
openshell gateway add <gateway-url> --name <name>
openshell gateway select <name>
```

在 OpenShell 沙箱中启动 `pi`：

```bash
openshell sandbox create --name pi-sandbox --from pi -- pi
```

在这种模式下，完整的 `pi` 进程都在沙箱中运行。内置工具、`!` 命令和扩展工具都在 OpenShell 边界内执行。

如果 gateway 位于远程，项目文件不会从宿主机进行绑定挂载，因此沙箱中的写入不会反映到你的计算机上。请在沙箱内克隆仓库，或使用 OpenShell 文件传输命令：

```bash
openshell sandbox upload pi-sandbox ./repo /workspace
openshell sandbox download pi-sandbox /workspace/repo ./repo-out
```

OpenShell 提供商可以将原始模型 API key 保留在沙箱之外。配置推理路由后，沙箱中的代码可以调用 `https://inference.local`，gateway 会在上游注入已配置的提供商凭据。如果希望模型流量使用此路由，请将 Pi 配置为使用相应的 OpenAI 兼容或 Anthropic 兼容端点。

## Docker Sandboxes

[Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) 是 Docker 提供的托管沙箱运行时，可在沙箱中运行完整的 `pi` 进程。它属于[无内置沙箱](security.zh-CN.md#无内置沙箱)中提到的容器边界之一。

与上面的原生 Docker 模式不同，提供商凭据不会传入容器。沙箱收到的是哨兵值，`sbx` 代理会在向 `api.anthropic.com` 发出请求时将其替换为真实凭据。凭据在创建时接入，因此请在创建沙箱前将凭据存储在宿主机上。

对于 Claude Pro/Max 订阅，请在装有 Claude Code 的计算机上运行 `claude setup-token`，然后将结果存储在宿主机上。如果已经绑定了 `anthropic` secret，请先将其删除；否则代理会在 Bearer token 之外再添加一个 `x-api-key` header，Anthropic 会拒绝该请求。`sbx secret set-custom` 从 stdin 读取 token，因此不会将其留在 shell 历史记录中。

```bash
sbx secret rm anthropic

sbx secret set-custom \
  --host api.anthropic.com \
  --env ANTHROPIC_OAUTH_TOKEN \
  --placeholder 'sk-ant-oat01-{rand}'
```

沙箱获得的是 OAuth 形式的占位符，而不是真实 token。代理会在发往该主机的请求中替换它；`ANTHROPIC_OAUTH_TOKEN` 是 Pi 已经读取并优先于 API key 使用的变量，因此不需要额外配置 Pi。

如果使用 API key，请改用 `sbx secret set anthropic` 存储。该工具包以相同方式接入它：使用一个由代理在出站请求中替换的哨兵值。

凭据存储完毕后，从需要挂载的项目中启动 `pi`：

```bash
sbx run --kit "docker.io/sbx/pi-kit:latest" pi
```

该工具包已经在镜像中预装了 `pi`，因此沙箱无需安装任何内容即可启动，当前目录就是沙箱工作区。

不要从沙箱内部进行身份验证：在那里运行 `/login` 会把真实 token 写入容器，破坏代理模型的安全性。

脚本化用法与此相同：

```bash
sbx exec <sandbox-name> -- pi -p "list the failing tests"
```

有关完整的凭据矩阵、故障排除和版本固定方法，请参阅[工具包文档](https://github.com/docker/sbx-kits-contrib/tree/main/pi)。
