---
id: H-testloop-redundant-final-fullsuite-after-restore
title: "Test loop re-runs the entire full suite after a sha1-verified byte-identical restore of a state whose full-suite green it already observed"
status: OPEN
severity: Low
workflow: fix
category: performance-cost
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Test loop re-runs the entire full suite after a sha1-verified byte-identical restore of a state whose full-suite green it already observed

When the cost-bounded differential runs LAST, the mandated sequence ends with a 'definitive final full-suite run' even after a sha1-verified byte-identical restore of the exact source state that already ran 3664/0 green minutes earlier in the same session — a redundant ~10-15 min cold-load suite run that re-validates nothing. Quality-neutral by construction (the earlier run on identical bytes IS the final full-suite evidence); re-running only the restored/differential tests proves the rebuilt binaries equally. Recurs on every iteration whose differential executes after the full suite.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a1e9e46ad5fa486c2.jsonl
- quote: "restored the file byte-for-byte (sha1 match), rebuilt clean, and re-ran the full suite to a definitive 3664 Success / 0 Fail green"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace (anchor verified at line 433): "but you MUST do one final FULL-suite run (the step-4 command) with no failures before declaring GREEN, so cross-cutting regressions are caught." -> with: "but you MUST do one final FULL-suite run (the step-4 command) with no failures before declaring GREEN, so cross-cutting regressions are caught. ONE exception: if your last edit was a byte-identical restore (verify with a sha1/hash match against your pre-differential backup) of a state whose FULL-suite green run you already observed THIS session, that earlier run IS the final full run — rebuild, then re-run only the restored/differential tests to prove the rebuilt binaries, and declare GREEN off that."

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
