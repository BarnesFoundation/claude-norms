# LTI 1.3 in Next.js App Router

## Summary
Complete LTI 1.3 OIDC authentication + Dynamic Registration implemented in Next.js 15 App Router using `jose` for all JWT/JWKS operations. No Express, no ltijs, no external LTI library.

## Project
VXP 2.0 (BarnesFoundation/VXP-2.0)

## Branch
`feature/lti-1.3-integration`

## Key Files
- `client/src/app/api/lti/login/route.ts` — OIDC login initiation
- `client/src/app/api/lti/callback/route.ts` — JWT validation + session creation
- `client/src/app/api/lti/register/route.ts` — Dynamic Registration
- `client/src/app/api/lti/.well-known/jwks.json/route.ts` — Public key endpoint
- `client/src/lti/` — Library code (types, keys, claims, roles, stores)

## Approach
- Built from the IMS LTI 1.3 spec rather than using an LTI library
- `jose` v6 handles all cryptographic operations (JWT verify, JWKS fetch, RSA key generation)
- Route handlers are standard Next.js App Router API routes
- In-memory stores (platform, nonce) use `globalThis` for HMR survival — need database backing for production
- Session stored in httpOnly cookie with all LTI claims
- Cross-origin cookies (`SameSite=none; Secure`) required for LMS iframe embedding

## What Worked
- `jose` is excellent — clean API, full OIDC/JWT support, actively maintained
- Next.js route handlers are a natural fit for the OIDC redirect flow
- Dynamic Registration is simpler than it looks — fetch config, POST registration, save response
- Testing against live Moodle via ngrok static domain was smooth

## What Didn't / Gotchas
- **307 vs 303 redirects**: POST-preserving redirects after callback cause Next.js Server Actions error. MUST use 303.
- **Cookie `Secure` flag**: `SameSite=none` silently requires `Secure=true`. `process.env.NODE_ENV === 'production'` is `false` in dev, breaking the flow.
- **HMR wipes state**: Module-level Maps get cleared on hot reload. `globalThis` attachment is mandatory for dev.
- **Cookie size**: Full JWT payload in session cookie can exceed ~4KB browser limit. Strip debug data for production.
- **ltijs won't work**: It's Express-coupled with its own DB layer. Doesn't fit App Router.

## Reuse Potential
High. The `client/src/lti/` library code is framework-agnostic TypeScript. The route handlers are Next.js-specific but the pattern translates to any framework. Could be extracted into a standalone `@barnes/lti-core` package if other projects need LTI.

## People
- Steve (design + testing)
- Claude/Steve/VXP2 (implementation)

## Date
2026-03-12 through 2026-03-13
