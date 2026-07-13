---
id: H-board-repair-scan-shell-loop-timeout
title: "Startup board-reconcile agent wastes a full 2-minute tool timeout on a per-file shell loop over the 1147-ticket board"
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

# Startup board-reconcile agent wastes a full 2-minute tool timeout on a per-file shell loop over the 1147-ticket board

The resume-reconcile prompt says only 'Scan every <board>\*.md' with no scan mechanics. The agent ran an ls+fetch producing an 80.1KB persisted output, then a Git-Bash per-file grep loop (3+ subprocesses per ticket) that hit the 120s Bash timeout (exit 143) on the 1147-ticket board, then answered the whole question in seconds with one ripgrep pass. The step runs at EVERY workflow start on a board that keeps growing, so the naive-loop trap re-fires whenever an agent picks a per-file loop.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-aec72a2a713948a0b.jsonl
- quote: "RESULT: Exit code 143 / Command timed out after 2m 0s"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace (anchor verified at line 581): "Scan every ${boardPath}\\*.md (exclude README.md and agent-conventions.md)." -> with: "Scan every ${boardPath}\\*.md (exclude README.md and agent-conventions.md). SCAN MECHANICS: the board holds 1000+ tickets — gather the three frontmatter fields with ONE ripgrep pass (e.g. \`rg -n \"^(status|claimedBy|claimedAt):\" --glob \"*.md\"\` via your Grep tool) and post-filter the hits; NEVER a per-file shell loop spawning grep/sed per ticket (it exceeds the 2-minute tool timeout on this board), and do NOT ls the whole board into context." (keep template-literal backtick escaping consistent with the file)

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
