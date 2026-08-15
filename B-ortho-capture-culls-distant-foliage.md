---
id: B-ortho-capture-culls-distant-foliage
title: "Wide orthographic captures distance-cull their own foliage, so a whole-map top-down renders an empty forest"
status: IN-REVIEW
severity: High
category: bug
tags: [render, capture, orthographic, foliage, culling, view-distance]
encounters: 3
lastSeen: 2026-08-15T00:00:00Z
---

# A wide ortho frame culls the content it was taken to show

`render.capture_open_level` with `projectionMode:"orthographic"` and a large `orthoWidth`
renders a frame whose distant foliage is missing. The host project's whole-map top-down
(`Saved/MapVerification/current/ortho_topdown_plan.png`) shows the Radiant half with
essentially **zero tree instances**, while a perspective capture from the same batch three
minutes earlier shows dense tree mats in the same region.

This is not a blank-capture defect and not the alpha defect fixed in
`B-horizontal-orthographic-views-render-no-geometry`: the image is fully valid
(58489 distinct RGB values, 99.3% of rows carry detail per the host project's
`Docs/map/data/ortho_scrutiny.json`). The terrain, cliffs and structures render. Only the
distance-culled geometry is gone.

## Why it happens

An orthographic frame's world coverage comes from `orthoWidth` and does **not** shrink as the
camera pulls back. Distance culling stays radial from the single camera point. A 64000 cm
top-down shot from 60000 cm up therefore has frame corners ~74000 cm from the camera while its
centre is ~60000 cm, and every primitive with a finite cull distance vanishes over most of the
frame. UE 5.8 mechanism, both halves scaled by the same value:

- foliage / instanced meshes — `Runtime/Engine/Private/HierarchicalInstancedStaticMesh.cpp`
  computes `EndCullDistance = UserData_AllInstances.EndCullDistance * MaxDrawDistanceScale`
  and folds it into `FinalCull`, which `CalcLOD` then uses to drop whole cluster nodes;
- ordinary primitives — `Runtime/Renderer/Private/SceneVisibility.cpp` `FrustumCull` scales
  `Bounds.MaxCullDistance` by the same `MaxDrawDistanceScale`.

`MaxDrawDistanceScale` is `GetCachedScalabilityCVars().ViewDistanceScale`, i.e.
`r.ViewDistanceScale`. The capture path never touched it.

**The screen-size cull was the wrong first hypothesis.** It is already disabled for
orthographic views (`MinSize = bIsOrtho ? 0.0f : CVarFoliageMinimumScreenSize`,
`HierarchicalInstancedStaticMesh.cpp:1656`), so foliage with **no** cull distance set was never
affected. Only a finite `EndCullDistance` culls, which is why the defect looks partial.

## Impact

Forest paths and clearings are *made of* the instanced geometry being dropped, so an
orthographic plan capture cannot answer any question about them. In the host project this
forced every top-down measurement onto perspective renders, which splay side faces near the
frame edges and inflate edge-adjacent measurements.

## History

- `#1-filed-with-mechanism` **OPEN** — Reporter. Filed from the host project's
  `Docs/map/REVIEW_PROTOCOL.md` note ("ortho currently culls foliage at distance — the
  measuring mode is the unreliable one"), which recorded the symptom and explicitly asked for a
  board entry to be filed before ortho was trusted for a measurement. Mechanism identified
  against UE 5.8 source; the culling attribution is no longer a hypothesis.
- `#2-view-distance-override` **IN-REVIEW** — Developer. Added a capture-scoped
  `r.ViewDistanceScale` override that is restored on every exit path.
  `Handlers/Render/PreviewViewportCaptureUtils.{h,cpp}`: new `viewDistanceScale` parameter
  (must be `> 0`; `0` is rejected because it is also a legal cvar value meaning "cull
  everything"); `SurveyViewDistances` walks the world's primitive components once for the
  smallest finite cull distance (`CachedMaxDrawDistance` and, for instanced components,
  `GetCullDistances`' end value) and the farthest reach from the camera;
  `ComputeAutoViewDistanceScale` returns the ratio so
  `minCullDistance * scale >= maxPrimitiveDistance`, capped at 4096; `FScopedViewDistanceScale`
  applies and restores it. Two engine details are load-bearing and are commented at the call
  site: the renderer reads a **cache** refreshed only by a console-variable sink, so
  `CallAllConsoleVariableSinks()` is called explicitly after each write or the override is a
  no-op that reports success; and both writes go in at the priority the variable already
  carries, so the restore does not leave it pinned at `ECVF_SetByCode`.
  Auto-derivation is enabled for **orthographic** `render.capture_open_level` and
  `render.capture_annotated` only — a perspective frame's coverage and its cull distances scale
  together so it does not have this failure, and auto-enabling it there would move every
  perspective measurement already recorded against these verbs. Both verbs now report an
  unconditional `viewDistance` block whose `restored` field is a **read-back**, not an
  assertion. Tests in `Tests/EditorOps/TestCaptureViewDistanceScale.cpp` assert the coverage
  inequality directly (including that it *fails* un-scaled and *fails* when the cap binds),
  that the cvar returns to its pre-call value even on hosts with no viewport, and that a
  perspective capture is left alone. **Not compiled** — written by an agent without a build;
  a separate integration pass owns the build and the live verification.

- `#3-integration-pass-9` — **Built, tested and pushed.** Compiled clean on UE 5.8 first
  attempt (no fixes needed; the flagged `CVar->Set(float, EConsoleVariableFlags)` binding
  resolved). Full suite **3749 tests performed, 3747 pass, 2 fail** — both the pre-existing
  `localization.Validation.*`; the 7 tests in `TestCaptureViewDistanceScale.cpp` all pass.
  Shipped as `75abe316` "Keep distant foliage in wide orthographic captures".

  Live verification on `Dota2_Blockout`, new DLL loaded:
  - The `viewDistance` block is now present on every capture (its absence was the previous
    proof the old DLL was running).
  - Explicit `viewDistanceScale: 64` -> `scaleBefore 0.6`, `scaleApplied 64`,
    `overridden true`, `restored true`, `source "caller"`. The restore was confirmed
    **independently of the plugin's own read-back** by reading the cvar through Python
    afterwards: `r.ViewDistanceScale = 0.6000000238418579`. The equal-priority write does
    not leave the variable pinned — that risk is closed.
  - Auto-derivation correctly no-ops here: the survey reports `minCullDistance: 0`,
    `distanceCulledPrimitives: 0`, `primitives: 3204`, `instancedComponents: 10`, so
    `source: "none"`, `overridden: false`.

  **Still unverified — the culling recovery itself.** Because nothing in this map is
  distance-culled at this framing, scaling changed the frame only in the noise
  (mean luminance 0.36832330 un-scaled vs 0.36831305 at scale 64). The set/sink/restore
  mechanism is proven end to end; that `CallAllConsoleVariableSinks()` actually restores
  *culled foliage* is not, and needs a framing where `distanceCulledPrimitives > 0`.
  Leaving status IN-REVIEW for that reason.
