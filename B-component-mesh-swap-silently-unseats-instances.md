---
id: B-component-mesh-swap-silently-unseats-instances
title: "Re-pointing a component's StaticMesh moves every instance's geometry by the difference between the two meshes' pivot-to-bottom offsets, and actor.set_component_properties reports none of it — the write path's own comment says a mesh swap changes every instance's bounds and rebuilds the HISM cluster tree for exactly that reason, then returns applied:[StaticMesh] with no delta, so a ground-seated scatter comes back unseated with every reported number correct"
status: OPEN
severity: Medium
category: bug
tags: [actor, set_component_properties, add_component, static-mesh, ism, hism, instanced-static-mesh, mesh-swap, re-point, pivot, bounds, grounding, seating, ground_instances, silent-wrong-output, no-readback, diagnostic, vegetation]
encounters: 1
costly: 1
lastSeen: 2026-08-29T18:20:00+05:00
---

# The write path knows the bounds moved. It acts on that, and tells the caller nothing.

Re-pointing a placed component's mesh is a supported typed write:
`actor.set_component_properties {actorName, componentName, properties:{"StaticMesh":"/Game/..."}}`
(`Handlers/Actor/ComponentHandler.cpp:188`), which routes the `StaticMesh` key out of the generic
reflection loop into `PinWright::ApplyComponentAssetProperty` at `:322-323`.

That routing exists because a raw reflection store is wrong here, and the handler is careful about
it — the comment at `ComponentHandler.cpp:316-321` explains that a reflection write leaves the
engine's private shadow copy stale and turns the later `UpdateComponentToWorld()` into
`Ensure condition failed: KnownStaticMesh == StaticMesh`. So the write goes through the engine
setter (`Utils/ComponentAssetPropertyWrite.cpp:134`, `StaticMeshComp->SetStaticMesh(Requested);`)
and is then **verified against the component rather than the return value** (`:141-148`), because
`SetStaticMesh` returns false both for "already this mesh" and for "refused" (`:136-140`).

Then it does one more thing, and the comment on it is the whole ticket.
`ComponentAssetPropertyWrite.cpp:150-155`, verbatim:

```cpp
// The cluster tree is NOT rebuilt by SetStaticMesh, and a mesh swap changes
// every instance's bounds. Mirrors the StaticMesh branch of
// UHierarchicalInstancedStaticMeshComponent::PostEditChangeChainProperty
// (HierarchicalInstancedStaticMesh.cpp:2120-2129), including its
// FApp::CanEverRender() guard and its synchronous, forced form - the details
// panel takes exactly this path when a user swaps the mesh on a HISM.
```

followed by `Hism->BuildTreeIfOutdated(/*Async*/ false, /*ForceUpdate*/ true);` at `:161`.

**The handler states that a mesh swap changes every instance's bounds, and rebuilds the
acceleration structure because of it.** It then returns `EComponentAssetWrite::Applied` (`:164`),
and the response is `applied: ["StaticMesh"]` (`ComponentHandler.cpp:381`) with nothing about where
those bounds went. The knowledge is in the code path, spent entirely on the renderer, and withheld
from the caller who is the only party that can act on it.

## What actually moves, and why a correct write produces wrong geometry

A per-instance transform positions the mesh's **origin**. Different meshes hang different distances
below their own origin, so a re-point holds the origin fixed and translates the visible geometry by
the difference of the two offsets, scaled by each instance's own scale. Nothing about the instance
array changes — the transforms that were correct for the old mesh are still exactly the transforms
that were written, and they are now wrong.

**Measured**, re-speciating an authored scatter over zone F of `PW_VegetationTest`
(`Docs/map/vegetation-zone-f.md` § Re-speciation, Findings 3). 477 instances across six components
were re-pointed to preserve every authored position, yaw and lean. The four pivot-to-bottom offsets
involved, as recorded there: `HillTree_P2` **-37.5**, `SM_Bush_Dire_Bramble` **-12.0**,
Bird-of-Paradise **-2.1**, rock shelf **-7.5**. Where the replacement hung less far below its own
origin than the mesh it replaced, the geometry floated:

> 31 instances were sunk afterwards (`dev/zonef2/f_reseat.py`): re-pointing preserves the instance
> **origin**, and each mesh hangs a different distance below its own [...], so where the replacement
> hangs less far the geometry floats - up to 83 cm here. **Any component re-point needs a seat
> pass; the swap alone does not ground it.**

Every number the RPC reported was true. `applied: ["StaticMesh"]` was true. The read-back at
`:141-148` was true. The cluster tree really was rebuilt. 31 instances were in the air.

## The deciding number is already a first-class readback. Nothing subtracts it.

This is not a missing measurement. `static_mesh.describe`
(`Handlers/Asset/StaticMeshDescribeHandler.cpp:14`) returns `bounds` in the asset-dump shape —
built at `Handlers/Asset/StaticMeshDumpBuilder.cpp:28` from `Mesh->GetExtendedBounds()` through
`Handlers/Asset/MeshBoundsHelpers.h:11-27`, which emits `origin{x,y,z}` and `extent{x,y,z}`. A
mesh's pivot-to-bottom offset is therefore `bounds.origin.z - bounds.extent.z`, one subtraction from
a shipped field, for either mesh.

And at the moment of the swap the handler is holding both of them:
`ComponentAssetPropertyWrite.cpp:124` binds `Requested`, and `:141` reads
`StaticMeshComp->GetStaticMesh()` — which one line earlier, at `:134`, was still the outgoing mesh.
The delta is computable inside the branch that already knows it matters, from two objects already in
scope. **This is emission, not computation, and not even a new response field**: `warnings[]` is
already built and emitted on this verb (`ComponentHandler.cpp:390-395`), and the comment at
`:383-389` records that dropping those warnings was itself a filed defect.

## Precedent for the shape of the fix, on a verb one call away

`static_mesh.describe` already ships exactly this kind of forward-looking diagnostic. Its registered
summary (`StaticMeshDescribeHandler.cpp:15`) promises *"rebuildRenderConsumers, the live components
a rebuild of this mesh in place would have to quiesce first"*, emitted at `:69`. The comment above
it (`:33-38`) states the design rule this ticket is asking to apply once more:

> Without this field the refusal is the first time a caller hears such a component exists. It is
> computed from the same scan the guard acts on [...], so the read and the refusal cannot disagree
> about who is holding the mesh.

A mesh swap on a populated ISM/HISM is the same situation with the roles reversed: the plugin knows
a consequence the caller does not, and today the first time the caller hears about it is when they
look at a picture.

## Same shape as

`B-foliage-paint-does-no-ground-projection` § *Same shape as*: *the call succeeds, every number it
reports is correct, and the output is wrong because the deciding number was never reported.*

**The variant here is that the write path names the deciding quantity out loud and then spends it
internally.** In every sibling below, the unreported number is one nobody computed. In this one
`ComponentAssetPropertyWrite.cpp:150-151` states *"a mesh swap changes every instance's bounds"* and
`:161` acts on that statement — the plugin both knows the bounds moved and does work because of it,
and still emits nothing. The deciding number is the pivot-to-bottom delta between the outgoing and
the incoming mesh.

Nearest siblings, all this session, all different mechanisms:

- `F-ism-create-and-clear-scatter` `#2` (OPEN, Medium) — the same session's other encounter of the
  class on the same scatter workflow, and the counterpart hazard: that one is the cost of
  clear-and-refill, this one is the cost of the re-point that avoids it.
- `B-wiki-hism-recipe-resets-actor-transform` (OPEN, Medium) — deciding number is the actor's
  location, with a green `placed: 41` from an unrelated verb confirming the damage.
- `B-pcg-spawner-mixed-mesh-heights-silently-misscaled` (OPEN, Medium) — deciding number is the
  height ratio between meshes sharing one scale range. Closest in subject: both are "two meshes were
  treated as interchangeable and the geometry moved".
- `B-ground-probe-hits-hull-not-render` (DONE, High) — its § *Why a warning and not just
  documentation: the numbers looked right* is the argument for why a `warnings[]` entry is the fix
  here too. An 83 cm float on instances whose transforms are all exactly as authored is an in-family
  number, and in-family numbers cannot be caught by reading numbers.

## Fix

**Primary — a `warnings[]` entry on the mesh-swap branch**, emitted from
`Utils/ComponentAssetPropertyWrite.cpp` inside the `bIsStaticMesh` block when the component is an
ISM/HISM carrying instances and the two meshes' `origin.z - extent.z` differ:

    "StaticMesh re-pointed on a component carrying 178 instance(s): SM_Bush_Dire_Bramble sits
     12.0 uu below its origin, <new mesh> 2.1 uu, so every instance's geometry moved +9.9 uu
     before scale. Per-instance transforms were NOT changed. Re-seat with spatial.ground_instances."

Three properties worth keeping: it fires only when there are instances and the offsets differ (a
swap between two identically-pivoted meshes stays silent, and so does a swap on a plain
`UStaticMeshComponent`, whose transform the caller can already see); it names both offsets rather
than only the delta, so a caller can check the arithmetic; and it says the transforms were not
touched, which is the fact that makes the condition recoverable rather than alarming.

**Secondary — echo the numbers, not only the prose.** A `meshSwap: {previous, requested,
previousPivotToBottom, requestedPivotToBottom, deltaZ, instanceCount}` block lets a caller decide
programmatically instead of parsing a string. This is the half a fixer can drop if the warning alone
is judged enough.

**Explicitly NOT asked for: do not move the instances.** Re-seating is a placement decision — it
needs a surface spec, `samples`, `seatPercentile` and an embed depth, all of which
`spatial.ground_instances` already owns and none of which a property write has any business
choosing. A verb that silently corrected Z here would be a second, larger version of the same
defect. The ask is that the caller be told, once, that the seat is now stale.

## Related

- `F-ism-create-and-clear-scatter` (OPEN, Medium) — the other half of this workflow from the same
  session, and the sibling encounter of the recurring class. A re-point is the cheap alternative to
  clear-and-refill precisely because it preserves the transforms; this ticket is the cost of that
  preservation.
- `F-resimulate-existing-foliage-volume` (OPEN, Medium) — its `#2` establishes the foliage-side
  counterpart: `UFoliageType::Mesh` re-points already-spawned instances on the next frame via
  `PostEditChangeProperty`, which is the same silent retarget on a different owner. The seat is
  invalidated there too, and a fixer landing either warning should land both.
- `B-ground-instances-default-component-foreign-scatter` (OPEN, Critical) — why the recovery pass
  here was written offline rather than as a `spatial.ground_instances` call. Any warning text
  recommending that verb must not steer a caller into its default-component hazard; name
  `component` explicitly in the suggestion.
- `F-ground-instances-align-to-surface` (OPEN, Medium) — the re-seat this warning points at is
  currently vertical-only, so a swap on sloped ground is two gaps deep.
- `B-pcg-spawner-mixed-mesh-heights-silently-misscaled` (OPEN, Medium) — the closest precedent for
  the scoping of this ask: a mutation that silently mis-places instanced geometry, filed as a
  diagnostic PinWright can add rather than as a placement field it cannot own. Same reasoning, same
  deliberate narrowness.
- `E-static-mesh-describe-no-live-consumer-report` (IN-REVIEW, Medium) — the neighbouring ticket on
  the verb whose `rebuildRenderConsumers` field is this ticket's design precedent.
- `B-wiki-hism-recipe-resets-actor-transform` (OPEN, Medium) — the other shipped-artefact defect on
  the same scatter workflow, and the ticket carrying this class's canonical statement in its
  `## Same shape as` section.

## Severity

**Medium.** The impact class is argued down one band from High rather than asserted. The **shape**
is the High band's *silent wrong data on a normal path*: a supported typed write leaves 31 of 477
instances floating, the caller gets no signal, and the verb's `applied` list actively reads as
confirmation. One band is deducted because **no PinWright field is false**. `applied:
["StaticMesh"]` is a true statement about the property that was written; the verb never claimed to
have placed anything, and the instance transforms it did not touch are exactly the transforms
`actor.get_instances` will report. That is the difference between this and
`B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High), whose response asserts `projected`
about a projection that did not happen. Here the response is silent, and silence about a consequence
the code path has explicitly reasoned about lands on the Medium band's *"a readback omits a field
and forces a fallback"* — the fallback being a bespoke offline re-seat script.

**Explicitly not Low.** The Low band is friction where a doc or a field fails to help. What is
missing is not a convenience: it is the only observable separating a re-point that landed correctly
from one that unseated an entire authored layer, and the condition is invisible in every top-down
frame — an 83 cm float over a 160 x 240 m zone is sub-pixel from above and obvious at eye level. A
gap whose detection depends on which camera angle someone happened to use is not friction.

**Not Critical.** Nothing is corrupted or lost. The instance array is intact and internally correct;
only its relationship to the ground is stale, and one `spatial.ground_instances` call repairs it
once the caller knows.

**Reach modifier declined in both directions, and named.**
`actor.set_component_properties` is a common verb, but the `StaticMesh` key on a *populated
instanced* component is not an every-session path, so no bump up. No bump down either: the
bump-down band is a rare edge path, and this is the route the plugin's own ticket set steers callers
toward — a re-point is the cheapest way to re-speciate a scatter precisely because
`F-ism-create-and-clear-scatter` shows the alternative is unbuilt. The reading that would bump it up
is that the identical defect exists on the foliage side (`F-resimulate-existing-foliage-volume`
`#2`), which makes the *class* every-session even where this verb is not; that is an argument for
fixing both, not for re-rating this one. Medium stands unmodified.

## History
- `#1-mesh-swap-reports-no-pivot-delta` `OPEN` reporter — Found re-speciating zone F of
  `PW_VegetationTest` (`Docs/map/vegetation-zone-f.md` § Re-speciation, Findings 3): 477 instances
  across six ISM/HISM components were re-pointed to new meshes, preserving every authored position,
  and **31 came out floating by up to 83 cm** because a per-instance transform positions the mesh's
  origin and the four meshes involved hang different distances below their own — `HillTree_P2`
  -37.5, `SM_Bush_Dire_Bramble` -12.0, Bird-of-Paradise -2.1, rock shelf -7.5. Recovered with a
  bespoke offline re-seat (`dev/zonef2/f_reseat.py`). MECHANISM, source-read at HEAD: the typed
  route is `actor.set_component_properties` (`ComponentHandler.cpp:188`) dispatching `StaticMesh`
  to `PinWright::ApplyComponentAssetProperty` (`:322-323`), which calls `SetStaticMesh`
  (`ComponentAssetPropertyWrite.cpp:134`), verifies against the component rather than the bool
  return (`:141-148`, reasoning at `:136-140`), and then rebuilds the HISM cluster tree at `:161`
  under the comment at `:150-155` that states **"a mesh swap changes every instance's bounds"** — so
  the write path names the consequence, spends it on the renderer, and returns
  `EComponentAssetWrite::Applied` (`:164`) with a response of `applied: ["StaticMesh"]`
  (`ComponentHandler.cpp:381`) and no delta. THE DECIDING NUMBER IS ALREADY SHIPPED, which is why
  this ask is narrow: `static_mesh.describe` (`StaticMeshDescribeHandler.cpp:14`) returns `bounds`
  via `StaticMeshDumpBuilder.cpp:28` and `MeshBoundsHelpers.h:11-27`, so pivot-to-bottom is
  `bounds.origin.z - bounds.extent.z`; and at `ComponentAssetPropertyWrite.cpp:124` / `:134` /
  `:141` the handler holds both meshes in scope. The response already carries a `warnings[]` array
  (`ComponentHandler.cpp:390-395`) whose earlier omission was itself a filed defect (`:383-389`), so
  the fix is emission into an existing field. DESIGN PRECEDENT on a neighbouring verb: the same file
  family already ships `rebuildRenderConsumers` (`StaticMeshDescribeHandler.cpp:15`, `:69`,
  rationale `:33-38`) — *"without this field the refusal is the first time a caller hears such a
  component exists"* — which is this ask with the roles reversed. Deliberately NOT asked for: any
  automatic Z correction, because re-seating needs a surface spec, `samples`, `seatPercentile` and
  an embed depth that `spatial.ground_instances` owns and a property write must not choose. Rated
  **Medium**: the shape is the High band's silent-wrong-output on a normal path, argued down one
  band because no PinWright field is false — `applied` is a true statement about the property
  written, unlike `B-foliage-paint-does-no-ground-projection` (IN-REVIEW, High) whose response
  asserts `projected` about a projection that did not happen; not Low, because the missing
  observable is the only thing separating a correct re-point from an unseated layer and the
  condition is sub-pixel from every top-down camera; not Critical, because nothing is lost and one
  `ground_instances` call repairs it. Reach declined both ways, with the up-bump reading named and
  refused: the identical defect on the foliage side (`F-resimulate-existing-foliage-volume` `#2`)
  makes the class every-session, which argues for fixing both rather than re-rating this one.
