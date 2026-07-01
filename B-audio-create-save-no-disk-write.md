---
id: B-audio-create-save-no-disk-write
title: "audio.authoring create_sound_class / create_sound_mix / create_reverb_effect (save:true) report existsAfter:true but SaveAudioAsset only marks dirty — nothing reaches disk, so the assets vanish on cold restart"
status: IN-REVIEW
severity: Critical
category: bug
tags: [audio, soundclass, soundmix, reverbeffect, save, save-audio-asset, no-disk-write, silent-failure, false-success, cold-load, persistence]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# audio.authoring create_* claims success but writes nothing to disk — assets are lost on restart

The `audio.authoring` create handlers — `create_sound_class`, `create_sound_mix`,
`create_reverb_effect` (and, via the same helper, `create_sound_cue`,
`create_sound_attenuation`, `create_sound_wave`) — take a `save` param defaulting
to `true`, return an `AddAssetVerification` block with `existsAfter:true`, and so
make a caller reasonably believe the new `.uasset` is on disk. It is not. With
`save:true` the handler only **marks the package dirty + notifies the asset
registry**; the file is never written. The asset exists only in memory + the
registry (hence `existsAfter:true` and the in-session `get_audio_info` /
`describe_sound_mix` readbacks work), but after the editor closes (or a fuzz
`git reset --hard`) **the asset is gone**, with no error ever surfaced.

This is **cold-load-confirmed asset loss**: a build-an-audio-mixing-foundation
task created five assets — `/Game/Audio/Classes/SFX`, `/Game/Audio/Classes/Weapons`,
`/Game/Audio/Classes/Footsteps`, `/Game/Audio/Mixes/CombatDuck`,
`/Game/Audio/Reverb/CombatHall` — every create returned success and every in-session
verify (`get_audio_info`, `describe_sound_mix`) confirmed the hierarchy, the
Weapons vol 1.1 bump, the CombatDuck adjusters (Footsteps 0.4 / Weapons 1.2), and
the ReverbEffect. On a cold editor restart (no baseline restore) **all five failed
`editor.open_asset` with `[ASSET_NOT_FOUND]`**, and `Content/Audio/` is entirely
absent on disk — none of the packages were ever persisted. The whole "set up a
reusable audio-mixing foundation" workflow is a silent no-op as far as durable
state is concerned, even though every call reported success.

## Root cause (verified in source)

`Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`

`SaveAudioAsset` (`:190-200`) is a **mark-dirty no-op** — despite the name it never
calls any package-save API:

```cpp
// Save asset - mark dirty and notify registry (avoids modal dialogs in UE 5.7+)
static bool SaveAudioAsset(UObject* Asset, bool bShouldSave)
{
    if (!bShouldSave || !Asset)
    {
        return true;
    }

    Asset->MarkPackageDirty();
    FAssetRegistryModule::AssetCreated(Asset);
    return true;
}
```

There is no `UPackage::Save` / `UEditorAssetLibrary::SaveLoadedAsset` /
`SavePackage` anywhere in it. Every audio create handler routes its `save:true`
through this no-op and then reports `existsAfter:true`:

- `create_sound_class` (`:1347` `SaveAudioAsset(NewClass, bSave)` → `AddAssetVerification` `:1352`)
- `create_sound_mix` (`:1589` `SaveAudioAsset(NewMix, bSave)`)
- `create_reverb_effect` (`:2289` `SaveAudioAsset(NewEffect, bSave)`)
- and the rest of the namespace (`create_sound_cue` `:453`, `create_sound_attenuation`
  `:1836`, `create_sound_wave` `:2115`, plus the reparent/property/modifier edit
  paths that re-dirty an existing asset).

So the asset is registered (`existsAfter:true`, `get_audio_info` /
`describe_sound_mix` read it) but never lands on disk. The `save:true` default +
`existsAfter:true` response together imply a persistence that did not happen, and
the response carries no `pendingFlush` / disk-presence signal to say otherwise.
`editor.save_all` is the only thing that actually flushes these dirty packages, and
nothing tells the agent that is required.

This is the **audio-authoring analog** of `B-niagara-save-no-disk-write` (niagara
path) and `B-metasound-create-save-no-disk-write` (metasound create path), but on a
**distinct third no-op helper**: `SaveAudioAsset`, not the shared `McpSafeAssetSave`.
Both sibling tickets scope their fix to their own namespace and leave the audio
create path unfixed — there is no audio-create save ticket, so this fills the gap.
(The metasound *creates* inside this same file — `:764`, `:794`, … — already use
`McpSafeAssetSave` and are covered by `B-metasound-create-save-no-disk-write`; this
ticket is the non-metasound audio assets that go through `SaveAudioAsset`.)

## Cold-load repro (confirmed by a real editor restart)

A `CorruptionCheck` cold-restart (quit via MCP `editor.quit` discard=true,
relaunch headless WITHOUT restoring the baseline) reopened the five saved assets:

```
editor.open_asset /Game/Audio/Classes/SFX       → open_ok:false, [ASSET_NOT_FOUND] Asset not found
editor.open_asset /Game/Audio/Classes/Weapons   → open_ok:false, [ASSET_NOT_FOUND] Asset not found
editor.open_asset /Game/Audio/Classes/Footsteps → open_ok:false, [ASSET_NOT_FOUND] Asset not found
editor.open_asset /Game/Audio/Mixes/CombatDuck   → open_ok:false, [ASSET_NOT_FOUND] Asset not found
editor.open_asset /Game/Audio/Reverb/CombatHall  → open_ok:false, [ASSET_NOT_FOUND] Asset not found
```

The editor stayed fully responsive (each `open_asset` returned a clean structured
`ASSET_NOT_FOUND`; the namespace index returned afterward) — not an editor crash.
Disk confirms the cause: the entire `Content/Audio/` tree is absent on disk
(`Glob Content/Audio/**` → no files), so none of these packages were ever persisted
by the create calls. They were SoundClass / SoundMix / ReverbEffect (non-Blueprint,
so the compile readback is N/A). Net: the assets the create calls reported saving
(with `existsAfter:true`) did not survive cold load because they were never written
to disk — they vanish entirely on restart.

## What it should do

A `save:true` create must persist to disk for real (or, if deferred, the response
must say so — never an unqualified `existsAfter:true`). Mirror the accepted sibling
fixes (`B-niagara-save-no-disk-write` #2, `B-create-level-saved-true-no-umap`):

- Route the audio create `save:true` path through the in-tree real-save helper
  `SaveLoadedAssetThrottled` (`Utils/AssetUtils.cpp`, which calls
  `UEditorAssetLibrary::SaveLoadedAsset` behind the Blueprint integrity gate)
  instead of the mark-dirty `SaveAudioAsset` — SoundClass/SoundMix/ReverbEffect are
  not Blueprint/SCS assets, so the corruption vector that forced `McpSafeAssetSave`
  on Blueprint edits (`B-bp-saved-state-corruption-mcp-edits`) does not apply here.
- After the save, probe on-disk presence (`IFileManager::FileSize(PackageFilename)`)
  and gate the result through the shared `ShouldTreatAssetSaveAsSuccess` predicate:
  report a `saved` / disk-presence field that is `true` only when the `.uasset` is
  actually on disk, and a `pendingFlush:true` signal when it is dirty-only — instead
  of an unqualified `existsAfter:true`. Keep the multi-asset reparent path
  (`SetSoundClassParentMaintainingChildren`, which dirties old/new parent + child)
  flushing every touched class.

## Workaround

After the create calls, run `editor.save_all` to flush the dirty packages to disk
before the editor closes or any `git reset --hard`. Nothing in the create responses
signals this is required.

## History
- `#1-initial-repro` `OPEN` reporter — COLD-LOAD-confirmed asset loss from a REALISM-mode audio-mixing-foundation task (SFX root SoundClass + Weapons/Footsteps children, Weapons vol 1.1, CombatDuck SoundMix with Footsteps-0.4/Weapons-1.2 adjusters, CombatHall ReverbEffect). Every `audio.authoring.create_*` returned success with `existsAfter:true`, and in-session `get_audio_info` / `describe_sound_mix` confirmed the full build. A real editor cold restart (MCP `editor.quit` discard=true, relaunch headless without baseline restore) then failed `editor.open_asset` on ALL FIVE assets (`/Game/Audio/Classes/SFX`, `/Game/Audio/Classes/Weapons`, `/Game/Audio/Classes/Footsteps`, `/Game/Audio/Mixes/CombatDuck`, `/Game/Audio/Reverb/CombatHall`) with `[ASSET_NOT_FOUND]`; the entire `Content/Audio/` tree is absent on disk — the packages were never persisted. Root cause: `SaveAudioAsset` (`AudioAuthoringHandler.cpp:190-200`) is a mark-dirty no-op (`MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()`, no package-save API), used by `create_sound_class` (`:1347`), `create_sound_mix` (`:1589`), `create_reverb_effect` (`:2289`) and the rest of the namespace, while each handler reports `existsAfter:true` from the registry (not disk). Same defect shape as `B-niagara-save-no-disk-write` / `B-metasound-create-save-no-disk-write` but on a distinct third no-op helper (`SaveAudioAsset`, not `McpSafeAssetSave`); both siblings scope their fix to their own namespace and leave this audio-create path unfixed. The cold restart IS the replay-confirmation (asset corruption / loss the save-time integrity gate let through). Dedup: ripgrep over OPEN + closed found no ticket on `SaveAudioAsset` / audio-create no-disk-write — the audio tickets that exist are about fade-param drop (`B-add-mix-modifier-fade-params-dropped`), parent/child link maintenance (`B-sound-class-parent-no-child-link`), readback thinness (`E-audio-get-info-soundclass-mix-readback-thin`), and seed `applyToChildren` divergence (`E-create-sound-mix-seed-applytochildren-false`) — none touch the disk-write defect.
- `#2-fix-real-disk-save` `IN-REVIEW` developer — Root-caused and fixed. `SaveAudioAsset` (`AudioAuthoringHandler.cpp`) was a mark-dirty no-op (`MarkPackageDirty` + `FAssetRegistryModule::AssetCreated`, no package-save); it now registers the asset then routes through the in-tree real-save helper `SaveAssetToDiskReportingPresence` (`Utils/AssetUtils.cpp`: forced `SaveLoadedAssetThrottled`/`UEditorAssetLibrary::SaveLoadedAsset` + `IFileManager::FileSize` disk probe, gated by `ShouldTreatAssetSaveAsSuccess`) with `bForce=true`, returning true only when the `.uasset` is actually on disk — the same pattern the niagara/metasound siblings shipped. This single file-local helper change persists every audio create/edit `save:true` path uniformly (creates, the SoundClass/Submix reparent paths, and the edit/set paths). SoundClass/SoundMix/ReverbEffect are non-Blueprint/non-SCS so the `McpSafeAssetSave` bulkdata-corruption deferral does not apply (the Blueprint integrity gate inside `SaveLoadedAssetThrottled` cannot trip for these). Added a new file-local `AddAudioSaveReport` that emits an honest `saved` (gated on disk presence) + `pendingFlush` signal instead of an unqualified `existsAfter:true`, wired into the five unconditional create handlers (`create_sound_cue`, `create_sound_class`, `create_sound_mix`, `create_sound_attenuation`, `create_reverb_effect`). Files: `Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`. Regression test: `Source/PinWright/Private/Tests/Assets/TestAudioCreateSaveWritesToDisk.cpp` drives the real `audio.authoring.create_sound_class` handler through the dispatcher and asserts (save:true) the `.uasset` genuinely lands on disk (`IFileManager::FileSize > 0`) and the response reports `saved:true`, plus (save:false) no file is written and `saved:false` — it fails if `SaveAudioAsset` is reverted to the mark-dirty-only body.
