<!-- Synced with README.md as of 2026-04-18 -->

**[🇺🇸 English](README.md)** | **🇨🇳 中文**

# better-code

### 你的 AI agent 不是偷懒——是盲飞。这个 repo 给它装上车灯。

AI coding agent 在 10 万行代码里翻车的五种方式：

1. **搜索不充分** — 改了接口没找到所有调用方
2. **不了解架构** — 在错误的层做了修改
3. **上下文窗口耗尽** — 读了太多代码，早期信息被遗忘
4. **没跑对测试** — 跑了单元测试，没跑集成测试
5. **不知道项目约定** — 功能正确但违反规范

这五个都是**信息问题**，不是态度问题。再多"请认真点"也解决不了。解决它们的方式是：把 agent 需要的信息一次性给到它，放在每次会话都会加载的文件里。

这就是 `better-code` 做的事——[Full Context + Lite Control](https://github.com/d-wwei/better-work) 框架里研发那一半的落地。一份 150 行的项目记忆，告诉 agent 那些它从代码本身看不到的东西。

## 为什么 CLAUDE.md 单打独斗不行

agent 会忘事，最朴素的反应是往 `CLAUDE.md` 里塞更多内容。我试过。它会慢慢长成 400 行的无人维护的大作文，没人记得里面写了什么，一个月后有一半已经过期。

项目知识其实分两种，老化速度完全不同：

- **架构、约定、危险区域** 变化慢——以月为尺度。适合放进持久化的知识文件，每次会话加载一次。
- **任务状态、悬而未决的问题、进行中的决定** 按小时变化。适合放进会话级的进度文件，随时覆盖重写。

`better-code` 把这两类分开。持久的放进 `shared/index.md` 和 `code/`——通过 `@` 引用每次会话加载。进度放进 `shared/progress.md`——`checkpoint` / `resume` 随时覆盖。

## 150 行里到底有什么

每个文件对应一个失败模式：

| 文件 | 内容 | 对应失败模式 |
|------|------|-------------|
| `shared/index.md`（≤150 行） | 架构概览、模块边界、数据流 | 不了解架构 |
| `code/danger-zones.md` | 高入度文件 + 改前必跑的搜索命令 | 搜索不充分 |
| `code/conventions.md` | 编码规则 + 正反例 + 理由 | 不知道项目约定 |
| `shared/map.md`（≤400 行） | 任务→文件定位表；"X 在哪里"一跳答完 | 上下文窗口耗尽 |
| `code/protocol.md`（≤15 行） | 研发专用的认知底线 | 上述的兜底 |

不是手写文档。**信号驱动生成**：

- `git log` 分析高频修改文件 → 放进活跃区域
- `import` 图算入度 → 高入度进危险区域
- Agent 搜了 3 次以上才找到文件 → 自动补定位表
- 修改引发意外测试失败 → 自动补危险区域
- 违反了某条约定 → 用正/错对比记录

不是一次性生成文档，是日常使用中持续学习。支持 `checkpoint` / `resume`，跨 session 接力。

## init 之后 agent 的行为变化

没跑 `/better-code init` 之前，新会话的首轮长这样：

```
你：   修一下 funds 接口的超时
Agent：让我搜 funds... grep... grep...
       HTTP 路由挂在哪个文件？
       超时在哪里定义？
       [5 次以上工具调用才到有用动作]
```

init 之后，同样一个问题：

```
你：   修一下 funds 接口的超时
Agent：index.md 说 REST handler 在 src/rest/mod.rs，funds 相关逻辑在
       src/rest/funds.rs。danger-zones.md 标了这条路径为高风险（3 个
       服务依赖它）。我先读 funds.rs。
       [1 次工具调用直接进入动作]
```

不是魔法——就是 agent 在 grep 之前先读一遍知识文件。CLAUDE.md 的 `@` 引用让这件事每次会话自动发生。

## 安装

### Claude Code（原生）

```bash
git clone https://github.com/d-wwei/better-code.git ~/repos/better-code
ln -s ~/repos/better-code ~/.claude/skills/better-code
```

### 其他平台

Cursor / Gemini CLI / Codex / OpenCode / OpenClaw 的 adapter 在 `references/adapters.md`。`/better-code init` 产出的知识文件跨平台通用，各 adapter 只是决定注入语法。

## 快速开始

进到项目目录：

```
/better-code init
```

skill 先判断项目类型，用「信号驱动采样」方式探索（git 历史 + 依赖图 + 文件名模式——不是全文读），然后生成：

- `.better-work/shared/index.md` — 项目知识入口
- `.better-work/code/protocol.md` — 研发认知约束
- `.better-work/code/conventions.md` — 带示例的代码约定
- `.better-work/code/danger-zones.md` — 高风险区域 + 检查命令

然后往项目 `CLAUDE.md` 里追加 `@.better-work/shared/index.md` + `@.better-work/code/protocol.md`，以后每次会话都自动带上项目上下文。

做了几个多文件任务之后：

```
/better-code update
```

把新发现的约定和危险区增量折叠进知识库。信号驱动——不是全量重探。

## 命令参考

| 命令 | 做什么 |
|------|--------|
| `/better-code init` | 首次探索 + 生成知识文件 + 注入 CLAUDE.md 引用 |
| `/better-code update` | 信号驱动的增量更新（新文件 / 新约定 / 新踩坑） |
| `/better-code checkpoint` | 把当前任务状态写到 `progress.md`，下次会话可恢复 |
| `/better-code resume` | 读 `progress.md`，按文件/函数级别汇报进度并继续 |
| `/better-code learn <topic>` | 深入调研某个子系统，把发现折回已有的知识文件 |

五个命令直接调用或通过 `/better-work code <cmd>`（装了 better-work 时）行为完全一样。

## 输出结构

知识存储在 `~/.better-work/<project-name>/`——独立的 git repo，和产品代码分开。项目目录通过符号链接访问：

```
<project>/.better-work/                      → ~/.better-work/<project-name>/
├── protocol.md                              （better-work 管；better-code 不写）
├── shared/
│   ├── index.md                             ≤150 行——项目知识入口
│   ├── map.md                               ≤400 行——模块图 + 定位表
│   └── progress.md                          已 gitignore——当前任务状态
└── code/
    ├── protocol.md                          ≤15 行——研发认知约束
    ├── conventions.md                       代码约定 + 正反例
    └── danger-zones.md                      高风险文件 + 检查命令
```

`shared/index.md` 纯项目知识，零认知规则。`code/protocol.md` 纯认知约束，零项目事实。一起通过 CLAUDE.md `@` 引用加载，职责各自单一。

**为什么放在产品仓库外？** 项目知识是团队的持久资产。放在 `~/.better-work/<project-name>/` 作为独立 git repo，团队可以独立共享它，不必和产品代码共用权限体系。产品仓库保持干净（符号链接已 gitignore）。

## Better-Work 系列

- **[better-work](https://github.com/d-wwei/better-work)** — Lite Control 层 + 系列入口。完整的设计故事看那里。
- **better-code**（本 repo） — 研发 Full Context
- **[better-test](https://github.com/d-wwei/better-test)** — 测试 Full Context

`better-code` 独立可用。装了 `better-work` 之后，`/better-work code <cmd>` 成为别名，行为一致。

## 已知限制

- **`update` 是手动的。** 没有文件监视器。任务跑完不跑 `/better-code update` 知识就会过期，直到你下次想起来。
- **信号驱动生成不是无所不知。** 首次 `index.md` 可能漏掉需要用一周才会暴露的细节。这就是 `update` 的用处。
- **产品仓库和 `~/.better-work/<project>/` 是两个 git 面。** 这是故意的——让团队独立共享项目知识，代价是你要看两次 `git status`。
- **大规模重构后 `shared/map.md` 会漂移。** 不一致的条目会加 `[未验证]` 标记，但不会阻止你继续用。
- **没有包管理器集成。** 每个平台都是 symlink 或 curl 安装。

## License

MIT。

---

完整故事：系列入口 README 里的 [Full Context, Lite Control](https://github.com/d-wwei/better-work) 长文。

问题、反馈或讨论：[GitHub issues](https://github.com/d-wwei/better-code/issues)。
