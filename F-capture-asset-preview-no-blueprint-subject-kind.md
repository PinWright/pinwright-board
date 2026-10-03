---
id: F-capture-asset-preview-no-blueprint-subject-kind
title: "render.capture_asset_preview has no Blueprint subject kind, so what a construction script builds can be listed (blueprint.preview_construction) but not looked at"
status: OPEN
severity: Low
category: feature
tags: [render, capture-asset-preview, blueprint, construction-script, subject-kind, preview]
encounters: 1
lastSeen: 2026-10-01T00:00:00Z
rice: [1, 1, 1, 2]
priority: 4
---

# No Blueprint subject kind on the preview-capture verb

`render.capture_asset_preview` classifies its subject in `ClassifyAssetKind`
(`Source/PinWright/Private/Handlers/Render/CaptureSubject.cpp`, the `UNSUPPORTED_ASSET_EDITOR`
refusal around line 707): Static Mesh, Skeletal Mesh, animation assets and Niagara System. An
Actor Blueprint is refused with "is a Blueprint, which is not a capture subject kind".

`blueprint.preview_construction` (F-blueprint-preview-construction) now returns the components a
construction script builds, as data. Looking at the result still needs a placed actor plus a level
capture, which dirties the map and puts the actor in the outliner.

**Possible shape:** a `blueprint` subject provider that spawns the class the same way
`blueprint.preview_construction` does (deferred, `RF_Transient`, private `FPreviewScene`, caller
`variables` applied before `FinishSpawning`) and frames it with the existing preview-scene rig.
This shares the spawn with that verb instead of driving the Blueprint editor's viewport toolkit,
which the capture allow-list would also need to admit.

## History
- `#1-split-from-preview-construction` `OPEN` developer — Split from F-blueprint-preview-construction, whose title names this gap but whose Fix and Acceptance cover only the data verb. The verb shipped. This capture half is left open.
