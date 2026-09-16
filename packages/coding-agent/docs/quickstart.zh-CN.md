# 快速入门

本页将帮助你完成安装并开始第一个实用的 Pi 会话。

## 安装

Pi 以 npm 包的形式分发：

```bash
npm install -g --ignore-scripts @earendil-works/pi-coding-agent
```

`--ignore-scripts` 会在安装期间禁用依赖生命周期脚本。Pi 的常规 npm 安装不需要运行安装脚本。

### 卸载

请使用安装 Pi 时使用的包管理器。curl 安装程序会进行 npm 全局安装，因此通过 curl 和 npm 完成的安装都使用 npm 移除：

```bash
# curl 安装程序或 npm install -g
npm uninstall -g @earendil-works/pi-coding-agent

# pnpm
pnpm remove -g @earendil-works/pi-coding-agent

# Yarn
yarn global remove @earendil-works/pi-coding-agent

# Bun
bun uninstall -g @earendil-works/pi-coding-agent
```

卸载 Pi 后，设置、凭据、会话和已安装的 Pi 包仍会保留在 `~/.pi/agent/` 中。

然后在希望 Pi 操作的项目目录中启动它：

```bash
cd /path/to/project
pi
```

## 身份验证

Pi 可以通过 `/login` 使用订阅提供商，也可以通过环境变量或身份验证文件使用 API key 提供商。

### 方案 1：订阅登录

启动 Pi 并运行：

```text
/login
```

然后选择提供商。内置订阅登录包括 Claude Pro/Max、ChatGPT Plus/Pro（Codex）和 GitHub Copilot。

### 方案 2：API key

启动 Pi 前设置 API key：

```bash
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

也可以运行 `/login` 并选择 API key 提供商，将 key 存储到 `~/.pi/agent/auth.json`。

有关所有支持的提供商、环境变量和云提供商配置，请参阅[提供商](providers.zh-CN.md)。

## 第一个会话

Pi 启动后，输入请求并按 Enter：

```text
总结这个仓库，并告诉我如何运行它的检查。
```

默认情况下，Pi 为模型提供四个工具：

- `read`——读取文件
- `write`——创建或覆盖文件
- `edit`——修补文件
- `bash`——运行 shell 命令

其他内置只读工具（`grep`、`find`、`ls`）可通过工具选项启用。Pi 在当前工作目录中运行，并且可以修改其中的文件。如需方便地回滚，请使用 git 或其他检查点工作流。

## 为 Pi 提供项目说明

Pi 会在启动时加载上下文文件。添加 `AGENTS.md` 文件可以告诉它如何在项目中工作：

```markdown
# 项目说明

- 更改代码后运行 `npm run check`。
- 不要在本地运行生产迁移。
- 保持回复简洁。
```

Pi 会加载：

- `~/.pi/agent/AGENTS.md` 中的全局说明
- 父目录和当前目录中的 `AGENTS.md` 或 `CLAUDE.md`

如果某个目录包含 `AGENTS.override.md`，Pi 会加载该文件，而不是同目录中的 `AGENTS.md` 或 `CLAUDE.md`。

更改上下文文件后，请重启 Pi 或运行 `/reload`。

## 常见尝试

### 引用文件

在编辑器中输入 `@` 可以模糊搜索文件，也可以在命令行中传入文件：

```bash
pi @README.md "Summarize this"
pi @src/app.ts @src/app.test.ts "Review these together"
```

可以使用 Ctrl+V（Windows 上为 Alt+V）粘贴图像或文本；在支持的终端中也可以拖入图像。

### 运行 shell 命令

在交互模式中：

```text
!npm run lint
```

命令输出会发送给模型。使用 `!!command` 可以运行命令而不将其输出添加到模型上下文。

### 切换模型

使用 `/model` 或 Ctrl+L 为当前会话选择模型。在模型选择器中按 Ctrl+S，可将高亮显示的模型保存为启动默认值。使用 `/thinking` 为当前会话选择思考级别，或在该选择器中按 Ctrl+S 保存启动时的默认思考级别。使用 Shift+Tab 循环切换思考级别。使用 Ctrl+P / Shift+Ctrl+P 循环切换范围内的模型。

### 稍后继续

会话会自动保存：

```bash
pi -c                  # 继续最近的会话
pi -r                  # 浏览之前的会话
pi --name "my task"    # 启动时设置会话显示名称
pi --session <path|id> # 打开指定会话
```

在 Pi 中，使用 `/resume`、`/new`、`/tree`、`/fork` 和 `/clone` 管理会话。

### 非交互模式

对于一次性提示：

```bash
pi -p "Summarize this codebase"
cat README.md | pi -p "Summarize this text"
pi -p @screenshot.png "What's in this image?"
```

使用 `--mode json` 输出 JSON 事件，或使用 `--mode rpc` 进行进程集成。

## 后续步骤

- [使用 Pi](usage.zh-CN.md)——交互模式、斜杠命令、会话、上下文文件和 CLI 参考。
- [提供商](providers.zh-CN.md)——身份验证和模型配置。
- [设置](settings.zh-CN.md)——全局配置和项目配置。
- [键绑定](keybindings.zh-CN.md)——快捷键和自定义。
- [Pi 包](packages.zh-CN.md)——安装共享扩展、技能、提示和主题。

平台说明：[Windows](windows.zh-CN.md)、[Termux](termux.zh-CN.md)、[tmux](tmux.zh-CN.md)、[终端配置](terminal-setup.zh-CN.md)、[Shell 别名](shell-aliases.zh-CN.md)。
