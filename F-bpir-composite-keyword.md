---
id: F-bpir-composite-keyword
title: "Inline `K2Node_Composite` at decompile (no BPIR keyword)"
status: DONE
severity: Medium
category: feature
tags: [bpir, decompiler, composite, inline]
---

# Inline composites at decompile

## Why

`UK2Node_Composite` is editor-only visual grouping. It has no `ExpandNode`
override; the engine compiler dissolves every composite into its inner
nodes during `FKismetCompilerContext::ExpandTunnelsAndMacros`, which
delegates to `UEdGraphSchema_K2::CollapseGatewayNode`. By the time any
node-handler scheduling runs, no `K2Node_Composite` survives.

Round-trip identity for composites is not a real requirement — visual
grouping is editor ergonomics, not a contract anyone depends on. So the
keyword approach originally proposed here (mirror `macro` with a new
`composite Name(...)` opcode + tokenizer + parser + compiler dispatch)
adds BPIR surface area to round-trip a node class the engine itself
treats as transient.

Walking the composite's bound graph at decompile time and emitting the
inner instructions in the parent BPIR stream side-steps the keyword
entirely, leaves the BPIR surface smaller, and matches the existing
inline-at-decompile pattern used for knots, pure `VarGet`s, and `Self`.

## Approach (decompile-time inline)

Add an `InlineCompositesInPlace(UEdGraph*)` pre-pass in
`BpirDecompiler.cpp`. For every `UK2Node_Composite` in the (cloned)
graph, recurse into `BoundGraph` first (depth-first, so nested
composites flatten before the enclosing one), then call
`GetDefault<UEdGraphSchema_K2>()->CollapseGatewayNode(Composite,
Composite->InputSinkNode, Composite->OutputSourceNode,
/*CompilerContext=*/nullptr, /*OutExpandedNodes=*/nullptr)`. Move the
surviving inner nodes into the parent graph via
`FEdGraphUtilities::MergeChildrenGraphsIn`; remove the composite,
`InputSinkNode`, and `OutputSourceNode`. A `Hops > 1024` guard mirrors
the knot loop guard at `BpirDecompiler.cpp:80`.

`UEdGraphSchema_K2::CollapseGatewayNode` is verified null-safe with both
trailing args null — header default is `nullptr` at
`EdGraphSchema_K2.h:1213`, function never dereferences the pointer
(every use forwards into already-null-guarded callees).

Mutation safety: the inline pass operates on a clone of the input
graph (`FEdGraphUtilities::CloneGraph` with
`bCloningForCompile=true`), never on the live editor `UEdGraph`.
Decompile stays read-only from the user's perspective.

The composite branch in `WalkExecChain` (`BpirDecompiler.cpp:1265–1287`)
becomes dead code once the inline pass runs first and is deleted.
`EmitCompositeNode` (`BpirTextEmitter.cpp:1702–1741`, the transitional
`call K2Node_Composite(__compositeName: $Name, ...)` form added by
`B-bpir-composite-inline-name-lost`) is deleted outright — composites
no longer appear in BPIR text in any form. No deprecation stub
(plugin-scope no-legacy policy).

No region-marker comments / no `# region composite Name` annotations;
visual round-trip is not a goal.

## Tests

- Rewrote `TestBpirCompositeNamePreserved.cpp`: keeps the
  `BuildMidGraphCompositeFixture` BP fixture; assertions now check
  decompiled BPIR contains the inner instruction, does NOT contain
  `K2Node_Composite` or `composite ` substrings, recompiles cleanly,
  and the recompiled graph has zero `UK2Node_Composite` nodes.
- New `TestBpirCompositeInlineNested.cpp`: two-level nested composite
  fixture; asserts decompile is flat at every level and recompile yields
  no `UK2Node_Composite` nodes with both inner-most instructions
  preserved in source order.

## Spawned ticket

The composite-inline research surfaced an independent multi-input-exec
gap: BPIR has no syntax to wire an upstream exec output to a non-first
input exec pin on the target. For composites this is theoretical (zero
PDS composites with 2+ exec inputs). For runtime-stateful macros (Gate
Enter/Open/Close/Toggle, MultiGate Enter/Reset, DoOnce Start/Reset, DoN
Enter/Reset) it's an active corruption affecting real PDS BPs
(`BP_Train`, `BP_RobotHend`, `AC_DoorAction`). Filed as
`F-bpir-multi-input-exec-targets` (severity High, OPEN). Not implemented
in this sprint.

## Cross-references

- `B-bpir-composite-inline-name-lost` — the transitional `call
  K2Node_Composite(__compositeName: $Name)` shape this ticket
  structurally obsoletes. `EmitCompositeNode` is deleted; that
  ticket's fix is superseded but not reverted.
- `F-bpir-multi-input-exec-targets` — spawn ticket for the
  multi-input-exec gap discovered during research.

## History
- `#1-filed-from-decompose` `OPEN` reporter — Filed during sprint implementation of `B-bpir-composite-inline-name-lost`. The original ticket's proposed-fix paragraph requested round-trip-compile support (new opcode + parser + tokenizer + compiler dispatch); that's feature-sized. Sprint scope was reshaped to decompile-only name preservation; round-trip lives here.
- `#2-pivot-to-inline` `IN-REVIEW` developer — Reshaped from "add composite keyword" to "inline composites at decompile" after multi-agent investigation established that K2Node_Composite is editor-only (no ExpandNode override; engine dissolves via Schema->CollapseGatewayNode pre-pass) and round-trip identity is not a real requirement. Implementation: InlineCompositesInPlace pre-pass in BpirDecompiler.cpp before walker construction, operating on a graph clone (FEdGraphUtilities::CloneGraph) to keep decompile read-only. EmitCompositeNode deleted. Tests rewritten/added: TestBpirCompositeNamePreserved (rewrite — asserts inlined body + no composite identity), TestBpirCompositeInlineNested (new — recursive composites). Multi-input-exec on macros and composites filed separately as F-bpir-multi-input-exec-targets.
- `#3-verified-composite-inlined` `DONE` tester — Verified: ran `blueprint.decompile` on `/App/App/UI/LobbyAndMenu/Popups/W_SaveTrack`. BPIR contains zero `K2Node_Composite` occurrences and zero `composite ` keyword occurrences. Former mid-graph composites in OnLoggedIn_Event and Save/Publish button bodies appear as inlined labeled blocks in the parent flow.
