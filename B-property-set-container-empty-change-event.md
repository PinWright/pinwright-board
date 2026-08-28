---
id: B-property-set-container-empty-change-event
title: "property.set / property.reset / 11 container.* verbs notify with a bare PostEditChange(), whose empty event skips every name-matched engine branch"
status: IN-REVIEW
severity: High
category: bug
tags: [property, container, property-write, derived-state, post-edit-change, camouflaged, silent-noop, misleading-success]
encounters: 3
lastSeen: 2026-08-28T11:20:00+05:00
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

## Encounter 2026-08-27 — the first end-to-end RUNTIME measurement on this ticket

`#1` recorded "NOT reproduced at runtime … this is a source-level finding", and both
adversarial reviews close with "Runtime: NOT VERIFIED". This section is that missing proof:
a pinned-exposure, fixed-pose pixel measurement showing a `property.set` write that reports
success at every layer and never reaches the renderer. Measured on host project
EAContentExamples58, level `/Game/Maps/Atlantis`, UE 5.8, target
`/Game/Maps/Atlantis.Atlantis:PersistentLevel.ExponentialHeightFog_0.HeightFogComponent0`.

Every capture is `render.capture_open_level` at an identical pose — `location {x:-9000,y:0,z:260}`,
`rotation {pitch:-4,yaw:0,roll:0}`, `fov 60`, 768x768, `exposure {mode:"fixed", ev100:0}`
(`adaptedSource: "fixedPin"` on all six), `hideEditorSprites: true`, game view on — so the
only variable between rows is the call in the middle column.

| # | call | frame `meanLuminance` | reached the renderer? |
|---|---|---|---|
| 1 | `lighting.setup_volumetric_fog {viewDistance: 65000}` | 0.338753 | — (baseline) |
| 2 | `property.set FogDensity = 2.0` | 0.220004 | **yes** |
| 3 | `property.set bEnableVolumetricFog = false` | 0.220080 | **no** |
| 4 | `property.set FogDensity = 0.2` | 0.220077 | **no** |
| 5 | `lighting.setup_volumetric_fog {viewDistance: 65000}` | 0.339024 | flushes 3 **and** 4 at once |
| 6 | *(no call at all — capture repeated)* | 0.339045 | stable, so 5 is not a settling artefact |

Row 4 is the case: `FogDensity` is 0.2 in the object, `property.get` returns 0.2, the write
answered `applied: true, markedDirty: true, pendingSave: true`, and the renderer is still
drawing 2.0 — a **10x** difference, held indefinitely, not for one frame.

**Row 6 is the control** that rules out "the refresh just takes an extra frame": with no
call between the two captures the frame does not move (2e-5).

**Noise floor.** Each capture's own reported `viewport.warmup.meanLuminanceDelta` is
4e-6..1.3e-5. The 0.119 gap between the stale and flushed frames is four orders of
magnitude above that floor; the 7e-5 "change" at rows 3-4 is inside it.

Captures under `Saved/Screenshots/OpenLevel/`: `env_probe_flush2.png` (5),
`env_probe_density2_nopush.png` (2), `env_probe_volfog_off.png` (3),
`env_probe_d02_volfogfalse.png` (4), `env_probe_flush2_nowrite_recap.png` (6).

### Correcting the reporter's own hypothesis against this tree

The defect log this came from guessed that `property.set` "does not call
`MarkRenderStateDirty()` / `PostEditChangeProperty()`". Half right, and the wrong half
matters:

- **`property.set` DOES notify.** The generic reflected path (`property.set` registered at
  `Handlers/Utility/UtilityPropertyHandler.cpp:893`, generic path `:1108-1141`) calls
  `RootObject->PostEditChange()` at **`:1129`** — the bare no-arg form this ticket is about.
  So the engine override *runs*; it just runs on an empty `FPropertyChangedEvent`. The log's
  guess is wrong in letter, and the mechanism is this ticket's stated one.
- **`MarkRenderStateDirty()` is genuinely absent.** `grep MarkRenderStateDirty` over
  `UtilityPropertyHandler.cpp` returns **zero** hits, as does `PostEditChangeProperty`
  (the only near-hit in the file is `PostEditChangeChainProperty` at `:773`, inside
  `property.reset`'s override-clear path). That half of the guess holds.

### Why the flush workaround works — the plugin already owns the helper it did not use

`lighting.setup_volumetric_fog` is a *different verb* that does the render-state push the
write path omits:

```cpp
// Handlers/Environment/LightingHandler.cpp:696-698
// Raw field writes: neither bEnableVolumetricFog nor VolumetricFogDistance
// has an engine setter that pushes render state.
PinWright::MarkComponentRenderStateDirty(FogComp);
```

and that helper is one line of plugin code:

```cpp
// Handlers/Environment/EnvironmentDirtyUtils.h:81-87
inline void MarkComponentRenderStateDirty(UActorComponent* Component)
{
    if (Component) { Component->MarkRenderStateDirty(); }
}
```

Its own doc comment at `EnvironmentDirtyUtils.h:74-76` already names this exact case:
"Raw writes (SkyComp->SourceType, SkyComp->Cubemap, **FogComp->bEnableVolumetricFog,
FogComp->VolumetricFogDistance**) push nothing and need it." That is precisely why row 5
flushes rows 3 and 4 together, and it turns the workaround from folklore into a mechanism:
the environment handlers learned this lesson and `property.set` did not.

### Engine trace — for THIS component the empty event is not the whole story, and the fix as scoped would not cover it

`UExponentialHeightFogComponent::PostEditChangeProperty`
(`C:\UE_5.8\Engine\Source\Runtime\Engine\Private\Components\ExponentialHeightFogComponent.cpp:241-254`)
does only unconditional clamping and then `Super::PostEditChangeProperty(...)`. It has
**no** `GET_MEMBER_NAME_CHECKED` branch and **no** `MarkRenderStateDirty()` of its own —
every `MarkRenderStateDirty()` in that file lives in the generated `Set*` setters
(`UEXPONENTIALHEIGHTFOG_VARIABLE` macro at `:272-281`, plus hand-written ones such as
`SetSecondFogDensity` at `:287-295`). So naming the property in the event, which is this
ticket's proposed fix, would change nothing for this class.

The refresh in the editor's own path comes from the **`PreEditChange`/`PostEditChange`
reregister pair**, not from the event's name:

- `UActorComponent::PreEditChange` (`ActorComponent.cpp:1311-1340`) does
  `EditReregisterContexts.Add(this, new FComponentReregisterContext(this))`;
- `UActorComponent::PostEditChangeProperty` (`:1493-1498`) forwards to
  `ConsolidatedPostEditChange` (`:1432`), whose first act is
  `EditReregisterContexts.RemoveAndCopyValue(this, ReregisterContext); delete ReregisterContext;`
  — destroying the context is what re-registers the component and rebuilds its render state.

`property.set` never calls `PreEditChange` (**zero** hits in `UtilityPropertyHandler.cpp`),
so there is no context to remove and `ConsolidatedPostEditChange` takes the `else` branch —
no reregister, no render-state rebuild. That matches the measurement exactly, and it is a
source trace, not a stepped debugger.

**Consequence for the fix in this ticket's "What the fix has to get right" section:** a
correctly-named `FPropertyChangedEvent` is necessary but **not sufficient** for
renderer-backed component targets. Either pair the notification with the matching
`PreEditChange(Property)` call, or — narrower and safer — call
`PinWright::MarkComponentRenderStateDirty` when the resolved `RootObject` is a
`UActorComponent`, reusing `EnvironmentDirtyUtils.h:81`. Whichever is chosen, the acceptance
test is a pixel measurement like the table above, not a read-back.

### Unexplained, recorded as an observation and NOT a rule

Row 2 landed and rows 3-4 did not — "roughly the first write after a refresh lands, every
write after it is swallowed until something refreshes the state again". Nothing tested here
explains that, and the reregister trace above does not predict it. **Do not rely on it**;
it is recorded so a fixer who sees a first-write-works pattern does not conclude the bug is
intermittent.

### Response-shape ask

Beyond the write fix: the call currently reports only what was *requested*. A **measured**
`renderStateRefreshed` field, in the house style of the capture verbs' `viewport.*` blocks,
would let a caller tell a write that reached the renderer from one that did not — which no
layer of the current response distinguishes.

### Relation to the sibling ticket, and one caution on the workaround

`B-set-component-properties-no-change-notification` (IN-REVIEW, High) lists
`UExponentialHeightFogComponent` at its `:84` among components it flagged as at risk but
never measured. This is the first **measured** instance of a component that ticket
predicted — measured here on the *other* verb, `property.set`, which that ticket does not
cover.

Caution on routing around this via `actor.set_component_properties`: that verb is the
notifying one only because of the fix on that sibling ticket, which is **IN-REVIEW, not
DONE**, so the workaround depends on that fix being in the build under test. On this host it
was: that ticket's `#4` records the same session confirming `notified[]` beside `applied[]`
in every response, and an unflushed `FogDensity` 0.06 -> 0.6 through that verb moving the
same fixed pose 0.333654 -> 0.275127 against a 0.0017 no-change control. On a build without
it, both verbs fail and there is no workaround left.

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
- `#2-additional-current-path-review` `OPEN` reporter — Additional evidence: **Adversarial review A — the generic defect remains, but the ticket overstates reset and `property.set` coverage.** Actuality: **PARTIAL**. Framing: the generic reflected `property.set` path and all 11 listed container mutators still call bare `RootObject->PostEditChange()` (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:1129,1321,1991,2034,2063,2130,2249,2351,2480,2609,2687,2773,2884`); UE 5.8 constructs that as a null-property event (`C:\UE_5.8\Engine\Source\Runtime\CoreUObject\Private\UObject\Obj.cpp:549-558`), while named branches require a property-bearing event (`C:\UE_5.8\Engine\Source\Runtime\CoreUObject\Public\UObject\UnrealType.h:6977-6990`). This supports High (silent stale derived state), not Critical: writes persist. `property.reset` can already dispatch a named chain event only when `FOverridableManager` is available (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:744-793`), and four actor-special-case setters explicitly skip notification (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:979-982`). Proposed fix: **INCOMPLETE**; a centralized named event/change type is systemic and matches UE’s property editor (`C:\UE_5.8\Engine\Source\Editor\PropertyEditor\Private\PropertyNode.cpp:3337-3420`), but the ticket’s advertised `Utils/PropertyChangeNotify.h` helper is absent from the current PinWright checkout/history. The replacement must avoid reset double-notification, preserve mark-dirty restoration ordering (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:1317-1329`), and handle nested/container indices (the current visitor-chain builder is object/struct-only at `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:617-720`, while the resolver supports array paths at `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\PropertyInspection.cpp:204-349`). The ticket’s non-chain-only crash rationale is not established: current chain construction sets an active member node (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:691-720`), though this still needs focused validation. No current PinWright test asserts these callbacks/change types (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Utility\TestPropertyMarkDirtyRespected.cpp:322-389`, `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Utility\TestUtilityHandlers.cpp:203-421`). Evidence: named UE branches in `C:\UE_5.8\Engine\Plugins\Experimental\Water\Source\Runtime\Private\WaterBodyComponent.cpp:1244-1284` and `C:\UE_5.8\Engine\Source\Runtime\Engine\Private\Components\PrimitiveComponent.cpp:1541-1643`. Runtime: NOT VERIFIED. Recommendation: **REFRAME**; scope to generic reflected writes, state reset’s conditional path and actor exceptions, then implement/test a version-aware notifier and run focused UE 5.8 proof.
- `#3-additional-event-shape-chain-correction` `OPEN` reporter — Additional evidence: **Adversarial review B — A’s PARTIAL/REFRAME verdict survives, but its blanket non-chain preference is too broad and the operation event shape is underspecified.** Actuality: **PARTIAL**. Framing: the generic `property.set` and 11 container mutators still end in bare calls (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:1129,1991,2034,2063,2130,2249,2351,2480,2609,2687,2773,2884`), and `property.reset` still does so after the manager-conditional chain path (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:744-793,1317-1329`); actor setter fast paths still deliberately emit no hook (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:979-982`). High remains appropriate for silent stale derived state, but the title should say “generic reflected path” and distinguish reset’s conditional duplicate event and actor exceptions. Proposed fix: **INCOMPLETE**; UE’s own editor treats array/set/map add, remove, and clear uniformly as `ArrayAdd`/`ArrayRemove`/`ArrayClear` (`C:\UE_5.8\Engine\Source\Editor\PropertyEditor\Private\PropertyHandleImpl.cpp:1274-1278,1438-1443,1616-1626,1911-1915`), while element value writes use `ValueSet` plus `SetArrayIndexPerObject` (`C:\UE_5.8\Engine\Source\Editor\PropertyEditor\Private\PropertyHandleImpl.cpp:367-375,667-674`). A notifier must therefore pass the edited leaf (array inner/map key-or-value), top-level member, and logical index, not merely the container property. A valid `FEditPropertyChain` with an active member is not disproved by ISM’s null dereference (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:691-720`; `C:\UE_5.8\Engine\Source\Runtime\Engine\Private\InstancedStaticMesh.cpp:5638`), and blanket non-chain dispatch would omit legitimate chain-only transform handling (`C:\UE_5.8\Engine\Source\Runtime\Engine\Private\Components\PrimitiveComponent.cpp:1698-1716`). Choose one safe event form per operation; never double-notify reset; preserve dirty-restore ordering. The current chain builder remains object/struct-only while the resolver accepts indexed arrays (`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Utility\UtilityPropertyHandler.cpp:633-684`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\PropertyInspection.cpp:242-361`), so indexed paths need explicit support or rejection and tests that capture Property/MemberProperty/ChangeType/index. Runtime: NOT VERIFIED. Recommendation: **REFRAME**; keep OPEN, implement one central version-aware notifier with exact per-verb metadata, then add focused UE 5.8 probes for generic, reset, arrays, maps, sets, nested/indexed paths, and chain-only overrides.

- `#4-runtime-proof-fog-component` `OPEN` reporter — Additional evidence: **first end-to-end RUNTIME measurement on this ticket**, which `#1` and both adversarial reviews record as missing ("Runtime: NOT VERIFIED"). Found on host project EAContentExamples58, level `/Game/Maps/Atlantis`, UE 5.8, target `…PersistentLevel.ExponentialHeightFog_0.HeightFogComponent0`. Six captures at one fixed pose with pinned exposure (`ev100 0`, `adaptedSource "fixedPin"`): `property.set FogDensity=2.0` moved `meanLuminance` 0.338753 -> 0.220004 (landed), then `property.set bEnableVolumetricFog=false` (0.220080) and `property.set FogDensity=0.2` (0.220077) moved nothing while `property.get` returned the new values and the responses said `applied/markedDirty/pendingSave` true — the renderer kept drawing 2.0, a 10x error held indefinitely. A `lighting.setup_volumetric_fog` call then flushed BOTH stale writes at once (0.339024), and a no-call control recapture held at 0.339045, ruling out a one-frame settling delay. Per-capture `viewport.warmup.meanLuminanceDelta` was 4e-6..1.3e-5, so the 0.119 stale/flushed gap is four orders of magnitude above the floor and the 7e-5 non-changes are inside it. Full table, capture filenames and the source trace are in the body section "Encounter 2026-08-27". **Corrected the reporting log's hypothesis against this tree**: it guessed `property.set` calls neither `MarkRenderStateDirty()` nor `PostEditChangeProperty()`; in fact the generic path DOES call `RootObject->PostEditChange()` at `UtilityPropertyHandler.cpp:1129` (registered `:893`, generic path `:1108-1141`) — the bare form this ticket is about, so the log is wrong in letter and the cause is this ticket's own — while `MarkRenderStateDirty` and `PostEditChangeProperty` really are absent from that file (zero grep hits; only `PostEditChangeChainProperty` at `:773`). **Explained the flush workaround**: the plugin already owns the helper it did not use here — `PinWright::MarkComponentRenderStateDirty` at `Handlers/Environment/EnvironmentDirtyUtils.h:81-87`, whose own comment at `:74-76` names `FogComp->bEnableVolumetricFog` / `VolumetricFogDistance` as raw writes that "push nothing and need it" — and `lighting.setup_volumetric_fog` calls it at `Handlers/Environment/LightingHandler.cpp:698`. **New constraint on this ticket's proposed fix**: for this component a correctly-named event is necessary but NOT sufficient. `UExponentialHeightFogComponent::PostEditChangeProperty` (`C:\UE_5.8\…\ExponentialHeightFogComponent.cpp:241-254`) only clamps and calls Super — no `GET_MEMBER_NAME_CHECKED` branch, no `MarkRenderStateDirty` (those live in the `Set*` setters at `:272-281`, `:287-295`). The editor's refresh comes from the `PreEditChange`/`PostEditChange` reregister pair: `UActorComponent::PreEditChange` adds `EditReregisterContexts.Add(this, new FComponentReregisterContext(this))` (`ActorComponent.cpp:1311-1340`) and `ConsolidatedPostEditChange` (`:1432`, reached via `:1493`) destroys it, which is what re-registers the component. `property.set` never calls `PreEditChange` (zero hits), so that branch never runs. Fix must therefore either pair the notification with `PreEditChange`, or call `MarkComponentRenderStateDirty` for `UActorComponent` targets. Recorded as an observation and explicitly NOT a rule: the first write after a refresh appears to land and later ones are swallowed — unexplained by the trace above, so a fixer must not read it as intermittency. Also asked for a measured `renderStateRefreshed` response field in the house style of the capture verbs' `viewport.*` blocks. Status and severity deliberately unchanged; `B-set-component-properties-no-change-notification:84` lists `UExponentialHeightFogComponent` among components it inferred but never measured, so this is the first measured instance of a component that ticket predicted, on the verb it does not cover. Caution: routing around this via `actor.set_component_properties` depends on that sibling's fix, which is IN-REVIEW not DONE — it was live on this host (that ticket's `#4`), but on a build without it both verbs fail.
- `#5-named-event-and-render-state-push` `IN-REVIEW` developer — Replaced all 13 bare `RootObject->PostEditChange()` calls in `Handlers/Utility/UtilityPropertyHandler.cpp` with a file-local `NotifyReflectedPropertyChanged(RootObject, PropertyName, Property, ChangeType)` that dispatches a properly populated non-chain `FPropertyChangedEvent` through `PinWright::NotifyPropertyChanged`. Extended that helper (`Utils/PropertyChangeNotify.h`) with two defaulted params — `MemberProperty` (applied via `SetActiveMemberProperty` only when it differs from the leaf, so a dotted path names leaf + top-level member and member-matched branches fire) and `ChangeType`; the existing 2-arg `actor.set_component_properties` call site is unchanged. Member resolution is a new `ResolveMemberPropertyForPath` that takes the first path segment (stripping `.` and `[N]`) off `RootObject`'s class — deliberately RootObject-relative, since the notification target is unchanged. Per-verb change types: `property.set` and `container.array.set` ValueSet; `array.append`/`array.insert`/`map.set`/`set.add` ArrayAdd; `array.remove`/`map.remove`/`set.remove` ArrayRemove; `array.clear`/`map.clear`/`set.clear` ArrayClear; `property.reset` ResetToDefault on 5.6+ (ValueSet below, per the file's existing gate — `ResetToDefault` is 1 << 10 and does not exist on 5.5). Addressed the encounter-4 finding that a named event is necessary but not sufficient for renderer-backed components: a new `PushRenderStateForComponentTarget` calls `PinWright::MarkComponentRenderStateDirty` (`Handlers/Environment/EnvironmentDirtyUtils.h:81`) when the resolved target is a `UActorComponent`, guarded by `IsValid` because it runs after the notification. `PreEditChange` deliberately still not called (flush + construction-script rerun; `Utils/PropertyChangeNotify.h`). Removed the reset double-notify flagged by adversarial review B: `ClearExplicitOverrideState` now reports through a `bool& bOutNotified` whether it dispatched its chain event, and the value path skips its own notification when it did — one write, one engine change event. Notification ordering relative to `FinalizeApplied` / the `markDirty=false` restore is unchanged, so `B-property-set-markdirty-false-still-dirties` is not regressed, but a named event runs strictly more overrides so the `markDirty:false` tests must be re-run rather than assumed. Tests added in `Source/PinWright/Private/Tests/Utility/TestPropertyChangeEventShape.cpp` with fixture `TestPropertyChangeEventHost.h` (a UObject that records `Property`/`MemberProperty`/`ChangeType`/notify count from every event it receives — the only way to observe this defect, since the bare call also reaches `PostEditChangeProperty`): `PinWright.property.set.ChangeEventNamesLeafAndMember`, `PinWright.property.reset.ChangeEventNamesProperty`, `PinWright.container.array.ChangeEventNamesPropertyAndOperation`. NOT done: the requested measured `renderStateRefreshed` response field — the response shapes of all 13 verbs are unchanged, and a truthful "measured" field needs a render-state probe this fix does not add. NOT compiled or run (fix-agent rules forbid building); the acceptance test named in the body is a pixel measurement, still owed.

- `#6-named-event-half-measured-fixed-render-half-undecided` `IN-REVIEW` verifier — 2026-08-28, editor
  running the plugin built at `b79ba53e`. **Status deliberately left at IN-REVIEW.** The defect in the
  title is measured fixed; the render-state half this ticket acquired at `#4` is not decidable from what
  I could run, and `#5` itself records its acceptance test as "a pixel measurement, still owed".

  **Measured, and this half is closed.** Probe: a scratch `StaticMeshActor` (`/Engine/BasicShapes/Cube`)
  spawned at `{400000,400000,200000}`, mesh component set `Movable` first so the cull-distance-volume
  branch cannot confound the reading, then deleted. The instrument is `CachedMaxDrawDistance`, which
  `UPrimitiveComponent::PostEditChangeProperty` copies from `LDMaxDrawDistance` *inside*
  `if (PropertyThatChanged)` (`PrimitiveComponent.cpp:1546-1553`, the copy committed by
  `SetCachedMaxDrawDistance` at `:1628`) — so it is written only by a **property-bearing** event and is
  exactly the branch a null-property event skips. Baseline `CachedMaxDrawDistance` 0.
  - `property.set {propertyName:"LDMaxDrawDistance", value:5678}` on
    `…PersistentLevel.StaticMeshActor_36.StaticMeshComponent0` -> `CachedMaxDrawDistance` **5678**.
    The leaf-named `ValueSet` event reaches the name-matched branch.
  - `property.reset {propertyName:"LDMaxDrawDistance"}` -> `oldValue:5678, defaultValue:0,
    wasOverridden:true` and `CachedMaxDrawDistance` back to **0**. `#5`'s `ResetToDefault` path fires a
    named event too, and the double-notify removal did not cost the notification.
  - **The `MemberProperty` half, which is the harder claim and untested by the leaf case:**
    `container.array.append {propertyName:"CustomPrimitiveData.Data", value:7.5}` -> the transient
    `CustomPrimitiveDataInternal` went `{"Data":[]}` -> `{"Data":[7.5]}`. That copy is
    `ResetCustomPrimitiveData()` (`PrimitiveComponent.cpp:2646-2648`), reached **only** from the
    `if (FProperty* MemberPropertyThatChanged = PropertyChangedEvent.MemberProperty)` block at `:1583-1592`
    matching `GET_MEMBER_NAME_CHECKED(UPrimitiveComponent, CustomPrimitiveData)`. So a dotted path really
    does now carry leaf **and** top-level member, and `#5`'s `ResolveMemberPropertyForPath` works on a
    container mutator, not just on `property.set`. Three of the thirteen call sites, two event shapes
    (leaf-matched and member-matched), three change types (`ValueSet`, `ResetToDefault`, `ArrayAdd`).

  Values 0 -> 5678 -> 0 and `[]` -> `[7.5]` are three distinct states tracked across two properties, so
  neither reading is a coincidental match against a pre-existing value.

  **Not measured, and why this stays open.** `#4` is this ticket's only prior runtime proof and it is not
  a name-matched-branch failure at all: `#4` established that `UExponentialHeightFogComponent::PostEditChangeProperty`
  has no `GET_MEMBER_NAME_CHECKED` branch and no `MarkRenderStateDirty`, so for that component a correctly
  named event is *necessary but not sufficient* — the editor's refresh comes from the
  `PreEditChange`/`ConsolidatedPostEditChange` reregister pair, and `#5` substituted a
  `PushRenderStateForComponentTarget` -> `PinWright::MarkComponentRenderStateDirty` for it. **I did not
  re-run the fog repro.** It requires writing `FogDensity` on the shipping level's `ExponentialHeightFog_0`
  and taking pinned-exposure captures, and at the time of writing several other agents were running their
  own pixel measurements against this same open level; perturbing the level's fog for the length of a
  five-capture sequence would have corrupted their readings, which is a worse outcome than an undecided
  half-ticket. `MarkRenderStateDirty` sets a bitfield cleared at end-of-frame and is not reflected, so
  there is no non-visual substitute for the capture.

  **What would settle it**, for whoever picks this up on a quiet editor: `#4`'s own table, rows 2-4 — the
  diagnostic signature was that the *first* `property.set` after a flush landed and every later one was
  swallowed until `lighting.setup_volumetric_fog` released them in a batch. Two consecutive
  `property.set FogDensity` writes each moving `meanLuminance` at one fixed pose, with **no**
  `setup_volumetric_fog` between them and a no-call control recapture, closes it. Restore the original
  density afterwards.

  Also unverified here and worth not losing: the ticket's own warning that a real event makes more
  overrides run and therefore dirty more packages — `#5` says the `markDirty:false` tests "must be re-run
  rather than assumed", and I did not re-run them (`B-property-set-markdirty-false-still-dirties`).
  Note in passing that every `property.set`/`container.*` response in this session did report
  `markedDirty:true`/`pendingSave:true`, which is the default path and says nothing about the
  `markDirty:false` one.

  Cross-check: the sibling `B-set-component-properties-no-change-notification` was closed DONE by another
  verifier on the same day; my own independent run of `actor.set_component_properties
  {LDMaxDrawDistance:1234}` on the same probe returned `applied:["LDMaxDrawDistance"],
  notified:["LDMaxDrawDistance"]` and moved `CachedMaxDrawDistance` 0 -> 1234, and
  `{Mobility:"Movable"}` returned `applied:["Mobility"]` with no `notified` entry — the engine-setter
  exclusion that ticket's `#2` claims. Recorded here only because this ticket warns it must not be closed
  by association with that one; it is not, and this entry does not close it.

  Footprint: the probe actor was deleted. `/Game/Maps/Atlantis` is left dirty and was NOT saved.
