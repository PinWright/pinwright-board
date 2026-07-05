---
id: F-pcg-authoring-parity
title: "pcg: audit vs UE 5.8 PCGToolset (31 tools) and close authoring gaps"
status: OPEN
severity: Medium
category: feature
tags: [pcg, authoring, audit, parity-ue58]
---

# pcg: audit vs UE 5.8 PCGToolset (31 tools) and close authoring gaps

PinWright's `pcg` namespace has 10 methods plus PCGIR decompile. The UE 5.8.0 release added a new PCGToolset with 31 tools (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\PCGToolset\`), a 3x surface that likely covers graph authoring depth we lack. Exact deltas are unaudited; this is an audit-first ticket.

Proposed scope:
1. Audit: enumerate Epic's 31 tools against our 10; produce the gap table (node/edge CRUD depth, subgraphs, params/overrides, attribute operations, generation trigger + result readback, debugging).
2. Close the gaps that matter for agent graph authoring; likely candidates based on Epic's tool names: generation trigger with completion readback, per-node settings editing, graph parameter CRUD.
3. Skip anything PCGIR decompile already serves better as text.

Acceptance: gap table attached to this ticket via IN-REVIEW note; new RPCs let an agent build a scatter-on-surface graph from empty, generate, and read back point counts without python.execute.

## History
- `#1-pcg-3x-gap` `OPEN` reporter — UE 5.8.0 shipped PCGToolset with 31 tools vs our 10 + PCGIR; deltas unaudited. Audit first, then close authoring-relevant gaps (generation trigger/readback, node settings, graph params).
