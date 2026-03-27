# Cross-Project Messaging for Claude Sessions

## Problem

When Claude sessions on different repos need to coordinate (e.g., PPT2VXP needs to know VXP's data contract), there is no communication channel. Each session starts with a blank slate and no awareness of decisions made in other repos.

## Solution

Use the `claude-norms` repo as a shared mailbox with three mechanisms: contracts, memos, and the expertise registry.

### Contracts (`contracts/`)

Shared data contracts, API shapes, and interface agreements between repos.

- File format: `contracts/<name>.md`
- Each contract MUST declare its **owner** (the repo responsible for maintaining it) at the top of the file
- Any repo's Claude session can READ contracts
- Only the owning repo's Claude session MUST update a contract — other repos MUST NOT modify contracts they do not own
- Contracts MUST include: target system, current schema/shape, resolution rules, and related repos
- When a contract changes, the owning repo SHOULD also post a memo (see below)

### Memos (`memos/`)

One-way messages from one project's Claude session to others. Asynchronous notifications — no response expected in the same file.

- File format: `memos/<date>-<from-repo>-<subject>.md`
- MUST include YAML frontmatter: `from`, `to`, `date`, `subject`
- Use for:
  - "I changed X which affects your repo"
  - "I need Y from your repo"
  - "FYI this decision was made"
  - "This proposal needs your review"
- Claude sessions MUST check for new memos relevant to their repo at session start
- Memos are **append-only** — NEVER edit or delete another repo's memos
- A receiving repo MAY post a reply memo (new file, not an edit)

### Expertise Registry (`expertise/`)

Already exists. See `expertise/README.md` for the full system.

- `expertise/people/<name>.md` — Human expertise profiles
- `expertise/implementations/<name>.md` — Technical implementations and learnings

## Why This Works

- **Git-native**: All coordination is version-controlled and reviewable
- **Asynchronous**: No real-time coupling between sessions
- **Conflict-detectable**: Git surfaces divergence the same way it surfaces code conflicts
- **Low-overhead**: Claude sessions just read markdown files — no APIs, no databases
- **Auditable**: The full history of cross-project communication lives in git log

## Example Flow

1. VXP-2.0 Claude proposes a new board type
2. VXP-2.0 Claude updates `contracts/vxp-board-api.md` with the proposal (marked as "Proposed")
3. VXP-2.0 Claude posts `memos/2026-03-26-vxp2-iiif-gen2-proposal.md` notifying PPT2VXP
4. Next PPT2VXP session reads the memo, reviews the contract, and posts a reply memo with its assessment
5. Once both sides agree, VXP-2.0 Claude updates the contract status from "Proposed" to "Active"
