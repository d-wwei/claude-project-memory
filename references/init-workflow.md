# Init Workflow

首次进入项目时执行。目标：用最少的时间建立可操作的项目记忆。

## Step 1: 项目分类

在探索代码之前，先判断项目类型。读取根目录文件列表 + README 前 50 行 + 包管理器配置即可判断。

| 项目类型 | 识别信号 | 探索侧重 |
|----------|---------|----------|
| Backend API / Web service | FastAPI/Express/Gin + routes 目录 + DB 配置 | 路由结构、中间件链、DB 访问模式、认证流程 |
| CLI tool / Library | setup.py/Cargo.toml + bin/ 或 cmd/ + 无 HTTP | 公开 API 面、参数解析、版本策略 |
| Frontend SPA | React/Vue/Svelte + components/ + 状态管理 | 组件树、状态流、API 调用层 |
| Data pipeline / ML | DAG 定义 + transformers + data schemas | 数据流向、schema 演进、任务编排 |
| Monorepo / Platform | packages/ 或 apps/ + workspace 配置 | 包边界、共享依赖、CI 矩阵 |
| 混合 / 不确定 | 以上都不明确 | 先找入口点，追踪一条主流程 |

## Step 2: 信号驱动探索

不做"读完整个项目"。按以下信号源高效采集信息：

### 信号源 A：Git 历史（最高价值）

```
# 最近 3 个月高频修改文件（= 活跃区域，最需要了解）
git log --since="3 months ago" --name-only --pretty=format: | sort | uniq -c | sort -rn | head -30

# 常见改动集（经常一起被修改的文件 = 一个功能单元）
git log --since="3 months ago" --name-only --pretty=format:"---" | 分析共现关系

# 最近 bug 修复集中在哪些文件
git log --since="3 months ago" --grep="fix\|bug\|hotfix" --name-only --pretty=format: | sort | uniq -c | sort -rn | head -15
```

### 信号源 B：依赖图（结构信息）

```
# 被引用最多的模块（= 改动影响面大 = 危险区域）
搜索所有 import/require/use 语句，按目标模块频率排序，取 top 20

# 模块间调用方向
从入口文件追踪 3 层深度的 import 链，画出 A→B→C 关系
```

### 信号源 C：测试结构（行为规格）

```
# 测试目录结构 → 揭示模块划分和测试策略
# 测试命令 → 从 Makefile / package.json scripts / CI 配置中提取
# 测试覆盖 → 哪些模块测试厚、哪些薄（薄的 = 改动风险高）
```

### 信号源 D：采样阅读（约定提取）

从不同模块各读 1 个典型文件（选 git 历史中修改频率中等的——太活跃的可能正在重构，太冷门的可能是遗留代码）。提取：
- 错误处理模式
- 日志方式
- 数据库访问方式
- 命名约定
- 注释风格

## Step 3: 生成 .project-memory/

按以下顺序生成各文件（参考 `references/templates.md` 的质量标准）：

1. `index.md` — 纯项目知识，从 Step 1 分类结果 + Step 2 关键发现中提炼
2. `cognitive-protocol.md` — 认知约束，根据项目风险等级选择模板（严格/标准/宽松），询问用户选择
3. `map.md` — 从信号源 B 的依赖图 + 信号源 A 的改动集构建
4. `conventions.md` — 从信号源 D 的采样阅读中提取
5. `danger-zones.md` — 从信号源 B 的高入度模块 + 信号源 A 的 bug 热点构建
6. `progress.md` — 初始化为空模板

**风险等级选择指引**：
- **严格版**：涉及资金、用户数据、核心基础设施的项目
- **标准版**：日常业务项目、中等规模的功能开发
- **宽松版**：内部工具、实验项目、原型开发

## Step 4: 注入到当前平台

检测当前 agent 平台，生成对应 adapter 并注入**两个文件**：
- Claude Code → 在项目 CLAUDE.md 中追加：
  ```
  @.project-memory/index.md
  @.project-memory/cognitive-protocol.md
  ```
- 其他平台 → 参考 `references/adapters.md`

## Step 5: 报告

向用户展示：
- 项目类型判断结果
- 生成了哪些文件
- 发现的关键信息摘要（top 5 危险区域、核心约定）
- 建议后续 `learn` 的主题（基于探索中发现的复杂模块）

## 深度控制

| 项目规模 | 预计耗时 | 探索深度 |
|----------|---------|----------|
| < 10k 行 | 1-2 分钟 | 可以全面读，直接读核心模块全文 |
| 10k-50k 行 | 3-5 分钟 | 只读签名和结构，采样 5 个文件全文 |
| 50k-200k 行 | 5-10 分钟 | 只依赖 git 历史和依赖图，采样 3 个文件 |
| > 200k 行 | 10-15 分钟 | 分模块逐步探索，init 只覆盖最活跃的 top 5 模块 |
