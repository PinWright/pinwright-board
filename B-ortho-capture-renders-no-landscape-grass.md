---
id: B-ortho-capture-renders-no-landscape-grass
title: "An orthographic capture renders zero landscape grass at a frame the perspective capture fills with it, and distance culling is measurably not the cause — cullingOriginPushback 0, derived r.ViewDistanceScale 2.29, unlit, still no grass"
status: OPEN
severity: High
category: bug
tags: [render, capture_open_level, capture_ortho_tiles, orthographic, landscape, grass, vegetation, unattributed-mechanism, silent-wrong-data, verification-evidence]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# Ortho renders the terrain and none of the grass on it

At matched world coverage, an orthographic capture of a grassed landscape contains **no grass at
all**, while the perspective capture of the same ground contains a full carpet. The terrain,
lighting and everything else render; only the landscape grass is missing, and it is missing
entirely rather than thinning toward the frame edges.

## Measured, matched coverage

`orthoWidth 6928` was chosen to match the world span of `fov 60` at Z 6000, so the two frames cover
the same ground:

| projection | `meanLuminance` | grass in frame |
|---|---|---|
| perspective, fov 60, Z 6000 | 0.4797 | full carpet |
| orthographic, `orthoWidth 6928`, same ground | 0.5161 | **none** |

Direction, stated because the arithmetic misleads: grass darkens the frame, so the *higher* number
is the one with *less* grass. The ortho frame is brighter because the ground under the missing
grass is brighter.

## Distance culling is ruled out by measurement, not by argument

`B-ortho-capture-culls-distant-foliage` (OPEN, High, `encounters: 4`) is the board's wide-ortho
distance-culling defect, and it is the first thing a reader will reach for. It does not explain
this frame:

- With `viewMode: "unlit"` the capture's own survey reported **`cullingOriginPushback: 0`** — the
  field the culling fix added precisely to expose the editor-ortho view-origin pushback that ticket
  `#4` identified as the residual defect (`Handlers/Render/PreviewViewportCaptureUtils.cpp:1983`
  computes it, `:2341` publishes it as `survey.cullingOriginPushback`; declared at
  `PreviewViewportCaptureUtils.h:589`). Zero pushback means the ~2.1e6 uu offset that ticket blames
  is not present in this frame.
- The auto-derivation ran and applied a **derived `r.ViewDistanceScale` of 2.29**, so the
  view-distance override was live, not skipped.
- The grass was still completely absent.

- The framing is also the wrong shape for that ticket. `B-ortho-capture-culls-distant-foliage` is
  about a *wide* frame (its evidence is `orthoWidth 64000` from Z 60000, corners ~74000 cm out)
  losing distant instances while nearer ones survive — a partial, distance-graded loss. This frame
  is `orthoWidth 6928`; its corners are a few thousand cm from centre, and the loss is total.

## The mechanism was not attributed

**The zone agent could not attribute the mechanism.** No cause is proposed here. What is
established is the measurement above and the ruling-out above; anything past that would be a guess,
and a confident wrong cause in a ticket is worse than an honest gap.

Specifically **not** claimed, and left for whoever picks this up:

- whether the grass instances exist and are not drawn, or were never built for this view;
- whether an orthographic editor view contributes a usable entry to
  `World->ViewLocationsRenderedLastFrame` at all;
- whether any show flag or projection-dependent path in the grass renderer is involved.

A sibling ticket filed the same session, `B-capture-open-level-pose-params-photograph-stale-grass`,
does attribute a camera-keyed staleness mechanism for the *perspective* path. **That attribution is
cross-referenced only and is explicitly not offered as this ticket's cause** — it was not tested
against an orthographic view, and this frame's grass is absent even at the pose the persistent
camera is already sitting at.

## Consequence

`render.capture_ortho_tiles` (`Handlers/Render/OrthoTileCaptureHandler.cpp:162`) cannot verify
grass at all. That removes the only tiling path for large-area vegetation review, and it removes it
*silently*: the tiles come back valid, non-blank and settled, showing ground that in the perspective
capture is covered in grass.

It also collides with the board's own prescription. `B-ortho-capture-culls-distant-foliage` `#5`
records tiled orthographic capture as the cause-level workaround for wide-frame foliage loss —
"capture per tile rather than whole-map". For landscape grass specifically, the prescribed
workaround produces an empty frame, so the two tickets currently point a caller in a circle.

**Workaround:** perspective captures only for any question about landscape grass. There is no
orthographic path.

## Fix

Unattributed, so no implementation is proposed. The first useful step is diagnostic and is worth
recording as the ask: from an orthographic capture, report whether landscape grass components
exist in the scene and how many of their instances are in the frustum, so the next investigator can
tell "not built" from "built and not drawn" without guessing. The capture path already surveys
primitives for the view-distance work (`PreviewViewportCaptureUtils.cpp:1983` and the survey block
published at `:2341`), so the walk is already there to extend.

## Same shape as

- `B-ortho-capture-culls-distant-foliage` (OPEN, High) — **the ticket this one rules out**, not a
  duplicate of it. See the section above for the three measurements that separate them. Filed
  separately rather than as a fifth encounter because appending it there would attach evidence that
  contradicts that ticket's mechanism to that ticket's mechanism, and would let the culling fix
  close a defect it does not touch.
- `B-horizontal-orthographic-views-render-no-geometry` (DONE) — horizontal (`elevation 0`)
  orthographic frames that rendered overlays and no scene at all. Distinct: that was every
  primitive missing on the four side views, fixed; here the top-down view renders the whole scene
  correctly *except* landscape grass.
- `B-capture-open-level-pose-params-photograph-stale-grass` (OPEN) — grass absent from a capture,
  perspective, mechanism attributed. Cross-reference only.

## Severity

**High**, by impact class: *silent wrong data on a normal path*. The ortho frame is not a refusal
and not a blank — it is a plausible, well-formed picture of ground that is actually covered in
grass, and the response has no field that hints otherwise. A reviewer reading it concludes the
vegetation is missing.

**Reach modifier declined.** Orthographic capture is not an every-session path, which by the rubric
would bump this down to Medium. Declined on two grounds: it is the board's own prescribed workaround
for the sibling wide-ortho foliage defect, so it sits on a recommended path rather than an edge
one; and the failure is silent rather than an error, which is the property the High band is keyed
on.

Not rated Critical: nothing is written, nothing is lost, and the editor stays up.

## History
- `#1-ortho-renders-zero-grass-culling-ruled-out` `OPEN` reporter — Measured live against a running
  editor at matched world coverage (`orthoWidth 6928` against `fov 60` at Z 6000): perspective
  `meanLuminance` 0.4797 with a full grass carpet, orthographic 0.5161 with none. Distance culling
  ruled out by measurement rather than argument: with `viewMode: "unlit"` the response reported
  `cullingOriginPushback: 0` and a derived `r.ViewDistanceScale` of 2.29, and the grass was still
  absent; the frame is also two orders of magnitude narrower than the wide-frame case
  `B-ortho-capture-culls-distant-foliage` documents, and the loss is total rather than
  distance-graded. Field citations re-derived: `PreviewViewportCaptureUtils.cpp:1983` computes
  `CullingOriginPushback`, `:2341` publishes it, `PreviewViewportCaptureUtils.h:589` declares it;
  `OrthoTileCaptureHandler.cpp:162` registers `render.capture_ortho_tiles`. **Mechanism not
  attributed** — no cause proposed, three specific unknowns listed in the body.
