---
id: F-render-runtime-spawned-actor
title: "No verb renders an actor that is not placed in the active level — nothing captures an actor class or a runtime-spawned actor without dirtying the user's map"
status: IN-REVIEW
severity: Medium
category: feature
tags: [render, capture, preview, actor, blueprint, runtime-spawn, visual-verification, active-world]
encounters: 2
costly: 1
lastSeen: 2026-08-18T00:00:00Z
---

# The capture surface covers assets and the active level, and nothing in between

Current capture verbs:

- `render.capture_asset_preview` — **Static Mesh asset editors only**. Anything else is rejected
  with a typed error naming `render.capture_animation_preview`
  (`Handlers/Render/RenderHandler.cpp:212`, rejection at `:277-280`).
- `render.capture_animation_preview` — Persona / skeletal preview.
- `render.capture_open_level`, `editor.screenshot`, `render.capture_annotated` — whatever is
  already sitting in the active level viewport.
- `asset.generate_thumbnail` (`Handlers/Asset/AssetWorkflowHandler.cpp:578`) — the nearest
  existing capability, and it does render an actor Blueprint offscreen. It is a *thumbnail*: no
  camera placement, no framing to bounds, no exact-size output — and it renders the **asset**, not
  an actor instance carrying the state it only has once spawned.

Nothing answers "show me this actor". An actor that exists only at runtime — spawned by a game
mode, a spawner, or a data payload — cannot be rendered at all.

## Why it came up, and why it is worth a verb

This session's football world (arena, ball, both goals, six spawn points, 28 boost pickups) was
spawned at runtime from a JSON payload and placed in no map, so no capture verb could see any of
it while the visual work — ball scale, boost-pickup size and emissive — was exactly what needed
looking at.

**Two agents independently invented the same workaround**: spawn a stand-in (`DFH_BoxPreview`)
into the active level, `render.capture_open_level`, then delete it. Independent reinvention of the
same three-step dance is the signal here — the shape is obvious enough that everyone derives it,
which is the argument for it living in the plugin rather than in every caller.

The workaround is also unsafe in the way that matters: it **mutates the user's active level to
take a read-only picture**, dirtying the map and, if any step fails, leaving the stand-in behind.
This is the same "read path that mutates the active world" objection
`E-level-getters-require-loaded-not-on-disk` argued from, here paid in a dirty map. It is
particularly bad against a live editor — this session's editor was serving the user's own flight
testing.

**Requested:** `render.capture_actor_preview` taking either `classPath` (spawn a transient
instance) or `actorPath` (an existing actor, including one in a PIE world), framing the camera to
the actor's bounds the way `B-capture-asset-preview-renders-empty`'s fix already frames a static
mesh, capturing at an exact size, and destroying/restoring on **every** exit path. It should reuse
`PreviewViewportCaptureUtils` — realtime override, pump loop, `ReadPixels`, `imageStats`, blank
classification — rather than growing a second capture path with its own bugs.

## Related

- `B-capture-asset-preview-renders-empty` (IN-REVIEW) — its Workaround section already names this
  gap from the other direction: *"for mesh visual review, fall back to spawning the mesh into a
  level and using a level-viewport capture ... which is exactly the no-spawn workflow this verb
  was meant to avoid."* That ticket fixes the static-mesh path; it does not add an actor path.
- `B-inspect-misses-pie-world` (DONE) — closed the same blind spot for `system.inspect.*` and
  `actor.*` queries. The render surface never got the equivalent, so a PIE-world actor can be
  queried but not seen.
- `F-widget-designer-screenshot` (DONE) — precedent shape: a capture verb for a thing with no
  level presence.
- `E-generate-thumbnail-undocumented` (DONE) — the thumbnail verb this is repeatedly mistaken for.

## History
- `#1-two-agents-same-workaround` `OPEN` reporter — No capture verb reaches an actor that is not already in the active level. `render.capture_asset_preview` is Static Mesh asset editors only and rejects everything else with a typed error (`RenderHandler.cpp:212`, `:277-280`); `render.capture_animation_preview` is Persona; `render.capture_open_level` / `editor.screenshot` / `render.capture_annotated` capture whatever is already in the level viewport; `asset.generate_thumbnail` (`AssetWorkflowHandler.cpp:578`) renders the **asset** at thumbnail scale with no camera, framing or exact-size control, not a spawned instance. So an actor that exists only at runtime is unrenderable. Hit this session on the football world (arena, ball, 2 goals, 6 spawn points, 28 boost pickups) — spawned at runtime from a JSON payload, in no map, while ball scale and boost-pickup size/emissive were exactly the things under review. **Two agents independently invented the same `DFH_BoxPreview` spawn-render-delete workaround**, which is the signal: the shape is obvious, so it belongs in the plugin. It also mutates the user's active level to take a read-only picture (dirties the map, leaks the stand-in if any step fails) — the same objection `E-level-getters-require-loaded-not-on-disk` raised against mutating reads, and worse against a live editor, which this session's was. Requested: `render.capture_actor_preview {classPath | actorPath}` — spawn transient or target an existing actor including one in a PIE world, frame to bounds as `B-capture-asset-preview-renders-empty`'s fix already does for static meshes, exact-size capture, destroy/restore on every exit path, reusing `PreviewViewportCaptureUtils` (realtime override, pump, `ReadPixels`, `imageStats`, blank classification) rather than a second capture path. Cross-refs: `B-capture-asset-preview-renders-empty` (IN-REVIEW) already names this gap in its own Workaround section; `B-inspect-misses-pie-world` (DONE) closed the equivalent blind spot for `actor.*`/`system.inspect.*` queries but not for rendering; `F-widget-designer-screenshot` (DONE) is the precedent shape.
- `#2-capture-actor-preview` `IN-REVIEW` developer — Added `render.capture_actor_preview {classPath | actorPath}`. `classPath` (classref: UClass name, /Script path, BP asset path) spawns an `RF_Transient` instance into a private `FPreviewScene` world; an `ON_SCOPE_EXIT` destroys the instance and the world on every exit path, error paths included, and `transientDestroyed` is read off a weak handle afterwards. `actorPath` resolves through `ActorNameParamUtils::ResolveActorOrSendError` (PIE world first, then editor; ambiguity -> `AMBIGUOUS_ACTOR_NAME`) and draws the actor in its own world under `FScopedPackageDirtyRestore`; the actor is never moved or hidden. Both paths render through the existing ownerless `FSceneCaptureProbe::CaptureColor` (render.capture_mesh's renderer: `ImageStats`/`blank`, PNG encode, `Saved/Screenshots/ActorPreview`), with a new additive `bShowOnlyActors`/`ShowOnlyActors` request field (`PRM_UseShowOnlyList`) so only the subject's primitives draw; `subjectCoverage` is measured against the same pose with an EMPTY show-only list, so the subject is never touched. Framing reuses `PinWrightCameraFrame::ComputeFitDistance` / `PlaceOrbitCamera` / `ComputeOrthoWorldWidth` (`azimuth`, `elevation`, `padding`, or explicit `location`+`rotation`); exact `width`/`height`; `exposure` and scene-capture `viewMode` via `ParseOffscreenCaptureRequest`. Deviation from the request: it does not reuse `PreviewViewportCaptureUtils`' editor-viewport path - that drives a Slate viewport and cannot isolate one actor; the scene-capture probe is the path render.capture_mesh already uses. `previewScene` is deliberately not declared (show-only draw cannot show a floor; actorPath has no preview scene), recorded as a reasoned row in `TestCaptureVerbParameterParity.cpp`. A spawned class gets construction scripts but no `BeginPlay`; runtime-only state needs `actorPath` on a PIE actor. Files: new `Handlers/Render/ActorPreviewCapture.{h,cpp}`, `Handlers/Render/ActorPreviewCaptureHandler.cpp`, `Tests/Render/TestActorPreviewCapture.cpp`; edited `Handlers/Render/SceneCaptureProbeUtils.{h,cpp}` (show-only fields), `Dispatch/SafePoint.cpp` (tick-unsafe table), `Tests/Infra/TestRenderingUnavailableGuard.cpp` (guarded-verb list), `Tests/Render/TestCaptureVerbParameterParity.cpp` (previewScene exception row), `docs/wiki-src/render.md`, `CHANGELOG.md`. Tests: `PinWright.render.capture_actor_preview.SpawnedClassRendersAndIsDestroyed`, `.ExistingActorDrawnShowOnlyAndUntouched` (neighbour-in-frame control proves show-only), `.EmptyBoundsRefusedAndInstanceDestroyed` (no GPU needed), `.ExactlyOneSourceRequired`; GPU tests emit `PINWRIGHT_ASSERTIONS_SKIPPED` under NullRHI. Not run here (manager owns the suite slot); no PIE-world test (host PIE logs FRelayClient errors).
- `#3-review-fixes` `IN-REVIEW` developer — Applied the first review. (1) Child Actor Components: the show-only list now adds `Actor->GetAllChildActors(..., true)`, so a Blueprint assembled from child actors is drawn, not just framed. (2) View mode is reported: `MeshPreviewCaptureUtils`' `ConfigureMeshPreviewOutput` is exported (it now takes the `FViewportCaptureRequest`) and fills the actor capture's output, and the response carries render.capture_mesh's `viewport` block (`type` `TransientPreviewScene` | `ActorWorld`, `viewModeOverride.applied` / `showFlagMismatches`). NITs: `skyCaptureIncomplete` + `skyCaptureWarning` when the spawned world's sky capture stayed queued; the probe's `ON_SCOPE_EXIT` restores `PRM_RenderScenePrimitives` after a show-only draw; render.md lists `world: other` and the `ACTOR_SPAWN_FAILED` / `SCENE_CAPTURE_FAILED` / `READ_PIXELS_FAILED` / `WRITE_FAILED` / `SAVE_FAILED` refusals. README row left for commit time. New tests: `PinWright.render.capture_actor_preview.ChildActorComponentPartsAreDrawn`, `.EditorActorByNameReportsWorldAndViewport` (editor-world `actorPath` through the handler: `world: editor`, dirty flag unchanged, `viewport.viewModeOverride` for `unlit`). Files: `ActorPreviewCapture.{h,cpp}`, `ActorPreviewCaptureHandler.cpp`, `MeshPreviewCaptureUtils.{h,cpp}`, `SceneCaptureProbeUtils.cpp`, `TestActorPreviewCapture.cpp`, `docs/wiki-src/render.md`, `CHANGELOG.md`.
- `#4-run1-test-fixes` `IN-REVIEW` developer — Run1 fixes. (a) `previewScene` is read by the shared `ParseOffscreenCaptureRequest`, so it is now declared and refused by the verb with `INVALID_ARGUMENT` and a reason (test `.PreviewSceneRefusedWithReason`); the parity Declaration row became obsolete and was replaced by a Description row (the shared rig prose would be false here). (b) The editor-world test now passes the object path: label/name lookup skips `RF_Transient` editor actors by design; filed `E-actor-not-found-hides-transient-label-match` for the misleading refusal; test renamed `.EditorActorByPathReportsWorldAndViewport`. (c) `CaptureInWorld` calls `World->SendAllEndOfFrameUpdates()` before drawing: a mesh set after registration only queues its proxy rebuild, so an actor edited in the same frame (the test fixtures, or a prior RPC in the same tick) drew nothing.
- `#5-existing-actor-test-lighting` `IN-REVIEW` developer — Corrects #4 (c): the end-of-frame-flush hypothesis was wrong and the flush is removed. A live diagnostic run showed the proxy existed before the capture, and the camera was at +X looking -X (coverage 0, frame max luminance 0.047). The FPreviewScene's default key light arrives from azimuth 112.5, so every face the azimuth-0 camera saw (the subject's +X face and the neighbour's -Y face) was in full shadow, black on the black show-only background. The subject was drawn and still measured 0 coverage. Test-only fix in `ExistingActorDrawnShowOnlyAndUntouched`: the scene's key light now arrives from +X (`SetLightRotation(-40,180,0)`), and the neighbour moves to (-80,110,0) so that its lit +X face is in frame. No production change. Note for later: `subjectCoverage` reads ~0 for a subject drawn in full shadow against a black background, the same as render.capture_mesh's measure.
- `#6-linux-verification` `IN-REVIEW` tester — run3/full on the committed tree (PinWright 8de8a5a2, pushed as 7230b41d), all non-skipped: `PinWright.render.capture_actor_preview.SpawnedClassRendersAndIsDestroyed`, `.ExistingActorDrawnShowOnlyAndUntouched`, `.ChildActorComponentPartsAreDrawn`, `.EditorActorByPathReportsWorldAndViewport` (world editor, dirty flag unchanged, viewModeOverride), `.EmptyBoundsRefusedAndInstanceDestroyed`, `.ExactlyOneSourceRequired` and `.PreviewSceneRefusedWithReason`. Demonstrated: `classPath` spawns a transient instance that is destroyed on success and error paths; editor-world `actorPath` draws only the subject and leaves the map clean; framing, exact size, imageStats/blank and coverage are present. Remaining: (1) the ticket asks for `actorPath` on "an existing actor, including one in a PIE world". No PIE-world test exists (developer #2), so that half is unverified. An owned host-neutral PIE test, like PinWright.editor.screenshot.PieExposureAndAim uses, could cover it. (2) A human must accept a deliberate deviation: it renders through the scene-capture probe (render.capture_mesh's renderer), not `PreviewViewportCaptureUtils` as requested. (3) A spawned class runs construction scripts but not BeginPlay, so runtime-only state needs the PIE path that is still unverified.
