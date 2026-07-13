---
id: H-host-repo-never-synced-at-launch
title: "Neither workflow ever pulls the HOST repo — Baseline/Prep sync only the plugin clone, so harness changes (skills, helper scripts) pushed to the hub never reach sibling hosts until a manual sync"
status: OPEN
severity: Medium
workflow: both
category: harness-process
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: supervisor@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/SKILL.md
  - .polyskill/skills/mcp-test-workflow/SKILL.md
---

# Neither workflow ever pulls the HOST repo — harness changes don't propagate to the fleet

The fix workflow's Baseline (`mcp-fix-workflow.workflow.js` step 2: `reset --hard` +
`clean -fd -e .claude` + `lfs checkout`) and the test workflow's Prep (step B, same commands)
restore the host repo to its **local committed HEAD** only; the `pull --rebase origin master`
each iteration applies **only to the plugin clone**. Host-repo content — the harness itself
(`.polyskill/` skills, board-commit.ps1/build-plugin.ps1 copies, CLAUDE.md) — propagates
any-host → hub → others strictly manually (`git pull --rebase` + `npx polyskill` per host, or
`sync-fuzz-plugins.ps1`). Consequence: a harness fix pushed from one host (e.g. an applied
audit batch) silently never takes effect on the other three; they keep filing/fixing with old
prompts until someone remembers to sync — an unbounded staleness window that defeats the point
of pushing harness fixes to the hub.

Why the naive fix is wrong: a mid-run host pull alone changes almost nothing (agents read
protocol files from the generated `.claude/skills/`, which only a `npx polyskill` regen
refreshes, and the running workflow's js is in memory from launch), while pull+regen mid-run
creates version skew — fresh protocol files driven by an old in-memory script.

## Proposed edit

Sync at **run boundaries**, not in Baseline/Prep: in BOTH skills' SKILL.md, add to the Launch
preflight AND to the supervisor's transient-relaunch path (before invoking the Workflow):

> Host-repo sync: `git -C <repo-root> pull --rebase origin master` (retry transient network
> errors; a conflict with local uncommitted skill edits = stop and tell the user), then if the
> pull advanced HEAD run `npx --yes polyskill` in `<repo-root>\.polyskill\` and re-read this
> skill before launching — the run must start from the fleet-current harness.

Each run stays internally consistent (launch-time snapshot), and the fleet converges
automatically at every launch/relaunch instead of never.

## Evidence

Run `wf_510c6c2c-eb5` session (fuzz2, 2026-07-13): harness-audit rework was pushed to the hub
as `8924e2c`; upstream simultaneously carried 4 sibling-host harness commits
(`3f4093a..3f5e382`) that fuzz2 only received because a push conflict forced a manual
`pull --rebase` — no workflow mechanism would ever have delivered them. Conversely fuzz1/3/4
will not receive `8924e2c` until manually synced.

## History
- `#1-filed-from-design-review` `OPEN` supervisor — Filed during the harness-audit rework review (user question "do fix/test workflows pull the main repo in first agent or only plugin repo?"). Verified in both workflow sources: host repo is reset to local HEAD only, never pulled; only the plugin clone is synced per iteration.
