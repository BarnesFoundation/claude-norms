# Claude Norms

Cross-project norms and patterns for Claude Code AI-assisted development at the Barnes Foundation.

This repo is a living reference that any project's Claude Code session can read to bootstrap best practices. It captures hard-won lessons from real multi-developer, multi-AI projects.

## How to Use This Repo

**For humans**: Read these docs to understand the patterns your Claude sessions will follow.

**For Claude Code sessions**: Your project's `CLAUDE.md` or your developer can point you here. Read the relevant patterns, then apply them to your project. Save a reference to this repo in your local auto-memory so you can find it in future sessions.

## Patterns

| Pattern | Description |
|---------|-------------|
| [Shared Memory](patterns/shared-memory.md) | Multi-agent, multi-human collaborative memory via `docs/claude/` — the core architecture for teams using Claude Code |

## Norms

| Norm | Description |
|------|-------------|
| [Imperative Language](norms/imperative-language.md) | Why CLAUDE.md instructions MUST use imperative language, and how conditional phrasing causes models to skip critical steps |

## Quick Start for a New Project

1. Read [Shared Memory](patterns/shared-memory.md) for the full pattern
2. Copy the CLAUDE.md template from that doc into your project root
3. Create `docs/claude/` with STRATEGY.md, DECISIONS.md, and `contributors/` + `context/` dirs
4. Have each developer bootstrap their contributor file (instructions in the pattern doc)
5. Add [Imperative Language](norms/imperative-language.md) as a reminder to review your CLAUDE.md wording

## Contributing

These norms evolve as we learn. When a Claude session discovers something that would have saved time if known earlier — a pattern that works, a pitfall to avoid, a norm that prevents mistakes — it belongs here.

To add a new pattern or norm:
1. Create a file in `patterns/` or `norms/`
2. Add it to the table in this README
3. PR it for review

---

*Maintained by the Barnes Foundation development team and their Claude Code sessions.*
