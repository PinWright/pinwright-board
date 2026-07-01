---
id: E-gas-add-execution-calc-requires-c-suffix
title: "gas.add_effect_execution_calculation rejects the bare exec-calc asset path with CLASS_NOT_FOUND, silently requiring the undocumented _C generated-class suffix"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [gas, add_effect_execution_calculation, class-resolution, loadclass, docs]
encounters: 1
lastSeen: 2026-06-25T07:14:35Z
---

# `gas.add_effect_execution_calculation` rejects the bare exec-calc asset path, requiring an undocumented `_C` suffix

`gas.add_effect_execution_calculation {... calculationClass:"/Game/GAS/DamageMitigationExec"}`
— the bare `/Game/<Path>/<Asset>` exec-calc path that `gas.create_execution_calculation`
*returns* as its `assetPath`, and that every other GAS verb in a normal authoring
run takes as an asset path (`gas.get_gas_info`, `gas.set_execution_capture`,
`asset.save`) — fails with:

```
[CLASS_NOT_FOUND] Calculation class not found: /Game/GAS/DamageMitigationExec
```

The call only succeeds once the caller appends the generated-class suffix `_C`
(`/Game/GAS/DamageMitigationExec.DamageMitigationExec_C`). Nothing in the
`gas.add_effect_execution_calculation` wiki section, the calculation-class param
description ("Path to the calculation class"), or the error text tells the caller
this — the error reads like the calc asset is missing rather than "you gave me an
asset path where I wanted a generated-class path." The caller is forced into a
trial-and-error retry (one `is_error` CLASS_NOT_FOUND, then the `_C` variant) to
discover the required form.

## Why this is awkward (resolver-consistency gap — quotable contradiction)

This is the **same defect class** that `E-class-name-format-inconsistency` (DONE)
built `ClassUtils.cpp::ResolveUClass` to fix — that helper tries direct load,
`_C`-appended load, then `LoadObject<UBlueprint>` → `GeneratedClass`, so callers
never have to type `_C`. That sweep routed `inspect_class`,
`widget.create_widget_blueprint`, and `asset.search_assets` through `ResolveUClass`;
follow-ups routed `ui.create_hud` (`E-create-hud-requires-c-suffix-class-path`,
IN-REVIEW) and `ui.activatable_push` (`E-activatable-push-requires-c-suffix`,
IN-REVIEW), and the GAS cooldown/cost setters are being routed through it in
`B-set-ability-cooldown-cost-class-path-silent-drop` (IN-REVIEW).
`gas.add_effect_execution_calculation` was in **none** of those sweeps and still
does a raw `LoadClass<UGameplayEffectExecutionCalculation>`-style load on the
literal string, rejecting the bare asset path.

The `ui.create_hud` wiki page now documents, verbatim, that the `_C` suffix is
resolved for you and optional "**matching every other class-path slot in the
API**" — a class-path slot in the `gas.*` namespace that `add_effect_execution_calculation`
concretely violates.

Distinct from the sibling GAS path-resolution ticket
`B-set-ability-cooldown-cost-class-path-silent-drop`: that one is a **silent
no-op** (the cooldown/cost setters return `ok:true` and echo the path while
writing nothing to the CDO when the path doesn't resolve as a UClass). This verb
**hard-rejects** with `CLASS_NOT_FOUND` — no silent success, a different failure
mode and a different handler — so it is an ergonomic retry (E-), not a silent-write
bug (B-). It is also distinct from `E-gas-execution-capture-attribute-set-prereq-undocumented`
(the `attribute` capture *format* + compile prereq on `set_execution_capture`) and
from the judge-filed `E-gas-info-exec-calc-no-captures-readback` (the readback gap
on the same exec-calc asset).

## Evidence (this task — focus `gas.create_execution_calculation`)

From the audited task's call-log and friction note:

> "(1) gas.add_effect_execution_calculation rejected the plain asset path
> /Game/GAS/DamageMitigationExec with CLASS_NOT_FOUND — it requires the _C
> generated-class path (.DamageMitigationExec_C), which the wiki's vague 'Path to
> the calculation class' param doc does not state."

Call sequence (consecutive):
1. `gas.add_effect_execution_calculation {... calc:"/Game/GAS/DamageMitigationExec"}` (bare asset path)
   → `is_error` `[CLASS_NOT_FOUND] Calculation class not found: /Game/GAS/DamageMitigationExec`
2. `gas.add_effect_execution_calculation {... calc:"/Game/GAS/DamageMitigationExec.DamageMitigationExec_C"}` (with `_C`)
   → success.

Identical asset, only `_C` differs — so the load failure is purely the missing
`_C` append, not a missing asset.

## What it should do

- Route `gas.add_effect_execution_calculation`'s calculation-class param through
  the canonical `ResolveUClass` (instead of the raw `LoadClass`) so the bare
  `/Game/<Path>/<Blueprint>.<Blueprint>` path (and short names) resolve to the
  generated class without `_C` — matching `ui.create_hud`, `widget.add`, and the
  resolver contract from `E-class-name-format-inconsistency`. Keep an
  `IsChildOf(UGameplayEffectExecutionCalculation)` guard after resolution so the
  downstream cast stays type-safe (`ResolveUClass` returns any `UClass*`); reject
  `INVALID_TYPE` if it resolves to a non-exec-calc class.
- Until the resolver lands, the `gas.add_effect_execution_calculation` wiki overlay
  (`docs/wiki-src/gas.md`) should state that the calculation-class param is a
  **generated-class** path requiring `_C`
  (`/Game/.../DamageMitigationExec.DamageMitigationExec_C`), and the
  `CLASS_NOT_FOUND` error text should hint "append `_C` for a Blueprint exec calc"
  when the bare asset path was given.

**Workaround:** Append `_C` to the exec-calc asset path passed to
`gas.add_effect_execution_calculation`.

## History
- `#2-resolveuclass-route` `IN-REVIEW` developer — Routed `gas.add_effect_execution_calculation`'s `calculationClass` param through the canonical forgiving `ResolveUClass` instead of the raw `LoadClass<UGameplayEffectExecutionCalculation>(nullptr, *Path)`, so the bare asset path (`/Game/.../DamageMitigationExec` — exactly what `gas.create_execution_calculation` returns as its `assetPath`) now resolves to the generated class without a hand-appended `_C`. Because `ResolveUClass` returns any `UClass*`, added an `IsChildOf(UGameplayEffectExecutionCalculation::StaticClass())` guard that rejects a resolved-but-wrong-type class with `INVALID_TYPE` before the downstream wire — mirroring the in-file `ResolveGameplayEffectClassArg` (cost/cooldown) pattern. CLASS_NOT_FOUND is preserved for a genuinely unresolvable path. File: `Source/PinWright/Private/Handlers/Systems/GASHandler.cpp` (`gas.add_effect_execution_calculation`, ~:2191). Regression test: `PinWright.gas.add_effect_execution_calculation.ResolvesBarePath` in `Source/PinWright/Private/Tests/Gameplay/TestGASHandlers.cpp` — creates an exec-calc BP via `gas.create_execution_calculation`, adds it to a GE using the BARE path (no `_C`), asserts success + that the CDO's single execution's `CalculationClass` is the exec-calc generated class, and asserts a GameplayEffect (non-exec-calc) path is rejected `INVALID_TYPE`. Reverting to the raw `LoadClass` fails the bare-path success assertion; dropping the `IsChildOf` guard fails the `INVALID_TYPE` assertion. Not compiled here (a later phase runs the build + suite).
- `#1-initial-audit` `OPEN` reporter — Struggle-auditor finding from a GAS damage-mitigation exec-calc task (focus `gas.create_execution_calculation`; culprit `gas.add_effect_execution_calculation`). The bare `/Game/GAS/DamageMitigationExec` path (returned by `gas.create_execution_calculation`) was rejected `[CLASS_NOT_FOUND] Calculation class not found` (1 is_error), then the `…DamageMitigationExec_C` variant succeeded — a trial-and-error retry the wiki/param/error never warned about. Same `_C`-suffix raw-`LoadClass` defect class as the DONE `E-class-name-format-inconsistency` sweep and its IN-REVIEW follow-ups (`E-create-hud-requires-c-suffix-class-path`, `E-activatable-push-requires-c-suffix`, `B-set-ability-cooldown-cost-class-path-silent-drop`), but `gas.add_effect_execution_calculation` was in none of them. Distinct PROCESS angle from the judge-filed `E-gas-info-exec-calc-no-captures-readback` (readback gap on the same exec calc) and from `E-gas-execution-capture-attribute-set-prereq-undocumented` (the capture *format* on `set_execution_capture`). Unlike the sibling `B-set-ability-cooldown-cost-class-path-silent-drop` (silent no-op), this verb hard-rejects, so it is E- not B-. Proposed: route the calc-class param through `ResolveUClass` + `IsChildOf(UGameplayEffectExecutionCalculation)` guard so the bare path works; floor docs on `docs/wiki-src/gas.md`.
