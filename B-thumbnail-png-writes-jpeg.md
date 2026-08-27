---
id: B-thumbnail-png-writes-jpeg
title: "asset.generate_thumbnail writes JPEG bytes into a .png outputPath (reports success)"
status: DONE
severity: Medium
category: bug
tags: [asset, thumbnail, image-format, png, jpeg, silent-mismatch]
encounters: 2
lastSeen: 2026-08-27T18:58:26+05:00
---

# asset.generate_thumbnail writes JPEG bytes into a .png outputPath

`asset.generate_thumbnail` honors the `outputPath` and the requested
`width`/`height`, and returns `success:true`, but the file it writes is
**JPEG/JFIF-encoded** regardless of the `.png` extension on `outputPath`.
The on-disk file therefore has no PNG signature and no IHDR chunk — its
magic bytes are `FF D8 FF E0 00 10 4A 46 49 46` (`...JFIF`), i.e. JPEG SOI +
APP0, not the PNG signature `89 50 4E 47 0D 0A 1A 0A`.

The wiki documents `outputPath` only as "File path to save thumbnail to
disk" with no format field, so a caller naturally infers the format from the
extension they pass. Passing `Foo.png` and getting JPEG bytes is a silent,
misleading success: any consumer that detects format by signature or by
extension (image importers, contact-sheet builders, `PIL.Image.open` format
asserts, browsers honoring the declared type) is fed a mislabeled file. The
RPC neither rejects `.png` nor reports the real encoding in the result.

Reproduced on two different StaticMeshes in this session — both `.png` files
came out JPEG.

**What it should do:** either (a) encode according to the requested extension
(`.png` -> PNG via `IImageWrapper`/`EImageFormat::PNG`), or at minimum
(b) report the actual format in the success payload (e.g. `"format":"jpeg"`)
and/or write to a matching extension so the file is not mislabeled. PNG is the
lossless format expected for thumbnail/contact-sheet workflows.

**Workaround:** treat the output as JPEG regardless of the `.png` name
(re-detect by magic bytes), or transcode after the fact.

## Verbatim repro

RPC: `asset.generate_thumbnail`
args: `{"assetPath":"/Game/Global/DemoRoom/Meshes/Display_Main_B","width":512,"height":512,"outputPath":"X:/src/unreal/EAContentExamples57-fuzz2/Saved/EditorAutomation/thumbnails/Display_Main_B.png"}`
result: `{"success":true,"assetPath":"/Game/Global/DemoRoom/Meshes/Display_Main_B","width":512,"height":512,"outputPath":".../Display_Main_B.png"}`
on disk (`od -t x1 -N 16`): `ff d8 ff e0 00 10 4a 46 49 46 00 01 01 00 00 01` — JPEG/JFIF, no `89 50 4e 47` PNG signature.

Second confirmation, `assetPath=/Game/ExampleContent/Blueprint_Communication/Meshes/SM_Lightbulb` -> `SM_Lightbulb.png` first bytes `ff d8 ff e0 ... 4a 46 49 46` (same JPEG/JFIF).

(Note: the documented-optional inline mode — calling with no `outputPath` —
returns only `success`+`width`+`height` with no image bytes/temp path. That is
within the documented contract, `outputPath` is optional and no result field
promises returned bytes, so it is not filed here; the format mismatch above is
the load-bearing defect.)

## Encounter 2026-08-27 — the `#4` primitive-override proof ran on a `UMaterial`, so it does not cover material instances

Recorded as a narrowing of scope, **not** a status change — only the tester workflow closes or
reopens a ticket, and nothing here contradicts what `#4` verified.

`#4-built-and-verified-live` closed with a non-vacuousness proof for the scoped-restore assertion:
that the `primitive` override really applies, shown on
`/Game/DotaBlockout/Materials/M_DotaRock` — "the cylinder and sphere renders differ in 97.3% of a
64x64 luma grid, mean abs diff 80/255". That proof is sound and the restore assertion it protects
still stands.

The narrowing: **`M_DotaRock` is a `UMaterial`.** The Atlantis build (2026-08-27, UE 5.8, this
checkout) found that `primitive` is silently replaced by a flat plane on some material
**instances** — filed as `B-thumbnail-primitive-ignored-on-instances`, where a plain instance of
`M_Fish` renders a flat quad for `primitive:"sphere"` while `M_Fish` itself renders a sphere. A
master is exactly the case that works, so `#4`'s demonstration says nothing about the failing case
and must not be cited as evidence that `primitive` is honoured generally.

Nothing in this ticket's own subject — the encoder, the alpha stamp, the size reporting, the
scoped restore — is affected. `encounters` seeded and bumped to 2, `lastSeen` refreshed; `status`
deliberately untouched.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `asset.generate_thumbnail` with a `.png` `outputPath` and 512x512 on `/Game/Global/DemoRoom/Meshes/Display_Main_B`; RPC returned `success:true` with the `.png` path, but the file on disk is JPEG/JFIF (`ff d8 ff e0 ... 4a 46 49 46`), no PNG signature. Reproduced on a second mesh (`SM_Lightbulb`). Extension says PNG, bytes are JPEG, success is reported anyway — silent format mismatch.
- `#2-encode-by-extension` `IN-REVIEW` developer — Root cause: `FImageUtils::ThumbnailCompressImageArray` always emits JPEG for images >= 8x8 (engine `USE_JPEG_FOR_THUMBNAILS`), and the bytes were written to the caller's `.png` path unchanged. Fix (option a): added `Private/Handlers/Asset/ThumbnailEncodeUtils.h` with `EditorAutomationThumbnail::EncodeByExtension()`, which selects the encoder from the output extension via `IImageWrapper` (`.png` -> `EImageFormat::PNG`, `.jpg`/`.jpeg` -> JPEG, default PNG) and no longer falls back to the JPEG thumbnail compressor for a `.png` request. Routed `asset.generate_thumbnail` (`AssetWorkflowHandler.cpp:~649`) through it. Also fixed the same latent JPEG-in-`.png` defect in `ui.screenshot` (`UiHandler.cpp:~86`, where the JPEG bytes were stored in a var literally named `PngData` and its PNG fallback only ran when the JPEG output was empty — never, for a valid bitmap). Regression test: `EditorAutomationRpcGateway.asset.generate_thumbnail.EncodesPngByExtension` in `Tests/Assets/TestAssetHandlers.cpp` drives the production `EncodeByExtension` helper and asserts the `.png` output carries the PNG signature `89 50 4E 47 0D 0A 1A 0A` (and is NOT a JPEG SOI) while `.jpg` carries `FF D8` — it fails if the encode-by-extension path is reverted.

- `#3-ported-to-pinwright` `IN-REVIEW` developer — The `#2` fix landed in a different (older, `EditorAutomation`-named) checkout and was never ported here; this tree still encoded through `FImageUtils::ThumbnailCompressImageArray` and `ThumbnailEncodeUtils.h` did not exist. Ported with this tree's naming: added `Private/Handlers/Asset/ThumbnailEncodeUtils.h/.cpp` exposing `PinWrightThumbnail::EncodeByExtension()`, which selects the encoder from the output extension (`.png` and anything unrecognised -> PNG via the shared `PinWrightScreenshotUtils::EncodeBitmapToPng`, `.jpg`/`.jpeg` -> JPEG via `IImageWrapper` at quality 90) and reports the chosen format back. `asset.generate_thumbnail` (`AssetWorkflowHandler.cpp`) now routes through it and echoes `format` in the response. Two further defects fixed in the same pass, both of which the JPEG encoding was masking: (a) **alpha** — the handler copied the render alpha through untouched and the thumbnail path does no alpha fix-up, so switching to PNG without stamping would have shipped the ~99.97%-transparent-image defect; `ForceOpaqueAlpha` is now called INSIDE `EncodeByExtension` so no caller can bypass it; (b) **size** — the engine treats width/height as a maximum and shrinks to preserve aspect for some asset types, and the handler encoded at the REQUESTED size, feeding the encoder short data; it now encodes at `FObjectThumbnail::GetImageWidth()/GetImageHeight()` and reports the real size plus `requestedWidth`/`requestedHeight` when they differ. Also removed the unconditional `MarkPackageDirty()`, which left a read-only preview verb dirtying the asset it previewed. Regression tests in `Tests/Assets/TestGenerateThumbnail.cpp`: `EncodesByExtension` asserts the PNG signature (and NOT JPEG SOI) for `.png`, JPEG SOI for `.jpg`/`.jpeg`, PNG for absent/unknown extensions; `EncodedImageIsOpaque` feeds a fully transparent bitmap through the PRODUCTION encode entry point and asserts every decoded pixel is opaque with RGB intact (deliberately not calling `ForceOpaqueAlpha` itself — the earlier alpha test did and kept passing with the production call site deleted); `WritesPngAndLeavesAssetClean` drives the RPC end to end and asserts the file signature, opacity, and that the package is not dirtied. Not verified at runtime: no build and no editor run were permitted in this session.

- `#4-built-and-verified-live` `DONE` tester — Built clean on UE 5.8 (zero errors, zero warnings,
  unity merging in force via `-DisableAdaptiveUnity`). Suite 3605/3607 against a 3588/3590 baseline;
  the only 2 failures are the pre-existing `localization.Validation.*` pair. All 6
  `asset.generate_thumbnail` tests pass, including `WritesPngAndLeavesAssetClean`. Live verification
  on `/Game/DotaBlockout/Materials/M_DotaRock`, measured rather than eyeballed: a `.png` `outputPath`
  produced magic bytes `89 50 4E 47 0D 0A 1A 0A` with `format: "png"` (the original defect is gone —
  it is no longer `FF D8 FF E0`), and a `.jpg` path produced `FF D8 FF E0 00 10 4A 46` with
  `format: "jpeg"`, so the extension really selects the encoder in both directions. **Alpha checked
  by histogram, not by eye**: 262144/262144 pixels at alpha 255 = 100.0000% opaque on every PNG
  written, so the transparency defect the JPEG format was masking is genuinely fixed. **The
  scoped-restore assertion no unit test can make**: `M_DotaRock` has `thumbnail_info = None` and a
  clean package before the call — the hardest case, because the guard must synthesise a transient
  ThumbnailInfo and put a null back. After rendering twice with overrides
  (`primitive: cylinder|sphere`, `azimuth: 45`, `elevation: 10`), the asset was still
  `dirty = False` with `thumbnail_info = None`, byte-identical to before. The override is proven to
  have actually APPLIED rather than silently no-opping (which would make the restore check vacuous):
  the cylinder and sphere renders differ in 97.3% of a 64x64 luma grid, mean abs diff 80/255, and
  visual inspection confirms one cylinder and one sphere. Committed as `753a3a4a`.

- `#5-primitive-proof-narrowed-to-umaterial` `DONE` reporter — Encounter 1 -> 2, scope narrowing only; status deliberately NOT changed (only the tester workflow closes or reopens). Found on the Atlantis build 2026-08-27, UE 5.8, PinWright at this checkout's HEAD. `#4-built-and-verified-live` proved the `primitive` override is not a silent no-op — and therefore that its scoped-restore assertion is not vacuous — on `/Game/DotaBlockout/Materials/M_DotaRock` ("cylinder and sphere renders differ in 97.3% of a 64x64 luma grid, mean abs diff 80/255"). That proof is sound and stands. The narrowing: `M_DotaRock` is a **`UMaterial`**. `B-thumbnail-primitive-ignored-on-instances` (new, OPEN, High) reports `primitive` being silently replaced by a flat plane on material **INSTANCES** whose base material carries particle-sprite or Niagara usage, while the master renders the requested shape correctly — so a master is exactly the case that works, and `#4`'s demonstration does not cover the failing case and cannot be cited as evidence that `primitive` is honoured generally. Nothing in this ticket's own subject (encoder, alpha stamp, size reporting, scoped restore) is affected.
