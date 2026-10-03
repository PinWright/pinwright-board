---
id: E-asset-dump-asset-compiling-no-wait
title: "asset.dump / texture.describe refuse ASSET_COMPILING with no bounded-wait option, and the asset.dump method page never names the code"
status: OPEN
severity: Medium
category: ergonomic
tags: [asset-dump, texture, async-compilation, wait, docs]
encounters: 1
lastSeen: 2026-09-30T00:00:00Z
rice: [2, 2, 1, 2]
priority: 17
---

# asset.dump / texture.describe refuse ASSET_COMPILING with no bounded-wait option

Follow-up to `B-texture-describe-size-resident-mip` (#4-defer-compiling-textures), which added the refusal. A direct `asset.dump` on a texture whose platform data is still compiling calls `DumpSingleAsset(..., /*bDeferAsyncCompilation=*/true)` and returns `ASSET_COMPILING` ("Retry asset.dump after compilation finishes"): `Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp:3514-3529`. `texture.describe` does the same at `Source/PinWright/Private/Handlers/Material/TextureHandler.cpp:3039-3046`.

Nothing lets the caller wait:
- `asset.dump` params are only `assetPath`, `outRoot`, `diff` and `includeWidgetScreenshot` (`AssetDumpHandler.cpp:3406-3411`), and there is no wait-for-compilation verb in any namespace.
- The folder sweep already waits. It requeues deferred entries and times out only after 120 s with no compile progress (`AssetDumpHandler.cpp:1934-1946`). The single-asset path skips that wait.
- The plugin already bounds-waits on one object elsewhere: `FAssetCompilingManager::Get().FinishCompilationForObjects(WaitSet)` at `Source/PinWright/Private/Handlers/Asset/ThumbnailFrameEvidence.cpp:61`.
- The generated `asset.dump.md` page never mentions `ASSET_COMPILING`. It appears only on the `asset.dump-sidecars` sub-page (`docs/wiki-src/asset.dump-sidecars.md:83`) and on `asset-audit.md:65`, so a caller that reads the method page has no retry contract.
- Python can't wait either, because `FAssetCompilingManager` is not exposed to Python.

Repro: consolidate or import a texture, then immediately `asset.dump` it.

**Workaround:** `asset.dump_folder {folderPath: <containing folder>, recursive: false}` dumps the asset through the deferring sweep, at the cost of re-dumping its siblings. The other option is to retry `asset.dump` after a delay.

**Fix:** Add an optional `waitForCompileSeconds` param (default 0 keeps the refusal) to `asset.dump` and `texture.describe`. When it is set, call `FinishCompilationForObjects({Texture})` before the dump, or poll `IsCompiling` until a deadline, and return `ASSET_COMPILE_TIMEOUT` if the deadline passes. Name the code and the new param in the `asset.dump` / `texture.describe` registration descriptions so they show up on the method pages.

## History

- `#1-no-wait-on-compiling` `OPEN` reporter — A subagent ran `asset.dump` on `.../T_Gray_Linear` right after consolidating assets and got `[ASSET_COMPILING] ... still compiling platform data. Retry asset.dump after compilation finishes.` A Python `unreal.AssetCompilingManager` wait returned `"output":"none"`. The subagent then abandoned the dump and wrote a custom compare script. Source check confirmed three things: no wait param or verb exists, the folder sweep already waits, and `FinishCompilationForObjects` is already used in `ThumbnailFrameEvidence.cpp:61`.
