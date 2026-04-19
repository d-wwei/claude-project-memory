# Init Workflow

首次进入项目时执行 `/better-code init`。目标：用最少时间建立可操作的项目知识 + 研发认知约束。

## Step 0: 检测已有状态

**在探索之前必做**：检测 `~/.better-work/<project>/` 是否已存在（可能由 better-work 或 better-test 先创建了 `shared/`）。

```
若 ~/.better-work/<project>/ 不存在：
    完整初始化（执行 Step 1–6）
若存在但只有 shared/（例如 better-test 已先占）：
    跳过 shared/ 生成，只生成 code/ 下的文件（Step 3 中过滤）
若 shared/ 和 code/ 都存在：
    退出，提示用户改用 /better-code update
若存在 code/ 但没有 shared/（异常半完成状态）：
    报告异常并询问用户：a) 仅补生成 shared/ 后注入；b) 人工清理后重跑 init
    不要静默继续
```

项目名推导：取项目根目录名（去掉特殊字符），与 `~/.better-work/<project>/` 对应。

## Step 1: 项目分类

探索代码之前先判断项目类型。读取根目录文件列表 + README 前 50 行 + 包管理器配置即可判断。

| 项目类型 | 识别信号 | 探索侧重 |
|----------|---------|----------|
| Backend API / Web service | FastAPI/Express/Gin + routes 目录 + DB 配置 | 路由结构、中间件链、DB 访问模式、认证流程 |
| CLI tool / Library | setup.py/Cargo.toml + bin/ 或 cmd/ + 无 HTTP | 公开 API 面、参数解析、版本策略 |
| Frontend SPA | React/Vue/Svelte + components/ + 状态管理 | 组件树、状态流、API 调用层 |
| Data pipeline / ML | DAG 定义 + transformers + data schemas | 数据流向、schema 演进、任务编排 |
| Monorepo / Platform | packages/ 或 apps/ + workspace 配置 | 包边界、共享依赖、CI 矩阵 |
| 混合 / 不确定 | 以上都不明确 | 先找入口点，追踪一条主流程 |

## Step 2: 信号驱动探索

不做"读完整个项目"。按以下信号源高效采集：

### 信号源 A：Git 历史（最高价值）

```
# 最近 3 个月高频修改文件（= 活跃区域）
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

- 测试目录结构 → 揭示模块划分
- 测试命令 → 从 Makefile / package.json scripts / CI 配置提取
- 测试覆盖 → 哪些模块测试厚、哪些薄（薄的 = 改动风险高）

### 信号源 D：采样阅读（约定提取）

从不同模块各读 1 个典型文件（选 git 历史中修改频率中等的）。提取：错误处理模式、日志方式、数据库访问方式、命名约定、注释风格。

## Step 3: 生成知识文件

按以下顺序生成（参考 `references/templates.md` 的质量标准）：

### 3a. shared/（若已存在则跳过对应文件）

1. `shared/index.md` — 从 Step 1 分类结果 + Step 2 关键发现中提炼。纯项目知识
2. `shared/map.md` — 从信号源 B 的依赖图 + 信号源 A 的改动集构建
3. `shared/progress.md` — 初始化为空模板（仅当文件不存在时）

### 3b. code/（始终生成）

4. `code/protocol.md` — 研发认知约束（≤15 行）。询问用户选择风险等级：
   - **严格版**：涉及资金、用户数据、核心基础设施的项目
   - **标准版**：日常业务项目
   - **宽松版**：内部工具、实验项目

   **无人值守 fallback**（fork 会话 / CI / 无交互通道）：默认选 **标准版**，并在 Step 6 报告中明示选择原因（"无交互通道，按默认"），让用户事后可自行切换。
5. `code/conventions.md` — 从信号源 D 采样阅读提取
6. `code/danger-zones.md` — 从信号源 B 高入度模块 + 信号源 A bug 热点构建

## Step 4: 创建符号链接 + 注册

符号链接创建必须幂等，不能覆盖意外存在的路径：

```bash
TARGET="$HOME/.better-work/<project>"
LINK="<project-root>/.better-work"

if [ ! -e "$LINK" ]; then
    ln -s "$TARGET" "$LINK"
elif [ -L "$LINK" ] && [ "$(readlink "$LINK")" = "$TARGET" ]; then
    : # 链接已正确，无操作
else
    echo "冲突：$LINK 已存在但不是指向 $TARGET 的符号链接"
    echo "请人工检查并决定：保留现状 / 备份后重建 / 修正目标"
    exit 1
fi
```

### .gitignore 追加

```
# 追加到 <project-root>/.gitignore（若未包含）
grep -q "^\.better-work/$" .gitignore || echo ".better-work/" >> .gitignore
```

### 注册到全局

```
在 ~/.better-work/registry.yml 中加一项：
  <project>:
    path: <project-root 绝对路径>
    initialized_by: better-code
    created_at: <ISO 时间戳>
```

若 `registry.yml` 不存在，创建之（空 yaml 根对象）。

## Step 5: 注入到当前平台

检测当前 agent 平台，注入**两个引用**：

- Claude Code → 项目 `CLAUDE.md`（若不存在则创建新文件）追加：
  ```
  @.better-work/shared/index.md
  @.better-work/code/protocol.md
  ```
- 其他平台 → 参考 `references/adapters.md`

若检测到已装 better-work，不再注入 `@.better-work/protocol.md`（通用执行标准由 better-work 负责）。

## Step 6: 报告

向用户展示：
- 项目类型判断结果
- 生成了哪些文件（按 shared/ 和 code/ 分组）
- 关键信息摘要（top 5 危险区域、核心约定）
- 建议后续 `/better-code learn <topic>` 的主题（基于探索中发现但未深入的复杂模块）

## 深度控制

| 项目规模 | 预计耗时 | 探索深度 |
|----------|---------|----------|
| < 10k 行 | 1-2 分钟 | 可以全面读，直接读核心模块全文 |
| 10k-50k 行 | 3-5 分钟 | 只读签名和结构，采样 5 个文件全文 |
| 50k-200k 行 | 5-10 分钟 | 只依赖 git 历史和依赖图，采样 3 个文件 |
| > 200k 行 | 10-15 分钟 | 分模块逐步探索，init 只覆盖最活跃的 top 5 模块 |
