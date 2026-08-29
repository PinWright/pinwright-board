---
id: E-material-usage-flags-unreadable
title: "No verb reads or writes a material's bUsedWith* usage flags — get_material_info reports domain, blend mode, two-sidedness, shading model, node count, main inputs and parameters and not one usage flag — so a caller building the ISM/HISM scatter the plugin's own wiki teaches cannot see that the material will be swapped for the Default Material in a packaged build"
status: OPEN
severity: Medium
category: ergonomic
tags: [material, material-authoring, get_material_info, readback, usage-flags, ism, hism, instanced-static-mesh, nanite, shader-recompile, packaged-build, vegetation]
encounters: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# The editor hides this condition by fixing it, once per launch, forever

A `UMaterial` drawn by an instanced static mesh needs `bUsedWithInstancedStaticMeshes`. Without it,
the editor silently patches the flag at draw time and recompiles the shader; if the patch cannot be
saved it re-pays that on **every launch**, and a packaged build substitutes the Default Material
instead. None of that is observable through any PinWright verb, because the plugin has no read and no
write for material usage flags at all.

## Found, and how

`M_DotaFoliage` in `EAContentExamples58` carries `bUsedWithInstancedStaticMeshes = false` and
`bUsedWithNanite = false` while being drawn by 2000+ ISM/HISM instances across three zones of
`PW_VegetationTest`, and by roughly **6668** instances on `Dota2_Map_688`
(`Docs/map/vegetation-polish.md` § 3).

It was found by a `python.execute` audit script written for the purpose (`dev/polish/p_mataudit.py`),
because no verb reports it. That is the whole of the in-scope defect; the paragraphs below are why it
is worth a ticket rather than a shrug.

## The reason it looks harmless is the reason it is invisible

The polish pass declined to set the flag, on three grounds, all recorded here because one of them
turns out to be evidence for the ticket rather than against it: the change is additive and correct
but forces a shader recompile of a material carrying ~6668 instances on another map, it produces
**no visible change in this editor session**, and the file sits in another agent's active area.

The middle ground is the interesting one. The change is invisible in the editor because the editor is
already applying it, every session, behind the caller's back:

- `UInstancedStaticMeshSceneProxy` checks the flag while building mesh batches —
  `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/InstancedStaticMesh.cpp:1488`,
  `if (!Section.Material->CheckMaterialUsage_Concurrent(MATUSAGE_InstancedStaticMeshes))`.
- `CheckMaterialUsage` is a thin wrapper —
  `C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Materials/MaterialInterface.cpp:2975-2979`,
  `check(IsInGameThread()); return SetMaterialUsage(Usage);` (off-thread it is bounced back with a log
  line at `:3013` asking someone to *"fix material usage flag"*).
- `UMaterial::SetMaterialUsage`
  (`C:/UE_5.8/Engine/Source/Runtime/Engine/Private/Materials/Material.cpp:1873`), when
  `GIsEditor && !bIsGame && bAutomaticallySetUsageInEditor` (`:1893`), sets the flag and recompiles
  inside an `FMaterialUpdateContext` (`:1904-1909`):

```cpp
// If the flag is missing in the editor, set it, and recompile shaders.
SetUsageByFlag(Usage, true);
// Compile and force the Id to be regenerated, since we changed the material in a way that changes compilation
CacheResourceShadersForRendering(true);
```
`Material.cpp:1911-1915`

- It then dirties the package (`:1920`) and, when it cannot, raises a MapCheck warning carrying a
  "Fix" action (`:1927-1930`). The engine's own message says what happens if nobody takes it:

> `"     The material will recompile every editor launch until resaved."` — `Material.cpp:1949`

- And with auto-set off: `"Material %ls missing usage flag %ls! Default Material will be used in
  game."` — `Material.cpp:1945`.

So *"changes nothing visible in-editor"* is precise and is the defect's signature, not its
mitigation. The recompile the polish pass wanted to avoid is a recompile the editor is already
performing, unsaved, on every launch; and the state that ships is one where 6668 instances draw the
Default Material.

## In-scope residue: PinWright cannot see or set this

**Read.** `material.authoring.get_material_info`
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp:3698`,
body `:3704-3779`) emits `domain` (`:3722-3728`), `blendMode` (`:3734`), `twoSided` (`:3736`),
`shadingModel` (`:3740`), `lightFunctionAtlas` when applicable (`:3749`), `nodeCount` (`:3751`),
`mainInputs` (`:3755`) and `parameters` (`:3775`). **No usage flag of any kind.** The handler never
touches `GetUsageByFlag` or `EMaterialUsage`.

**Write.** There is no usage-flag writer either. A repo-wide grep of
`Plugins/PinWright/Source/` for `bUsedWith` / `UsedWithInstancedStaticMeshes` / `EMaterialUsage`
returns **four** hits and not one is in a handler:

| hit | what |
|---|---|
| `Source/PinWrightGeometry/Private/Handlers/Geometry/GeometrySkeletalAssetCreate.cpp:41` | a file-local `GeometrySkeletalAssetCreate_HasUsage(UMaterialInterface*, EMaterialUsage)` helper |
| `Source/PinWrightGeometry/Private/Tests/Geometry/TestGeometrySkeletalAssetCreate.cpp:402` | test fixture setting `bUsedWithSkeletalMesh` |
| `Source/PinWright/Private/Tests/Assets/TestGenerateThumbnail.cpp:103`, `:847` | test reaching `bUsedWithNiagaraMeshParticles` by reflection |

Zero hits for `UsedWithInstancedStaticMeshes` or `bUsedWithNanite` anywhere.

**Why this lands on PinWright rather than on the project.** The board's scope line excludes
game-level bugs, and `M_DotaFoliage`'s missing flag is one — it is deliberately left off the board,
exactly as `Docs/map/vegetation-findings-dossier.md` § H left `F-2` and `F-3` off it. What is in
scope is that the plugin ships `level-building.instancing-and-scatter` as the documented way to build
a caller-owned ISM/HISM layer, and a caller who follows that page, assigns a material, and then asks
the material-inspection verb what it just built has no field that could tell them the assignment is
conditionally invalid. The routing precedent is the same section's `F-2`, whose readback half was
sent to `E-texture-describe-omits-lodbias-wrap` while its content half stayed off the board.

## Fix

**Primary — the readback.** Add a `usage` block to `get_material_info`, built from
`GetUsageByFlag()` rather than the raw members, since `bUsedWithInstancedStaticMeshes`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Public/Materials/Material.h:795`) and
`bUsedWithNanite` (`:863`) both carry `UE_DEPRECATED(5.8, "Always use the GetUsageByFlag()
accessor.")`. Reporting the whole `EMaterialUsage` set is cheaper than choosing a subset and avoids
a second ticket when the next one matters.

**Secondary — the diagnostic that would have caught this without anyone looking.** A material
assigned to an ISM/HISM whose usage flag is unset is a mechanically checkable condition, and two
verbs are already in the right place to check it: `actor.add_component` when it assigns a mesh
(`Handlers/Actor/ComponentHandler.cpp:113-119`), and whatever lands for
`F-ism-create-and-clear-scatter`. A `warnings[]` entry naming the material and the flag is enough.

**On the write side, a trap worth documenting rather than a verb worth adding.** `property.set` can
reach the flag today, but it writes a `UE_DEPRECATED` member directly rather than going through
`UMaterial::SetMaterialUsage` (`Material.cpp:1873`), which is the path that pairs the write with
`CacheResourceShadersForRendering(true)` (`:1911-1915`) inside an `FMaterialUpdateContext`.
**Unverified and flagged as such:** whether `PostEditChangeProperty` on a `UMaterial` recompiles
equivalently was not checked here. Until it is, the honest doc line is *use the material editor or
let the engine auto-set it, then save* — not *`property.set` the flag*.

## Related

- `E-get-material-info-no-param-defaults` (IN-REVIEW, Low) — the same verb, the same shape: fields
  available on objects the handler is already iterating and not emitted. If both are worked
  together the per-parameter loop and the top-level block change once.
- `E-texture-describe-omits-lodbias-wrap` (encounter 2) — the routing precedent: a content defect
  stays off the board and its readback half becomes an omitted-field ticket.
- `B-thumbnail-primitive-ignored-on-instances` (OPEN) — the only other board ticket that reads
  usage flags, and it had to reach them by raw reflection in a test
  (`TestGenerateThumbnail.cpp:103`) for the same reason: there is no verb.
- `F-ism-create-and-clear-scatter` (OPEN, Medium) / `F-ism-per-instance-transforms` (IN-REVIEW,
  High) — the scatter verbs a usage warning should ride along with.

## Severity

**Medium.** Impact class is the rubric's Medium band, verbatim: *"a readback omits a field and forces
a fallback"*. The fallback here was a purpose-written `python.execute` audit script, which is the
expensive end of that band — the caller has to already suspect the condition to write the script that
finds it.

**Not High.** No PinWright response is false. `get_material_info` does not claim completeness and
nothing on the material pages says usage flags are covered; the omission is silence, not an
over-claim. The engine's own behaviour also softens it: in the editor the flag is auto-patched, so
the material renders correctly in every frame a PinWright caller will ever capture, and the damage is
deferred to a packaged build this project does not produce.

**Not Low.** The Low band is friction where a doc or a field fails to help. What is missing here is
not a convenience but the only observable that distinguishes a correct material assignment from one
that will be substituted at runtime, and the engine actively conceals the difference by repairing it
per session. A readback whose absence is masked by an automatic repair is worse than one that merely
inconveniences.

**Reach modifier declined in both directions.** No bump up: `get_material_info` is a common but not
every-session verb, and the condition needs an ISM/HISM in the picture. No bump down: instanced
scatter is not an edge path — it is the subject of a shipped wiki page, three typed verbs and two
open feature tickets, and every material assigned to one has this question. Medium stands
unmodified.

## History
- `#1-no-usage-flag-read-or-write` `OPEN` reporter — Found during the look-dev polish pass over
  `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 3): `M_DotaFoliage` carries
  `bUsedWithInstancedStaticMeshes = false` and `bUsedWithNanite = false` while drawn by 2000+
  ISM/HISM instances across three zones and ~6668 on `Dota2_Map_688` — found only by a
  purpose-written `python.execute` audit (`dev/polish/p_mataudit.py`), because no verb reports it.
  The content fix was **deliberately not applied** and the reasoning is recorded: additive and
  correct, but it forces a shader recompile of a material with ~6668 instances on another map, it
  produces no visible change in-editor, and the file is in another agent's active area. ENGINE READ
  (UE 5.8) that reframes the middle reason: the editor is already applying the change every session
  — `InstancedStaticMesh.cpp:1488` calls `CheckMaterialUsage_Concurrent(MATUSAGE_InstancedStaticMeshes)`
  while building batches, `MaterialInterface.cpp:2975-2979` forwards to `SetMaterialUsage`, and
  `Material.cpp:1893` gates the editor auto-set which at `:1911-1915` runs
  `SetUsageByFlag(Usage,true)` + `CacheResourceShadersForRendering(true)` inside an
  `FMaterialUpdateContext` (`:1904-1909`), dirties the package at `:1920` and raises a MapCheck
  warning with a Fix action at `:1927-1930`; the engine's own strings state the consequences —
  `"The material will recompile every editor launch until resaved."` (`:1949`) and `"Default
  Material will be used in game."` (`:1945`). So "changes nothing visible in-editor" is the defect's
  signature, not its mitigation: the avoided recompile is one the editor already pays, unsaved, per
  launch, and the shipping state is 6668 instances drawing the Default Material. IN-SCOPE RESIDUE,
  which is what this ticket is: `material.authoring.get_material_info`
  (`MaterialAuthoringHandler.cpp:3698`, body `:3704-3779`) emits `domain` `:3722-3728`, `blendMode`
  `:3734`, `twoSided` `:3736`, `shadingModel` `:3740`, `lightFunctionAtlas` `:3749`, `nodeCount`
  `:3751`, `mainInputs` `:3755`, `parameters` `:3775` — and no usage flag; and a repo-wide grep of
  `Plugins/PinWright/Source/` for `bUsedWith` / `EMaterialUsage` returns four hits, all in a
  file-local geometry helper (`GeometrySkeletalAssetCreate.cpp:41`) or tests
  (`TestGeometrySkeletalAssetCreate.cpp:402`, `TestGenerateThumbnail.cpp:103`, `:847`), with zero
  hits for `UsedWithInstancedStaticMeshes` or `bUsedWithNanite`. The content half stays off the
  board per the scope line, following `Docs/map/vegetation-findings-dossier.md` § H's handling of
  `F-2`/`F-3` and its routing of `F-2`'s readback half to a readback ticket. Asked for: a `usage`
  block on `get_material_info` built from `GetUsageByFlag()` (both members carry
  `UE_DEPRECATED(5.8)` steering to that accessor — `Material.h:795`, `:863`); secondarily a
  `warnings[]` entry when a material is assigned to an ISM/HISM without the flag, which
  `actor.add_component` (`ComponentHandler.cpp:113-119`) and `F-ism-create-and-clear-scatter` are
  both positioned to emit. Write side deliberately NOT requested as a verb: `property.set` reaches
  the flag but writes a deprecated member directly instead of going through
  `UMaterial::SetMaterialUsage` (`Material.cpp:1873`) with its paired
  `CacheResourceShadersForRendering(true)` — and whether `PostEditChangeProperty` recompiles
  equivalently was **not verified here**, so it is flagged rather than recommended. Rated
  **Medium** on the readback-omission band, at its expensive end (the fallback is a bespoke audit
  script the caller must already suspect the problem to write); not High, since no response is false
  and the editor's auto-patch means every frame a caller captures is correct; not Low, since the
  missing observable is the only thing distinguishing a valid assignment from one that is
  substituted at runtime, and the engine masks the difference by repairing it per session. Reach
  declined both ways.
