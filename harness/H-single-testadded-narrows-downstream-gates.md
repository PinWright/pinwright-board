---
id: H-single-testadded-narrows-downstream-gates
title: "testAdded contract: a single string described as a 'path' — narrows the NEW-TEST ASSERTION GATE and exempts sibling tests from the differential check"
status: OPEN
severity: Medium
workflow: fix
category: harness-process
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# testAdded contract: a single string described as a 'path' — narrows the NEW-TEST ASSERTION GATE and exempts sibling tests from the differential check

Merged weakness (single-testadded-narrows-downstream-gates + implementer-testadded-name-vs-path). The implement Return contract says 'testAdded (path or "none")' while the GO branch and the test-loop's NEW-TEST ASSERTION GATE need the automation-test NAME (a literal 'path' reading would make the gate report the test absent -> STUCK-UNVERIFIABLE on a green fix); and it holds ONE string while GO fixes legitimately add multiple tests (NEW-BEHAVIOR COVERAGE mandates it). In wf_510c6c2c-eb5 the lead added two tests but could return only one: the sibling's fails-without-fix property was reasoned, never executed (STEP 4b is skipped once differentialVerified:true), and the assertion gate hard-verified only the named test — a vacuous or silently-absent sibling would still let the run publish CLEAN. Structural on every GO adding more than the adopted red test.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a5e49f1951cd1fb59.jsonl
- quote: "added add_blend_sample.RejectedSampleReportsFailure yet returned testAdded:\"PinWright.animation.authoring.add_aim_offset_sample.RejectedSampleReportsFailure\""
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
Three replacements: (1) anchor verified line 307: replace `testAdded (path or "none")` -> `testAdded (comma-separated list of EVERY automation-test NAME you added — PinWright.<group>.<case>, NOT a file path; the adopted red test AND any sibling test you wrote — or "none"; the Test phase greps the suite log for exactly these strings, so a file path here makes its assertion gate report the test absent)`. (2) anchor line 299: replace `skip when differentialVerified is already true from STEP 4` -> `the adopted red test is differential by construction, but EVERY ADDITIONAL test you wrote in STEP 2 still needs this check — run it for each sibling test, or label it explicitly in differentialNote as 'differential REASONED, not executed: <test>'; the adopted test's red-green never exempts your sibling tests`. (3) anchor verified line 407: replace `This ticket added the automation test \`${testAdded || '(none reported)'}\`. A CLEAN result additionally REQUIRES that THIS specific test actually executed AND asserted — see the NEW-TEST ASSERTION GATE in step 5.` -> `This ticket added the automation test(s) \`${testAdded || '(none reported)'}\` (may be a comma-separated list). A CLEAN result additionally REQUIRES that EACH listed test actually executed AND asserted — see the NEW-TEST ASSERTION GATE in step 5.` — and update the step-5 NEW-TEST ASSERTION GATE wording to iterate over EACH listed name.

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
