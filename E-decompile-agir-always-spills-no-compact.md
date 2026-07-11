---
id: E-decompile-agir-always-spills-no-compact
title: "anim.decompile_agir output for a real Animation Blueprint (~10.7KB) always overflows the 10000-char inline display limit and spills to disk, with no compact/summary mode — every decompile forces a follow-up file Read"
status: OPEN
severity: Low
category: ergonomic
tags: [animation, anim, decompile_agir, agir, response-size, oversized-readback, compact, response-spill, docs]
encounters: 1
lastSeen: 2026-07-11T04:58:19+03:00
---

# anim.decompile_agir never returns inline for a standard ABP — its output sits just above the 10000-char limit, so every call mandates an extra Read

`anim.decompile_agir` captures an Animation Blueprint's AGIR text (the transfer
format used to inspect / round-trip an anim graph). For any real locomotion-class
ABP the AGIR is ~10.7KB — just above the **10000-char inline display threshold** —
so the method is **never** returned inline in practice: it always trips
`outputTooLong` and spills the full payload to an `HttpResponses/*.json` file that
the caller must open with a separate `Read` before it can inspect the AGIR.

This bites hardest in the exact workflow AGIR exists for: an author capturing a
source graph, compiling it into a variant, then decompiling both to confirm
fidelity does **three** decompiles, and every one spills. A fidelity diff that
should be two inline reads becomes opening two (or three) response files by hand.

There is no compact / field-select / summary mode to keep the common
"does the graph carry the expected node families?" check inline.

## Evidence
AGIR-transfer task (namespace `anim`): all three `anim.decompile_agir` calls
overflowed —
- source `ABP_Manny` decompile → `{"outputTooLong":true,"message":"Response exceeds display limit (10720 chars, threshold 10000); full payload written to ...HttpResponses/...json"}`
- variant post-transfer decompile → 10704 chars vs 10000
- `ABP_Manny` re-decompile (confirm unchanged) → 10720 chars vs 10000

Each therefore required an extra file `Read` to see the AGIR — 3 avoidable
round-trips in one task.

## Fix
Two options (same family as the other per-method spill/compact tickets —
`E-describe-sequence-no-compact-mode`, `E-search-metasound-nodes-no-compact-mode`,
`E-blueprint-list-no-projection-spills`):
- **Inline threshold** — raise the inline display limit for `anim.decompile_agir`
  specifically, since a real ABP's AGIR is inherently ~10.7KB and the whole
  document is the payload (there is no per-row array to trim); or
- **Compact/summary mode** — an optional flag that returns an inline structural
  summary (state machines / states / blend-space players / top-level output
  binding present) with the full AGIR text still spilled, so a fidelity check
  doesn't require opening two response files by hand.
Also note on `docs/wiki-src/anim.md` that a full-ABP decompile spills by default.

severity rationale: impact=Low (pure response-spill that only forces a Read; recovery is obvious) × reach=every-session-for-this-workflow (any AGIR capture/round-trip on a real ABP hits it, and it's structurally 100% of calls) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an AGIR-transfer task (namespace `anim`, outcome `tool_bug` on the separate `anim.compile_agir` cached-pose corruption, `B-agir-cached-pose-forward-ref-corruption`). Distinct PROCESS angle surfaced by the CallAnalyzer: all three `anim.decompile_agir` calls (source `ABP_Manny`, the variant, and the re-decompile of `ABP_Manny`) returned `outputTooLong` at 10720 / 10704 / 10720 chars against the 10000-char threshold, spilling to `HttpResponses/*.json` and forcing an extra file `Read` each time — the method is structurally never inline for a real ABP. No compact/summary mode exists to keep the "expected node families present?" check inline. Same per-method oversized-readback family as `E-describe-sequence-no-compact-mode` et al. Overlay page = `docs/wiki-src/anim.md`. Severity Low (response spill; extra Read only).
