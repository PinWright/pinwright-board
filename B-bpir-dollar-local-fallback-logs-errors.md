---
id: B-bpir-dollar-local-fallback-logs-errors
title: "BPIR $local fallback logs errors and emits orphan variable getters before %local resolution"
status: DONE
severity: Medium
category: bug
tags: [bpir, compiler, value-resolver, dollar-local]
---

# BPIR dollar-local fallback creates orphan getters before resolving locals

BPIR allows `$name` to resolve as a Blueprint member variable or registered parameter, with an ergonomic fallback to an existing `%name` local when no Blueprint variable is available. The fallback order was wrong: `PreEmitDollarVar("$v")` tried to create a `UK2Node_VariableGet` for `v` before `ResolveValue` could fall back to the existing `%v` local.

For non-member locals, that produced failed getter logs such as `PreEmitDollarVar: VariableGet node for '$v' has no non-exec output pin` and left an orphan variable-get node in the generated graph. Compilation could still recover through `%v`, but the emitted graph and logs were wrong.

**Workaround:** Use `%v` explicitly when referencing locals.

**Fix:** Only pre-emit `$name` getters for registered parameters, cached getters, or real Blueprint member variables on `SkeletonGeneratedClass` / `GeneratedClass`. During direct `$name` resolution and alias RHS resolution (`%alias = $name`), check `FCodePinResolver`, then the variable-get cache, then `Block.ValueIndex` for `%name` fallback before logging a missing dollar variable.

## History
- `#1-misfiled-return` `OPEN` reporter — Returned failure was originally attached to `F-anim-state-machine-internals`, but the observed logs were BPIR value-resolution failures for `$v`: `PreEmitDollarVar` attempted an invalid variable getter before the existing `%v` local fallback could resolve.
- `#2-dollar-local-fallback` `IN-REVIEW` developer — Changed `BpirValueResolver.cpp` to gate dollar pre-emission on actual Blueprint member variables and to resolve direct `$name` and alias RHS `$name` through `FCodePinResolver`, cached getters, then `%name` local fallback before logging. Added `FBpirDollarLocalFallbackNoOrphanGetterTest` covering `%v = make<Vector>(...)` followed by both `break<FVector>($v)` and `%alias = $v` / `break<FVector>(%alias)`, asserting compile success, a break-struct node, and zero `UK2Node_VariableGet` nodes. Counterfactual: if the resolver fix is reverted, the direct `$v` path or alias RHS `$v` path emits or logs through an orphan getter instead of falling back to `%v`, so the compile or zero-getter assertion fails.
- `#3-verify-fix` `DONE` tester — Verified: ran `system.run_tests` for `EditorAutomationRpcGateway.bpir.compiler.DollarLocalFallback.NoOrphanGetter` (ticket `j_20260516T154349_7e816472`), completed in 5.3s with `has_errors: false`. Also live-exercised the direct fallback path via `blueprint.compile_bpir` on a temp `W_McpVerifyTemp_BpirDollarLocal`: `entry function TestDollarLocal { %v = make<Vector>(...); %parts = break<FVector>($v) }` returned `compiled: true`, `nodeCount: 2`, zero errors — only the MakeVector + BreakStruct primaries, no orphan VariableGet. Temp BP cleaned up via `asset.delete`.
