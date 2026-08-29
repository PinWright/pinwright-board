---
id: F-ortho-tile-reference-compare
title: "No chunked orthographic capture and no world-coordinate image verbs, so every reference-image comparison hand-rolls its own pixel maths"
status: IN-REVIEW
severity: High
category: feature
tags: [render, image, orthographic, tiling, annotation, comparison, reference-image, world-coordinates, batch, visual-review]
---

# A reference-image workflow with doctrine but no verbs

Comparing a level against a reference image is already a documented, prescribed workflow —
`level-building.working-from-references` (register before you measure; tile the reference and the
capture **identically**; coarsest grid that answers the question), `level-review.framing-math`
(ortho is axis-aligned only; establish which screen axis is which before measuring a pixel),
`level-building.capture-and-review` (pin exposure or two passes are not comparable). Every verb
those pages need is missing. The plugin can take **one** ortho frame over **one** `orthoWidth`; it
cannot cut a grid, cannot mark a world coordinate on an image, and cannot compare two images tile
for tile.

So the caller writes it. In the host project that is `Docs/scripts/compare/tile_compare.py` — 457
lines of `plan` / `ref` / `compare` / `check` reimplementing world-to-pixel projection, tile boxes,
capture-arg emission, reference cropping and a per-tile metric, entirely outside the plugin and
entirely unverified by it. Every caller who does this writes the same maths again and gets a
different subset of it wrong.

**The error is not theoretical.** Coordinates eyeballed off a whole-map reference were off by
roughly **1188 uu**; chunking the comparison to tiles at the reference's native resolution improved
positional accuracy about **10x**. The published registration budget for that same reference is
**+/-195 uu at map centre and +/-867 uu at the edge** — 5.4% of a tile on a 4x4 grid, 21.7% on
16x16, which is what ruled 16x16 out before a single capture was taken
(`Docs/map/reference_tile_compare.md` section 1). Getting that arithmetic right is the entire value
of the comparison, and it currently lives in a throwaway script.

## The four verbs

| Verb | Namespace | Does |
|---|---|---|
| `render.capture_ortho_tiles` | existing `render.*` | One call renders an N x M grid of orthographic captures covering a caller-given **world extent** — one PNG per tile, one held resolution, one held exposure, one held view mode. Returns each tile's world box alongside its file. |
| `image.tile` | **new** `image.*` | Cuts an existing image (the external reference) into the *same* grid, given that image's registration (centre pixel, uu-per-pixel, world extent). Returns per-tile world boxes and crop rects. |
| `image.annotate` | **new** `image.*` | Draws marks, boxes, labels and a scale bar on an image at **world** coordinates, using that image's registration. |
| `image.compare` | **new** `image.*` | Pairs two images (or two tile sets) that claim the same world box and reports per-tile agreement plus a side-by-side sheet. |

`render.capture_ortho_tiles` belongs under `render.*` to sit beside `render.capture_open_level`,
whose parameters and ortho response fields it extends rather than replaces. `image.*` is a **new
top-level namespace** — nothing named `image` exists today.

**Design principle: the plugin renders, the reader judges.** These verbs are deterministic
rendering, cutting and pairing only. No feature detection, no registration solving, no "this tile
differs because...". The plugin's job is to make two images provably describe the same world box;
deciding what the difference means is the caller's.

## The load-bearing design rule: world space in, world space out

**Every coordinate crossing the boundary is world space, never pixels.** The caller says "tile
x[-32000,32000] y[-32000,32000] into 4x4" and "mark (-22000, +22000)"; the verb does the
projection. That single rule is what removes the error-prone hand-rolled conversion, and it is the
difference between this feature and a thin wrapper over `render.capture_open_level`.

It also forces the verbs to *carry* the registration rather than assume it. `image.tile`,
`image.annotate` and `image.compare` all operate on an image whose pixel-to-world mapping came from
somewhere; that mapping must be an explicit input and must be echoed in the response, so a
comparison can be audited afterwards instead of trusted. Per `rpc-design.md:191`, design the verb
so the cross-tile question (coverage, spread, where the deficit concentrates) is answerable — that
is the reason it is one call and not N.

## Implementation constraints, each a real trap

- **Ortho supports only six cardinal poses, and the capture reports an *effective* rotation that is
  not the requested one.** The cardinal table is `PreviewViewportCaptureUtils.cpp:63-70`
  (`LVT_OrthoXY` top ... `LVT_OrthoXZ` left); `ResolveOrthographicView` (`:407-439`) sets
  `OutResolution.EffectiveRotation` (`:421`) and `bRotationSnapped` (`:424-425`) against a 1-degree
  tolerance (`PreviewViewportCaptureUtils.h:60`). A pose further off axis is **rejected**, not
  silently rendered — `ERR_UNSUPPORTED_ORTHOGRAPHIC_ROTATION` at `:629`, guarded by the comment at
  `:621-623`. But even an accepted pose is quantised: the response returns `cameraRotation` (the
  pose the **pixels** show, `RenderHandler.cpp:112-117`), `requestedRotation` only when it differs
  (`:118-122`), and `orthoView` (`:126-129`). **Build the world-to-tile mapping from
  `cameraRotation`/`orthoView`, never from the request.** Concretely: `LVT_OrthoXY` renders
  screen-up = world **-X** and screen-right = world **-Y**, so a requested yaw 0 comes back as yaw
  180. The host project's first comparison run missed this and produced plausible-looking sheets
  that were wrong in **every tile**; the fix was a blanket `ROTATE_90` in the harness
  (`Docs/map/reference_tile_compare.md` section 1, "Two registration traps"). Note the split
  tracked by `B-blockout-review-ortho-snap-contradiction`: raw capture rejects off-axis while the
  `camera.*` helpers snap and report `orthoAxisSnapped` instead.
- **There is no `orthoHeight`** — zero matches plugin-wide. `orthoWidth` is the world span left to
  right (`PreviewViewportCaptureUtils.h:76-80`, "NOT the editor's raw ortho zoom") and the vertical
  span follows from aspect ratio (`Docs/wiki-src/render.md:156`); the zoom maths reads only `SizeX`
  (`PreviewViewportCaptureUtils.cpp:452,465`), so `H/W` is implicit and written nowhere in code. A
  non-square tile must be expressed through `orthoWidth` plus `width`/`height`, and the verb must
  reject a height it would otherwise silently ignore. Full existing param set for reference:
  `RenderHandler.cpp:385-399`.
- **Ortho zoom is viewport-pixel-width dependent, so `SetFixedViewportSize` must run *before*
  `ApplyCaptureCamera`.** Already stated in-source at `PreviewViewportCaptureUtils.cpp:687-690`
  ("Fixed size FIRST: the ortho zoom conversion in ApplyCaptureCamera is calibrated against the
  viewport's pixel width"), with a second ordering constraint inside `ApplyCaptureCamera` itself
  (`:498-500`, must follow `SetViewportType`). Reversing either gives a frame whose world coverage
  is not the requested `orthoWidth`, and the frame still looks fine.
- **Capture resolution must not vary within a session.** `Docs/wiki-src/render.md:26-38`:
  *"Varying capture resolution call-to-call killed an editor on 2026-08-13 and cost 66 actors and
  125 emitters of unsaved level state"* — `Assertion failed: ProxyMap.Num() == TestSizeX *
  TestSizeY` in `FViewport::GetHitProxy()`. *"Resizing is not the problem; changing the size is"* —
  48 back-to-back cycles at a constant 640 produced zero asserts. A tile burst must hold **one**
  size for the whole run and reject a request that would change it mid-burst.
- **Any verb reaching `CaptureEditorViewportToPng` must be added to `GTickUnsafeMethodNames`**
  (`Private/Dispatch/SafePoint.cpp:45-144`). The rationale is in the file at `:26-35`: the capture
  family pumps Slate, draws the viewport and flushes rendering commands three times per capture, and
  doing that from inside `UWorld::Tick` is a documented stall/deadlock source. **Entries are the
  registered method name — the first argument of `REGISTER_RPC_HANDLER` — and a typo fails
  *silently*, leaving the verb ungated** (`:15-18`); `PinWright.core.safe_point.TickUnsafeMethodsAreRegistered`
  is the guard. Existing capture entries are at `:64-87`. `CaptureEditorViewportToPng` itself:
  decl `PreviewViewportCaptureUtils.h:361-369`, def `PreviewViewportCaptureUtils.cpp:593` — and the
  inline `(:415-418)` citation in the `SafePoint.cpp` comment is now **stale** against that `:593`,
  worth correcting in the same change.
- **The burst is long-running, so it needs `Ctx.StartJob`** (`HandlerContext.h:153`,
  `HandlerContext.cpp:577`; pattern is `FJobBindArgs` + `BindNativeDelegate` + `Ctx.StartJob`, e.g.
  `RenderHandler.cpp:622-677`). **But `StartJob` is not a deferral primitive** — `rpc-design.md:198`
  and `Dispatch/SafePoint.h:127-128`: the bind delegate is invoked *synchronously*, so a job whose
  delegate does the work inline still runs on the caller's stack. Getting a ticket back is not the
  same as getting the burst off the caller's thread.
- **One batch verb, not a caller loop** — `rpc-design.md:182-192` ("Batch the work; never make the
  caller loop"). The section's own evidence is round-trip cost (~5100 calls wedged an editor for
  168+ minutes; the same engine work inside one call took 0.1 s), but here there is a second reason:
  a caller loop over `render.capture_open_level` cannot hold resolution, exposure and view mode
  constant across the burst, and that constancy is the only thing making the tiles comparable. Note
  `:192` — the write side must be idempotent.
- **All tiles must share one exposure pin, or the comparison is invalid**, and **no capture verb
  takes an exposure parameter today** (zero `exposure`/`EyeAdaptation` matches under
  `Private/Handlers/Render/`). Pinning is a separate prior call:
  `lighting.set_exposure {minBrightness: 1.0, maxBrightness: 1.0}` (`LightingHandler.cpp:767-772`,
  writing `AutoExposureMin/MaxBrightness` on a single unbound PPV), or `r.EyeAdaptationQuality 0`
  via `system.console_command`, read back and restored (`Docs/wiki-src/render.md:43-47`). Auto-
  exposure is a live scalar gain that re-balances between shots, so an unpinned burst silently
  cancels part of whatever is being measured. **Decide explicitly whether `capture_ortho_tiles`
  pins per call (a new parameter, nothing does this today) or requires a pinned scene and reports
  the pin state it observed.** Either way the whole grid must be one session — `level-review.md:21`
  records that editor exposure drifts session to session even when pinned within one.
- **Hold view mode across the burst too**, and verify it from the capture response rather than the
  setter: `B-set-view-mode-writes-only-active-projection-slot` (IN-REVIEW) —
  `editor.set_view_mode` returned success while every orthographic capture stayed on the previous
  mode, because `FEditorViewportClient` keeps separate Persp and Ortho slots.

## Extract `BitmapPaint.h`; do not write a third rasteriser

The CPU rasteriser `image.annotate` needs **already exists twice**, both file-local and unexported:

- `Private/Handlers/Drive/DriveSetOfMarkRenderer.cpp`, namespace `PinWrightDriveSomRender` (`:20`):
  `SetPixelClamped :56`, `FillRect :67`, `StrokeBox :80`, `DrawDigit :99`, `DrawLabel :124`, plus a
  digit-only `GDigitFont3x5` (`:41-53`). Its header exports only `DrawMarks`, `MarkColor`,
  `LabelColor`.
- `Private/Handlers/Render/AnnotatedCaptureHandler.cpp`, namespace `PinWrightAnnotatedCapture`
  (`:61`), no header at all: `Glyph3x5 :144`, `SetPixelClamped :169`, `FillRect :179`,
  `DrawLine :191` (Bresenham), `DrawLabel :219`, `Project :271`, `DrawSegment :291`, plus digit
  **and** A-Z fonts (`:95`, `:111`).

The duplication is acknowledged in-source — `AnnotatedCaptureHandler.cpp:22-27` says the Drive
primitives "are NOT reusable directly" because they are file-local, "so the equivalents are
re-implemented below and EXTENDED". A shared `BitmapPaint.h` must reconcile the delta: `StrokeBox`
exists only in Drive; `DrawLine`/`DrawSegment`/`Glyph3x5`/`GLetterFont` only in Annotated; the two
`DrawLabel` signatures differ (Drive takes `int32 Number` with badge/ink colours, Annotated takes
`FString` with a per-call scale and draws its own badge); Drive's glyph scale is a file constant
while Annotated passes it per call. **Preserve the uniquely-named-namespace guard** — both files
carry the same comment explaining it exists so unity-build TU merges cannot ODR-clash the font
tables, and that hazard gets worse, not better, when the symbols move into a header.

Do the extraction the way `F-animated-capture-verbs` did `SequencePlayheadUtils.h` /
`CameraShotPlanUtils.h`: reuse the existing namespace names so no call site changes, and land it
with both existing consumers moved over. A third copy is how one of them keeps a bug the others
fixed — `B-drive-setofmark-writes-jpeg` (OPEN: JPEG bytes labelled `image/png`) is one copy already
diverging.

## The new namespace needs a `maturity.json` entry, and nothing will tell you if you forget

`Docs/wiki-src/maturity.json` is a flat `"<namespace>": "<tier>"` map (`"render": "core"`,
`"camera": "experimental"`, `"pipeline": "internal"`), loaded by `WikiHandler.cpp:136-172`. A
registered namespace with no entry produces **only an editor-log warning**
(`WikiHandler.cpp:242-246`) and an unmarked wiki page; the loader treats a missing file as an empty
map (`:131-135`), and `Maturity.UnmappedNamespaceUnmarked`
(`Private/Tests/Infra/TestWikiHandler.cpp:1416`) deliberately asserts that an unmapped namespace
renders fine. So the `image` entry is easy to skip and nothing fails when it is skipped — add it in
the same change, and add a namespace page beside it.

## Relationship to `B-ortho-capture-culls-distant-foliage`

**`render.capture_ortho_tiles` is the prescribed fix for that open bug, and the source already says
so.** `PreviewViewportCaptureUtils.cpp:723-724`:

    // Name WHICH cap bound. "the scale was clipped" is not actionable; "the int32 bound in
    // the foliage path clipped it" tells a caller to narrow orthoWidth and tile instead.

with a second hint at `:980` (*"orthoWidth and capture in tiles, or pass viewDistanceScale
explicitly."*). The bug's mechanism is that a **lit** orthographic editor view pushes its culling
origin back along the view direction by roughly **2.1e6 cm** (`UE_OLD_WORLD_MAX`), so distance
culling drops geometry **regardless of camera height**, and the existing mitigation — a derived
`r.ViewDistanceScale` override, currently the `viewDistanceScale` parameter — is fighting a constant
offset ~50x larger than the primitives' true distance from the camera. Narrower frames attack the
cause directly: measured on the host map, a whole-map ortho retained **0.4%** of the Dire instanced
foliage and **53.1%** of the Radiant canopy that the same world region kept when captured at tile
scale, and the host project records "capture per tile rather than whole-map" as the workaround
already in use (`Docs/map/reference_tile_compare.md` section 4).

That bug stays OPEN and is **not** resolved by this ticket — its `ViewDistanceScale` derivation is
still under-scaled for orthographic views. But a caller who tiles stops depending on it. That is
why this is rated High rather than Medium: it is a missing verb family **and** the documented remedy
for an open High bug, so ranking it below that bug deadlocks both.

## History
- `#1-filed-with-constraints` `OPEN` reporter — Filed from the host project's reference-image
  comparison work. Gap: no chunked ortho capture, no world-coordinate annotation, no paired
  comparison, so callers hand-roll pixel maths outside the plugin
  (`Docs/scripts/compare/tile_compare.py`, 457 lines); eyeballed whole-map coordinates were off by
  ~1188 uu and chunking to native resolution improved accuracy ~10x. Four verbs proposed:
  `render.capture_ortho_tiles` under the existing `render.*`, and `image.tile` / `image.annotate` /
  `image.compare` in a **new** top-level `image.*` namespace. Load-bearing rule recorded: all
  coordinates in and out are world space and the verb owns the projection. Constraints recorded with
  citations: build the mapping from the reported `cameraRotation`/`orthoView`, not the request
  (`PreviewViewportCaptureUtils.cpp:407-439`, `RenderHandler.cpp:112-129`); no `orthoHeight`;
  `SetFixedViewportSize` before `ApplyCaptureCamera` (`:687-690`); constant capture size per session
  (`FViewport::GetHitProxy` `check()`, 66 actors + 125 emitters lost);
  `GTickUnsafeMethodNames` registration for anything reaching `CaptureEditorViewportToPng`
  (`SafePoint.cpp:45-144`, silent failure on a typo); `Ctx.StartJob` for the burst, with the caveat
  that it binds synchronously (`rpc-design.md:198`); one batch verb per `rpc-design.md:182-192`; one
  exposure pin and one view mode for the whole burst. Scope includes extracting a shared
  `BitmapPaint.h` from the two existing unexported rasterisers (`DriveSetOfMarkRenderer.cpp`,
  `AnnotatedCaptureHandler.cpp`) rather than writing a third, and adding the `image` entry to
  `Docs/wiki-src/maturity.json` (a missing entry is only a log warning, so it is easy to skip).
  Cross-referenced from `B-ortho-capture-culls-distant-foliage` as that bug's prescribed fix.
- `#2-additional-ortho-tile-review` `OPEN` reporter — Additional evidence: **Adversarial review A — the batch-capture gap is current, but the ticket overstates the image gap and relies on stale proof.** Actuality: `PARTIAL`. Framing: `render.capture_open_level` still exposes one capture only and there is no `render.capture_ortho_tiles`, so the systemic tile-burst need survives; however `render.capture_annotated` plus `ViewProjectionUtils::ProjectWorldToScreen` already provide world-coordinate annotation/projection for level captures, and the host harness is pointed at retired/missing input, so the claimed live ~10x result is not current acceptance evidence. Proposed fix: `INCOMPLETE`, because a batch verb with one shared projection/capture session, fixed size, effective-pose manifest, exposure/view-mode pin, and safe-point registration is systemic, while generic `image.annotate` duplicates the existing level path, `image.compare`/`BitmapPaint.h` are underspecified or premature, and B's culling defect remains independent. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\RenderHandler.cpp:385-487`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\AnnotatedCaptureHandler.cpp:303-325`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\ViewProjectionUtils.h:25-34,76-114`; `X:\src\unreal\EAContentExamples58\Docs\scripts\compare\tile_compare.py:66-73,126-164`; `X:\src\unreal\EAContentExamples58\Docs\map\reference_tile_compare.md:2-3`; `X:\src\unreal\.pinwright-board\B-ortho-capture-culls-distant-foliage.md:184-208`. Runtime: `NOT VERIFIED`; the current reference asset is absent and no fresh tile burst was run. Recommendation: `REFRAME`; lower to Medium unless a current end-to-end run demonstrates a release-blocking manual burden, repoint the reference, rerun bounded capture/compare, then scope implementation to a manifest-producing tile burst before considering external-image verbs/metrics.
- `#3-additional-external-image-scope` `OPEN` reporter — Additional evidence: **Adversarial review B — A is right that the old end-to-end proof is stale and the tile-burst portion remains missing, but existing level annotation does not close the external-image gap.** Actuality: `PARTIAL`. Framing: `render.capture_open_level` is one-shot; `render.capture_annotated` captures a new Level Editor PNG with fixed axes/grid/actor bounds, and `ViewProjectionUtils` is a private live-viewport projector, not a mapper for a caller-supplied reference image. The `image.*` need is real, but this ticket combines it with batching and an `image.compare` metric that should not encode reader judgment. Severity should be Medium: `tc_capture.py` already loops fixed-size captures, while the retired 7.33 reference has no current scale/asset. Proposed fix: `INCOMPLETE`, the capture manifest is systemic if it owns fixed resolution, effective pose/world boxes, exposure/view-mode state and safe-point registration; external image tile/crop/annotation should be a separate pure-data feature, and comparison should return pairings/manifests rather than heuristic agreement. Evidence: `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\RenderHandler.cpp:385-399,479-504`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\AnnotatedCaptureHandler.cpp:303-323,493-534`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Render\ViewProjectionUtils.h:10-35,96-113`; `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Dispatch\SafePoint.cpp:45-68`; `X:\src\unreal\EAContentExamples58\Docs\scripts\compare\tc_capture.py:5-17,68-83`; `X:\src\unreal\EAContentExamples58\Docs\map\reference_tile_compare.md:2-10`; `X:\src\unreal\EAContentExamples58\Docs\reference\dota2_720\README.md:123-143`. Runtime: `NOT VERIFIED`; no Unreal run or fresh current-reference comparison was performed. Recommendation: `REFRAME`; split capture-tiles from external-image operations, rerun a bounded current-reference registration gate, and keep the culling bug separate.
- `#4-verified-shipped-and-citation-fixed` `IN-REVIEW` developer — "Audited the whole ticket against the current tree and found the feature already landed, so the only code change here is the one stale citation the ticket itself flagged. **All four verbs exist and are registered:** `render.capture_ortho_tiles` (`Private/Handlers/Render/OrthoTileCaptureHandler.cpp:162`, landed `3576104c`), `image.tile` (`Private/Handlers/Image/ImageTileHandler.cpp:94`), `image.annotate` (`ImageAnnotateHandler.cpp:138`), `image.compare` (`ImageCompareHandler.cpp:62`), each with a fully declared `RPC_PARAM_*` surface. Every named constraint is satisfied: `render.capture_ortho_tiles` is in `GTickUnsafeMethodNames` (`Private/Dispatch/SafePoint.cpp:87`) with its own rationale comment; the world-coordinate maths is factored into an RHI-free, UObject-free header (`Private/Handlers/Render/TileGridUtils.h/.cpp`, 394+609 lines) whose header comment states the axis-mapping trap, the absent `orthoHeight` and the top-left continuous-pixel convention, and it is unit-tested directly by 8 ids in `Private/Tests/Render/TestTileGridUtils.cpp` including `PinWright.render.tile_grid.WorldPixelRoundTrip` and `.TileSeamLandsInExactlyOneTile`; the shared rasteriser was extracted to `Private/Handlers/Render/BitmapPaint.h/.cpp` and **both** former consumers moved over (`DriveSetOfMarkRenderer.cpp:6,46-73` and `AnnotatedCaptureHandler.cpp:44,96-122`) rather than a third copy being written, covered by 5 `PinWright.render.bitmap_paint.*` ids; `maturity.json` carries an `image` entry set to `experimental` and `Docs/wiki-src/image.md` is a full namespace page with one `###` section per verb (checked: no `##` orphaned below the first `###`, so nothing is silently unrendered), and `Docs/wiki-src/render.md:521` documents the burst. Response honesty holds: a tile that fails to plan, capture or encode aborts the burst with the tile **named** by `(col,row)` plus how many were already written and where (`OrthoTileCaptureHandler.cpp:600-635`), never omitted or zero-filled; blank tiles are flagged per tile and warned about in aggregate, and an all-blank burst is an `ERR_BLANK_CAPTURE` error carrying the whole measured response. 13 `PinWright.image.*` and 13 `render.capture_ortho_tiles`/`tile_grid` test ids exist. **Changed in this pass:** corrected the stale source citation in the `SafePoint.cpp` header comment that the ticket body asked for in the same change — `PumpViewport` is at `PreviewViewportCaptureUtils.cpp:106-124` (was cited `:63-81`) and `CaptureEditorViewportToPng` is defined at `:1589` with its three pump sites at `:2092`, `:2148`, `:2231` (was cited `:415-418`); comment only, no behaviour change, so no test or aspect-version bump. **Deliberately not shipped:** the scale bar named in the ticket's `image.annotate` row. `grid {spacingCm, labels}` already draws a world-aligned, edge-labelled scale reference and answers the same question more directly; a second, weaker spelling of it is the speculative surface the plugin's YAGNI rule refuses. Dispute it here if a reviewer disagrees. Also unchanged per the ticket's own instruction and the two reviewers on the sibling: `B-ortho-capture-culls-distant-foliage` keeps its `ViewDistanceScale` derivation defect and is not touched from here. Note for triage: reviews #2 and #3 read a different checkout (`unreal-fpv-dev`) and concluded the tile burst was missing; in `unreal-fpv-new` it is present and has been since `3576104c`. No build or suite run was performed — the orchestrator owns that."
