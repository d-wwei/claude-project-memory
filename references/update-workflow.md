# Update Workflow

基于本次会话中的工作信号，增量更新 `.better-work/` 下的知识。

## 核心原则

Update 不是"回顾并总结"，而是**检测信号并响应**。只有明确信号才触发更新，没有信号就不动。

## 信号检测

回顾本次会话，逐项检测以下信号：

### 信号 1：搜索挫折 → 补充 shared/map.md

**检测**：本次会话中是否发生了"搜索 3+ 次才找到目标文件"的情况？

**响应**：在 `shared/map.md` 的"任务→文件映射"表中添加本次的映射关系。

```markdown
# 添加到 shared/map.md

| 要做什么 | 先看这里 | 然后看这里 |
|----------|---------|-----------|
| [本次任务描述] | [最终找到的文件] | [相关联的文件] |
```

### 信号 2：意外测试失败 → 补充 code/danger-zones.md

**检测**：修改了某个文件后，是否导致了**不在预期内的**其他测试失败？

**响应**：在 `code/danger-zones.md` 记录这个依赖关系。

```markdown
# 添加到 code/danger-zones.md

## [被修改的文件]

- **入度**: [有多少文件依赖它]
- **依赖方**: [列出主要依赖方]
- **为什么危险**: 修改它会破坏 [意外失败的测试所在模块]
- **改之前做**: [能提前发现影响的搜索命令]
- **历史事故**: 本次 session 发现（[日期]）
```

### 信号 3：发现约定 → 补充 code/conventions.md

**检测**：本次会话中是否发现了"项目中一致遵循的模式"，且该模式之前未记录？

来源：
- code review 中被要求修改的代码风格
- 发现多个文件用同一种方式处理某个问题
- 阅读 lint 规则或 CI 检查发现的约束

**响应**：在 `code/conventions.md` 中用正确/错误对比对格式记录。

```markdown
# 添加到 code/conventions.md

## [约定名称]

✓ 正确:
[实际代码示例，标注文件路径:行号]

✗ 错误:
[反面示例]

为什么: [一句话解释后果]
```

### 信号 4：知识修正 → 更新已有内容

**检测**：本次会话中是否发现了 `.better-work/` 中已有信息**不再准确**的情况？

来源：
- 重构后文件路径变了
- 约定被新实践取代
- 危险区域被重新设计，不再危险

**响应**：直接修正对应文件中的过时信息。不保留旧版本（git 会追踪）。

### 信号 5：新模块/新功能 → 扩展 shared/map.md 和 shared/index.md

**检测**：本次任务是否新增了模块、服务、或重要的新文件？

**响应**：
- `shared/map.md` 的"目录结构与职责"、"任务→文件映射"追加新条目
- 若新模块影响项目整体架构认知，同步更新 `shared/index.md`（保持 ≤150 行）

## 优先级权重

| 信号来源 | 权重 | 理由 |
|----------|------|------|
| 实际踩坑（搜索挫折、意外失败） | 最高 | 来自真实痛苦，确定有用 |
| code review 反馈 | 高 | 来自人类判断 |
| 采样观察到的模式 | 中 | 可能只是局部现象 |
| 推测性结论 | 不记录 | 除非标注 [未验证] |

## 更新流程

1. 扫描本次会话的信号（按上述 5 类）
2. 读取 `.better-work/` 中对应文件的当前内容
3. 按信号类型执行增量修改
4. 如果 `shared/index.md` 需要同步更新（新增了重要模块或关键约定），一并修改
5. 检查 `shared/index.md` 是否仍然 ≤ 150 行、`shared/map.md` 仍然 ≤ 400 行
6. 若本次 update 修改了 `shared/index.md` 或 `code/protocol.md`（即 CLAUDE.md `@` 引用的两个文件），刷新所有已注入的 adapter 产物：
   - Claude Code/Gemini CLI 使用 `@` 引用，无需刷新
   - Cursor/Codex/OpenCode/OpenClaw 是内容嵌入，需同步重写 `.cursor/rules/better-work.mdc` 等（参考 `references/adapters.md`）
7. 向用户报告更新了什么（逐条列出信号和响应），以及是否触发了 adapter 刷新

## 来源标注

每条新增内容必须可追溯到信号：

```markdown
<!-- source: 信号2（意外测试失败）- session <session_id> - <date> -->
```

便于后续 review 时判断知识可信度。

## 不更新的情况

以下情况**不触发 update**，即使用户执行了命令：

- 本次会话只做了信息查询，没有修改代码
- 所有发现的知识已经在 `.better-work/` 中记录过
- 唯一的"新知识"是推测性的且无法标注具体来源

此时报告："未检测到新信号，项目知识保持不变。"

## shared/ 写入的协作规则

`shared/` 下的文件由多个 skill 共同维护。写入时遵守：

- 每次修改作为一个 git commit，message 格式：`[<skill>] <action>: <summary>`
  - `<skill>` 取值：`better-code` / `better-test` / `better-work` 等
  - `<action>` 取值表：`init` / `update` / `fix` / `refactor` / `prune`
  - 示例：`[better-code] update: add payment_service to danger-zones`
- 不要删除其他 skill 写入的内容（除非信号 4 明确判定过时）
- 冲突（同一段落被两个 skill 改动）→ 标记 `<!-- CONFLICT: <skill-a> vs <skill-b> -->`，等人类决策
