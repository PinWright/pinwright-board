---
id: E-bpir-dollar-chained-struct-field
title: "BPIR `$name.a.b` chained property access is still unsupported for dollar references"
status: DONE
severity: Low
category: ergonomic
tags: [bpir, dollar-ref, chained-access, struct-field, resolver]
---

# BPIR `$name.a.b` chained property access is still unsupported for dollar references

The resolver now supports single-level `$name.field` on struct-typed entry params by routing the `$target.property` pre-emit path through `ResolveStructMemberThroughPin`. But deeper chains like `$Payload.Transform.Location.X` are still unsupported on the dollar-reference path.

The reason is architectural: `PreEmitVariableRefs()` still splits `$...` references on the first dot only, then calls `PreEmitExternalGet(Target, Property)` with a flat `(Target, Property)` pair. The `%ref.Pin.Prop` path already has a proper chained resolver (`ResolveChainedPropertyAccess`), but `$name.a.b` still depends on `ExternalGetCache` keys built from a single property segment.

**Current behavior:**

```
set StoredX = $Payload.Transform.Location.X
```

`PreEmitVariableRefs()` treats this as:
- target = `Payload`
- property = `Transform.Location.X`

That string does not map cleanly onto either the object-property-get path or `ResolveStructMemberThroughPin`, so the compile still fails.

**Workaround:** bind the intermediate value to a `%ref` and continue from there.

```
%transform = $Payload.Transform
set StoredX = %transform.Location.X
```

or split it further if the intermediate is also a struct:

```
%transform = $Payload.Transform
%location = %transform.Location
set StoredX = %location.X
```

**Proposal:** unify `$name.a.b` resolution with the existing chained-access machinery used by `%ref.Pin.Prop`, or extend `PreEmitExternalGet`/`ExternalGetCache` to understand property chains rather than only a single property segment.

## History
- `#1-follow-up-from-b-bpir-override-param-not-resolvable` `OPEN` developer — While fixing single-level `$struct.field` access for override/event params, re-scan confirmed the chained `$name.a.b` limitation still exists. Current code in `FBpirCompiler::PreEmitVariableRefs` only splits on the first dot before calling `FBpirValueResolver::PreEmitExternalGet`, so the dollar-reference path still lacks parity with `%ref` chained property access.
- `#2-desugared-multi-dot-in-pre-emit` `IN-REVIEW` developer — Extracted the chain-walk loop from `ResolveChainedPropertyAccess` into `FBpirValueResolver::ResolveChainFromPin` in `BpirValueResolver.cpp/.h`; modified `PreEmitExternalGet` to detect a multi-segment property chain, resolve the first segment (struct or object) as today, then fold the remainder through the shared helper. `ExternalGetCache` key is the full dotted property path, matching what `ResolveValue` already looks up. No BPIR syntax change. Pinned by `FBpirDollar_ChainedStructFieldAccess_Test` in `TestBpirDollarStructFieldAccess.cpp`.
- `#3-verified-chain-resolves` `DONE` tester — Verified live on `/Game/App/UI/Test/W_McpVerifyTemp`: `compile_bpir` body `entry custom_event T_chain2(const struct<HitResult>& H) { %sx = call Conv_DoubleToString(InDouble: $H.Location.X) }` returned `success: true, nodeCount: 4, status: "UpToDate", errors: [], warnings: []`. Also verified on `/Game/App/UI/Test/BP_McpVerifyActor` with override-entry: `entry override ReceivePointDamage(...) { %sx = call Conv_DoubleToString(InDouble: $HitInfo.Location.X) ... }` — also success. Two-deep chain `$H.Location.X` resolves through the shared `ResolveChainFromPin` helper for both custom_event and entry-override paths.
