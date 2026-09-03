---
id: B-tests-addtoroot-fixtures-never-unrooted
title: "Six test files AddToRoot their fixtures and never RemoveFromRoot — five Niagara tests plus AnimAuthoringTestFixtures.h — so those objects and their packages stay GC-immortal for the rest of the editor process while sibling files use an RAII root guard"
status: IN-REVIEW
severity: Low
category: bug
tags: [tests, hygiene, garbage-collection, addtoroot, fixture-teardown, niagara, memory, raii-guard]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# `AddToRoot` with no matching `RemoveFromRoot`

Six test files root a fixture object and never unroot it. A rooted `UObject` is unreachable by
GC for the life of the process, and it keeps its outer package alive with it, so a full suite
run accumulates one immortal fixture (plus its package) per affected test.

```
Source/PinWright/Private/Tests/Assets/AnimAuthoringTestFixtures.h
Source/PinWright/Private/Tests/Niagara/TestNIRGraphDataflow.cpp
Source/PinWright/Private/Tests/Niagara/TestNIRGraphLinkCoverage.cpp
Source/PinWright/Private/Tests/Niagara/TestNiagaraGetModuleInputs.cpp
Source/PinWright/Private/Tests/Niagara/TestNiagaraMoveModule.cpp
Source/PinWright/Private/Tests/Niagara/TestNiagaraResetModuleInput.cpp
```

Exactly the set difference between the 80 test files that call `AddToRoot` and the 89 that call
`RemoveFromRoot`.

The idiom to adopt already ships beside them. `FAuthorableSystemRoots` — an RAII root guard
used in `Tests/Assets/NiagaraEditTestUtils.h` and by seven sibling Niagara test files
(`TestNiagaraEditHandler`, `TestNiagaraAdvancedEditHandler`, `TestNiagaraAddEmitterQuiesce`,
`TestNiagaraDataInterfaceConsistency`, `TestNiagaraEditorOpenGuard`,
`TestNiagaraFinalizeEditDataInterfaceGate`, `TestNiagaraGraphCreateNode`,
`TestNiagaraSetParameterEmitterScope`, `TestNiagaraValidateComponentActivation`,
`TestCaptureSubjectNiagara`) — unroots on scope exit. Five of the six offenders sit in the same
directory as files that use it. `AnimAuthoringTestFixtures.h` has no equivalent guard in its
own family; the nearby `Tests/Gameplay/TestAnimationFixtures.h` pairs its 3 `AddToRoot` calls
with 2 `RemoveFromRoot` calls by hand.

## Provenance

Recorded as an incidental finding on `B-suite-host-gc-crash-in-combined-group-run` `#4` — "five
Niagara test files plus `AnimAuthoringTestFixtures.h` `AddToRoot` fixtures with no
`RemoveFromRoot`" — while that ticket's own GC-elimination hypothesis was being refuted. That
audit's conclusion was explicit that **no exposed GC site was found** ("EXPOSED SITES = 0"), so
this is not a crash candidate; it is the leak the same sweep noticed on the way past. Filed as
its own ticket because a finding in another ticket's history is never scheduled by the fix
picker. Re-derived independently here; the set matches exactly.

## Severity

**Low.** Impact class is pure hygiene: bounded memory held for the life of a test process that
is destroyed at the end of the run. Nothing is wrong, nothing is blocked, no result is a lie,
and no crash is attributable — the audit that found it explicitly refuted the crash link.

Reach modifier considered and **not** applied: the sites run on every full suite run, which
argues a bump up, but the rubric's bump is for a *gap on a method that runs every session*,
and the cost here does not scale with reach in any way a caller feels — six immortal objects in
a process that already loads thousands.

**Escalation condition, recorded so a re-triage need not re-derive it:** a rooted fixture also
roots its package, and a package that stays resident under a `/Game` path can be found by a
later test's `FindObject`/`LoadObject` where a fresh one was expected. That would be
cross-test contamination rather than a leak, and would move this to Medium. **It was not
observed and not tested for** — flagged as the thing to check, not as a claim.

**Fix:** wrap the five Niagara fixtures in `FAuthorableSystemRoots` from
`Tests/Assets/NiagaraEditTestUtils.h`, the guard their directory siblings already use; add a
matching `RemoveFromRoot` (or a small guard of the same shape) to the two sites in
`AnimAuthoringTestFixtures.h`. Six files, mechanical, no design. A `check_test_skips.py`-style
source lint pairing `AddToRoot` against `RemoveFromRoot`-or-guard per file would stop the class
recurring, and is cheap because the shape is file-local.

## Fix

**Verdict: PARTLY TRUE (source-verified).** The five Niagara tests did not each leak on their
normal and existing error paths: `NIRTestFixtures::DestroyFixture` already removes the rooted
system and emitters. The original file-local census missed that indirect teardown and the
caller-owned contract of the animation factories. It did find a real gap: eight post-setup early
returns in `TestNIRGraphDataflow.cpp` skipped teardown, and that helper redundantly called
`AddToRoot`.

**Root cause.** Root ownership was split between shared fixture construction, explicit
`DestroyFixture`, and caller-owned animation factories without a source-level ownership guard.
The source-only set difference therefore over-reported normal-path leaks and could not see the
dataflow null-node returns that actually bypassed cleanup.

**Files changed.**

- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Niagara\TestNIRGraphDataflow.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Niagara\TestNIRGraphLinkCoverage.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Niagara\TestNiagaraGetModuleInputs.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Niagara\TestNiagaraMoveModule.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Niagara\TestNiagaraResetModuleInput.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Assets\AnimAuthoringTestFixtures.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Assets\TestAnimSequenceCreate.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Assets\TestAnimSequenceDumpBuilder.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Format\TestPwAnimCompiler.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Assets\NiagaraEditTestUtils.h`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Niagara\TestNIRFixtures.cpp`
- `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Tests\Infra\TestFixtureOwnershipContracts.cpp`

The five Niagara tests now use `FAuthorableSystemRoots` for the system and first emitter, while
their existing `DestroyFixture` calls remain. Dataflow's eight null-node returns now explicitly
destroy the fixture; the direct helper `AddToRoot` was removed. The animation header keeps its
intentional two factory roots behind move-only `FScopedAnimAssetRoot`; all 13 calls to those two
factories construct that owner as the next statement, so early returns cannot skip unrooting.
`BuildEmptySystemWithEmitter` now destroys the already-rooted system if emitter construction
fails. `FAuthorableSystemRoots` is move-only, transfers ownership on move, and clears both owned
pointers after release so copying cannot cause a second unroot.

The Python-only scanner and its Python unit test were removed. Registered Unreal test
`PinWright.infra.contract.AddToRoot.ScopedFixtureOwnership` now reads every `.cpp` and `.h` under
`Source/PinWright/Private/Tests`, checks exact evidence-derived file/count allow-lists for raw
`AddToRoot` and the four known rooted producers, and verifies every `RunTest` producer call.
Niagara calls require `FAuthorableSystemRoots` unless their exact file/function/count is in the
legacy caller allow-list; animation calls require immediate `FScopedAnimAssetRoot` ownership in
the same lexical scope. Unreadable or missing expected files, changed counts, and new bypass files
all fail.

**Test IDs covered.**

`PinWright.niagara.decompile_nir.GraphOp`,
`PinWright.niagara.decompile_nir.GraphParameterMapGet.GetLineFormat`,
`PinWright.niagara.decompile_nir.GraphParameterMapGet.MetadataType`,
`PinWright.niagara.decompile_nir.GraphParameterMapSet.SetLineFormat`,
`PinWright.niagara.decompile_nir.GraphExplicitLink`,
`PinWright.niagara.decompile_nir.GraphFunctionCall`,
`PinWright.niagara.decompile_nir.GraphInput`,
`PinWright.niagara.decompile_nir.GraphOutput`,
`PinWright.niagara.decompile_nir.GraphParameterMapGet.AddPinSuppressed`,
`PinWright.niagara.decompile_nir.GraphParameterMapSet.AddPinSuppressed`,
`PinWright.niagara.decompile_nir.GraphLinkCoverage`,
`PinWright.niagara.ModuleInputsSchema`,
`PinWright.niagara.move_module.Registration`,
`PinWright.niagara.move_module.RelayoutsNodePosY`,
`PinWright.niagara.reset_module_input.HandlersRegistered`,
`PinWright.niagara.reset_module_input.FindOverrideNodeNullGraph`, and
`PinWright.niagara.reset_module_input.DynamicInputResetKeepsStackChain`.

The animation factory callers are covered by `PinWright.Assets.AnimSequenceCreate.KeyCountMismatchIsRefused`,
`PinWright.Assets.AnimSequence.DumpBuilder.Shape`,
`PinWright.Assets.AnimSequence.AssetDump.WritesAnimSequenceAspectFile`,
`PinWright.Assets.AnimSequence.DumpBuilder.SyncMarkerMoveAppearsInDiff`,
`PinWright.Assets.AnimSequence.DumpBuilder.BoneTracksReadback`,
`PinWright.Animation.Compiler.BakesDenseReferencePoseTracks`,
`PinWright.Animation.Compiler.HeldPoseTrackCompilesAsOneKey`,
`PinWright.Animation.Compiler.ReportsSkeletonAndLoopDiagnostics`,
`PinWright.Animation.Compiler.PropagatesLoopFlag`,
`PinWright.Animation.Compiler.SyncMarkersAreIdempotentAndRefreshDerivedState`, and
`PinWright.Animation.Compiler.ProvenanceRefusalIsNotABadValue`.

The new registered structural coverage is
`PinWright.infra.contract.AddToRoot.ScopedFixtureOwnership`. Existing functional test IDs above
were not renamed.

**Verification.** Static source review only. No build, Unreal process, automation test, MCP call,
or Python scanner was run. No production handler, engine, outer project, Config, or Saved file
was edited.

## History
- `#1-six-unrooted-fixtures` `OPEN` reporter — Source-only census over all eight modules; **no suite was run and no plugin source was modified** (tree is mid-verification on another wave). Derived as `comm -23` between the sorted list of test files containing `AddToRoot` and those containing `RemoveFromRoot` across `Source/*/Private/Tests/`: exactly the six files listed. Per-file counts corroborate — the five Niagara files carry 1 `AddToRoot` and 0 `RemoveFromRoot` each; `AnimAuthoringTestFixtures.h` carries 2 and 0. Confirmed `FAuthorableSystemRoots` exists at `Tests/Assets/NiagaraEditTestUtils.h` and is used by ten sibling files, five of them in the same `Tests/Niagara/` directory as the offenders, so the fix is adoption of an in-tree idiom and not a design. Independently re-derives the incidental finding on `B-suite-host-gc-crash-in-combined-group-run` `#4`; the set matches that note exactly. Filed separately from `B-tests-spawn-live-world-no-guard` (the world-actor half of the same audit) rather than merged into one test-hygiene ticket: different mechanism (GC-root teardown vs editor-world actor teardown), different fix sites, and different honest severities (Low vs Medium) — merging would force the Low half to be worked at Medium priority or the Medium half at Low, and the picker orders by severity. Severity Low, with the cross-test-contamination escalation condition recorded as an untested hypothesis rather than a claim.
- `#2-source-verified-partial-fix` `IN-REVIEW` developer — Verdict **PARTLY TRUE** in
  `X:\src\unreal\unreal-fpv-dev\Plugins\PinWright`: the original same-file census was too
  coarse. `NIRTestFixtures::DestroyFixture` already removes the rooted system and emitters on
  the existing normal/error paths in the five Niagara files, and every current caller of the
  animation factories already removes the returned root. The real gap was the six post-setup
  early returns in `TestNIRGraphDataflow.cpp`; its setup also redundantly called `AddToRoot`.
  The five Niagara tests now use `FAuthorableSystemRoots` (system plus first emitter), retain
  `DestroyFixture`, and cover those early returns. `AnimAuthoringTestFixtures.h` now documents
  its deliberate caller-owned root contract. Added the stdlib-only
  `Content\Python\check_test_roots.py` source ratchet and focused
  `Content\Python\tests\test_root_teardown_scan.py`; the ratchet scans 870 PinWright test
  sources and reports CLEAN. No build, Unreal run, automation suite, MCP, or Git operation was
  run per the ticket brief.
- `#3-registered-root-ownership-ratchet` `IN-REVIEW` developer — Review correction: the
  dataflow patch covers eight post-setup early returns, not six. Fixed the separate
  `BuildEmptySystemWithEmitter` failure after system rooting, made `FAuthorableSystemRoots`
  move-only, removed the two task-created Python scanner files, and added registered Unreal test
  `PinWright.infra.contract.AddToRoot.ScopedFixtureOwnership` with per-`RunTest` Niagara ownership
  and receiver-specific animation caller checks. Static source review only; no build, Unreal,
  automation, MCP, or Python test was run.
- `#4-whole-tree-root-ratchet` `IN-REVIEW` developer — Follow-up fixes the verifier gaps:
  `ScopedFixtureOwnership` now scans every main-module test `.cpp`/`.h`, requires exact
  evidence-derived raw-root and rooted-producer allow-lists plus exact legacy Niagara caller
  entries, and fails on unreadable/missing sources or new bypasses. Animation factory callers now
  use move-only `FScopedAnimAssetRoot` immediately after all 13 calls, replacing the unsafe
  later-`RemoveFromRoot` check. Static source review only; no build or automation run.
