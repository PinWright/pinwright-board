---
id: B-input-fixtures-deleted-early
title: "Input smoke-test fixtures are deleted before the handlers under test use them"
status: OPEN
severity: Medium
category: bug
tags: [input, tests, fixture-lifetime, false-green, cleanup]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# Input smoke-test fixtures are deleted before use

## What happens

`EnsureInputTestAssets()` creates `/Game/Input/IA_Jump` and
`/Game/Input/IMC_Default`, then immediately passes both paths to
`CleanupTestAsset()` before returning
(`Source/PinWright/Private/Tests/Gameplay/TestInputHandlers.cpp:17-30`). The
`input.add_mapping`, `input.remove_mapping`, and `input.get_input_info` smoke
tests call that helper and only afterwards invoke their handlers with those paths
(`TestInputHandlers.cpp:84-135`).

The tests assert that dispatch found and invoked a handler, not that the handler
successfully loaded and used the fixture. They can therefore pass through an
asset-not-found response without exercising the operation they name.

## Why it matters

This is false-green coverage for three Input routes. A production regression in
asset loading, mapping mutation, or readback can survive while the smoke suite stays
green. Severity is Medium: the hidden failure affects test coverage rather than the
production handler itself, but the current tests do not provide a useful fallback
signal.

## What should happen

Keep both assets alive until the invoking test finishes, move cleanup into scoped
teardown that runs on every exit, and assert the captured handler response and the
expected mapping/readback state rather than dispatch alone.

## Workaround

Run the handlers against pre-existing assets or use a dedicated fixture that cleans
up only after response and state assertions complete.

## Related

- `F-add-mapping-no-modifiers-ue58-mappings-moved` — wave-6 ticket whose
  implementation review exposed the fixture-lifetime defect.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed that `EnsureInputTestAssets()` deletes both assets at `TestInputHandlers.cpp:29-30`, before the three consumers at `:84-135` invoke their handlers. No build, test, editor, or MCP call was run. Severity Medium because this is narrow false-green test coverage, not a demonstrated production failure.
