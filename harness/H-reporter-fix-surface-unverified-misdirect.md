---
id: H-reporter-fix-surface-unverified-misdirect
title: "Test-workflow reporter prescribes a concrete but wrong fix surface (docs/overlay edit for a registry-generated wiki section) plus wrong domain vocabulary, forcing full REWORDs downstream"
status: OPEN
severity: Medium
workflow: test
category: ticket-quality
knob: prompt
encounters: 1
lastSeen: 2026-07-13T10:15:00Z
filedBy: audit@fuzz2
editTargets:
  - .polyskill/skills/mcp-test-workflow/mcp-test-workflow.workflow.js
---

# Test-workflow reporter prescribes a concrete but wrong fix surface (docs/overlay edit for a registry-generated wiki section) plus wrong domain vocabulary, forcing full REWORDs downstream

The audited ticket's '## Fix (docs / overlay — downstream)' header directed the implementer to edit Docs/wiki-src for the auto-generated `## Methods` index — which is sourced from the handler registry (REGISTER_RPC_HANDLER Category arg) and cannot be changed by any overlay edit; it also under-scoped the defect (~11 vs the real 16 methods) and asked for 'gameplay tags' vocabulary on a method reading AActor::Tags (a different UE system; adversarial lens rated it WEAK-to-HARMFUL). All three lenses + the lead paid to disprove the framing; disposition became a full REWORD. The historian cites two earlier REWORDs for the same reporter over-statement pattern — 3 tickets evidenced, pattern-level producer noise. The tell is MCP-visible: every served page begins '<!-- GENERATED from docs/wiki-src overlays + handler registry at editor launch -->'.

## Evidence
- run: wf_510c6c2c-eb5
- agent: agent-acf7dd0576c1ce936.jsonl
- quote: "it wrongly labeled a source Category change as a Docs/wiki-src overlay edit (no overlay can touch the registry-sourced index), under-scoped it (16 not ~11)"
- report: C:\Users\Alexander\.claude\projects\X--src-unreal-EAContentExamples57-fuzz2\workflow-audit\reports\2026-07-13T10-15-00Z.md

## Proposed edit
file: .polyskill/skills/mcp-test-workflow/mcp-test-workflow.workflow.js
In the struggle-auditor filing rules, after the sentence beginning "- Genuinely new -> write board/<id>.md with full frontmatter" (auditor block near line 589), append a new bullet: "- PROPOSED-FIX SURFACE DISCIPLINE: your proposed fix is a HYPOTHESIS. When the friction is on a served wiki page, check its header: pages marked GENERATED from the handler registry have registry-sourced sections (the auto `## Methods` index, the `Namespace:` line) that NO Docs/wiki-src overlay edit can change — for those, name the handler registration (the REGISTER_RPC_HANDLER Category arg) as the probable surface and mark it unverified instead of prescribing an overlay edit. Never assert engine-domain vocabulary (e.g. 'gameplay tags') that the method's observed output/summary does not actually use — describe the observed field names." Mirror the same bullet in the judge's 'Genuinely new' filing rules near line 135.

## History
- `#1-filed-by-audit` filed by audit@fuzz2 (run wf_510c6c2c-eb5)
