---
id: B-dumpcache-staleness-loop
title: "asset.dump_folder cache staleness loop — .dumpcache.json never written for self-dirtied or load-failed assets, causing perpetual re-dumps"
status: OPEN
severity: Medium
category: bug
tags: [asset-dump, dump-folder, cache, dumpcache, staleness, load-failed]
encounters: 1
lastSeen: 2026-07-23T00:00:00Z
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
