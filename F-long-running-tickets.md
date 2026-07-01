---
id: F-long-running-tickets
title: "Ticket + JSONL monitor for long-running RPCs"
status: DONE
severity: High
category: feature
tags: [async, jobs, long-running, monitoring, fire-and-forget]
---

# Ticket + JSONL monitor for long-running RPCs

Handlers that kick off slow operations (lighting builds, shader compilation, test runs, UBT compilation, navigation rebuilds, screenshot capture, level loads, saves) previously either blocked the HTTP connection until MCP's 120 s timeout killed the request, or returned `{started:true}` with no way to learn when or whether the operation completed. This left callers with no progress signal and forced them to either re-poll through unrelated heuristics or assume success.

This umbrella entry tracks the full ticket-system feature: a shared `FJobRegistry` that assigns each async operation a stable `jobId`; a JSONL event writer that appends structured completion/failure records to `.editor-automation/jobs.jsonl`; a `Ctx.StartJob` convenience helper that wires a delegate to the registry in one call; three new control RPCs (`system.job_status`, `system.job_list`, `system.job_cancel`); and the migration of 20 previously fire-and-forget handlers to use the new infrastructure. On subsystem startup the JSONL file is wiped so only jobs from the current editor session appear.

**Affected handlers:** `system.run_tests`, `system.run_ubt`, `level.build_lighting`, `level.build_level_lighting`, `lighting.build_lighting`, `level.build_navigation`, `navigation.rebuild_navigation`, `level.build_all`, `editor.screenshot`, `performance.optimize_shaders`, `performance.run_benchmark`, `pipeline.run_ubt`, `asset.dump_folder`, `level.load`, `editor.open_level`, `level.save`, `level.save_as`, `editor.save_all`, `render.nanite_rebuild_mesh`, `blueprint.build_api_index`.

**Removed dead code:** `system.test_progress_protocol` and `system.test_stale_progress` test-scaffolding handlers removed.

## History
- `#1-no-completion-signal` `OPEN` reporter — Long-running RPCs (system.run_tests, level.build_*, asset.dump_folder, ...) return `{started:true}` and never signal completion; HTTP times out at 120s; need a ticket+monitor mechanism.
- `#2-feature-implemented` `IN-REVIEW` developer — All 20 handlers migrated, infrastructure landed (`FJobRegistry`, `FJobMonitorLog`, `Ctx.StartJob`, `system.job_*` RPCs, settings additions, startup wipe). Dead test-scaffolding handlers removed. See per-component issues `F-job-registry`, `F-jsonl-monitor`, `F-handler-context-startjob`, `F-job-control-rpcs`, `F-jobs-settings`, `F-jobs-startup-wipe`, and per-handler `B-*-no-completion-signal` issues for implementation details.
- `#3-verified-umbrella-pass` `DONE` tester — Verified the umbrella by exercising 15 of the 20 affected handlers live this session and 5 more via discovery schemas (live execution skipped because they would mutate or launch external processes — pipeline.run_ubt, system.run_ubt, render.nanite_rebuild_mesh, level.save_as, level.load). All 6 sub-component F-tickets (F-job-registry, F-jsonl-monitor, F-handler-context-startjob, F-job-control-rpcs, F-jobs-settings, F-jobs-startup-wipe) and F-test-progress-protocol-removed verified DONE in this same review pass. Per-handler `B-*-no-completion-signal` tickets verified individually.
