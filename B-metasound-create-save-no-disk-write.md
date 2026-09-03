---
id: B-metasound-create-save-no-disk-write
title: "create_metasound (Source) / create_metasound_patch / create_metasound_preset save never writes the .uasset — McpSafeAssetSave only marks dirty, but existsAfter:true implies persistence (cold-load-confirmed loss)"
status: IN-REVIEW
severity: Critical
category: bug
tags: [audio, metasound, save, create-metasound, create-metasound-source, create-metasound-patch, mcp-safe-asset-save, no-disk-write, cold-load, persistence, silent-failure, false-success]
encounters: 3
lastSeen: 2026-09-02T20:35:00Z
---

# create_metasound_patch's save:true claims the asset is saved but writes nothing to disk

`audio.authoring.create_metasound_patch` (and the sibling
`create_metasound_preset`) take a `save` param defaulting to `true` and,
on success, return an `existsAfter:true` verification block — so a caller
reasonably believes the new `.uasset` is on disk. It is not. With
`save:true` the handler only **marks the package dirty**; the file is not
written until a separate `editor.save_all`. After the editor closes (or a
fuzz `git reset --hard`) the asset is gone, with no error ever surfaced.

This is the MetaSound-create analog of `B-niagara-save-no-disk-write`
(niagara path) and `B-create-level-saved-true-no-umap` (level path): a
`save:true`-style success signal backed only by a mark-dirty, masked by a
registry-based `existsAfter:true`.

## Root cause (verified in source)

`MetaSoundPatchPresetHandler.cpp` routes the `save` param through the
shared no-op helper:

```cpp
// create_metasound_patch
bool bSave = Ctx.GetBool(TEXT("save"), true);
...
if (bSave) { McpSafeAssetSave(Patch); }            // :90-93
AddAssetVerification(Result, Patch);               // sets existsAfter:true (registry, not disk)

// create_metasound_preset
if (bSave) { McpSafeAssetSave(NewAsset); }          // :217-220
```

`McpSafeAssetSave` (`Utils/AssetUtils.cpp:214-226`, quoted in
`B-niagara-save-no-disk-write`) **never calls any package-save API** —
it only `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` and
returns `true` unconditionally (the deferred-save behaviour is deliberate
and corruption-driven, per `B-bp-saved-state-corruption-mcp-edits`). So
the patch/preset is registered (hence `existsAfter:true`,
`describe_metasound` reads it) but never lands on disk. The handler's
`save:true` default + `existsAfter:true` response together imply a
persistence that did not happen, and the response carries no
`pendingFlush`/disk-presence signal to say otherwise.

`B-niagara-save-no-disk-write` explicitly names this same path as affected
("MetaSound, StateTree, etc. — all report a save that did not happen") but
**scopes its fix to the niagara save path only**, leaving the MetaSound
create handlers unfixed — there is no MetaSound save ticket. This ticket
fills that gap.

## What it should do

Mirror the accepted sibling fixes (`B-niagara-save-no-disk-write` #2,
`B-create-level-saved-true-no-umap`): persist for real on the MetaSound
create path (route `save:true` through the in-tree real-save helper
`SaveLoadedAssetThrottled` instead of `McpSafeAssetSave`), then probe
on-disk presence (`IFileManager::FileSize(PackageFilename)`) and gate the
result via the shared `ShouldTreatAssetSaveAsSuccess` predicate — report a
`saved`/disk-presence field that is true only when the `.uasset` is
actually on disk, and a `pendingFlush:true` signal when it is dirty-only,
instead of an unqualified `existsAfter:true`. The shared no-op
`McpSafeAssetSave` (and its ~220 corruption-sensitive callers) stays
untouched.

## Evidence (this task — create_metasound_patch "GainStage_Patch")

Friction note verbatim: "create_metasound_patch's save:true did not
actually write the .uasset until a separate save_all (which did persist
the .uasset to disk)." The call log shows the patch was created, then
later `editor.save_all` reports "saved 1" — i.e. the create's own
`save:true` left the package dirty and only the explicit `save_all`
flushed it to disk. The `existsAfter:true` in the create response gave no
hint the asset was memory-only.

(Distinct from `B-metasound-patch-mutators-reject`, the judge-filed bug on
the same task: that is the mutators rejecting a Patch with a false
`ASSET_NOT_FOUND`; this is the *create* path's `save:true` not persisting
to disk — an orthogonal save-fidelity defect on the same handler family.)

## Additional affected method — `audio.authoring.create_metasound` (Source), cold-load-confirmed loss

The #2 fix rerouted only the two handlers in `MetaSoundPatchPresetHandler.cpp`. The **Source** create — `audio.authoring.create_metasound` (builds a `UMetaSoundSource`) — is a SEPARATE handler in `AudioAuthoringHandler.cpp` that still routes its save through the mark-dirty-only `McpSafeAssetSave` and is NOT touched by that fix, so it still loses the whole asset on cold load.

Guilty source (verbatim, verified in-tree):

- `Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp:836` — `McpSafeAssetSave(MetaSound);` inside the `create_metasound` handler (the `save` param read at `:814` is not even consulted), immediately followed by `AddAssetVerification(Result, MetaSound);` (`:841`) which sets `existsAfter:true` from the asset registry, not from disk.
- `Source/PinWright/Private/Utils/AssetUtils.cpp:214-226` — `McpSafeAssetSave` only `MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated(Asset)` then returns true; it never calls any package-save API.

### Cold-load repro (confirmed by a real editor restart)

A build-an-engine-rev-synth task built `/Game/Audio/MS_EngineRev` (a MetaSound Source): `create_metasound` (save=true) then 3 Float inputs (Frequency / Gain / Detune) then set defaults (120 / 0.8 / 5) then `remove_metasound_input Detune` (save=true) then `describe_metasound` confirmed exactly two user inputs in-session. Every call returned success. A CorruptionCheck cold restart then found the asset ENTIRELY ABSENT on disk:

- `editor.open_asset /Game/Audio/MS_EngineRev` -> `[ASSET_NOT_FOUND]`
- `asset.dump_folder /Game/Audio` -> `assetCount: 0`
- no `MS_EngineRev.uasset` anywhere under `Content/Audio` on disk

The save reported success but never flushed — a no-disk-write persistence loss identical in shape to the patch/preset defect this ticket already covers, on the same `McpSafeAssetSave` helper. The subsequent input-mutation verbs on the task operated in-memory on the same never-persisted package, so nothing they did survives either — but the create is the root loss (the whole asset never lands on disk).

Fix must extend the #2 real-save reroute to `create_metasound` (Source) in `AudioAuthoringHandler.cpp` (route `save` through `SaveAssetToDiskReportingPresence` / `SaveLoadedAssetThrottled(bForce=true)` — a fresh `UMetaSoundSource` is not a Blueprint/SCS asset, so the bulkdata-corruption vector that pins `McpSafeAssetSave` on Blueprint edits does not apply), and report an honest `saved` / `pendingFlush` instead of an unqualified `existsAfter:true`.

severity rationale (elevated from Medium): impact=corruption/silent-persistence-loss × reach=every-session -> Critical. The prior Medium reflected the patch/preset "workaround = save_all" framing; the cold-load confirmation on the Source create proves genuine silent asset loss with no signal.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS/save-fidelity friction surfaced by the `audio.authoring.create_metasound_patch` "GainStage_Patch" fuzz task (outcome tool_bug; this save defect is orthogonal to the judge-filed mutator-cast bug). `create_metasound_patch`/`create_metasound_preset` default `save:true` to `McpSafeAssetSave(Patch)` (`MetaSoundPatchPresetHandler.cpp:90-93` / `:217-220`), which only `MarkPackageDirty()` + `AssetCreated()` and never writes the `.uasset` — but the response sets `existsAfter:true` (registry, not disk), implying persistence. The user observed the patch was not on disk until a separate `editor.save_all` ("saved 1") flushed it; the create's own `save:true` gave no hint the asset was memory-only. Same root cause/code as `B-niagara-save-no-disk-write`, which explicitly names the MetaSound path as affected but scopes its fix to niagara only — no MetaSound save ticket exists, so this fills the gap. Proposes the accepted sibling pattern: route MetaSound create `save:true` through the real-save helper `SaveLoadedAssetThrottled`, probe disk presence, and gate via `ShouldTreatAssetSaveAsSuccess` with a `pendingFlush:true` signal when dirty-only — leaving the shared corruption-sensitive `McpSafeAssetSave` untouched.
- `#2-route-create-save-through-disk-write` `IN-REVIEW` developer — Fixed both MetaSound create handlers in `MetaSoundPatchPresetHandler.cpp`: `create_metasound_patch` (was `:90-93`) and `create_metasound_preset` (was `:217-220`) now route `save:true` through the in-tree real-save helper `SaveAssetToDiskReportingPresence(Asset, /*bForce=*/true)` (which wraps `SaveLoadedAssetThrottled` -> `UEditorAssetLibrary::SaveLoadedAsset`, probes `IFileManager::FileSize` for on-disk presence, and gates via `ShouldTreatAssetSaveAsSuccess`) instead of the mark-dirty-only `McpSafeAssetSave`. Each response now carries honest `saveRequested`/`saved` booleans plus `pendingFlush:true` when the asset is dirty-only (mirrors the accepted `niagara.create_*` and `create_level` fixes). The shared corruption-sensitive `McpSafeAssetSave` and its ~220 other callers are untouched. Considered the adversarial suggestion to scope down to a `pendingFlush`-only honesty signal (relying on the existing generic `asset.save`), but a signal-only change leaves `save:true` silently not persisting — strictly worse than the sibling-accepted real-save, and the per-handler real-save reuses the centralized turnkey helper rather than reinventing per-site, so kept the real-save scope. Regression test: added `FMetaSoundCreateSaveWritesToDiskTest` (`PinWright.Assets.MetaSoundCreateSaveWritesToDisk`) to `Tests/Assets/TestMetaSoundPatchPreset.cpp` — factory-creates a real MetaSound patch, asserts no `.uasset` on disk pre-save, then drives the production helper `SaveAssetToDiskReportingPresence` and asserts the `.uasset` genuinely lands on disk (`IFileManager::FileSize >= 0`, reported `OutSize > 0`); reverting either handler to `McpSafeAssetSave` (mark-dirty only, no file written) fails the disk-presence assertions.
- `#3-additional-cold-load` `IN-REVIEW` reporter — Additional evidence (SYMPTOM-FAMILY, new method + new angle): the #2 fix rerouted only the two `MetaSoundPatchPresetHandler.cpp` handlers; the **Source** create `audio.authoring.create_metasound` (`AudioAuthoringHandler.cpp:836`) still calls the mark-dirty-only `McpSafeAssetSave` (`AssetUtils.cpp:214-226`) and reports `existsAfter:true`, so it STILL loses the asset on cold load. A CorruptionCheck cold restart of a build-an-engine-rev-synth task (`create_metasound MS_EngineRev` save=true -> 3 Float inputs with defaults -> `remove_metasound_input Detune` -> `describe_metasound` confirmed 2 inputs in-session) found `/Game/Audio/MS_EngineRev` ENTIRELY ABSENT on disk: `editor.open_asset` -> `[ASSET_NOT_FOUND]`, `asset.dump_folder /Game/Audio` -> assetCount 0, no `MS_EngineRev.uasset` under `Content/Audio`. Added `create_metasound` (Source) to the affected-methods list; bumped `encounters` -> 2. Severity raised Medium -> Critical: the cold-load confirmation proves silent persistence loss (asset corruption per rubric), not a workaround-covered Medium. Fix must extend the #2 real-save reroute to the Source create in `AudioAuthoringHandler.cpp`.
- `#4-extends-to-every-mutator-not-just-create` `IN-REVIEW` reporter — Still reproducing on UE 5.8 / EAContentExamples58 (gateway 27145), and the scope is **wider than the create verbs this ticket is named for**: on the MetaSound *Source* path every mutator ignores `save` too. Building `/Game/FPS/Audio/MetaSounds/MS_Fire_AR` (5 Wave Players, 3 RandomFloat, a Subtract, a 5-in mono mixer, 22 edges) I called, all with an explicit `save: true` where the verb documents one: `create_metasound`, `add_metasound_input`, `add_metasound_node` x9, `set_metasound_node_input_default` x12, `connect_metasound_nodes` x22. **Every single response carried `existsOnDisk: false, pendingSave: true`**, and no `.uasset` existed under `Content/FPS/Audio/MetaSounds/` at any point. `compile_metasound` is the sharpest case: it answered `compiled:true, valid:true, diagnostics:[]` alongside `saveRequested:false, markedForSave:false, saved:false` — a successful compile of a graph that exists only in memory, which is the most convincing possible false-success because the one field an author checks (`valid`) is genuinely true. Only an explicit `asset.save {force:true}` wrote it: `saved:true, sizeBytes:127195`, confirmed on disk by `ls` (127195 bytes, mtime current) and by `grep -a` finding all five bound SoundWave names plus the `Indoor` graph input in the package bytes. Two things make this worse than an ergonomic nit in a shared editor. First, `set_metasound_node_input_default` returns `readBack:true` with the stored literal echoed, which reads as confirmation while the package is still unwritten — the exact in-memory read-back that `CLAUDE.md` warns is worthless. Second, this same editor was killed by an unrelated `level.load` fatal earlier in this very session; had the graph been built before that crash rather than after, all 44 authoring calls would have been lost with no error ever having been returned. Suggested addition to the `#2`/`#3` fix scope: route `save` through the disk write on the Source mutators in `AudioAuthoringHandler.cpp` as well, not only the create verbs, or — if deferred saving is deliberate for batching — stop accepting a `save` parameter that does nothing and say on each method page that `asset.save` is mandatory. Workaround in use: pass `save:false` everywhere and finish every MetaSound with `asset.save {force:true}` plus an `ls` and a `grep -a` of the package bytes.
