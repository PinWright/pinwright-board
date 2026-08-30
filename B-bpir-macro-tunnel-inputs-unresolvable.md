---
id: B-bpir-macro-tunnel-inputs-unresolvable
title: "BPIR decompiler fails to trace macro body uses back through K2Node_Tunnel Inputs node, emits ?"
status: DONE
severity: Medium
category: bug
tags: [bpir, decompiler, macro, tunnel]
---

# Macro tunnel `Inputs` value not propagated to body

When a macro graph reads one of its input parameters inside the body,
the decompiler's value-resolution pass fails to trace the use back
through the macro's `K2Node_Tunnel` `Inputs` node and instead emits
`?` at the use site, plus a warning
`Unresolvable value: source node 'Inputs'`. 45 `bpir.txt` files in the
current cache contain this warning.

This is distinct from the (already DONE) composite-entry-point bug —
that fixed entry-point detection on composite subgraphs. This bug
is about value flow inside the macro body, where the body walker
should resolve a pin connected to the tunnel's input side back to
the macro's named input parameter.

**Repro:**
- `Game\Audio\Blueprints\WeaponAudioMacros\bpir.txt` — macro parameter
  `Pawn` arrives through the `Inputs` tunnel; its use inside the
  body emits `?.Instigator` instead of `$Pawn.Instigator`.
- `App\App\UI\LobbyAndMenu\TornamentSelector\W_TornamentSelectButton\bpir.txt`
  — 7 instances in one file.

**Expected:** Macro input pins resolve to `$ParamName` references in the
body, matching the parameter list emitted at the macro header.

**Actual:** `?` at use site + decompile warning per use.

**Fix (proposed):** Extend the value resolver to follow
`K2Node_Tunnel.Inputs.{PinName}` → macro parameter `PinName`.
Likely a small map lookup added when entering a macro body during the
walk.

## History
- `#1-initial-audit` `OPEN` reporter — 45 cached `bpir.txt` files contain `Unresolvable value: source node 'Inputs'`. Macro parameter identity lost at use site inside the body. Repro on `WeaponAudioMacros` and `W_TornamentSelectButton`.
- `#2-resolve-macro-entry-tunnel` `IN-REVIEW` developer — Extended `FBpirDecompiler::ResolveInputValue` in `Source/EditorAutomationRpcGateway/Private/Decompiler/BpirDecompiler.cpp` with a sibling `UK2Node_Tunnel` branch that detects the macro-entry tunnel via `bCanHaveOutputs && !bCanHaveInputs` (matching `GraphWalker.cpp:149`) and emits `$<PinName>` for non-exec output pins, mirroring the existing function-entry parameter branch. `UK2Node_MacroInstance` is unaffected because instances reach the `NodeToValueName` lookup before this branch. Added regression test `Source/EditorAutomationRpcGatewayTests/Private/Bpir/TestBpirDecompileMacroTunnelInputs.cpp` (`FBpirDecompileMacroTunnelInputsTest`) which builds a transient pure macro graph with one `Pawn` object input piped through `UKismetSystemLibrary::IsValid` to the exit tunnel and asserts the BPIR body contains `$Pawn`, contains no ` ?` placeholder, and produces no `source node 'Inputs'` warning.
- `#3-verify-macro-tunnel-fix` `DONE` tester — Decompiled `/Game/Audio/Blueprints/WeaponAudioMacros` via `blueprint.decompile`. `LyraGetWeaponAmmo` macro body emits `$Pawn.Instigator` at the `GetController` call site (was `?.Instigator`); `warnings: []` is empty (no `source node 'Inputs'` warning). PASS.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
