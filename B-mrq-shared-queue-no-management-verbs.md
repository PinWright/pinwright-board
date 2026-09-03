---
id: B-mrq-shared-queue-no-management-verbs
title: "mrq.create_job appends to the editor-global queue and mrq.run_jobs renders ALL of it — no verb lists, removes or clears a job, so a stale entry from earlier in the session silently re-renders and overwrites its outputs"
status: IN-REVIEW
severity: High
category: bug
tags: [mrq, run_jobs, create_job, queue, shared-state, movie-render-queue, destructive, missing-verb, no-workaround]
encounters: 1
lastSeen: 2026-09-02T20:15:00+03:00
---
# `mrq.create_job` appends to the editor-global queue and `mrq.run_jobs` renders ALL of it — no verb lists, removes or clears a job

`mrq.create_job` allocates into the **editor-global, session-persistent** MRQ queue:
`GEditor->GetEditorSubsystem<UMoviePipelineQueueSubsystem>()` (`MRQHandler.cpp:238`) →
`QSS->GetQueue()` (`:244`) → `Queue->AllocateNewJob(...)` (`:289`). Nothing ever removes
an entry — a plugin-wide grep for `DeleteJob`, `DeleteAllJobs`, `SetJobEnabled`,
`Empty`, `Reset` on the queue returns **zero hits in handler code** (three hits total,
all test-fixture cleanup: `Tests/Media/TestMRQHandlers.cpp:45`, `:394`).

`mrq.run_jobs` then renders **the whole queue**. Its only parameter is `executorClass`
(`MRQHandler.cpp:357-359`); the body checks `Queue->GetJobs().Num() == 0`
(`:374`) and calls `LiveQSS->RenderQueueWithExecutor(ExecutorClassCopy)` (`:410`).
There is no job id, no job list, no `RenderJob`, and no filtering pass. Whatever any
earlier call — or the editor UI, or another agent sharing this editor — left in the
queue is rendered, into the output directory that job carries, **overwriting whatever
is already there**.

The namespace is exactly three verbs and none of them manages the queue:
`mrq.create_job` (`MRQHandler.cpp:205`), `mrq.run_jobs` (`:344`), `mrq.list_presets`
(`:466`). A tree-wide grep for `REGISTER_RPC_HANDLER("mrq` returns only these three, so
the missing capability is not hidden under another name.

**The response publishes the count and withholds everything needed to act on it.**
`create_job` returns `queueSize` = `Queue->GetJobs().Num()` (`:307-309`, emitted `:317`)
— the length of the *whole shared queue*, foreign jobs included — and `jobIndex` =
`Jobs.Num() - 1` (`:308`), a position rather than an identity. `run_jobs` repeats
`queueSize` in its started payload (`:396`). Neither publishes the **names, sequences or
output directories** of the other entries, and neither warns. So `queueSize: 3` on a call
that queued one job is the *only* signal that two unknown renders are about to run, and
it is a bare integer with no documented meaning: `queueSize` and `jobIndex` appear
**nowhere** in `Docs/` or in the generated wiki (`Saved/PinWright/wiki/mrq.create_job.md`,
`mrq.run_jobs.md` — zero occurrences of either name).

**The hazard is already named in the source, as a code comment nobody on the wire can
read.** `MRQHandler.cpp:266-267`, justifying the `MRQ_PRESET_NOT_LOADABLE` refusal:

    // and the misconfigured job would sit in the
    // shared queue as a trap for anyone's later mrq.run_jobs.

That reasoning was accepted for the *preset* defect and never applied to the queue
itself. `B-mrq-render-result-omits-bitrate-and-size` `#3` makes the same argument in
prose ("the misconfigured job would have sat in the editor-global queue as a trap for
anyone's later `run_jobs`") and likewise files nothing about it.

## Field evidence, 2026-09-02

**Relayed** from the rendering agent (not reproduced here — read-only probes only, no
MRQ runs): `mrq.create_job` returned `queueSize: 3` carrying two stale jobs from earlier
the same day, which `mrq.run_jobs` would have re-rendered over their existing outputs.

**Corroborated first-hand from this checkout**, which makes the relayed claim checkable:

- `Saved/PinWright/jobs.jsonl` L67, `2026-09-02T17:32:28.853Z`, method `python.execute`
  — the queue enumerated as `0 vid_columns_v3 | /Game/Atlantis/Cine/Video/LS_Vid_Columns`,
  `1 vid_statues_v2 | /Game/Atlantis/Cine/Video/LS_Vid_Statues`,
  `2 TT_Column_test12 | /Game/Atlantis/Cine/Video/Scratch/LS_TT_Column`. Three jobs, two
  of them stale.
- `jobs.jsonl` L69, `17:32:40.842Z`, `python.execute` — `"deleting vid_columns_v3"`,
  `"deleting vid_statues_v2"`, `"remaining: ['TT_Column_test12']"`. The caller had to
  leave the RPC surface entirely and drive `UMoviePipelineQueueSubsystem::DeleteJob`
  through `python.execute` to make its own render safe.
- What the two stale jobs would have overwritten is on disk and measurable: the
  `mrq.run_jobs` response spilled at
  `Saved/PinWright/HttpResponses/20260902T092641Z/20260902T101655Z_bdd51126-4ded-4e6f-461b-6a84f1758c50.json`
  shows `vid_columns_v3` → 240 files `Saved/MovieRenders/Video/src/columns/columns.0000-0239.png`
  (125,386,037 B) and `vid_statues_v2` → 240 files `.../statues/statues.0000-0239.png`
  (128,581,689 B). Both directories still hold exactly 240 frames. A `run_jobs` at 17:42
  would have spent minutes re-rendering **480 frames / 254 MB** the caller never asked
  for and rewritten both deliverables in place.

## Severity

**High.** Impact class is the rubric's "hard blocker with no workaround, so a reasonable
task is impossible" (High or Medium) — "render only the job I just queued" cannot be
expressed at all, and the observed workaround (`python.execute` plus an engine subsystem
call) is outside the tool, not inside it. It is pushed to the top of that band by two
aggravating facts the rubric weighs elsewhere: the failure is **destructive** — it
rewrites deliverables already on disk, in a namespace whose whole purpose is producing
them — and it is **silent**, since `queueSize` is a correct number with no name, no
contents and no warning attached.

The reach modifier is argued **neutral, not negative**. `mrq` is a small experimental
namespace, which would normally read as a rare edge path and bump the rating down. But
the queue is editor-global and never emptied, so this is the namespace's *normal* path
rather than an edge: from the second `create_job` of any editor session onward, every
`run_jobs` carries it, and the risk compounds with session length. A reader who weighs
namespace rarity above that would land on Medium; the destructive, no-workaround pair is
why this is filed High.

**Workaround:** `python.execute` →
`GEditor.get_editor_subsystem(unreal.MoviePipelineQueueSubsystem).get_queue()`, enumerate
`get_jobs()`, `delete_job(...)` every entry you did not create, then `mrq.run_jobs`.
Verify by re-enumerating; nothing in the `mrq` response confirms it.

**Fix:** three asks, in priority order.

1. **`mrq.list_jobs`** — enumerate the queue: index, `jobName`, sequence path, map path,
   configuration/preset, and the resolved `outputDirectory`/`fileNameFormat` already read
   by `ReadPreflightContext` for `create_job`'s `preflight` block. Without this the caller
   cannot even see what `queueSize` counts.
2. **Job selection on `run_jobs`** — `run_jobs {jobs: [...]}`, or `mrq.remove_job`.
   Selection is preferable: it is non-destructive to a sibling agent's queued work,
   whereas removal makes one caller delete another's job to protect its own. If selection
   is taken, the natural implementation is a scoped `SetJobEnabled` around
   `RenderQueueWithExecutor` with restoration on every exit, matching the plugin's
   existing scoped-pin idiom.
3. **`create_job` publishes what else is queued** — alongside `queueSize`, a
   `queuedJobs[]` of `{index, jobName, sequencePath}` for the entries this call did not
   create, plus a `warnings[]` entry when that list is non-empty naming the count and
   saying `run_jobs` will render them too. This is the same disclosure the handler already
   performs for the encode (`preflight`, `MRQHandler.cpp:323-335`), applied to the other
   thing that decides what the render costs, and it is the cheapest of the three.

Stopgap regardless of which lands: `Docs/wiki-src/mrq.md` documents neither `queueSize`
nor the shared queue's persistence. The namespace page mentions "the shared queue" exactly
once, in passing, at `Docs/wiki-src/mrq.md:28`.

## Fix

- `MRQHandler.cpp` now registers `mrq.list_jobs`, `mrq.remove_job`, and `mrq.clear_queue`.
  `list_jobs` reports positional `index`, `jobName`, enabled state, sequence/map paths, preset and
  resolved configuration paths, plus the resolved output path shape. The removal and clear responses
  report what they removed and the resulting `queueSize`; both refuse mutation during an active render.
- `mrq.run_jobs` accepts optional `jobs: [index, ...]`. The deferred executor callback duplicates
  the shared queue into a transient selected-only queue, enables requested copies in the supplied
  order, removes unselected copies in reverse index order, and keeps that queue strongly referenced
  through completion. This is required because UE 5.8's PIE executor validates disabled jobs too;
  the shared queue is not mutated, and temporary copy state is restored on executor completion or
  allocation failure. Invalid, duplicate, empty, and out-of-range selections are refused before a
  ticket starts.
- `mrq.create_job` refuses with `MRQ_RENDER_IN_PROGRESS` while an executor is active, so appending
  cannot race the shared queue used by an in-flight render.
- `mrq.create_job` now always returns `queuedJobs[]` for entries that existed before this call and
  adds a warning naming their count and explaining that enabled entries among them may also be
  rendered by an unfiltered `mrq.run_jobs`.
- Added editor-context automation coverage in `Tests/Media/TestMRQHandlers.cpp` plus a synchronous
  no-PIE test executor fixture:
  `PinWright.mrq.list_jobs.ReportsQueueEntries`,
  `PinWright.mrq.create_job.DisclosesPreviouslyQueuedJobs`,
  `PinWright.mrq.run_jobs.RejectsEmptySelection`,
  `PinWright.mrq.run_jobs.FiltersSelectedJobs`,
  `PinWright.mrq.run_jobs.RestoresSelectedCopyOnCompletion`,
  `PinWright.mrq.remove_job.RemovesIndexedEntry`, and
  `PinWright.mrq.clear_queue.RemovesJobs` (the last skips when shared queue state is non-empty).
  Updated `Plugins/PinWright/docs/wiki-src/mrq.md` with the shared-queue contract.

Static diff/search checks passed. Live UE compilation and automation execution were skipped because
this fix task forbids UnrealEditor-Cmd/live editor runs; the listed tests remain available for the
tester to execute in an MRQ-enabled editor.

## Related

- `B-mrq-render-result-omits-bitrate-and-size` (IN-REVIEW/High) — its `#3` states this
  ticket's hazard as the rationale for refusing an unloadable preset, and files nothing
  about it. Same handler, one step earlier.
- `E-mrq-run-jobs-doc-promises-per-job-exit-status` (OPEN/Low) — its workaround, "run one
  job per queue (single `mrq.create_job` before each `mrq.run_jobs`)", silently assumes
  the queue starts empty; this ticket is why that assumption does not hold.
- `F-mrq-render-queue` (DONE/Low) — the origin feature ticket that specified exactly these
  three verbs. No management verb was ever proposed, so this is a gap in the original
  design rather than a regression.
- `F-mrq-preset-authoring` (OPEN/Medium) — the other capability the three-verb surface
  cannot reach.

## History
- `#1-initial-repro` `OPEN` reporter — `mrq.create_job` allocates into the editor-global, session-persistent MRQ queue (`MRQHandler.cpp:238` → `:244` → `Queue->AllocateNewJob` `:289`) and `mrq.run_jobs` renders **all** of it: its only parameter is `executorClass` (`:357-359`) and it calls `RenderQueueWithExecutor` (`:410`) with no job id, no job list and no filtering. No verb lists, removes or clears a job — grep for `DeleteJob`/`DeleteAllJobs`/`SetJobEnabled`/`list_jobs`/`remove_job`/`clear_queue` returns zero hits in handler code (three total, all test-fixture cleanup at `Tests/Media/TestMRQHandlers.cpp:45`, `:394`), and `REGISTER_RPC_HANDLER("mrq` returns exactly three verbs (`:205`, `:344`, `:466`). The only signal is `queueSize` (`:307-309`, emitted `:317`; repeated in the `run_jobs` started payload `:396`), which counts foreign jobs, carries no names or output paths, triggers no warning, and is documented **nowhere** — zero occurrences in `Docs/` or in `Saved/PinWright/wiki/mrq.create_job.md` / `mrq.run_jobs.md`. The hazard is already written in the source as a comment no caller can read (`MRQHandler.cpp:266-267`, "the misconfigured job would sit in the shared queue as a trap for anyone's later mrq.run_jobs"). Field evidence 2026-09-02: the `queueSize: 3` observation is RELAYED from the rendering agent and not reproduced here (read-only probes only, no MRQ runs), but it is corroborated first-hand from this checkout — `Saved/PinWright/jobs.jsonl` L67 (`17:32:28.853Z`, `python.execute`) enumerates the queue as `vid_columns_v3` / `vid_statues_v2` / `TT_Column_test12`, and L69 (`17:32:40.842Z`) records `"deleting vid_columns_v3"`, `"deleting vid_statues_v2"`, `"remaining: ['TT_Column_test12']"` — i.e. the caller had to leave the RPC surface for `python.execute` + `UMoviePipelineQueueSubsystem::DeleteJob` to make its own render safe. Had it not, the spilled response `20260902T101655Z_bdd51126-…json` shows the two stale jobs would have re-rendered and overwritten 480 frames / 254 MB in `Saved/MovieRenders/Video/src/columns/` and `.../statues/`, both of which still hold exactly 240 frames. Severity High: rubric "hard blocker with no workaround" (High or Medium), pushed to High because the failure is destructive of deliverables already on disk and silent; reach modifier argued neutral rather than negative because the queue is editor-global and never emptied, so this is the namespace's normal path from the second `create_job` of a session onward, not an edge. Asks: `mrq.list_jobs`; job selection on `run_jobs {jobs:[…]}` (preferred over `mrq.remove_job`, which makes one caller delete another's work); and `create_job` publishing `queuedJobs[]` plus a warning when the queue already holds entries, the same disclosure it already performs for the encode (`:323-335`).
- `#2-shared-queue-management` `IN-REVIEW` developer — Implemented queue enumeration, selected-only transient run queues with temporary-copy state restoration, active-render create refusal, create-time stale-entry disclosure, and trivial queue removal/clear verbs; added no-PIE behavior coverage and updated the MRQ wiki contract. Live UE build/automation remains for tester verification.
- `#3-queue-safety-corrections` `IN-REVIEW` developer — Corrected selected rendering for UE 5.8 PIE's all-job validation by duplicating the shared queue, retaining the transient queue through completion, and removing unselected copies; added synchronous no-PIE filtering coverage, nonempty clear behavior coverage, and registered `MRQ_RENDER_IN_PROGRESS` for the create/run/remove/clear guard.
