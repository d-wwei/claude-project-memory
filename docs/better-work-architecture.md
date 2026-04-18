# Better-Work 系列架构设计文档

> 版本: v1.0 | 日期: 2026-04-18 | 状态: 设计完成，待实现

## 1. 愿景

Better-Work 是一套 AI agent 技能系列，核心理念是 **Context, Not Control**：通过给 agent 提供持久化的项目知识和学科约束，从根本上减少错误，而不是只在出错后约束行为。

系列覆盖多个学科，共享统一的知识基础设施，各学科独立演进。

## 2. 设计原则

1. **Context > Control** — 给 agent 正确的知识比约束它的行为更有效
2. **预防 > 治疗** — 执行标准始终生效，不是等出错才激活
3. **独立可用，组合更强** — 每个子技能可单独安装，组合后共享知识层
4. **学科与方法论正交** — code/test/plan 是学科维度，rounds/waves/verify 是执行模式维度，可自由组合
5. **知识随项目演进** — 信号驱动的增量更新，不是一次性生成的静态文档

## 3. 系列组成

### 3.1 技能清单

| 技能 | 职责 | 独立命令 | 作为子技能 |
|------|------|---------|-----------|
| **better-work** | 通用执行协议 + 统一入口 + 初始化 | `/better-work` | — |
| **better-code** | 研发知识管理 + 研发认知约束 | `/better-code` | `/better-work code` |
| **better-test** | 测试知识管理 + 测试认知约束 | `/better-test` | `/better-work test` |
| **better-plan** | 策划知识管理 + 策划认知约束 | `/better-plan` | `/better-work plan` |
| **better-design** | 设计知识管理 + 设计认知约束 | `/better-design` | `/better-work design` |
| **better-write** | 写作知识管理 + 写作认知约束 | `/better-write` | `/better-work write` |

### 3.2 两个正交维度

```
维度 1: 学科（做什么）              维度 2: 执行模式（怎么做）
├── code   研发                    ├── 默认     轻量线性执行
├── test   测试                    ├── rounds   PDCA 回合（复杂任务）
├── plan   策划                    ├── waves    波次交付（项目级）
├── design 设计                    ├── verify   验证偏重
└── write  写作                    ├── unstick  恢复偏重
                                   └── handoff  结构化交接
```

执行模式可叠加在任何学科上：做研发时可以进入 rounds 模式，做测试时也可以。

## 4. 文件系统架构

### 4.1 全局目录

```
~/.better-work/                              ← 个人 git repo（所有项目的注册中心）
├── .gitmodules                              ← 子模块注册（需要团队共享的项目）
├── registry.yml                             ← 项目注册表
├── global-config.yml                        ← 全局偏好（如默认协议严格程度）
├── futu-opend-rs/                           ← 项目 A 的知识（独立 git repo 或 submodule）
├── trading-platform/                        ← 项目 B 的知识（独立 git repo 或 submodule）
└── internal-tools/                          ← 项目 C 的知识（直接追踪，不共享）
```

换电脑恢复：`git clone --recursive <your-remote> ~/.better-work`

### 4.2 每个项目的知识目录

```
~/.better-work/<project-name>/               ← 独立 git repo，可团队共享
├── protocol.md                              ← 通用执行标准（≤30 行）
├── shared/                                  ← 所有学科共用
│   ├── index.md                             ← 项目知识入口（≤150 行）
│   ├── map.md                               ← 模块地图 + 任务→文件映射
│   └── progress.md                          ← 当前任务进度（.gitignore）
├── code/                                    ← 研发专用
│   ├── protocol.md                          ← 研发认知约束（≤15 行）
│   ├── conventions.md                       ← 编码约定（知识）
│   └── danger-zones.md                      ← 高风险区域（知识）
├── test/                                    ← 测试专用
│   ├── protocol.md                          ← 测试认知约束（≤15 行）
│   ├── test-groups.md                       ← 测试组定义（知识）
│   ├── impact-map.md                        ← 变更→测试映射（知识）
│   └── known-issues.md                      ← 已知问题（知识）
├── plan/                                    ← 策划专用（结构类似）
├── design/                                  ← 设计专用（结构类似）
├── write/                                   ← 写作专用（结构类似）
└── .gitignore                               ← 排除 progress.md、adapters/ 等
```

### 4.3 项目代码仓库中

```
~/repos/futu-opend-rs/                       ← 项目代码（产品仓库）
├── .better-work/                            ← 符号链接 → ~/.better-work/futu-opend-rs/
├── .gitignore                               ← 包含 .better-work/
├── CLAUDE.md                                ← @ 引用（见下方注入机制）
└── src/...                                  ← 产品代码
```

## 5. 注入机制

### 5.1 CLAUDE.md 中的引用

```markdown
# 项目 CLAUDE.md（每次对话自动加载）

@.better-work/shared/index.md               # 项目知识（共享）
@.better-work/code/protocol.md              # 研发认知约束
@.better-work/test/protocol.md              # 测试认知约束
```

通用执行标准 `protocol.md` 默认注入**全局** `~/.claude/CLAUDE.md`（所有项目生效），用户可选择注入到项目级。

### 5.2 什么时候加载什么

| 文件 | 加载时机 | 内容类型 |
|------|---------|---------|
| `protocol.md` | 每次对话（全局或项目 CLAUDE.md） | 通用执行标准 |
| `shared/index.md` | 每次对话（项目 CLAUDE.md @引用） | 项目知识 |
| `code/protocol.md` | 每次对话（项目 CLAUDE.md @引用） | 研发认知约束 |
| `test/protocol.md` | 每次对话（项目 CLAUDE.md @引用） | 测试认知约束 |
| `code/conventions.md` | agent 做研发时主动 Read | 研发知识 |
| `code/danger-zones.md` | agent 改高风险文件前主动 Read | 研发知识 |
| `test/test-groups.md` | agent 做测试时主动 Read | 测试知识 |
| `test/impact-map.md` | 代码变更后推荐测试时 Read | 测试知识 |
| better-work 完整 SKILL.md | 复杂任务/卡住时按需加载 | 详细执行方法论 |

### 5.3 多平台适配

| 平台 | 注入方式 |
|------|---------|
| Claude Code | CLAUDE.md 中 `@` 引用 |
| Cursor | `.cursor/rules/better-work.mdc` 嵌入内容 |
| Gemini CLI | GEMINI.md 中 `@` 引用 |
| Codex / OpenCode / OpenClaw | AGENTS.md 中嵌入内容 |

## 6. 命令设计

### 6.1 better-work（主技能）

```
/better-work init                  首次初始化项目 .better-work/ 目录
/better-work status                查看当前项目的知识状态
/better-work rounds                进入 PDCA 回合执行模式
/better-work waves                 进入波次交付模式
/better-work verify                进入验证模式
/better-work unstick               进入恢复模式
/better-work handoff               生成结构化交接文档
/better-work code <cmd>            路由到 better-code 子技能
/better-work test <cmd>            路由到 better-test 子技能
```

### 6.2 better-code（研发子技能，可独立安装）

```
/better-code init                  探索项目 → 生成 code/ 知识文件
/better-code update                信号驱动的增量更新
/better-code checkpoint            保存当前任务进度
/better-code resume                从断点恢复
/better-code learn <topic>         深入调研某子系统
```

### 6.3 better-test（测试子技能，可独立安装）

```
/better-test init                  探索测试结构 → 生成 test/ 知识文件
/better-test update                更新测试知识
/better-test strategy              分析变更 → 推荐测试策略
/better-test feedback <id> <verdict>  记录测试反馈
```

## 7. 独立安装 vs 完整安装

### 7.1 独立安装（只装 better-code）

```
用户执行 /better-code init:
  1. 创建 ~/.better-work/<project>/（如不存在）
  2. 生成 shared/index.md（轻量项目探索）
  3. 生成 code/ 下的所有文件
  4. 创建符号链接 <project>/.better-work/ → ~/.better-work/<project>/
  5. 在项目 CLAUDE.md 中注入:
     @.better-work/shared/index.md
     @.better-work/code/protocol.md
  6. 注册到 registry.yml

  注意：不注入通用执行协议（没装 better-work）
```

### 7.2 完整安装（先 better-work 再 better-code）

```
用户执行 /better-work init:
  1. 创建 ~/.better-work/<project>/
  2. 生成 protocol.md（通用执行标准）
  3. 生成 shared/index.md（轻量项目探索）
  4. 创建符号链接
  5. 注入 protocol.md 到全局或项目 CLAUDE.md（用户选择）
  6. 注入 @.better-work/shared/index.md 到项目 CLAUDE.md
  7. 注册到 registry.yml
  8. 询问：要初始化哪些子技能？

用户执行 /better-work code init（或 /better-code init）:
  1. 检测 .better-work/ 已存在 ✓
  2. 只生成 code/ 下的文件（不重复创建 shared/）
  3. 在项目 CLAUDE.md 追加 @.better-work/code/protocol.md
```

### 7.3 后补安装 better-work

```
用户已有 better-code，后来装了 better-work:
  /better-work init:
    1. 检测 .better-work/ 已存在
    2. 检测 shared/index.md 已存在（better-code 创建的）
    3. 只补充缺失的：protocol.md
    4. 注入 protocol.md 到全局/项目 CLAUDE.md
    5. 更新 registry.yml
```

## 8. 知识维护机制

### 8.1 shared/ 目录的多 skill 协作

所有 skill 都可以读写 `shared/` 目录下的文件。协作规则：

- 每次修改是一个 git commit（自动）
- 多个 skill 需要改同一文件 → 各自在 worktree 中修改 → merge 回 main
- 冲突时标记为 blocker，等人类决策
- 每条修改必须附带来源标注（哪个 skill、因为什么信号）

### 8.2 版本管理

```
每次知识更新 = 一个 git commit
  commit message 格式：[skill-name] <action>: <summary>
  例：[better-code] update: 发现 payment_service.py 入度为 23，加入 danger-zones
  例：[better-test] update: 修复了 WebSocket 测试后更新 known-issues
```

### 8.3 protocol.md 的严格程度选择

| 等级 | 适用场景 | 行数 | 内容 |
|------|---------|------|------|
| 严格 | 支付系统、核心基础设施 | ~25 行 | 完整原则 + 完成标准 + 5 条触发器 + 失败归因 |
| 标准 | 日常业务项目 | ~15 行 | 完成标准 + 4 条触发器 |
| 宽松 | 内部工具、实验项目 | ~5 行 | 2 条最基本约束 |

init 时询问用户选择。可以后续随时切换。

## 9. 跨 skill 信息流

```
better-plan 输出需求 → shared/index.md 或 plan/requirements.md
    ↓
better-code 读取需求 → 知道要做什么 → 开发完更新 code/danger-zones.md
    ↓
better-test 读取 code/danger-zones.md → 知道哪些改动高风险 → 调整测试策略
    ↓
better-test 发现 bug → 更新 test/known-issues.md
    ↓
better-code 读取 known-issues.md → 知道什么在挂 → 修复
```

每个 skill **只写自己目录下的文件 + shared/**，可以**读任何 skill 的文件**。

## 10. 实现路线

### Phase 1（当前）
- 重构 better-code（从 project-memory 改名，适配新架构）
- 从 futu-opend-tester 抽象出 better-test
- 验证两个 skill 的共享知识层协作

### Phase 2
- 将 better-work 的核心协议拆分为 protocol.md（始终注入）+ 完整 SKILL.md（按需加载）
- 实现 `/better-work init` 统一入口
- 实现子技能别名路由

### Phase 3
- 扩展其他学科 skill（plan / design / write）
- 跨 skill 信息流的自动化
- 多平台 adapter 完善

## 11. 设计决策记录

| 决策 | 选择 | 理由 |
|------|------|------|
| 知识文件存放位置 | `~/.better-work/` + 符号链接到项目 | 统一管理 + 换电脑一键恢复 + 项目仓库不污染 |
| 每个项目独立 git repo | 是 | 不同项目共享给不同团队 |
| 通用协议默认注入位置 | 全局 CLAUDE.md | 执行标准是通用的，所有项目都应生效 |
| 子技能独立安装 | 支持 | 降低入门门槛，不强制全套 |
| rounds/waves 等执行模式 | 正交于学科，任何学科都可激活 | 两个维度不应耦合 |
| 学科目录结构 | protocol.md（约束）+ 知识文件 | 约束始终注入，知识按需读取 |
| shared/ 维护权限 | 所有 skill 可读写 | 知识是共同积累的，不应有单一 owner |
| shared/ 冲突处理 | git worktree + merge | 版本管理是解决多方写入的成熟方案 |
