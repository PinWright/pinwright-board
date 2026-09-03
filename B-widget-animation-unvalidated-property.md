---
id: B-widget-animation-unvalidated-property
title: "Widget animation track and keyframe verbs accept any property name and report a bound float track even when the widget has no compatible property"
status: OPEN
severity: High
category: bug
tags: [widget, animation, moviescene, property-validation, false-success]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Widget animation authoring never validates the property it claims to animate

## What happens

`widget.add_animation_track` reads arbitrary `propertyName`, creates a
`UMovieSceneFloatTrack`, and copies the string into its property path
(`WidgetAnimationHandler.cpp:335`, `:399-425`). It never resolves an `FProperty` on
the target widget or checks that the property is float-compatible. The handler then
returns `applied:true` and `changesApplied:1` (`:430-444`).

`widget.add_animation_keyframe` repeats the same behavior when it cannot find an
existing float track (`:567-619`) and returns success with the requested property
name (`:624-641`). A missing property such as `DefinitelyMissing`, or a non-float
property, therefore produces a structurally present but inert track/key.

## Why it matters

The normal success response says the animation was authored while runtime playback
cannot drive the requested property. This is a High-severity silent false success.

## What should happen

Resolve the complete property path on the selected widget before mutation. Refuse
missing or unsupported properties with the resolved widget class and compatible
types, create the MovieScene track class that matches the property, and read back
the stored property path and track type before reporting success.

## Workaround

Use a known float property such as `RenderOpacity`, then inspect the animation in
Sequencer or via animation export before relying on it.

## Related

- `B-widget-animation-moviescene-unbound` — binding/MovieScene identity, not
  property existence or type.

## History
- `#1-source-scan-unvalidated-property` `OPEN` reporter — Source-only scan confirmed both handlers convert every supplied property name into a float track and echo it as applied without any `FProperty` lookup or type gate. No build, test, editor, MCP call, or plugin edit was performed.
