---
id: E-tick-unsafe-hazard-derivation-ratchet
title: "Item 3 only: migrate the remaining tick-unsafe table entries to REGISTER_RPC_HANDLER_TICK_UNSAFE and update the string-searching tests"
status: WONTFIX
severity: Medium
category: ergonomic
tags: [safepoint, dispatch, tick-gate, handler-registration, coverage-gap, test-infrastructure, source-lint, ratchet]
encounters: 1
lastSeen: 2026-10-02T00:00:00Z
---

# Follow-up split out of E-tick-unsafe-declared-at-registration

`E-tick-unsafe-declared-at-registration` #2 landed the declaration half: `REGISTER_RPC_HANDLER_TICK_UNSAFE`
sets `FHandlerRegistration::bTickUnsafe`, and `PinWrightSafePoint::IsTickUnsafeMethod()` /
`GetTickUnsafeMethods()` read `table || flag`. That relocates the omission from `Dispatch/SafePoint.cpp`
to the handler; it does not make anyone set the flag. Three pieces remain.

**1. The derivation ratchet (the part that closes the hole).** Scan plugin source for the hazard calls the
table's families are written around — `CompileBlueprintWithDiagnostics` / `FKismetEditorUtilities::CompileBlueprint`
(L/K), `CollectGarbage` / `EditorDestroyWorld` / `GEditor->NewMap` (A), `FlushRenderingCommands` /
`PumpViewport` / `Viewport->Draw` (B, H.4, I), `UPackageTools::ReloadPackages` (J) — per `REGISTER_RPC_*`
body plus one call hop into helpers (the resolver `Tests/Infra/TestDeclaredParamCoverage.cpp` already has),
and fail on any registered verb that reaches one without `IsTickUnsafeMethod()` true. Seed as a recorded
baseline (fail on new, warn on stale), exempt verbs gated in-handler (`RunAtSafePoint` / `DeferJobToSafePoint`,
already pinned by `PinWright.infra.tick_safety.HandlerHazardsStayGated`) and state the cross-dispatch limit
(`FRpcDispatcher::DispatchMethod` bypasses the gate) rather than pretending a flag covers it. The current
`HandlerHazardsStayGated` only pins the compile chokepoint's call count and a hand list of 60 verbs, so a
61st verb that calls the chokepoint is not caught.

**2. Teach the exact-string source scanners the new macro BEFORE any verb migrates.** These search the
literal `REGISTER_RPC_HANDLER(` / `REGISTER_RPC_HANDLER(\"verb\"` and would silently stop seeing a verb
whose macro becomes `REGISTER_RPC_HANDLER_TICK_UNSAFE(`: `Tests/Infra/TestParamTypeGate.cpp`
(`TryGetRegisteredMethod`), `Tests/Render/TestCaptureVerbParameterParity.cpp` (`RegisterPattern`),
`Tests/Core/TestNoParamHandlersReadNoArgs.cpp`, `Tests/Infra/TestHandlerTickSafetyRatchet.cpp`
(`HandlerBlockReportsCompile`), and per-verb needles in `Tests/Assets/TestAssetImportSafePoint.cpp`,
`TestGenerateLodsPersistenceAndCompile.cpp`, `TestAssetPieSafeResolution.cpp`, `Tests/Niagara/TestEffectStepAndCapture.cpp`.
`REGISTER_RPC_MUTATING_HANDLER` already has the same blind spot today. `TestDeclaredParamCoverage` steps over
the macro tail and is fine.

**3. Migrate the legacy table.** For each `GTickUnsafeMethodNames` entry: switch that verb's macro to
`REGISTER_RPC_HANDLER_TICK_UNSAFE` and delete the table line in the same change; keep the family evidence
comments next to the handler. Then add a contract that the table only shrinks (or delete it). Deferred from
#2 because ~36 agents were adding table entries concurrently. Hand-written tick-safety prose in
`docs/wiki-src/{audio.music,blueprint,editor,level,python,system,unattended}.md` could then be derived from
the flag.

## History
- `#1-split-from-registration-flag` `OPEN` developer — Split from `E-tick-unsafe-declared-at-registration` #2 when the registration flag landed; the three items above are what that ticket's #1 called the half that actually closes the hole, plus the migration work it deferred.
- `#2-merged-into-registration-ticket` `IN-REVIEW` developer — Duplicate: items 1 (the ratchet) and 2 (scanners accept every registration spelling) are implemented under `E-tick-unsafe-declared-at-registration` #3, item 3 (table migration) is partially done there (two entries) and its remainder is tracked in that History. Close together with that ticket; no separate verification needed.
- `#3-rescoped-to-table-migration` `OPEN` developer — Re-opened and re-scoped on review: the #2 duplicate close was wrong, because item 3 has not landed. Items 1 (the ratchet `PinWright.infra.tick_safety.HazardReachingVerbsAreGated`, `Tests/Infra/TestTickUnsafeHazardDerivation.cpp`) and 2 (the scanners go through `FindNextRpcRegistration` / `RpcRegistrationMethod` / `FindRpcRegistrationBlock` in `Tests/TestUtils.h`) were delivered in `E-tick-unsafe-declared-at-registration` #3 and are verified there. This ticket now covers item 3 only: migrate the ~140 remaining `GTickUnsafeMethodNames` entries in `Dispatch/SafePoint.cpp` to the macro (switch the macro and delete the table line in the same change), add a table-only-shrinks contract (or delete the table), and teach the per-verb needle tests that still search literal registration strings (`TestAssetPieSafeResolution`, `TestVFXHandlers`, `TestEffectStepAndCapture`). Already migrated: `editor.jump_to_bookmark`, `audio.authoring.set_sound_wave_gain`, `render.capture_actor_preview`. If maintainers judge the migration not worth it, `WONTFIX` is acceptable: the table-or-flag union plus the ratchet already close the omission hole.
- `#4-wontfix-yagni` `WONTFIX` developer — Pure refactor with no user-visible defect left: the table-or-flag union plus the `HazardReachingVerbsAreGated` ratchet (delivered under `E-tick-unsafe-declared-at-registration` `#3`) already close the omission hole, so migrating the ~140 remaining `GTickUnsafeMethodNames` entries changes spelling, not behaviour; `#3` names WONTFIX as acceptable. Reopen if the table and the macro are ever found to disagree or a new tick-unsafe verb slips past the ratchet.
