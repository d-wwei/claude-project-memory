# Platform Adapters

将 `.project-memory/` 知识库注入到不同 agent 平台。

## 设计原则

- `.project-memory/` 是唯一的知识源（single source of truth）
- adapters 是薄薄一层胶水，只做路径引用，不复制内容
- 团队成员各自用不同 agent 平台，都从同一套文件读取

---

## Claude Code

注入方式：在项目 CLAUDE.md 中使用 `@` 引用。

### 生成 adapters/claude.md

```markdown
# 以下内容自动生成，勿手动编辑
# 生成时间: [timestamp]
# 由 project-memory skill 管理

@.project-memory/index.md
@.project-memory/cognitive-protocol.md
```

### 注入步骤

1. 检查项目根目录是否有 `CLAUDE.md`
2. 如果有 → 检查是否已包含两个 `@` 引用
3. 如果缺少引用 → 追加缺少的引用到文件末尾
4. 如果不存在 CLAUDE.md → 创建，内容为两行 `@` 引用
5. 注意：不要覆盖 CLAUDE.md 中已有的其他内容（用户可能有自定义规则）

### 深度文档加载

Agent 在需要时通过 Read 工具直接读取：
- `.project-memory/map.md`
- `.project-memory/conventions.md`
- `.project-memory/danger-zones.md`
- `.project-memory/progress.md`

Claude Code 的 `@` 引用只用于 index.md（自动加载）。其他文件按需读取，不自动加载。

---

## Cursor

注入方式：通过 `.cursor/rules/` 目录的 .mdc 文件。

### 生成 adapters/cursor.md

Cursor 不支持 `@` 引用外部文件，需要将 index.md 的**内容**写入规则文件。

### 注入步骤

1. 创建 `.cursor/rules/project-memory.mdc`
2. 内容：

```markdown
---
description: "Project memory - loaded for all tasks in this project"
alwaysApply: true
---

[index.md 的完整内容嵌入此处]

## 深度文档位置（按需阅读）
- .project-memory/map.md
- .project-memory/conventions.md
- .project-memory/danger-zones.md
- .project-memory/progress.md
```

### 同步策略

由于 Cursor 规则是内容嵌入（非引用），每次 `update` 修改了 index.md 后，需要同步重新生成 `.cursor/rules/project-memory.mdc`。

---

## Gemini CLI

注入方式：在 GEMINI.md 中引用（Gemini CLI 支持 `@` 语法）。

### 注入步骤

1. 检查项目根目录或 `~/.gemini/` 是否有 `GEMINI.md`
2. 追加 `@.project-memory/index.md`

与 Claude Code 逻辑相同。

---

## Codex CLI / OpenCode / OpenClaw

注入方式：在各自的 AGENTS.md 中嵌入 index.md 内容。

### 注入步骤

1. 找到对应配置文件（`~/.codex/AGENTS.md` / `~/.config/opencode/AGENTS.md` / `~/.openclaw/workspace/AGENTS.md`）
2. 嵌入 index.md 内容（用 `<!-- PROJECT-MEMORY:BEGIN -->` / `<!-- PROJECT-MEMORY:END -->` 标记包裹，方便更新时替换）

---

## 通用 Agent（无特定平台）

如果 agent 平台不在上述列表中，提供以下指引：

1. 找到 agent 的系统提示词或启动配置文件
2. 在其中加入："在开始工作前，先读取项目根目录下的 `.project-memory/index.md` 文件"
3. 或者直接将 index.md 内容粘贴到系统提示词中

---

## inject 命令逻辑

`/project-memory inject <platform>` 执行流程：

1. 验证 `.project-memory/index.md` 存在
2. 根据 platform 参数选择对应适配策略
3. 执行注入
4. 报告注入位置和方式
5. 如果 platform = "all"，遍历所有检测到的平台配置文件并注入

支持的 platform 值：`claude` / `cursor` / `gemini` / `codex` / `opencode` / `openclaw` / `all`
