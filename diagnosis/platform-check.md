# Platform Compatibility Check — claude-project-memory

## Search Method

Scanned SKILL.md for platform-specific tool names, absolute paths, and platform-dependent features.

## Findings

| Issue | Location | Severity |
|-------|----------|----------|
| `find . -type f -name "*.py"` | Step 2 (line 33) | Low — works on macOS/Linux but not Windows |
| `find . -type d -maxdepth 3` | Step 2 (line 34) | Low — same |
| `grep -rh "^from \|^import " src/` | Step 6 (line 57) | Low — Unix grep syntax |
| `find src/ -name "*.py" -exec wc -l {} \;` | Step 6 (line 61) | Low — Unix-specific |
| `grep -r 'send_notification\|notify' src/` | Section 2.1 (line 191) | Low — example only |

## Assessment

The skill uses shell commands as **examples** within exploration steps, not as hard-coded tool invocations. These are guidance for the agent, which will translate them to platform-appropriate tools (Glob, Grep, etc.) at runtime.

However, for cross-platform distribution (Codex, Gemini, Cursor), these should use **semantic verbs** instead:
- "find . -type f" → "Search for source files by extension"
- "grep -rh" → "Search file contents for import patterns"

**Verdict**: Low severity. The commands are examples, not mandates. But upgrading to semantic verbs would improve clarity and cross-platform portability.

## Platform-Specific Tool Names

No direct use of platform-specific tool names (Read, Write, Edit, Bash, Glob, Grep). The skill uses generic descriptions. CLEAN on this dimension.
