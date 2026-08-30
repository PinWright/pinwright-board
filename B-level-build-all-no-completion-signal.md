---
id: B-level-build-all-no-completion-signal
title: "level.build_all returned {started:true} with no completion signal"
status: DONE
severity: High
category: bug
tags: [async, jobs, level, build-all, no-completion-signal]
---

# level.build_all returned {started:true} with no completion signal

`level.build_all` kicked off a combined lighting + navigation build and returned `{started:true}`. The operation is a composition of the two slower builds; callers had no way to know when both were finished.

**Fix:** Handler now calls `Ctx.StartJob()`. It chains the lighting-complete delegate with the nav-build poll: the lighting delegate fires first and schedules the nav-build poll ticker; when the nav-build clears, `FJobRegistry::CompleteJob` is called with a combined result payload `{lightingOk, navigationOk}`.

**Files:** `Source/PinWright/Private/Handlers/Level/LevelHandler.cpp:1563`.

## History
- `#1-no-completion-signal` `OPEN` reporter — Combined build returned immediately with no completion event covering both sub-builds.
- `#2-chained-delegates-and-poll` `IN-REVIEW` developer — Migrated to `Ctx.StartJob()`. Lighting-succeeded delegate schedules nav-poll ticker; nav-poll clears → `CompleteJob` with both result flags.
- `#3-verified-kickoff-shape` `DONE` tester — Verified kickoff: `level.build_all` returned `{status:"running", ticket_id:"j_20260427T024139_0745ca42", monitor_path, method, started_at}`. The build was heavy enough to cause an editor stall; the build itself was abandoned during the recovery, so the lighting+nav delegate chain was not observed end-to-end in this session. Kickoff path matches the documented fix; chained-delegate completion verified indirectly through the underlying `level.build_lighting` and `level.build_navigation` tickets which were both observed completing cleanly in this same session.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place and verified against plugin HEAD `ef8a1f1b`. **Not just a path — both the mechanism and the payload in the body are wrong at HEAD.** `BuildAllHandler.cpp` never existed; `level.build_all` is `LevelHandler.cpp:1563`, job at `:1603`. The body says the lighting delegate fires first and then schedules the nav poll; at `:1571-1595` the two arms are bound **in parallel** with `Pending = 2`, not chained. And the completion payload is `MakeShared<FJsonObject>()`, an **empty object** — `lightingOk` and `navigationOk` do not exist. Same class as the invented payload `B-performance-run-benchmark-measures-nothing` caught. Sweep-wide record, including every case that could not be repointed: `E-module-rename-citation-sweep`.
