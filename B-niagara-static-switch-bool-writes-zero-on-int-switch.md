---
id: B-niagara-static-switch-bool-writes-zero-on-int-switch
title: "niagara.set_static_switch coerces boolean true to \"0\" on an Integer static switch and reports success"
status: OPEN
severity: High
category: bug
tags: [niagara, set-static-switch, integer-switch, silent-wrong-data, light-attributes]
encounters: 2
lastSeen: 2026-09-02T20:01:00Z
---

# `true` on an Integer switch is written as `0` — the branch it selects is "off"

Niagara's stock `Light_Attributes` module gates each light attribute behind an **Integer** static
switch (`Radius`, `Enabled`, `Color`, `Falloff Exponent`, `Diffuse Scale`, `Specular Scale`,
`Volumetric Scattering`, `Radius Calculation Type`, `Color Mode`). `niagara.inspect` reports them
correctly as `"type": "Integer"`. They read like booleans in the Niagara UI, so `true` is the natural
thing to send — and it is accepted:

```
niagara.set_static_switch {entryId: <Light_Attributes>, inputName: "Radius", value: true}
  -> {"success":true, "inputName":"Radius", "value":"0"}
niagara.set_static_switch {entryId: <Light_Attributes>, inputName: "Enabled", value: true}
  -> {"success":true, "inputName":"Enabled", "value":"0"}
```

`true` became `"0"`, which is the *disabled* branch — the exact opposite of the request. Sending `1`
writes `"1"` correctly. Nothing in the response says a coercion happened; `success: true` and a value
echo are all the caller gets, and the echoed `"0"` is easy to read past when you asked for a boolean.

The echo is the only thing that makes this catchable at all, and only if you read it against your
intent rather than against the request. There is no `warning`, no `coerced` field, and the enum
apparatus that makes `set_static_switch` honest for **enum** switches (`enumOptions`, `index`,
`displayName`, `INVALID_VALUE` on an out-of-range index — see
`B-niagara-static-switch-enum-value-map-undiscoverable`) does not cover the Integer case.

Consequence in this session: an `E_FPS_MuzzleAR_Light` emitter with a `NiagaraLightRendererProperties`
renderer and a `Light_Attributes` module whose `Radius` and `Enabled` branches were both off — so the
emitter writes no `Particles.LightRadius`, the renderer's radius binding stays unset, and the light
renders nothing. It compiles clean and validates clean at `level: "strict"`.

**Workaround:** send integers to any static switch `niagara.inspect` reports as
`"type": "Integer"`, and check the response's `value` against the branch you meant, not against what
you sent.

**Fix:** either coerce `true`/`false` to `1`/`0` for an Integer switch (the reading a caller
plainly intends), or refuse a boolean on an Integer switch with an `INVALID_VALUE` naming the accepted
range — the same shape the enum path already uses. Silently mapping `true` to the falsy branch is the
one option that cannot be right.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8, shared editor, port 27145) wiring
  `/Niagara/Modules/Light/Light_Attributes` into `/Game/FPS/VFX/Emitters/E_FPS_MuzzleAR_Light`.
  Evidence: the two write echoes above, the `"type": "Integer", "defaultValue": 0` entries for those
  switches in the same session's `niagara.inspect {includeStack:true}`, and the corrected writes with
  `value: 1` echoing `"1"`. No plugin source read; diagnosis is from the RPC responses.
- `#2-out-of-range-int-accepted-silently` `OPEN` reporter — Additional evidence for this ticket's claim that "the enum apparatus ... does not cover the Integer case", measured directly on the same module. Authoring `/Game/FPS/VFX/Emitters/E_Explosion_Light` with `/Niagara/Modules/Light/Light_Attributes`, I probed the `Radius` Integer switch, whose only meaningful branches are `0` (use the renderer's value) and `1` (set directly):

    set_static_switch {inputName:"Radius", value:99} -> {"success":true,"inputName":"Radius","value":"99"}
    set_static_switch {inputName:"Radius", value:-5} -> {"success":true,"inputName":"Radius","value":"-5"}
    set_static_switch {inputName:"RadiusZzz", value:1} -> [STATIC_SWITCH_NOT_FOUND] No static switch named 'RadiusZzz' in called graph.

  So the switch **name** is validated and a bad one is refused with a clear code, while the switch **value** is not range-checked at all on the Integer path: `99` and `-5` are both stored and echoed verbatim. The enum path, by contrast, returns `enumOptions`/`index`/`displayName` on every write (verified in the same session on `Lifetime Mode`, `Sprite Size Mode`, `Sprite Rotation Mode`, `Mass Mode`, `Color Mode`), which is exactly the discoverability and range-checking the Integer path lacks. This widens the fix in "Fix" above: coercing `true`/`false` is necessary but not sufficient — an Integer static switch should also reject a value outside the branches its graph actually defines, with the same `INVALID_VALUE` shape the enum path uses. As filed, a typo'd `10` for `1` is unrecoverable-by-inspection and, like `#1`, compiles and validates clean at `level:"strict"`. Restored `Radius` to `1` afterwards and re-verified the emitter; no asset was left in the probed state. `encounters` bumped to 2 (distinct observation, same defect surface).
