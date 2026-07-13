---
id: H-pick-double-board-query-toctou
title: "Pick prompt's primary example re-runs board-query.ps1 for the random pick — two full board scans per pick and a TOCTOU window between displayed and sampled candidate sets"
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

# Pick prompt's primary example re-runs board-query.ps1 for the random pick — two full board scans per pick and a TOCTOU window between displayed and sampled candidate sets

Merged (pick-double-board-query-toctou + pick-eligibility-query-double-run; seen by 2 readers). Step 1 captures the eligibility helper's stdout, but step 3's PRIMARY inline example re-invokes the helper ($pick = (& pwsh ... board-query.ps1 ... | ConvertFrom-Json | Get-Random)) — contradicting its own 'WITHOUT re-typing the array' rule — and the fallback ('store step-1's stdout in a variable') is impossible because PowerShell tool state does not persist between calls. The agent dutifully ran the full board scan twice; the second run re-reads the live shared board, so the candidate set actually sampled can differ from the one displayed and parsed (another host can claim/file in between), silently invalidating the step-2 empty-check. Structural on every pick.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a42a8fb248e107e7a.jsonl
- quote: "second CALL runs `$pick = (& pwsh -NoProfile -File ...board-query.ps1... | ConvertFrom-Json | Get-Random)` — the helper executes twice"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace (anchor verified at line 229): `3. Pick ONE candidate at random WITHOUT re-typing the array — pipe the helper's stdout in one call: \`$pick = (& pwsh -NoProfile -File "${SKILL_DIR}\\board-query.ps1" -Board "${boardPath}" -HostId "${hostId}" -ExcludeCsv "${excludeCsv || ''}" -TtlHours ${claimTtlHours} | ConvertFrom-Json | Get-Random); $pick | ConvertTo-Json -Compress\` (or store step-1's stdout in a variable and pipe that).` -> with: `3. Pick ONE candidate at random WITHOUT re-running the helper — PowerShell tool state does NOT persist between calls, so run steps 1+3 as ONE call: \`$json = & pwsh -NoProfile -File "${SKILL_DIR}\\board-query.ps1" -Board "${boardPath}" -HostId "${hostId}" -ExcludeCsv "${excludeCsv || ''}" -TtlHours ${claimTtlHours}; $json; $pick = $json | ConvertFrom-Json | Get-Random; $pick | ConvertTo-Json -Compress\` (a second helper run re-scans the live shared board and can return a DIFFERENT candidate set than the one you parsed in step 2).`

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
