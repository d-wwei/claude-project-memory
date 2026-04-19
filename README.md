**🇺🇸 English** | **[🇨🇳 中文](README.zh-CN.md)**

# better-code

### Stop making your AI agent rediscover your codebase every session.

`better-code` is a Claude Code skill that builds a persistent, project-specific memory for your AI agent. Instead of blindly searching the repo on every new session, it boots with a tight index of the architecture, the coding conventions, and the zones where past changes have burned you.

Works natively on Claude Code. Ships with adapters for Cursor, Gemini CLI, Codex, and OpenCode. The knowledge itself lives in plain markdown — portable, team-shareable, and version-controlled as its own git repo.

Part of the [Better-Work series](https://github.com/d-wwei/better-work-skill). Installs and runs standalone — no dependency on the rest of the series.

## Why This?

Every new session with a large codebase starts the same way. The agent `grep`s for a class it already saw yesterday. It asks which file owns the HTTP routes, when last Tuesday you told it. It proposes a refactor inside `payment_service.py` — the file you marked "don't touch, three teams depend on this."

The usual fix is to stuff more context into `CLAUDE.md`. But `CLAUDE.md` grows into an unmaintainable 400-line essay, nobody remembers what's in there, and half of it is stale within a month.

Project knowledge comes in two flavors that age at different speeds:

- **Architecture, conventions, danger zones** change slowly — on the order of months. They belong in durable knowledge files, loaded once per session.
- **Task state, open questions, in-flight decisions** change by the hour. They belong in session-scoped progress notes that are rewritten freely.

`better-code` separates the two. Durable knowledge goes into `shared/index.md` and `code/` — loaded on every session via `@`-reference. Progress lives in `shared/progress.md` — rewritten by `checkpoint` / `resume` without touching the durable files.

## Installation

### Claude Code (native)

```bash
git clone https://github.com/d-wwei/better-code.git ~/repos/better-code
ln -s ~/repos/better-code ~/.claude/skills/better-code
```

`better-code` shows up in the skill list on the next Claude Code session.

### Other platforms

`references/adapters.md` has copy-paste install commands for:

- Cursor — `.cursor/rules/better-code.mdc`
- Gemini CLI — `GEMINI.md @reference`
- Codex — `AGENTS.md` embed
- OpenCode / OpenClaw

Knowledge files produced by `/better-code init` are platform-agnostic. Adapters differ only in how the project CLAUDE.md / rules file / AGENTS.md references them.

## Quick Start

Inside any project directory:

```
/better-code init
```

The skill classifies the project type, explores with signal-driven sampling (git history + dependency graph + filename patterns — not full reads), and writes:

- `.better-work/shared/index.md` — ≤150-line project entry point
- `.better-work/code/protocol.md` — ≤15 lines of coding cognitive constraints
- `.better-work/code/conventions.md` — concrete coding rules with before/after examples
- `.better-work/code/danger-zones.md` — files/modules where changes tend to break things

Then it injects `@.better-work/shared/index.md` + `@.better-work/code/protocol.md` into your project `CLAUDE.md`, so every subsequent session boots with context.

On the next session, the agent loads those files through `@`-references. First search drops to 1–2 hops instead of 5+.

After a few multi-file tasks, run:

```
/better-code update
```

This folds newly-found conventions and danger zones into the knowledge base. Signals drive the update — no full re-exploration.

### What `init` actually produces

After a first run on a typical Rust daemon project, you'll see:

```
<project>/
├── .better-work/              → ~/.better-work/<project>/
├── .gitignore                 (.better-work/ is excluded from the product repo)
└── CLAUDE.md                  (two new @ lines appended)
```

And in the knowledge repo:

```
~/.better-work/<project>/
├── .git/                      (this is its own git repo)
├── shared/
│   ├── index.md               119 lines — modules, entry points, key paths
│   └── progress.md            empty, gitignored
└── code/
    ├── protocol.md            14 lines
    ├── conventions.md         86 lines — 12 rules with examples
    └── danger-zones.md        53 lines — 7 files with check commands
```

## Command Reference

| Command | What it does |
|---------|--------------|
| `/better-code init` | First-time exploration + generate knowledge files + inject CLAUDE.md references |
| `/better-code update` | Signal-driven incremental update (new files, new conventions, new burns) |
| `/better-code checkpoint` | Save current task state to `shared/progress.md` for a future session to resume |
| `/better-code resume` | Read `progress.md`, continue the task with a file/function-level status report |
| `/better-code learn <topic>` | Deep-dive a subsystem; fold findings back into existing knowledge files |

All five work identically whether invoked directly, or via `/better-work code <cmd>` when `better-work` is installed.

### What changes after init

Before `/better-code init`, a typical first-session interaction looks like:

```
You:   Fix the timeout on the funds endpoint
Agent: Let me search for funds... grep... grep...
       Which file owns the HTTP routes?
       Where's the timeout defined?
       [5+ tool calls before a useful action]
```

After init, the same turn opens with context already loaded:

```
You:   Fix the timeout on the funds endpoint
Agent: Based on index.md, the REST handlers live in src/rest/mod.rs and funds-specific
       logic is in src/rest/funds.rs. danger-zones.md flags this path as high-risk
       (3 services depend on it). Let me read funds.rs first.
       [1 tool call, straight to the action]
```

Not magic. Just the agent reading the knowledge files before searching — which CLAUDE.md's `@`-references make automatic every session.

## Output Structure

Knowledge lives in `~/.better-work/<project-name>/` — its own git repo, separate from your product code. The project directory sees it through a symlink:

```
<project>/.better-work/                      → ~/.better-work/<project-name>/
├── protocol.md                              (better-work's; better-code does not write here)
├── shared/                                  (readable by all Better-Work subskills)
│   ├── index.md                             ≤150 lines — project entry point
│   ├── map.md                               ≤400 lines — module graph + task→file map
│   └── progress.md                          gitignored — current task state
└── code/                                    (better-code's own)
    ├── protocol.md                          ≤15 lines — coding cognitive constraints
    ├── conventions.md                       coding rules with before/after examples
    └── danger-zones.md                      high-risk files + check commands
```

**Strict separation.** `shared/index.md` is pure project knowledge — zero cognitive rules. `code/protocol.md` is pure cognitive constraints — zero project facts. Loaded together via CLAUDE.md `@` references, each file has a single job.

### Design decisions

| Choice | Why |
|--------|-----|
| Knowledge lives outside the product repo | Lets teams share project knowledge without coupling access to product code |
| `.better-work/` in the product dir is a symlink | Product repo stays clean; the symlink itself is gitignored |
| Each project is its own git repo | Different projects are shared with different teams; collapsing them into one repo would leak knowledge across boundaries |
| `shared/` is read-write for any subskill | Knowledge is a common asset, not owned by one discipline |
| `index.md` hard cap at 150 lines | Forces structural discipline; overflow must be split into `map.md` or module-specific files |

## Where better-code Fits in the Better-Work Series

`better-code` is the development-discipline subskill in the [Better-Work series](https://github.com/d-wwei/better-work-skill). The series is a family of AI-agent skills that share one project knowledge tree:

- `better-work` — series entry point, project init, and generic execution protocol
- `better-code` — this repo; coding knowledge and constraints
- `better-test` — testing knowledge and constraints ([better-test](https://github.com/d-wwei/better-test))
- `better-plan` / `better-design` / `better-write` — forthcoming

Installing `better-work` enables `/better-work code <cmd>` as an alias for `/better-code <cmd>` (identical behavior, forwarded unchanged).

Without `better-work`, `/better-code` still works. You supply the generic execution protocol yourself — either via the [cognitive-kernel](https://github.com/d-wwei/cognitive-kernel) skill, or a custom `CLAUDE.md` preamble, or nothing if your project is simple enough.

## Interface Contract

Every Better-Work subskill exposes these four standard commands:

| Command | Guarantee |
|---------|-----------|
| `init` | Idempotent first-time setup; never overwrites existing files without explicit `--force` |
| `update` | Incremental; preserves unrelated content |
| `checkpoint` | Writes `shared/progress.md` in a format the next session can parse |
| `resume` | Reads `progress.md`, reports file-level / function-level status — no vague "almost done" |

`better-code` adds one discipline-specific command (`learn`). When invoked via `/better-work code`, better-work forwards arguments unchanged — no inspection, no audit.

The `shared/` directory is readable by any Better-Work subskill. If `better-code` writes there (e.g., to `shared/map.md`), the commit message is tagged `[better-code]` for provenance.

## Limitations

- **`update` is manual.** No filesystem watcher. Finish a task without running `/better-code update` and the knowledge stays stale until next time you remember.
- **`shared/map.md` drifts after large refactors.** Stale entries get a `[未验证]` tag when better-code notices inconsistency, but the skill doesn't block on them.
- **Product repo and `~/.better-work/<project>/` are separate git surfaces.** Intentional — lets teams share project knowledge without coupling it to product code — but means two `git status` checks instead of one.
- **No package manager integration.** Installation is symlink or curl on every platform.
- **One platform per task.** Each adapter writes to its own injection point (CLAUDE.md vs `.cursor/rules` vs AGENTS.md). The underlying files are identical, but Claude Code, Cursor, and Gemini CLI don't share session context even when configured from the same source.

## License

MIT License.

---

Questions, issues, or discussion: [GitHub issues](https://github.com/d-wwei/better-code/issues).
