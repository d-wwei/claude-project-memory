**🇺🇸 English** | **[🇨🇳 中文](README.zh-CN.md)**

# better-code

### Your AI agent isn't lazy — it's blind. This repo hands it a map.

Here are the five ways AI coding agents actually fail on 100k-line codebases:

1. **Insufficient search** — changed an interface, didn't find every caller
2. **Doesn't know the architecture** — made the change at the wrong layer
3. **Context window exhausted** — read too much, forgot the earlier info
4. **Didn't run the right tests** — ran units, missed integration
5. **Doesn't know project conventions** — correct feature, wrong style

All five are **information problems**, not attitude problems. No amount of "be more careful" fixes them. What fixes them is handing the agent the information it needs, once, in a file it loads every session.

That's what `better-code` builds — the coding half of a [Full Context + Lite Control](https://github.com/d-wwei/better-work) framework. A 150-line project memory that tells the agent what it can't see from the code alone.

## Why CLAUDE.md alone doesn't scale

The natural first response to "agent forgets stuff" is to stuff more into `CLAUDE.md`. I did that. It grows into an unmaintainable 400-line essay. Nobody remembers what's in there. Half of it goes stale within a month.

Project knowledge actually comes in two flavors that age at different speeds:

- **Architecture, conventions, danger zones** change slowly — on the order of months. They belong in durable knowledge files, loaded once per session.
- **Task state, open questions, in-flight decisions** change by the hour. They belong in session-scoped progress notes, rewritten freely.

`better-code` separates them. Durable knowledge goes into `shared/index.md` and `code/` — loaded via `@`-reference every session. Progress lives in `shared/progress.md` — overwritten by `checkpoint` / `resume`.

## What the 150 lines contain

Each file targets a specific failure mode from the list above:

| File | Content | Fixes |
|------|---------|-------|
| `shared/index.md` (≤150) | Architecture overview, module boundaries, data flow | "Doesn't know the architecture" |
| `code/danger-zones.md` | High-import-count files + search commands to run before changing them | "Insufficient search" |
| `code/conventions.md` | Coding rules with right/wrong examples and the reason | "Doesn't know project conventions" |
| `shared/map.md` (≤400) | Task → file locator; "where does X live?" answered in one hop | "Context window exhausted" |
| `code/protocol.md` (≤15) | The cognitive floor specifically for coding work | Safety net for the rest |

Not written by hand. **Signal-driven generation**:

- `git log` analysis → hot files go in the active area
- `import` graph → high in-degree files become danger zones
- Agent searches 3+ times for a file → auto-add a locator entry
- A code change triggers unexpected test failures → auto-add a danger zone
- Agent violates a convention → record it as a before/after example

Not a one-time document generation. Continuous learning during actual use. Supports `checkpoint` / `resume` for cross-session continuity.

## What changes after init

Before `/better-code init`, a first-session turn looks like:

```
You:   Fix the timeout on the funds endpoint.
Agent: Let me search for funds... grep... grep...
       Which file owns the HTTP routes?
       Where's the timeout defined?
       [5+ tool calls before a useful action]
```

After init, the same turn opens with context already loaded:

```
You:   Fix the timeout on the funds endpoint.
Agent: index.md says REST handlers are in src/rest/mod.rs, funds-specific
       logic in src/rest/funds.rs. danger-zones.md flags this path as
       high-risk (3 services depend on it). Reading funds.rs now.
       [1 tool call, straight to action]
```

Not magic. Just the agent reading the knowledge files before searching — which CLAUDE.md's `@`-references make automatic every session.

## Installation

### Claude Code (native)

```bash
git clone https://github.com/d-wwei/better-code.git ~/repos/better-code
ln -s ~/repos/better-code ~/.claude/skills/better-code
```

### Other platforms

Adapters for Cursor, Gemini CLI, Codex, OpenCode, and OpenClaw live in `references/adapters.md`. The knowledge files produced by `/better-code init` are platform-agnostic; only the injection syntax differs per platform.

## Quick Start

Inside any project directory:

```
/better-code init
```

The skill classifies the project type, explores via signal-driven sampling (git history + dependency graph + filename patterns — not full reads), and writes:

- `.better-work/shared/index.md` — project entry point
- `.better-work/code/protocol.md` — coding cognitive constraints
- `.better-work/code/conventions.md` — concrete coding rules with examples
- `.better-work/code/danger-zones.md` — high-risk files + check commands

Then it injects `@.better-work/shared/index.md` + `@.better-work/code/protocol.md` into the project `CLAUDE.md`. Every subsequent session boots with project context.

After a few multi-file tasks:

```
/better-code update
```

This folds newly-discovered conventions and danger zones into the knowledge base. Signals drive the update — no full re-exploration.

## Command Reference

| Command | What it does |
|---------|--------------|
| `/better-code init` | First-time exploration + generate knowledge files + inject CLAUDE.md references |
| `/better-code update` | Signal-driven incremental update (new files, conventions, burns) |
| `/better-code checkpoint` | Save current task state to `progress.md` for a future session to resume |
| `/better-code resume` | Read `progress.md`, continue with file/function-level status |
| `/better-code learn <topic>` | Deep-dive a subsystem; fold findings back into existing files |

All five work identically whether invoked directly or via `/better-work code <cmd>` (when better-work is installed).

## Output Structure

Knowledge lives in `~/.better-work/<project-name>/` — its own git repo, separate from the product code. The project sees it through a symlink:

```
<project>/.better-work/                      → ~/.better-work/<project-name>/
├── protocol.md                              (better-work's; better-code doesn't write here)
├── shared/
│   ├── index.md                             ≤150 lines — project entry point
│   ├── map.md                               ≤400 lines — module graph + locator
│   └── progress.md                          gitignored — current task state
└── code/
    ├── protocol.md                          ≤15 lines — coding cognitive constraints
    ├── conventions.md                       coding rules with examples
    └── danger-zones.md                      high-risk files + check commands
```

`shared/index.md` is pure project knowledge, zero rules. `code/protocol.md` is pure cognitive constraints, zero project facts. Loaded together via CLAUDE.md `@` references. Each file does one job.

**Why external?** Project knowledge is a durable team asset. Putting it in `~/.better-work/<project-name>/` as its own git repo lets teams share it without coupling to the product repo's access controls. The product repo stays clean (the symlink is gitignored).

## The Better-Work series

- **[better-work](https://github.com/d-wwei/better-work)** — the Lite Control layer + series entry point. Start there for the full story of how this framework was built.
- **better-code** (this repo) — Full Context for coding
- **[better-test](https://github.com/d-wwei/better-test)** — Full Context for testing

`better-code` works standalone. Installing `better-work` enables `/better-work code <cmd>` as an alias with identical behavior.

## Limitations

- **`update` is manual.** No filesystem watcher. Finish a task without running `/better-code update` and the knowledge stays stale until you remember.
- **Signal-driven generation isn't omniscient.** First-run `index.md` may miss nuances that only surface after a week of using the project. That's what `update` is for.
- **Product repo and `~/.better-work/<project>/` are separate git surfaces.** Intentional — so teams can share project knowledge without coupling to product code — but two `git status` checks instead of one.
- **`shared/map.md` drifts after large refactors.** Stale entries get marked `[unverified]` when inconsistency shows up, but the skill doesn't block on them.
- **No package manager integration.** Installation is symlink or curl on every platform.

## License

MIT.

---

Companion write-up: the full [Full Context, Lite Control](https://github.com/d-wwei/better-work) story lives in the series entry-point README.

Questions, issues, discussion: [GitHub issues](https://github.com/d-wwei/better-code/issues).
