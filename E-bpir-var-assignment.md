---
id: E-bpir-var-assignment
title: "BPIR rejects simple value-aliasing `%tmp = $var`"
status: DONE
severity: Low
category: ergonomic
tags: []
---

# BPIR rejects simple value-aliasing `%tmp = $var`

BPIR doesn't support aliasing a parameter/variable to a new local name:
```
%replayCopy = $Replay   # → COMPILE_FAILED: Expected keyword after '=': %replay = $Replay
```

**Workaround:**
Use `break<StructType>($Var)` to wrap the access: `%b = break<EditorReplay>($Replay)` then `%b.Uid` for member access. Or pass `$Var` directly to every call site.

**Proposal:**
Support `%tmp = $var` as a pure value-aliasing instruction (creates no node — just binds the name in the resolver table for downstream `%tmp.Field` / `call Foo(x: %tmp)` references). Or document the `break<T>(...)` workaround in `bpir-language-reference.md` alongside the existing struct-member-access examples.

## History
- `#1-value-alias-not-supported` `OPEN` reporter — Hit during BPIR wiring of `BP_OnClicked → RequestOpen(Replay.Uid)` on `W_MyReplayListItem`. The intuitive form `%uid = $Replay.Uid` (dot-chain on a variable) also failed, so fell back to `break<EditorReplay>($Replay)` + `%broken.Uid`.
- `#2-alias-opcode-added` `IN-REVIEW` developer — Added `EBpirOpcode::Alias` for pure value-aliasing `%tmp = $var` and `%tmp = %other`. Parser recognizes `DollarRef`/`PercentRef` as valid RHS heads before the keyword dispatch. Emit pass skips node creation and registers the alias's `PrimaryOutputPin` to the same pin as the RHS via `PreEmitDollarVar` / `EmitMap` pin sharing. `ResolvePercentRef` recursively resolves aliases. Decompiler emits `%tmp = rhs` for round-trip via `FBpirTextEmitter::EmitAlias`. Inline member access `%tmp = $var.Field` is NOT supported — callers must use `break<T>(...)` for field access (documented workaround).
- `#3-verified-alias-compiles` `DONE` tester — Verified on `W_McpVerify_BpirAlias`: `%x = $MyFloat` + `call PrintString(%x)` compiled cleanly (3 nodes, no errors). Alias chain `%a = $MyFloat; %b = %a; call PrintString(%b)` also compiled. Negative `%x = $MyFloat.Field` correctly rejected with `COMPILE_FAILED: Unknown instruction keyword after '=': $MyFloat.Field` (documented limitation). Decompile round-trip note: aliases are collapsed via pin sharing (no node emitted), so `%tmp = rhs` doesn't appear in decompiled output — the decompiler bypasses the alias entirely. This is correct given the pin-sharing implementation but slightly inconsistent with the original claim "Decompiler emits `%tmp = rhs`" — minor doc nit, not a functional failure.
