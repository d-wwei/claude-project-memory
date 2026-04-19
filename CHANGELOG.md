# Changelog

All notable changes to **better-code** (Better-Work series development subskill) are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); this project uses [Semantic Versioning](https://semver.org/).

## [1.3.1] - 2026-04-19

### Fixed
- `init-workflow.md` Step 0: added the empty-`~/.better-work/<project>/` directory branch (previously a gap between "doesn't exist" and "has shared/"). Empty dir is now handled equivalently to "doesn't exist" with a warning in Step 6 report

### Added
- `init-workflow.md` Step 4: new "知识 repo 的 .gitignore 标准模板" subsection. References `better-work/references/gitignore-template.md` as canonical source with inline fallback for standalone installs

## [1.3.0] - 2026-04-19

### Changed
- README rewritten with **"blind agent" framing**: the 5 failure modes on 100k-line codebases (insufficient search / architectural blindness / context exhaustion / wrong tests / unknown conventions) map directly to what the 150-line `shared/index.md` prevents
- `docs/better-work-architecture.md` migrated to the meta repo (`docs/architecture.md` in the private `better-work-skills-dev-repo`); better-code no longer carries the architecture doc in its own tree (simpler public surface)

## [1.2.0] - 2026-04-18

### Added
- `LICENSE` (MIT)
- `README.md` — 207 lines, value proposition + install + command reference + output structure + series relation + interface contract
- `README.zh-CN.md` — native Chinese version (not a translation)

### Fixed
- `init-workflow.md` Step 3b.4: risk-level prompt now has unattended fallback (default `standard` in fork/CI sessions without interactive channel)
- `init-workflow.md` Step 5: clarified CLAUDE.md is created if missing

### Changed
- Refactored from `project-memory` to Better-Work series architecture:
  - Command namespace: `/project-memory` → `/better-code`
  - Output directory: `.project-memory/` → `.better-work/shared/` + `.better-work/code/`
  - Added `references/learn-workflow.md` (subsystem deep-dive command)
- Architecture doc revised v1.1 → v1.2 (recorded Phase G implementation: Cognitive Layer + protocol.md auto-inject)
- Removed `diagnosis/` directory (skill boost artifacts, not runtime-relevant)

## Prior to 1.2.0

Earlier iterations of `project-memory` are outside the Better-Work series scope. The v1.2.0 tag marks the series-aligned first stable release.

---

[1.3.1]: https://github.com/d-wwei/better-code/compare/v1.3.0...v1.3.1
[1.3.0]: https://github.com/d-wwei/better-code/compare/v1.2.0...v1.3.0
[1.2.0]: https://github.com/d-wwei/better-code/releases/tag/v1.2.0
