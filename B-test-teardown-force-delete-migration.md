---
id: B-test-teardown-force-delete-migration
title: "881 test teardowns still route through CleanupTestAsset -> ObjectTools::ForceDeleteObjects, the idiom the plugin's own headers document as a crash vector, while the shipped safe replacement PwTestAssetTeardown::DiscardCreatedAssetByObjectPath is adopted at only 60 sites"
status: IN-REVIEW
severity: Medium
category: bug
tags: [tests, teardown, force-delete, objecttools, garbage-collection, suite-stability, truncated-run, migration, cleanup-test-asset]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# The hazardous teardown is the default; the safe one is the exception

The plugin documents `ObjectTools::ForceDeleteObjects` as a crash vector in three separate
headers, ships a replacement that avoids it, and then routes almost every test teardown
through it anyway.

**The hazard, in the plugin's own words.** `Tests/TestAssetTeardown.h:32-40`, on why
`DiscardCreatedAssetByObjectPath` exists:

> Deliberately NOT `UEditorAssetLibrary::DeleteAsset` -> `ObjectTools::ForceDeleteObjects`:
> force-delete's `GatherObjectReferencersForDeletion` serializes the freshly-created,
> never-reloaded asset to collect referencers, and for a `UMetaSoundSource` (whose frontend
> document graph holds transient/uninitialized references) that reference-gathering archive
> **crashes the editor** under `-unattended -RenderOffScreen` — the same never-reloaded-asset
> hazard `CleanupTestAsset` documents for 5.4.

`Tests/TestUtils.h:522-545` documents two more failure modes on the same call — the UE 5.4
null-deref in the reference-gathering archive (worked around by *skipping cleanup entirely* on
5.4), and a UE 5.5 crash where `ForceDeleteObjects`' GC collects a material whose shader map is
still compiling (worked around by a blocking `FinishAllCompilation` first).
`Utils/AssetDeletePolicy.h:7-36` documents the reference-replacement window that Epic names in
`ForceDeleteObjects`' own terminal `ensureMsgf`.

**The adoption ratio.** Counted across all eight modules:

| teardown | call sites | files |
|---|---|---|
| `CleanupTestAsset(...)` → `ForceDeleteObjects` | **881** | 214 |
| `PwTestAssetTeardown::DiscardCreatedAssetByObjectPath(...)` | **60** | 17 |

The safe helper is confined to the audio/MetaSound fixtures it was written for. Everything
else still force-deletes.

## Relationship to the truncated suite runs

`B-suite-host-gc-crash-in-combined-group-run` (OPEN, High) `#4` audited the teardown idiom,
**refuted** that ticket's own GC-elimination hypothesis, and recorded this on the way past:

> `ForceDeleteObjects` (535/331/599 calls per full run) is where both PDS truncations stopped,
> with a safe replacement idiom already shipping in
> `PwTestAssetTeardown::DiscardCreatedAssetByObjectPath`.

and, on the two runs concerned:

> batch4 and batch5 carry no crash marker, no Fatal and NO crash report [...] and both end on
> a complete line inside `ObjectTools::ForceDeleteObjects` — they are the truncated-log class,
> not GC crashes, so two of the three 'nondeterministic' data points are unattributable.

Read carefully, that is **correlation, not attribution**: two truncated logs happen to end
inside this call, with no fault recorded. The per-run call counts (535 / 331 / 599) are quoted
from that audit and were **not** re-measured here — verifying them requires a suite run, which
was not permitted in this pass.

Filed as a scoped child of that ticket rather than as a history entry on it, for two reasons.
First, the parent's subject is the truncation itself, whose cause is still open; migrating the
teardown is a hardening action that does not close it, so folding them together would leave the
parent unclosable. Second, a finding recorded only in another ticket's history is never
selected by the fix picker, which works `OPEN` tickets by severity.

## Severity

**Medium**, and deliberately not carried up from the parent's High.

- *Established:* a call the plugin's own headers document as a crash vector, with a shipped
  safe replacement, runs on the overwhelming majority of test teardowns. Two of the three
  crash-workaround comments in `TestUtils.h` exist solely to survive it, and one of them
  (`UE_VERSION_OLDER_THAN(5,5,0)`) resolves the problem by *not cleaning up at all*.
- *Not established:* that it caused the truncations. The evidence is two logs ending inside the
  call with no fault, no crash marker and no crash report — consistent with it being where the
  process spends its time as much as with it being the fault site. Rating Critical or High off
  that would be inflation, and the parent's own `#4` is explicit that those two data points are
  "unattributable".
- Reach: every suite run, hundreds of calls each — which argues a bump up from a hygiene
  Medium; the unproven-cause discount argues down. Recorded as cancelling.

**Escalation condition:** a crash report, a `Fatal` line, or a callstack naming
`ObjectTools::ForceDeleteObjects` / `GatherObjectReferencersForDeletion` in a suite run moves
this to High immediately, and to Critical if it is reproducible.

**Fix, and the reason it is not simply a find-and-replace.** The two helpers are not
interchangeable: `CleanupTestAsset` takes a *package path* and deletes an asset that may exist
on disk (it "also removes the package's on-disk file if one exists");
`DiscardCreatedAssetByObjectPath` takes an *object path*, detaches from the standalone/public
roots, renames into the transient package and lets GC reclaim it — which is correct only for a
**never-saved, in-memory** fixture. It also drains the game-thread task queue and joins
`USoundWave` compilation first, for reasons its header documents at length. So the migration is
per-call-site triage, not a sed:

1. Classify the 881 sites into never-saved-in-memory fixtures (migrate) and
   fixtures-with-an-on-disk-`.uasset` (cannot migrate as-is).
2. Migrate group 1 in batches, one module or one directory per commit, so a regression is
   bisectable.
3. For group 2, decide whether a third helper is needed — a safe path that also removes the
   file — or whether those fixtures should stop writing to disk in the first place, which would
   also serve `B-tests-leak-host-content`.

Do not attempt this in the same wave as anything else touching the test tree; 214 files is a
merge hazard on its own.

## Fix

**Verdict: IN-REVIEW — the helper rewrite supersedes the migration.** The 958-site triage is
obsolete: `CleanupTestAsset` itself no longer reaches `ObjectTools::ForceDeleteObjects`, so all
callers are fixed at the shared helper without touching their files. The full-suite profile
measured 811 force-delete calls taking 3325 s of 4102 s wall time (81%), with individual calls
drifting from 1.06 s to 6.31 s as the live UObject population grew.

`CleanupTestAsset` and `DiscardCreatedAssetByObjectPath` now share
`PwTestAssetTeardown::DiscardLoadedAssetNoGc`. The core drains no tasks, performs Blueprint-aware
flag clearing and collision-free transient renames, detaches an assetless source package, and
does not collect garbage. The package-path helper adds the game-thread drain and on-disk half;
the object-path wrapper adds the drain and retains its trailing GC for callers that require
immediate reclamation.

The former “cannot migrate disk-backed fixtures” blocker is answered in the helper: saved
fixtures have their `.uasset` deleted directly, then registry state is purged through
`FAssetRegistryModule::PackageDeleted` for a resident package or
`IAssetRegistry::ScanModifiedAssetFiles` for a non-resident one. The path probe deliberately uses
`TryConvertLongPackageNameToFilename` plus `FileSize`, not `DoesPackageExist`, because the latter
logs a warning for unmounted names and warning elevation turns that into an automation failure.

Ratchets cover both structure and behavior:

- `PinWright.infra.contract.TestAssetTeardown.BpirTransientDiscard`
- `PinWright.infra.contract.TestAssetTeardown.CleanupTestAssetHasNoForceDelete`
- `PinWright.infra.contract.TestAssetTeardown.CleanupRemovesSavedFixtureFromDiskAndRegistry`
- `PinWright.infra.contract.TestAssetTeardown.CleanupDiscardsNeverSavedFixture`

`B-suite-host-gc-crash-in-combined-group-run` remains separate: its truncations ended inside
`ForceDeleteObjects` but remain unattributed. This change removes that call from ordinary test
asset teardown without claiming it caused those historical truncations. Source review only; the
separate pipeline owns compilation and runtime verification.

## History
- `#1-force-delete-still-the-default-teardown` `OPEN` reporter — Source-only census over all eight modules; **no suite was run, no editor was started and no plugin source was modified** (tree is mid-verification on another wave). Established here: `ObjectTools::ForceDeleteObjects` is named in 14 files, including the three headers that document it as a hazard (`Tests/TestUtils.h:522-545`, `Tests/TestAssetTeardown.h:32-40`, `Utils/AssetDeletePolicy.h:7-36`); `CleanupTestAsset(` is called at **818 sites across 195 files** and `PwTestAssetTeardown::DiscardCreatedAssetByObjectPath(` at **29 sites across 9 files**; `CleanupTestAsset`'s UE 5.4 branch resolves the crash by returning without cleaning up at all (`TestUtils.h:530-533`), and its 5.5+ branch blocks on `GShaderCompilingManager->FinishAllCompilation()` purely to survive the GC that `ForceDeleteObjects` runs. **NOT established here and quoted from `B-suite-host-gc-crash-in-combined-group-run` `#4` rather than re-measured:** the per-run call counts 535/331/599, and the observation that two archived PDS truncations end on a complete line inside `ObjectTools::ForceDeleteObjects` with no crash marker or report. Both require a suite run to verify. Filed as a scoped child of that ticket, not as a history entry on it, because the parent's subject (the truncation's cause) stays open after this migration lands and because the picker never selects findings buried in another ticket's history. Severity Medium with the escalation condition recorded rather than pre-applied; explicitly declined to inherit the parent's High, since the parent's High rests on the truncation symptom and `#4` itself calls those two data points unattributable. The fix is per-call-site triage, not a mechanical replacement — the two helpers take different path forms and only one of them removes an on-disk `.uasset`; that constraint is spelled out above so the implementing agent does not discover it mid-sweep.
- `#2-bpir-safe-teardown-batch` `IN-REVIEW` developer — Source confirmed the force-delete hazard and the safe-helper precondition. Migrated all 25 package-path teardowns in the eight `Tests/Bpir` files listed above to `PwTestAssetTeardown::DiscardCreatedAssetByObjectPath(ToObjectPath(AssetPath))`; these tests construct unsaved in-memory Blueprint fixtures and never write `.uasset` files. Static checks found no remaining `CleanupTestAsset` call in `Tests/Bpir` and no whitespace errors in the scoped diff. Remaining package-path teardowns are deliberately open for per-call-site classification; disk-backed fixtures were not changed.
- `#3-record-partial-bpir-scope` `IN-REVIEW` developer — Static review confirms this batch only:
  25 never-saved transient Blueprint teardown calls in the eight listed BPIR files use the safe
  helper, with no `CleanupTestAsset` left in those files. The other 881 current sites remain unclassified;
  no broader migration or claim that `ForceDeleteObjects` caused the host truncation is made. No
  build, Unreal, automation, MCP, or runtime verification was run.
- `#4-returned-partial-scope-and-covered-batch` `OPEN` tester — Returned: the Ask requires all
  teardown sites to be classified and only allows batching the implementation; it does not make
  a 25-site partial batch a completed ticket. Current source has 881 remaining calls in 214 files
  (727 main-module plus 154 satellite-module) and 60 safe-helper calls in 17 files, excluding the
  helper definition. Added registered structural/behavioral coverage for the landed BPIR batch;
  no build, Unreal, automation, MCP, or runtime verification was run.
- `#5-compiled-blueprint-discard-collision` `OPEN` developer — A real suite crash established a
  separate defect in the safe helper: `UBlueprint::Rename` moved generated classes and CDOs into
  `/Engine/Transient`, where a retained `REINST_SKEL_` CDO already owned the target name. Updated
  the helper to remove generated classes through the engine path before a uniquely named transient
  move, and to garbage the captured generated/skeleton classes and CDOs. Extended
  `PinWright.infra.contract.TestAssetTeardown.BpirTransientDiscard` to compile with
  `SkipGarbageCollection`, assert the retained conflicting CDO, verify post-GC asset/package
  removal, and verify a second discard call is safe. Static review only; the wider migration stays
  `OPEN`, and no build, Unreal, automation, editor, MCP, or Git-mutating operation was run.
- `#6-helper-rewrite-supersedes-migration` `IN-REVIEW` developer — Rewrote `CleanupTestAsset` so
  all 958 call sites share the non-forcing `DiscardLoadedAssetNoGc` core; saved fixtures add direct
  file deletion plus `PackageDeleted` / `ScanModifiedAssetFiles`, while only the object-path wrapper
  retains an immediate GC. Added structural and saved/never-saved fixture ratchets. The measured
  profile was 3325 s of 4102 s across 811 calls, drifting 1.06 s to 6.31 s. The related suite
  truncations remain unattributed. Source review only; no build, automation, editor, MCP, or Git
  mutation was run.
