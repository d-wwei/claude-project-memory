# Progress Workflow — 断点续传

管理 `.better-work/shared/progress.md` 文件，实现跨 session 的任务接力。该文件位于 `shared/` 下，可被 better-work 系列的其他 skill 读取。

## Checkpoint（保存断点）

当用户执行 `/better-code checkpoint`（或 `/better-work code checkpoint`）或 session 即将结束时执行。

### 写入内容

```markdown
# progress.md — 任务进度
# 最后更新: [ISO 时间戳]
# 维护者: better-code (本次)

## 当前任务
[一句话描述正在做什么]

## 已完成
- [x] [具体动作 + 涉及的文件路径:行号]
- [x] [具体动作 + 涉及的文件路径:行号]

## 进行中
- [ ] [当前正在做的事]
  - 已完成：[做到哪一步了，精确到函数/方法级别]
  - 未完成：[还剩什么没做]
  - 卡在：[如果有阻塞，描述阻塞原因]

## 待做
- [ ] [接下来要做的事，按顺序]
- [ ] [...]

## 关键决策
- [做了什么决策 + 为什么这样选择]
- [如果有被否定的方案，简述为什么否定]

## 恢复上下文
[给下一个 session 的 agent 的交接信息：]
- 当前在哪个文件的哪个位置
- 需要参考什么文档/资料
- 需要注意的特殊情况
- 相关的测试怎么跑

## 相关知识
- 本任务涉及的 danger-zones 条目：[引用 code/danger-zones.md 中的文件]
- 本任务涉及的 conventions：[引用 code/conventions.md 中的约定名]
```

### 质量标准

每条记录必须满足"下一个 agent 不用问用户就能接上"的标准：

| ✓ 可接受 | ✗ 不可接受 |
|----------|-----------|
| "已完成 AlipayProvider.create_order()，见 src/payment/alipay.py:45" | "差不多做完了支付部分" |
| "卡在：Alipay 签名验证失败，RSA2 公钥格式不对，需要查 Alipay 文档的密钥格式要求" | "卡在签名那里" |
| "待做：修改 order_service.py:78 的 process_payment() 从直接调用 stripe 改为调用 provider.charge()" | "待做：改 order service" |

### 多任务情况

如果同时有多个进行中的任务（或一个大任务的多个并行分支），用 `## 任务 1` / `## 任务 2` 分隔，每个任务独立追踪。

### 与其他 skill 的协作

- better-test 做测试时也可能写 progress.md（比如"正在跑 Group C 第 3 项"），格式与上述相同
- 跨 skill 切换任务时，保留对方的条目，只添加自己的
- 用 `## 维护者: <skill>` 标注每个任务条目的写入方

---

## Resume（从断点恢复）

当用户执行 `/better-code resume` 或新 session 开始时说"继续上次的"。

### 执行步骤

1. 读取 `.better-work/shared/progress.md`
2. 向用户汇报：
   ```
   上次进度：
   - 任务：[任务描述]
   - 已完成 N 项，进行中 M 项，待做 K 项
   - 上次停在：[恢复上下文中的位置信息]
   - 距上次更新：[时间差]
   ```
3. 如果距上次更新超过 7 天，主动检查期间代码变化：
   - 运行 `git log --oneline --since='<checkpoint-date>' -- .` 查看相关文件的改动
   - 运行 `git diff <checkpoint-commit>..HEAD` 看整体差异
   - 向用户汇报期间有哪些 commit / 哪些文件被改，再询问是否继续原计划
   - 注意：此处**不**运行 `/better-code update`——update 是信号驱动的，新 session 尚无会话信号，跑了也无效
4. 询问用户："从 [进行中的任务] 继续，还是有新的安排？"
5. 用户确认后，按"恢复上下文"中提到的文件开始工作

### 冲突处理

如果 `progress.md` 记录的文件在上次 checkpoint 之后被修改过（git 显示有新 commit）：
- 报告哪些文件有变更
- 读取变更内容
- 评估是否影响待做任务的方案
- 如果有冲突，向用户说明并调整计划

---

## 自动 Checkpoint 建议

Agent 检测到以下情况时，主动建议用户执行 checkpoint：

1. 已经连续工作超过 30 分钟且有实质进展
2. 对话长度接近 context 压缩阈值
3. 完成了一个阶段性目标（如"所有单元测试通过"）
4. 即将开始风险较高的操作（重构、删除、迁移）

建议话术："建议保存断点——如果接下来的操作出问题，可以从这里恢复。要我执行 checkpoint 吗？"

## .gitignore 规则

`shared/progress.md` 默认在 `~/.better-work/<project>/.gitignore` 中，不会被团队共享。

如果团队希望协作推进同一任务，可以从 `.gitignore` 移除 `progress.md`，让它进入版本控制。此时 checkpoint 时需更谨慎避免敏感信息（比如本地路径、未公开的方案细节）。
