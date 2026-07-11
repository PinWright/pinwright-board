---
id: B-drive-setofmark-writes-jpeg
title: "drive set-of-mark screenshot (DriveSetOfMarkRenderer::CaptureAnnotated) writes JPEG bytes but reports mimeType image/png"
status: OPEN
severity: Medium
category: bug
tags: [drive, screenshot, image-format, jpeg, png, silent-mismatch, jpeg-in-png-silent-mismatch]
encounters: 1
lastSeen: 2026-07-11T21:45:00+03:00
---

# drive set-of-mark screenshot encodes JPEG but declares mimeType image/png

`FDriveSetOfMarkRenderer::CaptureAnnotated`
(`Source/PinWright/Private/Handlers/Drive/DriveSetOfMarkRenderer.cpp`) captures the
game/PIE viewport (or the editor window), paints its interaction marks, then
PNG-encodes the annotated bitmap — except the encode has the **same
JPEG-in-a-PNG-name anti-pattern** as `B-viewport-screenshot-writes-jpeg`:

```cpp
// DriveSetOfMarkRenderer.cpp:262-276
TArray<uint8> PngData;
FImageUtils::ThumbnailCompressImageArray(Width, Height, Bitmap, PngData);
if (PngData.Num() == 0)
{
    IImageWrapperModule& ImageWrapperModule = ...CreateImageWrapper(EImageFormat::PNG);
    if (ImageWrapper.IsValid() && ImageWrapper->SetRaw(..., ERGBFormat::BGRA, 8))
    {
        PngData = ImageWrapper->GetCompressed(100);
    }
}
```

`FImageUtils::ThumbnailCompressImageArray` emits **JPEG** for any image >= 8x8
(engine `USE_JPEG_FOR_THUMBNAILS`), and the genuine `IImageWrapper` PNG encode sits
in a dead `if (PngData.Num() == 0)` fallback that never runs for a valid bitmap. The
bytes are then delivered (inline base64 or written to `Saved/Screenshots/Drive`) and
the result is stamped `OutScreenshot.Mime = TEXT("image/png")`
(`DriveSetOfMarkRenderer.cpp:291`) — so the set-of-mark screenshot is JPEG/JFIF bytes
declared as `image/png`, exactly the silent mislabel of the parent ticket but on the
drive agentic-vision surface (reached via `drive.observe` / set-of-mark capture).

Distinct code path from `B-viewport-screenshot-writes-jpeg` (that ticket fixes the
shared `PinWrightScreenshotUtils::CaptureGameViewportToPngFile`; this is a separate
function with its own inline encode) and from `B-thumbnail-png-writes-jpeg`
(`asset.generate_thumbnail`). Filed separately so folding it into the viewport-screenshot
fix does not over-scope a distinct verb.

Note: `F-drive-observe-screenshot-inline-base64` describes this same delivery path and
assumes it "PNG-encodes" — that assumption is false at HEAD for the reason above.

**Workaround:** treat the returned bytes as a mislabeled image — detect the true format
by magic bytes (`FF D8 FF` = JPEG); do not trust the `image/png` mime.

**Fix:** encode genuine PNG. Reuse `PinWrightScreenshotUtils::EncodeBitmapToPng` (the
headless-callable BGRA8 -> PNG helper extracted for `B-viewport-screenshot-writes-jpeg`)
in place of the `ThumbnailCompressImageArray` + dead-fallback block, so the delivered
bytes match the declared `image/png` mime. Add a differential regression test that feeds a
synthetic >=8x8 FColor bitmap through the encode path and asserts a PNG signature, not JPEG.

severity rationale: impact=silent-false-success (JPEG bytes + false `mimeType:image/png` +
lossy on crisp annotations) but reach=specialized (the drive set-of-mark capture surface,
narrower than the two canonical screenshot verbs) -> Medium. May warrant High if the false
`image/png` on the primary agentic-vision path breaks mime-sniffing consumers in practice.

## History
- `#1-source-identified-sibling` `OPEN` reporter — Identified during the fix analysis of `B-viewport-screenshot-writes-jpeg`: `FDriveSetOfMarkRenderer::CaptureAnnotated` (`DriveSetOfMarkRenderer.cpp:262-276`) runs the identical `FImageUtils::ThumbnailCompressImageArray` (JPEG for >=8x8) -> `PngData` with the real `IImageWrapper` PNG encode gated behind a dead `PngData.Num()==0` fallback, then stamps `OutScreenshot.Mime = "image/png"` (`:291`). So drive set-of-mark screenshots are JPEG/JFIF bytes reported as `image/png`. Source-read confirmation (not an independent live RPC repro); a distinct verb/path from the parent's shared `CaptureGameViewportToPngFile`, so tracked as its own ticket rather than folded into the parent fix.
