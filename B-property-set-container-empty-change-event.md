---
id: B-property-set-container-empty-change-event
title: "property.set / property.reset / 11 container.* verbs notify with a bare PostEditChange(), whose empty event skips every name-matched engine branch"
status: OPEN
severity: High
category: bug
tags: [property, container, property-write, derived-state, post-edit-change, camouflaged, silent-noop, misleading-success]
---

# A notification that is guaranteed to match nothing

`UObject::PostEditChange()` builds an **empty** event and forwards it:

```cpp
void UObject::PostEditChange(void)                       // Obj.cpp:549-553
{
    FPropertyChangedEvent EmptyPropertyUpdateStruct(NULL);
    this->PostEditChangeProperty(EmptyPropertyUpdateStruct);
}
```

`Property` and `MemberProperty` are both `nullptr`, so `GetPropertyName()` returns
`NAME_None`. Every engine override that branches on
`GET_MEMBER_NAME_CHECKED(Class, SomeProperty)` — which is how nearly all of them are
written — therefore matches **nothing**, for **any** assignment. Only overrides that do
unconditional work run at all.

Thirteen call sites in `Handlers/Utility/UtilityPropertyHandler.cpp` write an arbitrary
UPROPERTY on an arbitrary `UObject` through `ApplyJsonValueToProperty` and then notify
this way:

| line | verb | registered at |
|---|---|---|
| `:1129` | `property.set` | `:893` |
| `:1321` | `property.reset` | `:1147` |
| `:1991` | `container.array.append` | `:1951` |
| `:2034` | `container.array.remove` | `:2006` |
| `:2063` | `container.array.clear` | `:2049` |
| `:2130` | `container.array.insert` | `:2078` |
| `:2249` | `container.array.set` | `:2199` |
| `:2351` | `container.map.set` | `:2263` |
| `:2480` | `container.map.remove` | `:2443` |
| `:2609` | `container.map.clear` | `:2595` |
| `:2687` | `container.set.add` | `:2624` |
| `:2773` | `container.set.remove` | `:2701` |
| `:2884` | `container.set.clear` | `:2870` |

## Why this is worse than the sibling defect, not a duplicate of it

`B-set-component-properties-no-change-notification` is the **absent** notification:
`actor.set_component_properties` simply never called one. This is the **camouflaged** one.
A reviewer reading `RootObject->PostEditChange();` three lines under the write sees what
looks like correct, careful handling — the line is even preceded by a comment explaining
its *ordering* relative to the dirty-flag restore, which reads as evidence someone thought
about it. Nothing on the page says the event is empty. That is why it survived a fix pass
that landed on the neighbouring handler.

It also has a **different fix** and a **different blast radius**:

- different fix — the remedy is not "add a call", it is "build a real
  `FPropertyChangedEvent` naming the property", plus the right `EPropertyChangeType` per
  verb (`ArrayAdd` / `ArrayRemove` / `ArrayClear` / `ValueSet`), which the container verbs
  need and the component verb never did;
- different blast radius — `property.set` accepts **any** `UObject`, not just a component:
  asset CDOs, actors, settings objects, Niagara/Blueprint/Material assets. The affected set
  is every `PostEditChangeProperty` override in the engine, not the component subset.

Filing separately so it cannot be closed by association when the component ticket closes.

## What the fix has to get right

- **Name the member, not just the leaf.** `property.set` resolves dotted paths
  (`ResolvePropertyFromObject`, walk at `:640-688`), so for `Foo.Bar` the event needs
  `Property` = leaf and `MemberProperty` = top-level, via
  `FPropertyChangedEvent::SetActiveMemberProperty`. Naming only the leaf misses every
  `MemberProperty`-matched branch (e.g. `UWaterBodyComponent`'s `LayerWeightmapSettings`
  and `StaticMeshSettings` tests, `WaterBodyComponent.cpp:1248`, `:1277`). The file already
  contains the helper that computes exactly this pair —
  `BuildEditPropertyChainFromVisitorPath` (`:691-719`) sets both the active property node
  and the active member node from an `FPropertyVisitorPath`.
- **`property.reset` already builds the correctly-shaped event and throws it away.** The
  5.4 branch constructs `FPropertyChangedEvent PropertyEventForOverride(Property,
  EPropertyChangeType::ValueSet)` plus a real `FEditPropertyChain` (`:1267-1270`) purely to
  feed `FOverridableManager`, and then still notifies the object with the bare form fifty
  lines later (`:1321`). Reuse it. On 5.5+ prefer `EPropertyChangeType::ResetToDefault`,
  which is what that branch's own comment says exists from 5.5.
- **Non-chain form only.** `UInstancedStaticMeshComponent::PostEditChangeChainProperty`
  dereferences `PropertyChain.GetActiveMemberNode()` unguarded
  (`InstancedStaticMesh.cpp:5638`); a synthesized chain crashes the editor for any property
  outside its three known branches. Use
  `PinWright::NotifyPropertyChanged` (`Utils/PropertyChangeNotify.h`), which exists as of
  the component fix and encodes this decision.
- **Do not regress `markDirty:false`.** `B-property-set-markdirty-false-still-dirties` is
  why the notification is sequenced before `FinalizeApplied` — concrete overrides call
  `Modify()`/`MarkPackageDirty()` internally, so the dirty restore has to stay last. A real
  event makes MORE overrides run, so more of them will dirty the package. Whatever replaces
  the bare call must keep that ordering, and the `markDirty:false` tests must be re-run
  rather than assumed.

## Scope note — what this ticket does NOT cover

Bare `PostEditChange()` also appears on concrete asset targets elsewhere
(`MaterialAuthoringHandler.cpp` ~20 sites, `LandscapeHandler.cpp:233`, `:676`,
`VolumeHandler.cpp:126`, `GameplayTagBuildQueryHandler.cpp:365`, `MGIRCompiler.cpp:552`,
`PCGSetSelfPruningSettings.cpp:103`, `ChooserAuthoringHandler.cpp:439`,
`PoseSearchHandler.cpp:160`). Those target a **known** class whose override may well do
unconditional work, so the empty event is not automatically a defect there and each needs
its own judgement. **Not audited** — do not read their absence from this ticket as a
clean bill. `LandscapeHandler.cpp:233` is explicitly documented in place as the
degraded-but-better-than-nothing fallback, and `rpc-design.md` §5b already governs the
material ones.

## Related

- `B-set-component-properties-no-change-notification` (IN-REVIEW) — the absent-notification
  sibling. Its fix added `Utils/PropertyChangeNotify.h`, which this should reuse.
- `B-landscape-set-material-stale-mics` (IN-REVIEW) — the shipped instance of exactly this
  empty-event failure, on `ALandscapeProxy::PostEditChangeProperty`; the analysis at
  `LandscapeHandler.cpp:192-216` is the best existing write-up of the mechanism.
- `B-property-set-markdirty-false-still-dirties` — the constraint any fix must not regress.
- `rpc-design.md` §5d — "A bare `PostEditChange()` is not the notification."

## History
- `#1-empty-event-matches-nothing` `OPEN` reporter — Found while auditing the general case behind `B-set-component-properties-no-change-notification`. Verified in engine source that `UObject::PostEditChange()` builds `FPropertyChangedEvent EmptyPropertyUpdateStruct(NULL)` and forwards it (`Obj.cpp:549-553`), so `Property` and `MemberProperty` are both null and every `GET_MEMBER_NAME_CHECKED` branch in every override is skipped for any assignment. Enumerated the 13 generic call sites in `UtilityPropertyHandler.cpp` and mapped each to its registered verb (table above); confirmed `property.set` resolves dotted paths so the fix needs the leaf/member pair, and that `property.reset` already constructs a correctly-shaped `FPropertyChangedEvent` + `FEditPropertyChain` for `FOverridableManager` at `:1267-1270` and then discards it in favour of the bare call at `:1321`. NOT reproduced at runtime: the shared editor for the host project was live with roughly a dozen agents attached, so no verb was driven against it; this is a source-level finding corroborated by the frame measurement on the sibling ticket and by `B-landscape-set-material-stale-mics`, which is the same empty-event failure already observed end-to-end on landscapes. Severity High rather than Critical because the write itself does land and persist — what is lost is every derived-state effect, silently.
