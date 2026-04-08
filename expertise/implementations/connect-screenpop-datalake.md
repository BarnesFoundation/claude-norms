# Amazon Connect Screenpop — CRM Data Lake Integration

## Date: 2026-04-01 through 2026-04-07

## Summary
Custom Amazon Connect agent desktop that embeds the standard CCP alongside a caller lookup screenpop panel. Queries a PostgreSQL data lake (synced from ACME ticketing) by phone number to show contact identity, membership status, guest pass balance, recent orders with line items, and upcoming events — all in ~0.5 seconds.

## Project: Amazon Connect Screenpop (BarnesFoundation/Amazon-Connect-Screenpop)

## Key Files
- `frontend/src/components/CallerCard.tsx` — Rich screenpop card with membership, orders, guest passes, venue badges
- `frontend/src/hooks/useLookup.ts` — Two-phase loading hook (contacts fast, details async)
- `frontend/src/hooks/useContact.ts` — Connect Streams contact event hook + simulateCall() dev helper
- `frontend/src/App.tsx` — MSAL provider, group claim gating, CCP always accessible
- `backend/lookup/datalake.ts` — Data lake queries (Phase 1: contacts/addresses, Phase 2: OPTIDP joins)
- `backend/authorizer/index.ts` — Entra JWT validation with dual-issuer (v1+v2)
- `backend/lookup/index.ts` — Lambda handler routing phase1/phase2

## Approach
- **Phone → Contact**: PhoneNumber.normalizedNumber (E.164) → PhoneNumber.acmeCustomerId → Contact.acmeCustomerId
- **Membership**: OPTIDP.acmeCustomerId → OPTIDP.membershipId → Membership.acmeMembershipId
- **Orders**: OPTIDP.acmeCustomerId directly (ACMEOrder.contactId is unpopulated)
- **Events**: OPTIDP.eventInstanceId → EventInstance (+ EventTemplate fallback for eventLocation)
- **Guest Passes**: Issued via "Member Guest Passes" voucher event, redeemed via ACMEOrder.checkInStatus on issuance orders
- **Two-phase loading**: Phase 1 (~200ms) shows contacts immediately, Phase 2 (~300ms with index) loads OPTIDP data

## What Worked
- E.164 normalized phone matching — 97.9% coverage, direct equality check
- OPTIDP as the universal transaction table — one table for orders, memberships, events, guest passes
- esbuild bundling — 39KB lookup Lambda, cold starts are fast
- Cross-project memos via claude-norms — data lake team shipped ticketType field and DB indexes same-day
- Two-phase loading — eliminates perceived latency even before indexes existed

## What Didn't / Gotchas
- Data lake memo's suggested join paths were wrong (acmeOrgContactId::text != TBF-prefixed acmeCustomerId)
- ACMEOrder.contactId is NULL on all 719K rows — completely unusable
- pg v8 forbids concurrent queries on same pooled client
- Express 5 breaking changes (path-to-regexp, CachedMethods nesting)
- Connect identity management is immutable — can't switch to SAML on existing instance
- Windows zombie node processes survive Unix-style pkill

## Reuse Potential
- **Entra JWT Lambda Authorizer**: Drop-in for any API Gateway needing Entra auth (dual-issuer pattern from TemiDataMCP)
- **Two-phase loading pattern**: Applicable to any screenpop where some data is fast and some is slow
- **OPTIDP query patterns**: Reusable for any ACME data lake consumer
- **Guest pass tracking logic**: Specific to Barnes but the issuance/redemption via checkInStatus pattern is generalizable
- **Venue badges (B/C icons)**: Reusable for any Barnes UI showing event locations
