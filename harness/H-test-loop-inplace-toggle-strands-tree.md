---
id: H-test-loop-inplace-toggle-strands-tree
title: "Test-loop differential via in-place pre-fix toggles has no transactional restore — a mid-window death leaves deliberately broken production code that salvage/retry never detects"
status: OPEN
severity: Medium
workflow: fix
category: resume-restart
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Test-loop differential via in-place pre-fix toggles has no transactional restore — a mid-window death leaves deliberately broken production code that salvage/retry never detects

The NEW-TEST DIFFERENTIAL prescribes only pathspec git stash; when concern edits interleave with the base fix in shared files, the agent reasonably improvised labeled 'DIFFERENTIAL TOGGLE' in-place edits, backed the fixed .cpp up only to the session scratchpad, and was interrupted during the toggled rebuild — restoration never ran (goal_met=false, the partition's only real casualty). The harness retries a dead test-loop agent up to 5 times in-run (agentOrRetry(...,5)); the SALVAGE CHECK inspects only build-log tail + suite-log mtime, so a retry would compile the toggled tree and burn a full ~15+ min compile+suite cycle on phantom failures. The window opens on every differential whose concern edits share files with the base fix.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-afb2e32b9a00b6d9f.jsonl
- quote: "continue; // DIFFERENTIAL TOGGLE: pre-fix skipped reverted-add packages ... [Request interrupted by user for tool use]"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
Two additions. (1) replace: "run ONLY that new test and observe Result={Fail} (a pre-fix no-compile also counts), then git stash pop." -> with: "run ONLY that new test and observe Result={Fail} (a pre-fix no-compile also counts), then git stash pop. If a pathspec stash CANNOT isolate the concern fix (its edits are interleaved with the base fix in shared files), use labeled IN-PLACE toggles instead — but transactionally: FIRST `git add` every file you are about to toggle (snapshotting the fixed state in the index), mark every toggled region with the literal comment 'DIFFERENTIAL TOGGLE', and `git restore <file>` (worktree from index) IMMEDIATELY after observing the differential — the restore is part of this same step, never deferred past another build or suite run." (2) replace: "If either check fails, start the normal loop." -> with: "If either check fails, start the normal loop. (c) ALWAYS, before the first COMPILE: grep the plugin Source tree for the literal 'DIFFERENTIAL TOGGLE' — a hit means a prior attempt died mid-differential with pre-fix toggles still applied; `git restore` each toggled file (its fixed state was staged before toggling) before any compile or log parse."

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
