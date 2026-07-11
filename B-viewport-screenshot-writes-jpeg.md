---
id: B-viewport-screenshot-writes-jpeg
title: "ui.screenshot / editor.screenshot (PIE) write JPEG bytes into a .png file via shared CaptureGameViewportToPngFile — reported success + mimeType image/png"
status: IN-REVIEW
severity: High
category: bug
tags: [ui, editor, screenshot, image-format, jpeg, png, silent-mismatch, jpeg-in-png-silent-mismatch]
encounters: 1
lastSeen: 2026-07-11T21:29:01.6005724+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T21:41:58.0160616+03:00
---

# Game-viewport screenshots write JPEG bytes into a .png file

`ui.screenshot` and `editor.screenshot` (its PIE / game-viewport branch) both
capture through the shared helper
`PinWrightScreenshotUtils::CaptureGameViewportToPngFile`
(`Source/PinWright/Private/Utils/ScreenshotUtils.cpp`). That helper reports
`success` and writes a file at the caller's `.png` path, but the bytes it
writes are **JPEG/JFIF-encoded**, not PNG. The on-disk file has no PNG
signature and no IHDR chunk — its magic bytes are
`FF D8 FF E0 00 10 4A 46 49 46` (`...JFIF`, JPEG SOI + APP0), not the PNG
signature `89 50 4E 47 0D 0A 1A 0A`.

Both verbs document/return PNG: `editor.screenshot`'s wiki says "Capture a PNG
screenshot"; `ui.screenshot`'s success payload additionally sets
`"mimeType":"image/png"` and returns the same JPEG bytes as `imageBase64` — so
the response itself declares a format the bytes are not. Any consumer that
honors the extension, the declared mimeType, or sniffs the signature
(image importers, `PIL.Image.open` format asserts, browsers honoring the
declared type, contact-sheet builders) is fed a mislabeled file, and the
screenshot is silently **lossy** (JPEG artifacts on crisp UI text / thin
overlay lines — the exact opposite of what a UI-review/press-kit capture
wants).

## Root cause (guilty source)

`Source/PinWright/Private/Utils/ScreenshotUtils.cpp:155-176`:

```
TArray<uint8> PngData;
FImageUtils::ThumbnailCompressImageArray(OutWidth, OutHeight, Bitmap, PngData);
if (PngData.Num() == 0)
{
    IImageWrapperModule& ImageWrapperModule =
        FModuleManager::LoadModuleChecked<IImageWrapperModule>(FName("ImageWrapper"));
    TSharedPtr<IImageWrapper> ImageWrapper =
        ImageWrapperModule.CreateImageWrapper(EImageFormat::PNG);
    if (ImageWrapper.IsValid() &&
        ImageWrapper->SetRaw(Bitmap.GetData(), Bitmap.Num() * sizeof(FColor),
            OutWidth, OutHeight, ERGBFormat::BGRA, 8))
    {
        PngData = ImageWrapper->GetCompressed(100);
    }
}
```

`FImageUtils::ThumbnailCompressImageArray` emits **JPEG** for any image >= 8x8
(engine `USE_JPEG_FOR_THUMBNAILS`), and the result is stored in a var named
`PngData`. The genuine PNG encode via `IImageWrapper` / `EImageFormat::PNG`
sits inside an `if (PngData.Num() == 0)` fallback that only runs when the JPEG
compressor returned nothing — which never happens for a valid, non-trivial
bitmap. So for every normal capture the JPEG bytes are saved unchanged to the
`.png` `OutPath` (`SaveArrayToFile(PngData, *OutPath)` at line 179).

## Affected methods (shared-helper sweep)

`CaptureGameViewportToPngFile` call-sites in
`Plugins/PinWright/Source/PinWright/Private/`:

- `ui.screenshot` — `Handlers/UI/UiHandler.cpp:144` (reproduced: JPEG on disk + `mimeType:image/png` in response)
- `editor.screenshot`, PIE / game-viewport branch — `Handlers/Editor/ViewportHandler.cpp:448` (reproduced: JPEG on disk)

Not affected: `editor.screenshot` **outside** PIE takes the level-editor
viewport branch (`ViewportHandler.cpp:426`, `CaptureActiveLevelViewportToScreenshot`),
which correctly emits a genuine PNG (verified: `89 50 4E 47` signature). The
defect is specific to the game/PIE viewport capture path shared by the two
verbs above. (`Handlers/Drive/DriveSetOfMarkRenderer.cpp:194` only *mentions*
this helper in a comment; the drive set-of-mark path uses its own encoder and
is out of scope for this ticket.)

## Relationship to B-thumbnail-png-writes-jpeg

Same engine root pattern (`ThumbnailCompressImageArray` emits JPEG; PNG
fallback only runs when the JPEG output is empty), but a **different code
path**. `B-thumbnail-png-writes-jpeg` (IN-REVIEW) fixed only
`asset.generate_thumbnail` (via `ThumbnailEncodeUtils::EncodeByExtension`
routed into the thumbnail handler). Its history entry `#2` also *claims* it
"fixed the same latent JPEG-in-.png defect in ui.screenshot (UiHandler.cpp:~86
... its PNG fallback only ran when the JPEG output was empty)" — but that claim
does not hold at current HEAD: the capture was refactored into the shared
`CaptureGameViewportToPngFile` helper and still has exactly that structure, so
the `EncodeByExtension` fix never reaches the screenshot path. This ticket
tracks the still-live screenshot family separately so it is not marked resolved
on the thumbnail fix alone. (Distinct again from
`E-ui-screenshot-doubles-png-extension`, which is filename composition and now
produces a single `.png`, verified.)

## What it should do

Encode PNG for a `.png` output path. Either reuse the same
`EncodeByExtension` helper (`.png` -> `EImageFormat::PNG`) that
`asset.generate_thumbnail` was routed through, or drop
`ThumbnailCompressImageArray` here entirely and always PNG-encode via
`IImageWrapper` (the fallback block already present) since these verbs are
contractually PNG. At minimum, stop writing JPEG bytes to a `.png` name and
stop declaring `mimeType:image/png` for JPEG bytes.

**Workaround:** treat the output as a mislabeled image — re-detect format by
magic bytes and/or transcode; do not rely on the `.png` extension or the
declared `image/png` mimeType, and expect JPEG-lossy quality.

severity rationale: impact=silent-false-success (documented-PNG contract + false `mimeType:image/png` + lossy JPEG bytes on a normal PIE capture) × reach=common (game/PIE screenshot capture, two methods sharing one helper) -> High

## Verbatim repro

Editor launched, then PIE started via `editor.play` (`{}` -> `{"success":true}`).

RPC: `ui.screenshot`
args: `{"path":".../scratchpad","filename":"oracle_ui_pie_shot.png","returnBase64":false}`
result: `{"screenshotPath":".../scratchpad/oracle_ui_pie_shot.png","filename":"oracle_ui_pie_shot.png","width":926,"height":525,"sizeBytes":16751}`
on disk (`od -t x1 -N 16`): `ff d8 ff e0 00 10 4a 46 49 46 00 01 01 00 00 01` — JPEG/JFIF, no `89 50 4e 47` PNG signature; file size 16751 matches the reported `sizeBytes`.

RPC: `editor.screenshot`
args: `{"filename":"oracle_editor_pie_shot.png"}`
result: `{"width":926,"height":525,"path":".../Saved/Screenshots/oracle_editor_pie_shot.png"}`
on disk: `ff d8 ff e0 00 10 4a 46 49 46 ...` — same JPEG/JFIF.

Control (no PIE): `editor.screenshot {"filename":"oracle_replay_editor_shot.png"}` (level-editor viewport branch) -> on disk `89 50 4e 47 0d 0a 1a 0a 00 00 00 0d 49 48 44 52` — genuine PNG + IHDR, confirming the defect is confined to the game/PIE viewport path.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `ui.screenshot` and `editor.screenshot` inside PIE against a live viewport: both report success with a `.png` path (and `ui.screenshot` additionally returns `mimeType:image/png` + `imageBase64`), but the on-disk bytes are JPEG/JFIF (`ff d8 ff e0 ... 4a 46 49 46`), no PNG signature. Traced to shared helper `CaptureGameViewportToPngFile` (`ScreenshotUtils.cpp:156`): `FImageUtils::ThumbnailCompressImageArray` writes JPEG into a var named `PngData`, and the real PNG-encode fallback only runs on `PngData.Num()==0`, which never happens for a valid bitmap. `editor.screenshot` outside PIE (level-viewport branch, a different function) correctly emits PNG, isolating the defect to the game/PIE capture path shared by both verbs. Distinct code path from `B-thumbnail-png-writes-jpeg`'s thumbnail fix, whose `#2` claim of a ui.screenshot fix does not hold at HEAD.
- `#2-implement-png-encode` `IN-REVIEW` developer — Fixed via option (b): extracted a headless-callable helper `PinWrightScreenshotUtils::EncodeBitmapToPng(Width, Height, Bitmap, OutPng)` that always PNG-encodes the BGRA8 bitmap via `IImageWrapper` (no `FImageUtils::ThumbnailCompressImageArray`), and routed `CaptureGameViewportToPngFile` through it. So `ui.screenshot` and `editor.screenshot`'s PIE branch now write genuine PNG bytes to their `.png` paths and `ui.screenshot`'s `mimeType:image/png` is honest. Option (a)'s `EncodeByExtension`/`ThumbnailEncodeUtils` helper confirmed absent at HEAD `caa3a8a`, so option (b) was the only implementable route. Files: `Source/PinWright/Private/Utils/ScreenshotUtils.h`, `Source/PinWright/Private/Utils/ScreenshotUtils.cpp`. Regression test `PinWright.ui.screenshot.EncodesPngNotJpeg` (`Source/PinWright/Private/Tests/EditorOps/TestScreenshotPngEncode.cpp`) feeds a synthetic 16x16 FColor bitmap through the extracted helper and asserts the PNG signature `89 50 4E 47`, not-JPEG (`FF D8 FF`), and a PNG round-trip. Compile clean; test executed and passed; differential verified (pre-fix tree fails to compile — the test references the fix-introduced `EncodeBitmapToPng`). Same-defect third site `DriveSetOfMarkRenderer.cpp:263` (distinct verb/path, also declares `Mime:image/png`) filed separately as `B-drive-setofmark-writes-jpeg`.
- `#3-test-phase-fix` `IN-REVIEW` developer — Shipped mechanism diverges from `#2`: the diff-review fixer rewrote `EncodeBitmapToPng`'s body to delegate to the engine's canonical `FImageUtils::PNGCompressImageArray(Width, Height, TArrayView64<const FColor>(Bitmap.GetData(), Bitmap.Num()), Png64)` — the same FColor(BGRA8)->PNG encoder the sibling capture verbs use (e.g. `PreviewViewportCaptureUtils`, the level-viewport branch of `editor.screenshot`) — instead of the hand-rolled `IImageWrapper` `CreateImageWrapper(PNG)`+`SetRaw(BGRA,8)`+`GetCompressed(100)` dance `#2` described. The explicit `Width/Height<=0 || Bitmap.Num()<Width*Height` OOB guard (which `PNGCompressImageArray` does not perform) is retained; the now-orphaned `IImageWrapper.h`/`IImageWrapperModule.h`/`Modules/ModuleManager.h` includes were dropped from `ScreenshotUtils.cpp` (`ImageUtils.h` stays — now live for `PNGCompressImageArray`). Output contract unchanged (genuine PNG to the `.png` path, honest `mimeType:image/png`); no verb/param/field changed and no new test was added this phase. Verified green in this host: full PinWright suite 3657/3657 `Result={Success}`, 0 `Result={Fail}`, no crash; the `#2` regression test `PinWright.ui.screenshot.EncodesPngNotJpeg` executed and asserted (empty-events pass over its PNG-signature / not-JPEG / PNG round-trip `TestTrue`/`TestFalse`/`TestEqual` checks). Reviewers escalated no correctness concerns; the load-bearing OOB guard and the PNG (not JPEG) output were re-confirmed by the green suite.
