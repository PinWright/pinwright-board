---
id: F-spatial-raycast-no-batch-multi-origin
title: "spatial.raycast is one-ray-per-call — mapping a floor/surface region costs N separate probes (no batch/grid/scan form)"
status: OPEN
severity: Low
category: feature
tags: [batch, spatial, raycast, surface-scan, floor-probe, region-scan, workaround]
encounters: 1
lastSeen: 2026-07-05T01:09:45.3126624+03:00
---

# `spatial.raycast` has no batch / multi-origin form — sampling a surface region is N round-trips

`spatial.raycast` casts exactly **one** world-space line trace per call and
reports the first blocking hit (registered args: `origin` (required), one of
`direction`/`target`, `maxDistance`, `channel`, `traceComplex`, `ignoreActors` —
`Plugins/PinWright/Docs/wiki-src/spatial.md:32-56`). The only sibling,
`spatial.raycast_screen`, is likewise single-pixel. Neither accepts an array of
origins nor a grid spec, and there is no `spatial.raycast_batch` and no
surface-scan / find-flat-region page anywhere in the spatial namespace.

So the canonical "how far does this floor patch extend / is it flat here" intent —
learning the extent and flatness of a surface before committing to place something
on it — can only be answered by firing **one raycast per sample point**. In the
evidence task the agent hand-rolled a grid of **16 single-ray `spatial.raycast`
calls** (roughly half of all ~31 RPCs in the trace) just to map the flat-floor
patch near origin before daring to build a small vignette there.

## What it should do

Add a batched trace form, either:
- `spatial.raycast` accepting an `origins` array (one hit row per sample), or
- a grid spec `{min, max, step, direction}` returning a hit row per sampled cell, or
- a higher-level `spatial.scan_surface` / `find_flat_region` helper that samples a
  region and returns extent/flatness (e.g. the largest flat patch and its bounds)
  in one call.

Keep the existing single-`origin` form for the trivial one-ray case.

## Evidence of precedent

The plugin already ships `*_batch` convenience verbs for other namespaces
(`actor.spawn_batch`, and the several batch tickets below), so multi-item RPCs are
an established shape. A raycast grid-of-probes is arguably the *most* naturally
batched spatial operation, yet raycast is the one lacking a plural form.

## Why it's process friction (clean per-call outcome)

Every one of the 16 raycasts succeeded first-try (zero `is_error` in the whole
trace); this is a pure round-trip / batch-convenience gap, not a failure. The
workaround is simply to issue the N single-ray calls — no source-dive or error
recovery needed — hence Low.

Honest scope note: the over-probing here was **partly self-inflicted**. This task's
success check is pure center-column raycast + actor-to-actor AABB box math
(`place_relative`/`verify_placement`/`measure_*`), so exhaustively mapping the
floor was not strictly required — the agent could have committed at x=0 after 2-3
probes. But the underlying capability gap — no one-call way to sample a surface
across an area — is real and would help any place-on-uneven-terrain task, where
knowing the flat extent up front genuinely matters.

**Evidence (this audit):** clean-outcome process audit of a spatial placement task
(namespace `spatial`; story: block out a "loot pedestal" vignette — a low wide
platform on the floor, two crates stacked flush and centered on it, a barrel
standing on the floor with a deliberate 30 cm side gap). The Attempt fired 16
consecutive `spatial.raycast` probes with **distinct** origins — (0,0), (0,250),
(200,200), (-200,-200), (0,0,z995 inside-slab check), (150,0), (-150,0), (0,-150),
(0,210), (100,0), (120,0), (-100,0), (90,150), (50,0), (-50,0), (50,200) — a
genuine hand-rolled grid scan (no exact duplicates), because the map
(ExampleProjectWelcome) has no clean flat floor near origin (a narrow uneven
stepped ridge along +Y). The agent's own friction note: *"the current map … has no
clean flat floor near origin … so I spent ~16 raycasts probing floor extent before
committing to build the vignette centered at x=0."* Outcome `clean`; all four
required deterministic checks passed. Confirmed against the on-disk wiki
(`Docs/wiki-src/spatial.md`) that `spatial.raycast` accepts only a single
`origin`+`direction`/`target` (no array/grid) and no batch/scan sibling exists.

severity rationale: impact=convenience/round-trip gap (N extra calls, no failure,
no source-dive workaround needed) × reach=rare (mapping an uneven surface region
is not an every-session path) -> Low

## Cross-ref (same `batch` family, distinct methods — kept separate)

- `F-effect-draw-debug-shapes-batch` (OPEN) — one-shape-per-call preview markers.
- `F-console-batch-get-cvar-values` (OPEN) — one-search-per-cvar reads.
- `F-animation-set-curve-key-no-batch` (OPEN) — one-key-per-RPC curve authoring.
  Each is the same "verb has no batch form → N round-trips" family but a DISTINCT
  method/subsystem; this raycast case is filed separately per the board's
  one-ticket-per-verb convention, seeded with the shared `batch` family tag.

## History
- `#1-initial-audit` `OPEN` reporter — Clean-outcome process audit of a `spatial` placement task (loot-pedestal vignette). The Attempt fired 16 consecutive single-ray `spatial.raycast` probes with distinct origins (a hand-rolled grid scan, ~half of the ~31 RPCs) to map the flat-floor extent near origin before placing, because ExampleProjectWelcome has no clean flat floor there. Confirmed on the on-disk wiki that `spatial.raycast` takes only a single `origin`+direction/target and there is no `spatial.raycast_batch` / surface-scan / find-flat-region form, even though `actor.spawn_batch` and other `*_batch` verbs exist. Pure round-trip/ergonomic gap: every raycast landed first-try, zero is_error, no workaround needed beyond issuing N calls; over-probing was partly self-inflicted (the success check is center-column + actor box math) but the underlying "no one-call surface-region sample" gap is real for place-on-uneven-terrain tasks. Proposes a batched raycast (`origins` array or `{min,max,step,direction}` grid) or a `spatial.scan_surface`/`find_flat_region` helper. No prior spatial/raycast batch ticket on the board (ripgrep clean for `raycast` under OPEN+closed); nearest siblings are distinct-method `batch`-family tickets, kept separate.
