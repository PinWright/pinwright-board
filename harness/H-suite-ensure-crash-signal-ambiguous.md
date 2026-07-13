---
id: H-suite-ensure-crash-signal-ambiguous
title: "Test-loop prompt lists 'ensure mid-suite' as a crash signal, forcing agents to improvise unverified 'pre-existing' waivers for routine non-fatal ensures"
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

# Test-loop prompt lists 'ensure mid-suite' as a crash signal, forcing agents to improvise unverified 'pre-existing' waivers for routine non-fatal ensures

Step 5 says 'Also detect a crash (log ends abruptly / Fatal / assertion / ensure mid-suite)' — but non-fatal ensures fire routinely (expected-error tests, benign engine paths), so a literal reading would fail every green run. The agent contradicted the letter: it found 3 mid-suite 'Ensure condition failed' lines — including one in the plugin's own RpcDispatcher.cpp:224 — and waved all three through as 'pre-existing and unrelated' from a 2-line grep context, with no baseline-log comparison and no check that a green expected-error test owns the ensure's window. The same improvised waiver would normalize a genuinely NEW fix-introduced ensure (a missed-regression channel), and the 'pre-existing' claim is an unverified assertion in the durable summary. Latent in every full-suite run that logs ensures.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a03a8c066a2d40da6.jsonl
- quote: "Three non-fatal ensures observed (... a PinWright RpcDispatcher reentrancy-guard test path) are pre-existing and unrelated"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace: `Also detect a crash (log ends abruptly / Fatal / assertion / ensure mid-suite).` -> with: `Also detect a crash (log ends abruptly / Fatal error / assertion failure). ENSURE POLICY: non-fatal 'Ensure condition failed' lines do not fail the run by themselves, but never label one 'pre-existing' without evidence — for each ensure in PLUGIN source (Plugins\\PinWright\\Source), attribute it by timestamp to the surrounding Test Started/Completed window and confirm that green test deliberately exercises the guarded path, OR show the identical ensure signature in a prior run's log; if you can do neither, treat it as introduced by this diff — investigate and fix (or return STUCK-TEST) instead of tolerating it. List every tolerated ensure and its evidence in your summary.`

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
