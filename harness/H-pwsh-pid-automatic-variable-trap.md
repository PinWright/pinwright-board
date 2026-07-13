---
id: H-pwsh-pid-automatic-variable-trap
title: "Wait-Process mechanics invite the read-only $pid PowerShell automatic-variable trap — waits on the wrong process with a misleading 'exited' message"
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

# Wait-Process mechanics invite the read-only $pid PowerShell automatic-variable trap — waits on the wrong process with a misleading 'exited' message

The STEP-4 run-your-test mechanics say 'Wait-Process -Id <the PID> -Timeout 570' without warning that $pid is a read-only PowerShell automatic variable (the shell's own PID). The lead read the editor PID back into $pid: assignment silently failed ('Cannot overwrite variable PID'), Wait-Process errored 'cannot wait on itself', yet the compound command still printed 'Editor process 34828 exited'. A less careful agent would parse a still-running editor's log and report a phantom test result; here it cost one diagnose+retry cycle. The identical mechanics text appears at lines 293 and 339 (plus 428), so any agent naming its variable $pid re-hits it in every editor-run step.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-acf7dd0576c1ce936.jsonl
- quote: "Cannot overwrite variable PID because it is read-only or constant ... it cannot wait on itself"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
In BOTH occurrences (anchors verified at lines 293 and 339; also consider line 428): replace: "run \`Wait-Process -Id <the PID> -Timeout 570\`" -> with: "run \`Wait-Process -Id <the PID> -Timeout 570\` (hold the PID in a variable like \`$edPid\` — NEVER \`$pid\`, a read-only PowerShell automatic variable holding the shell's own PID: assigning it silently fails and Wait-Process then errors 'cannot wait on itself' while the surrounding command may still print a false 'exited')"

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
