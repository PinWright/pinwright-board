---
id: E-performance-run-benchmark-async-poll-undocumented
title: "performance wiki overlay never documents the run_benchmark async ticket → system.job_status poll pattern"
status: OPEN
severity: Low
category: ergonomic
tags: [performance, benchmark, docs, async, jobs, job_status, discoverability]
encounters: 2
lastSeen: 2026-07-11T13:17:47+03:00
---

# performance wiki overlay never documents the run_benchmark async ticket → system.job_status poll pattern

`performance.run_benchmark` is an async-job handler: it `Ctx.StartJob()`s and
returns a ticket **synchronously**, then completes via a one-shot timer at the
configured duration (see the now-DONE `B-performance-run-benchmark-no-completion-signal`).
The real "the benchmark actually ran for its N seconds and recorded load"
disposition is only knowable by separately polling `system.job_status` with that
ticket until it reaches a terminal `completed`. But the performance namespace wiki
overlay that should teach a caller this two-call contract —
`docs/wiki-src/performance.md` — is a 3-line namespace intro and says **nothing**
about `run_benchmark` being async, the ticket it returns, or the
`system.job_status` follow-up (`grep -niE "async|ticket|poll|job_status|run_benchmark"
docs/wiki-src/performance.md` = no matches).

The consequence is a discoverability gap: a caller who needs the benchmark's load
window to actually have elapsed before the next step (e.g. stopping a stat-file
capture so the `.uestats` records the load) has to **self-discover** that
`run_benchmark` returns a ticket and that `system.job_status` is how you wait for
it. If they instead treat the synchronous response as "done" and proceed
immediately, they stop the capture before the 5s of load is recorded — a silent
correctness trap, not just an ergonomic one, because the captured artifact would be
empty/idle.

This is the direct performance-namespace analog of the already-filed
`E-pipeline-run-ubt-async-poll-undocumented` (same friction, pipeline overlay /
`run_ubt` handler): a different overlay page (`performance.md` vs `pipeline.md`) and
a different async handler (`run_benchmark` vs `run_ubt`), so it is a distinct docs
edit, but the same root pattern — an async ticket→`system.job_status` contract that
the namespace overlay never surfaces. It is also distinct from the DONE bug
`B-performance-run-benchmark-no-completion-signal` (which *added* the completion
signal): the signal now exists, but nothing tells a caller it exists or how to
consume it. The discoverability gap survives that fix because the async response
shape is the permanent design.

**What it should do:** The `docs/wiki-src/performance.md` overlay should document,
for `performance.run_benchmark`, that the call is fire-and-forget: it returns a
ticket synchronously (NOT benchmark results), and callers who need the load window
to have elapsed (or who want the result payload) must poll
`system.job_status {ticket_id}` until a terminal `status`. Cross-link the
`system.job_status` doc and the `F-long-running-tickets` job model so the two-call
pattern is discoverable from the performance page a profiling-pass author starts on.
Mirror the wording already proposed in `E-pipeline-run-ubt-async-poll-undocumented`.

**Workaround:** After `performance.run_benchmark`, poll
`system.job_status {ticket_id}` (from the kickoff response) until the status is
terminal before doing anything that depends on the load having run (e.g.
`performance.stop_profiling`); treat the synchronous ticket as "accepted", never as
"the benchmark finished".

## Process friction this caused (this task)

A profiling-pass task (seed `performance.start_profiling`) ran the full sequence
`show_fps` → `show_stats unit` → `show_stats gpu` → `start_profiling` →
`run_benchmark {duration:5, type:all}` → `system.job_status` (poll to completed) →
`stop_profiling` → `generate_memory_report` → `show_fps off` — 10 calls, all
`ok=true`, no retries/crashes (a clean "ergo" outcome). The benchmark step's
async-ness was pure process friction: quoting the friction note, *"run_benchmark is
async (returns a ticket), so I had to discover/poll system.job_status to ensure the
5s of load was actually recorded before stopping the capture."* The agent had to
self-discover the poll contract — nothing in the performance overlay pointed there —
specifically because stopping the stat-file capture before the load elapsed would
have produced an idle `.uestats`. (The same task's *path-omission* friction is
already tracked under `E-stop-profiling-no-uestats-path` and `E-memory-report-no-path`;
this ticket is the orthogonal *async-discoverability* angle.)

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean profiling-pass task (seed `performance.start_profiling`, 10 calls, all `ok=true`, no retries/crashes, outcome "ergo"). Friction note: "run_benchmark is async (returns a ticket), so I had to discover/poll system.job_status to ensure the 5s of load was actually recorded before stopping the capture." Verified `docs/wiki-src/performance.md` is a 3-line namespace intro (`grep -niE "async|ticket|poll|job_status|run_benchmark"` = no matches); the async ticket→`system.job_status` poll contract is undocumented in the performance overlay, so a profiling author must self-discover the poll — a silent correctness trap, since stopping the stat capture before the load window elapses records an idle `.uestats`. Direct performance-namespace analog of `E-pipeline-run-ubt-async-poll-undocumented` (distinct overlay page + handler); distinct from the DONE bug `B-performance-run-benchmark-no-completion-signal` (signal now exists but is undocumented). Ergonomic/docs, not an outcome bug — every call succeeded. Fix: document the async/ticket/poll contract on `docs/wiki-src/performance.md`, cross-link `system.job_status` and `F-long-running-tickets`.
- `#2-liveness` `OPEN` reporter — Still observed on a clean level-health baseline task (seed `system.inspect.get_performance_stats`, outcome "clean"). CallAnalyzer + call-log ground truth: agent read the per-method wiki page `performance.run_benchmark.md` (body only "Start a performance benchmark" + duration/type params, no async hint), called `run_benchmark {duration:3}` expecting perf figures, got a job ticket instead, and had to make 3 unplanned follow-up ops (Glob `system.job_status.md`, Read it, then poll `system.job_status {ticket_id}` → completed, `result.statFilePath`=.uestats). Confirms the async/ticket→poll contract is undocumented not only on the namespace overlay `performance.md` but on the per-method `performance.run_benchmark.md` page too. Same root cause, no new fix — the overlay/per-method doc edit already proposed in #1 covers it.
