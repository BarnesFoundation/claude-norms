# Cookie Extraction and Rotation for Reverse-Engineered Portals

When you need automated access to an internal SaaS portal that has no public API, use a two-tier cookie model: short-lived session cookies refreshed automatically via silent OIDC re-auth in plain Node fetch, paired with a long-lived device-trust cookie rotated by a human ritual once per ~13 months.

Battle-tested on **Quarantine Canary** (BarnesFoundation/quarantine-canary) against the AppRiver / OpenText Secure Cloud customer portal. Six months from "Playwright-driven container" first-draft to "single-Lambda ZIP, ~2-second silent re-auth" working architecture.

## The Problem

Many enterprise SaaS products have a customer portal but no API at most SKU levels. Examples: AppRiver, several legacy ticketing systems, niche EDR consoles, a long tail of vertical-specific tools that the buyer was promised "API access in a future release."

Programmatic access requires authenticating cookies. Cookies expire. The naive approaches all fail:

- **Embed username/password and re-login on every call** — works once, locks you out of MFA, often violates ToS, and leaves credentials in env vars at every step.
- **Headless browser (Playwright/Puppeteer) on every invocation** — heavy. Container-image Lambda. Cold starts. Browser binary to maintain. Bot-detection trips. Easy to break when the portal's CSS changes.
- **Manual cookie refresh whenever auth fails** — operationally painful if the session cookie is hours-lived. Calls IT support every few hours.

## The Solution: Two-Tier Cookie Model

Most modern SaaS portals already use this model internally; you just need to ride it instead of fighting it.

### Tier 1: Short-lived OIDC session cookie

- Lifetime: hours to days. Refreshed on demand.
- Refresh: programmatic, no human input. Submit username + password to the IDP's login form. The IDP issues a fresh session cookie via the standard OIDC redirect chain.

### Tier 2: Long-lived device-trust cookie

- Lifetime: ~13 months (typical for "remember this device" tokens).
- Refresh: manual, human ritual. Once per year, the operator logs in via a real browser, completes the SMS or app-MFA challenge, names the device, and copies the resulting cookie value into Lambda env vars.
- Crucially: the existence of a valid device-trust cookie causes the IDP to **skip the SMS challenge** during silent re-auth. That is the key affordance that makes Tier 1's automation work.

### Why this works

Tier 1's username/password POST would normally trigger SMS, which is unautomatable (and any attempt to automate it crosses a real security/policy line). Tier 2's device-trust cookie is exactly the IDP affordance that lets a returning device skip MFA. Combine them: Tier 2 lifts the SMS gate; Tier 1 then refreshes session cookies all year without human touch.

The annual ritual is 60 seconds of human time, alarmed via CloudWatch + SNS so the operator gets a notification when it is needed.

## Architecture

```
                   ┌───────────────────────────────┐
   Lambda          │   1. GET portal root          │
   (per invoke)    │      (with KnownDevice cookie)│
                   └───────────────┬───────────────┘
                                   │ 302 to IDP authorize
                                   ▼
                   ┌───────────────────────────────┐
                   │   2. IDP serves login form    │
                   │      (HTML, antiforgery       │
                   │       token in hidden field)  │
                   └───────────────┬───────────────┘
                                   │ POST username + password
                                   ▼
                   ┌───────────────────────────────┐
                   │   3. IDP redirects through    │
                   │      callback. SMS gate       │
                   │      bypassed because         │
                   │      KnownDevice present.     │
                   └───────────────┬───────────────┘
                                   │ Auth code → /signin-oidc
                                   ▼
                   ┌───────────────────────────────┐
                   │   4. Portal sets fresh        │
                   │      session cookie. Lambda   │
                   │      now has a working        │
                   │      authenticated session.   │
                   └───────────────────────────────┘
```

Five required runtime values, four of them in env vars and one in code:

- `PORTAL_USERNAME` — the operator's portal account username.
- `PORTAL_PASSWORD` — the operator's password.
- `PORTAL_KNOWN_DEVICE` — the device-trust cookie value extracted via the manual ritual.
- `PORTAL_BASE_URL` — the portal hostname (constant per deployment).
- (in code) The IDP URL pattern. Discoverable from the portal's redirect chain on a test request.

## Implementation Notes

### Plain Node fetch, no SDK

Node 22 has fetch and a streams API capable enough to drive the OIDC dance. You do NOT need:

- Playwright or Puppeteer.
- A persistent browser context.
- The provider's official SDK (often there isn't one anyway).

You DO need:

- A small cookie jar that survives the redirect chain and merges Set-Cookie headers correctly across hops.
- A redirect-following fetch wrapper that respects the cookie jar.
- An HTML form parser. Regex on `<input>` and `<form>` tags is fine for this; the login form is stable.

Roughly 150 lines of TypeScript for the whole module, including JSDoc.

### Failure modes you must handle

- **IDP rejects the device-trust cookie.** Possibly revoked, possibly expired. Detect by: silent re-auth completes but SMS form appears, or final landing URL is the IDP login screen instead of the portal. Alarm and require manual ritual.
- **Username/password rejected.** Detect by: form re-renders with an error banner. Alarm.
- **IDP HTML changes shape.** The form parser sees no recognizable login form. Alarm with `bodyLength` (NEVER log the raw body; antiforgery tokens are session-bound and could leak).
- **Portal silently corrupts the response.** A 200 response with HTML instead of expected JSON means the cookie jar lost auth at some hop. Treat as auth failure; do not parse and continue.

### What NEVER to do

- **NEVER attempt to handle SMS or app-MFA programmatically.** The device-trust cookie exists exactly so you do not have to. Trying to automate the MFA challenge is a security violation, a ToS violation for most providers, and the engineering required (Twilio inbound number? virtual phone? carrier API?) is out of scale for the value.
- **NEVER store cookie values or credentials in the git repo.** Local `.env` (gitignored) for development; AWS Secrets Manager or equivalent for production. The pattern here assumes both stay encrypted at rest with a customer-managed key.
- **NEVER follow redirects blindly past the IDP boundary without validating the final landing host.** A redirect to `account.example.com/login` is the signal that auth has actually expired; following it lands on a login page and silently corrupts your data path.

## When to Use

- Portal has no API at your SKU level, but support has confirmed there is one inside the customer-facing UI that you can reverse-engineer from network traffic.
- The internal API is cookie-authenticated with OIDC and a "remember this device" token.
- The volume of access is bounded (poll cadence in minutes, not seconds).
- The operator is willing to perform a one-minute manual ritual once a year.

## When NOT to Use

- The portal does not use OIDC, or uses a proprietary auth flow that does not have a device-trust equivalent. (Different problem; different pattern needed.)
- The portal explicitly forbids automation in its ToS. (Negotiate with the vendor or use a different product.)
- The volume of access is high enough that you would actually trigger the IDP's rate limits or bot-detection systems on every minute. (Reduce cadence; cache aggressively; coordinate with the vendor.)
- You need write access to the portal's data, not just read. (Reverse-engineering write endpoints is materially riskier than read; revisit.)

## The Annual Refresh Ritual

Schedule a calendar reminder dated 11 months from each successful device-trust cookie mint. The ritual is short:

1. Open Chrome (or any modern browser with DevTools) on a workstation you trust.
2. Navigate to the portal's login URL.
3. Authenticate normally: username, password, SMS code (or app prompt).
4. When prompted, check "remember this browser/device" and name the device with a year stamp (`AutomationDevice-YYYY` or similar). This makes revocation unambiguous.
5. Open DevTools → Application → Cookies. Copy the device-trust cookie value.
6. Paste into the Lambda env var (or Secrets Manager) for the new value.
7. Verify with a smoke test that pulls one record.

CloudWatch alarm on `AuthFailures` notifies the operator when this is needed sooner than the calendar reminder (e.g., the IDP revoked the device for some reason).

## Reference Implementation

[BarnesFoundation/quarantine-canary](https://github.com/BarnesFoundation/quarantine-canary):

- `src/auth.ts` — OIDC dance orchestrator. Multi-step login form handling, token-relay form handling, identity-host validation.
- `src/cookie-jar.ts` — Cookie jar with parent-domain matching, Set-Cookie ingestion, redirect-following fetch wrapper.
- `src/appriver.ts` — Per-call wrapper that uses the silent re-auth to talk to the actual portal endpoint, with a retry-once-on-401 pattern.
- `docs/claude/context/appriver-auth.md` — The auth model for this specific portal (reverse-engineered, kept in repo).
- `docs/claude/DECISIONS.md` ADR-002 / ADR-010 — The architectural progression: from "manual ritual on every cookie expiry" (ADR-002, original) to "silent re-auth with KnownDevice" (ADR-010, current).

The annual ritual itself is documented at `docs/claude/context/appriver-auth.md` "The KnownDevice Mint Ritual." Total wall-clock time: about 60 seconds.
