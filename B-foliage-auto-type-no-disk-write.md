---
id: B-foliage-auto-type-no-disk-write
title: "The UFoliageType that foliage.paint / foliage.add_instances auto-create is mark-dirty-only and never reaches disk, and the only presence field either verb publishes is a hardcoded existsAfter:true — so a scatter that survives the session is gone after an editor restart"
status: OPEN
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
