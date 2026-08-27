---
id: B-lighting-configure-shadows-echoes-unmeasured
title: "lighting.configure_shadows echoes virtualShadowMaps off the request and returns success even when the cvar lookup returns null or the write is refused"
status: OPEN
severity: High
category: bug
tags: [lighting, configure_shadows, virtual-shadow-maps, cvar, measured-vs-requested, false-success]
encounters: 1
lastSeen: 2026-08-28
---

# Nothing in this verb is measured

`lighting.configure_shadows` reports `virtualShadowMaps` straight off the **request**. It returns
`success: true` when `FindConsoleVariable("r.Shadow.Virtual.Enable")` returns **null**, and when a
higher-priority setter refuses the write. Nothing it publishes is read back from anything.

That is worse than `B-setup-volumetric-fog-enabled-true-while-cvar-off` and
`B-light-shaft-flags-decorative-under-scalability`, both now fixed: those at least wrote the component
flag they claimed. This one can report a shadow configuration that was never applied at all.

**Fix:** the pattern both those fixes established, now house style — read the cvar through
`IConsoleManager`, report the measured value, distinguish "not found" from "found and zero" by omitting
the value rather than zeroing it, and warn when the requested state cannot take effect. See
`lighting.setup_volumetric_fog` and `lighting.setup_light_shafts` for the shape.

## History
- `#1-worst-of-the-lighting-audit` `OPEN` reporter — Identified as the highest-value item by the audit
  the `B-light-shaft-flags-decorative-under-scalability` agent ran across `lighting.*` after fixing its
  own ticket. Source reading, not reproduced.
