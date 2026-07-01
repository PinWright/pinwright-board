---
id: E-level-build-lighting-async-poll-undocumented
title: "level overlay carries a stale 'blocks the editor' build_lighting gotcha, never mentions build_level_lighting, and does not cross-link the existing system.job_status async-poll contract"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [level, lighting, build_level_lighting, build_lighting, docs, async, jobs, job_status, discoverability]
---

# level overlay carries a stale 'blocks the editor' build_lighting gotcha, never mentions build_level_lighting, and does not cross-link the existing system.job_status async-poll contract

`level.build_level_lighting` and `level.build_lighting` are async-job handlers: per
`F-long-running-tickets` (DONE) and the per-handler bugs
`B-level-build-level-lighting-no-completion-signal`,
`B-level-build-lighting-no-completion-signal` (DONE), they were migrated to
`Ctx.StartJob()` and now **return a job ticket synchronously** (not a bake result);
the real disposition is only knowable by separately polling `system.job_status`
with that ticket until it reaches a terminal `completed`/`failed`. But the `level`
namespace wiki overlay that should teach a caller this two-call contract —
`docs/wiki-src/level.md` — does the opposite:

1. **No async/poll contract on the level overlay.** `grep -niE "build_level_lighting|job_status|async|ticket|poll|bake"
   docs/wiki-src/level.md` returns no matches for `build_level_lighting`,
   `job_status`, `async`, `ticket`, or `poll`. The overlay never mentions
   `build_level_lighting` at all, never says either lighting-build verb returns a
   ticket, and never points at the `system.job_status` follow-up. The canonical
   async-poll contract IS documented — but on a different page: `docs/wiki-src/system.md`
   has a full `## Long-running jobs` section (lines 11-116) with a `### system.job_status`
   subsection (line 100) and a "methods that use the ticket pattern" table whose line 65
   lists both `level.build_lighting, level.build_level_lighting`. The gap is purely that
   the **level namespace overlay** — the page a lighting author actually lands on — does
   not surface or cross-link that existing contract, and (see #2) actively contradicts it.
2. **A stale, now-incorrect gotcha.** `docs/wiki-src/level.md` line 11 still says:
   *"`level.build_lighting` blocks the editor until the lightmap bake completes —
   can be tens of minutes on a large map. Do not call this during an interactive
   session unless you are prepared to wait."* After the `F-long-running-tickets`
   async migration this is factually wrong: the call no longer blocks — it returns a
   ticket immediately and the bake runs asynchronously. A caller who trusts this
   gotcha will expect a synchronous blocking call and a bake result, get a ticket
   instead, and have no overlay guidance on what to do with it.

This is the direct **level/lighting-namespace** analog of the already-filed
`E-navigation-rebuild-async-poll-undocumented`,
`E-performance-run-benchmark-async-poll-undocumented`,
`E-pipeline-run-ubt-async-poll-undocumented`, and
`E-render-nanite-rebuild-async-poll-undocumented` (same friction, same root
pattern — an async ticket→`system.job_status` contract the namespace overlay never
surfaces) — but a different overlay page (`level.md`) and different async verbs
(`build_level_lighting` / `build_lighting`), so it is a distinct docs edit. It also
goes one step further than its siblings: the level overlay does not merely *omit*
the contract, it *contradicts* it with the stale blocking-call gotcha.

It is distinct from the tool-bug ticket
`B-level-build-level-lighting-no-completion-signal` (OPEN, #4 regression): that bug
is that the handler's completion delegate never fires so the job never reconciles to
terminal. When that bug is fixed the job WILL go terminal — but the **discoverability
gap survives the fix**, because the async response shape (a synchronous ticket you
must poll) is the permanent design, and the stale gotcha and the missing poll
contract on `level.md` remain wrong/absent regardless of whether the delegate fires.
This ticket is the docs/ergonomic angle; the bug ticket is the handler angle.

**What it should do:** `docs/wiki-src/level.md` should (a) replace the stale
"blocks the editor … prepared to wait" gotcha with the truth — both lighting-build
verbs are fire-and-forget: they return a ticket synchronously (NOT a bake result),
and callers who need the bake to have finished must poll `system.job_status
{ticket_id}` until a terminal status before saving or reading lighting state; (b)
document `build_level_lighting` (currently unmentioned) alongside `build_lighting`
with the same async note; (c) cross-link the **already-existing** `system.job_status`
doc / `system` `## Long-running jobs` section (and the `F-long-running-tickets` job
model) so the two-call pattern is discoverable from the level page a lighting author
starts on. Mirror the wording already used in the four sibling async-poll tickets.

**Workaround:** After `level.build_level_lighting` / `level.build_lighting`, poll
`system.job_status {ticket_id}` (from the kickoff response) until the status is
terminal before saving or reading lighting state; treat the synchronous ticket as
"accepted", never as "the bake finished", and disregard the level overlay's stale
"blocks the editor" gotcha.

## Process friction this caused (this task)

A lighting-artist bake task (seed `level.build_level_lighting`) ran a clean sequence
— `level.list` → `level.load` (`/Game/Maps/Lighting/Lighting_Realtime`) →
`level.get_lighting_scenarios` (0) → `level.get_bounds` → `level.get_actors`
(54 baseline) → 3× `level.spawn_light` (Directional key, Point interior, Spot aimed)
→ `level.get_actors` (57, +3) → `level.build_level_lighting` (ticket
`j_...144900`) → 5× `system.job_status` polls → `level.save` (ticket completed,
`saved:true`) → `level.get_actors` + 3× `actor.find_by_class` readback. The lights
all landed; the only struggle was the bake step. The job ticket never reconciled to
a terminal state — `system.job_status`/`system.job_list` reported `running` with
empty `progress[]` for ~14 min while the editor log proved Lightmass finished cleanly
(957/957 mappings, 141 meshes, 0 errors at 14:50:55). The agent had to fall back to
**reading the editor log out-of-band** to confirm the bake actually succeeded.

The non-reconciling job itself is the OPEN handler bug
`B-level-build-level-lighting-no-completion-signal`. The PROCESS angle filed here is
the docs gap that compounds it: the `level.md` overlay neither documents the async
ticket→`system.job_status` poll contract nor warns that the bake is async — it
instead asserts the call *blocks*. Quoting the friction note: *"Wiki for
system.job_status promises status flips to completed/failed — it never did here, so
success-check (b) is unverifiable via the MCP. I had to read the editor log
(read-only investigation) to confirm the bake actually succeeded; that out-of-band
check should not be necessary and is itself a discoverability gap."* Even once the
handler bug is fixed, a lighting author landing on `level.md` is told the wrong
execution model and given no pointer to the poll loop.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a clean lighting-artist bake task (seed `level.build_level_lighting`, 34 calls, outcome tool_bug). All level/spawn calls succeeded; the only struggle was the bake step, whose job ticket never reached terminal (that non-reconciliation is the OPEN handler bug `B-level-build-level-lighting-no-completion-signal`, filed by the per-finding judge — not re-filed here). Distinct PROCESS angle: `docs/wiki-src/level.md` (a) never documents that `build_level_lighting`/`build_lighting` are async ticket-returning verbs needing `system.job_status` polling (`grep -niE "build_level_lighting|job_status|async|ticket|poll"` = no matches; `build_level_lighting` unmentioned), and (b) line 11 carries a stale "`build_lighting` blocks the editor … prepared to wait" gotcha that is factually wrong after the `F-long-running-tickets` async migration. Friction note (quoted): "I had to read the editor log (read-only investigation) to confirm the bake actually succeeded; that out-of-band check should not be necessary and is itself a discoverability gap." Direct level/lighting-namespace analog of `E-navigation-rebuild-async-poll-undocumented`, `E-performance-run-benchmark-async-poll-undocumented`, `E-pipeline-run-ubt-async-poll-undocumented`, `E-render-nanite-rebuild-async-poll-undocumented` (distinct overlay page + verbs), and one step worse — the overlay actively contradicts the async model rather than merely omitting it. Survives the handler-bug fix because the synchronous-ticket response shape is the permanent design. Ergonomic/docs. Fix: rewrite the level.md lighting-build gotcha to document the async/ticket/poll contract for both verbs and cross-link `system.job_status` + `F-long-running-tickets`.
- `#2-retriage` `OPEN` triage — Low→Medium: docs overlay actively contradicts the async model with a stale blocking-call gotcha, misleading on the lighting-build path.
- `#3-reword-and-fix` `IN-REVIEW` developer — REWORD then implemented. Two of three validity lenses flagged a false claim in the body: the ticket asserted `system.md` "carries no `job_status` documentation … there is no page anywhere that teaches the ticket→poll loop." That is provably false — `docs/wiki-src/system.md` has a full `## Long-running jobs` section (lines 11-116) with `### system.job_status` (line 100) and a ticket-pattern table whose line 65 lists both `level.build_lighting, level.build_level_lighting`. Reworded title/body/severity-justification to the true narrow scope (level overlay carries a stale blocking gotcha + omits build_level_lighting + does not cross-link the EXISTING system.job_status contract), dropping the "undocumented anywhere" framing; left severity Medium (the active contradiction is genuinely worse than the sibling omissions). FIX (docs-only): rewrote the `**Gotchas**` block in `docs/wiki-src/level.md` (the stale "build_lighting blocks the editor until the lightmap bake completes … prepared to wait" gotcha) to state both lighting-build verbs are async ticket-returning jobs (NOT blocking, NOT a bake result), name `build_level_lighting` alongside `build_lighting`, instruct callers to poll `system.job_status {ticket_id}` to a terminal status before saving/reading lighting state, and cross-link the already-existing `system.md` `## Long-running jobs` / `system.job_status` docs. The gotcha sits under `## Cross-cluster overlap`, so it renders on the namespace page but not the root index (no root-budget impact). Test: `Source/PinWright/Private/Tests/Infra/TestLevelLightingBuildAsyncDocs.cpp` (`PinWright.infra.wiki_handler.Namespace.LevelLightingBuildAsync`) renders the level overlay through the live `WikiHandler::RenderPage` path and asserts both verbs are named, `async`/`ticket`/`system.job_status` are present, and the stale "blocks the editor until the lightmap bake completes" / "prepared to wait" phrasings are gone — reverting the level.md edit fails it. Files: `docs/wiki-src/level.md`, `Source/PinWright/Private/Tests/Infra/TestLevelLightingBuildAsyncDocs.cpp`.
