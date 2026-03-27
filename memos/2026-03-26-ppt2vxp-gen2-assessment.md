---
from: PPT2VXP
to: VXP-2.0
date: 2026-03-26
subject: Gen 2 composed boards — PPT2VXP readiness assessment
---

Reviewed the contract and Gen 2 proposal. Good news: PPT2VXP already extracts most of the data Gen 2 needs.

## What We Already Capture (Firestore)

| Field | Images | Text | Shapes |
|-------|--------|------|--------|
| x, y position | Yes | Yes | Yes (if has text) |
| width, height | Yes | Yes | Yes |
| rotation (degrees) | Yes | Yes | Yes |
| z-order (orderIndex) | Yes | Yes | Yes |
| text runs (bold, italic, fontSize, fontFamily) | — | Yes | Yes |
| title/bullet detection | — | Yes | — |

All stored per-item with `ElementPosition { x, y, width, height, rotation? }` in Firestore at `decks/{deckId}/slides/{slideId}/items/{itemId}`.

## Gaps

1. **Text color** — Parsed from PPTX but stripped during `sanitize()`. Fix: ~10 lines to preserve the color field through to Firestore. Smallest gap.
2. **Empty shapes** — Shapes without text (decorative rectangles, lines, arrows) are skipped entirely. Would need parser changes to capture fill, stroke, and shape type.
3. **Text alignment and line spacing** — Not captured. Would need `pptxtojson` output inspection to determine availability.
4. **Shape geometry** — Shape type (rect, circle, arrow), fill color, stroke, shadow — none captured. The `pptxtojson` library is text/image-focused; shape styling would require either extending it or dropping to raw OOXML parsing.

## Assessment

**For positioned images + text overlays with basic formatting**: Ready now. The position and dimension data is accurate and complete. A proof-of-concept composed board could be built today using existing Firestore data.

**For full slide fidelity with shape styling**: Moderate effort. Shape styling (fill, stroke, type) is the biggest gap and requires parser work.

## Recommended Mapping to Gen 2 Fields

```
ImageItemDoc.position.x       → BoardImage.x
ImageItemDoc.position.y       → BoardImage.y
ImageItemDoc.position.width   → BoardImage.width
ImageItemDoc.position.height  → BoardImage.height
ImageItemDoc.position.rotation → (needs Gen 2 field?)
TextItemDoc.*                  → SVG overlay layer (text elements)
slide dimensions (from pptxtojson) → Gen 2 canvas dimensions
```

## Open Question

Does Gen 2 need a `rotation` field on BoardImage? PPT2VXP captures it. Also: what coordinate system? PPTX uses EMUs (English Metric Units, 914400 per inch). We currently store the raw `pptxtojson` output values (which are already converted to a normalized coordinate space). Need to align on units.
