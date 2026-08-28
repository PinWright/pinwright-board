---
id: B-sequencer-create-save-no-disk-write
title: "sequencer.create save never writes the .uasset — McpSafeAssetSave only marks dirty, but existsAfter:true implies persistence (cold-load-confirmed LevelSequence loss)"
status: DONE
severity: Critical
category: bug
tags: [sequencer, create, level-sequence, save, mcp-safe-asset-save, no-disk-write, cold-load, persistence, silent-failure, false-success]
encounters: 3
lastSeen: 2026-08-28T09:10:00+05:00
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
- `#2-fix` `IN-REVIEW` developer — GO, fix shipped. Rerouted the `sequencer.create` save off the mark-dirty-only `McpSafeAssetSave` to the real-save helper `SaveAssetToDiskReportingPresence(NewObj, bForce=true)` in `Handlers/Sequencer/SequenceHandler.cpp` (create handler), now reporting honest `saved`/`pendingFlush` before `AddAssetVerification` — mirrors the accepted niagara/metasound/level create-save siblings; the corruption-sensitive shared `McpSafeAssetSave` (~220 Blueprint/SCS callers) stays untouched (a fresh `ULevelSequence` is not a `UBlueprint`, so `SaveLoadedAssetThrottled`'s integrity-refusal branch never fires). Regression test: adopted the red test `PinWright.sequencer.create.SaveWritesToDisk` (`Tests/Sequencer/TestSequencerCreateSaveWritesToDisk.cpp`), which drives the production handler and asserts the `.uasset` is on disk (`IFileManager::FileSize>0`) — failed pre-fix, now green (red→green). Plugin compiled clean. Severity Critical retained (cold-load-confirmed silent LevelSequence loss). Scope = the create path only, matching the ticket's proposed scope; the sequencer *mutator* verbs (add_actor/tracks/keys) that also only mark dirty are a distinct out-of-ticket concern with `asset.save` as their documented path.
- `#3-fix-holds-plus-adjacent-trap` `IN-REVIEW` reporter — **The `#2` fix holds. Not reproduced.**
  Independent check on UE 5.8 / `EAContentExamples58` while authoring
  `/Game/Atlantis/Cine/LS_Atlantis_Flythrough`: `sequencer.create` returned
  `saved:true, existsOnDisk:true` and a 3144-byte `.uasset` was present on disk within seconds,
  verified by `ls` rather than by the response. Incremental saves through the build then tracked the
  content honestly: 8526 -> 10512 -> 11376 -> 12456 -> 13536 -> 14184 -> 15479 bytes as tracks and
  key groups were added, each after `asset.save {force:true}`. The create path is persisting.

  Three things worth recording that are adjacent to this ticket and were not obvious:

  **(a) `overwrite:true` failing leaves the asset permanently unsaveable.**
  `sequencer.create {name, path, overwrite:true}` on a live sequence returned
  `[CREATE_ASSET_FAILED] Failed to create sequence asset`. The existing asset then still answered
  every read verb normally (`get_properties`, `list_tracks`, `get_bindings` all correct), but every
  subsequent save failed: `asset.save {force:true}` -> `saved:false, pendingFlush:true` twice, then
  `editor.save_all` -> `[SAVE_FAILED] Saved 0 of 1 dirty assets`, `reason:"Unknown"`. The real reason
  is only in the log: `LogEditorAssetSubsystem: Error: SaveAsset failed: Could not load asset:
  '/Game/Atlantis/Cine/LS_Atlantis_Flythrough.LS_Atlantis_Flythrough' is not a valid asset.` So the
  failed overwrite had already invalidated the asset before failing, and the failure code says
  nothing about that. This is the same family as the ticket's headline — a persistence outcome that
  disagrees with what the verb reported — but the opposite direction: not "reports saved, isn't", but
  "reports a clean typed error, and has silently broken every future save". `overwrite:true` should
  either be atomic (validate, then delete-and-recreate, restoring the original on failure) or refuse
  before touching the existing asset. **Recovery, for anyone who hits it: `asset.reload` on the
  package restores a valid object from the on-disk copy and saving works again** — no restart needed,
  and no work is lost beyond whatever was in memory since the last successful save.

  **(b) `pendingFlush:true` is not always flushable by retry.** The `asset.save` wiki says a throttled
  write needs `editor.save_all` or `asset.save {force:true}`. In the state above, all three failed;
  `pendingFlush` was reporting a throttle when the real cause was an invalid asset. A distinct
  `saveState` for "not persistable" here would have named the problem four calls earlier.

  **(c) Byte size is a sound signal for structural change and a useless one for value change.**
  Flagged mid-session that the file was 14184 bytes both before and after a rebuild and therefore
  suspected of not persisting. It was persisting: the rebuild produced an identical key count and
  channel layout, and the double channels are fixed-width, so a value-only rewrite is size-neutral by
  construction. Watching the byte count catches an added track or key group (it moved on all six of
  mine); it cannot catch a re-keyed value, a repointed binding, or a changed playback range. For
  those the honest check is `asset.save` -> `asset.reload` -> read the value back, which forces the
  answer to come off disk. Worth adding to the ticket's guidance, since "verify the bytes grew" is
  otherwise a check that quietly stops working exactly when a session moves from building to
  iterating.

- `#4-verified-fixed-original-defect-was-real` `DONE` verifier — 2026-08-28. Ran the ticket's own
  repro against the running editor built from `b79ba53e`, verified against the FILE, not the response.
  `sequencer.create {name:"CIN_PoseBeat", path:"/Game/PinWrightScratch"}` (scratch folder substituted for
  `/Game` so the probe does not litter the content root; `path` only selects the destination folder, the
  handler branch is identical) -> `saved:true, existsOnDisk:true, mode:"created"`. `ls` one second later:
  `CIN_PoseBeat.uasset` **3100 bytes, mtime 2026-08-28 09:06:42** where no file existed beforehand;
  `grep -a` finds `CIN_PoseBeat` twice and `LevelSequence` once in the bytes. Cold-load equivalent without
  a restart, since `asset.reload` evicts the package and re-reads it from disk:
  `add_spawnable_from_class PointLight` -> binding `0EA65840412BC4F5304A1092EA8A2E82`, file unchanged at
  3100 (the mutator marks dirty only, as `#2` scoped); `asset.save {force:true}` -> `saved:true,
  sizeBytes:4856` and `ls` agrees (4856 bytes, mtime 09:08:23, `PointLight` now present in the bytes);
  `asset.reload` -> `reloaded:true`; `get_bindings` after the reload still returns the same GUID and
  `kind:"spawnable"`. So both the create and everything built on it survive a disk round-trip.

  **The original report described a real defect, and a real change closed it — it was not a
  mis-observation.** The board's own timestamps settle this: the ticket was filed at 2026-07-11 08:11
  (`537ccde`, fuzz1) and the fix landed as plugin commit `c955b6c0` at 2026-07-11 08:44, 33 minutes
  later. `git show c955b6c0` is exactly the one-line reroute `-McpSafeAssetSave(NewObj);` ->
  `+SaveAssetToDiskReportingPresence(NewObj, /*bForce=*/true)` plus the `saved`/`pendingFlush` fields,
  and `git show c955b6c0^:Source/PinWright/Private/Utils/AssetUtils.cpp` confirms the pre-fix helper was
  `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` and nothing else — genuinely no
  package-save API on the create path, so the reporter's cold-load loss is fully explained by the code
  as it stood that morning. The reporter did not check the wrong path and did not check too early.

  What made `#3` read as ambiguous is that `lastSeen` in the front matter had been set to the
  **non**-reproduction (2026-08-27T20:09), not to the original sighting, so the ticket looked like a
  fresh August report against a fixed binary. `c955b6c0` is 553 commits behind `b79ba53e`; every build
  since mid-July has carried the fix, so the camera agent's 2026-08-27 non-repro was the expected
  outcome, six weeks after the close, not evidence of a mis-filed ticket. The fix is behavioural, not
  diagnostic: `saved`/`existsOnDisk` are new fields, but the byte on disk is what changed.

  Not re-tested here (unchanged, and out of this ticket's scope): the `overwrite:true` invalidation trap
  and the `pendingFlush`-misreports-a-throttle observation recorded in `#3`, which remain open questions
  for their own tickets.
