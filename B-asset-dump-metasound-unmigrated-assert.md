---
id: B-asset-dump-metasound-unmigrated-assert
title: "asset.dump* on an unmigrated MetaSound kills the editor via FindConstGraphChecked, silently truncating every folder sweep"
status: OPEN
severity: High
category: bug
tags: [asset-dump, metasound, audio, crash, assertion, dump-folder]
encounters: 1
lastSeen: 2026-09-01T00:00:00Z
---

# asset.dump* on an unmigrated MetaSound kills the editor via FindConstGraphChecked, silently truncating every folder sweep

`MetaSoundDumpBuilder::BuildMetaSoundJson` reads the document's paged graph through the engine's `checked` accessor. When the asset's document has not finished versioning, that graph does not exist yet and `FMetasoundFrontendGraphClass::FindConstGraphChecked` aborts the process — no MCP error, no skip stub, the editor dies. `asset.dump_folder` calls the same builder through `DumpSingleAsset`, so one such asset takes down a whole sweep mid-run.

**Stack:**

```
Assertion failed: FoundGraph [MetasoundFrontendDocument.cpp:1702]
  MetasoundFrontend.dll!FMetasoundFrontendGraphClass::FindConstGraphChecked()
  PinWright.dll!MetaSoundDumpBuilder::BuildMetaSoundJson()  [MetaSoundDumpBuilder.cpp:283 / :219]
  PinWright.dll!BuildAllFilesForAsset()                     [AssetDumpHandler.cpp:668]
  PinWright.dll!AssetDumpHandler::DumpSingleAsset()         [AssetDumpHandler.cpp:2232]
  PinWright.dll!TickFolderDump()                            [AssetDumpHandler.cpp:1539]
```

**Repro** (UE 5.8, host project `X:\src\unreal\unreal-fpv-dev`, 2026-09-01):

1. Fresh editor, no MetaSound asset preloaded.
2. `asset.dump_folder {folderPath:"/Game"}`.
3. Editor aborts at 753 / 22,426 assets on `/Game/Audio/Sounds/Weapons/MS_WavePlayerCrossfader`. `asset.dump` on that path alone reproduces it directly.

**Root cause — a same-tick race, not a corrupt asset.** MetaSound logs `Delaying asset versioning due to need to async load soft references` and defers document migration to a later tick for any unmigrated asset that has soft references. The dump loads the asset and builds `metasound.json` inside the same synchronous call, before the deferred migration runs, so the paged graph is genuinely absent when the `checked` accessor runs.

**Impact.** Beyond the crash, this silently truncated every prior full sweep: the committed asset-dump mirror holds 15,274 dumps against 30,807 assets in scope — roughly half missing, consistent with earlier sweeps dying on this assert partway through. A partial mirror reads as a complete one, so consumers trust missing data as absent data.

**Workaround** (verified, no source change): via `python.execute`, pre-load every MetaSound asset under the roots being dumped (143 in this project), wait ~20 s for the deferred versioning tick to land — visible in the log as `Migrated Class Interface paged graph` — then run the sweep. It completed clean.

**Fix:** in `MetaSoundDumpBuilder.cpp`, replace the `checked` graph accessor with the non-checked one and, when the document version is still pending, either emit a skip stub (`"skipped": true` plus a `skipReason`) or requeue the asset for a later tick of the folder dump. A folder sweep must never be able to take the editor down on one asset.

## History
- `#1-initial-repro` `OPEN` reporter — `asset.dump_folder {folderPath:"/Game"}` aborted the editor at 753 / 22,426 on `/Game/Audio/Sounds/Weapons/MS_WavePlayerCrossfader` with `Assertion failed: FoundGraph [MetasoundFrontendDocument.cpp:1702]` from `FindConstGraphChecked`, called by `MetaSoundDumpBuilder::BuildMetaSoundJson` (`MetaSoundDumpBuilder.cpp:283`/`:219`) via `BuildAllFilesForAsset` (`AssetDumpHandler.cpp:668`) → `DumpSingleAsset` (`:2232`) → `TickFolderDump` (`:1539`). Cause is a same-tick race: MetaSound logs `Delaying asset versioning due to need to async load soft references` and defers migration to a later tick, while the dump builds the sidecar synchronously on the same tick, so the paged graph is absent. Consequence beyond the crash: the committed asset-dump mirror held 15,274 dumps against 30,807 assets in scope, consistent with earlier sweeps dying here unnoticed. Workaround that completed a clean sweep: pre-load all 143 MetaSound assets under the dumped roots via `python.execute`, wait ~20 s for `Migrated Class Interface paged graph` in the log, then dump. Wanted fix: non-checked accessor plus a skip stub or a deferral to a later tick.
