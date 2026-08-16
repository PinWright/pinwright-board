---
id: B-set-component-properties-drops-warnings
title: "actor.set_component_properties collected PropertyWarnings and never emitted them — every per-property failure vanished under an unqualified success, contradicting the verb's own registered summary"
status: DONE
severity: High
category: bug
tags: [actor, components, warnings, misleading-success, silent-elision, response-honesty, doc-contract-drift]
---

# `set_component_properties` built a warnings array and dropped it on the floor

The handler collected per-property failures into a local `PropertyWarnings` array and then never
put it in the response. An unknown property name, a value the importer could not convert, or a
setter that declined all produced an **unqualified success** whose `applied` list was simply
shorter — which a caller cannot distinguish from having asked for less.

## The contradiction is quotable from the verb's own registration

`Handlers/Actor/ComponentHandler.cpp:185`:

```
"Set one or more UPROPERTY values on a named component of a placed actor. Mobility is applied
 first (so subsequent edits succeed); SimulatePhysics has dedicated path.
 Per-property failures returned in 'warnings'."
```

No `warnings` key was ever emitted. The sibling `actor.add_component` **does** emit exactly that
array (`ComponentHandler.cpp:171-176`), so the response contradicted both its own documentation and
its neighbour.

## Root cause

`PropertyWarnings` is declared at `ComponentHandler.cpp:237` and populated at three sites:

- `:300-303` — property not found: `PropertyWarnings.Add(FString::Printf(TEXT("Property not found: %s"), *Pair.Key));`
- `:317-319` — setter declined (site added later by the shadow-copy fix)
- `:323-325` — generic importer failure: `Failed to set %s: %s`

The pre-fix response built `applied` and went straight on to `AddActorVerification(Data, Found)`.
**Nothing removed a `warnings` field — there was never one.** The fix hunk is a pure insertion with
zero deleted lines.

## Fix (shipped `88f6ee4f`)

`ComponentHandler.cpp:342-354` now emits the array when non-empty:

```cpp
  if (PropertyWarnings.Num() > 0) {
    TArray<TSharedPtr<FJsonValue>> WarnArray;
    for (const FString &Warning : PropertyWarnings)
      WarnArray.Add(MakeShared<FJsonValueString>(Warning));
    Data->SetArrayField(TEXT("warnings"), WarnArray);
  }
```

## Verification

`Tests/Actor/TestComponentAssetPropertyWrite.cpp:410-458`,
`PinWright.actor.set_component_properties.PerPropertyFailuresAreReported`: sends one real property
(`CastShadow`) plus one bogus (`PinWrightNoSuchProperty`) in a single call and requires the bogus
one to appear in `warnings` and **not** in `applied` (`:437-455`). Suite reported by the fix commit
as 3795 performed / 3795 pass / 0 fail, queue drained, 0 ensures.

## Why this is its own ticket

Found in the same handler, in the same commit, as
`B-set-component-properties-staticmesh-shadow-corruption` — and deliberately filed apart. The
causes are unrelated (a missing response field vs. a bypassed engine setter), and only one of them
is fixed by routing asset writes. Recording them as one entry is how a two-defect bug gets closed
with one of them still live.

## Related

- `B-set-component-properties-staticmesh-shadow-corruption` — same handler, same commit, unrelated cause.
- `rpc-design.md` §1 "report only what happened" — this is the plain form of that rule's violation.

## History
- `#1-warnings-collected-never-emitted` `OPEN` reporter — `actor.set_component_properties` collected per-property failures into a local `PropertyWarnings` array (`ComponentHandler.cpp:237`, populated at `:300-303` for an unknown property name and `:323-325` for an importer failure) and never wrote it into the response. The call therefore returned an unqualified success with a silently shorter `applied` list, which a caller cannot tell apart from having asked for fewer properties. The verb's own registered summary at `ComponentHandler.cpp:185` already promised "Per-property failures returned in 'warnings'", and the sibling `actor.add_component` emits exactly that array at `:171-176` — so the response contradicted both its documentation and its neighbour. Confirmed to be an omission rather than a regression: the fix hunk is a pure insertion with zero deleted lines, so no `warnings` field was ever built and later removed.
- `#2-emitted` `IN-REVIEW` developer — Fixed in `88f6ee4f` alongside (but deliberately separate from) the StaticMesh shadow-copy defect in the same handler. `ComponentHandler.cpp:342-354` now emits `warnings` whenever `PropertyWarnings` is non-empty. The setter-declined path added by the sibling fix (`:317-319`) feeds the same array, so the newly-routed asset writes report their refusals through the channel the summary already promised rather than needing a second one.
- `#3-verified` `DONE` tester — `PinWright.actor.set_component_properties.PerPropertyFailuresAreReported` (`Tests/Actor/TestComponentAssetPropertyWrite.cpp:410-458`) drives one real property (`CastShadow`) and one bogus (`PinWrightNoSuchProperty`) through a single call and asserts the bogus one appears in `warnings` and is absent from `applied` (`:437-455`) — a failure-direction assertion, so it fails against the pre-fix response for the right reason. Suite reported by the fix commit as **3795 performed / 3795 pass / 0 fail**, queue drained, 0 ensures. Filed retroactively: discovery, fix and verification all landed inside one working day, which would otherwise have left no board record that this defect existed.
- `#4-fix-is-local-only-not-on-origin` `DONE` reporter — Same qualification as the sibling ticket: **`88f6ee4f` is committed locally and is NOT on `origin/master`** (plugin HEAD was ahead of origin by 5 at filing time, `88f6ee4f` among the unpushed). The verdict holds for this checkout; a fresh clone does not have the fix. Not pushed by the filer because the working tree carried another agent's live in-progress edit. Re-confirm the push before treating this as closed on any other host.
