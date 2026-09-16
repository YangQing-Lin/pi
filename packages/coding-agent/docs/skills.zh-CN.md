> Pi 可以创建技能。让它为你的使用场景构建一个技能即可。

# 技能

技能是 agent 按需加载的自包含能力包。技能为特定任务提供专门的工作流、配置说明、辅助脚本和参考文档。

Pi 实现了 [Agent Skills 标准](https://agentskills.io/specification)。它会对大多数违规情况发出警告，但仍保持宽松。尽管标准不允许技能名称与父目录不同，Pi 仍允许这样做；因为对于由多个 agent 执行框架共享的技能目录，该规则并不理想。

## 目录

- [位置](#位置)
- [技能如何工作](#技能如何工作)
- [技能命令](#技能命令)
- [技能结构](#技能结构)
- [Frontmatter](#frontmatter)
- [校验](#校验)
- [示例](#示例)
- [技能仓库](#技能仓库)

## 位置

> **安全：** 技能可以指示模型执行任何操作，也可能包含由模型调用的可执行代码。使用前请审查技能内容。

Pi 从以下位置加载技能：

- 全局：
  - `~/.pi/agent/skills/`
  - `~/.agents/skills/`
- 项目（仅在项目受信任后）：
  - `.pi/skills/`
  - `cwd` 和祖先目录中的 `.agents/skills/`（最多到 git 仓库根目录；不在仓库中时最多到文件系统根目录）
- 包：`skills/` 目录或 `package.json` 中的 `pi.skills` 条目
- 设置：包含文件或目录的 `skills` 数组
- CLI：`--skill <path>`（可重复指定；即使使用 `--no-skills` 也会追加）

发现规则：

- 在 `~/.pi/agent/skills/` 和 `.pi/skills/` 中，具有有效技能 frontmatter 且 `description` 非空的根目录直属 `.md` 文件会被发现为独立技能
- 在所有技能位置中，包含 `SKILL.md` 的目录都会被递归发现
- 在 `~/.agents/skills/` 和项目 `.agents/skills/` 中，根目录直属 `.md` 文件会被忽略；但分组文件夹中的嵌套 `.md` 文件如果声明了技能 frontmatter，则会被发现
- 除 `SKILL.md` 之外，看起来不像技能的根目录 Markdown 文件会被静默忽略

使用 `--no-skills` 禁用自动发现（显式指定的 `--skill` 路径仍会加载）。

### 使用其他执行框架的技能

要使用 Claude Code 或 OpenAI Codex 的技能，请将其目录添加到设置：

```json
{
  "skills": [
    "~/.claude/skills",
    "~/.codex/skills"
  ]
}
```

对于项目级 Claude Code 技能，请添加到 `.pi/settings.json`：

```json
{
  "skills": ["../.claude/skills"]
}
```

## 技能如何工作

1. Pi 启动时扫描技能位置，并提取名称和说明
2. 系统提示按照[规范](https://agentskills.io/integrate-skills)以 XML 格式包含可用技能
3. 当任务匹配时，agent 使用 `read` 加载完整的 SKILL.md；如果 `read` 不可用，则使用 `bash`（模型并不总会这样做；可以通过提示或 `/skill:name` 强制加载）
4. agent 遵循说明，并使用相对路径引用脚本和资源

这称为渐进式披露：只有说明始终位于上下文中，完整指令则按需加载。

## 技能命令

技能会注册为 `/skill:name` 命令：

```bash
/skill:brave-search           # 加载并执行技能
/skill:pdf-tools extract      # 加载技能并传入参数
```

命令后的参数会以 `User: <args>` 的形式追加到技能内容中。

通过交互模式中的 `/settings` 或 `settings.json` 切换技能命令：

```json
{
  "enableSkillCommands": true
}
```

## 技能结构

技能是包含 `SKILL.md` 文件的目录，其他内容不受限制。

```
my-skill/
├── SKILL.md              # 必需：frontmatter + 指令
├── scripts/              # 辅助脚本
│   └── process.sh
├── references/           # 按需加载的详细文档
│   └── api-reference.md
└── assets/
    └── template.json
```

### SKILL.md 格式

````markdown
---
name: my-skill
description: 此技能的作用以及何时使用。请具体说明。
---

# 我的技能

## 配置

首次使用前运行一次：
```bash
cd /path/to/skill && npm install
```

## 用法

```bash
./scripts/process.sh <input>
```
````

请使用相对于技能目录的路径：

```markdown
详情请参阅[参考指南](references/REFERENCE.md)。
```

## Frontmatter

根据 [Agent Skills 规范](https://agentskills.io/specification#frontmatter-required)：

| 字段 | 必需 | 说明 |
|-------|----------|-------------|
| `name` | 是 | 最多 64 个字符。仅限小写 a-z、0-9 和连字符。与标准不同，Pi 不要求它与父目录匹配，因为该标准要求不适合共享技能目录。 |
| `description` | 是 | 最多 1024 个字符。说明技能的作用和使用时机。 |
| `license` | 否 | 许可证名称或对捆绑文件的引用。 |
| `compatibility` | 否 | 最多 500 个字符。环境要求。 |
| `metadata` | 否 | 任意键值映射。 |
| `allowed-tools` | 否 | 以空格分隔的预批准工具列表（实验性）。 |
| `disable-model-invocation` | 否 | 为 `true` 时，技能不在系统提示中显示。用户必须使用 `/skill:name`。 |

### 名称规则

- 1–64 个字符
- 只能使用小写字母、数字和连字符
- 不能以连字符开头或结尾
- 不能包含连续的连字符

Pi 不要求名称与父目录匹配。Agent Skills 标准有此要求，但它不适合由多个工具共享的技能目录。

有效：`pdf-processing`、`data-analysis`、`code-review`
无效：`PDF-Processing`、`-pdf`、`pdf--processing`

### 说明最佳实践

说明决定 agent 何时加载技能。请具体说明。

良好示例：

```yaml
description: 从 PDF 文件中提取文本和表格、填写 PDF 表单并合并多个 PDF。处理 PDF 文档时使用。
```

不佳示例：

```yaml
description: 帮助处理 PDF。
```

## 校验

Pi 会按照 Agent Skills 标准校验技能。大多数问题会产生警告，但技能仍会加载：

- 名称超过 64 个字符或包含无效字符
- 名称以连字符开头/结尾，或包含连续连字符
- 说明超过 1024 个字符

未知 frontmatter 字段会被忽略。

声明为技能但缺少说明的文件不会加载。格式错误的 `SKILL.md` 文件和没有说明的 `SKILL.md` 文件会产生警告，并且不会加载。其他不含有效技能 frontmatter 的 Markdown 文件会被忽略。

发生名称冲突（来自不同位置的技能同名）时会发出警告，并保留第一个发现的技能。

## 示例

```
brave-search/
├── SKILL.md
├── search.js
└── content.js
```

**SKILL.md：**

````markdown
---
name: brave-search
description: 通过 Brave Search API 搜索网页和提取内容。搜索文档、事实或任何网页内容时使用。
---

# Brave Search

## 配置

```bash
cd /path/to/brave-search && npm install
```

## 搜索

```bash
./search.js "query"              # 基础搜索
./search.js "query" --content    # 包含页面内容
```

## 提取页面内容

```bash
./content.js https://example.com
```
````

## 技能仓库

- [Anthropic Skills](https://github.com/anthropics/skills)——文档处理（docx、pdf、pptx、xlsx）和 Web 开发
- [Pi Skills](https://github.com/badlogic/pi-skills)——Web 搜索、浏览器自动化、Google API 和转录
