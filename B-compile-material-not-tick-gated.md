---
id: B-compile-material-not-tick-gated
title: "material.authoring.compile_material flushes rendering commands and recreates live render state from the handler body, but is not in the tick-unsafe table"
status: OPEN
severity: High
category: bug
tags: [material, material-authoring, landscape, tick-safety, safepoint, render-flush, deadlock-risk, editor-crash, observed-fault]
encounters: 3
lastSeen: 2026-08-16T06:46:42Z
---

# `compile_material` is hazard family B and is not gated

`Dispatch/SafePoint.cpp:24-36` defines hazard family B as "RE-ENTRANT SLATE TICK / VIEWPORT DRAW /
RENDER FLUSH from a handler body", and names the mechanism explicitly: *"FlushRenderingCommands from
inside the tick's parallel-task wait is a documented stall/deadlock source."*
`material.authoring.compile_material` now sits squarely in that family and is **absent** from
`GTickUnsafeMethodNames` (`SafePoint.cpp:46-138`; only `render.detect_z_fighting` at `:72`,
`python.execute`, and the two console verbs are listed).

## What the verb now does per call

- `MaterialAuthoringHandler.cpp` opens `FMaterialUpdateContext UpdateContext;` with default options
  (`RecreateRenderStates | SyncWithRenderingThread`). `bSyncWithRenderingThread` makes **both** the
  constructor and the destructor call `FlushRenderingCommands` (`MaterialShared.cpp:5026`, `:5078`).
- `MaterialLandscapeConsumers.h` then calls
  `ALandscapeProxy::UpdateAllComponentMaterialInstances(true)` **per consuming landscape**, which
  builds one `FComponentRecreateRenderStateContext` per landscape component plus a second
  `FMaterialUpdateContext` that also carries `SyncWithRenderingThread`, and
  `GetCombinationMaterial` adds a further `FlushRenderingCommands` per new combination MIC
  (`LandscapeEdit.cpp:620`).

So the flush count per call scales with the number of consuming landscapes and their components,
where before the change it was bounded by the pre-existing
`GShaderCompilingManager->FinishAllCompilation()` in `MaterialCompileErrorCollector.h`.

## Why this is filed rather than fixed

Stated so the severity is not over-read:

- The gap is **pre-existing, and widened** — not introduced. `compile_material` already called
  `FinishAllCompilation()` before this change, which is comparable, and it was never gated.
- `landscape.set_material` reaches the same engine code through `PostEditChangeProperty` and is
  **also** absent from the table, so the table is already inconsistent about this code path.
- ~~No hang, stall or deadlock has been observed.~~ **Superseded by `#2` — the editor crashed twice,
  90 seconds apart, on 2026-08-16.** The original wording is kept for the record: *"The verb was
  driven repeatedly through the dispatcher during integration pass 11 without incident — but that
  exercised ordinary arrivals, not a mid-tick arrival, which is the actual hazard."* That reasoning
  was right; only its conclusion ("no fault observed") has expired. The observed fault is also not
  the hang this predicted — it is a render-thread access violation during resource release — so the
  failure mode is restated in `#2` while the fix rationale is unchanged.

## Fix shape

A one-line entry in `GTickUnsafeMethodNames` for `material.authoring.compile_material`, per
`docs/rpc-design.md` §10 ("table entry preferred over a hand-written gate"). Decide at the same time
whether `landscape.set_material` belongs there too — the two now reach identical engine code, and
leaving one gated and the other not is the kind of inconsistency the table exists to remove. Any
entry needs a test, because gating changes when the response is delivered.

## History
- `#1-found-by-audit` `OPEN` reporter — Found by auditing the derived-state-honesty changeset
  (`099b83b3`..`0fe35187`) against `Docs/rpc-design.md`'s own "before you ship a verb" checklist
  during integration pass 11. The checklist line "Tick-unsafe work is gated through
  `Dispatch/SafePoint.h`" is the one that fails. Verified directly:
  `grep -n "compile_material" Dispatch/SafePoint.cpp` returns no table entry.
- `#2-the-predicted-fault-happened-twice` `OPEN` reporter — **Severity raised Medium -> High: the editor crashed mid-call, twice, 90 seconds apart, leaving a no-op saved to disk.** `Saved/Crashes/UECC-Windows-56C3674444F20B8FB6805D919C4B8685_0002` (log `2026.08.16-06.45.11` UTC = 11:45:11 local; logs are UTC+0, this machine UTC+5) and `Saved/Crashes/UECC-Windows-67BA29ED4207DCECFC3DF6BC1CD2D5E4_0002` (`06.46.42` UTC = 11:46:42 local). Both `<CrashType>Crash`, `<IsEnsure>false`, `EXCEPTION_ACCESS_VIOLATION reading address 0xffffffffffffffff`, engine `5.8.1-56057345`, command line `... -AutoDeclinePackageRecovery -RunningUnattendedScript`, `<SecondsSinceStart>` 49 and 55. **The faulting thread is the RENDER thread**, inside `BeginReleaseResource`'s lambda via `ExecuteCommand` (`RenderingThread.cpp:1533`) under `FRenderThreadCommandPipe::EnqueueAndLaunch` — i.e. draining queued release commands. **The game thread is parked in a render fence directly below PinWright frames**: `UnrealEditor-PinWright +2be8c2 / +292f39 / +719b8e / +6f8565` -> `UnrealEditor-Engine` -> `UnrealEditor-RenderCore` fence wait. Immediately preceding each fault, `LogRendererCore: Warning: FlushRenderingCommands called recursively! 2 calls on the stack.` fires **15 times in ~170 ms** in the first crash and **21 times** in the second. That is exactly the hazard-family-B signature this ticket describes, produced by exactly the work it enumerates: `FMaterialUpdateContext` with `RecreateRenderStates | SyncWithRenderingThread` flushing in both ctor and dtor (`MaterialLandscapeConsumers.h:261`), `UpdateAllComponentMaterialInstances(true)` per consuming landscape (`:215`, inside the consumer loop at `:205`), and `GShaderCompilingManager->FinishAllCompilation()` (`MaterialCompileErrorCollector.h:51`). **Attribution limit, stated rather than papered over:** `compile_material` does not appear literally in either crash log, because the plugin logs method names only on failure — `rg -c "compile_material"` is 0 in both crash dirs and in every `Saved/Logs/EAContentExamples58*.log`. The attribution rests on the PinWright game-thread frames plus the recursive-flush storm, not on a named-verb log line.
- `#3-distinct-from-the-reentrant-gc-crash` `OPEN` reporter — Assessed against `B-python-execute-reentrant-gc-crash` (OPEN, Critical) and found **distinct on four independent, checkable discriminators**, recorded so the two are never merged. (1) **No GC in the path** — `CollectGarbage`/`ForceGarbageCollection` have zero occurrences across `Private/Handlers/Material/` and `Private/MGIR/`; the GC ticket's entire mechanism is a synchronous collect inside `FMaterialEditorUtilities::BuildTextureStreamingData`. (2) **No Python frame** — neither crash context contains a `python311.dll`, `PyUtil`, `FPyWrapper*` or `PythonScriptPlugin` stack frame; the only Python hits are `<Modules>` entries and two plugin-manifest `FriendlyName` strings, and there is no `OnPreGarbageCollect` frame anywhere. (3) **Different thread and different fault** — the GC crash faults on the GAME thread inside `FPythonScriptPlugin::OnPreGarbageCollect` -> `python311.dll` at `0x00007ff800007383`; these fault on the RENDER thread in `BeginReleaseResource` at `0xffffffffffffffff` with the game thread parked in a fence. (4) **Different code route** — already established in that ticket's own body: `compile_material` does not take the `RecompileMaterial` path, and a grep for `RecompileMaterial|BuildTextureStreamingData` over the whole plugin source returns zero matches. **Explicit warning against a false merge:** `FlushRenderingCommands called recursively` appears in BOTH incidents and is therefore **not** a discriminator — the GC ticket itself notes 9 seconds of it before its fault, benignly. Anyone tempted to unify these on that warning should read all four discriminators first. Filed here rather than as a new ticket because this ticket already names the verb, the mechanism, the exact engine call sites and the fix; the crash supplies the one element it explicitly said it lacked.
