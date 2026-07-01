---
id: B-thumbnail-png-writes-jpeg
title: "asset.generate_thumbnail writes JPEG bytes into a .png outputPath (reports success)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset, thumbnail, image-format, png, jpeg, silent-mismatch]
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

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `asset.generate_thumbnail` with a `.png` `outputPath` and 512x512 on `/Game/Global/DemoRoom/Meshes/Display_Main_B`; RPC returned `success:true` with the `.png` path, but the file on disk is JPEG/JFIF (`ff d8 ff e0 ... 4a 46 49 46`), no PNG signature. Reproduced on a second mesh (`SM_Lightbulb`). Extension says PNG, bytes are JPEG, success is reported anyway — silent format mismatch.
- `#2-encode-by-extension` `IN-REVIEW` developer — Root cause: `FImageUtils::ThumbnailCompressImageArray` always emits JPEG for images >= 8x8 (engine `USE_JPEG_FOR_THUMBNAILS`), and the bytes were written to the caller's `.png` path unchanged. Fix (option a): added `Private/Handlers/Asset/ThumbnailEncodeUtils.h` with `EditorAutomationThumbnail::EncodeByExtension()`, which selects the encoder from the output extension via `IImageWrapper` (`.png` -> `EImageFormat::PNG`, `.jpg`/`.jpeg` -> JPEG, default PNG) and no longer falls back to the JPEG thumbnail compressor for a `.png` request. Routed `asset.generate_thumbnail` (`AssetWorkflowHandler.cpp:~649`) through it. Also fixed the same latent JPEG-in-`.png` defect in `ui.screenshot` (`UiHandler.cpp:~86`, where the JPEG bytes were stored in a var literally named `PngData` and its PNG fallback only ran when the JPEG output was empty — never, for a valid bitmap). Regression test: `EditorAutomationRpcGateway.asset.generate_thumbnail.EncodesPngByExtension` in `Tests/Assets/TestAssetHandlers.cpp` drives the production `EncodeByExtension` helper and asserts the `.png` output carries the PNG signature `89 50 4E 47 0D 0A 1A 0A` (and is NOT a JPEG SOI) while `.jpg` carries `FF D8` — it fails if the encode-by-extension path is reverted.
