# Claude (Steve's sessions) — Amazon Connect Screenpop

## Session Context
- Project: Amazon Connect Screenpop (Barnes Foundation)
- Dates: 2026-04-01 through 2026-04-07
- Human: Steve
- Model: Claude Opus 4.6

## Implemented

### Full-Stack Agent Desktop
- React 18 + Vite + TypeScript + Tailwind SPA
- CCP embed via amazon-connect-streams (left panel)
- Screenpop panel with caller lookup (right panel)
- MSAL v5 auth with Entra ID SSO, security group gating
- CCP always accessible regardless of screenpop auth state

### Data Lake Integration
- PostgreSQL queries against CRM data lake (RDS us-east-1)
- Phone lookup via normalized E.164 on PhoneNumber table
- Join path discovery: PhoneNumber.acmeCustomerId → Contact.acmeCustomerId (not acmeOrgContactId::text as memo suggested)
- Orders via OpportunityProductTransactionItemDonationPayment (OPTIDP) — ACMEOrder.contactId is unpopulated across all 719K rows
- Membership via OPTIDP.membershipId → Membership.acmeMembershipId
- Events via OPTIDP.eventInstanceId → EventInstance + EventTemplate fallback for eventLocation
- Two-phase loading: Phase 1 (contacts ~200ms) renders immediately, Phase 2 (OPTIDP queries) loads async

### Guest Pass Tracking
- Issued: ticketType='Member Guest Pass' on 'Member Guest Passes' voucher event
- Redeemed: ACMEOrder.checkInStatus on issuance orders (NOT admission event counts — "plus tickets" are 1 guest pass = 2 admissions)
- Visual: available/issued ratio, segmented bar, per-issuance order breakdown with ACME links

### Rich Order Display
- Line items grouped by event with ticket types (Member Adult, Member Child, etc.)
- Venue badges: Barnes (orange rounded square B), Calder (black circle C), onsite/online icons
- Discounted pricing: strikethrough original, "Free" for member benefits
- ACME backoffice links on orders (via acmeOrderId) and memberships (via acmeMembershipId)

### AWS Deployment
- S3 + CloudFront (OAC) at ccp.barnesfoundation.org
- ACM cert (DNS validated via Route 53)
- Lambda Authorizer (Entra JWT, dual-issuer v1+v2, JWKS caching)
- Lookup Lambda (VPC, same subnets as RDS)
- HTTP API Gateway (not REST — avoids /prod/ prefix)
- esbuild bundling: 74KB authorizer, 39KB lookup

## Knows But Hasn't Implemented
- SFDC integration (jsforce module exists, no credentials configured)
- SAML federation for Connect (eliminates two-login UX — requires new Connect instance)
- ACME dedupe integration (duplicate contact cards ready for "pick winner" UI)
- Direct ACME API fallback for real-time order status

## Lessons Learned
1. ACMEOrder.contactId is NULL on all 719K rows — OPTIDP.acmeCustomerId is the real join key for everything
2. PhoneNumber.acmeCustomerId uses TBF-prefixed strings — join to Contact.acmeCustomerId (text), NOT Contact.acmeOrgContactId (int)
3. Membership status enum has mixed case ('Active', 'active') — use IN clause, not LOWER() (fails on enums)
4. OPTIDP table (1.9M rows) has no index on acmeCustomerId — causes ~1s per query via Parallel Seq Scan. Index request sent to data lake team, they shipped it same day → 0.5s total
5. pg v8 deprecates concurrent queries on same client — run sequentially, not Promise.all
6. Express 5 changed path-to-regexp syntax — `app.all("*")` throws, use named params
7. Zombie node processes on Windows survive pkill — use PowerShell Get-NetTCPConnection + Stop-Process
8. MSAL v5 is current (not v2/v4) — storeAuthStateInCookie removed, SPA platform type required for PKCE
9. Connect approved origins: HTTP vs HTTPS matters, and changes may take minutes to propagate
10. Connect identity management (built-in vs SAML) is immutable after instance creation — no migration path as of 2026-04
11. Guest pass redemption: count via checkInStatus on issuance orders, NOT by counting admission events (plus tickets break that math)
12. eventLocation field: check both EventInstance and EventTemplate (COALESCE fallback)
13. Two-phase loading eliminates perceived latency — agent sees caller name in <500ms even if full data takes longer
