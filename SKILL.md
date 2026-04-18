---
name: better-code
description: "研发知识管理：为代码库构建持久化的项目知识和研发认知约束，支持多 agent 平台、团队共享、断点续传。Better-Work 系列子技能，可独立使用。触发场景：首次进入大型项目（/better-code init）、完成多文件任务后（/better-code update）、从上次中断处继续（/better-code resume）。独立命令 /better-code，或作为子技能 /better-work code <cmd>。"
license: MIT
---

# better-code

Better-Work 系列的研发学科子技能。为大型代码库构建平台无关、团队可共享、跨 session 可恢复的项目知识与研发认知约束。

## Stance

像资深工程师做入职交接——只讲能防止犯错的关键信息，用具体文件路径和代码示例，而非泛泛描述。宁可漏掉不重要的，也不写没验证的。信号驱动探索，不做"读完整个项目"。

## Commands

- `/better-code init` — 首次探索项目，生成 `.better-work/shared/` + `.better-work/code/` 知识库
- `/better-code update` — 基于本次会话信号，增量更新知识
- `/better-code checkpoint` — 保存当前任务进度到 `.better-work/shared/progress.md`
- `/better-code resume` — 从 `.better-work/shared/progress.md` 恢复
- `/better-code learn <topic>` — 深入调研某子系统并记录

作为子技能调用时：`/better-work code <cmd>` 等价于 `/better-code <cmd>`（仅当安装了 better-work）。

## Output Structure

知识文件集中存储于 `~/.better-work/<project>/`，项目目录通过符号链接访问：

```
<project>/.better-work/              ← 符号链接 → ~/.better-work/<project>/
├── protocol.md                      ← 通用执行标准（better-work 管，本 skill 不读写）
├── shared/                          ← 所有 skill 共用（读写均可）
│   ├── index.md                     ← 项目知识入口（≤150 行）
│   ├── map.md                       ← 模块地图 + 任务→文件映射（≤400 行）
│   └── progress.md                  ← 当前任务进度（.gitignore）
└── code/                            ← 研发专用
    ├── protocol.md                  ← 研发认知约束（≤15 行）
    ├── conventions.md               ← 编码约定
    └── danger-zones.md              ← 高风险区域
```

### CLAUDE.md 注入

```
@.better-work/shared/index.md        ← 项目知识（共享）
@.better-work/code/protocol.md       ← 研发认知约束
```

两个文件职责严格分离：
- `shared/index.md` — 纯项目知识，零认知规则
- `code/protocol.md` — 纯研发认知约束，零项目信息

如果同时装了 better-work，它会往 CLAUDE.md 追加 `@.better-work/protocol.md`（通用执行标准）；better-code 不重复注入。

**独立安装场景**（不装 better-work）：`~/.better-work/<project>/protocol.md` 不会生成——通用执行标准需用户自行在全局 `~/.claude/CLAUDE.md` 维护。better-code 只保证研发学科约束（`code/protocol.md`）和项目知识（`shared/index.md`）注入。

## Red Lines

1. `shared/index.md` 超过 150 行 → 必须分流到其他文件，不可突破
2. `code/protocol.md` 超过 15 行 → 过长，精简或拆分
3. `shared/map.md` 超过 400 行 → 必须按模块拆分到 `shared/map-<module>.md`
4. `shared/index.md` 中出现认知规则，或 `code/protocol.md` 中出现项目信息 → 职责混淆，移到正确文件
5. 写入任何事实性声明时没有附带源文件路径或 `[未验证]` 标签 → 违规
6. `init` 时全文读超过 30 个文件 → 过深，改用签名/结构扫描 + 采样阅读
7. 生成内容与项目 README 重复 → 违规，删除重复部分
8. `update` 写入推测性内容但不标注 `[未验证]` → 违规
9. `danger-zones.md` 条目缺少影响范围或检查命令 → 不完整，补全后再写入
10. `init` 未判断项目类型就开始探索 → 违规，先分类再探索
11. `progress.md` 记录模糊状态（如"快完成了"） → 违规，必须具体到文件和函数级别
12. init/update 完成后，`<project>/.better-work/` 不是指向 `~/.better-work/<project>/` 的符号链接（例如变成了实体目录或断链） → 违规，知识必须集中存储
13. init 覆盖 `shared/` 下已有的 `index.md`/`map.md` 内容 → 违规。允许的操作仅限：文件不存在时整体生成；文件存在且某章节为空时追加该章节内容

## Acceptance Criteria

1. 新对话加载 `shared/index.md` 后，agent 定位目标文件的搜索次数 ≤ 2（对比无知识时 5+）
2. `danger-zones.md` 中每个条目都可通过其"检查命令"验证影响范围
3. `checkpoint` + `resume` 后，agent 能准确复述上次进度且无遗漏
4. 同一套 `.better-work/` 文件通过不同 adapter 注入后，在 Claude Code / Cursor / Gemini CLI 各自新会话首轮询问"本项目 top 3 danger zones"，agent 能背出 `code/danger-zones.md` 前 3 条的文件名
5. `/better-code init` 可独立执行（不依赖 better-work）；与 better-test 等同系列 skill 共存时，`shared/` 复用而非重建

## References

| File | Load When | Content |
|------|-----------|---------|
| `references/init-workflow.md` | `/better-code init` | 完整探索流程（项目分类 + 信号驱动探索） |
| `references/update-workflow.md` | `/better-code update` | 信号检测 + 增量更新逻辑 |
| `references/progress-workflow.md` | `/better-code checkpoint` 或 `resume` | 断点续传机制 |
| `references/learn-workflow.md` | `/better-code learn <topic>` | 子系统深度调研流程 |
| `references/templates.md` | 生成输出文件时 | 各文件模板 + 质量标准 |
| `references/adapters.md` | 跨平台注入时 | 多平台注入方法 |
