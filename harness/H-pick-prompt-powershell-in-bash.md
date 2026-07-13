---
id: H-pick-prompt-powershell-in-bash
title: "Pick prompt embeds PowerShell one-liners without the tool-discipline warning, causing a Bash syntax-error retry"
status: OPEN
severity: Low
workflow: fix
category: prompt-instruction
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
---

# Pick prompt embeds PowerShell one-liners without the tool-discipline warning, causing a Bash syntax-error retry

The pick agent ran the prompt's suggested `$pick = (& pwsh ...)` one-liner in the Bash tool and got 'syntax error near unexpected token' (exit 2), then retried correctly in PowerShell. Every other fix-workflow prompt (lenses, red-test, lead, reviewers) carries an explicit TOOL DISCIPLINE paragraph; pickPrompt only says 'Run PowerShell for timestamps + the claim write', which the agent read as advisory. One wasted call per occurrence, at every pick that misreads it.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a88393e4ff4eb29e2.jsonl
- quote: "/usr/bin/bash: eval: line 1: syntax error near unexpected token `('"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace (anchor verified at line 215): "select semi-randomly and claim so the hosts don't collide. Run PowerShell for timestamps + the claim write." -> with: "select semi-randomly and claim so the hosts don't collide. Run PowerShell for timestamps + the claim write — the one-liners below are PowerShell syntax: run them in the PowerShell tool, NOT the Bash tool (Bash errors on `$pick = (& pwsh ...)`)."

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
