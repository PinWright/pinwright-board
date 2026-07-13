---
id: H-audit-journal-shape-mismatch-fix-runs
title: "Audit skill's journal jq recipe assumes the test-workflow shape and errors on fix-run journals"
status: OPEN
severity: Low
workflow: fix
category: harness-process
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/workflow-log-audit/workflow-log-audit.workflow.js
---

# Audit skill's journal jq recipe assumes the test-workflow shape and errors on fix-run journals

The workflow-log-audit file-to-board prompt tells auditors to start with jq -rc 'select(.type=="result").result | {outcome, friction, plan_divergence}' — but mcp-fix-workflow journals carry heterogeneous per-role result payloads (role-specific objects or raw prose strings from the schema-less Analyze agents), so the recipe emitted 36x 'Cannot index string with string' this audit session and null fields for object results. Every fix-partition auditor burns ~2-3 exploratory calls reverse-engineering the journal shape.

## Evidence
- run: wf_510c6c2c-eb5
- agent: journal.jsonl
- quote: "jq: error (at journal.jsonl:N): Cannot index string with string"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/workflow-log-audit/workflow-log-audit.workflow.js
replace (anchor verified at line 228): "Example: jq -rc 'select(.type==\"result\").result | {outcome, friction, plan_divergence}' journal.jsonl" -> with: "Example: jq -rc 'select(.type==\"result\") | .result | if type==\"object\" then {outcome, friction, plan_divergence} else {prose: (tostring | .[0:160])} end' journal.jsonl  (NOTE: only mcp-test-workflow journals carry {outcome, friction, plan_divergence}; mcp-fix-workflow journals carry per-role payloads — role-specific objects or raw prose strings from the schema-less Analyze agents — so use the shape-tolerant form and pull role context from the agent transcripts instead)"

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
