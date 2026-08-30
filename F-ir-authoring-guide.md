---
id: F-ir-authoring-guide
title: "Write docs/ir-authoring.md — IrCore contract guide for new IR authors"
status: DONE
severity: Low
category: feature
tags: [docs, ircore, bpir, mgir, agir, niagara, behavior-tree, metasound, sound-cue]
---

# IR-Authoring Guide (docs/ir-authoring.md)

Four new IRs (Sound Cue, Behavior Tree, MetaSound, Niagara) are in the queue.
Today, every new IR author has to re-derive the shared text/grammar/diagnostic
contract by reading BPIR, MGIR, and AGIR sources side-by-side. The architecture
audit flagged this as the highest-ROI documentation gap: ~4 hours of writing
makes the contract explicit up front for every future IR.

Pure-docs task. No code risk. Recommended-before (soft prereq) for the four
new-IR tickets, not a strict blocker — those tickets can still land without
it, they just lose the consistency lift.

## Scope: what the guide must cover

1. **Ephemeral-IR principle (TOP OF GUIDE, most important).** IR text
   (BPIR, MGIR, AGIR, SCIR, BTIR, MSIR, NIR, any future IR) is EPHEMERAL.
   It exists ONLY as (a) asset-dump sidecars fully rewritten on every
   re-dump, or (b) compile-input strings immediately converted to assets.
   No user ever stores raw IR text; agents read fresh dumps, humans rarely
   touch the text directly. Therefore: NEVER write backwards-compatibility
   code, dual-accept parser windows, deprecation cycles, or migration
   utilities. Grammar can change at any time — just update emitter and
   parser together. Tests don't need version compatibility; goldens get
   regenerated alongside the change. This must be the first section, not
   buried — new IR authors will otherwise add backcompat code that becomes
   permanent dead weight.
2. **IrCore contract.** Inventory and usage of `FIrTextUtils`:
   `Quote`, `EscapeString`, `FormatNameToken`, `FormatPositionSuffix`,
   `FormatFieldList`, `IsSafeReflectedProperty`, `StripTrailingComment`,
   `SmartSplit`, `FindMatchingChar`, `TryExtractPosition`. One-line summary
   plus a "use when" cue for each.
3. **Mandatory grammar shape (post-harmonization).** Backtick name tokens,
   `@(x, y)` position suffix, `:` field separator, `#` line comments,
   double-quoted strings with the standard escape set. Show a minimal
   illustrative snippet.
4. **`IIrGrammar` implementation pattern.** Keyword map + opcode enum;
   reference the existing BPIR/MGIR/AGIR grammar classes.
5. **`IrTypeSpec` usage.** When a pin/port carries a structured type and
   when a bare name is enough.
6. **Diagnostic emission via `FIrCompileDiagnostic`.** Severity levels,
   source-range attachment, the "lossy-roundtrip vs hard-error" distinction.
7. **File layout.** Where `Decompiler.cpp`, `TextEmitter.cpp`, `Grammar.cpp`,
   and per-node handler files live for an IR; how the directory mirrors the
   BPIR/MGIR/AGIR convention.
8. **Asset-dump sidecar integration.** How the IR text becomes the `<ir>.txt`
   sidecar; what `meta.json` aspect to declare; behavior on stub/skip.
8a. **Dual surface (dump sidecar + MCP RPC) is required for every IR; both
    call one shared builder function.** Every IR must ship BOTH a dump
    sidecar (`<ir>.txt` registered in `AssetDumpHandler.h` `DumpFileNames`
    and emitted from the matching dispatch branch in `AssetDumpHandler.cpp`)
    AND an MCP RPC (`<namespace>.decompile_<ir>` handler under
    `Private/Handlers/<Domain>/`). Both surfaces MUST call ONE shared
    builder function `BuildXxxIrText(UAsset*) → FIrResult { Text, Warnings,
    bSuccess }`; the sidecar pipeline and the RPC dispatch are thin
    adapters around the same builder. Never ship one without the other;
    never let the two implementations drift. If output needs to vary
    between dump and RPC (e.g. include-subgraphs toggle), route the
    variation through a builder-options struct, not a second
    implementation.
9. **Testing pattern.** Round-trip (BPIR-style full bidirectional) vs golden
   (decompile snapshot) vs decompile-only (MGIR/AGIR-style emit-but-don't-parse).
   Which to pick for which IR maturity.
10. **Cross-refs to existing examples.** BPIR for full bidirectional, MGIR
    for decompile-shape with graph-edit RPCs, AGIR for decompile-only on top
    of an immutable native graph.

## Cross-link targets that already exist

- `docs/bpir-language-reference.md` — BPIR grammar/keywords
- `docs/bpir-examples.md` — BPIR snippets by node kind
- `docs/bpir-compiler-internals.md` — BPIR compile pipeline
- `docs/wiki/material.mgir.md` — MGIR wiki overlay
- AGIR: no dedicated wiki page yet; cross-link to
  `Source/PinWright/Private/AGIR/` and to
  `docs/wiki/animation.authoring.md` for the surface RPCs.

## Out of scope

- Per-IR grammar specifics (those live in each IR's own reference doc).
- Tutorial-style walkthroughs. This is a contract reference, not a how-to.

**Fix:** Single new file at `docs/ir-authoring.md`. ~4 hours.

## History
- `#1-initial-filing` `OPEN` reporter — Filed as soft prereq for the four new-IR tickets (Sound Cue, Behavior Tree, MetaSound, Niagara). Audit found the IrCore contract is invisible to new IR authors today — each one re-derives it from BPIR/MGIR/AGIR source. Pure-docs scope, no code risk, severity Low.
- `#2-ephemeral-ir-principle` `OPEN` reporter 2026-05-13 — Adding the **ephemeral-IR invariant** as a mandatory, top-of-guide section. (1) The invariant is now a required content section of the guide, not optional context. (2) Specific text the guide must include: IR text (BPIR, MGIR, AGIR, SCIR, BTIR, MSIR, NIR, any future IR) is EPHEMERAL; exists only as asset-dump sidecars fully rewritten on every re-dump, or as compile-input strings immediately converted to assets; no user ever stores raw IR text; agents read fresh dumps, humans rarely touch the text directly; therefore NEVER write backwards-compatibility code, dual-accept parser windows, deprecation cycles, or migration utilities; grammar can change at any time — just update emitter and parser together; tests don't need version compatibility, goldens get regenerated alongside the change. (3) This is the highest-leverage content in the entire doc — must be called out at the very top, not buried. Without it, new IR authors will add backcompat code that becomes permanent dead weight. Body's "Content the guide must cover" list updated: ephemeral-IR principle inserted as item #1, subsequent items renumbered 2–10.
- `#3-dual-surface-mandate` `OPEN` reporter 2026-05-13 — Adding the **dual-surface invariant** as a mandatory content section of the guide. The guide must explicitly require, for every new IR: (a) an asset-dump sidecar `<ir>.txt` registered in `AssetDumpHandler.h` `DumpFileNames` and emitted from the matching dispatch branch in `AssetDumpHandler.cpp`; (b) an MCP RPC `<namespace>.decompile_<ir>` handler under `Private/Handlers/<Domain>/`; (c) ONE shared builder function `BuildXxxIrText(UAsset*) → FIrResult { Text, Warnings, bSuccess }` that both surfaces call — the sidecar pipeline and the RPC dispatch are thin adapters around the same builder. Never ship one surface without the other; never let the two implementations drift. Output variation (e.g. include-subgraphs toggle) routes through a builder-options struct, not a parallel implementation. Added to "Content the guide must cover" as item 8a (kept adjacent to item 8 "Asset-dump sidecar integration" rather than renumbering downstream items, since the dual-surface mandate is a directly-related elaboration on the sidecar integration rule). Motivation: the five in-flight new-IR tickets (SCIR, BTIR, MSIR, NIR, PCGIR) were filed with both surfaces but without an explicit single-builder mandate — without this rule in the guide, future IR authors will write the dump and RPC paths as two separate implementations that drift over time.
- `#4-ir-authoring-guide` `IN-REVIEW` developer - Added `Docs/ir-authoring.md` as the maintainer guide for new IR authors, documenting the ephemeral-IR invariant, IrCore text utilities, the harmonized grammar target, IIrGrammar/type-spec/diagnostic patterns, file layout, asset-dump sidecar plus MCP dual-surface builder rule, testing patterns, and BPIR/MGIR/AGIR cross-links. Updated `Docs/index.md`, `Docs/tags.md`, and the architecture cross-link for discoverability. No runtime test added because this is docs-only.
- `#5-verify-docs-guide` `DONE` tester — Verified: read `Docs/ir-authoring.md` plus `Docs/index.md`, `Docs/tags.md`, and `Docs/arch.md`; observed the guide covers the required ephemeral-IR invariant, IrCore utilities, grammar, IIrGrammar/type-spec/diagnostic patterns, file layout, asset-dump plus MCP dual-surface builder rule, testing patterns, and BPIR/MGIR/AGIR cross-links, with discoverability links present.
- `#6-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
