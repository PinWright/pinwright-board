---
id: H-audit-filer-no-session-limit-retry
title: "workflow-log-audit's File stage — the single agent holding ALL persistence (tickets, report, manifest watermark) — is one plain agent() call; a session-limit death loses the entire audit's output"
status: OPEN
severity: Medium
workflow: both
category: resume-restart
knob: middleware
encounters: 1
lastSeen: 2026-07-13T10:56:07Z
filedBy: supervisor@fuzz2
editTargets:
  - .polyskill/skills/workflow-log-audit/workflow-log-audit.workflow.js
---

# The audit's File stage has no session-limit resilience despite being the only persistence point

Scout/Sweep/Synthesize produce only in-journal intermediate results; every durable output —
H- ticket creation/dedup, the narrative report, the manifest watermark advance — happens in the
single File agent at the end. That agent is invoked as one plain `agent()` call with no retry.
A session-limit outage (which kills every spawn instantly, possibly for hours) at that moment
discards the entire audit's work product: `filed: 0`, no report, no watermark — while the
600k+-token Sweep/Synthesize spend is already sunk. The mcp-fix-workflow solved this exact
shape with `agentOrRetryForever` for its board-only abandon step (workflow.js comment: a
session-limit outage burned 3 bounded tries in 3 minutes; only an unbounded retry crosses the
reset window). The manual mitigation — Workflow resume with `resumeFromRunId`, which replays
the cached stages and re-runs only the filer — works but requires a live supervisor to notice
and act; the hourly-tick invocation path has no such operator.

## Evidence

Run `wf_241f430c-bff` (fuzz2, 2026-07-13, the file-to-board smoke test over `wf_510c6c2c-eb5`):
5/5 readers ok, 20 raw findings, synth complete — then `failures[]: "[file-to-board] failed:
You've reached your Fable 5 limit."` → returned `{filed: 0, tickets: []}`; manifest still
legacy, report unwritten. A manual `resumeFromRunId` relaunch re-ran only the filer and filed
all 18 tickets (112k tokens vs the original 1.26M).

## Proposed edit

In `.polyskill/skills/workflow-log-audit/workflow-log-audit.workflow.js`, wrap the File-stage
`agent()` call in an unbounded retry loop mirroring mcp-fix-workflow's `agentOrRetryForever`
(re-run on a null result, `log()` each attempt). The filing prompt is already idempotent by
construction — per-finding dedup-before-create means a partial prior attempt's tickets are
re-matched as `dedup`, not duplicated — so retry is safe. Optionally do the same for Scout
(cheap, also idempotent); Sweep/Synthesize can stay bounded since resume covers them and their
loss is recoverable from cache.

## History
- `#1-filed-from-smoke-test` `OPEN` supervisor — Filed after the first file-to-board smoke run lost all 18 tickets to a session-limit death in the File stage and needed a manual resumeFromRunId to recover. The unattended hourly-tick path would have silently produced a no-op audit with the full reader spend wasted.
