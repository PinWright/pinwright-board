---
id: B-mgir-reroute-displayname-dropped
title: "MGIR parses a named reroute's DisplayName: and then discards it on the declaration branch — the author's name is replaced by the node's own GUID handle on every decompile/compile round trip, and fixing it changes the usage-resolution key"
status: OPEN
severity: Medium
category: bug
tags: [mgir, material, named-reroute, round-trip, data-loss, decompiler, grammar, design-call]
---

# `DisplayName:` is parsed on both branches and read on only one

`MGIRCompiler.cpp:682-715`, the whole `case EMGIROpcode::Reroute` (line range verified by
`git blame -L 682,715` — untouched since `17a331d7`, so the reported `:685-705` is accurate at HEAD):

```cpp
        FString SourceReference;
        FString DisplayName = Instruction.SymbolName;
        for (const FMGIRArg& Arg : Instruction.Args)
        {
            if (Arg.Name.Equals(TEXT("Source"), ESearchCase::IgnoreCase))      { SourceReference = Arg.Value; }
            else if (Arg.Name.Equals(TEXT("DisplayName"), ESearchCase::IgnoreCase)) { DisplayName = TrimQuotes(Arg.Value); }
        }

        if (!SourceReference.IsEmpty())
        {
            FMGIRRerouteDeclarationSpec Spec;
            Spec.Name = Instruction.ResultName;
            Spec.SourceReference = SourceReference;
            Spec.Position = Instruction.Position;
            EmitResult = Emitter.EmitNamedRerouteDeclaration(Spec);   // DisplayName never read
        }
        else
        {
            FMGIRRerouteUsageSpec Spec;
            Spec.DeclarationName = DisplayName;                       // the only consumer
            ...
        }
```

- `:692-695` — the parse, populating `DisplayName` on both branches.
- `:698-705` — the **declaration** branch. `DisplayName` is never read; the local dies at `:715`.
- `:706-713` — the **usage** branch, the only consumer, at `:710`.

The drop is structural, not a missing line: `FMGIRRerouteDeclarationSpec`
(`MGIRExpressionEmitter.h:37-43`) has **no field to carry it**, while its usage twin (`:45-49`) has
`DeclarationName`. A fix must add a field.

What the engine name gets instead — `MGIRExpressionEmitter.cpp:343`:

```cpp
    Declaration->Name = FName(*Spec.Name);
```

`Spec.Name` is the MGIR result symbol, which on a round trip is the GUID handle.

## The loss, traced end to end

`MGIRDecompiler.cpp:429` is the **only** `DisplayName:` producer in the whole plugin source. An
author names a reroute `Roughness`:

1. **decompile** -> `%n1a2b3c4d5e6f = reroute n1a2b3c4d5e6f (Source: %n99..., DisplayName: "Roughness") @(x, y)`
   (`MGIRDecompiler.cpp:429`, `:432-437`) — both name slots come from `State.NameFor`, which returns
   `MGIRHelpers::ExpressionHandleName` = `"n"` + first 12 hex digits of the expression GUID
   (`MGIRDecompiler.cpp:46-67`, `MGIRHelpers.h:24-29`).
2. **compile (Append)** -> `:698` sees `Source:`, takes the declaration branch, discards
   `DisplayName`, and sets `Declaration->Name = FName("n1a2b3c4d5e6f")`.
3. **decompile again** -> `DisplayName: "n1a2b3c4d5e6f"`.

`Roughness` is gone from the asset, and the node in the material editor now carries its own GUID
prefix as its label. This is **not** covered by the documented round-trip carve-out:
`Saved/PinWright/wiki/material.mgir.md:85` scopes that exemption to *"comment boxes, node colors,
node order in the editor list, and pure-layout reroute knots"* — a named reroute's name is logical
data the editor resolves usages by, not visual organisation.

## Why this is a design call and not a one-liner

The token `DisplayName` currently means two different things depending on branch: on a **usage** it
*is* the resolution key, on a **declaration** it is inert.

Today the key space is the MGIR symbol space. `MGIRCompiler.cpp:710` sets
`Spec.DeclarationName = DisplayName`, defaulting to `Instruction.SymbolName` (`:685`) — the head
token, per `MGIRParser.cpp:397-401`. The emitter looks up
`RerouteDeclarations.Find(NormalizeSymbolName(Spec.DeclarationName))`
(`MGIRExpressionEmitter.cpp:357-371`, `MGIR_REROUTE_NOT_FOUND` at `:369-370`), and that map is
populated under the declaration's **MGIR result symbol** (`:349-353`). Binding is then by object
pointer plus GUID (`:391-392`). Symbol uniqueness is guaranteed per document by `ValidateNewSymbol`
(`:159-167`, `MGIR_DUPLICATE_SYMBOL`) and by `MakeUniqueIdentifier` on the decompile side.

Honouring `DisplayName` on the declaration branch forces a choice, and the options are not
equivalent:

1. **Set `Declaration->Name` only, keep keying on `Spec.Name`.** Label and key diverge:
   `%a = reroute a (Source: %x, DisplayName: "Roughness")` yields a node labelled `Roughness` that
   usages must still address as `a`. Non-breaking, but confusing.
2. **Re-key `RerouteDeclarations` on the display name.** Every document that references a
   declaration by its MGIR symbol breaks with `MGIR_REROUTE_NOT_FOUND` — and that is exactly what
   the decompiler emits. Display names also carry **no uniqueness guarantee**: two declarations may
   legitimately share a `Name` in one material, collapsing the map.
3. **Key on both, with a stated precedence and collision rule.** Needs both rules written down.

## A second defect in the same branch

The declaration-vs-usage discriminator at `MGIRCompiler.cpp:698` is `!SourceReference.IsEmpty()`,
but the decompiler **omits `Source:` entirely** when the declaration's input is unconnected
(`MGIRDecompiler.cpp:422-426`, `FormatInputRef` returning empty at `:150-155`). So an **unwired**
named-reroute declaration decompiles to `%nXXXX = reroute nXXXX (DisplayName: "Foo")`, recompiles
down the **usage** branch, and dies with `MGIR_REROUTE_NOT_FOUND` — it does not round-trip at all.
The emitter already supports the unwired declaration
(`MGIRExpressionEmitter.cpp:314-323`, `const bool bHasSource = !Spec.SourceReference.IsEmpty();`),
so that path is dead code the compiler can never reach. `DisplayName:` present + `Source:` absent is
precisely the signal that would disambiguate — another reason the fix is a grammar decision.

## No test coverage at all

`Tests/` has **zero** hits for `NamedReroute`. Every `reroute` hit is a different subsystem: BPIR
knots (`Tests/Bpir/TestDecompiler.cpp:2041,2084,2136,2177`), `blueprint.graph.create_reroute_node`
(`Tests/Blueprint/TestBlueprintHandlers.cpp:1427-1438`), CRIR `URigVMRerouteNode`
(`Tests/Assets/TestCRIRReroute.cpp`), Niagara `UNiagaraNodeReroute`
(`Tests/Niagara/TestNIRGraphUtil.cpp:110-132`). So `EmitNamedRerouteDeclaration`,
`EmitNamedRerouteUsage` and `MGIR_REROUTE_NOT_FOUND` are entirely untested. The failing test to
write first is a `TestMGIRNamedRerouteRoundTrip.cpp` asserting `Declaration->Name` survives
compile -> decompile -> compile, in the graph-asserting style of `TestMGIRExtendExistingHandle.cpp:10-11`.

## The opcode is undocumented, which lowers the compatibility cost

`wiki-src/material.mgir.md` is the only MGIR overlay and there is no MGIR language reference (unlike
`Docs/crir-language-reference.md:170-182`, which does spec CRIR's `reroute`). Grepping the whole
generated wiki for `DisplayName` returns only `audio.authoring.search_metasound_nodes.md:11`,
`sequencer.add_track.md:13,21`, `system.inspect.search_classes.md:11` and
`bpir.examples.foreach-loop.md:14` — nothing MGIR. So no published grammar constrains a re-key, but
equally **no caller can currently discover the feature**.

## History
- `#1-displayname-parsed-then-discarded` `OPEN` reporter — `MGIRCompiler.cpp:682-715` parses `DisplayName:` into a local on both branches (`:692-695`) and then never reads it on the declaration branch (`:698-705`), while the usage branch consumes it at `:710`. Line range verified at HEAD by `git blame -L 682,715` — untouched since `17a331d7` — so the reported `:685-705` is accurate. The drop is structural rather than a missing line: `FMGIRRerouteDeclarationSpec` (`MGIRExpressionEmitter.h:37-43`) has no field to carry it, unlike its usage twin at `:45-49`. The engine-visible name is instead set from the MGIR result symbol — `MGIRExpressionEmitter.cpp:343`, `Declaration->Name = FName(*Spec.Name);` — which on a round trip is the GUID handle. Loss traced end to end: decompile emits `DisplayName: "Roughness"` (`MGIRDecompiler.cpp:429`, the only producer of that token in the whole plugin source, with both name slots coming from `State.NameFor` -> `MGIRHelpers::ExpressionHandleName`, `"n"` + 12 hex digits, `MGIRHelpers.h:24-29`); recompiling discards it and sets the name to `n1a2b3c4d5e6f`; decompiling again emits `DisplayName: "n1a2b3c4d5e6f"`. The author's name is gone from the asset and the editor node is labelled with its own GUID prefix. Not covered by the documented round-trip carve-out, which `Saved/PinWright/wiki/material.mgir.md:85` scopes to comment boxes, node colours, node order and pure-layout reroute knots — a named reroute's name is logical data, not visual organisation.
- `#2-fixing-it-moves-the-resolution-key` `OPEN` reporter — Why this needs a decision before a patch. `DisplayName` currently means two different things by branch: on a usage it IS the resolution key, on a declaration it is inert. The key space today is the MGIR symbol space — `MGIRCompiler.cpp:710` defaults `Spec.DeclarationName` to `Instruction.SymbolName` (`:685`, the head token per `MGIRParser.cpp:397-401`), the emitter resolves via `RerouteDeclarations.Find(NormalizeSymbolName(...))` (`MGIRExpressionEmitter.cpp:357-371`), the map is keyed on the declaration's result symbol (`:349-353`), and binding is by pointer + GUID (`:391-392`), with per-document uniqueness guaranteed by `ValidateNewSymbol` (`:159-167`) and `MakeUniqueIdentifier`. Three non-equivalent options: (a) set `Declaration->Name` only and keep keying on `Spec.Name` — label and key diverge, non-breaking but confusing; (b) re-key on the display name — breaks every document that references a declaration by its MGIR symbol, which is exactly what the decompiler emits, and display names carry no uniqueness guarantee so two declarations sharing a name collapse the map; (c) key on both with a written precedence and collision rule. Compatibility cost is low in one specific sense: the `reroute` opcode is undocumented — `wiki-src/material.mgir.md` is the only MGIR overlay, there is no MGIR language reference (unlike `Docs/crir-language-reference.md:170-182` for CRIR), and a wiki-wide `DisplayName` grep returns only four unrelated pages — so no published grammar constrains a re-key. The same fact means no caller can currently discover the feature either.
- `#3-second-defect-in-the-same-branch-and-zero-test-coverage` `OPEN` reporter — Found while confirming the above, and it argues for fixing both together. The declaration-vs-usage discriminator at `MGIRCompiler.cpp:698` is `!SourceReference.IsEmpty()`, but the decompiler omits `Source:` entirely when the declaration's input is unconnected (`MGIRDecompiler.cpp:422-426`; `FormatInputRef` returns empty at `:150-155`). So an **unwired** named-reroute declaration decompiles to `%nXXXX = reroute nXXXX (DisplayName: "Foo")`, recompiles down the USAGE branch, and dies with `MGIR_REROUTE_NOT_FOUND` — it does not round-trip at all, and the emitter's own support for the unwired case (`MGIRExpressionEmitter.cpp:314-323`) is unreachable dead code. `DisplayName:` present with `Source:` absent is exactly the signal that would disambiguate the two branches, which is a third reason this is a grammar decision rather than a patch. Coverage: `Tests/` has **zero** hits for `NamedReroute` — every `reroute` hit belongs to BPIR, `blueprint.graph.create_reroute_node`, CRIR or Niagara — so `EmitNamedRerouteDeclaration`, `EmitNamedRerouteUsage` and `MGIR_REROUTE_NOT_FOUND` are all untested. Write the failing test first: a `TestMGIRNamedRerouteRoundTrip.cpp` asserting `Declaration->Name` survives compile -> decompile -> compile, in the graph-asserting style of `TestMGIRExtendExistingHandle.cpp:10-11`.
