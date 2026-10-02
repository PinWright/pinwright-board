---
id: B-resolve-uclass-short-name-load-warns
title: "ResolveUClass runs LoadObject<UClass> on bare short names such as 'Object', which can never load and logs \"Failed to find object 'Class Object'\" on every short-name lookup"
status: IN-REVIEW
severity: Medium
category: bug
tags: [class-resolution, resolve-uclass, log-noise, loadobject, gap-analysis-2026-09-28]
encounters: 1
lastSeen: 2026-09-29T13:03:05Z
---

# ResolveUClass warns on every bare short class name

`ResolveUClass` (`Source/PinWright/Private/Utils/ClassUtils.cpp:122`, ~56 call sites across >=45
verbs) tries, in order: `FindObject<UClass>(nullptr, Input)` (step 1, `:157`), then
`LoadObject<UClass>(nullptr, Input)` (step 2, `:162`), and only then the short-name tiers (step 4
`/Script/<Pkg>.<Name>`, `:199-215`; step 5 `TObjectIterator` by name, `:218-224`; prefix tiers after
that). For a bare name like `Object`, `Actor` or `PointLight`, step 2 has nothing to load: there is no
package path, so `StaticLoadObject` fails and, without `LOAD_NoWarn`, reports
`LogUObjectGlobals: Warning: Failed to find object 'Class Object'`
(`C:\UE_5.8\Engine\Source\Runtime\CoreUObject\Private\UObject\UObjectGlobals.cpp:1500-1520`). The
lookup then succeeds in a later tier, so every successful short-name resolution still leaves a
failure warning in the log.

Observed: three `Failed to find object 'Class Object'` lines per run of the macro-recompile test,
traced to `object<Object>` in its IR (`B-macro-recompile-test-invalid-ir` body and `#1`, which
explicitly leaves this resolver noise unfixed). The same mechanism, by code read: step 4 also calls
`LoadObject<UClass>` on `/Script/Engine.<Name>` before reaching `/Script/CoreUObject`, and a native
`/Script/` class that `FindObject` did not find cannot be loaded, so those loads can only add more
warnings. `B-bpir-preload-resolves-bool-literals` (OPEN) is a separate caller-side defect (literals
sent to the resolver); fixing this ticket would also silence its `Class true` lines, but that
ticket's candidate filter should still be fixed.

**Impact:** log noise that reads as a class-resolution failure and sends log readers after the wrong
defect (pure friction), on a path nearly every session hits (bumped up one). Also a wasted load
attempt per short-name call.
**Fix:** in `ResolveUClass`, skip step 2 when the input contains no `/` and no `.` (a bare name can
only be found, never loaded), and in the `/Script/` tiers use `FindObject` only (or pass
`LOAD_NoWarn | LOAD_Quiet` where a load is genuinely possible). Test: resolving `Object`, `Actor` and
`PointLight` succeeds and emits no `LogUObjectGlobals` warning (capture with an output device).

## History
- `#1-short-name-load-warns` `OPEN` reporter — Found in today's verification; step 2 read at `ClassUtils.cpp:161-164`, engine warning path read at `UObjectGlobals.cpp:1500-1520`. Distinct from `B-bpir-preload-resolves-bool-literals` (caller sends non-class literals) and left open by `B-macro-recompile-test-invalid-ir`.
- `#2-path-shaped-loads-only` `IN-REVIEW` developer — `ResolveUClass` step 2 now calls `LoadObject<UClass>` only when the input contains `/` or `.` (a dotted name like `Engine.Actor` still loads via the short script-package map); the `/Script/<Pkg>.<Name>` tiers (4, 5.5, 6) are `FindObject`-only, since script packages are `PKG_CompiledIn` and `StaticLoadObjectInternal` never loads into them - the load could only warn, create a stray `/Script/<Module>` package for an unloaded module, and mark the package known-missing (silencing later genuine warnings). Same defect fixed in the sibling `ResolveClassByName` step 2.5 (`/Script/Engine.<Name>` load removed). Files: `Source/PinWright/Private/Utils/ClassUtils.cpp`, `Source/PinWright/Private/Tests/Core/TestClassUtils.cpp`, `docs/arch.md` (resolution chain), `CHANGELOG.md`. Test: `PinWright.core.class.resolve_uclass.BareNamesLogNoLoadWarning` - resolves `Object`/`Actor`/`PointLight` through both resolvers under a GLog needle watch and asserts zero `Failed to find object` lines (a revert of step 2 is red every run; a revert of only the `/Script/` loads is red only on the session's first miss because of the engine's known-missing cache). No verb contract change.
