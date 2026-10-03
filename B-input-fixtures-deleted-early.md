---
id: B-input-fixtures-deleted-early
title: "Input smoke-test fixtures are deleted before the handlers under test use them"
status: DONE
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
- `#2-scoped-fixtures-and-state-asserts` `IN-REVIEW` developer — Still reproducible: `EnsureInputTestAssets()` deleted both assets before the three consumers ran, and the consumers asserted dispatch only. Rewrote `Source/PinWright/Private/Tests/Gameplay/TestInputHandlers.cpp`: the helper is replaced by `TestInputHandlersHelpers::FInputFixture`, an RAII pair (pre-cleans stale residue, creates action + context through the production create verbs with asserted success and a load check, deletes both in its destructor on every exit). Each consumer gets its own pair under the scratch root `/Game/PinWrightTests/Input` instead of host `/Game/Input`. `add_mapping` now asserts success, the response key/actionPath, and that the in-memory context package is clean (saved) and holds one mapping with key SpaceBar whose action is the fixture action. Nothing is reloaded from disk; the dirty-flag check is the evidence that the forced save ran. `remove_mapping` adds a mapping first, then asserts success, `keysRemoved == 1`, `removedKeys` holds SpaceBar, and the in-memory context has 0 mappings and a clean (saved) package. `get_input_info` reads back both assets and asserts `type`, plus `mappingCount == 1` and the SpaceBar mapping on the context. The same pattern in sibling tests in that file is fixed too: both `create_*` tests now assert success and a loadable asset of the right class, with `ON_SCOPE_EXIT` cleanup, and `EmitsDocumentedConsumeKey` asserts fixture creation and handler success instead of skipping its key assertions under `if (Capture.bSuccess)`. All fixtures moved to the scratch root. `Tests/Input/TestInputMappingModifiers.cpp` already used scoped scratch-root fixtures and needed no change. Test ids are unchanged: `PinWright.input.create_input_action.ValidParamsNoCrash`, `PinWright.input.create_input_mapping_context.ValidParamsNoCrash`, `PinWright.input.add_mapping.ValidParamsNoCrash`, `PinWright.input.remove_mapping.ValidParamsNoCrash`, `PinWright.input.get_input_info.ValidParamsNoCrash`, `PinWright.input.get_input_info.EmitsDocumentedConsumeKey` (filter `PinWright.input.`). Failure direction: restoring the early cleanup makes the fixture load checks and the handler-success asserts fail. fastcheck is OK; check_test_ids and check_test_skips are CLEAN. The tests have not been run in an editor yet.
- `#3-verified-linux` `DONE` tester — Fix commit 44d26cb7. Passed non-skipped in run3/full: `PinWright.input.create_input_action.ValidParamsNoCrash`, `PinWright.input.create_input_mapping_context.ValidParamsNoCrash`, `PinWright.input.add_mapping.ValidParamsNoCrash`, `PinWright.input.remove_mapping.ValidParamsNoCrash`, `PinWright.input.get_input_info.ValidParamsNoCrash` and `PinWright.input.get_input_info.EmitsDocumentedConsumeKey`. Acceptance: the fixtures are an RAII pair alive until the test ends, with cleanup on every exit, under /Game/PinWrightTests/Input. Each consumer now asserts handler success and state, not dispatch alone: add_mapping asserts one SpaceBar mapping to the fixture action and a clean, saved package; remove_mapping asserts `keysRemoved == 1` and zero mappings; get_input_info asserts `type`, `mappingCount` and the SpaceBar mapping. All three "What should happen" items are met. Limit: the reverted-fixture counterfactual was not run.
