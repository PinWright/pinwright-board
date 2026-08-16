---
id: B-job-progress-interval-default-60s
title: "ProgressEventMinIntervalMs defaulted to 60000, so FJobRegistry::RecordProgress accepted at most one progress event per minute per ticket — a polling agent could not tell a slow job from a wedged editor, which is the one question job tickets exist to answer"
status: IN-REVIEW
severity: Medium
category: bug
tags: [jobs, progress, polling, job-status, settings, default-value, observability]
---

# One progress event per minute is indistinguishable from a wedged editor

`ProgressEventMinIntervalMs` defaulted to **60000**. `FJobRegistry::RecordProgress` therefore
accepted at most one event per minute per ticket, so a five-minute job left five progress lines in
`jobs.jsonl`, and an agent polling `system.job_status` could not tell a slow job from a wedged
editor at that resolution.

## The throttle drops the event entirely

`State/JobRegistry.cpp:88-105`:

```cpp
        if (!T || T->Status != TEXT("running")) return false;

        const double Now = FPlatformTime::Seconds();
        if (!bBypassRateLimit && ProgressMinIntervalMs > 0 &&
            (Now - T->LastProgressSeconds) * 1000.0 < ProgressMinIntervalMs)
        {
            return false;
        }
        T->LastProgressSeconds = Now;
        FJobProgressEvent Ev{ FDateTime::UtcNow(), Message, Payload };
        T->Progress.Add(MoveTemp(Ev));
```

A throttled call returns **before** `T->Progress.Add(...)`, so the event never enters the ticket's
progress array at all. That array is exactly what the polling verb serves —
`Handlers/System/JobControlHandler.cpp:10-27` (`system.job_status` -> `FJobRegistry::ToJson(T)`) ->
`JobRegistry.cpp:261-284` (`Out->SetArrayField(TEXT("progress"), ProgressJson)`). The same array is
additionally ring-trimmed at `JobRegistry.cpp:99-104` (`constexpr int32 MaxProgressEvents = 50;`).

Wiring: `State/PluginState.cpp:79-82` passes `S->ProgressEventMinIntervalMs` into the registry
constructor (`JobRegistry.cpp:32-34`, member at `JobRegistry.h:126`).

## Why this is deliberately its own ticket

It shares a **symptom** with `B-mcp-progress-token-never-echoed` (progress not arriving) and nothing
else. This one degrades the **polling** path, which has no MCP involvement at all, and it would have
gone on doing so after the wire-format fix landed. Filing them together is how a two-defect bug gets
recorded as closed with one of them still live.

Streamed progress was never throttled — `JobRegistry.h:90` documents `bBypassRateLimit` as skipping
`ProgressMinIntervalMs` for streamed jobs — so the two defects also had disjoint blast radii.

## Fix (shipped `dbaa5704`)

`PinWrightSettings.cpp:53-59`, now `1000`:

```cpp
    // One minute was the old default, chosen when this throttle only bounded jobs.jsonl
    // growth. It is the wrong number for a progress feature: a job polled through
    // system.job_status recorded a single event per minute, which reads the same as a
    // wedged editor. 1s bounds the log at 60 lines/minute per job while keeping a poll
    // informative. Streamed progress bypasses this entirely (RecordProgress's
    // bBypassRateLimit) - a client watching live wants every event.
    ProgressEventMinIntervalMs = 1000;
```

Declaration at `Public/PinWrightSettings.h:154-158` (`ClampMin 0`, `ClampMax 60000`). **No `Config/`
override exists** in either the host project or the plugin, so the C++ default is the effective
value and changing it changed behaviour everywhere.

One deliberate non-bypass: the Python-script progress sink respects the throttle rather than
bypassing it (`State/ActiveProgressSink.cpp:64-77`), because the obvious way to write a caller's
loop is to report every iteration — and 1 s is the right resolution for something a human or an
agent is watching.

## Verification

`Tests/Transport/TestJobStreamBridge.cpp:147,157-173` (throttled -> observer fires once) and
`:190-206` (bypass -> both events) pin the behaviour on both sides of the flag; settings read-back
at `Tests/State/TestActiveProgressSink.cpp:123`.

Docs: `rpc-design.md:203` — *"Check the throttle when you add progress. `ProgressEventMinIntervalMs`
defaulted to 60000 — one event per minute — which would have swallowed any new per-item reporting
and produced a bug report against the wrong layer."* Wire doc at `wiki-src/system.md:49`.

## Related

- `B-mcp-progress-token-never-echoed` — same symptom, unrelated cause, separate path.
- `F-jobs-settings` — where this setting was introduced.

## History
- `#1-one-event-per-minute` `OPEN` reporter — `ProgressEventMinIntervalMs` defaulted to **60000**, so `FJobRegistry::RecordProgress` accepted at most one progress event per minute per ticket: a five-minute job left five lines in `jobs.jsonl`, and an agent polling `system.job_status` could not tell a slow job from a wedged editor at that resolution — the single question a job ticket exists to answer. The throttle discards rather than defers: `JobRegistry.cpp:88-95` returns false BEFORE `T->Progress.Add(...)` (`:96-98`), so the event never enters the ticket's progress array, and that array is what `system.job_status` serves (`JobControlHandler.cpp:10-27` -> `JobRegistry.cpp:261-284`) and is additionally ring-trimmed to 50 (`:99-104`). Effective everywhere: no `Config/` override exists in the host project or the plugin, so the C++ default was the live value. Deliberately filed apart from `B-mcp-progress-token-never-echoed` despite the shared symptom — this degrades the POLLING path, which has no MCP involvement at all, and would have gone on doing so after the wire-format fix landed; streamed progress bypasses the throttle entirely (`JobRegistry.h:90`), so the two defects have disjoint blast radii as well as unrelated causes.
- `#2-fix` `IN-REVIEW` developer — Fixed in `dbaa5704` (pushed to `origin/master`), which touches only `PinWrightSettings.cpp` and `wiki-src/system.md`. The default is now **1000**, with the reasoning recorded in place at `PinWrightSettings.cpp:53-59`: one minute was chosen when this throttle only bounded `jobs.jsonl` growth, and is the wrong number for a progress feature; 1 s bounds the log at 60 lines/minute per job while keeping a poll informative. The Python-script progress sink deliberately does NOT bypass the throttle (`ActiveProgressSink.cpp:64-77`), unlike the plugin's own streamed jobs, because the obvious way to write a caller's loop is to report every iteration — so the naive loop is correct by construction rather than by instruction. Behaviour pinned on both sides of the flag by `Tests/Transport/TestJobStreamBridge.cpp:147,157-173` (throttled -> one observer call) and `:190-206` (bypass -> both), with the settings read-back at `Tests/State/TestActiveProgressSink.cpp:123`. Transferable note recorded at `rpc-design.md:203`: check the throttle when you add progress, because at 60000 any new per-item reporting would have been swallowed and produced a bug report against the wrong layer.
