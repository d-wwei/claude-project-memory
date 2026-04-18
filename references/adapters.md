# Platform Adapters

将 `.better-work/` 知识库注入到不同 agent 平台。

## 设计原则

- `~/.better-work/<project>/` 是唯一的知识源（single source of truth）
- 项目中 `<project>/.better-work/` 是**符号链接**，不要创建实体目录
- adapters 是薄胶水层，只做引用/嵌入，不复制内容
- 团队成员各自用不同 agent 平台，都从同一套文件读取

## 注入的两个文件

无论哪个平台，最少注入这两项：

```
@.better-work/shared/index.md        ← 项目知识（共享）
@.better-work/code/protocol.md       ← 研发认知约束
```

如果同时装了 better-work，会再加一项 `@.better-work/protocol.md`（通用执行标准）。better-code 不管那一项——由 better-work 自己注入。

---

## Claude Code

注入方式：项目 `CLAUDE.md` 中用 `@` 引用。

### 注入步骤

1. 检查项目根目录是否有 `CLAUDE.md`
2. 如果有 → 检查是否已包含这两个 `@` 引用
3. 如果缺少引用 → 追加缺少的引用到文件末尾
4. 如果不存在 `CLAUDE.md` → 创建，内容为两行 `@` 引用
5. 不要覆盖 `CLAUDE.md` 中已有的其他内容（用户可能有自定义规则）

### 深度文档加载

Agent 在需要时通过 Read 工具直接读取：
- `.better-work/shared/map.md`
- `.better-work/code/conventions.md`
- `.better-work/code/danger-zones.md`
- `.better-work/shared/progress.md`

Claude Code 的 `@` 引用只用于 index.md 和 code/protocol.md（自动加载）。其他文件按需读取。

---

## Cursor

注入方式：`.cursor/rules/` 目录的 `.mdc` 文件。

Cursor 不支持 `@` 引用外部文件，需要把内容写入规则文件。

### 注入步骤

1. 创建 `.cursor/rules/better-work.mdc`
2. 内容：

```markdown
---
description: "Project knowledge + code cognitive constraints"
alwaysApply: true
---

<!-- BEGIN: .better-work/shared/index.md -->
[index.md 的完整内容嵌入此处]
<!-- END: .better-work/shared/index.md -->

<!-- BEGIN: .better-work/code/protocol.md -->
[code/protocol.md 的完整内容嵌入此处]
<!-- END: .better-work/code/protocol.md -->

## 深度文档位置（按需阅读）
- .better-work/shared/map.md
- .better-work/code/conventions.md
- .better-work/code/danger-zones.md
- .better-work/shared/progress.md
```

### 同步策略

Cursor 是内容嵌入（非引用），每次 `update` 修改了 `shared/index.md` 或 `code/protocol.md` 后，必须同步重新生成 `.cursor/rules/better-work.mdc`。建议在 update 命令结束时自动触发。

---

## Gemini CLI

注入方式：`GEMINI.md` 中引用（Gemini CLI 支持 `@` 语法）。

### 注入步骤

1. 检查项目根目录或 `~/.gemini/` 是否有 `GEMINI.md`
2. 追加：
   ```
   @.better-work/shared/index.md
   @.better-work/code/protocol.md
   ```

逻辑与 Claude Code 相同。

---

## Codex CLI / OpenCode / OpenClaw

注入方式：各自的 `AGENTS.md` 中嵌入内容。

### 注入步骤

1. 找到对应配置文件（`~/.codex/AGENTS.md` / `~/.config/opencode/AGENTS.md` / `~/.openclaw/workspace/AGENTS.md`）
2. 用标记包裹嵌入两个文件的内容，便于 update 时替换：

```markdown
<!-- BETTER-WORK:BEGIN -->
[shared/index.md 内容]

[code/protocol.md 内容]
<!-- BETTER-WORK:END -->
```

---

## 通用 Agent（无特定平台）

如果 agent 平台不在上述列表，提供以下指引：

1. 找到 agent 的系统提示词或启动配置文件
2. 在其中加入："开始工作前，先读取项目根目录下的 `.better-work/shared/index.md` 和 `.better-work/code/protocol.md`"
3. 或者直接把两个文件的内容粘贴到系统提示词

---

## 符号链接与注册

所有平台都共享同一套文件，路径约定：

```
~/.better-work/<project>/            ← 实际存储位置（独立 git repo 或 submodule）
<project>/.better-work               ← 符号链接 → ~/.better-work/<project>/
```

符号链接由 `/better-code init` 自动创建。若链接已存在但指向错误，init 会报告冲突等待人类决策，不会自动替换。

`<project>/.gitignore` 应包含 `.better-work/`，避免把符号链接提交到项目仓库。

## 平台切换时

用户从 Claude Code 切到 Cursor（或反向）时，手动执行 `/better-code init`（detect 到 `.better-work/` 已存在会跳过知识生成，只补注入）。或者在 `update` 后自动触发（`shared/index.md` 变了 → 所有已检测到的 adapter 都刷新）。
