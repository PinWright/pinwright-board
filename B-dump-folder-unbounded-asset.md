---
id: B-dump-folder-unbounded-asset
title: "asset.dump_folder per-tick budget does not bound one asset, allowing an editor hard-freeze"
status: OPEN
severity: Critical
category: bug
tags: [asset-dump, performance, editor-freeze, progress, ticker]
encounters: 1
lastSeen: 2026-07-31T11:57:40Z
---

# asset.dump_folder per-tick budget does not bound one asset, allowing an editor hard-freeze

`asset.dump_folder` advertises small per-tick batches, but `TickFolderDump` only checks its 8 ms budget before synchronously calling `DumpSingleAsset` (`Source/PinWright/Private/Handlers/Asset/AssetDumpHandler.cpp:1010-1027`). One asset then runs `LoadObject` (`AssetDumpHandler.cpp:1447-1453`), every applicable sidecar builder, and the dump write (`AssetDumpHandler.cpp:1582-1592`) as one opaque game-thread operation with no yield or bound.

A pathological asset can therefore hold the editor thread indefinitely. Progress is emitted only after that call returns (`AssetDumpHandler.cpp:1190-1204`) and carries aggregate counts, while `FAsyncFolderDumpState` has no current-asset or current-phase field (`Source/PinWright/Private/State/AsyncFolderDumpState.h:27-55`). During the stall, progress stays flat and the same blocked editor cannot answer `system.job_status`, so the caller cannot identify the offending asset or phase.

Live `/Game` sweep evidence: progress stalled at about 4,679 / 22,390, the editor hard-froze inside the dump, required termination, and Windows continued terminating process residue afterward. By contrast, `/App` completed and later converged with 8,192 assets unchanged. This is crash-equivalent loss of editor availability on a normal deep-inspection workflow.

**Workaround:** dump smaller subfolders and restart Unreal after a stall; this narrows the suspect set but cannot prevent a single pathological asset from freezing the editor.
**Fix:** split per-asset dumping into yieldable phases/aspects with a real per-tick bound, and publish `currentAsset` plus `currentPhase` before entering each phase. Any phase that cannot yield must have a bounded failure path rather than holding the editor indefinitely.

## History
- `#1-game-hard-freeze` `OPEN` reporter — `asset.dump_folder {folderPath:"/Game"}` stalled at about 4,679 / 22,390 with flat progress, hard-froze Unreal, and required terminating the editor; source validation shows the 8 ms loop budget cannot preempt one synchronous `DumpSingleAsset`, and progress exposes neither the current asset nor phase. `/App` completed and later converged with 8,192 unchanged.
- `#2-observable-cooperative-boundary` `OPEN` developer — Partial mitigation implemented: progress now publishes current asset, phase, and elapsed timing before synchronous work, compiling textures are deferred with a 120-second timeout, and cancellation stops at the next asset boundary. This does not satisfy the ticket's hard-bound acceptance criterion because arbitrary synchronous UObject work already executing on the game thread still cannot be preempted; keep OPEN for a yieldable or isolated per-asset execution design.
