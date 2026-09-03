---
id: B-horizontal-orthographic-views-render-no-geometry
title: "Horizontal orthographic views (elevation 0) render editor overlays but no scene geometry"
status: DONE
severity: Medium
category: bug
tags: [render, camera, orthographic, capture, blank-image]
encounters: 1
lastSeen: 2026-08-13T00:00:00Z
---

# Horizontal orthographic views show overlays but no scene

After the orthographic capture fix, **top-down** orthographic (`elevation` ~90, `orthoView:
"top"`) renders correctly. The **horizontal** orthographic views — `elevation` 0, resolving to
`orthoView: "front"` / `"left"` (and by inference `back` / `right`) — still do not show scene
geometry. The capture pipeline clearly runs: the axis gizmo and the scale bar are drawn, and
the PNG is ~120-150 KB rather than the ~21 KB of the old fully-blank output. The level itself
simply is not visible, so the frame reads as near-white.

This is the residual half of the defect fixed in "Render orthographic captures instead of
returning blank images". Before that fix, **all** orthographic shots were uniformly white with
no gizmo and no scale bar at all.

## Reproduction (2026-08-13, UE 5.8, map `/Game/Maps/Dota2_Blockout`)

```
camera.orbit_shots {point:{x:0,y:0,z:0}, radius:40000, width:1024, height:1024}
```

| shot | angles | projection | orthoView | bytes | result |
| --- | --- | --- | --- | --- | --- |
| 00 | az45 / el30 | perspective | — | 1,815,587 | renders correctly |
| 01 | az0 / el0 | orthographic | front | 123,848 | **overlays only, no geometry** |
| 02 | az90 / el0 | orthographic | left | 120,936 | **overlays only, no geometry** |
| 03 | az0 / el90 | orthographic | top | 933,766 | renders correctly |

Not a framing or zoom artifact — reproduced with a tighter frame and the camera raised to
sit inside the terrain's height range:

```
render.capture_open_level {
  location:{x:40000,y:0,z:1500}, rotation:{pitch:0,yaw:180,roll:0},
  projectionMode:"orthographic", orthoWidth:70000, width:1024, height:1024
}
```

Same outcome: gizmo and scale bar drawn, no level geometry (`ortho_side_probe.png`, 147 KB).

The subject is well inside the frame at these settings. The map spans roughly
x,y ∈ [-32000, 32000] with terrain only ~3000 uu tall, so viewed edge-on at `orthoWidth`
70000-92000 it should occupy a thin but clearly visible horizontal band (~40 px of 1024).

## CONFIRMED ROOT CAUSE — transparency, not rendering

**The title of this ticket is wrong: the geometry rendered correctly every time.** The capture
encoded the editor back buffer's ALPHA channel, which is 0 over every scene pixel and 255 only
under Slate-composited overlays (the world-axis gizmo, the ortho scale bar). The written PNG was
therefore ~99.97% transparent. Whether it "worked" depended entirely on the consumer: one that
composites alpha showed a blank frame with nothing but the overlays; one that re-encodes without
an alpha channel showed the identical file rendering correctly.

That consumer split ran along FILE SIZE, which is exactly why it looked like a top-down-vs-
horizontal defect — in the canonical orbit set the two edge-on ortho shots are small enough to
survive as PNG, while the perspective and top-down shots are large enough to be re-encoded on
the way to a viewer.

Measured on the artefacts already on disk (Pillow, alpha histogram):

| file | alpha==0 | alpha==255 | distinct colours in RGB | distinct after compositing over white |
| --- | --- | --- | --- | --- |
| `ortho_side_probe.png` (pre-fix) | 99.96% | 0.04% | 885 | 21 |
| `CameraOrbit_20260813_084811_shot01_az0_el0.png` (pre-fix) | 100.00% | 0.00% | 482 | **1** |
| `CameraOrbit_20260813_093032_shot01_az0_el0.png` (post-fix) | 0.00% | 100.00% | 692 | 692 |

Corroborating, without needing a decoder: across the three generations of the same
`camera.orbit_shots` call, the alpha fix changed file size by 0.17% (shot01), 0.08% (shot02) and
1.7% (shot03). A PNG whose RGB planes held no geometry could not compress to within 0.2% of one
that does. The genuinely blank pre-`a70d6337` artefacts are visibly different at 20-28 KB.

**The near/far clip-slab hypothesis in the original report is REFUTED.** In
`C:\UE_5.8\Engine\Source\Editor\UnrealEd\Private\EditorViewportClient.cpp`, the world-bounds slab
at `:1423` is consulted only in the `!bShouldCalculateDepthRange` branches (`:1429`, `:1438`),
i.e. wireframe/unlit only — `bShouldCalculateDepthRange = ViewFamily->ViewMode > VMI_Unlit`
(`:1407`). Nothing in `:1403-1456` differs between `LVT_OrthoXY` and `LVT_OrthoYZ` except the
rotation matrix, so that code is structurally incapable of producing a top-vs-side asymmetry.

## Fix

`PinWrightScreenshotUtils::ForceOpaqueAlpha` (`Source/PinWright/Private/Utils/ScreenshotUtils.h:34`)
stamps every capture opaque; the shared editor-viewport capture calls it at
`Handlers/Render/PreviewViewportCaptureUtils.cpp:380`, before the encode. `render.capture_open_level`,
`render.capture_asset_preview`, `render.capture_annotated`, the `camera.*` verbs and
`editor.screenshot`'s level-viewport fallback all route through that one function.

Follow-up pass closed three writers that still encoded raw back-buffer alpha:
`Handlers/UI/WidgetDesignerScreenshotHandler.cpp` (window mode), the scene-only `ReadPixels`
fallback in `Utils/ScreenshotUtils.cpp` (the headless / `-RenderOffScreen` path), and
`Handlers/Drive/DriveSetOfMarkRenderer.cpp` (whose `IImageWrapper` PNG fallback would have
emitted a transparent frame).

Deliberately NOT stamped: `Handlers/UI/WidgetDesignerCaptureUtil.cpp` — a widget preview render
target has genuine transparency that is meaningful to preserve, unlike a screen capture.

## Remaining work before DONE

- Not yet compiled or run: the three follow-up writers and the new handler-level test are
  authored only. A build is required.
- `Handlers/Drive/DriveSetOfMarkRenderer.cpp:263` writes **JPEG** bytes down a PNG contract via
  `FImageUtils::ThumbnailCompressImageArray` (JPEG for any image >= 8x8). Same defect shape as the
  already-fixed `B-viewport-screenshot-writes-jpeg`, in a path that ticket did not cover. Worth
  its own ticket.

## History
- `#1-initial-repro` `OPEN` reporter — "Top-down orthographic renders correctly after the ortho fix, but elevation-0 horizontal views (front/left) still show only the axis gizmo and scale bar with no scene geometry. Reproduced via camera.orbit_shots shots 01/02 and independently via render.capture_open_level at orthoWidth 70000 with the camera at z=1500, so it is not a framing/zoom artifact. Suspect the ortho near/far slab or a dropped view-origin depth component in ResolveOrthographicView. Workaround: perspective side pose."
- `#2-root-caused-as-alpha` `IN-REVIEW` developer — "Root cause is the alpha channel, not rendering or clipping: captures encoded the back buffer's alpha, 0 over scene pixels and 255 only under Slate overlays, producing a ~99.97% transparent PNG whose appearance depended on whether the consumer composited alpha. The top-down-vs-horizontal split was a file-size artifact of that consumer behaviour, not a projection difference. Fixed by stamping every capture opaque through PinWrightScreenshotUtils::ForceOpaqueAlpha, called from the shared CaptureEditorViewportToPng before the encode. Confirmed by alpha histogram on the on-disk artefacts (pre-fix 100% alpha 0 with 482 distinct colours still present in RGB; post-fix 100% opaque) and corroborated by post-fix file sizes landing within 0.2% of pre-fix. The near/far clip-slab hypothesis is refuted at EditorViewportClient.cpp:1407/1423/1429/1438 — that slab is wireframe/unlit-only and cannot vary between LVT_OrthoXY and LVT_OrthoYZ. Also stamped three writers the first pass missed (WidgetDesignerScreenshotHandler window mode, the scene-only ReadPixels fallback in ScreenshotUtils, DriveSetOfMarkRenderer) and added a handler-level regression test on render.capture_open_level asserting zero non-opaque pixels — the pre-existing TestCaptureOpaqueAlpha.cpp unit test calls the helper itself and still passes with the production call site deleted, so it did not cover the regression. Needs a build to verify."
- `#3-runtime-verified-and-committed` `DONE` tester — Built and runtime-verified. Every check was made
  by **alpha histogram, not by eye** (eyeballing is what caused the original misdiagnosis, where a
  frame looked blank while holding 482 distinct RGB colours behind a fully transparent alpha channel).
  1. The new handler-level regression test `PinWright.render.capture_open_level.CapturedPngIsOpaque`
     passes in the full suite — unlike the shipped `TestCaptureOpaqueAlpha` unit test, it fails if the
     production call site is deleted.
  2. `widget.screenshot_designer {target:"window"}` (the clearest remaining defect, newly stamped):
     1920x1032, `alpha==0` 0.00%, `alpha==255` **100.00%**, 1791 distinct RGB colours. OPAQUE.
  3. `editor.screenshot` under **`-RenderOffScreen`** — the scene-only `ReadPixels` fallback the stamp
     was moved below the branch merge to cover, and the path least likely to be noticed by eye:
     927x525, `alpha==0` 0.00%, `alpha==255` **100.00%**, 268 distinct colours. OPAQUE.
  Also confirmed opaque: `render.capture_open_level` on the `sceneViewportReadPixels` renderer
  (640x480, 100% alpha 255, 584 colours) and `editor.screenshot` in a normal windowed editor
  (1357x866, 100% alpha 255, 2830 colours). Committed as `b0e1a6d6`.
  `DriveSetOfMarkRenderer`'s PNG fallback is stamped in this commit but was NOT exercised at runtime —
  it is only reached when the primary encoder is unavailable. The separate JPEG-down-a-PNG-contract
  defect in that same file is already tracked as `B-drive-setofmark-writes-jpeg`.
