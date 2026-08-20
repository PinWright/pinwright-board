---
id: B-tests-destroy-host-assets
title: "asset.bulk_delete's default redirector fixup sweeps the whole project, deleting and rewriting host content the caller never named"
status: OPEN
severity: Critical
category: bug
tags: [asset, bulk-delete, redirectors, host-safety, data-loss, blast-radius]
encounters: 1
lastSeen: 2026-08-19T21:05:00Z
---

# asset.bulk_delete's default redirector fixup sweeps the whole project, deleting and rewriting host content the caller never named

Work in the host project `EAContentExamples58` **deleted 4 and modified 13 pre-existing
assets** — not fixtures, but host content tracked in the host's git. All 17 files stamped
2026-08-19 21:05. Recovered only because they were tracked (`git checkout`); anything
untracked at those paths would be gone.

Deleted — all four are `ObjectRedirector` packages (1.5–2.7 KB; `grep -a ObjectRedirector`
confirms), left behind by an earlier content move:
- `Content/ExampleContent/Choosers/1-6/1-6_ABP_Randomize.uasset` → `.../1-6_ABP_Randomize1`
- `Content/ExampleContent/Niagara/PBD/InitializeNeighborGrid.uasset` → `/Game/ExampleContent/Niagara/Modules/Collision/InitializeNeighborGrid`
- `Content/ExampleContent/Niagara/PBD/PopulateNeighborGrid.uasset` → `.../Modules/Collision/PopulateNeighborGrid`
- `Content/ExampleContent/Niagara/PBD/PBD_IntraParticleCollision.uasset` → `.../Modules/Collision/PBD_IntraParticleCollision`

Modified (13), and **every one is a referencer of a deleted redirector** — verified by
grepping the referencing `.uasset` import tables:
`Choosers/1-6/1-6_Randomize_Chooser.uasset` and `1-6_Randomize_Chooser1.uasset` (both import
`1-6_ABP_Randomize`); `Niagara/NeighborGrid3D/Boids.uasset` (`InitializeNeighborGrid`,
`PopulateNeighborGrid`); `Niagara/NeighborGrid3D/Plexus/Plexus.uasset` (all three PBD names);
plus `Material_Nodes/Materials/M_RefractionFresnel.uasset`,
`Math_Hall/Materials/M_MathHall_Mats_WorldPosition_01.uasset`,
`Niagara/DynamicTransforms/DynamicGridTransforms.uasset` and others.

Delete-the-redirector-plus-resave-its-referencers is the exact signature of
`RedirectorFixupPolicy::FixupReferencers(..., bDeleteFixedUpRedirectors=true)`
(`Utils/RedirectorFixupPolicy.cpp:78`), which loads every referencing package
(`:133`), rewrites its soft paths, saves it (`:220`), then deletes the redirector.

## Root cause: the sweep has no path filter and no way to give it one

`asset.bulk_delete` (`Handlers/Asset/AssetWorkflowHandler.cpp:518-549`) runs an automatic
redirector fixup whenever `fixupRedirectors` is set — and it defaults to **true**
(`bool bFixupRedirectors = true;`, `AssetWorkflowHandler.cpp:483`; docstring at `:462` and
`:466`). The filter it builds constrains **class only**:

    FARFilter Filter;
    Filter.ClassPaths.Add(FTopLevelAssetPath(TEXT("/Script/CoreUObject"), TEXT("ObjectRedirector")));
    TArray<FAssetData> RedirectorAssets;
    AssetRegistry.GetAssets(Filter, RedirectorAssets);          // AssetWorkflowHandler.cpp:525-530

No `PackagePaths`, no `bRecursivePaths`, and no correlation to the assets that were just
deleted. `GetAssets` therefore returns **every ObjectRedirector in the project**; `:537`
loads each one and all of them are fixed up and destroyed. Deleting a handful of assets in
one folder silently performs a project-wide redirector purge, rewriting arbitrary unrelated
host packages on the way. `asset.bulk_delete` exposes no scope parameter, so a caller cannot
bound it even deliberately — the only escape is `fixupRedirectors: false`.

## Blast radius: any `/Game` path, in any host

**Not** confined to `ExampleContent`. The filter is class-only, so the sweep reaches every
mounted content root. `ExampleContent` was hit only because that is where this host's
leftover redirectors happened to live. On a host whose redirectors are untracked, or whose
redirectors are still load-bearing for references the registry has not resolved (soft paths
in config/DataTables, packages on another branch, unloaded or cooked content), the deletion
is unrecoverable and silently breaks those references.

`asset.fixup_redirectors` (`AssetWorkflowHandler.cpp:67-82`) has the same whole-project
default when `directoryPath` is empty, but is mitigated: it *has* a scope parameter and the
default is documented (`Docs/wiki-src/asset.md:269`). `bulk_delete` has neither mitigation.

## Trigger: a client RPC, not the test suite

The automation suite **cannot** reach this path. Exhaustively verified across all modules
(`PinWright`, `PinWrightGeometry`, `PinWrightChooser`, `PinWrightPCG`, `PinWrightPoseSearch`,
`PinWrightCommonUI`, `PinWrightRecorder`, `V55Fixtures`):

- **No test invokes `asset.bulk_delete`.** All six source hits are the registration
  (`AssetWorkflowHandler.cpp:460,462`) or comments (`AssetPathParamUtils.h:101`,
  `RedirectorFixupPolicy.h:24`, `TestAssetDeletePathParamAliases.cpp:8`,
  `TestRedirectorFixupPolicy.cpp:3`).
- **The one test dispatching `asset.fixup_redirectors` is scoped** —
  `TestAssetHandlers.cpp:354-358` sets `directoryPath="/Game/__Automation_NoAssets__"`.
  `AsyncHandlerResponseCaptureTest.cpp:117-128` only asserts registration
  (`IsHandlerRegistered`), never dispatches.
- **Both test call sites of `FixupReferencers`** (`TestRedirectorFixupPolicy.cpp:149,197`)
  pass a locally-built array of their own redirectors under
  `/Game/PinWrightTests/RedirectorFixupPolicy/` (`:47-51`).
- **Registry-walk dispatchers are safe by construction.**
  `TestContractConsistency.cpp` `RequiredParamGate.EveryVerb` (`:477-560`) skips non-required
  specs (`:488-491`), so `asset.fixup_redirectors` — which declares **zero** required params
  (`AssetWorkflowHandler.cpp:46-49`) — is never dispatched; `asset.bulk_delete` is enumerated
  only with its required `assetPaths` omitted, so `ValidateHandlerParams` rejects before the
  handler body, and the walk explicitly refuses to dispatch when no required slot would be
  left unsatisfied (`:512-520`). `TestAutoRegistration.cpp` never dispatches at all.

So the route is an **explicit `asset.bulk_delete` RPC from a client** during the session,
which is corroborated in-tree: `Utils/RedirectorFixupPolicy.h:24` records
"Reproduced 2026-08-19 16:05:49 through asset.bulk_delete" — the same day as the damage.

This also means the suite's existing fixture-scoping convention
(`/Game/PinWrightTests`, `/Game/__PW_GatewayTests`) is being followed correctly everywhere it
applies. The hole is in the production handler, not in test hygiene.

## Already known in-tree, but framed as a performance defect

`Docs/plans/defect-backlog.md:672-683` carries this as **D-72, status CONFIRMED**, citing the
same `AssetWorkflowHandler.cpp:525-530` and prescribing the same fix (default `PackagePaths`
to the deleted assets' folders, with an explicit `fixupScope: "project"` opt-in). Its
**Symptom** line is only "Deleting three assets in one folder freezes the editor for
minutes", and `Docs/wiki-src/asset.md:249,269` documents the whole-project sweep as
*intentional*. Neither records that the sweep **permanently deletes host packages the caller
never named and rewrites their referencers**. This ticket exists to put that consequence, and
the field evidence for it, on the board where the fix picker ranks it — D-72's perf framing
would not earn Critical.

Related and worth sequencing with it: **D-75** (`defect-backlog.md:705-712`) lists five more
asset verbs that sweep the project by default, and **D-70/D-73** (`:641-667`,
`:685-694`) cover `bulk_delete`'s modal-hang and its lack of an async token/cancellation.

## Second symptom: fixtures persist in the host's default startup map

`Content/Maps/ExampleProjectWelcome.umap` stays dirty across runs. Its bytes hold **20
distinct `PW_DupMeshIntegrity_<guid>` actor labels** — one per suite run, accumulating. That
map is this host's `EditorStartupMap`, `GameDefaultMap` **and** `ServerDefaultMap`
(`Config/DefaultEngine.ini:22,32,33`), i.e. the map an editor opens with no map argument.

Traced to `FActorDuplicateNeverSilentlySubstitutesMeshTest`
(`PinWrightGeometry/Private/Tests/Geometry/TestActorDuplicateMeshIntegrity.cpp:112-190`,
`PinWright.actor.duplicate.NeverSilentlySubstitutesDynamicMesh`). It spawns into
`GEditor->GetEditorWorldContext().World()` — the live host map, not a scratch world — via
`geometry.create_box`, force-unlocks that level (`GEngine->bLockReadOnlyLevels = false` plus
`ToggleLevelLock`, `:159-168`), and **does not use `FScopedEditorWorldActorGuard`**. It rolls
its own `ON_SCOPE_EXIT { DestroyDuplicateMeshProbeActors(Label); }` (`:51-74`), a label-prefix
scan calling `World->DestroyActor` that, unlike the real guard, **never restores the
persistent level package's dirty flag** (compare `Tests/TestWorldUtils.h:72-75,122-125`,
which does `LevelPackage->SetDirtyFlag(bLevelWasDirty)`). The map is therefore left dirty for
any later `editor.save_all`.

This is a **different mechanism** from the deletions above — it is the failure mode
`B-tests-leak-host-content` describes (dirty package flushed by the suite's own save-all
tests), with an offender that ticket's IN-REVIEW fix does not cover; its `#2` bullet treats
only the sequencer and foliage fixtures. Recorded here as evidence; the fix belongs with that
ticket.

## Relationship to `B-tests-leak-host-content`

Filed separately, not appended there, because:
1. **Different mechanism.** That ticket is about tests *creating* `/Game` packages and
   leaving them dirty for a save-all to flush. This is *deletion and rewriting of
   pre-existing* host assets by a production handler's unfiltered registry sweep — no
   save-all involved.
2. **Different fix site.** That one is test-side (`SetDirtyFlag(false)` + `CleanupTestAsset`);
   this needs a production change in `AssetWorkflowHandler.cpp`.
3. **Different severity class.** Leaked litter is reversible by deleting files. This destroys
   host state.
4. That ticket is `IN-REVIEW` awaiting verification of a specific shipped fix. Returning it to
   `OPEN` for an unrelated symptom its fix never claimed to address would misreport that fix
   as failed and scramble the tester's scope. Per the board rules, `IN-REVIEW` returns to
   `OPEN` only with *that fix's* failure reason.

## Severity rationale

Rated **Critical** against the rubric's "a write that corrupts or loses asset data": a call
that named a handful of assets irreversibly deleted four unrelated host packages and rewrote
thirteen more, with no signal to the caller that it had happened.

Reach, stated honestly: this is **not** an every-session path, and — contrary to the initial
report — the automation suite does not trigger it. It fires on any `asset.bulk_delete` at
default parameters. The rubric's bump-down is for a *rare edge path*; this is the verb's
**default** path, so no reduction applies. Held at Critical.

Counter-argument, recorded so a reader can re-rate: fixing up and deleting a redirector is
semantically what the machinery is *for*, the 13 rewrites are valid path resolutions rather
than corruption, and the whole-project scope is documented (`Docs/wiki-src/asset.md:269`). If
the board reads that as documented-but-dangerous default rather than data loss, `High` is the
floor — it must not go below that, and it should still outrank D-72's perf framing.

**Workaround:** pass `fixupRedirectors: false` on every `asset.bulk_delete`; never call
`asset.fixup_redirectors` without an explicit `directoryPath`. Exercise either verb only in a
fully-tracked checkout that can be reset, so `git checkout` can undo the collateral.

**Fix (proposed, not implemented):**
1. Scope the `asset.bulk_delete` auto-fixup to the redirectors its own deletions produced, or
   at minimum to the deleted assets' package paths, instead of `GetAssets(class-only filter)`.
   Add a `fixupScope: "project"` opt-in for today's behaviour — this is D-72's prescription,
   and adopting it closes both tickets.
2. Report the collateral either way: `redirectorsDeleted` should name paths outside the
   requested set so a caller can see what else was touched.
3. Structural guard: a write/delete gate that refuses mutation outside a caller-named or
   test-owned root while automation is running. The suite already follows that convention by
   hand in every fixture; enforcing it turns host safety from disciplinary into structural.
4. Convert `TestActorDuplicateMeshIntegrity.cpp` to `FScopedEditorWorldActorGuard` (or make
   its bespoke cleanup restore the level dirty flag) so probe actors stop accumulating in the
   host startup map.

## History
- `#1-initial-repro` `OPEN` reporter — Suite run deleted 4 pre-existing host ObjectRedirectors under Content/ExampleContent and rewrote 13 host packages that referenced them; restored via git checkout, unrecoverable if untracked. Root-caused to the class-only, path-unfiltered redirector sweep in asset.bulk_delete's default auto-fixup (AssetWorkflowHandler.cpp:525-530), which feeds every ObjectRedirector in the project to RedirectorFixupPolicy::FixupReferencers with bDeleteFixedUpRedirectors=true; asset.fixup_redirectors has the same whole-project default when directoryPath is empty. Reach is all mounted content, not just ExampleContent. Could not name the invoking test — no test in Source/ references the damaged paths or calls asset.bulk_delete; the defect is structural regardless of caller. Separately traced 20 accumulated PW_DupMeshIntegrity_<guid> probe actors in the host's default startup map to TestActorDuplicateMeshIntegrity.cpp, which spawns into the live editor world without FScopedEditorWorldActorGuard and never restores the level dirty flag. Filed separately from B-tests-leak-host-content (different mechanism, different fix site, and that ticket is IN-REVIEW on an unrelated fix).
- `#2-trigger-is-client-rpc-not-suite` `OPEN` reporter — "Corrected the trigger and retitled. The automation suite CANNOT reach this path: exhaustive sweep of all eight modules found zero test invocations of asset.bulk_delete; the only test dispatching asset.fixup_redirectors scopes it to /Game/__Automation_NoAssets__ (TestAssetHandlers.cpp:354-358); both test FixupReferencers call sites pass locally-built arrays under /Game/PinWrightTests/RedirectorFixupPolicy (TestRedirectorFixupPolicy.cpp:149,197); and the registry-walk dispatcher TestContractConsistency.cpp:477-560 never dispatches a zero-required-param verb and refuses to dispatch when no required slot is left unsatisfied (:512-520). Route is an explicit client asset.bulk_delete RPC, corroborated by RedirectorFixupPolicy.h:24 ('Reproduced 2026-08-19 16:05:49 through asset.bulk_delete'). Confirmed fixupRedirectors defaults true at AssetWorkflowHandler.cpp:483 and captured the four redirectors' targets (a prior Niagara PBD -> Modules/Collision move plus 1-6_ABP_Randomize -> 1-6_ABP_Randomize1). Cross-referenced Docs/plans/defect-backlog.md D-72 (CONFIRMED, same call site and same prescribed fix, but framed as an editor-freeze perf defect with the destructive consequence unrecorded) and D-70/D-73/D-75. Severity held Critical with the reach claim corrected: not every suite run, but the verb's default path. Startup-map symptom re-scoped as a distinct mechanism belonging to B-tests-leak-host-content."
