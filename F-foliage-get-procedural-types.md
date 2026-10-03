---
id: F-foliage-get-procedural-types
title: "No verb reads back which UFoliageTypes an EXISTING UProceduralFoliageSpawner drives (foliage.create_procedural now echoes its own types, but a later tuning pass on a spawner it did not just create still keys on the positional _FT_<index> name); the property.get FoliageTypes[N].FoliageTypeObject route is untested and undocumented"
status: OPEN
severity: Low
category: feature
tags: [foliage, procedural-foliage, spawner, readback, missing-verb, foliage-type, reflection, naming-convention, vegetation, premise-corrected]
encounters: 1
costly: 1
lastSeen: 2026-08-29T18:00:00+05:00
rice: [1, 1, 0.8, 1]
priority: 7
---

# Reading an existing spawner's foliage types has no verb and no documented route

`foliage.create_procedural` now returns the types it built: one `foliage_types[]` row per type with
`index` and `asset_path` (`Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:2487-2490`,
attached `:2818-2819`). So the create call itself no longer forces a caller to guess names.

What is still missing is a readback for a spawner that already exists. None of the `foliage.*` verbs —
`paint` (`:665`), `remove` (`:1278`), `get_instances` (`:1390`), `add_type` (`:1651`),
`add_instances` (`:1858`), `create_procedural` (`:2183`) — reads a `UProceduralFoliageSpawner`'s
`FoliageTypes`. A later tuning pass (not holding the create response) falls back to the generated
asset name `<Name>_Spawner_FT_<index>` (`:2467-2468`), which is positional: re-running
`create_procedural` with reordered `foliageTypes` rebinds `_FT_0` to a different mesh, and a pass that
writes `Mesh` by name then edits the wrong asset with every response reporting success.

A generic route probably exists and has not been tried: `UProceduralFoliageSpawner::FoliageTypes` and
`FFoliageTypeObject::FoliageTypeObject` are both `EditAnywhere`
(`Engine/Source/Runtime/Foliage/Public/ProceduralFoliageSpawner.h:41-42`,
`FoliageTypeObject.h:45-46`), and `property.get` resolves array subscripts
(`docs/wiki-src/property.md:49`), so `property.get {objectPath: <spawner>, propertyName:
"FoliageTypes[0].FoliageTypeObject"}` should return the type. Python's natural spellings fail
(`FFoliageTypeObject` is not `BlueprintType`, so `to_dict()` is `{}`, and the member is
`foliage_type_object`, not `foliage_type`), which is why the earlier pass concluded it was
unreachable.

**Workaround:** keep the `foliage_types[]` rows from the create response; otherwise resolve by the
`_FT_<index>` name and accept the reorder hazard.

**Fix:** first run the `property.get` subscript route (and a whole-array `property.get` of
`FoliageTypes`) on a spawner made by `create_procedural`. If it returns the type paths, document it in
the `foliage` wiki page next to `create_procedural` and close this. If it does not, add
`foliage.get_procedural_types {assetPath | actorName}` -> `{spawner, types: [{index, assetPath, mesh,
isAsset}], count}`, sharing the reflection walk `create_procedural` already uses (`:2534-2558`) so
reader and writer cannot drift.

**Acceptance:** for a spawner created with three types, a single documented call returns the three
`UFoliageType` asset paths in array order (and their meshes, for the verb form), matching the
`foliage_types[].asset_path` rows of the create response.

## History
- `#1-no-readback-and-the-name-is-the-api` `OPEN` reporter — Filed from the zone C/D re-speciation
  of `PW_VegetationTest` (`Docs/map/vegetation-style-split.md` § Zone C, § Findings 2). ABSENCE
  confirmed mechanically: `ProceduralFoliageSpawner` occurs in exactly three places under
  `Plugins/PinWright/Source/` (`FoliageHandler.cpp:26`, `:1364`, plus one test at
  `TestFoliageCreateProceduralTypeConfigHonesty.cpp:225-226`), all inside
  `foliage.create_procedural`; none of the six `foliage.*` verbs reads a spawner's configuration,
  and the create response says only `foliage_types_requested` (`:1589`), which restates the request.
  The reader is already written as a writer: `FoliageHandler.cpp:1454-1456` finds the private
  `FoliageTypes` array by `FProperty` reflection under a comment saying why, with `FScriptArrayHelper`
  at `:1458-1462` and the per-struct member lookups at `:1464-1478`. **PREMISE CORRECTED, and the
  correction is the load-bearing part of this ticket.** The reported claim was "the foliage types are
  unreachable from Python". Both observations behind it are real — `to_dict()` on a
  `FoliageTypeObject` is `{}`, and `get_editor_property('foliage_type')` raises "Failed to find
  property" — but the conclusion is very likely false: the member is named `FoliageTypeObject`
  (`FoliageTypeObject.h:45-46`), so the Python spelling is `foliage_type_object`, and
  `PropertyAccessUtil::CanGetPropertyValue` (`PropertyAccessUtil.cpp:425-433`) denies a read only
  when a property has none of `CPF_Edit | CPF_BlueprintVisible | CPF_BlueprintAssignable`, while
  both `FoliageTypes` (`ProceduralFoliageSpawner.h:41-42`) and `FoliageTypeObject` carry
  `EditAnywhere`. C++ `private:` does not gate `get_editor_property`. What the evidence *does*
  support is that the scriptable surface is empty and undiscoverable: the struct is `USTRUCT()` and
  not `BlueprintType` (`FoliageTypeObject.h:13-14`), all members are private with no Blueprint
  specifier (`:43`), `FoliageTypes` has no Blueprint specifier where its own siblings
  `NumUniqueTiles` (`:30-31`) / `MinimumQuadTreeSize` (`:34-35`) do, and `GetFoliageTypes()` (`:62`)
  carries no `UFUNCTION`. NOT SETTLED, one line to settle, flagged for whoever picks this up:
  whether `fto.get_editor_property('foliage_type_object')` actually returns the type. CONSEQUENCE,
  which is why this is filed as a verb request rather than a doc note: believing the readback
  impossible, the pass resolved every type by the asset name the create verb emits —
  `<Name>_Spawner_FT_<index>` (`FoliageHandler.cpp:1438-1439`, `AssetName` at `:1359`, hardcoded
  `/Game/ProceduralFoliage` at `:1357`) — which is positional and encodes nothing about the mesh.
  **Narrow reversal of a prior decision, stated exactly:** the dossier's § H dropped E-9 (stale name
  after a mesh swap) on reasoning that is still correct, and E-9 as written stays dropped; what
  revives is the *adjacent* hazard the same section recorded and left unfiled for want of a repro —
  re-running `create_procedural` with reordered `foliageTypes` rewrites `_FT_0` to a different mesh,
  and there is now a real pass that resolved types solely by that name and wrote `Mesh` onto what it
  found, so a reorder would have edited the wrong asset with every response reporting success. The
  remedy is this verb, not a rename: a stable name would only relocate the problem. Proposed
  `foliage.get_procedural_types {assetPath|actorName}` → `{spawner, types:[{index, assetPath, mesh,
  isAsset}], count}`, implemented by extracting `:1454-1478` into a helper both verbs call so reader
  and writer cannot drift; plus, if cheap, echoing the same `types[]` from `create_procedural`
  itself. Rated **Medium** on the missing-verb band's lower half, because two workarounds exist (the
  `_FT_<n>` convention, which the measured pass used successfully, and a Python read that is one
  correct spelling away); not High, since nothing is silently wrong today and the reorder hazard has
  not been hit; not Low, since the fallback is a positional string built by another verb, i.e. API
  by accident. Reach declined both ways: procedural foliage is not an every-session namespace, but
  reading a spawner's types is step one of every tuning iteration after the first.
- `#2-rephrased` `OPEN` developer — Re-checked at PinWright `7230b41d`. The optional half shipped: `foliage.create_procedural` now echoes `foliage_types[]` with `index` and `asset_path` per built type (`FoliageHandler.cpp:2487-2490`, `:2818-2819`), so the claim that the create response says only `foliage_types_requested` is dropped. All line citations were stale and are refreshed (create verb `:2183`, name build `:2467-2468`, reflection walk `:2534-2558`; the "hardcoded /Game/ProceduralFoliage" note dropped since `savePath` exists, `:2227`). Narrowed to reading an EXISTING spawner, with the untested `property.get` `FoliageTypes[N].FoliageTypeObject` route (`docs/wiki-src/property.md:49`) as the first step and the verb only if it fails. Severity Medium -> Low: with the create echo shipped and a likely one-call generic route, the remaining gap is discoverability on a non-every-session path.
