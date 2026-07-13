---
id: H-fix-stop-procedure-missing-ticket-reconcile
title: "mcp-fix-workflow's user-stop procedure never reconciles the mid-flight ticket — a deliberate stop strands it IN-REVIEW (invisible to all pickers) with its fix never pushed"
status: OPEN
severity: Medium
workflow: fix
category: harness-process
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: supervisor@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/SKILL.md
---

# mcp-fix-workflow's user-stop procedure never reconciles the mid-flight ticket

`## Stopping the workflow (main conversation)` has exactly 3 steps: disarm supervision →
`TaskStop` → scoped editor kill. If the run is mid-pipeline on a GO ticket (the common case),
the ticket is already `IN-REVIEW` (flipped at Implement, by design — TTL-free cross-host
exclusion) but its fix is still an uncommitted diff in the plugin clone. The stop procedure
leaves it that way: `IN-REVIEW` + fix-not-on-origin = invisible to every host's picker, and
the diff is swept as trash by the next Baseline. The safety nets don't cover a clean user stop:

- the startup resume-reconcile (CASE A) only runs when THIS host launches another run — which
  after a deliberate stop may be days away or never;
- the supervisor's dirty-death reconcile runs on the STOPPED/ENDED classification path, which
  step 1 of the stop procedure deliberately disarms.

So a clean, by-the-book stop silently loses the ticket from the retry pool.

## Evidence

Run `wf_510c6c2c-eb5` (fuzz2, 2026-07-13): user requested stop while the pipeline was in the
Diff-Review/Test tail of `B-source-control-revert-no-package-reload` (implement + local
verification complete, nothing pushed). The 3 stop steps executed as written left the ticket
stranded `IN-REVIEW` with `claimedBy: fuzz2`; the supervisor reopened it manually (history
`#3-attempt-abandoned`, board commit "attempt-failed reopen") — knowledge imported from the
problem-scan section, not from the stop procedure being followed.

## Proposed edit

In `.polyskill/skills/mcp-fix-workflow/SKILL.md`, append step 4 to
`## Stopping the workflow (main conversation)`:

> 4. **Reconcile the mid-flight ticket (board-only).** For any ticket `IN-REVIEW` +
> `claimedBy: <this hostId>` whose fix is NOT on the plugin origin
> (`git -C <pluginClone> log origin/master --oneline --grep "<ticket-id>"` after a fetch —
> empty = not landed): flip it back to `OPEN`, append a `#N-attempt-abandoned` history line
> (deliberate user stop; fix never reached origin; diff will be swept by the next Baseline),
> remove `claimedBy`/`claimedAt`, and commit via `board-commit.ps1`
> (`<ticket-id> attempt-failed reopen`). Discard NOTHING in the clone — the next run's
> Baseline sweeps it. If the fix IS on origin, leave the ticket `IN-REVIEW` and just clear a
> lingering lease (bare write, not committed).

Mirror of the run-internal abandon step and resume-reconcile CASE A — same wording can be
reused nearly verbatim.

## History
- `#1-filed-from-live-stop` `OPEN` supervisor — Filed after the 2026-07-13 user stop of wf_510c6c2c-eb5 required a manual reopen of B-source-control-revert-no-package-reload; the stop section's 3 steps never mention the board. Gap became fully unguarded once the supervision rewrite scoped the old problem-scan reconcile to the (disarmed-on-user-stop) STOPPED/ENDED path.
