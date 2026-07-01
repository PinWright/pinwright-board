---
id: F-niagara-link-module-input-to-parameter
title: "niagara.set_module_input cannot link an input to a User/system parameter (literal-only)"
status: IN-REVIEW
severity: Medium
category: feature
tags: [niagara, stack, set-module-input, linked-input, parameter-binding, authoring, docs]
---

# `niagara.set_module_input` cannot link a module input to a User/system parameter

`niagara.set_module_input` writes a **literal** override into the stack
module's override store. There is no way to make the input *read from* an
existing parameter (e.g. bind a particle `Color` input to a
`User.WispColor` LinearColor parameter, or a `SpawnRate` input to
`User.SpawnRate`). The documented `value` parameter is typed `any` (wiki
`niagara.set_module_input.md` line 14: `value (any, required): Value to
assign`) with no hint that a parameter reference / linked input is not a
valid value — it silently behaves as literal-only.

This is a real authoring intent, not an exotic one. The natural way to
expose a tunable in a Niagara system is: create a `User.*` parameter, then
**link** the relevant stack-module input to it so the user-facing
parameter actually drives the effect. The engine supports this — it is the
`UNiagaraNodeInput` / `$Scope.Name` "linked parameter" override case that
the NIR decompiler already *reads back* (see
`F-niagara-decompile-nir-overrides` `#2-implementation`, which classifies
override pins into `direct literal` vs `$Scope.Name linked parameter` vs
`dynamic ModuleName {...}`). The read side understands linked parameters;
the **write** side (`set_module_input`) exposes only the literal case.

## Why it matters

In this task the author's intent (step 5) was: "Tint the particles a
glowing blue color via a system/user color parameter." They correctly
`add_parameter`'d `User.WispColor (0.05,0.4,1,1)`, then went to wire the
particle `Color` input to it — and could not. The only available move was
to set the particle `Color` input to a **matching blue literal** that
duplicates the parameter's value. The result *looks* correct but is
**desynchronized**: changing `User.WispColor` later (the entire point of a
user parameter) will not retint the particles, because the particle Color
is a hardcoded literal, not a link. The user-exposed knob is decorative.

This is the standard pattern for every "expose a tunable" workflow
(spawn rate, lifetime, speed, color driven by `User.*`), so the literal-
only restriction quietly defeats parameter-driven authoring across the
namespace.

## Evidence (this task, namespace `niagara`)

End-to-end authoring of `NS_ArcaneWisp` (33 calls). Friction note,
verbatim second half:

> "set_module_input only accepts literal values so I could not link the
> Color input to User.WispColor (had to set a matching blue literal)."

Call-log corroboration:
- `niagara.add_parameter` `User.WispColor` color `(0.05,0.4,1,1)` -> ok
- `niagara.set_module_input` `InitializeParticle Color=(0.05,0.4,1,1)` ->
  ok, but a **duplicated literal**, not a link to `User.WispColor`.

The author achieved the *appearance* of the requested behavior but not the
requested behavior (a parameter that drives the color).

## Proposal

Allow `niagara.set_module_input` to accept a **linked-parameter** value, or
add a sibling RPC. Two shapes (pick one; first is least surface):

```
niagara.set_module_input(
    assetPath, target, entryId, inputName,
    value: { link: "User.WispColor" }   // OR a typed sentinel, e.g.
    value: "$User.WispColor"             // string with link prefix
    ...
) -> { linked: true, parameter: "User.WispColor" }
```

or an explicit verb:

```
niagara.link_module_input(
    assetPath, target, entryId, inputName,
    parameter: "User.WispColor",   // must already exist; type must match input
    scriptUsage?, compile?, save?
) -> { linked: true, parameter, parameterType }
```

Reject with a clear error if the parameter does not exist or its type does
not match the input (mirror `B-set-niagara-param-no-validation`'s lesson —
no silent no-ops). Implementation surface is the override-pin path already
used by `set_module_input` / `reset_module_input`: instead of writing a
literal default pin, attach a `UNiagaraNodeInput` (or parameter-map-get
linked pin) wired to the named parameter in the
`UNiagaraNodeParameterMapSet` override node. The decompiler's
`$Scope.Name` classifier (`F-niagara-decompile-nir-overrides`) is the
inverse and confirms the node shape.

## Docs (discoverability half)

Even before the capability lands, the **literal-only restriction is
undocumented**. `value: any` actively implies a parameter reference might
be accepted. Improve overlay page `docs/wiki-src/niagara.authoring.md`
(and/or the `niagara.md` edit-list it links): state that
`set_module_input` writes a **literal** override only, that linking an
input to a `User.*`/system parameter is **not** currently supported, and
that hardcoding the parameter's value as a literal does **not** create a
live binding (it desynchronizes if the parameter changes).

## Cross-ref

- `F-niagara-decompile-nir-overrides` (DONE) — read side already classifies
  the `$Scope.Name` linked-parameter override case this ticket wants to
  *write*.
- `F-niagara-reset-module-input` (DONE) — shares the override-pin write
  path; the link writer plugs into the same node walk.
- `B-niagara-module-input-stack-infer` (OPEN) — the *other* friction on the
  same call in this task (INVALID_STACK when `scriptUsage` omitted); this
  ticket is the orthogonal capability gap.
- `B-set-niagara-param-no-validation` — precedent for "no silent no-op /
  validate the parameter name+type."

## History
- `#2-implementation` `IN-REVIEW` developer — Implemented the linked-parameter value form (shape #1, least surface): `niagara.set_module_input` now accepts `value: { link: "User.WispColor" }` (alias `{ parameter: ... }`) and binds the module input to *read from* that parameter instead of writing a literal. In `NiagaraEditHandler.cpp` `ApplyModuleMutation`'s `SetModuleInput` branch: detect the link request (`TryGetLinkedParameterRequest`), resolve+validate the parameter (`ResolveLinkedParameter` — a `User.*` parameter must exist in `System->GetExposedParameters()` else `PARAMETER_NOT_FOUND`; engine/system-scope reads use the module input's declared type), type-match against the input's declared type (`FindModuleInputDeclaredType` via `EnumerateScriptInputs`, else `PARAMETER_TYPE_MISMATCH`), clear any prior override (`NiagaraResetModuleInput::RemoveOverridePinAndChainedNodes`), then create a fresh override pin and wire the parameter read via the engine's `FNiagaraStackGraphUtilities::SetLinkedParameterValueForFunctionInput` (5.6+) / `SetLinkedValueHandleForFunctionInput` (≤5.5) — the inverse of the NIR decompiler's `$Namespace.Name` classifier. The handler now reports `{ linked: true, parameter, parameterType }`; the literal path still reports `linked: false`. Files: `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraEditHandler.cpp` (helpers + branch + handler result + `value` param doc), `Docs/wiki-src/niagara.authoring.md` (new "Module inputs: literal value vs linked parameter" section documenting both shapes, the validation errors, and the literal-is-not-a-live-binding desync warning). Tests (in `Source/EditorAutomationRpcGateway/Private/Tests/Niagara/TestNiagaraSetModuleInput.cpp`): `LinksParameter` adds `User.WispColor` via the production `add_parameter` RPC, links the InitializeParticle `Color` input to it, and asserts the response reports `linked: true` / `parameter: "User.WispColor"` AND the override pin actually reads from a parameter-read node naming `User.WispColor` (would fail if the write regressed to a literal); `LinkMissingParameterRejected` asserts a link to a non-existent user parameter returns `PARAMETER_NOT_FOUND` (no silent literal fallback). Not compiled/tested here — a later phase drives green.
- `#1-initial-audit` `OPEN` reporter — Process-audit finding from the `niagara` `NS_ArcaneWisp` end-to-end authoring task (33 calls). `niagara.set_module_input` accepts literal values only; the author created `User.WispColor` then could not link the particle `Color` input to it, so set a duplicated blue literal — producing the *appearance* of a user-driven tint while leaving the user parameter decorative (changing it won't retint). Wiki types `value` as `any` (`niagara.set_module_input.md:14`) with no hint that a parameter link is unsupported. Engine supports linked params (the NIR decompiler reads the `$Scope.Name` linked-parameter override case — `F-niagara-decompile-nir-overrides` `#2-implementation`); only the write RPC lacks it. Proposes a linked-parameter value form or a `niagara.link_module_input` verb on the existing override-pin path, plus a `docs/wiki-src/niagara.authoring.md` note that `set_module_input` is literal-only and a hardcoded literal is not a live binding.
