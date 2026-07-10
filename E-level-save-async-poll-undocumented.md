---
id: E-level-save-async-poll-undocumented
title: "level.save wiki (per-method page + level overlay) never documents its async ticket → system.job_status poll contract"
status: OPEN
severity: Medium
category: ergonomic
tags: [async-poll-undocumented, level, save, docs, async, jobs, job_status, discoverability]
encounters: 1
lastSeen: 2026-07-10T21:44:13+03:00
---

# level.save wiki (per-method page + level overlay) never documents its async ticket → system.job_status poll contract

`level.save` is an async-job handler: per the now-DONE bug
`B-level-save-no-completion-signal` its handler was migrated to `Ctx.StartJob()`
+ an `AsyncTask`-wrapped `FEditorFileUtils::SaveLevel`, so the call **returns a
job ticket synchronously** (`{status:"running", ticket_id, method:"level.save",
docs:"call(\"system.job_status\") for guidance"}`), NOT a synchronous
`{saved:true}`. The real disposition (`{saved:true}`) is only knowable by
separately polling `system.job_status {ticket_id}` until a terminal `completed`.
But the wiki a caller reads beforehand never surfaces this two-call contract:

1. **Per-method page omits it.** The generated `level.save` wiki page
   (`Saved/PinWright/wiki/level.save.md`) documents only: *"Save the active editor
   world's persistent level package. To save with a new name, use level.save_as.
   Parameters: none."* It makes no mention of asynchronous execution, a ticket, or
   `system.job_status`. An agent/user reading it expects a simple synchronous
   `{saved:true}`.
2. **Level overlay implies level.save is the SYNCHRONOUS endpoint.** The
   `docs/wiki-src/level.md` overlay (line 11, added by the sibling
   `E-level-build-lighting-async-poll-undocumented` fix) documents the *lighting-build*
   verbs as async and tells callers to poll `system.job_status` *"before you
   `level.save` or read lighting state"* — treating `level.save` itself as the
   plain synchronous "done" step you reach after the poll. It never says
   `level.save` is itself an async ticket-returning job. So the one overlay mention
   of `level.save` actively frames it as synchronous, the opposite of its real
   response shape.

The canonical async-poll contract IS documented globally — `docs/wiki-src/system.md`
has the `## Long-running jobs` section + `### system.job_status`, and the kickoff
response itself carries the inline `docs: call("system.job_status") for guidance`
hint — but neither the `level.save` per-method page nor the `level` overlay a save
author actually lands on surfaces or cross-links it.

This is the direct **level.save-verb** analog of the already-filed
`E-navigation-rebuild-async-poll-undocumented` (navigation.md / rebuild_navigation),
`E-performance-run-benchmark-async-poll-undocumented`,
`E-pipeline-run-ubt-async-poll-undocumented`,
`E-render-nanite-rebuild-async-poll-undocumented`, and
`E-level-build-lighting-async-poll-undocumented`: same friction, same root pattern —
an async ticket→`system.job_status` contract the namespace wiki never surfaces — but
a different verb (`level.save`), so it is a distinct docs edit. It is distinct from:

- `B-level-save-no-completion-signal` (DONE) — that bug *added* the completion
  signal (the job + `CompleteJob`); the signal now exists, but nothing in the
  `level.save` wiki tells a caller it exists or how to consume it. The
  discoverability gap survives that fix because the synchronous-ticket response
  shape is the permanent design.
- `E-level-build-lighting-async-poll-undocumented` (IN-REVIEW) — same `level.md`
  page but a different verb pair (`build_lighting`/`build_level_lighting`); that
  ticket's fix even *introduced* the line that mis-frames `level.save` as
  synchronous, so this is a distinct verb + distinct overlay edit.

Higher reach than its Low siblings: unlike the rare rebuild/benchmark/UBT/nanite
verbs, `level.save` runs in almost every editing session (nearly every mutate task
ends with a save), so the reach modifier bumps it one level → **Medium**.

**What it should do:** The `level.save` per-method docs (and the `docs/wiki-src/level.md`
overlay) should state that `level.save` is a fire-and-forget async job: it returns a
ticket synchronously (`{status:"running", ticket_id}`, NOT `{saved:true}`), and
callers who need the save to have persisted (e.g. before a source-control submit or a
disk-read verify) must poll `system.job_status {ticket_id}` until a terminal status,
mirroring the response's inline `docs: call("system.job_status")` hint. Fix the
`level.md` line-11 phrasing so `level.save` is not implied to be the synchronous
endpoint. Cross-link `system.md#long-running-jobs` / `system.md#systemjob_status`
(where `level.save` should also appear in the ticket-pattern table) and the
`F-long-running-tickets` job model. Mirror the wording already used in the sibling
async-poll tickets.

**Workaround:** After `level.save`, poll `system.job_status {ticket_id}` (from the
kickoff response) until the status is terminal (`completed`, `result.saved:true`)
before doing anything that depends on the save having persisted; treat the
synchronous ticket as "accepted", never as "the save finished".

## Process friction this caused (this task)

A clean, fully-successful dynamic-relight task (seed `lighting.configure_shadows`,
17 pinwright RPCs, zero retries, zero failed calls — every call landed first-try
with a concrete readback: KeyLight `existsAfter:true`, `ensure_single_sky_light`
`removed:1`, VSM `virtualShadowMaps:true`, `LumenGI`, fog `enabled:true`, AO +
exposure on `PostProcessVolume_0`). The only process divergence was the final
`level.save`: the agent's plan assumed a single synchronous save, but the call
returned an async job ticket (`{status:"running",
ticket_id:"j_20260710T183151_970c3a3f", method:"level.save",
docs:"call(\"system.job_status\") for guidance"}`). Quoting the agent's narration:
*"The save is an async job. Let me learn how to poll job status from the wiki."* It
then had to add two discovery steps (glob + read of `system.job_status.md`) plus one
extra `system.job_status` RPC poll (→ `completed`, `result.saved:true`) that a
documented signature would have pre-empted. The runtime response's inline `docs`
hint was the ONLY thing that surfaced the async/ticket contract — the wiki page a
real user would read beforehand omitted it entirely. Self-correcting round-trip, not
a blocked task, but repeated on a method almost every save-ending task hits.

The per-finding judge filed a separate Critical crash ticket
(`B-open-asset-world-map-load-crash`, culprit `editor.open_asset`) that is unrelated
to this docs gap; this PROCESS finding is the CallAnalyzer's sole trace inefficiency.

severity rationale: impact=Low (pure docs/discoverability; the save succeeded and
the runtime response self-documented the next step) × reach=every-session
(`level.save` ends nearly every mutate task) -> Medium.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean dynamic-relight task (seed `lighting.configure_shadows`, 17 RPCs, outcome tool_bug on an unrelated `editor.open_asset` crash filed as `B-open-asset-world-map-load-crash`). CallAnalyzer's sole trace inefficiency: `level.save` returned an async ticket (`{status:"running", ticket_id:"j_20260710T183151_970c3a3f", docs:"call(\"system.job_status\")"}`), but the per-method wiki page `Saved/PinWright/wiki/level.save.md` documents only "Save the active editor world's persistent level package. Parameters: none." with no async/ticket/poll note, and the `level.md` overlay line 11 mis-frames `level.save` as the synchronous endpoint you reach *after* polling the lighting-build job. Cost the attempt two discovery reads (glob + `system.job_status.md`) + one extra poll to confirm `saved:true`. Direct level.save-verb analog of `E-navigation-rebuild-async-poll-undocumented`, `E-performance-run-benchmark-async-poll-undocumented`, `E-pipeline-run-ubt-async-poll-undocumented`, `E-render-nanite-rebuild-async-poll-undocumented`, `E-level-build-lighting-async-poll-undocumented`; distinct from the DONE bug `B-level-save-no-completion-signal` (signal now exists but is undocumented in the save wiki) and from the sibling lighting-build ticket (different verb on the same page). Ergonomic/docs. Fix: document the async ticket→`system.job_status` poll contract on the `level.save` docs, fix the level.md line-11 synchronous mis-framing, and cross-link `system.job_status` + `F-long-running-tickets`.
