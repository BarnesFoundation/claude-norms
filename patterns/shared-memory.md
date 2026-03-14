# Shared Claude Memory Pattern

A battle-tested architecture for multi-agent, multi-human collaborative memory in codebases. Developed on VXP 2.0 (Barnes Foundation), refined through real-world use with multiple developers and their Claude Code sessions working on the same repo.

## The Problem

When multiple developers use Claude Code on the same codebase, each session starts with a blank slate. Without shared memory:
- Contradictory strategies emerge across sessions
- Architectural decisions get relitigated every conversation
- Context is lost between sessions and between developers
- One developer's Claude undoes another developer's Claude's work
- Strategic thinking lives only in chat history that expires

## The Solution: Three-Layer Architecture

### Layer 1: Personal Memory (`~/.claude/projects/*/memory/`)
- Per-user, per-machine
- Workflow preferences, cross-project patterns, personal context
- NOT shared with team — this is Claude Code's built-in auto-memory
- Can be synced across a single user's machines (e.g., OneDrive)

### Layer 2: Project Root `CLAUDE.md`
- Auto-loaded by Claude Code at the start of every session — no action required
- Contains project conventions, build commands, coding standards
- **Critical**: Points Claude to Layer 3 with imperative instructions
- This is the "boot sequence" — if instructions here are soft or conditional, models (especially smaller ones) will skip them

### Layer 3: `docs/claude/` (Shared, version-controlled)
- Lives in the repo, travels with the code via git
- Strategic thinking, decisions, per-contributor context
- Git merge conflicts surface strategic divergence the same way they surface code conflicts
- PRs on strategy changes enable human oversight

## Why Three Layers (Not One or Two)

**Layer 1 alone** = every developer's Claude is an island. No shared context.

**Layer 2 alone** = project conventions exist but no strategic memory. Decisions get relitigated.

**Layer 3 alone** = rich shared memory but Claude doesn't automatically load it. Someone has to remember to tell Claude to read it every session — they won't.

The key insight: **Layer 2 (CLAUDE.md) is the bridge.** It auto-loads and contains imperative instructions that force Claude to read Layer 3. Without this bridge, Layer 3 is dead documentation.

## Directory Structure

```
project-root/
  CLAUDE.md                        # Layer 2: auto-loaded boot instructions
  docs/claude/
    STRATEGY.md                    # Shared roadmap (team consensus)
    DECISIONS.md                   # ADR-style decision log
    contributors/
      <developer-name>.md          # Per-developer context and thinking
    context/
      <feature-or-topic>.md        # Feature-specific deep context
```

## File Purposes and Formats

### STRATEGY.md
**What**: The team's agreed-upon roadmap and architectural direction.
**Format**: Current initiatives with phases, open questions, future considerations.
**Rule**: This represents team consensus. Any Claude session that is about to contradict this file MUST stop and flag it to the user before proceeding.

### DECISIONS.md
**What**: Architectural Decision Records (ADRs) — numbered, with context, choice, and reasoning.
**Format**: `## ADR-NNN: Title` → Date, Status, Context, Decision, Consequences.
**Rule**: Every non-trivial technical choice made during a session MUST be recorded here before the session ends. "Non-trivial" = any choice where a reasonable developer might have gone a different way.

### contributors/\<name\>.md
**What**: Each developer's current focus, active thinking, open questions, and working patterns.
**Who writes it**: The developer's own Claude session creates and maintains it. Other Claudes read it but NEVER modify it.
**Why**: When Developer B's Claude reads Developer A's contributor file, it can spot conflicts ("A is building auth this way, but B is about to build it a different way") and surface them before they become code conflicts.

**How to bootstrap**: After the shared memory system is merged, each developer pastes this into their first Claude Code session:
```
Please read docs/claude/STRATEGY.md and docs/claude/contributors/<existing-example>.md
as an example, then create my contributor file at docs/claude/contributors/<my-name>.md.
Ask me about my current focus, what I'm working on, and my role on the project
so you can fill it in properly. Keep it updated as we work together.
```

### context/\<topic\>.md
**What**: Deep technical context for specific features or initiatives — research, trade-offs, evolving understanding.
**Format**: Living documents, not specs. They capture the *thinking* behind decisions, not just the decisions themselves.
**Examples**: `lti-1.3-integration.md`, `auth-architecture.md`, `database-migration-plan.md`

## CLAUDE.md Template

This is the critical piece. The language MUST be imperative — see [Imperative Language](../norms/imperative-language.md) for why.

```markdown
# <Project Name> — Claude Code Project Guide

## What is <Project>?
<One paragraph description>

## Tech Stack
<Bullet list of key technologies>

## Development
<Code block with build/run commands>

## Code Conventions
<Bullet list — use imperative voice: "Use X", not "Prefer X" or "Consider X">

---

## Shared Claude Memory

This project uses a **multi-agent, multi-human collaborative memory system** in `docs/claude/`.

### Required Reading

Every Claude session MUST read these files before starting substantive work. Do NOT skip this step. Do NOT assume you know the current strategy without reading these files.

1. **`docs/claude/STRATEGY.md`** — The shared roadmap. This represents team consensus. If you are about to do something that contradicts this file, STOP and flag it to the user before proceeding.

2. **`docs/claude/DECISIONS.md`** — Architectural decision log. Check this BEFORE making any architectural choice — the decision may already be made.

3. **`docs/claude/contributors/<name>.md`** — Read your contributor's file to pick up where they left off. Read other contributors' files to detect conflicts or alignment opportunities. NEVER modify another contributor's file.

4. **`docs/claude/context/<topic>.md`** — Read any context files relevant to your current task. These contain research, trade-offs, and evolving understanding that is NOT in the code.

### Rules

- **Read before writing.** ALWAYS check STRATEGY.md and the relevant contributor file before proposing changes. Do NOT propose changes based on assumptions.
- **Flag conflicts.** If the current contributor's direction contradicts another contributor's file or the shared strategy, raise it explicitly. Do NOT silently proceed.
- **Update after decisions.** When a meaningful choice is made during a session, update the relevant files (contributor file, DECISIONS.md, or context docs) before the session ends. Do NOT leave decisions unrecorded.
- **Never overwrite others.** Contributor files belong to their contributor. Only update your own contributor's file. For STRATEGY.md and DECISIONS.md, propose changes to the user — do NOT silently rewrite shared files.
- **Keep it concise.** These files load into context windows. Every unnecessary sentence costs tokens and dilutes attention. Brevity is a feature.

### Why This Exists

When multiple developers use AI assistants on the same codebase, each session starts with a blank slate. Without shared memory, contradictory strategies emerge, decisions get relitigated, and context is lost. This system makes strategic thinking **version-controlled, reviewable, and conflict-detectable** — git surfaces divergence the same way it surfaces code conflicts.
```

## Implementation Checklist

For any new project adopting this pattern:

1. **Create `docs/claude/` directory** with STRATEGY.md, DECISIONS.md, and `contributors/` and `context/` subdirectories
2. **Write STRATEGY.md** with current initiatives, phases, and open questions
3. **Seed DECISIONS.md** with any existing architectural decisions (even retroactively)
4. **Create the first contributor file** — have your Claude session interview you and create `contributors/<your-name>.md`
5. **Add the Shared Claude Memory section to CLAUDE.md** using the template above — imperative language only
6. **Commit on a dedicated branch** and PR for team review — this is infrastructure, not a feature
7. **Add PR comments** with onboarding instructions for other developers (the bootstrap prompt above)
8. **After merge, each developer bootstraps** their contributor file via the prompt above

## Anti-Patterns

### Conditional language in CLAUDE.md
- BAD: "Every Claude session should read these files before starting substantive work"
- GOOD: "Every Claude session MUST read these files before starting substantive work. Do NOT skip this step."
- WHY: Smaller models interpret "should" as "optional." Larger models may deprioritize it under context pressure. See [Imperative Language](../norms/imperative-language.md).

### Putting strategy in CLAUDE.md instead of docs/claude/
- BAD: Writing the full roadmap directly in CLAUDE.md
- GOOD: CLAUDE.md points to docs/claude/STRATEGY.md with imperative "read this" instructions
- WHY: CLAUDE.md is for boot instructions. Strategy changes frequently and needs its own reviewable file.

### Skipping the contributor file system
- BAD: Just having STRATEGY.md and DECISIONS.md
- GOOD: Every active developer has a contributor file
- WHY: Contributor files prevent cross-developer conflicts. Strategy tells you *where* the project is going. Contributor files tell you *who is doing what right now*.

### Not recording decisions in the moment
- BAD: Making a decision in a session and moving on
- GOOD: Recording it in DECISIONS.md immediately
- WHY: The next session (yours or someone else's) will relitigate the same decision without the record.

## Origin and Evolution

- **Created**: 2026-03-12 on VXP 2.0 (Barnes Foundation)
- **First tested**: Multi-developer LTI 1.3 integration with 3 humans + 3 Claude sessions
- **Key refinement**: Conditional → imperative language (learned from MCP Chat Claude project)
- **PR model**: Shared memory infrastructure goes on its own branch/PR, separate from feature work
