---
id: E-tick-unsafe-declared-at-registration
title: "Tick-unsafety lives in a hand-maintained name table in a file no handler author opens, and the only test on it catches typos rather than omissions — every gap so far was found by an editor death, not by the suite"
status: OPEN
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
