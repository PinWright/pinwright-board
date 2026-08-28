---
id: B-shape-extent-stale-physics
title: "actor.set_component_properties rebuilds a shape component's render proxy from the new extent but not its live physics body, so a capture and a collision query disagree about the same actor"
status: IN-REVIEW
severity: High
category: bug
tags: [actor, components, property-write, shape-component, physics-body, collision, capture-verification, silent-half-success, derived-state]
encounters: 1
lastSeen: 2026-08-28T13:20:00+05:00
---

# The picture and the trace describe two different boxes

Writing `BoxExtent` (or `SphereRadius` / `CapsuleHalfHeight` / `CapsuleRadius`) on a
`UShapeComponent` through `actor.set_component_properties` returns
`{"applied":["BoxExtent"],"notified":["BoxExtent"]}` — full success on both lists — and then:

- the **renderer** shows the new extent (`UBoxComponent::CreateSceneProxy`,
  `BoxComponent.cpp:129`, reads `BoxExtent`, and the handler's `MarkRenderStateDirty()`
  at `ComponentHandler.cpp:356` rebuilds the proxy);
- **`actor.get_bounding_box`** reports the new extent (`CalcBounds` reads `BoxExtent`);
- a **scene query** — `spatial.raycast`, any overlap, anything that goes through
  `FBodyInstance` — still sees the **old** geometry, because the Chaos shapes were built
  from `AggGeom` at `CreatePhysicsState()` time and nothing re-reads them.

Two instruments, both confident, disagreeing about one actor. **The screenshot is the one
that lies**, which is the part that matters: it confirms the write for the half nobody
queries and stays silent about the half everything else queries.

## Live repro (measured, not inferred)

Editor on `/Game/Maps/Atlantis`, plugin at HEAD `b79ba53e`. Throwaway `TriggerBox`
`PWSHAPE_PHYSSTALE_01` (`TriggerBox_0`) spawned at `(20000, 0, 2000)` — well clear of the
city, verified empty by a `multiHit` raycast that found one layer — and deleted afterwards.
The level was **not** saved. Its root `CollisionComp` is a `UBoxComponent`; its profile was
moved off `Trigger` to `BlockAll` first (via `SetCollisionProfileName`, which updates filter
data only and does not rebuild shape geometry) purely so a `visibility` line trace can block
at all — `UShapeComponent` ships as `OverlapAllDynamic` (`ShapeComponent.cpp:47-48`) and
`ATriggerBase` as `Trigger`, and neither blocks a line trace.

All traces are `spatial.raycast` from `x=20800` toward `x=19000`, `z=2000`,
`onlyActors:["TriggerBox_0"]`. All captures are `render.capture_open_level` at the identical
pose `location {x:18800,y:0,z:2000}`, `rotation {0,0,0}`, `fov 60`, `768x768`,
`exposure {mode:"fixed", ev100:0}`, `hideEditorSprites:true`.

| # | action | `get_bounding_box` extent | capture | ray `y=0` | ray `y=200` | ray `y=300` |
|---|---|---|---|---|---|---|
| 0 | baseline (spawned, extent 40) | 40 | box **46 px** wide | hit `x=20040`, d 760 | **miss** | — |
| 1 | `set_component_properties BoxExtent {400,400,400}` → `applied`+`notified` | **400** | box **~668 px** wide | hit `x=20040`, **d 760 — unchanged** | **miss — unchanged** | — |
| 2 | `actor.set_collision false` then `true` (recreates the physics state) | 400 | — | hit `x=20400`, **d 400** | **hit** `x=20400` | — |
| 3 | `set_component_properties BoxExtent {150,150,150}` → `applied`+`notified` | **150** | box **~191 px** wide | — | — | **hit** `x=20400` |
| 4 | Python `UBoxComponent::SetBoxExtent(150,150,150)` — the value already stored | 150 | — | hit `x=20150`, d 650 | — | **miss** |

Read the rows against each other:

- **Row 1 is the defect.** Both instruments were asked the same question after a call that
  reported complete success. The renderer went from an 80 cm cube to an 800 cm cube — the
  wireframe grows from 46 px to filling the 768 px frame, and the measured widths match the
  projection arithmetic for extent 40 at 1160 cm and extent 400 at 800 cm to within a pixel.
  The two traces came back **byte-identical to the baseline**.
- **Row 2 is the control that rules out a bad ray.** Nothing about the geometry, the pose or
  the query changed — only the physics state was destroyed and rebuilt, from the same
  `AggGeom` that `GetBodySetup()` self-repairs. The ray immediately reports the 400 face and
  the `y=200` ray flips from miss to hit. So rows 0 and 1 were not measuring a wrong ray;
  they were measuring a stale body.
- **Row 3 is the worse direction.** Shrinking leaves the body **larger** than the drawing: a
  ray 300 cm off axis, 150 cm outside the box anyone can see, still blocks. That is an
  invisible collider sitting in empty space — a phantom the capture cannot show and the
  bounding box denies.
- **Row 4 is the workaround, verified.** The typed engine setter resynchronises the body even
  when called with the value the reflection write already stored, because its work is the
  `BodyInstance.UpdateBodyScale(..., true)` at `BoxComponent.cpp:40`, not the assignment.

Captures kept: `Saved/Screenshots/OpenLevel/pwshape_before_extent40.png`,
`pwshape_after_extent400.png`, `pwshape_after_extent150.png`.

## Mechanism

`AggGeom` is **not** the stale thing. `UShapeComponent::GetBodySetup()` calls
`UpdateBodySetup()` on the way past (`ShapeComponent.cpp:100-103`), so any reader repairs it
— which is exactly why row 2 rebuilds correctly. What is stale is the **live body**.

The engine's own setter closes both halves (`BoxComponent.cpp:28-47`):

```cpp
void UBoxComponent::SetBoxExtent(FVector NewBoxExtent, bool bUpdateOverlaps)
{
    BoxExtent = NewBoxExtent;
    UpdateBounds();
    MarkRenderStateDirty();
    UpdateBodySetup();                                                   // :33
    if (bPhysicsStateCreated)
    {
        BodyInstance.UpdateBodyScale(GetComponentTransform().GetScale3D(), true);  // :40
        if (bUpdateOverlaps && IsCollisionEnabled() && GetOwner()) { UpdateOverlaps(); }
    }
}
```

The Details panel reaches the same end state by a different route: its `PreEditChange`
creates an `FComponentReregisterContext`, so `ConsolidatedPostEditChange` tears the component
down and back up and the physics state is rebuilt from the fresh `AggGeom`.

`actor.set_component_properties` takes neither route. It stores by reflection
(`ComponentHandler.cpp:328`), notifies (`:338` → `UShapeComponent::PostEditChangeProperty`,
`ShapeComponent.cpp:157-164`, whose entire body is `UpdateBodySetup()`), then commits with
`MarkRenderStateDirty()` + `UpdateComponentToWorld()` (`:356-357`). `RecreatePhysicsState`
appears nowhere in the file (grep-verified at `b79ba53e`). So the notification buys the body
setup — which self-repaired anyway — and nothing touches the shapes already in the scene.

**This is deliberate and correctly so, which is why it needs its own decision rather than a
patch.** `B-set-component-properties-no-change-notification` `#2` rejected `PreEditChange`
on purpose: it unregisters the component and calls `FlushRenderingCommands`
(`ActorComponent.cpp:1327`, `:1336-1339`) and it is the only thing that populates
`EditReregisterContexts`, the sole trigger for rerunning the owner's construction scripts
(`:1437-1446`). Avoiding that cost is what leaves this gap.

## Scope

Not a box bug and not a shape bug. The general shape is: **a property whose engine setter
follows the assignment with a body update.** Reachable from this verb today:

- `UBoxComponent::BoxExtent`, `USphereComponent::SphereRadius`,
  `UCapsuleComponent::CapsuleHalfHeight` / `CapsuleRadius` — each has a typed setter with the
  same `bPhysicsStateCreated` → `UpdateBodyScale` tail.
- Anything else where the live body is derived state: a `BodySetup` swap, collision-geometry
  fields on primitives whose setters call `RecreatePhysicsState()`.

Not enumerated exhaustively here on purpose — the design question below decides which
properties qualify, and guessing the list before that decision would bake in the wrong
answer.

## Also wrong in the docs

`Plugins/PinWright/docs/wiki-src/actor.md:248` (mirrored to
`Saved/PinWright/wiki/actor.set_component_properties.md`) lists the change hook as driving
"`ShapeComponent` collision setup". That is half true in the way that is worst for a reader:
it drives the body **setup** and not the body. A caller who reads that line and then confirms
the write with a screenshot has done everything the documentation asks and still has a wrong
world.

## Workarounds (both measured above)

- Call the typed setter — `UBoxComponent::SetBoxExtent` and siblings, via `python.execute` —
  instead of, or immediately after, the reflection write (row 4).
- Or force a physics-state rebuild with `actor.set_collision {collisionEnabled:false}` then
  `{true}` (row 2). Note this leaves the component's collision profile invalidated to
  `Custom`, so it is the cruder of the two.

Verifying a shape-extent write **only** by capture, or only by `actor.get_bounding_box`,
cannot detect the failure. A `spatial.raycast` at the new face is the cheapest honest check.

## Fix — the open design question, stated rather than guessed

Should the handler call `RecreatePhysicsState()` (or the narrower
`BodyInstance.UpdateBodyScale(..., true)`) after a notified write when the changed property is
one the engine's own setter follows with a body update? Both halves are real:

- **Cost.** `RecreatePhysicsState()` on every notified write is far heavier than the
  notification and would show up on batch edits. `UpdateBodyScale` is much cheaper but is
  shape-specific, so it needs a per-class hook rather than a generic one.
- **Which properties qualify.** There is no reflection-visible marker for "this property's
  setter also updates the body". A name list per component class is maintainable but is the
  same enumerate-the-engine shape that `B-set-component-properties-no-change-notification`
  ended up with, and it will drift.
- **A third option:** route shape extents to their typed setters the way `StaticMesh` and
  `SkinnedAsset` already are (`ComponentHandler.cpp`, per
  `B-set-component-properties-staticmesh-shadow-corruption`) — narrow, no new policy, and it
  matches the existing precedent for "the setter is a superset of the notification". These
  properties would then correctly appear in `applied` and not in `notified`.

Whichever is chosen, the response should stop being able to report unqualified success while
one of the two representations is stale.

## Severity

**High**, and the argument matters more than the label.

*Impact class.* The board's rubric puts "silent false-success, or silent wrong / stale data on
a normal path (the caller trusts a result that is a lie and builds on it)" at High. This is
that, with an aggravation the rubric does not anticipate: the call does not fail silently, it
succeeds **halfway**, and the half it completes is the half a human or an agent looks at.
`applied` and `notified` are both populated, `get_bounding_box` agrees, and the screenshot
agrees. Every instrument a careful caller would reach for confirms the write; the one that
would catch it is the one nobody runs *because the others already said yes*. A defect that
corrupts the verification instrument outranks one the instrument catches, and this project's
whole verification culture is capture-first.

*Reach.* `actor.set_component_properties` is a core every-session verb, and shape components
are ordinary level-building objects — triggers, blockers, collision probes. The affected
sub-path is narrower than the verb, so no upward bump; it is nowhere near a rare edge path, so
no downward bump either. Reach leaves it where impact put it.

*Why not Critical.* Critical is reserved for an editor crash or a write that corrupts or loses
asset data. Nothing here is corrupted: the stored `BoxExtent` is correct, it serializes
correctly, and the next level load or physics-state recreate produces a consistent world. The
damage is confined to the live session and self-heals. Two workarounds exist and both are
measured above. Calling this Critical would rank it alongside data loss and push genuinely
unrecoverable tickets down the queue.

*Why not Medium.* Medium is a soft blocker — doable via a documented workaround. The
workaround exists but it is **not discoverable from the failure**, because there is no
failure to notice: nothing warns, nothing degrades, and the documentation
(`wiki-src/actor.md:248`) affirmatively tells the reader that collision setup is handled. A
workaround you only find after your collision is already wrong in a shipped level is not a
Medium-grade mitigation.

## Related

- `B-set-component-properties-no-change-notification` (**DONE**) — **this defect was buried
  inside that ticket's body**, under § *Shape extents: only half closed*, added by its `#3`
  entry. It **survived that ticket's fix** and was still true when `#5` verified and closed
  it; `#5` says so explicitly and notes it had no ticket of its own. The two are distinct
  defects: that one was a missing notification, this one is a notification that runs and
  still leaves half the state stale. Filed separately so it is not closed by association.
- `B-set-component-properties-staticmesh-shadow-corruption` (DONE) — the precedent for the
  third fix option: two properties routed to typed setters because the setter is a superset of
  the notification.
- `B-property-set-container-empty-change-event` (DONE) — the camouflaged sibling one verb over.
- `B-trace-complex-hits-render-geometry` — unrelated cause, adjacent symptom: another way a
  trace and a picture can disagree.

## History
- `#1-reproduced-and-split-out` `OPEN` reporter — Reproduced live on the running editor at plugin HEAD `b79ba53e`, level `/Game/Maps/Atlantis`, on a throwaway `TriggerBox` (`PWSHAPE_PHYSSTALE_01` / `TriggerBox_0`) spawned at `(20000,0,2000)` past the city and deleted afterwards; the level was not saved. `actor.set_component_properties {BoxExtent:{400,400,400}}` returned `{"applied":["BoxExtent"],"notified":["BoxExtent"]}`; `actor.get_bounding_box` returned extent `400` and a fixed-pose pinned-exposure capture showed the wireframe grow from 46 px to filling a 768 px frame (both matching the projection arithmetic), while `spatial.raycast` from `(20800,0,2000)` toward `(19000,0,2000)` returned the **baseline-identical** hit at `x=20040`, `distance 760`, and a parallel ray at `y=200` stayed a miss — i.e. the live physics body was still the 40-extent box. Control: `actor.set_collision false` then `true` (destroy/recreate physics state, nothing else changed) immediately moved the same rays to `x=20400`, `distance 400`, and turned the `y=200` miss into a hit — so the ray and the geometry were right all along. Second write, `BoxExtent {150,150,150}`, showed the worse direction: bounds and capture both report 150 while a ray at `y=300` — 150 cm outside the drawn box — still blocks at `x=20400`, an invisible collider in empty space. Workaround verified: Python `UBoxComponent::SetBoxExtent(150,150,150)`, called with the value the reflection write had already stored, resynchronised the body (ray `y=0` → `x=20150`, ray `y=300` → miss), because the work is `BodyInstance.UpdateBodyScale(..., true)` at `BoxComponent.cpp:40` and not the assignment. Mechanism confirmed against UE 5.8 source at the cited lines: `AggGeom` self-repairs through `UShapeComponent::GetBodySetup()` (`ShapeComponent.cpp:100-103`) so it is not the stale thing; `UShapeComponent::PostEditChangeProperty` (`:157-164`) does only `UpdateBodySetup()`; the handler commits with `MarkRenderStateDirty()` + `UpdateComponentToWorld()` (`ComponentHandler.cpp:356-357`) and contains no `RecreatePhysicsState` (grep-verified). **Split out of `B-set-component-properties-no-change-notification`, where this was written up inside the body under § *Shape extents: only half closed* and then went DONE around it** — that ticket's `#5` verification entry flagged that it had no ticket of its own and would otherwise close by association. It is a distinct defect: that one was a missing notification, this one is a notification that runs and still leaves half the state stale, and it survived that fix. Dedup: searched the board for `shape` / `extent` / `physics` / `body` / `collision` / `overlap` / `trace` and for the phrases `physics body`, `RecreatePhysicsState`, `SetBoxExtent`, `UpdateBodySetup` — the only hit is the parent ticket above. Also recorded: `docs/wiki-src/actor.md:248` tells readers the change hook drives "`ShapeComponent` collision setup", which is half true in the direction that keeps the reader from checking.
- `#2-route-shape-sizes-to-typed-setters` `IN-REVIEW` developer — Took the ticket's **third option**: the four shape size properties now route to their typed setters, mirroring the `StaticMesh` / `SkinnedAsset` precedent, rather than adding a `RecreatePhysicsState` policy to every notified write. New `Utils/ShapeExtentPropertyWrite.{h,cpp}` (`PinWright::ApplyShapeExtentProperty`) intercepts `UBoxComponent::BoxExtent` → `SetBoxExtent`, `USphereComponent::SphereRadius` → `SetSphereRadius`, `UCapsuleComponent::CapsuleHalfHeight`/`CapsuleRadius` → `SetCapsuleHalfHeight`/`SetCapsuleRadius`; `Handlers/Actor/ComponentHandler.cpp` calls it in the same ladder as `ApplyComponentAssetProperty`, only when that returns `NotApplicable`. The JSON value is resolved through the SHARED importer (landed on the real property, read back, previous value restored — including on failure, because the `FStructProperty` object branch can partially fill an `FVector` and still return false), so accepted syntax and error strings cannot drift from a generic write, and the setter performs the only mutation. Ticket confirmed against UE 5.8 source on every load-bearing claim: `UShapeComponent::PostEditChangeProperty` is `UpdateBodySetup()` only (`ShapeComponent.cpp:155-164`), `GetBodySetup()` self-repairs `AggGeom` (`:100-103`), `UPrimitiveComponent::PostEditChangeProperty` (`PrimitiveComponent.cpp:1541`) never touches physics, and the handler had no `RecreatePhysicsState`. **The engine call that actually closes it is `FBodyInstance::UpdateBodyScale(Scale3D, bForceUpdate=true)` (`BodyInstance.cpp:2204`)** — it walks the live `FPhysicsShapeHandle`s, reads each shape's `FKShapeElem` back out of its user data (the `AggGeom` element `UpdateBodySetup()` just refreshed) and rebuilds the Chaos implicit geometry; `bForceUpdate` is load-bearing because an extent edit does not change the component scale and the function returns early at `:2214` without it. Not `UpdateBodySetup` (already self-repairing), not `MarkRenderStateDirty` (rebuilds the half that was already right), and not `RecreatePhysicsState` (correct but destroys/rebuilds the whole body for an in-place size change). Enumerated the whole set from engine source: UE 5.8 has exactly three concrete `UShapeComponent` subclasses across `Source` and `Plugins` (grep `public UShapeComponent`), four size properties, each with the same `bPhysicsStateCreated → UpdateBodyScale(..., true)` tail. **One deliberate behaviour change, stated because it is real:** `CapsuleRadius` now follows the setter's rule (keep the requested radius, grow `CapsuleHalfHeight` to match, `CapsuleComponent.cpp:171-172`) instead of the Details-panel rule of clamping the radius down to the existing half-height (`:160-163`) — the setter is the only route that also updates the body, and honouring the requested value beats silently discarding half of it. The four properties now report in `applied` and not in `notified`, as the ticket predicted. **The new spatial occupancy verbs WERE exposed:** `SpatialTraceUtils::ProbeFootprintOccupancy` (`Handlers/Spatial/SpatialTraceUtils.cpp:370-393`) is `UWorld::OverlapMultiByChannel` with `FCollisionShape::MakeBox`, on `ECC_Visibility` by default, and `UShapeComponent`'s default `OverlapAllDynamic` profile answers Overlap on Visibility (`BaseEngine.ini:3107`) — so `spatial.find_clear_placement` and `MeasureFootprintClearance`, which bisects the same probe, would have measured the OLD footprint after any extent write and reported occupied space clear. Regression tests in `Source/PinWright/Private/Tests/Actor/TestShapeExtentPhysicsBody.cpp`, all behavioural and all measuring the live body through `UPrimitiveComponent::OverlapComponent` → `FBodyInstance::OverlapTest` (no channel, no profile, no world state to make the result host-dependent): `BoxExtentUpdatesThePhysicsBody` (a 1 cm probe 200 cm out is missed at extent 40, HIT after the verb writes 400, and missed again after it writes 50 — the grow assertion is the one that fails on the pre-fix handler, and the shrink assertion pins the invisible-collider direction), `ShapeExtentReachesFootprintOccupancy` (drives `ProbeFootprintOccupancy` directly, filtered to the probe actor by pointer identity — this is the proof the spatial verbs are safe), `SphereAndCapsuleExtentsUpdateThePhysicsBody`, and `ShapeExtentRoutingBoundary` (an ordinary shape property must still appear in `notified`). Doc line the ticket flagged is fixed: `Docs/wiki-src/actor.md` no longer claims the change hook drives "`ShapeComponent` collision setup" and now documents the four routed properties and the capsule divergence. NOT COMPILED and NOT RUN — the wave orchestrator owns the build and the suite; the failing-before/passing-after claim is reasoned from source, not measured.
