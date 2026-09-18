---
id: B-dump-folder-release-cadence-unbounded-by-bytes
title: "Folder-sweep release cadence is unbounded in bytes: one interval grew the working set 41 GiB"
status: IN-REVIEW
severity: High
category: bug
tags: [asset-dump, memory, performance, jobs]
---

# Folder-sweep release cadence is unbounded in bytes

`asset.dump_folder`'s release step had two triggers, and neither bounds what a
single interval *costs*:

- `AssetDumpReleaseIntervalAssets` (default 200) counts assets, not bytes. Two
  hundred UI widgets and two hundred Nanite meshes are the same number.
- `AssetDumpReleaseMemoryWatermark` (default 0.5 of physical RAM) is an absolute
  ceiling. On a host with a lot of RAM it is never reached, so it never caps the
  growth of one interval — it only saves the process from dying.

So a sweep that walks into a heavy region accumulates the whole region before the
200th asset arrives, and the deeper the region the worse it gets.

**Measured (2026-09-17, `/Game` sweep, reporting host, ~158 GiB physical):** the
gap between release step 17 and 18 ran **4.8 minutes** over ~200 heavy
Nanite/Megascans meshes, and the working set went **11.88 -> 52.63 GiB** (one
sample read 77 GiB), of which **41.87 GiB** was handed straight back by the next
collect. Peak for that run was 63.5 GB. The count trigger and the watermark were
the only two triggers that could fire, and the count is what eventually did. The
compile-deferral exemption was not the cause: it held only 25-31 packages.

Log lines named no trigger, so the cadence could not be diagnosed from a log
alone — "release step 18 requested" reads the same whichever knob fired.

## History
- `#1-measured-and-fixed` `IN-REVIEW` developer — Added a third, bytes-based trigger: `UPinWrightSettings::AssetDumpReleaseGrowthGiB` (default 8, 0 disables) releases once the working set has grown that much since the working set measured AFTER the last release collect (`FAsyncFolderDumpState::ReleaseWorkingSetBaselineBytes`, seeded at the sweep's first releasable tick and advanced on every collect, including a timed-out one). Evaluated alongside the existing two by `AssetDumpHandler::DecideDumpReleaseTrigger`, which returns *which* trigger fired instead of a bare bool; `ShouldRunDumpReleaseStep` stays as the count/watermark-only forwarder for the automation-suite maintenance caller. `DumpReleaseMinAssetsBetweenSteps` (25) is now the shared floor for both byte triggers, so neither can thrash. The release-step log line now carries `trigger=count|watermark|growth|final` and the growth since the last release. Provenance for the default is recorded at the assignment in `PinWrightSettings.cpp` and in `docs/wiki-src/asset.md`; no aspect version bump (dump output is unchanged). Unit tests: `PinWright.asset.dump.AsyncFolderDump.ReleaseStepGrowthTrigger` covers the fire case, both failure directions (growth below the limit, growth disabled by 0), the no-baseline and shrinking-working-set cases, the floor, and trigger precedence.
- `#2-live-verification` `IN-REVIEW` developer — Live headless sweeps with `force:true` into a throwaway `outRoot`, same host. A heavy Nanite/Megascans content folder (911 assets, 911 dumped, 0 skips, 0 `ASSET_COMPILE_TIMEOUT`): 5 release steps, growth fired once, max `workingSetBefore` 26.00 GiB, peak RSS 25.5 GiB, every interval's growth <= 8.01 GiB. a second heavy content folder (458 assets, 458 dumped, 0 skips, 0 timeouts) in the same editor: the growth trigger fired first at **68 assets** (+8.13 GiB) instead of waiting for asset 200, max `workingSetBefore` 32.80 GiB, peak RSS 32.3 GiB. Compare the 2026-09-17 baseline: a single interval grew 40.75 GiB and the run peaked at 63.5 GB. One defect was caught by this live pass and fixed before it shipped: the baseline was seeded but never advanced after a collect, which made the trigger fire every 25 assets forever once it first crossed the limit (20 release steps over the same folder instead of 5). Tester still required for DONE.
