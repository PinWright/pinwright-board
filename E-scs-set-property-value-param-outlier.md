---
id: E-scs-set-property-value-param-outlier
title: "blueprint.scs.set_property names its value param 'propertyValue' — a plugin-wide outlier (every other value-set verb uses 'value'); no 'value' alias"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint, scs, set-property, param-alias, value, propertyvalue, drift]
encounters: 1
lastSeen: 2026-07-17
---

# `blueprint.scs.set_property` uses `propertyValue` where the rest of the plugin uses `value`

`blueprint.scs.set_property` requires its assigned value under the key
`propertyValue` (`SCSHandler.cpp:217`, `RPC_PARAM_REQ("propertyValue", "any", ...)`,
read at `:222/:233`). Every other value-setting verb in the plugin spells that
parameter `value`, including the sibling CDO mutator
`blueprint.set_default` (`BlueprintPropertyHandler.cpp:396`,
`RPC_PARAM_REQ("value", ...)`). `propertyValue` is not merely inconsistent with
one sibling — it is a **plugin-wide outlier**: a scan of required/optional value
params shows `value` used by `blueprint.set_default`
(BlueprintPropertyHandler.cpp:396), `AIHandler.cpp:2193`, `GASHandler.cpp:1634`,
`EffectHandler.cpp:454`, `MorphTargetHandler.cpp:359`, `NiagaraHandler.cpp:453`,
`AnimationAuthoringHandler_Sequence.cpp:704`,
`AnimationAuthoringHandler_AnimBlueprint.cpp:1852`, and
`UtilityPropertyHandler.cpp:897`, while `propertyValue` as a declared param name
appears in exactly one place: `SCSHandler.cpp:217`. There is no `value` alias, so
the canonical spelling is rejected outright.

Because `blueprint.scs.set_property` and `blueprint.set_default` are the same
conceptual operation (set a UPROPERTY that every spawned instance inherits — the
`set_property` summary even points at `actor.set_component_properties` as its
per-instance counterpart), an agent moving from `set_default` to
`scs.set_property` naturally reuses `value` and gets
`[MISSING_REQUIRED_PARAM] Missing required parameter 'propertyValue' (type: any)`
before the handler runs. Accurate error, one round-trip, self-correcting, no
blocked progress — pure friction, matching the Low severity of the rest of the
param-drift family. CLAUDE.md's camelCase/snake_case alias rule does not cover
this: `value` and `propertyValue` are distinct names, not casing variants.

Scope note: this ticket covers **only** the value-param name. The separate
`blueprintPath`-vs-`path`/`assetPath` alias gap on the same `blueprint.scs.*`
verbs (including `set_property`) is already tracked by
`E-scs-blueprintpath-no-path-alias` (OPEN), whose fix keeps the whole namespace
uniformly path-aliased "as new verbs land" — it is deliberately not re-filed
here.

**Fix:** accept `value` as an alias for `propertyValue` on
`blueprint.scs.set_property` (declare the alias in the `FParamSpec` and read it
body-side via the multi-key getter, keeping `propertyValue` for back-compat), so
the value key matches the plugin-wide `value` convention. Audit the rest of the
`blueprint.scs.*` namespace for the same value-name drift while there.

## History
- `#1-propertyvalue-outlier` `OPEN` reporter — Session 2026-07-17: after supplying
  `blueprintPath`, `blueprint.scs.set_property {blueprintPath, componentName,
  propertyName, value}` → `[MISSING_REQUIRED_PARAM] Missing required parameter
  'propertyValue' (type: any)`; retry with `propertyValue` succeeded. Novel half
  of the proposed `E-scs-set-property-param-naming-inconsistent`; the proposal's
  other half (`path`→`blueprintPath`) is a duplicate of the OPEN
  `E-scs-blueprintpath-no-path-alias` and was not re-filed. Verified in source:
  `SCSHandler.cpp:217` declares `RPC_PARAM_REQ("propertyValue", "any", ...)` (read
  at `:222`/`:233`, no `value` alias); `blueprint.set_default` at
  `BlueprintPropertyHandler.cpp:396` declares `RPC_PARAM_REQ("value", ...)`.
  `value` is the dominant plugin-wide spelling (AIHandler.cpp:2193,
  GASHandler.cpp:1634, EffectHandler.cpp:454, MorphTargetHandler.cpp:359,
  NiagaraHandler.cpp:453, UtilityPropertyHandler.cpp:897, and both
  AnimationAuthoring handlers), with `propertyValue` unique to SCSHandler.cpp.
  Severity Low (accurate error, one round-trip, self-corrects, no blocked
  progress).
