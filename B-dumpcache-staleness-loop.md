---
id: B-dumpcache-staleness-loop
title: "asset.dump_folder cache staleness loop — .dumpcache.json never written for self-dirtied or load-failed assets, causing perpetual re-dumps"
status: IN-REVIEW
severity: Medium
category: bug
tags: [asset-dump, dump-folder, cache, dumpcache, staleness, load-failed]
encounters: 1
lastSeen: 2026-09-02T00:00:00Z
---
# asset.dump_folder cache staleness loop — .dumpcache.json never written for self-dirtied or load-failed assets, causing perpetual re-dumps

The sweep's sole cache writer `WriteDumpCacheForSuccessfulBaseline`
(`Source\PinWright\Private\Handlers\Asset\AssetDumpHandler.cpp:857`, call site
`:1473`) is silently vetoed by `IsCacheEligibleSource` (`:851-855`, predicate
`Kind != Uncached && !bPackageIsDirty`) whenever the package is dirty at write
time. Blueprint-class assets dirty THEMSELVES via compile-on-load during the
dump's own `LoadObject` (`:1294`), and the dirty-guard restore (predicate
`dirty && !BaselineDirty`) runs only at tick end (`:1169`) — so at cache-write
time the self-dirtied package always looks dirty. The eligibility gate
pre-dates the guard (guard added in `7c7350bd`) and was never reconciled with
it: the guard makes the dump output correct, but the gate still refuses the
cache record.

Separately, packages whose inner asset name differs from the package tail
never get a cache record either. Confirmed case: "Bake Out Materials" outputs
(engine `MeshMergeUtilities.cpp:388`) — package `M_<mesh>_<mat>_<GUID>`
containing `MI_…`/`T_…` assets. `MakePendingDumpPath` (`:1545`) returns
`Data.PackageName` only, so the bare-package-name `LoadObject` fails →
ASSET_LOAD_FAILED skip stubs (`:1090-1164`) that skip the cache write too.

Symptoms:

- **Perpetual re-dump loop** — measured in PDS: exactly 21 of 8192 `/App`
  dump dirs lack `.dumpcache.json` and re-dump on every sweep (17
  Blueprints/widgets + 4 bake packages), no matter how often they are swept.
- **Order-unstable stub content** — the load-failed stub's `className` is
  taken from order-unstable `Assets[0]` (`:1121`), so it flips across sweeps
  and churns the committed dump mirror.

Severity rubric: no wrong dump data (output is correct); impact is wasted
editor churn on every sweep plus dump-mirror noise from flapping stubs — a
soft cost with no workaround inside the tool, on a method that runs in most
sessions — Medium.

Related: `B-bpir-clone-graph-entry-name-leak` made these re-dumps byte-stable
(no more spurious `bpir.txt` diffs); this ticket removes the re-dumps
themselves.

**Workaround:** none inside the tool — the 21 dirs re-dump every sweep; for
mirror churn, ignore stub `className` flips on the 4 bake packages.
**Fix:** (in flight, same-day, single-shot) baseline-aware eligibility —
thread the sweep's `BaselineDirty` into the `IsCacheEligibleSource` gate so it
uses the same predicate as the tick-end guard; object-path loading with
deterministic primary-asset selection for packages whose inner asset name
differs from the package tail; cache records for skip stubs (additive
`skipped` field, NO cacheVersion bump); veto-reason and staleness-reason log
lines.

## History
- `#1-initial-repro` `OPEN` reporter — `.dumpcache.json` never written for two asset classes, causing a perpetual re-dump loop: (a) Blueprint-class assets dirty themselves via compile-on-load during the dump's own `LoadObject` (AssetDumpHandler.cpp:1294) and the sole cache writer `WriteDumpCacheForSuccessfulBaseline` (:857, call :1473) is vetoed by `IsCacheEligibleSource` (:851-855, `Kind != Uncached && !bPackageIsDirty`) because the dirty-guard restore (:1169, `dirty && !BaselineDirty`) runs only at tick end — gate pre-dates the guard (7c7350bd), never reconciled; (b) packages whose inner asset name differs from the package tail (Bake Out Materials outputs, engine MeshMergeUtilities.cpp:388, `M_<mesh>_<mat>_<GUID>` containing `MI_…`/`T_…`) fail the bare-package-name LoadObject (`MakePendingDumpPath` :1545) → ASSET_LOAD_FAILED stubs (:1090-1164) with no cache record and order-unstable `className` from `Assets[0]` (:1121). Measured: exactly 21 of 8192 /App dump dirs lack `.dumpcache.json` and re-dump every sweep (17 Blueprints/widgets + 4 bake packages). Fix in flight: baseline-aware eligibility (thread sweep `BaselineDirty` into the gate), object-path loading with deterministic primary-asset selection, cache records for skip stubs (additive `skipped` field, no cacheVersion bump), veto/staleness-reason log lines.
- `#2-baseline-eligibility-objectpath-stub-cache` `IN-REVIEW` developer — Fixed in plugin commit `715f3c8d`: baseline-aware cache eligibility (sweep `BaselineDirty` threaded into the `IsCacheEligibleSource` gate — same predicate as the tick-end guard, so self-dirtied Blueprint packages now write cache records), object-path asset loading with deterministic primary-asset selection (bake packages with mismatched inner names now load and dump instead of stubbing), and cache records for skip stubs (additive `skipped` field, no cacheVersion bump). Full automation suite green on UE 5.8, 0 failures: new tests BaselineAwareEligibility, MidSweepDirtyStillWritesCache, BaselineDirtySkipsCacheWrite, MismatchedInnerNameDumps, SkipStubCacheRecord, SelectPrimaryAssetData all pass, plus updated FolderSweepPendingObjectPathShape and legacy SkipStub. Live two-sweep convergence check on the PDS project (second sweep must hit cache on all 21 formerly-loop dirs) is running now — result to be recorded at DONE time.
- `#3-field-evidence-loop-gone` `IN-REVIEW` reporter — Additional field evidence, **not a verification**; status and `encounters` deliberately unchanged (this is corroboration of the fix, not a re-hit of the bug). After a forced full `/Game` + `/App` sweep of the PDS project (2026-09-02, plugin 0.7.0, UE 5.8.1, 30,807 assets), the symptom this ticket filed is absent at scale: **0 of 30,807 dump dirs lack a `.dumpcache.json`** — the "exactly 21 of 8192 `/App` dirs re-dump every sweep" population from `#1` is empty. That is the shape the pending live convergence check in `#2` was looking for, but it is not that check: the sweep ran with `force`, so it proves cache records are now *written* for the formerly-vetoed classes (self-dirtied Blueprint/widget packages and mismatched-inner-name bake packages), not that a second non-forced sweep *hits* them. `.dumpcache.json` is correctly gitignored in the host mirror with zero `git status` entries. The order-unstable stub `className` half is untested here — no load-failed stubs remained after the sweep. One adjacent finding the sweep did surface, filed separately as `B-dump-reconcile-blind-to-metaless-dirs`: the inverse shape (126 dirs holding a cache record and no `meta.json`, 121 of them with a current fingerprint), which this ticket's fix does not address.
- `#4-fix-survived-upstream-pull` `IN-REVIEW` reporter — Bookkeeping only: the host plugin clone was pulled 398 commits forward (`b16f0f2b` -> `347826a6`) after `#4` was written, so this confirms the pull is not a confounder for the pending verification. Status, severity and `encounters` unchanged. Fix commit `715f3c8d` is an ancestor of `347826a6`, and its three parts are present at HEAD: baseline-aware eligibility — `IsCacheEligibleSource(Source, PackageName, BaselineDirty)` now returns `Kind != Uncached && (!bPackageIsDirty || !BaselineDirty.Contains(FName(*PackageName)))` (`AssetDumpHandler.cpp:1960-1971`), i.e. the same predicate as the tick-end guard, with the comment naming compile-on-load re-dirtying; the sweep call site passes `State.BaselineDirty` and logs the veto through `LogCacheEligibilityVeto` (`:1773-1781`, `:2001-2003`); skip stubs now write cache records on the eligible branch (`:1782-1785`). No commit in the pulled range reverts or re-narrows the gate. The live two-sweep convergence check promised in `#2` is still the outstanding item and is unaffected.
