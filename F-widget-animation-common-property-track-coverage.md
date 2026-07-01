---
id: F-widget-animation-common-property-track-coverage
title: "Export common UMG property animation tracks instead of warning and dropping them"
status: DONE
severity: High
category: feature
tags: [widget, animation, dump, umg]
---

# Export common UMG property animation tracks instead of warning and dropping them

`widget.export_animations_json` and the asset dump animation aspect still drop common authored UMG animation tracks. The dump contains warnings, but the animation data is not exported, so agents cannot reason about color, byte/visibility-like, or vector property animation behavior from cached dumps.

**Workaround:** Use live `widget.get_animation_info` on a specific animation to inspect the authored tracks.

**Fix:** Add serializers for `MovieSceneColorTrack`, `MovieSceneByteTrack`, `MovieSceneDoubleVectorTrack`, and `MovieSceneFloatVectorTrack`, preserving target widget, property, section ranges, channels, keys, and interpolation data.

## History
- `#1-dump-audit-remaining-track-gaps` `OPEN` reporter — Read-only audit of `.editor-automation/asset-dumps` found 108 `widget_animations.json` files. 37 files still contain unsupported track warnings, 65 warnings total: `MovieSceneColorTrack=31`, `MovieSceneByteTrack=18`, `MovieSceneDoubleVectorTrack=14`, `MovieSceneFloatVectorTrack=2`. Live verification on `/App/App/UI/LobbyAndMenu/Elements/W_SelectAvatarButton` showed `widget.get_animation_info OnHover` reports real authored `MovieSceneColorTrack` and `MovieSceneByteTrack` sections, while `widget.export_animations_json` drops those bindings and emits 5 warnings. This is a follow-up to `F-widget-animation-full-fidelity`; that DONE ticket fixed material/2D-transform/text tracks, but common color/visibility/vector property tracks remain non-exportable.
- `#2-color-byte-vector-track-export` `IN-REVIEW` developer — Added byte/double channel helper families and four MakeXxxTrackObject/ValidateXxx/ApplyXxx triads (color, byte, floatVector, doubleVector) in `WidgetAnimationJsonSerializer.cpp`; wired export, validate, and apply dispatch chains. Added `FCommonPropertyAnimationFixture` + `CreateCommonPropertyAnimationFixture` to test utils. Tests: `EditorAutomationRpcGateway.widget.animation_json.RoundTripsCommonPropertyTracks` (round-trip asserts per-channel key counts + first-key values within float epsilon, byte `GetEnum()` == `StaticEnum<ESlateVisibility>()`, vector `ChannelsUsed` preserved) and `EditorAutomationRpcGateway.widget.animation_json.ExportEmitsNoCommonPropertyWarnings` (asserts `warningCount == 0`). Counterfactual: if any MakeXxxTrackObject writer is missing, `DocumentHasTrackType` for that type returns false; if the matching ApplyXxxTrackObject is missing, `FindTrackByClass<T>` on the imported MovieScene returns null.
- `#3-verify-color-byte-vector-export` `DONE` tester — Verified: `widget.export_animations_json` on `/App/App/UI/LobbyAndMenu/Elements/W_SelectAvatarButton` (the original audit asset) now returns `warningCount: 0` (was 5), with full color tracks (red/green/blue/alpha channels w/ keys) and byte Visibility track (`enumPath: /Script/UMG.ESlateVisibility`, key values 2 and 4) emitted on `BorderHovered`. Cross-checked `/App/App/UI/LobbyAndMenu/Elements/W_SimpleButton` — `doubleVector` tracks for `Size` exported with `channelsUsed: 2` and per-axis x/y key arrays, also `warningCount: 0`.
