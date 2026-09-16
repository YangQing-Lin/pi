<p align="center">
  <a href="https://pi.dev">
    <img alt="Pi 标志" src="https://pi.dev/logo-auto.svg" width="128">
  </a>
</p>
<p align="center">
  <a href="https://discord.com/invite/3cU7Bz4UPx"><img alt="Discord" src="https://img.shields.io/badge/discord-community-5865F2?style=flat-square&logo=discord&logoColor=white" /></a>
  <a href="https://www.npmjs.com/package/@earendil-works/pi-coding-agent"><img alt="npm" src="https://img.shields.io/npm/v/@earendil-works/pi-coding-agent?style=flat-square" /></a>
</p>

> 默认情况下，新贡献者创建的 issue 和 PR 会被自动关闭。维护者每天都会审查这些自动关闭的 issue。详情请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)。

# Pi Agent 框架

这里是 Pi agent 框架项目的主仓库，其中包含可自扩展的编码 agent。

* **[@earendil-works/pi-coding-agent](packages/coding-agent)**：交互式编码 agent CLI
* **[@earendil-works/pi-agent-core](packages/agent)**：支持工具调用和状态管理的 agent 运行时
* **[@earendil-works/pi-ai](packages/ai)**：统一的多提供商 LLM API（OpenAI、Anthropic、Google 等）

要进一步了解 Pi：

* [访问 pi.dev](https://pi.dev)，查看项目网站和演示
* [阅读文档](https://pi.dev/docs/latest)，也可以直接让 agent 介绍自己

## 所有包

| 包 | 说明 |
|---------|-------------|
| **[@earendil-works/chord](packages/chord)** | 面向服务、复制状态、RPC 和插件的独立应用组合运行时 |
| **[@earendil-works/pi-telemetry](packages/telemetry)** | 与供应商无关的遥测契约、参考适配器、一致性测试和类型化 schema |
| **[@earendil-works/pi-ai](packages/ai)** | 统一的多提供商 LLM API（OpenAI、Anthropic、Google 等） |
| **[@earendil-works/pi-agent-core](packages/agent)** | 支持工具调用和状态管理的 agent 运行时 |
| **[@earendil-works/pi-coding-agent](packages/coding-agent)** | 交互式编码 agent CLI |
| **[@earendil-works/pi-tui](packages/tui)** | 支持差异化渲染的终端 UI 库 |

有关 Slack/聊天自动化和工作流的信息，请参阅 [earendil-works/pi-chat](https://github.com/earendil-works/pi-chat)。

## 权限与容器化

Pi 不包含用于限制文件系统、进程、网络或凭据访问的内置权限系统。默认情况下，它使用启动它的用户和进程所拥有的权限运行。

如果需要更严格的边界，请通过容器或沙箱运行 Pi。三种可选方案请参阅 [packages/coding-agent/docs/containerization.md](packages/coding-agent/docs/containerization.md)：

- **Gondolin 扩展**：将 `pi` 和提供商身份验证保留在宿主机上，同时把内置工具和 `!` 命令转发到本地 Linux 微型虚拟机中。
- **原生 Docker**：在本地容器中运行完整的 `pi` 进程，实现简单隔离。
- **OpenShell**：在受策略控制的沙箱中运行完整的 `pi` 进程。

## 参与贡献

贡献指南请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)，项目特定规则（适用于人类和 agent）请参阅 [AGENTS.md](AGENTS.md)。Pi 的长期规划也可以在 [RFC](https://rfc.earendil.com/keyword/pi/) 中找到。

## 开发

```bash
npm install --ignore-scripts  # 安装所有依赖，但不运行生命周期脚本
npm run build         # 刷新模型数据，然后构建所有包
npm run build:offline # 使用现有模型数据离线重新构建
npm run check         # 执行 lint、格式化检查和类型检查
./test.sh            # 运行测试（没有 API key 时跳过依赖 LLM 的测试）
./pi-test.sh         # 从源码运行 Pi（可以从任意目录执行）
```

## 从发布版源码构建独立二进制文件

GitHub Release 中包含带版本号的源码归档，其校验和记录在该版本的 `SHA256SUMS` 文件中。解压归档，然后运行与官方独立二进制文件相同的构建脚本：

```bash
VERSION="<release-version>"
tar -xzf "pi-${VERSION}-source.tar.gz"
cd "pi-${VERSION}"
./scripts/build-binaries.sh --offline-model-data --platform linux-x64 --out "$PWD/out"
```

归档中包含发布时的模型数据和预构建原生模块。`--offline-model-data` 会直接使用这些模型数据，不刷新提供商目录。该脚本会安装依赖，并将运行时资源一同构建到可执行文件中；如果依赖已经就绪，请传入 `--skip-install`。

## 供应链加固

我们将 npm 依赖变更视同代码变更进行审查。

- 直接外部依赖固定为精确版本，内部 workspace 包仍使用版本范围。
- `.npmrc` 设置了 `save-exact=true` 和 `min-release-age=2`，避免 npm 解析依赖时采用当天刚发布的版本。
- `package-lock.json` 是依赖信息的权威来源。除非设置 `PI_ALLOW_LOCKFILE_CHANGE=1`，否则 pre-commit 会阻止意外提交 lockfile。
- `npm run check` 会检查直接依赖是否固定版本、原生 TypeScript import 的兼容性，以及生成的 coding-agent shrinkwrap。
- 发布的 CLI 包包含从根目录 lockfile 生成的 `packages/coding-agent/npm-shrinkwrap.json`，以便为 npm 用户固定传递依赖版本。
- 发布冒烟测试使用 `npm run release:local`，在仓库外部构建、打包并创建隔离的 npm 和 Bun 安装环境，然后才会创建版本标签。
- 在支持的情况下，本地发布安装、文档中的 npm 安装命令和 `pi update --self` 都会使用 `--ignore-scripts`。
- CI 使用 `npm ci --ignore-scripts` 安装依赖；定时 GitHub workflow 会运行 `npm audit --omit=dev` 和 `npm audit signatures --omit=dev`。
- shrinkwrap 生成流程为依赖生命周期脚本维护了显式允许列表；包含生命周期脚本的新依赖在通过审查前无法通过检查。

## 分享你的开源软件编码 agent 会话

如果你使用 Pi 或其他编码 agent 参与开源软件开发，请分享你的会话。

公开的开源软件会话数据以真实任务、工具使用、失败和修复为基础，有助于改进编码 agent，而不是依赖玩具式基准测试。

完整说明请参阅 [X 上的这篇帖子](https://x.com/badlogicgames/status/2037811643774652911)。

要发布会话，请使用 [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf)。请阅读它的 README.md 了解配置方法。你只需要一个 Hugging Face 账号、Hugging Face CLI 和 `pi-share-hf`。

你也可以观看[这段视频](https://x.com/badlogicgames/status/2041151967695634619)，其中演示了如何发布 `pi-mono` 会话。

我会定期在这里发布自己的 `pi-mono` 工作会话：

- [Hugging Face 上的 badlogicgames/pi-mono](https://huggingface.co/datasets/badlogicgames/pi-mono)

## 许可证

MIT

<p align="center">
  <a href="https://pi.dev">pi.dev</a> 域名由以下项目慷慨捐赠：
  <br /><br />
  <a href="https://exe.dev"><img src="packages/coding-agent/docs/images/exy.png" alt="Exy 吉祥物" width="48" /><br />exe.dev</a>
</p>
