---
id: H-lens-live-mcp-probe-editor-down
title: "Lens prompts do not state the editor is stopped during Analyze, so lenses burn calls attempting live MCP confirmation"
status: OPEN
severity: Low
workflow: fix
category: editor-lifecycle
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Lens prompts do not state the editor is stopped during Analyze, so lenses burn calls attempting live MCP confirmation

The adversarial lens spent a ToolSearch (loading mcp__pinwright__call), a gateway-port + editor-process check, and the MCP call itself before discovering the editor was down, then hedged its finding ('Editor was down, so I could not confirm via a live game_features.list call'). In the fix workflow the editor is stopped BY CONSTRUCTION during Analyze (Baseline kills it; nothing relaunches it until the implement step's test run), but no lens/red-test prompt says so — a thorough lens reasonably wastes the calls every time it wants runtime evidence.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a0883d016e586769a.jsonl
- quote: "Editor not reachable at http://127.0.0.1:23000/mcp (connection refused) - it is not running."
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace (anchor verified at line 244): "Verify claims against the plugin source under ${pluginClone}\\Source (cite file:line) and search the board at ${boardPath} with ripgrep (NOT qmd)." -> with: "Verify claims against the plugin source under ${pluginClone}\\Source (cite file:line) and search the board at ${boardPath} with ripgrep (NOT qmd). The host editor is STOPPED during this phase (Baseline killed it; nothing relaunches it until the implement step) — do NOT spend calls attempting live MCP confirmation; when a claim needs runtime evidence, say so in your findings and verify what you can from plugin + engine source instead."

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
