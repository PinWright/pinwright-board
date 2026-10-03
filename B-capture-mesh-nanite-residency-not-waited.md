---
id: B-capture-mesh-nanite-residency-not-waited
title: "render.capture_mesh does not wait for Nanite page residency before its first draw; only the settle redraw can catch a frame missing Nanite pages"
status: OPEN
severity: Medium
category: bug
tags: [render, capture_mesh, nanite, streaming, cold-frame, readiness]
encounters: 1
lastSeen: 2026-10-03
rice: [1, 3, 0.5, 3]
priority: 6
---

# Nanite pages are not waited on by `render.capture_mesh`

**Verified in source:** `B-capture-mesh-cold-first-frame-no-readiness-wait` `#2` waits for the mesh build, the slot materials' shader maps and their texture mips (`PinWrightThumbnail::WaitForThumbnailSubjectReadiness`). It does not wait for Nanite page streaming. The first shot is then redrawn until two consecutive frames agree (`PinWrightMeshPreviewCapture::SettleFrame`). Both draws run inside one game tick, so this catches only pages the renderer streams in between the two draws. A Nanite mesh whose pages need ticks to arrive can draw at coarse clusters twice and report `frameSettled:true`. The engine streaming manager (`Nanite::GStreamingManager`, Engine `Rendering/NaniteStreamingManager.h`) has no public "all pages resident for this resource" wait. A fix needs a bounded pump-and-poll (tick / `FlushRenderingCommands`, repeated draws until stable over ticks) or a project-side residency request.

Not reproduced: the relayed cold frame in the parent ticket was a propeller mesh, and its Nanite state was not recorded.

**Fix:** for a Nanite-enabled mesh (`UStaticMesh::IsNaniteEnabled()`), pump a bounded number of render frames and redraw until two frames taken across ticks agree. Report the result in `readiness` (e.g. `naniteResidencyWaited`, pumped frames) and warn when the bound is hit.

## History

- `#1-split-from-cold-frame-ticket` `OPEN` developer — Split out of `B-capture-mesh-cold-first-frame-no-readiness-wait`, which named Nanite residency as a candidate cause. That ticket's `#2` did not wait on it.
