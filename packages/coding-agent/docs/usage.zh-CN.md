# 使用 Pi

本页汇总快速入门页未涵盖的日常使用细节。

## 交互模式

<p align="center"><img src="images/interactive-mode.png" alt="交互模式" width="600"></p>

界面包含四个主要区域：

- **启动标题区**——快捷键、已加载的上下文文件、提示模板、技能和扩展
- **消息区**——用户消息、assistant 回复、工具调用、工具结果、通知、错误和扩展 UI
- **编辑器**——输入区域；边框颜色表示当前思考级别
- **页脚**——工作目录、会话名称、token/缓存用量、费用、上下文用量和当前模型。总计包括 assistant 回复、工具报告的用量和摘要生成。

编辑器可以临时替换为 `/settings` 等内置 UI 或自定义扩展 UI。

### 编辑器功能

| 功能 | 使用方法 |
|---------|-----|
| 文件引用 | 输入 `@` 模糊搜索项目文件 |
| 路径补全 | 按 Tab 补全路径 |
| 多行输入 | Shift+Enter；Windows Terminal 上也可使用 Ctrl+Enter |
| 复制回复 | Ctrl+X 在 `/tree` 中复制所选消息；其他情况下复制最后一条 assistant 消息，或在禁用 `fullscreenCopyOnSelect` 时复制活动的全屏文本选择 |
| 图像 | 使用 Ctrl+V 粘贴，Windows 上使用 Alt+V，或拖入终端 |
| Shell 命令 | `!command` 运行命令并将输出发送给模型 |
| 隐藏 Shell 命令 | `!!command` 运行命令但不将输出发送给模型 |
| 外部编辑器 | Ctrl+G 打开 `externalEditor`、`$VISUAL`、`$EDITOR`、Windows 上的 Notepad 或其他平台上的 `nano` |

所有快捷键及自定义方法请参阅[键绑定](keybindings.zh-CN.md)。

## 斜杠命令

在编辑器中输入 `/` 可打开命令补全。扩展可以注册自定义命令；技能以 `/skill:name` 提供；提示模板通过 `/templatename` 展开。

| 命令 | 说明 |
|---------|-------------|
| `/login`、`/logout` | 管理 OAuth 或 API key 凭据 |
| [`/llama`](llama-cpp.zh-CN.md) | 下载、加载和卸载 llama.cpp 路由器模型 |
| `/model` | 切换模型；在选择器中按 Ctrl+S 保存启动默认值 |
| `/thinking` | 切换思考级别；在选择器中按 Ctrl+S 保存启动默认值 |
| `/scoped-models` | 启用/禁用用于 Ctrl+P 循环切换的模型 |
| `/settings` | 配置主题、消息投递、传输和其他首选项 |
| `/resume` | 从以前的会话中选择 |
| `/new` | 启动新会话 |
| `/name <name>` | 设置会话显示名称 |
| `/session` | 显示会话文件、ID、消息、token 和费用 |
| `/tree` | 跳转到会话中的任意位置并从那里继续 |
| `/trust` | 保存项目信任决定，供未来会话使用 |
| `/fork` | 从之前的用户消息创建新会话 |
| `/clone` | 将当前活动分支复制到新会话 |
| `/compact [prompt]` | 手动压缩上下文，可选择提供自定义指令 |
| `/copy` | 将最后一条 assistant 消息复制到剪贴板 |
| `/export [file]` | 将会话导出为 HTML 或 JSONL |
| `/import <file>` | 从 JSONL 文件导入并恢复会话 |
| `/share` | 作为私有 GitHub gist 上传，并生成可分享的 HTML 链接 |
| `/reload` | 重新加载键绑定、扩展、技能、提示、主题和上下文文件 |
| `/hotkeys` | 显示所有键盘快捷键 |
| `/changelog` | 显示版本历史 |
| `/quit` | 退出 Pi |

## 消息队列

agent 仍在工作时也可以提交消息：

- **Enter** 将引导消息加入队列，在当前 assistant 轮次执行完工具调用后投递。
- **Alt+Enter** 将后续消息加入队列，在 agent 完成所有工作后投递。
- **Escape** 中止操作，并将排队消息恢复到编辑器。
- **Alt+Up** 将排队消息取回编辑器。

Windows Terminal 默认将 Alt+Enter 用于全屏。如果希望 Pi 接收该快捷键，请按照[终端配置](terminal-setup.zh-CN.md)中的说明重新映射。

在[设置](settings.zh-CN.md)中使用 `steeringMode` 和 `followUpMode` 配置投递方式。

## 会话

会话会自动保存到 `~/.pi/agent/sessions/`，并按工作目录组织。

```bash
pi -c                  # 继续最近的会话
pi -r                  # 浏览并选择会话
pi --no-session        # 临时模式；不保存
pi --name "my task"    # 启动时设置会话显示名称
pi --session <path|id> # 使用指定的会话文件或会话 ID
pi --fork <path|id>    # 将会话分叉到新会话文件
```

实用的会话命令：

- `/session` 显示当前会话文件和 ID。
- `/tree` 导航文件内的会话树，并可总结被放弃的分支。
- `/fork` 从较早的用户消息创建新会话。
- `/clone` 将当前活动分支复制到新会话文件。
- `/compact` 总结较早的消息以释放上下文。

详情请参阅[会话](sessions.zh-CN.md)和[上下文压缩](compaction.zh-CN.md)。

## 上下文文件

Pi 启动时从以下位置加载 `AGENTS.md` 或 `CLAUDE.md`：

- `~/.pi/agent/AGENTS.md` 中的全局指令
- 从当前工作目录向上遍历的父目录
- 当前目录

如果某个目录包含 `AGENTS.override.md`，Pi 会加载它，而不是该目录中的 `AGENTS.md` 或 `CLAUDE.md`。其他目录中的上下文文件仍会正常叠加。

使用上下文文件记录项目约定、命令、安全规则和首选项。使用 `--no-context-files` 或 `-nc` 禁用加载。

### 系统提示文件

使用以下文件替换默认系统提示：

- 项目中的 `.pi/SYSTEM.md`
- 全局的 `~/.pi/agent/SYSTEM.md`

在任一位置使用 `APPEND_SYSTEM.md`，可以追加到默认提示而不替换它。

### 项目信任

交互启动时，如果项目文件夹包含项目本地设置、资源或项目 `.agents/skills`，且 `~/.pi/agent/trust.json` 中没有该文件夹或父文件夹的已保存决定，Pi 会先询问是否信任。信任项目后，Pi 可以加载 `.pi/settings.json` 和 `.pi` 资源、安装缺失的项目包并执行项目扩展。

信任决定作出之前，Pi 只加载上下文文件、用户/全局扩展和 CLI `-e` 扩展，以便它们处理 `project_trust` 事件。项目本地扩展、项目包管理的扩展和项目设置仅在项目受信任后加载。当切换到另一个 cwd 的会话，且当前进程尚未确定该目录的信任状态时，也会进行同样的分阶段加载。

非交互模式（`-p`、`--mode json` 和 `--mode rpc`）不会显示信任提示。没有适用的已保存决定时，它们使用全局设置中的 `defaultProjectTrust`：`ask`（默认）和 `never` 会忽略这些项目资源，而 `always` 会信任它们。传入 `--approve`/`-a` 或 `--no-approve`/`-na` 可覆盖单次运行的项目信任设置。

如果扩展和已保存决定均不适用，`defaultProjectTrust` 控制回退行为。在 `~/.pi/agent/settings.json` 中将其设为 `"ask"`、`"always"` 或 `"never"`，也可以通过 `/settings` 更改。

`pi config` 和包命令使用相同的项目信任流程，但 `pi update` 绝不会显示提示。传入 `--approve` 可在单次命令中信任项目本地设置，传入 `--no-approve` 则忽略它们。

在交互模式中使用 `/trust` 可保存项目信任决定，供未来会话使用，也可以信任直接父文件夹。它只写入 `~/.pi/agent/trust.json`；当前会话不会重新加载，因此需重启 Pi 才能应用更改。

## 导出和分享会话

使用 `/export [file]` 将会话写入 HTML。

使用 `/share` 将会话作为私有 GitHub gist 上传，并生成可分享的 HTML 链接。

如果你使用 Pi 参与开源工作，并希望发布会话用于模型、提示、工具和评估研究，请参阅 [`badlogic/pi-share-hf`](https://github.com/badlogic/pi-share-hf)。它会将会话发布到 Hugging Face 数据集。

## CLI 参考

```bash
pi [options] [--] [@files...] [messages...]
```

### 包命令

```bash
pi install <source> [-l]     # 安装包；-l 表示项目本地
pi remove <source> [-l]      # 移除包
pi uninstall <source> [-l]   # remove 的别名
pi update [source|self|pi]   # 仅更新 Pi，或更新一个包来源
pi update --all              # 更新 Pi 和包；协调固定的 git ref
pi update --extensions       # 仅更新包；协调固定的 git ref
pi update --models           # 仅刷新模型目录
pi update --self             # 仅更新 Pi
pi update --extension <src>  # 更新一个包
pi list                      # 列出已安装的包
pi config                    # 启用/禁用包资源
```

这些命令用于管理 Pi 包，`pi update` 还可以更新 Pi CLI 安装。要卸载 Pi 本身，请参阅[快速入门](quickstart.zh-CN.md#卸载)。`pi config` 和项目包命令接受 `--approve`/`--no-approve`，用于在单次命令中信任或忽略项目本地设置。`pi update` 绝不会提示项目信任。

有关包来源和安全说明，请参阅 [Pi 包](packages.zh-CN.md)。

### 模式

| Flag | 说明 |
|------|-------------|
| 默认值 | 交互模式 |
| `-p`、`--print` | 打印回复并退出 |
| `--mode json` | 将所有事件输出为 JSON Lines；参阅 [JSON 模式](json.zh-CN.md) |
| `--mode rpc` | 通过 stdin/stdout 使用 RPC 模式；参阅 [RPC 模式](rpc.zh-CN.md) |
| `--export <in> [out]` | 将会话导出为 HTML |

在打印模式下，Pi 还会读取管道传入的 stdin，并将其合并到初始提示：

```bash
cat README.md | pi -p "Summarize this text"
```

### 模型选项

| 选项 | 说明 |
|--------|-------------|
| `--provider <name>` | 提供商，例如 `anthropic`、`openai` 或 `google` |
| `--model <pattern>` | 模型模式或 ID；支持 `provider/id` 和可选的 `:<thinking>` |
| `--api-key <key>` | API key，覆盖环境变量 |
| `--thinking <level>` | `off`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max` |
| `--models <patterns>` | 用于 Ctrl+P 循环切换的逗号分隔模式 |
| `--list-models [search]` | 列出可用模型 |

### 会话选项

| 选项 | 说明 |
|--------|-------------|
| `-c`、`--continue` | 继续最近的会话 |
| `-r`、`--resume` | 浏览并选择会话 |
| `--session <path\|id>` | 使用指定的会话文件或部分 UUID |
| `--fork <path\|id>` | 将会话文件或部分 UUID 分叉到新会话 |
| `--session-dir <dir>` | 自定义会话存储目录 |
| `--no-session` | 临时模式；不保存 |
| `--name <name>`、`-n <name>` | 启动时设置会话显示名称 |

### 工具选项

| 选项 | 说明 |
|--------|-------------|
| `--tools <list>`、`-t <list>` | 仅允许指定的内置、扩展和自定义工具 |
| `--exclude-tools <list>`、`-xt <list>` | 禁用指定的内置、扩展和自定义工具 |
| `--no-builtin-tools`、`-nbt` | 禁用内置工具，但保留扩展/自定义工具 |
| `--no-tools`、`-nt` | 禁用所有工具 |

内置工具：`read`、`bash`、`powershell`（Windows）、`edit`、`write`、`grep`、`find`、`ls`。

### 资源选项

| 选项 | 说明 |
|--------|-------------|
| `-e`、`--extension <source>` | 从路径、npm 或 git 加载扩展；可重复指定 |
| `--no-extensions` | 禁用扩展发现 |
| `--skill <path>` | 加载技能；可重复指定 |
| `--no-skills` | 禁用技能发现 |
| `--prompt-template <path>` | 加载提示模板；可重复指定 |
| `--no-prompt-templates` | 禁用提示模板发现 |
| `--theme <path>` | 加载主题；可重复指定 |
| `--no-themes` | 禁用主题发现 |
| `--no-context-files`、`-nc` | 禁用 `AGENTS.md` 和 `CLAUDE.md` 发现 |

将 `--no-*` 与显式 flag 结合，可忽略设置并仅加载所需内容。示例：

```bash
pi --no-extensions -e ./my-extension.ts
```

### 其他选项

| 选项 | 说明 |
|--------|-------------|
| `--system-prompt <text>` | 替换默认提示；上下文文件和技能仍会追加 |
| `--append-system-prompt <text>` | 追加到系统提示 |
| `--tui-mode <mode>` | TUI 模式：`regular`（默认）或实验性的 `fullscreen` |
| `--use-theme <name[/name]>` | 设置本次运行的初始交互主题，不更改设置 |
| `--verbose` | 强制显示详细启动信息 |
| `-a`、`--approve` | 本次运行信任项目本地文件 |
| `-na`、`--no-approve` | 本次运行忽略项目本地文件 |
| `--` | 停止解析选项；剩余参数作为提示或 `@file` 输入 |
| `-h`、`--help` | 显示帮助 |
| `-v`、`--version` | 显示版本 |

在 `fullscreen` 模式下，记录在终端视口内滚动，而排队消息、工作状态、扩展 widget、编辑器和页脚固定在底部。鼠标/触控板输入会滚动指针下方的区域；键盘视口操作始终可用。内联图像可在支持 Kitty 图形协议的终端（包括 Kitty 和 Ghostty）中工作。在 iTerm2 中，它们会渲染为文本占位符，因为应用程序自行控制滚动时，iTerm2 的内联图像协议无法删除或裁剪图像。在 `regular` 模式下，Pi 使用主屏幕和终端自身的回滚缓冲区，iTerm2 内联图像可以正常渲染。终端特定的设置和变通方法请参阅[终端配置](terminal-setup.zh-CN.md)。

在 `/settings` 中设置 **TUI 模式**，可以立即在 `regular` 和 `fullscreen` 之间切换，并选择未来会话的默认模式。**全屏退出输出**控制退出全屏时是打印最终记录，还是恢复之前的屏幕并仅打印会话恢复提示。

### 文件参数

在文件前添加 `@` 可将其包含在消息中：

```bash
pi @prompt.md "Answer this"
pi -p @screenshot.png "What's in this image?"
pi @code.ts @test.ts "Review these files"
```

### 示例

```bash
# 使用初始提示进入交互模式
pi "List all .ts files in src/"

# 非交互模式
pi -p "Summarize this codebase"

# 以短横线开头的提示
pi -p -- "- Summarize these points"

# 通过管道 stdin 进行非交互调用
cat README.md | pi -p "Summarize this text"

# 命名的一次性会话
pi --name "release audit" -p "Audit this repository"

# 使用不同模型
pi --provider openai --model gpt-4o "Help me refactor"

# 带提供商前缀的模型
pi --model openai/gpt-4o "Help me refactor"

# 带思考级别简写的模型
pi --model sonnet:high "Solve this complex problem"

# 限制模型循环切换范围
pi --models "claude-*,gpt-4o"

# 只读模式
pi --tools read,grep,find,ls -p "Review the code"

# 禁用一个扩展或内置工具，同时保留其他工具
pi --exclude-tools ask_question
```

## 设计原则

Pi 保持核心小巧，并将工作流特定行为交由扩展、技能、提示模板和包实现。

它有意不内置 MCP、子 agent、权限弹窗、计划模式、待办事项或后台 bash。你可以将这些工作流构建或安装为扩展或包，也可以使用容器和 tmux 等外部工具。

完整设计理由请阅读[博客文章](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)。
