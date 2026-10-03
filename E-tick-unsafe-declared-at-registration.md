---
id: E-tick-unsafe-declared-at-registration
title: "Tick-unsafety lives in a hand-maintained name table in a file no handler author opens, and the only test on it catches typos rather than omissions — every gap so far was found by an editor death, not by the suite"
status: DONE
severity: Medium
category: ergonomic
tags: [safepoint, dispatch, tick-gate, handler-registration, rpc-dispatcher, coverage-gap, test-infrastructure, maintainability, wiki]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# The table is correct about what it lists and silent about what it omits

`GTickUnsafeMethodNames` (`Dispatch/SafePoint.cpp:56-521`) is a hand-written array of method-name
string literals in a file that no handler author edits while writing a handler. A verb is gated
only if somebody remembered to come here and add a line.

Track record of that arrangement, from the file's own comments and the board:

- Four level/lighting verbs shipped ungated behind `level.load`'s hand-written gate —
  `Dispatch/SafePoint.h:341-348` says so in as many words ("precisely what let four level verbs
  and the entire capture family ship ungated").
- The whole capture family shipped ungated (same comment, family B).
- `material.authoring.compile_material` shipped ungated after reaching family B's mechanism
  through the material door — found by two editor kills 90 seconds apart, not by a test
  (`B-compile-material-not-tick-gated`, still IN-REVIEW).
- 59 Blueprint-mutating verbs are ungated right now
  (`B-blueprint-mutators-compile-ungated-tick-unsafe`), found by reading the known-gap note the
  previous fix left behind.

The only automated check is
`PinWright.core.safe_point.TickUnsafeMethodsAreRegistered` (`Tests/World/TestSafePointGate.cpp:169-198`).
It walks `GetTickUnsafeMethods()` and asserts each name resolves to a registered handler — a
guard against a typo silently gating nothing. It cannot detect the opposite and more expensive
error: a verb that reaches a hazard and is not listed. Every gap above is that error, and the
suite was green through all of them.

## Two changes, and only one of them closes the hole

**1. Declare at the handler (the ergonomic half).** `FHandlerRegistration`
(`Handlers/HandlerRegistration.h:14-20`) carries `MethodName`, `Category`, `Summary`, `Params`,
`Func` and no flags field. Add a `bTickUnsafe` bit and a
`REGISTER_RPC_HANDLER_TICK_UNSAFE(...)` variant next to the existing macro
(`HandlerRegistration.h:39-48`), so the declaration sits on the handler an author is already
editing and a misspelling is a compile error instead of a silent no-gate.

Migration cost is small and almost entirely mechanical, because the read API can stay exactly as
it is: keep `PinWrightSafePoint::IsTickUnsafeMethod()` and `GetTickUnsafeMethods()`
(`Dispatch/SafePoint.h:363`, `:366`) and populate them from
`FAutoRegisterHandler::GetPendingRegistrations()`, which `FRpcDispatcher` already reads the same
way at `RpcDispatcher.cpp:488` — same module, same private headers, no dependency inversion. Then
nothing else changes: the single production consumer (`RpcDispatcher.cpp:696-697`), the
`SetExtraTickUnsafeMethodForTests` override, and the eight test files that assert against the
table (`Tests/World/TestSafePointGate.cpp`, `Tests/Blueprint/TestBlueprintReinstancingGuard.cpp:363`,
`Tests/Media/TestAudioAnalysisHandler.cpp:1187`, `TestAudioMusicHandler.cpp:1102`,
`TestAudioSynthAudition.cpp:304`, `TestAudioSynthGenerate.cpp:1027`,
`TestSoundWavePcmHandler.cpp:501`) all keep working unmodified. Nothing outside the module reads
the table; the wiki generator does not read it either — the tick-unsafety prose in
`docs/wiki-src/{audio.music,blueprint,editor,level,python,system,unattended}.md` is hand-written
per verb, which is a second hand-maintained copy of the same fact and would become derivable from
the flag.

What the flag does **not** do is make anyone remember to set it. It moves the omission from one
file to another.

**2. Derive it, and ratchet (the half that actually closes the hole).** The plugin already lints
its own source from automation tests, with a helper-following call-hop resolver and a recorded
baseline: `Tests/Infra/TestDeclaredParamCoverage.cpp` locates the plugin source through
`IPluginManager` (`:2225`), walks every `.h`/`.cpp` (`:2245-2246`), extracts each
`REGISTER_RPC_HANDLER` body, resolves **one bounded call hop** into helpers the body hands `Ctx`
to, and compares the result against the live registry — declaration side from the registry,
usage side from source text. `PinWright.core.error_codes.AllEmittedCodesAreRegistered` and
`PinWright.infra.wiki_src.SourcePagesFollowRenderingRules` lint source the same way.

The same machine answers this question. Scan for the hazard calls the table's own families are
written around — `FKismetEditorUtilities::CompileBlueprint` (K), `CollectGarbage` /
`EditorDestroyWorld` / `GEditor->NewMap` (A), `FlushRenderingCommands` /
`PumpViewport` / `Viewport->Draw` (B, H.4, I), `UPackageTools::ReloadPackages` (J) — and fail on
any registered verb whose body reaches one, transitively through the same one-hop helper index,
without a tick-unsafe declaration. That test is what makes the declaration reliable; the
registration flag is what lets it read the declared side off the registry instead of re-parsing
`SafePoint.cpp` string literals. The two are complements, not alternatives, and the ordering that
matters is: the ratchet is the deliverable, the flag is the ergonomics.

Seed it as a **baseline**, not a clean assert, for the reason `TestDeclaredParamCoverage.cpp`
states for itself ("WHY A BASELINE INSTEAD OF A CLEAN ASSERT", `:29-35`): the first sweep will
record the 59 verbs from `B-blueprint-mutators-compile-ungated-tick-unsafe` plus whatever the
other four hazard families turn up, some of which will be deliberate. Record them, fail on
anything new, warn when a recorded entry stops reproducing. The count can then only go down.

## Severity, and what it is not

`Medium`. This is not itself a defect a caller can hit — the crash exposure is
`B-blueprint-mutators-compile-ungated-tick-unsafe` (Critical) and it must be worked first and
does not wait on this. What this ticket buys is that the 60th hazard-reaching verb fails a test
instead of an editor, which is a work-ordering argument for `Medium` and not for more. No
`blockedBy`: the two are independent, and either can land first.

**Known limit, stated so the ticket is not over-sold.** A handler reached through
`FRpcDispatcher::DispatchMethod` (handler-to-handler cross dispatch) bypasses
`ProcessRequest` and therefore the gate, however the gate is declared —
`Dispatch/SafePoint.h:353-362` and family C/I's notes on `level.load`,
`system.console_command` and `landscape.set_material`. The registration flag does not change
that rule, and the ratchet must exempt or separately flag those verbs rather than pretend a
declaration would have gated them.

## History
- `#1-table-catches-typos-not-omissions` `OPEN` reporter — Filed from the "the right fix is declaring tick-unsafety at `REGISTER_RPC_HANDLER` instead of in this table" note in `Dispatch/SafePoint.cpp:517-519` and the parent `E-compile-reinstances-live-instances-no-guard`. Verified by source reading, no editor run. Confirmed `FHandlerRegistration` (`HandlerRegistration.h:14-20`) has no flags field today; confirmed `GetTickUnsafeMethods()`/`IsTickUnsafeMethod()` have exactly one production consumer (`RpcDispatcher.cpp:696-697`) plus eight test files, so the read API can be preserved and the migration is populate-side only; confirmed the wiki generator does NOT read the table (the safe-point prose in `docs/wiki-src/` is hand-written per verb). Confirmed `TestSafePointGate.cpp:169-198` asserts only that listed names are registered — a false-positive guard with no false-negative side. Established that a derivation test is feasible with mechanisms already in this repo: `Tests/Infra/TestDeclaredParamCoverage.cpp` reads plugin source off disk (`:2225-2246`), extracts `REGISTER_RPC_HANDLER` bodies, follows one call hop into helpers, and ratchets a recorded baseline. Sharpened the parent's proposal accordingly: the registration attribute alone relocates the omission, the source-scan ratchet is what closes it.
- `#2-registration-flag-landed` `IN-REVIEW` developer — Landed the declaration half (change 1) without touching the legacy table, so the concurrent table edits keep working. `FHandlerRegistration` gains `bTickUnsafe`; `FAutoRegisterHandler` takes it as a trailing defaulted argument; new macro `REGISTER_RPC_HANDLER_TICK_UNSAFE(Method, Category, Summary, Params)` next to `REGISTER_RPC_MUTATING_HANDLER` (`Handlers/HandlerRegistration.h`). A flagged registration is also `bMutating` unless it is one of the read-only probes `HandlerRegistration.cpp` already names. Read API unchanged in shape: `PinWrightSafePoint::IsTickUnsafeMethod()` is now `table || flag`, and `GetTickUnsafeMethods()` returns the sorted union (`Dispatch/SafePoint.cpp`); the flag side is read straight off `FAutoRegisterHandler::GetPendingRegistrations()` and rebuilt whenever the registry grows, because handler TUs and later-loaded integration modules register after the first query (the constructor itself queries it). So the dispatcher consumer, `SetExtraTickUnsafeMethodForTests` and every existing test keep working; `TickUnsafeMethodsAreRegistered` now walks the union, which still means "every table entry is a registered verb". Migration path, written into `SafePoint.cpp`'s header, `docs/rpc-design.md` (tick safety), plugin `CLAUDE.md` (auto-registration) and `docs/arch.md` (registration flags): new verbs use the macro; migrate a table entry by switching its macro and deleting its table line in the same change. No existing entry was migrated in this pass (concurrent table edits). Test: `PinWright.core.safe_point.RegistrationDeclaredTickUnsafeIsGated` (`Tests/World/TestSafePointRegistrationDeclared.cpp`) registers a `_test.` probe ONLY through the new macro and asserts the flag, `bMutating`, `IsTickUnsafeMethod`, `GetTickUnsafeMethods`, and that the real `FRpcDispatcher::ProcessRequest` parks it on a forced-unsafe stack and runs it from `ProcessPendingRequests`; it fails if the flag term is dropped from `IsTickUnsafeMethod` or the macro stops passing it. Filter `PinWright.core.safe_point`. Compile-checked (clang syntax pass with UBT flags): HandlerRegistration.cpp, SafePoint.cpp, RpcDispatcher.cpp, TestAutoRegistration.cpp, the new test, one PinWrightGeometry handler. **Not done here, split out:** change 2 (the hazard-derivation ratchet that actually closes the omission hole), the mass migration of the table, and teaching the exact-string source scanners (`TestParamTypeGate`, `TestCaptureVerbParameterParity`, `TestNoParamHandlersReadNoArgs`, `TestHandlerTickSafetyRatchet::HandlerBlockReportsCompile`, several per-verb tests that search `REGISTER_RPC_HANDLER(\"verb\"`) to accept the new macro name before any verb migrates — see `E-tick-unsafe-hazard-derivation-ratchet`. `TestDeclaredParamCoverage` already steps over the macro-name tail, so it sees migrated verbs.
- `#3-derivation-ratchet-and-scanners` `IN-REVIEW` developer — Review follow-up: the hazard-derivation ratchet (the deliverable) now lands here, and `E-tick-unsafe-hazard-derivation-ratchet` is merged back into this ticket. **Ratchet:** `Tests/Infra/TestTickUnsafeHazardDerivation.cpp`, `PinWright.infra.tick_safety.HazardReachingVerbsAreGated` scans plugin source (Tests/ excluded), finds every registration of any spelling, and flags a registered verb whose body, or ONE helper it calls (matched by qualified name: `Class::Name` or the enclosing namespace/struct/class; member calls and `Type Name(args)` declarations not followed), reaches `CompileBlueprintWithDiagnostics` (the only full-compile site, already pinned by `HandlerHazardsStayGated`), `CollectGarbage`, `EditorDestroyWorld`, `NewMap`, `FlushRenderingCommands`, `PumpViewport`, `Viewport->Draw` or `ReloadPackages`, unless the body/helper calls `RunAtSafePoint`/`DeferJobToSafePoint`/`DeferRequestToSafePoint`/`DeferToSafePoint`; such a verb must be `IsTickUnsafeMethod()`, else it fails unless listed in `KnownUngatedHazardVerbs()` (stale entries warn). Floor assertion of 60 reaching verbs against scanner blindness. Seeded with an offline mirror of the same algorithm: 70 verbs reach a hazard, 69 gated, one ungated — `editor.jump_to_bookmark` (`EditorHandlerUtils::ForceRedrawViewportClient` -> `Viewport->Draw()`, the same synchronous draw `editor.set_camera`/`editor.set_game_view` are gated for). Gated it with `REGISTER_RPC_HANDLER_TICK_UNSAFE` (`Handlers/Editor/EditorCommandHandler.cpp`), so the baseline is empty. Scanner shapes pinned by `PinWright.infra.tick_safety.HazardScannerSeesEveryFollowedShape` (13 synthetic verbs: direct, helper hop, `return Helper(`, namespace/class/qualified-definition hops, viewport draw, plus member call, declaration, wrong qualifier, comment/string, gated body, gated helper that must NOT count; all three macro spellings, method literal on the next line). **Scanners:** shared `RpcRegistrationMacroNames` / `FindNextRpcRegistration` / `RpcRegistrationMethod` / `FindRpcRegistrationBlock` in `Tests/TestUtils.h` (all three spellings, whitespace before `(`, method literal on the next line); routed `TestHandlerTickSafetyRatchet::HandlerBlockReportsCompile`, `TestAssetBatchMutators::ExtractHandlerBlock`, `TestAssetImportSafePoint`, `TestGenerateLodsPersistenceAndCompile`, `TestNoParamHandlersReadNoArgs::ScanFile` and `TestParamTypeGate::TryGetRegisteredMethod` through it, and widened `TestCaptureVerbParameterParity`'s line regex the same way. Those scanners now also see the existing `REGISTER_RPC_MUTATING_HANDLER` verbs and the three multi-line `NiagaraSearchHandler.cpp` registrations they used to skip, so new findings there are real, not regressions of this change. **Migration:** moved this batch's two new table entries onto the macro — `render.capture_actor_preview` (`Handlers/Render/ActorPreviewCaptureHandler.cpp`) and `audio.authoring.set_sound_wave_gain` (`Handlers/Audio/SoundWavePcmHandler.cpp`), evidence comments moved with them, table lines deleted. The rest of the legacy table is still not migrated (concurrent edits); that and teaching per-verb literal needles (`TestAssetPieSafeResolution`, `TestVFXHandlers`, `TestEffectStepAndCapture`, which fail loudly rather than silently) remain follow-up work for whoever migrates an entry. **Review nits:** `DeclaredTickUnsafeMethodSet` now folds in only registrations added since its last call (O(N) total, was a full rescan per registration); game-thread-only note on `IsTickUnsafeMethod`/`GetTickUnsafeMethods` in `SafePoint.h`; `docs/arch.md` now lists all three callers; `docs/rpc-design.md` names the ratchet. Compile-checked (clang syntax pass with UBT flags) every touched .cpp; `check_test_ids.py` and `check_test_skips.py` clean. Filters: `PinWright.infra.tick_safety`, `PinWright.core.safe_point`, and the touched scanner tests `PinWright.infra.dispatcher.ParamTypeGate`, `PinWright.core.param_specs`, `PinWright.render.parameter_parity`, `PinWright.asset.batch_mutators`, `PinWright.asset.import`, `PinWright.asset.generate_lods`.
- `#4-review-fixes` `IN-REVIEW` developer — Re-review fixes. CHANGELOG entries for the new `REGISTER_RPC_HANDLER_TICK_UNSAFE` macro and for the `editor.jump_to_bookmark` behaviour change (now deferred to a safe point, gains `expectWorld` / `WORLD_MISMATCH`, response carries `world`); `docs/wiki-src/editor.md` jump_to_bookmark section updated to match. `render.capture_actor_preview` added to `PinWright.core.safe_point.KnownVictimsAreGated` (`Tests/World/TestSafePointGate.cpp`), so reverting its macro now fails a test. `FindRpcRegistrationBlock` (`Tests/TestUtils.h`) returns early when the quoted method literal is absent from the file, removing the ~9x slowdown of `HandlerHazardsStayGated`. NITs: reworded the "misspelling is a compile error" claim in `docs/rpc-design.md` and `Dispatch/SafePoint.cpp` (only a misspelled macro name fails to compile), fixed the stale assertion text in `TestSoundWavePcmHandler.cpp`, removed the unused `FScanResult` in `TestTickUnsafeHazardDerivation.cpp`. `E-tick-unsafe-hazard-derivation-ratchet` re-opened, scoped to the table migration (its #3). Nothing built or run yet; fastcheck clean.
- `#5-verified-linux` `DONE` tester — Fix commit 5e0bd230. Passed non-skipped in run3/full: `PinWright.core.safe_point.RegistrationDeclaredTickUnsafeIsGated` (change 1: a verb registered only via `REGISTER_RPC_HANDLER_TICK_UNSAFE` is flagged, mutating, in `IsTickUnsafeMethod`/`GetTickUnsafeMethods`, and parked by the real dispatcher), `PinWright.infra.tick_safety.HazardReachingVerbsAreGated` (change 2, the ratchet: no registered verb reaches a hazard without a gate, empty baseline, floor of 60 reaching verbs) and `PinWright.infra.tick_safety.HazardScannerSeesEveryFollowedShape` (13 synthetic shapes). Also passed: `PinWright.infra.tick_safety.HandlerHazardsStayGated`, `PinWright.core.safe_point.TickUnsafeMethodsAreRegistered` and `PinWright.core.safe_point.KnownVictimsAreGated` (now pins `render.capture_actor_preview`). The two skipped safe_point tests (`NestedNamedThreadPumpIsUnsafe`, `DispatcherDefersFromNestedNamedThreadPump`) are unrelated to this ticket. Acceptance: the declaration sits at registration, and the derivation ratchet that closes the omission hole is the deliverable; both are demonstrated, and the ratchet found and gated `editor.jump_to_bookmark`. Coverage limits: the legacy table is not migrated; that is now scoped to `E-tick-unsafe-hazard-derivation-ratchet`. The scanner follows one helper hop only, and cross-dispatched verbs still bypass the gate, as the ticket states.
