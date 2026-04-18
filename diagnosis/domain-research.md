# Domain Research — Project Context Memory for AI Coding Agents

## Sources

| # | Source | Type | Key Finding | Implication for Skill |
|---|--------|------|-------------|----------------------|
| 1 | **Aider's repo-map** (aider.chat) | Tool | Generates a condensed map of the entire repository using tree-sitter AST — shows file structure + function/class signatures without full code. Automatically determines which files are most relevant to current task via embedding similarity. | Init should prioritize structural overview (signatures, not full code). Relevance-based loading > exhaustive scanning |
| 2 | **Claude Code's auto-memory** (built-in `~/.claude/projects/` memory) | Platform feature | Persists learnings across sessions automatically. Stores semantic memories ("user prefers X", "project uses Y pattern"). Loaded into context on every conversation start. Hard limit of 200 lines before truncation. | The skill competes with built-in memory. Must differentiate: auto-memory captures preferences/corrections; project-memory captures structural knowledge (architecture, flows, danger zones). The 200-line limit validates the <150 line CLAUDE.md constraint. |
| 3 | **Cursor's .cursorrules + docs context** | Platform feature | Project rules loaded per-workspace. Separate "docs" feature indexes external documentation. Key insight: rules are short imperative constraints; context docs are long reference material. Two distinct modes of knowledge. | Validates the tiered approach (CLAUDE.md = rules/constraints, @docs = reference material). But Cursor's approach doesn't have "learn over time" — it's static. The update/learn commands are differentiators. |
| 4 | **Developer onboarding literature** (industry practice) | Domain knowledge | New developer onboarding typically focuses on: (1) architecture overview, (2) local dev setup, (3) "where to find things", (4) "who owns what", (5) "don't touch these". Average effective onboarding doc is 2-5 pages. Beyond that, people stop reading. | Confirms: concise > comprehensive. But also reveals a missing category: "who owns what" / ownership boundaries. Large projects have team boundaries that affect where changes go. |
| 5 | **Repomix / repo-to-text tools** | Tool | Serialize entire repos into single files for LLM consumption. Key lesson: naive serialization produces 100k+ tokens that overflow context. Effective approaches use: (a) file-level summaries, (b) dependency graphs, (c) API surface extraction. | Init should NOT try to "read everything." Extract API surfaces and dependency structure, not implementation details. Implementation is read on-demand during tasks. |
| 6 | **Cline/Roo Code's memory bank** | Tool | Uses a structured memory system with: productContext.md, activeContext.md, techContext.md, systemPatterns.md, progress.md. Each file has a specific role. Explicitly separated into "read every time" vs "read when needed." Also has a "memory decay" concept where stale info gets pruned. | Multiple distinct memory files > one monolithic file. The current skill's "Deep Docs" approach aligns, but lacks: (a) active context tracking (what's being worked on), (b) explicit decay/pruning rules. |
| 7 | **Windsurf's cascade memory** | Platform feature | Maintains a rolling "project understanding" that updates automatically. Key innovation: it tracks what the agent learned FROM DOING TASKS, not just from reading code. Distinguishes between "declared knowledge" (from docs) and "discovered knowledge" (from experience). | The update command captures "discovered knowledge" — this is a strong differentiator. But the skill doesn't distinguish between declared vs discovered, or how to weight them differently. Discovered knowledge (from actual task failures) should be weighted higher. |

## Expert Workflow Mapping

| Expert Practice | Current Skill Coverage | Gap |
|----------------|----------------------|-----|
| Architecture overview first | Step 1-3 in init | Covered but no quality criteria for "good enough" |
| Size-appropriate depth | Not addressed | Gap — 10k project needs different exploration than 500k |
| Distinguish API surface from implementation | Implicit in "按需加载" | Gap — init tries to read implementation too early |
| Track what was learned from failures | Type B in update | Partially covered — but no priority weighting |
| Decay/prune stale information | One line: "连续 5 次没用到就移除" | Weak — no concrete mechanism |
| Separate rules from reference | CLAUDE.md vs @docs split | Covered |
| Team/ownership boundaries | Not addressed | Gap — who owns what module, affects PR routing |
| Active task context | Not addressed | Gap — what's being worked on right now |
| Project type classification | Not addressed | Gap — different project types need different exploration |

## Key Insights

1. **The skill's strongest differentiator is the "learn from doing" loop** (update command). Most competing tools either do static analysis or require manual updates. The auto-update-after-task pattern is genuinely novel.

2. **The biggest gap is lack of adaptation by project type/size.** A 10k-line CLI tool and a 500k-line microservices platform need fundamentally different exploration strategies and CLAUDE.md structures.

3. **Quality criteria are missing.** The skill says "write Danger Zones" but doesn't define what makes a good Danger Zones section vs a superficial one. Expert onboarding docs have clear quality bars (e.g., "each danger zone must include: what breaks, who's affected, how to check").
