---
id: F-localization-gather-compile
title: "No localization namespace: agents cannot run Gather/Compile, so manifest/archive/locres regeneration is a manual Dashboard step"
status: OPEN
severity: Medium
category: feature
tags: [localization, gather, compile, locres, archive, manifest, commandlet]
encounters: 1
lastSeen: 2026-07-14T08:20:00Z
---

# No localization namespace: Gather/Compile can only be done by hand in the Localization Dashboard

PinWright has **no localization surface at all** — no `localization.*` namespace, and no
`GatherText` / `ULocalizationTarget` handling anywhere in `Source/` (grep for
`GatherText|LocalizationCommandlet|LocalizationTarget` returns nothing). So any workflow that
invalidates the Unreal localization data ends in a hard stop: the agent must ask the user to open the
Localization Dashboard and click Gather/Compile by hand.

That is the only step in the whole asset pipeline with no automation path. Everything else
(create/reparent/compile/save assets, dump, decompile, drive PIE) is scriptable.

## Concrete encounter that filed this

Merging `origin/master` into a feature branch produced conflicts in
`Content/Localization/Game/{Game.manifest, en|ru|keys/Game.archive, *.locres}`. These are UTF-16 JSON
(+ compiled binaries) that git treats as binary, so the merge was resolved by taking one side —
which silently dropped 4 keys the branch had added
(`DA_{ClimbMap,DunesOasis,MeadowMap}_Freeflight.TileTitle`, `SaveButton`).

Recovering them requires a **Gather** (to re-add the keys to the manifest from the source assets),
then re-merging the preserved translations, then a **Compile** (to regenerate `.locres`). Steps 1 and
3 cannot be done through the MCP, so the work stalls on a manual user action even though the data and
the intent are fully known.

Note the host project's `localization` skill separately forbids agents from *deciding* to gather or
compile ("the user must manually run Gather/Compile in the Unreal Localization Dashboard"). That is a
policy about **when**, and it is fine — but it currently doubles as the only reason the workflow
can't proceed even when the user explicitly asks for it, because there is no **how**.

## Proposed

A `localization` namespace wrapping the in-editor localization target APIs (the same ones the
Dashboard drives — `ULocalizationTargetSet` / `ULocalizationTarget` +
`LocalizationCommandletTasks::{GatherTextForTargets, CompileTextForTargets, ExportTextForTargets,
ImportTextForTargets}` in `LocalizationCommandletTasks.h`), rather than shelling out to a second
editor process with `-run=GatherText` (which would fight the running editor for file locks):

- `localization.targets` — list configured targets (name, cultures, word counts, config paths).
- `localization.gather { target }` — run the gather pipeline; returns keys added/removed.
- `localization.compile { target, cultures? }` — regenerate `.locres`.
- `localization.export` / `localization.import { target, culture, path }` — archive round-trip for
  external translation tooling.
- All long-running: return a job id and report through `system.job_status` (gather on a large project
  is minutes, not seconds).

## Acceptance

- `localization.gather` on a target with a newly-added `NSLOCTEXT`/`LOCTEXT` key in an asset or C++
  file adds that key to `Game.manifest` and to every culture's `Game.archive` with an empty
  translation, matching byte-for-byte what the Dashboard's Gather produces.
- `localization.compile` regenerates `.locres` for the requested cultures; a PIE run with that culture
  shows the translated string.
- Both work against the live editor without spawning a competing editor process.
- Failure modes (missing target, bad culture code, gather config errors) return structured errors, not
  a silent no-op.

## History

- `#1-filed` **OPEN** (Reporter) — Filed after a master merge dropped 4 localization keys and the
  recovery stalled: no MCP path exists to Gather/Compile, so regeneration requires a manual
  Localization Dashboard run by the user. Verified the gap by grepping the plugin source (no
  `GatherText`/`LocalizationTarget` references) and the generated RPC reference (no `localization`
  namespace).
