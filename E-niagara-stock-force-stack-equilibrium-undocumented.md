---
id: E-niagara-stock-force-stack-equilibrium-undocumented
title: "The stock VortexForce + PointAttractionForce + Drag + SolveForcesAndVelocity stack has a closed-form equilibrium the wiki never states, so tuning it is guesswork: AttractionStrength is a spring constant, not an acceleration, and Drag.Ignore Mass is the only switch that turns mass variance into speed variance"
status: IN-REVIEW
severity: Medium
category: enhancement
tags: [niagara, wiki, modules, forces, vortex, point-attraction, drag, mass, tuning, discoverability]
encounters: 1
lastSeen: 2026-08-27T23:05:00+05:00
---

# Four stock modules, one equation, and no page says what it is

Building a fish school on the Atlantis map, the emitter rendered as a **perfect hollow torus** — a
30 uu wide band of 89 particles at radius 120. Two days of the previous agent's tuning had moved
`ShapeLocation.Sphere Radius` across its whole range with "no measurable effect", and the conclusion
recorded in the host spec was that the spawn radius does nothing. Both the torus and that conclusion
fall out of one equation that nothing in `niagara.*` documentation states.

## The equation

For the standard force stack — `VortexForce` (tangential, axis `A`), `PointAttractionForce`,
`Drag`, then `SolveForcesAndVelocity`:

```
terminal speed    v = F_vortex / (Drag * mass)     when Drag.Ignore Mass = TRUE
                  v = F_vortex / Drag              when it is FALSE   (mass cancels)
equilibrium radius r = v / sqrt(k)                 where k = PointAttractionForce.AttractionStrength
```

Fitted and then confirmed against this emitter: `v` was pinned at the `Speed Limit` of 330 and
`k` was 8, predicting `r = 330/sqrt(8) = 117` against a **measured 120**. After the retune
(`F_vortex` 420 -> 90, `k` 8 -> 0.23, `Ignore Mass` false -> true, `Mass` 0.8-2.2) the same formula
predicted the new spread and the sim cache agreed.

## Three things this makes obvious that are otherwise invisible

1. **`AttractionStrength` is a spring constant, not an acceleration.** The force grows linearly with
   distance. Reading it as cm/s^2 — which is what its name and every neighbouring force module
   suggest — puts the predicted radius out by two orders of magnitude, which is exactly why the
   value 8 looked reasonable and produced a 120 uu ring.
2. **`Drag.Ignore Mass` is the *only* way per-particle mass variance reaches the visible motion.**
   With it FALSE, drag and force both divide by mass, mass cancels, and `InitializeParticle`'s
   `Mass Mode: Random` is a **no-op on speed** — the particles are randomised and move identically.
   With it TRUE, `v` is proportional to `1/mass` and `r` follows. A user who randomises mass to break
   up a flock and leaves this switch at its default gets nothing and has no way to find out why.
3. **A spawn-shape input can be *erased* rather than ignored.** `ShapeLocation`'s radius does set the
   spawn distribution; the well then pulls every particle onto `r = v/sqrt(k)` in a few seconds of a
   26-42 s lifetime. "No measurable effect" was a misdiagnosis of a strong attractor, and weakening
   `k` made the spawn radius matter again. Any `<X>Location` input can fail this way behind any
   attractive force.

Also worth stating once: the equilibrium is **isochronous** — `omega = v/r = sqrt(k)`, independent of
speed and mass — so a harmonic well alone gives differential radii but a *rigid* angular rate. Curl
noise at a frequency comparable to the group size is what breaks that; noise much finer than the
group jitters individuals without touching the rigid-body read.

## Suggested change

A short "tuning the stock force stack" section on `niagara.md` (or a `niagara.forces` subpage)
carrying the two formulas, the `Ignore Mass` consequence, and the erased-spawn-distribution warning.
This is not plugin behaviour and not a defect — it is engine-module semantics — but it is exactly
the class of thing the wiki exists to shorten, it is cheap to state, and it cost a full session to
rediscover. It generalises to any flock, swarm, debris cloud or vortex effect built from these
modules on any project.

## Environment

UE 5.8, `EAContentExamples58`, `/Game/Atlantis/VFX/NS_FishSchool` + `_Orange` + `_Silver`, emitter
`Fountain`. Measurements from `UNiagaraSimCacheFunctionLibrary.capture_niagara_sim_cache_immediate`
followed by `read_position_attribute`. Full retune table and before/after distribution:
host repo `Docs/map/atlantis-spec.md` section "Fish schools: why they were a torus".

## History

- `#1-observed` `OPEN` reporter — Derived while opening a torus-shaped fish school that an acceptance
  review had rejected as "a conveyor belt, not a shoal". The formula was fitted from a single
  measured data point, used to predict the retune, and then confirmed against the sim cache, which
  is why it is filed as fact rather than as a hypothesis. Not raised as a defect against any verb:
  every `niagara.set_module_input` call involved behaved exactly as documented.
- `#2-documented` `IN-REVIEW` developer — "Documented the stock force-stack equilibrium in wiki-src/niagara.forces.md"
