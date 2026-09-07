---
id: B-raycast-screen-lacks-trace-diagnostics
title: "The collisionless-mesh fix landed on spatial.raycast and never reached spatial.raycast_screen: the screen pick still has a bare traceComplex with no warning, none of the accept-filters or multiHit, and a response with no simpleCollisionShapes / renderGeometryHit — so a pick over instanced vegetation returns a confident hit on whatever is behind it and nothing in the response can say why"
status: OPEN
severity: Medium
category: bug
tags: [spatial, raycast_screen, raycast, trace, collision, foliage, hism, instance-index, incomplete-fix, sibling-verb-drift, diagnostics, vegetation]
encounters: 2
costly: 1
lastSeen: 2026-08-29T18:20:00+05:00
---

# One of the two trace verbs was taught about collisionless meshes

`B-trace-complex-hits-render-geometry` (DONE, High) is the ticket for "a mesh with no simple
collision still blocks a complex trace, and the naive call silently produces a wrong height". Its
fix was three parts: document the hazard, give the verb accept-filters and `multiHit` so the correct
call is expressible, and surface `simpleCollisionShapes` / `renderGeometryHit` in the response so
the condition is detectable.

All three landed on `spatial.raycast`. **None of them reached `spatial.raycast_screen`**, which is
the verb whose entire purpose — mapping a pixel in a capture back to the object that drew it — is
the case where the caller has the least idea what is in front of what.

## The two verbs, side by side at HEAD

| | `spatial.raycast` | `spatial.raycast_screen` |
|---|---|---|
| registration | `Handlers/Spatial/RaycastHandler.cpp:248-288` | `Handlers/Spatial/RaycastScreenHandler.cpp:74-108` |
| `traceComplex` doc | full warning: *"complex collision for a StaticMesh IS its render triangle soup, so a mesh with NO collision set up still BLOCKS the ray … Filter with onlyActors/actorFilter/onlyClasses, or use multiHit and pick the hit you want"* | *"Trace against per-triangle (complex) collision instead of simple collision. Default false."* (`:101-103`) — one sentence, no warning |
| namespace-level warning in the description | yes (`:249`) | no (`:75`) |
| `multiHit` / `maxHits` | yes | **no** |
| `onlyActors` / `actorFilter` / `onlyClasses` | yes | **no** |
| `simpleCollisionShapes` in response | yes (`RaycastHandler.cpp:199`) | **no** |
| `renderGeometryHit` in response | yes (`:203`) | **no** |

Both bottom out in the same helper — `RaycastScreenHandler.cpp:263-264` calls
`SpatialTraceUtils::TraceLine`, which `RaycastHandler.cpp:501` also calls, reaching
`World->LineTraceSingleByChannel` at `SpatialTraceUtils.cpp:306` with `bReturnFaceIndex = true`
(`:303`). The diagnostic fields are defined on the shared struct (`SpatialTraceUtils.h:47-60`), so
the data the screen verb needs is already computed on its own code path and then not emitted. Its
full response is `channel` (`:268`), `ray{origin,direction}` (`:273-275`), `view{…}` (`:280-297`),
`hit` (`:302`/`:307`), `location` / `normal` / `distance` (`:308-310`), `actor{name,path}`
(`:315-317`) and `component` (`:321`).

## Measured

Look-dev polish over `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 5.6): a screen pick
aimed at a tree canopy returned a hit on the landscape roughly **86 m behind** the canopy — a
well-formed answer naming the wrong object, with `hit: true` and a plausible distance.

**Honest gap in that measurement, recorded rather than papered over:** the source note does not
record whether `traceComplex` was set on that call, and the default is `false`
(`RaycastScreenHandler.cpp:200-201`). So the likeliest reading is the ordinary one — a simple-collision
trace correctly passing through vegetation that has no simple collision — rather than anything
exotic. That does not weaken the ticket; it sharpens it. **With `traceComplex:false` the engine's
answer is right and the response still cannot say what happened**, because there is no field
distinguishing *"nothing was on that ray"* from *"the thing on that ray has no simple collision and
was skipped"*. That is exactly the distinction `simpleCollisionShapes` and `renderGeometryHit` were
added to `spatial.raycast` to draw, and the screen verb is the one where the caller cannot infer it
from context — they are looking at a picture of the object they just failed to hit.

## Second half: neither verb can name the instance

Even when a screen pick does land on an instanced scatter, the response says `component` and stops
(`RaycastScreenHandler.cpp:321`). `FSpatialHit` (`SpatialTraceUtils.h:30-60`) carries `HitComponent`
and **no instance index**; the only `InstanceIndex` in the module is on `FSpatialOccupant`
(`SpatialTraceUtils.h:269`), which belongs to the box-overlap footprint path, not to either
raycast.

The plugin can already do this and does it elsewhere: `SpatialTraceUtils.cpp:435` resolves a
blocking instance's index via `Overlap.GetItemIndex()`, with a `GetInstancesOverlappingBox` fallback
at `:480`, and `spatial.find_clear_placement` reports `instanceIndices[]` (`MeasureHandler.cpp:1227`)
alongside `GetInstanceCount()` (`:1231`). So *"which instance of this HISM did I click"* is answerable with
code that exists, on a hit result that already sets `bReturnFaceIndex`.

For a level built the way the plugin's own wiki teaches — `level-building.instancing-and-scatter`,
a caller-owned HISM holding thousands of instances — `component: "HISM_ZF_Oak"` is the whole answer,
and the question was which of 50.

## Fix

Bring the sibling up to parity rather than inventing anything:

1. **Response fields.** Emit `simpleCollisionShapes` and `renderGeometryHit` on the screen verb's
   hit block. They come off the same `FSpatialHit` (`SpatialTraceUtils.h:47-60`) the shared
   `TraceLine` already populates, so this is emission, not computation.
2. **The `traceComplex` description and the verb summary.** Copy the warning text from
   `RaycastHandler.cpp` verbatim. A caller reading `raycast_screen.md` today gets no hint that the
   default silently ignores collisionless geometry, and the sibling page says it twice.
3. **`multiHit` + the three accept-filters.** `multiHit` is the one that matters most here: the
   natural screen-pick intent is *"tell me everything along this ray, nearest first"*, and the
   caller can then pick. The filters follow the same shape as the sibling's and share its
   implementation.
4. **Instance index on the hit** (both verbs), reusing `SpatialTraceUtils.cpp:435` / `:480`.

Items 1–2 are the honesty half and are small. 3–4 are the capability half; if they are split,
split them there.

## Workaround, and why it holds this at Medium rather than higher

`raycast_screen` echoes the deprojected ray it built (`ray.origin` / `ray.direction`,
`RaycastScreenHandler.cpp:273-275`), specifically so a caller can confirm pose-coherence with the
image. That echo is also an escape hatch: feed those two vectors to `spatial.raycast` with
`traceComplex:true, multiHit:true` and the full diagnostic surface applies. Two calls instead of
one, and the second call is the fixed verb.

The workaround is real, undocumented, and not discoverable from either method page — nothing on
`raycast_screen.md` suggests handing the echoed ray to the other verb, and a caller who has not read
`B-trace-complex-hits-render-geometry` has no reason to think there is anything to escape from.

## Related

- `B-trace-complex-hits-render-geometry` (DONE, High) — the parent. Its fix is correct and complete
  for the verb it touched; this ticket is the half of the surface it did not reach. Its own framing
  (*"make the correct call expressible, and make the wrong one loud"*) is the acceptance criterion
  here.
- `F-ism-per-instance-transforms` (IN-REVIEW, High) — source of the instance-index citations, and
  the ticket that first observed the plugin can see an instance and not name it.
- `F-spatial-raycast-no-batch-multi-origin` (OPEN, Low) — the other `raycast_screen` gap, single-pixel
  rather than single-ray. Same verb, unrelated axis.
- `B-ortho-capture-culls-distant-foliage`, `B-capture-open-level-pose-params-photograph-stale-grass`
  — the neighbouring "the picture and the level disagree" family; a screen pick is how a caller
  reconciles them, which is why this verb matters more than its call count suggests.

## Severity

**Medium.** Impact class is the rubric's Medium band, *"doable, but only via a documented workaround
… or many extra calls"* — the echoed-ray + `spatial.raycast` route works and is two calls, and the
`raycast` half of it is already fixed.

**Not High, argued.** The High band is silent wrong data that the caller trusts and builds on, and
the parent ticket was rated High for exactly that. The difference is what the response claims. On a
default `traceComplex:false` pick the hit reported is genuinely the first blocking hit, and the verb
promises no more than that; there is no field asserting anything untrue. What is missing is the
diagnostic that would let the caller notice the answer is not the one they meant. A reviewer could
argue the parent's reasoning applies unchanged — a screen pick is a *"what is this"* question and a
confident answer naming the wrong object is a lie by any practical standard — and that reading gives
High. It is declined here because the two verbs are not symmetric in exposure: `spatial.raycast`'s
wrong answer becomes a heightmap and propagates, while a screen pick's wrong answer is read once by
a caller who is simultaneously looking at the image.

**Reach modifier declined in both directions.** No bump up: `raycast_screen` is not an
every-session verb. No bump down either, and this is the more interesting half — it is not a rare
edge path but a *systematically avoided* one, because the verb does not work well for the thing it
exists to do. Counting how rarely a broken verb is called is measuring the defect, not the reach, so
the rubric's down-bump is refused on principle here. Medium stands unmodified.

## History
- `#1-fix-landed-on-one-of-two-siblings` `OPEN` reporter — Found during the look-dev polish pass
  over `PW_VegetationTest` (`Docs/map/vegetation-polish.md` § 5.6): a screen pick aimed at a tree
  canopy returned `hit:true` on the landscape ~86 m behind it. Re-derived at HEAD as a parity gap
  rather than a trace bug: `B-trace-complex-hits-render-geometry` (DONE, High) fixed the
  collisionless-mesh hazard in three parts — docs, accept-filters + `multiHit`, and
  `simpleCollisionShapes`/`renderGeometryHit` in the response — and all three landed only on
  `spatial.raycast` (`RaycastHandler.cpp:248-288`, fields at `:199` and `:203`), while
  `spatial.raycast_screen` (`RaycastScreenHandler.cpp:74-108`) still has a bare one-sentence
  `traceComplex` description at `:101-103`, no `multiHit`/`maxHits`, no
  `onlyActors`/`actorFilter`/`onlyClasses`, and a response (`:268-322`) carrying neither diagnostic
  field. Both verbs bottom out in the same `SpatialTraceUtils::TraceLine`
  (`RaycastScreenHandler.cpp:263-264`, `SpatialTraceUtils.cpp:306` with `bReturnFaceIndex` at
  `:303`) and the diagnostics live on the shared `FSpatialHit` (`SpatialTraceUtils.h:47-60`), so the
  screen verb computes them and drops them. HONEST GAP IN THE MEASUREMENT, recorded: the source note
  does not say whether `traceComplex` was set, and the default is `false`
  (`RaycastScreenHandler.cpp:200-201`), so the likeliest reading is an ordinary simple-collision
  trace passing through collisionless vegetation — which sharpens rather than weakens the ticket,
  because the response then has no field distinguishing "nothing was on that ray" from "the thing
  on that ray has no simple collision", which is the exact distinction the parent added to the
  sibling. SECOND HALF: neither raycast verb can name a struck instance — `FSpatialHit`
  (`SpatialTraceUtils.h:30-60`) carries `HitComponent` and no index, the module's only
  `InstanceIndex` is on `FSpatialOccupant` (`:269`) for the box-overlap path — although the plugin
  already resolves one at `SpatialTraceUtils.cpp:435` (`Overlap.GetItemIndex()`) with a
  `GetInstancesOverlappingBox` fallback at `:480`, and `spatial.find_clear_placement` reports
  `instanceIndices[]` (`MeasureHandler.cpp:1227`) beside `GetInstanceCount()` (`:1231`). For a
  level built the plugin's own documented way, `component:"HISM_ZF_Oak"` is the whole answer to a
  question that was about one of fifty instances. WORKAROUND (undocumented, and what holds severity
  down): `raycast_screen` echoes the deprojected `ray.origin`/`ray.direction` at `:273-275`, so
  feeding them to the already-fixed `spatial.raycast` with `traceComplex:true, multiHit:true` gets
  the full diagnostic surface in a second call. Rated **Medium**; **not High**, with the
  counter-argument stated in the body so a reviewer can take it — nothing in the response is untrue,
  and a screen pick's wrong answer is read once by someone looking at the image, where the parent
  ticket's wrong answer became a heightmap and propagated. Reach declined both ways, and the
  down-bump refused on principle: this verb's low call count is a consequence of the defect, so
  counting it would be measuring the defect rather than the reach.
- `#2-second-pass-did-not-attempt-the-verb` `OPEN` reporter — Second independent encounter on the
  same map, `encounters` 1 -> 2. The zone F re-speciation pass over `PW_VegetationTest`
  (`Docs/map/vegetation-zone-f.md` § Re-speciation, Findings 2) met the identical question — *which
  species is the dark mass in this frame* — and **did not call `spatial.raycast_screen` at all**. It
  went straight to a bespoke `python.execute` script, `dev/zonef2/f_whatis.py`, which projects every
  ISM/HISM instance into the camera frustum analytically. The species defect still took four
  captures to pin down. STATED PLAINLY SO NOBODY OVER-READS IT: this encounter adds **no new
  call-level measurement**, so it does not close `#1`'s recorded honest gap about whether
  `traceComplex` was set on the original 86 m pick. What it is evidence for is `#1`'s severity
  reasoning, which predicted exactly this: the reach down-bump was refused there on the ground that
  *"this verb's low call count is a consequence of the defect"*, and here is an independent pass, on
  the same content, choosing to write a projector rather than call the verb. A verb avoided on sight
  by the second team to need it is not a rare edge path. Also confirms the ticket's SECOND HALF
  independently — the pass needed a **named instance** out of 5888 in one holder, which is the
  question `component: "HISM_ZF_Oak"` cannot answer, and it is why `SpatialTraceUtils.cpp:435` /
  `:480` are the load-bearing citations for a fixer rather than the diagnostic fields. Severity
  unchanged at Medium: the workaround in the body (feed the echoed `ray.origin`/`ray.direction` at
  `RaycastScreenHandler.cpp:273-275` to the already-fixed `spatial.raycast`) is untouched by this
  encounter, and `encounters` is a same-severity work-ordering tiebreak, never a severity input.
  THE DISTINCT RESIDUE FROM THIS PASS WAS NOT FILED HERE: the ask that came out of it is an
  enumerating manifest on `render.capture_open_level` — a different verb, the opposite question
  (*what is in this frame* rather than *what is under this pixel*), and an analytic projection with
  no trace at all, which matters because a trace cannot see the collisionless vegetation both
  encounters were found on. Filed separately as `F-capture-drawn-primitive-manifest` (OPEN, Medium)
  with the dedup argued in its body; neither ticket closes the other, and they share
  `SpatialTraceUtils.cpp:435` / `:480`, so a fixer should take them together.
