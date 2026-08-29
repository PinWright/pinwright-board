---
id: B-ortho-capture-culls-distant-foliage
title: "Wide orthographic captures distance-cull their own foliage, so a whole-map top-down renders an empty forest"
status: IN-REVIEW
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

- `#6-additional-ortho-origin-audit` `OPEN` reporter — Additional evidence: **Adversarial review A — residual auto-scale defect.** Actuality: PARTIAL. The landed scoped cvar/sink/restore path is current, but auto-derivation still measures from the requested camera and surveys every visible primitive. Framing: the wide-lit-orthographic symptom remains accurate, but the title should name the residual mechanism (effective editor-ortho culling origin is not included); High remains justified as silent false-success. Proposed fix: INCOMPLETE, the ratio ignores UE's near-plane origin shift and the `MinDrawDistance` side effect, and has no direct HISM proof. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\PreviewViewportCaptureUtils.cpp:223-305,592-607`; `C:\UE_5.8\Engine\Source\Editor\UnrealEd\Private\EditorViewportClient.cpp:1333-1448`; `C:\UE_5.8\Engine\Source\Runtime\Engine\Private\SceneView.cpp:577-614,839-845`; `C:\UE_5.8\Engine\Source\Runtime\Renderer\Private\SceneVisibility.cpp:867-1008`; `C:\UE_5.8\Engine\Source\Runtime\Engine\Private\HierarchicalInstancedStaticMesh.cpp:1656-1689`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\EditorOps\TestCaptureViewDistanceScale.cpp:68-130,182-257`. Runtime: NOT VERIFIED in this review; prior #4 evidence covers ordinary cubes, not foliage/HISM. Recommendation: REFRAME; keep OPEN, derive against the actual `FSceneView` culling origin and frame-reachable bounds, then add HISM and nonzero-near-cull regression captures. Keep `F-ortho-tile-reference-compare` separate as the workaround/feature.
- `#7-additional-hism-origin-and-min-cull` `OPEN` reporter — Additional evidence: **Adversarial review B — A's residual auto-scale finding survives, but the title/proof boundary is narrower than stated.** Actuality: PARTIAL. Framing: I agree the cvar/sink/restore path is current and the UE 5.8 lit-editor ortho path moves `ViewOrigin` by the near plane before `CullingOrigin` is assigned; however, #4's recovery signal is ordinary cubes only, while current HISM code uses its temporal LOD origin and `EndCullDistance`. `SurveyViewDistances` still measures from the requested camera, takes only `EndCullDistance`, and does not account for separate minimum-distance terms; current tests assert a synthetic inequality and explicitly skip coverage when the 4096 cap binds, so they cannot disprove a visually valid-looking but empty forest. Proposed fix: INCOMPLETE, not a band-aid: scoped cvar restoration is systemic, but derivation/validation is not. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\PreviewViewportCaptureUtils.cpp:223-305`; `C:\UE_5.8\Engine\Source\Editor\UnrealEd\Private\EditorViewportClient.cpp:1403-1448`; `C:\UE_5.8\Engine\Source\Runtime\Engine\Private\SceneView.cpp:368-375,577-614,839-845`; `C:\UE_5.8\Engine\Source\Runtime\Renderer\Private\SceneVisibility.cpp:992-1008`; `C:\UE_5.8\Engine\Source\Runtime\Engine\Private\HierarchicalInstancedStaticMesh.cpp:1652-1691`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\EditorOps\TestCaptureViewDistanceScale.cpp:68-130,182-257`. Runtime: NOT VERIFIED in this review; the checkout has no host capture evidence, and #4 proves ordinary primitives only. Recommendation: REFRAME; keep OPEN, broaden the title to wide-ortho distance culling across primitives/HISM, derive against actual `FSceneView`/HISM culling origins and frame-reachable bounds, include min/end terms, and add HISM/foliage pixel recovery plus cap/near-cull failure tests. Keep `F-ortho-tile-reference-compare` separate as the tiling workaround/feature.
- `#8-hism-ceiling-and-pixel-recovery` `IN-REVIEW` developer — "**Reviews #6/#7 are stale on paths AND on substance.** Both cite `X:\src\unreal\unreal-fpv-dev\`; re-verified against `X:\src\unreal\unreal-fpv-new\`, where commit `afed5fb4` ('Derive an ortho capture's view distance from the point the renderer culls from', an ancestor of HEAD) already landed most of what they asked for. Present in this checkout before this pass: `MeasureCaptureCullingOrigin` builds the capture's own `FSceneView` and reads `View->CullingOrigin` (so the near-plane pushback is MEASURED, not a constant, and it correctly reads 0 for Unlit/Wireframe where `SceneView.cpp:579/619` gates the correction off); `RequiredScale` is per-primitive `reach/ownCullDistance` (the 7.6e12 giant no longer pegs anything); `MinDrawDistance` is surveyed and bounds the scale; the 4096 cap is retired for `MaxAutoViewDistanceScaleFor` = min(1048576, 1073741823/minCull), derived from the int32 truncation at `HierarchicalInstancedStaticMesh.cpp:1675`. So 'measures from the requested camera', 'ignores MinDrawDistance' and 'the 4096 cap' are no longer true here.

  **What was still wrong, and is now fixed.** (1) `foliage.MaxEndCullDistance` (`HierarchicalInstancedStaticMesh.cpp:90`, default 0) was unmodelled, and it is the one instanced term `r.ViewDistanceScale` cannot lift because the engine clamps AFTER the multiply (`:1675-1686`). Two silent failures followed: a HISM with NO end cull distance still culls at that ceiling, so the survey read an actively-culled forest as un-cullable and derived nothing; and a HISM past the ceiling drove `RequiredScale` toward a cap it could never cash in. `SurveyViewDistances` now reads the cvar once, substitutes/clamps the instanced cull distance with it, EXCLUDES past-ceiling components from `RequiredScale`, counts them in `NumFoliageCeilingLimited`, and the response carries `survey.foliageMaxEndCullDistance`, `survey.foliageCeilingLimited` and a `foliageCeilingWarning` naming the real remedy (raise/clear the cvar or narrow `orthoWidth` — NOT a bigger scale). With the cvar at its 0 default the survey is bit-identical to before. (2) The clamp disclosure was prose only: `viewDistanceWarning` existed but no machine-readable flag, so a caller had to pattern-match a string to learn the scale was clipped. `viewDistance.clamped` (unconditional) and `viewDistance.clampReason` (`nearCull`/`int32`/`absolute`) are now published, matching the `toneRangeApplicable`/`pinned` pairing convention already in this response.

  **Tests.** `capture_open_level.ReportsViewDistance` no longer STOPS ASSERTING when the cap binds — that was the named blind spot; it now branches on the published `clamped` flag and asserts the disclosure (reason present and one of three, warning non-empty, `coverageSufficient` false, effective < required) on the clamped side. Four new tests, all against real engine components rather than hand-built structs: `SurveyReadsInstancedEndCullDistance` (a real `UHierarchicalInstancedStaticMeshComponent`, surveyed twice with only `SetCullDistances` changing — proves the instanced end term drives the survey and that the requirement is ~110, i.e. measured from the culling origin, not ~1 from the camera; pins `foliage.MaxEndCullDistance` to 0 so the reading is hermetic); `SurveyReadsMinDrawDistance` (nonzero near-cull off a real component, plus the bound the derivation owes it); `FoliageCeilingIsNotScalable` (ceiling below reach -> counted unreachable and contributes NOTHING to `RequiredScale`; ceiling above reach -> nothing extra; end cull inside the ceiling -> requirement moves; ceiling back below -> contribution withdrawn); and `capture_open_level.HismFoliagePixelRecovery`, which renders FOUR frames and counts pixels — `diff(with scatter, without scatter)` at scale 1 versus the same diff at the derived scale. The differential is taken WITHIN a scale, never across one, so exposure, temporal passes and every other distance-culled thing in the level cancel and the count is attributable to the 3x3 instanced scatter alone; that is what #4's cross-scale evidence could not do (it needed a hand-corrected 2.4-2.65% noise floor and used ordinary cubes). It asserts the scatter draws >1% of the frame at the derived scale and <1/4 of that at scale 1. Its only early exits are host limitations (no editor world, no engine cube, no renderable level viewport) and one content limitation (the open map's own cull distances clamp the scale below full coverage) — each emits `PINWRIGHT_ASSERTIONS_SKIPPED` through `PinWrightTestSkip::SkipAssertions`, so such a run cannot be quoted as clean. `source != 'auto'` is a hard failure, not a skip. `check_test_ids.py`: CLEAN, 4748 ids, no dot-prefix collision. Docs: `Docs/wiki-src/render.md` response-shape section updated.

  **Not done, and why.** NOTHING WAS COMPILED OR RUN — the orchestrator owns the build this wave, so every claim above is a source-level claim and the pixel test in particular is unproven. One term stays unmodelled, now stated precisely instead of as review B's general worry: HISM reads `View->GetTemporalLODOrigin(0/1, bMultipleSections)` (`:1652-1653`), which returns `ViewMatrices.GetViewOrigin()` — the same value as `CullingOrigin` — EXCEPT when `bMultipleSections` is true (dithered LOD transitions, `:1520-1522`) and `FTemporalLODState::TemporalLODLag != 0` (`lod.TemporalLag`, default 0.5 s), where it returns a LAGGED SAMPLE of an earlier frame's view origin (`SceneManagement.cpp:108-137`). That is a settling lag, not a systematic offset, and a capture pumps frames at the final pose before reading back — but it is not proven to have settled, and a dithered-LOD foliage type is exactly the case that would expose it. Also unverified: the new ceiling modelling has no live-frame evidence because this project leaves `foliage.MaxEndCullDistance` at 0. `F-ortho-tile-reference-compare` deliberately untouched."
- `#9-ceiling-substitution-was-wrong` `IN-REVIEW` developer — "**One claim in `#8` was wrong and shipped a red test.** `PinWright.render.view_distance_scale.FoliageCeilingIsNotScalable` failed on `a component the ceiling already covers does not raise the required scale`. Cause: `SurveyViewDistances` SUBSTITUTED the ceiling as the cull distance of an instanced component that sets none of its own (`InstanceCull = FoliageMaxEndCullDistance` in the not-ceiling-limited branch). That branch is entered only when reach <= ceiling, so it manufactured a `reach/ceiling` ratio — always <= 1 and therefore never a scale the derivation could use — and additionally fed the ceiling into `MinCullDistance`, where it can clip the int32-truncation cap for a distance the scale never multiplies. `#8`'s stated rationale for the substitution ('the survey would otherwise read an actively-culled forest as un-cullable and derive nothing') is unachievable in either direction: the engine's product is `EndCullDistance * MaxDrawDistanceScale` (`HierarchicalInstancedStaticMesh.cpp:1675`), so for a component whose own `EndCullDistance` is 0 the product is 0 at EVERY scale and `EndCullDistance = MaxEndCullDistance` (`:1684`) outright — no `r.ViewDistanceScale` moves it. Such a component is either already inside the ceiling (drawn at every scale) or past it (drawn at none, which the exclusion already counts and warns about); it has no requirement to contribute.

  **Fixed** in `PreviewViewportCaptureUtils.cpp`: the else-branch now only CLAMPS an existing end cull distance (`FMath::Min(InstanceCull, ceiling)`, mirroring the engine's own `FMath::Min` at `:1680`) and no longer substitutes for a missing one. The ceiling-limited detection, the count, the response fields and the warning are unchanged. `foliage.MaxEndCullDistance == 0` handling was ALREADY correct — the whole block is gated on `> 0.0`, matching the engine's `if (MaxEndCullDistance > 0)` at `:1676` — so 0 means NO ceiling, not a ceiling of zero, and the survey is bit-identical to its pre-ceiling behaviour at the default.

  **The 0 default now has assertions of its own**, which is what `#8` left uncovered and is the state this project and this suite actually run in: the same test now takes a `ZeroBaseline`, then with the no-end-cull probe present asserts `FoliageMaxEndCullDistance == 0`, `NumFoliageCeilingLimited == 0`, that the probe is still surveyed as an instanced component, and that it adds neither an at-risk primitive nor a requirement. The inversion (0 read as a ceiling of zero) would put every instanced component past the ceiling and silently disable the whole derivation for foliage, while every pre-existing assertion — all taken at non-zero ceilings — still passed. Wrong comments corrected in `PreviewViewportCaptureUtils.h` (the `FoliageMaxEndCullDistance` field doc), the test's header comment, and `docs/wiki-src/render.md`. NOT COMPILED OR RUN — the orchestrator owns the build."
- `#10-ceiling-is-hism-only` `IN-REVIEW` developer — "**Second modelling error in the same block, found while gathering the engine evidence for `#9`: the ceiling was applied to every `UInstancedStaticMeshComponent`, but it is HISM-ONLY.** Both reads of `foliage.MaxEndCullDistance` sit inside `FHierarchicalStaticMeshSceneProxy` (`HierarchicalInstancedStaticMesh.cpp:1658` in `::GetDynamicMeshElements`, and `:1870`), and only `UHierarchicalInstancedStaticMeshComponent::CreateStaticMeshSceneProxy` builds that proxy (`:3004`); a plain `UInstancedStaticMeshComponent` gets `FInstancedStaticMeshSceneProxy` (`InstancedStaticMesh.cpp:2600`) and is never clamped — a grep of the whole engine source finds the cvar in that one file only. Consequence of the over-application, and it points the wrong way: a plain ISM reaching past a set ceiling was counted in `NumFoliageCeilingLimited` and EXCLUDED from `RequiredScale`, so the capture derived a scale too small for a component the scale genuinely recovers, and warned about a limit that component does not have — the same silent-missing-content failure this ticket exists to remove. Dormant at the 0 default, live for any project that sets the cvar.

  **Fixed** by gating the ceiling block on `Instanced->IsA<UHierarchicalInstancedStaticMeshComponent>()`. **Pinned** by a control probe in `FoliageCeilingIsNotScalable`: the shared scatter helper now takes a component class (`ViewDistanceTestSpawnScatterOfClass`, with the old HISM-typed `ViewDistanceTestSpawnScatter` as a thin wrapper so existing call sites are unchanged), and the test spawns a plain ISM with the same geometry, reach and end cull distance while the ceiling that just excluded its HISM twin is still set, then asserts it is NOT ceiling-limited and DOES raise the requirement to >= 110. Both assertions fail if the ceiling is applied to plain ISMs.

  **Known unmodelled, deliberately:** a Nanite-rendered HISM takes `Nanite::FSceneProxy` (`:2996-2999`) and is likewise never ceiling-clamped, but deciding whether a component renders as Nanite needs a material audit the survey does not run, and guessing wrong the other way would under-report a real limit. Recorded in the `NumFoliageCeilingLimited` field doc rather than modelled. NOT COMPILED OR RUN."
