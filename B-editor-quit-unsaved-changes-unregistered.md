---
id: B-editor-quit-unsaved-changes-unregistered
title: "editor.quit emits UNSAVED_CHANGES without a registered ERR_UNSAVED_CHANGES constant (contract test red)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [editor, editor-quit, error-codes, registry, contract-test]
---

# editor.quit emits UNSAVED_CHANGES without a registered ERR_UNSAVED_CHANGES constant

`editor.quit` refuses on dirty packages with `Ctx.SendError(TEXT("UNSAVED_CHANGES"), ...)`
(`Handlers/Editor/EditorQuitHandler.cpp:116`), but `Handlers/ErrorCodes.h` never declared a
matching `ERR_UNSAVED_CHANGES` constant. The contract test
`EditorAutomationRpcGateway.core.error_codes.AllEmittedCodesAreRegistered`
(`Tests/Core/TestErrorCodeRegistry.cpp:87`) asserts that every error code emitted via
`SendError(TEXT("..."))` has a corresponding `ERR_*` entry in the central registry, so this
gap made the suite fail (2888/2889 pass, 1 fail):

> Error code 'UNSAVED_CHANGES' is emitted via SendError(TEXT("...")) but has no ERR_UNSAVED_CHANGES

## Root cause
The `editor.quit` RPC was added emitting the `UNSAVED_CHANGES` literal without the companion
registry entry. 152+ other handler callsites use raw `SendError` literals that ARE registered
centrally in `ErrorCodes.h`; this one was missed. The handler's other code (`SAVE_FAILED`) was
already registered.

## Fix
Register the constant in `ErrorCodes.h` in alphabetical order (between `ERR_UNKNOWN_VIEW_MODE`
and `ERR_UNSUPPORTED_ARGUMENT`):
`inline constexpr TCHAR ERR_UNSAVED_CHANGES[] = TEXT("UNSAVED_CHANGES");`

## History
- `#1-initial-repro` `OPEN` reporter — Surfaced deterministically by the mcp-test-workflow loop while verifying `B-actor-find-by-class-short-name-fails`: the `AllEmittedCodesAreRegistered` contract test (`Tests/Core/TestErrorCodeRegistry.cpp:87`) failed with "Error code 'UNSAVED_CHANGES' is emitted via SendError(TEXT(\"...\")) but has no ERR_UNSAVED_CHANGES". Of 2889 tests, 2888 passed and this one failed. Root cause: `EditorQuitHandler.cpp:116` emits the literal `TEXT("UNSAVED_CHANGES")` but `ErrorCodes.h` has no matching `ERR_*` registration.
- `#2-register-constant` `IN-REVIEW` developer — Registered `ERR_UNSAVED_CHANGES = TEXT("UNSAVED_CHANGES")` in `Handlers/ErrorCodes.h` (alphabetical, between `ERR_UNKNOWN_VIEW_MODE` and `ERR_UNSUPPORTED_ARGUMENT`). Test-loop confirmed the suite then runs fully green (2889/2889). Applied directly to the canonical source.
