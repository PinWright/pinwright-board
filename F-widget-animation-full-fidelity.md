---
id: F-widget-animation-full-fidelity
title: "Export full-fidelity UMG animation tracks"
status: DONE
severity: High
category: feature
tags: [widget, animation, umg, json, introspection]
---

# Export full-fidelity UMG animation tracks

The current widget animation MCP surface is not sufficient to fully inspect or round-trip real production UMG animations.

During montage export countdown investigation on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_TrackStartTimer`, MCP successfully reported that the `Full` animation exists and exported some float tracks via `widget.export_animations_json`, but it dropped or reduced the exact tracks needed to understand and preserve race-parity visuals.

Observed gaps from the session:

- `widget.export_animations_json` emitted warnings for unsupported tracks on the real countdown animation:
  - `MovieSceneWidgetMaterialTrack`
  - `MovieScene2DTransformTrack`
  - `MovieSceneTextTrack`
- the exported JSON only carried a subset of the authored animation state, even though those omitted tracks are exactly what drive the visible result:
  - ring material animation on `Fill`
  - glow material animation on `FillGlow`
  - transform animation on `RadialSB`
  - transform and text animation on `Time`
  - transform animation on `TextOvr`

This blocks exact visual analysis and exact export-safe reproduction of UMG animations. For the countdown widget specifically, the missing track types are load-bearing: without them MCP can tell that an animation exists, but cannot expose enough data to answer "what exactly makes the race widget look this way?" or to safely clone the animation into an export-safe copy.

Desired behavior:

- `widget.export_animations_json` should export the real authored track set for supported production UMG animations, not just float-property subsets.
- the schema should include at least:
  - `MovieSceneWidgetMaterialTrack`
  - `MovieScene2DTransformTrack`
  - `MovieSceneTextTrack`
- `widget.import_animations_json` should be able to round-trip those tracks back into a widget blueprint.
- warnings should remain only for genuinely unsupported track classes, not for core UMG countdown/title animation primitives.

This is a feature gap, not just an ergonomic issue. The current surface is enough for simple float opacity tracks, but not for exact asset-level animation inspection or editing on real widgets.

**Workaround:** Fall back to partial MCP inspection plus manual editor-side reasoning. This is enough to infer architecture, but not enough to preserve exact animation behavior confidently.

**Proposal:** Extend `WidgetAnimationJsonSerializer` and related handler/tests so widget animation export/import supports the common UMG track classes that appear in authored production widgets, starting with material, 2D transform, and text tracks.

## History
- `#1-initial-repro` `OPEN` reporter — No existing board match found after checking 132 board entries. While investigating `W_HUD_TrackStartTimer.Full`, `widget.export_animations_json` reported unsupported `MovieSceneWidgetMaterialTrack`, `MovieScene2DTransformTrack`, and `MovieSceneTextTrack`, which blocked exact visual introspection of the race countdown animation.
- `#2-core-umg-track-roundtrip` `IN-REVIEW` developer — Changed the widget animation JSON serializer/importer, handler descriptions, docs, and tests to round-trip `widgetMaterial`, `2dTransform`, and `text` tracks while preserving float support; added `FWidgetAnimationJsonRoundTripsCoreUmgTracksTest` to cover native track export/import and key preservation.
- `#3-verified-real-widget-tracks` `DONE` tester — Verified live on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_TrackStartTimer.W_HUD_TrackStartTimer`: `widget.export_animations_json includeEventMetadata:true` now exports `widgetMaterial` tracks for `Fill` and `FillGlow`, `2dTransform` tracks for `RadialSB`, `Time`, and `TextOvr`, and a `text` track for `Time`; `warningCount` returned `0`.
