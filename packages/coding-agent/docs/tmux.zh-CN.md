# tmux 配置

Pi 可以在 tmux 中运行，但 tmux 默认会移除某些按键的修饰键信息。如果不进行配置，`Shift+Enter` 和 `Ctrl+Enter` 通常无法与普通 `Enter` 区分。

## 推荐配置

将以下内容添加到 `~/.tmux.conf`：

```tmux
set -g extended-keys on
set -g extended-keys-format csi-u
```

然后彻底重启 tmux：

```bash
tmux kill-server
tmux
```

当 Kitty 键盘协议不可用时，Pi 会自动请求扩展按键报告。设置 `extended-keys-format csi-u` 后，tmux 会以 CSI-u 格式转发带修饰键的按键，这是最可靠的配置。`extended-keys-format` 选项要求 tmux 3.5 或更高版本。

## 为什么推荐 `csi-u`

如果只设置：

```tmux
set -g extended-keys on
```

tmux 默认使用 `extended-keys-format xterm`。应用程序请求扩展按键报告时，带修饰键的按键会以 xterm `modifyOtherKeys` 格式转发，例如：

- `Ctrl+C` → `\x1b[27;5;99~`
- `Ctrl+D` → `\x1b[27;5;100~`
- `Ctrl+Enter` → `\x1b[27;5;13~`

使用 `extended-keys-format csi-u` 后，相同按键会被转发为：

- `Ctrl+C` → `\x1b[99;5u`
- `Ctrl+D` → `\x1b[100;5u`
- `Ctrl+Enter` → `\x1b[13;5u`

Pi 支持这两种格式，但建议为 tmux 配置 `csi-u`。

## 此配置解决的问题

如果不启用 tmux 扩展按键，带修饰键的 Enter 会退化为传统序列：

| 按键 | 未启用 extkeys | 使用 `csi-u` |
|-----|-----------------|--------------|
| Enter | `\r` | `\r` |
| Shift+Enter | `\r` | `\x1b[13;2u` |
| Ctrl+Enter | `\r` | `\x1b[13;5u` |
| Alt/Option+Enter | `\x1b\r` | `\x1b[13;3u` |

这会影响默认键绑定（`Enter` 用于提交，`Shift+Enter` 用于换行），以及所有使用带修饰键 Enter 的自定义键绑定。

## 要求

- 要使用 `extended-keys-format csi-u`，需安装 tmux 3.5 或更高版本（运行 `tmux -V` 检查）
- 支持扩展按键的终端模拟器（Ghostty、Kitty、iTerm2、WezTerm、Windows Terminal）

使用 tmux 3.2 至 3.4 时，请省略 `extended-keys-format csi-u`；Pi 仍支持 tmux 默认的 xterm `modifyOtherKeys` 格式。
