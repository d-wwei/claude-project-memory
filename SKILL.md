---
name: claude-project-memory
description: "探索项目并生成/更新项目级 CLAUDE.md，把隐性知识显式化。"
---

# Project Memory

探索项目代码库，生成和维护项目级 CLAUDE.md，让 agent 在大项目中拥有持久的项目知识。

**核心理念**: 在大项目（10k+ 行）中，agent 的瓶颈不是"不够认真"而是"不知道"。这个 skill 把每次探索和迭代中获得的项目知识沉淀到 CLAUDE.md 中，下次对话直接可用。

**命令**:
- `/project-memory init` — 首次探索项目，生成初始 CLAUDE.md
- `/project-memory update` — 基于本次会话的工作，增量更新 CLAUDE.md
- `/project-memory learn <topic>` — 深入调研某个主题并记录到 CLAUDE.md

---

## 1. Init（首次探索）

当用户执行 `/project-memory init` 或 agent 发现当前项目没有 CLAUDE.md 时执行。

**目标**: 用最少的时间建立对项目的"大局观"，生成一份能让未来的 agent 快速上手的 CLAUDE.md。

### 1.1 探索步骤（按顺序执行）

**Step 1: 确定项目根目录和基本信息**
- 找到 git 根目录
- 读取已有的 README.md、CONTRIBUTING.md、Makefile、package.json、pyproject.toml 等
- 识别语言、框架、包管理器

**Step 2: 目录结构扫描**
- 用 `find . -type f -name "*.py" -o -name "*.ts" -o -name "*.go" | head -200` 获取文件列表
- 用 `find . -type d -maxdepth 3` 获取目录树
- 识别分层模式: 是否有 api/service/repository/model 这样的分层？
- 识别模块边界: 顶层目录各自负责什么？

**Step 3: 入口点和核心流程**
- 找到入口点: main.py、app.py、index.ts、cmd/main.go 等
- 追踪一个核心流程（如：一个 API 请求从进入到返回经过哪些层）
- 记录调用链: 入口 → 路由 → handler → service → repository → model

**Step 4: 配置和基础设施**
- 找配置文件: .env.example、config.py、settings 目录
- 找 Docker/K8s 配置: Dockerfile、docker-compose.yml、k8s/
- 找 CI 配置: .github/workflows、.gitlab-ci.yml、Jenkinsfile

**Step 5: 测试结构**
- 找测试目录和测试命令
- 尝试运行测试（先问用户是否允许）: `make test`、`npm test`、`pytest` 等
- 记录: 测试框架、怎么跑、大概要多久

**Step 6: 依赖关系和高引用文件**
- 找被最多文件 import 的模块（这些是改动高风险区域）
  ```bash
  # Python 示例
  grep -rh "^from \|^import " src/ | sort | uniq -c | sort -rn | head -20
  ```
- 找最大的文件（通常是核心逻辑所在）
  ```bash
  find src/ -name "*.py" -exec wc -l {} \; | sort -rn | head -20
  ```

**Step 7: 发现项目约定**
- 读 3-5 个不同模块的代码，提取共同模式:
  - 错误处理方式（自定义异常？错误码？）
  - 日志方式（什么库？什么格式？）
  - 数据库访问方式（ORM？raw SQL？query builder？）
  - 测试写法（用什么 fixture？mock 方式？）

### 1.2 生成 CLAUDE.md

将探索结果写入项目根目录的 CLAUDE.md（如果项目根目录已有 CLAUDE.md，写到 `.claude/` 目录下的项目级配置中）。

**模板结构**:

```markdown
# [Project Name]

## Architecture

[一段话概述：这个项目是什么、用什么技术栈、怎么部署]

### Module Map

[目录结构 + 每个顶层目录的职责说明]

调用方向: [A → B → C, 不允许反向调用]

### Core Flow

[追踪一个典型请求的完整路径，标注经过哪些文件]

## Patterns & Conventions

### [Pattern 1: 错误处理]
[具体怎么做，引用实际代码文件作为示例]

### [Pattern 2: 数据库访问]
[具体怎么做]

### [Pattern 3: 测试写法]
[具体怎么做]

### Adding a New Feature (checklist)
[基于观察到的模式，列出新增功能的标准步骤]

## Testing

[怎么跑测试、不同测试类型的命令、大致耗时]

## Danger Zones

### High-Risk Files
[被大量引用的文件，改动影响面大]

### Known Gotchas
[探索中发现的容易踩的坑]

## Useful Searches

[常用的 grep 模式，帮助未来的 agent 搜索依赖方、调用方等]
```

### 1.3 重要原则

- **只写确认过的信息**。如果没有实际读代码验证，标注"[未验证]"
- **引用具体文件路径**。不要写"在 service 层"，写"在 `src/services/order_service.py:45`"
- **宁少勿多**。只写对 agent 做开发有用的信息，不写项目历史、设计哲学等
- **不要重复 README**。README 是给人看的，CLAUDE.md 是给 agent 看的。侧重点不同

---

## 2. Update（增量更新）

当用户执行 `/project-memory update` 或 agent 完成一个较大的任务后执行。

**目标**: 把本次会话中获得的新知识沉淀到 CLAUDE.md，避免下次重复探索。

### 2.1 回顾本次会话

回顾当前对话中的所有操作，提取以下类型的新知识：

**类型 A: 新发现的模块关系**
- "原来 X 模块依赖 Y 模块，改 Y 的接口会影响 X"
- "Z 文件是被 20+ 处引用的核心模块"

**类型 B: 踩过的坑**
- "直接改 order.status 会绕过状态机"
- "这个测试需要先启动 Redis"
- "config 里的 X 选项名字有误导性，实际含义是 Y"

**类型 C: 发现的项目约定**
- "所有 API 返回值都套一层 `{"code": 0, "data": {...}}` 格式"
- "数据库字段命名用 snake_case，API 字段用 camelCase"

**类型 D: 有用的搜索模式**
- "找所有发送通知的代码: `grep -r 'send_notification\|notify' src/`"

**类型 E: 新增/修改的功能**
- "新增了 X 功能，入口在 Y 文件"
- "重构了 Z 模块，原来的 A 方法改名为 B"

### 2.2 更新规则

1. 读取现有 CLAUDE.md
2. 对每条新知识:
   - 如果已有类似内容 → 更新/补充
   - 如果是新内容 → 添加到对应 section
   - 如果与已有内容矛盾 → 替换旧内容（项目在演进，旧信息可能过时）
3. 写入更新后的 CLAUDE.md
4. 向用户报告更新了什么

### 2.3 不要更新的内容

- 本次任务的具体细节（"我修了一个 bug"——这是 git log 的事）
- 临时性信息（"当前正在做 X 功能"）
- 推测性结论（"这个模块可能有性能问题"——除非你验证了）

---

## 3. Learn（深入调研）

当用户执行 `/project-memory learn <topic>` 时执行。

**目标**: 深入调研项目的某个特定方面，并将结论记录到 CLAUDE.md。

**示例**:
- `/project-memory learn auth` — 调研认证/授权机制
- `/project-memory learn database` — 调研数据库访问模式、迁移方式
- `/project-memory learn deployment` — 调研部署流程
- `/project-memory learn testing` — 调研测试策略和基础设施

### 3.1 调研步骤

1. 围绕 topic 搜索相关文件
2. 读取核心文件，理解实现方式
3. 追踪关键流程（如 auth: 从请求进入 → token 验证 → 权限检查 → 通过/拒绝）
4. 记录发现的模式、约定、注意事项
5. 更新 CLAUDE.md 的对应 section

### 3.2 输出格式

调研结果直接更新到 CLAUDE.md 中，以新的 subsection 形式添加。如果内容较多，可以创建单独的 `.claude/docs/<topic>.md` 文件，在 CLAUDE.md 中用 `@` 引用。

---

## 触发建议

以下场景建议 agent 主动提议执行 update：

1. **完成了涉及 3+ 文件的修改后** — "我在这次修改中发现了一些项目模式，要更新 CLAUDE.md 吗？"
2. **踩了一个花了 10+ 分钟才搞清楚的坑** — 这类知识最值得记录
3. **发现 CLAUDE.md 中的信息已过时** — 直接修正
4. **首次进入一个没有 CLAUDE.md 的项目** — 提议执行 init
