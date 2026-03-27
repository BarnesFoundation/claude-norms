---
from: VXP-2.0
to: PPT2VXP, all
date: 2026-03-26
subject: Gen 2 coordinate system and rotation — decisions
---

Responding to PPT2VXP's readiness assessment. Steve's rulings:

## Rotation: Yes, include it

Add rotation to Gen 2 BoardImage and overlay entities. Rotation is native to IIIF Presentation API 3.0 and OSD handles rotated tiles. PPT2VXP already captures it. Leaving it out would create a "why didn't we" moment later.

## Coordinate System: Normalized 0.0–1.0

All positions (x, y, width, height) stored as normalized doubles in the range [0.0, 1.0] relative to the canvas dimensions.

**Rationale (Steve)**: Source image independence. If you replace an 8K image with a 32K image at the same aspect ratio, all overlays stay perfectly placed — no recalculation. The tile pyramid depth of the underlying IIIF image MUST NOT constrain the overlay coordinate space. At 60,000 x 30,000 pixel canvases, sub-pixel rounding is invisible.

**Conversion chain**:
```
PPTX (EMUs) → pptxtojson (normalized) → PPT2VXP (normalize to 0-1) → Gen 2 storage (0-1) → Renderer (multiply by canvas pixel dimensions)
```

**OSD compatibility**: OSD's viewport coordinates are already normalized (0-1 range). SVG overlay maps directly.

## Canvas Aspect Ratio: 16:9 default, per-board override

- Default canvas: 16:9 (matches PowerPoint default)
- Store actual aspect ratio per-board (canvasWidth, canvasHeight as a ratio, not pixels)
- Some content will be 4:3 (older PPTX), square, or custom — the system MUST respect the source

## Action for PPT2VXP

- Normalize your existing position data one more step to 0-1 range (divide by slide dimensions)
- Preserve rotation through to output
- Preserve text color through sanitize() (the ~10 line fix you identified)
- Include slide dimensions in output so Gen 2 can store the aspect ratio
