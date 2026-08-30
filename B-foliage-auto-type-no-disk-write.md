---
id: B-foliage-auto-type-no-disk-write
title: "The UFoliageType that foliage.paint / foliage.add_instances auto-create is mark-dirty-only and never reaches disk, and the only presence field either verb publishes is a hardcoded existsAfter:true — so a scatter that survives the session is gone after an editor restart"
status: IN-REVIEW
severity: High
category: bug
tags: [foliage, paint, add_instances, auto-create, foliage-type, save, no-disk-write, persistence, mcp-safe-asset-save, existsAfter, hardcoded-response]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# The auto-created foliage type is never written, and nothing in the response says so

Hand `foliage.paint` or `foliage.add_instances` a **static-mesh** path and both route through
one shared helper, `PinWrightResolveOrCreateAutoFoliageTypeForMesh`
(`Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:108-153`), which creates a
`UFoliageType_InstancedStaticMesh` in a fresh `/Game/Foliage/Auto_<Mesh>` package and finishes
with `McpSafeAssetSave(AutoFT)` (`:150`).

`McpSafeAssetSave` writes nothing. Its header states the contract outright
(`Source/PinWright/Private/Utils/AssetUtils.h:100-113`):

> Mark an asset for a later save: MarkPackageDirty + FAssetRegistryModule::AssetCreated.
> It deliberately does NOT write the .uasset (the immediate write is the documented
> bulkdata-corruption vector on UE 5.7+), so NOTHING it does makes an edit durable.

The same header names the two helpers that would make it honest — `AddMarkDirtySaveReport`
(`:293`, publishes a *measured* mark-dirty report) and `SaveAssetToDiskReportingPresence`
(`:241`, actually lands the `.uasset`). **Neither appears anywhere in `FoliageHandler.cpp`:**
`grep -c` for both returns 0 over the whole file, while the rest of the plugin uses them at 89
and 46 call sites respectively.

## What the caller is told instead

Nothing about persistence. Four foliage verbs close their response with the same two lines:

```cpp
Resp->SetStringField(TEXT("foliageActorPath"), IFA->GetPathName());
Resp->SetBoolField(TEXT("existsAfter"), true);          // :1081 :1237 :1479 :1928
```

`existsAfter` is a literal at all four sites (`paint` `:1081`, `remove` `:1237`,
`get_instances` `:1479`, `add_instances` `:1928`) and is reached only on a path where the
foliage actor pointer is already known non-null — so it is a constant published under a
measurement name, the shape `E-foliage-add-type-auto-save-undocumented` `#2` and
`AddAssetVerification` were already called out for. There is no `saved` field, no `package`,
no `sizeBytes`, and no disk probe of any kind on either verb.

Consequence: a caller scatters foliage, reads back the instances successfully, sees
`existsAfter: true`, and quits the editor. The instances are gone, because the type asset they
reference was only ever dirty in memory. Recovery requires knowing to call `asset.save` (or
`EditorAssetLibrary.save_loaded_asset`) on a path the verb echoes but never says needs saving.

## Provenance

Carved out of `B-add-instances-auto-foliage-type-name-mismatch` (IN-REVIEW, High), whose `#1`
filed this as its "second, separable half" and whose `#3` closed the naming half and recorded
the residue verbatim:

> **NOT DONE, still open work:** the ticket's "Second half, separable" — `McpSafeAssetSave`
> marks dirty and writes nothing, and `add_instances` still reports a hardcoded
> `existsAfter:true` with no disk probe. Neither `SaveAssetToDiskReportingPresence` nor
> `AddMarkDirtySaveReport` was wired in.

Filed as its own ticket rather than left there because that ticket is **IN-REVIEW on the
naming fix**: a tester verifying `#3` will flip it to `DONE` on the half it did fix, and the
picker only works `OPEN` tickets — so the residue would never be scheduled. Re-verified
against the current tree (post-`#3`, with the `Auto_X.Auto_X` rename in place): still live.

`E-foliage-add-type-auto-save-undocumented` `#2` (WONTFIX) reached the same finding from the
opposite direction, refuting that ticket's premise and routing the genuine phenomenon here:

> The only genuine phenomenon (`exists_after:true` is a misleading persistence signal after a
> mark-dirty-only create) is the INVERSE of this ticket and is already an owned defect class
> (`B-niagara-save-no-disk-write`, `B-metasound-create-save-no-disk-write`,
> `B-audio-create-save-no-disk-write`); it belongs there, not as a reword of this docs ticket.

This ticket is that missing foliage member. The family — `B-audio-create-save-no-disk-write`,
`B-metasound-create-save-no-disk-write`, `B-texture-save-no-disk-write`,
`B-material-authoring-save-no-disk-write`, `B-pose-search-create-save-no-disk-write`
(Critical); `B-niagara-save-no-disk-write`, `B-geometry-generate-lods-no-disk-write` (High) —
had no foliage entry.

## Severity

**High**, not Critical. Impact class is silent non-persistence with a constant published under
a measurement name — the family's shape. Declined Critical, which the majority of that family
carries, on one difference that is real: those verbs answer `saved: true`, an affirmative lie
about disk. These two answer nothing about disk at all, so a caller is under-informed rather
than actively misled, and the asset is fully recoverable for as long as the editor is up.
Declined the reach up-bump for the same reason `B-add-instances-auto-foliage-type-name-mismatch`
did: the auto-create fires only on the static-mesh-path branch, which a caller passing a real
`UFoliageType` path never enters. Held at High to match the parent.

**Workaround:** after any `foliage.paint` / `foliage.add_instances` that passed a static-mesh
path, call `asset.save` on the `foliageTypePath` the response echoes. Note the pre-fix package
handle does not resolve on hosts that predate `B-add-instances-auto-foliage-type-name-mismatch`
`#3` — use the dotted object path the verb returns.

**Fix:** route the auto-create through `SaveAssetToDiskReportingPresence` (`AssetUtils.h:241`)
and publish its measured `saved` / `package` / `sizeBytes`, or keep `McpSafeAssetSave` and
report it honestly with `AddMarkDirtySaveReport` (`AssetUtils.h:293`). A
`UFoliageType_InstancedStaticMesh` is not a Blueprint/SCS asset, so the UE 5.7+ bulkdata
corruption vector that forced `McpSafeAssetSave` on Blueprint-shaped assets does not obviously
apply here — but that is the one thing to establish before choosing the first option, and it
was not established in this pass. Either way, delete the hardcoded `existsAfter: true` at all
four sites or replace it with a probe; a constant under a measurement name is the defect
regardless of which save path wins.

## History
- `#1-persistence-half-carved-out` `OPEN` reporter — Source-only, verified against the current working tree (which carries the uncommitted `#3` fix of `B-add-instances-auto-foliage-type-name-mismatch`, so the naming half is already in). Established: `PinWrightResolveOrCreateAutoFoliageTypeForMesh` (`FoliageHandler.cpp:108-153`) ends at `McpSafeAssetSave(AutoFT)` (`:150`); `McpSafeAssetSave`'s own header (`Utils/AssetUtils.h:100-113`) states it "deliberately does NOT write the .uasset ... so NOTHING it does makes an edit durable"; `grep -c` for `SaveAssetToDiskReportingPresence` and `AddMarkDirtySaveReport` over `FoliageHandler.cpp` both return **0**, against 89 and 46 call sites elsewhere in `Source/`; and `existsAfter` is a bare `SetBoolField(..., true)` at `:1081` (paint), `:1237` (remove), `:1479` (get_instances) and `:1928` (add_instances), each on a branch where the foliage actor is already known non-null. No `saved`, `package` or `sizeBytes` field exists on either verb. **No editor call was made and no foliage was scattered this pass** — the loss-after-restart consequence is inferred from the mark-dirty-only contract, not observed here; it *was* observed independently by the `#4` reporter on `B-add-instances-auto-foliage-type-name-mismatch`, who measured `EditorAssetLibrary.save_asset` silently failing for 17 of 33 auto-created types in one batch. Filed as a new ticket rather than appended to that parent because the parent is IN-REVIEW on the naming fix and the fix picker works only OPEN tickets, so a residue left in an IN-REVIEW ticket's history is never scheduled; a cross-reference append was made there in the same pass. Severity High, argued above against the mostly-Critical no-disk-write family: these verbs publish no `saved` field, so they under-inform rather than lie, and the asset survives for the life of the editor session.
- `#2-measured-persistence-report` `IN-REVIEW` developer — "Kept `McpSafeAssetSave` in `PinWrightResolveOrCreateAutoFoliageTypeForMesh` deliberately and fixed the REPORTING, which is where the defect actually was. The decision is argued in a comment at that call site: (1) a scatter needs TWO packages to survive a restart — the type asset AND the level package holding the `AInstancedFoliageActor`'s instances — and neither verb saves the level, so force-writing only the type would move `saved` to true for the half that is unreadable without the other, a *more* misleading report than the honest 'both are dirty'; (2) neither verb takes a `save` flag, so writing a `.uasset` as a side effect of a paint is an unrequested mutation, against the house model in `Docs/rpc-design.md` §5 and `Docs/wiki-src/safe-mutation-save.md`; (3) `foliage.add_type` and `foliage.create_procedural` are already on `McpSafeAssetSave`, so all three foliage creates stay in one persistence regime. No `SavePackage` call was added anywhere. **Two new file-local helpers in `Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp`, both grep-verified unique across `Source/` for the Unity build.** `PinWrightReportAutoFoliageTypePersistence` wires the auto-create path onto the two honest reporters the ticket named, rather than a third spelling: `AddMarkDirtySaveReport` (top-level `saveRequested` / `markedForSave` / `saved` / `pendingFlush`, with `saved` MEASURED through `IsAssetPersistedToDisk`) plus `AddAssetVerificationNested(Resp, \"foliageType\", …)` (the measured `existsAfter` / `existsOnDisk` / `pendingSave` triple, NESTED so it cannot be confused with the top-level `existsAfter`, which is about the foliage actor), plus a `warnings[]` line naming the remedy whenever the measurement says not-durable — appended to any `warnings[]` the verb already wrote, never replacing it. The same measurement covers both branches of the resolve-or-create helper with no created/reused flag: a fresh type answers `saved:false` + `pendingFlush:true`, a reused already-written one answers `saved:true`. It is emitted ONLY on the static-mesh auto-create branch (gated by a new `bAutoResolvedFoliageType` local in each verb) — a caller who passed a real `UFoliageType` path gets none of these fields, because that call created and dirtied nothing: OMIT rather than assert. `PinWrightWriteFoliageActorPresence` replaces the hardcoded presence pair at every site: `existsAfter` is now the round trip the write path cannot fake — the very `foliageActorPath` string the response publishes is resolved BACK through `FindObject<AInstancedFoliageActor>` and compared against the actor it names — and it also single-sources the `foliageActorPath` spelling that was open-coded four times. **FIVE hardcoded sites found in total, not four.** The ticket's four `existsAfter: true` literals (`foliage.paint`, `foliage.remove`, `foliage.get_instances`, `foliage.add_instances`) plus a FIFTH the ticket did not name: `foliage.add_type` published a snake_case `exists_after: true` literal three lines above its own `AddAssetVerification(Resp, FoliageType)` call, which emits a MEASURED `existsAfter` — two fields spelling one name, agreeing with each other while both could disagree with the disk, the exact shape `Docs/rpc-design.md` §1 records. `exists_after` is now mirrored from that measurement, and `add_type` also gained `AddMarkDirtySaveReport` over its own `McpSafeAssetSave`, which said nothing about persistence either. `foliage.create_procedural` was checked and needs neither: it publishes no `exists_after` literal and already routes through `AddAssetVerification`/`AddActorVerification` (its own `McpSafeAssetSave(FT)`/`(Spawner)` disclosure is the same family and is NOT fixed here — a separate ticket's worth). Error-code trap respected: no new codes, and `grep -c 'ErrorCodes::'` over `FoliageHandler.cpp` is still **0**, so its raw-literal style is untouched. **Regression test** `Source/PinWright/Private/Tests/Environment/TestFoliageAutoTypePersistenceHonesty.cpp`, two ids — `PinWright.foliage.add_instances.AutoCreatedTypePersistenceIsMeasured` and `PinWright.foliage.paint.AutoCreatedTypePersistenceIsMeasured`. Every assertion compares the response's OWN on-disk claim against an independent `IFileManager::FileSize` probe of the `.uasset` (through the shared `PackageFilenameFromAssetPath`), in BOTH directions: Direction 1 auto-creates and asserts probe-absent == `foliageType.existsOnDisk:false` + `saved:false` + `pendingFlush:true` + `markedForSave:true` + a warning naming the handle; Direction 2 force-saves through `asset.save {force:true}` (the 0.5 s throttle would otherwise make Direction 2 measure Direction 1), re-scatters the same mesh so the type is REUSED, and asserts probe-present == `existsOnDisk:true` + `saved:true` + no warning — which is what stops a hardcoded `false` passing the test the hardcoded `true` failed; Direction 3 passes a real `UFoliageType` path and asserts the save fields are ABSENT. A cross-field invariant (`saved` may never be true while `existsOnDisk` is false; `pendingFlush == !saved`) is asserted on every response carrying the block. It fails on today's code at the presence assertions — pre-fix the response carries no `foliageType` object, no `saved` and no `saveRequested`, and its only presence claim is the literal `existsAfter:true`. Fixture isolation: drives `BasicShapes/Cone` (add_instances) and `BasicShapes/Plane` (paint), neither of which any other foliage test touches (Cube and Cylinder are taken), and discards both the `Auto_X.Auto_X` and the pre-`B-add-instances-auto-foliage-type-name-mismatch` `Auto_X.X` forms on entry and on exit so an asset left on disk by an older build cannot satisfy Direction 1's probe. `Content/Python/check_test_ids.py` re-run after adding them: CLEAN, 4800 ids, no dot-prefix collision, no duplicate. Docs: `Docs/wiki-src/foliage.md` gained a `##` section (above the first `###`, so it renders) — 'An auto-created foliage type is dirty, not written' — stating the field set, that top-level `existsAfter` is about the ACTOR and never about durability, and that durability needs `asset.save` on the type AND a level save; the `### foliage.add_instances` and `### foliage.paint` sections point at it. That page was already ~22 KB before this change and is now ~24 KB, over the soft ~20 KB guideline — flagged, not addressed, since trimming unrelated sections is out of scope here. Built on `B-add-instances-auto-foliage-type-name-mismatch` `#3` rather than undoing it: the shared resolve-or-create helper, its `Auto_X.Auto_X`-then-`Auto_X.X` probe order and its `GetPathName()` return are all intact, and the new persistence report hangs off that single place the asset comes into existence. NOT compiled and NOT run — the orchestrator owns builds and the suite. Four other agents were editing this file concurrently; every edit re-read its target region immediately beforehand and nothing authored elsewhere was reverted (`foliage.add_type`'s new `save_path` echo and `add_instances`' cluster-tree summary text were both present and left alone)."
