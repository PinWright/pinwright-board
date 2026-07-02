---
id: E-scalability-console-escape-hatch-misleading
title: "performance.set_scalability summary wrongly lists console 'scalability N' as an ECVF_SetByConsole pin — misleads callers into a no-op escape-hatch attempt"
status: OPEN
severity: Low
category: ergonomic
tags: [performance, scalability, sg-cvar-priority-pin, set-scalability, docs]
encounters: 1
lastSeen: 2026-07-02T12:10:39.8312929+03:00
---

# `set_scalability`'s "prior console `scalability N` pins at ECVF_SetByConsole" claim is wrong and sends callers down a dead-end escape hatch

`performance.set_scalability`'s registered summary (which renders into the wiki page) explains why a
requested level can silently fail to land on some `sg.*` groups. Verbatim
(`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp` ~L277):

> "... A group already pinned higher (by a device profile, a config, or a prior console
> `'sg.<Group> N'` / `'scalability N'` at ECVF_SetByConsole) is therefore NOT overwritten and keeps
> its pinned value."

The claim lumps the **aggregate** `scalability N` console command in with the **per-CVar**
`sg.<Group> N` console command as "at ECVF_SetByConsole." That is factually wrong for the aggregate
form: `scalability N` is handled by `Scalability::ProcessCommand` → `SetFromSingleQualityLevel` →
`Scalability::SetQualityLevels`, which writes every `sg.*` group at **ECVF_SetByScalability** — the
LOWEST settable priority, the exact same priority `set_scalability` itself uses. Only the direct
per-CVar console form `sg.<Group> N` actually writes at ECVF_SetByConsole (the highest priority).

Two practical consequences of the wrong phrasing:
1. A prior `scalability N` does **not** "pin higher" — a later `set_scalability`/`scalability`
   write at equal priority overwrites it. So `scalability N` is a wrong example of a higher pin.
2. Read the other direction, the summary implies `scalability N` writes at console priority and can
   therefore be used to **force** a level over an existing pin. It cannot: routing through
   `SetQualityLevels` at ECVF_SetByScalability, it loses to any config / device-profile /
   `ConsoleVariablesIni` / direct-console pin. The real escape hatch is a per-CVar `sg.<Group> N`
   (ECVF_SetByConsole) or `editor.set_preferences` at console/code priority.

## Process friction this caused (this task)

A perf-baseline task requested Medium (level 1). This host's `DefaultEngine.ini [ConsoleVariables]`
pins the `sg.*` groups at ECVF_SetByConsoleVariablesIni, so the first `set_scalability {level:1}`
correctly reported `requestedLevelApplied:false` (10/11 groups still 0 — that readback is the
already-fixed `B-set-scalability-no-sg-update`). Following the summary's implied escape hatch, the
agent then ran `editor.console_command "scalability 1"` — it no-op'd — and re-ran `set_scalability`,
which **still** reported `requestedLevelApplied:false`. Only after hand-rolling all 11 `sg.*` groups
via `editor.set_preferences` (console/code priority) did `set_scalability` finally report
`requestedLevelApplied:true`. Net waste attributable to the wrong guidance: 1 `editor.console_command`
+ 1 `set_scalability` re-run, plus the reasoning to work out from first principles why `scalability N`
routes at SetByScalability. The agent's own note: *"the aggregate scalability command routes through
Scalability::SetQualityLevels at ECVF_SetByScalability, the same low priority set_scalability uses, so
it also can't beat the pin; only a DIRECT per-CVar console set writes at ECVF_SetByConsole."*

Note: the ini pin that forced the pin-collision is partly this fuzz host's `DefaultEngine.ini`
artifact, but device profiles and project config pin `sg.*` on real projects too, and the docs
inaccuracy (the `scalability N` priority claim) is host-independent.

**Sibling to correct too:** `B-set-scalability-no-sg-update`'s **Workaround** prose currently repeats
the same error — "drive scalability via `system.console_command {command:"scalability <N>"}` (which
writes at console priority and therefore lands over any existing pin)." That workaround does not work
against a higher pin, for the same reason; whoever corrects this should fix that line too.

**Workaround:** to force `sg.*` groups over an existing higher-priority pin, set each `sg.<Group> N`
directly (ECVF_SetByConsole) or via `editor.set_preferences` — not the aggregate `scalability N`.

**Fix:** (a — docs, primary) correct the `set_scalability` registered summary in
`PerformanceHandler.cpp`: drop `scalability N` from the ECVF_SetByConsole example (it writes at
ECVF_SetByScalability, same as this RPC), and name the real escape hatch (per-CVar `sg.<Group> N` /
`editor.set_preferences`) for callers who must beat a pin; mirror any of this that surfaces in the
`docs/wiki-src/performance.md` overlay. (b — optional ergonomic) `set_scalability` could grow an
optional `force` flag that writes the `sg.*` groups at ECVF_SetByConsole, so a caller who genuinely
wants the level to win doesn't have to hand-roll 11 CVars via `editor.set_preferences`.

severity rationale: impact=pure docs/naming friction (misleading guidance, recoverable) × reach=normal-but-not-every-session perf-tuning path -> Low

## History
- `#1-initial-audit` `OPEN` reporter — set_scalability's registered summary lists a prior console `scalability N` alongside `sg.<Group> N` as pinning "at ECVF_SetByConsole"; the aggregate command actually routes through `Scalability::SetQualityLevels` at ECVF_SetByScalability (lowest priority), so it neither pins higher nor escapes an existing pin. A perf-baseline task (focus performance.show_fps) followed the implied escape hatch: `editor.console_command "scalability 1"` no-op'd against the host's ini-pinned `sg.*`, a `set_scalability` re-run still read `requestedLevelApplied:false`, and only per-CVar `editor.set_preferences` on all 11 `sg.*` groups made the level land — ~2 wasted calls. Distinct from the readback fix in `B-set-scalability-no-sg-update` (that's a code readback, landed); this is the docs/escape-hatch inaccuracy, which also appears in that ticket's Workaround prose. Fix: correct the summary (name per-CVar `sg.<Group> N` / `editor.set_preferences` as the real force path) and/or add an optional `force` flag.
