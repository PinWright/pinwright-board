---
id: H-differential-gate-compile-only-proof
title: "STEP 4b differential gate is satisfiable by a no-compile (missing new symbol), so new-helper fixes ship with no executed behavioral red"
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

# STEP 4b differential gate is satisfiable by a no-compile (missing new symbol), so new-helper fixes ship with no executed behavioral red

The implementer's differential check counts a pre-fix tree that fails to COMPILE (test references a fix-introduced symbol, C1083) as differentialVerified:true, so the test is never observed to FAIL behaviorally. Proven real in wf_510c6c2c-eb5: the lead's new helper contained a genuine overcount bug (reloadedCount on partial failure/duplicates) the green test could not detect — only the diff reviewers caught it. Structural: every fix extracting a NEW production symbol + a test including its header trivially hits the clause; the same clause is repeated in the test-loop's NEW-TEST DIFFERENTIAL. A behavioral red (stub the symbol body to a no-op, observe Fail) costs one bounded scoped rebuild+run.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a00ca8203879508ed.jsonl
- quote: "The green regression test only reloads a single always-succeeding package, so it cannot catch this."
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace (anchor verified at line 302): "If the pre-fix tree does not even COMPILE because the test references a symbol the fix introduced, that also demonstrates the dependency — count it as verified (note it)." -> with: "If the pre-fix tree does not even COMPILE because the test references a symbol the fix introduced, that demonstrates only the DEPENDENCY — not that the test can detect a wrong implementation. Do ONE bounded follow-up: pop the stash, stub ONLY the introduced symbol's body to a no-op/default return (keep the signature so the test compiles), rebuild, run ONLY the new test and observe Result={Fail}; then restore the real body (never leave the stub). Set differentialVerified:true with differentialNote:'stub-verified' only on that observed behavioral Fail; if budget genuinely forbids the stub run, set differentialVerified:true with differentialNote:'compile-dependency only — behavioral differential NOT executed' so the gap stays visible." Apply the same policy to the test-loop's NEW-TEST DIFFERENTIAL clause '(a pre-fix no-compile also counts)'.

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
