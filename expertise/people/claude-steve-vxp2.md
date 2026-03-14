# Claude (Steve's sessions) — VXP 2.0

What this Claude has learned and implemented through hands-on work on VXP 2.0. This is accumulated implementation knowledge, not general training data.

## Implemented

### LTI 1.3 OIDC Authentication (Phase 1) — Complete
- **Files**: `client/src/app/api/lti/login/route.ts`, `callback/route.ts`, `.well-known/jwks.json/route.ts`, `register/route.ts`
- **Library**: `client/src/lti/` (types, keys, claims, role-mapping, nonce-store, platform-store)
- **What was built**:
  - Full 3-step OIDC flow: login initiation → auth redirect → JWT validation + session creation
  - RSA key pair generation and JWKS endpoint using `jose` v6
  - LTI claim extraction with typed interfaces for all IMS Global claim URIs
  - Role mapping: LTI role URIs → VXP meeting roles (Learner→0/Audience, Instructor→2/Moderator)
  - Nonce store with expiry for replay attack prevention
  - Platform store for multi-LMS registration
  - `globalThis` pattern for in-memory stores surviving Next.js HMR
  - Session cookie with full LTI claims + debug payload
- **Tested against**: Live Moodle instance (learn.barnesfoundation.org) via ngrok static domain
- **Key decisions**: Built from spec with `jose` (not ltijs), routes in App Router API handlers

### LTI Dynamic Registration — Complete
- **File**: `client/src/app/api/lti/register/route.ts`
- **What was built**:
  - Fetches platform's OpenID configuration from `openid_configuration` URL
  - POSTs tool registration JSON to platform's `registration_endpoint` with Bearer token
  - Auto-registers platform in local store with all returned config (client_id, deployment_id, endpoints)
  - Returns HTML page with success UI + `postMessage({subject:'org.imsglobal.lti.close'})` for iframe close
  - Root domain redirect (`page.tsx`) detects `?openid_configuration=` and forwards to register endpoint
- **Tested against**: Moodle Dynamic Registration iframe flow

### Cross-Origin Cookie Handling for LTI — Solved
- **Problem**: LTI launches come from LMS domain (cross-origin). Cookies need `SameSite=none; Secure` for cross-origin delivery. This applies to both the OIDC state cookie and the session cookie.
- **Gotcha 1**: `SameSite=none` silently requires `Secure=true` — browsers drop the cookie without error if Secure is false
- **Gotcha 2**: `secure: process.env.NODE_ENV === 'production'` evaluates to `false` in dev, breaking the flow. Hardcoded `true` works because ngrok provides HTTPS.
- **Gotcha 3**: POST-preserving redirects (307) after callback cause Next.js to interpret the request as a Server Action. Use 303 instead.

### Next.js HMR State Survival — Solved
- **Problem**: In-memory Maps (nonce store, platform store) get wiped on hot module reload during development
- **Solution**: Attach to `globalThis` with type-safe accessor pattern:
  ```typescript
  const globalForX = globalThis as typeof globalThis & { __myStore?: Map<string, Thing> };
  const store = globalForX.__myStore ?? (globalForX.__myStore = new Map());
  ```

### LTI Debug Landing Page — Complete
- **File**: `client/src/app/lti/launch/page.tsx`
- **What**: Server component that reads `lti_session` cookie and displays all LTI claims in organized sections (user, roles, context, resource link, Advantage service URLs, raw JWT payload)
- **Note**: Uses inline styles, no Mantine dependency — works even without backend running

### Shared Claude Memory System — Complete
- **Files**: `CLAUDE.md`, `docs/claude/STRATEGY.md`, `DECISIONS.md`, `contributors/steve.md`, `context/*.md`
- **What**: Three-layer collaborative memory architecture. Imperative language throughout.
- **Extracted to**: BarnesFoundation/claude-norms repo as reusable pattern

## Knows But Hasn't Implemented Yet

### LTI Advantage Services (Phase 2)
- NRPS: OAuth 2.0 client_credentials → GET roster from `context_memberships_url`
- AGS: POST scores (0.0–1.0) to gradebook via `lineitems` URL
- Deep Linking: Return content items for LMS to embed
- Designed but blocked on team input (database schema, user model, grading philosophy)

### VXP Existing Architecture
- Google OAuth flow via `auth/` service on port 3111, `LXP` session cookie
- GraphQL backend on port 4002 (separate repo)
- Redux Toolkit for meeting state, SWR for data fetching
- Amazon Chime SDK for conferencing
- Mantine 8 component library

### Product Vision
- Three-tier LTI distribution model (Open/Institutional/Premium)
- Evergreen library concept for K-12 art education
- Deweyan engagement-based grading: `score = min(1.0, active_time / content_duration)`
- LTI chaining architecture (College Board Canvas → Barnes Moodle → VXP)
- Session analytics (NoSQL/log-based, aggregatable per-object engagement)

## Lessons Learned on This Project
1. **Always use 303 redirects after POST** in Next.js App Router — 307 preserves POST method and triggers Server Actions validation
2. **Cross-origin cookies require both** `SameSite=none` AND `Secure=true` — one without the other silently fails
3. **`globalThis` is essential** for in-memory stores during Next.js development — HMR wipes module-level variables
4. **Cookie size limit (~4KB)** means debug payloads in session cookies will truncate — remove before production
5. **ngrok free tier now includes one static domain** — no need for paid plan just for a persistent URL
6. **Imperative language in CLAUDE.md** — "should" gets skipped, "MUST" gets followed
