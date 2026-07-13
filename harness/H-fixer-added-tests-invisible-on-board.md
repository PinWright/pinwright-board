---
id: H-fixer-added-tests-invisible-on-board
title: "Diff-review fixer's in-diff additions (a second regression test + a ticket-named symptom fix) never reach the board ticket — the reconciliation trigger only covers the test loop's OWN changes"
status: OPEN
severity: Medium
workflow: fix
category: prompt-instruction
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Diff-review fixer's in-diff additions (a second regression test + a ticket-named symptom fix) never reach the board ticket — the reconciliation trigger only covers the test loop's OWN changes

The fixer added FScopedEditorWorldActorGuard to the add_camera test AND a new regression test PinWright.sequencer.add_camera.LeavesLevelClean; the lead's history line predates the reviewers so it names only LeavesNoDirtyPackage. The fixer prompt gives it no board-history duty for IN-DIFF fixes, and the test-loop's BOARD RECONCILIATION trigger reads 'if YOU changed PRODUCTION code' — the loop touched nothing (touched:[]) and explicitly concluded no reconciliation was warranted. Result: the shipped commit contains a regression test and the fix for the ticket's explicitly-named L_Core.umap symptom that appear NOWHERE on the board — invisible to acceptance verification, future dedup, and the conventions-mandated acceptance-symbol greps. Structurally recurs whenever the fixer adds a test or fixes a reviewer-found symptom.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a1e9e46ad5fa486c2.jsonl
- quote: "no board reconciliation, history line, or title amendment is warranted"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace: "BOARD RECONCILIATION: if you changed PRODUCTION code beyond mechanical compile fixes (you fixed a reviewer-flagged concern, changed a handler's mechanism, or added a new test)," -> with: "BOARD RECONCILIATION: if the working tree's uncommitted diff contains ANY substantive change the ticket's current history lines do not describe — whether made by YOU or by the diff-review fixer before you (a reviewer-flagged concern fixed, a mechanism changed, a TEST ADDED: compare the diff's IMPLEMENT_*_AUTOMATION_TEST name-strings against the tests the ticket names),"

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
