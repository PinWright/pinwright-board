---
id: B-widget-animation-error-mutation
title: "Widget animation verbs mutate the MovieScene before returning WIDGET_NOT_FOUND, BINDING_NOT_FOUND, or NOT_SUPPORTED"
status: IN-REVIEW
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

## Fix

The defect was confirmed: request validation and several fallible object-creation steps ran after
`EnsureAnimationMovieScene`, binding creation, or playback-range expansion. The track and keyframe
handlers now resolve the target widget, validate the reflected float property and numeric inputs,
and stage new tracks/sections plus any new possessable/binding off-asset before the transaction
commits. Existing bindings are resolved before `EnsureAnimationMovieScene`; after it runs, track
publication has no recoverable failure branch. The speed handler returns `NOT_SUPPORTED` without
calling the mutating MovieScene repair helper.

Files changed:
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\UI\WidgetAnimationHandler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\UI\WidgetAnimationTestHooks.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Widget\TestWidgetAnimationValidationAtomicity.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\docs\wiki-src\widget.md`

Tests added:
- `PinWright.widget.animation.RejectedRequestsPreserveMovieScene`
- `PinWright.widget.animation.AttachPreflightFailureIsAtomic`
- `PinWright.widget.animation.PropertyValidationIsAtomic`

Deliberately unchanged: `widget.set_animation_loop` already rejects without touching its
MovieScene, and no live editor/build/test run was performed under this source-only worker brief.

## History
- `#1-source-scan-error-mutation` `OPEN` reporter — Source-only review followed all three error paths through the mutating `EnsureAnimationMovieScene` helper and the keyframe playback/binding writes. No rollback or cleanup is present before the cited error responses. No build, test, editor, MCP call, or plugin edit was performed.
- `#2-stage-before-animation-mutation` `IN-REVIEW` developer — Confirmed the mutation ordering defect. Widget/property/numeric validation and off-asset track/section staging now precede MovieScene mutation; rejected speed calls do not repair the MovieScene, and behavioral tests snapshot pointer, name, range, bindings, and tracks across failures. Source-only verification; no Unreal run was performed.
- `#3-preflight-track-publication` `IN-REVIEW` developer — Replaced the ineffective attach-failure cleanup with pre-commit possessable/binding staging and existing-binding resolution, leaving no recoverable failure after MovieScene repair. Expanded behavioral snapshots cover package dirtiness, possessables, sections, keys, and desired-name occupants; forced final-preflight failures exercise both track and keyframe paths. Source-only verification; no Unreal run was performed.
- `#4-remove-test-request-param` `IN-REVIEW` developer — The full-suite declared-parameter ratchet correctly rejected the hidden JSON injection key. Moved forced attach-preflight failure control to a dev-only inline hook with scoped reset, so neither animation handler reads undeclared request data. Source-only follow-up; no build or test run was performed.
- `#5-force-failure-before-allocation` `IN-REVIEW` developer — Scoped-run callstacks traced the zero-ensure violation to the test's abstract `UObject` name-collision fixture. Removed that invalid allocation and moved both forced attach failures immediately after the no-track predicate, before transient track, section, possessable, or binding staging; full animation snapshots remain asserted. Source-only follow-up; no new build or test run was performed.
