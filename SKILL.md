---
name: better-code
description: "研发知识管理：为代码库构建持久化的项目知识和研发认知约束，支持多 agent 平台、团队共享和断点续传。Better-Work 系列子技能。触发场景：首次进入大型项目、完成多文件任务后、需要从上次中断处继续。独立命令 /better-code，或作为子技能 /better-work code。"
---

# Project Memory

为大型代码库（10k+ 行）构建和维护平台无关的项目记忆系统。

## Stance

像一个资深工程师在做入职交接——只讲能防止犯错的关键信息，用具体文件路径和代码示例而非泛泛描述。宁可漏掉不重要的，也不写没验证的。

## Commands

- `/project-memory init` — 首次探索项目，生成 `.project-memory/` 知识库
- `/project-memory update` — 基于本次会话的信号，增量更新知识库
- `/project-memory checkpoint` — 保存当前任务进度，供下次会话恢复
- `/project-memory resume` — 从上次断点恢复，汇报进度并继续
- `/project-memory learn <topic>` — 深入调研某个子系统并记录
- `/project-memory inject <platform>` — 为指定平台生成注入配置

## Output Structure

```
.project-memory/                      ← 项目根目录，git 追踪，团队共享
├── index.md                          ← 项目知识入口（≤150 行），每次对话加载
├── cognitive-protocol.md             ← 该项目的认知约束（≤30 行），每次对话加载
├── map.md                            ← 模块地图 + 任务→文件映射 + 依赖方向
├── conventions.md                    ← 编码约定（正确/错误对比对 + 为什么）
├── danger-zones.md                   ← 高风险区域（入度、影响范围、检查命令）
├── progress.md                       ← 当前任务进度（断点续传）
└── adapters/                         ← 平台注入配置（自动生成）
    ├── claude.md
    ├── cursor.md
    └── gemini.md
```

**双文件注入**（以 Claude Code 为例）：
```
<project>/CLAUDE.md:
  @.project-memory/index.md                 ← Context: 项目知识
  @.project-memory/cognitive-protocol.md    ← Control: 认知约束
```

两个文件职责严格分离：
- `index.md` — 纯项目知识（架构、约定、危险区域），不含任何认知规则
- `cognitive-protocol.md` — 纯认知约束（自检规则、触发器），不含项目信息。可按项目风险等级调整严格程度

## Red Lines

1. `index.md` 超过 150 行 → 必须将内容分流到其他文件，不可突破
2. `cognitive-protocol.md` 超过 30 行 → 过长，精简或拆分
3. `index.md` 中出现认知规则，或 `cognitive-protocol.md` 中出现项目信息 → 职责混淆，移到正确文件
4. 写入任何事实性声明时没有附带源文件路径或 `[未验证]` 标签 → 违规
5. `init` 时详细阅读（全文 read）超过 30 个文件 → 过深，改用签名/结构扫描
6. 生成内容与项目 README 重复 → 违规，删除重复部分
7. `update` 写入推测性内容但不标注 `[未验证]` → 违规
8. `danger-zones.md` 条目缺少影响范围（哪些模块会受影响）或检查命令 → 不完整，补全后再写入
9. `init` 未判断项目类型就开始探索 → 违规，先分类再探索
10. `progress.md` 中记录无法被下一个 session 理解的模糊状态（如"快完成了"） → 违规，必须写具体到文件和函数级别

## Acceptance Criteria

1. 新对话加载 `index.md` 后，agent 定位目标文件的搜索次数 ≤ 2（对比无记忆时 5+）
2. `danger-zones.md` 中列出的每个条目，均可通过其"检查命令"验证影响范围
3. `checkpoint` + `resume` 后，agent 能准确复述上次进度且无遗漏
4. 同一套 `.project-memory/` 文件通过不同 adapter 注入后，在 Claude Code、Cursor、Gemini CLI 上均可被 agent 正确读取

## References

| File | Load When | Content |
|------|-----------|---------|
| `references/init-workflow.md` | `/project-memory init` | 完整探索流程（项目分类 + 信号驱动探索） |
| `references/update-workflow.md` | `/project-memory update` | 信号检测 + 增量更新逻辑 |
| `references/progress-workflow.md` | `/project-memory checkpoint` 或 `resume` | 断点续传机制 |
| `references/templates.md` | 生成输出文件时 | 各文件的模板 + 质量标准 |
| `references/adapters.md` | `/project-memory inject` | 多平台注入方法 |
