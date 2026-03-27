# VXP Board API Contract

**Owner**: VXP-2.0 (BarnesFoundation/VXP-2.0)
**Target**: VXP 2.0 GraphQL API (port 4002)
**Last updated**: 2026-03-26

Only the VXP-2.0 repo's Claude sessions MUST update this file. Other repos MUST treat it as read-only.

---

## Current Board Types

| Type | Description |
|------|-------------|
| `IIIF` | Simple image grid via OSD collection mode |
| `PANORAMA` | 360° cube map panorama |
| `THREEDEE` | 3D model viewer |
| `VIDEO` | Video player with trim points |

## Creating a Board (Current Gen 1)

```graphql
mutation CreateBoard($input: CreateBoardInput!) {
  createBoard(input: $input) { board { id title sortOrder type } }
}
```

Input shape:

```json
{
  "meetingId": 123,
  "title": "Slide Title",
  "type": "IIIF",
  "caption": "Optional description",
  "sortOrder": 1,
  "boardImages": ["https://example.com/manifest.json"]
}
```

## Adding Images to a Board

```graphql
mutation CreateBoardImage($input: CreateBoardImage!) {
  createBoardImage(input: $input) { boardImage { id tileSource x y width height } }
}
```

Input shape:

```json
{
  "boardId": 456,
  "tileSource": "https://example.com/objects/789/manifest.json",
  "imageUploadId": 101,
  "x": 0, "y": 0, "width": 1024, "height": 768
}
```

**Important**: x, y, width, height exist in the schema but are NOT currently used by the renderer. They will become active with Gen 2 composed boards.

## Tile Source Resolution Order

The renderer resolves the display image using this fallback chain:

1. `BoardImage.tileSource` (direct URL)
2. `BoardImage.imageUpload.tileSource` (IIIF manifest after conversion)
3. `BoardImage.imageUpload.uploadUrl` (raw S3 image)
4. `BoardImage.imageUpload.thumbnailUrl` (fallback)

Producers MUST provide at least one of these. Prefer `tileSource` for direct IIIF manifests.

## Proposed: Gen 2 Composed Board

New BoardType (name TBD) that supports:

- Positioned images (x, y, width, height actually wired to OSD)
- SVG text/shape overlays
- Explicit canvas dimensions
- Does NOT replace existing IIIF type (backwards compatible)

Full spec: `VXP-2.0/docs/claude/context/iiif-board-gen2.md`

**Status**: Proposal — not yet implemented. Producers MUST NOT depend on Gen 2 fields until this contract is updated to reflect the final schema.

## Related Repos

| Repo | Role |
|------|------|
| **VXP-2.0** | Platform that renders boards (consumer) |
| **PPT2VXP** | PPTX to board conversion (producer) |
| **Future tinyPPT** | Authoring tool (producer) |
