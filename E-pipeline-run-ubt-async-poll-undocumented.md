---
id: E-pipeline-run-ubt-async-poll-undocumented
title: "pipeline wiki overlay never documents the run_ubt async ticket → system.job_status poll pattern"
status: OPEN
severity: Low
category: ergonomic
tags: [pipeline, ubt, docs, async, jobs, job_status, discoverability]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# pipeline wiki overlay never documents the run_ubt async ticket → system.job_status poll pattern

`pipeline.run_ubt` is an async-job handler: it returns `status=running` with a
`ticket_id` **synchronously** for every accepted input, and the actual
disposition (succeeded / failed / error) is only knowable by separately polling
`system.job_status` with that `ticket_id`. The pipeline namespace wiki overlay
that should teach a caller this two-call contract —
`docs/wiki-src/pipeline.md` — is a single descriptive sentence about the
namespace scope and says **nothing** about the async/ticket/poll pattern, the
`system.job_status` follow-up, or the fact that the synchronous response is not
a build result.

The consequence is a discoverability trap: the synchronous `status=running` is a
**false-success signal**. A caller (here, someone wiring `run_ubt` into nightly
CI) who reads only the synchronous response — or only the pipeline overlay —
will conclude the build launched and is progressing, when the only authoritative
outcome lives behind a second `system.job_status` call they were never told to
make. In this task the agent had to discover the poll pattern on its own: it
called `system.job_status` (initially with no args, getting the doc page per the
omit-args=docs convention), read the doc, then re-called with `ticket_id` to find
the jobs had actually terminated. Nothing in the pipeline overlay pointed there.

This is distinct from the already-filed code bug
`B-pipeline-run-ubt-bad-exe-path` (the hardcoded `RunUBT.bat` dead path that
makes every Windows build fail `CREATEPROC_FAILED`). That bug is about builds not
running; **this** ticket is about discoverability of the async contract, which
survives the bug fix: even once UBT launches correctly, `run_ubt` will still
return `status=running` synchronously and still require a `system.job_status`
poll to learn the result, and the overlay still won't say so.

**What it should do:** The `docs/wiki-src/pipeline.md` overlay should document,
for `pipeline.run_ubt` (and any other async pipeline handlers), that the call is
fire-and-forget: it returns `{status:"running", ticket_id, monitor_path, args}`
synchronously, that this is NOT a build outcome, and that callers must poll
`system.job_status {ticket_id}` until a terminal `status` (`completed`/`failed`)
to read the real disposition. Cross-link to the `system.job_status` doc and the
`F-long-running-tickets` ticket/JSONL job model so the two-call pattern is
discoverable from the pipeline page a CI author starts on.

**Workaround:** After any `pipeline.run_ubt` call, poll
`system.job_status {ticket_id}` (from the kickoff response) until the status is
terminal; treat the synchronous `status=running` as "accepted", never as
"succeeded".

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean pipeline CI-readiness task (17 calls, all ok=true, no retries/crashes). Friction note: "run_ubt returns status=running synchronously for EVERY input ... so the real disposition is only visible by separately polling system.job_status — the synchronous response gives a false-success signal" and "my first system.job_status call omitted args and returned the wiki page ... then I read the doc and called it correctly with ticket_id." Verified `docs/wiki-src/pipeline.md` is a one-line stub (grep for job_status|ticket_id|poll|async = no matches); the async two-call contract is undocumented in the pipeline overlay, so a CI author must self-discover the poll. Distinct PROCESS/docs angle from the code bug `B-pipeline-run-ubt-bad-exe-path` (dead exe path): the discoverability gap persists after that bug is fixed because the async response shape is the permanent design. Ergonomic/docs, not an outcome bug — every call in the task succeeded.
