# 终端配置

Pi 使用 [Kitty 键盘协议](https://sw.kovidgoyal.net/kitty/keyboard-protocol/)可靠地检测修饰键。大多数现代终端都支持该协议，但有些终端需要进行配置。

## 功能覆盖设置

Pi 会自动检测 OSC 8 超链接、内联图像协议和真彩色。如果终端代理或多路复用器妨碍了检测，请使用以下高级覆盖设置：

| 功能 | 环境变量 | JSON 设置 |
|------------|----------------------|--------------|
| OSC 8 超链接 | `PI_HYPERLINKS=1\|0\|auto` | `terminal.hyperlinks: true\|false\|"auto"` |
| 内联图像 | `PI_IMAGE_PROTOCOL=kitty\|iterm2\|none\|auto` | `terminal.images: "kitty"\|"iterm2"\|false\|"auto"` |
| 真彩色 | `PI_TRUE_COLOR=1\|0\|auto` | `terminal.trueColor: true\|false\|"auto"` |

设置的优先级高于环境变量；不设置或使用 `auto` 会保留自动检测。只能强制启用完整终端路径都支持的功能，因为不受支持的转义序列可能破坏渲染。

## Kitty

无需配置即可使用。

## iTerm2

### 常规 TUI 模式

无需配置即可使用。

### 全屏 TUI 模式

Pi 会接管视口，因此 iTerm2 会发送鼠标滚轮报告，而不是滚动原生回滚缓冲区。使用 iTerm2 默认的快速触控板行为时，这些报告可能丢失加速滚轮增量中的大部分内容，导致全屏模式的滚动速度远低于常规模式。

如果在全屏模式中快速滚动鼠标或触控板时每次只能移动大约一行：

1. 打开 **iTerm2 → Settings → Advanced**。
2. 搜索 **Trackpad scrolls fast?** 并将其设为 **No**。

这是影响整个 iTerm2 的变通方案，也可能改变原生触控板滚动行为。相关底层行为记录在 [iTerm2 issue 9619](https://gitlab.com/gnachman/iterm2/-/work_items/9619) 中。

## Apple Terminal

Pi 会在可用时启用增强按键报告。如果 Terminal.app 仍然为 `Shift+Enter` 发送普通 Return，Pi 会使用本地 macOS 修饰键回退机制，将该 Return 视为 `Shift+Enter`。

此回退机制仅在 Pi 与 Terminal.app 运行于同一台 Mac 时有效，无法通过远程 SSH 检测本地键盘。

## Ghostty

将以下内容添加到 Ghostty 配置（macOS 上为 `~/Library/Application Support/com.mitchellh.ghostty/config`，Linux 上为 `~/.config/ghostty/config`）：

```
keybind = alt+backspace=text:\x1b\x7f
```

较旧版本的 Claude Code 可能添加过以下 Ghostty 映射：

```
keybind = shift+enter=text:\n
```

该映射会发送原始换行字节。在 Pi 中，它与 `Ctrl+J` 无法区分，因此 tmux 和 Pi 无法再看到真正的 `shift+enter` 按键事件。

如果添加该映射只是为了使用 Claude Code 2.x 或更高版本，可以将它删除；但如果希望在 tmux 中使用 Claude Code，则仍然需要该 Ghostty 映射。

Pi 默认将 `Ctrl+J` 绑定为换行别名，因此通过该重映射在 tmux 中仍可使用 `Shift+Enter`，无需额外配置 Pi。

### 全屏 TUI 模式

在全屏模式中，链接仍可点击；但 Pi 捕获鼠标输入时，Ghostty 不会显示悬停下划线或左下角 URL 预览。在 macOS 上按住 `Shift+Command`，或在 Linux 上按住 `Shift+Ctrl`，即可使用 Ghostty 的原生链接处理。

## WezTerm

WezTerm 通常可以通过 xterm modifyOtherKeys 直接支持 `Shift+Enter`。要显式使用 Kitty 键盘协议，请创建 `~/.wezterm.lua`：

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.enable_kitty_keyboard = true
return config
```

在 macOS 上，WezTerm 默认将 `Option+Enter` 绑定到全屏。要用 `Option+Enter` 将消息加入 Pi 的后续队列，请添加以下按键覆盖设置：

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()
config.keys = {
  {
    key = 'Enter',
    mods = 'ALT',
    action = wezterm.action.SendString('\x1b[13;3u'),
  },
}
return config
```

如果已有 `config.keys` 表，请将该条目添加到其中。

在 WSL 上，WezTerm 可能需要显示硬件光标才能定位 IME 候选窗口。如果 CJK IME 候选项没有跟随文本光标，请在运行 Pi 前设置 `PI_HARDWARE_CURSOR=1`，或在设置中将 `showHardwareCursor` 设为 `true`。

## Alacritty

Alacritty 通常无需配置即可支持 `Shift+Enter`。在 macOS 上，`Option+Enter` 可能会被当作普通 `Enter`。要用 `Option+Enter` 将消息加入 Pi 的后续队列，请将以下内容添加到 `~/.config/alacritty/alacritty.toml`：

```toml
[[keyboard.bindings]]
key = "Enter"
mods = "Alt"
chars = "\u001b[13;3u"
```

更改配置后重启 Alacritty。

## VS Code（集成终端）

VS Code 1.109.5 及更高版本默认在集成终端中启用 Kitty 键盘协议，因此 `Shift+Enter` 应当无需配置即可使用。

低于 1.109.5 的 VS Code 版本需要为 `Shift+Enter` 显式配置终端键绑定。

`keybindings.json` 的位置：

- macOS：`~/Library/Application Support/Code/User/keybindings.json`
- Linux：`~/.config/Code/User/keybindings.json`
- Windows：`%APPDATA%\Code\User\keybindings.json`

添加到 `keybindings.json`：

```json
{
  "key": "shift+enter",
  "command": "workbench.action.terminal.sendSequence",
  "args": { "text": "\u001b[13;2u" },
  "when": "terminalFocus"
}
```

## Zed（集成终端）

将以下键绑定添加到 Zed 的 `keymap.json`：

```json
{
  "context": "Terminal",
  "bindings": {
    "shift-enter": ["terminal::SendText", "\u001b[13;2u"],
    "ctrl--": ["terminal::SendText", "\u001b[45;5u"],
    "ctrl-alt-]": ["terminal::SendText", "\u001b[93;7u"]
  }
}
```

## Windows Terminal

Pi 在原生 Windows 或 WSL 中运行时使用 Windows 风格键绑定：

- `Alt+V` 粘贴图像或剪贴板文本。
- `Ctrl+F` 在全屏模式下搜索记录，`Ctrl+Up`/`Ctrl+Down` 在已标记消息之间跳转。
- `Alt+P` 循环切换到上一个模型。
- `Ctrl+Z` 在原生 Windows 上撤销编辑；WSL 使用 `Alt+Z`，以便 `Ctrl+Z` 可以挂起 Pi。
- `Ctrl+Q` 将后续消息加入队列，`Alt+Q` 恢复已排队消息。

在 `settings.json` 中添加以下内容（按 Ctrl+Shift+,，或依次选择 Settings → Open JSON file），以转发用于插入新行的 `Shift+Enter`：

```json
{
  "actions": [
    {
      "command": { "action": "sendInput", "input": "\u001b[13;2u" },
      "keys": "shift+enter"
    }
  ]
}
```

Windows Terminal 默认将 `Alt+Enter` 绑定到全屏。要使用它代替 Pi 默认的 `Ctrl+Q` 将后续消息加入队列，请将 Windows Terminal 配置为发送该按键，并在 Pi 中将 `app.message.followUp` 绑定到 `alt+enter`。

如果已有 `actions` 数组，请将该对象添加到其中。更改设置后，请完全关闭并重新打开 Windows Terminal。

## xfce4-terminal、terminator

这些终端对转义序列的支持有限。`Ctrl+Enter` 和 `Shift+Enter` 等带修饰键的 Enter 无法与普通 `Enter` 区分，因此 `submit: ["ctrl+enter"]` 等自定义键绑定无法工作。

为获得最佳体验，请使用支持 Kitty 键盘协议的终端：

- [Kitty](https://sw.kovidgoyal.net/kitty/)
- [Ghostty](https://ghostty.org/)
- [WezTerm](https://wezfurlong.org/wezterm/)
- [iTerm2](https://iterm2.com/)
- [Alacritty](https://github.com/alacritty/alacritty)（需要在编译时启用 Kitty 协议支持）

## IntelliJ IDEA（集成终端）

内置终端对转义序列的支持有限。在 IntelliJ 终端中，Shift+Enter 无法与 Enter 区分。

如需显示硬件光标，请在运行 Pi 前设置 `PI_HARDWARE_CURSOR=1`（出于兼容性考虑，默认禁用）。

建议使用独立的终端模拟器以获得最佳体验。
