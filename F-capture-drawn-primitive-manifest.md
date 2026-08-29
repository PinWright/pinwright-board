---
id: F-capture-drawn-primitive-manifest
title: "render.capture_open_level returns an image and no inventory of what is in it — the subject/framing block answers 'is this ONE named actor provably out of frame' from a single GetActorBounds AABB, which on an instanced scatter is the holder's whole 160 x 240 m box, so the level's 30 000 instances are unidentifiable and 'what is that thing in my screenshot' has no supported answer"
status: OPEN
severity: Medium
category: feature
tags: [render, capture_open_level, capture, manifest, subject, framing, ism, hism, instanced-static-mesh, per-instance, look-dev, verification, missing-verb, vegetation]
encounters: 1
lastSeen: 2026-08-29T18:20:00+05:00
---

# The capture publishes a picture and no census of what it drew

`render.capture_open_level` (`Handlers/Render/RenderHandler.cpp:1513`) returns a PNG, the pose the
renderer resolved to, `imageStats`, a `blank` verdict, and — when a `subject` is supplied — a
`framing` block. Nothing in that response names a single thing the frame contains.

That is a gap and not an oversight, because the verb already carries the *idea* of frame content.
Its `subject` parameter description says so (`RenderHandler.cpp:1530-1537`): *"What this frame is
supposed to CONTAIN"*, buying *"the `framing` block — the verdict, measured against the pose the
renderer resolved to, on whether the subject is provably out of frame"*. The block is emitted at
`:1727-1741` through `EvaluateBoundsFraming` (`Handlers/Render/PreviewViewportCaptureUtils.cpp:982`).

**But it answers a different question, singular and coarse.** The caller must already know what they
are looking for and name it; `kind` is `world` or `actor` (`:1531-1532`); and the geometry it
reasons about is one sphere. For `kind:"actor"` the bounds come from
`Actor->GetActorBounds(false, Origin, Extent)` (`Handlers/Render/CaptureSubjectProviders_Level.cpp:382`),
reduced to `BoundsOrigin` / `BoundsRadius` at `:393-394` with `BoundsSource` `actorBounds` at `:395`.

## Why one AABB per actor is the wrong object for a scatter — the plugin has already said so

`GetActorBounds` unions the actor's components, so on the holder pattern the plugin's own wiki
teaches (`level-building.instancing-and-scatter`) it returns the box around the entire scatter.
Concretely, on `PW_VegetationTest` zone F (`Docs/map/vegetation-zone-f.md`): `ZoneF_Veg` is one
actor carrying **15 HISM components and 5888 instances of six species** spread across a
160 x 240 m zone, and its actor pivot sits at the world origin *outside the zone entirely*. Asking
`framing` about it returns "in frame" for essentially any camera standing in the zone. It cannot
say which species, which component, or which of 5888 instances — and "which one" was the whole
question every time.

**The plugin has already made exactly this judgement once, in `spatial`.** The header comment on
`Handlers/Spatial/MeasureHandler.cpp:9-13`, verbatim:

> The first three read actor world-space AABBs the same way actor.get_bounding_box does
> (GetActorBounds(false,...) -> FBox) so their numbers agree with that read.
> find_clear_placement does not: **an AABB per actor cannot answer an occupancy SEARCH**, so it
> queries the physics scene with a box overlap, which resolves against ISM/HISM per-instance
> bodies as well as actors.

A frustum is a region and "what is in this picture" is an occupancy search. `spatial` accepted that
a per-actor AABB is the wrong primitive for it and moved to per-instance resolution;
`render.capture_open_level` is still on the AABB.

## Measured — what it cost, and what had to be written instead

Re-speciating zone F (`Docs/map/vegetation-zone-f.md` § Re-speciation, Findings 2). A species defect
took **four captures** to pin down because no response could say which mesh was the dark mass in the
frame. The pass had to write `dev/zonef2/f_whatis.py`, which projects every instance of every
ISM/HISM into the camera frustum analytically and lists what is in frame, nearest first, with
apparent pixel height:

> **Wanted: `render.capture_open_level` returning an optional manifest of the instanced primitives
> it drew.** Today "what is that thing in my screenshot?" has no supported answer for the 30 000
> instances that make up this level.

Two species were cut on the strength of a single close frame each — `SM_Tree_Radiant_Blossom`
(*"a wall of smooth untextured mauve lobes ... two reviewers read it as a missing material"*) and
`PivotPainter/StaticMesh/tree` (*"at 12 m it is a spray of flat, tan, bark-textured blades"*). Both
decisions required knowing which mesh occupied which part of the frame, on a map where six species
share a canopy. That identification is currently a human squinting at a picture or a bespoke script.

## Proposed shape

An **optional** `manifest` block on `render.capture_open_level`, off by default so no existing
caller pays for it:

    render.capture_open_level {..., manifest: {enabled: true, maxRows: 200,
                                              minPixelHeight: 8, groupBy: "mesh"}}

    -> manifest: {
         rows: [ {actor, component, meshPath, instanceIndex, instanceCount,
                  distanceCm, apparentPixelHeight, screen:{x,y}, occluded?} ],
         rowsTruncated: false, rowsDropped: 0,
         totalCandidates: 5888, evaluatedComponents: 15
       }

`groupBy: "mesh"` collapses to one row per mesh with a count and the nearest representative, which
is the form that answers the actual question (*which species is that*) in a response a caller can
read; `groupBy: "instance"` gives the raw rows for a caller who needs to address one.

**Do not build it on a physics query.** `find_clear_placement`'s box overlap resolves against
per-instance *bodies*, and on this project that route sees almost nothing: every stock vegetation
mesh ships with zero simple collision, and the auto-created `UFoliageType` carries
`CollisionProfileName = NoCollision` (`Docs/map/vegetation-zone-f.md` § *Engine foliage cannot
answer a ground probe at all*). A manifest that silently omitted every collisionless plant would be
a worse lie than no manifest.

**The non-physics primitive is already in the tree.**
`UInstancedStaticMeshComponent::GetInstancesOverlappingBox(WorldBox, /*bBoxInWorldSpace=*/true)` is
called at `Handlers/Spatial/SpatialTraceUtils.cpp:480`, in a purely geometric resolver that upgrades
a component-level occupant to a named instance index — no physics, no collision requirement. A
frustum AABB feeding that, then a per-instance frustum test, is the whole computation. The response
shape is precedented too: `spatial.find_clear_placement` already emits `instanceIndices[]`
(`Handlers/Spatial/MeasureHandler.cpp:1227`) beside `instanceCount`
(`Instanced->GetInstanceCount()`, `:1231`).

## Why the client-side workaround is not adequate

It is real — the pass wrote it — and it is expensive in three separate ways:

1. **Round-trips.** `actor.get_instances` clamps `limit` to `InstanceRpcMaxBatch`
   (`Handlers/Actor/InstancedMeshHandler.cpp:257-258`) with `InstanceRpcDefaultLimit` as the
   default, so pulling 30 000 instances is tens of calls before any projection happens.
2. **The transcription ceiling.** `Docs/map/vegetation-polish.md` § 5.7 measured ~130 KB each way for
   2600 instances; a whole level's scatter does not fit through the wire for a question whose answer
   is one line of text.
3. **The projection is re-derived per caller.** The camera basis, aspect, FOV and near plane all have
   to be reconstructed client-side from the pose echo, and getting any of them wrong produces a
   plausible wrong answer — which is the failure mode the manifest exists to end.

In practice callers do what this pass did: drop to `python.execute` and write the projection by
hand, once per agent, with no shared correctness.

## Related

- `B-raycast-screen-lacks-trace-diagnostics` (OPEN, Medium) — **the nearest neighbour and
  deliberately not a duplicate.** Same underlying frustration, opposite question and opposite
  mechanism: that ticket is *"what is under this pixel"*, answered by a physics trace, and its
  entire second half is that neither raycast verb can name a struck instance. This ticket is
  *"what is in this frame"*, answered by analytic projection with no trace at all — which matters
  precisely because a trace cannot see collisionless vegetation, the case both tickets were found
  on. They should be worked together (`SpatialTraceUtils.cpp:435` / `:480` serve both) and neither
  closes the other: a per-pixel pick still cannot enumerate, and an enumeration still cannot
  disambiguate two instances behind one pixel.
- `B-ortho-capture-culls-distant-foliage` (OPEN, High, encounters 4) — the strongest argument that
  this block is verification infrastructure and not a convenience. Its `#5` measured a whole-map
  ortho retaining **0.4 %** of the instanced foliage a tile-scale ortho shows, and that number was
  obtainable only by comparing two captures. A manifest turns it into a field, and its `#4`/`#6`
  note that HISM/foliage recovery under `viewDistanceScale` is *still unverified* — a manifest is
  the instrument that verification needs.
- `B-capture-open-level-pose-params-photograph-stale-grass` (OPEN, High) — the same verb
  photographing content that does not match the level. Its fix option 2 is *publish what the frame
  was actually built around*; this is the general form of that remedy.
- `B-ortho-capture-renders-no-landscape-grass` (OPEN, High) — mechanism explicitly unattributed. A
  drawn-primitive census is the first thing that would narrow it, and landscape grass is the one
  content class a per-instance manifest would still miss, which a fixer must state rather than
  quietly omit.
- `B-unlit-level-capture-no-warning` (OPEN, Medium) — the same principle one field over: the
  response knows something about the scene it declines to report.
- `F-ortho-tile-reference-compare` (OPEN, High) — the other "the picture and the level disagree"
  feature. Both want the capture to be measurable rather than merely viewable.

## Severity

**Medium.** Impact class is the rubric's soft-blocker band verbatim — *"doable, but only via a
documented workaround, a source dive, or many extra calls"* — and it is all three at once: the
route is undocumented, it requires reading the camera-basis reconstruction out of the pose echo, and
it costs tens of `actor.get_instances` calls or a hand-written `python.execute` projection. The task
is not impossible, which is what separates this from the higher half of the missing-verb band; it is
that every caller pays for it separately and none of their implementations agree.

**Not High.** The High band is silent false-success or silent wrong data. Nothing here is false:
`render.capture_open_level` never claims to enumerate content, and `framing` is honest about the
narrow question it answers — it says "this named subject is provably out of frame" and that verdict
is correct for the AABB it was given. The gap is silence, and the image itself is right.

**Not Low.** Low is friction where a doc or a field fails to help. What is missing is the only route
from a rendered frame back to the assets in it, on a plugin whose look-dev workflow is *capture,
judge, change, re-capture*. Three species decisions on this level were made from single frames, and
each needed an identification the RPC surface could not supply.

**Reach modifier declined, with the up-bump named because it is arguable.**
`render.capture_open_level` genuinely does run in almost every session on this project, which by a
literal reading of the rubric would bump this to High. It is declined because the rubric's reach
modifier exists to rank a *defect* by how many callers meet it, and every caller of this verb today
gets a correct image; what they do not get is an optional block that bites only when the frame's
content is instanced and ambiguous. Bumping a feature request for an opt-in field on the strength of
the host verb's call count would let any proposed addition to a hot verb outrank real defects on
cold ones. A reviewer who reads reach as "how often is the *situation* met" rather than "how often
does the *method* run" gets High, and on a look-dev-heavy project that reading is defensible; it is
recorded here rather than hidden. No bump down: this is not an edge path — instanced scatter is the
subject of a shipped wiki page, three typed verbs and two open feature tickets. Medium stands.

## History
- `#1-capture-publishes-no-content-census` `OPEN` reporter — Found re-speciating zone F of
  `PW_VegetationTest` (`Docs/map/vegetation-zone-f.md` § Re-speciation, Findings 2): a species
  defect took four captures to pin down because no response can name what a frame contains, and the
  pass had to write `dev/zonef2/f_whatis.py` to project every ISM/HISM instance into the camera
  frustum analytically and list what is in frame, nearest first, with apparent pixel height. SOURCE
  READ AT HEAD, and the reason this is not a duplicate of the `subject` block that already ships:
  `render.capture_open_level` (`RenderHandler.cpp:1513`) does carry a content concept — its
  `subject` param says *"What this frame is supposed to CONTAIN"* (`:1530-1537`) and emits `framing`
  at `:1727-1741` via `EvaluateBoundsFraming` (`PreviewViewportCaptureUtils.cpp:982`) — but it is
  singular, caller-named, and reasons about ONE sphere derived from
  `Actor->GetActorBounds(false, Origin, Extent)` (`CaptureSubjectProviders_Level.cpp:382`,
  `BoundsRadius` `:394`, `BoundsSource` `:395`). On the holder pattern the plugin's own wiki teaches,
  that is the box around the whole scatter: `ZoneF_Veg` is one actor with 15 HISM components, 5888
  instances of six species over 160 x 240 m, and an actor pivot at the world origin outside the
  zone — so `framing` answers "in frame" for any camera in the zone and cannot name a species, a
  component or an instance. THE PLUGIN HAS ALREADY MADE THIS CALL ONCE: `MeasureHandler.cpp:9-13`
  records that *"an AABB per actor cannot answer an occupancy SEARCH"* and that
  `find_clear_placement` therefore resolves per-instance; a frustum is a region and this is the same
  question. IMPLEMENTATION NOTE, load-bearing: do NOT build it on the physics overlap
  `find_clear_placement` uses — every stock vegetation mesh here has zero simple collision and the
  auto-created `UFoliageType` is `NoCollision`, so that route would silently omit the content the
  manifest exists to name. The correct primitive is already called at `SpatialTraceUtils.cpp:480`,
  `GetInstancesOverlappingBox(WorldBox, true)`, a purely geometric per-instance resolver; and the
  response shape is precedented by `find_clear_placement`'s `instanceIndices[]`
  (`MeasureHandler.cpp:1227`) beside `instanceCount` (`:1231`). WHY THE CLIENT-SIDE WORKAROUND DOES
  NOT SUFFICE: `actor.get_instances` clamps `limit` to `InstanceRpcMaxBatch`
  (`InstancedMeshHandler.cpp:257-258`), so 30 000 instances is tens of calls; the transcription
  ceiling was measured at ~130 KB each way for 2600 instances (`Docs/map/vegetation-polish.md`
  § 5.7); and the frustum projection is re-derived per caller from the pose echo, where a wrong
  camera basis yields a plausible wrong answer. Asked for: an OPT-IN `manifest` block with
  `groupBy:"mesh"|"instance"`, `minPixelHeight`, `maxRows`, and explicit `rowsTruncated` /
  `rowsDropped` (following `E-ground-instances-two-truncation-flags`' complaint rather than
  repeating it). DEDUP against `B-raycast-screen-lacks-trace-diagnostics` (OPEN, Medium) stated
  explicitly in the body and NOT collapsed: same frustration, opposite question (enumerate a frame
  vs pick a pixel) and opposite mechanism (analytic projection vs physics trace), and the trace
  route cannot see the collisionless vegetation both tickets were found on — neither closes the
  other, though they share `SpatialTraceUtils.cpp:435`/`:480` and should be worked together. Rated
  **Medium** on the soft-blocker band, all three of its clauses at once; not High because nothing in
  the response is false and the image is correct; not Low because this is the only route from a
  rendered frame back to the assets in it on a plugin whose look-dev loop is capture-judge-change.
  REACH BUMP NAMED AND DECLINED: `capture_open_level` does run almost every session, which read
  literally gives High, but every caller gets a correct image today and the missing piece is an
  opt-in block that bites only on ambiguous instanced content; bumping an addition to a hot verb on
  the host's call count would outrank real defects on cold ones. The contrary reading — reach as
  "how often the situation is met" — is defensible on a look-dev-heavy project and is recorded
  rather than hidden.
