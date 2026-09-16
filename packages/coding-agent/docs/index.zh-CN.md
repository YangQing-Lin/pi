# Pi 文档

Pi 是一个精简的终端编码执行框架。其核心保持小巧，并可通过 TypeScript 扩展、技能、提示模板、主题和 Pi 包进行扩展。

## 快速开始

使用 npm 安装 Pi：

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` 会在安装期间禁用依赖生命周期脚本。Pi 的常规 npm 安装不需要运行安装脚本。

在 Linux 或 macOS 上，也可以使用安装程序：

```bash
curl -fsSL https://pi.dev/install.sh | sh
```

对于通过 curl 或 npm 完成的安装，请使用 npm 卸载 Pi：

```bash
npm uninstall -g @earendil-works/pi-coding-agent
```

对于 pnpm、Yarn 或 Bun 安装，请使用对应的全局移除命令：`pnpm remove -g @earendil-works/pi-coding-agent`、`yarn global remove @earendil-works/pi-coding-agent` 或 `bun uninstall -g @earendil-works/pi-coding-agent`。

然后在项目目录中运行：

```bash
pi
```

对于订阅提供商，请使用 `/login` 进行身份验证；也可以在启动 Pi 前设置 `ANTHROPIC_API_KEY` 等 API key。

完整的首次运行流程请参阅[快速入门](quickstart.zh-CN.md)。

## 从这里开始

- [快速入门](quickstart.zh-CN.md)——安装、身份验证并运行第一个会话。
- [使用 Pi](usage.zh-CN.md)——交互模式、斜杠命令、上下文文件和 CLI 参考。
- [提供商](providers.zh-CN.md)——内置提供商的订阅和 API key 配置。
- [llama.cpp](llama-cpp.zh-CN.md)——运行本地路由器，并使用 `/llama` 管理模型。
- [安全](security.zh-CN.md)——项目信任、沙箱边界和漏洞报告。
- [容器化](containerization.zh-CN.md)——使用 Gondolin、Docker 或 OpenShell 将 Pi 置于沙箱中。
- [设置](settings.zh-CN.md)——全局设置和项目设置。
- [键绑定](keybindings.zh-CN.md)——默认快捷键和自定义键绑定。
- [会话](sessions.zh-CN.md)——会话管理、分支和树状导航。
- [上下文压缩](compaction.zh-CN.md)——上下文压缩和分支摘要。

## 自定义

- [扩展](extensions.zh-CN.md)——用于工具、命令、事件和自定义 UI 的 TypeScript 模块。
- [技能](skills.zh-CN.md)——用于按需复用能力的 Agent Skills。
- [提示模板](prompt-templates.zh-CN.md)——可从斜杠命令展开的可复用提示。
- [主题](themes.zh-CN.md)——内置和自定义终端主题。
- [Pi 包](packages.zh-CN.md)——打包并共享扩展、技能、提示和主题。
- [自定义模型](models.zh-CN.md)——为支持的提供商 API 添加模型条目。
- [自定义提供商](custom-provider.zh-CN.md)——实现自定义 API 和 OAuth 流程。

## 以编程方式使用

- [SDK](sdk.zh-CN.md)——将 Pi 嵌入 Node.js 应用程序。
- [RPC 模式](rpc.zh-CN.md)——通过 stdin/stdout JSONL 集成。
- [JSON 事件流模式](json.zh-CN.md)——输出结构化事件的打印模式。
- [TUI 组件](tui.zh-CN.md)——为扩展构建自定义终端 UI。

## 参考

- [环境变量](environment-variables.zh-CN.md)——Pi 进程配置，以及 bash 工具可用的会话元数据。
- [会话格式](session-format.zh-CN.md)——JSONL 会话文件格式、条目类型和 SessionManager API。

## 平台配置

- [Windows](windows.zh-CN.md)
- [Android 上的 Termux](termux.zh-CN.md)
- [tmux](tmux.zh-CN.md)
- [终端配置](terminal-setup.zh-CN.md)
- [Shell 别名](shell-aliases.zh-CN.md)

## 开发

- [开发](development.zh-CN.md)——本地配置、项目结构和调试。
