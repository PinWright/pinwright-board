---
id: E-niagara-connect-pins-add-pin-opaque
title: "niagara.graph.connect_pins discards the Niagara schema's specific rejection reason and returns a generic [CONNECTION_FAILED] that names no pin/type/cause"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [niagara, niagara-graph, connect-pins, dynamic-add-pin, parameter-map, error-message, docs]
encounters: 1
lastSeen: 2026-07-05T11:42:41.6664801+03:00
---

# `niagara.graph.connect_pins` swallows the schema's own rejection reason

`niagara.graph.connect_pins` delegates the whole decision to a bare
`TargetGraph->GetSchema()->TryCreateConnection(FromPin, ToPin)` and, on `false`,
reports only:

```
[CONNECTION_FAILED] Failed to connect pins (schema blocked connection).
```

That message names **no pin, no pin type, and no cause** — even though the Niagara
schema *computed* a specific reason and `TryCreateConnection` then threw it away.
Internally `UEdGraphSchema_Niagara::TryCreateConnection` calls
`CanCreateConnection` and switches on the response; a rejection is a
`CONNECT_RESPONSE_DISALLOW` whose `.Message` says exactly why (e.g. "Types are not
compatible", "Cannot make connections to or from add pins for non-parameter
types"). The handler discards that message and substitutes the opaque generic
string, so the caller is left with nowhere to go.

## The originally-reported case (and why the first framing was wrong)

The report came from wiring a new `Multiply` op's `Result` into a Map Set add pin
(`connect_pins Mul.Result -> MapSet.Add (DynamicAddPin)`), which returned the
opaque `[CONNECTION_FAILED]`. The original ticket concluded that `connect_pins`
*categorically cannot* wire an output into a Map Set add pin and that the RPC needs
to replicate the editor's drag-onto-`+` behavior. Source verification shows that
premise is inaccurate:

- `UEdGraphSchema_Niagara::CanCreateConnection` **does** allow wiring a
  concretely-typed output into a Map Set `DynamicAddPin`. The add-pin disallow
  (`GetPinsAreInvalidAddPinCombination`, `AddPinIncompatibleTypeText`) fires only
  when the add pin's peer is *also* a Misc pin — a concretely-typed output is a
  Type pin and falls through to `CONNECT_RESPONSE_MAKE`.
- On a MAKE, `UNiagaraNodeWithDynamicPins::PinConnectionListChanged` then
  **auto-materializes** a named, typed pin from the linked output (deriving the
  name and type from the source) — i.e. the engine already performs the
  "materialize a named map entry and wire the value into it" that the original
  option (a) proposed to hand-roll.
- The reported failure was scenario-specific: a `Multiply` `Result` pin is
  generic-numeric until inference resolves it, and the add-pin accept check
  rejects an unresolved generic-numeric source, returning
  `CONNECT_RESPONSE_DISALLOW` with "Types are not compatible" — a **specific**
  reason `connect_pins` swallowed.

So the defect is an **error-message-quality** one: the reason exists, and the
handler discards it. It is not a missing capability.

## What it should do

Mirror the dual-remedy pattern the board settled on for the sibling opaque
Niagara-graph error `E-niagara-graph-set-parameter-opaque-scope` (name the case +
point at the supported path):

- **Make the error actionable** — compute `CanCreateConnection` first and, on
  `CONNECT_RESPONSE_DISALLOW`, return the schema's own `Response.Message`, naming
  both pins and their Niagara types. When a pin is a Map Set add pin
  (`DynamicAddPin`), add a supported-path hint: the add pin takes a direct wire
  only from a concretely-typed output (the engine then materializes the named
  entry), so a generic-numeric source must have its type resolved first. Use a
  distinct `CONNECTION_DISALLOWED` code for parity with the singular
  `niagara.connect_pin`, which already reports `CanCreateConnection().Message`.
- **Docs (`docs/wiki-src/niagara.graph.md`, `connect_pins` section):** state that a
  blocked connection reports the schema's reason, and that a Map Set `+`
  (`DynamicAddPin`) *does* accept a direct wire from a concretely-typed output
  (auto-materializing a named entry) but rejects a generic-numeric source until
  its type resolves.

The original option (a) — have `connect_pins` special-case the add pin to
materialize-and-wire a named entry — is **dropped**: the engine's
`PinConnectionListChanged` already does this for a concretely-typed source. If a
distinct "create a named parameter entry from an inferred name and wire into it"
capability is ever wanted, it is a separate `F-` feature, not this `E-` ticket.

## Distinct from / related

- `E-niagara-graph-set-parameter-opaque-scope` (IN-REVIEW) — same opaque-error
  *shape* (a Niagara-graph error that discards the specific cause), different
  method (`niagara.graph.set_parameter`) and cause (scope/type restriction). Its
  shipped remedy (disambiguated `UNSUPPORTED_PARAM_TYPE`/`PARAMETER_NOT_FOUND` at
  `NiagaraGraphHandler.cpp`) is the calibration anchor and the pattern this fix
  follows.
- `F-niagara-link-module-input-to-parameter` (IN-REVIEW) — feeds a value into the
  parameter map at the **stack/module layer** (`niagara.set_module_input`), a
  different method/layer from this raw graph-layer `connect_pins`.
- `E-connect-metasound-graph-input-pin-docs` (OPEN) — analogous special-graph-pin
  docs gap in the **MetaSound** namespace; different domain, same docs-gap class.

## Severity rationale

impact = an opaque error that discards the schema's own, already-computed rejection
reason, forcing the caller to guess what was blocked and why (ergonomic, not a
capability gap — the operation can succeed for a concretely-typed source, and where
it fails the schema supplies a reason, now surfaced) × reach = rare (raw Niagara
script-graph `connect_pins` authoring, not an every-session path) → **Medium**,
held for parity with the sibling opaque-error precedent
`E-niagara-graph-set-parameter-opaque-scope` (same shape, same namespace). (The
original "hard blocker with no workaround" justification is removed — the goal was
abandoned on the false belief that add pins can't be wired; the schema does supply
a reason.)

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `niagara.graph.create_node` task on `/Game/ExampleContent/Niagara/Simple/Simple_system` (emitter `Simple_Emitter`, ParticleUpdate; 10 RPCs; outcome graded `done`/`nonrepro`, judge filed nothing). `create_node` (Multiply op) and the two input wires (`FloatFromCurve.Value -> Mul.A`, `FloatFromCurve001.Value -> Mul.B`) succeeded first-try, but `niagara.graph.connect_pins Mul.Result -> MapSet.Add` (a `UNiagaraNodeParameterMapSet` DynamicAddPin) was rejected with the opaque `[CONNECTION_FAILED] Failed to connect pins (schema blocked connection).` — which names no pin, no type, no cause, and offers no supported path (materialize a named map entry, then wire into it, the way the editor's drag-onto-`+` does). With no recovery path the agent abandoned the "wire the Multiply result into the update flow" half of the goal and left the Result unconnected. Proposes (a) `connect_pins` materialize-and-wire the DynamicAddPin case, and/or (b) an actionable error that names the add-pin incompatibility and the supported path, plus a `docs/wiki-src/niagara.graph.md` `connect_pins` note that a Map Set `+` takes no direct wire. Dedup: ripgrep across OPEN/closed found no ticket naming `niagara.graph.connect_pins` at all; `E-niagara-graph-set-parameter-opaque-scope` (same opaque-error shape, different method — sets a value, Medium anchor), `F-niagara-link-module-input-to-parameter` (same intent, stack layer not graph layer), and `E-connect-metasound-graph-input-pin-docs` (MetaSound special-pin docs) are related but distinct.
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD + fix, verified against plugin + engine source. The opaque error is real: `NiagaraGraphHandler.cpp` ran a bare `GetSchema()->TryCreateConnection(FromPin, ToPin)` and, on false, sent `[CONNECTION_FAILED] Failed to connect pins (schema blocked connection).`, discarding the schema's own reason. But the ticket's categorical premise was wrong: `UEdGraphSchema_Niagara::CanCreateConnection` (EdGraphSchema_Niagara.cpp:1090) DOES allow wiring a concretely-typed output into a Map Set `DynamicAddPin` — the add-pin disallow (:1136/`AddPinIncompatibleTypeText`) fires only when the peer is also Misc, so a Type pin falls through to `CONNECT_RESPONSE_MAKE` (:1299), after which `UNiagaraNodeWithDynamicPins::PinConnectionListChanged` (NiagaraNodeWithDynamicPins.cpp:25-62) auto-materializes the named typed pin. The reported failure was a generic-numeric `Multiply` `Result` (rejected by the four-way add-pin accept check at :1250 → `TypesAreNotCompatibleText` "Types are not compatible"), a specific reason the handler swallowed. Dropped option (a) (materialize-and-wire — the engine already does it) and implemented option (b): `connect_pins` now computes `CanCreateConnection` first and, on `CONNECT_RESPONSE_DISALLOW`, sends `CONNECTION_DISALLOWED` carrying the schema's `Response.Message`, both pin names + Niagara types, and (when a pin is a `DynamicAddPin`) a supported-path hint — mirroring the singular `niagara.connect_pin` (NiagaraEditHandler.cpp:1128-1131) and the sibling remedy (NiagaraGraphHandler.cpp:490-518). Files: `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraGraphHandler.cpp` (link-safe DynamicAddPin detector + restructured connect result), `Plugins/PinWright/Docs/wiki-src/niagara.graph.md` (new `### niagara.graph.connect_pins` section). Test: `PinWright.niagara.graph.connect_pins.DisallowedSurfacesReason` (TestNiagaraHandlers.cpp) — an in-code same-node DISALLOW asserts the `CONNECTION_DISALLOWED` code, the schema reason surfaced, both pins named, the `DynamicAddPin` hint present, and the old opaque sentinel absent. Severity held at Medium for parity with the sibling opaque-error precedent; rationale rewritten to drop the false no-workaround-blocker framing.
