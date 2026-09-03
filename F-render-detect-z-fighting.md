---
id: F-render-detect-z-fighting
title: "No way to detect z-fighting from a camera; render.detect_z_fighting plus a reusable offscreen scene-capture probe"
status: DONE
severity: Medium
category: feature
tags: [render, scene-capture, z-fighting, level-review, visual-verification]
---

# render.detect_z_fighting

Nothing in the plugin could answer "is anything z-fighting in this view?", and the
`level-review` guide carried a paragraph saying so. Reviewers were left comparing screenshots by
eye, which cannot see a one-pixel seam and cannot distinguish shimmer from compression noise.

Shipped: `render.detect_z_fighting` renders the same instant twice with the near clipping plane
perturbed and compares **G-buffer surface identity** (base colour and world normal). Pixels whose
depth comparison was close enough to a tie get re-decided by the perturbation and show a different
surface; everything else is bit-identical, because changing the near plane changes exactly one
projection matrix element and leaves screen position, attribute interpolation and mip selection
untouched. Both renders are the same instant, so animated water, foliage and particles contribute
exactly zero — no exclusion masks, no temporal sampling.

Also shipped: `Handlers/Render/SceneCaptureProbeUtils.{h,cpp}`, a reusable offscreen
`USceneCaptureComponent2D` probe. The plugin had **no** scene-capture path at all; every capture
drove the Level Editor viewport, which cannot produce float render targets, non-colour sources or a
caller-chosen near plane. The probe registers an ownerless transient component into the world
rather than spawning an actor, so it leaves the level, its outliner and its dirty state untouched.

## Two negative results worth preserving so they are not re-attempted

**1. Differencing two depth captures cannot work.** Z-fighting is by definition two surfaces
landing within one quantum of depth-buffer resolution, so the linear-depth difference at a flipped
pixel is **at most one quantum** — while rescaling the near plane moves *every* pixel's
reconstructed depth by about the same amount. Signal and noise are the same physical quantity.
Depth is therefore used only for clip exclusion and world placement, never differenced.

**2. A power-of-two `nearPlaneRatio` is guaranteed to report zero on any scene.** A scene capture
leaves far equal to near, selecting `FReversedZPerspectiveMatrix`'s `MinZ == MaxZ` branch, so
`DeviceZ = Near / W`. Scaling a float32 by a power of two is exact — an exponent shift, no mantissa
rounding — so every stored depth scales exactly, ordering is preserved everywhere, and not one
depth comparison in the frame can change. `4.0` is exactly the value a caller reaches for, so the
verb **rejects** power-of-two ratios with `INVALID_ARGUMENT` and an explanation. Default is 3.

## Runtime verification (integration pass 4)

The fixture pair is the evidence and neither half is meaningful alone.

| Fixture | Camera | Resolution | affectedPixels | pass |
|---|---|---|---|---|
| two coplanar slabs | grazing, pitch -15 | 1920 | **1797** (89 regions) | **false** |
| same slabs, 50 cm apart | grazing, pitch -15 | 1920 | **0** | **true** |

Every reported region named **both** fixture actors and every `worldLocation.z` landed on
200001.0, the exact slab surface. No side effects: actor count unchanged, level package not newly
dirty, viewport camera unmoved (`cameraSource: "caller"` throughout).

**Two findings from running it, both now encoded in the fixture:**

- **Exactly coplanar surfaces seen FACE-ON cannot be detected, and correctly so.** Both are planes
  perpendicular to the view, so `W` is the same constant for every pixel of both and `DeviceZ` is
  bit-identical rather than merely close. `CF_DepthNearOrEqual` resolves an exact tie by draw order
  under every projection — there is no tie to re-decide. Measured 0 affected pixels that way at
  `nearPlaneRatio` 3, 5, 7 and 11 and in `farPlane` mode. Tilting one slab 1 degree does not help:
  fighting needs the pair within ~6e-8 of `W`, and a 1-degree gradient crosses that band over
  ~1e-3 uu, about 0.007 px. Seen obliquely the same pair fights properly.
- **The resolution default is load-bearing, measured.** At the same camera on the same scene,
  1920x1920 reports 1797 affected pixels and 512x512 reports **exactly zero**. The verb emits a
  `resolutionWarning` below the 1920 long edge; do not lower it.

Nanite false-positive rate was not measured on a Nanite-heavy view; `perturbation: "farPlane"`
remains the documented escape hatch (it re-dices the depth encoding while `ComputeNearPlane()`
still evaluates to `N`, so Nanite LOD is untouched).

## History
- `#1-designed-and-built` `IN-REVIEW` developer — New files `Handlers/Render/SceneCaptureProbeUtils.{h,cpp}`, `ZFightingAnalysis.{h,cpp}`, `ZFightingHandler.cpp`; `ErrorCodes.h` +3 (`READ_PIXELS_FAILED`, `RENDER_TARGET_CREATE_FAILED`, `SCENE_CAPTURE_FAILED`); 5 tests in `Tests/Render/TestZFightingDetect.cpp`. No `Build.cs` change. Not built or run at authoring time.
- `#2-runtime-verified` `DONE` tester — Built (UE 5.8, `Result: Succeeded`, zero errors and zero warnings in both console and UBT logs, `-DisableAdaptiveUnity` proven by 0 standalone `.cpp` compile actions, fresh DLL timestamps — not the exit code). `CoplanarFixtureIsDetected` FAILED on first run at `affectedPixels: 0`; escalated the note's ladder live (ratios 5/7/11, `farPlane` mode, 1-degree tilt — all zero), diagnosed the face-on exact-tie cause, and corrected the fixture camera to oblique plus removed its 512 resolution override. All 5 tests pass after the fix; the two findings above are recorded in comments beside the fixture. Committed as `bb801588`.
