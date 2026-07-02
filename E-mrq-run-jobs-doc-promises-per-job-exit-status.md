---
id: E-mrq-run-jobs-doc-promises-per-job-exit-status
title: "`mrq` wiki claims 'the ticket result includes per-job exit status' but the mrq.run_jobs ticket result is only aggregate `{\"success\":true}` — no per-job array"
status: OPEN
severity: Low
category: ergonomic
tags: [mrq, movie-pipeline, run_jobs, job_status, docs, misleading-doc]
encounters: 1
lastSeen: 2026-07-02T11:39:42.5870560+03:00
---

# `mrq` wiki promises per-job exit status the run_jobs ticket never delivers

The `mrq` namespace wiki page states, in its "Long-running pattern" section:

> The ticket result includes per-job exit status.

This is false. The `mrq.run_jobs` completion handler builds the ticket result
as a single aggregate boolean and nothing else. A caller who batches several
jobs (the natural use of a *queue*) and follows the doc expecting to read a
per-job exit-status array from the resolved `system.job_status` result finds
only `{"success":true}` — one boolean for the whole queue, no per-job breakdown,
empty `progress[]`. If job 1 rendered but job 2 failed, the caller cannot tell
which job failed from the ticket payload; they get a single rolled-up flag.

## What's wrong
The wiki overlay makes a concrete promise about the shape of the result payload
that the handler does not honor. The claim reads as if the ticket resolves to a
list of per-job outcomes; the actual contract is one aggregate `success` bool.

## What it should do
Either (a) enrich the `mrq.run_jobs` completion result to actually carry a
per-job array (e.g. `jobs: [{jobName, success, outputDir?}, ...]` built from the
queue's `UMoviePipelineExecutorJob` entries) so the doc's promise holds and a
batch caller can attribute a failure to a specific job; or (b) correct the wiki
prose to state the truth — the ticket result is an aggregate `{"success":bool}`
for the whole queue, with no per-job breakdown — so callers don't hunt for a
per-job field that isn't emitted.

## Verbatim repro
- Wiki claim — `Plugins/PinWright/docs/wiki-src/mrq.md:22` (served at
  `wiki/mrq.md:24`): `The ticket result includes per-job exit status.`
- Actual result — `mrq.create_job` (x2) → `mrq.run_jobs {}` → poll
  `system.job_status {ticket_id}` until terminal. Terminal result observed
  verbatim (also recorded in `Saved/PinWright/jobs.jsonl`):
  `{"ts":"...","ticket_id":"j_20260702T083222_82b06d5b","method":"mrq.run_jobs","event":"completed","result":{"success":true}}`
  — i.e. `result` is `{"success":true}`, no `jobs[]`, no per-job exit status.
- Guilty source — `Plugins/PinWright/Source/PinWright/Private/Handlers/MRQ/MRQHandler.cpp:180-183`,
  the `OnExecutorFinished` lambda:
  ```cpp
  TSharedPtr<FJsonObject> Result = MakeShared<FJsonObject>();
  Result->SetBoolField(TEXT("success"), bSuccess);
  OnComplete(bSuccess, Result, bSuccess ? FString() : TEXT("MRQ executor reported failure"));
  ```
  The result object only ever receives a single `success` bool; nothing iterates
  `Queue->GetJobs()` to emit per-job status. `UMoviePipelineExecutorBase::OnExecutorFinished`
  also delivers only an aggregate `bool bSuccess`, so the per-job data the doc
  advertises isn't even plumbed to this callback.

## Workaround
Treat the ticket `result.success` as a whole-queue rollup. To attribute a failure
to a specific job, run one job per queue (single `mrq.create_job` before each
`mrq.run_jobs`) so the aggregate maps 1:1 to that job, or inspect the MRQ output
directory / editor log out-of-band.

severity rationale: impact=docs-overclaim (caller trusts a doc promise about the
result shape that isn't emitted; the render itself succeeds and the aggregate flag
is honest) × reach=rare (offline MRQ batch render) -> Low.

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode task: batch-render two Sequencer demo cinematics offline through MRQ and watch to completion. Full round-trip succeeded (list_presets count 0; two create_job; run_jobs → running ticket under MoviePipelinePIEExecutor; system.job_status resolved to `completed` `{success:true}`). Friction surfaced by the attempt: the `mrq` wiki claims "the ticket result includes per-job exit status," but the resolved result is just aggregate `{"success":true}` with empty `progress[]` and no per-job array. Replay-confirmed against source: `MRQHandler.cpp:180-183` sets only `success` on the result object (engine `OnExecutorFinished` hands the lambda only an aggregate `bool bSuccess`), and the recorded terminal ledger `Saved/PinWright/jobs.jsonl` shows `result:{"success":true}`. Doc concretely overclaims the result payload; the call itself is correct. Same doc-promises-absent-result-field family as `E-asset-get-doc-promises-tags` / `E-blueprint-get-omits-components-readback-guidance` (both distinct methods, filed separately). Not a gap (F-mrq-render-queue is DONE and the batch render works); not a bug (aggregate success is truthful) — ergonomic doc overclaim.
