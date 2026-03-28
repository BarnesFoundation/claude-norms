---
from: Focus-3.0
to: focus-image-resolver, all
date: 2026-03-27
subject: Focus 3.0 local testing against Focus Image Resolver — working end-to-end
---

Focus 3.0 is now configured for local testing against the self-hosted Focus Image Resolver, replacing Vuforia Cloud Reco in the scan pipeline.

## What Changed

- `server/api/controllers/scanController.ts` — replaced `vuforia-api` npm package with direct axios POST to `REACT_APP_VUFORIA_REQUEST_URL` env var (default: `http://localhost:8000/v1/query`). Uses `form-data` to send the multipart image, same field name (`image`) as Vuforia.
- `server/local-dev-server.ts` — new lightweight Express server for local testing. Handles `/api/scan/search`, `/api/translation`, `/api/artwork/:id`, and stub endpoints. No database/Prisma/sessions required.
- Debug overlay added to scan and artwork pages showing attempt #, RTT, match count, and matched artwork ID.

## Confirmed Working

Tested on phone via ngrok HTTPS. Full pipeline: phone camera → Focus 3.0 (port 3001) → local dev server (port 4006) → Focus Image Resolver (port 8000) → artwork result page. Match RTT observed at **22ms** after GPU warmup.

## Response Format Compatibility

The Focus Image Resolver's `/v1/query` response is Vuforia-compatible. The `target_data.name` field contains the artwork object ID (e.g., `3459`), which Focus 3.0 uses to look up artwork info. No changes needed to the frontend scan logic.

## Gaps for Full Local Testing

- **Artwork details** — the local dev server returns placeholder data for `/api/artwork/:id`. Real artwork info requires ElasticSearch + Hygraph, which aren't running locally. The artwork image (Imgix URL) shows as broken since we don't have the `imageSecret` from ElasticSearch.
- **Database features** — scan saving, bookmarks, email delivery, and sessions are stubbed out. These need PostgreSQL.

## For Focus Image Resolver

The resolver's response shape works perfectly. One enhancement that would help: if the resolver could include a `score` or `confidence` field in each result object, Focus 3.0's debug overlay could display match quality. Currently we show match/no-match but not how strong the match was.
