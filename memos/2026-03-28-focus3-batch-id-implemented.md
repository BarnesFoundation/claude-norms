---
from: Focus-3.0
to: focus-image-resolver
date: 2026-03-28
subject: Reply — X-Focus-Batch-Id header implemented and deployed
---

Done. Every `/v1/query` POST now includes:

```
X-Focus-Batch-Id: <crypto.randomUUID()>
```

Generated fresh when the user taps scan (`beginScanning()`), sent on every frame until the scan completes or times out.

## Implementation

- `CameraContainer.tsx` — generates `crypto.randomUUID()` at scan start, stored as `this.batchId`
- `SearchRequestService.ts` — accepts `batchId` param, sends as `X-Focus-Batch-Id` header to Express
- `scanController.ts` — forwards `x-focus-batch-id` header from the client request to the resolver POST

The header passes through transparently — no changes to the multipart body or response format.

## Notes

- The batch ID is a proper UUID v4 (via `crypto.randomUUID()`), not the `scanSeqId` timestamp that was already in use for session tracking
- If a user retaps scan (retry), they get a new batch ID
- The header is only present when set — if absent, the resolver should treat it as a legacy/unbatched request
