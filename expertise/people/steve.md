# Steve

## Role
Technical lead / product visionary at Barnes Foundation. Bridges education mission with engineering execution.

## Proven Expertise

### LTI 1.3 / EdTech Standards
- **Project**: VXP 2.0
- **What**: Built complete LTI 1.3 OIDC authentication from spec using `jose` library in Next.js 15 App Router. Dynamic Registration against live Moodle instance. Designed three-tier distribution model (Open/Institutional/Premium) for LMS integration.
- **Depth**: Spec-level understanding of OIDC flow, JWT/JWKS validation, LTI Advantage services (NRPS, AGS, Deep Linking), Dynamic Registration protocol, cross-origin cookie handling for iframe embedding.

### MCP (Model Context Protocol) Servers
- **Project**: TemiDataMCP
- **What**: Built an MCP server for Temi visitor tracking database. Lambda + PostgreSQL + Entra ID OAuth. Deployed as serverless function.
- **Depth**: Full MCP server implementation, schema engineering, Lambda deployment patterns, Entra ID (Azure AD) OAuth integration.

### Next.js App Router Architecture
- **Project**: VXP 2.0
- **What**: Working in production Next.js 15 App Router codebase with API routes, middleware, server components, and complex auth flows (Google OAuth + LTI sessions coexisting).
- **Depth**: Route handlers, middleware patterns, cookie management across auth providers, `globalThis` pattern for HMR state survival.

### AI-Assisted Development Patterns
n### Amazon Connect Agent Desktop (2026-04)
- Custom CCP + screenpop agent desktop deployed at ccp.barnesfoundation.org
- CRM data lake integration for real-time caller lookup (~0.5s)
- Entra ID SSO with security group gating
- Cross-project coordination via claude-norms memos (data lake team shipped indexes and ticketType same-day)
- AWS deployment: S3 + CloudFront + Lambda (VPC) + HTTP API Gateway
- **Projects**: VXP 2.0, TemiDataMCP, claude-norms
- **What**: Designed and implemented the shared Claude memory pattern (three-layer architecture for multi-agent collaboration). Discovered imperative language requirement for CLAUDE.md. Built cross-project norms repository.
- **Depth**: Multi-developer Claude Code coordination, CLAUDE.md design, contributor file system, strategic memory via git.

### Product Vision / EdTech
- **Projects**: VXP 2.0, CyberCharters
- **What**: Designed VXP's LTI tier model for art education distribution. Conceptualized evergreen library, Deweyan engagement-based grading, session analytics architecture. Advocacy work for PA cyber charter school funding.
- **Depth**: K-12 and higher ed LMS ecosystems, LTI as distribution channel, engagement metrics, curriculum standards alignment.

### Artwork Identification Pipeline / Museum API Integration
- **Project**: PPT2VXP
- **What**: Built multi-strategy artwork identification pipeline: Vuforia Cloud Recognition → Google Vision → 8 museum API resolvers (Met, AIC, NGA, Rijksmuseum, Getty, LoC, WikiArt, Wikimedia Commons) → Claude AI fallback. Client-side PPTX parsing with canvas laundering. Two-phase Wikimedia downloads. Job queue with stall recovery on AWS Lambda.
- **Depth**: Museum open-access API landscape (REST, IIIF, CSV open data, Schema.org, Wikidata). Vuforia HMAC-SHA1 auth. Wikimedia OAuth 1.0a + circuit breakers. Firestore transaction-based queue. AWS Amplify Lambda constraints. Cost optimization ($0.24 → $0.06/image).

### CRM Data Lake / ETL Optimization
- **Project**: crm-data-lake-sync
- **What**: Optimized ETL sync from 26 min to 55 sec (96%). Database-as-cursor with snapshot. Tiered conditional sync (Tier 1 every run, Tier 2 on orphaned FKs, Tier 3 daily safety net). Membership v2→v3 + co-processed cards in single API call. Orders early-termination. Phone E.164 normalization. ticketType/addOn/eventLocation backfills. OPTIDP indexes for screenpop. Cross-project memo coordination with amazon-connect-screenpop.
- **Depth**: Prisma ORM in production PostgreSQL, ACME Ticketing API ecosystem (v2/v3, reports, pagination quirks, custom fields evolution), cursor clock alignment, tiered sync architecture, surgical backfill patterns (day-by-day API, month-by-month reports), Hygraph→Prisma schema gap analysis, EC2 deployment constraints.

### Reverse-Engineered Portal Integration / Email Security (2026-05)
- **Project**: Quarantine Canary
- **What**: Built end-to-end AppRiver-to-Teams notification system in a single 30-hour pair-coding stretch. Reverse-engineered AppRiver's customer-portal search API (no public API at the SKU level). Silent OIDC re-auth in plain Node fetch using a long-lived `KnownDevice` cookie to skip SMS, eliminating Playwright and headless-browser approaches. DynamoDB conditional-put dedup with TTL. Bot Framework proactive messaging via Graph-resolved chat ids (sidesteps the encrypted-pairwise-id requirement of `POST /v3/conversations`). Chat-as-management UX: users add and manage their per-user watchlist by typing `add foo@example.com` directly into the bot's chat with strict-syntax commands; no separate web portal. Three-tier threat-aware notification cards branching on `groupClass` (low / dangerous / malware). Tenant-wide Teams app distribution with lazy-firing welcome cards.
- **Depth**: AppRiver / OpenText Secure Cloud portal API (reverse-engineered: silent 2,500-record cap, undocumented `viewMailUrl` admin/user URL portability, `inboundspam` vs `inboundmalware` classification model). Microsoft Bot Framework v3 REST without the SDK (no `@azure/msal-node`, no `botframework-connector` — pure native fetch). Microsoft Graph application permissions for Teams (`User.Read.All`, `TeamsAppInstallation.ReadForUser.All`; no `ChatMessage.Send` because it does not exist as application permission). JWT validation against Microsoft's Bot Framework JWKS via `jose`. Teams app manifest authoring (`bots`, `commandLists`, tenant-wide install propagation timing). SAM/CloudFormation with KMS-encrypted Lambda env vars and DynamoDB at rest, customer-managed key, plus the policy-statement gotcha for CloudWatch Logs. Discovered "natural consequences > paternalistic validation" as a reusable design philosophy; promoted to `norms/natural-consequences-over-paternalistic-validation.md`.

## Working Style
- Thinks in systems and product vision, then drills into technical specifics
- Values documenting *why* alongside *what*
- Prefers branch-per-concern with PR review gates
- Rapid prototyper — "get it working, clean it up later" balanced by team's stability-first culture
