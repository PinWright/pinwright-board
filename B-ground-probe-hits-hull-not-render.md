---
id: B-ground-probe-hits-hull-not-render
title: "`spatial` ground probes resolve against SIMPLE collision by default, so on authored architecture they report a floor the render mesh does not have — a prop seated on a phantom hull surface floats 202 cm in the air and every reported number looks right"
status: OPEN
severity: High
category: bug
tags: [spatial, raycast, ground_actors, verify_grounding, trace, collision, simple-vs-complex, hull, silent-wrong-data, review-hazard, level-building, placement, face-index, provenance, non-uniform-scale]
encounters: 1
lastSeen: 2026-08-28T00:00:00+05:00
---

# A ground probe answers "where is the collision hull", and every caller reads it as "where is the stone"

`spatial.raycast` defaults to `traceComplex: false`, and `spatial.ground_actors` /
`spatial.verify_grounding` default their `surface.traceComplex` to `false` as well — deliberately,
because `B-trace-complex-hits-render-geometry` established that a complex probe is blocked by
collisionless foliage. So the default probe resolves against **simple/physics collision**. On a
landscape that is harmless: a heightfield is single-valued per column and the two representations
are the same surface. **On authored architecture they are different surfaces**, and nothing in the
response says which one answered.

## The repro, in two calls

`/Game/Maps/Atlantis`, UE 5.8, `EAContentExamples58`. Same ray, one flag apart:

```js
spatial.raycast {origin:{x:-2390,y:-900,z:20000}, direction:{x:0,y:0,z:-1},
                 traceComplex:false, multiHit:true, maxHits:8}
// -> hits[0] Temple_Podium  z = 210        simpleCollisionShapes: 3
//    hits[1] Seafloor       z = 0

spatial.raycast {origin:{x:-2390,y:-900,z:20000}, direction:{x:0,y:0,z:-1},
                 traceComplex:true,  multiHit:true, maxHits:8}
// -> hits[0] Seafloor       z = 0
//    Temple_Podium IS NOT IN THE LIST AT ALL.
```

The hull reports a podium floor at `z = 210`. There is no podium there. `SM_Temple_Podium`'s
`collision {}` block is three plain boxes (`SM_Temple_Podium.pwmodel:100-102`) while the render mesh
is three noised parts carrying **seven `subtract` chip tools** (`:45,49,53,63,67,81,85`), none of
which exist in the hull. Two of those tools cut clean through the bottom step, so the hull has a
solid tread over a hole.

A rubble chunk, `Rubble_Step_C4` (`SM_Rubble_Chunk_C`, StaticMeshActor_122), was seated on that
phantom tread at `z = 250.53`. Measured against the render surface:

```js
spatial.verify_grounding {surface:{preset:"any_solid", traceComplex:true},
                          filter:"Rubble_Step_", samples:4, maxPenetration:120, detail:"all"}
// Rubble_Step_C4 -> pass:false, ACTOR_NOT_GROUNDED,
//   "This actor touches no ground: its closest point floats 202.02 cm above the surface."
//   contactPoints: 0, coverage: 1, supportedColumns: 16
```

**202 cm of air, on the level's hero building, and no visual review found it in months.**

## Landscape is provably out of scope — this is an authored-geometry defect

Same ray, both flags, on open seafloor at `(-8000, 3000)`:

```
traceComplex:false -> z 182.80974622339272  normal (0.12926541875928269, -0.1900962040577687,
                      0.9732183129780361)  faceIndex 20  distance 19817.189453125
traceComplex:true  -> z 182.80974622339272  normal (0.12926541875928269, -0.1900962040577687,
                      0.9732183129780361)  faceIndex 20  distance 19817.189453125
```

Bit-identical, including `faceIndex`. A landscape probe cannot exhibit this bug; a probe over a
`.pwmodel`-authored mesh routinely can.

### Control: the hull is correct almost everywhere, which is exactly why this hid

At `(0, -1800)` — the same podium, an unchipped stretch of stylobate — both traces return
`Temple_Podium` at **`z = 600.000`, identical**, and both return the seafloor beneath it at `0`. The
hull is not a coarse approximation of this mesh; it agrees with the render surface everywhere the
mesh is unmodified, and diverges only over the `subtract` features. So a spot-check anywhere else on
the podium confirms the probe is trustworthy, and the one place it lies is the one place nobody
re-checked. **Sampling more points does not find this defect; comparing representations does.**

## Every divergence measured, with the probe XY for each

Same downward ray at each XY, `traceComplex` false then true, `multiHit:true` so an absent actor is
visible as an absent layer. All reproducible in one call pair on `/Game/Maps/Atlantis`.

| mesh | actor | probe XY | hull Z (`traceComplex:false`) | render Z (`traceComplex:true`) | divergence |
|---|---|---|---|---|---|
| `SM_Temple_Podium` | `Temple_Podium` | (-2390, -900) | `210.000` | **no podium layer** → `Seafloor 0` | **≥ +210** |
| `SM_Temple_Podium` | `Temple_Podium` | (1600, 2400) | `210.000` | **no podium layer** → `Seafloor 0` | **≥ +210** |
| `SM_Temple_Podium` | `Temple_Podium` | (-1750, 1750) | `600.000` | `404.480` | **+195.52** |
| `SM_Temple_Podium` | `Temple_Podium` | (-2000, 1500) | `600.000` | `402.466` | **+197.53** |
| `SM_Temple_Cella` | `Temple_Cella` | (0, 1350) | `2700.000` | `2560.024` | **+139.98** |
| `SM_Temple_Cella` | `Temple_Cella` | (700, 1350) | `2700.000` | `2576.215` | **+123.78** |
| `SM_Temple_Cella` | `Temple_Cella` | (1350, 700) | `2700.000` | `2585.909` | **+114.09** |
| `SM_Temple_Cella` | `Temple_Cella` | (0, -1350) | `2700.000` | `2600.542` | **+99.46** |
| `SM_Temple_Cella` | `Temple_Cella` | (0, 0) | **no cella layer** | **no cella layer** | hull hole; both agree |
| `SM_Wall_Ruin_A` | `BLD_Wall_03` | (8341.4, 10867.3) | `1754.400` | `1667.945` | **+86.46** |
| `SM_Wall_Ruin_A` | `BLD_Wall_03` | (9696.7, 10203.4) | `1754.400` | `1388.001` | **+366.40** |
| `SM_Wall_Ruin_A` | `BLD_Wall_03` | (7489.3, 11284.6) | `1754.400` | `1859.232` | **-104.83** |
| `SM_Dome_Ruin_A` | `BLD_Dome_A3` | (3327.0, 10408.8) | `1098.902` | `1224.744` | **-125.84** |
| `SM_Dome_Ruin_A` | `BLD_Dome_A3` | (1907.1, 9335.9) | `949.664` | `1346.232` | **-396.57** |
| `SM_Dome_Ruin_A` | `BLD_Dome_A1` | (5043.3, 7163.3) | `1446.400` | `1483.386` | **-36.99** |
| `SM_Dome_Ruin_A` | `BLD_Dome_A1` | (5457.46, 8618.09) | **no dome layer** → `Seafloor 260.015` | `1420.039` | hull absent under 1160 uu of render |

Four things this table says that a one-line summary would not:

**1. The sign is not constant, and not even constant within one mesh.** `SM_Wall_Ruin_A` measures
**+86.46** at one XY and **-104.83** at another, on the same actor. So there is no "take the higher
one" or "take the lower one" heuristic available to a fixer, and no safe default. The caller has to
be told the two numbers.

**2. A hull can be missing where render geometry exists, not only present where it is absent.** At
(5457.46, 8618.09) the hull returns no dome at all and the probe falls **1160 uu** to the seafloor
while the dome's cornice hangs overhead: the cornice sits at local r=1420, outside the hull
cylinder's r=1380. A ground probe there reports the seabed under a roof.

**3. Two of these are magnitudes a reviewer would never think to look for.** `SM_Temple_Cella`'s
hull is a flat **2700.000** across every wall crest because its seven collision boxes are squared
off above the ruined render top; the render crest is 99-140 cm lower and varies wall by wall. The
crumbled end of `BLD_Wall_03` diverges by **366 cm**.

**4. There is a plausible engine-level cause for the dome, worth its own investigation.** UE scales
an `FKSphereElem` radius by the **minimum** absolute scale component. `BLD_Dome_A3` is scaled
(1.1, 1.1, 1.0), so `min = 1.0` and the r=1300 hull sphere stays at 1300 while the render dome is
10% wider — the render pokes outside its own hull around the whole flank, which is why A3 reaches
-396.57 while `BLD_Dome_A1` (scale 1, 1, 1.04, uniform in XY) tops out at -36.99. **A non-uniformly
scaled actor with a sphere hull diverges by construction, on any project.** That generalises well
past this map and is the strongest argument that the probe must report the divergence rather than
expect callers to anticipate it.

### The podium's top-course patch, bounded

The hull is a flat `600.000` across the entire top step (checked at 8 separate XYs). Inside the
`step_top` chip prism at (-2080, 1750) the render surface drops to the `step_mid` top at
**402.5-406.6**, so the divergence runs **+193.4 to +197.5**. Measured edges, bracketed to 10-50 uu:
inboard **x -1645**, outboard it breaks through the arris at **x -2080**, **y 1530** to **y 2018**.
That is roughly **435 × 490 uu**, a square rotated ~38 deg, and it reaches further in -Y as x goes
more negative.

Five `Rubble_Top_*` props stand on that course, at (-1590, -600), (-1590, 500), (-600, -1590),
(1200, 1590) and (1590, -400), every one of them with pivot Z **648.8** — i.e. resting on the
`600.000` top face. **All five fall outside the patch**, so nothing is broken today. Nothing told
anyone the patch was there either, and the next prop placed on that course is a coin flip.

### Corrections to the figures this ticket was filed from

Recorded so the next reader does not re-derive them:

- **`SM_Temple_Cella` at (0, 0) is a hull hole, not a divergence.** The hull returns no cella hit, as
  reported — but the complex trace misses it too, because the roof-collapse tool removes the slab
  there. Both fall through to `Temple_Podium` at 600. It is still a real gap (none of the seven
  collision boxes covers the interior; a prop dropped inside the cella lands on the podium floor
  under it), but it is not an instance of the two surfaces disagreeing.
- **The -125.7 dome reading is on `BLD_Dome_A3`, not `BLD_Dome_A1`.** A1 was swept at 11 angles
  × 5 radii and its deepest negative is **-36.99**, which is the analytic ceiling for that
  instance. The two actors differ only in transform — see point 4 above.
- **`SM_Dome_Ruin_A` has three instances**: `BLD_Dome_A1`, `BLD_Dome_A3`, `BLD_Dome_C2`.
  `BLD_Dome_A2`, `BLD_Dome_B1` and `BLD_Dome_C1` are `SM_Dome_Ruin_B`.

## The ask: make the probe say which surface answered

This is the valuable half of the ticket, and it matters more than the seating fix.

**Preferred — a ground probe should re-probe with the opposite trace complexity and report the
divergence.** One extra line trace per column, and the caller learns the one fact the current
response cannot express: *the two collision representations of this surface disagree by N cm here.*
Surface it as a field (`hullVsRenderDeltaCm`, or a `divergence` block carrying both Z values and
which actor each came from) and, past a threshold, a `warnings[]` entry naming the actor and the
gap. `spatial.ground_actors` and `spatial.verify_grounding` should carry the same field per contact
column, because that is where a placement decision is actually made.

**Minimum — report which collision produced the hit.** `spatial.raycast` already publishes
`simpleCollisionShapes`, `faceIndex` and `renderGeometryHit` (all added by
`B-trace-complex-hits-render-geometry`), and they are close but they do not answer this question: on
the failing ray above the hull hit carries `simpleCollisionShapes: 3` and no `faceIndex`, and the
complex hit carries `faceIndex` — but a caller has to *know* that `faceIndex`-present means
render-triangle and `faceIndex`-absent means primitive, and no doc says so. Make it explicit:
`hitCollision: "simple" | "complex"` on every hit. A hull hit and a render-triangle hit must be
distinguishable in the response without inference.

### Why a warning and not just documentation: the numbers looked *right*

This defect survived because it produced clean, plausible, on-contract data.

- The probes land on **krepidoma course values exactly**: `210.000` at the breached XY and
  `600.000` on the stylobate, both to the last digit, and both are constants
  `Docs/map/atlantis-spec.md` publishes as the podium's own step heights. A reviewer checking a
  seat against the spec sees the spec's own number come back and stops reading.
- **Internal consistency is not evidence here.** Against the hull the seat is textbook, and it is
  textbook because the hull really *is* a flat stone tread — it just is not the stone. Every
  seating field agrees with every other seating field precisely because they all read the same
  wrong surface, so cross-checking them cannot break the tie. Compare
  `B-verify-grounding-maxgap-false-fail`, where a numerically flawless batch (`placed: 13,
  failed: 0`, `seatErrorCm: 1.1e-13`) still had to be re-judged by hand: there the numbers were
  right and the verdict wrong, and it was caught by a control. Here the numbers themselves come
  from the wrong surface, so no amount of re-reading them finds it — and the only control that
  works is a probe against the other representation.
- **`Rubble_Step_C4` was seated correctly — against the wrong surface.** Its pre-move underside
  measured `z = 202.02` (lowest sample; `undersideReliefCm 7.008`, so the highest underside sample
  was at `209.03`). Against the hull tread at `210.000` that is a bed of **7.98 cm** — inside the
  7.83–14.58 cm band its five surviving siblings `Rubble_Step_C1/C2/C3/C5/C6` occupy on the same
  course. The seat was not sloppy and was not an outlier; it executed the level's rubble recipe
  faithfully against a surface that is not there. **A defect that produces in-family numbers cannot
  be caught by looking at the numbers.**
- The probe's answer itself flips on the flag alone, and that flip is the whole bug: same origin,
  same direction, `traceComplex` false returns `Temple_Podium z 210`, true returns no podium at
  all. Both answers are available to the verb on every call; it publishes one and marks nothing as
  a choice.

A `pass` that inverts on an argument the caller was told to leave at its default is the failure mode
this plugin keeps paying for. The fix is not "trace complex"; it is **say what you measured**.

## The constraint: "always trace complex" is NOT the fix

Do not close this by flipping the default.

1. **`B-trace-complex-hits-render-geometry` (DONE) is the counter-example, on this same project.** A
   complex trace resolves against render triangles, so a StaticMesh with *no* simple collision still
   blocks: **268 of 930** downward ground probes returned the height of a collisionless HISM tree
   instead of the terrain. Flipping the default here re-opens that ticket.
2. **Some geometry on this map has no collision at all** — collisionless kelp and haze cards already
   blocked complex probes here. The working recipe is complex **plus peeling**
   (`onlyClasses` / `actorFilter` / `onlyActors`, or `multiHit` and pick), which is exactly what that
   ticket shipped. Any guidance this ticket produces must point at peeling, not at the bare flag.
3. **Complex traces are slower**, and `ground_actors` already fires `samples^2` columns per actor
   with a post-move re-measurement; doubling that unconditionally is a real cost on a batch seat.
4. **The divergence is not systematically signed, so there is no safe heuristic to substitute for
   reporting it.** The podium and cella hulls sit above their render surfaces; `SM_Dome_Ruin_A`'s
   sits below; and `SM_Wall_Ruin_A` measures **+86.46 at one XY and -104.83 at another on the same
   actor**. Neither "always take the higher hit" nor "always take the lower" is correct, and a
   fixer cannot pick one. The caller has to be handed both numbers and decide.

**Workaround until then:** when seating against anything authored (a `.pwmodel` mesh, not the
landscape), probe twice and compare before trusting either number, then seat with
`surface: {preset: "any_solid", traceComplex: true}` and **look at the result** — a `collisionSimple`
capture beside a `collisionComplex` one from the same pose shows the divergence directly, which is
how this was confirmed here.

## The published guidance points straight at this trap

`Saved/PinWright/wiki/spatial.md:46`, under *"traceComplex hits render geometry — read this before
writing a height probe"*, currently tells the caller:

> **Or drop `traceComplex` entirely.** It is `false` by default and simple collision is what you
> want for ground probing in almost every case. Reach for complex only when you genuinely need
> per-triangle precision on a surface whose simple hull is too coarse.

That advice is what put `Rubble_Step_C4` 202 cm in the air. It is not wrong about foliage — the
whole page is about foliage — but it generalises a foliage rule to "ground probing" as a category,
and the reader has no way to tell that the sentence stops being true the moment the ground is a
hand-authored mesh rather than terrain. "A surface whose simple hull is too coarse" reads as a
precision nicety; here the hull is not coarse, it is **solid where the stone is absent**, which is a
different failure with a different consequence.

Whatever the code fix turns out to be, that section needs the other half: *simple collision is right
for terrain and for meshes whose hull is their shape; for authored architecture with subtract/boolean
detail the hull is a different surface from the render mesh, and a probe against it will report a
floor that is not there.* The same page's own line — "Landscape has no simple/complex split and is
never flagged" — is the correct statement of where the safe case ends, and it is already there; it
just is not connected to the advice three lines above it.

The seating page repeats it more strongly. `Saved/PinWright/wiki/spatial.ground-placement.md:37`:

> `traceComplex` defaults `false` and **should stay there for these two verbs**; see the
> `traceComplex` section on the `spatial` page for why.

"These two verbs" are `spatial.ground_actors` and `spatial.verify_grounding` — the two that seated
and then cleared `Rubble_Step_C4`. A caller who follows the documentation exactly gets the wrong
surface, gets no signal that a second surface exists, and gets a `pass`. That is why this is filed
as a bug against the response shape rather than as a docs ticket: the docs are giving the best
advice available *given what the response can express*, and the response cannot express the thing
the caller needs to know.

Note the asymmetry worth preserving in any fix: both pages already teach the caller to distrust the
**far** side of the ray (something in front of the ground). Nothing teaches them to distrust the
**hit surface itself**.

## History
- `#1-initial-repro-and-detection-proposal` `OPEN` reporter — Filed from live measurement on
  `/Game/Maps/Atlantis` (UE 5.8, `EAContentExamples58`), 2026-08-28. **Dedup sweep done before
  filing.** `B-trace-complex-hits-render-geometry` (DONE) is the nearest neighbour and does **not**
  own this: it covers the opposite direction (a *complex* trace blocked by a collisionless mesh) and
  its fix — `multiHit` / `onlyActors` / `actorFilter` / `onlyClasses` peeling plus
  `simpleCollisionShapes` / `renderGeometryHit` / `warnings[]` — is about which actor answers,
  never about which *representation of one actor* answers. `B-verify-grounding-maxgap-false-fail`
  (DONE) is cited as a precedent for how a numerically flawless batch gets waved through, but its
  defect was the gate reading a silhouette; this one is the measurement reading the wrong surface,
  and its fix (float scoped to the actor's own lowest underside sample) does not touch it.
  `F-grounding-holder-not-seatable` (IN-REVIEW) is about ISM/HISM holders and is unrelated. Also
  checked and unrelated: `B-static-mesh-missing-collision-body-counts`,
  `E-simplify-collision-decimates-render-mesh`, `B-geometry-convert-static-mesh-drops-collision`,
  `F-spatial-no-clear-footprint-search`. Filed new. **Evidence:** 16 probe pairs across four meshes,
  each XY listed in the table; one prop (`Rubble_Step_C4`) measured floating **202.02 cm** and
  re-seated the same day; the landscape control at (-8000, 3000) returns bit-identical hits under
  both flags including `faceIndex`, and the unchipped-stylobate control at (0, -1800) returns
  `600.000` under both — so the divergence is localised to authored `subtract`/`noise_deform`
  detail and to non-uniformly scaled hulls, not global. Confirmed visually: a
  `render.capture_open_level` pair from one pose at 768×432, `viewMode: "collisionSimple"` beside
  `viewMode: "collisionComplex"`, shows the bottom step as an unbroken box course in the hull and
  bitten clean through in the complex view, with the prop hovering over the gap. **Severity High**
  by the board's impact class — silent wrong data on a normal path, where the caller trusts a
  result that is a lie and builds on it — with the reach modifier neutral: `spatial.raycast` and
  the two grounding verbs run in most level-building sessions, but the divergence only bites on
  authored architecture, not on terrain.
- `#2-ask-recast-to-surface-existing-provenance` `OPEN` reporter — 2026-08-28, source-verified at
  HEAD in this tree. **The ask in *"The ask: make the probe say which surface answered"* is the
  wrong shape and is recast here — read this entry before working those rungs.** Nothing in the
  body is retracted: the 16 probe pairs, both controls, and the *"always trace complex is NOT the
  fix"* constraint all stand. What changes is what to build.

  **1. `renderGeometryHit` cannot cover this case by construction — it is not "close".** It is
  exactly `bTraceComplex && SimpleCollisionShapes == 0`
  (`Handlers/Spatial/SpatialTraceUtils.cpp:43`), serialized only when true
  (`Handlers/Spatial/RaycastHandler.cpp:201-204`). `simpleCollisionShapes` is
  `UBodySetup->AggGeom.GetElementCount()` verbatim on the hit component, `-1` when there is no body
  setup — the Landscape case — and `-1` is omitted from the response rather than forged into a `0`
  (`Utils/CollisionSummaryUtils.h:66-90`, `RaycastHandler.cpp:191-200`). So it detects exactly one
  thing: *a complex trace hit render triangles because there was nothing else to hit.* Every mesh in
  the divergence table **has** simple collision, so `simpleCollisionShapes >= 1`, and at the
  documented default `traceComplex: false` the flag is **unreachable — it cannot fire on this
  defect at any argument the caller can pass.** **Record this as the evidence, because it is why the
  defect published with every number looking correct:** the probe that left `Rubble_Step_C4` 202 cm
  above the visible stone returned `simpleCollisionShapes: 1`, **no `renderGeometryHit`, and no
  `warnings[]`** — a completely clean, on-contract response. The warning loop at
  `RaycastHandler.cpp:516-535` `continue`s past every hit whose `bRenderGeometryHit` is false, so
  there was nothing for it to say.

  **2. `faceIndex` IS a per-hit detector for this, and it already works on a `traceComplex: false`
  probe.** It is **provenance**: present means a triangle mesh or heightfield answered, absent means
  a simple primitive did (`Handlers/Spatial/SpatialTraceUtils.h:39-43`). Unlike `renderGeometryHit`
  it is **independent of both `channel` and `traceComplex`** — `QueryParams.bReturnFaceIndex = true`
  is set in `TraceLine` (`SpatialTraceUtils.cpp:290`), under an in-code comment at `:286-289`
  stating it is "what lets the response tell a caller WHICH representation answered", and
  `TraceLineLayered` calls `TraceLine` once per layer (`:320`), so multiHit, filtered and
  ground-column probes all inherit it. Scope, stated accurately rather than as "every trace": the
  two exceptions are `:387` `PinWrightFootprintProbe`, an `OverlapMultiByChannel` where a face index
  has no meaning, and `GroundPlacementUtils.cpp:83`, the actor's **underside** probe via
  `LineTraceComponent` with its own params — not the ground probe, so not part of this gap. **This
  ticket's own repro already shows the signal working**: the hull hit carries no `faceIndex`, the
  complex hit does, and the landscape control at (-8000, 3000) returns `faceIndex 20` under *both*
  flags. **This corrects the body's framing** — detection does **not** require a second trace.

  **3. Absence is presumptive, not proof. A fixer must not build a hard assertion on it.**
  `SpatialTraceUtils.h:41-43` is explicit: `FaceIndex` is `INDEX_NONE` for a simple-primitive hit
  **or when the physics backend reported none**. Presence is a positive signal; absence is a strong
  prior and must be surfaced as one. Landscape returns a face index under either trace, which is a
  second and independent reason terrain is the easy case here.

  **4. The discard is three fields wide, not one — and closing it is the whole primary ask.** The
  ground column probe goes through `TraceLineLayered` (`GroundPlacementUtils.cpp:784-787`) and then
  reads only `Hit.Location.Z`, `Hit.Normal` and `Hit.HitActor` off the returned `FSpatialHit`
  (`:791-799`; commit `9e17fefc` cites the same block more narrowly as `:787-792`). **`FaceIndex`,
  `SimpleCollisionShapes` and `bRenderGeometryHit` are all computed on that same hit and all thrown
  away before serialization.** `faceIndex` reaches a response at `RaycastHandler.cpp:195` and
  **nowhere else in the plugin** — a repo-wide grep for the serialized key returns that one site
  plus a docs test. Both presets pin `bTraceComplex = false` (`GroundPlacementUtils.cpp:134`
  landscape, `:153` any_solid); only `custom` leaves it to the caller. So a batch seated onto
  structures is choosing blind while the one signal that would have caught it sits in a local
  variable. **Recast ask, replacing both rungs above:** `spatial.ground_actors` and
  `spatial.verify_grounding` should surface the per-column provenance their probe **already
  computes** — all three fields — on each contact column. No extra trace, nothing new measured, no
  new cost: a serialization change. The "Minimum" rung's `hitCollision: "simple" | "complex"` is a
  fair *presentation* of `faceIndex` and worth having on `spatial.raycast`, but it must carry
  `#3`'s caveat rather than assert a certainty the signal does not have.

  **5. The divergence comparison stays, as the richer follow-on rather than the entry price.**
  Provenance says *which* representation answered; it carries neither **magnitude nor sign**, and
  those are what a seating decision needs. Keep the "Preferred" rung — re-probe at the opposite
  trace complexity and report both Z values, which actor answered each, and the delta
  (`hullDivergenceCm` per contact column on the grounding verbs, an agreement/divergence readout on
  `spatial.raycast`), with a `warnings[]` entry past a threshold — but ship it **after** the three
  provenance fields, not instead of them.

  **6. No sampling heuristic can substitute for either, and the control proves it.** On unmodified
  stone on the same podium at (0, -1800) **both traces return an identical `600.000`**. The hull is
  not a coarse approximation that more sample points would eventually catch out: it **agrees
  everywhere the mesh is unmodified** and diverges only over `subtract`-cut or eroded features and
  non-uniformly scaled hulls. More samples cannot find that; comparing representations — or reading
  the provenance of the one trace you already took — can.

  **7. `simpleCollisionShapes` is a usable weak prior, explicitly not a solution.** A count of `1`
  standing in for an elaborately authored mesh is the signature shape of a divergent hull, and it is
  already in the response at zero cost — worth naming as **cheap triage** ("this hull is far simpler
  than this mesh; check the provenance"). It carries neither magnitude nor sign, so it is not the
  fix. Sign is the harder half: `SM_Wall_Ruin_A` measures **+86.46 at one XY and -104.83 at another
  on the same actor**, so no "take the higher hit" rule exists at any granularity — not even
  per-mesh, let alone globally.

  **8. The defect does not require an authoring mistake, which materially widens who is affected.**
  UE scales an `FKSphereElem` radius by the **minimum absolute scale component**, so an actor at
  (1.1, 1.1, 1.0) gets a hull sphere sized to its *unscaled* axis while the render surface is 10%
  wider — measured up to **-396 cm under the render surface** on `BLD_Dome_A3`. A correctly authored
  mesh with a correct hull, placed at a non-uniform scale, produces this defect with nothing done
  wrong by anyone. Any project that non-uniformly scales an actor whose hull contains a sphere is
  exposed, on any map.

  **The documentation half is CLOSED — reference it, do not re-ask for it.** Fixed and committed in
  the plugin repo as **`4fafde6b`** and its follow-up **`9e17fefc`**. `spatial.md` gained a
  `## Which trace, for which ground` section splitting the advice three ways (terrain / authored
  architecture / foliage-and-cards) and its *"simple collision is what you want ... in almost every
  case"* bullet — the exact sentence quoted in *"The published guidance points straight at this
  trap"* above — was rewritten; it now also separates the three fields by the question each answers
  and states plainly that `renderGeometryHit` cannot flag a hull.
  `spatial.ground-placement.md:35` no longer says `traceComplex` "should stay" `false` for the two
  grounding verbs, and says outright that neither verb surfaces any of the provenance.
  `editor.collision-review.md` now records that capturing both collision views from one pose is how
  you *see* a divergent hull. **The guidance gap this ticket raised is closed; only the tooling ask
  remains open**, which is why this stays `OPEN` against the response shape alone.

  **Severity stays `High` — argued, not silently held.** Impact class is unchanged and already at
  the board's ceiling short of `Critical`: silent wrong data on a normal path, where the caller
  trusts a result that is a lie and builds on it. `Critical` is defined by impact class alone —
  editor crash, or a write that corrupts or loses asset data — and neither applies: nothing crashes,
  no asset is corrupted, and the damage is a misplaced actor, recoverable by re-seating once you can
  see it. The `FKSphereElem` finding in `#8` is the strongest candidate for a bump and it lands on
  the **reach modifier**, where it widens *within* neutral rather than up a band: the widening is in
  the **population of affected geometry** (no authoring mistake required, any project), not in the
  **methods** — the same three verbs are affected as before, and terrain, which is most probes,
  stays provably immune per `#1`'s bit-identical landscape control. Reach neutral, severity `High`.

  **Frontmatter deliberately unchanged apart from tags.** `encounters` stays `1` and `lastSeen`
  stays `2026-08-28T00:00:00+05:00`: this entry is source analysis of the filed observation, not a
  second sighting, and bumping either would assert an encounter that did not happen (same reasoning
  as `B-verify-grounding-maxgap-false-fail` `#2`). Tags `face-index`, `provenance` and
  `non-uniform-scale` added so a future dedup sweep on any of the three lands here.
