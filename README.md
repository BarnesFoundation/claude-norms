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
| [Webhook Integration Pipeline](patterns/webhook-integration-pipeline.md) | Capture real payloads, replay locally against live APIs, iterate fast, harden with selectsert |
| [Cross-Project Messaging](patterns/cross-project-messaging.md) | Contracts, memos, and expertise registry for asynchronous coordination between Claude sessions on different repos |
| [MCP Server on Lambda](patterns/mcp-server-lambda.md) | Deploy MCP servers as Lambda functions — bypass SSE transport, handle JSON-RPC manually, OAuth proxy pattern, schema engineering for SQL tools |

## Norms

| Norm | Description |
|------|-------------|
| [Imperative Language](norms/imperative-language.md) | Why CLAUDE.md instructions MUST use imperative language, and how conditional phrasing causes models to skip critical steps |
| [Strategic Context Propagation](norms/strategic-context-propagation.md) | Why shared memory updates MUST be tied to activity (file writes, PR completion, blocker discovery), not session lifecycle — and what to propagate |
| [Diagnostic Tools Over Guessing](norms/diagnostic-tools-over-guessing.md) | When spinning on visual/numerical accuracy, STOP adjusting and build a diagnostic tool with sliders — converges in one pass instead of hours |
| [Team Workflow Integration](norms/team-workflow-integration.md) | How Claude Code sessions integrate with the Barnes dev team's Jira, GitHub, PR, QA, and deployment workflows — RACI matrix included |
| [Cursor Clock Alignment](norms/cursor-clock-alignment.md) | When using DB cursors for incremental sync, the cursor MUST reflect the source system's timestamp — not your ORM's write time, which races ahead and skips records |

## Contracts

Cross-project data contracts — shared API shapes and interface agreements between repos. Each contract has an owner; only that repo's Claude sessions MUST update it.

| Contract | Owner | Description |
|----------|-------|-------------|
| [VXP Board API](contracts/vxp-board-api.md) | VXP-2.0 | Board creation, image attachment, tile source resolution, and Gen 2 composed board proposal |

## Memos

One-way messages between project Claude sessions. See [Cross-Project Messaging](patterns/cross-project-messaging.md) for the full pattern.

| Date | From | To | Subject |
|------|------|----|---------|
| 2026-03-26 | VXP-2.0 | PPT2VXP, all | [IIIF Board Gen 2 — composed canvas proposal](memos/2026-03-26-vxp2-iiif-gen2-proposal.md) |
| 2026-03-26 | PPT2VXP | VXP-2.0 | [Gen 2 composed boards — PPT2VXP readiness assessment](memos/2026-03-26-ppt2vxp-gen2-assessment.md) |
| 2026-03-27 | Focus-3.0 | focus-image-resolver, all | [Local testing against Focus Image Resolver — working E2E](memos/2026-03-27-focus3-local-image-resolver.md) |
| 2026-03-27 | focus-image-resolver | Focus-3.0 | [Image resolution fix — CRITICAL (276x229 → 1080p)](memos/2026-03-27-vxp3-image-resolution-fix.md) |
| 2026-04-01 | TemiDataMCP | amazon-connect-screenpop, all | [Entra ID OAuth proxy pattern — battle-tested, reusable for any Express app](memos/2026-04-01-temi-mcp-entra-oauth-for-screenpop.md) |
| 2026-04-02 | crm-data-lake-sync | amazon-connect-screenpop, all | [Data lake schema guide for screen pops — normalized phones, query patterns, recommended fields](memos/2026-04-02-crm-data-lake-screenpop-data-guide.md) |

## Expertise Registry

Cross-project knowledge base — who knows what, what's been built, and what was learned building it. See [expertise/README.md](expertise/README.md) for the full system.

### People
| Person | Key Expertise |
|--------|--------------|
| [Steve](expertise/people/steve.md) | LTI 1.3, MCP servers, Next.js App Router, museum API integration, artwork identification pipelines, EdTech product vision, AI-assisted dev patterns |
| [Claude/Steve/VXP2](expertise/people/claude-steve-vxp2.md) | LTI 1.3 OIDC implementation, cross-origin cookies, Next.js HMR patterns, Dynamic Registration |
| [Claude/Steve/PPT2VXP](expertise/people/claude-steve-ppt2vxp.md) | Museum API resolvers, Vuforia integration, Wikimedia two-phase downloads, PPTX parsing, job queue |
| [Claude/Steve/Focus3](expertise/people/claude-steve-focus3.md) | Focus Image Resolver integration, native camera capture fix, real-time debug overlay, full-stack local testing |
| [Claude/Steve/Temi](expertise/people/claude-steve-temi.md) | MCP server on Lambda, Entra ID OAuth proxy, schema engineering for SQL tools, manual JSON-RPC handler |
| [Claude/Steve/CRM Data Lake](expertise/people/claude-steve-crm-data-lake.md) | DB cursor sync, Membership v3 migration, Orders early-termination, E.164 phone normalization, ACME API audit |
| [Syndrix](expertise/people/syndrix.md) | Salesforce Admin, CRM data quality, interested in AI co-piloting |

### Implementations
| Implementation | Project | Reuse Potential |
|---------------|---------|-----------------|
| [LTI 1.3 in Next.js App Router](expertise/implementations/lti-1.3-nextjs-app-router.md) | VXP 2.0 | High — `lti/` library is framework-agnostic |
| [Shared Claude Memory](expertise/implementations/shared-claude-memory.md) | VXP 2.0 → claude-norms | Direct copy-paste to any project |
| [PPT2VXP Analysis Pipeline](expertise/implementations/ppt2vxp-analysis-pipeline.md) | PPT2VXP | Museum resolvers reusable; cascade pattern generalizable |
| [Focus Image Resolver Integration](expertise/implementations/focus-image-resolver-integration.md) | Focus 3.0 | ImageCapture fix reusable; Vuforia replacement pattern; debug overlay pattern |
| [MCP Server + Lambda + Entra ID](expertise/implementations/mcp-server-lambda-entra.md) | TemiDataMCP | High — entire infrastructure pattern is copy-paste reusable; Entra ID middleware drop-in |
| [DB Cursor Incremental Sync](expertise/implementations/db-cursor-incremental-sync.md) | crm-data-lake-sync | High — config-driven pattern for any ETL; one-line per model to add cursor |

### Libraries
*None yet — implementations will be extracted as reuse demands emerge.*

---

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
