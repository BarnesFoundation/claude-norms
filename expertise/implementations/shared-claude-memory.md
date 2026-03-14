# Shared Claude Memory System

## Summary
Three-layer collaborative memory architecture enabling multiple developers and their Claude Code sessions to share strategy, decisions, and context through version-controlled files.

## Project
VXP 2.0 (BarnesFoundation/VXP-2.0), extracted to BarnesFoundation/claude-norms

## Key Files
- `CLAUDE.md` (any project root) — auto-loaded boot instructions pointing to Layer 3
- `docs/claude/STRATEGY.md` — team roadmap
- `docs/claude/DECISIONS.md` — ADR log
- `docs/claude/contributors/<name>.md` — per-developer context
- `docs/claude/context/<topic>.md` — feature deep-dives

## Approach
- Layer 1 (personal `~/.claude/` memory) + Layer 2 (`CLAUDE.md` auto-load) + Layer 3 (`docs/claude/` shared)
- CLAUDE.md is the bridge — without it, Layer 3 never gets read
- All instructions use imperative language (MUST/NEVER/ALWAYS) — conditional language causes models to skip steps
- Contributor files are owned by their developer's Claude — others read but NEVER modify
- Git merge conflicts on strategy files surface strategic divergence

## What Worked
- Claude sessions immediately pick up context from previous sessions
- Conflicts between developers' directions get flagged before they become code conflicts
- ADR log prevents decision relitigating
- Works as documentation even for non-Claude-using team members

## What Didn't / Gotchas
- **Conditional language fails**: "should read" gets skipped. "MUST read" gets followed. This is the single most important lesson.
- **CLAUDE.md must point to docs/claude/**: Without the pointer, the shared files are dead documentation that nobody reads.
- **Keep files concise**: These load into context windows. Verbose files waste tokens and dilute attention.

## Reuse Potential
Direct copy-paste. The pattern is project-agnostic. Full template and implementation guide in BarnesFoundation/claude-norms.

## People
- Steve (design)
- Claude/Steve/VXP2 (implementation)

## Date
2026-03-12 through 2026-03-13
