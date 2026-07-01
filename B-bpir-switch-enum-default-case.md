---
id: B-bpir-switch-enum-default-case
title: "BPIR switch_enum rejects `default ->` case despite being documented"
status: DONE
severity: Medium
category: bug
tags: [bpir, compiler, switch-enum, docs-mismatch]
---

# `switch_enum` `default ->` arm rejected at compile time

The BPIR compiler refuses `default ->` (and any other catch-all label)
inside `switch_enum<T>(...) [...]` case lists, even though the syntax
is explicitly documented:

- `docs/bpir-language-reference.md:427` —
  `%sw = switch_enum<EWeaponType>($WeaponType) [Sword -> @sword, Bow -> @bow, default -> @other]`
- `docs/bpir-examples.md:476` and `:765` — both show `default -> @label`
  arms (one example uses `EReplaySaveState` specifically).

Live repro (this session):

```
%sw = switch_enum<EReplaySaveState>(%state) [EReplaySaveState::NotAvailable -> @hide, default -> @maybe_show]
```

→ `COMPILE_FAILED: Could not find exec output pin 'default' on node`.

`UK2Node_SwitchEnum` doesn't expose a `default` exec pin (unlike
`UK2Node_SwitchInteger`); the engine instead synthesizes the
"default" behavior by leaving unused enum values unwired. The BPIR
compiler must take the same approach: when a `default -> @label`
arm appears, enumerate every `T` value not already listed and wire
each unspecified value's exec output to `@label`.

## Why it matters

Without `default ->`, agents authoring switches on multi-value
enums must enumerate every non-default case explicitly. For 5-value
enums (e.g. `EReplaySaveState`) that's tolerable. For 10+ value enums
(`EDataValidationResult`, project enums, `ESlateVisibility` if used
in full) it produces noisy, fragile case lists that must be updated
every time the enum gains a value.

It also breaks the published BPIR docs as a contract: examples that
agents copy-paste fail to compile.

## Workaround

Enumerate every enum value explicitly. For the session's case:

```
[EReplaySaveState::NotAvailable -> @hide,
 EReplaySaveState::Idle         -> @show,
 EReplaySaveState::Recording    -> @show,
 EReplaySaveState::Uploading    -> @show,
 EReplaySaveState::Ready        -> @show]
```

## Fix

Recommended: keep the documented `default -> @label` form. In the
BPIR compiler's `switch_enum` lowering, after walking the explicit
arms, look up the `UEnum` for `T` via the existing `ResolveUEnum`
helper, iterate its entries via `UEnum::GetMaxEnumValue()` /
`GetNameByIndex()`, and for each value not already in the explicit
arm map, wire that value's exec output pin to the default label.
Skip the `_MAX` hidden sentinel.

The decompiler should perform the inverse: if N-1 of the N enum
values share a single target label, emit those as a single
`default -> @label` arm and only enumerate the odd-one-out
explicitly. This collapses common patterns (e.g. "NotAvailable
hides, everything else shows") back to the readable form on
round-trip.

Alternatives weighed:

- **`_ -> @label` wildcard** — shorter, but introduces a new
  reserved token. The docs already advertise `default ->`, so
  use that.
- **A macro / helper opcode** (e.g. `switch_enum_with_default`)
  — adds surface area for no real win; the `default` arm is a
  property of the case list, not the opcode.
- **Status quo + better error message** — would clarify the
  failure but doesn't fix the docs-vs-impl contract violation
  or the verbosity problem on large enums.

## History
- `#1-initial-repro` `OPEN` reporter — `switch_enum<EReplaySaveState>(%state) [EReplaySaveState::NotAvailable -> @hide, default -> @maybe_show]` failed with `COMPILE_FAILED: Could not find exec output pin 'default' on node`. Two documented examples in `bpir-language-reference.md` and `bpir-examples.md` advertise the `default -> @label` arm, but the compiler doesn't synthesize it for `switch_enum`. Fix: in switch_enum lowering, expand `default` to all enum values not explicitly listed; mirror with a decompiler that collapses N-1-shared-target into a single `default ->` arm. Workaround in-session was to enumerate the 4 non-NotAvailable values explicitly.
- `#2-default-arm-enum-expansion` `IN-REVIEW` developer — switch_enum `default ->` arm now expands to every non-_MAX non-Hidden enum entry not already wired by an explicit arm; compiler fix in `BpirCompiler.cpp` (new `WireSwitchEnumDefaultArm` helper + intercept in the step-3 wiring loop before `FindExecOutputPin`); regression test `FCompilerIntegrationCompileSwitchEnumDefaultTest` in `TestCompilerIntegration.cpp` compiles `switch_enum<ESlateVisibility>(...) [Visible -> @a, default -> @b]` and asserts each of the 4 non-Visible entries (Collapsed, Hidden, HitTestInvisible, SelfHitTestInvisible) links to the default chain; decompile-side collapse of N-1-shared arms back into a `default ->` arm deferred to a follow-up (TODO breadcrumb left in `BpirTextEmitter::FormatEnumExecTargets`).
- `#3-verify-default-expansion` `DONE` tester — Verified: created `/Game/App/UI/Test/W_McpVerifyTemp_B_bpir_switch_enum_default_case`, ran `blueprint.compile_bpir` with `switch_enum<ESlateVisibility>(%e) [ESlateVisibility::Visible -> @a, default -> @b]`, observed `compiled: true`, `errors: []`, `nodeCount: 4`, and `blueprint.decompile` showed Collapsed, Hidden, HitTestInvisible, and SelfHitTestInvisible routing to the default "Other" chain; temp asset deleted with `asset.delete`.
