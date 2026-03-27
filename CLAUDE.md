# claude-norms — Claude Code Project Guide

## What is this?
A living reference of cross-project norms and patterns for Claude Code AI-assisted development at the Barnes Foundation. Any project's Claude Code session can read these docs to bootstrap best practices.

## Structure
- `patterns/` — Reusable architectural patterns (e.g., shared memory, cross-project messaging)
- `norms/` — Behavioral norms and lessons learned (e.g., imperative language)
- `expertise/` — Cross-project knowledge registry (people, implementations, libraries)
  - `expertise/people/` — Human expertise profiles AND Claude session expertise (e.g., `claude-steve-vxp2.md`)
  - `expertise/implementations/` — Significant technical implementations with approach, gotchas, and reuse potential
  - `expertise/libraries/` — Reusable packages extracted from implementations
- `contracts/` — Cross-project data contracts and API interface agreements. Each contract has an owner repo; only that repo's Claude sessions MUST update it.
- `memos/` — One-way messages between project Claude sessions. Append-only; NEVER edit or delete another repo's memos.
- `README.md` — Index with quick-start instructions

## Rules for This Repo
- Every pattern, norm, and expertise entry MUST be listed in the README tables
- Use imperative language in all instructional content (see `norms/imperative-language.md`)
- Each document MUST include: the problem, the solution, the reasoning (why), and concrete examples
- Keep documents self-contained — a Claude reading one file gets the full picture without cross-referencing
- Expertise entries MUST cite the specific project, date, and files — no vague claims
- Claude expertise files MUST distinguish "implemented" from "knows but hasn't implemented"
- After completing significant work on any Barnes project, update the relevant expertise files here
- When working on a Barnes project, CHECK `memos/` for messages addressed to your repo at session start
- Only the owning repo's Claude sessions MUST update a contract file — other repos MUST NOT modify contracts they do not own
