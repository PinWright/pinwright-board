---
id: B-niagara-compile-wait-test-rejects-unversioned-emitter
title: "BoundedPumpCompletesTransientSystem skips on every run: it requires a valid emitter-handle version guid, which a non-versioned emitter never has"
status: IN-REVIEW
severity: Medium
category: bug
tags: [gap-analysis-2026-09-28, niagara, compile, tests-skip, test-fixture]
---

# BoundedPumpCompletesTransientSystem skips on every run: it requires a valid emitter-handle version guid, which a non-versioned emitter never has

`PinWright.niagara.CompileWait.BoundedPumpCompletesTransientSystem` (`Tests/Niagara/TestNiagaraCompileWait.cpp`) skips with `reason=niagara-compile-fixture-not-compilable -- the saved fixture lacks a compilable system script or a versioned emitter` in every 5.8 run on record (`Saved/PinWright/test-runs/499a9295d82f445ba80a44ebe091bbb9/automation.log:43025`, `8579e054.../automation.log:43035`, `Saved/Logs/pw_gapwave_full_offscreen2.log:42599`, `pw_gapwave_full_headless2.log:41465`). No warning or error precedes the skip. The behavioural half of the compile-wait fix (`B-niagara-compile-wait-does-not-wait`) is therefore never measured.

Root cause is the test, not production. The guard at `:181-182` rejects `!EmitterVersionGuid.IsValid()`, and `FindFixtureEmitter` took that guid from `Handle.GetInstance().Version`. For an emitter with versioning disabled the handle version is `FGuid()` (`NiagaraEmitterHandle.cpp:312` on 5.8), and the stock fixture's emitter (`/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion`, standard mode, no `bVersioningEnabled` in its name table) is not versioned. The script half of the guard is not the cause: `PinWright.effect.spawn_niagara.MeasuredStateAndCompileFailure` duplicates the same fixture and hard-fails (`AddError`) on the identical `IsCompilable() / GetLatestSource()` check, and it passes in the same run (`499a9295.../automation.log:28899`, compile landed in 0.159 s).

An invalid handle guid does not break the fan-out either: `FNiagaraEmitterHandle::UsesEmitter` (`NiagaraEmitterHandle.cpp:363-377`) ignores the version for a non-versioned emitter. Production `niagara.compile` never reads the handle version: `ResolveEmitterVersionGuid` (`NiagaraEditTypes.cpp:1661`) uses `EmitterData->Version.VersionGuid`, which is always populated.

**Fix:** the test resolves the guid the way production does, from the handle's emitter data.

## History
- `#1-handle-guid-invalid-skip` `OPEN` reporter — `BoundedPumpCompletesTransientSystem` skips on UE 5.8 in all four recorded suite logs with `niagara-compile-fixture-not-compilable`. By elimination against the passing sibling `effect.spawn_niagara.MeasuredStateAndCompileFailure` (same fixture, same script checks), the failing term is `!EmitterVersionGuid.IsValid()`: the handle version of the fixture's non-versioned emitter is `FGuid()`.
- `#2-guid-from-emitter-data` `IN-REVIEW` developer — `TestNiagaraCompileWait.cpp` `FindFixtureEmitter` now returns the emitter only when `Handle.GetEmitterData()` is non-null and sets `OutVersionGuid = EmitterData->Version.VersionGuid` (mirrors `ResolveEmitterVersionGuid`). Guard and skip reason unchanged, so a host whose fixture has no emitter data or no system source still skips. No production change. Compile-checked with UBT `-SingleFile` on UE 5.8 only; not yet run. Needs a 5.8 run showing the test executes its assertions (no `PINWRIGHT_ASSERTIONS_SKIPPED` for it); the next gate it can hit is `niagara-compile-request-not-observable`.
