> Pi 可以创建提示模板。你可以让它为自己的工作流构建一个模板。

# 提示模板

提示模板是可以展开为完整提示的 Markdown 片段。在编辑器中输入 `/name` 即可调用模板，其中 `name` 是不含 `.md` 的文件名。

## 位置

Pi 从以下位置加载提示模板：

- 全局：`~/.pi/agent/prompts/*.md`
- 项目：`.pi/prompts/*.md`（仅在项目受信任后）
- 包：`prompts/` 目录或 `package.json` 中的 `pi.prompts` 条目
- 设置：包含文件或目录的 `prompts` 数组
- CLI：`--prompt-template <path>`（可重复指定）

使用 `--no-prompt-templates` 禁用自动发现。

## 格式

```markdown
---
description: 审查已暂存的 git 变更
---
审查已暂存的变更（`git diff --cached`），重点关注：
- bug 和逻辑错误
- 安全问题
- 错误处理缺陷
```

- 文件名会成为命令名称。`review.md` 对应 `/review`。
- `description` 可选。如未提供，则使用第一个非空行。
- `argument-hint` 可选。设置后，提示会显示在自动补全下拉菜单中的说明之前。

### 参数提示

在 frontmatter 中使用 `argument-hint`，可以在自动补全中显示预期参数。使用 `<尖括号>` 表示必需参数，使用 `[方括号]` 表示可选参数：

```markdown
---
description: 通过 URL 审查 PR，并进行结构化 issue 和代码分析
argument-hint: "<PR-URL>"
---
```

它在自动补全下拉菜单中的呈现效果如下：

```
→ pr   <PR-URL>       — 通过 URL 审查 PR，并进行结构化 issue 和代码分析
  is   <issue>        — 分析 GitHub issue（bug 或功能请求）
  wr   [instructions] — 端到端完成当前任务
  cl   — 发布前审查 changelog 条目
```

## 用法

在编辑器中输入 `/`，然后输入模板名称。自动补全会显示可用模板及其说明。

```
/review                           # 展开 review.md
/component Button                 # 展开并传入参数
/component Button "click handler" # 传入多个参数
```

## 参数

模板支持位置参数、默认值和简单切片：

- `$1`、`$2`……表示位置参数
- `$@` 或 `$ARGUMENTS` 表示拼接后的全部参数
- `${1:-default}` 在参数 1 存在且非空时使用参数 1，否则使用 `default`
- `${@:-default}` 或 `${ARGUMENTS:-default}` 在全部参数存在且非空时使用全部参数，否则使用 `default`
- `${@:N}` 表示从第 N 个位置开始的参数（从 1 开始计数）
- `${@:N:L}` 表示从 N 开始的 `L` 个参数

示例：

```markdown
---
description: 创建组件
---
创建一个名为 $1 的 React 组件，并包含以下功能：$@
```

默认值适用于可选参数：

```markdown
用 ${1:-7} 个要点总结当前状态。
```

用法：`/component Button "onClick handler" "disabled support"`

## 加载规则

- `prompts/` 中的模板发现不会递归子目录。
- 如果需要使用子目录中的模板，请通过 `prompts` 设置或包清单显式添加。
