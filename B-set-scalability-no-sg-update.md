---
id: B-set-scalability-no-sg-update
title: "performance.set_scalability returns a bare 'Scalability set' with no read-back — a sg.* group pinned at a higher CVar priority is silently dropped and the success can't reveal it"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [performance, scalability, sg-cvars, readback, set-scalability]
---

# `performance.set_scalability` reports success without reading back what landed

`performance.set_scalability {level:N}` calls `Scalability::SetFromSingleQualityLevel(Level)`
→ `Scalability::SetQualityLevels(Quals)` → `Scalability::SaveState`, then returns a bare
`{"message":"Scalability set"}` with no read-back.

`Scalability::SetQualityLevels` **does** drive the canonical `sg.*` group CVars
(`sg.ViewDistanceQuality`, `sg.ShadowQuality`, `sg.TextureQuality`, ...). Engine source
`Engine/Private/Scalability.cpp` `SetQualityLevels` calls
`SetQualityLevelCVar(CVarViewDistanceQuality, ...)`, `SetQualityLevelCVar(CVarShadowQuality, ...)`,
etc. for every group, and `CVarViewDistanceQuality` *is* the `sg.ViewDistanceQuality` console
variable. So the earlier claim that this RPC "never writes `sg.*`" is **false**, and so is the
claim that it "contradicts `E-baseline-no-sg-setall`" — E-baseline correctly states that
set_scalability drives `sg.*` via `SetFromSingleQualityLevel → SetQualityLevels`; nothing there
needs correcting.

The real, smaller gap is ergonomic and concerns CVar **priority**, not a missing write.
`SetQualityLevelCVar` ends in `CVar->Set(DesiredValue, ECVF_SetByScalability)` — the LOWEST
settable priority. UE's priority system keeps a higher-priority value when a lower-priority
`Set` arrives. So if a group is already pinned higher — by a device profile, a project config,
or a prior console `sg.<Group> N` / `scalability N` (both `ECVF_SetByConsole`, the highest) —
the scalability write for that group is silently shadowed and the requested level does **not**
land for it. The handler then returns an unconditional `"Scalability set"` with no read-back,
so a caller cannot tell which (if any) groups actually reached the requested level.

The original "live repro" that reported `sg.*` staying at 2/3 after `set_scalability` was
self-contaminated: its earlier `system.console_command {sg.ShadowQuality 2}` /
`{sg.ViewDistanceQuality 3}` pinned those groups at console priority FIRST, which then correctly
shadowed the later lowest-priority scalability write. That is expected, documented engine
behavior — not a broken RPC. In a clean session (no prior console pin) `set_scalability {N}`
does move the `sg.*` groups.

Handler (`Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp`,
`performance.set_scalability`):

```cpp
int32 Level = Ctx.GetInt(TEXT("level"), 3);
Scalability::FQualityLevels Quals;
Quals.SetFromSingleQualityLevel(Level);
Scalability::SetQualityLevels(Quals);
Scalability::SaveState(GEditorIni);
Ctx.SendSuccess(TEXT("Scalability set"));   // bare success, no read-back
```

**Impact:** an agent that sets a scalability level via this RPC gets no read-back at all and a
clean success that cannot distinguish "the requested level landed on every group" from "a
higher-priority pin silently dropped it on some groups." Pure discoverability friction on a
normal performance-tuning path. Low.

**Workaround:** read the `sg.*` groups yourself after the call via
`system.console.search {query:"sg."}`, or drive scalability via `system.console_command
{command:"scalability <N>"}` (which writes at console priority and therefore lands over any
existing pin).

**Fix:** after `Scalability::SetQualityLevels`, read the effective `sg.*` group CVars back and
return them in the response — `requestedLevel` + an `appliedGroups` array of per-group effective
values + a `requestedLevelApplied` boolean — instead of a bare `"Scalability set"`, so a clean
success cannot mean "a group was silently shadowed." Do **not** switch to issuing `scalability
<N>` / writing `sg.*` at console priority: the handler already uses the canonical programmatic
engine API, and forcing a higher priority would change engine-standard scalability semantics.
Do **not** "correct" `E-baseline-no-sg-setall` — its `sg.*` claim is correct.

## History
- `#1-initial-repro` `OPEN` reporter — `performance.set_scalability {level:N}` returns `{"message":"Scalability set"}` for every level but never updates the `sg.*` group CVars and leaves the dependent `r.*` CVars stuck. Replay-confirmed live: a directly-set `sg.ShadowQuality=2` survived `set_scalability {level:3}` unchanged (group never went to 3); `sg.TextureQuality` stayed `0`; `r.ViewDistanceScale` sat at `0.4` across `set_scalability {0}/{2}/{3}` (Epic never restored 1.0); and a raw `sg.ViewDistanceQuality 3` console command DID update both `sg.ViewDistanceQuality`→3 and `r.ViewDistanceScale`→1, proving the engine path works and `system.console.search` reads `sg.*` correctly — so the RPC's programmatic path (`Scalability::SetFromSingleQualityLevel`→`SetQualityLevels`, `PerformanceHandler.cpp` ~L160) is the culprit. Surfaced by a low-spec playtest-budget task that requested Medium(1) and could not confirm it via the `sg.*` readback (read back `0`/Low). Silent false-success on a normal path; directly contradicts `E-baseline-no-sg-setall`'s claim that this RPC drives `sg.*`.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded High/bug → Low/ergonomic. Verified against UE 5.7 `Engine/Private/Scalability.cpp` that `SetQualityLevels` DOES write the `sg.*` group CVars (`SetQualityLevelCVar(CVarViewDistanceQuality, ...)` etc. → `CVar->Set(DesiredValue, ECVF_SetByScalability)`), so the original "never writes sg.*" root cause and the "contradicts E-baseline" claim are both false — `CVarViewDistanceQuality` IS `sg.ViewDistanceQuality`. The original repro pinned `sg.*` via console (`ECVF_SetByConsole`, highest priority) FIRST, which correctly shadowed the later lowest-priority (`ECVF_SetByScalability`) scalability write; that is expected engine priority behavior, and the `r.*`-stuck symptom is the same shadowing, not a broken RPC. The genuine residual is ergonomic: the handler returned a bare `"Scalability set"` with no read-back, so a caller could not detect when a higher-priority pin meant the requested level did not land. Fix: `Source/PinWright/Private/Handlers/Debug/PerformanceHandler.cpp` `performance.set_scalability` now reads the 11 `sg.*` level-group CVars back after `SetQualityLevels` (effective `GetInt()` values) and returns `requestedLevel` + `appliedGroups` (per-group `{cvar,value}`) + `requestedLevelApplied`; updated the registered summary to document the `sg.*` write and its lowest-priority semantics. Regression test `PinWright.performance.set_scalability.ReadsBackAppliedGroups` in `Source/PinWright/Private/Tests/EditorOps/TestDebugHandlers.cpp` snapshots/restores live scalability state and asserts the success carries `requestedLevel`, an `appliedGroups` array naming `sg.ShadowQuality`, and the `requestedLevelApplied` flag — it fails if reverted to the bare message. Did NOT change `E-baseline-no-sg-setall` (its `sg.*` claim is correct) and did NOT switch the handler to console-priority writes.
