# Strategic Context Propagation

## The Problem

Claude Code sessions accumulate strategic knowledge during work — planned next steps, discovered blockers, architectural intentions, dependency awareness, "we tried X and it failed because Y so the next move is Z." This knowledge lives only in the session's context window. There is no reliable "session exit" event in the VS Code extension — the human stops talking, closes a tab, or the laptop sleeps. Any norm built around a teardown event will fail silently.

When this knowledge isn't promoted to shared memory, the next session (same developer or different) starts without it. The consequences:

- Another session re-attempts a known-failed approach
- Strategic direction drifts because the intent behind recent work was never recorded
- Cross-project dependencies go unannounced (e.g., VXP-2.0 plans a breaking API change that PPT2VXP doesn't know about)
- The human has to re-explain context that a prior session already understood

## The Solution

Tie strategic context propagation to activity, not lifecycle. Shared memory MUST never be more than one commit behind the session's understanding of where the project is headed.

### Mandatory Triggers

Whenever a session does ANY of the following, it MUST also check whether its current working context contains unrecorded future-state knowledge and promote it to the appropriate `docs/claude/` file in the same commit:

1. **Writing to any `docs/claude/` file** — memos, decisions, contributor updates, context docs. If you're already touching shared memory, propagate everything you know.
2. **Completing a meaningful unit of work** — PR, feature, bug fix, refactor. Before considering the work done, record what comes next and why.
3. **Discovering a blocker or changing strategic direction** — if the session learns something that changes the plan, that knowledge is shared-memory-worthy immediately, not "when I get around to it."
4. **Receiving a human directive** — "update shared memory," "sync your context," or any equivalent instruction. Treat this as a full dump of current strategic understanding.

### What to Propagate

- **Planned next steps**: "The next task is X because Y is now complete"
- **Known blockers**: "X is blocked on Y; workaround is Z"
- **Architectural intentions**: "We're going to restructure X to support Y"
- **Failed approaches**: "We tried X; it failed because Y; don't retry without Z"
- **Cross-project dependencies**: "This change means repo Y needs to update Z"
- **Open questions**: "We need to decide X before proceeding with Y"

### Where to Propagate

- **Your contributor file** (`docs/claude/contributors/<name>.md`) — current focus, next steps, active blockers
- **STRATEGY.md** — if the roadmap or priorities shifted
- **DECISIONS.md** — if a non-trivial technical choice was made
- **Context docs** (`docs/claude/context/<topic>.md`) — if deep technical understanding was gained
- **Memos** (`memos/`) — if another project's sessions need to know about a change or dependency
- **Contracts** (`contracts/`) — if an API interface changed (owner repo only)

## The Principle

> Shared memory is never more than one commit behind the session's understanding of where the project is headed.

If a session knows something strategically relevant and hasn't written it down, it is in debt. Activity-based triggers ensure the debt is paid as part of normal work, not deferred to an exit event that may never fire.

## Why Activity-Based, Not Lifecycle-Based

The VS Code extension has no reliable session teardown hook. Building a norm around "before you end the session" is conditional language in disguise — it assumes an event that doesn't reliably occur. See [Imperative Language](imperative-language.md) for why conditional framing causes models to skip critical steps.

Activity-based triggers work because:

- They attach to events that ARE observable (file writes, PR completion, blocker discovery)
- They create a habit of propagation as part of the work, not after it
- They survive the common case where a session just... stops

## Examples

### Good

Session completes an LTI auth fix, writes a PR, and in the same commit updates `contributors/steve.md` with "LTI auth working; next step is Deep Linking integration; blocked on Moodle sandbox access" and adds ADR-012 to DECISIONS.md recording the cookie strategy chosen.

### Bad

Session completes the same fix, writes the PR, and moves on. The next session opens, reads the contributor file (which still says "working on LTI auth"), and either re-investigates the problem or asks the human what happened.

### Good

Session discovers that the Gen 2 composed canvas approach requires a IIIF Presentation API 3.0 feature that the current viewer doesn't support. Immediately updates `context/iiif-gen2.md` with the finding and writes a memo to dependent projects.

### Bad

Session notes the issue in its thinking, works around it locally, and the knowledge evaporates when the session ends. A PPT2VXP session later builds Gen 2 output assuming the feature exists.

## Origin

- **Proposed by**: Steve (via Claude.ai conversation, 2026-03-28)
- **Rationale**: Identified gap where Layer 1 (local session context) knowledge about future state and roadmap was not reliably making it to Layer 3 (shared `docs/claude/`). Root cause: no session exit event in VS Code extension to trigger propagation. Solution: activity-based triggers attached to observable events.
