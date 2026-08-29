---
id: B-foliage-mutators-no-transaction
title: "No mutating foliage verb opens an FScopedTransaction — three call IFA->Modify() after the write and two call nothing at all — so no undo record is ever written and editor.undo cannot reverse any scatter, foliage type or procedural volume the namespace authors"
status: OPEN
severity: Medium
category: bug
tags: [foliage, paint, remove, add_instances, add_type, create_procedural, undo, transaction, mutator-no-undo-transaction]
---

# The whole `foliage` namespace mutates outside the transaction system

`Handlers/Environment/FoliageHandler.cpp` contains **zero** occurrences of `FScopedTransaction`
across its entire 1233 lines. Six verbs are registered in the file; `foliage.get_instances`
(`Handlers/Environment/FoliageHandler.cpp:370`) is read-only and correctly needs nothing. The
other **five all mutate, and none of them opens a transaction**:

| verb | registered | mutation | what stands in for the transaction |
| --- | --- | --- | --- |
| `foliage.paint` | `:75` | `FFoliageInfo::AddInstance` loop `:221-238` | `IFA->Modify()` at `:240` — **after** the loop |
| `foliage.remove` | `:257` | `Info.Instances.Empty()` `:340`, `Info->Instances.Empty()` `:348` | `IFA->Modify()` at `:343` and `:349` — **after** each wipe |
| `foliage.add_type` | `:489` | `NewObject<UFoliageType_InstancedStaticMesh>` `:601-602`, property writes `:609-620`, `McpSafeAssetSave` `:622` | **nothing** — no `Modify()` anywhere in the verb (`:489-637`) |
| `foliage.add_instances` | `:640` | `FFoliageInfo::AddInstance` loop `:882-897` | `IFA->Modify()` at `:898` — **after** the loop |
| `foliage.create_procedural` | `:934` | spawner `NewObject` `:1006-1007`, foliage-type `NewObject` `:1071`, actor spawn `:1114-1118`, `SetActorScale3D` `:1125`, resimulate `:1188` | **nothing** — no `Modify()` anywhere in the verb (`:934-1233`) |

This is against the standing house rule, `agent-conventions.md:46`: *"Wrap EVERY mutation in
`FScopedTransaction` — placed after validation, before the first mutation."* The same rule is
restated as a contract for overlay authors in `Docs/SCHEMA.md:76` and as a lesson in
`Docs/lessons.md:26`.

## What this actually costs, and what it does not

**Undo is gone for all five verbs.** With no transaction open, `GUndo` is null on these call
paths, so `UObject::Modify` never reaches `SaveToTransactionBuffer` and no undo record is
written. `editor.undo` (which correctly calls `GEditor->UndoTransaction()`, per `B-no-undo-redo`
`#9`) has nothing to pop. Every scatter, every wipe, every generated `UFoliageType` asset and
every procedural volume is one-way.

**Persistence is *not* additionally at risk — checked, and the claim does not hold.** The
obvious second-order worry is the one `Handlers/Environment/EnvironmentDirtyUtils.h:6-12`
documents for the sibling light/fog/PPV verbs: `UWorld::SpawnActor` dirties the level **only**
under a transaction (`if (GUndo) ModifyLevel(LevelToSpawnIn);`, `LevelActor.cpp:735-739`), and
`create_procedural` spawns its volume through `SpawnActorInActiveWorld`
(`Utils/AssetUtils.h:538-615`), which issues a bare `World->SpawnActor` (`:560`) with no dirty
ceremony and does not include that header. But the verb passes `Name` as the optional label
(`Handlers/Environment/FoliageHandler.cpp:1114-1118`), so `Spawned->SetActorLabel(...)`
(`Utils/AssetUtils.h:611`) runs and its default `bMarkDirty` path calls `Modify()` on the actor,
which dirties the package. The interactive-editor branch also spawns via
`UEditorActorSubsystem::SpawnActorFromClass` (`:592`), which dirties on its own. So the volume
persists — **incidentally**, off a label call, not because anything intended it. Worth making
explicit in the fix, but this ticket is about undo, not lost work.

**`paint` / `remove` / `add_instances` also still dirty.** Their trailing `IFA->Modify()` falls
through to `MarkPackageDirty()` when `SaveToTransactionBuffer` returns false
(`UObject::Modify`, `Obj.cpp:1652-1680`, cited at
`Handlers/Environment/EnvironmentDirtyUtils.h:44-48`), and `MarkPackageDirty` is
order-independent. Those three lose undo only. `add_type` persists via `McpSafeAssetSave`.

**The REINST_ half of the house rule's stated mechanism does not bite here.** The rule is
justified by blueprint recompiles minting `REINST_` classes that `UTransBuffer` holds stale
refs to, causing PIE ensures in `CheckAndHandleStaleWorldObjectReferences`. Foliage verbs
recompile no blueprint and mint no `REINST_` class, so that specific failure mode is not in
play. The rule still applies — it is unconditional, and undo is a user-facing editor
expectation independent of the REINST_ pathology — but this ticket does not claim a PIE-ensure
risk it cannot evidence. Everything above is a source read at HEAD; the editor was not started
and no `foliage.*` call was made.

## The trap that will get this wrongly closed

Wrapping the existing code in an `FScopedTransaction` without moving the `Modify()` calls
**leaves undo broken while making it look fixed**. `Modify()` snapshots the object's state at
the moment it is called; the three existing calls sit *after* their mutation, so the snapshot
would record the already-changed value and Ctrl+Z would restore the post-edit state — a no-op.
This project has already written that rule down, in this same directory:
`Handlers/Environment/EnvironmentDirtyUtils.h:30-37` — *"Ordering is the whole point: Modify()
must precede the write... A post-hoc Modify() would snapshot the already-changed value and make
Ctrl+Z a no-op."*

`create_procedural` has the harder shape: `Transaction.Cancel()` does not reverse `NewObject`
creation (`agent-conventions.md:89`), so its spawner and foliage-type asset creation need the
manual-cleanup treatment on the error paths, not just a scope.

## Sibling that does it right

`Handlers/Environment/LandscapeHandler.cpp:653-654` — same directory, same Environment
namespace, correct shape: validation completes, then the transaction opens and `Modify()` is
called inside it and before the first write.

```cpp
const FScopedTransaction Transaction(FText::FromString(TEXT("Create Landscape")));
Landscape->Modify();
```

That is the pattern to copy into each of the five foliage verbs.

**Fix:** open one `FScopedTransaction` per verb after validation and before the first mutation;
move each `IFA->Modify()` to precede its write and inside the scope; give `add_type` and
`create_procedural` the `Modify()` calls they currently lack; and while in
`create_procedural`, replace the incidental label-driven dirty with an explicit
`PinWright::MarkLevelActorSpawned` (`Handlers/Environment/EnvironmentDirtyUtils.h:95-109`) after
the spawn, so persistence stops depending on the caller supplying a label. One transaction
around the whole `add_instances` / `paint` batch is the correct granularity and is not
expensive: `FTransaction::SaveObject` deduplicates per object per transaction, so a
10k-instance batch costs exactly one snapshot — the arithmetic is worked in
`B-ism-undo-record-unsafe`, which found the opposite (per-`Modify()`) assumption to be wrong.

## Scope

Foliage is the subject. The gap is wider: of the 84 handler `.cpp` files under `Handlers/` that
call `Modify()`, **47 contain no `FScopedTransaction`** — a separate audit, not this ticket.

severity rationale: impact=soft-blocker — every foliage mutation is one-way and reversal is only possible by composing other verbs, and coarsely: `foliage.remove` clears every instance of a type on the IFA rather than the batch just added, so a scatter authored in passes cannot be stepped back, while `add_type` and `create_procedural` need `asset.delete` / actor destruction instead; nothing here is a false success, a corrupting write or lost work, and the rule's REINST_/PIE-ensure mechanism does not apply to foliage × reach=neither modifier applies — these are the namespace's five main authoring verbs, not a fallback branch, so the "rare edge path" bump-down is declined, and they are not every-session verbs, so the bump-up is declined too -> Medium.

## History
- `#1-namespace-wide-no-transaction` `OPEN` reporter — Source read at HEAD, editor not started; no `foliage.*` call was made. `grep -c FScopedTransaction Handlers/Environment/FoliageHandler.cpp` returns **0** over 1233 lines. Five mutating verbs, each cited above with its registration line and its mutation site; note the file lives under `Handlers/Environment/`, not `Handlers/Foliage/`. Correcting the premise this was filed from: it is **not** true that all five call a bare `IFA->Modify()` — only `paint` (`:240`), `remove` (`:343`, `:349`) and `add_instances` (`:898`) do, and all three call it *after* the mutation; `add_type` and `create_procedural` contain no `Modify()` of any kind, which is strictly worse. A second claim was checked and dropped rather than filed: `create_procedural`'s bare `World->SpawnActor` looked like it would leave the level clean per the `LevelActor.cpp:735-739` mechanism this plugin already documents, but `SpawnActorInActiveWorld` calls `SetActorLabel` on the spawned actor (`Utils/AssetUtils.h:611`) and the verb does pass a label, so the package dirties anyway — persistence is intact, incidentally, and only undo is actually lost. Found while filing `B-foliage-paint-does-no-ground-projection` (OPEN, High) and deliberately left out of it as a namespace-wide gap needing its own ticket — recorded there under "Adjacent, not folded in" and in that ticket's `#1` history. Dedup: ripgrep across every board file for `FScopedTransaction`, `transaction`, `undo` and `Modify()`. `B-no-undo-redo` (DONE, High) is the closest and does **not** cover this — read in full, its body names `compile_bpir`, `insert_bpir_at_node`, `widget_import_xml`, `blueprint_graph_delete_node`, and its ten history entries wrap only the widget (`#2`), Blueprint/BPIR/SCS (`#3`) and widget-XML (`#5`-`#10`) handler families; foliage was never in scope, and `E-sequencer-add-camera-track-no-transaction` already relies on that same reading to file a non-duplicate sequencer ticket. `B-ism-undo-record-unsafe` (OPEN, High) is the ISM/HISM per-instance verbs in `Handlers/Actor/InstancedMeshHandler.cpp` and `Handlers/Spatial/GroundPlacementHandler.cpp` — a different namespace with a *deliberate* no-transaction design and an echoed `movedInstances[]` record; foliage has no such substitute, it simply has nothing. Its transaction-cost arithmetic is reused above. `E-sequencer-add-camera-track-no-transaction` (OPEN, Low) is one sequencer verb. `B-compile-bpir-transaction-ensure`, `B-undo-last-bpir-doesnt-restore-phase0-sweeps` and `E-add-event-then-default-compile-bpir-unundoable` are BPIR-only. `B-property-set-markdirty-false-still-dirties`, `B-niagara-refused-edit-dirties-package`, `B-landscape-get-heights-dirties-map` and `E-python-cannot-mark-package-dirty` are dirty-flag tickets in other namespaces. Every `B-foliage-*` / `E-foliage-*` / `F-foliage-*` file was checked: `B-foliage-create-procedural-empty-callback-noop`, `B-create-procedural-ignores-scale-and-normal-fields`, `E-foliage-get-instances-drops-scale`, `E-foliage-nested-input-schemas-undocumented`, `E-foliage-remove-silent-edge-inputs`, `E-foliage-add-type-auto-save-undocumented` and `F-foliage-namespace-has-no-behavioural-tests` are placement, readback, input-schema or docs defects; none mentions undo, transactions or level dirtying. `B-create-procedural-terrain-paints-nothing` (DONE) is `landscape.create_procedural_terrain`, a different verb in a different namespace. Reusing the existing symptom-family tag `mutator-no-undo-transaction`.
