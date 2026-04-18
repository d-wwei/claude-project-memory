# Constraint Enforcement Audit — claude-project-memory

## Current State

The skill has **zero** red lines. No constraints are defined.

## Implicit Constraints (extracted from principles in §1.3)

| # | Implicit Constraint | Enforcement Axis | Mechanism |
|---|--------------------|--------------------|-----------|
| 1 | "只写确认过的信息" (only write verified info) | Think-only | No structural check |
| 2 | "CLAUDE.md 控制在 100-150 行" (line limit) | Think+Do possible | Could enforce via word count check in output |
| 3 | "不要重复 README" (no README duplication) | Think-only | No diff/comparison mechanism |
| 4 | "引用具体文件路径" (cite specific paths) | Think+Do possible | Could scan output for vague references like "在 service 层" |
| 5 | "连续 5 次没用到就移除" (decay rule) | Think-only | No tracking mechanism |

## Enforcement Ratio

- Think-only: 3/5 (60%)
- Think+Do: 2/5 (40%) — but NONE are actually implemented with Do-axis mechanisms
- **Effective enforcement ratio: 0%** (no constraint has structural enforcement)

## Top 3 Think-Only Constraints to Upgrade

1. **Line limit (100-150)** — Highest leverage. Can enforce by counting lines of generated CLAUDE.md and rejecting/splitting if over budget. Mechanism: word count gate before writing.

2. **Cite specific paths** — High leverage. Can scan generated content for vague location references ("在某个模块", "service 层有") without file paths. Mechanism: regex check for section content.

3. **Only verified info** — Hardest to enforce structurally, but could require "[来源: 读了 X 文件]" or "[未验证]" tags on every factual claim. Mechanism: mandatory source annotation.
