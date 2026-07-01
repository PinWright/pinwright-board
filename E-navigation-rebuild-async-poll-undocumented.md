---
id: E-navigation-rebuild-async-poll-undocumented
title: "navigation wiki overlay never documents the rebuild_navigation async ticket → system.job_status poll pattern"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [navigation, rebuild_navigation, docs, async, jobs, job_status, discoverability]
---

# navigation wiki overlay never documents the rebuild_navigation async ticket → system.job_status poll pattern

`navigation.rebuild_navigation` is an async-job handler: per the now-DONE bug
`B-navigation-rebuild-navigation-no-completion-signal` it `Ctx.StartJob()`s and
registers a game-thread ticker that polls
`FNavigationSystem::IsNavigationBuildInProgress()` until the flag clears, then
calls `FJobRegistry::CompleteJob`. So the call returns a job **ticket
synchronously** (not a rebuild result); the real `{nav_built:true, …}`
disposition is only knowable by separately polling `system.job_status` with that
ticket until it reaches a terminal `completed`. But the navigation namespace wiki
overlay that should teach a caller this two-call contract —
`docs/wiki-src/navigation.md` — says **nothing** about `rebuild_navigation` being
async, the ticket it returns, or the `system.job_status` follow-up
(`grep -niE "async|ticket|poll|job_status|rebuild_navigation"
docs/wiki-src/navigation.md` = no matches; the overlay's only mention of
`rebuild_navigation` is as the last step of the level-nav-setup order in the
RecastNavMesh-prerequisite section added by
`E-navmesh-config-requires-bounds-volume-prereq`, with no note that the step is
asynchronous).

The consequence is a discoverability gap (the async ticket→`system.job_status`
contract IS documented globally — `docs/wiki-src/system.md` lists
`navigation.rebuild_navigation` in its ticket-pattern table and the kickoff
response itself carries `docs: call("system.job_status") for guidance` — it is
just absent from the navigation overlay the level-nav author starts on). A caller
who needs the nav data to have actually regenerated before the next step (here:
reporting nav-system status to "confirm the nav data regenerated with the link in
place") has to **self-discover** the poll by falling through to the `system`
page. If they instead treat the synchronous ticket as "done" and immediately read
`get_navigation_info` / `actor.describe`, they could report a nav state captured
mid-rebuild — but the ticket/`job_status` mechanism is itself correct and emits no
wrong data on its own, so this is the same docs/discoverability friction the
sibling async-poll tickets carry, not a correctness defect. Pure friction on a
niche nav path → **Low/ergonomic**, matching the three siblings below (all Low).

This is the direct **navigation-namespace** analog of the already-filed
`E-pipeline-run-ubt-async-poll-undocumented` (pipeline overlay / `run_ubt`),
`E-performance-run-benchmark-async-poll-undocumented` (performance overlay /
`run_benchmark`), and `E-render-nanite-rebuild-async-poll-undocumented` (render
overlay / `nanite_rebuild_mesh`): same friction, same root pattern — an async
ticket→`system.job_status` contract that the namespace overlay never surfaces —
but a different overlay page (`navigation.md`) and a different async handler
(`rebuild_navigation`), so it is a distinct docs edit. It is also distinct from:

- `B-navigation-rebuild-navigation-no-completion-signal` (DONE) — that bug *added*
  the completion signal (the job + `CompleteJob` ticker poll); the signal now
  exists, but nothing in the navigation overlay tells a caller it exists or how to
  consume it. The discoverability gap survives that fix because the async response
  shape (synchronous ticket) is the permanent design.
- `E-navmesh-config-requires-bounds-volume-prereq` (IN-REVIEW) — that is the
  orthogonal *ordering-prerequisite* angle on the same overlay (the RecastNavMesh /
  NavMeshBoundsVolume precondition for the two config setters). This ticket is the
  *async-discoverability* angle on the **rebuild** verb, on the same page but a
  different section.

**What it should do:** The `docs/wiki-src/navigation.md` overlay should document,
for `navigation.rebuild_navigation`, that the call is fire-and-forget: it returns a
ticket synchronously (NOT a rebuild result), and callers who need the rebuild to
have finished (or who want the `{nav_built, …}` payload) must poll
`system.job_status {ticket_id}` until a terminal `status` before reading nav state
(`get_navigation_info`) or describing nav actors. Annotate the existing
level-nav-setup order line so the final `rebuild_navigation` step is marked async
("then poll `system.job_status`"). Cross-link the `system.job_status` doc and the
`F-long-running-tickets` job model so the two-call pattern is discoverable from the
navigation page a level-nav author starts on. Mirror the wording already proposed
in the three sibling async-poll tickets.

**Workaround:** After `navigation.rebuild_navigation`, poll
`system.job_status {ticket_id}` (from the kickoff response) until the status is
terminal before doing anything that depends on the rebuild having finished (e.g. a
final `get_navigation_info` status report); treat the synchronous ticket as
"accepted", never as "the rebuild finished".

## Process friction this caused (this task)

A two-tier scout-drone nav-setup task (seed `navigation`) ran a clean 12-call
sequence — `set_nav_agent_properties` (r24/h90/slope50/step45, first call failed
`[NO_NAVMESH]` then succeeded after a recovery bounds-volume — that ordering
friction is the separate `E-navmesh-config-requires-bounds-volume-prereq`),
`configure_nav_mesh_settings`, `create_nav_link_proxy` (LedgeDropLink),
`set_nav_link_type → smart`, `configure_smart_link_behavior`,
`rebuild_navigation`, a `system.job_status` poll (→ completed, `nav_built=true`),
and a final `get_navigation_info` + `actor.describe` verification — outcome clean.
The rebuild step's async-ness was pure process friction: quoting the friction note,
*"the rebuild is async (returns a ticket needing system.job_status polling) rather
than synchronous, which the wiki didn't flag upfront on the rebuild page."* The
agent had to self-discover the poll contract — nothing in the navigation overlay
pointed there — specifically because the task's final step reports nav-system
status to confirm the nav data regenerated with the link in place, which depends on
the rebuild having actually finished.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean two-tier scout-drone nav-setup task (seed `navigation`, 12 calls, outcome clean). Friction note (quoted): "the rebuild is async (returns a ticket needing system.job_status polling) rather than synchronous, which the wiki didn't flag upfront on the rebuild page." Verified `docs/wiki-src/navigation.md` (16 lines) documents the RecastNavMesh prerequisite + level-nav-setup order but never marks `rebuild_navigation` async; `grep -niE "async|ticket|poll|job_status"` = no matches — the async ticket→`system.job_status` poll contract is undocumented in the navigation overlay, so a level-nav author must self-discover the poll, a silent correctness trap since the task's final step reports nav status to confirm the rebuild baked the link in. Direct navigation-namespace analog of `E-pipeline-run-ubt-async-poll-undocumented`, `E-performance-run-benchmark-async-poll-undocumented`, and `E-render-nanite-rebuild-async-poll-undocumented` (distinct overlay page + handler); distinct from the DONE bug `B-navigation-rebuild-navigation-no-completion-signal` (signal now exists but is undocumented in the overlay) and from the IN-REVIEW `E-navmesh-config-requires-bounds-volume-prereq` (orthogonal ordering-prerequisite angle on the same page). Ergonomic/docs, not an outcome bug — every call in the task succeeded; the per-finding judge filed nothing (filed_id empty). Fix: document the async/ticket/poll contract on `docs/wiki-src/navigation.md`, annotate the rebuild step in the setup order as async, cross-link `system.job_status` and `F-long-running-tickets`.
- `#2-retriage` `OPEN` triage — Low→Medium: undocumented async ticket invites treating a sync ticket as done and reading mid-rebuild state, a silent correctness trap, niche nav path.
- `#3-reword-and-fix` `IN-REVIEW` developer — Reworded Medium→Low/ergonomic (reverting the #2 retriage): the async ticket→`system.job_status` contract IS documented globally (`system.md` lists `navigation.rebuild_navigation` in the ticket-pattern table and every kickoff response carries `docs: call("system.job_status") for guidance`), and the ticket/job_status mechanism emits no wrong data on its own, so this is pure docs/discoverability friction on a niche nav path — matching the three identical sibling async-poll tickets (`E-pipeline-run-ubt-async-poll-undocumented`, `E-performance-run-benchmark-async-poll-undocumented`, `E-render-nanite-rebuild-async-poll-undocumented`, all Low) and the board README severity rubric. Verified the defect is real and present: `NavigationHandler.cpp:335` calls `Ctx.StartJob(Args)` (binding `BindNavigationBuildCompletion(World)` + `NavSys->Build()`), returning a ticket synchronously; the pre-fix `navigation.md` mentioned `rebuild_navigation` only at the setup-order line with no async note. Fix (docs only): added a `## Rebuilding navigation is async` section to `docs/wiki-src/navigation.md` stating the verb is an async ticket-returning job (not blocking, `{nav_built:true}` only via the job), pointing callers at `system.job_status` polling before reading nav state, and cross-linking `system.md#long-running-jobs` / `system.md#systemjob_status` (where `navigation.rebuild_navigation` is in the ticket-pattern table) and `F-long-running-tickets`; annotated the setup-order line so the final `rebuild_navigation` step is marked async. Regression test `Source/PinWright/Private/Tests/Infra/TestNavigationRebuildAsyncDocs.cpp` (`PinWright.infra.wiki_handler.Namespace.NavigationRebuildAsync`) renders the `navigation` namespace page through the live `WikiHandler::RenderPage` path and asserts the page names `rebuild_navigation`, contains `ticket`+`async`, and cross-links `system.job_status` — it fails if the navigation.md edit is reverted. Mirrors the sibling `TestLevelLightingBuildAsyncDocs.cpp`.
</content>
</invoke>
