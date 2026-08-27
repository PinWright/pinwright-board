---
id: B-pwmodel-health-blind-to-interior-membrane
title: "The published health gate cannot see a surface spanning a solid's interior: two oppositely wound fans cancel in signedVolume and every other field reads clean"
status: OPEN
severity: High
category: bug
tags: [pwmodel, health, isClosed, signedVolume, false-green, membrane, revolve, model.validate]
encounters: 1
lastSeen: 2026-08-27
---

# `isClosed && signedVolume > 0` returns green on a solid with a membrane through it

This is the general defect behind `B-revolve-closed-profile-fills-bore`. That ticket's geometry half is
fixed -- a closed profile no longer gets capped to the axis -- but the reason it went unnoticed is
untouched: **no published health field can see an interior membrane.**

The two axis fans are oppositely wound, so their contributions to `signedVolume` cancel exactly.
`isClosed` is true (the surface really is closed). `boundaryEdges` is zero. `orientationConsistent` is
true. `degenerateTriangles` is zero. The documented gate `isClosed && signedVolume > 0` passes, and the
solid has a disc through its bore.

So any future op that produces an interior surface -- a boolean leaving a shared wall, a sweep that
self-intersects and welds, an authored profile that doubles back -- gets the same green.

**Fix direction:** a closure check that also asks whether any surface spans the interior. Candidates: a
ray cast from an interior point that should hit exactly one surface and hits three; or a per-component
volume check that would catch two nested shells summing to a plausible total. Related to
`B-pwmodel-health-no-self-intersection`, which asks for the neighbouring signal -- worth designing the
two together rather than bolting on two independent checks.

## History
- `#1-generalised-from-the-revolve-fix` `OPEN` reporter -- Raised by the agent that fixed
  `B-revolve-closed-profile-fills-bore`, which verified the health fields read green both before and
  after its fix and therefore wrote every new assertion geometrically rather than against the gate.
