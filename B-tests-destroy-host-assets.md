---
id: B-tests-destroy-host-assets
title: "Automation runs delete and rewrite pre-existing host Content: the redirector fixup sweep is project-wide and unfiltered"
status: OPEN
severity: Critical
category: bug
tags: [tests, host-safety, redirectors, data-loss, asset-cleanup]
encounters: 1
lastSeen: 2026-08-19T21:05:00Z
---

# Automation runs delete and rewrite pre-existing host Content: the redirector fixup sweep is project-wide and unfiltered

A PinWright automation suite run **deleted 4 and modified 13 pre-existing assets** in the
host project `EAContentExamples58` — not fixtures the suite created, but host content
tracked in the host's git. All 17 files stamped 2026-08-19 21:05. Recovered only because
they were tracked (`git checkout`); anything untracked at those paths would be gone.

Deleted (all four are `ObjectRedirector` packages, 1.5–2.7 KB, `grep -a ObjectRedirector`
confirms):
- `Content/ExampleContent/Choosers/1-6/1-6_ABP_Randomize.uasset`
- `Content/ExampleContent/Niagara/PBD/InitializeNeighborGrid.uasset`
- `Content/ExampleContent/Niagara/PBD/PBD_IntraParticleCollision.uasset`
- `Content/ExampleContent/Niagara/PBD/PopulateNeighborGrid.uasset`

Modified (13), and **every one is a referencer of a deleted redirector** — verified by
grepping the referencing `.uasset` bytes for the redirector name:
`Choosers/1-6/1-6_Randomize_Chooser.uasset` and `1-6_Randomize_Chooser1.uasset` (both
carry `1-6_ABP_Randomize`); `Niagara/NeighborGrid3D/Boids.uasset`
(`InitializeNeighborGrid`, `PopulateNeighborGrid`);
`Niagara/NeighborGrid3D/Plexus/Plexus.uasset` (all three PBD names); plus
`Material_Nodes/Materials/M_RefractionFresnel.uasset`,
`Math_Hall/Materials/M_MathHall_Mats_WorldPosition_01.uasset`,
`Niagara/DynamicTransforms/DynamicGridTransforms.uasset` and others.

Delete-the-redirector-plus-resave-its-referencers is the exact signature of
`RedirectorFixupPolicy::FixupReferencers(..., bDeleteFixedUpRedirectors=true)`
(`Utils/RedirectorFixupPolicy.cpp:78`), which loads every referencing package, rewrites the
soft paths and saves them, then deletes the redirector.

## Root cause: the sweep has no path filter and no way to give it one

`asset.bulk_delete` (`Handlers/Asset/AssetWorkflowHandler.cpp:518-549`) runs an automatic
redirector fixup whenever `fixupRedirectors` is set — **the documented default**
(`AssetWorkflowHandler.cpp:462`: "By default automatically runs asset.fixup_redirectors
afterwards"). The filter it builds constrains **class only**:

    FARFilter Filter;
    Filter.ClassPaths.Add(FTopLevelAssetPath(TEXT("/Script/CoreUObject"), TEXT("ObjectRedirector")));
    TArray<FAssetData> RedirectorAssets;
    AssetRegistry.GetAssets(Filter, RedirectorAssets);          // AssetWorkflowHandler.cpp:525-530

No `PackagePaths`, no `bRecursivePaths`, no correlation to the assets that were just
deleted. `GetAssets` therefore returns **every ObjectRedirector in the entire project**, and
all of them are fixed up and destroyed. Deleting one throwaway fixture silently performs a
project-wide redirector purge as a side effect, rewriting arbitrary unrelated host packages
on the way. `asset.bulk_delete` exposes no `directoryPath` parameter, so a caller cannot
scope it even deliberately.

The sibling verb `asset.fixup_redirectors` (`AssetWorkflowHandler.cpp:71-80`) does honour a
`directoryPath` — but that parameter is **optional and empty by default**, and the docstring
states "empty (default) scans the entire project". So both redirector call sites default to
whole-project reach.

## Blast radius: any `/Game` path, in any host

This is **not** confined to `ExampleContent`. The filter is class-only, so the sweep reaches
every mounted content root — game content, plugin content, anything the asset registry
knows. `ExampleContent` was hit only because that is where this host's leftover redirectors
happened to live. On a host whose redirectors are untracked, or whose redirectors are still
load-bearing for references the registry has not scanned (soft paths in config/DataTables,
packages on another branch, unloaded or cooked content), the deletion is unrecoverable and
silently breaks those references. That is the property that puts this in the data-loss band.

## Which tests

**Not established.** Grepping `Source/` for the damaged asset paths returns nothing
(the only `ExampleContent` literals in the whole plugin are two
`ControlRig/Animations/Punch_Montage` loads at
`Tests/Gameplay/TestAnimationHandlers.cpp:1169,1248`), so no test names these assets — they
were collected by the registry sweep, which is consistent with the mechanism above. No test
invoking `asset.bulk_delete` was found either; the trigger is therefore either an MCP-driven
call made during the run, or a path not yet located. **The defect is structural regardless of
the caller**: any `asset.bulk_delete` at default settings destroys host redirectors.

Ruled out by inspection — these are correctly scoped and are *not* the offenders:
- `PinWright.asset.fixup_redirectors.ValidParamsNoCrash` (`Tests/Assets/TestAssetHandlers.cpp:348-359`)
  passes `directoryPath = /Game/__Automation_NoAssets__`.
- `PinWright.asset.delete.*` (`TestAssetHandlers.cpp:32-57`) targets a nonexistent path and
  has no fixup step.
- `Tests/Assets/TestRedirectorFixupPolicy.cpp:49` builds fixtures under
  `/Game/PinWrightTests/RedirectorFixupPolicy/` and passes explicit redirector arrays.
- `Tests/Assets/AssetRefDirectionFixtures.h:40-42` builds under `/Game/__PW_GatewayTests`.
- `PinWright.chooser.*SaveWritesToDisk` (`PinWrightChooser/.../TestChooserCreateSaveWritesToDisk.cpp:44`)
  uses `/Game/PinWrightTests/CH_SaveToDisk_<guid>` with `CleanupTestAsset` on every path.

So the suite already has a good scoping convention — it is simply not enforced anywhere, and
the production handler ignores it entirely.

## Second symptom: fixtures persist in the host's default startup map

`Content/Maps/ExampleProjectWelcome.umap` stays dirty across runs. Its bytes hold **20
distinct `PW_DupMeshIntegrity_<guid>` actor labels** — one per suite run, accumulating. That
map is this host's `EditorStartupMap`, `GameDefaultMap` **and** `ServerDefaultMap`
(`Config/DefaultEngine.ini:22,32,33`), i.e. the map an editor opens with no map argument.

Traced to `FActorDuplicateNeverSilentlySubstitutesMeshTest`
(`PinWrightGeometry/Private/Tests/Geometry/TestActorDuplicateMeshIntegrity.cpp:107-125`). It
spawns into `GEditor->GetEditorWorldContext().World()` — the live host map, not a scratch
world — and **does not use `FScopedEditorWorldActorGuard`**. It rolls its own
`ON_SCOPE_EXIT { DestroyDuplicateMeshProbeActors(Label); }`
(`TestActorDuplicateMeshIntegrity.cpp:51-73`), a label-prefix scan calling `World->DestroyActor`
that, unlike the real guard, **never restores the level package's dirty flag** (compare
`Tests/TestWorldUtils.h:117-120`, which does `LevelPackage->SetDirtyFlag(bLevelWasDirty)`).
The test also deliberately clears `ULevel::bLocked` and `GEngine->bLockReadOnlyLevels`
(`TestActorDuplicateMeshIntegrity.cpp:158-168`) to reach the duplication path, disarming the
host's own protection against level mutation.

Note this is a **different mechanism** from the deletions above, not the same cause. It is
the failure mode `B-tests-leak-host-content` describes (dirty package flushed by the suite's
own save-all tests) with an offender that ticket's IN-REVIEW fix does not cover — its `#2`
bullet treats only the sequencer and foliage fixtures. Recorded here as evidence; the fix
belongs with that ticket's tester.

## Relationship to `B-tests-leak-host-content`

Filed separately, not appended there, because:
1. **Different mechanism.** That ticket is about tests *creating* `/Game` packages and
   leaving them dirty for a save-all to flush. This is *deletion and rewriting of
   pre-existing* host assets by a production handler's unfiltered registry sweep — no
   save-all needed.
2. **Different severity class.** Leaked litter is reversible by deleting files. This
   destroys host state.
3. **Different fix site.** That one is test-side (`SetDirtyFlag(false)` + `CleanupTestAsset`).
   This needs a production-code change in `AssetWorkflowHandler.cpp`.
4. That ticket is `IN-REVIEW` awaiting verification of a specific shipped fix. Returning it
   to `OPEN` for an unrelated symptom its fix never claimed to address would misreport that
   fix as failed and scramble the tester's scope. Per the board rules, `IN-REVIEW` returns to
   `OPEN` only with *that fix's* failure reason.

## Severity rationale

Rated **Critical** against the rubric's "a write that corrupts or loses asset data": the run
irreversibly deleted four host packages and rewrote thirteen more without being asked.
Reach cannot raise it further, but is worth recording — it fires on every suite run and the
suite runs constantly, and the sweep is unbounded across all mounted content.

Counter-argument, recorded so a reader can re-rate: fixing up and deleting a redirector is
semantically what the verb is *for*, and the 13 rewrites are valid path resolutions, not
corruption. The defect is the **unbounded scope** and that it fires as an uninvited side
effect of an unrelated delete. If the board prefers to read that as silent-wrong-behaviour
rather than data loss, `High` is the floor — it should not go below that.

**Workaround:** pass `fixupRedirectors: false` to every `asset.bulk_delete`; never call
`asset.fixup_redirectors` without an explicit `directoryPath`. Run the suite only in a host
checkout that is fully tracked and reset between runs, so `git checkout` can undo it.

**Fix (proposed, not implemented):**
1. Scope the `asset.bulk_delete` auto-fixup to the redirectors its own deletions produced —
   or at minimum to the package paths of the deleted assets — instead of
   `GetAssets(class-only filter)`. Add a `directoryPath` parameter so a caller can bound it.
2. Make whole-project reach opt-in rather than the default on `asset.fixup_redirectors`:
   require an explicit scope, or an explicit `wholeProject: true`.
3. Structural guard: a write/delete gate that refuses any mutation outside a test-owned root
   (`/Game/PinWrightTests`, `/Game/__PW_GatewayTests`, `/Temp`, transient) while automation
   is running. The suite already follows that convention by hand in every correctly-written
   fixture; enforcing it turns host safety from disciplinary into structural.
4. Convert `TestActorDuplicateMeshIntegrity.cpp` to `FScopedEditorWorldActorGuard` (or make
   its bespoke cleanup restore the level dirty flag) so probe actors stop accumulating in the
   host startup map.

## History
- `#1-initial-repro` `OPEN` reporter — Suite run deleted 4 pre-existing host ObjectRedirectors under Content/ExampleContent and rewrote 13 host packages that referenced them; restored via git checkout, unrecoverable if untracked. Root-caused to the class-only, path-unfiltered redirector sweep in asset.bulk_delete's default auto-fixup (AssetWorkflowHandler.cpp:525-530), which feeds every ObjectRedirector in the project to RedirectorFixupPolicy::FixupReferencers with bDeleteFixedUpRedirectors=true; asset.fixup_redirectors has the same whole-project default when directoryPath is empty. Reach is all mounted content, not just ExampleContent. Could not name the invoking test — no test in Source/ references the damaged paths or calls asset.bulk_delete; the defect is structural regardless of caller. Separately traced 20 accumulated PW_DupMeshIntegrity_<guid> probe actors in the host's default startup map to TestActorDuplicateMeshIntegrity.cpp, which spawns into the live editor world without FScopedEditorWorldActorGuard and never restores the level dirty flag. Filed separately from B-tests-leak-host-content (different mechanism, different fix site, and that ticket is IN-REVIEW on an unrelated fix).
