---
id: E-save-all-sync-fast-path
title: "editor.save_all returns a job ticket for sub-second saves; sync-when-small would remove polling overhead"
status: DONE
severity: Low
category: ergonomic
tags: [editor, save-all, jobs, ergonomics, sync-fast-path]
---

# editor.save_all returns a job ticket for sub-second saves; sync-when-small would remove polling overhead

`editor.save_all` is registered with `Ctx.StartJob(Args)` and returns a
`ticket_id` synchronously, forcing callers to poll `system.job_status`.
In practice most invocations complete in well under a second — an observed
run with 1 dirty asset took 403 ms (job `j_20260521T133204_871b0fbe`:
`started_at`=`13:32:04.353Z`, `completed_at`=`13:32:04.756Z`) — so the
poll roundtrip and the 2-second human-pacing wait dominate elapsed time
and waste tool calls. User feedback during the session ("why you were
waiting on that save_all when it is instant?") flagged the friction
directly.

Inspecting the handler at
`Source/PinWright/Private/Handlers/Editor/EditorCommandHandler.cpp:446-468`,
the "async" wrap is `AsyncTask(ENamedThreads::GameThread, ...)` calling
`EditorSaveAllDiagnostic::SaveDirtyPackagesWithIntegrityGate` (declared
in `EditorSaveAllDiagnostic.h`). The save itself runs on the game thread
— same thread as the dispatcher — so the ticket pattern defers work by
one tick without actually backgrounding it. Because UE's save pipeline
is game-thread-bound, even large saves don't get any concurrency benefit
from the wrap; they just block longer.

Two options:
- (a) **Sync-when-small fast path:** execute inline and return the result
  when the operation completes within a short wall-clock budget (e.g.
  1 s); fall back to a job ticket only when the budget is exceeded.
  Generalisable as a `StartJob` option for any cheap-most-of-the-time
  handler.
- (b) **Always-sync `editor.save_all`:** drop the `StartJob` wrap entirely
  since the work is naturally synchronous on the game thread. Simpler,
  but loses the safety valve for genuinely slow saves (huge content
  trees, source-control hooks). HTTP timeout is 120 s per
  `EditorAutomationRpcGatewaySettings`, which is probably enough head
  room for any realistic Save All; if not, (a) is the safer choice.

Hint, not in scope here: an audit of other handlers that are wrapped as
jobs but usually finish in well under a second (e.g. `asset.dump` for a
single asset) would likely surface more candidates for the same pattern.

**Workaround:** Caller polls `system.job_status` immediately after the
start; on small editors the first poll already returns `completed`.

**Fix:** Prefer (a). Add an optional `SyncBudgetSeconds` field to
`FJobBindArgs` (or a sibling `Ctx.StartJobOrInlineIfFast` helper) that
runs the bound delegate on the calling stack and returns the completion
result directly when it finishes inside the budget; otherwise emit the
ticket as today. Apply to `editor.save_all` first.

## History
- `#1-sync-fast-path-proposal` `OPEN` reporter — `editor.save_all` ticket pattern wastes a roundtrip on instant saves (403 ms observed, user complained mid-session). Handler at `EditorCommandHandler.cpp:277-299` wraps a game-thread `AsyncTask` in `Ctx.StartJob`, so the work is logically synchronous. Propose a `SyncBudgetSeconds` option that returns inline when the bound delegate completes within the budget and falls back to a ticket otherwise. Hint: similar audit may find other sync-when-small candidates (e.g. `asset.dump`).
- `#2-always-sync-implemented` `IN-REVIEW` developer — Implemented as option (b) per user direction — dropped the async-job machinery entirely rather than add `SyncBudgetSeconds` budgeted fast-path. Rationale: simpler one-handler change; HTTP timeout (120s default per `EditorAutomationRpcGatewaySettings`) is acceptable upper bound for save operations. Handler at `EditorCommandHandler.cpp:277-296` now calls `EditorSaveAllDiagnostic::SaveDirtyPackagesWithIntegrityGate` on the calling stack and forwards the existing `BuildSaveAllResultJson(...)` payload via `Ctx.SendSuccess` (or `Ctx.SendError("SAVE_FAILED", ...)` on failure, with the result attached for diagnostics). Removed `FJobBindArgs`, `StartedPayload`, the `AsyncTask(ENamedThreads::GameThread, ...)` wrap, and the `Ctx.StartJob(Args)` call. Trade-off: callers that polled `system.job_status` for save_all completion now receive the result inline instead — observable response-shape change (no `ticket_id` / `status:"running"` / `monitor_path`). Updated wiki overlays at `system.md` and `safe-mutation-save.md` to reflect the synchronous contract (other overlays only mentioned `save_all` semantically without referencing the job envelope, so were correctly skipped). Added sibling test `FEditorSaveAllRespondsSynchronouslyTest` in `TestEditorHandlers.cpp` asserting the response shape lacks job-envelope fields. Method summary string updated.
- `#3-verified-sync-response` `DONE` tester — Called `editor.save_all` mid-session with one dirty asset (`W_AnalyzerControls`). Response: `{"success":true,"savedCount":1,"totalDirty":1,"pieActive":false,"editorMode":"Editor"}` returned inline — no `ticket_id`, no `status:"running"`, no `monitor_path`. Asset persisted to disk. User-visible roundtrip dropped from "submit + poll-with-2s-sleep loops" to a single synchronous call.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. **Not just a path — the code the body quotes is gone because this ticket's own request landed.** There is no `AsyncTask` anywhere in `Handlers/Editor/`; the only surviving token is the comment at `:451` explaining why backgrounding is *not* used. `editor.save_all` (`:447-468`) asserts `check(IsInGameThread())` and calls `SaveDirtyPackagesWithIntegrityGate` (shared routine `:237-284`) inline. The cited `:277-299` is now that routine's tail plus the head of the `editor.console_command` registration. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
