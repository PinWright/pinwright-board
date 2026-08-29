---
id: F-ortho-capture-depth-slab
title: "No near/far clip or depth-slab control on an orthographic capture, so an elevation or section view is not expressible — every horizontal ortho composites the entire world depth into one image"
status: OPEN
severity: Medium
category: feature
tags: [render, capture, orthographic, elevation, section, depth-slab, clip-plane, near-plane, far-plane, level-review, missing-verb]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# An ortho frame has a width and no depth

`render.capture_open_level` in `projectionMode: "orthographic"` takes `orthoWidth` — *"Orthographic
frame width in WORLD CENTIMETRES (the world span the image covers left to right)"*,
`Handlers/Render/RenderHandler.cpp:1522-1524` — and nothing that bounds the frame along the view
direction. The full parameter list is `RenderHandler.cpp:1513-1537`: `filename`, `width`, `height`,
`location`, `rotation`, `projectionMode`, `fov`, `allowBlank`, `orthoWidth`, `viewDistanceScale`,
`exposure`, `hideEditorSprites`, `viewMode`, `subject`. There is no near clip, no far clip, no slab
thickness, and no cutting plane.

For a top-down plan that does not matter. For anything shot horizontally it is the whole problem: an
elevation of one building composites every building behind it, and a section through one zone is
simply not a thing this verb can be asked for. The frame is a projection of the entire world along
the view axis, and no combination of the existing parameters narrows it.

**And it cannot be worked around by moving the camera.** An orthographic frame's world coverage
comes from `orthoWidth` and is independent of how far back the camera sits — stated in-source at
`Handlers/Render/PreviewViewportCaptureUtils.h:534-537`. Pulling back does not add depth clipping and
pushing in does not remove it.

## This is the inverse of `B-ortho-capture-culls-distant-foliage`, which is why it needs its own file

That ticket (OPEN, High) is that a wide ortho frame drops content it should show, and every remedy
in it makes *this* symptom worse by construction:

- its `#2` fix raises `r.ViewDistanceScale` so distance culling stops discarding far geometry — i.e.
  it deliberately removes the only depth-limiting effect an ortho frame currently has;
- its `#5` prescribes narrowing `orthoWidth` and tiling, which attacks the frame's **lateral**
  coverage and leaves depth untouched — a 16000-wide tile shot horizontally still composites the
  whole world behind it.

So the two asks pull in opposite directions along the same axis: one wants distant geometry back,
the other wants a bounded slab. A comment on that ticket would read as a request to undo its fix.
They must be worked as separate, independent changes — the same conclusion that ticket reached about
its own neighbour, `#5`, verbatim:

> **This ticket is not fixed and does not close on the feature landing.** ... Tiling gives callers a
> way around the defect; it does not correct the derivation. **Keep both open and independent.**

That precedent is established for this area and this ticket relies on it.

**Field evidence for why the depth problem is now visible.** A session this week ran an ortho
`render.capture_open_level` intended as a profile through one zone; the verb auto-derived
`r.ViewDistanceScale = 220` on a map with ordinary bounds and the resulting frame composited the
whole world depth. The auto-derivation is doing its job — the culling origin of a lit editor ortho
view sits ~2.1e6 cm behind the camera (`PreviewViewportCaptureUtils.h:538-552`), so defeating
distance culling needs a scale of that order. The consequence is that the accidental depth limit
callers used to get from distance culling is gone, and nothing replaced it. That observation belongs
on the bug ticket as an encounter; here it is the motivation.

## What it should do

A depth bound on the orthographic capture path, expressed the way the rest of the surface expresses
geometry — in world centimetres, and echoed back. Two shapes, either acceptable:

- `nearClip` / `farClip` as distances along the view direction from the capture camera, or
- `depthSlab: {near, far}` / a slab thickness centred on the `subject` when one is given, which
  composes with the existing `subject` block and lets a caller say "a 2000 cm slice through this
  actor" without computing camera-relative distances.

Whatever is chosen must be **echoed in the response**, like `orthoWidth` and the `viewDistance`
block already are, so a capture can be audited afterwards rather than trusted.

## Implementation notes, each a real trap

- **The plumbing already exists in the tree, on a different path.** `SceneCaptureProbeUtils` carries
  both terms — `NearClippingPlane` (`SceneCaptureProbeUtils.h:61`, default `DefaultNearClippingPlane
  = 10.0f` at `:48`) and `FarClippingPlane` (`:76`) — applied at
  `SceneCaptureProbeUtils.cpp:193-206` via `bOverride_CustomNearClippingPlane` /
  `CustomNearClippingPlane` and `MaxViewDistanceOverride`. Its only consumer is
  `render.detect_z_fighting` (`ZFightingHandler.cpp:171`, values set at `:399-409`). So the
  capability is proven in-plugin — but on a `USceneCaptureComponent2D`, **not** on the editor
  viewport that `capture_open_level` drives. This is a reference, not a lift.
- **The ortho near plane on the viewport path is load-bearing and must not simply be overwritten.**
  `PreviewViewportCaptureUtils.h:538-552` records the mechanism in full: `FEditorViewportClient` sets
  `OrthoNearClipPlane = -UE_OLD_WORLD_MAX` (`EditorViewportClient.cpp:1420`), and
  `FSceneViewProjectionData::UpdateOrthoPlanes` then applies `ViewOrigin += ViewForward * NearPlane`
  (`SceneView.cpp:609`), which is *why* the culling origin sits ~2.1e6 cm back. Changing the near
  plane to clip a slab therefore moves the culling origin, which changes what
  `ComputeAutoViewDistanceScale` must derive. The same comment notes the pushback is conditional —
  it needs `ViewFamily->ViewMode > VMI_Unlit` and `r.Ortho.AllowNearPlaneCorrection != 0` — so a
  Wireframe or Unlit slab capture behaves differently from a Lit one. Any implementation has to
  measure, not assume; the existing `MeasureCaptureCullingOrigin` /
  `bCullingOriginMeasured` pattern (`PreviewViewportCaptureUtils.cpp:659-690`, reported at `:2340`)
  is the model.
- **`r.ViewDistanceScale` is not a substitute and must not be repurposed as one.** It scales each
  primitive's own cull distance, so it can only ever remove content that was already being dropped;
  it cannot exclude content in front of or behind a plane, and it multiplies `MinDrawDistance` too
  (bounded at `PreviewViewportCaptureUtils.cpp:809-811`). A slab is a view property, not a
  per-primitive property.
- **Hold the capture-size rule.** Anything added here inherits `Docs/wiki-src/render.md:26-38` —
  capture resolution must not vary within a session. A slab parameter changes neither `width` nor
  `height`, so this is a constraint to preserve, not to solve.

## Distinct from

- `B-ortho-capture-culls-distant-foliage` (OPEN, High) — the inverse, argued in full above.
- `F-ortho-tile-reference-compare` (OPEN, High) — tiles the frame **laterally** into a grid over a
  world extent, for reference-image comparison. Orthogonal axis: a tile burst with no depth bound
  still composites full depth in every tile. Its `#5`-side reasoning is the precedent cited above.
- `B-capture-preview-ortho-drops-elevation` and `B-horizontal-orthographic-views-render-no-geometry`
  — both about a horizontal ortho frame coming back **wrong or empty** (the latter an alpha defect,
  since fixed). This is about a frame that renders correctly and shows too much. Verified neither
  asks for a clip control.
- `B-blockout-review-ortho-snap-contradiction` — pose snapping and rotation reporting, not depth.

## Dedup

Board-wide search across all statuses for `nearClip`, `farClip`, "depth slab", "clip plane",
"section cut" and "elevation view": **zero files match any of them.** The ortho family is
`B-ortho-capture-culls-distant-foliage`, `F-ortho-tile-reference-compare`,
`B-capture-preview-ortho-drops-elevation`, `B-horizontal-orthographic-views-render-no-geometry` and
`B-blockout-review-ortho-snap-contradiction`, each distinguished above. Nothing asks for depth
control on a capture.

## History
- `#1-no-depth-bound-on-ortho` `OPEN` reporter — Verified against `Handlers/Render/RenderHandler.cpp:1513-1537`: `render.capture_open_level` takes `orthoWidth` (`:1522-1524`, lateral world span only) and thirteen other parameters, none of which bounds the frame along the view direction — no near clip, no far clip, no slab, no cutting plane. A top-down plan is unaffected; a horizontal elevation or section is not expressible at all, because the frame projects the entire world depth and the camera cannot be moved to fix it (ortho coverage is independent of camera distance — `PreviewViewportCaptureUtils.h:534-537`). Filed as the **inverse** of `B-ortho-capture-culls-distant-foliage` (OPEN, High) and separate for that reason: every remedy in that ticket worsens this symptom — its `#2` raises `r.ViewDistanceScale` to stop distance culling discarding far geometry, deliberately removing the only depth limit an ortho frame has, and its `#5` narrows `orthoWidth` and tiles, which is lateral and leaves depth untouched. A comment on that ticket would read as a request to undo its fix. Precedent quoted and verified: that ticket's `#5` concludes "Tiling gives callers a way around the defect; it does not correct the derivation. **Keep both open and independent.**" Motivation from the field: an ortho capture this session auto-derived `r.ViewDistanceScale = 220` on a map with ordinary bounds and composited the whole world depth into a shot intended as a profile through one zone — the derivation behaving correctly, since the lit-ortho culling origin sits ~2.1e6 cm behind the camera (`PreviewViewportCaptureUtils.h:538-552`), and the accidental depth limit callers used to get from distance culling therefore gone with nothing replacing it. Ask: `nearClip`/`farClip` in world cm along the view direction, or `depthSlab: {near, far}` composable with the existing `subject` block; echoed in the response like `orthoWidth` and the `viewDistance` block already are. Implementation notes recorded with citations: the near/far plumbing exists in-plugin but on the `USceneCaptureComponent2D` path only (`SceneCaptureProbeUtils.h:48,61,76`, applied `.cpp:193-206`, sole consumer `render.detect_z_fighting` at `ZFightingHandler.cpp:171,399-409`) and is a reference rather than a lift; the viewport path's ortho near plane is load-bearing, since `OrthoNearClipPlane = -UE_OLD_WORLD_MAX` is what pushes the culling origin back (`EditorViewportClient.cpp:1420` -> `SceneView.cpp:609`), conditional on `ViewMode > VMI_Unlit` and `r.Ortho.AllowNearPlaneCorrection`, so a slab implementation must measure the resulting origin the way `MeasureCaptureCullingOrigin` already does (`PreviewViewportCaptureUtils.cpp:659-690`, reported at `:2340`); and `r.ViewDistanceScale` is not a substitute, being per-primitive rather than a view property. Dedup: board-wide search for `nearClip`, `farClip`, "depth slab", "clip plane", "section cut", "elevation view" returns **zero** matches; the five existing ortho tickets are each distinguished in the body. Severity Medium: this is the rubric's "hard blocker with no workaround — a missing verb, so a reasonable task is impossible" band, which is High or Medium, and Medium is the right end. Declining the bump to High deliberately: unlike `F-ortho-tile-reference-compare`, which is rated High because it is *also* the documented remedy for an open High bug and ranking it lower would deadlock both, nothing depends on this one, and the plugin's main-line ortho use — top-down plan capture — is unaffected. Not bumped down either: an elevation view is a normal level-review request, not a rare edge path.
