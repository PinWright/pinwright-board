---
id: B-ortho-capture-culls-distant-foliage
title: "Wide orthographic captures distance-cull their own foliage, so a whole-map top-down renders an empty forest"
status: OPEN
severity: High
category: bug
tags: [render, capture, orthographic, foliage, culling, view-distance]
encounters: 4
lastSeen: 2026-08-16T00:00:00Z
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

- `#4-recovery-proven-derivation-under-scales` **OPEN** — Tester (integration pass 10, on the
  user's explicit instruction to verify). **Recovery confirmed. Returned to OPEN anyway, because
  the auto-derivation does not reliably produce a sufficient scale.**

  *Constructed case.* `#3`'s gap was that this map distance-culls nothing. Built one: 16
  throwaway `StaticMeshActor` cubes (`/Engine/BasicShapes/Cube`, scale 70) in a 4×4 grid at
  `x,y ∈ {-24000,-8000,8000,24000}`, `z 20000`, each given `LDMaxDrawDistance = 20000` on its
  `StaticMeshComponent0`. Survey then reported the gate met: `minCullDistance: 20000`,
  `distanceCulledPrimitives: 16`, `primitives` 3202 → 3218.

  *Write path worth recording.* `actor.set_component_properties` works because it calls
  `MarkRenderStateDirty()`, and `UPrimitiveComponent::CreateRenderState_Concurrent`
  (`PrimitiveComponent.cpp:620-627`) folds `LDMaxDrawDistance` into `CachedMaxDrawDistance` when
  the latter is 0. **`property.set` does NOT work here** — it calls bare `PostEditChange()` with a
  null `Property`, skipping the `bCullDistanceInvalidated` block in `PostEditChangeProperty`, and
  never dirties render state.

  *Result, top-down ortho `(0,0,60000)` `pitch -90` `orthoWidth 64000` 1024×1024, `Lit`:*
  **0/16 probes rendered at `viewDistanceScale: 1`; 16/16 at the auto-derived scale.** Pixel diff
  203,463 px changed (19.40%) against a predicted 16-cube footprint of 19.14%, with **90.1% of all
  changed pixels inside the 16 predicted boxes**, versus a measured noise floor of 2.40–2.65%
  between should-be-identical frames — signal ~8× noise. Mean luminance moved 3.98e-3 (0.37593 →
  0.37991), 400× `#3`'s 1e-5, but it is **confounded by eye adaptation** and the pixel diff is the
  metric to trust. Perspective control correctly untouched: `source:"none"`, `overridden:false`.
  So `CallAllConsoleVariableSinks()` really does restore culled geometry — `#3`'s open question is
  answered, affirmatively.

  *Why this is not DONE.* An explicit-scale sweep put the recovery threshold between effective cull
  distances 2.0e6 and 2.2e6 — **50× the probes' actual 40,000–50,675 cm from the camera**, and flat
  across probes at different distances. That is a constant offset K with K + ~40,000 ∈ (2.0e6,
  2.2e6], which brackets UE's `UE_OLD_WORLD_MAX` = 2,097,152: an editor orthographic view's origin
  sits ~2.1M uu *behind* the camera, and `FrustumCull` measures from
  `View.ViewMatrices.GetViewOrigin()`, not from the camera location.
  `ComputeAutoViewDistanceScale` derives `Required = maxPrimitiveDistance / minCullDistance` with
  `maxPrimitiveDistance` measured **from the camera location**, so it under-scales by that
  pushback.

  **It only passed here by accident.** This map's `maxPrimitiveDistance` is 7.6e12 (some primitive
  carries an enormous bounding sphere), which pins the derived scale to the 4096 cap → effective
  81,920,000, comfortably past the ~2.1e6 the ortho origin demands. On a map with sane bounds this
  same framing derives ≈ 75,000/20,000 = **3.75** → effective 75,000, and the sweep shows effective
  160,000 already recovers **nothing**. The verb would report `source:"auto"`, `overridden:true`,
  `restored:true` and no warning while recovering zero primitives — the exact false-success class
  `rpc-design.md` §1 exists to prevent. The 4096 cap is currently load-bearing by luck.

  *Suggested fix:* have `SurveyViewDistances` measure reach from the effective orthographic view
  origin (camera − viewDir × pushback), or add the pushback term in
  `ComputeAutoViewDistanceScale` for orthographic captures. Either way the derived scale should be
  validated against a recovery it can actually observe, not just published.

  *Also unverified, and worth closing next time round:*
  - **Instanced/foliage (HISM) recovery specifically.** The probes were ordinary
    `UStaticMeshComponent`s, so this proves the `SceneVisibility.cpp` `FrustumCull` path. The
    `HierarchicalInstancedStaticMesh.cpp` `FinalCull`/`CalcLOD` path — the ticket's headline
    foliage case — was not directly exercised.
  - **The 2,097,152 constant** is inferred from an empirical bracket, not read out of the ortho
    view-origin code.
  - **`r.ViewDistanceScale` also multiplies `MinDrawDistance`** (`SceneVisibility.cpp`:
    `MinDrawDistanceSq = FMath::Square(Bounds.MinDrawDistance * MaxDrawDistanceScale)`), so a 4096
    auto scale pushes *near*-culling out 4096× as well. Nothing on this map sets a non-zero
    `MinDrawDistance`, so whether that makes near geometry vanish is untested — a plausible
    regression on maps that use it.
  - Which primitive carries the 7.6e12 bounding sphere.

  *Housekeeping:* all 16 probes destroyed (`actor.find_by_name "PWCullProbe"` → `count: 0`, and a
  post-cleanup survey back to the exact pre-task `primitives: 3202` / `minCullDistance: 0` /
  `distanceCulledPrimitives: 0`). No existing map actor mutated, nothing saved, no editor started
  or killed, viewport camera byte-identical to the pre-task reading, and `r.ViewDistanceScale`
  restored to `0.6000000238418579` verified through Python independently of the plugin's own
  read-back. The map package is left dirty-but-unsaved by the spawn/destroy cycle — discard by
  reverting the level; do not save it.

- `#5-tiling-is-the-prescribed-fix` **OPEN** — Reporter. **Cross-reference only; status unchanged.**
  Filed `F-ortho-tile-reference-compare` (OPEN), whose `render.capture_ortho_tiles` verb is the fix
  this file's own source comment already prescribes:
  `PreviewViewportCaptureUtils.cpp:723-724` — *"the int32 bound in the foliage path clipped it"
  tells a caller to narrow orthoWidth and tile instead* — with a second hint at `:980`
  (*"orthoWidth and capture in tiles, or pass viewDistanceScale explicitly."*).

  Why tiling is the *cause-level* fix and `viewDistanceScale` is not. Per `#4`, the culling origin
  of a lit orthographic editor view sits ~2.1e6 cm behind the camera (`UE_OLD_WORLD_MAX`), so the
  distance every primitive is measured at is dominated by a constant that camera height cannot
  reduce, and the derived scale under-scales by exactly that pushback. Narrowing `orthoWidth` and
  tiling attacks the frame's world coverage — the one term the caller actually controls. Measured
  on the host map over the same world region at matched pixel scale: a whole-map ortho
  (`orthoWidth 64000`) retained **0.4%** of the Dire instanced foliage and **53.1%** of the Radiant
  canopy that a tile-scale ortho (`orthoWidth 16000`) kept, and the host project records "capture
  per tile rather than whole-map" as the workaround already in use
  (`Docs/map/reference_tile_compare.md` section 4). That project's 4x4 comparison tables exist only
  because the tiled frames retain foliage.

  **This ticket is not fixed and does not close on the feature landing.** The
  `ComputeAutoViewDistanceScale` under-scaling from `#4` is a live silent-false-success on any map
  with sane bounds (it reports `source:"auto"`, `overridden:true`, `restored:true` while recovering
  nothing — the 4096 cap is load-bearing by luck), and the HISM/foliage recovery path named in the
  title is still unexercised. Tiling gives callers a way around the defect; it does not correct the
  derivation. Keep both open and independent.
