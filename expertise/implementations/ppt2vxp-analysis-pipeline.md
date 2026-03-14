# PPT2VXP Analysis Pipeline

## Summary
Multi-strategy artwork identification pipeline that extracts images from PowerPoint presentations and cascades through Vuforia Cloud Recognition, Google Vision reverse image search, 8 museum API resolvers, and Claude AI to find the highest-resolution version of each artwork.

## Project
PPT2VXP (BarnesFoundation/PPT2VXP)

## Key Files
- `src/lib/jobs/processor.ts` — Main pipeline orchestrator, short-circuit cascade
- `src/lib/verification/` — 8 museum API resolvers (met, aic, nga, rijksmuseum, getty, loc, wikiart, wikimedia)
- `src/lib/verification/metadata-search.ts` — Title+artist search for museum metadata upgrade
- `src/lib/vuforia/` — Vuforia Cloud Recognition with HMAC-SHA1 auth
- `src/lib/google/` — Google Vision API integration
- `src/lib/claude/` — Anthropic Claude fallback analyzer
- `src/lib/jobs/wikimedia-upgrade.ts` — Two-phase Wikimedia download (CDN thumb → background full-res)
- `src/lib/pptx/parser.ts` — Client-side PPTX parsing
- `src/lib/upload/client-pipeline.ts` — Browser image processing + Firebase Storage upload

## Approach
Short-circuit cascade minimizes API costs:
1. Cache hit (cross-job SHA-256 dedup, free)
2. Vuforia + Vision in parallel (~1-2s each)
3. Museum resolvers on Vision results (structured metadata + high-res URLs)
4. Vision-only fast path for trusted sources
5. Claude AI fallback (expensive but comprehensive)

Cost dropped from $0.24/image (Claude-only) to ~$0.06/image (full pipeline + two-phase Wikimedia).

## What Worked
- **Parallel Vuforia + Vision** cuts 1-2s off every image
- **Museum resolver cascade** avoids Claude for ~80% of images
- **Two-phase Wikimedia** keeps job time fast (41s for 17 images) while full-res arrives in background
- **Canvas laundering** in browser fixes malformed PPTX PNGs that crash server decoders
- **Cross-job hash dedup** means repeated artwork across decks is free after first analysis

## What Didn't / Gotchas
- **Lambda cold starts** add 3-5s and fire-and-forget patterns die when Lambda returns
- **Wikimedia hotlinking** — CDN thumbnail URLs blocked from external sites, need own storage
- **LoC TIFF** — browsers can't display, must select JPEG alternative from files array
- **sharp/opencv deadlocks** — native image processing is incompatible with this workload, pure JS required
- **Getty API fragility** — adding `open_content=1&images=1` params breaks the search, returns null

## Reuse Potential
- Individual museum resolvers are self-contained and could serve any artwork identification project
- The short-circuit cascade pattern applies to any multi-source lookup with varying cost/speed
- Wikimedia OAuth + two-phase download pattern is reusable for any Commons-heavy project
- `titlesMatch()` / `artistsMatch()` fuzzy matching is reusable for artwork metadata comparison

## People
- Steve (architecture, product, museum domain expertise)
- Claude/Steve/PPT2VXP (implementation)

## Date
2026-02 through 2026-03 (ongoing)
