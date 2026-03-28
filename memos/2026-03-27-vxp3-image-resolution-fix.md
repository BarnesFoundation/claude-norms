# VXP-3.0 Image Resolution Fix — CRITICAL

## Problem
The image recognition backend is receiving **276x229 pixel** images from VXP-3.0.
This is far too small for ORB feature matching to work reliably. The user has to be
practically inside the painting before a match registers.

Even our multi-scale center crops (50%, 33%, 25%) are useless at this resolution
because the crops become 138x115, 94x78, and 69x57 pixels respectively — too small
for any meaningful feature extraction.

## Root Cause (suspected)
VXP-3.0 is likely screenshotting the viewport/canvas element at display resolution
rather than capturing the actual camera frame at native resolution. The camera
captures at 1080p+ but the image sent to the API is only 276x229.

## Required Fix
When sending images to the `/v1/query` endpoint, VXP-3.0 must send the **native
camera frame** at high resolution, NOT a screenshot of the viewport canvas.

### Target resolution
- **Minimum**: 640x480 (will work but not ideal)
- **Recommended**: 1280x720 or higher
- **Ideal**: Native camera resolution (1080p+), JPEG compressed at quality 80-85

### How to capture
If using `getUserMedia` + `<video>` + `<canvas>`:

```javascript
// WRONG — captures at canvas display size (276x229)
canvas.width = canvas.clientWidth;
canvas.height = canvas.clientHeight;

// RIGHT — captures at native camera resolution
const videoTrack = stream.getVideoTracks()[0];
const settings = videoTrack.getSettings();
canvas.width = settings.width;   // e.g. 1920
canvas.height = settings.height; // e.g. 1080
```

Then when converting to blob for upload:
```javascript
// JPEG at quality 0.85 — good balance of size vs quality
canvas.toBlob(blob => {
    const formData = new FormData();
    formData.append('image', blob, 'query.jpg');
    fetch('http://localhost:8000/v1/query', { method: 'POST', body: formData });
}, 'image/jpeg', 0.85);
```

### File size expectations
- 276x229 JPEG: ~15-20 KB (current, too small)
- 1280x720 JPEG @ 0.85: ~100-200 KB (good)
- 1920x1080 JPEG @ 0.85: ~200-400 KB (ideal)

These sizes are fine for localhost and even for network transmission. The backend
processes a 1080p image in ~50-200ms depending on GPU availability.

## Impact
This single change will dramatically improve matching. Our test harness shows:
- At full resolution: 96% accuracy, matches at 30% of frame coverage
- At 276x229: barely matches even when filling the entire frame

## API endpoint
- **URL**: `http://localhost:8000/v1/query`
- **Method**: POST
- **Content-Type**: multipart/form-data
- **Field name**: `image`
- **Response format**: Vuforia-compatible JSON (unchanged)
