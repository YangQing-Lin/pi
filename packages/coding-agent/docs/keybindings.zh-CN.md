# 键绑定

所有键盘快捷键都可以通过 `~/.pi/agent/keybindings.json` 自定义。每个操作可以绑定一个或多个按键。

配置文件使用 Pi 内部所用的 namespaced 键绑定 ID；扩展作者在 `keyHint()` 和注入的 `keybindings` 管理器中也使用这些 ID。

使用 `cursorUp` 或 `expandTools` 等旧式非 namespaced ID 的配置，会在启动时自动迁移到 namespaced ID。

编辑 `keybindings.json` 后，在 Pi 中运行 `/reload` 即可应用更改，无需重启会话。

## 按键格式

格式为 `modifier+key`。修饰键包括 `ctrl`、`shift`、`alt`、`super`（可组合），按键包括：

- **字母：** `a-z`
- **数字：** `0-9`
- **特殊键：** `escape`、`esc`、`enter`、`return`、`tab`、`space`、`backspace`、`delete`、`insert`、`clear`、`home`、`end`、`pageUp`、`pageDown`、`up`、`down`、`left`、`right`
- **功能键：** `f1`-`f12`
- **符号：** `` ` ``、`-`、`=`、`[`、`]`、`\`、`;`、`'`、`,`、`.`、`/`、`!`、`@`、`#`、`$`、`%`、`^`、`&`、`*`、`(`、`)`、`_`、`+`、`|`、`~`、`{`、`}`、`:`、`<`、`>`、`?`

修饰键组合示例：`ctrl+shift+x`、`alt+ctrl+x`、`ctrl+shift+alt+x`、`super+k`、`ctrl+super+k`、`ctrl+1` 等。

`super` 绑定要求终端能够单独报告该修饰键，通常需要 Kitty 键盘协议。不支持该功能的终端可能无法使用这些绑定。

## 所有操作

### TUI 编辑器光标移动

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `tui.editor.cursorUp` | `up` | 向上移动光标；位于顶部时浏览较早的历史记录 |
| `tui.editor.cursorDown` | `down` | 向下移动光标；位于底部时浏览较新的历史记录 |
| `tui.editor.historyPrevious` | *（无）* | 选择上一条提示历史记录 |
| `tui.editor.historyNext` | *（无）* | 选择下一条提示历史记录 |
| `tui.editor.cursorLeft` | `left`、`ctrl+b` | 向左移动光标 |
| `tui.editor.cursorRight` | `right`、`ctrl+f` | 向右移动光标 |
| `tui.editor.cursorWordLeft` | `alt+left`、`ctrl+left`、`alt+b` | 向左移动一个单词 |
| `tui.editor.cursorWordRight` | `alt+right`、`ctrl+right`、`alt+f` | 向右移动一个单词 |
| `tui.editor.cursorLineStart` | `home`、`ctrl+home`、`ctrl+a` | 移动到行首 |
| `tui.editor.cursorLineEnd` | `end`、`ctrl+end`、`ctrl+e` | 移动到行尾 |
| `tui.editor.jumpForward` | `ctrl+]` | 向前跳转到字符 |
| `tui.editor.jumpBackward` | `ctrl+alt+]` | 向后跳转到字符 |
| `tui.editor.pageUp` | `pageUp`、`ctrl+pageUp` | 向上翻页 |
| `tui.editor.pageDown` | `pageDown`、`ctrl+pageDown` | 向下翻页 |

专用历史记录操作始终切换历史条目，不受多行提示中光标位置的影响。当主编辑器获得焦点时，显式历史记录绑定优先于应用操作；因此，将 `tui.editor.historyPrevious` 绑定到 `ctrl+p` 会在该上下文中覆盖模型循环切换，但不会更改选择器中的 `Ctrl+P`。

### TUI 编辑器删除

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `tui.editor.deleteCharBackward` | `backspace` | 向后删除字符 |
| `tui.editor.deleteCharForward` | `delete`、`ctrl+d` | 向前删除字符 |
| `tui.editor.deleteWordBackward` | `ctrl+w`、`alt+backspace` | 向后删除单词 |
| `tui.editor.deleteWordForward` | `alt+d`、`alt+delete` | 向前删除单词 |
| `tui.editor.deleteToLineStart` | `ctrl+u` | 删除到行首 |
| `tui.editor.deleteToLineEnd` | `ctrl+k` | 删除到行尾 |

### TUI 输入

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `tui.input.newLine` | `shift+enter`、`ctrl+j` | 插入新行 |
| `tui.input.submit` | `enter` | 提交输入 |
| `tui.input.tab` | `tab` | Tab / 自动补全 |

### TUI Kill Ring

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `tui.editor.yank` | `ctrl+y` | 粘贴最近删除的文本 |
| `tui.editor.yankPop` | `alt+y` | yank 后循环切换已删除文本 |
| `tui.editor.undo` | `ctrl+-`（Windows 上为 `ctrl+z`；WSL 上为 `alt+z`） | 撤销上一次编辑 |

### TUI 剪贴板和选择

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `tui.input.copy` | `ctrl+c` | 复制所选内容 |
| `tui.select.up` | `up` | 向上移动选择项 |
| `tui.select.down` | `down` | 向下移动选择项 |
| `tui.select.pageUp` | `pageUp` | 在列表中向上翻页 |
| `tui.select.pageDown` | `pageDown` | 在列表中向下翻页 |
| `tui.select.confirm` | `enter` | 确认选择 |
| `tui.select.cancel` | `escape`、`ctrl+c` | 取消选择 |

### TUI 全屏视口

这些操作在交互模式使用 `--tui-mode fullscreen` 时生效，目标是主要记录滚动区域。双指触控板和鼠标滚轮输入会滚动指针下方的区域；在固定的编辑器/状态/页脚停靠区上方则回退到记录区域。点击 OSC 8 超链接会通过默认处理程序打开。按住鼠标主键拖动会选择文本并复制到剪贴板；在记录顶部或底部边缘持续按住会自动滚动到屏幕外内容。当记录向上滚动后，其底行会显示可点击的“跳转到最新消息”标签，其中标有 `tui.altScreen.bottom` 快捷键。终端特定的鼠标和触控板行为请参阅[终端配置](terminal-setup.zh-CN.md)。

全屏记录绑定优先于编辑器绑定。因此，在全屏模式下，默认的无修饰导航键控制记录，其 `ctrl` 变体仍控制编辑器。在全屏模式之外，两种变体都控制编辑器。

记录搜索面板会显示已配置的上一个/下一个快捷键和可点击箭头控件。再次按 `tui.altScreen.search` 或使用 `tui.altScreen.searchClose` 可以关闭它。

| 按键 | 默认模式 | 全屏模式 |
|-----|--------------|-----------------|
| `home`、`end` | 编辑器 | 记录 |
| `ctrl+home`、`ctrl+end` | 编辑器 | 编辑器 |
| `pageUp`、`pageDown` | 编辑器 | 记录 |
| `ctrl+pageUp`、`ctrl+pageDown` | 编辑器 | 编辑器 |

此路由仍可通过普通操作绑定进行配置。例如，`"tui.altScreen.pageUp": "ctrl+pageUp"` 会让 `pageUp` 在全屏模式中控制编辑器，而 `ctrl+pageUp` 控制记录。绑定 `tui.altScreen.halfPageUp` 和 `tui.altScreen.halfPageDown` 可按半页移动；绑定 `tui.altScreen.lineUp` 和 `tui.altScreen.lineDown` 可按单行移动。设置 `"tui.altScreen.pageUp": []` 会完全禁用该记录快捷键。用户绑定会替换该操作的默认值。

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `tui.altScreen.pageUp` | `pageUp` | 将记录向上滚动一页 |
| `tui.altScreen.pageDown` | `pageDown` | 将记录向下滚动一页 |
| `tui.altScreen.halfPageUp` | *（无）* | 将记录向上滚动半页 |
| `tui.altScreen.halfPageDown` | *（无）* | 将记录向下滚动半页 |
| `tui.altScreen.lineUp` | *（无）* | 将记录向上滚动一行 |
| `tui.altScreen.lineDown` | *（无）* | 将记录向下滚动一行 |
| `tui.altScreen.previousPrompt` | `ctrl+shift+up`、`ctrl+up`（仅 Windows 和 WSL 使用 `ctrl+up`） | 跳转到上一条已标记消息 |
| `tui.altScreen.nextPrompt` | `ctrl+shift+down`、`ctrl+down`（仅 Windows 和 WSL 使用 `ctrl+down`） | 跳转到下一条已标记消息 |
| `tui.altScreen.search` | `ctrl+shift+f`（Windows 和 WSL 上为 `ctrl+f`） | 搜索已渲染的记录 |
| `tui.altScreen.searchNext` | `enter`、`ctrl+g` | 搜索时选择下一个匹配项 |
| `tui.altScreen.searchPrevious` | `shift+enter`、`ctrl+shift+g` | 搜索时选择上一个匹配项 |
| `tui.altScreen.searchClose` | `escape` | 关闭记录搜索 |
| `tui.altScreen.top` | `home` | 滚动到记录开头 |
| `tui.altScreen.bottom` | `end` | 滚动到记录末尾并跟随新输出 |

### 应用

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `app.interrupt` | `escape` | 取消 / 中止 |
| `app.clear` | `ctrl+c` | 第一次清空编辑器 / 第二次退出 |
| `app.exit` | `ctrl+d` | 退出（编辑器为空时） |
| `app.suspend` | `ctrl+z`（Windows 上无） | 挂起到后台 |
| `app.editor.external` | `ctrl+g` | 在外部编辑器中打开（`externalEditor`、`$VISUAL`、`$EDITOR`、Windows 上的 Notepad 或其他平台上的 `nano`） |
| `app.clipboard.pasteImage` | `ctrl+v`（Windows 和 WSL 上为 `alt+v`） | 从剪贴板粘贴图像或文本 |

### 会话

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `app.session.new` | *（无）* | 启动新会话（`/new`） |
| `app.session.tree` | *（无）* | 打开会话树导航器（`/tree`） |
| `app.session.fork` | *（无）* | 分叉当前会话（`/fork`） |
| `app.session.resume` | *（无）* | 打开会话恢复选择器（`/resume`） |
| `app.session.togglePath` | `ctrl+p` | 切换路径显示 |
| `app.session.toggleSort` | `ctrl+s` | 切换排序模式 |
| `app.session.toggleNamedFilter` | `ctrl+n` | 切换仅显示已命名会话的筛选器 |
| `app.session.rename` | `ctrl+r` | 重命名会话 |
| `app.session.delete` | `ctrl+d` | 删除会话 |
| `app.session.deleteNoninvasive` | `ctrl+backspace` | 查询为空时删除会话 |

### 模型和思考

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `app.model.select` | `ctrl+l` | 打开模型选择器 |
| `app.model.cycleForward` | `ctrl+p` | 循环切换到下一个模型 |
| `app.model.cycleBackward` | `shift+ctrl+p`（Windows 和 WSL 上为 `alt+p`） | 循环切换到上一个模型 |
| `app.models.save` | `ctrl+s` | 将选定的默认模型或范围模型配置保存到设置 |
| `app.thinking.cycle` | `shift+tab` | 循环切换思考级别 |
| `app.thinking.save` | `ctrl+s` | 将当前思考级别保存到设置 |
| `app.thinking.toggle` | `ctrl+t` | 折叠或展开思考块 |

### 显示和消息队列

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `app.tools.expand` | `ctrl+o` | 折叠或展开工具输出 |
| `app.message.copy` | `ctrl+x` | 在 `/tree` 中复制所选消息；其他情况下复制最后一条 assistant 消息，或在禁用 `fullscreenCopyOnSelect` 时复制活动的全屏文本选择 |
| `app.message.followUp` | `alt+enter`（Windows 和 WSL 上为 `ctrl+q`） | 将后续消息加入队列 |
| `app.message.dequeue` | `alt+up`（Windows 和 WSL 上为 `alt+q`） | 将已排队消息恢复到编辑器 |

### 树导航

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `app.tree.foldOrUp` | `ctrl+left`、`alt+left` | 折叠当前分支片段，或跳转到上一片段开头 |
| `app.tree.unfoldOrDown` | `ctrl+right`、`alt+right` | 展开当前分支片段，或跳转到下一片段开头或分支末尾 |
| `app.tree.editLabel` | `shift+l` | 编辑所选树节点的标签 |
| `app.tree.toggleLabelTimestamp` | `shift+t` | 切换树中的标签时间戳 |
| `app.tree.filter.default` | `ctrl+d` | 将树筛选器设为默认视图 |
| `app.tree.filter.noTools` | `ctrl+t` | 切换隐藏工具结果的树筛选器 |
| `app.tree.filter.userOnly` | `ctrl+u` | 切换仅显示用户消息的树筛选器 |
| `app.tree.filter.labeledOnly` | `ctrl+l` | 切换仅显示有标签条目的树筛选器 |
| `app.tree.filter.all` | `ctrl+a` | 切换显示所有条目的树筛选器 |
| `app.tree.filter.cycleForward` | `ctrl+o` | 向前循环切换树筛选器 |
| `app.tree.filter.cycleBackward` | `shift+ctrl+o` | 向后循环切换树筛选器 |

### 范围模型选择器

在范围模型选择器（通过 `/scoped-models` 打开）中使用。

| 键绑定 ID | 默认值 | 说明 |
|--------|---------|-------------|
| `app.models.enableAll` | `ctrl+a` | 启用所有模型（或与当前搜索匹配的所有模型） |
| `app.models.clearAll` | `ctrl+x` | 清除所有模型（或与当前搜索匹配的所有模型） |
| `app.models.toggleProvider` | `ctrl+p` | 切换当前提供商的所有模型 |
| `app.models.reorderUp` | `alt+up` | 将所选模型在循环顺序中上移 |
| `app.models.reorderDown` | `alt+down` | 将所选模型在循环顺序中下移 |

## 自定义配置

创建 `~/.pi/agent/keybindings.json`：

```json
{
  "tui.editor.historyPrevious": "ctrl+p",
  "tui.editor.historyNext": "ctrl+n",
  "tui.editor.deleteWordBackward": ["ctrl+w", "alt+backspace"]
}
```

每个操作可以使用单个按键或按键数组。用户配置会覆盖默认值。

在原生 Windows 上，`app.suspend` 没有默认绑定，因为 Windows 终端不支持 Unix 作业控制。如果手动绑定，Pi 会显示状态消息而不是挂起。在 WSL 中，正常的 Linux `ctrl+z`/`fg` 行为仍然有效。

### Emacs 示例

```json
{
  "tui.editor.historyPrevious": "ctrl+p",
  "tui.editor.historyNext": "ctrl+n",
  "tui.editor.cursorLeft": ["left", "ctrl+b"],
  "tui.editor.cursorRight": ["right", "ctrl+f"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+f"],
  "tui.editor.deleteCharForward": ["delete", "ctrl+d"],
  "tui.editor.deleteCharBackward": ["backspace", "ctrl+h"],
  "tui.input.newLine": ["shift+enter", "ctrl+j"]
}
```

### Vim 示例

```json
{
  "tui.editor.cursorUp": ["up", "alt+k"],
  "tui.editor.cursorDown": ["down", "alt+j"],
  "tui.editor.cursorLeft": ["left", "alt+h"],
  "tui.editor.cursorRight": ["right", "alt+l"],
  "tui.editor.cursorWordLeft": ["alt+left", "alt+b"],
  "tui.editor.cursorWordRight": ["alt+right", "alt+w"]
}
```
