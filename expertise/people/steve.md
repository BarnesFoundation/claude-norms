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

## Working Style
- Thinks in systems and product vision, then drills into technical specifics
- Values documenting *why* alongside *what*
- Prefers branch-per-concern with PR review gates
- Rapid prototyper — "get it working, clean it up later" balanced by team's stability-first culture

### Self-Hosted Image Recognition / Computer Vision
- **Project**: Focus-Image-Resolver
- **What**: Replacing PTC Vuforia Cloud (~$48k/year) with self-hosted ORB + RANSAC image matching on AWS. Designed two-stage matching pipeline (geometric correspondence + quality validation). Led migration of 5,465 targets (3,124 unique artworks) from Vuforia. Defined naming conventions mapping multi-image targets to single objects.
- **Depth**: OpenCV feature detection algorithm tradeoffs (ORB vs. SIFT vs. AKAZE vs. BRISK). Vuforia VWS API internals (HMAC-SHA1 auth, server vs. client credential separation). Production image matching constraints for museum use: rejecting gallery photos, enforcing quality gates, optimizing for head-on scanning. Reference image sizing strategy (~1024x768 for better fingerprinting).
