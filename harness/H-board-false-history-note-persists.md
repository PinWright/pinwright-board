---
id: H-board-false-history-note-persists
title: "Conclusively refuted board history notes are never corrected in place, so every later iteration re-litigates them"
status: OPEN
severity: Medium
workflow: fix
category: ticket-quality
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Conclusively refuted board history notes are never corrected in place, so every later iteration re-litigates them

F-game-features-plugin-management note #3 asserts an unverified environment fact ('this host actually loads engine built-in GF plugins') that a sibling ticket's note #2 already empirically refuted, yet the false note still stands. In this iteration THREE agents (historian, adversarial, correctness lenses) each independently re-derived the disproof against UE engine source — the load-bearing validity question for the ticket — and at least one prior iteration did the same: 4+ re-derivations across 2 iterations. The lead's BOARD-REPAIR rule covers only phantom IN-REVIEW fix claims, so no agent is authorized to correct the note and the tax recurs on every GF-family analysis.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a5f826bff2fdb9579.jsonl
- quote: "Board-history factual dispute I resolved (the load-bearing point). The two GF tickets contradict each other on whether the fixture is even needed."
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace: "a phantom IN-REVIEW is invisible to the picker, so its real defect is never re-fixed and every future sibling analysis re-pays the disproof." -> with: "a phantom IN-REVIEW is invisible to the picker, so its real defect is never re-fixed and every future sibling analysis re-pays the disproof. The same repair duty applies to a conclusively DISPROVED factual claim standing in ANOTHER ticket's history (e.g. a test-phase note asserting an environment/behavior fact your lenses refuted with engine-source or runtime evidence, and no fresh claimedBy lease on that ticket): append a one-line `#{N}-history-correction` note to THAT ticket citing the disproving evidence (append-only — do not delete the original line) and include its file in the STEP 2b -Files list — an uncorrected false note taxes every future analysis of the family with the same re-litigation."

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
