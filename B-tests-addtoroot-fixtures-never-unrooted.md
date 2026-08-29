---
id: B-tests-addtoroot-fixtures-never-unrooted
title: "Six test files AddToRoot their fixtures and never RemoveFromRoot — five Niagara tests plus AnimAuthoringTestFixtures.h — so those objects and their packages stay GC-immortal for the rest of the editor process while sibling files use an RAII root guard"
status: OPEN
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

## History
- `#1-six-unrooted-fixtures` `OPEN` reporter — Source-only census over all eight modules; **no suite was run and no plugin source was modified** (tree is mid-verification on another wave). Derived as `comm -23` between the sorted list of test files containing `AddToRoot` and those containing `RemoveFromRoot` across `Source/*/Private/Tests/`: exactly the six files listed. Per-file counts corroborate — the five Niagara files carry 1 `AddToRoot` and 0 `RemoveFromRoot` each; `AnimAuthoringTestFixtures.h` carries 2 and 0. Confirmed `FAuthorableSystemRoots` exists at `Tests/Assets/NiagaraEditTestUtils.h` and is used by ten sibling files, five of them in the same `Tests/Niagara/` directory as the offenders, so the fix is adoption of an in-tree idiom and not a design. Independently re-derives the incidental finding on `B-suite-host-gc-crash-in-combined-group-run` `#4`; the set matches that note exactly. Filed separately from `B-tests-spawn-live-world-no-guard` (the world-actor half of the same audit) rather than merged into one test-hygiene ticket: different mechanism (GC-root teardown vs editor-world actor teardown), different fix sites, and different honest severities (Low vs Medium) — merging would force the Low half to be worked at Medium priority or the Medium half at Low, and the picker orders by severity. Severity Low, with the cross-test-contamination escalation condition recorded as an untested hypothesis rather than a claim.
