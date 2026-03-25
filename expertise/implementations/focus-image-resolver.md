# Focus Image Resolver — Self-Hosted Image Recognition

## Summary
Self-hosted image recognition service replacing PTC Vuforia Cloud (~$48k/year) for museum artwork identification. Uses ORB feature detection + RANSAC geometric verification to match visitor phone camera captures against ~5,465 artwork reference images.

## Project
Focus-Image-Resolver (BarnesFoundation/Focus-Image-Resolver)

## Key Files
- `src/matching/extractor.py` — ORB and AKAZE feature extraction
- `src/matching/matcher.py` — Descriptor matching + RANSAC homography verification
- `src/matching/index.py` — In-memory descriptor index (runtime source of truth)
- `src/api/main.py` — FastAPI with Vuforia-compatible `/v1/query` endpoint
- `src/management/vuforia_export.py` — Migration tool: exports all 5,465 targets from Vuforia
- `src/config.py` — Pydantic-settings configuration (`FOCUS_` env prefix)
- `src/quality/validator.py` — Quality gates (sharpness, overlap, scale validation)

## Approach
- **Two-stage matching pipeline**: ORB keypoint extraction → Hamming distance matching → RANSAC homography → confidence scoring
- **In-memory descriptor index** loaded at startup from pre-computed `.npz` files, backed by PostgreSQL + pgvector for persistence
- **Vuforia-compatible API** (`/v1/query`) as drop-in replacement — Focus 3.0 requires minimal changes
- **Vuforia migration tooling** with HMAC-SHA1 VWS authentication to export all target metadata
- **Quality gating** rejects low-quality matches (blur, perspective distortion, insufficient inliers) — false positives are worse than false negatives
- **Cost optimization**: EC2 with museum-hours scheduling vs. always-on $48k cloud service

## What Worked
- Vuforia VWS API export: all 5,465 targets metadata retrieved with zero errors using rate-limited concurrent requests
- ORB provides the right performance/accuracy tradeoff for head-on artwork scanning (minimal skew)
- Python + OpenCV is vastly more capable than any Node OpenCV binding for this workload
- Naming convention analysis (`__REF_` suffix, `SPEX_` prefix) cleanly maps to the existing multi-image-per-object model

## What Didn't / Gotchas
- Vuforia VWS auth is HMAC-SHA1 with specific header ordering — easy to get wrong
- Vuforia advised *smaller* reference images (~1024x768) for better fingerprinting, not maximum resolution
- Binary ORB descriptors use Hamming distance, not L2 — pgvector float vectors require a translation layer

## Reuse Potential
- **Vuforia migration tool**: Any organization leaving Vuforia Cloud can reuse the export pipeline
- **Matching engine**: ORB + RANSAC pipeline is generic image matching, applicable beyond museum use
- **Architecture pattern**: In-memory index with DB persistence is reusable for any feature-matching service

## People
- Steve (architecture, product requirements, Vuforia domain knowledge)
- Claude/Steve/Focus-Image-Resolver (implementation)

## Date
2026-03-25 (ongoing)
