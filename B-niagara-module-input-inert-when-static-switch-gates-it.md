---
id: B-niagara-module-input-inert-when-static-switch-gates-it
title: "niagara.set_module_input reports success on an input whose only consumer is the 'if true' pin of a static switch that is currently false — the write is inert, and neither the response nor niagara.inspect says so"
status: OPEN
severity: High
category: bug
tags: [niagara, set_module_input, static-switch, silent-noop, false-success, subuv, inspect, readback]
encounters: 1
lastSeen: 2026-09-07
---

# A valid input name, a green write, and a graph that never reads it

`niagara.set_module_input` resolves the input from the placed module stack, writes the
override pin, and echoes the canonical pin default. That is all true and all useless when the
input's only downstream consumer is the `if true` pin of a `UNiagaraNodeStaticSwitch` whose
caller value is `false`: the compiler takes the `if false` branch, the override pin is dead
code, and the effect does not change.

Nothing published distinguishes the two cases. The success response is byte-for-byte the shape
of a live write, and `niagara.inspect {includeStack:true}` afterwards reports
`valueMode: "local"` with the new value — which reads as "this input is set and in force".

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-07

Target: `/Game/FPS/VFX/NS_Impact_Water`, emitter `Column`, module `SubUVAnimation`
(`/Niagara/Modules/Update/SubUV/V2/SubUVAnimation`), entryKey
`Column:00035F2F4CA6C3845E1F9486857D5731`.

```
call("niagara.set_module_input", {
  assetPath: "/Game/FPS/VFX/NS_Impact_Water",
  entryId: "Column:00035F2F4CA6C3845E1F9486857D5731",
  inputName: "Start Frame Range Override",
  value: 2, emitter: "Column", scriptUsage: "ParticleUpdateScript" })
# -> success:true ... "inputName":"Start Frame Range Override",
#    "pinId":"D949BC2D4A7CCCA62CD7DFB8696FFBC6","linked":false,"value":"2.0"
```

`niagara.graph.get {assetPath: "/Niagara/Modules/Update/SubUV/V2/SubUVAnimation"}` shows the
full consumer set of that parameter. `Module.Start Frame Range Override` leaves its
`NiagaraNodeParameterMapGet` output pin, passes a reroute and an `int32 -> float` convert, and
terminates on exactly two pins:

| from | to node | to pin |
|---|---|---|
| Reroute `69440581` | `NiagaraNodeStaticSwitch (UseStartFrame)` `86E3F80E` | `NiagaraInt32 if true` |
| Convert `B322E3A4` | `NiagaraNodeStaticSwitch (UseStartFrame)` `86E3F80E` | `NiagaraFloat if true` |

That switch's `if false` pins are unlinked literals (`'0'` / `'0.000000'`). `Module.End Frame
Range Override` is the same shape on `UseEndFrame` (`BEA7312F`), whose `if false` side is fed by
a `Subtract` off the renderer's SubImage count.

`niagara.inspect {includeStack:true}` on the placed module reported, **before** the write:

```
staticSwitchInputs: ... {"name":"UseStartFrame","value":false,"source":"override"} x5
                        {"name":"UseEndFrame","value":false,"source":"override"}   x6
moduleInputs:       {"name":"Start Frame Range Override","valueMode":"default"}
                    {"name":"End Frame Range Override","valueMode":"default"}
```

and **after** the write, `valueMode:"local", value:"2.0"` — while the switches were still
`false`. Two `niagara.set_static_switch` calls (`UseStartFrame` / `UseEndFrame` -> `true`) are
what actually made the numbers reach the simulation; without them the whole edit is a no-op
that every published signal calls a success.

## Why this is not the dotted-sub-input ticket

`B-niagara-module-input-dotted-subinput-silent-noop` is a **wrong name** accepted and routed to
an unread rapid-iteration parameter. Here the name is right, the input is real, the override pin
is the correct one, and the write lands where it belongs — it is the **graph** that does not read
it, because of a sibling static-switch value the caller was never told about. A caller cannot
avoid this by spelling the input correctly.

## What was expected

The write should still happen (a caller may be staging a value before flipping the switch), but
the response must say the value is currently unreachable. Both facts are already computable in
the handler's own walk:

- The override pin's consumers are one graph hop away — the same walk
  `moduleInputs[].valueMode` already performs.
- The gating switch's resolved value is already published as `staticSwitchInputs[]`.

Minimum: a `reachable: false` / `gatedBy: {switch: "UseStartFrame", value: false, branchTaken:
"false"}` block on the `set_module_input` response, and the same field on
`niagara.inspect`'s `moduleInputs[]` entry so a readback cannot report a dead override as
configuration. `niagara.validate` naming every locally-overridden but unreachable input would
close the class.

## Workaround

Before writing any module input, read `niagara.inspect {includeStack:true}` **and**
`niagara.graph.get` on the module **script** asset, trace the input's `ParameterMapGet` output
pin, and if it terminates on a static-switch branch pin, set the switch with
`niagara.set_static_switch` first. There is no cheaper published signal.

## Wiki pages that did not warn

`Saved/PinWright/wiki/niagara.set_module_input.md` documents the linked-vs-literal refusal and
the rapid-iteration readback trap in detail and says nothing about static-switch gating.
`niagara.compile-state.md` § "Static switch override storage" describes where switch values live
but never that a switch can strand an input override.

## History

- `#1-filed` `OPEN` reporter — Hit while applying a measured VFX plan to `NS_Impact_Water`: step
  P3 was written as two `set_module_input` calls on `SubUVAnimation`'s Start/End Frame Range
  Override. Both returned success with the pin defaults echoed back, and `niagara.inspect`
  confirmed `valueMode:"local"`. A pre-write `niagara.graph.get` of the module script showed both
  parameters terminate only on `UseStartFrame`/`UseEndFrame` `if true` pins, and both switches
  read `false` on that placement — so the plan as written would have shipped a green no-op. Fixed
  in-place with two `niagara.set_static_switch` calls; asset compiles, `niagara.validate
  {level:"strict"}` returns `valid:true`, `dataInterfaceCheck:"consistent"`.
