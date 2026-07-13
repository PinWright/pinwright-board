---
id: H-board-edit-leaks-toolcall-xml
title: "Lead's board-ticket Write leaked literal tool-call closing tags into file content again (EOF-strip guard caught it; mid-file leaks would still ship)"
status: OPEN
severity: Low
workflow: fix
category: harness-process
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Lead's board-ticket Write leaked literal tool-call closing tags into file content again (EOF-strip guard caught it; mid-file leaks would still ship)

Recurrence of the known board-edit XML leak: the lead's full-file Write of the reworded ticket ended its content parameter with the literal closing tags of the tool-call markup (content + invoke closers, verified in raw jsonl tool_use input). board-commit.ps1's trailing-tag strip worked — both committed versions (87eff00, a5da69f) are clean. Residual exposure: the on-disk working ticket carries the junk between Write and commit (parallel hosts can read it), and the strip regex is end-of-file-anchored so a mid-file leak (e.g. before an appended history line) would ship to every host. The lead prompt's STRING FIELD HYGIENE warning covers structured-output string fields but says nothing about board file Writes. 4 tickets were corrupted this way on 2026-07-04 before the guard existed.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-acf7dd0576c1ce936.jsonl
- quote: "raw Write tool_use input ... ends with the literal content/invoke closing tags after 'SystemInspectSceneReadersIndex`.'"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
In the lead/implement prompt's STRING FIELD HYGIENE block (line ~308; the same sentence also ends the test-loop block at line 439 — appending to both is harmless), replace: "Paraphrase; the transcript keeps the verbatim text.`;" -> with: "Paraphrase; the transcript keeps the verbatim text. The same discipline applies to board ticket file Writes/Edits: never end (or embed in) the file content with tool-call closing tags (the content/invoke closers) — board-commit.ps1 strips trailing leaks at commit time, but a mid-file leak would ship to every host.`;"

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
