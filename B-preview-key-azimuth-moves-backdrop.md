---
id: B-preview-key-azimuth-moves-backdrop
title: "render.capture_asset_preview: previewScene.key.azimuth silently changes the backdrop as well as the key light — 41% of the frame goes black while the response's previewScene block reports the same environment on both shots"
status: IN-REVIEW
severity: Medium
category: bug
tags: [render, capture_asset_preview, previewScene, key-light, azimuth, environment, cubemap, rotateLightingRig, litPixelFraction, meanLuminance, silent-wrong-data, comparability, weapons]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Two shots, one parameter changed, and the response says the thing that changed did not

`previewScene.key.azimuth` is the parameter a caller reaches for to answer "does this asset read
right from the other side?" The A/B is only meaningful if the key light is the only thing that
moved. It is not, and the response block that exists precisely to report the preview-scene state
does not say so.

## What was called

Two `render.capture_asset_preview` calls, **same asset, same camera, same pinned exposure**
(`ev100 −1.3979541`), differing in exactly one field:

```
previewScene: { key: { azimuth: 112.5 } }      →      previewScene: { key: { azimuth: 292.5 } }
```

(180° apart — the standard opposed-key A/B.)

## What happened — measured

| | azimuth 112.5 | azimuth 292.5 |
|---|---|---|
| `imageStats.meanLuminance` | 0.3368 | 0.2093 |
| `imageStats.litPixelFraction` | 0.9985 | **0.5912** |

**41% of the frame went to black.**

## What the response reported — measured

The `previewScene` block is **identical on both shots**:

- `sky.intensity: 1`
- the same `cubemap`: `EpicQuadPanorama_CC+EV1`
- `showEnvironment: true`
- `rotateLightingRig: false`

and `restore` reported clean. No field predicts the environment change, and no field records it
after the fact. A caller reading the response believes only the key moved, because that is what the
response says.

## Why the backdrop, and not just the key — inference, stated as one

`litPixelFraction` counts lit pixels across the **whole frame**, not the subject. With
`showEnvironment: true`, the background is a lit HDRI sky sphere that fills most of the frame, and
its brightness does not depend on where the directional key points — a key light swinging 180°
re-shades the *subject* and leaves the backdrop alone. A 0.9985 → 0.5912 drop is therefore not
reachable by moving the key alone: something took 41% of the frame's pixels to zero, and the only
thing large enough to do that is the environment itself rotating so a dark region of the panorama
faces the camera.

**This is an inference from the two numbers, not a source read** — no plugin or engine source was
opened for this ticket. It is the reason the ticket is filed against the *reporting*, which is
wrong either way: if the environment did rotate, `previewScene` failed to record it; if it did not,
then some other unreported change ate 41% of the frame, and `previewScene` still failed to record
it. The response is inconsistent with its own `imageStats` in both readings.

## Root cause — guess, no source read taken

The azimuth is likely applied by rotating the preview scene's whole lighting rig (the engine's
`FAdvancedPreviewScene` rotates its sky sphere together with the directional light), rather than by
setting the directional light's yaw alone. If so, `rotateLightingRig: false` is reporting the state
of a *different* knob — the auto-rotate animation — while the caller's azimuth takes the rig-rotation
path regardless. **Guess. Not read from source, and no `file:line` is claimed.**

## What is asked for

Either behaviour is defensible; the response must match whichever ships.

1. **Preferred — apply `key.azimuth` to the key light only**, leaving the environment fixed, so the
   opposed-key A/B measures what its name says. Expose the backdrop separately
   (`previewScene.environment.rotation`) for callers who *want* to turn the sky.
2. **At minimum — record it.** Emit the environment/sky rotation actually applied in the
   `previewScene` block of every capture response (`environment.rotation`, alongside the existing
   `cubemap` / `showEnvironment` / `sky.intensity`), so two responses that differ in backdrop differ
   in a field. The current block is the surface a caller uses to certify a comparison is
   apples-to-apples, and it certifies one that is not.
3. **Docs** — if the coupling is intentional, `render.capture_asset_preview`'s `previewScene.key`
   documentation should say that azimuth turns the environment with the key, and that an A/B across
   azimuths is not an isolated lighting comparison.

## Severity

**Medium.** It sits on the boundary with the High silent-wrong-data band — the response's
`previewScene` block does assert an unchanged environment across two shots whose environments
differ, on the normal path, and a caller comparing azimuths builds on that assertion. It is held at
Medium because **the same response carries the contradiction**: `litPixelFraction` fell to 0.5912
and `meanLuminance` moved 38%, so a caller who reads `imageStats` has the signal that something
beyond the key moved. The lie is in one block and the tell is in another, which makes this a
misleading readback rather than an undetectable one.

## Related

- `B-preview-rig-first-capture-stale-sky` (OPEN, Medium) — the same `previewScene` block reporting
  `applied: true` over pixels the rig never reached. Same block, same class of false certification,
  different mechanism (there the rig is not applied at all; here it is applied to more than it
  claims). A fixer touching what `previewScene` reports should read both.
- `B-preview-capture-lighting-cycles-per-shot` (OPEN, High) — captures of one stationary pose
  returning different shading solutions while every published signal reads healthy. It is the reason
  this ticket's A/B was run with a pinned EV100 in the first place, and it is the other reason two
  capture responses cannot currently be compared.
- `B-capture-preview-default-camera-vs-light` (OPEN, Medium) — records the preview scene's fixed
  environment as a 2000×-scaled `EpicQuadPanorama` sky sphere, i.e. the object this ticket infers is
  rotating. Its source citations are the nearest existing map of this area.
- `B-thumbnail-plane-elevation-edge-on` — the only other board ticket carrying a `key.azimuth`
  payload; different verb, different failure.
- `B-exposure-pin-black-frame` — a separate cause of an unexpectedly dark frame, explicitly not this
  one: exposure was pinned and `pinnedFrameUsable` held across both shots here.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Measured during a WEAPONS critic review round 2. Two `render.capture_asset_preview` calls on the same asset with the same camera and the same pinned exposure (`ev100 −1.3979541`), differing only in `previewScene.key.azimuth` 112.5 → 292.5: `imageStats.meanLuminance` 0.3368 → 0.2093 and `imageStats.litPixelFraction` 0.9985 → **0.5912**, i.e. 41% of the frame went to black. The response's `previewScene` block is **identical on both shots** — `sky.intensity 1`, the same `cubemap` `EpicQuadPanorama_CC+EV1`, `showEnvironment: true`, `rotateLightingRig: false` — and `restore` reported clean; no field predicts the change and none records it, so a caller comparing two azimuths believes only the key moved. That the backdrop is what moved is an **inference, not a source read**: `litPixelFraction` is whole-frame, the environment sky sphere fills most of it with `showEnvironment: true`, and its brightness does not depend on key direction, so a 41-point drop is not reachable by re-pointing the directional light alone. The ticket is filed against the reporting because it is wrong under either reading — if the environment rotated, `previewScene` failed to record it; if it did not, something else unreported ate 41% of the frame and `previewScene` still failed to record it. Root cause is a **guess**: the azimuth is likely applied by rotating the whole preview lighting rig (engine `FAdvancedPreviewScene` turns its sky sphere with the directional light) rather than setting the light's yaw, in which case `rotateLightingRig: false` describes the auto-rotate animation rather than the path the caller's azimuth takes — no plugin or engine source was opened for this ticket and no `file:line` is claimed. Ask, in order: apply `key.azimuth` to the key light only and expose backdrop rotation as its own parameter; failing that, emit the environment/sky rotation actually applied in the `previewScene` block of every response so two shots with different backdrops differ in a field; and if the coupling is intentional, document that an azimuth A/B is not an isolated lighting comparison. Severity Medium — it borders the High silent-wrong-data band since the block asserts an unchanged environment across two shots whose environments differ, but is held at Medium because the same response contains the tell (`litPixelFraction` 0.5912, `meanLuminance` −38%), making it a misleading readback rather than an undetectable one.
- `#2-already-fixed-no-repro` `IN-REVIEW` WEAPONS-critic — **ALREADY-FIXED** per the README convention (the defect is gone from current behaviour, so no new code; recorded as `IN-REVIEW` because `OPEN → DONE` is never allowed). Re-run in a WEAPONS critic review round 3 with the controlled pairs `#1` asked for and it does not reproduce: the backdrop no longer moves with `key.azimuth`. The response now also carries what ask (2) requested — a `previewScene.previous` block, a `previewScene.afterRestore` block, and `previewScene.restore` with `profilesRestored: true` and `configFileUnchanged: true` — so a caller can diff the preview-scene state across two shots instead of reading an identical block over two different environments. Neither the code change nor a `file:line` is claimed here: no plugin source was opened, and this entry records observed behaviour only.
- `#3-verified-controlled-pairs` `IN-REVIEW` tester — Verified with two controlled pairs, identical camera and pinned exposure, `key.azimuth` the only field changed; **disclosure: the same agent recorded `#2` and this entry**, acting as the tester the user asked to verify. Pair A (`ev100Equivalent −5.022`): backdrop mean **0.95425255** at az 112.5 vs **0.95424935** at az 292.5 — **Δ 3.2e−6**. Pair B (`ev100Equivalent −2.75`, backdrop region 210,085 px): **0.69160310** vs **0.69160806** — **Δ 5.0e−6**, with `unlitFraction` **9.5200e−5 identical** on both. Against `#1`'s measured failure — `litPixelFraction` 0.9985 → 0.5912, 41% of the frame to black — the backdrop is now stationary to six decimal places, and the unlit-pixel count does not move at all. **The rig genuinely changed on the same calls**, so this is not a no-op mistaken for a fix: subject mean moved **0.68327 → 0.68664** and the reported `key.rotation.yaw` moved **−67.5 → +112.5**, i.e. the key light rotated 180° while the environment held. Both readings `#1` left open are now closed in the same direction: the environment did not rotate, and nothing else unreported ate any part of the frame. Ask (1) (azimuth applies to the key only) is satisfied by behaviour and ask (2) (record the preview-scene state) by the new `previous`/`afterRestore`/`restore` blocks; ask (3) (document the coupling) is moot since the coupling is gone. Not re-checked this round: whether `previewScene.environment.rotation` exists as an explicit parameter for callers who *do* want to turn the sky — if a fixer wants that surface it is a new feature ask, not this defect.
