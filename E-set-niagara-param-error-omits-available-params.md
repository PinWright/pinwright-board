---
id: E-set-niagara-param-error-omits-available-params
title: "effect.set_niagara_parameter / niagara.modify_parameter PARAMETER_NOT_FOUND names the bad param but doesn't enumerate the system's exposed user params, forcing a follow-up niagara.inspect"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, effect, set-niagara-parameter, modify-parameter, error-message, discovery, docs]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# Runtime Niagara param-setter `PARAMETER_NOT_FOUND` doesn't list the params that ARE exposed

Now that `B-set-niagara-param-no-validation` (IN-REVIEW) makes the runtime
component-side setters fail loudly for a non-existent name, the error they return
is correct but **dead-ends discovery**. `effect.set_niagara_parameter`
(`Handlers/VFX/EffectHandler.cpp:712-715`) emits:

```
[PARAMETER_NOT_FOUND] Niagara system on '<actor>' has no <Type> user parameter named '<name>'
```

It names the bad parameter and its type, but says **nothing about which user
parameters the system DOES expose**. The caller is told "not this one" with no
list of valid alternatives, so the only way forward is a separate discovery call
(`niagara.inspect <assetPath>` to read the asset's exposed-parameter defaults)
and then a guess at the right name. The sibling runtime setter
`niagara.modify_parameter` (`Handlers/Niagara/NiagaraHandler.cpp`) shares the same
validation gate and the same enumeration-free error.

The fix is cheap because the enumeration is already in hand on the failing code
path. The shared validation helper
`EditorAutomationNiagara::ComponentExposesParameter` resolves the component's
system and queries `System->GetExposedParameters()` — the
`FNiagaraUserRedirectionParameterStore` (`NiagaraInstanceUtils.cpp:69-82`) — to
do the `FindParameterVariable` lookup. The exposed user-parameter names are
directly enumerable from that same store, so the `PARAMETER_NOT_FOUND` branch
could append the available `User.*` params (optionally filtered to the requested
type, with a "did you mean" near-match) without any extra asset load.

This is the discovery/error-quality complement of the family already on the board:
distinct from `B-set-niagara-param-no-validation` (that ticket makes the *write*
fail — owned by the judge — and is exactly what produced this clean but
list-less error); distinct from `E-niagara-graph-set-parameter-opaque-scope`
(that is the **asset-side** `niagara.graph.set_parameter` whose error is *opaque*
about which failure case you hit — here the case is unambiguous, only the valid
set is missing); distinct from `E-niagara-modify-parameter-no-override-readback`
(reading a *successful* override back off the live component). It is the same
"error names the bad value but doesn't enumerate the valid ones" pattern already
accepted as `E-rendering-unknown-property-no-hint`, `E-add-metasound-node-error-no-hint`,
and `E-make-struct-error-hint`, applied to the runtime Niagara param setters.

## Why it matters (process cost in this task)

The destruction-set-piece task (seed `effect.create_impact_effect`) asked to set
a `SpawnRate` Float user param on an ambient emitter driven by
`/Game/ExampleContent/Destruction/FX/NiagaraSystems/NS_Chaos_NDC`.
`effect.set_niagara_parameter {systemName:"ImpactAmbient", parameterName:"SpawnRate",
parameterType:"Float", value:250}` correctly returned
`[PARAMETER_NOT_FOUND] Niagara system on 'ImpactAmbient' has no Float user
parameter named 'SpawnRate'` — the system genuinely has no such param. But the
error gave no hint as to what it *does* expose, so the agent then spent a
follow-up `niagara.inspect NS_Chaos_NDC` (properties-only) purely to discover the
exposed set — which turned out to be only `User.ChaosDestructionData_*` params,
none of them a `SpawnRate`. Friction note, verbatim: *"SpawnRate set failed with a
correct typed error ... verified via niagara.inspect ... the system exposes only
User.ChaosDestructionData_* params."* Had the `PARAMETER_NOT_FOUND` error listed
those exposed params, that extra inspect round-trip would have been unnecessary and
the agent could have reported the available knobs to the user in one shot. The rest
of the task ran clean (no retries, no python fallback).

**Workaround:** on a `PARAMETER_NOT_FOUND` from a runtime Niagara setter, call
`niagara.inspect <systemAssetPath>` to read the system's exposed user-parameter
defaults and pick a real name/type from there.

## What it should do (downstream — not this ticket)

- **Method (E-ergonomic):** in the `PARAMETER_NOT_FOUND` branch of
  `effect.set_niagara_parameter` (`EffectHandler.cpp:712`) and the twin in
  `niagara.modify_parameter`, append the system's exposed `User.*` parameter names
  (and their types) — enumerable from the `FNiagaraUserRedirectionParameterStore`
  the validation helper already holds — so the caller gets the valid set in the
  same error. A type-filtered list plus a near-match suggestion is the ideal.
- **Docs (`docs/wiki-src/effect.md` / `docs/wiki-src/niagara.md`):** under the
  runtime param-setter methods, state that the settable names are the system's
  **exposed `User.*` params**, and that `niagara.inspect <assetPath>` is the way to
  enumerate them before setting — so the round-trip is documented until the richer
  error lands.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the destruction-set-piece task (seed `effect.create_impact_effect`, namespace `effect`, outcome agent_fail on the genuine SpawnRate-absence, not a tool defect). After `B-set-niagara-param-no-validation`'s validation correctly rejected `SpawnRate` with `[PARAMETER_NOT_FOUND] Niagara system on 'ImpactAmbient' has no Float user parameter named 'SpawnRate'`, the error named the bad param but enumerated none of the exposed ones, so the agent spent an extra `niagara.inspect NS_Chaos_NDC` (properties-only) to discover the system exposes only `User.ChaosDestructionData_*` (friction note quoted above). The enumeration is reachable for free: the shared validation helper (`NiagaraInstanceUtils.cpp:69-82`) already holds `System->GetExposedParameters()` on the failing path. Distinct from the judge-owned write-validation bug `B-set-niagara-param-no-validation` (IN-REVIEW), from `E-niagara-graph-set-parameter-opaque-scope` (asset-side opaque-which-case error), and from `E-niagara-modify-parameter-no-override-readback` (successful-override readback). Same "list the valid values in the error" pattern as `E-rendering-unknown-property-no-hint` / `E-add-metasound-node-error-no-hint` / `E-make-struct-error-hint`. Ripgrep across OPEN/closed found no existing ticket on the runtime setter's error omitting the exposed-param list. Pages to improve: `docs/wiki-src/effect.md`, `docs/wiki-src/niagara.md`.
