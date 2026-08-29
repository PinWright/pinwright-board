---
id: B-property-set-object-hop-notification-noop
title: "property.set through a dotted path that crosses an OBJECT boundary notifies the outer object, so the inner object's PostEditChangeProperty override never runs — applied:true, markedDirty:true and a matching property.get all agree with a write nothing downstream ever saw"
status: IN-REVIEW
severity: High
category: bug
tags: [property, property-write, property-set, nested-path, object-hop, subobject, post-edit-change, derived-state, silent-noop, misleading-success, pcg, wrong-notification-target]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The right object was resolved, and a different object was notified

`property.set` resolves a dotted path by walking it. When a segment is an `FObjectProperty`
the walk **hops onto a different `UObject`** and keeps going —
`ResolveNestedPropertyPath` does `CurrentContainer = NextObject; CurrentTypeScope =
NextObject->GetClass();` (`Utils/PropertyInspection.cpp:336-337`, and the array-element
form at `:193-194`) — then returns the leaf `FProperty` together with
`OutContainerPtr = CurrentContainer` (`:318`), which is now the **inner** object.

The store therefore lands in the inner object's memory. The **notification does not
follow it.** `NotifyReflectedPropertyChanged` (`Handlers/Utility/UtilityPropertyHandler.cpp:292`)
dispatches to `RootObject`, the object the path started from:

```cpp
// Handlers/Utility/UtilityPropertyHandler.cpp:300-303
    PinWright::NotifyPropertyChanged(RootObject, Property,
        ResolveMemberPropertyForPath(RootObject, PropertyName), ChangeType);

    PushRenderStateForComponentTarget(RootObject);
```

`PinWright::NotifyPropertyChanged` (`Utils/PropertyChangeNotify.h:88`) builds the event and
calls `Object->PostEditChangeProperty(ChangedEvent)` at `:103` — on `RootObject`. The inner
object, which is where the value now lives and whose override exists to recompute from it, is
never told anything. `PushRenderStateForComponentTarget` at `:303` has the same target and the
same blind spot.

Nothing here fails. `NotifyPropertyChanged` returns `true`; the event is well-formed and names
a real member of the notified object; `applied`, `markedDirty` and `pendingSave` are all
truthful about what they measure.

## The design comment is right for one hop kind and wrong for the other

`ResolveMemberPropertyForPath` (`Handlers/Utility/UtilityPropertyHandler.cpp:222`) takes the
FIRST path segment off `RootObject`'s class, and says why at `:219-221`:

> Resolution deliberately stops at the FIRST segment: the notification is dispatched to
> RootObject, so the member half of the event has to be a property RootObject's own class
> declares. Deeper hops are already described by the leaf.

For a **struct** hop that reasoning is correct and the code is correct. `BodyInstance.CollisionEnabled`,
`CustomPrimitiveData.Data`, `LightingChannels.bChannel0` — the value lives inside `RootObject`'s
own memory, `RootObject` is the object whose override must run, and `BodyInstance` /
`CustomPrimitiveData` / `LightingChannels` genuinely are properties `RootObject`'s class
declares. RootObject-relative member resolution is the right answer, and it is measured working
(`B-property-set-container-empty-change-event` `#7`).

For an **object** hop it is wrong, and it is wrong in the premise rather than the execution.
The value does not live on `RootObject`; it lives on a different `UObject`, and it is *that*
object's `PostEditChangeProperty` override that has to run. Dispatching to the outer object
cannot reach it, whatever the event says. "Deeper hops are already described by the leaf"
describes the *event*, and is true — but the defect is not in the event's contents, it is in
**who receives it**.

Worse, on an object hop the first segment usually *does* resolve, so nothing looks degraded:
`SettingsInterface` is a real `UPROPERTY` on `UPCGNode`
(`C:\UE_5.8\Engine\Plugins\PCG\Source\PCG\Public\PCGNode.h:242-243`,
`TObjectPtr<UPCGSettingsInterface> SettingsInterface`). The event that reaches the node is a
properly-populated `FPropertyChangedEvent` naming a real member and a real leaf. It is simply
addressed to the wrong object, and there is no branch anywhere that notices.

## Repro — measured

On a PCG graph node, through `property.set`:

| # | call | response | `pcg.generate` instances |
|---|---|---|---|
| 1 | `property.set SettingsInterface.LowerBound = 0.9` (path rooted at the **node**) | `applied: true`, `markedDirty: true`; follow-up `property.get` returns **0.9** | **26664** — unchanged |
| 2 | same value written directly against the inner object (`…DensityFilter_12.PCGDensityFilterSettings_12`) | — | **2093** |

Row 1 is the defect in one line: three independent honest-looking signals — the verb's
`applied`, the verb's `markedDirty`, and a *separate* `property.get` round trip — all agree with
a write that changed nothing downstream. Row 2 is the control that proves the value itself is
the right lever and 0.9 really is a 12.7x cut: the same number, written one object further in,
moves the output.

The read-back cannot catch this and never will. `property.get` resolves through
`ResolvePropertyFromObject` (`Handlers/Utility/UtilityPropertyHandler.cpp:1729`) — the same
call `property.set` makes at `:1238`, hence the same walk to the same inner container. It
reports the inner object's memory faithfully. Memory is exactly the half that is correct.

### Engine trace for row 1

- `UPCGNode::PostEditChangeProperty` (`C:\UE_5.8\Engine\Plugins\PCG\Source\PCG\Private\PCGNode.cpp:859-867`)
  branches on one name only — `GET_MEMBER_NAME_CHECKED(UPCGNode, NodeTitle)` at `:863`. A
  `LowerBound` leaf with a `SettingsInterface` member matches neither, so the node's override
  runs and does nothing.
- The regeneration trigger is on the settings object, not the node:
  `UPCGSettings::PostEditChangeProperty` (`C:\UE_5.8\Engine\Plugins\PCG\Source\PCG\Private\PCGSettings.cpp:719`)
  computes a change type and calls `OnSettingsChangedDelegate.Broadcast(this, ChangeType)` at
  `:744`, which is what `UPCGNode::OnSettingsChanged` (`PCGNode.cpp:869`) is subscribed to.
  That object was never notified, so the delegate never fires and the graph is never dirtied.
- `UPCGDensityFilterSettings` is a `UPCGSettings`
  (`C:\UE_5.8\Engine\Plugins\PCG\Source\PCG\Public\Elements\PCGDensityFilter.h:10`) and
  `LowerBound` is a plain float on it (`:31`) with no setter of its own — so the delegate is
  the only route out.

**Not PCG-specific.** Any subsystem that recomputes inside an inner object's
`PostEditChangeProperty` override has this hole. PCG is where it was caught because it
publishes a number that moves.

## Contradicted cause — recorded here, the other ticket's text left alone

`B-property-set-container-empty-change-event` (**DONE**, High, `encounters: 3`) is where
`ResolveMemberPropertyForPath` was introduced, and its `#5-named-event-and-render-state-push`
states the design this ticket contradicts:

> Member resolution is a new `ResolveMemberPropertyForPath` that takes the first path segment
> (stripping `.` and `[N]`) off `RootObject`'s class — deliberately RootObject-relative, since
> the notification target is unchanged.

"since the notification target is unchanged" is the load-bearing clause, and it is exactly the
assumption that fails on an object hop: the notification target *should* have changed, because
the write target did. Nothing is edited on that ticket — this is the correction, filed where it
can be seen.

**Its DONE was earned on a case class that excludes this one.** That is a statement about
coverage, not a claim the fix was wrong. Read against its own verification entries:

- `#6-named-event-half-measured-fixed-render-half-undecided` grades `LDMaxDrawDistance` (a leaf
  directly on the notified component), `property.reset LDMaxDrawDistance` (same), and
  `container.array.append CustomPrimitiveData.Data`. `CustomPrimitiveData` is an
  `FCustomPrimitiveData` **struct** on `UPrimitiveComponent`.
- `#7-render-half-decided-by-pixels` re-runs those three with different values and adds the
  pixel measurement on `LightingChannels.bChannel0`. `LightingChannels` is an
  `FLightingChannels` **struct** on `UPrimitiveComponent`.

So both dotted paths ever graded are struct hops, and the two single-segment paths are leaves
on the root. **No grade in `#6` or `#7` crosses an object boundary.** Every one of them is a
case where RootObject-relative resolution is the correct answer, which is why they all passed
and why this shape survived a fix, two verification passes and a pixel measurement.

## Related, and how each is different

- `B-property-set-container-empty-change-event` (DONE, High) — origin of the helper and of the
  contradicted sentence. That was the *empty* event: right object, no usable event. This is a
  *populated* event delivered to the wrong object. Its fix is a precondition for this one, not a
  competitor: a correctly-named event addressed to the correct object is what the pair adds up to.
- `F-pcg-set-node-property` (OPEN, High) — its `#1` argued Low partly because "a PCG node's
  `UPCGSettings` sub-object has a real object path surfaced by `pcg.inspect`, so editing node
  settings is reachable today (if unpolished)". `#2-re-severity-low-to-high` already demolished
  the "surfaced by `pcg.inspect`" half (the verb emits a *class* path). This finding demolishes
  the remainder: even with a hand-built path, a write **through the hop** is inert, so the
  fallback route is not merely unpolished, it is silently ineffective. **This constrains its
  proposed implementation**, which currently reads "resolve the node's `UPCGSettings`, write the
  reflected property via the shared property-write path, `PostEditChange`": the `PostEditChange`
  must land on the **settings object**, not on the node. A `pcg.set_node_property` that resolves
  the node and then notifies the node reproduces this defect behind a friendlier name.
- `E-property-set-no-measured-render-state` (**WONTFIX**, Medium) — closed with "Reopen with a
  concrete case where `markedDirty` plus a read is demonstrably insufficient." **This is that
  case**, and it is stronger than the render-state framing that ticket was closed on: here
  `markedDirty` is true, the read agrees, and the caller is still wrong — and there is no
  renderer in the loop at all, so "a render-state probe is disproportionate" does not answer it.
  **Recorded as a reopen candidate; that file is deliberately untouched.** Whoever reopens it
  should note the ask generalises from `renderStateRefreshed` to *"which object did the
  notification actually reach"*, which is cheap to report and not a probe.
- `B-property-path-silent-world-fallback` (IN-REVIEW, High) — **distinct, do not merge.** There
  the resolver *fails* and silently substitutes an ancestor (the World), so the wrong object is
  read *and* reported: `objectPath`/`className` in the response change. Here resolution
  **succeeds**, the correct inner object is written, and the response is accurate about
  everything it names — the wrong object appears only as the notification target, which no
  response field mentions. Wrong-object-on-failure versus right-object-resolved,
  wrong-object-**notified**. A fix for either leaves the other standing.
- `B-set-component-properties-no-change-notification` (DONE, High) — the absent-notification
  sibling on `actor.set_component_properties`. Not a workaround for this one: it takes a
  component and a property name, so it cannot express a path that crosses into a sub-object at all.

## Same shape as

`B-foliage-paint-does-no-ground-projection` — the fullest statement of the class: the call
succeeds, every number it reports is correct, and the output is wrong because the deciding fact
was never reported. Nearest members here: `B-ground-probe-hits-hull-not-render`,
`B-niagara-validate-green-while-component-inactive`,
`B-mrq-render-result-omits-bitrate-and-size`.

What this one adds to the class is that the misleading signal is **corroborated**: a second,
independent verb (`property.get`) confirms the write. The usual defence against a lying
response — read it back — is the thing that fails hardest, because the read walks the same path
to the same memory and memory is the half that is right.

## Fix

The notification target must be **the object that owns the leaf property**, so the traversal
that already knows this has to say so. `ResolveNestedPropertyPath` computes the inner object at
`Utils/PropertyInspection.cpp:336` and then discards its identity — it returns
`OutContainerPtr` as an opaque `void*`, which is a struct offset for a struct hop and a
`UObject*` for an object hop, indistinguishable to the caller. The fix has to surface that
distinction, not just move a call.

What it has to decide:

- **Where the notification goes.** The last object crossed on the walk, defaulting to
  `RootObject` when no `FObjectProperty` was crossed. That means an out-parameter (an
  `OutOwningObject`, or a small resolved-path struct) threaded from the resolver back to
  `NotifyReflectedPropertyChanged` — the current signature takes only `RootObject` and a path
  string (`Handlers/Utility/UtilityPropertyHandler.cpp:292`) and cannot recover it.
- **What the member half is then relative to.** The existing comment's reasoning at `:219-221`
  still holds *verbatim* — it just holds against the **inner** object. `ResolveMemberPropertyForPath`
  should take the first segment **after the final object hop** off the **notified** object's
  class. For `SettingsInterface.LowerBound` notified on the settings object, that member is
  `LowerBound` itself, i.e. leaf == member and `SetActiveMemberProperty` is correctly skipped
  (`Utils/PropertyChangeNotify.h:99-102`). For `SettingsInterface.SomeStruct.Field` it is
  `SomeStruct`.
- **Branch, do not replace.** Struct hops must keep today's behaviour exactly — this is a branch
  on whether the traversal crossed an `FObjectProperty`, and a path that crossed none must
  produce byte-identical behaviour to now, or it regresses the three grades in
  `B-property-set-container-empty-change-event` `#7`.
- **Both halves of the dispatch move, not one.** `PushRenderStateForComponentTarget` at `:303`
  takes the same target and is subject to the same argument: a path hopping into a component
  sub-object should push render state on *that* component. Moving the notification and leaving
  the render push on `RootObject` swaps one wrong target for two disagreeing ones.
- **`Modify()` too, probably.** `property.set` calls `RootObject->Modify(bMarkDirty)` at
  `Handlers/Utility/UtilityPropertyHandler.cpp:1245` before the write. On an object hop the
  memory being changed belongs to the inner object, so the transaction record covers the wrong
  object and undo will not restore the edit. **Not measured** — flagged as adjacent and likely,
  not asserted.

Acceptance must be a downstream-effect measurement, never a read-back: read-back is proven blind
to this defect above. Row 1/row 2 of the repro table is a ready-made shape — the same value
through the hop and directly on the inner object must produce the **same** `pcg.generate`
instance count.

## Honesty

- The PCG numbers (26664 / 2093, `applied: true`, `markedDirty: true`, `property.get` returning
  0.9) are **measured**, live, on this host.
- The generalisation beyond PCG — "any subsystem that recomputes in an inner object's override
  has this hole" — is **inference from the mechanism**, not a second measurement. **No non-PCG
  subsystem was tested.** The mechanism is source-read and cited above, and it does not depend on
  anything PCG-specific, but a reader should treat the PCG row as the evidence and the rest as
  the argument.
- The `Modify()`/undo consequence in § *Fix* is inference from the same source read, not observed.
- **No fix was attempted.** Nothing under `Plugins/PinWright/Source/` was edited for this ticket.

severity rationale: impact=High — silent false-success on a normal path: the write lands in the inner object's memory and every signal the caller can reach agrees with it (`applied:true`, `markedDirty:true`, and a *separate* `property.get` round trip returning the written value), while the override that gives the value its meaning is never invoked; the caller trusts three corroborating results and builds on a change that does not exist × reach=**both modifiers declined, and they are declined for different reasons**. The bump-**up** to Critical on "`property.set` runs in almost every session" is declined on the rubric's own wording — the modifier is about "the affected method", and `property.set` as a method is not affected: leaf writes and struct-hop writes are measured working (`B-property-set-container-empty-change-event` `#7` (a)-(d)). What is affected is `property.set` *when the path crosses an `FObjectProperty`*, which is the honest scope of the affected method here; and Critical's impact band is defined as crash or asset-data corruption/loss, so promoting a non-corrupting defect into the picker's pre-emption band on a reach argument would mis-order the queue against the rubric's own definition. The bump-**down** to Medium is declined because an object hop is not a "rare edge path" in the rubric's sense: it is one of exactly two hop kinds the resolver implements (`FObjectProperty`, `Utils/PropertyInspection.cpp:325`; `FStructProperty`, `:339`), it is the only path shape that reaches a settings sub-object without a hand-built object path, and it is the shape every settings-object target takes — and Medium's own definition ("doable, but only via a documented workaround") does not fit, because a caller cannot reach for a workaround they have no way to know they need: the two hop kinds are indistinguishable in every response field, and the read-back that would normally expose a bad write is structurally blind to this one -> High

## History
- `#1-object-hop-notifies-outer-object` `OPEN` reporter — Measured live on a PCG graph node: `property.set SettingsInterface.LowerBound = 0.9` returned `applied:true, markedDirty:true`, a follow-up `property.get` read back **0.9**, and `pcg.generate` produced **26664** instances — unchanged. The same value written directly against the inner object (`…DensityFilter_12.PCGDensityFilterSettings_12`) produced **2093**, so the value is the right lever and the hop is what swallowed it. Mechanism, source-read and every line re-derived in this tree: `ResolveNestedPropertyPath` hops onto the inner `UObject` on an `FObjectProperty` segment (`Utils/PropertyInspection.cpp:336-337`, array-element form `:193-194`) and returns it as `OutContainerPtr` (`:318`), so the store lands in the inner object; but `NotifyReflectedPropertyChanged` (`Handlers/Utility/UtilityPropertyHandler.cpp:292`) dispatches to `RootObject` at `:300-301` via `PinWright::NotifyPropertyChanged` (`Utils/PropertyChangeNotify.h:88`, `Object->PostEditChangeProperty(ChangedEvent)` at `:103`), and `PushRenderStateForComponentTarget` at `:303` has the same target. The event is well-formed and names a real member — `SettingsInterface` IS a `UPROPERTY` on `UPCGNode` (`C:\UE_5.8\…\PCGNode.h:242-243`) — it is simply addressed to the wrong object, so nothing errors and `NotifyPropertyChanged` returns true. Engine side: `UPCGNode::PostEditChangeProperty` (`PCGNode.cpp:859-867`) branches only on `NodeTitle` (`:863`), while the regeneration trigger `OnSettingsChangedDelegate.Broadcast` lives in `UPCGSettings::PostEditChangeProperty` (`PCGSettings.cpp:719`, `:744`) on the object that was never notified. The read-back cannot catch this: `property.get` uses the same `ResolvePropertyFromObject` (`UtilityPropertyHandler.cpp:1729`) that `property.set` uses (`:1238`), so it walks to the same inner container and reports memory faithfully — memory is the half that is correct. **Contradicts** `B-property-set-container-empty-change-event` `#5`'s "deliberately RootObject-relative, since the notification target is unchanged" and the in-source comment at `UtilityPropertyHandler.cpp:219-221` that states the same design; that ticket's text is deliberately NOT edited. Checked its verification entries: `#6` grades `LDMaxDrawDistance` (leaf on the notified component), `property.reset` of the same, and `CustomPrimitiveData.Data` (an `FCustomPrimitiveData` **struct** member on the root); `#7` re-runs those with different values and adds `LightingChannels.bChannel0` (an `FLightingChannels` **struct**). **No grade in either crosses an object boundary** — so its DONE was earned on a case class that excludes this one, which is a coverage statement and not a claim the fix was wrong: for struct hops RootObject-relative resolution is genuinely correct. Dedup: searched the board for object-hop, sub-object, nested-path and PCG-settings-write shapes. `B-property-set-object-array-silent-null` (IN-REVIEW) is an unresolvable object-array **element**, a resolution failure; `E-property-path-array-index-unsupported` (IN-REVIEW) is `[N]` subscripts returning `PROPERTY_NOT_FOUND`, also resolution; `B-property-path-silent-world-fallback` (IN-REVIEW) is wrong-object-on-resolution-**failure** and is named in the body so a triager does not merge it — this one resolves correctly and notifies wrongly. Constrains `F-pcg-set-node-property` (OPEN, High): its proposed "resolve the node's `UPCGSettings` … `PostEditChange`" must notify the **settings object**, or it reproduces this defect behind a new verb name; also finishes off its `#1` Low-priority rationale, whose surviving half (`property.set` on a hand-built path) is what this shows to be inert. `E-property-set-no-measured-render-state` (WONTFIX) asked to be reopened with "a concrete case where `markedDirty` plus a read is demonstrably insufficient" — this is that case, recorded as a reopen candidate only; that file was NOT touched. Severity High: impact is silent false-success with three corroborating honest signals, and BOTH reach modifiers are declined — Critical because the rubric's modifier is about "the affected method" and the affected method is the object-hop path rather than all of `property.set` (leaf and struct-hop writes are measured working), plus Critical's band is crash/corruption; Medium because an object hop is one of only two hop kinds the resolver implements and is the only route to a settings sub-object without a hand-built path, and Medium's "documented workaround" wording cannot apply when no response field distinguishes the two hop kinds. Honesty: the PCG numbers are measured, the generalisation to other subsystems is inference from the mechanism with **no non-PCG subsystem tested**, the `Modify()`/undo consequence in the Fix section is inference and not observed, and **no fix was attempted** — nothing under `Plugins/PinWright/Source/` was edited.
- `#2-notification-retargeted-to-the-owning-object` `IN-REVIEW` developer — **Fixed as a retarget, not a refusal: what the engine needs here is a plain non-chain `PostEditChangeProperty` on the inner object, which is exactly what the plugin already emits — only the target was wrong.** Re-derived from source before acting: `UPCGSettings::PostEditChangeProperty` (`C:\UE_5.8\Engine\Plugins\PCG\Source\PCG\Private\PCGSettings.cpp:718`) is a NON-chain override and `OnSettingsChangedDelegate.Broadcast(this, ChangeType)` at `:744` sits inside it, while `UPCGNode::PostEditChangeProperty` (`PCGNode.cpp:858-867`) branches only on `NodeTitle` — so no `PostEditChangeChainProperty` is required and the `Utils/PropertyChangeNotify.h` hazard (an unguarded `PropertyChain.GetActiveMemberNode()` deref in `UInstancedStaticMeshComponent`) is never touched. This is therefore NOT the `F-ism-per-instance-transforms` shape, where the handling lives only in the chain override and a typed refusal is the honest answer. **Resolver (`Utils/PropertyInspection.h/.cpp`):** new `FPropertyNotifyTarget {UObject* Object; FString RelativePath;}` plus an optional `OutNotifyTarget` out-param on `ResolveNestedPropertyPath` (defaulted, so its ~10 other call sites are untouched). The walk now tracks `LastCrossedObject` / `LastObjectHopSegment`, set at BOTH object-hop sites — the named-segment `FObjectProperty` branch and the array-element branch, for which file-local `DrillIntoArrayElement` gained an `OutHoppedObject` out-param — and a `FillNotifyTarget` lambda fires on the two success returns, joining the segments after the hop back into a path relative to the crossed object. **A path that crosses no `FObjectProperty` leaves both at RootObject / INDEX_NONE, so the target is byte-identically the old one**; this is the branch `#7` of `B-property-set-container-empty-change-event` depends on. New `ResolvePropertyNotifyTarget(RootObject, Path)` wraps it for callers that already resolved the leaf, short-circuiting a non-dotted name with no walk (same condition `ResolvePropertyOnObject` dispatches on). **Handler (`Handlers/Utility/UtilityPropertyHandler.cpp`):** `NotifyReflectedPropertyChanged` now resolves the notify target and sends BOTH halves there — `PinWright::NotifyPropertyChanged(NotifiedObject, …)` and `PushRenderStateForComponentTarget(NotifiedObject)` — so the pair cannot disagree (a path into a component sub-object now pushes render state on that component; before, the push landed on the actor and did nothing, `Cast<UActorComponent>` failing). `ResolveMemberPropertyForPath` takes `NotifiedObject` and the RELATIVE path, so `SettingsInterface.LowerBound` yields member `LowerBound` on the settings object (leaf == member, `SetActiveMemberProperty` correctly skipped) rather than `SettingsInterface`, which the settings class does not declare; the in-source comment at the old `:219-221` that this ticket contradicted is rewritten to say why. **Deliberately re-resolved inside the notify helper rather than threaded through 13 call sites**: one extra walk of a few `FindFProperty` lookups, through the SAME resolver (no second traversal implementation), and the path cannot have moved because only the leaf was written — the alternative was 13 near-identical one-line edits in a file ~17 agents are editing concurrently. **`container.*` DOES share the exposure, and the "bare `Modify()` with no notification at all" reading of it is stale**: all 11 mutators already call `NotifyReflectedPropertyChanged` (that landed with `B-property-set-container-empty-change-event`), so they carried the same wrong target for the same paths and are fixed by the same change, with no per-verb edit. **`Modify()` is deliberately NOT moved and stays on RootObject** — the ticket flags it as inference, not measurement, and retargeting it would dirty a second package that `property.set`'s `markDirty=false` restore (anchored to RootObject's package) does not cover; that is a new defect traded for an unmeasured one, and it belongs in its own ticket. Also unchanged: `property.reset`'s `bAlreadyNotified` branch, whose chain event comes from `FOverridableManager` against a RootObject-built `FPropertyVisitorPath` — a different mechanism, out of scope. **Regression test, behavioural and read-back-free:** `Tests/Utility/TestPropertyObjectHopNotify.cpp` + `TestPropertyObjectHopNotifyHost.h`. The fixture's inner class recomputes `DerivedFrom*` ONLY inside its own name-matched `PostEditChangeProperty` (the `UPCGSettings` shape), so every assertion is on state no reflection store can produce — the stored value is asserted too, but only so a red cannot be mistaken for a write that failed to land. `PinWright.property.set.ObjectHopNotifiesTheInnerObject` covers a leaf on the hopped-to object, an object-then-struct hop (member must read `Nested`, not `Inner`) and an ARRAY-ELEMENT object hop (`InnerArray[0].Source`, the other resolver hop site); `PinWright.container.array.ObjectHopNotifiesTheInnerObject` covers the shared container path. **Counterfactual: with the notification back on RootObject every `DerivedFrom*` stays at its `-1` sentinel, inner `NotifyCount` is 0 and outer is 1 — all three fail, while the stored values still land, which is the ticket's whole point.** `PinWright.property.set.StructHopStillNotifiesTheRootObject` passes before AND after by design: it is the guard that struct hops did not move. Doc: `Docs/wiki-src/property.md` gains a paragraph stating the notification follows the hop and that no response field reports which object was notified. `check_test_ids.py` clean (4710 ids, no dot-prefix collision). **NOT compiled and NOT run — the wave owner builds and runs the suite.** Untouched as instructed: `E-property-set-no-measured-render-state` (still a reopen candidate; this fix makes "which object was notified" cheaper to report than ever, since the helper now computes it), `B-property-set-container-empty-change-event`, and `F-pcg-set-node-property`, whose proposed implementation this fix satisfies generically — a hand-built path through the hop now notifies the settings object, so that verb no longer has to re-solve it.
