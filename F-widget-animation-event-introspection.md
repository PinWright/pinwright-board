---
id: F-widget-animation-event-introspection
title: "Expose UMG animation event tracks and finish bindings"
status: DONE
severity: High
category: feature
tags: [widget, animation, events, delegate, sequencer]
---

# Expose UMG animation event tracks and finish bindings

The current widget animation MCP surface does not expose enough information to determine whether a UMG animation is safe to scrub in tooling without triggering gameplay-side effects.

During montage export countdown investigation on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_TrackStartTimer`, the session needed to answer a very specific question:

"Does the `Full` animation itself carry event-driven behavior, or is all unsafe behavior outside the animation in `BP_OnActivated()`?"

MCP gave partial evidence:

- `widget.get_animation_info` listed bindings and track classes for `Full`
- `widget.export_animations_json` showed visual tracks and warnings
- asset/BP dumps showed a `SequenceEvent()` custom event that calls `GameProgressSubsystem.CompleteCountdown()`

But MCP could not answer the key ownership question directly:

- whether `Full` contains a `MovieSceneEventTrack`
- whether the widget blueprint has animation-finished delegate bindings from `Full` to `SequenceEvent` or another event
- whether a given custom event is animation-driven, delegate-driven, or only called from graph logic

That gap makes it hard to safely automate export-safe animation scrubbing. In this session, exact visual parity required the real authored animation, but the missing introspection meant MCP could not conclusively prove whether scrubbing `Full` would re-trigger `CompleteCountdown()` or other race-state behavior.

Desired behavior:

- `widget.get_animation_info` should optionally report:
  - animation event tracks (`MovieSceneEventTrack`) and their keyed events
  - animation-finished / animation-started delegate bindings
  - target function/custom event names bound to a given animation
- `widget.export_animations_json` should optionally include event-track metadata, even if import support lands later.
- MCP should allow an agent to answer:
  - "what functions or custom events can this animation trigger?"
  - "is this custom event referenced by animation timeline events or by animation delegate bindings?"

This is needed for safe tooling work on real UMG assets where presentation and gameplay are mixed in one widget.

**Workaround:** Combine BP graph dumps, partial animation export, and engine knowledge to infer risk manually. That is slow and still leaves ambiguity.

**Proposal:** Extend widget animation inspection to include event-track and delegate-binding metadata, either by enhancing `widget.get_animation_info` directly or by adding a dedicated animation event inspection RPC.

## History
- `#1-initial-repro` `OPEN` reporter — No existing board match found after checking 132 board entries. While investigating `W_HUD_TrackStartTimer.Full`, MCP could list visual tracks and separately show a `SequenceEvent()` custom event that calls `CompleteCountdown()`, but it could not directly reveal whether the animation contains event tracks or animation-finished delegate bindings that trigger that event.
- `#2-event-metadata-introspection` `IN-REVIEW` developer — Added read-only widget animation event metadata for `widget.get_animation_info` via `includeEvents` and `widget.export_animations_json` via `includeEventMetadata`, using a dedicated event/delegate introspection helper plus wiki docs. Added `FWidgetAnimationEventMetadataIntrospectionTest` to cover root event-track function metadata and generated animation-finished delegate metadata.
- `#3-verified-sequenceevent-owner` `DONE` tester — Verified live on `/App/App/UI/LobbyAndMenu/HUD/W_HUD_TrackStartTimer.W_HUD_TrackStartTimer`: `widget.get_animation_info animationName:"Full" includeEvents:true` reports one `MovieSceneEventTrack` at `time:4.0` targeting `SequenceEvent__ENTRYPOINTW_HUD_TrackStartTimer` with endpoint `SequenceEvent`, `delegateBindings` is empty, and `triggeredFunctions` includes only that SequenceEvent entrypoint. This answers the ownership question directly.
