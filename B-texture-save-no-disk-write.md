---
id: B-texture-save-no-disk-write
title: "texture.* pixel-mutation / setter / create verbs (save:true) report success but McpSafeAssetSave only marks dirty — nothing reaches disk, so texture edits vanish on cold restart"
status: IN-REVIEW
severity: Critical
category: bug
tags: [texture, texture-sharpen, texture-blur, no-disk-write, mcp-safe-asset-save, false-success, silent-failure, persistence, cold-load]
encounters: 1
lastSeen: 2026-07-11T12:12:14.1030419+03:00
---

# texture.* save:true writes report success but never reach disk — texture edits are lost on cold restart

The entire `texture` write surface routes its `save:true` path through the shared
mark-dirty no-op `McpSafeAssetSave`. Every pixel-mutation verb, every settings
setter, and every create verb takes a `save` param defaulting to `true`
(documented literally as "Save the asset to disk"), then returns an unqualified
`success:true` — but **no `.uasset` is ever written**. The package is only marked
dirty + registered with the asset registry, so the in-session readback
(`get_pixel_stats`, `describe`) reflects the change, but after the editor closes
(or a fuzz `git reset --hard`) the edit is gone. Cold restart proves it (below).

This is the **texture-namespace sibling** of the established no-disk-write family
(`B-niagara-save-no-disk-write`, `B-audio-create-save-no-disk-write`,
`B-metasound-create-save-no-disk-write`, `B-material-authoring-save-no-disk-write`,
`B-geometry-convert-static-mesh-no-disk-write`, `B-geometry-generate-lods-no-disk-write`,
`B-pose-search-create-save-no-disk-write`, `B-sequencer-create-save-no-disk-write`,
`B-create-level-saved-true-no-umap`) — same shared `McpSafeAssetSave` root cause,
a distinct un-fixed namespace. Those siblings each scope their fix to their own
handler and leave the texture path unfixed; there is no texture save-no-disk-write
ticket, so this fills the gap.

For the pixel-mutation verbs it is **worse than the create-family siblings**: the
sharpen/blur/adjust responses do not even carry a `saved` / `existsAfter` /
`sizeBytes` field that a caller could second-guess — they report a bare
`success:true` + `message` ("Sharpen applied (amount: N.NN)"), so the `save:true`
default and the unqualified success are the only persistence signal the agent gets,
and both are lies.

## Root cause (verified in source)

`Plugins/PinWright/Source/PinWright/Private/Utils/AssetUtils.cpp:220-232` —
`McpSafeAssetSave` never calls any package-save API. It only marks the package
dirty + notifies the registry, then unconditionally returns `true`:

```cpp
bool McpSafeAssetSave(UObject* Asset)
{
    if (!Asset)
        return false;

    // UE 5.7+ Fix: Do not immediately save newly created assets to disk.
    // Saving immediately causes bulkdata corruption and crashes.
    // Instead, mark the package dirty and notify the asset registry.
    Asset->MarkPackageDirty();
    FAssetRegistryModule::AssetCreated(Asset);

    return true;
}
```

The `texture.sharpen` handler
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/TextureHandler.cpp:1996-2008`)
mutates the source mip in place, then routes the save through this no-op and reports
unqualified success:

```cpp
Texture->Source.UnlockMip(0);
Texture->UpdateResource();
Texture->MarkPackageDirty();

if (bSave)
{
    McpSafeAssetSave(Texture);          // :2002 — mark-dirty only, no disk write
}

Response->SetBoolField(TEXT("success"), true);   // :2005 — unqualified success
Response->SetStringField(TEXT("message"), FString::Printf(TEXT("Sharpen applied (amount: %.2f)"), Amount));
```

The file even documents its intent to persist (`TextureHandler.cpp:77-79`): "Use
McpSafeAssetSave(Asset) ... for saving textures. McpSafeAssetSave marks the package
dirty and notifies the asset registry safely for UE 5.7+." — i.e. the code believes
this saves textures; it does not.

The codebase already knows this helper does not persist:
`ShouldTreatAssetSaveAsSuccess` (`AssetUtils.cpp:460-468`) exists precisely because
"the shared mark-dirty helper McpSafeAssetSave returns true without writing ... so
the report cannot trust its return alone — the on-disk file is the only honest
persistence signal." The texture handlers never call that predicate.

## Affected methods (all route save:true through McpSafeAssetSave in TextureHandler.cpp)

Cold-load CONFIRMED (this iteration): `texture.sharpen` (:2002).

Source-confirmed (identical `if (bSave) McpSafeAssetSave(...)` save site, same defect):

- Pixel-mutation (mutate an existing on-disk texture; edit lost on restart):
  `texture.blur` (:1928), `texture.invert` (:1637), `texture.desaturate` (:1757),
  `texture.adjust_levels` (:1849), `texture.adjust_curves` (:2386),
  `texture.resize_texture` (:1523).
- Settings setters (setting change lost on restart):
  `texture.set_compression_settings` (:1076), `texture.set_texture_group` (:1143),
  `texture.set_lod_bias` (:1196), `texture.configure_virtual_texture` (:1249),
  `texture.set_streaming_priority` (:1301), `texture.set_texture_filter` (:2584),
  `texture.set_texture_wrap` (:2622).
- Create-from-scratch (new asset never lands on disk — ASSET_NOT_FOUND on cold load,
  exactly the niagara/material create-family shape):
  `texture.create_noise_texture` (:305), `texture.create_gradient_texture` (:458),
  `texture.create_pattern_texture` (:627), `texture.create_normal_from_height` (:876),
  `texture.create_ao_from_mesh` (:1006), `texture.channel_pack` (:2097),
  `texture.combine_textures` (:2200), `texture.channel_extract` (:2508),
  `texture.create_render_target` (:2688).

## Cold-load repro (confirmed by a real editor restart — the replay-confirmation)

A texture-artist "make these read crisper" task sharpened two real project textures
(`save:true`, the default):

1. `texture.sharpen {assetPath:"/Game/Global/Textures/T_FloorMarble_D", amount:1.5}` → `success:true`.
2. `texture.sharpen {assetPath:"/Game/Global/Textures/T_Fabric_Weave", amount:3.5}` → `success:true`.

Warm-session `texture.get_pixel_stats` (mip0) confirmed both mutations took effect in
memory: FloorMarble content hash 51ccc2e1 → d9d693bc; Fabric d5b27f57 → f0d13026.
`texture.describe` reported structure unchanged (FloorMarble 2048x2048 PF_DXT1;
Fabric 1024x1024 PF_DXT1).

A `CorruptionCheck` cold restart (fresh detached headless editor, no baseline restore)
then re-read both textures. Both cold-open and describe cleanly — **structure fully
intact, no crash, no ASSET_NOT_FOUND** — but the mutation is ABSENT:

- Objective proof independent of pixel interpretation: both `.uasset` files are
  git-clean, byte-identical to the committed baseline, mtime 2026-06-13 — untouched
  this iteration even though the attempt ran 2026-07-11. A real edit+save would have
  rewritten the file (new mtime + git-modified).
- Corroborating: the deterministic cold source-mip readback (stable across repeated
  cold reads) reverts to the baseline pixels: FloorMarble cold hash 51ccc2e1 (the
  pre-sharpen baseline, not the warm post-sharpen d9d693bc); Fabric cold hash d5b27f57
  (baseline, not the warm f0d13026).

So the warm save reported success but never wrote the package — a no-disk-write / silent
false-success. Editor stayed healthy throughout (cold outcome load_failed, not a crash).

severity rationale: impact=corruption × reach=every-session -> Critical

## What it should do

A `save:true` texture write must persist to disk for real (or, if deferred, the response
must say so — never an unqualified `success:true`). Mirror the accepted sibling fixes
(`B-material-authoring-save-no-disk-write` #2, `B-niagara-save-no-disk-write` #2):

- Route the texture save:true path through the in-tree real-save helper
  (`SaveAssetToDiskReportingPresence` / `SaveLoadedAssetThrottled` in
  `Utils/AssetUtils.cpp`, which calls `UEditorAssetLibrary::SaveLoadedAsset` behind the
  integrity gate) instead of the mark-dirty `McpSafeAssetSave`. Texture2D assets are
  non-Blueprint / non-SCS, so the bulkdata-corruption vector that forced
  `McpSafeAssetSave` on Blueprint edits (`B-bp-saved-state-corruption-mcp-edits`) does
  not apply — exactly the reasoning the material-authoring fix used.
- After the save, probe on-disk presence (`IFileManager::FileSize(PackageFilename)`) and
  gate the result through the shared `ShouldTreatAssetSaveAsSuccess` predicate: report a
  `saved` field that is `true` only when the `.uasset` is actually on disk, plus a
  `pendingFlush:true` signal when the package is dirty-only — instead of the current bare
  `success:true`.

## Workaround

After the texture create/mutate/set calls, run `editor.save_all` to flush the dirty
texture packages to disk before the editor closes or any `git reset --hard`. Nothing in
the texture responses signals this is required.

## Cross-ref

- `B-niagara-save-no-disk-write`, `B-material-authoring-save-no-disk-write`,
  `B-audio-create-save-no-disk-write`, `B-metasound-create-save-no-disk-write` (IN-REVIEW)
  — same `McpSafeAssetSave` / per-namespace-save-helper root cause, each fixed in its own
  handler; texture is the un-fixed namespace here.
- `B-texture-create-placeholder-fake-success` (existing texture false-success ticket) —
  distinct: that is about a placeholder-creation path, not the McpSafeAssetSave disk-write
  defect.

## History
- `#1-initial-repro` `OPEN` reporter — Cold-load-confirmed texture persistence loss on `texture.sharpen`. A texture-artist task sharpened two real project textures `save:true` (T_FloorMarble_D amount=1.5, T_Fabric_Weave amount=3.5); both returned `success:true` and warm `texture.get_pixel_stats` (mip0) confirmed the mutation in memory (FloorMarble 51ccc2e1->d9d693bc, Fabric d5b27f57->f0d13026), `texture.describe` unchanged structure. A real cold editor restart (fresh headless, no baseline restore) then read both back as the UN-mutated baseline: both `.uasset` files git-clean, byte-identical to baseline, mtime 2026-06-13 (untouched despite the attempt running 2026-07-11), and the deterministic cold source-mip readback reverted to baseline hashes (FloorMarble 51ccc2e1, Fabric d5b27f57). Editor healthy, no crash, no ASSET_NOT_FOUND — pure no-disk-write / silent false-success. Root cause (source-verified): `McpSafeAssetSave` (`Utils/AssetUtils.cpp:220-232`) is a mark-dirty no-op (`MarkPackageDirty()` + `FAssetRegistryModule::AssetCreated()` then `return true`, no package-save API), and `texture.sharpen` (`Handlers/Material/TextureHandler.cpp:2002`) routes its `if (bSave)` through it, then reports unqualified `success:true` (:2005) with no `saved`/`pendingFlush` field. DEFECT-FAMILY probe: every texture write verb shares the identical `if (bSave) McpSafeAssetSave(...)` save site in TextureHandler.cpp — pixel-mutation (blur/invert/desaturate/adjust_levels/adjust_curves/resize), setters (set_compression_settings/set_texture_group/set_lod_bias/configure_virtual_texture/set_streaming_priority/set_texture_filter/set_texture_wrap), and create (create_noise/gradient/pattern/normal_from_height/ao_from_mesh/channel_pack/combine_textures/channel_extract/create_render_target) — so filed at the texture-namespace family level. DEDUP: ripgrep over OPEN + closed found no texture save-no-disk-write ticket (the niagara ticket only lists "textures" in passing among the ~220 shared callers); distinct from `B-texture-create-placeholder-fake-success` (placeholder path, not the disk-write defect). Same family/root cause as the accepted sibling per-namespace fixes; fix belongs in the texture save path (real-save helper + `ShouldTreatAssetSaveAsSuccess` disk gate), leaving the shared corruption-driven no-op helper untouched.
- `#2-accept-fix` `IN-REVIEW` developer — GO (Critical). Root cause confirmed in current source: `McpSafeAssetSave` (`Utils/AssetUtils.cpp:220-232`) marks the package dirty + notifies the registry and returns true WITHOUT writing a `.uasset`; every texture `save:true` verb routed through it. Fix (one file, `Handlers/Material/TextureHandler.cpp`): added file-local `McpSaveTextureToDisk(Response, Texture, bSave)` that notifies the registry then force-saves for real via `SaveAssetToDiskReportingPresence(Asset, bForce=true)` (→ `UEditorAssetLibrary::SaveLoadedAsset` behind the `ShouldTreatAssetSaveAsSuccess` on-disk `FileSize` gate) and records the honest `saved`/`saveRequested`/`pendingFlush` report via `AddAssetSaveReport`; replaced ALL 23 `McpSafeAssetSave` save sites (10 creates incl. resize/channel_extract/channel_pack/combine/create_render_target, 7 setters, 6 pixel-mutations) with it — zero `McpSafeAssetSave` code calls remain. Shared corruption-driven `McpSafeAssetSave` no-op left untouched for the Blueprint bulkdata vector (textures are non-Blueprint; `SaveLoadedAssetThrottled` only Blueprint-gates at `:688-702`). Regression test: adopted red `PinWright.texture.create_noise_texture.SaveWritesToDisk` (`Tests/Material/TestTextureCreateSaveWritesToDisk.cpp`) — failing pre-fix (no disk file, no `saved`), now `Result={Success}`. Plugin compiles clean; scoped test green (red→green differential).
