---
id: B-niagara-decode-pin-default-coerces-to-zero
title: "DecodePinDefault silently coerces an unresolvable stored enum name to 0, and its one caller then reports the declared default while still stamping source:override"
status: OPEN
severity: Medium
category: bug
tags: [niagara, static-switch, DecodePinDefault, asset-dump, silent-wrong-data, self-contradicting]
encounters: 1
lastSeen: 2026-08-27
---

# A dump entry that contradicts itself, whichever way this is fixed

`DecodePinDefault` returns 0 when it cannot resolve the enum name stored on a static-switch pin. Its
one caller, `NiagaraDumpBuilder.cpp` (~:1315), branches on the return value; the `else` branch writes
the **declared default** while still stamping `source: "override"`.

So the dump can say "this switch is overridden, to the declared default value" about a pin whose
stored name resolved to nothing -- two statements that cannot both be true.

Making `DecodePinDefault` return `false` on a miss does not fix it on its own: it just moves the
entry from a plausible-wrong number to a self-contradicting one. The call site's control flow has to
change with it, so the entry either reports the unresolvable name honestly or omits the value and says
why.

This is the third of `B-niagara-static-switch-enum-display-name`'s observations and the only one not
addressed by that ticket's fix, which covered the *write* path (resolution and the published branch
table) rather than the read-back of an already-broken pin.

## History
- `#1-left-by-the-static-switch-fix` `OPEN` reporter -- Recorded by the agent fixing
  `B-niagara-static-switch-enum-value-map-undiscoverable` and
  `B-niagara-static-switch-enum-display-name`, which deliberately left this because it is a different
  function from the two those tickets name and needs a call-site change. Source-level claim.
