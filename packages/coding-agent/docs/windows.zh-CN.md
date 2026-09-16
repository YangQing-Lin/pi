# Windows 配置

Pi 在 Windows 上默认使用 Git Bash。它会按以下顺序检查：

1. `~/.pi/agent/settings.json` 中的自定义路径
2. Git Bash（`C:\Program Files\Git\bin\bash.exe`）
3. PATH 中的 `bash.exe`（Cygwin、MSYS2、WSL）

对大多数用户而言，安装 [Git for Windows](https://git-scm.com/download/win) 即可。

## PowerShell 工具

可选的 `powershell` 工具会在 `pwsh.exe` 可用时通过它运行命令，否则使用 Windows PowerShell。PowerShell 启动参数为 `-NoProfile -NonInteractive -ExecutionPolicy Bypass`。管理员强制执行的策略仍可能拥有更高优先级。

使用 `defaultTools` 替换面向模型的 `bash` 工具：

```json
{
  "defaultTools": ["read", "powershell", "edit", "write"]
}
```

也可以同时启用两者以比较其行为：

```json
{
  "defaultTools": ["read", "bash", "powershell", "edit", "write"]
}
```

编辑器命令 `!` 和 `!!` 仍然使用 Bash。

## 自定义 Bash 路径

```json
{
  "shellPath": "C:\\cygwin64\\bin\\bash.exe"
}
```
