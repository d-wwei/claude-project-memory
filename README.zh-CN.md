<!-- Synced with README.md as of 2026-04-18 -->

**[🇺🇸 English](README.md)** | **🇨🇳 中文**

# better-code

### 让 AI agent 别在每次新会话都重新摸一遍你的代码库

`better-code` 是一个 Claude Code skill，给 AI agent 一份持续更新的项目记忆。每次新会话开启，agent 不再盲目 grep——它带着一份紧凑的索引开跑：架构是什么样、代码约定是什么、哪些文件是"改了就炸"的高风险区。

原生支持 Claude Code，同时自带 Cursor / Gemini CLI / Codex / OpenCode 适配器。知识本体是纯 markdown——可跨平台、可团队共享、作为独立 git repo 受版本管理。

这是 [Better-Work 系列](https://github.com/d-wwei/better-work-skill) 的一部分，但可以独立安装独立使用，不依赖系列里的其他 skill。

## 为什么需要它

每次新会话进大项目，流程都是这样：agent 在 grep 昨天刚见过的类名。它在问 HTTP 路由挂在哪个文件——上周二你告诉过它。它在 `payment_service.py` 里提议重构——就是你标注过"别动，三个团队在用"的那个文件。

常规做法是往 `CLAUDE.md` 里塞更多上下文。但 `CLAUDE.md` 会慢慢长成 400 行的无人维护的大作文，没人记得里面写了什么，一个月后有一半已经过期。

项目知识其实分两种，老化速度完全不同：

- **架构、约定、高风险区** 变化慢——以月为尺度。适合放进持久化的知识文件，每次会话加载一次。
- **任务状态、悬而未决的问题、进行中的决定** 按小时变化。适合放进会话级的进度文件，随时覆盖重写。

`better-code` 把这两类分开。持久的放进 `shared/index.md` 和 `code/`——通过 `@` 引用每次会话加载。进度放进 `shared/progress.md`——`checkpoint` / `resume` 随时覆盖，不碰持久文件。

## 安装

### Claude Code（原生）

```bash
git clone https://github.com/d-wwei/better-code.git ~/repos/better-code
ln -s ~/repos/better-code ~/.claude/skills/better-code
```

下次开 Claude Code 会话，skill 列表里就能看到 `better-code`。

### 其他平台

`references/adapters.md` 里有可直接粘贴的安装命令，覆盖：

- Cursor — `.cursor/rules/better-code.mdc`
- Gemini CLI — `GEMINI.md @reference`
- Codex — 在 `AGENTS.md` 里嵌入
- OpenCode / OpenClaw

`/better-code init` 产出的知识文件本身跨平台通用，各 adapter 只是决定项目 CLAUDE.md / rules 文件 / AGENTS.md 怎么引用它们。

## 快速开始

进到项目目录：

```
/better-code init
```

skill 会先判断项目类型，用「信号驱动采样」的方式探索（git 历史 + 依赖图 + 文件名模式——不是全文读），然后生成：

- `.better-work/shared/index.md` — 项目知识入口，≤150 行
- `.better-work/code/protocol.md` — 研发认知约束，≤15 行
- `.better-work/code/conventions.md` — 具体的代码约定，带正反例
- `.better-work/code/danger-zones.md` — 高风险区域清单 + 检查命令

然后往项目 `CLAUDE.md` 里追加 `@.better-work/shared/index.md` + `@.better-work/code/protocol.md`，以后每次会话都自动带上项目上下文。

下次新会话，agent 通过 `@` 引用加载这两个文件，第一跳定位目标降到 1–2 次，而不是之前的 5+。

做了几个多文件任务之后：

```
/better-code update
```

这会把新发现的约定和危险区增量折叠进知识库。信号驱动——不是全量重探。

### init 实际会产出什么

在一个典型的 Rust daemon 项目上首次跑完，大概会看到：

```
<project>/
├── .better-work/              → ~/.better-work/<project>/
├── .gitignore                 （.better-work/ 已被排除在产品仓库之外）
└── CLAUDE.md                  （末尾新增两行 @ 引用）
```

知识仓库这边：

```
~/.better-work/<project>/
├── .git/                      （它本身就是独立的 git repo）
├── shared/
│   ├── index.md               119 行——模块、入口、关键路径
│   └── progress.md            空，已 gitignore
└── code/
    ├── protocol.md            14 行
    ├── conventions.md         86 行——12 条规则带示例
    └── danger-zones.md        53 行——7 个文件 + 检查命令
```

## 命令参考

| 命令 | 做什么 |
|------|--------|
| `/better-code init` | 首次探索 + 生成知识文件 + 注入 CLAUDE.md 引用 |
| `/better-code update` | 信号驱动的增量更新（新文件 / 新约定 / 新踩坑） |
| `/better-code checkpoint` | 把当前任务状态写到 `shared/progress.md`，下次会话可恢复 |
| `/better-code resume` | 读 `progress.md`，按文件/函数级别汇报进度并继续 |
| `/better-code learn <topic>` | 深入调研某个子系统，把发现折回已有的知识文件 |

五个命令直接调用或通过 `/better-work code <cmd>`（装了 better-work 时）行为完全一样。

### init 之后 agent 的行为变化

没跑 `/better-code init` 之前，新会话的首轮大概长这样：

```
你：   修一下 funds 接口的超时
Agent：让我搜 funds... grep... grep...
       HTTP 路由挂在哪个文件？
       超时在哪里定义？
       [5 次以上工具调用才到有用动作]
```

init 之后，同样的问题一开口 agent 就有上下文：

```
你：   修一下 funds 接口的超时
Agent：根据 index.md，REST handler 在 src/rest/mod.rs，funds 相关逻辑在
       src/rest/funds.rs。danger-zones.md 标了这条路径为高风险（3 个服务依赖它）。
       我先读 funds.rs。
       [1 次工具调用直接进入动作]
```

不是魔法——就是 agent 在 grep 之前先读一遍知识文件。CLAUDE.md 的 `@` 引用让这件事每次会话自动发生。

## 输出结构

知识存储在 `~/.better-work/<project-name>/`——独立的 git repo，和产品代码分开。项目目录通过符号链接访问：

```
<project>/.better-work/                      → ~/.better-work/<project-name>/
├── protocol.md                              （better-work 管；better-code 不写）
├── shared/                                  （所有 Better-Work 子技能可读）
│   ├── index.md                             ≤150 行——项目知识入口
│   ├── map.md                               ≤400 行——模块图 + 任务→文件映射
│   └── progress.md                          已 gitignore——当前任务状态
└── code/                                    （better-code 专属）
    ├── protocol.md                          ≤15 行——研发认知约束
    ├── conventions.md                       代码约定 + 正反例
    └── danger-zones.md                      高风险文件 + 检查命令
```

**严格分层。** `shared/index.md` 纯项目知识，零认知规则。`code/protocol.md` 纯认知约束，零项目事实。两者一起通过 CLAUDE.md `@` 引用加载，职责各自单一。

### 设计取舍

| 选择 | 为什么 |
|------|--------|
| 知识放在产品仓库外 | 团队可以独立共享项目知识，不必和产品代码共用权限体系 |
| 项目目录里的 `.better-work/` 是符号链接 | 产品仓库保持干净，符号链接本身已 gitignore |
| 每个项目独立 git repo | 不同项目可能共享给不同团队，合并成一个 repo 会越界 |
| `shared/` 所有子技能都可读写 | 知识是共同资产，不应该由某一个学科独占 |
| `index.md` 硬上限 150 行 | 强制结构性约束，溢出必须拆到 `map.md` 或模块级文件 |

## 在 Better-Work 系列里的位置

`better-code` 是 [Better-Work 系列](https://github.com/d-wwei/better-work-skill) 的研发学科子技能。整个系列是一组 AI agent skill，共享同一套项目知识树：

- `better-work` — 系列入口、项目初始化、通用执行协议
- `better-code` — 本 repo，研发知识和约束
- `better-test` — 测试知识和约束（[better-test](https://github.com/d-wwei/better-test)）
- `better-plan` / `better-design` / `better-write` — 规划中

装了 `better-work` 之后，`/better-work code <cmd>` 就是 `/better-code <cmd>` 的别名，行为完全一致（参数透传，不做检查）。

没装 `better-work` 的话，`/better-code` 照样能跑。通用执行协议你自己补——可以装 [cognitive-kernel](https://github.com/d-wwei/cognitive-kernel)，可以在 `CLAUDE.md` 里手写，也可以什么都不加（项目简单的话）。

## 接口契约

Better-Work 系列的每个子技能都必须暴露四条标准命令：

| 命令 | 承诺 |
|------|------|
| `init` | 幂等的首次初始化，不显式 `--force` 就不覆盖已有文件 |
| `update` | 增量更新，保留不相关内容 |
| `checkpoint` | 把状态写到 `shared/progress.md`，下次会话能解析 |
| `resume` | 读 `progress.md`，按文件/函数级汇报——不能是"快完成了"这种模糊状态 |

`better-code` 多加一条学科专属命令（`learn`）。通过 `/better-work code` 调用时，better-work 原样透传参数——不检查、不审计。

`shared/` 目录所有子技能都能读。如果 `better-code` 要往那里写（比如写 `shared/map.md`），commit message 会打上 `[better-code]` 标签标明来源。

## 已知限制

- **`update` 是手动的。** 没有文件监视器。任务跑完不跑 `/better-code update` 知识就会过期，直到你下次想起来。
- **大规模重构后 `shared/map.md` 会漂移。** better-code 发现不一致的条目会加 `[未验证]` 标记，但不会阻止你继续用。
- **产品仓库和 `~/.better-work/<project>/` 是两个 git 面。** 这是故意的——让团队能独立共享项目知识，不和产品代码耦合——代价是你要看两次 `git status`。
- **没有包管理器集成。** 每个平台都是 symlink 或 curl 安装。
- **一次一个平台。** 每个 adapter 写到各自的注入点（CLAUDE.md / `.cursor/rules` / AGENTS.md）。底层文件一样，但 Claude Code / Cursor / Gemini CLI 之间不共享会话上下文。

## License

MIT License.

---

问题、反馈或讨论：[GitHub issues](https://github.com/d-wwei/better-code/issues)。
