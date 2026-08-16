---
id: B-set-view-mode-writes-only-active-projection-slot
title: "editor.set_view_mode returned success while every orthographic capture stayed on the previous mode — FEditorViewportClient keeps Persp and Ortho view-mode slots separately and SetViewMode writes only the one matching the current projection"
status: IN-REVIEW
severity: High
category: bug
tags: [editor, viewport, set_view_mode, ortho, projection, capture, misleading-success, readback]
---

# Success on one projection slot, stale on the other

`editor.set_view_mode {viewMode: "Lit"}` returned `{"success": true, "viewMode": "Lit"}` and the very
next capture reported `viewMode: "Unlit"`, `lit: false`.

`FEditorViewportClient` keeps **`PerspViewModeIndex` and `OrthoViewModeIndex` as separate fields**,
and `SetViewMode` writes only the one matching the client's *current* projection. A mode set from a
perspective viewport therefore left every **orthographic** capture rendering in the previous mode,
while the response reported the value that had been written to the slot nobody was reading.

## Measured consequence

A 20-capture chunked reference-comparison pass came back entirely Unlit — recorded at
`Docs/map/reference_tile_compare.md:239-244` in the host project:

> **`editor.set_view_mode` returned `{"success": true, "viewMode": "Lit"}` and the very next capture
> still reported `viewMode: "Unlit"`, `lit: false`.** A verb reporting success while doing nothing.
> All 20 captures in this pass are therefore **Unlit** ... these frames cannot support any material,
> lighting or colour judgement.

The pass survived because it only needed geometry positions. Any pass needing material, lighting or
colour would have drawn conclusions from frames rendered in the wrong mode, with a success response
saying otherwise.

## Distinct from the already-closed ticket

`B-set-view-mode-exec-failed` (DONE) fixed a **hard failure** — `GEditor->Exec` returning false and
the handler emitting `EXEC_FAILED` because the `viewmode` console command never reached the active
level viewport client. Its verification was `{viewMode:"Lit"}` -> `{"success":true,"viewMode":"Lit"}`,
which is exactly the response this defect also produces. **The verdict was scoped to the return
value, not to the rendered frame** — the same shape as
`B-compile-material-landscape-consumers-stale` vs `B-compile-mgir-landscape-consumers-stale`. Filed
as a separate ticket rather than reopening, following that precedent.

## Fix (shipped `5af2b544`, plus the capture half in `2f243c48`)

- `Handlers/Editor/ViewportHandler.cpp:447` onward now writes **both** projection slots and reports
  each one measured, rather than reporting the requested value. Regression test:
  `Tests/EditorOps/TestSetViewModeProjectionSlots.cpp` (244 lines), whose header at `:6` states the
  defect it guards.
- `2f243c48` closed the reporting half that had hidden it: the view mode was present in the shared
  capture struct but **only one of seven response builders serialized it**, so most captures never
  reported the mode they were taken in. `Handlers/Render/PreviewViewportCaptureUtils.{h,cpp}` now
  owns it and `AnimationShotsHandler`, `AnnotatedCaptureHandler`, `CameraFrameHandler` and
  `RenderHandler` all emit it; test `Tests/Render/TestCaptureViewModeReporting.cpp` (210 lines).

Both commits are on `origin/master`.

## Related

- `B-set-view-mode-exec-failed` (DONE) — the earlier, different failure in the same verb.
- `B-set-game-view-does-not-suppress-spline-overlays` — the sibling capture-fidelity finding from the
  same pass.
- `B-set-camera-no-viewport-redraw` — same family: viewport commands whose success does not imply the
  frame changed.

## History
- `#1-success-on-the-slot-nobody-reads` `OPEN` reporter — `editor.set_view_mode {viewMode:"Lit"}` returned `{"success": true, "viewMode": "Lit"}` while the very next capture reported `viewMode: "Unlit"`, `lit: false`. Root cause: `FEditorViewportClient` keeps `PerspViewModeIndex` and `OrthoViewModeIndex` as **separate fields**, and `SetViewMode` writes only the slot matching the client's current projection, so a mode set from a perspective viewport left every orthographic capture on the previous mode — and the response echoed the value written to the slot nobody was reading. Measured consequence recorded in the host project at `Docs/map/reference_tile_compare.md:239-244`: all 20 captures of a chunked reference-comparison pass came back Unlit and were explicitly disqualified from supporting "any material, lighting or colour judgement". The pass survived only because it measured geometry positions. **Deliberately filed separately from `B-set-view-mode-exec-failed` (DONE) rather than reopening it:** that ticket fixed a hard `EXEC_FAILED` where the console command never reached the active level viewport client, and its verification was `{viewMode:"Lit"}` -> `{"success":true,"viewMode":"Lit"}` — the very response this defect also produces. Its verdict was scoped to the return value, not to the rendered frame, which is the same pattern as the `compile_material` / `compile_mgir` pair.
- `#2-both-slots-written-and-each-reported-measured` `IN-REVIEW` developer — Fixed in `5af2b544` (on `origin/master`). `Handlers/Editor/ViewportHandler.cpp:447` onward writes **both** projection slots and reports each one **measured** rather than echoing the request, with `Tests/EditorOps/TestSetViewModeProjectionSlots.cpp` (244 lines) guarding it — its header at `:6` names the separate-fields defect directly. The reporting half that let this hide for as long as it did was closed in `2f243c48`: the view mode already existed in the shared capture struct but **only one of seven response builders serialized it**, so most captures never disclosed the mode they were taken in; `Handlers/Render/PreviewViewportCaptureUtils.{h,cpp}` now owns it and `AnimationShotsHandler`, `AnnotatedCaptureHandler`, `CameraFrameHandler` and `RenderHandler` all emit it, covered by `Tests/Render/TestCaptureViewModeReporting.cpp` (210 lines). Worth keeping the pairing visible: the write defect and the reporting gap were independent, and either alone would have kept the other invisible. Filed retroactively — this went discovery -> fix -> ship inside a day and would otherwise have left no board record.
