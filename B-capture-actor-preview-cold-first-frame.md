---
id: B-capture-actor-preview-cold-first-frame
title: "render.capture_actor_preview draws one CaptureScene with no material/texture readiness wait and no settle check, so a cold first frame returns as success"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_actor_preview, cold-frame, readiness, streaming, shader-map, silent-false-success]
encounters: 1
lastSeen: 2026-10-03
rice: [1, 3, 0.8, 2]
priority: 13
---

# `render.capture_actor_preview` has the cold-first-frame gap `render.capture_mesh` had

**Verified in source (PinWright 7230b41d + the B-capture-mesh-cold-first-frame-no-readiness-wait fix):**
`render.capture_actor_preview` does NOT use `FMeshCaptureSession`; it builds its own
`FSceneCaptureProbe` in `ActorPreviewCaptureLocal::CaptureInWorld`
(`Handlers/Render/ActorPreviewCapture.cpp:68,118`). It draws once, with no wait on the actor's
primitives' materials (shader maps) or their textures' mips, no settle redraw, and it publishes no
`readiness` / `redrawRetries`. So the readiness wait and settle check added to `capture_mesh`
(`PinWrightThumbnail::WaitForThumbnailSubjectReadiness`, `PinWrightMeshPreviewCapture::SettleFrame`)
do not cover it, contrary to the batch-5 ranking note that assumed a shared session.

Not reproduced on this verb; the mechanism is the same one filed for `capture_mesh`.

**Fix:** collect the used materials of the actor's primitive components (`GetUsedMaterials`), run the
readiness wait over them before the first draw, call `SettleFrame` on the first draw, and publish
`readiness`, `redrawRetries`, `frameSettled` / `frameWarning` through
`PinWrightMeshPreviewCapture::AddMeshFrameEvidenceFields`.

## History

- `#1-filed-from-source` `OPEN` developer — Filed while fixing `B-capture-mesh-cold-first-frame-no-readiness-wait`: the scorer's note said fixing `FMeshCaptureSession` would cover this verb, but the source shows it does not share that session. Dedup: no other ticket covers cold frames on this verb (`B-subject-coverage-blind-to-shadowed-subject` is about coverage, `F-render-runtime-spawned-actor` introduced the verb).
