---
id: F-editor-build-midbuild-edit-guard
title: "editor_build reports succeeded even when a source or header changed while it ran, and there is no build lease, so a mid-build header edit can leave a module compiled against two class layouts that crashes every later launch"
status: OPEN
severity: High
category: feature
tags: [proxy, editor_build, editor_build_status, ubt, build, shared-checkout, multi-agent, stale-objects, editor-crash]
encounters: 1
costly: 1
lastSeen: 2026-09-29T00:00:00Z
---

# Mid-build source edits are invisible to editor_build

Observed 2026-09-29 on `X:\src\unreal\unreal-fpv` (UE 5.8), several agents editing one checkout: an
`App` plugin header was edited while a build of the editor target was running. Some translation units
had already compiled against the old class layout, others against the new one, and UBT then
considered the module up to date. Every editor launch afterwards crashed at startup with
`EXCEPTION_ACCESS_VIOLATION reading address 0xbf800010` in
`FObjectInstancingGraph::InstancePropertyValue` during a Blueprint CDO load. It was fixed only by
touching the header and rebuilding. (Which launcher ran that build was not recorded; the request
applies to the new `editor_build` because it is now the sanctioned build path, see
`F-python-capped-launcher` `#4`.)

`editor_build` / `editor_build_status` (`Content/Python/mcp_proxy.py:3112`, `:3212`) grade a build on
UBT's `Result:` line and the exit code only. Nothing compares source timestamps against the build's
start, and nothing tells other sessions that a build is in progress, so `succeeded` is reported for
exactly this inconsistent state.

**Asked for:**
- Record the build start time. When the build ends, scan the target's source roots (project
  `Source/` and project plugins' `Source/`) for `.h` / `.cpp` / `.inl` / `.Build.cs` files modified
  after that start; if any, report `sourcesChangedDuringBuild: [...]` and a non-success status such
  as `stale` (or re-run the build once), never plain `succeeded`.
- A documented build lease: while `editor_build` runs, a lock file (owner, reason, start time, log
  path) that `editor_list` / `editor_build_status` expose, so concurrent sessions can see a build is
  in flight and hold header edits until it ends; a second `editor_build` should report the holder
  instead of racing.
- Document the recovery for the crash signature above (touch the changed header, rebuild) next to
  `editor_build` in `docs/wiki-src/mcp-transport.md` Building.

**Workaround:** do not edit headers while any build runs; after a suspicious startup AV, touch the
recently edited headers and rebuild.

## History
- `#1-header-edited-mid-build` `OPEN` reporter - Observed on `X:\src\unreal\unreal-fpv` (UE 5.8): an `App` header edited during a build left mixed-layout objects that UBT treated as up to date; every editor launch then crashed (`EXCEPTION_ACCESS_VIOLATION` 0xbf800010 in `FObjectInstancingGraph::InstancePropertyValue`, BP CDO load) until the header was touched and the module rebuilt. `editor_build_status` grades only UBT's `Result:` and the exit code (`mcp_proxy.py:3212-3260`) and has no source-timestamp check or lease. severity rationale: impact=High (silent false-success: a `succeeded` build whose output crashes every launch) x reach=multi-agent shared-checkout builds, no modifier -> High. costly=1 (several editor launches crashed and a rebuild round was lost before diagnosis).
