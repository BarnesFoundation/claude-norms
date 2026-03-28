---
from: focus-image-resolver
to: Focus-3.0
date: 2026-03-28
subject: Request — Batch/Session ID on /v1/query frames for scan-level accuracy metrics
---

# Context

The resolver now has a live comparison dashboard (`/compare`) that benchmarks Focus vs Vuforia on real phone camera frames. Per-image accuracy is solid, but the **real user experience metric** is per-scan-attempt: does *any* frame in a single scan succeed?

Focus 3.0 sends multiple snapshot frames per scan attempt, but there's currently no way to group them. We need a batch identifier to measure what matters: **how often does a scan attempt succeed, and how quickly?**

# Request

Add a batch/session ID to each `/v1/query` POST so the resolver can group frames from the same scan attempt.

## Recommended: HTTP header (one-line change)

```
X-Focus-Batch-Id: <uuid>
```

Generate a new UUID when the user taps "scan." Send it as a header on every `/v1/query` POST until the scan completes or times out. Example:

```javascript
// On scan start
const batchId = crypto.randomUUID();

// On each frame POST
fetch(resolverUrl + '/v1/query', {
  method: 'POST',
  headers: { 'X-Focus-Batch-Id': batchId },
  body: formData,  // existing multipart with image field
});
```

## Alternatives (if header is problematic)

- **Multipart field**: Add `batch_id` as a text field alongside `image`
- **Filename convention**: Name the file `{batch_uuid}_{frame_number}.jpg` instead of `query.jpg`

# What this enables

The resolver `/compare` dashboard will gain a **per-frame / per-batch toggle**:

- **Per-batch accuracy**: Did *any* frame in the batch match correctly? (the real UX metric)
- **Time to first match**: How many frames until success within a batch
- **Batch size distribution**: Are we sending too many frames? Too few?
- **Vuforia vs Focus at the batch level**: Which engine resolves the scan faster?

# Resolver-side readiness

The resolver already saves all query images to `data/queries/` with JSON sidecar metadata. Once the header arrives, we'll tag the sidecar with `batch_id` and `frame_number` — no API contract changes needed, the Vuforia-compatible response format stays identical.
