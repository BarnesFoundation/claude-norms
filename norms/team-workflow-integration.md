# Team Workflow Integration — Human + Claude Code

How Claude Code sessions integrate with the Barnes Foundation development team's existing workflow, Jira, and GitHub practices.

## Morning Updates

**Human norm**: Async morning update — what you did yesterday, what you plan to do today, blockers.

**Claude integration**: Claude sessions cannot attend standups or post async updates. Instead:
- At session start, Claude MUST read the relevant contributor file in `docs/claude/` and any new memos in `claude-norms/memos/`
- At session end (or after significant work), Claude MUST update the contributor file with: what was done, what's next, any blockers discovered
- Steve will relay Claude session progress in Tuesday/Friday check-ins until real-time meeting participation is solved

**Future**: Meeting recordings from Tuesday/Friday check-ins will be parsed by a Claude session to extract action items, update Jira cards, and sync with codebases.

## Jira Integration

**Human norm**: Cards track all work. Branch names use `<ticket-number>-<brief-description>`. Jira-GitHub integration links PRs to cards automatically.

**Claude integration**:
- When working on a Jira card, Claude MUST use the ticket number in branch names: `PPT2VXP-42-add-crop-tool`
- Claude can read Jira boards via the Atlassian CLI (`acli`) at `C:\atlassian\acli.exe`
- Claude SHOULD check for assigned cards before starting work on a new feature
- When creating branches without a Jira card (exploratory/urgent work), note the missing card and create one after if the work is kept

**Story points reference**:
| Points | Effort | Claude equivalent |
|--------|--------|-------------------|
| 0 | < 1 hour | Single quick fix |
| 1 | Few hours – ½ day | One focused feature |
| 2 | ½ day – 1 day | Feature + tests |
| 3 | 1-2 days | Multi-file feature |
| 5 | 3-5 days | Architecture change |
| 8+ | 1 week+ | Break into multiple cards |

## GitHub — Branches and Commits

**Human norm**: Small, frequent commits. Branch per feature. Descriptive messages.

**Claude norm (already established)**: ALWAYS branch for changes, never commit directly to main. Branch → commit → merge → push.

**Integration rule**: Claude sessions MUST follow the same branch naming convention as human developers. If a Jira ticket exists, use `<ticket-number>-<brief-description>`. If no ticket, use descriptive names like `fix-wikimedia-thumbnails` or `iiif-manifest-generation`.

## Pull Requests

**Human norm**: Small PRs (≤10 files), one card per PR, description with decisions and testing steps, link to Jira card, screenshots for UI changes.

**Claude integration**:
- For work that will be reviewed by human developers, Claude MUST create PRs instead of merging directly to main
- PR description MUST follow the team template: Description, Testing, Jira link, Screenshots
- Claude SHOULD keep PRs small and focused — break large features into multiple PRs
- For rapid iteration with Steve (CTO), direct merges to main are acceptable per Steve's preference, but the work SHOULD be consolidated into reviewable PRs before the next team check-in

**Transition plan**: As the team gets comfortable with Claude's output quality, the PR review process will tighten. Initially, Steve reviews Claude PRs. Eventually, Leigh and other developers review them like any other PR.

## Code Reviews

**Human norm**: Review by end of next business day. Productive feedback with solutions. Test locally before approving.

**Claude integration**:
- Claude CAN review human PRs if asked — read the diff, check for bugs, suggest improvements
- Claude SHOULD flag when its own changes might affect another developer's in-progress work (check contributor files)
- Claude MUST NOT approve its own PRs or bypass review processes

## QA

**Human norm**: QA checklist before deployment. ACs met. No regressions.

**Claude integration**:
- Claude SHOULD run `tsc --noEmit` and `npm run build` before committing
- Claude SHOULD describe testing steps in commit messages or PRs
- For UI changes, Claude SHOULD note what to visually verify
- Human QA remains required for production deployment

## Deployment

**Human norm**: LD responsible. Deploy when galleries are closed. Instructions in README.

**Claude integration**:
- Claude MUST NOT deploy to production without explicit human approval
- Claude CAN deploy to development/staging environments when asked
- Claude MUST check deployment instructions in the repo before attempting deployment
- Each repo's `CLAUDE.md` SHOULD include deployment commands and constraints

## Bug Reporting

**Human norm**: Jira ticket with steps to reproduce, expected outcome, actual outcome.

**Claude integration**:
- When Claude discovers a bug during development, it SHOULD create a Jira card (via `acli`) with the standard template
- Claude MUST include exact steps, code references, and screenshots/logs when available
- Claude SHOULD tag bugs with the appropriate component and priority

## Incident Response

**Human norm**: Not on-call but expected to respond when able.

**Claude integration**: Claude sessions are not persistent — they can't monitor or respond to incidents. However:
- Claude CAN help diagnose incidents when given logs or error messages
- Diagnostic endpoints (`/api/diagnose-env`, `/api/diagnose-pipeline`) are available for Claude to check system health
- Claude SHOULD document known failure modes in `docs/claude/context/` files so the next session can help faster

## RACI for Claude Sessions

| Activity | CTO (Steve) | Lead Dev (Leigh) | Claude |
|----------|-------------|-------------------|--------|
| Feature development | C, I | A, C | R |
| Architecture decisions | A, C | C | R (proposes), records in DECISIONS.md |
| Code review | I | A, R | C (can review if asked) |
| Deployment to prod | A | R | I (never deploys without approval) |
| Jira card management | I | A, C | R (can create/update cards) |
| QA | I | A, C | R (automated checks), C (manual QA by humans) |
| Documentation | C | C | R (shared memory, ADRs, norms) |
| Incident diagnosis | A | R, C | C (when given logs/context) |
