> Pi 可以帮助你创建 Pi 包。让它打包你的扩展、技能、提示模板或主题即可。

# Pi 包

Pi 包将扩展、技能、提示模板和主题捆绑在一起，方便通过 npm 或 git 分享。包可以在 `package.json` 的 `pi` key 下声明资源，也可以使用约定目录。

## 目录

- [安装和管理](#安装和管理)
- [包来源](#包来源)
- [创建 Pi 包](#创建-pi-包)
- [包结构](#包结构)
- [依赖](#依赖)
- [包筛选](#包筛选)
- [启用和禁用资源](#启用和禁用资源)
- [作用域和去重](#作用域和去重)

## 安装和管理

> **安全：** Pi 包拥有完整的系统访问权限。扩展可以执行任意代码，技能可以指示模型执行任何操作，包括运行可执行文件。安装第三方包前请审查源代码。

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install https://github.com/user/repo  # 也支持原始 URL
pi install /absolute/path/to/package
pi install ./relative/path/to/package

pi remove npm:@foo/bar
pi list                     # 显示设置中已安装的包
pi update                   # 仅更新 Pi
pi update --all             # 更新 Pi、更新包并协调固定的 git ref
pi update --extensions      # 仅更新包并协调固定的 git ref
pi update --models          # 仅刷新模型目录
pi update --self            # 仅更新 Pi
pi update --self --force    # 即使已是当前版本也重新安装 Pi
pi update npm:@foo/bar      # 更新一个包
pi update --extension npm:@foo/bar
```

这些命令用于管理 Pi 包，`pi update` 还可以更新 Pi CLI 安装。对于由实验性安装程序管理的安装，`pi update` 会将经过检查的确切版本安装到由 lockfile 支持的暂存版本中，并且只在验证后激活；如果更新失败，当前版本保持不变。托管安装不支持 `--force`；如需修复，请重新运行安装程序。要卸载 Pi 本身，请参阅[快速入门](quickstart.zh-CN.md#卸载)。

默认情况下，`install` 和 `remove` 写入用户设置（`~/.pi/agent/settings.json`）。使用 `-l` 可改为写入项目设置（`.pi/settings.json`）。项目设置可以与团队共享；项目受信任后，Pi 会在启动时自动安装缺失的包。

要在不安装的情况下试用包，请使用 `--extension` 或 `-e`。它只会为当前运行安装到临时目录：

```bash
pi -e npm:@foo/bar
pi -e git:github.com/user/repo
```

## 包来源

Pi 在设置和 `pi install` 中接受三种来源类型。

### npm

```
npm:@scope/pkg@1.2.3
npm:pkg
```

- 带版本的 spec 会被固定，包更新（`pi update --extensions`、`pi update --all`）会跳过它们。
- 用户安装位于 `~/.pi/agent/npm/`。
- 项目安装位于 `.pi/npm/`。
- 在 `settings.json` 中设置 `npmCommand`，可将 npm 包查询和安装操作固定到 `mise` 或 `asdf` 等指定包装命令。

示例：

```json
{
  "npmCommand": ["mise", "exec", "node@20", "--", "npm"]
}
```

### git

```
git:github.com/user/repo@v1
git:git@github.com:user/repo@v1
https://github.com/user/repo@v1
ssh://git@github.com/user/repo@v1
```

- 不带 `git:` 前缀时，只接受协议 URL（`https://`、`http://`、`ssh://`、`git://`）。
- 带 `git:` 前缀时，支持简写格式，包括 `github.com/user/repo` 和 `git@github.com:user/repo`。
- 同时支持 HTTPS 和 SSH URL。
- SSH URL 会自动使用已配置的 SSH key（遵循 `~/.ssh/config`）。
- 对于非交互运行（例如 CI），可以设置 `GIT_TERMINAL_PROMPT=0` 禁用凭据提示，并设置 `GIT_SSH_COMMAND`（例如 `ssh -o BatchMode=yes -o ConnectTimeout=5`）以快速失败。
- ref 是固定的 tag 或 commit。`pi update --extensions` 和 `pi update --all` 不会将它们移动到较新的 ref，但会将现有 clone 协调到配置的 ref。
- 使用 `pi install git:host/user/repo@new-ref` 更新设置，并将现有包移动到新的固定 ref。
- clone 到 `~/.pi/agent/git/<host>/<path>`（全局）或 `.pi/git/<host>/<path>`（项目）。
- 协调操作更改 checkout 时，Pi 会重置并清理 clone，然后在存在 `package.json` 时运行 `npm install`。

**SSH 示例：**

```bash
# git@host:path 简写（需要 git: 前缀）
pi install git:git@github.com:user/repo

# ssh:// 协议格式
pi install ssh://git@github.com/user/repo

# 带版本 ref
pi install git:git@github.com:user/repo@v1.0.0
```

### 本地路径

```
/absolute/path/to/package
./relative/path/to/package
```

本地路径指向磁盘上的文件或目录，并直接添加到设置中，不会复制。相对路径根据其所在的设置文件解析。如果路径是文件，它会作为单个扩展加载；如果是目录，Pi 会按包规则加载资源。

## 创建 Pi 包

在 `package.json` 中添加 `pi` 清单，或使用约定目录。加入 `pi-package` 关键字可提高可发现性。

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

路径相对于包根目录。数组支持 glob 模式和 `!exclusions`。清单中的正向 glob 会按词法顺序发现可见路径。以点开头的路径需直接列出。如果 glob 需要穿过符号链接继续搜索，请直接列出符号链接指向的资源根目录。

### Gallery 元数据

[包 gallery](https://pi.dev/packages) 会显示带有 `pi-package` 标签的包。添加 `video` 或 `image` 字段可显示预览：

```json
{
  "name": "my-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "video": "https://example.com/demo.mp4",
    "image": "https://example.com/screenshot.png"
  }
}
```

- **video**：仅支持 MP4。在桌面端悬停时自动播放，点击可打开全屏播放器。
- **image**：支持 PNG、JPEG、GIF 或 WebP，显示为静态预览。

如果两者都设置，video 优先。

## 包结构

### 约定目录

如果不存在 `pi` 清单，Pi 会从以下目录自动发现资源：

- `extensions/` 加载 `.ts` 和 `.js` 文件
- `skills/` 递归查找包含 `SKILL.md` 的文件夹，并将顶层 `.md` 文件作为技能加载
- `prompts/` 加载 `.md` 文件
- `themes/` 加载 `.json` 文件

## 依赖

第三方运行时依赖应放入 `package.json` 的 `dependencies`。不会注册扩展、技能、提示模板或主题的依赖也应放入 `dependencies`。Pi 从 npm 或 git 安装包时会运行 `npm install`，因此这些依赖会自动安装。

Pi 为扩展和技能捆绑了核心包。如果导入以下任何包，请在 `peerDependencies` 中以 `"*"` 范围列出，并且不要捆绑它们：`@earendil-works/pi-ai`、`@earendil-works/pi-agent-core`、`@earendil-works/pi-coding-agent`、`@earendil-works/pi-tui`、`typebox`。

其他 Pi 包必须捆绑在你的 tarball 中。将它们添加到 `dependencies` 和 `bundledDependencies`，然后通过 `node_modules/` 路径引用其资源。Pi 使用不同的模块根加载包，因此不同安装之间不会冲突或共享模块。

示例：

```json
{
  "dependencies": {
    "shitty-extensions": "^1.0.1"
  },
  "bundledDependencies": ["shitty-extensions"],
  "pi": {
    "extensions": ["extensions", "node_modules/shitty-extensions/extensions"],
    "skills": ["skills", "node_modules/shitty-extensions/skills"]
  }
}
```

## 包筛选

在设置中使用对象形式筛选包加载的内容：

```json
{
  "packages": [
    "npm:simple-pkg",
    {
      "source": "npm:my-package",
      "extensions": ["extensions/*.ts", "!extensions/legacy.ts"],
      "skills": [],
      "prompts": ["prompts/review.md"],
      "themes": ["+themes/legacy.json"]
    }
  ]
}
```

`+path` 和 `-path` 是相对于包根目录的精确路径。

- 省略 key 会加载该类型的全部资源。
- 使用 `[]` 不加载该类型的任何资源。
- `!pattern` 排除匹配项。
- `+path` 强制包含精确路径。
- `-path` 强制排除精确路径。
- 筛选器叠加在清单之上，只会缩小已允许内容的范围。

## 启用和禁用资源

使用 `pi config` 启用或禁用来自已安装包和本地目录的扩展、技能、提示模板与主题。`pi config` 默认从全局设置（`~/.pi/agent/settings.json`）启动；按 Tab 可在全局和项目本地模式之间切换。使用 `pi config -l` 可从项目覆盖设置（`.pi/settings.json`）启动，并以暗色显示继承的全局资源。

## 作用域和去重

包可以同时出现在全局设置和项目设置中。如果同一个包同时出现，项目条目优先；但如果项目条目的 `autoload: false`，则会将其作为增量应用到全局条目上。身份判定方式如下：

- npm：包名
- git：不含 ref 的仓库 URL
- local：解析后的绝对路径
