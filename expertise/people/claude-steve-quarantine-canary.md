# Claude/Steve — Quarantine Canary

**Date**: 2026-05-04 through 2026-05-05 (initial MVP + production refinements)
**Project**: Quarantine Canary (BarnesFoundation/quarantine-canary)

A scheduled AWS Lambda + Microsoft Teams bot that watches the AppRiver / OpenText Secure Cloud quarantine for messages from senders on a per-user watchlist and notifies the recipient via 1:1 Teams chat. Watchlist management lives entirely inside the chat with the bot. Built end-to-end in a single 30-hour pair-coding stretch and shipped to production with two real human users on it within hours.

## Implemented

- **Silent OIDC re-auth in plain Node fetch** against AppRiver's reverse-engineered customer portal. Two-tier cookie model: long-lived `KnownDevice` device-trust cookie (manual annual ritual) plus auto-refreshed session cookie via the OIDC redirect chain. No Playwright, no headless browser, no MFA automation. ~150 lines including the cookie jar and redirect-following fetch wrapper. See `patterns/cookie-extraction-and-rotation.md`.
- **Binary-descent truncation guard** for AppRiver's silent 2,500-record cap. Recursive window halving; deduplication by `messageGuid` to handle inclusive-boundary overlap. Unit-tested against synthetic peaks before any live traffic.
- **DynamoDB conditional-put dedup** keyed on `messageGuid` with TTL-based expiry. Idempotent processing across overlapping poll windows and Lambda retries; cost of a Lambda crash mid-notification is one missed notification, never duplicates.
- **Bot Framework proactive messaging from Lambda** without standing up a real bot endpoint for v1. Graph resolves the user's installation and personal chat with the bot; Bot Framework REST posts the activity directly to that chat. Sidesteps the encrypted-pairwise-id requirement of `POST /v3/conversations`, which only works after receiving an inbound activity.
- **Bot endpoint Lambda for chat-managed watchlist UX.** JWT validation against Microsoft's Bot Framework JWKS via `jose`. Strict-syntax command parser (`add`, `remove`, `list`, `help`). DynamoDB writes from chat messages. Replies via the same Bot Framework path used for outbound notifications. See `patterns/chat-as-management-ux.md`.
- **Three-tier threat classification on notification cards.** Maps `groupClass` + `quarantineType` to (low, dangerous, malware) and varies card title, color, description, and button label per tier. Releasability and danger are separate axes; PHISHING is technically releasable but treated as dangerous in the card framing.
- **Tenant-wide Teams app distribution.** Manifest v1.1.0, single-tenant, `bots` capability, `commandLists` exposed in the chat header. Admin uploads to org catalog and assigns to "all licensed users." Welcome card on `installationUpdate.add` (lazy-firing as users open Teams) drives organic discovery.
- **End-to-end production deploy in one pass.** SAM template with Lambda + EventBridge + DynamoDB tables + KMS key + SNS alarm topic + four CloudWatch alarms. `sam deploy --guided` produced the full stack; env vars pushed via `aws lambda update-function-configuration` from the local `.env`. Verified with synchronous test invokes plus a deliberately spoofed email from outside `stevebrady.com`'s SPF record.

## Key Lessons

1. **`ChatMessage.Send` is not an application permission in Microsoft Graph.** Application-only sending of arbitrary Teams chat messages via raw Graph is not supported in 2026. The viable paths are Bot Framework proactive messaging or activity feed notifications. ADR-009 had to be superseded by ADR-011 mid-flight.
2. **`POST /v3/conversations` requires encrypted "pairwise ids."** The bot can only learn these from inbound activities. If the bot is send-only with a stub messaging endpoint (no inbound), use Graph `/users/{upn}/teamwork/installedApps/{installationId}/chat` to resolve the personal chat, then POST activities directly to that chat id. The encrypted pairwise id is never needed.
3. **Tenant-wide install fires `installationUpdate.add` lazily.** Microsoft activates the install when each user actually opens their Teams client, not at admin-install time. Welcome cards trickle in over hours, not minutes. Plan onboarding accordingly.
4. **Teams wraps email-shaped strings in angle brackets** when rendering them as hyperlinks before sending the activity to the bot. Defensive parser normalization (strip `<`, `>`, `mailto:`) is required to handle real user input. Found by the second user typing `add ehaider@pewtrusts.org` and getting `<ehaider@pewtrusts.org>` stored in DynamoDB. Fixed in production within 30 minutes via the new parser plus a one-shot backfill script that DM'd affected users from the bot.
5. **`jose@6` is ESM-only and breaks CommonJS Lambda bundles.** Pin to `jose@^5` (last CJS-compatible major) or migrate the whole project to ESM. Caught by Lambda init crash with `ERR_REQUIRE_ESM` on the bot endpoint, ~3 hours into production traffic.
6. **CloudWatch Logs needs explicit KMS key policy permission** when the log group is configured with a customer-managed key. The Lambda service principal alone is not sufficient. Add `logs.<region>.amazonaws.com` to the key policy with `kms:Encrypt*` etc. First deploy attempt rolled back at the log-group-creation step until this was fixed.
7. **Reverse-engineered admin URLs may be "portable" to user-side endpoints.** AppRiver's quarantine search API returns `/ngp/email-security/admin/view-message?emIdfs=...` URLs. Stripping the `admin/` segment lands non-admin users on the user-side message view; the encrypted `emIdfs` token is portable across both surfaces. Empirically verified, captured in ADR-014.
8. **Releasability and danger are different axes.** AppRiver's `quarantineType=inboundspam` covers both genuinely-spam (releasable, low-risk) and PHISHING (releasable but the dominant breach vector for credential theft and BEC). Quarantine notifications should branch on `groupClass`, not `quarantineType`, when framing the user's response. Spam gets "review and decide"; phishing gets "exercise caution, do not enter passwords"; malware gets "cannot be released, contact IT Support."
9. **Natural consequences beat paternalistic validation.** Steve added `@gmail.com` as a stress-test entry expecting to be flooded; the prediction failed because his specific inbox doesn't draw much gmail-spam. The architecture handled either outcome gracefully without any "are you sure?" prompts. Captured as a reusable norm in `norms/natural-consequences-over-paternalistic-validation.md`.

## Key Files

- `src/auth.ts` — Silent OIDC re-auth orchestrator. Multi-step login form handling, token-relay form handling, identity-host validation. Reusable for any cookie-authenticated portal with OIDC.
- `src/cookie-jar.ts` — Cookie jar with parent-domain matching, Set-Cookie ingestion, redirect-following fetch wrapper. Reusable.
- `src/appriver.ts` — Per-call wrapper around the search endpoint with retry-once-on-401 + module-scoped session cookie cache.
- `src/teams.ts` — Microsoft Graph + Bot Framework client. `getGraphToken`, `getBotToken`, `resolveUserByUpn`, `resolveBotChat`, `sendActivity`, `sendQuarantineNotification`. Reusable.
- `src/bot-endpoint.ts` — Bot endpoint Lambda. JWT validation, command parser, watchlist writes, replies. Reusable for any chat-as-management-UX bot.
- `src/watchlist.ts` — Watchlist storage with the `@`-prefix domain extension. Defensive input parser handling Teams' angle-bracket wrapping.
- `src/dedup.ts` — DynamoDB conditional-put dedup with TTL. Generalizable to any "process exactly once" Lambda.
- `infra/template.yaml` — Full SAM template with KMS key, two DynamoDB tables, two Lambda functions, EventBridge schedule, SNS alarm topic, four alarms, Function URL.
- `infra/teams-app/manifest.json` — Single-tenant Teams app manifest with `bots`, `commandLists`, and the canary mascot icon.
- `docs/claude/STRATEGY.md` — Phase plan and posture (zero-trust at the gateway, non-negotiable).
- `docs/claude/DECISIONS.md` — 14 ADRs documenting the architectural progression, including the ADR-009 → ADR-011 supersession when Microsoft Graph turned out not to support `ChatMessage.Send` as application permission.
- `docs/claude/context/appriver-auth.md` — The auth model for the specific portal, reverse-engineered.
- `docs/claude/context/microsoft-graph-setup.md` — One-time human ritual for Azure AD app registration, Azure Bot resource creation, Teams app package upload.
