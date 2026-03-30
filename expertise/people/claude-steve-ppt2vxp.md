# Claude (Steve's sessions) — PPT2VXP

What this Claude has learned and implemented through hands-on work on PPT2VXP. This is accumulated implementation knowledge, not general training data.

## Implemented

### Museum API Resolver Suite — Complete
- **Files**: `src/lib/verification/` (met.ts, aic.ts, nga.ts, rijksmuseum.ts, getty.ts, loc.ts, wikiart.ts, wikimedia.ts)
- **What**: 8 museum API resolvers with distinct integration patterns per institution. `titlesMatch()` uses 40% coverage substring + stop-word-filtered token overlap. `artistsMatch()` handles "First Last" vs "Last, First".

### Vuforia Cloud Recognition Integration — Complete
- **Files**: `src/lib/vuforia/`
- **What**: HMAC-SHA1 signed multipart POST to Cloud Reco API. Client Access Keys required, not Server. Query limit is 2.1MB.

### Client-Side PPTX Parsing — Complete
- **Files**: `src/lib/pptx/parser.ts`, `src/lib/upload/client-pipeline.ts`
- **What**: Browser parses PPTX via `pptxtojson` + JSZip, canvas-launders images, uploads to Firebase Storage with SHA-256 dedup.

### OOXML Direct Text Parser — Complete (2026-03-29)
- **Files**: `src/lib/pptx/ooxml.ts`
- **What**: Replaces pptxtojson for ALL text extraction. Reads raw OOXML from PPTX ZIP for precise formatting: margins (bodyPr lIns/tIns/rIns/bIns), line spacing (spcPct), paragraph spacing (spcAft/spcBef), tab stops, font inheritance (theme → master otherStyle → run), alignment, anchor. Preserves `\t` characters that pptxtojson strips.
- **Key discovery**: Font size inheritance chain: runs inherit from endParaRPr → pPr defRPr → layout placeholder → master otherStyle (18pt for non-placeholder text boxes).

### IIIF Presentation 3.0 Manifest Generation — Complete (2026-03-29)
- **Files**: `src/lib/iiif/manifest.ts`, `types.ts`, `image-service.ts`, `svg-overlay.ts`, `text-rasterizer.ts`
- **What**: Full manifest builder with dynamic canvas sizing, positioned image annotations, IIIF Image API service detection (AIC, Getty, LoC), text-to-PNG rasterization for universal viewer support. Storage proxy for single-tunnel ngrok.
- **Round-trip proven**: PPTX → parse → analyze → QC → manifest → Mirador with correct positions + text.

### Canvas Preview + Comparison Tool — Complete (2026-03-29)
- **Files**: `src/components/canvas/CanvasPreview.tsx`, `TextOverlay.tsx`, `OoxmlReference.tsx`, compare page
- **What**: CSS-positioned slide layout with 720px minimum (browser font hinting breaks below ~10px). Diagnostic sliders (Y offset, X offset, spacing %) for calibration. Reference renderer for side-by-side comparison.
- **Key discovery**: PT_TO_PX (1.333) cancels algebraically in all scaling calculations — not a real conversion issue. CSS renders ~4% taller than PowerPoint (0.96 correction). Below 720px, Chromium font hinting changes behavior entirely.

### Canvas Editor — Complete (2026-03-30)
- **Files**: `src/components/canvas/CanvasEditor.tsx`, `DraggableItem.tsx`, `CropOverlay.tsx`, `CanvasToolbar.tsx`, `TextFormatToolbar.tsx`, `AlignmentGuides.tsx`
- **What**: Lightweight drag-and-drop editor (~600 lines total, no heavy framework). react-rnd for drag/resize, native contentEditable for text editing, caretRangeFromPoint for cursor placement. Features:
  - Drag, resize, z-order for all items
  - Image crop with aspect-ratio-locked resize
  - Inline text editing (second-click to enter, Escape to exit)
  - Format toolbar: bold, italic, font size, color, bullet/numbered lists
  - Alignment guides + snap system (slide midpoints + item edges/centers)
  - Add text block, add image from URL, delete items

### QC Workflow Enhancements — Complete (2026-03-30)
- **What**: "Use Original" action (skip high-res), retry failed downloads button, carousel initializes to selected candidate, Wikimedia thumb URL rewrite in downloader.

### Wikimedia Download Fix — Complete (2026-03-30)
- **What**: `/thumb/` URLs get 429'd by Wikimedia hotlinking policy. Downloader now rewrites to direct `/commons/` URLs before download. Display-time rewrite also applied in ImageItem.tsx.

## Knows But Hasn't Implemented Yet

### VXP Gen 2 Composed Boards
- Architecture designed, contract reviewed, coordinate system decided (0-1 normalized)
- Waiting on VXP-2.0 to ship the Gen 2 Board API

### Per-Word Text Formatting
- Current formatting applies to all runs. Need to split runs at selection boundaries for bold/italic on individual words.

### Image Derivatives (Thumb/Mid)
- Need ~300px thumbs and ~1024px mids for faster canvas preview. Currently loads full high-res images.

## Lessons Learned on This Project
1. **Firestore `serverTimestamp()` is client-side poison** — stores a sentinel, not a value. Use `Timestamp.now()`.
2. **Response bodies MUST be drained on error paths** — undrained fetch responses leak connections.
3. **Lambda fire-and-forget dies** — `void processJob()` gets killed. Must `await` with `maxDuration`.
4. **Wikimedia hotlinking policy** — `/thumb/` URLs get 429'd from external sites. Rewrite to direct `/commons/` URLs.
5. **Canvas laundering fixes truncated PNGs** — PPTX images with incomplete data crash server decoders.
6. **Turbopack starves the event loop** — use webpack for dev server.
7. **Browser font hinting at <720px** — CSS renders text at measurably different positions below ~10px font size. Potential Chromium bug.
8. **PT_TO_PX cancels out** — the 1pt=1.333px conversion factor cancels in all scaling math. Not a real issue.
9. **0.96 spacing correction** — CSS line boxes render ~4% taller than PowerPoint. Empirically calibrated via comparison tool.
10. **When spinning, build a diagnostic tool** — the comparison tool with sliders saved hours of guess-and-check.
11. **`textContent` strips line breaks, `innerText` preserves them** — critical for contentEditable extraction.
12. **`pointerEvents: 'none'` blocks contentEditable and crop** — must disable parent drag to enable child interaction.

## Session Stats (2026-03-29/30)
- ~3,000+ lines of new code across 40+ commits
- 13 ADRs documented
- OOXML parser, IIIF manifest generator, canvas preview, comparison tool, text rasterizer, canvas editor with crop/text/alignment
- Full IIIF round-trip proven in Mirador via ngrok
