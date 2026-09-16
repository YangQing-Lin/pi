# 开发

其他开发规范请参阅 [AGENTS.md](https://github.com/earendil-works/pi/blob/main/AGENTS.md)。

## 环境配置

```bash
git clone https://github.com/earendil-works/pi
cd pi
npm install
npm run build
```

从源码运行：

```bash
/path/to/pi/pi-test.sh
```

该脚本可以从任意目录运行。Pi 会保留调用者的当前工作目录。

### 实验性远程执行框架

远程执行框架的服务端/客户端集成仅用于开发。请在仓库中运行：

```bash
PI_EXPERIMENTAL=1 ./pi-test.sh server
PI_EXPERIMENTAL=1 ./pi-test.sh client
```

`PI_SERVER_DIR` 用于覆盖服务端配置和套接字目录（默认为 `~/.pi/server`）。未指定 `--server-id` 时，`PI_SERVER_ID` 用于选择逻辑服务端 ID。

在源码检出目录中，`client` 和 `experimental/plugin` 包子路径仅在 `source` 条件下可解析。它们的实现以及服务端/客户端命令不会包含在 npm 包和独立二进制文件中。`pi-client`、`pi-protocol` 和 `pi-server` 是 coding-agent 的开发依赖，而非运行时依赖。本地 SDK 和 stdio RPC API 保持不变。

## 分叉与品牌定制

通过 `package.json` 配置：

```json
{
  "piConfig": {
    "name": "pi",
    "configDir": ".pi"
  }
}
```

请为你的分叉版本修改 `name`、`configDir` 和 `bin` 字段。这些字段会影响 CLI 横幅、配置路径和环境变量名称。

## 路径解析

共有三种执行模式：通过 npm 安装、使用独立二进制文件，以及从源码通过 tsx 运行。

访问包资源时，**始终使用 `src/config.ts`**：

```typescript
import { getPackageDir, getThemeDir } from "./config.js";
```

访问包资源时，切勿直接使用 `__dirname`。

## 调试命令

隐藏命令 `/debug` 会将以下内容写入 `~/.pi/agent/pi-debug.log`：

- 包含 ANSI 代码的已渲染 TUI 行
- 最近发送给 LLM 的消息

## 测试

```bash
./test.sh                         # 运行不依赖 LLM 的测试（无需 API key）
npm test                          # 运行所有测试
npm test -- test/specific.test.ts # 运行指定测试
```

### 已发布包的冒烟测试

构建完成后，运行 `npm run check:package-install`。该命令会打包公开包，并在仓库外的临时目录中仅将 coding-agent 作为直接依赖安装。本地 tarball 覆盖项会替换部分已声明的传递依赖，但不会安装仅用于开发的包。该检查会在不使用凭据、不发起模型请求的情况下验证 SDK import 和 CLI 启动。

`npm run check` 还会检查运行时依赖声明，并拒绝因 import 而被引入包构建产物的、已排除的开发源文件。

## 项目结构

```
packages/
  ai/           # LLM 提供商抽象
  agent/        # Agent 循环和消息类型
  tui/          # 终端 UI 组件
  coding-agent/ # CLI 和交互模式
```
