---
id: B-asset-dump-texture2d-duplicate-sidecars
title: "asset.dump emits overlapping texture.json + texture_2d.json for every Texture2D"
status: DONE
severity: Low
category: bug
tags: [asset-dump, texture, sidecar-duplication]
---

# asset.dump emits overlapping texture.json + texture_2d.json for every Texture2D

After `B-asset-dump-non-texture2d-no-native-summary` (DONE) added a generic
`texture.json` sidecar for all `UTexture` subclasses while preserving the
existing `texture_2d.json` sidecar for `UTexture2D` "backward compatibility",
every `UTexture2D` now ships **two overlapping native-summary sidecars** whose
contents are ~85% identical. `texture_2d.json` is a strict subset of
`texture.json` for the Texture2D case.

The dispatch is unconditional in `AssetDumpHandler.cpp:653-659`:

```cpp
else if (UTexture* Texture = Cast<UTexture>(Asset))
{
    AddJsonFile(DumpFileNames::Texture, TextureDumpBuilder::BuildTextureJson(Texture));
    if (UTexture2D* Texture2D = Cast<UTexture2D>(Texture))
    {
        AddJsonFile(DumpFileNames::Texture2D, Texture2DDumpBuilder::BuildTexture2DJson(Texture2D));
    }
}
```

Both handlers run for every Texture2D — no "specialization wins" check.

**Scope:** every `UTexture2D` asset in any future `asset.dump_folder` sweep.
The bug is latent in the current dump cache at
`C:\Unity\unreal-fpv-pluginwork\.editor-automation\asset-dumps\` because that
cache predates the `B-asset-dump-non-texture2d-no-native-summary` fix and
still holds 8,737 `texture_2d.json` files with zero `texture.json` companions;
re-running `asset.dump_folder` on `/App` or `/Game` will materialize the
duplicate pairs en masse.

**Exact key-set diff (from `TextureDumpBuilder.cpp` vs `Texture2DDumpBuilder.cpp`):**

`texture.json` keys (BuildTextureJson, 12 top-level keys):
`kind`, `textureClass`, `size{x,y,z}`, `arraySize`, `pixelFormat`,
`compressionSettings`, `lodGroup`, `srgb`, `mipGenSettings`, `neverStream`,
`source{x,y,slices,format}`.

`texture_2d.json` keys (BuildTexture2DJson, 8 top-level keys):
`size{x,y}`, `pixelFormat`, `compressionSettings`, `lodGroup`, `srgb`,
`mipGenSettings`, `neverStream`, `source{x,y,format}`.

Set-difference (keys only in `texture.json`, never in `texture_2d.json`):
`kind`, `textureClass`, `arraySize`, `size.z`, `source.slices`.

For a Texture2D specifically, all five "extra" keys are degenerate constants
(`kind="Texture2D"`, `textureClass="TEXTURECLASS_TwoD"`, `arraySize=1`,
`size.z=0`, `source.slices=1`), so `texture_2d.json` carries no information
that `texture.json` doesn't already carry for Texture2D inputs. The duplicate
sidecar is pure noise on disk and a consumer-confusion footgun ("which one is
canonical?").

**Root cause:** the fix in `B-asset-dump-non-texture2d-no-native-summary`
preserved `texture_2d.json` to avoid breaking existing consumers, then added
`texture.json` as a parallel emission rather than as a replacement. The
dispatcher's `if/else if` ladder at `AssetDumpHandler.cpp:645-676` already
selects exactly one branch per asset, but inside the `UTexture` branch both
sidecars are emitted unconditionally.

**Fix (proposed):** pick one of:

1. **Drop `texture_2d.json` (preferred).** `texture.json` is a strict superset
   for Texture2D and the per-type registry pattern from
   `E-asset-dump-ir-sidecar-registry` makes a single canonical filename the
   simpler contract. Update `LoadBaselineDumpFiles`, the registered IR
   sidecar list, `TestAssetDumpNativeSummaries.cpp` (the
   `texture_2d.json`-named test expectations at lines 100/118/123 become
   `texture.json` assertions), and the wiki asset.md / asset-audit.md /
   texture.md references. The Texture2D dump cache (8,737 files) will
   regenerate clean on next `asset.dump_folder` sweep.

2. **Document an explicit base-vs-leaf convention** in
   `docs/wiki/asset.md`: "the most-specific sidecar wins; `texture.json` is
   present iff no specialized sidecar exists." Then guard the inner
   `Texture2D` block so it skips `texture.json` when `texture_2d.json` will
   write, by inverting the current code:
   ```cpp
   else if (UTexture* Texture = Cast<UTexture>(Asset))
   {
       if (UTexture2D* Texture2D = Cast<UTexture2D>(Texture))
       {
           AddJsonFile(DumpFileNames::Texture2D, Texture2DDumpBuilder::BuildTexture2DJson(Texture2D));
       }
       else
       {
           AddJsonFile(DumpFileNames::Texture, TextureDumpBuilder::BuildTextureJson(Texture));
       }
   }
   ```
   This trades the duplication for a per-class lookup rule that consumers
   then have to encode. Less attractive than option 1 unless there's a real
   consumer relying on the Texture2D-specific shape.

Option 1 is recommended because `texture.json` already contains everything
`texture_2d.json` carries and the registry pattern favors one canonical
filename per asset class.

## History
- `#1-initial-repro` `OPEN` reporter — `AssetDumpHandler.cpp:653-659` writes both `texture.json` and `texture_2d.json` for every UTexture2D. `BuildTexture2DJson` is a strict-subset of `BuildTextureJson` (8 top-level keys vs 12; `texture_2d.json` omits `kind`/`textureClass`/`arraySize`/`size.z`/`source.slices`, all degenerate constants for the Texture2D case). Existing dump cache at `.editor-automation/asset-dumps/` predates the `B-asset-dump-non-texture2d-no-native-summary` fix and contains 8,737 `texture_2d.json` files with zero `texture.json` companions; re-sweeping will materialize duplicate pairs across `App/` and `Game/`. Recommended fix: drop `texture_2d.json`, keep `texture.json` as the single canonical native-summary sidecar for all UTexture subclasses, and update the IR sidecar registry, native-summary tests (`TestAssetDumpNativeSummaries.cpp:100,118,123`), and wiki references.
- `#2-drop-texture-2d-json` `IN-REVIEW` developer — Dropped texture_2d.json emission and Texture2DDumpBuilder; collapsed UTexture branch in AssetDumpHandler.cpp to single texture.json AddJsonFile; removed DumpFileNames::Texture2D constant and FixedCanonical entry. Retargeted FAssetDumpTexture2DSummaryTest to assert texture.json singleton + kind=='Texture2D' + size keys; added negative assertion against legacy texture_2d.json. Wiki references in asset.md, asset-audit.md, and E-dump-rpc-parity.md updated.
- `#3-verify-fix-source` `DONE` tester — Verified by source inspection (editor offline at port 19880, live MCP unavailable). AssetDumpHandler.cpp:738-741 UTexture branch now contains a single AddJsonFile(DumpFileNames::Texture, ...) call with no nested UTexture2D cast/emit. Texture2DDumpBuilder.cpp/h files are absent from Handlers/Asset/ (only TextureDumpBuilder.* remains). FixedCanonical list at line 841 carries DumpFileNames::Texture and no Texture2D entry. TestAssetDumpNativeSummaries.cpp:118-123 asserts texture.json present and texture_2d.json absent. Remaining `texture_2d`/`Texture2DDumpBuilder` matches are a stale comment in PropertyUtils.cpp:1885 plus board/wiki docs — not active code paths.
- `#4-verify-live-dump` `DONE` tester — Live MCP verification (prior #3 was source-only, which the verify-protocol disallows). Ran `asset.dump` on two Texture2D assets: `/Game/UI/Hud/Art/T_UI_Effects_TriHalftone_Mask` and `/Game/UI/Menu/Art/T_UI_MapTile_Expanse`. Both responses' `writtenPaths` contain exactly one native-summary sidecar — `texture.json` — and zero `texture_2d.json` entries. `texture.json` content carries the unified-schema keys (`kind: "Texture2D"`, `textureClass: "TwoD"`, `arraySize`, `size.z`, `source.slices`) plus all subset keys, confirming the dispatcher no longer double-emits and `texture.json` is now the single canonical Texture2D sidecar.
- `#5-repoint-citations-after-property-utils-split` `DONE` reporter — Citation maintenance only; **no behavioural claim changes and the status is untouched**. `Utils/PropertyUtils.cpp` was split into `PropertyExport.cpp` / `PropertyImport.cpp` / `PropertyInspection.cpp` / `PropertyDiff.cpp` (`PropertyUtils.h` survives only as a deprecated umbrella forwarder), so this ticket's single `PropertyUtils.cpp` citation was an unresolvable path, not a stale line number. It sits inside `#3`, which is left verbatim per the append-only rule, so the map is recorded here: **`PropertyUtils.cpp:1885` → `Source/PinWright/Private/Utils/PropertyExport.cpp:1177`**, in the comment block `:1174-1178` above `IsTextureSourceNoisyProperty` (`:1179`). Verified at plugin HEAD `ef8a1f1b`. **The finding `#3` recorded is still open, and now sharper.** The comment survives and still names `Texture2DDumpBuilder`, which `#2` deleted — a whole-`Source/` grep returns **exactly one hit** for `Texture2DDumpBuilder`, that comment, so `PropertyExport.cpp:1177` is the last reference in shipped code to a class and a pair of files that no longer exist. The lowercase `texture_2d` token that `#3` also flagged is **no longer in the comment**; its only remaining occurrences in `Source/` are `Tests/Utility/TestAssetDumpNativeSummaries.cpp:141`, `:163` and `:164`, which are the deliberate negative assertions `#2` added and must stay. So the residue is one stale comment line, wanting a source commit rather than a board one — the same shape as the stale in-code line references noted on `B-asset-dump-tmap-struct-key-mangled` `#4`, in the same file.
