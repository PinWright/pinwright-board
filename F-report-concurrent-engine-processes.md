---
id: F-report-concurrent-engine-processes
title: "Nothing in PinWright reports that another engine process is alive on the machine, and a stray UnrealEditor-Cmd made every single GPU pass ~40% more expensive at identical resolution and identical draw calls (P3 ShadowDepths 7.12 ms vs 3.90 ms) — system.identity reports this process only, and the transport notices a sibling editor solely by port collision, which cannot fire for a different project"
status: OPEN
severity: Medium
category: feature
tags: [system, identity, performance, measurement, profiling, contention, concurrent-process, gpu, readback, missing-field, reproducibility, shared-machine, editor-cmd]
encounters: 1
costly: 1
lastSeen: 2026-08-30T18:00:00+03:00
---

# A second engine process poisons every number and nothing reports it

Measured on host `EAContentExamples58` (UE 5.8, `/Game/Maps/PW_VegetationTest`) during a vegetation
performance pass, recorded at `Docs/map/vegetation-performance.md:164-167`:

> **Another process on the machine can slow the whole GPU uniformly.** A stray `UnrealEditor-Cmd`
> made every single GPU pass ~40% more expensive at identical resolution and identical draw calls
> (P3 ShadowDepths 7.12 ms vs 3.90 ms). Check `Get-Process` before baselining; discard any baseline
> taken with a second engine process alive.

The two guards a measuring caller *can* reach both pass clean in that state: the resolution is
identical, so `E-viewport-info-no-render-resolution`'s field would not fire; the draw calls are
identical, so a same-scene guard would not fire. **Every observable a verb reports is correct and
the number is 40% wrong.** The only tell is on the machine, outside the process, and PinWright
cannot see it.

This is also the confounder that **interleaving does not cancel**. A second process appearing and
disappearing is fast and categorical, not slow and gradual, so `F-interleaved-ab-measurement`'s
method — which cancels drift by alternating — will happily average one A inside the contention and
one outside it. The two tickets are complements.

## Nothing in the plugin enumerates a process

Swept the whole plugin tree, not only `Source/`, for `FPlatformProcess::IsApplicationRunning`,
`EnumProcesses`, `CreateToolhelp32Snapshot`, `OpenProcess`, `GetProcess`, `tasklist`: **zero
occurrences in `Source/`**. Everything that touches OS processes *creates* one and never enumerates
one:

- `Private/Handlers/Editor/EditorLaunchHandler.cpp:112` — `FPlatformProcess::CreateProc`, launching
  a standalone game.
- `Private/Handlers/BuildTools/ProcPollBind.h:20` — `FPlatformProcess::CreateProc`, spawning build
  tools.
- `Private/Handlers/Localization/LocalizationHandler.cpp:156` — `Process->GetProcessArguments()` on
  a commandlet *config object*, not an OS process.

`tasklist` / `Get-Process` appear only in **prose telling a human to check by hand** —
`Docs/wiki-src/unattended.md:129`, `Public/PinWrightSubsystem.h:63`,
`Public/Transport/BindRetryPolicy.h:13`, `Private/Tests/Transport/TestBindRetryPolicy.cpp:9`,
`ci/demo_smoke_mcp.py:265`. The plugin already knows this question matters and answers it by
telling the operator to leave and use a shell.

## What exists, and exactly how far short it falls

**`system.identity` reports this process only** — pid, per-boot GUID, project, engine version,
plugin stamp, executable, command line, port. Its own wiki page frames a second editor purely as a
*misrouting* hazard (*"Two editors of the same project derive the SAME MCP port... pass
`_expect_editor`"*), which is a correctness concern, not a measurement one. Nothing in it looks
outward.

**The transport notices a sibling only by port collision**, i.e. only for *the same project*. That
is the mechanism behind `B-port-conflict-false-active`, `B-no-editor-identity-handshake` and
`B-failed-bind-no-retry`. The process that cost the baseline here was an `UnrealEditor-Cmd` —
a different binary, on a different project, deriving a different port. **It could not have
collided, and so it could not have been noticed.** That is the whole gap in one sentence.

## Ask

**A `system.identity` field, or a small verb beside it, reporting other engine processes on the
machine.** Minimum useful shape:

```
otherEngineProcesses: {
  measured: true,
  count: 1,
  processes: [ { pid, name: "UnrealEditor-Cmd", startedAt, commandLineHint } ]
}
```

Notes for whoever takes it:

- **`measured` must be a real field, not implied by an empty array.** A permissions failure or an
  unimplemented platform has to be distinguishable from "nothing else is running" — that
  distinction is the entire value of the field, since a caller who reads `count: 0` from a probe
  that never ran is worse off than one who was told nothing.
- **Name-matching is enough and exactness is not required.** Any process whose image name begins
  `UnrealEditor` (`UnrealEditor.exe`, `UnrealEditor-Cmd.exe`, `UnrealEditor-Win64-DebugGame.exe`)
  plus `UnrealBuildTool`, `UnrealLightmass`, `ShaderCompileWorker` and `UnrealPak` covers what
  actually contends. A false positive costs a caller one look; a false negative costs a wrong
  conclusion.
- **`ShaderCompileWorker` deserves calling out separately** — it is spawned by *this* process and is
  contention the caller can wait out rather than a foreign process they must go and close. Reporting
  it undifferentiated would train callers to ignore the field.
- **The honest place for it is wherever measurement receipts go.** If
  `F-performance-frame-time-statistics` lands a measurement verb, this belongs in its response as
  well as on `system.identity`, for the same reason the render resolution does: a frame time whose
  provenance is unknown is not a result.
- Cheapest implementation is `FPlatformProcess::FProcEnumerator` (Windows), which the engine already
  ships; nothing here needs a new dependency.

## Distinct from

- **`E-viewport-info-no-render-resolution`** (OPEN, Medium) — **cites this exact evidence and files
  nothing against it.** Its `## Measured` block (`:74`) reads: *"The class of error is not
  hypothetical on this project... a stray `UnrealEditor-Cmd` process made every GPU pass ~40% more
  expensive at *identical resolution and identical draw calls* (P3 ShadowDepths 7.12 ms against
  3.90 ms). That one at least had a cause the agent could find; a resolution change hides inside the
  number the tool reports as correct."* It uses the case as a rhetorical sibling to argue for a
  resolution field. **Stated up front here so a reviewer does not read this as double-filing:** that
  ticket asks for the denominator and explicitly contrasts it with this case; nothing on the board
  asks for the process check itself. The two confounders are also opposite in kind — a resolution
  change is silent and internal, a second process is loud and external, and neither field detects
  the other.
- **`F-interleaved-ab-measurement`** (OPEN, Medium) — the sibling from this session. Interleaving
  cancels *slow* confounders; a process appearing mid-run is not one. Neither closes the other.
- **`F-performance-frame-time-statistics`** (OPEN, Medium) — the measurement primitive. This is a
  provenance field on whatever that returns.
- **`B-port-conflict-false-active`**, **`B-no-editor-identity-handshake`**,
  **`B-failed-bind-no-retry`** — transport correctness for two editors of *the same project*
  colliding on one MCP port. Blind by construction to a different-project process, which is the case
  that cost the baseline.

Board-wide sweep for `concurrent`, `second process`, `contention`, `Get-Process`, `tasklist`,
`another editor` found nothing owning this. **Unowned.**

## Severity

**Medium**, on the rubric's Medium band in its readback clause: *"a readback omits a field and
forces a fallback"*. The fallback is literal and is written into this project's own notes — run
`Get-Process` in a shell before every baseline and discard anything taken with a second engine
process alive.

**Not High.** No verb returns a false value: the frame time reported under contention is the frame
time that actually occurred. What is missing is the provenance needed to know the number does not
generalise. That is an omission, not a lie.

**Reach modifier declined, in both directions, because they cancel.** Arguing down: a stray engine
process is a rare edge condition, seen once in this session. Arguing up: when it does occur it is
undetectable in-band and silently poisons *every* number taken while it lasts, and the two guards a
caller can already reach (resolution, draw calls) both read clean — so the rare event has no
ceiling on its damage. Medium stands unmodified.

severity rationale: impact=readback omits a field and forces an out-of-process fallback (Medium)
x reach=rare but undetectable in-band, arguments cancel -> Medium

## Not done

Plugin source was not modified and no new measurement was taken. The 40% / 7.12-vs-3.90 ms figure
is cited from `Docs/map/vegetation-performance.md:164-167` on the host that produced it and was not
reproduced for this filing — reproducing it would mean deliberately starting a second engine
process on a machine several agents share. The plugin-side negatives (no process enumeration
anywhere in `Source/`) were re-derived by grep at HEAD rather than relayed.

## History
- `#1-no-concurrent-process-report` `OPEN` reporter — Filed from the final triage sweep of a
  vegetation performance session on host `EAContentExamples58` (UE 5.8, editor build 13:32, plugin
  commit `d8f1bc32`). Evidence cited from `Docs/map/vegetation-performance.md:164-167` (hazard 3): a
  stray `UnrealEditor-Cmd` made every GPU pass ~40% more expensive at identical resolution and
  identical draw calls, P3 ShadowDepths 7.12 ms against 3.90 ms, with the recorded remedy being to
  run `Get-Process` by hand and discard the baseline. **Plugin side, re-derived at HEAD by grep:**
  `FPlatformProcess::IsApplicationRunning`, `EnumProcesses`, `CreateToolhelp32Snapshot` and
  `OpenProcess` have **zero occurrences** in `Plugins/PinWright/Source/`; the only process work is
  creation — `Handlers/Editor/EditorLaunchHandler.cpp:112` and `Handlers/BuildTools/ProcPollBind.h:20`
  both `FPlatformProcess::CreateProc`, while `Handlers/Localization/LocalizationHandler.cpp:156`
  operates on a commandlet config object rather than an OS process; `tasklist` / `Get-Process` appear
  only in prose instructing a human to check by hand (`Docs/wiki-src/unattended.md:129`,
  `Public/PinWrightSubsystem.h:63`, `Public/Transport/BindRetryPolicy.h:13`,
  `Private/Tests/Transport/TestBindRetryPolicy.cpp:9`, `ci/demo_smoke_mcp.py:265`). `system.identity`
  reports this process only, and the transport detects a sibling editor solely by MCP port collision
  on the same project — which cannot fire for an `UnrealEditor-Cmd` on a different project, and is
  why this went unnoticed. **Dedup declared up front rather than left for a reviewer:**
  `E-viewport-info-no-render-resolution` (OPEN, Medium) already quotes this exact 40% /
  7.12-vs-3.90 ms evidence at its `:74`, as a rhetorical sibling arguing for a render-resolution
  field; it files nothing against the process case and asks for a different field, and the two
  confounders are opposite in kind. Ask: an `otherEngineProcesses` block on `system.identity` (and on
  any measurement receipt) with a real `measured` flag, name-prefix matching over `UnrealEditor*`
  plus `UnrealBuildTool` / `UnrealLightmass` / `ShaderCompileWorker` / `UnrealPak`, and
  `ShaderCompileWorker` distinguished because it is this process's own child rather than a foreign
  contender; `FPlatformProcess::FProcEnumerator` is sufficient and adds no dependency. Severity
  Medium on the readback-omission band; High declined because the reported frame time is true, only
  its provenance is missing; reach modifier declined with the arguments cancelling — rare, but
  undetectable in-band and unbounded in damage while it lasts.
