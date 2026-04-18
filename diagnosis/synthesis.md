# Synthesis — claude-project-memory

## Cross-Reference Summary

| Artifact | Key Finding |
|----------|-------------|
| `structural-audit.md` | 5/9 checks FAIL. Missing: triggers, red lines, acceptance criteria, stance, ADR |
| `layer-diagnosis.md` | Layer 1 (template). No scenario adaptation, no quality benchmarks |
| `token-audit.md` | Currently healthy (751 words), but will need decomposition after upgrades |
| `platform-check.md` | Low severity. Shell commands used as examples, not mandates. Semantic verbs preferred |
| `domain-research.md` | 7 sources analyzed. Strongest differentiator: learn-from-doing loop. Biggest gaps: project-type adaptation, quality criteria, decay mechanism |
| `constraint-enforcement-audit.md` | 0% effective enforcement. Zero red lines defined |

## Rethinking Questions

### 1. Does the workflow need structural redesign?

**Partial redesign needed.** The three commands (init/update/learn) are sound. But:
- Init needs project-type classification before exploration (don't explore a CLI tool the same as a microservice platform)
- The output template is too rigid — different project types produce different CLAUDE.md structures
- Update needs priority weighting — knowledge from task failures > knowledge from reading code

The core concept is validated by domain research (Cline's memory bank, Windsurf's cascade memory, Aider's repo-map all solve similar problems). But the current implementation is Layer 1: a fixed template applied uniformly.

### 2. Are the skill's principles still valid?

**Yes, with refinement.** The tiered approach (CLAUDE.md < 150 lines + @docs) is validated by:
- Claude Code auto-memory's 200 line limit (source 2)
- Cursor's rules-vs-docs separation (source 3)
- Developer onboarding literature's "2-5 pages max" (source 4)
- Cline's multi-file memory structure (source 6)

But principles need to become **enforceable constraints** rather than advisory guidelines.

### 3. What does the quality bar look like?

From domain research, a good project memory system:
- Classifies project type before exploring (source: expert onboarding practice)
- Prioritizes API surface over implementation (sources 1, 5)
- Distinguishes "declared knowledge" from "discovered knowledge" (source 7)
- Has explicit decay/pruning rules (source 6)
- Defines quality criteria for each output section (not just "write a Danger Zones section" but "each entry must include: what breaks, blast radius, how to verify")
- Adapts depth to project size (source: developer onboarding practice)

---

## Architecture Prescription

| Defect | Fix |
|--------|-----|
| No trigger phrases | Add 3rd-person triggers: "when entering a new project for the first time", "after completing a multi-file task", "when needing to understand a specific subsystem" |
| No red lines | Design 7+ constraints from domain anti-patterns (see below) |
| No acceptance criteria | Define 4 testable criteria from user perspective |
| No stance | Define cognitive position: "Thinks like a senior engineer onboarding a teammate" |
| No layer decomposition | Split into router SKILL.md + references/ for workflows |
| No ADR | Create docs/adr/ with decision for tiered structure |

### Proposed Red Lines

1. Generated CLAUDE.md exceeds 150 lines → split to @docs before writing
2. Any factual claim in CLAUDE.md without a source file path or [未验证] tag → violation
3. Init explores more than 30 files in detail (full read) → too deep, stay at surface (signatures/structure)
4. Generated content duplicates information available in project's README → violation
5. Update adds speculative/unverified information without [未验证] tag → violation
6. Generated Danger Zones entry lacks blast radius (what breaks if this file is modified) → incomplete
7. Init proceeds without determining project type classification → violation

### Proposed Acceptance Criteria

1. Generated CLAUDE.md, when used in a new conversation, enables the agent to correctly identify which file to modify for a given task within 2 searches (not 5+)
2. Danger Zones section, when read before making changes, prevents at least one common mistake per session
3. Update captures at least one non-obvious learning per multi-file task
4. Generated output is ≤ 150 lines for CLAUDE.md with overflow in @docs

## Mechanical Prescription

| Issue | Fix |
|-------|-----|
| Shell commands as examples | Replace with semantic descriptions: "Search source files by extension", "Search contents for import patterns" |
| Mixed imperative/descriptive style | Rewrite all instructions in imperative form |

## Content Prescription

### Layer 1 → Layer 2 Upgrade Plan

Based on domain research, add **project type classification** with 5 scenarios:

| Project Type | Exploration Focus | CLAUDE.md Emphasis |
|--------------|-------------------|-------------------|
| Backend API / Web service | Routes, middleware, DB access, auth | API patterns, DB conventions, auth flow |
| CLI tool / Library | Public API surface, argument parsing | Public interface, versioning, testing |
| Frontend SPA | Component tree, state management, API calls | Component patterns, state conventions |
| Data pipeline / ML | DAGs, transformers, data schemas | Data flow, schema evolution, job orchestration |
| Monorepo / Platform | Package boundaries, shared libs, CI | Ownership map, cross-package dependencies |

Each scenario has tailored exploration steps and output template emphasis.

### Quality Criteria for Output Sections

Define what "good" looks like for each CLAUDE.md section:

- **Architecture**: Must include calling direction. Must name all top-level modules.
- **Must-Know Rules**: Each rule must be something that, if violated, causes a bug or failed review. Not preferences.
- **Danger Zones**: Each entry must include: file path, why it's dangerous (blast radius), how to check impact.
- **Gotchas**: Each must describe: what you'd naively do, why it's wrong, what to do instead.
- **Useful Searches**: Each must be a working command/pattern, not a vague description.

### Token Architecture After Upgrade

```
SKILL.md (router)                    ≤ 800 words
├── references/init-workflow.md      ≤ 2000 words (exploration steps by project type)
├── references/update-workflow.md    ≤ 1000 words (update logic + priority weighting)
├── references/templates.md          ≤ 1500 words (output templates + quality criteria)
└── references/scenarios.md          ≤ 1000 words (project type classification guide)
```

Always-loaded: ~800 words (SKILL.md only). Total with all references: ~5300 words. Each reference loaded only when its command is invoked.
