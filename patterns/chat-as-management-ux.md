# Chat as Management UX

When a user-facing bot needs settings or per-user state, use the bot's own 1:1 chat as the management interface. Do NOT build a separate web portal.

Battle-tested on **Quarantine Canary** (BarnesFoundation/quarantine-canary), where users add and manage their email-quarantine watchlists by typing `add foo@example.com` directly into the bot's chat instead of through a web app. Eliminated an entire codebase that the original architecture had assumed.

## The Problem

A typical "notification bot with user-tunable settings" requires two surfaces:

1. The bot itself (Teams, Slack, Discord) sending notifications based on the user's saved preferences.
2. A web app where the user logs in, sees their preferences, edits them, and saves.

That second surface is a real chunk of work: hosting, sign-in, identity propagation, deploy pipeline, mobile responsiveness, accessibility, copy, support. Most teams build it because "of course you have a settings page." Then they discover that:

- Most users don't even know the settings page exists.
- The notifications themselves are the natural place users would think about settings ("ugh, I keep getting alerts about X, can I turn that off?").
- The chat platform already has the user's identity, mobile UX, and notification surface for free.

## The Solution

Use the bot's existing 1:1 chat as the management interface. Strict-syntax commands (verb plus argument) cover the common operations. Replies confirm actions and surface errors. The user's identity is implicit because they are already authenticated to Teams/Slack/etc; the bot reads `from.aadObjectId` (or equivalent) on each inbound activity and maps to the same user-key the bot uses elsewhere.

Four-command surface covers most state-management needs:

| Verb | Purpose |
| ---- | ------- |
| `add <thing>` | Add an entry to the user's state |
| `remove <thing>` | Remove an entry |
| `list` | Show current state |
| `help` | Show the verb list |

Welcome card on first install (when the platform fires `installationUpdate.add` or equivalent) tells the user what they can do. Reply UX defaults to plain text for action confirmations and a rendered card only for `list` (where a structured layout earns its keep).

## Architecture

```
   Teams chat (compose box visible)
              │
              ▼
   Bot Framework (or Slack Events API)
              │  HTTP POST inbound activity
              ▼
   Bot endpoint Lambda (Function URL)
              │
              ├──── Validate inbound JWT against platform JWKS
              ├──── Parse command verb + arg
              ├──── Resolve user identity (from.aadObjectId → mail localpart)
              ├──── DynamoDB write/read on user's state
              └──── Send reply via Bot Framework REST
                       (plain text, or Adaptive Card for `list`)
```

Three load-bearing pieces in code:

1. **JWT validation.** The bot endpoint is a publicly reachable HTTPS URL. Without rigorous JWT validation against the platform's JWKS, anyone who can reach the URL can manipulate any user's state by forging activities. Use a maintained library (`jose` for Microsoft, equivalent for Slack) and pin issuer + audience.
2. **Strict-syntax parser.** Lowercase the verb, trim, split on whitespace, take first token as verb and the rest as argument. No NLP, no fuzzy matching. Determinism beats cleverness for management UIs; users learn the four verbs in seconds.
3. **Identity resolution and caching.** Map `from.aadObjectId` to a stable user-key (mail local-part, employee ID, whatever the rest of the system uses). Cache the mapping per Lambda execution context to avoid a Graph round-trip per command.

## When to Use This Pattern

- The bot already sends the user notifications, so they have it in their chat list.
- The state being managed is per-user and small (watchlists, preferences, subscriptions, mute rules).
- A two-page wizard would be enough to capture the management need; you do NOT need rich forms, drag-and-drop, multi-step workflows, or anything that would feel cramped in a chat.
- The platform supports a real messaging endpoint. Send-only bots cannot do this; the chat surface must be bidirectional.

## When NOT to Use This Pattern

- The state is shared across multiple users, requiring concurrent editing, RBAC, or per-field permissions. (Use a real app.)
- The user needs to upload, browse, search, sort, or visualize complex data. (Chat is a poor surface for that.)
- The actions are irreversible and destructive enough to warrant explicit confirmation modals or two-factor steps. (Chat can confirm, but type-confirmations are weak compared to a UI dialog.)
- The platform's bot quotas would price out high-volume command traffic. (Most quotas comfortably handle "manage your settings" volumes; check for your specific platform.)

## Welcome and Discovery

Tenant-wide bot install puts the bot in every user's chat list silently. Do NOT assume users will discover the management commands on their own. On the first inbound activity from each user (`installationUpdate.add` for Microsoft Teams, `app_home_opened` for Slack, etc.), send a one-time welcome card with:

- One sentence explaining what the bot does.
- The four-command list with one example each.
- That is it. Resist the urge to add a tour, a video, or a "set up your first entry" wizard. The user will discover the rest by typing.

## Edge Cases the Pattern Handled in Production

- **Platform input munging.** Microsoft Teams wraps email-shaped strings in angle brackets when rendering them as hyperlinks before sending the activity to the bot, so `add foo@example.com` arrives as `add <foo@example.com>`. Defensive normalization in the parser handled it. (See ADR-013 in the Quarantine Canary repo.)
- **Lazy welcome propagation.** `installationUpdate.add` fires when each user opens their Teams client for the first time after the tenant install, NOT at install time. Welcome cards trickled in over hours, not minutes. Plan accordingly.
- **Empty arguments.** `help` and `list` take no argument; `add` and `remove` require one. Reject malformed input with a friendly message that includes a usage example.

## The Anti-Pattern

Building a "Settings" page that lives at `app.example.com/settings`, requires its own OIDC sign-in, has to be linked from every notification, and ends up being used by 3% of the recipient population while the rest just live with whatever defaults you shipped.

## Reference Implementation

[BarnesFoundation/quarantine-canary](https://github.com/BarnesFoundation/quarantine-canary). Specifically:

- `src/bot-endpoint.ts` — Lambda handler, JWT validation, command dispatcher, reply paths.
- `src/watchlist.ts` — DynamoDB read/write plus the parser for the user's typed input.
- `docs/claude/DECISIONS.md` ADR-012 — the decision rationale and supersession of the original web-portal plan.
- `docs/claude/DECISIONS.md` ADR-013 — schema extension for domain-shaped entries (extends the four-verb model with one more distinguishable input shape).
- `docs/claude/DECISIONS.md` ADR-014 — production-driven UX refinements (parser hardening for platform input munging, response card tiering).

The bot endpoint is a single Lambda with a Function URL, JWT-validated, ~250 lines of TypeScript. The whole "management UI" came in under the line count of a typical settings page's CSS.
