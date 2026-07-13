---
id: H-preprobe-mechanism-claim-propagation
title: "Pre-probe asserts helper-function behavior from its NAME without reading its body; the false mechanism claim is injected verbatim into all downstream lens prompts"
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

# Pre-probe asserts helper-function behavior from its NAME without reading its body; the false mechanism claim is injected verbatim into all downstream lens prompts

The pre-probe claimed foliage.create_procedural 'writes real spawner + FoliageType assets to disk (McpSafeAssetSave(FT))' without reading McpSafeAssetSave — a misleadingly-named 12-line mark-dirty-only helper. probeEvidence is spliced verbatim into every lens + red-test prompt with a caveat that actively discourages re-derivation, so 2 of 4 downstream agents (adversarial, historian) repeated the claim as fact and the historian's valid vote was partly premised on it; only the correctness lens and red-test writer read the body and refuted it. Had they also skipped the check, the REWORD could have shipped the wrong mechanism and an inapplicable fix. Once observed; the propagation channel exists on every iteration.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-a7b42cef1a7d24bfd.jsonl
- quote: "PRE-PROBE'S CENTRAL CLAIM IS FALSE ... McpSafeAssetSave (Utils/AssetUtils.cpp:220-232) is a MARK-DIRTY-ONLY no-op"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-fix-workflow/mcp-fix-workflow.workflow.js
replace: "Read the ticket, find the exact handler/method/code path it cites, and inspect it. Do NOT edit anything, do NOT run the MCP, do NOT compile." -> with: "Read the ticket, find the exact handler/method/code path it cites, and inspect it. MECHANISM DISCIPLINE: before your evidence asserts WHAT any helper/function DOES (saves, writes to disk, deletes, locks...), read that function's BODY — NEVER infer behavior from its name (e.g. a 'Save' helper may be mark-dirty-only). Your evidence is injected VERBATIM into every downstream lens prompt, so an unverified mechanism claim propagates as fact. Do NOT edit anything, do NOT run the MCP, do NOT compile." Optionally strengthen the lens-prompt caveat to require spot-checking the body of any helper the citation credits with the defect's key action.

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
