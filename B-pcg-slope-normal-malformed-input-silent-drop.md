---
id: B-pcg-slope-normal-malformed-input-silent-drop
title: "pcg.add_slope_filter silently ignores a short normal array and treats missing object axes as zero, then returns success with no applied settings"
status: IN-REVIEW
severity: High
category: bug
tags: [pcg, slope-filter, vector, nested-validation, silent-drop, false-success, readback]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Malformed `normal` input is accepted but not honestly applied

`PCGAddSlopeFilter.cpp:22-59` implements a private vector reader. An array with fewer than three
items returns false; the caller at `:114-118` treats false exactly like an omitted parameter and
keeps the settings default. An object path is looser in the opposite direction: missing `X`, `Y`,
or `Z` fields default silently to zero and the helper returns true. The handler creates the node
before parsing, marks the graph dirty, and responds only with `nodeId` (`:96-138`).

Thus `normal:[0,1]` returns success while applying world-up rather than the request, and
`normal:{Z:1}` is silently interpreted as `{0,0,1}`. The top-level `array|object` schema cannot
catch either nested-shape error.

## What should happen

Strictly validate exactly three numeric array entries or three required numeric object axes before
creating the node. Reject malformed input with `INVALID_ARGUMENT`, and return the effective normal,
offset, and strength read from the settings object. Reuse the project's nested-key/vector validation
helper rather than maintaining a second permissive parser.

**Workaround:** send exactly three numeric array entries and inspect the node settings after creation.

## Fix

Root cause: the handler's local vector reader treated a malformed array as an omitted option and
defaulted missing object axes to zero, after the node had already been created. The shared
`PinWrightPCG::TryReadStrictVector` helper now accepts exactly three finite JSON numbers in an
array or all three numeric `X/Y/Z` axes (case-insensitive) in an object. The handler validates the
optional value before `AddNodeOfType`, returns the registered `INVALID_ARGUMENT` code on failure,
and includes measured `normal`, `offset`, and `strength` fields on success.

Changed files:

- `Plugins/PinWright/Source/PinWrightPCG/Private/Handlers/PCG/PCGAddSlopeFilter.cpp`
- `Plugins/PinWright/Source/PinWrightPCG/Private/Handlers/PCG/PCGHandlerHelpers.h`
- `Plugins/PinWright/Source/PinWrightPCG/Private/Tests/PCG/PCGTypedHelpersTests.cpp`
- `Plugins/PinWright/Docs/wiki-src/pcg.md`

Tests added: `PinWright.pcg.add_slope_filter.MalformedNormalArrayNoMutation` and
`PinWright.pcg.add_slope_filter.MalformedNormalObjectNoMutation`; each asserts
`INVALID_ARGUMENT`, unchanged node count, and preserved package dirtiness through
`InvokeHandlerWithCapture`.

Deliberately unchanged: omitted/null `normal` keeps the engine's world-up default, offset and
strength parsing remains as before, and the separate successful-mutation undo ticket.

## History
- `#1-source-pattern-scan` `OPEN` reporter — The optional-vector helper maps a short array to omission and missing object axes to zero, while the handler always returns success with only `nodeId`. Source-only; no RPC was run.
- `#2-strict-normal-validation` `IN-REVIEW` developer — Replaced permissive normal parsing with shared strict validation before node creation; added malformed array/object atomicity tests and measured settings readback. Static-only; Unreal was not run.
