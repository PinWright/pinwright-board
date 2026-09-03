---
id: B-widget-animation-error-mutation
title: "Widget animation verbs mutate the MovieScene before returning WIDGET_NOT_FOUND, BINDING_NOT_FOUND, or NOT_SUPPORTED"
status: OPEN
severity: High
category: bug
tags: [widget, animation, moviescene, transactions, rollback, partial-mutation]
encounters: 1
lastSeen: 2026-09-03T23:27:21+03:00
---

# Widget animation error paths leave MovieScene changes behind

## What happens

`EnsureAnimationMovieScene` is a mutator: it can create and assign a MovieScene or
rename the existing one (`WidgetAuthoringUtils.cpp:710-741`). Several callers run
it before they know the request can succeed:

- `widget.add_animation_track` calls it before resolving `widgetName`, then returns
  `WIDGET_NOT_FOUND` (`WidgetAnimationHandler.cpp:357-378`).
- `widget.add_animation_keyframe` calls it before its transaction, expands the
  playback range at `:495-503`, and only then validates the widget and binding.
  Later binding, track and section failures at `:515-615` also have no rollback.
- `widget.set_animation_speed` calls it at `:713` and then always returns
  `NOT_SUPPORTED` at `:720-722`.

## Why it matters

A request reported as rejected can create or rename a serialized MovieScene,
change its playback range, or add bindings/tracks. That makes retries and later
saves operate on hidden partial state. Severity is High for a failed authoring
request that mutates the asset in memory.

## What should happen

Resolve all names and reject unsupported verbs before calling the mutating helper.
For paths with unavoidable late failures, include `EnsureAnimationMovieScene` and
all subsequent writes in a snapshot-backed transaction, reverse created subobjects
and bindings, then compare the restored animation state before sending the error.

## Workaround

Resolve widget and animation names with `widget.describe` and
`widget.get_animation_info` first. Do not call `widget.set_animation_speed`; reload
or undo the widget blueprint after any animation-authoring error.

## Related

- `B-widget-animation-moviescene-unbound` — introduced the MovieScene repair helper,
  but does not cover mutations made by rejected requests.
- `E-widget-anim-loop-speed-phantom-authoring-verbs` — covers the misleading verbs,
  not the `set_animation_speed` side effect.

## History
- `#1-source-scan-error-mutation` `OPEN` reporter — Source-only review followed all three error paths through the mutating `EnsureAnimationMovieScene` helper and the keyframe playback/binding writes. No rollback or cleanup is present before the cited error responses. No build, test, editor, MCP call, or plugin edit was performed.
