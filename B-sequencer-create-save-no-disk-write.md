---
id: B-sequencer-create-save-no-disk-write
title: "sequencer.create save never writes the .uasset — McpSafeAssetSave only marks dirty, but existsAfter:true implies persistence (cold-load-confirmed LevelSequence loss)"
status: IN-REVIEW
severity: Critical
category: bug
tags: [sequencer, create, level-sequence, save, mcp-safe-asset-save, no-disk-write, cold-load, persistence, silent-failure, false-success]
encounters: 1
lastSeen: 2026-07-11T08:02:05.5151743+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T08:13:50.3505141+03:00
---

# `sequencer.create` reports the LevelSequence created (existsAfter:true) but never writes the .uasset to disk

`sequencer.create` builds a new `ULevelSequence` via
`LevelSequenceFactoryNew` and, on success, returns an
`existsAfter:true`/`assetClass:LevelSequence` verification block — so a
caller reasonably believes the new `.uasset` is on disk. It is not. The
handler routes the save through the shared no-op helper `McpSafeAssetSave`,
which only `MarkPackageDirty()` + `AssetCreated()` and never calls any
package-save API. After the editor closes (or a fuzz `git reset --hard`)
the sequence is gone, with no error ever surfaced — and every subsequent
verb (`add_actor`, tracks, keys) mutates the same never-persisted package
in memory, so nothing on the sequence survives either.

This is the LevelSequence analog of `B-metasound-create-save-no-disk-write`
(MetaSound path), `B-niagara-save-no-disk-write` (niagara path), and
`B-create-level-saved-true-no-umap` (level path): a create success signal
backed only by a mark-dirty, masked by a registry-based `existsAfter:true`.
No sequencer save ticket exists, so this fills the gap.

## Root cause (verified in source)

- `Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp:335` — `McpSafeAssetSave(NewObj);` inside the `sequencer.create` handler, immediately followed by `AddAssetVerification(Resp, NewObj);` (`:338`), which sets `existsAfter:true` from the asset registry, not from disk.
- `Plugins/PinWright/Source/PinWright/Private/Utils/AssetUtils.cpp:220-232` — `McpSafeAssetSave` only `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated(Asset)` then returns `true`; it never calls any package-save API. Its comment even documents the deferred-save behaviour ("UE 5.7+ Fix: Do not immediately save newly created assets to disk … mark the package dirty and notify the asset registry.") — that is deliberate for Blueprint/SCS bulkdata-corruption reasons, but it means a plain LevelSequence create is memory-only.

## Cold-load repro (confirmed by a real editor restart — CorruptionCheck)

Task: block out a cinematic beat. `sequencer.create {name: CIN_PoseBeat, path: /Game}` reported success with `existsAfter:true` and `assetClass: LevelSequence`; in the warm session the sequence read back as a `LevelSequence` with `bindingCount=2` (1 spawnable CineManny + 1 possessable BP_PhysicsControlCharacter), `playbackStart=0`/`playbackEnd=120000`, `frameRate 30/1`, `tickResolution 24000/1`. A plain quit+relaunch (NO git reset/clean/lfs) then found the asset ABSENT on disk:

- `editor.open_asset /Game/CIN_PoseBeat` -> `[ASSET_NOT_FOUND]`
- `asset.get /Game/CIN_PoseBeat` -> `[ASSET_NOT_FOUND]`
- `asset.exists` -> `exists=false`
- no `CIN_PoseBeat.uasset` anywhere under `Content/`, and `git status` shows no untracked file
- `editor.quit` reported `dirtyCount=2` (the package was still unsaved in memory) — the reported save never flushed to the `.uasset`

The editor stayed healthy (the MCP namespace index still responded), so the outcome is `load_failed` persistence loss, not a crash — a no-disk-write persistence bug the warm editor masked, identical in shape to the accepted `McpSafeAssetSave` siblings.

## What it should do

Mirror the accepted sibling fixes (`B-niagara-save-no-disk-write` #2,
`B-metasound-create-save-no-disk-write` #2, `B-create-level-saved-true-no-umap`):
persist for real on the sequencer create path — route the save through the
in-tree real-save helper (`SaveAssetToDiskReportingPresence` /
`SaveLoadedAssetThrottled(bForce=true)`), probe on-disk presence
(`IFileManager::FileSize`), gate via the shared `ShouldTreatAssetSaveAsSuccess`
predicate, and report an honest `saved`/`pendingFlush` field instead of an
unqualified `existsAfter:true`. A fresh `ULevelSequence` is not a
Blueprint/SCS asset, so the bulkdata-corruption vector that pins
`McpSafeAssetSave` on Blueprint edits does not apply — the shared no-op
`McpSafeAssetSave` (and its ~220 corruption-sensitive callers) stays untouched.

severity rationale: impact=corruption/silent-persistence-loss × reach=every-session -> Critical. The cold-load confirmation proves genuine silent asset loss with no signal; a caller who trusts `existsAfter:true` loses the whole sequence (and everything built on it) on the next editor close.

## History
- `#1-initial-repro` `OPEN` reporter — Cold-load-confirmed persistence loss on a cinematic-blockout task. `sequencer.create CIN_PoseBeat` reported `existsAfter:true`/`LevelSequence`, warm session read back 2 bindings + playback 0-120000 @ 30fps; a plain quit+relaunch (no baseline reset) then returned `[ASSET_NOT_FOUND]` from `editor.open_asset`/`asset.get`, `asset.exists=false`, no `.uasset` under `Content/`, and `editor.quit` had reported `dirtyCount=2`. Root cause verified in source: `SequenceHandler.cpp:335` `McpSafeAssetSave(NewObj)` + `:338` `AddAssetVerification` (registry existsAfter, not disk); `McpSafeAssetSave` (`AssetUtils.cpp:220-232`) only `MarkPackageDirty()`+`AssetCreated()`, never writes the package. Same root cause/code as the accepted `B-niagara-save-no-disk-write`/`B-metasound-create-save-no-disk-write`/`B-create-level-saved-true-no-umap` fixes, which each reroute their own create handler to the real-save helper but leave the sequencer create path untouched — no sequencer save ticket existed, so this fills the gap. Proposes routing the create save through `SaveAssetToDiskReportingPresence`, probing disk presence, and reporting `saved`/`pendingFlush` while leaving the shared corruption-sensitive `McpSafeAssetSave` alone.
- `#2-fix` `IN-REVIEW` developer — GO. Verified valid against current source: `sequencer.create` routes its save through the mark-dirty-only `McpSafeAssetSave` while `AddAssetVerification` hardcodes `existsAfter:true` (disk-unverified), and a red test reproduces the no-disk-write. Rerouting the create-path save to the real-save helper `SaveAssetToDiskReportingPresence(bForce=true)` and reporting honest `saved`/`pendingFlush`, mirroring the accepted niagara/metasound/level create-save siblings; the corruption-sensitive shared `McpSafeAssetSave` (Blueprint/SCS callers) stays untouched. Severity Critical retained (cold-load-confirmed silent LevelSequence loss). Scope = the create path only, matching the ticket's proposed scope.
