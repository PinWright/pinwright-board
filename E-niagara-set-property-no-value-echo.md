---
id: E-niagara-set-property-no-value-echo
title: "niagara.set_property confirms nothing: no echo of the value written, no report of the value replaced, so every renderer/emitter property write needs a second inspect the sibling verbs made unnecessary"
status: OPEN
severity: Low
category: ergonomic
tags: [niagara, set-property, renderer, readback, echo, verification, asymmetry]
encounters: 1
lastSeen: 2026-09-05
---

# The sibling verbs echo; this one does not

`niagara.set_module_input` returns the canonical pin-default string it wrote
(`"value":"(X=4.0,Y=14.0)"`), plus `pinId`, `linked`, and `replacedOverride`
naming whatever the write destroyed. Its wiki page makes that the *contract*:

> Confirm a literal edit from this result directly — no second inspect needed.

`niagara.set_static_switch` does the same, and goes further: it echoes `value`
(the authored name now on the pin), `index`, and `displayName`, which is the
only reason a caller can tell that the display name `Random Non-Uniform`
resolved to branch **4** / `NewEnumerator2` on `InitializeParticle`.

`niagara.set_property` echoes neither. A successful renderer write returns:

```json
{"success": true, "operation": "set_property", "assetPath": "/Game/FPS/VFX/NS_Muzzle_AR",
 "assetKind": "NiagaraSystem", "compileRequested": false, "compiled": false,
 "saveRequested": false, "saved": false, "saveState": "notRequested",
 "saveDetail": "No save was requested; the edit is in memory only...",
 "quiescedInstances": 0, "emitter": "Smoke",
 "dataInterfaceCheck": "mismatched", "mismatchedScripts": [...],
 "targetKind": "renderer", "propertyPath": "Alignment"}
```

Everything about the *call* and nothing about the *value*. There is no `value`,
no `previousValue`, and no field distinguishing "assigned `VelocityAligned`"
from "assigned something the enum parser coerced elsewhere" or from
"assigned the value it already held".

## Why the asymmetry matters more here than it looks

The properties this verb owns are exactly the ones where a silent coercion is
plausible and invisible:

- **Enum-by-name.** `Alignment: "VelocityAligned"` and
  `FacingMode: "FaceCameraPosition"` are byte enums addressed by name. Niagara
  clamps an unresolvable selector rather than failing — the same failure shape
  `B-niagara-static-switch-enum-value-map-undiscoverable` documents for static
  switches, where the fix was precisely to publish `index` + `displayName` in
  the response. `set_property` has no equivalent.
- **Signed ints with a sign convention that decides layering.**
  `SortOrderHint: -10` is only correct if lower submits first. It does
  (`NiagaraSystem.cpp:2481`, `RendererSortInfo.Sort(A.SortHint < B.SortHint)`),
  but the response cannot say whether `-10` was stored, clamped, or coerced to
  `0` — and getting it backwards puts alpha-blended smoke *in front* of the
  additive muzzle flash instead of behind it.

## What it cost, concretely

Retuning the AR muzzle smoke tail on `/Game/FPS/VFX/NS_Muzzle_AR` (emitter
`Smoke`, renderer 0), three properties were set: `Alignment`, `FacingMode`,
`SortOrderHint`. All three returned `success: true` and nothing else, so all
three had to be confirmed with a follow-up
`niagara.inspect {includeProperties:true}` — the exact round-trip the
`set_module_input` wiki page tells callers they no longer need.

A `grep -a` on the saved `.uasset` does **not** substitute here, and that is
the sharp edge. `"VelocityAligned"` and `"FaceCameraPosition"` are name-table
entries: this system's `Core`, `Petals` and `Streak` renderers already used
both, so the on-disk count was `1` **before and after** the edit. The
project-standard disk check therefore returns a green answer that carries no
information about the renderer actually written. `SortOrderHint` is worse —
a raw `int32`, not greppable at all.

So for this verb the only verification path is a second read, and the second
read is `niagara.inspect`, which on a real system spills past the display
threshold (449 KB for this one) and lands in
`B-response-spill-file-wraps-payload-in-mcp-envelope`.

## Proposal

Add to the success response, mirroring `set_module_input`:

- `value` — the value as stored, read back off the resolved property after the
  write (for a byte enum, the enumerator name; for a numeric, the number).
- `previousValue` — what it held before, so an idempotent re-set is
  distinguishable from a real change and an accidental clobber of another
  agent's edit is visible in the record.
- For an enum property, `index` alongside the name, matching what
  `set_static_switch` already publishes.

`previousValue` is the higher-value half on a shared editor: three streams edit
these assets concurrently, and a write that silently replaced a teammate's
value currently leaves no trace in either the response or the file.

## History

- `#1-filed` `OPEN` VFX — Hit setting `Alignment`, `FacingMode` and
  `SortOrderHint` on the `Smoke` renderer of `/Game/FPS/VFX/NS_Muzzle_AR` and
  the standalone `/Game/FPS/VFX/Emitters/E_FPS_MuzzleAR_Smoke`, against the
  live editor on port 27145. All six writes returned `success: true` with no
  `value` field; the response body is quoted verbatim above. Confirmed the
  writes landed only via a follow-up `niagara.inspect {includeProperties:true}`
  (`Alignment=VelocityAligned, FacingMode=FaceCameraPosition,
  SortOrderHint=-10`). Also confirmed the disk-grep fallback is uninformative
  for this verb: `VelocityAligned` and `FaceCameraPosition` each occur exactly
  once in `NS_Muzzle_AR.uasset` both before and after, because three sibling
  renderers already referenced those name-table entries. No workaround beyond
  the second inspect. Contrast is with the same namespace's own
  `set_module_input` / `set_static_switch`, whose echoes were used in this same
  session to confirm 14 module-input writes and one static-switch write with no
  follow-up read at all.
