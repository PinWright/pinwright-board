---
id: B-foliage-remove-empties-ledger-not-component
title: "`foliage.remove` empties FFoliageInfo::Instances and never touches the HISM, so instances keep rendering while the verb reports an exact count — and `foliage.get_instances` reads the same emptied array, so the mutator and its verification verb agree with each other and both are wrong"
status: DONE
severity: Critical
category: bug
tags: [foliage, remove, get_instances, silent-false-success, ledger-vs-component, hism, data-loss, readback-blind-spot, behavioural-test-passes-on-defect]
encounters: 2
lastSeen: 2026-08-30T16:00:00+05:00
---

# The mutator and its readback share one blind spot, so the response and the verification agree and neither describes the level

**Thesis: this is worse than a silent failure, because the verb that exists to check it fails the
same way.** `foliage.remove` empties the *bookkeeping ledger* (`FFoliageInfo::Instances`) and never
touches the component that draws anything. `foliage.get_instances` iterates that same array. So
after a removal the mutator says `instancesRemoved: 56`, the readback says `count: 0,
orphanedInstanceCount: 0`, and the level still holds and draws 56 instances. Every honesty surface
the namespace has is on the wrong side of the split.

## Measured live

Six scoped `foliage.remove` calls, each with a valid `foliageTypePath`, returned
`success: true, mode: "type"` with exact counts **56 / 84 / 60 / 54 / 12 / 10**.
`foliage.get_instances` afterwards returned `count: 0` and `orphanedInstanceCount: 0` for every one
of them. The live `UHierarchicalInstancedStaticMeshComponent::GetInstanceCount()` still reported the
same 56 / 84 / 60 / 54 / 12 / 10, and the instances were still drawn — the capture taken after the
removal is pixel-identical to the frame before it.

## Mechanism, confirmed in source

`Handlers/Environment/FoliageHandler.cpp` (registration `:592`):

- `removeAll` branch — counts, then discards: `:673-678`, the `Info.Instances.Empty()` at `:675`.
- scoped branch — `RemovedCount = Info->Instances.Num()` at `:682`, `Info->Instances.Empty()` at
  `:683`, `IFA->Modify()` *after* the write at `:684`.
- unconditional `success: true` / `instancesRemoved` / `mode`, `:687-694`.

**The counts are exact because they are read off the ledger at `:682` immediately before it is
discarded.** That is why the response looks so trustworthy: the number is a true statement about the
array the handler is about to throw away, and a false statement about the level.

`FFoliageInfo::Instances` is the ledger (`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/InstancedFoliage.h:283`,
comment "Editor-only placed instances"); the rendered instances live on `FFoliageInfo::GetComponent()`
(`InstancedFoliage.h:305`). The engine's removal API is
`FFoliageInfo::RemoveInstances(TArrayView<const int32>, bool RebuildFoliageTree)`
(`InstancedFoliage.h:338`), which forwards to `RemoveInstancesImpl`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:2422`) with a lambda calling
`Impl->RemoveInstance(Index)` (`:2419`). Per index, `RemoveInstancesImpl` does four things the
handler does none of:

| engine does | site |
|---|---|
| `RemoveFromBaseHash(InstanceIndex)` | `InstancedFoliage.cpp:2453` |
| `InstanceHash->RemoveInstance(...)` | `:2454` |
| `ImplementationFunc(Implementation.Get(), InstanceIndex)` — comment at `:2456` reads *"remove from the component"* | `:2457` |
| `SelectedIndices` / `ComponentHash` / swap-index fixups | `:2460`, `:2473-2486` |

## The add path *does* reach the component. That asymmetry is the tell

`foliage.add_instances` calls `Info->AddInstance(FoliageType, Instance, nullptr)`
(`FoliageHandler.cpp:1228`, and `:1232` on the create-info branch), which reaches
`FFoliageInfo::AddInstanceImpl` (`InstancedFoliage.cpp:2275-2293`) and its
`ImplementationFunc(Implementation.Get(), IFA, AddedInstance)` at `:2292`, under the engine comment
*"Add the instance to the component"* at `:2291`. Adds land on the HISM. Removes do not.

**The correction to the obvious framing:** it is *not* true that the plugin never calls
`FFoliageInfo::RemoveInstances`. It calls it twice, correctly, in this same file —
`FoliageHandler.cpp:481` and `:498`, both inside `foliage.paint`, to withdraw an instance whose
ground projection found nothing. Those call sites even carry a comment reasoning about index shuffle
("it is the last index, which `RemoveInstances` handles as a plain truncation"). So the file knows
the API, uses it, and documents its semantics — and `foliage.remove`, the verb whose entire job is
removal, is the one place that does not use it. That is a stronger fact than the absence would have
been, and it makes the fix a two-line swap rather than a design question.

## The readback shares the blind spot

`foliage.get_instances` (registration `FoliageHandler.cpp:705`) reads `Info->Instances` in both
branches:

- type-filtered: block `:791-798`, loop at `:793`;
- unfiltered: block `:814-826`, loop at `:819`;
- `count` is `InstancesArray.Num()` at `:832`;
- `orphanedInstanceCount` at `:836` counts only the null-key case at `:816`.

`GetInstanceCount()` appears **nowhere** in `FoliageHandler.cpp` — verified by repo-wide grep across
`Plugins/PinWright/Source`, which finds it in `InstancedMeshHandler.cpp`, `InstancedMeshUtils.h`,
`LevelAuditUtils.cpp`, `GroundPlacementHandler.cpp`, `GroundPlacementUtils.cpp`, `MeasureHandler.cpp`
and the PCG module, and in no foliage handler or foliage test. So `count: 0` and
`orphanedInstanceCount: 0` are both fully consistent with a populated HISM. There is no verb in the
namespace that can observe the defect.

## Why this is data loss and not merely a no-op

The divergence is not inert, and it does not stay a display bug.

1. **The engine's only reconciler resolves in the ledger's favour — destructively.**
   `FFoliageStaticMesh::Reapply` (`InstancedFoliage.cpp:1838-1869`) compares the two and, when the
   component holds more than the ledger, builds the surplus index list and calls
   `Component->RemoveInstances(InstancesToRemove)` (`:1859-1867`), then asserts
   `check(Component->GetInstanceCount() == Info->Instances.Num())` at `:1869`. Against an emptied
   ledger that deletes **every** still-drawn instance. `Reapply` is reached from
   `FFoliageStaticMesh::PostEditUndo` (`:1431`) and from the foliage-type change path (`:1584`), so
   the actual deletion is executed later, by an unrelated action, with nothing linking it back to
   the `foliage.remove` that caused it.
2. **Index identity desynchronises permanently.** `FFoliageInfo::AddInstance` appends at
   `Instances.Num()` (`InstancedFoliage.cpp:2328-2341` -> `AddInstanceImpl` `:2278`) while the
   component appends at its own count. After a `remove` + `add_instances` on one type, ledger index
   `i` and component index `i` name different instances — and `spatial.ground_instances` /
   `actor.set_instance_transforms` address the **component** by index while `foliage.get_instances`
   reports the **ledger**. Two subsystems then disagree about what index 0 is.
3. **Saving persists the divergence.** Both sides are serialized: the ledger on `FFoliageInfo`, the
   transforms on the component. `FFoliageInfo::PostLoad` (`:2066-2072`) forwards to the
   implementation and performs no count reconciliation, so a reload does not repair it either way.

**Honest counterweight, which is why this is not filed as a crash:** `FFoliageInfo::CheckValid()`
(`:2194-2209`) does assert `check(Instances.Num() == Implementation->GetInstanceCount())` at `:2200`
and is called from `AddInstancesImpl` (`:2323`) and `RemoveInstancesImpl` (`:2510`) — but it is
wrapped in `#if DO_FOLIAGE_CHECK`, and `DO_FOLIAGE_CHECK` is `0` (`InstancedFoliage.cpp:67`). So the
invariant this handler breaks is one the engine states explicitly and does not enforce in any
shipping configuration. No crash. The loss is delayed, not immediate.

## The existing behavioural test passes on the defect

`Source/PinWright/Private/Tests/Environment/TestFoliagePlacementBehaviour.cpp` has
`PinWright.foliage.remove.ScopedRemovalSparesOtherTypes` at `:710-822`. It labels its central
assertion **"GROUND TRUTH on both sides of the scope boundary"** (`:810`) and asserts
`GatherInstances(World, TargetType, TargetStored) == 0` at `:811-812`. But `GatherInstances`
(`:176-195`) reads `Info->Instances` at `:190` — the array under test. Its own header comment claims
the opposite of what it does: *"Every instance the level actually holds for this type, read off
`FFoliageInfo::Instances` across every `AInstancedFoliageActor` in the world"* (`:173-175`). The
array is named correctly and described as "what the level actually holds", which it is not.

The test author reasoned about the adjacent failure and missed this one. Counterfactual (b), stated
at `:706-708`:

> (b) A handler that computed `instancesRemoved` from the info's count but did not empty the array
> reports the same number while every instance is still there, and the store re-read catches it.

The handler **does** empty the array. It is the component it leaves alone, and no assertion anywhere
in the file reads `GetInstanceCount()`. The counterfactual is one layer short.

`Tests/Environment/TestFoliageRemoveEdgeInputHonesty.cpp` (100 lines, two tests) asserts only error
codes on bad input and reads no store at all, so it does not cover this either.

## Contradicted claim on a sibling ticket (cross-link, do not fold)

`E-foliage-remove-silent-edge-inputs` (IN-REVIEW, Medium) is built on the happy path being correct:

> works correctly on its two happy paths — a valid `foliageTypePath` removes ONLY that type's
> instances (replay: 4 rocks removed, 3 trees left intact), and `removeAll:true` clears every type

The replay that proved it was `foliage.get_instances`, which reads the emptied ledger, so it would
have reported exactly that on a level where all seven were still drawn. Neither happy path is
correct. That ticket's own subject — undifferentiated responses on *edge* inputs — is unaffected and
still worth fixing; this ticket says its baseline is wrong, not its ask.

## Fix

Replace both `Instances.Empty()` calls with `FFoliageInfo::RemoveInstances`, exactly as
`foliage.paint` already does at `:481` / `:498`:

```cpp
TArray<int32> All;
All.Reserve(Info->Instances.Num());
for (int32 i = 0; i < Info->Instances.Num(); ++i) { All.Add(i); }
RemovedCount = All.Num();
Info->RemoveInstances(All, /*RebuildFoliageTree*/ true);
```

`IFA->Modify()` must move *before* the write (it is after it at `:678` and `:684`), though
`RemoveInstancesImpl` calls `IFA->Modify()` itself at `:2432`.

Then make the readback capable of catching a regression: `foliage.get_instances` should report the
component's `GetInstanceCount()` alongside `count`, and the behavioural test's `GatherInstances`
helper must read the component, not `Info->Instances` — a differential assertion is the only shape
that fails on this defect, because every ledger-side assertion passes.

## Third field reproduction, on a different type, with a caller-side repair

Look-dev polish pass over `PW_VegetationTest`, 2026-08-29 (`Docs/map/vegetation-polish.md` § 5.1).
`foliage.remove {foliageTypePath: "/Game/Foliage/Auto_SM_Trees.SM_Trees"}` returned
`success: true, instancesRemoved: 16, existsAfter: true`. The live
`UInstancedStaticMeshComponent::GetInstanceCount()` on `FoliageInstancedStaticMeshComponent_31`,
read immediately afterwards through `python.execute`, was **still 16**. Same shape as `#1`, on a
type nobody in `#1` touched, in a different level — so the defect is not specific to the six types
or the world `#1` measured.

Two things this run adds.

**A caller-side repair exists and is one line: `comp.clear_instances()` on the component.** It is
worth stating precisely *because* it is not a fix. The two halves are exactly complementary — the
verb empties `FFoliageInfo::Instances` (`FoliageHandler.cpp:675`, `:683`) and never touches the
component; `clear_instances()` empties the component and never touches the ledger — so calling
both, in either order, lands on the consistent state a correct `FFoliageInfo::RemoveInstances`
would have produced in one call. That makes the divergence recoverable for a caller who already
knows about it, and changes nothing about the severity: the caller has no way to *learn* it,
because § *The readback shares the blind spot* still holds and `GetInstanceCount()` appears
nowhere in the foliage handler. A repair that only works if you have already read this ticket is
not a workaround the rubric credits.

**`existsAfter: true` came back here too**, alongside the exact count. Noted because it is a third
true-sounding field in the same response: the caller sees `success`, an exact `instancesRemoved`
and `existsAfter`, and all three are consistent with a component that still draws everything.

## Same shape as

`B-foliage-paint-does-no-ground-projection`'s § *Same shape as* states the class: *"the call
succeeds, every number it reports is correct, and the output is wrong because the deciding number
was never reported."* This is the sharper case — the deciding number is not merely unreported, it is
unreadable through any verb the namespace exposes, so the caller cannot obtain it even by asking.

## Cross-links

- `E-foliage-remove-silent-edge-inputs` (IN-REVIEW) — its stated-correct happy path is the thing
  this ticket contradicts, quoted above.
- `B-foliage-get-instances-null-type-deref` (IN-REVIEW) — source of the `orphanedInstanceCount`
  field this evidence quotes; that field is also ledger-derived (`:816`).
- `B-foliage-mutators-no-transaction` (OPEN) — indicts `:678` / `:684` (the misplaced `IFA->Modify()`)
  for a different property. Both fixes touch the same two lines; whoever lands second should read
  the other.
- `F-foliage-namespace-has-no-behavioural-tests` (IN-REVIEW) — **live counter-example to its fix.**
  The behavioural tests it added assert against the ledger, so the coverage it claims does not reach
  the component. Its premise ("tested for what it says, never for what it does") is right and its
  remedy landed one layer above the thing that does anything.
- `F-ism-per-instance-transforms` (IN-REVIEW) — the index-desync consequence above lands on its
  verbs, which address the component while foliage reports the ledger.
- `B-tests-wipe-host-map-foliage` (IN-REVIEW, High) — sting: those test teardowns called
  `foliage.remove {removeAll:true}`, so they were **ledger-only too**. The host map's foliage may
  still be sitting on its components under an empty ledger, invisible to `foliage.get_instances`
  and one `PostEditUndo` away from real deletion. Worth re-checking the host map with
  `GetInstanceCount()` before that ticket is closed.

## Severity

**Critical.** Impact class per the rubric: *a write that corrupts or loses asset data*. The write
leaves the level's foliage ledger and its render state divergent, saving persists the divergence,
and the engine's only reconciler (`Reapply`, `:1859-1867`) resolves it by deleting the surviving
instances — later, from `PostEditUndo`, with no trace back to the verb that caused it. The
false-success half alone would be High; the two-sided agreement (mutator and readback both wrong,
and no verb in the namespace able to see it) removes the caller's detection route entirely.

**Reach modifier declined, in both directions.** `foliage.remove` is not an every-session verb, so
no upward bump. I also decline the downward "rare edge path" bump: this is not an edge path, it is
*both* of the verb's documented happy paths — the verb has no unaffected path. Critical rests on
impact alone.

**What was checked before rating, and what it cost.** The crash reading was tested and abandoned:
`CheckValid`'s `check(Instances.Num() == Implementation->GetInstanceCount())` is real
(`InstancedFoliage.cpp:2200`) but compiled out (`DO_FOLIAGE_CHECK 0`, `:67`), so there is no crash
and this is not Critical-by-crash. It is Critical by the data-loss clause, via `Reapply`.

**What would settle the remaining uncertainty:** whether a save-and-reload of an affected level, with
no intervening undo or foliage-type edit, leaves both sides intact (divergence dormant) or trips
`Reapply` during load. `FFoliageInfo::PostLoad` (`:2066-2072`) does not reconcile, which suggests
dormant — but `AInstancedFoliageActor::PostLoad` calls `ReallocateClusters` on two repair paths
(`:4605`, `:4613`) and that path is not read here. If a plain reload turns out to drop the
instances, the loss is immediate rather than delayed and Critical is unarguable; if it stays
dormant, Critical still holds on the undo path but the window is longer.

## History
- `#1-ledger-vs-component` `OPEN` reporter — Six scoped removals reported exact counts 56/84/60/54/12/10
  with `success:true`; `foliage.get_instances` reported `count:0, orphanedInstanceCount:0`;
  `GetInstanceCount()` still reported the same six numbers and the post-removal capture is
  pixel-identical to the pre-removal frame. Mechanism confirmed in source: `Info.Instances.Empty()`
  at `FoliageHandler.cpp:675` and `:683` with no call into the component, against an add path that
  does reach it (`:1228`/`:1232` -> `AddInstanceImpl` `:2292`). Readback confirmed to read the same
  array (`:793`, `:819`, `:832`, `:836`); `GetInstanceCount()` occurs nowhere in the file.
  **Brief correction recorded:** the plugin *does* call `FFoliageInfo::RemoveInstances` — twice, at
  `:481` and `:498` inside `foliage.paint` — so the fix is a two-line swap to an API this file
  already uses. **Crash reading rejected:** `CheckValid`'s invariant assert is compiled out
  (`DO_FOLIAGE_CHECK 0`). Rated Critical on the data-loss clause via `FFoliageStaticMesh::Reapply`
  (`:1859-1867`), which trims the component to the emptied ledger from `PostEditUndo` (`:1431`).
- `#2-third-repro-and-clear-instances-repair` `OPEN` reporter — Third field reproduction, the
  first from outside the run that filed this ticket: look-dev polish pass over `PW_VegetationTest`
  (`Docs/map/vegetation-polish.md` § 5.1). `foliage.remove
  {foliageTypePath:"/Game/Foliage/Auto_SM_Trees.SM_Trees"}` returned `success:true,
  instancesRemoved:16, existsAfter:true`, while `GetInstanceCount()` on
  `FoliageInstancedStaticMeshComponent_31` read back **16** immediately afterwards through
  `python.execute`. Different type, different level, so the defect is not specific to `#1`'s six
  types or its world. New in this encounter: a **caller-side repair**, `comp.clear_instances()` on
  the component — exactly complementary to what the verb does, so calling both lands the
  consistent state `FFoliageInfo::RemoveInstances` would have produced alone. Recorded in the body
  as recoverable and explicitly NOT as a workaround, because the readback blind spot means a
  caller cannot learn it exists; severity unchanged at **Critical** for that reason (the data-loss
  clause via `Reapply` `InstancedFoliage.cpp:1859-1867` is untouched by a repair nobody can
  discover). Re-derived at HEAD rather than inherited: the ledger-only writes are still
  `FoliageHandler.cpp:675` (removeAll) and `:683` (scoped), `instancesRemoved` is still emitted at
  `:689`, and `FFoliageInfo::RemoveInstances` still has exactly two call sites, both in
  `foliage.paint` (`:481`, `:498`). Split out rather than folded in: the same run found that
  `foliage.remove` **rejects `mode` as `UNKNOWN_PARAMS` while emitting `"mode"` in its own
  response** (`:694` against the registered param list at `:592-596`), filed separately as
  `E-foliage-remove-mode-is-output-only` because it is naming friction on a verb whose real
  defect is this one. `encounters` 1 → 2.
- `#3-removeinstances-and-differential-readback` `IN-REVIEW` developer — Every claim re-verified at
  HEAD before editing; the stale line numbers still landed exactly (`Info.Instances.Empty()` at
  `FoliageHandler.cpp:675`/`:683`, `RemoveInstances` only at `:481`/`:498`, `GetInstanceCount()`
  absent from the file). **Fix.** Both branches now go through one helper,
  `RemoveAllFoliageInstances(FFoliageInfo&)`, which builds `[0..N-1]` and calls
  `FFoliageInfo::RemoveInstances(All, RebuildFoliageTree=true)`; `IFA->Modify()` moved before the
  write in both. **One correction to the ticket's two-line-swap fix, verified in source:**
  `RemoveInstancesImpl` opens with `check(IsInitialized())` (`InstancedFoliage.cpp:2431`) — the
  proposed unconditional swap crashes on an info holding instances with no implementation, which is
  a state the engine names and repairs at `AInstancedFoliageActor::PostLoad:4588`. The helper guards
  on `IsInitialized()` and falls back to `Instances.Empty()` there, which is the *complete* removal
  in that case because nothing is drawn. Also note `RemoveInstancesImpl`'s `Num() <= 0` early return
  precedes the check, so the zero case was never at risk. **Readback.** `foliage.get_instances` now
  emits `renderedInstanceCount` (summed `Info.Implementation->GetInstanceCount()` over the same
  scope, orphaned infos included) and `ledgerMatchesRendered`
  (`renderedInstanceCount == count + orphanedInstanceCount`). `Implementation->GetInstanceCount()`
  rather than the ticket's `GetComponent()->GetInstanceCount()`: `GetComponent()`
  (`InstancedFoliage.cpp:2039`) returns null for every non-static-mesh impl, which would report
  "draws nothing" for actor foliage. The two cannot legitimately diverge — `CheckValid:2200` states
  the equality — so the verb reports the verdict rather than picking a side. **Today's compromised
  tests.** `TestFoliagePlacementBehaviour.cpp`: added `GatherRenderedInstances` (reads
  `Info->GetComponent()->GetInstanceCount()`) and `RequireInstanceCount`, which pins record AND
  rendered to the same number; all 9 count assertions across the 7 tests re-pointed through it, so
  each now fails on this defect class. `GatherInstances` kept for transforms only (the component
  cannot answer an `FFoliageInstance`) with its "what the level actually holds" comment corrected,
  and counterfactual (b) extended with the (b') this ticket names. New regression
  `PinWright.foliage.remove.RemovalReachesTheRenderedInstances` drives add → read → remove → read
  and asserts `GatherRenderedInstances == 0` off the component; it fails before the fix (reads 3).
  `TestEnvironmentHandlers.cpp` needed **no change**: its three teardowns are already scoped by
  `foliageTypePath` (no `removeAll:true` remains anywhere under `Tests/`), and the handler fix is
  what makes them reach the component — they were leaking into the shared host map only because
  `remove` was half-done. **Residual, for whoever closes `B-tests-wipe-host-map-foliage`:** host
  maps run against the broken build may still carry component instances under an emptied record;
  `ledgerMatchesRendered:false` from `foliage.get_instances` is now the way to see it. Wiki overlay
  `Docs/wiki-src/foliage.md` updated for both verbs. Not compiled or run — build and suite are the
  wave owner's.
- `#4-verified-fixed-21-37-measured-13-32-proven-identical` `DONE` verifier — 2026-08-30. **Closed on the 21:37 build's live measurement plus a source-identity proof that the 13:32 build runs the same code, NOT on source reading alone — and the split is stated because the two halves have different authors and different strength.**

  **The measurement half (21:37 build, plugin `2ba21649`), relayed from the independent verification pass that ran it, attributed rather than asserted:** `foliage.remove` reported `instancesRemoved: 14`; `foliage.get_instances` then returned `count: 0, renderedInstanceCount: 0, ledgerMatchesRendered: true`; a live `GetInstanceCount()` read through `python.execute` returned **0**; and a capture at a fixed pose moved 0.5612 → 0.6361 against a bare-meadow reference of 0.6359. That pass also proved `ledgerMatchesRendered` was not a constant by injecting a divergence and reading back `14/15/false`. **The capture is the signal that decides this ticket**, because it is the only one of the four that shares no representation with the defect: `#1`'s whole thesis is that the mutator and every readback in the namespace sit on the same side of the ledger/component split, and a pixel measurement sits on neither. `#1` recorded that the pre-fix capture was *pixel-identical* to the frame before the removal; a 0.075 move to within 0.0002 of the bare reference is that finding inverted.

  **The currency half, derived at HEAD in this session and checkable by anyone:** waves 2 and 3 (`d8f1bc32`) added ~1000 lines to `FoliageHandler.cpp`, so the 21:37 result does not transfer for free. It transfers because the code it exercised did not move. Extracted and diffed `2ba21649` against HEAD `1a9e5778`: the whole `foliage.remove` handler body (113 lines, `// ---- foliage.remove ----` through `// ---- foliage.get_instances ----`) is **byte-identical**; so is `RemoveAllFoliageInstances`; so is `CountRenderedFoliageInstances`; so is `GetOrCreateFoliageActorForWorldSafe`, the only other plugin function the handler calls. That is the verb's entire in-plugin call graph. The `renderedInstanceCount` accumulation is the sole delta and it is a hoist into a named local (`DrawnForInfo`) to feed the new `expectedDrawnInstances` field — the summed expression is unchanged. At HEAD the fix reads `RemoveAllFoliageInstances` at `FoliageHandler.cpp:1106-1128` calling `FFoliageInfo::RemoveInstances(AllIndices, /*RebuildFoliageTree*/ true)` at `:1126`, from `:1216` (removeAll) and `:1223` (scoped); the readback is `CountRenderedFoliageInstances` at `:1094-1096` feeding `renderedInstanceCount` at `:1464` and `ledgerMatchesRendered` at `:1465-1466`.

  **What this session did NOT manage, stated so nobody reads more into the close than is there.** A fresh 13:32 behavioural re-run was attempted and never completed: the scratch foliage type it needed was to be created with `foliage.add_type {name: "/Game/PinWrightScratch/..."}`, which **killed the editor process** — an unrelated Critical filed as `B-foliage-add-type-name-with-slash-kills-the-editor`, and the editor could not be restarted in this session. So there is no 13:32 measurement of this verb. The close rests on the identity proof above; if a reader rejects that argument, the correct action is to re-open and re-run the four-signal sequence, not to trust a spot check.

  **The verification field this fix shipped is indicted, and that does not weaken the removal close — the reason is worth stating exactly.** `ledgerMatchesRendered` compares `count + orphanedInstanceCount` against `CountRenderedFoliageInstances`, which is `Info.Implementation->GetInstanceCount()` (`:1095`). For a HISM that resolves `FFoliageStaticMesh::GetInstanceCount` (`C:/UE_5.8/.../InstancedFoliage.cpp:1188-1195`) → `UInstancedStaticMeshComponent::GetInstanceCount` (`.../InstancedStaticMesh.cpp:4844-4847`) → **`PerInstanceSMData.Num()`** — a second array, not the built cluster tree (`NumBuiltInstances`). `B-foliage-adds-never-rebuild-the-hism-tree` (High) is right that this makes the field structurally unable to be false on the **add** path, where one call appends to both arrays together. It is still a real signal on the **remove** path, and that is precisely what `#1` measured: the pre-fix verb left `PerInstanceSMData` populated at 56/84/60/54/12/10 while the record read 0, so a post-fix 0 from that same array is a genuine observation that the component array was emptied. **A second tautology that ticket does not name, found at HEAD here:** `FFoliageISMActor::GetInstanceCount()` returns `Info->Instances.Num()` (`.../FoliageISMActor.cpp:237-240`), so for ISM-actor foliage `ledgerMatchesRendered` compares the ledger with itself and is true by construction — a stronger statement than the `PerInstanceSMData` one, and it belongs on that ticket. Recorded here and cross-linked; that file is deliberately not edited.

  **Residuals left open on purpose.** `B-tests-wipe-host-map-foliage` — `#3`'s residual stands untouched: host maps exercised against the broken build may still carry component instances under an emptied record, and `ledgerMatchesRendered:false` is the way to find them; not checked here. `E-foliage-remove-silent-edge-inputs` — this ticket contradicted its "happy path is correct" premise; that premise is now true again, so its remaining ask (undifferentiated responses on edge inputs) is unaffected and its file is untouched. The `#3` suite run is still the wave owner's: nothing was compiled or run in this session.
- `#5-directly-measured-on-the-1332-build` `DONE` verifier — Same session as `#4`, after the editor came back up as **pid 18592 on the 13:32 build**. `#4` closed this on the 21:37 measurement plus a build-identity proof because the editor was down; that indirection is no longer needed. **The sequence `#4` said it could not run was run, on the build the close claims:** scratch type `/Game/Foliage/PWScratchRemoveVerify1332` over `SM_SphereProbe`, five instances added at `z = 200000` (well away from the level's vegetation, nothing of the map's own foliage touched). Before: `foliage.get_instances` → `count: 5, renderedInstanceCount: 5, ledgerMatchesRendered: true`. `foliage.remove {foliageTypePath}` → `instancesRemoved: 5`. After: `count: 0, renderedInstanceCount: 0, ledgerMatchesRendered: true`.

  **The deciding signal is none of those.** A direct `python.execute` read of the live component — `FoliageInstancedStaticMeshComponent_60`, located by its `static_mesh` being `SM_SphereProbe` and read through `get_instance_count()`, touching no `foliage.*` verb at all — returned **5 before the removal and 0 after**. That is `#1`'s exact instrument: `#1` recorded the same read returning 56/84/60/54/12/10 against a record of 0, and `#2` recorded 16 against a record of 0. It now tracks the record in both directions. `#4`'s byte-identity argument stands and is now redundant rather than load-bearing.

  **Two housekeeping facts for whoever works this level next.** (1) The scratch type never reached disk (`existsOnDisk: false`; `Content/Foliage/` has no such file) but `asset.delete` correctly refused it `ASSET_IN_USE` — `InstancedFoliageActor_0` still holds the now-empty `FFoliageInfo` for it, and the foliage namespace exposes no verb to drop a type from an actor (`AInstancedFoliageActor` has no `remove_foliage_type` on the Python surface either, checked). `force:true` was NOT used. The residue is an empty info pointing at an in-memory-only package; it disappears on editor restart, and it is a small ergonomic gap worth its own ticket if anyone else trips on it. (2) `expectedDrawnInstances` (5 → 0, `effectiveDensityScale: 1`) behaved consistently throughout; it is not evidence for this ticket and is noted only so a reader does not mistake it for a fourth signal.
