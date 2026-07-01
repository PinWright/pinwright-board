---
id: B-widget-animations-aspect-emits-empty-on-bps-with-anim
title: "widget_animations.json is empty even when widget BP has UMG animations"
status: DONE
severity: Medium
category: bug
tags: [widget-animations, sidecar]
---

# widget_animations.json is empty even when widget BP has UMG animations

Distinct from [B-asset-dump-widget-animation-aspect](B-asset-dump-widget-animation-aspect.md) (existence/registration) — this is about CONTENT: even when the sidecar IS generated, its `animations` array is empty for widgets that clearly have visual UMG animations at runtime.

Out of all sampled widget_animations.json files in App/Game/Engine, **only one** contains actual keyframe data (Fadein_Widget). Most contain `"animations": []` despite the source widget BP having opacity / transform / color animations authored.

## Samples

- Empty: `App/PathTracer/Blueprints/Info_Widget/WB_Info_Widget/widget_animations.json`
- Empty: `App/Blueprints/UI/W_Drone/widget_animations.json`
- Has data: `App/HELIOS/Global/UI/Widgets/Fadein_Widget/widget_animations.json` (only one observed)

## MCP verification

`call("widget.export_xml", {assetPath})` returns animation track info that doesn't surface in widget_animations.json.

## Fix sketch

`WidgetAnimationJsonSerializer.cpp::ExportAnimations()` — verify the animation enumeration path (likely `UWidgetBlueprint::Animations` or equivalent), confirm it walks the source widget BP's animations correctly. The single working case (Fadein_Widget) suggests the emitter works under some conditions but not others — diff the working case vs a failing case to find the discriminating factor.

## Implementation note

Treated as stale/narrowed on 2026-05-22 after source-level triage. `WidgetAnimationJsonSerializer.cpp::ExportAnimations()` already walks `UWidgetBlueprint::Animations`, `AssetDumpHandler.cpp` unconditionally writes `widget_animations.json` for widget blueprints, `TestWidgetAnimationAssetDump.cpp` covers a production `Full` animation with bindings/tracks/event metadata, and `TestWidgetAnimationAssetDumpEmpty.cpp` confirms `animations: []` is valid for a true zero-animation widget. No serializer change was made because no named sample with non-empty `UWidgetBlueprint::Animations` and empty export was identified in source.

## History
- `#1-empty-anim-arrays` `OPEN` reporter — only 1 widget out of all samples has actual animation data emitted; most widget BPs with visible runtime animations have empty arrays.
- `#2-verify-no-false-positives` `DONE` tester — Verified: `widget.get_animation_info` on the two "failing" samples returns `animationCount: 0` (WB_Info_Widget, W_Drone), confirming the source BPs have no authored animations; the positive control Fadein_Widget returns `animationCount: 1` with a `FadeIn_Anim` entry that matches the populated sidecar. Fresh `asset.dump` of WB_Info_Widget regenerated `widget_animations.json` with `animations: []`, which is the correct emission. Reporter's "visible runtime animations" claim does not match the persisted `UWidgetBlueprint::Animations` for those samples.
