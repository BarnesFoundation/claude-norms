# Claude/Steve/Focus-3.0 — Expertise Profile

## Session Context
- **Project**: Focus 3.0 (Barnes Foundation visitor app)
- **Date**: 2026-03-27 through 2026-03-28
- **Human**: Steve
- **Model**: Claude Opus 4.6

## Implemented

### Focus Image Resolver Integration
- Replaced `vuforia-api` npm package with direct axios POST to configurable URL
- Vuforia-compatible response format — zero frontend changes to scan logic
- Handles SPEX, __REF_, and plain numeric target names through existing `processIdentifiedItem`

### Native Camera Resolution Capture
- Diagnosed 276x229 capture bug (CSS display size vs native camera resolution)
- Implemented `ImageCapture.grabFrame()` for fresh native-resolution frames
- Fixed stale frame issue on mobile (setInterval vs requestAnimationFrame timing)

### Real-time Debug Overlay
- Per-query: attempt #, RTT, image size, score, crop level, inliers, capture resolution
- TSTTM (Total Scan Time To Match) and TSTTF (Total Scan Time To Failure)
- Persists across scan → artwork page navigation via router state
- Error boundary on artwork page for crash visibility

### Local Dev Server
- Lightweight Express server (`server/local-dev-server.ts`) for DB-free testing
- Hardcoded translations from seed SQL, stub endpoints for bookmarks/sessions
- Real ElasticSearch + Imgix integration for artwork data
- Later superseded by full server against prod RDS

### Full Stack Local Testing
- Connected to prod RDS, prod ElasticSearch, prod Imgix
- Only image matching is local (Focus Image Resolver on port 8000)
- ngrok HTTPS tunnel for phone camera access

## Knows But Hasn't Implemented
- Focus 3.0 AdminJS admin panel configuration
- Prisma migration workflow
- GraphCMS/Hygraph content model for special exhibitions
- Story delivery and bookmark email jobs (SendGrid)
- Serverless Framework deployment to AWS Lambda
