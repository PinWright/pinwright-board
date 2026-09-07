---
id: E-input-modifier-no-rpc-route-docs
title: "input.md never states there is NO RPC route to author a UInputAction modifier — the property.set / container.array.append fallback silently no-ops, forcing a wiki-wide grep + engine/plugin source dive"
status: OPEN
severity: Low
category: ergonomic
tags: [no-rpc-route-undocumented, input, enhanced-input, docs, discoverability, wiki]
encounters: 1
costly: 1
lastSeen: 2026-07-04T22:48:11.7838674+03:00
---

# The `input` wiki does not warn that Input Action modifiers cannot be authored by ANY RPC

When `input.set_input_modifier` returns `NOT_IMPLEMENTED` ("Author modifiers
in the editor"), the natural next question for an automation caller is "can I
reach `UInputAction.Modifiers` through the generic reflection writers instead?"
The `input` wiki pages give no answer, so the caller has to reverse-engineer one
from C++.

The answer is no: there is currently **no** RPC route to attach a modifier
(Negate or otherwise) to a `UInputAction`:

- `input.set_input_modifier` is an explicit `NOT_IMPLEMENTED` stub
  (tracked by `B-input-trigger-modifier-stub-silent-success`).
- `property.set` on the instanced `Modifiers` array silently stores `null`
  instead of instancing the subobject (tracked by
  `B-property-set-object-array-silent-null`).
- `container.array.append` shares the same importer, so it has the same
  limitation.

Both code defects are filed. This ticket is the **docs/discoverability** angle
they do not cover: neither `docs/wiki-src/input.md` nor
`input.set_input_modifier.md` tells the caller that the generic
`property.set` / `container.array.append` fallback on `Modifiers` is also a
dead end. The stub's `NOT_IMPLEMENTED` message points at editor authoring, but
does not rule out the generic writers — so a thorough caller still probes them
and then source-dives to confirm the negative.

## Why it's process friction (clean per-call outcomes)

The block itself is a genuine tool gap (the two code tickets above). The
*process* cost this audit is filing is the discovery burned to establish that
the gap is total. The trace shows the agent, after reading
`input.set_input_modifier.md` (which already flags the stub), spent the bulk of
its remaining budget hunting a workaround the docs never rule out:

- grep of the whole wiki for `Negate` / `Modifiers` / `Instanced`;
- reads of `property.set.md`, `container.array.append.md` + `container.array.md`,
  `asset.dump-sidecars.md`;
- greps of engine `InputModifiers.h` / `InputAction.h` (confirming
  `UInputModifierNegate` bX/bY/bZ and the Instanced `Modifiers` array);
- a `LAST RESORT` read of the plugin's `PropertyImport.cpp` to prove the
  `FObjectProperty` array-inner path only accepts a path-to-existing-object and
  cannot instance a subobject.

A single line on the input page ("there is currently no RPC route to attach an
Input Action modifier; `property.set` / `container.array.append` on `Modifiers`
store `null` and do not instance the subobject — author modifiers in the
editor") would have short-circuited that entire loop.

## What it should do

In `docs/wiki-src/input.md` (and/or the `input.set_input_modifier.md` overlay),
add one sentence to the modifier/trigger Gotchas note stating that the generic
reflection fallbacks are ALSO a dead end for `Modifiers`/`Triggers`:

- `input.set_input_modifier` / `input.set_input_trigger` return
  `NOT_IMPLEMENTED`.
- `property.set` and `container.array.append` on the instanced
  `Modifiers` / `Triggers` arrays cannot instance the modifier/trigger
  subobject — they store `null` (a silent no-op today) and do not error.
- Therefore there is no RPC route to author an Input Action modifier or
  trigger; author them in the editor.

Wiki page to improve: `docs/wiki-src/input.md`. This is editorial only and is
**contingent** on the two code tickets: if `set_input_modifier` is implemented
(Option 1 of `B-input-trigger-modifier-stub-silent-success`) so a real route
exists, drop this and document the route instead.

## Evidence

Focus method `input.set_input_modifier`; outcome `blocked_by_mcp`; 24
`mcp__pinwright__call` RPCs. Everything except the invert-Y modifier authored
and verified cleanly (3 UInputActions, IMC_Default with 6 bindings,
mappingCount=6, valueTypes 2/2/0). Friction note: "the intended
input.set_input_modifier is stubbed, and I had to try property.set (which
silently no-op'd to [null]); confirmed the root cause by reading the plugin's
PropertyImport.cpp (last resort) ... no authoring RPC can attach the Negate;
container.array.append shares the same importer." CallAnalyzer flagged the same
discovery as a wiki-nav inefficiency: "the wiki says 'Author modifiers in the
editor' but never states that property.set / container.array ALSO cannot
instantiate instanced modifier subobjects, so the agent had to reverse-engineer
that from C++."

severity rationale: impact=discoverability (docs-only; the tool already directs
to editor authoring, so the miss only costs a source-dive, no wrong data) ×
reach=rare (the fallback-spelunk path fires only when a caller distrusts the
NOT_IMPLEMENTED message) -> Low.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit of a Realism-mode Enhanced-Input task (author IA_Move/IA_Look Axis2D + IA_Jump Boolean + IMC_Default with WASD/Mouse2D/Space bindings, then apply a Negate invert-Y modifier to IA_Look and verify). All 24 RPCs clean except the modifier sub-goal, which is genuinely blocked. The two code defects are already filed (`B-input-trigger-modifier-stub-silent-success` for the stub verb, `B-property-set-object-array-silent-null` for the property.set null). This is the distinct docs angle they omit: the `input` wiki never warns that the generic `property.set` / `container.array.append` fallback on `Modifiers` is also a dead end, so the agent burned a wiki-wide grep + engine-header greps + a last-resort plugin `PropertyImport.cpp` read to confirm the negative. One Gotchas sentence on `docs/wiki-src/input.md` would erase that loop.
