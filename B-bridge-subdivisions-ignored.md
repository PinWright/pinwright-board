---
id: B-bridge-subdivisions-ignored
title: "geometry.bridge reads and echoes `subdivisions` but never applies it (dead knob, survived the batch-2 bridge fix)"
status: OPEN
severity: Medium
category: bug
tags: [geometry, bridge, subdivisions, ignored-param, dead-knob, silent-success, rpc-audit]
---

# `geometry.bridge`'s `subdivisions` parameter is decorative

`geometry.bridge` accepts a `subdivisions` parameter, reads it, and **echoes it
back in the success result**, but never applies it to the bridge geometry. The
caller asks for a subdivided bridge, gets `success:true` with
`"subdivisions": <n>` named back at them, and gets an unsubdivided bridge.

This is the same dead-knob defect class as the original `geometry.bevel` finding
(a `steps` parameter that influenced nothing, which is why the duplicate
`geometry.chamfer` was removed and `bevel` was fixed in the batch-2 RPC audit,
[`E-rpc-audit-43-record`](E-rpc-audit-43-record.md)). It is filed separately
because it **survived that audit**: the batch-2 fix to `geometry.bridge` corrected
a different defect (a bogus `<5.5` upper version guard that dead-ended the feature
on current engines) and deliberately left the `subdivisions` knob alone, so the
method is now reachable *and* still lying about this parameter.

The echo is what makes it a bug rather than a docs gap: an agent has no in-band
signal that the value was dropped. The response affirms the parameter it ignored,
which is the exact silent-false-success shape the house convention rejects (see
`B-probe-subobject-handle-ignores-class` for the same echoed-but-unused-param
family).

## What it should do

One of:

- **Apply it.** Subdivide the bridge span into `subdivisions` segments along its
  length, which is what the parameter name promises and what the caller reaching
  for it wants (a bridge with intermediate edge rings that can be deformed).
- **Reject it.** If the underlying bridge operation genuinely cannot subdivide,
  drop the parameter from the handler and the wiki page, and reject it with
  `UNKNOWN_PARAMS` like every other unsupported argument, so a caller who passes
  it learns immediately instead of trusting the echo.

Applying it is preferable. Silently accepting a documented knob that does nothing
is the defect; removing the knob only makes the lie honest.

**Workaround:** none in band. Verify the bridge's topology with
`geometry.get_mesh_info` and do not trust the echoed `subdivisions`; add edge rings
separately (`geometry.subdivide`, globally) if you need them.

## History
- `#1-audit-finding` `OPEN` reporter - Found during the batch-2 RPC audit ([`E-rpc-audit-43-record`](E-rpc-audit-43-record.md)) and deliberately left unfixed there (the audit's `geometry.bridge` fix addressed a different defect: a bogus `<5.5` upper version guard that dead-ended the method on current engines). `geometry.bridge` reads a `subdivisions` param, never applies it to the bridge geometry, and echoes it back in the success result, so the caller gets an affirmative `"subdivisions":<n>` alongside an unsubdivided bridge with no in-band signal that the value was dropped. Same dead-knob class as the original `geometry.bevel` `steps` finding (which drove the `geometry.chamfer` removal and the `bevel` fix in the same audit), except this one survives the fix that landed, so the method is now reachable and still lying about this parameter. Fix: apply the subdivision along the bridge span, or drop the param and reject it with `UNKNOWN_PARAMS`; applying it is preferable.
