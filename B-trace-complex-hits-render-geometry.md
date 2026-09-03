---
id: B-trace-complex-hits-render-geometry
title: "trace_complex=True hits render geometry of collisionless meshes, silently blocking ground traces"
status: DONE
severity: High
category: bug
tags: [spatial, raycast, trace, collision, silent-wrong-data]
encounters: 1
lastSeen: 2026-08-12T00:00:00Z
---

# trace_complex=True hits render geometry of collisionless meshes

With `trace_complex=True` a trace resolves against the **render** geometry of static
meshes, including meshes that have **no collision at all**. A downward ground-height
probe near foliage was therefore blocked by collisionless tree meshes: **268 of 930
traces** returned a hit on the wrong actor. There is no error and no warning — the
caller gets a plausible-looking hit location that is simply wrong, and any height map
built from it is silently corrupt.

Repro (2026-08-12, UE 5.8, `EAContentExamples58`): run downward ground traces over a
landscape populated with a HISM foliage actor whose source mesh has no collision, with
`trace_complex=True`. 268/930 traces hit the foliage rather than the landscape. The
traces only came back correct after the stale HISM actor was destroyed.

**Workaround:** any height-probe helper must verify the hit **actor** (e.g.
`hitActor == "DotaTerrain"`) rather than taking the first blocking hit; or issue a
multi-hit trace and filter. Neither is discoverable from the current docs.

**Corrected framing (developer, at fix time):** the *engine* is right. `bTraceComplex`
traces against COMPLEX collision, and for a StaticMesh with no simple collision
primitives the complex representation IS the render triangle soup — so asking for a
complex trace is asking to hit render geometry. Nothing in the trace call was wrong.
The real defect is **API ergonomics**: `spatial.raycast` returned only the first
blocking hit and gave the caller no way to express "I only care about the landscape"
or "ignore these actors", so every height-probe author had to rediscover the rule the
hard way and hand-roll filtering, and the naive call silently produced wrong heights.
That reframes the ticket from "fix the trace" to "make the correct call expressible,
and make the wrong one loud".

**Fix (implemented):** three parts.
1. Document the hazard on `spatial.raycast` and on the `spatial` namespace page: a mesh
   with no collision primitive **still blocks** a complex trace, this is expected engine
   behaviour, and here is how to filter it.
2. Give the verb the missing surface: `multiHit` (walk the layers along the ray) plus
   `onlyActors` / `actorFilter` / `onlyClasses` accept-filters, so "the ground under this
   point, whatever is standing on it" is one call.
3. Surface the condition in the response: `simpleCollisionShapes` (0 = the struck
   component has no simple collision at all), `renderGeometryHit: true`, and a top-level
   `warnings[]` entry naming the actor.

## History
- `#1-initial-repro` `OPEN` reporter — "trace_complex=True resolves against render geometry, so collisionless meshes block traces with no warning. 268/930 downward ground traces near a collisionless HISM foliage actor returned wrong hits; correct only after the actor was destroyed. Needs a documented warning on spatial.raycast plus ideally a no-collision flag in the response."
- `#2-reframed-as-api-ergonomics-and-implemented` `IN-REVIEW` developer — "Reframed: the engine behaviour is CORRECT (bTraceComplex resolves against complex collision, which for a collisionless StaticMesh is its render triangles); the defect is that spatial.raycast returned only the first blocking hit with no way to say which hits count. Implemented: (a) SpatialTraceUtils.h/.cpp gains FSpatialHitFilter (OnlyActors by pointer identity / ActorFilter delegating to the shared NameMatch::FFilter from Utils/NameMatchFilter.h, so spatial.raycast speaks actor.list's filter+matchMode+caseSensitive vocabulary verbatim instead of inventing a second policy — see B-actor-list-filter-case-mismatch / OnlyClasses matched case-insensitively up the class ancestry) and TraceLineLayered(), which peels the hit actor and re-traces because UWorld::LineTraceMultiByChannel stops at the first blocking hit and cannot see behind it; FSpatialHit gains FaceIndex, SimpleCollisionShapes (UBodySetup->AggGeom element count, -1 when no body setup) and bRenderGeometryHit. (b) RaycastHandler.cpp adds multiHit/maxHits/onlyActors/actorFilter/onlyClasses with snake_case aliases, echoes traceComplex, emits hits[]/count/truncated under multiHit, and reports filteredOut/filteredOutCount, unresolvedIgnoreActors, unresolvedOnlyActors and warnings[] so nothing is dropped silently; onlyActors resolving to nothing is a typed ACTOR_NOT_FOUND and maxHits<1 a typed INVALID_PARAMS instead of a quiet miss. The unparameterised call path is unchanged (one LineTraceSingleByChannel, same top-level fields; new fields are additive). (c) Docs/wiki-src/spatial.md gains a '## traceComplex hits render geometry' namespace section (states it is expected engine behaviour, carries the 268/930 evidence, lists the remedies) and a rewritten '### spatial.raycast' method section. Tests: Tests/Spatial/TestRaycastHandlers.cpp +4 (multiHit sees past a blocker, actorFilter/onlyActors/ignoreActors select the wanted hit and report filteredOut, legacy shape unchanged, typed filter errors); Tests/Infra/TestSpatialRaycastFilteringDocs.cpp +2 doc-regression tests. Build coupling to note: RaycastHandler.cpp and SpatialTraceUtils.h now include Utils/NameMatchFilter.h, which landed in the same unbuilt working tree from the concurrent actor.list fix; the two changes must build together. NOT VERIFIED AT RUNTIME: not compiled or executed by this agent (build owned by a separate integration agent), and the literal collisionless-mesh + traceComplex path is not reproducible from engine assets in an automation test — renderGeometryHit/warnings still need a live check against real foliage."
- `#3-runtime-verified-and-committed` `DONE` tester — Built and runtime-verified on
  `/Game/Maps/Dota2_Blockout`. **First real positive proof of `renderGeometryHit`** (automation could
  only assert it negatively): the probe mesh `SM_Tree_Dire_Claw` was confirmed via Python to have a
  BodySetup with **zero** simple primitives (0 boxes/spheres/convex/sphyl) and
  `CollisionEnabled.QUERY_AND_PHYSICS`. A downward `traceComplex:true` probe at its instance-0 XY
  `(18202.86, 13988.45)` from z=100000 returns `hit:true`, `actor.name:"FO_Trees_Dire_Claw"`,
  `simpleCollisionShapes: 0`, `renderGeometryHit: true`, `faceIndex: 810` and the `warnings[]` entry
  naming the actor and the remedy. **The fix:** the same trace with `onlyClasses:["LandscapeProxy"]`
  returns `DotaTerrain` with `filteredOut:["FO_Trees_Dire_Claw"]`, `filteredOutCount:1`,
  `truncated:false` and no warnings. The error magnitude is concrete: tree at `z=639.94` vs terrain at
  `z=-0.0008`. `actorFilter:"DotaTerrain"` + `matchMode:"exact"` + `caseSensitive:true` does the same
  and echoes all three. **multiHit:** `{multiHit:true, maxHits:8}` returns 3 layers with strictly
  increasing distance (99360 → 100000 → 100090) — foliage, landscape, then a cube with
  `simpleCollisionShapes:1`; top-level `location` equals `hits[0].location`. **Typed errors:**
  `onlyActors:["NoSuchActorXYZ"]` → `ACTOR_NOT_FOUND`; `maxHits:0` → `INVALID_PARAMS`;
  `matchMode` with no `actorFilter` → `INVALID_ARGUMENT`; `actorFilter:"["`+`regex` →
  `INVALID_PATTERN`; `ignoreActors:["NoSuchActorXYZ"]` still succeeds and now echoes
  `unresolvedIgnoreActors`. **No regression on the three shared-`TraceLine` callers** (which gained
  `bReturnFaceIndex`): `spatial.raycast_screen` deprojects and hits `DotaTerrain` with its `ray`/`view`
  echo intact; `spatial.place_on_surface {dropDown:true}` dropped a probe cube from z=5000 to
  z=49.999 resting on the terrain; `spatial.verify_placement {expect:{grounded:{maxGap:2}}}` →
  `pass:true`, gap ~1.7e-13. Probe actor deleted afterwards. Committed as `daf05ba3`, deliberately
  placed AFTER the `actor.list` commit so its `Utils/NameMatchFilter.h` dependency already exists and
  each commit builds standalone.
