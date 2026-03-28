# Focus Image Resolver Integration with Focus 3.0

## Project
Focus 3.0 + Focus Image Resolver

## Date
2026-03-28

## What Was Built
Replaced Vuforia Cloud Reco with the self-hosted Focus Image Resolver in the Focus 3.0 visitor app. The resolver returns a Vuforia-compatible response format, so the frontend scan logic needed zero changes — only the server-side image submission was modified.

## Approach

### Server-side change (`scanController.ts`)
Replaced the `vuforia-api` npm package (which hardcodes the Vuforia Cloud Reco URL) with a direct `axios` + `form-data` POST to a configurable `REACT_APP_VUFORIA_REQUEST_URL` environment variable. The resolver's `/v1/query` endpoint accepts the same multipart `image` field and returns:

```json
{
  "results": [{
    "target_id": "6622",
    "target_data": { "name": "6622" },
    "score": 1.0,
    "crop": "center_50%",
    "_debug": { "confidence": 1.0, "num_inliers": 2850 }
  }]
}
```

The `target_data.name` field feeds directly into the existing `processIdentifiedItem` logic which handles `SPEX_*` (special exhibitions), `####__REF_*` (reference images), and plain numeric IDs.

### Client-side camera fix (`Camera.tsx`)
**Critical discovery**: the original code captured frames at CSS display resolution (276x229) instead of native camera resolution (1080x1080). The `getUserMedia` video element renders at display size, so `canvas.width = getBoundingClientRect().width` produces a tiny image. Fixed by using the `ImageCapture` API (`grabFrame()`) which reads directly from the camera hardware at native resolution.

### Debug overlay (`DebugOverlay.tsx`)
Added a real-time debug overlay showing per-query: attempt number, RTT, image size in KB, match result with score/crop/inliers, capture resolution. Also tracks TSTTM (Total Scan Time To Match) and TSTTF (Total Scan Time To Failure) — elapsed time from scan button press to result page or failure state.

## Key Files
- `server/api/controllers/scanController.ts` — axios POST replacing vuforia-api package
- `src/components/Camera.tsx` — ImageCapture API for native resolution
- `src/components/DebugOverlay.tsx` — debug overlay component
- `src/components/CameraContainer.tsx` — debug log state + TSTTM/TSTTF tracking
- `src/components/ArtworkWrapper.tsx` — debug overlay on result page
- `server/local-dev-server.ts` — lightweight server for DB-free testing

## Gotchas

1. **`getUserMedia` resolution ≠ canvas resolution** — The video element renders at CSS dimensions. Calling `drawImage(video)` onto a canvas sized with `getBoundingClientRect()` produces display-resolution images, not camera-resolution. Use `ImageCapture.grabFrame()` or size the canvas to `video.videoWidth/videoHeight`.

2. **Stale frames on mobile** — `drawImage(video)` from a `setInterval` callback may return the same frame repeatedly on mobile browsers. The video element doesn't advance frames between `requestAnimationFrame` calls. `ImageCapture.grabFrame()` solves this by reading directly from the camera track.

3. **CRLF line endings on Windows** — `react-scripts` 5.0.1 with `eslint-plugin-prettier` fails to compile if source files have CRLF. Convert to LF or add `endOfLine: "auto"` to prettier config.

4. **react-scripts 5.0.1 + Node 22** — The `allowedHosts` option causes a crash. Workaround: `DANGEROUSLY_DISABLE_HOST_CHECK=true` in `.env`.

5. **ngrok static domain** — On the hobbyist plan, every tunnel claims the static domain by default. You cannot create a random-URL tunnel via CLI flags alone; need separate domains from the dashboard.

## Reuse Potential
- **ImageCapture fix**: Any web app using `getUserMedia` + canvas for image capture at native resolution. This is a common pitfall.
- **Debug overlay pattern**: Reusable for any real-time image processing pipeline — shows the full request/response cycle with timing.
- **Vuforia replacement pattern**: Any app migrating from Vuforia Cloud Reco to a self-hosted matcher. The response format compatibility means zero frontend changes.
