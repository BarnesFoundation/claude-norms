# Claude (Steve's sessions) — PPT2VXP

What this Claude has learned and implemented through hands-on work on PPT2VXP. This is accumulated implementation knowledge, not general training data.

## Implemented

### Museum API Resolver Suite — Complete
- **Files**: `src/lib/verification/` (met.ts, aic.ts, nga.ts, rijksmuseum.ts, getty.ts, loc.ts, wikiart.ts, wikimedia.ts)
- **What was built**:
  - 8 museum API resolvers with distinct integration patterns per institution
  - Met: REST API + customprints subdomain for high-res
  - AIC: IIIF image API with dynamic manifest parsing
  - NGA: GitHub CSV open data (~101K rows) — no public API, Cloudflare blocks scraping
  - Rijksmuseum: GitHub ZIP CSV (~240K with images, Google CDN `=s0` URLs) — API deprecated Jan 2026
  - Getty: Schema.org JSON-LD + IIIF v2, CC0 filtering in code (API breaks with `open_content` param)
  - LoC: JSON files API, largest JPEG by file size (skip TIFF — browsers can't display), `/pictures/resource/` fallback
  - WikiArt: Scrape `ng-init thumbnailSizesModel` for image URLs
  - Wikimedia Commons: Batch API with OAuth 1.0a, pick largest by pixel area, two-phase download (CDN thumb → background full-res)
- **Key decisions**: Each resolver is self-contained. `titlesMatch()` uses 40% coverage substring + stop-word-filtered token overlap. `artistsMatch()` handles "First Last" vs "Last, First" and strips attribution prefixes.

### Metadata Search Upgrade — Complete
- **Files**: `src/lib/verification/metadata-search.ts`
- **What**: After Wikimedia resolves with title+artist, searches Met+AIC+Getty APIs by name for museum-quality metadata upgrade
- **Key learning**: Must validate both title AND artist to prevent wrong-painting matches (e.g., Avercamp vs van den Berghe)

### Vuforia Cloud Recognition Integration — Complete
- **Files**: `src/lib/vuforia/`
- **What**: HMAC-SHA1 signed multipart POST to Cloud Reco API
- **Key gotcha**: String-to-sign uses bare `multipart/form-data` (no boundary), but HTTP header includes boundary. Query limit is 2.1MB (512KB is for VWS target management only). Client Access Keys required, not Server.

### Client-Side PPTX Parsing — Complete
- **Files**: `src/lib/pptx/parser.ts`, `src/lib/upload/client-pipeline.ts`
- **What**: Browser parses PPTX via `pptxtojson` + JSZip, canvas-launders images to clean JPEGs, uploads to Firebase Storage with SHA-256 dedup via SubtleCrypto
- **Key learning**: `Buffer.from` doesn't exist in browser — replaced with `atob()`. Canvas laundering fixes malformed PNGs that crash server-side decoders.

### Wikimedia Two-Phase Download System — Complete
- **Files**: `src/lib/jobs/wikimedia-upgrade.ts`
- **What**: Phase 1 uses CDN `/thumb/` URLs during analysis (fast, no 429s). Phase 2 background worker downloads full-res after job completes with 5s gaps and circuit breaker.
- **Key learning**: Separate circuit breakers needed for API vs CDN. `checkOAuthError()` auto-disables on `mwoauth-*` errors. Wikimedia Enterprise doesn't cover Commons.

### Job Queue with Stall Recovery — Complete
- **Files**: `src/lib/jobs/processor.ts`, queue logic in Firestore
- **What**: Firestore transaction lock (`queue/lock` doc) for atomic claim-or-enqueue. Stall detection (90s timeout), auto-resume (5 retries), fire-and-forget with manual fallback.
- **Key learning**: `serverTimestamp()` stores unresolved sentinel on client `setDoc` — use `Timestamp.now()`. Lambda kills `void processJob()` — must `await` with `maxDuration = 300`.

### Pure JS Image Processing — Complete
- **What**: `pngjs` + `jpeg-js` for resize/encode. Downsample 2x for >1500px, 3x for >3000px.
- **Key learning**: `sharp` (native C) deadlocks on malformed PPTX PNGs. `opencv-js` (WASM) blocks event loop. Both were uninstalled entirely.

### QC Workflow — Complete
- **What**: Approve/Swap/Reject with candidate carousel sorted by pixel area. Cross-job cache only reuses `qcStatus === 'approved'` items. Pulsing cyan dot for pending Wikimedia upgrades.

### Shared Claude Memory System — Complete
- **Files**: `CLAUDE.md`, `docs/claude/STRATEGY.md`, `DECISIONS.md`, `contributors/steve.md`, `context/*.md`
- **What**: Applied three-layer pattern from claude-norms to PPT2VXP as team onboards

## Knows But Hasn't Implemented Yet

### VXP Gen 2 Composed Boards (assessed 2026-03-26)
- PPT2VXP already stores image/text positions, dimensions, rotation, and z-order in Firestore
- Gen 2 mapping: `ElementPosition` → `BoardImage.{x, y, width, height}`, text → SVG overlay
- Gaps: text color stripped during sanitize, empty shapes skipped, shape styling (fill/stroke/type) not captured
- Reply memo posted to claude-norms with full assessment
- See `memos/2026-03-26-ppt2vxp-gen2-assessment.md`

## Lessons Learned on This Project
1. **Firestore `serverTimestamp()` is client-side poison** — stores a sentinel, not a value. Use `Timestamp.now()` for any field read client-side.
2. **Response bodies MUST be drained on error paths** — undrained fetch responses leak connections and eventually hang the pipeline.
3. **Lambda fire-and-forget dies** — `void processJob()` gets killed when the Lambda returns the HTTP response. Must `await` with `maxDuration`.
4. **LoC TIFF is a browser dead-end** — browsers can't display TIFF. Pick the largest JPEG from the files array instead.
5. **Wikimedia hotlinking policy** — CDN thumbnail URLs (`upload.wikimedia.org/thumb/...`) may show error pages when loaded from external sites. Download to own storage instead.
6. **Canvas laundering fixes truncated PNGs** — PPTX images with incomplete data crash server decoders but browsers render them fine. Canvas → JPEG in browser produces clean files.
7. **Turbopack starves the event loop** — use webpack for dev server when running async pipelines during development.
8. **Never `taskkill //IM node.exe`** on Windows — kills Firebase emulators (Java process survives, but Hub/Storage die). Kill specific PIDs only.
