# Claude (Steve's sessions) — Focus Image Resolver

What this Claude has learned and implemented through hands-on work on Focus-Image-Resolver. This is accumulated implementation knowledge, not general training data.

## Implemented

### Vuforia VWS API Integration — Complete
- **File**: `src/management/vuforia_export.py`
- **What was built**:
  - HMAC-SHA1 authentication matching Vuforia VWS signing spec
  - Rate-limited concurrent export of all 5,465 targets with metadata
  - Name manifest builder mapping target UUIDs to human-readable names
  - Naming convention parser: primary objects (numeric ID), reference images (`__REF_` suffix), special exhibitions (`SPEX_` prefix)
- **Key insight**: Vuforia VWS uses server keys (not client keys) for target management API.

### ORB Feature Extraction Pipeline — Complete
- **File**: `src/matching/extractor.py`
- **What was built**: ORB keypoint detection + binary descriptor computation. AKAZE as configurable fallback. Configurable feature count via `FOCUS_ORB_FEATURES`.
- **Design decision**: ORB over SIFT/SURF for speed + patent-freedom. AKAZE available but not default.

### Descriptor Matching + Geometric Verification — Complete
- **File**: `src/matching/matcher.py`
- **What was built**: BFMatcher with Hamming distance, Lowe's ratio test, RANSAC homography, confidence scoring based on inlier count/ratio.

### In-Memory Descriptor Index — Complete
- **File**: `src/matching/index.py`
- **What was built**: Runtime descriptor store loaded from `.npz` files. Multiple descriptors per object ID grouped by naming convention.

### FastAPI Service with Vuforia-Compatible Endpoint — Complete
- **File**: `src/api/main.py`
- **What was built**: `POST /v1/query` drop-in Vuforia replacement + native CRUD endpoints.
- **Critical constraint**: Response format MUST match Vuforia's schema — Focus 3.0 depends on it.

### Quality Validation Gates — Complete
- **File**: `src/quality/validator.py`
- **What was built**: Laplacian variance sharpness scoring, minimum inlier thresholds, scale factor validation.

## Knows But Hasn't Implemented Yet
- Image download pipeline from Barnes S3/CloudFront (`{imageId}_{imageSecret}_{size}.jpg`)
- PostgreSQL + pgvector persistence (binary ORB descriptors need float conversion or bytea storage)
- EC2 deployment with museum-hours scheduling (Docker container built, IaC not yet written)

## Lessons Learned
1. Vuforia VWS auth is HMAC-SHA1 with signing string `{method}\n{content-md5}\n{content-type}\n{date}\n{path}`
2. Server keys vs. client keys are completely separate credential pairs in Vuforia
3. Smaller reference images (~1024x768) fingerprint better than massive originals
4. 5,465 targets = 3,124 unique objects via `__REF_` naming convention
5. Python is the right call for CV services even in a Node shop

## Date
2026-03-25 (ongoing)
