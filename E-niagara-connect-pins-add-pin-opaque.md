---
id: E-niagara-connect-pins-add-pin-opaque
title: "niagara.graph.connect_pins rejects wiring an output into a Map Set add pin (DynamicAddPin) with an opaque [CONNECTION_FAILED] that names no pin/type/cause and no supported path"
status: OPEN
severity: Medium
category: ergonomic
tags: [niagara, niagara-graph, connect-pins, dynamic-add-pin, parameter-map, error-message, docs]
encounters: 1
lastSeen: 2026-07-05T11:42:41.6664801+03:00
---

# `niagara.graph.connect_pins` can't feed an output into a Map Set add pin, and the error explains nothing

`niagara.graph.connect_pins` wires plain pin-to-pin fine (in this task it wired
two curve-float `Value` outputs into a new Multiply node's `A`/`B` inputs on the
first try), but it **cannot** wire a node output into a Map Set's **DynamicAddPin**
(the `+` "add" pin of a `UNiagaraNodeParameterMapSet`) — the exact operation the
Niagara editor supports to feed a computed value into the parameter map (dropping
onto `+` materializes a new named map entry and wires the value into it). The call
fails with:

```
[CONNECTION_FAILED] Failed to connect pins (schema blocked connection).
```

That error names **no pin, no pin type, no cause**, and offers **no alternative
path** — it does not say the target is a DynamicAddPin, that a raw `+` pin cannot
take a direct wire, or that the supported move is to materialize a named
parameter entry first (and, if so, via which RPC). Feeding a computed value into
the parameter map is *the* central operation of graph-layer particle-update
authoring, so an opaque rejection here leaves the caller with nowhere to go.

## Why it matters (process cost in this task)

The stated goal had two halves: (1) add a Multiply op and (2) **wire its result
into the update flow**. Half (1) succeeded cleanly. For half (2) the only free
reachable input in this module-based ParticleUpdate chain was the Map Set `Add`
(DynamicAddPin); `connect_pins Mul.Result -> MapSet.Add` was rejected with the
opaque `[CONNECTION_FAILED]`, and with no pin/type/cause and no pointer to the
supported path, the agent could not recover in place — it **abandoned the second
half of the goal and left the Multiply Result unconnected**. The task was still
graded `done` only because node-presence was the primary pass condition; the
"wire the result into the update flow" objective was silently dropped.

Agent friction note, verbatim:

> "wiring the Mul Result INTO the update flow failed — connect_pins to the only
> free reachable input (a Map Set DynamicAddPin) was rejected with
> '[CONNECTION_FAILED] schema blocked connection', so I could only wire the two
> inputs (curve floats) and left the Result unconnected; suspected connect_pins
> gap handling output->DynamicAddPin."

Attempt's own SAY, verbatim:

> "its float Result has no schema-compatible free input pin in this module-based
> chain ... the only free reachable input, the Map Set DynamicAddPin, refuses a
> direct output wire"

Call-log corroboration (namespace `niagara`, focus `niagara.graph.create_node`,
10 RPCs; the two prior `connect_pins` wired `FloatFromCurve.Value -> Mul.A` and
`FloatFromCurve001.Value -> Mul.B`, both `connected:true`):

- `niagara.graph.connect_pins` `Mul.Result -> MapSet.Add (DynamicAddPin)` →
  `is_error:true` `[CONNECTION_FAILED] Failed to connect pins (schema blocked
  connection).`

## What it should do

Pick one (or both) — mirror the dual remedy the board settled on for the sibling
opaque Niagara-graph error `E-niagara-graph-set-parameter-opaque-scope` (name the
case + point at the supported path):

1. **Handle the add-pin case** — when the target is a `DynamicAddPin` on a
   `UNiagaraNodeParameterMapSet`, materialize a new named map entry (from a
   supplied parameter name, or an inferred one) and wire the output into that
   newly-created named pin, so a single `connect_pins` feeds a value into the
   parameter map the way the editor's drag-onto-`+` does. **and/or**
2. **Make the error actionable** — return a specific message that says the target
   is a Map Set add pin (DynamicAddPin), that a raw `+` pin cannot take a direct
   wire, and names the supported path (materialize a named entry first via the
   relevant RPC, then wire into that named pin). Name the source/target pin and
   types so the caller can tell *what* was blocked and *why*.
- **Docs (`docs/wiki-src/niagara.graph.md`, `connect_pins` section):** the wiki
  page currently only says "Pin types must be compatible; otherwise the
  connection is rejected" with no mention of DynamicAddPin handling. State that a
  Map Set `+` (DynamicAddPin) does not accept a direct output wire, and document
  the supported way to feed a computed value into the parameter map.

## Distinct from / related

- `E-niagara-graph-set-parameter-opaque-scope` (IN-REVIEW) — same *shape*
  (an opaque Niagara-graph error that names no cause and no escape path, forcing
  the caller to abandon/redo) but a **different method** (`niagara.graph.set_parameter`,
  which *sets a value* on an exposed `User.*` scalar) and a different root cause
  (scope/type restriction, not a DynamicAddPin wire). This ticket is the
  `connect_pins` member; the Medium precedent there is the calibration anchor.
- `F-niagara-link-module-input-to-parameter` (IN-REVIEW) — feeds a value into the
  parameter map at the **stack/module layer** (`niagara.set_module_input` linking
  an input to a `User.*` parameter). This ticket is the **raw graph layer**
  (`connect_pins` onto a Map Set node's add pin) — different method, different
  layer, both are the "get a computed/parameter value into the map" intent.
- `E-connect-metasound-graph-input-pin-docs` (OPEN) — analogous "connecting into
  a special graph pin needs docs" shape in the **MetaSound** namespace; different
  method/domain, same class of docs gap.
- `B-niagara-create-op-bare-leaf-pinless` / `F-niagara-graph-create-node` — the
  `create_node` half of graph authoring (which worked cleanly here); this is the
  downstream `connect_pins` half.

## Severity rationale

severity rationale: impact=hard blocker with no discoverable in-session
workaround (the "wire the result into the update flow" goal-leg was genuinely
abandoned — the opaque error named no pin/type/cause and no supported path, so
the agent had nowhere to go) × reach=rare (raw Niagara script-graph `connect_pins`
authoring, not an every-session path) → Medium (base High/Medium for a
no-workaround blocker, minus one for the narrow raw-graph reach; held at Medium
not Low because feeding a computed value into the parameter map is *the* central
graph-layer operation and the failure is a silent goal-drop, matching the Medium
precedent on the sibling opaque-error ticket `E-niagara-graph-set-parameter-opaque-scope`).

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `niagara.graph.create_node` task on `/Game/ExampleContent/Niagara/Simple/Simple_system` (emitter `Simple_Emitter`, ParticleUpdate; 10 RPCs; outcome graded `done`/`nonrepro`, judge filed nothing). `create_node` (Multiply op) and the two input wires (`FloatFromCurve.Value -> Mul.A`, `FloatFromCurve001.Value -> Mul.B`) succeeded first-try, but `niagara.graph.connect_pins Mul.Result -> MapSet.Add` (a `UNiagaraNodeParameterMapSet` DynamicAddPin) was rejected with the opaque `[CONNECTION_FAILED] Failed to connect pins (schema blocked connection).` — which names no pin, no type, no cause, and offers no supported path (materialize a named map entry, then wire into it, the way the editor's drag-onto-`+` does). With no recovery path the agent abandoned the "wire the Multiply result into the update flow" half of the goal and left the Result unconnected. Proposes (a) `connect_pins` materialize-and-wire the DynamicAddPin case, and/or (b) an actionable error that names the add-pin incompatibility and the supported path, plus a `docs/wiki-src/niagara.graph.md` `connect_pins` note that a Map Set `+` takes no direct wire. Dedup: ripgrep across OPEN/closed found no ticket naming `niagara.graph.connect_pins` at all; `E-niagara-graph-set-parameter-opaque-scope` (same opaque-error shape, different method — sets a value, Medium anchor), `F-niagara-link-module-input-to-parameter` (same intent, stack layer not graph layer), and `E-connect-metasound-graph-input-pin-docs` (MetaSound special-pin docs) are related but distinct.
